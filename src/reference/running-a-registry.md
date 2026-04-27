# 运行注册中心 {#running-a-registry}

可以通过包含索引的 git 仓库和包含由 [`cargo package`] 创建的压缩 `.crate` 文件的服务器来实现一个最小化的注册中心。用户将无法使用 Cargo 向其发布，但这对于封闭环境可能已经足够。索引格式在 [Registry Index] 中描述。

支持发布功能的全功能注册中心还需要提供一个符合 [Registry Web API] 所述 API 的 Web API 服务。

商业和社区项目可用于构建和运行注册中心。请参阅 <https://github.com/rust-lang/cargo/wiki/Third-party-registries> 获取可用项目列表。

[Registry Web API]: registry-web-api.md
[Registry Index]: registry-index.md
[`cargo publish`]: ../commands/cargo-publish.md
[`cargo package`]: ../commands/cargo-package.md