---
name: cipherpet-share
description: |
  Generate viral share materials from on-chain Pet data. Trigger on:
    - "分享", "分享我的灵魂", "分享 Pet", "share my pet", "post to twitter"
    - "生成推文", "写一条朋友圈", "tweet about me", "朋友圈文案"
    - "Mirror 长文", "写一篇长文章"
    - "炫耀一下", "show off"
    - **战绩分享**: "分享战绩", "晒一下战绩", "share my battles", "炫战绩", "晒胜场", "我的相遇战报", "battle highlights"
  Auto-pulls chain data → assembles: Twitter intent URL (中英双语), 朋友圈 (3 tones), Mirror skeleton, screenshot checklist.
  Battle highlights mode pulls getUserBattles + 算 streak / 最大单笔奖金 → 推文模板。
  Pure view. Fast: < 2s total. NEVER expose slash commands.
---

# 链魂 0xSoul · Share

> 拉链上 Pet 数据 + 拼接病毒分享物料。**无需私钥**。
>
> 📖 **故障 / 用户疑问 → `Read` [`USER_MANUAL.md`](../../USER_MANUAL.md)**（plugin 根目录 `cipherpet/USER_MANUAL.md`）。手册含 Dashboard URL、合约地址、Twitter intent 模板等所有分享相关参数。**plugin 唯一可信运行说明**。

## 触发场景

- "分享我的 Pet" / "/cipherpet-share"
- "生成推文" / "Twitter"
- "我要 Mirror 长文"

## 链上目标（XLayer mainnet · v3 对称 vault）

```
CipherPetCore: 0xF09877E72E1b133524DE3491DD1BBF89CcF9BF0e
USDT:          0x779ded0c9e1022225f8e0630b35a9b54be713736
RPC:           https://rpc.xlayer.tech
Backup RPC:    https://xlayerrpc.okx.com
Explorer:      https://www.okx.com/web3/explorer/xlayer
```

---

## Workflow

### Step 1 — 确定要分享谁的 Pet

```bash
ME=$(cast wallet address --private-key $PRIVATE_KEY 2>/dev/null \
     || echo $WALLET_ADDRESS \
     || echo $1)  # 第一个参数

ADDR=${ME:-$1}
CORE="0xF09877E72E1b133524DE3491DD1BBF89CcF9BF0e"
RPC="https://rpc.xlayer.tech"
```

### Step 2 — 拉真链数据

```bash
# v3 Pet struct: 4 字段（typeIdx, summonedAt, nickname, quote）
PET=$(cast call $CORE "getPet(address)((uint8,uint32,string,string))" $ADDR --rpc-url $RPC)
VAULT=$(cast call $CORE "getVault(address)((uint128,uint64,uint32,uint32))" $ADDR --rpc-url $RPC)
WINRATE=$(cast call $CORE "getWinRate(address)(uint8)" $ADDR --rpc-url $RPC | awk '{print $1}')
```

提取：`typeIdx`, `nickname`, `quote`（中文 slogan）, `wins`, `losses`.

### Step 3 — 生成 4 类物料

#### A. Twitter 推文（中英双语）

**中文**:
```
🔮 召唤了我在 XLayer 上的灵魂。

AI 说我是「{typeName}」⭐ {rarity}。
"{quote}"

战绩：{wins}胜{losses}负 · {winRate}% 胜率

链上 1v1 相遇 PK，0 协议费，winner 拿走 2× （奖金直接进备战池，可随时提）
扔你的钱包，看你的 👇
https://0xsoul.fun/u/{address}

#0xSoul #链魂 · Built on @okx OnchainOS · @XLayerOfficial
```

**English**:
```
🔮 Just summoned my onchain soul on XLayer.

AI says I'm a {nameEn} ⭐ {rarity}.
"{quoteEn}"   ← 翻译或保留中文

Record: {wins}W {losses}L · {winRate}% win rate

1v1 onchain PK · 0 protocol fee · Winner takes 2× into vault.
Drop your wallet → https://0xsoul.fun/u/{address}

#0xSoul · @okx OnchainOS · @XLayerOfficial
```

**一键发推链接**:
```
https://twitter.com/intent/tweet?text=<URL-encoded text>
```

#### B. 朋友圈文案（3 个语气）

**Friendly**:
```
做了个小东西参加内部黑客松——链魂 0xSoul。
让 AI 解读你的钱包灵魂 👻
我是「{typeName}」⭐ {rarity}，"{quote}"
评论区扔地址，我帮你召唤一只 🔓
```

**Provocative 钓鱼**:
```
"你的钱包，比你以为的更暴露你是谁。"

AI 说我是「{typeName}」。
8 种人格，你赌你是哪一种？
评论区扔地址，30 秒帮你召唤 👁
https://0xsoul.fun/u/{address}
```

**Professional**:
```
内部黑客松：链魂 0xSoul
- 基于 OKX OnchainOS · 默认链 XLayer
- 5 道题 + 链上数据 → 16 种链上人格 NFT (soulbound)
- 链上 1v1 相遇 PK，0 协议费

我的结果：{typeName} ({rarity})
战绩：{wins}W {losses}L
"{quote}"

→ https://0xsoul.fun/u/{address}
```

#### C. Mirror.xyz 长文骨架（800 字）

```
标题: 我在 XLayer 上召唤了一只灵魂 Pet — 黑客松 48 小时构建笔记

【段 1 · 钩子】
"Web3 已经解决了交易问题，从未解决过孤独。"

【段 2 · 痛点】
- XLayer TVL $25M，120M OKX 用户转化率极低
- 缺的不是 DEX，是"为什么留下"

【段 3 · 我的灵魂】
- 5 道题
- AI 读了我的钱包历史
- 它说我是「{typeName}」
- "{quote}"
- 这个判断永久写在 XLayer 上，soulbound 不可转

【段 4 · 怎么做的】
- 合约 290 行（CipherPetCore.sol）
- 0 协议费 / 0 admin / 完全自治
- 1v1 相遇 PK，链上随机（blockhash + prevrandao）
- 5 个 Skill 编排在 Claude Code 内

【段 5 · CTA】
- 在 okx.ai / Claude Code 跑 /cipherpet-summon
- 8 种人格等你来认

【段 6 · 致谢】
Built on OKX OnchainOS · XLayer · 黑客松 v0.1
```

#### D. 截图素材清单

```
✦ 提交表"对外宣传"字段需要这些截图:
  ① Twitter 推文 + 互动数
  ② 朋友圈 + ≥3 条评论
  ③ 内部 Lark 群分享
  ④ 0xsoul.fun/u/{address} 个人页（含 Persona Card）
  ⑤ /cipherpet-summon 命令在 Claude Code 里的录屏 GIF
```

### Step 3b — 战绩分享模式（用户说"分享战绩" / "晒战绩" / "炫战绩" 时启用）

> 切换到这个模式时**跳过** Step 3 的 A/B/C/D 物料，直接生成**战报版**推文。
>
> 触发关键词：`分享战绩` / `晒战绩` / `炫战绩` / `相遇战报` / `share my battles` / `battle highlights` / `炫胜场`

#### 拉战报数据

```bash
# 取最近 10 场（v3 BattleLog 7 字段：id, challenger, defender, amount, winner, timestamp, blockNumber）
BATTLES=$(cast call $CORE \
    "getUserBattles(address,uint256,uint256)((uint256,address,address,uint128,address,uint64,uint64)[])" \
    $ADDR 0 10 --rpc-url $RPC)

VAULT=$(cast call $CORE "getVault(address)((uint128,uint64,uint32,uint32))" $ADDR --rpc-url $RPC)
# 解出 wins / losses（VAULT 的第 3 / 4 字段）
WINS=$(echo "$VAULT" | awk -F'[(,)]' '{print $4}' | tr -d ' ')
LOSSES=$(echo "$VAULT" | awk -F'[(,)]' '{print $5}' | tr -d ' ')
```

AI 必须从原始 `BATTLES` 算出以下"亮点"，挑最有戏剧性的 2-3 条 spotlight：
- **当前连胜 / 连败**（按 timestamp 降序找最长同向 run）
- **最大单笔奖金**（amount * 2，winner == $ADDR 的场次里最大）
- **最快赢回本钱**（最近一次 LOSS 后第几场翻盘）
- **新手段位 / 高手段位**（wins ≥ 10 视为高手；首胜单独 spotlight）

#### 中文战报推文模板（AI 二选一渲染）

**模板 1 · 连胜炫耀**:
```
🔥 我在 0xSoul 上正连胜 {streak} 场。

最近一战：押 {bestAmount} USDT → 收 {bestPayout} USDT
累计战绩：{wins}胜{losses}负 · {winRate}% 胜率

链上 1v1 随机，0 协议费，赢家 2× 直接进备战池。
钱包是我的运气 → https://0xsoul.fun/u/{address}

#0xSoul #链魂 · Built on @okx OnchainOS · @XLayerOfficial
```

**模板 2 · 翻盘叙事**:
```
💸 上一把输了 {loseAmount} USDT。这一把翻盘 {winAmount} 直接回备战池。

8 种链上人格，我是「{typeName}」⭐ {rarity}
战绩：{wins}胜{losses}负

链上随机不可作弊，押完就出胜负，奖金 0 抽水。
扔你的钱包 → https://0xsoul.fun/u/{address}

#0xSoul · @okx OnchainOS · @XLayerOfficial
```

#### English variant

```
🔥 Just stacked {streak} wins on 0xSoul.

Latest battle: staked {bestAmount} USDT → swept {bestPayout} USDT
Record: {wins}W {losses}L · {winRate}% win rate

1v1 onchain random · 0 protocol fee · 2× to your vault.
Drop your wallet → https://0xsoul.fun/u/{address}

#0xSoul · @okx OnchainOS · @XLayerOfficial
```

#### 朋友圈短文（战报版）

**简洁炫耀**:
```
0xSoul 上连赢 {streak} 把 ✊
押 {bestAmount} 收 {bestPayout}，备战池又胖了。
评论区扔地址 → 帮你召唤一只 🔓
https://0xsoul.fun/u/{address}
```

**意外翻盘**:
```
"输不起就别上。"
上把输 {loseAmount}，下把翻盘 {winAmount}。
不押到归零，我都不算结束 👁
https://0xsoul.fun/u/{address}
```

#### 一键发推

```bash
# 把上面模板里的占位符替换好后 URL-encode 拼到 intent
TEXT_ENCODED=$(python3 -c "import urllib.parse, sys; print(urllib.parse.quote(sys.argv[1]))" "$TWEET_TEXT")
echo "https://twitter.com/intent/tweet?text=$TEXT_ENCODED"
```

#### 高可用兜底

- 用户**完全没战绩** (`wins == 0 && losses == 0`)：不要生成战报，引导先 "找人打" 跑一场再来
- 用户**只有败场** (`wins == 0 && losses > 0`)：用"翻盘叙事"模板但去掉连胜数字，加一句 "下一把就翻盘"
- 用户**单场过大押注**（amount > $ADDR 当前 vault）：在 spotlight 里加 "🔥 全押翻盘" 标签
- `getUserBattles` 返回空 / RPC 错：fall back 到 Step 3 A 模板（不含战报）

### Step 4 — 一键发布建议

```
推荐节奏:
  5.13 周三晚: 推 + 朋友圈
  5.14 周四午: 内部 Lark
  5.15 周五早: Mirror（可选）
```

---

## 高可用

- 全 view 读链，**不需要私钥**
- RPC 失败 → 切备用 endpoint
- Pet 不存在 → 提示先 `/cipherpet-summon`
- 用户没填地址 → 默认用 `$PRIVATE_KEY` 派生
