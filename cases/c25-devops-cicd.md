# C25: 企业级 DevOps 平台的 CI/CD 系统

## 业务场景

某大型互联网公司建设内部 DevOps 平台，支撑 200+ 业务团队的 CI/CD：

- 代码仓库：GitLab（内部部署）
- 每日构建任务：5000+
- 构建环境：多种语言/框架（Java/Go/Node/Python/Rust）
- 部署目标：Kubernetes（50+ 集群，跨 3 个区域）
- 发布策略：蓝绿、金丝雀、滚动更新

**已知数据：**
- 业务团队数：200+
- 代码仓库数：5000+
- 日均 CI 任务：5000+
- 日均 CD 任务：500+
- 构建并发峰值：200 个任务同时运行
- 构建资源：CI 集群 50 台（8核16GB）
- 部署延迟要求：从代码提交到部署上线 < 15 分钟

## 核心挑战

1. **资源调度** —— 200 个并发构建如何高效分配计算资源？
2. **多环境管理** —— 50+ K8s 集群如何统一管理部署？
3. **金丝雀发布** —— 如何实现自动化金丝雀发布（流量逐步切换 + 自动回滚）
4. **权限与安全** —— 不同团队的代码/部署权限如何隔离？

## 设计约束

- 构建集群使用 Kubernetes（Job 调度）
- 各团队可自定义 CI 流程（Pipeline YAML）
- 部署必须经过审批（生产环境至少 1 人审批）
- 构建产物（镜像/包）存储在 Harbor/Nexus

## 请思考

1. CI 流程定义用什么格式？Jenkinsfile vs GitLab CI vs GitHub Actions vs 自定义 DSL？
2. 构建资源如何弹性调度？K8s Job + HPA？
3. 金丝雀发布的流量切换如何实现？Istio vs Nginx Ingress？
4. 如何设计"部署审批"流程，既保证安全又不拖慢发布速度？

---

## 设计解析

### CI/CD 架构总览

```
代码提交 → GitLab Webhook → CI调度服务 → K8s Job(构建) → Harbor(镜像)
                                                              ↓
                                              CD调度服务 → 金丝雀网关 → K8s集群(部署)
                                                              ↓
                                              监控服务 → 自动回滚/推进决策
```

### CI 流程定义：Pipeline YAML

**为什么选 YAML 而非 Jenkinsfile？**
- YAML 语法简洁，业务团队容易上手
- GitLab CI / GitHub Actions 都是 YAML 格式，生态成熟
- 可以直接在代码仓库中定义（`.gitlab-ci.yml`），与代码同版本管理

**平台增强的 Pipeline YAML：**

```yaml
# .pipeline.yml（平台增强版）
stages:
  - build
  - test
  - security-scan
  - deploy-staging
  - deploy-production

build:
  stage: build
  image: ${PLATFORM_REGISTRY}/builder/java:17
  commands:
    - mvn clean package -DskipTests
  artifacts:
    - path: target/*.jar
      type: jar
    - path: Dockerfile
      type: file
  resources:                    # 资源申请
    cpu: 4
    memory: 8Gi
  timeout: 30m

test:
  stage: test
  image: ${PLATFORM_REGISTRY}/builder/java:17
  commands:
    - mvn test
  parallel: 4                   # 4 个测试任务并行
  resources:
    cpu: 2
    memory: 4Gi

security-scan:
  stage: security-scan
  image: ${PLATFORM_REGISTRY}/scanner/trivy:latest
  commands:
    - trivy fs --exit-code 1 .
  resources:
    cpu: 1
    memory: 2Gi

deploy-staging:
  stage: deploy-staging
  type: rolling                 # 滚动更新
  target:
    cluster: cluster-cn-east
    namespace: ${TEAM}-staging
  approval: false               # staging 不需要审批

deploy-production:
  stage: deploy-production
  type: canary                  # 金丝雀发布
  canary:
    initial_weight: 5           # 初始 5% 流量
    step_weight: 10             # 每步增加 10%
    step_interval: 3m           # 每步观察 3 分钟
    max_weight: 100             # 最终 100%
    rollback_threshold:         # 自动回滚阈值
      error_rate: 1%            # 错误率 > 1%
      latency_p99: 500ms        # P99 延迟 > 500ms
  target:
    cluster: cluster-cn-east
    namespace: ${TEAM}-production
  approval:                     # 需审批
    required: 1                 # 1 人审批
    timeout: 30m                # 30 分钟未审批则取消
```

### CI 调度与资源管理

**构建任务调度：**

```python
class CIScheduler:
    def submit_job(self, pipeline, stage, job_config):
        # 1. 调度到构建集群
        k8s_job = {
            "apiVersion": "batch/v1",
            "kind": "Job",
            "metadata": {
                "name": f"ci-{pipeline.id}-{stage}",
                "labels": {
                    "team": pipeline.team,
                    "pipeline": pipeline.id,
                    "stage": stage
                }
            },
            "spec": {
                "template": {
                    "spec": {
                        "containers": [{
                            "name": "builder",
                            "image": job_config.image,
                            "command": job_config.commands,
                            "resources": {
                                "requests": {
                                    "cpu": job_config.resources.cpu,
                                    "memory": job_config.resources.memory
                                }
                            }
                        }],
                        "restartPolicy": "Never"
                    }
                },
                "backoffLimit": 0  # 不重试（CI 任务失败不自动重试）
            }
        }

        # 2. 提交到 K8s
        self.k8s.create_job(k8s_job)

        # 3. 监控任务状态
        asyncio.create_task(self.watch_job(pipeline.id, stage))
```

**资源弹性调度：**

- 构建集群使用 K8s HPA + Cluster Autoscaler
- 基础节点：30 台（常驻）
- HPA 触发条件：Pending Job > 5 → 扩容新节点
- Cluster Autoscaler：自动申请/释放云主机
- 构建高峰期可扩至 80 台，低谷缩至 30 台

**构建优先级：**
```python
class BuildPriority:
    """高优先级构建优先获得资源"""

    PRIORITY_MAP = {
        "main_branch": 100,      # 主分支构建最高优先级
        "merge_request": 80,     # MR 构建
        "scheduled": 60,         # 定时构建
        "feature_branch": 40,    # 特性分支
        "hotfix": 150,           # 紧急修复最高
    }

    def get_priority(self, pipeline):
        return self.PRIORITY_MAP.get(pipeline.trigger_type, 50)
```

### CD 金丝雀发布

**基于 Istio 的金丝雀发布：**

```
用户请求 → Istio VirtualService → 旧版本(v1, weight:95%) + 新版本(v2, weight:5%)
```

**Istio 配置：**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ${SERVICE}
spec:
  hosts:
    - ${SERVICE}
  http:
    - route:
        - destination:
            host: ${SERVICE}
            subset: v1
          weight: 95
        - destination:
            host: ${SERVICE}
            subset: v2
          weight: 5
```

**金丝雀决策引擎：**

```python
class CanaryDecisionEngine:
    """基于监控指标自动决定推进或回滚"""

    def evaluate(self, canary_config, service):
        # 1. 从 Prometheus 获取指标
        metrics = self.prometheus.query_range(
            query=f"""
              sum(rate(http_requests_total{{service="{service}",version="v2",status=~"5.."}}[3m]))
              /
              sum(rate(http_requests_total{{service="{service}",version="v2"}}[3m]))
            """,
            start=now() - canary_config.step_interval
        )

        error_rate = metrics.error_rate
        latency_p99 = metrics.latency_p99

        # 2. 判断是否需要回滚
        if error_rate > canary_config.rollback_threshold.error_rate:
            return Decision.ROLLBACK, f"Error rate {error_rate} > threshold"

        if latency_p99 > canary_config.rollback_threshold.latency_p99:
            return Decision.ROLLBACK, f"P99 latency {latency_p99}ms > threshold"

        # 3. 推进金丝雀
        current_weight = canary_config.current_weight
        new_weight = min(current_weight + canary_config.step_weight, 100)

        return Decision.PROGRESS, f"Increase weight to {new_weight}"
```

**金丝雀流程：**
```
5% → 观察3分钟 → 指标OK → 15% → 观察3分钟 → 指标OK → 25% → ... → 100%
5% → 观察3分钟 → 错误率 > 1% → 自动回滚到 0%（v1 100%）
```

### 部署审批流程

```python
class ApprovalService:
    def request_approval(self, deployment):
        # 1. 创建审批单
        approval = {
            "id": uuid4(),
            "deployment_id": deployment.id,
            "team": deployment.team,
            "environment": "production",
            "service": deployment.service,
            "version": deployment.version,
            "required_approvals": 1,
            "current_approvals": 0,
            "status": "pending",
            "timeout": now() + timedelta(minutes=30),
            "auto_approve_rules": [
                # 配置变更（非代码变更）可自动审批
                {"change_type": "config", "auto_approve": True},
                # 白名单团队（成熟团队）可自动审批
                {"team": "platform", "auto_approve": True},
            ]
        }

        # 2. 检查是否可以自动审批
        for rule in approval["auto_approve_rules"]:
            if self.matches_rule(deployment, rule):
                self.auto_approve(approval)
                return

        # 3. 通知审批人（团队负责人/运维）
        self.notify_approvers(approval)

    def approve(self, approval_id, approver):
        approval = self.get_approval(approval_id)
        approval.current_approvals += 1

        if approval.current_approvals >= approval.required_approvals:
            approval.status = "approved"
            self.trigger_deployment(approval.deployment_id)
```

### 多集群管理

```python
class ClusterManager:
    """统一管理 50+ K8s 集群"""

    def deploy(self, deployment, cluster_name):
        cluster = self.get_cluster_config(cluster_name)

        # 根据集群类型选择部署策略
        if cluster.type == "istio_enabled":
            return self.deploy_with_canary(deployment, cluster)
        elif cluster.type == "standard":
            return self.deploy_rolling(deployment, cluster)

    def deploy_with_canary(self, deployment, cluster):
        # 1. 部署新版本（v2），初始副本数 = 1
        # 2. 配置 Istio VirtualService（5% 流量到 v2）
        # 3. 启动金丝雀决策引擎
        # 4. 自动推进到 100% 或回滚
```

### 安全与权限

**RBAC 权限模型：**
```
团队管理员：可审批生产部署、管理团队成员
团队开发者：可提交代码、触发 CI/CD、部署 staging
运维管理员：可管理集群、审批所有生产部署
审计员：可查看所有操作日志，不可执行操作
```

**镜像安全：**
- 所有镜像必须经过 Trivy 扫描才能部署到生产
- 扫描报告随 CI 流程自动生成
- 发现高危漏洞 → 阻止部署

## 常见陷阱

| 陷阱 | 说明 |
|------|------|
| CI 不做资源限制 | 大构建任务占用全部资源，其他任务排队 |
| 金丝雀无自动回滚 | 需要人工判断，响应慢，故障持续时间长 |
| 审批流程一刀切 | 小改动也要审批，发布速度变慢 |
| 多集群无统一管理 | 各集群独立操作，版本不一致 |
| 不做镜像安全扫描 | 携带漏洞的镜像部署到生产 |

## 延伸思考

- 如何实现"自动发布"——CI 测试全通过后自动触发 CD，无需人工干预？
- 如何设计构建缓存？同一仓库不同分支的构建如何共享依赖缓存？
- GitOps（ArgoCD/Flux）模式与传统 CI/CD 模式的差异和优劣？