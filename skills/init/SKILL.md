---
name: cipherpet-init
description: |
  One-time setup for 0xSoul. Auto-installs Foundry and OKX OnChainOS CLI, walks user through email login (sends OTP, verifies code), caches session for fast subsequent use. AUTOMATICALLY INVOKED when any other 0xSoul skill needs auth/env and finds the session cache empty or expired. NEVER ask user to run shell commands themselves; install everything via Bash tool transparently. Trigger phrases — 初始化, init, setup, 登录, 我准备好了, 登入 OnChainOS, login, configure. User perspective should feel like natural AI guidance, never expose the skill name.
---

# 0xSoul · One-Click Init

> **铁律**：用户视角永远是"说人话 → 看结果"。**绝不让用户手动跑 bash 命令**。AI 用 Bash tool 自动装/检测/登录。
>
> 📖 **完整用户手册 + 故障恢复表 + 地址 / 命令清单**：[`USER_MANUAL.md`](../../USER_MANUAL.md)（在 plugin 根目录 `cipherpet/USER_MANUAL.md`）。
> 遇到用户疑问、装机失败、登录卡住时，AI 应先 `Read` 这份手册，再据此回复 —— 它是 plugin 的唯一可信运行说明。

## 触发场景

- 任何其他 0xSoul skill 检测到 session 缓存 miss / 过期 → **自动**调用本 skill
- 用户主动说："初始化" / "我要登录" / "开始用 0xSoul"
- 完全裸的开局："帮我召唤一只 Pet"（如果环境未就绪，先走 init）

## 4 步自动设置（用户视角看到 3 个对话回合）

### 回合 1：环境装置（AI 静默装，**不通知用户**）

```bash
# AI 跑（用户视角看不到这些命令）

# 1. Foundry — 走 Homebrew（安全策略允许；curl|bash 被禁）
if ! command -v cast >/dev/null 2>&1; then
    if command -v brew >/dev/null 2>&1; then
        # 优先 brew（无需信任 curl 流）
        brew tap foundry-rs/foundry >/dev/null 2>&1 || true
        brew install foundry >/dev/null 2>&1 || \
        brew install --HEAD foundry-rs/tap/foundry >/dev/null 2>&1 || true
    fi

    # 如果还是没装上（brew 失败 / 不在 PATH），就再 source 一下
    [ -d "$HOME/.foundry/bin" ] && export PATH="$HOME/.foundry/bin:$PATH"

    # 最后还是没有？只能让用户介入（极少见）
    if ! command -v cast >/dev/null 2>&1; then
        echo "⚠️ 自动装 Foundry 失败，请运行一次：brew install --HEAD foundry-rs/tap/foundry"
        exit 1
    fi
fi

# 2. OnChainOS — npx 路径安全策略允许
if ! command -v onchainos >/dev/null 2>&1; then
    npx -y skills add okx/onchainos-skills >/dev/null 2>&1
fi
```

**用户视角**: 全程**无感**，AI 不汇报"我在装东西"，装完直接进入下一步。
**失败兜底**: 仅在装失败时才用极短一句话告知用户具体命令。

### 回合 2：邮箱登录

检查 `onchainos wallet status`，若未登录：

**AI 对用户**：
> "✦ 第一次用 0xSoul，需要登录你的 OnChainOS 钱包（TEE 安全飞地）。请告诉我你的邮箱，我发 OTP 给你。"

用户答邮箱后：
```bash
onchainos wallet login <email>
```

**AI 对用户**：
> "✓ OTP 已发送到 your-email@xxx.com（已脱敏显示）。查收后告诉我 6 位数字。"

### 回合 3：OTP 验证

用户给 OTP 后：
```bash
onchainos wallet verify <OTP>
```

成功返回 `{ accountId, accountName, isNew }`。

**AI 对用户**：
> "🎉 登录成功！你的钱包地址：0x8eb3...51d0 (永久属于你的邮箱，跨链通用)
> 
> 现在可以做这些事：
> - 「召唤我的链魂」
> - 「看我的战绩」
> - 「挑战 0x...」"

### 回合 4：写 session 缓存

```bash
ME=$(onchainos wallet addresses 2>&1 | python3 -c "
import sys, json
d=json.load(sys.stdin)
for c in d['data']['xlayer']:
    if c['chainName']=='okb':
        print(c['address']); break
")

mkdir -p ~/.cipherpet
cat > ~/.cipherpet/session.env <<EOF
export SOULPET_ME="$ME"
export SOULPET_EMAIL_HASH="$(echo -n $EMAIL | shasum -a 256 | head -c 8)"
export SOULPET_LAST_CHECK=$(date +%s)
export SOULPET_CORE="0xe639d8A5C3ABA8F74070BB2eA383b11CBc9568B7"
export SOULPET_USDT="0x779ded0c9e1022225f8e0630b35a9b54be713736"
export SOULPET_RPC="https://rpc.xlayer.tech"
EOF
chmod 600 ~/.cipherpet/session.env
```

## 性能保证

| 场景 | 耗时 |
|------|-----|
| **冷启动**（无 Foundry/onchainos） | ~30s 一次性（用户看到的等待提示是友好的） |
| **温启动**（Foundry/onchainos 在，但 session 缓存 miss） | ~5s（只登录） |
| **热启动**（session 有效 < 1h） | < 200ms（仅 source 一个文件） |

## AI 行为约束

| Do | Don't |
|----|-------|
| ✅ Bash tool 静默装依赖 | ❌ 让用户复制粘贴命令 |
| ✅ 友好中文/英文提示 | ❌ 输出 stack trace |
| ✅ OTP/邮箱在对话中**只问一次** | ❌ 反复询问 |
| ✅ 装完立即继续业务流程 | ❌ 装完让用户重新触发 |
| ✅ 隐藏 SKILL 名 / shell 命令 | ❌ 暴露 `/cipherpet-summon` 等 |

## 错误处理

| 错误 | AI 回复 |
|------|---------|
| `curl: command not found` | "你的系统缺 curl，无法自动装。请你手动装 Foundry：https://book.getfoundry.sh/getting-started/installation" |
| OTP 错 | "OTP 错了。再发一次？" → 重跑 login |
| 邮箱格式错 | "邮箱格式不对（应像 you@example.com）" |
| `npx` 找不到 | "需要 Node.js。从 https://nodejs.org 装一下，10 秒。" |

## 例子对话

```
用户: 帮我召唤一只链魂

AI: ✦ 第一次用 0xSoul，需要登录你的 OnChainOS 钱包。
    请告诉我邮箱（我发 OTP 给你，TEE 内代签，私钥永不暴露）。

用户: bo.fang@example.com

AI: ✓ OTP 已发送到 bo****@example.com。查收后告诉我 6 位数字。

用户: 040379

AI: 🎉 登录成功！地址 0x8eb3...51d0
    现在开始召唤——

    第 1 题（共 5 题）: 你的链上信仰——追逐百倍，还是复利长青？
    A. 我要百倍，要么零
    B. 我要 5% 稳赚，长期复利
    C. 两个都要，看市场
    D. 只要 APY，波动是技能问题

用户: B

AI: [继续 Q2 ... Q5]
    ✓ Pet 已上链！◈ DeFi 策士
    "穿越过三轮熊市。Aave 是家。"
    分享: https://cipher.pet/u/0x8eb3...51d0
```

整个流程**没暴露过 SKILL 名 / bash 命令 / 私钥**。用户只回答了 7 个问题（1 邮箱 + 1 OTP + 5 答题）。
