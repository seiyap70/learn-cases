# C18: 企业权限的RBAC+ABAC混合模型

## 业务场景

某大型企业内部平台，管理 5000 名员工的系统权限。企业权限远比"管理员/普通用户"复杂——同一员工在不同部门有不同权限（矩阵式管理），权限受数据范围约束（华东区销售只能看华东区客户），临时权限需要自动回收。

**为什么不能只用 RBAC？**

RBAC 的"用户→角色→权限"模型无法表达：
- "销售经理只能看华东区客户数据"——角色是"销售经理"，但数据范围是"华东区"
- 如果为每个区域创建角色（华东销售经理、华南销售经理...）→ 角色数量爆炸
- 临时权限（项目期间授权）无法自然表达

**为什么不能只用 ABAC？**

ABAC 可以表达任何条件，但：
- 策略极其复杂，难以审计"谁能访问什么"
- 每次访问都要评估多条策略 → 性能问题
- 非技术人员无法理解和配置 ABAC 策略

## 核心挑战

### 挑战 1：角色粒度与数据范围的分离

功能权限（能做什么）和数据范围（对哪些数据做）是两个正交维度。RBAC 管功能权限，ABAC 管数据范围——两者如何统一？

### 挑战 2：临时权限的自动回收

项目期间授权，项目结束后权限应自动回收。如果忘记回收 → "权限蠕变"（权限只增不减），安全风险累积。

### 挑战 3：权限变更的审计

合规要求：每一次权限变更必须记录——谁在什么时间授予了什么权限给谁。

## 设计约束

- 权限检查延迟 < 10ms
- 权限变更即时生效
- 所有权限变更有审计记录
- 支持跨部门矩阵式管理

## 请先独立思考（限时 30 分钟）

1. RBAC 和 ABAC 如何分工？具体到代码层面，权限检查的流程是什么？
2. 数据范围策略如何设计？以"华东区销售只能看华东区客户"为例，数据库查询如何自动加上数据范围过滤？
3. 临时权限到期后如何可靠地自动回收？如果回收服务故障怎么办？

---

## 设计解析

### RBAC + ABAC 分工原则

```
RBAC：决定"能做什么"（功能权限）
  用户 → 角色 → API权限/页面权限

ABAC：决定"对哪些数据做"（数据范围）
  用户 + 角色 + 属性 → 数据过滤条件
```

**具体例子：**

```
张三 → 角色：销售经理
     → 功能权限（RBAC）：查看客户列表、编辑客户、查看销售报表
     → 数据范围（ABAC）：region = "华东"

李四 → 角色：销售经理
     → 功能权限（RBAC）：同上（角色相同）
     → 数据范围（ABAC）：region = "华南"

两人有相同的功能权限，但看到的数据不同。
```

### 数据模型

```sql
-- 用户表
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(64),
    department VARCHAR(64),
    region VARCHAR(32)          -- 用户属性（用于 ABAC）
);

-- 角色表
CREATE TABLE roles (
    id SERIAL PRIMARY KEY,
    name VARCHAR(64) NOT NULL,
    description VARCHAR(256)
);

-- 功能权限表
CREATE TABLE permissions (
    id SERIAL PRIMARY KEY,
    resource VARCHAR(64) NOT NULL,   -- 客户/订单/报表
    action VARCHAR(16) NOT NULL,     -- read/write/delete
    description VARCHAR(256),
    UNIQUE KEY (resource, action)
);

-- 角色-权限关联
CREATE TABLE role_permissions (
    role_id INTEGER NOT NULL,
    permission_id INTEGER NOT NULL,
    PRIMARY KEY (role_id, permission_id)
);

-- 用户-角色关联（支持多角色 + 临时权限）
CREATE TABLE user_roles (
    user_id INTEGER NOT NULL,
    role_id INTEGER NOT NULL,
    granted_by INTEGER NOT NULL,      -- 授权人
    granted_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,             -- NULL=永久，非NULL=临时
    revoke_reason VARCHAR(256),       -- 回收原因
    PRIMARY KEY (user_id, role_id)
);

-- 数据范围策略（ABAC）
CREATE TABLE data_scope_policies (
    id SERIAL PRIMARY KEY,
    role_id INTEGER NOT NULL,
    user_id INTEGER,                  -- NULL=角色级别，非NULL=用户级别覆盖
    resource VARCHAR(64) NOT NULL,    -- 适用资源
    attribute VARCHAR(64) NOT NULL,   -- 过滤属性：region/department/amount
    operator VARCHAR(8) NOT NULL,     -- EQ/IN/BETWEEN
    value VARCHAR(256) NOT NULL,      -- 华东 / 华东,华南 / 0-50000
    priority INTEGER DEFAULT 0,       -- 高优先级覆盖低优先级
    created_at TIMESTAMP DEFAULT NOW()
);

-- 权限审计日志
CREATE TABLE permission_audit_log (
    id BIGSERIAL PRIMARY KEY,
    action VARCHAR(16) NOT NULL,      -- GRANT / REVOKE / EXPIRE
    user_id INTEGER NOT NULL,
    role_id INTEGER,
    granted_by INTEGER,
    reason VARCHAR(256),
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_user_time (user_id, created_at)
);
```

### 权限检查引擎

```python
class PermissionEngine:
    def __init__(self, db, cache):
        self.db = db
        self.cache = cache

    def check(self, user_id, resource, action, context=None):
        """
        权限检查：两步走
        1. RBAC：是否有功能权限？
        2. ABAC：如果有，数据范围是什么？
        """
        # Step 1: RBAC 功能权限检查
        roles = self.get_user_active_roles(user_id)
        has_permission = self.check_permission(roles, resource, action)
        if not has_permission:
            return AccessDenied(f"无 {action} {resource} 权限")

        # Step 2: ABAC 数据范围
        data_filter = self.build_data_filter(user_id, roles, resource)

        return AccessAllowed(data_filter=data_filter)

    def get_user_active_roles(self, user_id):
        """获取用户当前有效的角色（排除过期的临时权限）"""
        return self.db.query("""
            SELECT r.* FROM roles r
            JOIN user_roles ur ON r.id = ur.role_id
            WHERE ur.user_id = %s
              AND (ur.expires_at IS NULL OR ur.expires_at > NOW())
        """, user_id)

    def check_permission(self, roles, resource, action):
        """检查角色集是否有指定功能权限"""
        for role in roles:
            if self.cache.sismember(f"role_perms:{role.id}", f"{resource}:{action}"):
                return True
        return False

    def build_data_filter(self, user_id, roles, resource):
        """
        构建 ABAC 数据范围过滤条件
        合并所有适用策略，高优先级覆盖低优先级
        """
        # 获取所有适用的数据范围策略
        policies = self.db.query("""
            SELECT attribute, operator, value, priority, user_id AS policy_user_id
            FROM data_scope_policies
            WHERE resource = %s
              AND (role_id = ANY(%s) OR user_id = %s)
            ORDER BY priority DESC
        """, resource, [r.id for r in roles], user_id)

        # 合并策略（高优先级覆盖低优先级）
        merged = {}
        for p in policies:
            attr = p.attribute
            if attr not in merged or p.priority > merged[attr]["priority"]:
                merged[attr] = {
                    "operator": p.operator,
                    "value": p.value,
                    "priority": p.priority,
                    "is_user_level": p.policy_user_id is not None
                }

        # 转换为 SQL WHERE 条件
        conditions = []
        for attr, policy in merged.items():
            if policy["operator"] == "EQ":
                conditions.append(f"{attr} = '{policy['value']}'")
            elif policy["operator"] == "IN":
                values = "','".join(policy["value"].split(","))
                conditions.append(f"{attr} IN ('{values}')")
            elif policy["operator"] == "BETWEEN":
                low, high = policy["value"].split("-")
                conditions.append(f"{attr} BETWEEN {low} AND {high}")

        return " AND ".join(conditions) if conditions else "1=1"

    def apply_filter(self, query, data_filter):
        """将数据范围过滤应用到业务查询"""
        return f"{query} WHERE {data_filter}"
```

**使用示例：**

```python
# 张三（销售经理，华东区）查看客户列表
result = engine.check(user_id=101, resource="customer", action="read")
# → AccessAllowed(data_filter="region = '华东'")

# 业务查询自动加上数据范围
customers = db.query(f"SELECT * FROM customers WHERE {result.data_filter}")
# → 只返回华东区的客户

# 李四（销售经理，华南区 + VIP客户权限覆盖）
result = engine.check(user_id=102, resource="customer", action="read")
# → AccessAllowed(data_filter="region = '华南' AND customer_level IN ('normal','VIP')")
```

### 临时权限自动回收

```python
class TempPermissionManager:
    def grant_temp(self, user_id, role_id, granted_by, duration_hours, reason):
        """授予临时权限"""
        expires_at = now() + timedelta(hours=duration_hours)
        
        self.db.execute("""
            INSERT INTO user_roles (user_id, role_id, granted_by, expires_at)
            VALUES (%s, %s, %s, %s)
        """, user_id, role_id, granted_by, expires_at)
        
        # 审计日志
        self.audit("GRANT", user_id, role_id, granted_by, reason)
        
        # 注册延迟回收任务
        self.schedule_revoke(user_id, role_id, expires_at)

    def schedule_revoke(self, user_id, role_id, expires_at):
        """注册定时回收任务"""
        delay = (expires_at - now()).total_seconds()
        self.delay_queue.enqueue(
            task="revoke_temp_permission",
            payload={"user_id": user_id, "role_id": role_id},
            delay_seconds=delay
        )

    def revoke(self, user_id, role_id, reason="临时权限过期自动回收"):
        """回收权限"""
        self.db.execute("""
            DELETE FROM user_roles WHERE user_id = %s AND role_id = %s
        """, user_id, role_id)
        
        # 审计日志
        self.audit("EXPIRE", user_id, role_id, reason=reason)
        
        # 清除缓存
        self.cache.delete(f"user_roles:{user_id}")

    def cleanup_missed(self):
        """
        定时扫描：处理延迟队列遗漏的过期权限
        （兜底机制，防止延迟队列故障导致权限未回收）
        """
        expired = self.db.query("""
            SELECT user_id, role_id FROM user_roles
            WHERE expires_at IS NOT NULL AND expires_at <= NOW()
        """)
        
        for record in expired:
            self.revoke(record.user_id, record.role_id)
```

**双重保障：** 延迟队列主动回收 + 定时扫描兜底 → 即使延迟队列故障，过期权限也不会遗漏。

### 权限缓存策略

```python
class PermissionCache:
    """权限缓存：避免每次请求都查数据库"""

    def get_user_permissions(self, user_id):
        cache_key = f"perms:{user_id}"
        cached = self.redis.get(cache_key)
        if cached:
            return json.loads(cached)
        
        # 查询数据库
        permissions = self.engine.get_all_permissions(user_id)
        self.redis.setex(cache_key, 300, json.dumps(permissions))  # 5 分钟缓存
        return permissions

    def invalidate(self, user_id):
        """权限变更时清除缓存"""
        self.redis.delete(f"perms:{user_id}")
```

## 常见陷阱（深度分析）

### 陷阱 1：纯 RBAC 无数据范围

**后果：** 同是"销售经理"角色的张三和李四，张三能看到华南区客户数据 → 数据泄露。

**解决方案：** RBAC + ABAC 混合，数据范围策略强制过滤。

### 陷阱 2：临时权限不回收

**具体场景：** 项目期间授予开发者"生产数据库写权限"，项目结束后忘记回收 → 开发者长期拥有高危权限 → 安全隐患。

**解决方案：** 临时权限必须设 `expires_at`，延迟队列自动回收 + 定时扫描兜底。

### 陷阱 3：权限检查绕过

**具体场景：** 某个 API 没有调用 `engine.check()` → 任何人都可以访问 → 权限形同虚设。

**解决方案：** 中间件统一拦截，所有 API 请求必须经过权限检查。

### 陷阱 4：数据范围策略不覆盖所有查询

**具体场景：** 列表查询加了 `region = '华东'` 过滤，但详情查询（`/api/customer/{id}`）没有加 → 用户可以通过猜 ID 查看其他区域的客户。

**解决方案：** 详情查询也必须先检查数据范围——从数据库取出记录后，验证记录的 region 是否在用户的数据范围内。

## 延伸思考

- **权限继承**：子部门自动继承父部门的权限。需要部门树 + 权限继承规则，但也要支持"阻断继承"（子部门不继承某条权限）。
- **数据脱敏**：不同角色看到同一字段的不同精度。如销售看手机号后 4 位，运营看完整手机号。在数据范围策略中增加"字段脱敏"维度。
- **审批流**：权限变更需要上级审批。与工作流引擎集成，审批通过后才执行 GRANT。