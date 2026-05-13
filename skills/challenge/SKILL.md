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

# 链魂 0xSoul · Challenge (OnChainOS TEE 版)

> B 挑战 A，一笔 ERC-4337 UserOperation 在 TEE 内代签 → EntryPoint 执行 → 合约决胜 + 转账
>
> 📖 **故障 / 用户疑问 → `Read` [`USER_MANUAL.md`](../../USER_MANUAL.md)**（plugin 根目录 `cipherpet/USER_MANUAL.md`）。手册含 `CannotChallengeSelf` / `NotSummoned` / `AmountBelowMin` / `InsufficientVaultBalance` 等所有相遇 PK 错误的一句话修复。**plugin 唯一可信运行说明**。

## 链上目标（XLayer mainnet · v3 对称 vault）

```
CipherPetCore: 0xF09877E72E1b133524DE3491DD1BBF89CcF9BF0e
USDT:      0x779ded0c9e1022225f8e0630b35a9b54be713736
```

> **v3 关键变化**：挑战时双方押金均从 vault 扣减，赢家奖金一律存入 vault。**不再需要 USDT approve**——因为 challenge() 全程不调用 ERC20。攻击面 = 0。

---

## 模式 1: List 可挑战目标

```bash
MY_STAKE_WEI=50000   # 0.05 USDT（v3 min = 0.01 USDT = 1e4）
cast call $CORE "getChallengeableTargets(uint128,uint256,uint256)(address[])" \
    $MY_STAKE_WEI 0 20 --rpc-url https://rpc.xlayer.tech
```

输出可挑战的地址列表 + 各自 Pet 类型 + 胜率。

---

## 模式 2: Challenge ⭐

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
AMOUNT_USDT=1
AMOUNT_WEI=$((AMOUNT_USDT * 1000000))

if [ "$ME_BAL" -lt "$AMOUNT_WEI" ]; then
    # 1) 先 approve USDT 给 CipherPetCore（一次性，仅 deposit 时需要）
    ALLOW=$(cast call $USDT "allowance(address,address)(uint256)" $ME $CORE --rpc-url $RPC | awk '{print $1}')
    if [ "$ALLOW" -lt "$AMOUNT_WEI" ]; then
        APPROVE_DATA=$(cast calldata "approve(address,uint256)" $CORE \
            "115792089237316195423570985008687907853269984665640564039457584007913129639935")
        onchainos wallet contract-call --to $USDT --chain xlayer \
            --input-data "$APPROVE_DATA" --biz-type "dapp"
    fi
    # 2) deposit 把 USDT 进备战池
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

| Error | 修复 |
|-------|------|
| `NotSummoned` | 双方都要先 /cipherpet-summon |
| `AmountBelowMin` | amount ≥ 0.01 USDT（v3 无上限） |
| `InsufficientVaultBalance` | 双方 vault 都需 ≥ amount → 自己不足先 deposit；对方不足换对手或押少点 |
| `CannotChallengeSelf` | 不能挑战自己 |
| `insufficient funds for gas` | 你 OKB 不够 → 充 XLayer mainnet OKB |
