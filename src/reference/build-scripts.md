# 构建脚本 {#build-scripts}

有些包需要编译第三方的非 Rust 代码，例如 C 库。其他包可能需要链接到 C 库，这些库可能位于系统上，或者可能需要从源代码构建。还有一些包需要诸如构建前代码生成（例如解析器生成器）等功能。

Cargo 的目标不是替换那些针对这些任务优化良好的其他工具，但它确实通过自定义构建脚本与它们集成。在包的根目录中放置一个名为 `build.rs` 的文件将导致 Cargo 编译该脚本并在构建包之前执行它。

```rust,ignore
// 自定义构建脚本示例。
fn main() {
    // 告诉 Cargo，如果给定的文件发生变化，则重新运行此构建脚本。
    println!("cargo::rerun-if-changed=src/hello.c");
    // 使用 `cc` crate 构建一个 C 文件并静态链接它。
    cc::Build::new()
        .file("src/hello.c")
        .compile("hello");
}
```

构建脚本的一些示例用例包括：

* 构建捆绑的 C 库。
* 在主机系统上查找 C 库。
* 根据规范生成 Rust 模块。
* 执行 crate 所需的任何平台特定配置。

以下部分描述了构建脚本的工作原理，[示例章节](build-script-examples.md)展示了如何编写脚本的各种示例。

> 注意：可以使用 [`package.build` 清单键](manifest.md#the-build-field) 来更改构建脚本的名称，或完全禁用它。

## 构建脚本的生命周期 {#life-cycle-of-a-build-script}

在构建包之前，Cargo 会将构建脚本编译为可执行文件（如果尚未构建）。然后它将运行该脚本，该脚本可以执行任意数量的任务。脚本可以通过向 stdout 打印以 `cargo::` 为前缀的特殊格式命令来与 Cargo 通信。

如果构建脚本的任何源文件或依赖项发生变化，它将重新构建。

默认情况下，如果包中的任何文件发生变化，Cargo 将重新运行构建脚本。通常，最好使用下面[变更检测](#change-detection)部分描述的 `rerun-if` 命令来缩小触发构建脚本重新运行的范围。

一旦构建脚本成功执行完毕，包的其余部分将被编译。如果出现错误，脚本应以非零退出代码退出以停止构建，此时构建脚本的输出将显示在终端上。

## 构建脚本的输入 {#inputs-to-the-build-script}

当构建脚本运行时，有许多输入传递给构建脚本，所有输入都以[环境变量][build-env]的形式传递。

除了环境变量之外，构建脚本的当前目录是构建脚本包的源目录。

[build-env]: environment-variables.md#environment-variables-cargo-sets-for-build-scripts

## 构建脚本的输出 {#outputs-of-the-build-script}

构建脚本可以将任何输出文件或中间产物保存在 [`OUT_DIR` 环境变量][build-env]指定的目录中。脚本不应修改该目录之外的任何文件。

构建脚本通过打印到 stdout 与 Cargo 通信。Cargo 将把每一行以 `cargo::` 开头的行解释为将影响包编译的指令。所有其他行将被忽略。

> 构建脚本打印的 `cargo::` 指令的*顺序*可能会影响 `cargo` 传递给 `rustc` 的参数顺序。反过来，传递给 `rustc` 的参数顺序可能会影响传递给链接器的参数顺序。因此，你需要注意构建脚本指令的顺序。例如，如果对象 `foo` 需要链接到库 `bar`，你可能希望确保库 `bar` 的 [`cargo::rustc-link-lib`](#rustc-link-lib) 指令出现在链接对象 `foo` 的指令*之后*。

在正常编译期间，脚本的输出对终端是隐藏的。如果你希望直接在终端中看到输出，请使用 `-vv` 标志以“非常详细”的方式调用 Cargo。这仅在构建脚本运行时发生。如果 Cargo 确定没有任何变化，它将不会重新运行脚本，更多信息请参见下面的[变更检测](#change-detection)。

构建脚本打印到 stdout 的所有行都写入到类似 `target/debug/build/<pkg>/output` 的文件中（确切位置可能取决于你的配置）。stderr 输出也保存在同一目录中。

以下是 Cargo 识别的指令摘要，每个指令的详细信息如下。

* [`cargo::rerun-if-changed=PATH`](#rerun-if-changed) --- 告诉 Cargo 何时重新运行脚本。
* [`cargo::rerun-if-env-changed=VAR`](#rerun-if-env-changed) --- 告诉 Cargo 何时重新运行脚本。
* [`cargo::rustc-link-arg=FLAG`](#rustc-link-arg) --- 为基准测试、二进制文件、`cdylib` crate、示例和测试向链接器传递自定义标志。
* [`cargo::rustc-link-arg-cdylib=FLAG`](#rustc-cdylib-link-arg) --- 为 cdylib crate 向链接器传递自定义标志。
* [`cargo::rustc-link-arg-bin=BIN=FLAG`](#rustc-link-arg-bin) --- 为二进制文件 `BIN` 向链接器传递自定义标志。
* [`cargo::rustc-link-arg-bins=FLAG`](#rustc-link-arg-bins) --- 为二进制文件向链接器传递自定义标志。
* [`cargo::rustc-link-arg-tests=FLAG`](#rustc-link-arg-tests) --- 为测试向链接器传递自定义标志。
* [`cargo::rustc-link-arg-examples=FLAG`](#rustc-link-arg-examples) --- 为示例向链接器传递自定义标志。
* [`cargo::rustc-link-arg-benches=FLAG`](#rustc-link-arg-benches) --- 为基准测试向链接器传递自定义标志。
* [`cargo::rustc-link-lib=LIB`](#rustc-link-lib) --- 添加要链接的库。
* [`cargo::rustc-link-search=[KIND=]PATH`](#rustc-link-search) --- 添加到库搜索路径。
* [`cargo::rustc-flags=FLAGS`](#rustc-flags) --- 向编译器传递某些标志。
* [`cargo::rustc-cfg=KEY[="VALUE"]`](#rustc-cfg) --- 启用编译时 `cfg` 设置。
* [`cargo::rustc-check-cfg=CHECK_CFG`](#rustc-check-cfg) --- 注册自定义 `cfg` 作为预期的配置，用于编译时配置检查。
* [`cargo::rustc-env=VAR=VALUE`](#rustc-env) --- 设置环境变量。
* [`cargo::error=MESSAGE`](#cargo-error) --- 在终端上显示错误。
* [`cargo::warning=MESSAGE`](#cargo-warning) --- 在终端上显示警告。
* [`cargo::metadata=KEY=VALUE`](#the-links-manifest-key) --- 元数据，由 `links` 脚本使用。

> **MSRV：** 需要 1.77 版本才能使用 `cargo::KEY=VALUE` 语法。
> 要支持旧版本，请使用 `cargo:KEY=VALUE` 语法。

### `cargo::rustc-link-arg=FLAG` {#rustc-link-arg}

`rustc-link-arg` 指令告诉 Cargo 将 [`-C link-arg=FLAG` 选项][link-arg]传递给编译器，但仅在构建受支持的目标（基准测试、二进制文件、`cdylib` crate、示例和测试）时使用。它的使用高度依赖于平台。它对于设置共享库版本或链接器脚本很有用。

[link-arg]: ../../rustc/codegen-options/index.md#link-arg

### `cargo::rustc-link-arg-cdylib=FLAG` {#rustc-cdylib-link-arg}

`rustc-link-arg-cdylib` 指令告诉 Cargo 将 [`-C link-arg=FLAG` 选项][link-arg]传递给编译器，但仅在构建 `cdylib` 库目标时使用。它的使用高度依赖于平台。它对于设置共享库版本或运行时路径很有用。

由于历史原因，`cargo::rustc-cdylib-link-arg` 形式是 `cargo::rustc-link-arg-cdylib` 的别名，并且具有相同的含义。

### `cargo::rustc-link-arg-bin=BIN=FLAG` {#rustc-link-arg-bin}

`rustc-link-arg-bin` 指令告诉 Cargo 将 [`-C link-arg=FLAG` 选项][link-arg]传递给编译器，但仅在构建名为 `BIN` 的二进制目标时使用。它的使用高度依赖于平台。它对于设置链接器脚本或其他链接器选项很有用。

### `cargo::rustc-link-arg-bins=FLAG` {#rustc-link-arg-bins}

`rustc-link-arg-bins` 指令告诉 Cargo 将 [`-C link-arg=FLAG` 选项][link-arg]传递给编译器，但仅在构建二进制目标时使用。它的使用高度依赖于平台。它对于设置链接器脚本或其他链接器选项很有用。

### `cargo::rustc-link-arg-tests=FLAG` {#rustc-link-arg-tests}

`rustc-link-arg-tests` 指令告诉 Cargo 将 [`-C link-arg=FLAG` 选项][link-arg]传递给编译器，但仅在构建测试目标时使用。

### `cargo::rustc-link-arg-examples=FLAG` {#rustc-link-arg-examples}

`rustc-link-arg-examples` 指令告诉 Cargo 将 [`-C link-arg=FLAG` 选项][link-arg]传递给编译器，但仅在构建示例目标时使用。

### `cargo::rustc-link-arg-benches=FLAG` {#rustc-link-arg-benches}

`rustc-link-arg-benches` 指令告诉 Cargo 将 [`-C link-arg=FLAG` 选项][link-arg]传递给编译器，但仅在构建基准测试目标时使用。

### `cargo::rustc-link-lib=LIB` {#rustc-link-lib}

`rustc-link-lib` 指令告诉 Cargo 使用编译器的 [`-l` 标志][option-link]链接给定的库。这通常用于通过 [FFI] 链接本机库。

`LIB` 字符串直接传递给 rustc，因此它支持 `-l` 的任何语法。\
目前 `LIB` 完全支持的语法是 `[KIND[:MODIFIERS]=]NAME[:RENAME]`。

`-l` 标志仅传递给包的库目标，除非没有库目标，在这种情况下它将传递给所有目标。这样做是因为所有其他目标都隐式依赖于库目标，并且要链接的给定库应该只包含一次。这意味着，如果一个包同时具有库和二进制目标，则*库*可以访问给定库中的符号，而二进制文件应通过库目标的公共 API 访问它们。

可选的 `KIND` 可以是 `dylib`、`static` 或 `framework` 之一。更多详细信息请参阅 [rustc 手册][option-link]。

[option-link]: ../../rustc/command-line-arguments.md#option-l-link-lib
[FFI]: ../../nomicon/ffi.md

### `cargo::rustc-link-search=[KIND=]PATH` {#rustc-link-search}

`rustc-link-search` 指令告诉 Cargo 将 [`-L` 标志][option-search]传递给编译器，以将目录添加到库搜索路径。

可选的 `KIND` 可以是 `dependency`、`crate`、`native`、`framework` 或 `all` 之一。更多详细信息请参阅 [rustc 手册][option-search]。

如果这些路径在 `OUT_DIR` 内，它们也会被添加到[动态库搜索路径环境变量](environment-variables.md#dynamic-library-paths)中。不建议依赖此行为，因为这使得使用生成的二进制文件变得困难。通常，最好避免在构建脚本中创建动态库（使用现有的系统库是可以的）。

[option-search]: ../../rustc/command-line-arguments.md#option-l-search-path

### `cargo::rustc-flags=FLAGS` {#rustc-flags}

`rustc-flags` 指令告诉 Cargo 将给定的空格分隔的标志传递给编译器。这只允许 `-l` 和 `-L` 标志，相当于使用 [`rustc-link-lib`](#rustc-link-lib) 和 [`rustc-link-search`](#rustc-link-search)。

### `cargo::rustc-cfg=KEY[="VALUE"]` {#rustc-cfg}

`rustc-cfg` 指令告诉 Cargo 将给定值传递给编译器的 [`--cfg` 标志][option-cfg]。这可以用于在编译时检测特性以启用[条件编译]。自定义 cfg 必须使用 [`cargo::rustc-check-cfg`](#rustc-check-cfg) 指令进行预期，否则使用将需要允许 [`unexpected_cfgs`][unexpected-cfgs] lint 以避免意外的 cfg 警告。

请注意，这*不会*影响 Cargo 的依赖项解析。这不能用于启用可选依赖项或启用其他 Cargo 特性。

请注意，[Cargo 特性]使用 `feature="foo"` 形式。使用此标志传递的 `cfg` 值不限于该形式，可以提供单个标识符或任何任意的键/值对。例如，发出 `cargo::rustc-cfg=abc` 将允许代码使用 `#[cfg(abc)]`（注意缺少 `feature=`）。或者，可以使用 `=` 符号的任意键/值对，如 `cargo::rustc-cfg=my_component="foo"`。键应为 Rust 标识符，值应为字符串。

[Cargo 特性]: features.md
[条件编译]: ../../reference/conditional-compilation.md
[option-cfg]: ../../rustc/command-line-arguments.md#option-cfg
[unexpected-cfgs]: ../../rustc/lints/listing/warn-by-default.md#unexpected-cfgs

### `cargo::rustc-check-cfg=CHECK_CFG` {#rustc-check-cfg}

添加到预期的配置名称和值列表中，该列表用于检查*可到达的* cfg 表达式时使用 [`unexpected_cfgs`][unexpected-cfgs] lint。

`CHECK_CFG` 的语法镜像了 `rustc` 的 [`--check-cfg` 标志][option-check-cfg]，更多详细信息请参阅[检查条件配置][checking-conditional-configurations]。

该指令可以这样使用：

```rust,no_run
// build.rs
println!("cargo::rustc-check-cfg=cfg(foo, values(\"bar\"))");
if foo_bar_condition {
    println!("cargo::rustc-cfg=foo=\"bar\"");
}
```

请注意，应定义所有可能的 cfg，无论当前启用了哪些 cfg。这包括给定 cfg 名称的所有可能值。

建议将 `cargo::rustc-check-cfg` 和 [`cargo::rustc-cfg`][option-cfg] 指令尽可能紧密地分组，以避免拼写错误、缺少 check-cfg、过时的 cfg...

另请参阅[条件编译][conditional-compilation-example]示例。

> **MSRV：** 从 1.80 版本开始被尊重

[checking-conditional-configurations]: ../../rustc/check-cfg.html
[option-check-cfg]: ../../rustc/command-line-arguments.md#option-check-cfg
[conditional-compilation-example]: build-script-examples.md#conditional-compilation

### `cargo::rustc-env=VAR=VALUE` {#rustc-env}

`rustc-env` 指令告诉 Cargo 在编译包时设置给定的环境变量。然后，编译后的 crate 中的 [`env!` 宏][env-macro]可以检索该值。这对于在 crate 的代码中嵌入额外的元数据很有用，例如 git HEAD 的哈希值或持续集成服务器的唯一标识符。

另请参阅[Cargo 自动包含的环境变量][env-cargo]。

> **注意**：这些环境变量在运行可执行文件时也会设置，使用 `cargo run` 或 `cargo test`。但是，不鼓励这种用法，因为它将可执行文件与 Cargo 的执行环境绑定。通常，这些环境变量应仅在编译时使用 `env!` 宏检查。

[env-macro]: ../../std/macro.env.html
[env-cargo]: environment-variables.md#environment-variables-cargo-sets-for-crates

### `cargo::error=MESSAGE` {#cargo-error}

`error` 指令告诉 Cargo 在构建脚本运行完毕后显示错误，然后使构建失败。

> 注意：构建脚本库应仔细考虑是否要使用 `cargo::error` 与返回 `Result`。最好返回 `Result`，并允许调用者决定错误是否致命。然后调用者可以决定是否使用 `cargo::error` 显示 `Err` 变体。

> **MSRV：** 从 1.84 版本开始被尊重

### `cargo::warning=MESSAGE` {#cargo-warning}

`warning` 指令告诉 Cargo 在构建脚本运行完毕后显示警告。警告仅针对 `path` 依赖项（即你在本地工作的那些）显示，因此例如，默认情况下不会发出 [crates.io] crate 中打印的警告，除非构建失败。可以使用 `-vv` “非常详细”标志让 Cargo 显示所有 crate 的警告。

## 构建依赖项 {#build-dependencies}

构建脚本也可以依赖于其他基于 Cargo 的 crate。依赖项通过清单的 `build-dependencies` 部分声明。

```toml
[build-dependencies]
cc = "1.0.46"
```

构建脚本**不能**访问列在 `dependencies` 或 `dev-dependencies` 部分中的依赖项（它们尚未构建！）。此外，构建依赖项对包本身不可用，除非也显式添加到 `[dependencies]` 表中。

建议仔细考虑添加的每个依赖项，权衡对编译时间、许可、维护等的影响。Cargo 将尝试重用构建依赖项和普通依赖项之间共享的依赖项。然而，这并不总是可能的，例如在交叉编译时，因此请考虑对编译时间的影响。

## 变更检测 {#change-detection}

当重新构建包时，Cargo 不一定知道是否需要再次运行构建脚本。默认情况下，它采取保守的方法，即如果包中的任何文件发生更改（或由 [`exclude` 和 `include` 字段]控制的文件列表），则始终重新运行构建脚本。对于大多数情况，这不是一个好的选择，因此建议每个构建脚本至少发出一个 `rerun-if` 指令（如下所述）。如果发出这些指令，则 Cargo 仅在给定值发生更改时才会重新运行脚本。如果 Cargo 正在重新运行你自己的 crate 或依赖项的构建脚本，而你不知道为什么，请参阅 FAQ 中的["为什么 Cargo 在重新构建我的代码？"](../faq.md#why-is-cargo-rebuilding-my-code)。

[`exclude` 和 `include` 字段]: manifest.md#the-exclude-and-include-fields

### `cargo::rerun-if-changed=PATH` {#rerun-if-changed}

`rerun-if-changed` 指令告诉 Cargo，如果给定路径的文件发生更改，则重新运行脚本。目前，Cargo 仅使用文件系统最后修改的 "mtime" 时间戳来确定文件是否已更改。它会与构建脚本上次运行时的内部缓存时间戳进行比较。

如果路径指向一个目录，它将扫描整个目录以查找任何修改。

如果构建脚本在任何情况下都不需要重新运行，那么发出 `cargo::rerun-if-changed=build.rs` 是防止其重新运行的一种简单方法（否则，如果没有发出 `rerun-if` 指令，默认是扫描整个包目录以查找更改）。Cargo 会自动处理脚本本身是否需要重新编译，当然，脚本在重新编译后将被重新运行。否则，指定 `build.rs` 是冗余且不必要的。

### `cargo::rerun-if-env-changed=NAME` {#rerun-if-env-changed}

`rerun-if-env-changed` 指令告诉 Cargo，如果给定名称的环境变量的值发生更改，则重新运行构建脚本。

请注意，这里的环境变量旨在用于全局环境变量，如 `CC` 等，不能用于像 `TARGET` 这样的环境变量，这些变量是[Cargo 为构建脚本设置的][build-env]。使用的环境变量是 `cargo` 调用接收到的环境变量，而不是构建脚本可执行文件接收到的环境变量。

从 1.46 版本开始，在源代码中使用 [`env!`][env-macro] 和 [`option_env!`][option-env-macro] 将自动检测更改并触发重新构建。对于这些宏已经引用的变量，不再需要 `rerun-if-env-changed`。

[option-env-macro]: ../../std/macro.option_env.html

## `links` 清单键 {#the-links-manifest-key}

可以在 `Cargo.toml` 清单中设置 `package.links` 键，以声明包链接到给定的本机库。此清单键的目的是让 Cargo 了解包具有的本机依赖项集，并提供在包构建脚本之间传递元数据的原则性系统。

```toml
[package]
# ...
links = "foo"
```

此清单声明包链接到 `libfoo` 本机库。使用 `links` 键时，包必须具有构建脚本，并且构建脚本应使用 [`rustc-link-lib` 指令](#rustc-link-lib)来链接库。

首先，Cargo 要求每个 `links` 值最多有一个包。换句话说，禁止有两个包链接到同一个本机库。这有助于防止 crate 之间的重复符号。但是，请注意，有一些[约定](#-sys-packages)可以缓解这种情况。

构建脚本可以生成任意键值对形式的元数据集。此元数据使用 `cargo::metadata=KEY=VALUE` 指令设置。

元数据传递给**依赖**包的构建脚本。例如，如果包 `foo` 依赖于 `bar`，而 `bar` 链接到 `baz`，那么如果 `bar` 生成 `key=value` 作为其构建脚本元数据的一部分，则 `foo` 的构建脚本将具有环境变量 `DEP_BAZ_KEY=value`（注意 `links` 键的值被使用，并且 `key` 的大小写发生变化）。请参阅["使用另一个 `sys` crate"][using-another-sys]以了解如何使用此功能的示例。

请注意，元数据仅传递给直接依赖项，而不是传递依赖项。

> **MSRV：** 需要 1.77 版本才能使用 `cargo::metadata=KEY=VALUE`。
> 要支持旧版本，请使用 `cargo:KEY=VALUE`（不受支持的指令被假定为元数据键）。

[using-another-sys]: build-script-examples.md#using-another-sys-crate

## `*-sys` 包 {#-sys-packages}

一些链接到系统库的 Cargo 包有一个命名约定，即带有 `-sys` 后缀。任何名为 `foo-sys` 的包应提供两个主要功能：

* 库 crate 应链接到本机库 `libfoo`。这通常会在从源代码构建之前探测当前系统以查找 `libfoo`。
* 库 crate 应提供 `libfoo` 中的类型和函数的**声明**，但**不**提供更高级别的抽象。

`*-sys` 包集提供了一组用于链接到本机库的公共依赖项。从这种本机库相关包的约定中获得了一些好处：

* 对 `foo-sys` 的公共依赖项缓解了每个 `links` 值一个包的规则。
* 其他 `-sys` 包可以利用 `DEP_LINKS_KEY=value` 环境变量更好地与其他包集成。请参阅["使用另一个 `sys` crate"][using-another-sys]示例。
* 公共依赖项允许集中发现 `libfoo` 本身（或从源代码构建）的逻辑。
* 这些依赖项很容易[被覆盖](#overriding-build-scripts)。

通常有一个没有 `-sys` 后缀的配套包，它在 sys 包之上提供安全、高级别的抽象。例如，[`git2` crate] 提供了对 [`libgit2-sys` crate] 的高级接口。

[`git2` crate]: https://crates.io/crates/git2
[`libgit2-sys` crate]: https://crates.io/crates/libgit2-sys

## 覆盖构建脚本 {#overriding-build-scripts}

如果清单包含 `links` 键，则 Cargo 支持使用自定义库覆盖指定的构建脚本。此功能的目的是完全防止运行所讨论的构建脚本，而是提前提供元数据。

要覆盖构建脚本，请在任何可接受的 [`config.toml`](config.md) 文件中放置以下配置。

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

使用此配置，如果包声明它链接到 `foo`，则构建脚本将**不会**被编译或运行，而是使用指定的元数据。

不应使用 `warning`、`rerun-if-changed` 和 `rerun-if-env-changed` 键，它们将被忽略。

## Jobserver {#jobserver}

Cargo 和 `rustc` 使用为 GNU make 开发的 [jobserver 协议][jobserver protocol]来协调进程间的并发性。它本质上是一个控制并发运行作业数量的信号量。并发性可以使用 `--jobs` 标志设置，默认为逻辑 CPU 的数量。

每个构建脚本从 Cargo 继承一个作业槽，并且应努力在运行时仅使用一个 CPU。如果脚本想要并行使用更多 CPU，它应使用 [`jobserver` crate] 与 Cargo 协调。

例如，[`cc` crate] 可以启用可选的 `parallel` 特性，该特性将使用 jobserver 协议尝试同时构建多个 C 文件。

[`cc` crate]: https://crates.io/crates/cc
[`jobserver` crate]: https://crates.io/crates/jobserver
[jobserver protocol]: http://make.mad-scientist.net/papers/jobserver-implementation/
[crates.io]: https://crates.io/