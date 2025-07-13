# Time-Based Deterministic Token (TDT)

一种结合了时间戳、密钥和哈希处理，生成固定长度、不可逆且无时间窗口滞留的字节数组的算法。

## 用途

该算法被设计用于：

- 辅助验证资源服务器和客户端身份
- 抵抗重放攻击，保护用户数据

## 算法内容

### 输入

| 名称            | 用途                      |
| --------------- | ------------------------- |
| `tdt_secret`    | 密钥 (字节数组)           |
| `timestamp`     | **毫秒级** **UTC** 时间戳 |
| `result_length` | 结果长度 (默认为 256)     |

### 输出

长度为 `result_length` 的字节数组（`byte[]`）

### 流程

1. 将 `timestamp` 转换为 **8 字节大端序** 字节数组
2. 以 `tdt_secret` 为密钥, 对 `timestamp` 使用 **KMAC128** 算法计算生成最终结果，长度为 `result_length`

> [!IMPORTANT]
>
> - 输入时间戳应当为 8 字节无符号整数
> - 算法中，时间戳转换为字节数组时采用大端序 (Big-Endian)
> - 核心令牌生成使用固定域分离标签 "5beeb687e266"

### 伪代码示例

```
function GenerateTDT(tdt_secret: bytes, timestamp: int, resultLength: int = 256) -> bytes:
    # 时间戳转换 (大端序 8字节)
    timestamp_bytes = timestamp.to_bytes(8, 'big')

    # KMAC128 核心令牌生成 (抗量子MAC)
    result = kmac128(
        key = secret,
        data = timestamp_bytes,
        output_len = resultLength,  # 输出长度为 resultLength
        customization = "5beeb687e266"  # 固定域分离标签
    )

    return result
```

验证 TDT 值时，可以：

```
function ValidateTDT(tdt: byte[], secret: byte[], timestamp: int64) -> bool:
    // 获取 tdt 长度
    resultLength = getByteLength(tdt)

    // 用同样的参数代入生成 generatedTDT
    generatedTDT = GenerateTDT(secret, timestamp, resultLength)

    // 判断 tdt 与 generatedTDT 是否相等并返回结果
    return HMAC.compareDigest(tdt, generatedTDT);
```

## 其它要求

在 Veritas Access Protocol 协议中使用 TDT 算法时，**必须**严格遵守以下要求:

- `result_length` 参数不应小于 256。
- `tdt_secret` 长度不得小于为 32 字节
- 在实现算法文档的伪代码示例中 `ValidateTDT` 函数 (或任何功能等效的 TDT 验证函数) 时，**必须**使用具备恒定时间比较特性的函数来比较两个 TDT 值 (例如 HMAC 算法中常见的 `compareDigest` 函数) 。**禁止**使用普通的字节数组比较操作，以防止潜在的时序攻击漏洞。恒定时间比较确保验证操作所需的时间不依赖于 TDT 值本身的匹配程度。

## 时间同步

**发送方**和**认证服务器**之间的时间同步解决方案应当由**认证服务器**规定。同理，此设计旨在为认证服务器与客户端提供更高的灵活性，使其能够不断更新并提高安全性。

作为最低安全要求，**认证服务器**应当指定信任的 NTP 服务器实现各方之间的时间同步。对于采用最低安全要求的，我们强烈建议启用 RFC 8915 认证扩展。

## TDT 密钥轮换

本协议**不会强制规定**具体 `tdt_secret` 轮换方案。此设计旨在为认证服务器与客户端提供更高的灵活性，使其能够不断更新并采用更高安全级别的 `tdt_secret` 轮换方案。

## `tdt` 传输流程

假设**发送方**和**认证服务器**共同持有同一 TDT 密钥 `tdt_secret`。且在此之前**发送方**和**认证服务器**已经按照**认证服务器**规定的方式进行时间同步

**发送方**操作:

1. 使用 `tdt_secret` 和**发送方**当前的毫秒级 UTC 时间戳 `current_timestamp` 生成 TDT 值 `tdt_value`。
2. 将 `current_timestamp` 和 `tdt_value` 用空格隔开，生成 `tdt_message`
3. 将 `tdt_message` 依据[信息加密传输要求](./Information%20Transfer%20Recommendations.md)和**认证服务器**规定的方法签名、加密、封装，得到 JSON 字符串 `tdt`
4. 依据 VAP [核心操作流程](./Core.md#核心操作流程)，以规定的形式发送 `tdt`

**认证服务器**操作:

1. 依据 VAP [核心操作流程](./Core.md#核心操作流程)，以规定的形式接收 `tdt`
2. 使用[信息加密传输要求](./Information%20Transfer%20Recommendations.md)和**认证服务器**规定的方法对 `tdt` 解密、验签，得到 `tdt_message`
3. 将 `tdt_message` 以第一个空格作为分界线进行分割，按顺序得到时间戳 `current_timestamp` 和 `tdt_value`
4. 将 `current_timestamp` 与 **认证服务器**当前的毫秒级 UTC 时间戳进行比较，误差的绝对值必须小于 `timestamp_offset`
5. 获取存储的时间戳 `last_timestamp` (首次验证时或 `last_timestamp` 过期时跳过这一步)
6. 将 `current_timestamp` 与 `last_timestamp` 进行比较，`current_timestamp` 必须大于 `last_timestamp` (首次验证时或 `last_timestamp` 过期时跳过这一步)
7. 依据 `current_timestamp` 和 `tdt_secret` 验证 `tdt_value`
8. 将 `current_timestamp` 作为 `last_timestamp` 进行存储

> [!IMPORTANT]
>
> - `timestamp_offset` 的单位为毫秒，具体大小由**认证服务器**综合规定，但**禁止大于** 60000，即 1min
> - 对于 `last_timestamp` 的存储必须持久化，以对抗重放攻击。
> - `last_timestamp` 应当与(**资源服务器的**)唯一标识符或(**客户端的**)资源访问令牌关联存储。

### `last_timestamp` 过期的的判定标准

该判定标准应当由**认证服务器**规定，以下为**最低安全标准**:

已知 `current_timestamp`、`last_timestamp` 与 `timestamp_offset`，当:

```
last_timestamp + (2 * timestamp_offset) < current_timestamp
```

时，判定 `last_timestamp` 过期。
