# 词汇表

## Artifact

*产物* 是指编译过程创建的文件或文件集。这包括可链接的库、可执行的二进制文件以及生成的文档。

## Cargo

*Cargo* 是 Rust 的[*包管理器*](#package-manager)，也是本书的主要主题。

## Cargo.lock

参见 [*lock file*](#lock-file)（锁定文件）。

## Cargo.toml

参见 [*manifest*](#manifest)（清单）。

## Crate

Rust 的 *crate* 指一个库或一个可执行程序，分别称为*库 crate* 或*二进制 crate*。

为 Cargo [package](#package) 定义的每个 [target](#target) 都是一个 *crate*。

通俗地说，术语 *crate* 可以指目标的源代码，也可以指目标编译后产生的产物。它还可以指从 [registry](#registry)（注册中心）获取的压缩包。

给定 crate 的源代码可以细分为 [*modules*](#module)（模块）。

## Edition

*Rust 版本* 是 Rust 语言发展的里程碑。[包的版本][edition-field]在 `Cargo.toml` [manifest](#manifest)（清单）中指定，各个目标可以指定它们使用的版本。更多信息请参阅 [Edition Guide]。

## Feature

*feature* 的含义取决于上下文：

- [*feature*][feature] 是一个允许条件编译的命名标志。一个 feature 可以引用一个可选的依赖项，也可以引用在 `Cargo.toml` [manifest](#manifest) 中定义的任意名称，该名称可以在源代码中检查。

- Cargo 有 [*unstable feature flags*][cargo-unstable]（不稳定功能标志），可用于启用 Cargo 本身的实验性行为。

- Rust 编译器和 Rustdoc 有它们自己的不稳定功能标志（参见 [The Unstable Book][unstable-book] 和 [The Rustdoc Book][rustdoc-unstable]）。

- CPU 目标有 [*target features*][target-feature]（目标功能），用于指定 CPU 的能力。

## Index

*index* 是 [*registry*](#registry) 中可搜索的 [*crates*](#crate) 列表。

## Lock file

`Cargo.lock` *lock file*（锁定文件）是一个文件，用于记录 [*workspace*](#workspace) 或 [*package*](#package) 中使用的每个依赖项的确切版本。它由 Cargo 自动生成。参见 [Cargo.toml vs Cargo.lock]。

## Manifest

[*manifest*][manifest] 是在名为 `Cargo.toml` 的文件中对 [package](#package) 或 [workspace](#workspace) 的描述。

[*virtual manifest*][virtual]（虚拟清单）是一个仅描述工作空间而不包含包的 `Cargo.toml` 文件。

## Member

*member* 是属于 [*workspace*](#workspace) 的 [*package*](#package)。

## Module

Rust 的模块系统用于将代码组织成称为 *modules* 的逻辑单元，它们在代码中提供隔离的命名空间。

给定 [crate](#crate) 的源代码可以细分为一个或多个单独的模块。这样做通常是为了将代码按相关功能区域组织，或者控制源代码中符号（结构体、函数等）的可见范围（公有/私有）。

[`Cargo.toml`](#manifest) 文件主要关注其定义的 [package](#package)、其 crate 以及它们所依赖的包的 crate。尽管如此，在使用 Rust 时，您会经常看到“模块”这个术语，因此您应该了解它与给定 crate 的关系。

## Package

*package* 是源文件和描述包的 `Cargo.toml` [*manifest*](#manifest) 文件的集合。包具有名称和版本，用于指定包之间的依赖关系。

一个包包含多个 [*targets*](#target)（目标），每个目标都是一个 [*crate*](#crate)。`Cargo.toml` 文件描述了包中 crate 的类型（二进制或库），以及每个 crate 的一些元数据——每个 crate 如何构建、它们的直接依赖是什么等，如本书各处所述。

*package root*（包根目录）是包的 `Cargo.toml` 清单所在的目录。（与 [*workspace root*](#workspace)（工作空间根目录）对比。）

[*package ID specification*][pkgid-spec]（包 ID 规范），或称 *SPEC*，是一个用于唯一标识来自特定源的包的特定版本的字符串。

中小型 Rust 项目通常只需要一个包，但它们通常有多个 crate。

大型项目可能涉及多个包，在这种情况下，可以使用 Cargo [*workspaces*](#workspace)（工作空间）来管理包之间的公共依赖项和其他相关元数据。

## Package manager

广义上说，*package manager*（包管理器）是软件生态系统中一个自动化获取、安装和升级产物过程的程序（或相关程序的集合）。在编程语言生态系统中，包管理器是一个面向开发者的工具，其主要功能是从某个中央仓库下载库产物及其依赖项；此功能通常与执行软件构建的能力（通过调用特定语言的编译器）相结合。

[*Cargo*](#cargo) 是 Rust 生态系统中的包管理器。Cargo 下载您的 Rust [package](#package) 的依赖项（称为 [*crates*](#crate) 的 [*artifacts*](#artifact)），编译您的包，制作可分发的包，并（可选地）将它们上传到 Rust 社区的 [*package registry*](#registry)（包注册中心）[crates.io][]。

## Package registry

参见 [*registry*](#registry)。

## Project

[package](#package)（包）的另一种说法。

## Registry

*registry* 是一项服务，包含一系列可下载的 [*crates*](#crate)，这些 crate 可以被安装或用作 [*package*](#package) 的依赖项。Rust 生态系统中的默认注册中心是 [crates.io](https://crates.io)。注册中心有一个 [*index*](#index)（索引），其中包含所有 crate 的列表，并告诉 Cargo 如何下载所需的 crate。

## Source

*source* 是一个提供者，包含可以作为 [*package*](#package) 依赖项包括在内的 [*crates*](#crate)。有几种类型的 source：

- **Registry source**（注册中心源）--- 参见 [registry](#registry)。
- **Local registry source**（本地注册中心源）--- 一组以压缩文件形式存储在文件系统上的 crate。参见 [Local Registry Sources]。
- **Directory source**（目录源）--- 一组以未压缩文件形式存储在文件系统上的 crate。参见 [Directory Sources]。
- **Path source**（路径源）--- 位于文件系统上的单个包（例如 [path dependency]（路径依赖））或多个包的集合（例如 [path overrides]（路径覆盖））。
- **Git source**（Git 源）--- 位于 git 仓库中的包（例如 [git dependency]（Git 依赖）或 [git source]（Git 源））。

更多信息请参见 [Source Replacement]。

## Spec

参见 [package ID specification](#package)（包 ID 规范）。

## Target

术语 *target* 的含义取决于上下文：

- **Cargo Target**（Cargo 目标）--- Cargo [*packages*](#package) 由 *targets* 组成，这些 targets 对应于将产生的 [*artifacts*](#artifact)。包可以有库、二进制、示例、测试和基准目标。 [targets 列表][targets] 在 `Cargo.toml` [*manifest*](#manifest) 中配置，通常根据源代码的 [directory layout] 自动推断。
- **Target Directory**（目标目录）--- Cargo 将构建的产物放置在 *target* 目录中。默认情况下，这是一个位于 [*workspace*](#workspace) 根目录下名为 `target` 的目录，如果不使用工作空间，则在包根目录下。可以使用 `--target-dir` 命令行选项、`CARGO_TARGET_DIR` [环境变量] 或 `build.target-dir` [配置选项] 更改目录。更多信息请参阅 [build cache] 文档。
- **Target Architecture**（目标架构）--- 构建产物的操作系统和机器架构通常称为 *target*。
- **Target Triple**（目标三元组）--- 三元组是指定目标架构的一种特定格式。三元组可以称为 *target triple*，即产物产生的架构，以及 *host triple*，即编译器运行的架构。目标三元组可以通过 `--target` 命令行选项或 `build.target` [配置选项] 指定。三元组的一般格式是 `<arch><sub>-<vendor>-<sys>-<abi>`，其中：

  - `arch` = 基本 CPU 架构，例如 `x86_64`、`i686`、`arm`、`thumb`、`mips` 等。
  - `sub` = CPU 子架构，例如 `arm` 有 `v7`、`v7s`、`v5te` 等。
  - `vendor` = 供应商，例如 `unknown`、`apple`、`pc`、`nvidia` 等。
  - `sys` = 系统名称，例如 `linux`、`windows`、`darwin` 等。`none` 通常用于没有操作系统的裸机。
  - `abi` = ABI，例如 `gnu`、`android`、`eabi` 等。

  某些参数可以省略。运行 `rustc --print target-list` 以获取支持的目标列表。

## Test Targets

Cargo *test targets*（测试目标）生成有助于验证代码正常运行和正确性的二进制文件。有两种类型的测试产物：

* **Unit test**（单元测试）--- *unit test* 是直接从库或二进制目标编译的可执行二进制文件。它包含库或二进制代码的全部内容，并运行 `#[test]` 注解的函数，旨在验证代码的各个单元。
* **Integration test target**（集成测试目标）--- [*integration test target*][integration-tests] 是从 *test target* 编译的可执行二进制文件，该测试目标是一个独立的 [*crate*](#crate)，其源代码位于 `tests` 目录中，或由 `Cargo.toml` [*manifest*](#manifest) 中的 [`[[test]]` table][targets] 指定。它旨在仅测试库的公共 API，或执行二进制文件以验证其操作。

## Workspace

[*workspace*][workspace] 是一个或多个 [*packages*](#package) 的集合，它们共享共同的依赖解析（使用共享的 `Cargo.lock` [*lock file*](#lock-file)）、输出目录和各种设置，例如配置文件。

[*virtual workspace*][virtual]（虚拟工作空间）是一个其根 `Cargo.toml` [*manifest*](#manifest) 不定义包，仅列出工作空间 [*members*](#member) 的工作空间。

*workspace root*（工作空间根目录）是工作空间的 `Cargo.toml` 清单所在的目录。（与 [*package root*](#package) 对比。）

[Cargo.toml vs Cargo.lock]: ../guide/cargo-toml-vs-cargo-lock.md
[Directory Sources]: ../reference/source-replacement.md#directory-sources
[Local Registry Sources]: ../reference/source-replacement.md#local-registry-sources
[Source Replacement]: ../reference/source-replacement.md
[build cache]: ../reference/build-cache.html
[cargo-unstable]: ../reference/unstable.md
[config option]: ../reference/config.md
[crates.io]: https://crates.io/
[directory layout]: ../guide/project-layout.md
[edition guide]: ../../edition-guide/index.html
[edition-field]: ../reference/manifest.md#the-edition-field
[environment variable]: ../reference/environment-variables.md
[feature]: ../reference/features.md
[git dependency]: ../reference/specifying-dependencies.md#specifying-dependencies-from-git-repositories
[git source]: ../reference/source-replacement.md
[integration-tests]: ../reference/cargo-targets.md#integration-tests
[manifest]: ../reference/manifest.md
[path dependency]: ../reference/specifying-dependencies.md#specifying-path-dependencies
[path overrides]: ../reference/overriding-dependencies.md#paths-overrides
[pkgid-spec]: ../reference/pkgid-spec.md
[rustdoc-unstable]: https://doc.rust-lang.org/nightly/rustdoc/unstable-features.html
[target-feature]: ../../reference/attributes/codegen.html#the-target_feature-attribute
[targets]: ../reference/cargo-targets.md#configuring-a-target
[unstable-book]: https://doc.rust-lang.org/nightly/unstable-book/index.html
[virtual]: ../reference/workspaces.md
[workspace]: ../reference/workspaces.md