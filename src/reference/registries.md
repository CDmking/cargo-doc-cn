# 注册中心 {#registries}

Cargo 从"注册中心 (Registries)"安装 crate 并获取依赖项。默认注册中心是 [crates.io]。注册中心包含一个"索引"，其中包含可搜索的可用 crate 列表。注册中心还可以提供 Web API，以支持直接从 Cargo 发布新的 crate。

> 注意：如果您对镜像或打包现有注册中心感兴趣，请参阅 [Source Replacement]。

如果您正在实现注册中心服务器，请参阅 [Running a Registry] 了解有关 Cargo 与注册中心之间协议的更多详细信息。

如果您使用的注册中心需要身份验证，请参阅 [Registry Authentication]。
如果您正在实现凭据提供程序，请参阅 [Credential Provider Protocol] 了解详细信息。

## 使用备用注册中心 {#using-an-alternate-registry}

要使用 [crates.io] 以外的注册中心，必须将注册中心的名称和索引 URL 添加到 [`.cargo/config.toml` 文件][config]中。`registries` 表为每个注册中心设置一个键，例如：

```toml
[registries]
my-registry = { index = "https://my-intranet:8080/git/index" }
```

`index` 键应指向包含注册中心索引的 git 仓库的 URL，或带有 `sparse+` 前缀的 Cargo 稀疏注册中心 URL。

然后，crate 可以通过在 `Cargo.toml` 中该依赖项的条目中指定 `registry` 键和注册中心名称的值，来依赖来自另一个注册中心的 crate：

```toml
# Sample Cargo.toml
[package]
name = "my-project"
version = "0.1.0"
edition = "2024"

[dependencies]
other-crate = { version = "1.0", registry = "my-registry" }
```

与大多数配置值一样，索引可以通过环境变量而非配置文件来指定。例如，设置以下环境变量将实现与定义配置文件相同的效果：

```ignore
CARGO_REGISTRIES_MY_REGISTRY_INDEX=https://my-intranet:8080/git/index
```

> 注意：[crates.io] 不接受依赖来自其他注册中心的 crate 的包。

## 发布到备用注册中心 {#publishing-to-an-alternate-registry}

如果注册中心支持 Web API 访问，则可以直接从 Cargo 将包发布到注册中心。Cargo 的几个命令（如 [`cargo publish`]）接受 `--registry` 命令行标志来指示使用哪个注册中心。例如，要发布当前目录中的包：

1. `cargo login --registry=my-registry`

    这只需要执行一次。您必须输入从注册中心网站获取的密钥 API 令牌。或者，可以通过 `--token` 命令行标志或名为 `CARGO_REGISTRIES_MY_REGISTRY_TOKEN` 的环境变量将令牌直接传递给 `publish` 命令。

2. `cargo publish --registry=my-registry`

除了始终传递 `--registry` 命令行选项外，还可以在 [`.cargo/config.toml`][config] 中使用 `registry.default` 键设置默认注册中心。例如：

```toml
[registry]
default = "my-registry"
```

在 `Cargo.toml` 清单中设置 `package.publish` 键可以限制包允许发布到哪些注册中心。这对于防止意外将闭源包发布到 [crates.io] 很有用。该值可以是注册中心名称的列表，例如：

```toml
[package]
# ...
publish = ["my-registry"]
```

`publish` 值也可以是 `false` 以限制所有发布，这与空列表相同。

[`cargo login`] 保存的身份验证信息存储在 Cargo 主目录（默认为 `$HOME/.cargo`）中的 `credentials.toml` 文件中。它为每个注册中心设置一个单独的表，例如：

```toml
[registries.my-registry]
token = "854DvwSlUwEHtIo3kWy6x7UCPKHfzCmy"
```

## 注册中心协议 {#registry-protocols}
Cargo 支持两种远程注册中心协议：`git` 和 `sparse`。如果注册中心索引 URL 以 `sparse+` 开头，Cargo 使用稀疏协议。否则 Cargo 使用 `git` 协议。

`git` 协议将索引元数据存储在 git 仓库中，并要求 Cargo 克隆整个仓库。

`sparse` 协议使用纯 HTTP 请求获取单个元数据文件。
由于 Cargo 仅下载相关 crate 的元数据，`sparse` 协议可以节省大量时间和带宽。

[crates.io] 注册中心支持两种协议。crates.io 的协议通过 [`registries.crates-io.protocol`] 配置键控制。

[Source Replacement]: source-replacement.md
[Running a Registry]: running-a-registry.md
[Credential Provider Protocol]: credential-provider-protocol.md
[Registry Authentication]: registry-authentication.md
[`cargo publish`]: ../commands/cargo-publish.md
[`cargo package`]: ../commands/cargo-package.md
[`cargo login`]: ../commands/cargo-login.md
[config]: config.md
[crates.io]: https://crates.io/
[`registries.crates-io.protocol`]: config.md#registriescrates-ioprotocol