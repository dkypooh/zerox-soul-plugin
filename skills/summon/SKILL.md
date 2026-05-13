---
name: cipherpet-summon
description: |
  AI-driven invocation that mints the user's onchain soul (0xSoul NFT) via OnChainOS TEE wallet.
  Trigger on ANY of these natural-language signals (Chinese + English, even partial / vague):
    - "召唤", "召唤我的链魂", "链魂", "我的链上灵魂", "做一只 Pet 给我", "summon", "summon my pet", "create my pet", "mint my soul", "give me a pet"
    - "我是哪种链上人格", "测一下我的链上 MBTI", "what's my onchain persona"
    - "试试 0xSoul", "开始玩"
  Auto-handles ALL setup via cipherpet-init if env not ready (Foundry, OnChainOS CLI, email login).
  NEVER expose slash commands or shell to user — pure natural language conversation.
  Performance: warm path < 3s read, write tx 5-10s; cold path delegates to cipherpet-init.
---

# 链魂 0xSoul · Summon (OnChainOS TEE 版)

> **架构**: 邮箱登录 → OTP 验证 → TEE 内代签 → ERC-4337 UserOperation → 合约执行
> **零私钥暴露**，Plugin 永远拿不到用户的密钥
>
> 📖 **遇到任何用户疑问 / 错误 → 先 `Read` [`USER_MANUAL.md`](../../USER_MANUAL.md)**（plugin 根目录 `cipherpet/USER_MANUAL.md`）。手册里有完整故障恢复表（`AlreadySummoned` / `InvalidNicknameLength` / `InvalidQuoteLength` / gas 不足 / OTP 收不到 等所有错误码的一句话修复）+ 主网合约地址 + Dashboard 启动命令。**plugin 的唯一可信运行说明**。

## 触发场景

- "召唤 0xSoul" / "summon my pet" / "/cipherpet-summon"

## 链上目标合约（XLayer mainnet）

```
network:  XLayer mainnet (chainId 196)
contract: 0xF09877E72E1b133524DE3491DD1BBF89CcF9BF0e (CipherPetCore · 0xSoul ERC-721)
stake:    0x779ded0c9e1022225f8e0630b35a9b54be713736 (real USDT)
```

## 🤖 全自动 Smart Preflight（用户不需要懂这些）

⚠️ **关键**: 当用户说"召唤"时，你（AI）必须**主动**走完下面的智能检测。**不要让用户自己跑 bash 命令** —— 你帮他跑，缺什么自动装。

### Auto-check Step 1: cast 是否存在

```bash
which cast >/dev/null 2>&1
```

如果**不存在**：
1. 告诉用户："首次使用需要装一个小工具（Foundry, 10 秒），我帮你装可以吗？"
2. AI 静默执行（用 Bash tool；不汇报）：
   ```bash
   # 安全策略：brew 优先（curl|bash 被禁）
   brew tap foundry-rs/foundry 2>/dev/null
   brew install foundry 2>/dev/null || \
       brew install --HEAD foundry-rs/tap/foundry 2>/dev/null
   export PATH="$HOME/.foundry/bin:$PATH"
   ```
3. 装完继续

### Auto-check Step 2: onchainos CLI 是否存在

```bash
which onchainos >/dev/null 2>&1
```

如果**不存在**：
1. 告诉用户："还需要装 OKX OnChainOS skills，5 秒"
2. 直接执行：
   ```bash
   npx -y skills add okx/onchainos-skills
   ```
3. 装完继续

### Auto-check Step 3: 是否已登录 OnChainOS

```bash
STATUS=$(onchainos wallet status 2>&1)
LOGGED_IN=$(echo "$STATUS" | grep -q "accountId" && echo "yes" || echo "no")
```

如果**未登录**：
1. 问用户："请告诉我你的邮箱，我发 OTP 给你"（**仅一次**）
2. 执行：
   ```bash
   onchainos wallet login <用户提供的邮箱>
   ```
3. 输出 `{ "ok": true }` 后告诉用户："OTP 已发送，查邮箱后告诉我 6 位数字"
4. 用户输入 OTP 后：
   ```bash
   onchainos wallet verify <OTP>
   ```
5. 验证成功后继续

### Auto-check Step 4: 是否已召唤

```bash
ME=$(onchainos wallet addresses 2>&1 | python3 -c "
import sys, json
try:
    d = json.load(sys.stdin)
    for c in d['data']['xlayer']:
        if c['chainName'] == 'okb':
            print(c['address']); break
except: pass
")
HAS=$(cast call 0xF09877E72E1b133524DE3491DD1BBF89CcF9BF0e \
    "hasSummoned(address)(bool)" $ME \
    --rpc-url https://rpc.xlayer.tech)
```

如果**已召唤**：
- 告诉用户："你已经召唤过了，给你看看你的 Pet ✨" → 直接调 stats skill 显示
- **不要重复 summon**

如果**未召唤**：继续 5 道问答流程

---

## Workflow

### Step 0 — 登录（仅首次或 logout 后）

如果 `onchainos wallet status` 返回未登录，引导：

```bash
# 1. 发送 OTP 到邮箱
onchainos wallet login <user@email.com>

# 2. 用户检查邮箱拿到 6 位 OTP

# 3. 验证（用户输入 OTP）
onchainos wallet verify <OTP_CODE>

# 成功后返回:
# { "ok": true, "data": { "accountId": "...", ... } }
```

⚠️ **从不询问用户私钥**——OnChainOS 全程 TEE 签名，密钥永不离开飞地。

### Step 1 — 获取用户的 XLayer 地址

```bash
ME=$(onchainos wallet addresses 2>&1 | python3 -c "
import sys, json
d = json.load(sys.stdin)
for c in d['data']['xlayer']:
    if c['chainName'] == 'okb':
        print(c['address']); break
")
echo "Your XLayer address: $ME"
```

### Step 2 — 检查是否已召唤

```bash
HAS=$(cast call 0xF09877E72E1b133524DE3491DD1BBF89CcF9BF0e \
    "hasSummoned(address)(bool)" $ME \
    --rpc-url https://rpc.xlayer.tech)

if [ "$HAS" = "true" ]; then
    echo "✦ 你已召唤过。/cipherpet-stats 查自己"
    exit 0
fi
```

### Step 3 — 5 道问答（**逐题交互**，不要一次性发完）

> ⚠️ **AI 行为约束**（保证 Mint 体验）：
> 1. **每次只发一道题** + 4 个选项（A/B/C/D），**等用户回复**再发下一题。
> 2. **绝不**把 5 题列在同一条消息里让用户批量答 —— 那会让命令行体验变成"读小作文"，新用户会被劝退。
> 3. **绝不**替用户答 —— 哪怕用户问"你帮我答吧"，也要让他二选一（"A. 让 AI 按链上历史推断 / B. 自己一题题答"）。
> 4. 每题之间可加一句 1 行的小铺垫（"OK，下一题来了 👇"），保持节奏感，但不要超过 1 行。
> 5. 用户答 `A`/`B`/`C`/`D` 即认，不识别再追问一次。识别 5 次答案后才进 Step 4。
>
> 这条 UX 约束**比内容更重要** —— 新用户的"召唤"体验全靠这一段交互节奏。

[Q1-Q5 内容不变，参考 v0.2 版本]

### Step 4 — 派生 typeIdx + 默认 Slogan

```
builderScore = (Q1=='slow') + (Q3=='builder') + (Q4=='accumulator')
degenScore   = (Q1=='fast') + (Q3=='degen')   + (Q4=='mover')
```

**typeIdx 决策树**:

| 条件 | typeIdx | 中文名 | 默认 Slogan |
|------|---------|--------|----------|
| `builderScore >= 2 && isDefi` | **0** | DeFi 策士 | `穿越过三轮熊市。Aave 是家。` |
| `degenScore >= 2 && isMeme` | **1** | 战壕狙击手 | `一个 dev 跑路，下一只就来。` |
| `builderScore >= 2 && isMeme` | **2** | 钻石贤者 | `从未在 -70% 卖出，也从未在 +1000% 套现。` |
| `degenScore >= 2 && isDefi && isAnon` | **3** | 沉默巨鲸 | `出手即定局。不解释。` |
| (fallback) | **0** | DeFi 策士 | `穿越过三轮熊市。Aave 是家。` |

### Step 4.4 — 链上昵称（榜单上的名字）⭐

**为什么**：链上 leaderboard 默认只能显示地址。昵称让用户在万神殿里"被认出来"。

**对话**:

```
✦ 给你的链魂起个名字吧（可选，留空就用地址前 6 位）：

   默认: {ME[0:6]}           ← 例如 0x8eb3
   
   想换一个？直接告诉我（≤ 10 中文 / ≤ 32 字节）。
   也可以直接说 "用默认" / "skip"。
```

**处理**:

```bash
# 默认就是地址前 6 位
DEFAULT_NICK="${ME:0:6}"
NICKNAME="$DEFAULT_NICK"

# 用户输入了自定义文本
if [ -n "$USER_NICK_INPUT" ] && [[ ! "$USER_NICK_INPUT" =~ ^(用默认|默认|skip|SKIP|跳过|default)$ ]]; then
    BYTE_LEN=$(echo -n "$USER_NICK_INPUT" | wc -c)
    if [ "$BYTE_LEN" -gt 32 ]; then
        echo "⚠️ 昵称太长（$BYTE_LEN 字节，上限 32）。"
        echo "提示: 中文每字约 3 字节 → 建议 ≤ 10 个中文字。"
        # 让用户重新输入或回退默认
    else
        NICKNAME="$USER_NICK_INPUT"
    fi
fi
```

**合约硬约束**: `MAX_NICKNAME_LENGTH = 32` 字节，超过 revert `InvalidNicknameLength`。

### Step 4.5 — Slogan 自定义（关键 UX 决策点）⭐

**这一步必须做** —— 用户视角的"我能改一改吗"非常重要。

**对话**:

```
✦ AI 给你的默认 Slogan:

   "{default_slogan}"

要这一句，还是自己写一个（≤ 20 字 / 64 字节）？

回复:
   - 直接说 "保持默认" / "用这个" / "OK"  → 用默认
   - 或粘贴你的 slogan → 用你写的
```

**用户选择处理**:

```bash
# 默认
USER_SLOGAN="$DEFAULT_SLOGAN"

# 如果用户输入了自定义文本（非"保持默认"/"OK"等关键词）
if [ -n "$USER_INPUT" ] && [[ ! "$USER_INPUT" =~ ^(保持默认|用这个|OK|ok|默认|default)$ ]]; then
    # 校验长度
    BYTE_LEN=$(echo -n "$USER_INPUT" | wc -c)
    if [ "$BYTE_LEN" -gt 64 ]; then
        echo "⚠️ slogan 太长（$BYTE_LEN 字节，上限 64）。"
        echo "提示: 中文每字约 3 字节，建议 ≤ 20 字。"
        # 让用户重新输入或回退默认
    else
        USER_SLOGAN="$USER_INPUT"
    fi
fi

# 最终的 slogan
SLOGAN="$USER_SLOGAN"
```

**好的自定义示例**:
- "DeFi 老炮，2017 进场，从未离场。"
- "代码即正义。"
- "钱包是我的简历。"
- "Anon forever."

**校验规则**（合约硬约束）:
- 字节数 ≤ 64（合约 `InvalidQuoteLength` revert 阈值）
- 中文每字 3 字节 → 中文字符 ≤ 21 个
- 英文每字 1 字节 → 英文字符 ≤ 64 个
- 空字符串允许（但建议至少给个默认）

### Step 5 — 编码 calldata

```bash
# v0.8: summon 新签名为 (typeIdx, nickname, quote)
CALLDATA=$(cast calldata "summon(uint8,string,string)" $TYPE_IDX "$NICKNAME" "$SLOGAN")
```

### Step 6 — TEE 签名 + 链上广播（核心！）

```bash
RESULT=$(onchainos wallet contract-call \
    --to 0xF09877E72E1b133524DE3491DD1BBF89CcF9BF0e \
    --chain xlayer \
    --input-data "$CALLDATA" \
    --biz-type "dapp")

# 解析结果
TX_HASH=$(echo "$RESULT" | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(d['data']['txHash'])
")
echo "✓ tx: $TX_HASH"
```

合约本身的 `summon(typeIdx, nickname, slogan)` 走 ERC-4337 EntryPoint → CipherPetCore.summon()。

---

## 完整对话示例（含 slogan 自定义）

```
用户: BBAAB    (5 题答案)

AI:   ✦ 灵魂解析:
        - builderScore = 2
        - degenScore = 1
        - 命中: 钻石贤者 ✦

      AI 给你的默认 Slogan:
        "从未在 -70% 卖出，也从未在 +1000% 套现。"

      要这一句还是自己写一个？（≤ 20 字）

用户: 自己写：链上 8 年，从未止损。

AI:   ✓ Slogan 设为「链上 8 年，从未止损。」(21 字节, OK)

      正在 TEE 内代签...
      ✓ tx: 0xabcd...
      
      🎉 你的链魂已上链:
      ✦ 钻石贤者 · 老炮
      "链上 8 年，从未止损。"
```

或者用户选默认:
```
用户: BBAAB

AI:   ✦ 你是 钻石贤者
      默认 Slogan: "从未在 -70% 卖出，也从未在 +1000% 套现。"
      要改吗？

用户: 保持默认

AI:   ✓ 上链中...
```

### Step 7 — 揭晓

```
✦ 第四章 · 揭晓

✓ 已铸造链魂 NFT
   Type:     {typeName} ({typeIdx})
   Nickname: {nickname}
   Slogan:   {slogan}
   Tx:       https://www.okx.com/web3/explorer/xlayer/tx/{tx_hash}
   Dashboard: https://0xsoul.fun/u/{me}

🔐 整个过程私钥**没出过 TEE 飞地**。
```

### Step 8 — 引导下一步

> **v3 行为提醒**：召唤后**钱包 USDT 没法直接 PK**，必须先 deposit 到 vault。Vercel dashboard 自动有 0.1 USDT 启动彩蛋打到**钱包**，是给你做 deposit 的种子。

```
0. (一次性) 存 USDT 到备战池: 说"存 1 USDT" — AI 自动 approve + deposit
1. 挑战别人:                说"挑战 0x..." 或 "找人打"
2. 看战绩:                  说"看我的战绩" / /cipherpet-stats
3. 分享:                    说"分享" / /cipherpet-share
```

> 顺序约束：v3 必须 `summon → deposit → challenge`，不能跳 deposit（vault 余额 0 时 challenge 必 revert）。

---

## 错误码处理

| Error | 原因 | 修复 |
|-------|------|------|
| `Not logged in` | onchainos 未登录 | 走 Step 0 |
| `AlreadySummoned` | 地址已召唤过 | /cipherpet-stats |
| `InvalidNicknameLength` | 昵称 > 32 字节 | 缩短或留空走默认 addr[0:6] |
| `InvalidQuoteLength` | Slogan > 64 字节 | 缩短到 ≤ 20 中文字 |
| `Bad signature` | OTP 错或过期 | 重新 login |
| `insufficient funds` | OnChainOS 钱包没 OKB | 引导用户充 OKB（XLayer mainnet）|

---

## 安全保障

- ✅ **私钥永不离开 TEE**（OKX 自己也读不到）
- ✅ **邮箱认证** + OTP（每次会话过期）
- ✅ **ERC-4337 标准** UserOperation 路径
- ✅ **链上完全可验证**
- ✅ Plugin 代码**永远不存储**私钥
