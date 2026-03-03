# 依赖项解析 {#dependency-resolution}

Cargo 的主要任务之一是根据每个包中指定的版本要求来确定要使用的依赖版本。这个过程称为“依赖解析”，由“解析器”执行。解析结果存储在 [`Cargo.lock` 文件] 中，该文件将依赖“锁定”到特定版本，并随时间推移保持固定。可以使用 [`cargo tree`] 命令来可视化解析器的结果。

[`Cargo.lock` 文件]: ../guide/cargo-toml-vs-cargo-lock.md
[dependency specifications]: specifying-dependencies.md
[dependency specification]: specifying-dependencies.md
[`cargo tree`]: ../commands/cargo-tree.md

## 约束与启发式方法 {#constraints-and-heuristics}

在许多情况下，并没有单一的“最佳”依赖解析方案。解析器在各种约束和启发式方法下运行，以找到一个普遍适用的解析方案。为了理解这些因素如何相互作用，粗略了解依赖解析的工作原理会有所帮助。

以下伪代码近似地模拟了 Cargo 解析器的工作方式：

```rust
pub fn resolve(workspace: &[Package], policy: Policy) -> Option<ResolveGraph> {
    let dep_queue = Queue::new(workspace);
    let resolved = ResolveGraph::new();
    resolve_next(dep_queue, resolved, policy)
}

fn resolve_next(dep_queue: Queue, resolved: ResolveGraph, policy: Policy) -> Option<ResolveGraph> {
    let Some(dep_spec) = policy.pick_next_dep(&mut dep_queue) else {
        // Done
        return Some(resolved);
    };

    if let Some(resolved) = policy.try_unify_version(dep_spec, resolved.clone()) {
        return Some(resolved);
    }

    let dep_versions = dep_spec.lookup_versions()?;
    let mut dep_versions = policy.filter_versions(dep_spec, dep_versions);
    while let Some(dep_version) = policy.pick_next_version(&mut dep_versions) {
        if policy.needs_version_unification(&dep_version, &resolved) {
            continue;
        }

        let mut dep_queue = dep_queue.clone();
        dep_queue.enqueue(&dep_version.dependencies);
        let mut resolved = resolved.clone();
        resolved.register(dep_version);
        if let Some(resolved) = resolve_next(dep_queue, resolved, policy) {
            return Some(resolved);
        }
    }

    // No valid solution found, backtrack and `pick_next_version`
    None
}
```

关键步骤：
- **遍历依赖（`pick_next_dep`）**：依赖项的遍历顺序会影响同一依赖项的相关版本要求的解析方式（参见“统一版本”），以及解析器回溯的程度，从而影响解析器的性能。
- **统一版本（`try_unify_version`、`needs_version_unification`）**：Cargo 尽可能重用版本，以减少构建时间，并允许来自公共依赖项的类型在不同 API 之间传递。如果由于 [依赖规范][dependency specifications] 中存在冲突而无法统一多个版本，Cargo 将进行回溯，如果找不到解决方案则报错，而不是选择多个版本。一个 [依赖规范][dependency specifications] 或 Cargo 可能认为某个版本不可取，宁可回溯或报错也不使用它。
- **优先选择版本（`pick_next_version`）**：Cargo 可能认为应该优先选择特定版本，并在回溯时回退到下一个版本。

### 版本号 {#version-numbers}

通常，Cargo 偏好当前可用的最高版本。

例如，如果你的解析图中有一个包包含：

```toml
[dependencies]
bitflags = "*"
```

如果在生成 `Cargo.lock` 文件时，`bitflags` 的最高版本是 `1.2.1`，那么该包将使用 `1.2.1`。

关于可能存在的例外情况，请参阅 [Rust 版本](#rust-version)。

### 版本要求 {#version-requirements}

包通过[版本要求]指定它们支持的版本，拒绝所有其他版本。

例如，如果你的解析图中有一个包包含：

```toml
[dependencies]
bitflags = "1.0"  # 含义是 `>=1.0.0,<2.0.0`
```

如果在生成 `Cargo.lock` 文件时，`bitflags` 的最高版本是 `1.2.1`，那么该包将使用 `1.2.1`，因为它是兼容范围内的最高版本。如果发布了 `2.0.0`，它仍将使用 `1.2.1`，因为 `2.0.0` 被认为是不兼容的。

[版本要求]: specifying-dependencies.md#version-requirement-syntax

### SemVer 兼容性 {#semver-compatibility}

Cargo 假设包遵循 [SemVer]，并且如果版本根据 [脱字符版本要求] 是 [SemVer] 兼容的，则会统一依赖版本。如果由于冲突的版本要求而无法统一两个兼容版本，Cargo 将报错。

有关哪些更改被认为是“兼容”的指南，请参见 [SemVer 兼容性] 章节。

示例：

以下两个包将统一它们对 `bitflags` 的依赖，因为选择的任何版本对彼此都是兼容的。

```toml
# Package A
[dependencies]
bitflags = "1.0"  # 含义是 `>=1.0.0,<2.0.0`

# Package B
[dependencies]
bitflags = "1.1"  # 含义是 `>=1.1.0,<2.0.0`
```

以下包将报错，因为版本要求冲突，选择了两个不同的兼容版本。

```toml
# Package A
[dependencies]
log = "=0.4.11"

# Package B
[dependencies]
log = "=0.4.8"
```

以下两个包将不会统一它们对 `rand` 的依赖，因为每个包只有不兼容的版本可用。相反，两个不同的版本（例如 0.6.5 和 0.7.3）将被解析并构建。这可能导致潜在问题，更多细节请参阅 [版本不兼容的风险] 部分。

```toml
# Package A
[dependencies]
rand = "0.7"  # 含义是 `>=0.7.0,<0.8.0`

# Package B
[dependencies]
rand = "0.6"  # 含义是 `>=0.6.0,<0.7.0`
```

通常，以下两个包将不会统一它们的依赖，因为存在满足版本要求的不兼容版本可用。相反，两个不同的版本（例如 0.6.5 和 0.7.3）将被解析并构建。其他约束或启发式方法的应用可能会导致它们统一，选择一个版本（例如 0.6.5）。

```toml
# Package A
[dependencies]
rand = ">=0.6,<0.8.0"

# Package B
[dependencies]
rand = "0.6"  # 含义是 `>=0.6.0,<0.7.0`
```

[SemVer]: https://semver.org/
[SemVer 兼容性]: semver.md
[脱字符版本要求]: specifying-dependencies.md#default-requirements
[版本不兼容的风险]: #version-incompatibility-hazards

#### 版本不兼容的风险 {#version-incompatiblity-hazards}

当一个 crate 的多个版本出现在解析图中时，如果使用这些 crate 的包暴露了来自这些 crate 的类型，就可能引发问题。这是因为 Rust 编译器认为它们是不同的类型和项，即使它们名称相同。库在发布不兼容的 SemVer 版本时（例如，在 `1.0.0` 被广泛使用后发布 `2.0.0`）应该特别小心，尤其是对于广泛使用的库。

"[SemVer 技巧]" 是在发布破坏性更改的同时保持与旧版本兼容性的一种解决方法。链接页面详细介绍了问题是什么以及如何解决。简而言之，当一个库想要发布一个破坏 SemVer 的版本时，发布新版本，同时也发布旧版本的一个点版本，该版本从新版本重新导出类型。

这些不兼容性通常表现为编译时错误，但有时也可能仅在运行时表现为错误行为。例如，假设有一个名为 `foo` 的公共库最终在解析图中同时出现了版本 `1.0.0` 和 `2.0.0`。如果在由使用版本 `1.0.0` 的库创建的对象上使用了 [`downcast_ref`]，而调用 `downcast_ref` 的代码向下转型为目标类型来自版本 `2.0.0`，那么向下转型将在运行时失败。

因此，确保如果你使用了库的多个版本，需要正确使用它们，特别是当可能存在来自不同版本的类型一起使用的情况时，这非常重要。可以使用 `cargo tree -d` 命令来识别重复版本及其来源。同样，发布一个流行库的 SemVer 不兼容版本时，考虑其对生态系统的影响也非常重要。

[SemVer 技巧]: https://github.com/dtolnay/semver-trick
[`downcast_ref`]: ../../std/any/trait.Any.html#method.downcast_ref

### 锁文件 {#lock-file}

在使用时，Cargo 会优先考虑已包含在 [`Cargo.lock` 文件] 中的版本。这旨在平衡可重复构建与适应清单（manifest） 变化的调整。

例如，如果你的解析图中有一个包包含：

```toml
[dependencies]
bitflags = "*"
```

如果你的 `Cargo.lock` 文件生成时 `bitflags` 的最高版本是 `1.2.1`，那么该包将使用 `1.2.1` 并记录在 `Cargo.lock` 文件中。

到 Cargo 下一次运行时，`bitflags` `1.3.5` 发布了。在解析依赖时，仍将使用 `1.2.1`，因为它存在于你的 `Cargo.lock` 文件中。

然后将该包编辑为：

```toml
[dependencies]
bitflags = "1.3.0"
```

`bitflags` `1.2.1` 不满足此版本要求，因此 `Cargo.lock` 文件中的该条目将被忽略，现在将使用版本 `1.3.5` 并记录到你的 `Cargo.lock` 文件中。

### Rust 版本 {#rust-version}

为了支持开发具有最低受支持 [Rust 版本] 的软件，解析器可以考虑依赖版本与你的 Rust 版本的兼容性。这由配置字段 [`resolver.incompatible-rust-versions`] 控制。

使用 `fallback` 设置时，解析器将优先选择其 Rust 版本小于或等于你的 Rust 版本的包。
例如，你正在使用 Rust 1.85 来开发以下包：

```toml
[package]
name = "my-cli"
rust-version = "1.62"

[dependencies]
clap = "4.0"  # 解析为 4.0.32
```

解析器将选择版本 4.0.32，因为它的 Rust 版本是 1.60.0。
- 不会选择 4.0.0，因为尽管它的 Rust 版本也是 1.60.0，但它的 [版本号更低](#version-numbers)。
- 不会选择 4.5.20，因为它与 `my-cli` 的 Rust 版本 1.62 不兼容，尽管它的 [版本号更高](#version-numbers)，并且它的 Rust 版本为 1.74.0，与你的 1.85 工具链兼容。

如果某个版本要求不包含任何与 Rust 版本兼容的依赖版本，解析器不会报错，而是会选择一个版本，即使它可能不是最优的。
例如，你更改了对 `clap` 的依赖：

```toml
[package]
name = "my-cli"
rust-version = "1.62"

[dependencies]
clap = "4.2"  # 解析为 4.5.20
```

没有 `clap` 的 [版本要求](#version-requirements) 与 Rust 版本 1.62 兼容。
然后解析器将选择一个不兼容的版本，例如 4.5.20，即使它的 Rust 版本是 1.74。

当解析器选择包的某个依赖版本时，它并不知道最终会有哪些工作空间成员对该版本有传递性依赖，因此它无法只考虑与该依赖相关的 Rust 版本。当工作空间成员具有不同的 Rust 版本时，解析器有启发式方法来找到一个“足够好”的解决方案。即使对于工作空间中没有 Rust 版本的包也是如此。

当工作空间中的成员具有不同的 Rust 版本时，解析器可能会选择比必要更低的依赖版本。
例如，你有以下工作空间成员：

```toml
[package]
name = "a"
rust-version = "1.62"

[package]
name = "b"

[dependencies]
clap = "4.2"  # 解析为 4.5.20
```

尽管包 `b` 没有指定 Rust 版本，并且可以使用更高的版本如 4.5.20，但由于包 `a` 的 Rust 版本为 1.62，仍将选择 4.0.32。

或者解析器可能会选择过高的版本。
例如，你有以下工作空间成员：

```toml
[package]
name = "a"
rust-version = "1.62"

[dependencies]
clap = "4.2"  # 解析为 4.5.20

[package]
name = "b"

[dependencies]
clap = "4.5"  # 解析为 4.5.20
```

尽管每个包对 `clap` 的版本要求都满足其自身的 Rust 版本，但由于 [版本统一](#version-numbers)，解析器需要选择一个适用于两种情况的版本，那将是像 4.5.20 这样的版本。

[Rust 版本]: rust-version.md
[`resolver.incompatible-rust-versions`]: config.md#resolverincompatible-rust-versions

### 特性 {#features}

为了生成 `Cargo.lock`，解析器会假设 [workspace] 所有成员的所有 [特性] 都已启用来构建依赖图。这确保当通过 [`--features` 命令行标志] 添加或移除特性时，任何可选依赖都是可用的，并正确地与图的其他部分一起解析。解析器会运行第二次以确定在*编译* crate 时实际使用的特性，基于在命令行上选择的特性。

依赖项的解析会启用所有包在其上启用的所有特性的并集。例如，如果一个包依赖 [`im`] 包时启用了 [`serde` 依赖]，而另一个包依赖它时启用了 [`rayon` 依赖]，那么 `im` 将启用这两个特性进行构建，并且 `serde` 和 `rayon` crate 将包含在解析图中。如果没有包依赖启用了这些特性的 `im`，那么这些可选依赖将被忽略，并且它们不会影响解析。

当在工作空间中构建多个包时（例如使用 `--workspace` 或多个 `-p` 标志），所有这些包的依赖的特性将被统一。如果你遇到希望避免不同工作空间成员之间这种统一的情况，你需要通过单独的 `cargo` 调用来构建它们。

解析器将跳过缺少所需特性的包版本。例如，如果一个包依赖于版本 `^1` 的 [`regex`] 并启用了 [`perf` 特性]，那么它可以选择的最旧版本是 `1.3.0`，因为比这更早的版本不包含 `perf` 特性。同样，如果某个特性在新版本中被移除，那么需要该特性的包将停留在包含该特性的旧版本上。不鼓励在 SemVer 兼容的版本中移除特性。请注意，可选依赖也定义了一个隐式特性，因此移除可选依赖或使其变为非可选依赖可能会导致问题，请参阅 [移除可选依赖]。

[`im`]: https://crates.io/crates/im
[`perf` 特性]: https://github.com/rust-lang/regex/blob/1.3.0/Cargo.toml#L56
[`rayon` 依赖]: https://github.com/bodil/im-rs/blob/v15.0.0/Cargo.toml#L47
[`regex`]: https://crates.io/crates/regex
[`serde` 依赖]: https://github.com/bodil/im-rs/blob/v15.0.0/Cargo.toml#L46
[特性]: features.md
[移除可选依赖]: semver.md#cargo-remove-opt-dep
[workspace]: workspaces.md
[`--features` 命令行标志]: features.md#command-line-feature-options

#### 特性解析器版本 2 {#feature-resolver-version-2}

当在 `Cargo.toml` 中指定 `resolver = "2"`（请参阅下面的 [解析器版本](#resolver-versions)）时，将使用不同的特性解析器，该解析器使用不同的算法来统一特性。版本 `"1"` 的解析器无论在哪里指定都会为包统一特性。版本 `"2"` 的解析器会在以下情况下避免统一特性：

* 目标特定依赖的特性在未构建该目标时不会被启用。例如：

  ```toml
  [dependencies.common]
  version = "1.0"
  features = ["f1"]

  [target.'cfg(windows)'.dependencies.common]
  version = "1.0"
  features = ["f2"]
  ```

  为非 Windows 平台构建此示例时，`f2` 特性将*不会*被启用。

* 在 [build-dependencies] 或 proc-macros 上启用的特性，当这些相同的依赖被用作普通依赖时，将不会被统一。例如：

  ```toml
  [dependencies]
  log = "0.4"

  [build-dependencies]
  log = {version = "0.4", features=['std']}
  ```

  构建构建脚本时，`log` crate 将启用 `std` 特性构建。构建你的包的库时，则不会启用该特性。

* 在 [dev-dependencies] 上启用的特性，当这些相同的依赖被用作普通依赖时，将不会被统一，除非这些开发依赖当前正在被构建。例如：

  ```toml
  [dependencies]
  serde = {version = "1.0", default-features = false}

  [dev-dependencies]
  serde = {version = "1.0", features = ["std"]}
  ```

  在此示例中，该库通常会链接到不含 `std` 特性的 `serde`。但是，当构建为测试或示例时，它将包含 `std` 特性。例如，`cargo test` 或 `cargo build --all-targets` 将统一这些特性。请注意，依赖中的 dev-dependencies 总是被忽略，这仅与顶级包或工作空间成员相关。

[build-dependencies]: specifying-dependencies.md#build-dependencies
[dev-dependencies]: specifying-dependencies.md#development-dependencies

### `links` {#links}

[`links` field] 用于确保只有一个本机库副本被链接到二进制文件中。解析器将尝试找到一个满足每个 `links` 名称只有一个实例的图。如果无法找到满足该约束的图，它将返回错误。

例如，如果一个包依赖于 [`libgit2-sys`] 版本 `0.11`，另一个包依赖于 `0.12`，这就是一个错误，因为 Cargo 无法统一它们，但它们都链接到 `git2` 本机库。由于此要求，如果你的库被广泛使用，那么在使用 `links` 字段进行 SemVer 不兼容的版本发布时，应格外小心。

[`links` field]: manifest.md#the-links-field
[`libgit2-sys`]: https://crates.io/crates/libgit2-sys

### 撤回版本 {#yanked-versions}

[撤回发布]（yanked）是指被标记为不应使用的发布。解析器在构建图时将忽略所有撤回的发布，除非它们已存在于 `Cargo.lock` 文件中，或由 `cargo update` 的 [`--precise`] 标志（仅 nightly）显式请求。

[撤回发布]: publishing.md#cargo-yank
[`--precise`]: ../commands/cargo-update.md#option-cargo-update---precise

## 依赖更新 {#dependency-updates}

依赖解析由所有需要了解依赖图的 Cargo 命令自动执行。例如，[`cargo build`] 将运行解析器以发现所有需要构建的依赖项。第一次运行后，结果存储在 `Cargo.lock` 文件中。后续命令将运行解析器，*在可能的情况下*将依赖锁定在 `Cargo.lock` 中的版本。

如果 `Cargo.toml` 中的依赖列表被修改，例如将依赖版本从 `1.0` 更改为 `2.0`，那么解析器将为该依赖项选择符合新要求的版本。如果新依赖引入了新的要求，这些新要求也可能触发额外的更新。`Cargo.lock` 文件将更新为新结果。`--locked` 或 `--frozen` 标志可用于更改此行为，防止在需求改变时自动更新，而是返回错误。

[`cargo update`] 可用于在新版本发布时更新 `Cargo.lock` 中的条目。不带任何选项时，它将尝试更新锁文件中的所有包。`-p` 标志可用于将更新目标锁定到特定包，其他标志如 `--recursive` 或 `--precise` 可用于控制版本的选择方式。

[`cargo build`]: ../commands/cargo-build.md
[`cargo update`]: ../commands/cargo-update.md

## 覆盖 {#overrides}

Cargo 有几种机制来覆盖图中的依赖项。[覆盖依赖] 章节详细介绍了如何使用覆盖。覆盖作为注册表的覆盖层出现，将修补的版本替换为新条目。除此之外，解析过程与正常情况相同。

[覆盖依赖]: overriding-dependencies.md

## 依赖种类 {#dependency-kinds}

包中有三种依赖：普通依赖、[构建依赖] 和 [开发依赖]。从解析器的角度来看，这些在大多数情况下都被视为相同的。一个区别是，非工作空间成员的开发依赖总是被忽略，并且不影响解析。

具有 `[target]` 表的 [平台特定依赖] 在解析时被当作所有平台都启用来处理。换句话说，解析器会忽略平台或 `cfg` 表达式。

[构建依赖]: specifying-dependencies.md#build-dependencies
[开发依赖]: specifying-dependencies.md#development-dependencies
[平台特定依赖]: specifying-dependencies.md#platform-specific-dependencies

### 开发依赖循环 {#dev-dependency-cycles}

通常解析器不允许图中出现循环，但 [开发依赖] 允许循环。例如，项目 "foo" 有一个对 "bar" 的开发依赖，而 "bar" 有一个对 "foo" 的普通依赖（通常是通过 "path" 依赖）。这是允许的，因为从构建产物的角度来看并没有真正的循环。在这个例子中，首先构建 "foo" 库（它不需要 "bar"，因为 "bar" 仅用于测试），然后构建 "bar"（它依赖于 "foo"），然后构建 "foo" 的测试并链接到 "bar"。

请注意，这可能导致令人困惑的错误。在构建库单元测试的情况下，实际上有两个库副本被链接到最终的测试二进制文件中：一个链接了 "bar" 的副本，以及包含单元测试的构建副本。类似于 [版本不兼容的风险] 部分中强调的问题，两者之间的类型不兼容。在这种情况下，如果从 "bar" 中暴露 "foo" 的类型，需要特别小心，因为 "foo" 的单元测试不会将它们视为本地类型。

如果可能，请尝试将包拆分为多个包，并重新组织以使其保持严格的无环结构。

## 解析器版本 {#resolver-versions}

可以通过 `Cargo.toml` 中的解析器版本指定不同的解析行为，如下所示：

```toml
[package]
name = "my-package"
version = "1.0.0"
resolver = "2"
```
- `"1"`（默认值）
- `"2"` （[`edition = "2021"`] 的默认值）：引入了 [特性统一](#features) 方面的更改。更多详情请参阅 [features 章节][features-2]。
- `"3"` （[`edition = "2024"`] 的默认值，需要 Rust 1.84+）：将 [`resolver.incompatible-rust-versions`] 的默认值从 `allow` 更改为 `fallback`

解析器是一个影响整个工作空间的全局选项。依赖中的 `resolver` 版本会被忽略，只有顶级包中的值会被使用。如果使用 [虚拟工作空间]，版本应在 `[workspace]` 表中指定，例如：

```toml
[workspace]
members = ["member1", "member2"]
resolver = "2"
```

> **MSRV:** 要求 Rust 1.51+

[虚拟工作空间]: workspaces.md#virtual-workspace
[features-2]: features.md#feature-resolver-version-2
[`edition = "2021"`]: manifest.md#the-edition-field
[`edition = "2024"`]: manifest.md#the-edition-field

## 建议 {#recommendations}

以下是为你的包设置版本以及指定依赖要求的一些建议。这些都是适用于常见情况的通用指南，当然某些情况可能需要指定不寻常的要求。

* 在决定如何更新版本号，以及是否需要做出 SemVer 不兼容的版本变更时，请遵循 [SemVer 指南]。
* 在大多数情况下，对依赖使用脱字符号要求，例如 `"1.2.3"`。这确保了解析器在选择版本时能够最大程度地灵活，同时保持构建兼容性。
  * 使用当前使用的版本号指定所有三个组件。这有助于设置将使用的最低版本，并确保其他用户不会最终使用可能缺少你的包所需内容的较旧依赖版本。
  * 避免使用 `*` 要求，因为它在 [crates.io] 上是不允许的，并且在正常的 `cargo update` 过程中可能会引入破坏性的 SemVer 变更。
  * 避免过于宽泛的版本要求。例如，`>=2.0.0` 可能会引入任何 SemVer 不兼容的版本，如 `5.0.0`，这可能在将来导致构建失败。
  * 如果可能，避免过于严格的版本要求。例如，如果你指定一个波浪号要求如 `bar="~1.3"`，而另一个包指定要求 `bar="1.4"`，这将无法解析，尽管次要版本应该是兼容的。
* 尝试保持依赖版本与实际库所需的最低版本同步。例如，如果你有一个 `bar="1.0.12"` 的要求，然后在未来的发布中，你开始使用 "bar" 的 `1.1.0` 版中新增的特性，请将你的依赖要求更新为 `bar="1.1.0"`。

  如果你未能做到这一点，可能不会立即表现出来，因为当你运行全面的 `cargo update` 时，Cargo 可能会机会主义地选择最新版本。但是，如果另一个用户依赖于你的库，并运行 `cargo update your-library`，如果 "bar" 在他们的 `Cargo.lock` 中被锁定，它不会自动更新 "bar"。仅当依赖声明也被更新时，它才会在此情况下更新 "bar"。未能更新依赖声明可能会给使用 `cargo update your-library` 的用户造成混乱的构建错误。
* 如果两个包紧密结合，那么 `=` 依赖要求可能有助于确保它们保持同步。例如，一个库与其配套的 proc-macro 库有时会在这两个库之间做出假设，如果两者不同步（并且从不期望独立使用这两个库），这些假设将无法正常工作。父库可以在 proc-macro 上使用 `=` 要求，并重新导出宏以便于访问。
* `0.0.x` 版本可用于永久不稳定的包。

一般来说，依赖要求越严格，解析器失败的可能性就越大。相反，如果要求过于宽松，则可能发布破坏构建的新版本。

[SemVer 指南]: semver.md
[crates.io]: https://crates.io/

## 故障排除 {#troubleshooting}

以下说明了一些你可能遇到的问题及一些可能的解决方案。

### 为什么包含了某个依赖？ {#why-was-a-dependency-included}

假设你在 `cargo check` 的输出中看到依赖 `rand`，但你认为它不需要，想了解为什么它被引入了。

你可以运行：

```console
$ cargo tree --workspace --target all --all-features --invert rand
rand v0.8.5
└── ...

rand v0.8.5
└── ...
```

### 为什么在这个依赖上启用了那个特性？ {#why-was-that-feature-on-this-dependency-enabled}

你可能会识别出是激活的某个特性导致了 `rand` 的出现。**要找出是哪个包激活了该特性，你可以添加 `--edges features` 参数**：

```console
$ cargo tree --workspace --target all --all-features --edges features --invert rand
rand v0.8.5
└── ...

rand v0.8.5
└── ...
```

### 意外的依赖重复 {#unexpected-dependency-duplication}

当你运行以下命令时看到 `rand` 的多个实例：

```console
$ cargo tree --workspace --target all --all-features --duplicates
rand v0.7.3
└── ...

rand v0.8.5
└── ...
```

解析器算法收敛于一个包含两个依赖副本的解决方案，而一个副本就足够了。例如：

```toml
# Package A
[dependencies]
rand = "0.7"

# Package B
[dependencies]
rand = ">=0.6"  # 注意：不鼓励使用这种开放式要求
```

在此示例中，Cargo 可能构建两个 `rand` crate 的副本，即使单个 `0.7.3` 版本的副本就能满足所有要求。这是因为解析器的算法倾向于为包 B 构建可用的最新 `rand` 版本，即本文撰写时的 `0.8.5`，而该版本与包 A 的规范不兼容。解析器算法目前不会尝试在这种情况下“去重”。

在 Cargo 中不鼓励使用像 `>=0.6` 这样的开放式版本要求。但是，如果你遇到这种情况，可以使用带有 `--precise` 标志的 [`cargo update`] 命令手动删除重复项。

### 为什么没有选择较新的版本？ {#why-wasnt-a-newer-version-selected}

假设你注意到运行以下命令时没有选择某个依赖的最新版本：

```console
$ cargo update
```

你可以启用一些额外的日志记录来查看发生这种情况的原因：

```console
$ env CARGO_LOG=cargo::core::resolver=trace cargo update
```

**注意：** Cargo 的日志目标（target）和级别可能会随时间变化。

### 破坏 SemVer 的补丁发布破坏了构建 {#semver-breaking-patch-release-breaks-the-build}

有时，一个项目可能会无意中发布一个包含 SemVer 破坏性更改的补丁版本。当用户使用 `cargo update` 更新时，他们将获取到这个新版本，然后他们的构建可能会中断。在这种情况下，建议项目 [撤回] 该发布，并要么移除 SemVer 破坏性更改，要么将其作为新的 SemVer 主版本递增发布。

如果更改发生在第三方项目中，如果可能，请尝试（礼貌地！）与项目合作解决问题。

在等待发布被撤回期间，一些解决方法取决于具体情况：

* 如果你的项目是最终产品（如二进制可执行文件），只需避免在 `Cargo.lock` 中更新有问题的包。这可以通过 [`cargo update`] 的 `--precise` 标志实现。
* 如果你在 [crates.io] 上发布二进制文件，则可以临时添加一个 `=` 要求，强制依赖到一个特定的可用版本。
  * 二进制项目也可以建议用户使用 `--locked` 标志配合 [`cargo install`] 来使用包含已知可用版本的原始 `Cargo.lock`。
* 库也可以考虑发布一个临时的新版本，使用更严格的要求来避免麻烦的依赖。你可能需要考虑使用范围要求（而不是 `=`）来避免过于严格的要求，这些要求可能会与其他使用相同依赖的包发生冲突。一旦问题解决，你可以发布另一个补丁版本，将依赖放宽回脱字符号要求。
* 如果看起来第三方项目无法或不愿撤回发布，那么一种选择是更新你的代码以兼容更改，并将依赖要求的最低版本设置为新发布。你还需要考虑这是否是你自己库的 SemVer 破坏性更改，例如，如果它暴露了来自该依赖的类型。

[`cargo install`]: ../commands/cargo-install.md