# Go 后端项目开发规范

本文档用于约束 Agent 在开发 Go 后端项目时的代码组织、命名、分层、注释、校验、错误处理和接口返回规范。Agent 必须优先遵守本文档要求，再结合具体业务需求进行实现。

## 1. 技术栈约定

项目默认使用以下技术栈：

- Web 框架：`Gin`
- 配置管理：`Viper`
- ORM：`Gorm`
- 数据库：默认按项目已有数据库选择；如项目使用 PostgreSQL，必须同时遵守 `.claude/rules/postgresql.md`；新建项目未指定数据库时可选择 PostgreSQL 或 MySQL，但需保持 Repository 层接口稳定
- 参数校验：`go-playground/validator`
- 日志：推荐使用 `zap` 或 `log/slog`
- API 文档：推荐使用 `swaggo/swag`

禁止在无明确需求的情况下引入过重框架或与现有技术栈冲突的库。

## 2. 基本开发原则

### 2.1 分层清晰

项目必须按职责分层，禁止将所有代码写在一个文件中。业务代码必须至少拆分为以下层级：

- `handler`：处理 HTTP 请求、参数绑定、参数校验、调用 service、返回响应
- `service`：处理业务逻辑、事务编排、业务校验
- `repository`：封装数据库访问，不在 handler 中直接操作数据库
- `model`：数据库实体模型
- `request`：接口入参结构体
- `response`：接口出参结构体
- `enum`：业务枚举
- `middleware`：中间件
- `router`：路由注册
- `config`：配置结构体和配置加载
- `pkg` 或 `internal/pkg`：通用工具、响应封装、错误码、分页等

### 2.2 单一职责

每个文件和每个方法只处理一个明确职责。

禁止出现以下情况：

- 一个文件同时包含 handler、service、model、request、response
- 一个方法同时处理参数解析、业务逻辑、数据库操作和响应拼装
- 所有枚举集中写在一个巨大文件中
- 所有实体集中写在一个巨大文件中
- 多个数据库表对应的实体结构体写在同一个文件中
- 所有接口入参或出参集中写在一个巨大文件中

### 2.3 按表和业务实体拆分文件

涉及多个数据库表、多个业务实体或多个管理对象时，必须按表或业务实体拆分文件，不得为了省事把同一模块下的所有代码写进一个总文件。

后端拆分规则：

- `model`：一个主要数据表一个文件，例如 `user.go`、`role.go`、`menu.go`；关系表可按职责放入 `relation.go` 或独立文件。
- `request`：按实体或接口场景拆分，例如 `user_request.go`、`role_request.go`、`menu_request.go`。
- `response`：按实体拆分，例如 `user_response.go`、`role_response.go`、`menu_response.go`。
- `repository`：按数据表或聚合根拆分，例如 `user_repository.go`、`role_repository.go`、`menu_repository.go`；公共事务、软删除、错误映射等可放在 `repository.go`。
- `service`：按业务实体拆分，例如 `user_service.go`、`role_service.go`、`menu_service.go`；公共辅助函数可放在 `service.go`。
- `handler`：按路由资源拆分，例如 `user_handler.go`、`role_handler.go`、`menu_handler.go`；公共绑定、ID 解析、错误响应可放在 `handler.go`。
- `router`：多个资源路由必须拆分到独立路由文件，例如 `admin_router.go`、`user_router.go`。

允许例外：

- 单表 Demo 或极小模块可以合并少量代码，但一旦出现两个以上实体、三个以上接口或文件超过 200 行，必须优先拆分。
- 多个强相关的小关系表可以放在同一个 `relation.go`，但不得把所有主表实体混在一起。

### 2.4 面向维护

代码应优先考虑可读性、可维护性和可测试性。

要求：

- 命名必须表达业务含义
- 复杂逻辑必须拆分为私有方法
- 公共能力必须沉淀为通用组件
- 不允许复制粘贴大段重复逻辑
- 不允许写无意义注释，但实体字段和核心方法必须有中文注释

## 3. 推荐目录结构

### 3.1 目录结构选择规则

Agent 必须先判断项目规模，再选择目录结构。

规则：

- 小型项目、Demo、单模块项目：允许使用横向分层结构
- 中大型项目、多业务域项目、Agent 长期迭代项目：必须优先使用 `internal/module/{业务模块}` 结构
- 涉及用户、订单、任务、文件、支付、AI 模型调用、积分等多个业务域时，必须使用模块化结构
- 一旦项目选择模块化结构，新增业务必须放入对应 module，禁止重新回到全局 `handler/service/repository` 目录
- 同一个项目只能选择一种主要组织方式，禁止同时混用导致结构混乱

### 3.2 小型项目横向分层结构

小型项目可使用以下目录结构：

```text
.
├── cmd
│   └── server
│       └── main.go
├── configs
│   ├── config.yaml
│   ├── config.dev.yaml
│   └── config.prod.yaml
├── internal
│   ├── config
│   │   ├── config.go
│   │   └── loader.go
│   ├── handler
│   │   └── user_handler.go
│   ├── service
│   │   ├── user_service.go
│   │   └── user_service_impl.go
│   ├── repository
│   │   ├── user_repository.go
│   │   └── user_repository_impl.go
│   ├── model
│   │   ├── user.go
│   │   └── role.go
│   ├── enum
│   │   ├── user_status.go
│   │   └── gender.go
│   ├── request
│   │   ├── user_create_request.go
│   │   └── user_update_request.go
│   ├── response
│   │   └── user_response.go
│   ├── router
│   │   ├── router.go
│   │   └── user_router.go
│   ├── middleware
│   │   ├── cors.go
│   │   └── auth.go
│   └── pkg
│       ├── app
│       │   ├── response.go
│       │   ├── error_code.go
│       │   └── pagination.go
│       ├── database
│       │   └── mysql.go
│       ├── logger
│       │   └── logger.go
│       └── validator
│           └── validator.go
├── migrations
├── docs
├── go.mod
└── go.sum
```

### 3.3 中大型项目模块化结构

中大型项目必须优先按业务模块拆分：

```text
internal
└── module
    └── user
        ├── assembler
        ├── handler
        ├── service
        ├── repository
        ├── model
        ├── request
        ├── response
        └── enum
```

示例：

```text
internal
└── module
    ├── user
    ├── order
    ├── task
    ├── file
    ├── payment
    └── ai
```

## 4. 命名规范

### 4.1 包命名

- 包名使用小写单词
- 包名不使用下划线
- 包名应简短并表达职责

正确示例：

```go
package service
package repository
package middleware
```

错误示例：

```go
package user_service
package CommonUtils
```

### 4.2 文件命名

文件名使用小写字母加下划线，格式为：

```text
业务名_职责.go
```

示例：

```text
user_handler.go
user_service.go
user_repository.go
user_create_request.go
user_status.go
```

### 4.3 结构体命名

结构体使用大驼峰命名。

示例：

```go
type User struct {}
type CreateUserRequest struct {}
type UserResponse struct {}
```

### 4.4 接口命名

接口名应表达能力或角色。

示例：

```go
type UserService interface {}
type UserRepository interface {}
```

接口实现结构体使用小驼峰或明确实现名。

示例：

```go
type userService struct {}
type userRepository struct {}
```

### 4.5 方法命名

方法名必须使用动词或动宾短语。

示例：

```go
CreateUser
UpdateUser
DeleteUser
GetUserByID
ListUsers
```

禁止使用无意义命名：

```go
Do
Handle
Process
Data
```

除非上下文非常明确，否则不要使用过短缩写。

## 5. 配置管理规范

### 5.1 配置文件

配置文件统一放在 `configs` 目录。

示例：

```yaml
server:
  port: 8080
  mode: debug

```

### 5.2 Viper 使用要求

必须通过配置结构体承载配置，禁止在业务代码中散落读取配置。

示例：

```go
type Config struct {
	Server ServerConfig `mapstructure:"server"`
	MySQL  MySQLConfig  `mapstructure:"mysql"`
}

type ServerConfig struct {
	Port int    `mapstructure:"port"`
	Mode string `mapstructure:"mode"`
}

type MySQLConfig struct {
	Host         string `mapstructure:"host"`
	Port         int    `mapstructure:"port"`
	Username     string `mapstructure:"username"`
	Password     string `mapstructure:"password"`
	Database     string `mapstructure:"database"`
	MaxIdleConns int    `mapstructure:"max_idle_conns"`
	MaxOpenConns int    `mapstructure:"max_open_conns"`
}
```

要求：

- 启动时完成配置加载
- 配置加载失败必须立即返回错误或终止启动
- 业务代码通过依赖注入使用配置
- 禁止在 handler、service、repository 中直接调用 `viper.GetString`

### 5.3 配置优先级规范

配置读取优先级从高到低：

1. 环境变量
2. 启动参数
3. 环境专属配置文件，例如 `config.prod.yaml`
4. 默认配置文件，例如 `config.yaml`

要求：

- 敏感配置必须优先通过环境变量注入
- 禁止将生产数据库密码、Token、密钥写入仓库
- 配置项必须映射到配置结构体
- 新增配置项必须同步更新中文 `README.md`
- 配置默认值必须安全，不能默认连接生产环境

## 6. Gin 使用规范

### 6.1 路由注册

路由必须集中注册，按模块拆分文件。

示例：

```go
func RegisterUserRoutes(rg *gin.RouterGroup, handler *handler.UserHandler) {
	userGroup := rg.Group("/users")
	{
		userGroup.POST("", handler.CreateUser)
		userGroup.GET("/:id", handler.GetUser)
		userGroup.PUT("/:id", handler.UpdateUser)
		userGroup.DELETE("/:id", handler.DeleteUser)
	}
}
```

### 6.2 Handler 职责

Handler 只允许处理：

- 参数绑定
- 参数校验
- 调用 service
- 统一响应

Handler 禁止处理：

- 复杂业务逻辑
- 直接操作数据库
- 拼接 SQL
- 开启事务

示例：

```go
// CreateUser 创建用户
func (h *UserHandler) CreateUser(c *gin.Context) {
	var req request.CreateUserRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		app.FailWithError(c, app.ErrInvalidParams, err)
		return
	}

	result, err := h.userService.CreateUser(c.Request.Context(), &req)
	if err != nil {
		app.FailWithError(c, app.ErrInternalServer, err)
		return
	}

	app.Success(c, result)
}
```

### 6.3 API 版本规范

对外接口必须带版本前缀。

要求：

- 默认路由前缀使用 `/api/v1`
- 破坏性变更必须升级 API 版本，例如 `/api/v2`
- 禁止直接修改已上线接口行为，除非保持向后兼容
- 内部接口、管理端接口、开放平台接口应使用清晰前缀区分

示例：

```text
/api/v1/users
/api/v1/orders
/api/v1/admin/users
```

### 6.4 Swagger 文档规范

每个 handler 对外接口必须包含 Swagger 注释。

必须包含：

- `Summary`
- `Description`
- `Tags`
- `Accept`
- `Produce`
- `Param`
- `Success`
- `Failure`
- `Router`

示例：

```go
// CreateUser 创建用户
// @Summary 创建用户
// @Description 创建一个新的系统用户
// @Tags 用户管理
// @Accept json
// @Produce json
// @Param request body request.CreateUserRequest true "创建用户参数"
// @Success 200 {object} app.Response{data=response.UserResponse}
// @Failure 400 {object} app.Response
// @Router /api/v1/users [post]
func (h *UserHandler) CreateUser(c *gin.Context) {
	// ...
}
```

## 7. 接口入参规范

### 7.1 入参必须独立定义

每个接口入参必须定义在 `request` 目录中，禁止直接复用数据库实体作为请求参数。

示例：

```go
type CreateUserRequest struct {
	// 用户名
	Username string `json:"username" binding:"required,min=2,max=32"`

	// 手机号
	Phone string `json:"phone" binding:"required,len=11"`

	// 密码
	Password string `json:"password" binding:"required,min=8,max=64"`

	// 性别
	Gender enum.Gender `json:"gender" binding:"required,oneof=1 2 3"`
}
```

### 7.2 参数校验要求

所有接口入参必须加参数校验。

要求：

- 必填字段必须加 `required`
- 字符串长度必须加 `min`、`max` 或 `len`
- 数值范围必须加 `min`、`max`、`gte`、`lte`
- 枚举字段必须加枚举范围校验
- 分页参数必须限制最大页大小

示例：

```go
type PageRequest struct {
	// 页码，从 1 开始
	Page int `form:"page" binding:"required,min=1"`

	// 每页数量，最大 100
	PageSize int `form:"page_size" binding:"required,min=1,max=100"`
}
```

禁止无校验接收外部参数。

### 7.3 查询分页和排序规范

列表接口必须支持分页，除非业务明确说明数据量极小且不需要分页。

要求：

- `page` 从 1 开始
- `page_size` 默认 20，最大 100
- 排序字段必须做白名单校验
- 禁止前端直接传任意 `order by` 字段
- 模糊查询字段必须限制长度
- 时间范围查询必须限制最大跨度，避免大范围扫描

示例：

```go
type UserListRequest struct {
	// 页码，从 1 开始
	Page int `form:"page" binding:"omitempty,min=1"`

	// 每页数量，最大 100
	PageSize int `form:"page_size" binding:"omitempty,min=1,max=100"`

	// 排序字段
	SortBy string `form:"sort_by" binding:"omitempty,oneof=created_at updated_at username"`

	// 排序方向
	SortOrder string `form:"sort_order" binding:"omitempty,oneof=asc desc"`
}
```

## 8. 统一返回规范

所有接口必须使用统一响应结构，禁止在 handler 中直接返回任意 JSON。

### 8.1 响应结构

```go
type Response struct {
	// 业务状态码
	Code int `json:"code"`

	// 响应消息
	Message string `json:"message"`

	// 响应数据
	Data any `json:"data,omitempty"`
}
```

### 8.2 分页响应

```go
type PageResponse struct {
	// 当前页码
	Page int `json:"page"`

	// 每页数量
	PageSize int `json:"page_size"`

	// 总数量
	Total int64 `json:"total"`

	// 列表数据
	List any `json:"list"`
}
```

### 8.3 响应方法

```go
func Success(c *gin.Context, data any) {
	c.JSON(http.StatusOK, Response{
		Code:    0,
		Message: "success",
		Data:    data,
	})
}

func Fail(c *gin.Context, code int, message string) {
	c.JSON(http.StatusOK, Response{
		Code:    code,
		Message: message,
	})
}
```

要求：

- HTTP 状态码和业务状态码分开处理
- HTTP 状态码用于表达协议层结果，业务 `code` 用于表达业务层结果
- 成功请求：HTTP 200 + `code=0`
- 参数错误：HTTP 400
- 未登录：HTTP 401
- 无权限：HTTP 403
- 资源不存在：HTTP 404
- 业务冲突：HTTP 409
- 系统异常：HTTP 500
- 是否允许普通业务失败返回 HTTP 200 + 业务错误码，必须在项目启动阶段确定，全项目禁止混用
- 响应格式必须全项目统一
- 错误码必须集中维护

## 9. 错误码规范

错误码集中定义在 `internal/pkg/app/error_code.go`。

示例：

```go
const (
	// ErrSuccess 请求成功
	ErrSuccess = 0

	// ErrInvalidParams 参数错误
	ErrInvalidParams = 40001

	// ErrUnauthorized 未授权
	ErrUnauthorized = 40101

	// ErrForbidden 无权限
	ErrForbidden = 40301

	// ErrInternalServer 系统内部错误
	ErrInternalServer = 50001
)
```

要求：

- 错误码必须有中文注释
- 错误码命名必须表达业务含义
- 禁止在业务代码中直接写魔法错误码
- 业务错误应封装为可识别的错误类型

### 9.1 错误码区间规范

错误码必须按区间分配，禁止随意编写错误码。

通用区间：

- `0`：成功
- `40000-40999`：通用客户端错误
- `40100-40199`：认证错误
- `40300-40399`：权限错误
- `40400-40499`：资源不存在
- `40900-40999`：冲突、重复提交、幂等冲突
- `50000-50999`：系统内部错误
- `51000-51999`：第三方服务错误
- `60000-69999`：业务模块错误

业务模块错误码建议按模块分配：

- `60100-60199`：用户模块
- `60200-60299`：订单模块
- `60300-60399`：任务模块
- `60400-60499`：文件模块
- `60500-60599`：支付模块
- `60600-60699`：AI 模型调用模块

新增错误码时必须检查是否已有相同含义的错误码，禁止重复定义语义相同的错误码。

### 9.2 错误处理分层规范

错误必须按类型分层，禁止在业务代码中到处直接 `errors.New("xxx")` 后交给 handler 猜测错误含义。

推荐错误类型：

- `AppError`：系统统一错误类型
- `BusinessError`：业务可预期错误，例如手机号已存在
- `ValidationError`：参数校验错误
- `NotFoundError`：资源不存在
- `UnauthorizedError`：未登录
- `ForbiddenError`：无权限
- `ConflictError`：数据冲突或重复提交
- `InternalError`：系统内部错误
- `ThirdPartyError`：第三方服务错误
- `DatabaseError`：数据库错误

要求：

- service 层只能返回 `error`，不直接决定 HTTP 状态码
- handler 层必须通过统一错误转换方法将 `error` 转成响应结构
- 参数校验错误必须统一转换为可读的中文提示
- 数据库原始错误、SQL、堆栈信息禁止直接返回前端
- 业务可预期错误必须使用明确错误码和中文错误消息

示例：

```go
type AppError struct {
	// 业务状态码
	Code int

	// 对外错误消息
	Message string

	// 原始错误
	Cause error
}

func (e *AppError) Error() string {
	return e.Message
}

// NewBusinessError 创建业务错误
func NewBusinessError(code int, message string) *AppError {
	return &AppError{
		Code:    code,
		Message: message,
	}
}
```

## 10. Gorm 使用规范

### 10.1 Model 定义

数据库实体必须放在 `model` 目录，并按数据库表拆分文件。每张数据库表只能对应一个独立的实体文件，禁止把多张表的结构体写在同一个文件中。

文件命名规则：

```text
表名.go
```

示例：

```text
internal/model/user.go
internal/model/user_profile.go
internal/model/order.go
internal/model/order_item.go
```

如果数据库中存在 `users`、`roles`、`user_roles` 三张表，则必须分别定义为：

```text
internal/model/user.go
internal/model/role.go
internal/model/user_role.go
```

禁止定义为：

```text
internal/model/models.go
internal/model/entity.go
internal/model/user_models.go
```

示例：

```go
type User struct {
	// 用户 ID
	ID uint64 `gorm:"primaryKey;column:id" json:"id"`

	// 用户名
	Username string `gorm:"column:username;size:32;not null;comment:用户名" json:"username"`

	// 手机号
	Phone string `gorm:"column:phone;size:11;not null;uniqueIndex;comment:手机号" json:"phone"`

	// 用户状态
	Status enum.UserStatus `gorm:"column:status;not null;comment:用户状态" json:"status"`

	// 创建时间
	CreatedAt time.Time `gorm:"column:created_at;autoCreateTime;comment:创建时间" json:"created_at"`

	// 更新时间
	UpdatedAt time.Time `gorm:"column:updated_at;autoUpdateTime;comment:更新时间" json:"updated_at"`
}

func (User) TableName() string {
	return "users"
}
```

要求：

- 每个数据库表对应的结构体必须单独放在一个文件中
- 一个 model 文件只允许定义一个主实体结构体
- 实体字段必须加中文注释
- 数据库字段必须明确 `column`
- 重要字段必须加 `comment`
- 表名必须通过 `TableName()` 明确指定
- 禁止直接将请求结构体作为数据库实体使用
- 禁止把多个数据库表对应的结构体集中写在 `models.go`、`entity.go`、`tables.go` 等文件中
- 禁止在 model 中编写复杂业务逻辑

### 10.2 数据库基础字段规范

业务表默认建议包含以下基础字段：

- `id`：主键 ID
- `created_at`：创建时间
- `updated_at`：更新时间
- `deleted_at`：删除时间，软删除表使用
- `created_by`：创建人 ID，可按业务决定是否需要
- `updated_by`：更新人 ID，可按业务决定是否需要
- `remark`：备注，可按业务决定是否需要

要求：

- 涉及状态的表必须包含明确状态字段
- 涉及外部系统的表必须保存外部业务 ID、请求流水号或回调流水号
- 涉及金额的字段必须使用整数最小货币单位或定点 decimal，禁止使用 float
- 重要查询条件必须设计索引
- 唯一业务键必须通过唯一索引保障
- 字段是否允许为空必须在 Gorm tag 和迁移 SQL 中保持一致

### 10.3 Repository 层

数据库操作必须封装在 repository 层。

示例：

```go
type UserRepository interface {
	Create(ctx context.Context, user *model.User) error
	FindByID(ctx context.Context, id uint64) (*model.User, error)
	FindByPhone(ctx context.Context, phone string) (*model.User, error)
	WithDB(db *gorm.DB) UserRepository
}

type userRepository struct {
	db *gorm.DB
}

func NewUserRepository(db *gorm.DB) UserRepository {
	return &userRepository{db: db}
}

// WithDB 使用指定数据库连接创建仓储
func (r *userRepository) WithDB(db *gorm.DB) UserRepository {
	return &userRepository{db: db}
}

// Create 创建用户记录
func (r *userRepository) Create(ctx context.Context, user *model.User) error {
	return r.db.WithContext(ctx).Create(user).Error
}
```

要求：

- 所有数据库操作必须使用 `WithContext`
- 查询必须考虑 `record not found`
- 更新必须明确更新字段，禁止无条件全量覆盖
- 删除默认使用软删除，除非业务明确要求物理删除
- 禁止 handler 直接依赖 `*gorm.DB`
- Repository 如需在事务中复用，必须提供 `WithDB(db *gorm.DB)` 方法，确保事务内所有数据库操作使用同一个 `tx`

### 10.4 事务规范

事务必须放在 service 层编排。

示例：

```go
// CreateUser 创建用户并初始化用户资料
func (s *userService) CreateUser(ctx context.Context, req *request.CreateUserRequest) (*response.UserResponse, error) {
	var result *response.UserResponse

	err := s.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
		user := &model.User{
			Username: req.Username,
			Phone:    req.Phone,
			Status:   enum.UserStatusEnabled,
		}

		if err := s.userRepo.WithDB(tx).Create(ctx, user); err != nil {
			return err
		}

		result = response.NewUserResponse(user)
		return nil
	})
	if err != nil {
		return nil, err
	}

	return result, nil
}
```

要求：

- 涉及多个写操作必须使用事务
- 事务内禁止执行外部网络请求
- 事务内逻辑必须尽量短
- 事务失败必须返回明确错误

### 10.5 数据库迁移规范

数据库结构变更必须通过 `migrations` 管理，禁止只修改 Gorm Model 而不提供迁移脚本。

要求：

- 新增表、修改字段、增加索引、调整约束时必须提供迁移文件
- 迁移文件命名格式为 `时间戳_动作_表名.up.sql` 和 `时间戳_动作_表名.down.sql`
- 每个可回滚迁移必须同时提供 `up` 和 `down`
- 如果迁移不可回滚，必须在迁移文件或交付说明中明确说明原因
- 生产环境禁止自动执行 `AutoMigrate`，除非项目明确允许
- Model 字段、索引、表名和迁移 SQL 必须保持一致

示例：

```text
20260507103000_create_users_table.up.sql
20260507103000_create_users_table.down.sql
```

## 11. 枚举规范

枚举必须按业务含义拆分文件，禁止所有枚举写在一个文件中。

示例：

```go
type UserStatus int

const (
	// UserStatusEnabled 启用
	UserStatusEnabled UserStatus = 1

	// UserStatusDisabled 禁用
	UserStatusDisabled UserStatus = 2
)

// IsValid 判断用户状态是否合法
func (s UserStatus) IsValid() bool {
	switch s {
	case UserStatusEnabled, UserStatusDisabled:
		return true
	default:
		return false
	}
}

// String 返回用户状态文本
func (s UserStatus) String() string {
	switch s {
	case UserStatusEnabled:
		return "启用"
	case UserStatusDisabled:
		return "禁用"
	default:
		return "未知"
	}
}
```

要求：

- 枚举类型必须定义明确类型，禁止裸用 `int`
- 枚举值必须加中文注释
- 需要对外展示时必须提供 `String()` 或转换方法
- 需要校验时必须提供 `IsValid()` 方法
- 枚举文件名必须表达枚举业务含义，例如 `user_status.go`

## 12. Service 层规范

Service 层负责业务逻辑，禁止把业务逻辑写在 handler 或 repository 中。

示例：

```go
type UserService interface {
	CreateUser(ctx context.Context, req *request.CreateUserRequest) (*response.UserResponse, error)
}

type userService struct {
	userRepo repository.UserRepository
}

func NewUserService(userRepo repository.UserRepository) UserService {
	return &userService{userRepo: userRepo}
}

// CreateUser 创建用户
// 负责校验手机号唯一性、构造用户实体并保存用户信息。
func (s *userService) CreateUser(ctx context.Context, req *request.CreateUserRequest) (*response.UserResponse, error) {
	existsUser, err := s.userRepo.FindByPhone(ctx, req.Phone)
	if err != nil {
		return nil, err
	}
	if existsUser != nil {
		return nil, app.NewBusinessError(app.ErrInvalidParams, "手机号已存在")
	}

	user := &model.User{
		Username: req.Username,
		Phone:    req.Phone,
		Status:   enum.UserStatusEnabled,
	}

	if err := s.userRepo.Create(ctx, user); err != nil {
		return nil, err
	}

	return response.NewUserResponse(user), nil
}
```

要求：

- 核心业务方法必须有中文注释
- 业务校验必须放在 service 层
- service 不直接依赖 `gin.Context`
- service 方法必须接收 `context.Context`
- service 返回业务响应结构体，不直接操作 HTTP 响应
- 本规范采用应用服务直接返回 Response DTO 的简化风格，适合 Web API 项目快速开发
- 如项目引入领域模型或 DDD 架构，可改为 service 返回领域对象，由 handler、assembler 或 converter 负责 DTO 转换

## 13. Response DTO 规范

接口出参必须定义在 `response` 目录，禁止直接返回数据库实体。

示例：

```go
type UserResponse struct {
	// 用户 ID
	ID uint64 `json:"id"`

	// 用户名
	Username string `json:"username"`

	// 手机号
	Phone string `json:"phone"`

	// 用户状态
	Status enum.UserStatus `json:"status"`

	// 用户状态文本
	StatusText string `json:"status_text"`
}

// NewUserResponse 构造用户响应数据
func NewUserResponse(user *model.User) *UserResponse {
	if user == nil {
		return nil
	}

	return &UserResponse{
		ID:         user.ID,
		Username:   user.Username,
		Phone:       user.Phone,
		Status:     user.Status,
		StatusText: user.Status.String(),
	}
}
```

要求：

- 出参字段必须加中文注释
- 禁止把敏感字段返回给前端，例如密码、盐值、密钥、内部状态
- DTO 转换方法必须集中维护
- 时间字段格式必须全项目统一

### 13.1 DTO 转换规范

简单 DTO 可在 `response` 包中提供 `NewXxxResponse` 方法。

复杂 DTO、跨多个实体组装的响应，必须放入 `assembler` 或 `converter` 包中，禁止在 handler 中拼装复杂响应。

适用场景：

- 用户 + 角色 + 权限
- 订单 + 支付 + 商品
- 任务 + 文件 + 模型调用记录
- AI 生成任务 + 结果资源

推荐目录：

```text
internal/module/user/assembler/user_assembler.go
internal/module/task/assembler/task_assembler.go
```

要求：

- handler 禁止拼装复杂响应
- assembler 只负责数据组装和 DTO 转换，不编写核心业务逻辑
- assembler 不直接操作数据库，所需数据由 service 或 repository 提供
- 复杂响应字段必须有中文注释

## 14. 注释规范

### 14.1 必须注释的内容

以下内容必须有中文注释：

- 数据库实体字段
- 请求参数字段
- 响应字段
- 枚举类型和枚举值
- 核心业务方法
- 对外暴露的公共方法
- 复杂逻辑代码块

### 14.2 注释要求

注释必须说明业务含义，而不是重复代码。

正确示例：

```go
// CreateUser 创建用户
// 校验手机号唯一性，初始化默认状态，并写入用户表。
func (s *userService) CreateUser(ctx context.Context, req *request.CreateUserRequest) (*response.UserResponse, error) {
	// ...
}
```

错误示例：

```go
// 调用 Create 方法
r.Create(ctx, user)
```

## 15. 依赖注入规范

项目必须通过构造函数注入依赖，禁止在业务代码中随意创建全局对象。

示例：

```go
func NewUserHandler(userService service.UserService) *UserHandler {
	return &UserHandler{userService: userService}
}
```

要求：

- handler 依赖 service 接口
- service 依赖 repository 接口
- repository 依赖 `*gorm.DB`
- 配置、日志、数据库连接在启动阶段初始化后注入
- 禁止在 service 中重新初始化数据库连接

## 16. Context 使用规范

项目必须正确传递和使用 `context.Context`，用于请求链路传递、超时控制和取消控制。

要求：

- handler 必须优先使用 `c.Request.Context()` 获取请求上下文
- service、repository 方法必须接收 `context.Context`
- repository 查询必须使用 `WithContext(ctx)`
- 外部 HTTP、RPC、消息队列调用必须设置超时时间
- 禁止使用 `context.Background()` 替代请求上下文，除非是启动任务、后台任务或测试初始化
- 长耗时任务必须支持取消
- 超时错误和取消错误必须转换为统一错误响应

示例：

```go
ctx, cancel := context.WithTimeout(c.Request.Context(), 3*time.Second)
defer cancel()

result, err := h.userService.CreateUser(ctx, &req)
```

## 17. 中间件规范

中间件统一放在 `middleware` 目录。

常见中间件包括：

- 跨域中间件
- 鉴权中间件
- 日志中间件
- 错误恢复中间件
- 请求 ID 中间件

要求：

- 中间件必须职责单一
- 鉴权结果应写入 `gin.Context`
- 中间件失败必须使用统一响应结构返回
- 禁止在中间件中写复杂业务逻辑

## 18. 日志规范

要求：

- 日志组件启动时统一初始化
- 业务代码禁止直接使用 `fmt.Println`
- 错误日志必须包含必要上下文
- 不允许打印密码、Token、密钥等敏感信息
- 请求日志应包含请求 ID、方法、路径、状态码、耗时

示例：

```go
logger.Error("创建用户失败",
	zap.String("phone", mask.Phone(req.Phone)),
	zap.Error(err),
)
```

敏感字段进入日志前必须通过 `mask` 工具脱敏。项目应提供统一脱敏工具，例如 `internal/pkg/mask`，禁止各业务模块重复手写脱敏逻辑。

### 18.1 可观测性规范

要求：

- 每个请求必须生成 `request_id`
- 日志必须包含 `request_id`
- 关键业务操作必须记录结构化日志
- 慢查询、外部请求失败、任务失败必须记录
- 推荐接入 Prometheus、OpenTelemetry 或 Jaeger
- 指标、链路追踪和日志中的字段命名必须保持稳定

## 19. 安全规范

要求：

- 密码必须加密存储，禁止明文保存
- Token、密钥、数据库密码必须通过配置或环境变量读取
- 禁止把敏感配置提交到代码仓库
- SQL 查询必须使用参数绑定，禁止字符串拼接 SQL
- 文件上传必须校验类型、大小和扩展名
- 接口必须做权限校验，除非明确为公开接口
- 返回错误信息时禁止泄露数据库结构、SQL、堆栈等内部细节

### 19.1 鉴权与权限规范

要求：

- 认证信息必须通过中间件解析
- 用户 ID、角色、权限信息应写入 `gin.Context`
- service 层需要进行业务权限校验，不能只依赖前端隐藏按钮
- 管理端接口必须默认需要鉴权
- 公开接口必须显式声明
- 权限校验失败必须使用统一错误响应

### 19.2 数据脱敏规范

以下字段禁止原样打印到日志或返回前端：

- `password`
- `token`
- `secret`
- `authorization`
- `phone`
- `id_card`
- `email`
- `access_key`
- `refresh_token`

要求：

- 手机号、邮箱、身份证号等需要展示时必须脱敏
- 日志中禁止记录完整 Token、密码、密钥
- 第三方接口响应中的敏感字段必须过滤后再记录
- 敏感字段进入日志前必须通过统一 `mask` 工具处理

### 19.3 限流和风控规范

以下高风险接口必须支持限流、验证码或其他风控策略：

- 登录
- 注册
- 短信验证码
- 邮箱验证码
- 文件上传
- 支付回调
- 外部系统回调
- 高频查询接口

限流策略必须可配置，不能硬编码在业务代码中。

## 20. 外部服务调用规范

外部服务调用必须封装在 `client`、`provider` 或 `adapter` 层。

适用场景：

- AI 音乐模型
- 图片模型
- 视频模型
- 对象存储
- 支付接口
- 短信接口
- 邮件接口
- 第三方登录

推荐目录：

```text
internal/pkg/client
internal/module/music/provider
internal/module/ai/adapter
```

要求：

- 禁止 handler 或 repository 直接调用第三方 API
- 外部调用必须设置超时时间
- 外部调用失败必须记录结构化日志
- 外部调用必须区分可重试错误和不可重试错误
- 第三方原始错误禁止直接返回前端
- 第三方请求参数和响应结果如包含敏感信息，必须脱敏后记录
- 外部服务配置必须通过 Viper 配置结构体注入
- 外部服务调用必须支持 mock 或 fake，以便测试 service 层逻辑

## 21. 测试规范

后端项目应按风险分级使用 TDD（Test-Driven Development，测试驱动开发）模式。核心业务和高风险逻辑必须优先 TDD，普通脚手架和低风险 CRUD 可采用“实现后补充必要测试或验证说明”的方式。

核心原则：

```text
核心业务逻辑和高风险逻辑必须遵循 TDD：没有失败测试，不允许编写对应生产代码。
```

核心业务逻辑、复杂查询、参数校验、错误处理、权限控制、金额计算、状态流转、幂等控制、异步任务状态变更等必须先写测试，再写实现。

项目骨架、配置文件、简单 DTO、Swagger 注释、路由注册、`main.go` 启动骨架、数据库连接初始化等非业务逻辑代码可先实现，但完成后必须补充必要验证、启动测试或交付说明。

个人项目、Demo、简单内部工具可以按需求复杂度裁剪 TDD 流程，但 bug 修复、权限、支付、订单、数据一致性、状态机、迁移相关改动仍必须补充回归测试或说明无法测试的原因。

### 21.1 TDD 开发流程

必须严格遵循 Red-Green-Refactor 流程：

1. Red：先编写一个能表达目标行为的失败测试
2. 验证 Red：运行测试，确认测试因目标功能未实现而失败
3. Green：编写最小生产代码，让测试通过
4. 验证 Green：再次运行测试，确认新增测试和相关测试全部通过
5. Refactor：在测试保持通过的前提下整理命名、结构和重复逻辑
6. Repeat：继续为下一个行为编写失败测试

禁止跳过“验证 Red”步骤。测试如果没有先失败，就不能证明它真的覆盖了目标行为。

### 21.2 测试优先范围

以下内容必须先写测试：

- service 层核心业务逻辑
- repository 层复杂查询和数据写入
- handler 层参数绑定、参数校验和响应结果
- 枚举合法性判断
- 错误码和业务错误转换
- 分页、统一响应、参数校验等公共组件
- bug 修复对应的回归测试

### 21.3 测试编写要求

要求：

- service 层核心逻辑必须编写单元测试
- repository 层复杂查询必须编写测试
- 参数校验、枚举校验、错误码转换等公共能力必须编写测试
- 测试文件使用 `_test.go` 后缀
- 测试方法命名格式为 `TestXxx`
- 测试名称必须表达具体业务行为，禁止使用 `TestDo`、`TestHandle`、`TestLogic` 等模糊命名
- 每个测试只验证一个明确行为
- 优先测试对外行为，不要过度测试内部实现细节
- 除非外部依赖不可控，否则不要滥用 mock

推荐结构：

```text
internal
└── service
    ├── user_service.go
    └── user_service_test.go
```

## 22. 幂等性规范

以下接口或流程必须考虑幂等：

- 创建订单
- 支付回调
- 消息消费
- 文件上传
- 重试任务
- 外部系统回调
- 任务创建
- 状态流转

要求：

- 必须使用唯一业务键、幂等键或数据库唯一索引防止重复写入
- 外部回调必须记录请求流水或业务流水
- 重试逻辑不能造成重复扣款、重复创建、重复发放权益
- 幂等冲突必须返回明确业务错误或已有处理结果
- 幂等键的生成规则和有效期必须在代码或 README 中说明

## 23. 异步任务规范

耗时任务禁止长时间阻塞 HTTP 请求，应进入任务表、消息队列或后台任务系统。

要求：

- 长任务必须有任务 ID
- 任务必须有状态，例如 `pending`、`running`、`success`、`failed`、`canceled`
- 任务失败必须记录失败原因
- 任务必须支持重试次数限制
- 消息消费必须保证幂等
- 任务状态流转必须有测试覆盖
- 任务结果查询接口必须支持鉴权和权限校验
- 任务执行日志必须包含 `request_id` 或 `task_id`

适用场景：

- AI 内容生成
- 视频、音频、图片处理
- 大文件导入导出
- 批量任务
- 第三方异步回调

## 24. 状态机规范

涉及订单、任务、审核、支付、生成流程等状态流转时，必须定义明确状态机。

要求：

- 状态必须使用 enum 定义
- 状态流转必须集中维护
- 禁止在业务代码中随意修改状态
- 非法状态流转必须返回业务错误
- 状态流转必须有测试覆盖
- 状态变更必须记录操作人、变更时间和必要上下文
- 涉及异步任务的状态变更必须保证幂等

示例：

```go
// CanTransferTaskStatus 判断任务状态是否允许流转
func CanTransferTaskStatus(from, to enum.TaskStatus) bool {
	switch from {
	case enum.TaskStatusPending:
		return to == enum.TaskStatusRunning || to == enum.TaskStatusCanceled
	case enum.TaskStatusRunning:
		return to == enum.TaskStatusSuccess || to == enum.TaskStatusFailed || to == enum.TaskStatusCanceled
	default:
		return false
	}
}
```

## 25. Agent 开发执行要求

Agent 在生成或修改代码时必须遵守以下流程：

1. 先理解业务需求，再设计目录和文件归属
2. 优先按模块创建文件，禁止把多个职责塞进一个文件
3. 核心业务逻辑和高风险逻辑必须先为目标行为编写失败测试，并运行测试确认失败原因正确
4. 再定义 request、response、enum、model，然后实现 repository、service、handler
5. 每完成一个行为，都必须运行对应测试确认通过
6. 所有接口入参必须加校验标签
7. 所有实体字段、枚举字段、请求字段、响应字段必须加中文注释
8. 核心业务方法必须加中文注释
9. 所有接口必须使用统一响应结构
10. 所有数据库操作必须经过 repository 层
11. 所有业务逻辑必须放在 service 层
12. 开发过程中必须同步更新中文 `README.md`
13. 修改完成后必须运行格式化、静态检查和全量测试
14. 默认不自动提交 Git；只有用户明确授权或项目约定允许自动提交时，才提交本次独立修改，提交信息必须使用中文
15. 如果未获得提交授权，必须在交付说明中给出推荐中文提交信息

## 26. 禁止事项

Agent 禁止执行以下行为：

- 禁止把所有代码写在 `main.go`
- 禁止核心业务逻辑未先编写失败测试就直接编写对应生产代码
- 禁止核心业务测试没有经历失败阶段就声称完成 TDD
- 禁止核心业务功能开发完成后才补测试冒充 TDD
- 禁止 handler 直接操作数据库
- 禁止直接返回 Gorm 实体给前端
- 禁止接口入参不加参数校验
- 禁止实体字段不写中文注释
- 禁止多个数据库表对应的结构体写在同一个文件中
- 禁止枚举全部堆在一个文件
- 禁止错误码散落在业务代码中
- 禁止在业务代码中使用魔法数字和魔法字符串
- 禁止在 service 中依赖 `gin.Context`
- 禁止在 repository 中编写业务逻辑
- 禁止 handler 拼装复杂跨实体响应
- 禁止 handler 或 repository 直接调用第三方 API
- 禁止业务代码绕过状态机随意修改状态
- 禁止敏感字段未脱敏直接写入日志
- 禁止在代码中硬编码数据库密码、Token、密钥
- 禁止未格式化代码直接提交
- 禁止功能、配置、启动方式、接口或目录结构已变化但不更新 `README.md`
- 禁止使用英文 `README.md` 作为项目主说明文档
- 禁止未经用户授权或项目约定擅自执行 Git 提交
- 禁止在正式提交时使用英文提交信息
- 禁止把多个不相关修改混在同一次提交中

## 27. 推荐开发顺序

新增一个业务模块时，推荐按以下顺序开发：

1. 拆分业务行为，明确第一个可测试目标
2. 为第一个核心业务目标行为编写失败测试
3. 运行对应测试，确认测试失败且失败原因正确
4. 定义枚举：`internal/enum/xxx.go`
5. 定义实体：`internal/model/xxx.go`
6. 定义请求结构：`internal/request/xxx_request.go`
7. 定义响应结构：`internal/response/xxx_response.go`
8. 定义 repository 接口和实现
9. 定义 service 接口和实现
10. 定义 handler
11. 注册 router
12. 增加参数校验和错误码
13. 运行对应测试，确认测试通过
14. 重构代码并保持测试通过
15. 重复 TDD 流程直到模块完成
16. 同步更新中文 `README.md`
17. 运行 `gofmt`
18. 运行 `go test ./...`
19. 如用户或项目约定授权提交，使用 Git 提交本次代码修改，提交信息必须为中文
20. 如未提交，在交付说明中写明原因和推荐提交信息

## 28. 本地开发热更新规范

后端项目本地开发推荐使用 `Air` 进行热更新，提高开发和 Agent 调试效率。已有项目应优先沿用现有开发命令，例如 `make dev`、`go run`、Docker Compose、Taskfile 或其他热更新工具。

要求：

- 新建 Go 后端项目可默认提供 `.air.toml`
- `Air` 只用于本地开发环境，禁止作为生产启动方式
- 如项目使用 `.air.toml`，必须监听 Go 源码、配置文件和必要模板文件
- 如项目使用 `.air.toml`，必须排除 `.git`、`tmp`、`vendor`、`docs`、`migrations` 等不需要触发重启的目录
- 如项目使用 `.air.toml`，热更新构建产物必须输出到临时目录，例如 `tmp`
- README 必须提供实际本地开发命令；如使用 Air，需说明 Air 安装方式和启动命令
- 如果项目使用 Docker Compose，本地开发服务应优先支持通过 Air 启动

推荐安装命令：

```bash
go install github.com/air-verse/air@latest
```

推荐启动命令：

```bash
air
```

推荐 `.air.toml` 基础配置：

```toml
root = "."
tmp_dir = "tmp"

[build]
cmd = "go build -o ./tmp/server ./cmd/server"
bin = "./tmp/server"
include_ext = ["go", "yaml", "yml", "toml"]
exclude_dir = ["tmp", "vendor", ".git", "docs", "migrations"]
delay = 1000
stop_on_error = true

[log]
time = true
```

Agent 新建后端项目时，推荐同时创建 `.air.toml` 并在中文 `README.md` 中说明热更新启动方式；如选择其他本地开发方式，必须在 README 中写清楚启动命令。

## 29. 代码质量检查

每次完成开发后必须执行：

```bash
gofmt -w .
go test ./...
```

如项目引入 `golangci-lint`，还必须执行：

```bash
golangci-lint run
```

检查不通过时，必须修复后再交付。

## 30. README 维护规范

后端项目开发过程中必须同步维护项目根目录下的 `README.md` 文件。

要求：

- `README.md` 必须使用中文编写
- 新增、修改或删除功能时，必须同步更新功能说明
- 修改配置项时，必须同步更新配置说明
- 修改启动方式、构建方式、测试方式时，必须同步更新运行说明
- 修改 Air 热更新配置或本地开发启动方式时，必须同步更新运行说明
- 新增接口或调整接口行为时，必须同步更新接口说明或 API 文档入口
- 修改目录结构时，必须同步更新项目结构说明
- 新增中间件、定时任务、消息队列、外部服务依赖时，必须同步更新依赖说明
- README 内容必须与实际代码保持一致，禁止写过期说明

推荐 `README.md` 至少包含以下内容：

```text
# 项目名称

## 项目简介

## 技术栈

## 目录结构

## 环境要求

## 配置说明

## 启动方式

## 本地热更新

## 测试方式

## 接口说明

## 开发规范

## 常见问题
```

Agent 每次交付代码时，必须检查本次变更是否影响 `README.md`。如果有影响，必须同步更新；如果无影响，交付说明中可以说明“本次变更不涉及 README 更新”。

## 31. Git 提交规范

默认不自动提交 Git。只有用户明确要求提交，或项目文档明确授权自动提交时，Agent 才能在完成一个独立修改后提交。

要求：

- 未获得提交授权时，不得执行 Git 提交，只在交付说明中给出建议提交信息
- 获得提交授权后，每次提交只包含一个明确功能、bug 修复、重构、文档或配置调整
- 提交前必须先完成格式化、测试和 README 同步检查
- 提交信息必须使用中文
- 提交信息必须说明本次修改的业务目的或技术目的
- 一次提交只包含一个明确主题，禁止混入多个不相关修改
- 如果本次修改只更新规范、文档或配置，提交信息也必须明确说明
- 如果未提交，必须在交付说明中说明原因，并给出推荐中文提交信息

推荐提交信息格式：

```text
类型：简要说明本次修改
```

常用类型：

```text
新增：新增用户管理接口
修复：修复用户分页查询条件错误
优化：优化订单列表查询性能
重构：拆分用户服务层业务逻辑
文档：更新项目启动说明
配置：调整数据库连接配置
测试：补充用户创建业务测试
```

禁止提交信息示例：

```text
update
fix
test
wip
修改
提交代码
```

## 32. 最小模块示例

一个标准用户模块至少应包含：

```text
internal
├── handler
│   └── user_handler.go
├── service
│   ├── user_service.go
│   └── user_service_impl.go
├── repository
│   ├── user_repository.go
│   └── user_repository_impl.go
├── model
│   └── user.go
├── enum
│   └── user_status.go
├── request
│   ├── user_create_request.go
│   └── user_update_request.go
└── response
    └── user_response.go
```

如果模块变复杂，应继续按职责拆分，不能通过扩大单个文件来承载复杂度。

## 33. 交付标准

Agent 交付代码时必须满足：

- 项目可以正常启动
- 核心业务和高风险变更遵循 TDD，并有对应测试记录
- 核心业务新增测试已确认经历过失败阶段，然后通过；如因项目阶段裁剪，必须说明原因和补测建议
- 代码已按职责拆分
- 命名清晰统一
- Gin 路由注册清晰
- Viper 配置集中管理
- Gorm 实体按数据库表拆分为独立文件，且字段注释完整
- 枚举按文件拆分并带校验方法
- 请求参数包含校验规则
- 响应结构全局统一
- 错误码集中维护
- 核心业务方法有中文注释
- 中文 `README.md` 已根据本次变更同步更新
- 本地开发命令已在 README 中说明；如使用 Air，已提供或更新 `.air.toml`
- 已执行格式化和测试
- 如用户或项目授权提交，已使用中文提交信息提交本次独立修改
- 如未提交，已在交付说明中明确原因并给出推荐中文提交信息

Agent 交付说明必须包含：

- 本次完成内容
- 涉及文件列表
- 新增文件、修改文件、删除文件
- 测试执行结果
- README 更新情况
- Git 提交情况
- 未完成或未验证事项

如果因环境、依赖或外部服务导致无法完成验证，必须在交付说明中明确说明原因和未验证项。
