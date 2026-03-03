# 指定依赖项 {#specifying-dependencies}

您的 crate 可以依赖于来自 [crates.io] 或其他注册中心、`git` 仓库或本地文件系统的子目录中的其他库。您还可以临时覆盖依赖项的位置——例如，以便能够测试您正在本地开发的依赖项中的错误修复。您可以为不同的平台使用不同的依赖项，以及仅在开发期间使用的依赖项。让我们看看如何做到这些。

## 指定来自 crates.io 的依赖项 {#specifying-dependencies-from-cratesio}

Cargo 默认配置为在 [crates.io] 上查找依赖项。在这种情况下，只需要名称和版本字符串。在 [cargo 指南](../guide/index.md)中，我们指定了对 `time` crate 的依赖：

```toml
[dependencies]
time = "0.1.12"
```

版本字符串 `"0.1.12"` 被称为[版本要求](#version-requirement-syntax)。它指定了在[解析依赖项](resolver.md)时可以选择的一系列版本。在这种情况下，`"0.1.12"` 表示版本范围 `>=0.1.12, <0.2.0`。如果更新在该范围内，则允许更新。例如，如果我们运行 `cargo update time`，cargo 应该更新到版本 `0.1.13`（如果它是最新的 `0.1.z` 版本），但不会更新到 `0.2.0`。

## 版本要求语法 {#version-requirement-syntax}

### 默认要求 {#default-requirements}

**默认要求**指定了最低版本，并允许更新到[语义化版本][SemVer]兼容的版本。如果版本最左侧非零的主版本/次版本/补丁版本组件相同，则被认为是兼容的。这与[语义化版本][SemVer]不同，后者认为所有 1.0.0 之前的包都是不兼容的。

`1.2.3` 是默认要求的一个示例。

```notrust
1.2.3  :=  >=1.2.3, <2.0.0
1.2    :=  >=1.2.0, <2.0.0
1      :=  >=1.0.0, <2.0.0
0.2.3  :=  >=0.2.3, <0.3.0
0.2    :=  >=0.2.0, <0.3.0
0.0.3  :=  >=0.0.3, <0.0.4
0.0    :=  >=0.0.0, <0.1.0
0      :=  >=0.0.0, <1.0.0
```

### 脱字号要求 {#caret-requirements}

**脱字号要求**是默认的版本要求策略。此版本策略允许[语义化版本][SemVer]兼容的更新。它们被指定为带有前导脱字号（`^`）的版本要求。

`^1.2.3` 是脱字号要求的一个示例。

省略脱字号是使用脱字号要求的简化等效语法。虽然脱字号要求是默认的，但建议尽可能使用简化语法。

`log = "^1.2.3"` 与 `log = "1.2.3"` 完全等效。

### 波浪号要求 {#tilde-requirements}

**波浪号要求**指定了具有某些更新能力的最低版本。如果您指定了主版本、次版本和补丁版本，或者仅指定了主版本和次版本，则只允许补丁级别的更改。如果您仅指定主版本，则允许次版本和补丁级别的更改。

`~1.2.3` 是波浪号要求的一个示例。

```notrust
~1.2.3  := >=1.2.3, <1.3.0
~1.2    := >=1.2.0, <1.3.0
~1      := >=1.0.0, <2.0.0
```

### 通配符要求 {#wildcard-requirements}

**通配符要求**允许通配符位置上的任何版本。

`*`、`1.*` 和 `1.2.*` 是通配符要求的示例。

```notrust
*     := >=0.0.0
1.*   := >=1.0.0, <2.0.0
1.2.* := >=1.2.0, <1.3.0
```

> **注意**：[crates.io] 不允许裸 `*` 版本。

### 比较要求 {#comparison-requirements}

**比较要求**允许手动指定要依赖的版本范围或确切版本。

以下是比较要求的一些示例：

```notrust
>= 1.2.0
> 1
< 2
= 1.2.3
```

<span id="multiple-requirements"></span>
### 多个版本要求 {#multiple-version-requirements}

如上面的示例所示，多个版本要求可以用逗号分隔，例如 `>= 1.2, < 1.5`。
所有要求都必须满足，因此像 `<1.2, ^1.2.2` 这样的非重叠要求会导致没有匹配的版本。

### 预发布版本 {#pre-releases}

版本要求排除[预发布版本](manifest.md#the-version-field)，例如 `1.0.0-alpha`，除非明确要求。例如，如果包 `foo` 的 `1.0.0-alpha` 已发布，那么要求 `foo = "1.0"` 将*不会*匹配，并会返回错误。必须指定预发布版本，例如 `foo = "1.0.0-alpha"`。类似地，[`cargo install`] 将避免预发布版本，除非明确要求安装一个。

Cargo 允许自动使用“较新”的预发布版本。例如，如果发布了 `1.0.0-beta`，那么要求 `foo = "1.0.0-alpha"` 将允许更新到 `beta` 版本。请注意，这仅适用于相同的发布版本，`foo = "1.0.0-alpha"` 不允许更新到 `foo = "1.0.1-alpha"` 或 `foo = "1.0.1-beta"`。

Cargo 还将自动从预发布版本升级到语义化版本兼容的发布版本。要求 `foo = "1.0.0-alpha"` 将允许更新到 `foo = "1.0.0"` 以及 `foo = "1.2.0"`。

请注意，预发布版本可能不稳定，因此在使用时应谨慎。有些项目可能选择在预发布版本之间发布破坏性更改。建议如果您的库不是预发布版本，则不要在库中使用预发布依赖项。在更新 `Cargo.lock` 时也应谨慎，并准备好在预发布更新导致问题时进行处理。

[`cargo install`]: ../commands/cargo-install.md

### 版本元数据 {#version-metadata}

[版本元数据](manifest.md#the-version-field)，例如 `1.0.0+21AF26D3`，将被忽略，不应在版本要求中使用。

> **建议**：如有疑问，请使用默认的版本要求运算符。
>
> 在极少数情况下，一个具有“公共依赖项”（重新导出依赖项或在其公共 API 中与之交互）的包可能与多个语义化版本不兼容的版本兼容（例如，仅使用一个在版本之间未更改的简单类型，如 `Id`），可能支持用户选择使用哪个版本的“公共依赖项”。在这种情况下，像 `">=0.4, <2"` 这样的版本要求可能值得关注。*但是*，如果用户还依赖于此依赖项，则包的用户可能会遇到错误，并且需要手动通过 `cargo update` 选择“公共依赖项”的版本，因为 Cargo 在[解析依赖项版本](resolver.md)时可能会选择“公共依赖项”的不同版本（参见 [#10599]）。
>
> 避免将版本的上限约束设置为任何低于下一个语义化版本不兼容版本的值（例如，避免 `">=2.0, <2.4"`、`"2.0.*"` 或 `~2.0`），因为依赖树中的其他包可能需要更新版本，从而导致无法解析的错误（参见 [#9029]）。考虑在您的 [`Cargo.lock`] 中控制版本是否更合适。
>
> 在某些情况下，这可能无关紧要，或者好处可能超过成本，包括：
> - 当没有其他人依赖您的包时；例如，它只有一个 `[[bin]]`
> - 当依赖预发布包并希望避免破坏性更改时，那么完全指定的 `"=1.2.3-alpha.3"` 可能是合理的（参见 [#2222]）
> - 当库重新导出过程宏，但过程宏生成的代码调用了重新导出的库时，那么完全指定的 `=1.2.3` 可能是合理的，以确保过程宏不会比重新导出的库更新，并生成使用当前版本中不存在的 API 部分的代码

[`Cargo.lock`]: ../guide/cargo-toml-vs-cargo-lock.md
[#2222]: https://github.com/rust-lang/cargo/issues/2222
[#9029]: https://github.com/rust-lang/cargo/issues/9029
[#10599]: https://github.com/rust-lang/cargo/issues/10599

## 指定来自其他注册中心的依赖项 {#specifying-dependencies-from-other-registries}

要指定来自 [crates.io] 以外的注册中心的依赖项，将 `registry` 键设置为要使用的注册中心的名称：

```toml
[dependencies]
some-crate = { version = "1.0", registry = "my-registry" }
```

其中 `my-registry` 是在 `.cargo/config.toml` 文件中配置的注册中心名称。有关更多信息，请参见[注册中心文档][registries documentation]。

> **注意**：[crates.io] 不允许发布依赖于 [crates.io] 外部代码的包。

[registries documentation]: registries.md

## 指定来自 `git` 仓库的依赖项 {#specifying-dependencies-from-git-repositories}

要依赖于位于 `git` 仓库中的库，您需要指定的最小信息是仓库的位置，使用 `git` 键：

```toml
[dependencies]
regex = { git = "https://github.com/rust-lang/regex.git" }
```

Cargo 获取该位置的 `git` 仓库，并遍历文件树以在 `git` 仓库内的任何位置找到请求的 crate 的 `Cargo.toml` 文件。例如，`regex-lite` 和 `regex-syntax` 是 `rust-lang/regex` 仓库的成员，可以通过仓库的根 URL（`https://github.com/rust-lang/regex.git`）来引用，无论它们在文件树中的位置如何。

```toml
regex-lite   = { git = "https://github.com/rust-lang/regex.git" }
regex-syntax = { git = "https://github.com/rust-lang/regex.git" }
```

上述规则不适用于[`path` 依赖项](#specifying-path-dependencies)。

### 提交选择 {#choice-of-commit}

Cargo 假设如果我们只指定了仓库 URL（如上面的示例中），则我们打算使用默认分支上的最新提交来构建我们的包。

您可以将 `git` 键与 `rev`、`tag` 或 `branch` 键结合使用，以更具体地指定要使用的提交。以下是一个使用名为 `next` 的分支上的最新提交的示例：

```toml
[dependencies]
regex = { git = "https://github.com/rust-lang/regex.git", branch = "next" }
```

任何不是分支或标签的内容都属于 `rev` 键。这可以是提交哈希，如 `rev = "4c59b707"`，也可以是远程仓库暴露的命名引用，例如 `rev = "refs/pull/493/head"`。

`rev` 键可用的引用因仓库托管位置而异。GitHub 将每个拉取请求的最新提交暴露为一个引用，如上例所示。其他 git 主机可能在不同的命名方案下提供等效的内容。

**更多 `git` 依赖项示例：**

```toml
# 如果主机接受这样的 URL，可以省略 .git 后缀——两个示例效果相同
regex = { git = "https://github.com/rust-lang/regex" }
regex = { git = "https://github.com/rust-lang/regex.git" }

# 带有特定标签的提交
regex = { git = "https://github.com/rust-lang/regex.git", tag = "1.10.3" }

# 通过 SHA1 哈希的提交
regex = { git = "https://github.com/rust-lang/regex.git", rev = "0c0990399270277832fbb5b91a1fa118e6f63dba" }

# PR 493 的 HEAD 提交
regex = { git = "https://github.com/rust-lang/regex.git", rev = "refs/pull/493/head" }

# 无效示例

# 在 # 之后指定提交会忽略提交 ID 并生成警告
regex = { git = "https://github.com/rust-lang/regex.git#4c59b70" }

# git 和 path 不能同时使用
regex = { git = "https://github.com/rust-lang/regex.git#4c59b70", path = "../regex" }
```

Cargo 在 `git` 依赖项添加到 `Cargo.lock` 文件时锁定其提交，并且仅在您运行 `cargo update` 命令时检查更新。

### `version` 键的作用 {#the-role-of-the-version-key}

`version` 键总是意味着包在注册中心中可用，无论是否存在 `git` 或 `path` 键。

`version` 键*不会*影响 Cargo 获取 `git` 依赖项时使用哪个提交，但 Cargo 会检查依赖项的 `Cargo.toml` 文件中的版本信息是否与 `version` 键匹配，并在检查失败时引发错误。

在此示例中，Cargo 从 Git 获取名为 `next` 的分支的 HEAD 提交，并检查 crate 的版本是否与 `version = "1.10.3"` 兼容：

```toml
[dependencies]
regex = { version = "1.10.3", git = "https://github.com/rust-lang/regex.git", branch = "next" }
```

`version`、`git` 和 `path` 键被认为是解析依赖项的独立位置。有关详细解释，请参见下面的[多个位置](#multiple-locations)部分。

> **注意**：[crates.io] 不允许发布依赖于 [crates.io] 本身之外的代码的包（[dev-dependencies] 被忽略）。有关 `git` 和 `path` 依赖项的备用方案，请参见[多个位置](#multiple-locations)部分。

### Git 子模块 {#git-submodules}

在克隆 `git` 依赖项时，Cargo 会自动递归获取其子模块，以便所有必需的代码都可用于构建。

要跳过获取与构建无关的子模块，您可以在依赖项仓库的 `.gitmodules` 中设置 [`submodule.<name>.update = none`][submodule-update]。这需要写权限，并将更普遍地禁用子模块更新。

[submodule-update]: https://git-scm.com/docs/gitmodules#Documentation/gitmodules.txt-submodulenameupdate

### 访问私有 Git 仓库 {#accessing-private-git-repositories}

有关私有仓库的 Git 认证帮助，请参见 [Git 认证](../appendix/git-authentication.md)。

## 指定路径依赖项 {#specifying-path-dependencies}

随着时间的推移，我们在[指南](../guide/index.md)中的 `hello_world` 包已经显著增长！它已经增长到我们可能希望分离出一个单独的 crate 供他人使用。为此，Cargo 支持**路径依赖项**，这些通常是存在于一个仓库内的子 crate。让我们首先在 `hello_world` 包内创建一个新的 crate：

```console
# 在 hello_world/ 内部
$ cargo new hello_utils
```

这将在内部创建一个新文件夹 `hello_utils`，其中 `Cargo.toml` 和 `src` 文件夹已准备好进行配置。为了告诉 Cargo 这一点，打开 `hello_world/Cargo.toml` 并将 `hello_utils` 添加到您的依赖项中：

```toml
[dependencies]
hello_utils = { path = "hello_utils" }
```

这告诉 Cargo 我们依赖于名为 `hello_utils` 的 crate，它位于 `hello_utils` 文件夹中，相对于编写此依赖项的 `Cargo.toml` 文件。

下一次 `cargo build` 将自动构建 `hello_utils` 及其所有依赖项。

### 无本地路径遍历 {#no-local-path-traversal}

本地路径必须指向包含依赖项的 `Cargo.toml` 的确切文件夹。与 `git` 依赖项不同，Cargo 不遍历本地路径。例如，如果 `regex-lite` 和 `regex-syntax` 是本地克隆的 `rust-lang/regex` 仓库的成员，它们必须通过完整路径来引用：

```toml
# git 键接受仓库根 URL，并且 Cargo 遍历树以找到 crate
[dependencies]
regex-lite   = { git = "https://github.com/rust-lang/regex.git" }
regex-syntax = { git = "https://github.com/rust-lang/regex.git" }

# path 键要求成员名称包含在本地路径中
[dependencies]
regex-lite   = { path = "../regex/regex-lite" }
regex-syntax = { path = "../regex/regex-syntax" }
```

### 已发布 crate 中的本地路径 {#local-paths-in-published-crates}

仅使用路径指定的依赖项在 [crates.io] 上是不允许的。

如果我们想发布我们的 `hello_world` crate，我们需要将 `hello_utils` 的一个版本作为单独的 crate 发布到 [crates.io]，并在 `hello_world` 的依赖项行中指定其版本：

```toml
[dependencies]
hello_utils = { path = "hello_utils", version = "0.1.0" }
```

同时使用 `path` 和 `version` 键的情况在[多个位置](#multiple-locations)部分有解释。

> **注意**：[crates.io] 不允许发布依赖于 [crates.io] 外部代码的包，除了 [dev-dependencies]。有关 `git` 和 `path` 依赖项的备用方案，请参见[多个位置](#multiple-locations)部分。

## 多个位置 {#multiple-locations}

可以同时指定注册中心版本和 `git` 或 `path` 位置。`git` 或 `path` 依赖项将在本地使用（在这种情况下，将检查本地副本的版本），而当发布到 [crates.io] 等注册中心时，将使用注册中心版本。其他组合是不允许的。示例：

```toml
[dependencies]
# 在本地使用时使用 `my-bitflags`，发布时使用 crates.io 的版本 1.0。
bitflags = { path = "my-bitflags", version = "1.0" }

# 在本地使用时使用给定的 git 仓库，发布时使用 crates.io 的版本 1.0。
smallvec = { git = "https://github.com/servo/rust-smallvec.git", version = "1.0" }

# 注意：如果版本不匹配，Cargo 将无法编译！
```

这种情况的一个有用示例是，当您将一个库拆分为同一个工作空间中的多个包时。然后，您可以使用 `path` 依赖项指向工作空间内的本地包，以便在开发期间使用本地版本，然后在发布后使用 [crates.io] 版本。这类似于指定[覆盖](overriding-dependencies.md)，但仅适用于此依赖项声明。

## 平台特定依赖项 {#platform-specific-dependencies}

平台特定依赖项采用相同的格式，但列在 `target` 部分下。通常使用类似 Rust 的 [`#[cfg]` 语法](../../reference/conditional-compilation.html)来定义这些部分：

```toml
[target.'cfg(windows)'.dependencies]
winhttp = "0.4.0"

[target.'cfg(unix)'.dependencies]
openssl = "1.0.1"

[target.'cfg(target_arch = "x86")'.dependencies]
native-i686 = { path = "native/i686" }

[target.'cfg(target_arch = "x86_64")'.dependencies]
native-x86_64 = { path = "native/x86_64" }
```

与 Rust 类似，此处的语法支持 `not`、`any` 和 `all` 运算符来组合各种 cfg 名称/值对。

如果您想知道您的平台上有哪些 cfg 目标可用，请从命令行运行 `rustc --print=cfg`。如果您想知道另一个平台（例如 64 位 Windows）有哪些 `cfg` 目标可用，请运行 `rustc --print=cfg --target=x86_64-pc-windows-msvc`。

与在 Rust 源代码中不同，您不能使用 `[target.'cfg(feature = "fancy-feature")'.dependencies]` 来基于可选功能添加依赖项。请改用[`[features]` 部分](features.md)：

```toml
[dependencies]
foo = { version = "1.0", optional = true }
bar = { version = "1.0", optional = true }

[features]
fancy-feature = ["foo", "bar"]
```

这同样适用于 `cfg(debug_assertions)`、`cfg(test)` 和 `cfg(proc_macro)`。这些值将不起作用，并且总是具有 `rustc --print=cfg` 返回的默认值。目前没有基于这些配置值添加依赖项的方法。

除了 `#[cfg]` 语法外，Cargo 还支持列出依赖项将适用的完整目标：

```toml
[target.x86_64-pc-windows-gnu.dependencies]
winhttp = "0.4.0"

[target.i686-unknown-linux-gnu.dependencies]
openssl = "1.0.1"
```

### 自定义目标规范 {#custom-target-specifications}

如果您使用自定义目标规范（例如 `--target foo/bar.json`），请使用不带 `.json` 扩展名的基本文件名：

```toml
[target.bar.dependencies]
winhttp = "0.4.0"

[target.my-special-i686-platform.dependencies]
openssl = "1.0.1"
native = { path = "native/i686" }
```

> **注意**：自定义目标规范不能在稳定频道上使用。

## 开发依赖项 {#development-dependencies}

您可以在 `Cargo.toml` 中添加一个 `[dev-dependencies]` 部分，其格式与 `[dependencies]` 等效：

```toml
[dev-dependencies]
tempdir = "0.3"
```

开发依赖项在构建包时不被使用，但在编译测试、示例和基准测试时使用。

这些依赖项*不会*传播给依赖于该包的其他包。

您也可以拥有特定目标的开发依赖项，方法是在目标部分头中使用 `dev-dependencies` 而不是 `dependencies`。例如：

```toml
[target.'cfg(unix)'.dev-dependencies]
mio = "0.0.1"
```

> **注意**：当包发布时，只有指定了 `version` 的开发依赖项才会包含在发布的 crate 中。对于大多数用例，发布的 crate 不需要开发依赖项，尽管某些用户（如操作系统打包者）可能希望在 crate 内运行测试，因此如果可能，提供 `version` 仍然有益。

## 构建依赖项 {#build-dependencies}

您可以依赖其他基于 Cargo 的 crate 以用于构建脚本中。依赖项通过清单的 `build-dependencies` 部分声明：

```toml
[build-dependencies]
cc = "1.0.3"
```

您也可以拥有特定目标的构建依赖项，方法是在目标部分头中使用 `build-dependencies` 而不是 `dependencies`。例如：

```toml
[target.'cfg(unix)'.build-dependencies]
cc = "1.0.3"
```

在这种情况下，仅当主机平台与指定目标匹配时，才会构建依赖项。

构建脚本**无法**访问 `dependencies` 或 `dev-dependencies` 部分中列出的依赖项。构建依赖项同样不会对包本身可用，除非它们也列在 `dependencies` 部分下。包本身及其构建脚本是分开构建的，因此它们的依赖项不需要一致。Cargo 通过为独立目的使用独立依赖项而保持更简单和更清晰。

## 选择功能 {#choosing-features}

如果您依赖的包提供了条件功能，您可以指定使用哪些：

```toml
[dependencies.awesome]
version = "1.3.5"
default-features = false # 不包括默认功能，并且可选地
                         # 挑选个别功能
features = ["secure-password", "civet"]
```

有关功能的更多信息可以在[功能章节](features.md#dependency-features)中找到。

## 在 `Cargo.toml` 中重命名依赖项 {#renaming-dependencies-in-cargotoml}

在 `Cargo.toml` 中编写 `[dependencies]` 部分时，您为依赖项编写的键通常与代码中导入的 crate 名称匹配。然而，对于某些项目，您可能希望无论 crate 如何在 crates.io 上发布，都在代码中以不同的名称引用它。例如，您可能希望：

* 避免在 Rust 源代码中需要 `use foo as bar`。
* 依赖于一个 crate 的多个版本。
* 依赖于来自不同注册中心的同名 crate。

为了支持这一点，Cargo 在 `[dependencies]` 部分中支持一个 `package` 键，用于指定应该依赖哪个包：

```toml
[package]
name = "mypackage"
version = "0.0.1"

[dependencies]
foo = "0.1"
bar = { git = "https://github.com/example/project.git", package = "foo" }
baz = { version = "0.1", registry = "custom", package = "foo" }
```

在这个例子中，现在在您的 Rust 代码中有三个 crate 可用：

```rust,ignore
extern crate foo; // crates.io
extern crate bar; // git 仓库
extern crate baz; // 注册中心 `custom`
```

这三个 crate 在它们自己的 `Cargo.toml` 中都有包名 `foo`，因此我们明确使用 `package` 键来通知 Cargo 我们想要 `foo` 包，即使我们在本地称它为其他名称。如果未指定，`package` 键默认为所请求的依赖项的名称。

请注意，如果您有一个可选依赖项，如：

```toml
[dependencies]
bar = { version = "0.1", package = 'foo', optional = true }
```

您依赖于来自 crates.io 的 crate `foo`，但您的 crate 具有 `bar` 功能而不是 `foo` 功能。也就是说，当重命名时，功能的名称取自依赖项的名称，而不是包名。

启用传递依赖项的工作方式类似，例如，我们可以将以下内容添加到上面的清单中：

```toml
[features]
log-debug = ['bar/log-debug'] # 使用 'foo/log-debug' 将是错误的！
```

## 从工作空间继承依赖项 {#inheriting-a-dependency-from-a-workspace}

依赖项可以通过在[工作空间的 `[workspace.dependencies]` 表][workspace.dependencies]中指定来从工作空间继承。之后，使用 `workspace = true` 将其添加到 `[dependencies]` 表中。

除了 `workspace` 键外，依赖项还可以包含这些键：
- [`optional`][optional]：请注意，`[workspace.dependencies]` 表不允许指定 `optional`。
- [`features`][features]：这些与在 `[workspace.dependencies]` 中声明的功能是叠加的。

除了 `optional` 和 `features` 之外，继承的依赖项不能使用任何其他依赖项键（例如 `version` 或 `default-features`）。

`[dependencies]`、`[dev-dependencies]`、`[build-dependencies]` 和 `[target."...".dependencies]` 部分中的依赖项支持引用 `[workspace.dependencies]` 定义的依赖项的能力。

```toml
[package]
name = "bar"
version = "0.2.0"

[dependencies]
regex = { workspace = true, features = ["unicode"] }

[build-dependencies]
cc.workspace = true

[dev-dependencies]
rand = { workspace = true, optional = true }
```


[SemVer]: https://semver.org
[crates.io]: https://crates.io/
[dev-dependencies]: #development-dependencies
[workspace.dependencies]: workspaces.md#the-dependencies-table
[optional]: features.md#optional-dependencies
[features]: features.md