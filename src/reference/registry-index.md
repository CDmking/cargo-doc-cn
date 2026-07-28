# 索引格式 {#index-format}

以下定义了索引的格式。新功能会偶尔添加，只有引入这些功能的 Cargo 版本才能理解它们。较旧版本的 Cargo 可能无法使用使用了新功能的包。但是，旧版包的格式不应改变，因此较旧版本的 Cargo 应该能够使用它们。

## 索引配置 {#index-configuration}
索引的根目录包含一个名为 `config.json` 的文件，其中包含 Cargo 用于访问注册中心的 JSON 信息。以下是 [crates.io] 配置文件的一个示例：

```javascript
{
    "dl": "https://crates.io/api/v1/crates",
    "api": "https://crates.io"
}
```

这些键的含义如下：
- `dl`：这是下载索引中列出的 crate 的 URL。该值可以包含以下标记，这些标记将被替换为相应的值：

  - `{crate}`：crate 的名称。
  - `{version}`：crate 的版本。
  - `{prefix}`：根据 crate 名称计算出的目录前缀。例如，名为 `cargo` 的 crate 的前缀为 `ca/rg`。详见下文。
  - `{lowerprefix}`：`{prefix}` 的小写变体。
  - `{sha256-checksum}`：crate 的 sha256 校验和。

  如果不存在任何标记，则会将 `/{crate}/{version}/download` 追加到末尾。
- `api`：Web API 的基础 URL。此键是可选的，但如果未指定，则 [`cargo publish`] 等命令将无法工作。Web API 将在下文描述。
- `auth-required`：指示这是否是一个私有注册中心，要求所有操作都需要认证，包括 API 请求、crate 下载和稀疏索引更新。

## 下载端点 {#download-endpoint}
下载端点应发送所请求包的 `.crate` 文件。Cargo 支持 https、http 和 file URL、HTTP 重定向、HTTP1 和 HTTP2。TLS 支持的具体细节取决于 Cargo 运行所在的平台、Cargo 的版本及其编译方式。

如果在 `config.json` 中设置了 `auth-required: true`，则在 http(s) 下载请求中将包含 `Authorization` 头。

## 索引文件 {#index-files}
索引仓库的其余部分包含每个包一个文件，文件名是包名称的小写形式。包的每个版本在文件中各占一行。这些文件按目录层级组织：

- 1 个字符名称的包放置在名为 `1` 的目录中。
- 2 个字符名称的包放置在名为 `2` 的目录中。
- 3 个字符名称的包放置在目录 `3/{first-character}` 中，其中 `{first-character}` 是包名称的第一个字符。
- 所有其他包存储在名为 `{first-two}/{second-two}` 的目录中，其中顶层目录是包名称的前两个字符，下一个子目录是包名称的第三和第四个字符。例如，`cargo` 将存储在名为 `ca/rg/cargo` 的文件中。

> 注意：虽然索引文件名是小写的，但 `Cargo.toml` 和索引 JSON 数据中包含包名称的字段是区分大小写的，并且可能包含大写和小写字符。

上述目录名称是基于转换为小写后的包名称计算的；它由标记 `{lowerprefix}` 表示。当使用原始包名称而不进行大小写转换时，生成的目录名称由标记 `{prefix}` 表示。例如，包 `MyCrate` 的 `{prefix}` 为 `My/Cr`，`{lowerprefix}` 为 `my/cr`。通常，建议使用 `{prefix}` 而非 `{lowerprefix}`，但每种选择各有利弊。在大小写不敏感的文件系统上使用 `{prefix}` 会导致（无害但不优雅的）目录别名。例如，`crate` 和 `CrateTwo` 的 `{prefix}` 值分别为 `cr/at` 和 `Cr/at`；它们在 Unix 机器上是不同的，但在 Windows 上会别名为同一目录。使用规范化大小写的目录可以避免别名，但在大小写敏感的文件系统上，较难支持缺少 `{prefix}`/`{lowerprefix}` 的旧版 Cargo。例如，nginx 重写规则可以轻松构建 `{prefix}`，但无法执行大小写转换来构建 `{lowerprefix}`。

## 名称限制 {#name-restrictions}

注册中心应考虑对添加到其索引中的包名称实施限制。Cargo 本身允许包含任何 [alphanumeric]、`-` 或 `_` 字符的名称。[crates.io] 施加了其自身的限制，包括以下内容：

- 仅允许 ASCII 字符。
- 仅允许字母数字、`-` 和 `_` 字符。
- 第一个字符必须是字母。
- 大小写不敏感的冲突检测。
- 防止 `-` 与 `_` 的差异。
- 在特定长度以下（最大 64）。
- 拒绝保留名称，例如 Windows 特殊文件名如 "nul"。

注册中心应考虑纳入类似的限制，并考虑安全隐患，例如 [IDN 同形字攻击](https://en.wikipedia.org/wiki/IDN_homograph_attack)以及 [UTR36](https://www.unicode.org/reports/tr36/) 和 [UTS39](https://www.unicode.org/reports/tr39/) 中的其他问题。

## 版本唯一性 {#version-uniqueness}

索引*必须*确保每个版本在每个包中只出现一次。这包括忽略 SemVer 构建元数据。
例如，索引不得包含版本 `1.0.7` 和 `1.0.7+extra` 的两个条目。

## JSON 模式 {#json-schema}

包文件中的每一行包含一个 JSON 对象，描述该包的一个已发布版本。以下是一个带有注释的格式化示例，解释了条目的格式。

```javascript
{
    // 包的名称。
    // 必须仅包含字母数字、`-` 或 `_` 字符。
    "name": "foo",
    // 此行描述的包的版本。
    // 必须是根据 https://semver.org/ 上的语义化版本 2.0.0 规范的合法版本号。
    "vers": "0.1.0",
    // 包的直接依赖项数组。
    "deps": [
        {
            // 依赖项的名称。
            // 如果依赖项是从原始包名称重命名的，
            // 则这是新名称。原始包名称存储在
            // `package` 字段中。
            "name": "rand",
            // 此依赖项的 SemVer 要求。
            // 必须是在 https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html
            // 定义的合法版本要求。
            "req": "^0.6",
            // 为此依赖项启用的功能数组（字符串形式）。
            // 自 Cargo 1.84 起，如果未指定，默认为 `[]`。
            "features": ["i128_support"],
            // 布尔值，指示这是否是可选的依赖项。
            // 自 Cargo 1.84 起，如果未指定，默认为 `false`。
            "optional": false,
            // 布尔值，指示是否启用默认功能。
            // 自 Cargo 1.84 起，如果未指定，默认为 `true`。
            "default_features": true,
            // 依赖项的目标平台。
            // 如果未指定或为 `null`，则不是目标依赖项。
            // 否则为字符串，例如 "cfg(windows)"。
            "target": null,
            // 依赖项的类型。
            // "dev"、"build" 或 "normal"。
            // 如果未指定或为 `null`，则默认为 "normal"。
            "kind": "normal",
            // 此依赖项来源的注册中心索引的 URL
            // （字符串形式）。如果未指定或为 `null`，则假定
            // 该依赖项位于当前注册中心。
            "registry": null,
            // 如果依赖项被重命名，则为实际包名称的字符串。
            // 如果未指定或为 `null`，则此依赖项未被重命名。
            "package": null,
        }
    ],
    // `.crate` 文件的 SHA256 校验和。
    "cksum": "d867001db0e2b6e0496f9fac96930e2d42233ecd3ca0413e0753d4c7695d289c",
    // 为包定义的功能集合。
    // 每个功能映射到一个它启用的功能或依赖项数组。
    // 自 Cargo 1.84 起，如果未指定，默认为 `{}`。
    "features": {
        "extras": ["rand/simd_support"]
    },
    // 布尔值，指示此版本是否已被 yanked。
    "yanked": false,
    // 包清单中的 `links` 字符串值，如果未指定则为 null。
    // 此字段是可选的，默认为 null。
    "links": null,
    // 一个无符号 32 位整数值，指示此条目的模式版本。
    //
    // 如果未指定，则应解释为默认值 1。
    //
    // Cargo（从 1.51 版本开始）将忽略它无法识别的版本。
    // 这提供了一种安全引入索引条目变更的方法，并允许较旧版本的 cargo
    // 忽略它不理解的新条目。1.51 之前的版本会忽略此字段，
    // 因此可能会误解索引条目的含义。
    //
    // 当前值如下：
    //
    // * 1：本文档中描述的模式，不包括较新的添加。
    //      此版本在 Rust 1.51 及更新版本中受支持。
    // * 2：添加了 `features2` 字段。
    //      此版本在 Rust 1.60 及更新版本中受支持。
    "v": 2,
    // 此可选字段包含具有新的扩展语法的新功能。
    // 具体来说，是命名空间功能（`dep:`）和弱依赖（`pkg?/feat`）。
    //
    // 这与 `features` 分开，因为 1.19 之前的版本
    // 由于无法解析新语法而加载失败，即使有
    // `Cargo.lock` 文件也是如此。
    //
    // Cargo 将把此处列出的任何值与 "features" 字段合并。
    //
    // 如果包含此字段，则 "v" 字段应至少设置为 2。
    //
    // 注册中心不需要为此扩展功能语法使用此字段，
    // 它们可以将这些内容包含在 "features" 字段中。
    // 仅在注册中心想要支持 1.19 之前的 cargo 版本时才需要使用此字段，
    // 实际上只有 crates.io 会这样做，因为那些旧版本不支持其他注册中心。
    "features2": {
        "serde": ["dep:serde", "chrono?/serde"]
    }
    // 最低支持的 Rust 版本（可选）
    // 必须是有效的版本要求，不带运算符（例如不加 `=`）
    "rust_version": "1.60"
}
```

JSON 对象在添加后不应修改，但 `yanked` 字段的值可能随时更改。

> **注意**：索引 JSON 格式与 [Publish API] 和 [`cargo metadata`] 的 JSON 格式存在细微差异。
> 如果您使用其中一项作为生成索引条目的来源，建议仔细检查它们之间的文档差异。
>
> 对于 [Publish API]，差异如下：
>
> * `deps`
>     * `name` --- 当依赖项在 `Cargo.toml` 中被[重命名][renamed]时，发布 API 将原始包名称放在 `name` 字段中，将别名放在 `explicit_name_in_toml` 字段中。
>       索引将别名放在 `name` 字段中，将原始包名称放在 `package` 字段中。
>     * `req` --- 发布 API 中的此字段名为 `version_req`。
> * `cksum` --- 发布 API 不指定校验和，注册中心必须在添加到索引之前计算它。
> * `features` --- 某些功能可能放在 `features2` 字段中。
>   注意：这只是 [crates.io] 的遗留要求；其他注册中心不需要操心修改功能映射。
>   `v` 字段指示 `features2` 字段的存在。
> * 发布 API 包含其他几个字段，例如 `description` 和 `readme`，这些字段不会出现在索引中。
>   这些字段旨在使注册中心更容易获取关于 crate 的元数据以在网站上显示，而无需提取和解析 `.crate` 文件。
>   这些额外信息通常会添加到注册中心服务器的数据库中。
> * 虽然此处包含了 `rust_version`，但 [crates.io] 将忽略此字段，
>   而是从 `.crate` 文件中包含的 `Cargo.toml` 中读取它。
>
> 对于 [`cargo metadata`]，差异如下：
>
> * `vers` --- `cargo metadata` 中的此字段名为 `version`。
> * `deps`
>   * `name` --- 当依赖项在 `Cargo.toml` 中被[重命名][renamed]时，`cargo metadata` 将原始包名称放在 `name` 字段中，将别名放在 `rename` 字段中。
>     索引将别名放在 `name` 字段中，将原始包名称放在 `package` 字段中。
>   * `default_features` --- `cargo metadata` 中的此字段名为 `uses_default_features`。
>   * `registry` --- `cargo metadata` 使用值 `null` 表示依赖项来自 [crates.io]。
>     索引使用值 `null` 表示依赖项来自与索引相同的注册中心。
>     在创建索引条目时，非 [crates.io] 的注册中心应将值 `null` 转换为 `https://github.com/rust-lang/crates.io-index`，并将与当前索引匹配的 URL 转换为 `null`。
>   * `cargo metadata` 包含一些额外字段，例如 `source` 和 `path`。
> * 索引包含额外字段，例如 `yanked`、`cksum` 和 `v`。

[renamed]: specifying-dependencies.md#renaming-dependencies-in-cargotoml
[Publish API]: registry-web-api.md#publish
[`cargo metadata`]: ../commands/cargo-metadata.md

## 索引协议 {#index-protocols}
Cargo 支持两种远程注册中心协议：`git` 和 `sparse`。`git` 协议
将索引文件存储在 git 仓库中，而 `sparse` 协议通过 HTTP 获取单个文件。

### Git 协议 {#git-protocol}
git 协议在索引 URL 中没有协议前缀。例如，[crates.io] 的 git 索引 URL
为 `https://github.com/rust-lang/crates.io-index`。

Cargo 将 git 仓库缓存在磁盘上，以便能够高效地进行增量更新。

### 稀疏协议 {#sparse-protocol}
稀疏协议在注册中心 URL 中使用 `sparse+` 协议前缀。例如，
[crates.io] 的稀疏索引 URL 为 `sparse+https://index.crates.io/`。

稀疏协议使用单独的 HTTP 请求下载每个索引文件。由于这会产生大量的小型 HTTP 请求，
支持管道化和 HTTP/2 的服务器可以显著提升性能。

#### 稀疏认证 {#sparse-authentication}
在获取任何其他文件之前，Cargo 将尝试获取 `config.json` 文件。如果服务器响应 HTTP 401，
则 Cargo 将假定该注册中心需要认证，并重新尝试请求 `config.json`，同时包含认证令牌。

在认证失败（或缺少认证令牌）时，服务器可能会包含一个 `www-authenticate` 头，其中包含
`Cargo login_url="<URL>"` 挑战，以指示用户可以在何处获取令牌。

需要认证的注册中心必须在 `config.json` 中设置 `auth-required: true`。

#### 缓存 {#caching}
Cargo 缓存 crate 元数据文件，并捕获每个条目的 `ETag` 或 `Last-Modified` HTTP 头。在刷新 crate 元数据时，Cargo 发送 `If-None-Match` 或 `If-Modified-Since` 头，以允许服务器在本地缓存有效时响应 HTTP 304 "Not Modified"，从而节省时间和带宽。如果同时存在 `ETag` 和 `Last-Modified` 头，Cargo 仅使用 `ETag`。

#### 缓存失效 {#cache-invalidation}
如果注册中心使用了某种 CDN 或代理来缓存对索引文件的访问，则建议注册中心在文件更新时实现某种形式的缓存失效。如果这些缓存未更新，则用户在缓存清除之前可能无法访问新的 crate。

#### 不存在的 Crate {#nonexistent-crates}
对于不存在的 crate，注册中心应响应 404 "Not Found"、410 "Gone" 或 451 "Unavailable For Legal Reasons" 状态码。

#### 稀疏限制 {#sparse-limitations}
由于注册中心的 URL 存储在锁文件中，不建议同时提供两种协议的注册中心。关于迁移计划的讨论在 issue [#10964] 中。[crates.io] 注册中心是一个例外，因为 Cargo 在使用稀疏协议时会在内部替换为等效的 git URL。

如果注册中心确实提供两种协议，目前建议选择一种协议作为规范协议，并为另一种协议使用 [source replacement]。


[`cargo publish`]: ../commands/cargo-publish.md
[alphanumeric]: ../../std/primitive.char.html#method.is_alphanumeric
[crates.io]: https://crates.io/
[source replacement]: ../reference/source-replacement.md
[#10964]: https://github.com/rust-lang/cargo/issues/10964