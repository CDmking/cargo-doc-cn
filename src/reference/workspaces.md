# 工作空间 {#workspaces}

*工作空间*是一个由一个或多个包（称为*工作空间成员*）组成的集合，它们被一起管理。

工作空间的关键点包括：

* 常见命令可以在所有工作空间成员上运行，例如 `cargo check --workspace`。
* 所有包共享一个位于*工作空间根目录*的公共 [`Cargo.lock`] 文件。
* 所有包共享一个公共的[输出目录]，默认情况下是*工作空间根目录*中名为 `target` 的目录。
* 共享包元数据，例如通过 [`workspace.package`](#the-package-table)。
* `Cargo.toml` 中的 [`[patch]`][patch]、[`[replace]`][replace] 和 [`[profile.*]`][profiles] 部分仅在*根*清单中被识别，在成员 crate 的清单中被忽略。

工作空间的根目录 `Cargo.toml` 支持以下部分：

* [`[workspace]`](#the-workspace-section) --- 定义一个工作空间。
  * [`resolver`](resolver.md#resolver-versions) --- 设置要使用的依赖解析器。
  * [`members`](#the-members-and-exclude-fields) --- 包含在工作空间中的包。
  * [`exclude`](#the-members-and-exclude-fields) --- 从工作空间中排除的包。
  * [`default-members`](#the-default-members-field) --- 当未选择特定包时要操作的包。
  * [`package`](#the-package-table) --- 包中可继承的键。
  * [`dependencies`](#the-dependencies-table) --- 包依赖中可继承的键。
  * [`lints`](#the-lints-table) --- 包代码检查中可继承的键。
  * [`metadata`](#the-metadata-table) --- 外部工具的额外设置。
* [`[patch]`](overriding-dependencies.md#the-patch-section) --- 覆盖依赖。
* [`[replace]`](overriding-dependencies.md#the-replace-section) --- 覆盖依赖（已弃用）。
* [`[profile]`](profiles.md) --- 编译器设置和优化。

## `[workspace]` 部分 {#the-workspace-section}

要创建工作空间，您需要在 `Cargo.toml` 中添加 `[workspace]` 表：

```toml
[workspace]
# ...
```

至少，一个工作空间必须有一个成员，可以是根包，也可以是虚拟清单。

### 根包 {#root-package}

如果 `[workspace]` 部分被添加到一个已经定义了 `[package]` 的 `Cargo.toml` 中，则该包是工作空间的*根包*。*工作空间根目录*是工作空间的 `Cargo.toml` 文件所在的目录。

```toml
[workspace]

[package]
name = "hello_world" # 包的名称
version = "0.1.0"    # 当前版本，遵循语义化版本规范
```

### 虚拟工作空间 {#virtual-workspace}

或者，可以创建一个带有 `[workspace]` 部分但没有 [`[package]` 部分][package] 的 `Cargo.toml` 文件。这称为*虚拟清单*。当没有“主要”包，或者您希望将所有包组织在单独的目录中时，这通常很有用。

```toml
# [PROJECT_DIR]/Cargo.toml
[workspace]
members = ["hello_world"]
resolver = "3"
```

```toml
# [PROJECT_DIR]/hello_world/Cargo.toml
[package]
name = "hello_world" # 包的名称
version = "0.1.0"    # 当前版本，遵循语义化版本规范
edition = "2024"     # 版本，对工作空间中使用的解析器没有影响
```

通过拥有没有根包的工作空间：

- 必须在虚拟工作空间中显式设置 [`resolver`](resolver.md#resolver-versions)，因为它们没有用于推断[解析器版本](resolver.md#resolver-versions)的 [`package.edition`][package-edition]。
- 在工作空间根目录中运行的命令将默认对所有工作空间成员运行，参见 [`default-members`](#the-default-members-field)。

## `members` 和 `exclude` 字段 {#the-members-and-exclude-fields}

`members` 和 `exclude` 字段定义哪些包是工作空间的成员：

```toml
[workspace]
members = ["member1", "path/to/member2", "crates/*"]
exclude = ["crates/foo", "path/to/other"]
```

位于工作空间目录中的所有 [`path` 依赖项]自动成为成员。其他成员可以在 `members` 键中列出，这应该是一个字符串数组，包含带有 `Cargo.toml` 文件的目录。

`members` 列表还支持使用[通配符]来匹配多个路径，使用典型的文件名通配符模式，如 `*` 和 `?`。

`exclude` 键可用于阻止路径被包含在工作空间中。如果某些路径依赖项根本不被希望包含在工作空间中，或者使用了通配符模式并且您希望移除一个目录，这将非常有用。

当位于工作空间内的子目录中时，Cargo 将自动搜索父目录以查找带有 `[workspace]` 定义的 `Cargo.toml` 文件，以确定使用哪个工作空间。可以在成员 crate 中使用 [`package.workspace`] 清单键来指向工作空间的根目录，以覆盖此自动搜索。如果成员不在工作空间根目录的子目录中，则手动设置可能很有用。

### 包选择 {#package-selection}

在工作空间中，与包相关的 Cargo 命令（如 [`cargo build`]）可以使用 `-p` / `--package` 或 `--workspace` 命令行标志来确定要操作哪些包。如果未指定这些标志，Cargo 将使用当前工作目录中的包。但是，如果当前目录是工作空间根目录，则将使用 [`default-members`](#the-default-members-field)。

## `default-members` 字段 {#the-default-members-field}

`default-members` 字段指定当处于工作空间根目录且未使用包选择标志时要操作的[成员](#the-members-and-exclude-fields)路径：

```toml
[workspace]
members = ["path/to/member1", "path/to/member2", "path/to/member3/*"]
default-members = ["path/to/member2", "path/to/member3/foo"]
```

> 注意：当存在[根包](#root-package)时，您只能使用 `--package` 和 `--workspace` 标志来操作它。

如果未指定，则将使用[根包](#root-package)。对于[虚拟工作空间](#virtual-workspace)，将使用所有成员（就像在命令行上指定了 `--workspace` 一样）。

## `package` 表 {#the-package-table}

`workspace.package` 表是定义可以被工作空间成员继承的键的地方。这些键可以通过在成员包中定义 `{key}.workspace = true` 来继承。

支持的键包括：

|                |                 |
|----------------|-----------------|
| `authors`      | `categories`    |
| `description`  | `documentation` |
| `edition`      | `exclude`       |
| `homepage`     | `include`       |
| `keywords`     | `license`       |
| `license-file` | `publish`       |
| `readme`       | `repository`    |
| `rust-version` | `version`       |

- `license-file` 和 `readme` 是相对于工作空间根目录的
- `include` 和 `exclude` 是相对于您的包根目录的

示例：

```toml
# [PROJECT_DIR]/Cargo.toml
[workspace]
members = ["bar"]

[workspace.package]
version = "1.2.3"
authors = ["Nice Folks"]
description = "A short description of my package"
documentation = "https://example.com/bar"
```

```toml
# [PROJECT_DIR]/bar/Cargo.toml
[package]
name = "bar"
version.workspace = true
authors.workspace = true
description.workspace = true
documentation.workspace = true
```

> **MSRV：** 需要 1.64+

## `dependencies` 表 {#the-dependencies-table}

`workspace.dependencies` 表是定义可以被工作空间成员继承的依赖项的地方。

指定工作空间依赖项类似于[包依赖项][specifying-dependencies]，除了：
- 此表中的依赖项不能被声明为 `optional`
- 此表中声明的 [`features`][features] 与 `[dependencies]` 中的 `features` 是叠加的

然后，您可以[将工作空间依赖项作为包依赖项继承][inheriting-a-dependency-from-a-workspace]

示例：

```toml
# [PROJECT_DIR]/Cargo.toml
[workspace]
members = ["bar"]

[workspace.dependencies]
cc = "1.0.73"
rand = "0.8.5"
regex = { version = "1.6.0", default-features = false, features = ["std"] }
```

```toml
# [PROJECT_DIR]/bar/Cargo.toml
[package]
name = "bar"
version = "0.2.0"

[dependencies]
regex = { workspace = true, features = ["unicode"] }

[build-dependencies]
cc.workspace = true

[dev-dependencies]
rand.workspace = true
```

> **MSRV：** 需要 1.64+

## `lints` 表 {#the-lints-table}

`workspace.lints` 表是定义可以被工作空间成员继承的代码检查配置的地方。

指定工作空间代码检查配置类似于[包代码检查](manifest.md#the-lints-section)。

示例：

```toml
# [PROJECT_DIR]/Cargo.toml
[workspace]
members = ["crates/*"]

[workspace.lints.rust]
unsafe_code = "forbid"
```

```toml
# [PROJECT_DIR]/crates/bar/Cargo.toml
[package]
name = "bar"
version = "0.1.0"

[lints]
workspace = true
```

> **MSRV：** 自 1.74 起受支持

## `metadata` 表 {#the-metadata-table}

`workspace.metadata` 表被 Cargo 忽略，并且不会发出警告。此部分可用于希望将工作空间配置存储在 `Cargo.toml` 中的工具。例如：

```toml
[workspace]
members = ["member1", "member2"]

[workspace.metadata.webcontents]
root = "path/to/webproject"
tool = ["npm", "run", "build"]
# ...
```

包级别有一组类似的表：[`package.metadata`][package-metadata]。虽然 cargo 没有为这些表的内容指定格式，但建议外部工具可能希望以一致的方式使用它们，例如，如果对相关工具有意义，则在缺少 `package.metadata` 中的数据时引用 `workspace.metadata` 中的数据。

[package]: manifest.md#the-package-section
[`Cargo.lock`]: ../guide/cargo-toml-vs-cargo-lock.md
[package-metadata]: manifest.md#the-metadata-table
[package-edition]: manifest.md#the-edition-field
[输出目录]: build-cache.md
[patch]: overriding-dependencies.md#the-patch-section
[replace]: overriding-dependencies.md#the-replace-section
[profiles]: profiles.md
[`path` 依赖项]: specifying-dependencies.md#specifying-path-dependencies
[`package.workspace`]: manifest.md#the-workspace-field
[通配符]: https://docs.rs/glob/0.3.0/glob/struct.Pattern.html
[`cargo build`]: ../commands/cargo-build.md
[指定依赖项]: specifying-dependencies.md
[功能]: features.md
[从工作空间继承依赖项]: specifying-dependencies.md#inheriting-a-dependency-from-a-workspace