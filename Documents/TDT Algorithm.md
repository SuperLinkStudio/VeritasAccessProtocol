# Time-Based Deterministic Token (TDT)

一种结合了时间戳、密钥和双重哈希处理，生成固定长度、不可逆且无时间窗口滞留的字符串的算法。

> [!TIP]
> 你这辈子大概再也见不到这么暴(ruo)力(zhi)的算法了

## 输入参数

| 名称           | 用途                  |
| -------------- | --------------------- |
| `Key`          | 密钥 (Byte 类型)      |
| `Timestamp`    | 当前时间戳            |
| `EncryptTime`  | 加密次数 (默认为 2)   |
| `ResultLength` | 结果长度 (默认为 256) |

## 算法流程

1. 使用 `SHA256` 哈希加密算法对 `Key` 进行哈希加密，得到 `HashedKey`
2. 使用 `HMAC-SHA256` 加密算法以 `HashedKey` 为密钥对 `Timestamp` 进行加密，得到 `RawCiphertext`
3. 使用 `SHA256` 哈希加密算法对 `RawCiphertext` 进行多次加密，加密次数为 `EncryptTime`，得到 `Sha256Result`
4. 对 `Sha256Result` 进行 `SHAKE128` 算法加密，得到最终结果

> [!IMPORTANT]
> 该算法中，所有的 `Byte[]` 类型数据均采用大端序.

## 伪代码

```
byte[] GenerateTDT(byte[] key, long timestamp, int encryptTime = 2, int resultLength = 256)
{
    byte[] hashedKey = Sha256(key);

    byte[] timestampByte = timestamp.ToByte();
    byte[] rawCiphertext = Hmac_Sha256(key: hashedKey, plaintext：timestampByte);

    byte[] sha256Result = rawCiphertext;
    for (int i = 0; i < encryptTime; i ++)
    {
        sha256Result = Sha256(sha256Result);
    }

    byte[] result = Shake128(plaintext: sha256Result, resultLength: resultLength)

    return result;
}

bool ValidateTDT(byte[] tdt, byte[] key, long timestamp, int encryptTime = 2, int resultLength = 256)
{
    return GenerateTDT(byte[] key, long timestamp, int encryptTime = 2, int resultLength = 256) == tdt;
}
```
