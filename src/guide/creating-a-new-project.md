# 创建新项目 {#creating-a-new-package}

要使用 Cargo 启动一个新的 [package][def-package]（包），请使用 `cargo new`：

```console
$ cargo new hello_world --bin
```

我们传递了 `--bin` 参数，因为我们正在创建一个二进制程序：如果我们正在创建一个库，我们会传递 `--lib`。默认情况下，这也会初始化一个新的 `git` 仓库。如果您不希望这样做，请传递 `--vcs none`。

让我们看看 Cargo 为我们生成了什么：

```console
$ cd hello_world
$ tree .
.
├── Cargo.toml
└── src
    └── main.rs

1 directory, 2 files
```

让我们仔细看看 `Cargo.toml`：

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"

[dependencies]

```

这被称为 [***manifest***][def-manifest]（清单），它包含了 Cargo 编译您的包所需的所有元数据。此文件采用 [TOML] 格式（发音为 /tɑməl/）。

以下是 `src/main.rs` 中的内容：

```rust
fn main() {
    println!("Hello, world!");
}
```

Cargo 为您生成了一个“hello world”程序，也称为 [*binary crate*][def-crate]（二进制 crate）。让我们编译它：

```console
$ cargo build
   Compiling hello_world v0.1.0 (file:///path/to/package/hello_world)
```

然后运行它：

```console
$ ./target/debug/hello_world
Hello, world!
```

您也可以使用 `cargo run` 来编译并运行，一步完成（如果您自上次编译以来未做任何更改，则不会看到 `Compiling` 这一行）：

```console
$ cargo run
   Compiling hello_world v0.1.0 (file:///path/to/package/hello_world)
     Running `target/debug/hello_world`
Hello, world!
```

现在您会注意到一个新文件 `Cargo.lock`。它包含有关您的依赖项的信息。由于目前还没有依赖项，它并不那么有趣。

当您准备好发布时，可以使用 `cargo build --release` 来启用优化编译您的文件：

```console
$ cargo build --release
   Compiling hello_world v0.1.0 (file:///path/to/package/hello_world)
```

`cargo build --release` 将生成的二进制文件放在 `target/release` 目录中，而不是 `target/debug`。

调试模式编译是开发时的默认设置。由于编译器不进行优化，编译时间更短，但代码运行速度较慢。发布模式编译时间更长，但代码运行速度更快。

[TOML]: https://toml.io/
[def-crate]:     ../appendix/glossary.md#crate     '"crate"（词汇表条目）'
[def-manifest]:  ../appendix/glossary.md#manifest  '"manifest"（词汇表条目）'
[def-package]:   ../appendix/glossary.md#package   '"package"（词汇表条目）'