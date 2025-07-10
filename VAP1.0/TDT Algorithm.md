# Time-Based Deterministic Token (TDT)

一种结合了时间戳、密钥和哈希处理，生成固定长度、不可逆且无时间窗口滞留的字节数组的算法。

> [!TIP]
> 你这辈子大概再也见不到这么暴(ruo)力(zhi)的算法了

## 输入

| 名称           | 用途                      |
| -------------- | ------------------------- |
| `Secret`       | 密钥 (字节数组)           |
| `Salt`         | 盐值 (字节数组)           |
| `Timestamp`    | **毫秒级** **UTC** 时间戳 |
| `ResultLength` | 结果长度 (默认为 256)     |

## 输出

一个长度为 `ResultLength` 的字节数组（`byte[]`）。

## 算法流程

1. 使用 `HKDF-SHA256` 加密算法以 `HashedSecret` 为密钥、以 `Salt` 为盐值对 `Secret` 进行加密，得到 `HashedKey`
2. 使用 `HMAC_BLAKE3` 哈希加密算法以 `HashedKey` 为密钥对 `Timestamp` 进行加密，最终得到 `Blake3Result`
3. 对 `Blake3Result` 进行 `SHAKE128` 算法加密，得到最终结果

> [!IMPORTANT]
> 输入时间戳应当为 8 字节无符号整数
> 算法中，时间戳转换为字节数组时采用大端序 (Big-Endian)
> Blake3Result 长度应当为标准的 32 字节

## 伪代码示例

```
function GenerateTDT(secret: byte[], salt: byte[], timestamp: int64, resultLength: int = 256) -> byte[]:
   function GenerateTDT(secret: byte[], timestamp: int64, context: string, resultLength: int = 32) -> byte[]:
    // 密钥派生
    hashed_hey = HKDF_SHA256(secret: secret, salt: salt)

    // 时间戳转换 (大端序 8 字节)
    timestampBytes = toBigEndianBytes(timestamp, 8)

    // HMAC-BLAKE3 加密
    hmac_result = HMAC_BLAKE3(key: hashed_key, data: timestampBytes)

    // SHAKE128 扩展输出
    result = SHAKE128(input: hmac_result, outputLength: resultLength)

    return result
```

验证 TDT 值时，可以：

```
function ValidateTDT(tdt: byte[], secret: byte[], timestamp: int64) -> bool:
    // 获取 tdt 长度
    resultLength = getByteLength(tdt)

    // 用同样的参数代入生成 generatedTDT
    generatedTDT = GenerateTDT(secret, timestamp, resultLength)

    // 判断 tdt 与 generatedTDT 是否相等并返回结果 (推荐采用 HMAC 算法的 compareDigest)
    return Compare(tdt, generatedTDT);
```
