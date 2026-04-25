# 注册中心认证 {#registry-authentication}
Cargo 通过凭据提供程序向注册中心进行身份认证。这些凭据提供程序是可执行的外部程序或内置提供程序，Cargo 使用它们来存储和检索凭据。

使用需要认证的备用注册中心*必须*配置凭据提供程序，以避免在不知情的情况下将未加密的凭据存储在磁盘上。出于历史原因，公共（非认证）注册中心不需要配置凭据提供程序，如果未配置任何提供程序，则将使用 `cargo:token` 提供程序。

Cargo 还包含特定于平台的提供程序，这些提供程序使用操作系统来安全地存储令牌。还包括 `cargo:token` 提供程序，它以未加密的纯文本形式将凭据存储在 [credentials](config.md#credentials) 文件中。

## 推荐配置 {#recommended-configuration}
建议在 `$CARGO_HOME/config.toml` 中配置全局凭据提供程序列表，其默认路径为：
* Windows: `%USERPROFILE%\.cargo\config.toml`
* Unix: `~/.cargo/config.toml`

此推荐配置使用操作系统提供程序，并回退到 `cargo:token` 以在 Cargo 的 [credentials](config.md#credentials) 文件或环境变量中查找：
```toml
# ~/.cargo/config.toml
[registry]
global-credential-providers = ["cargo:token", "cargo:libsecret", "cargo:macos-keychain", "cargo:wincred"]
```
*注意，列表中靠后的条目具有更高的优先级。
有关更多详细信息，请参阅 [`registry.global-credential-providers`](config.md#registryglobal-credential-providers)。*

某些私有注册中心也可能推荐特定于注册中心的凭据提供程序。请查看您的注册中心文档以确认是否如此。

## 内置提供程序 {#built-in-providers}
Cargo 包含几个内置的凭据提供程序。可用的内置提供程序可能会在未来的 Cargo 版本中发生变化（尽管目前没有这样做的计划）。

### `cargo:token`
使用 Cargo 的 [credentials](config.md#credentials) 文件以纯文本形式未加密地存储令牌。
检索令牌时，会检查 `CARGO_REGISTRIES_<NAME>_TOKEN` 环境变量。
如果未列出此凭据提供程序，则 `*_TOKEN` 环境变量将无法工作。

### `cargo:wincred`
使用 Windows 凭据管理器存储令牌。

凭据以 `cargo-registry:<index-url>` 的形式存储在凭据管理器的“Windows 凭据”下。

### `cargo:macos-keychain`
使用 macOS Keychain 存储令牌。

可以使用 Keychain Access 应用程序查看存储的令牌。

### `cargo:libsecret`
使用 [libsecret](https://wiki.gnome.org/Projects/Libsecret) 存储令牌。

任何支持 libsecret 的密码管理器都可以用来查看存储的令牌。以下是一些示例（非详尽）：

- [GNOME Keyring](https://wiki.gnome.org/Projects/GnomeKeyring)
- [KDE Wallet Manager](https://apps.kde.org/kwalletmanager5/)（自 KDE Frameworks 5.97.0 起）
- [KeePassXC](https://keepassxc.org/)（自 2.5.0 起）

### `cargo:token-from-stdout <command> <args>`
启动一个子进程，该进程在 stdout 上返回一个令牌。换行符将被修剪。
* 该进程继承用户的 stdin 和 stderr。
* 成功时应以退出码 0 退出，错误时应以非零退出码退出。
* 不支持 [`cargo login`] 和 [`cargo logout`]，如果使用则会返回错误。

以下环境变量将提供给执行的命令：

* `CARGO` --- 执行命令的 `cargo` 二进制文件的路径。
* `CARGO_REGISTRY_INDEX_URL` --- 注册中心索引的 URL。
* `CARGO_REGISTRY_NAME_OPT` --- 可选的注册中心名称。不应作为查找键使用。

参数将传递给子命令。

[`cargo login`]: ../commands/cargo-login.md
[`cargo logout`]: ../commands/cargo-logout.md

## 凭据插件 {#credential-plugins}
对于遵循 Cargo 的 [凭据提供程序协议](credential-provider-protocol.md) 的凭据提供程序插件，配置值应为一个字符串，其中包含可执行文件的路径（如果在 `PATH` 中，则为可执行文件名称）。

例如，要从 crates.io 安装 [cargo-credential-1password](https://crates.io/crates/cargo-credential-1password)，请执行以下操作：

使用 `cargo install cargo-credential-1password` 安装提供程序。

在配置中，添加到（或创建）`registry.global-credential-providers`：
```toml
[registry]
global-credential-providers = ["cargo:token", "cargo-credential-1password --account my.1password.com"]
```

`global-credential-providers` 中的值按空格拆分为路径和命令行参数。要定义路径或参数包含空格的全局凭据提供程序，请使用 [`[credential-alias]` 表](config.md#credential-alias)。