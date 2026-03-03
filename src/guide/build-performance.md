# 优化构建性能 {#optimizing-build-performance}

通过优先考虑构建性能而不是其他在您的情况下可能不那么重要的方面，Cargo 配置选项和源代码组织模式可以帮助提高构建性能。

与优化运行时性能时一样，请务必根据您实际关心的工作流来衡量这些更改，因为我们提供的是通用指南，而您的情况可能不同，其中一些方法实际上可能会使您的用例的构建性能变差。

需要考虑的示例工作流包括：
- 开发过程中的编译器反馈（代码更改后的 `cargo check`）
- 开发过程中的测试反馈（代码更改后的 `cargo test`）
- CI 构建

## Cargo 和编译器配置 {#cargo-and-compiler-configuration}

Cargo 使用的配置默认值试图平衡多个方面，包括可调试性、运行时性能、构建性能、二进制大小等。本节描述了几种更改这些默认值的方法，这些方法应旨在最大化构建性能。

覆盖默认值的常见位置有：
- [`Cargo.toml` 清单](../reference/profiles.md)
  - 所有为项目做出贡献的开发人员都可以使用
  - 支持的配置有限（有关扩展此功能，请参见 [#12738](https://github.com/rust-lang/cargo/issues/12738)）
- [`$WORKSPACE_ROOT/.cargo/config.toml` 配置文件](../reference/config.md)
  - 所有为项目做出贡献的开发人员都可以使用
  - 与 `Cargo.toml` 不同，这取决于您调用 `cargo` 的目录（请参见 [#2930](https://github.com/rust-lang/cargo/issues/2930)）
- [`$CARGO_HOME/.cargo/config.toml` 配置文件](../reference/config.md)
  - 供开发人员控制其开发的默认值

### 减少生成的调试信息量 {#reduce-amount-of-generated-debug-information}

建议：添加到您的 `Cargo.toml` 或 `.cargo/config.toml` 中：

```toml
[profile.dev]
debug = "line-tables-only"

[profile.dev.package."*"]
debug = false

[profile.debugging]
inherits = "dev"
debug = true
```

这将：
- 将 [`dev` 配置文件](../reference/profiles.md#dev)（开发命令的默认值）更改为：
  - 将工作空间成员的[调试信息](../reference/profiles.md#debug)限制为提供有用 panic 回溯所需的内容
  - 避免为依赖项生成任何调试信息
- 在通过 [`--profile debugging`](../reference/profiles.md#custom-profiles) 进行调试时提供选择加入

> **注意：** 重新评估 `dev` 配置文件正在 [#15931](https://github.com/rust-lang/cargo/issues/15931) 中进行跟踪。

权衡：
- ✅ 更快的代码生成（`cargo build`）
- ✅ 更快的链接时间
- ✅ 更小的 `target` 目录磁盘使用量
- ❌ 需要完全重新构建才能获得高质量的调试器体验

### 使用替代的代码生成后端 {#use-an-alternative-codegen-backend}

建议：

- 安装 Cranelift 代码生成后端 rustup 组件
    ```console
    $ rustup component add rustc-codegen-cranelift-preview --toolchain nightly
    ```
- 添加到您的 `Cargo.toml` 或 `.cargo/config.toml`：
    ```toml
    [profile.dev]
    codegen-backend = "cranelift"
    ```
- 使用 `-Z codegen-backend` 运行 Cargo，或在 `.cargo/config.toml` 中启用 [`codegen-backend`](../reference/unstable.md#codegen-backend) 功能。
  - 这是必需的，因为这是一个不稳定的功能。

这将改变 [`dev` 配置文件](../reference/profiles.md#dev) 以使用 [Cranelift 代码生成后端](https://github.com/rust-lang/rustc_codegen_cranelift) 来生成机器代码，而不是默认的 LLVM 后端。Cranelift 后端应该比 LLVM 更快地生成代码，这应该会提高构建性能。

权衡：
- ✅ 更快的代码生成（`cargo build`）
- ❌ **需要使用 nightly Rust 和一个[不稳定的 Cargo 功能][codegen-backend-feature]**
- ❌ 生成的代码运行时性能更差
  - 加快了 `cargo test` 的构建部分，但可能会增加其测试执行部分
- ❌ 仅适用于[特定目标平台](https://github.com/rust-lang/rustc_codegen_cranelift?tab=readme-ov-file#platform-support)
- ❌ 可能不支持所有 Rust 功能（例如，栈展开）

[codegen-backend-feature]: ../reference/unstable.md#codegen-backend

### 启用实验性的并行前端 {#enable-the-experimental-parallel-frontend}

建议：添加到您的 `.cargo/config.toml`：

```toml
[build]
rustflags = "-Zthreads=8"
```

这个 [`rustflags`][build.rustflags] 将启用 Rust 编译器的[并行前端][parallel-frontend-blog]，并告诉它使用 `n` 个线程。`n` 的值应根据您系统上可用的核心数来选择，尽管收益会递减。我们建议最多使用 `8` 个线程。

权衡：
- ✅ 更快的构建时间（包括 `cargo check` 和 `cargo build`）
- ❌ **需要使用 nightly Rust 和一个[不稳定的 Rust 功能][parallel-frontend-issue]**

[parallel-frontend-blog]: https://blog.rust-lang.org/2023/11/09/parallel-rustc/
[parallel-frontend-issue]: https://github.com/rust-lang/rust/issues/113349
[build.rustflags]: ../reference/config.md#buildrustflags

### 使用替代的链接器 {#use-an-alternative-linker}

考虑：安装并配置替代链接器，例如 [LLD](https://lld.llvm.org/)、[mold](https://github.com/rui314/mold) 或 [wild](https://github.com/davidlattimore/wild)。例如，要在 Linux 上配置 mold，您可以添加到您的 `.cargo/config.toml`：

```toml
[target.'cfg(target_os = "linux")']
# mold，如果您有 GCC 12+
rustflags = ["-C", "link-arg=-fuse-ld=mold"]

# mold，否则
linker = "clang"
rustflags = ["-C", "link-arg=-fuse-ld=/path/to/mold"]
```

虽然依赖项可以并行构建，但链接所有依赖项发生在构建结束时，这可能会使链接时间占主导地位，尤其是对于增量重建。通常，Rust 使用的链接器已经相当快，更换带来的收益可能不值得，但并非总是如此。例如，除了 `x86_64-unknown-linux-gnu` 之外的 Linux 目标仍然使用相当慢的 Linux 系统链接器（有关更多详细信息，请参见 [rust#39915](https://github.com/rust-lang/rust/issues/39915)）。

权衡：
- ✅ 更快的链接时间
- ❌ 可能不支持所有用例，特别是如果您依赖 C 或 C++ 依赖项

### 为整个工作空间解析功能 {#resolve-features-for-the-whole-workspace}

考虑：添加到您项目的 `.cargo/config.toml`

```toml
[resolver]
feature-unification = "workspace"
```

当调用 `cargo` 时，[功能会根据您选择的工作空间成员被激活][resolver-features]。然而，当为应用程序做出贡献时，您可能需要构建和测试应用程序中的各种包，这可能导致额外的重建，因为为公共依赖项可能激活了不同的功能集。使用 [`feature-unification`][feature-unification]，您可以通过确保激活相同的依赖项功能集来重用更多的依赖项构建，而独立于当前正在构建和测试的包。

权衡：
- ✅ 在构建工作空间中的不同包时，减少重建次数
- ❌ **需要使用 nightly Rust 和一个[不稳定的 Cargo 功能][feature-unification]**
- ❌ 一个包激活一个功能可能会掩盖其他应该激活但未激活该功能的包中的错误
- ❌ 如果 `--workspace` 的功能统一对您不起作用，那么这个也不会起作用

[resolver-features]: ../reference/resolver.md#features
[feature-unification]: ../reference/unstable.md#feature-unification

## 减少构建的代码 {#reducing-built-code}

### 移除未使用的依赖项 {#removing-unused-dependencies}

建议：定期使用第三方工具（如 [cargo-machete](https://crates.io/crates/cargo-machete)、[cargo-udeps](https://crates.io/crates/cargo-udeps)、[cargo-shear](https://crates.io/crates/cargo-shear)）审查未使用的依赖项以进行移除。

更改代码时，很容易忽略某个依赖项不再使用，可以将其移除。

> **注意：** Cargo 中对这一功能的原生支持正在 [#15813](https://github.com/rust-lang/cargo/issues/15813) 中进行跟踪。

权衡：
- ✅ 更快的完整构建和链接时间
- ❌ 可能错误地将依赖项标记为未使用或遗漏某些依赖项

### 移除依赖项中未使用的功能 {#removing-unused-features-from-dependencies}

建议：定期使用第三方工具（如 [cargo-features-manager](https://crates.io/crates/cargo-features-manager)、[cargo-unused-features](https://crates.io/crates/cargo-unused-features)）审查依赖项中未使用的功能以进行移除。

更改代码时，很容易忽略某个依赖项的功能不再使用，可以将其移除。这可以减少正在构建的传递依赖项的数量，或者减少正在构建的 crate 内的代码量。移除功能时需要格外小心，因为功能也可能用于期望的行为或性能更改，这些可能并不总是从编译或测试中显而易见。

权衡：
- ✅ 更快的完整构建和链接时间
- ❌ 可能错误地将功能标记为未使用