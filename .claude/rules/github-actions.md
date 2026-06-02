# GitHub Actions 规范

本文档用于约束 Agent 在编写、修改和维护 GitHub Actions workflow 时的行为。Agent 修改 `.github/workflows/*.yml`、`.github/workflows/*.yaml`、`.github/actions/**`、发布脚本或与 CI/CD 强相关的配置时，必须优先遵守本文档。

本文内容按两条主线组织：

- CI 校验规则：代码校验、测试、构建、缓存、失败策略。
- Docker 发布规则：镜像仓库、部署、回滚、健康检查、Nginx 入口、镜像 Tag。

其余章节为通用约束，包括修改前要求、安全与密钥、验证要求和变更边界。

## 1. 基本原则

- 工作流必须服务于明确目标：代码校验、测试、构建、发布或部署，不得为了“看起来专业”随意增加流程。
- workflow 必须可重复、可审计、可回滚，不能依赖隐式环境状态。
- 默认遵循最小权限原则，禁止给 workflow 过宽的 `permissions`。
- 禁止在仓库中明文写入密钥、Token、私有证书、云厂商长期凭证。
- 任何会影响发布、部署或生产环境的变更，必须先确认影响范围和回滚方式。

## 2. 修改前要求

Agent 在修改 GitHub Actions 前必须先阅读：

1. 现有 `.github/workflows/**`
2. `README.md`
3. `package.json`、`pnpm-lock.yaml`、`go.mod`、`Makefile` 或项目已有构建脚本
4. 与 CI/CD 相关的文档、发布说明和部署说明

优先沿用项目已有约定：

- 已经存在 `make test`、`pnpm test`、`pnpm build`、`go test ./...` 就优先复用，不要在 workflow 里重复拼接新的命令链。
- 已经存在 reusable workflow、composite action 或统一脚本时，优先复用，不要复制一份新的实现。
- 已经有约定的缓存方式、环境变量命名和发布流程时，不要单独改成另一套风格。

## 3. CI 校验规则

### 3.1 按职责拆分

- 一个 workflow 只负责一类任务，例如 `ci`、`lint`、`test`、`build`、`release`、`deploy`。
- 不要把所有逻辑写进一个巨大 workflow。
- 测试、构建、发布如果触发条件和权限不同，应拆成不同 workflow。

### 3.2 命名要求

- workflow 名称必须能直接看懂用途，例如 `CI`, `Frontend CI`, `Backend Test`, `Release`.
- job 名称要表达阶段，例如 `lint`, `unit-test`, `build`, `publish`.
- step 名称要描述动作，不要只写 `Run`.

### 3.3 触发条件

- PR 校验应优先使用 `pull_request`。
- 主干分支校验或发布应优先使用 `push` 到受控分支。
- 定时任务使用 `schedule`，必须写明用途和时区假设。
- 手动发布、回滚、环境切换应使用 `workflow_dispatch`。
- 不要让高风险发布流程默认在所有分支自动执行。

### 3.4 权限控制

- workflow 只授予必要权限，例如只读 `contents: read`、PR 注释 `pull-requests: write`。
- 发布或推送 tag 的 job 才能请求更高权限，且应尽量局部授予到单个 job。
- 任何涉及 `id-token`、云部署、制品发布的权限提升，都必须说明原因。

## 4. 实现规范

### 4.1 可复现构建

- workflow 中调用的命令必须能在本地或 CI 中稳定复现。
- 优先调用仓库中的脚本或 package scripts，而不是直接堆长命令。
- 依赖安装、构建、测试、打包顺序必须明确，不要依赖隐式缓存结果。

### 4.2 版本固定

- GitHub Action、第三方 action、工具版本应尽量固定到稳定版本或 commit SHA。
- 禁止无节制使用漂移过大的 `@main`、`@master` 或不受控的宽泛版本。
- 升级 action 版本时必须说明兼容性和验证结果。

### 4.3 缓存与制品

- 缓存只缓存可重建、非敏感、收益明确的内容，例如依赖缓存、编译缓存。
- 不要缓存密钥、构建产物中的敏感文件或环境配置。
- artifact 命名必须能区分分支、平台、版本或构建号。

### 4.4 并行与矩阵

- 只有在任务互不依赖时才使用 matrix 或并行 job。
- matrix 的维度必须有明确意义，例如 `node-version`、`os`、`go-version`。
- 不要为了“跑得更快”盲目扩大矩阵维度。

### 4.5 失败策略

- 能提早失败的检查要放前面，例如格式化、lint、静态检查。
- 对于发布流程，关键步骤必须失败即停，不要吞错继续执行。
- 需要允许某些非关键步骤失败时，必须显式标注原因。

## 5. 安全与密钥

- 密钥只允许来自 GitHub Secrets、Environment secrets 或短期凭证，不得写入仓库文件。
- 生产环境相关 workflow 必须区分开发、预发、生产环境，不允许混用同一组配置。
- 需要访问云资源时，优先使用最小权限角色或短期 token。
- `pull_request` 场景默认不要暴露敏感 secrets。
- 如果 workflow 依赖外部服务回调、OIDC、云厂商角色授权或发布凭证，必须在文档中说明注入方式和失效处理。

### 5.1 Docker 镜像仓库

- 如果项目使用 Docker 镜像仓库，默认优先采用阿里云镜像仓库。
- 登录仓库所需的用户名和密码必须来自 GitHub `Secrets` 参数，不得写死在 workflow、脚本或仓库文件中。
- 前端镜像仓库地址和后端镜像仓库地址必须来自系统环境变量，不得硬编码在 workflow 中。
- 如果仓库地址、命名空间或地域依赖环境切换，必须在 README、部署文档或项目计划书中写明对应的环境变量名称和用途。
- 镜像推送和拉取的变量命名必须清晰区分前端、后端和公共基础镜像，避免混用。

## 6. Docker 发布规则

- 发布 workflow 必须写清楚触发条件、目标环境、产物来源和回滚方式。
- 部署前应完成构建和测试校验，不要在部署 job 里临时编译未验证代码。
- 需要 tag、release note 或版本号时，必须与项目已有版本策略一致。
- 生产发布必须保留审计线索，例如 release tag、commit SHA、artifact 版本或部署记录。

### 6.1 Docker 部署约束

- 使用 GitHub Actions 部署项目时，默认必须采用 Docker 方式交付和运行，不得绕过容器直接在服务器上安装应用运行时后手工启动服务。
- 服务器上必须存在一个统一的 `nginx` 容器作为入口层，负责对外暴露端口、转发请求和承载统一入口配置。
- 项目相关容器必须加入与该 `nginx` 容器相同的 Docker 网络，容器间通信应通过网络名或容器名完成，不得依赖裸露宿主机 IP 的临时写法。
- 如果项目包含前端、后端、数据库、缓存或其他辅助容器，部署流程必须明确哪些容器需要加入统一网络，以及哪些端口只允许在内部网络访问。
- 统一入口的 `nginx` 配置、Docker 网络名、容器名和端口映射规则必须文档化，避免不同 workflow 和服务器环境各自实现一套。
- 部署 workflow 应优先通过 `docker compose`、镜像拉取、容器重建或滚动替换完成更新，避免 SSH 上去手工 `docker run` 零散堆叠。
- 如需在服务器上执行初始化脚本、迁移脚本或健康检查，必须说明这些步骤是否在同一 Docker 网络内完成，以及失败后的回滚方式。

### 6.2 环境变量约定

- 所有部署相关变量必须有稳定、可预期的命名，不得在不同 workflow 中临时拼接不同写法。
- 前端镜像仓库地址、后端镜像仓库地址、服务器地址、Docker 网络名、`nginx` 容器名、镜像 tag 策略所需变量，必须通过环境变量注入，不得硬编码在 workflow 文件中。
- 环境变量命名必须能够区分前端、后端和公共基础设施，例如仓库地址、镜像标签、部署主机、端口映射、健康检查路径。
- 若变量仅在特定环境生效，必须在 README 或部署文档中说明适用环境和默认值。
- 推荐使用以下标准变量名。

说明：
- `GitHub Actions 变量`：配置在 GitHub 仓库、组织或 Environment 中，由 workflow 读取。
- `目标服务器变量`：配置在服务器上的部署文件、compose `.env` 或部署脚本中，由容器运行时或远程部署命令读取。

### GitHub Actions 变量

| 变量名 | 用途 | 推荐来源 | 是否必填 | 说明 |
| --- | --- | --- | --- | --- |
| `ALIYUN_DOCKER_USERNAME` | 阿里云镜像仓库登录用户名 | GitHub Secrets | 是 | 仅用于 `docker login` |
| `ALIYUN_DOCKER_PASSWORD` | 阿里云镜像仓库登录密码 | GitHub Secrets | 是 | 仅用于 `docker login` |
| `FRONTEND_IMAGE_REPO` | 前端镜像仓库地址 | GitHub Actions 变量 | 是 | 例如前端镜像完整仓库路径 |
| `BACKEND_IMAGE_REPO` | 后端镜像仓库地址 | GitHub Actions 变量 | 是 | 例如后端镜像完整仓库路径 |
| `DEPLOY_HOST` | 服务器地址或主机名 | GitHub Actions 变量 | 是 | 用于 SSH、部署和健康检查目标 |
| `DEPLOY_PORT` | SSH 连接端口 | GitHub Actions 变量 | 否 | 默认通常是 `22`，按服务器实际情况配置 |
| `DEPLOY_USER` | SSH 登录用户名 | GitHub Actions 变量 | 是 | 用于连接目标服务器执行部署命令 |
| `DEPLOY_SSH_KEY` | SSH 私钥 | GitHub Secrets | 是 | 用于 GitHub Actions 连接目标服务器 |
| `DEPLOY_KNOWN_HOSTS` | 目标服务器指纹 | GitHub Secrets 或环境变量 | 否 | 用于避免首次连接时的交互式确认 |
| `IMAGE_TAG_SHA` | commit 级镜像标签 | GitHub Actions 变量 | 是 | 通常自动生成，推荐用于回滚和审计 |
| `IMAGE_TAG_VERSION` | 版本号镜像标签 | GitHub Actions 变量 | 否 | 仅在正式发布时注入，例如 `v1.2.3` |
| `IMAGE_TAG_BRANCH` | 分支名镜像标签 | GitHub Actions 变量 | 否 | 通常由当前分支自动生成，例如 `main`、`develop` |

如果项目存在多个前端应用，例如 `H5`、`Web`、`Web Admin`，必须为每个前端单独定义镜像仓库地址、容器名和健康检查路径，命名应保持一致的后缀规则，例如：

- `H5_IMAGE_REPO`
- `WEB_IMAGE_REPO`
- `ADMIN_IMAGE_REPO`
- `H5_CONTAINER_NAME`
- `WEB_CONTAINER_NAME`
- `ADMIN_CONTAINER_NAME`
- `H5_HEALTHCHECK_PATH`
- `WEB_HEALTHCHECK_PATH`
- `ADMIN_HEALTHCHECK_PATH`

`IMAGE_TAG_SHA`、`IMAGE_TAG_VERSION` 和 `IMAGE_TAG_BRANCH` 可以继续复用同一套值，但镜像仓库和容器名必须按前端应用拆开，不能共用一组变量。

### 目标服务器变量

| 变量名 | 用途 | 推荐来源 | 是否必填 | 说明 |
| --- | --- | --- | --- | --- |
| `DOCKER_NETWORK_NAME` | 统一 Docker 网络名 | 目标服务器变量 | 是 | 供 `nginx` 和业务容器加入同一网络 |
| `NGINX_CONTAINER_NAME` | 统一入口 `nginx` 容器名 | 目标服务器变量 | 是 | 用于启动、更新和健康检查引用 |
| `FRONTEND_HEALTHCHECK_PATH` | 前端健康检查路径 | 目标服务器变量 | 否 | 例如 `/`、`/health` |
| `BACKEND_HEALTHCHECK_PATH` | 后端健康检查路径 | 目标服务器变量 | 否 | 例如 `/health`、`/api/health` |

如果项目存在多个前端应用，目标服务器变量也必须按应用拆分，例如 `H5_HEALTHCHECK_PATH`、`WEB_HEALTHCHECK_PATH`、`ADMIN_HEALTHCHECK_PATH`，并在 Nginx 配置中明确每个入口路由对应哪个容器。
- 如果项目因实际情况使用不同变量名，必须在 README、部署文档或项目计划书中写出映射关系，禁止只在 workflow 内部默默转换。
- GitHub Actions 连接目标服务器时，必须优先使用 SSH 私钥方式，不得将服务器密码直接写入 workflow。
- 连接前应明确目标服务器的地址、端口、用户名、私钥和 known_hosts 来源，避免把部署过程写成临时脚本。

### 6.3 发布流程

- GitHub Actions 发布流程应按固定顺序执行：构建镜像、推送镜像、连接服务器、拉取镜像、更新容器、执行健康检查、确认通过后结束。
- 发布 job 不得跳过构建或测试直接推送生产镜像。
- 如果前后端需要分开发布，必须写清楚依赖顺序，避免前端先切流但后端未就绪。
- 发布过程中如需执行数据库迁移，必须明确迁移是在容器内还是在独立迁移容器中执行，并说明与应用启动的先后关系。
- 所有发布步骤必须尽量由脚本或 compose 配置承接，不要把复杂逻辑散落在 workflow 多个 `run` 段中。

### 6.4 回滚规则

- 回滚默认优先回到上一个稳定镜像 tag 或上一个可用 release 版本，不得依赖手工重建未知状态。
- 回滚流程必须与发布流程使用同一套 Docker 网络、`nginx` 入口和镜像仓库约定，避免回滚时切换到另一套部署方式。
- 如果回滚需要保留旧容器，应明确旧容器命名、切换方式和清理时机。
- 每次发布都应保留可回滚的镜像 tag 或制品版本，禁止只保留 `latest` 而没有历史版本。

### 6.5 健康检查

- 部署完成后必须执行健康检查，至少覆盖 `nginx` 入口可达性和应用核心健康接口。
- 健康检查失败时，workflow 必须明确失败退出，不得静默忽略。
- 如果项目包含前端和后端，健康检查应分别验证入口页、API 入口或双方各自约定的检查地址。
- 健康检查路径、超时、重试次数和成功条件必须文档化。
- 推荐默认检查变量：
  - 前端：`FRONTEND_HEALTHCHECK_PATH`
  - 后端：`BACKEND_HEALTHCHECK_PATH`
- 如果 `nginx` 负责统一入口转发，至少应验证 `NGINX_CONTAINER_NAME` 对外暴露的入口路径或域名可访问。

### 6.6 Nginx 边界

- `nginx` 容器只负责统一入口、反向代理和基础转发，不得承载业务逻辑。
- 容器端口暴露原则上应优先由 `nginx` 统一对外暴露，业务容器默认仅在内部网络提供服务端口。
- `nginx` 配置归属必须明确，避免前端、后端、部署脚本各自修改同一份配置而互相覆盖。
- 如果需要对外开放多个入口，必须在文档中明确各入口路由和对应后端容器。
- 推荐 `nginx` 相关标准变量名：
  - `NGINX_CONTAINER_NAME`
  - `NGINX_CONF_PATH`
  - `NGINX_DOCKER_PORT`
- 如果项目新增多个入口域名或路径前缀，必须记录每个入口映射到哪个容器和健康检查地址。

### 6.7 镜像 Tag 策略

- 镜像 tag 必须可追溯、可回滚、可区分构建来源。
- 默认至少保留 commit SHA 维度的 tag，必要时可再叠加分支名或版本号，但不得只依赖不透明的 `latest`。
- 如果项目已有语义化版本策略，镜像 tag 应同时兼顾版本号与构建标识。
- 推送、拉取、回滚时使用的 tag 规则必须统一，不能在不同 workflow 中采用不同语义。

## 7. 验证要求

修改 GitHub Actions 后，Agent 必须检查：

1. workflow YAML 语法是否正确
2. 触发条件是否符合预期
3. 权限是否过宽
4. 命令是否与本地/CI 脚本一致
5. 是否存在明显的 secrets 泄露风险
6. 是否需要更新 README、部署文档或项目计划书

如果仓库支持本地验证，应优先运行可用的检查命令或至少做静态检查；如果无法运行，必须在交付说明中写明原因和风险。

## 8. 变更边界

- 只改与当前 CI/CD 目标直接相关的文件，不要顺手重构整个部署体系。
- 不要在未确认需求时，顺便引入新的发布平台、复杂环境编排或额外自建动作。
- 如果需要新增 reusable workflow 或 composite action，必须说明复用收益和影响范围。
- 如果修改会影响 PR 检查、发布节奏或部署门禁，必须明确提醒用户。
