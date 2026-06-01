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

### 数据库设计

```sql
-- CI 构建任务表
CREATE TABLE ci_jobs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    job_id VARCHAR(64) NOT NULL,
    team_id VARCHAR(32) NOT NULL,
    repo_url VARCHAR(500) NOT NULL,
    branch VARCHAR(200) NOT NULL,
    branch_type VARCHAR(20) NOT NULL,      -- main / hotfix / release / feature
    commit_sha VARCHAR(40) NOT NULL,
    status VARCHAR(20) DEFAULT 'queued',   -- queued / running / success / failed / timeout / cancelled
    cpu_needed INT DEFAULT 2,
    memory_needed_mb INT DEFAULT 4096,
    node_id VARCHAR(32),                    -- 分配的CI节点
    priority INT DEFAULT 50,
    started_at TIMESTAMP,
    finished_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE KEY uk_job (job_id),
    INDEX idx_team_status (team_id, status),
    INDEX idx_status_priority (status, priority DESC)
);

-- CD 部署任务表
CREATE TABLE cd_deployments (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    deployment_id VARCHAR(64) NOT NULL,
    service_name VARCHAR(100) NOT NULL,
    team_id VARCHAR(32) NOT NULL,
    version VARCHAR(100) NOT NULL,
    image_tag VARCHAR(200) NOT NULL,
    target_clusters JSONB,                  -- 目标集群列表
    strategy VARCHAR(20) DEFAULT 'canary',  -- blue_green / canary / rolling
    canary_weight INT DEFAULT 5,            -- 金丝雀流量百分比
    status VARCHAR(20) DEFAULT 'pending',   -- pending / approved / deploying / canary / promoting / completed / rolled_back
    approval_id VARCHAR(64),
    rollback_reason TEXT,
    started_at TIMESTAMP,
    finished_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE KEY uk_deployment (deployment_id),
    INDEX idx_team_status (team_id, status),
    INDEX idx_service (service_name, created_at DESC)
);

-- 金丝雀监控记录
CREATE TABLE canary_metrics (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    deployment_id VARCHAR(64) NOT NULL,
    service_name VARCHAR(100) NOT NULL,
    metric_time TIMESTAMP NOT NULL,
    canary_error_rate DECIMAL(6,4),
    stable_error_rate DECIMAL(6,4),
    canary_p99_ms INT,
    stable_p99_ms INT,
    canary_qps INT,
    stable_qps INT,
    canary_pod_count INT,
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_deployment_time (deployment_id, metric_time)
);

-- 审批表
CREATE TABLE approvals (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    approval_id VARCHAR(64) NOT NULL,
    deployment_id VARCHAR(64) NOT NULL,
    team_id VARCHAR(32) NOT NULL,
    approver_id VARCHAR(32),
    requested_by VARCHAR(32) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',  -- pending / approved / rejected / expired
    risk_level VARCHAR(10) DEFAULT 'normal', -- low / normal / high
    expires_at TIMESTAMP NOT NULL,
    decided_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE KEY uk_approval (approval_id),
    INDEX idx_approver_status (approver_id, status),
    INDEX idx_expires (status, expires_at)
);

-- CI 构建节点表
CREATE TABLE ci_nodes (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    node_id VARCHAR(32) NOT NULL,
    ip_address VARCHAR(45) NOT NULL,
    total_cpu INT NOT NULL,                   -- 总 CPU 核数
    total_memory_mb INT NOT NULL,             -- 总内存 MB
    status VARCHAR(20) DEFAULT 'healthy',     -- healthy / degraded / unreachable / draining
    last_heartbeat TIMESTAMP,
    running_jobs INT DEFAULT 0,               -- 当前运行任务数
    cpu_used DECIMAL(6,2) DEFAULT 0,          -- 实际 CPU 使用（来自 cgroup）
    memory_used_mb INT DEFAULT 0,             -- 实际内存使用
    build_success_count INT DEFAULT 0,        -- 成功构建累计
    build_failure_count INT DEFAULT 0,        -- 失败构建累计
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),

    UNIQUE KEY uk_node (node_id),
    INDEX idx_status (status)
);

-- 安全扫描结果表
CREATE TABLE security_scan_results (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    scan_id VARCHAR(64) NOT NULL,
    job_id VARCHAR(64) NOT NULL,
    scan_type VARCHAR(20) NOT NULL,           -- sast / dependency / secret / image
    scanner VARCHAR(32) NOT NULL,             -- semgrep / snyk / gitleaks / trivy
    passed BOOLEAN DEFAULT FALSE,
    critical_count INT DEFAULT 0,
    high_count INT DEFAULT 0,
    medium_count INT DEFAULT 0,
    low_count INT DEFAULT 0,
    scan_duration_sec INT,
    scan_report_url VARCHAR(500),             -- 扫描报告存储路径
    created_at TIMESTAMP DEFAULT NOW(),

    INDEX idx_job (job_id),
    INDEX idx_type_passed (scan_type, passed)
);

-- 部署锁表（防止同一服务并发部署冲突）
CREATE TABLE deployment_locks (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    service_name VARCHAR(100) NOT NULL,
    cluster_name VARCHAR(100) NOT NULL,
    locked_by VARCHAR(64) NOT NULL,           -- 锁持有者的 deployment_id
    team_id VARCHAR(32) NOT NULL,
    locked_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP NOT NULL,            -- 锁超时时间

    UNIQUE KEY uk_service_cluster (service_name, cluster_name),
    INDEX idx_expires (expires_at)
);

-- 构建缓存表
CREATE TABLE build_cache_entries (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    cache_key VARCHAR(200) NOT NULL,          -- 缓存键（language + hash）
    storage_path VARCHAR(500) NOT NULL,       -- S3/NFS 存储路径
    size_mb INT NOT NULL,
    hit_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP NOT NULL,

    UNIQUE KEY uk_cache_key (cache_key),
    INDEX idx_expires (expires_at)
);

-- 部署批次进度表
CREATE TABLE deployment_batches (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    deployment_id VARCHAR(64) NOT NULL,
    batch_id INT NOT NULL,
    cluster_names JSONB NOT NULL,             -- 该批次包含的集群列表
    status VARCHAR(20) DEFAULT 'pending',     -- pending / in_progress / completed / partial_failed / rolled_back
    success_count INT DEFAULT 0,
    failure_count INT DEFAULT 0,
    started_at TIMESTAMP,
    finished_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),

    INDEX idx_deployment (deployment_id),
    INDEX idx_status (status)
);
```

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

### CI Pipeline 执行器：分阶段执行 + 缓存 + 产物管理

流水线定义了 5 个阶段，但执行器需要处理：阶段间依赖、缓存命中、产物传递、失败重试。

```python
from dataclasses import dataclass
from enum import Enum
from typing import Optional
import json

class StageStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    SUCCESS = "success"
    FAILED = "failed"
    SKIPPED = "skipped"    # 前一阶段失败 → 后续阶段跳过
    CANCELLED = "cancelled"

@dataclass
class CacheEntry:
    """构建缓存条目"""
    key: str               # 缓存键（如 go-module-hash, maven-pom-hash）
    path: str              # 缓存文件路径
    size_mb: int
    created_at: float
    hits: int = 0          # 缓存命中次数

@dataclass
class Artifact:
    """构建产物"""
    name: str              # 产物名称（如 app-binary, docker-image）
    type: str              # binary / docker_image / report / coverage
    path: str              # 存储路径（S3 / Registry）
    size_mb: int
    sha256: str            # 产物哈希（完整性校验）
    stage_name: str        # 产生该产物的阶段
    expires_at: float      # 过期时间（默认 7 天）

class CIPipelineExecutor:
    """CI 流水线执行器 — 分阶段执行，含缓存和产物管理"""

    MAX_STAGE_RETRIES = 2        # 单阶段最大重试次数
    STAGE_TIMEOUT_SEC = 3600     # 单阶段超时 60 分钟
    CACHE_TTL_DAYS = 7           # 缓存有效期 7 天
    ARTIFACT_TTL_DAYS = 30       # 产物有效期 30 天

    def __init__(self, scheduler: EnhancedCIScheduler,
                 cache_store: dict[str, CacheEntry],
                 artifact_store: dict[str, Artifact]):
        self.scheduler = scheduler
        self.cache_store = cache_store      # 缓存仓库（分布式缓存如 Redis+S3）
        self.artifact_store = artifact_store  # 产物仓库（S3 + Registry）

    def execute_pipeline(self, pipeline_config: dict, job: dict) -> dict:
        """执行完整的 CI 流水线"""
        stages = pipeline_config["stages"]
        results = {}
        stage_artifacts = {}    # 阶段间产物传递
        pipeline_start = time.time()

        for i, stage in enumerate(stages):
            stage_name = stage["name"]
            stage_start = time.time()

            # ---- 前置检查：前一阶段是否成功 ----
            if i > 0:
                prev_stage = stages[i - 1]["name"]
                prev_result = results.get(prev_stage)
                if prev_result and prev_result["status"] != StageStatus.SUCCESS:
                    results[stage_name] = {
                        "status": StageStatus.SKIPPED,
                        "reason": f"前置阶段 {prev_stage} 失败",
                        "duration_sec": 0
                    }
                    continue

            # ---- 缓存命中检查 ----
            cache_key = self._compute_cache_key(stage, job)
            cache_entry = self.cache_store.get(cache_key)
            cache_hit = False
            if cache_entry and (time.time() - cache_entry.created_at < self.CACHE_TTL_DAYS * 86400):
                cache_hit = True
                cache_entry.hits += 1

            # ---- 执行阶段（含重试） ----
            retry_count = 0
            stage_result = None
            while retry_count <= self.MAX_STAGE_RETRIES:
                try:
                    stage_result = self._run_stage(
                        stage, job, cache_hit, stage_artifacts
                    )
                    if stage_result["status"] == StageStatus.SUCCESS:
                        break
                    retry_count += 1
                    if retry_count <= self.MAX_STAGE_RETRIES:
                        # 重试前清理：清除可能不完整的缓存
                        self._invalidate_cache(cache_key)
                        self._alert(f"阶段 {stage_name} 失败，第 {retry_count} 次重试")
                except TimeoutError:
                    stage_result = {
                        "status": StageStatus.FAILED,
                        "reason": "stage_timeout",
                        "duration_sec": self.STAGE_TIMEOUT_SEC
                    }
                    break

            duration = time.time() - stage_start
            stage_result["duration_sec"] = duration
            stage_result["cache_hit"] = cache_hit
            stage_result["retry_count"] = retry_count
            results[stage_name] = stage_result

            # ---- 收集产物 ----
            if stage_result["status"] == StageStatus.SUCCESS:
                artifacts = self._collect_artifacts(stage, job)
                stage_artifacts[stage_name] = artifacts
                for art in artifacts:
                    self.artifact_store[art.sha256] = art
                # 更新缓存（只在成功时）
                if not cache_hit:
                    new_cache = CacheEntry(
                        key=cache_key,
                        path=self._cache_path(stage, job),
                        size_mb=self._compute_cache_size(stage, job),
                        created_at=time.time()
                    )
                    self.cache_store[cache_key] = new_cache

        # ---- 汇总结果 ----
        total_duration = time.time() - pipeline_start
        all_success = all(
            r["status"] == StageStatus.SUCCESS or r["status"] == StageStatus.SKIPPED
            for r in results.values()
        )
        # SKIPPED 的阶段不算失败，但必须有至少 build 阶段成功
        build_ok = results.get("build", {}).get("status") == StageStatus.SUCCESS

        return {
            "job_id": job["job_id"],
            "status": "success" if (all_success and build_ok) else "failed",
            "stages": results,
            "total_duration_sec": total_duration,
            "artifacts": stage_artifacts
        }

    def _run_stage(self, stage: dict, job: dict,
                   cache_hit: bool, stage_artifacts: dict) -> dict:
        """执行单个阶段（在 CI 节点容器中运行）"""
        # 分配节点资源
        cpu = stage.get("cpu_needed", 2)
        mem = stage.get("memory_needed_mb", 4096)
        node = self.scheduler._find_best_node({"cpu_needed": cpu, "memory_needed_mb": mem})
        if not node:
            return {"status": StageStatus.FAILED, "reason": "no_available_node"}

        # 构建容器启动命令
        commands = stage.get("steps", [])
        env_vars = {
            "CI_COMMIT_SHA": job["commit_sha"],
            "CI_BRANCH": job["branch"],
            "CI_CACHE_HIT": str(cache_hit),
        }
        # 注入前一阶段的产物路径
        if stage_artifacts:
            env_vars["CI_PREV_ARTIFACTS"] = json.dumps(stage_artifacts)

        # 实际执行（通过容器 runtime）
        exec_result = self._execute_in_container(
            node, stage["image"], commands, env_vars,
            timeout=self.STAGE_TIMEOUT_SEC
        )

        if exec_result["exit_code"] == 0:
            return {"status": StageStatus.SUCCESS, "output": exec_result["output"]}
        else:
            return {
                "status": StageStatus.FAILED,
                "reason": f"exit_code={exec_result['exit_code']}",
                "output": exec_result["output"],
                "error": exec_result.get("stderr", "")
            }

    def _compute_cache_key(self, stage: dict, job: dict) -> str:
        """计算缓存键：基于阶段类型 + 依赖哈希"""
        # 不同语言用不同的依赖文件作为缓存键
        cache_key_files = {
            "build": {
                "golang": "go.sum",
                "java": "pom.xml",
                "node": "package-lock.json",
                "python": "requirements.txt",
                "rust": "Cargo.lock"
            }
        }
        # 实际计算：stage_name + language + dependency_hash + commit_sha
        lang = stage.get("language", "unknown")
        key_file = cache_key_files.get(stage["name"], {}).get(lang, "")
        return f"{stage['name']}:{lang}:{job['commit_sha']}:{key_file}"

    def _collect_artifacts(self, stage: dict, job: dict) -> list[Artifact]:
        """收集阶段产物"""
        artifact_configs = stage.get("artifacts", [])
        collected = []
        for ac in artifact_configs:
            art = Artifact(
                name=ac["name"],
                type=ac["type"],
                path=self._upload_artifact(ac, job),
                size_mb=ac.get("size_mb", 0),
                sha256=self._compute_sha256(ac["path"]),
                stage_name=stage["name"],
                expires_at=time.time() + self.ARTIFACT_TTL_DAYS * 86400
            )
            collected.append(art)
        return collected

    def _invalidate_cache(self, cache_key: str):
        """清除缓存（重试前使用）"""
        if cache_key in self.cache_store:
            del self.cache_store[cache_key]

    def _execute_in_container(self, node, image, commands, env_vars, timeout):
        """在容器中执行命令（抽象接口，实际对接 Docker/Kubernetes runtime）"""
        # 实际实现调用 node agent 启动容器并监控执行
        pass

    def _upload_artifact(self, artifact_config, job):
        """上传产物到存储（S3/Registry）"""
        pass

    def _compute_sha256(self, path):
        """计算文件哈希"""
        pass

    def _compute_cache_size(self, stage, job):
        """计算缓存大小"""
        return 0

    def _cache_path(self, stage, job):
        """缓存存储路径"""
        return ""

    def _alert(self, msg):
        """发送告警"""
        print(f"[ALERT] {msg}")
```

**Pipeline 执行器关键指标：**

| 指标 | 无缓存/无重试 | 有缓存+重试 | 有缓存+重试+产物管理 |
|------|-------------|-----------|-------------------|
| 单次构建成功率 | 92% | 98% | 99.5% |
| 平均构建时间（Go） | 8 分钟 | 2 分钟（缓存命中） | 1.5 分钟（缓存+产物传递） |
| 构建失败恢复时间 | 手动重提 | 3 分钟（自动重试） | 1 分钟（缓存+重试） |
| 产物丢失率 | 5% | 2% | < 0.1%（S3+校验） |

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

### 构建调度器增强：资源追踪 + 节点健康检查 + 队列超时

上述基础调度器解决了优先级和资源限额问题，但在生产环境中还需要处理：节点故障、队列超时、实时资源追踪。

```python
import threading
import time
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional

class NodeStatus(Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"       # 部分资源异常
    UNREACHABLE = "unreachable"  # 网络不通
    DRAINING = "draining"       # 正在排空任务，准备下线

@dataclass
class CINode:
    """CI 构建节点 — 实时资源追踪"""
    id: str
    total_cpu: int
    total_memory_mb: int
    status: NodeStatus = NodeStatus.HEALTHY
    last_heartbeat: float = 0.0
    running_job_ids: list = field(default_factory=list)
    # 实际资源使用（来自 cgroup 采集，非简单求和）
    actual_cpu_used: float = 0.0
    actual_memory_used_mb: int = 0
    # 历史统计
    build_success_count: int = 0
    build_failure_count: int = 0
    avg_build_duration_sec: float = 0.0

@dataclass
class QueuedJob:
    """队列中的构建任务 — 含入队时间用于超时检测"""
    job: dict
    priority: int
    enqueued_at: float       # time.time()
    requeue_count: int = 0   # 被重新入队次数

class EnhancedCIScheduler:
    """增强版构建调度器"""

    HEARTBEAT_TIMEOUT_SEC = 60       # 心跳超时阈值
    QUEUE_TIMEOUT_SEC = 1800         # 队列超时 30 分钟
    NODE_CHECK_INTERVAL_SEC = 30     # 节点健康检查间隔
    MAX_REQUEUE_COUNT = 3            # 最大重新入队次数

    def __init__(self, nodes: list[CINode]):
        self.nodes = {n.id: n for n in nodes}
        self.job_queue = []            # list[QueuedJob]，手动维护优先级堆
        self.running_jobs = {}         # job_id -> {job, node_id, started_at}
        self.lock = threading.Lock()
        self._start_background_checks()

    # ========== 资源实时追踪 ==========

    def get_node_available(self, node_id: str) -> dict:
        """获取节点实际可用资源（基于 cgroup 实时采集，而非简单求和）"""
        node = self.nodes[node_id]
        if node.status != NodeStatus.HEALTHY:
            return {"cpu": 0, "memory_mb": 0}

        # 使用 actual 而非 sum(running_jobs) — 因为 cgroup 计量更精确
        # 部分进程可能已退出但 job 状态尚未更新
        avail_cpu = node.total_cpu * 1.5 - node.actual_cpu_used
        avail_mem = node.total_memory_mb * 1.5 - node.actual_memory_used_mb
        return {"cpu": max(0, avail_cpu), "memory_mb": max(0, avail_mem)}

    def update_node_metrics(self, node_id: str, cpu_used: float, mem_used_mb: int):
        """由节点 Agent 定期上报实际资源使用"""
        with self.lock:
            node = self.nodes.get(node_id)
            if node:
                node.actual_cpu_used = cpu_used
                node.actual_memory_used_mb = mem_used_mb
                node.last_heartbeat = time.time()

    # ========== 节点健康检查 ==========

    def _start_background_checks(self):
        """启动后台健康检查和超时扫描线程"""
        t1 = threading.Thread(target=self._node_health_loop, daemon=True)
        t2 = threading.Thread(target=self._queue_timeout_loop, daemon=True)
        t1.start()
        t2.start()

    def _node_health_loop(self):
        """定期检查节点心跳，标记不健康节点"""
        while True:
            time.sleep(self.NODE_CHECK_INTERVAL_SEC)
            now = time.time()
            with self.lock:
                for node in self.nodes.values():
                    if node.status == NodeStatus.DRAINING:
                        continue
                    # 心跳超过 60 秒 → 标记为不可达
                    if now - node.last_heartbeat > self.HEARTBEAT_TIMEOUT_SEC:
                        if node.status != NodeStatus.UNREACHABLE:
                            self._handle_node_failure(node)

    def _handle_node_failure(self, node: CINode):
        """处理节点故障：将运行中任务迁移，标记节点不可达"""
        node.status = NodeStatus.UNREACHABLE
        failed_job_ids = list(node.running_job_ids)
        node.running_job_ids.clear()

        # 将该节点上的运行中任务重新入队
        for job_id in failed_job_ids:
            if job_id in self.running_jobs:
                job_info = self.running_jobs.pop(job_id)
                job = job_info["job"]
                job["status"] = "queued"
                # 重新入队，优先级提升（因为是被中断的任务）
                qj = QueuedJob(
                    job=job,
                    priority=200,  # 高优先级，尽快重新调度
                    enqueued_at=time.time(),
                    requeue_count=0
                )
                self.job_queue.append(qj)

        self._sort_queue()
        self._alert(f"CI 节点 {node.id} 心跳丢失，{len(failed_job_ids)} 个任务已重新入队")

    # ========== 队列超时处理 ==========

    def _queue_timeout_loop(self):
        """定期扫描队列中超时的任务"""
        while True:
            time.sleep(60)
            now = time.time()
            with self.lock:
                expired = []
                remaining = []
                for qj in self.job_queue:
                    wait_time = now - qj.enqueued_at
                    if wait_time > self.QUEUE_TIMEOUT_SEC:
                        expired.append(qj)
                    else:
                        remaining.append(qj)
                self.job_queue = remaining

                # 通知超时任务
                for qj in expired:
                    qj.job["status"] = "timeout"
                    self._notify_team(
                        qj.job["team_id"],
                        f"构建任务 {qj.job['job_id']} 排队超时（>30分钟），已自动取消"
                    )

    # ========== 增强调度逻辑 ==========

    def schedule(self):
        """调度循环：考虑节点健康状态和实际资源"""
        with self.lock:
            self._sort_queue()
            scheduled = []
            remaining = []

            for qj in self.job_queue:
                node = self._find_best_node(qj.job)
                if node:
                    self._assign_job(qj.job, node)
                    scheduled.append(qj)
                else:
                    # 无可用节点，检查是否超过重入队上限
                    qj.requeue_count += 1
                    if qj.requeue_count > self.MAX_REQUEUE_COUNT:
                        qj.job["status"] = "failed"
                        self._notify_team(
                            qj.job["team_id"],
                            f"构建任务 {qj.job['job_id']} 调度失败（重试 {qj.requeue_count} 次），请检查资源需求"
                        )
                    else:
                        remaining.append(qj)

            self.job_queue = remaining

    def _find_best_node(self, job: dict) -> Optional[CINode]:
        """寻找最优节点：资源最匹配 + 健康状态"""
        cpu_need = job.get("cpu_needed", 2)
        mem_need = job.get("memory_needed_mb", 4096)
        best_node = None
        best_score = -1

        for node in self.nodes.values():
            if node.status != NodeStatus.HEALTHY:
                continue
            avail = self.get_node_available(node.id)
            if avail["cpu"] < cpu_need or avail["memory_mb"] < mem_need:
                continue
            # 评分：资源碎片越小越好（紧凑调度）
            score = 1.0 / (1.0 + avail["cpu"] - cpu_need)
            # 历史成功率加权
            total = node.build_success_count + node.build_failure_count
            if total > 0:
                score *= (node.build_success_count / total)
            if score > best_score:
                best_score = score
                best_node = node

        return best_node

    def _sort_queue(self):
        """按优先级排序（优先级相同则按入队时间 FIFO）"""
        self.job_queue.sort(key=lambda qj: (-qj.priority, qj.enqueued_at))

    def _assign_job(self, job: dict, node: CINode):
        """将任务分配到节点"""
        job["status"] = "running"
        job["node_id"] = node.id
        job["started_at"] = time.time()
        node.running_job_ids.append(job["job_id"])
        self.running_jobs[job["job_id"]] = {
            "job": job, "node_id": node.id, "started_at": time.time()
        }

    def _alert(self, msg: str):
        """发送告警"""
        print(f"[ALERT] {msg}")

    def _notify_team(self, team_id: str, msg: str):
        """通知团队"""
        print(f"[NOTIFY] team={team_id} msg={msg}")
```

**增强调度器效果：**

| 指标 | 基础调度器 | 增强调度器 |
|------|----------|----------|
| 节点故障恢复时间 | 手动发现（>30 分钟） | 30 秒自动检测+迁移 |
| 队列超时任务处理 | 无（永远排队） | 30 分钟自动取消+通知 |
| 资源碎片率 | 30%（简单轮询） | 12%（紧凑调度） |
| 调度成功率 | 95% | 99.2% |

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

### 安全扫描：完整安全扫描流水线（Trivy + SAST + 依赖扫描 + 密钥检测）

仅做镜像漏洞扫描不足以覆盖所有安全风险。完整的安全扫描流水线应包含 4 层：

```
代码提交 → SAST（静态代码分析） → 依赖扫描 → 密钥检测 → 镜像漏洞扫描 → 部署
              ↓                      ↓            ↓              ↓
         代码级漏洞            供应链漏洞     凭证泄露        运行时漏洞
       (SQL注入/XSS/...)    (log4shell)   (AK/密码)     (CVE/配置错误)
```

```python
from dataclasses import dataclass
from enum import Enum
from typing import Optional
import json

class ScanSeverity(Enum):
    CRITICAL = "critical"
    HIGH = "high"
    MEDIUM = "medium"
    LOW = "low"
    INFO = "info"

@dataclass
class Vulnerability:
    id: str                 # CVE-ID 或规则 ID
    severity: ScanSeverity
    title: str
    description: str
    file_path: str
    line_number: int = 0
    package: str = ""       # 受影响的包名
    fixed_version: str = "" # 修复版本
    category: str = ""      # sast / dependency / secret / image

@dataclass
class ScanResult:
    scanner: str
    passed: bool
    vulnerabilities: list[Vulnerability]
    duration_sec: float
    error: str = ""

class SecurityScanPipeline:
    """完整安全扫描流水线 — 4 层扫描"""

    # 扫描策略配置
    POLICIES = {
        "sast": {
            "block_on": [ScanSeverity.CRITICAL, ScanSeverity.HIGH],
            "max_critical": 0,
            "max_high": 0,
        },
        "dependency": {
            "block_on": [ScanSeverity.CRITICAL, ScanSeverity.HIGH],
            "max_critical": 0,
            "max_high": 10,     # 允许最多 10 个高危依赖漏洞（逐步修复）
        },
        "secret": {
            "block_on": [ScanSeverity.CRITICAL, ScanSeverity.HIGH],
            "max_critical": 0,  # 任何密钥泄露 → 阻止
            "max_high": 0,
        },
        "image": {
            "block_on": [ScanSeverity.CRITICAL],
            "max_critical": 0,
            "max_high": 5,
        }
    }

    # 扫描超时
    SCAN_TIMEOUT_SEC = 300   # 每种扫描最多 5 分钟

    def __init__(self, sast_client, dependency_client,
                 secret_client, image_client):
        self.sast = sast_client          # Semgrep / SonarQube
        self.dependency = dependency_client  # Snyk / OWASP Dependency-Check
        self.secret = secret_client      # TruffleHog / Gitleaks
        self.image = image_client        # Trivy / Grype

    def full_scan(self, repo_path: str, image_tag: str,
                  branch: str = "") -> dict:
        """执行完整 4 层安全扫描"""
        results = {}
        all_vulnerabilities = []
        scan_start = time.time()

        # ---- 第 1 层：SAST 静态代码分析 ----
        sast_result = self._scan_sast(repo_path, branch)
        results["sast"] = sast_result
        all_vulnerabilities.extend(sast_result.vulnerabilities)

        # ---- 第 2 层：依赖漏洞扫描 ----
        dep_result = self._scan_dependencies(repo_path)
        results["dependency"] = dep_result
        all_vulnerabilities.extend(dep_result.vulnerabilities)

        # ---- 第 3 层：密钥/凭据泄露检测 ----
        secret_result = self._scan_secrets(repo_path)
        results["secret"] = secret_result
        all_vulnerabilities.extend(secret_result.vulnerabilities)

        # ---- 第 4 层：镜像漏洞扫描 ----
        if image_tag:
            image_result = self._scan_image(image_tag)
            results["image"] = image_result
            all_vulnerabilities.extend(image_result.vulnerabilities)

        # ---- 综合判定 ----
        total_duration = time.time() - scan_start
        blocked, block_reasons = self._evaluate_policies(results)

        # 按严重级别汇总
        summary = self._summarize(all_vulnerabilities)

        return {
            "passed": not blocked,
            "block_reasons": block_reasons,
            "summary": summary,
            "total_vulnerabilities": len(all_vulnerabilities),
            "total_duration_sec": total_duration,
            "scans": {
                name: {
                    "passed": r.passed,
                    "count": len(r.vulnerabilities),
                    "duration_sec": r.duration_sec
                }
                for name, r in results.items()
            }
        }

    def _scan_sast(self, repo_path: str, branch: str) -> ScanResult:
        """第 1 层：SAST — 静态代码分析（Semgrep / SonarQube）"""
        start = time.time()
        try:
            # Semgrep 扫描：支持多语言，规则集覆盖 OWASP Top 10
            raw = self.sast.scan(
                repo_path,
                config="auto",  # 自动检测语言并选择规则集
                # 额外规则集
                rules=[
                    "p/owasp-top-ten",     # OWASP Top 10
                    "p/security-audit",    # 安全审计
                    "p/r2c-security",      # r2c 安全规则
                ],
                timeout=self.SCAN_TIMEOUT_SEC
            )

            vulnerabilities = []
            for finding in raw.findings:
                sev = self._map_severity(finding.severity)
                vulnerabilities.append(Vulnerability(
                    id=finding.rule_id,
                    severity=sev,
                    title=finding.title,
                    description=finding.message,
                    file_path=finding.file_path,
                    line_number=finding.line_number,
                    category="sast"
                ))

            return ScanResult(
                scanner="semgrep",
                passed=True,
                vulnerabilities=vulnerabilities,
                duration_sec=time.time() - start
            )
        except Exception as e:
            return ScanResult(
                scanner="semgrep", passed=False,
                vulnerabilities=[], duration_sec=time.time() - start,
                error=str(e)
            )

    def _scan_dependencies(self, repo_path: str) -> ScanResult:
        """第 2 层：依赖漏洞扫描（Snyk / OWASP Dependency-Check）"""
        start = time.time()
        try:
            # 扫描所有锁文件：go.sum, package-lock.json, pom.xml, requirements.txt, Cargo.lock
            raw = self.dependency.scan(
                repo_path,
                # 检测范围
                scan_unmanaged=True,   # 扫描无锁文件的项目
                ignore_dev=False,      # 开发依赖也扫描
            )

            vulnerabilities = []
            for pkg_vuln in raw.vulnerabilities:
                sev = self._map_severity(pkg_vuln.severity)
                vulnerabilities.append(Vulnerability(
                    id=pkg_vuln.cve_id,
                    severity=sev,
                    title=pkg_vuln.title,
                    description=pkg_vuln.description,
                    file_path=pkg_vuln.manifest_path,
                    package=pkg_vuln.package_name,
                    fixed_version=pkg_vuln.fixed_version,
                    category="dependency"
                ))

            return ScanResult(
                scanner="snyk",
                passed=True,
                vulnerabilities=vulnerabilities,
                duration_sec=time.time() - start
            )
        except Exception as e:
            return ScanResult(
                scanner="snyk", passed=False,
                vulnerabilities=[], duration_sec=time.time() - start,
                error=str(e)
            )

    def _scan_secrets(self, repo_path: str) -> ScanResult:
        """第 3 层：密钥/凭据泄露检测（TruffleHog + Gitleaks）"""
        start = time.time()
        try:
            # Gitleaks：扫描 Git 历史中的密钥
            raw = self.secret.scan(
                repo_path,
                # 检测类型
                detect_types=[
                    "aws_access_key",
                    "aws_secret_key",
                    "github_token",
                    "gitlab_token",
                    "private_key",
                    "database_url",
                    "jwt_secret",
                    "api_key",
                    "password",
                ],
                # 扫描完整 Git 历史（不只是当前 commit）
                scan_history=True,
            )

            vulnerabilities = []
            for leak in raw.leaks:
                # 密钥泄露一律为 CRITICAL
                vulnerabilities.append(Vulnerability(
                    id=leak.rule_id,
                    severity=ScanSeverity.CRITICAL,
                    title=f"检测到 {leak.detect_type} 泄露",
                    description=f"在 {leak.commit_hash[:8]} 中发现 {leak.detect_type}，"
                                f"匹配规则: {leak.rule_id}",
                    file_path=leak.file_path,
                    line_number=leak.line_number,
                    category="secret"
                ))

            return ScanResult(
                scanner="gitleaks",
                passed=True,
                vulnerabilities=vulnerabilities,
                duration_sec=time.time() - start
            )
        except Exception as e:
            return ScanResult(
                scanner="gitleaks", passed=False,
                vulnerabilities=[], duration_sec=time.time() - start,
                error=str(e)
            )

    def _scan_image(self, image_tag: str) -> ScanResult:
        """第 4 层：镜像漏洞扫描（Trivy）"""
        start = time.time()
        try:
            result = self.image.scan(image_tag)

            vulnerabilities = []
            for v in result.vulnerabilities:
                sev = self._map_severity(v.severity)
                vulnerabilities.append(Vulnerability(
                    id=v.vulnerability_id,
                    severity=sev,
                    title=v.title,
                    description=v.description,
                    package=v.package_name,
                    fixed_version=v.fixed_version,
                    category="image"
                ))

            return ScanResult(
                scanner="trivy",
                passed=True,
                vulnerabilities=vulnerabilities,
                duration_sec=time.time() - start
            )
        except Exception as e:
            return ScanResult(
                scanner="trivy", passed=False,
                vulnerabilities=[], duration_sec=time.time() - start,
                error=str(e)
            )

    def _evaluate_policies(self, results: dict) -> tuple[bool, list[str]]:
        """根据策略判定是否阻止部署"""
        blocked = False
        reasons = []

        for scan_type, policy in self.POLICIES.items():
            result = results.get(scan_type)
            if not result or not result.vulnerabilities:
                continue

            # 按严重级别计数
            counts = {sev: 0 for sev in ScanSeverity}
            for v in result.vulnerabilities:
                counts[v.severity] += 1

            # 检查是否超过阈值
            if counts[ScanSeverity.CRITICAL] > policy["max_critical"]:
                blocked = True
                reasons.append(
                    f"[{scan_type}] CRITICAL 漏洞 {counts[ScanSeverity.CRITICAL]} 个 "
                    f"> 阈值 {policy['max_critical']}"
                )
            if counts[ScanSeverity.HIGH] > policy["max_high"]:
                blocked = True
                reasons.append(
                    f"[{scan_type}] HIGH 漏洞 {counts[ScanSeverity.HIGH]} 个 "
                    f"> 阈值 {policy['max_high']}"
                )

        return blocked, reasons

    def _summarize(self, vulns: list[Vulnerability]) -> dict:
        """按类别和严重级别汇总"""
        summary = {}
        for v in vulns:
            cat = v.category
            if cat not in summary:
                summary[cat] = {"critical": 0, "high": 0, "medium": 0, "low": 0}
            summary[cat][v.severity.value] += 1
        return summary

    def _map_severity(self, raw_severity: str) -> ScanSeverity:
        """映射严重级别"""
        mapping = {
            "CRITICAL": ScanSeverity.CRITICAL,
            "HIGH": ScanSeverity.HIGH,
            "MEDIUM": ScanSeverity.MEDIUM,
            "LOW": ScanSeverity.LOW,
            "WARNING": ScanSeverity.HIGH,
            "ERROR": ScanSeverity.CRITICAL,
        }
        return mapping.get(raw_severity.upper(), ScanSeverity.INFO)
```

**安全扫描流水线效果：**

| 扫描层 | 工具 | 平均耗时 | 检出率 | 误报率 |
|-------|------|---------|-------|-------|
| SAST | Semgrep | 1-2 min | 85% (代码级漏洞) | 15% |
| 依赖扫描 | Snyk | 30-60s | 95% (已知 CVE) | 5% |
| 密钥检测 | Gitleaks | 30-60s | 98% (硬编码密钥) | 3% |
| 镜像扫描 | Trivy | 2-3 min | 90% (镜像 CVE) | 10% |
| **总计** | **4 层** | **4-7 min** | **综合 92%** | **8%** |

**安全扫描与部署的集成策略：**

| 场景 | SAST | 依赖 | 密钥 | 镜像 | 部署动作 |
|------|------|------|------|------|---------|
| 开发环境 | 可选 | 可选 | 强制 | 可选 | 密钥泄露 → 阻止 |
| 测试环境 | 强制 | 强制 | 强制 | 可选 | CRITICAL → 阻止 |
| 生产环境 | 强制 | 强制 | 强制 | 强制 | CRITICAL/HIGH → 阻止 |
| 热修复 | 可选 | 强制 | 强制 | 强制 | 仅密钥+镜像 CRITICAL → 阻止 |

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

        # 4. 记录回滚日志（审计）
        self.db.insert("rollback_log", {
            "service": service,
            "reason": reason,
            "operator": "auto-canary",
            "created_at": now()
        })

    def rollback_failed(self, service, original_reason):
        """回滚本身也失败 → 紧急处理"""
        # Istio 流量切换失败 → 降级方案
        # 1. 直接删除金丝雀 Service（切断所有到新版本的流量）
        self.k8s.delete_service(f"{service}-canary")

        # 2. 如果 K8s API 也不可用 → 通知运维手动介入
        self.page_oncall(f"紧急！{service} 自动回滚失败，原始原因：{original_reason}")

        # 3. 全局流量兜底：Nginx/Ingress 层直接屏蔽金丝雀 Pod IP
        self.ingress.block_canary_pods(service)
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

### 多集群部署编排：50+ K8s 集群滚动部署 + 进度追踪

50+ 集群的部署不能同时推送（网络带宽、Registry 压力），需要分批滚动。核心问题：如何控制批次、追踪进度、处理部分集群失败。

```python
from dataclasses import dataclass, field
from enum import Enum
import time

class ClusterStatus(Enum):
    PENDING = "pending"
    SYNCING = "syncing"           # ArgoCD 正在同步
    DEPLOYING = "deploying"       # Pod 正在滚动更新
    VERIFYING = "verifying"       # 健康检查中
    COMPLETED = "completed"
    FAILED = "failed"
    ROLLED_BACK = "rolled_back"

class DeploymentBatchStatus(Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    PARTIAL_FAILED = "partial_failed"
    ROLLED_BACK = "rolled_back"

@dataclass
class ClusterInfo:
    """K8s 集群信息"""
    name: str
    region: str                 # cn-east / cn-south / us-west
    environment: str            # production / staging
    node_count: int
    priority: int               # 部署优先级（先部署低优先级集群）
    argocd_endpoint: str
    registry_endpoint: str      # 该区域就近的镜像仓库端点

@dataclass
class ClusterDeployment:
    """单个集群的部署状态"""
    cluster: ClusterInfo
    status: ClusterStatus = ClusterStatus.PENDING
    started_at: float = 0
    finished_at: float = 0
    old_version: str = ""
    new_version: str = ""
    error_message: str = ""
    pod_progress: float = 0.0   # 0.0 ~ 1.0

@dataclass
class DeploymentBatch:
    """部署批次"""
    batch_id: int
    clusters: list[ClusterDeployment]
    status: DeploymentBatchStatus = DeploymentBatchStatus.PENDING
    started_at: float = 0
    finished_at: float = 0

class MultiClusterDeployer:
    """多集群滚动部署编排器"""

    # 部署批次策略：
    # 第 1 批：3 个集群（灰度验证）
    # 第 2 批：7 个集群（区域扩展）
    # 第 3 批：15 个集群（大规模推广）
    # 第 4 批：剩余集群（全量完成）
    BATCH_SIZES = [3, 7, 15]  # 剩余归入最后一批

    BATCH_INTERVAL_SEC = 120    # 批次间等待 2 分钟
    HEALTH_CHECK_TIMEOUT = 300  # 单集群健康检查超时 5 分钟
    MAX_CLUSTER_FAILURES = 3    # 连续 3 个集群失败 → 暂停部署
    ROLLBACK_ON_BATCH_FAIL = True

    def __init__(self, clusters: list[ClusterInfo], argocd_client, prometheus_client):
        self.clusters = clusters
        self.argocd = argocd_client
        self.prometheus = prometheus_client
        self.deployment_state: dict[str, ClusterDeployment] = {}

    def deploy(self, service: str, version: str,
               strategy: str = "rolling") -> dict:
        """滚动部署到所有目标集群"""
        # 1. 按区域和优先级分批
        batches = self._create_batches()
        total_clusters = sum(len(b.clusters) for b in batches)
        deployment_start = time.time()

        # 2. 初始化部署状态
        for batch in batches:
            for cd in batch.clusters:
                cd.old_version = self._get_current_version(cd.cluster, service)
                cd.new_version = version
                self.deployment_state[cd.cluster.name] = cd

        # 3. 逐批部署
        overall_status = "success"
        consecutive_failures = 0

        for batch_idx, batch in enumerate(batches):
            self._log(f"=== 开始第 {batch.batch_id} 批部署 "
                      f"({len(batch.clusters)} 个集群) ===")
            batch.status = DeploymentBatchStatus.IN_PROGRESS
            batch.started_at = time.time()

            # 并行部署同一批次内的集群
            batch_results = self._deploy_batch_parallel(batch, service, version)

            # 检查批次结果
            failed_count = sum(1 for r in batch_results if r["status"] == "failed")
            success_count = len(batch_results) - failed_count

            if failed_count == 0:
                batch.status = DeploymentBatchStatus.COMPLETED
                consecutive_failures = 0
            elif failed_count < len(batch.clusters):
                batch.status = DeploymentBatchStatus.PARTIAL_FAILED
                consecutive_failures += failed_count
            else:
                batch.status = DeploymentBatchStatus.PARTIAL_FAILED
                consecutive_failures += failed_count

            batch.finished_at = time.time()

            # 连续失败过多 → 暂停整个部署
            if consecutive_failures >= self.MAX_CLUSTER_FAILURES:
                self._log(f"连续 {consecutive_failures} 个集群失败，暂停部署")
                if self.ROLLBACK_ON_BATCH_FAIL:
                    self._rollback_deployed_batches(batches[:batch_idx + 1], service)
                    overall_status = "rolled_back"
                else:
                    overall_status = "partial_failed"
                break

            # 批次间等待 + 全局健康观察
            if batch_idx < len(batches) - 1:
                self._log(f"批次间等待 {self.BATCH_INTERVAL_SEC}s，观察全局指标...")
                global_healthy = self._check_global_health(service)
                if not global_healthy:
                    self._log("全局指标异常，终止后续批次部署")
                    self._rollback_deployed_batches(batches[:batch_idx + 1], service)
                    overall_status = "rolled_back"
                    break
                time.sleep(self.BATCH_INTERVAL_SEC)

        total_duration = time.time() - deployment_start
        return {
            "service": service,
            "version": version,
            "status": overall_status,
            "total_clusters": total_clusters,
            "deployed_clusters": sum(1 for cd in self.deployment_state.values()
                                     if cd.status == ClusterStatus.COMPLETED),
            "failed_clusters": sum(1 for cd in self.deployment_state.values()
                                   if cd.status == ClusterStatus.FAILED),
            "total_duration_sec": total_duration,
            "batches": [
                {
                    "batch_id": b.batch_id,
                    "status": b.status.value,
                    "clusters": len(b.clusters),
                    "duration_sec": b.finished_at - b.started_at if b.finished_at else 0
                }
                for b in batches
            ]
        }

    def _create_batches(self) -> list[DeploymentBatch]:
        """按区域优先级分批：先非核心区域，后核心区域"""
        # 按优先级排序（低优先级先部署）
        sorted_clusters = sorted(self.clusters, key=lambda c: c.priority)

        batches = []
        remaining = [ClusterDeployment(cluster=c) for c in sorted_clusters]
        batch_id = 1

        for size in self.BATCH_SIZES:
            if not remaining:
                break
            batch_clusters = remaining[:size]
            remaining = remaining[size:]
            batches.append(DeploymentBatch(batch_id=batch_id, clusters=batch_clusters))
            batch_id += 1

        # 剩余集群归入最后一批
        if remaining:
            batches.append(DeploymentBatch(batch_id=batch_id, clusters=remaining))

        return batches

    def _deploy_batch_parallel(self, batch: DeploymentBatch,
                               service: str, version: str) -> list[dict]:
        """并行部署同一批次的集群"""
        results = []
        # 使用线程池并行部署（实际用 asyncio）
        from concurrent.futures import ThreadPoolExecutor, as_completed

        with ThreadPoolExecutor(max_workers=len(batch.clusters)) as executor:
            futures = {
                executor.submit(
                    self._deploy_to_cluster, cd, service, version
                ): cd for cd in batch.clusters
            }
            for future in as_completed(futures):
                cd = futures[future]
                try:
                    result = future.result()
                    results.append(result)
                except Exception as e:
                    cd.status = ClusterStatus.FAILED
                    cd.error_message = str(e)
                    results.append({"cluster": cd.cluster.name, "status": "failed",
                                    "error": str(e)})

        return results

    def _deploy_to_cluster(self, cd: ClusterDeployment,
                           service: str, version: str) -> dict:
        """部署到单个集群（完整流程）"""
        cluster = cd.cluster
        cd.status = ClusterStatus.SYNCING
        cd.started_at = time.time()

        try:
            # 1. 更新 Git manifest
            manifest_path = (f"manifests/{cluster.region}/{cluster.name}/"
                             f"{service}.yaml")
            self._update_manifest(manifest_path, {
                "image": f"{cluster.registry_endpoint}/{service}:{version}"
            })
            self._git_commit_push(f"deploy {service} v{version} to {cluster.name}")

            # 2. 触发 ArgoCD 同步（不等待轮询）
            cd.status = ClusterStatus.DEPLOYING
            self.argocd.sync(cluster.name, service)

            # 3. 等待 Pod 滚动更新完成
            self._wait_for_rollout(cd, service)

            # 4. 健康检查
            cd.status = ClusterStatus.VERIFYING
            healthy = self._check_cluster_health(cd, service)

            if healthy:
                cd.status = ClusterStatus.COMPLETED
                cd.pod_progress = 1.0
            else:
                cd.status = ClusterStatus.FAILED
                cd.error_message = "健康检查失败"
                # 单集群回滚
                self._rollback_cluster(cd, service)

        except Exception as e:
            cd.status = ClusterStatus.FAILED
            cd.error_message = str(e)

        cd.finished_at = time.time()
        return {
            "cluster": cluster.name,
            "status": cd.status.value,
            "duration_sec": cd.finished_at - cd.started_at,
            "error": cd.error_message
        }

    def _wait_for_rollout(self, cd: ClusterDeployment, service: str):
        """等待 Pod 滚动更新完成，追踪进度"""
        timeout = self.HEALTH_CHECK_TIMEOUT
        start = time.time()
        while time.time() - start < timeout:
            # 查询 ArgoCD 获取部署进度
            progress = self.argocd.get_sync_progress(cd.cluster.name, service)
            cd.pod_progress = progress.get("percent", 0) / 100.0

            if progress.get("status") == "Synced" and progress.get("health") == "Healthy":
                return
            if progress.get("status") == "Failed":
                raise RuntimeError(f"ArgoCD 同步失败: {progress.get('message')}")

            time.sleep(5)

        raise TimeoutError(f"集群 {cd.cluster.name} 滚动更新超时 ({timeout}s)")

    def _check_cluster_health(self, cd: ClusterDeployment, service: str) -> bool:
        """检查集群中服务的健康状态（通过 Prometheus）"""
        try:
            # 检查错误率和延迟
            query = (f'sum(rate(http_requests_total{{service="{service}",'
                     f'cluster="{cd.cluster.name}",status=~"5xx"}}[2m]))'
                     f'/sum(rate(http_requests_total{{service="{service}",'
                     f'cluster="{cd.cluster.name}"}}[2m]))')
            error_rate = self.prometheus.query(query)
            if error_rate and error_rate > 0.05:  # > 5% 错误率
                return False

            # 检查 Pod 就绪状态
            ready_pods = self.argocd.get_ready_pods(cd.cluster.name, service)
            desired_pods = self.argocd.get_desired_pods(cd.cluster.name, service)
            if ready_pods < desired_pods:
                return False

            return True
        except Exception:
            return False

    def _check_global_health(self, service: str) -> bool:
        """检查所有已部署集群的全局指标"""
        deployed = [cd for cd in self.deployment_state.values()
                    if cd.status == ClusterStatus.COMPLETED]
        unhealthy = 0
        for cd in deployed:
            if not self._check_cluster_health(cd, service):
                unhealthy += 1
        # 容忍 10% 的集群不健康
        return unhealthy < len(deployed) * 0.1

    def _rollback_deployed_batches(self, batches: list[DeploymentBatch],
                                   service: str):
        """回滚已部署的批次"""
        self._log("开始回滚所有已部署集群...")
        for batch in reversed(batches):
            for cd in batch.clusters:
                if cd.status == ClusterStatus.COMPLETED:
                    self._rollback_cluster(cd, service)

    def _rollback_cluster(self, cd: ClusterDeployment, service: str):
        """回滚单个集群"""
        try:
            # git revert 该集群的 manifest
            self._git_revert_manifest(cd.cluster, service, cd.old_version)
            self.argocd.sync(cd.cluster.name, service)
            cd.status = ClusterStatus.ROLLED_BACK
        except Exception as e:
            cd.error_message += f" | 回滚失败: {str(e)}"

    def _update_manifest(self, path, content):
        pass

    def _git_commit_push(self, msg):
        pass

    def _git_revert_manifest(self, cluster, service, old_version):
        pass

    def _get_current_version(self, cluster, service):
        return self.argocd.get_current_image(cluster.name, service)

    def _log(self, msg):
        print(f"[MultiClusterDeployer] {msg}")
```

**50+ 集群滚动部署效果：**

| 指标 | 全量同时部署 | 分批滚动（4 批） | 分批+全局健康检查 |
|------|-----------|---------------|----------------|
| 总部署时间 | 8 分钟（全部并行） | 20 分钟 | 22 分钟 |
| Registry 压力 | 50 并发拉取 → 超时 | 15 并发拉取 → 正常 | 15 并发拉取 → 正常 |
| 部署失败影响 | 5 个集群失败 → 手动处理 | 3 个集群失败 → 自动回滚 | 3 个集群失败 → 自动回滚+暂停 |
| 异常发现时间 | 10-30 分钟（用户反馈） | 2 分钟（批次间检查） | 30 秒（实时健康检查） |
| 回滚完整度 | 部分集群遗漏 | 按批次回滚 | 全量回滚保障 |

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
- **构建依赖图优化**：Monorepo 场景下，通过依赖图分析仅构建受影响模块 → 大型 Monorepo 构建时间从 30 分钟降至 5 分钟。
- **多架构构建**：ARM64 服务器成本低于 x86 约 30%，CI 平台支持多架构镜像构建 → 混合部署节省成本。
- **构建结果缓存（远程缓存）**：Bazel/Turbo 等工具支持远程缓存，团队成员可复用他人构建结果 → CI + 本地开发均受益。
- **暗部署（Dark Launch）**：新版本先部署但不暴露流量，内部测试通过后再逐步放量 → 比金丝雀更保守的策略，适合高风险变更。
- **可观测性反馈环**：部署后自动关联错误率、延迟、资源使用等指标，生成"部署健康报告"→ 让开发者直观感知变更影响。

## 性能与成本分析

**CI 构建资源规划：**

| 场景 | 并发构建 | 所需节点 | 月成本 |
|------|---------|---------|-------|
| 日均 5000 任务（均值） | 50 | 13 台 | ¥6.5 万 |
| 峰值 200 并发 | 200 | 50 台 | ¥25 万 |
| 弹性扩缩（均值） | 50-200 | 平均 35 台 | ¥17.5 万 |

**CD 部署性能：**

| 阶段 | 延迟 | 瓶颈 |
|------|------|------|
| 代码提交 → 构建启动 | 5-10s | Webhook 延迟 |
| 构建（Go，有缓存） | 2-3 min | CPU |
| 镜像打包 | 1-2 min | 网络（推送到 Registry） |
| 安全扫描 | 2-5 min | Trivy 扫描 |
| 审批 | 0-30 min | 人工 |
| 金丝雀部署 | 5 min | 观察期 |
| 全量发布 | 10-15 min | 逐步扩量 |
| **总计** | **20-55 min** | 审批是最大变量 |

**关键优化点：** 免审批场景（开发/测试环境、热修复）→ 总计 20-25 分钟，满足 15 分钟目标的 80% 场景。

**GitOps 同步延迟：**

| 操作 | ArgoCD 轮询模式 | Webhook 模式 |
|------|----------------|-------------|
| Git push → 集群同步 | 3 分钟 | 10-30 秒 |
| 回滚（git revert） | 3 分钟 | 10-30 秒 |
| 推荐 | 日常发布 | 紧急回滚 |

生产环境建议同时启用 Webhook + 轮询（轮询作为兜底）。

### 构建资源利用率深度分析

**各语言构建资源消耗对比：**

| 语言 | 典型项目 | CPU 峰值 | 内存峰值 | 构建时长（无缓存） | 构建时长（有缓存） | 镜像大小 |
|------|---------|---------|---------|-----------------|-----------------|---------|
| Go | 微服务 | 4 核 | 2 GB | 8 min | 2 min | 20-50 MB |
| Java/Maven | 后端服务 | 8 核 | 8 GB | 12 min | 3 min | 200-500 MB |
| Node.js | 前端/BFF | 2 核 | 1 GB | 3 min | 30s | 100-300 MB |
| Python | AI/数据 | 2 核 | 2 GB | 5 min | 1 min | 200-800 MB |
| Rust | 基础设施 | 8 核 | 6 GB | 20 min | 5 min | 10-30 MB |

**资源利用率时序分布（典型工作日）：**

```
00:00-08:00  ▏         10-20% 利用率（夜间值班构建）
08:00-10:00  ████      60% 利用率（上班第一波提交）
10:00-12:00  ████████  90% 利用率（上午高峰）
12:00-14:00  █████     50% 利用率（午间下降）
14:00-18:00  ████████  85% 利用率（下午高峰）
18:00-20:00  ████      40% 利用率（下班后收尾）
20:00-24:00  ▏         15% 利用率（夜间维护）
```

**弹性扩缩策略（基于队列深度的 HPA）：**

| 队列深度 | 动作 | 目标节点数 | 生效时间 |
|---------|------|----------|---------|
| 0-20 | 维持 | 30 台 | - |
| 21-50 | 扩容 | 40 台 | ~3 min |
| 51-100 | 扩容 | 50 台 | ~3 min |
| 101-150 | 扩容 | 60 台 | ~5 min |
| 150+ | 扩容 + 降级 | 80 台 + 降级大任务 | ~5 min |
| 队列 < 10 持续 15 min | 缩容 | -5 台 | ~1 min |

### 部署频率与稳定性关联分析

**DORA 指标对比（平台上线前 vs 上线后 6 个月）：**

| DORA 指标 | 上线前 | 上线后 | 改善幅度 |
|----------|-------|-------|---------|
| 部署频率 | 0.5 次/团队/周 | 3 次/团队/天 | 42x |
| 变更前置时间 | 3-5 天 | 25 分钟 | 170-340x |
| 变更失败率 | 15% | 3% | 5x |
| 服务恢复时间（MTTR） | 2 小时 | 8 分钟 | 15x |

**部署频率与事故率的关系（内部数据）：**

| 部署频率 | 月均事故数 | 事故平均恢复时间 | 每次事故影响用户数 |
|---------|----------|---------------|----------------|
| < 1 次/月 | 3.2 次 | 120 min | 500 万 |
| 1-4 次/月 | 2.1 次 | 45 min | 200 万 |
| 1-3 次/周 | 1.5 次 | 15 min | 50 万 |
| 每日部署 | 0.8 次 | 5 min | 10 万 |

结论：部署越频繁，单次变更范围越小，事故越少且恢复越快。

### ROI 计算模型

**平台建设成本（一次性）：**

| 项目 | 成本 | 说明 |
|------|------|------|
| 开发团队（10 人 × 6 个月） | ¥300 万 | 平台核心功能开发 |
| 基础设施搭建 | ¥50 万 | CI 集群、Registry、ArgoCD |
| 安全工具授权 | ¥30 万/年 | Trivy 企业版、Snyk、Semgrep |
| 培训与迁移 | ¥20 万 | 200 团队培训 |
| **总建设成本** | **¥400 万** | 首年 |

**平台运行成本（年度）：**

| 项目 | 月成本 | 年成本 | 说明 |
|------|-------|-------|------|
| CI 集群（弹性 35 台均值） | ¥17.5 万 | ¥210 万 | 8c16G ¥5000/台/月 |
| 镜像仓库存储 | ¥2 万 | ¥24 万 | 50 TB S3 存储 |
| 安全工具授权 | ¥2.5 万 | ¥30 万 | 持续更新 |
| 运维人力（2 人） | ¥6 万 | ¥72 万 | 平台维护（原 3 人做手动运维） |
| **年运行成本** | | **¥336 万** | |

**收益计算（年度）：**

| 收益项 | 计算方式 | 年收益 |
|-------|---------|-------|
| 减少运维人力 | 3 人手动运维 → 2 人平台维护 + 自动化，节省 1 人 | ¥60 万 |
| 减少部署事故损失 | 原年均 12 次事故 × ¥50 万 → 3 次 × ¥5 万 | ¥585 万 |
| 开发效率提升 | 200 团队 × 每团队每天节省 30 分钟 × ¥500/人时 | ¥750 万 |
| 加速上市时间 | 200 团队 × 发布频率提升 42x → 业务收益（保守估计 10% 提升） | ¥500 万 |
| 安全风险降低 | 避免重大安全事件（年均 1 次 × ¥200 万） | ¥200 万 |
| **年收益合计** | | **¥2095 万** |

**ROI 总结：**

| 指标 | 数值 |
|------|------|
| 首年总投资 | ¥736 万（建设 ¥400 万 + 运行 ¥336 万） |
| 首年总收益 | ¥2095 万 |
| 首年净收益 | ¥1359 万 |
| ROI | 185% |
| 投资回收期 | ~4 个月 |

### 成本优化建议

| 优化项 | 当前成本 | 优化后 | 节省 | 实施难度 |
|-------|---------|-------|------|---------|
| 弹性扩缩（按时段） | ¥210 万/年 | ¥168 万/年 | ¥42 万 | 低 |
| 增量构建（仅编译变更模块） | 构建时间 -40% | 节省 ¥30 万 | ¥30 万 | 中 |
| 镜像层缓存复用 | 推送时间 -50% | 节省 ¥10 万 | ¥10 万 | 低 |
| Spot 实例用于 PR 构建 | PR 构建成本 -60% | 节省 ¥25 万 | ¥25 万 | 中 |
| 缓存淘汰策略（LRU） | 存储成本 -30% | 节省 ¥7 万 | ¥7 万 | 低 |

## 异常场景完整演练

**场景 1：CI 集群资源耗尽**

```
触发：200 个团队同时提交代码 → 200 个构建排队
处理：
  1. 优先级调度：main/hotfix 优先 → feature 排队
  2. 超时检测：排队超过 30 分钟的任务 → 降级为小规格构建（2核4G → 1核2G）
  3. 弹性扩容：检测到队列 > 50 → 自动扩容 10 台节点
  4. 扩容生效时间：约 3 分钟（K8s 调度 + 镜像拉取）
  5. 极端情况：扩容来不及 → 通知团队"构建排队中，预计等待 X 分钟"
```

**场景 2：金丝雀发布后新版本导致数据库连接池耗尽**

```
触发：新版本有连接泄漏 → 5% 流量到金丝雀 → 5 分钟后连接池满
检测：
  1. 金丝雀观察器检测：canary_error_rate > stable_error_rate × 2
  2. 自动回滚：流量切回 100% stable → 金丝雀 Pod 缩容
  3. 但数据库连接池已经被金丝雀耗尽 → stable 版本也受影响
处理：
  1. 回滚后 → 重启 stable 版本的 Pod（释放泄漏的连接）
  2. 数据库侧：kill idle connections > 5 分钟
  3. 根因分析：新代码缺少 connection.close() → Code Review 增加连接泄漏检测规则
关键教训：金丝雀回滚只解决了流量问题，不一定解决资源泄漏问题
```

**场景 3：审批超时导致紧急热修复延迟**

```
触发：生产 P0 故障 → 需要紧急部署热修复 → 审批人不在
处理：
  1. 热修复自动审批（配置中标记 hotfix → 免审批）
  2. 如果未配置自动审批 → 30 分钟超时后自动拒绝 → 重新提交再次等待
  3. 改进：审批策略增加"热修复 5 分钟超时自动通过"规则
  4. 紧急联系：热修复提交后自动拨打审批人电话
```

**场景 4：构建节点在构建过程中故障**

```
触发：CI 节点 #17 在执行 Maven 全量编译时磁盘 IO 错误 → 节点不可达
影响：
  - 该节点上 4 个运行中的构建全部中断
  - 已编译 80% 的 Maven 项目需要重新开始
  - 4 个团队收到构建失败通知，但实际是基础设施问题
检测与处理：
  1. 增强调度器心跳检测：30 秒内发现节点 #17 不可达
  2. 自动将该节点上的 4 个任务重新入队（优先级提升至 200）
  3. 标记节点 #17 为 UNREACHABLE → 不再分配新任务
  4. 通知 K8s 调度器替换节点 #17（删除 Pod → 新 Pod 在其他节点启动）
  5. 约 3 分钟后新节点就绪，重新调度的任务开始执行
损失评估：
  - 4 个构建各浪费 5-10 分钟 → 团队等待时间增加 ~10 分钟
  - 如果有构建缓存（Maven .m2），重试时缓存命中 → 实际只浪费 ~3 分钟
关键改进：
  - 构建中间产物定期保存（每阶段完成时保存到 S3）
  - 重试时从最近的检查点恢复，而非从头开始
```

**场景 5：镜像仓库（Registry）不可用**

```
触发：Harbor 镜像仓库主节点磁盘满 → 无法推送新镜像
影响：
  - 所有正在打包阶段的构建失败（docker push 超时）
  - 正在部署的服务无法拉取新镜像 → ArgoCD 同步失败
  - 已部署的服务不受影响（镜像已拉取到本地）
检测与处理：
  1. 构建执行器检测到 docker push 连续失败 3 次 → 触发 Registry 健康检查
  2. 确认 Registry 不可用 → 暂停所有构建任务的 package 阶段
  3. 已在运行阶段的任务继续执行（不依赖 Registry）
  4. 运维处理：清理 Registry 磁盘（删除 > 30 天的旧 tag）/ 扩容磁盘
  5. Registry 恢复后 → 按优先级重新执行暂停的 package 阶段
预防措施：
  - Registry 配置自动清理策略：保留最近 30 天 + 最近 50 个 tag
  - Registry 多区域部署：cn-east/cn-south 各一个实例，互为备份
  - 推送失败自动重试：切换到备用 Registry 端点
  - 监控告警：Registry 磁盘使用 > 80% 时提前告警
```

**场景 6：ArgoCD 同步冲突（多人同时部署同一服务）**

```
触发：团队 A 和团队 B 同时修改同一服务的 manifest → Git 产生冲突
  - 团队 A：git push manifests/prod-east/order-service.yaml (v2.1.0)
  - 团队 B：git push manifests/prod-east/order-service.yaml (v2.2.0)
  - 后推送的团队 B 遇到 Git conflict → push 失败
影响：
  - 团队 B 的部署被阻塞
  - 如果团队 B 强制 push → 覆盖团队 A 的版本 → 生产运行错误版本
检测与处理：
  1. Git 仓库保护：manifests/ 目录禁止 force push
  2. Git pre-receive hook：检查同一文件的并发修改
  3. 部署锁机制：部署前获取服务级别的分布式锁
     - 团队 A 获取锁 → 部署中 → 团队 B 等待
     - 团队 A 部署完成 → 释放锁 → 团队 B 重新 rebase + 部署
  4. 锁超时：30 分钟未释放 → 自动释放 + 告警
代码实现要点：
  - 使用 Redis 分布式锁：SET service:order-service:deploy_lock <team_id> NX EX 1800
  - 锁粒度：service + cluster 级别（不同集群可以并行部署同一服务）
  - 锁可视化：平台 UI 显示当前部署锁状态和等待队列
关键教训：GitOps 的声明式特性不天然防止冲突，需要应用层锁来协调
```

**场景 7：金丝雀指标收集失败**

```
触发：金丝雀部署 5% 流量后 → Prometheus 查询超时（Prometheus 正在 OOM 重启）
影响：
  - 金丝雀观察器无法获取 canary/stable 指标
  - 无法判断金丝雀是否健康
  - 最坏情况：金丝雀有问题但无法检测 → 自动扩量到 100% → 全量故障
处理策略（多层防护）：
  1. 指标源降级：Prometheus 不可用 → 切换到 Thanos/VictoriaMetrics 备用查询
  2. 降级判断逻辑：
     - 指标连续 3 次获取失败 → 暂停金丝雀扩量（保持当前流量比例）
     - 暂停时间 > 10 分钟 → 自动回滚（宁可回滚，不冒险扩量）
  3. 基础健康检查兜底：即使 Prometheus 不可用，仍可通过 K8s API 检查：
     - Pod CrashLoopBackOff → 立即回滚
     - Pod 就绪探针失败 > 50% → 立即回滚
     - Pod 重启次数 > 3 次/分钟 → 立即回滚
  4. 告警升级：
     - 金丝雀观察异常 → 通知部署负责人
     - 观察超过 10 分钟 → 通知 SRE 值班
  5. 事后改进：
     - Prometheus 高可用部署（2 副本 + Thanos 长期存储）
     - 金丝雀指标采集超时阈值调低（5s → 3s）
     - 增加"金丝雀冻结"状态：保持当前流量比例，等待人工决策
关键教训：自动化系统必须有降级策略，不能因为监控系统本身故障而做出错误决策
```
## 蓝绿部署完整实现

```python
class BlueGreenDeployer:
    """蓝绿部署：零停机切换"""

    def deploy(self, service_name, new_version):
        """执行蓝绿部署"""
        deployment_id = str(uuid4())

        # 1. 确定当前活跃环境
        active = self.get_active_environment(service_name)
        target = "green" if active == "blue" else "blue"

        # 2. 在目标环境部署新版本
        self.deploy_to_environment(service_name, target, new_version)

        # 3. 健康检查：验证新环境是否正常
        healthy = self.health_check(service_name, target)
        if not healthy:
            self.destroy_environment(service_name, target)
            raise DeploymentHealthCheckFailedError(f"{target} 环境健康检查失败")

        # 4. 运行冒烟测试
        smoke_result = self.run_smoke_tests(service_name, target)
        if not smoke_result.passed:
            self.destroy_environment(service_name, target)
            raise DeploymentSmokeTestFailedError(smoke_result.failures)

        # 5. 切换流量（原子操作）
        self.switch_traffic(service_name, active, target)
        self.db.insert("deployments", {
            "deployment_id": deployment_id, "service": service_name,
            "version": new_version, "environment": target,
            "status": "active", "switched_at": now()
        })

        # 6. 监控 15 分钟
        monitor_result = self.monitor_for_duration(service_name, minutes=15)
        if not monitor_result.is_healthy:
            # 自动回滚
            self.switch_traffic(service_name, target, active)
            raise DeploymentRollbackError("切换后指标异常，已自动回滚")

        return {"deployment_id": deployment_id, "status": "deployed", "environment": target}

    def switch_traffic(self, service_name, from_env, to_env):
        """切换负载均衡器流量"""
        # 更新 Nginx upstream 配置
        self.lb_client.update_upstream(service_name, to_env)
        # 重载 Nginx（graceful）
        self.lb_client.reload()
        # 验证流量已切换
        current = self.lb_client.get_active_environment(service_name)
        if current != to_env:
            raise TrafficSwitchError(f"流量切换失败: 期望 {to_env}, 实际 {current}")

    def health_check(self, service_name, environment):
        """验证目标环境健康"""
        endpoints = self.get_health_endpoints(service_name, environment)
        for endpoint in endpoints:
            resp = requests.get(endpoint, timeout=5)
            if resp.status_code != 200:
                return False
        return True
```

## 功能开关服务

```python
class FeatureFlagService:
    """功能开关：按用户/环境评估"""

    def evaluate(self, flag_key, user_id, context=None):
        """评估功能开关"""
        flag = self.db.get_flag(flag_key)
        if not flag or not flag["enabled"]:
            return False

        # Kill switch: 全局关闭
        if flag.get("kill_switch"):
            return False

        # 环境检查
        if context and context.get("environment") not in flag.get("environments", ["production"]):
            return False

        # 用户白名单
        if user_id in flag.get("whitelist", []):
            return True

        # 百分比灰度
        percentage = flag.get("percentage", 0)
        if percentage == 0:
            return False
        if percentage == 100:
            return True

        # 一致性哈希：同一用户始终得到相同结果
        hash_val = int(hashlib.md5(f"{flag_key}:{user_id}".encode()).hexdigest(), 16)
        return (hash_val % 100) < percentage

    def set_rollout_percentage(self, flag_key, percentage):
        """设置灰度百分比"""
        self.db.update("feature_flags",
            {"percentage": percentage, "updated_at": now()},
            {"flag_key": flag_key})
        # 记录审计日志
        self.audit_log("feature_flag_rollout", flag_key=flag_key,
            percentage=percentage, operator=current_user())
```

## 异常场景补充

### 场景：构建环境污染（缓存依赖含 CVE）

```
触发：Docker 镜像中缓存了一个含 CVE 的基础库
      → 新构建复用了缓存 → CVE 进入生产环境
检测：
  1. 每次构建后扫描镜像漏洞（Trivy/Grype）
  2. 发现 CRITICAL CVE → 阻止部署
处理：
  1. 清除构建缓存 → 重新拉取基础镜像
  2. 重新构建 → 扫描通过 → 部署
  3. 已部署的版本：紧急回滚 + 补丁构建
预防：CI 中强制漏洞扫描 + 每周清除构建缓存
```

### 场景：金丝雀部署指标劣化自动回滚

```python
class CanaryAutoRollback:
    """金丝雀部署自动回滚"""
    def monitor_canary(self, service_name, canary_version):
        """监控金丝雀版本指标"""
        baseline = self.get_baseline_metrics(service_name)
        canary = self.get_canary_metrics(service_name, canary_version)

        checks = {
            "error_rate": canary.error_rate < baseline.error_rate * 1.5,
            "p99_latency": canary.p99_latency < baseline.p99_latency * 1.2,
            "cpu_usage": canary.cpu_usage < 80,
        }

        if not all(checks.values()):
            failed = [k for k, v in checks.items() if not v]
            self.rollback_canary(service_name, canary_version)
            self.alert(f"金丝雀回滚: {service_name} {canary_version}, 失败指标: {failed}")
            return False
        return True
```

### 场景：蓝绿部署中数据库迁移失败

```
触发：蓝绿切换后，新版本需要新增数据库列
      → 但数据库迁移在切换前未执行 → 新版本启动失败
检测：
  1. 新环境启动健康检查失败
  2. 日志：Unknown column 'new_field' in 'table'
处理：
  1. 数据库迁移必须向后兼容（只加列，不改列）
  2. 迁移在蓝绿切换前执行（两个版本都兼容）
  3. 如果迁移失败 → 不切换，保持当前版本
预防：数据库变更必须遵循兼容性规则 + 自动化迁移检查
```

## Pipeline as Code 完整实现

```yaml
# GitLab CI Pipeline 完整定义
stages:
  - build
  - test
  - security
  - deploy-canary
  - verify
  - deploy-full

variables:
  SERVICE_NAME: "payment-service"
  DOCKER_REGISTRY: "registry.example.com"

.build_template:
  stage: build
  script:
    - docker build --cache-from $DOCKER_REGISTRY/$SERVICE_NAME:latest
                   -t $DOCKER_REGISTRY/$SERVICE_NAME:$CI_COMMIT_SHA .
    - docker push $DOCKER_REGISTRY/$SERVICE_NAME:$CI_COMMIT_SHA
  after_script:
    - docker rmi $DOCKER_REGISTRY/$SERVICE_NAME:$CI_COMMIT_SHA  # 清理本地镜像

build_java:
  extends: .build_template
  script:
    - mvn package -DskipTests -T 4  # 4 线程并行编译
    - docker build -t $DOCKER_REGISTRY/$SERVICE_NAME:$CI_COMMIT_SHA .
    - docker push $DOCKER_REGISTRY/$SERVICE_NAME:$CI_COMMIT_SHA

test_unit:
  stage: test
  script:
    - mvn test -Dtest="*UnitTest" -T 4
  artifacts:
    reports:
      junit: target/surefire-reports/*.xml

test_integration:
  stage: test
  script:
    - docker-compose -f docker-compose.test.yml up -d
    - mvn verify -Dtest="*IntegrationTest"
    - docker-compose -f docker-compose.test.yml down

security_scan:
  stage: security
  script:
    - trivy image --severity HIGH,CRITICAL $DOCKER_REGISTRY/$SERVICE_NAME:$CI_COMMIT_SHA
    - mvn org.owasp:dependency-check-maven:check

deploy_canary:
  stage: deploy-canary
  script:
    - kubectl set image deployment/$SERVICE_NAME
        $SERVICE_NAME=$DOCKER_REGISTRY/$SERVICE_NAME:$CI_COMMIT_SHA
        --namespace=canary
    - kubectl rollout status deployment/$SERVICE_NAME --namespace=canary

verify_canary:
  stage: verify
  script:
    - ./scripts/verify-canary.sh $SERVICE_NAME  # 检查 error_rate, p99_latency
  rules:
    - if: '$CANARY_VERIFY_PASSED == "true"'

deploy_full:
  stage: deploy-full
  script:
    - kubectl set image deployment/$SERVICE_NAME
        $SERVICE_NAME=$DOCKER_REGISTRY/$SERVICE_NAME:$CI_COMMIT_SHA
        --namespace=production
    - kubectl rollout status deployment/$SERVICE_NAME --namespace=production
  when: manual  # 手动确认全量部署
```

## 性能分析详细数据

| 指标 | 数值 | 说明 |
|------|------|------|
| 构建时间(Java) | 8 分钟 | 含编译+打包+镜像构建 |
| 构建时间(Node) | 3 分钟 | npm install + docker build |
| 构建时间(Go) | 2 分钟 | 最快编译 |
| 单元测试时间 | 2 分钟 | 4 线程并行 |
| 集成测试时间 | 5 分钟 | 含 Docker Compose 启动 |
| 安全扫描时间 | 1 分钟 | Trivy + OWASP |
| 部署时间(Canary) | 3 分钟 | Kubernetes rollout |
| 验证时间 | 10 分钟 | 观察指标变化 |
| 全量部署时间 | 5 分钟 | 手动确认后执行 |
| **总流水线时间** | **~36 分钟** | 从 push 到生产 |

**月度成本估算：**

| 项目 | 计算 | 月成本 |
|------|------|-------|
| CI 运行时间 | 36min × 500 builds/day × $0.10/min | ¥5.4 万 |
| 制品存储 | 500MB × 500 builds × S3 ¥0.12/GB | ¥0.3 万 |
| Kubernetes 集群 | 3 个 namespace × 5 nodes | ¥3 万 |
| Docker Registry | 500 images × 200MB | ¥0.5 万 |
| **合计** | | **¥9.2 万** |

## 制品管理完整实现

```python
class ArtifactManager:
    """制品管理：构建产物存储、校验、晋升"""

    def upload_artifact(self, build_id, service_name, version):
        """上传构建制品"""
        artifact_id = str(uuid4())
        # 1. 计算文件 SHA256
        checksum = self._compute_sha256(f"/tmp/{build_id}.tar.gz")

        # 2. 上传到 S3
        s3_key = f"artifacts/{service_name}/{version}/{build_id}.tar.gz"
        self.s3_client.upload(f"/tmp/{build_id}.tar.gz", s3_key)

        # 3. 存储制品元数据
        self.db.insert("artifacts", {
            "artifact_id": artifact_id,
            "build_id": build_id,
            "service_name": service_name,
            "version": version,
            "s3_key": s3_key,
            "checksum_sha256": checksum,
            "size_bytes": os.path.getsize(f"/tmp/{build_id}.tar.gz"),
            "status": "dev",  # dev → staging → production
            "created_at": now()
        })
        return artifact_id

    def promote_artifact(self, artifact_id, target_env):
        """制品晋升：dev → staging → production"""
        artifact = self.db.get_artifact(artifact_id)

        # 验证 checksum
        downloaded = self.s3_client.download_temp(artifact["s3_key"])
        actual_checksum = self._compute_sha256(downloaded)
        if actual_checksum != artifact["checksum_sha256"]:
            raise ArtifactCorruptedError("Checksum mismatch: artifact corrupted")

        # 漏洞扫描
        vulns = self.vulnerability_scanner.scan(downloaded)
        if any(v["severity"] == "CRITICAL" for v in vulns):
            raise VulnerabilityFoundError(f"Critical CVE found: {vulns}")

        # 晋升
        env_order = ["dev", "staging", "production"]
        current_idx = env_order.index(artifact["status"])
        target_idx = env_order.index(target_env)
        if target_idx <= current_idx:
            raise InvalidPromotionError("只能向前晋升")

        self.db.update("artifacts",
            {"status": target_env, "promoted_at": now()},
            {"artifact_id": artifact_id})

    def cleanup_old_artifacts(self):
        """清理旧版本制品（保留最近 10 个）"""
        for service in self.get_all_services():
            artifacts = self.db.query(
                "SELECT * FROM artifacts WHERE service_name = %s "
                "AND status = 'production' ORDER BY created_at DESC", service)
            # 保留最近 10 个，其余归档到 Glacier
            for artifact in artifacts[10:]:
                self.s3_client.move_to_glacier(artifact["s3_key"])
                self.db.update("artifacts",
                    {"storage_class": "glacier"}, {"artifact_id": artifact["id"]})
```

## 部署回滚自动化

```python
class DeploymentRollbackService:
    """部署回滚自动化"""

    def rollback(self, service_name, target_version=None):
        """回滚到指定版本（默认上一个版本）"""
        if not target_version:
            # 获取上一个稳定版本
            target_version = self.db.query_one(
                "SELECT version FROM deployment_history "
                "WHERE service_name = %s AND status = 'stable' "
                "ORDER BY deployed_at DESC LIMIT 1 OFFSET 1",
                service_name)["version"]

        # 1. 保存当前版本（用于后续回滚）
        current = self.kubectl.get_current_version(service_name)

        # 2. 执行回滚
        self.kubectl.set_image(service_name, target_version)
        self.kubectl.rollout_status(service_name, timeout=120)

        # 3. 健康检查
        healthy = self.health_check(service_name)
        if not healthy:
            # 回滚也失败 → 紧急状态
            self.alert(f"回滚失败: {service_name}，当前版本 {current}，目标版本 {target_version}")
            return {"status": "rollback_failed", "current_version": current}

        # 4. 数据库迁移回滚（如有）
        migration_result = self.db_migration.rollback(service_name, target_version)

        self.db.insert("rollback_history", {
            "service_name": service_name,
            "from_version": current, "to_version": target_version,
            "reason": "auto_rollback", "rollback_at": now()
        })
        return {"status": "rollback_success", "new_version": target_version}
```

## 异常场景补充

### 场景：制品损坏检测

```
触发：S3 上的制品文件损坏（下载后 checksum 不匹配）
检测：promote_artifact 时校验 SHA256
处理：
  1. 标记制品为 corrupted
  2. 从 Glacier 恢复备份版本
  3. 重新构建该版本
预防：上传时计算 checksum + 定期校验存储中的制品
```

### 场景：Rolling Update 卡住

```
触发：Kubernetes rolling update 卡住：新 Pod 无法启动
      → 旧 Pod 已被替换 → 服务中断
检测：
  1. rollout status 超时 5 分钟 → 告警
  2. 新 Pod CrashLoopBackOff → 严重告警
处理：
  1. 立即暂停 rollout: kubectl rollout pause
  2. 检查新 Pod 日志 → 定位启动失败原因
  3. 回滚: kubectl rollout undo
  4. 如果回滚也失败 → 手动恢复旧版本
预防：使用 maxSurge=1, maxUnavailable=0（先扩再缩）
```

### 场景：Secret 轮换失败

```
触发：数据库密码轮换 → 新密码写入 Vault → 但应用未刷新
      → 应用用旧密码连接 → 连接失败 → 服务中断
检测：
  1. 连接池错误率 > 50% → 告警
  2. Secret 版本与应用使用版本不一致 → 告警
处理：
  1. 回滚 Vault 中的密码到旧版本
  2. 应用自动重连 → 恢复服务
  3. 重新轮换：先部署新密码 → 应用读取新密码 → 确认连接正常 → 删除旧密码
预防：Secret 轮换分两步：1) 通知应用读取新密码 2) 确认全部应用已更新 3) 删除旧密码
```

## 环境配置自动化完整实现

```python
class EnvironmentProvisioner:
    """环境配置自动化：dev/staging/production"""

    def provision(self, env_name, service_name, config):
        """配置新环境"""
        env_id = str(uuid4())
        namespace = f"{env_name}-{service_name}"

        # 1. 创建 Kubernetes Namespace
        self.kubectl.create_namespace(namespace)

        # 2. 部署基础资源
        resources = [
            {"type": "Deployment", "template": "deployment.yaml"},
            {"type": "Service", "template": "service.yaml"},
            {"type": "ConfigMap", "template": "configmap.yaml"},
            {"type": "Secret", "template": "secret.yaml"},
        ]
        for resource in resources:
            rendered = self.helm.render(resource["template"], {
                "namespace": namespace,
                "env": env_name,
                "service": service_name,
                "config": config
            })
            self.kubectl.apply(rendered, namespace=namespace)

        # 3. 等待 Pod 就绪
        self.kubectl.wait_for_rollout(namespace, service_name, timeout=300)

        # 4. 健康检查
        endpoint = self.get_service_endpoint(namespace, service_name)
        healthy = self.health_check(endpoint)
        if not healthy:
            self.teardown(namespace)
            raise EnvironmentProvisionError(f"环境健康检查失败: {endpoint}")

        # 5. 注册环境
        self.db.insert("environments", {
            "env_id": env_id, "name": env_name,
            "service": service_name, "namespace": namespace,
            "endpoint": endpoint, "status": "active",
            "created_at": now()
        })
        return {"env_id": env_id, "namespace": namespace, "endpoint": endpoint}

    def teardown(self, namespace):
        """销毁环境"""
        self.kubectl.delete_namespace(namespace)
        self.db.update("environments",
            {"status": "destroyed", "destroyed_at": now()},
            {"namespace": namespace})

    def clone_environment(self, source_env, target_env_name):
        """克隆环境（用于测试）"""
        source = self.db.get_environment(source_env)
        config = self.db.get_environment_config(source["env_id"])
        # 复制数据库
        self.db_clone(source["namespace"], f"{target_env_name}-{source['service']}")
        # 配置新环境
        return self.provision(target_env_name, source["service"], config)
```

## 部署指标仪表盘

```python
class DeploymentMetricsDashboard:
    """DORA 四大关键指标"""

    def get_metrics(self, service_name, period_days=30):
        """计算 DORA 指标"""
        return {
            "deployment_frequency": self._calc_deploy_frequency(service_name, period_days),
            "lead_time": self._calc_lead_time(service_name, period_days),
            "mttr": self._calc_mttr(service_name, period_days),
            "change_failure_rate": self._calc_change_failure_rate(service_name, period_days)
        }

    def _calc_deploy_frequency(self, service, days):
        """部署频率"""
        count = self.db.count("deployments", service=service,
            deployed_at__gte=now() - timedelta(days=days))
        return {"count": count, "per_day": count / days}

    def _calc_lead_time(self, service, days):
        """变更前置时间（commit → production）"""
        deploys = self.db.query(
            "SELECT AVG(EXTRACT(EPOCH FROM (deployed_at - committed_at))/3600) as hours "
            "FROM deployments WHERE service = %s AND deployed_at > NOW() - INTERVAL '%s DAY'",
            service, days)
        return {"avg_hours": deploys[0]["hours"]}

    def _calc_mttr(self, service, days):
        """平均恢复时间"""
        incidents = self.db.query(
            "SELECT AVG(EXTRACT(EPOCH FROM (resolved_at - detected_at))/60) as minutes "
            "FROM incidents WHERE service = %s AND detected_at > NOW() - INTERVAL '%s DAY'",
            service, days)
        return {"avg_minutes": incidents[0]["minutes"]}

    def _calc_change_failure_rate(self, service, days):
        """变更失败率"""
        total = self.db.count("deployments", service=service,
            deployed_at__gte=now() - timedelta(days=days))
        failed = self.db.count("deployments", service=service,
            deployed_at__gte=now() - timedelta(days=days), status="failed")
        return {"rate": failed / max(total, 1), "total": total, "failed": failed}
```

## 回滚策略矩阵

| 服务类型 | 代码回滚 | 配置回滚 | 数据库回滚 | 优先级 |
|---------|---------|---------|-----------|-------|
| 无状态服务 | kubectl rollout undo | git revert config | N/A | 自动 |
| 有状态服务 | kubectl rollout undo | git revert config | migration rollback | 人工确认 |
| 数据库迁移 | 不适用 | N/A | reverse migration | 人工确认 |
| 配置变更 | 不适用 | git revert + reload | N/A | 自动 |
| Feature Flag | 不适用 | 关闭 flag | N/A | 自动（即时） |

**自动 vs 手动回滚决策树：**

```
检测到异常？
├─ 错误率 > 5% → 自动回滚
├─ P99 延迟 > 2x 基线 → 自动回滚
├─ 错误率 1-5% → 人工确认（5 分钟超时 → 自动回滚）
└─ 数据库变更 → 必须人工确认
```

## 异常场景补充

### 场景：Terraform State Lock 冲突

```
触发：两个 CI 流水线同时运行 terraform apply
      → 状态文件被锁定 → 第二个执行失败
检测：Terraform 返回 "state already locked" 错误
处理：
  1. 等待前一个执行完成 → 自动重试
  2. 如果锁超时（> 30 分钟）→ 强制解锁
  3. 强制解锁前确认没有正在执行的 apply
预防：CI 流水线串行执行 terraform + 分布式锁
```

### 场景：部署到错误环境

```
触发：开发人员误将 staging 镜像部署到 production
检测：
  1. 部署前检查：镜像标签是否匹配目标环境
  2. production 只允许 "release-*" 标签
  3. 非 release 标签 → 阻止部署
处理：
  1. 自动阻止不合规的部署
  2. 通知开发人员正确打标签
预防：部署门禁 + 镜像标签策略 + RBAC 权限控制
```

### 场景：Helm Chart 版本不匹配

```
触发：Helm Chart 引用了不存在的镜像版本
      → Pod ImagePullBackOff → 部署失败
检测：
  1. 部署前验证：检查镜像是否存在于 Registry
  2. Pod 启动失败 → ImagePullBackOff → 告警
处理：
  1. 回滚到上一个 Chart 版本
  2. 修正镜像标签 → 重新部署
预防：部署前 dry-run 验证 + 镜像存在性检查
```

## GitOps 完整实现

```python
class GitOpsController:
    """GitOps：Git 仓库作为唯一事实来源"""

    def sync_cluster_state(self):
        """同步 Git 仓库状态到 Kubernetes 集群"""
        # 1. 拉取 Git 仓库最新配置
        self.git.pull()

        # 2. 对比 Git 声明状态 vs 集群实际状态
        desired = self._load_desired_state()
        actual = self._get_actual_state()

        drift = self._detect_drift(desired, actual)
        if not drift:
            return {"status": "in_sync", "drift_count": 0}

        # 3. 自动修复漂移
        for resource in drift:
            try:
                self.kubectl.apply(resource["manifest"])
                self.db.insert("gitops_sync_log", {
                    "resource": resource["key"],
                    "action": "synced",
                    "synced_at": now()
                })
            except Exception as e:
                self.alert(f"GitOps 同步失败: {resource['key']}, error: {str(e)}")

        return {"status": "synced", "drift_count": len(drift)}

    def _detect_drift(self, desired, actual):
        """检测配置漂移"""
        drift = []
        for key, desired_resource in desired.items():
            actual_resource = actual.get(key)
            if not actual_resource:
                drift.append({"key": key, "type": "missing", "manifest": desired_resource})
            elif actual_resource != desired_resource:
                drift.append({"key": key, "type": "changed", "manifest": desired_resource})
        return drift
```

## Secret 管理完整实现

```python
class SecretManager:
    """Secret 管理：自动轮换和注入"""

    def inject_secrets(self, namespace, service_name):
        """将 Secret 注入到 Pod 环境变量"""
        secrets = self.vault.read(f"secret/{service_name}")
        # 创建 Kubernetes Secret
        self.kubectl.create_or_update_secret(
            name=f"{service_name}-secrets",
            namespace=namespace,
            data={k: base64.b64encode(v.encode()).decode() for k, v in secrets.items()})

    def rotate_database_password(self, service_name):
        """数据库密码自动轮换"""
        # 1. 生成新密码
        new_password = self._generate_password()

        # 2. 在数据库中更新密码（双密码兼容期）
        self.db.execute(f"ALTER USER {service_name} IDENTIFIED BY '{new_password}'")

        # 3. 更新 Vault
        self.vault.write(f"secret/{service_name}", {"db_password": new_password})

        # 4. 触发 Pod 重启以加载新 Secret
        self.kubectl.restart_deployment(service_name)

        # 5. 验证新密码连接
        healthy = self._verify_connection(service_name, new_password)
        if not healthy:
            self.alert(f"密码轮换后连接验证失败: {service_name}")
```

## DORA 指标基准

| 级别 | 部署频率 | 变更前置时间 | MTTR | 变更失败率 |
|------|---------|------------|------|----------|
| 精英 | 按需(多次/天) | <1小时 | <1小时 | <5% |
| 高 | 每周 | 1天-1周 | <1天 | 5-10% |
| 中 | 每月 | 1周-1月 | 1天-1周 | 10-15% |
| 低 | 每季度 | >1月 | >1周 | >15% |
| **当前** | **50次/天** | **~4小时** | **<30分钟** | **~8%** |

## CI 流水线模板库

```yaml
# ============================================
# 模板 1：Java/Maven 流水线
# ============================================
apiVersion: ci.devops.io/v1
kind: PipelineTemplate
metadata:
  name: java-maven-pipeline
  labels:
    language: java
    build-tool: maven
spec:
  parameters:
    - name: mavenGoals
      default: "clean verify"
    - name: jdkVersion
      default: "17"
    - name: skipTests
      default: "false"
  stages:
    - name: 代码检出与缓存
      tasks:
        - name: git-clone
          action: git.clone
          params:
            depth: 50
            fetchTags: true
        - name: maven-cache-restore
          action: cache.restore
          params:
            key: maven-{{ .Branch }}-{{ .Checksum "pom.xml" }}
            fallbackKeys:
              - maven-{{ .Branch }}-
              - maven-main-
            path: /root/.m2/repository

    - name: 编译与单元测试
      parallel: true
      tasks:
        - name: maven-compile
          action: maven.run
          params:
            goals: "compile -T 4C"
            jdkVersion: "{{ .jdkVersion }}"
            env:
              MAVEN_OPTS: "-Xmx2g -XX:+UseG1GC"
        - name: maven-unit-test
          action: maven.run
          params:
            goals: "test -Dmaven.test.failure.ignore=false"
            jdkVersion: "{{ .jdkVersion }}"
            reports:
              - path: target/surefire-reports
                format: junit

    - name: 集成测试
      tasks:
        - name: maven-integration-test
          action: maven.run
          params:
            goals: "verify -DskipUnitTests=true"
            services:
              - name: postgres
                image: postgres:15
                env:
                  POSTGRES_DB: testdb
                  POSTGRES_USER: test
                  POSTGRES_PASSWORD: test
              - name: redis
                image: redis:7-alpine

    - name: 构建与版本管理
      tasks:
        - name: maven-package
          action: maven.run
          params:
            goals: "package -DskipTests"
        - name: version-stamp
          action: version.generate
          params:
            strategy: gitsha-short
            format: "{{ .Branch }}-{{ .ShortSHA }}-{{ .Timestamp }}"

    - name: 镜像构建与推送
      tasks:
        - name: docker-build
          action: docker.build
          params:
            dockerfile: Dockerfile
            context: .
            tags:
              - "{{ .Registry }}/{{ .Team }}/{{ .Service }}:{{ .Version }}"
              - "{{ .Registry }}/{{ .Team }}/{{ .Service }}:latest-{{ .Branch }}"
            buildArgs:
              JAR_FILE: target/*.jar
              BASE_IMAGE: eclipse-temurin:{{ .jdkVersion }}-jre-alpine
        - name: trivy-scan
          action: security.scan
          params:
            image: "{{ .Registry }}/{{ .Team }}/{{ .Service }}:{{ .Version }}"
            severity: "CRITICAL,HIGH"
            exitCode: 1

    - name: 缓存保存
      tasks:
        - name: maven-cache-save
          action: cache.save
          params:
            key: maven-{{ .Branch }}-{{ .Checksum "pom.xml" }}
            path: /root/.m2/repository

---
# ============================================
# 模板 2：Node/npm 流水线
# ============================================
apiVersion: ci.devops.io/v1
kind: PipelineTemplate
metadata:
  name: node-npm-pipeline
  labels:
    language: node
    build-tool: npm
spec:
  parameters:
    - name: nodeVersion
      default: "20"
    - name: packageManager
      default: "npm"
  stages:
    - name: 安装依赖
      tasks:
        - name: npm-ci
          action: npm.install
          params:
            packageManager: "{{ .packageManager }}"
            frozenLockfile: true
        - name: cache-restore
          action: cache.restore
          params:
            key: node-{{ .Branch }}-{{ .Checksum "package-lock.json" }}
            path: node_modules

    - name: 代码质量
      parallel: true
      tasks:
        - name: lint
          action: npm.run
          params:
            script: lint
        - name: type-check
          action: npm.run
          params:
            script: typecheck
        - name: unit-test
          action: npm.run
          params:
            script: test:coverage
            reports:
              - path: coverage/lcov.info
                format: lcov

    - name: 构建
      tasks:
        - name: build
          action: npm.run
          params:
            script: build
            env:
              NODE_ENV: production
              VITE_APP_VERSION: "{{ .Version }}"

    - name: 镜像与安全
      tasks:
        - name: docker-build
          action: docker.build
          params:
            dockerfile: Dockerfile
            tags:
              - "{{ .Registry }}/{{ .Team }}/{{ .Service }}:{{ .Version }}"
            buildArgs:
              NODE_VERSION: "{{ .nodeVersion }}"

---
# ============================================
# 模板 3：Go 流水线
# ============================================
apiVersion: ci.devops.io/v1
kind: PipelineTemplate
metadata:
  name: go-pipeline
  labels:
    language: go
spec:
  parameters:
    - name: goVersion
      default: "1.22"
  stages:
    - name: 编译与测试
      parallel: true
      tasks:
        - name: go-build
          action: go.build
          params:
            output: bin/service
            ldflags: "-s -w -X main.version={{ .Version }}"
            goVersion: "{{ .goVersion }}"
        - name: go-test
          action: go.test
          params:
            race: true
            coverProfile: coverage.out
            packages: ./...
        - name: go-vet
          action: go.vet
          params:
            packages: ./...

    - name: 镜像构建
      tasks:
        - name: docker-build
          action: docker.build
          params:
            dockerfile: Dockerfile
            tags:
              - "{{ .Registry }}/{{ .Team }}/{{ .Service }}:{{ .Version }}"
            buildArgs:
              GO_VERSION: "{{ .goVersion }}"
              ALPINE_VERSION: "3.19"

---
# ============================================
# 模板 4：Python 流水线
# ============================================
apiVersion: ci.devops.io/v1
kind: PipelineTemplate
metadata:
  name: python-pipeline
  labels:
    language: python
spec:
  parameters:
    - name: pythonVersion
      default: "3.12"
  stages:
    - name: 依赖安装
      tasks:
        - name: pip-install
          action: pip.install
          params:
            requirements: requirements.txt
            virtualEnv: true
            pythonVersion: "{{ .pythonVersion }}"
        - name: cache-restore
          action: cache.restore
          params:
            key: pip-{{ .Branch }}-{{ .Checksum "requirements.txt" }}
            path: .venv

    - name: 代码质量
      parallel: true
      tasks:
        - name: lint
          action: python.run
          params:
            command: ruff check .
        - name: type-check
          action: python.run
          params:
            command: mypy src/
        - name: unit-test
          action: python.run
          params:
            command: pytest --cov=src --junitxml=report.xml
            reports:
              - path: report.xml
                format: junit
              - path: coverage.xml
                format: coverage

    - name: 构建与镜像
      tasks:
        - name: docker-build
          action: docker.build
          params:
            dockerfile: Dockerfile
            tags:
              - "{{ .Registry }}/{{ .Team }}/{{ .Service }}:{{ .Version }}"
            buildArgs:
              PYTHON_VERSION: "{{ .pythonVersion }}"
```

## 部署策略对比与决策树

### 四种策略对比

| 维度 | 蓝绿部署 | 金丝雀发布 | 滚动更新 | Feature Flag |
|------|---------|-----------|---------|-------------|
| 原理 | 两套完整环境切换 | 小流量验证后逐步放量 | 逐个替换旧实例 | 代码层面控制功能开关 |
| 回滚速度 | 秒级（切流量） | 分钟级（缩减金丝雀） | 分钟级（重新部署旧版） | 秒级（关闭开关） |
| 资源开销 | 2x（双倍环境） | 1.1-1.5x（金丝雀实例） | 1x（逐步替换） | 1x（无额外资源） |
| 流量控制 | 全量切换 | 精细百分比控制 | 按 Pod 比例 | 按用户/租户/地区 |
| 适用场景 | 需要零停机的核心服务 | 风险敏感的业务服务 | 一般性服务迭代 | 需要灰度测试的功能 |
| 复杂度 | 中（需双环境管理） | 高（需流量分割与监控） | 低（K8s 原生支持） | 中（需 Flag 管理平台） |
| 数据库兼容 | 需双向兼容 | 需双向兼容 | 需双向兼容 | 需双向兼容 |
| 测试深度 | 切换前全量验证 | 生产流量真实验证 | 有限验证 | 指定用户群体验证 |

### 决策树

```
开始部署决策
│
├─ 该服务是否为核心交易链路？
│   ├─ 是 → 是否允许任何停机时间？
│   │   ├─ 否（零停机） → 蓝绿部署
│   │   └─ 是（可接受<5s） → 金丝雀发布
│   └─ 否 → 是否需要按用户维度灰度？
│       ├─ 是 → Feature Flag
│       └─ 否 → 变更是否涉及数据库 Schema 变更？
│           ├─ 是 → 蓝绿部署（切换前后需双兼容）
│           └─ 否 → 滚动更新
│
└─ 特殊场景：
    ├─ 大版本重构 → 蓝绿部署（旧版随时可切回）
    ├─ A/B 测试 → Feature Flag + 金丝雀
    ├─ 紧急 Hotfix → 滚动更新（速度优先）
    └─ 多租户隔离 → Feature Flag（按租户控制）
```

### 风险矩阵

```
                    影响范围
               小           大
         ┌──────────┬──────────┐
  高     │ 金丝雀    │ 蓝绿     │
风       │ (可控流量) │ (完整回滚)│
险       ├──────────┼──────────┤
程       │ 滚动更新  │ Feature  │
度       │ (逐步替换)│  Flag    │
  低     │          │ (精准控制)│
         └──────────┴──────────┘
```

### 策略组合模式

```
推荐组合：Feature Flag + 金丝雀发布

阶段 1：代码合并，Feature Flag 关闭 → 无用户影响
阶段 2：开启 Flag，金丝雀 1% 流量 → 内部测试
阶段 3：金丝雀 10% 流量 → 小范围验证
阶段 4：金丝雀 50% 流量 → 半量对比
阶段 5：全量发布 → 稳定后移除 Flag

如果任何阶段发现问题：
  - Feature Flag 秒级关闭
  - 金丝雀回滚到 0%
  - 双重保障，风险最低
```

## 基础设施即代码：Pulumi 实现

### VPC + EKS + RDS 完整配置

```python
import pulumi
import pulumi_aws as aws
import pulumi_eks as eks
import pulumi_awsx as awsx

# ============================================
# VPC 网络层
# ============================================
vpc = awsx.ec2.Vpc("devops-vpc",
    cidr_block="10.0.0.0/16",
    enable_dns_hostnames=True,
    enable_dns_support=True,
    nat_gateways={
        "strategy": "single",  # 成本优化：单 NAT
    },
    subnet_specs=[
        awsx.ec2.SubnetSpecArgs(
            type=awsx.ec2.SubnetType.PUBLIC,
            name="public",
            cidr_mask=24,
        ),
        awsx.ec2.SubnetSpecArgs(
            type=awsx.ec2.SubnetType.PRIVATE_WITH_NAT,
            name="private-app",
            cidr_mask=22,  # /22 支持 1024 IP，EKS Pod 用
            tags={"Tier": "app"},
        ),
        awsx.ec2.SubnetSpecArgs(
            type=awsx.ec2.SubnetType.PRIVATE_ISOLATED,
            name="private-db",
            cidr_mask=24,
            tags={"Tier": "db"},
        ),
    ],
    tags={
        "Environment": pulumi.get_stack(),
        "ManagedBy": "pulumi",
    })

# ============================================
# EKS 集群
# ============================================
cluster = eks.Cluster("devops-eks",
    vpc_id=vpc.vpc_id,
    public_subnet_ids=vpc.public_subnet_ids,
    private_subnet_ids=vpc.private_subnet_ids,
    cluster_security_group_tags={
        "kubernetes.io/cluster/devops-eks": "owned",
    },
    version="1.29",
    endpoint_private_access=True,
    endpoint_public_access=False,  # 安全：仅私有访问
    node_group=eks.NodeGroupArgs(
        instance_type="m6i.xlarge",
        desired_capacity=3,
        min_size=2,
        max_size=10,
        labels={"role": "workload"},
        taints=[{
            "key": "workload",
            "value": "true",
            "effect": "NO_SCHEDULE",
        }],
    ),
    node_security_group_tags={
        "kubernetes.io/cluster/devops-eks": "owned",
    },
    tags={
        "Environment": pulumi.get_stack(),
        "CostCenter": "devops-platform",
    })

# ============================================
# RDS 数据库
# ============================================
db_subnet_group = aws.rds.SubnetGroup("devops-db-subnet",
    subnet_ids=vpc.isolated_subnet_ids,
    tags={"Environment": pulumi.get_stack()})

db_security_group = aws.ec2.SecurityGroup("devops-db-sg",
    vpc_id=vpc.vpc_id,
    description="RDS access from EKS",
    ingress=[aws.ec2.SecurityGroupIngressArgs(
        protocol="tcp",
        from_port=5432,
        to_port=5432,
        security_groups=[cluster.node_security_group.id],
    )],
    egress=[aws.ec2.SecurityGroupEgressArgs(
        protocol="-1",
        from_port=0,
        to_port=0,
        cidr_blocks=["0.0.0.0/0"],
    )])

database = aws.rds.Instance("devops-db",
    engine="aurora-postgresql",
    engine_version="15.4",
    instance_class="db.r6g.large",
    allocated_storage=100,
    storage_encrypted=True,
    db_subnet_group_name=db_subnet_group.name,
    vpc_security_group_ids=[db_security_group.id],
    username="dbadmin",
    password=pulumi.Config().require_secret("db-password"),
    backup_retention_period=7,
    backup_window="03:00-04:00",
    maintenance_window="Mon:04:00-Mon:05:00",
    skip_final_snapshot=False,
    tags={
        "Environment": pulumi.get_stack(),
        "BackupPolicy": "daily",
    })

# ============================================
# 环境变量注入
# ============================================
pulumi.export("cluster_endpoint", cluster.core.endpoint)
pulumi.export("cluster_name", cluster.eks_cluster.name)
pulumi.export("db_endpoint", database.endpoint)
pulumi.export("vpc_id", vpc.vpc_id)

# 将关键输出写入 SSM Parameter Store → K8s ExternalSecrets 读取
for param_name, param_value in [
    ("DB_ENDPOINT", database.endpoint),
    ("DB_NAME", pulumi.Output.from_input("devops")),
    ("CLUSTER_NAME", cluster.eks_cluster.name),
]:
    aws.ssm.Parameter(f"env-{param_name}",
        name=f"/devops/{pulumi.get_stack()}/{param_name}",
        type="SecureString",
        value=param_value,
        tags={"Environment": pulumi.get_stack()})
```

### 漂移检测与自动修正

```python
class DriftDetector:
    """基础设施漂移检测与修正"""

    def __init__(self, pulumi_backend_url):
        self.backend = pulumi_backend_url
        self.stack_name = "devops/production"

    def detect_drift(self):
        """检测实际基础设施与期望状态之间的漂移"""
        # 1. 运行 pulumi preview 检测漂移
        result = self._run_pulumi_preview()
        drifts = []
        for change in result["changes"]:
            if change["kind"] == "update" or change["kind"] == "delete":
                drifts.append({
                    "resource": change["urn"],
                    "kind": change["kind"],
                    "detail": change.get("diff", {}),
                    "severity": "HIGH" if change["kind"] == "delete" else "MEDIUM",
                })
        return drifts

    def auto_remediate(self, drifts, dry_run=True):
        """自动修正漂移"""
        remediated = []
        for drift in drifts:
            if drift["severity"] == "HIGH" and not dry_run:
                # 高危漂移：立即修正
                result = self._run_pulumi_up()
                remediated.append({
                    "resource": drift["resource"],
                    "action": "remediated",
                    "result": result,
                })
            else:
                # 中低危：发送通知
                self._notify_drift(drift)
                remediated.append({
                    "resource": drift["resource"],
                    "action": "notified",
                    "dry_run": dry_run,
                })
        return remediated

    def _run_pulumi_preview(self):
        import subprocess
        result = subprocess.run(
            ["pulumi", "preview", "--json", "--non-interactive"],
            capture_output=True, text=True)
        return json.loads(result.stdout)

    def _run_pulumi_up(self):
        import subprocess
        result = subprocess.run(
            ["pulumi", "up", "--yes", "--non-interactive"],
            capture_output=True, text=True)
        return {"success": result.returncode == 0, "output": result.stdout[-500:]}

    def _notify_drift(self, drift):
        """发送漂移告警"""
        message = (
            f"基础设施漂移检测\n"
            f"资源: {drift['resource']}\n"
            f"类型: {drift['kind']}\n"
            f"严重度: {drift['severity']}\n"
            f"详情: {drift['detail']}"
        )
        self.slack.send("#infra-alerts", message)
```

## 异常场景补充

### 场景：CI Runner OOM

```
触发：大型 Maven 项目全量编译时，CI Runner 内存不足
      → OOM Killer 杀掉 Java 进程 → 构建失败
      → 错误信息："Process terminated (SIGKILL)" 或 "Java heap space"
检测：
  1. 构建日志中出现 "Cannot allocate memory" 或 SIGKILL
  2. Runner 节点可用内存 < 500MB → 资源告警
  3. 构建任务状态变为 failed，退出码 137（OOM Killed）
处理：
  1. 立即：在更大规格节点上重新触发构建
  2. 短期：
     - 增加 Runner 节点内存（8c16G → 8c32G）
     - 限制 MAVEN_OPTS="-Xmx4g"（避免单任务占满内存）
     - 设置构建内存请求/限制（K8s Resource Limit）
  3. 长期：
     - 按 CPU/内存需求分类 Runner（小/中/大/超大）
     - 构建模板声明资源需求 → 调度到匹配节点
     - 实现构建资源弹性扩缩容
预防：
  - 构建模板强制声明资源需求
  - Runner 节点预留 20% 内存缓冲
  - 大内存任务调度到专用 Runner 组
  - 监控 OOM 事件频率 → 自动触发扩容
```

### 场景：部署到错误区域

```
触发：运维误操作，将生产部署推送到错误的 AWS 区域
      → us-east-1 的服务被部署到 eu-west-1
      → eu-west-1 无对应数据库/缓存 → 服务启动失败
      → us-east-1 旧版本继续运行（未更新）→ 功能不一致
检测：
  1. 部署后健康检查失败（服务连接不到区域本地数据库）
  2. 区域资源标签与预期不符 → 配置校验告警
  3. EKS 集群名称与部署目标不匹配
处理：
  1. 立即：停止错误区域的部署，标记为异常
  2. 回滚：删除错误区域的新部署资源
  3. 重新部署：指定正确区域重新触发部署流水线
  4. 验证：确认正确区域服务版本已更新
  5. 审计：记录误操作日志，更新权限控制
预防：
  - 部署流水线强制区域校验（目标区域必须在团队白名单内）
  - 生产环境部署需双人确认目标集群/区域
  - IaC 模板固定区域参数，不允许运行时覆盖
  - CI/CD 平台增加区域与环境的交叉校验规则
  - GitOps 模式：部署配置存储在 Git → 变更可审计、可回滚
```

## 制品管理完整实现

```python
class ArtifactManager:
    """制品管理：构建产物存储 + 版本管理 + 晋升"""

    def store_artifact(self, build_id, artifact_type, content_hash, size_bytes):
        """存储构建产物"""
        artifact_id = str(uuid4())
        # 1. 去重检查（相同 hash 已存在 → 不重复存储）
        existing = self.db.query_one(
            "SELECT * FROM artifacts WHERE content_hash = %s", content_hash)
        if existing:
            return {"artifact_id": existing["id"], "deduplicated": True}

        # 2. 存储到制品仓库
        storage_path = f"artifacts/{build_id}/{artifact_type}/{content_hash[:8]}"
        self.object_storage.upload(storage_path, content_hash)

        # 3. 记录元数据
        self.db.insert("artifacts", {
            "id": artifact_id, "build_id": build_id,
            "type": artifact_type,  # docker_image / jar / npm_package
            "content_hash": content_hash,
            "size_bytes": size_bytes,
            "storage_path": storage_path,
            "status": "stable",  # stable → promoted → archived
            "created_at": now()
        })

        return {"artifact_id": artifact_id, "deduplicated": False}

    def promote_artifact(self, artifact_id, from_env, to_env):
        """制品晋升：dev → staging → production"""
        artifact = self.db.get_artifact(artifact_id)

        # 1. 验证晋升条件
        promotion_rules = {
            "dev→staging": {"requires": ["unit_test_passed", "security_scan_passed"]},
            "staging→production": {"requires": ["integration_test_passed", "performance_test_passed",
                                                 "manual_approval"]},
        }
        rule_key = f"{from_env}→{to_env}"
        checks = promotion_rules.get(rule_key, {}).get("requires", [])

        for check in checks:
            if check == "manual_approval":
                approval = self.db.query_one(
                    "SELECT * FROM artifact_approvals "
                    "WHERE artifact_id = %s AND env = %s AND approved = 1",
                    artifact_id, to_env)
                if not approval:
                    return {"status": "blocked", "reason": f"需要 {check}"}
            else:
                status = self.db.query_one(
                    "SELECT status FROM artifact_checks "
                    "WHERE artifact_id = %s AND check_name = %s",
                    artifact_id, check)
                if not status or status["status"] != "passed":
                    return {"status": "blocked", "reason": f"{check} 未通过"}

        # 2. 执行晋升
        self.db.insert("artifact_promotions", {
            "artifact_id": artifact_id,
            "from_env": from_env, "to_env": to_env,
            "promoted_at": now(), "promoted_by": current_user()
        })

        return {"status": "promoted", "artifact_id": artifact_id}
```

## 环境管理

```python
class EnvironmentManager:
    """环境管理：dev/staging/production 环境生命周期"""

    def provision_environment(self, env_name, env_type, config):
        """创建环境"""
        env_id = str(uuid4())
        # 1. 基础设施创建（Terraform/Pulumi）
        infra_result = self.iac_client.apply({
            "env_name": env_name,
            "vpc_cidr": config["vpc_cidr"],
            "cluster_size": config.get("cluster_size", 3),
            "database_instance": config.get("db_instance", "db.t3.medium"),
        })

        # 2. 配置注入
        for key, value in config.get("env_vars", {}).items():
            self.vault_client.set_secret(f"{env_name}/{key}", value)

        # 3. DNS 配置
        self.dns_client.create_record(
            f"{env_name}.internal", infra_result["load_balancer_ip"])

        self.db.insert("environments", {
            "env_id": env_id, "name": env_name,
            "type": env_type,  # dev / staging / production
            "status": "active",
            "infra_config": json.dumps(infra_result),
            "created_at": now()
        })

        return {"env_id": env_id, "endpoint": f"{env_name}.internal"}

    def destroy_environment(self, env_id):
        """销毁环境（仅允许非生产环境）"""
        env = self.db.get_environment(env_id)
        if env["type"] == "production":
            raise OperationNotAllowedError("不允许销毁生产环境")

        # 1. 销毁基础设施
        self.iac_client.destroy(env["name"])
        # 2. 清理密钥
        self.vault_client.delete_all(f"{env['name']}/")
        # 3. 清理 DNS
        self.dns_client.delete_record(f"{env['name']}.internal")
        # 4. 标记环境为已销毁
        self.db.update("environments",
            {"status": "destroyed", "destroyed_at": now()},
            {"env_id": env_id})
```

## 异常场景补充

### 场景：制品晋升被阻塞

```
触发：安全扫描发现漏洞 → 制品无法晋升到生产环境
检测：
  1. 安全扫描结果为 CRITICAL → 自动阻断晋升
  2. 阻断时间 > 24 小时 → 严重告警
处理：
  1. 修复漏洞后重新构建
  2. 紧急发布：安全团队审批例外 + 事后修复
  3. 审批例外需要 VP 级别授权
预防：安全扫描左移（PR 阶段） + 漏洞分级处理
```

### 场景：环境配置漂移

```
触发：手动修改生产环境配置 → 与 IaC 定义不一致 → 漂移
检测：
  1. 定期 drift detection（每小时比对）
  2. 发现漂移 → 告警
处理：
  1. 自动修正：重新 apply IaC 配置
  2. 或人工审查：确认手动修改是否必要
  3. 必要 → 更新 IaC 代码（纳管）
预防：禁止手动修改 + drift detection + 自动修正
```

## 部署验证套件完整实现

```python
class DeploymentVerificationSuite:
    """部署验证：冒烟测试 + 金丝雀分析 + 渐进流量切换"""

    def verify_deployment(self, env, service_name, version):
        """验证部署结果"""
        results = []

        # 1. 健康检查
        health = self.http_client.get(
            f"https://{env}.internal/{service_name}/health")
        results.append({
            "name": "health_check", "passed": health.status_code == 200,
            "detail": health.json() if health.status_code != 200 else "OK"
        })

        # 2. 关键 API 冒烟测试
        smoke_tests = self._get_smoke_tests(service_name)
        for test in smoke_tests:
            response = self.http_client.request(
                method=test["method"],
                url=f"https://{env}.internal{test['path']}",
                json=test.get("body"))
            passed = response.status_code == test["expected_status"]
            results.append({
                "name": f"smoke_{test['name']}", "passed": passed,
                "detail": f"expected={test['expected_status']} got={response.status_code}"
            })

        # 3. 数据库连接性
        db_check = self._check_db_connectivity(env)
        results.append({"name": "db_connectivity", "passed": db_check})

        # 4. 外部依赖可达性
        deps = self._get_service_dependencies(service_name)
        for dep in deps:
            reachable = self._check_dependency(env, dep)
            results.append({
                "name": f"dep_{dep}", "passed": reachable
            })

        all_passed = all(r["passed"] for r in results)
        return {"service": service_name, "version": version,
                "passed": all_passed, "results": results}

    def canary_analysis(self, service_name, baseline_version, canary_version,
                        duration_minutes=10):
        """金丝雀分析：对比新旧版本指标"""
        baseline_metrics = self._collect_metrics(
            service_name, baseline_version, duration_minutes)
        canary_metrics = self._collect_metrics(
            service_name, canary_version, duration_minutes)

        # Mann-Whitney U 检验（错误率差异）
        baseline_errors = baseline_metrics["error_rates"]
        canary_errors = canary_metrics["error_rates"]
        u_stat, p_value = self._mann_whitney_u_test(baseline_errors, canary_errors)

        # 判定
        error_rate_increase = (
            (canary_metrics["avg_error_rate"] - baseline_metrics["avg_error_rate"])
            / max(baseline_metrics["avg_error_rate"], 0.001) * 100
        )

        if p_value < 0.05 and error_rate_increase > 10:
            decision = "rollback"
        elif error_rate_increase > 50:
            decision = "rollback"
        else:
            decision = "promote"

        return {
            "baseline_avg_error_rate": baseline_metrics["avg_error_rate"],
            "canary_avg_error_rate": canary_metrics["avg_error_rate"],
            "error_rate_increase_pct": round(error_rate_increase, 1),
            "p_value": round(p_value, 4),
            "decision": decision
        }

    def progressive_traffic_shift(self, service_name, canary_version, stages):
        """渐进流量切换：5%→25%→50%→100%"""
        for stage in stages:
            # 切换流量比例
            self.load_balancer.set_weight(
                service_name, canary_version, stage["percentage"])

            # 等待观察期
            time.sleep(stage["hold_seconds"])

            # 检查指标
            analysis = self.canary_analysis(
                service_name, "baseline", canary_version,
                duration_minutes=stage["hold_seconds"] // 60)

            if analysis["decision"] == "rollback":
                self.load_balancer.set_weight(
                    service_name, canary_version, 0)
                return {"status": "rolled_back", "stage": stage,
                        "reason": analysis}

        return {"status": "fully_promoted"}
```

## Secret 管理生命周期

```python
class SecretLifecycleManager:
    """Secret 管理：轮换 + 零停机 + 泄露检测"""

    ROTATION_SCHEDULE = {
        "api_key": 90,      # 90 天轮换
        "db_password": 30,  # 30 天轮换
        "certificate": 365, # 365 天轮换
        "oauth_token": 1,   # 1 天轮换
    }

    def schedule_rotation(self, secret_type, service_name):
        """安排 Secret 轮换"""
        interval_days = self.ROTATION_SCHEDULE.get(secret_type, 90)
        next_rotation = now() + timedelta(days=interval_days)

        self.db.insert("secret_rotations", {
            "secret_type": secret_type,
            "service_name": service_name,
            "rotation_interval_days": interval_days,
            "next_rotation_at": next_rotation,
            "status": "scheduled"
        })

    def rotate_secret(self, secret_type, service_name):
        """零停机轮换 Secret"""
        # 1. 生成新 Secret
        new_secret = self._generate_secret(secret_type)

        # 2. 写入 Vault（新旧同时有效）
        old_path = f"secret/{service_name}/{secret_type}"
        new_path = f"secret/{service_name}/{secret_type}_new"
        self.vault_client.write(new_path, value=new_secret)

        # 3. 通知服务使用新 Secret
        self.service_mesh.inject_secret(service_name, new_path)

        # 4. 等待服务重新加载
        self._wait_for_secret_reload(service_name, timeout=120)

        # 5. 验证新 Secret 生效
        if not self._verify_new_secret(service_name, new_secret):
            self.alert(f"服务 {service_name} 新 Secret 未生效")
            return {"status": "failed"}

        # 6. 删除旧 Secret
        self.vault_client.delete(old_path)
        self.vault_client.move(new_path, old_path)

        return {"status": "rotated", "service": service_name}

    def scan_for_leaks(self, repo_path):
        """扫描代码库中的 Secret 泄露"""
        patterns = [
            r'(?i)password\s*=\s*["\'][^"\']+["\']',
            r'(?i)api_key\s*=\s*["\'][^"\']+["\']',
            r'(?i)secret\s*=\s*["\'][^"\']+["\']',
            r'-----BEGIN (RSA |EC )?PRIVATE KEY-----',
        ]
        leaks = []
        for root, dirs, files in os.walk(repo_path):
            for file in files:
                if file.endswith(('.py', '.js', '.yaml', '.env', '.json')):
                    filepath = os.path.join(root, file)
                    with open(filepath) as f:
                        for i, line in enumerate(f, 1):
                            for pattern in patterns:
                                if re.search(pattern, line):
                                    leaks.append({
                                        "file": filepath, "line": i,
                                        "pattern": pattern
                                    })
        return leaks
```

## 异常场景补充

### 场景：金丝雀指标异常误报

```
触发：金丝雀部署后错误率短暂升高 → 自动回滚 → 但实际是正常波动
检测：
  1. 回滚后基线版本错误率同样升高 → 环境问题，非版本问题
  2. 错误率升高 < 30 秒 → 可能是瞬时波动
处理：
  1. 增加金丝雀观察窗口（10 分钟 → 30 分钟）
  2. 使用统计检验替代简单阈值
  3. 重新部署金丝雀验证
预防：统计检验 + 充足观察窗口 + 排除环境噪声
```

### 场景：Secret 轮换导致服务认证失败

```
触发：数据库密码轮换 → 服务使用新密码连接失败 → 服务不可用
检测：
  1. 服务健康检查失败 → 告警
  2. 数据库连接错误日志 → 认证失败
处理：
  1. 临时恢复旧密码（双密码模式期间旧密码仍有效）
  2. 检查新密码是否正确注入
  3. 修复后重新使用新密码
预防：双密码过渡期 + 轮换前预验证 + 回滚机制
```

## CI/CD 分析看板完整实现

```python
class CIAnalyticsDashboard:
    """CI/CD 分析看板：DORA 指标 + 构建统计 + Flaky 测试"""

    def get_dora_metrics(self, team_id, period="monthly"):
        """获取 DORA 四大指标"""
        days = 30 if period == "monthly" else 7
        return {
            "deployment_frequency": self._deployment_frequency(team_id, days),
            "lead_time_for_changes": self._lead_time(team_id, days),
            "change_failure_rate": self._change_failure_rate(team_id, days),
            "mttr": self._mttr(team_id, days),
        }

    def _deployment_frequency(self, team_id, days):
        """部署频率"""
        deployments = self.db.query(
            "SELECT DATE(deployed_at) as date, COUNT(*) as count "
            "FROM deployments WHERE team_id = %s "
            "AND deployed_at > NOW() - INTERVAL %s DAY "
            "AND env = 'production' GROUP BY DATE(deployed_at)",
            team_id, days)
        total = sum(d["count"] for d in deployments)
        avg_per_day = total / days
        if avg_per_day >= 1:
            level = "elite"
        elif avg_per_day >= 1/7:
            level = "high"
        elif avg_per_day >= 1/30:
            level = "medium"
        else:
            level = "low"
        return {"total": total, "avg_per_day": round(avg_per_day, 2), "level": level}

    def _lead_time(self, team_id, days):
        """变更前置时间（commit → production）"""
        lead_times = self.db.query(
            "SELECT TIMESTAMPDIFF(HOUR, c.committed_at, d.deployed_at) as hours "
            "FROM commits c JOIN deployments d ON c.deployment_id = d.id "
            "WHERE d.team_id = %s AND d.deployed_at > NOW() - INTERVAL %s DAY "
            "AND d.env = 'production'", team_id, days)
        if not lead_times:
            return {"avg_hours": None, "level": "unknown"}
        avg = statistics.mean(lt["hours"] for lt in lead_times)
        if avg <= 24:
            level = "elite"
        elif avg <= 168:
            level = "high"
        elif avg <= 730:
            level = "medium"
        else:
            level = "low"
        return {"avg_hours": round(avg, 1), "level": level}

    def _change_failure_rate(self, team_id, days):
        """变更失败率"""
        total = self.db.count("deployments", team_id=team_id,
            env="production", deployed_at__gte=now()-timedelta(days=days))
        failed = self.db.count("deployment_incidents",
            team_id=team_id, created_at__gte=now()-timedelta(days=days))
        rate = failed / max(total, 1)
        if rate <= 0.05:
            level = "elite"
        elif rate <= 0.10:
            level = "high"
        elif rate <= 0.15:
            level = "medium"
        else:
            level = "low"
        return {"rate": round(rate, 3), "level": level}

    def _mttr(self, team_id, days):
        """平均恢复时间"""
        incidents = self.db.query(
            "SELECT TIMESTAMPDIFF(MINUTE, created_at, resolved_at) as minutes "
            "FROM deployment_incidents WHERE team_id = %s "
            "AND resolved_at IS NOT NULL "
            "AND created_at > NOW() - INTERVAL %s DAY",
            team_id, days)
        if not incidents:
            return {"avg_minutes": None, "level": "unknown"}
        avg = statistics.mean(i["minutes"] for i in incidents)
        if avg <= 60:
            level = "elite"
        elif avg <= 1440:
            level = "high"
        elif avg <= 4320:
            level = "medium"
        else:
            level = "low"
        return {"avg_minutes": round(avg, 0), "level": level}

    def identify_flaky_tests(self, project_id):
        """识别 Flaky 测试"""
        # 最近 100 次构建中，同一测试在同一代码上有时通过有时失败
        results = self.db.query(
            "SELECT test_name, commit_hash, "
            "SUM(CASE WHEN status = 'passed' THEN 1 ELSE 0 END) as passed, "
            "SUM(CASE WHEN status = 'failed' THEN 1 ELSE 0 END) as failed, "
            "COUNT(*) as total "
            "FROM test_results WHERE project_id = %s "
            "AND created_at > NOW() - INTERVAL 30 DAY "
            "GROUP BY test_name, commit_hash "
            "HAVING passed > 0 AND failed > 0", project_id)

        flaky = []
        for r in results:
            flakiness = min(r["passed"], r["failed"]) / r["total"]
            flaky.append({
                "test_name": r["test_name"],
                "flakiness": round(flakiness, 3),
                "passed": r["passed"], "failed": r["failed"],
                "action": "quarantine" if flakiness > 0.3 else "monitor"
            })

        return sorted(flaky, key=lambda x: x["flakiness"], reverse=True)
```

## Compliance-as-Code

```python
class ComplianceAsCode:
    """Compliance-as-Code：OPA 策略 + CI 集成"""

    POLICIES = {
        "no_hardcoded_secrets": {
            "description": "禁止硬编码密钥",
            "severity": "critical",
            "rego": """
                package ci.compliance
                deny[msg] {
                    input.file.content =~ "password\\s*=\\s*['\"][^'\"]+['\"]"
                    msg := sprintf("硬编码密钥发现于 %s", [input.file.path])
                }
            """
        },
        "dependency_vulnerability_scan": {
            "description": "依赖漏洞扫描",
            "severity": "high",
        },
        "license_compliance": {
            "description": "开源许可证合规",
            "severity": "medium",
            "blocked_licenses": ["GPL-2.0", "GPL-3.0", "AGPL-3.0"],
        },
        "no_debug_in_production": {
            "description": "生产环境禁止调试模式",
            "severity": "high",
        },
    }

    def evaluate(self, project_id, context):
        """评估合规性"""
        violations = []

        for policy_name, policy in self.POLICIES.items():
            if policy_name == "no_hardcoded_secrets":
                result = self._check_secrets(context["files"])
            elif policy_name == "dependency_vulnerability_scan":
                result = self._check_vulnerabilities(context["dependencies"])
            elif policy_name == "license_compliance":
                result = self._check_licenses(context["dependencies"])
            elif policy_name == "no_debug_in_production":
                result = self._check_debug_mode(context["config"])

            if result["violations"]:
                violations.append({
                    "policy": policy_name,
                    "severity": policy["severity"],
                    "violations": result["violations"]
                })

        # 严重违规 → 阻止部署
        blocking = any(v["severity"] == "critical" for v in violations)
        return {"compliant": len(violations) == 0, "blocking": blocking,
                "violations": violations}

    def _check_secrets(self, files):
        """检查硬编码密钥"""
        violations = []
        patterns = [
            r'password\s*=\s*["\'][^"\']+["\']',
            r'api_key\s*=\s*["\'][^"\']+["\']',
            r'-----BEGIN.*PRIVATE KEY-----',
        ]
        for f in files:
            for pattern in patterns:
                if re.search(pattern, f.get("content", "")):
                    violations.append({"file": f["path"], "pattern": pattern})
        return {"violations": violations}
```

## 异常场景补充

### 场景：Flaky 测试隔离掩盖真实故障

```
触发：被隔离的 flaky 测试实际是真实 bug → 隔离后不再关注 → 线上故障
检测：
  1. 生产环境出现被隔离测试覆盖的 bug → 遗漏
  2. 隔离测试列表过长 → 需要审查
处理：
  1. 定期审查隔离测试列表（每周）
  2. 每个隔离测试必须关联修复 issue
  3. 超过 30 天未修复 → 重新启用（阻止构建）
预防：隔离测试定期审查 + 关联修复 issue + 超期重新启用
```

### 场景：OPA 策略过严阻止合法部署

```
触发：合规策略误判 → 阻止紧急修复部署 → 影响线上问题恢复
检测：
  1. 合规阻止 + 部署延迟 > 30 分钟 → 影响恢复
  2. 申诉率 > 20% → 策略过严
处理：
  1. 紧急豁免：值班工程师可临时绕过（需事后审计）
  2. 审查误判策略 → 调整规则
  3. 紧急部署后 24 小时内补合规检查
预防：紧急豁免机制 + 策略定期审查 + 误判率监控
```

## 流水线模板市场完整实现

```python
class PipelineTemplateMarketplace:
    """流水线模板市场：注册 + 版本 + 组合"""

    TEMPLATES = {
        "java-maven": {
            "stages": ["build", "test", "sonar", "docker-build", "deploy"],
            "required_params": ["maven_version", "java_version", "docker_registry"],
            "default_params": {"maven_version": "3.9", "java_version": "17"},
        },
        "node-npm": {
            "stages": ["install", "lint", "test", "build", "deploy"],
            "required_params": ["node_version", "npm_registry"],
            "default_params": {"node_version": "20"},
        },
        "go-build": {
            "stages": ["fmt", "vet", "test", "build", "docker-push", "deploy"],
            "required_params": ["go_version"],
            "default_params": {"go_version": "1.21"},
        },
        "python-pip": {
            "stages": ["install", "lint", "test", "build-wheel", "deploy"],
            "required_params": ["python_version"],
            "default_params": {"python_version": "3.11"},
        },
    }

    def instantiate_template(self, template_name, project_id, params):
        """实例化流水线模板"""
        template = self.TEMPLATES.get(template_name)
        if not template:
            raise TemplateNotFoundError(f"模板 {template_name} 不存在")

        # 1. 参数校验
        merged_params = {**template["default_params"], **params}
        missing = [p for p in template["required_params"]
                   if p not in merged_params]
        if missing:
            raise MissingParamsError(f"缺少参数: {missing}")

        # 2. 生成流水线配置
        pipeline_config = {
            "project_id": project_id,
            "template": template_name,
            "params": merged_params,
            "stages": []
        }

        for stage_name in template["stages"]:
            stage = self._generate_stage(stage_name, merged_params)
            pipeline_config["stages"].append(stage)

        # 3. 创建流水线
        pipeline_id = self.ci_service.create_pipeline(pipeline_config)

        self.db.insert("pipeline_instances", {
            "pipeline_id": pipeline_id,
            "project_id": project_id,
            "template_name": template_name,
            "params": json.dumps(merged_params),
            "created_at": now()
        })

        return {"pipeline_id": pipeline_id, "template": template_name}

    def _generate_stage(self, stage_name, params):
        """生成阶段配置"""
        stage_generators = {
            "build": lambda p: {"image": f"maven:{p['maven_version']}",
                "script": "mvn clean package -DskipTests"},
            "test": lambda p: {"image": f"maven:{p['maven_version']}",
                "script": "mvn test", "coverage": True},
            "install": lambda p: {"image": f"node:{p['node_version']}",
                "script": "npm ci"},
            "lint": lambda p: {"image": f"node:{p['node_version']}",
                "script": "npm run lint"},
            "docker-build": lambda p: {
                "script": f"docker build -t {p['docker_registry']}/${{CI_PROJECT}}:${{CI_COMMIT_SHA}} ."},
            "deploy": lambda p: {
                "script": "kubectl apply -f k8s/",
                "environment": "staging", "manual_trigger": True},
        }
        generator = stage_generators.get(stage_name,
            lambda p: {"script": f"echo 'stage: {stage_name}'"})
        return {"name": stage_name, **generator(params)}
```

## 异常场景补充

### 场景：流水线模板参数不兼容

```
触发：模板使用 Maven 3.9 但项目需要 Maven 3.6 → 构建失败
检测：
  1. 构建日志中出现版本不兼容错误 → 参数问题
  2. 同模板其他项目正常 → 特定项目参数问题
处理：
  1. 项目级参数覆盖（project_id 级别）
  2. 模板支持版本范围声明（maven_version: "3.6-3.9"）
  3. 自动检测项目版本需求
预防：参数范围声明 + 项目级覆盖 + 版本兼容性矩阵
```

### 场景：合规策略阻止紧急修复

```
触发：生产环境紧急 bug → 需要绕过合规检查 → OPA 策略阻止部署
检测：
  1. 合规检查阻止 + 告警标记为"紧急" → 冲突
  2. MTTR 超过 SLA → 合规流程影响恢复
处理：
  1. 紧急豁免：值班工程师审批 + SRE 确认
  2. 豁免自动创建事后审计工单
  3. 24 小时内补全合规检查
预防：紧急豁免机制 + 审计追踪 + 事后补检
```

## 环境晋升流水线完整实现

```python
class EnvironmentPromotionService:
    """环境晋升：dev→staging→production 自动门控"""

    PROMOTION_GATES = {
        "dev_to_staging": ["unit_tests", "code_coverage", "lint"],
        "staging_to_production": ["integration_tests", "security_scan", "performance_test", "canary_check"],
    }

    def promote(self, artifact_id, from_env, to_env):
        """晋升构建产物到下一环境"""
        gate_key = f"{from_env}_to_{to_env}"
        gates = self.PROMOTION_GATES.get(gate_key, [])

        # 1. 运行所有门控检查
        results = []
        for gate in gates:
            result = self._run_gate(gate, artifact_id)
            results.append(result)
            if not result["passed"]:
                # 门控失败 → 停止晋升
                self.db.insert("promotion_failures", {
                    "artifact_id": artifact_id,
                    "from_env": from_env, "to_env": to_env,
                    "failed_gate": gate, "details": json.dumps(result),
                    "timestamp": now()
                })
                return {"status": "blocked", "failed_gate": gate, "details": result}

        # 2. 所有门控通过 → 配置差异检查
        config_diff = self._compare_configs(from_env, to_env)
        if config_diff["breaking_changes"]:
            return {"status": "config_mismatch", "diff": config_diff}

        # 3. 生产环境需要人工审批
        if to_env == "production":
            approval = self._request_approval(artifact_id, results, config_diff)
            if approval["status"] != "approved":
                return {"status": "pending_approval", "approval_id": approval["id"]}

        # 4. 执行部署
        self.deployment_service.deploy(artifact_id, to_env)

        # 5. 记录审计
        self.db.insert("promotion_audit", {
            "artifact_id": artifact_id,
            "from_env": from_env, "to_env": to_env,
            "gate_results": json.dumps(results),
            "config_diff": json.dumps(config_diff),
            "promoted_at": now(),
            "promoted_by": "auto" if to_env != "production" else approval["approved_by"]
        })

        return {"status": "promoted", "to_env": to_env}

    def _run_gate(self, gate, artifact_id):
        """运行门控检查"""
        if gate == "unit_tests":
            result = self.ci_service.get_test_results(artifact_id)
            return {"passed": result["passed"], "details": f"{result['passed']}/{result['total']} tests"}
        elif gate == "security_scan":
            vulns = self.scanner.get_vulnerabilities(artifact_id)
            critical = [v for v in vulns if v["severity"] in ["critical", "high"]]
            return {"passed": len(critical) == 0, "details": f"{len(critical)} critical/high vulnerabilities"}
        elif gate == "performance_test":
            perf = self.perf_service.get_results(artifact_id)
            return {"passed": perf["p99_latency_ms"] < 500, "details": f"P99={perf['p99_latency_ms']}ms"}
        return {"passed": True, "details": "skipped"}

    def _compare_configs(self, from_env, to_env):
        """比较环境配置差异"""
        from_config = self.config_service.get_all(from_env)
        to_config = self.config_service.get_all(to_env)

        diff = {"added": [], "removed": [], "changed": [], "breaking_changes": []}
        for key in set(list(from_config.keys()) + list(to_config.keys())):
            if key not in to_config:
                diff["removed"].append(key)
                if any(k in key for k in ["DB_HOST", "API_KEY", "ENDPOINT"]):
                    diff["breaking_changes"].append(f"Missing: {key}")
            elif key not in from_config:
                diff["added"].append(key)
            elif from_config[key] != to_config[key]:
                diff["changed"].append({"key": key, "from": from_config[key], "to": to_config[key]})

        return diff
```

## 制品管理与漏洞扫描

```python
class ArtifactSecurityScanner:
    """制品安全扫描：CVE + SBOM + 许可证合规"""

    def scan_artifact(self, artifact_id):
        """扫描构建产物"""
        artifact = self.db.get_artifact(artifact_id)

        # 1. 容器镜像 CVE 扫描
        if artifact["type"] == "docker_image":
            vulns = self.trivy.scan(artifact["image_url"])
            critical = [v for v in vulns if v["severity"] == "critical"]
            high = [v for v in vulns if v["severity"] == "high"]

            # 阻断策略
            if critical:
                self._block_deployment(artifact_id, f"{len(critical)} critical CVEs")
            elif high:
                self._warn_deployment(artifact_id, f"{len(high)} high CVEs")

        # 2. 生成 SBOM
        sbom = self.generate_sbom(artifact)
        self.db.update("artifacts",
            {"sbom": json.dumps(sbom)}, {"id": artifact_id})

        # 3. 许可证合规
        licenses = self._scan_licenses(sbom)
        blocked = ["GPL-2.0", "GPL-3.0", "AGPL-3.0"]
        violations = [l for l in licenses if l["license"] in blocked]
        if violations:
            self._block_deployment(artifact_id,
                f"许可证违规: {[v['package'] for v in violations]}")

        return {"vulnerabilities": len(vulns) if artifact["type"] == "docker_image" else 0,
                "critical": len(critical) if artifact["type"] == "docker_image" else 0,
                "license_violations": len(violations),
                "sbom_packages": len(sbom.get("packages", []))}

    def generate_sbom(self, artifact):
        """生成软件物料清单"""
        packages = []
        if artifact["type"] == "docker_image":
            # 解析 Dockerfile 中的依赖
            layers = self.docker_client.get_image_layers(artifact["image_url"])
            for layer in layers:
                for pkg in layer.get("packages", []):
                    packages.append({
                        "name": pkg["name"], "version": pkg["version"],
                        "license": pkg.get("license", "unknown"),
                        "source": pkg.get("source_url")
                    })
        return {"artifact_id": artifact["id"], "packages": packages,
                "generated_at": now().isoformat()}

    def check_patch_sla(self):
        """检查漏洞修补 SLA"""
        # Critical: 24 小时, High: 7 天
        overdue = self.db.query(
            "SELECT * FROM vulnerability_findings "
            "WHERE severity = 'critical' AND discovered_at < NOW() - INTERVAL 1 DAY "
            "AND patched_at IS NULL "
            "UNION ALL "
            "SELECT * FROM vulnerability_findings "
            "WHERE severity = 'high' AND discovered_at < NOW() - INTERVAL 7 DAY "
            "AND patched_at IS NULL")
        return {"overdue_patches": len(overdue), "items": overdue}
```

## 异常场景补充

### 场景：晋升门控误报阻断发布

```
触发：安全扫描误报 CVE → 阻断晋升 → 延误上线
检测：
  1. 开发团队申诉漏洞为误报 → 可能误判
  2. 同一 CVE 在其他项目中也被标记为误报 → 系统性问题
处理：
  1. 安全团队 4 小时内验证误报
  2. 确认误报 → 加入白名单 → 允许晋升
  3. 紧急情况 → 临时绕过（需安全团队审批）
预防：误报白名单 + 快速验证 SLA + 紧急绕过机制
```

### 场景：已部署制品发现严重 CVE

```
触发：生产环境运行的镜像被发现有 critical CVE → 需要紧急修复
检测：
  1. 每日定时扫描生产镜像 → 发现新 CVE
  2. CVE 评分 critical → 紧急告警
处理：
  1. 评估影响范围（哪些服务使用该镜像）
  2. 有补丁版本 → 紧急构建+晋升+部署
  3. 无补丁版本 → 应用缓解措施（WAF 规则、网络隔离）
  4. 24 小时内完成修复
预防：每日生产镜像扫描 + 紧急修复流程 + 缓解措施库
```

## 环境晋升深度实现：门禁引擎与审批编排

```python
class PromotionGateEngine:
    """晋升门禁引擎：可插拔门禁策略 + 自动重试 + 回滚编排"""

    GATE_DEFINITIONS = {
        "unit_tests": {
            "executor": "ci_runner",
            "timeout_seconds": 900,
            "retry_on_failure": True,
            "max_retries": 1,
            "retry_delay_seconds": 30,
            "blocking": True,
            "threshold_evaluator": "test_pass_rate",
            "params": {"min_pass_rate": 0.95, "min_coverage": 0.80},
        },
        "integration_tests": {
            "executor": "ci_runner",
            "timeout_seconds": 1800,
            "retry_on_failure": False,
            "blocking": True,
            "threshold_evaluator": "test_pass_rate",
            "params": {"min_pass_rate": 1.0},
            "requires": ["staging_db", "staging_redis", "staging_mq"],
        },
        "security_scan": {
            "executor": "trivy_scanner",
            "timeout_seconds": 300,
            "retry_on_failure": True,
            "max_retries": 2,
            "blocking": True,
            "threshold_evaluator": "vulnerability_count",
            "params": {"max_critical": 0, "max_high": 0},
        },
        "canary_deploy": {
            "executor": "k8s_canary_controller",
            "timeout_seconds": 900,
            "retry_on_failure": False,
            "blocking": True,
            "params": {
                "canary_weight_pct": 5,
                "observation_window_minutes": 10,
                "metrics": {
                    "error_rate_delta_max": 0.01,
                    "latency_p99_delta_ms_max": 50,
                    "cpu_usage_max": 0.80,
                    "memory_usage_max": 0.85,
                },
                "promote_weight_steps": [5, 25, 50, 100],
                "step_observation_minutes": 5,
            },
        },
        "smoke_tests": {
            "executor": "api_test_runner",
            "timeout_seconds": 300,
            "retry_on_failure": True,
            "max_retries": 1,
            "blocking": True,
            "threshold_evaluator": "test_pass_rate",
            "params": {"min_pass_rate": 1.0},
        },
    }

    def execute_gates(self, promotion_id, target_env, artifact_id):
        """执行目标环境的所有门禁检查"""
        gates = self._get_gates_for_env(target_env)
        results = []

        for gate_name in gates:
            gate_def = self.GATE_DEFINITIONS[gate_name]

            # 预置资源（如集成测试需要 staging 数据库）
            if "requires" in gate_def:
                self._provision_gate_resources(gate_def["requires"], artifact_id)

            result = self._execute_single_gate(gate_name, gate_def, artifact_id)
            results.append(result)

            if not result["passed"] and gate_def["blocking"]:
                # 阻塞性门禁失败 → 触发回滚
                self._trigger_rollback(promotion_id, target_env, result)
                return {"status": "gate_failed", "results": results,
                        "failed_gate": gate_name}

        return {"status": "all_gates_passed", "results": results}

    def _execute_single_gate(self, gate_name, gate_def, artifact_id):
        """执行单个门禁（含自动重试）"""
        attempt = 0
        max_attempts = 1 + (gate_def.get("max_retries", 0)
                            if gate_def.get("retry_on_failure") else 0)

        while attempt < max_attempts:
            attempt += 1
            execution = self.executor_registry[gate_def["executor"]].run(
                gate=gate_name,
                artifact_id=artifact_id,
                params=gate_def["params"],
                timeout=gate_def["timeout_seconds"],
            )

            passed = execution["exit_code"] == 0

            # 阈值评估
            if passed and "threshold_evaluator" in gate_def:
                evaluator = self.threshold_evaluators[gate_def["threshold_evaluator"]]
                passed = evaluator.evaluate(execution["output"], gate_def["params"])

            if passed:
                return {
                    "gate": gate_name, "passed": True,
                    "attempt": attempt, "output": execution["output"],
                    "duration_seconds": execution["duration"],
                }

            # 非最后一次尝试 → 等待重试
            if attempt < max_attempts:
                self.logger.info(
                    f"门禁 {gate_name} 第 {attempt} 次尝试失败，"
                    f"{gate_def.get('retry_delay_seconds', 30)} 秒后重试")
                time.sleep(gate_def.get("retry_delay_seconds", 30))

        return {
            "gate": gate_name, "passed": False,
            "attempt": attempt, "output": execution["output"],
            "duration_seconds": execution["duration"],
            "is_flaky_suspect": attempt > 1,  # 重试后才失败 → 可能 flaky
        }

    def _trigger_rollback(self, promotion_id, target_env, failed_result):
        """门禁失败触发回滚"""
        # 获取当前环境的部署版本
        current = self.db.query_one(
            "SELECT * FROM deployments WHERE env = %s "
            "AND status = 'active' ORDER BY deployed_at DESC LIMIT 1",
            target_env)

        if not current:
            return  # 没有活跃部署，无需回滚

        # 执行 Kubernetes 回滚
        rollback_result = self.k8s.rollback_deployment(
            namespace=self._get_namespace(target_env),
            deployment=current["deployment_name"],
            revision=current["previous_revision"],
        )

        # 记录回滚
        self.db.insert("promotion_rollbacks", {
            "promotion_id": promotion_id,
            "target_env": target_env,
            "failed_gate": failed_result["gate"],
            "rolled_back_to_revision": current["previous_revision"],
            "rollback_status": rollback_result["status"],
            "rolled_back_at": now(),
        })

        # 告警
        self.alerting.send(
            severity="high",
            title=f"晋升失败已回滚: {target_env}",
            message=f"门禁 {failed_result['gate']} 未通过（尝试 {failed_result['attempt']} 次），"
                    f"已回滚到上一稳定版本",
            tags={"promotion_id": promotion_id, "env": target_env,
                  "failed_gate": failed_result["gate"]},
        )


class PromotionApprovalOrchestrator:
    """晋升审批编排：自动审批 + 人工审批 + 审批超时"""

    APPROVAL_RULES = {
        "dev": {"mode": "auto", "conditions": "all_gates_passed"},
        "staging": {"mode": "auto", "conditions": "all_gates_passed"},
        "production": {
            "mode": "manual",
            "min_approvers": 2,
            "required_roles": ["release_manager", "sre_oncall"],
            "timeout_minutes": 30,
            "timeout_action": "reject",  # 超时自动拒绝
            "emergency_bypass": {
                "allowed": True,
                "required_roles": ["vp_engineering", "cto"],
                "max_duration_hours": 24,
                "post_bypass_audit": True,
            },
        },
    }

    def request_approval(self, promotion_id, target_env, gate_results):
        """请求晋升审批"""
        rules = self.APPROVAL_RULES[target_env]

        if rules["mode"] == "auto":
            # 自动审批：检查条件
            if self._evaluate_conditions(rules["conditions"], gate_results):
                return {"approved": True, "approver": "system:auto",
                        "mode": "auto"}
            else:
                return {"approved": False, "reason": "条件不满足"}

        # 人工审批流程
        approval_id = generate_id()
        required_approvers = self._find_approvers(rules["required_roles"])

        self.db.insert("promotion_approvals", {
            "approval_id": approval_id,
            "promotion_id": promotion_id,
            "target_env": target_env,
            "required_approvers": json.dumps(required_approvers),
            "min_approvals": rules["min_approvers"],
            "received_approvals": 0,
            "status": "pending",
            "timeout_at": now() + timedelta(minutes=rules["timeout_minutes"]),
            "created_at": now(),
        })

        # 多通道通知审批人
        self._notify_approvers(required_approvers, {
            "approval_id": approval_id,
            "promotion_id": promotion_id,
            "target_env": target_env,
            "gate_results": gate_results,
            "timeout_minutes": rules["timeout_minutes"],
        })

        # 启动超时监控
        self._start_timeout_monitor(approval_id, rules["timeout_minutes"])

        return {"approved": False, "status": "pending_approval",
                "approval_id": approval_id}

    def process_approval_decision(self, approval_id, approver_id, decision, reason=""):
        """处理审批决定"""
        approval = self.db.get("promotion_approvals", approval_id)
        if approval["status"] != "pending":
            return {"error": "审批已关闭", "current_status": approval["status"]}

        # 记录审批决定
        self.db.insert("approval_decisions", {
            "approval_id": approval_id,
            "approver_id": approver_id,
            "decision": decision,  # approved / rejected
            "reason": reason,
            "decided_at": now(),
        })

        if decision == "rejected":
            self.db.update("promotion_approvals",
                {"status": "rejected", "rejected_by": approver_id},
                {"approval_id": approval_id})
            return {"approved": False, "reason": reason}

        # 累计审批数
        approvals = self.db.query(
            "SELECT COUNT(*) as cnt FROM approval_decisions "
            "WHERE approval_id = %s AND decision = 'approved'", approval_id)
        current_count = approvals[0]["cnt"]

        if current_count >= approval["min_approvals"]:
            self.db.update("promotion_approvals",
                {"status": "approved", "approved_at": now()},
                {"approval_id": approval_id})
            return {"approved": True}

        return {"approved": False, "status": "pending_approval",
                "current_approvals": current_count,
                "required_approvals": approval["min_approvals"]}

    def handle_emergency_bypass(self, approval_id, bypass_authorizer, justification):
        """紧急绕过审批"""
        rules = self.APPROVAL_RULES["production"]["emergency_bypass"]

        # 验证授权人角色
        if not self._has_any_role(bypass_authorizer, rules["required_roles"]):
            return {"error": "无权执行紧急绕过",
                    "required_roles": rules["required_roles"]}

        # 创建紧急绕过记录
        self.db.insert("emergency_bypasses", {
            "approval_id": approval_id,
            "authorizer_id": bypass_authorizer,
            "justification": justification,
            "expires_at": now() + timedelta(hours=rules["max_duration_hours"]),
            "audit_required": rules["post_bypass_audit"],
            "created_at": now(),
        })

        # 自动审批通过
        self.db.update("promotion_approvals",
            {"status": "emergency_bypassed",
             "bypass_authorizer": bypass_authorizer,
             "approved_at": now()},
            {"approval_id": approval_id})

        # 创建事后审计工单
        if rules["post_bypass_audit"]:
            self.ticket_system.create({
                "type": "emergency_bypass_audit",
                "approval_id": approval_id,
                "bypass_authorizer": bypass_authorizer,
                "justification": justification,
                "deadline": now() + timedelta(hours=24),
            })

        return {"approved": True, "mode": "emergency_bypass"}


class EnvironmentConfigDiffService:
    """环境配置差异对比服务"""

    SENSITIVE_KEY_PATTERNS = [
        r".*password.*", r".*secret.*", r".*api_key.*", r".*token.*",
        r".*private_key.*", r".*credential.*",
    ]

    INFRASTRUCTURE_KEY_PATTERNS = [
        r".*DB_HOST.*", r".*DATABASE_URL.*", r".*REDIS_URL.*",
        r".*KAFKA_BROKERS.*", r".*ENDPOINT.*", r".*SERVICE_URL.*",
    ]

    def compare_environments(self, source_env, target_env):
        """比较两个环境的配置差异"""
        source_config = self._load_env_config(source_env)
        target_config = self._load_env_config(target_env)

        diffs = []
        all_keys = sorted(set(list(source_config.keys()) + list(target_config.keys())))

        for key in all_keys:
            src_val = source_config.get(key)
            tgt_val = target_config.get(key)

            if src_val is None and tgt_val is not None:
                diff_type = "only_in_target"
                risk = self._classify_risk(key, diff_type)
            elif src_val is not None and tgt_val is None:
                diff_type = "only_in_source"
                risk = self._classify_risk(key, diff_type)
            elif src_val != tgt_val:
                diff_type = "value_mismatch"
                risk = self._classify_risk(key, diff_type)
            else:
                continue

            diffs.append({
                "key": key,
                "diff_type": diff_type,
                "source_value": self._mask_if_sensitive(key, src_val),
                "target_value": self._mask_if_sensitive(key, tgt_val),
                "risk_level": risk,
                "recommendation": self._generate_recommendation(key, diff_type, risk),
            })

        high_risk = [d for d in diffs if d["risk_level"] == "high"]
        return {
            "total_diffs": len(diffs),
            "high_risk": high_risk,
            "safe_to_promote": len(high_risk) == 0,
            "diffs": diffs,
        }

    def _classify_risk(self, key, diff_type):
        """分类配置差异风险"""
        import re
        # 基础设施配置缺失 → 高风险
        if diff_type == "only_in_source":
            if any(re.match(p, key, re.IGNORECASE) for p in self.INFRASTRUCTURE_KEY_PATTERNS):
                return "high"

        # 目标环境多出敏感配置 → 中风险
        if diff_type == "only_in_target":
            if any(re.match(p, key, re.IGNORECASE) for p in self.SENSITIVE_KEY_PATTERNS):
                return "medium"

        return "low"

    def _mask_if_sensitive(self, key, value):
        """脱敏敏感配置值"""
        if value is None:
            return None
        import re
        if any(re.match(p, key, re.IGNORECASE) for p in self.SENSITIVE_KEY_PATTERNS):
            return "***MASKED***"
        return value

    def _generate_recommendation(self, key, diff_type, risk):
        """生成配置差异修复建议"""
        if risk == "high" and diff_type == "only_in_source":
            return f"关键配置 {key} 在目标环境缺失，必须在晋升前添加"
        elif risk == "medium" and diff_type == "only_in_target":
            return f"目标环境多出敏感配置 {key}，确认是否需要"
        return f"配置 {key} 存在差异，建议确认目标环境值是否正确"
```

## 制品仓库深度实现：保留策略与安全扫描编排

```python
class ArtifactRetentionManager:
    """制品保留策略管理器"""

    def __init__(self):
        self.retention_policies = {
            "docker_image": {
                "keep_last_n": 10,
                "keep_tag_patterns": ["release-.*", "v\\d+\\.\\d+\\.\\d+"],
                "never_delete_deployed": True,
                "grace_period_days": 7,  # 新版本发布后保留旧版本 7 天
            },
            "npm_package": {
                "keep_last_n": 10,
                "keep_tag_patterns": ["latest", "next", "release-.*"],
                "never_delete_deployed": True,
                "grace_period_days": 3,
            },
            "jar_file": {
                "keep_last_n": 10,
                "keep_tag_patterns": ["release-.*"],
                "never_delete_deployed": True,
                "grace_period_days": 7,
            },
        }

    def execute_retention(self, artifact_type, dry_run=False):
        """执行制品保留策略"""
        policy = self.retention_policies[artifact_type]
        artifacts = self.db.query(
            "SELECT * FROM artifacts WHERE type = %s AND status = 'active' "
            "ORDER BY created_at DESC", artifact_type)

        to_keep = []
        to_delete = []

        for i, artifact in enumerate(artifacts):
            keep_reasons = []

            # 规则 1：保留最近 N 个版本
            if i < policy["keep_last_n"]:
                keep_reasons.append(f"最近 {policy['keep_last_n']} 个版本之一")

            # 规则 2：保留匹配 tag 模式的版本
            import re
            for pattern in policy["keep_tag_patterns"]:
                if re.match(pattern, artifact["version"]):
                    keep_reasons.append(f"匹配 release tag 模式: {pattern}")
                    break

            # 规则 3：正在部署中的制品不删除
            if policy["never_delete_deployed"]:
                deployed = self.db.query(
                    "SELECT COUNT(*) as cnt FROM deployments "
                    "WHERE artifact_id = %s AND status = 'active'",
                    artifact["artifact_id"])[0]["cnt"]
                if deployed > 0:
                    keep_reasons.append(f"当前有 {deployed} 个活跃部署使用")

            # 规则 4：宽限期内不删除
            if policy["grace_period_days"] > 0:
                age_days = (now() - artifact["created_at"]).days
                if age_days < policy["grace_period_days"]:
                    keep_reasons.append("宽限期内")

            if keep_reasons:
                to_keep.append({"artifact": artifact, "reasons": keep_reasons})
            else:
                to_delete.append(artifact)

        if dry_run:
            return {"dry_run": True, "to_keep": len(to_keep),
                    "to_delete": len(to_delete),
                    "freed_space_gb": sum(a["size_mb"] for a in to_delete) / 1024}

        # 执行删除
        for artifact in to_delete:
            self._delete_from_registry(artifact)
            self.db.update("artifacts",
                {"status": "retention_deleted", "deleted_at": now()},
                {"artifact_id": artifact["artifact_id"]})

        freed_gb = sum(a["size_mb"] for a in to_delete) / 1024
        return {"dry_run": False, "kept": len(to_keep),
                "deleted": len(to_delete), "freed_gb": round(freed_gb, 2)}


class VulnerabilityNotificationService:
    """漏洞通知与修补 SLA 管理"""

    SLA_CONFIG = {
        "CRITICAL": {
            "sla_hours": 24,
            "notification_channels": ["pager_duty", "slack_critical", "email"],
            "escalation_after_hours": 4,
            "escalation_to": "vp_engineering",
        },
        "HIGH": {
            "sla_hours": 168,  # 7 天
            "notification_channels": ["slack_security", "email"],
            "escalation_after_hours": 48,
            "escalation_to": "engineering_manager",
        },
        "MEDIUM": {
            "sla_hours": 720,  # 30 天
            "notification_channels": ["slack_security", "jira"],
            "escalation_after_hours": None,
            "escalation_to": None,
        },
        "LOW": {
            "sla_hours": 2160,  # 90 天
            "notification_channels": ["jira"],
            "escalation_after_hours": None,
            "escalation_to": None,
        },
    }

    def process_new_vulnerability(self, cve_id, affected_artifacts, severity):
        """处理新发现的漏洞"""
        sla = self.SLA_CONFIG[severity]
        deadline = now() + timedelta(hours=sla["sla_hours"])

        # 创建安全工单
        ticket = self.ticket_system.create({
            "type": "vulnerability_patch",
            "priority": "critical" if severity == "CRITICAL" else "high",
            "cve_id": cve_id,
            "severity": severity,
            "affected_artifacts": [a["artifact_id"] for a in affected_artifacts],
            "affected_environments": list(set(
                env for a in affected_artifacts
                for env in a.get("deployed_environments", []))),
            "sla_deadline": deadline,
            "description": self._build_vuln_description(cve_id, affected_artifacts),
        })

        # 为每个受影响制品创建修补子任务
        for artifact in affected_artifacts:
            self.ticket_system.create_subtask(ticket["id"], {
                "artifact_id": artifact["artifact_id"],
                "package": artifact.get("package_name"),
                "installed_version": artifact.get("installed_version"),
                "fixed_version": artifact.get("fixed_version"),
                "has_fix": artifact.get("fixed_version") is not None,
            })

        # 发送通知
        for channel in sla["notification_channels"]:
            self._send_notification(channel, {
                "cve_id": cve_id,
                "severity": severity,
                "affected_count": len(affected_artifacts),
                "sla_deadline": deadline,
                "ticket_id": ticket["id"],
                "has_fix": any(a.get("fixed_version") for a in affected_artifacts),
            })

        # 自动修补：有修复版本且严重级别为 Critical/High
        fixable = [a for a in affected_artifacts if a.get("fixed_version")]
        if fixable and severity in ("CRITICAL", "HIGH"):
            self._auto_create_patch_prs(cve_id, fixable, severity)

        # 设置升级提醒
        if sla["escalation_after_hours"]:
            self.scheduler.schedule(
                task="escalate_vulnerability",
                run_at=now() + timedelta(hours=sla["escalation_after_hours"]),
                params={"ticket_id": ticket["id"],
                        "escalate_to": sla["escalation_to"]},
            )

        return {"ticket_id": ticket["id"], "sla_deadline": deadline,
                "subtasks": len(affected_artifacts),
                "auto_patch_prs": len(fixable) if severity in ("CRITICAL", "HIGH") else 0}

    def check_sla_compliance(self):
        """检查漏洞修补 SLA 合规性"""
        overdue_tickets = self.ticket_system.query({
            "type": "vulnerability_patch",
            "status__ne": "resolved",
            "sla_deadline__lt": now(),
        })

        results = []
        for ticket in overdue_tickets:
            severity = ticket["severity"]
            sla = self.SLA_CONFIG[severity]
            overdue_hours = (now() - ticket["sla_deadline"]).total_seconds() / 3600

            results.append({
                "ticket_id": ticket["id"],
                "cve_id": ticket["cve_id"],
                "severity": severity,
                "sla_hours": sla["sla_hours"],
                "overdue_hours": round(overdue_hours, 1),
                "affected_environments": ticket["affected_environments"],
            })

            # 升级通知
            if overdue_hours > 0:
                self._send_notification("slack_critical", {
                    "message": f"SLA 逾期: {ticket['cve_id']} ({severity}) "
                               f"已逾期 {overdue_hours:.0f} 小时",
                    "ticket_id": ticket["id"],
                })

        return {"overdue_count": len(results), "items": results}

    def _auto_create_patch_prs(self, cve_id, fixable_artifacts, severity):
        """自动创建安全修补 PR"""
        for artifact in fixable_artifacts:
            branch = f"security/fix-{cve_id.lower()}-{artifact['package_name']}"

            # 创建分支
            self.git_ops.create_branch(
                repo=artifact["repo_url"],
                branch=branch,
                from_branch="main",
            )

            # 更新依赖版本
            self.git_ops.update_dependency(
                repo=artifact["repo_url"],
                branch=branch,
                package=artifact["package_name"],
                current_version=artifact["installed_version"],
                new_version=artifact["fixed_version"],
            )

            # 提交变更
            self.git_ops.commit(
                repo=artifact["repo_url"],
                branch=branch,
                message=f"security: bump {artifact['package_name']} "
                        f"from {artifact['installed_version']} "
                        f"to {artifact['fixed_version']} "
                        f"(fixes {cve_id})",
            )

            # 创建 PR
            pr = self.git_ops.create_pull_request(
                repo=artifact["repo_url"],
                title=f"[Security][{severity}] Fix {cve_id}: "
                      f"upgrade {artifact['package_name']}",
                body=f"## 自动安全修补\n\n"
                     f"- **CVE**: {cve_id}\n"
                     f"- **严重级别**: {severity}\n"
                     f"- **受影响包**: {artifact['package_name']}\n"
                     f"- **当前版本**: {artifact['installed_version']}\n"
                     f"- **修复版本**: {artifact['fixed_version']}\n"
                     f"- **SLA 截止**: "
                     f"{(now() + timedelta(hours=self.SLA_CONFIG[severity]['sla_hours'])).isoformat()}\n\n"
                     f"此 PR 由安全扫描自动创建，请尽快审核合并。",
                labels=["security", "automated", f"severity-{severity.lower()}"],
                reviewers=["security-team"],
            )

            # 关联工单
            self.ticket_system.link_pr(
                cve_id=cve_id,
                pr_url=pr["url"],
                artifact_id=artifact["artifact_id"],
            )


class PromotionAuditTrailService:
    """晋升审计追踪服务"""

    def record_promotion(self, promotion_id, source_env, target_env,
                         artifact_id, gate_results, approval_info, deploy_result):
        """记录晋升审计"""
        audit_record = {
            "audit_id": generate_id(),
            "promotion_id": promotion_id,
            "source_env": source_env,
            "target_env": target_env,
            "artifact_id": artifact_id,
            "gate_results": json.dumps(gate_results),
            "approval_mode": approval_info.get("mode", "unknown"),
            "approver": approval_info.get("approver"),
            "approval_timestamp": approval_info.get("approved_at"),
            "deploy_result": json.dumps(deploy_result),
            "initiated_by": self._get_initiator(promotion_id),
            "timestamp": now(),
        }

        self.db.insert("promotion_audit_trail", audit_record)

        # 发送到外部审计系统（合规要求）
        self.compliance_auditor.log({
            "event": "environment_promotion",
            "audit_id": audit_record["audit_id"],
            "timestamp": audit_record["timestamp"].isoformat(),
            "actor": audit_record["initiated_by"],
            "action": f"promote from {source_env} to {target_env}",
            "resource": artifact_id,
            "result": "success" if deploy_result.get("status") == "deployed" else "failed",
            "gate_results_summary": {
                g["gate"]: g["passed"] for g in gate_results
            },
        })

    def query_audit_trail(self, filters, limit=100):
        """查询审计记录"""
        query = "SELECT * FROM promotion_audit_trail WHERE 1=1"
        params = []

        if filters.get("artifact_id"):
            query += " AND artifact_id = %s"
            params.append(filters["artifact_id"])
        if filters.get("env"):
            query += " AND (source_env = %s OR target_env = %s)"
            params.extend([filters["env"], filters["env"]])
        if filters.get("initiated_by"):
            query += " AND initiated_by = %s"
            params.append(filters["initiated_by"])
        if filters.get("start_time"):
            query += " AND timestamp >= %s"
            params.append(filters["start_time"])
        if filters.get("end_time"):
            query += " AND timestamp <= %s"
            params.append(filters["end_time"])

        query += " ORDER BY timestamp DESC LIMIT %s"
        params.append(limit)

        records = self.db.query(query, *params)
        for record in records:
            record["gate_results"] = json.loads(record.get("gate_results", "[]"))
            record["deploy_result"] = json.loads(record.get("deploy_result", "{}"))

        return records
```

### 场景：晋升门禁 Flaky 测试误判阻断发布

```
触发：staging 集成测试中一个不稳定的测试（历史通过率 85%）偶尔失败
      → 阻断 staging→production 晋升 → 影响上线时间窗口
检测：
  1. 门禁检查失败但重试后通过 → 可能是 flaky test
  2. 对比该测试历史通过率 < 95% → 确认 flaky
  3. 同一 artifact 在 dev 环境测试全通过 → 不一致 → 误判可能
  4. 晋升流水线耗时异常增长（因重试）→ 需要关注
处理：
  1. 门禁失败自动重试 1 次（不重新构建，仅重跑测试）
  2. 重试通过 → 标记测试为 flaky，晋升继续，创建技术债工单
  3. 重试仍失败 → 确认阻断，通知开发团队排查
  4. Flaky 测试自动加入观察列表，连续 3 次出现 flaky → 自动隔离
  5. 被隔离测试不再作为门控条件，但记录在质量报告中
  6. 隔离测试超过 30 天未修复 → 强制修复（重新启用为门控）
预防：门禁自动重试机制 + flaky 测试检测（历史通过率追踪）
      + 分级隔离（观察→隔离→强制修复）+ 测试稳定性 SLO（>98%）
```

### 场景：已部署生产制品发现严重 CVE 紧急响应

```
触发：生产环境已运行的 Docker 镜像被扫描出 CVE-2024-XXXX（CVSS 9.8）
      → 可被远程利用 → 需要立即响应
检测：
  1. 每日定时安全扫描发现新 CVE → 匹配已部署制品 → Critical 级别
  2. NVD/CVE 安全情报源推送 → 自动匹配镜像层组件
  3. SBOM 索引快速定位受影响服务（通过 component→artifact→deployment 链路）
处理：
  1. 【评估】5 分钟内确认影响范围：
     - 受影响制品列表（通过 SBOM 查询）
     - 受影响环境（哪些 K8s 集群/命名空间部署了该制品）
     - 受影响服务列表与负责人
  2. 【修补】有修复版本时：
     - 紧急修补流水线（绕过人工审批，安全总监即时授权）
     - 重新构建 → Trivy 扫描 → 直通金丝雀部署（5%→100%）
     - 目标 4 小时内完成全量替换
  3. 【缓解】无修复版本时：
     - WAF 规则拦截攻击向量
     - 网络策略限制受影响服务的外部访问
     - 必要时临时下线非核心功能
  4. 【通知】安全事件通报所有使用该制品的团队
  5. 【验证】修补部署后验证漏洞已消除（重新扫描确认）
  6. 【复盘】48 小时内完成安全事件复盘：
     - 根因分析（依赖引入路径、扫描时效）
     - 改进措施（依赖锁定、扫描频率提升、供应链安全）
预防：每日生产镜像持续扫描 + 新 CVE 自动匹配已部署制品
      + SBOM 全链路追踪 + 紧急修补 SOP + 缓解措施知识库

## CI/CD 制品管理完整实现

```python
class ArtifactManagementService:
    """制品管理：构建产物 → 版本标记 → 存储 → 生命周期"""

    ARTIFACT_TYPES = {
        "docker_image": "Docker 镜像",
        "jar": "Java JAR",
        "wheel": "Python Wheel",
        "binary": "可执行文件",
        "config": "配置包",
    }

    RETENTION_POLICIES = {
        "release": {"keep_days": 365, "max_versions": None},
        "snapshot": {"keep_days": 30, "max_versions": 50},
        "pr_build": {"keep_days": 7, "max_versions": 10},
    }

    def register_artifact(self, build_id, artifact_type, artifact_data):
        """注册制品"""
        # 1. 验证制品完整性
        checksum = artifact_data.get("checksum")
        if not checksum:
            checksum = self._calculate_checksum(artifact_data["storage_path"])

        # 2. 检查是否已存在（同一 checksum）
        existing = self.db.query_one(
            "SELECT * FROM artifacts "
            "WHERE checksum = %s AND artifact_type = %s",
            checksum, artifact_type)

        if existing:
            return {"status": "already_exists",
                    "artifact_id": existing["artifact_id"]}

        # 3. 确定制品级别（release/snapshot/pr_build）
        branch = artifact_data.get("branch", "unknown")
        if branch in ["main", "master", "release"]:
            level = "release"
        elif branch.startswith("feature/") or branch.startswith("PR-"):
            level = "pr_build"
        else:
            level = "snapshot"

        # 4. 注册制品
        artifact_id = str(uuid4())
        version = self._generate_version(artifact_type, level, build_id)

        self.db.insert("artifacts", {
            "artifact_id": artifact_id,
            "build_id": build_id,
            "artifact_type": artifact_type,
            "version": version,
            "level": level,
            "branch": branch,
            "checksum": checksum,
            "size_bytes": artifact_data.get("size_bytes", 0),
            "storage_path": artifact_data["storage_path"],
            "metadata": json.dumps(artifact_data.get("metadata", {})),
            "status": "available",
            "created_at": now()
        })

        # 5. 标记关联制品（如 docker image + config 包）
        if artifact_data.get("related_artifacts"):
            for related_id in artifact_data["related_artifacts"]:
                self.db.insert("artifact_relations", {
                    "parent_id": artifact_id,
                    "child_id": related_id,
                    "relation_type": "deployment_bundle",
                    "created_at": now()
                })

        return {"artifact_id": artifact_id, "version": version,
                "level": level, "checksum": checksum}

    def promote_artifact(self, artifact_id, target_level, approver_id):
        """提升制品级别（snapshot → release）"""
        artifact = self.db.get_artifact(artifact_id)

        if artifact["level"] == target_level:
            return {"status": "already_at_level"}

        # 1. 验证制品质量
        quality_gate = self._check_quality_gate(artifact_id, target_level)
        if not quality_gate["passed"]:
            return {"status": "quality_gate_failed",
                    "issues": quality_gate["issues"]}

        # 2. 生成正式版本号
        new_version = self._generate_release_version(
            artifact["artifact_type"], target_level)

        # 3. 更新制品级别
        self.db.update("artifacts",
            {"level": target_level, "version": new_version,
             "promoted_at": now(), "promoted_by": approver_id},
            {"artifact_id": artifact_id})

        # 4. 记录提升日志
        self.db.insert("artifact_promotion_log", {
            "log_id": str(uuid4()),
            "artifact_id": artifact_id,
            "from_level": artifact["level"],
            "to_level": target_level,
            "old_version": artifact["version"],
            "new_version": new_version,
            "approver_id": approver_id,
            "quality_gate_result": json.dumps(quality_gate),
            "promoted_at": now()
        })

        return {"artifact_id": artifact_id,
                "from_level": artifact["level"],
                "to_level": target_level,
                "new_version": new_version}

    def cleanup_old_artifacts(self):
        """清理过期制品"""
        deleted_count = 0

        for level, policy in self.RETENTION_POLICIES.items():
            cutoff = now() - timedelta(days=policy["keep_days"])

            old_artifacts = self.db.query(
                "SELECT * FROM artifacts "
                "WHERE level = %s AND created_at < %s AND status = 'available' "
                "ORDER BY created_at ASC",
                level, cutoff)

            # 保留最低版本数
            if policy["max_versions"]:
                total_at_level = self.db.count("artifacts",
                    level=level, status="available")
                if total_at_level <= policy["max_versions"]:
                    continue

            for artifact in old_artifacts:
                # 删除存储
                self.object_storage.delete(artifact["storage_path"])

                # 标记为已删除
                self.db.update("artifacts",
                    {"status": "deleted", "deleted_at": now()},
                    {"artifact_id": artifact["artifact_id"]})

                deleted_count += 1

        return {"deleted_count": deleted_count}

    def _check_quality_gate(self, artifact_id, target_level):
        """检查制品质量门禁"""
        if target_level != "release":
            return {"passed": True, "issues": []}

        issues = []

        # 检查关联构建是否成功
        artifact = self.db.get_artifact(artifact_id)
        build = self.db.get_build(artifact["build_id"])

        if build["status"] != "success":
            issues.append("关联构建未成功")

        # 检查测试覆盖率
        test_results = self.db.query(
            "SELECT * FROM test_results WHERE build_id = %s",
            artifact["build_id"])

        if not test_results or any(r["status"] == "failed" for r in test_results):
            issues.append("测试未通过")

        # 检查安全扫描
        security_scan = self.db.query_one(
            "SELECT * FROM security_scans WHERE artifact_id = %s "
            "ORDER BY scanned_at DESC LIMIT 1", artifact_id)

        if not security_scan or security_scan["vulnerabilities"] > 0:
            issues.append(f"安全扫描发现 {security_scan['vulnerabilities'] if security_scan else '未扫描'} 个漏洞")

        return {"passed": len(issues) == 0, "issues": issues}

    def _generate_version(self, artifact_type, level, build_id):
        """生成版本号"""
        if level == "release":
            return self._generate_release_version(artifact_type, level)
        elif level == "snapshot":
            build = self.db.get_build(build_id)
            return f"{build.get('base_version', '0.0')}-SNAPSHOT-{build['build_number']}"
        else:
            return f"PR-{build_id}"

    def _generate_release_version(self, artifact_type, level):
        """生成正式版本号"""
        latest = self.db.query_one(
            "SELECT version FROM artifacts "
            "WHERE artifact_type = %s AND level = 'release' "
            "ORDER BY created_at DESC LIMIT 1",
            artifact_type)

        if latest:
            # 递增版本号
            parts = latest["version"].split(".")
            parts[-1] = str(int(parts[-1]) + 1)
            return ".".join(parts)
        else:
            return "1.0.0"

    def _calculate_checksum(self, storage_path):
        """计算制品 checksum"""
        import hashlib
        sha256 = hashlib.sha256()
        # 从对象存储读取并计算
        data = self.object_storage.read(storage_path)
        for chunk in data:
            sha256.update(chunk)
        return sha256.hexdigest()
```

## 异常场景补充

### 场景：制品存储空间耗尽

```
触发：大量 PR 构建制品堆积 → 存储空间 100% → 新构建无法上传制品 → CI 失败
检测：
  1. 存储使用率 > 85% → 空间不足
  2. 制品上传失败 → 存储满
处理：
  1. 自动清理 PR 构建制品（保留 7 天）
  2. 清理旧版本 snapshot
  3. 扩展存储容量
预防：自动清理 + 保留策略 + 存储监控
```

### 场景：制品提升后旧版本未清理

```
触发：snapshot v1.0-SNAPSHOT-42 提升为 release v1.0 → 旧 snapshot 版本仍在 → 混淆
检测：
  1. 同一构建有 snapshot 和 release 两个制品 → 版本重复
  2. snapshot 制品与 release 制品 checksum 相同 → 需要清理
处理：
  1. 提升时自动清理原 snapshot 版本
  2. 标记 snapshot 为 "promoted" 状态
  3. 部署只指向 release 版本
预防：自动清理 + 标记 + 部署指向
```

## CI/CD 流水线回滚完整实现

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional
from datetime import datetime, timedelta
import logging
import json

logger = logging.getLogger(__name__)


class DeploymentStrategy(Enum):
    ROLLING = "rolling"
    CANARY = "canary"
    BLUE_GREEN = "blue_green"


class RollbackReason(Enum):
    ERROR_RATE_SPIKE = "error_rate_spike"
    HEALTH_CHECK_FAILURE = "health_check_failure"
    LATENCY_DEGRADATION = "latency_degradation"
    MANUAL_TRIGGER = "manual_trigger"
    DEPENDENCY_FAILURE = "dependency_failure"


class RollbackStatus(Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    FAILED = "failed"
    PARTIALLY_COMPLETED = "partially_completed"


class ServiceHealth(Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    UNHEALTHY = "unhealthy"
    UNKNOWN = "unknown"


@dataclass
class DeploymentVersion:
    service_name: str
    version: str
    deploy_time: datetime
    strategy: DeploymentStrategy
    previous_version: str = ""
    image_tag: str = ""
    config_hash: str = ""


@dataclass
class HealthCheckResult:
    service_name: str
    healthy: bool
    error_rate: float  # 0-1
    p99_latency_ms: float
    response_code_distribution: dict[str, int] = field(default_factory=dict)
    checked_at: datetime = field(default_factory=datetime.now)
    details: str = ""


@dataclass
class RollbackTarget:
    service_name: str
    current_version: str
    target_version: str
    previous_version: str  # 回滚目标的前一个版本(用于验证)
    strategy: DeploymentStrategy
    priority: int = 0  # 越高越先回滚
    canary_percentage: int = 0  # canary策略时的当前流量比例


@dataclass
class RollbackPlan:
    plan_id: str
    deployment_id: str
    reason: RollbackReason
    targets: list[RollbackTarget] = field(default_factory=list)
    created_at: datetime = field(default_factory=datetime.now)
    estimated_downtime_minutes: float = 0.0
    risk_level: str = "low"  # low, medium, high
    requires_approval: bool = False
    pre_rollback_snapshot: dict[str, str] = field(default_factory=dict)


@dataclass
class RollbackExecution:
    execution_id: str
    plan: RollbackPlan
    status: RollbackStatus = RollbackStatus.PENDING
    started_at: Optional[datetime] = None
    completed_at: Optional[datetime] = None
    completed_targets: list[str] = field(default_factory=list)
    failed_targets: list[str] = field(default_factory=list)
    rollback_steps: list[dict] = field(default_factory=list)


class PipelineRollbackService:
    """CI/CD 流水线回滚服务，检测部署故障并执行安全的版本回滚。"""

    ERROR_RATE_THRESHOLD = 0.05  # 错误率超过5%触发告警
    ERROR_RATE_CRITICAL = 0.15   # 错误率超过15%立即回滚
    LATENCY_THRESHOLD_MS = 2000  # P99延迟超过2秒触发告警
    HEALTH_CHECK_RETRIES = 3
    HEALTH_CHECK_INTERVAL_SECONDS = 10
    CANARY_ROLLBACK_STEP_PERCENT = 25  # canary每步回滚25%流量

    def __init__(
        self,
        deployment_history: dict[str, list[DeploymentVersion]],
        service_configs: dict[str, dict],
    ):
        self.deployment_history = deployment_history  # service_name -> list of versions
        self.service_configs = service_configs
        self._plan_counter = 0
        self._execution_counter = 0
        self._active_executions: dict[str, RollbackExecution] = {}
        self._snapshots: dict[str, dict] = {}  # 保存回滚前状态快照

    def _generate_plan_id(self) -> str:
        self._plan_counter += 1
        return f"RBP-{self._plan_counter:06d}"

    def _generate_execution_id(self) -> str:
        self._execution_counter += 1
        return f"RBE-{self._execution_counter:06d}"

    def detect_failure(self, health_results: list[HealthCheckResult]) -> list[dict]:
        """检测部署故障，返回故障描述列表。"""
        failures: list[dict] = []

        for result in health_results:
            service_failures = []

            # 检查错误率
            if result.error_rate >= self.ERROR_RATE_CRITICAL:
                service_failures.append({
                    "type": RollbackReason.ERROR_RATE_SPIKE,
                    "severity": "critical",
                    "metric": result.error_rate,
                    "threshold": self.ERROR_RATE_CRITICAL,
                    "message": f"服务 {result.service_name} 错误率 {result.error_rate:.1%} 超过临界值 {self.ERROR_RATE_CRITICAL:.1%}",
                })
            elif result.error_rate >= self.ERROR_RATE_THRESHOLD:
                service_failures.append({
                    "type": RollbackReason.ERROR_RATE_SPIKE,
                    "severity": "warning",
                    "metric": result.error_rate,
                    "threshold": self.ERROR_RATE_THRESHOLD,
                    "message": f"服务 {result.service_name} 错误率 {result.error_rate:.1%} 超过告警阈值 {self.ERROR_RATE_THRESHOLD:.1%}",
                })

            # 检查健康状态
            if not result.healthy:
                service_failures.append({
                    "type": RollbackReason.HEALTH_CHECK_FAILURE,
                    "severity": "critical",
                    "metric": result.details,
                    "message": f"服务 {result.service_name} 健康检查失败: {result.details}",
                })

            # 检查延迟
            if result.p99_latency_ms > self.LATENCY_THRESHOLD_MS:
                service_failures.append({
                    "type": RollbackReason.LATENCY_DEGRADATION,
                    "severity": "warning" if result.p99_latency_ms < self.LATENCY_THRESHOLD_MS * 2 else "critical",
                    "metric": result.p99_latency_ms,
                    "threshold": self.LATENCY_THRESHOLD_MS,
                    "message": f"服务 {result.service_name} P99延迟 {result.p99_latency_ms}ms 超过阈值 {self.LATENCY_THRESHOLD_MS}ms",
                })

            # 检查5xx比例
            total_requests = sum(result.response_code_distribution.values())
            if total_requests > 0:
                server_errors = sum(
                    count for code, count in result.response_code_distribution.items()
                    if code.startswith("5")
                )
                server_error_rate = server_errors / total_requests
                if server_error_rate > self.ERROR_RATE_THRESHOLD:
                    service_failures.append({
                        "type": RollbackReason.ERROR_RATE_SPIKE,
                        "severity": "critical" if server_error_rate > self.ERROR_RATE_CRITICAL else "warning",
                        "metric": server_error_rate,
                        "message": f"服务 {result.service_name} 5xx错误率 {server_error_rate:.1%}",
                    })

            failures.extend(service_failures)

        if failures:
            critical_count = sum(1 for f in failures if f["severity"] == "critical")
            logger.warning(f"检测到 {len(failures)} 个故障，其中 {critical_count} 个为严重级别")
        else:
            logger.info("所有服务健康检查通过，无故障检测")

        return failures

    def create_rollback_plan(
        self,
        deployment_id: str,
        reason: RollbackReason,
        affected_services: list[str],
        forced_target_versions: Optional[dict[str, str]] = None,
    ) -> RollbackPlan:
        """创建回滚计划，确定每个服务的回滚目标版本和策略。"""
        targets: list[RollbackTarget] = []

        for service_name in affected_services:
            history = self.deployment_history.get(service_name, [])
            if len(history) < 1:
                logger.warning(f"服务 {service_name} 没有部署历史，无法创建回滚目标")
                continue

            current = history[-1]
            # 查找上一个稳定版本
            target_version = self._find_rollback_target(
                service_name, history, forced_target_versions
            )
            if not target_version:
                logger.error(f"服务 {service_name} 无法找到合适的回滚目标版本")
                continue

            # 确定回滚前一个版本(用于验证目标版本本身是否稳定)
            previous_of_target = self._find_previous_of_target(service_name, target_version, history)

            target = RollbackTarget(
                service_name=service_name,
                current_version=current.version,
                target_version=target_version,
                previous_version=previous_of_target,
                strategy=current.strategy,
                priority=self._calculate_rollback_priority(service_name, reason),
                canary_percentage=0,
            )
            targets.append(target)

        # 按优先级排序(高优先级先回滚)
        targets.sort(key=lambda t: t.priority, reverse=True)

        # 评估风险等级
        risk_level = self._assess_risk_level(targets, reason)

        # 估算停机时间
        estimated_downtime = self._estimate_downtime(targets)

        # 保存回滚前快照
        snapshot = {}
        for target in targets:
            snapshot[target.service_name] = target.current_version

        plan = RollbackPlan(
            plan_id=self._generate_plan_id(),
            deployment_id=deployment_id,
            reason=reason,
            targets=targets,
            estimated_downtime_minutes=estimated_downtime,
            risk_level=risk_level,
            requires_approval=(risk_level == "high" or len(targets) > 5),
            pre_rollback_snapshot=snapshot,
        )

        logger.info(
            f"创建回滚计划 {plan.plan_id}: {len(targets)} 个服务, "
            f"风险={risk_level}, 预计停机={estimated_downtime:.1f}分钟"
        )
        return plan

    def _find_rollback_target(
        self,
        service_name: str,
        history: list[DeploymentVersion],
        forced_versions: Optional[dict[str, str]] = None,
    ) -> str:
        """查找合适的回滚目标版本。"""
        if forced_versions and service_name in forced_versions:
            return forced_versions[service_name]

        # 回退到上一个版本
        if len(history) >= 2:
            return history[-2].version
        elif len(history) == 1:
            return history[0].previous_version if history[0].previous_version else ""

        return ""

    def _find_previous_of_target(
        self, service_name: str, target_version: str, history: list[DeploymentVersion]
    ) -> str:
        """查找回滚目标版本的前一个版本(用于验证)。"""
        for i, deploy in enumerate(history):
            if deploy.version == target_version and i > 0:
                return history[i - 1].version
        return ""

    def _calculate_rollback_priority(self, service_name: str, reason: RollbackReason) -> int:
        """计算回滚优先级。"""
        priority = 0
        if reason == RollbackReason.ERROR_RATE_SPIKE:
            priority += 10
        elif reason == RollbackReason.HEALTH_CHECK_FAILURE:
            priority += 8
        elif reason == RollbackReason.LATENCY_DEGRADATION:
            priority += 5

        # 核心服务优先回滚
        config = self.service_configs.get(service_name, {})
        if config.get("tier") == "critical":
            priority += 5
        elif config.get("tier") == "important":
            priority += 3

        return priority

    def _assess_risk_level(self, targets: list[RollbackTarget], reason: RollbackReason) -> str:
        """评估回滚风险等级。"""
        critical_services = [
            t for t in targets
            if self.service_configs.get(t.service_name, {}).get("tier") == "critical"
        ]

        if len(critical_services) > 2 or reason == RollbackReason.HEALTH_CHECK_FAILURE:
            return "high"
        elif len(critical_services) > 0 or len(targets) > 3:
            return "medium"
        return "low"

    def _estimate_downtime(self, targets: list[RollbackTarget]) -> float:
        """估算回滚停机时间。"""
        total_minutes = 0.0
        for target in targets:
            if target.strategy == DeploymentStrategy.BLUE_GREEN:
                total_minutes += 1.0  # 切换几乎无停机
            elif target.strategy == DeploymentStrategy.CANARY:
                total_minutes += 5.0  # 逐步回滚需要时间
            else:  # ROLLING
                total_minutes += 3.0  # 滚动回滚
        return total_minutes

    def execute_rollback(self, plan: RollbackPlan, auto_approve: bool = False) -> RollbackExecution:
        """执行回滚计划，根据部署策略选择不同的回滚方式。"""
        if plan.requires_approval and not auto_approve:
            logger.warning(f"回滚计划 {plan.plan_id} 需要人工审批，但未获得自动审批授权")
            return RollbackExecution(
                execution_id=self._generate_execution_id(),
                plan=plan,
                status=RollbackStatus.PENDING,
            )

        execution = RollbackExecution(
            execution_id=self._generate_execution_id(),
            plan=plan,
            status=RollbackStatus.IN_PROGRESS,
            started_at=datetime.now(),
        )
        self._active_executions[execution.execution_id] = execution

        # 保存回滚前快照
        self._snapshots[execution.execution_id] = dict(plan.pre_rollback_snapshot)

        for target in plan.targets:
            try:
                logger.info(
                    f"开始回滚服务 {target.service_name}: "
                    f"{target.current_version} -> {target.target_version} "
                    f"(策略: {target.strategy.value})"
                )

                if target.strategy == DeploymentStrategy.CANARY:
                    steps = self._execute_canary_rollback(target, execution)
                elif target.strategy == DeploymentStrategy.BLUE_GREEN:
                    steps = self._execute_blue_green_rollback(target, execution)
                else:
                    steps = self._execute_rolling_rollback(target, execution)

                execution.rollback_steps.extend(steps)
                execution.completed_targets.append(target.service_name)
                logger.info(f"服务 {target.service_name} 回滚完成")

            except Exception as e:
                logger.error(f"服务 {target.service_name} 回滚失败: {e}")
                execution.failed_targets.append(target.service_name)
                execution.rollback_steps.append({
                    "service": target.service_name,
                    "action": "rollback_failed",
                    "error": str(e),
                    "timestamp": datetime.now().isoformat(),
                })

        # 确定最终状态
        if not execution.failed_targets:
            execution.status = RollbackStatus.COMPLETED
        elif execution.completed_targets:
            execution.status = RollbackStatus.PARTIALLY_COMPLETED
        else:
            execution.status = RollbackStatus.FAILED

        execution.completed_at = datetime.now()
        logger.info(
            f"回滚执行 {execution.execution_id} 完成: 状态={execution.status.value}, "
            f"成功={len(execution.completed_targets)}, 失败={len(execution.failed_targets)}"
        )
        return execution

    def _execute_canary_rollback(
        self, target: RollbackTarget, execution: RollbackExecution
    ) -> list[dict]:
        """执行Canary策略回滚：逐步将流量从新版本切回旧版本。"""
        steps: list[dict] = []
        current_canary_percent = 100  # 假设当前canary已100%流量

        while current_canary_percent > 0:
            step_percent = min(self.CANARY_ROLLBACK_STEP_PERCENT, current_canary_percent)
            current_canary_percent -= step_percent

            step = {
                "service": target.service_name,
                "action": "canary_traffic_shift",
                "new_version_traffic": f"{current_canary_percent}%",
                "old_version_traffic": f"{100 - current_canary_percent}%",
                "timestamp": datetime.now().isoformat(),
            }
            steps.append(step)
            logger.info(
                f"Canary回滚 {target.service_name}: 新版本流量降至 {current_canary_percent}%"
            )

        # 确认流量完全切回
        steps.append({
            "service": target.service_name,
            "action": "canary_rollback_complete",
            "active_version": target.target_version,
            "timestamp": datetime.now().isoformat(),
        })
        return steps

    def _execute_blue_green_rollback(
        self, target: RollbackTarget, execution: RollbackExecution
    ) -> list[dict]:
        """执行Blue-Green策略回滚：将流量切回之前的蓝色/绿色环境。"""
        steps: list[dict] = []

        # 切换路由指向旧环境
        steps.append({
            "service": target.service_name,
            "action": "blue_green_switch",
            "from_environment": "current",
            "to_environment": "previous",
            "active_version": target.target_version,
            "timestamp": datetime.now().isoformat(),
        })

        # 验证旧环境正常接收流量
        steps.append({
            "service": target.service_name,
            "action": "blue_green_verify",
            "status": "traffic_redirected",
            "timestamp": datetime.now().isoformat(),
        })

        logger.info(f"Blue-Green回滚 {target.service_name}: 流量切换至版本 {target.target_version}")
        return steps

    def _execute_rolling_rollback(
        self, target: RollbackTarget, execution: RollbackExecution
    ) -> list[dict]:
        """执行Rolling策略回滚：逐个实例替换为新版本。"""
        steps: list[dict] = []

        # 获取实例数量
        config = self.service_configs.get(target.service_name, {})
        instance_count = config.get("instance_count", 3)

        for i in range(instance_count):
            steps.append({
                "service": target.service_name,
                "action": "rolling_replace_instance",
                "instance_index": i,
                "from_version": target.current_version,
                "to_version": target.target_version,
                "timestamp": datetime.now().isoformat(),
            })

        steps.append({
            "service": target.service_name,
            "action": "rolling_rollback_complete",
            "active_version": target.target_version,
            "total_instances_replaced": instance_count,
            "timestamp": datetime.now().isoformat(),
        })

        logger.info(f"Rolling回滚 {target.service_name}: {instance_count} 个实例替换完成")
        return steps

    def validate_rollback(self, execution: RollbackExecution,
                          health_results: list[HealthCheckResult]) -> dict:
        """验证回滚是否成功，检查所有回滚服务的健康状态。"""
        if execution.status not in (RollbackStatus.COMPLETED, RollbackStatus.PARTIALLY_COMPLETED):
            return {
                "valid": False,
                "reason": f"回滚执行状态为 {execution.status.value}，无法验证",
                "services": {},
            }

        validation_results: dict[str, dict] = {}

        for target in execution.plan.targets:
            if target.service_name in execution.failed_targets:
                validation_results[target.service_name] = {
                    "rolled_back": False,
                    "healthy": ServiceHealth.UNKNOWN.value,
                    "error": "回滚失败，服务未切换版本",
                }
                continue

            # 查找该服务的健康检查结果
            service_health = next(
                (h for h in health_results if h.service_name == target.service_name),
                None,
            )

            if not service_health:
                validation_results[target.service_name] = {
                    "rolled_back": True,
                    "healthy": ServiceHealth.UNKNOWN.value,
                    "warning": "未找到健康检查结果",
                }
                continue

            # 综合判断健康状态
            health_status = ServiceHealth.HEALTHY
            issues: list[str] = []

            if not service_health.healthy:
                health_status = ServiceHealth.UNHEALTHY
                issues.append(f"健康检查失败: {service_health.details}")
            elif service_health.error_rate > self.ERROR_RATE_THRESHOLD:
                health_status = ServiceHealth.DEGRADED
                issues.append(f"错误率仍偏高: {service_health.error_rate:.1%}")
            elif service_health.p99_latency_ms > self.LATENCY_THRESHOLD_MS:
                health_status = ServiceHealth.DEGRADED
                issues.append(f"P99延迟偏高: {service_health.p99_latency_ms}ms")

            validation_results[target.service_name] = {
                "rolled_back": True,
                "target_version": target.target_version,
                "healthy": health_status.value,
                "error_rate": service_health.error_rate,
                "p99_latency_ms": service_health.p99_latency_ms,
                "issues": issues,
            }

        # 整体验证结论
        all_healthy = all(
            v.get("healthy") in (ServiceHealth.HEALTHY.value, ServiceHealth.UNKNOWN.value)
            for v in validation_results.values()
            if v.get("rolled_back", False)
        )
        any_unhealthy = any(
            v.get("healthy") == ServiceHealth.UNHEALTHY.value
            for v in validation_results.values()
        )

        if any_unhealthy:
            overall_valid = False
            overall_reason = "存在不健康服务，回滚验证失败"
        elif all_healthy:
            overall_valid = True
            overall_reason = "所有回滚服务健康检查通过"
        else:
            overall_valid = True
            overall_reason = "回滚服务基本正常，但存在降级服务需持续监控"

        report = {
            "valid": overall_valid,
            "reason": overall_reason,
            "execution_id": execution.execution_id,
            "services": validation_results,
            "snapshot_before": self._snapshots.get(execution.execution_id, {}),
        }

        logger.info(f"回滚验证完成: valid={overall_valid}, reason={overall_reason}")
        return report
```

## 异常场景补充

### 场景：回滚到也有问题的版本
```
trigger: 当前部署版本v2.3出现错误率飙升触发回滚，系统自动回滚到上一个版本v2.2，但v2.2本身也存在未修复的内存泄漏问题，回滚后错误率虽有下降但仍在告警阈值之上，服务处于降级运行状态
detection: 1) validate_rollback中检测到回滚后error_rate仍超过ERROR_RATE_THRESHOLD但低于ERROR_RATE_CRITICAL；2) 连续2次回滚后健康检查结果仍为DEGRADED；3) 回滚后P99延迟较回滚前无明显改善
handling: 1) 立即将回滚计划标记为risk_level=high，通知oncall工程师；2) 自动扩展搜索范围：在_find_rollback_target中跳过紧邻的上一版本，查找更早的已知稳定版本(检查该版本的运行时长是否超过72小时且当时健康)；3) 如果存在稳定版本则创建新的回滚计划指向该版本；4) 如果不存在已知稳定版本，则保持当前降级状态并启动紧急修复流程；5) 在deploy_history中标记v2.2为"unstable"避免后续再被选为回滚目标
prevention: 1) 每次validate_rollback通过后将该版本标记为"verified_stable"并存入稳定版本池；2) create_rollback_plan中增加版本稳定性校验：候选回滚版本必须是verified_stable或运行时长超过72小时的版本；3) 部署流水线中增加"预回滚验证"步骤——每次新部署前自动验证上一个版本仍可正常运行；4) 维护每个服务的稳定版本白名单，强制回滚只能选择白名单中的版本；5) 关键服务(tier=critical)至少保留3个历史版本可供回滚
```

### 场景：回滚过程中服务中断
```
trigger: 在执行rolling回滚时，多个实例同时被替换导致可用实例数低于最小可用数(min_ready=2)；或blue-green切换时旧环境已下线无法接收流量；或canary回滚时配置错误导致流量全部丢弃
detection: 1) execute_rollback中监控到某一步骤后服务健康检查连续2次返回unhealthy；2) 回滚过程中活跃实例数低于配置的min_ready数量；3) 回滚步骤超时(单步超过5分钟未完成)；4) 流量切换后5xx错误率瞬间超过50%
handling: 1) 立即暂停回滚执行，将execution.status设为PARTIALLY_COMPLETED；2) 触发紧急恢复：对已回滚的服务保持回滚后版本，对未回滚的服务恢复到回滚前版本(使用snapshots中的快照)；3) 对于rolling策略：停止实例替换，确保当前在线实例数恢复到min_ready；4) 对于blue-green策略：如果旧环境不可用，立即重新部署回滚前版本到备环境并切换流量；5) 对于canary策略：将流量立即100%指向当前任何可用的版本实例；6) 所有恢复操作记录到rollback_steps中供事后分析
prevention: 1) rolling回滚时强制串行替换，每批最多替换1个实例，等待健康检查通过后才替换下一个；2) 设置实例可用性守卫：在替换前检查剩余可用实例数是否>=min_ready，不满足则阻塞；3) blue-green部署时旧环境在下线前必须保留至少30分钟(冷却期)，确保可随时切回；4) canary回滚前先验证旧版本端点可达性，不可达则不执行流量切换；5) 每个回滚步骤设置超时(5分钟)，超时自动暂停并告警；6) 回滚前自动创建数据库schema快照和配置快照，确保回滚完整性
```
