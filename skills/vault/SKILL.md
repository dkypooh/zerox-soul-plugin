---
name: cipherpet-vault
description: |
  Vault management for 0xSoul (deposit / withdraw / status). Trigger on:
    - 存款: "存 5 USDT", "deposit 10", "充钱到 vault", "我想准备好被挑战", "put USDT in vault", "充值"
    - 取款: "取出来", "撤回 vault", "withdraw 3", "把 USDT 转回我钱包", "退出"
    - 查询: "看我的 vault", "vault 状态", "我有多少 USDT 锁定", "check vault"
  Reads/writes via OnChainOS TEE; auto-detects approve allowance, auto-mints first-time approval.
  Auto-routes to cipherpet-init if not logged in. NEVER show slash commands to user.
  Performance: status read < 500ms (cached session); deposit/withdraw 5-10s onchain.
---

# 链魂 0xSoul · Vault (OnChainOS TEE 版)

> 通过 `onchainos wallet contract-call` 走 ERC-4337，TEE 签名
>
> 📖 **故障 / 用户疑问 → `Read` [`USER_MANUAL.md`](../../USER_MANUAL.md)**（plugin 根目录 `cipherpet/USER_MANUAL.md`）。手册含 `InsufficientVaultBalance` / `AmountBelowMin` / approve / vault 无上限等所有 vault 错误的一句话修复。**plugin 唯一可信运行说明**。

## 链上目标（XLayer mainnet）

```
network:  XLayer mainnet (chainId 196)
CipherPetCore: 0xF09877E72E1b133524DE3491DD1BBF89CcF9BF0e
USDT:      0x779ded0c9e1022225f8e0630b35a9b54be713736
```

## 前置：登录 + 已召唤

```bash
onchainos wallet status            # 应已登录
cast call 0xF09877E72E1b133524DE3491DD1BBF89CcF9BF0e \
    "hasSummoned(address)(bool)" $ME \
    --rpc-url https://rpc.xlayer.tech                # 应为 true
```

否则引导 /cipherpet-summon 先召唤。

---

## 模式 1: Deposit（存款）

### 1.1 检查 USDT allowance（首次需 approve）

```bash
USDT="0x779ded0c9e1022225f8e0630b35a9b54be713736"
CORE="0xF09877E72E1b133524DE3491DD1BBF89CcF9BF0e"
RPC="https://rpc.xlayer.tech"

ALLOW=$(cast call $USDT "allowance(address,address)(uint256)" $ME $CORE --rpc-url $RPC | awk '{print $1}')

if [ "$ALLOW" -lt "1000000000" ]; then
    # Encode approve(MAX) calldata
    APPROVE_DATA=$(cast calldata "approve(address,uint256)" $CORE \
        "115792089237316195423570985008687907853269984665640564039457584007913129639935")

    onchainos wallet contract-call \
        --to $USDT --chain xlayer \
        --input-data "$APPROVE_DATA" \
        --biz-type "dapp"
fi
```

### 1.2 执行 deposit

```bash
AMOUNT_USDT=10
AMOUNT_WEI=$((AMOUNT_USDT * 1000000))

DEPOSIT_DATA=$(cast calldata "deposit(uint128)" $AMOUNT_WEI)

RESULT=$(onchainos wallet contract-call \
    --to $CORE --chain xlayer \
    --input-data "$DEPOSIT_DATA" \
    --biz-type "dapp")

# 解析 txHash
TX=$(echo "$RESULT" | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['txHash'])")
echo "✓ Deposit tx: $TX"
```

### 1.3 输出

```
✓ Deposit 完成（TEE 签名）
已存入: {amount} USDT
tx: https://www.okx.com/web3/explorer/xlayer/tx/{tx_hash}

🔥 现在可以被挑战了。其他玩家可押最多 {amount} USDT 与你相遇 PK。
```

---

## 模式 2: Withdraw（提款）

```bash
AMOUNT_WEI=$((AMOUNT_USDT * 1000000))
WITHDRAW_DATA=$(cast calldata "withdraw(uint128)" $AMOUNT_WEI)

onchainos wallet contract-call \
    --to $CORE --chain xlayer \
    --input-data "$WITHDRAW_DATA" \
    --biz-type "dapp"
```

---

## 模式 3: Status（查询，无需签名）

```bash
BAL=$(cast call $CORE "getBalance(address)(uint128)" $ME --rpc-url $RPC | awk '{print $1}')
WALLET_USDT=$(cast call $USDT "balanceOf(address)(uint256)" $ME --rpc-url $RPC | awk '{print $1}')

echo "Vault: $(echo "scale=2; $BAL / 1000000" | bc) USDT (上限 1000)"
echo "Wallet: $(echo "scale=2; $WALLET_USDT / 1000000" | bc) USDT"
```

---

## 错误码

| Error | 含义 |
|-------|------|
| `NotSummoned` | 还没召唤 Pet → /cipherpet-summon |
| `AmountBelowMin` | amount = 0 → 至少存 1 wei |
| `InsufficientVaultBalance` | withdraw 超余额 |

---

## 安全性

- ✅ deposit / withdraw / approve **全部 TEE 签名**
- ✅ 私钥永不离开 OKX 安全飞地
- ✅ 用户**只需邮箱 OTP** 认证一次会话
