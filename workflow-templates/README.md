# GitHub Actions 模板

本目录中的文件会出现在组织仓库的「Actions → New workflow」页面。选择模板后，GitHub 会将 YAML 复制到目标仓库；复制结果完全独立，不会在运行时引用本仓库，也不会自动同步后续变更。

## 使用方式

1. 在目标仓库的「Actions → New workflow」中选择合适的模板。
2. 复制后，先替换模板顶部注释标出的 `REPLACE_ME`、`CHART_PATH`、镜像名称或发布命名空间。Helm Chart 发布模板中的 `on.push.paths` 与 `CHART_PATH` 必须指向同一个 Chart 目录。
3. 将对应的 Makefile example 合并进仓库 Makefile，并按项目语义补充 module、Chart values、镜像二进制或其他验证。
4. 将触发分支、构建上下文、测试命令和发布语义调整为该仓库的实际合同，再提交。
5. 在 Pull Request 中验证构建类模板。镜像发布使用稳定版 `vX.Y.Z` tag；Helm Chart 发布仅由默认分支上递增的 `Chart.yaml.version` 触发。

模板不包含私有仓库名、内部 runner、网络信息或额外 secret 约定。发布模板只使用 GitHub 提供的 `GITHUB_TOKEN`，并默认发布到当前组织的 GHCR 命名空间。

## 模板选择

| 模板 | 适用场景 | 主要输出 | 首次复制后必须确认 |
| --- | --- | --- | --- |
| `go-ci` | Go 代码的变更感知检查 | 测试覆盖率 artifact | 本地 Make 入口、module 布局、私有依赖与变更过滤范围 |
| `helm-chart-ci` | Helm Chart 的变更感知检查 | 稳定的验证终态 | 本地 Make 入口、Chart 布局与变更过滤范围 |
| `build-image` | 变更感知的容器镜像构建 | 已加载的镜像 tar 与校验和 | Dockerfile、上下文、输入过滤、本地构建目标和目标平台 |
| `build-helm-chart` | 变更感知的 Chart 验证与打包 | 经过身份核对的 `.tgz` artifact | 本地 Make 入口、`CHART_PATH` 与依赖来源 |
| `release-image` | 按稳定 tag 发布多平台镜像到 GHCR | 带 digest 的 GHCR image 与 GitHub Release | `IMAGE_NAME`、平台、构建输入与版本策略 |
| `release-helm-chart` | 按 `Chart.yaml.version` 自动发布 Chart | 同一份 `.tgz`、OCI Chart 与 GitHub Release | `CHART_PATH`、Chart 依赖与 `OCI_NAMESPACE` |

## Makefile examples

Workflow 复制到目标仓库后不会引用本仓库，因此以下文件只是可复制的起点，不是中央 include 或同步机制：

- [`examples/go-ci/Makefile`](examples/go-ci/Makefile) 与 [`.golangci.yml`](examples/go-ci/.golangci.yml)：实现 `ci`、格式化、静态分析、测试和构建。
- [`examples/helm-chart/Makefile`](examples/helm-chart/Makefile)：实现 README/schema 生成物漂移、基础 lint 和 template 渲染检查。
- [`examples/build-image/Makefile`](examples/build-image/Makefile)：实现通用单平台构建，以及便于本地使用的 amd64/arm64 构建目标。

示例只定义跨仓可复用的最小语义。嵌套 Go module、build tag、CGO、私有依赖、Chart 专属 values、镜像 build arg 和多 Dockerfile 等内容必须在目标仓库的 Makefile 或脚本中显式补充。修改 Workflow 中的本地命令时，应同时修改或新增对应 example，避免 YAML 调用一个不存在的目标。

## Go CI 合同

`go-ci` 采用 job 级变更过滤。PR 没有 Go 相关变化时，耗时检查会跳过，但 `Go CI required` 仍会完成，避免分支保护检查长期处于 Pending。`push` 与手动运行始终执行完整检查。变更检测失败时，终态检查必须失败，不能将检测错误解释为「没有 Go 变化」。

职责边界如下：

- Workflow 负责事件触发、变更过滤、代码检出、Go 与 golangci-lint 环境、缓存、coverage artifact 和稳定终态检查。
- Makefile 负责格式化、格式检查、静态分析、测试、构建、嵌套 module、build tag、CGO 和仓库专属验证等实际语义。
- Protobuf、CRD 和 OpenAPI 生成物检查不写入通用 Go Workflow；Protobuf 模板不属于当前模板集合，后续按需要单独设计。

默认本地命令与 GitHub Actions 中执行的命令完全相同：

```bash
make ci
```

可从 [`examples/go-ci/Makefile`](examples/go-ci/Makefile) 与 [`.golangci.yml`](examples/go-ci/.golangci.yml) 复制完整入口。`make ci` 依次执行非修改型的格式检查、vet、lint、测试和构建；开发阶段使用 `make fmt`，通过 golangci-lint 配置中的 gofmt 与 goimports 写入格式化结果。示例覆盖根 module 的通用检查，目标仓库仍需按实际 module、二进制和 build tag 调整。

复制后必须确认以下内容：

1. `make ci` 及其依赖目标均为非修改型检查；`make fmt` 会写入格式化结果，不应在 CI 中调用。目标名不同的仓库应直接修改 Workflow 中这一行，不应在 CI 中自动探测目标。
2. `make test` 生成非空的 `cover.out`。使用其他路径时，应同时修改 `GO_COVERAGE_FILE`。
3. 应同时复制 golangci-lint v2 配置；配置文件改名时，应修改 `GOLANGCI_LINT_CONFIG`。已有 lint 规则或历史问题的仓库，应在 `standard` 基线之上明确调整 linter 与 exclusions。本地 golangci-lint 版本应与 Workflow 中的 `GOLANGCI_LINT_VERSION` 一致。
4. 根 module 不在 `go.mod` 时，应修改 `go-version-file` 与 `cache-dependency-path`。多 module 仓库还应保证本地 Make 目标覆盖每个 module。
5. `dorny/paths-filter` 列表应包含所有影响本地命令的输入，例如 included Makefile、生成工具、嵌入资源或仓库专属脚本。不要使用 workflow 级 `paths` 跳过整个检查。
6. 私有 Go module 的域名、Secret 和 Git URL rewrite 属于仓库配置，不应加入公开模板。

## Helm Chart CI 合同

`helm-chart-ci` 与 Go CI 一样采用 job 级变更过滤。无 Helm 相关变化的 PR 会跳过耗时检查，但 `Helm Chart CI required` 仍会完成；变更检测失败时，终态检查必须失败。默认分支 push 与手动运行始终执行完整检查。

Workflow 只负责事件、变更过滤、代码检出和固定版本的工具环境。Chart 的真实验证语义由仓库本地入口统一实现：

```bash
make verify-helm-chart
```

可从 [`examples/helm-chart/Makefile`](examples/helm-chart/Makefile) 复制完整入口。Workflow 负责安装明确版本的 Helm、helm-docs 和 schema 插件；Makefile 不重复检查工具版本。

该目标只负责三类基础检查：重新生成 README 与 `values.schema.json` 后确认没有 Git diff、执行 `helm lint`、执行一次 `helm template`。helm-docs 只处理当前根 Chart，不递归改写 vendored subchart；需要非默认 values 才能渲染的 Chart 通过 `HELM_TEMPLATE_ARGS` 补充参数。缺少生成工具时命令直接失败，不静默跳过。

模板当前使用 Helm v4.2.3、helm-docs v1.14.2 和 `helm-values-schema-json` v2.5.0。Helm 4 默认验证插件签名，而 schema 插件尚未发布 provenance 元数据，因此安装命令必须显式使用 `--verify=false`；插件版本仍固定且可复现。这一兼容参数不代表继续锁定旧 Helm 3.20。

复制后必须确认以下内容：

1. `make verify-helm-chart` 在本地 checkout 中可直接运行，并且 README 与 schema 是已提交的生成文件。
2. `dorny/paths-filter` 包含 Chart、Makefile include、生成配置、脚本及所有会影响验证的输入。不要用 workflow 级 `paths` 跳过整个必需检查。
3. 仓库若不生成 README 或 schema，应同时删除对应工具安装，而不是保留无效步骤。
4. 公共或私有 Chart repository 的注册、额外 registry 凭据和项目专属 values 断言继续留在目标仓库，并同时纳入本地入口。

## 镜像构建合同

`build-image` 使用 job 级变更过滤。无镜像相关变化的 PR 只跳过耗时构建，`Image Build required` 仍会完成；变更检测失败时终态检查失败。默认分支 push 与手动运行始终执行完整构建。

Workflow 负责事件、变更过滤、Buildx 环境、镜像 metadata 检查、tar artifact 和稳定终态；单平台构建命令由仓库 Makefile 提供：

```bash
make build-image
make build-image-amd64
make build-image-arm64
```

[`examples/build-image/Makefile`](examples/build-image/Makefile) 的 `build-image` 按 `IMAGE_PLATFORM` 构建并加载 `IMAGE_REF`；两个架构目标分别构建带 `-amd64` 与 `-arm64` 后缀的本地镜像。交叉构建 arm64 需要本机已经配置 QEMU 或使用原生 arm64 环境。私有 module Secret、build arg 和多 Dockerfile matrix 属于目标仓库，不加入公开模板。

模板对默认构建上下文 `.` 使用安全的 `**` 过滤，不会因为漏列语言、资产或配置路径而跳过构建。目标仓库只有在枚举完 Dockerfile `COPY`/`ADD`、构建脚本和上下文中的所有输入后才应收窄；若 Dockerfile 使用 `COPY . .`，通常应保留 `**`。构建模板不会登录 registry 或发布；多架构索引只由 `release-image` 在 GitHub Actions 中生成。

## Helm Chart 构建合同

`build-helm-chart` 用于需要可下载 `.tgz` 的 PR、默认分支或手工验证。它与 Helm Chart CI 使用同一套 job 级变更过滤和本地入口：

```bash
helm dependency build "$CHART_PATH"
make verify-helm-chart
rm -rf chart-packages
mkdir -p chart-packages
helm package "$CHART_PATH" --destination chart-packages
```

Workflow 只负责工具环境、调用本地合同、通用 package 和 artifact。Chart 声明 dependencies 时必须已有受 Git 跟踪的 `Chart.lock`，dependency build 不能改动 lock 或已提交的依赖树。Workflow 要求只生成一个非空包，并精确核对源 Chart 与 package 的 name、version 和 `<name>-<version>.tgz` 文件名；artifact 保留一天。没有 GHCR 登录、`helm push`、GitHub Release、tag 或 Pages 副作用。

若只需要 lint/render 和生成物漂移检查，使用 `helm-chart-ci`；需要在同一次检查中取得可下载安装包时，使用 `build-helm-chart`。两者都保持自包含，不互相调用。

## 镜像发布合同

`release-image` 只接受指向精确 commit 的稳定 tag `vX.Y.Z`。它拒绝预发布、build metadata 和前导零，镜像 tag 与 Git tag 保持一致，不自动创建 `latest`、major 或 minor 浮动 tag。发布运行按 tag 隔离且不会被取消。

写入前，Workflow 必须确认同名 GitHub Release 和 GHCR image tag 均不存在；已存在或远端状态无法确认时 fail-closed。随后按 QEMU → Buildx 顺序构建并发布 `linux/amd64` 与 `linux/arm64`，保留 Buildx provenance，写入 source/revision/version OCI 标签。发布后必须验证 action 返回的 index digest、tag 解析到同一 digest，以及两个目标平台均存在；只有这些检查通过后才创建 GitHub Release，并在正文记录不可变的 `image@sha256:...` 引用。

Dockerfile、context、build args、代码生成和仓库专属运行时验证仍由目标仓库维护。复杂项目应在发布前调用本地只读验证目标；需要对已发布镜像执行运行时验证时，应使用 `image@digest`，并明确部分失败的恢复方式。模板不覆盖已有 tag 或 Release；若 GHCR 已写入而后续步骤失败，应检查外部状态并选择清理或递增版本，不能把已有产物解释为成功重跑。

## Helm Chart 发布合同

`release-helm-chart` 只监听默认分支中目标 `Chart.yaml` 路径的 push，`Chart.yaml.version` 是唯一版本来源。触发后仍会比较 push 事件中的精确 `before` 与 `github.sha`，只有版本首次出现或严格递增时才进入 `package → publish`；版本未变时明确成功且不发布，基线不可解析、版本非法或版本降低时直接失败。tag、GitHub Release、手工输入和其他文件均不能另行指定版本。

发布前复用本地 CI 入口。打包本身是通用的发布编排，由 Workflow 使用标准 Helm 命令完成，并沿用相同的 `Chart.lock` 与依赖树漂移检查：

```bash
helm dependency build "$CHART_PATH"
make verify-helm-chart
helm package "$CHART_PATH" --destination chart-release
```

`make verify-helm-chart` 可从 [`examples/helm-chart/Makefile`](examples/helm-chart/Makefile) 复制并按仓库调整；CI、Build 与 Release 三个 Chart 模板使用同一入口，避免发布前验证与本地检查漂移。

Workflow 要求从 `CHART_PATH` 生成且只生成一个 `<name>-<version>.tgz`，并再次核对 package 元数据。随后，同一份 artifact 会用于 GitHub Release 和 `ghcr.io/<owner>/charts`，不会为不同目的重复打包。GitHub tag 使用 `<chart-name>-<chart-version>`，并设置 `make_latest: false`，避免 Chart Release 覆盖仓库的应用 Latest Release。

发布版本不可覆盖：同名 Git tag、GitHub Release 或 GHCR OCI Chart 只要已存在，或远端查询无法确认不存在，发布都会在写入前失败。Workflow 先推送并验证 OCI，再创建 GitHub Release；若远端写入后发生部分失败，应检查外部状态并显式清理，或递增 `Chart.yaml.version`，不能把跳过已有产物当作成功。

发布运行不会被取消。`Helm Chart Release required` 必须覆盖「版本未变且发布跳过」与「版本递增且完整发布」两种成功状态；规划、打包或发布任一阶段失败时均失败。GitHub Pages、签名、SBOM 和其他 registry 不属于通用模板，确有需要时由目标仓库单独实现。

## 共同约定

- PR 与普通分支上的旧运行会被取消，避免过期结果占用 runner；发布运行不会被取消。
- CI 模板只验证或生成 artifact，不会登录 registry 或发布。
- Go、Helm CI、镜像构建和 Chart 构建使用 job 级 changes-filter；过滤失败必须失败，不能被解释为「无相关变化」。
- 镜像发布模板只接受无预发布后缀、无前导零的 `vX.Y.Z` tag；Helm 发布只接受 `Chart.yaml` 中三段稳定且严格递增的版本。
- Helm 使用修订时的最新稳定版 v4.2.3，并显式记录相关工具版本，不继承历史的 Helm 3.20 固定值。特定项目如有兼容性约束，应在该项目 Workflow 中写明原因并固定版本。
- 所有模板均有超时、缓存或 artifact 校验，以及名称唯一的终态 job，便于目标仓库设置分支保护检查。

## 维护边界

这里提供的是可运行的起点，而不是跨仓同步机制。仓库 owner 应保留并维护复制后的完整 workflow；当模板更新或出现通用问题时，按仓库语义提交普通更新 PR。复杂发布需求，例如 Pages Helm repository、私有 registry、签名、SBOM 或自托管 runner，应在目标仓库单独实现和审查，不应把组织级公开模板变成大量开关的集合。
