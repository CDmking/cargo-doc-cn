# Cargo 文档

[![LICENSE-MIT](https://img.shields.io/badge/license-MIT-green)](https://raw.githubusercontent.com/rust-lang-cn/cargo-cn/master/LICENSE-MIT)
[![LICENSE-APACHE](https://img.shields.io/badge/license-Apache%202-blue)](https://raw.githubusercontent.com/rust-lang-cn/cargo-cn/master/LICENSE-APACHE)
![rustwiki.org](https://img.shields.io/website?up_message=rustwiki.org&url=https%3A%2F%2Frustwiki.org)

> Chinese translation of the [Cargo documentation][cargo-doc]
>
> 中文译注：
>
> 1. Cargo 文档源码主要包含两部分，一是《Cargo 手册》，二是《Cargo 帮助手册》。
> 2. 由于 [cargo-cn] 与原版差异较大，更新较难，现从 0.94.0 版本，即 [083ac51] 起翻译。
> 3. 我们怀着朴素的理想，努力将 Rust 资源翻译成中文，让中文世界中拥有更多官方的 Rust 资源。
> 4. 目前，由于工作量较大，翻译任务均采用 DeepSeek-V3.2 + [术语表] + 人工（粗略）校对。

[cargo-doc]: https://github.com/rust-lang/cargo/tree/master/src/doc
[cargo-cn]: https://github.com/rust-lang-cn/cargo-cn
[083ac51]: https://github.com/rust-lang/cargo/commit/083ac5135f967fd9dc906ab057a2315861c7a80d
[术语表]: https://github.com/rust-lang-cn/english-chinese-glossary-of-rust

本目录包含 Cargo 的文档，包含两部分，一是使用 [mdbook] 构建的[《Cargo 手册》][The Cargo Book]，二是使用 [mdman] 构建的帮助手册（man 手册）。

> 注：帮助手册暂未翻译成中文版。

[The Cargo Book]: https://doc.rust-lang.org/cargo/
[mdBook]: https://github.com/rust-lang/mdBook
[mdman]: https://github.com/rust-lang/cargo/tree/master/crates/mdman/

<details>
<summary>使用 AI 翻译时的提示词示例</summary>
<div>
<pre>
<code>

    你是一位技术文档译者，专业翻译各类技术文档。请严格遵循以下准则进行翻译：

    **1. 保持格式与锚点**
    *   保持所有 Markdown 格式和文档结构（如标题、列表、表格、代码块、链接），仅翻译文本内容。
    *   **锚点处理**：为确保标题翻译后锚点仍然有效，请在翻译后的标题后添加原英文标题的锚点ID。格式为 `{#original-anchor-id}`。
        *   **规则**：将原标题转换为小写，空格替换为连字符，移除标点，作为锚点ID。
        *   **示例**：
            *   标题`### First Steps with Cargo` 应翻译为 `### 初次使用 Cargo {#first-steps-with-cargo}`
            *   标题`## 2. Installation & Setup` 应翻译为 `## 2. 安装与设置 {#2-installation--setup}`
            *   标题`## Using `[patch]` with multiple versions` 应翻译为 `## 将 `[patch]` 用于多个版本 {#using-patch-with-multiple-versions}`

    **2. 保持原文**
    *   **所有代码**（代码块、内联代码、命令、路径、文件名）、**技术术语**（函数名、类名、变量名、API端点、关键字）以及**专有名词**（品牌、产品、项目名称）必须保留英文原   文，**不得翻译**。

    **3. 术语一致性**
    *   技术术语和核心概念的翻译必须准确、一致，并符合中文技术社区的常用表达。若遇到尚无通用译法或不确定的术语，可在首次出现时以括号附上英文原词（例如：“……通过一个抽象（abstraction）来实现。”）。

    **4. 表达风格**
    *   译文应简洁、流畅、客观，符合技术文档的风格。直译优先，避免过度意译或文学化表达，确保信息传递无损耗。

    **5. 遵循原文指令**
    *   如果原文中已通过注释（如 `<!-- 请勿翻译 -->`）或说明明确指示某部分无需翻译或采用特定处理方式，请严格遵循。

    你的最终目标是产出**格式完整、链接有效、术语准确、表达专业**的中文技术文档。请直接输出翻译后的完整内容。

    (后跟术语表)

</code>
</pre>
</div>
</details>

### 构建

构建书籍需要 [mdBook]。获取安装：

```console
$ cargo install mdbook
```

构建本书：

```console
$ mdbook build
```

`mdbook` 提供了各种不同的命令和选项来帮助你对书本进行操作：

* `mdbook build --open`：构建书本并在 Web 浏览器中打开。
* `mdbook serve`：在本地主机上启动 Web 服务器。每当任何文件更改时，它也会自动重建书籍，并自动重新加载 Web 浏览器。

书籍文件和目录由 [`SUMMARY.md`](src/SUMMARY.md) 确定，并且每个文件都必须在此处给出。

### 构建帮助手册页

帮助手册页使用名为 [mdman] 的工具将标记转换为手册页格式。 有关更多详细信息，请查阅位于 *cargo* 仓库（官方的 Cargo 工具仓库）中的 [`mdman/doc/`][nmman-doc] 文档。

手册页从 Markdown 模板（位于 [`src/etc/man/`][man-doc] 目录中）转换为三种不同的格式：

1. Troff 样式的手册页，保存在 `src/etc/man/` 目录中。
2. 《Cargo 手册》中的 Markdown 文件（包含一些 HTML），保存在 `src/doc/src/commands/` 目录中。
3. 纯文本（在Windows中没有人的平台上的嵌入式手册页需要），保存在 `src/doc/man/generated_txt/` 目录中。

要重新构建手册页，请build-man.sh在src/doc目录中运行脚本。

```console
$ ./build-man.sh
```

[nmman-doc]: https://github.com/rust-lang/cargo/tree/master/crates/mdman
[man-doc]: https://github.com/rust-lang/cargo/tree/master/src/etc/man

### 语义化版本章节测试

有一个脚本可以验证语义化版本章节中的示例是否按预期工作。要运行测试，请进入 `semver-check` 目录并运行 `cargo run`。

### 参与贡献

我们很乐意为您提供改善文档的帮助！您可以随时提出有关任何问题的信息，并发送 P R来解决您要修复或更改的问题。如果您的更改很大，请先打开一个 Issue，这样我们才能确保在您完成对应的 PR 的工作之前，我们会接受这一点。

### 许可协议

《Cargo 手册》和《Cargo 帮助手册》按照 MIT 许可证和 Apache 2.0 许可证进行授权。

有关详细信息，请参见 [LICENSE-APACHE](LICENSE-APACHE) 和 [LICENSE-MIT](LICENSE-MIT)。
