# C28: A/B测试平台的数据准确性

## 业务场景

某互联网公司的增长团队每天运行 50+ 个 A/B 实验：新按钮颜色是否提升点击率？新推荐算法是否提升停留时长？新定价策略是否提升转化？

**一个真实的故事：** 2023 年某电商的推荐算法 A/B 测试显示"新算法提升转化率 15%（p=0.03）"→ 团队花了 2 个月全面上线 → 上线后转化率反而下降 5% → 原因：实验期间恰逢促销季，新算法在促销流量上表现好，但日常流量上不如旧算法 → **辛普森悖论 + 样本量不足** → 浪费 2 个月开发资源 + 营收损失。

**已知数据：**
- 日均实验数：50+
- 每个实验用户量：1 万-100 万
- 实验时长：1-4 周
- 指标数：每个实验约 10 个（点击率、转化率、停留时长、收入等）
- 统计显著性要求：p-value < 0.05
- 用户换设备率：约 5%（导致前端分组丢失）

**为什么"分两组、看谁好"实际上很难？**

| 陷阱 | 假阳性率 | 后果 |
|------|---------|------|
| 不偷看 | 5%（正常） | - |
| 每天偷看一次 | 15-20% | 10 个实验中 2 个假阳性 |
| 每天偷看+发现显著就停 | 30%+ | 10 个实验中 3 个假阳性 |
| 10 个指标不校正 | 40% | 10 个指标中几乎必然有 1 个假阳性 |
| 偷看 + 多指标 + 样本不足 | 60%+ | 大部分结论不可信 |

## 核心挑战

### 挑战 1：实验分组的正确性与稳定性

用户必须被稳定分组——今天看到新版本，明天不能看到旧版本。但 5% 的用户会换设备/清 Cookie → 分组丢失。跨实验分组必须独立——否则用户在所有实验中都在实验组 → 偏差。

### 挑战 2：偷看与早期停止

实验还没结束就看数据，发现"显著"就停止。传统 t 检验假设"只看一次"——看 N 次让假阳性率从 5% 飙升。

**偷看对假阳性率的影响：**

| 偷看频率 | 假阳性率 | 是5%的几倍 |
|---------|---------|-----------|
| 只看1次（正确） | 5% | 1x |
| 每天1次（7天实验） | 15% | 3x |
| 每天1次+提前停止 | 30% | 6x |
| 每小时1次+提前停止 | 45% | 9x |

### 挑战 3：多指标多重比较

测试 10 个指标，每个 p < 0.05 → 至少 1 个假阳性概率 = 1 - 0.95^10 ≈ 40%。如果恰好这 1 个被当成了结论 → 错误决策。

### 挑战 4：辛普森悖论

整体数据看 A 好，但分设备看 B 好 → 结论取决于分析维度。

**具体案例：**

| 分组 | A 组用户数 | A 组转化率 | B 组用户数 | B 组转化率 |
|------|-----------|-----------|-----------|-----------|
| 手机端 | 80000 | 3% | 20000 | 2% |
| 桌面端 | 20000 | 10% | 80000 | 8% |
| **合计** | **100000** | **4.4%** | **100000** | **6.8%** |

整体 B 好于 A（6.8% > 4.4%），但分设备看 A 好于 B。原因：A 组手机端占比高（转化率天然低），拉低了整体均值。

## 设计约束

- 分组查询延迟 < 5ms（P99）
- 分组一致性 99.99%（同一用户同一组）
- 并发实验数 50+
- 日均分组写入 1000 万次
- 口径校准偏差 < 0.1%

## 数据库设计

A/B 测试平台的核心数据模型需要支持：实验配置管理、分组分配一致性、指标数据高效聚合、结果统计检验。以下是完整的数据库设计。

```sql
-- 实验配置表：存储实验的基本信息和参数
CREATE TABLE experiments (
    id              VARCHAR(64) PRIMARY KEY,          -- 实验 ID（如 exp_20240315_btn_color）
    name            VARCHAR(256) NOT NULL,             -- 实验名称
    description     TEXT,                              -- 实验描述与假设
    hypothesis      VARCHAR(512),                      -- 原假设与备择假设
    owner           VARCHAR(64) NOT NULL,              -- 实验负责人
    team            VARCHAR(64),                       -- 所属团队
    status          ENUM('DRAFT','RUNNING','PAUSED','COMPLETED','INVALID') NOT NULL DEFAULT 'DRAFT',
    -- 分流配置
    traffic_percent DECIMAL(5,2) NOT NULL DEFAULT 100.00,  -- 实验占全站流量百分比
    mutex_group_id  VARCHAR(64),                       -- 互斥组 ID（同组实验互斥）
    -- 统计参数
    baseline_rate   DECIMAL(8,6),                      -- 基线转化率（如 0.05 表示 5%）
    mde             DECIMAL(6,4),                      -- 最小可检测效果（如 0.10 表示 10% 提升）
    alpha           DECIMAL(4,3) NOT NULL DEFAULT 0.050,  -- 显著性水平
    power           DECIMAL(4,3) NOT NULL DEFAULT 0.800,  -- 统计功效
    sample_size_per_group INT,                         -- 每组所需样本量（计算得出）
    -- 时间配置
    min_duration_days   INT NOT NULL DEFAULT 7,        -- 最短实验天数（含完整周期效应）
    max_duration_days   INT NOT NULL DEFAULT 28,       -- 最长实验天数
    started_at      TIMESTAMP,                         -- 实际开始时间
    ended_at        TIMESTAMP,                         -- 实际结束时间
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_status (status),
    INDEX idx_owner (owner),
    INDEX idx_mutex_group (mutex_group_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 实验分组表：每个实验的变体配置
CREATE TABLE experiment_groups (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    experiment_id   VARCHAR(64) NOT NULL,              -- 关联实验
    group_name      VARCHAR(64) NOT NULL,              -- 分组名（control / treatment_a / treatment_b）
    description     VARCHAR(256),                      -- 分组描述
    ratio           DECIMAL(5,4) NOT NULL,             -- 该组占实验流量的比例（如 0.5000）
    is_control      BOOLEAN NOT NULL DEFAULT FALSE,    -- 是否为对照组
    config_snapshot JSON,                              -- 该组对应的特征配置快照
    UNIQUE KEY uk_exp_group (experiment_id, group_name),
    INDEX idx_experiment (experiment_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 用户分组分配表：保证分组一致性（核心表，写入量极大）
CREATE TABLE user_assignments (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id         VARCHAR(64) NOT NULL,              -- 用户 ID
    experiment_id   VARCHAR(64) NOT NULL,              -- 实验 ID
    group_name      VARCHAR(64) NOT NULL,              -- 分配的分组
    hash_bucket     INT NOT NULL,                      -- 哈希桶编号 0-9999（用于调试和校验）
    assigned_at     TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    -- 设备信息（用于 SRM 检测和跨设备分析）
    first_device    VARCHAR(32),                       -- 首次分配时的设备类型
    first_platform  VARCHAR(32),                       -- 首次分配时的平台
    UNIQUE KEY uk_user_exp (user_id, experiment_id),   -- 同一用户同一实验只有一条记录
    INDEX idx_experiment_group (experiment_id, group_name),
    INDEX idx_user (user_id),
    INDEX idx_assigned_at (assigned_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 分区策略：按月分区，方便历史数据归档
-- ALTER TABLE user_assignments PARTITION BY RANGE (TO_DAYS(assigned_at)) (...);

-- 指标定义表：每个实验关注的指标配置
CREATE TABLE metrics (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    experiment_id   VARCHAR(64) NOT NULL,
    metric_name     VARCHAR(128) NOT NULL,             -- 指标名（click_rate / conversion_rate / revenue_per_user）
    metric_type     ENUM('RATIO','CONTINUOUS','COUNT') NOT NULL,  -- 指标类型
    -- 指标类型说明：
    --   RATIO: 比率型（转化率、点击率），用比例检验
    --   CONTINUOUS: 连续型（停留时长、客单价），用 t 检验
    --   COUNT: 计数型（页面浏览次数），用泊松/负二项检验
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,    -- 是否为核心指标（核心指标用 Bonferroni，其余用 FDR）
    direction       ENUM('UP','DOWN','BOTH') NOT NULL DEFAULT 'BOTH',  -- 期望方向
    baseline_value  DECIMAL(12,6),                     -- 基线值
    min_detectable_change DECIMAL(8,6),               -- 最小可检测变化量
    correction_method ENUM('BONFERRONI','FDR','NONE') DEFAULT NULL,  -- 该指标的多重比较校正方法
    UNIQUE KEY uk_exp_metric (experiment_id, metric_name),
    INDEX idx_experiment (experiment_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 实验结果表：统计检验结果存储
CREATE TABLE experiment_results (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    experiment_id   VARCHAR(64) NOT NULL,
    metric_name     VARCHAR(128) NOT NULL,
    -- 原始统计量
    control_mean    DECIMAL(14,6),                     -- 对照组均值
    control_stddev  DECIMAL(14,6),                     -- 对照组标准差
    control_n       INT,                               -- 对照组样本量
    treatment_mean  DECIMAL(14,6),                     -- 实验组均值
    treatment_stddev DECIMAL(14,6),                    -- 实验组标准差
    treatment_n     INT,                               -- 实验组样本量
    -- 检验结果
    effect_size     DECIMAL(10,6),                     -- 效果量（Cohen's h / d）
    relative_change DECIMAL(8,4),                      -- 相对变化率（如 +0.1234 表示 +12.34%）
    p_value         DECIMAL(10,8),                     -- 原始 p 值
    adjusted_p_value DECIMAL(10,8),                    -- 校正后 p 值
    ci_lower        DECIMAL(14,6),                     -- 置信区间下界
    ci_upper        DECIMAL(14,6),                     -- 置信区间上界
    is_significant  BOOLEAN,                           -- 是否显著（校正后）
    -- 质量检测
    srm_p_value     DECIMAL(10,8),                     -- SRM 检测 p 值
    srm_detected    BOOLEAN DEFAULT FALSE,             -- 是否检测到 SRM
    paradox_detected BOOLEAN DEFAULT FALSE,            -- 是否检测到辛普森悖论
    -- 元信息
    analysis_time   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    info_fraction   DECIMAL(4,3),                      -- 当前信息分数（序贯检验用）
    test_method     VARCHAR(32),                       -- 使用的检验方法
    UNIQUE KEY uk_exp_metric_time (experiment_id, metric_name, analysis_time),
    INDEX idx_experiment (experiment_id),
    INDEX idx_significant (is_significant)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**关键设计决策说明：**

| 决策 | 选择 | 原因 |
|------|------|------|
| user_assignments 使用 UNIQUE KEY | (user_id, experiment_id) | 保证同一用户同一实验只有一条分组记录，避免重复分配 |
| hash_bucket 字段独立存储 | 调试和校验 | 当 SRM 检测到偏差时，可追溯哈希桶分布是否均匀 |
| 指标分 RATIO/CONTINUOUS/COUNT | 不同统计检验方法 | 比率型用 Z 检验比例，连续型用 t 检验，计数型用泊松检验 |
| experiment_results 按时间快照 | 支持序贯检验 | 每次分析产生一条记录，可回溯信息分数变化过程 |
| metrics.is_primary 区分 | 差异化多重比较校正 | 核心指标用严格校正（Bonferroni），辅助指标用宽松校正（FDR） |

## 请先独立思考（限时 30 分钟）

1. 如何保证分组稳定性？用什么哈希算法？为什么不能只用 user_id 哈希？
2. 如何防止偷看导致的假阳性？序贯检验是什么？OBF vs Pocock 边界？
3. 多重比较如何校正？Bonferroni vs FDR？各适用什么场景？
4. 样本量如何计算？MDE 如何确定？MDE 和样本量的关系是什么？

---

## 设计解析

### 端到端实验流水线

```python
class ExperimentPipeline:
    """A/B 实验完整流水线"""

    def create_experiment(self, config):
        """创建实验"""
        # 1. 校验配置
        self.validate_config(config)
        
        # 2. 计算所需样本量和时长
        sample_size = self.calculator.calculate(
            baseline_rate=config.baseline_rate,
            mde=config.mde,
            alpha=config.alpha,
            power=config.power
        )
        duration_days = self.calculator.recommend_duration(
            config.baseline_rate, config.mde, config.daily_traffic
        )
        
        # 3. 初始化实验
        experiment = Experiment(
            id=uuid4(),
            name=config.name,
            variants=config.variants,  # [{"name": "control", "ratio": 0.5}, ...]
            metrics=config.metrics,
            sample_size_per_group=sample_size,
            duration_days=duration_days,
            status="RUNNING",
            start_date=today(),
            end_date=today() + timedelta(days=duration_days)
        )
        
        # 4. 注册分组规则（新用户自动分配）
        self.assignment_service.register(experiment)
        
        # 5. 启动指标收集
        self.metric_collector.start(experiment)
        
        return experiment

    def get_result(self, experiment_id):
        """获取实验结果"""
        experiment = self.get_experiment(experiment_id)
        
        # 1. 收集各组的指标数据
        control_data = self.metric_collector.get_data(experiment, "control")
        treatment_data = self.metric_collector.get_data(experiment, "treatment")
        
        # 2. 检查样本量是否足够
        current_power = self.compute_power(control_data, treatment_data, experiment.mde)
        if current_power < 0.8:
            return ExperimentResult(
                status="INSUFFICIENT_POWER",
                current_power=current_power,
                days_remaining=self.estimate_remaining_days(experiment, current_power)
            )
        
        # 3. 统计检验（使用序贯检验防止偷看）
        for metric in experiment.metrics:
            test_result = self.sequential_tester.check(
                control=control_data[metric],
                treatment=treatment_data[metric],
                current_n=len(control_data),
                total_n=experiment.sample_size_per_group * 2
            )
            metric_result = MetricResult(
                metric=metric,
                is_significant=test_result.is_significant,
                p_value=test_result.p_value,
                effect_size=test_result.effect_size,
                confidence_interval=test_result.ci
            )
        
        # 4. 多重比较校正
        all_p_values = [r.p_value for r in metric_results]
        corrected = self.multiple_comparator.correct(all_p_values, method="bonferroni")
        for i, result in enumerate(metric_results):
            result.is_significant_after_correction = corrected[i]
        
        # 5. 辛普森悖论检测
        paradox = self.paradox_detector.detect(experiment_id)
        
        # 6. 组装结果
        return ExperimentResult(
            experiment_id=experiment_id,
            status="CONCLUSIVE" if any(corrected) else "INCONCLUSIVE",
            metric_results=metric_results,
            paradox_detected=paradox,
            sample_ratio_mismatch=self.check_srm(experiment)
        )
```

### 分组：服务端确定性哈希

**方案对比：**

| 方案 | 延迟 | 一致性 | 换设备问题 | 存储成本 |
|------|------|-------|-----------|---------|
| 前端哈希（Cookie） | 0ms | 弱（清Cookie丢失） | 5%分组丢失 | 0 |
| 服务端哈希（Redis） | 1ms | 强 | 0%丢失 | 低 |
| 服务端哈希（DB） | 5ms | 最强 | 0%丢失 | 中 |

选择**服务端哈希 + DB 持久化 + Redis 缓存**：

```python
class ExperimentAssigner:
    def get_assignment(self, user_id, experiment_id):
        """获取用户在实验中的分组"""
        # 1. 查 Redis 缓存
        cached = self.redis.get(f"ab:{user_id}:{experiment_id}")
        if cached:
            return cached  # 1ms
        
        # 2. 查数据库
        record = self.db.query_one("""
            SELECT group_name FROM experiment_assignments
            WHERE user_id = %s AND experiment_id = %s
        """, user_id, experiment_id)
        
        if record:
            self.redis.setex(f"ab:{user_id}:{experiment_id}", 3600, record.group_name)
            return record.group_name
        
        # 3. 新用户：确定性哈希分配
        group = self.assign(user_id, experiment_id)
        self.db.execute("""
            INSERT INTO experiment_assignments (user_id, experiment_id, group_name, assigned_at)
            VALUES (%s, %s, %s, NOW())
        """, user_id, experiment_id, group)
        self.redis.setex(f"ab:{user_id}:{experiment_id}", 3600, group)
        return group

    def assign(self, user_id, experiment_id):
        """确定性分组：MurmurHash3"""
        # 用 user_id + experiment_id 保证跨实验独立性
        hash_val = mmh3.hash(f"{user_id}:{experiment_id}") % 10000
        
        # 根据实验配置的流量比例分配
        experiment = self.get_experiment(experiment_id)
        cumulative = 0
        for variant in experiment.variants:
            cumulative += variant.ratio * 10000
            if hash_val < cumulative:
                return variant.name
        
        return experiment.variants[-1].name  # 兜底

    def assign_with_sticky_bucket(self, user_id, experiment_id):
        """粘性桶分配：用户换设备后仍能回到原组"""
        # 1. 先查粘性桶记录（跨实验维度的"桶号"，比单实验分组更稳定）
        sticky = self.db.query_one("""
            SELECT bucket FROM user_sticky_buckets
            WHERE user_id = %s AND bucket_type = %s
        """, user_id, "experiment_layer")
        if sticky:
            # 基于粘性桶号 + experiment_id 计算分组
            hash_val = mmh3.hash(f"{sticky.bucket}:{experiment_id}") % 10000
            return self._map_to_variant(experiment_id, hash_val)

        # 2. 新用户：生成粘性桶号（基于 user_id 的全局哈希）
        bucket = mmh3.hash(f"{user_id}") % 10000
        self.db.execute("""
            INSERT INTO user_sticky_buckets (user_id, bucket_type, bucket)
            VALUES (%s, 'experiment_layer', %s)
        """, user_id, bucket)
        hash_val = mmh3.hash(f"{bucket}:{experiment_id}") % 10000
        return self._map_to_variant(experiment_id, hash_val)

    def _map_to_variant(self, experiment_id, hash_val):
        """将哈希值映射到变体"""
        experiment = self.get_experiment(experiment_id)
        cumulative = 0
        for variant in experiment.variants:
            cumulative += variant.ratio * 10000
            if hash_val < cumulative:
                return variant.name
        return experiment.variants[-1].name
```

**粘性桶 vs 普通哈希对比：**

| 维度 | 普通哈希 | 粘性桶 |
|------|---------|-------|
| 分组确定性 | 同一 user_id+experiment_id → 同一组 | 同一粘性桶+experiment_id → 同一组 |
| 换设备恢复 | 需查 DB/Redis 才能恢复 | 知道桶号即可恢复（桶号更短，更容易缓存） |
| 实验修改后 | 新实验 ID → 新哈希 → 新分组 | 新实验 ID + 同桶号 → 可控分组 |
| 调整流量比例 | 原分组不变，新用户按新比例 | 可通过桶重新分配平滑调整 |

**一致性哈希在 A/B 测试中的应用：**

当需要调整实验流量比例（如从 50/50 调到 60/40），如果直接改哈希映射规则，已分配用户的分组会发生变化。一致性哈希可以缓解这个问题：

```python
class ConsistentHashAssigner:
    """一致性哈希分组：支持流量调整时保持原分组稳定"""

    def __init__(self, num_buckets=10000, num_virtual_nodes=100):
        self.num_buckets = num_buckets
        self.num_virtual_nodes = num_virtual_nodes
        # 每个变体占据若干虚拟节点环上的位置
        self.ring = []  # [(position_on_ring, variant_name)]

    def setup_variants(self, experiment_id, variants):
        """为实验变体设置一致性哈希环"""
        for variant in variants:
            for i in range(self.num_virtual_nodes):
                # 每个变体有 num_virtual_nodes 个虚拟节点
                # 虚拟节点数量比例 ≈ 流量比例（更多节点 = 更大概率被命中）
                key = f"{experiment_id}:{variant.name}:vn{i}"
                position = mmh3.hash(key) % self.num_buckets
                self.ring.append((position, variant.name))
        self.ring.sort(key=lambda x: x[0])

    def assign(self, user_id, experiment_id):
        """一致性哈希分配"""
        user_hash = mmh3.hash(f"{user_id}:{experiment_id}") % self.num_buckets
        # 在环上找到第一个 >= user_hash 的虚拟节点
        for position, variant in self.ring:
            if position >= user_hash:
                return variant
        # 环回开头
        return self.ring[0][1]

    def adjust_traffic(self, experiment_id, new_variants):
        """调整流量比例：只影响新用户，已分配用户不受影响"""
        # 1. 记录旧配置
        old_config = self.get_variant_config(experiment_id)
        # 2. 重建哈希环
        self.setup_variants(experiment_id, new_variants)
        # 3. 已分配用户保持原分组（DB 优先）
        # 新用户按新环分配
```

**分组完整性校验：**

```python
class AssignmentValidator:
    """分组完整性校验：确保哈希分配的正确性"""

    def validate_distribution(self, experiment_id, variants):
        """验证哈希桶分布是否均匀"""
        # 模拟 10000 个桶的分配结果
        bucket_counts = {}
        for bucket in range(10000):
            # 模拟每个桶对应的分组
            cumulative = 0
            for variant in variants:
                cumulative += variant.ratio * 10000
                if bucket < cumulative:
                    bucket_counts[variant.name] = bucket_counts.get(variant.name, 0) + 1
                    break

        # 检验实际 vs 预期
        for variant in variants:
            expected = 10000 * variant.ratio
            actual = bucket_counts.get(variant.name, 0)
            deviation = abs(actual - expected) / expected
            if deviation > 0.01:  # 偏差超过 1%
                self.alert(f"哈希分配偏差超限: {variant.name} "
                          f"预期 {expected:.0f} 实际 {actual} 偏差 {deviation:.2%}")

    def validate_cross_experiment_independence(self, user_id, experiment_ids):
        """验证跨实验分组独立性"""
        # 同一用户在不同实验中不应总是同一组
        groups = []
        for exp_id in experiment_ids:
            hash_val = mmh3.hash(f"{user_id}:{exp_id}") % 10000
            group = "control" if hash_val < 5000 else "treatment"
            groups.append(group)

        # 如果所有实验都在同一组 → 独立性有问题
        if len(set(groups)) == 1:
            # 检查是否系统性的（统计检测）
            correlation = self.compute_group_correlation(experiment_ids)
            if correlation > 0.15:  # 超过自然相关度阈值
                self.alert(f"跨实验分组相关性过高: {correlation:.2%}")
```

**为什么用 user_id + experiment_id？**

如果只用 user_id → 用户 A 在所有实验中 hash_val 都相同 → 总在同一组 → 实验间干扰（如 A 同时看到新按钮+新推荐 → 无法区分哪个改动导致了效果变化）。

**SRM（Sample Ratio Mismatch）检测：**

```python
class SRMChecker:
    def check(self, experiment):
        """检测实验组和对照组的样本比例是否符合预期"""
        counts = self.db.query("""
            SELECT group_name, COUNT(*) as n
            FROM experiment_assignments
            WHERE experiment_id = %s
            GROUP BY group_name
        """, experiment.id)
        
        expected_ratio = experiment.variants[0].ratio  # 如 0.5
        control_n = counts.get("control", 0)
        treatment_n = counts.get("treatment", 0)
        
        # 卡方检验
        expected_control = (control_n + treatment_n) * expected_ratio
        expected_treatment = (control_n + treatment_n) * (1 - expected_ratio)
        
        chi2 = ((control_n - expected_control)**2 / expected_control + 
                (treatment_n - expected_treatment)**2 / expected_treatment)
        
        p_value = 1 - chi2.cdf(chi2, df=1)
        
        if p_value < 0.01:
            # SRM 检测到！说明分组有偏差 → 实验结果不可信
            self.alert(f"SRM detected: control={control_n}, treatment={treatment_n}, "
                      f"expected ratio={expected_ratio}, p={p_value:.4f}")
            return True
        return False
```

### 样本量计算与时长推荐

```python
class SampleSizeCalculator:
    def calculate(self, baseline_rate, mde, alpha=0.05, power=0.8):
        """
        baseline_rate: 基线转化率（如当前点击率 5%）
        mde: 最小可检测效果（如期望提升 10%，即从5%到5.5%）
        """
        p1 = baseline_rate
        p2 = baseline_rate * (1 + mde)
        
        z_alpha = 1.96   # alpha=0.05 双侧
        z_beta = 0.84    # power=0.8
        
        n = ((z_alpha * math.sqrt(2 * p1 * (1-p1)) + 
              z_beta * math.sqrt(p1*(1-p1) + p2*(1-p2))) ** 2) / (p2-p1) ** 2
        
        return math.ceil(n)

    def recommend_duration(self, baseline_rate, mde, daily_traffic):
        n_per_group = self.calculate(baseline_rate, mde)
        total_n = n_per_group * 2
        days = math.ceil(total_n / daily_traffic)
        return days
```

**MDE 与样本量的关系（基线 5%）：**

| MDE | 效果 | 每组样本量 | 总样本量 | 日均1万用户所需天数 |
|-----|------|-----------|---------|-----------------|
| 5% | 5%→5.25% | 124000 | 248000 | 25 天 |
| 10% | 5%→5.5% | 31000 | 62000 | 7 天 |
| 20% | 5%→6% | 7800 | 15600 | 2 天 |
| 50% | 5%→7.5% | 1500 | 3000 | < 1 天 |

**核心洞察：MDE 越小，需要的样本量越大。** 微小的优化（如按钮颜色微调）需要很长时间的实验。

### 防止偷看：序贯检验（完整实现）

序贯检验的核心思想：允许在实验过程中多次查看数据，但通过数学上严格的边界函数控制整体假阳性率。下面给出三种主流方法的完整实现：Alpha 消费函数、始终有效 p 值、以及 OBF/Pocock 边界。

**O'Brien-Fleming vs Pocock 边界对比：**

| 检验方法 | 早期阈值 | 后期阈值 | 特点 |
|---------|---------|---------|------|
| OBF | 极严格（8.0） | 标准（1.96） | 早期几乎不可能停止，后期回归正常 |
| Pocock | 一致严格（2.36） | 一致严格（2.36） | 每次查看阈值相同，但比 OBF 宽松 |
| 固定样本 | 不允许查看 | 1.96 | 只能看一次，否则假阳性飙升 |
| Alpha 消费 | 精确计算 | 精确计算 | 连续型 alpha 分配，最灵活 |
| 始终有效 p 值 | 动态 | 动态 | 无需预设分析次数，随时查看 |

```python
class SequentialTester:
    def check_significant(self, control, treatment, current_n, total_n):
        """序贯检验：允许随时查看，但控制假阳性率"""
        info_fraction = current_n / total_n
        z_threshold = self.obf_boundary(info_fraction)
        z = self.compute_z_statistic(control, treatment)
        
        return SignificanceResult(
            is_significant=abs(z) > z_threshold,
            z_score=z,
            z_threshold=z_threshold,
            info_fraction=info_fraction,
            can_early_stop=abs(z) > z_threshold and info_fraction > 0.5
        )

    def obf_boundary(self, info_fraction):
        """O'Brien-Fleming 边界值"""
        if info_fraction < 0.25:
            return 8.0    # 极严格：几乎不可能早期显著
        elif info_fraction < 0.50:
            return 4.0
        elif info_fraction < 0.75:
            return 2.5
        else:
            return 1.96   # 最后阶段回归标准阈值

    def pocock_boundary(self, info_fraction):
        """Pocock 边界值：每次查看阈值相同"""
        # Pocock 边界在所有分析时间点使用相同的阈值
        # 对于 alpha=0.05, 5 次分析，Pocock 边界约 2.36
        # 不同分析次数的 Pocock 常数（已预先计算）
        pocock_constants = {
            2: 2.18,   # 2 次分析
            3: 2.29,   # 3 次分析
            4: 2.36,   # 4 次分析
            5: 2.41,   # 5 次分析
            10: 2.56,  # 10 次分析
        }
        # 根据实验配置获取分析次数
        num_analyses = self.config.get("num_analyses", 5)
        return pocock_constants.get(num_analyses, 2.41)

    def compute_z_statistic(self, control, treatment):
        """计算 Z 统计量（双样本比例检验或 t 检验）"""
        if control.metric_type == "RATIO":
            # 比例检验：转化率等
            p1 = treatment.sum / treatment.n
            p2 = control.sum / control.n
            p_pool = (treatment.sum + control.sum) / (treatment.n + control.n)
            se = math.sqrt(p_pool * (1 - p_pool) * (1/treatment.n + 1/control.n))
            return (p1 - p2) / se if se > 0 else 0
        else:
            # 连续型指标：t 检验
            mean_diff = treatment.mean - control.mean
            se = math.sqrt(treatment.var / treatment.n + control.var / control.n)
            return mean_diff / se if se > 0 else 0


class SequentialTestEngine:
    """完整的序贯检验引擎：支持 OBF 和 Pocock 两种边界"""

    def __init__(self, alpha=0.05, max_analyses=5, boundary_type="obf"):
        self.alpha = alpha
        self.max_analyses = max_analyses
        self.boundary_type = boundary_type
        self.tester = SequentialTester()

    def analyze(self, experiment, metric_name, current_data):
        """执行一次序贯检验分析"""
        # 1. 计算当前信息分数
        total_planned = experiment.sample_size_per_group * 2
        current_n = current_data.control.n + current_data.treatment.n
        info_fraction = current_n / total_planned

        # 2. 计算边界
        if self.boundary_type == "obf":
            z_boundary = self.tester.obf_boundary(info_fraction)
        else:
            z_boundary = self.tester.pocock_boundary(info_fraction)

        # 3. 计算检验统计量
        z = self.tester.compute_z_statistic(current_data.control, current_data.treatment)

        # 4. 判定
        is_significant = abs(z) > z_boundary
        can_early_stop = is_significant and info_fraction >= 0.5

        # 5. 计算 p 值（考虑序贯校正）
        # 使用的 p 值是边界值对应的 p 值，而非原始 p 值
        p_value = 2 * (1 - norm.cdf(abs(z)))  # 原始双尾 p 值
        adjusted_p_value = 2 * (1 - norm.cdf(z_boundary))  # 边界对应的 p 值

        return SequentialTestResult(
            is_significant=is_significant,
            can_early_stop=can_early_stop,
            z_score=z,
            z_boundary=z_boundary,
            info_fraction=info_fraction,
            p_value=p_value,
            adjusted_p_value=adjusted_p_value,
            recommendation=self._make_recommendation(
                is_significant, can_early_stop, info_fraction
            )
        )

    def _make_recommendation(self, is_significant, can_early_stop, info_fraction):
        """给出决策建议"""
        if can_early_stop and is_significant:
            return "REJECT_H0_EARLY"  # 可以提前拒绝原假设
        elif info_fraction >= 1.0 and is_significant:
            return "REJECT_H0"        # 实验结束时拒绝原假设
        elif info_fraction >= 1.0 and not is_significant:
            return "FAIL_TO_REJECT"   # 实验结束时无法拒绝原假设
        else:
            return "CONTINUE"         # 继续实验
```

### 防止偷看：Always-Valid p 值与 Alpha 消耗函数

OBF 和 Pocock 边界是经典的分组序贯检验方法，但它们要求预先指定分析次数和时间点。**Always-valid p 值**（也称"anytime-valid p-value"）是更现代的方法：无论何时查看，p 值都保持有效性，无需预指定分析计划。

**核心数学原理：**

Always-valid p 值基于**检验鞅（test martingale）**理论。定义累积和过程：

$$S_n = \sum_{i=1}^{n} (X_i^{treatment} - X_i^{control})$$

构造检验统计量（似然比鞅）：

$$M_n = \prod_{i=1}^{n} \exp\left(\lambda \cdot \Delta_i - \frac{\lambda^2 \sigma_i^2}{2}\right)$$

其中 $\Delta_i$ 是第 i 个观测的处理效应差，$\sigma_i^2$ 是方差，$\lambda$ 是最优化参数。

**Always-valid p 值定义为：**

$$p_n^{valid} = \frac{1}{M_n}$$

根据 Ville 不等式，对于任意 $\alpha$，$P(\exists n: p_n^{valid} \leq \alpha) \leq \alpha$，即假阳性率始终被控制。

**Alpha 消耗函数：**

Alpha 消耗函数 $\alpha^*(t)$ 将总显著性水平 $\alpha$ 随信息分数 $t$ 逐步"消耗"，确保累计消耗不超过 $\alpha$：

| 消耗函数 | 公式 | 特点 |
|---------|------|------|
| OBF 型 | $\alpha^*(t) = 2(1 - \Phi(z_{\alpha/2} / \sqrt{t}))$ | 早期极保守，后期宽松 |
| Pocock 型 | $\alpha^*(t) = \alpha \cdot \ln(1 + (e-1) \cdot t)$ | 均匀消耗 |
| Kim-DeMets 线性 | $\alpha^*(t) = \alpha \cdot t$ | 线性消耗 |
| Kim-DeMets 幂 | $\alpha^*(t) = \alpha \cdot t^\rho$（$\rho > 0$） | $\rho$ 越大越保守 |

其中 $\Phi$ 是标准正态 CDF，$t$ 是信息分数（当前样本量 / 计划总样本量）。

**第 k 次分析的边界值计算：**

$$z_k = \Phi^{-1}\left(1 - \frac{\alpha^*(t_k) - \alpha^*(t_{k-1})}{2}\right)$$

即每次分析消耗的 alpha 量决定了该次的 z 边界。

```python
import math
from scipy.stats import norm

class AlphaSpendingFunction:
    """Alpha 消耗函数：控制序贯检验的假阳性率"""

    def __init__(self, alpha=0.05, spending_type="obf"):
        self.alpha = alpha
        self.spending_type = spending_type
        self._prev_spent = 0.0  # 上一次分析时已消耗的 alpha

    def spend(self, info_fraction):
        """
        计算在当前信息分数下应消耗的 alpha 量，并返回本次分析的 z 边界值

        info_fraction: 当前信息分数 t ∈ (0, 1]，即 current_n / planned_n
        返回: (z_boundary, incremental_alpha, cumulative_alpha)
        """
        if info_fraction <= 0:
            return (float('inf'), 0.0, 0.0)

        # 计算累计消耗的 alpha
        if self.spending_type == "obf":
            # O'Brien-Fleming 型消耗函数
            # alpha*(t) = 2 * (1 - Phi(z_{alpha/2} / sqrt(t)))
            z_alpha_half = norm.ppf(1 - self.alpha / 2)
            cumulative_alpha = 2 * (1 - norm.cdf(z_alpha_half / math.sqrt(info_fraction)))
        elif self.spending_type == "pocock":
            # Pocock 型消耗函数
            # alpha*(t) = alpha * ln(1 + (e-1) * t)
            cumulative_alpha = self.alpha * math.log(1 + (math.e - 1) * info_fraction)
        elif self.spending_type == "linear":
            # Kim-DeMets 线性消耗
            cumulative_alpha = self.alpha * info_fraction
        elif self.spending_type == "power":
            # Kim-DeMets 幂消耗（rho=2，比 OBF 稍宽松）
            rho = 2
            cumulative_alpha = self.alpha * (info_fraction ** rho)
        else:
            raise ValueError(f"Unknown spending type: {self.spending_type}")

        # 保证单调递增
        cumulative_alpha = max(cumulative_alpha, self._prev_spent)
        # 不超过总 alpha
        cumulative_alpha = min(cumulative_alpha, self.alpha)

        # 本次增量消耗
        incremental_alpha = cumulative_alpha - self._prev_spent

        if incremental_alpha <= 0:
            # 没有新的 alpha 可消耗 → 边界无穷大（不可能显著）
            return (float('inf'), 0.0, cumulative_alpha)

        # 计算对应的 z 边界值
        # incremental_alpha = 2 * (1 - Phi(z_k)) → z_k = Phi^{-1}(1 - incremental_alpha/2)
        z_boundary = norm.ppf(1 - incremental_alpha / 2)

        # 更新已消耗量
        self._prev_spent = cumulative_alpha

        return (z_boundary, incremental_alpha, cumulative_alpha)

    def reset(self):
        """重置消耗状态（新实验开始时调用）"""
        self._prev_spent = 0.0


class AlwaysValidPValue:
    """Always-valid p 值：基于检验鞅，支持任意时间查看"""

    def __init__(self, alpha=0.05, spending_type="obf"):
        self.alpha = alpha
        self.spender = AlphaSpendingFunction(alpha, spending_type)
        self.martingale = 1.0  # 检验鞅初始值
        self._lambda = None     # 最优 lambda 参数

    def update(self, control_new_data, treatment_new_data, total_planned_n):
        """
        增量更新：每次有新数据时调用

        control_new_data: 对照组新增数据 {n, sum, sum_sq}
        treatment_new_data: 实验组新增数据 {n, sum, sum_sq}
        total_planned_n: 计划总样本量
        """
        # 1. 计算新增观测的处理效应差
        n_new = control_new_data['n'] + treatment_new_data['n']
        if n_new == 0:
            return self._current_result(0)

        control_mean = control_new_data['sum'] / control_new_data['n'] if control_new_data['n'] > 0 else 0
        treatment_mean = treatment_new_data['sum'] / treatment_new_data['n'] if treatment_new_data['n'] > 0 else 0
        delta = treatment_mean - control_mean

        # 2. 估计方差
        control_var = (control_new_data['sum_sq'] / control_new_data['n'] - control_mean**2) if control_new_data['n'] > 1 else 1.0
        treatment_var = (treatment_new_data['sum_sq'] / treatment_new_data['n'] - treatment_mean**2) if treatment_new_data['n'] > 1 else 1.0
        sigma_sq = treatment_var / treatment_new_data['n'] + control_var / control_new_data['n'] if control_new_data['n'] > 0 and treatment_new_data['n'] > 0 else 1.0

        # 3. 选择最优 lambda（基于当前方差估计）
        if self._lambda is None:
            # 使用预期效应量的先验估计初始化
            # lambda = delta_hat / sigma^2（MLE 估计）
            self._lambda = delta / sigma_sq if sigma_sq > 0 else 0.01

        # 4. 更新检验鞅
        # M_n = M_{n-1} * exp(lambda * delta - lambda^2 * sigma^2 / 2)
        log_multiplier = self._lambda * delta - (self._lambda ** 2 * sigma_sq) / 2
        self.martingale *= math.exp(log_multiplier)

        # 5. 计算 always-valid p 值
        p_valid = 1.0 / self.martingale if self.martingale > 0 else 1.0
        p_valid = min(p_valid, 1.0)  # p 值不超过 1

        # 6. 同时用 alpha 消耗函数计算边界
        current_n = control_new_data['n'] + treatment_new_data['n']
        info_fraction = min(current_n / total_planned_n, 1.0)
        z_boundary, inc_alpha, cum_alpha = self.spender.spend(info_fraction)

        # 7. 计算原始 z 统计量
        z_score = delta / math.sqrt(sigma_sq) if sigma_sq > 0 else 0

        return AlwaysValidResult(
            p_valid=p_valid,
            is_significant_martingale=p_valid <= self.alpha,
            is_significant_spending=abs(z_score) > z_boundary,
            z_score=z_score,
            z_boundary=z_boundary,
            info_fraction=info_fraction,
            martingale=self.martingale,
            cumulative_alpha_spent=cum_alpha,
            recommendation=self._recommend(p_valid, info_fraction)
        )

    def _recommend(self, p_valid, info_fraction):
        """决策建议"""
        if p_valid <= self.alpha:
            if info_fraction >= 0.5:
                return "REJECT_H0_EARLY"  # 可以提前拒绝
            else:
                return "REJECT_H0_EARLY_CAUTION"  # 显著但样本还少，谨慎
        elif info_fraction >= 1.0:
            return "FAIL_TO_REJECT"
        else:
            return "CONTINUE"

    def _current_result(self, info_fraction):
        return AlwaysValidResult(
            p_valid=1.0, is_significant_martingale=False,
            is_significant_spending=False, z_score=0, z_boundary=float('inf'),
            info_fraction=info_fraction, martingale=self.martingale,
            cumulative_alpha_spent=self.spender._prev_spent,
            recommendation="CONTINUE"
        )
```

**Alpha 消耗函数在不同信息分数下的边界值对比（alpha=0.05）：**

| 信息分数 t | OBF 型 z 边界 | Pocock 型 z 边界 | 线性型 z 边界 | 幂型(ρ=2) z 边界 |
|-----------|-------------|----------------|-------------|----------------|
| 0.1 | 6.36 | 2.44 | 3.44 | 5.07 |
| 0.2 | 4.50 | 2.44 | 2.86 | 3.61 |
| 0.3 | 3.67 | 2.44 | 2.57 | 3.00 |
| 0.5 | 2.84 | 2.44 | 2.24 | 2.41 |
| 0.7 | 2.40 | 2.44 | 2.04 | 2.11 |
| 1.0 | 1.96 | 2.44 | 1.96 | 1.96 |

**解读：** OBF 型在早期极为严格（t=0.1 时 z=6.36），几乎不可能早期停止；Pocock 型每次查看阈值相同（2.44），更宽松但总体功效较低；线性型介于两者之间；幂型(ρ=2)接近 OBF 但稍宽松。实践中推荐 OBF 型——产品经理很难在早期就做出正确决策，严格边界反而保护了他们。

#### 防偷看的工程化保障

仅靠统计方法不够，还需要工程层面的防护——防止人为绕过统计规则：

```python
from datetime import datetime, timedelta


class NoPeekingGuard:
    """
    防偷看守护：在平台层面强制执行序贯检验规则

    核心原则：统计方法防止数学上的假阳性，
    工程防护防止人为绕过统计规则。
    """

    def __init__(self, spending_engine):
        self.engine = spending_engine
        self._view_count = 0
        self._max_daily_views = 1       # 每天最多查看一次
        self._last_view_date = None
        self._early_stop_triggered = False

    def can_view_result(self, experiment_id: str) -> dict:
        """
        检查是否允许查看实验结果

        规则：
        1. 每天最多查看一次（防止高频偷看）
        2. 实验最短天数未过，只显示"实验进行中"
        3. 已触发提前停止，锁定结果
        """
        today = datetime.now().date()
        experiment = self._get_experiment(experiment_id)

        # 规则 1：每日查看限制
        if self._last_view_date == today:
            return {
                "allowed": False,
                "reason": "DAILY_LIMIT",
                "message": "今日已查看过实验结果，明天再来",
                "next_view_time": datetime(today + timedelta(days=1)),
            }

        # 规则 2：最短天数限制
        days_running = (today - experiment.started_at.date()).days
        if days_running < experiment.min_duration_days:
            return {
                "allowed": False,
                "reason": "MIN_DURATION",
                "message": (f"实验至少需运行 {experiment.min_duration_days} 天，"
                           f"当前仅 {days_running} 天"),
                "progress": days_running / experiment.min_duration_days,
            }

        # 规则 3：提前停止锁定
        if self._early_stop_triggered:
            return {
                "allowed": True,
                "reason": "EARLY_STOP_FINAL",
                "message": "实验已因早期显著性停止，结果已锁定",
            }

        return {"allowed": True, "reason": "OK"}

    def view_result(self, experiment_id: str, current_data) -> dict:
        """
        查看实验结果（受防护规则约束）

        返回内容根据实验阶段有差异：
        - 信息分数 < 0.5：只显示"方向性参考"，不显示 p 值
        - 信息分数 >= 0.5：显示始终有效 p 值和序贯检验结果
        - 信息分数 = 1.0：显示完整结果
        """
        view_check = self.can_view_result(experiment_id)
        if not view_check["allowed"]:
            return view_check

        # 执行序贯检验
        result = self.engine.analyze(
            current_data.control, current_data.treatment,
            current_data.total_n, current_data.planned_n,
        )

        # 记录查看
        self._view_count += 1
        self._last_view_date = datetime.now().date()

        info_frac = result.info_fraction

        if info_frac < 0.5:
            # 早期：只给方向性提示
            return {
                "allowed": True,
                "display_level": "DIRECTIONAL_ONLY",
                "direction": "POSITIVE" if result.z_score > 0 else "NEGATIVE",
                "message": "实验进行中，当前仅显示效果方向，不显示 p 值",
                "info_fraction": info_frac,
            }
        elif info_frac < 1.0:
            # 中期：显示始终有效 p 值
            return {
                "allowed": True,
                "display_level": "SEQUENTIAL",
                "is_significant": result.is_significant,
                "can_early_stop": result.can_early_stop,
                "always_valid_p_value": result.p_value,
                "info_fraction": info_frac,
                "message": ("可以提前停止" if result.can_early_stop
                           else "继续实验"),
            }
        else:
            # 完整结果
            return {
                "allowed": True,
                "display_level": "FULL",
                "is_significant": result.is_significant,
                "p_value": result.p_value,
                "effect_size": result.effect_size,
                "confidence_interval": result.confidence_interval,
                "info_fraction": 1.0,
            }

    def _get_experiment(self, experiment_id):
        """获取实验配置（从 DB 或缓存）"""
        # 实际实现中从数据库查询
        pass
```

**防偷看守护的三层防御：**

| 防御层 | 机制 | 效果 |
|--------|------|------|
| 统计层 | Alpha 消耗函数 / 始终有效 p 值 | 数学上保证假阳性率 ≤ alpha |
| 展示层 | 信息分数 < 0.5 不显示 p 值 | 防止"看到 p=0.04 就想停"的冲动 |
| 操作层 | 每日查看限制 + 最短天数限制 | 防止高频查看 + 强制完整周期效应 |

### 多重比较校正

```python
class MultipleComparisonCorrector:
    def correct(self, p_values, method="bonferroni"):
        m = len(p_values)
        
        if method == "bonferroni":
            corrected_alpha = 0.05 / m
            return [p < corrected_alpha for p in p_values]
        
        elif method == "fdr":
            sorted_indices = sorted(range(m), key=lambda i: p_values[i])
            sorted_pvals = [p_values[i] for i in sorted_indices]
            for k in range(m, 0, -1):
                if sorted_pvals[k-1] <= k / m * 0.05:
                    result = [False] * m
                    for i in range(k):
                        result[sorted_indices[i]] = True
                    return result
            return [False] * m

    def bonferroni_holm(self, p_values):
        """Bonferroni-Holm 逐步校正：比 Bonferroni 功效更高"""
        m = len(p_values)
        # 1. 将 p 值从小到大排序
        sorted_indices = sorted(range(m), key=lambda i: p_values[i])
        sorted_pvals = [p_values[i] for i in sorted_indices]

        results = [False] * m
        # 2. 逐步检验
        for k in range(m):
            # 第 k 小的 p 值与 (alpha / (m - k)) 比较
            # 注意：第 1 小与 alpha/m 比较，第 2 小与 alpha/(m-1) 比较...
            adjusted_threshold = 0.05 / (m - k)
            if sorted_pvals[k] <= adjusted_threshold:
                results[sorted_indices[k]] = True
            else:
                # 一旦有一个不显著，后续都不显著
                break

        return results

    def benjamini_hochberg(self, p_values, q=0.05):
        """Benjamini-Hochberg FDR 校正：控制错误发现率"""
        m = len(p_values)
        sorted_indices = sorted(range(m), key=lambda i: p_values[i])
        sorted_pvals = [p_values[i] for i in sorted_indices]

        # 找到最大的 k，使得 p_(k) <= k/m * q
        max_k = 0
        for k in range(1, m + 1):
            if sorted_pvals[k - 1] <= k / m * q:
                max_k = k

        results = [False] * m
        for i in range(max_k):
            results[sorted_indices[i]] = True
        return results

    def auto_select_method(self, metrics):
        """自动选择多重比较校正方法"""
        primary_count = sum(1 for m in metrics if m.is_primary)
        total_count = len(metrics)

        if total_count <= 3:
            # 指标很少，无需校正
            return "NONE"
        elif primary_count <= 5:
            # 核心指标少，用 Bonferroni-Holm（比 Bonferroni 功效高）
            return "BONFERRONI_HOLM"
        elif total_count <= 15:
            # 中等数量指标，核心用 Bonferroni，辅助用 FDR
            return "HYBRID"
        else:
            # 指标很多，用 FDR
            return "FDR"

    def hybrid_correction(self, p_values, metrics):
        """混合校正：核心指标用 Bonferroni-Holm，辅助指标用 FDR"""
        primary_indices = [i for i, m in enumerate(metrics) if m.is_primary]
        secondary_indices = [i for i, m in enumerate(metrics) if not m.is_primary]

        results = [False] * len(p_values)

        # 核心指标：Bonferroni-Holm（严格）
        if primary_indices:
            primary_pvals = [p_values[i] for i in primary_indices]
            primary_results = self.bonferroni_holm(primary_pvals)
            for idx, is_sig in zip(primary_indices, primary_results):
                results[idx] = is_sig

        # 辅助指标：FDR（宽松）
        if secondary_indices:
            secondary_pvals = [p_values[i] for i in secondary_indices]
            secondary_results = self.benjamini_hochberg(secondary_pvals)
            for idx, is_sig in zip(secondary_indices, secondary_results):
                results[idx] = is_sig

        return results
```

**不同校正方法的效果对比（10 个指标，3 个核心 + 7 个辅助）：**

| 方法 | 核心指标阈值 | 辅助指标阈值 | 整体假阳性率 | 统计功效 |
|------|-----------|-----------|-----------|---------|
| 不校正 | 0.05 | 0.05 | ~40% | 高 |
| Bonferroni | 0.005 | 0.005 | <5% | 很低 |
| Bonferroni-Holm | 0.005~0.05（递增） | 0.005~0.05 | <5% | 中 |
| FDR | 动态 | 动态（~0.01-0.05） | <5% FDR | 较高 |
| 混合校正 | Bonferroni-Holm | FDR | <5%（核心）/ <5% FDR（辅助） | 较高 |

**Bonferroni vs FDR 对比：**

| 维度 | Bonferroni | FDR (BH) |
|------|-----------|----------|
| 保守程度 | 极保守 | 适中 |
| 10个指标时每个阈值 | 0.005 | 动态（约 0.01-0.05） |
| 假阳性控制 | 控制 family-wise error | 控制 false discovery rate |
| 适用场景 | 核心指标少（3-10个） | 指标多（10+个） |
| 统计功效 | 低（容易漏掉真效果） | 高（更容易发现真效果） |

### 多重比较校正：完整数学推导与实现

**问题定义：** 同时检验 m 个假设 $H_1, H_2, \ldots, H_m$，每个假设的 p 值为 $p_1, p_2, \ldots, p_m$。如果不校正，每个检验的假阳性率为 $\alpha$，则至少一个假阳性的概率为：

$$P(\text{至少一个假阳性}) = 1 - (1-\alpha)^m$$

当 m=10, $\alpha=0.05$ 时，$P \approx 0.40$。

#### Bonferroni 校正的完整推导

**核心思想：** 控制 family-wise error rate (FWER)，即至少犯一次第一类错误的概率。

**Boole 不等式：** 对于任意事件 $A_1, A_2, \ldots, A_m$：

$$P\left(\bigcup_{i=1}^{m} A_i\right) \leq \sum_{i=1}^{m} P(A_i)$$

**推导：** 令 $A_i$ = "第 i 个检验犯第一类错误"，则：

$$FWER = P\left(\bigcup_{i: H_i \text{为真}} A_i\right) \leq \sum_{i: H_i \text{为真}} P(A_i) = \sum_{i: H_i \text{为真}} \alpha/m \leq m_0 \cdot \alpha/m \leq \alpha$$

其中 $m_0$ 是真实原假设的个数。因此将每个检验的显著性水平设为 $\alpha/m$ 即可控制 FWER $\leq \alpha$。

#### Bonferroni-Holm 逐步校正的完整推导

**改进思路：** Bonferroni 对所有假设同等对待，而 Holm 的逐步过程利用了已拒绝假设的信息。

**算法步骤：**

1. 将 p 值排序：$p_{(1)} \leq p_{(2)} \leq \ldots \leq p_{(m)}$
2. 从最小的 p 值开始：
   - 如果 $p_{(1)} \leq \alpha/m$，拒绝 $H_{(1)}$，继续
   - 如果 $p_{(2)} \leq \alpha/(m-1)$，拒绝 $H_{(2)}$，继续
   - 一般地：如果 $p_{(k)} \leq \alpha/(m-k+1)$，拒绝 $H_{(k)}$，继续
   - 一旦某个假设不能被拒绝，停止，后续所有假设均不拒绝

**FWER 控制证明：** 设 $H_{(j)}$ 是排序后第一个真实原假设。则 $j \geq m_0$（所有真实原假设的 p 值都 $\geq p_{(j)}$）。Holm 过程错误拒绝 $H_{(j)}$ 的条件是 $p_{(j)} \leq \alpha/(m-j+1) \leq \alpha/m_0$。而 $P(p_{(j)} \leq \alpha/m_0) \leq m_0 \cdot \alpha/m_0 = \alpha$。因此 FWER $\leq \alpha$。

#### Benjamini-Hochberg FDR 校正的完整推导

**核心思想：** 控制 false discovery rate (FDR) 而非 FWER。FDR 定义为：

$$FDR = E\left[\frac{V}{R \vee 1}\right] = E\left[\frac{\text{假阳性数}}{\max(\text{拒绝数}, 1)}\right]$$

其中 V 是假阳性数，R 是总拒绝数。

**BH 算法步骤：**

1. 将 p 值排序：$p_{(1)} \leq p_{(2)} \leq \ldots \leq p_{(m)}$
2. 找到最大的 k，使得 $p_{(k)} \leq \frac{k}{m} \cdot q$
3. 拒绝 $H_{(1)}, H_{(2)}, \ldots, H_{(k)}$

**FDR 控制证明（独立检验情况）：** 定义指标 $R_k = \mathbf{1}[p_{(k)} \leq kq/m]$，则拒绝数为 $R = \max\{k: R_k = 1\}$。假阳性数 $V = \sum_{i: H_i \text{为真}} \mathbf{1}[p_i \leq \hat{t}]$，其中 $\hat{t} = \max\{p_{(k)}: p_{(k)} \leq kq/m\}$ 是自适应阈值。

由于 $\hat{t} \leq Rq/m$，我们有：

$$E\left[\frac{V}{R \vee 1}\right] \leq E\left[\frac{V}{R}\right] \leq \frac{m_0}{m} q \leq q$$

#### 完整的多重比较校正引擎

```python
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class CorrectionResult:
    """多重比较校正结果"""
    original_p: List[float]           # 原始 p 值列表
    adjusted_p: List[float]           # 校正后 p 值列表
    is_significant: List[bool]        # 校正后是否显著
    method: str                       # 使用的校正方法
    threshold_used: List[float]       # 每个假设使用的阈值

class MultipleComparisonEngine:
    """完整的多重比较校正引擎：支持 Bonferroni、Bonferroni-Holm、BH-FDR"""

    def __init__(self, alpha=0.05):
        self.alpha = alpha

    def bonferroni(self, p_values: List[float]) -> CorrectionResult:
        """
        Bonferroni 校正：最简单也最保守

        调整后 p 值: p_adj_i = min(p_i * m, 1.0)
        阈值: alpha / m
        """
        m = len(p_values)
        adjusted_p = [min(p * m, 1.0) for p in p_values]
        is_significant = [ap <= self.alpha for ap in adjusted_p]
        threshold = self.alpha / m

        return CorrectionResult(
            original_p=p_values,
            adjusted_p=adjusted_p,
            is_significant=is_significant,
            method="bonferroni",
            threshold_used=[threshold] * m
        )

    def bonferroni_holm(self, p_values: List[float]) -> CorrectionResult:
        """
        Bonferroni-Holm 逐步校正：比 Bonferroni 功效更高

        调整后 p 值计算：
        1. 排序 p_(1) <= p_(2) <= ... <= p_(m)
        2. p_adj_(1) = min(p_(1) * m, 1.0)
        3. p_adj_(k) = max(p_adj_(k-1), min(p_(k) * (m - k + 1), 1.0))
           （保证单调性：调整后 p 值不递减）
        """
        m = len(p_values)
        sorted_indices = sorted(range(m), key=lambda i: p_values[i])
        sorted_pvals = [p_values[i] for i in sorted_indices]

        adjusted_sorted = [0.0] * m
        for k in range(m):
            raw_adj = sorted_pvals[k] * (m - k)
            if k == 0:
                adjusted_sorted[k] = min(raw_adj, 1.0)
            else:
                adjusted_sorted[k] = max(adjusted_sorted[k-1], min(raw_adj, 1.0))

        adjusted_p = [0.0] * m
        thresholds = [0.0] * m
        for rank, orig_idx in enumerate(sorted_indices):
            adjusted_p[orig_idx] = adjusted_sorted[rank]
            thresholds[orig_idx] = self.alpha / (m - rank)

        is_significant = [ap <= self.alpha for ap in adjusted_p]

        return CorrectionResult(
            original_p=p_values,
            adjusted_p=adjusted_p,
            is_significant=is_significant,
            method="bonferroni_holm",
            threshold_used=thresholds
        )

    def benjamini_hochberg(self, p_values: List[float], q: Optional[float] = None) -> CorrectionResult:
        """
        Benjamini-Hochberg FDR 校正

        调整后 p 值计算：
        1. 排序 p_(1) <= p_(2) <= ... <= p_(m)
        2. 从后往前：p_adj_(m) = min(p_(m), 1.0)
        3. p_adj_(k) = min(p_(k) * m / k, p_adj_(k+1))
           （从后往前保证单调性）
        """
        if q is None:
            q = self.alpha
        m = len(p_values)

        sorted_indices = sorted(range(m), key=lambda i: p_values[i])
        sorted_pvals = [p_values[i] for i in sorted_indices]

        adjusted_sorted = [0.0] * m
        for k in range(m - 1, -1, -1):
            rank = k + 1
            raw_adj = sorted_pvals[k] * m / rank
            if k == m - 1:
                adjusted_sorted[k] = min(raw_adj, 1.0)
            else:
                adjusted_sorted[k] = min(adjusted_sorted[k + 1], min(raw_adj, 1.0))

        adjusted_p = [0.0] * m
        thresholds = [0.0] * m
        for rank_0, orig_idx in enumerate(sorted_indices):
            adjusted_p[orig_idx] = adjusted_sorted[rank_0]
            thresholds[orig_idx] = (rank_0 + 1) / m * q

        is_significant = [ap <= q for ap in adjusted_p]

        return CorrectionResult(
            original_p=p_values,
            adjusted_p=adjusted_p,
            is_significant=is_significant,
            method="benjamini_hochberg",
            threshold_used=thresholds
        )

    def hybrid_correction(self, p_values: List[float], is_primary: List[bool],
                          primary_method="bonferroni_holm") -> CorrectionResult:
        """混合校正：核心指标用严格方法，辅助指标用 FDR"""
        primary_indices = [i for i, p in enumerate(is_primary) if p]
        secondary_indices = [i for i, p in enumerate(is_primary) if not p]

        adjusted_p = [1.0] * len(p_values)
        is_significant = [False] * len(p_values)
        thresholds = [self.alpha] * len(p_values)

        if primary_indices:
            primary_pvals = [p_values[i] for i in primary_indices]
            result = self.bonferroni_holm(primary_pvals)
            for idx, pi in enumerate(primary_indices):
                adjusted_p[pi] = result.adjusted_p[idx]
                is_significant[pi] = result.is_significant[idx]
                thresholds[pi] = result.threshold_used[idx]

        if secondary_indices:
            secondary_pvals = [p_values[i] for i in secondary_indices]
            result = self.benjamini_hochberg(secondary_pvals)
            for idx, si in enumerate(secondary_indices):
                adjusted_p[si] = result.adjusted_p[idx]
                is_significant[si] = result.is_significant[idx]
                thresholds[si] = result.threshold_used[idx]

        return CorrectionResult(
            original_p=p_values, adjusted_p=adjusted_p,
            is_significant=is_significant,
            method=f"hybrid({primary_method}+bh_fdr)",
            threshold_used=thresholds
        )


# ===== 使用示例 =====
def demo_multiple_comparison():
    engine = MultipleComparisonEngine(alpha=0.05)
    p_values = [0.001, 0.012, 0.025, 0.031, 0.038, 0.042, 0.065, 0.110, 0.230, 0.450]
    is_primary = [True, True, True, False, False, False, False, False, False, False]

    print("=== Bonferroni ===")
    r = engine.bonferroni(p_values)
    print(f"阈值: {r.threshold_used[0]:.4f}, 显著数: {sum(r.is_significant)}")

    print("\n=== Bonferroni-Holm ===")
    r = engine.bonferroni_holm(p_values)
    for i, (orig, adj, sig, thr) in enumerate(zip(r.original_p, r.adjusted_p, r.is_significant, r.threshold_used)):
        print(f"  指标{i}: p={orig:.3f} -> adj_p={adj:.4f}, 阈值={thr:.4f}, 显著={sig}")

    print("\n=== Benjamini-Hochberg FDR ===")
    r = engine.benjamini_hochberg(p_values)
    for i, (orig, adj, sig, thr) in enumerate(zip(r.original_p, r.adjusted_p, r.is_significant, r.threshold_used)):
        print(f"  指标{i}: p={orig:.3f} -> adj_p={adj:.4f}, 阈值={thr:.4f}, 显著={sig}")

    print("\n=== 混合校正 ===")
    r = engine.hybrid_correction(p_values, is_primary)
    for i, (orig, adj, sig, thr) in enumerate(zip(r.original_p, r.adjusted_p, r.is_significant, r.threshold_used)):
        role = "核心" if is_primary[i] else "辅助"
        print(f"  [{role}] 指标{i}: p={orig:.3f} -> adj_p={adj:.4f}, 阈值={thr:.4f}, 显著={sig}")
```

**10 个指标 p 值在不同校正方法下的结果对比：**

| 指标 | 原始 p | Bonferroni adj | Holm adj | BH-FDR adj | 是否核心 |
|------|--------|---------------|----------|------------|---------|
| 1 | 0.001 | 0.010* | 0.010* | 0.010* | 是 |
| 2 | 0.012 | 0.120 | 0.108 | 0.060* | 是 |
| 3 | 0.025 | 0.250 | 0.200 | 0.083* | 是 |
| 4 | 0.031 | 0.310 | 0.217 | 0.088 | 否 |
| 5 | 0.038 | 0.380 | 0.228 | 0.088 | 否 |
| 6 | 0.042 | 0.420 | 0.228 | 0.088 | 否 |
| 7 | 0.065 | 0.650 | 0.325 | 0.119 | 否 |
| 8 | 0.110 | 1.000 | 0.440 | 0.183 | 否 |
| 9 | 0.230 | 1.000 | 0.690 | 0.329 | 否 |
| 10 | 0.450 | 1.000 | 1.000 | 0.450 | 否 |

（*表示校正后仍显著。Bonferroni 最严格只有 1 个显著，Holm 同样，BH-FDR 更宽松有 3 个显著。）

### 辛普森悖论检测（完整实现）

辛普森悖论的本质：整体趋势与分层数据的趋势相反。在 A/B 测试中，这通常发生在实验组和对照组在各子群中的样本分布不均衡时。

**数学解释：** 设有 k 个子群，第 j 个子群中对照组转化率为 p_Cj，实验组为 p_Tj，对照组样本量为 n_Cj，实验组为 n_Tj。整体转化率为：

- 对照组整体：p_C = Σ(n_Cj * p_Cj) / Σn_Cj（加权平均）
- 实验组整体：p_T = Σ(n_Tj * p_Tj) / Σn_Tj（加权平均）

如果实验组在高转化率子群中占比少（n_Tj/Σn_Tj < n_Cj/Σn_Cj），即使每个子群中 p_Tj > p_Cj，整体也可能 p_T < p_C。

**检测方法：** 对每个分层维度，比较整体效果方向与各子群效果方向是否一致。不一致即悖论信号。

```python
import math
from scipy.stats import norm
from dataclasses import dataclass
from typing import List, Dict, Optional


@dataclass
class StratumResult:
    """子群分析结果"""
    stratum_name: str           # 子群名称（如 "iOS", "Android"）
    control_n: int              # 对照组样本量
    control_conversions: int    # 对照组转化数
    treatment_n: int            # 实验组样本量
    treatment_conversions: int  # 实验组转化数
    control_rate: float         # 对照组转化率
    treatment_rate: float       # 实验组转化率
    effect_direction: str       # 效果方向: "POSITIVE" / "NEGATIVE" / "NEUTRAL"
    relative_change: float      # 相对变化率
    z_score: float              # Z 统计量
    p_value: float              # p 值
    ci_lower: float             # 置信区间下界
    ci_upper: float             # 置信区间上界
    weight_in_total: float      # 该子群占总样本量的权重


@dataclass
class ParadoxDetection:
    """悖论检测结果"""
    dimension: str                      # 分层维度（如 "device"）
    overall_direction: str              # 整体效果方向
    contradictory_strata: List[StratumResult]  # 与整体方向矛盾的子群
    paradox_severity: str               # 严重程度: "HIGH" / "MEDIUM" / "LOW"
    explanation: str                    # 悖论解释
    recommended_action: str             # 建议行动


class SimpsonParadoxDetector:
    """
    辛普森悖论检测器

    检测维度：
    1. 设备类型（iOS / Android / Desktop）
    2. 地区（国内 / 海外）
    3. 用户类型（新用户 / 老用户）
    4. 流量来源（自然流量 / 付费流量）

    检测逻辑：
    1. 计算整体效果方向
    2. 按各维度分层计算效果方向
    3. 如果子群方向与整体相反 → 标记悖论
    4. 评估悖论严重程度（基于子群样本占比和效果量）
    """

    DEFAULT_DIMENSIONS = ["device", "country", "user_type", "traffic_source"]

    def __init__(self, db_connection, dimensions=None):
        self.db = db_connection
        self.dimensions = dimensions or self.DEFAULT_DIMENSIONS

    def detect(self, experiment_id: str,
               metric_name: str = "conversion_rate") -> List[ParadoxDetection]:
        """检测实验是否存在辛普森悖论"""
        overall = self._compute_overall(experiment_id, metric_name)
        paradoxes = []

        for dimension in self.dimensions:
            strata = self._compute_stratified(
                experiment_id, metric_name, dimension)
            contradictory = [
                s for s in strata
                if s.effect_direction != "NEUTRAL"
                and s.effect_direction != overall.effect_direction
            ]
            if contradictory:
                severity = self._assess_severity(
                    overall, contradictory, strata)
                explanation = self._generate_explanation(
                    dimension, overall, contradictory, strata)
                action = self._recommend_action(
                    severity, contradictory, strata)
                paradoxes.append(ParadoxDetection(
                    dimension=dimension,
                    overall_direction=overall.effect_direction,
                    contradictory_strata=contradictory,
                    paradox_severity=severity,
                    explanation=explanation,
                    recommended_action=action,
                ))

        if paradoxes:
            self._alert(experiment_id, paradoxes)
        return paradoxes

    def _compute_overall(self, experiment_id, metric_name):
        """计算整体效果"""
        row = self.db.query_one("""
            SELECT
                SUM(CASE WHEN variant='control' THEN n ELSE 0 END) as control_n,
                SUM(CASE WHEN variant='control' THEN conv ELSE 0 END) as control_conv,
                SUM(CASE WHEN variant='treatment' THEN n ELSE 0 END) as treatment_n,
                SUM(CASE WHEN variant='treatment' THEN conv ELSE 0 END) as treatment_conv
            FROM experiment_metric_aggregates
            WHERE experiment_id=%s AND metric_name=%s
        """, experiment_id, metric_name)
        control_rate = row.control_conv / row.control_n \
            if row.control_n > 0 else 0
        treatment_rate = row.treatment_conv / row.treatment_n \
            if row.treatment_n > 0 else 0
        z, p, ci_l, ci_u = self._proportion_test(
            row.control_n, row.control_conv,
            row.treatment_n, row.treatment_conv)
        rel = ((treatment_rate - control_rate) / control_rate
               if control_rate > 0 else 0)
        direction = self._determine_direction(z, p, rel)
        return StratumResult(
            stratum_name="整体",
            control_n=row.control_n,
            control_conversions=row.control_conv,
            treatment_n=row.treatment_n,
            treatment_conversions=row.treatment_conv,
            control_rate=control_rate, treatment_rate=treatment_rate,
            effect_direction=direction, relative_change=rel,
            z_score=z, p_value=p, ci_lower=ci_l, ci_upper=ci_u,
            weight_in_total=1.0)

    def _compute_stratified(self, experiment_id, metric_name, dimension):
        """按维度分层计算效果"""
        col_map = {"device": "device_type", "country": "country_code",
                   "user_type": "user_tier", "traffic_source": "traffic_source"}
        col = col_map.get(dimension, dimension)
        rows = self.db.query("""
            SELECT {col} as stratum,
                SUM(CASE WHEN variant='control' THEN n ELSE 0 END) as control_n,
                SUM(CASE WHEN variant='control' THEN conv ELSE 0 END) as control_conv,
                SUM(CASE WHEN variant='treatment' THEN n ELSE 0 END) as treatment_n,
                SUM(CASE WHEN variant='treatment' THEN conv ELSE 0 END) as treatment_conv
            FROM experiment_metric_aggregates
            WHERE experiment_id=%s AND metric_name=%s
            GROUP BY {col}
        """.format(col=col), experiment_id, metric_name)
        total_n = sum(r.control_n + r.treatment_n for r in rows)
        results = []
        for row in rows:
            cr = row.control_conv / row.control_n if row.control_n > 0 else 0
            tr = row.treatment_conv / row.treatment_n if row.treatment_n > 0 else 0
            z, p, ci_l, ci_u = self._proportion_test(
                row.control_n, row.control_conv,
                row.treatment_n, row.treatment_conv)
            rel = ((tr - cr) / cr if cr > 0 else 0)
            direction = self._determine_direction(z, p, rel)
            weight = (row.control_n + row.treatment_n) / total_n \
                if total_n > 0 else 0
            results.append(StratumResult(
                stratum_name=row.stratum,
                control_n=row.control_n,
                control_conversions=row.control_conv,
                treatment_n=row.treatment_n,
                treatment_conversions=row.treatment_conv,
                control_rate=cr, treatment_rate=tr,
                effect_direction=direction, relative_change=rel,
                z_score=z, p_value=p, ci_lower=ci_l, ci_upper=ci_u,
                weight_in_total=weight))
        return results

    def _proportion_test(self, n1, x1, n2, x2):
        """双样本比例检验"""
        if n1 == 0 or n2 == 0:
            return 0, 1.0, 0, 0
        p1 = x1 / n1; p2 = x2 / n2
        p_pool = (x1 + x2) / (n1 + n2)
        se = math.sqrt(p_pool * (1 - p_pool) * (1/n1 + 1/n2))
        if se == 0:
            return 0, 1.0, p2 - p1, p2 - p1
        z = (p2 - p1) / se
        p_val = 2 * (1 - norm.cdf(abs(z)))
        ci_half = 1.96 * math.sqrt(p1*(1-p1)/n1 + p2*(1-p2)/n2)
        return z, p_val, (p2-p1)-ci_half, (p2-p1)+ci_half

    def _determine_direction(self, z_score, p_value, relative_change):
        """判定效果方向"""
        if p_value > 0.05:
            return "NEUTRAL"
        return "POSITIVE" if relative_change > 0 else "NEGATIVE"

    def _assess_severity(self, overall, contradictory, all_strata):
        """评估悖论严重程度"""
        contradiction_weight = sum(s.weight_in_total for s in contradictory)
        significant = any(s.p_value < 0.05 for s in contradictory)
        if contradiction_weight > 0.30 and significant:
            return "HIGH"
        elif contradiction_weight > 0.10:
            return "MEDIUM"
        return "LOW"

    def _generate_explanation(self, dimension, overall, contradictory, all_strata):
        """生成悖论的自然语言解释"""
        parts = [f"在 {dimension} 维度上检测到辛普森悖论："]
        parts.append(f"  整体: {overall.effect_direction} "
                    f"(对照={overall.control_rate:.2%}, "
                    f"实验={overall.treatment_rate:.2%})")
        for s in contradictory:
            parts.append(f"  {s.stratum_name}: {s.effect_direction} "
                        f"(对照={s.control_rate:.2%} n={s.control_n}, "
                        f"实验={s.treatment_rate:.2%} n={s.treatment_n}, "
                        f"权重={s.weight_in_total:.1%})")
        for s in all_strata:
            ctrl_share = s.control_n / (s.control_n + s.treatment_n) \
                if (s.control_n + s.treatment_n) > 0 else 0.5
            if abs(ctrl_share - 0.5) > 0.1:
                parts.append(f"  注意: {s.stratum_name} 对照组占比 "
                            f"{ctrl_share:.1%}（偏离50%），可能是悖论原因")
        return "\n".join(parts)

    def _recommend_action(self, severity, contradictory, all_strata):
        """根据严重程度推荐行动"""
        if severity == "HIGH":
            return ("悖论严重：整体结论不可信。"
                    "建议：(1) 分层报告各子群结果；(2) 基于子群做分平台决策；"
                    "(3) 不全量上线，或仅对正向子群上线。")
        elif severity == "MEDIUM":
            return ("悖论中等：整体结论需补充说明。"
                    "建议：(1) 标注子群差异；(2) 关注矛盾子群趋势；"
                    "(3) 全量上线需监控矛盾子群指标。")
        return ("悖论轻微：整体结论基本可信。"
                "建议：在报告中附带子群分析结果。")

    def _alert(self, experiment_id, paradoxes):
        """发送悖论告警"""
        high = [p for p in paradoxes if p.paradox_severity == "HIGH"]
        if high:
            self._send_urgent_alert(experiment_id, high)
        else:
            self._add_to_dashboard(experiment_id, paradoxes)
```

**辛普森悖论的量化判定标准：**

| 严重程度 | 矛盾子群权重 | 统计显著性 | 建议 |
|---------|-----------|----------|------|
| HIGH | > 30% | 有显著矛盾 | 不采信整体结论，分层决策 |
| MEDIUM | 10-30% | 可能有 | 补充说明，监控子群 |
| LOW | < 10% | 无 | 整体结论基本可信，附带子群分析 |

**辛普森悖论的预防措施：**

| 措施 | 说明 | 成本 |
|------|------|------|
| 分层随机化 | 分组时按设备/地区分层，保证各组子群比例一致 | 低（修改分组逻辑） |
| 事后分层分析 | 实验结束后自动按各维度分层分析 | 低（计算量增加） |
| 标准化加权 | 用标准化人口结构计算加权效果量 | 中（需要参考分布） |
| CUPED + 协变量 | 将子群身份作为协变量调整效果量 | 中（需要预处理） |

### 辛普森悖论检测：完整自动化实现

辛普森悖论的本质是：处理效应的方向在总体和子群体之间发生反转。这通常由**混杂因素**导致——某个子群体在实验组和对照组中的分布不均衡，而该子群体本身的效果又与总体不同。

**数学条件：** 设 Y 为指标，T 为处理（0/1），S 为分层变量。辛普森悖论发生当：

$$\text{sign}(E[Y|T=1] - E[Y|T=0]) \neq \text{sign}(E[Y|T=1,S=s] - E[Y|T=0,S=s])$$

对某个子群体 s 成立。更严格地说，需要检验这种反转是否统计显著，而非仅仅因为采样噪声导致。

**自动化检测流程：**

1. **分层选择：** 自动枚举所有预定义分层维度（设备、地区、用户等级、新老用户、时段）
2. **效应方向比较：** 计算总体效应方向和每个子群体效应方向
3. **统计显著性检验：** 对反转的子群体进行 Breslow-Day 齐性检验，确认反转不是偶然
4. **影响评估：** 计算加权平均效应（用子群体比例加权），与总体效应比较
5. **建议生成：** 基于分析给出决策建议

```python
from dataclasses import dataclass
from typing import List, Dict, Optional
from scipy.stats import chi2, norm
import math

@dataclass
class StratumResult:
    """单个子群体的分析结果"""
    stratum_name: str           # 子群体名称
    control_n: int              # 对照组样本量
    treatment_n: int            # 实验组样本量
    control_mean: float         # 对照组均值
    treatment_mean: float       # 实验组均值
    effect: float               # 处理效应（treatment - control）
    effect_direction: str       # "positive" / "negative" / "neutral"
    p_value: float              # 该子群体的检验 p 值
    is_significant: bool        # 该子群体是否显著
    weight: float               # 该子群体在总体中的权重

@dataclass
class ParadoxReport:
    """辛普森悖论检测报告"""
    has_paradox: bool
    overall_direction: str
    overall_effect: float
    reversed_strata: List[Dict]
    weighted_effect: float      # 加权平均效应
    breslow_day_p: float        # Breslow-Day 齐性检验 p 值
    recommendation: str
    confidence: str             # "high" / "medium" / "low"


class SimpsonParadoxDetectorV2:
    """辛普森悖论完整检测引擎"""

    STRATIFICATION_DIMENSIONS = [
        "device_type",      # 设备类型：mobile / desktop / tablet
        "platform",         # 平台：ios / android / web
        "user_tier",        # 用户等级：new / casual / active / vip
        "region",           # 地区
        "traffic_source",   # 流量来源
    ]

    def __init__(self, db_client, alpha=0.05):
        self.db = db_client
        self.alpha = alpha

    def detect(self, experiment_id: str, metric_name: str) -> ParadoxReport:
        """完整的辛普森悖论检测流程"""
        # 1. 计算总体效应
        overall = self._compute_overall_effect(experiment_id, metric_name)

        # 2. 逐个分层维度检测
        all_reversed = []
        all_strata_results = {}

        for dimension in self.STRATIFICATION_DIMENSIONS:
            strata = self._compute_stratified_effects(experiment_id, metric_name, dimension)
            all_strata_results[dimension] = strata

            for stratum in strata:
                if stratum.effect_direction != overall.effect_direction and stratum.is_significant:
                    all_reversed.append({
                        "dimension": dimension,
                        "stratum": stratum.stratum_name,
                        "overall_direction": overall.effect_direction,
                        "stratum_direction": stratum.effect_direction,
                        "overall_effect": overall.effect,
                        "stratum_effect": stratum.effect,
                        "stratum_weight": stratum.weight,
                        "stratum_p_value": stratum.p_value,
                    })

        # 3. Breslow-Day 齐性检验
        bd_p_value = self._breslow_day_test(experiment_id, metric_name, all_reversed)

        # 4. 计算加权平均效应
        weighted_effect = self._compute_weighted_effect(all_strata_results)

        # 5. 判断悖论是否严重
        has_paradox = len(all_reversed) > 0 and bd_p_value < self.alpha
        is_severe = (len(all_reversed) > 0 and
                     abs(weighted_effect - overall.effect) > abs(overall.effect) * 0.3)

        # 6. 生成建议
        recommendation = self._generate_recommendation(
            has_paradox, is_severe, overall, weighted_effect, all_reversed
        )
        confidence = "high" if bd_p_value < 0.001 else ("medium" if bd_p_value < 0.01 else "low")

        return ParadoxReport(
            has_paradox=has_paradox,
            overall_direction=overall.effect_direction,
            overall_effect=overall.effect,
            reversed_strata=all_reversed,
            weighted_effect=weighted_effect,
            breslow_day_p=bd_p_value,
            recommendation=recommendation,
            confidence=confidence,
        )

    def _compute_overall_effect(self, experiment_id, metric_name):
        """计算总体的处理效应"""
        rows = self.db.query("""
            SELECT group_name, COUNT(*) as n, AVG(value) as mean, STDDEV(value) as stddev
            FROM experiment_metrics
            WHERE experiment_id = %s AND metric_name = %s
            GROUP BY group_name
        """, experiment_id, metric_name)

        control = next(r for r in rows if r['group_name'] == 'control')
        treatment = next(r for r in rows if r['group_name'] == 'treatment')

        effect = treatment['mean'] - control['mean']
        se = math.sqrt(treatment['stddev']**2 / treatment['n'] + control['stddev']**2 / control['n'])
        z = effect / se if se > 0 else 0
        p_value = 2 * (1 - norm.cdf(abs(z)))

        return StratumResult(
            stratum_name="overall", control_n=control['n'], treatment_n=treatment['n'],
            control_mean=control['mean'], treatment_mean=treatment['mean'], effect=effect,
            effect_direction="positive" if effect > 0 else ("negative" if effect < 0 else "neutral"),
            p_value=p_value, is_significant=p_value < self.alpha, weight=1.0,
        )

    def _compute_stratified_effects(self, experiment_id, metric_name, dimension):
        """计算按维度分层后每个子群体的处理效应"""
        rows = self.db.query(f"""
            SELECT {dimension} as stratum, group_name,
                   COUNT(*) as n, AVG(value) as mean, STDDEV(value) as stddev
            FROM experiment_metrics m JOIN user_profiles u ON m.user_id = u.user_id
            WHERE m.experiment_id = %s AND m.metric_name = %s
            GROUP BY {dimension}, group_name
        """, experiment_id, metric_name)

        strata_data = {}
        for row in rows:
            s = row['stratum']
            strata_data.setdefault(s, {})[row['group_name']] = row

        total_n = sum(r['n'] for r in rows)
        results = []
        for stratum, groups in strata_data.items():
            if 'control' not in groups or 'treatment' not in groups:
                continue
            c, t = groups['control'], groups['treatment']
            effect = t['mean'] - c['mean']
            se = math.sqrt((t['stddev']**2 / t['n'] if t['n'] > 1 else 1) +
                          (c['stddev']**2 / c['n'] if c['n'] > 1 else 1))
            z = effect / se if se > 0 else 0
            p_value = 2 * (1 - norm.cdf(abs(z)))

            results.append(StratumResult(
                stratum_name=str(stratum), control_n=c['n'], treatment_n=t['n'],
                control_mean=c['mean'], treatment_mean=t['mean'], effect=effect,
                effect_direction="positive" if effect > 0 else ("negative" if effect < 0 else "neutral"),
                p_value=p_value, is_significant=p_value < self.alpha,
                weight=(c['n'] + t['n']) / total_n if total_n > 0 else 0,
            ))
        return results

    def _breslow_day_test(self, experiment_id, metric_name, reversed_strata):
        """Breslow-Day 齐性检验：检验效应是否跨子群体一致"""
        if not reversed_strata:
            return 1.0
        dimension = reversed_strata[0]['dimension']
        rows = self.db.query(f"""
            SELECT {dimension} as stratum, group_name, COUNT(*) as n,
                   SUM(CASE WHEN value > 0 THEN 1 ELSE 0 END) as successes
            FROM experiment_metrics m JOIN user_profiles u ON m.user_id = u.user_id
            WHERE m.experiment_id = %s AND m.metric_name = %s
            GROUP BY {dimension}, group_name
        """, experiment_id, metric_name)

        strata_tables = {}
        for row in rows:
            strata_tables.setdefault(row['stratum'], {})[row['group_name']] = row

        # Mantel-Haenszel 公共 OR 估计
        num, den = 0.0, 0.0
        for stratum, groups in strata_tables.items():
            if 'control' not in groups or 'treatment' not in groups:
                continue
            c, t = groups['control'], groups['treatment']
            a, b = t['successes'], t['n'] - t['successes']
            cc, d = c['successes'], c['n'] - c['successes']
            n_i = a + b + cc + d
            if b * cc > 0:
                num += (a * d) / n_i
                den += (b * cc) / n_i
        common_or = num / den if den > 0 else 1.0

        # Breslow-Day 统计量
        bd_stat, df = 0.0, 0
        for stratum, groups in strata_tables.items():
            if 'control' not in groups or 'treatment' not in groups:
                continue
            c, t = groups['control'], groups['treatment']
            a, b = t['successes'], t['n'] - t['successes']
            cc, d = c['successes'], c['n'] - c['successes']
            n1i, n2i = a + b, cc + d
            a_exp = self._solve_expected_a(n1i, n2i, cc + d, common_or)
            if a_exp is not None:
                var_a = 1.0 / (1.0/max(a_exp,0.01) + 1.0/max(n1i-a_exp,0.01) +
                               1.0/max(cc+d-n2i+a_exp,0.01) + 1.0/max(n2i-a_exp,0.01))
                if var_a > 0:
                    bd_stat += (a - a_exp) ** 2 / var_a
                    df += 1

        if df <= 0:
            return 1.0
        return 1 - chi2.cdf(bd_stat, df=max(df - 1, 1))

    def _solve_expected_a(self, n1i, n2i, n_col, common_or, max_iter=100):
        """求解公共 OR 下的期望 a 值"""
        a = n1i * n2i / (n1i + n_col - n2i) if (n1i + n_col - n2i) > 0 else n1i / 2
        for _ in range(max_iter):
            denom = (n1i - a) * (n_col - n2i - n1i + a)
            if abs(denom) < 1e-10:
                break
            current_or = a * (n_col - n2i - n1i + a) / denom
            if abs(current_or - common_or) < 1e-6:
                break
            a = a * (common_or / current_or) ** 0.5
            a = max(0.01, min(a, n1i - 0.01))
        return a

    def _compute_weighted_effect(self, all_strata_results):
        """计算加权平均效应"""
        best_dim = max(all_strata_results.keys(),
                      key=lambda d: sum(s.control_n + s.treatment_n for s in all_strata_results[d]))
        strata = all_strata_results[best_dim]
        total_weight = sum(s.weight for s in strata)
        if total_weight == 0:
            return 0.0
        return sum(s.effect * s.weight for s in strata) / total_weight

    def _generate_recommendation(self, has_paradox, is_severe, overall, weighted_effect, reversed_strata):
        """生成决策建议"""
        if not has_paradox:
            if overall.is_significant:
                return f"结论可信：总体效应方向一致，加权效应={weighted_effect:.4f}，建议按总体结论决策。"
            return "无显著效应，无辛普森悖论风险。"
        dims = set(r['dimension'] for r in reversed_strata)
        main_dim = list(dims)[0] if dims else "未知"
        if is_severe:
            return (f"严重辛普森悖论！在 {main_dim} 维度上效应方向反转。"
                   f"总体效应={overall.effect:.4f}，加权效应={weighted_effect:.4f}。"
                   f"强烈建议：以 {main_dim} 分层分析结论为准，不要使用总体结论。")
        return (f"轻微辛普森悖论：在 {main_dim} 维度上部分子群体效应反转，"
               f"总体效应={overall.effect:.4f}，加权效应={weighted_effect:.4f}。"
               f"建议：参考加权效应结论，同时关注 {main_dim} 分层结果。")
```

### 指标收集与聚合

```python
class MetricCollector:
    def collect(self, event):
        """收集实验指标数据"""
        # 事件格式：{user_id, experiment_id, variant, metric_name, value, timestamp}
        self.kafka.produce("experiment_events", {
            "user_id": event.user_id,
            "experiment_id": event.experiment_id,
            "variant": event.variant,
            "metric_name": event.metric_name,
            "value": event.value,
            "timestamp": now_ms()
        })

    def aggregate(self, experiment_id, metric_name):
        """聚合指标数据（每小时 Flink 任务）"""
        return self.clickhouse.query("""
            SELECT variant,
                   AVG(value) as mean,
                   STDDEV(value) as stddev,
                   COUNT(*) as n
            FROM experiment_metrics
            WHERE experiment_id = %(exp_id)s AND metric_name = %(metric)s
            GROUP BY variant
        """, exp_id=experiment_id, metric=metric_name)
```

### 完整指标管道：事件采集 -> 流聚合 -> 统计检验 -> 看板

A/B 测试平台的指标管道是整个系统的数据命脉。从用户行为事件到最终实验看板，数据需要经过四个阶段，每个阶段都有不同的延迟和一致性要求。

**整体架构：**

```
用户行为 -> Kafka 事件流 -> Flink 流聚合 -> ClickHouse 存储 -> 统计检验服务 -> 实验看板
  (ms级)     (ms级延迟)      (秒级聚合)       (毫秒级查询)       (分钟级计算)      (秒级渲染)
```

**数据流详细说明：**

| 阶段 | 输入 | 输出 | 延迟 | 数据量 | 技术选型 |
|------|------|------|------|--------|---------|
| 事件采集 | 用户行为 JSON | Kafka Topic | < 100ms | 日 10 亿事件 | Kafka 3.x |
| 流聚合 | 原始事件 | 分钟/小时聚合指标 | 1-5min | 日 500 万聚合行 | Flink 1.18 |
| 统计检验 | 聚合指标 | 检验结果 | 1-5min | 日 5 万检验结果 | Python/Spark |
| 看板渲染 | 检验结果 | 可视化面板 | < 3s | 50+ 实验看板 | ClickHouse + Grafana |

#### 第一阶段：事件采集（Kafka 生产者）

```python
import json
import time
from confluent_kafka import Producer
from dataclasses import dataclass, asdict
from typing import Optional

@dataclass
class ExperimentEvent:
    """实验事件数据结构"""
    event_id: str               # 事件唯一 ID（幂等键）
    user_id: str                # 用户 ID
    experiment_id: str          # 实验 ID
    variant: str                # 分组（control / treatment）
    metric_name: str            # 指标名（click / purchase / dwell_time）
    metric_value: float         # 指标值
    event_time: int             # 事件时间戳（毫秒）
    device_type: Optional[str] = None    # 设备类型
    platform: Optional[str] = None       # 平台
    region: Optional[str] = None         # 地区
    user_tier: Optional[str] = None      # 用户等级


class ExperimentEventProducer:
    """实验事件 Kafka 生产者"""

    def __init__(self, bootstrap_servers="kafka:9092"):
        self.producer = Producer({
            'bootstrap.servers': bootstrap_servers,
            'compression.type': 'lz4',            # 压缩：节省 60% 带宽
            'batch.size': 65536,                   # 批量大小：64KB
            'linger.ms': 5,                        # 等待 5ms 凑批
            'acks': '1',                           # 主节点确认即可
            'retries': 3,                          # 重试 3 次
            'retry.backoff.ms': 100,
            'enable.idempotence': True,            # 幂等生产：防重复
        })
        self.topic = "experiment_events"

    def send(self, event: ExperimentEvent):
        """发送实验事件到 Kafka"""
        # 按实验 ID 分区：同一实验的事件在同一分区，保证有序
        partition_key = f"{event.experiment_id}:{event.metric_name}"

        self.producer.produce(
            topic=self.topic,
            key=partition_key.encode('utf-8'),
            value=json.dumps(asdict(event)).encode('utf-8'),
            callback=self._delivery_callback,
        )
        self.producer.poll(0)  # 触发回调

    def flush(self):
        """确保所有消息发送完成"""
        self.producer.flush()

    def _delivery_callback(self, err, msg):
        if err:
            # 发送失败：记录到死信队列
            self._send_to_dead_letter(msg, err)

    def _send_to_dead_letter(self, msg, err):
        """发送到死信队列：避免数据丢失"""
        self.producer.produce(
            topic="experiment_events_dql",
            key=msg.key(),
            value=json.dumps({
                "original_value": msg.value().decode('utf-8'),
                "error": str(err),
                "timestamp": int(time.time() * 1000),
            }).encode('utf-8'),
        )
```

#### 第二阶段：Flink 流聚合

```java
// Flink 作业：实时聚合实验指标
// 输入：Kafka topic "experiment_events"
// 输出：ClickHouse 表 "experiment_metric_aggregates"

public class ExperimentMetricAggregation {

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.enableCheckpointing(60000); // 每分钟 checkpoint
        env.getCheckpointConfig().setCheckpointTimeout(30000);

        // 1. Kafka Source
        KafkaSource<ExperimentEvent> source = KafkaSource.<ExperimentEvent>builder()
            .setBootstrapServers("kafka:9092")
            .setTopics("experiment_events")
            .setGroupId("flink-experiment-aggregation")
            .setStartingOffsets(OffsetsInitializer.latest())
            .setValueOnlyDeserializer(new ExperimentEventDeserializer())
            .build();

        DataStream<ExperimentEvent> events = env.fromSource(
            source, WatermarkStrategy.<ExperimentEvent>forBoundedOutOfOrderness(
                Duration.ofSeconds(10)
            ).withTimestampAssigner((event, ts) -> event.getEventTime()),
            "experiment-events"
        );

        // 2. 分钟级聚合（1 分钟滚动窗口）
        DataStream<MetricAggregate> minuteAggregates = events
            .keyBy(event -> new Tuple3<>(
                event.getExperimentId(),
                event.getVariant(),
                event.getMetricName()
            ))
            .window(TumblingEventTimeWindows.of(Time.minutes(1)))
            .aggregate(new MetricAggregateFunction());

        // 3. 写入 ClickHouse（分钟级聚合）
        minuteAggregates.addSink(new ClickHouseSink(
            "clickhouse:8123",
            "INSERT INTO experiment_metric_aggregates " +
            "(experiment_id, variant, metric_name, window_start, window_end, " +
            " event_count, sum_value, sum_sq_value, min_value, max_value, mean_value, " +
            " device_dist, platform_dist) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)"
        ));

        // 4. 小时级聚合（1 小时滚动窗口，基于分钟聚合）
        DataStream<MetricAggregate> hourAggregates = minuteAggregates
            .keyBy(agg -> new Tuple3<>(
                agg.getExperimentId(),
                agg.getVariant(),
                agg.getMetricName()
            ))
            .window(TumblingEventTimeWindows.of(Time.hours(1)))
            .reduce(new MetricAggregateReducer());

        // 5. 写入 ClickHouse（小时级聚合，用于实验看板）
        hourAggregates.addSink(new ClickHouseSink(
            "clickhouse:8123",
            "INSERT INTO experiment_metric_hourly " +
            "(experiment_id, variant, metric_name, window_start, window_end, " +
            " event_count, sum_value, sum_sq_value, stddev_value, mean_value) " +
            "VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)"
        ));

        env.execute("Experiment Metric Aggregation");
    }
}

/**
 * 聚合函数：计算每个窗口的统计量
 * 使用 Welford 在线算法计算均值和方差，避免数值精度问题
 */
public class MetricAggregateFunction implements AggregateFunction<
    ExperimentEvent, MetricAccumulator, MetricAggregate> {

    @Override
    public MetricAccumulator createAccumulator() {
        return new MetricAccumulator();
    }

    @Override
    public MetricAccumulator add(ExperimentEvent event, MetricAccumulator acc) {
        acc.experimentId = event.getExperimentId();
        acc.variant = event.getVariant();
        acc.metricName = event.getMetricName();
        acc.count++;

        // Welford 在线算法：增量计算均值和方差
        double delta = event.getMetricValue() - acc.mean;
        acc.mean += delta / acc.count;
        double delta2 = event.getMetricValue() - acc.mean;
        acc.m2 += delta * delta2;  // M2 用于计算方差

        acc.sum += event.getMetricValue();
        acc.min = Math.min(acc.min, event.getMetricValue());
        acc.max = Math.max(acc.max, event.getMetricValue());

        // 分层统计（用于辛普森悖论检测）
        acc.deviceCounts.merge(event.getDeviceType(), 1, Integer::sum);
        acc.platformCounts.merge(event.getPlatform(), 1, Integer::sum);

        return acc;
    }

    @Override
    public MetricAggregate getResult(MetricAccumulator acc) {
        double variance = acc.count > 1 ? acc.m2 / (acc.count - 1) : 0;
        return new MetricAggregate(
            acc.experimentId, acc.variant, acc.metricName,
            acc.count, acc.sum, acc.m2, acc.min, acc.max,
            acc.mean, Math.sqrt(variance),
            acc.deviceCounts, acc.platformCounts
        );
    }

    @Override
    public MetricAccumulator merge(MetricAccumulator a, MetricAccumulator b) {
        // 合并两个累加器（并行聚合时使用）
        long totalCount = a.count + b.count;
        double delta = b.mean - a.mean;
        a.mean = (a.count * a.mean + b.count * b.mean) / totalCount;
        a.m2 = a.m2 + b.m2 + delta * delta * a.count * b.count / totalCount;
        a.count = totalCount;
        a.sum += b.sum;
        a.min = Math.min(a.min, b.min);
        a.max = Math.max(a.max, b.max);
        b.deviceCounts.forEach((k, v) -> a.deviceCounts.merge(k, v, Integer::sum));
        b.platformCounts.forEach((k, v) -> a.platformCounts.merge(k, v, Integer::sum));
        return a;
    }
}
```

#### ClickHouse 聚合表设计

```sql
-- 分钟级聚合表：用于近实时监控和异常检测
CREATE TABLE experiment_metric_aggregates (
    experiment_id  String,
    variant        String,
    metric_name    String,
    window_start   DateTime,
    window_end     DateTime,
    event_count    UInt64,
    sum_value      Float64,
    sum_sq_value   Float64,
    min_value      Float64,
    max_value      Float64,
    mean_value     Float64,
    -- 分层分布（用于辛普森悖论检测）
    device_dist    Map(String, UInt64),
    platform_dist  Map(String, UInt64)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(window_start)
ORDER BY (experiment_id, variant, metric_name, window_start)
TTL window_start + INTERVAL 90 DAY;

-- 小时级聚合表：用于实验看板和统计检验
CREATE TABLE experiment_metric_hourly (
    experiment_id  String,
    variant        String,
    metric_name    String,
    window_start   DateTime,
    window_end     DateTime,
    event_count    UInt64,
    sum_value      Float64,
    sum_sq_value   Float64,
    stddev_value   Float64,
    mean_value     Float64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(window_start)
ORDER BY (experiment_id, variant, metric_name, window_start)
TTL window_start + INTERVAL 1 YEAR;

-- 物化视图：自动从分钟聚合生成小时聚合
CREATE MATERIALIZED VIEW experiment_metric_hourly_mv
TO experiment_metric_hourly AS
SELECT
    experiment_id,
    variant,
    metric_name,
    toStartOfHour(window_start) AS window_start,
    toStartOfHour(window_start) + INTERVAL 1 HOUR AS window_end,
    sum(event_count) AS event_count,
    sum(sum_value) AS sum_value,
    sum(sum_sq_value) AS sum_sq_value,
    sqrt(sum(sum_sq_value) / sum(event_count) -
         pow(sum(sum_value) / sum(event_count), 2)) AS stddev_value,
    sum(sum_value) / sum(event_count) AS mean_value
FROM experiment_metric_aggregates
GROUP BY experiment_id, variant, metric_name, toStartOfHour(window_start);
```

#### 第三阶段：统计检验服务

```python
class StatisticalTestService:
    """统计检验服务：从 ClickHouse 读取聚合数据，执行检验，写入结果"""

    def __init__(self, clickhouse_client, result_store):
        self.ch = clickhouse_client
        self.result_store = result_store
        self.sequential_engine = SequentialTestEngine(alpha=0.05, boundary_type="obf")
        self.mc_engine = MultipleComparisonEngine(alpha=0.05)
        self.simpson_detector = SimpsonParadoxDetectorV2(clickhouse_client)

    def run_analysis(self, experiment_id: str):
        """运行完整的统计分析流水线"""
        experiment = self._get_experiment(experiment_id)

        # 1. 检查数据新鲜度
        freshness = self._check_data_freshness(experiment_id)
        if freshness['delay_minutes'] > 30:
            self._alert_data_delay(experiment_id, freshness)

        # 2. SRM 检测
        srm_result = self._check_srm(experiment_id)

        # 3. 对每个指标执行统计检验
        metric_results = []
        for metric in experiment.metrics:
            result = self._test_metric(experiment, metric)
            metric_results.append(result)

        # 4. 多重比较校正
        p_values = [r.p_value for r in metric_results]
        is_primary = [m.is_primary for m in experiment.metrics]
        correction = self.mc_engine.hybrid_correction(p_values, is_primary)

        for i, result in enumerate(metric_results):
            result.adjusted_p_value = correction.adjusted_p[i]
            result.is_significant = correction.is_significant[i]

        # 5. 辛普森悖论检测（仅对核心指标）
        paradox_reports = {}
        for metric in experiment.metrics:
            if metric.is_primary:
                report = self.simpson_detector.detect(experiment_id, metric.metric_name)
                paradox_reports[metric.metric_name] = report

        # 6. 写入结果
        analysis = ExperimentAnalysis(
            experiment_id=experiment_id,
            analysis_time=datetime.now(),
            data_freshness=freshness,
            srm_detected=srm_result.is_detected,
            srm_p_value=srm_result.p_value,
            metric_results=metric_results,
            paradox_reports=paradox_reports,
            info_fraction=self._compute_info_fraction(experiment),
        )
        self.result_store.save(analysis)
        return analysis

    def _test_metric(self, experiment, metric):
        """对单个指标执行统计检验"""
        # 从 ClickHouse 读取聚合数据
        rows = self.ch.query("""
            SELECT variant, event_count as n, mean_value as mean,
                   stddev_value as stddev, sum_value as sum, sum_sq_value as sum_sq
            FROM experiment_metric_hourly
            WHERE experiment_id = %(exp_id)s AND metric_name = %(metric)s
              AND window_start >= %(start)s
            GROUP BY variant, event_count, mean_value, stddev_value, sum_value, sum_sq_value
        """, exp_id=experiment.id, metric=metric.metric_name,
             start=experiment.started_at)

        control = next(r for r in rows if r['variant'] == 'control')
        treatment = next(r for r in rows if r['variant'] == 'treatment')

        # 序贯检验
        seq_result = self.sequential_engine.analyze(
            experiment, metric.metric_name,
            current_data=StatData(**control, **treatment)
        )

        return MetricTestResult(
            metric_name=metric.metric_name,
            metric_type=metric.metric_type,
            control_mean=control['mean'],
            treatment_mean=treatment['mean'],
            control_n=control['n'],
            treatment_n=treatment['n'],
            effect_size=treatment['mean'] - control['mean'],
            relative_change=(treatment['mean'] - control['mean']) / control['mean'],
            p_value=seq_result.p_value,
            is_significant=seq_result.is_significant,
            info_fraction=seq_result.info_fraction,
            z_score=seq_result.z_score,
            z_boundary=seq_result.z_boundary,
            can_early_stop=seq_result.can_early_stop,
            recommendation=seq_result.recommendation,
        )

    def _check_data_freshness(self, experiment_id):
        """检查数据新鲜度"""
        row = self.ch.query_one("""
            SELECT max(window_end) as latest_data_time,
                   now() - max(window_end) as delay_seconds
            FROM experiment_metric_aggregates
            WHERE experiment_id = %(exp_id)s
        """, exp_id=experiment_id)

        return {
            "latest_data_time": row['latest_data_time'],
            "delay_minutes": row['delay_seconds'] / 60 if row['delay_seconds'] else 0,
        }
```

#### 第四阶段：看板查询优化

```sql
-- 实验看板核心查询：获取某个实验的实时结果
-- 优化策略：
--   1. 使用小时聚合表而非原始事件表
--   2. ClickHouse 主键排序与查询模式匹配
--   3. 预计算物化视图

-- 查询 1：实验概览（P95 < 500ms）
SELECT
    variant,
    metric_name,
    sum(event_count) as total_n,
    sum(sum_value) / sum(event_count) as mean,
    sqrt(sum(sum_sq_value) / sum(event_count) -
         pow(sum(sum_value) / sum(event_count), 2)) as pooled_stddev
FROM experiment_metric_hourly
WHERE experiment_id = 'exp_20240315_btn_color'
  AND window_start >= '2024-03-15'
GROUP BY variant, metric_name;

-- 查询 2：时间趋势（P95 < 1s）
SELECT
    window_start,
    variant,
    metric_name,
    mean_value,
    event_count
FROM experiment_metric_hourly
WHERE experiment_id = 'exp_20240315_btn_color'
  AND metric_name = 'conversion_rate'
ORDER BY window_start, variant;

-- 查询 3：分层分析（辛普森悖论检测用，P95 < 2s）
SELECT
    variant,
    arrayJoin(mapKeys(device_dist)) as device_type,
    mapValues(device_dist)[indexOf(mapKeys(device_dist), device_type)] as count
FROM experiment_metric_aggregates
WHERE experiment_id = 'exp_20240315_btn_color'
  AND metric_name = 'conversion_rate'
  AND window_start >= now() - INTERVAL 7 DAY;
```

## 常见陷阱（深度分析）

### 陷阱 1：偷看后提前停止

**后果：** 假阳性率从 5% 飙升到 30%+。10 个实验中 3 个假阳性 → 每个假阳性可能浪费 2-3 个月开发资源 → 年损失数百万元。

**解决方案：** 序贯检验（OBF 边界），或严格等到预设实验天数结束。平台层面：实验结果页面在未达到预设天数前显示"实验进行中，当前结果仅供参考"。

### 陷阱 2：样本量不足就下结论

**场景：** 基线 5%，MDE 10%，需要每组 31000 人。但只跑了 3 天（每组 3000 人）→ 统计功效仅 35% → "无显著差异"的结论不可信（可能是功效不足，而非真的无差异）。

**解决方案：** 实验前计算样本量，未达到前不下结论。平台显示"当前统计功效 35%，预计还需 4 天达到 80%"。

### 陷阱 3：辛普森悖论

**场景：** 新推荐算法 A/B 测试 → 整体 A 好（转化率 4.4% vs 6.8%）→ 但分设备看 B 好 → 如果 80% 用户用手机 → 应该选 B。

**解决方案：** 自动分层分析 + 悖论检测。按设备、地区、用户分层分群分析，不只看整体。

### 陷阱 4：不校正多重比较

**后果：** 10 个指标 × p < 0.05 → 至少 1 个假阳性概率 40%。错误结论 → 浪费开发资源。

**解决方案：** Bonferroni（指标 3-10 个）或 FDR（指标 10+ 个）。

### 陷阱 5：SRM（样本比例偏差）

**场景：** 实验配置 50/50 分流，但实际 control=55000 人，treatment=45000 人 → 分流有偏差 → 实验结果不可信（可能某种用户更容易被分到某组）。

**解决方案：** 自动 SRM 检测（卡方检验，p < 0.01 告警）。检测到 SRM → 实验结果标记为"不可信"。

### 陷阱 6：实验间干扰

**场景：** 用户同时在实验 A（新按钮）和实验 B（新推荐）的实验组 → 无法区分哪个改动导致的效果。

**解决方案：** 实验互斥组——同一互斥组内的实验不能同时作用于同一用户。或使用正交实验设计。

## 延伸思考

- **CUPED**：利用实验前的基线数据降低方差，减少所需样本量 30-50%。原理：用协变量（用户实验前的行为）调整指标值，消除个体差异导致的方差。
- **交错实验**：同一用户同时看到 A 和 B 的结果（如搜索结果混合 A/B 排序），更精确但更复杂。
- **多臂老虎机**：动态调整流量分配，表现好的组获得更多流量，减少实验成本。适合"找到最优策略"而非"验证假设"的场景。
- **贝叶斯 A/B 测试**：计算 P(B>A) 而非 p-value，更直观。但需要指定先验分布，且结果解释更容易被误用。
## 性能与成本分析（完整版）

### 基础设施规模估算（50+ 并发实验）

| 组件 | 规格 | 数量 | 月成本 | 说明 |
|------|------|------|-------|------|
| 实验服务 | 4c8G | 5 台 | ¥2 万 | 分组查询 + 实验管理 |
| Kafka（事件流） | 8c32G + 1TB | 6 节点 | ¥3 万 | 3 broker + 3 mirror |
| Flink（指标聚合） | 4c8G | 3 台 | ¥1.5 万 | 2 TM + 1 JM |
| ClickHouse（实验数据） | 8c64G + 2TB SSD | 3 节点 | ¥3 万 | 2 副本 + 1 备份 |
| Redis（用户分组缓存） | 8c16G | 3 节点 | ¥1 万 | 哨兵模式 |
| MySQL（元数据） | 4c16G + 500G SSD | 2 节点 | ¥0.8 万 | 主从 |
| **合计** | | | **¥11.3 万** | |

### 关键性能指标

| 指标 | 目标 | 实际 | 瓶颈分析 |
|------|------|------|---------|
| 用户分组延迟 P50 | < 2ms | 0.8ms | Redis 缓存命中时 |
| 用户分组延迟 P99 | < 5ms | 3.2ms | Redis miss → DB 查询 |
| 指标聚合延迟 | < 5min | 2-3min | Flink 窗口聚合 |
| 实验看板查询 P95 | < 3s | 1.2s | ClickHouse 小时聚合表 |
| 分层分析查询 P95 | < 5s | 3.5s | 分钟聚合表 + map 展开 |
| 分组一致性 | 99.99% | 99.997% | DB 持久化 + Redis 缓存 |
| 并发实验数 | 50+ | 80+ | 单用户最多 5 个实验 |

### 指标计算成本分析

不同统计检验的计算复杂度与耗时差异显著，在大规模实验场景下需要关注：

| 检验类型 | 计算复杂度 | 单次耗时(10万样本) | 50实验并行耗时 | 适用场景 |
|---------|----------|-----------------|-------------|---------|
| 比例 Z 检验 | O(1) | 0.01ms | 0.5ms | 转化率、点击率 |
| Welch t 检验 | O(1) | 0.02ms | 1ms | 停留时长、客单价 |
| Mann-Whitney U | O(n log n) | 15ms | 750ms | 非正态连续指标 |
| 序贯检验(OBF) | O(1) | 0.03ms | 1.5ms | 防偷看场景 |
| Alpha 消费函数 | O(k) | 0.05ms | 2.5ms | 灵活查看场景 |
| 始终有效 p 值 | O(1) 增量 | 0.02ms | 1ms | 实时监控 |
| Bonferroni 校正 | O(m) | 0.01ms | 0.5ms | m=指标数 |
| BH-FDR 校正 | O(m log m) | 0.02ms | 1ms | m=指标数 |
| 辛普森悖论检测 | O(d*k) | 5ms | 250ms | d=维度数,k=子群数 |
| SRM 卡方检验 | O(1) | 0.01ms | 0.5ms | 分组偏差检测 |

**关键发现：** 序贯检验和多重比较校正的计算开销极低（微秒级），真正的性能瓶颈在数据聚合（ClickHouse 查询）而非统计计算。辛普森悖论检测因为需要多维度分层查询，耗时相对较高但仍可接受。

### 存储成本分析

| 数据类型 | 日增量 | 月增量 | 单行大小 | 月存储量 | 保留策略 | 存储成本/月 |
|---------|--------|--------|---------|---------|---------|----------|
| 用户分组记录 | 1 亿行 | 30 亿行 | ~80B | ~240GB | 实验结束归档 | ¥50 |
| 分钟聚合结果 | 500 万行 | 1.5 亿行 | ~200B | ~30GB | 保留 90 天 | ¥6 |
| 小时聚合结果 | 80 万行 | 2400 万行 | ~150B | ~3.6GB | 保留 1 年 | ¥1 |
| 统计检验结果 | 5 万行 | 150 万行 | ~500B | ~0.75GB | 永久 | ¥0.2 |
| 原始事件(Kafka) | 10 亿行 | 300 亿行 | ~300B | ~9TB | 保留 7 天 | ¥180 |
| **合计** | | | | **~1.3TB(持久)** | | **~¥240** |

**存储优化策略：**

| 策略 | 效果 | 实现方式 |
|------|------|---------|
| Kafka LZ4 压缩 | 原始事件体积减少 60% | producer 端压缩 |
| ClickHouse 列存压缩 | 聚合数据体积减少 70-80% | LZ4 默认压缩 |
| 归档冷数据 | 活跃数据量减少 90% | 实验结束 → 对象存储 |
| 分区裁剪 | 查询扫描数据量减少 95% | 按月分区 + 时间过滤 |

### 查询延迟分析

| 查询类型 | 扫描行数 | 耗时 P50 | 耗时 P95 | 优化手段 |
|---------|---------|---------|---------|---------|
| 实验概览(小时表) | ~1000 行 | 50ms | 200ms | 主键排序 + 分区裁剪 |
| 时间趋势(小时表) | ~500 行 | 30ms | 100ms | 索引命中 |
| 分层分析(分钟表) | ~5 万行 | 500ms | 2s | map 展开 + 物化视图 |
| SRM 检测(分组表) | ~50 万行 | 200ms | 800ms | 索引 + 缓存 |
| 全局实验列表 | ~100 行 | 10ms | 50ms | Redis 缓存 |
| 序贯检验历史 | ~50 行 | 5ms | 20ms | 主键查询 |

**查询优化关键原则：**
1. 永远不在原始事件表上做分析查询——用聚合表
2. ClickHouse 排序键必须与查询的 WHERE + GROUP BY 匹配
3. 分层分析查询使用物化视图预计算，避免运行时 map 展开
4. SRM 检测结果缓存 10 分钟，避免高频重复查询

### CUPED 方差缩减效果

| 指标 | 原始方差 | CUPED 后方差 | 样本量缩减 | 实验周期缩短 | 协变量 |
|------|---------|------------|-----------|------------|-------|
| 转化率 | 0.21 | 0.12 | 43% | 4周→2.3周 | 实验 前 14 天转化率 |
| 点击率 | 0.18 | 0.08 | 56% | 3周→1.3周 | 实验 前 14 天点击率 |
| 停留时长 | 0.35 | 0.22 | 37% | 6周→3.8周 | 实验 前 14 天停留时长 |
| 客单价 | 0.42 | 0.30 | 29% | 5周→3.5周 | 实验 前 14 天客单价 |

CUPED 的计算公式：Y_cuped = Y - theta * (X - E[X])，其中 theta = Cov(Y,X) / Var(X)，X 是实验前协变量，Y 是实验期指标。

## 异常场景完整演练

### 场景 1：样本比例失调（SRM）

**触发条件：** 实验 A 组 50 万用户，B 组 30 万用户（期望 50:50）

**原因分析：**
- 分组逻辑 Bug（某些用户总是分到 A 组）
- 流量分配被其他实验挤占
- 用户换设备导致分组丢失（5% 换设备率）

```python
from scipy.stats import chi2
from dataclasses import dataclass


@dataclass
class SRMResult:
    """SRM 检测结果"""
    experiment_id: str
    expected_ratio: float        # 期望的对照组比例
    control_n: int               # 对照组实际样本量
    treatment_n: int             # 实验组实际样本量
    actual_ratio: float          # 实际对照组比例
    chi2_statistic: float        # 卡方统计量
    p_value: float               # p 值
    is_detected: bool            # 是否检测到 SRM
    severity: str                # "CRITICAL" / "WARNING" / "OK"
    diagnosis: str               # 诊断信息


class SRMDetector:
    """
    SRM（Sample Ratio Mismatch）检测器

    原理：卡方检验比较实际样本比例与期望比例。
    如果 p < 0.01，说明分组有系统性偏差，实验结果不可信。

    常见 SRM 原因：
    1. 分组逻辑 Bug（某些 user_id 格式总是分到同一组）
    2. 流量被其他实验挤占（互斥组配置错误）
    3. 用户换设备导致分组丢失
    4. 重定向/爬虫流量不均匀
    5. 缓存不一致（Redis 和 DB 不同步）
    """

    def __init__(self, db_connection, alert_service):
        self.db = db_connection
        self.alert = alert_service

    def check(self, experiment_id: str) -> SRMResult:
        """执行 SRM 检测"""
        experiment = self._get_experiment(experiment_id)
        counts = self.db.query("""
            SELECT group_name, COUNT(*) as n
            FROM user_assignments
            WHERE experiment_id = %s
            GROUP BY group_name
        """, experiment_id)

        control_n = next((r.n for r in counts
                         if r.group_name == "control"), 0)
        treatment_n = next((r.n for r in counts
                          if r.group_name == "treatment"), 0)
        total_n = control_n + treatment_n

        if total_n == 0:
            return SRMResult(
                experiment_id=experiment_id,
                expected_ratio=0.5, control_n=0, treatment_n=0,
                actual_ratio=0, chi2_statistic=0, p_value=1.0,
                is_detected=False, severity="OK",
                diagnosis="无样本数据")

        expected_ratio = experiment.variants[0].ratio
        actual_ratio = control_n / total_n

        # 卡方检验
        expected_ctrl = total_n * expected_ratio
        expected_treat = total_n * (1 - expected_ratio)
        chi2_stat = (
            (control_n - expected_ctrl) ** 2 / expected_ctrl +
            (treatment_n - expected_treat) ** 2 / expected_treat
        )
        p_value = 1 - chi2.cdf(chi2_stat, df=1)

        is_detected = p_value < 0.01
        if p_value < 0.001:
            severity = "CRITICAL"
        elif p_value < 0.01:
            severity = "WARNING"
        else:
            severity = "OK"

        diagnosis = self._diagnose(
            experiment_id, control_n, treatment_n,
            expected_ratio, actual_ratio, p_value)

        if is_detected:
            self.alert.send(
                title=f"SRM 检测告警: {experiment_id}",
                message=(f"对照 {control_n} vs 实验组 {treatment_n}, "
                        f"期望 {expected_ratio:.2f}, "
                        f"实际 {actual_ratio:.2f}, p={p_value:.6f}"),
                severity=severity)

        return SRMResult(
            experiment_id=experiment_id,
            expected_ratio=expected_ratio,
            control_n=control_n, treatment_n=treatment_n,
            actual_ratio=actual_ratio, chi2_statistic=chi2_stat,
            p_value=p_value, is_detected=is_detected,
            severity=severity, diagnosis=diagnosis)

    def _diagnose(self, experiment_id, ctrl_n, treat_n,
                  expected, actual, p_value):
        """诊断 SRM 原因"""
        parts = []
        deviation = abs(actual - expected)
        if deviation > 0.10:
            parts.append("偏差>10%，极可能是分组逻辑 Bug")
            bucket_skew = self._check_bucket_distribution(experiment_id)
            if bucket_skew:
                parts.append(f"哈希桶分布不均: {bucket_skew}")
        elif deviation > 0.05:
            parts.append("偏差5-10%，可能是流量挤占或换设备")
            conflict = self._check_mutex_conflicts(experiment_id)
            if conflict:
                parts.append(f"互斥实验冲突: {conflict}")
        else:
            parts.append("偏差<5%，可能是随机波动")
        return "; ".join(parts)

    def _check_bucket_distribution(self, experiment_id):
        """检查哈希桶分布是否均匀"""
        buckets = self.db.query("""
            SELECT hash_bucket %% 100 as bucket_range, COUNT(*) as n
            FROM user_assignments WHERE experiment_id = %s
            GROUP BY hash_bucket %% 100 ORDER BY bucket_range
        """, experiment_id)
        if not buckets:
            return None
        counts = [b.n for b in buckets]
        mean_c = sum(counts) / len(counts)
        max_dev = max(abs(c - mean_c) / mean_c for c in counts)
        return f"最大偏差 {max_dev:.1%}" if max_dev > 0.1 else None

    def _check_mutex_conflicts(self, experiment_id):
        """检查互斥实验冲突"""
        conflicts = self.db.query("""
            SELECT e2.id as conflict_id, COUNT(*) as overlap
            FROM user_assignments u1
            JOIN user_assignments u2 ON u1.user_id = u2.user_id
            JOIN experiments e2 ON u2.experiment_id = e2.id
            WHERE u1.experiment_id = %s AND u2.experiment_id != %s
              AND u1.group_name != 'control' AND u2.group_name != 'control'
              AND e2.mutex_group_id = (
                  SELECT mutex_group_id FROM experiments WHERE id = %s)
            GROUP BY e2.id
        """, experiment_id, experiment_id, experiment_id)
        if conflicts:
            return [f"{c.conflict_id}({c.overlap}人)" for c in conflicts]
        return None

    def _get_experiment(self, experiment_id):
        return self.db.query_one(
            "SELECT * FROM experiments WHERE id = %s", experiment_id)
```

**SRM 检测的处理流程：**

```
SRM 检测(p < 0.01) → 自动告警 → 暂停数据分析 → 排查原因 → 修复 → 重新开始实验
                                                          |
                                              +-----------+-----------+
                                              分组Bug     流量挤占    换设备
                                              修复代码   调整互斥组  粘性桶
```

### 场景 2：指标管道延迟导致错误决策

**触发条件：** Flink 聚合延迟 2 小时 → 看板显示"新方案转化率 -5%"，但实际缺了最新 2 小时的高转化时段数据 → 2 小时后数据补齐 → 新方案转化率 +3%

```python
from datetime import datetime
from dataclasses import dataclass


@dataclass
class DataFreshness:
    """数据新鲜度状态"""
    experiment_id: str
    latest_data_time: datetime
    current_time: datetime
    delay_minutes: float
    is_stale: bool
    severity: str  # "FRESH" / "STALE" / "CRITICALLY_STALE"


class PipelineDelayMonitor:
    """
    指标管道延迟监控器

    监控两个指标：
    1. 数据新鲜度：最新数据时间 vs 当前时间的差距
    2. 样本量异常：今日样本量与昨日同期的偏差

    当延迟 > 30 分钟 → 标记看板"数据延迟中，请勿做决策"
    当延迟 > 4 小时 → 人工介入排查
    """

    def __init__(self, clickhouse_client, alert_service):
        self.ch = clickhouse_client
        self.alert = alert_service

    def check_freshness(self, experiment_id: str) -> DataFreshness:
        """检查数据新鲜度"""
        row = self.ch.query_one("""
            SELECT max(window_end) as latest_data_time
            FROM experiment_metric_aggregates
            WHERE experiment_id = %(exp_id)s
        """, exp_id=experiment_id)

        now = datetime.now()
        latest = row['latest_data_time'] if row and row['latest_data_time'] else None

        if latest is None:
            return DataFreshness(
                experiment_id=experiment_id, latest_data_time=None,
                current_time=now, delay_minutes=float('inf'),
                is_stale=True, severity="CRITICALLY_STALE")

        delay = (now - latest).total_seconds() / 60
        is_stale = delay > 30
        if delay <= 30:
            severity = "FRESH"
        elif delay <= 240:
            severity = "STALE"
        else:
            severity = "CRITICALLY_STALE"

        return DataFreshness(
            experiment_id=experiment_id, latest_data_time=latest,
            current_time=now, delay_minutes=delay,
            is_stale=is_stale, severity=severity)

    def check_sample_anomaly(self, experiment_id: str) -> dict:
        """检查样本量异常（与昨日同期对比）"""
        row = self.ch.query_one("""
            SELECT
                sum(CASE WHEN toDateTime(window_start) >= now() - INTERVAL 1 HOUR
                    THEN event_count ELSE 0 END) as current_hour_n,
                sum(CASE WHEN toDateTime(window_start) >= now() - INTERVAL 25 HOUR
                     AND toDateTime(window_start) < now() - INTERVAL 24 HOUR
                    THEN event_count ELSE 0 END) as yesterday_same_hour_n
            FROM experiment_metric_aggregates
            WHERE experiment_id = %(exp_id)s
        """, exp_id=experiment_id)

        current_n = row['current_hour_n'] or 0
        yesterday_n = row['yesterday_same_hour_n'] or 0

        if yesterday_n == 0:
            return {"anomaly_detected": False, "reason": "无昨日对比数据"}

        deviation = (current_n - yesterday_n) / yesterday_n
        anomaly_detected = abs(deviation) > 0.20

        return {
            "anomaly_detected": anomaly_detected,
            "current_hour_n": current_n,
            "yesterday_same_hour_n": yesterday_n,
            "deviation": deviation,
            "reason": f"样本量偏差 {deviation:.1%}" if anomaly_detected else "正常",
        }

    def get_dashboard_status(self, experiment_id: str) -> dict:
        """
        获取看板展示状态

        - FRESH: 正常展示所有数据
        - STALE: 标记"数据延迟中，请勿做决策"
        - CRITICALLY_STALE: 隐藏统计检验结果，只显示原始数据
        """
        freshness = self.check_freshness(experiment_id)
        sample_check = self.check_sample_anomaly(experiment_id)

        if freshness.severity == "FRESH" and not sample_check["anomaly_detected"]:
            return {
                "status": "NORMAL", "message": None,
                "show_statistical_results": True,
                "data_as_of": freshness.latest_data_time,
            }
        elif freshness.severity == "STALE":
            return {
                "status": "DATA_DELAYED",
                "message": (f"数据延迟 {freshness.delay_minutes:.0f} 分钟，"
                           "当前结果可能不完整，请勿做决策"),
                "show_statistical_results": True,
                "data_as_of": freshness.latest_data_time,
                "delay_minutes": freshness.delay_minutes,
            }
        elif freshness.severity == "CRITICALLY_STALE":
            self.alert.send(
                title=f"数据严重延迟: {experiment_id}",
                message=f"数据延迟 {freshness.delay_minutes:.0f} 分钟",
                severity="CRITICAL")
            return {
                "status": "CRITICALLY_DELAYED",
                "message": (f"数据严重延迟 {freshness.delay_minutes:.0f} 分钟，"
                           "统计检验结果已隐藏，请联系数据团队"),
                "show_statistical_results": False,
                "data_as_of": freshness.latest_data_time,
                "delay_minutes": freshness.delay_minutes,
            }

        if sample_check["anomaly_detected"]:
            return {
                "status": "SAMPLE_ANOMALY",
                "message": f"样本量异常: {sample_check['reason']}",
                "show_statistical_results": True,
                "data_as_of": freshness.latest_data_time,
            }

        return {"status": "NORMAL", "message": None,
                "show_statistical_results": True,
                "data_as_of": freshness.latest_data_time}
```

**管道延迟的处理流程：**

```
Flink 延迟 → 数据新鲜度检测 → 看板标记 → 等待恢复 / 人工介入
                |
    +-----------+-----------+
    < 30min    30min-4h      > 4h
    正常显示   标记警告     隐藏统计结果 + 告警
```

### 场景 3：实验碰撞（用户同时被分配到冲突实验）

**触发条件：** 用户同时被分配到"新推荐算法"实验和"新 UI 布局"实验 → 两个实验效果互相干扰

```python
from dataclasses import dataclass
from typing import List


@dataclass
class ExperimentCollision:
    """实验碰撞检测结果"""
    user_id: str
    experiment_ids: List[str]
    mutex_group_id: str
    collision_type: str  # "SAME_GROUP_TREATMENT" / "CROSS_GROUP_INTERFERENCE"


class ExperimentCollisionGuard:
    """
    实验碰撞检测与防护

    两种碰撞类型：
    1. 同互斥组碰撞：同一互斥组的两个实验同时作用于用户 → 严格禁止
    2. 跨组干扰：不同互斥组的实验可能有隐性交互 → 警告

    防护策略：
    1. 实验创建时必须声明互斥组
    2. 分组时检查互斥组，同一组内只分配一个实验
    3. 已分配到互斥实验 → 从优先级较低的实验中移除
    """

    def __init__(self, db_connection, alert_service, redis_client):
        self.db = db_connection
        self.alert = alert_service
        self.redis = redis_client

    def assign_with_collision_guard(self, user_id: str,
                                     experiment_id: str) -> str:
        """带碰撞防护的分组分配"""
        experiment = self._get_experiment(experiment_id)
        active_experiments = self._get_user_active_experiments(user_id)
        collision = self._check_mutex_collision(
            user_id, experiment_id, experiment.mutex_group_id,
            active_experiments)

        if collision:
            return self._resolve_collision(
                user_id, experiment_id, collision, active_experiments)

        return self._normal_assign(user_id, experiment_id)

    def _check_mutex_collision(self, user_id, experiment_id,
                                mutex_group_id, active_experiments):
        """检查互斥组碰撞"""
        if mutex_group_id is None:
            return None
        for active_exp in active_experiments:
            if (active_exp.mutex_group_id == mutex_group_id and
                active_exp.experiment_id != experiment_id and
                active_exp.group_name != "control"):
                return ExperimentCollision(
                    user_id=user_id,
                    experiment_ids=[active_exp.experiment_id, experiment_id],
                    mutex_group_id=mutex_group_id,
                    collision_type="SAME_GROUP_TREATMENT")
        return None

    def _resolve_collision(self, user_id, new_experiment_id,
                           collision, active_experiments) -> str:
        """解决碰撞：优先级低的实验让位"""
        new_exp = self._get_experiment(new_experiment_id)
        existing_exp_id = collision.experiment_ids[0]
        existing_exp = self._get_experiment(existing_exp_id)

        if new_exp.priority < existing_exp.priority:
            # 新实验更重要 → 将用户从旧实验移除
            self._remove_from_experiment(user_id, existing_exp_id)
            self.alert.send(
                title=f"实验碰撞解决: {user_id}",
                message=(f"用户从实验 {existing_exp_id} 移除，"
                        f"优先分配到更高优先级实验 {new_experiment_id}"),
                severity="INFO")
            return self._normal_assign(user_id, new_experiment_id)
        else:
            # 旧实验更重要 → 新实验对用户分配为对照组
            self.alert.send(
                title=f"实验碰撞解决: {user_id}",
                message=(f"用户在互斥实验 {existing_exp_id} 中，"
                        f"对 {new_experiment_id} 分配为对照组"),
                severity="INFO")
            return "control"

    def _remove_from_experiment(self, user_id, experiment_id):
        """将用户从实验中移除（按对照组处理）"""
        self.db.execute("""
            UPDATE user_assignments SET group_name = 'control'
            WHERE user_id = %s AND experiment_id = %s
        """, user_id, experiment_id)
        self.redis.delete(f"ab:{user_id}:{experiment_id}")

    def _get_user_active_experiments(self, user_id):
        """查询用户当前参与的所有活跃实验"""
        return self.db.query("""
            SELECT ua.experiment_id, ua.group_name, e.mutex_group_id
            FROM user_assignments ua
            JOIN experiments e ON ua.experiment_id = e.id
            WHERE ua.user_id = %s AND e.status = 'RUNNING'
        """, user_id)

    def _normal_assign(self, user_id, experiment_id):
        """正常分组（确定性哈希）"""
        import mmh3
        experiment = self._get_experiment(experiment_id)
        hash_val = mmh3.hash(f"{user_id}:{experiment_id}") % 10000
        cumulative = 0
        for variant in experiment.variants:
            cumulative += variant.ratio * 10000
            if hash_val < cumulative:
                return variant.name
        return experiment.variants[-1].name

    def _get_experiment(self, experiment_id):
        return self.db.query_one(
            "SELECT * FROM experiments WHERE id = %s", experiment_id)

    def batch_check_collisions(self, experiment_ids: List[str]) -> dict:
        """
        批量检查实验间的碰撞（实验创建时使用）

        在实验创建阶段就发现潜在碰撞，而非等到用户分组时。
        """
        collisions = []
        experiments = [self._get_experiment(eid) for eid in experiment_ids]
        for i, exp_a in enumerate(experiments):
            for j, exp_b in enumerate(experiments):
                if i >= j:
                    continue
                if (exp_a.mutex_group_id and
                    exp_a.mutex_group_id == exp_b.mutex_group_id):
                    collisions.append({
                        "exp_a": exp_a.id,
                        "exp_b": exp_b.id,
                        "type": "SAME_MUTEX_GROUP",
                        "mutex_group_id": exp_a.mutex_group_id,
                        "resolution": "互斥组内只能同时运行一个实验",
                    })
        return {
            "total_experiments": len(experiment_ids),
            "collision_count": len(collisions),
            "collisions": collisions,
        }
```

**实验碰撞的防护层级：**

| 防护层级 | 时机 | 机制 | 覆盖率 |
|---------|------|------|-------|
| 实验创建 | 创建时 | 互斥组声明 + 批量碰撞检查 | 100% 可预知碰撞 |
| 用户分组 | 分配时 | 互斥组实时检查 + 优先级让位 | 100% 运行时碰撞 |
| 事后分析 | 分析时 | 互斥实验用户标记 + 排除分析 | 补偿漏网碰撞 |

### 场景 4：辛普森悖论检测

**触发条件：** 整体看新方案转化率 +2%（显著），但分 iOS/Android 看：iOS 上新方案 -1%（不显著），Android 上新方案 +5%（显著）→ 整体提升完全来自 Android，iOS 实际更差

```python
class SimpsonParadoxHandler:
    """
    辛普森悖论完整处理器：检测 → 告警 → 决策建议

    与 SimpsonParadoxDetector 不同，本类关注完整的处理流程，
    包括看板标记、决策建议和上线策略推荐。
    """

    def __init__(self, detector, alert_service, dashboard_service):
        self.detector = detector
        self.alert = alert_service
        self.dashboard = dashboard_service

    def handle(self, experiment_id: str) -> dict:
        """完整的辛普森悖论处理流程"""
        # 1. 检测
        paradoxes = self.detector.detect(experiment_id)

        if not paradoxes:
            return {
                "status": "NO_PARADOX",
                "message": "未检测到辛普森悖论",
                "action": "可正常使用整体结论",
            }

        # 2. 找到最严重的悖论
        worst = max(paradoxes, key=lambda p: self._severity_score(p))

        # 3. 更新看板标记
        self.dashboard.add_warning(
            experiment_id=experiment_id,
            warning_type="SIMPSON_PARADOX",
            message=f"子组分析发现不一致效果（{worst.dimension} 维度）",
            details={
                "overall_direction": worst.overall_direction,
                "contradictory_strata": [
                    {"name": s.stratum_name, "direction": s.effect_direction,
                     "weight": s.weight_in_total}
                    for s in worst.contradictory_strata
                ],
            },
        )

        # 4. 生成决策建议
        decision = self._generate_decision(experiment_id, worst)

        # 5. 高严重度告警
        if worst.paradox_severity == "HIGH":
            self.alert.send(
                title=f"辛普森悖论告警: {experiment_id}",
                message=(f"实验 {experiment_id} 在 {worst.dimension} 维度"
                        f"检测到严重辛普森悖论，整体结论不可信。"
                        f"{worst.explanation}"),
                severity="HIGH")

        return {
            "status": "PARADOX_DETECTED",
            "severity": worst.paradox_severity,
            "dimension": worst.dimension,
            "explanation": worst.explanation,
            "recommended_action": worst.recommended_action,
            "decision": decision,
        }

    def _severity_score(self, paradox):
        scores = {"HIGH": 3, "MEDIUM": 2, "LOW": 1}
        return scores.get(paradox.paradox_severity, 0)

    def _generate_decision(self, experiment_id, paradox):
        """生成决策建议"""
        if paradox.paradox_severity == "HIGH":
            positive_strata = [
                s for s in paradox.contradictory_strata
                if s.effect_direction == "POSITIVE"
            ]
            negative_strata = [
                s for s in paradox.contradictory_strata
                if s.effect_direction == "NEGATIVE"
            ]
            strategies = []
            if positive_strata:
                platforms = [s.stratum_name for s in positive_strata]
                strategies.append({
                    "strategy": "PARTIAL_ROLLOUT",
                    "description": f"仅对正向子群上线: {', '.join(platforms)}",
                    "risk": "中（需监控未上线子群的指标退化）",
                })
            if negative_strata:
                platforms = [s.stratum_name for s in negative_strata]
                strategies.append({
                    "strategy": "NO_ROLLOUT",
                    "description": f"不上线，因为 {', '.join(platforms)} 效果为负",
                    "risk": "低（最保守）",
                })
            return {
                "can_full_rollout": False,
                "strategies": strategies,
                "warning": "整体结论不可信，必须基于子群分析做决策",
            }
        elif paradox.paradox_severity == "MEDIUM":
            return {
                "can_full_rollout": True,
                "conditions": [
                    "全量上线后必须监控各子群指标",
                    "如果负向子群效果持续恶化 → 回滚",
                    "2 周后复查子群数据",
                ],
                "strategies": [{
                    "strategy": "FULL_ROLLOUT_WITH_MONITORING",
                    "description": "全量上线但需重点监控子群指标",
                    "risk": "中",
                }],
            }
        else:
            return {
                "can_full_rollout": True,
                "conditions": ["在报告中注明子群差异"],
                "strategies": [{
                    "strategy": "FULL_ROLLOUT",
                    "description": "悖论轻微，可正常上线",
                    "risk": "低",
                }],
            }
```

**辛普森悖论的处理决策树：**

```
检测到悖论
    |
    +-- 严重程度 HIGH → 不全量上线
    |       |
    |       +-- 有正向子群 → 分平台上线（仅正向子群）
    |       |
    |       +-- 无正向子群 → 不上线
    |
    +-- 严重程度 MEDIUM → 可上线，但需监控
    |       |
    |       +-- 全量上线 + 重点监控子群
    |       +-- 2 周后复查
    |       +-- 子群恶化 → 回滚
    |
    +-- 严重程度 LOW → 正常上线
            |
            +-- 报告中注明子群差异
```

### 始终有效 p 值（Always-Valid P-Value）完整实现

传统的 p 值只在固定样本量下有效——如果反复偷看数据，p 值会越来越小，导致假阳性率飙升。始终有效 p 值（Always-Valid P-Value）通过将累积信息映射为一个安全的检验统计量，使得无论在何时查看数据，p 值都保持有效。

```python
import math
from typing import List, Tuple, Optional
from dataclasses import dataclass

@dataclass
class AlwaysValidTestResult:
    """始终有效 p 值检验结果"""
    p_value: float              # 始终有效 p 值
    is_significant: bool        # 是否显著
    info_fraction: float        # 当前信息分数
    cumulative_s: float         # 累积检验统计量
    recommendation: str         # 决策建议
    can_stop: bool              # 是否可以停止实验

class AlwaysValidPValueTester:
    """
    始终有效 p 值检验器

    核心原理：
    基于 mixture sequential probability ratio test (mSPRT)。
    定义检验统计量 S_n = ∏(1..n) (1 + λ_i * (X_i - μ_0))，
    其中 λ_i 是预测因子，μ_0 是原假设下的期望值。
    始终有效 p 值定义为 p_n = min(1/S_n, 1)。

    在原假设下，E[1/S_n] ≤ 1 对所有 n 成立，
    因此 P(存在 n 使得 p_n ≤ α) ≤ α，即无论何时查看，假阳性率都被控制。

    实际使用的简化形式（基于 Hoeffding 不等式）：
    p_n = 2 * exp(-n * δ² / (2 * σ²))
    其中 δ = |x̄_n - μ_0|，σ² 是方差。

    但这过于保守。实践中更常用的是基于对数似然比的 mSPRT：
    使用混合正态先验的 mSPRT，计算始终有效 p 值。
    """

    def __init__(self, alpha: float = 0.05, prior_variance: float = 1.0,
                 max_observations: int = 100000):
        self.alpha = alpha
        self.prior_variance = prior_variance  # 混合先验的方差参数
        self.max_observations = max_observations

    def compute_p_value(self, observations: List[float],
                        null_mean: float = 0.0) -> AlwaysValidTestResult:
        """
        计算始终有效 p 值

        使用 mixture SPRT 方法（基于正态混合先验）

        参数：
          observations: 到当前为止的所有观测值
          null_mean: 原假设下的期望值

        返回：始终有效检验结果
        """
        n = len(observations)
        if n == 0:
            return AlwaysValidTestResult(
                p_value=1.0, is_significant=False,
                info_fraction=0.0, cumulative_s=1.0,
                recommendation="WAIT", can_stop=False
            )

        # 计算样本均值
        sample_mean = sum(observations) / n

        # 计算样本方差（用于标准化）
        if n > 1:
            sample_var = sum((x - sample_mean)**2 for x in observations) / (n - 1)
        else:
            sample_var = 1.0  # 单个观测点，使用默认方差

        # 偏差量
        delta = sample_mean - null_mean

        # 方法1：基于对数似然比的 mSPRT
        # 使用正态混合先验 N(0, prior_variance)
        # 始终有效 p 值 = 1 / S_n
        # S_n = sqrt(prior_variance / (n + prior_variance)) *
        #       exp(n * delta^2 / (2 * (n + prior_variance) * sample_var))

        # 注意：当 sample_var 很小时，需要加下限避免数值溢出
        safe_var = max(sample_var, 1e-10)

        # 对数 S_n
        log_S_n = (
            0.5 * math.log(self.prior_variance / (n + self.prior_variance)) +
            n * delta**2 / (2 * (n + self.prior_variance) * safe_var)
        )

        # S_n = exp(log_S_n)
        S_n = math.exp(min(log_S_n, 700))  # 防止溢出

        # 始终有效 p 值
        always_valid_p = min(1.0 / max(S_n, 1.0), 1.0)

        # 信息分数
        info_fraction = min(n / self.max_observations, 1.0)

        # 判定
        is_significant = always_valid_p < self.alpha
        can_stop = is_significant and info_fraction >= 0.3  # 至少 30% 信息量

        # 决策建议
        if can_stop:
            recommendation = "REJECT_H0_EARLY"
        elif info_fraction >= 1.0 and is_significant:
            recommendation = "REJECT_H0"
        elif info_fraction >= 1.0 and not is_significant:
            recommendation = "FAIL_TO_REJECT"
        else:
            recommendation = "CONTINUE"

        return AlwaysValidTestResult(
            p_value=always_valid_p,
            is_significant=is_significant,
            info_fraction=info_fraction,
            cumulative_s=S_n,
            recommendation=recommendation,
            can_stop=can_stop,
        )

    def compute_p_value_ratio_metric(
        self,
        control_conversions: int, control_total: int,
        treatment_conversions: int, treatment_total: int,
        null_diff: float = 0.0
    ) -> AlwaysValidTestResult:
        """
        针对比率型指标（如转化率）的始终有效 p 值

        使用双样本版本：将两组的差异序列作为观测值

        参数：
          control_conversions: 对照组转化数
          control_total: 对照组总用户数
          treatment_conversions: 实验组转化数
          treatment_total: 实验组总用户数
          null_diff: 原假设下的差异（默认 0 = 无差异）
        """
        p_control = control_conversions / max(control_total, 1)
        p_treatment = treatment_conversions / max(treatment_total, 1)
        delta = p_treatment - p_control - null_diff

        # 合并方差估计
        p_pool = (control_conversions + treatment_conversions) / max(control_total + treatment_total, 1)
        se = math.sqrt(p_pool * (1 - p_pool) * (1/max(control_total, 1) + 1/max(treatment_total, 1)))

        if se < 1e-10:
            return AlwaysValidTestResult(
                p_value=1.0, is_significant=False,
                info_fraction=0.5, cumulative_s=1.0,
                recommendation="CONTINUE", can_stop=False
            )

        # Z 统计量
        n_effective = (control_total * treatment_total) / max(control_total + treatment_total, 1)

        # 使用始终有效框架
        log_S_n = (
            0.5 * math.log(self.prior_variance / (n_effective + self.prior_variance)) +
            n_effective * delta**2 / (2 * (n_effective + self.prior_variance) * se**2)
        )

        S_n = math.exp(min(log_S_n, 700))
        always_valid_p = min(1.0 / max(S_n, 1.0), 1.0)

        info_fraction = min(n_effective / self.max_observations, 1.0)
        is_significant = always_valid_p < self.alpha
        can_stop = is_significant and info_fraction >= 0.3

        if can_stop:
            recommendation = "REJECT_H0_EARLY"
        elif info_fraction >= 1.0 and is_significant:
            recommendation = "REJECT_H0"
        elif info_fraction >= 1.0 and not is_significant:
            recommendation = "FAIL_TO_REJECT"
        else:
            recommendation = "CONTINUE"

        return AlwaysValidTestResult(
            p_value=always_valid_p,
            is_significant=is_significant,
            info_fraction=info_fraction,
            cumulative_s=S_n,
            recommendation=recommendation,
            can_stop=can_stop,
        )


class AlwaysValidExperimentMonitor:
    """
    始终有效实验监控器——支持随时查看实验结果

    与传统 A/B 测试不同，始终有效框架允许在任何时间点查看结果，
    而不会导致假阳性率膨胀。这使得实验监控看板可以实时展示 p 值，
    产品经理无需担心"偷看问题"。
    """

    def __init__(self, alpha: float = 0.05, db=None):
        self.alpha = alpha
        self.db = db
        self.tester = AlwaysValidPValueTester(alpha=alpha)

    def check_experiment(self, experiment_id: str) -> dict:
        """检查实验状态（可随时调用，无需担心偷看问题）"""
        # 获取实验数据
        data = self._get_experiment_data(experiment_id)

        if not data:
            return {"status": "NO_DATA"}

        # 使用始终有效 p 值检验
        result = self.tester.compute_p_value_ratio_metric(
            control_conversions=data["control_conversions"],
            control_total=data["control_total"],
            treatment_conversions=data["treatment_conversions"],
            treatment_total=data["treatment_total"],
        )

        # 构建看板数据（可安全展示，因为 p 值始终有效）
        return {
            "experiment_id": experiment_id,
            "always_valid_p": result.p_value,
            "is_significant": result.is_significant,
            "can_stop": result.can_stop,
            "recommendation": result.recommendation,
            "info_fraction": result.info_fraction,
            "control_rate": data["control_conversions"] / max(data["control_total"], 1),
            "treatment_rate": data["treatment_conversions"] / max(data["treatment_total"], 1),
            "relative_lift": (
                (data["treatment_conversions"] / max(data["treatment_total"], 1)) /
                (data["control_conversions"] / max(data["control_total"], 1)) - 1
            ) if data["control_conversions"] > 0 else 0,
            "sample_size": data["control_total"] + data["treatment_total"],
            "note": "始终有效 p 值：可随时查看，不存在偷看问题" if not result.is_significant
                    else "始终有效 p 值显示显著，可以安全决策",
        }

    def _get_experiment_data(self, experiment_id: str) -> Optional[dict]:
        """从数据库获取实验数据"""
        if not self.db:
            return None
        result = self.db.query_one("""
            SELECT
                SUM(CASE WHEN variant='control' AND converted=1 THEN 1 ELSE 0 END) as control_conversions,
                SUM(CASE WHEN variant='control' THEN 1 ELSE 0 END) as control_total,
                SUM(CASE WHEN variant='treatment' AND converted=1 THEN 1 ELSE 0 END) as treatment_conversions,
                SUM(CASE WHEN variant='treatment' THEN 1 ELSE 0 END) as treatment_total
            FROM experiment_metrics
            WHERE experiment_id = %s
        """, experiment_id)
        return result
```

**始终有效 p 值 vs 传统 p 值 vs OBF 边界对比：**

| 维度 | 传统 p 值 | OBF 序贯检验 | 始终有效 p 值 |
|------|---------|------------|-------------|
| 偷看安全性 | 不安全（假阳性率飙升） | 安全（边界控制） | 安全（数学证明保证） |
| 查看次数限制 | 只能看1次 | 预设固定次数 | 无限制，随时可看 |
| 实现复杂度 | 低 | 中 | 中 |
| 统计功效 | 最高（仅看1次时） | 中 | 略低于 OBF |
| 适用场景 | 严格预设终点 | 预设分析时间点 | 实时监控看板 |
| p 值解释 | 仅在终点有效 | 在分析时间点有效 | 在任意时刻有效 |

### 完整指标流水线（事件 → 聚合 → 检验 → 看板）

```python
import json
import time
from dataclasses import dataclass
from typing import Dict, List, Any, Optional
from enum import Enum

class MetricType(Enum):
    RATIO = "ratio"           # 比率型：转化率、点击率
    CONTINUOUS = "continuous"  # 连续型：停留时长、客单价
    COUNT = "count"           # 计数型：页面浏览次数

@dataclass
class ExperimentEvent:
    """实验事件：用户行为触发"""
    user_id: str
    experiment_id: str
    variant: str               # control / treatment
    event_type: str            # expose / click / convert / pageview / purchase
    metric_name: str           # 对应的指标名
    value: float               # 事件值（1.0 = 转化，实际金额 = 客单价等）
    timestamp: int             # 毫秒时间戳

@dataclass
class AggregatedMetric:
    """聚合后的指标"""
    experiment_id: str
    variant: str
    metric_name: str
    metric_type: MetricType
    n: int                     # 样本量
    sum_value: float           # 值总和
    mean: float                # 均值
    variance: float            # 方差
    window_start: int          # 聚合窗口开始时间
    window_end: int            # 聚合窗口结束时间

@dataclass
class TestResult:
    """统计检验结果"""
    experiment_id: str
    metric_name: str
    control_mean: float
    treatment_mean: float
    relative_change: float     # 相对变化率
    p_value: float             # 原始 p 值
    adjusted_p_value: float    # 校正后 p 值
    ci_lower: float            # 置信区间下界
    ci_upper: float            # 置信区间上界
    is_significant: bool       # 校正后是否显著
    test_method: str           # 使用的检验方法
    always_valid_p: float      # 始终有效 p 值


class EventIngestionService:
    """第一层：事件接入——接收用户行为事件并写入消息队列"""

    def __init__(self, kafka_producer, assignment_service):
        self.kafka = kafka_producer
        self.assignment_service = assignment_service

    def on_user_event(self, user_id: str, event_type: str,
                      metric_name: str, value: float = 1.0):
        """
        用户事件回调——当用户产生行为时调用

        流程：
        1. 查询用户所在实验和分组
        2. 构建实验事件
        3. 发送到 Kafka
        """
        # 查询用户参与的实验
        assignments = self.assignment_service.get_user_assignments(user_id)

        for assignment in assignments:
            event = ExperimentEvent(
                user_id=user_id,
                experiment_id=assignment.experiment_id,
                variant=assignment.variant,
                event_type=event_type,
                metric_name=metric_name,
                value=value,
                timestamp=int(time.time() * 1000),
            )

            # 发送到 Kafka（按 experiment_id 分区，保证同一实验的事件有序）
            self.kafka.produce(
                topic="experiment_events",
                key=event.experiment_id,
                value=json.dumps({
                    "user_id": event.user_id,
                    "experiment_id": event.experiment_id,
                    "variant": event.variant,
                    "event_type": event.event_type,
                    "metric_name": event.metric_name,
                    "value": event.value,
                    "timestamp": event.timestamp,
                })
            )


class MetricAggregationService:
    """第二层：指标聚合——从 Kafka 消费事件，按时窗聚合"""

    def __init__(self, clickhouse_client, redis_client):
        self.clickhouse = clickhouse_client
        self.redis = redis_client

    def aggregate_window(self, experiment_id: str, metric_name: str,
                         metric_type: MetricType,
                         window_minutes: int = 60) -> Dict[str, AggregatedMetric]:
        """
        执行一次聚合窗口计算

        聚合逻辑：
        - RATIO 型：计算每个变体的转化率（sum/count）
        - CONTINUOUS 型：计算均值和方差
        - COUNT 型：计算每个用户的平均次数
        """
        query = """
            SELECT
                variant,
                COUNT(*) as n,
                SUM(value) as sum_value,
                AVG(value) as mean,
                varPop(value) as variance
            FROM experiment_events
            WHERE experiment_id = %(exp_id)s
              AND metric_name = %(metric)s
              AND timestamp >= %(window_start)s
            GROUP BY variant
        """

        window_start = int((time.time() - window_minutes * 60) * 1000)
        results = self.clickhouse.query(query, {
            "exp_id": experiment_id,
            "metric": metric_name,
            "window_start": window_start,
        })

        aggregated = {}
        for row in results:
            variant = row["variant"]
            aggregated[variant] = AggregatedMetric(
                experiment_id=experiment_id,
                variant=variant,
                metric_name=metric_name,
                metric_type=metric_type,
                n=row["n"],
                sum_value=row["sum_value"],
                mean=row["mean"],
                variance=row["variance"],
                window_start=window_start,
                window_end=int(time.time() * 1000),
            )

        # 写入 Redis 缓存（看板查询用）
        cache_key = f"aggregated:{experiment_id}:{metric_name}"
        self.redis.setex(
            cache_key, 300,  # 缓存 5 分钟
            json.dumps({
                v: {"n": a.n, "mean": a.mean, "variance": a.variance}
                for v, a in aggregated.items()
            })
        )

        return aggregated


class StatisticalTestService:
    """第三层：统计检验——对聚合后的指标执行统计检验"""

    def __init__(self, db, correction_engine=None):
        self.db = db
        self.correction_engine = correction_engine
        self.always_valid_tester = AlwaysValidPValueTester()

    def run_tests(self, experiment_id: str, metrics_config: list) -> List[TestResult]:
        """
        对实验的所有指标执行统计检验

        参数：
          metrics_config: [{"name": "click_rate", "type": "RATIO", "is_primary": True}, ...]
        """
        results = []

        for metric_config in metrics_config:
            metric_name = metric_config["name"]
            metric_type = MetricType(metric_config["type"])

            # 获取聚合数据
            agg = self._get_aggregated(experiment_id, metric_name)
            if not agg or "control" not in agg or "treatment" not in agg:
                continue

            control = agg["control"]
            treatment = agg["treatment"]

            # 选择检验方法
            if metric_type == MetricType.RATIO:
                test_result = self._z_test_proportion(control, treatment)
                test_method = "z_test_proportion"
            elif metric_type == MetricType.CONTINUOUS:
                test_result = self._welch_t_test(control, treatment)
                test_method = "welch_t_test"
            else:  # COUNT
                test_result = self._poisson_test(control, treatment)
                test_method = "poisson_test"

            # 始终有效 p 值
            av_result = self.always_valid_tester.compute_p_value_ratio_metric(
                control_conversions=int(control.sum_value),
                control_total=control.n,
                treatment_conversions=int(treatment.sum_value),
                treatment_total=treatment.n,
            )

            # 相对变化
            relative_change = (treatment.mean - control.mean) / max(abs(control.mean), 1e-10)

            # 置信区间
            ci_lower, ci_upper = self._compute_ci(
                control, treatment, metric_type
            )

            result = TestResult(
                experiment_id=experiment_id,
                metric_name=metric_name,
                control_mean=control.mean,
                treatment_mean=treatment.mean,
                relative_change=relative_change,
                p_value=test_result["p_value"],
                adjusted_p_value=test_result["p_value"],  # 后续由校正引擎更新
                ci_lower=ci_lower,
                ci_upper=ci_upper,
                is_significant=False,  # 校正后更新
                test_method=test_method,
                always_valid_p=av_result.p_value,
            )
            results.append(result)

        # 多重比较校正
        if self.correction_engine and len(results) > 1:
            p_values = [r.p_value for r in results]
            is_primary = [any(
                m["name"] == r.metric_name and m.get("is_primary", False)
                for m in metrics_config
            ) for r in results]
            correction_result = self.correction_engine.hybrid_correction(
                p_values, is_primary
            )
            for i, result in enumerate(results):
                result.adjusted_p_value = correction_result.adjusted_p[i]
                result.is_significant = correction_result.is_significant[i]

        # 存储结果
        self._store_results(results)

        return results

    def _z_test_proportion(self, control: AggregatedMetric,
                           treatment: AggregatedMetric) -> dict:
        """双样本 Z 检验（比率型）"""
        p1 = treatment.sum_value / max(treatment.n, 1)
        p2 = control.sum_value / max(control.n, 1)
        p_pool = (treatment.sum_value + control.sum_value) / max(treatment.n + control.n, 1)

        se = math.sqrt(p_pool * (1 - p_pool) * (1/max(treatment.n, 1) + 1/max(control.n, 1)))
        if se < 1e-10:
            return {"p_value": 1.0, "z_score": 0}

        z = (p1 - p2) / se
        p_value = 2 * (1 - self._norm_cdf(abs(z)))
        return {"p_value": p_value, "z_score": z}

    def _welch_t_test(self, control: AggregatedMetric,
                      treatment: AggregatedMetric) -> dict:
        """Welch t 检验（连续型，不假设等方差）"""
        mean_diff = treatment.mean - control.mean
        se = math.sqrt(
            control.variance / max(control.n, 1) +
            treatment.variance / max(treatment.n, 1)
        )
        if se < 1e-10:
            return {"p_value": 1.0, "t_score": 0}

        t = mean_diff / se

        # Welch-Satterthwaite 自由度
        df_num = (control.variance / control.n + treatment.variance / treatment.n) ** 2
        df_denom = (
            (control.variance / control.n) ** 2 / (control.n - 1) +
            (treatment.variance / treatment.n) ** 2 / (treatment.n - 1)
        )
        df = df_num / max(df_denom, 1e-10)

        p_value = 2 * (1 - self._t_cdf(abs(t), df))
        return {"p_value": p_value, "t_score": t}

    def _poisson_test(self, control: AggregatedMetric,
                      treatment: AggregatedMetric) -> dict:
        """泊松检验（计数型）——使用条件检验"""
        # E[Poisson(λ1)] vs E[Poisson(λ2)]
        # 条件：在 N1+N2=K 的条件下，N1 ~ Binomial(K, λ1/(λ1+λ2))
        k = int(control.sum_value + treatment.sum_value)
        if k == 0:
            return {"p_value": 1.0}

        from math import comb
        n1 = int(control.sum_value)
        p = treatment.n / max(control.n + treatment.n, 1)  # 暴露比例

        # 二项检验
        p_value = 2 * min(
            sum(comb(k, i) * p**i * (1-p)**(k-i) for i in range(n1+1)),
            sum(comb(k, i) * p**i * (1-p)**(k-i) for i in range(n1, k+1)),
            0.5
        )
        return {"p_value": min(p_value, 1.0)}

    def _compute_ci(self, control: AggregatedMetric,
                    treatment: AggregatedMetric,
                    metric_type: MetricType) -> Tuple[float, float]:
        """计算 95% 置信区间"""
        mean_diff = treatment.mean - control.mean
        se = math.sqrt(
            control.variance / max(control.n, 1) +
            treatment.variance / max(treatment.n, 1)
        )
        z = 1.96  # 95% CI
        return (mean_diff - z * se, mean_diff + z * se)

    @staticmethod
    def _norm_cdf(x: float) -> float:
        """标准正态分布 CDF（近似）"""
        return 0.5 * (1 + math.erf(x / math.sqrt(2)))

    @staticmethod
    def _t_cdf(x: float, df: float) -> float:
        """t 分布 CDF（使用近似）"""
        # 简化：当 df > 30 时用正态近似
        if df > 30:
            return 0.5 * (1 + math.erf(x / math.sqrt(2)))
        # 小样本时使用更精确的近似（省略）
        return 0.5 * (1 + math.erf(x / math.sqrt(2)))

    def _get_aggregated(self, experiment_id: str, metric_name: str) -> Optional[Dict]:
        """获取聚合数据"""
        # 从 ClickHouse 查询
        pass

    def _store_results(self, results: List[TestResult]):
        """存储检验结果"""
        for r in results:
            self.db.execute("""
                INSERT INTO experiment_results
                (experiment_id, metric_name, control_mean, treatment_mean,
                 relative_change, p_value, adjusted_p_value, ci_lower, ci_upper,
                 is_significant, test_method, analysis_time)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, NOW())
            """, r.experiment_id, r.metric_name, r.control_mean, r.treatment_mean,
                 r.relative_change, r.p_value, r.adjusted_p_value,
                 r.ci_lower, r.ci_upper, r.is_significant, r.test_method)


class ExperimentDashboardService:
    """第四层：实验看板——将检验结果展示给用户"""

    def __init__(self, db, redis):
        self.db = db
        self.redis = redis

    def get_dashboard_data(self, experiment_id: str) -> dict:
        """获取实验看板数据"""
        # 实验基本信息
        experiment = self.db.query_one(
            "SELECT * FROM experiments WHERE id = %s", experiment_id
        )

        # 检验结果
        results = self.db.query("""
            SELECT * FROM experiment_results
            WHERE experiment_id = %s
            ORDER BY analysis_time DESC
        """, experiment_id)

        # 每个指标取最新结果
        latest_results = {}
        for r in results:
            if r["metric_name"] not in latest_results:
                latest_results[r["metric_name"]] = r

        # 构建看板数据
        metrics_dashboard = []
        for metric_name, result in latest_results.items():
            metrics_dashboard.append({
                "metric_name": metric_name,
                "control_mean": float(result["control_mean"]),
                "treatment_mean": float(result["treatment_mean"]),
                "relative_change": f"{float(result['relative_change']):.2%}",
                "p_value": float(result["adjusted_p_value"]),
                "always_valid_p": float(result.get("always_valid_p", 1.0)),
                "is_significant": bool(result["is_significant"]),
                "ci": f"[{float(result['ci_lower']):.4f}, {float(result['ci_upper']):.4f}]",
                "test_method": result["test_method"],
                "sample_size": int(result.get("control_n", 0) + result.get("treatment_n", 0)),
            })

        return {
            "experiment_id": experiment_id,
            "experiment_name": experiment["name"],
            "status": experiment["status"],
            "started_at": experiment["started_at"].isoformat() if experiment.get("started_at") else None,
            "duration_days": (datetime.now() - experiment["started_at"]).days if experiment.get("started_at") else 0,
            "metrics": metrics_dashboard,
            "srm_detected": bool(results[0].get("srm_detected", False)) if results else False,
            "paradox_detected": bool(results[0].get("paradox_detected", False)) if results else False,
        }
```

**指标流水线端到端延迟：**

| 阶段 | 数据流 | 延迟 | 频率 |
|------|--------|------|------|
| 事件接入 | 用户行为 → Kafka | < 100ms | 实时 |
| 指标聚合 | Kafka → Flink → ClickHouse | 2-5min | 每小时窗口 |
| 统计检验 | ClickHouse → Python | 1-3s | 每小时触发 |
| 看板展示 | Python → Redis → 前端 | < 500ms | 用户查询 |
| **端到端** | | **3-8min** | |

### 场景 5：实验分组被恶意刷量攻击

```python
# 场景描述：竞争对手注册大量机器人账号，全部被分到实验组
# → 实验组样本量异常膨胀但转化率为 0 → 拉低实验组整体指标
# → 误判"新方案显著更差"→ 错误回滚

class BotDetectionForExperiments:
    """实验中的机器人检测——防止恶意刷量影响实验结果"""

    def __init__(self, db, redis, risk_service):
        self.db = db
        self.redis = redis
        self.risk_service = risk_service

    def check_experiment_health(self, experiment_id: str) -> dict:
        """检查实验是否存在恶意刷量"""
        # 指标1：两组用户数比例异常
        counts = self.db.query("""
            SELECT variant, COUNT(*) as n,
                   COUNT(CASE WHEN is_bot = 1 THEN 1 END) as bot_count
            FROM user_assignments ua
            LEFT JOIN user_risk ur ON ua.user_id = ur.user_id
            WHERE ua.experiment_id = %s
            GROUP BY variant
        """, experiment_id)

        control = next((r for r in counts if r["variant"] == "control"), None)
        treatment = next((r for r in counts if r["variant"] == "treatment"), None)

        if not control or not treatment:
            return {"healthy": True}

        # 检测1：实验组机器人比例异常
        control_bot_rate = control["bot_count"] / max(control["n"], 1)
        treatment_bot_rate = treatment["bot_count"] / max(treatment["n"], 1)

        bot_ratio = treatment_bot_rate / max(control_bot_rate, 0.001)

        # 检测2：实验组新注册用户比例异常
        control_new_rate = self._new_user_rate(experiment_id, "control")
        treatment_new_rate = self._new_user_rate(experiment_id, "treatment")

        new_ratio = treatment_new_rate / max(control_new_rate, 0.001)

        is_healthy = bot_ratio < 3.0 and new_ratio < 3.0

        if not is_healthy:
            self.alert.send_critical(
                title=f"实验 {experiment_id} 疑似遭受刷量攻击",
                message=f"实验组机器人比例是对照组的 {bot_ratio:.1f} 倍，"
                        f"新用户比例是对照组的 {new_ratio:.1f} 倍"
            )

        return {
            "healthy": is_healthy,
            "control_bot_rate": control_bot_rate,
            "treatment_bot_rate": treatment_bot_rate,
            "bot_ratio": bot_ratio,
            "new_ratio": new_ratio,
        }

    def clean_experiment_data(self, experiment_id: str) -> int:
        """清洗实验中的机器人数据"""
        # 识别机器人用户
        bot_users = self.db.query("""
            SELECT user_id FROM user_risk
            WHERE is_bot = 1 AND user_id IN (
                SELECT user_id FROM user_assignments
                WHERE experiment_id = %s
            )
        """, experiment_id)

        # 从实验数据中排除机器人
        cleaned = 0
        for user in bot_users:
            self.db.execute("""
                UPDATE experiment_metrics
                SET excluded = 1
                WHERE experiment_id = %s AND user_id = %s
            """, experiment_id, user["user_id"])
            cleaned += 1

        return cleaned

    def _new_user_rate(self, experiment_id: str, variant: str) -> float:
        """计算某组中新注册用户占比"""
        result = self.db.query_one("""
            SELECT
                COUNT(CASE WHEN u.created_at > DATE_SUB(NOW(), INTERVAL 7 DAY) THEN 1 END) * 1.0 /
                COUNT(*) as new_rate
            FROM user_assignments ua
            JOIN users u ON ua.user_id = u.id
            WHERE ua.experiment_id = %s AND ua.variant = %s
        """, experiment_id, variant)
        return result["new_rate"] if result else 0
```

### 场景 6：实验指标定义不一致导致结论矛盾

```python
# 场景描述：同一个实验，团队 A 说"转化率提升 5%"，团队 B 说"转化率下降 2%"
# 根因：团队 A 的转化率 = 点击人数 / 曝光人数，团队 B 的转化率 = 点击人数 / 活跃人数
# 指标口径不一致导致完全相反的结论

class MetricDefinitionService:
    """指标定义服务——统一指标口径，防止定义不一致"""

    # 指标定义模板——所有实验必须使用统一定义
    METRIC_DEFINITIONS = {
        "click_rate": {
            "name": "点击率",
            "numerator": "点击人数（去重）",
            "denominator": "曝光人数（看到按钮的用户）",
            "sql_template": """
                COUNT(DISTINCT CASE WHEN event_type='click' THEN user_id END) * 1.0 /
                COUNT(DISTINCT CASE WHEN event_type='expose' THEN user_id END)
            """,
            "misuse_examples": [
                "❌ 点击次数 / 曝光次数（未去重，一个用户多次点击算多次）",
                "❌ 点击人数 / 活跃人数（分母不一致）",
                "❌ 点击人数 / 总注册用户（分母过大，包含未曝光用户）",
            ]
        },
        "conversion_rate": {
            "name": "转化率",
            "numerator": "下单人数（去重）",
            "denominator": "进入商品详情页的人数",
            "sql_template": """
                COUNT(DISTINCT CASE WHEN event_type='purchase' THEN user_id END) * 1.0 /
                COUNT(DISTINCT CASE WHEN event_type='view_product' THEN user_id END)
            """,
            "misuse_examples": [
                "❌ 下单次数 / 进入详情页次数",
                "❌ 下单人数 / 活跃用户数",
                "❌ 下单金额 / 进入详情页人数",
            ]
        },
        "retention_rate": {
            "name": "留存率",
            "numerator": "次日回访人数",
            "denominator": "当日新增用户数",
            "sql_template": """
                COUNT(DISTINCT CASE WHEN event_type='visit' AND date=next_day THEN user_id END) * 1.0 /
                COUNT(DISTINCT CASE WHEN event_type='first_visit' THEN user_id END)
            """,
        },
    }

    def validate_metric_definition(self, experiment_id: str, metric_name: str,
                                    custom_sql: str = None) -> dict:
        """验证指标定义是否与标准一致"""
        standard = self.METRIC_DEFINITIONS.get(metric_name)

        if not standard:
            return {
                "valid": False,
                "reason": f"指标 '{metric_name}' 没有标准定义，请联系数据团队添加"
            }

        if custom_sql and custom_sql.strip() != standard["sql_template"].strip():
            return {
                "valid": False,
                "reason": f"自定义 SQL 与标准定义不一致",
                "standard_sql": standard["sql_template"],
                "provided_sql": custom_sql,
            }

        return {"valid": True, "definition": standard}
```

### 场景 7：实验平台自身 Bug 导致分组错误

```python
# 场景描述：实验平台升级后，分组服务出现 Bug
# 10% 的用户每次请求被分配到不同的组 → 分组不稳定
# 后果：实验组和对照组互相"污染"→ 效果被稀释 → 真实效果被低估

class AssignmentConsistencyChecker:
    """分组一致性检测器——定期检查用户分组是否稳定"""

    def __init__(self, db, redis, alert_service):
        self.db = db
        self.redis = redis
        self.alert = alert_service

    def check_consistency(self, experiment_id: str, sample_size: int = 1000) -> dict:
        """
        检查分组一致性

        方法：随机抽取 N 个用户，重新计算其分组，与数据库记录对比
        不一致率 > 0.1% 说明存在问题
        """
        # 随机抽样
        users = self.db.query("""
            SELECT user_id, group_name FROM user_assignments
            WHERE experiment_id = %s
            ORDER BY RAND() LIMIT %s
        """, experiment_id, sample_size)

        inconsistent_count = 0
        inconsistent_users = []

        for record in users:
            user_id = record["user_id"]
            stored_group = record["group_name"]

            # 重新计算分组
            expected_group = self._compute_assignment(user_id, experiment_id)

            if stored_group != expected_group:
                inconsistent_count += 1
                inconsistent_users.append({
                    "user_id": user_id,
                    "stored_group": stored_group,
                    "expected_group": expected_group,
                })

        inconsistency_rate = inconsistent_count / max(len(users), 1)

        result = {
            "experiment_id": experiment_id,
            "sample_size": len(users),
            "inconsistent_count": inconsistent_count,
            "inconsistency_rate": inconsistency_rate,
            "is_healthy": inconsistency_rate < 0.001,  # 0.1% 以下正常
            "details": inconsistent_users[:20],  # 最多展示 20 条
        }

        if not result["is_healthy"]:
            self.alert.send_critical(
                title=f"实验 {experiment_id} 分组不一致率 {inconsistency_rate:.2%}",
                message=f"抽样 {sample_size} 个用户，{inconsistent_count} 个分组不一致。"
                        f"可能原因：分组算法变更、数据库写入错误、缓存失效"
            )

        return result

    @staticmethod
    def _compute_assignment(user_id: str, experiment_id: str) -> str:
        """使用 MurmurHash3 重新计算分组"""
        import mmh3
        hash_val = mmh3.hash(f"{user_id}:{experiment_id}") % 10000
        return "control" if hash_val < 5000 else "treatment"

    def auto_repair(self, experiment_id: str, dry_run: bool = True) -> int:
        """自动修复不一致的分组记录"""
        # 查找所有不一致记录
        all_users = self.db.query("""
            SELECT user_id, group_name FROM user_assignments
            WHERE experiment_id = %s
        """, experiment_id)

        repair_count = 0
        for record in all_users:
            expected = self._compute_assignment(record["user_id"], experiment_id)
            if record["group_name"] != expected:
                if not dry_run:
                    self.db.execute("""
                        UPDATE user_assignments
                        SET group_name = %s
                        WHERE user_id = %s AND experiment_id = %s
                    """, expected, record["user_id"], experiment_id)
                repair_count += 1

        if repair_count > 0 and not dry_run:
            self.alert.send_info(
                title=f"实验 {experiment_id} 分组修复完成",
                message=f"修复了 {repair_count} 条不一致记录"
            )

        return repair_count
```

## A/B 测试统计引擎完整实现

```python
class ABTestStatEngine:
    """A/B 测试统计引擎：序贯检验 + 贝叶斯 + 多重比较"""

    def sequential_test(self, experiment_id, metric_name):
        """序贯检验（节省样本量 30-50%）"""
        control = self._get_metric_data(experiment_id, "control", metric_name)
        treatment = self._get_metric_data(experiment_id, "treatment", metric_name)

        n_c, mean_c, var_c = len(control), statistics.mean(control), statistics.variance(control) if len(control) > 1 else 0
        n_t, mean_t, var_t = len(treatment), statistics.mean(treatment), statistics.variance(treatment) if len(treatment) > 1 else 0

        if n_c < 100 or n_t < 100:
            return {"status": "insufficient_data", "n_control": n_c, "n_treatment": n_t}

        # Z 检验
        pooled_se = math.sqrt(var_c / n_c + var_t / n_t)
        if pooled_se == 0:
            return {"status": "zero_variance"}

        z_score = (mean_t - mean_c) / pooled_se
        p_value = 2 * (1 - self._norm_cdf(abs(z_score)))

        # 效应量
        pooled_std = math.sqrt((var_c * (n_c - 1) + var_t * (n_t - 1)) / (n_c + n_t - 2))
        cohens_d = (mean_t - mean_c) / pooled_std if pooled_std > 0 else 0

        # 置信区间
        ci_half = 1.96 * pooled_se
        ci = (round(mean_t - mean_c - ci_half, 4), round(mean_t - mean_c + ci_half, 4))

        # 统计功效
        power = self._calculate_power(n_c, n_t, cohens_d, alpha=0.05)

        decision = "significant" if p_value < 0.05 else "not_significant"
        if decision == "significant" and abs(cohens_d) < 0.1:
            decision = "significant_but_trivial"  # 统计显著但实际不显著

        return {
            "experiment_id": experiment_id, "metric": metric_name,
            "control_mean": round(mean_c, 6), "treatment_mean": round(mean_t, 6),
            "lift": round((mean_t - mean_c) / abs(mean_c) * 100, 2) if mean_c != 0 else None,
            "z_score": round(z_score, 4), "p_value": round(p_value, 6),
            "ci_95": ci, "cohens_d": round(cohens_d, 4),
            "power": round(power, 3), "decision": decision
        }

    def bayesian_test(self, experiment_id, metric_name):
        """贝叶斯 A/B 测试（给出 P(B>A) 概率）"""
        control = self._get_metric_data(experiment_id, "control", metric_name)
        treatment = self._get_metric_data(experiment_id, "treatment", metric_name)

        # 共轭先验（正态-逆伽马）
        prior_mu, prior_sigma = 0, 10  # 弱先验
        n_c, mean_c = len(control), statistics.mean(control)
        n_t, mean_t = len(treatment), statistics.mean(treatment)
        var_c = statistics.variance(control) if len(control) > 1 else 1
        var_t = statistics.variance(treatment) if len(treatment) > 1 else 1

        # Monte Carlo 采样
        import numpy as np
        n_samples = 100000
        control_samples = np.random.normal(mean_c, math.sqrt(var_c / n_c), n_samples)
        treatment_samples = np.random.normal(mean_t, math.sqrt(var_t / n_t), n_samples)

        prob_treatment_better = float(np.mean(treatment_samples > control_samples))
        expected_lift = float(np.mean((treatment_samples - control_samples) / np.abs(control_samples) * 100))

        return {
            "prob_treatment_better": round(prob_treatment_better, 4),
            "expected_lift_pct": round(expected_lift, 2),
            "decision": "winner" if prob_treatment_better > 0.95 else
                       "loser" if prob_treatment_better < 0.05 else "inconclusive"
        }

    def multiple_comparison_correction(self, results, method="bonferroni"):
        """多重比较校正"""
        n_tests = len(results)
        if method == "bonferroni":
            for r in results:
                r["adjusted_p_value"] = min(1.0, r["p_value"] * n_tests)
                r["significant_after_correction"] = r["adjusted_p_value"] < 0.05
        elif method == "benjamini_hochberg":
            sorted_results = sorted(results, key=lambda x: x["p_value"])
            for i, r in enumerate(sorted_results):
                r["adjusted_p_value"] = min(1.0, r["p_value"] * n_tests / (i + 1))
                r["significant_after_correction"] = r["adjusted_p_value"] < 0.05
        return results

    def _norm_cdf(self, x):
        """标准正态 CDF"""
        return 0.5 * (1 + math.erf(x / math.sqrt(2)))

    def _calculate_power(self, n1, n2, d, alpha=0.05):
        """计算统计功效"""
        se = math.sqrt(1/n1 + 1/n2)
        ncp = d / se  # 非中心参数
        z_alpha = 1.96  # 双侧 alpha=0.05
        power = 1 - self._norm_cdf(z_alpha - ncp)
        return max(0, min(1, power))
```

## 实验护栏与互斥组

```python
class ExperimentGuardrailService:
    """实验护栏：互斥组 + 指标护栏 + 自动停止"""

    def check_guardrails(self, experiment_id):
        """检查实验护栏"""
        exp = self.db.get_experiment(experiment_id)
        guardrails = json.loads(exp.get("guardrails", "[]"))

        violations = []
        for guard in guardrails:
            if guard["type"] == "metric_threshold":
                current = self._get_current_metric(experiment_id, guard["metric"])
                if guard["direction"] == "below" and current < guard["threshold"]:
                    violations.append({
                        "guardrail": guard["name"],
                        "current": current,
                        "threshold": guard["threshold"],
                        "action": "stop_experiment"
                    })
                elif guard["direction"] == "above" and current > guard["threshold"]:
                    violations.append({
                        "guardrail": guard["name"],
                        "current": current,
                        "threshold": guard["threshold"],
                        "action": "stop_experiment"
                    })

        # 自动停止违规实验
        if any(v["action"] == "stop_experiment" for v in violations):
            self._stop_experiment(experiment_id, violations)

        return {"violations": violations, "status": "stopped" if violations else "ok"}

    def check_mutual_exclusion(self, experiment_id, user_id):
        """检查互斥组（同一用户不能同时进入冲突实验）"""
        group = self.db.query_one(
            "SELECT group_id FROM experiment_groups "
            "WHERE experiment_id = %s", experiment_id)

        if not group:
            return {"allowed": True}

        # 检查用户是否已在同组其他实验中
        active_in_group = self.db.query(
            "SELECT e.id, e.name FROM experiment_assignments ea "
            "JOIN experiments e ON ea.experiment_id = e.id "
            "WHERE ea.user_id = %s AND e.group_id = %s "
            "AND e.status = 'running'", user_id, group["group_id"])

        if active_in_group:
            return {"allowed": False,
                    "conflict": active_in_group[0]["name"]}

        return {"allowed": True}
```

## 异常场景补充

### 场景：A/B 测试辛普森悖论

```
触发：总体看 B > A，但每个细分群体都是 A > B → 辛普森悖论
原因：实验组和对照组的用户构成不同（如 B 组新用户比例更高）
检测：
  1. 总体结论与细分结论矛盾 → 辛普森悖论
  2. 分层分析后效应方向反转 → 确认
处理：
  1. 使用分层分析（按用户属性分层计算效应）
  2. 确保随机分配后各层比例均衡
  3. 报告分层结果而非总体结果
预防：分层随机分配 + 分层分析 + SRM 检测
```

### 场景：实验护栏过于敏感自动停止

```
触发：营收指标护栏阈值设为 -2% → 正常波动触发 → 实验被误停
检测：
  1. 实验被自动停止但指标在正常范围 → 护栏过敏感
  2. 护栏触发频率 > 20% → 过于敏感
处理：
  1. 放宽护栏阈值（-2% → -5%）
  2. 改用连续 N 小时低于阈值才触发
  3. 重启实验
预防：护栏阈值基于历史波动设定 + 连续触发条件 + 人工确认
```

## 实验配置管理完整实现

```python
class ExperimentConfigService:
    """实验配置管理：参数化 + 版本化 + 审批"""

    def create_experiment_config(self, name, parameters, metrics, guardrails):
        """创建实验配置"""
        config_id = str(uuid4())

        # 验证参数
        for param in parameters:
            if param["type"] == "continuous":
                if param.get("min") is None or param.get("max") is None:
                    raise InvalidConfigError(f"连续参数 {param['name']} 需要设置 min/max")
            elif param["type"] == "discrete":
                if not param.get("values"):
                    raise InvalidConfigError(f"离散参数 {param['name']} 需要设置 values 列表")

        # 验证指标
        for metric in metrics:
            if metric["direction"] not in ["increase", "decrease"]:
                raise InvalidConfigError(f"指标方向必须为 increase 或 decrease")

        self.db.insert("experiment_configs", {
            "config_id": config_id,
            "name": name,
            "version": 1,
            "parameters": json.dumps(parameters),
            "metrics": json.dumps(metrics),
            "guardrails": json.dumps(guardrails),
            "status": "draft",
            "created_at": now()
        })

        return {"config_id": config_id, "version": 1}

    def submit_for_approval(self, config_id):
        """提交审批"""
        config = self.db.get_config(config_id)

        # 自动检查
        checks = self._auto_validate(config)
        if not checks["passed"]:
            return {"status": "validation_failed", "issues": checks["issues"]}

        self.db.update("experiment_configs",
            {"status": "pending_approval"}, {"config_id": config_id})

        # 通知审批人
        self.notification.send("experiment_reviewer",
            f"新实验配置待审批: {config['name']}")

        return {"status": "pending_approval"}

    def _auto_validate(self, config):
        """自动验证配置"""
        issues = []
        params = json.loads(config["parameters"])

        # 检查最小样本量
        for metric in json.loads(config["metrics"]):
            min_detectable_effect = metric.get("min_detectable_effect", 0.01)
            required_n = self._calculate_min_sample_size(
                metric["baseline_rate"], min_detectable_effect)
            metric["required_sample_size"] = required_n
            if required_n > 100000:
                issues.append(f"指标 {metric['name']} 需要 {required_n} 样本，可能超出流量")

        # 检查护栏是否覆盖关键指标
        guardrails = json.loads(config["guardrails"])
        metric_names = set(m["name"] for m in json.loads(config["metrics"]))
        guarded_names = set(g["metric"] for g in guardrails)
        unguarded = metric_names - guarded_names
        if unguarded:
            issues.append(f"指标 {unguarded} 未设置护栏")

        return {"passed": len(issues) == 0, "issues": issues}

    def _calculate_min_sample_size(self, baseline_rate, min_effect, alpha=0.05, power=0.8):
        """计算最小样本量"""
        p1 = baseline_rate
        p2 = baseline_rate + min_effect
        z_alpha = 1.96
        z_beta = 0.84

        n = ((z_alpha * math.sqrt(2 * p1 * (1 - p1)) +
              z_beta * math.sqrt(p1 * (1 - p1) + p2 * (1 - p2))) ** 2) / \
            (p2 - p1) ** 2

        return int(math.ceil(n))
```

## 实验效果归因分析

```python
class ExperimentAttributionService:
    """实验归因分析：哪个参数变化导致了效果提升"""

    def attribute_effect(self, experiment_id, winning_variant):
        """归因分析：分解各参数的贡献"""
        exp = self.db.get_experiment(experiment_id)
        parameters = json.loads(exp["parameters"])
        metrics = json.loads(exp["metrics"])

        attributions = []
        for metric in metrics:
            metric_name = metric["name"]

            # 对每个参数计算单独的贡献
            param_contributions = {}
            for param in parameters:
                # 使用交互效应矩阵计算各参数贡献
                effect = self._calculate_param_effect(
                    experiment_id, param["name"], metric_name)
                param_contributions[param["name"]] = effect

            attributions.append({
                "metric": metric_name,
                "total_lift": self._get_total_lift(experiment_id, metric_name),
                "param_contributions": param_contributions
            })

        return {"experiment_id": experiment_id,
                "attributions": attributions}

    def _calculate_param_effect(self, experiment_id, param_name, metric_name):
        """计算单个参数对指标的效果"""
        # 获取参数不同值下的指标表现
        variants = self.db.query(
            "SELECT variant, param_values, metric_values "
            "FROM experiment_variant_results "
            "WHERE experiment_id = %s", experiment_id)

        # 按参数值分组
        by_param_value = {}
        for v in variants:
            params = json.loads(v["param_values"])
            metrics = json.loads(v["metric_values"])
            param_val = params.get(param_name)
            if param_val not in by_param_value:
                by_param_value[param_val] = []
            by_param_value[param_val].append(metrics.get(metric_name, 0))

        # 计算每个参数值的平均效果
        effects = {}
        baseline = by_param_value.get(parameters[0]["default_value"], [])
        baseline_avg = statistics.mean(baseline) if baseline else 0

        for val, observations in by_param_value.items():
            avg = statistics.mean(observations) if observations else 0
            effects[val] = round((avg - baseline_avg) / abs(baseline_avg) * 100, 2)

        return effects
```

## 异常场景补充

### 场景：实验配置审批延迟影响上线

```
触发：实验配置提交审批 → 审批人出差 → 3 天未审批 → 上线延迟
检测：
  1. 审批状态 pending > 48 小时 → 延迟
  2. 上线时间被推迟 → 业务影响
处理：
  1. 设置审批超时自动升级（48h→备选审批人）
  2. 低风险实验自动审批
  3. 紧急实验快速审批通道
预防：审批超时升级 + 风险分级自动审批 + 快速通道
```

### 场景：归因分析结果互相矛盾

```
触发：参数 A 在独立测试中提升 5%，但在组合实验中下降 3% → 交互效应
检测：
  1. 单参数效果与组合效果方向相反 → 交互效应
  2. 归因分析总提升 ≠ 各参数贡献之和 → 交互效应
处理：
  1. 报告交互效应
  2. 不要简单归因，考虑参数间交互
  3. 设计参数交互实验验证
预防：交互效应分析 + 组合实验 + 不要独立归因
```

## 实验流量分配完整实现

```python
class ExperimentTrafficAllocator:
    """实验流量分配：互斥层 + 正交分配 + 流量复用"""

    def allocate_traffic(self, user_id, layer_name):
        """分配实验流量（分层正交）"""
        # 每层独立哈希，确保层间正交
        hash_input = f"{layer_name}:{user_id}"
        bucket = int(hashlib.md5(hash_input.encode()).hexdigest(), 16) % 10000

        # 查找该层中正在运行的实验
        experiments = self.db.query(
            "SELECT * FROM experiments "
            "WHERE layer = %s AND status = 'running'", layer_name)

        if not experiments:
            return {"layer": layer_name, "variant": "control", "experiment": None}

        # 按流量比例分配
        cumulative = 0
        for exp in experiments:
            traffic_pct = exp["traffic_pct"]
            cumulative += traffic_pct * 100  # 转为万分比
            if bucket < cumulative:
                # 该用户属于此实验
                variant_bucket = bucket % 100
                control_pct = (1 - exp["treatment_ratio"]) * 100
                variant = "treatment" if variant_bucket >= control_pct else "control"
                return {"layer": layer_name, "variant": variant,
                        "experiment_id": exp["id"], "experiment_name": exp["name"]}

        # 不属于任何实验 → 控制组
        return {"layer": layer_name, "variant": "control", "experiment": None}

    def check_layer_conflicts(self, layer_name):
        """检查层内流量冲突（总流量 > 100%）"""
        experiments = self.db.query(
            "SELECT * FROM experiments "
            "WHERE layer = %s AND status = 'running'", layer_name)

        total_traffic = sum(exp["traffic_pct"] for exp in experiments)

        if total_traffic > 1.0:
            return {"conflict": True, "total_traffic": total_traffic,
                    "message": f"层 {layer_name} 流量总和 {total_traffic*100}% 超过 100%",
                    "experiments": [{"name": e["name"], "traffic": e["traffic_pct"]} for e in experiments]}

        return {"conflict": False, "total_traffic": total_traffic}

    def migrate_traffic(self, from_experiment_id, to_experiment_id,
                       migration_pct=0.1):
        """渐进式流量迁移"""
        from_exp = self.db.get_experiment(from_experiment_id)
        to_exp = self.db.get_experiment(to_experiment_id)

        if from_exp["layer"] != to_exp["layer"]:
            raise TrafficMigrationError("跨层迁移不允许")

        # 每次迁移 10% 流量
        new_from_traffic = from_exp["traffic_pct"] - migration_pct
        new_to_traffic = to_exp["traffic_pct"] + migration_pct

        if new_from_traffic < 0:
            raise TrafficMigrationError("源实验流量不足")

        self.db.update("experiments",
            {"traffic_pct": new_from_traffic},
            {"id": from_experiment_id})
        self.db.update("experiments",
            {"traffic_pct": new_to_traffic},
            {"id": to_experiment_id})

        return {"from_traffic": round(new_from_traffic, 3),
                "to_traffic": round(new_to_traffic, 3)}
```

## 实验指标计算引擎

```python
class ExperimentMetricCalculator:
    """实验指标计算引擎：实时 + 窗口 + 归一化"""

    METRIC_TYPES = {
        "rate": "比率型（CTR、转化率）",
        "mean": "均值型（客单价、停留时长）",
        "quantile": "分位型（P50/P95/P99 延迟）",
        "count": "计数型（发帖数、订单数）",
    }

    def calculate_metric(self, experiment_id, metric_name, variant):
        """计算实验指标"""
        metric = self.db.query_one(
            "SELECT * FROM experiment_metrics "
            "WHERE experiment_id = %s AND name = %s",
            experiment_id, metric_name)

        metric_type = metric["type"]

        if metric_type == "rate":
            # 比率型：分子/分母
            numerator = self._count_events(experiment_id, variant,
                metric["numerator_event"])
            denominator = self._count_events(experiment_id, variant,
                metric["denominator_event"])
            value = numerator / max(denominator, 1)
            variance = value * (1 - value) / max(denominator, 1)

        elif metric_type == "mean":
            # 均值型
            values = self._get_event_values(experiment_id, variant,
                metric["event_name"], metric["value_field"])
            value = statistics.mean(values) if values else 0
            variance = statistics.variance(values) / max(len(values), 1) if len(values) > 1 else 0

        elif metric_type == "quantile":
            values = self._get_event_values(experiment_id, variant,
                metric["event_name"], metric["value_field"])
            quantile = metric.get("quantile", 0.95)
            value = self._percentile(values, quantile) if values else 0
            variance = 0  # 分位数的方差计算较复杂

        elif metric_type == "count":
            count = self._count_events(experiment_id, variant,
                metric["event_name"])
            n_users = self._count_users(experiment_id, variant)
            value = count / max(n_users, 1)
            variance = value / max(n_users, 1)

        return {"metric": metric_name, "variant": variant,
                "type": metric_type, "value": round(value, 6),
                "variance": round(variance, 8),
                "sample_size": self._count_users(experiment_id, variant)}

    def _percentile(self, values, q):
        """计算分位数"""
        if not values:
            return 0
        sorted_vals = sorted(values)
        idx = q * (len(sorted_vals) - 1)
        lower = int(idx)
        upper = min(lower + 1, len(sorted_vals) - 1)
        frac = idx - lower
        return sorted_vals[lower] * (1 - frac) + sorted_vals[upper] * frac
```

## 异常场景补充

### 场景：流量分配哈希偏差

```
触发：MD5 哈希在特定层实验中分配不均匀 → 控制组 55%、实验组 45% → 样本偏差
检测：
  1. SRM（样本比率偏差）检测 p < 0.01 → 分配不均
  2. 实际比例与预期差异 > 2% → 偏差
处理：
  1. 更换哈希函数（MurmurHash3 替代 MD5）
  2. 加入 SRM 检测自动告警
  3. 严重偏差 → 暂停实验
预防：SRM 检测 + 哈希函数验证 + 分配验证
```

### 场景：指标计算口径不一致

```
触发：不同团队对"转化率"定义不同 → 点击转化率 vs 购买转化率 → 结果不可比
检测：
  1. 同名指标不同计算方式 → 口径不一致
  2. 实验结果无法复现 → 口径问题
处理：
  1. 统一指标注册中心（定义 + 计算公式 + 分子分母）
  2. 实验必须使用注册指标
  3. 禁止自定义未注册指标
预防：指标注册中心 + 统一口径 + 指标审核
```

## 实验护栏系统完整实现

```python
class ExperimentGuardrailService:
    """实验护栏：关键指标监控 + 自动暂停 + 回滚"""

    GUARDRAIL_METRICS = {
        "revenue_per_user": {"direction": "increase", "max_decline_pct": 5, "severity": "critical"},
        "crash_rate": {"direction": "decrease", "max_increase_pct": 50, "severity": "critical"},
        "page_load_time_p95": {"direction": "decrease", "max_increase_pct": 20, "severity": "high"},
        "user_retention_d1": {"direction": "increase", "max_decline_pct": 2, "severity": "high"},
        "support_ticket_rate": {"direction": "decrease", "max_increase_pct": 30, "severity": "medium"},
    }

    def check_guardrails(self, experiment_id):
        """检查实验护栏"""
        exp = self.db.get_experiment(experiment_id)
        violations = []

        for metric_name, config in self.GUARDRAIL_METRICS.items():
            control = self._get_metric_value(experiment_id, "control", metric_name)
            treatment = self._get_metric_value(experiment_id, "treatment", metric_name)

            if control is None or treatment is None:
                continue

            if config["direction"] == "increase":
                decline_pct = (control - treatment) / max(abs(control), 0.001) * 100
                if decline_pct > config["max_decline_pct"]:
                    violations.append({
                        "metric": metric_name,
                        "control": round(control, 4),
                        "treatment": round(treatment, 4),
                        "change_pct": round(-decline_pct, 2),
                        "threshold": f"最大下降 {config['max_decline_pct']}%",
                        "severity": config["severity"]
                    })

            elif config["direction"] == "decrease":
                increase_pct = (treatment - control) / max(abs(control), 0.001) * 100
                if increase_pct > config["max_increase_pct"]:
                    violations.append({
                        "metric": metric_name,
                        "control": round(control, 4),
                        "treatment": round(treatment, 4),
                        "change_pct": round(increase_pct, 2),
                        "threshold": f"最大增加 {config['max_increase_pct']}%",
                        "severity": config["severity"]
                    })

        # 根据违规严重性决定动作
        if any(v["severity"] == "critical" for v in violations):
            action = "auto_pause"
            self._pause_experiment(experiment_id, "护栏违规（严重）")
        elif any(v["severity"] == "high" for v in violations):
            action = "alert_and_review"
            self.alert(f"实验 {experiment_id} 护栏告警: {violations}")
        elif violations:
            action = "log"
        else:
            action = "none"

        return {"experiment_id": experiment_id, "violations": violations,
                "action": action, "violation_count": len(violations)}

    def _pause_experiment(self, experiment_id, reason):
        """自动暂停实验"""
        self.db.update("experiments",
            {"status": "paused", "paused_reason": reason,
             "paused_at": now()},
            {"id": experiment_id})

        # 流量回退到控制组
        self.redis.set(f"exp_override:{experiment_id}", "control")

        # 通知实验负责人
        exp = self.db.get_experiment(experiment_id)
        self.notification.send(exp["owner"],
            f"实验 {exp['name']} 已自动暂停: {reason}")

    def resume_experiment(self, experiment_id, approved_by, reason):
        """恢复已暂停的实验"""
        exp = self.db.get_experiment(experiment_id)
        if exp["status"] != "paused":
            return {"status": "not_paused"}

        # 需要审批人确认
        self.db.update("experiments",
            {"status": "running", "resumed_by": approved_by,
             "resumed_reason": reason, "resumed_at": now()},
            {"id": experiment_id})

        # 恢复流量分配
        self.redis.delete(f"exp_override:{experiment_id}")

        return {"status": "resumed"}
```

## 异常场景补充

### 场景：护栏阈值过于敏感

```
触发：收入下降 3%（在正常波动范围内）→ 护栏触发 → 实验被自动暂停 → 浪费流量
检测：
  1. 实验被频繁自动暂停 → 护栏过敏感
  2. 暂停后人工审查认为无需暂停 → 误报
处理：
  1. 考虑统计显著性（不只是点估计差异）
  2. 调整阈值（5% → 8%）
  3. 加入波动容忍度
预防：统计显著性判断 + 阈值调整 + 波动容忍
```

### 场景：护栏检查间隔过长

```
触发：护栏每小时检查一次 → 实验造成 30 分钟严重损害 → 延迟发现
检测：
  1. 关键指标恶化到护栏检查之间 → 延迟发现
  2. 损害扩大 → 检查间隔过长
处理：
  1. 关键指标实时监控（非每小时）
  2. 严重指标（crash_rate）实时触发检查
  3. 降低检查间隔
预防：实时监控关键指标 + 严重指标即时触发
```

## 实验结果自动化决策完整实现

```python
class ExperimentAutoDecisionService:
    """实验自动决策：统计显著性 + 自动推送 + 决策记录"""

    DECISION_RULES = {
        "launch": {"min_lift_pct": 2, "min_confidence": 0.95, "min_sample_days": 7},
        "iterate": {"min_lift_pct": -1, "max_lift_pct": 2, "min_confidence": 0.8},
        "kill": {"max_lift_pct": -1, "min_confidence": 0.9, "min_sample_days": 3},
    }

    def evaluate_experiment(self, experiment_id):
        """评估实验并生成决策建议"""
        exp = self.db.get_experiment(experiment_id)
        days_running = (now() - exp["started_at"]).days

        # 1. 收集所有关键指标
        metrics = self.db.query(
            "SELECT * FROM experiment_metrics "
            "WHERE experiment_id = %s", experiment_id)

        decisions = {}

        for metric in metrics:
            control_data = self._get_metric_data(experiment_id, "control", metric["name"])
            treatment_data = self._get_metric_data(experiment_id, "treatment", metric["name"])

            # 2. 计算lift
            control_mean = statistics.mean(control_data) if control_data else 0
            treatment_mean = statistics.mean(treatment_data) if treatment_data else 0

            lift_pct = ((treatment_mean - control_mean) / max(abs(control_mean), 0.001)) * 100

            # 3. 计算统计显著性（t-test）
            if len(control_data) > 10 and len(treatment_data) > 10:
                t_stat, p_value = self._t_test(control_data, treatment_data)
                confidence = 1 - p_value
            else:
                confidence = 0

            # 4. 生成决策
            if days_running < self.DECISION_RULES["launch"]["min_sample_days"]:
                decision = "continue"  # 样本天数不足
            elif lift_pct >= self.DECISION_RULES["launch"]["min_lift_pct"] and \
                 confidence >= self.DECISION_RULES["launch"]["min_confidence"]:
                decision = "launch"
            elif lift_pct <= self.DECISION_RULES["kill"]["max_lift_pct"] and \
                 confidence >= self.DECISION_RULES["kill"]["min_confidence"] and \
                 days_running >= self.DECISION_RULES["kill"]["min_sample_days"]:
                decision = "kill"
            elif abs(lift_pct) < self.DECISION_RULES["iterate"]["min_lift_pct"] or \
                 confidence < self.DECISION_RULES["iterate"]["min_confidence"]:
                decision = "iterate"
            else:
                decision = "continue"

            decisions[metric["name"]] = {
                "control_mean": round(control_mean, 4),
                "treatment_mean": round(treatment_mean, 4),
                "lift_pct": round(lift_pct, 2),
                "confidence": round(confidence, 4),
                "p_value": round(p_value, 4) if p_value else None,
                "sample_size": len(control_data) + len(treatment_data),
                "decision": decision,
                "days_running": days_running
            }

        # 5. 综合决策（所有关键指标达成一致）
        key_metrics = [m for m in decisions if decisions[m].get("is_primary", False)]
        overall_decision = self._aggregate_decisions(decisions, key_metrics)

        # 6. 记录决策
        self.db.insert("experiment_decisions", {
            "decision_id": str(uuid4()),
            "experiment_id": experiment_id,
            "overall_decision": overall_decision,
            "metric_decisions": json.dumps(decisions),
            "days_running": days_running,
            "decided_at": now()
        })

        return {"experiment_id": experiment_id,
                "overall_decision": overall_decision,
                "metrics": decisions}

    def _t_test(self, group_a, group_b):
        """独立样本 t-test"""
        n_a, n_b = len(group_a), len(group_b)
        mean_a, mean_b = statistics.mean(group_a), statistics.mean(group_b)
        var_a = statistics.variance(group_a) if n_a > 1 else 0
        var_b = statistics.variance(group_b) if n_b > 1 else 0

        se = math.sqrt(var_a / n_a + var_b / n_b)
        if se == 0:
            return 0, 1.0

        t_stat = (mean_a - mean_b) / se

        # 近似 p-value（使用正态分布近似）
        df = n_a + n_b - 2
        p_value = 2 * (1 - self._t_distribution_cdf(abs(t_stat), df))

        return t_stat, p_value

    def _t_distribution_cdf(self, t, df):
        """t 分布 CDF 近似"""
        x = df / (df + t**2)
        return math.sqrt(x)

    def _aggregate_decisions(self, decisions, key_metrics):
        """综合决策"""
        if not key_metrics:
            key_metrics = list(decisions.keys())

        vote_counts = {"launch": 0, "kill": 0, "iterate": 0, "continue": 0}
        for m in key_metrics:
            d = decisions[m]["decision"]
            vote_counts[d] += 1

        # 优先级：kill > continue > iterate > launch
        if vote_counts["kill"] > 0:
            return "kill"
        elif vote_counts["continue"] > 0:
            return "continue"
        elif vote_counts["iterate"] > vote_counts["launch"]:
            return "iterate"
        else:
            return "launch"
```

## 异常场景补充

### 场景：自动决策过早

```
触发：实验运行 3 天 → 自动决策判定 "launch" → 但周末效应导致数据偏差 → 实际效果差
检测：
  1. 实验天数 < 7 天 → 样本可能不足
  2. 数据覆盖不完整周 → 周期偏差
处理：
  1. 最低运行天数限制（7 天）
  2. 必须覆盖完整周（含周末）
  3. 新用户 vs 老用户分别评估
预防：最低天数 + 完整周覆盖 + 用户分组
```

### 场景：多个指标决策冲突

```
触发：收入指标 lift +3% → launch / 留存指标 lift -1% → kill → 决策冲突
检测：
  1. 关键指标决策不一致 → 冲突
  2. 无法做出明确决策 → 需要权衡
处理：
  1. 定义指标优先级（收入 > 留存 > 其他）
  2. 权衡后人工决策
  3. 延长实验观察更多数据
预防：指标优先级 + 人工介入 + 延长观察
```

## 实验流量分配与分层完整实现

```python
class ExperimentTrafficAllocator:
    """流量分配：互斥分组 + 分层实验 + 用户一致性"""

    MAX_EXPERIMENTS_PER_LAYER = 5
    TOTAL_TRAFFIC_BUCKETS = 1000

    def assign_user_to_experiments(self, user_id):
        """为用户分配所有活跃实验的组"""
        # 1. 计算用户哈希桶
        user_bucket = self._hash_to_bucket(user_id)

        # 2. 获取所有活跃实验层
        layers = self.db.query(
            "SELECT DISTINCT layer FROM experiments "
            "WHERE status = 'running'")

        assignments = {}

        for layer in layers:
            layer_name = layer["layer"]

            # 3. 获取该层所有运行中的实验
            experiments = self.db.query(
                "SELECT * FROM experiments "
                "WHERE layer = %s AND status = 'running' "
                "ORDER BY priority DESC", layer_name)

            # 4. 层内互斥分配
            range_start = 0
            for exp in experiments:
                traffic_pct = exp["traffic_pct"]
                range_end = range_start + int(traffic_pct * self.TOTAL_TRAFFIC_BUCKETS / 100)

                if range_start <= user_bucket < range_end:
                    # 用户在此实验的流量范围内
                    group = self._assign_group(user_id, exp["id"], exp.get("variants", []))
                    assignments[exp["id"]] = {
                        "experiment_id": exp["id"],
                        "experiment_name": exp["name"],
                        "layer": layer_name,
                        "group": group
                    }
                    break  # 层内互斥，一个用户只在一个实验中

                range_start = range_end

        # 5. 缓存分配结果
        self.redis.setex(f"exp_assignment:{user_id}", 3600,
            json.dumps(assignments))

        return assignments

    def _hash_to_bucket(self, user_id):
        """将用户 ID 哈希到 0-999 的桶"""
        hash_val = int(hashlib.md5(user_id.encode()).hexdigest(), 16)
        return hash_val % self.TOTAL_TRAFFIC_BUCKETS

    def _assign_group(self, user_id, experiment_id, variants):
        """分配实验组"""
        # 基于用户 ID + 实验 ID 确定性分配
        hash_key = f"{user_id}:{experiment_id}"
        hash_val = int(hashlib.md5(hash_key.encode()).hexdigest(), 16)

        # 按变体权重分配
        total_weight = sum(v.get("weight", 1) for v in variants)
        cumulative = 0

        for variant in variants:
            weight = variant.get("weight", 1)
            cumulative += weight
            if hash_val % total_weight < cumulative:
                return variant["name"]

        return variants[0]["name"]  # 默认

    def check_layer_conflicts(self, layer_name):
        """检查层内实验冲突"""
        experiments = self.db.query(
            "SELECT * FROM experiments "
            "WHERE layer = %s AND status IN ('running', 'scheduled')",
            layer_name)

        total_traffic = sum(exp["traffic_pct"] for exp in experiments)

        conflicts = []
        if total_traffic > 100:
            conflicts.append({
                "type": "traffic_overflow",
                "message": f"层 {layer_name} 总流量 {total_traffic}% 超过 100%",
                "experiments": [exp["name"] for exp in experiments]
            })

        if len(experiments) > self.MAX_EXPERIMENTS_PER_LAYER:
            conflicts.append({
                "type": "too_many_experiments",
                "message": f"层 {layer_name} 实验数 {len(experiments)} 超过上限 {self.MAX_EXPERIMENTS_PER_LAYER}"
            })

        return {"layer": layer_name, "total_traffic_pct": total_traffic,
                "experiment_count": len(experiments), "conflicts": conflicts}

    def get_user_assignment(self, user_id, experiment_id):
        """获取用户的实验分配"""
        # 先查缓存
        cached = self.redis.get(f"exp_assignment:{user_id}")
        if cached:
            assignments = json.loads(cached)
            if experiment_id in assignments:
                return assignments[experiment_id]

        # 重新计算
        assignments = self.assign_user_to_experiments(user_id)
        return assignments.get(experiment_id, {"group": "control"})
```

## 异常场景补充

### 场景：分层实验交叉干扰

```
触发：层 1 的实验影响页面布局 → 层 2 的实验影响按钮颜色 → 两层实验效果互相干扰 → 结果不准
检测：
  1. 跨层实验的指标有相关性 → 交叉干扰
  2. 分层后实验效果与预期不符 → 干扰
处理：
  1. 层间独立性检验（检查层间指标相关性）
  2. 不独立的实验放同一层（互斥）
  3. 使用因子设计分析交互效应
预防：独立性检验 + 互斥分组 + 交互分析
```

### 场景：流量分配不一致

```
触发：用户刷新页面 → 重新分配实验组 → 从控制组变为实验组 → 体验不一致
检测：
  1. 同一用户不同请求分配到不同组 → 不一致
  2. 用户反馈界面频繁变化 → 分配漂移
处理：
  1. 分配结果持久化（缓存 + DB）
  2. 确定性哈希（同一用户始终分到同一组）
  3. 分配后不再变更
预防：确定性哈希 + 持久化 + 不可变分配
```

## 实验效果评估与决策完整实现

```python
import math
import logging
from dataclasses import dataclass, field
from typing import List, Optional, Dict, Tuple
from scipy.stats import norm, t as t_dist, chi2
from enum import Enum
from datetime import datetime

logger = logging.getLogger(__name__)


class MetricType(Enum):
    RATIO = "ratio"           # 转化率、点击率
    CONTINUOUS = "continuous" # 停留时长、客单价
    COUNT = "count"           # 页面浏览次数


class DecisionRecommendation(Enum):
    SHIP_IT = "ship_it"                         # 上线实验组
    SHIP_CONTROL = "ship_control"               # 保持对照组
    EXTEND_EXPERIMENT = "extend_experiment"     # 延长实验
    INCONCLUSIVE = "inconclusive"               # 无结论
    INVALID = "invalid"                         # 实验无效


@dataclass
class GroupStatistics:
    """单组统计量"""
    n: int                     # 样本量
    mean: float                # 均值
    stddev: float              # 标准差
    sum_val: float = 0.0       # 总和（比例型指标用）
    count_val: int = 0         # 计数（比例型指标用）


@dataclass
class StatisticalTestResult:
    """统计检验结果"""
    metric_name: str
    metric_type: MetricType
    control_stats: GroupStatistics
    treatment_stats: GroupStatistics
    p_value: float
    confidence_interval: Tuple[float, float]
    effect_size: float              # Cohen's d 或 h
    relative_change: float          # 相对变化率
    mde_observed: float             # 实际观察到的最小可检测效果
    is_significant: bool
    test_method: str
    alpha: float


@dataclass
class SampleSizeEstimate:
    """样本量估算结果"""
    experiment_id: str
    current_n_per_group: int
    required_n_per_group: int
    remaining_n_per_group: int
    days_elapsed: int
    estimated_remaining_days: float
    current_power: float
    target_power: float
    observed_effect_size: float
    mde: float


@dataclass
class GoNoGoDecision:
    """Go/No-Go 决策结果"""
    experiment_id: str
    decision: DecisionRecommendation
    confidence_level: str           # "high" / "medium" / "low"
    statistical_significant: bool
    business_impact_assessment: str  # 商业影响评估
    practical_significant: bool
    minimum_effect_threshold: float
    revenue_lift: float
    revenue_lift_ci: Tuple[float, float]
    risk_factors: List[str]
    recommendation_detail: str


@dataclass
class EvaluationReport:
    """完整评估报告"""
    experiment_id: str
    experiment_name: str
    evaluation_time: datetime
    duration_days: int
    # 统计检验结果
    metric_results: List[StatisticalTestResult]
    # 多重比较校正
    adjusted_p_values: List[float]
    correction_method: str
    # 样本量评估
    sample_size_estimate: SampleSizeEstimate
    # 决策
    decision: GoNoGoDecision
    # 数据质量
    srm_detected: bool
    srm_p_value: float
    paradox_detected: bool
    # 摘要
    executive_summary: str
    recommended_action: str


class ExperimentEvaluationService:
    """
    实验效果评估与决策服务

    核心能力：
    1. 对连续型指标执行 Welch's t 检验
    2. 对比例型指标执行卡方检验
    3. 计算 Cohen's d 效果量和置信区间
    4. 基于 power analysis 估算所需样本量
    5. 综合统计显著性 + 商业影响 + 实际显著性做出决策
    6. 生成完整评估报告
    """

    def __init__(self, db_client, redis_client, config: dict = None):
        self.db = db_client
        self.redis = redis_client
        self.config = config or {}
        self.alpha = self.config.get("alpha", 0.05)
        self.power = self.config.get("power", 0.80)
        self.minimum_effect_threshold = self.config.get("minimum_effect_threshold", 0.01)
        self.revenue_per_conversion = self.config.get("revenue_per_conversion", 100.0)

    def evaluate_experiment(self, experiment_id: str) -> List[StatisticalTestResult]:
        """
        运行统计检验评估实验效果

        对每个指标：
        - 连续型指标：Welch's t 检验（不假设等方差）
        - 比例型指标：卡方检验（Chi-squared test）
        - 计算效果量（Cohen's d 或 h）、p 值、置信区间、MDE
        - 以 95% 置信度判定统计显著性
        """
        experiment = self._get_experiment(experiment_id)
        if not experiment:
            raise ValueError(f"实验 {experiment_id} 不存在")

        metrics = self._get_experiment_metrics(experiment_id)
        results = []

        for metric in metrics:
            control_data = self._load_group_stats(
                experiment_id, metric["metric_name"], "control"
            )
            treatment_data = self._load_group_stats(
                experiment_id, metric["metric_name"], "treatment"
            )

            if control_data.n < 30 or treatment_data.n < 30:
                logger.warning(
                    f"指标 {metric['metric_name']} 样本量不足: "
                    f"control={control_data.n}, treatment={treatment_data.n}"
                )

            metric_type = MetricType(metric["metric_type"])

            if metric_type == MetricType.CONTINUOUS:
                result = self._ttest_continuous(
                    metric["metric_name"], control_data, treatment_data
                )
            elif metric_type == MetricType.RATIO:
                result = self._chi_squared_ratio(
                    metric["metric_name"], control_data, treatment_data
                )
            elif metric_type == MetricType.COUNT:
                result = self._ttest_continuous(
                    metric["metric_name"], control_data, treatment_data
                )
            else:
                continue

            # 计算观察到的 MDE
            result.mde_observed = self._compute_observed_mde(
                control_data, treatment_data, metric_type
            )

            # 判定统计显著性（p < alpha）
            result.is_significant = result.p_value < self.alpha
            result.alpha = self.alpha

            # 保存结果到数据库
            self._save_test_result(experiment_id, result)

            results.append(result)

        return results

    def _ttest_continuous(self, metric_name: str,
                          control: GroupStatistics,
                          treatment: GroupStatistics) -> StatisticalTestResult:
        """
        Welch's t 检验（连续型指标）

        不假设两组方差相等，使用 Welch-Satterthwaite 自由度修正。
        公式：
          t = (mean_t - mean_c) / sqrt(s_t^2/n_t + s_c^2/n_c)
          df = (s_t^2/n_t + s_c^2/n_c)^2 / ((s_t^2/n_t)^2/(n_t-1) + (s_c^2/n_c)^2/(n_c-1))
        """
        mean_diff = treatment.mean - control.mean

        # Welch's t 统计量
        se = math.sqrt(
            (treatment.stddev ** 2 / treatment.n) +
            (control.stddev ** 2 / control.n)
        )
        t_statistic = mean_diff / se if se > 0 else 0.0

        # Welch-Satterthwaite 自由度
        var_t = treatment.stddev ** 2 / treatment.n
        var_c = control.stddev ** 2 / control.n
        if var_t + var_c > 0:
            df = (var_t + var_c) ** 2 / (
                (var_t ** 2 / (treatment.n - 1)) +
                (var_c ** 2 / (control.n - 1))
            ) if treatment.n > 1 and control.n > 1 else 1.0
        else:
            df = 1.0

        # 双尾 p 值
        p_value = 2 * (1 - t_dist.cdf(abs(t_statistic), df=max(df, 1)))

        # Cohen's d 效果量
        pooled_stddev = math.sqrt(
            ((treatment.n - 1) * treatment.stddev ** 2 +
             (control.n - 1) * control.stddev ** 2) /
            (treatment.n + control.n - 2)
        ) if treatment.n + control.n > 2 else 1.0
        cohens_d = mean_diff / pooled_stddev if pooled_stddev > 0 else 0.0

        # 置信区间（95%）
        ci_half = norm.ppf(1 - self.alpha / 2) * se
        ci_lower = mean_diff - ci_half
        ci_upper = mean_diff + ci_half

        # 相对变化率
        relative_change = mean_diff / control.mean if control.mean != 0 else 0.0

        return StatisticalTestResult(
            metric_name=metric_name,
            metric_type=MetricType.CONTINUOUS,
            control_stats=control,
            treatment_stats=treatment,
            p_value=p_value,
            confidence_interval=(ci_lower, ci_upper),
            effect_size=cohens_d,
            relative_change=relative_change,
            mde_observed=0.0,
            is_significant=p_value < self.alpha,
            test_method="welch_t_test",
            alpha=self.alpha,
        )

    def _chi_squared_ratio(self, metric_name: str,
                           control: GroupStatistics,
                           treatment: GroupStatistics) -> StatisticalTestResult:
        """
        卡方检验（比例型指标，如转化率、点击率）

        构建 2x2 列联表，计算 Pearson 卡方统计量。
        同时计算 Cohen's h 作为比例型效果量。
        """
        # 比例
        p_control = control.sum_val / control.n if control.n > 0 else 0.0
        p_treatment = treatment.sum_val / treatment.n if treatment.n > 0 else 0.0

        # 2x2 列联表
        a = treatment.sum_val               # 实验组转化数
        b = treatment.n - treatment.sum_val  # 实验组未转化数
        c = control.sum_val                  # 对照组转化数
        d = control.n - control.sum_val      # 对照组未转化数

        total = a + b + c + d
        if total == 0:
            return StatisticalTestResult(
                metric_name=metric_name,
                metric_type=MetricType.RATIO,
                control_stats=control,
                treatment_stats=treatment,
                p_value=1.0,
                confidence_interval=(0.0, 0.0),
                effect_size=0.0,
                relative_change=0.0,
                mde_observed=0.0,
                is_significant=False,
                test_method="chi_squared",
                alpha=self.alpha,
            )

        # Pearson 卡方统计量（带 Yates 修正）
        n1 = a + b  # 实验组总数
        n2 = c + d  # 对照组总数
        p_pool = (a + c) / total
        expected_a = n1 * p_pool
        expected_b = n1 * (1 - p_pool)
        expected_c = n2 * p_pool
        expected_d = n2 * (1 - p_pool)

        # Yates 修正
        chi2_stat = (
            (abs(a - expected_a) - 0.5) ** 2 / expected_a +
            (abs(b - expected_b) - 0.5) ** 2 / expected_b +
            (abs(c - expected_c) - 0.5) ** 2 / expected_c +
            (abs(d - expected_d) - 0.5) ** 2 / expected_d
        ) if min(expected_a, expected_b, expected_c, expected_d) > 5 else (
            (a - expected_a) ** 2 / expected_a +
            (b - expected_b) ** 2 / expected_b +
            (c - expected_c) ** 2 / expected_c +
            (d - expected_d) ** 2 / expected_d
        )

        # 卡方检验 p 值（df=1）
        p_value = 1 - chi2.cdf(chi2_stat, df=1)

        # Cohen's h 效果量（比例型）
        # h = 2 * arcsin(sqrt(p1)) - 2 * arcsin(sqrt(p2))
        # 小: 0.2, 中: 0.5, 大: 0.8
        phi_control = 2 * math.asin(math.sqrt(min(max(p_control, 0.0), 1.0)))
        phi_treatment = 2 * math.asin(math.sqrt(min(max(p_treatment, 0.0), 1.0)))
        cohens_h = phi_treatment - phi_control

        # 比例差的置信区间
        se_diff = math.sqrt(
            p_treatment * (1 - p_treatment) / treatment.n +
            p_control * (1 - p_control) / control.n
        ) if treatment.n > 0 and control.n > 0 else 0.0
        diff = p_treatment - p_control
        ci_half = norm.ppf(1 - self.alpha / 2) * se_diff
        ci_lower = diff - ci_half
        ci_upper = diff + ci_half

        # 相对变化率
        relative_change = diff / p_control if p_control > 0 else 0.0

        return StatisticalTestResult(
            metric_name=metric_name,
            metric_type=MetricType.RATIO,
            control_stats=control,
            treatment_stats=treatment,
            p_value=p_value,
            confidence_interval=(ci_lower, ci_upper),
            effect_size=cohens_h,
            relative_change=relative_change,
            mde_observed=0.0,
            is_significant=p_value < self.alpha,
            test_method="chi_squared",
            alpha=self.alpha,
        )

    def _compute_observed_mde(self, control: GroupStatistics,
                              treatment: GroupStatistics,
                              metric_type: MetricType) -> float:
        """
        计算当前样本量下能检测到的最小效果量（MDE）

        MDE = (z_alpha/2 + z_beta) * sqrt(2 * pooled_var / n_per_group)
        这是当前实验设置下理论上能检测到的最小效果量。
        """
        z_alpha = norm.ppf(1 - self.alpha / 2)
        z_beta = norm.ppf(self.power)

        n_min = min(control.n, treatment.n)

        if metric_type == MetricType.RATIO:
            p_control = control.sum_val / control.n if control.n > 0 else 0.05
            pooled_var = 2 * p_control * (1 - p_control)
        else:
            pooled_var = (
                (control.stddev ** 2 + treatment.stddev ** 2) / 2
                if control.stddev > 0 and treatment.stddev > 0
                else 1.0
            )

        if n_min <= 0:
            return float('inf')

        mde = (z_alpha + z_beta) * math.sqrt(2 * pooled_var / n_min)
        return mde

    def calculate_sample_size_needed(self, experiment_id: str) -> SampleSizeEstimate:
        """
        基于当前效果量计算剩余所需样本量和天数

        Power analysis 公式（双样本检验）：
          n_per_group = ((z_alpha/2 + z_beta)^2 * 2 * sigma^2) / delta^2

        其中 delta 是期望检测的最小效果量，取当前观察到的效果量
        和实验配置的 MDE 中较大的一个。

        剩余天数 = 剩余样本量 / 日均每组分流量
        """
        experiment = self._get_experiment(experiment_id)
        metrics = self._get_experiment_metrics(experiment_id)

        # 优先使用核心指标
        primary_metric = next(
            (m for m in metrics if m.get("is_primary", False)),
            metrics[0] if metrics else None
        )
        if not primary_metric:
            raise ValueError(f"实验 {experiment_id} 无可用指标")

        metric_name = primary_metric["metric_name"]
        metric_type = MetricType(primary_metric["metric_type"])

        control = self._load_group_stats(experiment_id, metric_name, "control")
        treatment = self._load_group_stats(experiment_id, metric_name, "treatment")

        # 观察到的效果量
        if metric_type == MetricType.RATIO:
            p_c = control.sum_val / control.n if control.n > 0 else 0.05
            p_t = treatment.sum_val / treatment.n if treatment.n > 0 else 0.05
            observed_delta = abs(p_t - p_c)
            baseline_rate = p_c
        else:
            observed_delta = abs(treatment.mean - control.mean)
            baseline_rate = control.mean if control.mean != 0 else 1.0

        # 使用 MDE 和观察效果量中较大的，避免因效果量过小导致所需样本爆炸
        configured_mde = experiment.get("mde", 0.05) if experiment else 0.05
        delta = max(observed_delta, baseline_rate * configured_mde)

        if delta <= 0:
            delta = baseline_rate * configured_mde if baseline_rate > 0 else 0.01

        # Power analysis 计算所需样本量
        z_alpha = norm.ppf(1 - self.alpha / 2)
        z_beta = norm.ppf(self.power)

        if metric_type == MetricType.RATIO:
            p_pool = (control.sum_val + treatment.sum_val) / (control.n + treatment.n) \
                if (control.n + treatment.n) > 0 else 0.05
            sigma_sq = 2 * p_pool * (1 - p_pool)
        else:
            pooled_std = math.sqrt(
                (control.stddev ** 2 + treatment.stddev ** 2) / 2
            ) if control.stddev > 0 or treatment.stddev > 0 else 1.0
            sigma_sq = 2 * pooled_std ** 2

        n_required = math.ceil((z_alpha + z_beta) ** 2 * sigma_sq / (delta ** 2))
        n_required = max(n_required, 100)  # 最少每组 100 个样本

        # 当前每组样本量
        current_n = min(control.n, treatment.n)

        # 剩余样本量
        remaining_n = max(n_required - current_n, 0)

        # 估算日均分流量
        days_elapsed = max(experiment.get("days_elapsed", 1), 1) if experiment else 1
        daily_traffic_per_group = current_n / days_elapsed if days_elapsed > 0 else 100

        # 估算剩余天数
        estimated_remaining_days = (
            remaining_n / daily_traffic_per_group
            if daily_traffic_per_group > 0 else float('inf')
        )

        # 当前统计功效
        if observed_delta > 0 and sigma_sq > 0:
            noncentrality = observed_delta / math.sqrt(sigma_sq / max(current_n, 1))
            current_power = 1 - norm.cdf(z_alpha / 2 - noncentrality)
        else:
            current_power = 0.0

        estimate = SampleSizeEstimate(
            experiment_id=experiment_id,
            current_n_per_group=current_n,
            required_n_per_group=n_required,
            remaining_n_per_group=remaining_n,
            days_elapsed=days_elapsed,
            estimated_remaining_days=round(estimated_remaining_days, 1),
            current_power=round(current_power, 4),
            target_power=self.power,
            observed_effect_size=round(observed_delta, 6),
            mde=round(delta, 6),
        )

        # 保存估算结果
        self._save_sample_size_estimate(estimate)

        return estimate

    def make_go_no_go_decision(self, experiment_id: str) -> GoNoGoDecision:
        """
        综合统计显著性 + 商业影响 + 实际显著性做出 Go/No-Go 决策

        决策框架：
        1. 统计显著性：校正后 p < 0.05
        2. 商业影响：收入提升 = 转化率变化 * 用户基数 * 客单价
        3. 实际显著性：效果量是否超过最小实际效果阈值

        三重检验全部通过 → SHIP_IT（上线实验组）
        统计显著但实际不显著 → SHIP_CONTROL（保持现状，效果太小不值得上线）
        不显著且样本充足 → SHIP_CONTROL
        不显著且样本不足 → EXTEND_EXPERIMENT
        SRM 或辛普森悖论 → INVALID
        """
        # 1. 获取统计检验结果
        test_results = self.evaluate_experiment(experiment_id)
        sample_estimate = self.calculate_sample_size_needed(experiment_id)
        experiment = self._get_experiment(experiment_id)

        # 2. 多重比较校正
        p_values = [r.p_value for r in test_results]
        metrics = self._get_experiment_metrics(experiment_id)
        is_primary = [m.get("is_primary", False) for m in metrics]
        adjusted_p = self._apply_hybrid_correction(p_values, is_primary)

        # 3. 数据质量检查
        srm_detected = self._check_srm(experiment_id)
        paradox_detected = self._check_paradox(experiment_id)

        risk_factors = []
        if srm_detected:
            risk_factors.append("SRM 检测到分组偏差，实验结果可能不可信")
        if paradox_detected:
            risk_factors.append("辛普森悖论检测到，整体结论可能与子群结论矛盾")

        # 数据质量不通过 → 实验无效
        if srm_detected or paradox_detected:
            return GoNoGoDecision(
                experiment_id=experiment_id,
                decision=DecisionRecommendation.INVALID,
                confidence_level="low",
                statistical_significant=False,
                business_impact_assessment="数据质量问题导致无法评估",
                practical_significant=False,
                minimum_effect_threshold=self.minimum_effect_threshold,
                revenue_lift=0.0,
                revenue_lift_ci=(0.0, 0.0),
                risk_factors=risk_factors,
                recommendation_detail="实验数据存在质量问题（SRM或辛普森悖论），建议排查实验配置后重新运行。",
            )

        # 4. 找到核心指标
        primary_results = [
            (r, adj_p) for r, adj_p in zip(test_results, adjusted_p)
            if any(m["metric_name"] == r.metric_name and m.get("is_primary", False)
                   for m in metrics)
        ]

        if not primary_results:
            primary_results = [(test_results[0], adjusted_p[0])] \
                if test_results else []

        # 5. 统计显著性判断（使用校正后 p 值）
        any_significant = any(adj_p < self.alpha for _, adj_p in primary_results)

        # 6. 计算商业影响（收入提升）
        revenue_lift = 0.0
        revenue_lift_ci = (0.0, 0.0)
        for result, adj_p in primary_results:
            if result.metric_type == MetricType.RATIO:
                # 转化率变化 * 日活 * 客单价 * 30天
                p_control = result.control_stats.sum_val / result.control_stats.n \
                    if result.control_stats.n > 0 else 0
                delta_rate = result.treatment_stats.mean - result.control_stats.mean \
                    if hasattr(result.treatment_stats, 'mean') else 0
                if result.control_stats.n > 0:
                    conv_rate_change = (
                        result.treatment_stats.sum_val / result.treatment_stats.n -
                        result.control_stats.sum_val / result.control_stats.n
                    )
                else:
                    conv_rate_change = 0.0
                daily_users = result.control_stats.n + result.treatment_stats.n
                monthly_revenue_lift = (
                    conv_rate_change * daily_users * self.revenue_per_conversion * 30
                )
                revenue_lift = monthly_revenue_lift
                # 置信区间
                ci = result.confidence_interval
                revenue_ci_lower = ci[0] * daily_users * self.revenue_per_conversion * 30
                revenue_ci_upper = ci[1] * daily_users * self.revenue_per_conversion * 30
                revenue_lift_ci = (revenue_ci_lower, revenue_ci_upper)
            else:
                # 连续型指标：均值变化 * 用户数 * 30天
                delta = result.treatment_stats.mean - result.control_stats.mean
                daily_users = result.control_stats.n + result.treatment_stats.n
                monthly_revenue_lift = delta * daily_users * 30
                revenue_lift = monthly_revenue_lift
                ci = result.confidence_interval
                revenue_ci_lower = ci[0] * daily_users * 30
                revenue_ci_upper = ci[1] * daily_users * 30
                revenue_lift_ci = (revenue_ci_lower, revenue_ci_upper)

        # 7. 实际显著性判断（效果量是否超过阈值）
        practical_significant = any(
            abs(r.effect_size) > self.minimum_effect_threshold
            for r, _ in primary_results
        )

        # 8. 样本充足性
        sample_sufficient = (
            sample_estimate.current_n_per_group >=
            sample_estimate.required_n_per_group * 0.9
        )

        # 9. 置信度等级
        if any_significant and practical_significant:
            confidence = "high"
        elif any_significant and not practical_significant:
            confidence = "medium"
        else:
            confidence = "low"

        # 10. 决策逻辑
        if any_significant and practical_significant:
            # 统计显著 + 实际显著 → 上线实验组
            decision = DecisionRecommendation.SHIP_IT
            detail = (
                f"核心指标统计显著（校正后 p<0.05）且实际显著（效果量>{self.minimum_effect_threshold}），"
                f"预估月收入提升 {revenue_lift:.2f} 元，"
                f"95% CI [{revenue_lift_ci[0]:.2f}, {revenue_lift_ci[1]:.2f}]。"
                f"建议全量上线实验组。"
            )
        elif any_significant and not practical_significant:
            # 统计显著但实际不显著 → 保持现状
            decision = DecisionRecommendation.SHIP_CONTROL
            detail = (
                f"核心指标统计显著但效果量过小（<{self.minimum_effect_threshold}），"
                f"实际业务价值不足，预估月收入提升 {revenue_lift:.2f} 元。"
                f"建议保持对照组，不值得为此上线。"
            )
            risk_factors.append("统计显著但效果量小，上线 ROI 可能不值得")
        elif not any_significant and sample_sufficient:
            # 样本充足但不显著 → 保持现状
            decision = DecisionRecommendation.SHIP_CONTROL
            detail = (
                f"样本量已充足（每组 {sample_estimate.current_n_per_group}），"
                f"但核心指标未达统计显著（校正后 p>0.05），"
                f"实验组效果不明确，建议保持对照组。"
            )
        elif not any_significant and not sample_sufficient:
            # 样本不足 → 延长实验
            decision = DecisionRecommendation.EXTEND_EXPERIMENT
            detail = (
                f"当前统计功效 {sample_estimate.current_power:.1%}，"
                f"每组还需 {sample_estimate.remaining_n_per_group} 个样本，"
                f"预计还需 {sample_estimate.estimated_remaining_days:.0f} 天。"
                f"建议延长实验至达到 80% 功效。"
            )
            risk_factors.append(
                f"样本量不足（当前 {sample_estimate.current_n_per_group}，"
                f"需要 {sample_estimate.required_n_per_group}）"
            )
        else:
            decision = DecisionRecommendation.INCONCLUSIVE
            detail = "无法做出明确判断，建议人工审查实验数据。"

        return GoNoGoDecision(
            experiment_id=experiment_id,
            decision=decision,
            confidence_level=confidence,
            statistical_significant=any_significant,
            business_impact_assessment=f"预估月收入变化: {revenue_lift:.2f} 元",
            practical_significant=practical_significant,
            minimum_effect_threshold=self.minimum_effect_threshold,
            revenue_lift=round(revenue_lift, 2),
            revenue_lift_ci=(round(revenue_lift_ci[0], 2),
                            round(revenue_lift_ci[1], 2)),
            risk_factors=risk_factors,
            recommendation_detail=detail,
        )

    def generate_evaluation_report(self, experiment_id: str) -> EvaluationReport:
        """
        生成完整评估报告

        包含：
        1. 所有指标的统计检验结果
        2. 多重比较校正后 p 值
        3. 样本量评估
        4. Go/No-Go 决策
        5. 数据质量检查结果
        6. 执行摘要和建议行动
        """
        experiment = self._get_experiment(experiment_id)
        test_results = self.evaluate_experiment(experiment_id)
        sample_estimate = self.calculate_sample_size_needed(experiment_id)
        decision = self.make_go_no_go_decision(experiment_id)
        metrics = self._get_experiment_metrics(experiment_id)

        # 多重比较校正
        p_values = [r.p_value for r in test_results]
        is_primary = [m.get("is_primary", False) for m in metrics]
        adjusted_p = self._apply_hybrid_correction(p_values, is_primary)

        # 数据质量检查
        srm_detected = self._check_srm(experiment_id)
        paradox_detected = self._check_paradox(experiment_id)
        srm_p_value = self._get_srm_p_value(experiment_id)

        # 生成执行摘要
        significant_count = sum(1 for ap in adjusted_p if ap < self.alpha)
        total_count = len(test_results)
        executive_summary = self._build_executive_summary(
            experiment, test_results, adjusted_p, decision,
            srm_detected, paradox_detected, significant_count, total_count
        )

        # 建议行动
        recommended_action = self._build_recommended_action(decision, test_results)

        report = EvaluationReport(
            experiment_id=experiment_id,
            experiment_name=experiment.get("name", "") if experiment else "",
            evaluation_time=datetime.now(),
            duration_days=sample_estimate.days_elapsed,
            metric_results=test_results,
            adjusted_p_values=[round(p, 8) for p in adjusted_p],
            correction_method="hybrid(bonferroni_holm+bh_fdr)",
            sample_size_estimate=sample_estimate,
            decision=decision,
            srm_detected=srm_detected,
            srm_p_value=srm_p_value,
            paradox_detected=paradox_detected,
            executive_summary=executive_summary,
            recommended_action=recommended_action,
        )

        # 保存报告
        self._save_evaluation_report(report)

        return report

    def _build_executive_summary(self, experiment, test_results, adjusted_p,
                                  decision, srm_detected, paradox_detected,
                                  significant_count, total_count) -> str:
        """构建执行摘要"""
        parts = []
        parts.append(f"实验 {experiment.get('name', '')} 评估结果：")

        if srm_detected:
            parts.append("【警告】检测到样本比例偏差（SRM），实验数据可能不可信。")
        if paradox_detected:
            parts.append("【警告】检测到辛普森悖论，整体结论可能不代表子群真实效果。")

        parts.append(
            f"共 {total_count} 个指标中 {significant_count} 个在校正后统计显著。"
        )

        for result, adj_p in zip(test_results, adjusted_p):
            sig_mark = "**显著**" if adj_p < self.alpha else "不显著"
            parts.append(
                f"  - {result.metric_name}: {sig_mark}, "
                f"校正p={adj_p:.4f}, "
                f"效果量={result.effect_size:.4f}, "
                f"相对变化={result.relative_change:+.2%}, "
                f"95%CI=[{result.confidence_interval[0]:.6f}, "
                f"{result.confidence_interval[1]:.6f}]"
            )

        parts.append(f"决策建议: {decision.decision.value}")
        parts.append(f"置信度: {decision.confidence_level}")
        if decision.risk_factors:
            parts.append("风险因素:")
            for risk in decision.risk_factors:
                parts.append(f"  - {risk}")

        return "\n".join(parts)

    def _build_recommended_action(self, decision: GoNoGoDecision,
                                   test_results: List[StatisticalTestResult]) -> str:
        """构建建议行动"""
        if decision.decision == DecisionRecommendation.SHIP_IT:
            return (
                "建议全量上线实验组版本。上线后需持续监控核心指标 1-2 周，"
                "确认效果与实验期一致。如出现指标回退，准备回滚方案。"
            )
        elif decision.decision == DecisionRecommendation.SHIP_CONTROL:
            return "建议保持对照组版本，实验组效果不明确或实际价值不足。"
        elif decision.decision == DecisionRecommendation.EXTEND_EXPERIMENT:
            return (
                f"建议延长实验至 {decision.recommendation_detail}，"
                "以获得足够的统计功效。延长期间不要修改实验配置。"
            )
        elif decision.decision == DecisionRecommendation.INVALID:
            return "实验数据存在质量问题，建议排查后重新运行实验。"
        else:
            return "结果不确定，建议人工审查实验数据和配置。"

    def _apply_hybrid_correction(self, p_values: List[float],
                                  is_primary: List[bool]) -> List[float]:
        """混合多重比较校正：核心指标用 Bonferroni-Holm，辅助指标用 BH-FDR"""
        m = len(p_values)
        if m == 0:
            return []

        adjusted = [1.0] * m

        # 核心指标：Bonferroni-Holm
        primary_indices = [i for i, p in enumerate(is_primary) if p]
        if primary_indices:
            primary_pvals = [p_values[i] for i in primary_indices]
            sorted_pri = sorted(range(len(primary_pvals)),
                               key=lambda k: primary_pvals[k])
            for rank, orig_idx in enumerate(sorted_pri):
                raw_adj = primary_pvals[orig_idx] * (len(primary_indices) - rank)
                if rank == 0:
                    adjusted_sorted = min(raw_adj, 1.0)
                else:
                    prev_val = adjusted[primary_indices[sorted_pri[rank - 1]]]
                    adjusted_sorted = max(prev_val, min(raw_adj, 1.0))
                adjusted[primary_indices[orig_idx]] = adjusted_sorted

        # 辅助指标：BH-FDR
        secondary_indices = [i for i, p in enumerate(is_primary) if not p]
        if secondary_indices:
            sec_pvals = [p_values[i] for i in secondary_indices]
            m_sec = len(sec_pvals)
            sorted_sec = sorted(range(m_sec), key=lambda k: sec_pvals[k])
            for k in range(m_sec - 1, -1, -1):
                rank = k + 1
                raw_adj = sec_pvals[sorted_sec[k]] * m_sec / rank
                if k == m_sec - 1:
                    adj_val = min(raw_adj, 1.0)
                else:
                    adj_val = min(adjusted[secondary_indices[sorted_sec[k + 1]]],
                                 min(raw_adj, 1.0))
                adjusted[secondary_indices[sorted_sec[k]]] = adj_val

        return adjusted

    # ========== 数据加载辅助方法 ==========

    def _get_experiment(self, experiment_id: str) -> Optional[dict]:
        """获取实验配置"""
        row = self.db.query_one(
            "SELECT * FROM experiments WHERE id = %s", experiment_id
        )
        return row

    def _get_experiment_metrics(self, experiment_id: str) -> List[dict]:
        """获取实验指标配置"""
        return self.db.query(
            "SELECT * FROM metrics WHERE experiment_id = %s", experiment_id
        )

    def _load_group_stats(self, experiment_id: str, metric_name: str,
                           group_name: str) -> GroupStatistics:
        """从 ClickHouse 加载某组某指标的统计量"""
        row = self.db.query_one("""
            SELECT variant, event_count as n, mean_value as mean,
                   stddev_value as stddev, sum_value as sum_val
            FROM experiment_metric_hourly
            WHERE experiment_id = %s AND metric_name = %s AND variant = %s
        """, experiment_id, metric_name, group_name)

        if not row:
            return GroupStatistics(n=0, mean=0.0, stddev=0.0,
                                  sum_val=0.0, count_val=0)

        return GroupStatistics(
            n=int(row.get("n", 0)),
            mean=float(row.get("mean", 0)),
            stddev=float(row.get("stddev", 0)),
            sum_val=float(row.get("sum_val", 0)),
            count_val=int(row.get("n", 0)),
        )

    def _check_srm(self, experiment_id: str) -> bool:
        """SRM 检测"""
        counts = self.db.query("""
            SELECT group_name, COUNT(*) as n
            FROM user_assignments
            WHERE experiment_id = %s
            GROUP BY group_name
        """, experiment_id)

        if len(counts) < 2:
            return False

        control_n = next((c["n"] for c in counts if c["group_name"] == "control"), 0)
        treatment_n = next((c["n"] for c in counts if c["group_name"] == "treatment"), 0)
        total = control_n + treatment_n

        if total == 0:
            return False

        expected_control = total * 0.5
        expected_treatment = total * 0.5
        chi2_val = (
            (control_n - expected_control) ** 2 / expected_control +
            (treatment_n - expected_treatment) ** 2 / expected_treatment
        )
        p_value = 1 - chi2.cdf(chi2_val, df=1)
        return p_value < 0.01

    def _get_srm_p_value(self, experiment_id: str) -> float:
        """获取 SRM p 值"""
        counts = self.db.query("""
            SELECT group_name, COUNT(*) as n
            FROM user_assignments
            WHERE experiment_id = %s
            GROUP BY group_name
        """, experiment_id)

        if len(counts) < 2:
            return 1.0

        control_n = next((c["n"] for c in counts if c["group_name"] == "control"), 0)
        treatment_n = next((c["n"] for c in counts if c["group_name"] == "treatment"), 0)
        total = control_n + treatment_n

        if total == 0:
            return 1.0

        expected = total * 0.5
        chi2_val = (control_n - expected) ** 2 / expected + \
                   (treatment_n - expected) ** 2 / expected
        return float(1 - chi2.cdf(chi2_val, df=1))

    def _check_paradox(self, experiment_id: str) -> bool:
        """辛普森悖论检测（简化版）"""
        dimensions = ["device_type", "user_tier", "region"]
        for dim in dimensions:
            rows = self.db.query(f"""
                SELECT {dim} as stratum,
                       SUM(CASE WHEN variant='control' THEN n ELSE 0 END) as cn,
                       SUM(CASE WHEN variant='control' THEN conv ELSE 0 END) as cc,
                       SUM(CASE WHEN variant='treatment' THEN n ELSE 0 END) as tn,
                       SUM(CASE WHEN variant='treatment' THEN conv ELSE 0 END) as tc
                FROM experiment_metric_aggregates
                WHERE experiment_id = %s
                GROUP BY {dim}
            """, experiment_id)

            if not rows:
                continue

            # 计算整体方向
            total_cc = sum(r["cc"] for r in rows)
            total_cn = sum(r["cn"] for r in rows)
            total_tc = sum(r["tc"] for r in rows)
            total_tn = sum(r["tn"] for r in rows)

            overall_control_rate = total_cc / total_cn if total_cn > 0 else 0
            overall_treatment_rate = total_tc / total_tn if total_tn > 0 else 0
            overall_direction = 1 if overall_treatment_rate > overall_control_rate else -1

            # 检查子群方向是否矛盾
            for r in rows:
                if r["cn"] > 0 and r["tn"] > 0:
                    sub_control_rate = r["cc"] / r["cn"]
                    sub_treatment_rate = r["tc"] / r["tn"]
                    sub_direction = 1 if sub_treatment_rate > sub_control_rate else -1
                    if sub_direction != overall_direction:
                        # 子群方向矛盾，存在辛普森悖论
                        weight = (r["cn"] + r["tn"]) / (total_cn + total_tn)
                        if weight > 0.10:
                            return True
        return False

    def _save_test_result(self, experiment_id: str, result: StatisticalTestResult):
        """保存统计检验结果到数据库"""
        self.db.execute_insert("""
            INSERT INTO experiment_results
                (experiment_id, metric_name, control_mean, control_stddev, control_n,
                 treatment_mean, treatment_stddev, treatment_n,
                 effect_size, relative_change, p_value, ci_lower, ci_upper,
                 is_significant, test_method, analysis_time)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, experiment_id, result.metric_name,
             result.control_stats.mean, result.control_stats.stddev,
             result.control_stats.n,
             result.treatment_stats.mean, result.treatment_stats.stddev,
             result.treatment_stats.n,
             result.effect_size, result.relative_change, result.p_value,
             result.confidence_interval[0], result.confidence_interval[1],
             result.is_significant, result.test_method, datetime.now())

    def _save_sample_size_estimate(self, estimate: SampleSizeEstimate):
        """保存样本量估算结果"""
        key = f"sample_estimate:{estimate.experiment_id}"
        self.redis.hmset(key, {
            "current_n": estimate.current_n_per_group,
            "required_n": estimate.required_n_per_group,
            "remaining_n": estimate.remaining_n_per_group,
            "remaining_days": estimate.estimated_remaining_days,
            "current_power": estimate.current_power,
        })
        self.redis.expire(key, 3600)

    def _save_evaluation_report(self, report: EvaluationReport):
        """保存完整评估报告"""
        import json as json_lib
        report_key = f"eval_report:{report.experiment_id}"
        self.redis.setex(
            report_key, 7200,
            json_lib.dumps({
                "experiment_id": report.experiment_id,
                "decision": report.decision.decision.value,
                "confidence": report.decision.confidence_level,
                "significant_metrics": sum(
                    1 for r in report.metric_results if r.is_significant
                ),
                "total_metrics": len(report.metric_results),
                "recommended_action": report.recommended_action,
                "evaluation_time": report.evaluation_time.isoformat(),
            }, default=str)
        )
```

## 异常场景补充

### 场景：实验结果因季节性因素被误判

```
触发: 某电商的推荐算法 A/B 测试在"双11促销季"期间运行，新算法在促销流量上表现优异（转化率提升15%），但日常流量上反而不如旧算法。由于促销季流量远大于日常流量，整体数据显示新算法显著更优，团队决定全量上线。上线后促销结束，日常流量下转化率反而下降5%。

检测:
  1. 时间段异质性检测：将实验数据按周分段，计算各周的独立效果量，若各周效果量方差过大（变异系数 > 0.5），则提示存在时间段异质性
  2. 流量组成分析：对比实验期间不同流量来源（促销/自然/付费）的比例与历史同期的偏差，若偏差超过20%则告警
  3. 外部事件关联：检查实验期间是否有大型促销、节假日、竞品活动等外部事件，若存在则标记实验结论可能受季节性影响
  4. 滞后效应检测：实验结束后继续监控2周，对比实验期效果与实验后效果，若效果量衰减超过30%则判定存在季节性偏差

处理:
  1. 发现季节性偏差后，立即暂停全量上线决策
  2. 将实验数据按"促销期"和"非促销期"分层分析，分别报告效果量
  3. 基于非促销期的效果量重新评估商业价值和上线决策
  4. 若非促销期效果不显著，建议在非促销期重新运行实验验证
  5. 在评估报告中明确标注"本实验结果受季节性因素影响，全量上线风险较高"

预防:
  - 实验设计阶段：排除已知促销期，或确保实验周期包含完整的业务周期（至少包含一个完整的周一到周日循环 + 促销和非促销各至少一周）
  - 实验配置：设置 min_duration_days >= 14 天，强制覆盖完整周期
  - 自动化检测：在评估报告中增加"季节性风险评估"模块，自动对比实验期间流量组成与历史基线
  - 决策流程：要求实验结论必须附带"时间段异质性分析"，显著异质的结论需要额外审批
  - CUPED 调整：使用历史同期数据作为协变量，消除季节性对效果量估计的影响
```

### 场景：样本量不足时过早下结论

```
触发: 产品经理在第3天偷看实验数据，发现核心指标 p=0.04（未校正），即宣布"实验组胜出"并要求立即全量上线。实际此时每组只有500个样本（需要10000个），统计功效仅15%，假阳性率因偷看已从5%飙升到30%以上。真实效果量为0，p=0.04纯属随机波动。

检测:
  1. 信息分数检查：当前样本量/计划样本量 < 0.5 时，标记为"过早查看"
  2. 序贯检验边界校验：对比当前 p 值与序贯检验边界值（OBF 边界在信息分数0.2时要求 z>4.5），若原始 p 值显著但序贯 p 值不显著，说明结论为假阳性
  3. 效果量稳定性检测：对比昨天的效果量与今天的效果量，若变化幅度超过50%则提示效果量不稳定
  4. 置信区间宽度检查：若95%置信区间宽度 > 效果量的2倍，说明估计精度不足

处理:
  1. 平台层面拦截：当信息分数 < 0.5 时，看板不显示 p 值，只显示"实验进行中，当前效果方向为正向/负向"
  2. 自动化提醒：当有人在样本量不足时请求完整结果，系统发送提醒"当前统计功效仅15%，结论不可靠，建议等待至少X天"
  3. 强制序贯检验：所有实验结果必须通过序贯检验（OBF 边界）才能标记为显著，早期偷看得到的 p 值必须与边界对比
  4. 效果量波动可视化：在看板中展示"效果量随时间变化"曲线，若波动剧烈则可视化提醒效果不稳定
  5. 决策锁定：在全量上线按钮前增加确认步骤，展示当前统计功效和所需剩余天数

预防:
  - 工程防护：部署 NoPeekingGuard，限制每日查看次数（1次/天），且最短实验天数内不显示 p 值
  - 序贯检验强制：平台默认启用 OBF 序贯检验边界，早期阈值极严格（信息分数0.1时 z>6.36），数学上杜绝早期假阳性
  - 最短实验天数：min_duration_days >= 7 天（覆盖完整周期效应），不可手动缩短
  - 样本量门控：看板增加"样本量充足性"指标，不达标时高亮警告
  - 教育培训：定期向产品团队宣导"偷看导致假阳性率从5%飙升到30%以上"的数据，建立数据驱动的实验文化
  - 审批流程：全量上线需要统计功效 >= 80% 的证据，自动校验不通过则审批流拒绝
```

## 实验互斥组与分层管理完整实现

```python
class ExperimentMutualExclusionService:
    """实验互斥：互斥组管理 → 层间独立性验证 → 冲突检测"""

    def create_mutual_exclusion_group(self, group_name, experiments, reason=None):
        """创建互斥组"""
        # 1. 验证实验不已在其他互斥组中
        for exp_id in experiments:
            existing_group = self.db.query_one(
                "SELECT * FROM experiment_exclusion_groups "
                "WHERE experiment_ids LIKE %s AND status = 'active'",
                f'%"{exp_id}"%')

            if existing_group:
                return {"status": "conflict",
                        "experiment_id": exp_id,
                        "existing_group": existing_group["group_id"]}

        # 2. 创建互斥组
        group_id = str(uuid4())
        self.db.insert("experiment_exclusion_groups", {
            "group_id": group_id,
            "name": group_name,
            "experiment_ids": json.dumps(experiments),
            "reason": reason,
            "status": "active",
            "created_at": now()
        })

        # 3. 更新实验的互斥约束
        for exp_id in experiments:
            self.db.update("experiments",
                {"exclusion_group_id": group_id},
                {"id": exp_id})

        return {"group_id": group_id, "name": group_name,
                "experiments": experiments}

    def check_experiment_compatibility(self, new_experiment_id, layer):
        """检查新实验与现有实验的兼容性"""
        conflicts = []

        # 1. 同层运行中的实验
        running_in_layer = self.db.query(
            "SELECT * FROM experiments "
            "WHERE layer = %s AND status = 'running'",
            layer)

        for existing in running_in_layer:
            # 检查流量重叠
            new_exp = self.db.get_experiment(new_experiment_id)
            overlap = self._calculate_traffic_overlap(new_exp, existing)

            if overlap > 0:
                # 检查是否在同一互斥组
                if new_exp.get("exclusion_group_id") and \
                   existing.get("exclusion_group_id") == new_exp["exclusion_group_id"]:
                    conflicts.append({
                        "type": "mutual_exclusion",
                        "existing_experiment": existing["id"],
                        "overlap_pct": overlap,
                        "message": "同互斥组实验不能同时运行"
                    })
                else:
                    # 不同互斥组但流量重叠 → 需确认层内分配
                    total_traffic = new_exp["traffic_pct"] + existing["traffic_pct"]
                    if total_traffic > 100:
                        conflicts.append({
                            "type": "traffic_overflow",
                            "existing_experiment": existing["id"],
                            "total_traffic_pct": total_traffic,
                            "message": f"同层总流量 {total_traffic}% 超过 100%"
                        })

        # 2. 跨层交互检测
        other_layers = self.db.query(
            "SELECT DISTINCT layer FROM experiments WHERE status = 'running' "
            "AND layer != %s", layer)

        for other_layer in other_layers:
            layer_experiments = self.db.query(
                "SELECT * FROM experiments "
                "WHERE layer = %s AND status = 'running'",
                other_layer["layer"])

            for exp in layer_experiments:
                # 检查指标重叠（可能互相影响）
                new_metrics = set(new_experiment_id and
                    self.db.get_experiment(new_experiment_id).get("metrics", []))
                existing_metrics = set(exp.get("metrics", []))

                if new_metrics & existing_metrics:
                    conflicts.append({
                        "type": "metric_overlap",
                        "layer": other_layer["layer"],
                        "experiment_id": exp["id"],
                        "overlapping_metrics": list(new_metrics & existing_metrics),
                        "message": f"跨层指标重叠: {new_metrics & existing_metrics}"
                    })

        return {"compatible": len(conflicts) == 0, "conflicts": conflicts}

    def _calculate_traffic_overlap(self, exp_a, exp_b):
        """计算两个实验的流量重叠"""
        # 简化：同层实验流量直接重叠
        if exp_a.get("layer") == exp_b.get("layer"):
            return min(exp_a["traffic_pct"], exp_b["traffic_pct"])
        return 0

    def validate_layer_independence(self, layer_a, layer_b, metric_name):
        """验证两层实验是否独立"""
        # 获取两层实验的用户分配
        layer_a_users = self.db.query(
            "SELECT user_id, experiment_id FROM experiment_assignments "
            "WHERE experiment_id IN ("
            "  SELECT id FROM experiments WHERE layer = %s AND status = 'running')",
            layer_a)

        layer_b_users = self.db.query(
            "SELECT user_id, experiment_id FROM experiment_assignments "
            "WHERE experiment_id IN ("
            "  SELECT id FROM experiments WHERE layer = %s AND status = 'running')",
            layer_b)

        # 计算卡方检验（检查两层分配是否独立）
        from collections import Counter
        a_assignments = Counter(u["experiment_id"] for u in layer_a_users)
        b_assignments = Counter(u["experiment_id"] for u in layer_b_users)

        # 简化独立性检验
        total = sum(a_assignments.values())
        if total == 0:
            return {"independent": True, "confidence": 1.0}

        # 检查两层分配比例是否一致
        is_independent = True
        for exp_id, count in a_assignments.items():
            expected_ratio = count / total
            b_count = b_assignments.get(exp_id, 0)
            b_total = sum(b_assignments.values())
            actual_ratio = b_count / max(b_total, 1)

            if abs(expected_ratio - actual_ratio) > 0.05:
                is_independent = False
                break

        return {"independent": is_independent,
                "layer_a": layer_a, "layer_b": layer_b,
                "metric": metric_name}
```

## 异常场景补充

### 场景：互斥组配置错误导致实验冲突

```
触发：运营创建了互斥组 A 包含实验 1 和实验 2 → 但实验 1 和实验 3 也有互斥关系 → 遗漏 → 冲突
检测：
  1. 互斥组外的实验出现流量冲突 → 配置遗漏
  2. 实验结果异常波动 → 可能是互斥实验干扰
处理：
  1. 互斥关系传递闭包计算（A 与 B 互斥，B 与 C 互斥 → A 与 C 也互斥）
  2. 冲突检测在实验启动前自动运行
  3. 发现冲突 → 暂停新实验
预防：传递闭包 + 启动前检测 + 自动暂停
```

### 场景：分层实验指标互相污染

```
触发：层 1 改变按钮颜色 → 层 2 改变按钮位置 → 两层同时影响点击率 → 无法归因
检测：
  1. 跨层实验共依赖同一指标 → 互相影响
  2. 层间独立性检验失败 → 不独立
处理：
  1. 将互相影响的实验放入同一层（互斥）
  2. 使用不同指标分别度量
  3. 设计交互实验分析
预防：同层互斥 + 指标分离 + 交互分析
```
