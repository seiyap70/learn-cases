# C27: 新能源车充电桩的智能调度

## 业务场景

某新能源车充电运营平台，管理 1 万个充电桩，分布在 500 个充电站。充电桩是重资产——利用率直接决定盈利能力。但充电桩不是"有空位就能用"——它受电网功率约束。

**已知数据：**
- 充电桩数：1 万个（快充 60kW 30%，慢充 7kW 70%）
- 日均充电次数：5 万次
- 峰值同时充电：8000 桩
- 充电时长：快充 30-60 分钟，慢充 4-8 小时
- 单次充电费用：30-80 元
- 电网约束：单个充电站最大功率 2MW

**为什么是难题？**

一个站有 20 个 60kW 快充桩，总功率需求 1.2MW，但站点最大功率只有 2MW。最多同时 30 个桩快充（1.8MW），超过则跳闸。如果 15 个桩已经在快充，第 16 个用户来了怎么办？——降功率充电（60kW → 30kW），还是排队等候？

## 核心挑战

### 挑战 1：功率约束下的充电调度

站内总功率有上限，不能所有桩同时满功率充电。需要动态分配功率。

### 挑战 2：预约与实际使用的偏差

用户预约了 10:00-11:00 的快充桩，但 10:30 才到 → 桩空闲 30 分钟（利用率损失）。充完不挪车 → 后面预约的用户无法使用。

### 挑战 3：电价波动下的充电优化

分时电价：谷时 ¥0.3/度，峰时 ¥1.2/度。如何引导用户谷时充电，提升谷时利用率？

## 设计约束

- 不超过站点功率上限（跳闸 = 全站断电 = 严重事故）
- 预约宽限期 15 分钟
- 用户等待排队时间 < 30 分钟

## 请先独立思考（限时 30 分钟）

1. 功率调度算法：如何在功率预算内分配各桩的充电功率？降功率 vs 排队？
2. 预约系统：如何处理迟到和不挪车？宽限期如何设计？
3. 电价引导：如何设计激励机制让用户选择谷时充电？

---

## 设计解析

### 功率调度：动态分配 + 降功率优先

```python
class PowerScheduler:
    """站点级功率调度器"""

    def __init__(self, station_id, max_power_kw):
        self.station_id = station_id
        self.max_power_kw = max_power_kw
        self.charging_piles = {}  # pile_id → {power_kw, priority, start_time}
        self.wait_queue = []      # 排队队列

    def request_charge(self, pile_id, requested_power_kw, priority="normal"):
        """请求充电"""
        current_total = sum(p["power_kw"] for p in self.charging_piles.values())
        
        # 策略1：功率充足 → 直接满功率充电
        if current_total + requested_power_kw <= self.max_power_kw:
            self.charging_piles[pile_id] = {
                "power_kw": requested_power_kw,
                "priority": priority,
                "start_time": now()
            }
            return {"status": "approved", "power_kw": requested_power_kw,
                    "estimated_duration": self.estimate_duration(requested_power_kw)}
        
        # 策略2：功率不足但有剩余 → 降功率充电
        available = self.max_power_kw - current_total
        if available > 7:  # 至少 7kW（慢充功率）
            actual_power = min(requested_power_kw, available)
            self.charging_piles[pile_id] = {
                "power_kw": actual_power,
                "priority": priority,
                "start_time": now()
            }
            # 通知用户：充电速度降低
            duration = self.estimate_duration(actual_power)
            return {"status": "derated", "power_kw": actual_power,
                    "estimated_duration": duration,
                    "note": f"当前功率{actual_power}kW，预计充电{duration}分钟"}
        
        # 策略3：完全无可用功率 → 排队
        self.wait_queue.append({
            "pile_id": pile_id,
            "requested_power": requested_power_kw,
            "priority": priority,
            "queued_at": now()
        })
        position = len(self.wait_queue)
        estimated_wait = self.estimate_wait_time()
        return {"status": "queued", "position": position,
                "estimated_wait": estimated_wait}

    def estimate_duration(self, power_kw):
        """根据充电功率估算充电时间"""
        # 快充 60kW → 约 40 分钟充 80%
        # 降功率 30kW → 约 80 分钟
        # 慢充 7kW → 约 6 小时
        if power_kw >= 60:
            return 40
        elif power_kw >= 30:
            return 80
        else:
            return 360  # 6小时

    def estimate_wait_time(self):
        """估算排队等待时间"""
        if not self.wait_queue:
            return 0
        
        # 基于当前在充车辆的预计完成时间
        finish_times = [p["start_time"] + timedelta(minutes=self.estimate_duration(p["power_kw"]))
                       for p in self.charging_piles.values()]
        
        earliest_finish = min(finish_times)
        wait = (earliest_finish - now()).total_seconds() / 60
        
        return max(0, round(wait))

    def on_charge_complete(self, pile_id):
        """充电完成 → 释放功率 → 唤醒排队用户"""
        del self.charging_piles[pile_id]
        
        if self.wait_queue:
            next_in_queue = self.wait_queue.pop(0)
            result = self.request_charge(
                next_in_queue["pile_id"],
                next_in_queue["requested_power"],
                next_in_queue["priority"]
            )
            self.notify_user(next_in_queue["user_id"], 
                           f"充电桩已就绪，预计{result.get('estimated_duration', '?')}分钟完成")

    def on_vehicle_left(self, pile_id):
        """车辆离开 → 释放桩位"""
        if pile_id in self.charging_piles:
            del self.charging_piles[pile_id]
        self.on_charge_complete(pile_id)  # 触发排队唤醒
```

**降功率 vs 排队的权衡：**

| 策略 | 用户等待 | 充电速度 | 体验 |
|------|---------|---------|------|
| 降功率30kW | 0（立即开始） | 慢（80分钟） | 可接受 |
| 排队等60kW | 30-40分钟 | 快（40分钟） | 总时间更长 |
| 混合（先降功率，有空位时升级） | 0→升级 | 渐快 | 最佳 |

**推荐策略：先降功率充电，功率释放后自动升级到满功率。**

```python
    def upgrade_power(self, pile_id):
        """功率释放后，尝试升级降功率车辆的充电速度"""
        if pile_id not in self.charging_piles:
            return
        
        current = self.charging_piles[pile_id]
        if current["power_kw"] < 60:
            # 尝试升级到 60kW
            total_with_upgrade = sum(p["power_kw"] for p in self.charging_piles.values()) \
                                - current["power_kw"] + 60
            
            if total_with_upgrade <= self.max_power_kw:
                current["power_kw"] = 60
                self.notify_user(current["user_id"], "充电速度已升级到60kW")
```

### 预约管理：弹性时间窗 + 占位超时

```python
class ReservationService:
    def reserve(self, user_id, station_id, pile_type, start_time, duration_min):
        """预约充电桩"""
        # 15 分钟宽限：用户迟到 15 分钟内保留预约
        grace_period = 15
        
        # 查找空闲桩
        available_piles = self.find_available(
            station_id, pile_type, start_time,
            start_time + timedelta(minutes=duration_min + grace_period)
        )
        
        if not available_piles:
            return {"status": "no_availability"}
        
        pile = available_piles[0]
        reservation_id = uuid4()
        
        self.db.insert("reservations", {
            "reservation_id": reservation_id,
            "user_id": user_id,
            "pile_id": pile.id,
            "start_time": start_time,
            "end_time": start_time + timedelta(minutes=duration_min),
            "grace_end": start_time + timedelta(minutes=duration_min + grace_period),
            "status": "reserved"
        })
        
        # 设置宽限期到期定时器
        self.schedule_grace_expiry(reservation_id, start_time + timedelta(minutes=grace_period))
        
        return {"status": "reserved", "reservation_id": reservation_id}

    def on_grace_period_expired(self, reservation_id):
        """宽限期过期 → 取消预约 → 释放桩位"""
        reservation = self.db.get("reservations", reservation_id)
        
        if reservation["status"] == "reserved":
            # 用户还没到 → 取消预约
            self.db.update("reservations", {"status": "expired"}, {"reservation_id": reservation_id})
            self.notify_user(reservation["user_id"], "预约已过期，请重新预约")
            # 释放桩位给排队用户
            self.power_scheduler.on_charge_complete(reservation["pile_id"])

    def on_charge_complete_not_left(self, pile_id):
        """充电完成但用户不挪车 → 提醒 + 超时罚则"""
        # 5 分钟提醒
        self.notify_user(user_id, "充电已完成，请在10分钟内挪车")
        
        # 10 分钟超时 → 开始收取占位费（¥0.5/分钟）
        self.schedule_occupancy_fee(pile_id, delay_minutes=10)

    def charge_occupancy_fee(self, pile_id):
        """收取占位费（激励用户挪车）"""
        fee_per_minute = 0.5
        elapsed = (now() - self.get_charge_complete_time(pile_id)).total_seconds() / 60
        
        total_fee = max(0, elapsed - 10) * fee_per_minute  # 10 分钟免费
        
        self.bill_user(user_id, total_fee, reason="占位费")
        self.notify_user(user_id, f"占位费 ¥{total_fee:.1f}，请尽快挪车")
```

### 电价引导：推荐 + 激励

```python
class PricingGuide:
    """分时电价引导"""
    
    TIME_OF_USE = {
        "valley": {"hours": range(22, 24) + range(0, 6), "price_per_kwh": 0.3},
        "flat":   {"hours": range(6, 8) + range(11, 18) + range(21, 22), "price_per_kwh": 0.7},
        "peak":   {"hours": range(8, 11) + range(18, 21), "price_per_kwh": 1.2},
    }

    def get_recommendation(self, user_id, desired_energy_kwh):
        """推荐最优充电时段"""
        options = []
        
        for period, config in self.TIME_OF_USE.items():
            # 充电时间估算（假设快充 60kW）
            hours_needed = desired_energy_kwh / 60
            cost = desired_energy_kwh * config["price_per_kwh"]
            savings_vs_peak = desired_energy_kwh * (1.2 - config["price_per_kwh"])
            
            options.append({
                "period": period,
                "start_hour": config["hours"][0],
                "hours_needed": round(hours_needed, 1),
                "cost": round(cost, 1),
                "savings_vs_peak": round(savings_vs_peak, 1)
            })
        
        options.sort(key=lambda x: x["cost"])
        
        return {
            "recommended": options[0],
            "all_options": options,
            "incentive": self.calculate_incentive(options[0])
        }

    def calculate_incentive(self, best_option):
        """谷时充电激励（折扣/积分）"""
        if best_option["period"] == "valley":
            # 谷时充电额外 10% 折扣
            discount = best_option["cost"] * 0.1
            return {"type": "discount", "amount": round(discount, 1),
                    "message": f"谷时充电额外优惠 ¥{discount:.1f}"}
        return None
```

**电价引导效果估算：**

| 策略 | 谷时利用率提升 | 收入影响 |
|------|--------------|---------|
| 无引导 | 0% | 基线 |
| 推荐+价格展示 | +15% | 略降（谷时单价低） |
| 推荐+折扣激励 | +30% | 谷时收入增加（量增弥补价低） |
| 推荐+占位费 | +20% | 峰时体验改善 |

## 常见陷阱（深度分析）

### 陷阱 1：不做功率调度

**后果：** 超过站点功率上限 → 变电站跳闸 → 全站断电 → 所有正在充电的车辆中断 → 安全事故。

**解决方案：** 功率调度器实时监控总功率，降功率或排队。

### 陷阱 2：预约无宽限期

**后果：** 用户迟到 5 分钟就取消 → 预约体验差 → 用户不愿预约 → 桩利用率低。

**解决方案：** 15 分钟宽限期 + 宽限期到期自动释放。

### 陷阱 3：不引导谷时充电

**后果：** 峰时充电桩拥挤（排队 30 分钟）、谷时空闲（利用率 < 20%） → 整体利用率低。

**解决方案：** 电价引导 + 谷时折扣激励。

### 陷阱 4：充完不挪车无罚则

**后果：** 用户充完后占着桩位 1-2 小时 → 后续用户无法使用 → 约纷投诉。

**解决方案：** 10 分钟免费挪车期 + 超时收取占位费（¥0.5/分钟）。

## 延伸思考

- **V2G（车到网）**：电动车在谷时充电、峰时放电 → 既赚钱又削峰填谷。需要双向充电桩和电池管理协议。
- **自动驾驶代客充电**：车辆自动驶向空闲桩，充完自动驶离 → 利用率大幅提升 + 无占位问题。
- **光伏+储能**：充电站屋顶安装光伏 + 储能电池 → 白天光伏充电，夜间储能放电 → 减少电网依赖。