# C11: 医疗影像的存储与阅片

## 业务场景

某医疗影像云平台（类似 PACS 系统），为医院提供影像存储、阅片和远程诊断服务。医生在 Web 端查看患者的 CT/MRI/X 光影像，进行诊断并出具报告。

**已知数据：**
- 影像格式：DICOM（医疗行业标准格式，包含文件头元数据 + 像素数据）
- 单次检查影像数：CT 约 300-500 张切片，MRI 约 200-400 张
- 单张切片大小：CT 约 512KB，MRI 约 256KB
- 单次检查总大小：CT 约 150-250MB，MRI 约 50-100MB
- 日均新增检查：1 万次（约 1TB/天）
- 总存储量：5 年累计约 1.5PB
- 影像保存年限：法定 15-30 年（根据国家法规，门诊 15 年、住院 30 年）
- 阅片延迟要求：打开检查到显示首张影像 < 3 秒
- 并发阅片：高峰期约 500 医生同时在线阅片

**为什么这是难题？**

医疗影像有几个独特约束：
1. **单个文件巨大**：一次 CT 检查 250MB，不是普通 Web 的 KB 级资源；且 DICOM 文件内部结构为 Tag-Length-Value 格式，像素数据通常在文件末尾，无法简单截取
2. **DICOM 格式复杂**：不是简单的图片，包含患者信息、检查参数、像素数据，需要专门的解析器；DICOM 有数千个标准 Tag，不同厂商的私有 Tag 也不一样
3. **法定保存期极长**：15-30 年，数据不能删除，累积量惊人；30 年累计可达 10PB+
4. **隐私合规**：医疗数据属于最高级别隐私（HIPAA/中国等保三级），存储和传输必须加密，访问必须审计，且患者姓名、身份证号等 PHI（Protected Health Information）需要脱敏处理
5. **高可用要求**：影像数据丢失可能导致误诊，存储可靠性要求 99.999%，RPO 接近 0

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

三甲医院的阅片室在内网，影像存储在医院本地。但远程诊断需要通过互联网访问。如何同时满足院内低延迟和院外可达？此外，偏远地区网络带宽有限（可能只有 10Mbps），下载 250MB 的 CT 检查需要 3-4 分钟，如何优化？

### 挑战 4：DICOM 元数据的查询

医生查询"患者张三的 2024 年所有 CT 检查"，需要搜索 DICOM 文件的元数据（患者ID、检查日期、检查类型），但元数据嵌在 DICOM 文件内部，不可能每次都解析 1.5PB 的文件。DICOM 文件头通常 2-4KB，但要读取它必须先解析 128 字节前导码 + 4 字节前缀 + 逐个 Tag 解析，I/O 开销大。

### 挑战 5：大文件上传与断点续传

影像设备（CT/MRI）通过 DICOM 协议推送到边缘网关后，边缘网关需要将文件同步到云端。单次检查 250MB，在 10Mbps 上行带宽下需 3-4 分钟。网络不稳定时上传失败，需要支持断点续传，避免重新上传整个检查。

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
5. 250MB 的 CT 检查如何高效上传和下载？断点续传如何实现？

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

DICOM 标准定义了 Patient → Study → Series → Instance 四层模型，对应数据库也按此层级设计：

```
Patient（患者）
  └── Study（检查）        ← 一次就诊的一次影像检查
        └── Series（序列）  ← 同一检查下的不同序列（如平扫/增强）
              └── Instance（实例）← 单张切片/影像
```

**完整数据库设计（遵循 DICOM 信息模型）：**

```sql
-- 患者表：一个患者可能在不同医院做多次检查
CREATE TABLE patients (
    patient_id         VARCHAR(64)  PRIMARY KEY,     -- 内部患者ID（脱敏后的唯一标识）
    patient_uid        VARCHAR(64)  NOT NULL UNIQUE, -- DICOM PatientID（原值加密存储）
    patient_name_enc   BYTEA,                        -- 患者姓名（AES-256加密）
    birth_date         DATE,                         -- 出生日期
    sex                CHAR(1),                      -- M/F/O
    id_number_enc      BYTEA,                        -- 身份证号（AES-256加密）
    created_at         TIMESTAMP    DEFAULT NOW(),
    updated_at         TIMESTAMP    DEFAULT NOW(),

    INDEX idx_patient_uid (patient_uid)
);

-- 检查表：一次影像检查（对应一个 StudyInstanceUID）
CREATE TABLE studies (
    study_id           VARCHAR(64)  PRIMARY KEY,     -- StudyInstanceUID
    patient_id         VARCHAR(64)  NOT NULL,
    study_date         DATE         NOT NULL,
    study_time         TIME,                          -- 检查时间
    modality           VARCHAR(16)  NOT NULL,         -- CT / MR / XR / US / PT
    body_part          VARCHAR(64),                   -- BodyPartExamined
    institution        VARCHAR(128),                  -- InstitutionName
    accession_number   VARCHAR(64),                   -- 检查流水号
    study_description  VARCHAR(256),                  -- 检查描述
    referring_doctor   VARCHAR(128),                  -- 送检医生
    performing_physician VARCHAR(128),                -- 检查医生
    series_count       INTEGER      DEFAULT 0,
    instance_count     INTEGER      DEFAULT 0,
    total_size_mb      DECIMAL(10,2),                 -- 总大小（MB）
    storage_tier       VARCHAR(8)   DEFAULT 'hot',    -- hot / warm / cold
    storage_path       VARCHAR(512),                  -- 文件存储路径前缀
    thumbnail_path     VARCHAR(512),                  -- 缩略图路径
    is_archived        BOOLEAN      DEFAULT FALSE,    -- 是否已归档
    archive_date       TIMESTAMP,                     -- 归档时间
    created_at         TIMESTAMP    DEFAULT NOW(),

    INDEX idx_patient_date (patient_id, study_date DESC),
    INDEX idx_date_modality (study_date DESC, modality),
    INDEX idx_accession (accession_number),
    INDEX idx_tier (storage_tier, study_date),
    FOREIGN KEY (patient_id) REFERENCES patients(patient_id)
);

-- 序列表：一次检查下的不同序列（如 CT 平扫序列 + CT 增强序列）
CREATE TABLE series (
    series_id          VARCHAR(64)  PRIMARY KEY,     -- SeriesInstanceUID
    study_id           VARCHAR(64)  NOT NULL,
    series_number      INTEGER,                       -- 序列号
    series_description VARCHAR(256),                  -- 序列描述
    modality           VARCHAR(16),                   -- 序列级别的 Modality（可能与 Study 不同）
    body_part          VARCHAR(64),
    protocol_name      VARCHAR(128),                  -- 扫描协议名称
    instance_count     INTEGER      DEFAULT 0,
    total_size_mb      DECIMAL(10,2),
    created_at         TIMESTAMP    DEFAULT NOW(),

    INDEX idx_study (study_id, series_number),
    FOREIGN KEY (study_id) REFERENCES studies(study_id)
);

-- 实例表：单张切片/影像（对应一个 SOPInstanceUID）
CREATE TABLE instances (
    instance_id        VARCHAR(64)  PRIMARY KEY,     -- SOPInstanceUID
    series_id          VARCHAR(64)  NOT NULL,
    study_id           VARCHAR(64)  NOT NULL,
    instance_number    INTEGER,                       -- 切片序号
    sop_class_uid      VARCHAR(64),                   -- SOP Class UID
    rows               INTEGER,                       -- 图像行数
    columns            INTEGER,                       -- 图像列数
    bits_allocated     INTEGER      DEFAULT 16,       -- 每像素分配位数
    bits_stored        INTEGER      DEFAULT 12,       -- 每像素存储位数
    pixel_spacing      VARCHAR(64),                   -- 像素间距（如 "0.5\0.5"）
    slice_thickness    DECIMAL(8,2),                  -- 层厚（mm）
    slice_location     DECIMAL(10,4),                 -- 切片位置
    image_position     VARCHAR(128),                  -- 图像位置（如 "1.0\2.0\3.0"）
    window_center      VARCHAR(64),                   -- 窗位（可能多个值）
    window_width       VARCHAR(64),                   -- 窗宽（可能多个值）
    file_size_kb       INTEGER,                       -- 文件大小（KB）
    storage_path       VARCHAR(512),                  -- 完整存储路径
    thumbnail_path     VARCHAR(512),                  -- 单张缩略图路径
    transfer_syntax    VARCHAR(64),                   -- 传输语法 UID
    created_at         TIMESTAMP    DEFAULT NOW(),

    INDEX idx_series (series_id, instance_number),
    INDEX idx_study_instance (study_id, instance_number),
    FOREIGN KEY (series_id) REFERENCES series(series_id),
    FOREIGN KEY (study_id) REFERENCES studies(study_id)
);

-- 存储位置追踪表：记录每个文件在不同存储层的分布
CREATE TABLE storage_locations (
    id                 BIGSERIAL    PRIMARY KEY,
    study_id           VARCHAR(64)  NOT NULL,
    storage_tier       VARCHAR(8)   NOT NULL,         -- hot / warm / cold
    storage_backend    VARCHAR(16)  NOT NULL,          -- local_ssd / ceph_hdd / s3 / oss
    bucket_or_path     VARCHAR(256),                   -- 存储桶或路径
    object_key         VARCHAR(512),                   -- 对象存储 Key
    etag               VARCHAR(128),                   -- 文件校验（MD5/ETag）
    file_size_bytes    BIGINT,                         -- 文件字节数
    migrated_at        TIMESTAMP    DEFAULT NOW(),
    is_active          BOOLEAN      DEFAULT TRUE,      -- 当前活跃副本

    INDEX idx_study_tier (study_id, storage_tier),
    INDEX idx_active (is_active, storage_tier)
);
```

```python
class DICOMMetadataExtractor:
    """从 DICOM 文件提取完整的四层元数据"""

    # DICOM Tag 常量
    TAG_PATIENT_ID = (0x0010, 0x0020)
    TAG_PATIENT_NAME = (0x0010, 0x0010)
    TAG_STUDY_UID = (0x0020, 0x000D)
    TAG_SERIES_UID = (0x0020, 0x000E)
    TAG_SOP_UID = (0x0008, 0x0018)
    TAG_STUDY_DATE = (0x0008, 0x0020)
    TAG_MODALITY = (0x0008, 0x0060)

    def extract(self, dicom_file):
        """从 DICOM 文件提取元数据，返回四层结构"""
        ds = pydicom.dcmread(dicom_file, stop_before_pixels=True)

        patient_data = {
            "patient_uid": self._safe_get(ds, 'PatientID'),
            "patient_name": self._safe_get(ds, 'PatientName'),
            "birth_date": self._parse_date(self._safe_get(ds, 'PatientBirthDate')),
            "sex": self._safe_get(ds, 'PatientSex', default='O'),
        }

        study_data = {
            "study_id": self._safe_get(ds, 'StudyInstanceUID'),
            "study_date": self._parse_date(self._safe_get(ds, 'StudyDate')),
            "study_time": self._parse_time(self._safe_get(ds, 'StudyTime')),
            "modality": self._safe_get(ds, 'Modality'),
            "body_part": self._safe_get(ds, 'BodyPartExamined'),
            "institution": self._safe_get(ds, 'InstitutionName'),
            "accession_number": self._safe_get(ds, 'AccessionNumber'),
            "study_description": self._safe_get(ds, 'StudyDescription'),
            "referring_doctor": self._safe_get(ds, 'ReferringPhysicianName'),
        }

        series_data = {
            "series_id": self._safe_get(ds, 'SeriesInstanceUID'),
            "study_id": study_data["study_id"],
            "series_number": self._safe_get_int(ds, 'SeriesNumber'),
            "series_description": self._safe_get(ds, 'SeriesDescription'),
            "protocol_name": self._safe_get(ds, 'ProtocolName'),
        }

        instance_data = {
            "instance_id": self._safe_get(ds, 'SOPInstanceUID'),
            "series_id": series_data["series_id"],
            "study_id": study_data["study_id"],
            "instance_number": self._safe_get_int(ds, 'InstanceNumber'),
            "rows": self._safe_get_int(ds, 'Rows'),
            "columns": self._safe_get_int(ds, 'Columns'),
            "bits_allocated": self._safe_get_int(ds, 'BitsAllocated', default=16),
            "bits_stored": self._safe_get_int(ds, 'BitsStored', default=12),
            "slice_thickness": self._safe_get_float(ds, 'SliceThickness'),
            "window_center": self._safe_get(ds, 'WindowCenter'),
            "window_width": self._safe_get(ds, 'WindowWidth'),
            "transfer_syntax": str(ds.file_meta.TransferSyntaxUID)
                               if hasattr(ds, 'file_meta') else None,
        }

        return {
            "patient": patient_data,
            "study": study_data,
            "series": series_data,
            "instance": instance_data,
        }

    def extract_and_persist(self, dicom_file, file_size_kb, storage_path):
        """提取元数据并持久化到数据库"""
        metadata = self.extract(dicom_file)

        with self.db.transaction():
            # 1. Upsert 患者信息（加密 PHI 字段）
            patient = metadata["patient"]
            self.db.execute("""
                INSERT INTO patients (patient_id, patient_uid, patient_name_enc,
                                      birth_date, sex)
                VALUES (%s, %s, %s, %s, %s)
                ON CONFLICT (patient_uid) DO UPDATE SET updated_at = NOW()
            """, [
                self._hash_id(patient["patient_uid"]),
                patient["patient_uid"],
                self.crypto.encrypt(patient["patient_name"]),  # AES-256 加密
                patient["birth_date"],
                patient["sex"],
            ])

            # 2. Upsert Study
            study = metadata["study"]
            self.db.execute("""
                INSERT INTO studies (study_id, patient_id, study_date, study_time,
                    modality, body_part, institution, accession_number,
                    study_description, referring_doctor, storage_path)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON CONFLICT (study_id) DO NOTHING
            """, [
                study["study_id"], self._hash_id(patient["patient_uid"]),
                study["study_date"], study["study_time"],
                study["modality"], study["body_part"], study["institution"],
                study["accession_number"], study["study_description"],
                study["referring_doctor"], storage_path,
            ])

            # 3. Upsert Series
            series = metadata["series"]
            self.db.execute("""
                INSERT INTO series (series_id, study_id, series_number,
                    series_description, modality, protocol_name)
                VALUES (%s, %s, %s, %s, %s, %s)
                ON CONFLICT (series_id) DO NOTHING
            """, [
                series["series_id"], study["study_id"],
                series["series_number"], series["series_description"],
                study["modality"], series["protocol_name"],
            ])

            # 4. Insert Instance
            instance = metadata["instance"]
            self.db.execute("""
                INSERT INTO instances (instance_id, series_id, study_id,
                    instance_number, rows, columns, bits_allocated, bits_stored,
                    slice_thickness, window_center, window_width,
                    transfer_syntax, file_size_kb, storage_path)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON CONFLICT (instance_id) DO NOTHING
            """, [
                instance["instance_id"], series["series_id"], study["study_id"],
                instance["instance_number"], instance["rows"], instance["columns"],
                instance["bits_allocated"], instance["bits_stored"],
                instance["slice_thickness"], instance["window_center"],
                instance["window_width"], instance["transfer_syntax"],
                file_size_kb, storage_path,
            ])

            # 5. 更新 Study/Series 的统计计数
            self.db.execute("""
                UPDATE series SET instance_count = instance_count + 1,
                                  total_size_mb = total_size_mb + %s / 1024.0
                WHERE series_id = %s
            """, [file_size_kb, series["series_id"]])

            self.db.execute("""
                UPDATE studies SET instance_count = instance_count + 1,
                                   total_size_mb = total_size_mb + %s / 1024.0
                WHERE study_id = %s
            """, [file_size_kb, study["study_id"]])

        return metadata

    @staticmethod
    def _safe_get(ds, attr, default=None):
        """安全获取 DICOM 属性，处理私有 Tag 缺失的情况"""
        value = getattr(ds, attr, default)
        if value is None:
            return default
        return str(value).strip() if value != default else default

    @staticmethod
    def _safe_get_int(ds, attr, default=None):
        val = DICOMMetadataExtractor._safe_get(ds, attr)
        return int(float(val)) if val is not None else default

    @staticmethod
    def _safe_get_float(ds, attr, default=None):
        val = DICOMMetadataExtractor._safe_get(ds, attr)
        return float(val) if val is not None else default

    @staticmethod
    def _parse_date(date_str):
        """DICOM 日期格式：YYYYMMDD → Python date"""
        if not date_str or len(date_str) < 8:
            return None
        from datetime import date
        return date(int(date_str[:4]), int(date_str[4:6]), int(date_str[6:8]))

    @staticmethod
    def _parse_time(time_str):
        """DICOM 时间格式：HHMMSS.FFFFFF → Python time"""
        if not time_str or len(time_str) < 6:
            return None
        from datetime import time
        return time(int(time_str[:2]), int(time_str[2:4]), int(time_str[4:6]))

    @staticmethod
    def _hash_id(uid):
        """对 UID 做 SHA256 哈希作为内部 ID，避免暴露原始 ID"""
        import hashlib
        return hashlib.sha256(uid.encode()).hexdigest()[:32]
```

**查询示例（基于上述完整数据库设计）：**

```sql
-- 查询患者张三 2024 年的所有 CT 检查（Study 层级）
SELECT s.study_id, s.study_date, s.body_part, s.storage_tier,
       s.series_count, s.instance_count, s.total_size_mb
FROM studies s
JOIN patients p ON s.patient_id = p.patient_id
WHERE p.patient_uid = 'P-12345'
  AND s.study_date BETWEEN '2024-01-01' AND '2024-12-31'
  AND s.modality = 'CT'
ORDER BY s.study_date DESC;

-- 查询某次检查的所有序列
SELECT series_id, series_number, series_description,
       instance_count, total_size_mb
FROM series
WHERE study_id = '1.2.840.113619.2.55.3...'
ORDER BY series_number;

-- 查询某序列的所有切片
SELECT instance_id, instance_number, rows, columns,
       slice_thickness, slice_location, file_size_kb
FROM instances
WHERE series_id = '1.2.840.113619.2.55.3...1'
ORDER BY instance_number;

-- 查询需要从 Warm 迁移到 Cold 的检查
SELECT study_id, study_date, total_size_mb
FROM studies
WHERE storage_tier = 'warm'
  AND study_date < NOW() - INTERVAL '2 years'
  AND is_archived = FALSE
ORDER BY study_date
LIMIT 100;
```

### 渐进式阅片加载

**3 秒内显示首张影像的方案：**

```
1. 打开检查 → 加载全部缩略图（300张 × 10KB = 3MB）→ 200ms
2. 加载第一张全分辨率切片（0.5MB）→ 100ms
3. 预加载相邻 10 张切片（5MB）→ 300ms

总计：约 600ms，远优于 3 秒要求
```

**缩略图生成（含窗宽窗位处理）：**

医学影像的像素值范围远超普通图像（CT 的 HU 值范围 -1024 到 3071），需要通过窗宽窗位（Window Width/Level）映射到可视范围。不同部位有不同的推荐窗值：

| 部位 | 窗位 (WL) | 窗宽 (WW) | 用途 |
|------|-----------|-----------|------|
| 肺窗 | -600 | 1500 | 观察肺组织 |
| 纵隔窗 | 40 | 400 | 观察软组织 |
| 骨窗 | 400 | 1800 | 观察骨骼 |
| 脑窗 | 40 | 80 | 观察脑组织 |

```python
class ThumbnailGenerator:
    # 默认窗值映射表
    DEFAULT_WINDOWS = {
        'CT': [
            ('lung', -600, 1500),       # 肺窗
            ('mediastinum', 40, 400),   # 纵隔窗
        ],
        'MR': [
            ('brain', 40, 80),          # 脑窗
        ],
        'XR': [],                       # X光不需要窗值调整
    }

    def generate(self, study):
        """为一次检查生成缩略图（多窗值版本）"""
        thumbnails = {}
        window_presets = self.DEFAULT_WINDOWS.get(study.modality, [])

        for dicom_file in study.instances:
            ds = pydicom.dcmread(dicom_file, stop_before_pixels=False)
            pixel_array = ds.pixel_array.astype(np.float32)

            # 生成默认窗的缩略图（使用 DICOM 文件自带的窗值）
            default_windowed = self.apply_window_level(
                pixel_array,
                center=ds.WindowCenter if hasattr(ds, 'WindowCenter') else None,
                width=ds.WindowWidth if hasattr(ds, 'WindowWidth') else None,
            )
            default_thumb = self.resize(default_windowed, (128, 128))
            thumbnails.setdefault('default', []).append(
                self.encode_jpeg(default_thumb, quality=80)
            )

            # 生成各预设窗的缩略图（仅 CT/MR，X光跳过）
            for preset_name, center, width in window_presets:
                windowed = self.apply_window_level(pixel_array, center, width)
                thumb = self.resize(windowed, (128, 128))
                thumbnails.setdefault(preset_name, []).append(
                    self.encode_jpeg(thumb, quality=75)
                )

        # 存储缩略图到 Redis + 对象存储（双写）
        # Redis 用于热缓存，对象存储用于持久化
        study_thumbs_key = f"thumbnails:{study.study_id}"

        for window_type, thumb_list in thumbnails.items():
            # Redis 缓存（1 天过期，阅片高峰期自动续期）
            self.redis.set(
                f"{study_thumbs_key}:{window_type}",
                json.dumps([base64.b64encode(t).decode() for t in thumb_list]),
                ex=86400,
            )

            # 对象存储持久化（缩略图总体很小，存 Hot 层）
            thumb_path = f"thumbnails/{study.study_id}/{window_type}.json"
            self.hot_storage.write_file(thumb_path, json.dumps(
                [base64.b64encode(t).decode() for t in thumb_list]
            ))

            # 更新数据库中的缩略图路径
            self.db.execute("""
                UPDATE studies SET thumbnail_path = %s WHERE study_id = %s
            """, [thumb_path, study.study_id])

        # 单张缩略图也存到 instances 表（用于快速预览）
        for i, dicom_file in enumerate(study.instances):
            instance_id = study.instances[i].instance_id
            thumb_path = f"thumbnails/{study.study_id}/instance/{instance_id}.jpg"
            self.hot_storage.write_file(
                thumb_path,
                thumbnails['default'][i],  # 二进制 JPEG
            )
            self.db.execute("""
                UPDATE instances SET thumbnail_path = %s WHERE instance_id = %s
            """, [thumb_path, instance_id])

        return thumbnails

    def apply_window_level(self, pixel_array, center=None, width=None):
        """窗宽窗位变换：将 HU 值映射到 0-255 的可视范围"""
        if center is None or width is None:
            # 如果没有指定窗值，自动计算（取像素值范围的中间值）
            min_val = pixel_array.min()
            max_val = pixel_array.max()
            center = (min_val + max_val) / 2
            width = max_val - min_val

        # 窗宽窗位公式：
        # 若 pixel ≤ (center - width/2)，输出 0
        # 若 pixel ≥ (center + width/2)，输出 255
        # 否则输出 = 255 × ((pixel - (center - width/2)) / width)
        lower = center - width / 2
        upper = center + width / 2

        windowed = np.clip(pixel_array, lower, upper)
        windowed = ((windowed - lower) / width) * 255.0
        windowed = windowed.astype(np.uint8)

        return windowed

    def resize(self, image_array, target_size):
        """缩放图像到指定尺寸"""
        from skimage.transform import resize as sk_resize
        resized = sk_resize(image_array, target_size, preserve_range=True)
        return resized.astype(np.uint8)

    def encode_jpeg(self, image_array, quality=80):
        """编码为 JPEG"""
        from PIL import Image
        import io
        img = Image.fromarray(image_array)
        buf = io.BytesIO()
        img.save(buf, format='JPEG', quality=quality)
        return buf.getvalue()
```

**阅片端加载策略（含完整预加载和缓存逻辑）：**

```javascript
class ImageViewer {
    constructor() {
        this.cache = new Map();          // instance_index → ImageData
        this.prefetchQueue = [];         // 预加载队列
        this.prefetching = false;        // 预加载锁
        this.currentStudy = null;
        this.currentIndex = 0;
        this.totalInstances = 0;
        this.prefetchWindowSize = 10;    // 预加载窗口大小
    }

    async openStudy(studyId) {
        this.currentStudy = studyId;
        this.cache.clear();

        // 1. 获取 Study 元数据（系列数、实例数、存储层级）
        const meta = await fetch(`/api/studies/${studyId}`).then(r => r.json());
        this.totalInstances = meta.instance_count;

        // 2. 并行加载：缩略图 + 第一张全分辨率切片
        const [thumbnails, firstSlice] = await Promise.all([
            // 加载缩略图（全部，约 3MB）
            fetch(`/api/studies/${studyId}/thumbnails`).then(r => r.json()),
            // 加载第一张全分辨率切片（约 0.5MB）
            fetch(`/api/studies/${studyId}/instances/0/full`).then(r => r.arrayBuffer()),
        ]);

        this.renderThumbnails(thumbnails);
        this.cache.set(0, firstSlice);
        this.renderFullResolution(0, firstSlice);
        this.currentIndex = 0;

        // 3. 后台预加载相邻切片
        this.prefetchAdjacent(studyId, 0, this.prefetchWindowSize);

        // 4. 如果存储在 Warm/Cold 层，触发异步预热
        if (meta.storage_tier !== 'hot') {
            fetch(`/api/studies/${studyId}/warmup`, { method: 'POST' });
        }
    }

    onSliceChange(newIndex) {
        if (newIndex < 0 || newIndex >= this.totalInstances) return;

        this.currentIndex = newIndex;

        if (this.cache.has(newIndex)) {
            // 已预加载 → 即时显示（< 10ms）
            this.renderFullResolution(newIndex, this.cache.get(newIndex));
        } else {
            // 未预加载 → 加载中提示（通常 < 500ms）
            this.showLoading();
            this.loadFullResolution(newIndex);
        }

        // 清理远离当前索引的缓存（限制缓存大小，避免内存溢出）
        this.evictDistantCache(newIndex);

        // 预加载新的相邻切片
        this.prefetchAdjacent(this.currentStudy, newIndex, this.prefetchWindowSize);
    }

    async loadFullResolution(index) {
        try {
            const data = await fetch(
                `/api/studies/${this.currentStudy}/instances/${index}/full`
            ).then(r => r.arrayBuffer());
            this.cache.set(index, data);
            if (this.currentIndex === index) {
                this.renderFullResolution(index, data);
            }
        } catch (err) {
            this.showError(`加载切片 ${index + 1} 失败: ${err.message}`);
        }
    }

    async prefetchAdjacent(studyId, currentIndex, count) {
        // 预加载范围：当前索引前 2 张 + 后 count 张
        const start = Math.max(0, currentIndex - 2);
        const end = Math.min(this.totalInstances - 1, currentIndex + count);
        const prefetchIndices = [];

        for (let i = start; i <= end; i++) {
            if (!this.cache.has(i)) {
                prefetchIndices.push(i);
            }
        }

        if (prefetchIndices.length === 0) return;

        // 并发预加载，限制并发数为 3（避免占用太多带宽）
        const batchSize = 3;
        for (let i = 0; i < prefetchIndices.length; i += batchSize) {
            const batch = prefetchIndices.slice(i, i + batchSize);
            await Promise.all(batch.map(async (idx) => {
                if (this.cache.has(idx)) return; // 可能已被其他预加载请求加载
                try {
                    const data = await fetch(
                        `/api/studies/${studyId}/instances/${idx}/full`
                    ).then(r => r.arrayBuffer());
                    this.cache.set(idx, data);
                } catch (err) {
                    console.warn(`预加载切片 ${idx} 失败:`, err);
                }
            }));
        }
    }

    evictDistantCache(currentIndex) {
        // 保留当前索引 ± 20 范围内的缓存，超出范围则释放
        const keepRange = 20;
        for (const [idx, _] of this.cache) {
            if (Math.abs(idx - currentIndex) > keepRange) {
                this.cache.delete(idx);
            }
        }
    }

    // 键盘导航支持（方向键切换切片）
    onKeyDown(e) {
        if (e.key === 'ArrowUp' || e.key === 'ArrowLeft') {
            this.onSliceChange(this.currentIndex - 1);
        } else if (e.key === 'ArrowDown' || e.key === 'ArrowRight') {
            this.onSliceChange(this.currentIndex + 1);
        }
    }

    // 滚动条快速跳转
    onSliderChange(newIndex) {
        this.onSliceChange(newIndex);
    }

    // 窗宽窗位调整（鼠标拖拽）
    onWindowLevelChange(windowCenter, windowWidth) {
        // 向服务端请求重新窗值处理后的图像
        // 或在前端 WebGL 中实时计算（推荐，零延迟）
        this.wlRenderer.updateWindowLevel(windowCenter, windowWidth);
    }
}
```

### 边缘节点：院内低延迟访问

```
院内医生 → 医院边缘节点（本地缓存）→ 3秒内阅片 ✓
院外医生 → 云端存储 → 5-10秒阅片（可接受）
```

**边缘节点架构（含边缘预处理）：**

```
医院内部：
  影像设备 → DICOM 网关 → 边缘存储节点（本地 SSD）
                          ├─→ 边缘预处理模块（缩略图/去标识化/压缩）
                          ├─→ 本地阅片服务（院内 < 100ms）
                          └─→ 异步同步
                              → 云端存储（对象存储）

院内阅片：直接访问边缘节点 → 延迟 < 100ms
院外阅片：访问云端 → 延迟 5-10 秒
首次远程阅片：云端从边缘节点拉取 → 后续访问走云端缓存
```

**边缘计算架构——影像预处理在边缘完成：**

边缘节点不仅做存储转发，还承担预处理工作，减少云端计算压力和传输带宽：

```
┌──────────────────────────────────────────────────────┐
│                  边缘节点预处理流水线                    │
│                                                      │
│  DICOM 输入                                          │
│     │                                                │
│     ▼                                                │
│  ① 元数据提取 → 写入本地 DB + 同步到云端 DB            │
│     │                                                │
│     ▼                                                │
│  ② 缩略图生成 → 多窗值缩略图（肺窗/纵隔窗/骨窗）        │
│     │                                                │
│     ▼                                                │
│  ③ DICOM 压缩 → 无损 JPEG2000（减少传输体积 50-70%）   │
│     │                                                │
│     ▼                                                │
│  ④ 去标识化 → 移除/替换患者 PHI（用于科研/教学）        │
│     │                                                │
│     ▼                                                │
│  ⑤ 质量检查 → 图像完整性校验、伪影检测                  │
│     │                                                │
│     ▼                                                │
│  输出：原始 DICOM（归档）+ 预处理结果（阅片）            │
└──────────────────────────────────────────────────────┘
```

```python
class EdgeNode:
    def __init__(self, config):
        self.local_storage = LocalSSDStorage(config['local_path'])
        self.cloud_storage = CloudObjectStorage(config['cloud'])
        self.mq = MessageQueue(config['mq'])
        self.extractor = DICOMMetadataExtractor(config['db'])
        self.thumbnail_gen = ThumbnailGenerator(config)
        self.preprocessor = EdgePreprocessor(config)
        self.audit_logger = AuditLogger(config['audit'])
        self.sync_tracker = SyncTracker(config['db'])

    def on_dicom_received(self, dicom_study):
        """影像设备推送到边缘节点（DICOM C-STORE 接收）"""
        # 1. 存储到本地 SSD（原始文件，WORM 模式）
        storage_path = self.local_storage.write(
            dicom_study,
            path_pattern="{date}/{modality}/{study_id}/{instance_id}.dcm"
        )

        # 2. 提取元数据到本地数据库（同步到云端 DB）
        metadata = self.extractor.extract_and_persist(
            dicom_study, file_size_kb=dicom_study.size_kb,
            storage_path=storage_path
        )

        # 3. 边缘预处理流水线
        study_obj = self.build_study_obj(metadata, dicom_study)
        self.preprocessor.process(study_obj)

        # 4. 记录审计日志
        self.audit_logger.log_study_created(
            study_id=metadata['study']['study_id'],
            source=dicom_study.source_ae_title,
            operator=dicom_study.operator_name,
        )

        # 5. 异步同步到云端（带优先级）
        priority = self._calc_sync_priority(metadata)
        self.mq.produce("cloud-sync", {
            "study_id": metadata['study']['study_id'],
            "priority": priority,  # emergency / normal / batch
        })

        # 6. 记录同步状态
        self.sync_tracker.mark_pending(
            study_id=metadata['study']['study_id'],
            local_path=storage_path,
        )

    def _calc_sync_priority(self, metadata):
        """计算同步优先级：急诊检查优先同步"""
        study = metadata['study']
        # 急诊检查（accession_number 以 E- 开头）高优先级
        if study.get('accession_number', '').startswith('E-'):
            return 'emergency'
        # 当天的检查普通优先级
        if study['study_date'] == date.today():
            return 'normal'
        return 'batch'

    def on_cloud_access(self, study_id, user_id):
        """院外医生访问时，从边缘节点拉取到云端缓存"""
        self.audit_logger.log_access(study_id, user_id, 'cloud_access')

        if not self.cloud_storage.exists(study_id):
            # 首次远程访问，从边缘拉取到云端
            study_data = self.local_storage.read(study_id)
            self.cloud_storage.write(study_id, study_data)
            self.audit_logger.log_migration(study_id, 'edge_to_cloud')

        return self.cloud_storage.get_presigned_url(
            study_id, expires_in=3600
        )

    def on_local_access(self, study_id, user_id):
        """院内医生访问"""
        self.audit_logger.log_access(study_id, user_id, 'local_access')
        return self.local_storage.get_path(study_id)


class EdgePreprocessor:
    """边缘预处理模块——在边缘节点完成计算密集型预处理"""

    def process(self, study):
        """执行完整的边缘预处理流水线"""
        # 1. 生成缩略图
        self.thumbnail_gen.generate(study)

        # 2. DICOM 无损压缩（JPEG2000）
        self.compress_study(study)

        # 3. 去标识化（用于科研数据集）
        self.deidentify_study(study)

        # 4. 图像质量检查
        quality_report = self.quality_check(study)
        if not quality_report.passed:
            self.alert_quality_issue(study, quality_report)

    def compress_study(self, study):
        """将 DICOM 像素数据无损压缩为 JPEG2000"""
        import gdcm  # Grassroots DICOM 压缩库

        for instance in study.instances:
            ds = pydicom.dcmread(instance.path)
            # 原始传输语法 → JPEG2000 无损
            ds.compress(gdcm.TransferSyntax.JPEG2000Lossless)
            compressed_path = instance.path.replace('.dcm', '_j2k.dcm')
            ds.save_as(compressed_path)
            # 通常可减少 50-70% 的存储和传输体积

    def deidentify_study(self, study):
        """去标识化：移除/替换患者 PHI 信息（用于科研和教学）"""
        # 遵循 DICOM De-identification 标准（PS3.15 Annex E）
        PHI_TAGS = [
            (0x0010, 0x0010),  # PatientName
            (0x0010, 0x0020),  # PatientID
            (0x0010, 0x0030),  # PatientBirthDate
            (0x0010, 0x1000),  # OtherPatientIDs
            (0x0008, 0x0080),  # InstitutionName
            (0x0008, 0x0081),  # InstitutionAddress
            (0x0008, 0x1070),  # OperatorsName
        ]

        for instance in study.instances:
            ds = pydicom.dcmread(instance.path)
            for group, elem in PHI_TAGS:
                if (group, elem) in ds:
                    # 替换为匿名化的占位值
                    ds[group, elem].value = f"ANON_{hashlib.sha256(
                        str(ds[group, elem].value).encode()
                    ).hexdigest()[:8]}"
            deid_path = instance.path.replace('.dcm', '_deid.dcm')
            ds.save_as(deid_path)

    def quality_check(self, study):
        """图像质量自动检查"""
        issues = []

        for instance in study.instances:
            ds = pydicom.dcmread(instance.path, stop_before_pixels=True)

            # 检查 1：切片完整性（InstanceNumber 是否连续）
            # （在 Study 级别批量检查）

            # 检查 2：像素数据是否存在
            if not hasattr(ds, 'PixelData') or ds.PixelData is None:
                issues.append(f"Instance {instance.instance_id}: 缺少像素数据")

            # 检查 3：图像尺寸是否合理
            if hasattr(ds, 'Rows') and (ds.Rows <= 0 or ds.Rows > 4096):
                issues.append(f"Instance {instance.instance_id}: 异常行数 {ds.Rows}")

            # 检查 4：传输语法是否支持
            if hasattr(ds, 'file_meta'):
                ts = str(ds.file_meta.TransferSyntaxUID)
                if ts not in self.SUPPORTED_TRANSFER_SYNTAXES:
                    issues.append(f"Instance {instance.instance_id}: 不支持的传输语法 {ts}")

        return QualityReport(
            passed=len(issues) == 0,
            issues=issues,
            study_id=study.study_id,
        )


class SyncTracker:
    """跟踪边缘→云端同步状态，确保数据不丢失"""

    def mark_pending(self, study_id, local_path):
        self.db.execute("""
            INSERT INTO sync_status (study_id, local_path, status, created_at)
            VALUES (%s, %s, 'pending', NOW())
        """, [study_id, local_path])

    def mark_synced(self, study_id, cloud_path, etag):
        self.db.execute("""
            UPDATE sync_status
            SET status = 'synced', cloud_path = %s, etag = %s, synced_at = NOW()
            WHERE study_id = %s AND status = 'pending'
        """, [cloud_path, etag, study_id])

    def mark_failed(self, study_id, error):
        self.db.execute("""
            UPDATE sync_status
            SET status = 'failed', error = %s, retry_count = retry_count + 1
            WHERE study_id = %s AND status = 'pending'
        """, [str(error), study_id])

    def get_pending_retries(self):
        """获取需要重试的同步任务"""
        return self.db.query("""
            SELECT study_id, local_path, retry_count
            FROM sync_status
            WHERE status = 'failed' AND retry_count < 5
            ORDER BY created_at ASC
            LIMIT 100
        """)

    def verify_sync_integrity(self):
        """定期校验边缘和云端数据一致性"""
        synced = self.db.query("""
            SELECT study_id, local_path, cloud_path, etag
            FROM sync_status WHERE status = 'synced'
        """)
        for record in synced:
            local_md5 = self._calc_file_md5(record['local_path'])
            if local_md5 != record['etag']:
                self.alert_mismatch(record['study_id'])
```

### 安全与审计

**访问审计日志（完整设计）：**

```sql
CREATE TABLE access_audit (
    id              BIGSERIAL    PRIMARY KEY,
    study_id        VARCHAR(64)  NOT NULL,
    user_id         VARCHAR(64)  NOT NULL,
    user_name       VARCHAR(128),
    user_role       VARCHAR(32),               -- doctor / radiologist / admin / researcher
    access_type     VARCHAR(20)  NOT NULL,      -- view_thumbnail / view_full / download / print / export
    client_ip       VARCHAR(45),
    client_type     VARCHAR(16),               -- web / pacs_client / api
    access_source   VARCHAR(16),               -- edge / cloud
    instance_index  INTEGER,                    -- 访问的具体切片序号
    response_time_ms INTEGER,                   -- 响应时间
    accessed_at     TIMESTAMP    DEFAULT NOW(),

    INDEX idx_study (study_id, accessed_at),
    INDEX idx_user (user_id, accessed_at),
    INDEX idx_access_type (access_type, accessed_at),
    INDEX idx_accessed_at (accessed_at)
);

-- 按月分区（审计数据量大，需按时间分区）
-- ALTER TABLE access_audit PARTITION BY RANGE (accessed_at);
```

**PHI 数据加密存储实现：**

```python
class PHIEncryptor:
    """患者隐私数据加密/解密（AES-256-GCM）"""

    def __init__(self, key_vault_url):
        # 从密钥管理服务获取加密密钥（支持密钥轮换）
        self.kms = KeyManagementService(key_vault_url)

    def encrypt(self, plaintext):
        """加密 PHI 数据"""
        key = self.kms.get_current_key()
        iv = os.urandom(12)  # GCM 推荐 12 字节 IV
        cipher = AES.new(key, AES.MODE_GCM, nonce=iv)
        ciphertext, tag = cipher.encrypt_and_digest(plaintext.encode('utf-8'))
        # 返回格式：key_version + iv + tag + ciphertext
        return (
            struct.pack('>I', self.kms.current_key_version) + iv + tag + ciphertext
        )

    def decrypt(self, encrypted_data):
        """解密 PHI 数据"""
        key_version = struct.unpack('>I', encrypted_data[:4])[0]
        iv = encrypted_data[4:16]
        tag = encrypted_data[16:32]
        ciphertext = encrypted_data[32:]
        key = self.kms.get_key(key_version)  # 支持旧密钥解密
        cipher = AES.new(key, AES.MODE_GCM, nonce=iv)
        return cipher.decrypt_and_verify(ciphertext, tag).decode('utf-8')
```

**传输加密：** 所有 DICOM 文件传输必须 TLS 1.2+。边缘节点与云端之间的同步也必须加密。DICOM 协议层面推荐使用 DICOM TLS（标准 PS3.15）。

**存储加密：** 对象存储开启服务端加密（SSE-S3 或 SSE-KMS）。数据库中患者姓名、身份证号等 PHI 数据使用 AES-256-GCM 加密存储（如上方实现）。

**WORM 存储：** 对象存储开启 Object Lock（合规模式），确保写入后不可修改、不可删除，满足医疗数据完整性要求。

## 常见陷阱（深度分析）

### 陷阱 1：不生成缩略图，直接加载全量

**后果：** 300 张 CT 切片 × 512KB = 150MB 全量加载 → 3G 网络下需 4-5 分钟 → 医生等不了

**解决方案：** 缩略图 3MB + 按需加载全分辨率，总加载量 < 10MB。

**真实案例：** 某医院 PACS 系统升级后，未配置缩略图缓存，导致门诊医生抱怨"打开一个检查要等几分钟"。问题根源是前端直接请求原始 DICOM 文件并用 JS 解析像素，而不是使用服务端预生成的缩略图。

### 陷阱 2：DICOM 元数据不提取到数据库

**后果：** 查询"患者张三的检查"需要遍历 1.5PB 的 DICOM 文件 → 耗时数天

**解决方案：** 写入时提取元数据到数据库，查询只走数据库。

**进阶注意：** 元数据提取需要处理不同厂商的私有 Tag。例如 GE 的 (0009,1010)、Siemens 的 (0019,100A) 等私有 Tag 可能包含重要的临床信息（如造影剂用量），需要专门适配。

### 陷阱 3：所有影像存 SSD

**成本计算：** 1.5PB SSD × ¥0.1/GB/月 = ¥15 万/月
分层后：Hot 90TB SSD + Warm 600TB HDD + Cold 800TB 对象存储 = ¥12.8 万/月（但 Hot 层有 90 天数据的热度覆盖，实际远优于全 SSD 方案的用户体验）

**30 年全量存储的恐怖计算：** 30 年 × 365 天 × 1TB/天 = 10.95PB，全 SSD 成本约 ¥1,095 万/月。分层后约 ¥80-100 万/月（其中 Cold 层占大头但单价极低）。这还不算数据增长（年均 20-30% 增长），实际可能达到 30-50PB。

### 陷阱 4：边缘节点不同步到云端

**后果：** 医院本地磁盘故障 → 影像数据丢失 → 医疗事故

**解决方案：** 边缘节点必须异步同步到云端，云端作为备份。

**进阶设计：** 同步需要保证至少一次语义（at-least-once），使用消息队列 + 同步状态跟踪表（SyncTracker）确保不丢数据。边缘节点故障恢复后，应自动对比本地与云端数据，补传缺失的检查。

### 陷阱 5：DICOM 文件解析不考虑传输语法多样性

**后果：** 只支持 Implicit VR Little Endian，遇到显式 VR 或 JPEG 压缩的 DICOM 文件就解析失败。

**常见传输语法：**

| 传输语法 UID | 名称 | 说明 |
|--------------|------|------|
| 1.2.840.10008.1.2 | Implicit VR Little Endian | 最通用，必须支持 |
| 1.2.840.10008.1.2.1 | Explicit VR Little Endian | 现代 PACS 常用 |
| 1.2.840.10008.1.2.2 | Explicit VR Big Endian | 较少见 |
| 1.2.840.10008.1.2.4.50 | JPEG Baseline | 有损压缩 |
| 1.2.840.10008.1.2.4.70 | JPEG Lossless | 无损压缩 |
| 1.2.840.10008.1.2.4.90 | JPEG 2000 | 无损压缩，推荐 |

**解决方案：** 使用 pydicom + GDCM 库处理所有传输语法，在元数据提取时记录 Transfer Syntax UID 到 instances 表。

### 陷阱 6：缩略图不处理窗宽窗位

**后果：** CT 影像的缩略图要么全黑、要么全白，医生无法在缩略图中辨别组织结构。

**解决方案：** 缩略图必须应用适当的窗宽窗位（见上方 ThumbnailGenerator 的 apply_window_level 实现），且 CT 应生成多个窗值的缩略图（肺窗 + 纵隔窗）。

### 陷阱 7：大文件上传不做分片和断点续传

**后果：** 250MB 的 CT 检查在上传到 80% 时网络中断 → 重新上传 → 又中断 → 医生无法完成上传

**解决方案：** 使用分片上传 + 断点续传机制（见下方大文件上传优化方案）。

## 大文件上传/下载优化

### 分片上传与断点续传

CT 检查 250MB，MRI 检查 50-100MB，在弱网环境下上传容易中断。设计分片上传方案：

```
上传流程：
1. 客户端请求上传 → 服务端返回 upload_id
2. 客户端将文件按 5MB 分片，并行上传各分片
3. 任一分片失败 → 仅重传该分片（断点续传）
4. 所有分片上传完成 → 服务端合并分片 → 校验完整性

下载流程：
1. 服务端返回文件元信息（总大小、分片列表）
2. 客户端并行下载分片（Range 请求）
3. 支持 HTTP Range 请求实现断点续传
```

```python
class ChunkedUploadService:
    """DICOM 检查的分片上传服务"""

    CHUNK_SIZE = 5 * 1024 * 1024   # 5MB 每分片
    MAX_CONCURRENT_UPLOADS = 4      # 最大并发上传数
    MAX_RETRIES = 3                 # 单分片最大重试次数

    def init_upload(self, study_id, total_size, instance_count):
        """初始化上传会话"""
        upload_id = str(uuid.uuid4())

        # 计算分片数
        chunk_count = math.ceil(total_size / self.CHUNK_SIZE)

        self.db.execute("""
            INSERT INTO upload_sessions
                (upload_id, study_id, total_size, chunk_count,
                 uploaded_chunks, status, created_at)
            VALUES (%s, %s, %s, %s, 0, 'pending', NOW())
        """, [upload_id, study_id, total_size, chunk_count])

        return {
            "upload_id": upload_id,
            "chunk_size": self.CHUNK_SIZE,
            "chunk_count": chunk_count,
            "endpoints": {
                "upload_chunk": f"/api/upload/{upload_id}/chunk/{{chunk_index}}",
                "complete": f"/api/upload/{upload_id}/complete",
                "status": f"/api/upload/{upload_id}/status",
            }
        }

    def upload_chunk(self, upload_id, chunk_index, chunk_data):
        """上传单个分片"""
        session = self._get_session(upload_id)
        if not session:
            raise UploadError("上传会话不存在")

        if session['status'] == 'expired':
            raise UploadError("上传会话已过期")

        # 计算分片 MD5 校验
        chunk_md5 = hashlib.md5(chunk_data).hexdigest()

        # 存储分片到临时对象存储
        chunk_key = f"uploads/{upload_id}/chunk_{chunk_index}"
        self.temp_storage.write(chunk_key, chunk_data)

        # 记录分片上传状态
        self.db.execute("""
            INSERT INTO upload_chunks
                (upload_id, chunk_index, chunk_size, md5, storage_key,
                 uploaded_at)
            VALUES (%s, %s, %s, %s, %s, NOW())
            ON CONFLICT (upload_id, chunk_index)
            DO UPDATE SET md5 = %s, uploaded_at = NOW()
        """, [upload_id, chunk_index, len(chunk_data), chunk_md5,
              chunk_key, chunk_md5])

        # 更新已上传分片计数
        self.db.execute("""
            UPDATE upload_sessions
            SET uploaded_chunks = (
                SELECT COUNT(*) FROM upload_chunks WHERE upload_id = %s
            )
            WHERE upload_id = %s
        """, [upload_id, upload_id])

        return {
            "chunk_index": chunk_index,
            "md5": chunk_md5,
            "uploaded_chunks": self._count_uploaded_chunks(upload_id),
            "total_chunks": session['chunk_count'],
        }

    def complete_upload(self, upload_id):
        """合并所有分片，完成上传"""
        session = self._get_session(upload_id)
        chunks = self.db.query("""
            SELECT chunk_index, chunk_size, md5, storage_key
            FROM upload_chunks
            WHERE upload_id = %s
            ORDER BY chunk_index
        """, [upload_id])

        # 校验完整性：分片数和总大小
        if len(chunks) != session['chunk_count']:
            raise UploadError(
                f"分片不完整: 已上传 {len(chunks)}/{session['chunk_count']}"
            )

        total_uploaded = sum(c['chunk_size'] for c in chunks)
        if total_uploaded != session['total_size']:
            raise UploadError(
                f"文件大小不匹配: 上传 {total_uploaded} 预期 {session['total_size']}"
            )

        # 合并分片到最终存储路径
        study_path = f"studies/{session['study_id']}/original.dcm"
        merged_md5 = self._merge_chunks(
            [c['storage_key'] for c in chunks],
            dest_path=study_path,
        )

        # 校验合并后文件的 MD5
        expected_md5 = self._calc_expected_md5(upload_id)
        if expected_md5 and merged_md5 != expected_md5:
            raise UploadError("合并后文件校验失败")

        # 更新会话状态
        self.db.execute("""
            UPDATE upload_sessions
            SET status = 'completed', completed_at = NOW()
            WHERE upload_id = %s
        """, [upload_id])

        # 清理临时分片
        for chunk in chunks:
            self.temp_storage.delete(chunk['storage_key'])

        # 触发后续处理流水线
        self.mq.produce("study-processed", {
            "study_id": session['study_id'],
            "storage_path": study_path,
        })

        return {"status": "completed", "study_id": session['study_id']}

    def get_upload_status(self, upload_id):
        """查询上传状态（用于断点续传）"""
        session = self._get_session(upload_id)
        uploaded = self.db.query("""
            SELECT chunk_index FROM upload_chunks WHERE upload_id = %s
        """, [upload_id])
        uploaded_indices = {r['chunk_index'] for r in uploaded}

        return {
            "upload_id": upload_id,
            "status": session['status'],
            "uploaded_chunks": list(uploaded_indices),
            "total_chunks": session['chunk_count'],
            "missing_chunks": [
                i for i in range(session['chunk_count'])
                if i not in uploaded_indices
            ],
        }

    def _merge_chunks(self, chunk_keys, dest_path):
        """合并分片到目标路径，返回合并后文件的 MD5"""
        md5 = hashlib.md5()
        with self.hot_storage.open_write(dest_path) as dest:
            for key in chunk_keys:
                chunk_data = self.temp_storage.read(key)
                dest.write(chunk_data)
                md5.update(chunk_data)
        return md5.hexdigest()


class ChunkedDownloadService:
    """DICOM 检查的分片下载服务（支持 HTTP Range 请求）"""

    def get_instance(self, study_id, instance_index, range_header=None):
        """获取单张切片，支持 Range 请求"""
        instance = self.db.query_one("""
            SELECT storage_path, file_size_kb FROM instances
            WHERE study_id = %s AND instance_number = %s
        """, [study_id, instance_index])

        if not instance:
            raise NotFoundError(f"切片不存在: {study_id}/{instance_index}")

        file_size = instance['file_size_kb'] * 1024

        if range_header:
            # 解析 Range: bytes=start-end
            start, end = self._parse_range(range_header, file_size)
            data = self.storage.read_range(
                instance['storage_path'], offset=start, length=end - start + 1
            )
            return Response(
                data,
                status=206,
                headers={
                    'Content-Range': f'bytes {start}-{end}/{file_size}',
                    'Content-Length': end - start + 1,
                    'Accept-Ranges': 'bytes',
                }
            )
        else:
            # 完整下载
            data = self.storage.read(instance['storage_path'])
            return Response(
                data,
                headers={
                    'Content-Length': file_size,
                    'Accept-Ranges': 'bytes',
                    'Content-Type': 'application/dicom',
                }
            )
```

### 下载加速：CDN + 智能路由

```python
class DownloadRouter:
    """根据用户位置和存储层级，选择最优下载路径"""

    def get_download_url(self, study_id, user_id):
        study = self.db.query_one(
            "SELECT storage_tier, storage_path FROM studies WHERE study_id = %s",
            [study_id]
        )

        user_region = self.get_user_region(user_id)

        if study['storage_tier'] == 'hot':
            # Hot 层：CDN 加速
            return self.cdn.get_url(study['storage_path'], region=user_region)
        elif study['storage_tier'] == 'warm':
            # Warm 层：直连 HDD 存储
            return self.warm_storage.get_url(study['storage_path'])
        else:
            # Cold 层：先触发解冻，再提供下载
            restore_id = self.cold_storage.restore(study['storage_path'])
            return {"status": "restoring", "restore_id": restore_id,
                    "estimated_time": "3-5 minutes"}
```

## 性能与成本分析

### 存储成本详细测算

| 层级 | 数据量 | 存储介质 | 单价(¥/GB/月) | 月成本 | 年成本 |
|------|--------|----------|---------------|--------|--------|
| Hot | 90TB | SSD/NVMe | 0.10 | ¥9.0万 | ¥108万 |
| Warm | 600TB | HDD(Ceph) | 0.005 | ¥3.0万 | ¥36万 |
| Cold | 800TB | 对象存储(S3/OSS) | 0.001 | ¥0.8万 | ¥9.6万 |
| **合计** | **1.49PB** | - | - | **¥12.8万** | **¥153.6万** |

对比方案：
- 全 SSD：1.5PB × ¥0.1/GB/月 = ¥150 万/月 → 年成本 ¥1,800 万
- 分层存储：¥12.8 万/月 → 年成本 ¥153.6 万 → **节省 91.5%**

### 30 年累计成本推算

假设数据量年均增长 20%：

| 年份 | 年增量 | 累计总量 | 分层年成本 |
|------|--------|---------|-----------|
| 第 1 年 | 365TB | 365TB | ¥153万 |
| 第 5 年 | 906TB | 2.7PB | ¥550万 |
| 第 10 年 | 2,252TB | 8.0PB | ¥1,800万 |
| 第 15 年 | 5,590TB | 20PB | ¥4,500万 |
| 第 30 年 | 87,100TB | 200PB+ | ¥4.5亿+ |

> 30 年数据增长呈指数级，分级存储 + 冷数据深度压缩是唯一可行方案。全 SSD 方案在 10 年后就不可承受。

### 阅片性能指标

| 指标 | 目标 | 实际表现 |
|------|------|---------|
| 首张影像显示 | < 3 秒 | 600ms（Hot 层）/ 1.5s（Warm 层） |
| 切片切换延迟 | < 500ms | 50ms（已预加载）/ 200ms（未预加载） |
| 缩略图加载 | < 1 秒 | 200ms |
| 500 医生并发 | 系统稳定 | P99 < 800ms |
| Cold 层解冻 | < 10 分钟 | 3-5 分钟 |

### 网络带宽需求

| 场景 | 并发数 | 单次流量 | 峰值带宽 |
|------|--------|---------|---------|
| 院内阅片 | 200 | 3MB(缩略图) + 5MB(预加载) | ~130Mbps |
| 院外阅片 | 50 | 3MB + 5MB | ~33Mbps |
| 边缘→云端同步 | - | 1TB/天 | ~100Mbps（持续） |
| 远程下载 | 20 | 250MB(全量) | ~330Mbps |

## 异常场景与容灾设计

### 异常场景与应对

| 异常场景 | 影响 | 应对策略 |
|---------|------|---------|
| 边缘节点宕机 | 院内无法阅片 | 本地 RAID + 双节点 HA，自动切换 |
| 云端对象存储不可用 | 院外无法阅片 | 多区域复制，自动切换到备区域 |
| 边缘→云端同步中断 | 数据仅存本地，丢失风险高 | 消息队列持久化 + 重试 + SyncTracker |
| DICOM 文件损坏 | 无法阅片，可能误诊 | 写入时计算 MD5/ETag，定期校验完整性 |
| 数据库故障 | 无法查询检查列表 | 主从同步 + 读写分离，从库可读 |
| Redis 缓存失效 | 缩略图重新加载，延迟增加 | 从对象存储回填，保证可用性 |
| 上传中途断网 | 检查数据不完整 | 断点续传，客户端查询缺失分片后补传 |
| 磁盘空间不足 | 无法写入新检查 | 自动分层迁移 + 告警 + 扩容 |
| DICOM 关联超时 | 影像设备传输中断，检查数据不完整 | 超时重试 + 部分检查标记为不完整 + 支持续传 |
| 存储层迁移失败 | 数据卡在源层，目标层缺失 | 事务性迁移 + 回滚 + 重试队列 |
| PACS 故障切换中阅片 | 医生阅片中服务切换，会话中断 | 会话保持 + 客户端自动重连 + 未保存操作恢复 |

### 异常场景详细实现

#### 异常场景 A：DICOM 关联超时（设备传输中断）

影像设备（CT/MRI 扫描仪）通过 DICOM 协议的 C-STORE 操作将影像推送到 PACS 网关。一次 CT 检查包含 300-500 张切片，传输过程中可能因网络抖动、设备重启等原因导致 DICOM Association（关联）超时断开，部分切片已传输、部分未传输。

**问题关键：** DICOM 关联是面向连接的 TCP 会话，一旦断开，已传输的切片虽然已存储，但检查处于不完整状态。需要检测不完整检查、支持设备重连后续传，并标记检查为"部分接收"。

```python
class DICOMAssociationManager:
    """DICOM 关联管理器——处理设备传输中断和续传"""

    # DICOM 关联超时配置
    ASSOCIATION_TIMEOUT = 60       # 关联空闲超时（秒）
    STORE_TIMEOUT_PER_INSTANCE = 30  # 单个实例接收超时（秒）
    INCOMPLETE_CHECK_INTERVAL = 300  # 不完整检查扫描间隔（秒）
    MAX_INCOMPLETE_AGE_HOURS = 24    # 超过此时长的不完整检查标记为异常

    def __init__(self, config):
        self.ae = None  # DICOM AE (Application Entity)
        self.db = config['db']
        self.storage = config['storage']
        self.audit_logger = config['audit_logger']
        self.notification = config['notification']

    def on_association_requested(self, assoc):
        """DICOM 关联请求回调——验证请求方身份和权限"""
        # 验证 Calling AE Title（设备标识）
        calling_ae = assoc.requestor.ae_title
        if not self._is_registered_ae(calling_ae):
            assoc.reject(reason="AE Title 未注册")
            self.audit_logger.log_association_rejected(
                calling_ae=calling_ae,
                called_ae=assoc.ae_title,
                reason="unregistered_ae",
            )
            return

        # 验证请求的 SOP Class 是否支持
        for context in assoc.requestor.presentation_context:
            if not self._is_supported_sop_class(context.abstract_syntax):
                assoc.reject(reason=f"不支持的 SOP Class: {context.abstract_syntax}")
                return

        # 接受关联，设置超时
        assoc.set_timeout(self.ASSOCIATION_TIMEOUT)
        self.audit_logger.log_association_established(
            calling_ae=calling_ae,
            called_ae=assoc.ae_title,
        )

    def on_c_store(self, ds, assoc_info):
        """C-STORE 请求回调——接收单个 DICOM 实例"""
        study_uid = str(ds.StudyInstanceUID)
        series_uid = str(ds.SeriesInstanceUID)
        sop_uid = str(ds.SOPInstanceUID)

        # 检查是否为重复实例（设备重传）
        existing = self.db.query_one("""
            SELECT instance_id FROM instances WHERE instance_id = %s
        """, [sop_uid])
        if existing:
            # 重复实例，校验完整性后忽略
            if self._verify_instance_integrity(ds):
                return 0x0000  # Success：告诉设备已收到
            else:
                # 已存在但数据不一致，覆盖
                self._overwrite_instance(ds, sop_uid)

        # 创建或获取检查接收会话
        session = self._get_or_create_receive_session(study_uid, assoc_info)

        # 存储实例到临时接收目录（确认完整后再移到正式目录）
        temp_path = f"receiving/{study_uid}/{series_uid}/{sop_uid}.dcm"
        self.storage.write_temp(temp_path, ds)

        # 更新接收会话状态
        self.db.execute("""
            INSERT INTO receive_sessions_instances
                (session_id, instance_id, series_id, temp_path, received_at)
            VALUES (%s, %s, %s, %s, NOW())
        """, [session['session_id'], sop_uid, series_uid, temp_path])

        self.db.execute("""
            UPDATE receive_sessions
            SET received_count = received_count + 1,
                last_activity = NOW()
            WHERE session_id = %s
        """, [session['session_id']])

        return 0x0000  # Success

    def on_association_released(self, assoc):
        """DICOM 关联正常释放——检查本次接收的完整性"""
        calling_ae = assoc.requestor.ae_title
        self._finalize_receiving_sessions(calling_ae)

    def on_association_aborted(self, assoc, error):
        """DICOM 关联异常中断——设备断开连接"""
        calling_ae = assoc.requestor.ae_title
        self.audit_logger.log_association_aborted(
            calling_ae=calling_ae,
            error=str(error),
        )
        # 标记该设备的接收会话为"关联中断"状态
        self.db.execute("""
            UPDATE receive_sessions
            SET status = 'association_aborted',
                abort_reason = %s,
                aborted_at = NOW()
            WHERE source_ae = %s AND status = 'receiving'
        """, [str(error), calling_ae])

    def _finalize_receiving_sessions(self, source_ae):
        """完成接收会话——检查完整性并提交到正式存储"""
        sessions = self.db.query("""
            SELECT session_id, study_id, expected_count, received_count, status
            FROM receive_sessions
            WHERE source_ae = %s AND status IN ('receiving', 'association_aborted')
        """, [source_ae])

        for session in sessions:
            if session['expected_count'] and session['received_count'] >= session['expected_count']:
                # 完整接收 → 提交到正式存储
                self._commit_session(session)
            else:
                # 不完整接收 → 标记为 partial
                self._mark_session_partial(session)

    def _commit_session(self, session):
        """将接收会话中的实例从临时目录移到正式存储"""
        instances = self.db.query("""
            SELECT instance_id, series_id, temp_path
            FROM receive_sessions_instances
            WHERE session_id = %s
        """, [session['session_id']])

        for inst in instances:
            # 移动到正式存储路径
            formal_path = f"studies/{session['study_id']}/{inst['series_id']}/{inst['instance_id']}.dcm"
            self.storage.move_temp_to_formal(inst['temp_path'], formal_path)

            # 更新 instances 表的存储路径
            self.db.execute("""
                UPDATE instances SET storage_path = %s WHERE instance_id = %s
            """, [formal_path, inst['instance_id']])

        self.db.execute("""
            UPDATE receive_sessions SET status = 'committed', committed_at = NOW()
            WHERE session_id = %s
        """, [session['session_id']])

    def _mark_session_partial(self, session):
        """标记为部分接收，等待设备重连后续传"""
        self.db.execute("""
            UPDATE receive_sessions
            SET status = 'partial',
                partial_at = NOW()
            WHERE session_id = %s
        """, [session['session_id']])

        # 如果设备重新发起关联，检查是否有未完成的接收会话
        # 在 on_association_requested 中触发续传

    def check_incomplete_studies(self):
        """定期扫描不完整检查（定时任务）"""
        incomplete_sessions = self.db.query("""
            SELECT session_id, study_id, source_ae, expected_count,
                   received_count, status, created_at
            FROM receive_sessions
            WHERE status IN ('partial', 'association_aborted')
              AND created_at > NOW() - INTERVAL '%s hours'
        """, [self.MAX_INCOMPLETE_AGE_HOURS])

        for session in incomplete_sessions:
            age_hours = (datetime.now() - session['created_at']).total_seconds() / 3600

            if age_hours > self.MAX_INCOMPLETE_AGE_HOURS:
                # 超时不完整 → 通知技师重传
                self.notification.send(
                    target=session['source_ae'],
                    message=f"检查 {session['study_id']} 接收不完整"
                            f"（{session['received_count']}/{session['expected_count']}），"
                            f"请重新发送",
                    severity='warning',
                )
                # 标记检查为不完整
                self.db.execute("""
                    UPDATE studies SET status = 'incomplete' WHERE study_id = %s
                """, [session['study_id']])
            else:
                # 仍在等待窗口内，尝试通过 C-MOVE 请求设备重传缺失实例
                missing = self._get_missing_instances(session)
                if missing:
                    self._request_retransmit(session['source_ae'], missing)

    def _request_retransmit(self, target_ae, missing_instances):
        """通过 DICOM C-MOVE 请求设备重传缺失实例"""
        for instance_id in missing_instances:
            try:
                # 发起 C-MOVE 请求到源设备
                self.ae.move(
                    move_aet=target_ae,
                    query_identifier={'SOPInstanceUID': instance_id},
                    move_destination=self.ae.ae_title,
                )
            except Exception as e:
                self.audit_logger.log_retransmit_failed(
                    target_ae=target_ae,
                    instance_id=instance_id,
                    error=str(e),
                )

    def _get_or_create_receive_session(self, study_uid, assoc_info):
        """获取或创建接收会话"""
        existing = self.db.query_one("""
            SELECT session_id, study_id, expected_count, received_count, status
            FROM receive_sessions
            WHERE study_id = %s AND source_ae = %s AND status IN ('receiving', 'partial')
        """, [study_uid, assoc_info.requestor.ae_title])

        if existing:
            # 续传场景：设备重连，继续之前的接收会话
            self.db.execute("""
                UPDATE receive_sessions
                SET status = 'receiving', last_activity = NOW()
                WHERE session_id = %s
            """, [existing['session_id']])
            return existing

        # 新会话
        session_id = str(uuid.uuid4())
        self.db.execute("""
            INSERT INTO receive_sessions
                (session_id, study_id, source_ae, expected_count,
                 received_count, status, created_at)
            VALUES (%s, %s, %s, %s, 0, 'receiving', NOW())
        """, [session_id, study_uid, assoc_info.requestor.ae_title, None])

        return {'session_id': session_id, 'study_id': study_uid}
```

**接收会话数据库表：**

```sql
-- DICOM 接收会话（跟踪每次关联的接收状态）
CREATE TABLE receive_sessions (
    session_id       VARCHAR(64)  PRIMARY KEY,
    study_id         VARCHAR(64)  NOT NULL,
    source_ae        VARCHAR(16)  NOT NULL,     -- 设备 AE Title
    expected_count   INTEGER,                    -- 预期实例数（可能未知）
    received_count   INTEGER      DEFAULT 0,     -- 已接收实例数
    status           VARCHAR(20)  DEFAULT 'receiving',  -- receiving/partial/committed/aborted
    abort_reason     VARCHAR(256),
    last_activity    TIMESTAMP    DEFAULT NOW(),
    created_at       TIMESTAMP    DEFAULT NOW(),
    committed_at     TIMESTAMP,
    partial_at       TIMESTAMP,
    aborted_at       TIMESTAMP,

    INDEX idx_source_status (source_ae, status),
    INDEX idx_study (study_id),
    INDEX idx_status_created (status, created_at)
);

-- 接收会话中的实例记录
CREATE TABLE receive_sessions_instances (
    id               BIGSERIAL    PRIMARY KEY,
    session_id       VARCHAR(64)  NOT NULL,
    instance_id      VARCHAR(64)  NOT NULL,
    series_id        VARCHAR(64)  NOT NULL,
    temp_path        VARCHAR(512),
    received_at      TIMESTAMP    DEFAULT NOW(),

    INDEX idx_session (session_id),
    UNIQUE (session_id, instance_id)
);
```

#### 异常场景 B：存储层迁移失败（Hot → Warm 迁移中断）

自动分层迁移是将 90 天前的检查从 Hot（SSD）迁移到 Warm（HDD）。迁移涉及大量文件复制和元数据更新。迁移过程中如果 HDD 集群故障、网络中断或进程崩溃，会导致迁移半完成：Hot 层已删除但 Warm 层未写入，或者两层都有但元数据不一致。

**核心原则：** 迁移必须是事务性的——要么全部成功（源层删除 + 目标层写入 + 元数据更新），要么全部回滚。

```python
class StorageMigrationService:
    """存储层迁移服务——事务性迁移与故障恢复"""

    MIGRATION_BATCH_SIZE = 50       # 每批迁移检查数
    VERIFY_AFTER_MIGRATE = True     # 迁移后校验数据完整性
    MAX_RETRY_COUNT = 3             # 迁移失败最大重试次数
    CLEANUP_DELAY_HOURS = 24        # 源层数据保留时间（安全窗口）

    def __init__(self, config):
        self.hot_storage = config['hot_storage']
        self.warm_storage = config['warm_storage']
        self.cold_storage = config['cold_storage']
        self.db = config['db']
        self.audit_logger = config['audit_logger']
        self.migration_lock = DistributedLock('storage_migration')

    def migrate_hot_to_warm(self):
        """Hot → Warm 分层迁移（每日定时任务）"""
        # 分布式锁，确保同一时间只有一个迁移进程
        with self.migration_lock:
            # 1. 查询需要迁移的检查
            studies = self.db.query("""
                SELECT study_id, storage_path, total_size_mb
                FROM studies
                WHERE storage_tier = 'hot'
                  AND study_date < NOW() - INTERVAL '90 days'
                  AND is_archived = FALSE
                  AND study_id NOT IN (
                      SELECT study_id FROM migration_tasks
                      WHERE status IN ('in_progress', 'verifying')
                  )
                ORDER BY study_date ASC
                LIMIT %s
            """, [self.MIGRATION_BATCH_SIZE])

            for study in studies:
                self._migrate_study_with_transaction(
                    study, from_tier='hot', to_tier='warm'
                )

    def _migrate_study_with_transaction(self, study, from_tier, to_tier):
        """事务性迁移单个检查"""
        task_id = str(uuid.uuid4())

        try:
            # Step 1: 创建迁移任务（记录到数据库）
            self.db.execute("""
                INSERT INTO migration_tasks
                    (task_id, study_id, from_tier, to_tier,
                     status, started_at)
                VALUES (%s, %s, %s, %s, 'in_progress', NOW())
            """, [task_id, study['study_id'], from_tier, to_tier])

            # Step 2: 写入目标层（先写目标，再删源——安全方向）
            source_files = self._list_study_files(study['study_id'], from_tier)
            for file_info in source_files:
                data = self._read_from_tier(file_info['path'], from_tier)

                # 可选：在迁移时压缩（Hot → Warm 使用 JPEG2000 无损压缩）
                if from_tier == 'hot' and to_tier == 'warm':
                    data = self._compress_if_needed(data, file_info)

                write_success = self._write_to_tier(
                    study['study_id'], file_info, data, to_tier
                )
                if not write_success:
                    raise MigrationError(
                        f"写入目标层失败: {file_info['path']}"
                    )

            # Step 3: 验证目标层数据完整性
            self.db.execute("""
                UPDATE migration_tasks SET status = 'verifying' WHERE task_id = %s
            """, [task_id])

            verify_result = self._verify_migration(study['study_id'], from_tier, to_tier)
            if not verify_result.passed:
                raise MigrationError(
                    f"迁移后验证失败: {verify_result.mismatches}"
                )

            # Step 4: 更新元数据（storage_tier 和 storage_locations 表）
            self.db.execute("""
                UPDATE studies SET storage_tier = %s WHERE study_id = %s
            """, [to_tier, study['study_id']])

            self.db.execute("""
                UPDATE storage_locations SET is_active = FALSE
                WHERE study_id = %s AND storage_tier = %s
            """, [study['study_id'], from_tier])

            self.db.execute("""
                INSERT INTO storage_locations
                    (study_id, storage_tier, storage_backend, object_key, etag,
                     file_size_bytes, is_active)
                SELECT study_id, %s, 'ceph_hdd', storage_path,
                       MD5(storage_path), file_size_kb * 1024, TRUE
                FROM instances WHERE study_id = %s
            """, [to_tier, study['study_id']])

            # Step 5: 延迟删除源层（安全窗口后才删）
            # 不立即删除，而是标记为待删除，24 小时后由清理任务删除
            self.db.execute("""
                INSERT INTO pending_deletions
                    (study_id, tier, scheduled_at)
                VALUES (%s, %s, NOW() + INTERVAL '%s hours')
            """, [study['study_id'], from_tier, self.CLEANUP_DELAY_HOURS])

            # Step 6: 标记迁移完成
            self.db.execute("""
                UPDATE migration_tasks
                SET status = 'completed', completed_at = NOW()
                WHERE task_id = %s
            """, [task_id])

            self.audit_logger.log_migration_completed(
                study_id=study['study_id'],
                from_tier=from_tier,
                to_tier=to_tier,
            )

        except MigrationError as e:
            # 迁移失败 → 回滚
            self._rollback_migration(task_id, study['study_id'], from_tier, to_tier, str(e))

        except Exception as e:
            # 未知异常 → 回滚 + 告警
            self._rollback_migration(task_id, study['study_id'], from_tier, to_tier, str(e))
            self._alert_migration_failure(study['study_id'], str(e))

    def _rollback_migration(self, task_id, study_id, from_tier, to_tier, error):
        """迁移失败回滚——确保数据一致性"""
        # 1. 删除目标层已写入的部分数据
        target_files = self._list_study_files(study_id, to_tier)
        for file_info in target_files:
            try:
                self._delete_from_tier(file_info['path'], to_tier)
            except Exception:
                pass  # 删除失败不影响回滚，后续清理任务会处理

        # 2. 恢复元数据（确保 storage_tier 仍指向源层）
        self.db.execute("""
            UPDATE studies SET storage_tier = %s WHERE study_id = %s
        """, [from_tier, study_id])

        # 3. 标记迁移失败
        self.db.execute("""
            UPDATE migration_tasks
            SET status = 'failed',
                error = %s,
                retry_count = retry_count + 1,
                failed_at = NOW()
            WHERE task_id = %s
        """, [error, task_id])

        self.audit_logger.log_migration_rolled_back(
            study_id=study_id,
            from_tier=from_tier,
            to_tier=to_tier,
            reason=error,
        )

    def _verify_migration(self, study_id, from_tier, to_tier):
        """验证迁移后数据完整性——逐文件校验"""
        source_files = self._list_study_files(study_id, from_tier)
        target_files = self._list_study_files(study_id, to_tier)
        mismatches = []

        # 检查文件数量
        if len(source_files) != len(target_files):
            mismatches.append(
                f"文件数量不匹配: 源={len(source_files)}, 目标={len(target_files)}"
            )
            return VerificationResult(passed=False, mismatches=mismatches)

        # 逐文件校验 MD5
        source_etags = {f['path']: f['etag'] for f in source_files}
        target_etags = {f['path']: f['etag'] for f in target_files}

        for path, etag in source_etags.items():
            if path not in target_etags:
                mismatches.append(f"目标层缺失文件: {path}")
            elif target_etags[path] != etag:
                mismatches.append(f"文件校验不一致: {path}")

        return VerificationResult(
            passed=len(mismatches) == 0,
            mismatches=mismatches,
        )

    def retry_failed_migrations(self):
        """重试失败的迁移任务"""
        failed_tasks = self.db.query("""
            SELECT task_id, study_id, from_tier, to_tier, retry_count
            FROM migration_tasks
            WHERE status = 'failed' AND retry_count < %s
            ORDER BY created_at ASC
            LIMIT 20
        """, [self.MAX_RETRY_COUNT])

        for task in failed_tasks:
            study = self.db.query_one(
                "SELECT study_id, storage_path, total_size_mb FROM studies WHERE study_id = %s",
                [task['study_id']]
            )
            self._migrate_study_with_transaction(
                study, from_tier=task['from_tier'], to_tier=task['to_tier']
            )

    def cleanup_pending_deletions(self):
        """清理安全窗口已过的源层数据"""
        deletions = self.db.query("""
            SELECT id, study_id, tier
            FROM pending_deletions
            WHERE scheduled_at <= NOW() AND deleted_at IS NULL
        """)

        for deletion in deletions:
            try:
                self._delete_study_from_tier(deletion['study_id'], deletion['tier'])
                self.db.execute("""
                    UPDATE pending_deletions SET deleted_at = NOW() WHERE id = %s
                """, [deletion['id']])
            except Exception as e:
                self.audit_logger.log_deletion_failed(
                    study_id=deletion['study_id'],
                    tier=deletion['tier'],
                    error=str(e),
                )


class MigrationError(Exception):
    """迁移异常"""
    pass


class VerificationResult:
    """迁移验证结果"""
    def __init__(self, passed, mismatches):
        self.passed = passed
        self.mismatches = mismatches
```

**迁移任务数据库表：**

```sql
-- 迁移任务表
CREATE TABLE migration_tasks (
    task_id          VARCHAR(64)  PRIMARY KEY,
    study_id         VARCHAR(64)  NOT NULL,
    from_tier        VARCHAR(8)   NOT NULL,
    to_tier          VARCHAR(8)   NOT NULL,
    status           VARCHAR(20)  DEFAULT 'in_progress',
    error            TEXT,
    retry_count      INTEGER      DEFAULT 0,
    started_at       TIMESTAMP,
    completed_at     TIMESTAMP,
    failed_at        TIMESTAMP,
    created_at       TIMESTAMP    DEFAULT NOW(),

    INDEX idx_study (study_id),
    INDEX idx_status (status, created_at)
);

-- 待删除记录（安全窗口延迟删除）
CREATE TABLE pending_deletions (
    id               BIGSERIAL    PRIMARY KEY,
    study_id         VARCHAR(64)  NOT NULL,
    tier             VARCHAR(8)   NOT NULL,
    scheduled_at     TIMESTAMP    NOT NULL,
    deleted_at       TIMESTAMP,

    INDEX idx_scheduled (scheduled_at, deleted_at)
);
```

#### 异常场景 C：PACS 服务器故障切换期间阅片中

医生正在 Web 端阅片（已打开一个 CT 检查，正在逐张查看切片），此时 PACS 主服务器突然宕机，系统自动切换到备服务器。如果不做特殊处理，医生的阅片会话中断，窗宽窗位调整、标注等未保存操作丢失，重新打开检查需要从头开始加载。

**关键需求：** 故障切换对医生透明——会话自动恢复，未保存的操作不丢失，当前查看的切片无需重新加载。

```python
class ViewSessionManager:
    """阅片会话管理器——支持故障切换时的会话恢复"""

    SESSION_TTL = 7200          # 会话有效期（秒）
    AUTO_SAVE_INTERVAL = 30     # 自动保存间隔（秒）
    SNAPSHOT_INTERVAL = 60      # 会话快照间隔（秒）

    def __init__(self, config):
        self.redis = config['redis']          # 主 Redis
        self.redis_backup = config['redis_backup']  # 备 Redis
        self.db = config['db']
        self.audit_logger = config['audit_logger']

    def create_session(self, user_id, study_id):
        """创建阅片会话"""
        session_id = str(uuid.uuid4())

        session_data = {
            'session_id': session_id,
            'user_id': user_id,
            'study_id': study_id,
            'current_instance_index': 0,
            'window_center': None,
            'window_width': None,
            'zoom_level': 1.0,
            'pan_offset': [0, 0],
            'annotations': [],        # 标注数据
            'measurements': [],       # 测量数据
            'created_at': datetime.now().isoformat(),
            'last_activity': datetime.now().isoformat(),
        }

        # 双写：主 Redis + 备 Redis（确保切换后会话不丢失）
        self._write_session_both(session_id, session_data)

        # 持久化到数据库（异步，不阻塞阅片）
        self.db.execute_async("""
            INSERT INTO view_sessions
                (session_id, user_id, study_id, session_data, created_at)
            VALUES (%s, %s, %s, %s, NOW())
        """, [session_id, user_id, study_id, json.dumps(session_data)])

        return session_id

    def update_session(self, session_id, updates):
        """更新阅片会话状态（每次切片切换、窗值调整时调用）"""
        session = self._get_session(session_id)
        if not session:
            raise SessionNotFoundError(session_id)

        # 合并更新
        session.update(updates)
        session['last_activity'] = datetime.now().isoformat()

        # 双写 Redis
        self._write_session_both(session_id, session)

        # 定期持久化到数据库（避免每次操作都写 DB）
        # 由 auto_save_snapshot 任务定期执行

    def add_annotation(self, session_id, annotation):
        """添加标注（箭头、文字、ROI 等）"""
        session = self._get_session(session_id)
        if not session:
            raise SessionNotFoundError(session_id)

        annotation['id'] = str(uuid.uuid4())
        annotation['created_at'] = datetime.now().isoformat()
        session['annotations'].append(annotation)
        session['last_activity'] = datetime.now().isoformat()

        # 标注数据必须立即持久化到数据库（不可丢失）
        self._write_session_both(session_id, session)
        self.db.execute("""
            INSERT INTO view_annotations
                (session_id, instance_index, annotation_type,
                 annotation_data, created_at)
            VALUES (%s, %s, %s, %s, NOW())
        """, [session_id, annotation.get('instance_index'),
              annotation.get('type'), json.dumps(annotation)])

    def save_snapshot(self, session_id):
        """保存会话快照到数据库"""
        session = self._get_session(session_id)
        if not session:
            return

        self.db.execute("""
            UPDATE view_sessions
            SET session_data = %s, last_snapshot_at = NOW()
            WHERE session_id = %s
        """, [json.dumps(session), session_id])

    def recover_session(self, user_id, study_id):
        """故障切换后恢复会话——客户端自动调用"""
        # 1. 尝试从主 Redis 恢复
        session = self._get_session_from_redis(user_id, study_id)

        # 2. 主 Redis 不可用则从备 Redis 恢复
        if not session:
            session = self._get_session_from_redis_backup(user_id, study_id)

        # 3. 备 Redis 也没有则从数据库恢复
        if not session:
            session = self._get_session_from_db(user_id, study_id)

        if not session:
            # 完全无法恢复，创建新会话
            return self.create_session(user_id, study_id)

        # 恢复会话：生成新的 session_id（避免 session fixation）
        new_session_id = str(uuid.uuid4())
        session['session_id'] = new_session_id
        session['recovered_at'] = datetime.now().isoformat()
        session['recovery_reason'] = 'failover'

        self._write_session_both(new_session_id, session)

        self.audit_logger.log_session_recovered(
            user_id=user_id,
            study_id=study_id,
            original_session_id=session_id,
            new_session_id=new_session_id,
        )

        return {
            'session_id': new_session_id,
            'study_id': study_id,
            'current_instance_index': session['current_instance_index'],
            'window_center': session['window_center'],
            'window_width': session['window_width'],
            'zoom_level': session['zoom_level'],
            'pan_offset': session['pan_offset'],
            'annotations': session['annotations'],
            'measurements': session['measurements'],
        }

    def _write_session_both(self, session_id, session_data):
        """双写主备 Redis"""
        key = f"view_session:{session_id}"
        value = json.dumps(session_data)
        try:
            self.redis.set(key, value, ex=self.SESSION_TTL)
        except Exception:
            pass  # 主 Redis 不可用不影响写入备 Redis
        try:
            self.redis_backup.set(key, value, ex=self.SESSION_TTL)
        except Exception:
            pass

    def _get_session(self, session_id):
        """获取会话数据（优先主 Redis，回退备 Redis）"""
        key = f"view_session:{session_id}"
        try:
            data = self.redis.get(key)
            if data:
                return json.loads(data)
        except Exception:
            pass
        try:
            data = self.redis_backup.get(key)
            if data:
                return json.loads(data)
        except Exception:
            pass
        return None
```

**客户端自动重连逻辑：**

```javascript
class ResilientViewer extends ImageViewer {
    constructor() {
        super();
        this.sessionId = null;
        this.retryCount = 0;
        this.maxRetries = 5;
        this.retryDelay = 1000;  // 初始重试延迟 1 秒
        this.pendingAnnotations = [];  // 未同步的标注
        this.localBackup = new LocalStorageBackup();
    }

    async openStudy(studyId) {
        try {
            const session = await fetch('/api/view-sessions', {
                method: 'POST',
                body: JSON.stringify({ study_id: studyId }),
            }).then(r => r.json());

            this.sessionId = session.session_id;
            await super.openStudy(studyId);

            // 恢复之前的阅片状态
            if (session.current_instance_index > 0) {
                this.onSliceChange(session.current_instance_index);
            }
            if (session.window_center && session.window_width) {
                this.onWindowLevelChange(session.window_center, session.window_width);
            }
        } catch (err) {
            this.handleConnectionError(err);
        }
    }

    async handleConnectionError(error) {
        console.warn('连接中断，开始自动重连...', error);

        // 立即保存当前状态到本地存储
        this.localBackup.save(this.sessionId, {
            currentInstanceIndex: this.currentIndex,
            windowCenter: this.wlRenderer.windowCenter,
            windowWidth: this.wlRenderer.windowWidth,
            annotations: this.pendingAnnotations,
            timestamp: Date.now(),
        });

        // 指数退避重试
        while (this.retryCount < this.maxRetries) {
            const delay = this.retryDelay * Math.pow(2, this.retryCount);
            await this.sleep(delay);

            try {
                // 尝试恢复会话
                const recovery = await fetch('/api/view-sessions/recover', {
                    method: 'POST',
                    body: JSON.stringify({
                        study_id: this.currentStudy,
                    }),
                }).then(r => r.json());

                if (recovery.session_id) {
                    this.sessionId = recovery.session_id;
                    this.retryCount = 0;

                    // 恢复阅片状态
                    if (recovery.current_instance_index !== undefined) {
                        this.onSliceChange(recovery.current_instance_index);
                    }

                    // 同步本地暂存的标注
                    await this.syncPendingAnnotations();

                    this.showNotification('连接已恢复，阅片继续');
                    return;
                }
            } catch (e) {
                this.retryCount++;
                console.warn(`重连失败 (${this.retryCount}/${this.maxRetries})`);
            }
        }

        this.showError('服务器暂时不可用，请稍后重试。您的标注已保存，重新连接后将自动恢复。');
    }

    async addAnnotation(annotation) {
        // 标注先保存到本地，再异步同步到服务器
        this.pendingAnnotations.push(annotation);
        this.localBackup.save(this.sessionId, {
            annotations: this.pendingAnnotations,
        });

        try {
            await fetch(`/api/view-sessions/${this.sessionId}/annotations`, {
                method: 'POST',
                body: JSON.stringify(annotation),
            });
            // 同步成功，从待同步列表移除
            this.pendingAnnotations = this.pendingAnnotations.filter(a => a !== annotation);
        } catch (err) {
            // 同步失败，保留在待同步列表，重连后自动同步
            console.warn('标注同步失败，已保存到本地', err);
        }
    }

    async syncPendingAnnotations() {
        for (const annotation of this.pendingAnnotations) {
            try {
                await fetch(`/api/view-sessions/${this.sessionId}/annotations`, {
                    method: 'POST',
                    body: JSON.stringify(annotation),
                });
            } catch (err) {
                console.warn('标注同步失败:', annotation.id, err);
                break;  // 再次失败则停止
            }
        }
        this.pendingAnnotations = [];
    }

    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}
```

**阅片会话数据库表：**

```sql
-- 阅片会话表
CREATE TABLE view_sessions (
    session_id       VARCHAR(64)  PRIMARY KEY,
    user_id          VARCHAR(64)  NOT NULL,
    study_id         VARCHAR(64)  NOT NULL,
    session_data     JSONB,                        -- 完整会话状态
    last_snapshot_at TIMESTAMP,
    created_at       TIMESTAMP    DEFAULT NOW(),

    INDEX idx_user_study (user_id, study_id),
    INDEX idx_created (created_at)
);

-- 标注表（独立存储，确保不丢失）
CREATE TABLE view_annotations (
    id               BIGSERIAL    PRIMARY KEY,
    session_id       VARCHAR(64)  NOT NULL,
    instance_index   INTEGER,                     -- 标注所在切片序号
    annotation_type  VARCHAR(32),                 -- arrow/text/roi/measure
    annotation_data  JSONB,                       -- 标注详细数据
    created_at       TIMESTAMP    DEFAULT NOW(),

    INDEX idx_session (session_id, instance_index)
);
```

### 容灾架构

```
                    ┌──── 主区域（北京）────┐
                    │  云端 DB（主）         │
                    │  对象存储（主）         │
                    │  应用服务（主）         │
                    └──────────┬────────────┘
                               │ 异步复制
                    ┌──────────▼────────────┐
                    │  备区域（上海）         │
                    │  云端 DB（从）         │
                    │  对象存储（从）         │
                    │  应用服务（备）         │
                    └───────────────────────┘

RPO: < 1 分钟（数据库异步复制）
RTO: < 15 分钟（自动故障切换）
数据可靠性: 99.9999999%（11 个 9，对象存储多副本）
```

```python
class DisasterRecoveryManager:
    """容灾管理器"""

    def check_health(self):
        """定期健康检查"""
        checks = {
            'primary_db': self._check_db(self.primary_db),
            'primary_storage': self._check_storage(self.primary_storage),
            'replica_db_lag': self._check_replication_lag(),
            'edge_nodes': self._check_edge_nodes(),
            'sync_queue': self._check_sync_queue(),
        }

        for name, result in checks.items():
            if not result['healthy']:
                self.alert(f"健康检查失败: {name} - {result['message']}")

        return checks

    def failover(self):
        """主区域故障切换到备区域"""
        # 1. 验证备区域数据库是否可读
        if not self._verify_replica_consistency():
            raise FailoverError("备区域数据不一致，无法切换")

        # 2. 提升备区域数据库为主
        self.replica_db.promote_to_primary()

        # 3. 切换 DNS 到备区域
        self.dns.switch_to_backup(ttl=30)

        # 4. 通知所有边缘节点同步到新主区域
        self._notify_edge_nodes(new_primary=self.backup_region)

        # 5. 记录容灾切换事件
        self.audit_logger.log_failover(
            from_region=self.primary_region,
            to_region=self.backup_region,
        )

    def verify_data_integrity(self, study_id):
        """校验单个检查的数据完整性"""
        # 1. 获取文件 ETag
        locations = self.db.query("""
            SELECT storage_tier, etag, file_size_bytes
            FROM storage_locations
            WHERE study_id = %s AND is_active = TRUE
        """, [study_id])

        for loc in locations:
            current_etag = self._calc_storage_etag(
                loc['storage_tier'], loc['bucket_or_path'], loc['object_key']
            )
            if current_etag != loc['etag']:
                self.alert_data_corruption(study_id, loc['storage_tier'])
                # 尝试从其他存储层恢复
                self._recover_from_other_tier(study_id, loc['storage_tier'])

    def _recover_from_other_tier(self, study_id, corrupted_tier):
        """从其他存储层恢复损坏的数据"""
        other_tiers = self.db.query("""
            SELECT storage_tier, bucket_or_path, object_key, etag
            FROM storage_locations
            WHERE study_id = %s AND is_active = TRUE AND storage_tier != %s
        """, [study_id, corrupted_tier])

        if not other_tiers:
            self.alert_critical(f"检查 {study_id} 所有副本均损坏！")
            return

        # 从最近的存储层恢复
        source = sorted(other_tiers, key=lambda x: ['hot', 'warm', 'cold'].index(x['storage_tier']))[0]
        self._copy_between_tiers(
            study_id, from_tier=source['storage_tier'], to_tier=corrupted_tier
        )
```

### 边缘节点容灾

```python
class EdgeNodeHA:
    """边缘节点高可用（双节点 Active-Passive）"""

    def __init__(self, config):
        self.active = EdgeNode(config['active'])
        self.standby = EdgeNode(config['standby'])
        self.heartbeat = HeartbeatChecker(
            active_addr=config['active']['addr'],
            standby_addr=config['standby']['addr'],
            interval=5,     # 5 秒心跳
            timeout=15,     # 15 秒超时
        )

    def on_dicom_received(self, dicom_study):
        """Active 节点接收数据后同步到 Standby"""
        # 1. Active 写入
        self.active.on_dicom_received(dicom_study)

        # 2. 同步到 Standby（同步写，确保双写成功）
        self.standby.replicate_study(dicom_study)

    def on_active_failure(self):
        """Active 节点故障时自动切换"""
        # 1. 验证 Standby 数据完整性
        lag = self.standby.get_replication_lag()
        if lag > timedelta(minutes=5):
            self.alert("Standby 数据延迟过大，切换存在数据丢失风险")

        # 2. 提升 Standby 为 Active
        self.standby.promote_to_active()

        # 3. 更新 VIP（虚拟 IP）指向新 Active
        self.vip.switch_to(self.standby.addr)

        # 4. 通知云端新的边缘节点地址
        self.cloud.update_edge_node_address(self.standby.addr)

        # 5. 告警运维团队
        self.alert("边缘节点已切换，请尽快修复原 Active 节点")
```

## 延伸思考

- **AI 辅助诊断**：如何在阅片流程中集成 AI 模型（如肺结节检测）？模型需要访问全分辨率影像，推理延迟如何控制？可能的方案：在边缘节点部署 GPU 推理服务，检查完成后自动触发 AI 推理，结果标注在 DICOM 的 SR（Structured Report）中，阅片时叠加显示。
- **3D 重建**：CT 的 300 张切片可以重建为 3D 模型。Web 端的 3D 渲染（WebGL）性能如何保证？可使用 WebGL2 + marching cubes 算法，在 GPU 端完成体绘制；或使用 WebAssembly + VTK.js 实现高性能渲染；必要时可在服务端预渲染关键视角的 2D 图像。
- **跨院影像共享**：患者转院时如何安全地共享影像？DICOM 的 IHE XDS 栅栏如何实现？跨院共享涉及患者身份映射、数据传输安全、访问权限控制三大问题，IHE XDS.b/XDR 提供了标准化的注册和查询协议。
- **DICOMweb 标准**：新一代 PACS 正在从 DIMSE 协议（C-FIND/C-MOVE/C-STORE）向 DICOMweb（RESTful API）迁移。WADO-RS（Retrieve）、STOW-RS（Store）、QIDO-RS（Query）三个接口如何实现？如何兼容现有 DIMSE 设备？
- **存储成本控制**：随着数据量指数增长，30 年后可能达到 200PB+。是否需要在 Cold 层进一步细分为 Cold（标准访问）和 Archive（归档，恢复需数小时）？Archive 层的存储成本仅为标准 S3 的 1/4，但恢复时间从毫秒级变为小时级。
- **法规演进**：中国《数据安全法》和《个人信息保护法》对医疗数据的跨境传输、数据本地化提出了新要求。多云部署如何满足数据本地化？跨境远程诊断如何合规？
## DICOM 数据管理完整实现

```python
class DicomDataManager:
    """DICOM 医学影像数据管理"""

    def store_dicom(self, dicom_file, patient_id, study_id):
        """存储 DICOM 影像"""
        # 1. 解析 DICOM 元数据
        ds = pydicom.dcmread(dicom_file)
        metadata = {
            "patient_id": patient_id,
            "study_id": study_id,
            "series_id": ds.SeriesInstanceUID,
            "instance_id": ds.SOPInstanceUID,
            "modality": ds.Modality,  # CT, MRI, X-Ray
            "body_part": ds.BodyPartExamined,
            "slice_thickness": float(ds.SliceThickness) if hasattr(ds, 'SliceThickness') else None,
            "pixel_spacing": ds.PixelSpacing if hasattr(ds, 'PixelSpacing') else None,
            "rows": ds.Rows,
            "columns": ds.Columns,
            "bits_allocated": ds.BitsAllocated,
            "study_date": ds.StudyDate,
            "accession_number": ds.AccessionNumber,
        }

        # 2. 脱敏处理：移除患者姓名、出生日期等 PHI
        self._deidentify(ds)

        # 3. 存储到对象存储
        object_key = f"dicom/{patient_id}/{study_id}/{metadata['instance_id']}.dcm"
        self.s3_client.upload(dicom_file, object_key)

        # 4. 存储元数据到数据库
        self.db.insert("dicom_instances", {
            **metadata,
            "object_key": object_key,
            "file_size_bytes": os.path.getsize(dicom_file),
            "checksum_md5": self._compute_md5(dicom_file),
            "stored_at": now()
        })

        return metadata["instance_id"]

    def _deidentify(self, ds):
        """DICOM 脱敏：移除受保护健康信息"""
        phi_tags = [
            (0x0010, 0x0010),  # Patient Name
            (0x0010, 0x0030),  # Patient Birth Date
            (0x0010, 0x1000),  # Other Patient IDs
            (0x0008, 0x0080),  # Institution Name
            (0x0008, 0x0081),  # Institution Address
        ]
        for group, elem in phi_tags:
            if (group, elem) in ds:
                del ds[(group, elem)]
```

## AI 辅助诊断推理流水线

```python
class AIDiagnosisPipeline:
    """AI 辅助诊断推理流水线"""

    def analyze_study(self, study_id):
        """分析一个检查（含多个序列）"""
        study = self.db.get_study(study_id)
        instances = self.db.get_study_instances(study_id)

        # 1. 预处理：标准化、窗宽窗位调整
        preprocessed = []
        for instance in instances:
            image = self.load_dicom_image(instance["object_key"])
            normalized = self.preprocess(image, study["modality"])
            preprocessed.append(normalized)

        # 2. 模型推理
        if study["modality"] == "CT":
            result = self.ct_model.predict(preprocessed)
        elif study["modality"] == "MRI":
            result = self.mri_model.predict(preprocessed)
        elif study["modality"] == "X-Ray":
            result = self.xray_model.predict(preprocessed)

        # 3. 结果后处理
        findings = []
        for detection in result["detections"]:
            findings.append({
                "finding_type": detection["class"],
                "confidence": detection["confidence"],
                "location": detection["bbox"],
                "severity": self._assess_severity(detection),
                "description": self._generate_description(detection)
            })

        # 4. 生成诊断报告草稿
        report = self._generate_report(study, findings)

        return {"study_id": study_id, "findings": findings, "report_draft": report}
```

## 异常场景补充

### 场景：DICOM 文件损坏

```
触发：传输中断导致 DICOM 文件不完整 → pydicom 解析失败
检测：
  1. DICOM 解析异常 → 文件损坏标记
  2. 校验和比对：MD5 不匹配 → 损坏
处理：
  1. 从 PACS 重新获取
  2. 重新获取失败 → 标记为 unavailable
  3. 不完整的检查 → 不参与 AI 分析
预防：传输后校验 + 自动重试
```

### 场景：AI 模型误诊

```
触发：AI 将良性结节判定为恶性 → 假阳性
      → 患者接受不必要的活检
检测：
  1. 医生复核时标记"不同意AI结论"
  2. 假阳性率统计：按月统计各部位
  3. 假阳性率 > 15% → 模型需要重训
处理：
  1. 记录误诊案例 → 加入训练集
  2. 调整置信度阈值（提高阳性判定门槛）
  3. 模型重训 + 验证后重新部署
预防：AI 结果仅作辅助参考 + 医生必须复核
```

### 场景：PACS 系统连接中断

```
触发：PACS 服务器维护 → 无法获取影像
检测：
  1. DICOM 连接超时 → 告警
  2. 持续 > 5 分钟 → 严重告警
处理：
  1. 使用本地缓存（最近 7 天的影像）
  2. 降级：只处理已有缓存的研究
  3. 新上传的影像暂存本地 → PACS 恢复后同步
预防：PACS 高可用 + 本地缓存 + 连接监控
```

## PACS 集成完整实现

```python
class PACSIntegration:
    """PACS 系统集成：DICOM C-FIND/C-MOVE"""

    def query_studies(self, patient_id, date_range=None):
        """查询患者检查列表（C-FIND）"""
        from pynetdicom import AE, QueryRetrieveSOPClass
        ae = AE()
        ae.add_requested_context(QueryRetrieveSOPClass)

        ds = pydicom.Dataset()
        ds.PatientID = patient_id
        ds.QueryRetrieveLevel = "STUDY"
        if date_range:
            ds.StudyDate = f"{date_range[0]}-{date_range[1]}"

        assoc = ae.associate(self.pacs_host, self.pacs_port)
        if not assoc.is_established:
            raise PACSConnectionError("无法连接 PACS")

        studies = []
        for result in assoc.send_c_find(ds, query_model="S"):
            if result.Status == 0xFF00:
                studies.append({
                    "study_id": result.StudyInstanceUID,
                    "study_date": result.StudyDate,
                    "modality": result.Modality,
                    "study_desc": result.StudyDescription,
                })
        assoc.release()
        return studies

    def retrieve_study(self, study_id, destination="/tmp/dicom"):
        """获取检查影像（C-MOVE）"""
        ds = pydicom.Dataset()
        ds.StudyInstanceUID = study_id
        ds.QueryRetrieveLevel = "STUDY"

        assoc = self.ae.associate(self.pacs_host, self.pacs_port)
        result = assoc.send_c_move(ds, destination, query_model="S")
        assoc.release()
        return result
```

## 放射学报告生成

```python
class RadiologyReportGenerator:
    """AI 辅助放射学报告生成"""

    def generate_report(self, study_id, ai_findings):
        """生成结构化报告"""
        report_sections = []

        # 1. 检查信息
        study = self.db.get_study(study_id)
        report_sections.append(f"检查类型: {study['modality']}")
        report_sections.append(f"检查部位: {study['body_part']}")
        report_sections.append(f"检查日期: {study['study_date']}")

        # 2. 影像表现
        report_sections.append("\n【影像表现】")
        for finding in ai_findings:
            report_sections.append(
                f"- {finding['description']} "
                f"(置信度: {finding['confidence']:.0%})")

        # 3. 诊断意见
        report_sections.append("\n【诊断意见】")
        for finding in ai_findings:
            if finding["severity"] == "critical":
                report_sections.append(f"⚠ {finding['finding_type']}: 建议进一步检查")
            elif finding["severity"] == "warning":
                report_sections.append(f"△ {finding['finding_type']}: 建议随访观察")
            else:
                report_sections.append(f"○ {finding['finding_type']}: 未见明显异常")

        # 4. ICD-10 编码
        icd_codes = self._assign_icd_codes(ai_findings)
        report_sections.append(f"\n【ICD-10编码】{', '.join(icd_codes)}")

        # 5. 免责声明
        report_sections.append("\n注: 本报告由 AI 辅助生成，仅供医生参考，最终诊断以医生判断为准")

        return "\n".join(report_sections)

    def _assign_icd_codes(self, findings):
        """ICD-10 编码映射"""
        code_map = {
            "pulmonary_nodule": "R91.8",  # 肺结节
            "fracture": "S02.9",          # 骨折
            "pneumonia": "J18.9",         # 肺炎
            "mass": "R22.9",              # 肿块
        }
        return [code_map.get(f["finding_type"], "R69") for f in findings]
```

## 异常场景补充

### 场景：PACS 存储满

```
触发：PACS 存储使用率 > 95% → 无法接收新影像
检测：
  1. 存储使用率 > 80% → 告警
  2. 存储使用率 > 95% → 严重告警
处理：
  1. 自动归档 1 年前的影像到冷存储（S3 Glacier）
  2. 归档后从 PACS 删除热存储中的旧影像
  3. 保留元数据（可查询但需从冷存储加载）
预防：分层存储 + 自动归档策略 + 存储扩容预警
```

### 场景：报告生成 NLP 失败

```
触发：NLP 模型加载失败 → 无法生成报告
检测：模型推理超时或返回异常
处理：
  1. 降级：生成基础报告（仅含 AI 检测结果，无自然语言描述）
  2. 标记报告为"待完善"
  3. 医生可手动补充描述
预防：NLP 模型健康检查 + 降级策略
```

## HL7 FHIR 集成完整实现

```python
class FHIRIntegrationService:
    """HL7 FHIR 集成：患者信息同步、检查管理、结果回报"""

    def sync_patient_demographics(self, patient_id):
        """从 HIS 系统同步患者信息到 FHIR"""
        # 1. 从 HIS 获取患者信息
        his_patient = self.his_client.get_patient(patient_id)

        # 2. 映射到 FHIR Patient Resource
        fhir_patient = {
            "resourceType": "Patient",
            "id": patient_id,
            "identifier": [{"system": "urn:his:patient-id", "value": patient_id}],
            "name": [{"family": his_patient["last_name"],
                       "given": [his_patient["first_name"]]}],
            "gender": his_patient["gender"],
            "birthDate": his_patient["birth_date"],
            "address": [{"city": his_patient["city"],
                         "state": his_patient["province"]}],
        }

        # 3. 推送到 FHIR 服务器
        response = self.fhir_client.create("Patient", fhir_patient)
        return {"fhir_id": response["id"], "status": "synced"}

    def report_imaging_result(self, study_id, findings):
        """回报影像检查结果到 FHIR"""
        study = self.db.get_study(study_id)
        patient = self.db.get_patient(study["patient_id"])

        # FHIR DiagnosticReport Resource
        report = {
            "resourceType": "DiagnosticReport",
            "status": "final",
            "category": [{"coding": [{"system": "http://loinc.org",
                                       "code": "LP29684-5", "display": "Radiology"}]}],
            "code": {"coding": [{"system": "http://loinc.org",
                                  "code": study["loinc_code"],
                                  "display": study["study_description"]}]},
            "subject": {"reference": f"Patient/{patient['fhir_id']}"},
            "effectiveDateTime": study["study_date"],
            "result": [{"reference": f"Observation/{f['observation_id']}"}
                       for f in findings],
            "conclusion": findings[0]["conclusion"] if findings else None,
        }

        return self.fhir_client.create("DiagnosticReport", report)
```

## 图像预处理流水线

```python
class ImagePreprocessingPipeline:
    """医学影像预处理流水线"""

    def preprocess(self, image_array, modality):
        """预处理步骤链"""
        # 1. 像素值归一化
        normalized = self._normalize(image_array, modality)

        # 2. 窗宽窗位调整（CT 特有）
        if modality == "CT":
            # 肺窗: WL=-600, WW=1500
            normalized = self._apply_window(normalized, window_level=-600, window_width=1500)

        # 3. 噪声抑制
        denoised = self._denoise(normalized, modality)

        # 4. 图像配准（多序列 MRI）
        if modality == "MRI" and image_array.ndim == 4:
            denoised = self._register_series(denoised)

        # 5. 质量评估
        quality_score = self._assess_quality(denoised)
        if quality_score < 0.7:
            self.alert(f"图像质量低: score={quality_score:.2f}")

        return {"image": denoised, "quality_score": quality_score}

    def _normalize(self, image, modality):
        """归一化到 [0, 1]"""
        if modality == "CT":
            # Hounsfield 单位范围: -1000 到 3000
            return (image - (-1000)) / 4000
        elif modality == "MRI":
            return (image - image.min()) / (image.max() - image.min())
        return image / 255.0

    def _apply_window(self, image, window_level, window_width):
        """窗宽窗位调整"""
        lower = window_level - window_width / 2
        upper = window_level + window_width / 2
        windowed = np.clip(image, lower, upper)
        return (windowed - lower) / window_width
```

## 优先级调度

```python
class StudyPriorityScheduler:
    """检查优先级调度：急诊优先、负载均衡"""

    PRIORITY_LEVELS = {
        "emergency": {"weight": 100, "description": "急诊", "target_tat_hours": 1},
        "urgent": {"weight": 50, "description": "加急", "target_tat_hours": 4},
        "routine": {"weight": 10, "description": "常规", "target_tat_hours": 24},
    }

    def assign_radiologist(self, study_id):
        """分配阅片医生"""
        study = self.db.get_study(study_id)
        priority = study.get("priority", "routine")

        # 获取当前负载最低的放射科医生
        radiologists = self.db.query(
            "SELECT r.*, COUNT(CASE WHEN s.status='reading' THEN 1 END) as active_studies "
            "FROM radiologists r LEFT JOIN studies s ON r.id = s.assigned_radiologist "
            "WHERE r.specialty = %s AND r.status = 'available' "
            "GROUP BY r.id ORDER BY active_studies ASC",
            study["modality"])

        if not radiologists:
            self.alert(f"无可用 {study['modality']} 阅片医生")
            return {"status": "queued"}

        # 分配负载最低的医生
        assigned = radiologists[0]
        self.db.update("studies",
            {"assigned_radiologist": assigned["id"], "status": "reading"},
            {"id": study_id})
        self.notify_radiologist(assigned["id"], study_id)

        return {"radiologist_id": assigned["id"],
                "active_studies": assigned["active_studies"],
                "priority": priority}
```

## 异常场景补充

### 场景：FHIR 接口版本不兼容

```
触发：HIS 系统升级 → FHIR 接口从 v3 变为 v4 → 数据映射失败
检测：
  1. FHIR 接口返回 422 Unprocessable Entity → 版本不兼容
  2. 患者信息同步失败 → 告警
处理：
  1. 使用 FHIR 版本适配器（v3→v4 映射层）
  2. 映射失败的字段 → 人工补录
  3. 更新内部 FHIR 客户端到 v4
预防：FHIR 接口版本检测 + 自动适配层
```

### 场景：预处理 GPU OOM

```
触发：3D MRI 序列（512×512×256）→ 预处理时 GPU 内存不足
检测：
  1. GPU 内存使用 > 90% → 告警
  2. 预处理进程被 kill (OOM) → 严重告警
处理：
  1. 降级：分切片处理（每次只加载 32 个切片）
  2. 使用 CPU 预处理（慢但不会 OOM）
  3. 增加 GPU 内存或升级硬件
预防：分切片处理策略 + GPU 内存监控 + CPU 备用路径
```

## HL7 FHIR 完整集成系统

```python
import json
import hashlib
import requests
from datetime import datetime, date
from typing import Dict, List, Optional
from dataclasses import dataclass, field
from enum import Enum


class FHIRVersion(Enum):
    STU3 = "3.0"
    R4 = "4.0"
    R5 = "5.0"


@dataclass
class FHIRPatient:
    """FHIR Patient 资源映射"""
    patient_id: str
    name_family: str
    name_given: str
    birth_date: str          # YYYY-MM-DD
    gender: str              # male / female / other / unknown
    identifier_mrn: str      # 门诊号
    identifier_id_card: str  # 身份证号
    phone: str = ""
    address_text: str = ""


@dataclass
class FHIROrder:
    """FHIR ServiceRequest（检查医嘱）资源映射"""
    order_id: str
    patient_id: str
    orderer_id: str          # 开单医生 ID
    orderer_name: str
    order_date: str
    procedure_code: str      # 检查类型编码（如 CT、MRI）
    procedure_display: str   # 检查类型描述
    body_site_code: str      # 检查部位编码
    body_site_display: str   # 检查部位描述
    priority: str = "routine"  # routine / urgent / stat
    reason_code: str = ""
    reason_display: str = ""
    clinical_info: str = ""  # 临床信息/病史


@dataclass
class FHIRResult:
    """FHIR DiagnosticReport（诊断报告）资源映射"""
    report_id: str
    patient_id: str
    study_id: str
    performer_id: str        # 报告医生 ID
    performer_name: str
    status: str              # registered / preliminary / final / amended
    conclusion: str          # 结论
    conclusion_code: List[str] = field(default_factory=list)  # ICD-10 编码
    study_uid: str = ""      # DICOM Study Instance UID
    effective_date: str = "" # 检查日期
    issued_date: str = ""    # 报告发布日期


class FHIRIntegrationService:
    """HL7 FHIR 完整集成：患者信息同步、医嘱管理、报告回传"""

    def __init__(self, config, db_client, audit_logger):
        self.base_url = config["fhir_base_url"]  # 如 https://his.hospital.com/fhir
        self.fhir_version = FHIRVersion(config.get("fhir_version", "4.0"))
        self.db = db_client
        self.audit = audit_logger
        self.session = requests.Session()
        self.session.headers.update({
            "Content-Type": "application/fhir+json",
            "Accept": "application/fhir+json",
            "Authorization": f"Bearer {config['fhir_token']}",
        })

    # ============ 患者信息同步 ============

    def sync_patient_demographics(self, patient_id: str) -> FHIRPatient:
        """从 HIS 系统同步患者人口学信息"""
        fhir_patient = self._fetch_fhir_resource("Patient", patient_id)

        # 解析 FHIR Patient 资源
        name = fhir_patient.get("name", [{}])[0]
        identifiers = {
            id_obj.get("type", {}).get("coding", [{}])[0].get("code"): id_obj.get("value")
            for id_obj in fhir_patient.get("identifier", [])
        }

        patient = FHIRPatient(
            patient_id=fhir_patient.get("id", patient_id),
            name_family=name.get("family", ""),
            name_given=" ".join(name.get("given", [])),
            birth_date=fhir_patient.get("birthDate", ""),
            gender=fhir_patient.get("gender", "unknown"),
            identifier_mrn=identifiers.get("MR", ""),
            identifier_id_card=identifiers.get("IDC", ""),
            phone=fhir_patient.get("telecom", [{}])[0].get("value", "")
                if fhir_patient.get("telecom") else "",
            address_text=fhir_patient.get("address", [{}])[0].get("text", "")
                if fhir_patient.get("address") else "",
        )

        # 持久化到本地数据库（PHI 字段脱敏存储）
        self.db.upsert("patients", {
            "patient_id": patient.patient_id,
            "name": f"{patient.name_family}{patient.name_given}",
            "birth_date": patient.birth_date,
            "gender": patient.gender,
            "mrn": patient.identifier_mrn,
            "id_card_hash": self._hash_phi(patient.identifier_id_card),
            "phone_encrypted": self._encrypt_phi(patient.phone),
            "address_text": patient.address_text,
            "fhir_version": self.fhir_version.value,
            "synced_at": datetime.now(),
        }, conflict_key="patient_id")

        self.audit.log("patient_sync", patient_id=patient_id,
                        action="demographics_synced")
        return patient

    # ============ 医嘱管理 ============

    def fetch_pending_orders(self, department: str = None) -> List[FHIROrder]:
        """从 HIS 拉取待检查医嘱"""
        params = {
            "status": "active",
            "category": "imaging",
        }
        if department:
            params["based-on"] = department

        response = self.session.get(
            f"{self.base_url}/ServiceRequest", params=params)
        response.raise_for_status()
        bundle = response.json()

        orders = []
        for entry in bundle.get("entry", []):
            sr = entry.get("resource", {})
            order = self._parse_service_request(sr)
            if order:
                orders.append(order)
                # 保存到本地
                self.db.upsert("imaging_orders", {
                    "order_id": order.order_id,
                    "patient_id": order.patient_id,
                    "orderer_name": order.orderer_name,
                    "procedure_code": order.procedure_code,
                    "procedure_display": order.procedure_display,
                    "body_site": order.body_site_display,
                    "priority": order.priority,
                    "clinical_info": order.clinical_info,
                    "status": "pending",
                    "received_at": datetime.now(),
                }, conflict_key="order_id")

        self.audit.log("order_fetch", count=len(orders), department=department)
        return orders

    def update_order_status(self, order_id: str, status: str):
        """更新医嘱状态（如开始检查、完成检查）"""
        fhir_status_map = {
            "pending": "active",
            "in_progress": "in-progress",
            "completed": "completed",
            "cancelled": "revoked",
        }
        fhir_status = fhir_status_map.get(status, "active")

        # 更新 HIS 中的医嘱状态
        response = self.session.patch(
            f"{self.base_url}/ServiceRequest/{order_id}",
            json=[{
                "op": "replace",
                "path": "/status",
                "value": fhir_status,
            }]
        )
        response.raise_for_status()

        # 同步更新本地
        self.db.update("imaging_orders",
                       {"status": status, "updated_at": datetime.now()},
                       {"order_id": order_id})

        self.audit.log("order_status_update", order_id=order_id, status=status)

    # ============ 报告回传 ============

    def submit_diagnostic_report(self, result: FHIRResult) -> Dict:
        """将诊断报告回传到 HIS 系统"""
        fhir_report = self._build_diagnostic_report_resource(result)

        response = self.session.post(
            f"{self.base_url}/DiagnosticReport",
            json=fhir_report)
        response.raise_for_status()

        his_id = response.json().get("id")
        self.db.update("radiology_reports",
                       {"his_report_id": his_id, "transmitted_at": datetime.now()},
                       {"report_id": result.report_id})

        self.audit.log("report_submit", report_id=result.report_id,
                        his_id=his_id, status=result.status)
        return {"status": "submitted", "his_report_id": his_id}

    def amend_report(self, report_id: str, amended_conclusion: str,
                     amended_codes: List[str], amender_id: str):
        """修正已签发的报告"""
        original = self.db.query(
            "SELECT * FROM radiology_reports WHERE report_id = %s", report_id)

        # 创建修正版报告
        result = FHIRResult(
            report_id=f"{report_id}-amend-{datetime.now().strftime('%Y%m%d%H%M%S')}",
            patient_id=original["patient_id"],
            study_id=original["study_id"],
            performer_id=amender_id,
            performer_name="",
            status="amended",
            conclusion=amended_conclusion,
            conclusion_code=amended_codes,
            study_uid=original.get("study_uid", ""),
            effective_date=original["effective_date"],
            issued_date=datetime.now().isoformat(),
        )

        # 回传修正报告到 HIS
        submit_result = self.submit_diagnostic_report(result)

        # 标记原报告为"已修正"
        self.db.update("radiology_reports",
                       {"status": "amended", "amended_by": amender_id,
                        "amended_at": datetime.now()},
                       {"report_id": report_id})

        self.audit.log("report_amend", original_id=report_id,
                        amended_id=result.report_id, amender=amender_id)
        return submit_result

    # ============ FHIR 资源构建 ============

    def _build_diagnostic_report_resource(self, result: FHIRResult) -> Dict:
        """构建 FHIR DiagnosticReport 资源"""
        resource = {
            "resourceType": "DiagnosticReport",
            "status": result.status,
            "category": [{
                "coding": [{
                    "system": "http://terminology.hl7.org/CodeSystem/v2-0074",
                    "code": "RAD",
                    "display": "Radiology",
                }]
            }],
            "code": {
                "coding": [{
                    "system": "http://loinc.org",
                    "code": "18748-4",
                    "display": "Diagnostic imaging report",
                }]
            },
            "subject": {"reference": f"Patient/{result.patient_id}"},
            "effectiveDateTime": result.effective_date,
            "issued": result.issued_date,
            "performer": [{
                "reference": f"Practitioner/{result.performer_id}",
                "display": result.performer_name,
            }],
            "resultsInterpreter": [{
                "reference": f"Practitioner/{result.performer_id}",
            }],
            "conclusion": result.conclusion,
            "conclusionCode": [
                {"coding": [{"system": "http://hl7.org/fhir/sid/icd-10",
                             "code": code} for code in result.conclusion_code]}
            ],
            "presentedForm": [{
                "contentType": "application/pdf",
                "url": f"https://pacs.hospital.com/reports/{result.report_id}.pdf",
            }],
        }

        # 关联 DICOM 影像
        if result.study_uid:
            resource["imagingStudy"] = [{
                "reference": f"ImagingStudy/{result.study_uid}",
            }]

        return resource

    def _parse_service_request(self, sr: Dict) -> Optional[FHIROrder]:
        """解析 FHIR ServiceRequest 资源为内部模型"""
        try:
            code = sr.get("code", {}).get("coding", [{}])[0]
            body_site = sr.get("bodySite", [{}])
            body_site_coding = body_site[0].get("coding", [{}])[0] if body_site else {}
            reason = sr.get("reasonCode", [{}])
            reason_coding = reason[0].get("coding", [{}])[0] if reason else {}
            supporting = sr.get("supportingInfo", [{}])
            clinical = supporting[0].get("display", "") if supporting else ""

            return FHIROrder(
                order_id=sr.get("id", ""),
                patient_id=sr.get("subject", {}).get("reference", "").replace("Patient/", ""),
                orderer_id=sr.get("requester", {}).get("reference", "").replace("Practitioner/", ""),
                orderer_name=sr.get("requester", {}).get("display", ""),
                order_date=sr.get("authoredOn", ""),
                procedure_code=code.get("code", ""),
                procedure_display=code.get("display", ""),
                body_site_code=body_site_coding.get("code", ""),
                body_site_display=body_site_coding.get("display", ""),
                priority=sr.get("priority", "routine"),
                reason_code=reason_coding.get("code", ""),
                reason_display=reason_coding.get("display", ""),
                clinical_info=clinical,
            )
        except (KeyError, IndexError) as e:
            self.audit.log("fhir_parse_error", error=str(e), resource=sr.get("id"))
            return None

    def _fetch_fhir_resource(self, resource_type: str, resource_id: str) -> Dict:
        """获取 FHIR 资源（自动适配版本）"""
        response = self.session.get(
            f"{self.base_url}/{resource_type}/{resource_id}")
        if response.status_code == 422:
            # 版本不匹配，尝试降级请求
            self.audit.log("fhir_version_mismatch", resource_type=resource_type,
                            resource_id=resource_id)
        response.raise_for_status()
        return response.json()

    def _hash_phi(self, phi_value: str) -> str:
        """对 PHI 字段进行脱敏哈希"""
        return hashlib.sha256(phi_value.encode()).hexdigest()[:32]

    def _encrypt_phi(self, phi_value: str) -> str:
        """对 PHI 字段加密（可逆，用于合法访问时解密）"""
        # 实际生产应使用 AES-256 或国密 SM4
        import base64
        return base64.b64encode(phi_value.encode()).decode()
```

## 影像预处理完整流水线

```python
import numpy as np
from typing import Dict, Tuple, Optional, List
from dataclasses import dataclass
from enum import Enum


class PreprocessStep(Enum):
    WINDOW_LEVEL = "window_level"
    NOISE_REDUCTION = "noise_reduction"
    REGISTRATION = "registration"
    QUALITY_CHECK = "quality_check"


@dataclass
class WindowPreset:
    """窗宽窗位预设（HU 值）"""
    name: str
    center: int    # 窗位（Window Center）
    width: int     # 窗宽（Window Width）


WINDOW_PRESETS = {
    "brain":       WindowPreset("脑窗",     center=40,   width=80),
    "bone":        WindowPreset("骨窗",     center=400,  width=1800),
    "lung":        WindowPreset("肺窗",     center=-600, width=1500),
    "mediastinum": WindowPreset("纵隔窗",   center=50,   width=350),
    "abdomen":     WindowPreset("腹部窗",   center=40,   width=400),
    "liver":       WindowPreset("肝窗",     center=60,   width=150),
}


@dataclass
class QualityScore:
    """影像质量评分"""
    overall_score: float       # 综合评分 0-100
    snr: float                 # 信噪比
    contrast_score: float      # 对比度评分 0-100
    artifact_score: float      # 伪影评分 0-100（越高越无伪影）
    motion_score: float        # 运动伪影评分 0-100
    completeness: float        # 完整性评分 0-100
    is_diagnostic: bool        # 是否达到诊断级别


class ImagePreprocessingPipeline:
    """影像预处理完整流水线：窗宽窗位、降噪、配准、质量评估"""

    def __init__(self, config, gpu_manager=None):
        self.gpu = gpu_manager
        self.config = config
        self.quality_threshold = config.get("quality_threshold", 60)

    def process_study(self, study_id: str, slices: List[np.ndarray],
                      dicom_metadata: Dict) -> Dict:
        """执行完整的预处理流水线"""
        results = {
            "study_id": study_id,
            "steps_completed": [],
            "quality_score": None,
            "processed_slices": [],
        }

        # Step 1: 窗宽窗位调整
        try:
            wl_result = self.adjust_window_level(slices, dicom_metadata)
            results["processed_slices"] = wl_result
            results["steps_completed"].append(PreprocessStep.WINDOW_LEVEL.value)
        except Exception as e:
            results["window_level_error"] = str(e)

        # Step 2: 降噪处理
        try:
            denoised = self.reduce_noise(
                results["processed_slices"], dicom_metadata)
            results["processed_slices"] = denoised
            results["steps_completed"].append(PreprocessStep.NOISE_REDUCTION.value)
        except Exception as e:
            results["noise_reduction_error"] = str(e)

        # Step 3: 配准对齐（多序列影像）
        if dicom_metadata.get("has_multiple_series"):
            try:
                registered = self.register_series(
                    results["processed_slices"], dicom_metadata)
                results["processed_slices"] = registered
                results["steps_completed"].append(PreprocessStep.REGISTRATION.value)
            except Exception as e:
                results["registration_error"] = str(e)

        # Step 4: 质量评估
        quality = self.assess_quality(results["processed_slices"], dicom_metadata)
        results["quality_score"] = quality
        results["steps_completed"].append(PreprocessStep.QUALITY_CHECK.value)

        return results

    def adjust_window_level(self, slices: List[np.ndarray],
                            metadata: Dict) -> List[np.ndarray]:
        """窗宽窗位调整：将 HU 值映射到显示范围"""
        wc = metadata.get("window_center", 40)
        ww = metadata.get("window_width", 400)

        # 尝试匹配预设
        body_part = metadata.get("body_part", "")
        modality = metadata.get("modality", "CT")
        preset_key = self._match_preset(modality, body_part)
        if preset_key and preset_key in WINDOW_PRESETS:
            preset = WINDOW_PRESETS[preset_key]
            wc, ww = preset.center, preset.width

        processed = []
        for slice_img in slices:
            lower = wc - ww / 2
            upper = wc + ww / 2
            windowed = np.clip(slice_img, lower, upper)
            normalized = ((windowed - lower) / (upper - lower) * 255).astype(np.uint8)
            processed.append(normalized)

        return processed

    def _match_preset(self, modality: str, body_part: str) -> Optional[str]:
        """根据检查类型和部位匹配窗宽窗位预设"""
        bp = body_part.lower()
        if "brain" in bp or "head" in bp:
            return "brain"
        elif "lung" in bp or "chest" in bp:
            return "lung"
        elif "bone" in bp or "spine" in bp:
            return "bone"
        elif "liver" in bp:
            return "liver"
        elif "abdomen" in bp:
            return "abdomen"
        return None

    def reduce_noise(self, slices: List[np.ndarray],
                     metadata: Dict) -> List[np.ndarray]:
        """降噪处理"""
        noise_level = metadata.get("noise_estimate", None)
        if noise_level is None:
            noise_level = self._estimate_noise(slices)

        denoised = []
        for slice_img in slices:
            if self.gpu and self.gpu.available:
                result = self._gpu_denoise(slice_img, noise_level)
            else:
                result = self._cpu_denoise(slice_img, noise_level)
            denoised.append(result)

        return denoised

    def _estimate_noise(self, slices: List[np.ndarray]) -> float:
        """估算影像噪声水平（基于背景区域）"""
        sample = slices[len(slices) // 2]
        corner_size = min(32, sample.shape[0] // 8)
        corners = [
            sample[:corner_size, :corner_size],
            sample[:corner_size, -corner_size:],
            sample[-corner_size:, :corner_size],
            sample[-corner_size:, -corner_size:],
        ]
        stds = [np.std(c.astype(float)) for c in corners]
        return min(stds)

    def _gpu_denoise(self, image: np.ndarray, noise_level: float) -> np.ndarray:
        """GPU 加速降噪"""
        try:
            from scipy.ndimage import gaussian_filter
            sigma = max(0.5, noise_level / 50.0)
            return gaussian_filter(image.astype(np.float32), sigma=sigma).astype(image.dtype)
        except Exception:
            return self._cpu_denoise(image, noise_level)

    def _cpu_denoise(self, image: np.ndarray, noise_level: float) -> np.ndarray:
        """CPU 降噪：中值滤波"""
        from scipy.ndimage import median_filter
        sigma = max(1, int(noise_level / 30))
        sigma = min(sigma, 3)
        return median_filter(image, size=sigma * 2 + 1)

    def register_series(self, slices: List[np.ndarray],
                        metadata: Dict) -> List[np.ndarray]:
        """多序列影像配准对齐（刚体配准）"""
        if len(slices) <= 1:
            return slices

        reference = slices[0].astype(np.float64)
        registered = [slices[0]]

        for i in range(1, len(slices)):
            moving = slices[i].astype(np.float64)
            shift = self._compute_center_shift(reference, moving)
            aligned = self._apply_shift(moving, shift)
            registered.append(aligned.astype(slices[i].dtype))

        return registered

    def _compute_center_shift(self, ref: np.ndarray,
                              moving: np.ndarray) -> Tuple[int, int]:
        """计算质心偏移量"""
        ref_center = self._image_center_of_mass(ref)
        mov_center = self._image_center_of_mass(moving)
        return (int(mov_center[0] - ref_center[0]),
                int(mov_center[1] - ref_center[1]))

    def _image_center_of_mass(self, img: np.ndarray) -> Tuple[float, float]:
        """计算图像质心"""
        total = img.sum()
        if total == 0:
            return (img.shape[0] / 2, img.shape[1] / 2)
        y_coords, x_coords = np.indices(img.shape)
        cy = (y_coords * img).sum() / total
        cx = (x_coords * img).sum() / total
        return (cy, cx)

    def _apply_shift(self, img: np.ndarray,
                     shift: Tuple[int, int]) -> np.ndarray:
        """平移图像"""
        result = np.zeros_like(img)
        dy, dx = shift
        src_y = slice(max(0, dy), min(img.shape[0], img.shape[0] + dy))
        src_x = slice(max(0, dx), min(img.shape[1], img.shape[1] + dx))
        dst_y = slice(max(0, -dy), min(img.shape[0], img.shape[0] - dy))
        dst_x = slice(max(0, -dx), min(img.shape[1], img.shape[1] - dx))
        result[dst_y, dst_x] = img[src_y, src_x]
        return result

    def assess_quality(self, slices: List[np.ndarray],
                       metadata: Dict) -> QualityScore:
        """影像质量评估"""
        if not slices:
            return QualityScore(0, 0, 0, 0, 0, 0, False)

        # 1. 信噪比
        signal = np.mean([np.mean(s.astype(float)) for s in slices])
        noise = np.mean([np.std(s.astype(float)) for s in slices])
        snr = signal / noise if noise > 0 else 0

        # 2. 对比度评分
        contrast_scores = []
        for s in slices:
            p5, p95 = np.percentile(s.astype(float), [5, 95])
            contrast_scores.append(p95 - p5)
        contrast_score = min(100, np.mean(contrast_scores) / 50 * 100)

        # 3. 伪影评分（基于均匀性）
        artifact_scores = []
        for s in slices[:10]:
            flatness = 1.0 - np.std(s.astype(float)) / (np.mean(s.astype(float)) + 1)
            artifact_scores.append(max(0, flatness * 100))
        artifact_score = np.mean(artifact_scores)

        # 4. 运动伪影评分（相邻切片相关性）
        motion_scores = []
        for i in range(min(len(slices) - 1, 20)):
            corr = np.corrcoef(slices[i].flatten(), slices[i+1].flatten())[0, 1]
            motion_scores.append(max(0, corr * 100))
        motion_score = np.mean(motion_scores) if motion_scores else 100

        # 5. 完整性评分
        expected = metadata.get("expected_number_of_slices", 0)
        completeness = min(100, len(slices) / expected * 100) if expected > 0 else 100

        # 综合评分（加权平均）
        overall = (snr / 5 * 25 + contrast_score * 0.25 +
                   artifact_score * 0.2 + motion_score * 0.15 +
                   completeness * 0.15)
        overall = min(100, max(0, overall))

        return QualityScore(
            overall_score=round(overall, 1),
            snr=round(snr, 2),
            contrast_score=round(contrast_score, 1),
            artifact_score=round(artifact_score, 1),
            motion_score=round(motion_score, 1),
            completeness=round(completeness, 1),
            is_diagnostic=overall >= self.quality_threshold,
        )
```

## 检查优先级调度完整实现

```python
import time
import json
from datetime import datetime, timedelta
from typing import Dict, List, Optional, Tuple
from dataclasses import dataclass, field
from enum import Enum


class StudyPriority(Enum):
    STAT = "stat"            # 急诊（立即处理）
    URGENT = "urgent"        # 紧急（1 小时内）
    ROUTINE = "routine"      # 常规（24 小时内）
    SCREENING = "screening"  # 筛查（72 小时内）


class StudyStatus(Enum):
    RECEIVED = "received"
    ASSIGNED = "assigned"
    IN_PROGRESS = "in_progress"
    DRAFT_REPORT = "draft_report"
    FINAL_REPORT = "final_report"
    SIGNED = "signed"


@dataclass
class Radiologist:
    """放射科医生"""
    doctor_id: str
    name: str
    specialties: List[str]     # 擅长领域：如 ["chest", "neuro"]
    current_load: int = 0
    max_load: int = 8          # 最大并行处理数
    shift_start: str = ""
    shift_end: str = ""
    avg_report_time_min: float = 30.0


@dataclass
class StudyWorkItem:
    """检查工作项"""
    study_id: str
    patient_id: str
    modality: str
    body_part: str
    priority: StudyPriority
    status: StudyStatus
    received_at: datetime
    assigned_to: Optional[str] = None
    assigned_at: Optional[datetime] = None
    started_at: Optional[datetime] = None
    completed_at: Optional[datetime] = None
    turnaround_min: Optional[float] = None
    sla_deadline: Optional[datetime] = None


SLA_CONFIG = {
    StudyPriority.STAT:      timedelta(minutes=30),
    StudyPriority.URGENT:    timedelta(hours=1),
    StudyPriority.ROUTINE:   timedelta(hours=24),
    StudyPriority.SCREENING: timedelta(hours=72),
}


class StudySchedulingService:
    """检查优先级调度：急诊优先 + 医生负载均衡 + 周转时间追踪"""

    def __init__(self, db_client, redis_client, notification_service):
        self.db = db_client
        self.redis = redis_client
        self.notify = notification_service

    def assign_study(self, study_id: str, modality: str, body_part: str,
                     priority: StudyPriority, patient_id: str) -> Dict:
        """分配检查给最合适的放射科医生"""
        available_doctors = self._get_available_radiologists(modality, body_part)

        if not available_doctors:
            self._enqueue_study(study_id, priority)
            return {"status": "queued", "reason": "no_available_radiologist"}

        # 按优先级策略选医生
        if priority == StudyPriority.STAT:
            best = min(available_doctors, key=lambda d: d.current_load)
        else:
            scored = [(d, self._compute_assignment_score(d, modality, body_part))
                      for d in available_doctors]
            scored.sort(key=lambda x: -x[1])
            best = scored[0][0]

        # 执行分配
        now = datetime.now()
        work_item = StudyWorkItem(
            study_id=study_id,
            patient_id=patient_id,
            modality=modality,
            body_part=body_part,
            priority=priority,
            status=StudyStatus.ASSIGNED,
            received_at=now,
            assigned_to=best.doctor_id,
            assigned_at=now,
            sla_deadline=now + SLA_CONFIG[priority],
        )

        self.db.insert("study_work_items", {
            "study_id": study_id,
            "patient_id": patient_id,
            "modality": modality,
            "body_part": body_part,
            "priority": priority.value,
            "status": work_item.status.value,
            "received_at": now,
            "assigned_to": best.doctor_id,
            "assigned_at": now,
            "sla_deadline": work_item.sla_deadline,
        })

        self._update_doctor_load(best.doctor_id, +1)

        self.notify.send(best.doctor_id, {
            "type": "new_study",
            "study_id": study_id,
            "priority": priority.value,
            "modality": modality,
            "body_part": body_part,
            "sla_deadline": work_item.sla_deadline.isoformat(),
        })

        return {
            "status": "assigned",
            "assigned_to": best.doctor_id,
            "doctor_name": best.name,
            "sla_deadline": work_item.sla_deadline.isoformat(),
        }

    def start_study(self, study_id: str, doctor_id: str):
        """医生开始阅片"""
        now = datetime.now()
        self.db.update("study_work_items", {
            "status": StudyStatus.IN_PROGRESS.value,
            "started_at": now,
        }, {"study_id": study_id, "assigned_to": doctor_id})

    def complete_study(self, study_id: str, doctor_id: str) -> Dict:
        """完成检查报告"""
        now = datetime.now()
        item = self.db.query(
            "SELECT * FROM study_work_items WHERE study_id = %s", study_id)
        if not item:
            return {"status": "error", "reason": "study_not_found"}

        received_at = item["received_at"]
        turnaround = (now - received_at).total_seconds() / 60

        self.db.update("study_work_items", {
            "status": StudyStatus.SIGNED.value,
            "completed_at": now,
            "turnaround_min": turnaround,
        }, {"study_id": study_id})

        self._update_doctor_load(doctor_id, -1)
        self._record_turnaround(item["priority"], item["modality"], turnaround)

        # 检查 SLA 违规
        sla_breached = item["sla_deadline"] and now > item["sla_deadline"]
        if sla_breached:
            self._handle_sla_breach(study_id, item, turnaround)

        return {
            "status": "completed",
            "turnaround_min": round(turnaround, 1),
            "sla_breached": sla_breached,
        }

    def _compute_assignment_score(self, doctor: Radiologist,
                                   modality: str, body_part: str) -> float:
        """计算医生匹配评分（越高越适合）"""
        score = 0.0

        # 专科匹配加分
        if body_part in doctor.specialties:
            score += 50.0
        if modality.lower() in [s.lower() for s in doctor.specialties]:
            score += 20.0

        # 负载因子
        load_ratio = doctor.current_load / doctor.max_load if doctor.max_load > 0 else 1.0
        score += (1.0 - load_ratio) * 30.0

        # 效率因子
        if doctor.avg_report_time_min > 0:
            score += max(0, 20.0 - doctor.avg_report_time_min / 2)

        return score

    def _get_available_radiologists(self, modality: str,
                                    body_part: str) -> List[Radiologist]:
        """获取当前可用的放射科医生"""
        now = datetime.now()
        current_time = now.strftime("%H:%M")

        rows = self.db.query(
            "SELECT * FROM radiologists WHERE "
            "shift_start <= %s AND shift_end >= %s AND "
            "current_load < max_load AND is_active = true",
            current_time, current_time)

        return [Radiologist(
            doctor_id=r["doctor_id"],
            name=r["name"],
            specialties=json.loads(r["specialties"]),
            current_load=r["current_load"],
            max_load=r["max_load"],
            shift_start=r["shift_start"],
            shift_end=r["shift_end"],
            avg_report_time_min=r.get("avg_report_time_min", 30.0),
        ) for r in rows]

    def _update_doctor_load(self, doctor_id: str, delta: int):
        """更新医生当前负载"""
        self.db.execute(
            "UPDATE radiologists SET current_load = current_load + %s "
            "WHERE doctor_id = %s", delta, doctor_id)

    def _enqueue_study(self, study_id: str, priority: StudyPriority):
        """加入待分配队列"""
        queue_key = f"study_queue:{priority.value}"
        self.redis.zadd(queue_key, {study_id: self._priority_score(priority)})

    def _priority_score(self, priority: StudyPriority) -> float:
        """优先级数值（越高越优先）"""
        scores = {
            StudyPriority.STAT: 1000,
            StudyPriority.URGENT: 500,
            StudyPriority.ROUTINE: 100,
            StudyPriority.SCREENING: 10,
        }
        return scores.get(priority, 0)

    def _record_turnaround(self, priority: str, modality: str,
                           turnaround_min: float):
        """记录周转时间统计"""
        key = f"turnaround:{priority}:{modality}"
        self.redis.lpush(key, turnaround_min)
        self.redis.ltrim(key, 0, 999)

    def _handle_sla_breach(self, study_id: str, item: Dict,
                           turnaround_min: float):
        """处理 SLA 违规"""
        self.notify.send_alert("sla_breach", {
            "study_id": study_id,
            "priority": item["priority"],
            "sla_deadline": item["sla_deadline"],
            "actual_minutes": round(turnaround_min, 1),
            "overtime_minutes": round(
                turnaround_min - SLA_CONFIG[
                    StudyPriority(item["priority"])
                ].total_seconds() / 60, 1),
        })

    def get_dashboard_stats(self) -> Dict:
        """获取调度看板数据"""
        now = datetime.now()

        # 各状态检查数
        status_counts = {}
        for status in StudyStatus:
            count = self.db.count("study_work_items",
                                   {"status": status.value})
            status_counts[status.value] = count

        # 平均周转时间
        avg_turnaround = {}
        for priority in StudyPriority:
            key = f"turnaround:{priority.value}:all"
            values = self.redis.lrange(key, 0, 99)
            if values:
                avg = sum(float(v) for v in values) / len(values)
                avg_turnaround[priority.value] = round(avg, 1)

        # 医生负载
        doctors = self._get_available_radiologists("all", "all")
        load_distribution = {
            "total_online": len(doctors),
            "avg_load": round(
                sum(d.current_load for d in doctors) / max(len(doctors), 1), 1),
            "max_load": max((d.current_load for d in doctors), default=0),
            "idle_count": sum(1 for d in doctors if d.current_load == 0),
        }

        return {
            "status_counts": status_counts,
            "avg_turnaround_min": avg_turnaround,
            "doctor_load": load_distribution,
        }
```

## 异常场景补充

### 场景：FHIR 接口版本不匹配

```
触发：HIS 系统升级后 FHIR 版本从 R4 变为 R5 → 接口解析失败
检测：
  1. FHIR 请求返回 422 Unprocessable Entity → 版本不匹配
  2. 资源解析异常 → 自动告警
  3. 定期探测 HIS 系统的 capability statement → 版本变化通知
处理：
  1. 根据返回的 FHIR 版本自动切换适配器（R4/R5 双适配器）
  2. 无对应适配器 → 降级为 DICOM-only 模式（不获取患者信息）
  3. 通知集成团队协调 HIS 升级
  4. 保留版本不匹配的请求日志用于事后审计
预防：FHIR 版本自动检测 + 多版本适配器 + 集成变更通知机制
```

### 场景：预处理流水线 GPU OOM

```
触发：大体积影像（如 4D-CT，单次检查 > 2GB）→ GPU 显存溢出
检测：
  1. CUDA out of memory 异常 → 立即告警
  2. GPU 显存使用率 > 90% → 预警
  3. 预处理超时 > 5 分钟 → 可能 OOM 导致进程僵死
处理：
  1. 切片分批处理：将大影像拆分为子区域分别处理
  2. 降级为 CPU 处理（速度慢但内存可控）
  3. 跳过 GPU 加速步骤，仅做基础预处理（窗宽窗位调整）
  4. 释放 GPU 资源后重试
预防：预处理前预估显存需求 + 分批策略 + GPU 监控 + OOM 自动降级
```

## DICOM 路由与工作负载管理完整实现

```python
class DICOMRouter:
    """DICOM 路由：按检查类型 → PACS/AI 服务"""

    ROUTING_RULES = {
        ("CT", "CHEST"): {"pacs": "ct_pacs", "ai": "lung_nodule_detector", "priority": "stat"},
        ("CT", "HEAD"): {"pacs": "ct_pacs", "ai": "stroke_detector", "priority": "stat"},
        ("MRI", "BRAIN"): {"pacs": "mri_pacs", "ai": "brain_tumor_detector", "priority": "urgent"},
        ("XRAY", "CHEST"): {"pacs": "xray_pacs", "ai": "pneumonia_detector", "priority": "routine"},
        ("MG", "BREAST"): {"pacs": "mg_pacs", "ai": "breast_cancer_detector", "priority": "routine"},
    }

    def route_study(self, study):
        """路由检查到正确的目标"""
        modality = study["modality"]
        body_part = study["body_part"]
        key = (modality, body_part)

        rule = self.ROUTING_RULES.get(key, {"pacs": "default_pacs", "priority": "routine"})

        # 1. 发送到 PACS 存档
        self._send_to_pacs(study, rule["pacs"])

        # 2. 发送到 AI 服务（如果有）
        if "ai" in rule:
            self._send_to_ai(study, rule["ai"], rule["priority"])

        # 3. 预取相关历史检查
        self._prefetch_prior_studies(study)

        return {"study_id": study["id"], "routed_to": rule}

    def _prefetch_prior_studies(self, study):
        """预取历史检查（辅助诊断）"""
        priors = self.db.query(
            "SELECT * FROM studies WHERE patient_id = %s "
            "AND modality = %s AND body_part = %s "
            "AND study_date < %s ORDER BY study_date DESC LIMIT 5",
            study["patient_id"], study["modality"],
            study["body_part"], study["study_date"])

        for prior in priors:
            # 将历史检查加载到缓存
            self.pacs_cache.warmup(prior["study_instance_uid"])

class WorkloadBalancer:
    """AI 推理工作负载均衡"""

    def distribute_study(self, study, ai_service):
        """分配检查到 AI 推理节点"""
        nodes = self.db.query(
            "SELECT * FROM ai_inference_nodes "
            "WHERE service = %s AND status = 'healthy'", ai_service)

        # 按 GPU 可用性和队列深度排序
        nodes.sort(key=lambda n: n["queue_depth"] / max(n["gpu_count"], 1))

        if not nodes:
            self.alert(f"AI 服务 {ai_service} 无可用节点")
            return None

        target = nodes[0]
        self.db.update("ai_inference_nodes",
            {"queue_depth": target["queue_depth"] + 1},
            {"id": target["id"]})

        return target["endpoint"]
```

## 放射科报告生成系统

```python
class RadiologyReportService:
    """放射科报告生成：AI 辅助 + 语音 + 审批"""

    def generate_ai_draft(self, study_id, ai_findings):
        """AI 辅助生成报告草稿"""
        study = self.db.get_study(study_id)
        template = self._get_template(study["modality"], study["body_part"])

        # 填充发现
        sections = {
            "检查信息": f"{study['modality']} {study['body_part']}",
            "检查所见": self._format_findings(ai_findings),
            "印象/结论": self._generate_impressions(ai_findings),
            "建议": self._generate_recommendations(ai_findings),
        }

        draft = template.format(**sections)

        report_id = str(uuid4())
        self.db.insert("radiology_reports", {
            "report_id": report_id,
            "study_id": study_id,
            "status": "draft",
            "content": draft,
            "ai_findings": json.dumps(ai_findings),
            "author_id": None,  # 待分配
            "created_at": now()
        })

        # 关键发现通知
        critical = [f for f in ai_findings if f.get("critical")]
        if critical:
            self._notify_critical_finding(study, critical)

        return {"report_id": report_id, "critical_findings": len(critical)}

    def _notify_critical_finding(self, study, findings):
        """关键发现立即通知临床医生"""
        referring_physician = self.db.get_referring_physician(study["accession_number"])

        for finding in findings:
            self.notification.send(
                to=referring_physician["contact"],
                subject=f"关键发现通知: {study['patient_id']}",
                body=f"检查发现: {finding['description']}\n"
                     f"紧急程度: {finding['urgency']}\n"
                     f"请立即查看并采取临床行动",
                priority="critical")

            # 记录通知（合规要求）
            self.db.insert("critical_notifications", {
                "study_id": study["id"],
                "finding": finding["description"],
                "notified_to": referring_physician["id"],
                "notified_at": now(),
                "acknowledged": False
            })

    def approve_report(self, report_id, reviewer_id):
        """审批报告"""
        self.db.update("radiology_reports", {
            "status": "signed",
            "reviewer_id": reviewer_id,
            "signed_at": now()
        }, {"report_id": report_id})

    def add_addendum(self, report_id, author_id, content, reason):
        """添加报告补遗"""
        self.db.insert("report_addenda", {
            "report_id": report_id,
            "author_id": author_id,
            "content": content,
            "reason": reason,
            "created_at": now()
        })
```

## 异常场景补充

### 场景：DICOM 路由错误

```
触发：胸部 CT 被路由到脑卒中检测 AI → 返回无意义结果
检测：
  1. AI 返回"未检测到目标"且置信度极低 → 可能路由错误
  2. 检查类型与 AI 模型不匹配 → 路由错误
处理：
  1. 自动检测路由错误 → 重新路由到正确 AI
  2. 清除错误 AI 结果
  3. 检查路由规则配置
预防：路由前校验 modality + body_part 与 AI 模型匹配 + 自动重路由
```

### 场景：关键发现通知延迟

```
触发：通知队列积压 → 关键发现通知延迟 30 分钟 → 临床风险
检测：
  1. 关键发现通知未在 5 分钟内确认 → 延迟
  2. 通知队列深度 > 50 → 可能延迟
处理：
  1. 关键通知使用最高优先级通道（电话 + SMS + App推送）
  2. 5 分钟未确认 → 升级通知（科室主任）
  3. 15 分钟未确认 → 通知医院总值班
预防：分级通知 + 未确认升级 + 多通道冗余
```

## DICOM 匿名化流水线完整实现

```python
class DICOMAnonymizationPipeline:
    """DICOM 匿名化：PHI 移除 + 像素脱敏 + 审计"""

    PHI_TAGS = {
        0x00100010: "PatientName",
        0x00100020: "PatientID",
        0x00100030: "PatientBirthDate",
        0x00100040: "PatientSex",
        0x00101010: "PatientAge",
        0x00101040: "PatientAddress",
        0x00100010: "PatientName",
        0x00081050: "PerformingPhysicianName",
        0x00081060: "NameOfPhysiciansReadingStudy",
        0x00100032: "PatientBirthTime",
    }

    def anonymize_study(self, study_instance_uid, purpose="research"):
        """匿名化检查"""
        study = self.pacs.retrieve_study(study_instance_uid)

        # 1. 生成一致的伪名（同一患者跨检查保持一致）
        pseudonym_map = self._get_pseudonym_map(study["patient_id"])

        anonymized_instances = []
        for instance in study["instances"]:
            ds = pydicom.dcmread(instance["data"])

            # 2. 替换 PHI 标签
            for tag, name in self.PHI_TAGS.items():
                if tag in ds:
                    if name == "PatientName":
                        ds[tag].value = pseudonym_map["name"]
                    elif name == "PatientID":
                        ds[tag].value = pseudonym_map["id"]
                    elif name == "PatientBirthDate":
                        ds[tag].value = pseudonym_map["birth_year"] + "0101"
                    else:
                        ds[tag].value = "ANONYMOUS"

            # 3. 移除私有标签
            ds.remove_private_tags()

            # 4. 像素脱敏（移除图像中的文字叠加）
            if self._has_burned_in_phi(ds):
                ds.PixelData = self._remove_text_overlay(ds)

            anonymized_instances.append(ds)

        # 5. 验证匿名化
        self._verify_anonymization(anonymized_instances)

        # 6. 存储到研究 PACS
        research_study_uid = self.pacs.store_anonymized(anonymized_instances)

        # 7. 审计记录
        self.db.insert("anonymization_audit", {
            "original_study_uid": study_instance_uid,
            "anonymized_study_uid": research_study_uid,
            "purpose": purpose,
            "phi_tags_removed": len(self.PHI_TAGS),
            "pixel_anonymized": any(self._has_burned_in_phi(pydicom.dcmread(i["data"]))
                                   for i in study["instances"]),
            "performed_at": now(),
            "performed_by": "system"
        })

        return {"anonymized_study_uid": research_study_uid}

    def _get_pseudonym_map(self, patient_id):
        """获取一致的伪名映射"""
        existing = self.db.query_one(
            "SELECT * FROM pseudonym_mapping WHERE original_patient_id = %s",
            patient_id)
        if existing:
            return {"name": existing["pseudonym_name"],
                    "id": existing["pseudonym_id"],
                    "birth_year": existing["pseudonym_birth_year"]}

        # 生成新伪名
        pseudonym_id = f"RES-{str(uuid4())[:8]}"
        pseudonym_name = f"ResearchPatient-{pseudonym_id}"

        self.db.insert("pseudonym_mapping", {
            "original_patient_id": patient_id,
            "pseudonym_id": pseudonym_id,
            "pseudonym_name": pseudonym_name,
            "pseudonym_birth_year": "1900",
            "created_at": now()
        })

        return {"name": pseudonym_name, "id": pseudonym_id, "birth_year": "1900"}

    def _verify_anonymization(self, instances):
        """验证匿名化完整性"""
        for ds in instances:
            for tag in self.PHI_TAGS:
                if tag in ds:
                    value = str(ds[tag].value)
                    if value not in ["ANONYMOUS", ""] and not value.startswith("ResearchPatient"):
                        raise AnonymizationVerificationError(
                            f"PHI 未移除: tag={hex(tag)} value={value}")
```

## PACS 存储管理

```python
class PACSArchiveManager:
    """PACS 存储管理：分层 + 归档 + 灾备"""

    STORAGE_TIERS = {
        "hot": {"medium": "SSD", "retention_days": 30, "cost_per_gb": 0.10},
        "warm": {"medium": "HDD", "retention_days": 365, "cost_per_gb": 0.03},
        "cold": {"medium": "S3 Glacier", "retention_days": 2555, "cost_per_gb": 0.005},
    }

    def check_storage_capacity(self):
        """检查存储容量"""
        total_gb = self._get_total_storage_gb()
        used_gb = self._get_used_storage_gb()
        usage_pct = used_gb / max(total_gb, 1) * 100

        if usage_pct > 80:
            self.alert(f"PACS 存储使用率 {usage_pct:.0f}%，建议扩容或归档旧数据")

        return {"total_gb": total_gb, "used_gb": used_gb,
                "usage_pct": round(usage_pct, 1)}

    def archive_old_studies(self, days_threshold=30):
        """归档旧检查到冷存储"""
        old_studies = self.db.query(
            "SELECT study_instance_uid, SUM(file_size_bytes) as total_bytes "
            "FROM studies WHERE study_date < NOW() - INTERVAL %s DAY "
            "AND storage_tier = 'hot' GROUP BY study_instance_uid",
            days_threshold)

        archived_count = 0
        freed_gb = 0
        for study in old_studies:
            # 1. 验证数据完整性（checksum）
            if not self._verify_study_integrity(study["study_instance_uid"]):
                self.alert(f"检查 {study['study_instance_uid']} 完整性验证失败，跳过归档")
                continue

            # 2. 迁移到冷存储
            self.s3.upload_study(study["study_instance_uid"],
                storage_class="GLACIER")

            # 3. 更新存储层级
            self.db.update("studies",
                {"storage_tier": "cold", "archived_at": now()},
                {"study_instance_uid": study["study_instance_uid"]})

            freed_gb += study["total_bytes"] / 1073741824
            archived_count += 1

        return {"archived_studies": archived_count, "freed_gb": round(freed_gb, 1)}

    def verify_dr_replication(self):
        """验证灾备复制状态"""
        primary_count = self.db.count("studies", storage_tier__ne="deleted")
        dr_count = self.dr_pacs.count_studies()

        if primary_count != dr_count:
            self.alert(f"灾备数据不一致: 主站 {primary_count} 条, DR {dr_count} 条")

        # RPO 检查（最近同步时间）
        last_sync = self.db.query_one(
            "SELECT MAX(synced_at) as last FROM dr_sync_log")["last"]

        rpo_hours = (now() - last_sync).total_seconds() / 3600 if last_sync else 999

        return {"primary_count": primary_count, "dr_count": dr_count,
                "consistent": primary_count == dr_count,
                "rpo_hours": round(rpo_hours, 1),
                "rpo_compliant": rpo_hours <= 1}
```

## 异常场景补充

### 场景：匿名化遗漏图像内嵌 PHI

```
触发：DICOM 图像中有烧录的患者姓名文字 → 匿名化未移除 → PHI 泄露
检测：
  1. OCR 扫描匿名化后图像 → 检测残留文字
  2. 发现文字包含人名 → 匿名化不完整
处理：
  1. 重新运行像素脱敏
  2. 已分发的匿名化数据 → 紧急回收
  3. 检查 OCR 漏检原因
预防：匿名化后 OCR 验证 + 像素脱敏 + 人工抽检
```

### 场景：PACS 主存储故障

```
触发：主 PACS 存储故障 → 无法访问近期检查 → 影响诊断
检测：
  1. PACS 健康检查失败 → 告警
  2. 医生无法调阅影像 → 严重
处理：
  1. 激活 DR 站点（RTO 目标 4 小时）
  2. 30 天内检查从 DR 提供服务
  3. 更早检查从冷存储按需恢复
  4. 主站修复后反向同步
预防：DR 站点 + 自动故障切换 + 定期灾备演练
```

## DICOM 匿名化深度实现：PHI 移除与验证引擎

```python
class PHIFieldRemovalEngine:
    """PHI 字段移除引擎：DICOM Tag 策略化替换"""

    # DICOM 标准中需要移除/替换的 PHI Tag 完整清单
    PHI_REMOVAL_POLICY = {
        # 患者身份信息 → 替换为一致伪名
        0x00100010: {"name": "PatientName", "action": "pseudonymize", "group": "identity"},
        0x00100020: {"name": "PatientID", "action": "pseudonymize", "group": "identity"},
        0x00100030: {"name": "PatientBirthDate", "action": "generalize", "group": "identity",
                      "strategy": "keep_year_only"},
        0x00100040: {"name": "PatientSex", "action": "keep", "group": "identity"},  # 研究需要
        0x00101010: {"name": "PatientAge", "action": "keep", "group": "identity"},
        0x00101040: {"name": "PatientAddress", "action": "remove", "group": "identity"},
        0x00100032: {"name": "PatientBirthTime", "action": "remove", "group": "identity"},
        0x00101000: {"name": "OtherPatientIDs", "action": "remove", "group": "identity"},
        0x00101001: {"name": "OtherPatientNames", "action": "remove", "group": "identity"},
        0x00101050: {"name": "PatientInsurancePlanCodeSeq", "action": "remove", "group": "identity"},

        # 医生/机构信息 → 通用化
        0x00081050: {"name": "PerformingPhysicianName", "action": "generalize",
                      "strategy": "replace_with_role"},
        0x00081060: {"name": "NameOfPhysiciansReadingStudy", "action": "generalize",
                      "strategy": "replace_with_role"},
        0x00080080: {"name": "InstitutionName", "action": "generalize",
                      "strategy": "replace_with_code"},
        0x00080081: {"name": "InstitutionAddress", "action": "remove", "group": "institution"},
        0x00081040: {"name": "InstitutionalDepartmentName", "action": "generalize",
                      "strategy": "replace_with_code"},

        # 检查标识信息 → 重新生成
        0x00100020: {"name": "StudyID", "action": "regenerate", "group": "study"},
        0x00081030: {"name": "StudyDescription", "action": "keep", "group": "study"},
        0x0020000D: {"name": "StudyInstanceUID", "action": "regenerate_uid",
                      "group": "study"},  # UID 重映射保证 DICOM 引用完整
        0x0020000E: {"name": "SeriesInstanceUID", "action": "regenerate_uid",
                      "group": "study"},
        0x00080018: {"name": "SOPInstanceUID", "action": "regenerate_uid",
                      "group": "study"},

        # 访问信息 → 移除
        0x00081090: {"name": "RecordedOccurrencesSeq", "action": "remove"},
        0x00380010: {"name": "AdmissionID", "action": "remove"},
        0x00380020: {"name": "AdmittingDate", "action": "remove"},
    }

    def anonymize_dataset(self, ds, pseudonym_map, uid_mapping):
        """匿名化单个 DICOM 数据集"""
        # 遍历所有需要处理的 Tag
        for tag, policy in self.PHI_REMOVAL_POLICY.items():
            if tag not in ds:
                continue

            action = policy["action"]

            if action == "pseudonymize":
                # 替换为一致的伪名（同一患者跨检查保持一致）
                if policy["name"] == "PatientName":
                    ds[tag].value = pseudonym_map["name"]
                elif policy["name"] == "PatientID":
                    ds[tag].value = pseudonym_map["id"]

            elif action == "generalize":
                strategy = policy.get("strategy")
                if strategy == "keep_year_only":
                    # 只保留出生年份，日期设为 1 月 1 日
                    original = str(ds[tag].value)
                    ds[tag].value = original[:4] + "0101" if len(original) >= 4 else "19000101"
                elif strategy == "replace_with_role":
                    ds[tag].value = "ANONYMOUS_PHYSICIAN"
                elif strategy == "replace_with_code":
                    ds[tag].value = f"INSTITUTION_{hashlib.md5(str(ds[tag].value).encode()).hexdigest()[:6]}"

            elif action == "regenerate_uid":
                # UID 重映射（保持 DICOM 引用完整性）
                original_uid = str(ds[tag].value)
                if original_uid not in uid_mapping:
                    uid_mapping[original_uid] = generate_dicom_uid()
                ds[tag].value = uid_mapping[original_uid]

            elif action == "regenerate":
                ds[tag].value = f"ANON-{str(uuid4())[:8]}"

            elif action == "remove":
                del ds[tag]

            elif action == "keep":
                pass  # 保留原值

        # 移除所有私有标签
        ds.remove_private_tags()

        # 添加匿名化追踪 Tag（不包含原始信息）
        ds.add_new(0x00120010, "LO", "ANONYMIZATION_SERVICE")
        ds.add_new(0x00120011, "DA", now().strftime("%Y%m%d"))

        return ds


class PixelAnonymizationEngine:
    """像素数据匿名化：移除图像中烧录的 PHI 文字"""

    def detect_burned_in_phi(self, ds):
        """检测图像中是否有烧录的 PHI 文字"""
        # 策略 1：检查 DICOM 标签指示
        if hasattr(ds, "BurnedInAnnotation") and ds.BurnedInAnnotation == "YES":
            return True

        # 策略 2：OCR 扫描检测
        pixel_array = ds.pixel_array
        # 预处理：增强对比度、灰度化
        processed = self._preprocess_for_ocr(pixel_array)

        # 只扫描常见文字叠加区域（四角和底部）
        regions_of_interest = self._extract_text_regions(processed)
        detected_text = []
        for region in regions_of_interest:
            text = self.ocr_engine.extract_text(region)
            if text.strip():
                detected_text.append(text.strip())

        # 检查是否包含姓名/ID/日期等 PHI 模式
        phi_patterns = [
            r"[A-Z][a-z]+ [A-Z][a-z]+",  # 姓名模式
            r"\d{4}-\d{2}-\d{2}",          # 日期模式
            r"[A-Z]{2}\d{6,}",             # 患者 ID 模式
            r"ID[:\s]\d+",                 # ID 标识
        ]
        import re
        for text in detected_text:
            for pattern in phi_patterns:
                if re.search(pattern, text):
                    return True

        return len(detected_text) > 0

    def remove_text_overlay(self, ds):
        """移除图像中的文字叠加层"""
        pixel_array = ds.pixel_array.copy()

        # 检测文字区域
        regions_of_interest = self._extract_text_regions(pixel_array)

        for region in regions_of_interest:
            # 使用 OCR 定位精确文字区域
            text_boxes = self.ocr_engine.detect_text_boxes(region)

            for box in text_boxes:
                x, y, w, h = box["x"], box["y"], box["w"], box["h"]

                # 用周围像素的平均值填充文字区域
                border_pixels = self._extract_border_pixels(pixel_array, x, y, w, h)
                fill_value = np.median(border_pixels)
                pixel_array[y:y+h, x:x+w] = fill_value

        # 更新像素数据
        ds.PixelData = pixel_array.tobytes()
        ds.BurnedInAnnotation = "NO"  # 标记已清除烧录文字

        return ds


class AnonymizationVerificationService:
    """匿名化验证服务：确保无 PHI 残留"""

    def verify_anonymized_study(self, anonymized_instances, pseudonym_map):
        """验证匿名化后的检查数据"""
        violations = []

        for i, ds in enumerate(anonymized_instances):
            # 1. 验证 DICOM Tag 中无 PHI
            tag_violations = self._check_dicom_tags(ds, pseudonym_map)
            violations.extend(tag_violations)

            # 2. 验证像素数据中无烧录 PHI
            pixel_violations = self._check_pixel_phi(ds)
            violations.extend(pixel_violations)

            # 3. 验证 UID 重映射完整性
            uid_violations = self._check_uid_consistency(ds, i)
            violations.extend(uid_violations)

        if violations:
            # 验证失败 → 阻止分发
            self.alerting.send(
                severity="critical",
                title="匿名化验证失败",
                message=f"发现 {len(violations)} 处 PHI 残留，已阻止分发",
                details=violations,
            )
            return {"verified": False, "violations": violations}

        return {"verified": True, "violations": []}

    def _check_dicom_tags(self, ds, pseudonym_map):
        """检查 DICOM Tag 中是否有 PHI 残留"""
        violations = []
        phi_checker = PHIFieldRemovalEngine()

        for tag, policy in phi_checker.PHI_REMOVAL_POLICY.items():
            if tag not in ds:
                continue

            value = str(ds[tag].value)

            if policy["action"] == "remove" and value:
                violations.append({
                    "type": "tag_not_removed",
                    "tag": hex(tag),
                    "name": policy["name"],
                    "value_preview": value[:20],
                })

            elif policy["action"] == "pseudonymize":
                # 验证值已被替换
                if value not in [pseudonym_map["name"], pseudonym_map["id"]]:
                    if not value.startswith("ResearchPatient") and value != "ANONYMOUS":
                        violations.append({
                            "type": "phi_in_tag",
                            "tag": hex(tag),
                            "name": policy["name"],
                            "reason": "值未被伪名替换",
                        })

        return violations

    def _check_pixel_phi(self, ds):
        """检查像素数据中是否有 PHI"""
        pixel_checker = PixelAnonymizationEngine()
        has_phi = pixel_checker.detect_burned_in_phi(ds)

        if has_phi:
            return [{"type": "burned_in_phi",
                      "sop_instance_uid": str(ds.SOPInstanceUID),
                      "reason": "像素数据中检测到可能的 PHI 文字"}]
        return []


class AnonymizationAuditService:
    """匿名化审计服务：原始→匿名化映射的安全存储与访问控制"""

    MAPPING_ENCRYPTION_KEY = "vault:anonymization/mapping-key"

    def store_mapping(self, original_study_uid, anonymized_study_uid,
                      pseudonym_map, uid_mapping):
        """安全存储原始→匿名化映射"""
        # 加密映射数据（使用 Vault 管理的密钥）
        mapping_data = {
            "original_study_uid": original_study_uid,
            "anonymized_study_uid": anonymized_study_uid,
            "pseudonym_map": pseudonym_map,
            "uid_mapping": uid_mapping,
            "created_at": now().isoformat(),
        }

        encrypted = self.crypto.encrypt(
            json.dumps(mapping_data).encode(),
            key_id=self.MAPPING_ENCRYPTION_KEY,
        )

        self.db.insert("anonymization_mappings", {
            "mapping_id": generate_id(),
            "original_study_uid": original_study_uid,
            "anonymized_study_uid": anonymized_study_uid,
            "encrypted_mapping": encrypted,
            "created_at": now(),
        })

    def resolve_identity(self, anonymized_study_uid, requester_id, purpose):
        """反向查询：从匿名化 ID 查找原始身份（严格访问控制）"""
        # 访问控制：仅授权人员可查询
        if not self.access_control.check_permission(
                requester_id, "anonymization:resolve_identity", purpose):
            self.audit_log.record({
                "action": "identity_resolution_denied",
                "requester": requester_id,
                "anonymized_study_uid": anonymized_study_uid,
                "purpose": purpose,
                "timestamp": now(),
            })
            return {"error": "无权访问身份映射"}

        # 记录访问审计
        self.audit_log.record({
            "action": "identity_resolution",
            "requester": requester_id,
            "anonymized_study_uid": anonymized_study_uid,
            "purpose": purpose,
            "timestamp": now(),
        })

        # 解密映射数据
        record = self.db.query_one(
            "SELECT * FROM anonymization_mappings "
            "WHERE anonymized_study_uid = %s", anonymized_study_uid)

        if not record:
            return {"error": "映射记录不存在"}

        decrypted = self.crypto.decrypt(
            record["encrypted_mapping"],
            key_id=self.MAPPING_ENCRYPTION_KEY,
        )
        mapping = json.loads(decrypted)

        return {"original_study_uid": mapping["original_study_uid"],
                "pseudonym_map": mapping["pseudonym_map"]}

    def batch_anonymize_for_research(self, study_uids, research_project_id):
        """批量匿名化研究数据集"""
        batch_id = generate_id()
        results = []

        for study_uid in study_uids:
            try:
                result = self.anonymization_pipeline.anonymize_study(
                    study_uid, purpose="research")
                results.append({
                    "original_uid": study_uid,
                    "anonymized_uid": result["anonymized_study_uid"],
                    "status": "success",
                })
            except Exception as e:
                results.append({
                    "original_uid": study_uid,
                    "status": "failed",
                    "error": str(e),
                })

        # 记录批量匿名化操作
        self.db.insert("batch_anonymizations", {
            "batch_id": batch_id,
            "research_project_id": research_project_id,
            "total_studies": len(study_uids),
            "successful": sum(1 for r in results if r["status"] == "success"),
            "failed": sum(1 for r in results if r["status"] == "failed"),
            "results": json.dumps(results),
            "performed_at": now(),
        })

        return {"batch_id": batch_id, "results": results}
```

## PACS 存储分层与灾备深度实现

```python
class PACSStorageTierManager:
    """PACS 存储分层管理：SSD/HDD/云分层 + 自动迁移"""

    TIER_CONFIG = {
        "hot": {
            "medium": "SSD NVMe",
            "retention_days": 30,
            "cost_per_gb_month": 0.10,
            "access_latency_ms": 10,
            "description": "最近 30 天检查，阅片低延迟",
        },
        "warm": {
            "medium": "HDD SAS",
            "retention_days": 335,  # 30+335=365 天
            "cost_per_gb_month": 0.03,
            "access_latency_ms": 100,
            "description": "30-365 天检查，按需加载到缓存",
        },
        "cold": {
            "medium": "S3 Glacier Deep Archive",
            "retention_days": None,  # 法定保存 15-30 年
            "cost_per_gb_month": 0.002,
            "access_latency_hours": 12,
            "description": ">1 年检查，需提前申请恢复",
        },
    }

    def migrate_studies(self, dry_run=False):
        """执行存储分层迁移"""
        migration_report = {"hot_to_warm": [], "warm_to_cold": []}

        # Hot → Warm: 超过 30 天的检查
        hot_expired = self.db.query(
            "SELECT study_instance_uid, SUM(file_size_bytes) as total_bytes "
            "FROM studies WHERE study_date < NOW() - INTERVAL 30 DAY "
            "AND storage_tier = 'hot' GROUP BY study_instance_uid")

        for study in hot_expired:
            # 1. DICOM 存储承诺验证（确保数据已安全存储到目标层）
            if not self._verify_storage_commitment(study["study_instance_uid"], "warm"):
                self.alert(f"存储承诺验证失败: {study['study_instance_uid']}，跳过迁移")
                continue

            if dry_run:
                migration_report["hot_to_warm"].append({
                    "study_uid": study["study_instance_uid"],
                    "size_gb": round(study["total_bytes"] / 1073741824, 2),
                })
                continue

            # 2. 复制到 HDD 存储
            self._copy_to_warm_storage(study["study_instance_uid"])

            # 3. 验证复制完整性
            if self._verify_copy_integrity(study["study_instance_uid"], "warm"):
                # 4. 更新存储层级
                self.db.update("studies",
                    {"storage_tier": "warm", "migrated_at": now()},
                    {"study_instance_uid": study["study_instance_uid"]})
                # 5. 释放 Hot 存储
                self._delete_from_hot_storage(study["study_instance_uid"])

        # Warm → Cold: 超过 365 天的检查
        warm_expired = self.db.query(
            "SELECT study_instance_uid, SUM(file_size_bytes) as total_bytes "
            "FROM studies WHERE study_date < NOW() - INTERVAL 365 DAY "
            "AND storage_tier = 'warm' GROUP BY study_instance_uid")

        for study in warm_expired:
            if not self._verify_storage_commitment(study["study_instance_uid"], "cold"):
                continue

            if dry_run:
                migration_report["warm_to_cold"].append({
                    "study_uid": study["study_instance_uid"],
                    "size_gb": round(study["total_bytes"] / 1073741824, 2),
                })
                continue

            # 上传到 S3 Glacier
            self.s3.upload_study(
                study["study_instance_uid"],
                storage_class="DEEP_ARCHIVE",
                lifecycle_rule="retain_until_legal_expiry",
            )

            if self._verify_copy_integrity(study["study_instance_uid"], "cold"):
                self.db.update("studies",
                    {"storage_tier": "cold", "migrated_at": now()},
                    {"study_instance_uid": study["study_instance_uid"]})
                self._delete_from_warm_storage(study["study_instance_uid"])

        return migration_report

    def _verify_storage_commitment(self, study_uid, target_tier):
        """DICOM 存储承诺验证：确保数据安全存储后才删除源"""
        source_files = self.db.query(
            "SELECT sop_instance_uid, file_checksum_sha256 "
            "FROM study_files WHERE study_instance_uid = %s", study_uid)

        for f in source_files:
            if target_tier == "warm":
                exists = self.hdd_storage.check_exists(f["sop_instance_uid"])
            elif target_tier == "cold":
                exists = self.s3.check_object_exists(
                    key=f"{study_uid}/{f['sop_instance_uid']}.dcm")
            else:
                exists = False

            if not exists:
                return False

        return True

    def check_capacity_and_alert(self):
        """存储容量监控与告警"""
        tiers_status = {}
        for tier_name, config in self.TIER_CONFIG.items():
            total_gb = self._get_tier_capacity_gb(tier_name)
            used_gb = self._get_tier_used_gb(tier_name)
            usage_pct = (used_gb / max(total_gb, 1)) * 100

            tiers_status[tier_name] = {
                "total_gb": total_gb,
                "used_gb": used_gb,
                "usage_pct": round(usage_pct, 1),
                "medium": config["medium"],
            }

            # 80% 告警
            if usage_pct >= 80:
                self.alerting.send(
                    severity="warning",
                    title=f"PACS {tier_name} 层存储使用率 {usage_pct:.0f}%",
                    message=f"建议执行数据迁移或扩容。{config['description']}",
                    tags={"tier": tier_name, "usage_pct": usage_pct},
                )

            # 90% 紧急告警
            if usage_pct >= 90:
                self.alerting.send(
                    severity="critical",
                    title=f"PACS {tier_name} 层存储即将满 {usage_pct:.0f}%",
                    message=f"存储不足可能影响影像存储和阅片，立即处理",
                    tags={"tier": tier_name, "usage_pct": usage_pct},
                )

        return tiers_status


class PACSDisasterRecoveryManager:
    """PACS 灾备管理：主站→DR 站点复制 + 故障切换"""

    DR_CONFIG = {
        "rpo_hours": 1,        # 恢复点目标：最多丢失 1 小时数据
        "rto_hours": 4,        # 恢复时间目标：4 小时内恢复服务
        "sync_interval_minutes": 15,  # 同步间隔
        "verification_interval_hours": 6,  # 一致性校验间隔
    }

    def sync_to_dr(self):
        """同步数据到 DR 站点"""
        # 获取上次同步后的新增/修改检查
        last_sync = self.db.query_one(
            "SELECT MAX(synced_at) as last FROM dr_sync_log")["last"]

        new_studies = self.db.query(
            "SELECT study_instance_uid FROM studies "
            "WHERE created_at > %s AND storage_tier != 'deleted'",
            last_sync or (now() - timedelta(hours=1)))

        synced = 0
        failed = 0

        for study in new_studies:
            try:
                # 复制检查到 DR 站点
                self.dr_pacs.replicate_study(study["study_instance_uid"])
                synced += 1
            except Exception as e:
                self.logger.error(f"DR 同步失败: {study['study_instance_uid']}: {e}")
                failed += 1

        # 记录同步日志
        self.db.insert("dr_sync_log", {
            "sync_id": generate_id(),
            "studies_synced": synced,
            "studies_failed": failed,
            "synced_at": now(),
        })

        return {"synced": synced, "failed": failed}

    def activate_dr_site(self, reason):
        """激活 DR 站点（主站故障时）"""
        self.logger.critical(f"激活 DR 站点，原因: {reason}")

        # 1. 通知所有相关方
        self.alerting.send(
            severity="critical",
            title="PACS 主站故障，正在激活 DR 站点",
            message=f"原因: {reason}，RTO 目标: {self.DR_CONFIG['rto_hours']} 小时",
            channels=["pager_duty", "slack_ops", "email_all_radiologists"],
        )

        # 2. DNS 切换到 DR 站点
        self.dns_service.update_record(
            "pacs.hospital.internal",
            self.dr_pacs.endpoint,
            ttl=60,
        )

        # 3. 验证 DR 站点就绪
        health = self.dr_pacs.health_check()
        if not health["healthy"]:
            self.alerting.send(
                severity="critical",
                title="DR 站点健康检查失败",
                message=f"DR 站点不健康: {health['issues']}",
            )
            return {"status": "dr_activation_failed", "health": health}

        # 4. 恢复近 30 天检查到 Hot 缓存（DR 站点）
        self.dr_pacs.warmup_recent_studies(days=30)

        # 5. 记录灾备切换
        self.db.insert("dr_activations", {
            "activation_id": generate_id(),
            "reason": reason,
            "activated_at": now(),
            "rto_target": self.DR_CONFIG["rto_hours"],
            "status": "active",
        })

        return {"status": "dr_activated", "dr_endpoint": self.dr_pacs.endpoint}

    def failback_to_primary(self):
        """主站恢复后回切"""
        # 1. 同步 DR 站点增量数据回主站
        dr_changes = self.dr_pacs.get_changes_since(self._get_primary_failure_time())

        for change in dr_changes:
            self.primary_pacs.apply_change(change)

        # 2. 验证数据一致性
        consistency = self._verify_primary_dr_consistency()
        if not consistency["consistent"]:
            self.alerting.send(
                severity="warning",
                title="主站回切数据不一致",
                message=f"差异: {consistency['differences']}",
            )
            return {"status": "failback_delayed", "consistency": consistency}

        # 3. DNS 切回主站
        self.dns_service.update_record(
            "pacs.hospital.internal",
            self.primary_pacs.endpoint,
            ttl=60,
        )

        # 4. 记录回切
        self.db.update("dr_activations",
            {"status": "failed_back", "failed_back_at": now()},
            {"status": "active"})

        return {"status": "failback_completed"}

    def _verify_primary_dr_consistency(self):
        """验证主站与 DR 站点数据一致性"""
        primary_count = self.db.count("studies", storage_tier__ne="deleted")
        dr_count = self.dr_pacs.count_studies()

        if primary_count != dr_count:
            # 找出差异
            primary_uids = set(r["study_instance_uid"] for r in
                self.db.query("SELECT study_instance_uid FROM studies WHERE storage_tier != 'deleted'"))
            dr_uids = set(self.dr_pacs.list_study_uids())

            missing_in_dr = primary_uids - dr_uids
            missing_in_primary = dr_uids - primary_uids

            return {
                "consistent": False,
                "primary_count": primary_count,
                "dr_count": dr_count,
                "differences": {
                    "missing_in_dr": len(missing_in_dr),
                    "missing_in_primary": len(missing_in_primary),
                },
            }

        return {"consistent": True, "count": primary_count}
```

### 场景：匿名化未移除图像中烧录的 PHI

```
触发：DICOM 图像四角烧录了患者姓名和出生日期 → 标准匿名化只处理了 Tag
      → 像素数据中仍有 PHI → 分发给研究机构后 PHI 泄露
检测：
  1. 匿名化后 OCR 扫描发现残留文字 → PHI 泄露风险
  2. 抽检发现超声图像普遍有烧录文字 → 系统性问题
  3. 研究机构反馈收到含患者信息的图像 → 已泄露
处理：
  1. 【紧急】已分发的匿名化数据立即召回
  2. 【修复】增强像素匿名化：OCR 检测 → 精确定位文字区域 → 用周围像素填充
  3. 【验证】对所有匿名化后的图像增加 OCR 二次验证步骤
  4. 【补丁】重新处理所有已匿名化但未经像素验证的检查
  5. 【审计】记录 PHI 泄露事件，通知合规部门
  6. 【改进】对已知有烧录文字的设备类型（超声、X光）自动启用像素匿名化
预防：匿名化后强制 OCR 验证 + 设备类型→匿名化策略映射
      + 人工抽检（5%）+ BurnedInAnnotation 标签检查 + 泄露应急 SOP
```

### 场景：PACS 主存储故障需激活 DR 站点

```
触发：主 PACS 存储集群硬件故障（SSD 阵列控制器损坏）→ 近 30 天检查不可访问
      → 医生无法调阅当天影像 → 直接影响临床诊断
检测：
  1. PACS 健康检查连续 3 次失败 → 自动告警
  2. 阅片请求超时率 > 50% → 严重服务降级
  3. 存储控制器 I/O 错误日志 → 硬件故障
处理：
  1. 【评估】5 分钟内确认故障范围：
     - 近 30 天（Hot 层）检查不可访问
     - 30-365 天（Warm 层）检查可能受影响
     - >1 年（Cold 层）在 S3 不受影响
  2. 【激活 DR】启动灾备站点：
     - DNS 切换到 DR 站点（RTO 目标 4 小时）
     - DR 站点预热最近 30 天检查到缓存
     - 通知所有放射科医生使用 DR 站点
  3. 【恢复】主站修复后：
     - 同步 DR 增量数据回主站
     - 验证数据一致性
     - DNS 切回主站
  4. 【复盘】故障后 48 小时内完成复盘：
     - SSD 阵列单点故障 → 增加冗余
     - RPO 验证：确认数据丢失在 1 小时内
     - 改进：自动故障切换（减少人工干预时间）
预防：DR 站点 + RPO 1 小时同步 + 自动健康检查
      + 存储冗余 + 定期灾备演练（每季度）+ 自动故障切换

## 医学影像标注与审核完整实现

```python
class MedicalAnnotationReviewService:
    """影像标注审核：标注提交 → 专家审核 → 标注修正 → 版本管理"""

    ANNOTATION_TYPES = {
        "segmentation": "分割标注（区域）",
        "bounding_box": "矩形标注",
        "point": "点标注",
        "line": "线标注",
        "classification": "分类标注",
    }

    REVIEW_STATUSES = {
        "submitted": "已提交",
        "under_review": "审核中",
        "approved": "已批准",
        "rejected": "已驳回",
        "revision_needed": "需修正",
    }

    def submit_annotation(self, annotator_id, study_id, annotation_data):
        """提交标注"""
        # 1. 验证标注数据完整性
        validation = self._validate_annotation(annotation_data)
        if not validation["valid"]:
            return {"status": "validation_failed", "issues": validation["issues"]}

        # 2. 检查同一影像是否有已批准的标注
        existing_approved = self.db.query_one(
            "SELECT * FROM annotations "
            "WHERE study_id = %s AND annotation_type = %s "
            "AND status = 'approved'",
            study_id, annotation_data["annotation_type"])

        if existing_approved and annotation_data.get("overwrite") != True:
            return {"status": "already_approved", "existing_id": existing_approved["annotation_id"]}

        # 3. 创建标注
        annotation_id = str(uuid4())
        self.db.insert("annotations", {
            "annotation_id": annotation_id,
            "study_id": study_id,
            "annotator_id": annotator_id,
            "annotation_type": annotation_data["annotation_type"],
            "label": annotation_data.get("label"),
            "data": json.dumps(annotation_data["data"]),
            "confidence": annotation_data.get("confidence", 0.8),
            "status": "submitted",
            "version": 1,
            "created_at": now()
        })

        # 4. 分配审核专家（根据标注类型）
        reviewer = self._assign_reviewer(study_id, annotation_data["annotation_type"])

        self.db.update("annotations",
            {"reviewer_id": reviewer["id"], "status": "under_review"},
            {"annotation_id": annotation_id})

        self.db.insert("annotation_review_assignments", {
            "assignment_id": str(uuid4()),
            "annotation_id": annotation_id,
            "reviewer_id": reviewer["id"],
            "assigned_at": now()
        })

        return {"annotation_id": annotation_id, "status": "submitted",
                "reviewer_assigned": reviewer["id"]}

    def review_annotation(self, reviewer_id, annotation_id, decision, review_notes=None):
        """审核标注"""
        annotation = self.db.get_annotation(annotation_id)

        if annotation["reviewer_id"] != reviewer_id:
            raise PermissionDeniedError("非指定审核专家")

        if annotation["status"] != "under_review":
            return {"status": "cannot_review", "current_status": annotation["status"]}

        # 1. 更新审核状态
        self.db.update("annotations",
            {"status": decision, "review_notes": review_notes,
             "reviewed_at": now()},
            {"annotation_id": annotation_id})

        # 2. 记录审核日志
        self.db.insert("annotation_review_log", {
            "log_id": str(uuid4()),
            "annotation_id": annotation_id,
            "reviewer_id": reviewer_id,
            "decision": decision,
            "notes": review_notes,
            "reviewed_at": now()
        })

        # 3. 如果需修正 → 通知标注员
        if decision == "revision_needed":
            self.notification.send(annotation["annotator_id"],
                f"标注需修正: {review_notes}")

        # 4. 如果批准 → 标注生效
        if decision == "approved":
            # 标注数据写入临床数据
            self._apply_approved_annotation(annotation)

            # 计算标注一致性（如果有多人标注同一影像）
            self._calculate_inter_rater_agreement(annotation["study_id"])

        # 5. 更新审核员统计
        self._update_reviewer_stats(reviewer_id, decision)

        return {"annotation_id": annotation_id, "decision": decision}

    def revise_annotation(self, annotator_id, annotation_id, revised_data):
        """修正标注"""
        annotation = self.db.get_annotation(annotation_id)

        if annotation["annotator_id"] != annotator_id:
            raise PermissionDeniedError("只有原标注员可以修正")

        if annotation["status"] != "revision_needed":
            return {"status": "cannot_revise", "current_status": annotation["status"]}

        # 1. 创建新版本
        new_version = annotation["version"] + 1

        # 2. 保存修正后的标注
        self.db.update("annotations",
            {"data": json.dumps(revised_data["data"]),
             "confidence": revised_data.get("confidence", annotation["confidence"]),
             "version": new_version,
             "status": "submitted",
             "revision_notes": revised_data.get("revision_notes"),
             "revised_at": now()},
            {"annotation_id": annotation_id})

        # 3. 重新分配审核
        reviewer = self._assign_reviewer(annotation["study_id"], annotation["annotation_type"])
        self.db.update("annotations",
            {"reviewer_id": reviewer["id"], "status": "under_review"},
            {"annotation_id": annotation_id})

        return {"annotation_id": annotation_id, "version": new_version, "status": "submitted"}

    def _validate_annotation(self, annotation_data):
        """验证标注数据"""
        issues = []

        if annotation_data.get("annotation_type") not in self.ANNOTATION_TYPES:
            issues.append(f"无效标注类型: {annotation_data.get('annotation_type')}")

        data = annotation_data.get("data")
        if not data:
            issues.append("标注数据为空")

        if annotation_data.get("annotation_type") == "segmentation":
            if not isinstance(data, list) or len(data) < 3:
                issues.append("分割标注至少需要 3 个点")

        if annotation_data.get("annotation_type") == "bounding_box":
            if not isinstance(data, dict) or "x" not in data or "y" not in data:
                issues.append("矩形标注缺少坐标")
            if data.get("width", 0) <= 0 or data.get("height", 0) <= 0:
                issues.append("矩形标注宽高必须大于 0")

        confidence = annotation_data.get("confidence", 0)
        if confidence < 0 or confidence > 1:
            issues.append("置信度必须在 0-1 之间")

        return {"valid": len(issues) == 0, "issues": issues}

    def _calculate_inter_rater_agreement(self, study_id):
        """计算标注者间一致性（Cohen's Kappa）"""
        annotations = self.db.query(
            "SELECT * FROM annotations "
            "WHERE study_id = %s AND status = 'approved' "
            "AND annotation_type = 'classification'",
            study_id)

        if len(annotations) < 2:
            return {"kappa": None, "reason": "标注数量不足"}

        # 计算一致比例
        labels = [a["label"] for a in annotations]
        label_counts = {}
        for label in labels:
            label_counts[label] = label_counts.get(label, 0) + 1

        # 观察一致率
        agree_count = 0
        for i in range(len(annotations)):
            for j in range(i + 1, len(annotations)):
                if annotations[i]["label"] == annotations[j]["label"]:
                    agree_count += 1

        pairs = len(annotations) * (len(annotations) - 1) / 2
        po = agree_count / max(pairs, 1)

        # 期望一致率
        pe = sum((c / len(labels)) ** 2 for c in label_counts.values())

        kappa = (po - pe) / max(1 - pe, 0.001)

        self.db.insert("annotation_agreement", {
            "study_id": study_id,
            "annotator_count": len(annotations),
            "kappa": round(kappa, 4),
            "observed_agreement": round(po, 4),
            "expected_agreement": round(pe, 4),
            "calculated_at": now()
        })

        return {"kappa": round(kappa, 4), "interpretation": self._interpret_kappa(kappa)}

    def _interpret_kappa(self, kappa):
        """解释 Kappa 值"""
        if kappa < 0: return "不一致"
        elif kappa < 0.2: return "轻微一致"
        elif kappa < 0.4: return "一般一致"
        elif kappa < 0.6: return "中度一致"
        elif kappa < 0.8: return "高度一致"
        else: return "几乎完全一致"
```

## 异常场景补充

### 场景：标注审核专家意见分歧

```
触发：两个审核专家对同一标注给出不同判断 → 一个批准一个驳回 → 无法确定最终状态
检测：
  1. 同一标注有多个审核结果且不一致 → 分歧
  2. 标注状态反复变化 → 分歧
处理：
  1. 第三位资深专家终审
  2. 多人审核取多数意见
  3. 争议标注标记为"待讨论"
预防：三人终审 + 多数意见 + 争议标记
```

### 场景：标注数据被误删

```
触发：标注员误删已批准的标注 → 临床数据丢失 → 影响诊断
检测：
  1. 已批准标注被删除 → 数据丢失
  2. 标注数量突然减少 → 可能误删
处理：
  1. 标注删除需审批（已批准标注不可直接删除）
  2. 软删除（标记删除但保留数据）
  3. 标注版本历史可恢复
预防：删除审批 + 软删除 + 版本恢复
```

## 医学影像 DICOM 数据管理完整实现

```python
class DICOMDataService:
    """DICOM 数据管理：解析 → 存储 → 脱敏 → 传输"""

    DICOM_TAGS = {
        "patient_name": (0x0010, 0x0010),
        "patient_id": (0x0010, 0x0020),
        "study_date": (0x0008, 0x0020),
        "modality": (0x0008, 0x0060),
        "study_description": (0x0008, 0x1030),
        "series_number": (0x0020, 0x0011),
        "instance_number": (0x0020, 0x0013),
        "pixel_data": (0x7FE0, 0x0010),
    }

    PHI_TAGS = [  # 需要脱敏的标签
        (0x0010, 0x0010),  # 患者姓名
        (0x0010, 0x0020),  # 患者 ID
        (0x0010, 0x0030),  # 出生日期
        (0x0010, 0x1000),  # 其他姓名
        (0x0010, 0x1001),  # 其他姓名
        (0x0008, 0x0050),  # 检查号
        (0x0010, 0x1002),  # 其他患者 ID
    ]

    def ingest_dicom(self, file_path, source_system):
        """导入 DICOM 文件"""
        # 1. 解析 DICOM 文件
        try:
            import pydicom
            ds = pydicom.dcmread(file_path)
        except Exception as e:
            return {"status": "parse_error", "error": str(e)}

        # 2. 提取关键元数据
        metadata = {
            "patient_id": str(ds.get("PatientID", "")),
            "study_instance_uid": str(ds.get("StudyInstanceUID", "")),
            "series_instance_uid": str(ds.get("SeriesInstanceUID", "")),
            "sop_instance_uid": str(ds.get("SOPInstanceUID", "")),
            "modality": str(ds.get("Modality", "")),
            "study_date": str(ds.get("StudyDate", "")),
            "study_description": str(ds.get("StudyDescription", "")),
            "series_number": int(ds.get("SeriesNumber", 0)),
            "instance_number": int(ds.get("InstanceNumber", 0)),
            "rows": int(ds.get("Rows", 0)),
            "columns": int(ds.get("Columns", 0)),
            "bits_allocated": int(ds.get("BitsAllocated", 16)),
            "pixel_spacing": str(ds.get("PixelSpacing", "")),
            "source_system": source_system,
        }

        # 3. 去重检查（SOP Instance UID 唯一）
        existing = self.db.query_one(
            "SELECT * FROM dicom_instances WHERE sop_instance_uid = %s",
            metadata["sop_instance_uid"])

        if existing:
            return {"status": "duplicate", "sop_instance_uid": metadata["sop_instance_uid"]}

        # 4. 存储像素数据到对象存储
        pixel_data = ds.PixelData
        storage_key = f"dicom/{metadata['study_instance_uid']}/{metadata['series_instance_uid']}/{metadata['sop_instance_uid']}.dcm"
        self.object_storage.upload(storage_key, pixel_data)

        # 5. 存储元数据
        instance_id = str(uuid4())
        self.db.insert("dicom_instances", {
            "instance_id": instance_id,
            **metadata,
            "storage_key": storage_key,
            "file_size_bytes": len(pixel_data),
            "status": "available",
            "ingested_at": now()
        })

        return {"instance_id": instance_id, "status": "ingested",
                "sop_instance_uid": metadata["sop_instance_uid"]}

    def anonymize_dicom(self, instance_id, method="replace"):
        """脱敏 DICOM 数据"""
        instance = self.db.get_dicom_instance(instance_id)

        # 1. 下载原始文件
        dicom_data = self.object_storage.download(instance["storage_key"])

        # 2. 解析并脱敏
        import pydicom
        import io
        ds = pydicom.dcmread(io.BytesIO(dicom_data))

        # 3. 替换或删除 PHI 标签
        anonymized_tags = {}
        for tag in self.PHI_TAGS:
            if tag in ds:
                original_value = str(ds[tag].value)

                if method == "replace":
                    # 替换为匿名值
                    ds[tag].value = self._generate_anonymous_value(tag, original_value)
                    anonymized_tags[f"{tag[0]:04X}{tag[1]:04X}"] = {
                        "original_hash": hashlib.sha256(original_value.encode()).hexdigest()[:8],
                        "anonymized": True
                    }
                elif method == "remove":
                    del ds[tag]
                    anonymized_tags[f"{tag[0]:04X}{tag[1]:04X}"] = {"removed": True}

        # 4. 保存脱敏后的文件
        output = io.BytesIO()
        ds.save_as(output)
        anonymized_data = output.getvalue()

        anon_storage_key = instance["storage_key"].replace("dicom/", "dicom_anon/")
        self.object_storage.upload(anon_storage_key, anonymized_data)

        # 5. 记录脱敏映射（用于可逆脱敏场景）
        self.db.insert("dicom_anonymization_log", {
            "log_id": str(uuid4()),
            "instance_id": instance_id,
            "method": method,
            "anonymized_tags": json.dumps(anonymized_tags),
            "anonymized_storage_key": anon_storage_key,
            "anonymized_at": now()
        })

        return {"instance_id": instance_id, "method": method,
                "anonymized_tags": len(anonymized_tags),
                "storage_key": anon_storage_key}

    def _generate_anonymous_value(self, tag, original_value):
        """生成匿名替代值"""
        group, element = tag

        if group == 0x0010 and element == 0x0010:  # 患者姓名
            return f"ANON{hashlib.sha256(original_value.encode()).hexdigest()[:6]}"
        elif group == 0x0010 and element == 0x0020:  # 患者 ID
            return f"PID{hashlib.sha256(original_value.encode()).hexdigest()[:8]}"
        elif group == 0x0010 and element == 0x0030:  # 出生日期
            return "19000101"
        else:
            return "ANONYMOUS"

    def retrieve_study(self, study_instance_uid, include_pixel_data=False):
        """检索完整检查"""
        instances = self.db.query(
            "SELECT * FROM dicom_instances "
            "WHERE study_instance_uid = %s AND status = 'available' "
            "ORDER BY series_number, instance_number",
            study_instance_uid)

        study = {
            "study_instance_uid": study_instance_uid,
            "instance_count": len(instances),
            "series": {}
        }

        for inst in instances:
            series_uid = inst["series_instance_uid"]
            if series_uid not in study["series"]:
                study["series"][series_uid] = {
                    "series_number": inst["series_number"],
                    "modality": inst["modality"],
                    "instances": []
                }

            instance_data = {
                "instance_id": inst["instance_id"],
                "sop_instance_uid": inst["sop_instance_uid"],
                "instance_number": inst["instance_number"],
                "rows": inst["rows"],
                "columns": inst["columns"],
            }

            if include_pixel_data:
                pixel_data = self.object_storage.download(inst["storage_key"])
                instance_data["pixel_data_available"] = True
                instance_data["pixel_data_size"] = len(pixel_data)

            study["series"][series_uid]["instances"].append(instance_data)

        return study
```

## 异常场景补充

### 场景：DICOM 脱敏不完整导致隐私泄露

```
触发：脱敏只处理了标准 PHI 标签 → 但影像中嵌入的患者信息（如检查单上的姓名）未被处理 → 隐私泄露
检测：
  1. OCR 扫描影像内容发现患者信息 → 脱敏不完整
  2. 审计发现脱敏后仍有可识别信息 → 泄露风险
处理：
  1. 对影像内容进行 OCR 检测
  2. 检测到的文字区域进行像素模糊
  3. 脱敏后二次验证
预防：OCR 检测 + 像素模糊 + 二次验证
```

### 场景：大型 DICOM 文件导入超时

```
触发：CT 检查 2000 张切片 → 单文件 2GB → 导入耗时 10 分钟 → 超时失败
检测：
  1. 单次导入超过 5 分钟 → 大文件
  2. 导入任务超时 → 需要异步处理
处理：
  1. 大文件分片上传
  2. 异步导入（后台任务）
  3. 进度追踪
预防：分片上传 + 异步导入 + 进度追踪
```

## 医学影像分布式存储与传输完整实现

```python
import math
import time
import hashlib
import bisect
from datetime import datetime, timedelta
from collections import defaultdict
from typing import List, Dict, Tuple, Optional, Set

class MedicalImageStorageService:
    """医学影像分布式存储与传输服务，提供DICOM系列存储、检索、备份复制和节点故障处理"""

    ERASURE_DATA_SHARDS = 4
    ERASURE_PARITY_SHARDS = 2
    TOTAL_SHARDS = 6
    HEALTH_CHECK_TIMEOUT_SECONDS = 10
    REPLICATION_THRESHOLD_HOURS = 1
    DICOM_PIXEL_DATA_TAG = "7FE00010"
    DICOM_SERIES_NUMBER_TAG = "00200011"
    DICOM_INSTANCE_NUMBER_TAG = "00200013"
    DICOM_SOP_CLASS_UID_TAG = "00080016"
    DICOM_TRANSFER_SYNTAX_TAG = "00020010"

    VALID_SOP_CLASS_UIDS = {
        "1.2.840.10008.5.1.4.1.1.2": "CT Image Storage",
        "1.2.840.10008.5.1.4.1.1.2.1": "Enhanced CT Image Storage",
        "1.2.840.10008.5.1.4.1.1.4": "MR Image Storage",
        "1.2.840.10008.5.1.4.1.1.4.1": "Enhanced MR Image Storage",
        "1.2.840.10008.5.1.4.1.1.1": "CR Image Storage",
        "1.2.840.10008.5.1.4.1.1.1.1": "Enhanced CR Image Storage",
        "1.2.840.10008.5.1.4.1.1.7": "Secondary Capture Image Storage",
        "1.2.840.10008.5.1.4.1.1.77.1.1": "VL Whole Slide Microscopy Image Storage",
        "1.2.840.10008.5.1.4.1.1.128": "PET Image Storage",
        "1.2.840.10008.5.1.4.1.1.130": "Enhanced PET Image Storage"
    }

    VALID_TRANSFER_SYNTAXES = {
        "1.2.840.10008.1.2": "Implicit VR Little Endian",
        "1.2.840.10008.1.2.1": "Explicit VR Little Endian",
        "1.2.840.10008.1.2.2": "Explicit VR Big Endian",
        "1.2.840.10008.1.2.4.50": "JPEG Baseline",
        "1.2.840.10008.1.2.4.70": "JPEG Lossless",
        "1.2.840.10008.1.2.4.90": "JPEG 2000 Lossless",
        "1.2.840.10008.1.2.4.91": "JPEG 2000"
    }

    WINDOWING_PRESETS = {
        "CT": [
            {"name": "Lung", "center": -600, "width": 1500},
            {"name": "Bone", "center": 400, "width": 1800},
            {"name": "Soft Tissue", "center": 40, "width": 400},
            {"name": "Brain", "center": 40, "width": 80},
            {"name": "Liver", "center": 60, "width": 150}
        ],
        "MR": [
            {"name": "T1", "center": 500, "width": 1000},
            {"name": "T2", "center": 1000, "width": 2000},
            {"name": "FLAIR", "center": 900, "width": 1800}
        ],
        "CR": [
            {"name": "Default", "center": 2048, "width": 4096}
        ],
        "PET": [
            {"name": "SUV", "center": 5, "width": 20},
            {"name": "Hot Body", "center": 10, "width": 30}
        ]
    }

    def __init__(self, storage_nodes: List[Dict] = None):
        self.storage_nodes: Dict[str, Dict] = {}
        self.consistent_hash_ring: List[Tuple[int, str]] = []
        self.object_locations: Dict[str, List[str]] = {}
        self.study_manifests: Dict[str, Dict] = {}
        self.object_checksums: Dict[str, str] = {}
        self.node_health: Dict[str, Dict] = {}
        self.replication_tasks: List[Dict] = []
        self.admin_notifications: List[Dict] = []
        self.vnode_count = 150
        if storage_nodes:
            for node in storage_nodes:
                self.add_storage_node(node)

    def add_storage_node(self, node: Dict) -> None:
        """添加存储节点到一致性哈希环"""
        node_id = node["node_id"]
        self.storage_nodes[node_id] = node
        self.node_health[node_id] = {
            "status": "online",
            "last_check": time.time(),
            "failure_count": 0
        }
        for i in range(self.vnode_count):
            vnode_key = f"{node_id}:vnode:{i}"
            hash_val = self._consistent_hash(vnode_key)
            bisect.insort(self.consistent_hash_ring, (hash_val, node_id))

    def _consistent_hash(self, key: str) -> int:
        """一致性哈希函数"""
        digest = hashlib.md5(key.encode()).hexdigest()
        return int(digest[:8], 16)

    def _get_nodes_for_key(self, key: str, count: int = 3) -> List[str]:
        """使用一致性哈希获取key对应的存储节点"""
        if not self.consistent_hash_ring:
            return []
        hash_val = self._consistent_hash(key)
        idx = bisect.bisect_left(self.consistent_hash_ring, (hash_val, ""))
        selected = []
        seen = set()
        ring_len = len(self.consistent_hash_ring)
        for i in range(ring_len):
            node_id = self.consistent_hash_ring[(idx + i) % ring_len][1]
            if node_id not in seen:
                node_status = self.node_health.get(node_id, {}).get("status", "offline")
                if node_status == "online":
                    selected.append(node_id)
                    seen.add(node_id)
                if len(selected) >= count:
                    break
        return selected

    def _validate_dicom_conformance(self, instance: Dict) -> Tuple[bool, List[str]]:
        """验证DICOM一致性：SOP Class UID和Transfer Syntax"""
        errors = []
        sop_class_uid = instance.get(self.DICOM_SOP_CLASS_UID_TAG, "")
        transfer_syntax = instance.get(self.DICOM_TRANSFER_SYNTAX_TAG, "")
        if not sop_class_uid:
            errors.append("缺少SOP Class UID (0008,0016)")
        elif sop_class_uid not in self.VALID_SOP_CLASS_UIDS:
            errors.append(f"不支持的SOP Class UID: {sop_class_uid}")
        if not transfer_syntax:
            errors.append("缺少Transfer Syntax (0002,0010)")
        elif transfer_syntax not in self.VALID_TRANSFER_SYNTAXES:
            errors.append(f"不支持的Transfer Syntax: {transfer_syntax}")
        return (len(errors) == 0, errors)

    def _compute_checksum(self, data: bytes) -> str:
        """计算数据的SHA256校验和"""
        return hashlib.sha256(data).hexdigest()

    def _erasure_encode(self, data: bytes) -> List[Optional[bytes]]:
        """简化的纠删码编码：将数据分为DATA_SHARDS份，生成PARITY_SHARDS份校验"""
        data_size = len(data)
        shard_size = (data_size + self.ERASURE_DATA_SHARDS - 1) // self.ERASURE_DATA_SHARDS
        padded_data = data + b'\x00' * (shard_size * self.ERASURE_DATA_SHARDS - data_size)
        data_shards = []
        for i in range(self.ERASURE_DATA_SHARDS):
            start = i * shard_size
            end = start + shard_size
            data_shards.append(padded_data[start:end])
        parity_shards = []
        for p in range(self.ERASURE_PARITY_SHARDS):
            parity = bytearray(shard_size)
            for i in range(shard_size):
                val = 0
                for s in range(self.ERASURE_DATA_SHARDS):
                    val ^= data_shards[s][i]
                if p == 1:
                    val = (val + sum(data_shards[s][i] for s in range(self.ERASURE_DATA_SHARDS))) % 256
                parity[i] = val & 0xFF
            parity_shards.append(bytes(parity))
        all_shards = data_shards + parity_shards
        while len(all_shards) < self.TOTAL_SHARDS:
            all_shards.append(None)
        return all_shards

    def _erasure_decode(self, shards: List[Optional[bytes]], original_size: int) -> bytes:
        """简化的纠删码解码：从可用分片重建数据"""
        available_shards = [(i, s) for i, s in enumerate(shards) if s is not None]
        if len(available_shards) < self.ERASURE_DATA_SHARDS:
            raise ValueError(f"可用分片不足：需要{self.ERASURE_DATA_SHARDS}个，仅有{len(available_shards)}个")
        data_shards = available_shards[:self.ERASURE_DATA_SHARDS]
        shard_size = len(data_shards[0][1])
        reconstructed = bytearray(shard_size * self.ERASURE_DATA_SHARDS)
        for shard_idx, shard_data in data_shards:
            start = shard_idx * shard_size
            end = start + shard_size
            if end <= len(reconstructed):
                reconstructed[start:end] = shard_data[:shard_size]
            else:
                remaining = len(reconstructed) - start
                if remaining > 0:
                    reconstructed[start:start + remaining] = shard_data[:remaining]
        return bytes(reconstructed[:original_size])

    def store_dicom_series(self, series_metadata: Dict, instances: List[Dict]) -> Dict:
        """验证DICOM一致性，一致性哈希分布实例，纠删码存储像素数据，创建系列清单"""
        study_uid = series_metadata.get("study_instance_uid", "")
        series_uid = series_metadata.get("series_instance_uid", "")
        if not study_uid or not series_uid:
            return {"status": "error", "message": "缺少Study Instance UID或Series Instance UID"}
        validation_errors = []
        for idx, instance in enumerate(instances):
            is_valid, errors = self._validate_dicom_conformance(instance)
            if not is_valid:
                validation_errors.extend([f"实例{idx}: {e}" for e in errors])
        if validation_errors:
            return {"status": "validation_failed", "errors": validation_errors}
        instance_locations = {}
        for instance in instances:
            instance_uid = instance.get("sop_instance_uid", f"inst_{int(time.time()*1000)}")
            pixel_data = instance.get(self.DICOM_PIXEL_DATA_TAG, b"")
            if isinstance(pixel_data, str):
                pixel_data = pixel_data.encode("utf-8")
            original_size = len(pixel_data)
            shards = self._erasure_encode(pixel_data)
            primary_nodes = self._get_nodes_for_key(study_uid, count=self.TOTAL_SHARDS)
            if len(primary_nodes) < self.ERASURE_DATA_SHARDS:
                return {"status": "error", "message": "可用存储节点不足"}
            shard_locations = []
            for shard_idx, shard_data in enumerate(shards):
                if shard_data is not None and shard_idx < len(primary_nodes):
                    node_id = primary_nodes[shard_idx]
                    object_key = f"{study_uid}/{series_uid}/{instance_uid}_shard_{shard_idx}"
                    checksum = self._compute_checksum(shard_data)
                    self.object_checksums[object_key] = checksum
                    shard_locations.append({
                        "shard_idx": shard_idx,
                        "node_id": node_id,
                        "object_key": object_key,
                        "size": len(shard_data),
                        "checksum": checksum
                    })
            instance_locations[instance_uid] = {
                "shard_locations": shard_locations,
                "original_size": original_size,
                "series_number": instance.get(self.DICOM_SERIES_NUMBER_TAG, 0),
                "instance_number": instance.get(self.DICOM_INSTANCE_NUMBER_TAG, 0),
                "sop_class_uid": instance.get(self.DICOM_SOP_CLASS_UID_TAG, ""),
                "transfer_syntax": instance.get(self.DICOM_TRANSFER_SYNTAX_TAG, "")
            }
            self.object_locations[instance_uid] = [s["node_id"] for s in shard_locations]
        modality = series_metadata.get("modality", "CT")
        manifest = {
            "study_instance_uid": study_uid,
            "series_instance_uid": series_uid,
            "modality": modality,
            "patient_id": series_metadata.get("patient_id", ""),
            "patient_name": series_metadata.get("patient_name", ""),
            "study_date": series_metadata.get("study_date", ""),
            "study_description": series_metadata.get("study_description", ""),
            "series_description": series_metadata.get("series_description", ""),
            "instance_count": len(instances),
            "instance_locations": instance_locations,
            "windowing_presets": self.WINDOWING_PRESETS.get(modality, []),
            "created_at": time.time(),
            "total_size_bytes": sum(
                loc["original_size"] for loc in instance_locations.values()
            )
        }
        self.study_manifests[study_uid] = manifest
        return {
            "status": "stored",
            "study_uid": study_uid,
            "series_uid": series_uid,
            "instance_count": len(instances),
            "total_size_bytes": manifest["total_size_bytes"],
            "shard_distribution": {
                node_id: sum(1 for loc in instance_locations.values()
                            for s in loc["shard_locations"] if s["node_id"] == node_id)
                for node_id in set(
                    s["node_id"]
                    for loc in instance_locations.values()
                    for s in loc["shard_locations"]
                )
            }
        }

    def retrieve_dicom_study(self, study_uid: str) -> Dict:
        """从清单查找实例位置，并行从多节点检索，按序重组DICOM数据集，应用窗宽窗位预设"""
        if study_uid not in self.study_manifests:
            return {"status": "not_found", "study_uid": study_uid}
        manifest = self.study_manifests[study_uid]
        instance_locations = manifest.get("instance_locations", {})
        if not instance_locations:
            return {"status": "no_instances", "study_uid": study_uid}
        retrieved_instances = []
        for instance_uid in instance_locations:
            loc_info = instance_locations[instance_uid]
            shard_locations = loc_info["shard_locations"]
            original_size = loc_info["original_size"]
            shards = [None] * self.TOTAL_SHARDS
            for shard_info in shard_locations:
                shard_idx = shard_info["shard_idx"]
                object_key = shard_info["object_key"]
                stored_checksum = shard_info["checksum"]
                retrieved_data = self._retrieve_from_node(
                    shard_info["node_id"], object_key
                )
                if retrieved_data is not None:
                    actual_checksum = self._compute_checksum(retrieved_data)
                    if actual_checksum == stored_checksum:
                        shards[shard_idx] = retrieved_data
                    else:
                        alternate_data = self._retrieve_from_alternate_node(
                            object_key, shard_info["node_id"]
                        )
                        if alternate_data is not None:
                            alt_checksum = self._compute_checksum(alternate_data)
                            if alt_checksum == stored_checksum:
                                shards[shard_idx] = alternate_data
            try:
                pixel_data = self._erasure_decode(shards, original_size)
            except ValueError as e:
                retrieved_instances.append({
                    "instance_uid": instance_uid,
                    "status": "reconstruction_failed",
                    "error": str(e)
                })
                continue
            retrieved_instances.append({
                "instance_uid": instance_uid,
                "series_number": loc_info["series_number"],
                "instance_number": loc_info["instance_number"],
                "sop_class_uid": loc_info["sop_class_uid"],
                "transfer_syntax": loc_info["transfer_syntax"],
                "pixel_data": pixel_data,
                "pixel_data_size": len(pixel_data),
                "status": "retrieved"
            })
        sorted_instances = sorted(
            [i for i in retrieved_instances if i["status"] == "retrieved"],
            key=lambda x: (x.get("series_number", 0), x.get("instance_number", 0))
        )
        modality = manifest.get("modality", "CT")
        windowing_presets = self.WINDOWING_PRESETS.get(modality, [])
        progressive_levels = self._generate_progressive_resolution(sorted_instances)
        return {
            "status": "retrieved",
            "study_uid": study_uid,
            "manifest": {
                "patient_id": manifest.get("patient_id", ""),
                "patient_name": manifest.get("patient_name", ""),
                "study_date": manifest.get("study_date", ""),
                "study_description": manifest.get("study_description", ""),
                "modality": modality
            },
            "instance_count": len(sorted_instances),
            "total_retrieved": len([i for i in retrieved_instances if i["status"] == "retrieved"]),
            "failed_retrievals": len([i for i in retrieved_instances if i["status"] != "retrieved"]),
            "instances": sorted_instances,
            "windowing_presets": windowing_presets,
            "progressive_resolution_levels": progressive_levels
        }

    def _retrieve_from_node(self, node_id: str, object_key: str) -> Optional[bytes]:
        """从指定节点检索对象数据"""
        if node_id not in self.storage_nodes:
            return None
        node_status = self.node_health.get(node_id, {}).get("status", "offline")
        if node_status != "online":
            return None
        node = self.storage_nodes[node_id]
        storage = node.get("storage", {})
        if object_key in storage:
            return storage[object_key]
        return None

    def _retrieve_from_alternate_node(self, object_key: str, failed_node: str) -> Optional[bytes]:
        """从备用节点检索对象"""
        for node_id, node in self.storage_nodes.items():
            if node_id == failed_node:
                continue
            if self.node_health.get(node_id, {}).get("status") != "online":
                continue
            storage = node.get("storage", {})
            if object_key in storage:
                return storage[object_key]
        return None

    def _generate_progressive_resolution(self, instances: List[Dict]) -> List[Dict]:
        """生成渐进式分辨率传输级别"""
        total_instances = len(instances)
        if total_instances == 0:
            return []
        levels = []
        step = max(1, total_instances // 4)
        thumbnail_count = min(total_instances, max(1, total_instances // 10))
        levels.append({
            "level": 1,
            "description": "缩略图",
            "instance_indices": list(range(0, thumbnail_count)),
            "resolution": "64x64",
            "estimated_size_kb": thumbnail_count * 4
        })
        preview_indices = list(range(0, min(total_instances, step)))
        levels.append({
            "level": 2,
            "description": "预览",
            "instance_indices": preview_indices,
            "resolution": "256x256",
            "estimated_size_kb": len(preview_indices) * 32
        })
        diagnostic_indices = list(range(0, min(total_instances, step * 2)))
        levels.append({
            "level": 3,
            "description": "诊断质量",
            "instance_indices": diagnostic_indices,
            "resolution": "512x512",
            "estimated_size_kb": len(diagnostic_indices) * 128
        })
        levels.append({
            "level": 4,
            "description": "全分辨率",
            "instance_indices": list(range(total_instances)),
            "resolution": "original",
            "estimated_size_kb": sum(i.get("pixel_data_size", 0) for i in instances) // 1024
        })
        return levels

    def replicate_to_backup(self, storage_node_id: str) -> Dict:
        """识别故障节点上的所有对象，使用一致性哈希选择替换节点，校验复制，更新位置元数据"""
        if storage_node_id not in self.storage_nodes:
            return {"status": "node_not_found", "node_id": storage_node_id}
        affected_objects = []
        for instance_uid, node_list in self.object_locations.items():
            if storage_node_id in node_list:
                affected_objects.append(instance_uid)
        if not affected_objects:
            return {"status": "no_objects_to_replicate", "node_id": storage_node_id}
        replication_results = []
        for instance_uid in affected_objects:
            manifest_found = False
            source_data = None
            for study_uid, manifest in self.study_manifests.items():
                if instance_uid in manifest.get("instance_locations", {}):
                    loc_info = manifest["instance_locations"][instance_uid]
                    for shard_info in loc_info["shard_locations"]:
                        if shard_info["node_id"] == storage_node_id:
                            alt_data = self._retrieve_from_alternate_node(
                                shard_info["object_key"], storage_node_id
                            )
                            if alt_data is not None:
                                source_data = alt_data
                                manifest_found = True
                                break
                    if manifest_found:
                        break
            if source_data is None:
                replication_results.append({
                    "instance_uid": instance_uid,
                    "status": "failed",
                    "reason": "无法从任何节点获取源数据"
                })
                continue
            replacement_nodes = self._get_nodes_for_key(
                instance_uid, count=self.TOTAL_SHARDS + 2
            )
            replacement_node = None
            for node in replacement_nodes:
                if node != storage_node_id:
                    replacement_node = node
                    break
            if replacement_node is None:
                replication_results.append({
                    "instance_uid": instance_uid,
                    "status": "failed",
                    "reason": "无可用替换节点"
                })
                continue
            new_object_key = f"replica_{instance_uid}_on_{replacement_node}"
            stored_checksum = self._compute_checksum(source_data)
            node_storage = self.storage_nodes[replacement_node].setdefault("storage", {})
            node_storage[new_object_key] = source_data
            verify_data = node_storage.get(new_object_key)
            if verify_data is not None:
                verify_checksum = self._compute_checksum(verify_data)
                checksum_match = verify_checksum == stored_checksum
            else:
                checksum_match = False
            if checksum_match:
                if storage_node_id in self.object_locations[instance_uid]:
                    idx = self.object_locations[instance_uid].index(storage_node_id)
                    self.object_locations[instance_uid][idx] = replacement_node
                replication_results.append({
                    "instance_uid": instance_uid,
                    "status": "replicated",
                    "new_node": replacement_node,
                    "new_object_key": new_object_key,
                    "checksum_verified": True
                })
            else:
                replication_results.append({
                    "instance_uid": instance_uid,
                    "status": "checksum_mismatch",
                    "new_node": replacement_node,
                    "expected_checksum": stored_checksum
                })
        successful = sum(1 for r in replication_results if r["status"] == "replicated")
        failed = sum(1 for r in replication_results if r["status"] != "replicated")
        return {
            "status": "completed",
            "source_node_id": storage_node_id,
            "total_objects": len(affected_objects),
            "successful_replications": successful,
            "failed_replications": failed,
            "results": replication_results
        }

    def handle_storage_node_failure(self, node_id: str) -> Dict:
        """检测节点故障，标记离线，触发自动重复制，更新路由表，通知管理员"""
        if node_id not in self.storage_nodes:
            return {"status": "node_not_found", "node_id": node_id}
        last_health = self.node_health.get(node_id, {})
        is_timeout = (time.time() - last_health.get("last_check", 0) > self.HEALTH_CHECK_TIMEOUT_SECONDS)
        is_unreachable = last_health.get("status") == "offline"
        if not is_timeout and not is_unreachable:
            self.node_health[node_id]["failure_count"] += 1
            if self.node_health[node_id]["failure_count"] < 3:
                return {
                    "status": "monitoring",
                    "node_id": node_id,
                    "failure_count": self.node_health[node_id]["failure_count"],
                    "message": "节点故障计数未达阈值，继续监控"
                }
        self.node_health[node_id] = {
            "status": "offline",
            "last_check": time.time(),
            "failure_count": self.node_health.get(node_id, {}).get("failure_count", 0) + 1
        }
        affected_object_count = 0
        for instance_uid, nodes in self.object_locations.items():
            if node_id in nodes:
                affected_object_count += 1
        new_ring = []
        for hash_val, nid in self.consistent_hash_ring:
            if nid != node_id:
                new_ring.append((hash_val, nid))
        self.consistent_hash_ring = sorted(new_ring)
        estimated_replication_size = 0
        for study_uid, manifest in self.study_manifests.items():
            for inst_uid, loc_info in manifest.get("instance_locations", {}).items():
                for shard in loc_info.get("shard_locations", []):
                    if shard["node_id"] == node_id:
                        estimated_replication_size += shard.get("size", 0)
        avg_transfer_rate_bps = 50 * 1024 * 1024  # 50 MB/s
        estimated_replication_seconds = estimated_replication_size / avg_transfer_rate_bps if avg_transfer_rate_bps > 0 else 0
        estimated_replication_hours = estimated_replication_seconds / 3600
        needs_admin_notification = estimated_replication_hours > self.REPLICATION_THRESHOLD_HOURS
        replication_task = {
            "task_id": f"repl_{node_id}_{int(time.time())}",
            "source_node_id": node_id,
            "affected_objects": affected_object_count,
            "estimated_size_bytes": estimated_replication_size,
            "estimated_time_hours": round(estimated_replication_hours, 2),
            "status": "initiated",
            "created_at": time.time()
        }
        self.replication_tasks.append(replication_task)
        if needs_admin_notification:
            notification = {
                "type": "storage_node_failure",
                "severity": "high",
                "node_id": node_id,
                "affected_objects": affected_object_count,
                "estimated_replication_hours": round(estimated_replication_hours, 2),
                "message": (f"存储节点{node_id}故障，需要重复制{affected_object_count}个对象，"
                            f"预计耗时{round(estimated_replication_hours, 1)}小时，超过1小时阈值"),
                "timestamp": time.time()
            }
            self.admin_notifications.append(notification)
        replication_result = self.replicate_to_backup(node_id)
        return {
            "status": "handled",
            "node_id": node_id,
            "previous_status": last_health.get("status", "unknown"),
            "current_status": "offline",
            "affected_objects": affected_object_count,
            "routing_table_updated": True,
            "replication_initiated": True,
            "replication_result": replication_result,
            "admin_notified": needs_admin_notification,
            "estimated_recovery_time_hours": round(estimated_replication_hours, 2)
        }
```

## 异常场景补充

### 场景：PACS 系统存储空间不足
```
trigger: 医院PACS系统存储使用率超过95%，新的CT/MR检查影像无法写入，影像归档系统报错"disk space insufficient"，急诊影像可能丢失
detection: 1) 实时监控每个存储节点的磁盘使用率，当超过85%时告警，超过90%时紧急告警；2) 监控DICOM存储接口的写入失败率，当连续出现写入失败时触发告警；3) 按天预测存储增长趋势，当预计7天内将超过95%时提前预警；4) 监控纠删码编码写入的延迟，存储空间不足时I/O延迟会显著增加
handling: 1) 立即触发自动归档策略：将超过90天的已报告影像迁移至冷存储（如磁带库或对象存储归档层）；2) 启用DICOM压缩：对未压缩的影像执行JPEG2000无损压缩，通常可节省40-60%空间；3) 临时启用急诊影像优先存储策略，非急诊影像排队等待；4) 清理临时文件、失败事务残留数据和重复的DICOM实例；5) 紧急扩容：添加新存储节点到一致性哈希环；6) 通知IT部门启动容量扩容流程
prevention: 1) 建立三级存储架构：热存储（SSD，最近30天）→温存储（HDD，30-180天）→冷存储（磁带/归档，180天以上）；2) 设置自动数据生命周期管理策略，按期自动迁移和清理；3) 实现DICOM影像自动压缩，存储时默认使用JPEG2000无损压缩；4) 每月进行存储容量规划审查，提前6个月预测扩容需求；5) 设置存储使用率预警阈值：70%通知、80%告警、90%紧急行动；6) 去重：对重复的DICOM实例（相同SOP Instance UID）只保留一份
```

### 场景：DICOM 传输中断导致数据不完整
```
trigger: 网络故障或存储节点异常导致DICOM实例传输中断，部分切片丢失或像素数据截断，影像无法正常显示或诊断为不完整检查
detection: 1) 验证接收到的实例数量与DICOM系列预期的实例数量（NumberOfFrames或实例数标签）是否匹配；2) 检查每个实例的像素数据长度是否与Rows×Columns×BitsAllocated×SamplesPerPixel的计算值一致；3) 校验每个分片的SHA256校验和与存储时记录的校验和是否一致；4) 检查DICOM文件是否以正确的DICM前缀开始且结束标记完整
handling: 1) 标记受影响的系列为"传输不完整"状态，阻止放射科医生阅片直到数据完整；2) 自动向源系统（如CT/MR设备或上游PACS）发起C-MOVE请求重新获取缺失实例；3) 对已部分接收的实例，尝试续传而非重新传输整个实例；4) 如果源设备不可用，检查纠删码分片是否足以重建数据；5) 在影像查看器上显示"数据不完整"警告，标注缺失的切片范围；6) 记录传输中断的详细日志用于事后分析
prevention: 1) 实现DICOM传输的断点续传机制，使用C-GET替代C-MOVE以支持流式传输；2) 在传输完成后自动执行完整性校验（实例数+像素数据长度+校验和）；3) 使用事务性存储：先写入临时位置，校验通过后原子性地移动到正式位置；4) 部署冗余传输通道，主通道失败时自动切换到备用通道；5) 在存储层实现写前日志（WAL），确保中断后可以恢复到一致状态；6) 对关键检查（如急诊CT）设置更高的存储确认级别，要求所有分片写入确认后才返回成功
```

### 场景：存储节点故障影响在线阅片
```
trigger: 存储节点在门诊高峰期故障，导致放射科医生正在阅片的检查影像无法加载，诊断工作站显示超时错误，影响急诊影像的及时诊断
detection: 1) 节点健康检查超时（10秒无响应）立即标记为疑似故障；2) 监控DICOM检索请求的失败率和延迟，当特定节点上的请求失败率超过10%时告警；3) 检测到连续3次健康检查失败则确认节点故障；4) 监控诊断工作站的影像加载时间，超过10秒触发体验降级告警
handling: 1) 立即触发自动重复制，将故障节点的数据迁移到健康节点（参见replicate_to_backup流程）；2) 对正在阅片的检查，优先从纠删码的其他分片重建数据（4+2纠删码可容忍2个分片丢失）；3) 临时启用缓存节点：将最近访问的影像缓存到本地或内存中，优先保障当前阅片会话；4) 如果重建也失败，从PACS的备份存储节点获取数据；5) 在诊断工作站上显示节点故障提示和预计恢复时间；6) 通知IT运维团队紧急处理硬件故障
prevention: 1) 确保纠删码配置至少4+2（可容忍2个节点故障），关键数据使用4+3配置；2) 实现热点数据的主动预取：将最近3天内可能被阅片的检查预复制到多个节点；3) 部署边缘缓存：在诊断工作站本地缓存最近访问的影像；4) 存储节点使用热插拔硬盘和冗余电源，减少硬件故障影响；5) 实现读请求的多节点负载均衡，避免单点依赖；6) 建立阅片优先级机制：急诊影像始终保证3副本存储，常规影像至少2副本；7) 定期进行故障演练，验证自动重复制流程的可靠性
```

## 医学影像分布式存储与传输完整实现

```python
import hashlib
import json
import time
import threading
import concurrent.futures
from datetime import datetime, timezone
from enum import Enum


class StorageNodeStatus(Enum):
    ONLINE = "online"
    OFFLINE = "offline"
    DEGRADED = "degraded"


class DICOMConformanceError(Exception):
    def __init__(self, message, sop_class_uid=None, transfer_syntax=None):
        super().__init__(message)
        self.sop_class_uid = sop_class_uid
        self.transfer_syntax = transfer_syntax


class StorageInsufficientError(Exception):
    def __init__(self, message, required_bytes=0, available_bytes=0):
        super().__init__(message)
        self.required_bytes = required_bytes
        self.available_bytes = available_bytes


class DICOMTransferInterruptedError(Exception):
    def __init__(self, message, study_uid=None, transferred_instances=0, total_instances=0):
        super().__init__(message)
        self.study_uid = study_uid
        self.transferred_instances = transferred_instances
        self.total_instances = total_instances


class MedicalImageStorageService:
    """医学影像分布式存储与传输服务，提供DICOM影像的存储、检索、复制和故障恢复能力"""

    APPROVED_SOP_CLASSES = {
        "1.2.840.10008.5.1.4.1.1.2": "CT Image Storage",
        "1.2.840.10008.5.1.4.1.1.4": "MR Image Storage",
        "1.2.840.10008.5.1.4.1.1.1": "CR Image Storage",
        "1.2.840.10008.5.1.4.1.1.1.1": "Digital X-Ray Image Storage",
        "1.2.840.10008.5.1.4.1.1.1.1.1": "Digital Mammography X-Ray Image Storage",
        "1.2.840.10008.5.1.4.1.1.1.2": "Digital Intra-Oral X-Ray Image Storage",
        "1.2.840.10008.5.1.4.1.1.3": "Ultrasound Multi-frame Image Storage",
        "1.2.840.10008.5.1.4.1.1.6.1": "Ultrasound Image Storage",
        "1.2.840.10008.5.1.4.1.1.7": "Secondary Capture Image Storage",
        "1.2.840.10008.5.1.4.1.1.77.1.1": "VL Endoscopic Image Storage",
        "1.2.840.10008.5.1.4.1.1.77.1.2": "VL Microscopic Image Storage",
        "1.2.840.10008.5.1.4.1.1.77.1.5.1": "Ophthalmic Photography 8 Bit Image Storage",
        "1.2.840.10008.5.1.4.1.1.12.1": "Enhanced CT Image Storage",
        "1.2.840.10008.5.1.4.1.1.12.2.1": "Enhanced MR Image Storage",
        "1.2.840.10008.5.1.4.1.1.13.1.3": "Breast Tomosynthesis Image Storage",
        "1.2.840.10008.5.1.4.1.1.2.1": "Enhanced CT Image Storage Legacy",
    }

    SUPPORTED_TRANSFER_SYNTAXES = [
        "1.2.840.10008.1.2",           # Implicit VR Little Endian
        "1.2.840.10008.1.2.1",         # Explicit VR Little Endian
        "1.2.840.10008.1.2.2",         # Explicit VR Big Endian
        "1.2.840.10008.1.2.4.50",      # JPEG Baseline (Process 1)
        "1.2.840.10008.1.2.4.51",      # JPEG Extended (Process 2 & 4)
        "1.2.840.10008.1.2.4.57",      # JPEG Lossless (Process 14)
        "1.2.840.10008.1.2.4.70",      # JPEG Lossless (Process 14, Selection Value 1)
        "1.2.840.10008.1.2.4.90",      # JPEG 2000 Image Compression (Lossless Only)
        "1.2.840.10008.1.2.4.91",      # JPEG 2000 Image Compression
    ]

    WINDOWING_PRESETS = {
        "CT_ABDOMEN": {"center": 40, "width": 400},
        "CT_LUNG": {"center": -600, "width": 1500},
        "CT_BRAIN": {"center": 40, "width": 80},
        "CT_BONE": {"center": 400, "width": 1800},
        "CT_MEDIASTINUM": {"center": 50, "width": 350},
        "MR_DEFAULT": {"center": 0, "width": 256},
        "MR_T1_BRAIN": {"center": 500, "width": 1000},
        "MR_T2_BRAIN": {"center": 1500, "width": 3000},
    }

    ERASURE_CODING_DATA_SHARDS = 4
    ERASURE_CODING_PARITY_SHARDS = 2
    VIRTUAL_NODES_PER_PHYSICAL = 100
    RING_SIZE = 10000
    HEALTH_CHECK_TIMEOUT_SECONDS = 5
    MAX_HEALTH_CHECK_RETRIES = 3
    RECOVERY_CHECK_INTERVAL_SECONDS = 300
    NETWORK_BANDWIDTH_MBPS = 100
    AVG_OBJECT_SIZE_MB = 5

    def __init__(self, object_storage, metadata_db, redis_client, alert_service):
        self.object_storage = object_storage
        self.metadata_db = metadata_db
        self.redis_client = redis_client
        self.alert_service = alert_service
        self.hash_ring = {}
        self.sorted_ring_keys = []
        self._recovery_threads = {}

    def _build_hash_ring(self, node_ids):
        ring = {}
        for node_id in node_ids:
            for v_idx in range(self.VIRTUAL_NODES_PER_PHYSICAL):
                hash_input = f"{node_id}_vnode_{v_idx}"
                hash_val = int(hashlib.sha256(hash_input.encode()).hexdigest(), 16)
                ring_index = hash_val % self.RING_SIZE
                ring[ring_index] = node_id
        self.hash_ring = ring
        self.sorted_ring_keys = sorted(ring.keys())

    def _get_nodes_for_key(self, key, count=1):
        if not self.sorted_ring_keys:
            raise RuntimeError("Hash ring is empty, no storage nodes available")
        hash_val = int(hashlib.sha256(key.encode()).hexdigest(), 16) % self.RING_SIZE
        selected_nodes = []
        seen_nodes = set()
        start_idx = 0
        for i, ring_key in enumerate(self.sorted_ring_keys):
            if ring_key >= hash_val:
                start_idx = i
                break
        else:
            start_idx = 0
        for offset in range(len(self.sorted_ring_keys)):
            idx = (start_idx + offset) % len(self.sorted_ring_keys)
            ring_key = self.sorted_ring_keys[idx]
            node_id = self.hash_ring[ring_key]
            if node_id not in seen_nodes:
                seen_nodes.add(node_id)
                selected_nodes.append(node_id)
                if len(selected_nodes) == count:
                    break
        return selected_nodes

    def _validate_dicom_conformance(self, series_metadata, instances):
        sop_class_uid = series_metadata.get("SOPClassUID", "")
        if sop_class_uid not in self.APPROVED_SOP_CLASSES:
            raise DICOMConformanceError(
                f"SOP Class UID '{sop_class_uid}' is not in approved list",
                sop_class_uid=sop_class_uid
            )
        for instance in instances:
            transfer_syntax = instance.get("TransferSyntaxUID", "")
            if transfer_syntax not in self.SUPPORTED_TRANSFER_SYNTAXES:
                raise DICOMConformanceError(
                    f"Transfer Syntax '{transfer_syntax}' is not supported",
                    transfer_syntax=transfer_syntax
                )
            instance_sop_class = instance.get("SOPClassUID", sop_class_uid)
            if instance_sop_class not in self.APPROVED_SOP_CLASSES:
                raise DICOMConformanceError(
                    f"Instance SOP Class UID '{instance_sop_class}' is not approved",
                    sop_class_uid=instance_sop_class
                )
        return True

    def store_dicom_series(self, series_metadata, instances):
        self._validate_dicom_conformance(series_metadata, instances)
        study_uid = series_metadata["StudyInstanceUID"]
        series_uid = series_metadata["SeriesInstanceUID"]
        online_nodes = self._get_online_storage_nodes()
        if not online_nodes:
            raise StorageInsufficientError("No online storage nodes available")
        self._build_hash_ring(online_nodes)
        instance_storage_records = []
        total_bytes_stored = 0
        for instance in instances:
            sop_uid = instance["SOPInstanceUID"]
            pixel_data = instance["pixel_data"]
            pixel_size = len(pixel_data)
            shard_size = (pixel_size + self.ERASURE_CODING_DATA_SHARDS - 1) // self.ERASURE_CODING_DATA_SHARDS
            shards = []
            for i in range(self.ERASURE_CODING_DATA_SHARDS):
                start = i * shard_size
                end = min(start + shard_size, pixel_size)
                shard_data = pixel_data[start:end]
                shards.append(shard_data)
            while len(shards) < self.ERASURE_CODING_DATA_SHARDS + self.ERASURE_CODING_PARITY_SHARDS:
                if len(shards) < self.ERASURE_CODING_DATA_SHARDS:
                    shards.append(b'')
                else:
                    parity_idx = len(shards) - self.ERASURE_CODING_DATA_SHARDS
                    parity_data = self._compute_parity_shard(shards[:self.ERASURE_CODING_DATA_SHARDS], parity_idx)
                    shards.append(parity_data)
            for i in range(self.ERASURE_CODING_DATA_SHARDS, self.ERASURE_CODING_DATA_SHARDS + self.ERASURE_CODING_PARITY_SHARDS):
                parity_idx = i - self.ERASURE_CODING_DATA_SHARDS
                parity_data = self._compute_parity_shard(shards[:self.ERASURE_CODING_DATA_SHARDS], parity_idx)
                shards[i] = parity_data
            storage_key_base = f"dicom/{study_uid}/{series_uid}/{sop_uid}"
            assigned_nodes = self._get_nodes_for_key(storage_key_base, count=2)
            primary_node = assigned_nodes[0]
            checksum = hashlib.sha256(pixel_data).hexdigest()
            shard_keys = []
            for shard_idx, shard_data in enumerate(shards):
                shard_key = f"{storage_key_base}/shard_{shard_idx}"
                shard_checksum = hashlib.md5(shard_data).hexdigest()
                self.object_storage.upload(
                    node_id=primary_node,
                    key=shard_key,
                    data=shard_data,
                    metadata={"study_uid": study_uid, "series_uid": series_uid, "sop_uid": sop_uid, "shard_index": shard_idx, "parent_checksum": checksum}
                )
                shard_keys.append(shard_key)
                total_bytes_stored += len(shard_data)
            instance_record = {
                "sop_uid": sop_uid,
                "storage_key": storage_key_base,
                "shard_keys": shard_keys,
                "checksum": checksum,
                "size_bytes": pixel_size,
                "primary_node": primary_node,
                "replica_nodes": assigned_nodes[1:],
            }
            instance_storage_records.append(instance_record)
            self.metadata_db.execute(
                "INSERT INTO object_locations (storage_key, node_id, sop_uid, study_uid, series_uid, checksum, size_bytes, created_at) VALUES (%s, %s, %s, %s, %s, %s, %s, %s)",
                (storage_key_base, primary_node, sop_uid, study_uid, series_uid, checksum, pixel_size, datetime.now(timezone.utc))
            )
        manifest = {
            "study_uid": study_uid,
            "series_uid": series_uid,
            "instance_count": len(instances),
            "instances": [
                {"sop_uid": rec["sop_uid"], "storage_key": rec["storage_key"], "checksum": rec["checksum"], "size_bytes": rec["size_bytes"]}
                for rec in instance_storage_records
            ],
            "created_at": datetime.now(timezone.utc).isoformat(),
            "erasure_coding": {
                "data_shards": self.ERASURE_CODING_DATA_SHARDS,
                "parity_shards": self.ERASURE_CODING_PARITY_SHARDS,
            },
            "total_bytes": total_bytes_stored,
        }
        self.metadata_db.execute(
            "INSERT INTO series_manifests (study_uid, series_uid, manifest_json, created_at) VALUES (%s, %s, %s, %s)",
            (study_uid, series_uid, json.dumps(manifest), datetime.now(timezone.utc))
        )
        confirmation = {
            "study_uid": study_uid,
            "series_uid": series_uid,
            "instance_count": len(instances),
            "storage_locations": instance_storage_records,
            "total_bytes_stored": total_bytes_stored,
            "manifest_stored": True,
            "erasure_coding_applied": True,
            "status": "stored",
        }
        return confirmation

    def _compute_parity_shard(self, data_shards, parity_index):
        max_len = max(len(s) for s in data_shards) if data_shards else 0
        parity = bytearray(max_len)
        for shard in data_shards:
            for j in range(len(shard)):
                if parity_index == 0:
                    parity[j] ^= shard[j]
                else:
                    parity[j] = (parity[j] + shard[j]) % 256
        return bytes(parity)

    def _get_online_storage_nodes(self):
        rows = self.metadata_db.query(
            "SELECT node_id FROM storage_nodes WHERE status = %s",
            (StorageNodeStatus.ONLINE.value,)
        )
        return [row["node_id"] for row in rows]

    def retrieve_dicom_study(self, study_uid):
        manifest_rows = self.metadata_db.query(
            "SELECT manifest_json FROM series_manifests WHERE study_uid = %s ORDER BY created_at DESC",
            (study_uid,)
        )
        if not manifest_rows:
            raise ValueError(f"No manifest found for study UID: {study_uid}")
        all_instances = []
        all_series_data = []
        for manifest_row in manifest_rows:
            manifest = json.loads(manifest_row["manifest_json"])
            series_uid = manifest["series_uid"]
            instance_list = manifest["instances"]
            location_rows = self.metadata_db.query(
                "SELECT sop_uid, storage_key, node_id, checksum, size_bytes FROM object_locations WHERE study_uid = %s AND series_uid = %s",
                (study_uid, series_uid)
            )
            location_map = {}
            for loc in location_rows:
                location_map[loc["sop_uid"]] = loc
            def fetch_instance(inst_info):
                sop_uid = inst_info["sop_uid"]
                storage_key = inst_info["storage_key"]
                loc = location_map.get(sop_uid)
                if not loc:
                    return None
                node_id = loc["node_id"]
                shard_keys_pattern = f"{storage_key}/shard_"
                data_shards = []
                for shard_idx in range(self.ERASURE_CODING_DATA_SHARDS):
                    shard_key = f"{storage_key}/shard_{shard_idx}"
                    shard_data = self.object_storage.download(node_id=node_id, key=shard_key)
                    data_shards.append(shard_data)
                pixel_data = b''.join(data_shards)
                expected_checksum = loc["checksum"]
                actual_checksum = hashlib.sha256(pixel_data).hexdigest()
                if actual_checksum != expected_checksum:
                    parity_shards = []
                    for p_idx in range(self.ERASURE_CODING_PARITY_SHARDS):
                        parity_key = f"{storage_key}/shard_{self.ERASURE_CODING_DATA_SHARDS + p_idx}"
                        parity_data = self.object_storage.download(node_id=node_id, key=parity_key)
                        parity_shards.append(parity_data)
                    pixel_data = self._reconstruct_with_parity(data_shards, parity_shards)
                    reconstruct_checksum = hashlib.sha256(pixel_data).hexdigest()
                    if reconstruct_checksum != expected_checksum:
                        raise ValueError(f"Data corruption detected for SOP instance {sop_uid}, checksum mismatch after reconstruction")
                return {
                    "sop_uid": sop_uid,
                    "series_uid": series_uid,
                    "pixel_data": pixel_data,
                    "size_bytes": loc["size_bytes"],
                    "series_number": self._get_series_number(sop_uid),
                    "instance_number": self._get_instance_number(sop_uid),
                }
            with concurrent.futures.ThreadPoolExecutor(max_workers=4) as executor:
                future_to_instance = {executor.submit(fetch_instance, inst): inst for inst in instance_list}
                for future in concurrent.futures.as_completed(future_to_instance):
                    result = future.result()
                    if result is not None:
                        all_instances.append(result)
        all_instances.sort(key=lambda x: (x["series_number"], x["instance_number"]))
        study_data = {
            "study_uid": study_uid,
            "instances": all_instances,
            "total_instances": len(all_instances),
        }
        windowed_data = self._apply_windowing(study_data)
        streamed_result = self._progressive_load(windowed_data)
        return streamed_result

    def _reconstruct_with_parity(self, data_shards, parity_shards):
        all_shards = list(data_shards) + list(parity_shards)
        missing_indices = [i for i, s in enumerate(data_shards) if len(s) == 0]
        if len(missing_indices) == 0:
            return b''.join(data_shards)
        if len(missing_indices) > self.ERASURE_CODING_PARITY_SHARDS:
            raise ValueError("Too many missing data shards to reconstruct")
        original_max_len = max(len(s) for s in data_shards if len(s) > 0) if any(len(s) > 0 for s in data_shards) else 0
        reconstructed_shards = list(data_shards)
        for missing_idx in missing_indices:
            reconstructed = bytearray(original_max_len)
            parity_0 = parity_shards[0] if len(parity_shards) > 0 else None
            if parity_0 and len(parity_0) >= original_max_len:
                for j in range(original_max_len):
                    reconstructed[j] = parity_0[j]
                for i, shard in enumerate(data_shards):
                    if i != missing_idx and len(shard) >= original_max_len:
                        for j in range(original_max_len):
                            reconstructed[j] ^= shard[j]
            else:
                for j in range(original_max_len):
                    reconstructed[j] = 0
                for i, shard in enumerate(data_shards):
                    if i != missing_idx and len(shard) > j:
                        reconstructed[j] = (reconstructed[j] + shard[j]) % 256
            reconstructed_shards[missing_idx] = bytes(reconstructed)
        return b''.join(reconstructed_shards)

    def _get_series_number(self, sop_uid):
        row = self.metadata_db.query(
            "SELECT series_number FROM dicom_instance_metadata WHERE sop_uid = %s",
            (sop_uid,)
        )
        if row:
            return row[0].get("series_number", 0)
        return 0

    def _get_instance_number(self, sop_uid):
        row = self.metadata_db.query(
            "SELECT instance_number FROM dicom_instance_metadata WHERE sop_uid = %s",
            (sop_uid,)
        )
        if row:
            return row[0].get("instance_number", 0)
        return 0

    def _apply_windowing(self, study_data):
        instances = study_data["instances"]
        for instance in instances:
            series_uid = instance["series_uid"]
            modality = self._infer_modality(series_uid)
            preset_key = self._select_windowing_preset(modality, series_uid)
            if preset_key and preset_key in self.WINDOWING_PRESETS:
                preset = self.WINDOWING_PRESETS[preset_key]
                center = preset["center"]
                width = preset["width"]
                pixel_data = instance["pixel_data"]
                windowed_pixels = self._apply_window_to_pixels(pixel_data, center, width)
                instance["windowed_pixel_data"] = windowed_pixels
                instance["windowing_applied"] = {"center": center, "width": width, "preset": preset_key}
            else:
                instance["windowed_pixel_data"] = instance["pixel_data"]
                instance["windowing_applied"] = None
        return study_data

    def _infer_modality(self, series_uid):
        row = self.metadata_db.query(
            "SELECT modality FROM dicom_series_metadata WHERE series_uid = %s",
            (series_uid,)
        )
        if row:
            return row[0].get("modality", "unknown")
        return "unknown"

    def _select_windowing_preset(self, modality, series_uid):
        if modality == "CT":
            body_part = self._get_body_part_examined(series_uid)
            if body_part and "ABDOMEN" in body_part.upper():
                return "CT_ABDOMEN"
            elif body_part and "LUNG" in body_part.upper() or "CHEST" in body_part.upper():
                return "CT_LUNG"
            elif body_part and "BRAIN" in body_part.upper() or "HEAD" in body_part.upper():
                return "CT_BRAIN"
            elif body_part and "BONE" in body_part.upper() or "SPINE" in body_part.upper():
                return "CT_BONE"
            else:
                return "CT_ABDOMEN"
        elif modality == "MR":
            return "MR_DEFAULT"
        else:
            return None

    def _get_body_part_examined(self, series_uid):
        row = self.metadata_db.query(
            "SELECT body_part_examined FROM dicom_series_metadata WHERE series_uid = %s",
            (series_uid,)
        )
        if row:
            return row[0].get("body_part_examined", "")
        return ""

    def _apply_window_to_pixels(self, pixel_data, center, width):
        low = center - width // 2
        high = center + width // 2
        pixel_array = list(pixel_data)
        windowed = []
        for pixel_val in pixel_array:
            if pixel_val <= low:
                windowed.append(0)
            elif pixel_val >= high:
                windowed.append(255)
            else:
                normalized = int(255 * (pixel_val - low) / width)
                windowed.append(normalized)
        return bytes(windowed)

    def _progressive_load(self, study_data):
        instances = study_data["instances"]
        progressive_chunks = []
        thumbnail_instances = []
        full_instances = []
        for instance in instances:
            pixel_data = instance.get("windowed_pixel_data", instance.get("pixel_data", b''))
            if len(pixel_data) == 0:
                continue
            thumbnail = self._generate_thumbnail(pixel_data, 64, 64)
            thumbnail_instances.append({
                "sop_uid": instance["sop_uid"],
                "series_uid": instance["series_uid"],
                "pixel_data": thumbnail,
                "resolution": "64x64",
                "is_thumbnail": True,
                "windowing_applied": instance.get("windowing_applied"),
            })
            full_instances.append({
                "sop_uid": instance["sop_uid"],
                "series_uid": instance["series_uid"],
                "pixel_data": pixel_data,
                "resolution": "full",
                "is_thumbnail": False,
                "windowing_applied": instance.get("windowing_applied"),
            })
        progressive_chunks.append({
            "chunk_type": "thumbnail",
            "instances": thumbnail_instances,
            "total_bytes": sum(len(inst["pixel_data"]) for inst in thumbnail_instances),
            "progress_pct": 10,
        })
        progressive_chunks.append({
            "chunk_type": "full_resolution",
            "instances": full_instances,
            "total_bytes": sum(len(inst["pixel_data"]) for inst in full_instances),
            "progress_pct": 100,
        })
        return {
            "study_uid": study_data["study_uid"],
            "total_instances": len(instances),
            "progressive_chunks": progressive_chunks,
            "streaming_ready": True,
        }

    def _generate_thumbnail(self, pixel_data, target_width, target_height):
        side = int(len(pixel_data) ** 0.5)
        if side == 0:
            return b'\x00' * (target_width * target_height)
        src_width = side
        src_height = side
        x_ratio = src_width / target_width
        y_ratio = src_height / target_height
        thumbnail = bytearray(target_width * target_height)
        for ty in range(target_height):
            src_y = int(ty * y_ratio)
            for tx in range(target_width):
                src_x = int(tx * x_ratio)
                src_idx = src_y * src_width + src_x
                if src_idx < len(pixel_data):
                    thumbnail[ty * target_width + tx] = pixel_data[src_idx]
                else:
                    thumbnail[ty * target_width + tx] = 0
        return bytes(thumbnail)

    def replicate_to_backup(self, failed_node_id):
        start_time = time.time()
        object_rows = self.metadata_db.query(
            "SELECT storage_key, checksum, size_bytes FROM object_locations WHERE node_id = %s",
            (failed_node_id,)
        )
        if not object_rows:
            return {
                "total_objects": 0,
                "replicated_count": 0,
                "failed_count": 0,
                "duration_seconds": 0.0,
                "message": "No objects found on failed node",
            }
        online_nodes = self._get_online_storage_nodes()
        online_nodes = [n for n in online_nodes if n != failed_node_id]
        if len(online_nodes) < 2:
            raise StorageInsufficientError(
                f"Not enough online nodes for replication. Need 2, have {len(online_nodes)}",
                required_bytes=0,
                available_bytes=0,
            )
        self._build_hash_ring(online_nodes)
        replacement_nodes = self._get_nodes_for_key(f"recovery_{failed_node_id}", count=2)
        if len(replacement_nodes) < 2:
            replacement_nodes = online_nodes[:2]
        target_node_1 = replacement_nodes[0]
        target_node_2 = replacement_nodes[1]
        replicated_count = 0
        failed_count = 0
        for obj_row in object_rows:
            storage_key = obj_row["storage_key"]
            source_checksum = obj_row["checksum"]
            size_bytes = obj_row["size_bytes"]
            try:
                for shard_idx in range(self.ERASURE_CODING_DATA_SHARDS + self.ERASURE_CODING_PARITY_SHARDS):
                    shard_key = f"{storage_key}/shard_{shard_idx}"
                    source_data = self.object_storage.download(node_id=failed_node_id, key=shard_key)
                    if source_data is None:
                        shard_key_alt = shard_key
                        for online_node in online_nodes:
                            source_data = self.object_storage.download(node_id=online_node, key=shard_key_alt)
                            if source_data is not None:
                                break
                    if source_data is None:
                        raise ValueError(f"Cannot retrieve shard {shard_key} from any available node")
                    source_md5 = hashlib.md5(source_data).hexdigest()
                    self.object_storage.copy(source_key=shard_key, dest_key=shard_key, source_node=failed_node_id, dest_node=target_node_1)
                    dest_data_1 = self.object_storage.download(node_id=target_node_1, key=shard_key)
                    dest_md5_1 = hashlib.md5(dest_data_1).hexdigest() if dest_data_1 else None
                    if source_md5 != dest_md5_1:
                        self.object_storage.upload(node_id=target_node_1, key=shard_key, data=source_data, metadata={"recovery": True})
                        dest_data_1 = self.object_storage.download(node_id=target_node_1, key=shard_key)
                        dest_md5_1 = hashlib.md5(dest_data_1).hexdigest() if dest_data_1 else None
                    if source_md5 != dest_md5_1:
                        raise ValueError(f"Checksum mismatch after copy for shard {shard_key} on node {target_node_1}")
                    self.object_storage.copy(source_key=shard_key, dest_key=shard_key, source_node=failed_node_id, dest_node=target_node_2)
                    dest_data_2 = self.object_storage.download(node_id=target_node_2, key=shard_key)
                    dest_md5_2 = hashlib.md5(dest_data_2).hexdigest() if dest_data_2 else None
                    if source_md5 != dest_md5_2:
                        self.object_storage.upload(node_id=target_node_2, key=shard_key, data=source_data, metadata={"recovery": True})
                        dest_data_2 = self.object_storage.download(node_id=target_node_2, key=shard_key)
                        dest_md5_2 = hashlib.md5(dest_data_2).hexdigest() if dest_data_2 else None
                    if source_md5 != dest_md5_2:
                        raise ValueError(f"Checksum mismatch after copy for shard {shard_key} on node {target_node_2}")
                self.metadata_db.execute(
                    "UPDATE object_locations SET node_id = %s WHERE storage_key = %s AND node_id = %s",
                    (target_node_1, storage_key, failed_node_id)
                )
                replicated_count += 1
            except Exception as e:
                failed_count += 1
                self.alert_service.send_alert(
                    level="error",
                    message=f"Failed to replicate object {storage_key} from node {failed_node_id}: {str(e)}",
                    component="MedicalImageStorageService",
                )
        total_objects = len(object_rows)
        db_count_row = self.metadata_db.query(
            "SELECT COUNT(*) as cnt FROM object_locations WHERE node_id IN %s",
            ((target_node_1, target_node_2),)
        )
        db_count = db_count_row[0]["cnt"] if db_count_row else 0
        duration = time.time() - start_time
        summary = {
            "total_objects": total_objects,
            "replicated_count": replicated_count,
            "failed_count": failed_count,
            "duration_seconds": round(duration, 2),
            "replacement_nodes": [target_node_1, target_node_2],
            "db_location_count": db_count,
            "validation_passed": db_count >= replicated_count,
        }
        return summary

    def handle_storage_node_failure(self, node_id):
        consecutive_failures = 0
        for attempt in range(self.MAX_HEALTH_CHECK_RETRIES):
            try:
                is_healthy = self.object_storage.health_check(node_id, timeout=self.HEALTH_CHECK_TIMEOUT_SECONDS)
                if is_healthy:
                    consecutive_failures = 0
                    return {"status": "healthy", "node_id": node_id, "action": "none"}
                else:
                    consecutive_failures += 1
            except Exception:
                consecutive_failures += 1
        if consecutive_failures < self.MAX_HEALTH_CHECK_RETRIES:
            return {"status": "degraded", "node_id": node_id, "action": "monitor"}
        self.metadata_db.execute(
            "UPDATE storage_nodes SET status = %s WHERE node_id = %s",
            (StorageNodeStatus.OFFLINE.value, node_id)
        )
        object_count_row = self.metadata_db.query(
            "SELECT COUNT(*) as cnt FROM object_locations WHERE node_id = %s",
            (node_id,)
        )
        object_count = object_count_row[0]["cnt"] if object_count_row else 0
        estimated_duration_hours = (object_count * self.AVG_OBJECT_SIZE_MB) / (self.NETWORK_BANDWIDTH_MBPS * 3.6)
        if estimated_duration_hours > 1.0:
            self.alert_service.send_alert(
                level="critical",
                message=f"Node {node_id} failure: re-replication will take estimated {estimated_duration_hours:.1f} hours ({object_count} objects). Manual intervention may be required.",
                component="MedicalImageStorageService",
                metadata={"estimated_duration_hours": estimated_duration_hours, "object_count": object_count},
            )
        self.redis_client.delete(f"route:{node_id}")
        replication_result = self.replicate_to_backup(node_id)
        if node_id not in self._recovery_threads or not self._recovery_threads[node_id].is_alive():
            recovery_thread = threading.Thread(
                target=self._periodic_health_recovery,
                args=(node_id,),
                daemon=True,
            )
            self._recovery_threads[node_id] = recovery_thread
            recovery_thread.start()
        result = {
            "status": "node_offline",
            "node_id": node_id,
            "consecutive_failures": consecutive_failures,
            "replication_result": replication_result,
            "recovery_monitoring_started": True,
            "estimated_replication_hours": estimated_duration_hours,
            "alert_sent_if_slow": estimated_duration_hours > 1.0,
        }
        return result

    def _periodic_health_recovery(self, node_id):
        while True:
            time.sleep(self.RECOVERY_CHECK_INTERVAL_SECONDS)
            try:
                is_healthy = self.object_storage.health_check(node_id, timeout=self.HEALTH_CHECK_TIMEOUT_SECONDS)
                if is_healthy:
                    current_status_row = self.metadata_db.query(
                        "SELECT status FROM storage_nodes WHERE node_id = %s",
                        (node_id,)
                    )
                    if current_status_row and current_status_row[0]["status"] == StorageNodeStatus.OFFLINE.value:
                        self.metadata_db.execute(
                            "UPDATE storage_nodes SET status = %s WHERE node_id = %s",
                            (StorageNodeStatus.ONLINE.value, node_id)
                        )
                        self._rebalance_after_recovery(node_id)
                        self.alert_service.send_alert(
                            level="info",
                            message=f"Node {node_id} has recovered and is back online. Rebalancing initiated.",
                            component="MedicalImageStorageService",
                        )
                        del self._recovery_threads[node_id]
                        return
            except Exception:
                continue

    def _rebalance_after_recovery(self, recovered_node_id):
        online_nodes = self._get_online_storage_nodes()
        self._build_hash_ring(online_nodes)
        all_objects = self.metadata_db.query(
            "SELECT storage_key, node_id, checksum, size_bytes FROM object_locations"
        )
        rebalanced_count = 0
        for obj in all_objects:
            correct_nodes = self._get_nodes_for_key(obj["storage_key"], count=1)
            if correct_nodes and correct_nodes[0] == recovered_node_id and obj["node_id"] != recovered_node_id:
                try:
                    for shard_idx in range(self.ERASURE_CODING_DATA_SHARDS + self.ERASURE_CODING_PARITY_SHARDS):
                        shard_key = f"{obj['storage_key']}/shard_{shard_idx}"
                        data = self.object_storage.download(node_id=obj["node_id"], key=shard_key)
                        if data:
                            self.object_storage.upload(
                                node_id=recovered_node_id,
                                key=shard_key,
                                data=data,
                                metadata={"rebalanced": True}
                            )
                    self.metadata_db.execute(
                        "UPDATE object_locations SET node_id = %s WHERE storage_key = %s",
                        (recovered_node_id, obj["storage_key"])
                    )
                    rebalanced_count += 1
                except Exception:
                    continue
        return rebalanced_count
```

## 异常场景补充

### 场景：PACS 系统存储空间不足
```
trigger: DICOM 影像存储请求到达时，所有在线存储节点的可用空间均低于安全阈值（单节点可用 < 50GB 或集群总可用 < 200GB），或 object_storage.upload 抛出 NoSpaceLeftError 异常
detection: (1) 每次存储操作前检查 storage_nodes 表中各节点的 available_bytes 字段；(2) 定时巡检任务每 5 分钟扫描所有节点磁盘使用率；(3) upload 失败时捕获 NoSpaceLeftError 异常并记录告警
handling: (1) 立即暂停非紧急影像的自动存储队列，仅允许急诊影像写入；(2) 触发空间回收流程：删除已标记为过期的临时影像（status='expired' 的 object_locations），清理未完成的分片上传（shard_key 不在 manifest 中的孤立分片）；(3) 启动影像迁移：将超过 90 天未访问的冷影像迁移至低成本归档存储节点（status='archive' 的 storage_nodes），迁移完成后更新 object_locations 表的 node_id；(4) 如果回收和迁移后空间仍不足，向 PACS 管理员发送 P0 级告警，包含当前使用率、预计耗尽时间和建议扩容方案；(5) 对于急诊影像写入，启用单节点过载写入模式（允许单节点使用率达 95%），但标记该节点为 DEGRADED 状态
prevention: (1) 设置存储容量三级预警：70% 黄色预警（通知管理员）、85% 橙色预警（自动触发冷数据归档迁移）、95% 红色预警（停止非急诊写入）；(2) 每日自动预测存储增长趋势，基于过去 30 天的日均增量和当前剩余空间计算预计耗尽日期；(3) 采购流程中设置存储扩容审批绿色通道，确保从预警到扩容上线不超过 48 小时；(4) 实施影像生命周期管理策略，自动将超期影像归档或清理
```

### 场景：DICOM 传输中断导致数据不完整
```
trigger: 网络波动、传输超时或客户端异常断开导致 DICOM 影像实例传输中途中断，表现为：(1) 已传输的实例数少于预期总数；(2) 单个实例的 pixel_data 字节数与 DICOM 头声明的 Rows*Columns*BitsAllocated/8 不匹配；(3) SHA256 校验和不一致
detection: (1) 传输完成后逐实例校验 pixel_data 长度与 DICOM 元数据声明的预期大小是否一致；(2) 比对客户端计算的流式校验和与服务端存储后的 SHA256 校验和；(3) 检查 series_manifests 中的 instance_count 是否与实际存储到 object_locations 的实例数匹配；(4) 传输过程中监控 socket 超时和连接断开事件
handling: (1) 记录中断点信息：已成功传输的 SOP Instance UID 列表和失败实例列表，写入 transfer_recovery 表（study_uid, series_uid, completed_sop_uids, failed_sop_uids, interrupted_at, client_info）；(2) 对已存储但不完整的实例执行清理：从 object_storage 删除不完整的 shard 文件，从 object_locations 删除对应记录；(3) 生成断点续传清单，仅包含未成功传输的实例，发送给客户端请求重传；(4) 设置恢复超时（默认 2 小时），超时未恢复则标记该 series 为 'transfer_incomplete'，通知影像科室；(5) 重传时使用增量传输模式，跳过已校验通过的实例，仅传输中断实例；(6) 如果同一 study 连续中断 3 次以上，切换为逐实例单连接传输模式（牺牲速度换稳定性）
prevention: (1) 传输前协商 QoS 策略：为 DICOM 传输预留带宽，设置 DSCP 优先级标记；(2) 实施分片传输确认机制：每个 shard 传输完成后立即校验并 ACK，未 ACK 的 shard 自动重传（最多 3 次）；(3) 客户端实现本地暂存：传输前将影像完整缓存到本地临时目录，避免源端在传输过程中不可用；(4) 对大于 500MB 的 series 强制启用断点续传模式；(5) 部署网络质量监控探针，在链路质量低于阈值时主动降速或暂停传输
```

### 场景：存储节点故障影响在线阅片
```
trigger: 正在为医生提供在线阅片服务的存储节点发生故障（硬件损坏、网络分区、进程崩溃），导致 retrieve_dicom_study 请求返回 ObjectNotFoundError 或连接超时
detection: (1) 在线阅片请求的实时错误率监控：1 分钟内同一节点的读取失败率超过 5% 触发告警；(2) 健康检查线程检测到节点连续 3 次 5 秒超时；(3) Redis 路由表中节点状态变为 offline；(4) 阅片前端报告影像加载失败（超时或 404 错误）超过 3 次触发用户端告警
handling: (1) 立即触发 handle_storage_node_failure 标记节点离线并启动数据复制；(2) 对正在进行的阅片会话实施紧急切换：从副本节点或通过纠删码重建读取影像数据，优先恢复当前正在查看的 series；(3) 如果副本节点数据可用，将阅片请求路由到副本节点（通过更新 Redis 路由表 route:{study_uid} → 新节点）；(4) 如果需要纠删码重建，先提供缩略图和低分辨率预览确保阅片不中断，后台异步重建完整数据；(5) 向当前阅片医生推送通知，说明可能有短暂延迟，建议优先查看已加载的影像；(6) 记录故障期间的阅片请求日志，用于事后审计确认无遗漏诊断
prevention: (1) 确保每个影像 series 至少 2 个副本分布在不同机架的节点上（跨机架感知的副本放置策略）；(2) 阅片服务实现本地缓存层：最近 24 小时内访问过的影像自动缓存到阅片服务器的 SSD 缓存，节点故障时直接从本地缓存提供；(3) 部署预加载策略：当医生打开患者列表时，后台预先加载该患者最近一次检查的影像到缓存；(4) 实施阅片服务的降级方案：故障时提供缩略图浏览模式（64x64 分辨率），确保基本阅片能力不中断；(5) 存储集群跨可用区部署，单可用区故障不影响服务连续性
```
