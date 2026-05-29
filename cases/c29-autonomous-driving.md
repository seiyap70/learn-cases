# C29: 自动驾驶的数据闭环

## 业务场景

某自动驾驶公司，需要构建"数据-标注-训练-部署"的闭环系统。自动驾驶的核心挑战不是算法，而是数据——如何从数百万公里的路测数据中找到有价值的长尾场景（Corner Case），标注后用于模型训练，最终部署到车载系统。

**已知数据：**
- 车队规模：1000 辆
- 每车每天产生数据：1TB（摄像头 6 路、激光雷达、毫米波雷达、GPS、IMU）
- 日均总数据：1000TB（1PB）
- 长尾场景占比：< 0.01%（1PB 中只有约 100MB 有价值）
- 标注成本：3D 边界框 ¥5/帧，语义分割 ¥20/帧
- 模型训练周期：3-7 天
- OTA 部署频率：每 2 周一次

**为什么是难题？**

1PB/天的数据不可能全量标注和训练（成本和时间量级不可行）。核心挑战是"如何从 1PB 中找到那 100MB 有价值的数据"——长尾场景挖掘。99.99% 的路测数据是常规驾驶（高速巡航、城市跟车），对模型训练无价值。只有 0.01% 是长尾场景（行人突然冲出、非标路口、施工路段），这些才是提升模型能力的关键。

## 核心挑战

### 挑战 1：长尾场景挖掘

如何从 1PB 数据中自动识别 < 0.01% 的长尾场景？靠人工查看不可能——1000 辆车 × 每天 24 小时 × 6 路摄像头 = 144000 小时视频/天。

### 挑战 2：标注的规模化

一帧 3D 点云标注需要 30 分钟人工 → 1 万帧需要 100 名标注员 × 5 小时 → 日成本 ¥5 万。长尾场景可能有数百帧 → 标注瓶颈。

### 挑战 3：模型部署的安全验证

新模型上车前必须经过仿真验证。仿真需要覆盖足够多的场景——如何保证仿真场景的分布与真实世界一致？

### 挑战 4：数据闭环的时效性

发现长尾场景 → 标注 → 训练 → 验证 → 部署 → 上车，全流程需要 2-3 周。但新的长尾场景每天都在出现——如何缩短闭环周期？

## 设计约束

- 数据闭环周期 < 1 周（从发现到部署）
- 标注准确率 > 95%
- 仿真覆盖 10000+ 场景
- OTA 部署安全：不能降低现有模型性能

## 请先独立思考（限时 35 分钟）

1. 长尾场景的自动挖掘方案：基于什么信号来判断"这是长尾场景"？
2. 自动标注 + 人工审核的方案：如何平衡标注速度和准确率？
3. 仿真场景库如何构建？如何保证仿真覆盖真实世界的长尾分布？
4. 数据闭环的 4 个阶段（挖掘→标注→训练→部署）如何串联成自动化流水线？

---

## 设计解析

### 数据闭环架构

```
车辆采集 → 边缘过滤（车上初步筛选）→ 云端存储（对象存储）
                                          ↓
                                    长尾挖掘（ML模型筛选）
                                          ↓
                                    自动标注 + 人工审核
                                          ↓
                                    模型训练（GPU集群）
                                          ↓
                                    仿真验证（场景库）
                                          ↓
                                    OTA部署 → 车辆更新
```

### 长尾场景挖掘

**核心思路：用"不确定性"信号筛选长尾场景。** 模型不确定的场景 = 长尾场景。

```python
class CornerCaseMiner:
    def mine(self, frame):
        """判断一帧数据是否是长尾场景"""
        # 信号 1：感知模型推理不确定性（熵）
        entropy = self.compute_perception_entropy(frame)
        
        # 信号 2：场景复杂度（目标数量、遮挡程度、光照变化）
        complexity = self.compute_scene_complexity(frame)
        
        # 信号 3：与训练集分布的距离（新场景 vs 已见场景）
        distribution_shift = self.compute_distribution_shift(frame)
        
        # 信号 4：驾驶行为异常（急刹车、大转向角）
        driving_anomaly = self.compute_driving_anomaly(frame)
        
        # 综合打分
        score = (entropy * 0.35 + complexity * 0.25 + 
                 distribution_shift * 0.25 + driving_anomaly * 0.15)
        
        return score > 0.6  # 高分 = 长尾场景

    def compute_perception_entropy(self, frame):
        """感知模型的不确定性"""
        predictions = self.perception_model.predict(frame)
        
        # 多目标检测的置信度分布
        entropies = [-p.confidence * math.log(p.confidence) for p in predictions if p.confidence > 0]
        
        # 平均熵：熵越高 → 模型越不确定 → 越可能是长尾
        return sum(entropies) / max(len(entropies), 1)

    def compute_scene_complexity(self, frame):
        """场景复杂度评分"""
        # 目标数量（越多越复杂）
        object_count = len(frame.detected_objects)
        
        # 遮挡程度（遮挡越多越复杂）
        occlusion_ratio = sum(o.occlusion for o in frame.detected_objects) / max(object_count, 1)
        
        # 光照变化（逆光、夜间 → 更复杂）
        lighting_score = self.lighting_model.score(frame)
        
        # 综合复杂度
        return min(1.0, object_count / 20 * 0.4 + occlusion_ratio * 0.3 + lighting_score * 0.3)

    def compute_driving_anomaly(self, frame):
        """驾驶行为异常度"""
        # 急刹车（加速度 > 3m/s²）
        # 大转向角（方向盘转角 > 30°）
        # 突然变道
        anomaly = 0
        
        if abs(frame.acceleration) > 3.0:
            anomaly += 0.5
        if abs(frame.steering_angle) > 30:
            anomaly += 0.3
        if frame.lane_change_detected:
            anomaly += 0.2
        
        return min(1.0, anomaly)
```

**边缘过滤：车上初步筛选，减少上传量**

```python
class EdgeFilter:
    """车载边缘计算：初步筛选，只上传有价值的帧"""
    
    def on_frame(self, frame):
        # 轻量级判断（车上 GPU 算力有限）
        score = self.lightweight_miner.mine(frame)
        
        if score > 0.3:  # 稍宽松的阈值，避免漏掉
            # 上传到云端做更精确的筛选
            self.upload_to_cloud(frame)
            return True
        else:
            # 丢弃（99.99% 的帧）
            return False
```

**上传量从 1PB/天 → 约 1GB/天**（99.99% 的帧被边缘过滤丢弃）。

### 自动标注 + 人工审核

```python
class AutoLabeler:
    def auto_label(self, frame):
        """先用感知模型自动标注，再人工审核修正"""
        # 模型预标注
        predictions = self.perception_model.predict(frame)
        
        # 置信度分级
        auto_approved = []   # 置信度 > 0.95 → 自动通过
        needs_review = []    # 置信度 0.5-0.95 → 送人工审核
        needs_full_manual = []  # 置信度 < 0.5 → 完全人工标注
        
        for pred in predictions:
            if pred.confidence > 0.95:
                auto_approved.append(pred)
            elif pred.confidence > 0.5:
                needs_review.append(pred)
            else:
                needs_full_manual.append(pred)
        
        savings_pct = len(auto_approved) / max(len(predictions), 1) * 100
        
        return {
            "auto_labels": auto_approved,      # ~70% 自动通过
            "needs_review": needs_review,      # ~20% 人工修正
            "needs_full_manual": needs_full_manual,  # ~10% 完全人工
            "savings_pct": savings_pct
        }
```

**标注成本对比：**

| 方案 | 1万帧成本 | 耗时 |
|------|---------|------|
| 全量人工 | ¥5万（3D边界框）| 100名标注员×5h |
| 自动+人工审核 | ¥1.5万（70%自动通过）| 30名标注员×5h |
| 节省 | 70% | 70% |

### 仿真验证：场景库

```python
class SimulationValidator:
    def validate_model(self, new_model, baseline_model):
        """新模型 vs 基线模型在场景库中的表现对比"""
        
        results = {"new_model": {}, "baseline": {}}
        
        for scenario in self.scenario_library:
            # 在仿真器中运行两个模型
            new_result = self.simulator.run(new_model, scenario)
            baseline_result = self.simulator.run(baseline_model, scenario)
            
            # 评估指标：碰撞率、违规率、舒适度
            results["new_model"][scenario.id] = self.evaluate(new_result)
            results["baseline"][scenario.id] = self.evaluate(baseline_result)
        
        # 关键检查：新模型不能在任何场景上比基线差
        degradation_count = 0
        for scenario_id in results["new_model"]:
            if results["new_model"][scenario_id]["score"] < results["baseline"][scenario_id]["score"]:
                degradation_count += 1
        
        if degradation_count > 0:
            # 有退化 → 不能部署 → 需要分析退化原因
            self.report_degradation(results, degradation_count)
            return {"decision": "reject", "reason": f"{degradation_count} 个场景性能退化"}
        
        # 新模型在所有场景上 ≥ 基线 → 可以部署
        improvement_count = sum(
            1 for sid in results["new_model"]
            if results["new_model"][sid]["score"] > results["baseline"][sid]["score"]
        )
        
        return {"decision": "approve", "improvements": improvement_count}
```

**场景库构建：**
- 路测数据中的长尾场景（自动挖掘）
- 合规场景（法规要求的测试场景，如 AEB 紧急制动）
- 对抗场景（故意设计的极端场景，如连环碰撞）
- 总计 10000+ 场景，覆盖 95%+ 的真实世界长尾分布

### OTA 安全部署

```python
class OTADeployer:
    def deploy(self, model_version, target_vehicle_ids):
        """灰度 OTA 部署"""
        
        # 1. 先部署到 10 辆测试车
        test_vehicles = target_vehicle_ids[:10]
        self.push_model(model_version, test_vehicles)
        
        # 2. 观察 1 周：测试车的表现是否正常
        self.observe_period = timedelta(days=7)
        
        # 3. 如果正常 → 扩大到 100 辆 → 观察 3 天 → 全量部署
        # 如果异常 → 回滚到基线模型
        pass
```

## 常见陷阱（深度分析）

### 陷阱 1：全量标注

1PB/天全量标注 → 不可承受的成本和时间。

**解决方案：** 长尾挖掘 + 自动标注 + 人工审核，只标注 < 0.01% 的有价值数据。

### 陷阱 2：不做长尾挖掘，随机采样训练数据

随机采样 → 99.99% 是常规场景 → 模型对常规场景表现好，但对长尾场景（行人突然冲出）表现差 → 安全事故。

**解决方案：** 用不确定性信号主动挖掘长尾场景，确保训练数据覆盖长尾分布。

### 陷阱 3：模型直接上车不做仿真验证

新模型可能在某些场景上比基线差 → 部署后可能导致安全事故。

**解决方案：** 场景库仿真验证 + 灰度 OTA 部署。

### 陷阱 4：仿真只覆盖法规场景

法规场景（如 AEB 测试）是有限的、标准化的。但真实世界的长尾场景是无限的、不可预测的。只覆盖法规场景 → 漏测大量真实长尾。

**解决方案：** 场景库 = 法规场景 + 路测长尾 + 对抗场景，覆盖 10000+ 场景。

## 延伸思考

- **联邦学习**：多车队的驾驶数据不出本地，只共享模型梯度 → 隐私保护 + 协作训练。
- **世界模型**：学习驾驶环境的生成模型 → 无限生成仿真场景 → 场景库动态扩展。
- **数据闭环加速**：自动标注 → 自动训练 → 自动仿真 → 自动部署，全流程无人介入。