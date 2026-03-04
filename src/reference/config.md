# config.toml {#configuration}

本文档解释了 Cargo 的配置系统如何工作，以及可用的键或配置。关于通过清单文件配置包的信息，请参阅[清单格式](manifest.md)。

## 层次结构 {#hierarchical-structure}

Cargo 允许针对特定包的本地配置以及全局配置。它会在当前目录及其所有父目录中查找配置文件。例如，如果在 `/projects/foo/bar/baz` 中调用 Cargo，则会按以下顺序探测并合并以下配置文件：

* `/projects/foo/bar/baz/.cargo/config.toml`
* `/projects/foo/bar/.cargo/config.toml`
* `/projects/foo/.cargo/config.toml`
* `/projects/.cargo/config.toml`
* `/.cargo/config.toml`
* `$CARGO_HOME/config.toml`，默认为：
    * Windows：`%USERPROFILE%\.cargo\config.toml`
    * Unix：`$HOME/.cargo/config.toml`

通过这种结构，你可以为每个包指定配置，甚至可能将其检入版本控制。你也可以在 home 目录中使用配置文件来指定个人默认设置。

如果一个键在多个配置文件中被指定，这些值将被合并。数字、字符串和布尔值将使用更深层配置目录中的值，优先于祖先目录，其中 home 目录的优先级最低。数组将被连接在一起，优先级较高的项被放置在合并数组的后面。

目前，当从工作空间调用时，Cargo 不会读取工作空间内 crate 的配置文件。例如，如果一个工作空间中有两个 crate，分别名为 `/projects/foo/bar/baz/mylib` 和 `/projects/foo/bar/baz/mybin`，并且在 `/projects/foo/bar/baz/mylib/.cargo/config.toml` 和 `/projects/foo/bar/baz/mybin/.cargo/config.toml` 处有 Cargo 配置，如果从工作空间根目录（`/projects/foo/bar/baz/`）调用 Cargo，则不会读取这些配置文件。

> **注意：** Cargo 也会读取不带 `.toml` 扩展名的配置文件，例如 `.cargo/config`。对 `.toml` 扩展名的支持是在 1.39 版本中添加的，并且是首选形式。如果两个文件都存在，Cargo 将使用不带扩展名的文件。

## 配置格式 {#configuration-format}

配置文件使用 [TOML 格式][toml]编写（类似于清单文件），其中包含部分（表）内的简单键值对。以下是所有设置的快速概述，详细描述见下文。

```toml
paths = ["/path/to/override"] # 路径依赖覆盖

[alias]     # 命令别名
b = "build"
c = "check"
t = "test"
r = "run"
rr = "run --release"
recursive_example = "rr --example recursions"
space_example = ["run", "--release", "--", "\"command list\""]

[build]
jobs = 1                      # 并行作业数，默认为 CPU 数量
rustc = "rustc"               # rust 编译器工具
rustc-wrapper = "…"           # 运行此包装器而不是 `rustc`
rustc-workspace-wrapper = "…" # 对工作空间成员运行此包装器而不是 `rustc`
rustdoc = "rustdoc"           # 文档生成工具
target = "triple"             # 为目标三元组构建（`cargo install` 忽略）
target-dir = "target"         # 放置生成产物的路径
build-dir = "target"          # 放置中间构建产物的路径
rustflags = ["…", "…"]        # 传递给所有编译器调用的自定义标志
rustdocflags = ["…", "…"]     # 传递给 rustdoc 的自定义标志
incremental = true            # 是否启用增量编译
dep-info-basedir = "…"        # depfiles 中目标的基础目录路径

[credential-alias]
# 提供一种定义凭证提供者别名的方式。
my-alias = ["/usr/bin/cargo-credential-example", "--argument", "value", "--flag"]

[doc]
browser = "chromium"          # 与 `cargo doc --open` 一起使用的浏览器，
                              # 覆盖 `BROWSER` 环境变量

[env]
# 为 Cargo 运行的任何进程设置 ENV_VAR_NAME=value
ENV_VAR_NAME = "value"
# 即使环境中已存在也设置
ENV_VAR_NAME_2 = { value = "value", force = true }
# `value` 相对于 `.cargo/config.toml` 的父目录，环境变量将是完整的绝对路径
ENV_VAR_NAME_3 = { value = "relative/path", relative = true }

[future-incompat-report]
frequency = 'always' # 何时显示关于未来不兼容报告的通知

[cache]
auto-clean-frequency = "1 day"   # 执行自动缓存清理的频率

[cargo-new]
vcs = "none"              # 使用的 VCS（'git'、'hg'、'pijul'、'fossil'、'none'）

[http]
debug = false               # HTTP 调试
proxy = "host:port"         # libcurl 格式的 HTTP 代理
ssl-version = "tlsv1.3"     # 使用的 TLS 版本
ssl-version.max = "tlsv1.3" # 最大 TLS 版本
ssl-version.min = "tlsv1.1" # 最小 TLS 版本
timeout = 30                # 每个 HTTP 请求的超时时间，单位秒
low-speed-limit = 10        # 网络超时阈值（字节/秒）
cainfo = "cert.pem"         # 证书颁发机构（CA）捆绑包的路径
proxy-cainfo = "cert.pem"   # 代理证书颁发机构（CA）捆绑包的路径
check-revoke = true         # 检查 SSL 证书吊销
multiplexing = true         # HTTP/2 多路复用
user-agent = "…"            # 用户代理头

[install]
root = "/some/path"         # `cargo install` 目标目录

[net]
retry = 3                   # 网络重试次数
git-fetch-with-cli = true   # 使用 `git` 可执行文件进行 git 操作
offline = true              # 不访问网络

[net.ssh]
known-hosts = ["..."]       # 已知的 SSH 主机密钥

[patch.<registry>]
# 与 Cargo.toml 中的 [patch] 相同的键

[profile.<name>]         # 通过配置修改配置文件设置。
inherits = "dev"         # 从 [profile.dev] 继承设置。
opt-level = 0            # 优化级别。
debug = true             # 包含调试信息。
split-debuginfo = '...'  # 调试信息拆分行为。
strip = "none"           # 移除符号或调试信息。
debug-assertions = true  # 启用调试断言。
overflow-checks = true   # 启用运行时整数溢出检查。
lto = false              # 设置链接时优化。
panic = 'unwind'         # panic 策略。
incremental = true       # 增量编译。
codegen-units = 16       # 代码生成单元数量。
rpath = false            # 设置 rpath 链接选项。
[profile.<name>.build-override]  # 覆盖构建脚本设置。
# 与普通配置文件相同的键。
[profile.<name>.package.<name>]  # 覆盖包的配置文件。
# 与普通配置文件相同的键（减去 `panic`、`lto` 和 `rpath`）。

[resolver]
incompatible-rust-versions = "allow"  # 指定解析器如何响应这些情况

[registries.<name>]  # crates.io 以外的注册中心
index = "…"          # 注册中心索引的 URL
token = "…"          # 注册中心的认证令牌
credential-provider = "cargo:token" # 此注册中心的凭证提供者。

[registries.crates-io]
protocol = "sparse"  # 用于访问 crates.io 的协议。

[registry]
default = "…"        # 默认注册中心的名称
token = "…"          # crates.io 的认证令牌
credential-provider = "cargo:token"           # crates.io 的凭证提供者。
global-credential-providers = ["cargo:token"] # 默认使用的凭证提供者。

[source.<name>]      # 源定义和替换
replace-with = "…"   # 用给定的命名源替换此源
directory = "…"      # 目录源的路径
registry = "…"       # 注册中心源的 URL
local-registry = "…" # 本地注册中心源的路径
git = "…"            # git 仓库源的 URL
branch = "…"         # git 仓库的分支名
tag = "…"            # git 仓库的标签名
rev = "…"            # git 仓库的修订版本

[target.<triple>]
linker = "…"              # 使用的链接器
runner = "…"              # 运行可执行文件的包装器
rustflags = ["…", "…"]    # `rustc` 的自定义标志
rustdocflags = ["…", "…"] # `rustdoc` 的自定义标志

[target.<cfg>]
linker = "…"            # 使用的链接器
runner = "…"            # 运行可执行文件的包装器
rustflags = ["…", "…"]  # `rustc` 的自定义标志

[target.<triple>.<links>] # `links` 构建脚本覆盖
rustc-link-lib = ["foo"]
rustc-link-search = ["/path/to/foo"]
rustc-flags = "-L /some/path"
rustc-cfg = ['key="value"']
rustc-env = {key = "value"}
rustc-cdylib-link-arg = ["…"]
metadata_key1 = "value"
metadata_key2 = "value"

[term]
quiet = false                    # cargo 输出是否静默
verbose = false                  # cargo 是否提供详细输出
color = 'auto'                   # cargo 是否对输出着色
hyperlinks = true                # cargo 是否在输出中插入链接
unicode = true                   # cargo 是否可以使用非 ASCII unicode 字符渲染输出
progress.when = 'auto'           # cargo 是否显示进度条
progress.width = 80              # 进度条宽度
progress.term-integration = true # cargo 是否向终端模拟器报告进度
```

## 环境变量 {#environment-variables}

除了 TOML 配置文件外，Cargo 还可以通过环境变量进行配置。对于每个形式为 `foo.bar` 的配置键，也可以使用环境变量 `CARGO_FOO_BAR` 来定义值。键被转换为大写，点和短横线被转换为下划线。例如，`target.x86_64-unknown-linux-gnu.runner` 键也可以通过 `CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_RUNNER` 环境变量定义。

环境变量将优先于 TOML 配置文件。目前只有整数、布尔值、字符串和一些数组值支持通过环境变量定义。[下面的描述](#configuration-keys)指出了哪些键支持环境变量，否则由于[技术问题](https://github.com/rust-lang/cargo/issues/5416)而不支持。

除了上述系统外，Cargo 还识别一些其他特定的[环境变量][env]。

## 命令行覆盖 {#command-line-overrides}

Cargo 还通过 `--config` 命令行选项接受任意的配置覆盖。参数应为 `KEY=VALUE` 的 TOML 语法，或作为额外配置文件的路径提供：

```console
# 使用 TOML 语法的 `KEY=VALUE`
cargo --config net.git-fetch-with-cli=true fetch

# 使用配置文件的路径
cargo --config ./path/to/my/extra-config.toml fetch
```

`--config` 选项可以指定多次，在这种情况下，值按从左到右的顺序合并，使用与多个配置文件应用时相同的合并逻辑。以此方式指定的配置值优先于环境变量，环境变量优先于配置文件。

当 `--config` 选项作为额外配置文件提供时，以此方式加载的配置文件遵循与直接使用 `--config` 指定的其他选项相同的优先级规则。

以下是一些使用 Bourne shell 语法的示例：

```console
# 大多数 shell 需要转义。
cargo --config http.proxy=\"http://example.com\" …

# 可以使用空格。
cargo --config "net.git-fetch-with-cli = true" …

# TOML 数组示例。单引号使其更易于读写。
cargo --config 'build.rustdocflags = ["--html-in-header", "header.html"]' …

# 复杂 TOML 键的示例。
cargo --config "target.'cfg(all(target_arch = \"arm\", target_os = \"none\"))'.runner = 'my-runner'" …

# 覆盖配置文件设置的示例。
cargo --config profile.dev.package.image.opt-level=3 …
```

## 配置相对路径 {#config-relative-paths}

配置文件中的路径可以是绝对路径、相对路径或不带任何路径分隔符的裸名称。不带路径分隔符的可执行文件路径将使用 `PATH` 环境变量搜索可执行文件。非可执行文件的路径将相对于定义配置值的位置。

具体规则如下：

* 对于环境变量，路径相对于当前工作目录。
* 对于直接从 [`--config KEY=VALUE`](#command-line-overrides) 选项加载的配置值，路径相对于当前工作目录。
* 对于配置文件，路径相对于定义配置文件的目录的父目录，无论这些文件是来自[层次结构探测](#hierarchical-structure)还是 [`--config <path>`](#command-line-overrides) 选项。

> **注意：** 为了与现有的 `.cargo/config.toml` 探测行为保持一致，通过 `--config <path>` 传递的配置文件中的路径也相对于配置文件本身向上两级目录，这是设计使然。
>
> 为避免意外结果，经验法则是将额外的配置文件放在项目中发现的 `.cargo/config.toml` 的同一级别。例如，给定一个项目 `/my/project`，建议将配置文件放在 `/my/project/.cargo` 下或同一级别的新目录中，例如 `/my/project/.config`。

```toml
# 相对路径示例。

[target.x86_64-unknown-linux-gnu]
runner = "foo"  # 在 `PATH` 中搜索 `foo`。

[source.vendored-sources]
# 目录相对于包含 `.cargo/config.toml` 的父目录。
# 例如，`/my/project/.cargo/config.toml` 将导致 `/my/project/vendor`。
directory = "vendor"
```

## 带参数的可执行文件路径 {#executable-paths-with-arguments}

一些 Cargo 命令会调用外部程序，这些程序可以配置为路径和一些参数。

值可以是字符串数组，如 `['/path/to/program', 'somearg']`，也可以是空格分隔的字符串，如 `'/path/to/program somearg'`。如果可执行文件的路径包含空格，则必须使用列表形式。

如果 Cargo 向程序传递其他参数，例如要打开或运行的路径，它们将在值的最后一个指定参数之后传递。如果指定的程序没有路径分隔符，Cargo 将在 `PATH` 中搜索其可执行文件。

## 凭证 {#credentials}

包含敏感信息的配置值存储在 `$CARGO_HOME/credentials.toml` 文件中。当使用 [`cargo:token`] 凭证提供者时，此文件由 [`cargo login`] 和 [`cargo logout`] 自动创建和更新。

令牌被一些 Cargo 命令使用，例如 [`cargo publish`]，用于与远程注册中心进行身份验证。应注意保护令牌并保持其机密性。

它遵循与 Cargo 配置文件相同的格式。

```toml
[registry]
token = "…"   # crates.io 的访问令牌

[registries.<name>]
token = "…"   # 命名注册中心的访问令牌
```

与大多数其他配置值一样，令牌可以通过环境变量指定。[crates.io] 的令牌可以通过 `CARGO_REGISTRY_TOKEN` 环境变量指定。其他注册中心的令牌可以通过形式为 `CARGO_REGISTRIES_<name>_TOKEN` 的环境变量指定，其中 `<name>` 是注册中心名称的大写形式。

> **注意：** Cargo 也会读取和写入不带 `.toml` 扩展名的凭证文件，例如 `.cargo/credentials`。对 `.toml` 扩展名的支持是在 1.39 版本中添加的。在 1.68 版本中，Cargo 默认写入带扩展名的文件。然而，出于向后兼容的原因，当两个文件都存在时，Cargo 将读取和写入不带扩展名的文件。

## 配置键 {#configuration-keys}

本节记录所有配置键。具有可变部分的键的描述用尖括号注释，如 `target.<triple>`，其中 `<triple>` 部分可以是任何[目标三元组][target triple]，如 `target.x86_64-pc-windows-msvc`。

### `paths`
* 类型：字符串数组（路径）
* 默认值：无
* 环境：不支持

本地包的路径数组，用作依赖项的覆盖。更多信息请参阅[覆盖依赖项指南](overriding-dependencies.md#paths-overrides)。

### `[alias]`
* 类型：字符串或字符串数组
* 默认值：见下文
* 环境：`CARGO_ALIAS_<name>`

`[alias]` 表定义了 CLI 命令别名。例如，运行 `cargo b` 是运行 `cargo build` 的别名。表中的每个键是子命令，值是要运行的实际命令。值可以是字符串数组，其中第一个元素是命令，后面的是参数。也可以是字符串，将按空格拆分为子命令和参数。以下别名是 Cargo 内置的：

```toml
[alias]
b = "build"
c = "check"
d = "doc"
t = "test"
r = "run"
rm = "remove"
```

不允许别名重定义现有的内置命令。

别名是递归的：

```toml
[alias]
rr = "run --release"
recursive_example = "rr --example recursions"
```

### `[build]`

`[build]` 表控制构建时操作和编译器设置。

#### `build.jobs`
* 类型：整数或字符串
* 默认值：逻辑 CPU 数量
* 环境：`CARGO_BUILD_JOBS`

设置并行运行的编译器进程的最大数量。如果为负数，则将编译器进程的最大数量设置为逻辑 CPU 数量加上提供的值。不应为 0。如果提供字符串 `default`，则将值重置为默认值。

可以通过 `--jobs` CLI 选项覆盖。

#### `build.rustc`
* 类型：字符串（程序路径）
* 默认值：`"rustc"`
* 环境：`CARGO_BUILD_RUSTC` 或 `RUSTC`

设置用于 `rustc` 的可执行文件。

#### `build.rustc-wrapper`
* 类型：字符串（程序路径）
* 默认值：无
* 环境：`CARGO_BUILD_RUSTC_WRAPPER` 或 `RUSTC_WRAPPER`

设置一个包装器来执行，而不是 `rustc`。传递给包装器的第一个参数是要使用的实际可执行文件的路径（即，如果设置了 `build.rustc`，则为该路径，否则为 `"rustc"`）。

#### `build.rustc-workspace-wrapper`
* 类型：字符串（程序路径）
* 默认值：无
* 环境：`CARGO_BUILD_RUSTC_WORKSPACE_WRAPPER` 或 `RUSTC_WORKSPACE_WRAPPER`

设置一个包装器来执行，而不是 `rustc`，仅适用于工作空间成员。当构建没有工作空间的单包项目时，该包被视为工作空间。传递给包装器的第一个参数是要使用的实际可执行文件的路径（即，如果设置了 `build.rustc`，则为该路径，否则为 `"rustc"`）。它影响文件名哈希，以便包装器生成的产物被单独缓存。

如果同时设置了 `rustc-wrapper` 和 `rustc-workspace-wrapper`，则它们将被嵌套：最终调用是 `$RUSTC_WRAPPER $RUSTC_WORKSPACE_WRAPPER $RUSTC`。

#### `build.rustdoc`
* 类型：字符串（程序路径）
* 默认值：`"rustdoc"`
* 环境：`CARGO_BUILD_RUSTDOC` 或 `RUSTDOC`

设置用于 `rustdoc` 的可执行文件。

#### `build.target`
* 类型：字符串或字符串数组
* 默认值：主机平台
* 环境：`CARGO_BUILD_TARGET`

默认编译的[目标平台三元组][target triple]。

可能的值：
- `rustc --print target-list` 中任何支持的目标。
- `"host-tuple"`，将在内部替换为主机的目标。这在交叉编译某些 crate 且不想将主机机器指定为目标时特别有用（例如，共享项目中可能由许多主机处理的 `xtask`）。
- 自定义目标规范文件的路径。更多信息请参阅[自定义目标查找路径](../../rustc/targets/custom.html#custom-target-lookup-path)。

可以通过 `--target` CLI 选项覆盖。

```toml
[build]
target = ["x86_64-unknown-linux-gnu", "i686-unknown-linux-gnu"]
```

#### `build.target-dir`
* 类型：字符串（路径）
* 默认值：`"target"`
* 环境：`CARGO_BUILD_TARGET_DIR` 或 `CARGO_TARGET_DIR`

所有编译器输出放置的路径。如果未指定，默认是工作空间根目录下名为 `target` 的目录。

可以通过 `--target-dir` 命令行选项覆盖。

更多信息请参阅[构建缓存文档](../reference/build-cache.md)。

#### `build.build-dir`

* 类型：字符串（路径）
* 默认值：默认为 `build.target-dir` 的值
* 环境：`CARGO_BUILD_BUILD_DIR`

存储中间构建产物的目录。中间产物是 Rustc/Cargo 在构建过程中产生的。

此选项支持路径模板。

可用的模板变量：
* `{workspace-root}` 解析为当前工作空间的根目录。
* `{cargo-cache-home}` 解析为 `CARGO_HOME`
* `{workspace-path-hash}` 解析为清单路径的哈希值

更多信息请参阅[构建缓存文档](../reference/build-cache.md)。

#### `build.rustflags`
* 类型：字符串或字符串数组
* 默认值：无
* 环境：`CARGO_BUILD_RUSTFLAGS` 或 `CARGO_ENCODED_RUSTFLAGS` 或 `RUSTFLAGS`

传递给 `rustc` 的额外命令行标志。值可以是字符串数组或空格分隔的字符串。

有四个互斥的额外标志来源。它们按顺序检查，使用第一个：

1. `CARGO_ENCODED_RUSTFLAGS` 环境变量。
2. `RUSTFLAGS` 环境变量。
3. 所有匹配的 `target.<triple>.rustflags` 和 `target.<cfg>.rustflags` 配置条目连接在一起。
4. `build.rustflags` 配置值。

额外的标志也可以通过 [`cargo rustc`] 命令传递。

如果使用了 `--target` 标志（或 [`build.target`](#buildtarget)），则标志仅传递给目标编译器。为主机构建的内容，如构建脚本或过程宏，将不会接收这些参数。如果没有 `--target`，标志将传递给所有编译器调用（包括构建脚本和过程宏），因为依赖项是共享的。如果你有不想传递给构建脚本或过程宏的参数，并且正在为主机构建，请使用[主机三元组][target triple]传递 `--target`。

不建议传递通常由 Cargo 本身管理的标志。例如，由[配置文件](profiles.md)驱动的标志最好通过设置适当的配置文件设置来处理。

> **注意**：由于直接将标志传递给编译器的低级性质，这可能会导致与未来版本的 Cargo 冲突，这些版本可能会自行发出相同或类似的标志，从而干扰你指定的标志。这是 Cargo 可能不总是向后兼容的领域。

#### `build.rustdocflags`
* 类型：字符串或字符串数组
* 默认值：无
* 环境：`CARGO_BUILD_RUSTDOCFLAGS` 或 `CARGO_ENCODED_RUSTDOCFLAGS` 或 `RUSTDOCFLAGS`

传递给 `rustdoc` 的额外命令行标志。值可以是字符串数组或空格分隔的字符串。

有四个互斥的额外标志来源。它们按顺序检查，使用第一个：

1. `CARGO_ENCODED_RUSTDOCFLAGS` 环境变量。
2. `RUSTDOCFLAGS` 环境变量。
3. 所有匹配的 `target.<triple>.rustdocflags` 配置条目连接在一起。
4. `build.rustdocflags` 配置值。

额外的标志也可以通过 [`cargo rustdoc`] 命令传递。

> **注意**：由于直接将标志传递给编译器的低级性质，这可能会导致与未来版本的 Cargo 冲突，这些版本可能会自行发出相同或类似的标志，从而干扰你指定的标志。这是 Cargo 可能不总是向后兼容的领域。

#### `build.incremental`
* 类型：布尔值
* 默认值：来自配置文件
* 环境：`CARGO_BUILD_INCREMENTAL` 或 `CARGO_INCREMENTAL`

是否执行[增量编译]。如果未设置，默认使用[配置文件](profiles.md#incremental)中的值。否则，这将覆盖所有配置文件的设置。

`CARGO_INCREMENTAL` 环境变量可以设置为 `1` 以强制为所有配置文件启用增量编译，或设置为 `0` 以禁用它。此环境变量覆盖配置设置。

#### `build.dep-info-basedir`
* 类型：字符串（路径）
* 默认值：无
* 环境：`CARGO_BUILD_DEP_INFO_BASEDIR`

从[依赖信息文件](../reference/build-cache.md#dep-info-files)路径中剥离给定的路径前缀。此配置设置旨在将绝对路径转换为相对路径，以满足需要相对路径的工具。

设置本身是一个配置相对路径。例如，值 `"."` 将剥离所有以 `.cargo` 目录的父目录开头的路径。

#### `build.pipelining`

此选项已弃用且未使用。Cargo 始终启用流水线。

### `[credential-alias]`
* 类型：字符串或字符串数组
* 默认值：空
* 环境：`CARGO_CREDENTIAL_ALIAS_<name>`

`[credential-alias]` 表定义了凭证提供者别名。这些别名可以作为 `registry.global-credential-providers` 数组的元素引用，或作为特定注册中心的凭证提供者，在 `registries.<NAME>.credential-provider` 下引用。

如果指定为字符串，值将按空格拆分为路径和参数。

例如，要定义一个名为 `my-alias` 的别名：

```toml
[credential-alias]
my-alias = ["/usr/bin/cargo-credential-example", "--argument", "value", "--flag"]
```
更多信息请参阅[注册中心认证](registry-authentication.md)。

### `[doc]`

`[doc]` 表定义了 [`cargo doc`] 命令的选项。

#### `doc.browser`

* 类型：字符串或字符串数组（[带参数的程序路径]）
* 默认值：`BROWSER` 环境变量，如果缺失，则以系统特定方式打开链接

此选项设置 [`cargo doc`] 使用的浏览器，在使用 `--open` 选项打开文档时覆盖 `BROWSER` 环境变量。

### `[cargo-new]`

`[cargo-new]` 表定义了 [`cargo new`] 命令的默认值。

#### `cargo-new.name`

此选项已弃用且未使用。

#### `cargo-new.email`

此选项已弃用且未使用。

#### `cargo-new.vcs`
* 类型：字符串
* 默认值：`"git"` 或 `"none"`
* 环境：`CARGO_CARGO_NEW_VCS`

指定用于初始化新仓库的源代码管理系统。有效值为 `git`、`hg`（用于 Mercurial）、`pijul`、`fossil` 或 `none` 以禁用此行为。默认为 `git`，如果已在 VCS 仓库中，则默认为 `none`。可以通过 `--vcs` CLI 选项覆盖。

### `[env]`

`[env]` 部分允许你为构建脚本、rustc 调用、`cargo run` 和 `cargo build` 设置额外的环境变量。

```toml
[env]
OPENSSL_DIR = "/opt/openssl"
```

默认情况下，指定的变量不会覆盖环境中已存在的值。可以通过设置 `force` 标志来改变此行为。

设置 `relative` 标志会将值评估为配置相对路径，该路径相对于包含 `config.toml` 文件的 `.cargo` 目录的父目录。环境变量的值将是完整的绝对路径。

```toml
[env]
TMPDIR = { value = "/home/tmp", force = true }
OPENSSL_DIR = { value = "vendor/openssl", relative = true }
```

### `[future-incompat-report]`

`[future-incompat-report]` 表控制[未来不兼容报告](future-incompat-report.md)的设置。

#### `future-incompat-report.frequency`
* 类型：字符串
* 默认值：`"always"`
* 环境：`CARGO_FUTURE_INCOMPAT_REPORT_FREQUENCY`

控制当未来不兼容报告可用时，我们向终端显示通知的频率。可能的值：

* `always`（默认）：当命令（例如 `cargo build`）产生未来不兼容报告时，始终显示通知。
* `never`：从不显示通知。

### `[cache]`

`[cache]` 表定义了 Cargo 缓存的设置。

#### 全局缓存 {#global-caches}

运行 `cargo` 命令时，Cargo 会自动跟踪你在全局缓存中使用的文件。Cargo 会定期删除一段时间未使用的文件。如果从网络下载的文件在 3 个月内未使用，Cargo 将删除它们。无需网络访问即可生成的文件如果在 1 个月内未使用，将被删除。

自动删除文件仅发生在已经执行大量工作的命令时，例如所有构建命令（`cargo build`、`cargo test`、`cargo check` 等）和 `cargo fetch`。

如果 Cargo 处于离线状态，例如使用 `--offline` 或 `--frozen`，则禁用自动删除，以避免删除如果你长时间离线可能需要使用的产物。

> **注意**：目前仅对 Cargo home 目录中的全局缓存实现了此跟踪。这包括从注册中心和 git 依赖项下载的注册中心索引和源文件。对构建产物的跟踪尚未实现，并在 [cargo#13136](https://github.com/rust-lang/cargo/issues/13136) 中跟踪。
>
> 此外，有一个不稳定功能支持*手动*触发缓存清理，并进一步自定义配置选项。更多信息请参阅[不稳定章节](unstable.md#gc)。

#### `cache.auto-clean-frequency`
* 类型：字符串
* 默认值：`"1 day"`
* 环境：`CARGO_CACHE_AUTO_CLEAN_FREQUENCY`

此选项定义 Cargo 自动删除全局缓存中未使用文件的频率。这*不*定义文件必须有多旧，这些阈值在[上文](#global-caches)描述。

支持以下设置：

* `"never"` --- 从不删除旧文件。
* `"always"` --- 每次 Cargo 运行时都检查删除旧文件。
* 一个整数后跟 "seconds"、"minutes"、"hours"、"days"、"weeks" 或 "months" --- 最多在给定的时间范围内检查删除旧文件。

### `[http]`

`[http]` 表定义了 HTTP 行为的设置。这包括获取 crate 依赖项和访问远程 git 仓库。

#### `http.debug`
* 类型：布尔值
* 默认值：false
* 环境：`CARGO_HTTP_DEBUG`

如果为 `true`，则启用 HTTP 请求的调试。可以通过设置 `CARGO_LOG=network=debug` 环境变量（或使用 `network=trace` 获取更多信息）查看调试信息。

在公共位置发布此输出的日志时要小心。输出可能包含带有认证令牌的头部，你不想泄露！发布日志前务必检查。

#### `http.proxy`
* 类型：字符串
* 默认值：无
* 环境：`CARGO_HTTP_PROXY` 或 `HTTPS_PROXY` 或 `https_proxy` 或 `http_proxy`

设置要使用的 HTTP 和 HTTPS 代理。格式为 [libcurl 格式]，如 `[protocol://]host[:port]`。如果未设置，Cargo 还将检查全局 git 配置中的 `http.proxy` 设置。如果这些都未设置，`HTTPS_PROXY` 或 `https_proxy` 环境变量为 HTTPS 请求设置代理，`http_proxy` 为 HTTP 请求设置代理。

#### `http.timeout`
* 类型：整数
* 默认值：30
* 环境：`CARGO_HTTP_TIMEOUT` 或 `HTTP_TIMEOUT`

设置每个 HTTP 请求的超时时间，单位秒。

#### `http.cainfo`
* 类型：字符串（路径）
* 默认值：无
* 环境：`CARGO_HTTP_CAINFO`

证书颁发机构（CA）捆绑文件的路径，用于验证 TLS 证书。如果未指定，Cargo 尝试使用系统证书。

#### `http.proxy-cainfo`
* 类型：字符串（路径）
* 默认值：如果未设置，则回退到 `http.cainfo`
* 环境：`CARGO_HTTP_PROXY_CAINFO`

证书颁发机构（CA）捆绑文件的路径，用于验证代理 TLS 证书。

#### `http.check-revoke`
* 类型：布尔值
* 默认值：true（Windows）false（所有其他系统）
* 环境：`CARGO_HTTP_CHECK_REVOKE`

这决定是否应执行 TLS 证书吊销检查。这仅在 Windows 上有效。

#### `http.ssl-version`
* 类型：字符串或最小/最大表
* 默认值：无
* 环境：`CARGO_HTTP_SSL_VERSION`

这设置要使用的最小 TLS 版本。它接受一个字符串，可能值为 `"default"`、`"tlsv1"`、`"tlsv1.0"`、`"tlsv1.1"`、`"tlsv1.2"` 或 `"tlsv1.3"`。

或者，这可以接受一个包含两个键 `min` 和 `max` 的表，每个键接受相同类型的字符串值，指定要使用的 TLS 版本的最小和最大范围。

默认是最小版本 `"tlsv1.0"`，最大版本为你的平台支持的最新版本，通常是 `"tlsv1.3"`。

#### `http.low-speed-limit`
* 类型：整数
* 默认值：10
* 环境：`CARGO_HTTP_LOW_SPEED_LIMIT`

此设置控制慢速连接的 timeout 行为。如果平均传输速度（字节/秒）在 [`http.timeout`](#httptimeout) 秒（默认 30 秒）内低于给定值，则认为连接太慢，Cargo 将中止并重试。

#### `http.multiplexing`
* 类型：布尔值
* 默认值：true
* 环境：`CARGO_HTTP_MULTIPLEXING`

当为 `true` 时，Cargo 将尝试使用带有多路复用的 HTTP2 协议。这允许多个请求使用同一连接，通常在获取多个文件时提高性能。如果为 `false`，Cargo 将使用不带流水线的 HTTP 1.1。

#### `http.user-agent`
* 类型：字符串
* 默认值：Cargo 的版本
* 环境：`CARGO_HTTP_USER_AGENT`

指定要使用的自定义用户代理头。如果未指定，默认是包含 Cargo 版本的字符串。

### `[install]`

`[install]` 表定义了 [`cargo install`] 命令的默认值。

#### `install.root`
* 类型：字符串（路径）
* 默认值：Cargo 的 home 目录
* 环境：`CARGO_INSTALL_ROOT`

设置 [`cargo install`] 安装可执行文件的根目录路径。可执行文件进入根目录下的 `bin` 目录。

为了跟踪已安装可执行文件的信息，还会在此根目录下创建一些额外文件，例如 `.crates.toml` 和 `.crates2.json`。

如果未指定，默认是 Cargo 的 home 目录（默认为 home 目录中的 `.cargo`）。

可以通过 `--root` 命令行选项覆盖。

### `[net]`

`[net]` 表控制网络配置。

#### `net.retry`
* 类型：整数
* 默认值：3
* 环境：`CARGO_NET_RETRY`

重试可能虚假的网络错误的次数。

#### `net.git-fetch-with-cli`
* 类型：布尔值
* 默认值：false
* 环境：`CARGO_NET_GIT_FETCH_WITH_CLI`

如果为 `true`，则 Cargo 将使用 `git` 可执行文件来获取注册中心索引和 git 依赖项。如果为 `false`，则使用内置的 `git` 库。

如果你有 Cargo 不支持的特殊身份验证要求，将此设置为 `true` 可能会有所帮助。有关设置 git 身份验证的更多信息，请参阅 [Git 身份验证](../appendix/git-authentication.md)。

#### `net.offline`
* 类型：布尔值
* 默认值：false
* 环境：`CARGO_NET_OFFLINE`

如果为 `true`，则 Cargo 将避免访问网络，并尝试使用本地缓存的数据继续。如果为 `false`，Cargo 将在需要时访问网络，并在遇到网络错误时生成错误。

可以通过 `--offline` 命令行选项覆盖。

#### `net.ssh`

`[net.ssh]` 表包含 SSH 连接的设置。

#### `net.ssh.known-hosts`
* 类型：字符串数组
* 默认值：见描述
* 环境：不支持

`known-hosts` 数组包含 SSH 主机密钥列表，这些密钥在连接到 SSH 服务器（例如用于 SSH git 依赖项）时应被接受为有效。每个条目应为类似于 OpenSSH `known_hosts` 文件的格式的字符串。每个字符串应以一个或多个用逗号分隔的主机名开头，后跟一个空格、密钥类型名称、一个空格和 base64 编码的密钥。例如：

```toml
[net.ssh]
known-hosts = [
    "example.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFO4Q5T0UV0SQevair9PFwoxY9dl4pQl3u5phoqJH3cF"
]
```

Cargo 将尝试从 OpenSSH 支持的常见位置加载已知主机密钥，并将这些密钥与 Cargo 配置文件中列出的任何密钥连接起来。如果任何匹配的条目具有正确的密钥，则允许连接。

Cargo 内置了 [github.com][github-keys] 的主机密钥。如果这些密钥发生变化，你可以将新密钥添加到配置或 known_hosts 文件中。

更多详细信息请参阅 [Git 身份验证](../appendix/git-authentication.md#ssh-known-hosts)。

[github-keys]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints

### `[patch]`

就像你可以在 `Cargo.toml` 中使用 [`[patch]`](overriding-dependencies.md#the-patch-section) 覆盖依赖项一样，你也可以在 Cargo 配置文件中覆盖它们，以将这些补丁应用于任何受影响的构建。格式与 `Cargo.toml` 中使用的格式相同。

由于 `.cargo/config.toml` 文件通常不检入源代码控制，因此应尽可能使用 `Cargo.toml` 进行补丁，以确保其他开发人员可以在自己的环境中编译你的 crate。通过 Cargo 配置文件进行补丁通常仅适用于补丁部分由外部构建工具自动生成的情况。

如果某个依赖项在 Cargo 配置文件和 `Cargo.toml` 文件中都被修补，则使用配置文件中的补丁。如果多个配置文件修补同一依赖项，则使用标准的 Cargo 配置合并，优先使用最接近当前目录定义的值，`$HOME/.cargo/config.toml` 的优先级最低。

此类 `[patch]` 部分中的相对 `path` 依赖项相对于它们出现的配置文件解析。

### `[profile]`

`[profile]` 表可用于全局更改配置文件设置，并覆盖 `Cargo.toml` 中指定的设置。它具有与 `Cargo.toml` 中指定的配置文件相同的语法和选项。有关选项的详细信息，请参阅[配置文件章节]。

[配置文件章节]: profiles.md

#### `[profile.<name>.build-override]`
* 环境：`CARGO_PROFILE_<name>_BUILD_OVERRIDE_<key>`

build-override 表覆盖构建脚本、过程宏及其依赖项的设置。它具有与普通配置文件相同的键。更多详细信息请参阅[覆盖部分](profiles.md#overrides)。

#### `[profile.<name>.package.<name>]`
* 环境：不支持

package 表覆盖特定包的设置。它具有与普通配置文件相同的键，减去 `panic`、`lto` 和 `rpath` 设置。更多详细信息请参阅[覆盖部分](profiles.md#overrides)。

#### `profile.<name>.codegen-units`
* 类型：整数
* 默认值：见配置文件文档。
* 环境：`CARGO_PROFILE_<name>_CODEGEN_UNITS`

见 [codegen-units](profiles.md#codegen-units)。

#### `profile.<name>.debug`
* 类型：整数或布尔值
* 默认值：见配置文件文档。
* 环境：`CARGO_PROFILE_<name>_DEBUG`

见 [debug](profiles.md#debug)。

#### `profile.<name>.split-debuginfo`
* 类型：字符串
* 默认值：见配置文件文档。
* 环境：`CARGO_PROFILE_<name>_SPLIT_DEBUGINFO`

见 [split-debuginfo](profiles.md#split-debuginfo)。

#### `profile.<name>.debug-assertions`
* 类型：布尔值
* 默认值：见配置文件文档。
* 环境：`CARGO_PROFILE_<name>_DEBUG_ASSERTIONS`

见 [debug-assertions](profiles.md#debug-assertions)。

#### `profile.<name>.incremental`
* 类型：布尔值
* 默认值：见配置文件文档。
* 环境：`CARGO_PROFILE_<name>_INCREMENTAL`

见 [incremental](profiles.md#incremental)。

#### `profile.<name>.lto`
* 类型：字符串或布尔值
* 默认值：见配置文件文档。
* 环境：`CARGO_PROFILE_<name>_LTO`

见 [lto](profiles.md#lto)。

#### `profile.<name>.overflow-checks`
* 类型：布尔值
* 默认值：见配置文件文档。
* 环境：`CARGO_PROFILE_<name>_OVERFLOW_CHECKS`

见 [overflow-checks](profiles.md#overflow-checks)。

#### `profile.<name>.opt-level`
* 类型：整数或字符串
* 默认值：见配置文件文档。
* 环境：`CARGO_PROFILE_<name>_OPT_LEVEL`

见 [opt-level](profiles.md#opt-level)。

#### `profile.<name>.panic`
* 类型：字符串
* 默认值：见配置文件文档。
* 环境：`CARGO_PROFILE_<name>_PANIC`

见 [panic](profiles.md#panic)。

#### `profile.<name>.rpath`
* 类型：布尔值
* 默认值：见配置文件文档。
* 环境：`CARGO_PROFILE_<name>_RPATH`

见 [rpath](profiles.md#rpath)。

#### `profile.<name>.strip`
* 类型：字符串或布尔值
* 默认值：见配置文件文档。
* 环境：`CARGO_PROFILE_<name>_STRIP`

见 [strip](profiles.md#strip)。

### `[resolver]`

`[resolver]` 表覆盖本地开发（例如排除 `cargo install`）的[依赖项解析行为](resolver.md)。

#### `resolver.incompatible-rust-versions`
* 类型：字符串
* 默认值：见 [`resolver`](resolver.md#resolver-versions) 文档
* 环境：`CARGO_RESOLVER_INCOMPATIBLE_RUST_VERSIONS`

解析要使用的依赖项版本时，选择如何处理具有不兼容 `package.rust-version` 的版本。值包括：
- `allow`：将 `rust-version` 不兼容的版本视为任何其他版本
- `fallback`：仅在没有其他版本匹配时才考虑 `rust-version` 不兼容的版本

可以通过以下方式覆盖：
- `--ignore-rust-version` CLI 选项
- 将依赖项的版本要求设置为高于任何具有兼容 `rust-version` 的版本
- 使用 `--precise` 向 `cargo update` 指定版本

更多详细信息请参阅[解析器](resolver.md#rust-version)章节。

> **MSRV：**
> - `allow` 在任何版本上都支持
> - `fallback` 从 1.84 版本开始被尊重

### `[registries]`

`[registries]` 表用于指定额外的[注册中心][registries]。它由每个命名注册中心的子表组成。

#### `registries.<name>.index`
* 类型：字符串（url）
* 默认值：无
* 环境：`CARGO_REGISTRIES_<name>_INDEX`

指定注册中心索引的 URL。

#### `registries.<name>.token`
* 类型：字符串
* 默认值：无
* 环境：`CARGO_REGISTRIES_<name>_TOKEN`

指定给定注册中心的认证令牌。此值应仅出现在[凭证](#credentials)文件中。用于需要身份验证的注册中心命令，如 [`cargo publish`]。

可以通过 `--token` 命令行选项覆盖。

#### `registries.<name>.credential-provider`
* 类型：字符串或路径和参数数组
* 默认值：无
* 环境：`CARGO_REGISTRIES_<name>_CREDENTIAL_PROVIDER`

指定给定注册中心的凭证提供者。如果未设置，将使用 [`registry.global-credential-providers`](#registryglobal-credential-providers) 中的提供者。

如果指定为字符串，路径和参数将按空格拆分。对于包含空格的路径或参数，请使用数组。

如果值存在于 [`[credential-alias]`](#credential-alias) 表中，则将使用别名。

更多信息请参阅[注册中心认证](registry-authentication.md)。

#### `registries.crates-io.protocol`
* 类型：字符串
* 默认值：`"sparse"`
* 环境：`CARGO_REGISTRIES_CRATES_IO_PROTOCOL`

指定用于访问 crates.io 的协议。允许的值为 `git` 或 `sparse`。

`git` 导致 Cargo 从 <https://github.com/rust-lang/crates.io-index/> 克隆所有曾经发布到 [crates.io] 的包的完整索引。由于索引的大小，这可能会影响性能。`sparse` 是一种较新的协议，它使用 HTTPS 仅从 <https://index.crates.io/> 下载必要的内容。在大多数情况下，这可以显著提高解析新依赖项的性能。

有关注册中心协议的更多信息，请参阅[注册中心章节](registries.md)。

### `[registry]`

`[registry]` 表控制未指定时使用的默认注册中心。

#### `registry.index`

此值不再被接受，不应使用。

#### `registry.default`
* 类型：字符串
* 默认值：`"crates-io"`
* 环境：`CARGO_REGISTRY_DEFAULT`

注册中心（来自 [`registries` 表](#registries)）的名称，默认用于注册中心命令，如 [`cargo publish`]。

可以通过 `--registry` 命令行选项覆盖。

#### `registry.credential-provider`
* 类型：字符串或路径和参数数组
* 默认值：无
* 环境：`CARGO_REGISTRY_CREDENTIAL_PROVIDER`

指定 [crates.io] 的凭证提供者。如果未设置，将使用 [`registry.global-credential-providers`](#registryglobal-credential-providers) 中的提供者。

如果指定为字符串，路径和参数将按空格拆分。对于包含空格的路径或参数，请使用数组。

如果值存在于 `[credential-alias]` 表中，则将使用别名。

更多信息请参阅[注册中心认证](registry-authentication.md)。

#### `registry.token`
* 类型：字符串
* 默认值：无
* 环境：`CARGO_REGISTRY_TOKEN`

指定 [crates.io] 的认证令牌。此值应仅出现在[凭证](#credentials)文件中。用于需要身份验证的注册中心命令，如 [`cargo publish`]。

可以通过 `--token` 命令行选项覆盖。

#### `registry.global-credential-providers`
* 类型：数组
* 默认值：`["cargo:token"]`
* 环境：`CARGO_REGISTRY_GLOBAL_CREDENTIAL_PROVIDERS`

指定全局凭证提供者列表。如果未使用 `registries.<name>.credential-provider` 为特定注册中心设置凭证提供者，Cargo 将使用此列表中的凭证提供者。列表末尾的提供者具有优先级。

路径和参数按空格拆分。如果路径或参数包含空格，则应在 [`[credential-alias]`](#credential-alias) 表中定义凭证提供者，并在此处通过其别名引用。

更多信息请参阅[注册中心认证](registry-authentication.md)。

### `[source]`

`[source]` 表定义可用的注册中心源。更多信息请参阅[源替换][Source Replacement]。它由每个命名源的子表组成。一个源应只定义一种类型（目录、注册中心、本地注册中心或 git）。

#### `source.<name>.replace-with`
* 类型：字符串
* 默认值：无
* 环境：不支持

如果设置，用给定的命名源或命名注册中心替换此源。

#### `source.<name>.directory`
* 类型：字符串（路径）
* 默认值：无
* 环境：不支持

设置用作目录源的路径。

#### `source.<name>.registry`
* 类型：字符串（url）
* 默认值：无
* 环境：不支持

设置用于注册中心源的 URL。

#### `source.<name>.local-registry`
* 类型：字符串（路径）
* 默认值：无
* 环境：不支持

设置用作本地注册中心源的路径。

#### `source.<name>.git`
* 类型：字符串（url）
* 默认值：无
* 环境：不支持

设置用于 git 仓库源的 URL。

#### `source.<name>.branch`
* 类型：字符串
* 默认值：无
* 环境：不支持

设置用于 git 仓库的分支名。

如果未设置 `branch`、`tag` 或 `rev` 中的任何一个，则默认为 `master` 分支。

#### `source.<name>.tag`
* 类型：字符串
* 默认值：无
* 环境：不支持

设置用于 git 仓库的标签名。

如果未设置 `branch`、`tag` 或 `rev` 中的任何一个，则默认为 `master` 分支。

#### `source.<name>.rev`
* 类型：字符串
* 默认值：无
* 环境：不支持

设置用于 git 仓库的[修订版本][revision]。

如果未设置 `branch`、`tag` 或 `rev` 中的任何一个，则默认为 `master` 分支。

### `[target]`

`[target]` 表用于指定特定平台目标的设置。它由一个子表组成，该子表可以是[平台三元组][target triple]或 [`cfg()` 表达式][`cfg()` expression]。如果目标平台匹配 `<triple>` 值或 `<cfg>` 表达式，则将使用给定的值。

```toml
[target.thumbv7m-none-eabi]
linker = "arm-none-eabi-gcc"
runner = "my-emulator"
rustflags = ["…", "…"]

[target.'cfg(all(target_arch = "arm", target_os = "none"))']
runner = "my-arm-wrapper"
rustflags = ["…", "…"]
```

`cfg` 值来自编译器内置的值（运行 `rustc --print=cfg` 查看）和传递给 `rustc` 的额外 `--cfg` 标志（例如在 `RUSTFLAGS` 中定义的）。不要尝试匹配 `debug_assertions`、`test`、Cargo 特性如 `feature="foo"` 或由[构建脚本][build scripts]设置的值。

如果使用目标规范 JSON 文件，[`<triple>`] 值是文件名主干。例如 `--target foo/bar.json` 将匹配 `[target.bar]`。

#### `target.<triple>.ar`

此选项已弃用且未使用。

#### `target.<triple>.linker`
* 类型：字符串（程序路径）
* 默认值：无
* 环境：`CARGO_TARGET_<triple>_LINKER`

指定传递给 `rustc` 的链接器（通过 [`-C linker`]），当为 [`<triple>`] 编译时。默认情况下，链接器不会被覆盖。

#### `target.<cfg>.linker`
这与[目标链接器](#targettriplelinker)类似，但使用 [`cfg()` 表达式][`cfg()` expression]。如果 [`<triple>`] 和 `<cfg>` runner 都匹配，则 `<triple>` 优先。如果多个 `<cfg>` runner 匹配当前目标，则会导致错误。

#### `target.<triple>.runner`
* 类型：字符串或字符串数组（[带参数的程序路径]）
* 默认值：无
* 环境：`CARGO_TARGET_<triple>_RUNNER`

如果提供了 runner，则目标 [`<triple>`] 的可执行文件将通过调用指定的 runner 并将实际可执行文件作为参数传递来执行。这适用于 [`cargo run`]、[`cargo test`] 和 [`cargo bench`] 命令。默认情况下，编译的可执行文件直接执行。

#### `target.<cfg>.runner`

这与[目标 runner](#targettriplerunner) 类似，但使用 [`cfg()` 表达式][`cfg()` expression]。如果 [`<triple>`] 和 `<cfg>` runner 都匹配，则 `<triple>` 优先。如果多个 `<cfg>` runner 匹配当前目标，则会导致错误。

#### `target.<triple>.rustflags`
* 类型：字符串或字符串数组
* 默认值：无
* 环境：`CARGO_TARGET_<triple>_RUSTFLAGS`

为此 [`<triple>`] 传递一组自定义标志给编译器。值可以是字符串数组或空格分隔的字符串。

有关指定额外标志的不同方式的更多详细信息，请参阅 [`build.rustflags`](#buildrustflags)。

#### `target.<cfg>.rustflags`

这与[目标 rustflags](#targettriplerustflags) 类似，但使用 [`cfg()` 表达式][`cfg()` expression]。如果多个 `<cfg>` 和 [`<triple>`] 条目匹配当前目标，则标志将连接在一起。

#### `target.<triple>.rustdocflags`
* 类型：字符串或字符串数组
* 默认值：无
* 环境：`CARGO_TARGET_<triple>_RUSTDOCFLAGS`

为此 [`<triple>`] 传递一组自定义标志给编译器。值可以是字符串数组或空格分隔的字符串。

有关指定额外标志的不同方式的更多详细信息，请参阅 [`build.rustdocflags`](#buildrustdocflags)。

#### `target.<triple>.<links>`

links 子表提供了一种[覆盖构建脚本][override a build script]的方式。当指定时，将不会运行给定 `links` 库的构建脚本，而是使用给定的值。

```toml
[target.x86_64-unknown-linux-gnu.foo]
rustc-link-lib = ["foo"]
rustc-link-search = ["/path/to/foo"]
rustc-flags = "-L /some/path"
rustc-cfg = ['key="value"']
rustc-env = {key = "value"}
rustc-cdylib-link-arg = ["…"]
metadata_key1 = "value"
metadata_key2 = "value"
```

### `[term]`

`[term]` 表控制终端输出和交互。

#### `term.quiet`
* 类型：布尔值
* 默认值：false
* 环境：`CARGO_TERM_QUIET`

控制 Cargo 是否显示日志消息。

指定 `--quiet` 标志将覆盖并强制静默输出。指定 `--verbose` 标志将覆盖并禁用静默输出。

#### `term.verbose`
* 类型：布尔值
* 默认值：false
* 环境：`CARGO_TERM_VERBOSE`

控制 Cargo 是否显示额外的详细消息。

指定 `--quiet` 标志将覆盖并禁用详细输出。指定 `--verbose` 标志将覆盖并强制详细输出。

#### `term.color`
* 类型：字符串
* 默认值：`"auto"`
* 环境：`CARGO_TERM_COLOR`

控制终端中是否使用彩色输出。可能的值：

* `auto`（默认）：自动检测终端是否支持颜色。
* `always`：始终显示颜色。
* `never`：从不显示颜色。

可以通过 `--color` 命令行选项覆盖。

#### `term.hyperlinks`
* 类型：布尔值
* 默认值：自动检测
* 环境：`CARGO_TERM_HYPERLINKS`

控制终端中是否使用超链接。

#### `term.unicode`
* 类型：布尔值
* 默认值：自动检测
* 环境：`CARGO_TERM_UNICODE`

控制输出是否可以使用非 ASCII unicode 字符渲染。

#### `term.progress.when`
* 类型：字符串
* 默认值：`"auto"`
* 环境：`CARGO_TERM_PROGRESS_WHEN`

控制终端中是否显示进度条。可能的值：

* `auto`（默认）：智能猜测是否显示进度条。
* `always`：始终显示进度条。
* `never`：从不显示进度条。

#### `term.progress.width`
* 类型：整数
* 默认值：无
* 环境：`CARGO_TERM_PROGRESS_WIDTH`

设置进度条的宽度。

#### `term.progress.term-integration`
* 类型：布尔值
* 默认值：自动检测
* 环境：`CARGO_TERM_PROGRESS_TERM_INTEGRATION`

向终端模拟器报告进度，以便在任务栏等位置显示。

[`cargo bench`]: ../commands/cargo-bench.md
[`cargo login`]: ../commands/cargo-login.md
[`cargo logout`]: ../commands/cargo-logout.md
[`cargo doc`]: ../commands/cargo-doc.md
[`cargo new`]: ../commands/cargo-new.md
[`cargo publish`]: ../commands/cargo-publish.md
[`cargo run`]: ../commands/cargo-run.md
[`cargo rustc`]: ../commands/cargo-rustc.md
[`cargo test`]: ../commands/cargo-test.md
[`cargo rustdoc`]: ../commands/cargo-rustdoc.md
[`cargo install`]: ../commands/cargo-install.md
[env]: environment-variables.md
[`cfg()` expression]: ../../reference/conditional-compilation.html
[build scripts]: build-scripts.md
[`-C linker`]: ../../rustc/codegen-options/index.md#linker
[override a build script]: build-scripts.md#overriding-build-scripts
[toml]: https://toml.io/
[增量编译]: profiles.md#incremental
[带参数的程序路径]: #executable-paths-with-arguments
[libcurl 格式]: https://everything.curl.dev/transfers/conn/proxies#proxy-types
[Source Replacement]: source-replacement.md
[revision]: https://git-scm.com/docs/gitrevisions
[registries]: registries.md
[`cargo:token`]: registry-authentication.md#cargotoken
[crates.io]: https://crates.io/
[target triple]: ../appendix/glossary.md#target '"target" (glossary)'
[`<triple>`]: ../appendix/glossary.md#target '"target" (glossary)'