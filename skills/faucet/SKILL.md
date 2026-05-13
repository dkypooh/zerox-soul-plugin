---
name: zerox-soul-faucet
description: |
  0xSoul 官方福袋自动领取（OnChainOS x402 原生赞助）。
  触发条件（AI 自动判断）：用户钱包 OKB 不足 → 调 `/api/pre-mint-gas` 拿 0.0001 OKB；
  用户成功 mint 后 → 调 `/api/post-mint-usdt` 拿 USDT 启动彩蛋。
  Trigger ONLY when user says "领福袋" / "领启动金" / "领 gas" / "我没钱包余额" / "claim faucet" / "starter pack" —
  或者其他 skill（summon / vault）内部级联调用本 skill 完成自动领取流程。
---

# 0xSoul Faucet · 0 Gas Mint + USDT 启动彩蛋

由 0xSoul 官方 Payment Worker 提供（Cloudflare + OnChainOS x402 兼容签名）。
**每地址限领一次** · IP/子网/当日总池多重防御 · 链上 hasSummoned 实时校验。

```
Worker URL: https://web3-payment-worker.fang18967131010.workers.dev
```

---

## 触发场景（AI 必须熟读）

| 用户语境 | 应触发 |
|---------|--------|
| 钱包刚连，没 OKB → 让我 mint | `pre-mint-gas`（自动） |
| Mint 成功 → 后续 PK 没押金 | `post-mint-usdt`（自动） |
| 用户主动："给我领福袋" | 按顺序两步全跑 |
| 用户余额已够 | 调用前先 check，已够则跳过 |

---

## 模式 1 · 检查状态（先于发起任何请求）

```bash
# Step 0: 拿用户钱包地址
ADDR=$(onchainos wallet address --json | jq -r '.address')

# Step 1: 链上读余额，先决定要不要请求 faucet
OKB_BAL=$(cast balance $ADDR --rpc-url https://rpc.xlayer.tech)
HAS_SUMMONED=$(cast call $CORE "hasSummoned(address)(bool)" $ADDR --rpc-url https://rpc.xlayer.tech)

# Step 2: 查 Worker KV 已领状态（去重）
STATUS=$(curl -s "https://web3-payment-worker.fang18967131010.workers.dev/api/status?addr=$ADDR")
GAS_CLAIMED=$(echo $STATUS | jq -r '.gasClaimed')
USDT_CLAIMED=$(echo $STATUS | jq -r '.usdtClaimed')
```

---

## 模式 2 · Pre-Mint Gas 自动领取（mint 之前）

**前置条件**：
- 链上 `hasSummoned === false`
- 钱包 OKB < 0.0001（约 100000000000000 wei）
- KV `gasClaimed === false`

```bash
curl -X POST https://web3-payment-worker.fang18967131010.workers.dev/api/pre-mint-gas \
  -H "Content-Type: application/json" \
  -d "{\"addr\":\"$ADDR\"}"
# 返回: { "ok": true, "txHash": "0x...", "status": "success" }
```

**等链上到账**（最多 15s 轮询）：

```bash
for i in {1..10}; do
  BAL=$(cast balance $ADDR --rpc-url https://rpc.xlayer.tech)
  if [ $(echo "$BAL >= 100000000000000" | bc) -eq 1 ]; then break; fi
  sleep 1.5
done
```

---

## 模式 3 · Post-Mint USDT 启动彩蛋（mint 之后）

**前置条件**：
- 链上 `hasSummoned === true`（必须真的 mint 过）
- 钱包 USDT < 0.1（约 100000 raw · 6 decimals）
- KV `usdtClaimed === false`

```bash
curl -X POST https://web3-payment-worker.fang18967131010.workers.dev/api/post-mint-usdt \
  -H "Content-Type: application/json" \
  -d "{\"addr\":\"$ADDR\"}"
# 返回: { "ok": true, "txHash": "0x...", "status": "success" }
```

---

## 模式 4 · 完整自动彩蛋链（summon skill 应级联调用）

```bash
# 假设当前已知 ADDR / CORE
ADDR=$(onchainos wallet address --json | jq -r '.address')

# 1. 检查状态
HAS=$(cast call $CORE "hasSummoned(address)(bool)" $ADDR --rpc-url https://rpc.xlayer.tech)
OKB=$(cast balance $ADDR --rpc-url https://rpc.xlayer.tech)

# 2. 还没召唤 + OKB 不够 → 先要 gas
if [ "$HAS" = "false" ] && [ $(echo "$OKB < 100000000000000" | bc) -eq 1 ]; then
  echo "→ 申请 gas 福袋"
  GAS_TX=$(curl -s -X POST https://web3-payment-worker.fang18967131010.workers.dev/api/pre-mint-gas \
    -H "Content-Type: application/json" \
    -d "{\"addr\":\"$ADDR\"}" | jq -r '.txHash')
  echo "  tx: https://www.oklink.com/xlayer/tx/$GAS_TX"

  # 等 OKB 到账
  for i in {1..10}; do
    OKB=$(cast balance $ADDR --rpc-url https://rpc.xlayer.tech)
    [ $(echo "$OKB >= 100000000000000" | bc) -eq 1 ] && break
    sleep 1.5
  done
fi

# 3. 用户在 summon skill 里完成 mint（不在本 skill 范围）
# ...

# 4. mint 完成后 → 拿 USDT 启动彩蛋
HAS=$(cast call $CORE "hasSummoned(address)(bool)" $ADDR --rpc-url https://rpc.xlayer.tech)
if [ "$HAS" = "true" ]; then
  echo "→ 申请 USDT 启动彩蛋"
  USDT_TX=$(curl -s -X POST https://web3-payment-worker.fang18967131010.workers.dev/api/post-mint-usdt \
    -H "Content-Type: application/json" \
    -d "{\"addr\":\"$ADDR\"}" | jq -r '.txHash')
  echo "  tx: https://www.oklink.com/xlayer/tx/$USDT_TX"
fi
```

---

## 错误码对照（HTTP 4xx/5xx）

| Code | 含义 | 给用户的话 |
|------|------|-----------|
| `INVALID_ADDR` (400) | 地址非法 | 检查 onchainos wallet address |
| `ALREADY_SUMMONED` (400) | 已 mint，不需 gas | 跳到 mint 后流程 |
| `ALREADY_HAS_GAS` (400) | 钱包已有 OKB | 直接 mint，不需福袋 |
| `NOT_SUMMONED` (400) | post-mint-usdt 时未 mint | 先去 summon |
| `ALREADY_CLAIMED` (409) | 该地址已领过 | 一次性福袋，不补 |
| `RATE_LIMITED` (429) | IP 24h 已领 | 明天再来或换网络 |
| `SUBNET_LIMIT` (429) | 同子网领取过多 | 换出口 IP |
| `BURST_LIMIT` (429) | 60s 内请求过多 | 间隔几秒重试 |
| `DAILY_CAP` (503) | 今日总池已发完 | 明天再来 |
| `SPONSOR_DEPLETED` (503) | 我们的池子干了 | 通知运营充值 |
| `TX_FAILED` / `TX_REVERTED` (502) | 上链失败 | 稍后重试 |

---

## 健康检查（运营/调试用）

```bash
curl -s https://web3-payment-worker.fang18967131010.workers.dev/api/health | jq
```

返回 sponsor 钱包余额 + 当日发放计数 + 单次额度。**地址余额 < 阈值时 worker 自动停发**（SPONSOR_DEPLETED）。

---

## 高可用设计要点（已实现）

1. **双层校验**：链上 `hasSummoned` + `balance` 实时读 + KV 永久去重
2. **链上 receipt 双确认**：worker 等 receipt + 前端二次 wait → 钱包 RPC 同步
3. **重试机制**：mint 刚成功时 `hasSummoned` 链上节点同步延迟，最多 5 次轮询
4. **熔断**：sponsor 余额低于一天总池停发，防被刷干
5. **防爆破**：60s burst + 24h IP + /24 子网当日 + 当日总池四道速率限制
6. **私钥安全**：放 Cloudflare Secret，never log，仅当日额度
7. **CORS**：仅 0xsoul.fun + localhost 白名单
8. **错误屏蔽**：500 通用，不泄 stack/key 细节
