---
name: cipherpet-challenge
description: |
  1v1 相遇 PK on XLayer — stake USDT, contract picks winner via on-chain randomness, winner takes 2X (0 fee). Trigger on:
    - "挑战 0x...", "和 0x... 打一场", "相遇", "challenge 0x...", "我要和他比一比"
    - "找人打", "随便挑一个挑战", "find me an opponent", "auto match"
    - "看谁能挑战", "可挑战的人", "list challengeable targets"
    - 任何包含"赢点 USDT" / "试试运气" / "测一下手气" 的意图
  Auto: detects auth, checks both summoned, ensures USDT approval, encodes calldata, broadcasts via OnChainOS TEE.
  NEVER expose slash commands. Performance: 8-15s per encounter PK (onchain confirmation).
---

# 链魂 0xSoul · Challenge (OnChainOS TEE 版 · v3 对称 vault)

> B 挑战 A，一笔 ERC-4337 UserOperation 在 TEE 内代签 → EntryPoint 执行 → 合约决胜 + 转账
>
> 📖 **故障 / 用户疑问 → `Read` [`USER_MANUAL.md`](../../USER_MANUAL.md)**（plugin 根目录 `cipherpet/USER_MANUAL.md`）。手册含 `CannotChallengeSelf` / `NotSummoned` / `AmountBelowMin` / `InsufficientVaultBalance` 等所有相遇 PK 错误的一句话修复。**plugin 唯一可信运行说明**。

## 链上目标（XLayer mainnet · v3 对称 vault）

```
CipherPetCore: 0xF09877E72E1b133524DE3491DD1BBF89CcF9BF0e
USDT:      0x779ded0c9e1022225f8e0630b35a9b54be713736
```

---

## Step 0 — v2 / v3 关键差异（AI 必须先读）

> **重要**：如果用户的旧记忆或老教程里有 v2 / v0.x 的操作步骤，**用 v3 流程替换**，不要照搬。

| 维度 | v2（旧合约，已废弃） | **v3（当前合约 `0xF09877…BF0e`）** |
|------|------|------|
| 挑战者押金来源 | USDT 钱包 → `transferFrom` 拉走 | **vault 直接扣减**（用户需先 deposit） |
| 防御者押金来源 | vault | vault（不变） |
| 是否需要 USDT approve | ✅ 挑战前必 approve | **❌ 挑战不需要 approve**（challenge() 全程 0 外部 ERC20 调用） |
| 赢家奖金到哪 | 挑战者赢→钱包；防御者赢→vault | **不论谁赢，2× 押注都进赢家 vault**（再 withdraw 才到钱包） |
| 一笔 challenge 发的 event | 1 条 BattleResolved | **4 条**（3× VaultBalanceChanged 全审计 + 1× BattleResolved） |
| 资金池守恒 | 无明确计数器 | `totalVaultLocked` 公开 + `getPoolStatus()` 自检 |

**对 AI 的行为约束**：
1. **绝不在 challenge 前 approve USDT** —— 只在 deposit 时 approve
2. **绝不告诉用户"挑战完直接到钱包"** —— 奖金永远先进 vault，要 withdraw 才到钱包
3. 用户问"为什么我赢了 USDT 没进钱包" → 告诉他奖金在 vault 里，让他说"取出来"触发 vault skill 的 withdraw

---

## 模式 1: List 可挑战目标

```bash
MY_STAKE_WEI=50000   # 0.05 USDT（v3 min = 0.01 USDT = 1e4）
cast call $CORE "getChallengeableTargets(uint128,uint256,uint256)(address[])" \
    $MY_STAKE_WEI 0 20 --rpc-url https://rpc.xlayer.tech
```

输出可挑战的地址列表 + 各自 Pet 类型 + 胜率。

### 子模式 1.1: 随机找人 PK ⭐

```bash
# 拿可挑战目标列表
TARGETS=$(cast call $CORE "getChallengeableTargets(uint128,uint256,uint256)(address[])" \
    $MY_STAKE_WEI 0 30 --rpc-url $RPC)

# 过滤掉自己（小写比较），随机抽一个
PICK=$(echo "$TARGETS" | tr -d '[]' | tr ',' '\n' | tr -d ' ' \
    | awk -v me="$(echo $ME | tr A-F a-f)" 'tolower($0) != tolower(me) && $0 != ""' \
    | shuf -n 1)

echo "🎲 自动选中：$PICK"
TARGET=$PICK
```

> 兜底：如果 `TARGETS` 为空（没人 vault 余额够），告诉用户"还没有目标 vault 余额 ≥ 你这笔押注，换小押注或稍后再试"。

---

## 模式 2: Challenge ⭐

> ⚠️ **押注阈值（AI 必读，不要弄错）**：
> - 最小：**0.01 USDT = 1e4 wei**（`MIN_STAKE_PER_BATTLE = 1e4`）
> - 最大：无上限（受 uint128/2 防溢出 + 双方 vault 余额约束）
> - **不要把 "1 USDT" 当成最小** —— 用户备战池有 0.05 USDT 也能开 5 局
> - AI 默认建议押注：用户没指定时优先 `0.05`（≈ 50000 wei），不是 1 USDT

### Step 1 — 前置检查（**双方 vault 都需 ≥ amount**）

```bash
# 自己已召唤 + vault ≥ amount？
HAS_ME=$(cast call $CORE "hasSummoned(address)(bool)" $ME --rpc-url $RPC)
ME_BAL=$(cast call $CORE "getBalance(address)(uint128)" $ME --rpc-url $RPC | awk '{print $1}')

# 对方已召唤 + vault ≥ amount？
HAS_T=$(cast call $CORE "hasSummoned(address)(bool)" $TARGET --rpc-url $RPC)
T_BAL=$(cast call $CORE "getBalance(address)(uint128)" $TARGET --rpc-url $RPC | awk '{print $1}')
```

### Step 2 — 我方 vault 不足时先 deposit

```bash
# 示例：默认押 0.05 USDT（远低于 v3 最低 0.01 / 1e4 wei，留出多局空间）
# 用户没指定金额时优先这个，不要默认 1 USDT
AMOUNT_USDT="0.05"
AMOUNT_WEI=$(awk -v u="$AMOUNT_USDT" 'BEGIN{printf "%d", u*1000000}')   # 浮点 → wei
MIN_STAKE_WEI=10000                                                       # 0.01 USDT = 1e4

# 守门：低于合约 MIN_STAKE 直接拒绝
if [ "$AMOUNT_WEI" -lt "$MIN_STAKE_WEI" ]; then
    echo "❌ 押注 $AMOUNT_USDT USDT < 最低 0.01 USDT，拒绝"
    exit 1
fi

if [ "$ME_BAL" -lt "$AMOUNT_WEI" ]; then
    # 1) 先 approve USDT 给 CipherPetCore（一次性，仅 deposit 时需要）
    ALLOW=$(cast call $USDT "allowance(address,address)(uint256)" $ME $CORE --rpc-url $RPC | awk '{print $1}')
    if [ "$ALLOW" -lt "$AMOUNT_WEI" ]; then
        APPROVE_DATA=$(cast calldata "approve(address,uint256)" $CORE \
            "115792089237316195423570985008687907853269984665640564039457584007913129639935")
        onchainos wallet contract-call --to $USDT --chain xlayer \
            --input-data "$APPROVE_DATA" --biz-type "dapp"
    fi
    # 2) deposit 把 USDT 进备战池（只够这一场即可，也可以一次存更大方便复用）
    DEPOSIT_DATA=$(cast calldata "deposit(uint128)" $AMOUNT_WEI)
    onchainos wallet contract-call --to $CORE --chain xlayer \
        --input-data "$DEPOSIT_DATA" --biz-type "dapp"
fi
```

### Step 3 — 执行 challenge（TEE 签名一笔搞定，零外部 token 调用）

```bash
CALLDATA=$(cast calldata "challenge(address,uint128)" $TARGET $AMOUNT_WEI)

RESULT=$(onchainos wallet contract-call \
    --to $CORE --chain xlayer \
    --input-data "$CALLDATA" \
    --biz-type "dapp")

TX_HASH=$(echo "$RESULT" | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['txHash'])")

# 等 receipt
sleep 5
LATEST=$(cast call $CORE "nextBattleId()(uint256)" --rpc-url $RPC | awk '{print $1}')
BATTLE=$(cast call $CORE "getBattle(uint256)((uint256,address,address,uint128,address,uint64,uint64))" $LATEST --rpc-url $RPC)
WINNER=$(echo "$BATTLE" | grep -oE '0x[a-fA-F0-9]{40}' | sed -n 3p)
```

### Step 4 — 输出胜负

```
if winner == me:
    🎉 你赢了！
    净赚 +{amount} USDT
    奖池 {2*amount} USDT 已存入你的备战池（可随时 withdraw）
    tx: https://www.okx.com/web3/explorer/xlayer/tx/{tx_hash}

if winner == target:
    💀 你输了
    损失 {amount} USDT（已被划入对方备战池）
    再战: /cipherpet-challenge <addr> <amount>
```

---

## 完整 ERC-4337 流程图（v3 对称版）

```
你的 Plugin 调用:
  onchainos wallet contract-call ...
              ↓
OnChainOS 客户端 → TEE 内签 UserOp
              ↓
EntryPoint (XLayer mainnet)
              ↓
CipherPetCore.challenge(target, amount):
  - 检查 cV.balance >= amount
  - 检查 dV.balance >= amount
  - cV.balance -= amount  (挑战者备战池 -)
  - dV.balance -= amount  (防御者备战池 -)
  - winner = keccak256(...) & 1
  - winnerVault.balance += 2*amount
  - emit 3×VaultBalanceChanged + BattleResolved
  - totalVaultLocked 不变（净零）
              ↓
返回:
  { txHash: "0x...", orderId: "..." }
```

→ 整个过程**零外部 ERC20 调用 + 私钥永远在 TEE 内**

---

## 错误码

| Error | 语义 | 一句话修复 |
|-------|------|------|
| `NotSummoned(挑战者)` | 你还没召唤 | 先走 /cipherpet-summon |
| `NotSummoned(防御者)` | 对方还没召唤 | 换一个目标，或让他先来召唤 |
| `AmountBelowMin` | amount < 0.01 USDT（v3 也支持 amount 上限 = uint128/2，正常用户碰不到） | 押注 ≥ 0.01 USDT |
| `InsufficientVaultBalance(挑战者)` | **你**的 vault < amount | "存 X USDT" → AI 自动 approve + deposit → 重试挑战 |
| `InsufficientVaultBalance(防御者)` | **对方**的 vault < amount | 换个目标（用 `getChallengeableTargets`）或押少一点 |
| `CannotChallengeSelf` | 自己挑战自己 | 换目标地址 |
| `insufficient funds for gas` | 你 OKB 不够 | XLayer mainnet 充 ≥ 0.001 OKB |

> **v3 区分技巧**：`InsufficientVaultBalance(has, want)` 里的 `has` 是不足那一方的余额。同时调 `getBalance(me)` / `getBalance(target)` 跟 want 比对，谁先 < want 谁就是问题方。

---

## 资金池健康度自检（demo / 故障排查）

```bash
# 任意时刻读资金池三元组
cast call $CORE "getPoolStatus()(uint256,uint128,uint256)" --rpc-url $RPC
# heldUSDT  ：合约链上 USDT 余额
# totalVaultLocked：所有 vault 余额之和（合约欠用户的总数）
# surplus    ：heldUSDT - totalVaultLocked（理应 ≥ 0；> 0 说明有人误转 USDT 进合约）
```

挑战一次后：
- `totalVaultLocked` **应不变**（双方各 -amount + 赢家 +2*amount 净零）
- `heldUSDT` 不变（vault 内部簿记）

这是 v3 的资金池守恒不变量。
