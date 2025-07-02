# VeritasAccessProtocol (VAP) 规范文档

## 协议概述

VeritasAccessProtocol (VAP) 是一种专注于**访问令牌使用阶段**的安全协议，用于保护客户端、服务端之间的受保护数据交互。VAP 不涉及用户认证流程，仅处理客户端已持有访问令牌后的安全通信。

## 术语表

### 身份/主体

| 身份/主体  | 解释                                                                                                           |
| ---------- | -------------------------------------------------------------------------------------------------------------- |
| 用户       | 即使用客户端服务、向认证服务器提供相关个人信息以供身份认证的一方                                               |
| 客户端     | 即通过 VAP 协议向资源服务器获取用户数据的一方                                                                  |
| 认证服务器 | 即向用户提供身份认证服务、向客户端提供 `access_token` 发放服务、向资源服务器提供 `access_token` 鉴权服务的一方 |
| 资源服务器 | 即向客户端提供用户数据的一方 (有时资源服务器与认证服务器是同一个主体)                                          |
| 服务端     | 认证服务器与资源服务器的统称                                                                                   |

### 凭据

| 名称           | 解释                  |
| -------------- | --------------------- |
| `client_id`    | 客户端唯一标识        |
| `code`         | `access_token` 兑换码 |
| `access_token` | 资源访问令牌          |
| `request_id`   | 单次请求的唯一标识    |
| `expire_date`  | 到期时间              |

### 加密

| 名称            | 解释                                                           |
| --------------- | -------------------------------------------------------------- |
| `tdt_secret`    | TDT 密钥，用于生成 TDT 密码                                    |
| `tdt`           | TDT，用于辅助客户端身份确认                                    |
| `access_secret` | 资源密钥，用于资源传输加密                                     |
| `client_secret` | 客户端密钥，用于令牌传输加密                                   |
| `server_secret` | 服务端密钥，用于传输加密，不同的服务端所持有的 `server_secret` |

### 网络请求

| 名称        | 解释                                                                                 |
| ----------- | ------------------------------------------------------------------------------------ |
| `scope`     | 请求的内容范围                                                                       |
| `payload`   | 载荷，包含请求的用户数据，客户端需通过 `RSA` 算法将 `access_secret` 作为私钥进行解密 |
| `data`      | 用户数据                                                                             |
| `data_hash` | 用户数据的 `SHA256` 哈希值                                                           |

### URL 地址

| 名称                   | 解释                                                        |
| ---------------------- | ----------------------------------------------------------- |
| `authorize_url`        | (认证服务器端) 认证平台地址                                 |
| `redirect_url`         | (客户端) 用于告知认证服务器用户完成身份认证后重定向到的地址 |
| `token_conversion_url` | (认证服务器端) `access_token` 兑换地址                      |

### 操作

| 名称                     | 解释                                                                           |
| ------------------------ | ------------------------------------------------------------------------------ |
| 存储 `A`                 | 在数据库、内存或别的数据存储方式中添加记录，写入 A                             |
| 将 `A` 与 `B` 建立关联   | 在数据库、内存或别的数据存储方式中添加记录，写入 A 与 B 的对应关系             |
| 验证 `A` 与 `B` 的关联性 | 在数据库、内存或别的数据存储方式中查询相关记录，检查 A 与 B 的对应关系是否存在 |

## 相关规范

### 网络请求规范

VAP 的所有网络请求均要求采用 `POST` 请求，内容类型应当为 `application/json`。

请求内容应当严格按照标准 `JSON` 规范。

### `request_id` 规范

`request_id` 由客户端随机生成，也可以由认证服务器约定生成规则。

我们要求：`request_id` 至少是一个长度 >= 8 的字符串，且**不能**和[术语表](#术语表)中[加密](#加密)一栏里提到的名词与其所对应的值有任何关系。

### `access_token` 规范

所谓`access_token` 可以是一串对于每一个 `cliend_id` 来说唯一的哈希值字符串，也可以是 `Json Web Token`。

### `expire_time` 规范

`expire_time` 为字符串，该字符串严格遵循 `yyyy-MM-dd HH:mm:ss` 的格式。

### `client_secret`、`access_secret` 与 `server_secret` 规范

`client_secret`、`access_secret` 与 `server_secret` 是参与 `RSA` 加密的密钥，其中：

- `client_secret`: 公钥，交予客户端保存
- `access_secret`: 公钥，交予客户端保存，不同的资源服务器有不同的 `access_secret`
- `server_secret`: 私钥，由认证服务器留存

`client_secret`、`access_secret` 与 `server_secret` 均为 `XML` 格式的密钥。

### TDT 规范

TDT 算法请移步至 [TDT Algorithm.md](/Documents/TDT%20Algorithm.md) 文件。

## VAP 流程

### 获取令牌

这段时间主要是为了通过用户认证获取 `access_token`。

1. 将用户重定向到 `authorize_url`

   在此之前，客户端应当进行如下操作：

   1. 根据与认证服务器约定好的规则生成 `scope`
   2. 根据[规范](#request_id-规范)生成 `request_id`

   重定向时，应添加如下查询:

   - `client_id`
   - `request_id`
   - `scope`

   例如：

   ```
   GET https://authorizeservice.com/account/login?client_id=testclient_f888e7af9768b8e2fff4bda10d0b1744b00d3040163b7c9c2bfa1b095373d290&request_id=1h4om5o9&scope=account.userdata
   ```

2. 用户自行完成身份认证

   _VAP 协议不涉及该方面_

3. 认证服务器将用户重定向到 `redirect_url`

   在此之前，认证服务器应当进行如下操作:

   1. 验证 `scope` 是否越界 (即非法获取管理员才能获得的相关授权或超出用户指定的权限范围)
   2. 将 `scope` 与 `code` 建立关联
   3. 生成 `code`
   4. 确定 `code` 的有效期 (一般不会超过 10min)，计算 `expire_date`
   5. 将 `expire_date` 与 `code` 建立关联
   6. 将 `code` 与 `client_id` 建立关联
   7. 使用 `server_secret` 加密 `code` 并对其签名

   重定向时，应添加如下查询:

   - `code`: 加密、签名后的 `code`
   - `expire_date`
   - `request_id`: 原样返回

   例如：

   ```
   GET https://service.com/api/receive_code?code=Ou5hDWOKps7xd9ULY8HvoNOxibJU2W1bRQUpxId3yL9xblSyvTxN538WqSe8zcQEb3HHUWwyb2TB09UjpzzSBLpwSw0NAW5vuCN1U0usdBszYjhwS%2BZy6DQQuN5cTbZaoDH7kYIyLRQUUtQcR8kH7EB%2FsorsGQL2rNg6JVCll7I%3D&expire_date=1145-01-04%2019%3A19%3A08&request_id=1h4om5o9

   ```

   客户端接收到之后，应当进行如下操作:

   1. 验证 `client_id` 是否与 步骤一 中的 `client_id` 一致
   2. 对收到的 `code` 进行解密、验签

4. 客户端凭借 `code` 向 `token_conversion_url` 发送 `POST` 请求

   请求时，Body 中应含有如下内容：

   - `client_id`
   - `code`
   - `tdt`
   - `request_id`: 根据[规范](#request_id-规范)重新生成的 `request_id`

   例如：

   ```json
   {
     "client_id": "testclient_f888e7af9768b8e2fff4bda10d0b1744b00d3040163b7c9c2bfa1b095373d290",
     "code": "I1AM2A3WHITE4CAT",
     "tdt": "ZjU3NjRiYTIyZjYyODkyMWU0NmVmMGZiMDQyZWY1Njk0YzU1YjA5NjBiZmRlOWUxNTkzYTliNGUwMjUzMTAzNw==",
     "request_id": "1SI1SIHT"
   }
   ```

5. 认证服务器返回 `access_token`

   在此之前，认证服务器应当进行如下操作:

   1. 验证该 `code` 与 `client_id` 的关联性
   2. 验证 `tdt` 的有效性
   3. 验证 `code` 是否过时
   4. 生成 `access_token`
   5. 确定 `access_token` 的有效期，计算 `expire_date`
   6. 将 `expire_date` 与 `access_token` 建立关联
   7. 将 `access_token` 与 `client_id` 建立关联
   8. 使用 `server_secret` 加密 `access_token` 并对其签名

   返回的内容中，应当存在以下内容:

   - `access_token`: 加密、签名后的 `access_token`
   - `expire_date`
   - `request_id`: 原样返回

   例如：

   ```json
   {
     "access_token": "hWCtgXC3Jfa2K0BnrMaOdfGtJAMnJrbkZBnB6MymJxT6142tOXVL7ErHupbv3QPHo56PRrtZOHZopt5btqn+EJUn6YDqp+IPQXKGvUD+r+dutwOcng9JLcfGxJSEbd8KYDy65JqSuh7apksjIiVNRuNwFAYBQc8bA0KnwC3evYY=",
     "expire_date": "1919-08-10 11:45:14",
     "request_id": "1SI1SIHT"
   }
   ```
