# 外部工具 {#external-tools}

Cargo 的目标之一是简化与第三方工具（如 IDE 和其他构建系统）的集成。为了便于集成，Cargo 提供了以下几种机制：

* [`cargo metadata`] 命令，以 JSON 格式输出包结构和依赖信息，
* `--message-format` 标志，输出特定构建的相关信息，以及
* 对自定义子命令的支持。

## 包结构信息 {#information-about-package-structure}

你可以使用 [`cargo metadata`] 命令来获取包结构和依赖信息。有关输出格式的详细信息，请参阅 [`cargo metadata`] 文档。

该格式是稳定且版本化的。在调用 `cargo metadata` 时，你应该显式传递 `--format-version` 标志，以避免向前兼容性风险。

如果你使用 Rust，可以使用 [cargo_metadata] crate 来解析输出。

[cargo_metadata]: https://crates.io/crates/cargo_metadata
[`cargo metadata`]: ../commands/cargo-metadata.md

## JSON 消息 {#json-messages}

当传递 `--message-format=json` 时，Cargo 将在构建过程中输出以下信息：

* 编译器错误和警告，
* 生成的制品，
* 构建脚本的结果（例如，原生依赖）。

输出以每行一个 JSON 对象的形式写入 stdout。`reason` 字段用于区分不同类型的消息。
`package_id` 字段是用于引用包的唯一标识符，也可作为许多命令的 `--package` 参数。语法规则可在 [Package ID Specifications] 章节中找到。

> **注意：** `--message-format=json` 仅控制 Cargo 和 Rustc 的输出。
> 这无法控制其他工具的输出，
> 例如 `cargo run --message-format=json`，
> 或过程宏的任意输出。
> 在这些情况下，一种可能的解决方法是仅当一行以 `{` 开头时才将其解释为 JSON。

`--message-format` 选项还可以接受额外的格式化值，这些值会改变 JSON 消息的计算和渲染方式。有关更多详细信息，请参阅 [build command documentation] 中 `--message-format` 选项的描述。

如果你使用 Rust，可以使用 [cargo_metadata] crate 来解析这些消息。

> **MSRV：** 1.77 版本要求 `package_id` 为 Package ID Specification。在此之前，它是不透明的。

[build command documentation]: ../commands/cargo-build.md
[cargo_metadata]: https://crates.io/crates/cargo_metadata
[Package ID Specifications]: ./pkgid-spec.md

### 编译器消息 {#compiler-messages}

"compiler-message" 消息包含来自编译器的输出，例如警告和错误。有关 `rustc` 消息格式的详细信息（该格式嵌入在以下结构中），请参阅 [rustc JSON 章节](../../rustc/json.md)：

```javascript
{
    /* The "reason" indicates the kind of message. */
    "reason": "compiler-message",
    /* The Package ID, a unique identifier for referring to the package. */
    "package_id": "file:///path/to/my-package#0.1.0",
    /* Absolute path to the package manifest. */
    "manifest_path": "/path/to/my-package/Cargo.toml",
    /* The Cargo target (lib, bin, example, etc.) that generated the message. */
    "target": {
        /* Array of target kinds.
           - lib targets list the `crate-type` values from the
             manifest such as "lib", "rlib", "dylib",
             "proc-macro", etc. (default ["lib"])
           - binary is ["bin"]
           - example is ["example"]
           - integration test is ["test"]
           - benchmark is ["bench"]
           - build script is ["custom-build"]
        */
        "kind": [
            "lib"
        ],
        /* Array of crate types.
           - lib and example libraries list the `crate-type` values
             from the manifest such as "lib", "rlib", "dylib",
             "proc-macro", etc. (default ["lib"])
           - all other target kinds are ["bin"]
        */
        "crate_types": [
            "lib"
        ],
        /* The name of the target.
           For lib targets, dashes will be replaced with underscores.
        */
        "name": "my_package",
        /* Absolute path to the root source file of the target. */
        "src_path": "/path/to/my-package/src/lib.rs",
        /* The Rust edition of the target.
           Defaults to the package edition.
        */
        "edition": "2018",
        /* Array of required features.
           This property is not included if no required features are set.
        */
        "required-features": ["feat1"],
        /* Whether the target should be documented by `cargo doc`. */
        "doc": true,
        /* Whether or not this target has doc tests enabled, and
           the target is compatible with doc testing.
        */
        "doctest": true
        /* Whether or not this target should be built and run with `--test`
        */
        "test": true
    },
    /* The message emitted by the compiler.

    See https://doc.rust-lang.org/rustc/json.html for details.
    */
    "message": {
        /* ... */
    }
}
```

### Artifact 消息 {#artifact-messages}

对于每个编译步骤，都会发出一个 "compiler-artifact" 消息，其结构如下：

```javascript
{
    /* "reason" 字段指示消息类型。 */
    "reason": "compiler-artifact",
    /* Package ID，用于引用包的唯一标识符。 */
    "package_id": "file:///path/to/my-package#0.1.0",
    /* 包清单的绝对路径。 */
    "manifest_path": "/path/to/my-package/Cargo.toml",
    /* 生成 Artifact 的 Cargo 目标（lib、bin、example 等）。
       有关详细信息，请参阅上面 `compiler-message` 中的定义。
    */
    "target": {
        "kind": [
            "lib"
        ],
        "crate_types": [
            "lib"
        ],
        "name": "my_package",
        "src_path": "/path/to/my-package/src/lib.rs",
        "edition": "2018",
        "doc": true,
        "doctest": true,
        "test": true
    },
    /* profile 指示使用了哪些编译器设置。 */
    "profile": {
        /* 优化级别。 */
        "opt_level": "0",
        /* 调试级别，整数 0、1 或 2，或字符串
           "line-directives-only" 或 "line-tables-only"。如果为 `null`，则暗示
           rustc 的默认值 0。
        */
        "debuginfo": 2,
        /* 是否启用了调试断言。 */
        "debug_assertions": true,
        /* 是否启用了溢出检查。 */
        "overflow_checks": true,
        /* 是否使用了 `--test` 标志。 */
        "test": false
    },
    /* Array of features enabled. */
    "features": ["feat1", "feat2"],
    /* Array of files generated by this step. */
    "filenames": [
        "/path/to/my-package/target/debug/libmy_package.rlib",
        "/path/to/my-package/target/debug/deps/libmy_package-be9f3faac0a26ef0.rmeta"
    ],
    /* 所创建可执行文件的路径字符串，如果
       此步骤未生成可执行文件，则为 null。
    */
    "executable": null,
    /* Whether or not this step was actually executed.
       When `true`, this means that the pre-existing artifacts were
       up-to-date, and `rustc` was not executed. When `false`, this means that
       `rustc` was run to generate the artifacts.
    */
    "fresh": true
}

```

### 构建脚本输出 {#build-script-output}

"build-script-executed" 消息包含构建脚本的解析输出。请注意，即使构建脚本未运行，也会发出此消息；它将显示先前缓存的值。有关构建脚本输出的更多详细信息，请参阅 [构建脚本章节](build-scripts.md)。

```javascript
{
    /* The "reason" indicates the kind of message. */
    "reason": "build-script-executed",
    /* The Package ID, a unique identifier for referring to the package. */
    "package_id": "file:///path/to/my-package#0.1.0",
    /* Array of libraries to link, as indicated by the `cargo::rustc-link-lib`
       instruction. Note that this may include a "KIND=" prefix in the string
       where KIND is the library kind.
    */
    "linked_libs": ["foo", "static=bar"],
    /* Array of paths to include in the library search path, as indicated by
       the `cargo::rustc-link-search` instruction. Note that this may include a
       "KIND=" prefix in the string where KIND is the library kind.
    */
    "linked_paths": ["/some/path", "native=/another/path"],
    /* Array of cfg values to enable, as indicated by the `cargo::rustc-cfg`
       instruction.
    */
    "cfgs": ["cfg1", "cfg2=\"string\""],
    /* Array of [KEY, VALUE] arrays of environment variables to set, as
       indicated by the `cargo::rustc-env` instruction.
    */
    "env": [
        ["SOME_KEY", "some value"],
        ["ANOTHER_KEY", "another value"]
    ],
    /* An absolute path which is used as a value of `OUT_DIR` environmental
       variable when compiling current package.
    */
    "out_dir": "/some/path/in/target/dir"
}
```

### 构建完成 {#build-finished}

"build-finished" 消息在构建结束时发出。

```javascript
{
    /* The "reason" indicates the kind of message. */
    "reason": "build-finished",
    /* Whether or not the build finished successfully. */
    "success": true,
}
```

此消息有助于工具知道何时停止读取 JSON 消息。诸如 `cargo test` 或 `cargo run` 等命令可能会在构建完成后产生额外的输出。此消息让工具知道 Cargo 不会产生额外的 JSON 消息，但之后可能会产生其他输出（例如由 `cargo run` 执行的程序生成的输出）。

> 注意：目前对测试的 JSON 输出支持处于实验性阶段（仅限 nightly），
> 因此，如果启用该功能，可能会在 "build-finished" 消息之后开始收到额外的测试特定 JSON 消息。

## 自定义子命令 {#custom-subcommands}

Cargo 设计为无需修改 Cargo 本身即可通过新的子命令进行扩展。这是通过将 `cargo (?<command>[^ ]+)` 形式的调用转换为外部工具 `cargo-${command}` 的调用来实现的。外部工具必须位于用户的 `$PATH` 目录之一中。

> **注意：** Cargo 默认优先使用 `$CARGO_HOME/bin` 中的外部工具，而非 `$PATH`。用户可以通过将 `$CARGO_HOME/bin` 添加到 `$PATH` 来覆盖此优先级。

当 Cargo 调用自定义子命令时，子命令的第一个参数将是自定义子命令的文件名，这是通常的做法。第二个参数将是子命令名称本身。例如，当调用 `cargo-${command}` 时，第二个参数将是 `${command}`。命令行上的任何其他参数将原样转发。

Cargo 还可以通过 `cargo help ${command}` 显示自定义子命令的帮助输出。Cargo 假设子命令的第三个参数是 `--help` 时会打印帮助消息。因此，`cargo help ${command}` 将调用 `cargo-${command} ${command} --help`。

自定义子命令可以使用 `CARGO` 环境变量回调 Cargo。或者，它可以将 `cargo` crate 作为库链接，但这种方法有缺点：

* Cargo 作为库是不稳定的：API 可能会在没有弃用的情况下更改
* 链接的 Cargo 库版本可能与 Cargo 二进制文件不同

相反，鼓励使用 CLI 接口来驱动 Cargo。[`cargo metadata`] 命令可用于获取当前项目的信息（[`cargo_metadata`] crate 提供了此命令的 Rust 接口）。

[`cargo metadata`]: ../commands/cargo-metadata.md
[`cargo_metadata`]: https://crates.io/crates/cargo_metadata
