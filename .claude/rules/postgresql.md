# PostgreSQL 数据库设计规范

本文档用于指导开发 Agent 在通用业务系统中设计 PostgreSQL 数据库表结构。目标是让 Agent 产出的表结构具备一致命名、清晰约束、可维护迁移、良好查询性能和可扩展的业务建模能力。

适用项目：SaaS、管理后台、CRM、ERP、内容平台、AI 应用后台、内部运营系统等通用业务系统。

## 1. 设计总原则

Agent 设计数据库时必须遵守以下原则：

1. 先建模业务实体和关系，再写 SQL。
2. 所有表、字段、约束、索引都必须命名清晰、可读、可追踪。
3. 优先使用数据库约束保证数据正确性，不把所有校验都放到应用层。
4. 每张业务表必须考虑查询场景、唯一性、生命周期、审计和权限边界。
5. 默认使用 PostgreSQL 原生能力：`timestamptz`、`jsonb`、`check`、`foreign key`、`partial index`、`generated identity`。
6. 不为“可能会用到”的查询提前创建大量索引。索引必须服务于明确的查询条件、排序或关联。
7. 数据库迁移必须可重复执行、可回滚或至少可安全补偿。
8. 禁止使用含糊字段名，例如 `data`、`info`、`type`、`flag`，除非有明确上下文和约束。

## 2. 命名规范

### 2.1 通用命名

- 使用小写英文和下划线：`snake_case`。
- 禁止使用大写、空格、拼音、保留字。
- 表名使用复数名词：`users`、`orders`、`departments`。
- 关联表使用两个实体名组合：`user_roles`、`order_items`。
- 字段名使用单数语义：`user_id`、`order_id`、`created_at`。
- 布尔字段使用明确前缀：`is_active`、`has_paid`、`can_login`。
- 时间字段统一使用 `_at` 后缀：`created_at`、`paid_at`、`deleted_at`。
- 日期字段使用 `_date` 后缀：`birth_date`、`settlement_date`。
- 金额字段使用 `_amount` 后缀，并配套币种字段：`total_amount`、`currency_code`。

### 2.2 约束命名

统一显式命名约束，便于排查和迁移：

```sql
-- 主键
<table>_pkey

-- 外键
<table>_<column>_fkey

-- 唯一约束
<table>_<column_or_columns>_key

-- 检查约束
<table>_<business_rule>_check
```

示例：

```sql
constraint users_email_key unique (email),
constraint orders_user_id_fkey foreign key (user_id) references users(id),
constraint orders_total_amount_check check (total_amount >= 0)
```

### 2.3 索引命名

索引命名格式：

```sql
idx_<table>_<columns>[_condition]
```

示例：

```sql
create index idx_orders_user_id on orders (user_id);
create index idx_orders_status_created_at on orders (status, created_at desc);
create index idx_users_email_active on users (email) where deleted_at is null;
```

## 3. Schema 组织

默认规则：

- 通用业务表放在 `public` schema，除非项目已有 schema 分层约定。
- 审计、日志、归档等可独立到 `audit`、`log`、`archive` schema。
- 不同业务模块不要随意创建 schema。只有当权限、生命周期或访问边界明显不同时才拆分。
- 所有对象引用建议显式带 schema，尤其在迁移脚本中：`public.users`。

## 4. 扩展规范

迁移脚本中如果使用 PostgreSQL 扩展，必须显式声明。

常用扩展：

```sql
create extension if not exists pgcrypto;
```

说明：

- `gen_random_uuid()` 依赖 `pgcrypto`。
- UUIDv7 依赖具体扩展实现，使用前必须确认目标数据库环境支持。
- 不要在业务 SQL 中隐式假设扩展已安装。

## 5. 主键规范

默认选择：

```sql
id bigint generated always as identity primary key
```

适用场景：

- 单体应用、普通后台、绝大多数业务系统使用 `bigint identity`。
- 需要对外暴露不可预测 ID 时，增加 `public_id` 字段，不要直接暴露自增主键。
- 分布式写入、跨库合并、离线生成 ID 时，可使用 UUIDv7、ULID 或业务统一 ID 方案。
- 避免在大表主键中使用随机 UUIDv4，因为随机写入会导致索引碎片和缓存命中下降。

推荐模式：

```sql
create table public.users (
  id bigint generated always as identity,
  public_id uuid not null default gen_random_uuid(),
  constraint users_pkey primary key (id),
  constraint users_public_id_key unique (public_id)
);
```

## 6. 基础字段规范

每张核心业务表默认包含：

```sql
id bigint generated always as identity,
created_at timestamptz not null default now(),
updated_at timestamptz not null default now(),
created_by bigint,
updated_by bigint,
deleted_at timestamptz,
deleted_by bigint,
version integer not null default 1
```

字段说明：

- `id`：内部主键。
- `created_at`：创建时间，必须为 `timestamptz`。
- `updated_at`：更新时间，由应用层或触发器维护。
- `created_by` / `updated_by`：操作人 ID，后台系统建议保留。
- `deleted_at` / `deleted_by`：软删除字段。需要永久删除的日志表、临时表可不使用。
- `version`：乐观锁版本号，适合订单、配置、审批流等存在并发修改风险的表。

`updated_at` 可以由应用层维护，也可以使用数据库触发器维护。若使用触发器，必须在项目中统一实现，不要每张表写一套不一致的函数。

不建议所有表都无脑添加软删除。以下表通常可以硬删除或按归档策略处理：

- 临时验证码、短期 token。
- 幂等记录。
- 任务队列中间表。
- 可重建缓存表。
- 高吞吐日志表。

## 7. 数据类型规范

### 7.1 常用类型选择

| 场景         | 推荐类型                              | 禁用或慎用                    |
| ------------ | ------------------------------------- | ----------------------------- |
| 主键         | `bigint generated always as identity` | `serial`、`int`               |
| 字符串       | `text`                                | 无必要的 `varchar(255)`       |
| 时间点       | `timestamptz`                         | `timestamp without time zone` |
| 日期         | `date`                                | 字符串日期                    |
| 金额         | `numeric(18,2)` 或按业务精度调整      | `float`、`double precision`   |
| 布尔         | `boolean`                             | `'Y'/'N'`、`0/1` 字符串       |
| JSON         | `jsonb`                               | `json`、字符串 JSON           |
| 状态         | `text` + `check`                      | 无约束字符串                  |
| IP 地址      | `inet`                                | `text`                        |
| 邮箱、手机号 | `text`                                | 过度依赖长度限制              |

### 7.2 字符串

默认使用 `text`。只有业务上确实需要长度限制时才使用 `varchar(n)` 或 `check`。

推荐：

```sql
email text not null,
constraint users_email_format_check check (position('@' in email) > 1)
```

### 7.3 金额

金额必须使用 `numeric`，并明确币种。

```sql
total_amount numeric(18,2) not null default 0,
currency_code char(3) not null default 'CNY',
constraint orders_total_amount_check check (total_amount >= 0)
```

不允许使用浮点数存储金额。

### 7.4 状态字段

状态字段优先使用 `text` + `check`，便于迁移和阅读。

```sql
status text not null default 'pending',
constraint orders_status_check check (
  status in ('pending', 'paid', 'cancelled', 'refunded')
)
```

只有状态集合稳定、跨表复用明显时才考虑 PostgreSQL enum。

### 7.5 JSONB 字段

`jsonb` 适合存储扩展属性、第三方回调原文、低频查询配置。

禁止把核心业务字段长期藏在 `jsonb` 中。以下字段应拆成独立列：

- 会被频繁过滤、排序、关联的字段。
- 需要唯一约束、外键约束的字段。
- 财务、权限、状态流转相关字段。
- 报表统计常用字段。

如果需要查询 `jsonb` 内容，必须设计对应 GIN 索引或表达式索引。

```sql
create index idx_products_attributes_gin
on public.products using gin (attributes);
```

## 8. 约束规范

### 8.1 Not Null

只要业务上必填，就必须设置 `not null`。不要只依赖应用层校验。

### 8.2 Unique

唯一性必须由数据库保证。

```sql
constraint users_email_key unique (email)
```

软删除表中，如果唯一值允许删除后复用，使用部分唯一索引：

```sql
create unique index idx_users_email_active_unique
on public.users (email)
where deleted_at is null;
```

### 8.3 Check

适合金额、数量、状态、比例、时间范围等基础规则。

```sql
constraint order_items_quantity_check check (quantity > 0),
constraint coupons_discount_rate_check check (discount_rate > 0 and discount_rate <= 1)
```

### 8.4 Foreign Key

外键默认应创建，除非存在明确的高吞吐、跨库、异步一致性理由。

外键删除策略必须明确：

- `restrict`：默认推荐，防止误删父数据。
- `cascade`：仅用于强从属关系，例如订单明细随订单删除。
- `set null`：保留历史记录，但关联对象可删除。
- `set default`：少用，必须有明确默认对象。

示例：

```sql
constraint orders_user_id_fkey
  foreign key (user_id)
  references public.users(id)
  on delete restrict
```

PostgreSQL 不会自动为外键列创建索引。所有外键列默认必须创建索引。

```sql
create index idx_orders_user_id on public.orders (user_id);
```

## 9. 索引设计规范

### 9.1 必须创建索引的场景

- 外键列。
- 高频 `where` 过滤字段。
- 高频 `join` 字段。
- 高频排序字段，尤其是分页排序字段。
- 唯一性约束字段。
- 软删除表的活跃数据查询条件。
- 多租户系统中的 `tenant_id` 组合查询。

### 9.2 复合索引顺序

复合索引字段顺序：

1. 等值过滤字段。
2. 范围过滤字段。
3. 排序字段。

示例：

```sql
create index idx_orders_tenant_status_created_at
on public.orders (tenant_id, status, created_at desc)
where deleted_at is null;
```

适配查询：

```sql
select *
from public.orders
where tenant_id = $1
  and status = 'paid'
  and deleted_at is null
order by created_at desc
limit 20;
```

### 9.3 部分索引

软删除和固定过滤条件优先使用部分索引。

```sql
create index idx_orders_active_user_id
on public.orders (user_id)
where deleted_at is null;
```

### 9.4 索引类型选择

- B-tree：默认索引，适合等值、范围、排序。
- GIN：适合 `jsonb`、数组、全文搜索。
- GiST：适合地理、范围、近邻查询。
- BRIN：适合超大时间序列表，如日志、事件。
- Hash：仅等值查询，一般不优先使用。

### 9.5 禁止的索引习惯

- 不要给每个字段都建索引。
- 不要创建与唯一约束重复的普通索引。
- 不要创建没有查询场景支撑的索引。
- 不要忽略索引维护成本。写多读少的表要谨慎加索引。

## 10. 多租户规范

通用业务系统如果存在组织、团队、企业、项目空间等隔离边界，应使用 `tenant_id`。

推荐基础表：

```sql
create table public.tenants (
  id bigint generated always as identity,
  name text not null,
  status text not null default 'active',
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  deleted_at timestamptz,
  constraint tenants_pkey primary key (id),
  constraint tenants_status_check check (status in ('active', 'disabled'))
);
```

租户下业务表：

```sql
tenant_id bigint not null,
constraint <table>_tenant_id_fkey
  foreign key (tenant_id)
  references public.tenants(id)
  on delete restrict
```

多租户表的唯一约束通常必须带 `tenant_id`：

```sql
create unique index idx_projects_tenant_name_active_unique
on public.projects (tenant_id, name)
where deleted_at is null;
```

多租户查询索引通常以 `tenant_id` 开头：

```sql
create index idx_projects_tenant_status_created_at
on public.projects (tenant_id, status, created_at desc)
where deleted_at is null;
```

如项目使用 Row Level Security，应在多租户表启用 RLS：

```sql
alter table public.projects enable row level security;
alter table public.projects force row level security;

create policy projects_tenant_policy on public.projects
  for all
  using (tenant_id = current_setting('app.current_tenant_id')::bigint);
```

## 11. 用户、角色、权限建模建议

通用权限模型建议使用 RBAC：

- `users`：用户。
- `tenants`：租户、组织或企业。
- `tenant_members`：用户和租户的成员关系。
- `roles`：角色。
- `permissions`：权限点。
- `role_permissions`：角色和权限关系。
- `member_roles`：成员和角色关系。

核心原则：

- 用户身份和租户成员身份分开建模。
- 权限不要直接挂在用户上，除非是临时授权或特殊覆盖。
- 权限编码使用稳定字符串，例如 `order.read`、`order.refund`。
- 角色可以是租户级，也可以是系统级，但必须有明确字段区分。

## 12. 审计与日志规范

业务表的审计字段记录最后状态：

```sql
created_at, created_by, updated_at, updated_by, deleted_at, deleted_by
```

关键业务操作应额外记录审计日志：

```sql
create table audit.audit_logs (
  id bigint generated always as identity,
  tenant_id bigint,
  actor_id bigint,
  action text not null,
  resource_type text not null,
  resource_id text not null,
  before_data jsonb,
  after_data jsonb,
  ip_address inet,
  user_agent text,
  created_at timestamptz not null default now(),
  constraint audit_logs_pkey primary key (id)
);
```

审计日志原则：

- 只追加，不更新，不软删除。
- 记录谁在什么时候对什么资源做了什么。
- 重要变更记录前后快照。
- 高增长日志表需要按时间分区或归档。

## 13. 软删除规范

默认软删除字段：

```sql
deleted_at timestamptz,
deleted_by bigint
```

查询业务数据时必须默认过滤：

```sql
where deleted_at is null
```

软删除表的唯一约束应使用部分唯一索引：

```sql
create unique index idx_users_email_active_unique
on public.users (email)
where deleted_at is null;
```

禁止使用 `is_deleted boolean` 作为唯一软删除标记。`deleted_at` 同时表达是否删除和删除时间，信息更完整。

## 14. 时间规范

- 所有时间点使用 `timestamptz`。
- 数据库存储 UTC 语义，由应用层负责展示时区。
- 使用 `now()` 作为创建时间默认值。
- 不使用字符串保存时间。
- 不使用本地时区含义不清的 `timestamp without time zone`。

常用字段：

```sql
created_at timestamptz not null default now(),
updated_at timestamptz not null default now(),
published_at timestamptz,
expired_at timestamptz
```

## 15. 迁移脚本规范

迁移脚本必须满足：

- 文件名包含顺序和语义，例如 `20260507103000_create_orders.sql`。
- 一个迁移只做一类变更，避免巨大混合迁移。
- DDL 使用显式 schema。
- 创建表、约束、索引分段清晰。
- 大表加字段时优先考虑锁表风险。
- 创建大表索引时优先使用 `create index concurrently`，但注意它不能放在事务块中。
- 不使用 PostgreSQL 不支持的语法，例如 `add constraint if not exists`。

安全添加约束示例：

```sql
do $$
begin
  if not exists (
    select 1
    from pg_constraint
    where conname = 'orders_total_amount_check'
      and conrelid = 'public.orders'::regclass
  ) then
    alter table public.orders
      add constraint orders_total_amount_check
      check (total_amount >= 0);
  end if;
end $$;
```

## 16. Agent 输出要求

Agent 在为项目设计 PostgreSQL 表时，必须输出以下内容：

1. 业务实体说明：每张表解决什么业务问题。
2. 表关系说明：一对一、一对多、多对多关系。
3. 完整 DDL：包含表、主键、外键、唯一约束、检查约束。
4. 索引设计：说明每个索引服务的查询场景。
5. 字段说明：字段含义、类型选择、是否必填、默认值。
6. 数据生命周期：是否软删除、是否归档、是否需要审计。
7. 多租户策略：是否需要 `tenant_id`、唯一约束是否带租户维度。
8. 权限边界：是否需要 RLS 或应用层权限过滤。
9. 常用查询示例：至少列出核心列表页、详情页、唯一性检查、关联查询。
10. 风险说明：可能的数据增长、锁表风险、索引成本、迁移注意事项。

Agent 不允许只输出一段 `create table` 就结束。

## 17. 建表 SQL 模板

以下模板中的 `<table_name>`、`<column>` 等尖括号内容是占位符。Agent 生成最终 SQL 时必须替换为真实名称，不能把占位符原样输出到迁移脚本。

```sql
create table public.<table_name> (
  id bigint generated always as identity,
  tenant_id bigint,

  -- business fields
  name text not null,
  status text not null default 'active',
  metadata jsonb not null default '{}'::jsonb,

  -- audit fields
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  created_by bigint,
  updated_by bigint,
  deleted_at timestamptz,
  deleted_by bigint,
  version integer not null default 1,

  constraint <table_name>_pkey primary key (id),
  constraint <table_name>_tenant_id_fkey
    foreign key (tenant_id)
    references public.tenants(id)
    on delete restrict,
  constraint <table_name>_status_check
    check (status in ('active', 'disabled'))
);

create index idx_<table_name>_tenant_id
on public.<table_name> (tenant_id);

create index idx_<table_name>_tenant_status_created_at
on public.<table_name> (tenant_id, status, created_at desc)
where deleted_at is null;
```

## 18. 通用业务表示例

### 18.1 用户表

```sql
create table public.users (
  id bigint generated always as identity,
  public_id uuid not null default gen_random_uuid(),
  email text not null,
  phone text,
  display_name text not null,
  avatar_url text,
  status text not null default 'active',
  last_login_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  deleted_at timestamptz,
  version integer not null default 1,

  constraint users_pkey primary key (id),
  constraint users_public_id_key unique (public_id),
  constraint users_status_check check (status in ('active', 'disabled', 'locked'))
);

create unique index idx_users_email_active_unique
on public.users (email)
where deleted_at is null;

create index idx_users_status_created_at
on public.users (status, created_at desc)
where deleted_at is null;
```

### 18.2 订单表

```sql
create table public.orders (
  id bigint generated always as identity,
  tenant_id bigint not null,
  user_id bigint not null,
  order_no text not null,
  status text not null default 'pending',
  total_amount numeric(18,2) not null default 0,
  currency_code char(3) not null default 'CNY',
  paid_at timestamptz,
  cancelled_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  deleted_at timestamptz,
  version integer not null default 1,

  constraint orders_pkey primary key (id),
  constraint orders_tenant_id_fkey foreign key (tenant_id) references public.tenants(id) on delete restrict,
  constraint orders_user_id_fkey foreign key (user_id) references public.users(id) on delete restrict,
  constraint orders_status_check check (status in ('pending', 'paid', 'cancelled', 'refunded')),
  constraint orders_total_amount_check check (total_amount >= 0)
);

create unique index idx_orders_tenant_order_no_active_unique
on public.orders (tenant_id, order_no)
where deleted_at is null;

create index idx_orders_tenant_status_created_at
on public.orders (tenant_id, status, created_at desc)
where deleted_at is null;

create index idx_orders_user_id
on public.orders (user_id);
```

## 19. 表设计自检清单

Agent 交付表设计前必须逐项检查：

- 表名和字段名是否符合 `snake_case`。
- 每张表是否有明确业务职责。
- 主键是否使用合适策略。
- 必填字段是否设置 `not null`。
- 状态、金额、数量等字段是否有 `check` 约束。
- 唯一性是否由数据库保证。
- 软删除表的唯一索引是否使用 `where deleted_at is null`。
- 外键列是否都创建了索引。
- 复合索引字段顺序是否匹配查询条件。
- 多租户表的唯一约束和索引是否包含 `tenant_id`。
- 是否避免了无查询场景的冗余索引。
- 金额是否使用 `numeric`，时间是否使用 `timestamptz`。
- 核心业务字段是否没有被藏进 `jsonb`。
- 迁移脚本是否显式命名约束和索引。
- 是否说明了常用查询和索引对应关系。

## 20. Agent 默认决策

当需求没有明确说明时，Agent 按以下默认规则设计：

- 主键使用 `bigint generated always as identity`。
- 字符串使用 `text`。
- 时间使用 `timestamptz`。
- 金额使用 `numeric(18,2)`。
- 状态使用 `text` + `check`。
- 业务表包含 `created_at`、`updated_at`。
- 核心业务表保留软删除字段。
- 外键默认启用，并为外键列创建索引。
- 多租户系统中所有租户级业务表添加 `tenant_id`。
- 唯一约束在多租户系统中默认带 `tenant_id`。
- 列表页查询必须配套合适索引。
- 审计日志只追加，不更新，不软删除。

## 21. 禁止事项

- 禁止用 `float` 或 `double precision` 存金额。
- 禁止用字符串存日期、时间、JSON。
- 禁止核心业务表没有主键。
- 禁止只在应用层保证唯一性。
- 禁止外键列没有索引。
- 禁止滥用 `jsonb` 替代正常字段建模。
- 禁止无约束的状态字段。
- 禁止用 `is_deleted` 替代 `deleted_at` 作为唯一删除标记。
- 禁止创建没有查询依据的索引。
- 禁止迁移脚本依赖手工执行顺序但不写清楚。