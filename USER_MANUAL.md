# 链魂 0xSoul · 用户手册 (v0.4)

> 你只需要做两件事：**装 Plugin** + **跟 AI 说话**。
> 不需要懂区块链、不需要记任何命令。

> 🔑 Plugin 内部 slug 叫 `zerox-soul`（Claude Code 校验规则不允许 plugin name 以数字开头）。
> 用户看到的品牌依然是「0xSoul · 链魂」—— ERC-721 合约、Dashboard、对话文案都是 0xSoul。

---

## ⚡ 30 秒上手（新终端冷启动）

### Step 1 · 打开 Claude Code

```bash
claude
```

### Step 2 · 装 Plugin（粘贴 2 行，一次性配置）

```
/plugin marketplace add dkypooh/zerox-soul-plugin
/plugin install zerox-soul@zerox-soul
```

看到 `✓ zerox-soul installed` 就完事。**这是你最后一次需要敲 `/` 命令**。

> Claude Code 会把 marketplace + plugin 持久化到 `~/.claude/`，下次开终端不用再装。

### Step 3 · 跟 AI 说话

下面任意一句都行：

```
召唤我的链魂
做一只 Pet 给我
我是哪种链上人格？
试试 0xSoul
```

AI 会自动接管：

- 没装 `cast` / `onchainos`？AI 静默装好
- 没登录 OnChainOS？AI 问你邮箱 → 发 OTP → 你回 6 位数字
- 然后 5 道题
- **问你昵称**（v0.4 新；默认填地址前 6 位 `0x8eb3`，也可自己起 ≤ 10 中文字）
- 问你 Slogan（默认有，也可改 ≤ 20 中文字）
- 链上 TEE 代签 + 广播 → 输出 tx hash + 灵魂卡

**全程你只输入：邮箱 → OTP → 5 题答案 → （可选）昵称 → （可选）Slogan。**

---

## 💰 召唤 vs PK · 钱怎么花

**很多人误以为「玩 0xSoul 必须先充几美金」—— 不是。** 召唤和 PK 是**两件独立的事**：

| 想做的事 | 需要的资产 | 充值步骤 |
|---------|----------|---------|
| **召唤自己的链魂**（mint NFT、答 5 题、拿身份卡）| 0.005 OKB（gas）| OKX 提币 → X Layer → 0.005 OKB · **一次性** |
| **1v1 PK 抢 USDT**（可选，不感兴趣可跳过）| 1–10 USDT（押注）| 想 PK 时再 "存 N USDT" 进 vault |

### 不想 PK 的玩家路径（**0 充值 USDT**）
```
准备 0.005 OKB gas → 召唤 → 拿到灵魂卡 → 分享 → 收工
```
就这样。Soulbound NFT 永久属于你，**0xSoul.fun/u/<你的地址>** 是你独占的展示页。

### 想 PK 的玩家路径
```
召唤完后 → "存 3 USDT" → "找人打" / "挑战 0x..." → 链上随机出胜负 → 赢家拿 2×
```

### 关于资金安全
- **私钥从未离开 OKX TEE 飞地**（你自己也不掌握私钥，OKX 也读不到）
- **合约 0 admin / 完全 immutable**（部署后没人能改规则，包括我们）
- **vault 上限 1000 USDT / 地址**：故意压低，反巨鲸 + 反金融化
- **任何时候说"取出来"**：vault 余额秒回你钱包

---

## 💬 全部都是说人话

| 你想干啥     | 直接说                                                   |
| ------------ | -------------------------------------------------------- |
| 召唤 Pet     | "召唤" / "做一只 Pet 给我" / "我是哪种链上人格"          |
| 看自己战绩   | "看我的 Pet" / "我的战绩" / "我赢了多少"                 |
| 看别人       | "看看 0x... 是什么"                                      |
| 排行榜       | "排行榜" / "万神殿" / "谁最强"                           |
| 存钱准备相遇 | "存 5 USDT" / "充值 vault" / "准备好被挑战"              |
| 取钱         | "取出来" / "退出" / "把 USDT 转回我钱包"                 |
| 看 vault     | "看我的 vault" / "我锁了多少"                            |
| 找人打       | "找人打" / "随便挑一个" / "看谁能挑战"                   |
| 挑战指定地址 | "挑战 0x..." / "和他打 1 U"                              |
| 改昵称展示   | _（v0.4 一次性写入，召唤后不可改；想改先 burn 重 mint）_ |
| 分享         | "分享" / "生成推文" / "炫耀一下"                         |

AI 听到这些自然语言**自动**匹配能力。完全**不需要**记 `/cipherpet-summon` 这种斜杠命令。

---

## 🎬 真实对话演示（v0.4 含昵称步骤）

```
你: 帮我做一只链魂

AI: ✦ 第一次来，需要你的邮箱（OnChainOS TEE 钱包认证用）

你: bo.fang@example.com

AI: ✓ OTP 已发到 bo***@example.com，告诉我 6 位数字

你: 040379

AI: 🎉 登录成功！开始 5 道题——

    Q1: 你的链上信仰——追逐百倍，还是复利长青？
    A. 我要百倍，要么零
    B. 我要 5% 稳赚，长期复利
    C. 两个都要，看市场
    D. 只要 APY

你: B

[... Q2 - Q5 ...]

AI: ✦ 灵魂解析:
    - builderScore = 2
    - degenScore = 1
    - 命中: DeFi 策士 ◈

    给你的链魂起个名字吧（默认就用地址前 6 位）：

       默认: 0x8eb3
       想换？直接说，≤ 10 中文 / 32 字节

你: 老炮

AI: ✓ 昵称 = 「老炮」(6 字节)

    AI 默认 Slogan: "穿越过三轮熊市。Aave 是家。"
    要保持还是自己写？

你: 保持默认

AI: 正在 TEE 内代签 → 链上广播...

    🎉 你的链魂已上链:
       Type:     DeFi 策士 (0)
       Nickname: 老炮
       Slogan:   穿越过三轮熊市。Aave 是家。
       Tx:       https://www.okx.com/web3/explorer/xlayer/tx/0xabcd...
       Dashboard: https://0xsoul.fun/u/0x8eb3...

    🔐 整个过程私钥**没出过 TEE 飞地**。
```

### 相遇 PK 场景

```
你: 找人打

AI: 当前可挑战目标（vault ≥ 1 USDT）：
    1. 0x1Cb8 · DeFi 策士 ◈ · vault 5 USDT · 0胜1负
    2. 0x4c8a · 沉默巨鲸 ◊ · vault 12 USDT · 8胜3负

你: 挑战第一个，押 1 USDT

AI: 🥊 提交挑战 → 一笔 tx 完成押注 + 链上随机 + 结算...

    🎉 你赢了！
    净赚 +1 USDT (2 USDT 已进你钱包)
    战绩: 2胜0负

    tx: https://www.okx.com/web3/explorer/xlayer/tx/0xa3ae...
```

---

## 🔐 你不用懂，但放心

| 担心         | 实际                                                |
| ------------ | --------------------------------------------------- |
| 私钥会泄漏吗 | ❌ 在 OKX TEE 安全飞地，OKX 自己也读不到            |
| 钱包会被盗吗 | ❌ 邮箱 OTP 每会话一次，过期需重 verify             |
| 平台抽水多少 | **0**（赢家拿走 2× 全部押注）                       |
| 合约会跑路吗 | ❌ 部署后 0 admin 函数，**immutable**，没人能改规则 |
| 能转 Pet 吗  | ❌ Soulbound ERC-721，永远属于召唤者                |
| 能随时退出吗 | ✅ 说"取出来"，USDT 直接回你钱包                    |
| Vault 上限   | 1,000 USDT / 地址 · 单场押注 1–10 USDT              |

---

## 🚨 故障恢复表（高可用版）

| 现象                                         | 根因                            | 一句话修复                                             |
| -------------------------------------------- | ------------------------------- | ------------------------------------------------------ |
| `/plugin install` 报 `Invalid plugin name`   | 早期版本 slug 以数字开头        | 用 `zerox-soul@zerox-soul`，不要用 `0xsoul`            |
| `/plugin install` 报 `marketplace not found` | 没跑 Step 2 第一行              | 先 `/plugin marketplace add dkypooh/zerox-soul-plugin` |
| AI 不识别"召唤"                              | plugin 没启用                   | `/plugin` 查看，必要时 `/plugin enable zerox-soul`     |
| AI 反复问邮箱                                | session 缓存丢失                | 重说"召唤"，session 会重新建立（< 5s）                 |
| OTP 收不到                                   | 邮箱过滤                        | 跟 AI 说"重发 OTP"，或检查垃圾邮箱                     |
| `Failed to estimate gas: AlreadySummoned`    | 该地址在 v0.8 合约已召唤过      | 说"看我的战绩"即可，**每地址一辈子只能召唤一次**       |
| `InvalidNicknameLength`                      | 昵称 > 32 字节（≈ 10 中文字）   | 缩短，或回复"用默认"走 `addr[0:6]`                     |
| `InvalidQuoteLength`                         | Slogan > 64 字节（≈ 20 中文字） | 缩短                                                   |
| `insufficient funds for gas`                 | 钱包没 OKB                      | 从 OKX 提 ≥ 0.001 OKB 到 XLayer 主网                   |
| `ERC20: insufficient allowance`              | USDT 还没 approve               | 说"存 X USDT"，AI 会自动 approve + deposit             |
| `InsufficientVaultBalance`                   | 对手 vault 没钱                 | 换个 vault 余额够的目标，或先存钱再被挑战              |
| Dashboard 一直转圈                           | RPC 偶发 502                    | 等 8 秒下次轮询自动恢复（stale-while-revalidate）      |
| Dashboard 显示空                             | 当前地址未召唤过 v0.8 合约      | 先在 Claude Code 里召唤，~30 秒后 Dashboard 出现       |

---

## ✨ 主网地址（信息透明 · v0.8）

```
CipherPetCore:  0xe639d8A5C3ABA8F74070BB2eA383b11CBc9568B7
USDT (stake):   0x779ded0c9e1022225f8e0630b35a9b54be713736
Chain:          XLayer mainnet (chainId 196)
RPC:            https://rpc.xlayer.tech
Explorer:       https://www.okx.com/web3/explorer/xlayer
ERC-721 name:   0xSoul
ERC-721 symbol: SOUL
```

任何人都能直接用 `cast` 验证：

```bash
cast call 0xe639d8A5C3ABA8F74070BB2eA383b11CBc9568B7 \
  "getPet(address)((uint8,uint32,string,string))" \
  <your_address> --rpc-url https://rpc.xlayer.tech
# 返回: (typeIdx, summonedAt, nickname, quote)
```

> **旧合约 `0x34dB...F9D9` (v0.7) 仍在链上**，但 Dashboard / Skill 已全部切到 v0.8 新合约。
> 历史数据可在浏览器查询，新流量不再写入。

---

## 📂 开发者向（普通用户跳过）

| 文件                                   | 说明                                    |
| -------------------------------------- | --------------------------------------- |
| `contracts/src/CipherPetCore.sol`      | 主合约（v0.8 含 `nickname`）            |
| `contracts/src/ICipherPetCore.sol`     | 接口 / Pet struct / errors / events     |
| `contracts/test/CipherPetCore.t.sol`   | 31 单元 + fuzz + 2 invariant，全过      |
| `contracts/deployment.json`            | 合约部署元信息                          |
| `skills/init/SKILL.md`                 | 环境检测 + 缓存（首次自动跑）           |
| `skills/summon/SKILL.md`               | 召唤主流程（含 v0.8 nickname 步骤）     |
| `skills/vault/SKILL.md`                | deposit / withdraw                      |
| `skills/challenge/SKILL.md`            | 相遇 PK                                 |
| `skills/stats/SKILL.md`                | 战绩查询                                |
| `skills/share/SKILL.md`                | 病毒分享素材                            |
| `dashboard/src/lib/contract.ts`        | viem 客户端 + ABI + 地址                |
| `dashboard/src/lib/useContractData.ts` | 8s 轮询 hooks（stale-while-revalidate） |

session 缓存路径：`~/.cipherpet/session.env`（地址 + 最后检查时间，TTL 1h）

---

**🎉 享受你的链上灵魂之旅。✦ 现在就跟 AI 说："召唤"。**

Built on OKX OnchainOS · XLayer · v0.4 · 2026-05-12
