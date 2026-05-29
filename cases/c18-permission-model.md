# C18: 企业权限的 RBAC+ABAC 混合模型

## 业务场景

某大型企业内部平台，管理 5000 名员工的系统权限。企业权限远比"管理员/普通用户"复杂——同一员工在不同部门有不同权限（矩阵式管理），权限受数据范围约束（华东区销售只能看华东区客户），临时权限需要自动回收。

**一个真实的权限事故：** 2023 年某公司财务系统用纯 RBAC 模型——"财务角色"可以查看所有客户的合同金额。但一个被分配了"财务角色"的实习生应该只能看自己负责区域的客户数据，纯 RBAC 无法表达"同一角色只能看指定区域数据"这个约束 → 实习生看到了全国客户数据 → 数据泄露。

**已知数据：**
- 员工数：5000
- 部门数：200
- 角色：50 个（销售经理、区域总监、财务审批等）
- 权限点：500 个（查看客户、编辑合同、审批退款等）
- 矩阵管理：约 30% 员工跨部门任职
- 临时授权：约 200 次/天（项目协作、代理审批）
- 权限判定延迟：< 50ms（每个 API 请求都要判定）

**为什么纯 RBAC 不够？**

RBAC（基于角色的访问控制）的核心是"用户→角色→权限"。但它无法表达：
- 数据范围约束："华东区销售经理"vs"华南区销售经理"→ 同一角色，不同数据范围
- 临时权限："张三代理李四审批 3 天"→ 3 天后自动回收
- 上下文约束："只能在公司内网查看薪资数据"→ 纯 RBAC 不考虑请求上下文

## 核心挑战

### 挑战 1：数据范围约束

"销售经理"角色可以查看客户列表，但华东区的销售经理只能看华东区客户。纯 RBAC 无法表达这个约束——它只能控制"能不能看客户列表"，不能控制"能看哪些客户"。

**数据范围的类型：**

| 范围类型 | 例子 | 实现复杂度 |
|---------|------|-----------|
| 区域范围 | 只看华东区客户 | 中（需要区域-用户映射） |
| 部门范围 | 只看本部门员工 | 低（直接用部门 ID 过滤） |
| 自定义范围 | 只看我负责的 50 个客户 | 高（需要客户-用户映射） |
| 上下级范围 | 经理看下属的数据 | 中（需要组织架构树） |

### 挑战 2：矩阵式管理

30% 员工跨部门任职 → 在 A 部门是普通员工，在 B 部门是经理。纯 RBAC 的"一个角色"无法同时表达这两个身份。

### 挑战 3：临时权限与自动回收

张三代理李四审批 3 天 → 3 天后权限应自动失效。纯 RBAC 的角色分配是永久的，没有过期机制。

### 挑战 4：权限判定的性能

每个 API 请求都要判定权限 → 5000 员工 × 日均 1000 请求 = 500 万次/天权限判定。每次判定必须 < 50ms。

## 设计约束

- 权限判定延迟 < 50ms
- 临时权限到期自动回收（无延迟）
- 权限变更 5 秒内生效
- 支持审计：谁在什么时间访问了什么数据

## 请先独立思考（限时 30 分钟）

1. RBAC 和 ABAC 各自的优缺点？为什么需要混合？
2. 数据范围约束如何在代码中实现？SQL 行级过滤 vs 应用层过滤？
3. 临时权限的存储和过期机制？Redis TTL vs 定时任务？
4. 权限判定如何做到 < 50ms？缓存策略？

---

## 设计解析

### RBAC+ABAC 混合模型

**核心原则：RBAC 管"能不能做"，ABAC 管"能做到什么范围"。**

```
权限判定 = RBAC判定（角色是否有该权限？）+ ABAC判定（属性是否满足约束？）

示例：
  "查看客户列表"权限：
    RBAC: 用户是否有"销售经理"角色？ → 有 → 可以查看
    ABAC: 用户的区域属性 == 客户的区域？ → 过滤数据范围
```

### 数据模型

```sql
-- 用户表
CREATE TABLE users (
    user_id VARCHAR(64) PRIMARY KEY,
    name VARCHAR(128),
    department_id VARCHAR(64),
    region VARCHAR(32)    -- 华东/华南/华北...
);

-- 角色表
CREATE TABLE roles (
    role_id VARCHAR(64) PRIMARY KEY,
    name VARCHAR(128),     -- 销售经理/财务审批/区域总监
    description TEXT
);

-- 权限表
CREATE TABLE permissions (
    permission_id VARCHAR(64) PRIMARY KEY,
    resource VARCHAR(64),   -- customer/contract/salary
    action VARCHAR(32),     -- read/write/approve/delete
    description TEXT
);

-- 用户-角色绑定（含有效期）
CREATE TABLE user_roles (
    user_id VARCHAR(64),
    role_id VARCHAR(64),
    scope_rules JSON,        -- ABAC 规则：{"region": "east", "level": "<=3"}
    granted_at TIMESTAMP,
    expires_at TIMESTAMP,    -- NULL 表示永久
    granted_by VARCHAR(64),
    PRIMARY KEY (user_id, role_id)
);

-- 角色-权限绑定
CREATE TABLE role_permissions (
    role_id VARCHAR(64),
    permission_id VARCHAR(64),
    scope_template JSON,     -- 数据范围模板
    PRIMARY KEY (role_id, permission_id)
);
```

### 权限判定引擎

```python
class PermissionEngine:
    def check(self, user_id, resource, action, context=None):
        """权限判定：RBAC + ABAC"""
        context = context or {}

        # 1. RBAC 判定：用户是否有该权限？
        user_perms = self.get_user_permissions(user_id)
        perm_key = f"{resource}:{action}"

        if perm_key not in user_perms:
            return PermissionResult(denied=True, reason="no_permission")

        perm = user_perms[perm_key]

        # 2. ABAC 判定：数据范围约束是否满足？
        if perm.scope_rules:
            scope_filter = self.evaluate_scope(perm.scope_rules, context)
            if not scope_filter:
                return PermissionResult(denied=True, reason="scope_violation")
            return PermissionResult(
                denied=False,
                scope_filter=scope_filter  # 返回 SQL 过滤条件
            )

        return PermissionResult(denied=False)

    def get_user_permissions(self, user_id):
        """获取用户的所有权限（带缓存）"""
        cache_key = f"perms:{user_id}"
        cached = self.redis.get(cache_key)
        if cached:
            return json.loads(cached)

        # 查数据库：用户的所有角色 + 角色的所有权限
        user_roles = self.db.query("""
            SELECT r.role_id, ur.scope_rules, ur.expires_at
            FROM user_roles ur
            JOIN roles r ON ur.role_id = r.role_id
            WHERE ur.user_id = %s
        """, user_id)

        permissions = {}
        now = datetime.now()
        for role in user_roles:
            # 跳过过期角色
            if role.expires_at and role.expires_at < now:
                continue

            role_perms = self.db.query("""
                SELECT p.resource, p.action, rp.scope_template
                FROM role_permissions rp
                JOIN permissions p ON rp.permission_id = p.permission_id
                WHERE rp.role_id = %s
            """, role.role_id)

            for perm in role_perms:
                key = f"{perm.resource}:{perm.action}"
                # 合并 scope_rules（角色级 + 用户级）
                scope = self.merge_scope(perm.scope_template, role.scope_rules)
                permissions[key] = PermissionWithScope(scope_rules=scope)

        # 缓存 5 秒（权限变更 5 秒内生效）
        self.redis.setex(cache_key, 5, json.dumps(permissions))
        return permissions

    def evaluate_scope(self, scope_rules, context):
        """ABAC 规则评估 → 返回 SQL 过滤条件"""
        filters = []

        for rule in scope_rules:
            if rule.type == "region":
                # 用户的区域 = 数据的区域
                filters.append(f"region = '{context.get('user_region')}'")
            elif rule.type == "department":
                filters.append(f"department_id = '{context.get('user_department')}'")
            elif rule.type == "subordinate":
                # 只看下级
                subordinate_ids = self.get_subordinates(context.get('user_id'))
                filters.append(f"owner_id IN ({','.join(subordinate_ids)})")
            elif rule.type == "custom":
                # 自定义客户列表
                customer_ids = self.get_assigned_customers(context.get('user_id'))
                filters.append(f"customer_id IN ({','.join(customer_ids)})")

        if not filters:
            return None

        return " AND ".join(filters)
```

**权限判定的性能：**

| 步骤 | 延迟 | 优化 |
|------|------|------|
| Redis 缓存读取 | < 1ms | 5 秒缓存 |
| 数据库查询（缓存未命中） | 5-10ms | 极少发生 |
| ABAC 规则评估 | < 1ms | 纯内存计算 |
| **总计（命中缓存）** | **< 2ms** | 远低于 50ms |

### 临时权限与自动回收

```python
class TempPermissionService:
    def grant_temp(self, user_id, role_id, duration_hours, granted_by, reason):
        """授予临时权限"""
        expires_at = now() + timedelta(hours=duration_hours)

        self.db.execute("""
            INSERT INTO user_roles (user_id, role_id, scope_rules, granted_at, expires_at, granted_by)
            VALUES (%s, %s, '{}', NOW(), %s, %s)
        """, user_id, role_id, expires_at, granted_by)

        # 记录审计日志
        self.audit_log(user_id, "temp_role_granted", {
            "role_id": role_id,
            "duration_hours": duration_hours,
            "granted_by": granted_by,
            "reason": reason,
            "expires_at": expires_at.isoformat()
        })

        # 清除权限缓存（5 秒内生效）
        self.redis.delete(f"perms:{user_id}")

        # 设置 Redis 过期事件（到期自动回收）
        self.redis.setex(
            f"temp_role:{user_id}:{role_id}",
            duration_hours * 3600,
            json.dumps({"user_id": user_id, "role_id": role_id})
        )

    def on_temp_expired(self, user_id, role_id):
        """临时权限到期 → 自动回收"""
        self.db.execute("""
            DELETE FROM user_roles
            WHERE user_id = %s AND role_id = %s AND expires_at IS NOT NULL
        """, user_id, role_id)

        self.redis.delete(f"perms:{user_id}")
        self.audit_log(user_id, "temp_role_expired", {"role_id": role_id})
```

**过期检测方案对比：**

| 方案 | 延迟 | 可靠性 | 复杂度 |
|------|------|-------|-------|
| Redis Key 过期通知 | < 1 秒 | 中（可能丢失通知） | 低 |
| 定时任务每分钟扫描 | 最多 1 分钟 | 高 | 中 |
| 数据库查询时过滤 | 实时 | 最高 | 低 |

选择"数据库查询时过滤 + 定时任务兜底"：查询时 `WHERE expires_at IS NULL OR expires_at > NOW()` → 即使 Redis 过期通知丢失，用户也无法使用过期权限。

### 数据范围过滤：SQL 行级 vs 应用层

```python
class ScopedQueryService:
    def query_with_scope(self, user_id, resource, action, base_query):
        """带数据范围过滤的查询"""
        perm = self.permission_engine.check(user_id, resource, action,
                                            context={"user_id": user_id})
        if perm.denied:
            raise PermissionDenied(perm.reason)

        if perm.scope_filter:
            # 在 SQL 层面过滤（而非应用层）
            # 好处：数据库只返回有权限的数据 → 减少网络传输和内存占用
            scoped_query = f"{base_query} WHERE {perm.scope_filter}"
            return self.db.query(scoped_query)

        return self.db.query(base_query)

# 使用示例
results = service.query_with_scope(
    user_id="U-123",
    resource="customer",
    action="read",
    base_query="SELECT * FROM customers"
)
# 生成: SELECT * FROM customers WHERE region = 'east'
```

**SQL 行级过滤 vs 应用层过滤：**

| 维度 | SQL 行级 | 应用层 |
|------|---------|-------|
| 数据库负载 | 低（只返回有权限数据） | 高（全量返回再过滤） |
| 网络传输 | 少 | 多 |
| 内存占用 | 低 | 高 |
| SQL 注入风险 | 中（需参数化） | 无 |
| 灵活性 | 中 | 高 |

选择 SQL 行级过滤：参数化查询防止注入，数据库只返回有权限的数据。

### 审计日志

```python
class AuditLogger:
    def log_access(self, user_id, resource, action, result, details=None):
        """记录每次权限判定结果"""
        self.kafka.produce("audit_log", {
            "user_id": user_id,
            "resource": resource,
            "action": action,
            "result": "allowed" if result else "denied",
            "scope_filter": details.get("scope_filter") if details else None,
            "timestamp": now_ms(),
            "ip": details.get("ip") if details else None,
            "user_agent": details.get("user_agent") if details else None
        })

    def query_audit(self, user_id=None, resource=None, start_time=None, end_time=None):
        """查询审计日志（合规检查用）"""
        query = "SELECT * FROM audit_logs WHERE 1=1"
        params = []
        if user_id:
            query += " AND user_id = %s"
            params.append(user_id)
        if resource:
            query += " AND resource = %s"
            params.append(resource)
        if start_time:
            query += " AND timestamp >= %s"
            params.append(start_time)
        if end_time:
            query += " AND timestamp <= %s"
            params.append(end_time)
        return self.db.query(query + " ORDER BY timestamp DESC LIMIT 1000", *params)
```

## 常见陷阱（深度分析）

### 陷阱 1：纯 RBAC 无法表达数据范围

**后果：** "财务角色"可以查看所有客户合同 → 实习生看到全国数据 → 数据泄露。每次出现此类问题就创建新角色（"华东财务"、"华南财务"）→ 角色爆炸（50 → 200+）。

**解决方案：** RBAC+ABAC 混合。RBAC 管"能不能看合同"，ABAC 管"能看哪些区域的合同"。

### 陷阱 2：临时权限不自动回收

**后果：** 张三代理审批 3 天 → 3 天后仍能审批 → 越权操作。如果权限是审批付款 → 可能导致资金损失。

**解决方案：** expires_at 字段 + 查询时过滤 + 定时任务兜底。

### 陷阱 3：权限缓存不同步

**后果：** 管理员撤销了某用户的权限 → 但缓存 5 分钟未更新 → 用户仍能操作 → 安全漏洞。

**解决方案：** 权限变更时主动删除缓存（`redis.delete`）→ 下次请求重新加载 → 5 秒内生效。

### 陷阱 4：应用层过滤导致数据泄露

**后果：** 查询全量数据 → 应用层过滤 → 全量数据经过应用层内存 → 内存 dump 可能泄露全部数据。

**解决方案：** SQL 行级过滤 → 数据库只返回有权限的数据 → 应用层不接触无权限数据。

### 陷阱 5：不做审计日志

**后果：** 发生数据泄露后无法追溯 → 不知道谁在什么时间看了什么数据 → 合规审计失败 → 罚款。

**解决方案：** 每次权限判定都记录审计日志（Kafka → ES/HDFS），保留至少 1 年。

## 延伸思考

- **权限自服务**：员工自助申请权限 → 审批流 → 自动授权 → 到期自动回收。减少管理员手动操作。
- **权限分析**：定期扫描"谁有过多的权限"→ 最小权限原则。如某员工有"管理员"角色但只用到了 3 个权限 → 建议收窄。
- **零信任**：每次请求都验证权限（而非只验证一次），结合设备信任度、网络位置等上下文。