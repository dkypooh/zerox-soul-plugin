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
> 📖 **故障 / 用户疑问 → `Read` [`USER_MANUAL.md`](../../USER_MANUAL.md)**（plugin 根目录 `cipherpet/USER_MANUAL.md`）。手册含 `CannotChallengeSelf` / `NotSummoned` / `AmountOutOfRange` / `InsufficientVaultBalance` 等所有相遇 PK 错误的一句话修复。**plugin 唯一可信运行说明**。

## 链上目标（XLayer mainnet）

```
CipherPetCore: 0xe639d8A5C3ABA8F74070BB2eA383b11CBc9568B7
USDT:      0x779ded0c9e1022225f8e0630b35a9b54be713736
```

---

## 模式 1: List 可挑战目标

```bash
MY_STAKE_WEI=1000000   # 1 USDT
cast call $CORE "getChallengeableTargets(uint128,uint256,uint256)(address[])" \
    $MY_STAKE_WEI 0 20 --rpc-url https://rpc.xlayer.tech
```

输出可挑战的地址列表 + 各自 Pet 类型 + 胜率。

---

## 模式 2: Challenge ⭐

### Step 1 — 前置检查

```bash
# 自己已召唤？
HAS_ME=$(cast call $CORE "hasSummoned(address)(bool)" $ME --rpc-url $RPC)

# 对方已召唤？
HAS_T=$(cast call $CORE "hasSummoned(address)(bool)" $TARGET --rpc-url $RPC)

# 对方 vault ≥ amount？
T_BAL=$(cast call $CORE "getBalance(address)(uint128)" $TARGET --rpc-url $RPC | awk '{print $1}')
```

### Step 2 — USDT approve (如未)

```bash
ALLOW=$(cast call $USDT "allowance(address,address)(uint256)" $ME $CORE --rpc-url $RPC | awk '{print $1}')
if [ "$ALLOW" -lt "10000000" ]; then
    APPROVE_DATA=$(cast calldata "approve(address,uint256)" $CORE \
        "115792089237316195423570985008687907853269984665640564039457584007913129639935")
    onchainos wallet contract-call --to $USDT --chain xlayer \
        --input-data "$APPROVE_DATA" --biz-type "dapp"
fi
```

### Step 3 — 执行 challenge（TEE 签名一笔搞定）

```bash
AMOUNT_USDT=1
AMOUNT_WEI=$((AMOUNT_USDT * 1000000))

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
    奖池 {2*amount} USDT 已直转入你钱包
    tx: https://www.okx.com/web3/explorer/xlayer/tx/{tx_hash}

if winner == target:
    💀 你输了
    损失 {amount} USDT（已被划入对方 vault）
    再战: /cipherpet-challenge <addr> <amount>
```

---

## 完整 ERC-4337 流程图

```
你的 Plugin 调用:
  onchainos wallet contract-call ...
              ↓
OnChainOS 客户端:
  - 构造 UserOperation
  - 拿用户 session token
              ↓
OnChainOS 后端 + TEE:
  - 验证 session
  - 飞地内私钥签 UserOperation
              ↓
ERC-4337 Bundler:
  - 打包 UserOps
  - 提交到 EntryPoint (0x0000...da032)
              ↓
EntryPoint (XLayer mainnet):
  - 验证签名
  - 路由到用户的 Smart Account
  - 调用 CipherPetCore.challenge(target, amount)
              ↓
CipherPetCore:
  - 收 B 的 USDT
  - 划 A 的 vault USDT
  - keccak256 算 winner
  - 转账 winner 2X
  - emit BattleResolved
              ↓
返回:
  { txHash: "0x...", orderId: "..." }
```

→ 整个过程**私钥永远在 TEE 内**

---

## 错误码

| Error | 修复 |
|-------|------|
| `NotSummoned` | 双方都要先 /cipherpet-summon |
| `AmountOutOfRange` | 1 ≤ amount ≤ 10 USDT |
| `InsufficientVaultBalance` | 对方 vault 不够 → 换对手 |
| `insufficient funds for gas` | 你 OKB 不够 → 充 XLayer mainnet OKB |
