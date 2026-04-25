# 凭据提供程序协议 {#credential-provider-protocol}
本文档描述了构建 Cargo 凭据提供程序的信息。有关设置或使用凭据提供程序的信息，请参阅 [Registry Authentication](registry-authentication.md)。

使用外部凭据提供程序时，Cargo 通过 stdin/stdout 消息与凭据提供程序通信，这些消息以单行 JSON 的形式传递。

Cargo 将始终使用 `--cargo-plugin` 参数执行凭据提供程序。这使得凭据提供程序可执行文件能够拥有超出 Cargo 所需范围的额外功能。额外的参数通过 JSON 中的 `args` 字段传递。

## JSON 消息 {#json-messages}
本文档中的 JSON 消息为便于阅读而添加了换行符。
实际消息不得包含换行符。

### 凭据 hello 消息 {#credential-hello}
* 发送方：凭据提供程序
* 目的：在进程启动时用于标识支持的协议
```javascript
{
    "v":[1]
}
```

Cargo 发送的请求将包含一个 `v` 字段，其值设置为此处列出的某个版本。
如果 Cargo 不支持凭据提供程序提供的任何版本，它将发出错误并关闭凭据进程。

### 注册中心信息 {#registry-information}
* 发送方：Cargo
并非独立的消息。包含在 Cargo 发送的所有消息中，作为 `registry` 字段。
```javascript
{
    // 注册中心的索引 URL
    "index-url":"https://github.com/rust-lang/crates.io-index",
    // 配置中的注册中心名称（可选）
    "name": "crates-io",
    // 尝试访问需要认证的注册中心时收到的 HTTP 头（可选）
    "headers": ["WWW-Authenticate: cargo"]
}
```

### 登录请求 {#login-request}
* 发送方：Cargo
* 目的：收集并存储凭据
```javascript
{
    // 协议版本
    "v":1,
    // 要执行的操作：login
    "kind":"login",
    // 注册中心信息（参见注册中心信息）
    "registry":{"index-url":"sparse+https://registry-url/index/", "name": "my-registry"},
    // 用户通过 stdin 或命令行指定的令牌（可选）
    "token": "<the token value>",
    // 用户可以访问以获取令牌的 URL（可选）
    "login-url": "http://registry-url/login",
    // 额外的命令行参数（可选）
    "args":[]
}
```

如果设置了 `token` 字段，则凭据提供程序应使用提供的令牌。如果未设置 `token`，则凭据提供程序应提示用户输入令牌。

除了可能在配置中传递给凭据提供程序的参数外，`cargo login` 还支持通过 `cargo login -- <additional args>` 传递额外的命令行参数。这些额外参数将在 Cargo 配置中的任何参数之后包含在 `args` 字段中。

### 读取请求 {#read-request}
* 发送方：Cargo
* 目的：获取用于读取 crate 信息的凭据
```javascript
{
    // 协议版本
    "v":1,
    // 请求类型：获取凭据
    "kind":"get",
    // 要执行的操作：读取 crate 信息
    "operation":"read",
    // 注册中心信息（参见注册中心信息）
    "registry":{"index-url":"sparse+https://registry-url/index/", "name": "my-registry"},
    // 额外的命令行参数（可选）
    "args":[]
}
```

### 发布请求 {#publish-request}
* 发送方：Cargo
* 目的：获取用于发布 crate 的凭据
```javascript
{
    // 协议版本
    "v":1,
    // 请求类型：获取凭据
    "kind":"get",
    // 要执行的操作：发布 crate
    "operation":"publish",
    // crate 名称
    "name":"sample",
    // crate 版本
    "vers":"0.1.0",
    // crate 校验和
    "cksum":"...",
    // 注册中心信息（参见注册中心信息）
    "registry":{"index-url":"sparse+https://registry-url/index/", "name": "my-registry"},
    // 额外的命令行参数（可选）
    "args":[]
}
```

### 获取成功响应 {#get-success-response}
* 发送方：凭据提供程序
* 目的：向 Cargo 提供凭据
```javascript
{"Ok":{
    // 响应类型：这是一个 get 请求
    "kind":"get",
    // 要发送到注册中心的令牌
    "token":"...",
    // 缓存控制。可以是以下之一：
    // * "never"：不缓存
    // * "session"：在当前 cargo 会话期间缓存
    // * "expires"：在当前 cargo 会话期间缓存，直到过期
    "cache":"expires",
    // Unix 时间戳（仅用于 "cache": "expires"）
    "expiration":1693942857,
    // 令牌操作是否独立
    "operation_independent":true
}}
```

`token` 将作为 `Authorization` HTTP 头的值发送到注册中心。

`operation_independent` 指示令牌是否可以跨不同操作（例如发布或获取）进行缓存。通常，这应为 `true`，除非提供程序希望生成限定于特定操作的令牌。

### 登录成功响应 {#login-success-response}
* 发送方：凭据提供程序
* 目的：指示登录成功
```javascript
{"Ok":{
    // 响应类型：这是一个 login 请求
    "kind":"login"
}}
```

### 登出成功响应 {#logout-success-response}
* 发送方：凭据提供程序
* 目的：指示登出成功
```javascript
{"Ok":{
    // 响应类型：这是一个 logout 请求
    "kind":"logout"
}}
```

### 失败响应（不支持的 URL）{#failure-response-url-not-supported}
* 发送方：凭据提供程序
* 目的：向 Cargo 提供错误信息
```javascript
{"Err":{
    "kind":"url-not-supported"
}}
```
如果凭据提供程序设计为仅处理特定的注册中心 URL，而给定的 URL 不受支持，则发送此响应。Cargo 将尝试其他可用的提供程序。

### 失败响应（未找到）{#failure-response-not-found}
* 发送方：凭据提供程序
* 目的：向 Cargo 提供错误信息
```javascript
{"Err":{
    // 错误：在提供程序中未找到凭据。
    "kind":"not-found"
}}
```
如果未找到凭据，则发送此响应。这在凭据不可用的 `get` 请求中，或未找到任何可擦除内容的 `logout` 请求中是预期的。

### 失败响应（不支持的操作）{#failure-response-operation-not-supported}
* 发送方：凭据提供程序
* 目的：向 Cargo 提供错误信息
```javascript
{"Err":{
    // 错误：在提供程序中未找到凭据。
    "kind":"operation-not-supported"
}}
```
如果凭据提供程序不支持请求的操作，则发送此响应。如果提供程序仅支持 `get` 而请求了 `login`，则提供程序应使用此错误进行响应。

### 失败响应（其他）{#failure-response-other}
* 发送方：凭据提供程序
* 目的：向 Cargo 提供错误信息
```javascript
{"Err":{
    // 错误：其他错误
    "kind":"other",
    // 要显示的错误消息字符串
    "message": "free form string error message",
    // 错误的详细原因链（可选）
    "caused-by": ["cause 1", "cause 2"]
}}
```

## 请求读取令牌的通信示例：{#example-communication-to-request-a-token-for-reading}
1. Cargo 启动凭据进程，捕获 stdin 和 stdout。
2. 凭据进程向 Cargo 发送 Hello 消息
    ```javascript
    { "v": [1] }
   ```
3. Cargo 向凭据进程发送 CredentialRequest 消息（为便于阅读添加了换行符）。
    ```javascript
    {
        "v": 1,
        "kind": "get",
        "operation": "read",
        "registry":{"index-url":"sparse+https://registry-url/index/"}
    }
    ```
4. 凭据进程向 Cargo 发送 CredentialResponse（为便于阅读添加了换行符）。
    ```javascript
    {
        "token": "...",
        "cache": "session",
        "operation_independent": true
    }
    ```
5. Cargo 关闭通往凭据提供程序的 stdin 管道，提供程序退出。
6. 在与该注册中心交互时，Cargo 在会话的剩余时间内（直到 Cargo 退出）使用该令牌。