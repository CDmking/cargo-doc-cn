# 安装 {#installation}

## 安装 Rust 和 Cargo {#install-rust-and-cargo}

获取 Cargo 最简单的方式是通过 [rustup] 安装当前稳定版的 [Rust]。使用 `rustup` 安装 Rust 的同时也会安装 `cargo`。

在 Linux 和 macOS 系统上，通过以下命令安装：

```console
curl https://sh.rustup.rs -sSf | sh
```

此命令将下载一个脚本并开始安装。如果一切顺利，您将看到以下信息：

```console
Rust 现已安装完毕。太棒了！
```

在 Windows 上，下载并运行 [rustup-init.exe]。它将在控制台中开始安装，并在成功后显示上述信息。

安装完成后，您可以使用 `rustup` 命令为 Rust 和 Cargo 安装 `beta` 或 `nightly` 频道。

有关其他安装选项和更多信息，请访问 Rust 网站的[安装][install-rust]页面。

## 从源代码构建和安装 Cargo {#build-and-install-cargo-from-source}

或者，您可以[从源代码构建 Cargo][compiling-from-source]。

[rust]: https://www.rust-lang.org/
[rustup]: https://rustup.rs/
[rustup-init.exe]: https://win.rustup.rs/
[install-rust]: https://www.rust-lang.org/tools/install
[compiling-from-source]: https://github.com/rust-lang/cargo#compiling-from-source