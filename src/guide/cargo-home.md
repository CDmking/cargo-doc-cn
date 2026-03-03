# Cargo 主目录 {#cargo-home}

"Cargo 主目录" 功能类似于下载和源代码缓存。在构建 [crate][def-crate] 时，Cargo 会将已下载的构建依赖存储在 Cargo 主目录中。您可以通过设置 `CARGO_HOME` [环境变量][env] 来更改 Cargo 主目录的位置。[home](https://crates.io/crates/home) crate 提供了一个 API，如果您需要在 Rust crate 中获取此位置信息，可以使用它。默认情况下，Cargo 主目录位于 `$HOME/.cargo/`。

请注意，Cargo 主目录的内部结构并未稳定，可能随时更改。

Cargo 主目录由以下组件组成：

## 文件： {#files}

* `config.toml`
	Cargo 的全局配置文件，请参阅 [配置参考][config]。

* `credentials.toml`
	来自 [`cargo login`] 的私有登录凭据，用于登录到[注册表][def-registry]。

* `.crates.toml`, `.crates2.json`
	这些隐藏文件包含通过 [`cargo install`] 安装的 crate 的[包][def-package]信息。请勿手动编辑！

## 目录： {#directories}

* `bin`
	`bin` 目录包含通过 [`cargo install`] 或 [`rustup`](https://rust-lang.github.io/rustup/) 安装的 crate 的可执行文件。为了使这些二进制文件可访问，请将此目录的路径添加到您的 `$PATH` 环境变量中。

* `git`
	Git 源代码存储在此处：

  * `git/db`
		当 crate 依赖于 Git 仓库时，Cargo 会将仓库作为裸仓库克隆到此目录，并在必要时更新它。

  * `git/checkouts`
		如果使用了 Git 源代码，则所需提交的仓库将从 `git/db` 中的裸仓库检出到此目录。这为编译器提供了该依赖项指定提交的仓库中实际包含的文件。同一仓库的不同提交的多次检出是可能的。

* `registry`
	注册表（例如 [crates.io](https://crates.io/)）的包和元数据位于此处。

  * `registry/index`
		索引是一个裸 Git 仓库，其中包含注册表中所有可用 crate 的元数据（版本、依赖项等）。

  * `registry/cache`
		已下载的依赖项存储在缓存中。这些 crate 是以 `.crate` 扩展名命名的压缩 gzip 存档。

  * `registry/src`
		如果包需要已下载的 `.crate` 存档，则会将其解压到 `registry/src` 文件夹中，rustc 将在该文件夹中找到 `.rs` 文件。

## 在 CI 中缓存 Cargo 主目录 {#caching-the-cargo-home-in-ci}

为了避免在持续集成期间重新下载所有 crate 依赖项，您可以缓存 `$CARGO_HOME` 目录。但是，缓存整个目录通常效率低下，因为它会重复存储已下载的源代码。例如，如果我们依赖一个像 `serde 1.0.92` 这样的 crate，并缓存整个 `$CARGO_HOME`，实际上我们会缓存源代码两次：一次在 `registry/cache` 中的 `serde-1.0.92.crate`，另一次在 `registry/src` 中解压的 serde 的 `.rs` 文件。这可能会不必要地减慢构建速度，因为下载、解压、重新压缩和重新上传缓存到 CI 服务器可能需要一些时间。

如果您希望缓存通过 [`cargo install`] 安装的二进制文件，则需要缓存 `bin/` 文件夹以及 `.crates.toml` 和 `.crates2.json` 文件。

跨构建缓存以下文件和目录应该足够了：

* `.crates.toml`
* `.crates2.json`
* `bin/`
* `registry/index/`
* `registry/cache/`
* `git/db/`

## 项目的依赖项本地化 {#vendoring-all-dependencies-of-a-project}

请参阅 [`cargo vendor`] 子命令。

## 清除缓存 {#clearing-the-cache}

理论上，您可以随时删除缓存的任何部分，Cargo 会尽力恢复 crate 所需的源代码，无论是通过重新提取存档、从裸仓库检出，还是直接从网络重新下载。

另外，[cargo-cache](https://crates.io/crates/cargo-cache) crate 提供了一个简单的 CLI 工具，可以仅清除缓存中选定的部分，或者在命令行中显示其组件的大小。

[`cargo install`]: ../commands/cargo-install.md
[`cargo login`]: ../commands/cargo-login.md
[`cargo vendor`]: ../commands/cargo-vendor.md
[config]: ../reference/config.md
[def-crate]: ../appendix/glossary.md#crate '"crate" (glossary entry)'
[def-package]: ../appendix/glossary.md#package '"package" (glossary entry)'
[def-registry]: ../appendix/glossary.md#registry '"registry" (glossary entry)'
[env]: ../reference/environment-variables.md