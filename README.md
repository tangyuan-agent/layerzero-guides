# LayerZero OFT 跨链转账指南（CLI 版本）

## 场景：从 Polygon 转账 3 USDT0 到 Arbitrum

纯命令行操作，使用 `cast` 和 `curl`，无需写脚本。

---

## 1. 基础信息

**合约地址（USDT0）**：
```bash
USDT0_POLYGON="0xf7C260136176100CE2A4Faf70045D3A0fB6fB86E"
USDT0_ARBITRUM="0xf7C260136176100CE2A4Faf70045D3A0fB6fB86E"
```

**LayerZero Endpoint ID**：
- Polygon: `109`
- Arbitrum: `110`

**RPC 端点**：
```bash
POLYGON_RPC="https://polygon-rpc.com"
ARBITRUM_RPC="https://arb1.arbitrum.io/rpc"
```

**金额**：
- 3 USDT0 = `3000000` (6 decimals)

---

## 2. 数据结构

### SendParam 结构

```solidity
struct SendParam {
    uint32 dstEid;          // 目标链 Endpoint ID
    bytes32 to;             // 目标地址（bytes32 格式）
    uint256 amountLD;       // 转账金额（本地精度）
    uint256 minAmountLD;    // 最小接收金额（滑点保护）
    bytes extraOptions;     // Gas 配置
    bytes composeMsg;       // 组合消息（通常为空）
    bytes oftCmd;           // OFT 命令（通常为空）
}
```

### MessagingFee 结构

```solidity
struct MessagingFee {
    uint256 nativeFee;      // 原生代币费用（如 POL）
    uint256 lzTokenFee;     // ZRO token 费用（通常为 0）
}
```

---

## 3. 核心函数签名

```solidity
// 查询对等合约地址
function peers(uint32 eid) external view returns (bytes32 peer);

// 查询费用
function quoteSend(
    SendParam calldata _sendParam,
    bool _payInLzToken
) external view returns (MessagingFee memory fee);

// 发送跨链交易
function send(
    SendParam calldata _sendParam,
    MessagingFee calldata _fee,
    address _refundAddress
) external payable returns (
    MessagingReceipt memory msgReceipt,
    OFTReceipt memory oftReceipt
);
```

---

## 4. 步骤详解

### Step 0: 设置环境变量

```bash
# 合约地址
export USDT0="0xf7C260136176100CE2A4Faf70045D3A0fB6fB86E"

# RPC
export POLYGON_RPC="https://polygon-rpc.com"
export ARBITRUM_RPC="https://arb1.arbitrum.io/rpc"

# 你的钱包地址
export MY_ADDRESS="0xYourAddressHere"

# 私钥（⚠️ 小心保管）
export PRIVATE_KEY="0xYourPrivateKeyHere"
```

### Step 1: 检查余额

```bash
# 检查 Polygon 上的 USDT0 余额
cast call $USDT0 \
  "balanceOf(address)(uint256)" \
  $MY_ADDRESS \
  --rpc-url $POLYGON_RPC

# 输出示例: 10000000 (10 USDT0)
```

### Step 2: 授权（如果还没授权）

```bash
# 授权 USDT0 合约操作你的代币（只需做一次）
cast send $USDT0 \
  "approve(address,uint256)" \
  $USDT0 \
  $(cast max-uint) \
  --private-key $PRIVATE_KEY \
  --rpc-url $POLYGON_RPC
```

### Step 3: 地址转换为 bytes32

```bash
# Cast 内置转换工具
MY_ADDRESS_BYTES32=$(cast --to-bytes32 $MY_ADDRESS)
echo $MY_ADDRESS_BYTES32

# 输出示例: 0x000000000000000000000000abcdefabcdefabcdefabcdefabcdefabcdefabcd
```

### Step 4: 构建 extraOptions

LayerZero 使用紧凑的 hex 编码设置 Gas：

**方式 1: 手动构建**
```bash
# 200,000 gas (0x030d40)
EXTRA_OPTIONS="0x00030100110100000000000000000000000000030d40"
```

**常用 Gas 设置**：
- 简单转账: `200,000` = `0x030d40`
- 复杂交互: `500,000` = `0x07a120`

### Step 5: 查询对等合约（可选）

```bash
# 查询 Arbitrum 上的对等合约地址
cast call $USDT0 \
  "peers(uint32)(bytes32)" \
  110 \
  --rpc-url $POLYGON_RPC

# 输出示例: 0x000000000000000000000000f7c260136176100ce2a4faf70045d3a0fb6fb86e
# 这是 Arbitrum 上的 USDT0 地址
```

### Step 6: 估算跨链费用

**函数签名**:
```solidity
function quoteSend(
    SendParam calldata _sendParam,
    bool _payInLzToken
) external view returns (MessagingFee memory)
```

**Cast 调用**:
```bash
# 调用 quoteSend 查询费用
# 返回值是 MessagingFee struct: (nativeFee, lzTokenFee)
cast call $USDT0 \
  "quoteSend((uint32,bytes32,uint256,uint256,bytes,bytes,bytes),bool)((uint256,uint256))" \
  "(110,$MY_ADDRESS_BYTES32,3000000,3000000,$EXTRA_OPTIONS,0x,0x)" \
  false \
  --rpc-url $POLYGON_RPC

# 输出示例: ((500000000000000000,0))
# nativeFee = 500000000000000000 (0.5 POL)
# lzTokenFee = 0
```

**提取 nativeFee**:
```bash
# 使用 awk 提取第一个值（nativeFee）
RESULT=$(cast call $USDT0 \
  "quoteSend((uint32,bytes32,uint256,uint256,bytes,bytes,bytes),bool)((uint256,uint256))" \
  "(110,$MY_ADDRESS_BYTES32,3000000,3000000,$EXTRA_OPTIONS,0x,0x)" \
  false \
  --rpc-url $POLYGON_RPC)

# 移除外层括号和内层括号，提取第一个数字
NATIVE_FEE=$(echo $RESULT | sed 's/((//;s/))//' | awk -F',' '{print $1}')

echo "Native Fee: $NATIVE_FEE wei"
echo "Native Fee in POL: $(cast --from-wei $NATIVE_FEE)"
```

### Step 7: 发送跨链交易

**函数签名**:
```solidity
function send(
    SendParam calldata _sendParam,
    MessagingFee calldata _fee,
    address _refundAddress
) external payable returns (MessagingReceipt, OFTReceipt)
```

**Cast 调用**:
```bash
cast send $USDT0 \
  "send((uint32,bytes32,uint256,uint256,bytes,bytes,bytes),(uint256,uint256),address)" \
  "(110,$MY_ADDRESS_BYTES32,3000000,3000000,$EXTRA_OPTIONS,0x,0x)" \
  "($NATIVE_FEE,0)" \
  "$MY_ADDRESS" \
  --value $NATIVE_FEE \
  --private-key $PRIVATE_KEY \
  --rpc-url $POLYGON_RPC \
  --legacy
```

**输出示例**：
```
blockHash               0x1234...
blockNumber             12345678
transactionHash         0xabcd...
status                  1 (success)
```

### Step 8: 跟踪消息状态

访问 [LayerZero Scan](https://layerzeroscan.com/)，输入交易哈希：

```bash
# 直接在浏览器打开（Linux）
xdg-open "https://layerzeroscan.com/tx/0xYourTxHash"
```

或者使用 LayerZero API：
```bash
TX_HASH="0xYourTxHash"

curl -s "https://api-mainnet.layerzero-scan.com/tx/$TX_HASH" | jq '.'
```

---

## 5. 完整一键脚本

将以下命令保存为 `bridge-usdt0.sh`：

```bash
#!/bin/bash
set -e

# ========== 配置 ==========
USDT0="0xf7C260136176100CE2A4Faf70045D3A0fB6fB86E"
POLYGON_RPC="https://polygon-rpc.com"
MY_ADDRESS="0xYourAddress"
PRIVATE_KEY="0xYourPrivateKey"
AMOUNT="3000000"  # 3 USDT0
DST_EID="110"     # Arbitrum

# ========== 转换地址 ==========
MY_ADDRESS_BYTES32=$(cast --to-bytes32 $MY_ADDRESS)
EXTRA_OPTIONS="0x00030100110100000000000000000000000000030d40"

# ========== 估算费用 ==========
echo "📊 估算跨链费用..."
RESULT=$(cast call $USDT0 \
  "quoteSend((uint32,bytes32,uint256,uint256,bytes,bytes,bytes),bool)((uint256,uint256))" \
  "($DST_EID,$MY_ADDRESS_BYTES32,$AMOUNT,$AMOUNT,$EXTRA_OPTIONS,0x,0x)" \
  false \
  --rpc-url $POLYGON_RPC)

NATIVE_FEE=$(echo $RESULT | sed 's/((//;s/))//' | awk -F',' '{print $1}')

echo "💰 Native Fee: $(cast --from-wei $NATIVE_FEE) POL"

# ========== 发送交易 ==========
echo "🚀 发送跨链交易..."
TX_HASH=$(cast send $USDT0 \
  "send((uint32,bytes32,uint256,uint256,bytes,bytes,bytes),(uint256,uint256),address)" \
  "($DST_EID,$MY_ADDRESS_BYTES32,$AMOUNT,$AMOUNT,$EXTRA_OPTIONS,0x,0x)" \
  "($NATIVE_FEE,0)" \
  "$MY_ADDRESS" \
  --value $NATIVE_FEE \
  --private-key $PRIVATE_KEY \
  --rpc-url $POLYGON_RPC \
  --legacy \
  --json | jq -r '.transactionHash')

echo "✅ 交易已发送: $TX_HASH"
echo "🔍 跟踪状态: https://layerzeroscan.com/tx/$TX_HASH"
```

运行：
```bash
chmod +x bridge-usdt0.sh
./bridge-usdt0.sh
```

---

## 6. 参数说明

### SendParam 各字段

| 字段 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `dstEid` | uint32 | 目标链 Endpoint ID | `110` (Arbitrum) |
| `to` | bytes32 | 接收地址（bytes32 格式） | `0x000...abc` |
| `amountLD` | uint256 | 发送金额（本地精度） | `3000000` (3 USDT0) |
| `minAmountLD` | uint256 | 最小接收金额 | `2985000` (0.5% 滑点) |
| `extraOptions` | bytes | Gas 配置 | `0x0003...` |
| `composeMsg` | bytes | 组合消息 | `0x` (空) |
| `oftCmd` | bytes | OFT 命令 | `0x` (空) |

### MessagingFee 结构

| 字段 | 类型 | 说明 |
|------|------|------|
| `nativeFee` | uint256 | 原生代币费用（POL） |
| `lzTokenFee` | uint256 | ZRO token 费用（通常为 0） |

---

## 7. extraOptions 详解

LayerZero 使用紧凑的二进制编码：

```
0x0003  ← Type 3: LZ Compose
0x0101  ← Option Type 1: Gas Limit
0x0011  ← Length = 17 bytes
0x01    ← 执行选项类型
0x00000000000000000000000000030d40  ← Gas = 200,000
```

**如何修改 Gas**：
```bash
# 200,000 gas
EXTRA_OPTIONS="0x00030100110100000000000000000000000000030d40"

# 500,000 gas (0x07a120)
EXTRA_OPTIONS="0x0003010011010000000000000000000000000007a120"

# 1,000,000 gas (0x0f4240)
EXTRA_OPTIONS="0x00030100110100000000000000000000000000f4240"
```

---

## 8. Cast 常用命令

### 地址工具
```bash
# 转换为 bytes32
cast --to-bytes32 0xYourAddress

# 转换为 checksum 格式
cast --to-checksum-address 0xabcdef...

# 生成随机地址
cast wallet new

# 获取最大 uint256
cast max-uint
```

### 单位转换
```bash
# Wei → Ether
cast --from-wei 500000000000000000
# 输出: 0.5

# Ether → Wei
cast --to-wei 0.5
# 输出: 500000000000000000

# Hex → Dec
cast --to-dec 0x030d40
# 输出: 200000

# Dec → Hex
cast --to-hex 200000
# 输出: 0x30d40
```

### 合约调用
```bash
# 查询（read）
cast call <CONTRACT> "functionName(args)(return)" <args> --rpc-url <RPC>

# 发送（write）
cast send <CONTRACT> "functionName(args)" <args> --private-key <KEY> --rpc-url <RPC>

# 解码调用数据
cast calldata "transfer(address,uint256)" 0xRecipient 1000000

# 解码事件日志
cast logs --from-block <BLOCK> --to-block <BLOCK> --address <CONTRACT> --rpc-url <RPC>
```

### 交易查询
```bash
# 查询交易详情
cast tx <TX_HASH> --rpc-url <RPC>

# 查询交易 receipt
cast receipt <TX_HASH> --rpc-url <RPC>

# 查询区块
cast block <BLOCK_NUMBER> --rpc-url <RPC>
```

---

## 9. LayerZero Endpoint IDs

| 链 | EID | RPC |
|----|-----|-----|
| Ethereum | 101 | https://eth.llamarpc.com |
| BSC | 102 | https://bsc-dataseed1.binance.org |
| Polygon | 109 | https://polygon-rpc.com |
| Arbitrum | 110 | https://arb1.arbitrum.io/rpc |
| Optimism | 111 | https://mainnet.optimism.io |
| Base | 184 | https://mainnet.base.org |
| zkSync | 165 | https://mainnet.era.zksync.io |

完整列表：https://docs.layerzero.network/v2/developers/evm/technical-reference/deployed-contracts

---

## 10. 费用参考

**Polygon → Arbitrum**：
- LayerZero 费用: ~0.3-0.8 POL
- Polygon Gas: ~0.01 POL
- **总计**: 约 0.31-0.81 POL (~$0.15-0.40 USD)

**Ethereum → Arbitrum**：
- LayerZero 费用: ~0.001-0.003 ETH
- Ethereum Gas: ~0.001-0.005 ETH (取决于 gas price)
- **总计**: 约 0.002-0.008 ETH (~$5-20 USD)

---

## 11. 常见问题

### Q1: 如何查看跨链进度？
A: 访问 [LayerZero Scan](https://layerzeroscan.com/)，输入交易哈希。通常 30-120 秒完成。

### Q2: 交易卡住了怎么办？
A: 可能是目标链 Gas 不足。访问 LayerZero Scan，使用 "Retry" 功能，或联系 LayerZero 支持。

### Q3: extraOptions 怎么计算？
A: 使用上面的模板，只需修改最后 8 个字节（Gas 数值的 hex）。

### Q4: minAmountLD 怎么设置？
A: 建议设为 `amountLD * 0.995`（0.5% 滑点保护）：
```bash
# 原始金额
AMOUNT=3000000

# 99.5% 接收
MIN_AMOUNT=$(echo "$AMOUNT * 995 / 1000" | bc)
echo $MIN_AMOUNT  # 2985000
```

### Q5: 支持哪些代币？
A: USDT0 是 Tether 的 LayerZero OFT 代币，部署在所有主流链。其他 OFT 代币类似操作。

### Q6: 如何查询 OFT 合约的 peers？
```bash
# 查询 Arbitrum 上的对端地址
cast call $USDT0 \
  "peers(uint32)(bytes32)" \
  110 \
  --rpc-url $POLYGON_RPC

# 输出应该匹配 USDT0 在 Arbitrum 的地址
```

---

## 12. 安全提示

⚠️ **重要**：
1. **永远不要分享私钥** - 使用环境变量 `$PRIVATE_KEY`，不要硬编码
2. **先测试小额** - 第一次跨链时用 1 USDT0 测试
3. **检查余额** - 确保有足够的原生代币（POL）支付费用
4. **验证地址** - 使用 `cast --to-checksum-address` 验证地址格式
5. **滑点保护** - 设置合理的 `minAmountLD`（建议 0.5-1%）
6. **Gas 预留** - LayerZero 费用估算可能波动，多预留 10-20%

---

## 13. 调试技巧

### 查看调用数据
```bash
# 生成 calldata（不发送）
cast calldata "send((uint32,bytes32,uint256,uint256,bytes,bytes,bytes),(uint256,uint256),address)" \
  "(110,$MY_ADDRESS_BYTES32,3000000,3000000,$EXTRA_OPTIONS,0x,0x)" \
  "($NATIVE_FEE,0)" \
  "$MY_ADDRESS"
```

### 模拟调用（dry-run）
```bash
# 使用 cast call 模拟 send（不花费 gas）
cast call $USDT0 \
  "send((uint32,bytes32,uint256,uint256,bytes,bytes,bytes),(uint256,uint256),address)" \
  "(110,$MY_ADDRESS_BYTES32,3000000,3000000,$EXTRA_OPTIONS,0x,0x)" \
  "($NATIVE_FEE,0)" \
  "$MY_ADDRESS" \
  --value $NATIVE_FEE \
  --from $MY_ADDRESS \
  --rpc-url $POLYGON_RPC
```

### 查看 Gas 估算
```bash
cast estimate $USDT0 \
  "send((uint32,bytes32,uint256,uint256,bytes,bytes,bytes),(uint256,uint256),address)" \
  "(110,$MY_ADDRESS_BYTES32,3000000,3000000,$EXTRA_OPTIONS,0x,0x)" \
  "($NATIVE_FEE,0)" \
  "$MY_ADDRESS" \
  --value $NATIVE_FEE \
  --from $MY_ADDRESS \
  --rpc-url $POLYGON_RPC
```

---

## 14. 相关链接

- **LayerZero V2 Docs**: https://docs.layerzero.network/v2
- **LayerZero Scan**: https://layerzeroscan.com/
- **USDT0 Etherscan**: https://etherscan.io/address/0xf7C260136176100CE2A4Faf70045D3A0fB6fB86E
- **Foundry Book**: https://book.getfoundry.sh/
- **Cast 文档**: https://book.getfoundry.sh/reference/cast/
- **OFT Standard**: https://docs.layerzero.network/v2/developers/evm/oft/quickstart

---

## 15. 快速参考卡

```bash
# 环境变量
export USDT0="0xf7C260136176100CE2A4Faf70045D3A0fB6fB86E"
export POLYGON_RPC="https://polygon-rpc.com"
export MY_ADDRESS="0x..."
export PRIVATE_KEY="0x..."

# 检查余额
cast call $USDT0 "balanceOf(address)(uint256)" $MY_ADDRESS --rpc-url $POLYGON_RPC

# 授权（一次性）
cast send $USDT0 "approve(address,uint256)" $USDT0 $(cast max-uint) --private-key $PRIVATE_KEY --rpc-url $POLYGON_RPC

# 转换地址
MY_BYTES32=$(cast --to-bytes32 $MY_ADDRESS)

# 估算费用
RESULT=$(cast call $USDT0 "quoteSend((uint32,bytes32,uint256,uint256,bytes,bytes,bytes),bool)((uint256,uint256))" "(110,$MY_BYTES32,3000000,3000000,0x00030100110100000000000000000000000000030d40,0x,0x)" false --rpc-url $POLYGON_RPC)
NATIVE_FEE=$(echo $RESULT | sed 's/((//;s/))//' | awk -F',' '{print $1}')

# 发送
cast send $USDT0 "send((uint32,bytes32,uint256,uint256,bytes,bytes,bytes),(uint256,uint256),address)" "(110,$MY_BYTES32,3000000,3000000,0x00030100110100000000000000000000000000030d40,0x,0x)" "($NATIVE_FEE,0)" "$MY_ADDRESS" --value $NATIVE_FEE --private-key $PRIVATE_KEY --rpc-url $POLYGON_RPC --legacy
```

---

**作者**: 汤圆 ⚪  
**更新**: 2026-02-12  
**工具**: Foundry Cast + curl  
**版本**: 2.0 - 修正函数签名
