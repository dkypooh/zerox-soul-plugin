---
name: cipherpet-share
description: |
  Generate viral share materials from on-chain Pet data. Trigger on:
    - "分享", "分享我的灵魂", "分享 Pet", "share my pet", "post to twitter"
    - "生成推文", "写一条朋友圈", "tweet about me", "朋友圈文案"
    - "Mirror 长文", "写一篇长文章"
    - "炫耀一下", "show off"
  Auto-pulls chain data → assembles: Twitter intent URL (中英双语), 朋友圈 (3 tones), Mirror skeleton, screenshot checklist.
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

## 链上目标

```
CipherPetCore: 0xF8ad4C706d70f6e3Fa0f082bfB8fE143d0d0DE71
RPC:           https://testrpc.xlayer.tech/terigon
```

---

## Workflow

### Step 1 — 确定要分享谁的 Pet

```bash
ME=$(cast wallet address --private-key $PRIVATE_KEY 2>/dev/null \
     || echo $WALLET_ADDRESS \
     || echo $1)  # 第一个参数

ADDR=${ME:-$1}
```

### Step 2 — 拉真链数据

```bash
PET=$(cast call $CORE "getPet(address)((uint8,uint32,string))" $ADDR --rpc-url $RPC)
VAULT=$(cast call $CORE "getVault(address)((uint128,uint64,uint32,uint32))" $ADDR --rpc-url $RPC)
WINRATE=$(cast call $CORE "getWinRate(address)(uint8)" $ADDR --rpc-url $RPC | awk '{print $1}')
```

提取：`typeIdx`, `quote`（中文 slogan）, `wins`, `losses`.

### Step 3 — 生成 4 类物料

#### A. Twitter 推文（中英双语）

**中文**:
```
🔮 召唤了我在 XLayer 上的灵魂。

AI 说我是「{typeName}」⭐ {rarity}。
"{quote}"

战绩：{wins}胜{losses}负 · {winRate}% 胜率

链上 1v1 相遇 PK，0 协议费，winner 拿走 2X
扔你的钱包，看你的 👇
https://cipher.pet/u/{address}

#0xSoul #链魂 · Built on @okx OnchainOS · @XLayerOfficial
```

**English**:
```
🔮 Just summoned my onchain soul on XLayer.

AI says I'm a {nameEn} ⭐ {rarity}.
"{quoteEn}"   ← 翻译或保留中文

Record: {wins}W {losses}L · {winRate}% win rate

1v1 相遇 PK. 0 protocol fee. Winner takes 2X.
Drop your wallet → https://cipher.pet/u/{address}

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
https://cipher.pet/u/{address}
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

→ https://cipher.pet/u/{address}
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
  ④ cipher.pet/u/{address} 个人页（含 Persona Card）
  ⑤ /cipherpet-summon 命令在 Claude Code 里的录屏 GIF
```

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
