---
name: cipherpet-stats
description: |
  Read-only chain query for any 0xSoul wallet. Trigger on:
    - 自查: "我的战绩", "看我的 Pet", "show me my pet", "我赢了多少", "我的胜率"
    - 查他人: "看看 0x... 是谁", "0x... 的灵魂", "show their pet"
    - 排行榜: "排行榜", "万神殿", "leaderboard", "谁是 top", "top players"
    - 战绩历史: "我的 相遇记录", "最近相遇", "相遇记录"
  Pure view (无需私钥). Fast: < 500ms for single-address read, < 2s for leaderboard.
  Auto-formats with Chinese / English persona names + emojis. NEVER expose raw addresses if user gave alias.
---

# 链魂 0xSoul · Stats (链上直读)

> 全部数据直接从合约 view 函数读，**任何人** 都能查任意地址，不需要私钥。
>
> 📖 **故障 / 用户疑问 → `Read` [`USER_MANUAL.md`](../../USER_MANUAL.md)**（plugin 根目录 `cipherpet/USER_MANUAL.md`）。手册含 RPC 502 / 地址未召唤 / nickname 显示规则等所有读取错误的一句话修复。**plugin 唯一可信运行说明**。

## 触发场景

- "我的战绩" / "/cipherpet-stats"
- "看看 0x... 的 Pet" / "/cipherpet-stats 0x..."
- "排行榜" / "万神殿" / "/cipherpet-stats leaderboard"

## 链上目标（XLayer mainnet · v3 对称 vault）

```
CipherPetCore: 0xF09877E72E1b133524DE3491DD1BBF89CcF9BF0e
USDT:          0x779ded0c9e1022225f8e0630b35a9b54be713736
RPC:           https://rpc.xlayer.tech
Backup RPC:    https://xlayerrpc.okx.com
Explorer:      https://www.okx.com/web3/explorer/xlayer
```

> 注意：stats 是 **read-only**，**不需要私钥**。
> v3 Pet struct 是 4 字段 `(uint8 typeIdx, uint32 summonedAt, string nickname, string quote)`，不要用 v0.7 旧 3 字段 ABI。

---

## 三种模式

### 模式 1: Self（默认）

```bash
# 优先取 $PRIVATE_KEY 派生的地址，否则取 $WALLET_ADDRESS env
ME=$(cast wallet address --private-key $PRIVATE_KEY 2>/dev/null || echo $WALLET_ADDRESS)
```

进入"模式 2"用 `$ME` 查。

### 模式 2: 查任意地址

```bash
ADDR=$1   # 0x... 或当前用户
CORE="0xF09877E72E1b133524DE3491DD1BBF89CcF9BF0e"
RPC="https://rpc.xlayer.tech"

# Pet 基本信息（v3: 4 字段含 nickname / quote）
PET=$(cast call $CORE "getPet(address)((uint8,uint32,string,string))" $ADDR --rpc-url $RPC)
HAS=$(cast call $CORE "hasSummoned(address)(bool)" $ADDR --rpc-url $RPC)
```

若 `HAS == false`，提示：
```
✦ {shortAddr} 还未召唤链魂
   引导对方运行 /cipherpet-summon
```

否则继续读：

```bash
# Vault + 战绩
VAULT=$(cast call $CORE "getVault(address)((uint128,uint64,uint32,uint32))" $ADDR --rpc-url $RPC)
WINRATE=$(cast call $CORE "getWinRate(address)(uint8)" $ADDR --rpc-url $RPC | awk '{print $1}')

# 最近 10 场
BATTLES=$(cast call $CORE \
    "getUserBattles(address,uint256,uint256)((uint256,address,address,uint128,address,uint64,uint64)[])" \
    $ADDR 0 10 --rpc-url $RPC)
```

**输出格式**:

```
✦ 第五章 · 居所 · 0x1Cb8...1106

┌─────────────────────────────────────────────┐
│  ◈  DeFi 策士  ·  Lv.{level}                  │
│      "穿越过三轮熊市。Aave 是家。"             │
│      ⭐ TOP 5%                                │
├─────────────────────────────────────────────┤
│  备战池:  0.150000 USDT （v3 无上限）          │
│  胜场:    5    败场: 7    胜率: 41%            │
│  总场次:  12                                  │
└─────────────────────────────────────────────┘

最近 10 场 相遇:
  #12  vs 0x2e64 · 0.05 USDT · 挑战者胜  block #29928960
  #11  vs 0x2e64 · 0.10 USDT · 守卫胜    block #29928943
  #10  vs 0x2e64 · 0.20 USDT · 守卫胜    block #29928931
  ...

链上验证: https://www.okx.com/web3/explorer/xlayer/address/0x1Cb8...1106
Dashboard: https://0xsoul.fun/u/0x1Cb8...1106
```

### 模式 3: Leaderboard（万神殿）

用户说"排行榜" / "万神殿"：

```bash
# 拿所有 summoned 地址（vault >= 0 即返回全部）
ADDRS=$(cast call $CORE \
    "getChallengeableTargets(uint128,uint256,uint256)(address[])" \
    0 0 50 --rpc-url $RPC)

# 对每个 addr 拿 wins/winrate，按 wins desc 排
```

输出：

```
✦ 第九章 · 万神殿 · 战神榜

  #   类型           地址            胜场   胜率
  ─────────────────────────────────────────────
  1   战壕狙击手     0x2e64...0042   7      58%
  2   DeFi 策士      0x1Cb8...1106   5      41%
  ...

总召唤数: 2 · 总场次: 12 · 总成交量: 24 USDT
```

---

## 关键技术细节

### Pet 反查 → 中文名

合约只存 `typeIdx` (0-7)。中文/颜色/符印在前端 `persona.ts`。Skill 里硬编码映射表：

```
typeIdx → (中文名, symbol, rarity, default-slogan)
  0 → DeFi 策士     ◈ TOP 5%
  1 → 战壕狙击手    ◆ TOP 12%
  2 → 钻石贤者      ✦ TOP 8%
  3 → 沉默巨鲸      ◊ TOP 3%
  4 → Meme 哲人     ☾ Phase 2
  5 → 协议建造者    ⬡ Phase 2
  6 → 收益猎人      ◉ Phase 2
  7 → 文化策展人    ◐ Phase 2
```

### USDT 单位转换

链上是 uint128 wei (6 decimals)。展示用 wei / 1_000_000 → USDT。

### Level 派生

```
level = min(99, 30 + wins * 3)
```

未存合约，view 时即时计算。

---

## 高可用约束

- **无需私钥** — 全部 view 函数，离线签名都不要
- **无失败** — 主 RPC 失败 → 切备用 RPC `https://xlayerrpc.okx.com`（同 mainnet 集群）
- **缓存可选** — 高频查同一地址可缓存 8 秒（同 Dashboard polling 节奏）
- **格式化容错** — 地址自动 checksum 化，case-insensitive 匹配
