# 清单格式 {#the-manifest-format}

每个包的 `Cargo.toml` 文件称为其*清单*。它以 [TOML] 格式编写。它包含编译包所需的元数据。有关 cargo 如何查找清单文件的更多详细信息，请查看 `cargo locate-project` 部分。

每个清单文件由以下部分组成：

* [`cargo-features`](unstable.md) --- 不稳定的、仅限 nightly 的功能。
* [`[package]`](#the-package-section) --- 定义一个包。
  * [`name`](#the-name-field) --- 包的名称。
  * [`version`](#the-version-field) --- 包的版本。
  * [`authors`](#the-authors-field) --- 包的作者。
  * [`edition`](#the-edition-field) --- Rust 版本。
  * [`rust-version`](rust-version.md) --- 最低支持的 Rust 版本。
  * [`description`](#the-description-field) --- 包的描述。
  * [`documentation`](#the-documentation-field) --- 包文档的 URL。
  * [`readme`](#the-readme-field) --- 包 README 文件的路径。
  * [`homepage`](#the-homepage-field) --- 包主页的 URL。
  * [`repository`](#the-repository-field) --- 包源代码仓库的 URL。
  * [`license`](#the-license-and-license-file-fields) --- 包的许可证。
  * [`license-file`](#the-license-and-license-file-fields) --- 许可证文本的路径。
  * [`keywords`](#the-keywords-field) --- 包的关键词。
  * [`categories`](#the-categories-field) --- 包的类别。
  * [`workspace`](#the-workspace-field) --- 包的工作空间路径。
  * [`build`](#the-build-field) --- 包构建脚本的路径。
  * [`links`](#the-links-field) --- 包链接的本地库的名称。
  * [`exclude`](#the-exclude-and-include-fields) --- 发布时要排除的文件。
  * [`include`](#the-exclude-and-include-fields) --- 发布时要包含的文件。
  * [`publish`](#the-publish-field) --- 可用于防止发布包。
  * [`metadata`](#the-metadata-table) --- 外部工具的额外设置。
  * [`default-run`](#the-default-run-field) --- [`cargo run`] 默认运行的二进制文件。
  * [`autolib`](cargo-targets.md#target-auto-discovery) --- 禁用库的自动发现。
  * [`autobins`](cargo-targets.md#target-auto-discovery) --- 禁用二进制文件的自动发现。
  * [`autoexamples`](cargo-targets.md#target-auto-discovery) --- 禁用示例的自动发现。
  * [`autotests`](cargo-targets.md#target-auto-discovery) --- 禁用测试的自动发现。
  * [`autobenches`](cargo-targets.md#target-auto-discovery) --- 禁用基准测试的自动发现。
  * [`resolver`](resolver.md#resolver-versions) --- 设置要使用的依赖解析器。
* 目标表：（设置请参见[配置](cargo-targets.md#configuring-a-target)）
  * [`[lib]`](cargo-targets.md#library) --- 库目标设置。
  * [`[[bin]]`](cargo-targets.md#binaries) --- 二进制目标设置。
  * [`[[example]]`](cargo-targets.md#examples) --- 示例目标设置。
  * [`[[test]]`](cargo-targets.md#tests) --- 测试目标设置。
  * [`[[bench]]`](cargo-targets.md#benchmarks) --- 基准测试目标设置。
* 依赖表：
  * [`[dependencies]`](specifying-dependencies.md) --- 包库依赖。
  * [`[dev-dependencies]`](specifying-dependencies.md#development-dependencies) --- 示例、测试和基准测试的依赖。
  * [`[build-dependencies]`](specifying-dependencies.md#build-dependencies) --- 构建脚本的依赖。
  * [`[target]`](specifying-dependencies.md#platform-specific-dependencies) --- 平台特定依赖。
* [`[badges]`](#the-badges-section) --- 要在注册表上显示的徽章。
* [`[features]`](features.md) --- 条件编译功能。
* [`[lints]`](#the-lints-section) --- 为此包配置代码检查器。
* [`[hints]`](#the-hints-section) --- 提供编译此包的提示。
* [`[patch]`](overriding-dependencies.md#the-patch-section) --- 覆盖依赖。
* [`[replace]`](overriding-dependencies.md#the-replace-section) --- 覆盖依赖（已弃用）。
* [`[profile]`](profiles.md) --- 编译器设置和优化。
* [`[workspace]`](workspaces.md) --- 工作空间定义。

## `[package]` 部分 {#the-package-section}

`Cargo.toml` 中的第一部分是 `[package]`。

```toml
[package]
name = "hello_world" # 包的名称
version = "0.1.0"    # 当前版本，遵循语义化版本规范
```

Cargo 唯一必需的字段是 [`name`](#the-name-field)。如果发布到注册表，注册表可能需要额外字段。有关发布到 [crates.io] 的要求，请参见下面的注释和[发布章节][publishing]。

### `name` 字段 {#the-name-field}

包名称是用于引用包的标识符。当在另一个包中列为依赖项时使用，并作为推断的库和二进制目标的默认名称。

名称只能使用[字母数字]字符或 `-` 或 `_`，且不能为空。

请注意，[`cargo new`] 和 [`cargo init`] 对包名称施加了一些额外的限制，例如强制其是有效的 Rust 标识符而不是关键字。[crates.io] 施加了更多限制，例如：

- 只允许 ASCII 字符。
- 不使用保留名称。
- 不使用特殊的 Windows 名称，如 "nul"。
- 长度最多为 64 个字符。

[字母数字]: ../../std/primitive.char.html#method.is_alphanumeric

### `version` 字段 {#the-version-field}

`version` 字段根据 [SemVer] 规范格式化：

版本必须包含三个数字部分：主版本、次版本和修订版本。

可以在短划线后添加预发布部分，例如 `1.0.0-alpha`。预发布部分可以用句点分隔以区分不同的组件。数字组件将使用数字比较，而其他所有内容将按字典顺序比较。例如，`1.0.0-alpha.11` 高于 `1.0.0-alpha.4`。

可以在加号后添加元数据部分，例如 `1.0.0+21AF26D3`。这仅用于信息目的，通常被 Cargo 忽略。

Cargo 内置了[语义化版本控制](https://semver.org/)的概念，因此如果它们最左边的非零主/次/修订版本组件相同，则版本被视为[兼容](semver.md)。有关 Cargo 如何使用版本解析依赖项的更多信息，请参见[解析器]章节。

此字段是可选的，默认为 `0.0.0`。发布包时需要此字段。

> **MSRV：** 在 1.75 之前，此字段是必需的

[SemVer]: https://semver.org
[解析器]: resolver.md
[语义化版本兼容性]: semver.md

### `authors` 字段 {#the-authors-field}

> **警告：** 此字段已弃用

可选的 `authors` 字段以数组形式列出了被视为包"作者"的个人或组织。每个作者条目末尾可以在尖括号内包含一个可选的电子邮件地址。

```toml
[package]
# ...
authors = ["Graydon Hoare", "Fnu Lnu <no-reply@rust-lang.org>"]
```

此字段在包元数据中体现，并且为了向后兼容，在 `build.rs` 内的 `CARGO_PKG_AUTHORS` 环境变量中体现。

### `edition` 字段 {#the-edition-field}

`edition` 键是一个可选键，它影响您的包使用哪个 [Rust 版本] 进行编译。在 `[package]` 中设置 `edition` 键将影响包中的所有目标/crate，包括测试套件、基准测试、二进制文件、示例等。

```toml
[package]
# ...
edition = '2024'
```

大多数清单的 `edition` 字段由 [`cargo new`] 自动填充为最新的稳定版本。目前，默认情况下 `cargo new` 会创建带有 2024 版本的清单。

如果 `Cargo.toml` 中没有 `edition` 字段，则为了向后兼容，假定为 2015 版本。请注意，所有用 [`cargo new`] 创建的清单都不会使用这个历史回退，因为它们会将 `edition` 显式指定为较新的值。

### `rust-version` 字段 {#the-rust-version-field}

`rust-version` 字段告诉 cargo 您的包支持的 Rust 工具链版本。更多详细信息请参见 [Rust 版本章节](rust-version.md)。

### `description` 字段 {#the-description-field}

描述是关于包的简短介绍。[crates.io] 会将其与您的包一起显示。这应该是纯文本（不是 Markdown）。

```toml
[package]
# ...
description = "关于我的包的简短描述"
```

> **注意：** [crates.io] 要求设置 `description`。

### `documentation` 字段 {#the-documentation-field}

`documentation` 字段指定托管 crate 文档的网站的 URL。如果清单文件中未指定 URL，[crates.io] 会在文档已构建且可用时自动将您的 crate 链接到相应的 [docs.rs] 页面（请参见 [docs.rs 队列]）。

```toml
[package]
# ...
documentation = "https://docs.rs/bitflags"
```

[docs.rs 队列]: https://docs.rs/releases/queue

### `readme` 字段 {#the-readme-field}

`readme` 字段应该是包根目录下（相对于此 `Cargo.toml`）包含包一般信息的文件路径。发布时，此文件将传输到注册表。[crates.io] 会将其解释为 Markdown 并在 crate 页面上呈现。

```toml
[package]
# ...
readme = "README.md"
```

如果未指定此字段的值，并且包根目录中存在名为 `README.md`、`README.txt` 或 `README` 的文件，则将使用该文件名。您可以通过将此字段设置为 `false` 来抑制此行为。如果字段设置为 `true`，则假定默认值为 `README.md`。

### `homepage` 字段 {#the-homepage-field}

`homepage` 字段应该是您的包主页网站的 URL。

```toml
[package]
# ...
homepage = "https://serde.rs"
```

只有当 crate 有专门的网站而不只是源代码仓库或 API 文档时，才应设置 `homepage` 的值。请不要让 `homepage` 与 `documentation` 或 `repository` 值重复。

### `repository` 字段 {#the-repository-field}

`repository` 字段应该是您的包源代码仓库的 URL。

```toml
[package]
# ...
repository = "https://github.com/rust-lang/cargo"
```

### `license` 和 `license-file` 字段 {#the-license-and-license-file-fields}

`license` 字段包含包发布的软件许可证名称。`license-file` 字段包含许可证文本文件的路径（相对于此 `Cargo.toml`）。

[crates.io] 将 `license` 字段解释为 [SPDX 2.3 许可证表达式][spdx-2.3-license-expressions]。名称必须是 [SPDX 许可证列表 3.20][spdx-license-list-3.20] 中的已知许可证。更多信息请参见 [SPDX 网站]。

SPDX 许可证表达式支持 AND 和 OR 运算符来组合多个许可证。[^斜杠]

```toml
[package]
# ...
license = "MIT OR Apache-2.0"
```

使用 `OR` 表示用户可以选择任一许可证。使用 `AND` 表示用户必须同时遵守两个许可证。`WITH` 运算符表示带有特殊例外的许可证。一些示例：

* `MIT OR Apache-2.0`
* `LGPL-2.1-only AND MIT AND BSD-2-Clause`
* `GPL-2.0-or-later WITH Bison-exception-2.2`

如果包使用非标准许可证，则可以指定 `license-file` 字段来代替 `license` 字段。

```toml
[package]
# ...
license-file = "LICENSE.txt"
```

> **注意：** [crates.io] 要求设置 `license` 或 `license-file`。

[^斜杠]: 以前多个许可证可以用 `/` 分隔，但该用法已弃用。

### `keywords` 字段 {#the-keywords-field}

`keywords` 字段是描述此包的字符串数组。这有助于在注册表上搜索包时，您可以选择任何有助于他人找到此 crate 的词语。

```toml
[package]
# ...
keywords = ["gamedev", "graphics"]
```

> **注意：** [crates.io] 最多允许 5 个关键词。每个关键词必须是 ASCII 文本，最多 20 个字符，以字母数字字符开头，并且只包含字母、数字、`_`、`-` 或 `+`。

### `categories` 字段 {#the-categories-field}

`categories` 字段是此包所属类别的字符串数组。

```toml
categories = ["command-line-utilities", "development-tools::cargo-plugins"]
```

> **注意：** [crates.io] 最多有 5 个类别。每个类别应匹配 <https://crates.io/category_slugs> 上可用的字符串之一，并且必须完全匹配。

### `workspace` 字段 {#the-workspace-field}

`workspace` 字段可用于配置此包将加入的工作空间。如果未指定，则将推断为文件系统中向上的第一个带有 `[workspace]` 的 Cargo.toml。如果成员不在工作空间根目录的子目录中，则设置此字段很有用。

```toml
[package]
# ...
workspace = "path/to/workspace/root"
```

如果清单中已经定义了一个 `[workspace]` 表，则不能指定此字段。也就是说，一个 crate 不能既是工作空间中的根 crate（包含 `[workspace]`），又是另一个工作空间的成员 crate（包含 `package.workspace`）。

更多信息请参见[工作空间章节](workspaces.md)。

### `build` 字段 {#the-build-field}

`build` 字段指定包根目录中的一个文件，该文件是用于构建本地代码的[构建脚本]。更多信息可以在[构建脚本指南][构建脚本]中找到。

[构建脚本]: build-scripts.md

```toml
[package]
# ...
build = "build.rs"
```

默认值为 `"build.rs"`，它从包根目录中名为 `build.rs` 的文件加载脚本。使用 `build = "custom_build_name.rs"` 指定不同文件的路径，或使用 `build = false` 禁用自动检测构建脚本。

### `links` 字段 {#the-links-field}

`links` 字段指定正在链接的本地库的名称。更多信息可以在构建脚本指南的 [`links`][links] 部分找到。

[links]: build-scripts.md#the-links-manifest-key

例如，链接一个名为 "git2" 的本地库（例如，在 Linux 上是 `libgit2.a`）的 crate 可以指定：

```toml
[package]
# ...
links = "git2"
```

### `exclude` 和 `include` 字段 {#the-exclude-and-include-fields}

`exclude` 和 `include` 字段可用于明确指定将项目[发布][publishing]时包含哪些文件，以及某些类型的更改跟踪（如下所述）。`exclude` 字段中指定的模式标识一组不包含的文件，而 `include` 中的模式指定明确包含的文件。您可以运行 [`cargo package --list`][`cargo package`] 来验证哪些文件将包含在包中。

```toml
[package]
# ...
exclude = ["/ci", "images/", ".*"]
```

```toml
[package]
# ...
include = ["/src", "COPYRIGHT", "/examples", "!/examples/big_example"]
```

如果两个字段都未指定，则默认为包含包根目录下的所有文件，除了下面列出的排除项。

如果未指定 `include`，则以下文件将被排除：

* 如果包不在 Git 仓库中，所有以点开头的"隐藏"文件将被跳过。
* 如果包在 Git 仓库中，任何被仓库的 [gitignore] 规则和全局 Git 配置忽略的文件将被跳过。

无论是否指定 `exclude` 或 `include`，以下文件始终被排除：

* 任何子包将被跳过（任何包含 `Cargo.toml` 文件的子目录）。
* 包根目录中名为 `target` 的目录将被跳过。

以下文件始终被包含：

* 包本身的 `Cargo.toml` 文件始终被包含，不需要列在 `include` 中。
* 自动包含一个最小化的 `Cargo.lock`。更多信息请参见 [`cargo package`]。
* 如果指定了 [`license-file`](#the-license-and-license-file-fields)，它始终被包含。

这些选项是互斥的；设置 `include` 将覆盖 `exclude`。如果您需要对一组 `include` 文件进行排除，请使用下面描述的 `!` 运算符。

模式应为 [gitignore] 风格的模式。简要介绍：

- `foo` 匹配包中任何位置名为 `foo` 的任何文件或目录。这等同于模式 `**/foo`。
- `/foo` 仅匹配包根目录中名为 `foo` 的任何文件或目录。
- `foo/` 匹配包中任何位置名为 `foo` 的任何*目录*。
- 支持常见的通配符模式，如 `*`、`?` 和 `[]`：
  - `*` 匹配除 `/` 之外的零个或多个字符。例如，`*.html` 匹配包中任何位置具有 `.html` 扩展名的任何文件或目录。
  - `?` 匹配除 `/` 之外的任何字符。例如，`foo?` 匹配 `food`，但不匹配 `foo`。
  - `[]` 允许匹配一系列字符。例如，`[ab]` 匹配 `a` 或 `b`。`[a-z]` 匹配字母 a 到 z。
- `**/` 前缀匹配任何目录。例如，`**/foo/bar` 匹配直接在目录 `foo` 下的任何文件或目录 `bar`。
- `/**` 后缀匹配内部的所有内容。例如，`foo/**` 匹配目录 `foo` 内的所有文件，包括 `foo` 下子目录中的所有文件。
- `/**/` 匹配零个或多个目录。例如，`a/**/b` 匹配 `a/b`、`a/x/b`、`a/x/y/b` 等。
- `!` 前缀否定一个模式。例如，模式 `src/*.rs` 和 `!foo.rs` 将匹配 `src` 目录内具有 `.rs` 扩展名的所有文件，除了任何名为 `foo.rs` 的文件。

包含/排除列表也用于某些情况下的更改跟踪。对于使用 `rustdoc` 构建的目标，它用于确定要跟踪的文件列表，以确定是否应重新构建目标。如果包有一个[构建脚本]，该脚本不发出任何 `rerun-if-*` 指令，则包含/排除列表用于跟踪如果这些文件中的任何一个发生更改，是否应重新运行构建脚本。

[gitignore]: https://git-scm.com/docs/gitignore

### `publish` 字段 {#the-publish-field}

`publish` 字段可用于控制包可以发布到哪些注册表名称：
```toml
[package]
# ...
publish = ["some-registry-name"]
```

为防止包意外发布到注册表（如 crates.io），例如，为了在公司内保持包的私有性，您可以省略 [`version`](#the-version-field) 字段。如果您想更明确，可以禁用发布：
```toml
[package]
# ...
publish = false
```

如果发布数组包含单个注册表，`cargo publish` 命令将在未指定 `--registry` 标志时使用它。

### `metadata` 表 {#the-metadata-table}

默认情况下，Cargo 会警告 `Cargo.toml` 中未使用的键，以帮助检测拼写错误等。然而，`package.metadata` 表完全被 Cargo 忽略，不会发出警告。此部分可用于希望将包配置存储在 `Cargo.toml` 中的工具。例如：

```toml
[package]
name = "..."
# ...

# 例如，生成 Android APK 时使用的元数据。
[package.metadata.android]
package-name = "my-awesome-android-app"
assets = "path/to/static"
```

您需要查看工具的文档以了解如何使用此字段。对于使用 `package.metadata` 表的 Rust 项目，请参见：
- [docs.rs](https://docs.rs/about/metadata)

工作空间级别有一个类似的表：[`workspace.metadata`][workspace-metadata]。虽然 cargo 没有为这两个表的内容指定格式，但建议外部工具可能希望以一致的方式使用它们，例如，如果缺少 `package.metadata` 中的数据，则引用 `workspace.metadata` 中的数据（如果这对相关工具有意义的话）。

[workspace-metadata]: workspaces.md#the-metadata-table

### `default-run` 字段 {#the-default-run-field}

清单的 `[package]` 部分中的 `default-run` 字段可用于指定由 [`cargo run`] 选择的默认二进制文件。例如，当同时存在 `src/bin/a.rs` 和 `src/bin/b.rs` 时：

```toml
[package]
default-run = "a"
```

## `[lints]` 部分 {#the-lints-section}

通过将来自不同工具的默认检查级别分配给新级别来覆盖它们，例如：
```toml
[lints.rust]
unsafe_code = "forbid"
```

这是以下内容的简写：
```toml
[lints.rust]
unsafe_code = { level = "forbid", priority = 0 }
```

`level` 对应于 `rustc` 中的[代码检查级别](https://doc.rust-lang.org/rustc/lints/levels.html)：
- `forbid`
- `deny`
- `warn`
- `allow`

`priority` 是一个有符号整数，控制哪些检查或检查组覆盖其他检查组：
- 较低（尤其是负数）数字具有较低的优先级，被较高的数字覆盖，并首先出现在 `rustc` 等工具的命令行中

要了解特定检查属于 `[lints]` 下的哪个表，它是检查名称中 `::` 之前的部分。如果没有 `::`，则工具是 `rust`。例如，关于 `unsafe_code` 的警告将是 `lints.rust.unsafe_code`，但关于 `clippy::enum_glob_use` 的检查将是 `lints.clippy.enum_glob_use`。

例如：
```toml
[lints.rust]
unsafe_code = "forbid"

[lints.clippy]
enum_glob_use = "deny"
```

通常，这些只会影响当前包的本地开发。Cargo 仅将它们应用于当前包，而不应用于依赖项。至于依赖者，Cargo 通过[`--cap-lints`](../../rustc/lints/levels.html#capping-lints) 等功能抑制来自非路径依赖项的检查。

> **MSRV：** 自 1.74 起受支持

## `[hints]` 部分 {#the-hints-section}

`[hints]` 部分允许为此包的编译指定提示。默认情况下，Cargo 在编译此包时会尊重这些提示，尽管正在构建的顶层包可以通过 `[profile]` 机制覆盖这些值。提示本质上对于 Cargo 忽略总是安全的；如果 Cargo 遇到它不理解的提示，或者它理解但具有它不理解的值的提示，它将发出警告，而不是错误。因此，在 crate 中指定提示不会影响 crate 的 MSRV。

个别提示可能具有关联的不稳定功能门，您需要传递该功能门才能应用它们指定的配置，但如果您没有指定该不稳定功能门，您将再次只收到警告，而不是错误。

目前没有稳定的提示。有关不稳定提示的信息，请参见 [提示大部分未使用的文档](unstable.md#profile-hint-mostly-unused-option)。

> **MSRV：** 自 1.90 起受支持。

## `[badges]` 部分 {#the-badges-section}

`[badges]` 部分用于指定在包发布时可以在注册表网站上显示的状态徽章。

> 注意：[crates.io] 以前在网站上显示 crate 旁边的徽章，但该功能已被移除。包应将其徽章放在其 README 文件中，该文件将在 [crates.io] 上显示（请参见 [`readme` 字段](#the-readme-field)）。

```toml
[badges]
# `maintenance` 表指示 crate 的维护状态。
# 注册表可能会使用此表，但 crates.io 目前未使用。
# 更多详细信息请参见 https://github.com/rust-lang/crates.io/issues/2437
# 和 https://github.com/rust-lang/crates.io/issues/2438。
#
# `status` 字段是必需的。可用选项有：
# - `actively-developed`：正在添加新功能并修复错误。
# - `passively-maintained`：没有新功能的计划，但维护者打算响应提交的问题。
# - `as-is`：crate 功能完整，维护者不打算继续处理或提供支持，但它适用于其设计目的。
# - `experimental`：作者希望与社区分享，但不打算满足任何人的特定用例。
# - `looking-for-maintainer`：当前维护者希望将 crate 转移给其他人。
# - `deprecated`：维护者不推荐使用此 crate（crate 的描述可以说明原因，可能有更好的解决方案可用，或者可能存在维护者不想修复的问题）。
# - `none`：在 crates.io 上不显示徽章，因为维护者尚未选择指定其意图，潜在的 crate 用户需要自行调查。
maintenance = { status = "..." }
```

## 依赖部分 {#dependency-sections}

有关 `[dependencies]`、`[dev-dependencies]`、`[build-dependencies]` 和目标特定的 `[target.*.dependencies]` 部分的信息，请参见[指定依赖项页面](specifying-dependencies.md)。

## `[profile.*]` 部分 {#the-profile-sections}

`[profile]` 表提供了一种自定义编译器设置（如优化和调试设置）的方式。更多详细信息请参见[配置章节](profiles.md)。



[`cargo init`]: ../commands/cargo-init.md
[`cargo new`]: ../commands/cargo-new.md
[`cargo package`]: ../commands/cargo-package.md
[`cargo run`]: ../commands/cargo-run.md
[crates.io]: https://crates.io/
[docs.rs]: https://docs.rs/
[发布]: ../guide/publishing.md
[Rust 版本]: ../../edition-guide/index.html
[spdx-2.3-license-expressions]: https://spdx.github.io/spdx-spec/v2.3/SPDX-license-expressions/
[spdx-license-list-3.20]: https://github.com/spdx/license-list-data/tree/v3.20
[SPDX 网站]: https://spdx.org
[TOML]: https://toml.io/