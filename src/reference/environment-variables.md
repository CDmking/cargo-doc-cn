# 环境变量 {#environment-variables}

Cargo 设置并读取许多环境变量，你的代码可以检测或覆盖这些变量。以下是 Cargo 设置的环境变量列表，按 Cargo 与它们交互的时间组织：

## Cargo 读取的环境变量 {#environment-variables-cargo-reads}

你可以覆盖这些环境变量来改变 Cargo 在你系统上的行为：

* `CARGO_LOG` --- Cargo 使用 [`tracing`] crate 来显示调试日志消息。`CARGO_LOG` 环境变量可以设置为启用调试日志，值如 `trace`、`debug` 或 `warn`。通常仅在调试期间使用。更多详细信息请参阅[调试日志]。
* `CARGO_HOME` --- Cargo 维护注册中心索引和 crate 的 git 检出本地缓存。默认情况下，这些存储在 `$HOME/.cargo`（Windows 上为 `%USERPROFILE%\.cargo`）下，但此变量覆盖此目录的位置。一旦 crate 被缓存，它不会被 clean 命令移除。更多详细信息请参阅[指南](../guide/cargo-home.md)。
* `CARGO_TARGET_DIR` --- 所有生成产物放置的位置，相对于当前工作目录。参见 [`build.target-dir`] 以通过配置设置。
* `CARGO` --- 如果设置，Cargo 在构建 crate 以及执行构建脚本和外部子命令时，将转发此值而不是将其设置为自己的自动检测路径。此值不由 Cargo 直接执行，应始终指向行为与 `cargo` 完全相同的命令，因为使用此变量的用户将期望如此。
* `RUSTC` --- Cargo 将执行此指定的编译器，而不是运行 `rustc`。参见 [`build.rustc`] 以通过配置设置。
* `RUSTC_WRAPPER` --- Cargo 将执行此指定的包装器，而不是简单地运行 `rustc`，将 rustc 调用作为其命令行参数传递，第一个参数是实际 rustc 的路径。用于设置构建缓存工具，如 `sccache`。参见 [`build.rustc-wrapper`] 以通过配置设置。将此设置为空字符串将覆盖配置并重置 cargo 不使用包装器。
* `RUSTC_WORKSPACE_WRAPPER` --- 对于工作空间成员，Cargo 将执行此指定的包装器，而不是简单地运行 `rustc`，将 rustc 调用作为其命令行参数传递，第一个参数是实际 rustc 的路径。当构建没有工作空间的单包项目时，该包被视为工作空间。它影响文件名哈希，以便包装器生成的产物被单独缓存。参见 [`build.rustc-workspace-wrapper`] 以通过配置设置。将此设置为空字符串将覆盖配置并重置 cargo 不对工作空间成员使用包装器。如果同时设置了 `RUSTC_WRAPPER` 和 `RUSTC_WORKSPACE_WRAPPER`，则它们将被嵌套：最终调用是 `$RUSTC_WRAPPER $RUSTC_WORKSPACE_WRAPPER $RUSTC`。
* `RUSTDOC` --- Cargo 将执行此指定的 `rustdoc` 实例，而不是运行 `rustdoc`。参见 [`build.rustdoc`] 以通过配置设置。
* `RUSTDOCFLAGS` --- 一个空格分隔的自定义标志列表，传递给 Cargo 执行的所有 `rustdoc` 调用。与 [`cargo rustdoc`] 相比，这对于将标志传递给*所有* `rustdoc` 实例很有用。参见 [`build.rustdocflags`] 了解设置标志的更多方法。此字符串由空格分割；对于多个参数的更健壮编码，请参见 `CARGO_ENCODED_RUSTDOCFLAGS`。
* `CARGO_ENCODED_RUSTDOCFLAGS` --- 一个自定义标志列表，由 `0x1f`（ASCII 单元分隔符）分隔，传递给 Cargo 执行的所有 `rustdoc` 调用。
* `RUSTFLAGS` --- 一个空格分隔的自定义标志列表，传递给 Cargo 执行的所有编译器调用。与 [`cargo rustc`] 相比，这对于将标志传递给*所有*编译器实例很有用。参见 [`build.rustflags`] 了解设置标志的更多方法。此字符串由空格分割；对于多个参数的更健壮编码，请参见 `CARGO_ENCODED_RUSTFLAGS`。
* `CARGO_ENCODED_RUSTFLAGS` --- 一个自定义标志列表，由 `0x1f`（ASCII 单元分隔符）分隔，传递给 Cargo 执行的所有编译器调用。
* `CARGO_INCREMENTAL` --- 如果此变量设置为 1，则 Cargo 将强制为当前编译启用[增量编译]；如果设置为 0，则强制禁用。如果此环境变量不存在，则将使用 Cargo 的默认值。另请参见 [`build.incremental`] 配置值。
* `CARGO_CACHE_RUSTC_INFO` --- 如果此变量设置为 0，则 Cargo 将不会尝试缓存编译器版本信息。
* `HTTPS_PROXY` 或 `https_proxy` 或 `http_proxy` --- 使用的 HTTP 代理，更多详细信息请参见 [`http.proxy`]。
* `HTTP_TIMEOUT` --- HTTP 超时时间（秒），更多详细信息请参见 [`http.timeout`]。
* `TERM` --- 如果此变量设置为 `dumb`，则禁用进度条。
* `BROWSER` --- 用于执行以打开文档的 Web 浏览器，使用 [`cargo doc`] 的 `--open` 标志，更多详细信息请参见 [`doc.browser`]。
* `RUSTFMT` --- [`cargo fmt`](https://github.com/rust-lang/rustfmt) 将执行此指定的 `rustfmt` 实例，而不是运行 `rustfmt`。

### 配置环境变量 {#configuration-environment-variables}

Cargo 为一些配置值读取环境变量。更多详细信息请参见[配置章节][config-env]。总结来说，支持的环境变量有：

* `CARGO_ALIAS_<name>` --- 命令别名，参见 [`alias`]。
* `CARGO_BUILD_JOBS` --- 并行作业数，参见 [`build.jobs`]。
* `CARGO_BUILD_RUSTC` --- `rustc` 可执行文件，参见 [`build.rustc`]。
* `CARGO_BUILD_RUSTC_WRAPPER` --- `rustc` 包装器，参见 [`build.rustc-wrapper`]。
* `CARGO_BUILD_RUSTC_WORKSPACE_WRAPPER` --- 仅用于工作空间成员的 `rustc` 包装器，参见 [`build.rustc-workspace-wrapper`]。
* `CARGO_BUILD_RUSTDOC` --- `rustdoc` 可执行文件，参见 [`build.rustdoc`]。
* `CARGO_BUILD_TARGET` --- 默认目标平台，参见 [`build.target`]。
* `CARGO_BUILD_TARGET_DIR` --- 默认输出目录，参见 [`build.target-dir`]。
* `CARGO_BUILD_BUILD_DIR` --- 默认构建目录，参见 [`build.build-dir`]。
* `CARGO_BUILD_RUSTFLAGS` --- 额外的 `rustc` 标志，参见 [`build.rustflags`]。
* `CARGO_BUILD_RUSTDOCFLAGS` --- 额外的 `rustdoc` 标志，参见 [`build.rustdocflags`]。
* `CARGO_BUILD_INCREMENTAL` --- 增量编译，参见 [`build.incremental`]。
* `CARGO_BUILD_DEP_INFO_BASEDIR` --- 依赖信息相对目录，参见 [`build.dep-info-basedir`]。
* `CARGO_CACHE_AUTO_CLEAN_FREQUENCY` --- 配置自动缓存清理运行的频率，参见 [`cache.auto-clean-frequency`]。
* `CARGO_CARGO_NEW_VCS` --- [`cargo new`] 的默认源代码控制系统，参见 [`cargo-new.vcs`]。
* `CARGO_FUTURE_INCOMPAT_REPORT_FREQUENCY` --- 我们应生成未来不兼容报告通知的频率，参见 [`future-incompat-report.frequency`]。
* `CARGO_HTTP_DEBUG` --- 启用 HTTP 调试，参见 [`http.debug`]。
* `CARGO_HTTP_PROXY` --- 启用 HTTP 代理，参见 [`http.proxy`]。
* `CARGO_HTTP_TIMEOUT` --- HTTP 超时时间，参见 [`http.timeout`]。
* `CARGO_HTTP_CAINFO` --- TLS 证书颁发机构文件，参见 [`http.cainfo`]。
* `CARGO_HTTP_PROXY_CAINFO` --- 代理 TLS 证书颁发机构文件，参见 [`http.proxy-cainfo`]。
* `CARGO_HTTP_CHECK_REVOKE` --- 禁用 TLS 证书吊销检查，参见 [`http.check-revoke`]。
* `CARGO_HTTP_SSL_VERSION` --- 使用的 TLS 版本，参见 [`http.ssl-version`]。
* `CARGO_HTTP_LOW_SPEED_LIMIT` --- HTTP 低速限制，参见 [`http.low-speed-limit`]。
* `CARGO_HTTP_MULTIPLEXING` --- 是否使用 HTTP/2 多路复用，参见 [`http.multiplexing`]。
* `CARGO_HTTP_USER_AGENT` --- HTTP 用户代理头，参见 [`http.user-agent`]。
* `CARGO_INSTALL_ROOT` --- [`cargo install`] 的默认目录，参见 [`install.root`]。
* `CARGO_NET_RETRY` --- 重试网络错误的次数，参见 [`net.retry`]。
* `CARGO_NET_GIT_FETCH_WITH_CLI` --- 启用使用 `git` 可执行文件进行获取，参见 [`net.git-fetch-with-cli`]。
* `CARGO_NET_OFFLINE` --- 离线模式，参见 [`net.offline`]。
* `CARGO_PROFILE_<name>_BUILD_OVERRIDE_<key>` --- 覆盖构建脚本配置文件，参见 [`profile.<name>.build-override`]。
* `CARGO_PROFILE_<name>_CODEGEN_UNITS` --- 设置代码生成单元，参见 [`profile.<name>.codegen-units`]。
* `CARGO_PROFILE_<name>_DEBUG` --- 包含的调试信息类型，参见 [`profile.<name>.debug`]。
* `CARGO_PROFILE_<name>_DEBUG_ASSERTIONS` --- 启用/禁用调试断言，参见 [`profile.<name>.debug-assertions`]。
* `CARGO_PROFILE_<name>_INCREMENTAL` --- 启用/禁用增量编译，参见 [`profile.<name>.incremental`]。
* `CARGO_PROFILE_<name>_LTO` --- 链接时优化，参见 [`profile.<name>.lto`]。
* `CARGO_PROFILE_<name>_OVERFLOW_CHECKS` --- 启用/禁用溢出检查，参见 [`profile.<name>.overflow-checks`]。
* `CARGO_PROFILE_<name>_OPT_LEVEL` --- 设置优化级别，参见 [`profile.<name>.opt-level`]。
* `CARGO_PROFILE_<name>_PANIC` --- 使用的 panic 策略，参见 [`profile.<name>.panic`]。
* `CARGO_PROFILE_<name>_RPATH` --- rpath 链接选项，参见 [`profile.<name>.rpath`]。
* `CARGO_PROFILE_<name>_SPLIT_DEBUGINFO` --- 控制调试文件输出行为，参见 [`profile.<name>.split-debuginfo`]。
* `CARGO_PROFILE_<name>_STRIP` --- 控制符号和/或调试信息的剥离，参见 [`profile.<name>.strip`]。
* `CARGO_REGISTRIES_<name>_CREDENTIAL_PROVIDER` --- 注册中心的凭证提供者，参见 [`registries.<name>.credential-provider`]。
* `CARGO_REGISTRIES_<name>_INDEX` --- 注册中心索引的 URL，参见 [`registries.<name>.index`]。
* `CARGO_REGISTRIES_<name>_TOKEN` --- 注册中心的认证令牌，参见 [`registries.<name>.token`]。
* `CARGO_REGISTRY_CREDENTIAL_PROVIDER` --- [crates.io] 的凭证提供者，参见 [`registry.credential-provider`]。
* `CARGO_REGISTRY_DEFAULT` --- `--registry` 标志的默认注册中心，参见 [`registry.default`]。
* `CARGO_REGISTRY_GLOBAL_CREDENTIAL_PROVIDERS` --- 用于未定义特定提供者的注册中心的凭证提供者。参见 [`registry.global-credential-providers`]。
* `CARGO_REGISTRY_TOKEN` --- [crates.io] 的认证令牌，参见 [`registry.token`]。
* `CARGO_TARGET_<triple>_LINKER` --- 使用的链接器，参见 [`target.<triple>.linker`]。三元组必须[转换为大写和下划线](config.md#environment-variables)。
* `CARGO_TARGET_<triple>_RUNNER` --- 可执行文件运行器，参见 [`target.<triple>.runner`]。
* `CARGO_TARGET_<triple>_RUSTFLAGS` --- 目标的额外 `rustc` 标志，参见 [`target.<triple>.rustflags`]。
* `CARGO_TERM_QUIET` --- 静默模式，参见 [`term.quiet`]。
* `CARGO_TERM_VERBOSE` --- 默认终端详细程度，参见 [`term.verbose`]。
* `CARGO_TERM_COLOR` --- 默认颜色模式，参见 [`term.color`]。
* `CARGO_TERM_PROGRESS_WHEN` --- 默认进度条显示模式，参见 [`term.progress.when`]。
* `CARGO_TERM_PROGRESS_WIDTH` --- 默认进度条宽度，参见 [`term.progress.width`]。

[`cargo doc`]: ../commands/cargo-doc.md
[`cargo install`]: ../commands/cargo-install.md
[`cargo new`]: ../commands/cargo-new.md
[`cargo rustc`]: ../commands/cargo-rustc.md
[`cargo rustdoc`]: ../commands/cargo-rustdoc.md
[config-env]: config.md#environment-variables
[crates.io]: https://crates.io/
[增量编译]: profiles.md#incremental
[`alias`]: config.md#alias
[`build.jobs`]: config.md#buildjobs
[`build.rustc`]: config.md#buildrustc
[`build.rustc-wrapper`]: config.md#buildrustc-wrapper
[`build.rustc-workspace-wrapper`]: config.md#buildrustc-workspace-wrapper
[`build.rustdoc`]: config.md#buildrustdoc
[`build.target`]: config.md#buildtarget
[`build.target-dir`]: config.md#buildtarget-dir
[`build.build-dir`]: config.md#buildbuild-dir
[`build.rustflags`]: config.md#buildrustflags
[`build.rustdocflags`]: config.md#buildrustdocflags
[`build.incremental`]: config.md#buildincremental
[`build.dep-info-basedir`]: config.md#builddep-info-basedir
[`doc.browser`]: config.md#docbrowser
[`cache.auto-clean-frequency`]: config.md#cacheauto-clean-frequency
[`cargo-new.name`]: config.md#cargo-newname
[`cargo-new.email`]: config.md#cargo-newemail
[`cargo-new.vcs`]: config.md#cargo-newvcs
[`future-incompat-report.frequency`]: config.md#future-incompat-reportfrequency
[`http.debug`]: config.md#httpdebug
[`http.proxy`]: config.md#httpproxy
[`http.timeout`]: config.md#httptimeout
[`http.cainfo`]: config.md#httpcainfo
[`http.proxy-cainfo`]: config.md#httpproxy-cainfo
[`http.check-revoke`]: config.md#httpcheck-revoke
[`http.ssl-version`]: config.md#httpssl-version
[`http.low-speed-limit`]: config.md#httplow-speed-limit
[`http.multiplexing`]: config.md#httpmultiplexing
[`http.user-agent`]: config.md#httpuser-agent
[`install.root`]: config.md#installroot
[`net.retry`]: config.md#netretry
[`net.git-fetch-with-cli`]: config.md#netgit-fetch-with-cli
[`net.offline`]: config.md#netoffline
[`profile.<name>.build-override`]: config.md#profilenamebuild-override
[`profile.<name>.codegen-units`]: config.md#profilenamecodegen-units
[`profile.<name>.debug`]: config.md#profilenamedebug
[`profile.<name>.debug-assertions`]: config.md#profilenamedebug-assertions
[`profile.<name>.incremental`]: config.md#profilenameincremental
[`profile.<name>.lto`]: config.md#profilenamelto
[`profile.<name>.overflow-checks`]: config.md#profilenameoverflow-checks
[`profile.<name>.opt-level`]: config.md#profilenameopt-level
[`profile.<name>.panic`]: config.md#profilenamepanic
[`profile.<name>.rpath`]: config.md#profilenamerpath
[`profile.<name>.split-debuginfo`]: config.md#profilenamesplit-debuginfo
[`profile.<name>.strip`]: config.md#profilenamestrip
[`registries.<name>.credential-provider`]: config.md#registriesnamecredential-provider
[`registries.<name>.index`]: config.md#registriesnameindex
[`registries.<name>.token`]: config.md#registriesnametoken
[`registry.credential-provider`]: config.md#registrycredential-provider
[`registry.default`]: config.md#registrydefault
[`registry.global-credential-providers`]: config.md#registryglobal-credential-providers
[`registry.token`]: config.md#registrytoken
[`target.<triple>.linker`]: config.md#targettriplelinker
[`target.<triple>.runner`]: config.md#targettriplerunner
[`target.<triple>.rustflags`]: config.md#targettriplerustflags
[`term.quiet`]: config.md#termquiet
[`term.verbose`]: config.md#termverbose
[`term.color`]: config.md#termcolor
[`term.progress.when`]: config.md#termprogresswhen
[`term.progress.width`]: config.md#termprogresswidth

## Cargo 为 crate 设置的环境变量 {#environment-variables-cargo-sets-for-crates}

Cargo 在编译你的 crate 时将这些环境变量暴露给你的 crate。注意，这同样适用于使用 `cargo run` 和 `cargo test` 运行二进制文件。要在 Rust 程序中获取这些变量的值，请这样做：

```rust,ignore
let version = env!("CARGO_PKG_VERSION");
```

现在 `version` 将包含 `CARGO_PKG_VERSION` 的值。

注意，如果清单中没有提供这些值之一，则相应的环境变量将设置为空字符串 `""`。

* `CARGO` --- 执行构建的 `cargo` 二进制文件的路径。
* `CARGO_MANIFEST_DIR` --- 包含你的包清单的目录。
* `CARGO_MANIFEST_PATH` --- 你的包清单的路径。
* `CARGO_PKG_VERSION` --- 你的包的完整版本。
* `CARGO_PKG_VERSION_MAJOR` --- 你的包的主版本。
* `CARGO_PKG_VERSION_MINOR` --- 你的包的次版本。
* `CARGO_PKG_VERSION_PATCH` --- 你的包的补丁版本。
* `CARGO_PKG_VERSION_PRE` --- 你的包的预发布版本。
* `CARGO_PKG_AUTHORS` --- 你的包清单中的作者列表，以冒号分隔。
* `CARGO_PKG_NAME` --- 你的包的名称。
* `CARGO_PKG_DESCRIPTION` --- 你的包清单中的描述。
* `CARGO_PKG_HOMEPAGE` --- 你的包清单中的主页。
* `CARGO_PKG_REPOSITORY` --- 你的包清单中的仓库。
* `CARGO_PKG_LICENSE` --- 你的包清单中的许可证。
* `CARGO_PKG_LICENSE_FILE` --- 你的包清单中的许可证文件。
* `CARGO_PKG_RUST_VERSION` --- 你的包清单中的 Rust 版本。注意，这是包支持的最小 Rust 版本，不是当前的 Rust 版本。
* `CARGO_PKG_README` --- 你的包的 README 文件路径。
* `CARGO_CRATE_NAME` --- 当前正在编译的 crate 的名称。它是 [Cargo 目标][Cargo target] 的名称，其中 `-` 转换为 `_`，例如库、二进制文件、示例、集成测试或基准测试的名称。
* `CARGO_BIN_NAME` --- 当前正在编译的二进制文件的名称。仅针对[二进制文件][binaries]或二进制[示例][examples]设置。此名称不包括任何文件扩展名，例如 `.exe`。
* `OUT_DIR` --- 如果包有构建脚本，则此变量设置为构建脚本应放置其输出的文件夹。更多信息请参见下文。（仅在编译期间设置。）
* `CARGO_BIN_EXE_<name>` --- 二进制目标可执行文件的绝对路径。仅在构建[集成测试][integration test]或基准测试时设置。这可以与 [`env` 宏][`env` macro] 一起使用，以找到用于测试目的的可执行文件。`<name>` 是二进制目标的名称，完全按原样。例如，对于名为 `my-program` 的二进制文件，为 `CARGO_BIN_EXE_my-program`。除非二进制文件具有未启用的必需特性，否则在构建测试时会自动构建二进制文件。
* `CARGO_PRIMARY_PACKAGE` --- 如果正在构建的包是主要包，则将设置此环境变量。主要包是用户在命令行上选择的包，无论是通过 `-p` 标志还是基于当前目录和默认工作空间成员的默认值。构建依赖项时不会设置此变量，除非依赖项也是命令行上选择的工作空间成员。仅在编译包时设置（运行二进制文件或测试时不设置）。
* `CARGO_TARGET_TMPDIR` --- 仅在构建[集成测试][integration test]或基准测试代码时设置。这是目标目录内的一个目录路径，集成测试或基准测试可以自由放置测试/基准所需的任何数据。Cargo 最初创建此目录，但不以任何方式管理其内容，这是测试代码的责任。

[Cargo target]: cargo-targets.md
[binaries]: cargo-targets.md#binaries
[examples]: cargo-targets.md#examples
[integration test]: cargo-targets.md#integration-tests
[`env` macro]: ../../std/macro.env.html

### 动态库路径 {#dynamic-library-paths}

Cargo 在编译和运行二进制文件时（例如使用 `cargo run` 和 `cargo test` 命令）也会设置动态库路径。这有助于定位属于构建过程的共享库。变量名称取决于平台：

* Windows：`PATH`
* macOS：`DYLD_FALLBACK_LIBRARY_PATH`
* Unix：`LD_LIBRARY_PATH`
* AIX：`LIBPATH`

当 Cargo 启动时，该值从现有值扩展。macOS 有特殊考虑，如果 `DYLD_FALLBACK_LIBRARY_PATH` 尚未设置，它将添加默认的 `$HOME/lib:/usr/local/lib:/usr/lib`。

Cargo 包括以下路径：

* 来自任何构建脚本的搜索路径，使用 [`rustc-link-search` 指令][`rustc-link-search` instruction]。`target` 目录之外的路径被移除。运行 Cargo 的用户有责任正确设置环境，如果系统上的额外库需要在搜索路径中。
* 基本输出目录，例如 `target/debug`，以及 "deps" 目录。这主要用于支持过程宏。
* rustc sysroot 库路径。这通常对大多数用户不重要。

## Cargo 为构建脚本设置的环境变量 {#environment-variables-cargo-sets-for-build-scripts}

Cargo 在运行构建脚本时设置多个环境变量。由于这些变量在构建脚本编译时尚未设置，因此上述使用 `env!` 的示例将不起作用，而是需要在构建脚本运行时检索值：

```rust,ignore
use std::env;
let out_dir = env::var("OUT_DIR").unwrap();
```

现在 `out_dir` 将包含 `OUT_DIR` 的值。

* `CARGO` --- 执行构建的 `cargo` 二进制文件的路径。
* `CARGO_MANIFEST_DIR` --- 包含正在构建的包（包含构建脚本的包）清单的目录。另请注意，这是构建脚本启动时当前工作目录的值。
* `CARGO_MANIFEST_PATH` --- 你的包清单的路径。
* `CARGO_MANIFEST_LINKS` --- 清单的 `links` 值。
* `CARGO_MAKEFLAGS` --- 包含 Cargo 的 [jobserver] 实现并行化子进程所需的参数。来自 build.rs 的 Rustc 或 cargo 调用已经可以读取 `CARGO_MAKEFLAGS`，但 GNU Make 要求将标志直接指定为参数，或通过 `MAKEFLAGS` 环境变量。目前 Cargo 不设置 `MAKEFLAGS` 变量，但构建脚本调用 GNU Make 可以自由地将其设置为 `CARGO_MAKEFLAGS` 的内容。
* `CARGO_FEATURE_<name>` --- 对于正在构建的包的每个激活特性，将存在此环境变量，其中 `<name>` 是特性的名称，大写并将 `-` 转换为 `_`。
* `CARGO_CFG_<cfg>` --- 对于正在构建的包的每个[配置选项][configuration]，此环境变量将包含配置的值，其中 `<cfg>` 是配置的名称，大写并将 `-` 转换为 `_`。布尔配置如果设置则存在，否则不存在。具有多个值的配置将连接到一个变量，值由 `,` 分隔。这包括编译器内置的值（可以通过 `rustc --print=cfg` 查看）以及构建脚本设置的值和传递给 `rustc` 的额外标志（例如在 `RUSTFLAGS` 中定义的）。这些变量的一些示例：
    * `CARGO_CFG_FEATURE` --- 正在构建的包的每个激活特性。
    * `CARGO_CFG_UNIX` --- 在[类 Unix 平台]上设置。
    * `CARGO_CFG_WINDOWS` --- 在[类 Windows 平台]上设置。
    * `CARGO_CFG_TARGET_FAMILY=unix,wasm` --- [目标系列]。
    * `CARGO_CFG_TARGET_OS=macos` --- [目标操作系统]。
    * `CARGO_CFG_TARGET_ARCH=x86_64` --- CPU [目标架构]。
    * `CARGO_CFG_TARGET_VENDOR=apple` --- [目标供应商]。
    * `CARGO_CFG_TARGET_ENV=gnu` --- [目标环境] ABI。
    * `CARGO_CFG_TARGET_ABI=eabihf` --- [目标 ABI]。
    * `CARGO_CFG_TARGET_POINTER_WIDTH=64` --- CPU [指针宽度]。
    * `CARGO_CFG_TARGET_ENDIAN=little` --- CPU [目标字节序]。
    * `CARGO_CFG_TARGET_FEATURE=mmx,sse` --- 启用的 CPU [目标特性] 列表。
  > 注意，不同的[目标三元组][Target Triple]有不同的 `cfg` 值集合，因此一个目标三元组中存在的变量可能在另一个中不可用。
  >
  > 一些 cfg 值，如 `test`，不可用。
* `OUT_DIR` --- 所有输出和中间产物应放置的文件夹。此文件夹位于正在构建的包的构建目录内，并且对于该包是唯一的。
* `TARGET` --- 正在编译的目标三元组。本机代码应为此三元组编译。更多信息请参见[目标三元组][Target Triple]描述。
* `HOST` --- Rust 编译器的主机三元组。
* `NUM_JOBS` --- 指定为顶级并行度的并行度。这对于向 `make` 等系统传递 `-j` 参数可能有用。请注意，解释此环境变量时应小心。由于历史原因，仍然提供此变量，但例如，Cargo 的最近版本不需要运行 `make -j`，而是可以将 `MAKEFLAGS` 环境变量设置为 `CARGO_MAKEFLAGS` 的内容，以激活使用 Cargo 的 GNU Make 兼容 [jobserver] 进行子 make 调用。
* `OPT_LEVEL`、`DEBUG` --- 当前正在构建的配置文件的相应变量的值。
* `PROFILE` --- 发布构建为 `release`，其他构建为 `debug`。这是根据[配置文件][profile]是否继承自 [`dev`] 或 [`release`] 配置文件确定的。不建议使用此环境变量。使用其他环境变量，如 `OPT_LEVEL`，可以提供正在使用的实际设置的更正确视图。
* `DEP_<links>_<key>` --- 有关此组环境变量的更多信息，请参见关于 [`links`][links] 的构建脚本文档。
* `RUSTC`、`RUSTDOC` --- Cargo 已决定使用的编译器和文档生成器，传递给构建脚本，以便它也可以使用它们。
* `RUSTC_WRAPPER` --- Cargo 正在使用的 `rustc` 包装器（如果有）。参见 [`build.rustc-wrapper`]。
* `RUSTC_WORKSPACE_WRAPPER` --- Cargo 为工作空间成员使用的 `rustc` 包装器（如果有）。参见 [`build.rustc-workspace-wrapper`]。
* `RUSTC_LINKER` --- Cargo 已决定用于当前目标的链接器二进制文件的路径（如果指定）。可以通过编辑 `.cargo/config.toml` 来更改链接器；更多信息请参见关于 [cargo 配置][cargo-config] 的文档。
* `CARGO_ENCODED_RUSTFLAGS` --- Cargo 调用 `rustc` 时使用的额外标志，由 `0x1f` 字符（ASCII 单元分隔符）分隔。参见 [`build.rustflags`]。注意，自 Rust 1.55 起，`RUSTFLAGS` 已从环境中移除；脚本应改用 `CARGO_ENCODED_RUSTFLAGS`。
* `CARGO_PKG_<var>` --- 包信息变量，与[构建 crate 期间提供的][variables set for crates] 名称和值相同。

[`tracing`]: https://docs.rs/tracing
[调试日志]: https://doc.crates.io/contrib/implementation/debugging.html#logging
[类 Unix 平台]: ../../reference/conditional-compilation.html#unix-and-windows
[类 Windows 平台]: ../../reference/conditional-compilation.html#unix-and-windows
[目标系列]: ../../reference/conditional-compilation.html#target_family
[目标操作系统]: ../../reference/conditional-compilation.html#target_os
[目标架构]: ../../reference/conditional-compilation.html#target_arch
[目标供应商]: ../../reference/conditional-compilation.html#target_vendor
[目标环境]: ../../reference/conditional-compilation.html#target_env
[目标 ABI]: ../../reference/conditional-compilation.html#target_abi
[指针宽度]: ../../reference/conditional-compilation.html#target_pointer_width
[目标字节序]: ../../reference/conditional-compilation.html#target_endian
[目标特性]: ../../reference/conditional-compilation.html#target_feature
[links]: build-scripts.md#the-links-manifest-key
[configuration]: ../../reference/conditional-compilation.html
[jobserver]: https://www.gnu.org/software/make/manual/html_node/Job-Slots.html
[cargo-config]: config.md
[Target Triple]: ../appendix/glossary.md#target
[variables set for crates]: #environment-variables-cargo-sets-for-crates
[profile]: profiles.md
[`dev`]: profiles.md#dev
[`release`]: profiles.md#release
[`rustc-link-search` instruction]: build-scripts.md#rustc-link-search

## Cargo 为第三方子命令设置的环境变量 {#environment-variables-cargo-sets-for-3rd-party-subcommands}

Cargo 将此环境变量暴露给第三方子命令（即放置在 `$PATH` 中名为 `cargo-foobar` 的程序）：

* `CARGO` --- 执行构建的 `cargo` 二进制文件的路径。
* `CARGO_MAKEFLAGS` --- 包含 Cargo 的 [jobserver] 实现并行化子进程所需的参数。仅当 Cargo 检测到 jobserver 存在时才设置。

有关环境的扩展信息，你可以运行 `cargo metadata`。