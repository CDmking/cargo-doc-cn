# 特性示例 {#features-examples}

以下展示了一些实际使用中的功能特性示例。

## 最小化构建时间和文件大小 {#minimizing-build-times-and-file-sizes}

一些包使用特性，以便在特性未启用时减少 crate 的大小和编译时间。一些例子包括：

* [`syn`] 是一个用于解析 Rust 代码的流行 crate。由于它如此流行，减少编译时间对许多项目都有帮助。它有一个[明确记录的功能列表][syn-features]，可以用来最小化其包含的代码量。
* [`regex`] 有[几个功能][regex-features]，并且[文档记录良好][regex-docs]。去掉 Unicode 支持可以减少生成文件的大小，因为它可以移除一些大型表格。
* [`winapi`] 有[大量功能][winapi-features]，用于限制它支持的 Windows API 绑定。
* [`web-sys`] 是另一个类似于 `winapi` 的例子，它提供了[巨大的 API 绑定范围][web-sys-features]，通过使用功能来限制。

[`winapi`]: https://crates.io/crates/winapi
[winapi-features]: https://github.com/retep998/winapi-rs/blob/0.3.9/Cargo.toml#L25-L431
[`regex`]: https://crates.io/crates/regex
[`syn`]: https://crates.io/crates/syn
[syn-features]: https://docs.rs/syn/1.0.54/syn/#optional-features
[regex-features]: https://github.com/rust-lang/regex/blob/1.4.2/Cargo.toml#L33-L101
[regex-docs]: https://docs.rs/regex/1.4.2/regex/#crate-features
[`web-sys`]: https://crates.io/crates/web-sys
[web-sys-features]: https://github.com/rustwasm/wasm-bindgen/blob/0.2.69/crates/web-sys/Cargo.toml#L32-L1395

## 扩展行为 {#extending-behavior}

[`serde_json`] 包有一个 [`preserve_order` 功能][serde_json-preserve_order]，该功能[改变了 JSON 映射的行为][serde_json-code]，以保留键的插入顺序。请注意，它启用了一个可选依赖 [`indexmap`] 来实现新行为。

当像这样更改行为时，请小心确保更改是[语义化版本兼容的][SemVer compatible]。也就是说，启用该功能不应破坏通常在该功能关闭时构建的代码。

[`serde_json`]: https://crates.io/crates/serde_json
[serde_json-preserve_order]: https://github.com/serde-rs/json/blob/v1.0.60/Cargo.toml#L53-L56
[SemVer compatible]: features.md#semver-compatibility
[serde_json-code]: https://github.com/serde-rs/json/blob/v1.0.60/src/map.rs#L23-L26
[`indexmap`]: https://crates.io/crates/indexmap

## `no_std` 支持 {#no_std-support}

一些包希望同时支持 [`no_std`] 和 `std` 环境。这对于支持嵌入式资源受限平台非常有用，同时仍然允许支持完整标准库的平台具有扩展能力。

[`wasm-bindgen`] 包定义了一个默认启用的 [`std` 功能][wasm-bindgen-std][wasm-bindgen-default]。在库的顶部，它[无条件地启用 `no_std` 属性][wasm-bindgen-no_std]。这确保了 `std` 和 [`std` 预导入][`std` prelude]不会自动在作用域内。然后，在代码的各个地方（[示例1][wasm-bindgen-cfg1]，[示例2][wasm-bindgen-cfg2]），它使用 `#[cfg(feature = "std")]` 属性来有条件地启用需要 `std` 的额外功能。

[`no_std`]: ../../reference/names/preludes.html#the-no_std-attribute
[`wasm-bindgen`]: https://crates.io/crates/wasm-bindgen
[`std` prelude]: ../../std/prelude/index.html
[wasm-bindgen-std]: https://github.com/rustwasm/wasm-bindgen/blob/0.2.69/Cargo.toml#L25
[wasm-bindgen-default]: https://github.com/rustwasm/wasm-bindgen/blob/0.2.69/Cargo.toml#L23
[wasm-bindgen-no_std]: https://github.com/rustwasm/wasm-bindgen/blob/0.2.69/src/lib.rs#L8
[wasm-bindgen-cfg1]: https://github.com/rustwasm/wasm-bindgen/blob/0.2.69/src/lib.rs#L270-L273
[wasm-bindgen-cfg2]: https://github.com/rustwasm/wasm-bindgen/blob/0.2.69/src/lib.rs#L67-L75

## 重新导出依赖功能 {#re-exporting-dependency-features}

重新导出依赖的功能可能很方便。这允许依赖该 crate 的用户控制这些功能，而无需直接指定这些依赖。例如，[`regex`] [重新导出了来自 `regex_syntax` 包的功能][regex-re-export][regex_syntax-features]。`regex` 的用户不需要知道 `regex_syntax` 包，但他们仍然可以访问它包含的功能。

[regex-re-export]: https://github.com/rust-lang/regex/blob/1.4.2/Cargo.toml#L65-L89
[regex_syntax-features]: https://github.com/rust-lang/regex/blob/1.4.2/regex-syntax/Cargo.toml#L17-L32

## C 库的本地构建 {#vendoring-of-c-libraries}

一些包提供了对常见 C 库的绑定（有时称为 ["sys" crates][sys]）。有时，这些包让你选择使用系统上安装的 C 库，或者从源代码构建它。例如，[`openssl`] 包有一个 [`vendored` 功能][openssl-vendored]，它启用了 [`openssl-sys`] 的相应 `vendored` 功能。`openssl-sys` 的构建脚本有一些[条件逻辑][openssl-sys-cfg]，导致它从 OpenSSL 源代码的本地副本构建，而不是使用系统版本。

[`curl-sys`] 包是另一个例子，其中 [`static-curl` 功能][curl-sys-static]导致它从源代码构建 libcurl。请注意，它还有一个 [`force-system-lib-on-osx` 功能][curl-sys-macos]，强制[它使用系统的 libcurl][curl-sys-macos-code]，覆盖 static-curl 设置。

[`openssl`]: https://crates.io/crates/openssl
[`openssl-sys`]: https://crates.io/crates/openssl-sys
[sys]: build-scripts.md#-sys-packages
[openssl-vendored]: https://github.com/sfackler/rust-openssl/blob/openssl-v0.10.31/openssl/Cargo.toml#L19
[build script]: build-scripts.md
[openssl-sys-cfg]: https://github.com/sfackler/rust-openssl/blob/openssl-v0.10.31/openssl-sys/build/main.rs#L47-L54
[`curl-sys`]: https://crates.io/crates/curl-sys
[curl-sys-static]: https://github.com/alexcrichton/curl-rust/blob/0.4.34/curl-sys/Cargo.toml#L49
[curl-sys-macos]: https://github.com/alexcrichton/curl-rust/blob/0.4.34/curl-sys/Cargo.toml#L52
[curl-sys-macos-code]: https://github.com/alexcrichton/curl-rust/blob/0.4.34/curl-sys/build.rs#L15-L20

## 功能优先级 {#feature-precedence}

一些包可能具有互斥的功能。处理此问题的一种方法是优先选择其中一个功能。[`log`] 包就是一个例子。它有[几个功能][log-features]，用于在编译时选择最大日志级别，描述[在此][log-docs]。它使用 [`cfg-if`] 来[选择优先级][log-cfg-if]。如果启用了多个功能，较高的 "max" 级别将优先于较低的级别。

[`log`]: https://crates.io/crates/log
[log-features]: https://github.com/rust-lang/log/blob/0.4.11/Cargo.toml#L29-L42
[log-docs]: https://docs.rs/log/0.4.11/log/#compile-time-filters
[log-cfg-if]: https://github.com/rust-lang/log/blob/0.4.11/src/lib.rs#L1422-L1448
[`cfg-if`]: https://crates.io/crates/cfg-if

## 过程宏配套包 {#proc-macro-companion-package}

一些包有一个与它紧密相连的过程宏。然而，并非所有用户都需要使用过程宏。通过将过程宏设为可选依赖，这允许你方便地选择是否包含它。这很有用，因为有时过程宏版本必须与父包保持同步，而你不想强迫用户必须指定两个依赖项并保持它们同步。

一个例子是 [`serde`]，它有一个 [`derive` 功能][serde-derive]，用于启用 [`serde_derive`] 过程宏。`serde_derive` crate 与 `serde` 紧密相连，因此它使用[等于版本要求][serde-equals]来确保它们保持同步。

[`serde`]: https://crates.io/crates/serde
[`serde_derive`]: https://crates.io/crates/serde_derive
[serde-derive]: https://github.com/serde-rs/serde/blob/v1.0.118/serde/Cargo.toml#L34-L35
[serde-equals]: https://github.com/serde-rs/serde/blob/v1.0.118/serde/Cargo.toml#L17

## 仅限 Nightly 的功能 {#nightly-only-features}

一些包想要试验仅在 Rust [nightly 频道][nightly channel]上可用的 API 或语言功能。然而，他们可能不希望要求他们的用户也使用 nightly 频道。一个例子是 [`wasm-bindgen`]，它有一个 [`nightly` 功能][wasm-bindgen-nightly]，启用了一个[扩展的 API][wasm-bindgen-unsize]，该 API 使用了仅在撰写本文时才在 nightly 频道上可用的 [`Unsize`] 标记特质。

请注意，在 crate 的根部，它使用 [`cfg_attr` 来启用 nightly 功能][wasm-bindgen-cfg_attr]。请记住，[`feature` 属性][`feature` attribute]与 Cargo 功能无关，用于选择加入实验性语言功能。

[`rand`] 包的 [`simd_support` 功能][rand-simd_support]是另一个例子，它依赖于仅在 nightly 频道上构建的依赖项。

[`wasm-bindgen`]: https://crates.io/crates/wasm-bindgen
[nightly channel]: ../../book/appendix-07-nightly-rust.html
[wasm-bindgen-nightly]: https://github.com/rustwasm/wasm-bindgen/blob/0.2.69/Cargo.toml#L27
[wasm-bindgen-unsize]: https://github.com/rustwasm/wasm-bindgen/blob/0.2.69/src/closure.rs#L257-L269
[`Unsize`]: ../../std/marker/trait.Unsize.html
[wasm-bindgen-cfg_attr]: https://github.com/rustwasm/wasm-bindgen/blob/0.2.69/src/lib.rs#L11
[`feature` attribute]: ../../unstable-book/index.html
[`rand`]: https://crates.io/crates/rand
[rand-simd_support]: https://github.com/rust-random/rand/blob/0.7.3/Cargo.toml#L40

## 实验性功能 {#experimental-features}

一些包具有他们可能想要试验的新功能，而不必承诺这些 API 的稳定性。这些功能通常被记录为实验性的，因此即使是在次要发布期间，也可能在未来发生变化或破坏。一个例子是 [`async-std`] 包，它有一个 [`unstable` 功能][async-std-unstable]，该功能[对新 API 进行门控][async-std-gate]，人们可以选择使用，但可能还没有完全准备好依赖。

[`async-std`]: https://crates.io/crates/async-std
[async-std-unstable]: https://github.com/async-rs/async-std/blob/v1.8.0/Cargo.toml#L38-L42
[async-std-gate]: https://github.com/async-rs/async-std/blob/v1.8.0/src/macros.rs#L46