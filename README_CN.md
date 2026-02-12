# LayerZero OFT 跨链转账 - 快速指南

## 📋 准备工作

### 需要准备的变量

1. **合约地址**
   - 源链 OFT 合约地址（如 Polygon 上的 USDT0）
   - 目标链 OFT 合约地址（如 Arbitrum 上的 USDT0）

2. **链信息**
   - 源链 RPC URL
   - 目标链 RPC URL
   - 源链 LayerZero Endpoint ID（如 Polygon = 109）
   - 目标链 LayerZero Endpoint ID（如 Arbitrum = 110）

3. **钱包信息**
   - 你的钱包地址
   - 你的钱包私钥（⚠️ 保密）

4. **转账参数**
   - 转账金额（如 3 USDT0 = 3000000）
   - 目标地址（可以是自己）
   - Gas 配置（extraOptions，通常用默认值）

---

## 🔄 操作流程

### 1. 检查余额
**函数**: `balanceOf(address)(uint256)`
- 输入: 你的钱包地址
- 输出: 代币余额（整数）

### 2. 授权 OFT 合约
**函数**: `approve(address,uint256)`
- 输入: 
  - OFT 合约地址
  - 授权额度（建议用最大值）
- 作用: 允许 OFT 合约操作你的代币
- ⚠️ 只需执行一次

### 3. 地址格式转换
**工具**: `cast --to-bytes32`
- 输入: 目标地址（0x 开头）
- 输出: bytes32 格式（32 字节，左填充 0）

### 4. 查询对等合约（可选）
**函数**: `peers(uint32)(bytes32)`
- 输入: 目标链 Endpoint ID
- 输出: 目标链上的对等合约地址（bytes32 格式）
- 作用: 验证目标链合约是否正确

### 5. 估算跨链费用
**函数**: `quoteSend(SendParam,bool)(MessagingFee)`
- 输入: 
  - SendParam 结构体（包含所有转账参数）
  - 是否使用 ZRO token（通常为 false）
- 输出: MessagingFee 结构体
  - nativeFee: 原生代币费用（如 POL）
  - lzTokenFee: ZRO token 费用（通常为 0）
- 作用: 查询需要支付的原生代币费用

### 6. 发送跨链交易
**函数**: `send(SendParam,MessagingFee,address)`
- 输入:
  - SendParam 结构体（转账参数）
  - MessagingFee 结构体（费用信息）
  - refund 地址（退款地址，通常是你自己）
- 附加: 需要支付 nativeFee（作为交易的 value）
- 输出: 
  - MessagingReceipt: 消息回执
  - OFTReceipt: OFT 回执

### 7. 跟踪跨链状态
**方式**: 访问 LayerZero Scan
- URL: https://layerzeroscan.com/
- 输入: 交易哈希
- 查看: 消息状态、执行进度

---

## 📦 数据结构详解

### SendParam（转账参数）
```
dstEid         → 目标链 Endpoint ID（如 110 表示 Arbitrum）
to             → 目标地址（bytes32 格式）
amountLD       → 转账金额（本地精度，如 3000000）
minAmountLD    → 最小接收金额（用于滑点保护）
extraOptions   → Gas 配置（hex 字符串，如 0x0003...）
composeMsg     → 组合消息（通常为空 0x）
oftCmd         → OFT 命令（通常为空 0x）
```

### MessagingFee（费用信息）
```
nativeFee      → 原生代币费用（如 0.5 POL）
lzTokenFee     → ZRO token 费用（通常为 0）
```

### MessagingReceipt（消息回执）
```
guid           → 消息全局唯一标识
nonce          → 消息序号
fee            → 实际支付的费用
```

### OFTReceipt（OFT 回执）
```
amountSentLD       → 发送的金额（本地精度）
amountReceivedLD   → 接收的金额（本地精度）
```

---

## ⚙️ extraOptions 配置

用于设置目标链执行时的 Gas：

| Gas 值 | Hex 编码 |
|--------|----------|
| 200,000 (简单转账) | `0x00030100110100000000000000000000000000030d40` |
| 500,000 (复杂交互) | `0x0003010011010000000000000000000000000007a120` |
| 1,000,000 (高 Gas) | `0x00030100110100000000000000000000000000f4240` |

---

## 🔢 函数签名（Solidity）

### 查询函数

```solidity
// 查询余额
function balanceOf(address account) external view returns (uint256);

// 查询对等合约
function peers(uint32 eid) external view returns (bytes32 peer);

// 查询费用
function quoteSend(
    SendParam calldata _sendParam,
    bool _payInLzToken
) external view returns (MessagingFee memory fee);
```

### 写入函数

```solidity
// 授权
function approve(address spender, uint256 amount) external returns (bool);

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

## 🌐 常用 LayerZero Endpoint IDs

| 链名 | EID |
|------|-----|
| Ethereum | 101 |
| BSC | 102 |
| Polygon | 109 |
| Arbitrum | 110 |
| Optimism | 111 |
| Base | 184 |
| zkSync | 165 |

完整列表: https://docs.layerzero.network/v2/developers/evm/technical-reference/deployed-contracts

---

## 💰 费用参考

**Polygon → Arbitrum**:
- LayerZero 费用: ~0.3-0.8 POL
- Gas 费: ~0.01 POL
- 总计: ~0.31-0.81 POL (~$0.15-0.40)

**Ethereum → Arbitrum**:
- LayerZero 费用: ~0.001-0.003 ETH
- Gas 费: ~0.001-0.005 ETH
- 总计: ~0.002-0.008 ETH (~$5-20)

---

## ⚠️ 安全提示

1. **私钥安全**: 永远不要分享私钥，使用环境变量存储
2. **小额测试**: 第一次跨链建议用小额测试（如 1 USDT0）
3. **余额检查**: 确保有足够的原生代币支付费用
4. **地址验证**: 仔细检查目标地址是否正确
5. **滑点保护**: 设置合理的 minAmountLD（建议 0.5-1%）

---

## 🔍 Cast 命令速查

### 查询操作
```bash
# 检查余额
cast call $OFT "balanceOf(address)(uint256)" $ADDRESS --rpc-url $RPC

# 查询对等合约
cast call $OFT "peers(uint32)(bytes32)" 110 --rpc-url $RPC

# 估算费用（quoteSend）
cast call $OFT \
  "quoteSend((uint32,bytes32,uint256,uint256,bytes,bytes,bytes),bool)((uint256,uint256))" \
  "(110,$TO_BYTES32,1000000,1000000,$OPTIONS,0x,0x)" \
  false \
  --rpc-url $RPC
```

### 写入操作
```bash
# 授权
cast send $OFT "approve(address,uint256)" $OFT $(cast max-uint) \
  --private-key $PK --rpc-url $RPC

# 发送跨链交易
cast send $OFT \
  "send((uint32,bytes32,uint256,uint256,bytes,bytes,bytes),(uint256,uint256),address)" \
  "(110,$TO_BYTES32,1000000,1000000,$OPTIONS,0x,0x)" \
  "($NATIVE_FEE,0)" \
  "$MY_ADDRESS" \
  --value $NATIVE_FEE \
  --private-key $PK \
  --rpc-url $RPC
```

### 工具命令
```bash
# 地址转 bytes32
cast --to-bytes32 0xYourAddress

# Wei 转 Ether
cast --from-wei 500000000000000000

# 获取最大 uint256
cast max-uint
```

---

## 📊 Cast 返回值解析

### quoteSend 返回值
```
输入: quoteSend((uint32,bytes32,uint256,uint256,bytes,bytes,bytes),bool)((uint256,uint256))
输出: ((500000000000000000,0))
       ↑                  ↑
       nativeFee          lzTokenFee
```

**提取 nativeFee**:
```bash
RESULT=$(cast call ...)
NATIVE_FEE=$(echo $RESULT | sed 's/((//;s/))//' | awk -F',' '{print $1}')
```

### send 返回值
```
交易成功后返回:
- transactionHash: 交易哈希
- blockNumber: 区块号
- status: 1 (成功) / 0 (失败)
```

---

## 🔗 相关链接

- **完整 CLI 教程**: [README.md](./README.md)
- **LayerZero Docs**: https://docs.layerzero.network/v2
- **LayerZero Scan**: https://layerzeroscan.com/
- **Foundry Book**: https://book.getfoundry.sh/

---

**作者**: 汤圆 ⚪  
**更新**: 2026-02-12  
**版本**: 2.0 - 修正函数签名
