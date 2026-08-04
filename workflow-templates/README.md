# GitHub Actions 模板

本目录中的文件会出现在组织仓库的「Actions → New workflow」页面。选择模板后，GitHub 会将 YAML 复制到目标仓库；复制结果完全独立，不会在运行时引用本仓库，也不会自动同步后续变更。

> 当前 Draft 中，`go-ci`、`helm-chart-ci` 与 `release-helm-chart` 已按仓库现有实践完成修订。其余模板仍在逐项复核，不应复制使用。

## 使用方式

1. 在目标仓库的「Actions → New workflow」中选择合适的模板。
2. 复制后，先替换模板顶部注释标出的 `REPLACE_ME`、`CHART_PATH`、镜像名称或发布命名空间。
3. 将触发分支、构建上下文、测试命令和发布语义调整为该仓库的实际合同，再提交。
4. 在 Pull Request 中验证构建类模板。镜像发布使用稳定版 `vX.Y.Z` tag；Helm Chart 发布仅由默认分支上递增的 `Chart.yaml.version` 触发。

模板不包含私有仓库名、内部 runner、网络信息或额外 secret 约定。发布模板只使用 GitHub 提供的 `GITHUB_TOKEN`，并默认发布到当前组织的 GHCR 命名空间。

## 模板选择

| 模板 | 适用场景 | 主要输出 | 首次复制后必须确认 |
| --- | --- | --- | --- |
| `go-ci` | Go 代码的变更感知检查 | 测试覆盖率 artifact | 本地 Make 入口、module 布局、私有依赖与变更过滤范围 |
| `helm-chart-ci` | Helm Chart 的变更感知检查 | 稳定的验证终态 | 本地 Make 入口、Chart 布局与变更过滤范围 |
| `build-image` | 容器镜像构建验证 | 可验证的本地镜像 tar artifact | Dockerfile、上下文和目标平台 |
| `build-helm-chart` | Chart 打包验证 | `.tgz` artifact | `CHART_PATH` 与依赖来源 |
| `release-image` | 发布多平台镜像到 GHCR | GHCR image 与 GitHub Release | `IMAGE_NAME`、平台、版本策略 |
| `release-helm-chart` | 按 `Chart.yaml.version` 自动发布 Chart | 同一份 `.tgz`、OCI Chart 与 GitHub Release | `CHART_PATH`、Chart 依赖与 `OCI_NAMESPACE` |

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

## Helm Chart CI 合同

`helm-chart-ci` 与 Go CI 一样采用 job 级变更过滤。无 Helm 相关变化的 PR 会跳过耗时检查，但 `Helm Chart CI required` 仍会完成；变更检测失败时，终态检查必须失败。默认分支 push 与手动运行始终执行完整检查。

Workflow 只负责事件、变更过滤、代码检出和固定版本的工具环境。Chart 的真实验证语义由仓库本地入口统一实现：

```bash
make verify-helm-chart
```

该目标至少应覆盖生成文件漂移、依赖策略、`helm lint --strict`、有代表性的 values 渲染和临时打包检查，并保证不会发布或访问 Kubernetes 集群。README 与 `values.schema.json` 等生成物必须先确认已被 Git 跟踪，再重新生成并检查 diff；缺少 helm-docs 或 schema 工具时应直接失败，不能跳过检查。

模板当前使用 Helm v4.2.3、helm-docs v1.14.2 和 `helm-values-schema-json` v2.5.0。Helm 4 默认验证插件签名，而 schema 插件尚未发布 provenance 元数据，因此安装命令必须显式使用 `--verify=false`；插件版本仍固定且可复现。这一兼容参数不代表继续锁定旧 Helm 3.20。

复制后必须确认以下内容：

1. `make verify-helm-chart` 在本地 checkout 中可直接运行，且不会静默跳过任何生成物或工具。
2. `dorny/paths-filter` 包含 Chart、Makefile include、生成配置、脚本及所有会影响验证的输入。不要用 workflow 级 `paths` 跳过整个必需检查。
3. 仓库若不生成 README 或 schema，应同时删除对应工具安装，而不是保留无效步骤。
4. 私有依赖、额外 registry 凭据和项目专属 values 断言继续留在目标仓库。

## Helm Chart 发布合同

`release-helm-chart` 只监听默认分支 push，`Chart.yaml.version` 是唯一版本来源。每次运行都比较 push 事件中的精确 `before` 与 `github.sha`：版本未变时明确成功且不发布；首次加入 Chart 时允许首发；基线不可解析、版本非法或版本降低时直接失败。tag、GitHub Release、手工输入和其他文件均不能另行指定版本。

发布前复用本地 CI 入口。打包本身是通用的发布编排，由 Workflow 使用标准 Helm 命令完成：

```bash
make verify-helm-chart
helm dependency build "$CHART_PATH"
helm package "$CHART_PATH" --destination chart-release
```

Workflow 要求从 `CHART_PATH` 生成且只生成一个 `<name>-<version>.tgz`，并再次核对 package 元数据。随后，同一份 artifact 会用于 GitHub Release 和 `ghcr.io/<owner>/charts`，不会为不同目的重复打包。GitHub tag 使用 `<chart-name>-<chart-version>`，并设置 `make_latest: false`，避免 Chart Release 覆盖仓库的应用 Latest Release。

发布版本不可覆盖：同名 Git tag、GitHub Release 或 GHCR OCI Chart 只要已存在，或远端查询无法确认不存在，发布都会在写入前失败。Workflow 先推送并验证 OCI，再创建 GitHub Release；若远端写入后发生部分失败，应检查外部状态并显式清理，或递增 `Chart.yaml.version`，不能把跳过已有产物当作成功。

发布运行不会被取消。`Helm Chart Release required` 必须覆盖「版本未变且发布跳过」与「版本递增且完整发布」两种成功状态；规划、打包或发布任一阶段失败时均失败。GitHub Pages、签名、SBOM 和其他 registry 不属于通用模板，确有需要时由目标仓库单独实现。

## 共同约定

- PR 与普通分支上的旧运行会被取消，避免过期结果占用 runner；发布运行不会被取消。
- CI 模板只验证或生成 artifact，不会登录 registry 或发布。
- 镜像发布模板只接受无预发布后缀、无前导零的三段 tag；Helm 发布只接受 `Chart.yaml` 中同样格式且严格递增的版本。
- Helm 使用修订时的最新稳定版 v4.2.3，并显式记录相关工具版本，不继承历史的 Helm 3.20 固定值。特定项目如有兼容性约束，应在该项目 Workflow 中写明原因并固定版本。
- 所有模板均有超时、缓存或 artifact 校验，以及名称唯一的终态 job，便于目标仓库设置分支保护检查。

## 维护边界

这里提供的是可运行的起点，而不是跨仓同步机制。仓库 owner 应保留并维护复制后的完整 workflow；当模板更新或出现通用问题时，按仓库语义提交普通更新 PR。复杂发布需求，例如 Pages Helm repository、私有 registry、签名、SBOM 或自托管 runner，应在目标仓库单独实现和审查，不应把组织级公开模板变成大量开关的集合。
