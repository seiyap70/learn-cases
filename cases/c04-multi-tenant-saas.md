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

```python
class TenantMiddleware:
    """每个请求设置租户上下文"""

    def process_request(self, request):
        # 1. 从 JWT Token 中提取租户 ID
        token = request.headers.get("Authorization")
        payload = jwt.decode(token, secret_key)
        tenant_id = payload["tenant_id"]

        # 2. 验证用户确实属于该租户
        user_id = payload["user_id"]
        user = self.get_user(user_id)
        if user.tenant_id != tenant_id:
            raise ForbiddenError("User does not belong to tenant")

        # 3. 设置数据库租户上下文
        self.db.execute(f"SET app.current_tenant = '{tenant_id}'")

        # 4. 将租户信息存入请求上下文
        request.tenant_id = tenant_id

    def process_response(self, request, response):
        # 请求结束后重置租户上下文（连接归还连接池时）
        self.db.execute("RESET app.current_tenant")
```

**为什么请求结束后要 RESET？**

数据库连接是复用的（连接池）。如果请求 A 设置了 `app.current_tenant = 12345`，请求 B 复用同一个连接时，如果 B 没有设置租户上下文，就会以租户 12345 的身份执行查询——数据泄露！

解决方案：
1. 每次请求结束后 RESET
2. 或使用连接池的 `reset_on_return` 配置

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

**Schema 迁移的挑战：**

5000 个 Schema 的表结构升级需要：
- 逐个 Schema 执行 ALTER TABLE → 太慢
- 并行执行 → 可能部分成功部分失败

```python
class SchemaMigration:
    def migrate_all(self, migration_sql):
        schemas = self.get_all_schemas()  # 5000 个
        
        # 并行迁移，每批 50 个
        for batch in self.chunk(schemas, size=50):
            with ThreadPoolExecutor(max_workers=50) as executor:
                futures = {
                    executor.submit(self.migrate_schema, schema, migration_sql): schema
                    for schema in batch
                }
                
                for future in as_completed(futures):
                    schema = futures[future]
                    try:
                        future.result()
                    except Exception as e:
                        # 记录失败的 Schema，后续重试
                        self.record_migration_failure(schema, e)

    def migrate_schema(self, schema, migration_sql):
        conn = self.get_connection(schema)
        try:
            conn.execute(migration_sql)
        except Exception:
            # 回滚该 Schema 的迁移
            conn.rollback()
            raise
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

### 租户升级/降级

当小租户成长为中/大租户时，需要迁移数据：

```python
class TenantMigrator:
    def upgrade_tier(self, tenant_id, from_tier, to_tier):
        if from_tier == "tier3" and to_tier == "tier2":
            # 共享表 → 独立Schema
            self.migrate_shared_to_schema(tenant_id)
        elif from_tier == "tier2" and to_tier == "tier1":
            # 独立Schema → 独立数据库
            self.migrate_schema_to_dedicated(tenant_id)

    def migrate_shared_to_schema(self, tenant_id):
        """从共享表迁移到独立Schema"""
        # 1. 创建目标 Schema
        self.db.execute(f"CREATE SCHEMA tenant_{tenant_id}")
        self.db.execute(f"CREATE TABLE tenant_{tenant_id}.projects (...)")
        # ...

        # 2. 迁移数据（双写期间）
        # 开启双写：新数据同时写入共享表和独立Schema
        self.enable_dual_write(tenant_id)

        # 3. 迁移历史数据
        self.db.execute(f"""
            INSERT INTO tenant_{tenant_id}.projects
            SELECT * FROM projects WHERE tenant_id = {tenant_id}
        """)

        # 4. 验证数据一致性
        self.verify_migration(tenant_id)

        # 5. 切换读取到新 Schema
        self.routing_table.update(tenant_id, tier="tier2", schema_name=f"tenant_{tenant_id}")

        # 6. 关闭双写
        self.disable_dual_write(tenant_id)

        # 7. 删除共享表中的旧数据
        self.db.execute(f"DELETE FROM projects WHERE tenant_id = {tenant_id}")
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

## 常见陷阱（深度分析）

### 陷阱 1：应用层过滤而非数据库强制

**具体的数据泄露构造：**

```python
# 开发者写了这个 API，忘记加 tenant_id 过滤
def get_project(project_id):
    return db.query("SELECT * FROM projects WHERE id = %s", project_id)
    # 如果 project_id 属于其他租户 → 数据泄露！
```

RLS 方案下，即使 SQL 没有 `WHERE tenant_id = ?`，数据库也会自动添加，不会泄露。

### 陷阱 2：连接池未重置租户上下文

**具体场景：**
```
请求A（租户1001）→ SET app.current_tenant = 1001 → 查询 → 连接归还
请求B（租户1002）→ 复用同一连接 → 未 SET → 以租户1001身份查询 → 数据泄露！
```

### 陷阱 3：所有租户用独立数据库

5000 个 MySQL 实例的运维噩梦：
- 每次升级需要逐个实例执行
- 监控告警需要覆盖 5000 个实例
- 备份策略需要管理 5000 个备份任务
- 连接池管理需要 5000 个连接池

2 人运维团队根本无法承受。

### 陷阱 4：Schema 迁移不原子

5000 个 Schema 的 ALTER TABLE，如果第 3000 个失败：
- 前 2999 个已升级，后 2001 个未升级
- 应用代码已经是新版本，但部分 Schema 还是旧结构 → 运行时错误

**解决方案：** 迁移必须幂等，失败的 Schema 可以重试。应用代码必须兼容新旧两个版本的 Schema（双版本兼容期）。

## 延伸思考

- **租户自定义字段**：某些租户要求在 projects 表上加自定义字段（如"审批人"）。共享表模式下如何支持？方案：JSONB 扩展字段 + GIN 索引。
- **跨租户数据分析**：平台运营需要分析所有租户的汇总数据（如"全平台项目创建趋势"），但租户数据是隔离的。方案：只读副本 + 脱敏聚合管道。
- **合规审计**：金融租户要求记录每一次数据访问的审计日志。如何在 RLS 层面实现？方案：PostgreSQL 审计扩展（pgaudit）+ 自定义审计日志。