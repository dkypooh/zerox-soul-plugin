# 链魂 0xSoul · Claude Code Plugin

> Web3 解决了交易，从没解决过孤独。
> 0xSoul 把你的钱包**读成一个灵魂**。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![XLayer](https://img.shields.io/badge/Chain-XLayer%20mainnet-d4a84b)](https://www.okx.com/web3/explorer/xlayer)
[![OnChainOS](https://img.shields.io/badge/Wallet-OnChainOS%20TEE-5ab490)](https://onchainos.com)
[![Live](https://img.shields.io/badge/Demo-0xsoul.fun-c84b3c)](https://0xsoul.fun)

> **Live Dashboard**: <https://0xsoul.fun>
> **Mainnet Contract**: [`0x1e58374A103BB37613586B79f7c9aA90fb1b6d26`](https://www.okx.com/web3/explorer/xlayer/address/0x1e58374A103BB37613586B79f7c9aA90fb1b6d26)

---

## 一句话定位

5 道题 + AI 读你链上历史 → 召唤你的**链上灵魂**（Soulbound ERC-721）→ 在 XLayer 上 1v1 PK 真 USDT。
**全程对话，零私钥，零抽水，零 admin。**

---

## ⚡ 30 秒上手

打开 Claude Code，粘贴下面 2 行：

```text
/plugin marketplace add dkypooh/zerox-soul-plugin
/plugin install zerox-soul@zerox-soul
```

然后跟 AI 说一句：

> **召唤我的链魂**

AI 会自动：
1. 装好缺失工具（Foundry / OnChainOS CLI）
2. 引导邮箱 + OTP 登录（OKX OnChainOS TEE 钱包）
3. 问你 5 道 MBTI 风格题 + 昵称 + Slogan
4. 链上 mint Soulbound NFT
5. 给你一张可分享的灵魂卡

---

## ✨ 6 个核心 Skill

| Skill | 触发短语 | 链上动作 |
|-------|---------|----------|
| **init** | 自动调用（环境检测） | 安装 + 登录 + 缓存 session |
| **summon** | "召唤我的链魂" / "summon my soul" | `summon(uint8,string,string)` mint NFT |
| **vault** | "存 5 USDT" / "取出来" | `deposit` / `withdraw` USDT |
| **challenge** | "找人打" / "挑战 0x..." | `challenge(address,uint128)` 1v1 |
| **stats** | "我的战绩" / "排行榜" | 纯 view 读 leaderboard |
| **share** | "分享我的灵魂" / "生成推文" | 拼接 Twitter intent + 朋友圈文案 |

所有 skill 都是**纯自然语言触发** —— 用户从不需要敲斜杠命令。

---

## 🏗️ 架构

```
用户对话 (Claude Code)
   ↓
0xSoul 6 个 Skill
   ↓
┌──────────────┬──────────────┬──────────────┐
│ OnChainOS    │ cast (RPC)   │ XLayer       │
│ TEE Wallet   │ (read only)  │ mainnet      │
└──────────────┴──────────────┴──────────────┘
   ↓ ERC-4337 UserOperation
0xSoul Contract (Soulbound ERC-721)
   ↓
0xsoul.fun  ← 静态 Dashboard 直读 RPC
```

**关键设计**：
- **零私钥暴露** — 所有签名在 OKX TEE 飞地内完成
- **零后端** — Dashboard 是静态 SPA，viem 直读 XLayer RPC
- **零抽水** — 合约 0 admin function，赢家拿 100% 押注
- **可重放验证** — 链上随机数 = `keccak256(blockhash + prevrandao + ...)`

---

## 📜 链上数据透明

```
Network:        XLayer mainnet (chainId 196)
RPC:            https://rpc.xlayer.tech
Explorer:       https://www.okx.com/web3/explorer/xlayer

CipherPetCore:  0x1e58374A103BB37613586B79f7c9aA90fb1b6d26
USDT (stake):   0x779ded0c9e1022225f8e0630b35a9b54be713736

ERC-721 name:   0xSoul
ERC-721 symbol: SOUL
```

任何人都能用 `cast` 一行验证：

```bash
cast call 0x1e58374A103BB37613586B79f7c9aA90fb1b6d26 \
  "getProtocolStats()(uint256,uint256,uint256)" \
  --rpc-url https://rpc.xlayer.tech
# 返回: (已召唤数, 相遇 PK 总数, 总成交量)
```

---

## 🎮 用户体验

完整的「人话 → 链上 → 视觉化」流程：

```
用户: 召唤我的链魂
AI:   ✦ 请告诉我邮箱（用于 TEE 钱包登录）
用户: bo@example.com
AI:   ✓ OTP 已发，6 位数字告诉我
用户: 040379
AI:   🎉 登录成功，开始 5 道题——
      Q1: 你的链上信仰——追逐百倍，还是复利长青？
      ...
用户: BBAAB
AI:   ✦ 命中: DeFi 策士 ◈
      给你的链魂起个名字（默认: 0x8eb3）
用户: 老炮
AI:   ✓ 昵称 = 老炮 (6 字节)
      默认 Slogan: "穿越过三轮熊市。Aave 是家。"
用户: 保持默认
AI:   🎉 你的链魂已上链:
      Type:     DeFi 策士
      Nickname: 老炮
      Tx:       https://www.okx.com/web3/explorer/xlayer/tx/0xabcd...
      Dashboard: https://0xsoul.fun/u/0x8eb3...
```

---

## 🔐 安全保障

- **私钥永不离开 TEE** — OKX 自己也读不到，OnChainOS 用 Trusted Execution Environment
- **邮箱 OTP** — 每会话过期，无长期 token
- **ERC-4337 UserOperation** — 走 EntryPoint 标准路径，链上可追溯
- **0 admin function** — 合约部署后**完全 immutable**，所有参数公开 constant
- **Soulbound NFT** — Pet 永远绑定召唤者，不能转、不能卖、不能 approve

---

## 📂 仓库结构

```
zerox-soul-plugin/
├── README.md              ← 你正在看
├── USER_MANUAL.md         ← 完整用户手册（故障表 + 一键问答）
├── LICENSE                ← MIT
├── .claude-plugin/
│   ├── plugin.json        ← Claude Code plugin manifest
│   └── marketplace.json   ← Marketplace manifest
└── skills/
    ├── _shared/           ← 共用：合约地址 / preflight 检测
    ├── init/              ← 一键环境装置 + 邮箱 OTP 登录
    ├── summon/            ← 召唤 5 题 + nickname + slogan + mint
    ├── vault/             ← USDT 存提
    ├── challenge/         ← 1v1 PK
    ├── stats/             ← 战绩/排行榜读取
    └── share/             ← 病毒分享物料生成
```

---

## 🧪 故障速查

| 现象 | 一句话修复 |
|------|----------|
| `/plugin install` 报 `Invalid plugin name` | 用 `zerox-soul`，不要用 `0xsoul`（数字开头不合规） |
| AI 不识别"召唤" | `/plugin enable zerox-soul` |
| `AlreadySummoned` revert | 每地址一辈子只能召唤一次，换地址或看战绩 |
| `InvalidNicknameLength` | 昵称 > 32 字节（≈ 10 中文）— 缩短或留空走默认 |
| `insufficient funds for gas` | 钱包没 OKB，从 OKX 提 ≥ 0.001 OKB 到 XLayer 主网 |
| `InsufficientVaultBalance` | 对手 vault 没钱 — 换目标或先存钱让别人挑战你 |

完整故障表见 [`USER_MANUAL.md`](./USER_MANUAL.md)。

---

## 💎 Built on

- [**OKX OnChainOS**](https://onchainos.com) — TEE Agentic Wallet
- [**XLayer**](https://www.okx.com/xlayer) — OKX 自研 L2 (chainId 196)
- [**Claude Code**](https://claude.com/claude-code) — Anthropic 官方 CLI
- [**Foundry**](https://book.getfoundry.sh/) — Solidity 工具链
- [**viem**](https://viem.sh/) — TypeScript Web3 客户端

---

## 📄 License

MIT — see [LICENSE](./LICENSE).

---

**召唤指令：「召唤我的链魂」**
**Demo：[0xsoul.fun](https://0xsoul.fun)** ✦
