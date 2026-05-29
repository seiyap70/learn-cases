# C15: 网约车的派单系统

## 业务场景

某网约车平台，需要实现实时派单系统：乘客发单 → 系统匹配最优司机 → 司机接单 → 服务执行。派单系统是网约车平台的核心——派单质量直接决定了乘客等待时间、司机收入和平台效率。

**已知数据：**
- 同时在线司机：30 万（高峰期），10 万（平峰期）
- 同时等待乘客：5 万（高峰期）
- 派单延迟要求：< 3 秒（从乘客发单到司机收到派单通知）
- 司机位置更新频率：每 5 秒上报一次
- 服务区域：全国 400+ 城市

**派单质量指标：**
- 乘客平均等待时间：< 5 分钟
- 司机接单率：> 85%（派单成功率）
- 司机拒单率：< 10%
- 空驶率：< 30%（司机无客行驶比例）

**为什么这比"找最近司机"复杂得多？**

表面上看，派单就是"找最近的空闲司机"。但实际考虑：

1. **最近的不一定最优** —— 距离 500 米的司机正在反方向行驶，掉头需要 3 分钟；距离 1.5 公里的司机正在同方向行驶，2 分钟就能到
2. **公平性问题** —— 如果总是派给最近的司机，远离热点的司机永远接不到单
3. **全局最优 vs 局部最优** —— 把最近的司机派给 A 乘客，可能意味着 B 乘客（更紧急）要等更久
4. **供需不平衡** —— 高峰期需求远大于供给，如何分配有限的司机资源

## 核心挑战

### 挑战 1：实时空间匹配

30 万在线司机，每 5 秒更新一次位置。乘客发单时，需要在毫秒级内找到 3km 范围内的空闲司机，并计算实际到达时间（考虑道路网络，不是直线距离）。

### 挑战 2：多目标优化

派单不是单一目标的优化问题，而是多目标权衡：
- 最小化乘客等待时间
- 最大化司机收入（减少空驶）
- 保证公平性（司机接单机会均衡）
- 提高拼车效率（如果支持拼车）

这些目标之间有根本性矛盾：公平性往往意味着牺牲效率。

### 挑战 3：供需严重不平衡

早晚高峰时，需求是供给的 3-5 倍。不可能让所有乘客都打到车，必须做资源分配——哪些乘客优先？如何定价调节需求？

### 挑战 4：信息不对称

系统知道司机的实时位置和行驶方向，但不知道司机的主观意愿——司机可能不想接长途单、不想去某个区域、不想接低评分乘客。

## 设计约束

- 司机位置每 5 秒更新一次（存在 5 秒的位置延迟）
- 派单决策需要考虑：距离、方向夹角、司机评分、等待时长、历史接单率
- 拒单率需控制 < 10%（派单准确性指标）
- 3 秒内完成匹配+通知（包括网络传输时间）

## 请先独立思考（限时 45 分钟）

1. 设计一个评分函数，给候选司机打分。明确每个因子的权重和计算方式，然后分析：权重如何影响"效率 vs 公平"的权衡？
2. 派单是"抢单"还是"系统派单"？从司机体验、匹配质量、公平性三个维度对比。
3. 如果某区域有 100 个等待乘客但只有 20 个空闲司机，应该怎么分配？先到先得？出价最高？距离最近？
4. 设计一个动态加价算法，在供需不平衡时自动调节价格。注意：加价不能无上限（舆论风险），也不能不起作用（调节失败）。

---

## 设计解析

### 派单策略选择：为什么选系统派单而非抢单

| 维度 | 抢单 | 系统派单 |
|------|------|---------|
| 匹配质量 | 差——最近的司机不一定手最快，可能 500 米外的司机网速慢抢不到 | 好——算法选最优 |
| 公平性 | 差——好单被手快的人抢，差单无人接 | 可控——算法可加入公平因子 |
| 司机体验 | 焦虑——需要不断盯着手机抢单，可能分心驾驶 | 安心——系统分配，可以专注驾驶 |
| 乘客体验 | 不稳定——等单时间取决于附近有没有"手快"的司机 | 可预期——等单时间由算法优化 |
| 系统复杂度 | 低——只需广播订单 | 高——需要维护司机状态和评分系统 |

**结论：系统派单。** 几乎所有主流网约车平台都已从抢单转向派单。

### 整体架构

```
乘客发单 → 派单服务
              │
              ├─ 1. 空间查询：Redis GEO 找 3km 内空闲司机
              │
              ├─ 2. 方向过滤：排除方向完全相反的司机
              │
              ├─ 3. 状态过滤：排除不合规司机（拒单率过高/休息中等）
              │
              ├─ 4. 综合打分：多因子评分
              │
              ├─ 5. 选择 Top 1 司机
              │
              ├─ 6. 发送派单通知
              │
              └─ 7. 司机接单/拒单 → 接单成功 or 重试 Top 2
```

### 第一步：空间查询

**Redis GEO 存储司机位置：**

```
司机位置上报（每5秒）：
  GEOADD geo:drivers:bj 116.3971 39.9165 driver_12345

3km 范围查询：
  GEORADIUS geo:drivers:bj 116.3971 39.9165 3 km WITHDIST WITHCOORD COUNT 50
```

**按城市分片：** `geo:drivers:{cityId}`，避免跨城市搜索。

**查询耗时：** < 5ms（Redis GEO 基于 Sorted Set + GeoHash，O(log(N)+M)）

**空驶司机标记：**

```
司机状态存储：
  HSET driver:status:12345 state "free"       # free/serving/offline
  HSET driver:status:12345 last_heartbeat 1715000000
```

查询时只返回 `state=free` 的司机（应用层过滤，Redis GEO 不支持条件过滤）。

### 第二步：方向过滤

**问题：** 距离最近的司机可能正在反方向行驶，实际到达时间可能比远处的同向司机更长。

**方向计算：**

```python
def direction_score(driver_bearing, pickup_to_dest_bearing):
    """
    计算司机行驶方向与乘客出行方向的一致性
    
    driver_bearing: 司机当前行驶方向（0-360度，正北为0）
    pickup_to_dest_bearing: 接客点到目的地的方向
    
    返回 0-1 的分数，1 = 完全同向
    """
    diff = abs(driver_bearing - pickup_to_dest_bearing) % 360
    if diff > 180:
        diff = 360 - diff
    
    return 1.0 - (diff / 180.0)

# 示例：
# 司机向北行驶 (0°)，乘客要去北方 (10°) → diff=10°, score=0.94 (同向)
# 司机向北行驶 (0°)，乘客要去南方 (180°) → diff=180°, score=0.00 (反向)
```

**过滤策略：** 方向分数 < 0.3（夹角 > 126°）的司机降低优先级，而非直接排除——因为如果附近没有同向司机，反向司机仍然是选择。

### 第三步：综合打分

**评分函数设计：**

```
score = w1 × distance_score
      + w2 × direction_score
      + w3 × driver_rating_score
      + w4 × idle_time_score
      + w5 × fairness_score
```

**各因子的计算：**

```python
class DriverScorer:
    # 默认权重（可按城市/时段动态调整）
    WEIGHTS = {
        "distance": 0.35,
        "direction": 0.20,
        "rating": 0.15,
        "idle_time": 0.15,
        "fairness": 0.15,
    }

    def score(self, driver, order):
        d = driver
        o = order

        distance_score = self._distance_score(d.distance_to_pickup)
        direction_score = self._direction_score(d.bearing, o.bearing)
        rating_score = d.rating / 5.0
        idle_score = self._idle_score(d.idle_seconds)
        fairness_score = self._fairness_score(d)

        weights = self.get_weights(o.city_id, o.time_of_day)

        return (weights["distance"] * distance_score +
                weights["direction"] * direction_score +
                weights["rating"] * rating_score +
                weights["idle_time"] * idle_score +
                weights["fairness"] * fairness_score)

    def _distance_score(self, distance_m):
        """距离评分：越近分越高，3km 以上为0"""
        max_distance = 3000  # 3km
        if distance_m > max_distance:
            return 0
        return 1.0 - (distance_m / max_distance)

    def _direction_score(self, driver_bearing, order_bearing):
        """方向评分"""
        diff = abs(driver_bearing - order_bearing) % 360
        if diff > 180:
            diff = 360 - diff
        return 1.0 - (diff / 180.0)

    def _idle_score(self, idle_seconds):
        """空闲等待评分：空闲越久优先级越高（10分钟封顶）"""
        max_idle = 600  # 10分钟
        return min(idle_seconds / max_idle, 1.0)

    def _fairness_score(self, driver):
        """公平性评分：近期接单少的司机优先级更高"""
        recent_trips = driver.trips_last_4_hours
        target_trips = driver.online_hours_last_4 * 2  # 目标：每小时2单

        if recent_trips >= target_trips:
            return 0  # 已达目标，不再优先
        return (target_trips - recent_trips) / target_trips
```

**权重的动态调整：**

```python
class WeightManager:
    """根据时段和供需比动态调整权重"""

    PROFILES = {
        "normal": {  # 平峰期：强调公平性
            "distance": 0.30, "direction": 0.15,
            "rating": 0.15, "idle_time": 0.20, "fairness": 0.20,
        },
        "peak": {  # 高峰期：强调效率（快速匹配）
            "distance": 0.45, "direction": 0.25,
            "rating": 0.10, "idle_time": 0.10, "fairness": 0.10,
        },
        "late_night": {  # 深夜：强调安全（评分高优先）
            "distance": 0.30, "direction": 0.15,
            "rating": 0.30, "idle_time": 0.10, "fairness": 0.15,
        },
    }

    def get_weights(self, city_id, time_of_day, supply_demand_ratio):
        # 根据供需比选择配置
        if supply_demand_ratio < 0.5:  # 供不应求
            return self.PROFILES["peak"]
        elif time_of_day.hour < 6:  # 深夜
            return self.PROFILES["late_night"]
        else:
            return self.PROFILES["normal"]
```

**为什么高峰期加重距离权重？**
- 高峰期供需严重不平衡，快速匹配比精准匹配更重要
- 等待时间每多 1 分钟，乘客取消率增加 15%
- 把最近的司机派出去，比花时间找"最合适"的司机更高效

### 第四步：派单执行

```python
class DispatchService:
    def dispatch(self, order):
        """执行派单"""
        # 1. 空间查询
        nearby_drivers = self.geo_search(order.pickup_location, radius_km=3)

        if not nearby_drivers:
            return self.expand_search(order)  # 扩大搜索范围

        # 2. 过滤 + 打分
        candidates = []
        for driver in nearby_drivers:
            if not self.is_eligible(driver, order):
                continue
            score = self.scorer.score(driver, order)
            candidates.append((driver, score))

        if not candidates:
            return self.expand_search(order)

        # 3. 按 score 降序排序
        candidates.sort(key=lambda x: -x[1])

        # 4. 向 Top 1 司机发送派单
        top_driver = candidates[0][0]
        return self.send_dispatch(order, top_driver, candidates[1:])

    def send_dispatch(self, order, driver, remaining_candidates):
        """发送派单通知，等待司机响应"""
        # 发送推送通知
        self.push_service.send(driver.id, {
            "type": "dispatch",
            "order_id": order.id,
            "pickup_location": order.pickup_location,
            "pickup_distance": driver.distance_to_pickup,
            "estimated_fare": order.estimated_fare,
            "timeout_seconds": 10
        })

        # 设置 10 秒超时
        self.set_timeout(order.id, 10, callback=lambda: self.on_driver_timeout(
            order, remaining_candidates
        ))

    def on_driver_timeout(self, order, remaining_candidates):
        """司机超时未响应，派给下一个"""
        if remaining_candidates:
            next_driver = remaining_candidates[0][0]
            self.send_dispatch(order, next_driver, remaining_candidates[1:])
        else:
            # 无更多候选，扩大搜索
            self.expand_search(order)

    def on_driver_accept(self, order_id, driver_id):
        """司机接单"""
        self.cancel_timeout(order_id)
        self.match_service.confirm_match(order_id, driver_id)
        self.notify_passenger(order_id, "司机已接单，正在赶来")

    def on_driver_reject(self, order_id, driver_id, reason):
        """司机拒单"""
        # 记录拒单（影响司机评分和派单优先级）
        self.record_rejection(driver_id, reason)

        # 派给下一个候选
        remaining = self.get_remaining_candidates(order_id)
        if remaining:
            self.send_dispatch(order_id, remaining[0], remaining[1:])
        else:
            self.expand_search(order_id)
```

**为什么串行派单而非并行？**

并行派单（同时发给 3 个司机）的问题：
- 3 个司机同时接单 → 只能 1 个成功，其他 2 个白跑
- 司机看到"订单已被接走"会降低对平台的信任
- 串行派单更公平（综合分最高的优先）

但串行的延迟更高：如果 Top 1 司机 10 秒超时 + Top 2 司机 10 秒超时 = 20 秒。

**折中方案：双发策略**

```python
# 同时发给 Top 1 和 Top 2，先接单者获得订单
def dual_dispatch(self, order, candidates):
    if len(candidates) >= 2:
        driver1, driver2 = candidates[0][0], candidates[1][0]
        self.send_dispatch(order, driver1)
        self.send_dispatch(order, driver2)

        # 第一个接单的获得订单
        # 第二个收到"订单已被接走"
    else:
        self.send_dispatch(order, candidates[0][0])
```

**双发的权衡：** 减少 50% 的等待时间，但增加了 10% 的"白跑"概率。

### 公平性保障：详细机制

**问题：** 如果纯按综合分排序，某些司机总是得到低分——
- 偏远地区的司机：distance_score 总是低
- 新注册的司机：rating_score 初始值低
- 高峰期才上线的司机：idle_time_score 低

**解决方案：公平性配额 + 接单保障**

```python
class FairnessTracker:
    """追踪和保障司机的接单公平性"""

    def get_fairness_boost(self, driver):
        """计算公平性加成"""
        boost = 1.0  # 默认无加成

        # 1. 饥饿保护：在线超过 30 分钟未接单 → 加成
        if driver.minutes_since_last_trip > 30:
            boost *= 1.5  # 综合分乘以 1.5

        # 2. 新司机保护：注册 < 7 天 → 加成
        if driver.days_since_registration < 7:
            boost *= 1.3

        # 3. 偏远区域补偿：所在区域司机少 → 加成
        driver_density = self.get_driver_density(driver.location)
        if driver_density < 5:  # 3km 内 < 5 个司机
            boost *= 1.2

        return boost
```

**应用方式：** `final_score = base_score * fairness_boost`

公平性加成不是单独的评分因子，而是对综合分的乘数——这保证了即使基础分低，饥渴司机也能获得派单机会。

### 动态加价：供需平衡的价格调节

**加价算法：**

```python
class SurgePricing:
    def calculate_multiplier(self, region, current_time):
        """
        计算动态加价倍数
        返回 >= 1.0 的倍数
        """
        # 1. 计算供需比
        demand = self.get_waiting_passengers(region)  # 等待乘客数
        supply = self.get_free_drivers(region)          # 空闲司机数
        ratio = demand / max(supply, 1)                 # 供需比

        # 2. 基础加价倍数
        if ratio < 1.0:
            return 1.0  # 供大于求，不加价

        # 3. 加价计算（弹性系数控制幅度）
        elasticity = 0.5
        base_multiplier = ratio ** elasticity

        # 示例：
        # ratio=1.5 → multiplier = 1.5^0.5 = 1.22 (加价 22%)
        # ratio=3.0 → multiplier = 3.0^0.5 = 1.73 (加价 73%)
        # ratio=5.0 → multiplier = 5.0^0.5 = 2.24 (加价 124%)

        # 4. 上限保护（防止舆论危机）
        MAX_MULTIPLIER = 2.5  # 最高 2.5 倍
        multiplier = min(base_multiplier, MAX_MULTIPLIER)

        # 5. 雨雪天气加成
        weather = self.get_weather(region)
        if weather == "heavy_rain":
            multiplier *= 1.3
        elif weather == "snow":
            multiplier *= 1.5

        # 再次上限保护
        multiplier = min(multiplier, MAX_MULTIPLIER)

        return round(multiplier, 1)

    def get_surge_display(self, region):
        """用户看到的加价信息"""
        multiplier = self.calculate_multiplier(region)

        if multiplier <= 1.0:
            return "正常价格"
        elif multiplier <= 1.5:
            return f"高峰期加价 {multiplier}x"
        else:
            return f"供需紧张，加价 {multiplier}x（建议稍后再试）"
```

**加价的双面效果：**
- 正面：激励更多司机上线（高收入吸引力），减少无效需求（价格敏感的乘客选择公交）
- 负面：舆论风险（"趁火打劫"）、用户体验差、可能违反某些城市的监管要求

**加价的替代方案：拼车推荐**

```
供需紧张时：
  1. 先推荐拼车（1 车服务 2-3 人）→ 供给效率提升 2-3 倍
  2. 拼车价格 = 原价 × 0.6 → 比加价更便宜
  3. 拼车等待时间 = 正常等车 + 绕路 5-10 分钟
```

### 全链路延迟分析

```
乘客点击"叫车"                    0ms
  → 空间查询(GEORADIUS)            5ms
  → 过滤 + 打分(50个候选)          10ms
  → 选择 Top 1                     1ms
  → 推送通知(FCM/APNs)             100ms
  → 司机手机收到通知               300ms (网络延迟)
  → 司机看到并点击接单             2000ms (人为响应)
  → 接单确认推送                   100ms
  → 乘客收到"司机已接单"           200ms
  ────────────────────────────────
  总计                         ~2716ms (< 3s 目标 ✓)
```

**关键发现：** 机器决策部分仅 16ms，超过 95% 的延迟来自网络传输和人为响应。优化算法本身的空间不大，但可以优化推送通道（使用厂商通道而非 FCM）和司机端 UI（大按钮、语音播报）。

## 常见陷阱（深度分析）

### 陷阱 1：只按距离派单

**具体后果：** 司机在高速公路上，距离乘客 500 米但无法掉头（最近出口在 2km 外）。实际到达时间 15 分钟，而 1.5km 外的同向司机只需 5 分钟。

**方向过滤的必要性：** 计算行驶方向与乘客出行方向的夹角，反向司机降低优先级。

### 陷阱 2：并行多派导致冲突

**具体场景：** 同时派给 3 个司机，2 个同时接单。

处理方案：
1. 第一个接单的获得订单（用分布式锁保证原子性）
2. 第二个收到"订单已被接走"通知
3. 司机的信任度下降——"明明我接了单为什么说被接走了？"

**更好的方案：** 串行派单 + 短超时（5 秒而非 10 秒），或双发策略（最多同时派 2 个）。

### 陷阱 3：加价无上限

**Uber 的真实教训：** 2014 年悉尼人质事件期间，Uber 的算法自动将加价提高到 4 倍以上，引发公众强烈抗议。最终 Uber 退还了加价费用并设置了加价上限。

**教训：** 加价必须有上限，且在紧急事件时应手动暂停加价。

### 陷阱 4：忽略拒单率

**问题：** 如果司机频繁拒单却不受惩罚，派单效率会急剧下降。

```
某司机拒单率 50%：
  派给他 10 次 → 拒 5 次 → 5 次超时(10秒) → 50 秒浪费
  期间 5 个乘客的等待时间增加 50 秒
```

**拒单惩罚机制：**
- 拒单率 > 30%：降低派单优先级（fairness_score 乘 0.5）
- 拒单率 > 50%：暂停派单 30 分钟
- 连续拒单 3 次：暂停派单 1 小时

但需要区分"恶意拒单"和"合理拒单"：
- 恶意拒单：只接长途单、只接高价单
- 合理拒单：车辆故障、身体不适、接孩子

**合理拒单的识别：** 司机可以在 App 中选择拒单原因，标记为"合理"的不计入拒单率。

## 延伸思考

- **智能调度**：在乘客发单前，系统预测需求热点，提前将司机引导到需求密集区域。这需要需求预测模型（基于历史数据+天气+事件）和司机激励（引导奖励金）。
- **自动驾驶**：派单系统无需考虑司机意愿和拒单，但需要考虑车辆电量/续航、车队管理、远程监控等新问题。
- **顺路单**：司机设置目的地，系统只派同方向的订单。实现方式：在评分函数中大幅提升 direction_score 的权重（> 0.5），同时放宽距离限制（5km 而非 3km）。