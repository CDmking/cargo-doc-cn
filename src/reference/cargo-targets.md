# Cargo 目标 {#cargo-targets}

Cargo 包由*目标*组成，这些目标对应于可以编译成 crate 的源文件。包可以包含[库](#library)、[二进制文件](#binaries)、[示例](#examples)、[测试](#tests)和[基准测试](#benchmarks)目标。目标列表可以在 `Cargo.toml` 清单中配置，通常通过源文件的[目录布局][package layout][自动推断](#target-auto-discovery)。

有关配置目标设置的详细信息，请参见下面的[配置目标](#configuring-a-target)。

## 库 {#library}

库目标定义了一个可以被其他库和可执行文件使用和链接的"库"。文件名默认为 `src/lib.rs`，库的名称默认为包的名称，破折号替换为下划线。一个包只能有一个库。库的设置可以在 `Cargo.toml` 的 `[lib]` 表中[自定义]。

```toml
# Cargo.toml 中自定义库的示例。
[lib]
crate-type = ["cdylib"]
bench = false
```

## 二进制文件 {#binaries}

二进制目标是在编译后可以运行的可执行程序。二进制文件的源可以是 `src/main.rs` 和/或存储在 [`src/bin/` 目录][package layout]中。对于 `src/main.rs`，默认的二进制名称是包名。每个二进制文件的设置可以在 `Cargo.toml` 的 `[[bin]]` 表中[自定义]。

二进制文件可以使用包的库的公共 API。它们还与 `Cargo.toml` 中定义的 [`[dependencies]`][dependencies] 链接。

您可以使用 [`cargo run`] 命令和 `--bin <bin-name>` 选项来运行单个二进制文件。[`cargo install`] 可以用于将可执行文件复制到公共位置。

```toml
# Cargo.toml 中自定义二进制文件的示例。
[[bin]]
name = "cool-tool"
test = false
bench = false

[[bin]]
name = "frobnicator"
required-features = ["frobnicate"]
```

## 示例 {#examples}

位于 [`examples` 目录][package layout]下的文件是库提供的功能的示例用法。编译后，它们被放置在 [`target/debug/examples` 目录][build cache]中。

示例可以使用包的库的公共 API。它们还与 `Cargo.toml` 中定义的 [`[dependencies]`][dependencies] 和 [`[dev-dependencies]`][dev-dependencies] 链接。

默认情况下，示例是可执行的二进制文件（带有 `main()` 函数）。您可以指定 [`crate-type` 字段](#the-crate-type-field) 来使示例编译为库：

```toml
[[example]]
name = "foo"
crate-type = ["staticlib"]
```

您可以使用 [`cargo run`] 命令和 `--example <example-name>` 选项来运行单个可执行示例。库示例可以使用 [`cargo build`] 和 `--example <example-name>` 选项构建。[`cargo install`] 和 `--example <example-name>` 选项可以用于将可执行的二进制文件复制到公共位置。默认情况下，示例由 [`cargo test`] 编译以防止它们逐渐损坏。如果示例中有 `#[test]` 函数并且您希望用 [`cargo test`] 运行它们，请将 [the `test` 字段](#the-test-field) 设置为 `true`。

## 测试 {#tests}

Cargo 项目中有两种测试风格：

* *单元测试*，这些是用 [`#[test]` 属性][test-attribute] 标记的函数，位于您的库或二进制文件（或任何启用了 [the `test` 字段](#the-test-field) 的目标）中。这些测试可以访问它们定义所在目标中的私有 API。
* *集成测试*，这是一个独立可执行的二进制文件，同样包含 `#[test]` 函数，它与项目的库链接并可以访问其*公共* API。

测试使用 [`cargo test`] 命令运行。默认情况下，Cargo 和 `rustc` 使用 [libtest 测试工具][libtest harness]，它负责收集用 [`#[test]` 属性][test-attribute] 注释的函数并并行执行它们，报告每个测试的成功和失败。如果您想使用不同的测试工具或测试策略，请参见 [the `harness` 字段](#the-harness-field)。

> **注意**：Cargo 中还有另一种特殊的测试风格：[文档测试][documentation examples]。
> 它们由 `rustdoc` 处理，具有稍微不同的执行模型。
> 更多信息，请参见 [`cargo test`][cargo-test-documentation-tests]。

[libtest harness]: ../../rustc/tests/index.html
[cargo-test-documentation-tests]: ../commands/cargo-test.md#documentation-tests

### 集成测试 {#integration-tests}

位于 [`tests` 目录][package layout]下的文件是集成测试。当您运行 [`cargo test`] 时，Cargo 会将每个文件编译为单独的 crate 并执行它们。

集成测试可以使用包的库的公共 API。它们还与 `Cargo.toml` 中定义的 [`[dependencies]`][dependencies] 和 [`[dev-dependencies]`][dev-dependencies] 链接。

如果要在多个集成测试之间共享代码，可以将其放在单独的模块中，例如 `tests/common/mod.rs`，然后在每个测试中放入 `mod common;` 来导入它。

每个集成测试都会产生一个单独的可执行二进制文件，[`cargo test`] 将串行运行它们。在某些情况下，这可能效率低下，因为编译可能需要更长时间，并且在运行测试时可能无法充分利用多个 CPU。如果您有很多集成测试，您可能希望考虑创建一个单一的集成测试，并将测试拆分为多个模块。libtest 测试工具将自动找到所有用 `#[test]` 注释的函数并并行运行它们。您可以将模块名称传递给 [`cargo test`] 以仅运行该模块中的测试。

如果有集成测试，二进制目标会自动构建。这允许集成测试执行二进制文件以练习和测试其行为。集成测试构建时会设置 `CARGO_BIN_EXE_<name>` [环境变量]，以便它可以使用 [`env` 宏] 来定位可执行文件。

[环境变量]: environment-variables.md#environment-variables-cargo-sets-for-crates
[`env` 宏]: ../../std/macro.env.html

## 基准测试 {#benchmarks}

基准测试提供了一种使用 [`cargo bench`] 命令测试代码性能的方法。它们遵循与[测试](#tests)相同的结构，每个基准测试函数都用 `#[bench]` 属性注释。与测试类似：

* 基准测试放在 [`benches` 目录][package layout]中。
* 库和二进制文件中定义的基准测试函数可以访问它们定义所在目标中的*私有* API。`benches` 目录中的基准测试可以使用*公共* API。
* [the `bench` 字段](#the-bench-field) 可以用于定义默认情况下哪些目标被基准测试。
* [the `harness` 字段](#the-harness-field) 可以用于禁用内置的测试工具。

> **注意**：[`#[bench]` 属性](../../unstable-book/library-features/test.html) 目前不稳定，仅在 [nightly 频道] 上可用。[crates.io](https://crates.io/keywords/benchmark) 上有一些包可以帮助在稳定频道上运行基准测试，例如 [Criterion](https://crates.io/crates/criterion)。

## 配置目标 {#configuring-a-target}

`Cargo.toml` 中的所有 `[lib]`、`[[bin]]`、`[[example]]`、`[[test]]` 和 `[[bench]]` 部分都支持类似的配置，用于指定应如何构建目标。像 `[[bin]]` 这样的双括号部分是 [TOML 的表数组](https://toml.io/en/v1.0.0-rc.3#array-of-tables)，这意味着您可以编写多个 `[[bin]]` 部分来在您的 crate 中创建多个可执行文件。您只能指定一个库，因此 `[lib]` 是一个普通的 TOML 表。

以下是每个目标的 TOML 设置概述，每个字段的详细描述如下。

```toml
[lib]
name = "foo"           # 目标的名称。
path = "src/lib.rs"    # 目标的源文件。
test = true            # 默认情况下是否被测试。
doctest = true         # 默认情况下是否测试文档示例。
bench = true           # 默认情况下是否被基准测试。
doc = true             # 默认情况下是否包含在文档中。
proc-macro = false     # 对于过程宏库，设置为 `true`。
harness = true         # 使用 libtest 测试工具。
crate-type = ["lib"]   # 生成的 crate 类型。
required-features = [] # 构建此目标所需的功能（对于库不适用）。
```

### `name` 字段 {#the-name-field}

`name` 字段指定目标的名称，对应于将生成的产物的文件名。对于库，这是依赖项将用来引用它的 crate 名称。

对于库目标，默认是包的名称，破折号替换为下划线。对于默认的二进制文件（`src/main.rs`），它也默认为包的名称，破折号不作替换。对于[自动发现](#target-auto-discovery)的目标，它默认为目录或文件名。

除了 `[lib]` 外，所有目标都需要此字段。

### `path` 字段 {#the-path-field}

`path` 字段指定 crate 的源文件位置，相对于 `Cargo.toml` 文件。

如果未指定，则使用基于目标名称的[推断路径](#target-auto-discovery)。

### `test` 字段 {#the-test-field}

`test` 字段指示目标是否默认由 [`cargo test`] 测试。对于库、二进制文件和测试，默认为 `true`。

> **注意**：示例默认由 [`cargo test`] 构建以确保它们继续编译，但它们默认不被*测试*。将示例的 `test = true` 将使其作为测试构建并运行示例中定义的任何 [`#[test]`][test-attribute] 函数。

### `doctest` 字段 {#the-doctest-field}

`doctest` 字段指示[文档示例][documentation examples]是否默认由 [`cargo test`] 测试。这只与库相关，对其他部分没有影响。对于库，默认为 `true`。

### `bench` 字段 {#the-bench-field}

`bench` 字段指示目标是否默认由 [`cargo bench`] 进行基准测试。对于库、二进制文件和基准测试，默认为 `true`。

### `doc` 字段 {#the-doc-field}

`doc` 字段指示目标是否默认包含在 [`cargo doc`] 生成的文档中。对于库和二进制文件，默认为 `true`。

> **注意**：如果二进制文件的名称与 lib 目标相同，则将被跳过。

### `plugin` 字段 {#the-plugin-field}

此选项已弃用且未使用。

### `proc-macro` 字段 {#the-proc-macro-field}

`proc-macro` 字段指示该库是一个[过程宏][procedural macro]（[参考][proc-macro-reference]）。这仅对 `[lib]` 目标有效。

### `harness` 字段 {#the-harness-field}

`harness` 字段指示将向 `rustc` 传递 [`--test` 标志]，这将自动包含 libtest 库，该库是用于收集和运行用 [`#[test]` 属性][test-attribute] 标记的测试或用 `#[bench]` 属性标记的基准测试的驱动程序。对于所有目标，默认为 `true`。

如果设置为 `false`，则您需要负责定义一个 `main()` 函数来运行测试和基准测试。

无论是否启用测试工具，测试都具有启用的 [`cfg(test)` 条件表达式][cfg-test]。

### `crate-type` 字段 {#the-crate-type-field}

`crate-type` 字段定义了目标将生成的[crate 类型]。它是一个字符串数组，允许您为单个目标指定多种 crate 类型。这只能为库和示例指定。二进制文件、测试和基准测试始终是 "bin" crate 类型。默认值如下：

目标 | Crate 类型
-------|-----------
普通库 | `"lib"`
过程宏库 | `"proc-macro"`
示例 | `"bin"`

可用选项有 `bin`、`lib`、`rlib`、`dylib`、`cdylib`、`staticlib` 和 `proc-macro`。您可以在 [Rust 参考手册][crate types]中阅读有关不同 crate 类型的更多信息。

### `required-features` 字段 {#the-required-features-field}

`required-features` 字段指定构建目标所需的[功能]。如果任一所需功能未启用，目标将被跳过。这只与 `[[bin]]`、`[[bench]]`、`[[test]]` 和 `[[example]]` 部分相关，对 `[lib]` 没有影响。

```toml
[features]
# ...
postgres = []
sqlite = []
tools = []

[[bin]]
name = "my-pg-tool"
required-features = ["postgres", "tools"]
```

### `edition` 字段 {#the-edition-field}

`edition` 字段定义目标将使用的[Rust 版本]。如果未指定，则默认为 `[package]` 的 [`edition` 字段][package-edition]。

> **注意：** 此字段已弃用，将在未来的版本中移除

## 目标自动发现 {#target-auto-discovery}

默认情况下，Cargo 根据文件系统上的[文件布局][package layout]自动确定要构建的目标。目标配置表，如 `[lib]`、`[[bin]]`、`[[test]]`、`[[bench]]` 或 `[[example]]`，可以用于添加不遵循标准目录布局的额外目标。

可以禁用自动目标发现，以便仅构建手动配置的目标。在 `[package]` 部分将 `autolib`、`autobins`、`autoexamples`、`autotests` 或 `autobenches` 键设置为 `false` 将禁用相应目标类型的自动发现。

```toml
[package]
# ...
autolib = false
autobins = false
autoexamples = false
autotests = false
autobenches = false
```

禁用自动发现应仅在需要特殊情况下使用。例如，如果您有一个库，其中想要一个名为 `bin` 的*模块*，这将带来问题，因为 Cargo 通常会尝试将 `bin` 目录中的任何内容编译为可执行文件。以下是此场景的示例布局：

```text
├── Cargo.toml
└── src
    ├── lib.rs
    └── bin
        └── mod.rs
```

为了防止 Cargo 将 `src/bin/mod.rs` 推断为可执行文件，请在 `Cargo.toml` 中设置 `autobins = false` 以禁用自动发现：

```toml
[package]
# …
autobins = false
```

> **注意**：对于 2015 版本的包，如果在 `Cargo.toml` 中手动定义了至少一个目标，则自动发现的默认值为 `false`。
> 从 2018 版本开始，默认值始终为 `true`。

> **MSRV：** 自 1.27 起支持 `autobins`、`autoexamples`、`autotests` 和 `autobenches`

> **MSRV：** 自 1.83 起支持 `autolib`

[构建缓存]: build-cache.md
[Rust 版本]: ../../edition-guide/index.html
[`--test` 标志]: ../../rustc/command-line-arguments.html#option-test
[`cargo bench`]: ../commands/cargo-bench.md
[`cargo build`]: ../commands/cargo-build.md
[`cargo doc`]: ../commands/cargo-doc.md
[`cargo install`]: ../commands/cargo-install.md
[`cargo run`]: ../commands/cargo-run.md
[`cargo test`]: ../commands/cargo-test.md
[cfg-test]: ../../reference/conditional-compilation.html#test
[crate types]: ../../reference/linkage.html
[crates.io]: https://crates.io/
[自定义]: #configuring-a-target
[dependencies]: specifying-dependencies.md
[dev-dependencies]: specifying-dependencies.md#development-dependencies
[documentation examples]: ../../rustdoc/documentation-tests.html
[功能]: features.md
[nightly 频道]: ../../book/appendix-07-nightly-rust.html
[package layout]: ../guide/project-layout.md
[package-edition]: manifest.md#the-edition-field
[proc-macro-reference]: ../../reference/procedural-macros.html
[procedural macro]: ../../book/ch19-06-macros.html
[test-attribute]: ../../reference/attributes/testing.html#the-test-attribute