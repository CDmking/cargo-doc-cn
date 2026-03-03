# 源替换 {#source-replacement}

本文档是关于将[注册中心]或[基于 git 的依赖项]的仓库通信重定向到另一个数据源，例如镜像原始注册中心的服务器或精确的本地副本。

如果您想修补单个依赖项，请参阅本文档的[覆盖依赖项]部分。如果您想控制 Cargo 如何发出网络请求，请参阅 [`[http]`](config.md#http) 和 [`[net]`](config.md#net) 配置。

*源*是提供可能作为包依赖项包含的 crate 的提供者。Cargo 支持**用一个源替换另一个源**的能力来表达以下策略：

* 供应（Vendoring）——可以定义表示本地文件系统上 crate 的自定义源。这些源是它们所替换的源的一个子集，如果需要可以检入到包中。

* 镜像（Mirroring）——源可以被替换为充当 crates.io 本身缓存的等效版本。

Cargo 有一个关于源替换的核心假设，即两个源的源代码完全相同。请注意，这也意味着不允许替代源中出现原始源中不存在的 crate。

因此，源替换不适用于修补依赖项或私有注册中心等场景。Cargo 通过使用 [`[patch]` 键][覆盖依赖项]支持修补依赖项，私有注册中心支持在[注册中心章节][注册中心]中描述。

使用源替换时，运行需要直接联系注册中心的命令[^1]需要传递 `--registry` 选项。这有助于避免关于联系哪个注册中心的任何歧义，并将使用指定注册中心的认证令牌。

[^1]: 此类命令的示例在[发布命令]中。

[发布命令]: ../commands/publishing-commands.md
[覆盖依赖项]: overriding-dependencies.md
[注册中心]: registries.md

## 配置 {#configuration}

替换源的配置通过 [`.cargo/config.toml`][配置] 完成，所有可用键的完整集合如下：

```toml
# `source` 表是存储所有与源替换相关的键的地方。
[source]

# 在 `source` 表下是许多其他表，这些表的键是相关源的名称。例如，此部分定义了一个新源，称为 `my-vendor-source`，它来自包含此 `.cargo/config.toml` 文件的目录的相对路径 `vendor`
[source.my-vendor-source]
directory = "vendor"

# crate 的默认 crates.io 源在名称 "crates-io" 下可用，这里我们使用 `replace-with` 键来指示它被上面的源替换。
#
# `replace-with` 键也可以引用在 `[registries]` 表中定义的替代注册中心名称。
[source.crates-io]
replace-with = "my-vendor-source"

# 每个源都有自己的表，键是源的名称
[source.the-source-name]

# 指示 `the-source-name` 将被其他地方定义的 `another-source` 替换
replace-with = "another-source"

# 可以指定几种源（下面将详细描述）：
registry = "https://example.com/path/to/index"
local-registry = "path/to/registry"
directory = "path/to/vendor"

# Git 源可以可选地指定分支/标签/修订版本
git = "https://example.com/path/to/repo"
# branch = "master"
# tag = "v1.0.1"
# rev = "313f44e8"
```

[配置]: config.md

## 注册中心源 {#registry-sources}

“注册中心源”是指像 crates.io 本身一样工作的源。它是一个符合 https://doc.rust-lang.org/cargo/reference/registry-index.html 规范的索引，并具有指示从哪里下载 crate 的配置文件。

注册中心源可以使用 [git 或稀疏 HTTP 协议][协议]：

```toml
# Git 协议
registry = "ssh://git@example.com/path/to/index.git"

# 稀疏 HTTP 协议
registry = "sparse+https://example.com/path/to/index"

# HTTPS git 协议
registry = "https://example.com/path/to/index"
```

[协议]: registries.md#registry-protocols

[crates.io 索引]: https://doc.rust-lang.org/cargo/reference/registry-index.html

## 本地注册中心源 {#local-registry-sources}

“本地注册中心源”旨在成为另一个注册中心源的子集，但可在本地文件系统上使用（也称为供应）。本地注册中心是预先下载的，通常与 `Cargo.lock` 同步，并包含一组 `*.crate` 文件和一个类似于普通注册中心的索引。

管理和创建本地注册中心源的主要方式是通过 [`cargo-local-registry`][cargo-local-registry] 子命令，[可在 crates.io 上获取][cargo-local-registry]，并可以通过 `cargo install cargo-local-registry` 安装。

[cargo-local-registry]: https://crates.io/crates/cargo-local-registry

本地注册中心包含在一个目录中，包含许多从 crates.io 下载的 `*.crate` 文件以及一个 `index` 目录，其格式与 crates.io-index 项目相同（仅填充了存在的 crate 的条目）。

## 目录源 {#directory-sources}

“目录源”类似于本地注册中心源，其中包含本地文件系统上可用的许多 crate，适合供应依赖项。目录源主要由 `cargo vendor` 子命令管理。

但目录源与本地注册中心不同，它们包含 `*.crate` 文件的解压版本，使得在某些情况下更适合将一切检入源代码控制。目录源只是一个包含许多其他目录的目录，这些目录包含 crate 的源代码（`*.crate` 文件的解压版本）。目前对每个目录的名称没有限制。

目录源中的每个 crate 还有一个关联的元数据文件，指示每个文件的校验和，以防止意外修改。

## Git 源 {#git-sources}

Git 源表示[基于 git 的依赖项]使用的仓库。它们用于指定哪些基于 git 的依赖项应该被替代源替换。

Git 源与 [git 注册中心][协议]无关，不能用于替换注册中心源。

[基于 git 的依赖项]: specifying-dependencies.md#specifying-dependencies-from-git-repositories