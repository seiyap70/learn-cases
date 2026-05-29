# C11: 医疗影像的存储与阅片

## 业务场景

某医疗影像云平台（类似 PACS 系统），为医院提供影像存储、阅片和远程诊断服务。医生在 Web 端查看患者的 CT/MRI/X 光影像，进行诊断并出具报告。

**已知数据：**
- 影像格式：DICOM（医疗行业标准格式）
- 单次检查影像数：CT 约 300-500 张切片，MRI 约 200-400 张
- 单张切片大小：CT 约 512KB，MRI 约 256KB
- 单次检查总大小：CT 约 150-250MB，MRI 约 50-100MB
- 日均新增检查：1 万次（约 1TB/天）
- 总存储量：5 年累计约 1.5PB
- 影像保存年限：法定 15-30 年（根据国家法规）
- 阅片延迟要求：打开检查到显示首张影像 < 3 秒

**为什么这是难题？**

医疗影像有几个独特约束：
1. **单个文件巨大**：一次 CT 检查 250MB，不是普通 Web 的 KB 级资源
2. **DICOM 格式复杂**：不是简单的图片，包含患者信息、检查参数、像素数据，需要专门的解析器
3. **法定保存期极长**：15-30 年，数据不能删除，累积量惊人
4. **隐私合规**：医疗数据属于最高级别隐私（HIPAA/中国等保三级），存储和传输必须加密

## 核心挑战

### 挑战 1：1.5PB 的存储成本

5 年 1.5PB 的影像数据，如果全存 SSD：约 ¥150 万/月。如果全存对象存储：约 ¥7.5 万/月。但对象存储的访问延迟不满足 3 秒阅片要求。

### 挑战 2：DICOM 的渐进式加载

CT 检查 300 张切片，医生不需要同时看 300 张。工作流是：
1. 先看缩略图（全部 300 张的小图）→ 快速浏览
2. 选中某张切片 → 加载全分辨率
3. 拖动滑动条切换切片 → 每次只看 1 张

因此不需要一次性加载 250MB，而是先加载约 3MB 缩略图，再按需加载约 0.5MB/张。

### 挑战 3：边缘阅片

三甲医院的阅片室在内网，影像存储在医院本地。但远程诊断需要通过互联网访问。如何同时满足院内低延迟和院外可达？

### 挑战 4：DICOM 元数据的查询

医生查询"患者张三的 2024 年所有 CT 检查"，需要搜索 DICOM 文件的元数据（患者ID、检查日期、检查类型），但元数据嵌在 DICOM 文件内部，不可能每次都解析 1.5PB 的文件。

## 设计约束

- DICOM 文件不可修改（医疗数据的完整性要求）
- 所有传输必须 TLS 加密
- 存储必须支持 WORM（Write Once Read Many），防止篡改
- 合规：日志需记录每一次影像访问

## 请先独立思考（限时 35 分钟）

1. 1.5PB 的冷热分层方案：什么存 SSD、什么存 HDD、什么存对象存储？
2. DICOM 文件的元数据如何提取和索引？
3. 阅片时的渐进式加载方案：缩略图如何生成和缓存？
4. 院内院外双通道如何设计？

---

## 设计解析

### 存储架构：三层分级

```
┌────────────────────────────────────────────┐
│ Hot（最近 90 天）     SSD / NVMe           │
│   原始 DICOM + 缩略图 + 预加载             │
│   约 90TB（90天 × 1TB/天）                 │
│   成本：¥9万/月                            │
│   访问延迟：< 100ms                        │
├────────────────────────────────────────────┤
│ Warm（90天-2年）      HDD                  │
│   原始 DICOM（压缩）                        │
│   约 600TB                                 │
│   成本：¥3万/月                            │
│   访问延迟：< 2 秒                         │
├────────────────────────────────────────────┤
│ Cold（2年以上）        对象存储(S3/OSS)     │
│   原始 DICOM（深度压缩）                    │
│   约 800TB                                 │
│   成本：¥0.8万/月                          │
│   访问延迟：5-10 秒                        │
└────────────────────────────────────────────┘

总成本：约 ¥12.8万/月（vs 全SSD的¥150万/月，节省 91%）
```

**自动分层规则：**

```python
class ImageLifecycleManager:
    def on_study_created(self, study):
        # 新检查写入 Hot 层
        self.hot_storage.write(study.dicom_files)
        self.generate_thumbnails(study)  # 生成缩略图

    def daily_lifecycle(self):
        # 90 天前 → Hot → Warm
        studies = self.get_studies_older_than(days=90)
        for study in studies:
            self.warm_storage.write(study.dicom_files)
            self.hot_storage.delete(study.dicom_files)
            # 缩略图保留在 Hot 层（很小，3MB/检查）
        
        # 2 年前 → Warm → Cold
        studies = self.get_studies_older_than(days=730)
        for study in studies:
            self.cold_storage.write(study.dicom_files)
            self.warm_storage.delete(study.dicom_files)
```

### DICOM 元数据索引

**核心思路：写入时提取元数据到数据库，查询时只查数据库。**

```python
class DICOMMetadataExtractor:
    def extract(self, dicom_file):
        """从 DICOM 文件提取元数据"""
        ds = pydicom.dcmread(dicom_file)
        
        return {
            "patient_id": ds.PatientID,
            "patient_name": ds.PatientName,
            "study_id": ds.StudyInstanceUID,
            "series_id": ds.SeriesInstanceUID,
            "study_date": ds.StudyDate,
            "modality": ds.Modality,               # CT / MR / XR
            "body_part": ds.BodyPartExamined,
            "institution": ds.InstitutionName,
            "rows": ds.Rows,
            "columns": ds.Columns,
            "slice_thickness": getattr(ds, 'SliceThickness', None),
            "accession_number": ds.AccessionNumber,
        }
```

**元数据表设计：**

```sql
CREATE TABLE studies (
    study_id VARCHAR(64) PRIMARY KEY,    -- StudyInstanceUID
    patient_id VARCHAR(64) NOT NULL,
    patient_name VARCHAR(128),
    study_date DATE NOT NULL,
    modality VARCHAR(16) NOT NULL,       -- CT / MR / XR / US
    body_part VARCHAR(64),
    institution VARCHAR(128),
    accession_number VARCHAR(64),
    study_description VARCHAR(256),
    series_count INTEGER,
    instance_count INTEGER,
    storage_tier VARCHAR(8) DEFAULT 'hot',  -- hot / warm / cold
    storage_path VARCHAR(512),            -- 文件存储路径
    thumbnail_path VARCHAR(512),          -- 缩略图路径
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_patient (patient_id, study_date),
    INDEX idx_date (study_date, modality),
    INDEX idx_accession (accession_number)
);

CREATE TABLE series (
    series_id VARCHAR(64) PRIMARY KEY,   -- SeriesInstanceUID
    study_id VARCHAR(64) NOT NULL,
    series_number INTEGER,
    series_description VARCHAR(256),
    instance_count INTEGER,
    FOREIGN KEY (study_id) REFERENCES studies(study_id)
);
```

**查询示例：**

```sql
-- 查询患者张三 2024 年的所有 CT 检查
SELECT study_id, study_date, body_part, storage_tier
FROM studies
WHERE patient_id = 'P-12345'
  AND study_date BETWEEN '2024-01-01' AND '2024-12-31'
  AND modality = 'CT'
ORDER BY study_date DESC;
```

### 渐进式阅片加载

**3 秒内显示首张影像的方案：**

```
1. 打开检查 → 加载全部缩略图（300张 × 10KB = 3MB）→ 200ms
2. 加载第一张全分辨率切片（0.5MB）→ 100ms
3. 预加载相邻 10 张切片（5MB）→ 300ms

总计：约 600ms，远优于 3 秒要求
```

**缩略图生成：**

```python
class ThumbnailGenerator:
    def generate(self, study):
        """为一次检查生成缩略图"""
        thumbnails = []
        
        for dicom_file in study.instances:
            ds = pydicom.dcmread(dicom_file)
            
            # 将 DICOM 像素数据转为 JPEG 缩略图
            pixel_array = ds.pixel_array
            
            # 窗宽窗位调整（医学影像特有的亮度/对比度调整）
            windowed = self.apply_window_level(
                pixel_array,
                center=ds.WindowCenter,
                width=ds.WindowWidth
            )
            
            # 缩放到 128×128
            thumbnail = self.resize(windowed, (128, 128))
            
            # 编码为 JPEG（质量 80%）
            jpeg_data = self.encode_jpeg(thumbnail, quality=80)
            
            thumbnails.append(jpeg_data)  # 约 10KB/张
        
        # 存储缩略图
        self.redis.set(
            f"thumbnails:{study.study_id}",
            json.dumps([base64.b64encode(t).decode() for t in thumbnails]),
            ex=86400  # 1 天过期
        )
```

**阅片端加载策略：**

```javascript
class ImageViewer {
    async openStudy(studyId) {
        // 1. 加载缩略图（全部，3MB）
        const thumbnails = await fetch(`/api/studies/${studyId}/thumbnails`);
        this.renderThumbnails(thumbnails);

        // 2. 加载第一张全分辨率切片
        const firstSlice = await fetch(`/api/studies/${studyId}/instances/0/full`);
        this.renderFullResolution(firstSlice);

        // 3. 后台预加载相邻切片
        this.prefetchAdjacent(studyId, currentIndex=0, count=10);
    }

    onSliceChange(newIndex) {
        // 用户切换切片时：
        // 如果已预加载 → 即时显示
        // 如果未预加载 → 加载中提示（通常 < 500ms）
        if (this.cache.has(newIndex)) {
            this.renderFullResolution(this.cache.get(newIndex));
        } else {
            this.showLoading();
            this.loadFullResolution(newIndex);
        }

        // 预加载新的相邻切片
        this.prefetchAdjacent(this.studyId, newIndex, count=5);
    }

    async prefetchAdjacent(studyId, currentIndex, count) {
        const start = Math.max(0, currentIndex - 2);
        const end = currentIndex + count;

        for (let i = start; i <= end; i++) {
            if (!this.cache.has(i)) {
                const slice = await fetch(`/api/studies/${studyId}/instances/${i}/full`);
                this.cache.set(i, slice);
            }
        }
    }
}
```

### 边缘节点：院内低延迟访问

```
院内医生 → 医院边缘节点（本地缓存）→ 3秒内阅片 ✓
院外医生 → 云端存储 → 5-10秒阅片（可接受）
```

**边缘节点架构：**

```
医院内部：
  影像设备 → DICOM 网关 → 边缘存储节点（本地 SSD）
                          ↓ 异步同步
                      云端存储（对象存储）

院内阅片：直接访问边缘节点 → 延迟 < 100ms
院外阅片：访问云端 → 延迟 5-10 秒
首次远程阅片：云端从边缘节点拉取 → 后续访问走云端缓存
```

```python
class EdgeNode:
    def on_dicom_received(self, dicom_study):
        """影像设备推送到边缘节点"""
        # 1. 存储到本地 SSD
        self.local_storage.write(dicom_study)
        
        # 2. 提取元数据到本地数据库
        self.extract_and_index(dicom_study)
        
        # 3. 生成缩略图
        self.generate_thumbnails(dicom_study)
        
        # 4. 异步同步到云端
        self.mq.produce("cloud-sync", {
            "study_id": dicom_study.study_id,
            "priority": "normal"
        })

    def on_cloud_access(self, study_id):
        """院外医生首次访问时，从边缘节点拉取"""
        if not self.cloud_cache.exists(study_id):
            study_data = self.local_storage.read(study_id)
            self.cloud_cache.write(study_id, study_data)
        
        return self.cloud_cache.get_url(study_id)
```

### 安全与审计

**访问审计日志：**

```sql
CREATE TABLE access_audit (
    id BIGSERIAL PRIMARY KEY,
    study_id VARCHAR(64) NOT NULL,
    user_id VARCHAR(64) NOT NULL,
    user_name VARCHAR(128),
    access_type VARCHAR(20),    -- view_thumbnail / view_full / download / print
    client_ip VARCHAR(45),
    accessed_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_study (study_id),
    INDEX idx_user (user_id, accessed_at)
);
```

**传输加密：** 所有 DICOM 文件传输必须 TLS 1.2+。边缘节点与云端之间的同步也必须加密。

**存储加密：** 对象存储开启服务端加密（SSE-S3）。数据库中患者姓名等 PHI 数据加密存储。

## 常见陷阱（深度分析）

### 陷阱 1：不生成缩略图，直接加载全量

**后果：** 300 张 CT 切片 × 512KB = 150MB 全量加载 → 3G 网络下需 4-5 分钟 → 医生等不了

**解决方案：** 缩略图 3MB + 按需加载全分辨率，总加载量 < 10MB。

### 陷阱 2：DICOM 元数据不提取到数据库

**后果：** 查询"患者张三的检查"需要遍历 1.5PB 的 DICOM 文件 → 耗时数天

**解决方案：** 写入时提取元数据到数据库，查询只走数据库。

### 陷阱 3：所有影像存 SSD

**成本计算：** 1.5PB SSD × ¥0.1/GB/月 = ¥15 万/月
分层后：Hot 90TB SSD + Warm 600TB HDD + Cold 800TB 对象存储 = ¥12.8 万/月（但 Hot 层有 90 天数据的热度覆盖，实际远优于全 SSD 方案的用户体验）

### 陷阱 4：边缘节点不同步到云端

**后果：** 医院本地磁盘故障 → 影像数据丢失 → 医疗事故

**解决方案：** 边缘节点必须异步同步到云端，云端作为备份。

## 延伸思考

- **AI 辅助诊断**：如何在阅片流程中集成 AI 模型（如肺结节检测）？模型需要访问全分辨率影像，推理延迟如何控制？
- **3D 重建**：CT 的 300 张切片可以重建为 3D 模型。Web 端的 3D 渲染（WebGL）性能如何保证？
- **跨院影像共享**：患者转院时如何安全地共享影像？DICOM 的 IHE XDS 标准如何实现？