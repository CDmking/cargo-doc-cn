# 构建缓存 {#build-cache}

Cargo 将构建输出存储在 "target" 和 "build" 目录中。默认情况下，这两个目录都指向您的[*工作空间*][def-workspace]根目录下名为 `target` 的目录。要更改 target-dir 的位置，可以设置 `CARGO_TARGET_DIR` [环境变量]、[`build.target-dir`] 配置值或 `--target-dir` 命令行标志。要更改 build-dir 的位置，可以设置 `CARGO_BUILD_BUILD_DIR` [环境变量]或 [`build.build-dir`] 配置值。

构建产物分为两类：
* 最终构建产物
  * 最终构建产物是供 Cargo 最终用户使用的输出
  * 例如，二进制 crate 的可执行文件、`cargo doc` 的输出、Cargo `--timings` 报告
  * 存储在 target-dir 中
* 中间构建产物
  * 中间构建产物是 Cargo 和 Rust 编译器内部使用的
  * 最终用户通常不需要与中间构建产物交互
  * 存储在 Cargo build-dir 中

目录布局取决于您是否使用 `--target` 标志为特定平台构建。如果未指定 `--target`，Cargo 将以为主机架构构建的模式运行。输出放在目标目录的根目录下，每个[配置文件]存储在一个单独的子目录中：

目录 | 描述
----------|------------
<code style="white-space: nowrap">target/debug/</code> | 包含 `dev` 配置文件的输出。
<code style="white-space: nowrap">target/release/</code> | 包含 `release` 配置文件的输出（使用 `--release` 选项时）。
<code style="white-space: nowrap">target/foo/</code> | 包含 `foo` 配置文件的构建输出（使用 `--profile=foo` 选项时）。

由于历史原因，`dev` 和 `test` 配置文件存储在 `debug` 目录中，而 `release` 和 `bench` 配置文件存储在 `release` 目录中。用户定义的配置文件存储在与配置文件同名的目录中。

当使用 `--target` 为其他目标构建时，输出放在以[目标]命名的目录中：

目录 | 示例
----------|--------
<code style="white-space: nowrap">target/&lt;triple&gt;/debug/</code> | <code style="white-space: nowrap">target/thumbv7em-none-eabihf/debug/</code>
<code style="white-space: nowrap">target/&lt;triple&gt;/release/</code> | <code style="white-space: nowrap">target/thumbv7em-none-eabihf/release/</code>

> **注意**：当不使用 `--target` 时，这会导致 Cargo 将与构建脚本和过程宏共享您的依赖项。[`RUSTFLAGS`] 将与每个 `rustc` 调用共享。使用 `--target` 标志时，构建脚本和过程宏是单独构建的（为主机架构），并且不共享 `RUSTFLAGS`。

在配置文件目录（如 `debug` 或 `release`）中，构建产物被放置到以下目录中：

目录 | 描述
----------|------------
<code style="white-space: nowrap">target/debug/</code> | 包含正在构建的包的输出（[二进制可执行文件]和[库目标]）。
<code style="white-space: nowrap">target/debug/examples/</code> | 包含[示例目标]。

一些命令将它们的输出放在 `target` 目录顶层的专用目录中：

目录 | 描述
----------|------------
<code style="white-space: nowrap">target/doc/</code> | 包含 rustdoc 文档（[`cargo doc`]）。
<code style="white-space: nowrap">target/package/</code> | 包含 [`cargo package`] 的输出。

Cargo 还会在 build-dir 中创建构建过程所需的其他几个目录和文件。build-dir 的布局被视为 Cargo 内部实现，可能会发生变化。其中一些目录包括：

目录 | 描述
----------|------------
<code style="white-space: nowrap">\<build-dir>/debug/deps/</code> | 依赖项和其他构建产物。
<code style="white-space: nowrap">\<build-dir>/debug/incremental/</code> | `rustc` [增量输出]，用于加速后续构建的缓存。
<code style="white-space: nowrap">\<build-dir>/debug/build/</code> | [构建脚本][build scripts]的输出。

## 依赖信息文件 {#dep-info-files}

每个编译产物旁边都有一个名为“依赖信息”的文件，后缀为 `.d`。该文件采用类似 Makefile 的语法，指示重新构建该产物所需的所有文件依赖项。这些文件旨在与外部构建系统一起使用，以便它们可以检测是否需要重新执行 Cargo。默认情况下，文件中的路径是绝对路径。有关使用相对路径的信息，请参阅 [`build.dep-info-basedir`] 配置选项。

```Makefile
# 在 target/debug/foo.d 中找到的依赖信息文件示例
/path/to/myproj/target/debug/foo: /path/to/myproj/src/lib.rs /path/to/myproj/src/main.rs
```

## 共享缓存 {#shared-cache}

可以使用第三方工具 [sccache] 在不同工作空间之间共享已构建的依赖项。

要设置 `sccache`，请使用 `cargo install sccache` 安装它，并在调用 Cargo 之前将 `RUSTC_WRAPPER` 环境变量设置为 `sccache`。如果您使用 bash，将 `export RUSTC_WRAPPER=sccache` 添加到 `.bashrc` 中是有意义的。或者，您可以在 [Cargo 配置] 中设置 [`build.rustc-wrapper`]。有关更多详细信息，请参阅 sccache 文档。

[`RUSTFLAGS`]: ../reference/config.md#buildrustflags
[`build.dep-info-basedir`]: ../reference/config.md#builddep-info-basedir
[`build.rustc-wrapper`]: ../reference/config.md#buildrustc-wrapper
[`build.target-dir`]: ../reference/config.md#buildtarget-dir
[`build.build-dir`]: ../reference/config.md#buildbuild-dir
[`cargo doc`]: ../commands/cargo-doc.md
[`cargo package`]: ../commands/cargo-package.md
[`cargo publish`]: ../commands/cargo-publish.md
[构建脚本]: ../reference/build-scripts.md
[配置]: ../reference/config.md
[def-workspace]:  ../appendix/glossary.md#workspace  '"workspace" (glossary entry)'
[目标]: ../appendix/glossary.md#target '"target" (glossary entry)'
[环境变量]: ../reference/environment-variables.md
[增量输出]: ../reference/profiles.md#incremental
[sccache]: https://github.com/mozilla/sccache
[配置文件]: ../reference/profiles.md
[二进制可执行文件]: ../reference/cargo-targets.md#binaries
[库目标]: ../reference/cargo-targets.md#library
[示例目标]: ../reference/cargo-targets.md#examples