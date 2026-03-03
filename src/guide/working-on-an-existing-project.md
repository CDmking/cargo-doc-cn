# 在已有的 Cargo 项目上工作 {#working-on-an-existing-cargo-package}

如果您下载了一个使用 Cargo 的现有 [package][def-package]（包），那么开始工作非常简单。

首先，从某个地方获取该包。在此示例中，我们将使用从 GitHub 上的仓库克隆的 `regex`：

```console
$ git clone https://github.com/rust-lang/regex.git
$ cd regex
```

要构建，请使用 `cargo build`：

```console
$ cargo build
   Compiling regex v1.5.0 (file:///path/to/package/regex)
```

这将获取所有依赖项，然后构建它们以及该包。

[def-package]:  ../appendix/glossary.md#package  '"package"（词汇表条目）'