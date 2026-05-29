# C25: 企业级 DevOps 平台的 CI/CD 系统

## 业务场景

某大型互联网公司建设内部 DevOps 平台，支撑 200+ 业务团队的 CI/CD。一次部署事故：2023 年某团队周五晚上手动部署，配置错误导致生产服务中断 2 小时，影响 1000 万用户，营收损失约 500 万元。事故后决定建设自动化 CI/CD 平台，杜绝手动部署。

**已知数据：**
- 业务团队数：200+
- 代码仓库数：5000+
- 日均 CI 任务：5000+
- 日均 CD 任务：500+
- 构建并发峰值：200 个任务同时运行
- 构建资源：CI 集群 50 台（8核16GB）
- 部署目标：Kubernetes（50+ 集群，跨 3 个区域）
- 发布策略：蓝绿、金丝雀、滚动更新
- 部署延迟要求：从代码提交到部署上线 < 15 分钟

**为什么是难题？**

- 200 团队用不同语言（Java/Go/Node/Python/Rust）→ 构建环境异构
- 50+ K8s 集群 → 部署管理复杂
- 手动审批成为瓶颈 → 200 团队 × 5 次部署/天 = 1000 次审批请求/天
- 金丝雀发布需要自动监控和回滚 → 需要与可观测性系统集成

## 核心挑战

### 挑战 1：构建资源调度

200 个并发构建 vs 50 台 8c16G 节点 = 4 个构建/节点 → 资源争抢。大构建（Maven 全量编译）占 8 核 15 分钟 → 小构建（Go 编译 2 分钟）排队等 15 分钟。

### 挑战 2：多集群部署管理

50+ K8s 集群跨 3 个区域。手动 kubectl 操作 → 3 名运维全职维护集群 → 效率低且易出错。

### 挑战 3：金丝雀发布与自动回滚

手动金丝雀：观察 15 分钟 → 发现问题 → 手动回滚 15 分钟 = 30 分钟影响时间。
自动金丝雀：观察 5 分钟 → 发现问题 → 自动回滚 30 秒 = 5.5 分钟影响时间。

### 挑战 4：审批效率

200 团队 × 5 次/天 = 1000 次审批/天。审批平均等待 30 分钟 → 部署总时长从 5 分钟变成 35 分钟。

## 设计约束

- 构建超时上限：60 分钟
- 单团队最大并发构建：10
- 镜像安全扫描：< 5 分钟
- 审批超时：30 分钟（超时自动拒绝）
- 资源超配比：<= 1.5x
- 金丝雀回滚延迟：< 60 秒

## 请先独立思考（限时 30 分钟）

1. 200 个构建同时运行，大构建占 8 核 15 分钟，小构建占 2 核 2 分钟——如何调度避免小构建被大构建饿死？
2. 50+ K8s 集群如何统一管理？GitOps vs Push 模式？
3. 金丝雀回滚如果本身也失败了怎么办？
4. 审批如何既保证安全又不过度阻碍效率？

---

## 设计解析

### 端到端 CI/CD 流水线

```
代码提交 → Webhook → CI Pipeline → CD Pipeline → 生产环境
                         ↓                    ↓
                    构建+测试+扫描       审批+金丝雀+全量发布

CI Pipeline:
  1. 拉取代码 → 2. 构建 → 3. 单元测试 → 4. 镜像打包 → 5. 安全扫描

CD Pipeline:
  6. 审批请求 → 7. 金丝雀部署(5%流量) → 8. 监控观察(5分钟)
  → 9. 逐步扩量(5%→25%→50%→100%) → 10. 发布完成
```

### 构建调度器：优先级 + 资源限额

```python
class CIScheduler:
    def __init__(self, nodes):
        self.nodes = nodes  # 50 台 8c16G 节点
        self.job_queue = PriorityQueue()
        self.running_jobs = {}

    def submit(self, job):
        """提交构建任务"""
        # 资源限额：单团队最多 10 个并发构建
        team_running = sum(1 for j in self.running_jobs.values() 
                          if j.team_id == job.team_id)
        if team_running >= 10:
            return {"status": "rejected", "reason": "team_concurrent_limit"}

        # 计算优先级
        priority = self.calculate_priority(job)
        self.job_queue.put((priority, job))
        return {"status": "queued", "position": self.job_queue.qsize()}

    def calculate_priority(self, job):
        """优先级：主分支 > 热修复 > 功能分支"""
        priority_map = {
            "main": 100,
            "hotfix": 90,
            "release": 80,
            "feature": 50,
        }
        base = priority_map.get(job.branch_type, 50)
        # 等待时间越长优先级越高（防止饿死）
        wait_time_bonus = min(50, job.wait_minutes)
        return base + wait_time_bonus

    def schedule(self):
        """调度循环：为等待中的任务分配节点"""
        while not self.job_queue.empty():
            _, job = self.job_queue.get()
            
            # 找一个有足够资源的节点
            node = self.find_available_node(job.cpu_needed, job.memory_needed)
            if node:
                self.run_job(job, node)
            else:
                # 无可用节点 → 放回队列（优先级降低以避免反复调度）
                self.job_queue.put((0, job))
                break

    def find_available_node(self, cpu, memory):
        """寻找有足够资源的节点（考虑超配 1.5x）"""
        for node in self.nodes:
            used_cpu = sum(j.cpu_needed for j in self.running_jobs.get(node.id, []))
            used_mem = sum(j.memory_needed for j in self.running_jobs.get(node.id, []))
            
            if (used_cpu + cpu <= node.total_cpu * 1.5 and 
                used_mem + memory <= node.total_memory * 1.5):
                return node
        return None
```

**调度效果：**

| 指标 | 无调度（FIFO） | 优先级调度 |
|------|-------------|-----------|
| 小构建平均等待 | 15 分钟 | 2 分钟 |
| 大构建平均等待 | 10 分钟 | 12 分钟 |
| 资源利用率 | 60% | 85% |
| 饿死率 | 20%（小构建等超时） | < 1% |

### 构建环境：容器化 + 缓存

```yaml
# pipeline.yaml — 流水线配置
version: "2"
stages:
  - name: build
    image: golang:1.22
    cache:
      - /go/pkg/mod     # Go module 缓存
      - /root/.cache    # 构建缓存
    steps:
      - run: go mod download
      - run: go build -o app
      - run: go test -v ./...

  - name: package
    image: docker:24
    steps:
      - run: docker build -t registry.example.com/app:${CI_COMMIT_SHA} .

  - name: scan
    image: trivy:latest
    steps:
      - run: trivy image registry.example.com/app:${CI_COMMIT_SHA}
```

**构建缓存效果：**

| 场景 | 无缓存 | 有缓存 | 提升 |
|------|-------|-------|------|
| Go 全量编译 | 8 分钟 | 2 分钟 | 4x |
| Maven 全量编译 | 12 分钟 | 3 分钟 | 4x |
| npm install | 3 分钟 | 30 秒 | 6x |

### 安全扫描：Trivy 镜像扫描

```python
class SecurityScanService:
    def scan(self, image_tag):
        """镜像安全扫描"""
        result = self.trivy.scan(image_tag)
        
        vulnerabilities = {
            "CRITICAL": len([v for v in result.vulnerabilities if v.severity == "CRITICAL"]),
            "HIGH": len([v for v in result.vulnerabilities if v.severity == "HIGH"]),
            "MEDIUM": len([v for v in result.vulnerabilities if v.severity == "MEDIUM"]),
        }
        
        # 任何 CRITICAL 漏洞 → 阻止部署
        if vulnerabilities["CRITICAL"] > 0:
            return ScanResult(
                passed=False,
                reason=f"发现 {vulnerabilities['CRITICAL']} 个严重漏洞",
                details=result.vulnerabilities
            )
        
        # HIGH 漏洞 > 5 → 阻止部署
        if vulnerabilities["HIGH"] > 5:
            return ScanResult(
                passed=False,
                reason=f"发现 {vulnerabilities['HIGH']} 个高危漏洞"
            )
        
        return ScanResult(passed=True, vulnerabilities=vulnerabilities)
```

### 金丝雀发布：Istio 流量切换 + 自动回滚

```python
class CanaryDeployer:
    def deploy_canary(self, service, new_version):
        """金丝雀部署"""
        # 1. 部署新版本 Pod（0 副本 → 1 副本）
        self.k8s.scale_deployment(service, new_version, replicas=1)
        
        # 2. Istio 配置 5% 流量到新版本
        self.istio.set_weight(service, {
            "stable": 95,
            "canary": 5
        })
        
        # 3. 观察期（5 分钟）
        self.observe_canary(service, duration_minutes=5)

    def observe_canary(self, service, duration_minutes):
        """观察金丝雀指标"""
        start_time = now()
        stable_baseline = self.get_metrics(service, "stable")
        
        while (now() - start_time).seconds < duration_minutes * 60:
            canary_metrics = self.get_metrics(service, "canary")
            
            # 检查错误率
            if canary_metrics.error_rate > stable_baseline.error_rate * 2:
                self.auto_rollback(service, "错误率翻倍")
                return
            
            # 检查 P99 延迟
            if canary_metrics.p99_latency > stable_baseline.p99_latency * 1.5:
                self.auto_rollback(service, "P99 延迟升高 50%")
                return
            
            sleep(10)  # 每 10 秒检查一次
        
        # 观察期通过 → 逐步扩量
        self.progressive_rollout(service)

    def progressive_rollout(self, service):
        """渐进式扩量"""
        stages = [25, 50, 75, 100]
        
        for weight in stages:
            self.istio.set_weight(service, {"stable": 100-weight, "canary": weight})
            self.observe_canary(service, duration_minutes=3)
            # observe_canary 内部会在异常时自动回滚

    def auto_rollback(self, service, reason):
        """自动回滚"""
        # 1. 流量切回稳定版本
        self.istio.set_weight(service, {"stable": 100, "canary": 0})
        
        # 2. 缩容金丝雀 Pod
        self.k8s.scale_deployment(service, "canary", replicas=0)
        
        # 3. 告警
        self.alert(f"金丝雀自动回滚: {service}, 原因: {reason}")
```

**金丝雀发布 vs 蓝绿发布 vs 滚动更新：**

| 维度 | 蓝绿 | 金丝雀 | 滚动更新 |
|------|------|-------|---------|
| 回滚速度 | 1 秒（切换流量） | 30 秒 | 5 分钟（滚动回退） |
| 资源占用 | 2x（两套环境） | 1.1x（5% 金丝雀） | 1x |
| 影响范围 | 无（瞬间切换） | 5% 用户可能受影响 | 逐步替换 |
| 适用场景 | 关键服务 | 日常发布 | 低风险更新 |

### 审批工作流

```python
class ApprovalService:
    def request_approval(self, deployment):
        """请求部署审批"""
        approval = {
            "id": uuid4(),
            "deployment_id": deployment.id,
            "service": deployment.service,
            "version": deployment.version,
            "requested_by": deployment.requester,
            "status": "PENDING",
            "created_at": now(),
            "expires_at": now() + timedelta(minutes=30),
        }
        
        self.db.insert("approvals", approval)
        
        # 通知审批人
        self.notify_approvers(deployment.team_id, approval)
        
        # 设置超时自动拒绝
        self.schedule_expiry(approval.id, delay_minutes=30)
        
        return approval

    def on_approval_timeout(self, approval_id):
        """审批超时 → 自动拒绝"""
        self.db.update("approvals", 
                      {"status": "EXPIRED"}, 
                      {"id": approval_id})
        self.notify_requester(approval_id, "审批超时，请重新提交")
```

**审批策略：**

| 场景 | 是否需要审批 | 原因 |
|------|------------|------|
| 开发环境部署 | 不需要 | 低风险 |
| 测试环境部署 | 不需要 | 低风险 |
| 生产环境（工作日白天） | 需要 | 高风险 |
| 生产环境（周末/夜间） | 需要更高级别审批 | 高风险+人少 |
| 热修复（紧急） | 自动审批 | 时效性优先 |

### GitOps：ArgoCD 多集群管理

```python
class GitOpsManager:
    """ArgoCD 管理多集群部署"""
    
    def sync_deployment(self, service, version, target_clusters):
        """通过 Git 仓库管理部署状态"""
        # 1. 更新 Git 仓库中的部署清单
        for cluster in target_clusters:
            manifest_path = f"manifests/{cluster.region}/{cluster.name}/{service}.yaml"
            self.update_manifest(manifest_path, {
                "image": f"registry.example.com/{service}:{version}",
                "replicas": self.get_replica_count(cluster, service)
            })
        
        # 2. Git commit + push
        self.git.commit_and_push(f"deploy {service} v{version}")
        
        # 3. ArgoCD 自动检测到变更 → 同步到各集群
        # ArgoCD 每 3 分钟轮询 Git 仓库 → 自动应用变更
        # 无需手动 kubectl apply
```

**GitOps vs Push 模式：**

| 维度 | GitOps (ArgoCD) | Push (kubectl) |
|------|----------------|---------------|
| 审计 | Git log 完整记录 | 需额外审计系统 |
| 回滚 | git revert → 自动回滚 | 手动回滚 |
| 多集群 | 天然支持 | 需要脚本 |
| 延迟 | 3 分钟（轮询） | 即时 |
| 安全 | 只需 Git 写权限 | 需要 K8s 管理员权限 |

## 常见陷阱（深度分析）

### 陷阱 1：大构建饿死小构建

**后果：** 一个 Maven 全量编译占 8 核 15 分钟 → 10 个 Go 编译（2核2分钟）排队等 15 分钟 → 构建总时长从 2 分钟变成 17 分钟 → 开发者体验极差。

**解决方案：** 优先级调度 + 等待时间加权 + 资源限额。

### 陷阱 2：手动金丝雀回滚

**后果：** 发现问题后手动回滚需要 15 分钟 → 5% 用户受影响 15 分钟 = 50 万用户受影响 → 营收损失 ¥50 万。

**解决方案：** 自动化金丝雀 + 自动回滚（30 秒内完成）。

### 陷阱 3：不做镜像安全扫描

**后果：** 含严重漏洞的镜像部署到生产 → 被攻击 → 数据泄露。

**解决方案：** 部署前强制 Trivy 扫描，CRITICAL 漏洞阻止部署。

### 陷阱 4：审批成为瓶颈

**后果：** 1000 次/天审批请求 → 审批人处理不过来 → 平均等待 30 分钟 → 部署总时长 35 分钟 → 违反 15 分钟目标。

**解决方案：** 分级审批（开发/测试环境免审批 + 生产按风险分级）+ 超时自动拒绝。

### 陷阱 5：50+ 集群手动管理

**后果：** 3 名运维全职做 kubectl 操作 → 人均管理 17 个集群 → 容易出错（操作错集群）→ 需要更多人 → 成本高。

**解决方案：** GitOps（ArgoCD）——所有集群通过 Git 仓库管理，变更自动同步。

### 陷阱 6：构建环境不一致

**后果：** 开发者本地构建成功，CI 构建失败 → 排查 2 小时 → 原因是本地 Go 1.22 vs CI Go 1.21。

**解决方案：** 容器化构建环境 + 构建缓存。pipeline.yaml 中声明 image: golang:1.22 → 确保一致性。

## 延伸思考

- **弹性 CI 集群**：CI 节点按需扩缩（30-80 台），闲时缩到 30 台，忙时扩到 80 台 → 月成本从 50 台 × ¥5000 = ¥25 万 → 降到平均 45 台 × ¥5000 = ¥22.5 万。
- **构建缓存共享**：Go module 缓存、Maven 缓存、npm 缓存在 CI 集群内共享 → 构建时间减少 60%。
- **ChatOps**：在 Slack/钉钉中审批部署、查看构建状态 → 减少上下文切换。