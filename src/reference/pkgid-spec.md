# 包 ID 规范 {#package-id-specifications}

## 包 ID 规范 {#package-id-specifications}

Cargo 的子命令经常需要引用依赖图中的特定包以进行各种操作，如更新、清理、构建等。为了解决这个问题，Cargo 支持*包 ID 规范*。规范是一个字符串，用于唯一地引用包图中的某个包。

规范可以是完全限定的，例如 `registry+https://github.com/rust-lang/crates.io-index#regex@1.4.3`，也可以是缩写的，例如 `regex`。只要缩写形式能唯一标识依赖图中的单个包，就可以使用缩写形式。如果存在歧义，可以添加额外的限定符使其唯一。例如，如果图中有两个版本的 `regex` 包，则可以使用版本进行限定，使其唯一，例如 `regex@1.4.3`。

Cargo 输出的包 ID 规范，例如在 [cargo metadata](../commands/cargo-metadata.md) 输出中，是完全限定的。

### 规范语法 {#specification-grammar}

包 ID 规范的形式语法如下：

```notrust
spec := pkgname |
        [ kind "+" ] proto "://" hostname-and-path [ "?" query] [ "#" ( pkgname | semver ) ]
query = ( "branch" | "tag" | "rev" ) "=" ref
pkgname := name [ ("@" | ":" ) semver ]
semver := digits [ "." digits [ "." digits [ "-" prerelease ] [ "+" build ]]]

kind = "registry" | "git" | "path"
proto := "http" | "git" | "file" | ...
```

这里，方括号表示内容是可选的。

URL 形式可用于 git 依赖项，或用于区分来自不同源（如不同注册表）的包。

### 规范示例 {#example-specifications}

以下是对 `crates.io` 上 `regex` 包的引用：

| 规范                                                              | 名称    | 版本 |
|:------------------------------------------------------------------|:-------:|:-------:|
| `regex`                                                           | `regex` | `*`     |
| `regex@1.4`                                                       | `regex` | `1.4.*` |
| `regex@1.4.3`                                                     | `regex` | `1.4.3` |
| `https://github.com/rust-lang/crates.io-index#regex`              | `regex` | `*`     |
| `https://github.com/rust-lang/crates.io-index#regex@1.4.3`        | `regex` | `1.4.3` |
| `registry+https://github.com/rust-lang/crates.io-index#regex@1.4.3` | `regex` | `1.4.3` |

以下是一些针对不同 git 依赖项的规范示例：

| 规范                                                       | 名称             | 版本  |
|:-----------------------------------------------------------|:----------------:|:--------:|
| `https://github.com/rust-lang/cargo#0.52.0`                | `cargo`          | `0.52.0` |
| `https://github.com/rust-lang/cargo#cargo-platform@0.1.2`  | <nobr>`cargo-platform`</nobr> | `0.1.2`  |
| `ssh://git@github.com/rust-lang/regex.git#regex@1.4.3`     | `regex`          | `1.4.3`  |
| `git+ssh://git@github.com/rust-lang/regex.git#regex@1.4.3` | `regex`          | `1.4.3`  |
| `git+ssh://git@github.com/rust-lang/regex.git?branch=dev#regex@1.4.3` | `regex`          | `1.4.3`  |

文件系统上的本地包可以使用 `file://` URL 来引用：

| 规范                                        | 名称  | 版本 |
|:--------------------------------------------|:-----:|:-------:|
| `file:///path/to/my/project/foo`            | `foo` | `*`     |
| `file:///path/to/my/project/foo#1.1.8`      | `foo` | `1.1.8` |
| `path+file:///path/to/my/project/foo#1.1.8` | `foo` | `1.1.8` |

### 规范的简洁性 {#brevity-of-specifications}

此规范的目标是提供简洁而详尽的语法来引用依赖图中的包。模糊的引用可能指向一个或多个包。如果同一规范可能引用多个包，大多数命令会生成错误。