# 持续集成 {#continuous-integration}

## 开始使用 {#getting-started}

基础的 CI 会构建并测试您的项目：

### GitHub Actions

要在 GitHub Actions 上测试您的包，以下是一个示例 `.github/workflows/ci.yml` 文件：

```yaml
name: Cargo Build & Test

on:
  push:
  pull_request:

env:
  CARGO_TERM_COLOR: always

jobs:
  build_and_test:
    name: Rust project - latest
    runs-on: ubuntu-latest
    strategy:
      matrix:
        toolchain:
          - stable
          - beta
          - nightly
    steps:
      - uses: actions/checkout@v4
      - run: rustup update ${{ matrix.toolchain }} && rustup default ${{ matrix.toolchain }}
      - run: cargo build --verbose
      - run: cargo test --verbose

```

这将测试所有三个发布通道（注意任何工具链版本的失败都会导致整个作业失败）。您也可以在 GitHub UI 中点击 `"Actions" > "new workflow"` 并选择 Rust，将[默认配置](https://github.com/actions/starter-workflows/blob/main/ci/rust.yml)添加到您的仓库。更多信息请参阅 [GitHub Actions 文档](https://docs.github.com/en/actions)。

### GitLab CI

要在 GitLab CI 上测试您的包，以下是一个示例 `.gitlab-ci.yml` 文件：

```yaml
stages:
  - build

rust-latest:
  stage: build
  image: rust:latest
  script:
    - cargo build --verbose
    - cargo test --verbose

rust-nightly:
  stage: build
  image: rustlang/rust:nightly
  script:
    - cargo build --verbose
    - cargo test --verbose
  allow_failure: true
```

这将在稳定通道和 nightly 通道上进行测试，但 nightly 通道中的任何故障不会导致整个构建失败。更多信息请参阅 [GitLab CI 文档](https://docs.gitlab.com/ce/ci/yaml/index.html)。

### builds.sr.ht

要在 sr.ht 上测试您的包，以下是一个示例 `.build.yml` 文件。
请务必将 `<your repo>` 和 `<your project>` 替换为要克隆的仓库和克隆后的目录。

```yaml
image: archlinux
packages:
  - rustup
sources:
  - <your repo>
tasks:
  - setup: |
      rustup toolchain install nightly stable
      cd <your project>/
      rustup run stable cargo fetch
  - stable: |
      rustup default stable
      cd <your project>/
      cargo build --verbose
      cargo test --verbose
  - nightly: |
      rustup default nightly
      cd <your project>/
      cargo build --verbose ||:
      cargo test --verbose  ||:
  - docs: |
      cd <your project>/
      rustup run stable cargo doc --no-deps
      rustup run nightly cargo doc --no-deps ||:
```

这将在稳定通道和 nightly 通道上测试并构建文档，但 nightly 通道中的任何故障不会导致整个构建失败。更多信息请参阅 [builds.sr.ht 文档](https://man.sr.ht/builds.sr.ht/)。

### CircleCI

要在 CircleCI 上测试您的包，以下是一个示例 `.circleci/config.yml` 文件：

```yaml
version: 2.1
jobs:
  build:
    docker:
      # 检查 https://circleci.com/developer/images/image/cimg/rust#image-tags 获取最新版本
      - image: cimg/rust:1.77.2
    steps:
      - checkout
      - run: cargo test
```

要运行更复杂的流水线，包括检测不稳定测试、缓存和工件管理，请参阅 [CircleCI 配置参考](https://circleci.com/docs/configuration-reference/)。

## 验证最新依赖 {#verifying-latest-dependencies}

在 `Cargo.toml` 中[指定依赖](../reference/specifying-dependencies.md)时，它们通常匹配一个版本范围。详尽测试所有版本组合将非常繁琐。至少验证最新版本可以测试那些运行 [`cargo add`] 或 [`cargo install`] 的用户。

测试最新版本时的一些考虑因素包括：
- 最小化影响本地开发或 CI 的外部因素
- 新依赖发布的频率
- 项目愿意接受的风险级别
- CI 成本，包括间接成本，例如如果 CI 服务有并行运行器的上限，导致在达到上限时新作业被序列化

一些潜在的解决方案包括：
- [不提交 `Cargo.lock`](../faq.md#why-have-cargolock-in-version-control)
  - 根据 PR 的频率，许多版本可能未经测试
  - 这会牺牲确定性
- 让一个 CI 作业验证最新依赖，但标记为“失败时继续”
  - 根据 CI 服务的不同，失败可能不明显
  - 根据 PR 的频率，可能使用比必要更多的资源
- 安排一个定时 CI 作业来验证最新依赖
  - 托管的 CI 服务可能会停用一段时间未触及的仓库的定时作业，影响被动维护的包
  - 根据 CI 服务的不同，通知可能无法路由到可以处理失败的人员
  - 如果没有与依赖发布频率平衡，可能测试的版本不足或进行冗余测试
- 通过 PR 定期更新依赖，例如使用 [Dependabot] 或 [RenovateBot]
  - 可以将依赖隔离到它们自己的 PR 中，或将它们合并到一个 PR 中
  - 仅使用必要的资源
  - 可以配置频率以平衡 CI 资源和依赖版本的覆盖范围

使用 GitHub Actions 验证最新依赖的 CI 作业示例：
```yaml
jobs:
  latest_deps:
    name: Latest Dependencies
    runs-on: ubuntu-latest
    continue-on-error: true
    env:
      CARGO_RESOLVER_INCOMPATIBLE_RUST_VERSIONS: allow
    steps:
      - uses: actions/checkout@v4
      - run: rustup update stable && rustup default stable
      - run: cargo update --verbose
      - run: cargo build --verbose
      - run: cargo test --verbose
```
注意：
- 设置 [`CARGO_RESOLVER_INCOMPATIBLE_RUST_VERSIONS`](../reference/config.md#resolverincompatible-rust-versions) 以确保[解析器](../reference/resolver.md)不会因为您项目的 [Rust 版本](../reference/rust-version.md)而限制选择的依赖。

对于每个平台或每个 Rust 版本失败风险较高的项目，可能需要测试更多组合。

## 验证 `rust-version` {#verifying-rust-version}

发布指定了 [`rust-version`](../reference/manifest.md#the-rust-version-field) 的包时，验证该字段的正确性非常重要。

一些可用于此目的的第三方工具包括：
- [`cargo-msrv`](https://crates.io/crates/cargo-msrv)
- [`cargo-hack`](https://crates.io/crates/cargo-hack)

一种使用 GitHub Actions 的示例方法：
```yaml
jobs:
  msrv:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: taiki-e/install-action@cargo-hack
    - run: cargo hack check --rust-version --workspace --all-targets --ignore-private
```
这试图在彻底性和周转时间之间取得平衡：
- 使用单一平台，因为大多数项目与平台无关，信任平台特定的依赖来验证它们的行为。
- 使用 `cargo check`，因为贡献者遇到的大多数问题是 API 可用性，而不是行为。
- 跳过来发布的包，因为这假设只有通过注册表使用已验证项目的消费者才会关心 `rust-version`。

[`cargo add`]: ../commands/cargo-add.md
[`cargo install`]: ../commands/cargo-install.md
[Dependabot]: https://docs.github.com/en/code-security/dependabot/working-with-dependabot
[RenovateBot]: https://renovatebot.com/