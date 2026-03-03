# 覆盖依赖 {#overriding-dependencies}

出于多种情况，可能会产生覆盖依赖的需求。然而，大多数情况归结为能够在包发布到 [crates.io] 之前使用它。例如：

* 您正在开发的包也用于您正在开发的一个更大的应用程序中，并且您希望在这个更大的应用程序中测试对库的 bug 修复。
* 一个您未参与的上游包在其 git 仓库的主分支上有一个新功能或 bug 修复，您希望进行测试。
* 您即将发布包的新的主版本，但希望在整个包中进行集成测试，以确保新的主版本能够工作。
* 您为发现的一个 bug 向上游包提交了修复，但希望您的应用程序立即开始依赖于该包的修复版本，以避免因等待 bug 修复合并而被阻塞。

这些场景可以通过 [`[patch]` 清单部分](#the-patch-section)来解决。

本章将介绍几种不同的用例，并详细说明覆盖依赖的不同方法。

* 示例用例
    * [测试 bug 修复](#testing-a-bugfix)
    * [处理未发布的次要版本](#working-with-an-unpublished-minor-version)
        * [覆盖仓库 URL](#overriding-repository-url)
    * [预发布破坏性变更](#prepublishing-a-breaking-change)
    * [将 `[patch]` 用于多个版本](#using-patch-with-multiple-versions)
* 参考
    * [`[patch]` 部分](#the-patch-section)
    * [`[replace]` 部分](#the-replace-section)
    * [`paths` 覆盖](#paths-overrides)

> **注意**：另请参阅[多个位置][multiple locations]指定依赖项，这可用于覆盖本地包中单个依赖项声明的源。


## 测试 bug 修复 {#testing-a-bugfix}

假设您正在使用 [`uuid` crate]，但在使用过程中发现了一个 bug。不过，您非常积极进取，因此决定尝试修复这个 bug！最初，您的清单如下所示：

[`uuid` crate]: https://crates.io/crates/uuid

```toml
[package]
name = "my-library"
version = "0.1.0"

[dependencies]
uuid = "1.0"
```

我们要做的第一件事是通过以下方式在本地克隆 [`uuid` 仓库][uuid-repository]：

```console
$ git clone https://github.com/uuid-rs/uuid.git
```

接下来，我们将编辑 `my-library` 的清单，使其包含：

```toml
[patch.crates-io]
uuid = { path = "../path/to/uuid" }
```

这里我们声明我们正在用一个新依赖 *修补* 源 `crates-io`。这将有效地将本地签出的 `uuid` 版本添加到我们本地包的 crates.io 注册表中。

接下来，我们需要确保我们的锁文件更新为使用这个新版本的 `uuid`，以便我们的包使用本地签出的副本而不是来自 crates.io 的副本。`[patch]` 的工作方式是加载 `../path/to/uuid` 处的依赖项，然后每当查询 crates.io 中 `uuid` 的版本时，它 *也* 会返回本地版本。

这意味着本地签出版本的版本号很重要，并且会影响是否使用补丁。我们的清单声明了 `uuid = "1.0"`，这意味着我们只会解析到 `>= 1.0.0, < 2.0.0`，并且 Cargo 的贪婪解析算法也意味着我们将解析到该范围内的最大版本。通常，这并不重要，因为 git 仓库的版本已经大于或等于在 crates.io 上发布的最大版本，但牢记这一点很重要！

无论如何，通常您现在需要做的只是：

```console
$ cargo build
   Compiling uuid v1.0.0 (.../uuid)
   Compiling my-library v0.1.0 (.../my-library)
    Finished dev [unoptimized + debuginfo] target(s) in 0.32 secs
```

就是这样！您现在正在使用本地版本的 `uuid` 进行构建（注意构建输出中括号内的路径）。如果您没有看到正在构建本地路径版本，那么您可能需要运行 `cargo update uuid --precise $version`，其中 `$version` 是本地签出的 `uuid` 副本的版本。

修复了最初发现的 bug 后，您接下来要做的可能是将其作为拉取请求提交给 `uuid` 包本身。完成此操作后，您还可以更新 `[patch]` 部分。`[patch]` 内部的列表就像 `[dependencies]` 部分一样，因此一旦您的拉取请求被合并，您可以将 `path` 依赖更改为：

```toml
[patch.crates-io]
uuid = { git = 'https://github.com/uuid-rs/uuid.git' }
```

[uuid-repository]: https://github.com/uuid-rs/uuid

## 处理未发布的次要版本 {#working-with-an-unpublished-minor-version}

现在让我们稍微转换一下话题，从 bug 修复转向添加功能。在开发 `my-library` 时，您发现 `uuid` 包中需要一个全新的功能。您已经实现了此功能，在上面使用 `[patch]` 在本地进行了测试，并提交了拉取请求。让我们看看在实际发布之前如何继续使用和测试它。

同时假设 crates.io 上 `uuid` 的当前版本是 `1.0.0`，但自从那时起 git 仓库的主分支已更新到 `1.0.1`。此分支包含了您之前提交的新功能。为了使用这个仓库，我们将编辑我们的 `Cargo.toml`，使其看起来像：

```toml
[package]
name = "my-library"
version = "0.1.0"

[dependencies]
uuid = "1.0.1"

[patch.crates-io]
uuid = { git = 'https://github.com/uuid-rs/uuid.git' }
```

请注意，我们对 `uuid` 的本地依赖已更新为 `1.0.1`，因为一旦该包发布，这实际上就是我们需要使用的版本。然而，这个版本在 crates.io 上并不存在，因此我们通过清单的 `[patch]` 部分提供它。

现在，当我们的库被构建时，它将从 git 仓库获取 `uuid`，并解析为仓库内的 1.0.1 版本，而不是尝试从 crates.io 下载版本。一旦 1.0.1 在 crates.io 上发布，就可以删除 `[patch]` 部分。

还值得注意的是，`[patch]` 是 *可传递* 的。假设您在一个更大的包中使用 `my-library`，例如：

```toml
[package]
name = "my-binary"
version = "0.1.0"

[dependencies]
my-library = { git = 'https://example.com/git/my-library' }
uuid = "1.0"

[patch.crates-io]
uuid = { git = 'https://github.com/uuid-rs/uuid.git' }
```

请记住，`[patch]` 是 *可传递的*，但只能在 *顶级* 定义，因此我们作为 `my-library` 的消费者，如果需要，必须重复 `[patch]` 部分。但是在这里，新的 `uuid` 包既适用于我们对 `uuid` 的依赖，也适用于 `my-library -> uuid` 的依赖。整个包图将解析为 `uuid` 的一个版本，即 1.0.1，并且它将从 git 仓库拉取。

### 覆盖仓库 URL {#overriding-repository-url}

如果您想要覆盖的依赖不是从 `crates.io` 加载的，那么您需要稍微改变一下使用 `[patch]` 的方式。例如，如果依赖是 git 依赖，您可以使用本地路径覆盖它：

```toml
[patch."https://github.com/your/repository"]
my-library = { path = "../my-library/path" }
```

就是这样！

## 预发布破坏性变更 {#prepublishing-a-breaking-change}

让我们看看如何处理包的新主版本，通常伴随着破坏性更改。继续使用我们之前的包，这意味着我们将创建 `uuid` 包的 2.0.0 版本。在我们将所有更改提交到上游后，我们可以更新 `my-library` 的清单，使其看起来像：

```toml
[dependencies]
uuid = "2.0"

[patch.crates-io]
uuid = { git = "https://github.com/uuid-rs/uuid.git", branch = "2.0.0" }
```

就是这样！与前面的示例一样，2.0.0 版本实际上并不存在于 crates.io 上，但我们仍然可以通过 `[patch]` 部分的使用，通过 git 依赖引入它。作为思考练习，让我们再看一下上面的 `my-binary` 清单：

```toml
[package]
name = "my-binary"
version = "0.1.0"

[dependencies]
my-library = { git = 'https://example.com/git/my-library' }
uuid = "1.0"

[patch.crates-io]
uuid = { git = 'https://github.com/uuid-rs/uuid.git', branch = '2.0.0' }
```

请注意，这实际上会解析为两个版本的 `uuid` 包。`my-binary` 将继续使用 `uuid` 的 1.x.y 系列，但 `my-library` 将使用 `uuid` 的 `2.0.0` 版本。这将允许您通过依赖图逐步推出破坏性更改，而不必一次更新所有内容。

## 将 `[patch]` 用于多个版本 {#using-patch-with-multiple-versions}

您可以使用 `package` 键来重命名依赖，从而打上同一个包的多个版本的补丁。例如，假设 `serde` 包有一个 bug 修复，我们希望将其用于其 `1.*` 系列，但我们也想尝试使用我们 git 仓库中 `serde` 的 `2.0.0` 版本进行原型设计。要配置这个，我们可以这样做：

```toml
[patch.crates-io]
serde = { git = 'https://github.com/serde-rs/serde.git' }
serde2 = { git = 'https://github.com/example/serde.git', package = 'serde', branch = 'v2' }
```

第一个 `serde = ...` 指令指示应该从 git 仓库使用 serde `1.*`（拉入我们需要的 bug 修复），第二个 `serde2
= ...` 指示 `serde` 包也应该从 `https://github.com/example/serde` 的 `v2` 分支拉取。我们在这里假设该分支上的 `Cargo.toml` 提到了版本 `2.0.0`。

注意，当使用 `package` 键时，这里的 `serde2` 标识符实际上被忽略了。我们只需要一个不与其它补丁包冲突的唯一名称。

## `[patch]` 部分 {#the-patch-section}

`Cargo.toml` 的 `[patch]` 部分可用于用其他副本覆盖依赖项。其语法类似于
[`[dependencies]`][dependencies] 部分：

```toml
[patch.crates-io]
foo = { git = 'https://github.com/example/foo.git' }
bar = { path = 'my/local/bar' }

[dependencies.baz]
git = 'https://github.com/example/baz.git'

[patch.'https://github.com/example/baz']
baz = { git = 'https://github.com/example/patched-baz.git', branch = 'my-branch' }
```

> **注意**：`[patch]` 表也可以指定为[配置选项](config.md)，例如在 `.cargo/config.toml` 文件中或像 `--config 'patch.crates-io.rand.path="rand"'` 这样的 CLI 选项。这对于您不想提交的仅本地更改或临时测试补丁很有用。

`[patch]` 表由类似依赖的子表组成。`[patch]` 之后的每个键都是正在被修补的源的 URL 或注册表的名称。名称 `crates-io` 可用于覆盖默认注册表 [crates.io]。上面示例中的第一个 `[patch]` 演示了覆盖 [crates.io]，第二个 `[patch]` 演示了覆盖 git 源。

这些表中的每个条目都是普通的依赖规范，与清单的 `[dependencies]` 部分中找到的相同。`[patch]` 部分中列出的依赖项被解析并用于修补在指定 URL 的源。上面的清单代码片段用 `foo` 包和 `bar` 包修补了 `crates-io` 源（即 crates.io 本身）。它还使用来自其他地方的 `my-branch` 修补了 `https://github.com/example/baz` 源。

源可以用不存在的包版本来修补，也可以用已存在的包版本来修补。如果一个源被一个已经在源中存在的包版本修补，那么源的原始包将被替换。

Cargo 仅查看工作空间根目录的 `Cargo.toml` 清单中的补丁设置。依赖项中定义的补丁设置将被忽略。

## `[replace]` 部分 {#the-replace-section}

> **注意**：`[replace]` 已弃用。您应该使用 [`[patch]`](#patch-部分) 表。

Cargo.toml 的这一部分可用于用其他副本覆盖依赖项。其语法类似于 `[dependencies]` 部分：

```toml
[replace]
"foo:0.1.0" = { git = 'https://github.com/example/foo.git' }
"bar:1.0.2" = { path = 'my/local/bar' }
```

`[replace]` 表中的每个键都是一个[包 ID 规范](pkgid-spec.md)，它允许任意选择依赖图中的节点进行覆盖（需要 3 部分版本号）。每个键的值与指定依赖项的 `[dependencies]` 语法相同，只是不能指定特性。请注意，当一个包被覆盖时，覆盖它的副本必须具有相同的名称和版本，但可以来自不同的源（例如，git 或本地路径）。

Cargo 仅查看工作空间根目录的 `Cargo.toml` 清单中的替换设置。依赖项中定义的替换设置将被忽略。

## `paths` 覆盖 {#paths-overrides}

有时您只是临时处理一个包，并且不想像上面的 `[patch]` 部分那样修改 `Cargo.toml`。对于这种用例，Cargo 提供了一个更有限的覆盖版本，称为 **路径覆盖**。

路径覆盖通过 [`.cargo/config.toml`](config.md) 指定，而不是 `Cargo.toml`。在 `.cargo/config.toml` 中，您需要指定一个名为 `paths` 的键：

```toml
paths = ["/path/to/uuid"]
```

这个数组应该填满包含 `Cargo.toml` 的目录。在这个例子中，我们只添加了 `uuid`，因此它将是唯一被覆盖的包。此路径可以是绝对路径，也可以是相对于包含 `.cargo` 文件夹的目录的相对路径。

然而，路径覆盖比 `[patch]` 部分更受限制，因为它们无法更改依赖图的结构。当使用路径替换时，先前的依赖项集必须与新 `Cargo.toml` 规范完全匹配。例如，这意味着路径覆盖不能用于测试向包添加依赖项。相反，那种情况下必须使用 `[patch]`。因此，路径覆盖的使用通常仅限于快速错误修复，而不是较大的更改。

> **注意**：使用本地配置来覆盖路径仅适用于已发布到 [crates.io] 的包。您不能使用此功能来告诉 Cargo 如何查找本地未发布的包。


[crates.io]: https://crates.io/
[multiple locations]: specifying-dependencies.md#multiple-locations
[dependencies]: specifying-dependencies.md