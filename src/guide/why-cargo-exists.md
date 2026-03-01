# Cargo 存在的原因

## 预备知识

在 Rust 中，如您所知，一个库或可执行程序被称为 [*crate*][def-crate]。Crate 使用 Rust 编译器 `rustc` 进行编译。当开始学习 Rust 时，大多数人遇到的第一段源代码是经典的“hello world”程序，他们通过直接调用 `rustc` 来编译它：

```console
$ rustc hello.rs
$ ./hello
Hello, world!
```

请注意，上述命令要求您明确指定文件名。如果您要直接使用 `rustc` 编译另一个程序，则需要不同的命令行调用。如果需要指定任何特定的编译器标志或包含外部依赖项，那么所需的命令将更加具体（且复杂）。

此外，大多数非简单的程序都可能依赖于外部库，因此也会传递性地依赖于*它们*的依赖项。如果手动完成，获取所有必要依赖项的正确版本并保持其最新状态将非常困难且容易出错。

与其仅与 crate 和 `rustc` 打交道，您可以通过引入更高层次的 [*package*][def-package] 抽象和使用 [*package manager*][def-package-manager] 来避免执行上述任务所涉及的困难。

## 引入：Cargo

*Cargo* 是 Rust 的包管理器。它是一个工具，允许 Rust [*package*][def-package] 声明其各种依赖项，并确保您始终获得可重复的构建。

为了实现这个目标，Cargo 做了四件事：

* 引入两个包含各种包信息的元数据文件。
* 获取并构建您的包的依赖项。
* 使用正确的参数调用 `rustc` 或其他构建工具来构建您的包。
* 引入约定，使使用 Rust 包更加容易。

在很大程度上，Cargo 规范化了构建给定程序或库所需的命令；这是上述约定的一个方面。正如我们稍后将展示的，相同的命令可以用于构建不同的 [*artifact*][def-artifact]，无论它们的名称是什么。与其直接调用 `rustc`，您可以改为调用一些通用的命令，例如 `cargo build`，并让 Cargo 负责构造正确的 `rustc` 调用。此外，Cargo 会自动从 [*registry*][def-registry] 获取您为构建产物定义的任何依赖项，并安排它们根据需要添加到构建中。

毫不夸张地说，一旦您知道如何构建一个基于 Cargo 的项目，您就知道如何构建*所有*基于 Cargo 的项目。

[def-artifact]:         ../appendix/glossary.md#artifact         '"artifact"（词汇表条目）'
[def-crate]:            ../appendix/glossary.md#crate            '"crate"（词汇表条目）'
[def-package]:          ../appendix/glossary.md#package          '"package"（词汇表条目）'
[def-package-manager]:  ../appendix/glossary.md#package-manager  '"package manager"（词汇表条目）'
[def-registry]:         ../appendix/glossary.md#registry         '"registry"（词汇表条目）'