# 包布局 {#package-layout}

Cargo 使用文件放置的约定，以便于快速了解新的 Cargo [包][def-package]：

```text
.
├── Cargo.lock
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── main.rs
│   └── bin/
│       ├── named-executable.rs
│       ├── another-executable.rs
│       └── multi-file-executable/
│           ├── main.rs
│           └── some_module.rs
├── benches/
│   ├── large-input.rs
│   └── multi-file-bench/
│       ├── main.rs
│       └── bench_module.rs
├── examples/
│   ├── simple.rs
│   └── multi-file-example/
│       ├── main.rs
│       └── ex_module.rs
└── tests/
    ├── some-integration-tests.rs
    └── multi-file-test/
        ├── main.rs
        └── test_module.rs
```

* `Cargo.toml` 和 `Cargo.lock` 存储在包的根目录（*包根目录*）。
* 源代码放在 `src` 目录中。
* 默认的库文件是 `src/lib.rs`。
* 默认的可执行文件是 `src/main.rs`。
    * 其他可执行文件可以放在 `src/bin/` 目录中。
* 基准测试放在 `benches` 目录中。
* 示例放在 `examples` 目录中。
* 集成测试放在 `tests` 目录中。

如果一个二进制文件、示例、基准测试或集成测试由多个源文件组成，请在 `src/bin`、`examples`、`benches` 或 `tests` 目录的子目录中放置一个 `main.rs` 文件以及额外的[模块][def-module]。可执行文件的名称将是目录名。

> **注意**：按照约定，除非存在兼容性原因（例如，与现有的二进制文件名兼容），否则二进制文件、示例、基准测试和集成测试遵循 `kebab-case` 命名风格。这些目标中的模块遵循 `snake_case`，遵循[Rust 标准](https://rust-lang.github.io/rfcs/0430-finalizing-naming-conventions.html)。

您可以在[书中][book-modules]了解更多关于 Rust 模块系统的信息。

有关手动配置目标的更多详细信息，请参阅[配置目标][Configuring a target]。有关控制 Cargo 如何自动推断目标名称的更多信息，请参阅[目标自动发现][Target auto-discovery]。

[book-modules]: ../../book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html
[Configuring a target]: ../reference/cargo-targets.md#configuring-a-target
[def-package]:           ../appendix/glossary.md#package          '"package"（术语表条目）
[def-module]:            ../appendix/glossary.md#module           '"module"（术语表条目）
[Target auto-discovery]: ../reference/cargo-targets.md#target-auto-discovery