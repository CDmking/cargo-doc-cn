# 初次使用 Cargo {#first-steps-with-cargo}

本节简要介绍 `cargo` 命令行工具。我们将演示其为我们生成新 [***package***][def-package]（包）的能力、编译包内 [***crate***][def-crate] 的能力以及运行生成程序的能力。

要使用 Cargo 启动一个新包，请使用 `cargo new`：

```console
$ cargo new hello_world
```

默认情况下，Cargo 使用 `--bin` 来创建二进制程序。如果要创建库，我们需要传递 `--lib` 参数。

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

这就是我们开始所需的一切。首先，我们看一下 `Cargo.toml`：

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"

[dependencies]
```

这被称为 [***manifest***][def-manifest]（清单），它包含了 Cargo 编译您的包所需的所有元数据。

以下是 `src/main.rs` 中的内容：

```rust
fn main() {
    println!("Hello, world!");
}
```

Cargo 为我们生成了一个“hello world”程序，也称为 [***binary crate***][def-crate]（二进制 crate）。让我们编译它：

```console
$ cargo build
   Compiling hello_world v0.1.0 (file:///path/to/package/hello_world)
```

然后运行它：

```console
$ ./target/debug/hello_world
Hello, world!
```

我们也可以使用 `cargo run` 来编译并运行，一步完成：

```console
$ cargo run
     Fresh hello_world v0.1.0 (file:///path/to/package/hello_world)
   Running `target/hello_world`
Hello, world!
```

## 进一步学习 {#going-further}

有关使用 Cargo 的更多详细信息，请查阅 [Cargo 指南](../guide/index.md)

[def-crate]:     ../appendix/glossary.md#crate     '"crate"（词汇表条目）'
[def-manifest]:  ../appendix/glossary.md#manifest  '"manifest"（词汇表条目）'
[def-package]:   ../appendix/glossary.md#package   '"package"（词汇表条目）'