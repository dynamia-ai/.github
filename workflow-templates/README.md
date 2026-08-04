# GitHub Actions 模板

本目录中的文件会出现在组织仓库的「Actions → New workflow」页面。选择模板后，GitHub 会将 YAML 复制到目标仓库；复制结果完全独立，不会在运行时引用本仓库，也不会自动同步后续变更。

## 使用方式

1. 在目标仓库的「Actions → New workflow」中选择合适的模板。
2. 复制后，先替换模板顶部注释标出的 `REPLACE_ME`、`CHART_PATH`、镜像名称或发布命名空间。
3. 将触发分支、构建上下文、测试命令和发布语义调整为该仓库的实际合同，再提交。
4. 在 Pull Request 中验证构建类模板；发布类模板只应在版本与发布物已准备好后通过稳定版 `vX.Y.Z` tag 触发。

模板不包含私有仓库名、内部 runner、网络信息或额外 secret 约定。发布模板只使用 GitHub 提供的 `GITHUB_TOKEN`，并默认发布到当前组织的 GHCR 命名空间。

## 模板选择

| 模板 | 适用场景 | 主要输出 | 首次复制后必须确认 |
| --- | --- | --- | --- |
| `go-ci` | 单 Go module 的日常检查 | 测试覆盖率 artifact | module 布局、linter 配置、额外测试命令 |
| `helm-chart-ci` | Helm Chart 的静态验证 | 渲染后的 manifest | `CHART_PATH` 与 values 覆盖场景 |
| `build-image` | 容器镜像构建验证 | 可验证的本地镜像 tar artifact | Dockerfile、上下文和目标平台 |
| `build-helm-chart` | Chart 打包验证 | `.tgz` artifact | `CHART_PATH` 与依赖来源 |
| `release-image` | 发布多平台镜像到 GHCR | GHCR image 与 GitHub Release | `IMAGE_NAME`、平台、版本策略 |
| `release-helm-chart` | 发布 Chart 到 GHCR OCI | `.tgz`、OCI chart 与 GitHub Release | `CHART_PATH`、`OCI_NAMESPACE` 与 Chart version |

## 共同约定

- PR 与普通分支上的旧运行会被取消，避免过期结果占用 runner；发布运行不会被取消。
- CI 模板只验证或生成 artifact，不会登录 registry 或发布。
- 发布模板只接受无预发布后缀、无前导零的三段版本；Helm 发布额外要求 `Chart.yaml` 的 version 与 tag 版本一致。
- Helm 通过 `azure/setup-helm` 的 `latest` 安装最新稳定版，不继承历史的 Helm `3.20` 固定值。若特定项目有兼容性约束，应在该项目的 workflow 中写明原因并固定版本。
- 所有模板均有超时、缓存或 artifact 校验，以及名称唯一的终态 job，便于目标仓库设置分支保护检查。

## 维护边界

这里提供的是可运行的起点，而不是跨仓同步机制。仓库 owner 应保留并维护复制后的完整 workflow；当模板更新或出现通用问题时，按仓库语义提交普通更新 PR。复杂发布需求，例如 Pages Helm repository、私有 registry、签名、SBOM 或自托管 runner，应在目标仓库单独实现和审查，不应把组织级公开模板变成大量开关的集合。
