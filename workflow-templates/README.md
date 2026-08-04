# GitHub Actions 模板

本目录中的文件会出现在组织仓库的「Actions → New workflow」页面。选择模板后，GitHub 会将 YAML 复制到目标仓库；复制结果完全独立，不会在运行时引用本仓库，也不会自动同步后续变更。

> 当前 Draft 中仅 `go-ci` 已按仓库现有实践完成修订。其余模板仍在逐项复核，不应复制使用。

## 使用方式

1. 在目标仓库的「Actions → New workflow」中选择合适的模板。
2. 复制后，先替换模板顶部注释标出的 `REPLACE_ME`、`CHART_PATH`、镜像名称或发布命名空间。
3. 将触发分支、构建上下文、测试命令和发布语义调整为该仓库的实际合同，再提交。
4. 在 Pull Request 中验证构建类模板；发布类模板只应在版本与发布物已准备好后通过稳定版 `vX.Y.Z` tag 触发。

模板不包含私有仓库名、内部 runner、网络信息或额外 secret 约定。发布模板只使用 GitHub 提供的 `GITHUB_TOKEN`，并默认发布到当前组织的 GHCR 命名空间。

## 模板选择

| 模板 | 适用场景 | 主要输出 | 首次复制后必须确认 |
| --- | --- | --- | --- |
| `go-ci` | Go 代码的变更感知检查 | 测试覆盖率 artifact | 本地 Make 入口、module 布局、私有依赖与变更过滤范围 |
| `helm-chart-ci` | Helm Chart 的静态验证 | 渲染后的 manifest | `CHART_PATH` 与 values 覆盖场景 |
| `build-image` | 容器镜像构建验证 | 可验证的本地镜像 tar artifact | Dockerfile、上下文和目标平台 |
| `build-helm-chart` | Chart 打包验证 | `.tgz` artifact | `CHART_PATH` 与依赖来源 |
| `release-image` | 发布多平台镜像到 GHCR | GHCR image 与 GitHub Release | `IMAGE_NAME`、平台、版本策略 |
| `release-helm-chart` | 发布 Chart 到 GHCR OCI | `.tgz`、OCI chart 与 GitHub Release | `CHART_PATH`、`OCI_NAMESPACE` 与 Chart version |

## Go CI 合同

`go-ci` 采用 job 级变更过滤。PR 没有 Go 相关变化时，耗时检查会跳过，但 `Go CI required` 仍会完成，避免分支保护检查长期处于 Pending。`push` 与手动运行始终执行完整检查。变更检测失败时，终态检查必须失败，不能将检测错误解释为「没有 Go 变化」。

职责边界如下：

- Workflow 负责事件触发、变更过滤、代码检出、Go 与 golangci-lint 环境、缓存、coverage artifact 和稳定终态检查。
- Makefile 负责格式、静态分析、测试、构建、嵌套 module、build tag、CGO 和仓库专属验证等实际语义。
- Protobuf、CRD 和 OpenAPI 生成物检查由独立模板负责，不写入通用 Go Workflow。

默认本地命令与 GitHub Actions 中执行的命令完全相同：

```bash
make fmt-check vet lint test verify-build
```

复制后必须确认以下内容：

1. 上述目标均为非修改型检查，并映射到仓库已有入口。目标名不同的仓库应直接修改 Workflow 中这一行，不应在 CI 中自动探测目标。
2. `make test` 生成非空的 `cover.out`。使用其他路径时，应同时修改 `GO_COVERAGE_FILE`。
3. 本地使用的 golangci-lint 版本与 `GOLANGCI_LINT_VERSION` 一致。仓库已自行安装固定版本工具时，可移除 Workflow 中的安装步骤。
4. 根 module 不在 `go.mod` 时，应修改 `go-version-file` 与 `cache-dependency-path`。多 module 仓库还应保证本地 Make 目标覆盖每个 module。
5. `dorny/paths-filter` 列表应包含所有影响本地命令的输入，例如 included Makefile、生成工具、嵌入资源或仓库专属脚本。不要使用 workflow 级 `paths` 跳过整个检查。
6. 私有 Go module 的域名、Secret 和 Git URL rewrite 属于仓库配置，不应加入公开模板。

## 共同约定

- PR 与普通分支上的旧运行会被取消，避免过期结果占用 runner；发布运行不会被取消。
- CI 模板只验证或生成 artifact，不会登录 registry 或发布。
- 发布模板只接受无预发布后缀、无前导零的三段版本；Helm 发布额外要求 `Chart.yaml` 的 version 与 tag 版本一致。
- Helm 通过 `azure/setup-helm` 的 `latest` 安装最新稳定版，不继承历史的 Helm `3.20` 固定值。若特定项目有兼容性约束，应在该项目的 workflow 中写明原因并固定版本。
- 所有模板均有超时、缓存或 artifact 校验，以及名称唯一的终态 job，便于目标仓库设置分支保护检查。

## 维护边界

这里提供的是可运行的起点，而不是跨仓同步机制。仓库 owner 应保留并维护复制后的完整 workflow；当模板更新或出现通用问题时，按仓库语义提交普通更新 PR。复杂发布需求，例如 Pages Helm repository、私有 registry、签名、SBOM 或自托管 runner，应在目标仓库单独实现和审查，不应把组织级公开模板变成大量开关的集合。
