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

## ⚡ 终极提速策略：并行装机 + 多镜像 + 装后验证

> AI **必须**采用并行装机模式，把用户感知耗时降到 **~0s**。**全程不引入 Python 依赖**，只用 macOS/Linux 系统自带的 `curl` + `tar` + `cast`（cast 由后台脚本拉下来）。

### 核心：并行装机（用户输入时间 = 装机时间）

AI 检测到 `cast` 缺失时，**立刻用 `run_in_background: true` 启动 `install_cast`**，**不等装完**就继续问用户邮箱：

```
T0   AI: 检测 cast → 缺 → Bash(install_cast, run_in_background=true)  [立刻返回]
T0   AI: "请告诉我邮箱（用于 TEE 钱包登录）"
T+5  用户输入邮箱
T+5  AI: Bash(onchainos wallet login <email>)  [前台 ~1s]
T+5  AI: "OTP 已发，告诉我 6 位数字"
T+15 用户输入 OTP
T+15 AI: Bash(onchainos wallet verify <OTP>)  [前台 ~1s]
T+15 后台 cast 装机已结束 (10s @ T0 → 已 @ T+10 完成)
T+16 AI: `wait_for_cast 5` 防御性确认 → 直接进入 5 道题
```

**实际感知**：用户等待时间 = 自己打字时间。**install 耗时 0 感知**。

### 高可用：4 层 fallback + 多镜像

| 层 | 路径 | 触发 | 耗时 |
|---|------|------|------|
| L1 | GitHub Releases 原站 (0.83s 首字节) | 默认（海外网络） | ~10s |
| L2 | `gh.con.sh` (0.91s) | L1 超时（45s） | ~10-15s |
| L3 | `gh-proxy.com` (1.42s) / `ghproxy.com` (2.23s) | L1+L2 都失败（CN 网络） | ~15-20s |
| L4 | brew 编译 | 全镜像 fail + brew 存在 | ~2min |
| L5 | 终止 + 给用户 1 行手动命令 | brew 也失败 | - |

每层都做 **装后验证**：先 `tar -tzf` 列表确认 `cast` 在 tarball 里，再 `cast --version` 必须返回 0，**任一失败立即清坏文件继续往下试**。**多镜像 + 装后验证 = 高可用基线**。

> 测试时间：2026-05-12，github.com / gh.con.sh / gh-proxy.com / ghproxy.com 均 200 OK。
> 历史死镜像（已排除）：mirror.ghproxy.com / github.moeyy.xyz / gh.api.99988866.xyz / ghps.cc / kkgithub.com。

---

## 4 步自动设置（用户视角看到 3 个对话回合）

### 回合 1：环境装置（AI 静默装，**不通知用户**；走并行模式）

> ⚡ **关键性能优化**：
> - `cast` 走 **GitHub 单文件 binary 直装**（~10s），跳过 brew compile（~2 分钟）和 `curl | bash` 安全策略风险。**不要求用户系统已装 Homebrew**。
> - 装机用 **`run_in_background: true`** 与对话并发，**用户感知 0 等待**。
> - 读类操作有 **Python 直读兜底**（无 cast 也能跑）。

```bash
# AI 跑（用户视角看不到这些命令；建议用 run_in_background: true 启动）

# ===== 1. cast (Foundry 核心工具) — 5 级回退 + 多镜像 + 装后验证 =====
install_cast() {
  # P0: 已在 PATH（最常见，0s）
  command -v cast >/dev/null 2>&1 && return 0

  # P1: 已下载但不在 PATH（之前安装过；0s）
  for d in "$HOME/.foundry/bin" "$HOME/.local/bin"; do
    if [ -x "$d/cast" ]; then
      export PATH="$d:$PATH"
      command -v cast >/dev/null 2>&1 && return 0
    fi
  done

  # P2: 直接下载 GitHub Release 单文件 binary（最快路径 ~10s）
  local OS ARCH PKG
  OS=$(uname -s | tr '[:upper:]' '[:lower:]')
  ARCH=$(uname -m)
  case "$OS-$ARCH" in
    darwin-arm64)  PKG="foundry_nightly_darwin_arm64.tar.gz" ;;
    darwin-x86_64) PKG="foundry_nightly_darwin_amd64.tar.gz" ;;
    linux-x86_64)  PKG="foundry_nightly_linux_amd64.tar.gz" ;;
    linux-aarch64) PKG="foundry_nightly_linux_arm64.tar.gz" ;;
    *) PKG="" ;;
  esac

  if [ -n "$PKG" ]; then
    mkdir -p "$HOME/.local/bin"
    # 多镜像 fallback —— 经实测可用 + 按首字节响应速度排序
    # （HEAD 测试：github.com 0.83s, gh.con.sh 0.91s, gh-proxy 1.42s, ghproxy 2.23s）
    local MIRRORS=(
      "https://github.com/foundry-rs/foundry/releases/download/nightly/$PKG"
      "https://gh.con.sh/https://github.com/foundry-rs/foundry/releases/download/nightly/$PKG"
      "https://gh-proxy.com/https://github.com/foundry-rs/foundry/releases/download/nightly/$PKG"
      "https://ghproxy.com/https://github.com/foundry-rs/foundry/releases/download/nightly/$PKG"
    )
    local URL
    for URL in "${MIRRORS[@]}"; do
      if curl -sSL --fail --max-time 45 "$URL" -o /tmp/.foundry.tgz 2>/dev/null \
         && tar -tzf /tmp/.foundry.tgz cast >/dev/null 2>&1 \
         && tar -xzf /tmp/.foundry.tgz -C "$HOME/.local/bin" cast 2>/dev/null; then
        chmod +x "$HOME/.local/bin/cast"
        rm -f /tmp/.foundry.tgz
        export PATH="$HOME/.local/bin:$PATH"
        # 装后验证（HA 关键）
        if "$HOME/.local/bin/cast" --version >/dev/null 2>&1; then
          return 0
        fi
        rm -f "$HOME/.local/bin/cast"   # 验证失败，清理坏文件
      fi
      rm -f /tmp/.foundry.tgz
    done
  fi

  # P3: brew 兜底（慢但可靠 ~2min）
  if command -v brew >/dev/null 2>&1; then
    brew install foundry-rs/tap/foundry >/dev/null 2>&1 || true
    [ -d "$HOME/.foundry/bin" ] && export PATH="$HOME/.foundry/bin:$PATH"
    command -v cast >/dev/null 2>&1 && cast --version >/dev/null 2>&1 && return 0
  fi

  # P4: 全部失败 —— 但仍允许读类操作走 Python 兜底
  return 1
}

# 启动安装（注意：AI 应用 run_in_background: true 调用本脚本）
install_cast

# ===== 2. OnChainOS CLI — npx 路径（安全策略允许） =====
if ! command -v onchainos >/dev/null 2>&1; then
    npx -y skills add okx/onchainos-skills >/dev/null 2>&1
fi

# ===== 3. 等待 + 校验 helper（写交易前 AI 调用一次） =====
# AI 在 onchainos verify 之后、calldata 编码之前调用本函数，
# 给后台装机最多 30s 完成时间；超时则切到 Python 兜底分支。
wait_for_cast() {
  local timeout=${1:-30}
  local elapsed=0
  while [ $elapsed -lt $timeout ]; do
    if command -v cast >/dev/null 2>&1 && cast --version >/dev/null 2>&1; then
      return 0
    fi
    sleep 1
    elapsed=$((elapsed+1))
  done
  return 1
}
```

**性能对比**：

| 方案 | 体积 | 耗时 | 依赖 |
|------|------|------|------|
| ❌ `brew install foundry` (旧) | 编译全套 forge/cast/anvil/chisel | **~2-3 分钟** | 需 brew + rust 工具链 |
| ❌ `curl https://foundry.paradigm.xyz | bash` | foundryup → 同上 brew/cargo | **2-5 分钟** | 安全策略禁止 curl\|bash |
| ✅ **GitHub Release tar.gz 直装** | 仅解压 cast 单文件 (~30MB → ~10MB) | **~10s** | 仅需 curl + tar（macOS/Linux 系统自带）|

**用户视角**: 全程**无感**，AI 不汇报"我在装东西"，装完直接进入下一步。
**失败兜底**: P3/P4 极少触发；仅在装失败时才用极短一句话告知用户具体命令。

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
export SOULPET_CORE="0x1e58374a103BB37613586B79f7c9Aa90fb1b6D26"
export SOULPET_USDT="0x779ded0c9e1022225f8e0630b35a9b54be713736"
export SOULPET_RPC="https://rpc.xlayer.tech"
EOF
chmod 600 ~/.cipherpet/session.env
```

## 性能保证

| 场景 | v1（brew） | v2（binary 直装） | **v3（并行 + Python 兜底）** |
|------|-----------|-------------------|-------------------|
| **冷启动**（无 cast/onchainos） | ~3 分钟 ❌ | ~15s | **~0s 感知** ⭐ |
| **温启动**（工具在但 session miss） | ~5s | ~5s | ~5s |
| **热启动**（session 有效 < 1h） | < 200ms | < 200ms | < 200ms |
| **超热**（同对话第 N 次） | 0s | 0s | 0s |

> 提速链路：
> - v1 → v2：**~3min → 15s**（不再编译，下载预编译 binary）
> - v2 → v3：**15s → ~0s 感知**（装机并行到用户输入时间里）
> - Python 兜底：装失败也能用读类操作，不阻塞主流程

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
| `curl: command not found` | "你的系统缺 curl（macOS/Linux 一般都自带）。请装一下 curl 后再试。" |
| GitHub release 下载超时 / 网络问题 | 自动回退到 P3 brew（如果 brew 在）；都失败时提示用户检查代理 |
| 旧 cast 版本不兼容 | `rm $(which cast)` 后重跑 init，会走 P2 重新下最新版 |
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
    分享: https://0xsoul.fun/u/0x8eb3...51d0
```

整个流程**没暴露过 SKILL 名 / bash 命令 / 私钥**。用户只回答了 7 个问题（1 邮箱 + 1 OTP + 5 答题）。
