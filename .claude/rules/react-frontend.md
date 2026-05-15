# React 前端 Agent 开发规范

本文档用于约束前端 Agent 在 React 项目中的开发行为，目标是交付精美、可维护、可长期迭代的前端项目。Agent 必须优先遵守本规范，再根据具体业务需求补充实现细节。

## 1. 核心目标

前端项目不能只追求“能跑”或“页面看起来还行”。每一次开发都必须同时满足以下目标：

- 视觉统一：页面风格、间距、颜色、字号、组件状态一致。
- 组件清晰：组件职责单一，命名准确，层级明确。
- 业务内聚：同一业务模块的页面、接口、类型、组件、Hooks 尽量放在一起。
- 接口规范：请求入口统一，错误处理统一，业务接口有明确类型。
- 状态可控：区分组件状态、全局状态、服务端缓存、表单状态和 URL 状态。
- 类型安全：默认使用 TypeScript 严格类型，禁止随意使用 `any`。
- 错误可查：页面错误、接口错误、权限错误、异常状态都有明确处理。
- 后期好改：目录、命名、抽象和依赖选择都要服务长期迭代。

### 1.1 项目规模分级

Agent 必须先判断项目规模，再决定规范执行强度，避免个人项目被不必要的企业级流程拖慢。

```txt
轻量项目：Demo、个人工具、验证性页面、一次性活动页
长期项目：个人长期维护产品、用户平台、管理后台、SaaS
高风险项目：涉及支付、权限、账号、订单、隐私数据、生产运营的系统
```

执行原则：

- 轻量项目必须保证可运行、类型基本安全、样式统一、核心状态完整，但可暂不接入复杂权限、暗色主题、E2E 和完整 Mock 体系。
- 长期项目必须建立稳定目录、请求封装、状态分层、Design Token、README、必要测试和交付检查。
- 高风险项目必须严格执行权限、安全、错误处理、测试、审计、确认流程和构建验证。
- 已有项目优先沿用现有工程约定，不因默认模板强行迁移技术栈。
- 规范裁剪必须在交付报告中说明原因和后续补齐建议。

## 2. 默认技术栈

新项目默认使用：

```txt
React 19+
TypeScript
Vite
pnpm
ESLint
Prettier
UI 方案按项目类型选择其一
React Router / TanStack Router
TanStack Query
Zustand
Axios / Fetch 封装
React Hook Form
```

选型原则：

- 优先选择稳定、社区成熟、维护活跃的库。
- 不为简单需求引入重型依赖。
- 不重复造轮子，例如请求缓存、表单校验、路由、虚拟列表应优先使用成熟方案。
- 不随意混用多个同类库，例如同一项目不要同时使用多个全局状态库或多个 UI 体系。
- 引入新依赖前必须说明用途、替代方案和维护成本。
- UI 方案必须按项目类型选择其一，禁止无迁移方案地混用多套 UI 体系。

## 3. 推荐目录结构

项目应按业务模块和应用基础设施组织，不要把所有代码都堆进 `components`、`utils`、`pages`。

```txt
src/
  app/                  # 应用入口、全局 Provider、应用级初始化
  assets/               # 图片、字体、静态资源
  components/           # 全局通用组件
    ui/                 # 基础 UI 组件
    business/           # 跨业务复用组件
  features/             # 业务功能模块
    user/
      api.ts
      types.ts
      hooks.ts
      components/
      pages/
    music/
      api.ts
      types.ts
      hooks.ts
      components/
      pages/
  layouts/              # 页面布局
  router/               # 路由配置
  services/             # 请求实例、接口基础封装
  stores/               # 全局状态
  hooks/                # 全局 Hooks
  utils/                # 通用工具函数
  constants/            # 全局常量
  styles/               # 全局样式
  types/                # 全局类型
```

目录规则：

- 业务模块内部优先自包含：页面、接口、类型、Hooks、业务组件放在同一个 `features/<name>` 下。
- 只有真正跨模块复用的内容，才放到全局 `components`、`hooks`、`utils`、`types`。
- 不允许出现职责不清的目录名，例如 `common2`、`newPages`、`testComponents`。
- 避免过深嵌套，常规业务模块建议不超过 4 层。

### 3.1 按业务实体拆分文件

涉及多个业务实体、多个管理对象或多个页面资源时，必须按业务实体拆分文件，禁止把用户、角色、菜单、订单等不同实体的接口、类型、页面和状态集中写进一个总文件。

前端拆分规则：

- `types`：一个主要业务实体一个类型文件，例如 `user.ts`、`role.ts`、`menu.ts`；聚合导出可放在 `index.ts`。
- `api` 或 `services`：按业务实体拆分，例如 `userApi.ts`、`roleApi.ts`、`menuApi.ts`，或 `services/adminUser.ts`、`services/adminRole.ts`。
- `hooks`：按业务实体或页面流程拆分，例如 `useUsers.ts`、`useRolePermissions.ts`。
- `pages`：一个页面一个文件，例如 `UserManagementPage.tsx`、`RoleManagementPage.tsx`、`MenuManagementPage.tsx`。
- `components`：可复用业务组件按实体或功能拆分，例如 `UserForm.tsx`、`RolePermissionTree.tsx`。
- `router`：多个业务页面路由较多时，应拆出模块路由文件，避免把所有路由堆在一个巨大配置中。

允许例外：

- 极小 Demo 可以暂时合并少量类型或接口，但一旦出现两个以上业务实体、三个以上接口或文件超过 200 行，必须优先拆分。
- 公共请求实例、响应泛型、分页类型等通用基础类型可以集中放在 `api.ts`、`request.ts` 或 `common.ts`，但业务实体类型必须拆开。

## 4. UI 设计系统规范

Agent 开发页面前，必须先确认或建立项目级 UI 规范。没有设计稿时，也要保持统一、克制、清晰。

至少定义：

```txt
颜色规范
主题适配规范
字体大小
行高
间距
圆角
阴影
按钮
表单
卡片
表格
弹窗
空状态
加载状态
错误状态
页面布局
响应式断点
```

默认建议：

```txt
主色：#1677ff
成功色：#52c41a
警告色：#faad14
错误色：#ff4d4f
页面最大宽度：1200px
模块间距：24px
表单项间距：16px
卡片圆角：8px 到 12px
```

UI 实现要求：

- 同类页面使用一致的布局、标题、工具栏、筛选区、表格、分页和操作按钮。
- 不用大面积装饰性渐变、无意义背景图或杂乱阴影掩盖信息结构。
- 后台、SaaS、管理系统应优先保证信息密度、扫描效率和操作稳定性。
- 按钮、表单、表格、弹窗、通知、空状态必须有统一样式。
- 页面必须覆盖 loading、empty、error、success、disabled、permission denied 等状态。
- 长期维护型项目、SaaS、后台管理系统、用户平台型项目必须支持亮色/暗色主题；临时页面或一次性 Demo 可按项目要求裁剪，但样式仍应使用语义 Token。

## 5. 组件分层规范

组件分为 4 层：

```txt
基础组件：Button、Input、Modal、Card、Table
业务组件：UserSelect、MusicCard、ScorePanel
页面组件：MusicCreatePage、UserListPage
布局组件：DashboardLayout、AuthLayout
```

组件规则：

- 基础组件只处理通用 UI 表现，不包含具体业务逻辑。
- 业务组件可以理解业务实体，但不直接承担完整页面流程。
- 页面组件负责组织页面数据、业务流程和状态展示。
- 布局组件负责导航、侧边栏、顶部栏、页面容器等结构。
- 普通组件建议控制在 100 行以内。
- 复杂组件建议控制在 200 行以内。
- 超过 300 行必须优先考虑拆分。

命名要求：

```txt
Good:
MusicScoreCard
GenerateMusicForm
UserPermissionTable
DashboardLayout

Bad:
Card1
MyComponent
TestPage
IndexView
NewComp
```

## 6. 接口请求规范

禁止在页面中到处直接写 `axios.get()`、`fetch()` 或拼接接口地址。

推荐结构：

```txt
services/request.ts       # 请求实例、拦截器、错误归一化
features/music/api.ts     # 音乐模块接口
features/user/api.ts      # 用户模块接口
```

示例：

```ts
// features/music/api.ts
export function createMusic(data: CreateMusicParams) {
  return request.post<CreateMusicResult>('/music/create', data)
}
```

页面中只能调用业务接口函数：

```ts
const result = await createMusic(formData)
```

接口规范：

- 每个接口必须定义请求参数类型和响应类型。
- Token、超时、错误提示、登录失效等逻辑必须在请求层统一处理。
- 业务接口文件只暴露明确函数，不暴露底层请求实例给页面。
- 列表、详情、分页、刷新等服务端数据优先接入 TanStack Query。

## 7. 状态管理规范

不要把所有状态都放进全局 store。

状态归属规则：

```txt
组件内部状态：useState
跨组件简单状态：Zustand
服务端数据缓存：TanStack Query
表单状态：React Hook Form
URL 状态：searchParams
持久化偏好：localStorage + 封装 Hook 或 Zustand persist
```

Agent 必须先判断状态归属，再实现状态逻辑：

- 只影响当前组件的状态，不允许放入全局 store。
- 接口返回的列表、详情、分页、缓存、刷新，不应塞进 Zustand。
- 搜索、筛选、分页这类可分享页面状态，优先放到 URL。
- 表单校验、提交中、错误信息，优先交给表单库和接口层处理。

## 8. TypeScript 类型规范

项目必须开启 TypeScript strict。禁止为了省事使用 `any`。

必须定义：

```txt
接口请求参数类型
接口响应类型
组件 Props 类型
业务实体类型
枚举类型
表单类型
路由参数类型
Store 类型
```

示例：

```ts
export interface MusicItem {
  id: string
  title: string
  style: string
  duration: number
  score: number
  audioUrl: string
}

export interface CreateMusicParams {
  prompt: string
  style: string
  mood: string
  lyrics?: string
}
```

类型规则：

- 禁止无理由使用 `any`，确实无法确定时优先使用 `unknown`。
- API 类型放在对应 feature 的 `types.ts`，跨模块共享类型放到 `src/types`。
- Props 类型命名使用 `ComponentNameProps`。
- 不为了复用而过度抽象类型，业务含义不同就保持独立。

## 9. 页面状态规范

每个页面都必须设计完整状态，而不是只实现“有数据”的理想情况。

页面状态清单：

```txt
首次加载
局部加载
加载失败
空数据
提交中
提交成功
提交失败
权限不足
网络异常
接口超时
删除确认
二次确认
```

示例：音乐列表页面至少要覆盖：

```txt
暂无音乐
正在生成中
生成失败
重新生成
加载更多
删除确认
加载失败重试
```

页面状态要求：

- loading 不能导致页面大面积跳动。
- empty 状态必须告诉用户当前没有什么，以及下一步能做什么。
- error 状态必须提供重试或返回路径。
- submit 状态必须防止重复提交。
- 危险操作必须确认。

## 10. 性能规范

性能优化应先从结构和数据流入手，不要滥用 `useMemo`、`useCallback`。

优先级：

```txt
组件拆分
状态下沉
避免无意义全局状态
路由懒加载
图片懒加载
列表虚拟滚动
接口缓存
减少重复请求
稳定 key
减少不必要的 Provider 范围
```

性能规则：

- 不在渲染过程中执行重计算、请求、写存储或产生副作用。
- 长列表必须考虑分页、无限滚动或虚拟滚动。
- 大模块页面必须路由懒加载。
- 图片必须设置尺寸约束，避免布局抖动。
- 只有在存在实际性能问题或引用稳定性需求时，才使用 memo 化。

## 11. 表单规范

复杂表单优先使用 React Hook Form。

表单要求：

- 表单字段必须有明确类型。
- 校验规则集中管理，不散落在 JSX 中。
- 提交中禁用提交按钮。
- 提交失败展示明确错误。
- 编辑表单必须处理初始值加载、重置、脏数据提示。
- 表单组件不直接写接口请求，提交逻辑由页面或 Hook 编排。

## 12. 权限规范

权限控制必须统一封装，禁止在页面中散落字符串判断。

推荐方式：

```txt
components/business/PermissionGuard.tsx
hooks/usePermission.ts
constants/permissions.ts
```

规则：

- 路由权限、按钮权限、数据权限要区分。
- 无权限页面要有明确提示和返回入口。
- 按钮无权限时，根据场景选择隐藏或禁用，并保持项目一致。
- 权限 key 必须集中定义。

## 13. 错误处理与日志规范

错误处理必须分层：

```txt
请求错误：services/request.ts
业务错误：feature Hook 或页面处理
渲染错误：ErrorBoundary
表单错误：表单库处理
全局错误：统一日志上报
```

要求：

- 登录失效、403、404、500、网络超时要有统一处理。
- 用户可理解的错误要展示给用户。
- 调试用错误要记录日志，避免直接暴露技术细节。
- 关键业务失败必须保留上下文，方便排查。

## 14. 工程规范

必须配置：

```txt
ESLint
Prettier
TypeScript strict
Git commit 规范
中文注释规范
目录命名规范
组件命名规范
接口命名规范
环境变量规范
构建部署规范
错误日志规范
```

推荐脚本：

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "lint": "eslint .",
    "format": "prettier --write .",
    "typecheck": "tsc --noEmit"
  }
}
```

提交前必须至少通过：

```txt
pnpm lint
pnpm typecheck
pnpm build
```

提交规范：

- 项目内容发生修改后，Agent 必须检查 Git 状态并在交付报告中说明建议提交内容。
- 是否自动提交以用户授权或项目约定为准。
- 如用户授权自动提交，则只提交与当前需求相关的改动。
- 如用户未授权提交，则不得擅自提交，只需给出建议提交信息。
- 提交信息必须使用中文，清楚描述本次改动目的。
- 提交前必须查看 `git status`，确认只提交与当前需求相关的文件。
- 不允许把临时文件、构建产物、调试日志、无关改动混入提交。
- 推荐提交信息格式：`类型：说明`，例如 `文档：补充前端 Agent 开发规范`、`修复：处理音乐列表空状态`。

## 15. 环境变量规范

环境变量必须按环境区分：

```txt
.env.development
.env.test
.env.production
```

规则：

- Vite 客户端环境变量必须使用 `VITE_` 前缀。
- 不允许把密钥、私有 token、后端私钥放到前端环境变量。
- API base URL、运行模式、日志开关等应通过环境变量配置。
- README 必须说明每个环境变量的用途。

## 16. README 规范

项目 README 至少包含：

```txt
项目介绍
技术栈
启动方式
构建方式
目录结构
开发规范
环境变量
接口约定
路由说明
常用脚本
部署方式
```

README 必须让新 Agent 或新开发者能在短时间内理解项目如何启动、如何开发、如何验证。

## 17. Agent 开发流程

Agent 接到需求后必须按以下顺序工作：

```txt
1. 阅读现有目录、README、package.json、路由、相关 feature。
2. 判断需求属于新页面、新组件、接口接入、状态改造、样式优化还是 bug 修复。
3. 明确改动边界，不做无关重构。
4. 复用现有 UI、Hooks、请求封装、类型和工具函数。
5. 先补类型和接口，再实现页面逻辑。
6. 覆盖 loading、empty、error、success 等页面状态。
7. 保持组件小而清晰，超过规模及时拆分。
8. 运行 lint、typecheck、build 或项目已有测试。
9. 检查 Git 状态，按用户授权或项目约定决定是否提交。
10. 总结改动内容、验证结果和遗留风险。
```

## 18. 禁止事项

Agent 不允许：

- 在页面里直接散落请求逻辑。
- 无理由使用 `any`。
- 把服务端数据全部塞进 Zustand。
- 为了一个简单需求引入大型依赖。
- 新增和项目 UI 风格冲突的组件样式。
- 创建 `Component1`、`NewPage`、`TestView` 这类临时命名。
- 写几百行不拆分的组件。
- 只实现有数据状态，不处理 loading、empty、error。
- 跳过类型检查和构建验证就声称完成。
- 擅自重构无关模块。
- 在前端暴露密钥。
- 未经用户授权或项目约定擅自提交 Git。
- 自动提交时使用非中文提交信息，或混入无关改动。

## 19. 必须项清单

```txt
1. React + TypeScript + Vite
2. 按业务模块拆目录
3. 统一 UI 设计规范
4. 组件分层：基础组件 / 业务组件 / 页面组件 / 布局组件
5. 接口统一封装
6. 状态管理分层
7. 禁止滥用 any
8. 页面状态完整：loading / empty / error / success
9. ESLint + Prettier + typecheck
10. 路由懒加载和组件拆分
11. 表单统一处理
12. 权限控制统一封装
13. 错误处理统一封装
14. 环境变量区分 dev/test/prod
15. README 写清楚启动、构建、目录、规范
16. 页面设计按固定结构交付
17. Design Token 统一沉淀，禁止随机硬编码样式
18. 路由 path、权限、懒加载统一管理
19. Mock 数据与真实接口类型保持一致
20. 重要需求补充或更新测试
21. 每次交付输出修改范围、验证结果和风险说明
22. 长期项目必须具备亮色/暗色主题能力
23. 相关方法必须添加必要中文注释
24. 项目内容修改后必须说明 Git 状态和建议提交内容
```

## 20. 最终验收标准

一个需求完成后，Agent 必须确认：

- 页面视觉与项目整体一致。
- 长期项目的页面和组件在亮色、暗色主题下都可读、可用。
- 页面标题区、筛选区、内容区、操作区、反馈区结构完整。
- 组件命名、目录位置、职责划分合理。
- 接口、类型、状态、错误处理都有明确边界。
- 路由、权限、API 响应、Mock、枚举等没有散落魔法字符串。
- loading、empty、error、success 等状态可用。
- 没有无意义的全局状态和重复请求。
- 没有随意使用 `any`。
- 没有随机硬编码颜色、间距、圆角、阴影。
- 关键流程已有必要测试或说明了测试缺口。
- 代码通过 lint、typecheck、build 或项目要求的测试。
- 关键方法、复杂逻辑、业务编排已有必要中文注释。
- 已检查 Git 状态，并按用户授权或项目约定处理提交。
- README 或相关文档在必要时已更新。

最终交付不是“代码写完”，而是达到：

```txt
视觉统一
组件清晰
业务内聚
接口规范
状态可控
类型安全
错误可查
后期好改
```

## 21. 页面设计交付标准

Agent 生成或改造页面时，必须按稳定的信息结构组织页面。除非产品形态明确不适用，常规业务页面应包含以下区域：

```txt
1. 页面标题区
   - 标题
   - 副标题或说明
   - 主要操作按钮

2. 筛选区
   - 搜索框
   - 筛选条件
   - 重置按钮
   - 查询按钮

3. 内容区
   - 表格 / 卡片 / 列表 / 表单
   - loading 状态
   - empty 状态
   - error 状态

4. 操作区
   - 新增
   - 编辑
   - 删除
   - 查看详情
   - 批量操作

5. 反馈区
   - 成功提示
   - 失败提示
   - 二次确认
```

页面设计要求：

- 标题区要让用户立即知道当前页面是什么、能做什么。
- 筛选区不能挤压主要内容，字段过多时应折叠或分组。
- 内容区优先保证可读性、扫描效率和操作稳定性。
- 操作按钮应区分主操作、次操作和危险操作。
- 危险操作必须二次确认，确认文案要清楚说明后果。
- 同类页面的布局、按钮位置、分页位置、空状态表现必须一致。

## 22. Design Token 规范

项目中的颜色、字号、间距、圆角、阴影不允许在页面中随意硬编码。必须沉淀为项目统一 Token，并通过 UI 组件、主题配置或样式变量使用。

推荐位置：

```txt
theme.ts
tailwind.config.ts
styles/tokens.css
constants/design.ts
```

常用 Token：

```txt
color.primary
color.success
color.warning
color.error
color.text.primary
color.text.secondary
color.border
color.background
color.surface
color.surfaceElevated
radius.sm
radius.md
radius.lg
spacing.xs
spacing.sm
spacing.md
spacing.lg
font.size.sm
font.size.md
font.size.lg
shadow.sm
shadow.md
```

禁止：

```tsx
<div style={{ color: '#1677ff', marginTop: 17 }} />
```

推荐：

```tsx
<Button variant="primary" />
<Card className="rounded-xl p-6 shadow-sm" />
```

规则：

- 新增颜色前必须判断是否能复用现有 Token。
- 数值间距优先使用 Token 或 Tailwind 约定值。
- 同一语义状态只能有一套颜色表达，例如成功、警告、错误、禁用。
- 页面级样式不能绕过设计系统随意创造新视觉语言。
- 所有颜色 Token 必须同时提供亮色和暗色主题取值。
- 组件禁止直接依赖固定黑色或固定白色表达文本、背景、边框和阴影。

## 23. Agent 最小改动原则

Agent 修改代码时必须控制改动范围。

必须遵守：

- 只修改与当前需求直接相关的文件。
- 不因为个人偏好重构已有结构。
- 不擅自替换 UI 库、请求库、状态库、路由库。
- 不修改无关样式。
- 不删除已有逻辑，除非明确确认无用。
- 不大范围格式化无关文件。
- 不引入破坏性 API 变更。
- 不把临时调试代码、测试数据、console 噪声留在业务代码中。

每次交付必须说明：

```txt
本次新增了什么
本次修改了什么
本次删除了什么
是否影响已有功能
是否存在风险
如何验证
```

## 24. 路由规范

路由必须集中管理，禁止在页面中临时拼接未知路径。

推荐结构：

```txt
router/
  routes.tsx
  guards.tsx
  lazy.tsx
constants/
  routes.ts
```

路由常量示例：

```ts
export const ROUTES = {
  HOME: '/',
  MUSIC_LIST: '/music',
  MUSIC_CREATE: '/music/create',
  MUSIC_DETAIL: '/music/:id',
} as const
```

路由要求：

- 路由 path 必须集中定义。
- 页面组件必须懒加载。
- 路由权限必须统一处理。
- 必须提供 404 页面。
- 必须提供 403 页面。
- 页面跳转禁止硬编码字符串，优先使用路由常量。
- 动态路由参数必须有类型约束和异常兜底。
- 菜单、高亮、面包屑应尽量从路由元信息生成，避免多处维护。

## 25. API 响应结构规范

前端默认期望后端返回统一结构：

```ts
export interface ApiResponse<T> {
  code: number
  message: string
  data: T
  requestId?: string
}
```

分页结构：

```ts
export interface PageResult<T> {
  list: T[]
  total: number
  page: number
  pageSize: number
}
```

请求层负责处理：

- HTTP 错误。
- 业务 code 错误。
- token 失效。
- 网络超时。
- 重复请求。
- requestId 日志追踪。
- 响应结构异常。

规则：

- feature API 函数不直接返回未归一化的响应对象。
- 页面不直接判断 HTTP 状态码。
- 业务错误文案优先使用后端 `message`，必要时前端做兜底。
- 关键请求失败时应保留 `requestId`，方便后端排查。

## 26. Mock 数据规范

当后端接口未完成时，可以使用 Mock，但必须遵守统一规范。

推荐位置：

```txt
mocks/
features/<feature>/mock.ts
```

Mock 规则：

- Mock 字段必须和真实接口类型一致。
- Mock 数据不能直接写死在页面组件中。
- Mock 数据应覆盖正常、空数据、错误、边界值等情况。
- 接口完成后必须移除 Mock 或切换到真实接口。
- README 中必须说明当前哪些接口仍为 Mock。

禁止：

```tsx
const data = [
  { name: '测试1' },
  { name: '测试2' },
]
```

直接写在页面组件中。

## 27. 测试规范

项目至少包含以下测试策略：

```txt
1. 工具函数测试
   - 使用 Vitest
   - 重点测试 utils、formatters、validators

2. 组件测试
   - 使用 React Testing Library
   - 重点测试表单、弹窗、复杂业务组件

3. 接口 Mock
   - 使用 MSW 或项目统一 Mock 方案

4. E2E 测试
   - 核心流程可使用 Playwright
   - 例如登录、创建、编辑、删除、支付、生成任务等流程
```

Agent 完成重要需求后，必须优先补充或更新相关测试。

测试要求：

- 纯函数、格式化、校验逻辑必须优先单测覆盖。
- 复杂表单必须测试校验、提交中、提交成功、提交失败。
- 关键业务流程必须有组件测试或 E2E 测试。
- 修复 bug 时应补充回归测试，除非项目暂时没有测试基础，并在交付报告中说明。
- 不允许为了通过测试而删除有效断言。

## 28. 响应式与可访问性规范

响应式要求：

- 页面至少适配桌面端和常见平板宽度。
- 移动端页面不能出现不可控横向滚动。
- 表格在小屏幕下应提供横向滚动、列裁剪或卡片化方案。
- 关键操作按钮在移动端必须可点击、可识别。
- 固定宽高组件要设置合理的 `min-width`、`max-width`、`aspect-ratio` 或容器约束。

可访问性要求：

- 图片必须提供有意义的 `alt`，装饰图可使用空 `alt`。
- 表单项必须有 label。
- 按钮必须有明确文本或 `aria-label`。
- 弹窗必须支持键盘关闭和焦点管理。
- 禁止只依赖颜色表达状态。
- 错误信息应靠近对应字段或操作区域。

## 29. 枚举与字典规范

业务状态必须集中定义，禁止页面中散落魔法字符串。

常见字典：

```txt
音乐生成状态
视频生成状态
审核状态
任务状态
支付状态
用户状态
权限状态
订单状态
```

推荐：

```ts
export enum MusicGenerateStatus {
  Pending = 'pending',
  Processing = 'processing',
  Success = 'success',
  Failed = 'failed',
}

export const MUSIC_GENERATE_STATUS_LABEL = {
  [MusicGenerateStatus.Pending]: '等待中',
  [MusicGenerateStatus.Processing]: '生成中',
  [MusicGenerateStatus.Success]: '生成成功',
  [MusicGenerateStatus.Failed]: '生成失败',
}
```

禁止：

```tsx
{status === 'success' ? '成功' : '失败'}
```

规则：

- 枚举值、展示文案、颜色、操作权限应集中维护。
- 页面只能消费字典，不直接创造业务状态文案。
- 新增状态必须同步处理列表、详情、筛选、表单、空状态和错误状态。

## 30. 交付报告规范

Agent 每次完成需求后，必须输出交付报告。

推荐格式：

```txt
本次完成：
- 

修改文件：
- 

新增文件：
- 

删除文件：
- 

验证结果：
- pnpm lint
- pnpm typecheck
- pnpm build
- 其他测试

Git 状态：
-

建议提交内容：
-

风险说明：
- 

后续建议：
- 
```

交付报告要求：

- 必须说明本次改动是否影响已有功能。
- 必须说明实际运行过哪些验证命令。
- 必须说明 Git 状态和建议提交内容。
- 未能运行的验证必须说明原因。
- 存在 Mock、临时方案、测试缺口时必须明确写出。
- 不允许只写“已完成”而不说明范围和验证结果。

## 31. 国际化规范

默认中文项目可以不立即接入 i18n，但禁止把大量业务文案散落在复杂逻辑中。

大型项目建议：

- 使用 i18next 或同类方案。
- 菜单、按钮、表单错误、状态文案集中管理。
- 不在业务逻辑中拼接大段用户可见文案。
- 日期、货币、数字、时区格式必须统一封装。
- 面向海外用户的项目必须在早期确定语言切换和文案管理方案。

## 32. 亮色与暗色主题规范

长期维护型项目、SaaS、后台管理系统、用户平台型项目必须具备亮色/暗色主题能力。临时页面、一次性 Demo、低成本内部工具可按项目要求裁剪，但新增样式仍应优先使用语义 Token，避免未来无法扩展主题。

推荐主题模式：

```txt
light：亮色主题
dark：暗色主题
system：跟随系统，可选但推荐
```

推荐结构：

```txt
src/
  app/
    ThemeProvider.tsx
  hooks/
    useTheme.ts
  stores/
    themeStore.ts
  styles/
    tokens.css
    themes.css
  constants/
    theme.ts
```

长期项目实现要求：

- 必须提供全局主题 Provider 或等价机制。
- 必须提供用户可操作的主题切换入口，例如 Header、Settings、UserMenu。
- 主题选择必须持久化，刷新页面后保持用户选择。
- 默认主题可跟随系统，也可以根据产品要求固定为亮色，但必须允许用户切换。
- 切换主题时不能导致页面闪烁、布局跳动或状态丢失。
- 主题状态不应和业务状态混在一起，建议独立 store 或独立 Hook 管理。

Token 要求：

- 文本、背景、边框、分割线、卡片、弹窗、表格、输入框、悬浮层、阴影都必须使用语义 Token。
- 禁止在业务页面中直接写 `#000`、`#fff`、`black`、`white` 作为核心视觉颜色。
- 允许在极少数品牌 Logo、固定素材、图标资源中使用固定颜色，但必须确认暗色背景下仍可识别。
- 图表、状态标签、代码块、空状态插画都必须适配亮色和暗色主题。

示例：

```css
:root {
  --color-bg: #ffffff;
  --color-surface: #f7f8fa;
  --color-text: #1f2329;
  --color-border: #e5e6eb;
}

[data-theme='dark'] {
  --color-bg: #0f1115;
  --color-surface: #171a21;
  --color-text: #f2f3f5;
  --color-border: #2f3440;
}
```

组件要求：

- Button、Input、Select、Modal、Drawer、Table、Card、Tabs、Dropdown、Tooltip、Toast 必须适配亮色和暗色主题。
- hover、active、focus、disabled、selected、error、success、warning 状态必须分别检查。
- 图标颜色应继承文本或使用语义 Token，不要写死颜色。
- 阴影在暗色主题下应更克制，必要时使用边框或背景层级替代。

长期项目验收要求：

- 新页面必须在亮色和暗色主题下都检查。
- 文本与背景必须有足够对比度。
- 表格、表单、弹窗、下拉菜单、通知提示不能出现看不清或边界丢失。
- loading、empty、error、success、permission denied 状态必须同时适配两种主题。
- 交付报告中涉及 UI 改动时，必须说明是否检查了亮色和暗色主题；临时项目裁剪主题能力时也必须说明原因。

## 33. 中文注释规范

相关方法必须添加必要中文注释。注释的目标是解释业务意图、复杂判断和维护风险，不是重复代码表面含义。

必须添加中文注释的场景：

- 业务规则复杂的方法。
- 涉及权限、计费、订单、生成任务、审核状态等关键流程的方法。
- 请求拦截器、错误归一化、重试、去重、缓存失效等基础设施逻辑。
- 状态流转复杂的 Hooks 或 store action。
- 兼容历史数据、兼容旧接口、处理边界条件的逻辑。
- 性能优化、虚拟滚动、懒加载、手动缓存等不容易一眼看懂的实现。

注释要求：

- 注释必须使用中文。
- 优先说明“为什么这样做”和“业务规则是什么”。
- 不要写无意义注释，例如“设置变量”“调用函数”“返回结果”。
- 方法较长时，可在关键逻辑块前添加简短说明。
- 如果修改了已有复杂方法，应同步更新过期注释。

推荐：

```ts
// 生成任务失败后允许用户重试，但已进入审核中的任务不能重复提交，避免后端创建重复任务。
function canRetryGenerateTask(status: GenerateTaskStatus) {
  return status === GenerateTaskStatus.Failed
}
```

禁止：

```ts
// 判断状态
function canRetryGenerateTask(status: GenerateTaskStatus) {
  return status === GenerateTaskStatus.Failed
}
```

## 34. UI 技术栈选择规范

UI 方案必须根据项目类型选择其一，不允许在同一项目中随意混用多套 UI 体系。

推荐选择：

```txt
后台管理系统：Ant Design / Arco Design
高审美 SaaS / 官网 / 创作平台：Tailwind CSS + shadcn/ui
强定制视觉项目：Tailwind CSS + 自定义基础组件
移动优先 H5：Tailwind CSS + 轻量组件封装
```

选择规则：

- 新项目启动时必须明确 UI 方案，并写入 README 或项目规范。
- 已有项目优先沿用现有 UI 体系，不因个人偏好替换。
- 同一项目中不允许混用 Ant Design、Arco Design、Material UI、shadcn/ui 等多套组件体系，除非有明确迁移方案。
- 如果需要从一套 UI 体系迁移到另一套，必须拆成独立迁移计划，不应混在普通业务需求中完成。
- Tailwind 与组件库可以配合使用，但必须明确边界：组件库负责基础交互，Tailwind 负责布局和少量业务样式。

## 35. 组件文件组织规范

简单组件可以使用单文件：

```txt
MusicCard.tsx
```

复杂组件必须拆分目录：

```txt
MusicCreateForm/
  index.tsx
  types.ts
  constants.ts
  hooks.ts
  components/
    LyricsInput.tsx
    StyleSelector.tsx
    GenerateButton.tsx
```

组织规则：

- `index.tsx` 只负责组合、主流程和导出，不堆积所有细节。
- `types.ts` 放组件私有类型。
- `constants.ts` 放选项、默认值、静态配置。
- `hooks.ts` 放组件内部复杂逻辑。
- `components/` 放只服务当前复杂组件的局部子组件。
- 局部子组件不应被外部模块直接引用；需要跨模块复用时再上移到合适目录。
- 单文件组件超过 200 行应评估拆分，超过 300 行必须拆分或说明原因。

## 36. Hooks 规范

Hooks 用于封装可复用逻辑，不用于隐藏混乱的页面流程。

命名规则：

```txt
普通 Hook：useXxx
请求查询 Hook：useMusicListQuery
详情查询 Hook：useMusicDetailQuery
创建变更 Hook：useCreateMusicMutation
删除变更 Hook：useDeleteMusicMutation
业务编排 Hook：useMusicCreateFlow
```

规则：

- Hook 命名必须以 `use` 开头。
- Hook 不应直接渲染 UI。
- Hook 返回值必须稳定、语义清晰。
- 请求类 Hook 优先基于 TanStack Query 封装。
- 表单类 Hook 可以封装默认值、校验、提交编排，但不要隐藏页面全部状态。
- 单个 Hook 过大时应拆分为请求、状态、事件、派生数据等更小 Hook。
- Hook 内部涉及复杂业务规则时必须添加中文注释。

推荐：

```ts
useMusicListQuery()
useMusicDetailQuery(id)
useCreateMusicMutation()
useDeleteMusicMutation()
```

## 37. 文件命名规范

文件命名必须让 Agent 和开发者一眼看出文件职责。

推荐：

```txt
组件文件：PascalCase，例如 MusicScoreCard.tsx
Hook 文件：camelCase，例如 useMusicList.ts
工具文件：camelCase，例如 formatDuration.ts
类型文件：types.ts
接口文件：api.ts
常量文件：constants.ts
样式文件：kebab-case 或跟随组件名
测试文件：跟随被测文件，例如 MusicCard.test.tsx
```

禁止：

```txt
index1.tsx
new.tsx
test.tsx
demo2.tsx
aaa.ts
utils2.ts
temp.tsx
```

规则：

- `index.ts` 和 `index.tsx` 只用于目录导出或复杂组件入口，不作为含糊命名的借口。
- 临时实验文件不能进入正式提交。
- 同一目录下命名风格必须一致。
- 文件名应匹配主要导出的组件、Hook 或函数。

## 38. 不确定信息处理规范

当 Agent 遇到需求不明确时，必须按风险等级处理。

处理规则：

- 优先从现有代码、README、接口类型、路由、相似页面中推断。
- 能安全推断的，按项目现有模式实现，并在交付报告中说明假设。
- 不能安全推断且会影响数据、权限、支付、删除、发布、审核等关键行为时，必须停止并询问用户。
- 不允许凭空创造接口字段、权限 key、业务状态、支付规则、审核规则。
- 临时假设必须集中标记，并在交付报告中说明。
- 如果需要 Mock，必须按 Mock 数据规范实现，不直接写死在页面中。

判断标准：

```txt
低风险：文案、普通布局、非关键展示字段，可按现有模式推断。
中风险：筛选条件、列表字段、交互反馈，应说明假设。
高风险：权限、支付、删除、数据写入、审核流、外部发布，必须确认。
```

## 39. 前端安全规范

前端安全必须作为基础工程能力处理，不能只依赖后端兜底。

规则：

- 不在前端保存明文敏感信息。
- 不把 token、密钥、私有配置写入代码、日志、URL 参数。
- 用户输入内容必须按场景转义、过滤或校验。
- 富文本、Markdown、HTML 渲染必须防范 XSS。
- 下载、跳转、外链必须校验来源。
- 权限判断前端只做展示控制，最终权限以后端为准。
- 错误信息不得暴露数据库、服务器路径、内部堆栈。
- 上传文件必须限制类型、大小和数量，并展示清晰错误。
- 预览图片、音频、视频、脚本、提示词等用户内容时必须考虑恶意内容和异常链接。
- 涉及支付、订单、账号、权限、发布的操作必须有明确确认和错误兜底。
