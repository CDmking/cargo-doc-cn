# 依赖项 {#dependencies}

[crates.io] 是 Rust 社区的中心化 [*package registry*][def-package-registry]（包注册中心），作为发现和下载 [package][def-package]（包）的位置。`cargo` 默认配置为使用它来查找所需的包。

要依赖于托管在 [crates.io] 上的库，请将其添加到您的 `Cargo.toml` 中。

[crates.io]: https://crates.io/

## 添加依赖 {#adding-a-dependency}

如果您的 `Cargo.toml` 中还没有 `[dependencies]` 部分，请添加该部分，然后列出您想要使用的 [crate][def-crate]（包）名称和版本。此示例添加了对 `time` 包的依赖：

```toml
[dependencies]
time = "0.1.12"
```

版本字符串是一个 [SemVer] 版本要求。[指定依赖项](../reference/specifying-dependencies.md) 文档提供了有关此处可用选项的更多信息。

[SemVer]: https://semver.org

如果您还想添加对 `regex` 包的依赖，则无需为列出的每个包都添加 `[dependencies]`。以下是包含对 `time` 和 `regex` 包依赖项的整个 `Cargo.toml` 文件：

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"

[dependencies]
time = "0.1.12"
regex = "0.1.41"
```

重新运行 `cargo build`，Cargo 将获取新的依赖项及其所有依赖项，编译它们，并更新 `Cargo.lock`：

```console
$ cargo build
      Updating crates.io index
   Downloading memchr v0.1.5
   Downloading libc v0.1.10
   Downloading regex-syntax v0.2.1
   Downloading memchr v0.1.5
   Downloading aho-corasick v0.3.0
   Downloading regex v0.1.41
     Compiling memchr v0.1.5
     Compiling libc v0.1.10
     Compiling regex-syntax v0.2.1
     Compiling memchr v0.1.5
     Compiling aho-corasick v0.3.0
     Compiling regex v0.1.41
     Compiling hello_world v0.1.0 (file:///path/to/package/hello_world)
```

`Cargo.lock` 包含了所有这些依赖项具体使用哪个修订版本的精确信息。

现在，如果 `regex` 包更新了，您仍然会使用相同的修订版本进行构建，直到您选择运行 `cargo update`。

您现在可以在 `main.rs` 中使用 `regex` 库了。

```rust,ignore
use regex::Regex;

fn main() {
    let re = Regex::new(r"^\d{4}-\d{2}-\d{2}$").unwrap();
    println!("Did our date match? {}", re.is_match("2014-01-01"));
}
```

运行它将显示：

```console
$ cargo run
   Running `target/hello_world`
Did our date match? true
```

[def-crate]:             ../appendix/glossary.md#crate             '"crate"（词汇表条目）'
[def-package]:           ../appendix/glossary.md#package           '"package"（词汇表条目）'
[def-package-registry]:  ../appendix/glossary.md#package-registry  '"package-registry"（词汇表条目）'