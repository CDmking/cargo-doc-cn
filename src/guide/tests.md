# 测试 {#tests}

Cargo 可以通过 `cargo test` 命令运行您的测试。Cargo 在两个位置查找要运行的测试：您的每个 `src` 文件中的测试和 `tests/` 目录中的任何测试。您 `src` 文件中的测试应是单元测试和[文档测试]。`tests/` 目录中的测试应是集成风格的测试。因此，您需要将您的 crate 导入到 `tests/` 中的文件中。

以下是在我们的[包][def-package]中运行 `cargo test` 的示例，该包目前没有测试：

```console
$ cargo test
   Compiling regex v1.5.0 (https://github.com/rust-lang/regex.git#9f9f693)
   Compiling hello_world v0.1.0 (file:///path/to/package/hello_world)
     Running target/test/hello_world-9c2b65bbb79eabce

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

如果您的包有测试，您将看到包含正确测试数量的更多输出。

您也可以通过传递过滤器来运行特定测试：

```console
$ cargo test foo
```

这将运行任何名称中包含 `foo` 的测试。

`cargo test` 还会执行额外的检查。它将编译您包含的任何示例以确保它们仍然可以编译。同时，它会运行文档测试以确保来自文档注释的代码示例能够编译。关于编写和组织测试的概述，请参阅 Rust 文档中的[测试指南][testing]。要了解 Cargo 中不同风格的测试，请参阅 [Cargo 目标：测试]。

[文档测试]: ../../rustdoc/write-documentation/documentation-tests.html
[def-package]: ../appendix/glossary.md#package '"包" (glossary entry)'
[testing]: ../../book/ch11-00-testing.html
[Cargo 目标：测试]: ../reference/cargo-targets.html#tests