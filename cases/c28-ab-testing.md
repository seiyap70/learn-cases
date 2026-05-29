# C28: A/B测试平台的数据准确性

## 业务场景

某互联网公司的增长团队需要 A/B 测试验证产品决策：新按钮颜色是否提升点击率？新推荐算法是否提升停留时长？错误的 A/B 结论可能导致公司做出错误决策，浪费数月开发资源。

**已知数据：**
- 日均实验数：50+
- 每个实验用户量：1 万-100 万
- 实验时长：1-4 周
- 指标数：每个实验约 10 个
- 统计显著性要求：p-value < 0.05

**为什么看似简单的"分两组、看谁好"实际上很难？**

- **样本量不足** → 假阳性（噪音被误认为效果）
- **偷看结果** → 实验未结束就停止 → 假阳性率从 5% 飙升到 30%+
- **辛普森悖论** → 整体看 A 好，分设备看 B 好 → 结论取决于分组
- **多指标问题** → 测试 10 个指标，至少 1 个假阳性概率 = 1 - 0.95^10 ≈ 40%

## 核心挑战

### 挑战 1：实验分组的正确性

用户必须被稳定分组——今天看到新版本，明天不能看到旧版本。但清除 Cookie 或换设备会导致分组变化。

### 挑战 2：偷看与早期停止

实验还没结束就看数据，发现"显著"就停止。这会让假阳性率从 5% 飙升到 30%+。

### 挑战 3：多指标多重比较

测试 10 个指标，每个 p < 0.05 → 至少 1 个假阳性概率 ≈ 40%。如果恰好这 1 个被当成了结论 → 错误决策。

### 挑战 4：辛普森悖论

整体数据看 A 好，但分设备看 B 好 → 哪个结论正确？

## 设计约束

- 分组稳定性：同一用户始终在同一组
- 偷看保护：早期停止需要更严格的阈值
- 统计显著性：p < 0.05，多重比较校正
- 样本量自动推荐

## 请先独立思考（限时 30 分钟）

1. 如何保证分组稳定性？用什么哈希算法？为什么不能只用 user_id 哈希？
2. 如何防止偷看导致的假阳性？序贯检验是什么？
3. 多重比较如何校正？Bonferroni vs FDR？
4. 样本量如何计算？MDE（最小可检测效果）如何确定？

---

## 设计解析

### 分组：确定性哈希

```python
class ExperimentAssigner:
    def assign(self, user_id, experiment_id):
        """确定性分组：同一用户永远在同一组"""
        # MurmurHash3(user_id + experiment_id) % 1000
        hash_val = mmh3.hash(f"{user_id}:{experiment_id}") % 1000
        
        # 实验配置：对照组 50%，实验组 50%
        if hash_val < 500:
            return "control"
        else:
            return "treatment"
```

**为什么用 user_id + experiment_id？**

如果只用 user_id 哈希 → 用户 A 在所有实验中都在实验组 → 实验间相关性 → 偏差。

用 user_id + experiment_id → 同一用户在不同实验中的分组独立 → 无相关性。

**换设备怎么办？**

用户清 Cookie 或换设备 → 前端分组丢失。解决方案：
- 服务端存储分组结果（user_id → experiment_id → group）
- 用户登录后从服务端获取分组（而非前端本地计算）

```python
class ServerSideAssignment:
    def get_assignment(self, user_id, experiment_id):
        """服务端分组（持久化到数据库）"""
        # 先查缓存
        cached = self.redis.get(f"ab:{user_id}:{experiment_id}")
        if cached:
            return cached
        
        # 查数据库
        record = self.db.query_one("""
            SELECT group_name FROM experiment_assignments
            WHERE user_id = %s AND experiment_id = %s
        """, user_id, experiment_id)
        
        if record:
            self.redis.setex(f"ab:{user_id}:{experiment_id}", 3600, record.group_name)
            return record.group_name
        
        # 新用户：计算分组并持久化
        group = self.assign(user_id, experiment_id)
        self.db.execute("""
            INSERT INTO experiment_assignments (user_id, experiment_id, group_name)
            VALUES (%s, %s, %s)
        """, user_id, experiment_id, group)
        
        return group
```

### 样本量计算

```python
class SampleSizeCalculator:
    def calculate(self, baseline_rate, mde, alpha=0.05, power=0.8):
        """
        baseline_rate: 基线转化率（如当前点击率 5%）
        mde: 最小可检测效果（如期望提升 10%，即从5%到5.5%）
        alpha: 显著性水平
        power: 统计功效（1 - beta，真阳性率）
        """
        p1 = baseline_rate
        p2 = baseline_rate * (1 + mde)
        
        z_alpha = 1.96   # alpha=0.05 双侧
        z_beta = 0.84    # power=0.8
        
        # 每组样本量
        n = ((z_alpha * math.sqrt(2 * p1 * (1-p1)) + 
              z_beta * math.sqrt(p1*(1-p1) + p2*(1-p2))) ** 2) / (p2-p1) ** 2
        
        return math.ceil(n)

# 示例计算
calc = SampleSizeCalculator()

# 基线 5%，期望提升 10%（mde=0.1）→ 每组约 31000 人
# 基线 5%，期望提升 50%（mde=0.5）→ 每组约 1500 人
# 基线 50%，期望提升 5%（mde=0.05）→ 每组约 63000 人

# MDE 越小，需要样本量越大！
# 这就是为什么"微小优化"需要很长时间的实验
```

**自动推荐实验时长：**

```python
    def recommend_duration(self, baseline_rate, mde, daily_traffic):
        n_per_group = self.calculate(baseline_rate, mde)
        total_n = n_per_group * 2  # 两组
        days = math.ceil(total_n / daily_traffic)
        return days
```

### 防止偷看：序贯检验（Sequential Testing）

**问题：** 传统 t 检验假设"只在实验结束时看一次结果"。如果中途偷看 N 次，每次 p < 0.05 就停止 → 假阳性率远超 5%。

**O'Brien-Fleming 边界：早期需要更严格的阈值**

```python
class SequentialTester:
    def check_significant(self, control, treatment, current_n, total_n):
        """
        允许随时查看，但早期需要更严格的阈值
        """
        info_fraction = current_n / total_n  # 信息比
        
        # OBF 边界：早期 z 阈值极高，后期回归 1.96
        z_threshold = self.obf_boundary(info_fraction)
        
        z = self.compute_z_statistic(control, treatment)
        
        return abs(z) > z_threshold

    def obf_boundary(self, info_fraction):
        """O'Brien-Fleming 边界值"""
        if info_fraction < 0.25:
            return 8.0    # 极严格：几乎不可能早期显著
        elif info_fraction < 0.50:
            return 4.0    # 很严格
        elif info_fraction < 0.75:
            return 2.5    # 较严格
        else:
            return 1.96   # 最后阶段回归标准阈值
```

**效果：** 即使天天偷看，假阳性率仍控制在 5% 以内。

### 多重比较校正

```python
class MultipleComparisonCorrector:
    def correct(self, p_values, method="bonferroni"):
        m = len(p_values)
        
        if method == "bonferroni":
            # Bonferroni：最保守，阈值 = alpha / m
            # 10 个指标 → 每个阈值 = 0.05 / 10 = 0.005
            corrected_alpha = 0.05 / m
            return [p < corrected_alpha for p in p_values]
        
        elif method == "fdr":
            # FDR（Benjamini-Hochberg）：控制假发现率
            # 更宽松，适合探索性分析
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

**选择建议：**

| 场景 | 方法 | 原因 |
|------|------|------|
| 核心指标 1-2 个 | 不需要校正 | 指标少，假阳性概率低 |
| 指标 3-10 个 | Bonferroni | 保守但可靠，决策风险低 |
| 指标 10+ 个 | FDR | Bonferroni 太保守，FDR 更实用 |

### 辛普森悖论的识别

```python
class SimpsonParadoxDetector:
    def detect(self, experiment_id):
        """检测辛普森悖论"""
        overall = self.get_overall_result(experiment_id)
        by_device = self.get_stratified_result(experiment_id, "device")
        by_country = self.get_stratified_result(experiment_id, "country")
        
        paradox_found = False
        
        # 检查：整体结论 vs 分层结论是否一致
        for strat_name, strat_results in [("device", by_device), ("country", by_country)]:
            for stratum, result in strat_results.items():
                if result.direction != overall.direction:
                    paradox_found = True
                    self.alert(f"辛普森悖论：整体{overall.direction}，"
                             f"但{strat_name}={stratum}中{result.direction}")
        
        return paradox_found

# 示例：
# 整体：A 的转化率 5% > B 的转化率 4% → A 更好
# 手机端：A 3% > B 2% → A 更好
# 桌面端：A 10% > B 8% → A 更好
# → 一致，无悖论

# 另一示例：
# 整体：A 5% > B 4% → A 更好
# 手机端：B 3% > A 2% → B 更好
# 桌面端：B 10% > A 8% → B 更好
# → 悖论！整体和分层结论矛盾
# 原因：A 的用户中手机端占比高（转化率天然低）
```

## 常见陷阱（深度分析）

### 陷阱 1：偷看后提前停止

**具体危害：** 假阳性率从 5% 飙升到 30%+。10 个实验中可能有 3 个假阳性 → 错误决策。

**解决方案：** 序贯检验（OBF 边界），或严格等到预设实验天数结束。

### 陷阱 2：样本量不足就下结论

**场景：** 基线 5%，MDE 10%，需要每组 31000 人。但只跑了 3 天（每组 3000 人）就下结论"无显著差异" → 可能是统计功效不足，而非真的无差异。

**解决方案：** 实验前计算样本量，未达到前不下"无差异"结论。显示"实验进行中，当前统计功效 35%，预计还需 N 天"。

### 陷阱 3：辛普森悖论

**场景：** A/B 测试新推荐算法 → 整体看 A 好，但手机端 B 好。如果 80% 用户用手机 → 应该选 B，而非 A。

**解决方案：** 分析时按设备/地区分层，或用 CUPED 方法消除协变量影响。

### 陷阱 4：不校正多重比较

10 个指标 × p < 0.05 → 至少 1 个假阳性概率 40%。如果恰好这 1 个被当成了结论 → 错误决策 → 浪费数周开发资源。

**解决方案：** Bonferroni 或 FDR 校正。

## 延伸思考

- **交错实验**：同一用户同时看到 A 和 B 的结果（如搜索结果混合 A/B），更精确但更复杂。
- **多臂老虎机**：动态调整流量分配，表现好的组获得更多流量，减少实验成本。
- **CUPED**：利用实验前的基线数据降低方差，减少所需样本量 30-50%。