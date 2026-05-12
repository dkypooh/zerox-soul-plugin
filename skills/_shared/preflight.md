# Shared Preflight — 一次性环境就绪

> 所有 0xSoul skill 在执行前**必须**走这套检测。
> **关键性能优化**: session 缓存在 `~/.cipherpet/session.env`，命中后整段跳过（< 5ms vs 冷启动 30s）。

## 一句话决策树

```
读 ~/.cipherpet/session.env
  ├── 命中 + 未过期 → 直接进业务逻辑 ✓
  └── miss/过期 → 走下面 4 步首次设置
```

## 首次设置（4 步，自动化）

每一步都用 Bash tool 跑。**用户视角永远是「说人话 → 看结果」**，绝不让用户手动跑命令。

### Step 1 — 检查 `cast` (Foundry)
```bash
if ! command -v cast >/dev/null 2>&1; then
    # 友好提示后自动装
    echo "首次使用需装 Foundry (10s)..."
    curl -L https://foundry.paradigm.xyz | bash >/dev/null 2>&1
    source ~/.zshenv
    foundryup >/dev/null 2>&1
fi
```

### Step 2 — 检查 `onchainos`
```bash
if ! command -v onchainos >/dev/null 2>&1; then
    npx -y skills add okx/onchainos-skills >/dev/null 2>&1
fi
```

### Step 3 — 检查登录状态
```bash
STATUS=$(onchainos wallet status 2>&1 || echo "")
if ! echo "$STATUS" | grep -q "accountId"; then
    # 需要登录
    echo "请告诉我邮箱（用于 OnChainOS TEE 钱包认证）"
    # 用户答邮箱 -> onchainos wallet login <email>
    # 用户答 OTP -> onchainos wallet verify <code>
fi
```

### Step 4 — 拉用户地址 + 写缓存
```bash
ME=$(onchainos wallet addresses 2>&1 | python3 -c "
import sys, json
try:
    d=json.load(sys.stdin)
    for c in d['data']['xlayer']:
        if c['chainName']=='okb':
            print(c['address']); break
except: pass
")

mkdir -p ~/.cipherpet
cat > ~/.cipherpet/session.env <<EOF
# 0xSoul session cache — last refreshed $(date +%s)
export SOULPET_ME="$ME"
export SOULPET_LAST_CHECK=$(date +%s)
EOF
chmod 600 ~/.cipherpet/session.env
```

## 命中缓存的快速路径

```bash
source ~/.cipherpet/session.env 2>/dev/null
NOW=$(date +%s)
AGE=$((NOW - SOULPET_LAST_CHECK))

# 缓存有效期 1 小时（避免每次都跑 onchainos status 检查）
if [ -n "$SOULPET_ME" ] && [ "$AGE" -lt 3600 ]; then
    # ✓ 缓存命中，直接用 $SOULPET_ME
    :
else
    # 缓存过期或没有 → 重跑首次设置
    :
fi
```

## 用户视角的体验目标

| 场景 | 用户操作 | 实际耗时 |
|------|--------|----------|
| 首次召唤 | "召唤" → 答邮箱 → 答 OTP → 答 5 题 | ~60s |
| 第 2 次操作（任何 skill） | 直接说人话 | < 3s（缓存命中） |
| 会话重启后 | 还是直接说人话 | < 5s（缓存有效就免登录） |

## 标准 Preflight Bash 块（复制到每个 SKILL.md 的开头）

```bash
# === 0xSoul Shared Preflight ===
mkdir -p ~/.cipherpet
source ~/.cipherpet/session.env 2>/dev/null || true

if [ -z "$SOULPET_ME" ] || [ $(($(date +%s) - ${SOULPET_LAST_CHECK:-0})) -gt 3600 ]; then
    # 调用 cipherpet-init skill 走完整流程
    # AI 视角：检测到缓存 miss，主动跑 init
    :
fi
# === End Preflight ===
```
