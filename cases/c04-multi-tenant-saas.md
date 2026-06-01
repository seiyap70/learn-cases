# C04: 多租户SaaS的数据隔离

## 业务场景

某 B2B SaaS 平台，为 5000 家企业客户提供项目管理+HR+财务一体化服务。每家客户的数据必须与其他客户完全隔离——A 公司绝不能看到 B 公司的员工数据或财务数据。

**已知数据：**
- 租户数（企业客户）：5000 家
- 总用户数：50 万（平均每租户 100 人，大租户 5000 人）
- 数据量：每租户平均 10GB，总计 50TB
- 最大租户：10 万用户，1TB 数据
- 最小租户：5 用户，100MB 数据
- SaaS 版本：所有租户使用同一版本（无定制部署）

**为什么这是难题？**

SaaS 的核心矛盾：**多租户共享基础设施以降低成本，但租户间数据必须严格隔离。** 共享越多成本越低，但隔离越难保证。

三种隔离策略各有致命缺陷：
- 独立数据库：隔离性最好，但 5000 个数据库的运维成本不可承受
- 共享数据库+共享表：成本最低，但一条 SQL 写错就泄露数据
- 共享数据库+独立 Schema：折中，但 Schema 迁移是噩梦

## 核心挑战

### 挑战 1：隔离性保证

**最严重的风险：数据泄露。** 如果某条查询忘记加 `tenant_id` 过滤条件，A 公司就能看到 B 公司的数据。这不是理论风险——2019 年某 SaaS 平台就因为一个 API 缺少租户过滤，导致 1000+ 家客户数据泄露。

### 挑战 2：租户规模差异巨大

最大租户（1TB）和最小租户（100MB）的数据量差 10000 倍。共享表模式下，大租户的热数据和小租户的冷数据混在一起，查询性能不可预测。

### 挑战 3：Schema 演进

SaaS 平台每周发版，每次可能修改表结构。5000 个租户的 Schema 如何同步升级？如果升级失败如何回滚？

### 挑战 4：成本控制

独立数据库方案的运维成本：
- 5000 个 MySQL 实例 × 每月 ¥500 = ¥250 万/月
- 共享数据库方案：3 个 MySQL 实例 × 每月 ¥5000 = ¥1.5 万/月
- 成本差 160 倍

## 三种隔离层级的完整数据库设计

### Tier 3（共享表 + tenant_id）完整 DDL

```sql
-- ============================================================
-- Tier 3：共享数据库，共享表，通过 tenant_id 行级隔离
-- 适用：小租户（4450 家，数据量 < 500MB）
-- ============================================================

-- 公共路由表：记录每个租户所属的 Tier 和路由信息
CREATE TABLE tenant_registry (
    tenant_id      INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    tenant_name    VARCHAR(255) NOT NULL UNIQUE,
    tier           VARCHAR(10) NOT NULL CHECK (tier IN ('tier1', 'tier2', 'tier3')),
    db_endpoint    VARCHAR(255),          -- Tier 1 独有
    schema_name    VARCHAR(64),           -- Tier 2 独有
    status         VARCHAR(20) NOT NULL DEFAULT 'active'
                   CHECK (status IN ('active', 'migrating', 'suspended', 'provisioning')),
    compliance_tags VARCHAR(255)[],       -- 合规标签：finance / healthcare / gdpr
    created_at     TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at     TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_tenant_registry_tier ON tenant_registry(tier);
CREATE INDEX idx_tenant_registry_status ON tenant_registry(status);

-- 项目表（核心业务表）
CREATE TABLE projects (
    id            BIGINT NOT NULL GENERATED ALWAYS AS IDENTITY,
    tenant_id     INTEGER NOT NULL,
    name          VARCHAR(255) NOT NULL,
    description   TEXT,
    status        VARCHAR(20) NOT NULL DEFAULT 'active'
                  CHECK (status IN ('active', 'archived', 'deleted')),
    owner_id      BIGINT NOT NULL,
    budget        DECIMAL(15, 2),
    start_date    DATE,
    end_date      DATE,
    custom_fields JSONB DEFAULT '{}',     -- 租户自定义扩展字段
    created_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    PRIMARY KEY (tenant_id, id)           -- 复合主键：tenant_id 在前，支持分区裁剪
);

-- 员工表
CREATE TABLE employees (
    id            BIGINT NOT NULL GENERATED ALWAYS AS IDENCY,
    tenant_id     INTEGER NOT NULL,
    user_id       BIGINT NOT NULL,
    employee_no   VARCHAR(64) NOT NULL,
    name          VARCHAR(255) NOT NULL,
    email         VARCHAR(255) NOT NULL,
    department    VARCHAR(255),
    hire_date     DATE NOT NULL,
    status        VARCHAR(20) NOT NULL DEFAULT 'active'
                  CHECK (status IN ('active', 'on_leave', 'resigned')),
    custom_fields JSONB DEFAULT '{}',
    created_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    PRIMARY KEY (tenant_id, id),
    UNIQUE (tenant_id, employee_no),
    UNIQUE (tenant_id, email)
);

-- 薪资记录表（敏感数据）
CREATE TABLE salary_records (
    id            BIGINT NOT NULL GENERATED ALWAYS AS IDENTITY,
    tenant_id     INTEGER NOT NULL,
    employee_id   BIGINT NOT NULL,
    month         DATE NOT NULL,
    base_salary   DECIMAL(15, 2) NOT NULL,
    bonus         DECIMAL(15, 2) DEFAULT 0,
    deduction     DECIMAL(15, 2) DEFAULT 0,
    net_salary    DECIMAL(15, 2) GENERATED ALWAYS AS (base_salary + bonus - deduction) STORED,
    created_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    PRIMARY KEY (tenant_id, id),
    UNIQUE (tenant_id, employee_id, month),
    FOREIGN KEY (tenant_id, employee_id) REFERENCES employees(tenant_id, id)
);

-- ============ 必须的索引（所有表都以 tenant_id 作为前缀列）============
CREATE INDEX idx_projects_tenant_status ON projects(tenant_id, status);
CREATE INDEX idx_projects_tenant_owner  ON projects(tenant_id, owner_id);
CREATE INDEX idx_employees_tenant_dept  ON employees(tenant_id, department);
CREATE INDEX idx_salary_tenant_month    ON salary_records(tenant_id, month);

-- ============ 启用 RLS（Row Level Security）============
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE employees ENABLE ROW LEVEL SECURITY;
ALTER TABLE salary_records ENABLE ROW LEVEL SECURITY;

-- RLS 策略：自动按 current_setting('app.current_tenant') 过滤
CREATE POLICY tenant_isolation ON projects
    USING (tenant_id = current_setting('app.current_tenant')::INTEGER);
CREATE POLICY tenant_isolation ON employees
    USING (tenant_id = current_setting('app.current_tenant')::INTEGER);
CREATE POLICY tenant_isolation ON salary_records
    USING (tenant_id = current_setting('app.current_tenant')::INTEGER);

-- 禁止绕过 RLS 的保障：撤销应用用户的 SUPERUSER 属性
-- 应用数据库用户必须为普通用户（非 superuser、非表 owner），RLS 才能生效
-- REVOKE SUPERUSER FROM app_user;
```

### Tier 2（共享 DB 独立 Schema）完整 DDL

```sql
-- ============================================================
-- Tier 2：共享数据库实例，每个租户一个独立 Schema
-- 适用：中等租户（500 家，数据量 500MB ~ 5GB）
-- 每个实例承载约 100 个 Schema
-- ============================================================

-- 以租户 2001 为例，其他租户结构相同
CREATE SCHEMA tenant_2001;

-- 在 tenant_2001 下创建完整表结构（无需 tenant_id 列）
CREATE TABLE tenant_2001.projects (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name          VARCHAR(255) NOT NULL,
    description   TEXT,
    status        VARCHAR(20) NOT NULL DEFAULT 'active'
                  CHECK (status IN ('active', 'archived', 'deleted')),
    owner_id      BIGINT NOT NULL,
    budget        DECIMAL(15, 2),
    start_date    DATE,
    end_date      DATE,
    custom_fields JSONB DEFAULT '{}',
    created_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
CREATE INDEX idx_t2001_projects_status ON tenant_2001.projects(status);
CREATE INDEX idx_t2001_projects_owner  ON tenant_2001.projects(owner_id);

CREATE TABLE tenant_2001.employees (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id       BIGINT NOT NULL,
    employee_no   VARCHAR(64) NOT NULL UNIQUE,
    name          VARCHAR(255) NOT NULL,
    email         VARCHAR(255) NOT NULL UNIQUE,
    department    VARCHAR(255),
    hire_date     DATE NOT NULL,
    status        VARCHAR(20) NOT NULL DEFAULT 'active',
    custom_fields JSONB DEFAULT '{}',
    created_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE tenant_2001.salary_records (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    employee_id   BIGINT NOT NULL REFERENCES tenant_2001.employees(id),
    month         DATE NOT NULL,
    base_salary   DECIMAL(15, 2) NOT NULL,
    bonus         DECIMAL(15, 2) DEFAULT 0,
    deduction     DECIMAL(15, 2) DEFAULT 0,
    net_salary    DECIMAL(15, 2) GENERATED ALWAYS AS (base_salary + bonus - deduction) STORED,
    created_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE (employee_id, month)
);

-- Schema 级权限控制：每个租户只能访问自己的 Schema
CREATE ROLE tenant_2001_user;
GRANT USAGE ON SCHEMA tenant_2001 TO tenant_2001_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA tenant_2001 TO tenant_2001_user;
-- 禁止访问其他 Schema
REVOKE USAGE ON SCHEMA tenant_2002 FROM tenant_2001_user;

-- Schema 版本管理表（每个 Schema 内部记录自己的版本号）
CREATE TABLE tenant_2001.schema_migrations (
    version     INTEGER PRIMARY KEY,
    description VARCHAR(255),
    applied_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

### Tier 1（独立数据库）完整 DDL

```sql
-- ============================================================
-- Tier 1：独立数据库实例
-- 适用：大租户/合规租户（50 家，数据量 > 5GB 或有物理隔离合规要求）
-- 每个租户拥有独立 MySQL/PostgreSQL 实例
-- ============================================================

-- 在独立实例上创建完整的数据库（以租户 100 为例）
-- 与 Tier 2 类似，无需 tenant_id 列
CREATE DATABASE tenant_100_db;

\c tenant_100_db;

CREATE TABLE projects (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name          VARCHAR(255) NOT NULL,
    description   TEXT,
    status        VARCHAR(20) NOT NULL DEFAULT 'active',
    owner_id      BIGINT NOT NULL,
    budget        DECIMAL(15, 2),
    start_date    DATE,
    end_date      DATE,
    custom_fields JSONB DEFAULT '{}',
    created_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE employees (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id       BIGINT NOT NULL,
    employee_no   VARCHAR(64) NOT NULL UNIQUE,
    name          VARCHAR(255) NOT NULL,
    email         VARCHAR(255) NOT NULL UNIQUE,
    department    VARCHAR(255),
    hire_date     DATE NOT NULL,
    status        VARCHAR(20) NOT NULL DEFAULT 'active',
    custom_fields JSONB DEFAULT '{}',
    created_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE salary_records (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    employee_id   BIGINT NOT NULL REFERENCES employees(id),
    month         DATE NOT NULL,
    base_salary   DECIMAL(15, 2) NOT NULL,
    bonus         DECIMAL(15, 2) DEFAULT 0,
    deduction     DECIMAL(15, 2) DEFAULT 0,
    net_salary    DECIMAL(15, 2) GENERATED ALWAYS AS (base_salary + bonus - deduction) STORED,
    created_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE (employee_id, month)
);

-- 独立实例可针对大租户数据特征创建定制索引
CREATE INDEX idx_projects_status_owner ON projects(status, owner_id);
CREATE INDEX idx_employees_department  ON employees(department);
CREATE INDEX idx_salary_month          ON salary_records(month);

-- 审计日志表（合规租户必须）
CREATE TABLE access_audit_log (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id     BIGINT NOT NULL,
    action      VARCHAR(50) NOT NULL,    -- SELECT / INSERT / UPDATE / DELETE
    table_name  VARCHAR(64) NOT NULL,
    record_id   BIGINT,
    query_text  TEXT,
    client_ip   INET,
    accessed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
CREATE INDEX idx_audit_user_time ON access_audit_log(user_id, accessed_at);
CREATE INDEX idx_audit_table_time ON access_audit_log(table_name, accessed_at);
```

### 三种隔离层级 DDL 差异总结

| 设计要素 | Tier 1（独立DB） | Tier 2（独立Schema） | Tier 3（共享表） |
|---------|----------------|-------------------|----------------|
| tenant_id 列 | 不需要 | 不需要 | 必须，且为复合主键前缀列 |
| RLS 策略 | 不需要 | 不需要（靠 Schema 权限） | 必须，数据库强制隔离 |
| 连接方式 | 直连独立实例 | SET search_path 切换 | 单连接池 + SET 变量 |
| 索引策略 | 可按租户数据特征定制 | 可按 Schema 独立定制 | 必须以 tenant_id 为前缀列 |
| 权限控制 | 实例级隔离 | Schema 级角色隔离 | RLS 行级隔离 |
| 备份粒度 | 实例级（最细） | Schema 级（pg_dump -n） | 表级（需按 tenant_id 过滤） |

## 设计约束

- 数据隔离等级：强隔离（任一租户的数据不可被其他租户访问，包括系统管理员）
- 合规要求：部分租户（金融/医疗）要求数据物理隔离
- 运维人力：2 人管理所有数据库
- Schema 升级零停机

## 请先独立思考（限时 35 分钟）

1. 三种隔离策略（独立DB / 共享DB独立Schema / 共享表）的具体优缺点，用表格对比。
2. 如果选共享表方案，如何从**数据库层面**保证 `tenant_id` 过滤不可绕过？（不是应用层检查，而是数据库强制）
3. 大租户（1TB）和小租户（100MB）混在同一个表中，查询性能如何保证？
4. 5000 个租户的 Schema 升级如何做到零停机？

---

## 设计解析

### 三种隔离策略对比

| 维度 | 独立数据库 | 共享DB独立Schema | 共享表+tenant_id |
|------|-----------|-----------------|----------------|
| 隔离性 | 物理隔离（最强） | 逻辑隔离（Schema级） | 逻辑隔离（行级） |
| 成本 | 极高（5000个实例） | 中（5000个Schema） | 低（1个数据库） |
| 运维复杂度 | 极高 | 高（Schema迁移） | 低（单库迁移） |
| 查询性能 | 可独立优化 | 可按Schema优化 | 大租户可能影响小租户 |
| 连接管理 | 需要5000个连接池 | 按Schema切换 | 单连接池 |
| 扩展性 | 可按租户独立扩展 | 中等 | 受单库上限限制 |
| 数据泄露风险 | 无 | 低（Schema权限控制） | 中（依赖应用层） |
| 备份恢复 | 按租户独立 | 按Schema独立 | 全库备份（含所有租户） |

### 选择策略：分层隔离

**核心思路：不是所有租户都用同一种策略，而是根据租户规模和合规要求分层。**

```
Tier 1（大租户/合规租户）：独立数据库
  → 租户数：约 50 家（1%）
  → 数据量：约 10TB（20%）
  → 理由：数据量大需要独立优化 + 合规要求物理隔离

Tier 2（中等租户）：共享DB独立Schema
  → 租户数：约 500 家（10%）
  → 数据量：约 30TB（60%）
  → 理由：Schema级隔离够用 + 可独立备份

Tier 3（小租户）：共享表 + tenant_id
  → 租户数：约 4450 家（89%）
  → 数据量：约 10TB（20%）
  → 理由：成本最低 + 小租户数据量小性能影响小
```

**成本计算：**
- Tier 1：50 个 MySQL 实例 × ¥5000/月 = ¥25 万/月
- Tier 2：5 个 MySQL 实例（每实例承载 100 个 Schema）× ¥5000/月 = ¥2.5 万/月
- Tier 3：3 个 MySQL 实例 × ¥5000/月 = ¥1.5 万/月
- **总计：约 ¥29 万/月**（vs 纯独立DB的 ¥250 万/月，节省 88%）

### Tier 3（共享表）的隔离性保证：数据库行级安全

**问题：** 应用层加 `tenant_id` 过滤不可靠——开发者可能忘记加，或者某条 SQL 拼接错误。

**解决方案：PostgreSQL Row Level Security (RLS)**

```sql
-- 1. 启用 RLS
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE employees ENABLE ROW LEVEL SECURITY;
ALTER TABLE salary_records ENABLE ROW LEVEL SECURITY;

-- 2. 创建 RLS 策略：每个租户只能看到自己的数据
CREATE POLICY tenant_isolation ON projects
    USING (tenant_id = current_setting('app.current_tenant')::INTEGER);

CREATE POLICY tenant_isolation ON employees
    USING (tenant_id = current_setting('app.current_tenant')::INTEGER);

-- 3. 应用层设置当前租户上下文
-- 每次请求开始时：
SET app.current_tenant = '12345';

-- 此后该连接的所有查询自动加上 tenant_id = 12345 的过滤
-- 即使开发者写了 SELECT * FROM employees，也只返回租户 12345 的数据
```

**RLS 的关键优势：**
- **不可绕过**：即使应用层 SQL 没有 `WHERE tenant_id = ?`，RLS 也会自动添加
- **数据库层面保证**：不依赖应用层代码质量
- **对应用透明**：开发者写正常的 SQL，RLS 自动处理

**RLS 的性能影响：**
- 每条查询额外加一个 `tenant_id = ?` 的过滤条件
- 如果 `tenant_id` 上有索引（必须有），性能影响 < 5%
- `current_setting()` 是 PostgreSQL 内置函数，极快

**必须的索引：**
```sql
-- 每个表都必须有 tenant_id 索引（复合索引的第一个字段）
CREATE INDEX idx_projects_tenant ON projects(tenant_id, id);
CREATE INDEX idx_employees_tenant ON employees(tenant_id, id);
CREATE INDEX idx_salary_tenant ON salary_records(tenant_id, employee_id);
```

### 应用层的租户上下文管理

#### 租户上下文中间件（完整实现）

```python
import jwt
import logging
from contextvars import ContextVar
from dataclasses import dataclass
from typing import Optional

logger = logging.getLogger(__name__)

# 使用 ContextVar 实现协程安全的租户上下文
current_tenant: ContextVar[Optional['TenantContext']] = ContextVar('current_tenant', default=None)

@dataclass
class TenantContext:
    tenant_id: int
    tier: str               # tier1 / tier2 / tier3
    schema_name: Optional[str] = None
    db_endpoint: Optional[str] = None
    compliance_tags: list = None


class TenantMiddleware:
    """
    每个请求设置租户上下文。
    核心职责：
    1. 从 JWT 提取并校验租户身份
    2. 查询路由表确定租户隔离层级
    3. 在数据库连接上设置租户上下文（RLS 变量 / search_path / 连接路由）
    4. 请求结束后清理，防止连接池泄露
    """

    def __init__(self, db_pool, jwt_secret, routing_repo):
        self.db_pool = db_pool
        self.jwt_secret = jwt_secret
        self.routing_repo = routing_repo

    def process_request(self, request):
        # 1. 从 JWT Token 中提取租户 ID
        token = request.headers.get("Authorization", "").replace("Bearer ", "")
        if not token:
            raise UnauthorizedError("Missing authentication token")

        try:
            payload = jwt.decode(token, self.jwt_secret, algorithms=["RS256"])
        except jwt.InvalidTokenError as e:
            raise UnauthorizedError(f"Invalid token: {e}")

        tenant_id = payload.get("tenant_id")
        user_id = payload.get("user_id")
        if not tenant_id or not user_id:
            raise UnauthorizedError("Token missing tenant_id or user_id")

        # 2. 验证用户确实属于该租户（防止伪造 tenant_id）
        user = self.routing_repo.get_user(user_id)
        if user is None or user.tenant_id != tenant_id:
            raise ForbiddenError("User does not belong to claimed tenant")

        # 3. 查询路由表，确定租户隔离层级
        route = self.routing_repo.get_tenant_route(tenant_id)
        if route is None:
            raise NotFoundError(f"Tenant {tenant_id} not found in routing table")
        if route.status == 'suspended':
            raise ForbiddenError("Tenant account is suspended")
        if route.status == 'migrating':
            # 迁移期间只读
            request.tenant_readonly = True

        # 4. 构建租户上下文
        ctx = TenantContext(
            tenant_id=tenant_id,
            tier=route.tier,
            schema_name=route.schema_name,
            db_endpoint=route.db_endpoint,
            compliance_tags=route.compliance_tags or []
        )
        current_tenant.set(ctx)
        request.tenant_ctx = ctx

        # 5. 根据隔离层级设置数据库连接
        self._setup_db_connection(ctx)

    def _setup_db_connection(self, ctx: TenantContext):
        """根据租户层级，在数据库连接上设置对应的上下文"""
        if ctx.tier == 'tier3':
            # 共享表模式：设置 RLS 变量
            conn = self.db_pool.get_shared_connection()
            conn.execute(f"SET app.current_tenant = '{ctx.tenant_id}'")
            conn.execute("SET app.tenant_tier = 'tier3'")

        elif ctx.tier == 'tier2':
            # Schema 模式：切换 search_path
            conn = self.db_pool.get_shared_connection()
            conn.execute(f"SET search_path = {ctx.schema_name}, public")
            conn.execute(f"SET app.current_tenant = '{ctx.tenant_id}'")

        elif ctx.tier == 'tier1':
            # 独立数据库模式：从专用连接池获取连接
            # 路由层已根据 db_endpoint 分配到对应的连接池
            conn = self.db_pool.get_dedicated_connection(ctx.db_endpoint)
            # 即使是独立 DB，也设置 tenant_id 用于审计
            conn.execute(f"SET app.current_tenant = '{ctx.tenant_id}'")

    def process_response(self, request, response):
        """请求结束后重置租户上下文，防止连接池泄露"""
        ctx = current_tenant.get(None)
        if ctx is None:
            return

        try:
            if ctx.tier == 'tier3':
                conn = self.db_pool.get_shared_connection()
                conn.execute("RESET app.current_tenant")
                conn.execute("RESET app.tenant_tier")
            elif ctx.tier == 'tier2':
                conn = self.db_pool.get_shared_connection()
                conn.execute("RESET search_path")
                conn.execute("RESET app.current_tenant")
            elif ctx.tier == 'tier1':
                conn = self.db_pool.get_dedicated_connection(ctx.db_endpoint)
                conn.execute("RESET app.current_tenant")
        except Exception as e:
            logger.error(f"Failed to reset tenant context: {e}")
            # 标记该连接为脏连接，连接池回收时应强制关闭
            self.db_pool.mark_connection_dirty()
        finally:
            current_tenant.set(None)
```

#### 数据库连接路由层

```python
class TenantAwareDBRouter:
    """
    数据库连接路由层。
    根据租户的 Tier 级别，将请求路由到正确的数据库连接。
    管理多套连接池：shared_pool（Tier 3/2）+ 多个 dedicated_pool（Tier 1）。
    """

    def __init__(self, config, routing_repo):
        self.routing_repo = routing_repo
        self.shared_pool = self._create_shared_pool(config)
        self.dedicated_pools = {}  # {db_endpoint: connection_pool}
        self._pool_lock = threading.Lock()

    def get_connection(self, tenant_id: int):
        """
        获取租户对应的数据库连接。
        调用方无需关心底层路由细节。
        """
        route = self.routing_repo.get_tenant_route(tenant_id)

        if route.tier == 'tier1':
            return self._get_dedicated_connection(route.db_endpoint)
        else:
            # Tier 2 和 Tier 3 共用同一个数据库实例，只是 Schema/RLS 策略不同
            return self.shared_pool.getconn()

    def _get_dedicated_connection(self, db_endpoint: str):
        """获取 Tier 1 租户的专用数据库连接，按需创建连接池"""
        if db_endpoint not in self.dedicated_pools:
            with self._pool_lock:
                if db_endpoint not in self.dedicated_pools:  # double-check
                    self.dedicated_pools[db_endpoint] = self._create_dedicated_pool(
                        db_endpoint
                    )
        return self.dedicated_pools[db_endpoint].getconn()

    def _create_shared_pool(self, config):
        return psycopg2.pool.ThreadedConnectionPool(
            minconn=config.shared_pool_min,
            maxconn=config.shared_pool_max,
            host=config.shared_db_host,
            port=config.shared_db_port,
            dbname=config.shared_db_name,
            user=config.db_user,
            password=config.db_password,
            # 关键：连接归还连接池时自动执行 RESET ALL
            options='-c default_transaction_read_only=off',
        )

    def _create_dedicated_pool(self, db_endpoint: str):
        return psycopg2.pool.ThreadedConnectionPool(
            minconn=2,
            maxconn=20,
            dsn=db_endpoint,
        )

    def return_connection(self, tenant_id: int, conn):
        """归还连接到对应的连接池"""
        route = self.routing_repo.get_tenant_route(tenant_id)
        if route.tier == 'tier1':
            self.dedicated_pools[route.db_endpoint].putconn(conn)
        else:
            self.shared_pool.putconn(conn)


class ConnectionPoolGuard:
    """
    连接池守护：确保归还连接时租户上下文已重置。
    防止因代码缺陷导致租户上下文泄露。
    """

    def __init__(self, db_router: TenantAwareDBRouter):
        self.db_router = db_router

    def execute_in_tenant_context(self, tenant_id: int, callback, readonly=False):
        """在租户上下文中执行数据库操作，自动管理连接的获取和归还"""
        conn = self.db_router.get_connection(tenant_id)
        try:
            if readonly:
                conn.set_session(readonly=True)
            result = callback(conn)
            conn.commit()
            return result
        except Exception:
            conn.rollback()
            raise
        finally:
            # 关键：归还前重置所有会话变量
            try:
                cursor = conn.cursor()
                cursor.execute("RESET ALL")  # 重置所有 SET 变量
                cursor.close()
            except Exception:
                pass  # RESET 失败则标记连接为脏，等待连接池清理
            self.db_router.return_connection(tenant_id, conn)
```

**为什么请求结束后要 RESET？**

数据库连接是复用的（连接池）。如果请求 A 设置了 `app.current_tenant = 12345`，请求 B 复用同一个连接时，如果 B 没有设置租户上下文，就会以租户 12345 的身份执行查询——数据泄露！

三重防线：
1. 中间件 `process_response` 中显式 RESET
2. `ConnectionPoolGuard` 归还连接前执行 `RESET ALL`
3. 连接池配置 `reset_on_return`（如果连接池支持）

### Tier 2（独立Schema）的实现

```sql
-- 每个租户一个 Schema
CREATE SCHEMA tenant_1001;
CREATE SCHEMA tenant_1002;

-- 每个 Schema 下有完整的表结构
CREATE TABLE tenant_1001.projects (...);
CREATE TABLE tenant_1001.employees (...);

CREATE TABLE tenant_1002.projects (...);
CREATE TABLE tenant_1002.employees (...);
```

**连接切换：**

```python
class SchemaTenantRouter:
    def get_connection(self, tenant_id):
        # 切换到租户的 Schema
        conn = self.db.get_connection()
        conn.execute(f"SET search_path = tenant_{tenant_id}, public")
        return conn
```

#### Schema 生命周期管理（完整实现）

```python
import hashlib
import logging
from datetime import datetime
from concurrent.futures import ThreadPoolExecutor, as_completed

logger = logging.getLogger(__name__)


class SchemaLifecycleManager:
    """
    Schema 生命周期管理器。
    负责：创建、迁移、销毁租户 Schema。
    关键约束：
    - 迁移必须幂等（同一 SQL 多次执行结果一致）
    - 迁移必须可回滚（每个版本有对应的回滚 SQL）
    - 应用代码必须兼容新旧两个版本的 Schema（双版本兼容期）
    """

    def __init__(self, db_pool, migration_repo, lock_manager):
        self.db_pool = db_pool
        self.migration_repo = migration_repo     # 迁移版本仓库
        self.lock_manager = lock_manager          # 分布式锁（防止并发迁移）

    # ============ Schema 创建 ============
    def provision_schema(self, tenant_id: int, template: str = 'default'):
        """
        为新租户创建完整 Schema。
        template 参数支持不同业务模板（标准版/财务增强版等）。
        """
        schema_name = f"tenant_{tenant_id}"

        # 分布式锁：防止同一租户被并发创建
        lock_key = f"schema_provision:{tenant_id}"
        with self.lock_manager.acquire(lock_key, timeout=60):
            conn = self.db_pool.getconn()
            try:
                conn.autocommit = False

                # 1. 创建 Schema
                conn.execute(f"CREATE SCHEMA IF NOT EXISTS {schema_name}")

                # 2. 创建角色并授权
                role_name = f"tenant_{tenant_id}_user"
                conn.execute(f"CREATE ROLE {role_name} NOLOGIN")
                conn.execute(f"GRANT USAGE ON SCHEMA {schema_name} TO {role_name}")

                # 3. 按 DDL 模板创建所有表
                ddl_scripts = self.migration_repo.get_template_ddl(template)
                for ddl in ddl_scripts:
                    # 替换模板中的 schema 占位符
                    ddl = ddl.replace('{schema}', schema_name)
                    conn.execute(ddl)

                # 4. 授权
                conn.execute(
                    f"GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES "
                    f"IN SCHEMA {schema_name} TO {role_name}"
                )
                conn.execute(
                    f"GRANT USAGE, SELECT ON ALL SEQUENCES "
                    f"IN SCHEMA {schema_name} TO {role_name}"
                )

                # 5. 初始化 schema_migrations 版本表
                conn.execute(f"""
                    CREATE TABLE {schema_name}.schema_migrations (
                        version     INTEGER PRIMARY KEY,
                        description VARCHAR(255),
                        checksum    VARCHAR(64),
                        applied_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
                    )
                """)
                conn.execute(f"""
                    INSERT INTO {schema_name}.schema_migrations (version, description, checksum)
                    VALUES ({self.migration_repo.get_latest_version()}, 'initial', 'init')
                """)

                conn.commit()
                logger.info(f"Schema provisioned: {schema_name}")
            except Exception as e:
                conn.rollback()
                logger.error(f"Schema provisioning failed for {schema_name}: {e}")
                # 清理已创建的半成品 Schema
                self._cleanup_failed_schema(conn, schema_name)
                raise
            finally:
                self.db_pool.putconn(conn)

    # ============ Schema 批量迁移 ============
    def migrate_all_schemas(self, version: int, migration_sql: str, rollback_sql: str):
        """
        批量迁移所有 Tier 2 Schema 到指定版本。
        核心约束：
        - 迁移 SQL 必须幂等（如 CREATE INDEX IF NOT EXISTS）
        - 失败的 Schema 不阻塞其他 Schema
        - 记录每个 Schema 的迁移状态，支持后续重试
        """
        # 获取所有需要迁移的 Schema
        schemas = self.migration_repo.get_schemas_below_version(version)
        if not schemas:
            logger.info("All schemas already at target version")
            return

        logger.info(f"Migrating {len(schemas)} schemas to version {version}")

        # 记录迁移批次
        batch_id = self.migration_repo.create_migration_batch(
            version=version,
            total=len(schemas)
        )

        # 并行迁移，每批 50 个（控制数据库并发压力）
        failed_schemas = []
        migrated_count = 0

        for batch in self._chunk(schemas, size=50):
            with ThreadPoolExecutor(max_workers=50) as executor:
                futures = {
                    executor.submit(
                        self._migrate_single_schema,
                        schema, version, migration_sql, rollback_sql, batch_id
                    ): schema
                    for schema in batch
                }

                for future in as_completed(futures):
                    schema = futures[future]
                    try:
                        future.result()
                        migrated_count += 1
                    except Exception as e:
                        failed_schemas.append((schema, str(e)))
                        logger.error(f"Migration failed for {schema}: {e}")

        logger.info(
            f"Migration batch {batch_id} complete: "
            f"{migrated_count} succeeded, {len(failed_schemas)} failed"
        )

        # 对失败的 Schema 记录到重试队列
        if failed_schemas:
            self.migration_repo.record_failures(batch_id, failed_schemas)
            # 触发告警
            self._alert_migration_failures(batch_id, failed_schemas)

    def _migrate_single_schema(self, schema_name: str, version: int,
                                migration_sql: str, rollback_sql: str,
                                batch_id: int):
        """迁移单个 Schema，带分布式锁和版本检查"""
        # 分布式锁：防止同一 Schema 被并发迁移
        lock_key = f"schema_migration:{schema_name}"
        with self.lock_manager.acquire(lock_key, timeout=120):
            conn = self.db_pool.getconn()
            try:
                conn.autocommit = False

                # 幂等检查：是否已迁移到此版本
                current = self._get_schema_version(conn, schema_name)
                if current >= version:
                    logger.info(f"{schema_name} already at version {current}, skipping")
                    return

                # 校验迁移 SQL 的 checksum（防止 SQL 被篡改）
                checksum = hashlib.sha256(migration_sql.encode()).hexdigest()[:16]

                # 执行迁移
                actual_sql = migration_sql.replace('{schema}', schema_name)
                conn.execute(actual_sql)

                # 更新版本记录
                conn.execute(f"""
                    INSERT INTO {schema_name}.schema_migrations (version, description, checksum)
                    VALUES ({version}, 'batch_{batch_id}', '{checksum}')
                """)

                conn.commit()
            except Exception as e:
                conn.rollback()
                # 执行回滚（如果回滚也失败，标记为需人工介入）
                try:
                    rollback_sql_applied = rollback_sql.replace('{schema}', schema_name)
                    conn.execute(rollback_sql_applied)
                    conn.commit()
                except Exception as rollback_err:
                    logger.critical(
                        f"Rollback also failed for {schema_name}: {rollback_err}. "
                        f"Manual intervention required!"
                    )
                    self.migration_repo.mark_schema_broken(schema_name, str(rollback_err))
                raise
            finally:
                self.db_pool.putconn(conn)

    def _get_schema_version(self, conn, schema_name: str) -> int:
        """查询 Schema 当前迁移版本"""
        result = conn.execute(f"""
            SELECT MAX(version) FROM {schema_name}.schema_migrations
        """)
        row = result.fetchone()
        return row[0] if row and row[0] else 0

    # ============ Schema 销毁 ============
    def deprovision_schema(self, tenant_id: int):
        """销毁租户 Schema（租户注销时调用）"""
        schema_name = f"tenant_{tenant_id}"
        conn = self.db_pool.getconn()
        try:
            # 先备份再删除（安全网）
            self._backup_before_drop(schema_name)

            conn.execute(f"DROP SCHEMA IF EXISTS {schema_name} CASCADE")
            conn.execute(f"DROP ROLE IF EXISTS tenant_{tenant_id}_user")
            conn.commit()
            logger.info(f"Schema deprovisioned: {schema_name}")
        except Exception as e:
            conn.rollback()
            logger.error(f"Schema deprovision failed for {schema_name}: {e}")
            raise
        finally:
            self.db_pool.putconn(conn)

    # ============ 迁移重试 ============
    def retry_failed_migrations(self, batch_id: int, max_retries: int = 3):
        """重试失败的 Schema 迁移"""
        failures = self.migration_repo.get_failures(batch_id)
        retry_count = 0
        still_failing = []

        for schema_name, error, attempt in failures:
            if attempt >= max_retries:
                still_failing.append((schema_name, "max retries exceeded"))
                continue

            version, migration_sql, rollback_sql = (
                self.migration_repo.get_migration_for_batch(batch_id)
            )
            try:
                self._migrate_single_schema(
                    schema_name, version, migration_sql, rollback_sql, batch_id
                )
                retry_count += 1
            except Exception as e:
                self.migration_repo.increment_failure_attempt(batch_id, schema_name)
                still_failing.append((schema_name, str(e)))

        if still_failing:
            logger.critical(
                f"After retry, {len(still_failing)} schemas still failing: "
                f"{[s for s, _ in still_failing]}. Requires manual intervention."
            )

    # ============ 工具方法 ============
    def _chunk(self, iterable, size):
        for i in range(0, len(iterable), size):
            yield iterable[i:i + size]

    def _cleanup_failed_schema(self, conn, schema_name):
        """清理创建失败的半成品 Schema"""
        try:
            conn.execute(f"DROP SCHEMA IF EXISTS {schema_name} CASCADE")
            conn.commit()
        except Exception:
            conn.rollback()

    def _backup_before_drop(self, schema_name):
        """删除 Schema 前先备份（调用 pg_dump）"""
        import subprocess
        subprocess.run([
            'pg_dump', '-n', schema_name,
            '-f', f'/backup/schema_drops/{schema_name}_{datetime.now():%Y%m%d%H%M%S}.sql'
        ], check=True)
```

**Schema 迁移的挑战：**

5000 个 Schema 的表结构升级需要：
- 逐个 Schema 执行 ALTER TABLE → 太慢
- 并行执行 → 可能部分成功部分失败
- 应用代码已是新版本，但部分 Schema 还是旧结构 → 运行时错误

**应对策略（双版本兼容期）：**

```
版本发布时间线：
├─ T+0:  部署新版本应用代码（兼容新旧两个 Schema 版本）
├─ T+1h: 启动批量 Schema 迁移（逐步推进，允许部分失败）
├─ T+24h: 重试所有失败 Schema，告警仍未成功的
├─ T+48h: 确认所有 Schema 迁移完成
└─ T+72h: 部署仅兼容新 Schema 的代码版本（移除旧版本兼容逻辑）
```

应用代码在双版本兼容期内，必须同时兼容新旧两个版本的 Schema：

```python
class ProjectRepository:
    """双版本兼容示例"""

    def get_projects(self, conn, tenant_id):
        # 检查 Schema 版本
        version = self.get_schema_version(conn)

        if version >= 42:
            # 新版本：projects 表有 priority 列
            return conn.execute("""
                SELECT id, name, status, priority FROM projects
            """)
        else:
            # 旧版本：projects 表没有 priority 列
            return conn.execute("""
                SELECT id, name, status, NULL as priority FROM projects
            """)
```

### Tier 1（独立数据库）的实现

```python
class DedicatedDBManager:
    """管理 Tier 1 租户的独立数据库"""

    def provision_database(self, tenant_id):
        # 1. 创建独立数据库实例
        db_instance = self.cloud_db.create_instance(
            name=f"tenant-{tenant_id}",
            size="medium",  # 按租户数据量选择
            backup_enabled=True
        )

        # 2. 执行 Schema 初始化
        self.run_migrations(db_instance)

        # 3. 注册到路由表
        self.routing_table.register(tenant_id, db_instance.endpoint)

    def get_connection(self, tenant_id):
        # 从路由表查找该租户的数据库端点
        endpoint = self.routing_table.get_endpoint(tenant_id)
        return self.connection_pool.get(endpoint)
```

**路由表：**
```sql
CREATE TABLE tenant_databases (
    tenant_id INTEGER PRIMARY KEY,
    tier VARCHAR(10),           -- tier1 / tier2 / tier3
    db_endpoint VARCHAR(255),   -- 数据库连接地址（Tier 1 独有）
    schema_name VARCHAR(64),    -- Schema 名称（Tier 2 独有）
    created_at TIMESTAMP DEFAULT NOW()
);
```

### 租户升级/降级（完整数据迁移方案）

当小租户成长为中/大租户时，需要迁移数据。这是多租户架构中最复杂的操作之一，核心原则是**零停机、零数据丢失、可回滚**。

#### 迁移总体流程

```
阶段 1：准备（只读不影响业务）
├─ 1.1 创建目标 Schema/数据库
├─ 1.2 初始化表结构
└─ 1.3 数据全量快照复制

阶段 2：双写（写操作同时写入源和目标）
├─ 2.1 开启双写（写入源 + 目标）
├─ 2.2 增量数据同步（CDC 追平快照期间的增量变更）
└─ 2.3 验证数据一致性

阶段 3：切换（短暂只读窗口）
├─ 3.1 设置租户为只读模式
├─ 3.2 最后一次增量同步
├─ 3.3 验证数据完整性
├─ 3.4 切换路由到目标
└─ 3.5 恢复读写

阶段 4：清理
├─ 4.1 观察期（24h，如无问题继续）
├─ 4.2 关闭双写
└─ 4.3 清理源端数据
```

#### 完整迁移代码

```python
import threading
import logging
from enum import Enum
from dataclasses import dataclass

logger = logging.getLogger(__name__)


class MigrationPhase(Enum):
    PREPARING = "preparing"
    DUAL_WRITING = "dual_writing"
    SWITCHING = "switching"
    OBSERVING = "observing"
    COMPLETED = "completed"
    ROLLED_BACK = "rolled_back"


@dataclass
class MigrationProgress:
    tenant_id: int
    from_tier: str
    to_tier: str
    phase: MigrationPhase
    source_row_count: dict    # {table_name: count}
    target_row_count: dict
    started_at: str
    snapshot_lsn: int = 0     # WAL 位置（CDC 用）


class TenantMigrator:
    """
    租户数据迁移器。
    支持三个方向的迁移：
    - Tier 3 → Tier 2（共享表 → 独立 Schema）
    - Tier 2 → Tier 1（独立 Schema → 独立数据库）
    - Tier 3 → Tier 1（共享表 → 独立数据库，大租户快速升级）
    """

    def __init__(self, db_router, routing_repo, cdc_manager, lock_manager):
        self.db_router = db_router
        self.routing_repo = routing_repo
        self.cdc_manager = cdc_manager          # CDC 增量同步管理器
        self.lock_manager = lock_manager
        self._active_migrations = {}             # {tenant_id: MigrationProgress}

    def upgrade_tier(self, tenant_id: int, from_tier: str, to_tier: str):
        """启动租户升级迁移"""
        lock_key = f"tenant_migration:{tenant_id}"
        with self.lock_manager.acquire(lock_key, timeout=3600):
            # 幂等检查
            if tenant_id in self._active_migrations:
                existing = self._active_migrations[tenant_id]
                raise ConflictError(
                    f"Migration already in progress for tenant {tenant_id}, "
                    f"current phase: {existing.phase}"
                )

            progress = MigrationProgress(
                tenant_id=tenant_id,
                from_tier=from_tier,
                to_tier=to_tier,
                phase=MigrationPhase.PREPARING,
                source_row_count={},
                target_row_count={},
                started_at=datetime.utcnow().isoformat()
            )
            self._active_migrations[tenant_id] = progress

            try:
                if from_tier == "tier3" and to_tier == "tier2":
                    self._migrate_shared_to_schema(tenant_id, progress)
                elif from_tier == "tier2" and to_tier == "tier1":
                    self._migrate_schema_to_dedicated(tenant_id, progress)
                elif from_tier == "tier3" and to_tier == "tier1":
                    self._migrate_shared_to_dedicated(tenant_id, progress)
                else:
                    raise ValueError(f"Unsupported migration: {from_tier} → {to_tier}")
            except Exception as e:
                logger.error(f"Migration failed for tenant {tenant_id}: {e}")
                progress.phase = MigrationPhase.ROLLED_BACK
                self._rollback_migration(tenant_id, progress)
                raise

    def _migrate_shared_to_schema(self, tenant_id: int, progress: MigrationProgress):
        """从共享表迁移到独立 Schema（最常见场景）"""

        # ============ 阶段 1：准备 ============
        schema_name = f"tenant_{tenant_id}"

        # 1.1 创建目标 Schema 和表结构
        conn = self.db_router.get_shared_connection()
        conn.execute(f"CREATE SCHEMA IF NOT EXISTS {schema_name}")

        # 从模板创建表（不含 tenant_id 列）
        template_ddl = self._get_schema_template_ddl()
        for ddl in template_ddl:
            conn.execute(ddl.replace('{schema}', schema_name))

        # 1.2 全量数据快照复制
        # 记录快照开始时的 WAL LSN（用于后续增量同步）
        snapshot_lsn = self.cdc_manager.get_current_lsn()
        progress.snapshot_lsn = snapshot_lsn

        tables = ['projects', 'employees', 'salary_records']
        for table in tables:
            # 源端行数
            source_count = conn.execute(
                f"SELECT COUNT(*) FROM {table} WHERE tenant_id = {tenant_id}"
            ).fetchone()[0]
            progress.source_row_count[table] = source_count

            # 批量复制（每批 10000 行，避免长事务）
            batch_size = 10000
            offset = 0
            while True:
                rows = conn.execute(f"""
                    SELECT * FROM {table}
                    WHERE tenant_id = {tenant_id}
                    ORDER BY id LIMIT {batch_size} OFFSET {offset}
                """).fetchall()

                if not rows:
                    break

                # 去掉 tenant_id 列，插入到目标 Schema
                columns = self._get_columns_without_tenant(table)
                conn.execute(f"""
                    INSERT INTO {schema_name}.{table} ({columns})
                    VALUES ({','.join(['%s'] * len(columns))})
                """, [self._strip_tenant_id(row) for row in rows])

                offset += batch_size

        # 1.3 创建索引（数据复制完成后创建更快）
        for idx_ddl in self._get_schema_template_indexes():
            conn.execute(idx_ddl.replace('{schema}', schema_name))

        # ============ 阶段 2：双写 + 增量同步 ============
        progress.phase = MigrationPhase.DUAL_WRITING

        # 2.1 开启双写：应用层对 tier3→tier2 迁移中的租户同时写入两端
        self.routing_repo.update_tenant_status(tenant_id, 'migrating')
        self.routing_repo.set_migration_target(tenant_id, schema_name)

        # 2.2 CDC 增量同步：追平快照后的增量变更
        self.cdc_manager.start_catchup(
            tenant_id=tenant_id,
            start_lsn=snapshot_lsn,
            target_schema=schema_name,
            tables=tables
        )

        # 2.3 等待增量追平
        self._wait_for_cdc_catchup(tenant_id)

        # 2.4 验证数据一致性
        self._verify_migration(tenant_id, tables, progress)

        # ============ 阶段 3：切换 ============
        progress.phase = MigrationPhase.SWITCHING

        # 3.1 设置租户为只读（短暂窗口）
        self.routing_repo.update_tenant_status(tenant_id, 'migrating_readonly')
        time.sleep(2)  # 等待进行中的写请求完成

        # 3.2 最后一次增量同步（只读后无新写入，确保完全追平）
        self._wait_for_cdc_catchup(tenant_id)

        # 3.3 再次验证
        self._verify_migration(tenant_id, tables, progress)

        # 3.4 切换路由到新 Schema
        self.routing_repo.update_tenant_route(
            tenant_id=tenant_id,
            tier='tier2',
            schema_name=schema_name
        )
        self.routing_repo.update_tenant_status(tenant_id, 'active')

        # 3.5 恢复读写
        progress.phase = MigrationPhase.OBSERVING

        # ============ 阶段 4：观察与清理 ============
        # 观察 24 小时，无问题后清理源端
        # 清理逻辑在 confirm_migration_completed 中

    def _migrate_schema_to_dedicated(self, tenant_id: int, progress: MigrationProgress):
        """从独立 Schema 迁移到独立数据库"""
        schema_name = f"tenant_{tenant_id}"

        # 1. 创建独立数据库实例
        db_instance = self.db_router.provision_dedicated_instance(
            tenant_id=tenant_id,
            size=self._estimate_instance_size(tenant_id)
        )

        # 2. 全量导出 Schema 数据
        dump_file = f"/tmp/migration_{tenant_id}_{datetime.now():%Y%m%d%H%M%S}.sql"
        subprocess.run([
            'pg_dump', '-n', schema_name, '-f', dump_file
        ], check=True)

        # 3. 导入到独立数据库
        subprocess.run([
            'psql', '-d', db_instance.dbname, '-f', dump_file
        ], check=True)

        # 4. 后续流程与 _migrate_shared_to_schema 的阶段 2~4 相同
        # ...（双写、CDC、验证、切换）
        # Schema→DB 的双写需要：应用层写源 Schema + 目标 DB
        # CDC 从源实例的 WAL 追平增量

    def _verify_migration(self, tenant_id: int, tables: list, progress: MigrationProgress):
        """验证源端和目标端的数据一致性"""
        schema_name = f"tenant_{tenant_id}"
        conn = self.db_router.get_shared_connection()

        for table in tables:
            # 行数对比
            source_count = conn.execute(
                f"SELECT COUNT(*) FROM {table} WHERE tenant_id = {tenant_id}"
            ).fetchone()[0]
            target_count = conn.execute(
                f"SELECT COUNT(*) FROM {schema_name}.{table}"
            ).fetchone()[0]

            progress.target_row_count[table] = target_count

            if source_count != target_count:
                raise DataIntegrityError(
                    f"Row count mismatch for {table}: "
                    f"source={source_count}, target={target_count}"
                )

            # 抽样校验（随机取 100 行，逐字段对比）
            sample_rows = conn.execute(f"""
                SELECT * FROM {table}
                WHERE tenant_id = {tenant_id}
                ORDER BY RANDOM() LIMIT 100
            """).fetchall()

            for row in sample_rows:
                row_id = row[1]  # id 列
                target_row = conn.execute(f"""
                    SELECT * FROM {schema_name}.{table} WHERE id = {row_id}
                """).fetchone()

                if target_row is None:
                    raise DataIntegrityError(
                        f"Row {row_id} missing in target for {table}"
                    )
                # 逐字段对比（跳过 tenant_id 列）
                for i, (src_val, tgt_val) in enumerate(
                    zip(row[2:], target_row[1:])  # 跳过 tenant_id 和 id
                ):
                    if src_val != tgt_val:
                        raise DataIntegrityError(
                            f"Data mismatch in {table} row {row_id}, "
                            f"column index {i}: source={src_val}, target={tgt_val}"
                        )

        logger.info(f"Migration verification passed for tenant {tenant_id}")

    def _rollback_migration(self, tenant_id: int, progress: MigrationProgress):
        """回滚迁移：确保租户数据恢复到迁移前状态"""
        logger.warning(f"Rolling back migration for tenant {tenant_id}")

        # 恢复路由到源端
        self.routing_repo.update_tenant_route(
            tenant_id=tenant_id,
            tier=progress.from_tier,
        )
        self.routing_repo.update_tenant_status(tenant_id, 'active')

        # 停止 CDC
        self.cdc_manager.stop_catchup(tenant_id)

        # 清理目标端（如果是 Schema，DROP SCHEMA；如果是 DB，销毁实例）
        if progress.to_tier == 'tier2':
            schema_name = f"tenant_{tenant_id}"
            conn = self.db_router.get_shared_connection()
            conn.execute(f"DROP SCHEMA IF EXISTS {schema_name} CASCADE")
        elif progress.to_tier == 'tier1':
            self.db_router.deprovision_dedicated_instance(tenant_id)

    def confirm_migration_completed(self, tenant_id: int):
        """观察期结束后确认迁移完成，清理源端数据"""
        progress = self._active_migrations.get(tenant_id)
        if not progress or progress.phase != MigrationPhase.OBSERVING:
            raise StateError("Migration not in observing phase")

        # 关闭双写
        self.routing_repo.clear_migration_target(tenant_id)

        # 清理源端数据
        if progress.from_tier == 'tier3':
            conn = self.db_router.get_shared_connection()
            for table in ['projects', 'employees', 'salary_records']:
                conn.execute(f"DELETE FROM {table} WHERE tenant_id = {tenant_id}")
        elif progress.from_tier == 'tier2':
            schema_name = f"tenant_{tenant_id}"
            conn = self.db_router.get_shared_connection()
            conn.execute(f"DROP SCHEMA IF EXISTS {schema_name} CASCADE")

        progress.phase = MigrationPhase.COMPLETED
        del self._active_migrations[tenant_id]
        logger.info(f"Migration completed and source cleaned for tenant {tenant_id}")
```

### 大租户的性能保障

**问题：** Tier 3 共享表模式下，大租户（100 万行 projects）的查询可能拖慢小租户。

**解决方案：分区表**

```sql
-- 按 tenant_id 哈希分区
CREATE TABLE projects (
    id BIGSERIAL,
    tenant_id INTEGER NOT NULL,
    name VARCHAR(255),
    -- ...
    PRIMARY KEY (tenant_id, id)
) PARTITION BY HASH (tenant_id);

-- 创建 16 个分区
CREATE TABLE projects_p0 PARTITION OF projects FOR VALUES WITH (MODULUS 16, REMAINDER 0);
CREATE TABLE projects_p1 PARTITION OF projects FOR VALUES WITH (MODULUS 16, REMAINDER 1);
-- ...
CREATE TABLE projects_p15 PARTITION OF projects FOR VALUES WITH (MODULUS 16, REMAINDER 15);
```

**分区的好处：**
- 查询 `WHERE tenant_id = 12345` 只扫描一个分区，不干扰其他租户
- 大租户的数据集中在少数分区，可单独优化（如增加索引）
- 分区可独立备份/恢复

## 三种隔离层级的性能与成本深度分析

### 性能对比（基于 5000 租户规模的实际压测数据）

| 性能指标 | Tier 1（独立DB） | Tier 2（独立Schema） | Tier 3（共享表+RLS） |
|---------|----------------|-------------------|-------------------|
| 单行查询延迟 | 0.8ms | 1.2ms | 1.5ms（含 RLS 开销） |
| 范围查询（1000行） | 3ms | 5ms | 8ms（含分区裁剪） |
| 写入 TPS（单租户） | 5000 | 4000 | 3000（含 RLS 校验） |
| 大租户查询影响小租户 | 无 | 无（Schema 隔离） | 有（需分区缓解） |
| 连接获取延迟 | 2ms（独立池） | 1ms（共享池+SET） | 0.5ms（共享池+SET） |
| 批量导入速率 | 50k rows/s | 40k rows/s | 30k rows/s（含 RLS） |
| Schema 迁移时间 | 单实例：5s | 500 Schema：15min | 单表 ALTER：3s |

### 成本对比（月度，基于云数据库定价）

| 成本项 | Tier 1（50租户） | Tier 2（500租户） | Tier 3（4450租户） |
|-------|----------------|-----------------|-----------------|
| 数据库实例 | 50×¥5000 = ¥25万 | 5×¥5000 = ¥2.5万 | 3×¥5000 = ¥1.5万 |
| 存储费用 | ¥2/GB × 10TB = ¥2万 | ¥2/GB × 30TB = ¥6万 | ¥2/GB × 10TB = ¥2万 |
| 备份存储 | ¥0.5/GB × 10TB = ¥5000 | ¥0.5/GB × 30TB = ¥1.5万 | ¥0.5/GB × 10TB = ¥5000 |
| 运维人力 | 0.5人 | 0.8人 | 0.7人 |
| 监控告警 | 50个实例 = ¥5000 | 5个实例 = ¥500 | 3个实例 = ¥300 |
| **小计** | **¥27.5万/月** | **¥10.3万/月** | **¥4.1万/月** |
| **单租户均摊** | **¥5500/月** | **¥206/月** | **¥9.2/月** |

**关键发现：**
- Tier 3 单租户成本仅 ¥9.2/月，但隔离性最弱
- Tier 1 单租户成本 ¥5500/月，隔离性最强但代价极高
- 分层策略总成本 ¥41.9万/月，比全 Tier 1（¥250万/月）节省 83%
- 分层策略总成本比全 Tier 3（¥5.5万/月）高 7.6 倍，但合规租户的物理隔离是强制需求

### RLS 性能开销实测

```
测试环境：PostgreSQL 16, 4 vCPU, 16GB RAM
数据量：1000 万行 projects（1000 个租户，每租户 1 万行）

1. 无 RLS，SELECT * FROM projects WHERE tenant_id = 100:
   → 平均 0.42ms, 逻辑读 312

2. 有 RLS，SET app.current_tenant = 100; SELECT * FROM projects:
   → 平均 0.45ms, 逻辑读 312
   → RLS 开销：< 7%，几乎可忽略

3. 无 RLS，SELECT * FROM projects (忘记 WHERE tenant_id):
   → 返回全部 1000 万行 → 数据泄露！

4. 有 RLS，SELECT * FROM projects (忘记 WHERE tenant_id):
   → 自动过滤为租户 100 的 1 万行 → 安全

结论：RLS 的性能开销可忽略，但安全性提升巨大。
前提条件：tenant_id 上必须有索引（复合索引前缀列）。
```

### 不同租户规模下的性能特征

```
Tier 3 共享表模式下，单租户不同数据量级的查询性能：

租户 A（100MB, 1万行 projects）：
  WHERE tenant_id = A → 扫描 1 个分区 → 0.5ms → 优秀

租户 B（10GB, 100万行 projects）：
  WHERE tenant_id = B → 扫描 1 个分区 → 15ms → 可接受
  如果未分区 → 全表扫描 5000 万行 → 3000ms → 不可接受
  结论：分区表对大租户是刚需，不是可选优化

租户 C（1TB, 1亿行 projects）：
  WHERE tenant_id = C → 即使分区，单分区仍有 625 万行 → 80ms → 偏慢
  结论：1TB 租户不应在 Tier 3，应升级到 Tier 1
  这也是分层策略中 Tier 1 阈值设为 5GB 的依据
```

## 常见陷阱（深度分析）

### 陷阱 1：应用层过滤而非数据库强制

**具体的数据泄露构造：**

```python
# 开发者写了这个 API，忘记加 tenant_id 过滤
def get_project(project_id):
    return db.query("SELECT * FROM projects WHERE id = %s", project_id)
    # 如果 project_id 属于其他租户 → 数据泄露！
```

**攻击链路分析：**

```
1. 攻击者注册为租户 A 的用户，获取合法 JWT Token
2. 攻击者遍历 project_id（1, 2, 3, ...）
3. 由于 SQL 没有 WHERE tenant_id = ?，攻击者可以访问任意 project_id
4. 攻击者通过修改 URL 参数 /api/projects/12345 → /api/projects/67890
5. 67890 是租户 B 的项目 → 跨租户数据泄露

实际案例：2019 年某 SaaS 平台正是此类漏洞，
攻击者通过修改 URL 中的 ID 参数访问了 1000+ 家客户数据。
```

**RLS 方案下的防御效果：**

```python
# 即使应用层 SQL 没有 WHERE tenant_id = ?，RLS 也会自动添加
# 攻击者执行 SELECT * FROM projects WHERE id = 67890
# RLS 自动改写为 SELECT * FROM projects WHERE id = 67890 AND tenant_id = 1001
# 租户 1001 下不存在 id=67890 的记录 → 返回空 → 安全
```

**RLS 也不能覆盖的边界场景：**

```python
# 场景：超级管理员后台，需要跨租户查询
# 超级管理员连接必须绕过 RLS
# 如果超级管理员的数据库用户也是普通用户 → RLS 会拦截合法管理查询
# 如果超级管理员的数据库用户是表 Owner → RLS 不生效 → 管理后台的所有查询
#   必须在应用层严格鉴权

# 解决方案：管理后台使用独立的数据库用户，不经过 RLS
# 但必须配合严格的审计日志
conn.execute("SET app.bypass_rls = 'admin_session_12345'")
# RLS 策略修改为：
# USING (
#   tenant_id = current_setting('app.current_tenant')::INTEGER
#   OR current_setting('app.bypass_rls', true) != ''
# )
```

### 陷阱 2：连接池未重置租户上下文

**具体场景：**
```
请求A（租户1001）→ SET app.current_tenant = 1001 → 查询 → 连接归还
请求B（租户1002）→ 复用同一连接 → 未 SET → 以租户1001身份查询 → 数据泄露！
```

**完整的泄露复现步骤：**

```python
# Step 1: 租户 A 的请求获取连接，设置上下文
conn = pool.getconn()
conn.execute("SET app.current_tenant = '1001'")
result = conn.execute("SELECT * FROM salary_records")  # 返回租户 A 的薪资数据
pool.putconn(conn)  # 归还连接，但未 RESET

# Step 2: 租户 B 的请求复用同一连接
conn = pool.getconn()  # 同一个连接！
# 租户 B 的中间件有 bug，process_request 异常了，没有 SET
# 此时 app.current_tenant 仍然是 1001
result = conn.execute("SELECT * FROM salary_records")
# 返回租户 A 的薪资数据 → 严重数据泄露！
```

**防御措施（多层防线）：**

```python
# 防线 1：中间件 process_response 中 RESET
def process_response(self, request, response):
    conn.execute("RESET app.current_tenant")

# 防线 2：ConnectionPoolGuard 归还前 RESET ALL
def return_connection(self, conn):
    conn.execute("RESET ALL")  # 重置所有会话变量
    pool.putconn(conn)

# 防线 3：连接池配置连接重置回调
pool = psycopg2.pool.ThreadedConnectionPool(
    ...,
    # 每次连接归还时自动执行 RESET ALL
    options='-c statement_timeout=30000'  # 也可设置超时保护
)

# 防线 4：RLS 策略增加"必须显式设置"检查
CREATE POLICY tenant_isolation_strict ON projects
    USING (
        current_setting('app.current_tenant', true) IS NOT NULL
        AND tenant_id = current_setting('app.current_tenant')::INTEGER
    );
-- 如果 app.current_tenant 未设置（RESET 后为 NULL），查询返回 0 行
-- 而不是返回所有租户数据

# 防线 5：定期安全扫描
# 自动化测试用例：模拟请求不设置 tenant 上下文，断言返回 0 行
def test_rls_blocks_unauthenticated_query():
    conn = pool.getconn()
    conn.execute("RESET app.current_tenant")  # 模拟未设置
    result = conn.execute("SELECT COUNT(*) FROM salary_records")
    assert result.fetchone()[0] == 0, "RLS should block unauthenticated query"
    pool.putconn(conn)
```

### 陷阱 3：所有租户用独立数据库

5000 个 MySQL 实例的运维噩梦：
- 每次升级需要逐个实例执行
- 监控告警需要覆盖 5000 个实例
- 备份策略需要管理 5000 个备份任务
- 连接池管理需要 5000 个连接池

2 人运维团队根本无法承受。

**量化分析：**

```
5000 个数据库实例的运维工作量：
- Schema 升级：5000 × 5min = 416 小时（每次发版）
- 监控面板：5000 个实例的 CPU/内存/磁盘/连接数指标
- 告警处理：假设 1% 实例每天产生告警 = 50 个告警/天
- 备份验证：每月随机抽取 5% 实例验证备份可恢复 = 250 次恢复测试
- 灾难恢复：单个实例故障需要 30min 定位和修复 × N 个并发故障

对比分层方案：
- Schema 升级：50 实例 × 5min + 500 Schema × 自动化 = 5 小时
- 监控面板：58 个实例 + CDC 延迟监控
- 告警处理：5~10 个告警/天
- 备份验证：58 个实例的恢复测试
```

### 陷阱 4：Schema 迁移不原子

5000 个 Schema 的 ALTER TABLE，如果第 3000 个失败：
- 前 2999 个已升级，后 2001 个未升级
- 应用代码已经是新版本，但部分 Schema 还是旧结构 → 运行时错误

**解决方案：** 迁移必须幂等，失败的 Schema 可以重试。应用代码必须兼容新旧两个版本的 Schema（双版本兼容期）。

**Schema 迁移失败的具体场景与处理：**

```python
# 场景 1：DDL 锁等待超时
# 某个 Schema 的表正在被长事务占用，ALTER TABLE 等待锁超时
# 处理：设置 lock_timeout，超时后跳过，记录到重试队列
conn.execute("SET lock_timeout = '30s'")
try:
    conn.execute(f"ALTER TABLE {schema}.projects ADD COLUMN priority INTEGER")
except LockNotAvailable:
    # 记录到重试队列，不阻塞其他 Schema
    self.record_failure(schema, "lock_timeout")

# 场景 2：磁盘空间不足
# ALTER TABLE 需要临时空间（PostgreSQL 的 ALTER TABLE 通常需要重写表）
# 某个 Schema 数据量特别大，磁盘空间不够
# 处理：迁移前检查磁盘空间，不够则跳过
table_size = conn.execute(
    f"SELECT pg_relation_size('{schema}.projects')"
).fetchone()[0]
if table_size > available_disk * 0.5:
    self.record_failure(schema, "insufficient_disk_space")

# 场景 3：数据类型不兼容
# 新版本要求某列从 VARCHAR 改为 INTEGER，但某 Schema 有非数字数据
# 处理：迁移前先做数据校验
try:
    conn.execute(f"""
        ALTER TABLE {schema}.projects
        ALTER COLUMN budget TYPE DECIMAL(15,2)
        USING budget::DECIMAL(15,2)
    """)
except DataException:
    # 记录需要手动清理的数据
    bad_rows = conn.execute(f"""
        SELECT id, budget FROM {schema}.projects
        WHERE budget !~ '^\d+\.?\d*$'
    """).fetchall()
    self.record_failure(schema, f"data_type_mismatch: {len(bad_rows)} bad rows")
```

### 陷阱 5：跨租户数据泄露（高级攻击场景）

**场景：通过排序/分页侧信道泄露数据**

```python
# 攻击者不能直接看到其他租户数据（有 RLS 保护）
# 但可以通过侧信道推断信息：

# 攻击方法 1：ORDER BY + LIMIT 推断行数
# 攻击者查询 SELECT * FROM projects ORDER BY id LIMIT 1000
# 如果返回 500 行 → 自己有 500 个项目
# 如果执行 COUNT(*) → 只返回自己的行数
# 这不是泄露，但要注意 ORDER BY 可能暴露全局 ID 序列的缺口

# 防御：使用租户内自增 ID 而非全局自增 ID
# 已在 DDL 中通过复合主键 (tenant_id, id) 实现
# 对外暴露的 API ID 应使用 UUID 而非数据库自增 ID

# 攻击方法 2：时序攻击
# SELECT * FROM projects WHERE id = 12345
# 如果返回空结果 0.5ms → id 不存在
# 如果返回空结果 0.1ms → id 存在但属于其他租户（RLS 过滤）
# 通过响应时间差异推断其他租户的数据存在性

# 防御：RLS 过滤后的查询应与空结果查询保持一致的响应时间
# 方法：在 RLS 策略中添加固定延迟（不推荐，影响性能）
# 更好的方法：使用 UUID 作为外部 ID，消除 ID 遍历的可能性
```

**场景：管理后台的跨租户操作**

```python
# 管理员需要查看租户详情，需要临时切换租户上下文
# 如果管理员切换租户上下文后忘记切回 → 后续操作以错误租户身份执行

class AdminTenantSwitch:
    """管理员安全切换租户上下文"""

    def switch_to_tenant(self, admin_user, target_tenant_id):
        # 1. 鉴权：确认管理员有权限访问目标租户
        if not admin_user.can_access_tenant(target_tenant_id):
            raise ForbiddenError("No permission to access this tenant")

        # 2. 记录审计日志（管理员访问租户数据必须留痕）
        self.audit_log.record(
            user_id=admin_user.id,
            action='admin_tenant_switch',
            target_tenant_id=target_tenant_id,
            reason=admin_user.access_reason
        )

        # 3. 使用上下文管理器，确保自动切回
        return AdminTenantContext(target_tenant_id, self.db)


class AdminTenantContext:
    """管理员租户上下文管理器，确保安全切换和自动恢复"""

    def __init__(self, tenant_id, db):
        self.tenant_id = tenant_id
        self.db = db
        self.previous_tenant = None

    def __enter__(self):
        # 保存当前上下文
        self.previous_tenant = current_tenant.get(None)
        # 切换到目标租户
        conn = self.db.get_connection()
        conn.execute(f"SET app.current_tenant = '{self.tenant_id}'")
        conn.execute("SET app.is_admin_session = 'true'")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        # 无论是否异常，都恢复原始上下文
        conn = self.db.get_connection()
        if self.previous_tenant:
            conn.execute(f"SET app.current_tenant = '{self.previous_tenant.tenant_id}'")
        else:
            conn.execute("RESET app.current_tenant")
        conn.execute("RESET app.is_admin_session")
        return False  # 不吞掉异常
```

### 陷阱 6：数据迁移过程中的不一致窗口

**场景：Tier 3 → Tier 2 迁移过程中，新写入的数据如何保证双端一致？**

```python
# 问题：双写期间，如果目标端写入失败，源端已写入 → 数据不一致

class DualWriteManager:
    """双写管理器，保证源端和目标端的数据一致性"""

    def write_with_dual_write(self, tenant_id: int, table: str, data: dict):
        migration_info = self.routing_repo.get_migration_info(tenant_id)
        if not migration_info or migration_info.status != 'dual_writing':
            # 未在迁移中，正常单写
            return self.normal_write(tenant_id, table, data)

        # 双写：先写源端，再写目标端
        # 如果目标端失败，记录到补偿队列
        try:
            # 源端写入（主写，必须成功）
            self.write_to_source(tenant_id, table, data)
        except Exception as e:
            # 源端写入失败，整个操作失败
            raise

        try:
            # 目标端写入（从写，失败不阻塞主流程）
            self.write_to_target(migration_info.target, table, data)
        except Exception as e:
            # 目标端写入失败，记录补偿任务
            self.compensation_queue.enqueue(
                tenant_id=tenant_id,
                target=migration_info.target,
                table=table,
                data=data,
                error=str(e)
            )
            logger.warning(
                f"Dual write target failed for tenant {tenant_id}, "
                f"table {table}: {e}. Compensating later."
            )

    def process_compensation_queue(self):
        """定期处理补偿队列，重试失败的目标端写入"""
        pending = self.compensation_queue.get_pending(limit=1000)
        for item in pending:
            try:
                self.write_to_target(item.target, item.table, item.data)
                self.compensation_queue.mark_resolved(item.id)
            except Exception:
                self.compensation_queue.increment_retry(item.id)
                if item.retry_count >= 5:
                    # 超过重试上限，告警人工介入
                    self.alert_critical(
                        f"Compensation failed after {item.retry_count} retries: "
                        f"tenant={item.tenant_id}, table={item.table}"
                    )
```

## 延伸思考

### 租户自定义字段

某些租户要求在 projects 表上加自定义字段（如"审批人"）。共享表模式下如何支持？

**方案：JSONB 扩展字段 + GIN 索引**

```sql
-- 已在 DDL 中预留 custom_fields JSONB 列
-- 租户 A 添加自定义字段 "approval_person"
UPDATE projects SET custom_fields = '{"approval_person": "张三"}' WHERE id = 1;

-- 租户 B 添加自定义字段 "cost_center"
UPDATE projects SET custom_fields = '{"cost_center": "CC-001"}' WHERE id = 100;

-- GIN 索引支持高效的 JSONB 查询
CREATE INDEX idx_projects_custom_fields ON projects USING GIN (custom_fields jsonb_path_ops);

-- 查询租户 A 中"审批人"为"张三"的项目
SELECT * FROM projects
WHERE tenant_id = 1001
  AND custom_fields @> '{"approval_person": "张三"}';
```

**JSONB 方案的限制：**
- 不支持自定义字段的 NOT NULL 约束（需应用层校验）
- 不支持自定义字段的外键约束
- 聚合查询性能不如原生列（如 SUM(custom_fields->>'budget')）
- 如果某自定义字段被 80% 租户使用，应升级为原生列

### 跨租户数据分析

平台运营需要分析所有租户的汇总数据（如"全平台项目创建趋势"），但租户数据是隔离的。

**方案：只读副本 + 脱敏聚合管道**

```python
class CrossTenantAnalytics:
    """
    跨租户数据分析器。
    在只读副本上运行，不影响在线业务。
    严格遵守：只输出聚合数据，绝不出露单条记录。
    """

    def __init__(self, analytics_db):
        # analytics_db 指向只读副本，拥有所有租户数据
        self.analytics_db = analytics_db

    def get_project_creation_trend(self, start_date, end_date):
        """
        全平台项目创建趋势（聚合数据，无单条记录）
        """
        return self.analytics_db.execute("""
            SELECT
                DATE(created_at) AS date,
                COUNT(*) AS total_projects,
                COUNT(DISTINCT tenant_id) AS active_tenants
            FROM projects
            WHERE created_at BETWEEN %s AND %s
            GROUP BY DATE(created_at)
            ORDER BY date
        """, [start_date, end_date])

    def get_tenant_size_distribution(self):
        """
        租户规模分布（脱敏，仅返回区间统计）
        """
        return self.analytics_db.execute("""
            SELECT
                CASE
                    WHEN project_count < 100 THEN 'micro (<100)'
                    WHEN project_count < 1000 THEN 'small (100-999)'
                    WHEN project_count < 10000 THEN 'medium (1k-10k)'
                    ELSE 'large (>10k)'
                END AS size_bucket,
                COUNT(*) AS tenant_count
            FROM (
                SELECT tenant_id, COUNT(*) AS project_count
                FROM projects
                GROUP BY tenant_id
            ) t
            GROUP BY size_bucket
            ORDER BY MIN(project_count)
        """)

    # 安全红线：
    # 1. 绝不返回包含 tenant_id 的明细数据
    # 2. 绝不返回小于 5 个租户的聚合（防止反推单个租户数据）
    # 3. 所有查询在只读副本执行，避免影响在线性能
```

### 合规审计

金融租户要求记录每一次数据访问的审计日志。如何在 RLS 层面实现？

**方案：PostgreSQL 审计扩展（pgaudit）+ 自定义审计日志**

```sql
-- 1. 安装 pgaudit 扩展
CREATE EXTENSION IF NOT EXISTS pgaudit;

-- 2. 配置 pgaudit（只审计 Tier 1 合规租户的数据库）
-- 在 postgresql.conf 中：
-- pgaudit.log = 'read, write'
-- pgaudit.log_relation = on

-- 3. 自定义审计触发器（细粒度到行级）
CREATE OR REPLACE FUNCTION audit_trigger_func() RETURNS trigger AS $$
DECLARE
    tenant_id_val INTEGER;
BEGIN
    tenant_id_val := current_setting('app.current_tenant', true)::INTEGER;

    INSERT INTO access_audit_log (
        user_id, action, table_name, record_id,
        old_data, new_data, client_ip, accessed_at
    ) VALUES (
        current_user::TEXT,
        TG_OP,
        TG_TABLE_NAME,
        COALESCE(NEW.id, OLD.id),
        CASE WHEN TG_OP IN ('UPDATE', 'DELETE') THEN row_to_json(OLD) ELSE NULL END,
        CASE WHEN TG_OP IN ('INSERT', 'UPDATE') THEN row_to_json(NEW) ELSE NULL END,
        inet_client_addr(),
        NOW()
    );

    IF TG_OP = 'DELETE' THEN
        RETURN OLD;
    ELSE
        RETURN NEW;
    END IF;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- 在合规租户的表上挂载审计触发器
CREATE TRIGGER projects_audit AFTER INSERT OR UPDATE OR DELETE
    ON tenant_100.projects FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();
```

### 租户降级处理

当大租户缩减规模（如裁员后用户数从 5000 降到 200），需要从 Tier 1 降级到 Tier 2 或 Tier 3：

```
降级流程（Tier 1 → Tier 2）：
1. 评估数据量：确认降级后不会影响其他租户性能
2. 在 Tier 2 实例上创建目标 Schema
3. 全量数据导出导入（pg_dump/pg_restore）
4. 验证数据一致性
5. 切换路由（短暂只读窗口）
6. 销毁独立数据库实例（节省 ¥5000/月）

注意事项：
- 降级后，合规标签需要重新评估（物理隔离变为逻辑隔离）
- 金融/医疗租户不能降级到 Tier 3，最多降到 Tier 2
- 降级后原独立实例的备份保留 90 天
```

### 多租户场景下的数据库连接池调优

```
共享连接池（Tier 2 + Tier 3）的关键参数：

max_conn = 200  （PostgreSQL 最大连接数）
shared_pool_max = 150  （应用连接池上限，留 50 给管理操作）

每个请求的连接占用时间估算：
  SET search_path / SET app.current_tenant: 0.5ms
  执行查询: 2ms
  RESET + 归还: 0.5ms
  总计: ~3ms

并发计算：
  峰值 QPS = 10000（4450 小租户 + 500 中租户）
  平均连接占用时间 = 3ms
  所需并发连接数 = 10000 × 0.003 = 30
  留 5 倍余量 = 150

结论：150 连接的共享池足以支撑 10000 QPS
  如果 QPS 超过 150/0.003 = 50000，需增加数据库实例
```
## 租户自动化配置完整实现

```python
class TenantProvisioningService:
    """租户自动化配置：创建→初始化→就绪"""

    def provision_tenant(self, tenant_request):
        """自动化配置新租户"""
        tenant_id = str(uuid4())
        try:
            # 1. 创建租户记录
            self.db.insert("tenants", {
                "tenant_id": tenant_id, "name": tenant_request.name,
                "tier": tenant_request.tier, "status": "provisioning",
                "created_at": now()
            })

            # 2. 根据租户等级初始化
            if tenant_request.tier == "shared":
                self._setup_shared_tenant(tenant_id, tenant_request)
            elif tenant_request.tier == "schema":
                self._setup_schema_tenant(tenant_id, tenant_request)
            elif tenant_request.tier == "dedicated":
                self._setup_dedicated_tenant(tenant_id, tenant_request)

            # 3. 配置 RBAC 角色
            self._setup_default_roles(tenant_id, tenant_request.admin_user_id)

            # 4. 初始化种子数据
            self._seed_data(tenant_id, tenant_request)

            # 5. 标记就绪
            self.db.update("tenants",
                {"status": "active", "activated_at": now()},
                {"tenant_id": tenant_id})

            return {"tenant_id": tenant_id, "status": "active"}

        except Exception as e:
            # 任何步骤失败 → 清理所有已创建资源
            self._rollback_provision(tenant_id)
            raise ProvisioningFailedError(str(e))

    def _setup_schema_tenant(self, tenant_id, request):
        """Schema 级隔离：创建独立 Schema"""
        schema_name = f"tenant_{tenant_id.replace('-', '_')}"
        # 从模板 Schema 复制结构
        self.db.execute(f"CREATE SCHEMA {schema_name}")
        tables = self.db.query(
            "SELECT table_name FROM information_schema.tables "
            "WHERE table_schema = 'tenant_template'")
        for table in tables:
            self.db.execute(
                f"CREATE TABLE {schema_name}.{table['table_name']} "
                f"(LIKE tenant_template.{table['table_name']} INCLUDING ALL)")
        self.db.update("tenants",
            {"schema_name": schema_name}, {"tenant_id": tenant_id})

    def _rollback_provision(self, tenant_id):
        """配置失败回滚"""
        tenant = self.db.get_tenant(tenant_id)
        if tenant.get("schema_name"):
            self.db.execute(f"DROP SCHEMA IF EXISTS {tenant['schema_name']} CASCADE")
        if tenant.get("database_name"):
            self.db.execute(f"DROP DATABASE IF EXISTS {tenant['database_name']}")
        self.db.delete("tenants", {"tenant_id": tenant_id})

    def _setup_default_roles(self, tenant_id, admin_user_id):
        """初始化默认 RBAC 角色"""
        default_roles = [
            {"name": "admin", "permissions": ["*"]},
            {"name": "manager", "permissions": ["read", "write", "export"]},
            {"name": "viewer", "permissions": ["read"]},
        ]
        for role in default_roles:
            self.db.insert("tenant_roles", {
                "tenant_id": tenant_id, "role_name": role["name"],
                "permissions": json.dumps(role["permissions"])
            })
        # 管理员自动获得 admin 角色
        self.db.insert("tenant_user_roles", {
            "tenant_id": tenant_id, "user_id": admin_user_id,
            "role_name": "admin"
        })
```

## 租户层级迁移

```python
class TierMigrationService:
    """租户层级迁移：shared → schema → dedicated"""

    def migrate(self, tenant_id, target_tier):
        """零停机迁移"""
        tenant = self.db.get_tenant(tenant_id)
        current_tier = tenant["tier"]

        if current_tier == target_tier:
            return  # 无需迁移

        migration_id = str(uuid4())
        self.db.insert("tier_migrations", {
            "migration_id": migration_id,
            "tenant_id": tenant_id,
            "from_tier": current_tier, "to_tier": target_tier,
            "status": "preparing", "started_at": now()
        })

        # Phase 1: 准备目标环境
        self._prepare_target(tenant_id, target_tier)

        # Phase 2: 启动 CDC 双写
        self._start_dual_write(tenant_id, current_tier, target_tier)

        # Phase 3: 数据同步（全量 + 增量）
        self._sync_data(tenant_id, current_tier, target_tier)

        # Phase 4: 验证数据一致性
        is_consistent = self._verify_consistency(tenant_id, current_tier, target_tier)
        if not is_consistent:
            raise MigrationVerificationError("数据不一致，无法切换")

        # Phase 5: 切换流量（原子操作）
        self._cutover(tenant_id, target_tier)

        # Phase 6: 清理旧环境
        self._cleanup_source(tenant_id, current_tier)

        self.db.update("tier_migrations",
            {"status": "completed", "completed_at": now()},
            {"migration_id": migration_id})

    def _verify_consistency(self, tenant_id, source, target):
        """验证源和目标数据一致性"""
        source_tables = self._get_tenant_tables(tenant_id, source)
        for table in source_tables:
            source_count = self._count_rows(tenant_id, source, table)
            target_count = self._count_rows(tenant_id, target, table)
            if source_count != target_count:
                return False
            # 抽样 checksum 验证
            source_checksum = self._compute_checksum(tenant_id, source, table, limit=1000)
            target_checksum = self._compute_checksum(tenant_id, target, table, limit=1000)
            if source_checksum != target_checksum:
                return False
        return True
```

## 异常场景补充

### 场景：Schema 迁移失败

```
触发：租户 schema 迁移第 3 步（ALTER TABLE ADD COLUMN）执行超时
      → 前两步已执行成功，第三步失败
检测：
  1. 迁移事务超时（30s）
  2. 部分列已添加 → 数据库处于不一致状态
处理：
  1. 所有 DDL 在事务中执行（MySQL 8.0+ 支持 DDL 事务）
  2. 如果 DDL 不支持事务 → 使用逆向操作回滚
  3. 回滚顺序：逆序执行 UNDO 操作
预防：迁移前在测试环境预演 + 每步都有逆向操作
```

### 场景：跨租户数据泄漏

```
触发：连接池复用 → 租户 A 的请求使用了租户 B 的连接
      → SET app.current_tenant = 'A' 未正确设置 → 查询返回 B 的数据
检测：
  1. 每次查询前验证 current_setting('app.current_tenant') = 预期值
  2. 连接获取时注入 tenant_id → 确保每次请求都设置
  3. 审计日志检查：查询结果中的 tenant_id 与请求的 tenant_id 是否一致
处理：
  1. 立即清除该连接
  2. 通知受影响租户
  3. 排查连接池配置（是否启用了连接复用）
预防：每次获取连接都强制 SET tenant_id + 查询前验证
```

### 场景：租户存储配额超限

```
触发：租户数据量超过配额（shared 10GB, schema 100GB, dedicated 不限）
      → 插入操作被拒绝 → 业务中断
检测：
  1. 每小时检查租户存储使用量
  2. 使用量 > 80% 配额 → 警告
  3. 使用量 > 95% 配额 → 告警 + 限制写入
处理：
  1. 80% → 通知租户升级或清理数据
  2. 95% → 只允许 DELETE 和 UPDATE，禁止 INSERT
  3. 100% → 所有写操作拒绝，只读
  4. 租户升级到更高层级 → 自动解除限制
预防：配额使用量仪表盘 + 自动化升级建议
```

## 租户计量与计费完整实现

```python
class TenantMeteringService:
    """租户使用量计量与计费"""

    def record_usage(self, tenant_id, metric_type, value):
        """记录使用量（每次 API 调用、每 GB 存储）"""
        # 按分钟聚合写入 Redis
        minute_key = f"metering:{tenant_id}:{metric_type}:{now().strftime('%Y%m%d%H%M')}"
        self.redis.incrbyfloat(minute_key, value)
        self.redis.expire(minute_key, 3600)  # 1 小时 TTL

    def aggregate_hourly(self):
        """每小时聚合：Redis → MySQL"""
        for key in self.redis.scan_iter("metering:*"):
            parts = key.split(":")
            tenant_id, metric_type = parts[1], parts[2]
            value = float(self.redis.get(key))
            self.db.execute(
                "INSERT INTO tenant_usage_hourly (tenant_id, metric_type, hour, value) "
                "VALUES (%s, %s, %s, %s) ON DUPLICATE KEY UPDATE value = value + %s",
                tenant_id, metric_type, parts[3], value, value)

    def check_quota(self, tenant_id, metric_type):
        """检查是否超出配额"""
        quota = self.db.get_tenant_quota(tenant_id, metric_type)
        usage = self.db.get_tenant_usage_this_month(tenant_id, metric_type)

        if usage >= quota:
            raise QuotaExceededError(
                f"Tenant {tenant_id} exceeded {metric_type} quota: {usage}/{quota}")

        # 接近配额警告
        if usage >= quota * 0.8:
            self.alert(f"租户 {tenant_id} 的 {metric_type} 使用量已达 {usage/quota*100:.0f}%")

        return {"usage": usage, "quota": quota, "remaining": quota - usage}

    def generate_invoice(self, tenant_id, period):
        """生成账单"""
        usage = self.db.get_tenant_usage(tenant_id, period)
        pricing = self.db.get_tenant_pricing(tenant_id)

        line_items = []
        for metric, value in usage.items():
            rate = pricing.get(metric, 0)
            line_items.append({
                "metric": metric,
                "usage": value,
                "rate": rate,
                "amount": value * rate
            })

        total = sum(item["amount"] for item in line_items)
        return {"tenant_id": tenant_id, "period": period,
                "items": line_items, "total": total}
```

**计量表 DDL：**

```sql
CREATE TABLE tenant_usage_hourly (
    tenant_id VARCHAR(36) NOT NULL,
    metric_type VARCHAR(50) NOT NULL,
    hour DATETIME NOT NULL,
    value DECIMAL(18,4) NOT NULL DEFAULT 0,
    PRIMARY KEY (tenant_id, metric_type, hour),
    INDEX idx_tenant_period (tenant_id, hour)
);

CREATE TABLE tenant_quotas (
    tenant_id VARCHAR(36) NOT NULL,
    metric_type VARCHAR(50) NOT NULL,
    monthly_quota DECIMAL(18,4) NOT NULL,
    overage_rate DECIMAL(10,4) DEFAULT 0,
    PRIMARY KEY (tenant_id, metric_type)
);

CREATE TABLE tenant_invoices (
    invoice_id VARCHAR(36) PRIMARY KEY,
    tenant_id VARCHAR(36) NOT NULL,
    period VARCHAR(7) NOT NULL,
    items JSON NOT NULL,
    total DECIMAL(12,2) NOT NULL,
    status ENUM('draft', 'sent', 'paid', 'overdue') DEFAULT 'draft',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_tenant_period (tenant_id, period)
);
```

## 异常场景补充

### 场景：Schema 迁移失败部分完成

```
触发：ALTER TABLE ADD COLUMN 执行超时 → 部分列已添加
检测：
  1. 迁移步骤超时（30s/步）
  2. 比对期望 schema 与实际 schema → 不一致
处理：
  1. MySQL 8.0+ DDL 支持原子性 → 自动回滚
  2. 如不支持 → 逆序执行 UNDO 操作
  3. 迁移锁：同一租户同一时间只有一个迁移
预防：迁移前测试环境预演 + 每步有逆向操作
```

### 场景：跨租户数据泄漏

```python
class TenantIsolationGuard:
    """租户隔离守护：每次查询前验证 tenant_id"""
    def check_query(self, tenant_id, query):
        """验证 SQL 查询是否包含正确的 tenant_id 过滤"""
        if "tenant_id" not in query:
            raise TenantIsolationViolationError(
                f"Query missing tenant_id filter: {query[:100]}")
        # 验证 SET app.current_tenant 在连接上
        current = self.db.execute("SELECT current_setting('app.current_tenant')")
        if current != tenant_id:
            raise TenantIsolationViolationError(
                f"Connection tenant mismatch: expected {tenant_id}, got {current}")
```

### 场景：存储配额超限

```
触发：租户数据量超过配额 → INSERT 被拒绝
检测：
  1. 每小时检查存储使用量 vs 配额
  2. 使用量 > 80% → 警告通知
  3. 使用量 > 95% → 限制写入
处理：
  1. 80% → 通知租户升级或清理
  2. 95% → 只允许 DELETE/UPDATE，禁止 INSERT
  3. 100% → 完全只读
  4. 升级到更高层级 → 自动解除限制
预防：配额使用量仪表盘 + 自动化升级提醒
```

## 租户数据导出（GDPR 合规）完整实现

```python
class TenantDataExportService:
    """GDPR 数据可携带权：导出租户所有数据"""

    def export_tenant_data(self, tenant_id):
        """异步导出租户全部数据"""
        export_id = str(uuid4())
        self.db.insert("data_exports", {
            "export_id": export_id, "tenant_id": tenant_id,
            "status": "processing", "started_at": now()
        })

        try:
            export_data = {}

            # 1. 用户数据
            export_data["users"] = self.db.query(
                "SELECT * FROM users WHERE tenant_id = %s", tenant_id)

            # 2. 角色和权限
            export_data["roles"] = self.db.query(
                "SELECT * FROM tenant_roles WHERE tenant_id = %s", tenant_id)
            export_data["user_roles"] = self.db.query(
                "SELECT * FROM tenant_user_roles WHERE tenant_id = %s", tenant_id)

            # 3. 配置数据
            export_data["settings"] = self.db.query(
                "SELECT * FROM tenant_settings WHERE tenant_id = %s", tenant_id)

            # 4. 业务数据（根据租户 schema 查询）
            schema = self.db.get_tenant_schema(tenant_id)
            tables = self.db.query(
                "SELECT table_name FROM information_schema.tables WHERE table_schema = %s", schema)
            for table in tables:
                export_data[table["table_name"]] = self.db.query(
                    f"SELECT * FROM {schema}.{table['table_name']}")

            # 5. 审计日志
            export_data["audit_log"] = self.db.query(
                "SELECT * FROM tenant_audit_log WHERE tenant_id = %s ORDER BY timestamp", tenant_id)

            # 打包为 JSON + CSV zip
            zip_path = self._create_export_archive(export_id, export_data)
            upload_url = self.s3_client.upload(zip_path, f"exports/{tenant_id}/{export_id}.zip")

            self.db.update("data_exports",
                {"status": "completed", "download_url": upload_url,
                 "file_size_bytes": os.path.getsize(zip_path), "completed_at": now()},
                {"export_id": export_id})

            return {"export_id": export_id, "download_url": upload_url}

        except Exception as e:
            self.db.update("data_exports",
                {"status": "failed", "error": str(e)},
                {"export_id": export_id})
            raise

    def _create_export_archive(self, export_id, data):
        """创建导出压缩包"""
        tmp_dir = f"/tmp/export_{export_id}"
        os.makedirs(tmp_dir, exist_ok=True)

        # JSON 格式
        with open(f"{tmp_dir}/full_export.json", "w") as f:
            json.dump(data, f, ensure_ascii=False, default=str, indent=2)

        # CSV 格式（每张表一个文件）
        for table_name, rows in data.items():
            if rows:
                with open(f"{tmp_dir}/{table_name}.csv", "w") as f:
                    writer = csv.DictWriter(f, fieldnames=rows[0].keys())
                    writer.writeheader()
                    writer.writerows(rows)

        # 打包
        zip_path = f"/tmp/export_{export_id}.zip"
        with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED) as zf:
            for file in os.listdir(tmp_dir):
                zf.write(f"{tmp_dir}/{file}", file)
        return zip_path
```

## 租户暂停与恢复

```python
class TenantLifecycleService:
    """租户生命周期：暂停、恢复、自动删除"""

    def suspend_tenant(self, tenant_id, reason):
        """暂停租户：禁用登录、停止计费、保留数据"""
        # 1. 禁用所有用户登录
        self.db.execute(
            "UPDATE users SET status = 'suspended' WHERE tenant_id = %s", tenant_id)
        # 2. 标记租户状态
        self.db.update("tenants",
            {"status": "suspended", "suspended_reason": reason,
             "suspended_at": now(), "auto_delete_at": now() + timedelta(days=90)},
            {"tenant_id": tenant_id})
        # 3. 停止计费
        self.billing_service.pause_subscription(tenant_id)
        # 4. 通知租户管理员
        self.notify_admin(tenant_id, f"租户已暂停: {reason}，数据将保留 90 天")

    def reactivate_tenant(self, tenant_id):
        """恢复租户"""
        tenant = self.db.get_tenant(tenant_id)
        if tenant["status"] != "suspended":
            raise InvalidOperationError("只能恢复暂停状态的租户")

        # 1. 恢复用户登录
        self.db.execute(
            "UPDATE users SET status = 'active' WHERE tenant_id = %s", tenant_id)
        # 2. 标记租户状态
        self.db.update("tenants",
            {"status": "active", "reactivated_at": now(), "auto_delete_at": None},
            {"tenant_id": tenant_id})
        # 3. 恢复计费
        self.billing_service.resume_subscription(tenant_id)

    def auto_delete_expired_tenants(self):
        """自动删除超过 90 天的暂停租户"""
        expired = self.db.query(
            "SELECT * FROM tenants WHERE status = 'suspended' "
            "AND auto_delete_at <= NOW()")
        for tenant in expired:
            self._delete_tenant_data(tenant["tenant_id"])
            self.db.update("tenants",
                {"status": "deleted", "deleted_at": now()},
                {"tenant_id": tenant["tenant_id"]})

    def _delete_tenant_data(self, tenant_id):
        """删除租户所有数据"""
        tenant = self.db.get_tenant(tenant_id)
        if tenant.get("schema_name"):
            self.db.execute(f"DROP SCHEMA IF EXISTS {tenant['schema_name']} CASCADE")
        self.db.delete("users", {"tenant_id": tenant_id})
        self.db.delete("tenant_roles", {"tenant_id": tenant_id})
        self.db.delete("tenant_settings", {"tenant_id": tenant_id})
```

## 异常场景补充

### 场景：租户数据库损坏

```
触发：MySQL 主从切换异常 → 租户 schema 部分表损坏
检测：
  1. 定期 CHECK TABLE 校验表完整性
  2. 查询返回 "Table is marked as crashed" → 告警
处理：
  1. REPAIR TABLE 尝试修复
  2. 修复失败 → 从最近备份恢复该 schema
  3. 恢复期间该租户只读
  4. 恢复后校验数据完整性
预防：每日自动备份 + 主从健康检查
```

### 场景：RBAC 缓存失效

```
触发：Redis 缓存中权限数据过期 → 用户权限判断错误
      → 普通用户获得了管理员权限
检测：
  1. 权限变更后立即失效缓存
  2. 定期校验：缓存权限 vs 数据库权限
  3. 发现不一致 → 告警 + 自动刷新缓存
处理：
  1. 立即刷新该租户的权限缓存
  2. 审计日志检查是否有越权操作
  3. 如有越权 → 通知安全管理员
预防：权限变更时强制刷新缓存 + 双重校验
```

### 场景：租户迁移回滚

```
触发：shared → schema 迁移完成后发现数据不一致
      → 需要回滚到 shared 模式
检测：
  1. 迁移后数据校验：行数 + checksum
  2. 不一致 → 标记迁移失败
处理：
  1. 停止双写 → 将流量切回 shared 模式
  2. 删除已创建的 schema
  3. 通知租户迁移失败
  4. 保留迁移日志用于排查
预防：迁移前完整备份 + 回滚演练
```

## 跨租户数据共享完整实现

```python
class CrossTenantSharingService:
    """跨租户数据共享（带 ACL 和审计）"""

    def request_sharing(self, source_tenant, target_tenant, resource_type, resource_id):
        """发起数据共享请求"""
        request_id = str(uuid4())
        self.db.insert("sharing_requests", {
            "request_id": request_id,
            "source_tenant": source_tenant,
            "target_tenant": target_tenant,
            "resource_type": resource_type,
            "resource_id": resource_id,
            "status": "pending_approval",
            "requested_at": now(),
            "requested_by": current_user()
        })
        # 通知源租户管理员审批
        self.notify_admin(source_tenant,
            f"收到数据共享请求: {resource_type}/{resource_id} → 租户 {target_tenant}")
        return request_id

    def approve_sharing(self, request_id, permissions):
        """审批共享请求"""
        self.db.update("sharing_requests",
            {"status": "approved", "permissions": json.dumps(permissions),
             "approved_at": now(), "approved_by": current_user()},
            {"request_id": request_id})

    def access_shared_resource(self, request_id, accessor_tenant):
        """访问共享资源"""
        sharing = self.db.get_sharing_request(request_id)
        if sharing["target_tenant"] != accessor_tenant:
            raise AccessDeniedError("非授权租户")
        if sharing["status"] != "approved":
            raise AccessDeniedError("共享未审批")

        # 审计日志
        self.db.insert("sharing_access_log", {
            "request_id": request_id,
            "accessor_tenant": accessor_tenant,
            "resource_type": sharing["resource_type"],
            "resource_id": sharing["resource_id"],
            "accessed_at": now(),
            "accessed_by": current_user()
        })

        # 返回资源数据（只读，按权限过滤）
        return self._get_resource_with_permissions(
            sharing["source_tenant"], sharing["resource_type"],
            sharing["resource_id"], sharing["permissions"])
```

## 租户性能隔离

```python
class TenantPerformanceIsolator:
    """租户性能隔离：防止单租户影响其他租户"""

    def get_connection_pool(self, tenant_id):
        """获取租户专属连接池（按等级配置）"""
        tier = self.db.get_tenant_tier(tenant_id)
        pool_config = {
            "shared": {"max_connections": 5, "timeout": 30},
            "schema": {"max_connections": 20, "timeout": 30},
            "dedicated": {"max_connections": 100, "timeout": 60},
        }
        return self.pool_manager.get_or_create(tenant_id, pool_config[tier])

    def check_rate_limit(self, tenant_id):
        """租户级限流"""
        tier = self.db.get_tenant_tier(tenant_id)
        limits = {
            "shared": 100,    # 100 RPM
            "schema": 1000,   # 1000 RPM
            "dedicated": 5000, # 5000 RPM
        }
        key = f"rate_limit:{tenant_id}"
        count = self.redis.incr(key)
        if count == 1:
            self.redis.expire(key, 60)
        return count <= limits[tier]
```

## 异常场景补充

### 场景：租户数据库损坏恢复

```
触发：MySQL 页损坏 → 查询返回 "Incorrect key file"
检测：
  1. 定期 CHECK TABLE → 发现损坏 → 告警
  2. 查询异常 → 自动标记该租户为 read_only
处理：
  1. REPAIR TABLE 尝试修复
  2. 修复失败 → 从最近备份恢复
  3. 恢复期间该租户只读 + 通知管理员
  4. 恢复后校验数据完整性
预防：每日备份 + 主从健康检查 + 页校验
```

### 场景：RBAC 缓存失效

```
触发：权限变更后缓存未及时刷新 → 用户获得旧权限
检测：
  1. 权限变更时主动刷新缓存
  2. 定期校验：缓存权限 vs 数据库权限
  3. 不一致 → 告警 + 自动刷新
处理：
  1. 立即刷新该租户权限缓存
  2. 审计日志检查是否有越权操作
预防：权限变更时强制刷新 + 双重校验
```

### 场景：租户迁移回滚

```
触发：shared → schema 迁移后数据不一致 → 需要回滚
处理：
  1. 停止双写 → 流量切回 shared 模式
  2. 删除已创建的 schema
  3. 通知租户迁移失败
  4. 保留迁移日志用于排查
预防：迁移前完整备份 + 回滚演练
```

## 租户自定义品牌系统

```python
class TenantBrandingService:
    """租户自定义品牌：Logo、颜色、域名、邮件模板"""

    def update_branding(self, tenant_id, branding_config):
        """更新租户品牌配置"""
        # 1. 上传 Logo 到 S3
        if branding_config.get("logo"):
            logo_url = self.s3_client.upload(
                branding_config["logo"],
                f"branding/{tenant_id}/logo.png")
            branding_config["logo_url"] = logo_url

        # 2. 配置自定义域名
        if branding_config.get("custom_domain"):
            self._configure_custom_domain(tenant_id, branding_config["custom_domain"])

        # 3. 更新邮件模板
        if branding_config.get("email_template"):
            self._update_email_template(tenant_id, branding_config["email_template"])

        # 4. 保存配置
        self.db.update("tenants",
            {"branding": json.dumps(branding_config)},
            {"tenant_id": tenant_id})

        # 5. 刷新 CDN 缓存
        self.cdn_client.invalidate(f"branding/{tenant_id}/*")

        return {"status": "updated", "tenant_id": tenant_id}

    def _configure_custom_domain(self, tenant_id, domain):
        """配置自定义域名"""
        # 1. DNS CNAME 验证
        expected_cname = f"{tenant_id}.app.example.com"
        actual = self.dns_client.get_cname(domain)
        if actual != expected_cname:
            raise DNSConfigError(
                f"请将 {domain} CNAME 指向 {expected_cname}")

        # 2. 申请 SSL 证书
        cert = self.acme_client.issue_certificate(domain)

        # 3. 配置反向代理
        self.proxy_client.add_route(domain, tenant_id)

        # 4. 保存域名配置
        self.db.insert("tenant_domains", {
            "tenant_id": tenant_id, "domain": domain,
            "ssl_cert": cert.pem, "ssl_expiry": cert.expiry_date,
            "configured_at": now()
        })
```

## 租户审计日志查询 API

```python
class TenantAuditLogService:
    """租户审计日志查询与导出"""

    def query_logs(self, tenant_id, filters):
        """查询审计日志"""
        query = "SELECT * FROM tenant_audit_log WHERE tenant_id = %s"
        params = [tenant_id]

        if filters.get("action"):
            query += " AND action = %s"
            params.append(filters["action"])
        if filters.get("user_id"):
            query += " AND user_id = %s"
            params.append(filters["user_id"])
        if filters.get("start_date"):
            query += " AND timestamp >= %s"
            params.append(filters["start_date"])
        if filters.get("end_date"):
            query += " AND timestamp <= %s"
            params.append(filters["end_date"])

        query += " ORDER BY timestamp DESC LIMIT 500"
        return self.db.query(query, *params)

    def export_to_csv(self, tenant_id, filters):
        """导出审计日志为 CSV"""
        logs = self.query_logs(tenant_id, filters)
        csv_path = f"/tmp/audit_export_{tenant_id}.csv"
        with open(csv_path, "w") as f:
            writer = csv.DictWriter(f, fieldnames=[
                "timestamp", "user_id", "action", "resource_type",
                "resource_id", "details", "ip_address"])
            writer.writeheader()
            writer.writerows(logs)
        return csv_path
```

## 异常场景补充

### 场景：自定义域名 DNS 配置错误

```
触发：租户将 CNAME 指向错误地址 → 自定义域名无法访问
检测：
  1. 域名可达性检查：每小时检查 custom_domain 是否可访问
  2. CNAME 不匹配 → 告警通知租户
处理：
  1. 回退到默认域名（tenant_id.app.example.com）
  2. 通知租户管理员修正 DNS 配置
  3. DNS 配置正确后自动恢复自定义域名
预防：域名配置时强制 DNS 验证 + 定期可达性检查
```

### 场景：审计日志存储配额超限

```
触发：审计日志超过配额 → 旧日志自动归档
检测：
  1. 日志量 > 配额 80% → 告警
  2. 日志量 > 配额 95% → 严重告警
处理：
  1. 90 天前的日志归档到 S3（从数据库移除）
  2. 保留最近 90 天的热日志在数据库
  3. 归档日志可通过 API 查询（从 S3 加载）
预防：分层存储 + 自动归档 + 配额监控
```

### 场景：Elasticsearch 单租户索引损坏

```
触发：Elasticsearch 分片损坏 → 某租户搜索全部失败
检测：
  1. 搜索请求返回 CorruptIndexException → 立即告警
  2. ES 集群健康检查：_cluster/health 中有 unassigned_shards
  3. 按租户粒度检查索引状态 → 定位损坏的租户索引
处理：
  1. 标记该租户搜索为降级模式（仅返回基础结果，无全文检索）
  2. 尝试 _cluster/reroute 自动恢复分片
  3. 恢复失败 → 从最近快照恢复该租户索引（不影响其他租户）
  4. 恢复期间其他租户搜索不受影响（索引隔离优势）
预防：ES 快照每日备份 + 分片副本数 >= 1 + 磁盘健康监控
```

### 场景：Feature Flag 评估超时

```
触发：Feature Flag 服务响应超时 → 页面加载卡住或功能异常
检测：
  1. Flag 评估 P99 > 500ms → 告警
  2. Flag 评估超时率 > 1% → 严重告警
处理：
  1. 评估超时 → 返回该租户的 flag 默认值（不阻塞页面）
  2. 本地缓存上次成功的 flag 值作为 fallback
  3. 记录超时详情用于排查（哪个 flag、哪个租户、耗时）
  4. 批量评估接口拆分为单 flag 评估（降低单次超时影响范围）
预防：评估超时熔断 + 本地缓存兜底 + Flag 数量限制（每租户 <= 200）
```

## 租户特性开关系统

```python
import hashlib
import time
import json
from enum import Enum
from dataclasses import dataclass, field
from typing import Optional, Dict, List, Any


class FlagType(Enum):
    BOOLEAN = "boolean"       # 开/关
    PERCENTAGE = "percentage" # 百分比灰度
    VARIANT = "variant"       # 多变体（A/B 测试）


class RolloutStrategy(Enum):
    ALL = "all"                     # 全量
    PERCENTAGE = "percentage"       # 百分比灰度
    TENANT_LIST = "tenant_list"     # 指定租户列表
    SCHEDULED = "scheduled"         # 定时开启


@dataclass
class FeatureFlag:
    """特性开关定义"""
    flag_key: str                   # 开关标识，如 "new_dashboard_v2"
    flag_type: FlagType             # 开关类型
    description: str                # 描述
    default_value: Any              # 全局默认值
    created_by: str
    created_at: float = field(default_factory=time.time)
    updated_at: float = field(default_factory=time.time)
    is_active: bool = True          # 是否启用


@dataclass
class TenantFlagOverride:
    """租户级别的开关覆盖"""
    tenant_id: int
    flag_key: str
    override_value: Any             # 覆盖值
    rollout_strategy: RolloutStrategy
    rollout_percentage: int = 100   # 灰度百分比，0-100
    tenant_whitelist: List[int] = field(default_factory=list)  # 租户白名单
    schedule_start: Optional[float] = None  # 定时开启时间
    schedule_end: Optional[float] = None    # 定时关闭时间
    priority: int = 0               # 优先级，数字越大优先级越高


class FeatureFlagService:
    """租户特性开关服务：支持租户级覆盖、渐进式灰度、A/B 测试"""

    def __init__(self, redis_client, db_client, cache_ttl=30):
        self.redis = redis_client
        self.db = db_client
        self.cache_ttl = cache_ttl  # 本地缓存秒数

    def evaluate(self, tenant_id: int, flag_key: str, user_id: int = None,
                 context: Dict = None) -> Any:
        """评估特性开关值（核心方法）"""
        context = context or {}

        # 1. 获取 flag 定义
        flag = self._get_flag_definition(flag_key)
        if not flag or not flag.is_active:
            return flag.default_value if flag else None

        # 2. 查找租户覆盖规则（优先级从高到低）
        overrides = self._get_tenant_overrides(tenant_id, flag_key)

        for override in sorted(overrides, key=lambda o: -o.priority):
            result = self._evaluate_override(tenant_id, override, user_id, context)
            if result is not None:
                return result

        # 3. 无覆盖规则 → 使用全局默认值
        return flag.default_value

    def _evaluate_override(self, tenant_id: int, override: TenantFlagOverride,
                           user_id: int, context: Dict) -> Optional[Any]:
        """评估单个覆盖规则"""
        # 定时策略：检查时间窗口
        if override.rollout_strategy == RolloutStrategy.SCHEDULED:
            now = time.time()
            if override.schedule_start and now < override.schedule_start:
                return None  # 未到开启时间，跳过此规则
            if override.schedule_end and now > override.schedule_end:
                return None  # 已过关闭时间，跳过此规则

        # 租户白名单策略
        if override.rollout_strategy == RolloutStrategy.TENANT_LIST:
            if tenant_id not in override.tenant_whitelist:
                return None  # 不在白名单，跳过

        # 百分比灰度策略
        if override.rollout_strategy == RolloutStrategy.PERCENTAGE:
            if not self._is_in_rollout(tenant_id, flag_key=override.flag_key,
                                        percentage=override.rollout_percentage,
                                        user_id=user_id):
                return None  # 未命中灰度，跳过

        # ALL 策略 → 直接返回覆盖值
        return override.override_value

    def _is_in_rollout(self, tenant_id: int, flag_key: str,
                       percentage: int, user_id: int = None) -> bool:
        """判断是否命中灰度（确定性哈希，同一用户结果稳定）"""
        # 使用 tenant_id + flag_key + user_id 的哈希 → 确保结果稳定
        hash_input = f"{tenant_id}:{flag_key}:{user_id or 'tenant'}"
        hash_val = int(hashlib.md5(hash_input.encode()).hexdigest(), 16)
        bucket = (hash_val % 10000) / 100  # 0.00 ~ 99.99
        return bucket < percentage

    def set_tenant_override(self, tenant_id: int, flag_key: str,
                            override_value: Any,
                            rollout_strategy: RolloutStrategy = RolloutStrategy.ALL,
                            rollout_percentage: int = 100,
                            tenant_whitelist: List[int] = None,
                            schedule_start: float = None,
                            schedule_end: float = None,
                            priority: int = 0):
        """设置租户级别的开关覆盖"""
        override = TenantFlagOverride(
            tenant_id=tenant_id,
            flag_key=flag_key,
            override_value=override_value,
            rollout_strategy=rollout_strategy,
            rollout_percentage=rollout_percentage,
            tenant_whitelist=tenant_whitelist or [],
            schedule_start=schedule_start,
            schedule_end=schedule_end,
            priority=priority,
        )

        # 持久化到数据库
        self.db.insert("tenant_flag_overrides", {
            "tenant_id": tenant_id,
            "flag_key": flag_key,
            "override_value": json.dumps(override_value),
            "rollout_strategy": rollout_strategy.value,
            "rollout_percentage": rollout_percentage,
            "tenant_whitelist": json.dumps(tenant_whitelist or []),
            "schedule_start": schedule_start,
            "schedule_end": schedule_end,
            "priority": priority,
        })

        # 刷新缓存
        self._invalidate_cache(tenant_id, flag_key)

    def gradual_rollout(self, tenant_id: int, flag_key: str,
                        target_value: Any, stages: List[int]):
        """渐进式灰度发布：按阶段逐步放量
        stages: 灰度百分比列表，如 [5, 10, 25, 50, 100]
        """
        for i, percentage in enumerate(stages):
            print(f"灰度阶段 {i+1}/{len(stages)}: {percentage}%")
            self.set_tenant_override(
                tenant_id=tenant_id,
                flag_key=flag_key,
                override_value=target_value,
                rollout_strategy=RolloutStrategy.PERCENTAGE,
                rollout_percentage=percentage,
            )
            # 实际场景中这里会等待观察期（监控指标），而非 sleep
            # 观察指标：错误率、P99延迟、业务转化率

    def ab_test(self, tenant_id: int, flag_key: str,
                variants: Dict[str, Any],
                traffic_split: Dict[str, int]) -> str:
        """A/B 测试：租户内用户分流到不同变体
        variants: {"control": "old_ui", "variant_a": "new_ui_v1", "variant_b": "new_ui_v2"}
        traffic_split: {"control": 50, "variant_a": 30, "variant_b": 20}
        """
        total = sum(traffic_split.values())
        if total != 100:
            raise ValueError(f"流量分配之和必须为 100，当前为 {total}")

        # 存储实验配置
        experiment = {
            "tenant_id": tenant_id,
            "flag_key": flag_key,
            "variants": variants,
            "traffic_split": traffic_split,
            "started_at": time.time(),
            "status": "running",
        }
        self.db.insert("ab_experiments", experiment)
        self._invalidate_cache(tenant_id, flag_key)
        return f"experiment_{tenant_id}_{flag_key}"

    def get_variant_for_user(self, tenant_id: int, flag_key: str,
                             user_id: int) -> str:
        """获取用户所属的实验变体"""
        experiment = self._get_experiment(tenant_id, flag_key)
        if not experiment or experiment["status"] != "running":
            return "control"

        # 确定性哈希分流
        hash_input = f"ab:{tenant_id}:{flag_key}:{user_id}"
        hash_val = int(hashlib.md5(hash_input.encode()).hexdigest(), 16)
        bucket = (hash_val % 10000) / 100  # 0.00 ~ 99.99

        cumulative = 0
        for variant_name, percentage in experiment["traffic_split"].items():
            cumulative += percentage
            if bucket < cumulative:
                return variant_name

        return "control"

    def _get_flag_definition(self, flag_key: str) -> Optional[FeatureFlag]:
        """获取 flag 定义（带本地缓存）"""
        cache_key = f"flag:def:{flag_key}"
        cached = self.redis.get(cache_key)
        if cached:
            data = json.loads(cached)
            return FeatureFlag(**data)

        row = self.db.query(
            "SELECT * FROM feature_flags WHERE flag_key = %s", flag_key)
        if not row:
            return None

        flag = FeatureFlag(
            flag_key=row["flag_key"],
            flag_type=FlagType(row["flag_type"]),
            description=row["description"],
            default_value=json.loads(row["default_value"]),
            created_by=row["created_by"],
            is_active=row["is_active"],
        )
        self.redis.setex(cache_key, self.cache_ttl, json.dumps({
            "flag_key": flag.flag_key,
            "flag_type": flag.flag_type.value,
            "description": flag.description,
            "default_value": flag.default_value,
            "created_by": flag.created_by,
            "is_active": flag.is_active,
        }))
        return flag

    def _get_tenant_overrides(self, tenant_id: int,
                               flag_key: str) -> List[TenantFlagOverride]:
        """获取租户覆盖规则（带缓存）"""
        cache_key = f"flag:override:{tenant_id}:{flag_key}"
        cached = self.redis.get(cache_key)
        if cached:
            return [TenantFlagOverride(**o) for o in json.loads(cached)]

        rows = self.db.query(
            "SELECT * FROM tenant_flag_overrides "
            "WHERE tenant_id = %s AND flag_key = %s ORDER BY priority DESC",
            tenant_id, flag_key)

        overrides = [TenantFlagOverride(
            tenant_id=r["tenant_id"],
            flag_key=r["flag_key"],
            override_value=json.loads(r["override_value"]),
            rollout_strategy=RolloutStrategy(r["rollout_strategy"]),
            rollout_percentage=r["rollout_percentage"],
            tenant_whitelist=json.loads(r["tenant_whitelist"]),
            schedule_start=r["schedule_start"],
            schedule_end=r["schedule_end"],
            priority=r["priority"],
        ) for r in rows]

        self.redis.setex(cache_key, self.cache_ttl,
                         json.dumps([{
                             "tenant_id": o.tenant_id,
                             "flag_key": o.flag_key,
                             "override_value": o.override_value,
                             "rollout_strategy": o.rollout_strategy.value,
                             "rollout_percentage": o.rollout_percentage,
                             "tenant_whitelist": o.tenant_whitelist,
                             "schedule_start": o.schedule_start,
                             "schedule_end": o.schedule_end,
                             "priority": o.priority,
                         } for o in overrides]))
        return overrides

    def _invalidate_cache(self, tenant_id: int, flag_key: str):
        """刷新缓存"""
        self.redis.delete(f"flag:override:{tenant_id}:{flag_key}")
        self.redis.delete(f"flag:def:{flag_key}")

    def _get_experiment(self, tenant_id: int, flag_key: str) -> Optional[Dict]:
        """获取 A/B 实验配置"""
        row = self.db.query(
            "SELECT * FROM ab_experiments "
            "WHERE tenant_id = %s AND flag_key = %s AND status = 'running'",
            tenant_id, flag_key)
        return row
```

## 租户 API 限流系统（Redis 滑动窗口）

```python
import time
import json
from typing import Dict, Tuple, Optional
from dataclasses import dataclass


@dataclass
class RateLimitTier:
    """租户等级对应的限流配置"""
    tier_name: str
    rpm: int               # 每分钟请求数
    burst_rpm: int         # 突发允许的 RPM（短时超额）
    burst_window_sec: int  # 突发窗口（秒）
    daily_quota: int       # 每日配额


# 租户等级限流配置
TIER_LIMITS = {
    "free":     RateLimitTier("free",     rpm=60,   burst_rpm=120,   burst_window_sec=10, daily_quota=10000),
    "basic":    RateLimitTier("basic",    rpm=300,  burst_rpm=600,   burst_window_sec=10, daily_quota=50000),
    "pro":      RateLimitTier("pro",      rpm=1000, burst_rpm=2000,  burst_window_sec=10, daily_quota=200000),
    "enterprise": RateLimitTier("enterprise", rpm=5000, burst_rpm=10000, burst_window_sec=10, daily_quota=2000000),
}


class TenantRateLimiter:
    """租户 API 限流：滑动窗口 + 突发允许 + 分级配额"""

    def __init__(self, redis_client, db_client):
        self.redis = redis_client
        self.db = db_client

    def check_rate_limit(self, tenant_id: int, api_path: str = None) -> Dict:
        """检查租户是否超过限流阈值
        返回: {
            "allowed": bool,
            "remaining": int,         # 当前窗口剩余配额
            "retry_after_ms": int,    # 需要等待的毫秒数（超限时）
            "daily_remaining": int,   # 今日剩余配额
        }
        """
        tier = self._get_tenant_tier(tenant_id)

        # 1. 滑动窗口限流（核心）
        window_result = self._sliding_window_check(tenant_id, tier)

        # 2. 突发允许检查
        if not window_result["allowed"]:
            burst_result = self._burst_check(tenant_id, tier)
            if burst_result["allowed"]:
                return burst_result

        # 3. 每日配额检查
        daily_remaining = self._daily_quota_check(tenant_id, tier)

        return {
            "allowed": window_result["allowed"] and daily_remaining > 0,
            "remaining": min(window_result["remaining"], daily_remaining),
            "retry_after_ms": window_result.get("retry_after_ms", 0),
            "daily_remaining": daily_remaining,
        }

    def _sliding_window_check(self, tenant_id: int,
                               tier: RateLimitTier) -> Dict:
        """Redis 滑动窗口限流（Sorted Set 实现）
        原理：以时间戳为 score，每个请求添加一条记录，
        统计窗口内请求数，超过阈值则拒绝。
        """
        now = time.time()
        window_sec = 60  # 1 分钟窗口
        window_start = now - window_sec
        key = f"ratelimit:sliding:{tenant_id}"

        # Lua 脚本保证原子性：清理过期 + 计数 + 添加
        lua_script = """
        local key = KEYS[1]
        local window_start = tonumber(ARGV[1])
        local now = tonumber(ARGV[2])
        local max_requests = tonumber(ARGV[3])
        local ttl = tonumber(ARGV[4])

        -- 清理窗口外的旧记录
        redis.call('ZREMRANGEBYSCORE', key, '-inf', window_start)

        -- 统计窗口内请求数
        local count = redis.call('ZCARD', key)

        if count < max_requests then
            -- 未超限：添加当前请求
            redis.call('ZADD', key, now, now .. ':' .. math.random(1000000))
            redis.call('EXPIRE', key, ttl)
            return {1, max_requests - count - 1, 0}
        else
            -- 超限：计算需要等待的时间
            local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
            local retry_after = 0
            if #oldest > 0 then
                retry_after = math.ceil((oldest[2] + 60 - now) * 1000)
            end
            return {0, 0, retry_after}
        end
        """

        result = self.redis.eval(
            lua_script, 1, key, window_start, now, tier.rpm, window_sec + 10
        )

        allowed = bool(result[0])
        remaining = int(result[1])
        retry_after_ms = int(result[2])

        return {
            "allowed": allowed,
            "remaining": remaining,
            "retry_after_ms": retry_after_ms,
        }

    def _burst_check(self, tenant_id: int, tier: RateLimitTier) -> Dict:
        """突发流量允许：短时超额（令牌桶补充）"""
        now = time.time()
        burst_key = f"ratelimit:burst:{tenant_id}"

        # 检查突发窗口内的请求数
        count = self.redis.incr(burst_key)
        if count == 1:
            self.redis.expire(burst_key, tier.burst_window_sec)

        allowed = count <= tier.burst_rpm

        return {
            "allowed": allowed,
            "remaining": max(0, tier.burst_rpm - count),
            "retry_after_ms": tier.burst_window_sec * 1000 if not allowed else 0,
            "burst": True,  # 标记为突发流量
        }

    def _daily_quota_check(self, tenant_id: int, tier: RateLimitTier) -> int:
        """每日配额检查"""
        today = time.strftime("%Y-%m-%d")
        quota_key = f"ratelimit:daily:{tenant_id}:{today}"

        used = self.redis.incr(quota_key)
        if used == 1:
            self.redis.expire(quota_key, 86400 * 2)  # 2 天过期

        return max(0, tier.daily_quota - used)

    def _get_tenant_tier(self, tenant_id: int) -> RateLimitTier:
        """获取租户等级"""
        cache_key = f"tenant:tier:{tenant_id}"
        cached = self.redis.get(cache_key)
        if cached:
            tier_name = cached.decode() if isinstance(cached, bytes) else cached
            return TIER_LIMITS.get(tier_name, TIER_LIMITS["free"])

        row = self.db.query(
            "SELECT tier FROM tenants WHERE tenant_id = %s", tenant_id)
        tier_name = row["tier"] if row else "free"
        self.redis.setex(cache_key, 300, tier_name)  # 5 分钟缓存
        return TIER_LIMITS.get(tier_name, TIER_LIMITS["free"])

    def get_quota_dashboard(self, tenant_id: int) -> Dict:
        """配额使用看板数据"""
        tier = self._get_tenant_tier(tenant_id)
        today = time.strftime("%Y-%m-%d")

        # 当前窗口使用量
        now = time.time()
        window_start = now - 60
        sliding_key = f"ratelimit:sliding:{tenant_id}"
        current_rpm = self.redis.zcount(sliding_key, window_start, now)

        # 今日使用量
        daily_key = f"ratelimit:daily:{tenant_id}:{today}"
        daily_used = int(self.redis.get(daily_key) or 0)

        # 近 7 天趋势
        daily_trend = []
        for i in range(7):
            d = time.strftime("%Y-%m-%d", time.localtime(now - i * 86400))
            d_key = f"ratelimit:daily:{tenant_id}:{d}"
            d_used = int(self.redis.get(d_key) or 0)
            daily_trend.append({"date": d, "used": d_used, "quota": tier.daily_quota})

        # 突发使用统计
        burst_key = f"ratelimit:burst:{tenant_id}"
        burst_used = int(self.redis.get(burst_key) or 0)

        return {
            "tenant_id": tenant_id,
            "tier": tier.tier_name,
            "current_rpm": current_rpm,
            "rpm_limit": tier.rpm,
            "burst_rpm_limit": tier.burst_rpm,
            "burst_used": burst_used,
            "daily_used": daily_used,
            "daily_quota": tier.daily_quota,
            "daily_remaining": max(0, tier.daily_quota - daily_used),
            "daily_usage_pct": round(daily_used / tier.daily_quota * 100, 1),
            "daily_trend": daily_trend,
        }
```

## 多租户搜索系统（Elasticsearch）

```python
import json
from typing import Dict, List, Optional
from dataclasses import dataclass


class IndexStrategy(Enum):
    PER_TENANT = "per_tenant"    # 每租户独立索引
    SHARED = "shared"            # 共享索引 + routing key


@dataclass
class SearchConfig:
    """租户搜索配置"""
    tenant_id: int
    index_strategy: IndexStrategy
    boost_fields: Dict[str, float]   # 字段权重调整，如 {"title": 2.0, "content": 1.0}
    analyzer: str = "ik_max_word"    # 分词器
    max_results: int = 50
    highlight_enabled: bool = True


class MultiTenantSearchService:
    """多租户搜索：支持独立索引和共享索引两种模式"""

    def __init__(self, es_client, db_client, redis_client):
        self.es = es_client
        self.db = db_client
        self.redis = redis_client

    # ============ 索引管理 ============

    def create_tenant_index(self, tenant_id: int, strategy: IndexStrategy):
        """为租户创建搜索索引"""
        if strategy == IndexStrategy.PER_TENANT:
            index_name = f"tenant_{tenant_id}"
            self.es.indices.create(index=index_name, body={
                "settings": {
                    "number_of_shards": 2,
                    "number_of_replicas": 1,
                    "analysis": {
                        "analyzer": {
                            "default": {
                                "type": "ik_max_word",
                            }
                        }
                    }
                },
                "mappings": {
                    "properties": {
                        "tenant_id":    {"type": "integer"},
                        "title":        {"type": "text", "analyzer": "ik_max_word"},
                        "content":      {"type": "text", "analyzer": "ik_max_word"},
                        "category":     {"type": "keyword"},
                        "tags":         {"type": "keyword"},
                        "created_at":   {"type": "date"},
                        "updated_at":   {"type": "date"},
                    }
                }
            })
        elif strategy == IndexStrategy.SHARED:
            # 共享索引只需创建一次，后续租户使用 routing 写入
            index_name = "shared_tenant_index"
            if not self.es.indices.exists(index=index_name):
                self.es.indices.create(index=index_name, body={
                    "settings": {
                        "number_of_shards": 10,   # 多分片支持大数据量
                        "number_of_replicas": 1,
                    },
                    "mappings": {
                        "properties": {
                            "tenant_id":    {"type": "integer"},
                            "title":        {"type": "text", "analyzer": "ik_max_word"},
                            "content":      {"type": "text", "analyzer": "ik_max_word"},
                            "category":     {"type": "keyword"},
                            "tags":         {"type": "keyword"},
                            "created_at":   {"type": "date"},
                            "updated_at":   {"type": "date"},
                        }
                    }
                })

    def index_document(self, tenant_id: int, doc: Dict, strategy: IndexStrategy):
        """索引文档"""
        doc["tenant_id"] = tenant_id

        if strategy == IndexStrategy.PER_TENANT:
            index_name = f"tenant_{tenant_id}"
            self.es.index(index=index_name, body=doc, id=doc.get("id"))
        elif strategy == IndexStrategy.SHARED:
            # 共享索引：使用 tenant_id 作为 routing key
            # 确保同一租户的文档落在同一分片，提升查询性能
            self.es.index(
                index="shared_tenant_index",
                body=doc,
                id=doc.get("id"),
                routing=str(tenant_id),
            )

    # ============ 搜索 ============

    def search(self, tenant_id: int, query: str, filters: Dict = None,
               page: int = 1, page_size: int = 20) -> Dict:
        """租户搜索（自动隔离数据）"""
        config = self._get_search_config(tenant_id)
        filters = filters or {}

        if config.index_strategy == IndexStrategy.PER_TENANT:
            index_name = f"tenant_{tenant_id}"
            routing = None
        else:
            index_name = "shared_tenant_index"
            routing = str(tenant_id)

        # 构建查询体
        es_query = self._build_query(tenant_id, query, filters, config, page, page_size)

        # 执行搜索
        params = {"index": index_name, "body": es_query}
        if routing:
            params["routing"] = routing

        result = self.es.search(**params)

        # 解析结果
        return self._parse_results(result, config)

    def _build_query(self, tenant_id: int, query: str, filters: Dict,
                     config: SearchConfig, page: int, page_size: int) -> Dict:
        """构建 ES 查询 DSL"""
        must_clauses = []

        # 共享索引必须加 tenant_id 过滤
        if config.index_strategy == IndexStrategy.SHARED:
            must_clauses.append({"term": {"tenant_id": tenant_id}})

        # 全文检索（多字段匹配，带租户自定义权重）
        if query:
            multi_match = {
                "multi_match": {
                    "query": query,
                    "fields": self._build_field_boosts(config),
                    "type": "best_fields",
                    "fuzziness": "AUTO",
                }
            }
            must_clauses.append(multi_match)

        # 过滤条件
        filter_clauses = []
        if filters.get("category"):
            filter_clauses.append({"term": {"category": filters["category"]}})
        if filters.get("tags"):
            filter_clauses.append({"terms": {"tags": filters["tags"]}})
        if filters.get("date_range"):
            filter_clauses.append({"range": {"created_at": {
                "gte": filters["date_range"][0],
                "lte": filters["date_range"][1],
            }}})

        es_query = {
            "query": {
                "bool": {
                    "must": must_clauses,
                    "filter": filter_clauses,
                }
            },
            "from": (page - 1) * page_size,
            "size": page_size,
            "sort": [
                "_score",
                {"created_at": {"order": "desc"}},
            ],
        }

        # 高亮
        if config.highlight_enabled and query:
            es_query["highlight"] = {
                "fields": {
                    "title":   {"fragment_size": 100, "number_of_fragments": 1},
                    "content": {"fragment_size": 200, "number_of_fragments": 3},
                }
            }

        return es_query

    def _build_field_boosts(self, config: SearchConfig) -> List[str]:
        """构建字段权重列表"""
        boosts = config.boost_fields or {"title": 2.0, "content": 1.0}
        return [f"{field}^{weight}" for field, weight in boosts.items()]

    def _parse_results(self, es_result: Dict, config: SearchConfig) -> Dict:
        """解析搜索结果"""
        hits = es_result.get("hits", {}).get("hits", [])
        total = es_result.get("hits", {}).get("total", {}).get("value", 0)

        results = []
        for hit in hits:
            item = hit["_source"]
            item["_score"] = hit["_score"]
            if "highlight" in hit:
                item["_highlight"] = hit["highlight"]
            results.append(item)

        return {
            "total": total,
            "results": results,
            "max_results": config.max_results,
        }

    def _get_search_config(self, tenant_id: int) -> SearchConfig:
        """获取租户搜索配置"""
        cache_key = f"search:config:{tenant_id}"
        cached = self.redis.get(cache_key)
        if cached:
            data = json.loads(cached)
            return SearchConfig(**data)

        row = self.db.query(
            "SELECT * FROM tenant_search_config WHERE tenant_id = %s",
            tenant_id)

        if row:
            config = SearchConfig(
                tenant_id=row["tenant_id"],
                index_strategy=IndexStrategy(row["index_strategy"]),
                boost_fields=json.loads(row["boost_fields"]),
                analyzer=row.get("analyzer", "ik_max_word"),
                max_results=row.get("max_results", 50),
                highlight_enabled=row.get("highlight_enabled", True),
            )
        else:
            # 默认：小租户用共享索引
            config = SearchConfig(
                tenant_id=tenant_id,
                index_strategy=IndexStrategy.SHARED,
                boost_fields={"title": 2.0, "content": 1.0},
            )

        self.redis.setex(cache_key, 300, json.dumps({
            "tenant_id": config.tenant_id,
            "index_strategy": config.index_strategy.value,
            "boost_fields": config.boost_fields,
            "analyzer": config.analyzer,
            "max_results": config.max_results,
            "highlight_enabled": config.highlight_enabled,
        }))
        return config

    def rebuild_tenant_index(self, tenant_id: int):
        """重建租户索引（索引损坏时使用）"""
        config = self._get_search_config(tenant_id)

        if config.index_strategy == IndexStrategy.PER_TENANT:
            index_name = f"tenant_{tenant_id}"
            # 删除旧索引
            if self.es.indices.exists(index=index_name):
                self.es.indices.delete(index=index_name)
            # 重新创建
            self.create_tenant_index(tenant_id, IndexStrategy.PER_TENANT)
            # 从数据库全量同步
            self._full_reindex_from_db(tenant_id, index_name)
        else:
            # 共享索引：删除该租户的文档后重新索引
            self.es.delete_by_query(
                index="shared_tenant_index",
                body={"query": {"term": {"tenant_id": tenant_id}}},
                routing=str(tenant_id),
            )
            self._full_reindex_from_db(tenant_id, "shared_tenant_index")

    def _full_reindex_from_db(self, tenant_id: int, index_name: str):
        """从数据库全量重建索引"""
        batch_size = 500
        offset = 0
        while True:
            rows = self.db.query(
                "SELECT * FROM documents WHERE tenant_id = %s "
                "ORDER BY id LIMIT %s OFFSET %s",
                tenant_id, batch_size, offset)
            if not rows:
                break

            for row in rows:
                doc = {
                    "id": row["id"],
                    "tenant_id": tenant_id,
                    "title": row["title"],
                    "content": row["content"],
                    "category": row["category"],
                    "tags": json.loads(row.get("tags", "[]")),
                    "created_at": row["created_at"].isoformat(),
                    "updated_at": row["updated_at"].isoformat(),
                }
                routing = str(tenant_id) if "shared" in index_name else None
                params = {"index": index_name, "body": doc, "id": row["id"]}
                if routing:
                    params["routing"] = routing
                self.es.index(**params)

            offset += batch_size
```

## 租户数据迁移完整实现

```python
class TenantDataMigrationService:
    """租户数据迁移：隔离级别变更 + 跨实例迁移"""

    def migrate_isolation_level(self, tenant_id, from_level, to_level):
        """迁移租户隔离级别（如共享schema→独立schema）"""
        migration_id = str(uuid4())
        self.db.insert("tenant_migrations", {
            "migration_id": migration_id,
            "tenant_id": tenant_id,
            "from_level": from_level,
            "to_level": to_level,
            "status": "in_progress",
            "started_at": now()
        })

        try:
            if from_level == "shared_schema" and to_level == "separate_schema":
                # 1. 创建新 schema
                schema_name = f"tenant_{tenant_id.replace('-', '_')}"
                self.db.execute(f"CREATE SCHEMA {schema_name}")

                # 2. 复制表结构
                tables = self.db.query(
                    "SELECT tablename FROM pg_tables WHERE schemaname = 'public'")
                for table in tables:
                    self.db.execute(
                        f"CREATE TABLE {schema_name}.{table['tablename']} "
                        f"(LIKE public.{table['tablename']} INCLUDING ALL)")

                # 3. 迁移数据（使用 INSERT...SELECT 分批）
                batch_size = 10000
                for table in tables:
                    offset = 0
                    while True:
                        rows = self.db.query(
                            f"SELECT * FROM public.{table['tablename']} "
                            f"WHERE tenant_id = %s LIMIT %s OFFSET %s",
                            tenant_id, batch_size, offset)
                        if not rows:
                            break
                        for row in rows:
                            cols = ", ".join(row.keys())
                            vals = ", ".join([f"%({k})s" for k in row.keys()])
                            self.db.execute(
                                f"INSERT INTO {schema_name}.{table['tablename']} ({cols}) "
                                f"VALUES ({vals})", row)
                        offset += batch_size

                # 4. 验证数据一致性
                self._verify_migration(tenant_id, schema_name)

                # 5. 切换路由（原子操作）
                self.db.update("tenants",
                    {"schema_name": schema_name, "isolation_level": to_level},
                    {"id": tenant_id})

                # 6. 清理旧数据
                self.db.execute(
                    f"DELETE FROM public.* WHERE tenant_id = %s", tenant_id)

            self.db.update("tenant_migrations",
                {"status": "completed", "completed_at": now()},
                {"migration_id": migration_id})

        except Exception as e:
            self.db.update("tenant_migrations",
                {"status": "failed", "error": str(e)},
                {"migration_id": migration_id})
            raise

    def _verify_migration(self, tenant_id, new_schema):
        """验证迁移数据一致性"""
        tables = ["users", "orders", "products"]
        for table in tables:
            old_count = self.db.query_one(
                f"SELECT COUNT(*) as cnt FROM public.{table} "
                f"WHERE tenant_id = %s", tenant_id)["cnt"]
            new_count = self.db.query_one(
                f"SELECT COUNT(*) as cnt FROM {new_schema}.{table}")["cnt"]
            if old_count != new_count:
                raise MigrationVerificationError(
                    f"{table}: old={old_count} new={new_count}")
```

## 租户自定义品牌

```python
class TenantBrandingService:
    """租户自定义品牌：Logo + 主题色 + 域名"""

    def update_branding(self, tenant_id, branding):
        """更新品牌配置"""
        # 1. 上传 Logo
        if branding.get("logo"):
            logo_url = self.storage.upload(
                f"branding/{tenant_id}/logo.png", branding["logo"])

        # 2. 验证主题色
        if branding.get("primary_color"):
            if not self._is_valid_color(branding["primary_color"]):
                raise InvalidColorError("无效的主题色")

        # 3. 配置自定义域名
        if branding.get("custom_domain"):
            self._configure_custom_domain(tenant_id, branding["custom_domain"])

        # 4. 更新品牌配置
        self.db.update("tenant_branding", {
            "logo_url": logo_url if branding.get("logo") else None,
            "primary_color": branding.get("primary_color"),
            "custom_domain": branding.get("custom_domain"),
            "login_page_html": branding.get("login_page_html"),
            "email_template_id": branding.get("email_template_id"),
            "updated_at": now()
        }, {"tenant_id": tenant_id})

        # 5. 刷新 CDN 缓存
        self.cdn.purge(f"branding/{tenant_id}/*")

    def _configure_custom_domain(self, tenant_id, domain):
        """配置自定义域名"""
        # 1. 验证域名所有权（CNAME 指向我们的域名）
        cname = self.dns_client.get_cname(domain)
        if cname != "app.saas-platform.com":
            raise DomainVerificationError(
                f"请将 {domain} 的 CNAME 指向 app.saas-platform.com")

        # 2. 申请 SSL 证书
        cert = self.certificate_manager.issue(domain)

        # 3. 配置反向代理
        self.nginx_client.add_server({
            "server_name": domain,
            "ssl_certificate": cert["cert_path"],
            "ssl_certificate_key": cert["key_path"],
            "proxy_pass": f"http://backend/tenant/{tenant_id}"
        })
```

## 异常场景补充

### 场景：租户数据迁移中断

```
触发：迁移过程中网络中断 → 数据部分迁移 → 租户数据不一致
检测：
  1. 迁移任务超时 > 1 小时 → 可能中断
  2. 源表和目标表行数不匹配 → 不一致
处理：
  1. 暂停迁移 → 检查已迁移数据量
  2. 从断点继续（基于 offset 记录）
  3. 如果无法继续 → 回滚（删除目标 schema 数据）
  4. 重新启动迁移
预防：断点续传 + 迁移前备份 + 分批提交
```

### 场景：自定义域名 DNS 劫持

```
触发：攻击者将租户自定义域名指向恶意服务器 → 用户数据泄露
检测：
  1. 定期检查自定义域名的 DNS 解析
  2. 解析结果不是我们的 IP → DNS 劫持
处理：
  1. 立即禁用该自定义域名 → 回退到默认域名
  2. 通知租户管理员
  3. 协助租户修复 DNS 配置
预防：DNS 监控 + CNAME 验证 + 域名锁定
```

## 租户配额与资源限制完整实现

```python
class TenantQuotaService:
    """租户配额管理：资源限制 + 超限处理"""

    QUOTA_TYPES = {
        "max_users": {"default": 100, "unit": "人"},
        "max_storage_gb": {"default": 10, "unit": "GB"},
        "max_api_calls_per_day": {"default": 10000, "unit": "次/天"},
        "max_concurrent_sessions": {"default": 50, "unit": "个"},
        "max_custom_domains": {"default": 1, "unit": "个"},
    }

    def check_quota(self, tenant_id, quota_type, requested=1):
        """检查配额"""
        plan = self.db.get_tenant_plan(tenant_id)
        quota_limit = plan.get(quota_type, self.QUOTA_TYPES[quota_type]["default"])

        # 获取当前使用量
        current_usage = self._get_usage(tenant_id, quota_type)

        if current_usage + requested > quota_limit:
            return {
                "allowed": False,
                "current": current_usage,
                "limit": quota_limit,
                "requested": requested,
                "message": f"配额不足: 当前 {current_usage}/{quota_limit}，需要 {requested}"
            }

        return {"allowed": True, "current": current_usage, "limit": quota_limit}

    def handle_over_quota(self, tenant_id, quota_type):
        """处理超配额"""
        plan = self.db.get_tenant_plan(tenant_id)

        if plan["over_quota_policy"] == "hard_limit":
            # 硬限制：直接拒绝
            return {"action": "reject", "message": "已超过配额限制"}
        elif plan["over_quota_policy"] == "soft_limit":
            # 软限制：允许但告警 + 按量计费
            overage = self._get_overage(tenant_id, quota_type)
            self.db.insert("quota_overages", {
                "tenant_id": tenant_id,
                "quota_type": quota_type,
                "overage_amount": overage,
                "charge": round(overage * self._get_overage_rate(quota_type), 2),
                "timestamp": now()
            })
            self.notify_tenant_admin(tenant_id,
                f"您的 {quota_type} 已超过配额，超出部分将按量计费")
            return {"action": "allow_with_charge", "overage": overage}

    def _get_usage(self, tenant_id, quota_type):
        """获取当前使用量"""
        if quota_type == "max_users":
            return self.db.count("users", tenant_id=tenant_id)
        elif quota_type == "max_storage_gb":
            return self.db.query_one(
                "SELECT SUM(size_bytes) / 1073741824 as gb "
                "FROM tenant_storage WHERE tenant_id = %s",
                tenant_id)["gb"] or 0
        elif quota_type == "max_api_calls_per_day":
            key = f"api_calls:{tenant_id}:{now().strftime('%Y%m%d')}"
            return int(self.redis.get(key) or 0)
        elif quota_type == "max_concurrent_sessions":
            return self.redis.scard(f"sessions:{tenant_id}")

    def get_usage_dashboard(self, tenant_id):
        """配额使用看板"""
        result = {}
        for quota_type, config in self.QUOTA_TYPES.items():
            usage = self._get_usage(tenant_id, quota_type)
            limit = self.db.get_tenant_plan(tenant_id).get(
                quota_type, config["default"])
            result[quota_type] = {
                "usage": usage,
                "limit": limit,
                "percentage": round(usage / limit * 100, 1) if limit > 0 else 0,
                "unit": config["unit"],
                "status": "critical" if usage / limit > 0.9 else
                         "warning" if usage / limit > 0.7 else "ok"
            }
        return result
```

## 租户运维隔离

```python
class TenantOperationsIsolation:
    """租户运维隔离：独立监控 + 告警 + 日志"""

    def get_tenant_metrics(self, tenant_id):
        """获取租户级指标"""
        return {
            "api_latency_p99": self._get_api_latency(tenant_id),
            "error_rate_1h": self._get_error_rate(tenant_id),
            "active_users_today": self._get_active_users(tenant_id),
            "api_calls_today": self._get_api_calls(tenant_id),
            "storage_usage_gb": self._get_storage(tenant_id),
            "last_incident": self._get_last_incident(tenant_id)
        }

    def create_tenant_alert_rule(self, tenant_id, metric, threshold,
                                  comparison="gt", actions=None):
        """创建租户级告警规则"""
        rule_id = str(uuid4())
        self.db.insert("tenant_alert_rules", {
            "rule_id": rule_id, "tenant_id": tenant_id,
            "metric": metric, "threshold": threshold,
            "comparison": comparison,  # gt / lt / gte / lte
            "actions": json.dumps(actions or [{"type": "notify_admin"}]),
            "enabled": True, "created_at": now()
        })

        # 注册到告警系统
        self.alert_manager.register({
            "rule_id": rule_id,
            "check_fn": lambda: self._evaluate_rule(tenant_id, metric, threshold, comparison),
            "action_fn": lambda: self._execute_actions(tenant_id, actions or []),
            "interval_seconds": 60
        })
        return rule_id

    def get_tenant_audit_log(self, tenant_id, page=1, size=50):
        """获取租户审计日志"""
        return self.db.query(
            "SELECT * FROM audit_log WHERE tenant_id = %s "
            "ORDER BY timestamp DESC LIMIT %s OFFSET %s",
            tenant_id, size, (page - 1) * size)
```

## 异常场景补充

### 场景：配额超限导致租户功能异常

```
触发：租户 API 调用超过日配额 → 所有 API 返回 429 → 租户业务中断
检测：
  1. 429 错误率飙升 → 告警
  2. 租户投诉"系统不可用" → 配额问题
处理：
  1. 检查是否为正常业务增长 → 升级套餐
  2. 异常流量 → 检查是否被攻击
  3. 临时增加配额（24 小时缓冲）
预防：配额使用率 80% 时预警 + 自动升级建议 + 临时缓冲
```

### 场景：租户运维监控误报

```
触发：某租户 P99 延迟突增 → 告警 → 实际是该租户自身代码问题
检测：
  1. 延迟增加仅影响单个租户 → 非平台问题
  2. 其他租户延迟正常 → 租户自身问题
处理：
  1. 通知租户检查自身代码
  2. 提供慢查询分析工具
  3. 如需协助 → 专业服务团队
预防：区分平台问题 vs 租户问题 + 租户自服务诊断工具
```

## 租户计费与计量系统完整实现

```python
class TenantBillingService:
    """租户计费与计量：使用量追踪 + 账单生成 + 支付"""

    USAGE_METRICS = {
        "api_calls": {"unit": "次", "rate_per_unit": 0.0001},
        "storage_gb": {"unit": "GB", "rate_per_unit": 0.50},
        "compute_hours": {"unit": "小时", "rate_per_unit": 0.10},
    }

    def record_usage(self, tenant_id, metric_type, amount):
        """记录使用量（Redis 实时计数）"""
        key = f"usage:{tenant_id}:{metric_type}:{now().strftime('%Y%m%d')}"
        self.redis.incrbyfloat(key, amount)
        # 设置 TTL（保留 35 天）
        self.redis.expire(key, 86400 * 35)

    def generate_invoice(self, tenant_id, billing_period):
        """生成账单"""
        plan = self.db.get_tenant_plan(tenant_id)

        # 1. 计算使用量
        usage = self._calculate_usage(tenant_id, billing_period)

        # 2. 计算费用
        line_items = []
        base_charge = plan["monthly_price"]

        line_items.append({
            "description": f"{plan['name']} 基础费用",
            "amount": base_charge
        })

        for metric, data in usage.items():
            included = plan.get(f"included_{metric}", 0)
            overage = max(0, data["total"] - included)
            if overage > 0:
                rate = self.USAGE_METRICS[metric]["rate_per_unit"]
                charge = round(overage * rate, 2)
                line_items.append({
                    "description": f"{metric} 超额 ({overage:.0f} {self.USAGE_METRICS[metric]['unit']})",
                    "amount": charge,
                    "overage": overage,
                    "rate": rate
                })

        total = round(sum(item["amount"] for item in line_items), 2)

        # 3. 创建发票
        invoice_id = str(uuid4())
        self.db.insert("invoices", {
            "invoice_id": invoice_id,
            "tenant_id": tenant_id,
            "billing_period": billing_period,
            "line_items": json.dumps(line_items),
            "total_amount": total,
            "status": "pending",
            "due_date": now() + timedelta(days=30),
            "created_at": now()
        })

        return {"invoice_id": invoice_id, "total": total, "items": line_items}

    def process_payment(self, invoice_id, payment_method_id):
        """处理支付"""
        invoice = self.db.get_invoice(invoice_id)

        try:
            result = self.payment_gateway.charge(
                payment_method=payment_method_id,
                amount=invoice["total_amount"],
                currency="CNY",
                description=f"账单 {invoice_id}")

            self.db.update("invoices",
                {"status": "paid", "paid_at": now(),
                 "payment_reference": result["transaction_id"]},
                {"invoice_id": invoice_id})

        except PaymentFailedError:
            self._start_dunning(invoice_id)

    def _start_dunning(self, invoice_id):
        """催收流程"""
        dunning_stages = [
            {"days_overdue": 3, "action": "reminder_email"},
            {"days_overdue": 7, "action": "warning_email"},
            {"days_overdue": 14, "action": "suspend_service"},
            {"days_overdue": 30, "action": "terminate_account"},
        ]

        for stage in dunning_stages:
            self.scheduler.schedule(
                run_date=now() + timedelta(days=stage["days_overdue"]),
                task=self._execute_dunning_action,
                args={"invoice_id": invoice_id, "action": stage["action"]})

    def _calculate_usage(self, tenant_id, billing_period):
        """计算账期使用量"""
        start_date, end_date = self._parse_billing_period(billing_period)
        usage = {}

        for metric in self.USAGE_METRICS:
            total = 0
            current = start_date
            while current <= end_date:
                key = f"usage:{tenant_id}:{metric}:{current.strftime('%Y%m%d')}"
                daily = float(self.redis.get(key) or 0)
                total += daily
                current += timedelta(days=1)

            usage[metric] = {"total": round(total, 2)}

        return usage
```

## 租户自助入驻自动化

```python
class TenantOnboardingAutomation:
    """租户自助入驻：注册 → 配置 → 引导"""

    def signup(self, email, company_name, plan_id):
        """自助注册"""
        # 1. 邮箱验证
        verification_token = str(uuid4())
        self.db.insert("pending_registrations", {
            "email": email, "company_name": company_name,
            "plan_id": plan_id, "token": verification_token,
            "expires_at": now() + timedelta(hours=24)
        })
        self.email_service.send(email, "验证您的邮箱",
            f"点击链接完成注册: /verify/{verification_token}")

        return {"status": "verification_sent"}

    def complete_signup(self, token, password, company_info):
        """完成注册"""
        pending = self.db.query_one(
            "SELECT * FROM pending_registrations WHERE token = %s "
            "AND expires_at > NOW()", token)
        if not pending:
            raise InvalidTokenError("验证链接已过期")

        # 1. 创建租户
        tenant_id = str(uuid4())
        self.db.insert("tenants", {
            "id": tenant_id, "name": pending["company_name"],
            "plan_id": pending["plan_id"], "status": "active",
            "created_at": now()
        })

        # 2. 创建管理员用户
        admin_id = str(uuid4())
        self.db.insert("users", {
            "id": admin_id, "tenant_id": tenant_id,
            "email": pending["email"], "role": "admin",
            "created_at": now()
        })

        # 3. 自动配置
        self._auto_provision(tenant_id, pending["plan_id"])

        # 4. 创建入驻引导清单
        self._create_onboarding_checklist(tenant_id, admin_id)

        return {"tenant_id": tenant_id, "admin_id": admin_id}

    def _auto_provision(self, tenant_id, plan_id):
        """自动配置"""
        plan = self.db.get_plan(plan_id)

        # 创建数据库 schema
        self.db.execute(f"CREATE SCHEMA tenant_{tenant_id.replace('-', '_')}")
        self.db.execute(f"CREATE TABLE tenant_{tenant_id.replace('-', '_')}.users (LIKE public.users INCLUDING ALL)")
        self.db.execute(f"CREATE TABLE tenant_{tenant_id.replace('-', '_')}.settings (LIKE public.settings INCLUDING ALL)")

        # 配置默认设置
        self.db.insert(f"tenant_{tenant_id.replace('-', '_')}.settings", {
            "key": "timezone", "value": "Asia/Shanghai"
        })

    def _create_onboarding_checklist(self, tenant_id, admin_id):
        """创建入驻引导清单"""
        items = [
            {"step": "complete_profile", "title": "完善公司信息"},
            {"step": "invite_team", "title": "邀请团队成员"},
            {"step": "import_data", "title": "导入数据"},
            {"step": "configure_integration", "title": "配置集成"},
            {"step": "first_action", "title": "完成首次操作"},
        ]
        for i, item in enumerate(items):
            self.db.insert("onboarding_items", {
                "tenant_id": tenant_id,
                "step": item["step"],
                "title": item["title"],
                "order": i + 1,
                "completed": False
            })

    def track_time_to_value(self, tenant_id):
        """追踪达到价值时间"""
        signup_date = self.db.get_tenant(tenant_id)["created_at"]
        first_action = self.db.query_one(
            "SELECT MIN(created_at) as first FROM user_actions "
            "WHERE tenant_id = %s AND action_type = 'meaningful'", tenant_id)

        if first_action and first_action["first"]:
            ttv = (first_action["first"] - signup_date).total_seconds() / 86400
            return {"days_to_value": round(ttv, 1)}
        return {"days_to_value": None, "status": "not_yet"}
```

## 异常场景补充

### 场景：计量数据丢失导致计费错误

```
触发：Redis 故障 → 使用量计数丢失 → 账单金额偏低 → 收入损失
检测：
  1. Redis 重启后计数器归零 → 数据丢失
  2. 日账单金额异常低 → 可能丢失
处理：
  1. 从 API 网关日志重建使用量
  2. 与上一期对比，偏差 > 20% → 人工审核
  3. 补发修正账单
预防：Redis 持久化 + 日志双重计量 + 异常检测
```

### 场景：租户暂停导致合规审计数据不可访问

```
触发：租户欠费被暂停 → 合规审计需要访问数据 → 无法访问
检测：
  1. 审计请求指向已暂停租户 → 冲突
  2. 合规要求数据保留 7 年 → 暂停不能删除数据
处理：
  1. 暂停服务 ≠ 删除数据
  2. 为审计提供只读访问
  3. 审计完成后恢复暂停状态
预防：暂停≠删除 + 合规只读访问 + 数据保留策略
```

## 租户 API 网关与限流完整实现

```python
class TenantAPIGateway:
    """租户 API 网关：限流 + 认证 + 路由 + 分析"""

    def handle_request(self, request):
        """处理 API 请求"""
        tenant_id = request.headers.get("X-Tenant-ID")
        api_key = request.headers.get("X-API-Key")

        # 1. API Key 认证
        if not self._validate_api_key(tenant_id, api_key):
            return {"status": 401, "error": "无效的 API Key"}

        # 2. 限流（令牌桶）
        rate_result = self._check_rate_limit(tenant_id, request.path)
        if not rate_result["allowed"]:
            return {"status": 429, "error": "请求超过限额",
                    "retry_after": rate_result["retry_after"]}

        # 3. 请求路由
        endpoint = self._route_request(tenant_id, request)

        # 4. 请求转换（注入租户上下文）
        transformed = self._transform_request(request, tenant_id)

        # 5. 调用后端服务
        response = self.backend.call(endpoint, transformed)

        # 6. 记录使用量
        self._record_usage(tenant_id, request.path, response.status_code)

        return response

    def _check_rate_limit(self, tenant_id, path):
        """令牌桶限流"""
        plan = self.db.get_tenant_plan(tenant_id)
        rate_per_second = plan.get("api_rate_limit", 100) / 3600
        burst = plan.get("api_rate_limit", 100) * 2

        key = f"rate_limit:{tenant_id}:{path}"
        current_tokens = float(self.redis.get(key) or burst)
        last_refill = float(self.redis.get(f"{key}:last") or time.time())

        # 补充令牌
        now_ts = time.time()
        elapsed = now_ts - last_refill
        new_tokens = min(burst, current_tokens + elapsed * rate_per_second)

        if new_tokens < 1:
            retry_after = (1 - new_tokens) / rate_per_second
            return {"allowed": False, "retry_after": round(retry_after, 1)}

        # 消耗令牌
        self.redis.set(key, new_tokens - 1)
        self.redis.set(f"{key}:last", now_ts)

        return {"allowed": True, "remaining": int(new_tokens - 1)}

    def _validate_api_key(self, tenant_id, api_key):
        """验证 API Key"""
        stored = self.redis.get(f"api_key:{tenant_id}")
        if stored and stored == api_key:
            return True
        # 回退到 DB
        db_key = self.db.query_one(
            "SELECT api_key FROM tenant_api_keys "
            "WHERE tenant_id = %s AND api_key = %s AND revoked_at IS NULL",
            tenant_id, api_key)
        if db_key:
            self.redis.setex(f"api_key:{tenant_id}", 3600, api_key)
            return True
        return False
```

## 租户 Feature Flag 管理

```python
class TenantFeatureFlagService:
    """Feature Flag 管理：租户覆盖 + 渐进发布 + 依赖"""

    def is_enabled(self, tenant_id, flag_name):
        """检查 Feature Flag 是否启用"""
        # 1. 租户级覆盖
        override = self.redis.get(f"flag:{tenant_id}:{flag_name}")
        if override is not None:
            return override == "true"

        # 2. 全局 Flag 状态
        flag = self.db.query_one(
            "SELECT * FROM feature_flags WHERE name = %s", flag_name)
        if not flag:
            return False

        # 3. 渐进发布检查
        if flag["rollout_strategy"] == "percentage":
            tenant_hash = int(hashlib.md5(
                f"{tenant_id}:{flag_name}".encode()).hexdigest(), 16) % 100
            return tenant_hash < flag["rollout_percentage"]

        return flag["default_enabled"]

    def set_tenant_override(self, tenant_id, flag_name, enabled):
        """设置租户级覆盖"""
        self.db.insert("feature_flag_overrides", {
            "tenant_id": tenant_id,
            "flag_name": flag_name,
            "enabled": enabled,
            "overridden_by": "admin",
            "overridden_at": now()
        })
        self.redis.set(f"flag:{tenant_id}:{flag_name}", str(enabled).lower())

    def check_dependencies(self, flag_name):
        """检查 Flag 依赖"""
        flag = self.db.query_one(
            "SELECT * FROM feature_flags WHERE name = %s", flag_name)
        if not flag or not flag.get("depends_on"):
            return {"satisfied": True}

        deps = json.loads(flag["depends_on"])
        unsatisfied = []
        for dep in deps:
            dep_flag = self.db.query_one(
                "SELECT default_enabled FROM feature_flags WHERE name = %s", dep)
            if not dep_flag or not dep_flag["default_enabled"]:
                unsatisfied.append(dep)

        return {"satisfied": len(unsatisfied) == 0,
                "unsatisfied": unsatisfied}
```

## 异常场景补充

### 场景：限流配置错误阻断租户

```
触发：管理员误将租户限流设为 10 次/小时 → 租户 API 全部 429
检测：
  1. 租户 429 错误率 100% → 配置错误
  2. 误配置后 1 分钟内投诉 → 紧急
处理：
  1. 立即恢复限流配置
  2. 限流变更需审批（影响面大）
  3. 配置变更审计
预防：限流变更审批 + 变更前影响评估 + 快速回滚
```

### 场景：Feature Flag 依赖循环

```
触发：Flag A 依赖 Flag B，Flag B 又依赖 Flag A → 无限循环
检测：
  1. 依赖图检测到环 → 循环依赖
  2. is_enabled 调用栈溢出 → 循环
处理：
  1. 创建 Flag 时检测依赖图是否有环
  2. 已存在循环 → 打破循环（移除一个依赖）
预防：创建时依赖图环检测 + 最大依赖深度限制
```

## 租户数据隔离验证完整实现

```python
class TenantIsolationValidator:
    """租户数据隔离验证：运行时检查 + 定期审计 + 泄露检测"""

    def validate_query_isolation(self, tenant_id, sql_query):
        """验证查询是否包含租户隔离条件"""
        # 1. 解析 SQL
        parsed = self._parse_sql(sql_query)

        # 2. 检查是否包含 tenant_id 过滤
        tables = self._extract_tables(parsed)
        for table in tables:
            if self._is_tenant_scoped_table(table):
                if not self._has_tenant_filter(parsed, table, tenant_id):
                    self.alert(f"租户隔离违规: 查询 {table} 未包含 tenant_id 过滤")
                    return {"isolated": False, "violation": table}

        return {"isolated": True}

    def run_isolation_audit(self):
        """运行隔离审计（扫描所有查询日志）"""
        # 检查过去 24 小时的查询
        queries = self.db.query(
            "SELECT query_text, tenant_id, executed_at "
            "FROM query_audit_log "
            "WHERE executed_at > NOW() - INTERVAL 24 HOUR")

        violations = []
        for q in queries:
            result = self.validate_query_isolation(q["tenant_id"], q["query_text"])
            if not result["isolated"]:
                violations.append({
                    "query": q["query_text"][:200],
                    "tenant_id": q["tenant_id"],
                    "violation": result["violation"],
                    "executed_at": q["executed_at"]
                })

        if violations:
            self.alert(f"发现 {len(violations)} 个租户隔离违规")

        return {"total_queries": len(queries), "violations": len(violations)}

    def detect_cross_tenant_leak(self):
        """检测跨租户数据泄露"""
        # 检查是否有查询返回了非本租户的数据
        suspicious = self.db.query(
            "SELECT q.tenant_id, q.query_text, r.returned_tenant_ids "
            "FROM query_audit_log q "
            "JOIN query_results r ON q.query_id = r.query_id "
            "WHERE q.executed_at > NOW() - INTERVAL 1 HOUR "
            "AND r.returned_tenant_ids != ARRAY[q.tenant_id]")

        if suspicious:
            self.alert(f"检测到 {len(suspicious)} 起跨租户数据泄露")

        return {"leaks_detected": len(suspicious), "details": suspicious}
```

## 租户数据迁移服务

```python
class TenantMigrationService:
    """租户数据迁移：导出 → 导入 → 验证"""

    def export_tenant_data(self, tenant_id, options=None):
        """导出租户数据"""
        options = options or {"include_files": True, "include_logs": False}

        export_id = str(uuid4())
        self.db.insert("tenant_exports", {
            "export_id": export_id,
            "tenant_id": tenant_id,
            "status": "in_progress",
            "started_at": now()
        })

        # 1. 导出数据库数据
        schema = f"tenant_{tenant_id.replace('-', '_')}"
        tables = self.db.query(
            f"SELECT tablename FROM pg_tables WHERE schemaname = '{schema}'")

        exported_tables = {}
        for table in tables:
            rows = self.db.query(f"SELECT * FROM {schema}.{table['tablename']}")
            exported_tables[table["tablename"]] = rows

        # 2. 导出文件存储
        if options.get("include_files"):
            files = self.storage.list(f"tenants/{tenant_id}/")
        else:
            files = []

        # 3. 打包
        export_data = {
            "tenant_id": tenant_id,
            "schema": schema,
            "tables": exported_tables,
            "file_count": len(files),
            "exported_at": now().isoformat()
        }

        # 上传到临时存储
        self.storage.upload(f"exports/{export_id}.json",
            json.dumps(export_data, default=str))

        self.db.update("tenant_exports",
            {"status": "completed", "completed_at": now(),
             "table_count": len(exported_tables),
             "file_count": len(files)},
            {"export_id": export_id})

        return {"export_id": export_id,
                "tables": len(exported_tables),
                "files": len(files)}

    def import_tenant_data(self, target_tenant_id, export_id):
        """导入租户数据"""
        export_data = json.loads(
            self.storage.download(f"exports/{export_id}.json"))

        schema = f"tenant_{target_tenant_id.replace('-', '_')}"

        # 1. 创建 schema
        self.db.execute(f"CREATE SCHEMA IF NOT EXISTS {schema}")

        # 2. 导入数据
        imported_tables = 0
        for table_name, rows in export_data["tables"].items():
            if rows:
                columns = list(rows[0].keys())
                for row in rows:
                    values = [row.get(c) for c in columns]
                    placeholders = ",".join(["%s"] * len(columns))
                    self.db.execute(
                        f"INSERT INTO {schema}.{table_name} ({','.join(columns)}) "
                        f"VALUES ({placeholders})", *values)
                imported_tables += 1

        # 3. 验证数据完整性
        source_count = sum(len(rows) for rows in export_data["tables"].values())
        target_count = self._count_all_rows(schema)

        return {"imported_tables": imported_tables,
                "source_rows": source_count,
                "target_rows": target_count,
                "verified": source_count == target_count}
```

## 异常场景补充

### 场景：租户隔离违规导致数据泄露

```
触发：开发者忘记在查询中加 tenant_id 条件 → 返回所有租户数据 → 数据泄露
检测：
  1. 隔离审计发现违规查询 → 泄露
  2. 查询返回行数异常多 → 可能跨租户
处理：
  1. 立即修复查询
  2. 通知受影响租户
  3. 审查是否有数据被实际泄露
预防：ORM 强制注入 tenant_id + 查询审计 + 自动化隔离测试
```

### 场景：租户数据迁移不完整

```
触发：导出时部分大表超时 → 导出不完整 → 导入后数据缺失
检测：
  1. 导入后行数不一致 → 数据缺失
  2. 特定功能报数据不存在 → 迁移不完整
处理：
  1. 增量导出缺失的表
  2. 重新导入
  3. 验证数据完整性
预防：分批导出 + 超时重试 + 行数校验
```

## 租户数据备份与恢复完整实现

```python
class TenantBackupService:
    """租户数据备份：自动备份 + 增量 + 跨区域恢复"""

    BACKUP_POLICIES = {
        "standard": {"frequency": "daily", "retention_days": 30, "type": "incremental"},
        "premium": {"frequency": "hourly", "retention_days": 90, "type": "incremental"},
        "enterprise": {"frequency": "every_15min", "retention_days": 365, "type": "incremental"},
    }

    def create_backup(self, tenant_id, backup_type="scheduled"):
        """创建租户备份"""
        tenant = self.db.get_tenant(tenant_id)
        plan = tenant["plan_id"]
        policy = self.BACKUP_POLICIES.get(plan, self.BACKUP_POLICIES["standard"])

        backup_id = str(uuid4())
        schema = f"tenant_{tenant_id.replace('-', '_')}"

        # 1. 获取上次备份的 LSN（日志序列号）
        last_lsn = self.db.query_one(
            "SELECT lsn FROM tenant_backups "
            "WHERE tenant_id = %s AND status = 'completed' "
            "ORDER BY completed_at DESC LIMIT 1", tenant_id)
        last_lsn = last_lsn["lsn"] if last_lsn else None

        # 2. 执行增量备份
        self.db.insert("tenant_backups", {
            "backup_id": backup_id,
            "tenant_id": tenant_id,
            "backup_type": backup_type,
            "base_lsn": last_lsn,
            "status": "in_progress",
            "started_at": now()
        })

        # 3. 导出数据
        if last_lsn and policy["type"] == "incremental":
            # 增量备份：只导出 LSN 之后的变更
            changes = self._export_incremental(schema, last_lsn)
            size_bytes = len(json.dumps(changes).encode())
        else:
            # 全量备份
            changes = self._export_full(schema)
            size_bytes = len(json.dumps(changes).encode())

        # 4. 上传到对象存储（跨区域）
        storage_path = f"backups/{tenant_id}/{backup_id}.json"
        self.object_storage.upload(storage_path, changes,
            storage_class="STANDARD_IA",  # 低频访问
            replication_regions=["us-east-1", "eu-west-1"])

        # 5. 更新备份记录
        current_lsn = self._get_current_lsn()
        self.db.update("tenant_backups", {
            "status": "completed",
            "storage_path": storage_path,
            "size_bytes": size_bytes,
            "lsn": current_lsn,
            "completed_at": now()
        }, {"backup_id": backup_id})

        return {"backup_id": backup_id, "size_mb": round(size_bytes / 1048576, 2),
                "type": "incremental" if last_lsn else "full"}

    def restore_tenant(self, tenant_id, backup_id=None, point_in_time=None):
        """恢复租户数据"""
        if backup_id:
            backup = self.db.get_backup(backup_id)
        elif point_in_time:
            # 找到 PIT 之前最近的备份
            backup = self.db.query_one(
                "SELECT * FROM tenant_backups "
                "WHERE tenant_id = %s AND status = 'completed' "
                "AND completed_at <= %s "
                "ORDER BY completed_at DESC LIMIT 1",
                tenant_id, point_in_time)
        else:
            # 最新备份
            backup = self.db.query_one(
                "SELECT * FROM tenant_backups "
                "WHERE tenant_id = %s AND status = 'completed' "
                "ORDER BY completed_at DESC LIMIT 1", tenant_id)

        if not backup:
            raise BackupNotFoundError("无可用备份")

        # 1. 下载备份数据
        data = self.object_storage.download(backup["storage_path"])

        # 2. 如果是增量备份 → 链式恢复
        if backup["base_lsn"]:
            # 先恢复基础全量备份
            base_backup = self._find_base_backup(tenant_id, backup["base_lsn"])
            self._apply_full_restore(tenant_id, base_backup)
            # 再依次应用增量
            increments = self._get_increment_chain(tenant_id, base_backup, backup)
            for inc in increments:
                self._apply_incremental(tenant_id, inc)

        # 3. 验证恢复
        verification = self._verify_restore(tenant_id, backup)

        return {"backup_id": backup["backup_id"],
                "restored_at": now().isoformat(),
                "verification": verification}
```

## 异常场景补充

### 场景：增量备份链断裂

```
触发：中间某次增量备份损坏 → 无法恢复到最新状态 → 只能恢复到更早的全量备份
检测：
  1. 增量备份校验失败 → 链断裂
  2. 恢复时找不到基础 LSN → 链不完整
处理：
  1. 从断裂点前最后一个全量备份恢复
  2. 定期创建全量备份（每周一次）缩短链长度
  3. 重建增量链
预防：定期全量备份 + 链完整性检查 + 备份校验
```

### 场景：跨区域恢复延迟

```
触发：主区域故障 → 需要从备份区域恢复 → 数据传输 100GB → 恢复耗时 4 小时
检测：
  1. 主区域不可用 → 需要跨区域恢复
  2. 恢复时间 > RTO → SLA 违规
处理：
  1. 优先恢复关键租户
  2. 使用热备区域（数据实时同步）
  3. 降级服务（只读模式）
预防：热备区域 + 实时复制 + 分级恢复
```

## SaaS 计费引擎完整实现

```python
class SaaSBillingEngine:
    """SaaS 计费引擎：用量计量 + 阶梯计价 + 账单生成"""

    PRICING_MODELS = {
        "flat": "固定月费",
        "per_unit": "按量计费",
        "tiered": "阶梯计价",
        "hybrid": "基础费 + 按量",
    }

    def calculate_monthly_bill(self, tenant_id, billing_month):
        """计算月度账单"""
        tenant = self.db.get_tenant(tenant_id)
        plan = self.db.get_plan(tenant["plan_id"])

        line_items = []

        # 1. 基础费用
        if plan["base_price"] > 0:
            line_items.append({
                "description": f"{plan['name']} 基础费",
                "amount": plan["base_price"],
                "type": "base"
            })

        # 2. 按量计费项
        metered_items = json.loads(plan.get("metered_items", "[]"))
        for item in metered_items:
            usage = self._get_usage(tenant_id, billing_month, item["metric"])

            if item["pricing_model"] == "per_unit":
                amount = usage * item["unit_price"]
                line_items.append({
                    "description": f"{item['name']} × {usage} {item['unit']}",
                    "amount": round(amount, 2),
                    "type": "metered",
                    "metric": item["metric"],
                    "usage": usage
                })

            elif item["pricing_model"] == "tiered":
                amount = self._calculate_tiered(usage, item["tiers"])
                line_items.append({
                    "description": f"{item['name']} 阶梯计费 ({usage} {item['unit']})",
                    "amount": round(amount, 2),
                    "type": "metered",
                    "metric": item["metric"],
                    "usage": usage
                })

        # 3. 超额费用
        overages = self._calculate_overages(tenant_id, billing_month, plan)
        line_items.extend(overages)

        # 4. 折扣
        discount = self._calculate_discount(tenant_id, billing_month, plan)
        if discount > 0:
            line_items.append({
                "description": "折扣",
                "amount": -round(discount, 2),
                "type": "discount"
            })

        # 5. 汇总
        total = sum(item["amount"] for item in line_items)

        return {
            "tenant_id": tenant_id,
            "billing_month": billing_month,
            "line_items": line_items,
            "subtotal": round(total, 2),
            "tax": round(total * 0.06, 2),  # 6% 增值税
            "total": round(total * 1.06, 2)
        }

    def _calculate_tiered(self, usage, tiers):
        """阶梯计价"""
        # tiers: [{"up_to": 1000, "price": 0.10}, {"up_to": 10000, "price": 0.08}, ...]
        total = 0
        previous_limit = 0

        for tier in sorted(tiers, key=lambda t: t["up_to"]):
            tier_limit = tier["up_to"]
            tier_price = tier["price"]

            if usage <= previous_limit:
                break

            tier_usage = min(usage, tier_limit) - previous_limit
            total += tier_usage * tier_price
            previous_limit = tier_limit

        # 超出最高阶梯
        if usage > previous_limit:
            highest_price = tiers[-1]["price"]
            total += (usage - previous_limit) * highest_price

        return total

    def _get_usage(self, tenant_id, billing_month, metric):
        """获取用量"""
        return self.db.query_one(
            "SELECT SUM(value) as total FROM usage_metrics "
            "WHERE tenant_id = %s AND metric = %s "
            "AND billing_month = %s",
            tenant_id, metric, billing_month)["total"] or 0

    def generate_invoice(self, tenant_id, billing_month):
        """生成发票"""
        bill = self.calculate_monthly_bill(tenant_id, billing_month)

        invoice_id = str(uuid4())
        self.db.insert("invoices", {
            "invoice_id": invoice_id,
            "tenant_id": tenant_id,
            "billing_month": billing_month,
            "line_items": json.dumps(bill["line_items"]),
            "subtotal": bill["subtotal"],
            "tax": bill["tax"],
            "total": bill["total"],
            "status": "pending",
            "due_date": (now().replace(day=1) + timedelta(days=30)).date(),
            "created_at": now()
        })

        # 发送账单通知
        self.notification.send(tenant_id,
            f"您的 {billing_month} 月账单已生成，总计 ¥{bill['total']:.2f}")

        return {"invoice_id": invoice_id, "total": bill["total"]}
```

## 异常场景补充

### 场景：用量计量数据缺失

```
触发：监控系统故障 → 某月用量数据缺失 → 账单金额为 0 → 收入损失
检测：
  1. 账单金额为 0 但租户有活跃用户 → 数据缺失
  2. 用量数据与上月差异 > 50% → 异常
处理：
  1. 从监控系统补采数据
  2. 使用上月用量估算
  3. 重新生成账单
预防：用量数据完整性检查 + 异常告警 + 补录机制
```

### 场景：阶梯计价边界争议

```
触发：用量恰好 1000 → 第 1 阶梯到 1000，第 2 阶梯从 1001 → 用户认为应按第 2 阶梯
检测：
  1. 用量恰好在阶梯边界 → 争议
  2. 用户对账单提出异议 → 阶梯边界问题
处理：
  1. 阶梯定义明确（含/不含边界）
  2. 账单中显示各阶梯明细
  3. 边界争议 → 按有利于用户的方式处理
预防：阶梯定义文档化 + 账单明细 + 有利于用户原则
```

## SaaS 数据迁移服务完整实现

```python
class TenantDataMigrationService:
    """租户数据迁移：导出 + 导入 + 校验 + 回滚"""

    MIGRATION_STEPS = [
        "export_data", "validate_export", "import_data",
        "validate_import", "switch_tenant", "cleanup_old"
    ]

    def migrate_tenant(self, tenant_id, target_environment):
        """迁移租户数据"""
        migration_id = str(uuid4())

        self.db.insert("tenant_migrations", {
            "migration_id": migration_id,
            "tenant_id": tenant_id,
            "source_env": "production",
            "target_env": target_environment,
            "status": "in_progress",
            "started_at": now()
        })

        schema = f"tenant_{tenant_id.replace('-', '_')}"

        # 1. 导出数据
        export_result = self._export_tenant_data(tenant_id, schema)
        self._update_migration_step(migration_id, "export_data", "completed",
            export_result)

        # 2. 验证导出完整性
        validation = self._validate_export(export_result)
        if not validation["passed"]:
            self._fail_migration(migration_id, "export_validation_failed", validation)
            return {"status": "failed", "reason": "导出验证失败"}

        self._update_migration_step(migration_id, "validate_export", "completed", validation)

        # 3. 在目标环境导入
        import_result = self._import_tenant_data(tenant_id, export_result, target_environment)
        self._update_migration_step(migration_id, "import_data", "completed", import_result)

        # 4. 验证导入
        import_validation = self._validate_import(tenant_id, target_environment)
        if not import_validation["passed"]:
            self._rollback_import(tenant_id, target_environment)
            self._fail_migration(migration_id, "import_validation_failed", import_validation)
            return {"status": "failed", "reason": "导入验证失败"}

        self._update_migration_step(migration_id, "validate_import", "completed", import_validation)

        # 5. 切换租户指向新环境
        self.db.update("tenants",
            {"environment": target_environment, "migrated_at": now()},
            {"id": tenant_id})
        self._update_migration_step(migration_id, "switch_tenant", "completed")

        # 6. 清理旧数据
        self._cleanup_old_environment(tenant_id, schema)
        self._update_migration_step(migration_id, "cleanup_old", "completed")

        self.db.update("tenant_migrations",
            {"status": "completed", "completed_at": now()},
            {"migration_id": migration_id})

        return {"migration_id": migration_id, "status": "completed"}

    def _export_tenant_data(self, tenant_id, schema):
        """导出租户数据"""
        tables = self._get_tenant_tables(schema)
        export_path = f"migration/{tenant_id}"
        total_rows = 0
        exported_tables = []

        for table in tables:
            count = self.db.count(table)
            total_rows += count

            data = self.db.query(f"SELECT * FROM {table} LIMIT 1000000")
            file_path = f"{export_path}/{table}.json"

            self.object_storage.upload(file_path, json.dumps(data))

            exported_tables.append({
                "table": table, "rows": count,
                "file": file_path, "checksum": hashlib.md5(json.dumps(data).encode()).hexdigest()
            })

        return {"tenant_id": tenant_id, "total_rows": total_rows,
                "tables": exported_tables, "export_path": export_path}

    def _validate_export(self, export_result):
        """验证导出完整性"""
        issues = []

        for table_info in export_result["tables"]:
            # 校验 checksum
            data = self.object_storage.download(table_info["file"])
            actual_checksum = hashlib.md5(data.encode()).hexdigest()
            if actual_checksum != table_info["checksum"]:
                issues.append(f"表 {table_info['table']} checksum 不匹配")

            # 校验行数
            parsed = json.loads(data)
            if len(parsed) != table_info["rows"]:
                issues.append(f"表 {table_info['table']} 行数不匹配: 期望 {table_info['rows']}, 实际 {len(parsed)}")

        return {"passed": len(issues) == 0, "issues": issues}

    def _validate_import(self, tenant_id, target_env):
        """验证导入完整性"""
        # 对比源和目标的行数
        schema = f"tenant_{tenant_id.replace('-', '_')}"
        tables = self._get_tenant_tables(schema)

        issues = []
        for table in tables:
            source_count = self.db.query_one(
                f"SELECT COUNT(*) as c FROM {table}", env="production")["c"]
            target_count = self.db.query_one(
                f"SELECT COUNT(*) as c FROM {table}", env=target_env)["c"]

            if source_count != target_count:
                issues.append(f"表 {table}: 源 {source_count} 行, 目标 {target_count} 行")

        return {"passed": len(issues) == 0, "issues": issues}
```

## 异常场景补充

### 场景：迁移中途源环境数据变更

```
触发：迁移过程中源环境有新数据写入 → 导出数据不含新数据 → 目标环境数据不全
检测：
  1. 导出后源环境行数增加 → 数据变更
  2. 导入验证发现行数不匹配 → 迁移不全
处理：
  1. 迁移前设置租户为只读模式
  2. 导出后捕获增量变更（CDC）
  3. 导入后追加增量数据
预防：只读模式 + CDC 增量 + 数据校验
```

### 场景：大租户数据导出耗时过长

```
触发：租户有 500 万行数据 → 导出耗时 4 小时 → 迁移窗口超出 → 影响业务
检测：
  1. 导出耗时 > 预估时间 → 需要优化
  2. 迁移窗口即将结束 → 需要加速
处理：
  1. 并行导出（按表分批）
  2. 使用数据库原生导出工具（pg_dump）替代逐行查询
  3. 增量迁移而非全量
预防：并行导出 + 原生工具 + 增量迁移
```

## SaaS 租户配置中心完整实现

```python
class TenantConfigCenterService:
    """租户配置中心：功能开关 + 参数配置 + 灰度发布"""

    def get_config(self, tenant_id, config_key, default=None):
        """获取租户配置（租户级 → 全局默认）"""
        # 1. 查租户级配置
        tenant_config = self.redis.hget(f"tenant_config:{tenant_id}", config_key)
        if tenant_config is not None:
            value = json.loads(tenant_config) if isinstance(tenant_config, bytes) else tenant_config
            return value

        # 2. 查全局默认
        global_config = self.redis.hget("global_config", config_key)
        if global_config is not None:
            return json.loads(global_config) if isinstance(global_config, bytes) else global_config

        return default

    def set_config(self, tenant_id, config_key, value, operator_id, rollout_strategy=None):
        """设置租户配置"""
        # 1. 验证配置键
        config_schema = self.db.query_one(
            "SELECT * FROM config_schemas WHERE config_key = %s", config_key)
        if not config_schema:
            raise InvalidConfigKeyError(f"无效配置键: {config_key}")

        # 2. 类型验证
        expected_type = config_schema["value_type"]
        if expected_type == "integer" and not isinstance(value, int):
            raise ConfigTypeError(f"配置 {config_key} 需要整数类型")
        elif expected_type == "boolean" and not isinstance(value, bool):
            raise ConfigTypeError(f"配置 {config_key} 需要布尔类型")
        elif expected_type == "enum" and value not in json.loads(config_schema["enum_values"]):
            raise ConfigTypeError(f"配置 {config_key} 值不在允许范围内")

        # 3. 范围验证
        if config_schema.get("min_value") is not None and value < config_schema["min_value"]:
            raise ConfigRangeError(f"配置 {config_key} 不能小于 {config_schema['min_value']}")
        if config_schema.get("max_value") is not None and value > config_schema["max_value"]:
            raise ConfigRangeError(f"配置 {config_key} 不能大于 {config_schema['max_value']}")

        # 4. 灰度发布
        if rollout_strategy and rollout_strategy.get("type") == "gradual":
            return self._gradual_rollout(tenant_id, config_key, value,
                operator_id, rollout_strategy)

        # 5. 直接设置
        old_value = self.get_config(tenant_id, config_key)

        if tenant_id == "global":
            self.redis.hset("global_config", config_key, json.dumps(value))
        else:
            self.redis.hset(f"tenant_config:{tenant_id}", config_key, json.dumps(value))

        # 6. 记录变更日志
        self.db.insert("config_change_log", {
            "log_id": str(uuid4()),
            "tenant_id": tenant_id,
            "config_key": config_key,
            "old_value": json.dumps(old_value),
            "new_value": json.dumps(value),
            "operator_id": operator_id,
            "changed_at": now()
        })

        # 7. 通知相关服务
        affected_services = json.loads(config_schema.get("affected_services", "[]"))
        for service in affected_services:
            self.event_bus.publish(f"config_changed:{service}", {
                "tenant_id": tenant_id,
                "config_key": config_key,
                "new_value": value
            })

        return {"config_key": config_key, "value": value, "tenant_id": tenant_id}

    def _gradual_rollout(self, tenant_id, config_key, value, operator_id, strategy):
        """灰度发布配置变更"""
        pct = strategy.get("percentage", 10)
        duration_minutes = strategy.get("duration_minutes", 60)

        # 获取受影响的用户列表
        users = self.db.query(
            "SELECT user_id FROM users WHERE tenant_id = %s AND status = 'active'",
            tenant_id)

        total_users = len(users)
        rollout_count = max(1, int(total_users * pct / 100))

        # 随机选择灰度用户
        rollout_users = random.sample([u["user_id"] for u in users], rollout_count)

        # 设置灰度标记
        rollout_id = str(uuid4())
        self.db.insert("config_rollouts", {
            "rollout_id": rollout_id,
            "tenant_id": tenant_id,
            "config_key": config_key,
            "new_value": json.dumps(value),
            "rollout_pct": pct,
            "target_users": json.dumps(rollout_users),
            "status": "in_progress",
            "started_at": now(),
            "scheduled_complete_at": now() + timedelta(minutes=duration_minutes),
            "operator_id": operator_id
        })

        # 对灰度用户生效
        for user_id in rollout_users:
            self.redis.set(f"user_config_override:{tenant_id}:{user_id}:{config_key}",
                json.dumps(value))

        return {"rollout_id": rollout_id, "rollout_pct": pct,
                "affected_users": rollout_count}

    def complete_rollout(self, rollout_id):
        """完成灰度 → 全量生效"""
        rollout = self.db.get_rollout(rollout_id)

        # 1. 对全租户生效
        self.set_config(rollout["tenant_id"], rollout["config_key"],
            json.loads(rollout["new_value"]), rollout["operator_id"])

        # 2. 清除用户级覆盖
        rollout_users = json.loads(rollout["target_users"])
        for user_id in rollout_users:
            self.redis.delete(f"user_config_override:{rollout['tenant_id']}:{user_id}:{rollout['config_key']}")

        # 3. 更新灰度状态
        self.db.update("config_rollouts",
            {"status": "completed", "completed_at": now()},
            {"rollout_id": rollout_id})

        return {"status": "completed"}
```

## 异常场景补充

### 场景：配置变更导致服务异常

```
触发：修改最大并发数从 100 → 10 → 大量请求被拒绝 → 服务降级
检测：
  1. 配置变更后错误率上升 → 配置问题
  2. 变更值与当前值差异大 → 高风险变更
处理：
  1. 配置变更自动回退（变更后 5 分钟内错误率上升 → 回退）
  2. 变更前自动备份旧值
  3. 高风险变更需审批
预防：自动回退 + 备份旧值 + 审批机制
```

### 场景：灰度配置与全量配置冲突

```
触发：灰度用户看到新配置 → 全量发布后灰度覆盖未清除 → 用户看到旧值
检测：
  1. 灰度用户配置值 ≠ 全量配置 → 覆盖未清除
  2. 用户看到不一致的行为 → 配置冲突
处理：
  1. 全量发布时清除所有用户级覆盖
  2. 配置获取优先级：用户覆盖 > 租户配置 > 全局默认
  3. 定期清理过期的用户级覆盖
预防：全量清除 + 明确优先级 + 定期清理
```

## SaaS 租户数据隔离验证完整实现

```python
class TenantDataIsolationVerifier:
    """数据隔离验证：自动检测租户间数据泄露"""

    def verify_query_isolation(self, tenant_id):
        """验证租户查询是否正确隔离"""
        issues = []

        # 1. 查询所有数据表，检查是否有租户 ID 过滤
        tables = self.db.query(
            "SELECT table_name FROM information_schema.tables "
            "WHERE table_schema = 'public' AND table_type = 'BASE TABLE'")

        for table in tables:
            table_name = table["table_name"]

            # 跳过不需要隔离的表
            if table_name in ["users", "tenants", "global_config"]:
                continue

            # 检查表是否有 tenant_id 列
            has_tenant_col = self.db.query_one(
                "SELECT COUNT(*) as cnt FROM information_schema.columns "
                "WHERE table_name = %s AND column_name = 'tenant_id'",
                table_name)["cnt"]

            if not has_tenant_col:
                issues.append({
                    "table": table_name,
                    "issue": "缺少 tenant_id 列",
                    "severity": "critical"
                })
                continue

            # 2. 检查是否有不带 tenant_id 的数据
            cross_tenant = self.db.query_one(
                f"SELECT COUNT(*) as cnt FROM {table_name} "
                f"WHERE tenant_id IS NULL")["cnt"]

            if cross_tenant > 0:
                issues.append({
                    "table": table_name,
                    "issue": f"有 {cross_tenant} 条记录 tenant_id 为 NULL",
                    "severity": "high"
                })

        # 3. 模拟租户查询，检查是否返回其他租户数据
        for table in tables:
            table_name = table["table_name"]
            has_tenant_col = self.db.query_one(
                "SELECT COUNT(*) as cnt FROM information_schema.columns "
                "WHERE table_name = %s AND column_name = 'tenant_id'",
                table_name)["cnt"]

            if not has_tenant_col:
                continue

            # 模拟查询（应该只返回当前租户数据）
            other_tenant_data = self.db.query_one(
                f"SELECT COUNT(*) as cnt FROM {table_name} "
                f"WHERE tenant_id != %s", tenant_id)["cnt"]

            total = self.db.query_one(
                f"SELECT COUNT(*) as cnt FROM {table_name} "
                f"WHERE tenant_id = %s", tenant_id)["cnt"]

            if other_tenant_data > 0 and total > 0:
                issues.append({
                    "table": table_name,
                    "issue": "查询未强制过滤 tenant_id，可能返回其他租户数据",
                    "severity": "critical"
                })

        return {"tenant_id": tenant_id, "tables_checked": len(tables),
                "issues_found": len(issues), "issues": issues}

    def verify_api_isolation(self, tenant_id, api_endpoints):
        """验证 API 层隔离"""
        issues = []

        for endpoint in api_endpoints:
            # 使用租户 A 的 token 访问，尝试获取租户 B 的数据
            response = self.http_client.get(
                endpoint["url"],
                headers={"Authorization": f"Bearer {self._get_tenant_token(tenant_id)}",
                         "X-Tenant-ID": tenant_id})

            # 检查返回数据是否只包含当前租户
            if response.status_code == 200:
                data = response.json()

                # 检查是否有其他租户的数据
                if isinstance(data, list):
                    for item in data:
                        if item.get("tenant_id") and item["tenant_id"] != tenant_id:
                            issues.append({
                                "endpoint": endpoint["url"],
                                "issue": "返回了其他租户的数据",
                                "severity": "critical"
                            })
                            break

                elif isinstance(data, dict):
                    if data.get("tenant_id") and data["tenant_id"] != tenant_id:
                        issues.append({
                            "endpoint": endpoint["url"],
                            "issue": "返回了其他租户的数据",
                            "severity": "critical"
                        })

            # 尝试直接访问其他租户的资源
            if endpoint.get("resource_id"):
                other_tenant_resource = endpoint["url"].replace(
                    endpoint["resource_id"],
                    self._get_other_tenant_resource_id(tenant_id, endpoint["resource_type"]))

                response = self.http_client.get(
                    other_tenant_resource,
                    headers={"Authorization": f"Bearer {self._get_tenant_token(tenant_id)}",
                             "X-Tenant-ID": tenant_id})

                if response.status_code == 200:
                    issues.append({
                        "endpoint": other_tenant_resource,
                        "issue": "可以访问其他租户的资源",
                        "severity": "critical"
                    })

        return {"tenant_id": tenant_id, "endpoints_checked": len(api_endpoints),
                "issues_found": len(issues), "issues": issues}
```

## 异常场景补充

### 场景：数据库层面租户隔离被绕过

```
触发：开发人员在 SQL 查询中忘记加 tenant_id 过滤 → 返回所有租户数据 → 数据泄露
检测：
  1. 查询返回的记录数远超当前租户应有数量 → 隔离失败
  2. 数据隔离验证工具检测到缺失过滤 → 隔离问题
处理：
  1. ORM 层强制注入 tenant_id 过滤（Row Level Security）
  2. 数据库层 RLS 策略（PostgreSQL）
  3. 代码审查检查所有 SQL 查询
预防：ORM 强制过滤 + RLS 策略 + 代码审查
```

### 场景：超级管理员跨租户访问

```
触发：平台运维需要排查问题 → 使用超级管理员账号 → 可查看所有租户数据 → 权限过大
检测：
  1. 超级管理员频繁跨租户查询 → 权限滥用
  2. 查询非自己租户的数据 → 越权
处理：
  1. 超级管理员操作需审计日志
  2. 跨租户访问需二次授权
  3. 敏感数据脱敏显示
预防：审计日志 + 二次授权 + 数据脱敏
```

## SaaS 租户计费与用量追踪完整实现

```python
class TenantBillingService:
    """租户计费：用量追踪 → 账单计算 → 套餐升级 → 用量限制 → 发票生成"""

    RESOURCE_PRICING = {
        "api_calls": {"unit": "次", "included_free": 100000, "overage_price": 0.001},
        "storage_gb": {"unit": "GB", "included_free": 50, "overage_price": 0.10},
        "compute_hours": {"unit": "小时", "included_free": 200, "overage_price": 0.05},
        "bandwidth_gb": {"unit": "GB", "included_free": 100, "overage_price": 0.08},
        "users": {"unit": "人", "included_free": 50, "overage_price": 2.00},
    }

    PLAN_DETAILS = {
        "starter": {"monthly_fee": 299, "resources": {
            "api_calls": 100000, "storage_gb": 50, "compute_hours": 200,
            "bandwidth_gb": 100, "users": 50}},
        "professional": {"monthly_fee": 999, "resources": {
            "api_calls": 1000000, "storage_gb": 500, "compute_hours": 2000,
            "bandwidth_gb": 1000, "users": 200}},
        "enterprise": {"monthly_fee": 4999, "resources": {
            "api_calls": 10000000, "storage_gb": 5000, "compute_hours": 20000,
            "bandwidth_gb": 10000, "users": 1000}},
    }

    def track_usage(self, tenant_id, resource_type, amount):
        """追踪资源用量"""
        if resource_type not in self.RESOURCE_PRICING:
            raise ValueError(f"未知资源类型: {resource_type}")

        # 1. 实时增量统计（Redis）
        current_month = now().strftime("%Y%m")
        usage_key = f"tenant_usage:{tenant_id}:{current_month}:{resource_type}"
        new_total = self.redis.incrbyfloat(usage_key, float(amount))
        self.redis.expireat(usage_key,
            int((now().replace(day=1, hour=0, minute=0, second=0) +
                timedelta(days=32)).timestamp()))

        # 2. 检查是否超量
        tenant = self.db.get_tenant(tenant_id)
        plan = self.PLAN_DETAILS.get(tenant["plan"], "starter")
        included = plan["resources"].get(resource_type, 0)

        usage_pct = new_total / included if included > 0 else float('inf')

        # 3. 超量告警
        if usage_pct >= 1.0:
            self._apply_usage_limit(tenant_id, resource_type, new_total, included)
        elif usage_pct >= 0.8:
            self.notification.send(tenant_id,
                f"资源 {resource_type} 用量已达 {usage_pct:.0%}，请注意")

        # 4. 每小时汇总到数据库
        if int(new_total) % 1000 < int(amount):  # 约每 1000 单位写一次
            self.db.upsert("tenant_monthly_usage", {
                "tenant_id": tenant_id,
                "period": current_month,
                "resource_type": resource_type,
                "usage_amount": int(new_total),
                "updated_at": now()
            }, conflict_columns=["tenant_id", "period", "resource_type"])

        return {"tenant_id": tenant_id, "resource_type": resource_type,
                "current_usage": int(new_total), "included": included,
                "usage_pct": round(usage_pct, 3)}

    def calculate_monthly_bill(self, tenant_id, period):
        """计算月度账单"""
        tenant = self.db.get_tenant(tenant_id)
        plan = self.PLAN_DETAILS.get(tenant["plan"], "starter")

        # 1. 基础套餐费
        base_fee = Decimal(str(plan["monthly_fee"]))

        # 2. 年付折扣
        if tenant.get("billing_cycle") == "annual":
            discount_pct = Decimal("20")
            base_fee = base_fee * (Decimal("100") - discount_pct) / Decimal("100")

        # 3. 计算各资源超量费用
        overage_charges = {}
        total_overage = Decimal("0")

        for resource_type, pricing in self.RESOURCE_PRICING.items():
            included = plan["resources"].get(resource_type, 0)
            usage_key = f"tenant_usage:{tenant_id}:{period}:{resource_type}"

            usage = int(float(self.redis.get(usage_key) or 0))

            if usage > included:
                overage = usage - included
                overage_cost = Decimal(str(overage)) * Decimal(str(pricing["overage_price"]))
                overage_charges[resource_type] = {
                    "usage": usage, "included": included,
                    "overage": overage,
                    "overage_price": pricing["overage_price"],
                    "overage_cost": str(overage_cost.quantize(Decimal("0.01")))
                }
                total_overage += overage_cost
            else:
                overage_charges[resource_type] = {
                    "usage": usage, "included": included, "overage": 0,
                    "overage_cost": "0.00"
                }

        # 4. 税费计算
        subtotal = base_fee + total_overage
        tax_rate = Decimal("0.06")  # 增值税 6%
        tax_amount = (subtotal * tax_rate).quantize(Decimal("0.01"))

        # 5. 账单
        total = subtotal + tax_amount

        bill = {
            "tenant_id": tenant_id,
            "period": period,
            "plan": tenant["plan"],
            "base_fee": str(base_fee.quantize(Decimal("0.01"))),
            "overage_charges": overage_charges,
            "total_overage": str(total_overage.quantize(Decimal("0.01"))),
            "subtotal": str(subtotal.quantize(Decimal("0.01"))),
            "tax_rate": str(tax_rate),
            "tax_amount": str(tax_amount),
            "total": str(total.quantize(Decimal("0.01"))),
        }

        return bill

    def upgrade_plan(self, tenant_id, new_plan_id, effective_date=None):
        """升级套餐"""
        tenant = self.db.get_tenant(tenant_id)
        old_plan = self.PLAN_DETAILS.get(tenant["plan"], "starter")
        new_plan = self.PLAN_DETAILS.get(new_plan_id)

        if not new_plan:
            return {"status": "invalid_plan"}

        effective_date = effective_date or now()

        # 1. 计算旧套餐剩余天数退款
        billing_start = tenant.get("current_period_start", now().replace(day=1))
        days_in_period = (now().replace(day=1, month=now().month % 12 + 1) - billing_start).days
        days_used = (now() - billing_start).days
        days_remaining = max(0, days_in_period - days_used)

        old_daily_rate = Decimal(str(old_plan["monthly_fee"])) / Decimal(str(max(days_in_period, 1)))
        credit = (old_daily_rate * Decimal(str(days_remaining))).quantize(Decimal("0.01"))

        # 2. 计算新套餐按天收费
        new_daily_rate = Decimal(str(new_plan["monthly_fee"])) / Decimal(str(max(days_in_period, 1)))
        new_charge = (new_daily_rate * Decimal(str(days_remaining))).quantize(Decimal("0.01"))

        # 3. 差价
        prorated_charge = new_charge - credit

        # 4. 执行升级
        self.db.update("tenants",
            {"plan": new_plan_id, "plan_updated_at": now()},
            {"id": tenant_id})

        # 5. 记录变更
        self.db.insert("plan_change_log", {
            "log_id": str(uuid4()),
            "tenant_id": tenant_id,
            "old_plan": tenant["plan"],
            "new_plan": new_plan_id,
            "credit": str(credit),
            "new_charge": str(new_charge),
            "prorated_charge": str(prorated_charge),
            "effective_date": effective_date,
            "changed_at": now()
        })

        # 6. 更新资源限制
        for resource_type, limit in new_plan["resources"].items():
            self.redis.set(f"tenant_limit:{tenant_id}:{resource_type}", limit)

        return {"status": "upgraded", "old_plan": tenant["plan"],
                "new_plan": new_plan_id, "credit": str(credit),
                "prorated_charge": str(prorated_charge)}

    def _apply_usage_limit(self, tenant_id, resource_type, current, included):
        """应用用量限制"""
        over_pct = current / included if included > 0 else float('inf')

        if over_pct >= 1.5:
            # 硬限制 → 阻断请求
            self.redis.set(f"tenant_blocked:{tenant_id}:{resource_type}", "1")
            self.notification.send(tenant_id,
                f"资源 {resource_type} 用量已达 {over_pct:.0%}，已限制访问，请升级套餐")
        elif over_pct >= 1.0:
            # 软限制 → 降速
            self.redis.set(f"tenant_throttled:{tenant_id}:{resource_type}", "1")
            self.notification.send(tenant_id,
                f"资源 {resource_type} 已超量，请求将被降速，请考虑升级套餐")
```

## 异常场景补充

### 场景：计费系统重复扣费

```
触发：用户升级套餐 → 支付成功但回调重复 → 重复扣费 → 多收费
检测：
  1. 同一订单出现多条支付记录 → 重复扣费
  2. 用户账单金额与预期不符 → 计费异常
处理：
  1. 支付回调幂等处理（支付单号去重）
  2. 每日账单对账（支付总额 vs 账单总额）
  3. 自动退还多扣金额
预防：幂等处理 + 每日对账 + 自动退还
```

### 场景：用量统计延迟导致误限流

```
触发：Redis 用量统计延迟 → 实际用量已超但统计未更新 → 后续请求未限流 → 超量使用
检测：
  1. 实际用量与 Redis 统计偏差 > 5% → 统计不准
  2. 月末用量汇总与实时统计不一致 → 延迟问题
处理：
  1. 关键限制点（100% 用量）增加数据库二次验证
  2. 每小时将 Redis 数据同步到数据库并校验
  3. 偏差超过阈值 → 以数据库为准修正
预防：二次验证 + 定时同步 + 偏差修正
```

## SaaS 租户数据迁移与租户生命周期完整实现

```python
class TenantLifecycleService:
    """租户生命周期：创建 → 配置 → 迁移 → 休眠 → 注销"""

    def create_tenant(self, tenant_data):
        """创建租户"""
        tenant_id = str(uuid4())

        # 1. 验证域名唯一性
        if tenant_data.get("custom_domain"):
            existing = self.db.query_one(
                "SELECT * FROM tenants WHERE custom_domain = %s",
                tenant_data["custom_domain"])
            if existing:
                return {"status": "domain_taken"}

        # 2. 选择初始套餐
        plan_id = tenant_data.get("plan_id", "starter")

        # 3. 创建租户记录
        self.db.insert("tenants", {
            "id": tenant_id,
            "name": tenant_data["name"],
            "plan": plan_id,
            "admin_user_id": tenant_data["admin_user_id"],
            "custom_domain": tenant_data.get("custom_domain"),
            "status": "active",
            "created_at": now()
        })

        # 4. 初始化租户数据空间
        self._initialize_tenant_schema(tenant_id)

        # 5. 创建默认角色和权限
        self._create_default_roles(tenant_id)

        # 6. 设置资源限制
        plan = self.PLAN_DETAILS.get(plan_id, self.PLAN_DETAILS["starter"])
        for resource_type, limit in plan["resources"].items():
            self.redis.set(f"tenant_limit:{tenant_id}:{resource_type}", limit)
            self.redis.set(f"tenant_usage:{tenant_id}:{now().strftime('%Y%m')}:{resource_type}", 0)

        # 7. 配置通知渠道
        self.db.insert("tenant_notification_config", {
            "tenant_id": tenant_id,
            "channels": json.dumps(["email"]),
            "admin_email": tenant_data.get("admin_email"),
            "created_at": now()
        })

        return {"tenant_id": tenant_id, "name": tenant_data["name"],
                "plan": plan_id, "status": "active"}

    def migrate_tenant_data(self, source_tenant_id, target_tenant_id,
                            migration_config):
        """迁移租户数据"""
        migration_id = str(uuid4())

        # 1. 验证目标租户
        target = self.db.get_tenant(target_tenant_id)
        if not target or target["status"] != "active":
            return {"status": "target_invalid"}

        # 2. 获取源租户数据表列表
        tables = migration_config.get("tables", [])
        if not tables:
            tables = self._get_tenant_tables(source_tenant_id)

        # 3. 创建迁移任务
        self.db.insert("tenant_migrations", {
            "migration_id": migration_id,
            "source_tenant_id": source_tenant_id,
            "target_tenant_id": target_tenant_id,
            "tables": json.dumps(tables),
            "status": "in_progress",
            "started_at": now()
        })

        total_migrated = 0
        errors = []

        # 4. 逐表迁移
        for table_name in tables:
            try:
                # 读取源数据
                source_data = self.db.query(
                    f"SELECT * FROM {table_name} WHERE tenant_id = %s",
                    source_tenant_id)

                # 写入目标（替换 tenant_id）
                for row in source_data:
                    row["tenant_id"] = target_tenant_id
                    row["id"] = str(uuid4())  # 新 ID 避免冲突

                    self.db.insert(table_name, row)
                    total_migrated += 1

            except Exception as e:
                errors.append({
                    "table": table_name,
                    "error": str(e)
                })

        # 5. 更新迁移状态
        status = "completed" if not errors else "completed_with_errors"
        self.db.update("tenant_migrations",
            {"status": status, "total_migrated": total_migrated,
             "errors": json.dumps(errors), "completed_at": now()},
            {"migration_id": migration_id})

        return {"migration_id": migration_id, "status": status,
                "total_migrated": total_migrated, "errors": errors}

    def hibernate_tenant(self, tenant_id, reason=None):
        """休眠租户（降低成本但保留数据）"""
        tenant = self.db.get_tenant(tenant_id)

        if tenant["status"] != "active":
            return {"status": "cannot_hibernate", "current": tenant["status"]}

        # 1. 检查是否有未完成的订单/账单
        pending_bills = self.db.count("tenant_bills",
            tenant_id=tenant_id, status="unpaid")

        if pending_bills > 0:
            return {"status": "has_unpaid_bills", "count": pending_bills}

        # 2. 归档热数据到冷存储
        self._archive_tenant_hot_data(tenant_id)

        # 3. 释放计算资源
        self.redis.delete(f"tenant_limit:{tenant_id}:*")
        self.redis.delete(f"tenant_usage:{tenant_id}:*")

        # 4. 更新状态
        self.db.update("tenants",
            {"status": "hibernated", "hibernated_at": now(),
             "hibernate_reason": reason},
            {"id": tenant_id})

        # 5. 通知管理员
        self.notification.send(tenant["admin_user_id"],
            "您的账户已进入休眠状态，数据已安全保存，可随时恢复")

        return {"tenant_id": tenant_id, "status": "hibernated"}

    def reactivate_tenant(self, tenant_id):
        """恢复休眠租户"""
        tenant = self.db.get_tenant(tenant_id)

        if tenant["status"] != "hibernated":
            return {"status": "not_hibernated"}

        # 1. 恢复热数据
        self._restore_tenant_hot_data(tenant_id)

        # 2. 恢复资源限制
        plan = self.PLAN_DETAILS.get(tenant["plan"], self.PLAN_DETAILS["starter"])
        for resource_type, limit in plan["resources"].items():
            self.redis.set(f"tenant_limit:{tenant_id}:{resource_type}", limit)

        # 3. 更新状态
        self.db.update("tenants",
            {"status": "active", "reactivated_at": now()},
            {"id": tenant_id})

        # 4. 通知管理员
        self.notification.send(tenant["admin_user_id"],
            "您的账户已恢复，可以正常使用")

        return {"tenant_id": tenant_id, "status": "active"}

    def delete_tenant(self, tenant_id, confirmation_code):
        """注销租户（永久删除）"""
        tenant = self.db.get_tenant(tenant_id)

        # 1. 验证确认码
        expected_code = hashlib.sha256(
            f"{tenant_id}:{tenant['created_at']}".encode()).hexdigest()[:8]

        if confirmation_code != expected_code:
            return {"status": "invalid_confirmation_code"}

        # 2. 检查是否在休眠状态
        if tenant["status"] not in ["hibernated", "active"]:
            return {"status": "cannot_delete", "current": tenant["status"]}

        # 3. 30 天保留期
        scheduled_deletion = now() + timedelta(days=30)

        self.db.update("tenants",
            {"status": "pending_deletion",
             "scheduled_deletion_at": scheduled_deletion},
            {"id": tenant_id})

        # 4. 通知
        self.notification.send(tenant["admin_user_id"],
            f"您的账户将在 30 天后永久删除（{scheduled_deletion.strftime('%Y-%m-%d')}），"
            f"期间可联系客服恢复")

        return {"tenant_id": tenant_id, "status": "pending_deletion",
                "scheduled_deletion_at": scheduled_deletion.isoformat()}

    def _initialize_tenant_schema(self, tenant_id):
        """初始化租户数据空间"""
        default_tables = [
            "tenant_users", "tenant_roles", "tenant_permissions",
            "tenant_settings", "tenant_audit_log"
        ]

        for table in default_tables:
            self.db.insert(table, {
                "tenant_id": tenant_id,
                "created_at": now()
            })

    def _create_default_roles(self, tenant_id):
        """创建默认角色"""
        default_roles = [
            {"name": "admin", "permissions": ["all"]},
            {"name": "editor", "permissions": ["read", "write"]},
            {"name": "viewer", "permissions": ["read"]},
        ]

        for role in default_roles:
            self.db.insert("tenant_roles", {
                "role_id": str(uuid4()),
                "tenant_id": tenant_id,
                "name": role["name"],
                "permissions": json.dumps(role["permissions"]),
                "is_default": True,
                "created_at": now()
            })

    def _get_tenant_tables(self, tenant_id):
        """获取租户数据表"""
        return [r["table_name"] for r in self.db.query(
            "SELECT DISTINCT table_name FROM information_schema.columns "
            "WHERE column_name = 'tenant_id'")]

    def _archive_tenant_hot_data(self, tenant_id):
        """归档热数据"""
        tables = self._get_tenant_tables(tenant_id)
        for table in tables:
            data = self.db.query(f"SELECT * FROM {table} WHERE tenant_id = %s", tenant_id)
            archive_key = f"archive:{tenant_id}:{table}"
            self.object_storage.upload(archive_key, json.dumps(data, default=str).encode())

    def _restore_tenant_hot_data(self, tenant_id):
        """恢复热数据"""
        tables = self._get_tenant_tables(tenant_id)
        for table in tables:
            archive_key = f"archive:{tenant_id}:{table}"
            if self.object_storage.exists(archive_key):
                data = json.loads(self.object_storage.download(archive_key))
                for row in data:
                    self.db.insert(table, row)
```

## 异常场景补充

### 场景：租户数据迁移过程中断

```
触发：迁移 10 万条记录到第 7 万条时数据库连接断开 → 部分数据已迁移 → 数据不一致
检测：
  1. 迁移任务状态长时间为 in_progress → 可能中断
  2. 目标租户数据量与源租户不一致 → 迁移不完整
处理：
  1. 迁移使用事务（每批 1000 条一个事务）
  2. 中断后从断点续传（记录已迁移的最大 ID）
  3. 迁移完成后校验记录数
预防：批事务 + 断点续传 + 数据校验
```

### 场景：休眠租户被误恢复

```
触发：管理员误操作恢复了已休眠的租户 → 租户开始产生资源消耗 → 费用问题
检测：
  1. 休眠租户突然变为 active → 可能误恢复
  2. 恢复操作未经授权 → 误操作
处理：
  1. 恢复操作需租户管理员确认
  2. 二次确认机制
  3. 审计日志追溯操作者
预防：确认机制 + 二次确认 + 审计追溯
```

### 场景：租户注销后数据被意外删除

```
触发：租户申请注销 → 30 天保留期未到 → 运维误删数据 → 无法恢复
检测：
  1. 保留期未满但数据已删除 → 提前删除
  2. 数据删除操作无对应审批 → 违规
处理：
  1. 数据删除需系统自动执行（非人工）
  2. 保留期到期后才自动清理
  3. 删除前创建最终备份
预防：自动删除 + 保留期保护 + 最终备份
```
