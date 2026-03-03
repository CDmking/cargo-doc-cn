# Cargo 手册 {#the-cargo-book}

![Cargo 标识](images/Cargo-Logo-Small.png)

Cargo 是 [Rust] 语言的[*包管理器*][def-package-manager]。Cargo 负责下载您 Rust [包][def-package]的依赖项，编译您的包，制作可分发的包，并将其上传到 Rust 社区的[*包注册中心*][def-package-registry] [crates.io]。您可以在 [GitHub] 上参与本书的贡献。

## 章节 {#sections}

**[入门指南](getting-started/index.md)**

要开始使用 Cargo，请安装 Cargo（和 Rust）并设置您的第一个 [*crate*][def-crate]。

**[Cargo 指南](guide/index.md)**

本指南将为您提供使用 Cargo 开发 Rust 包所需了解的所有知识。

**[Cargo 参考](reference/index.md)**

参考手册涵盖了 Cargo 各个领域的详细信息。

**[Cargo 命令](commands/index.md)**

命令部分将使您能够通过命令行界面与 Cargo 交互。

**[常见问题解答](faq.md)**

**附录：**
* [词汇表](appendix/glossary.md)
* [Git 认证](appendix/git-authentication.md)

**其他文档：**
* [更新日志](CHANGELOG.md)
  --- 关于 Cargo 每个版本变更的详细说明。
* [Rust 文档网站](https://doc.rust-lang.org/) --- 官方 Rust 文档和工具的链接。

[def-crate]:            ./appendix/glossary.md#crate            '"crate" (词汇表条目)'
[def-package]:          ./appendix/glossary.md#package          '"package" (词汇表条目)'
[def-package-manager]:  ./appendix/glossary.md#package-manager  '"package manager" (词汇表条目)'
[def-package-registry]: ./appendix/glossary.md#package-registry '"package registry" (词汇表条目)'
[rust]: https://www.rust-lang.org/
[crates.io]: https://crates.io/
[GitHub]: https://github.com/rust-lang/cargo/tree/master/src/doc