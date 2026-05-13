# Shared: 合约地址 + 函数选择器（预编码）

> 所有 SKILL.md 引用此文件。**不要重复定义**。

## 合约地址（XLayer Mainnet）

```bash
export SOULPET_CORE="0x1e58374A103BB37613586B79f7c9aA90fb1b6d26"
export SOULPET_USDT="0x779ded0c9e1022225f8e0630b35a9b54be713736"
export SOULPET_RPC="https://rpc.xlayer.tech"
export SOULPET_CHAIN="xlayer"
export SOULPET_EXPLORER="https://www.okx.com/web3/explorer/xlayer"
```

## 函数选择器（预计算，避免运行时 keccak）

```
summon(uint8,string,string)         = 0xb434147f   // v0.8: (typeIdx, nickname, quote)
deposit(uint128)                    = 0x0efe6a8b
withdraw(uint128)                   = 0x2e1a7d4d
challenge(address,uint128)          = 0xa6dfe0e5
getPet(address)                     = 0xa3eaf7e2
hasSummoned(address)                = 0xc4a02ef5
getBalance(address)                 = 0xf8b2cb4f
getVault(address)                   = 0xab93f5e9
getWinRate(address)                 = 0xfe1d5b27
getRecentBattles(uint256,uint256)   = 0xe6d2c2c4

// ERC-20
approve(address,uint256)            = 0x095ea7b3
transfer(address,uint256)           = 0xa9059cbb
balanceOf(address)                  = 0x70a08231
allowance(address,address)          = 0xdd62ed3e
```

## Summon calldata（v0.8 新签名）

> v0.8 起 summon 多了 nickname 字段，硬编码 calldata 已废弃（昵称是用户输入，必须现编）。
> 性能影响可忽略（cast 启动 ~30ms 单次，每地址只 summon 一次）。

```bash
CALLDATA=$(cast calldata "summon(uint8,string,string)" "$TYPE_IDX" "$NICKNAME" "$SLOGAN")
```

## Session 缓存路径

```bash
export SOULPET_HOME="$HOME/.cipherpet"
export SOULPET_SESSION="$SOULPET_HOME/session.env"
mkdir -p "$SOULPET_HOME"
```

## 加载缓存的标准片段

每个 SKILL.md 在 Step 0 加这一段：

```bash
# 加载之前缓存的 session（地址、上次环境检查时间等）
if [ -f "$HOME/.cipherpet/session.env" ]; then
    source "$HOME/.cipherpet/session.env"
fi
```
