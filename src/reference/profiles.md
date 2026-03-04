# 构建配置 {#profiles}

配置（profiles）提供了一种改变编译器设置的方式，影响诸如优化和调试符号等内容。

Cargo 有 4 个内置配置：`dev`、`release`、`test` 和 `bench`。如果未在命令行指定配置，则会根据正在运行的命令自动选择配置。除了内置配置外，也可以指定自定义的用户定义配置。

可以在 [`Cargo.toml`](manifest.md) 中使用 `[profile]` 表来更改配置设置。在每个命名的配置中，可以通过键/值对来更改单独的设置，如下所示：

```toml
[profile.dev]
opt-level = 1               # 使用稍好一些的优化。
overflow-checks = false     # 禁用整数溢出检查。
```

Cargo 仅查看工作空间根目录 `Cargo.toml` 清单中的配置设置。在依赖项中定义的配置设置将被忽略。

此外，配置可以通过 [config] 定义进行覆盖。在配置文件或环境变量中指定配置将覆盖 `Cargo.toml` 中的设置。

[config]: config.md

## 配置设置 {#profile-settings}

以下是可以在一个配置中控制的设置列表。

### opt-level

`opt-level` 设置控制 [`-C opt-level` 标志]，该标志控制优化级别。更高的优化级别可能会以更长的编译时间为代价生成更快的运行时代码。更高的级别也可能改变和重新排列编译后的代码，这可能使其更难与调试器一起使用。

有效的选项是：

* `0`：无优化
* `1`：基本优化
* `2`：一些优化
* `3`：全部优化
* `"s"`：针对二进制大小进行优化
* `"z"`：针对二进制大小进行优化，但同时关闭循环向量化。

建议尝试不同的级别，为你的项目找到合适的平衡点。可能会有令人惊讶的结果，例如级别 `3` 比 `2` 慢，或者 `"s"` 和 `"z"` 级别不一定更小。随着 `rustc` 新版本改变优化行为，你可能还需要随着时间的推移重新评估你的设置。

另请参阅 [Profile Guided Optimization] 以获取更高级的优化技术。

[`-C opt-level` 标志]: ../../rustc/codegen-options/index.html#opt-level
[Profile Guided Optimization]: ../../rustc/profile-guided-optimization.html

### debug

`debug` 设置控制 [`-C debuginfo` 标志]，该标志控制编译后的二进制文件中包含的调试信息量。

有效的选项是：

* `0`、`false` 或 `"none"`：完全没有调试信息，[`release`](#release) 的默认值
* `"line-directives-only"`：仅行信息指令。对于 nvptx* 目标，这会启用 [profiling]。对于其他用例，`line-tables-only` 是更好、更兼容的选择。
* `"line-tables-only"`：仅行表。生成用于带有文件名/行号信息的回溯的最小量调试信息，但没有其他任何内容，即没有变量或函数参数信息。
* `1` 或 `"limited"`：不包含类型或变量级别信息的调试信息。生成比 `line-tables-only` 更详细的模块级别信息。
* `2`、`true` 或 `"full"`：完整的调试信息，[`dev`](#dev) 的默认值

有关每个选项作用的更多信息，请参阅 `rustc` 关于 [debuginfo] 的文档。

根据你的需要，你可能还希望配置 [`split-debuginfo`](#split-debuginfo) 选项。

> **MSRV：** 需要 1.71 版本才能使用 `none`、`limited`、`full`、`line-directives-only` 和 `line-tables-only`

[`-C debuginfo` 标志]: ../../rustc/codegen-options/index.html#debuginfo
[debuginfo]: ../../rustc/codegen-options/index.html#debuginfo
[profiling]: https://reviews.llvm.org/D46061

### split-debuginfo

`split-debuginfo` 设置控制 [`-C split-debuginfo` 标志]，该标志控制调试信息（如果生成）是放在可执行文件本身内部还是放在其旁边。

此选项是一个字符串，可接受的值与[编译器接受的][`-C split-debuginfo` 标志]值相同。此选项的默认值在 macOS 上对于启用了调试信息的配置是 `unpacked`。否则，此选项的默认值[在 rustc 文档中有记录][`-C split-debuginfo` 标志]并且是平台特定的。某些选项仅在 [nightly channel] 上可用。一旦进行了更多测试并且对 DWARF 的支持稳定下来，Cargo 的默认值将来可能会改变。

请注意，Cargo 和 rustc 对此选项有不同的默认值。此选项的存在是为了允许 Cargo 尝试不同的标志组合，从而提供更好的调试和开发者体验。

[nightly channel]: ../../book/appendix-07-nightly-rust.html
[`-C split-debuginfo` 标志]: ../../rustc/codegen-options/index.html#split-debuginfo

### strip

`strip` 选项控制 [`-C strip` 标志]，该标志指示 rustc 从二进制文件中剥离符号或调试信息。可以像这样启用：

```toml
[package]
# ...

[profile.release]
strip = "debuginfo"
```

`strip` 可能的字符串值是 `"none"`、`"debuginfo"` 和 `"symbols"`。默认值是 `"none"`。

你也可以使用布尔值 `true` 或 `false` 来配置此选项。`strip = true` 等同于 `strip = "symbols"`。`strip = false` 等同于 `strip = "none"` 并完全禁用 `strip`。

[`-C strip` 标志]: ../../rustc/codegen-options/index.html#strip

### debug-assertions

`debug-assertions` 设置控制 [`-C debug-assertions` 标志]，该标志打开或关闭 `cfg(debug_assertions)` [条件编译]。调试断言旨在包含仅在调试/开发版本中可用的运行时验证。这些可能是那些在发布版本中过于昂贵或其他方面不可取的东西。调试断言启用标准库中的 [`debug_assert!` 宏]。

有效的选项是：

* `true`：启用
* `false`：禁用

[`-C debug-assertions` 标志]: ../../rustc/codegen-options/index.html#debug-assertions
[条件编译]: ../../reference/conditional-compilation.md#debug_assertions
[`debug_assert!` 宏]: ../../std/macro.debug_assert.html

### overflow-checks

`overflow-checks` 设置控制 [`-C overflow-checks` 标志]，该标志控制[运行时整数溢出]的行为。当溢出检查启用时，溢出将发生 panic。

有效的选项是：

* `true`：启用
* `false`：禁用

[`-C overflow-checks` 标志]: ../../rustc/codegen-options/index.html#overflow-checks
[运行时整数溢出]: ../../reference/expressions/operator-expr.md#overflow

### lto

`lto` 设置控制 `rustc` 的 [`-C lto`]、[`-C linker-plugin-lto`] 和 [`-C embed-bitcode`] 选项，这些选项控制 LLVM 的[链接时优化]。LTO 可以使用全程序分析生成更好优化的代码，但代价是更长的链接时间。

有效的选项是：

* `true` 或 `"fat"`：执行"fat" LTO，尝试在依赖关系图中的所有 crate 之间执行优化。
* `"thin"`：执行["thin" LTO]。这类似于"fat"，但运行时间大大减少，同时仍能实现与"fat"类似的性能提升。
* `false`：执行"thin local LTO"，仅在其[代码生成单元](#codegen-units)上对本地 crate 执行"thin" LTO。如果代码生成单元为 1 或 [opt-level](#opt-level) 为 0，则不执行 LTO。
* `"off"`：禁用 LTO。

如果你对跨语言 LTO 感兴趣，请参阅 [linker-plugin-lto 章节]。这尚未在 Cargo 中原生支持，但可以通过 `RUSTFLAGS` 执行。

[`-C lto`]: ../../rustc/codegen-options/index.html#lto
[链接时优化]: https://llvm.org/docs/LinkTimeOptimization.html
[`-C linker-plugin-lto`]: ../../rustc/codegen-options/index.html#linker-plugin-lto
[`-C embed-bitcode`]: ../../rustc/codegen-options/index.html#embed-bitcode
[linker-plugin-lto 章节]: ../../rustc/linker-plugin-lto.html
["thin" LTO]: http://blog.llvm.org/2016/06/thinlto-scalable-and-incremental-lto.html

### panic

`panic` 设置控制 [`-C panic` 标志]，该标志控制使用哪种 panic 策略。

有效的选项是：

* `"unwind"`：panic 时展开堆栈。
* `"abort"`：panic 时终止进程。

当设置为 `"unwind"` 时，实际值取决于目标平台的默认值。例如，NVPTX 平台不支持展开，因此它总是使用 `"abort"`。

测试、基准测试、构建脚本和过程宏会忽略 `panic` 设置。`rustc` 测试工具目前需要 `unwind` 行为。请参阅启用 `abort` 行为的 [`panic-abort-tests`] 不稳定标志。

此外，当使用 `abort` 策略并构建测试时，所有依赖项也将被迫使用 `unwind` 策略构建。

[`-C panic` 标志]: ../../rustc/codegen-options/index.html#panic
[`panic-abort-tests`]: unstable.md#panic-abort-tests

### incremental

`incremental` 设置控制 [`-C incremental` 标志]，该标志控制是否启用增量编译。增量编译导致 `rustc` 将额外信息保存到磁盘，这些信息将在重新编译 crate 时重用，从而改善重新编译时间。额外的信息存储在 `target` 目录中。

有效的选项是：

* `true`：启用
* `false`：禁用

增量编译仅用于工作空间成员和"路径"依赖项。

增量值可以通过 `CARGO_INCREMENTAL` [环境变量] 或 [`build.incremental`] 配置变量全局覆盖。

[`-C incremental` 标志]: ../../rustc/codegen-options/index.html#incremental
[环境变量]: environment-variables.md
[`build.incremental`]: config.md#buildincremental

### codegen-units

`codegen-units` 设置控制 [`-C codegen-units` 标志]，该标志控制一个 crate 将被分割成多少个"代码生成单元"。更多的代码生成单元允许更多的 crate 内容被并行处理，可能减少编译时间，但可能生成更慢的代码。

此选项接受一个大于 0 的整数。

对于[增量](#incremental)构建，默认值为 256，对于非增量构建，默认值为 16。

[`-C codegen-units` 标志]: ../../rustc/codegen-options/index.html#codegen-units

### rpath

`rpath` 设置控制 [`-C rpath` 标志]，该标志控制是否启用 [`rpath`]。

[`-C rpath` 标志]: ../../rustc/codegen-options/index.html#rpath
[`rpath`]: https://en.wikipedia.org/wiki/Rpath

## 默认配置 {#default-profiles}

### dev

`dev` 配置用于正常的开发和调试。它是构建命令（如 [`cargo build`]）的默认配置，并用于 `cargo install --debug`。

`dev` 配置的默认设置是：

```toml
[profile.dev]
opt-level = 0
debug = true
split-debuginfo = '...'  # 平台特定。
strip = "none"
debug-assertions = true
overflow-checks = true
lto = false
panic = 'unwind'
incremental = true
codegen-units = 256
rpath = false
```

### release

`release` 配置用于发布和生产中使用的优化产物。当使用 `--release` 标志时使用此配置，并且是 [`cargo install`] 的默认配置。

`release` 配置的默认设置是：

```toml
[profile.release]
opt-level = 3
debug = false
split-debuginfo = '...'  # 平台特定。
strip = "none"
debug-assertions = false
overflow-checks = false
lto = false
panic = 'unwind'
incremental = false
codegen-units = 16
rpath = false
```

### test

`test` 配置是 [`cargo test`] 使用的默认配置。`test` 配置继承自 [`dev`](#dev) 配置的设置。

### bench

`bench` 配置是 [`cargo bench`] 使用的默认配置。`bench` 配置继承自 [`release`](#release) 配置的设置。

### 构建依赖项 {#build-dependencies}

为了快速编译，默认情况下，所有配置都不会优化构建依赖项（构建脚本、过程宏及其依赖项），并且当构建依赖项不用作运行时依赖项时，会避免计算调试信息。构建覆盖的默认设置是：

```toml
[profile.dev.build-override]
opt-level = 0
codegen-units = 256
debug = false # 在可能的情况下

[profile.release.build-override]
opt-level = 0
codegen-units = 256
```

但是，如果在运行构建依赖项时发生错误，打开完整的调试信息将在需要时改善回溯和可调试性：

```toml
debug = true
```

构建依赖项在其他情况下继承正在使用的活动配置的设置，如[配置选择](#profile-selection)中所述。

## 自定义配置 {#custom-profiles}

除了内置配置外，还可以定义额外的自定义配置。这对于设置多个工作流和构建模式可能很有用。定义自定义配置时，必须指定 `inherits` 键，以指定当未指定设置时，自定义配置从哪个配置继承设置。

例如，假设你想比较正常的发布构建与启用了 [LTO](#lto) 优化的发布构建，你可以在 `Cargo.toml` 中指定类似以下内容：

```toml
[profile.release-lto]
inherits = "release"
lto = true
```

然后可以使用 `--profile` 标志来选择这个自定义配置：

```console
cargo build --profile release-lto
```

每个配置的输出将放置在与配置同名的目录中，位于 [`target` 目录] 内。如上例所示，输出将进入 `target/release-lto` 目录。

[`target` 目录]: build-cache.md

## 配置选择 {#profile-selection}

使用的配置取决于命令、命令行标志（如 `--release` 或 `--profile`）以及包（在[覆盖](#overrides)的情况下）。如果未指定任何配置，则默认配置是：

| 命令 | 默认配置 |
|---------|-----------------|
| [`cargo run`]、[`cargo build`]、<br>[`cargo check`]、[`cargo rustc`] | [`dev` 配置](#dev) |
| [`cargo test`] | [`test` 配置](#test)
| [`cargo bench`] | [`bench` 配置](#bench)
| [`cargo install`] | [`release` 配置](#release)

你可以使用 `--profile=NAME` 选项切换到不同的配置，该选项将使用给定的配置。`--release` 标志等同于 `--profile=release`。

所选的配置适用于所有 Cargo 目标，包括[库](./cargo-targets.md#library)、[二进制文件](./cargo-targets.md#binaries)、[示例](./cargo-targets.md#examples)、[测试](./cargo-targets.md#tests)和[基准测试](./cargo-targets.md#benchmarks)。

特定包的配置可以通过[覆盖](#overrides)来指定，如下所述。

[`cargo bench`]: ../commands/cargo-bench.md
[`cargo build`]: ../commands/cargo-build.md
[`cargo check`]: ../commands/cargo-check.md
[`cargo install`]: ../commands/cargo-install.md
[`cargo run`]: ../commands/cargo-run.md
[`cargo rustc`]: ../commands/cargo-rustc.md
[`cargo test`]: ../commands/cargo-test.md

## 覆盖 {#overrides}

配置设置可以针对特定的包和构建时的 crate 进行覆盖。要覆盖特定包的设置，请使用 `package` 表来更改命名包的设置：

```toml
# `foo` 包将使用 -Copt-level=3 标志。
[profile.dev.package.foo]
opt-level = 3
```

包名实际上是一个[包 ID 规范](pkgid-spec.md)，因此你可以使用诸如 `[profile.dev.package."foo:2.1.0"]` 之类的语法来定位包的特定版本。

要覆盖所有依赖项（但不包括任何工作空间成员）的设置，请使用 `"*"` 包名：

```toml
# 设置依赖项的默认值。
[profile.dev.package."*"]
opt-level = 2
```

要覆盖构建脚本、过程宏及其依赖项的设置，请使用 `build-override` 表：

```toml
# 设置构建脚本和过程宏的设置。
[profile.dev.build-override]
opt-level = 3
```

> 注意：当一个依赖项既是普通依赖项又是构建依赖项时，Cargo 会尝试在未指定 `--target` 时只构建它一次。当使用 `build-override` 时，依赖项可能需要构建两次，一次作为普通依赖项，一次使用覆盖的构建设置。这可能会增加初始构建时间。

使用哪个值的优先级按以下顺序确定（首次匹配获胜）：

1. `[profile.dev.package.name]` --- 一个命名的包。
2. `[profile.dev.package."*"]` --- 对于任何非工作空间成员。
3. `[profile.dev.build-override]` --- 仅用于构建脚本、过程宏及其依赖项。
4. `[profile.dev]` --- `Cargo.toml` 中的设置。
5. Cargo 内置的默认值。

覆盖不能指定 `panic`、`lto` 或 `rpath` 设置。

### 覆盖与泛型 {#overrides-and-generics}

泛型代码被实例化的位置将影响用于该泛型代码的优化设置。当使用配置覆盖来更改特定 crate 的优化级别时，这可能会导致微妙的交互。如果你尝试提高定义泛型函数的依赖项的优化级别，当这些泛型函数在你的本地 crate 中使用时，它们可能不会被优化。这是因为代码可能在其实例化的 crate 中生成，因此可能使用该 crate 的优化设置。

例如，[nalgebra] 是一个大量使用泛型参数定义向量和矩阵的库。如果你的本地代码定义了具体的 nalgebra 类型，如 `Vector4<f64>` 并使用它们的方法，相应的 nalgebra 代码将在你的 crate 内实例化和构建。因此，如果你尝试使用配置覆盖来提高 `nalgebra` 的优化级别，它可能不会带来更快的性能。

使问题进一步复杂化的是，`rustc` 有一些优化，它会尝试在 crate 之间共享单态化的泛型。如果 opt-level 是 2 或 3，那么一个 crate 将不会使用来自其他 crate 的单态化泛型，也不会导出本地定义的单态化项以与其他 crate 共享。在为开发优化依赖项进行实验时，请考虑尝试 opt-level 1，这将应用一些优化，同时仍允许单态化项被共享。

[nalgebra]: https://crates.io/crates/nalgebra