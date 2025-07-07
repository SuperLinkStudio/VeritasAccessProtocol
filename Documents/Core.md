# VeritasAccessProtocol (VAP) 协议规范文档

VeritasAccessProtocol (VAP) 是一种**专门用于管理访问令牌使用阶段**的安全协议，旨在保障客户端与服务端之间受保护数据的交互安全。

## 术语表

### 凭证

| 名称                 | 解释                                                                                 |
| -------------------- | ------------------------------------------------------------------------------------ |
| `code`               | 资源访问令牌兑换码，有效期为 5 min。                                                 |
| `expire_time`        | 到期时间，为 `yyyy-MM-dd HH-mm-ss` 格式                                              |
| `access_token`       | 资源访问令牌，由认证服务器颁发，用于客户端向资源服务器证明其已获授权访问特定用户数据 |
| `account_id`         | 用户唯一标识符，由认证服务器分配                                                     |
| `client_id`          | 客户端唯一标识符，由认证服务器分配                                                   |
| `resource_server_id` | 资源服务器唯一标识符，由认证服务器分配                                               |

### 角色

| 角色       | 解释                                                                                            |
| ---------- | ----------------------------------------------------------------------------------------------- |
| 用户       | 使用客户端并作为其个人数据所有者的一方                                                          |
| 客户端     | 依据 VAP 协议向资源服务器请求访问用户数据的一方                                                 |
| 认证服务器 | 负责用户身份认证、向客户端颁发 `access_token`、并向资源服务器提供 `access_token` 验证服务的一方 |
| 资源服务器 | 存储并提供用户数据的一方或多方                                                                  |
| 服务端     | 认证服务器与资源服务器的统称                                                                    |

### 算法相关

| 名称 / 概念  | 解释                                                                                                                                  |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| TDT          | 一种用于生成固定长度、不可逆且无时间窗口滞留的字节数组的算法，详情可参考 [/Documents/TDT Algorithm.md](/Documents/TDT%20Algorithm.md) |
| `tdt`        | 一段由 TDT 算法生成的字节数组，详情可参考 [/Documents/TDT Algorithm.md](/Documents/TDT%20Algorithm.md)                                |
| TDT 值       | TDT 算法的结果，详情可参考 [/Documents/TDT Algorithm.md](/Documents/TDT%20Algorithm.md)                                               |
| `tdt_secret` | TDT 算法的共享密钥，由认证服务器生成并安全分发给通信双方，详情可参考 [/Documents/TDT Algorithm.md](/Documents/TDT%20Algorithm.md)     |
| 密钥组       | 部署于两个角色之间的两组非对称加密函数密钥对                                                                                          |
| RSA-3072     | 一种非对称加密算法                                                                                                                    |

### 其它

| 名称 | 解释                       |
| ---- | -------------------------- |
| 资源 | 和 "数据" 一词是一对同义词 |

> [!TIP]
> 有时资源服务器与认证服务器是同一个主体。

## 信息加密传输要求

VAP 中一切需要加密传输的信息均采用 `RSA-3072` 算法。服务（指客户端与资源服务器）在部署前，其开发者必须向认证服务器完成服务注册，以获取相应的密钥。

服务与认证服务器之间，需约定并部署两组密钥对，分别称为 `A` (包含私钥`private_A` 和公钥 `public_A`) 和 密钥对 `B` (包含私钥 `private_B` 和公钥 `public_B`) 表示。这两组密钥对构成一个**密钥组**。

假设:

- **发送方**持有密钥 `private_A` (`A` 的私钥) 和 `public_B` (`B` 的公钥)
- **接收方**持有密钥对 `private_B` (`B` 的私钥) 和 `public_A` (`A` 的公钥)

数据 `data` 的加密传输流程如下:

**发送方**操作:

1. 使用 `private_A` 对 `data` 进行签名，生成签名 `signature_data`
2. 使用 `public_B` 对 `data` 进行加密，生成密文 `data_ciphertext`
3. 依据 VAP 相关流程，以规定的形式发送 `signature_data` 和 `data_ciphertext`

**接收方**操作:

1. 依据 VAP 相关流程，以规定的形式接收 `signature_data` 和 `data_ciphertext`
2. 使用 `private_B` 对 `data_ciphertext` 进行解密，得到原始 `data`
3. 使用 `public_A` 和 `data` 对 `signature_data` 进行验签

### 关键配置原则

VAP 协议中仅配置两个密钥组:

1. 客户端与认证服务器之间部署一个密钥组。

2. 资源服务器与认证服务器之间部署一个密钥组。

协议**禁止在客户端与资源服务器之间**配置或共享密钥组。

### 密钥持有规则

- **客户端**与**认证服务器**之间

  客户端持有: `client_private` (客户端的私钥), `client_auth_public` (认证服务器的公钥)

  认证服务器持有: `client_auth_private` (认证服务器的私钥), `client_public` (客户端的公钥)

- **资源服务器**与**认证服务器**之间

  资源服务器持有: `resource_private` (资源服务器的私钥), `resource_auth_public` (认证服务器的公钥)

  认证服务器持有: `resource_auth_private` (认证服务器的私钥), `resource_public` (资源服务器的公钥)

可见，存在四组密钥对:

| 编号 | 私钥                    | 公钥                   |
| ---- | ----------------------- | ---------------------- |
| 1    | `client_private`        | `client_public`        |
| 2    | `client_auth_private`   | `client_auth_public`   |
| 3    | `resource_private`      | `resource_public`      |
| 4    | `resource_auth_private` | `resource_auth_public` |

## Time-Based Deterministic Token (TDT) 算法要求

TDT 算法的具体实现细节、输入参数定义及处理流程，详见参考文档: [/Documents/TDT Algorithm.md](/Documents/TDT%20Algorithm.md)。

在 VAP 协议中使用 TDT 算法时，**必须**严格遵守以下要求:

- `ResultLength` 参数不应小于 256。
- 在实现算法文档的伪代码示例中 `ValidateTDT` 函数（或任何功能等效的 TDT 验证函数）时，**必须**使用具备恒定时间比较特性的函数来比较两个 TDT 值（例如 HMAC 算法中常见的 `compareDigest` 函数）。**禁止**使用普通的字节数组比较操作，以防止潜在的时序攻击漏洞。恒定时间比较确保验证操作所需的时间不依赖于 TDT 值本身的匹配程度。

### `tdt` 校验流程

假设**发送方**和**认证服务器**共同持有同一 TDT 密钥 `tdt_secret`。

**发送方**操作:

1. 使用 `tdt_secret` 和**发送方**当前的毫秒级 UTC 时间戳 `current_timestamp` 生成 TDT 值 `tdt`。
2. 将 `current_timestamp` 和 `tdt` 用空格隔开，生成 `tdt_message`
3. 将 `tdt_message` 使用[信息加密传输要求](#信息加密传输要求)中规定的方法签名、加密，得到签名 `signature_tst` 和密文 `tdt_ciphertext`
4. 依据 VAP 相关流程，以规定的形式发送 `signature_tdt` 和 `tdt_ciphertext`

**认证服务器**操作:

1. 依据 VAP 相关流程，以规定的形式接收 `signature_tdt` 和 `tdt_ciphertext`
2. 使用[信息加密传输要求](#信息加密传输要求)中规定的方法对 `tdt_ciphertext` 解密，得到 `tdt_message`
3. 使用[信息加密传输要求](#信息加密传输要求)中规定的方法对 `signature_tdt` 验签
4. 将 `tdt_message` 以第一个空格作为分界线进行分割，按顺序得到时间戳 `current_timestamp` 和 `tdt`
5. 将 `current_timestamp` 与 **验证方**当前的毫秒级 UTC 时间戳进行比较，误差的绝对值必须小于 `timestamp_offset` (8 字节无符号整数，视情况自定，但**禁止大于** 300000，即 5min)
6. 获取存储的时间戳 `last_timestamp` (若未存储 `last_timestamp`，忽略这一步)
7. 将 `current_timestamp` 与 `last_timestamp` 进行比较，`current_timestamp` 必须大于 `last_timestamp` (若未存储 `last_timestamp`，忽略这一步)
8. 验证 `tdt`
9. 将 `current_timestamp` 作为 `last_timestamp` 进行存储

> [!IMPORTANT]
> 对于 `last_timestamp` 的存储必须持久化，服务重启后仍需保持，以对抗重放攻击。

### `last_timestamp` 存储要求

- 客户端与认证服务器之间:

  将 `last_timestamp` 与客户端的 `access_token` 关联存储。

- 资源服务器与认证服务器之间:

  将 `last_timestamp` 与资源服务器的`resource_server_id` 关联存储。

### 关键配置原则

VAP 协议中仅配置两个 `tdt_secret`:

1. 客户端与认证服务器之间部署一个 `tdt_secret`。

2. 资源服务器与认证服务器之间部署一个 `tdt_secret`。

协议**禁止在客户端与资源服务器之间**配置或共享 `tdt_secret`。

## scope 命名规范

### 基本结构

所有 scope 名称必须采用三级命名空间结构：

```
<service_name>:<scope>:<data_name>
```

当需要的数据范围包含某一 scope_group 下的所有 scope_name 时，可使用以下简写形式:

```
<service_name>:<scope>
```

### 命名规则

| 组成部分       | 规则           |
| -------------- | -------------- |
| `service_name` | 资源服务器名称 |
| `scope`        | 数据范围分组   |
| `data_name`    | 单个数据名称   |

例如：

```
authorize:account_data:name
```

`service_name`、`scope_group` 和 `scope_name` 应当遵循下划线命名法，禁止包含空格，且不应包含大写字母，不推荐使用拼音。

## 网络请求规范

VAP 协议所涉及的所有网络请求 (重定向除外) 均**必须**使用 `POST` 请求，并严格遵守以下规定:

- 请求头 Content-Type 必须设置为 `application/json`
- 所有请求参数和返回数据必须通过 Body 传输
- 禁止在 URL 查询参数、Header 或 Cookie 中传递敏感数据
- Body 数据必须为严格符合 `RFC 8259` 标准的 `JSON` 格式

## 核心操作流程

### 客户端上线前

该流程应当由客户端的开发者完成，客户端的开发者应当在**认证服务器的官方网站中**或**线下**向认证服务器完成注册。

VAP 协议不涉及注册时具体流程，但是客户端应当向认证服务器提供以下信息:

| 名称                      | 解释                                                 |
| ------------------------- | ---------------------------------------------------- |
| `after_auth_redirect_url` | 用户身份认证结束后，认证服务器将把用户重定向到的地址 |

当注册结束时，认证服务器至少应当生成如下信息:

| 名称                  | 应向客户端的开发者提供 | 解释                      |
| --------------------- | ---------------------- | ------------------------- |
| `client_id`           | 必须                   | 客户端唯一标识符          |
| `tdt_secret`          | 必须                   | 客户端的 TDT 算法密钥     |
| `client_private`      | 必须                   | 客户端私钥                |
| `client_public`       | 绝不                   | 客户端公钥                |
| `client_auth_private` | 绝不                   | 认证服务器私钥            |
| `client_auth_public`  | 必须                   | 认证服务器公钥            |
| `authorize_url`       | 应当                   | 用户身份认证界面地址      |
| `redeem_url`          | 应当                   | 资源访问令牌兑换 API 地址 |
| `update_url`          | 应当                   | 资源访问令牌更新 API 地址 |
| `destroy_url`         | 应当                   | 资源访问令牌销毁 API 地址 |

客户端必须对认证服务器所提供的信息进行保密，并做好充足的防泄漏措施。

### 资源服务器上线前

该流程应当由资源服务器的开发者完成，资源服务器的开发者应当在**认证服务器的官方网站中**或**线下**向认证服务器完成注册。

VAP 协议不涉及注册时具体流程，但是资源服务器应当向认证服务器提供并且对外公开以下信息:

| 名称           | 解释                                                                                                                                                                                                          |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `service_name` | 资源服务器名称                                                                                                                                                                                                |
| `resource_url` | 用户数据访问 API 地址                                                                                                                                                                                         |
| `scope_names`  | (字符串数组) 资源服务器提供的数据范围名称列表，对于其中每一个元素 `scope_name`，都是格式为 `<scope>:<data_name>` 的字符串，表示资源服务器提供的数据范围名称，具体细则请参考 [scope 命名规范](#scope-命名规范) |

> [!TIP]
> 注意: `service_name` 和 `scope_name` 应当遵循下划线命名法，禁止包含空格，且不应包含大写字母，不推荐使用拼音。

当注册结束时，认证服务器至少应当生成如下信息:

| 名称                    | 应向资源服务器的开发者提供 | 解释                            |
| ----------------------- | -------------------------- | ------------------------------- |
| `resource_server_id`    | 必须                       | 资源服务器唯一标识符            |
| `tdt_secret`            | 必须                       | 资源服务器的 TDT 算法密钥       |
| `resource_private`      | 必须                       | 资源服务器私钥                  |
| `resource_public`       | 绝不                       | 资源服务器公钥                  |
| `resource_auth_private` | 绝不                       | 认证服务器私钥                  |
| `resource_auth_public`  | 必须                       | 认证服务器公钥                  |
| `authentication_url`    | 应当                       | 客户端资源访问令牌鉴权 API 地址 |

资源服务器必须对认证服务器所提供的信息进行保密，并做好充足的防泄漏措施。

### 客户端获取资源访问令牌

该流程应当在客户端尝试获取用户信息前完成。

1. **客户端**将**用户**重定向到**认证服务器**的 `authorize_url`，并附上以下内容作为查询:

   | 内容        | 解释                                             |
   | ----------- | ------------------------------------------------ |
   | `client_id` | 客户端唯一标识符                                 |
   | `scope`     | 客户端将要被授权访问的数据范围，多个，由空格隔开 |
   | `state`     | 一串随机数                                       |

2. **用户**在**认证服务器**处完成用户身份认证，向**用户**展示**客户端**的 `scope` 并询问是否授权

   VAP 协议不涉及用户身份认证其余具体细节。

   建议认证服务器允许用户自行缩小 `scope` 范围。

3. **认证服务器**将**用户**重定向到**客户端**的 `after_auth_redirect_url`，并附上以下内容作为查询:

   | 内容             | 解释                                        |
   | ---------------- | ------------------------------------------- |
   | `code`           | (加密 / 密文) 资源访问令牌兑换码            |
   | `signature_code` | (加密 / 签名) 被**认证服务器**签名的 `code` |
   | `state`          | 原样返回                                    |

4. **客户端**向**认证服务器**的 `redeem_url` 发送 `POST` 请求，并附上以下内容作为 Body 部分:

   | 内容             | 解释                                    |
   | ---------------- | --------------------------------------- |
   | `code`           | (加密 / 密文) 资源访问令牌兑换码        |
   | `signature_code` | (加密 / 签名) 被**客户端**签名的 `code` |
   | `tdt_ciphertext` | (加密 / 密文) `tdt`                     |
   | `signature_tdt`  | (加密 / 签名) 被**客户端**签名的 `tdt`  |
   | `state`          | 一串随机数                              |

5. **认证服务器**对于来自**客户端**的请求返回如下内容:

   | 内容                     | 解释                                                |
   | ------------------------ | --------------------------------------------------- |
   | `access_token`           | (加密 / 密文) 资源访问令牌                          |
   | `signature_access_token` | (加密 / 签名) 被**认证服务器**签名的 `access_token` |
   | `expire_time`            | 到期时间，为 `yyyy-MM-dd HH-mm-ss` 格式             |
   | `state`                  | 原样返回                                            |

### 客户端获取用户数据

该流程应当在客户端获取资源访问令牌之后、资源访问令牌被销毁之前完成。

1. **客户端**向**资源服务器**的 `resource_url` 发送 `POST` 请求，并附上以下内容作为 Body 部分:

   | 内容                     | 解释                                            |
   | ------------------------ | ----------------------------------------------- |
   | `client_id`              | 客户端唯一标识符                                |
   | `access_token`           | (加密 / 密文) 资源访问令牌                      |
   | `signature_access_token` | (加密 / 签名) 被**客户端**签名的 `access_token` |
   | `scope`                  | 客户端将要尝试访问的数据范围，多个，由空格隔开  |
   | `tdt_ciphertext`         | (加密 / 密文) `tdt`                             |
   | `signature_tdt`          | (加密 / 签名) 被**客户端**签名的 `tdt`          |
   | `state`                  | 一串随机数                                      |

2. **资源服务器**向**认证服务器**的 `authentication_url` 发送 `POST` 请求，并附上以下内容作为 Body 部分:

   | 内容                            | 解释                                       |
   | ------------------------------- | ------------------------------------------ |
   | `resource_server_id`            | 资源服务器唯一标识符                       |
   | `scope`                         | 客户端将要尝试访问的数据范围，原样发送     |
   | `client_id`                     | 客户端 `client_id`，原样发送               |
   | `client_access_token`           | 客户端 `access_token`，原样发送            |
   | `client_signature_access_token` | 客户端 `signature_access_token`，原样发送  |
   | `client_tdt_ciphertext`         | 客户端 `tdt_ciphertext`，原样发送          |
   | `client_signature_tdt`          | 客户端 `signature_tdt`，原样发送           |
   | `tdt_ciphertext`                | (加密 / 密文) `tdt`                        |
   | `signature_tdt`                 | (加密 / 签名) 被**资源服务器**签名的 `tdt` |
   | `state`                         | 一串随机数                                 |

3. **认证服务器** 验证并确定以下内容有效:

   - `scope`
   - `client_id`
   - `client_access_token`
   - `client_signature_access_token`
   - `client_tdt_ciphertext`
   - `client_signature_tdt`
   - `tdt_ciphertext`
   - `signature_tdt`

4. **认证服务器**对于来自**资源服务器**的请求返回如下内容:

   | 内容                   | 解释                                                         |
   | ---------------------- | ------------------------------------------------------------ |
   | `scope`                | 认证服务器允许客户端访问的数据范围                           |
   | `account_id`           | (加密 / 密文) 客户端 `access_token` 所绑定的用户的唯一标识符 |
   | `signature_account_id` | (加密 / 签名) 被**认证服务器**签名的 `account_id`            |
   | `state`                | 资源服务器 `state`，原样发送                                 |

5. **资源服务器**对于来自**客户端**的请求返回如下内容:

   | 内容                  | 解释                                             |
   | --------------------- | ------------------------------------------------ |
   | `scope`               | 认证服务器允许客户端访问的数据范围，原样返回     |
   | `user_data`           | (加密 / 密文) 用户数据                           |
   | `signature_user_data` | (加密 / 签名) 被**资源服务器**签名的 `user_data` |
   | `state`               | 客户端 `state`，原样发送                         |

   `user_data` 原文应当是一个符合 `RFC 8259` 标准的 `JSON` 文本，以每个 `scope` 为 Key， 以每个 `scope` 所对应的用户数据为 Value。对于非户数据的 `scope`，其对应 Value 应当为 null，即空值。

   `user_data` 原文示例:

   ```JSON
   {
    "authorize:account_data:name": "A White Cat",
    "authorize:account_data:bio": "I'am a white cat",
    "service:file:delete": null
   }
   ```
