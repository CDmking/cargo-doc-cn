# Cargo.toml vs Cargo.lock {#cargotoml-vs-cargolock}

`Cargo.toml` 和 `Cargo.lock` 服务于两个不同的目的。在我们讨论它们之前，先做一个总结：

* `Cargo.toml` 是从广义上描述您的依赖关系，由您编写。
* `Cargo.lock` 包含关于您的依赖关系的精确信息。它由 Cargo 维护，不应手动编辑。

如有疑问，请将 `Cargo.lock` 提交到版本控制系统（例如 Git）。为了更好地理解原因以及可能的替代方案，请参见常见问题解答中的[“为什么要在版本控制中包含 Cargo.lock？”](../faq.md#why-have-cargolock-in-version-control)。我们建议将此与[验证最新依赖项](continuous-integration.md#verifying-latest-dependencies)结合使用。

让我们更深入地探讨一下。

`Cargo.toml` 是一个[**清单**][def-manifest]文件，您可以在其中指定关于您的包的大量不同元数据。例如，您可以说您依赖于另一个包：

```toml
[package]
name = "hello_world"
version = "0.1.0"

[dependencies]
regex = { git = "https://github.com/rust-lang/regex.git" }
```

这个包有一个依赖项，即 `regex` 库。在这个例子中，它声明依赖于一个特定的 Git 仓库，该仓库位于 GitHub 上。由于您没有指定任何其他信息，Cargo 假设您打算使用默认分支上的最新提交来构建我们的包。

听起来不错？但是，有一个问题：如果您今天构建这个包，然后将其副本发送给我，而我明天构建这个包，可能会发生一些不好的事情。在此期间 `regex` 可能会有更多的提交，我的构建将包含新的提交，而您的则不会。因此，我们会得到不同的构建。这将很糟糕，因为我们希望构建是可重现的。

您可以通过在我们的 `Cargo.toml` 中定义一个特定的 `rev` 值来解决这个问题，这样 Cargo 就可以知道在构建包时确切使用哪个修订版本：

```toml
[dependencies]
regex = { git = "https://github.com/rust-lang/regex.git", rev = "9f9f693" }
```

现在我们的构建将保持一致。但是有一个很大的缺点：现在每次想要更新我们的库时，您都必须手动考虑 SHA-1。这既繁琐又容易出错。

这就是 `Cargo.lock` 的作用。由于它的存在，您无需手动跟踪确切的修订版本：Cargo 会为您完成。当您有这样的清单时：

```toml
[package]
name = "hello_world"
version = "0.1.0"

[dependencies]
regex = { git = "https://github.com/rust-lang/regex.git" }
```

当您第一次构建时，Cargo 将获取最新的提交并将该信息写入您的 `Cargo.lock`。该文件将如下所示：

```toml
[[package]]
name = "hello_world"
version = "0.1.0"
dependencies = [
 "regex 1.5.0 (git+https://github.com/rust-lang/regex.git#9f9f693768c584971a4d53bc3c586c33ed3a6831)",
]

[[package]]
name = "regex"
version = "1.5.0"
source = "git+https://github.com/rust-lang/regex.git#9f9f693768c584971a4d53bc3c586c33ed3a6831"
```

您可以看到这里有更多的信息，包括用于构建的确切修订版本。现在当您将您的包发送给其他人时，他们将使用完全相同的 SHA，即使您没有在 `Cargo.toml` 中指定它。

当您准备好选择新版本的库时，Cargo 可以重新计算依赖关系并为您更新：

```console
$ cargo update         # 更新所有依赖
$ cargo update regex   # 仅更新“regex”
```

这将写出一个新的 `Cargo.lock`，其中包含新的版本信息。请注意，`cargo update` 的参数实际上是一个[包标识符规范](../reference/pkgid-spec.md)，而 `regex` 只是一个简短的规范。

[def-manifest]: ../appendix/glossary.md#manifest '"清单" (glossary entry)'
[def-package]: ../appendix/glossary.md#package '"包" (glossary entry)'