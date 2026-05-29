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

### 防止偷看：序贯检验

**O'Brien-Fleming vs Pocock 边界对比：**

| 检验方法 | 早期阈值 | 后期阈值 | 特点 |
|---------|---------|---------|------|
| OBF | 极严格（8.0） | 标准（1.96） | 早期几乎不可能停止，后期回归正常 |
| Pocock | 一致严格（2.36） | 一致严格（2.36） | 每次查看阈值相同，但比 OBF 宽松 |
| 固定样本 | 不允许查看 | 1.96 | 只能看一次，否则假阳性飙升 |

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
```

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
```

**Bonferroni vs FDR 对比：**

| 维度 | Bonferroni | FDR (BH) |
|------|-----------|----------|
| 保守程度 | 极保守 | 适中 |
| 10个指标时每个阈值 | 0.005 | 动态（约 0.01-0.05） |
| 假阳性控制 | 控制 family-wise error | 控制 false discovery rate |
| 适用场景 | 核心指标少（3-10个） | 指标多（10+个） |
| 统计功效 | 低（容易漏掉真效果） | 高（更容易发现真效果） |

### 辛普森悖论检测

```python
class SimpsonParadoxDetector:
    def detect(self, experiment_id):
        """检测辛普森悖论"""
        overall = self.get_overall_result(experiment_id)
        stratifications = {
            "device": self.get_stratified_result(experiment_id, "device"),
            "country": self.get_stratified_result(experiment_id, "country"),
            "user_tier": self.get_stratified_result(experiment_id, "user_tier"),
        }
        
        paradoxes = []
        for strat_name, strat_results in stratifications.items():
            for stratum, result in strat_results.items():
                if result.direction != overall.direction:
                    paradoxes.append({
                        "stratification": strat_name,
                        "stratum": stratum,
                        "overall_direction": overall.direction,
                        "stratum_direction": result.direction,
                        "sample_ratio": result.n / overall.n
                    })
        
        if paradoxes:
            self.alert(f"辛普森悖论: {len(paradoxes)} 个分层与整体结论矛盾")
        
        return paradoxes
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