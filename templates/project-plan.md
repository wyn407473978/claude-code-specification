# 项目计划书

> 本模板用于在新项目中明确“做什么、不做什么、按什么顺序做”。建议保存为 `docs/project-plan.md`，再让 Claude Code 先阅读计划书和 `CLAUDE.md` 后开始开发。

## 1. 项目目标

说明项目要解决什么问题，面向谁，最终交付什么。

- 项目名称：
- 一句话目标：
- 目标用户：
- 核心价值：
- 最终交付物：

## 2. 用户与场景

描述典型用户和关键使用场景。

| 用户类型 | 使用场景 | 主要目标 | 关键痛点 |
| --- | --- | --- | --- |
|  |  |  |  |

## 3. 功能范围

### 必须实现

- 

### 暂不实现

- 

### 后续可能实现

- 

## 4. 技术栈

- 前端：React 19+、TypeScript、Vite、pnpm、React Router 或 TanStack Router、TanStack Query、Zustand、React Hook Form、Axios 或 Fetch 封装。
- UI 方案：按项目类型选择其一，例如 Ant Design、Arco Design、shadcn/ui 或 Tailwind CSS + 自定义组件；同一项目不混用多套 UI 体系。
- 后端：Go、Gin、Gorm、Viper、go-playground/validator、log/slog 或 zap、swaggo/swag。
- 数据库：PostgreSQL，默认使用 `bigint generated always as identity`、`timestamptz`、`jsonb`、`check`、`foreign key`、`partial index`。
- 包管理与脚本：前端使用 pnpm；后端使用 Go Modules；常用命令必须写入 README。
- CI/CD：如使用 GitHub Actions，必须写清楚触发条件、校验内容、发布/部署流程、制品管理和 secrets 注入方式。
- 部署方式：如使用 GitHub Actions，默认必须采用 Docker 部署；服务器上应有统一的 Nginx 容器作为入口，项目相关容器必须加入同一 Docker 网络；其余部署方式需写明原因和替代方案。
- 第三方服务：文件存储默认使用阿里云 OSS；如涉及登录、支付、邮件、短信、AI API 等，必须写清用途、替代方案、成本和密钥管理方式。
- Docker 镜像仓库：如使用阿里云镜像仓库，必须写清用户名和密码的 Secret 来源，以及前端、后端镜像仓库地址对应的系统环境变量名称。
- GitHub Actions 发布：必须写清楚环境变量约定、发布步骤、回滚规则、健康检查、Nginx 边界和镜像 Tag 策略。
- 推荐环境变量：

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

- 文件存储：默认使用阿里云 OSS，计划书中必须说明 Bucket、Region、访问权限、上传方式、回调策略、生命周期规则、CDN 是否启用、密钥注入方式和本地开发替代方案。
- 其他约束：如需偏离以上默认技术栈，必须在本计划书中说明原因、影响范围和迁移成本。

## 5. 数据模型

说明核心实体、实体关系和关键字段。

### 核心实体

- 

### 核心关系

- 

### 关键字段与约束

- 

## 6. 页面与接口

### 页面

| 页面 | 目标 | 核心功能 | 状态要求 |
| --- | --- | --- | --- |
|  |  |  |  |

### 接口

| 接口 | 方法 | 说明 | 权限 |
| --- | --- | --- | --- |
|  |  |  |  |

### 权限

- 公开访问：
- 登录后访问：
- 管理员访问：
- 其他权限边界：

## 7. 开发阶段

### MVP 版本

- 目标：
- 范围：
- 验收标准：
- 不包含：
- 完成后必须暂停等待用户确认：是

### 完整版本

- 目标：
- 范围：
- 验收标准：
- 不包含：
- 开始条件：用户确认 MVP 版本后继续

### 后续版本

- 目标：
- 范围：
- 验收标准：
- 不包含：

## 8. 自动执行与暂停点

说明 Claude Code 是否可以按计划书自动执行，以及哪些信息缺口可以预留。

- 自动执行模式：开启 / 关闭
- 默认执行节奏：先完成 MVP 版本并暂停；用户确认后继续完整版本；完整版本完成后再次暂停。
- MVP Gate 停止条件：
- Final Gate 停止条件：
- 允许 Agent 自行决策的范围：
- 需要记录但不阻塞开发的信息：
  - 第三方服务密钥：
  - OSS Bucket、Region、CDN、回调地址：
  - 域名、部署环境、环境变量：
  - 支付、短信、邮件、AI API 等外部服务配置：
  - 品牌素材、正式文案、运营数据：
- 提前暂停的硬阻塞条件：
  - 核心业务规则缺失：
  - 数据模型或权限边界无法确定：
  - 需要执行破坏性命令：
  - 安全、隐私、资金或生产数据风险无法规避：
- Assumption 记录要求：
- TODO 或配置占位要求：

## 9. Agent Teams 策略

说明本项目什么时候允许启用多 Agent，默认团队组合和协作顺序。

- 默认是否启用 Agent Teams：
- 适合启用的场景：
- 不启用的场景：
- 默认 Lead Agent：
- 可能需要的专业 Agent：
  - Product Planner Agent：
  - Go Backend Agent：
  - Frontend Agent：
  - Database Agent：
  - QA Review Agent：
  - Docs Agent：
- 推荐串行顺序：
- 推荐并行顺序：
- 文件所有权约束：
- 最终验收要求：

## 10. 验收标准

项目做到什么程度算完成。

- 功能验收：
- 体验验收：
- 性能验收：
- 安全验收：
- 工程验收：

## 11. 约束与偏好

写清个人偏好和约束，避免 Agent 自行脑补。

- 优先级：
- 不希望引入的依赖：
- UI 风格：
- 语言与文案：
- 数据与隐私要求：
- 成本限制：
- 维护偏好：

## 12. 风险与注意事项

列出需要提前关注的问题。

- 安全风险：
- 数据风险：
- 权限风险：
- 迁移风险：
- 并发风险：
- 第三方服务风险：
- CI/CD 风险：
- 未确认问题：

## 13. 当前决策记录

记录已经确定的重要决策，避免后续反复摇摆。

| 日期 | 决策 | 原因 | 影响 |
| --- | --- | --- | --- |
|  |  |  |  |
