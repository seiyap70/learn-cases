# C29: 自动驾驶的数据闭环

## 业务场景

某自动驾驶公司，需要构建"数据-标注-训练-部署"的闭环系统。自动驾驶的核心挑战不是算法，而是数据——如何从数百万公里的路测数据中找到有价值的长尾场景（Corner Case），标注后用于模型训练，最终部署到车载系统。

**已知数据：**
- 车队规模：1000 辆测试车
- 每车每天产生数据：1TB（6路摄像头+激光雷达+毫米波雷达+GPS+IMU）
- 日均总数据：1000TB（约 1PB）
- 长尾场景占比：< 0.01%（1PB 中只有约 100MB 有价值）
- 标注成本：3D 边界框 ¥5/帧，语义分割 ¥20/帧
- 模型训练周期：3-7 天（8×A100 GPU 集群）
- OTA 部署频率：每 2 周一次

**为什么是难题？**

1PB/天的数据不可能全量标注和训练。核心挑战是"如何从 1PB 中找到那 100MB"——长尾场景挖掘。99.99% 的路测数据是常规驾驶（高速巡航、城市跟车），对模型训练无价值。只有 0.01% 是长尾场景（行人突然冲出、非标路口、施工路段），这些才是提升模型能力的关键。

## 核心挑战

### 挑战 1：长尾场景挖掘

如何从 1PB 数据中自动识别 < 0.01% 的长尾场景？靠人工查看不可能——1000 辆车 × 24小时 × 6路 = 144000 小时视频/天。

### 挑战 2：标注的规模化

一帧 3D 点云标注需要 30 分钟人工 → 1 万帧需要 100 名标注员 × 5 小时 → 日成本 ¥5 万。如何减少标注量？

### 挑战 3：模型部署的安全验证

新模型上车前必须经过仿真验证。仿真需要覆盖足够多的场景——如何保证仿真场景的分布与真实世界一致？

### 挑战 4：数据闭环的时效性

发现长尾场景 → 标注 → 训练 → 验证 → 部署 → 上车，全流程需要 2-3 周。但新的长尾场景每天都在出现——如何缩短闭环周期？

## 设计约束

- 数据闭环周期 < 1 周（从发现到部署）
- 标注准确率 > 95%
- 仿真覆盖 10000+ 场景
- OTA 部署安全：新模型不能降低现有性能

## 数据库与存储设计

自动驾驶数据闭环需要管理海量多模态数据，关系型数据库负责元数据与流程追踪，对象存储负责原始数据归档。以下是核心表设计。

### driving_sessions（路测会话表）

```sql
CREATE TABLE driving_sessions (
    session_id        BIGINT PRIMARY KEY AUTO_INCREMENT,
    vehicle_id        VARCHAR(32) NOT NULL COMMENT '车辆唯一标识',
    start_time        DATETIME NOT NULL COMMENT '会话开始时间',
    end_time          DATETIME NOT NULL COMMENT '会话结束时间',
    total_frames      INT NOT NULL DEFAULT 0 COMMENT '总帧数',
    total_size_bytes  BIGINT NOT NULL DEFAULT 0 COMMENT '原始数据大小(字节)',
    uploaded_frames   INT NOT NULL DEFAULT 0 COMMENT '边缘过滤后上传帧数',
    uploaded_bytes    BIGINT NOT NULL DEFAULT 0 COMMENT '上传数据大小(字节)',
    route_hash        VARCHAR(64) COMMENT '路线哈希，用于去重',
    weather_tag       VARCHAR(32) COMMENT '天气标签(sunny/rainy/foggy/snowy)',
    road_type         VARCHAR(32) COMMENT '道路类型(highway/urban/rural)',
    firmware_version  VARCHAR(64) COMMENT '车载固件版本',
    status            ENUM('uploading','uploaded','mined','archived') DEFAULT 'uploading',
    created_at        DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_vehicle_time (vehicle_id, start_time),
    INDEX idx_status (status),
    INDEX idx_weather_road (weather_tag, road_type)
) ENGINE=InnoDB COMMENT='路测会话元数据';
```

### corner_cases（长尾场景表）

```sql
CREATE TABLE corner_cases (
    case_id           BIGINT PRIMARY KEY AUTO_INCREMENT,
    session_id        BIGINT NOT NULL COMMENT '关联路测会话',
    frame_index       INT NOT NULL COMMENT '帧序号',
    timestamp_ms      BIGINT NOT NULL COMMENT '帧内毫秒时间戳',
    entropy_score     FLOAT NOT NULL COMMENT '感知不确定性得分',
    complexity_score  FLOAT NOT NULL COMMENT '场景复杂度得分',
    dist_shift_score  FLOAT NOT NULL COMMENT '分布偏移得分',
    anomaly_score     FLOAT NOT NULL COMMENT '驾驶异常度得分',
    fused_score       FLOAT NOT NULL COMMENT '融合得分',
    case_type         VARCHAR(64) COMMENT '场景类型标签(pedestrian_rush/construction/unmarked_intersection...)',
    storage_path      VARCHAR(512) NOT NULL COMMENT '对象存储路径(s3://...)',
    frame_size_bytes  BIGINT NOT NULL COMMENT '帧数据大小',
    thumbnail_path    VARCHAR(512) COMMENT '缩略图路径(用于人工审核)',
    source            ENUM('edge_filter','cloud_mining','manual_submit') NOT NULL COMMENT '发现来源',
    status            ENUM('detected','labeled','training','deployed','rejected') DEFAULT 'detected',
    created_at        DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (session_id) REFERENCES driving_sessions(session_id),
    INDEX idx_fused_score (fused_score DESC),
    INDEX idx_status (status),
    INDEX idx_type (case_type),
    INDEX idx_session_frame (session_id, frame_index)
) ENGINE=InnoDB COMMENT='长尾场景记录';
```

### labeling_tasks（标注任务表）

```sql
CREATE TABLE labeling_tasks (
    task_id           BIGINT PRIMARY KEY AUTO_INCREMENT,
    case_id           BIGINT NOT NULL COMMENT '关联长尾场景',
    task_type         ENUM('auto_approved','needs_review','full_manual') NOT NULL COMMENT '标注类型',
    label_type        ENUM('3d_bbox','semantic_seg','2d_bbox','keypoint') NOT NULL COMMENT '标注类型',
    auto_label_json   JSON COMMENT '自动标注结果(JSON存储)',
    auto_confidence   FLOAT COMMENT '自动标注置信度',
    reviewer_id       VARCHAR(32) COMMENT '审核员ID',
    review_result     JSON COMMENT '人工修正后的标注结果',
    review_status     ENUM('pending','in_review','approved','rejected') DEFAULT 'pending',
    review_time_ms    INT COMMENT '审核耗时(毫秒)',
    quality_score     FLOAT COMMENT '标注质量评分(0-1)',
    cost_cents        INT COMMENT '标注成本(分)',
    created_at        DATETIME DEFAULT CURRENT_TIMESTAMP,
    reviewed_at       DATETIME COMMENT '审核完成时间',
    FOREIGN KEY (case_id) REFERENCES corner_cases(case_id),
    INDEX idx_review_status (review_status),
    INDEX idx_reviewer (reviewer_id, review_status),
    INDEX idx_task_type (task_type, label_type)
) ENGINE=InnoDB COMMENT='标注任务追踪';
```

### model_versions（模型版本表）

```sql
CREATE TABLE model_versions (
    version_id        VARCHAR(64) PRIMARY KEY COMMENT '语义化版本号(v2.3.1)',
    base_version      VARCHAR(64) COMMENT '基线模型版本',
    training_config   JSON NOT NULL COMMENT '训练超参数配置',
    training_data_ids JSON COMMENT '训练数据集ID列表',
    training_data_size INT COMMENT '训练数据帧数',
    training_duration_h FLOAT COMMENT '训练时长(小时)',
    gpu_hours         FLOAT COMMENT 'GPU消耗(小时)',
    validation_result JSON COMMENT '仿真验证结果摘要',
    validation_pass   BOOLEAN COMMENT '是否通过仿真验证',
    deploy_status     ENUM('pending','simulating','approved','deploying','deployed','rolled_back') DEFAULT 'pending',
    deployed_vehicles INT DEFAULT 0 COMMENT '已部署车辆数',
    rollback_reason   VARCHAR(512) COMMENT '回滚原因',
    ota_batch_id      VARCHAR(64) COMMENT 'OTA批次号',
    created_at        DATETIME DEFAULT CURRENT_TIMESTAMP,
    deployed_at       DATETIME COMMENT '部署时间',
    INDEX idx_deploy_status (deploy_status),
    INDEX idx_base (base_version)
) ENGINE=InnoDB COMMENT='模型版本管理';
```

### simulation_results（仿真结果表）

```sql
CREATE TABLE simulation_results (
    result_id         BIGINT PRIMARY KEY AUTO_INCREMENT,
    version_id        VARCHAR(64) NOT NULL COMMENT '被测模型版本',
    scenario_id       VARCHAR(64) NOT NULL COMMENT '场景ID',
    scenario_category VARCHAR(32) NOT NULL COMMENT '场景类别(regulation/corner_case/adversarial/random)',
    collision         BOOLEAN NOT NULL DEFAULT FALSE COMMENT '是否碰撞',
    violation         BOOLEAN NOT NULL DEFAULT FALSE COMMENT '是否违规',
    comfort_score     FLOAT COMMENT '舒适度评分',
    completion_score  FLOAT COMMENT '任务完成评分',
    overall_score     FLOAT NOT NULL COMMENT '综合评分',
    baseline_score    FLOAT COMMENT '基线模型该场景得分',
    score_delta       FLOAT COMMENT '与基线差值(正=提升)',
    detail_json       JSON COMMENT '详细仿真日志',
    sim_duration_ms   INT COMMENT '仿真耗时(毫秒)',
    created_at        DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (version_id) REFERENCES model_versions(version_id),
    INDEX idx_version (version_id),
    INDEX idx_scenario_cat (scenario_category),
    INDEX idx_score_delta (score_delta),
    INDEX idx_collision (collision)
) ENGINE=InnoDB COMMENT='仿真验证结果';
```

### 存储分层策略

```python
class StorageTierManager:
    """自动驾驶数据存储分层：热→温→冷→归档"""

    TIERS = {
        "hot":    {"backend": "local_ssd",  "retention_days": 7,   "cost_per_tb_month": 800},    # 训练加速
        "warm":   {"backend": "s3_standard","retention_days": 30,  "cost_per_tb_month": 230},    # 近期数据
        "cold":   {"backend": "s3_ia",      "retention_days": 180, "cost_per_tb_month": 125},    # 历史数据
        "archive":{"backend": "s3_glacier", "retention_days": 3650,"cost_per_tb_month": 45},     # 合规归档
    }

    def auto_tier(self, data_age_days: int, access_count: int) -> str:
        """根据数据年龄和访问频率自动分层"""
        if data_age_days <= 7 and access_count > 10:
            return "hot"
        elif data_age_days <= 30:
            return "warm"
        elif data_age_days <= 180:
            return "cold"
        else:
            return "archive"

    def migrate(self, session_id: int, target_tier: str):
        """执行数据迁移"""
        session = db.query("SELECT * FROM driving_sessions WHERE session_id = %s", session_id)
        src_path = session["storage_path"]
        dst_backend = self.TIERS[target_tier]["backend"]
        new_path = self._copy_to_backend(src_path, dst_backend)
        db.update("UPDATE driving_sessions SET storage_path = %s WHERE session_id = %s",
                  new_path, session_id)
```

## 请先独立思考（限时 35 分钟）

1. 长尾场景的自动挖掘方案：基于什么信号来判断"这是长尾场景"？感知模型不确定性？场景复杂度？驾驶行为异常？
2. 自动标注 + 人工审核的方案：如何平衡标注速度和准确率？哪些可以用自动标注，哪些必须人工？
3. 仿真场景库如何构建？如何保证仿真覆盖真实世界的长尾分布？
4. 数据闭环的 4 个阶段（挖掘→标注→训练→部署）如何串联成自动化流水线？

---

## 设计解析

### 数据闭环架构

```
车辆采集 → 边缘过滤（车上初步筛选）→ 云端存储（对象存储）
                                          ↓
                                    尾挖掘（ML模型筛选）
                                          ↓
                                    自动标注 + 人工审核
                                          ↓
                                    模型训练（GPU集群）
                                          ↓
                                    仿真验证（场景库）
                                          ↓
                                    OTA部署 → 车辆更新
```

### 长尾场景挖掘：多信号融合

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
        entropies = [-p.confidence * math.log(p.confidence) 
                     for p in predictions if p.confidence > 0]
        
        # 平均熵：熵越高 → 模型越不确定 → 越可能是长尾
        return sum(entropies) / max(len(entropies), 1)

    def compute_scene_complexity(self, frame):
        """场景复杂度评分"""
        object_count = len(frame.detected_objects)
        occlusion_ratio = sum(o.occlusion for o in frame.detected_objects) / max(object_count, 1)
        lighting_score = self.lighting_model.score(frame)
        
        return min(1.0, object_count / 20 * 0.4 + 
                   occlusion_ratio * 0.3 + lighting_score * 0.3)

    def compute_distribution_shift(self, frame):
        """与训练集分布的距离"""
        # 提取场景特征向量
        features = self.scene_encoder.encode(frame)
        
        # 计算 features 与训练集最近邻的距离
        nearest = self.train_set_index.search(features, k=1)
        
        # 距离越大 → 越偏离训练集 → 越可能是长尾
        return min(1.0, nearest.distance / 5.0)

    def compute_driving_anomaly(self, frame):
        """驾驶行为异常度"""
        anomaly = 0
        if abs(frame.acceleration) > 3.0:  anomaly += 0.5  # 急刹车
        if abs(frame.steering_angle) > 30:  anomaly += 0.3  # 大转向
        if frame.lane_change_detected:      anomaly += 0.2  # 突然变道
        return min(1.0, anomaly)
```

**各信号的量化效果：**

| 信号 | 捕获的长尾类型 | 误报率 | 漏报率 |
|------|--------------|-------|-------|
| 感知不确定性 | 陌生目标、遮挡场景 | 15% | 5% |
| 场景复杂度 | 多目标、逆光 | 20% | 10% |
| 分布偏移 | 完全未见的新场景 | 10% | 15% |
| 驾驶异常 | 紧急避障、非预期行为 | 25% | 20% |
| **融合（4信号加权）** | **所有类型** | **8%** | **3%** |

### 边缘过滤：车上初步筛选

```python
class EdgeFilter:
    """车载边缘计算：初步筛选，只上传有价值的帧"""
    
    def __init__(self):
        # 车载 GPU 算力有限（约 30 TOPS）
        # 使用轻量级模型做初步筛选
        self.lightweight_miner = LightweightCornerCaseMiner()
    
    def on_frame(self, frame):
        # 轻量级判断（车上只算 2 个信号：感知不确定性 + 驾驶异常）
        # 车上不跑复杂模型（场景复杂度和分布偏移在云端计算）
        entropy = self.compute_perception_entropy_lite(frame)
        anomaly = self.compute_driving_anomaly_lite(frame)
        
        score = entropy * 0.6 + anomaly * 0.4
        
        if score > 0.3:  # 稍宽松的阈值（避免边缘漏掉）
            # 上传到云端做更精确的筛选
            # 只上传关键帧（不是全部 1TB），约 100MB
            compressed = self.compress_frame(frame)  # 去掉冗余数据
            self.upload_to_cloud(compressed)
            return True
        else:
            # 丢弃（99.99% 的帧）
            return False
```

**上传量从 1PB/天 → 约 1GB/天**（99.99% 的帧被边缘过滤丢弃）。

#### 边缘过滤完整实现：压缩与上传

车端带宽有限（4G/5G 上行约 50Mbps），即使只上传 1GB/天也需要高效压缩和可靠上传。

```python
import zlib
import hashlib
import time
import threading
from queue import Queue
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class FrameData:
    """一帧多传感器数据"""
    frame_id: str
    timestamp_ns: int                     # 纳秒时间戳
    camera_images: dict                   # {camera_id: numpy_array}
    lidar_points: Optional[bytes] = None  # 点云二进制（可选）
    radar_data: Optional[bytes] = None    # 毫米波雷达
    can_signals: dict = field(default_factory=dict)  # CAN总线信号（加速度、转向角等）
    perception_results: dict = field(default_factory=dict)  # 车端感知模型输出
    entropy_score: float = 0.0
    anomaly_score: float = 0.0


class FrameCompressor:
    """帧数据压缩：去除冗余，保留关键信息"""

    # 不同传感器的压缩策略
    CAMERA_QUALITY = 85           # JPEG 压缩质量（原图质量 100，85 几乎无视觉损失）
    CAMERA_KEYFRAME_QUALITY = 95  # 关键帧更高质量
    LIDAR_VOXEL_SIZE = 0.1        # 点云体素化分辨率(米)，0.1m 精度足够
    POINTCLOUD_MAX_POINTS = 50000 # 保留最大点数（原始约 20万点）

    def compress_frame(self, frame: FrameData, is_keyframe: bool = False) -> bytes:
        """压缩一帧数据，典型压缩比 10:1"""
        compressed_parts = {}

        # 1. 摄像头图像压缩
        for cam_id, img_array in frame.camera_images.items():
            quality = self.CAMERA_KEYFRAME_QUALITY if is_keyframe else self.CAMERA_QUALITY
            # 只裁剪检测到目标的 ROI 区域（减少无关背景）
            rois = frame.perception_results.get(cam_id, {}).get("rois", [])
            if rois and not is_keyframe:
                # 裁剪到目标区域，保留 20% 上下文
                img_cropped = self._crop_with_context(img_array, rois, context_ratio=0.2)
                compressed_parts[f"cam_{cam_id}"] = self._encode_jpeg(img_cropped, quality)
            else:
                compressed_parts[f"cam_{cam_id}"] = self._encode_jpeg(img_array, quality)

        # 2. 点云压缩：体素化降采样 + 只保留前景点
        if frame.lidar_points is not None:
            points = self._decode_pointcloud(frame.lidar_points)
            # 体素化降采样：20万点 → 5万点
            downsampled = self._voxel_downsample(points, self.LIDAR_VOXEL_SIZE)
            # 只保留感知范围内的点（自车 80m 半径）
            filtered = downsampled[downsampled[:, 0]**2 + downsampled[:, 1]**2 <= 80**2]
            if len(filtered) > self.POINTCLOUD_MAX_POINTS:
                # 最远点采样，保留空间结构
                filtered = self._furthest_point_sample(filtered, self.POINTCLOUD_MAX_POINTS)
            compressed_parts["lidar"] = self._encode_pointcloud(filtered)

        # 3. 毫米波雷达数据量小，直接序列化
        if frame.radar_data is not None:
            compressed_parts["radar"] = zlib.compress(frame.radar_data, level=6)

        # 4. CAN 信号与感知结果：JSON + zlib
        metadata = {
            "frame_id": frame.frame_id,
            "timestamp_ns": frame.timestamp_ns,
            "can_signals": frame.can_signals,
            "perception_results": frame.perception_results,
            "entropy_score": frame.entropy_score,
            "anomaly_score": frame.anomaly_score,
        }
        compressed_parts["metadata"] = zlib.compress(
            json.dumps(metadata).encode("utf-8"), level=6
        )

        # 5. 整体打包
        return self._pack_parts(compressed_parts)

    def _crop_with_context(self, img, rois, context_ratio=0.2):
        """裁剪到目标区域，保留上下文"""
        h, w = img.shape[:2]
        x_min = max(0, min(r["x1"] for r in rois) - int(w * context_ratio))
        x_max = min(w, max(r["x2"] for r in rois) + int(w * context_ratio))
        y_min = max(0, min(r["y1"] for r in rois) - int(h * context_ratio))
        y_max = min(h, max(r["y2"] for r in rois) + int(h * context_ratio))
        return img[y_min:y_max, x_min:x_max]

    def _pack_parts(self, parts: dict) -> bytes:
        """将多部分数据打包为二进制，带索引头"""
        header = {}
        offset = 0
        body = b""
        for key, data in parts.items():
            header[key] = {"offset": offset, "length": len(data)}
            body += data
            offset += len(data)
        header_bytes = json.dumps(header).encode("utf-8")
        header_len = len(header_bytes).to_bytes(4, "big")
        return header_len + header_bytes + body


class CloudUploader:
    """车端上传管理器：断点续传 + 带宽控制 + 优先级队列"""

    def __init__(self, endpoint: str, bandwidth_limit_mbps: float = 50.0):
        self.endpoint = endpoint
        self.bandwidth_limit = bandwidth_limit_mbps
        self.upload_queue: Queue = Queue(maxsize=1000)  # 最多缓存 1000 帧
        self.pending_acks: dict = {}                     # 等待确认的上传
        self.local_buffer_dir = "/data/upload_buffer"    # 本地缓冲目录
        self._start_upload_thread()

    def enqueue(self, frame: FrameData, compressed: bytes, priority: str = "normal"):
        """将压缩帧加入上传队列"""
        item = {
            "frame_id": frame.frame_id,
            "data": compressed,
            "size": len(compressed),
            "priority": priority,   # "high" | "normal" | "low"
            "timestamp": time.time(),
            "retries": 0,
        }
        try:
            self.upload_queue.put_nowait(item)
        except Full:
            # 队列满 → 本地落盘，等待后续上传
            self._spill_to_disk(item)

    def _start_upload_thread(self):
        """后台上传线程"""
        def upload_loop():
            while True:
                item = self.upload_queue.get(block=True)
                try:
                    self._upload_one(item)
                except Exception as e:
                    item["retries"] += 1
                    if item["retries"] < 5:
                        # 指数退避重试
                        time.sleep(min(2 ** item["retries"], 300))
                        self.upload_queue.put(item)
                    else:
                        # 超过重试次数，存入本地等待 WiFi 场景下手动同步
                        self._spill_to_disk(item)
                        logging.error(f"上传失败，已落盘: {item['frame_id']}, error: {e}")

        threading.Thread(target=upload_loop, daemon=True).start()

    def _upload_one(self, item: dict):
        """单帧上传，支持断点续传"""
        frame_id = item["frame_id"]
        data = item["data"]

        # 分片上传（每片 512KB，适配弱网环境）
        CHUNK_SIZE = 512 * 1024
        total_chunks = (len(data) + CHUNK_SIZE - 1) // CHUNK_SIZE

        # 先注册上传会话
        upload_id = self._init_multipart_upload(frame_id, total_chunks, item["size"])

        for chunk_idx in range(total_chunks):
            start = chunk_idx * CHUNK_SIZE
            end = min(start + CHUNK_SIZE, len(data))
            chunk_data = data[start:end]

            # 带宽限速
            self._rate_limit(len(chunk_data))

            # 上传分片
            self._upload_chunk(upload_id, chunk_idx, chunk_data)

        # 完成上传
        self._complete_multipart_upload(upload_id)
        logging.info(f"上传完成: {frame_id}, size={item['size']} bytes, chunks={total_chunks}")

    def _rate_limit(self, bytes_sent: int):
        """令牌桶限速"""
        target_time = bytes_sent / (self.bandwidth_limit_mbps * 1024 * 1024 / 8)
        time.sleep(max(0, target_time))

    def _spill_to_disk(self, item: dict):
        """队列溢出时，将帧数据写入本地磁盘缓冲"""
        path = os.path.join(self.local_buffer_dir, f"{item['frame_id']}.bin")
        with open(path, "wb") as f:
            f.write(item["data"])
        # 记录到本地 SQLite，WiFi 连接时由同步服务扫描上传
        self.local_db.execute(
            "INSERT INTO pending_uploads (frame_id, path, size, priority) VALUES (?, ?, ?, ?)",
            (item["frame_id"], path, item["size"], item["priority"])
        )


class EdgeFilterV2:
    """边缘过滤完整版：筛选 + 压缩 + 上传"""

    def __init__(self):
        self.compressor = FrameCompressor()
        self.uploader = CloudUploader(endpoint="https://cloud-api.autodrive.com/upload")
        self.entropy_threshold = 0.3
        self.keyframe_interval = 30  # 每 30 帧一个关键帧（保证时序上下文）

        # 统计计数器
        self.stats = {"total_frames": 0, "filtered_out": 0, "uploaded": 0, "bytes_uploaded": 0}

    def on_frame(self, frame: FrameData):
        """处理每一帧"""
        self.stats["total_frames"] += 1

        # 1. 轻量级打分
        entropy = self._compute_entropy_lite(frame)
        anomaly = self._compute_anomaly_lite(frame)
        frame.entropy_score = entropy
        frame.anomaly_score = anomaly

        score = entropy * 0.6 + anomaly * 0.4

        # 2. 判断是否需要上传
        is_keyframe = (self.stats["total_frames"] % self.keyframe_interval == 0)

        if score > self.entropy_threshold or is_keyframe:
            # 3. 压缩
            compressed = self.compressor.compress_frame(frame, is_keyframe=is_keyframe)

            # 4. 设置上传优先级
            priority = "high" if score > 0.6 else ("normal" if score > 0.3 else "low")

            # 5. 加入上传队列
            self.uploader.enqueue(frame, compressed, priority=priority)

            self.stats["uploaded"] += 1
            self.stats["bytes_uploaded"] += len(compressed)
        else:
            self.stats["filtered_out"] += 1

        # 每 1000 帧输出一次统计
        if self.stats["total_frames"] % 1000 == 0:
            self._log_stats()

    def _compute_entropy_lite(self, frame: FrameData) -> float:
        """车端轻量级不确定性计算（不跑完整感知模型）"""
        # 使用车端感知模型的输出置信度直接计算
        if not frame.perception_results:
            return 0.0
        confidences = []
        for cam_result in frame.perception_results.values():
            for obj in cam_result.get("objects", []):
                confidences.append(obj.get("confidence", 1.0))
        if not confidences:
            return 0.0
        # 平均熵
        import math
        entropy = sum(-c * math.log(c + 1e-9) for c in confidences if 0 < c < 1) / len(confidences)
        return min(1.0, entropy / 0.5)  # 归一化

    def _compute_anomaly_lite(self, frame: FrameData) -> float:
        """车端轻量级驾驶异常度"""
        anomaly = 0.0
        can = frame.can_signals
        if abs(can.get("acceleration_mps2", 0)) > 3.0: anomaly += 0.5
        if abs(can.get("steering_angle_deg", 0)) > 30: anomaly += 0.3
        if can.get("lane_change_flag", False):          anomaly += 0.2
        return min(1.0, anomaly)

    def _log_stats(self):
        """输出过滤统计"""
        total = self.stats["total_frames"]
        uploaded = self.stats["uploaded"]
        ratio = uploaded / max(total, 1) * 100
        mb = self.stats["bytes_uploaded"] / (1024 * 1024)
        logging.info(
            f"EdgeFilter: total={total}, uploaded={uploaded} ({ratio:.2f}%), "
            f"upload_size={mb:.1f}MB"
        )
```

**压缩效果量化：**

| 数据类型 | 原始大小 | 压缩后 | 压缩比 | 说明 |
|---------|---------|-------|-------|------|
| 6路摄像头 | 600MB | 60MB | 10:1 | JPEG 85 + ROI 裁剪 |
| 激光雷达点云 | 300MB | 15MB | 20:1 | 体素降采样 + 距离过滤 |
| 毫米波雷达 | 50MB | 5MB | 10:1 | zlib 压缩 |
| CAN/感知元数据 | 50MB | 2MB | 25:1 | JSON+zlib |
| **单帧合计** | **1GB** | **~82MB** | **~12:1** | |

### 自动标注 + 人工审核

```python
class AutoLabeler:
    def auto_label(self, frame):
        """先用感知模型自动标注，再按置信度分级"""
        predictions = self.perception_model.predict(frame)
        
        auto_approved = []       # 置信度 > 0.95 → 自动通过
        needs_review = []        # 置信度 0.5-0.95 → 人工修正
        needs_full_manual = []   # 置信度 < 0.5 → 完全人工标注
        
        for pred in predictions:
            if pred.confidence > 0.95:
                auto_approved.append(pred)
            elif pred.confidence > 0.5:
                needs_review.append(pred)
            else:
                needs_full_manual.append(pred)
        
        return {
            "auto_labels": auto_approved,      # ~70% 自动通过
            "needs_review": needs_review,      # ~20% 人工修正
            "needs_full_manual": needs_full_manual,  # ~10% 完全人工
            "total": len(predictions),
            "auto_pct": len(auto_approved) / max(len(predictions), 1) * 100
        }
```

**标注成本对比：**

| 方案 | 1万帧成本 | 耗时 | 准确率 |
|------|---------|------|-------|
| 全量人工（3D边界框） | ¥5万 | 100标注员×5h | 99% |
| 自动标注+人工审核 | ¥1.5万（70%自动） | 30标注员×5h | 95%（自动标注部分） |
| 全量自动标注 | ¥0 | 0 | 85%（不可接受） |

选择"自动标注+人工审核"：成本降低 70%，准确率 95%（满足要求）。

**人工审核流程：**

```python
class HumanReviewPlatform:
    def review_frame(self, reviewer_id, frame, auto_labels):
        """标注员审核自动标注结果"""
        # 显示自动标注 + 原始帧叠加
        # 标注员可以：确认/修正/删除/新增标注
        corrections = []
        
        for label in auto_labels:
            reviewer_action = self.get_reviewer_action(reviewer_id, label)
            if reviewer_action == "confirm":
                corrections.append(label)  # 确认自动标注
            elif reviewer_action == "modify":
                corrected = self.get_corrected_label(reviewer_id, label)
                corrections.append(corrected)  # 修正
            elif reviewer_action == "delete":
                pass  # 删除误标注
        
        # 新增遗漏的标注
        new_labels = self.get_new_labels(reviewer_id, frame)
        corrections.extend(new_labels)
        
        return corrections
```

### 仿真验证：场景库 + 性能对比

```python
class SimulationValidator:
    def validate_model(self, new_model, baseline_model):
        """新模型 vs 基线模型在场景库中的表现对比"""
        
        results = {"new_model": {}, "baseline": {}}
        
        for scenario in self.scenario_library:
            new_result = self.simulator.run(new_model, scenario)
            baseline_result = self.simulator.run(baseline_model, scenario)
            
            results["new_model"][scenario.id] = self.evaluate(new_result)
            results["baseline"][scenario.id] = self.evaluate(baseline_result)
        
        # 关键检查：新模型不能在任何场景上比基线差
        degradation_count = 0
        for scenario_id in results["new_model"]:
            new_score = results["new_model"][scenario_id]["score"]
            base_score = results["baseline"][scenario_id]["score"]
            
            if new_score < base_score * 0.95:  # 允许 5% 的微小退化
                degradation_count += 1
        
        if degradation_count > 0:
            self.report_degradation(results, degradation_count)
            return {"decision": "reject", "reason": f"{degradation_count} 个场景性能退化"}
        
        improvement_count = sum(
            1 for sid in results["new_model"]
            if results["new_model"][sid]["score"] > results["baseline"][sid]["score"] * 1.05
        )
        
        return {"decision": "approve", "improvements": improvement_count}
```

**场景库构成：**

| 来源 | 数量 | 说明 |
|------|------|------|
| 路测长尾场景 | 5000+ | 自动挖掘的真实场景 |
| 法规合规场景 | 500+ | AEB、LKA 等法规测试 |
| 对抗场景 | 1000+ | 极端天气、连环碰撞 |
| 随机仿真 | 3500+ | 参数随机化生成的补充场景 |
| **总计** | **10000+** | |

**仿真评估指标：**

| 指标 | 权重 | 说明 |
|------|------|------|
| 碰撞率 | 50% | 任何碰撞 → 严重问题 |
| 违规率 | 20% | 压线、闯红灯等 |
| 舒适度 | 15% | 急刹车/急转弯频率 |
| 完成率 | 15% | 能否到达目的地 |

### OTA 安全部署：灰度策略

```python
class OTADeployer:
    def deploy(self, model_version, target_fleet):
        """灰度 OTA 部署"""
        
        # 第一阶段：内部测试车队（10辆）
        phase1 = target_fleet[:10]
        self.push_model(model_version, phase1)
        self.observe(phase1, duration=timedelta(days=7))
        
        # 第二阶段：扩大到 100 辆（需第一阶段通过）
        phase2 = target_fleet[10:110]
        self.push_model(model_version, phase2)
        self.observe(phase2, duration=timedelta(days=3))
        
        # 第三阶段：全量部署（需第二阶段通过）
        self.push_model(model_version, target_fleet)
        
        # 任何阶段发现问题 → 自动回滚到基线模型
        # 所有车辆同时回滚（不允许新旧模型混跑）

    def observe(self, vehicles, duration):
        """观察部署后的表现"""
        start_time = now()
        
        while now() - start_time < duration:
            for vehicle in vehicles:
                metrics = self.get_vehicle_metrics(vehicle)
                
                # 检查关键指标
                if metrics["collision_count"] > 0:
                    self.rollback_all(vehicles)
                    self.alert("发现碰撞，立即回滚")
                    return
                
                if metrics["disengage_rate"] > baseline * 2:
                    self.alert("接管率异常升高，考虑回滚")
            
            sleep(3600)  # 每小时检查一次
```

### 数据闭环自动化流水线

#### 批处理版本（原设计）

```python
class DataPipeline:
    """数据闭环的自动化流水线（批处理版本）"""

    def run_daily(self):
        """每日执行一次闭环"""

        # 1. 长尾挖掘：从昨天的新数据中找出长尾场景
        corner_cases = self.miner.mine_from_yesterday()

        # 2. 自动标注
        labeled_frames = []
        for case in corner_cases:
            auto_result = self.auto_labeler.auto_label(case)
            # 70% 自动通过，30% 送人工审核
            labeled_frames.extend(auto_result["auto_labels"])
            self.enqueue_review(auto_result["needs_review"] + auto_result["needs_full_manual"])

        # 3. 训练：将标注数据加入训练集，增量训练模型
        self.trainer.incremental_train(labeled_frames)
        new_model = self.trainer.get_latest_model()

        # 4. 仿真验证
        validation_result = self.simulator.validate(new_model, self.baseline_model)

        if validation_result["decision"] == "approve":
            # 5. 灰度部署
            self.deployer.deploy(new_model, self.fleet)
        else:
            # 验证失败 → 不部署 → 记录退化场景 → 下次训练重点关注
            self.trainer.add_priority_scenarios(validation_result["degradation_scenarios"])

        return {
            "corner_cases_found": len(corner_cases),
            "frames_labeled": len(labeled_frames),
            "validation": validation_result,
            "pipeline_status": "completed"
        }
```

#### 实时流处理版本（Kafka + Flink）

批处理的问题在于闭环周期长（1天粒度），关键长尾场景需要等待 24 小时才能进入标注流程。实时流处理架构将闭环从"日级别"缩短到"小时级别"。

**架构总览：**

```
车端上传 → Kafka(uploaded-frames) → Flink(长尾挖掘)
                                          ↓ (高分数帧)
                                    Kafka(corner-cases)
                                          ↓
                                    Flink(自动标注) → Kafka(label-tasks)
                                          ↓                    ↓
                                    Kafka(labeled-frames)  标注平台(人工审核)
                                          ↓                    ↓
                                    训练触发器 ←─────── Kafka(review-completed)
                                          ↓
                                    GPU集群(增量训练)
                                          ↓
                                    Kafka(model-ready)
                                          ↓
                                    仿真验证引擎
                                          ↓
                                    Kafka(sim-result)
                                          ↓
                                    OTA部署管理器
```

**Kafka Topic 设计：**

| Topic | 分区数 | 保留时间 | 说明 |
|-------|--------|---------|------|
| uploaded-frames | 32 | 7天 | 车端上传的帧元数据 |
| corner-cases | 8 | 30天 | 长尾挖掘命中的场景 |
| label-tasks | 8 | 30天 | 待标注任务 |
| labeled-frames | 16 | 30天 | 已标注帧数据 |
| review-completed | 4 | 7天 | 人工审核完成事件 |
| model-ready | 4 | 90天 | 训练完成的新模型通知 |
| sim-result | 4 | 90天 | 仿真验证结果 |
| deploy-events | 4 | 90天 | OTA 部署事件 |
| alert-events | 4 | 7天 | 异常告警事件 |

**流处理实现：**

```python
from kafka import KafkaProducer, KafkaConsumer
from kafka.admin import KafkaAdminClient, NewTopic
import json
import hashlib
from datetime import datetime
from confluent_kafka import Consumer, Producer
from dataclasses import dataclass, asdict


# ==================== Kafka 配置 ====================

KAFKA_BROKERS = ["kafka-1:9092", "kafka-2:9092", "kafka-3:9092"]
CONSUMER_GROUP = "data-loop"


def create_topics():
    """初始化 Kafka Topics"""
    admin = KafkaAdminClient(bootstrap_servers=KAFKA_BROKERS)
    topics = [
        NewTopic(name="uploaded-frames",  num_partitions=32, replication_factor=3),
        NewTopic(name="corner-cases",     num_partitions=8,  replication_factor=3),
        NewTopic(name="label-tasks",      num_partitions=8,  replication_factor=3),
        NewTopic(name="labeled-frames",   num_partitions=16, replication_factor=3),
        NewTopic(name="review-completed", num_partitions=4,  replication_factor=3),
        NewTopic(name="model-ready",      num_partitions=4,  replication_factor=3),
        NewTopic(name="sim-result",       num_partitions=4,  replication_factor=3),
        NewTopic(name="deploy-events",    num_partitions=4,  replication_factor=3),
        NewTopic(name="alert-events",     num_partitions=4,  replication_factor=3),
    ]
    admin.create_topics(topics)


# ==================== 上传接入服务 ====================

class FrameIngestionService:
    """接收车端上传，写入 Kafka"""

    def __init__(self):
        self.producer = Producer({
            "bootstrap.servers": ",".join(KAFKA_BROKERS),
            "acks": "all",                    # 确保所有副本写入
            "compression.type": "lz4",        # 压缩降低带宽
            "linger.ms": 50,                  # 微批，提高吞吐
            "batch.size": 65536,
        })
        self.miner = CornerCaseMiner()  # 复用前文的挖掘器

    def on_upload(self, frame_metadata: dict):
        """车端上传回调"""
        # 1. 写入 uploaded-frames topic（全量记录，用于审计和重放）
        frame_key = frame_metadata["frame_id"].encode("utf-8")
        self.producer.produce(
            topic="uploaded-frames",
            key=frame_key,
            value=json.dumps(frame_metadata).encode("utf-8"),
        )

        # 2. 实时长尾挖掘
        score = self.miner.mine(frame_metadata)
        if score > 0.3:
            corner_event = {
                **frame_metadata,
                "fused_score": score,
                "detected_at": datetime.utcnow().isoformat(),
                "source": "stream_mining",
            }
            self.producer.produce(
                topic="corner-cases",
                key=frame_key,
                value=json.dumps(corner_event).encode("utf-8"),
            )

        self.producer.flush()


# ==================== 标注分发服务 ====================

class LabelingDispatchService:
    """消费 corner-cases，分发到标注流程"""

    def __init__(self):
        self.consumer = Consumer({
            "bootstrap.servers": ",".join(KAFKA_BROKERS),
            "group.id": f"{CONSUMER_GROUP}-labeling",
            "auto.offset.reset": "earliest",
            "enable.auto.commit": False,  # 手动提交，确保不丢消息
            "max.poll.records": 100,
        })
        self.producer = Producer({
            "bootstrap.servers": ",".join(KAFKA_BROKERS),
            "acks": "all",
            "compression.type": "lz4",
        })
        self.auto_labeler = AutoLabeler()

    def run(self):
        """持续消费长尾场景，执行自动标注并分发"""
        self.consumer.subscribe(["corner-cases"])

        while True:
            messages = self.consumer.consume(num_messages=50, timeout=1.0)
            for msg in messages:
                if msg is None:
                    continue

                try:
                    case = json.loads(msg.value().decode("utf-8"))
                    self._process_case(case)
                    self.consumer.commit(asynchronous=False)
                except Exception as e:
                    # 处理失败 → 发送到死信队列
                    self._send_to_dlq(msg, str(e))
                    logging.error(f"标注分发失败: {e}")

    def _process_case(self, case: dict):
        """处理单个长尾场景"""
        # 自动标注
        auto_result = self.auto_labeler.auto_label(case)

        # 高置信度 → 直接写入已标注队列
        for label in auto_result["auto_labels"]:
            labeled_event = {
                "frame_id": case["frame_id"],
                "label": label,
                "source": "auto_approved",
                "confidence": label.get("confidence", 0),
                "labeled_at": datetime.utcnow().isoformat(),
            }
            self.producer.produce(
                topic="labeled-frames",
                key=case["frame_id"].encode("utf-8"),
                value=json.dumps(labeled_event).encode("utf-8"),
            )

        # 低置信度 → 发送到人工标注队列
        needs_human = auto_result["needs_review"] + auto_result["needs_full_manual"]
        if needs_human:
            task = {
                "frame_id": case["frame_id"],
                "items": needs_human,
                "priority": "high" if any(i["confidence"] < 0.5 for i in needs_human) else "normal",
                "created_at": datetime.utcnow().isoformat(),
            }
            self.producer.produce(
                topic="label-tasks",
                key=case["frame_id"].encode("utf-8"),
                value=json.dumps(task).encode("utf-8"),
            )


# ==================== 训练触发器 ====================

class TrainingTrigger:
    """监听已标注数据，达到阈值触发增量训练"""

    def __init__(self, trigger_threshold: int = 500):
        """
        Args:
            trigger_threshold: 累积多少新标注帧触发一次训练
        """
        self.consumer = Consumer({
            "bootstrap.servers": ",".join(KAFKA_BROKERS),
            "group.id": f"{CONSUMER_GROUP}-training-trigger",
            "auto.offset.reset": "earliest",
        })
        self.producer = Producer({
            "bootstrap.servers": ",".join(KAFKA_BROKERS),
            "acks": "all",
        })
        self.trigger_threshold = trigger_threshold
        self.pending_frames = []

    def run(self):
        """消费已标注帧，累积触发训练"""
        self.consumer.subscribe(["labeled-frames", "review-completed"])

        while True:
            msg = self.consumer.poll(timeout=1.0)
            if msg is None:
                continue

            frame_data = json.loads(msg.value().decode("utf-8"))
            self.pending_frames.append(frame_data)

            if len(self.pending_frames) >= self.trigger_threshold:
                self._trigger_training()

    def _trigger_training(self):
        """触发增量训练"""
        training_event = {
            "trigger_time": datetime.utcnow().isoformat(),
            "new_frame_count": len(self.pending_frames),
            "frame_ids": [f["frame_id"] for f in self.pending_frames],
            "base_model_version": self._get_current_model_version(),
        }
        self.producer.produce(
            topic="model-ready",  # 训练完成后会由训练服务写入此 topic
            key=b"training-trigger",
            value=json.dumps(training_event).encode("utf-8"),
        )
        self.pending_frames = []
        logging.info(f"触发增量训练: {training_event['new_frame_count']} 帧")


# ==================== 仿真 + 部署服务 ====================

class SimAndDeployService:
    """消费训练完成事件，执行仿真验证和灰度部署"""

    def __init__(self):
        self.consumer = Consumer({
            "bootstrap.servers": ",".join(KAFKA_BROKERS),
            "group.id": f"{CONSUMER_GROUP}-sim-deploy",
            "auto.offset.reset": "earliest",
        })
        self.producer = Producer({
            "bootstrap.servers": ",".join(KAFKA_BROKERS),
            "acks": "all",
        })
        self.simulator = SimulationValidator()
        self.deployer = OTADeployer()

    def run(self):
        self.consumer.subscribe(["model-ready"])

        while True:
            msg = self.consumer.poll(timeout=1.0)
            if msg is None:
                continue

            event = json.loads(msg.value().decode("utf-8"))
            new_model = self._load_model(event["version_id"])

            # 仿真验证
            sim_result = self.simulator.validate_model(new_model, self.baseline_model)

            sim_event = {
                "version_id": event["version_id"],
                "decision": sim_result["decision"],
                "improvements": sim_result.get("improvements", 0),
                "degradation_scenarios": sim_result.get("degradation_scenarios", []),
                "simulated_at": datetime.utcnow().isoformat(),
            }
            self.producer.produce(
                topic="sim-result",
                key=event["version_id"].encode("utf-8"),
                value=json.dumps(sim_event).encode("utf-8"),
            )

            if sim_result["decision"] == "approve":
                # 灰度部署
                self.deployer.deploy(new_model, self.fleet)
            else:
                # 告警
                self.producer.produce(
                    topic="alert-events",
                    key=b"sim-failed",
                    value=json.dumps({
                        "alert_type": "simulation_failed",
                        "version_id": event["version_id"],
                        "reason": sim_result["reason"],
                    }).encode("utf-8"),
                )

            self.consumer.commit(asynchronous=False)
```

**批处理 vs 流处理对比：**

| 维度 | 日批处理 | Kafka 实时流处理 |
|------|---------|-----------------|
| 闭环延迟 | 24-48 小时 | 2-6 小时 |
| 长尾发现延迟 | 次日才能挖掘 | 上传后秒级挖掘 |
| 标注启动延迟 | 次日批量分发 | 发现后立即分发 |
| 训练触发 | 固定每日一次 | 累积 500 帧即触发 |
| 故障恢复 | 重跑整个批处理 | 从 Kafka offset 重放 |
| 扩展性 | 单机受限 | Kafka 分区水平扩展 |
| 运维复杂度 | 低 | 中（需维护 Kafka 集群） |

**Kafka 集群规模估算：**

- 日均消息量：约 100 万条（1000 辆车 × 1000 帧/天 × 边缘过滤后上传率 0.1%）
- 峰值吞吐：约 5000 msg/s（早晚高峰集中上传）
- 存储：7 天保留 × 100 万条/天 × 平均 2KB/条 = 14GB
- 集群配置：3 Broker + 3 ZooKeeper，每节点 16 核 64GB 内存

### 增量训练实现

增量训练是数据闭环的关键环节——不是每次都从头训练，而是在现有模型基础上用新标注数据微调，大幅缩短训练时间。

```python
import torch
import torch.nn as nn
from torch.utils.data import Dataset, DataLoader
from pathlib import Path
from typing import List, Optional
import hashlib
import json
import logging


class DrivingDataset(Dataset):
    """自动驾驶多模态数据集"""

    def __init__(self, data_dir: str, frame_ids: Optional[List[str]] = None):
        self.data_dir = Path(data_dir)
        if frame_ids:
            self.frame_ids = frame_ids
        else:
            # 加载全部已标注帧
            self.frame_ids = self._load_all_frame_ids()

        # 加载类别映射
        self.class_map = self._load_class_map()

    def __len__(self):
        return len(self.frame_ids)

    def __getitem__(self, idx):
        frame_id = self.frame_ids[idx]
        frame_dir = self.data_dir / frame_id

        # 加载多模态数据
        camera_tensors = self._load_cameras(frame_dir)
        lidar_tensor = self._load_lidar(frame_dir)
        labels = self._load_labels(frame_dir)

        return {
            "cameras": camera_tensors,       # [6, 3, H, W]
            "lidar": lidar_tensor,            # [N, 5] (x,y,z,intensity,ring)
            "labels": labels,                 # dict with 3d_boxes, seg_mask
            "frame_id": frame_id,
        }

    def _load_cameras(self, frame_dir: Path):
        """加载6路摄像头图像，做归一化和数据增强"""
        tensors = []
        for cam_id in range(6):
            img_path = frame_dir / f"cam_{cam_id}.jpg"
            img = self._read_and_augment(img_path)
            tensors.append(img)
        return torch.stack(tensors)

    def _load_lidar(self, frame_dir: Path):
        """加载点云，做体素化"""
        pcd_path = frame_dir / "lidar.bin"
        points = torch.from_numpy(np.fromfile(str(pcd_path), dtype=np.float32))
        points = points.reshape(-1, 5)  # x,y,z,intensity,ring_index
        # 体素化
        voxels = self._voxelize(points, voxel_size=0.1)
        return voxels

    def _load_labels(self, frame_dir: Path):
        """加载标注文件"""
        label_path = frame_dir / "labels.json"
        with open(label_path) as f:
            return json.load(f)

    def _voxelize(self, points, voxel_size):
        """点云体素化，返回稀疏体素张量"""
        # 量化坐标
        voxel_coords = (points[:, :3] / voxel_size).long()
        # 去重（同一体素内只保留一个点）
        unique_coords, inverse = torch.unique(voxel_coords, dim=0, return_inverse=True)
        # 取每个体素内点的平均值
        voxel_features = torch.zeros(len(unique_coords), points.shape[1])
        voxel_features.scatter_reduce_(0, inverse.unsqueeze(1).expand_as(points), points, reduce="mean")
        return {"coords": unique_coords, "features": voxel_features}


class IncrementalTrainer:
    """增量训练管理器：在新标注数据上微调现有模型"""

    def __init__(
        self,
        base_model_path: str,
        output_dir: str,
        gpu_ids: List[int] = None,
    ):
        self.base_model_path = base_model_path
        self.output_dir = Path(output_dir)
        self.output_dir.mkdir(parents=True, exist_ok=True)
        self.gpu_ids = gpu_ids or list(range(torch.cuda.device_count()))

        # 训练历史：记录每个版本的训练数据指纹，防止重复训练
        self.training_history = self._load_training_history()

    def incremental_train(
        self,
        new_frame_ids: List[str],
        epochs: int = 5,
        learning_rate: float = 1e-4,
        warmup_steps: int = 100,
        weight_decay: float = 0.01,
        replay_ratio: float = 0.3,
    ) -> str:
        """
        增量训练主流程

        Args:
            new_frame_ids: 新标注帧ID列表
            epochs: 微调轮数（比全量训练少，全量约 50-100 epoch）
            learning_rate: 学习率（比全量训练小 10 倍，避免灾难性遗忘）
            warmup_steps: 学习率预热步数
            weight_decay: 权重衰减
            replay_ratio: 旧数据回放比例（防止灾难性遗忘）

        Returns:
            新模型版本号
        """
        # 1. 计算训练数据指纹，检查是否已训练过
        data_fingerprint = self._compute_fingerprint(new_frame_ids)
        if data_fingerprint in self.training_history:
            logging.warning(f"训练数据无变化，跳过: {data_fingerprint[:8]}")
            return self.training_history[data_fingerprint]

        # 2. 加载基线模型
        model = self._load_base_model()
        logging.info(f"加载基线模型: {self.base_model_path}")

        # 3. 构建混合数据集（新数据 + 旧数据回放）
        new_dataset = DrivingDataset(data_dir=self.data_dir, frame_ids=new_frame_ids)
        replay_dataset = self._build_replay_dataset(replay_ratio, len(new_frame_ids))

        # 混合采样：70% 新数据 + 30% 旧数据
        mixed_dataset = torch.utils.data.ConcatDataset([new_dataset, replay_dataset])
        mixed_loader = DataLoader(
            mixed_dataset,
            batch_size=4,
            shuffle=True,
            num_workers=8,
            pin_memory=True,
            collate_fn=self._collate_fn,
        )

        # 4. 配置训练：小学习率 + 梯度裁剪 + EMA
        optimizer = torch.optim.AdamW(
            model.parameters(),
            lr=learning_rate,
            weight_decay=weight_decay,
        )
        scheduler = self._build_scheduler(optimizer, warmup_steps, len(mixed_loader) * epochs)
        scaler = torch.cuda.amp.GradScaler()  # 混合精度训练

        # 5. 训练循环
        best_loss = float("inf")
        model.train()

        for epoch in range(epochs):
            epoch_loss = 0
            num_batches = 0

            for batch_idx, batch in enumerate(mixed_loader):
                batch = self._move_to_gpu(batch)

                optimizer.zero_grad()

                with torch.cuda.amp.autocast():
                    loss_dict = model(batch)
                    loss = loss_dict["total_loss"]

                scaler.scale(loss).backward()
                # 梯度裁剪（防止增量训练的梯度爆炸）
                scaler.unscale_(optimizer)
                torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=5.0)
                scaler.step(optimizer)
                scaler.update()
                scheduler.step()

                epoch_loss += loss.item()
                num_batches += 1

                if batch_idx % 50 == 0:
                    logging.info(
                        f"Epoch {epoch+1}/{epochs}, Batch {batch_idx}/{len(mixed_loader)}, "
                        f"Loss: {loss.item():.4f}, LR: {scheduler.get_last_lr()[0]:.2e}"
                    )

            avg_loss = epoch_loss / max(num_batches, 1)
            logging.info(f"Epoch {epoch+1} 完成, 平均 Loss: {avg_loss:.4f}")

            # 保存最优检查点
            if avg_loss < best_loss:
                best_loss = avg_loss
                self._save_checkpoint(model, epoch, avg_loss, is_best=True)

        # 6. 生成新模型版本
        new_version = self._generate_version()
        model_path = self.output_dir / f"{new_version}.pt"
        torch.save(model.state_dict(), model_path)

        # 7. 记录训练历史
        self.training_history[data_fingerprint] = new_version
        self._save_training_history()

        # 8. 记录到数据库
        self._record_model_version(new_version, new_frame_ids, best_loss, epochs)

        logging.info(f"增量训练完成: version={new_version}, best_loss={best_loss:.4f}")
        return new_version

    def _build_replay_dataset(self, replay_ratio: float, new_data_size: int) -> Dataset:
        """构建旧数据回放集，防止灾难性遗忘"""
        # 从历史训练数据中按比例采样旧数据
        # 优先采样模型表现差的场景（困难样本挖掘）
        replay_size = int(new_data_size * replay_ratio)

        # 查询训练集中模型损失最高的帧
        hard_frames = db.query("""
            SELECT frame_id, AVG(loss) as avg_loss
            FROM training_loss_log
            WHERE model_version = %s
            GROUP BY frame_id
            ORDER BY avg_loss DESC
            LIMIT %s
        """, (self._get_current_model_version(), replay_size))

        replay_frame_ids = [row["frame_id"] for row in hard_frames]
        return DrivingDataset(data_dir=self.data_dir, frame_ids=replay_frame_ids)

    def _build_scheduler(self, optimizer, warmup_steps, total_steps):
        """余弦退火 + 线性预热"""
        from torch.optim.lr_scheduler import LambdaLR
        import math

        def lr_lambda(step):
            if step < warmup_steps:
                return step / max(warmup_steps, 1)
            progress = (step - warmup_steps) / max(total_steps - warmup_steps, 1)
            return 0.5 * (1 + math.cos(math.pi * progress))

        return LambdaLR(optimizer, lr_lambda)

    def _compute_fingerprint(self, frame_ids: List[str]) -> str:
        """计算训练数据指纹，用于去重"""
        sorted_ids = sorted(frame_ids)
        raw = "|".join(sorted_ids)
        return hashlib.sha256(raw.encode()).hexdigest()

    def _generate_version(self) -> str:
        """生成语义化版本号"""
        current = self._get_current_model_version()  # 如 "v2.3.7"
        parts = current.lstrip("v").split(".")
        parts[-1] = str(int(parts[-1]) + 1)  # 递增 patch 版本
        return f"v{'.'.join(parts)}"

    def _record_model_version(self, version: str, frame_ids: List[str],
                              best_loss: float, epochs: int):
        """记录到 model_versions 表"""
        gpu_hours = len(self.gpu_ids) * epochs * 0.5  # 估算
        db.insert("""
            INSERT INTO model_versions
            (version_id, base_version, training_config, training_data_size,
             training_duration_h, gpu_hours, validation_pass, deploy_status)
            VALUES (%s, %s, %s, %s, %s, %s, FALSE, 'pending')
        """, (
            version,
            self._get_current_model_version(),
            json.dumps({"epochs": epochs, "lr": 1e-4, "replay_ratio": 0.3}),
            len(frame_ids),
            epochs * 0.5,
            gpu_hours,
        ))


# ==================== 全量训练 vs 增量训练对比 ====================

class FullTrainer:
    """全量训练：从预训练权重开始，用全部数据训练"""

    def full_train(self, all_frame_ids: List[str], epochs: int = 80):
        """
        全量训练：耗时约 3-7 天
        - 数据量：全部已标注数据（约 50 万帧）
        - 学习率：1e-3（比增量训练大 10 倍）
        - 不需要回放（本身就是全量数据）
        """
        pass  # 实现省略，结构与 IncrementalTrainer 类似


# 对比总结：
# | 维度         | 全量训练         | 增量训练           |
# |-------------|-----------------|-------------------|
# | 训练时间     | 3-7 天          | 2-4 小时          |
# | 数据量       | 50 万帧（全量）  | 500-5000 帧（增量）|
# | GPU 消耗     | 576 GPU·h       | 8-16 GPU·h        |
# | 学习率       | 1e-3            | 1e-4              |
# | 灾难性遗忘   | 无               | 需回放旧数据防范   |
# | 触发频率     | 每月 1 次        | 每周 2-3 次       |
# | 模型提升幅度 | 大版本提升       | 小幅迭代改进       |
```

## 常见陷阱（深度分析）

### 陷阱 1：全量标注

**后果：** 1PB/天全量标注 → 日成本 ¥500 万 → 不可承受。即使只标注关键帧（每天约 1 万帧），也需要 ¥5 万/天。

**解决方案：** 长尾挖掘 + 自动标注 + 人工审核，只标注 < 0.01% 的有价值数据。

### 陷阱 2：不做长尾挖掘，随机采样训练数据

**后果：** 随机采样 → 99.99% 是常规场景 → 模型对常规场景表现好，但对长尾场景（行人突然冲出）表现差 → 安全事故。

**量化：** 不做长尾挖掘时，模型在长尾场景上的碰撞率比挖掘后高 3-5 倍。

**解决方案：** 用不确定性信号主动挖掘长尾场景，确保训练数据覆盖长尾分布。

### 陷阱 3：模型直接上车不做仿真验证

**后果：** 新模型可能在某些场景上比基线差 → 部署后可能导致安全事故。特别是"新模型整体提升但在夜间雨天场景退化"这种隐蔽退化。

**解决方案：** 场景库仿真验证 + 灰度 OTA 部署。仿真覆盖 10000+ 场景。

### 陷阱 4：仿真只覆盖法规场景

**后果：** 法规场景（如 AEB 测试）是有限的、标准化的。但真实世界的长尾场景是无限的。只覆盖法规场景 → 漏测大量真实长尾 → 安全隐患。

**解决方案：** 场景库 = 路测长尾 + 法规合规 + 对抗场景 + 随机仿真，覆盖 10000+ 场景。

### 陷阱 5：OTA 不做灰度部署

**后果：** 全量部署 → 发现问题时所有车辆都受影响 → 需要全量回滚 → 部署周期延长。

**解决方案：** 灰度部署（10辆→100辆→全量），任何阶段发现问题立即回滚。

## 性能与成本分析

### 存储成本：1PB/天的数据怎么办？

1000 辆车每天产生 1PB 原始数据，不可能全部长期存储。核心策略是"边缘过滤 + 分层存储 + 定期归档"。

**存储成本月度估算：**

| 存储层级 | 数据量 | 保留天数 | 月存储量 | 单价(¥/TB/月) | 月成本 |
|---------|-------|---------|---------|-------------|-------|
| 车端本地 SSD | 1PB/天 | 3天 | 3PB | 800 | ¥240 万 |
| 对象存储(标准) | 1GB/天(过滤后) | 30天 | 30GB | 230 | ¥0.7 万 |
| 对象存储(低频) | 1GB/天 | 180天 | 180GB | 125 | ¥2.3 万 |
| 对象存储(归档) | 合规数据 | 10年 | ~3.6TB | 45 | ¥16 万 |
| **合计** | | | | | **¥259 万/月** |

**关键发现：** 存储成本的大头是车端本地 SSD（3PB × ¥800/TB/月 = ¥240 万/月）。车端必须严格控制在 3 天覆盖——超过 3 天的数据要么上传到云端，要么丢弃。

**如果没有边缘过滤（1PB 全量上传到对象存储）：**

| 场景 | 月存储量 | 月成本 | 可行性 |
|------|---------|-------|-------|
| 无过滤，标准存储 7天 | 7PB | ¥161 万 | 不可行（带宽也撑不住） |
| 无过滤，标准存储 30天 | 30PB | ¥690 万 | 完全不可行 |
| 边缘过滤后，标准存储 30天 | 30GB | ¥0.7 万 | 可行 |

**结论：边缘过滤是存储可行性的前提。** 没有边缘过滤，1PB/天的数据存储成本是天文数字。

### GPU 训练成本

| 训练类型 | 频率 | GPU 规格 | 数量 | 单次时长 | 单次 GPU·h | 月频次 | 月 GPU·h |
|---------|------|---------|------|---------|-----------|-------|---------|
| 增量训练 | 每周 2 次 | A100 80G | 8卡 | 3 小时 | 24 | 8 | 192 |
| 全量训练 | 每月 1 次 | A100 80G | 8卡 | 120 小时 | 960 | 1 | 960 |
| 仿真验证 | 每周 2 次 | A10G | 4卡 | 2 小时 | 8 | 8 | 64 |
| 自动标注推理 | 每日 | A10G | 2卡 | 4 小时 | 8 | 30 | 240 |
| **合计** | | | | | | | **1456 GPU·h/月** |

**GPU 租赁成本（按需）：**
- A100 80G：¥35/GPU·h（阿里云/腾讯云）
- A10G：¥12/GPU·h
- 增量训练：192 × ¥35 = ¥6,720/月
- 全量训练：960 × ¥35 = ¥33,600/月
- 仿真验证：64 × ¥12 = ¥768/月
- 自动标注：240 × ¥12 = ¥2,880/月
- **合计：¥43,968/月（约 ¥4.4 万/月）**

**优化建议：** 全量训练使用抢占式实例（价格约 1/5），月成本可降至 ¥2 万以内。

### 标注成本详细分解

| 标注类型 | 帧数/天 | 单价 | 日成本 | 说明 |
|---------|--------|------|-------|------|
| 3D 边界框（自动通过） | ~700 | ¥0 | ¥0 | 置信度 > 0.95，自动标注直接使用 |
| 3D 边界框（人工修正） | ~200 | ¥3/帧 | ¥600 | 自动标注基础上微调，工作量小 |
| 3D 边界框（完全人工） | ~100 | ¥5/帧 | ¥500 | 置信度 < 0.5，从零标注 |
| 语义分割（人工） | ~50 | ¥20/帧 | ¥1,000 | 车道线、可行驶区域等 |
| 人工审核质检 | ~350 | ¥1/帧 | ¥350 | 抽检 50% 自动标注结果 |
| **合计** | **~1400** | | **¥2,450/天** | |

**月度标注成本：¥2,450 × 30 = ¥73,500/月（约 ¥7.4 万/月）**

**与无自动标注的对比：**

| 方案 | 日成本 | 月成本 | 年成本 |
|------|-------|-------|-------|
| 全量人工（1万帧/天） | ¥5 万 | ¥150 万 | ¥1800 万 |
| 自动标注+人工审核 | ¥0.25 万 | ¥7.4 万 | ¥89 万 |
| **节省** | **¥4.75 万** | **¥142.6 万** | **¥1711 万** |

### 总成本汇总

| 成本项 | 月成本 | 占比 |
|-------|-------|------|
| 存储（含车端 SSD） | ¥259 万 | 84.6% |
| GPU 训练 | ¥4.4 万 | 1.4% |
| 标注 | ¥7.4 万 | 2.4% |
| Kafka 集群（3 节点） | ¥3 万 | 1.0% |
| 仿真平台 | ¥2 万 | 0.7% |
| 网络带宽（上传 1GB/天 × 1000 车） | ¥30 万 | 9.8% |
| **合计** | **¥305.8 万/月** | |

**关键洞察：** 存储成本（车端 SSD）占总成本 84.6%。降本的关键不在 GPU 或标注，而在：
1. 缩短车端本地数据保留时间（3天→1天，需提升上传带宽）
2. 更激进的边缘过滤（0.01%→0.005%，需更精准的筛选模型）
3. 更高的压缩比（12:1→20:1，需更好的点云压缩算法）

## 异常场景深度分析

数据闭环系统在正常运行时表现良好，但异常场景才是区分"能用"和"可靠"的关键。以下是 4 个核心异常场景的详细分析。

### 异常 1：边缘过滤假阴性（漏掉真正的长尾场景）

**场景描述：** 车端边缘过滤的轻量级模型误判某个关键长尾场景为"普通场景"，未上传到云端。例如，夜间雨天 + 行人穿深色衣服，车端感知模型给出高置信度（因为置信度计算有 bug），边缘过滤将其丢弃。

**影响：** 这个长尾场景永远不会进入训练集，模型在该类场景上的缺陷永远无法修复——形成"盲区固化"。

**检测机制：**

```python
class EdgeFilterFalseNegativeDetector:
    """检测边缘过滤假阴性：云端定期抽样验证"""

    def __init__(self):
        self.cloud_miner = CornerCaseMiner()  # 云端完整版挖掘器
        self.sampling_rate = 0.001  # 对被丢弃的帧做 0.1% 抽样验证

    def audit_discarded_frames(self, vehicle_id: str, date: str):
        """抽样审查被边缘过滤丢弃的帧"""
        # 1. 从车端本地缓冲中获取被丢弃帧的元数据（CAN 信号 + 感知输出）
        discarded = self._fetch_discarded_metadata(vehicle_id, date)

        # 2. 云端完整模型重新打分
        false_negatives = []
        for frame_meta in discarded:
            # 用 0.1% 概率抽样
            if random.random() > self.sampling_rate:
                continue

            cloud_score = self.cloud_miner.mine(frame_meta)
            edge_score = frame_meta.get("edge_score", 0)

            # 云端判定为长尾，但边缘丢弃了 → 假阴性
            if cloud_score > 0.6 and edge_score <= 0.3:
                false_negatives.append({
                    "frame_id": frame_meta["frame_id"],
                    "cloud_score": cloud_score,
                    "edge_score": edge_score,
                    "gap": cloud_score - edge_score,
                })

        # 3. 量化假阴性率
        if false_negatives:
            fn_rate = len(false_negatives) / (len(discarded) * self.sampling_rate)
            self.alert(
                f"边缘过滤假阴性率: {fn_rate:.4f}，"
                f"发现 {len(false_negatives)} 个漏检长尾场景",
                severity="HIGH"
            )

            # 4. 请求车端重传这些帧（走 WiFi 场景下高优先级通道）
            for fn in false_negatives:
                self._request_retransmit(fn["frame_id"], vehicle_id)

        return false_negatives

    def _request_retransmit(self, frame_id: str, vehicle_id: str):
        """请求车辆重传被丢弃的帧"""
        # 下发重传指令到车端（通过控制通道）
        command = {
            "type": "retransmit",
            "frame_id": frame_id,
            "priority": "critical",  # 最高优先级
            "reason": "false_negative_detected",
        }
        self.mqtt_client.publish(f"cmd/{vehicle_id}", json.dumps(command))
```

**防御策略：**

1. **定期审计：** 每天对 0.1% 被丢弃帧做云端完整打分，量化假阴性率
2. **关键场景强制上传：** CAN 信号出现急刹车/气囊弹出等极端事件时，无条件上传（绕过边缘过滤）
3. **双阈值策略：** 边缘过滤使用宽松阈值（0.3），只丢弃 score < 0.3 的帧；0.3-0.6 之间的帧低优先级上传

### 异常 2：自动标注漂移（Auto-labeling Drift）

**场景描述：** 自动标注模型自身也存在偏差。当新出现的长尾场景恰好是自动标注模型不擅长的类型时，自动标注会产生系统性错误（不是随机错误，而是方向一致的偏差）。例如，施工区域的新类型路障被自动标注为"行人"（因为形状最接近已知类别），人工审核如果也疏忽，这个错误标注就进入训练集，导致下一代模型同样将此类路障误认为行人——形成正反馈漂移。

**影响：** 标注偏差在训练中放大 → 模型偏差加剧 → 自动标注偏差更大 → 恶性循环。

**检测机制：**

```python
class AutoLabelDriftDetector:
    """检测自动标注漂移"""

    def __init__(self, window_days: int = 7):
        self.window_days = window_days
        # 人工审核的"黄金集"——用于校准自动标注质量
        self.golden_set_size = 100  # 每周随机抽 100 帧做全人工标注

    def detect_drift(self):
        """检测自动标注是否发生漂移"""
        # 1. 对比自动标注 vs 人工审核在黄金集上的差异
        auto_labels = self._get_auto_labels_for_golden_set()
        human_labels = self._get_human_labels_for_golden_set()

        # 2. 计算类别级别的偏差
        category_errors = self._compute_category_errors(auto_labels, human_labels)

        # 3. 检查系统性偏差（不是随机错误，而是某个类别持续被误标）
        drift_detected = False
        for category, error_rate in category_errors.items():
            if error_rate > 0.15:  # 某类别误标率 > 15%
                # 检查趋势：过去 3 周是否持续恶化
                trend = self._get_error_trend(category, weeks=3)
                if trend == "worsening":
                    drift_detected = True
                    self.alert(
                        f"自动标注漂移告警: 类别 {category} 误标率 {error_rate:.1%}，"
                        f"过去 3 周趋势: {trend}",
                        severity="CRITICAL"
                    )

        # 4. 如果检测到漂移，自动降级自动标注
        if drift_detected:
            self._degrade_auto_labeling()

        return drift_detected

    def _degrade_auto_labeling(self):
        """漂移时自动降级：降低自动通过阈值，增加人工审核比例"""
        # 将自动通过的置信度阈值从 0.95 提高到 0.99
        # 更多帧进入人工审核流程
        config = {
            "auto_approve_threshold": 0.99,  # 原来是 0.95
            "review_sampling_rate": 0.5,      # 抽检 50%（原来 20%）
            "golden_set_size": 200,            # 黄金集翻倍
        }
        self.update_auto_labeler_config(config)
        logging.warning("自动标注漂移检测，已降级配置：提高人工审核比例")

    def _compute_category_errors(self, auto_labels, human_labels):
        """计算每个类别的误标率"""
        errors = {}
        for frame_id in auto_labels:
            auto = auto_labels[frame_id]
            human = human_labels[frame_id]

            for obj_auto in auto["objects"]:
                # 找到人工标注中匹配的对象
                matched_human = self._find_matching_object(obj_auto, human["objects"])
                if matched_human:
                    if obj_auto["category"] != matched_human["category"]:
                        cat = obj_auto["category"]
                        errors[cat] = errors.get(cat, 0) + 1

        total_by_category = Counter(obj["category"] for obj in auto_labels.values())
        error_rates = {cat: errors.get(cat, 0) / total_by_category.get(cat, 1)
                       for cat in total_by_category}
        return error_rates
```

**防御策略：**

1. **黄金集校准：** 每周 100 帧全人工标注，作为自动标注质量的"真值参照"
2. **漂移检测：** 追踪每个类别的误标率趋势，发现持续恶化立即告警
3. **自动降级：** 漂移时自动提高自动通过阈值，增加人工审核比例
4. **模型回滚：** 如果漂移严重，回滚自动标注模型到上一个已知稳定版本

### 异常 3：仿真假通过（Simulation False Pass）

**场景描述：** 仿真验证通过，但新模型部署到真实道路后表现退化。原因是仿真场景库不够全面，未能覆盖某些真实场景。例如，仿真中没有"逆光 + 积水路面反光"的组合场景，新模型在该场景上严重退化，但仿真显示一切正常。

**影响：** 带有隐蔽缺陷的模型被部署到车辆 → 实际驾驶中暴露问题 → 安全风险。

**检测机制：**

```python
class SimFalsePassDetector:
    """检测仿真假通过：对比仿真结果与实车表现"""

    def __init__(self):
        self.canary_fleet_size = 10  # 金丝雀车队规模

    def validate_with_canary(self, new_model_version: str):
        """金丝雀验证：小规模实车验证仿真结论"""
        # 1. 获取仿真验证结果
        sim_result = self._get_sim_result(new_model_version)

        # 2. 部署到金丝雀车队（内部测试车队 10 辆）
        canary_vehicles = self._get_canary_fleet()
        self.ota_deploy(new_model_version, canary_vehicles)

        # 3. 收集 72 小时实车数据
        real_metrics = self._collect_real_metrics(canary_vehicles, hours=72)

        # 4. 对比仿真 vs 实车
        gaps = self._compare_sim_vs_real(sim_result, real_metrics)

        # 5. 发现显著差距 → 仿真假通过
        if gaps["max_degradation"] > 0.1:
            self.alert(
                f"仿真假通过告警: 模型 {new_model_version}，"
                f"仿真通过但实车退化 {gaps['max_degradation']:.1%}，"
                f"退化场景: {gaps['degradation_categories']}",
                severity="CRITICAL"
            )
            # 立即回滚金丝雀车队
            self.ota_rollback(canary_vehicles)
            # 将缺失场景加入仿真库
            self._add_missing_scenarios(gaps["degradation_categories"])

        return gaps

    def _compare_sim_vs_real(self, sim_result, real_metrics):
        """对比仿真与实车的性能差异"""
        gaps = {"max_degradation": 0, "degradation_categories": []}

        # 按场景类别对比
        for category in ["pedestrian", "vehicle", "construction", "weather"]:
            sim_score = sim_result.get(category, {}).get("score", 1.0)
            real_score = real_metrics.get(category, {}).get("score", 1.0)
            degradation = sim_score - real_score

            if degradation > 0.05:  # 实车比仿真差 > 5%
                gaps["degradation_categories"].append(category)
                gaps["max_degradation"] = max(gaps["max_degradation"], degradation)

        return gaps

    def _add_missing_scenarios(self, categories: list):
        """将仿真缺失的场景类型补充到场景库"""
        for category in categories:
            # 从实车数据中提取该类别的典型场景
            real_scenarios = self._extract_scenarios_from_real_data(category)
            for scenario in real_scenarios:
                self.scenario_library.add(scenario, source="canary_gap")
                logging.info(f"补充仿真场景: category={category}, scenario_id={scenario.id}")
```

**防御策略：**

1. **金丝雀验证：** 仿真通过后，先在 10 辆内部测试车运行 72 小时
2. **仿真 vs 实车差距监控：** 持续对比仿真分数与实车表现
3. **场景库动态补充：** 发现仿真覆盖不足时，自动从实车数据补充
4. **对抗性测试：** 定期构造"仿真通过但已知困难"的边界场景，测试仿真可靠性

### 异常 4：OTA 回滚失败

**场景描述：** 新模型部署后发现问题，触发回滚，但回滚操作失败。原因可能是：(1) 车端存储空间不足，旧模型已被覆盖；(2) 网络中断，回滚指令未到达车辆；(3) 部分车辆回滚成功，部分失败，导致车队中混跑新旧版本。

**影响：** 带缺陷的模型继续运行 → 安全风险；新旧模型混跑 → 行为不一致 → 车队协同出问题。

**防御机制：**

```python
class OTARollbackManager:
    """OTA 回滚安全保障"""

    def __init__(self):
        self.model_cache_dir = "/data/model_cache"
        self.max_cached_versions = 3  # 车端保留最近 3 个模型版本
        self.rollback_timeout_s = 300  # 回滚超时 5 分钟

    def safe_rollback(self, vehicle_ids: list, target_version: str):
        """安全回滚：确保所有车辆回滚成功或进入安全模式"""
        # 1. 检查车端是否缓存了目标版本
        rollback_plan = {}
        for vid in vehicle_ids:
            cached_versions = self._get_cached_versions(vid)
            if target_version in cached_versions:
                rollback_plan[vid] = "local_rollback"  # 本地缓存回滚
            else:
                rollback_plan[vid] = "download_rollback"  # 需要下载旧版本

        # 2. 并行执行回滚
        results = {}
        with ThreadPoolExecutor(max_workers=50) as executor:
            futures = {}
            for vid, method in rollback_plan.items():
                if method == "local_rollback":
                    futures[vid] = executor.submit(
                        self._local_rollback, vid, target_version
                    )
                else:
                    futures[vid] = executor.submit(
                        self._download_rollback, vid, target_version
                    )

            for vid, future in futures.items():
                try:
                    success = future.result(timeout=self.rollback_timeout_s)
                    results[vid] = "success" if success else "failed"
                except TimeoutError:
                    results[vid] = "timeout"
                except Exception as e:
                    results[vid] = f"error: {e}"

        # 3. 检查回滚结果
        failed_vehicles = [vid for vid, r in results.items() if r != "success"]

        if failed_vehicles:
            # 4. 回滚失败的车辆进入安全模式
            for vid in failed_vehicles:
                self._activate_safe_mode(vid)
                logging.error(
                    f"车辆 {vid} 回滚失败({results[vid]})，已进入安全模式: "
                    f"限速 30km/h，仅启用基础 ADAS 功能"
                )

            # 5. 告警运维团队
            self.alert_ops(
                f"OTA 回滚部分失败: {len(failed_vehicles)}/{len(vehicle_ids)} 辆车回滚失败，"
                f"已进入安全模式，需人工介入",
                severity="CRITICAL"
            )

        return results

    def _local_rollback(self, vehicle_id: str, target_version: str) -> bool:
        """本地缓存回滚：切换到已缓存的旧版本"""
        try:
            # 1. 验证缓存模型完整性（SHA256 校验）
            model_path = f"{self.model_cache_dir}/{target_version}/model.pt"
            if not self._verify_model_integrity(model_path, target_version):
                return False

            # 2. 原子切换：先写新配置到临时文件，再 rename（原子操作）
            config = {"model_version": target_version, "active_since": time.time()}
            temp_path = f"/tmp/model_config_{vehicle_id}.json"
            final_path = f"/data/config/model_config.json"

            with open(temp_path, "w") as f:
                json.dump(config, f)
            os.rename(temp_path, final_path)  # 原子操作

            # 3. 重启感知进程
            self._restart_perception(vehicle_id)

            # 4. 验证新版本生效
            current = self._get_active_version(vehicle_id)
            return current == target_version

        except Exception as e:
            logging.error(f"本地回滚失败: vehicle={vehicle_id}, error={e}")
            return False

    def _activate_safe_mode(self, vehicle_id: str):
        """激活安全模式：降级运行"""
        safe_config = {
            "mode": "safe",
            "max_speed_kmh": 30,
            "features_enabled": ["AEB", "LKA", "FCW"],  # 只保留基础 ADAS
            "features_disabled": ["auto_lane_change", "auto_park", "navigate_on_autopilot"],
            "require_driver_attention": True,
        }
        self.mqtt_client.publish(
            f"cmd/{vehicle_id}/safe_mode",
            json.dumps(safe_config)
        )

    def _verify_model_integrity(self, model_path: str, version: str) -> bool:
        """校验模型文件完整性"""
        if not os.path.exists(model_path):
            return False
        sha256 = hashlib.sha256(open(model_path, "rb").read()).hexdigest()
        expected = self._get_expected_hash(version)
        return sha256 == expected
```

**防御策略总结：**

1. **车端保留最近 3 个模型版本：** 确保回滚时无需下载
2. **原子切换：** 配置文件使用 rename 操作（文件系统原子操作），避免写到一半断电
3. **SHA256 完整性校验：** 切换前验证模型文件未被损坏
4. **安全模式兜底：** 回滚失败的车辆进入安全模式（限速 + 禁用高级功能）
5. **超时控制：** 单车回滚 5 分钟超时，避免无限等待
6. **禁止混跑：** 回滚是全车队操作，不允许部分车辆回滚部分不回滚

## 延伸思考

- **联邦学习**：多车队的驾驶数据不出本地，只共享模型梯度 → 隐私保护 + 协作训练。具体而言，每辆车在本地用自己采集的数据计算梯度，将梯度上传到参数服务器做聚合（FedAvg），再下发全局模型。关键挑战是 Non-IID 数据分布（不同城市/路况的数据分布差异大）和通信效率（梯度压缩、异步聚合）。
- **世界模型**：学习驾驶环境的生成模型 → 无限生成仿真场景 → 场景库动态扩展。世界模型（如 GAIA-1、UniSim）学习视频+动作的条件分布，可以生成任意起始条件的未来驾驶视频。价值在于：将仿真场景库从"有限采样"升级为"无限生成"，大幅提高仿真覆盖面。挑战在于生成质量（时序一致性、物理合理性）和生成多样性（避免 mode collapse）。
- **闭环加速**：自动标注 → 自动训练 → 自动仿真 → 自动部署，全流程无人介入。目标：从发现到部署 < 24 小时。当前瓶颈在人工审核（约 8-12 小时），解决方向是提升自动标注置信度阈值（0.95→0.98），减少需要人工审核的比例。极端情况下可做"自动标注 + 自动仿真验证 + 灰度部署"，全流程零人工，但需要极高的自动标注和仿真可信度。
- **数据飞轮效应**：数据越多 → 模型越好 → 边缘过滤越精准 → 上传的数据越有价值 → 训练效果越好。这是自动驾驶数据闭环的终极目标——形成正反馈飞轮。但飞轮的"冷启动"阶段需要大量人工标注投入，且飞轮的"增速"取决于闭环周期的缩短。
## OTA 部署安全完整实现

```python
class OTADeploymentManager:
    """OTA 固件/模型部署安全管理器"""

    def deploy(self, version, target_vehicles):
        """安全 OTA 部署流程"""
        deployment_id = str(uuid4())

        # Phase 0: 预部署验证
        sim_result = self.simulation_runner.run(version, scenarios="critical_1000")
        if sim_result.pass_rate < 0.995:
            raise OTASafetyCheckFailedError(
                f"模拟通过率 {sim_result.pass_rate:.3f} < 99.5%")

        # Phase 1: Canary - 1% 车辆（最安全的一批）
        canary_vehicles = target_vehicles[:max(1, len(target_vehicles) // 100)]
        self._deploy_to_vehicles(deployment_id, version, canary_vehicles, phase="canary")

        # 等待 1 小时观察
        canary_result = self._monitor_for_duration(deployment_id, duration_hours=1)
        if not canary_result.is_safe:
            self._rollback_vehicles(canary_vehicles)
            raise OTACanaryFailedError(canary_result.reason)

        # Phase 2: Staged - 10% 车辆
        staged_vehicles = target_vehicles[:len(target_vehicles) // 10]
        self._deploy_to_vehicles(deployment_id, version, staged_vehicles, phase="staged")

        # 等待 12 小时观察
        staged_result = self._monitor_for_duration(deployment_id, duration_hours=12)
        if not staged_result.is_safe:
            self._rollback_vehicles(staged_vehicles)
            raise OTAStagedFailedError(staged_result.reason)

        # Phase 3: Full - 100% 车辆
        self._deploy_to_vehicles(deployment_id, version, target_vehicles, phase="full")

        # 部署后持续监控 48 小时
        self.schedule_post_deploy_monitoring(deployment_id, duration_hours=48)
        return {"deployment_id": deployment_id, "status": "deployed"}

    def _monitor_for_duration(self, deployment_id, duration_hours):
        """监控部署后安全指标"""
        metrics = {
            "critical_event_rate": self._get_critical_event_rate(deployment_id),
            "disengagement_rate": self._get_disengagement_rate(deployment_id),
            "perception_error_rate": self._get_perception_error_rate(deployment_id),
        }
        thresholds = {
            "critical_event_rate": 0.001,  # < 0.1%
            "disengagement_rate": 0.01,    # < 1%
            "perception_error_rate": 0.005, # < 0.5%
        }
        is_safe = all(metrics[k] < thresholds[k] for k in thresholds)
        return OTAMonitorResult(is_safe=is_safe, metrics=metrics,
            reason="" if is_safe else f"指标超标: {metrics}")

    def _rollback_vehicles(self, vehicles):
        """回滚车辆到上一版本"""
        prev_version = self.db.get_previous_stable_version()
        for v in vehicles:
            self.edge_client.rollback(v.id, prev_version)
            self.db.insert("ota_rollback_log", {
                "vehicle_id": v.id, "from_version": v.current_version,
                "to_version": prev_version, "reason": "safety_degradation"
            })
```

## 仿真-现实差距分析

```python
class Sim2RealGapAnalyzer:
    """仿真与现实差距检测与弥补"""

    def analyze_gap(self, model_version):
        """比较仿真指标与真实道路表现"""
        sim_metrics = self.db.get_simulation_metrics(model_version)
        real_metrics = self.db.get_real_world_metrics(model_version)

        gaps = []
        for metric_name in sim_metrics:
            sim_val = sim_metrics[metric_name]
            real_val = real_metrics[metric_name]
            gap_pct = abs(sim_val - real_val) / max(sim_val, real_val) * 100

            if gap_pct > 10:  # 差距 > 10%
                gaps.append({
                    "metric": metric_name,
                    "sim_value": sim_val, "real_value": real_val,
                    "gap_pct": gap_pct,
                    "severity": "HIGH" if gap_pct > 30 else "MEDIUM",
                    "root_cause": self._diagnose_gap(metric_name, sim_val, real_val)
                })

        # 自动生成新测试场景来弥补差距
        new_scenarios = []
        for gap in gaps:
            scenario = self._generate_gap_scenario(gap)
            new_scenarios.append(scenario)

        return {"gaps": gaps, "new_scenarios": new_scenarios}

    def _diagnose_gap(self, metric_name, sim_val, real_val):
        """诊断差距根因"""
        causes = {
            "sensor_noise": "仿真传感器噪声模型过于理想",
            "environmental": "仿真未覆盖雨天/雾天/逆光等环境",
            "behavioral": "仿真中其他车辆行为过于规律",
            "edge_case": "仿真未覆盖长尾场景（施工区域、非常规障碍物）",
        }
        if real_val < sim_val:
            return causes.get(metric_name, "现实表现差于仿真，可能存在未建模的干扰")
        return "仿真环境过于保守，可以适当放宽约束"

    def _generate_gap_scenario(self, gap):
        """根据差距生成新的测试场景"""
        return {
            "scenario_id": str(uuid4()),
            "target_metric": gap["metric"],
            "difficulty": "hard" if gap["severity"] == "HIGH" else "medium",
            "description": f"针对 {gap['metric']} 差距 ({gap['gap_pct']:.0f}%) 的补充场景",
            "environment_modifiers": self._get_env_modifiers(gap["root_cause"]),
        }
```

## 异常场景补充

### 场景：传感器融合分歧

```
触发：激光雷达检测到障碍物，但摄像头未检测到
      → 融合层判定不一致 → 自动驾驶系统降级
检测：
  1. 传感器置信度差异 > 0.3 → 告警
  2. 融合决策时间 > 100ms → 超时
处理：
  1. 保守策略：信任检测到障碍物的传感器
  2. 降低车速 + 增大安全距离
  3. 持续 30 秒未解决 → 要求驾驶员接管
预防：多传感器交叉验证 + 定期标定检查
```

### 场景：边缘计算节点故障

```
触发：车载计算平台 GPU 过热 → 降频 → 推理延迟从 50ms 升至 200ms
      → 无法在 100ms 内完成感知规划
检测：
  1. 推理延迟 > 100ms → 告警
  2. GPU 温度 > 85°C → 降频保护
处理：
  1. 切换到轻量模型（精度降低但延迟 < 50ms）
  2. 降低感知范围（只处理近处障碍物）
  3. 降低车速 + 增大跟车距离
  4. 持续过热 → 安全停车 + 通知驾驶员
预防：GPU 温度监控 + 预降温策略
```

### 场景：数据管道背压

```
触发：传感器数据产生速度 > 上传速度（4G 信号弱区域）
      → 本地存储逐渐填满 → 数据丢失风险
检测：
  1. 本地缓冲区使用率 > 80% → 告警
  2. 上传队列积压 > 10 分钟数据 → 严重告警
处理：
  1. 启用数据降级：只上传关键事件，丢弃常规帧
  2. 压缩上传：JPEG 质量 95 → 75，帧率 30fps → 15fps
  3. 恢复后补传丢失的常规数据
预防：边缘过滤 + 智能降级 + WiFi 补传
```

## 边缘到云端数据同步完整实现

```python
class EdgeCloudSyncService:
    """边缘到云端数据同步"""

    def sync_batch(self, vehicle_id):
        """批量同步：优先上传关键事件"""
        pending = self.db.query(
            "SELECT * FROM sync_queue WHERE vehicle_id = %s "
            "ORDER BY priority DESC, timestamp ASC LIMIT 500", vehicle_id)

        # 带宽限制：最多使用 50% 可用带宽
        bandwidth_limit = self.get_available_bandwidth(vehicle_id) * 0.5
        uploaded_bytes = 0

        for item in pending:
            if uploaded_bytes > bandwidth_limit:
                break  # 带宽限制，停止上传

            try:
                self.cloud_client.upload(item["url"], item["data"])
                self.db.update("sync_queue",
                    {"status": "uploaded", "uploaded_at": now()},
                    {"id": item["id"]})
                uploaded_bytes += item["data_size"]
            except Exception as e:
                self.db.update("sync_queue",
                    {"retry_count": item["retry_count"] + 1, "last_error": str(e)},
                    {"id": item["id"]})
```

## 异常场景补充

### 场景：训练数据污染检测

```python
class TrainingDataContaminationDetector:
    """训练数据污染检测"""
    def detect_mislabel(self, dataset):
        """检测错误标签数据"""
        suspicious = []
        for sample in dataset:
            # 用已训练模型推理 → 比较预测标签与标注标签
            prediction = self.model.predict(sample.features)
            if prediction.label != sample.label and prediction.confidence > 0.95:
                suspicious.append({
                    "sample_id": sample.id,
                    "annotated_label": sample.label,
                    "predicted_label": prediction.label,
                    "confidence": prediction.confidence,
                    "likely_mislabel": True
                })
        return suspicious
```

### 场景：OTA 回滚失败

```
触发：OTA 部署后指标劣化 → 自动回滚 → 但回滚版本文件损坏
      → 设备处于"降级模式"：只用轻量模型 + 降低速度
检测：
  1. 回滚后版本验证失败 → SHA256 不匹配
  2. 设备上报 "degraded_mode" → 告警
处理：
  1. 重新下载上一稳定版本的固件
  2. 如果下载失败 → 进入降级模式（轻量模型）
  3. 降级模式：速度限制 30km/h + 增大安全距离
  4. 持续尝试恢复 → 成功后退出降级模式
预防：固件多重校验（SHA256 + 数字签名）+ 回滚版本预缓存
```

## 性能分析详细数据

**端到端延迟分解：**

| 阶段 | P50 | P95 | P99 | 说明 |
|------|-----|-----|-----|------|
| 传感器采集 | 5ms | 8ms | 12ms | 激光雷达+摄像头同步 |
| 感知推理 | 30ms | 45ms | 60ms | GPU A100 推理 |
| 融合决策 | 20ms | 30ms | 40ms | 多传感器融合 |
| 规划 | 30ms | 50ms | 80ms | 路径规划 |
| 控制执行 | 10ms | 15ms | 20ms | CAN 总线通信 |
| **总计** | **95ms** | **148ms** | **212ms** | 满足 <250ms 要求 |

**训练数据规模：**

| 数据类型 | 日增量 | 月存储 | 年存储 |
|---------|--------|--------|--------|
| 原始感知数据 | 2TB | 60TB | 720TB |
| 标注数据 | 50GB | 1.5TB | 18TB |
| 训练检查点 | 100GB | 3TB | 36TB |
| 模型版本 | 10GB | 300GB | 3.6TB |

**月度成本：**

| 组件 | 规格 | 月成本 |
|------|------|-------|
| 边缘计算(1000辆车) | Orin 32TOPS × 2 | ¥20 万 |
| 云端训练集群 | 8×A100 × 3个月 | ¥50 万 |
| 数据标注平台 | SaaS | ¥5 万 |
| 对象存储(S3) | 100TB | ¥2 万 |
| Kafka+Flink | 6节点 | ¥3 万 |
| **合计** | | **¥80 万** |

## 数据标注流水线完整实现

```python
class DataLabelingPipeline:
    """数据标注流水线：标注 → 审核 → 质检"""

    def submit_labeling_task(self, raw_data_batch):
        """提交标注任务"""
        task_id = str(uuid4())
        # 按场景类型分配标注员
        labelers = self.get_available_labelers(raw_data_batch["scene_type"])

        self.db.insert("labeling_tasks", {
            "task_id": task_id,
            "data_type": raw_data_batch["data_type"],
            "scene_type": raw_data_batch["scene_type"],
            "total_frames": raw_data_batch["frame_count"],
            "assigned_labeler": labelers[0].id,
            "status": "assigned",
            "priority": "critical" if raw_data_batch["scene_type"] == "edge_case" else "normal"
        })
        return task_id

    def review_labeling(self, task_id):
        """审核标注质量"""
        task = self.db.get_labeling_task(task_id)
        # 质检规则：
        # 1. 目标框重叠 > 30% → 重复标注
        # 2. 目标框过小 (< 10px) → 可能误标
        # 3. 类别标注与模型预测差异大 → 需复查
        issues = self._check_quality(task)
        if issues:
            self.db.update("labeling_tasks",
                {"status": "needs_revision", "quality_issues": json.dumps(issues)},
                {"task_id": task_id})
            return {"status": "needs_revision", "issues": issues}
        else:
            self.db.update("labeling_tasks",
                {"status": "approved", "approved_at": now()},
                {"task_id": task_id})
            return {"status": "approved"}
```

## 模型训练编排

```python
class ModelTrainingOrchestrator:
    """模型训练编排：数据准备 → 训练 → 评估 → 部署"""

    def orchestrate_training(self, model_config):
        """编排完整的训练流程"""
        run_id = str(uuid4())

        # Phase 1: 数据准备
        dataset = self.data_prep.prepare(model_config["data_version"])
        if dataset.quality_score < 0.95:
            raise DataQualityError(f"数据质量 {dataset.quality_score} < 0.95")

        # Phase 2: 训练
        training_result = self.trainer.train(model_config, dataset)
        if training_result.loss > model_config["max_loss"]:
            raise TrainingQualityError(f"Loss {training_result.loss} > threshold")

        # Phase 3: 评估
        eval_result = self.evaluator.evaluate(training_result.model_path)
        if eval_result.accuracy < model_config["min_accuracy"]:
            raise EvaluationError(f"Accuracy {eval_result.accuracy} < {model_config['min_accuracy']}")

        # Phase 4: 注册模型版本
        self.model_registry.register(run_id, {
            "model_path": training_result.model_path,
            "accuracy": eval_result.accuracy,
            "loss": training_result.loss,
            "data_version": model_config["data_version"],
            "config": model_config
        })

        return {"run_id": run_id, "accuracy": eval_result.accuracy, "status": "ready_for_deploy"}
```

## 异常场景补充

### 场景：GPS 信号欺骗检测

```python
class GPSSpoofingDetector:
    """GPS 欺骗检测"""
    def check_spoofing(self, gps_point, imu_data):
        """检测 GPS 信号是否被欺骗"""
        # 1. GPS 速度 vs IMU 加速度不一致
        gps_velocity = gps_point.speed
        imu_acceleration = imu_data.linear_acceleration
        # 如果 GPS 显示高速移动但 IMU 无加速度 → 欺骗
        if gps_velocity > 30 and abs(imu_acceleration) < 0.5:
            return {"spoofed": True, "reason": "GPS速度与IMU加速度不一致"}

        # 2. GPS 位置跳跃（不可能的速度）
        # 3. GPS 信号强度异常（C/N0 值过低）
        # 4. 多路径效应检测
        return {"spoofed": False}
```

### 场景：传感器标定漂移

```
触发：摄像头标定参数随时间变化 → 3D 检测精度下降
检测：
  1. 定期自检：用已知尺寸的标定板检测精度
  2. 检测精度 > 5% 偏差 → 标定漂移告警
处理：
  1. 自动在线标定：利用道路标线和已知地标
  2. 标定参数缓慢修正（每 1000 帧修正 1%）
  3. 漂移 > 10% → 提示驾驶员前往服务中心重新标定
预防：定期在线标定 + 离线标定校验
```

### 场景：模型推理超时

```
触发：复杂场景（多目标+恶劣天气）→ 推理时间 > 100ms
      → 超过实时性要求 → 无法及时做出决策
检测：
  1. 推理时间 > 80ms → 告警
  2. 推理时间 > 100ms → 严重告警
处理：
  1. 切换到轻量模型（精度降低但延迟 < 50ms）
  2. 降低输入分辨率（从 1080p → 720p）
  3. 减少 BEV 范围（只处理近处 50 米）
  4. 降低车速 + 增大安全距离
预防：GPU 温度监控 + 推理延迟监控 + 轻量模型备用
```

## 车队管理完整实现

```python
class FleetManager:
    """车队管理：车辆注册、传感器标定、固件版本"""

    def register_vehicle(self, vehicle_info):
        """注册新车辆"""
        vehicle_id = str(uuid4())
        self.db.insert("vehicles", {
            "vehicle_id": vehicle_id,
            "model": vehicle_info["model"],
            "sensor_config": json.dumps(vehicle_info["sensors"]),
            "firmware_version": "1.0.0",
            "status": "registered",
            "registered_at": now()
        })
        # 注册所有传感器
        for sensor in vehicle_info["sensors"]:
            self.db.insert("vehicle_sensors", {
                "vehicle_id": vehicle_id,
                "sensor_type": sensor["type"],
                "sensor_id": sensor["id"],
                "calibration_date": now(),
                "next_calibration_date": now() + timedelta(days=90),
                "status": "calibrated"
            })
        return vehicle_id

    def check_calibration_status(self, vehicle_id):
        """检查传感器标定状态"""
        sensors = self.db.query(
            "SELECT * FROM vehicle_sensors WHERE vehicle_id = %s", vehicle_id)
        expired = [s for s in sensors if s["next_calibration_date"] < now()]
        if expired:
            return {"needs_recalibration": True, "expired_sensors": expired}
        return {"needs_recalibration": False}

    def get_firmware_compatibility(self, model):
        """获取固件兼容矩阵"""
        return self.db.query(
            "SELECT * FROM firmware_compatibility WHERE model = %s ORDER BY version DESC",
            model)
```

## 数据管道架构

```python
class DataPipelineManager:
    """自动驾驶数据管道管理"""

    PIPELINE_STAGES = [
        {"name": "edge_filter", "description": "边缘端数据过滤，丢弃无用帧"},
        {"name": "cloud_upload", "description": "上传关键事件和标注数据"},
        {"name": "data_labeling", "description": "人工标注和质量审核"},
        {"name": "dataset_preparation", "description": "数据集划分和特征工程"},
        {"name": "model_training", "description": "分布式 GPU 训练"},
        {"name": "model_evaluation", "description": "模拟和真实场景评估"},
        {"name": "model_deployment", "description": "OTA 金丝雀部署"},
    ]

    def get_pipeline_status(self, pipeline_id):
        """获取管道执行状态"""
        stages = self.db.query(
            "SELECT * FROM pipeline_stages WHERE pipeline_id = %s ORDER BY stage_order",
            pipeline_id)
        return {
            "pipeline_id": pipeline_id,
            "stages": [{
                "name": s["stage_name"],
                "status": s["status"],
                "started_at": s["started_at"],
                "completed_at": s["completed_at"],
                "duration_seconds": (s["completed_at"] - s["started_at"]).total_seconds() if s["completed_at"] else None
            } for s in stages]
        }
```

## 性能分析详细数据

**GPU 训练成本：**

| 模型 | 训练数据量 | GPU 时间 | A100 数量 | 单次训练成本 |
|------|----------|---------|----------|-------------|
| 感知模型 | 100 万帧 | 72 小时 | 8 | ¥5 万 |
| 规划模型 | 50 万场景 | 48 小时 | 4 | ¥2 万 |
| 融合模型 | 80 万帧 | 96 小时 | 8 | ¥7 万 |
| **总计** | | | | **¥14 万/月** |

**边缘计算硬件成本：**

| 硬件 | 单价 | 每车数量 | 1000 车总成本 |
|------|------|---------|-------------|
| NVIDIA Orin | ¥8000 | 2 | ¥1600 万 |
| 激光雷达 | ¥5000 | 1 | ¥500 万 |
| 摄像头组 | ¥2000 | 6 | ¥1200 万 |
| **总计** | | | **¥3300 万** |

**数据标注成本：**

| 标注类型 | 单帧成本 | 月标注量 | 月成本 |
|---------|---------|---------|-------|
| 2D 框标注 | ¥0.5 | 100 万帧 | ¥50 万 |
| 3D 框标注 | ¥3 | 20 万帧 | ¥60 万 |
| 语义分割 | ¥2 | 30 万帧 | ¥60 万 |
| **总计** | | | **¥170 万/月** |

## 传感器健康监控完整实现

```python
class SensorHealthMonitor:
    """传感器健康监控仪表盘"""

    def check_all_sensors(self, vehicle_id):
        """检查车辆所有传感器健康状态"""
        sensors = self.db.query(
            "SELECT * FROM vehicle_sensors WHERE vehicle_id = %s", vehicle_id)
        health_report = []
        for sensor in sensors:
            metrics = self._get_sensor_metrics(sensor)
            health = {
                "sensor_id": sensor["sensor_id"],
                "sensor_type": sensor["sensor_type"],
                "status": "healthy",
                "issues": []
            }

            # 1. 温度检查
            if metrics["temperature_c"] > 85:
                health["status"] = "degraded"
                health["issues"].append(f"温度过高: {metrics['temperature_c']}°C")

            # 2. 帧率检查
            if metrics["fps"] < sensor["expected_fps"] * 0.8:
                health["status"] = "degraded"
                health["issues"].append(f"帧率下降: {metrics['fps']}/{sensor['expected_fps']} fps")

            # 3. 错误率检查
            if metrics["error_count_1h"] > 10:
                health["status"] = "degraded"
                health["issues"].append(f"错误率过高: {metrics['error_count_1h']}/小时")

            # 4. 标定状态
            if sensor["next_calibration_date"] < now():
                health["status"] = "degraded"
                health["issues"].append("标定已过期")

            # 5. 严重故障 → 自动禁用
            if metrics["error_count_1h"] > 100:
                health["status"] = "failed"
                self._disable_sensor_and_redistribute(vehicle_id, sensor)

            health_report.append(health)

        return health_report

    def _disable_sensor_and_redistribute(self, vehicle_id, failed_sensor):
        """禁用故障传感器并重新分配工作负载"""
        self.db.update("vehicle_sensors",
            {"status": "disabled", "disabled_at": now()},
            {"sensor_id": failed_sensor["sensor_id"]})

        # 重新分配：如果摄像头失效 → 增加其他摄像头权重
        remaining = self.db.query(
            "SELECT * FROM vehicle_sensors WHERE vehicle_id = %s "
            "AND sensor_type = %s AND status = 'active'",
            vehicle_id, failed_sensor["sensor_type"])
        if not remaining:
            # 无备用传感器 → 降级运行
            self.alert(f"车辆 {vehicle_id} 无可用 {failed_sensor['sensor_type']}，进入降级模式")
            self.enter_degraded_mode(vehicle_id)
```

## 场景回归测试框架

```python
class ScenarioTestFramework:
    """场景回归测试：模型更新后验证所有场景"""

    def run_regression(self, model_version):
        """运行完整回归测试"""
        scenarios = self.db.query(
            "SELECT * FROM test_scenarios WHERE enabled = 1 ORDER BY priority DESC")
        results = []
        passed, failed = 0, 0

        for scenario in scenarios:
            result = self._run_scenario(model_version, scenario)
            if result["passed"]:
                passed += 1
            else:
                failed += 1
                # 高优先级失败 → 阻止部署
                if scenario["priority"] == "critical":
                    self.alert(f"关键场景失败: {scenario['name']}")
            results.append(result)

        pass_rate = passed / (passed + failed)
        return {
            "model_version": model_version,
            "total_scenarios": len(scenarios),
            "passed": passed, "failed": failed,
            "pass_rate": pass_rate,
            "can_deploy": pass_rate >= 0.995 and failed == 0,
            "results": results
        }

    def _run_scenario(self, model_version, scenario):
        """运行单个测试场景"""
        simulation = self.simulator.run(model_version, scenario)
        passed = all(
            simulation.metrics[k] <= scenario["thresholds"][k]
            for k in scenario["thresholds"]
        )
        return {
            "scenario_id": scenario["id"],
            "scenario_name": scenario["name"],
            "passed": passed,
            "metrics": simulation.metrics,
            "thresholds": scenario["thresholds"]
        }
```

## 异常场景补充

### 场景：传感器标定数据丢失

```
触发：标定服务器崩溃 → 最新标定参数丢失
      → 3D 感知精度下降
检测：
  1. 标定数据版本回退 → 告警
  2. 感知精度 < 阈值 → 告警
处理：
  1. 从备份恢复最近标定数据
  2. 如果备份也丢失 → 使用出厂标定参数
  3. 启动在线标定补偿
预防：标定数据三副本存储 + 定期备份
```

### 场景：回归测试失败阻止部署

```
触发：模型更新后 3 个非关键场景失败 → pass_rate = 99.7% < 99.5%
      → 部署被阻止
处理：
  1. 分析失败场景：是否与模型变更相关
  2. 无关失败（如仿真环境问题）→ 标记为 flaky → 允许部署
  3. 模型相关失败 → 修复后重新测试
  4. 紧急安全修复 → CTO 审批可跳过非关键场景
预防：flaky 测试自动检测 + 分级场景管理
```

### 场景：OTA 部署卡在 50%

```
触发：OTA 部署到 50% 车辆时停止 → 剩余车辆未收到更新
检测：
  1. 部署进度停滞 > 30 分钟 → 告警
  2. 部分车辆新版本，部分旧版本 → 版本分叉
处理：
  1. 检查是否被健康检查阻止（50% 车辆指标异常）
  2. 如健康问题 → 修复后继续部署或回滚
  3. 如网络问题 → 重试推送
  4. 长时间停滞 → 人工介入决策
预防：部署进度监控 + 超时自动决策
```

## 传感器健康监控仪表盘完整实现

自动驾驶的安全依赖于传感器数据的准确性和及时性。传感器健康监控仪表盘实时追踪每辆车每个传感器的运行状态，检测性能退化，自动禁用故障传感器并重新分配工作负载。

```python
class SensorHealthDashboard:
    """
    传感器健康监控仪表盘。
    追踪每辆车的每个传感器的：温度、帧率、错误计数、标定状态。
    检测传感器性能退化（渐进式下降），自动禁用故障传感器并重新分配负载。
    """

    HEALTH_THRESHOLDS = {
        "camera": {
            "max_temperature_c": 85,
            "min_fps": 25,
            "max_error_count_1h": 10,
            "max_error_count_1h_critical": 100,
            "fps_degradation_pct": 15,
            "calibration_max_age_days": 90,
        },
        "lidar": {
            "max_temperature_c": 75,
            "min_fps": 9,
            "max_error_count_1h": 5,
            "max_error_count_1h_critical": 50,
            "fps_degradation_pct": 10,
            "calibration_max_age_days": 60,
            "max_point_count_drop_pct": 20,
        },
        "radar": {
            "max_temperature_c": 80,
            "min_fps": 12,
            "max_error_count_1h": 5,
            "max_error_count_1h_critical": 50,
            "fps_degradation_pct": 10,
            "calibration_max_age_days": 90,
        },
        "gps": {
            "max_hdop": 2.0,
            "min_satellite_count": 6,
            "max_position_jump_m": 5.0,
        },
        "imu": {
            "max_temperature_c": 70,
            "max_accel_bias_mps2": 0.5,
            "max_gyro_bias_dps": 1.0,
        },
    }

    SENSOR_IMPACT = {
        "camera_front_main": {
            "impact": "HIGH",
            "affected_functions": ["车道检测", "交通标志识别", "前车距离估计"],
            "fallback": "camera_front_wide + lidar 前方点云补偿",
        },
        "camera_front_wide": {
            "impact": "MEDIUM",
            "affected_functions": ["广角视野", "近距离目标检测"],
            "fallback": "camera_front_main 覆盖主视野",
        },
        "lidar_top": {
            "impact": "HIGH",
            "affected_functions": ["3D目标检测", "点云分割", "距离精确测量"],
            "fallback": "camera + radar 融合补偿（精度降低）",
        },
        "radar_front": {
            "impact": "MEDIUM",
            "affected_functions": ["远距离目标检测", "速度测量"],
            "fallback": "lidar 近距离补偿",
        },
        "gps": {
            "impact": "MEDIUM",
            "affected_functions": ["定位", "路径规划"],
            "fallback": "IMU + 轮速计 + 高精地图匹配",
        },
    }

    def __init__(self, db, redis, mqtt_client):
        self.db = db
        self.redis = redis
        self.mqtt_client = mqtt_client

    def check_vehicle_sensors(self, vehicle_id: str) -> dict:
        """检查单辆车的所有传感器健康状态"""
        sensors = self.db.query(
            "SELECT * FROM vehicle_sensors "
            "WHERE vehicle_id = %s AND status = 'active'",
            [vehicle_id]
        )
        health_report = {
            "vehicle_id": vehicle_id,
            "check_time": now().isoformat(),
            "overall_health": "healthy",
            "healthy_count": 0,
            "degraded_count": 0,
            "failed_count": 0,
            "sensors": [],
        }
        for sensor in sensors:
            metrics = self._get_sensor_metrics(sensor)
            sensor_type = sensor["sensor_type"]
            thresholds = self.HEALTH_THRESHOLDS.get(sensor_type, {})
            health = self._evaluate_sensor_health(sensor, metrics, thresholds)
            health_report["sensors"].append(health)
            if health["status"] == "healthy":
                health_report["healthy_count"] += 1
            elif health["status"] == "degraded":
                health_report["degraded_count"] += 1
            elif health["status"] == "failed":
                health_report["failed_count"] += 1
                self._handle_sensor_failure(vehicle_id, sensor, health)

        if health_report["failed_count"] > 0:
            health_report["overall_health"] = "degraded"
        if health_report["failed_count"] >= 2:
            health_report["overall_health"] = "critical"

        self.redis.setex(
            f"sensor_health:{vehicle_id}", 60,
            json.dumps(health_report, ensure_ascii=False)
        )
        return health_report

    def _evaluate_sensor_health(self, sensor, metrics, thresholds) -> dict:
        """评估单个传感器健康状态"""
        sensor_type = sensor["sensor_type"]
        sensor_id = sensor["sensor_id"]
        health = {
            "sensor_id": sensor_id,
            "sensor_type": sensor_type,
            "status": "healthy",
            "issues": [],
            "metrics": metrics,
        }

        # 温度检查
        temp = metrics.get("temperature_c", 0)
        max_temp = thresholds.get("max_temperature_c", 85)
        if temp > max_temp:
            health["status"] = "degraded"
            health["issues"].append(f"温度过高: {temp}C（阈值 {max_temp}C）")

        # 帧率检查
        fps = metrics.get("fps", 0)
        min_fps = thresholds.get("min_fps", 0)
        if min_fps > 0 and fps < min_fps:
            health["status"] = "degraded"
            health["issues"].append(f"帧率下降: {fps} fps（最低 {min_fps} fps）")

        # 错误率检查
        error_count = metrics.get("error_count_1h", 0)
        max_errors = thresholds.get("max_error_count_1h", 10)
        critical_errors = thresholds.get("max_error_count_1h_critical", 100)
        if error_count > max_errors:
            health["status"] = "degraded"
            health["issues"].append(f"错误率过高: {error_count} 次/小时")
        if error_count > critical_errors:
            health["status"] = "failed"
            health["issues"].append(f"严重故障: {error_count} 次/小时，自动禁用")

        # 标定状态检查
        next_cal = sensor.get("next_calibration_date")
        if next_cal and next_cal < now():
            health["status"] = max(health["status"], "degraded")
            health["issues"].append("标定已过期")

        # 性能退化检测
        baseline_fps = metrics.get("baseline_fps", 0)
        if baseline_fps > 0 and fps > 0:
            degradation_pct = (baseline_fps - fps) / baseline_fps * 100
            degradation_threshold = thresholds.get("fps_degradation_pct", 15)
            if degradation_pct > degradation_threshold:
                health["status"] = max(health["status"], "degraded")
                health["issues"].append(
                    f"帧率退化: 较基线下降 {degradation_pct:.1f}%"
                )

        # LiDAR 专用：点云数量检查
        if sensor_type == "lidar":
            point_count = metrics.get("point_count", 0)
            baseline_points = metrics.get("baseline_point_count", 200000)
            if baseline_points > 0 and point_count > 0:
                drop_pct = (baseline_points - point_count) / baseline_points * 100
                max_drop = thresholds.get("max_point_count_drop_pct", 20)
                if drop_pct > max_drop:
                    health["status"] = max(health["status"], "degraded")
                    health["issues"].append(f"点云数量下降: {drop_pct:.1f}%")

        # GPS 专用检查
        if sensor_type == "gps":
            hdop = metrics.get("hdop", 0)
            satellites = metrics.get("satellite_count", 0)
            pos_jump = metrics.get("position_jump_m", 0)
            if hdop > thresholds.get("max_hdop", 2.0):
                health["status"] = max(health["status"], "degraded")
                health["issues"].append(f"GPS精度差: HDOP={hdop}")
            if satellites < thresholds.get("min_satellite_count", 6):
                health["status"] = max(health["status"], "degraded")
                health["issues"].append(f"卫星数不足: {satellites}")
            if pos_jump > thresholds.get("max_position_jump_m", 5.0):
                health["status"] = max(health["status"], "degraded")
                health["issues"].append(f"位置跳变: {pos_jump:.1f}m")

        # IMU 专用检查
        if sensor_type == "imu":
            accel_bias = metrics.get("accel_bias_mps2", 0)
            gyro_bias = metrics.get("gyro_bias_dps", 0)
            if abs(accel_bias) > thresholds.get("max_accel_bias_mps2", 0.5):
                health["status"] = max(health["status"], "degraded")
                health["issues"].append(f"加速度计偏差: {accel_bias:.3f} m/s2")
            if abs(gyro_bias) > thresholds.get("max_gyro_bias_dps", 1.0):
                health["status"] = max(health["status"], "degraded")
                health["issues"].append(f"陀螺仪偏差: {gyro_bias:.3f} deg/s")

        return health

    def _handle_sensor_failure(self, vehicle_id, sensor, health):
        """处理传感器故障：禁用传感器并重新分配工作负载"""
        sensor_id = sensor["sensor_id"]
        self.db.update("vehicle_sensors", {
            "status": "disabled",
            "disabled_at": now(),
            "disabled_reason": "; ".join(health["issues"]),
        }, {"sensor_id": sensor_id})

        remaining = self.db.query(
            "SELECT * FROM vehicle_sensors "
            "WHERE vehicle_id = %s AND sensor_type = %s AND status = 'active'",
            [vehicle_id, sensor["sensor_type"]]
        )

        impact = self.SENSOR_IMPACT.get(sensor_id, {
            "impact": "MEDIUM",
            "affected_functions": ["未知"],
            "fallback": "无备用方案",
        })

        if not remaining:
            self._activate_cross_sensor_fallback(vehicle_id, sensor_id, impact)
        else:
            for backup in remaining:
                self._increase_sensor_weight(vehicle_id, backup["sensor_id"])

        degradation_config = {
            "disabled_sensor": sensor_id,
            "impact_level": impact["impact"],
            "affected_functions": impact["affected_functions"],
            "fallback": impact["fallback"],
            "speed_limit_kmh": 60 if impact["impact"] == "HIGH" else 80,
            "safe_distance_multiplier": 1.5 if impact["impact"] == "HIGH" else 1.2,
        }
        self.mqtt_client.publish(
            f"cmd/{vehicle_id}/sensor_degradation",
            json.dumps(degradation_config)
        )
        self.alert_team(
            f"[传感器故障] 车辆 {vehicle_id} 传感器 {sensor_id} 已自动禁用。"
            f"原因: {'; '.join(health['issues'])}。"
            f"影响等级: {impact['impact']}。降级方案: {impact['fallback']}。"
        )

    def _activate_cross_sensor_fallback(self, vehicle_id, failed_sensor_id, impact):
        """跨类型传感器补偿"""
        fallback_plan = {
            "camera_front_main": {
                "use": ["lidar_top", "camera_front_wide"],
                "mode": "lidar_primary_detection",
                "description": "激光雷达为主检测，广角摄像头辅助",
            },
            "lidar_top": {
                "use": ["camera_front_main", "camera_front_wide", "radar_front"],
                "mode": "camera_radar_fusion",
                "description": "摄像头+雷达融合补偿（精度降低约15%）",
            },
            "radar_front": {
                "use": ["lidar_top", "camera_front_main"],
                "mode": "lidar_camera_fusion",
                "description": "激光雷达+摄像头融合，减少远距离检测能力",
            },
            "gps": {
                "use": ["imu", "wheel_odometry"],
                "mode": "dead_reckoning",
                "description": "惯性导航+轮速计+高精地图匹配",
            },
        }
        plan = fallback_plan.get(failed_sensor_id, {
            "use": [],
            "mode": "minimal_safe",
            "description": "最小安全模式：限速30km/h",
        })
        self.mqtt_client.publish(
            f"cmd/{vehicle_id}/fallback_plan",
            json.dumps({
                "failed_sensor": failed_sensor_id,
                "fallback_sensors": plan["use"],
                "fusion_mode": plan["mode"],
                "description": plan["description"],
                "activated_at": now().isoformat(),
            })
        )

    def _increase_sensor_weight(self, vehicle_id, sensor_id):
        """增加备用传感器的融合权重"""
        self.mqtt_client.publish(
            f"cmd/{vehicle_id}/sensor_weight",
            json.dumps({
                "sensor_id": sensor_id,
                "weight_adjustment": +0.3,
                "reason": "primary_sensor_failure_compensation",
            })
        )

    def detect_degradation_trend(self, vehicle_id: str) -> dict:
        """检测传感器性能退化趋势（渐进式下降）"""
        sensors = self.db.query(
            "SELECT * FROM vehicle_sensors "
            "WHERE vehicle_id = %s AND status = 'active'",
            [vehicle_id]
        )
        degradation_alerts = []
        for sensor in sensors:
            metrics_7d = self._get_avg_metrics(sensor["sensor_id"], days=7)
            metrics_30d = self._get_avg_metrics(sensor["sensor_id"], days=30)
            if not metrics_7d or not metrics_30d:
                continue

            fps_7d = metrics_7d.get("avg_fps", 0)
            fps_30d = metrics_30d.get("avg_fps", 0)
            if fps_30d > 0:
                fps_trend = (fps_30d - fps_7d) / fps_30d * 100
                if fps_trend > 5:
                    degradation_alerts.append({
                        "sensor_id": sensor["sensor_id"],
                        "sensor_type": sensor["sensor_type"],
                        "metric": "fps",
                        "trend": f"下降 {fps_trend:.1f}%",
                        "7d_avg": fps_7d,
                        "30d_avg": fps_30d,
                        "severity": "MEDIUM" if fps_trend < 10 else "HIGH",
                    })

            temp_7d = metrics_7d.get("avg_temperature_c", 0)
            temp_30d = metrics_30d.get("avg_temperature_c", 0)
            if temp_7d > temp_30d + 5:
                degradation_alerts.append({
                    "sensor_id": sensor["sensor_id"],
                    "sensor_type": sensor["sensor_type"],
                    "metric": "temperature",
                    "trend": f"上升 {temp_7d - temp_30d:.1f}C",
                    "severity": "MEDIUM",
                })

            errors_7d = metrics_7d.get("avg_error_count_1h", 0)
            errors_30d = metrics_30d.get("avg_error_count_1h", 0)
            if errors_30d > 0 and errors_7d > errors_30d * 2:
                degradation_alerts.append({
                    "sensor_id": sensor["sensor_id"],
                    "sensor_type": sensor["sensor_type"],
                    "metric": "error_rate",
                    "trend": f"上升 {(errors_7d / errors_30d - 1) * 100:.0f}%",
                    "severity": "HIGH",
                })

        return {
            "vehicle_id": vehicle_id,
            "degradation_count": len(degradation_alerts),
            "alerts": degradation_alerts,
            "checked_at": now().isoformat(),
        }

    def _get_sensor_metrics(self, sensor):
        """获取传感器实时指标"""
        sensor_id = sensor["sensor_id"]
        cached = self.redis.get(f"sensor_metrics:{sensor_id}")
        if cached:
            return json.loads(cached)
        return self.db.query_one(
            "SELECT * FROM sensor_metrics_latest WHERE sensor_id = %s",
            [sensor_id]
        ) or {}

    def _get_avg_metrics(self, sensor_id, days):
        """获取传感器历史平均指标"""
        return self.db.query_one(
            "SELECT AVG(fps) as avg_fps, "
            "AVG(temperature_c) as avg_temperature_c, "
            "AVG(error_count_1h) as avg_error_count_1h "
            "FROM sensor_metrics_daily "
            "WHERE sensor_id = %s "
            "AND date >= DATE_SUB(CURDATE(), INTERVAL %s DAY)",
            [sensor_id, days]
        ) or {}
```

## 场景回归测试框架完整实现

自动驾驶模型每次更新后，必须通过全量场景回归测试才能部署。场景测试框架管理所有测试场景的定义、执行和结果追踪，确保模型不会在已有场景上退化。

```python
class ScenarioRegistry:
    """
    场景注册表：管理所有测试场景及其通过/失败标准。
    场景来源：路测长尾、法规合规、对抗生成、随机参数化。
    每个场景有明确的通过/失败阈值，支持自动化判定。
    """

    SCENARIO_CATEGORIES = {
        "regulation": {
            "name": "法规合规",
            "priority": "critical",
            "pass_threshold": 1.0,
            "examples": ["AEB_行人横穿", "LKA_车道保持", "FCW_前车急刹"],
        },
        "corner_case": {
            "name": "长尾场景",
            "priority": "high",
            "pass_threshold": 0.995,
            "examples": ["施工路段_锥桶", "逆光_行人", "积水路面_障碍物"],
        },
        "adversarial": {
            "name": "对抗场景",
            "priority": "high",
            "pass_threshold": 0.99,
            "examples": ["暴雪_连环碰撞", "浓雾_紧急避障", "GPS欺骗"],
        },
        "random": {
            "name": "随机仿真",
            "priority": "medium",
            "pass_threshold": 0.98,
            "examples": ["随机交通流_城市路口", "随机天气_高速巡航"],
        },
    }

    def __init__(self, db):
        self.db = db

    def register_scenario(self, scenario: dict) -> str:
        """注册新测试场景"""
        scenario_id = scenario.get("scenario_id", f"SC-{uuid4().hex[:8]}")
        category = scenario["category"]
        category_config = self.SCENARIO_CATEGORIES.get(category, {})
        self.db.insert("test_scenarios", {
            "scenario_id": scenario_id,
            "name": scenario["name"],
            "category": category,
            "priority": category_config.get("priority", "medium"),
            "pass_threshold": scenario.get(
                "pass_threshold",
                category_config.get("pass_threshold", 0.99)
            ),
            "description": scenario.get("description", ""),
            "environment": json.dumps(scenario.get("environment", {})),
            "actors": json.dumps(scenario.get("actors", [])),
            "thresholds": json.dumps(scenario.get("thresholds", {})),
            "source": scenario.get("source", "manual"),
            "enabled": True,
            "created_at": now(),
        })
        return scenario_id

    def get_scenarios_for_regression(self) -> list:
        """获取需要执行回归测试的所有场景"""
        return self.db.query(
            "SELECT * FROM test_scenarios "
            "WHERE enabled = 1 "
            "ORDER BY "
            "  CASE priority "
            "    WHEN 'critical' THEN 1 "
            "    WHEN 'high' THEN 2 "
            "    WHEN 'medium' THEN 3 "
            "    ELSE 4 "
            "  END, category, scenario_id"
        )

    def generate_scenario_from_incident(self, incident: dict) -> str:
        """从真实事故生成新的测试场景"""
        scenario = {
            "scenario_id": f"INC-{incident['incident_id']}",
            "name": f"事故衍生: {incident.get('description', '未知')[:50]}",
            "category": "corner_case",
            "source": "real_incident",
            "description": (
                f"从真实事故 #{incident['incident_id']} 生成。"
                f"事故类型: {incident.get('type', '未知')}。"
                f"原始数据: {incident.get('data_path', '')}"
            ),
            "environment": {
                "weather": incident.get("weather", "unknown"),
                "road_type": incident.get("road_type", "unknown"),
                "time_of_day": incident.get("time_of_day", "unknown"),
                "visibility_m": incident.get("visibility_m", 1000),
            },
            "actors": incident.get("actors", []),
            "thresholds": {
                "collision": False,
                "violation": False,
                "min_comfort_score": 0.7,
                "min_completion_score": 0.9,
            },
        }
        return self.register_scenario(scenario)


class ScenarioRegressionRunner:
    """
    场景回归测试执行器。
    模型更新后，运行全部场景的回归测试，与基线模型对比。
    """

    def __init__(self, db, simulator, registry):
        self.db = db
        self.simulator = simulator
        self.registry = registry

    def run_full_regression(
        self, new_model_version: str, baseline_model_version: str
    ) -> dict:
        """执行完整回归测试"""
        scenarios = self.registry.get_scenarios_for_regression()
        results = {
            "new_model": new_model_version,
            "baseline_model": baseline_model_version,
            "total_scenarios": len(scenarios),
            "passed": 0,
            "failed": 0,
            "regressed": 0,
            "improved": 0,
            "details": [],
            "can_deploy": True,
            "blockers": [],
        }

        for scenario in scenarios:
            detail = self._run_single_scenario(
                scenario, new_model_version, baseline_model_version
            )
            results["details"].append(detail)
            if detail["new_passed"] and not detail["regressed"]:
                results["passed"] += 1
            else:
                results["failed"] += 1
            if detail["regressed"]:
                results["regressed"] += 1
                if scenario["priority"] == "critical":
                    results["can_deploy"] = False
                    results["blockers"].append({
                        "scenario_id": scenario["scenario_id"],
                        "name": scenario["name"],
                        "reason": f"关键场景退化: "
                                  f"新模型 {detail['new_score']:.3f} < "
                                  f"基线 {detail['baseline_score']:.3f}",
                    })
            if detail["improved"]:
                results["improved"] += 1

        pass_rate = results["passed"] / max(results["total_scenarios"], 1)
        if pass_rate < 0.995:
            results["can_deploy"] = False

        self.db.insert("regression_test_results", {
            "new_model_version": new_model_version,
            "baseline_model_version": baseline_model_version,
            "total_scenarios": results["total_scenarios"],
            "passed": results["passed"],
            "failed": results["failed"],
            "regressed": results["regressed"],
            "pass_rate": pass_rate,
            "can_deploy": results["can_deploy"],
            "details": json.dumps(results["details"], ensure_ascii=False),
            "run_at": now(),
        })

        return results

    def _run_single_scenario(self, scenario, new_model, baseline_model) -> dict:
        """运行单个场景的对比测试"""
        new_result = self.simulator.run(new_model, scenario)
        new_score = self._calculate_score(new_result, scenario)
        new_passed = new_score >= scenario.get("pass_threshold", 0.99)

        baseline_result = self.simulator.run(baseline_model, scenario)
        baseline_score = self._calculate_score(baseline_result, scenario)

        score_delta = new_score - baseline_score
        regressed = score_delta < -0.02
        improved = score_delta > 0.02

        self.db.insert("simulation_results", {
            "version_id": new_model,
            "scenario_id": scenario["scenario_id"],
            "scenario_category": scenario["category"],
            "collision": new_result.get("collision", False),
            "violation": new_result.get("violation", False),
            "comfort_score": new_result.get("comfort_score", 0),
            "completion_score": new_result.get("completion_score", 0),
            "overall_score": new_score,
            "baseline_score": baseline_score,
            "score_delta": score_delta,
            "detail_json": json.dumps(new_result, ensure_ascii=False),
        })

        return {
            "scenario_id": scenario["scenario_id"],
            "scenario_name": scenario["name"],
            "category": scenario["category"],
            "priority": scenario["priority"],
            "new_score": new_score,
            "baseline_score": baseline_score,
            "score_delta": score_delta,
            "new_passed": new_passed,
            "regressed": regressed,
            "improved": improved,
        }

    def _calculate_score(self, result, scenario) -> float:
        """计算场景综合评分"""
        weights = {"collision": 0.50, "violation": 0.20, "comfort": 0.15, "completion": 0.15}
        collision_score = 0.0 if result.get("collision") else 1.0
        violation_score = 0.0 if result.get("violation") else 1.0
        comfort_score = result.get("comfort_score", 0.5)
        completion_score = result.get("completion_score", 0.5)
        return (
            collision_score * weights["collision"]
            + violation_score * weights["violation"]
            + comfort_score * weights["comfort"]
            + completion_score * weights["completion"]
        )
```

## 车队 OTA 部署追踪完整实现

OTA 部署是自动驾驶数据闭环的最后一步。灰度部署策略需要精确追踪每辆车的部署状态，确保任何阶段的异常都能被及时发现和回滚。

```python
class FleetOTADeploymentTracker:
    """
    车队 OTA 部署追踪器。
    追踪部署进度（canary/staged/full）、每辆车的状态、回滚统计。
    """

    DEPLOYMENT_PHASES = {
        "canary": {
            "vehicle_pct": 0.01,
            "min_vehicles": 10,
            "observation_hours": 24,
            "required_pass_rate": 0.99,
        },
        "staged": {
            "vehicle_pct": 0.10,
            "min_vehicles": 100,
            "observation_hours": 72,
            "required_pass_rate": 0.995,
        },
        "full": {
            "vehicle_pct": 1.00,
            "min_vehicles": None,
            "observation_hours": 48,
            "required_pass_rate": 0.998,
        },
    }

    VEHICLE_DEPLOY_STATES = {
        "pending":     "待接收",
        "received":    "已接收推送通知",
        "downloading": "下载中",
        "downloaded":  "下载完成",
        "applying":    "应用中",
        "verified":    "已验证（部署成功）",
        "failed":      "部署失败",
        "rolled_back": "已回滚",
    }

    def __init__(self, db, redis, mqtt_client):
        self.db = db
        self.redis = redis
        self.mqtt_client = mqtt_client

    def create_deployment(
        self, model_version: str, target_fleet_id: str
    ) -> dict:
        """创建新的 OTA 部署计划"""
        deployment_id = f"deploy-{model_version}-{now().strftime('%Y%m%d%H%M')}"
        vehicles = self.db.query(
            "SELECT vehicle_id, current_model_version FROM fleet_vehicles "
            "WHERE fleet_id = %s AND status = 'active'",
            [target_fleet_id]
        )
        total_vehicles = len(vehicles)
        canary_count = max(
            self.DEPLOYMENT_PHASES["canary"]["min_vehicles"],
            int(total_vehicles * self.DEPLOYMENT_PHASES["canary"]["vehicle_pct"])
        )
        staged_count = max(
            self.DEPLOYMENT_PHASES["staged"]["min_vehicles"],
            int(total_vehicles * self.DEPLOYMENT_PHASES["staged"]["vehicle_pct"])
        )

        deployment = {
            "deployment_id": deployment_id,
            "model_version": model_version,
            "fleet_id": target_fleet_id,
            "total_vehicles": total_vehicles,
            "canary_count": canary_count,
            "staged_count": staged_count,
            "current_phase": "canary",
            "status": "in_progress",
            "created_at": now(),
        }
        self.db.insert("ota_deployments", deployment)

        for i, vehicle in enumerate(vehicles):
            phase = "canary" if i < canary_count else (
                "staged" if i < staged_count else "full"
            )
            self.db.insert("ota_vehicle_status", {
                "deployment_id": deployment_id,
                "vehicle_id": vehicle["vehicle_id"],
                "model_version": model_version,
                "previous_version": vehicle["current_model_version"],
                "phase": phase,
                "status": "pending",
                "created_at": now(),
            })

        return deployment

    def get_deployment_progress(self, deployment_id: str) -> dict:
        """获取部署进度详情"""
        deployment = self.db.query_one(
            "SELECT * FROM ota_deployments WHERE deployment_id = %s",
            [deployment_id]
        )
        if not deployment:
            return {"error": "部署不存在"}

        status_counts = self.db.query(
            "SELECT status, COUNT(*) as count "
            "FROM ota_vehicle_status "
            "WHERE deployment_id = %s GROUP BY status",
            [deployment_id]
        )
        status_map = {row["status"]: row["count"] for row in status_counts}

        phase_progress = self.db.query(
            "SELECT phase, status, COUNT(*) as count "
            "FROM ota_vehicle_status "
            "WHERE deployment_id = %s "
            "GROUP BY phase, status ORDER BY phase, status",
            [deployment_id]
        )

        total = deployment["total_vehicles"]
        verified = status_map.get("verified", 0)
        failed = status_map.get("failed", 0)
        rolled_back = status_map.get("rolled_back", 0)
        progress_pct = (verified + failed + rolled_back) / max(total, 1) * 100

        return {
            "deployment_id": deployment_id,
            "model_version": deployment["model_version"],
            "current_phase": deployment["current_phase"],
            "status": deployment["status"],
            "total_vehicles": total,
            "progress_pct": f"{progress_pct:.1f}%",
            "by_status": status_map,
            "by_phase": self._format_phase_progress(phase_progress),
            "can_proceed": self._check_phase_readiness(deployment),
            "rollback_stats": self._get_rollback_stats(deployment_id),
        }

    def update_vehicle_status(
        self, deployment_id, vehicle_id, new_status, detail=None
    ):
        """更新单辆车的部署状态"""
        self.db.update("ota_vehicle_status", {
            "status": new_status,
            "updated_at": now(),
            "detail": json.dumps(detail or {}, ensure_ascii=False),
        }, {
            "deployment_id": deployment_id,
            "vehicle_id": vehicle_id,
        })

        if new_status == "failed":
            self._auto_rollback_vehicle(deployment_id, vehicle_id)

        self.redis.setex(
            f"ota_status:{deployment_id}:{vehicle_id}", 3600, new_status
        )

    def _auto_rollback_vehicle(self, deployment_id, vehicle_id):
        """自动回滚部署失败的车辆"""
        vehicle = self.db.query_one(
            "SELECT * FROM ota_vehicle_status "
            "WHERE deployment_id = %s AND vehicle_id = %s",
            [deployment_id, vehicle_id]
        )
        if not vehicle:
            return
        previous_version = vehicle["previous_version"]
        self.mqtt_client.publish(
            f"cmd/{vehicle_id}/rollback",
            json.dumps({
                "target_version": previous_version,
                "reason": "deployment_failed",
                "deployment_id": deployment_id,
            })
        )
        self.db.update("ota_vehicle_status", {
            "status": "rolled_back",
            "rollback_reason": "deployment_failed",
            "rollback_version": previous_version,
            "rolled_back_at": now(),
        }, {
            "deployment_id": deployment_id,
            "vehicle_id": vehicle_id,
        })

    def _check_phase_readiness(self, deployment) -> dict:
        """检查当前阶段是否可以进入下一阶段"""
        current_phase = deployment["current_phase"]
        phase_config = self.DEPLOYMENT_PHASES.get(current_phase, {})
        phase_vehicles = self.db.query(
            "SELECT status, COUNT(*) as count "
            "FROM ota_vehicle_status "
            "WHERE deployment_id = %s AND phase = %s GROUP BY status",
            [deployment["deployment_id"], current_phase]
        )
        status_map = {row["status"]: row["count"] for row in phase_vehicles}
        total_in_phase = sum(status_map.values())
        verified = status_map.get("verified", 0)
        failed = status_map.get("failed", 0)
        rolled_back = status_map.get("rolled_back", 0)

        completed = verified + failed + rolled_back
        all_completed = completed >= total_in_phase

        observation_hours = phase_config.get("observation_hours", 24)
        elapsed_hours = (now() - deployment["created_at"]).total_seconds() / 3600
        time_ok = elapsed_hours >= observation_hours

        pass_rate = verified / max(verified + failed + rolled_back, 1)
        required_rate = phase_config.get("required_pass_rate", 0.99)
        rate_ok = pass_rate >= required_rate

        return {
            "current_phase": current_phase,
            "all_completed": all_completed,
            "observation_hours_required": observation_hours,
            "elapsed_hours": f"{elapsed_hours:.1f}",
            "time_ok": time_ok,
            "pass_rate": f"{pass_rate:.4f}",
            "required_rate": f"{required_rate:.4f}",
            "rate_ok": rate_ok,
            "can_proceed": all_completed and time_ok and rate_ok,
            "next_phase": self._get_next_phase(current_phase),
        }

    def _get_next_phase(self, current_phase) -> str:
        """获取下一阶段名称"""
        phase_order = ["canary", "staged", "full"]
        try:
            idx = phase_order.index(current_phase)
            return phase_order[idx + 1] if idx + 1 < len(phase_order) else "completed"
        except ValueError:
            return "unknown"

    def _format_phase_progress(self, phase_progress) -> dict:
        """格式化阶段进度"""
        result = {}
        for row in phase_progress:
            phase = row["phase"]
            if phase not in result:
                result[phase] = {"total": 0, "by_status": {}}
            result[phase]["total"] += row["count"]
            result[phase]["by_status"][row["status"]] = row["count"]
        return result

    def _get_rollback_stats(self, deployment_id) -> dict:
        """获取回滚统计"""
        rollback_records = self.db.query(
            "SELECT rollback_reason, COUNT(*) as count "
            "FROM ota_vehicle_status "
            "WHERE deployment_id = %s AND status = 'rolled_back' "
            "GROUP BY rollback_reason",
            [deployment_id]
        )
        total_rolled_back = sum(r["count"] for r in rollback_records)
        return {
            "total_rolled_back": total_rolled_back,
            "by_reason": {r["rollback_reason"]: r["count"] for r in rollback_records},
        }
```

### 更多异常场景

#### 场景：传感器标定数据丢失

```
触发：标定服务器崩溃（磁盘故障），最新标定参数丢失。
      车辆使用的标定版本回退到出厂默认值，3D 感知精度大幅下降。
检测：
  1. 标定数据版本校验：车端标定版本 ≠ 服务器最新版本 → 告警
  2. 感知精度指标：3D 检测平均精度从 92% 降至 78% → 告警
  3. 多车同时报告标定版本异常 → 系统性问题告警
处理：
  1. 从备份服务器恢复最近标定数据（RPO < 1 小时）
  2. 如果备份也丢失 → 使用出厂标定参数（精度降低但安全）
  3. 启动在线标定补偿：利用道路标线和已知地标自动微调
  4. 标定恢复后 → 通过 OTA 推送给受影响车辆
  5. 受影响期间 → 限速 60km/h + 增大安全距离
预防：标定数据三副本存储 + 跨区域异地备份 + 定期恢复演练
```

#### 场景：回归测试失败阻止部署

```
触发：模型更新后 3 个非关键场景测试失败，pass_rate = 99.7% < 99.5%
      → 部署被自动阻止。但这 3 个失败可能是仿真环境问题，非模型问题。
检测：
  1. 回归测试 pass_rate < 99.5% → 自动阻止部署
  2. 失败场景分析：检查是否为 flaky test（非确定性失败）
  3. 失败场景重跑：同一场景跑 3 次，如果 2 次通过 → 标记为 flaky
处理：
  1. 分析失败场景：是否与模型变更相关
     - 相关 → 修复模型后重新测试
     - 不相关 → 可能是仿真环境问题
  2. Flaky test 处理：标记为 flaky，不计入 pass_rate，降级为 advisory
  3. 模型相关失败 → 必须修复后重新跑完整回归
  4. 紧急安全修复 → CTO 审批可跳过非关键场景
  5. 失败场景自动加入优先训练集
预防：Flaky test 自动检测机制（3次重跑）+ 分级场景管理 + 仿真环境稳定性监控
```

#### 场景：车队 OTA 部署卡在 50%

```
触发：OTA 部署到 50% 车辆时停止推进，剩余车辆未收到更新。
      原因可能是：金丝雀阶段发现指标异常（自动暂停），
      或 CDN 带宽不足（下载超时），或部分车辆离线。
检测：
  1. 部署进度停滞 > 30 分钟 → 告警
  2. 部分车辆新版本、部分旧版本 → 版本分叉风险
  3. 部署队列中 pending 状态车辆数 > 0 且不减少
根因分类：
  A. 健康检查阻止：50% 车辆部署后指标异常（如接管率上升）
     → 系统自动暂停，等待人工决策
  B. 网络问题：车辆在信号弱区域，下载超时
     → 下载失败，状态停留在 pending
  C. 车端存储不足：部分车辆磁盘满，无法下载新模型
     → 下载失败，需要清理空间后重试
处理：
  A. 健康检查阻止：
     1. 评估指标异常是否由新模型导致
     2. 是 → 回滚已部署的 50% 车辆
     3. 否（如天气等外部因素）→ 继续部署
  B. 网络问题：
     1. 重试下载（指数退避，最多 3 次）
     2. 降低模型包大小（仅下载增量包）
     3. 等待车辆进入 WiFi 覆盖区域后下载
  C. 存储不足：
     1. 远程清理旧模型缓存
     2. 清理后重新推送
     3. 仍不足 → 标记为"暂不可更新"
预防：
  - 部署前预检查：磁盘空间、网络状态、电池电量
  - 增量更新包（仅传输变更部分，减少带宽需求）
  - 部署进度实时监控 + 超时自动决策（暂停/继续/回滚）
  - 版本分叉最大容忍时间：24 小时（之后强制统一）
```

## 感知模型集成完整实现

```python
class PerceptionModelEnsemble:
    """感知模型集成：多模型投票 + 不一致检测"""

    def predict(self, sensor_data):
        """运行 3 个模型并集成结果"""
        # 1. 并行推理 3 个模型
        results = parallel_execute([
            lambda: self.model_a.predict(sensor_data),
            lambda: self.model_b.predict(sensor_data),
            lambda: self.model_c.predict(sensor_data),
        ])

        # 2. 加权投票
        weighted_detections = self._weighted_vote(results)

        # 3. 不一致检测
        disagreements = self._detect_disagreement(results)
        if disagreements:
            # 模型间分歧 → 降低置信度
            for det in weighted_detections:
                if det["detection_id"] in disagreements:
                    det["confidence"] *= 0.7
                    det["disagreement_flag"] = True

        return {
            "detections": weighted_detections,
            "disagreement_count": len(disagreements),
            "model_agreement_rate": 1 - len(disagreements) / max(len(weighted_detections), 1)
        }

    def _weighted_vote(self, results):
        """加权投票：模型 A 权重 0.4, B 权重 0.35, C 权重 0.25"""
        weights = [0.4, 0.35, 0.25]
        all_detections = []
        for model_result, weight in zip(results, weights):
            for det in model_result["detections"]:
                det["weighted_confidence"] = det["confidence"] * weight
                all_detections.append(det)

        # NMS 去重
        return self._non_max_suppression(all_detections, iou_threshold=0.5)

    def _detect_disagreement(self, results):
        """检测模型间分歧"""
        disagreements = []
        # 如果模型 A 检测到目标但模型 B/C 未检测到 → 分歧
        for i, r1 in enumerate(results):
            for det in r1["detections"]:
                found_by_others = []
                for j, r2 in enumerate(results):
                    if i == j:
                        continue
                    if self._has_matching_detection(det, r2["detections"]):
                        found_by_others.append(j)

                if len(found_by_others) < 1:  # 只有 1 个模型检测到
                    disagreements.append(det["detection_id"])
        return disagreements
```

## HD Map 管理

```python
class HDMapManager:
    """高精地图管理：版本化 + 增量更新"""

    def check_map_version(self, vehicle_id):
        """检查车辆地图版本是否最新"""
        vehicle = self.db.get_vehicle(vehicle_id)
        latest = self.db.query_one(
            "SELECT version FROM hd_map_versions "
            "WHERE region = %s ORDER BY version DESC LIMIT 1",
            vehicle["region"])

        if vehicle["map_version"] != latest["version"]:
            # 版本不一致 → 需要更新
            diff = self._compute_diff(vehicle["map_version"], latest["version"])
            return {"needs_update": True, "diff_size_bytes": diff["size"],
                    "changes": diff["changes"]}
        return {"needs_update": False}

    def _compute_diff(self, old_version, new_version):
        """计算增量差异（车道级变更）"""
        changes = self.db.query(
            "SELECT * FROM hd_map_changes "
            "WHERE version > %s AND version <= %s "
            "ORDER BY version",
            old_version, new_version)
        return {
            "size": sum(c["patch_size_bytes"] for c in changes),
            "changes": [{"version": c["version"], "type": c["change_type"],
                         "lanes_affected": c["lanes_affected"]} for c in changes]
        }
```

## 异常场景补充

### 场景：HD 地图版本不匹配

```
触发：车辆使用旧版地图 → 车道信息与实际不符 → 偏离路线
检测：
  1. 车辆定位与地图车道不匹配 → 告警
  2. 地图版本号与最新版不一致 → 提示更新
处理：
  1. 降级为普通导航模式
  2. 后台下载增量更新
  3. 更新完成 → 重新进入自动驾驶模式
预防：地图版本定期检查 + 强制更新机制
```

### 场景：模型集成不一致

```
触发：3 个模型对同一目标输出完全不同的分类
      → 模型 A 认为是行人，模型 B 认为是车辆，模型 C 未检测到
检测：
  1. 模型间置信度差异 > 0.5 → 不一致
  2. 分类结果完全不同 → 严重不一致
处理：
  1. 降低该目标置信度 × 0.5
  2. 保守处理：按最危险分类处理（行人 > 车辆）
  3. 增大安全距离 + 降低车速
  4. 记录不一致案例 → 用于模型改进
预防：多模型交叉验证 + 不一致检测 + 保守策略
```

## 驾驶行为评分完整实现

```python
class DrivingBehaviorScorer:
    """驾驶行为评分：平顺性、安全性、效率"""

    DIMENSIONS = {
        "smoothness": {"weight": 0.3, "description": "平顺性"},
        "safety": {"weight": 0.5, "description": "安全性"},
        "efficiency": {"weight": 0.2, "description": "效率"},
    }

    def score_trip(self, trip_id):
        """评估单次行程驾驶行为"""
        events = self.db.query(
            "SELECT * FROM driving_events WHERE trip_id = %s ORDER BY timestamp",
            trip_id)

        smoothness = self._score_smoothness(events)
        safety = self._score_safety(events)
        efficiency = self._score_efficiency(events)

        total = (smoothness * 0.3 + safety * 0.5 + efficiency * 0.2)
        return {
            "trip_id": trip_id,
            "smoothness": smoothness,
            "safety": safety,
            "efficiency": efficiency,
            "total_score": total,
            "grade": "A" if total >= 90 else "B" if total >= 75 else "C" if total >= 60 else "D"
        }

    def _score_smoothness(self, events):
        """平顺性评分：急刹车、急加速次数越少越好"""
        hard_brakes = len([e for e in events if e["type"] == "hard_brake"])
        hard_accelerations = len([e for e in events if e["type"] == "hard_acceleration"])
        total_events = len(events) or 1
        # 0 次急刹=100分，每 100 次事件中 10 次急刹=0分
        rate = (hard_brakes + hard_accelerations) / total_events
        return max(0, 100 - rate * 1000)

    def _score_safety(self, events):
        """安全性评分：超速、违规变道、未保持安全距离"""
        speeding_events = len([e for e in events if e["type"] == "speeding"])
        unsafe_lane_changes = len([e for e in events if e["type"] == "unsafe_lane_change"])
        close_follows = len([e for e in events if e["type"] == "close_follow"])
        # 任何安全事件 → 扣分
        deductions = speeding_events * 10 + unsafe_lane_changes * 15 + close_follows * 5
        return max(0, 100 - deductions)

    def _score_efficiency(self, events):
        """效率评分：怠速时间、路线偏差"""
        idle_events = len([e for e in events if e["type"] == "excessive_idle"])
        route_deviations = len([e for e in events if e["type"] == "route_deviation"])
        deductions = idle_events * 5 + route_deviations * 10
        return max(0, 100 - deductions)
```

## 异常场景补充

### 场景：HD 地图版本不匹配

```python
class HDMapVersionGuard:
    """HD 地图版本一致性守卫"""
    def check_version_compatibility(self, vehicle_id, route):
        vehicle_map_version = self.db.get_vehicle_map_version(vehicle_id)
        route_required_version = self.db.get_route_map_version(route)

        if vehicle_map_version < route_required_version:
            # 地图版本不足 → 限制在该路段的自动驾驶能力
            missing_features = self.db.query(
                "SELECT * FROM hd_map_features "
                "WHERE version > %s AND version <= %s",
                vehicle_map_version, route_required_version)
            return {
                "compatible": False,
                "missing_features": missing_features,
                "action": "download_update_or_restrict_to_L2"
            }
        return {"compatible": True}
```

### 场景：感知模型不一致

```
触发：3 个感知模型对同一目标输出完全不同的分类
      → 模型 A: 行人, 模型 B: 车辆, 模型 C: 未检测到
检测：
  1. 模型间分类差异 > 2 种 → 不一致
  2. 置信度差异 > 0.5 → 严重不一致
处理：
  1. 保守策略：按最危险分类处理（行人 > 车辆）
  2. 降低该检测的置信度 × 0.5
  3. 增大安全距离 + 降低车速
  4. 记录不一致案例 → 用于模型改进
预防：多模型投票 + 不一致检测 + 保守策略
```

## 传感器融合流水线完整实现

```python
class SensorFusionPipeline:
    """传感器融合：LiDAR + Camera + Radar → 统一感知"""

    def process_frame(self, vehicle_id, sensor_data):
        """处理一帧传感器数据"""
        # 1. 时间同步（硬件触发 + NTP 校正）
        synced = self._synchronize_timestamps(sensor_data)

        # 2. LiDAR 点云处理
        lidar_objects = self._process_lidar(synced["lidar"])

        # 3. Camera 目标检测
        camera_objects = self._process_camera(synced["camera"])

        # 4. Radar 跟踪关联
        radar_tracks = self._process_radar(synced["radar"])

        # 5. 多传感器融合
        fused_objects = self._fuse_sensors(lidar_objects, camera_objects, radar_tracks)

        # 6. 多目标跟踪
        tracked_objects = self.tracker.update(fused_objects)

        # 7. 传感器健康检查
        health = self._check_sensor_health(sensor_data)

        return {"objects": tracked_objects, "sensor_health": health}

    def _process_lidar(self, point_cloud):
        """LiDAR 点云处理：降采样 → 地面移除 → 聚类"""
        # 体素网格降采样（0.1m 分辨率）
        downsampled = self.voxel_grid_filter(point_cloud, voxel_size=0.1)

        # RANSAC 地面平面估计与移除
        ground_plane, non_ground = self.ransac_plane_fit(
            downsampled, distance_threshold=0.2)

        # 欧几里得聚类提取物体
        clusters = self.euclidean_clustering(
            non_ground, cluster_tolerance=0.5, min_size=10, max_size=5000)

        # 生成边界框
        objects = []
        for cluster in clusters:
            bbox = self._compute_bounding_box(cluster)
            objects.append({
                "id": str(uuid4())[:8],
                "source": "lidar",
                "position": bbox["center"],
                "dimensions": bbox["size"],
                "point_count": len(cluster),
                "confidence": min(1.0, len(cluster) / 100)
            })

        return objects

    def _process_camera(self, image):
        """Camera 目标检测"""
        detections = self.yolo_model.detect(image)
        objects = []
        for det in detections:
            # 像素坐标 → 3D 位置（需要深度估计）
            position_3d = self._pixel_to_3d(det["bbox"], det.get("depth"))
            objects.append({
                "id": str(uuid4())[:8],
                "source": "camera",
                "position": position_3d,
                "class": det["class"],
                "confidence": det["confidence"],
                "bbox_2d": det["bbox"]
            })
        return objects

    def _fuse_sensors(self, lidar_objects, camera_objects, radar_tracks):
        """多传感器融合"""
        fused = []
        used_camera = set()
        used_radar = set()

        # LiDAR 为主传感器，关联 Camera 和 Radar
        for l_obj in lidar_objects:
            best_camera = self._find_nearest_camera(l_obj, camera_objects, used_camera)
            best_radar = self._find_nearest_radar(l_obj, radar_tracks, used_radar)

            fused_obj = {
                "position": l_obj["position"],
                "dimensions": l_obj.get("dimensions"),
                "lidar_confidence": l_obj["confidence"],
            }

            if best_camera:
                fused_obj["class"] = best_camera["class"]
                fused_obj["camera_confidence"] = best_camera["confidence"]
                used_camera.add(best_camera["id"])

            if best_radar:
                fused_obj["velocity"] = best_radar["velocity"]
                fused_obj["radar_confidence"] = best_radar["confidence"]
                used_radar.add(best_radar["id"])

            # 综合置信度
            confidences = [fused_obj["lidar_confidence"]]
            if "camera_confidence" in fused_obj:
                confidences.append(fused_obj["camera_confidence"])
            fused_obj["confidence"] = sum(confidences) / len(confidences)

            fused.append(fused_obj)

        # 添加仅 Camera 检测到的物体（远处小目标 LiDAR 可能漏检）
        for c_obj in camera_objects:
            if c_obj["id"] not in used_camera:
                fused.append({
                    "position": c_obj["position"],
                    "class": c_obj["class"],
                    "confidence": c_obj["confidence"] * 0.7,
                    "source": "camera_only"
                })

        return fused

    def _synchronize_timestamps(self, sensor_data):
        """时间戳同步"""
        # 使用硬件触发时间作为基准
        base_time = sensor_data.get("hardware_trigger_time", now())
        synced = {}
        for sensor, data in sensor_data.items():
            if sensor == "hardware_trigger_time":
                continue
            offset = self._get_sensor_offset(sensor)
            data["adjusted_timestamp"] = base_time + timedelta(microseconds=offset)
            synced[sensor] = data
        return synced

    def _check_sensor_health(self, sensor_data):
        """传感器健康检查"""
        health = {}
        for sensor in ["lidar", "camera", "radar"]:
            data = sensor_data.get(sensor)
            if not data:
                health[sensor] = "offline"
            elif data.get("error_rate", 0) > 0.1:
                health[sensor] = "degraded"
            else:
                health[sensor] = "healthy"
        return health
```

## 自动驾驶仿真框架

```python
class AutonomousDrivingSimulator:
    """自动驾驶仿真：场景定义 + 传感器模拟 + 回归测试"""

    def run_scenario(self, scenario):
        """运行仿真场景"""
        # 1. 初始化仿真环境
        env = self._init_environment(scenario)

        # 2. 传感器模拟
        sensor_sim = SensorSimulator(env)

        # 3. 运行仿真循环
        results = {"frames": [], "events": []}
        for tick in range(scenario["duration_ticks"]):
            # 模拟传感器输出
            sensor_data = sensor_sim.generate(tick)

            # 运行自动驾驶系统
            ad_output = self.autonomous_system.process(sensor_data)

            # 更新环境（NPC 行为）
            env = self._update_environment(env, ad_output, tick)

            # 记录结果
            results["frames"].append({
                "tick": tick,
                "ego_position": env["ego"]["position"],
                "ego_velocity": env["ego"]["velocity"],
                "decision": ad_output["decision"],
                "nearby_objects": len(ad_output.get("objects", []))
            })

            # 检测碰撞
            if env.get("collision"):
                results["events"].append({
                    "tick": tick, "type": "collision",
                    "description": f"碰撞: {env['collision']['with']}"
                })
                break

        # 4. 评分
        score = self._score_scenario(results, scenario)

        return {"scenario": scenario["name"], "score": score, "results": results}

    def _score_scenario(self, results, scenario):
        """场景评分"""
        score = 100

        # 碰撞 → 0 分
        if any(e["type"] == "collision" for e in results["events"]):
            score = 0

        # 车道偏移 > 0.3m → 扣分
        max_deviation = max(
            abs(f.get("lane_deviation", 0)) for f in results["frames"])
        if max_deviation > 0.3:
            score -= (max_deviation - 0.3) * 50

        # 舒适度：加速度 > 3m/s² → 扣分
        for i in range(1, len(results["frames"])):
            accel = abs(results["frames"][i]["ego_velocity"] -
                       results["frames"][i-1]["ego_velocity"]) / 0.1
            if accel > 3.0:
                score -= (accel - 3.0) * 5

        return max(0, round(score, 1))

    def run_regression_suite(self):
        """运行回归测试套件"""
        critical_scenarios = [
            {"name": "紧急制动60km/h", "ego_speed": 60, "obstacle_distance": 30},
            {"name": "行人横穿", "pedestrian_crossing": True, "ego_speed": 40},
            {"name": "高速变道120km/h", "ego_speed": 120, "lane_change": True},
            {"name": "施工区导航", "construction_zone": True},
        ]

        results = []
        for scenario in critical_scenarios:
            result = self.run_scenario(scenario)
            results.append(result)
            if result["score"] < 80:
                self.alert(f"仿真失败: {scenario['name']} 得分 {result['score']}")

        return {"total": len(results),
                "passed": sum(1 for r in results if r["score"] >= 80),
                "results": results}
```

## 异常场景补充

### 场景：Camera 与 LiDAR 位置分歧

```
触发：Camera 检测到物体在 50m，LiDAR 检测到同一物体在 55m → 5m 偏差
检测：
  1. 融合时同一物体位置差 > 2m → 分歧
  2. 标定参数可能偏移 → 需要重新标定
处理：
  1. 使用 LiDAR 位置（测距更准）
  2. 记录偏差 → 触发重新标定
  3. 标定前降级：仅使用 LiDAR 测距
预防：定期自动标定检查 + 传感器置信度加权
```

### 场景：仿真通过但实车失败

```
触发：仿真场景全部通过 → 实际道路测试同场景失败
原因：仿真传感器噪声模型与实际不符
检测：
  1. 仿真得分 100 但实车同一场景碰撞
  2. 对比仿真 vs 实车的传感器原始数据差异
处理：
  1. 收集实车传感器噪声特征
  2. 更新仿真噪声模型（增加雨雾、逆光等边缘场景）
  3. 重新运行回归测试
预防：仿真噪声模型定期校准 + 实车-仿真对比验证
```

## V2X 通信完整实现

```python
class V2XCommunicationService:
    """V2X 通信：V2V + V2I + V2P 消息处理"""

    def process_bsm(self, vehicle_id, bsm_message):
        """处理 V2V 基础安全消息（10Hz）"""
        # 1. 验证消息签名
        if not self.pki.verify(bsm_message["signature"],
                               bsm_message["certificate"],
                               bsm_message["payload"]):
            return {"status": "invalid_signature"}

        # 2. 解析消息
        other_vehicle = {
            "vehicle_id": bsm_message["vehicle_id"],
            "position": (bsm_message["lat"], bsm_message["lon"]),
            "speed_mps": bsm_message["speed"],
            "heading_deg": bsm_message["heading"],
            "brake_status": bsm_message.get("brake_applied", False),
            "timestamp": bsm_message["timestamp"]
        }

        # 3. 碰撞风险评估
        risk = self._assess_collision_risk(vehicle_id, other_vehicle)

        # 4. 高风险 → 生成警告
        if risk["level"] in ["high", "critical"]:
            return {
                "status": "warning",
                "warning_type": risk["type"],
                "time_to_collision_s": risk["ttc"],
                "recommended_action": risk["action"]
            }

        return {"status": "ok", "risk_level": risk["level"]}

    def process_rsu_message(self, vehicle_id, rsu_message):
        """处理 V2I 路侧单元消息（信号灯状态）"""
        phase = rsu_message["signal_phase"]  # green/yellow/red
        time_to_change = rsu_message["time_to_change"]

        current_speed = self._get_vehicle_speed(vehicle_id)
        distance_to_intersection = self._get_distance_to_intersection(vehicle_id)

        # 判断是否能安全通过
        if phase == "green" and time_to_change < 5:
            if distance_to_intersection / current_speed > time_to_change:
                return {"action": "decelerate", "reason": "信号灯即将变黄"}

        if phase == "yellow":
            # 黄灯决策：停还是过
            stopping_distance = current_speed ** 2 / (2 * 6.0)  # 减速度 6m/s²
            if distance_to_intersection > stopping_distance + 5:
                return {"action": "stop", "reason": "黄灯安全停车距离内"}
            else:
                return {"action": "proceed", "reason": "无法安全停车"}

        return {"action": "proceed"}

    def rotate_pseudonym(self, vehicle_id):
        """轮换伪名（隐私保护，每 5 分钟）"""
        new_pseudonym = self.pki.generate_pseudonym()
        self.redis.setex(f"v2x_pseudonym:{vehicle_id}", 300, new_pseudonym)
        return new_pseudonym
```

## 自动驾驶数据记录与回放

```python
class AutonomousDataRecorder:
    """自动驾驶数据记录：EDR + 全量日志 + 取证回放"""

    EDR_BUFFER_SECONDS = 30  # 事件前 30 秒
    EDR_POST_SECONDS = 10    # 事件后 10 秒

    def record_event(self, vehicle_id, event_type, event_data):
        """记录关键事件"""
        # 从环形缓冲区提取前后数据
        pre_event = self.ring_buffer.read_last(vehicle_id, self.EDR_BUFFER_SECONDS)

        edr_record = {
            "vehicle_id": vehicle_id,
            "event_type": event_type,  # collision / hard_brake / takeover
            "event_timestamp": now(),
            "pre_event_data": pre_event,
            "event_data": event_data,
            "post_event_data": [],  # 持续记录 10 秒
        }

        # 开始记录事件后数据
        self.recording_sessions[vehicle_id] = {
            "start": now(), "duration": self.EDR_POST_SECONDS,
            "record": edr_record
        }

        # 保存 EDR 记录
        self.db.insert("edr_records", {
            "vehicle_id": vehicle_id,
            "event_type": event_type,
            "event_timestamp": now(),
            "data": json.dumps(edr_record, default=str),
            "retention_until": now() + timedelta(days=365 * 3)  # NHTSA 3 年
        })

    def replay_incident(self, edr_id):
        """取证回放：重建事故时间线"""
        edr = self.db.get_edr(edr_id)

        timeline = []
        all_data = edr["pre_event_data"] + [edr["event_data"]] + edr["post_event_data"]

        for frame in all_data:
            entry = {
                "timestamp": frame["timestamp"],
                "speed_kmh": frame.get("speed", 0) * 3.6,
                "steering_angle_deg": frame.get("steering_angle", 0),
                "brake_applied": frame.get("brake", False),
                "accelerator_pct": frame.get("accelerator", 0),
                "detected_objects": len(frame.get("objects", [])),
                "decision": frame.get("decision", "unknown"),
                "latitude": frame.get("lat"),
                "longitude": frame.get("lon"),
            }
            timeline.append(entry)

        return {"edr_id": edr_id, "event_type": edr["event_type"],
                "timeline": timeline,
                "summary": self._summarize_incident(timeline)}
```

## 异常场景补充

### 场景：V2X 证书吊销列表过期

```
触发：V2X 消息签名验证失败 → 证书被吊销但 CRL 未更新 → 合法消息被拒
检测：
  1. V2V 消息拒绝率突增 → 可能 CRL 过期
  2. CRL 最后更新时间 > 24 小时 → 过期
处理：
  1. 紧急更新 CRL
  2. 过渡期降低验证严格度（仅拒绝已知恶意证书）
预防：CRL 每日自动更新 + 更新失败告警 + 本地缓存
```

### 场景：EDR 存储满覆盖关键数据

```
触发：EDR 环形缓冲区写满 → 新事件覆盖旧事件 → 未上传的 EDR 丢失
检测：
  1. EDR 存储使用率 > 90% → 即将写满
  2. 旧 EDR 记录被覆盖 → 数据丢失
处理：
  1. 加速 EDR 上传到云端
  2. 增大本地存储容量
  3. 关键事件优先上传（碰撞 > 紧急制动 > 接管）
预防：分级上传 + 存储容量监控 + 自动扩容
```

## 高精地图服务完整实现

```python
class HDMapService:
    """高精地图服务：车道级地图 + 增量更新 + 在线修正"""

    def get_local_map(self, lat, lng, radius_m=500):
        """获取局部高精地图"""
        # 1. 计算瓦片坐标
        tile_id = self._latlng_to_tile(lat, lng, zoom=16)

        # 2. 从缓存获取
        cached = self.redis.get(f"hdmap:tile:{tile_id}")
        if cached:
            return json.loads(cached)

        # 3. 从数据库查询
        map_data = self.db.query(
            "SELECT * FROM hdmap_elements "
            "WHERE ST_DWithin(location, ST_MakePoint(%s, %s), %s) "
            "AND status = 'active' "
            "ORDER BY element_type, distance",
            lng, lat, radius_m / 111000)

        result = {
            "center": {"lat": lat, "lng": lng},
            "radius_m": radius_m,
            "lanes": [e for e in map_data if e["element_type"] == "lane"],
            "signs": [e for e in map_data if e["element_type"] == "sign"],
            "barriers": [e for e in map_data if e["element_type"] == "barrier"],
            "junctions": [e for e in map_data if e["element_type"] == "junction"],
            "version": self._get_tile_version(tile_id),
            "timestamp": now().isoformat()
        }

        # 缓存（5 分钟）
        self.redis.setex(f"hdmap:tile:{tile_id}", 300, json.dumps(result))

        return result

    def apply_incremental_update(self, update_batch):
        """应用增量更新（差量地图）"""
        version = update_batch["target_version"]

        for change in update_batch["changes"]:
            if change["operation"] == "add":
                self.db.insert("hdmap_elements", {
                    "element_id": change["element_id"],
                    "element_type": change["element_type"],
                    "geometry": change["geometry"],
                    "attributes": json.dumps(change["attributes"]),
                    "version": version,
                    "status": "active"
                })
            elif change["operation"] == "modify":
                self.db.update("hdmap_elements",
                    {"geometry": change["geometry"],
                     "attributes": json.dumps(change["attributes"]),
                     "version": version},
                    {"element_id": change["element_id"]})
            elif change["operation"] == "delete":
                self.db.update("hdmap_elements",
                    {"status": "deleted", "version": version},
                    {"element_id": change["element_id"]})

            # 清除受影响瓦片的缓存
            tiles = self._get_affected_tiles(change["geometry"])
            for tile_id in tiles:
                self.redis.delete(f"hdmap:tile:{tile_id}")

        self.db.insert("hdmap_versions", {
            "version": version,
            "change_count": len(update_batch["changes"]),
            "applied_at": now()
        })

    def submit_map_correction(self, vehicle_id, correction):
        """提交地图修正（众包）"""
        correction_id = str(uuid4())

        # 验证修正合理性
        nearby_elements = self.get_local_map(
            correction["lat"], correction["lng"], radius_m=10)

        # 同一位置多人提交相同修正 → 高置信度
        similar = self.db.query(
            "SELECT * FROM map_corrections "
            "WHERE ST_DWithin(location, ST_MakePoint(%s, %s), 0.0001) "
            "AND correction_type = %s AND status = 'pending'",
            correction["lng"], correction["lat"],
            correction["correction_type"])

        confidence = min(1.0, len(similar) * 0.3 + 0.1)

        self.db.insert("map_corrections", {
            "correction_id": correction_id,
            "vehicle_id": vehicle_id,
            "correction_type": correction["correction_type"],
            "description": correction["description"],
            "location": f"POINT({correction['lng']} {correction['lat']})",
            "confidence": confidence,
            "status": "auto_approved" if confidence >= 0.7 else "pending_review",
            "submitted_at": now()
        })

        if confidence >= 0.7:
            self._auto_apply_correction(correction_id)

        return {"correction_id": correction_id, "confidence": confidence}
```

## OTA 远程升级安全

```python
class AutonomousOTAService:
    """自动驾驶 OTA：安全验证 + 灰度 + 回滚"""

    def push_update(self, vehicle_ids, firmware_version, priority="normal"):
        """推送 OTA 更新"""
        # 1. 安全校验
        firmware = self.db.get_firmware(firmware_version)
        if not self._verify_signature(firmware):
            raise SecurityError("固件签名验证失败")

        # 2. 兼容性检查
        for vid in vehicle_ids:
            vehicle = self.db.get_vehicle(vid)
            if not self._check_compatibility(vehicle, firmware):
                self.alert(f"车辆 {vid} 与固件 {firmware_version} 不兼容")
                vehicle_ids.remove(vid)

        # 3. 安全停车条件检查
        for vid in vehicle_ids:
            state = self._get_vehicle_state(vid)
            if state["speed_kmh"] > 0:
                # 车辆行驶中 → 排队等待停车
                self.redis.set(f"ota_pending:{vid}", firmware_version)
                continue

            # 4. 推送更新
            self._push_to_vehicle(vid, firmware)

        return {"pushed": len(vehicle_ids), "pending_parking": len(vehicle_ids) - len([v for v in vehicle_ids if self._get_vehicle_state(v)["speed_kmh"] == 0])}

    def _push_to_vehicle(self, vehicle_id, firmware):
        """推送到车辆"""
        self.mqtt_client.publish(f"vehicle/{vehicle_id}/ota", json.dumps({
            "action": "update",
            "firmware_id": firmware["id"],
            "version": firmware["version"],
            "download_url": firmware["download_url"],
            "sha256": firmware["sha256"],
            "signature": firmware["signature"],
            "target_partition": "B",
            "pre_conditions": ["parked", "charging_or_battery_gt_30"],
            "estimated_duration_minutes": 15
        }))

        self.db.insert("ota_history", {
            "vehicle_id": vehicle_id,
            "firmware_version": firmware["version"],
            "status": "downloading",
            "started_at": now()
        })
```

## 异常场景补充

### 场景：高精地图瓦片加载延迟

```
触发：车辆进入新区域 → 地图瓦片加载 > 2 秒 → 路径规划缺失车道信息
检测：
  1. 地图请求延迟 > 500ms → 告警
  2. 车辆报告"地图不可用" → 严重
处理：
  1. 车辆预加载前方 2km 地图（基于路线预测）
  2. 地图不可用 → 降级到标准导航
  3. 修复地图服务延迟
预防：路线预加载 + 本地地图缓存 + 降级导航
```

### 场景：OTA 推送到行驶中的车辆

```
触发：OTA 推送未检查车辆状态 → 行驶中收到更新 → 自动重启 → 失控
检测：
  1. OTA 推送时车辆速度 > 0 → 严重违规
  2. 车辆意外重启 → OTA 问题
处理：
  1. OTA 系统必须确认车辆已停车
  2. 车辆端拒绝行驶中安装更新
  3. 双重验证（云端 + 车端）
预防：停车条件强制检查 + 车端拒绝 + 双重验证
```

## 自动驾驶合规与监管完整实现

```python
class AutonomousComplianceService:
    """自动驾驶合规：法规跟踪 + 运营许可 + 事故报告"""

    REGULATORY_REQUIREMENTS = {
        "CN": {
            "test_permit": "道路测试许可",
            "operation_permit": "示范运营许可",
            "safety_operator": True,  # 需要安全员
            "data_localization": True,  # 数据本地化
            "accident_report_hours": 24,
        },
        "US_CA": {
            "test_permit": "DMV Testing Permit",
            "operation_permit": "Deployment Permit",
            "safety_operator": False,
            "data_localization": False,
            "accident_report_hours": 10,
        },
    }

    def check_permit_status(self, vehicle_id, jurisdiction):
        """检查运营许可状态"""
        requirements = self.REGULATORY_REQUIREMENTS.get(jurisdiction, {})
        permits = self.db.query(
            "SELECT * FROM vehicle_permits "
            "WHERE vehicle_id = %s AND jurisdiction = %s "
            "AND expires_at > NOW()", vehicle_id, jurisdiction)

        status = {
            "vehicle_id": vehicle_id,
            "jurisdiction": jurisdiction,
            "can_operate": True,
            "missing_permits": [],
            "expiring_soon": [],
        }

        required_permits = [requirements.get("test_permit"),
                           requirements.get("operation_permit")]
        for req in required_permits:
            if req and not any(p["permit_type"] == req for p in permits):
                status["can_operate"] = False
                status["missing_permits"].append(req)

        for p in permits:
            days_to_expiry = (p["expires_at"] - now()).days
            if days_to_expiry < 30:
                status["expiring_soon"].append({
                    "permit_type": p["permit_type"],
                    "expires_at": p["expires_at"].isoformat(),
                    "days_remaining": days_to_expiry
                })

        return status

    def file_accident_report(self, incident_data):
        """提交事故报告（法规要求）"""
        jurisdiction = incident_data["jurisdiction"]
        requirements = self.REGULATORY_REQUIREMENTS.get(jurisdiction, {})
        report_deadline_hours = requirements.get("accident_report_hours", 24)

        report_id = str(uuid4())
        self.db.insert("accident_reports", {
            "report_id": report_id,
            "vehicle_id": incident_data["vehicle_id"],
            "incident_time": incident_data["incident_time"],
            "location_lat": incident_data["location_lat"],
            "location_lng": incident_data["location_lng"],
            "severity": incident_data["severity"],
            "injuries": incident_data.get("injuries", 0),
            "fatalities": incident_data.get("fatalities", 0),
            "description": incident_data["description"],
            "edr_data_reference": incident_data.get("edr_id"),
            "jurisdiction": jurisdiction,
            "report_deadline": now() + timedelta(hours=report_deadline_hours),
            "status": "filed",
            "filed_at": now()
        })

        # 通知监管机构
        self._notify_regulator(jurisdiction, report_id)

        # 严重事故 → 暂停运营
        if incident_data["severity"] in ["serious", "fatal"]:
            self._suspend_operations(incident_data["vehicle_id"])

        return {"report_id": report_id,
                "deadline": (now() + timedelta(hours=report_deadline_hours)).isoformat()}

    def check_data_localization(self, vehicle_id, jurisdiction):
        """检查数据本地化合规"""
        requirements = self.REGULATORY_REQUIREMENTS.get(jurisdiction, {})

        if not requirements.get("data_localization"):
            return {"compliant": True, "reason": "无本地化要求"}

        # 检查数据存储位置
        data_locations = self.db.query(
            "SELECT data_type, storage_region FROM data_storage_map "
            "WHERE vehicle_id = %s", vehicle_id)

        non_local = [d for d in data_locations
                    if d["storage_region"] != self._get_jurisdiction_region(jurisdiction)]

        if non_local:
            return {"compliant": False,
                    "non_local_data": non_local,
                    "action": "需将数据迁移至本地存储"}

        return {"compliant": True}
```

## 异常场景补充

### 场景：运营许可过期但车辆仍在运行

```
触发：运营许可过期 → 车辆未停止 → 违规运营 → 法律风险
检测：
  1. 许可过期但车辆仍在调度 → 违规
  2. 许可到期前 30 天提醒 → 预警
处理：
  1. 立即停止该车辆的自动驾驶运营
  2. 切换到人类驾驶模式
  3. 紧急续期申请
预防：许可到期自动停运 + 30 天预警 + 自动续期
```

### 场景：数据本地化合规被违反

```
触发：数据跨境传输 → 违反本地化要求 → 监管处罚
检测：
  1. 数据存储区域与运营区域不匹配 → 违规
  2. 数据访问日志显示跨境访问 → 违规
处理：
  1. 立即停止跨境数据传输
  2. 将数据迁移至本地存储
  3. 向监管机构报告
预防：数据存储区域强制配置 + 跨境访问监控 + 自动迁移
```

## 自动驾驶仿真测试平台完整实现

```python
class SimulationTestPlatform:
    """仿真测试：场景库 + 回放 + 覆盖率"""

    SCENARIO_CATEGORIES = {
        "intersection": "交叉路口场景",
        "highway_merge": "高速合流场景",
        "pedestrian": "行人交互场景",
        "adverse_weather": "恶劣天气场景",
        "emergency": "紧急情况场景",
        "construction_zone": "施工区域场景",
    }

    def run_simulation(self, scenario_id, autopilot_version, iterations=100):
        """运行仿真测试"""
        scenario = self.db.get_scenario(scenario_id)

        results = []
        for i in range(iterations):
            # 1. 初始化场景（加入随机扰动）
            seed = random.randint(0, 999999)
            variation = self._apply_variation(scenario, seed)

            # 2. 运行仿真
            sim_result = self.simulator.run(
                scenario=variation,
                autopilot_version=autopilot_version,
                duration_seconds=scenario["duration_seconds"])

            # 3. 评估结果
            evaluation = self._evaluate_simulation(sim_result, scenario)

            results.append({
                "iteration": i + 1,
                "seed": seed,
                "outcome": evaluation["outcome"],  # pass / fail / marginal
                "metrics": evaluation["metrics"],
                "violations": evaluation["violations"]
            })

        # 4. 统计汇总
        pass_count = sum(1 for r in results if r["outcome"] == "pass")
        fail_count = sum(1 for r in results if r["outcome"] == "fail")

        self.db.insert("simulation_results", {
            "scenario_id": scenario_id,
            "autopilot_version": autopilot_version,
            "iterations": iterations,
            "pass_count": pass_count,
            "fail_count": fail_count,
            "pass_rate": round(pass_count / iterations, 4),
            "results": json.dumps(results),
            "run_at": now()
        })

        return {"scenario_id": scenario_id,
                "pass_rate": round(pass_count / iterations, 4),
                "fail_details": [r for r in results if r["outcome"] == "fail"]}

    def check_coverage(self, autopilot_version):
        """检查场景覆盖率"""
        all_scenarios = self.db.query(
            "SELECT category, COUNT(*) as total FROM scenarios GROUP BY category")

        tested = self.db.query(
            "SELECT s.category, COUNT(DISTINCT sr.scenario_id) as tested "
            "FROM simulation_results sr "
            "JOIN scenarios s ON sr.scenario_id = s.id "
            "WHERE sr.autopilot_version = %s "
            "GROUP BY s.category", autopilot_version)

        coverage = {}
        for cat in all_scenarios:
            tested_count = next((t["tested"] for t in tested if t["category"] == cat["category"]), 0)
            coverage[cat["category"]] = {
                "total": cat["total"],
                "tested": tested_count,
                "coverage_pct": round(tested_count / cat["total"] * 100, 1)
            }

        overall_pct = sum(c["tested"] for c in coverage.values()) / \
                      max(sum(c["total"] for c in coverage.values()), 1) * 100

        return {"overall_coverage_pct": round(overall_pct, 1),
                "by_category": coverage,
                "untested_scenarios": [
                    cat for cat, c in coverage.items() if c["coverage_pct"] < 80]}

    def replay_failure_scenario(self, scenario_id, failing_seed):
        """回放失败场景（调试）"""
        scenario = self.db.get_scenario(scenario_id)
        variation = self._apply_variation(scenario, failing_seed)

        # 详细回放（每帧记录）
        replay = self.simulator.run(
            scenario=variation,
            autopilot_version="latest",
            duration_seconds=scenario["duration_seconds"],
            record_frames=True)

        return {
            "scenario_id": scenario_id,
            "seed": failing_seed,
            "frame_count": len(replay.get("frames", [])),
            "decision_timeline": replay.get("decision_log", []),
            "failure_point": self._find_failure_point(replay)
        }
```

## 异常场景补充

### 场景：仿真结果与实车表现差异大

```
触发：仿真通过率 95% → 实车测试通过率 70% → 仿真不准确
检测：
  1. Sim-to-Real gap > 20% → 仿真模型需要改进
  2. 特定场景仿真/实车差异显著 → 模型缺陷
处理：
  1. 用实车数据校准仿真模型
  2. 增加真实场景扰动（行人随机行为）
  3. 降低仿真置信度权重
预防：实车校准 + 真实扰动 + 多源验证
```

### 场景：仿真覆盖率虚假高

```
触发：覆盖率 90% → 但每个场景只跑了 1 次 → 统计意义不足 → 覆盖率虚假
检测：
  1. 场景平均迭代次数 < 10 → 覆盖率虚假
  2. 不同种子差异大 → 需要更多迭代
处理：
  1. 要求每个场景至少 100 次迭代
  2. 覆盖率 = 已测试场景 × 通过率
  3. 加入置信度指标
预防：最低迭代次数要求 + 通过率加权 + 置信度指标
```

## 自动驾驶数据脱敏与隐私保护完整实现

```python
class AutonomousDataPrivacyService:
    """数据脱敏与隐私：人脸模糊 + 位置泛化 + 音频过滤"""

    SENSITIVITY_LEVELS = {
        "raw": {"face_blur": False, "location_precision": 6, "audio_filter": False},
        "internal": {"face_blur": True, "location_precision": 4, "audio_filter": True},
        "external": {"face_blur": True, "location_precision": 2, "audio_filter": True},
    }

    def process_data_for_release(self, dataset_id, target_level="external"):
        """数据脱敏处理"""
        config = self.SENSITIVITY_LEVELS[target_level]
        dataset = self.db.get_dataset(dataset_id)

        processed_records = 0
        for record in dataset["records"]:
            # 1. 人脸模糊
            if config["face_blur"] and record.get("image_urls"):
                for img_url in record["image_urls"]:
                    self._blur_faces(img_url)

            # 2. 位置泛化（减少精度）
            if config["location_precision"] < 6:
                if record.get("latitude"):
                    record["latitude"] = round(record["latitude"],
                        config["location_precision"])
                    record["longitude"] = round(record["longitude"],
                        config["location_precision"])

            # 3. 音频过滤（去除对话内容）
            if config["audio_filter"] and record.get("audio_urls"):
                self._filter_speech(record["audio_urls"])

            # 4. 车牌号脱敏
            if record.get("license_plate"):
                record["license_plate"] = self._mask_plate(record["license_plate"])

            # 5. 时间泛化（日期保留，时间模糊到小时）
            if config["location_precision"] <= 2:
                if record.get("timestamp"):
                    ts = record["timestamp"]
                    record["timestamp"] = ts.replace(minute=0, second=0)

            processed_records += 1

        self.db.update("datasets",
            {"privacy_level": target_level,
             "processed_at": now(),
             "processed_records": processed_records},
            {"id": dataset_id})

        return {"dataset_id": dataset_id, "target_level": target_level,
                "records_processed": processed_records}

    def _blur_faces(self, image_url):
        """人脸模糊处理"""
        # 检测人脸区域
        faces = self.vision_model.detect_faces(image_url)
        for face in faces:
            # 高斯模糊人脸区域
            self.image_processor.blur_region(image_url,
                face["bbox"], blur_strength=25)

    def _mask_plate(self, plate):
        """车牌号脱敏"""
        # 保留省份和最后一位，中间用 * 替换
        if len(plate) >= 7:
            return plate[0] + "***" + plate[-1]
        return "***"

    def _filter_speech(self, audio_urls):
        """音频过滤（保留环境音，过滤人声）"""
        for url in audio_urls:
            # 分离人声和环境音
            separated = self.audio_processor.separate_vocal(url)
            # 只保留环境音
            self.audio_processor.save(url, separated["background"])

    def audit_data_usage(self, dataset_id):
        """审计数据使用"""
        accesses = self.db.query(
            "SELECT * FROM dataset_access_log "
            "WHERE dataset_id = %s "
            "ORDER BY accessed_at DESC LIMIT 100", dataset_id)

        # 检查是否有超出授权范围的使用
        dataset = self.db.get_dataset(dataset_id)
        authorized_users = json.loads(dataset.get("authorized_users", "[]"))

        unauthorized = [a for a in accesses
            if a["user_id"] not in authorized_users]

        return {"dataset_id": dataset_id,
                "total_accesses": len(accesses),
                "unauthorized_accesses": len(unauthorized),
                "unauthorized_details": unauthorized[:5]}
```

## 异常场景补充

### 场景：脱敏处理遗漏人脸

```
触发：人脸检测模型漏检侧脸 → 未模糊 → 数据发布后隐私泄露
检测：
  1. 人工审查发现未模糊人脸 → 漏检
  2. 发布后被第三方发现人脸 → 严重
处理：
  1. 增加人脸检测模型灵敏度
  2. 使用多模型交叉检测（降低漏检率）
  3. 发布前人工审查
预防：多模型检测 + 人工审查 + 发布审批
```

### 场景：位置泛化后仍可追踪

```
触发：位置精度降到小数点后 2 位 → 但结合时间+路线仍可唯一识别 → 隐私泄露
检测：
  1. 泛化后位置仍可唯一识别用户 → 泛化不足
  2. 第三方报告可从数据中追踪特定车辆 → 严重
处理：
  1. 进一步泛化（精度降到小数点后 1 位）
  2. 加入随机偏移（±500m）
  3. 删除连续轨迹，只保留离散点
预防：进一步泛化 + 随机偏移 + 轨迹离散化
```

## 自动驾驶远程接管完整实现

```python
class RemoteTakeoverService:
    """远程接管：接管请求 + 视频流 + 控制指令 + 安全边界"""

    TAKEOVER_STATES = {
        "idle": "空闲（可接管）",
        "requesting": "请求中",
        "active": "接管中",
        "handing_back": "交还中",
        "emergency": "紧急接管",
    }

    def request_takeover(self, vehicle_id, operator_id, reason):
        """请求远程接管"""
        # 1. 检查车辆状态
        vehicle = self.db.get_vehicle(vehicle_id)
        if vehicle["autonomous_mode"] != "active":
            return {"status": "not_in_auto_mode"}

        # 2. 检查是否有其他操作员正在接管
        current = self.redis.get(f"takeover:{vehicle_id}")
        if current:
            return {"status": "already_taken_over",
                    "operator": json.loads(current)["operator_id"]}

        # 3. 创建接管请求
        takeover_id = str(uuid4())
        self.db.insert("remote_takeovers", {
            "takeover_id": takeover_id,
            "vehicle_id": vehicle_id,
            "operator_id": operator_id,
            "reason": reason,
            "status": "requesting",
            "created_at": now()
        })

        # 4. 建立视频流连接
        video_stream = self.video_service.create_stream(
            source=vehicle_id,
            target=f"operator_{operator_id}",
            quality="high",
            latency_requirement_ms=200)

        # 5. 建立控制通道
        control_channel = self.control_service.create_channel(
            vehicle_id=vehicle_id,
            operator_id=operator_id,
            max_latency_ms=100)

        # 6. 锁定接管状态
        self.redis.setex(f"takeover:{vehicle_id}", 3600, json.dumps({
            "takeover_id": takeover_id,
            "operator_id": operator_id,
            "started_at": now().isoformat()
        }))

        # 7. 通知车内乘客
        self.notification.send_in_vehicle(vehicle_id,
            "车辆已切换到远程人工驾驶模式")

        return {"takeover_id": takeover_id, "status": "active",
                "video_stream_url": video_stream["url"],
                "control_channel_id": control_channel["channel_id"]}

    def send_control_command(self, vehicle_id, operator_id, command):
        """发送控制指令"""
        # 1. 验证接管状态
        takeover_data = self.redis.get(f"takeover:{vehicle_id}")
        if not takeover_data:
            raise TakeoverNotActiveError("车辆未被接管")

        takeover = json.loads(takeover_data)
        if takeover["operator_id"] != operator_id:
            raise PermissionDeniedError("非接管操作员")

        # 2. 安全边界检查
        vehicle_state = self._get_vehicle_state(vehicle_id)

        if command["type"] == "steering":
            # 转向角限制（±30度）
            angle = command["value"]
            if abs(angle) > 30:
                command["value"] = max(-30, min(30, angle))

        elif command["type"] == "throttle":
            # 加速限制（最大 50%）
            throttle = command["value"]
            if throttle > 0.5:
                command["value"] = 0.5

            # 速度限制（最高 60 km/h 接管模式）
            if vehicle_state["speed_kmh"] > 60:
                command["value"] = 0  # 已超速 → 不允许加速

        elif command["type"] == "brake":
            # 刹车限制（最大 80% 避免急刹）
            brake = command["value"]
            if brake > 0.8:
                command["value"] = 0.8

        # 3. 发送指令
        command["timestamp"] = now().isoformat()
        command["sequence"] = self.redis.incr(f"cmd_seq:{vehicle_id}")

        self.control_service.send(vehicle_id, command)

        # 4. 记录指令日志
        self.db.insert("takeover_commands", {
            "command_id": str(uuid4()),
            "vehicle_id": vehicle_id,
            "operator_id": operator_id,
            "command_type": command["type"],
            "command_value": command["value"],
            "sent_at": now()
        })

        return {"status": "sent", "sequence": command["sequence"]}

    def hand_back_control(self, vehicle_id, operator_id):
        """交还控制权给自动驾驶"""
        takeover_data = self.redis.get(f"takeover:{vehicle_id}")
        if not takeover_data:
            return {"status": "not_taken_over"}

        takeover = json.loads(takeover_data)

        # 1. 检查当前环境是否安全交还
        vehicle_state = self._get_vehicle_state(vehicle_id)
        safety_check = self._check_handback_safety(vehicle_state)

        if not safety_check["safe"]:
            return {"status": "unsafe_to_hand_back",
                    "reasons": safety_check["reasons"]}

        # 2. 交还控制
        self.redis.delete(f"takeover:{vehicle_id}")

        # 3. 恢复自动驾驶
        self.autonomous_service.resume_autonomous(vehicle_id)

        # 4. 关闭视频流和控制通道
        self.video_service.stop_stream(vehicle_id, f"operator_{operator_id}")
        self.control_service.close_channel(vehicle_id, operator_id)

        # 5. 更新接管记录
        self.db.update("remote_takeovers",
            {"status": "completed", "handed_back_at": now()},
            {"takeover_id": takeover["takeover_id"]})

        return {"status": "handed_back"}

    def _check_handback_safety(self, vehicle_state):
        """检查交还安全性"""
        reasons = []

        if vehicle_state["speed_kmh"] > 30:
            reasons.append("车速过高（>30km/h）")

        if vehicle_state.get("obstacle_distance_m", 100) < 20:
            reasons.append("前方障碍物过近")

        if vehicle_state.get("lane_deviation_cm", 0) > 30:
            reasons.append("车道偏离过大")

        return {"safe": len(reasons) == 0, "reasons": reasons}
```

## 异常场景补充

### 场景：远程接管网络延迟

```
触发：操作员在 2000km 外 → 网络延迟 150ms → 刹车指令延迟 → 紧急情况响应慢
检测：
  1. 控制指令延迟 > 200ms → 延迟过高
  2. 指令发送到执行的时间差 > 300ms → 危险
处理：
  1. 延迟 > 200ms → 降速（限速 30km/h）
  2. 延迟 > 500ms → 自动交还自动驾驶
  3. 紧急情况由车载系统优先处理
预防：延迟限速 + 超时交还 + 车载优先
```

### 场景：操作员误操作

```
触发：操作员误点"急刹车" → 车辆在高速上急刹 → 后车追尾风险
检测：
  1. 急刹指令（brake > 0.5 且速度 > 60km/h）→ 危险操作
  2. 操作模式与当前路况不匹配 → 误操作
处理：
  1. 危险操作需二次确认
  2. 接管模式限速 60km/h
  3. 车载安全系统可覆盖远程指令
预防：二次确认 + 限速 + 车载安全覆盖
```

## 自动驾驶车队调度完整实现

```python
class FleetDispatchService:
    """车队调度：车辆分配 → 路线规划 → 车辆调度 → 实时重调度"""

    def dispatch_vehicle(self, request_id, pickup, dropoff, priority="normal"):
        """调度车辆"""
        # 1. 查找附近可用车辆
        nearby = self.redis.georadius(
            "fleet_locations", pickup["lng"], pickup["lat"],
            5, unit="km", withcoord=True, withdist=True, count=20)

        if not nearby:
            return {"status": "no_vehicle_available"}

        # 2. 按距离和车辆状态评分
        candidates = []
        for vehicle_data in nearby:
            vehicle_id = vehicle_data[0].decode() if isinstance(vehicle_data[0], bytes) else vehicle_data[0]
            distance = vehicle_data[1]
            coords = vehicle_data[2]

            vehicle = self.db.get_vehicle(vehicle_id)
            if not vehicle or vehicle["status"] != "idle":
                continue

            # 评分：距离越近越好
            score = 100 - distance * 10

            # 电量/油量
            energy_pct = vehicle.get("energy_pct", 100)
            if energy_pct < 20:
                continue  # 电量不足 → 排除
            score += energy_pct * 0.2

            # 车型匹配
            if vehicle.get("type") == "premium" and priority == "premium":
                score += 20

            # 车辆清洁度
            score += vehicle.get("cleanliness_score", 80) * 0.1

            candidates.append({
                "vehicle_id": vehicle_id,
                "distance_km": round(distance, 2),
                "score": round(score, 1),
                "eta_minutes": round(distance / 30 * 60, 0)  # 假设 30km/h
            })

        if not candidates:
            return {"status": "no_suitable_vehicle"}

        # 3. 选择最佳车辆
        candidates.sort(key=lambda c: c["score"], reverse=True)
        selected = candidates[0]

        # 4. 分配车辆
        self.db.update("vehicles",
            {"status": "dispatched", "current_request_id": request_id},
            {"id": selected["vehicle_id"]})

        self.db.insert("dispatch_records", {
            "dispatch_id": str(uuid4()),
            "request_id": request_id,
            "vehicle_id": selected["vehicle_id"],
            "pickup_lat": pickup["lat"], "pickup_lng": pickup["lng"],
            "dropoff_lat": dropoff["lat"], "dropoff_lng": dropoff["lng"],
            "distance_km": selected["distance_km"],
            "score": selected["score"],
            "status": "dispatched",
            "created_at": now()
        })

        # 5. 规划路线
        route = self._plan_route(selected["vehicle_id"], pickup, dropoff)

        # 6. 下发调度指令
        self.vehicle_control.send_dispatch(selected["vehicle_id"], {
            "request_id": request_id,
            "pickup": pickup,
            "dropoff": dropoff,
            "route": route,
            "eta_minutes": selected["eta_minutes"]
        })

        return {
            "status": "dispatched",
            "vehicle_id": selected["vehicle_id"],
            "distance_km": selected["distance_km"],
            "eta_minutes": selected["eta_minutes"],
            "route": route
        }

    def redispatch_on_failure(self, request_id, failed_vehicle_id, reason):
        """车辆故障时重新调度"""
        # 1. 标记原车辆故障
        self.db.update("vehicles",
            {"status": "maintenance", "failure_reason": reason},
            {"id": failed_vehicle_id})

        # 2. 获取原始请求信息
        dispatch = self.db.query_one(
            "SELECT * FROM dispatch_records "
            "WHERE request_id = %s AND vehicle_id = %s "
            "ORDER BY created_at DESC LIMIT 1",
            request_id, failed_vehicle_id)

        if not dispatch:
            return {"status": "dispatch_not_found"}

        pickup = {"lat": dispatch["pickup_lat"], "lng": dispatch["pickup_lng"]}
        dropoff = {"lat": dispatch["dropoff_lat"], "lng": dispatch["dropoff_lng"]}

        # 3. 紧急重新调度
        result = self.dispatch_vehicle(request_id, pickup, dropoff, priority="urgent")

        # 4. 通知乘客
        self.notification.send(dispatch.get("passenger_id"),
            "车辆调度变更，新车辆正在赶来")

        # 5. 记录重调度
        self.db.insert("redispatch_log", {
            "log_id": str(uuid4()),
            "request_id": request_id,
            "original_vehicle_id": failed_vehicle_id,
            "reason": reason,
            "new_vehicle_id": result.get("vehicle_id"),
            "created_at": now()
        })

        return result

    def _plan_route(self, vehicle_id, pickup, dropoff):
        """规划路线"""
        # 获取车辆当前位置
        vehicle_pos = self.redis.geopos("fleet_locations", vehicle_id)
        if not vehicle_pos:
            return None

        # 简化路线规划（实际使用地图 API）
        route = {
            "waypoints": [
                {"lat": vehicle_pos[0][1], "lng": vehicle_pos[0][0], "type": "current"},
                {"lat": pickup["lat"], "lng": pickup["lng"], "type": "pickup"},
                {"lat": dropoff["lat"], "lng": dropoff["lng"], "type": "dropoff"}
            ],
            "total_distance_km": self._estimate_distance(vehicle_pos[0], pickup, dropoff),
            "estimated_duration_minutes": 0
        }

        route["estimated_duration_minutes"] = round(
            route["total_distance_km"] / 30 * 60, 0)

        return route

    def _estimate_distance(self, vehicle_pos, pickup, dropoff):
        """估算路线距离"""
        d1 = self._haversine(vehicle_pos[1], vehicle_pos[0],
                            pickup["lat"], pickup["lng"])
        d2 = self._haversine(pickup["lat"], pickup["lng"],
                            dropoff["lat"], dropoff["lng"])
        # 实际道路距离约为直线距离的 1.3 倍
        return round((d1 + d2) * 1.3, 1)
```

## 异常场景补充

### 场景：全部附近车辆电量不足

```
触发：晚高峰 → 所有附近车辆电量 < 20% → 无法调度 → 用户等待超时
检测：
  1. 调度请求返回 no_suitable_vehicle → 无可用车
  2. 附近空闲车辆数 = 0 → 供给不足
处理：
  1. 扩大搜索半径（5km → 10km → 20km）
  2. 允许低电量车辆（限短途）
  3. 引导用户等待充电中的车辆
预防：扩大搜索 + 低电量短途 + 等待引导
```

### 场景：调度算法偏好导致车辆扎堆

```
触发：算法偏好距离最近的车辆 → 高峰区车辆都被调度 → 低峰区无车 → 供需失衡
检测：
  1. 某区域空闲车辆数持续为 0 → 扎堆调度
  2. 部分区域等待时间 > 30 分钟 → 供需失衡
处理：
  1. 调度算法加入区域均衡因子
  2. 预测需求提前调配车辆
  3. 空车巡游引导到高需求区域
预防：均衡因子 + 需求预测 + 巡游引导
```

## 自动驾驶路径规划与避障完整实现

```python
class PathPlanningService:
    """路径规划：全局路线 → 局部避障 → 动态重规划 → 安全约束"""

    def plan_route(self, start, destination, constraints=None):
        """全局路线规划"""
        constraints = constraints or {}

        # 1. 获取路网数据
        road_network = self._load_road_network(start, destination)

        # 2. 构建搜索图（考虑道路限速、车道数、收费等）
        graph = self._build_search_graph(road_network, constraints)

        # 3. A* 算法搜索最优路径
        path = self._a_star_search(graph, start, destination)

        if not path:
            return {"status": "no_path_found"}

        # 4. 计算路径属性
        total_distance = self._calculate_path_distance(path)
        estimated_time = self._estimate_travel_time(path)
        fuel_estimate = self._estimate_fuel_consumption(total_distance)

        # 5. 安全约束验证
        safety_result = self._validate_safety_constraints(path, constraints)
        if not safety_result["safe"]:
            # 重新规划避开危险区域
            constraints["avoid_areas"] = safety_result["dangerous_areas"]
            path = self.plan_route(start, destination, constraints)

        # 6. 生成路径指令序列
        instructions = self._generate_path_instructions(path)

        return {
            "status": "success",
            "path": path,
            "waypoints": [{"lat": p["lat"], "lng": p["lng"],
                          "road_name": p.get("road_name")} for p in path],
            "total_distance_km": round(total_distance, 1),
            "estimated_time_minutes": round(estimated_time, 0),
            "fuel_estimate_liters": round(fuel_estimate, 1),
            "instructions": instructions
        }

    def plan_local_trajectory(self, current_pos, current_speed, obstacles,
                              global_path, lookahead_distance=50):
        """局部轨迹规划（避障）"""
        # 1. 确定局部目标点（全局路径前方 lookahead_distance 处）
        local_goal = self._find_local_goal(current_pos, global_path,
                                           lookahead_distance)

        # 2. 感知障碍物
        dynamic_obstacles = [o for o in obstacles if o.get("moving")]
        static_obstacles = [o for o in obstacles if not o.get("moving")]

        # 3. 构建局部代价图
        cost_map = self._build_local_cost_map(current_pos, local_goal,
                                               static_obstacles, dynamic_obstacles)

        # 4. 生成候选轨迹
        candidate_trajectories = self._generate_candidate_trajectories(
            current_pos, current_speed, local_goal, num_candidates=20)

        # 5. 评估每条轨迹的代价
        best_trajectory = None
        best_cost = float('inf')

        for trajectory in candidate_trajectories:
            cost = self._evaluate_trajectory_cost(trajectory, cost_map, dynamic_obstacles)

            # 安全约束：轨迹不能与任何障碍物碰撞
            if not self._is_trajectory_safe(trajectory, obstacles):
                cost += 10000  # 碰撞代价极大

            # 平滑性约束：加速度和转向角变化率在安全范围内
            if not self._is_trajectory_smooth(trajectory):
                cost += 1000

            if cost < best_cost:
                best_cost = cost
                best_trajectory = trajectory

        # 6. 无安全轨迹 → 紧急停车
        if best_trajectory is None or best_cost >= 10000:
            return {"status": "emergency_stop",
                    "reason": "no_safe_trajectory"}

        return {
            "status": "success",
            "trajectory": best_trajectory,
            "waypoints": [{"x": p[0], "y": p[1], "speed": p[2],
                          "heading": p[3]} for p in best_trajectory],
            "cost": round(best_cost, 2)
        }

    def _a_star_search(self, graph, start, destination):
        """A* 搜索"""
        open_set = {start}
        came_from = {}
        g_score = {start: 0}
        f_score = {start: self._heuristic(start, destination)}

        while open_set:
            current = min(open_set, key=lambda n: f_score.get(n, float('inf')))

            if current == destination:
                # 重构路径
                path = [current]
                while current in came_from:
                    current = came_from[current]
                    path.append(current)
                path.reverse()
                return path

            open_set.remove(current)

            for neighbor in graph.get(current, {}):
                tentative_g = g_score[current] + graph[current][neighbor]

                if tentative_g < g_score.get(neighbor, float('inf')):
                    came_from[neighbor] = current
                    g_score[neighbor] = tentative_g
                    f_score[neighbor] = tentative_g + self._heuristic(neighbor, destination)
                    open_set.add(neighbor)

        return None  # 无路径

    def _heuristic(self, node, goal):
        """启发函数（欧几里得距离）"""
        return self._haversine(node["lat"], node["lng"], goal["lat"], goal["lng"])

    def _generate_candidate_trajectories(self, current_pos, speed, goal, num_candidates):
        """生成候选轨迹"""
        trajectories = []

        # 基于当前速度和方向生成多条候选
        for i in range(num_candidates):
            # 在不同转向角和加速度组合下生成轨迹
            steering_angle = -0.5 + (1.0 * i / num_candidates)
            acceleration = -2.0 + (4.0 * i / num_candidates)

            trajectory = self._simulate_trajectory(
                current_pos, speed, steering_angle, acceleration, time_steps=20)
            trajectories.append(trajectory)

        return trajectories

    def _simulate_trajectory(self, pos, speed, steering, accel, time_steps):
        """模拟轨迹"""
        trajectory = []
        x, y = pos.get("x", 0), pos.get("y", 0)
        heading = pos.get("heading", 0)
        dt = 0.5  # 0.5 秒时间步

        for _ in range(time_steps):
            new_speed = max(0, min(speed + accel * dt, 120))  # 限速 120km/h
            heading += steering * dt
            x += new_speed * dt * math.cos(heading) / 3600  # km/h → km/s
            y += new_speed * dt * math.sin(heading) / 3600

            trajectory.append((round(x, 4), round(y, 4),
                              round(new_speed, 1), round(heading, 4)))
            speed = new_speed

        return trajectory

    def _is_trajectory_safe(self, trajectory, obstacles):
        """检查轨迹安全性"""
        safety_margin = 2.0  # 2 米安全距离

        for point in trajectory:
            px, py = point[0], point[1]

            for obstacle in obstacles:
                ox, oy = obstacle.get("x", 0), obstacle.get("y", 0)
                distance = math.sqrt((px - ox)**2 + (py - oy)**2)
                obs_radius = obstacle.get("radius", 1.5)

                if distance < obs_radius + safety_margin:
                    return False

        return True

    def _is_trajectory_smooth(self, trajectory):
        """检查轨迹平滑性"""
        max_accel_change = 5.0  # m/s²
        max_steering_change = 0.5  # rad/s

        for i in range(1, len(trajectory)):
            speed_diff = abs(trajectory[i][2] - trajectory[i-1][2])
            heading_diff = abs(trajectory[i][3] - trajectory[i-1][3])

            if speed_diff > max_accel_change * 0.5:
                return False
            if heading_diff > max_steering_change * 0.5:
                return False

        return True

    def _evaluate_trajectory_cost(self, trajectory, cost_map, dynamic_obstacles):
        """评估轨迹代价"""
        cost = 0

        # 1. 偏离全局路径的代价
        # 2. 与障碍物距离的代价（越近代价越高）
        # 3. 路径长度的代价
        # 4. 速度变化的代价（加减速消耗能量）

        for point in trajectory:
            px, py = point[0], point[1]

            # 代价图中的代价
            map_cost = cost_map.get((round(px, 2), round(py, 2)), 0)
            cost += map_cost

            # 动态障碍物代价
            for obs in dynamic_obstacles:
                ox, oy = obs.get("x", 0), obs.get("y", 0)
                dist = math.sqrt((px - ox)**2 + (py - oy)**2)
                if dist < 10:
                    cost += 100 / max(dist, 0.1)

        return cost
```

## 异常场景补充

### 场景：全局路径与局部避障冲突

```
触发：全局路径沿主干道 → 局部避障发现前方事故 → 需偏离主干道 → 全局路径失效
检测：
  1. 局部轨迹与全局路径偏差 > 100 米 → 偏离过大
  2. 多次局部避障导致偏离累积 → 需重规划
处理：
  1. 偏离超过阈值 → 触发全局重规划
  2. 重规划从当前位置开始（而非原始起点）
  3. 保留原全局路径作为参考
预防：偏离阈值 + 动态重规划 + 路径参考
```

### 场景：传感器数据异常导致误判障碍物

```
触发：激光雷达受强光干扰 → 误报前方有大障碍物 → 紧急刹车 → 实际无障碍 → 危险
检测：
  1. 多传感器交叉验证不一致 → 可疑障碍物
  2. 障碍物突然出现又消失 → 误报
处理：
  1. 多传感器融合确认（激光雷达 + 摄像头 + 雷达至少 2 个确认）
  2. 低置信度障碍物 → 降速但不急刹
  3. 传感器异常 → 进入安全模式
预防：多传感器融合 + 低置信度降速 + 安全模式
```

## 自动驾驶 V2X 通信与协同完整实现

```python
class V2XCommunicationService:
    """V2X 通信：车辆广播 → 路侧单元 → 协同决策 → 消息验证"""

    MESSAGE_TYPES = {
        "BSM": "基本安全消息（车辆状态广播）",
        "RSI": "路侧信息（交通事件/障碍物）",
        "RSM": "路侧单元消息（信号灯状态）",
        "MAP": "地图消息（道路拓扑）",
        "SPAT": "信号相位与配时",
    }

    def broadcast_vehicle_status(self, vehicle_id, status):
        """广播车辆状态（BSM）"""
        bsm_message = {
            "message_type": "BSM",
            "vehicle_id": vehicle_id,
            "timestamp": now().isoformat(),
            "position": {
                "lat": status["lat"],
                "lng": status["lng"],
                "elevation": status.get("elevation", 0),
                "heading": status["heading"],
                "speed": status["speed"]
            },
            "motion": {
                "acceleration": status.get("acceleration", 0),
                "yaw_rate": status.get("yaw_rate", 0),
                "brake_status": status.get("braking", False),
                "turn_signal": status.get("turn_signal", "none")
            },
            "vehicle": {
                "length": status.get("length", 4.5),
                "width": status.get("width", 1.8),
                "type": status.get("vehicle_type", "passenger")
            }
        }

        # 1. 签名消息（防篡改）
        signed_message = self._sign_message(bsm_message)

        # 2. 广播到 V2X 网络
        self.v2x_network.broadcast(signed_message, range_meters=300)

        # 3. 上报到云端（低频）
        if self._should_report_to_cloud(vehicle_id):
            self.cloud_reporter.report(signed_message)

        return {"status": "broadcast", "vehicle_id": vehicle_id}

    def process_roadside_info(self, rsi_message):
        """处理路侧信息"""
        # 1. 验证消息签名
        if not self._verify_message_signature(rsi_message):
            return {"status": "signature_invalid"}

        # 2. 解析消息内容
        event_type = rsi_message["event_type"]
        event_location = rsi_message["location"]
        event_severity = rsi_message.get("severity", "info")

        # 3. 判断是否影响本车
        affected_vehicles = self._find_affected_vehicles(event_location, radius_m=500)

        # 4. 向受影响车辆推送
        for vehicle_id in affected_vehicles:
            vehicle_pos = self._get_vehicle_position(vehicle_id)

            # 计算距事件的距离
            distance = self._haversine(
                vehicle_pos["lat"], vehicle_pos["lng"],
                event_location["lat"], event_location["lng"])

            if distance < 200:
                urgency = "critical"
            elif distance < 500:
                urgency = "warning"
            else:
                urgency = "info"

            self.v2x_network.send_to_vehicle(vehicle_id, {
                "type": "roadside_alert",
                "event_type": event_type,
                "urgency": urgency,
                "distance_m": round(distance, 0),
                "location": event_location,
                "recommended_action": self._get_recommended_action(
                    event_type, distance, urgency)
            })

        return {"event_type": event_type, "vehicles_notified": len(affected_vehicles)}

    def cooperative_lane_change(self, vehicle_id, target_lane, current_speed):
        """协同变道"""
        # 1. 获取周围车辆状态
        nearby_vehicles = self._get_nearby_vehicles(vehicle_id, radius_m=100)

        # 2. 检查目标车道是否安全
        target_lane_vehicles = [v for v in nearby_vehicles
                               if v["lane"] == target_lane]

        gap_safe = True
        min_gap = float('inf')

        for v in target_lane_vehicles:
            # 计算纵向距离
            longitudinal_gap = self._calculate_longitudinal_gap(
                vehicle_id, v, target_lane)

            # 安全间距 = 速度 * 2 秒
            safe_gap = max(current_speed * 2, 10)

            if longitudinal_gap < safe_gap:
                gap_safe = False
                min_gap = min(min_gap, longitudinal_gap)

        if not gap_safe:
            # 3. 请求协同让行
            for v in target_lane_vehicles:
                longitudinal_gap = self._calculate_longitudinal_gap(
                    vehicle_id, v, target_lane)

                if longitudinal_gap < current_speed * 3:
                    # 请求该车辆减速让行
                    self.v2x_network.send_to_vehicle(v["vehicle_id"], {
                        "type": "cooperative_request",
                        "request": "decelerate",
                        "requesting_vehicle": vehicle_id,
                        "target_gap_m": current_speed * 2.5,
                        "urgency": "normal"
                    })

            return {"status": "waiting_for_cooperation",
                    "min_gap_m": round(min_gap, 1)}

        # 4. 安全 → 执行变道
        return {"status": "safe_to_change", "target_lane": target_lane}

    def _sign_message(self, message):
        """签名消息"""
        message_bytes = json.dumps(message, sort_keys=True).encode()
        signature = self.crypto_engine.sign(message_bytes)
        return {"payload": message, "signature": signature}

    def _verify_message_signature(self, message):
        """验证消息签名"""
        payload = message.get("payload", message)
        signature = message.get("signature")

        if not signature:
            return False

        payload_bytes = json.dumps(payload, sort_keys=True).encode()
        return self.crypto_engine.verify(payload_bytes, signature)

    def _find_affected_vehicles(self, location, radius_m):
        """查找受影响车辆"""
        nearby = self.redis.georadius(
            "vehicle_positions", location["lng"], location["lat"],
            radius_m / 1000, unit="km")

        return [v[0].decode() if isinstance(v[0], bytes) else v[0]
                for v in nearby]

    def _get_vehicle_position(self, vehicle_id):
        """获取车辆位置"""
        pos = self.redis.geopos("vehicle_positions", vehicle_id)
        if pos and pos[0]:
            return {"lat": pos[0][1], "lng": pos[0][0]}
        return {"lat": 0, "lng": 0}

    def _get_nearby_vehicles(self, vehicle_id, radius_m):
        """获取附近车辆"""
        pos = self._get_vehicle_position(vehicle_id)
        nearby = self.redis.georadius(
            "vehicle_positions", pos["lng"], pos["lat"],
            radius_m / 1000, unit="km", withdist=True)

        vehicles = []
        for v in nearby:
            vid = v[0].decode() if isinstance(v[0], bytes) else v[0]
            if vid == vehicle_id:
                continue

            vpos = self._get_vehicle_position(vid)
            vstatus = self.redis.hgetall(f"vehicle_status:{vid}")

            vehicles.append({
                "vehicle_id": vid,
                "distance_m": round(v[1] * 1000, 1),
                "lat": vpos["lat"], "lng": vpos["lng"],
                "speed": float(vstatus.get(b"speed", b"0")),
                "lane": vstatus.get(b"lane", b"unknown").decode()
            })

        return vehicles

    def _calculate_longitudinal_gap(self, vehicle_id, other_vehicle, target_lane):
        """计算纵向间距"""
        # 简化：使用距离估算
        return other_vehicle["distance_m"]

    def _get_recommended_action(self, event_type, distance, urgency):
        """获取建议操作"""
        if urgency == "critical":
            if event_type in ["accident", "obstacle"]:
                return "reduce_speed_and_prepare_to_stop"
            elif event_type == "construction":
                return "merge_to_adjacent_lane"
            else:
                return "proceed_with_caution"
        elif urgency == "warning":
            return "reduce_speed"
        else:
            return "be_aware"

    def _should_report_to_cloud(self, vehicle_id):
        """是否需要上报云端（降频）"""
        last_report = self.redis.get(f"cloud_report:{vehicle_id}")
        if last_report:
            return False
        self.redis.setex(f"cloud_report:{vehicle_id}", 10, "1")
        return True
```

## 异常场景补充

### 场景：V2X 通信延迟导致协同失败

```
触发：协同变道请求延迟 500ms → 目标车道车辆未及时减速 → 变道时间距不足 → 碰撞风险
检测：
  1. V2X 消息延迟 > 100ms → 通信延迟
  2. 协同请求未在预期时间内收到确认 → 超时
处理：
  1. 协同变道需确认响应（超时则放弃变道）
  2. 通信延迟高时自动降级为自主决策
  3. 关键安全消息使用最高优先级通道
预防：响应确认 + 降级决策 + 优先级通道
```

### 场景：恶意节点发送虚假 V2X 消息

```
触发：攻击者发送虚假事故消息 → 周边车辆急刹 → 追尾事故
检测：
  1. 同一位置多个车辆 BSM 数据与 RSI 消息矛盾 → 可疑
  2. 消息签名无法验证 → 伪造
处理：
  1. 验证所有 V2X 消息签名
  2. 多源交叉验证（RSI + 其他车辆 BSM + 摄像头）
  3. 疑似伪造消息 → 忽略 + 上报
预防：签名验证 + 多源交叉 + 忽略伪造
```

### 场景：大量车辆同时广播导致信道拥塞

```
触发：高速公路 1000 辆车同时广播 BSM → DSRC 信道拥塞 → 消息丢失 → 安全风险
检测：
  1. V2X 消息丢失率 > 5% → 信道拥塞
  2. 消息发送队列积压 → 拥塞
处理：
  1. 根据车速和密度动态调整广播频率
  2. 高密度场景降低广播频率（100ms → 500ms）
  3. 关键安全消息优先发送
预防：动态频率 + 密度感知 + 优先级发送
```
