# C14: 内容审核的多级流水线

## 业务场景

某社交平台日均产生 500 万条 UGC 内容（文本、图片、视频），需要审核后公开展示。审核目标：违规内容漏检率 < 0.1%，误杀率 < 1%。

**核心矛盾**：审核速度（毫秒级）与准确性（99.9%检出率）之间的张力。AI 快但不精准，人工精准但慢。

**已知数据：**
- 日均新增内容：500 万条（文本 60%，图片 30%，视频 10%）
- 峰值 QPS：2000 条/秒
- 违规率：约 3%（其中色情 0.5%、暴力 0.8%、广告 1.5%、政治 0.5%、辱骂 0.2%）
- AI 审核延迟：文本 < 100ms，图片 < 500ms，视频 < 30s
- 人工审核延迟：5-30 分钟
- 人工审核员：50 人，产能 1 万条/小时

**为什么不是简单的 AI 判定通过/拒绝？**

- AI 置信度 < 90% 的内容需要人工复审——约 20% 的内容需人工介入
- 新型违规（新诈骗话术、新色情变体）AI 无法识别 → 必须有人工兜底
- 误杀的代价（用户流失）vs 漏检的代价（监管处罚）的权衡——不同违规类型代价差异大

## 核心挑战

### 挑战 1：AI+人工混合审核的流水线编排

AI 毫秒级，人工分钟级。如何编排使高置信度内容秒级通过，低置信度内容进入人工队列不积压？

### 挑战 2：视频审核的延迟

5 分钟视频抽帧+AI 分析需要 30 秒。30 秒内不可见 → 用户体验差。但不审核就展示 → 违规视频已传播。

### 挑战 3：误杀与漏检的权衡

| 违规类型 | 漏检代价 | 误杀代价 | 阈值策略 |
|---------|---------|---------|---------|
| 政治 | 极高（监管处罚） | 低（内容少） | 更敏感（阈值低） |
| 色情 | 高（下架风险） | 中 | 敏感 |
| 暴力 | 高 | 中 | 敏感 |
| 广告 | 低 | 高（正常推广被删） | 更宽松 |
| 辱骂 | 低 | 高（正常吐槽被删） | 更宽松 |

### 挑战 4：人工审核积压

日均违规约 15 万条，其中 20%（3 万条）需人工。50 人×200条/小时 = 1 万条/小时 → 日产能 8 万条。大促期间可能积压。

## 设计约束

- 所有内容必须审核后公开展示（先审后发）
- 人工审核积压 < 2 小时
- 审核结果可追溯
- 规则更新即时生效

## 请先独立思考（限时 30 分钟）

1. 多级审核流水线如何设计？规则引擎 → AI → 人工，各级的职责边界？
2. 视频审核如何平衡延迟和安全？先发后审的条件是什么？
3. 人工审核积压时如何降级？降级的代价是什么？
4. 不同违规类型的阈值如何差异化？

---

## 设计解析

### 数据库设计

```sql
-- 内容表
CREATE TABLE contents (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    content_id VARCHAR(64) NOT NULL,
    user_id VARCHAR(32) NOT NULL,
    content_type VARCHAR(10) NOT NULL,     -- text / image / video
    content_text TEXT,
    content_url VARCHAR(500),               -- 图片/视频的存储URL
    publish_status VARCHAR(20) DEFAULT 'pending', -- pending / approved / rejected / reviewing
    publish_strategy VARCHAR(20) DEFAULT 'review_first', -- review_first / publish_first
    created_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE KEY uk_content (content_id),
    INDEX idx_user_time (user_id, created_at),
    INDEX idx_status_time (publish_status, created_at)
);

-- 审核结果表
CREATE TABLE moderation_results (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    content_id VARCHAR(64) NOT NULL,
    stage VARCHAR(20) NOT NULL,            -- rule / ai / human
    is_violation BOOLEAN,                  -- TRUE / FALSE / NULL(不确定)
    reason VARCHAR(20),                    -- politics / porn / violence / ad / abuse
    confidence DECIMAL(4,3),               -- AI 置信度 0.000-1.000
    all_scores JSONB,                      -- AI 各标签得分
    reviewer_id VARCHAR(32),               -- 人工审核员ID
    reviewed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_content (content_id),
    INDEX idx_stage_result (stage, is_violation)
);

-- 人工审核队列
CREATE TABLE review_queue (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    content_id VARCHAR(64) NOT NULL,
    content_type VARCHAR(10) NOT NULL,
    ai_confidence DECIMAL(4,3),
    ai_reason VARCHAR(20),
    priority INT NOT NULL DEFAULT 50,
    status VARCHAR(20) DEFAULT 'pending',  -- pending / locked / completed / expired
    locked_by VARCHAR(32),                 -- 审核员ID（锁定中）
    locked_at TIMESTAMP,
    enqueued_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP,
    
    INDEX idx_status_priority (status, priority DESC, enqueued_at ASC),
    INDEX idx_reviewer (locked_by, status)
);

-- 用户信任分
CREATE TABLE user_trust (
    user_id VARCHAR(32) PRIMARY KEY,
    trust_score DECIMAL(4,3) DEFAULT 0.5,  -- 0-1，越高越可信
    violation_count INT DEFAULT 0,
    content_count INT DEFAULT 0,
    account_age_days INT DEFAULT 0,
    updated_at TIMESTAMP DEFAULT NOW()
);

-- 用户申诉
CREATE TABLE appeals (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    content_id VARCHAR(64) NOT NULL,
    user_id VARCHAR(32) NOT NULL,
    appeal_reason TEXT,
    appeal_status VARCHAR(20) DEFAULT 'pending', -- pending / approved / rejected
    reviewed_by VARCHAR(32),
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_content (content_id),
    INDEX idx_status (appeal_status, created_at)
);
```

### 多级审核流水线

```
用户发布内容 → 第一级：规则过滤（黑名单/正则）→ 命中 → 直接拒绝
                                    → 未命中 ↓
               → 第二级：AI审核 → 高置信违规 → 拒绝
                                 → 高置信正常 → 通过
                                 → 低置信 ↓
               → 第三级：人工审核 → 违规/正常
```

**各级处理量和延迟：**

| 级别 | 处理占比 | 延迟 | 误判率 |
|------|---------|------|-------|
| 第一级：规则引擎 | 10%（命中拒绝） | < 1ms | < 0.1% |
| 第二级：AI审核 | 70%（自动通过/拒绝） | 100-500ms | 约 2% |
| 第三级：人工审核 | 20%（需复审） | 5-30min | < 0.5% |

### 第一级：规则引擎

```python
class RuleFilter:
    def __init__(self):
        self.ac_automaton = self.build_ac_automaton(self.load_blacklist())  # 10万关键词
        self.regex_rules = self.load_regex_rules()

    def check(self, content):
        # 黑名单词匹配（AC 自动机，O(N)扫描）
        for word in self.ac_automaton.search(content.text):
            return RuleResult(hit=True, reason=f"blacklist:{word}", level=1)

        # 正则匹配（手机号、微信号、URL）
        for rule in self.regex_rules:
            if re.search(rule.pattern, content.text):
                return RuleResult(hit=True, reason=f"regex:{rule.name}", level=1)

        return RuleResult(hit=False)

# 性能：10万关键词 AC 自动机扫描，延迟 < 1ms
# 命中率：约 10%（大部分违规内容不在黑名单中）
```

### 第二级：AI 审核

```python
class AIModerator:
    def check(self, content):
        if content.type == "text":
            return self.text_moderation(content)
        elif content.type == "image":
            return self.image_moderation(content)
        elif content.type == "video":
            return self.video_moderation(content)

    def text_moderation(self, content):
        """文本多标签分类"""
        scores = self.nlp_model.predict(content.text)
        # scores: {"porn": 0.01, "violence": 0.02, "politics": 0.85, "ad": 0.05}

        # 差异化阈值：政治敏感更敏感，广告更宽松
        thresholds = {
            "politics": 0.5,   # 政治敏感：0.5 以上即判违规
            "porn": 0.6,       # 色情：0.6
            "violence": 0.6,   # 暴力：0.6
            "ad": 0.8,         # 广告：0.8（避免误杀正常推广）
            "abuse": 0.8,      # 辱骂：0.8（避免误杀正常吐槽）
        }

        for label, score in scores.items():
            threshold = thresholds.get(label, 0.7)
            if score > threshold:
                return AIModerationResult(
                    is_violation=True, reason=label,
                    confidence=score, all_scores=scores
                )

        # 最高分在 0.5-0.8 之间 → 低置信 → 进人工
        max_label = max(scores, key=scores.get)
        max_score = scores[max_label]
        if max_score > 0.5:
            return AIModerationResult(
                is_violation=None,  # 不确定
                reason=max_label, confidence=max_score,
                all_scores=scores
            )

        return AIModerationResult(
            is_violation=False, reason=None,
            confidence=1-max_score, all_scores=scores
        )

    def image_moderation(self, content):
        """图片多模型并行"""
        results = {}
        with ThreadPoolExecutor() as executor:
            futures = {
                "nudity": executor.submit(self.nudity_model.predict, content.image_url),
                "violence": executor.submit(self.violence_model.predict, content.image_url),
                "ocr": executor.submit(self.ocr_model.predict, content.image_url),
            }
            for label, future in futures.items():
                results[label] = future.result()

        # OCR 检测的文本也需要文本审核
        if results.get("ocr"):
            text_result = self.text_moderation(Text(content=results["ocr"]))
            if text_result.is_violation:
                return text_result

        max_label = max(results, key=results.get)
        max_score = results[max_label]
        return AIModerationResult(
            is_violation=max_score > 0.6,
            reason=max_label if max_score > 0.6 else None,
            confidence=max_score,
            all_scores=results
        )

    def video_moderation(self, content):
        """视频抽帧 + 图片审核 + 音频审核"""
        frames = self.extract_keyframes(content.video_url, interval=10)

        max_score = 0
        violation_label = None
        for frame in frames:
            result = self.image_moderation(Image(content=frame))
            if result.confidence > max_score and result.is_violation:
                max_score = result.confidence
                violation_label = result.reason

        # 音频审核（语音转文字 + 文本审核）
        audio_text = self.speech_to_text(content.video_url)
        if audio_text:
            audio_result = self.text_moderation(Text(content=audio_text))
            if audio_result.confidence > max_score and audio_result.is_violation:
                max_score = audio_result.confidence
                violation_label = audio_result.reason

        return AIModerationResult(
            is_violation=max_score > 0.5,
            reason=violation_label,
            confidence=max_score
        )
```

### 完整视频审核流水线实现

视频审核是整个系统中最复杂的环节，涉及帧提取、多模态并行处理、结果聚合等多个步骤。以下为完整的生产级实现。

```python
import subprocess
import os
import asyncio
import time
from concurrent.futures import ThreadPoolExecutor, as_completed
from dataclasses import dataclass, field
from typing import List, Optional, Dict, Tuple
from enum import Enum


class VideoModerationStatus(Enum):
    PENDING = "pending"
    EXTRACTING = "extracting"
    ANALYZING = "analyzing"
    AGGREGATING = "aggregating"
    COMPLETED = "completed"
    FAILED = "failed"


@dataclass
class FrameResult:
    """单帧审核结果"""
    frame_index: int
    timestamp_ms: int
    is_violation: bool
    reason: Optional[str]
    confidence: float
    all_scores: Dict[str, float]
    latency_ms: int


@dataclass
class VideoModerationResult:
    """视频审核最终结果"""
    content_id: str
    is_violation: Optional[bool]       # True=违规, False=正常, None=不确定
    reason: Optional[str]
    confidence: float
    violation_frames: List[FrameResult] = field(default_factory=list)
    frame_count: int = 0
    total_latency_ms: int = 0
    audio_result: Optional[dict] = None
    status: VideoModerationStatus = VideoModerationStatus.PENDING


class FFmpegFrameExtractor:
    """FFmpeg 关键帧提取器——从视频中均匀抽帧用于审核"""

    def __init__(self, ffmpeg_path: str = "/usr/bin/ffmpeg",
                 ffprobe_path: str = "/usr/bin/ffprobe",
                 temp_dir: str = "/tmp/video_frames"):
        self.ffmpeg_path = ffmpeg_path
        self.ffprobe_path = ffprobe_path
        self.temp_dir = temp_dir
        os.makedirs(temp_dir, exist_ok=True)

    def get_video_info(self, video_url: str) -> Dict:
        """获取视频元信息（时长、分辨率、帧率）"""
        cmd = [
            self.ffprobe_path,
            "-v", "quiet",
            "-print_format", "json",
            "-show_format", "-show_streams",
            video_url
        ]
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=30)
        import json
        info = json.loads(result.stdout)

        video_stream = next(
            (s for s in info.get("streams", []) if s["codec_type"] == "video"), None
        )
        if not video_stream:
            raise ValueError(f"未找到视频流: {video_url}")

        duration = float(info["format"].get("duration", 0))
        width = int(video_stream.get("width", 0))
        height = int(video_stream.get("height", 0))
        fps = eval(video_stream.get("r_frame_rate", "30/1"))  # 如 "30000/1001"

        return {
            "duration": duration,
            "width": width,
            "height": height,
            "fps": fps,
            "frame_count": int(duration * fps),
        }

    def extract_keyframes(self, video_url: str, content_id: str,
                           strategy: str = "adaptive") -> List[Dict]:
        """
        抽取关键帧

        策略说明：
        - uniform: 均匀间隔抽帧（简单但可能漏掉关键画面）
        - scene: 场景变化检测抽帧（FFmpeg scene filter，保留画面跳变）
        - adaptive: 根据视频时长动态调整（短视频密集，长视频稀疏）
        """
        video_info = self.get_video_info(video_url)
        duration = video_info["duration"]

        # 根据时长决定抽帧间隔
        if strategy == "adaptive":
            if duration <= 30:
                interval_sec = 2      # 短视频：每2秒一帧
            elif duration <= 120:
                interval_sec = 5      # 中等视频：每5秒一帧
            elif duration <= 600:
                interval_sec = 10     # 5-10分钟：每10秒
            else:
                interval_sec = 15    # 长视频：每15秒
        elif strategy == "uniform":
            interval_sec = 10
        elif strategy == "scene":
            return self._extract_scene_change_frames(video_url, content_id)
        else:
            interval_sec = 10

        # 计算总帧数并限制上限（避免超长视频产生过多帧）
        total_frames = int(duration / interval_sec) + 1
        max_frames = 60  # 最多60帧
        if total_frames > max_frames:
            interval_sec = duration / max_frames
            total_frames = max_frames

        output_dir = os.path.join(self.temp_dir, content_id)
        os.makedirs(output_dir, exist_ok=True)

        # FFmpeg 抽帧命令
        output_pattern = os.path.join(output_dir, "frame_%04d.jpg")
        cmd = [
            self.ffmpeg_path,
            "-i", video_url,
            "-vf", f"fps=1/{interval_sec},scale=640:-1",  # 统一缩放至640宽
            "-q:v", "2",         # JPEG 质量（2=高质量）
            "-y",                # 覆盖已存在文件
            output_pattern
        ]

        start_time = time.time()
        subprocess.run(cmd, capture_output=True, text=True, timeout=120)
        extract_latency = int((time.time() - start_time) * 1000)

        # 收集帧信息
        frames = []
        for fname in sorted(os.listdir(output_dir)):
            if fname.endswith(".jpg"):
                frame_path = os.path.join(output_dir, fname)
                frame_index = int(fname.replace("frame_", "").replace(".jpg", ""))
                timestamp_ms = int((frame_index - 1) * interval_sec * 1000)
                frames.append({
                    "frame_index": frame_index,
                    "timestamp_ms": timestamp_ms,
                    "file_path": frame_path,
                    "url": f"file://{frame_path}",
                })

        return frames, extract_latency, video_info

    def _extract_scene_change_frames(self, video_url: str,
                                      content_id: str) -> Tuple[List[Dict], int, Dict]:
        """基于场景变化检测抽帧——画面跳变时自动抽帧"""
        output_dir = os.path.join(self.temp_dir, content_id)
        os.makedirs(output_dir, exist_ok=True)

        output_pattern = os.path.join(output_dir, "scene_%04d.jpg")
        # scene=0.3 表示画面变化超过30%时触发抽帧
        cmd = [
            self.ffmpeg_path,
            "-i", video_url,
            "-vf", f"select='gt(scene,0.3)',scale=640:-1",
            "-vsync", "vfr",
            "-q:v", "2",
            "-y",
            output_pattern
        ]

        start_time = time.time()
        subprocess.run(cmd, capture_output=True, text=True, timeout=120)
        extract_latency = int((time.time() - start_time) * 1000)

        frames = []
        for fname in sorted(os.listdir(output_dir)):
            if fname.endswith(".jpg"):
                frame_path = os.path.join(output_dir, fname)
                frames.append({
                    "frame_index": len(frames) + 1,
                    "timestamp_ms": 0,  # 场景检测模式时间戳需额外解析
                    "file_path": frame_path,
                    "url": f"file://{frame_path}",
                })

        video_info = self.get_video_info(video_url)
        return frames, extract_latency, video_info

    def cleanup(self, content_id: str):
        """清理临时帧文件"""
        output_dir = os.path.join(self.temp_dir, content_id)
        if os.path.exists(output_dir):
            for f in os.listdir(output_dir):
                os.remove(os.path.join(output_dir, f))
            os.rmdir(output_dir)


class VideoModerationPipeline:
    """视频审核完整流水线：帧提取 → 并行 AI 分析 → 结果聚合"""

    def __init__(self, frame_extractor: FFmpegFrameExtractor,
                 ai_moderator, speech_to_text_fn,
                 max_parallel_frames: int = 8):
        self.frame_extractor = frame_extractor
        self.ai_moderator = ai_moderator
        self.speech_to_text = speech_to_text_fn
        self.max_parallel_frames = max_parallel_frames

    def moderate(self, content) -> VideoModerationResult:
        """执行视频审核完整流程"""
        result = VideoModerationResult(content_id=content.content_id)
        pipeline_start = time.time()

        try:
            # ---- 阶段1：关键帧提取 ----
            result.status = VideoModerationStatus.EXTRACTING
            frames, extract_latency, video_info = self.frame_extractor.extract_keyframes(
                content.content_url, content.content_id, strategy="adaptive"
            )
            result.frame_count = len(frames)

            # ---- 阶段2：并行 AI 分析（帧 + 音频同时进行） ----
            result.status = VideoModerationStatus.ANALYZING
            frame_results, audio_result = self._parallel_analyze(
                frames, content.content_url
            )

            # ---- 阶段3：结果聚合 ----
            result.status = VideoModerationStatus.AGGREGATING
            aggregated = self._aggregate_results(frame_results, audio_result)

            result.is_violation = aggregated["is_violation"]
            result.reason = aggregated["reason"]
            result.confidence = aggregated["confidence"]
            result.violation_frames = aggregated["violation_frames"]
            result.audio_result = audio_result

            result.status = VideoModerationStatus.COMPLETED

        except Exception as e:
            result.status = VideoModerationStatus.FAILED
            result.is_violation = None  # 不确定 → 进人工审核
            result.reason = f"pipeline_error: {str(e)}"

        finally:
            result.total_latency_ms = int((time.time() - pipeline_start) * 1000)
            self.frame_extractor.cleanup(content.content_id)

        return result

    def _parallel_analyze(self, frames: List[Dict],
                          video_url: str) -> Tuple[List[FrameResult], Optional[dict]]:
        """帧审核与音频审核并行执行"""
        frame_results = []
        audio_result = None

        with ThreadPoolExecutor(max_workers=self.max_parallel_frames + 1) as executor:
            # 并行提交所有帧的图片审核
            frame_futures = {}
            for frame in frames:
                future = executor.submit(self._analyze_single_frame, frame)
                frame_futures[future] = frame

            # 同时提交音频审核
            audio_future = executor.submit(self._analyze_audio, video_url)

            # 收集帧审核结果
            for future in as_completed(frame_futures):
                try:
                    frame_result = future.result()
                    frame_results.append(frame_result)
                except Exception as e:
                    frame = frame_futures[future]
                    # 单帧审核失败不中断整体流程，记录为低置信度
                    frame_results.append(FrameResult(
                        frame_index=frame["frame_index"],
                        timestamp_ms=frame["timestamp_ms"],
                        is_violation=False,
                        reason=None,
                        confidence=0.0,
                        all_scores={},
                        latency_ms=0
                    ))

            # 收集音频审核结果
            try:
                audio_result = audio_future.result()
            except Exception:
                audio_result = None

        # 按帧序号排序
        frame_results.sort(key=lambda x: x.frame_index)
        return frame_results, audio_result

    def _analyze_single_frame(self, frame: Dict) -> FrameResult:
        """单帧图片审核"""
        start = time.time()
        img_result = self.ai_moderator.image_moderation(
            type("Image", (), {"content": frame["url"]})()
        )
        latency_ms = int((time.time() - start) * 1000)

        return FrameResult(
            frame_index=frame["frame_index"],
            timestamp_ms=frame["timestamp_ms"],
            is_violation=img_result.is_violation or False,
            reason=img_result.reason,
            confidence=img_result.confidence,
            all_scores=img_result.all_scores,
            latency_ms=latency_ms
        )

    def _analyze_audio(self, video_url: str) -> Optional[dict]:
        """音频审核：语音转文字 → 文本审核"""
        audio_text = self.speech_to_text(video_url)
        if not audio_text:
            return None

        text_result = self.ai_moderator.text_moderation(
            type("Text", (), {"text": audio_text})()
        )
        return {
            "transcribed_text": audio_text[:200],  # 仅保留前200字符
            "is_violation": text_result.is_violation,
            "reason": text_result.reason,
            "confidence": text_result.confidence,
        }

    def _aggregate_results(self, frame_results: List[FrameResult],
                           audio_result: Optional[dict]) -> Dict:
        """
        结果聚合策略

        规则：
        1. 任一帧高置信违规 → 整体判定违规（取最高置信度的帧）
        2. 多帧低置信违规 → 聚合置信度提升（连续3帧低置信 → 视为高置信）
        3. 音频违规与视觉违规独立判定，取更严重的结果
        4. 无任何违规信号 → 通过
        5. 存在低置信信号但不足以判定 → 不确定（进人工）
        """
        # 收集违规帧
        violation_frames = [f for f in frame_results if f.is_violation]
        uncertain_frames = [
            f for f in frame_results
            if not f.is_violation and f.confidence > 0.4
        ]

        # 视觉维度聚合
        visual_violation = None
        visual_confidence = 0.0
        visual_reason = None

        if violation_frames:
            # 取置信度最高的违规帧
            best = max(violation_frames, key=lambda f: f.confidence)
            visual_violation = True
            visual_confidence = best.confidence
            visual_reason = best.reason

        elif len(uncertain_frames) >= 3:
            # 连续3帧以上低置信 → 提升置信度
            # 聚合公式：combined = 1 - prod(1 - score_i)
            combined = 1.0
            for f in uncertain_frames[:5]:  # 最多取5帧
                combined *= (1 - f.confidence)
            combined = 1 - combined

            if combined > 0.6:
                visual_violation = True
                visual_confidence = combined
                visual_reason = max(uncertain_frames,
                                   key=lambda f: f.confidence).reason
            else:
                visual_violation = None  # 不确定
                visual_confidence = combined
                visual_reason = uncertain_frames[0].reason

        elif len(uncertain_frames) > 0:
            visual_violation = None  # 不确定
            visual_confidence = max(f.confidence for f in uncertain_frames)
            visual_reason = uncertain_frames[0].reason

        # 音频维度
        audio_violation = None
        audio_confidence = 0.0
        audio_reason = None
        if audio_result:
            audio_violation = audio_result["is_violation"]
            audio_confidence = audio_result.get("confidence", 0)
            audio_reason = audio_result.get("reason")

        # 跨模态聚合：取更严重的结果
        # 优先级：违规(True) > 不确定(None) > 正常(False)
        def severity(violation, confidence):
            if violation is True:
                return (2, confidence)
            elif violation is None:
                return (1, confidence)
            else:
                return (0, confidence)

        candidates = [
            (visual_violation, visual_confidence, visual_reason, "visual"),
            (audio_violation, audio_confidence, audio_reason, "audio"),
        ]
        best = max(candidates, key=lambda c: (severity(c[0], c[1])))

        final_violation = best[0]
        final_confidence = best[1]
        final_reason = best[2]

        return {
            "is_violation": final_violation,
            "reason": final_reason,
            "confidence": final_confidence,
            "violation_frames": violation_frames,
            "visual_confidence": visual_confidence,
            "audio_confidence": audio_confidence,
        }
```

**视频审核流水线延迟分解（5 分钟视频示例）：**

| 步骤 | 操作 | 延迟 | 并行性 |
|------|------|------|-------|
| 1. 获取视频元信息 | ffprobe | 500ms | - |
| 2. 关键帧提取 | ffmpeg 抽30帧 | 3-5s | - |
| 3. 帧审核（并行） | 图片模型推理 | 500ms/帧 × 30帧 / 8并发 = 2s | 8路并发 |
| 4. 音频审核（并行） | ASR + 文本审核 | 3-4s | 与帧审核并行 |
| 5. 结果聚合 | 内存计算 | < 10ms | - |
| **总计** | | **5-8s** | |

```python
# 使用示例
extractor = FFmpegFrameExtractor()
ai_mod = AIModerator()
pipeline = VideoModerationPipeline(extractor, ai_mod, speech_to_text_fn=whisper_api)

content = Content(content_id="v_20240101_001", content_url="https://cdn.example.com/video/xxx.mp4")
result = pipeline.moderate(content)

print(f"审核结果: is_violation={result.is_violation}, reason={result.reason}")
print(f"置信度: {result.confidence:.3f}, 违规帧数: {len(result.violation_frames)}")
print(f"总耗时: {result.total_latency_ms}ms")
```

### 人工审核队列

```python
class HumanReviewQueue:
    def enqueue(self, content, ai_result):
        priority = self.calculate_priority(ai_result)
        self.db.execute("""
            INSERT INTO review_queue 
            (content_id, content_type, ai_confidence, ai_reason, priority, enqueued_at)
            VALUES (%s, %s, %s, %s, %s, NOW())
        """, content.id, content.type, ai_result.confidence, ai_result.reason, priority)

    def calculate_priority(self, ai_result):
        """优先级：政治 > 色情 > 暴力 > 广告 > 辱骂"""
        priority_map = {"politics": 100, "porn": 80, "violence": 60, "ad": 40, "abuse": 20}
        return priority_map.get(ai_result.reason, 10) + ai_result.confidence * 50

    def get_next_batch(self, reviewer_id, count=20):
        """审核员获取待审内容（并发安全：FOR UPDATE SKIP LOCKED）"""
        # 开启事务
        with self.db.transaction():
            # 锁定行并跳过已被其他审核员锁定的行
            items = self.db.query("""
                SELECT * FROM review_queue
                WHERE status = 'pending'
                ORDER BY priority DESC, enqueued_at ASC
                LIMIT %s FOR UPDATE SKIP LOCKED
            """, count)

            if not items:
                return []

            # 批量标记为锁定
            ids = [item["id"] for item in items]
            self.db.execute("""
                UPDATE review_queue
                SET status = 'locked', locked_by = %s, locked_at = NOW()
                WHERE id IN (%s)
            """ % (reviewer_id, ",".join(str(id) for id in ids)))

            return items

    def submit_review(self, reviewer_id, content_id, is_violation, reason=None):
        """审核员提交审核结果"""
        with self.db.transaction():
            # 更新审核队列
            self.db.execute("""
                UPDATE review_queue
                SET status = 'completed', completed_at = NOW()
                WHERE content_id = %s AND locked_by = %s AND status = 'locked'
            """, content_id, reviewer_id)

            if self.db.affected_rows == 0:
                raise ReviewConflictError("该内容已被其他审核员处理或锁定已过期")

            # 记录审核结果
            self.db.insert("moderation_results", {
                "content_id": content_id,
                "stage": "human",
                "is_violation": is_violation,
                "reason": reason,
                "reviewer_id": reviewer_id,
                "reviewed_at": now()
            })

            # 更新内容发布状态
            if is_violation:
                self.db.update("contents",
                    {"publish_status": "rejected"},
                    {"content_id": content_id})
            else:
                self.db.update("contents",
                    {"publish_status": "approved"},
                    {"content_id": content_id})

    def release_expired_locks(self):
        """释放超时锁（审核员锁定超过30分钟未提交 → 释放回队列）"""
        self.db.execute("""
            UPDATE review_queue
            SET status = 'pending', locked_by = NULL, locked_at = NULL
            WHERE status = 'locked' AND locked_at < NOW() - INTERVAL 30 MINUTE
        """)
        released = self.db.affected_rows
        if released > 0:
            self.log.info(f"Released {released} expired review locks")
```

### 人工审核队列管理（优先级评分 + SLA 追踪 + 负载均衡）

人工审核队列是 AI 审核的下游缓冲区，管理效率直接决定了审核 SLA 是否达标。以下实现覆盖优先级评分、SLA 倒计时追踪、审核员负载均衡等完整能力。

```python
from datetime import datetime, timedelta
from typing import List, Dict, Optional, Tuple
from dataclasses import dataclass, field
import math


@dataclass
class SLAPolicy:
    """SLA 策略：不同违规类型有不同的审核时效要求"""
    # 违规类型 → 最大等待时间（分钟）
    max_wait_by_reason = {
        "politics": 15,    # 政治敏感：15分钟内必须审
        "porn": 30,        # 色情：30分钟
        "violence": 30,    # 暴力：30分钟
        "ad": 60,          # 广告：1小时
        "abuse": 60,       # 辱骂：1小时
    }
    default_max_wait = 120  # 默认2小时


@dataclass
class ReviewerProfile:
    """审核员画像——用于负载均衡和任务分配"""
    reviewer_id: str
    skill_tags: List[str]          # 擅长类型: ["politics", "porn"]
    shift_start: datetime          # 当班开始时间
    shift_end: datetime            # 当班结束时间
    items_reviewed_today: int = 0  # 今日已审数量
    avg_review_time_sec: float = 30.0  # 平均审核耗时（秒/条）
    accuracy_rate: float = 0.98    # 审核准确率
    current_load: int = 0          # 当前锁定待审数量
    max_concurrent: int = 20       # 最大并发锁定数
    last_active_at: datetime = None


class ReviewQueueManager:
    """
    人工审核队列管理器

    三大核心能力：
    1. 优先级评分——综合违规类型、AI置信度、等待时间、用户等级计算优先级
    2. SLA 追踪——实时监控每条内容的审核时效，超时前自动提升优先级
    3. 审核员负载均衡——按技能匹配和工作量均衡分配任务
    """

    def __init__(self, db, redis, alert_service, audit_logger):
        self.db = db
        self.redis = redis
        self.alert = alert_service
        self.audit = audit_logger
        self.sla_policy = SLAPolicy()
        # 审核员在线状态（从 Redis 心跳维护）
        self.reviewer_cache_key = "moderation:reviewers:online"
        # SLA 统计 Key
        self.sla_stats_key = "moderation:sla:stats"

    # ========== 1. 入队与优先级评分 ==========

    def enqueue(self, content, ai_result, user_trust_score: float = 0.5):
        """内容入队，计算优先级并插入审核队列"""
        priority = self._calculate_priority(ai_result, user_trust_score)

        # 计算 SLA 截止时间
        max_wait = self.sla_policy.max_wait_by_reason.get(
            ai_result.reason, self.sla_policy.default_max_wait
        )
        sla_deadline = datetime.now() + timedelta(minutes=max_wait)

        self.db.execute("""
            INSERT INTO review_queue
            (content_id, content_type, ai_confidence, ai_reason,
             priority, sla_deadline, enqueued_at)
            VALUES (%s, %s, %s, %s, %s, %s, NOW())
        """, content.id, content.type, ai_result.confidence,
             ai_result.reason, priority, sla_deadline)

        # 更新 SLA 统计
        self._update_sla_stats("enqueued", ai_result.reason)

    def _calculate_priority(self, ai_result, user_trust_score: float) -> int:
        """
        优先级评分算法

        评分维度（满分200分）：
        - 违规类型权重（0-100）：政治100、色情80、暴力60、广告40、辱骂20
        - AI 置信度加成（0-50）：置信度越高，越可能是违规，优先审
        - 用户信任度加成（0-30）：低信任用户的违规可能性更高
        - 等待时间加成（动态）：每分钟 +1，最多 +20
        """
        # 违规类型权重
        type_weight = {
            "politics": 100, "porn": 80, "violence": 60,
            "ad": 40, "abuse": 20,
        }
        type_score = type_weight.get(ai_result.reason, 10)

        # AI 置信度加成（0.5-1.0 映射到 0-50）
        confidence_score = int(max(0, min(50, (ai_result.confidence - 0.5) * 100)))

        # 用户信任度加成（越不信任优先级越高：1.0-0.0 映射到 0-30）
        trust_score = int((1 - user_trust_score) * 30)

        # 基础优先级
        base_priority = type_score + confidence_score + trust_score

        return min(200, base_priority)

    def update_priority_for_aging(self):
        """
        定时任务：为等待时间过长的内容提升优先级（防止低优先级内容饿死）

        策略：每超过 SLA 的 50%，优先级 +10；超过 80%，优先级 +25；超过 SLA，优先级设为最高
        """
        items = self.db.query("""
            SELECT id, content_id, priority, ai_reason,
                   sla_deadline, enqueued_at,
                   TIMESTAMPDIFF(MINUTE, enqueued_at, NOW()) as wait_minutes
            FROM review_queue
            WHERE status = 'pending'
        """)

        for item in items:
            max_wait = self.sla_policy.max_wait_by_reason.get(
                item["ai_reason"], self.sla_policy.default_max_wait
            )
            wait_ratio = item["wait_minutes"] / max_wait

            if wait_ratio >= 1.0:
                # 已超 SLA → 最高优先级
                new_priority = 200
            elif wait_ratio >= 0.8:
                new_priority = item["priority"] + 25
            elif wait_ratio >= 0.5:
                new_priority = item["priority"] + 10
            else:
                continue  # 不需要调整

            new_priority = min(200, new_priority)
            if new_priority != item["priority"]:
                self.db.execute("""
                    UPDATE review_queue SET priority = %s WHERE id = %s
                """, new_priority, item["id"])

    # ========== 2. SLA 追踪 ==========

    def check_sla_compliance(self) -> Dict:
        """
        检查 SLA 达成情况

        返回各违规类型的 SLA 达成率、超时数量等指标
        """
        stats = {}
        for reason, max_wait in self.sla_policy.max_wait_by_reason.items():
            total = self.db.query_one("""
                SELECT COUNT(*) as cnt FROM review_queue
                WHERE ai_reason = %s AND status IN ('completed', 'expired')
                  AND enqueued_at > NOW() - INTERVAL 24 HOUR
            """, reason)

            on_time = self.db.query_one("""
                SELECT COUNT(*) as cnt FROM review_queue
                WHERE ai_reason = %s AND status = 'completed'
                  AND completed_at <= sla_deadline
                  AND enqueued_at > NOW() - INTERVAL 24 HOUR
            """, reason)

            overdue = self.db.query_one("""
                SELECT COUNT(*) as cnt FROM review_queue
                WHERE ai_reason = %s AND status = 'pending'
                  AND sla_deadline < NOW()
            """, reason)

            total_cnt = total["cnt"] or 1  # 避免除零
            stats[reason] = {
                "total": total["cnt"],
                "on_time": on_time["cnt"],
                "sla_rate": on_time["cnt"] / total_cnt,
                "overdue_pending": overdue["cnt"],
                "max_wait_min": max_wait,
            }

        return stats

    def get_urgent_items(self, limit: int = 50) -> List[Dict]:
        """获取即将超 SLA 的内容（优先分配给审核员）"""
        return self.db.query("""
            SELECT * FROM review_queue
            WHERE status = 'pending'
              AND sla_deadline < NOW() + INTERVAL 10 MINUTE
            ORDER BY sla_deadline ASC, priority DESC
            LIMIT %s
        """, limit)

    def _update_sla_stats(self, event: str, reason: str):
        """更新 SLA 统计到 Redis（用于实时仪表盘）"""
        key = f"{self.sla_stats_key}:{reason}"
        if event == "enqueued":
            self.redis.hincrby(key, "enqueued", 1)
        elif event == "completed_on_time":
            self.redis.hincrby(key, "completed_on_time", 1)
        elif event == "completed_late":
            self.redis.hincrby(key, "completed_late", 1)

    # ========== 3. 审核员负载均衡 ==========

    def get_next_batch_balanced(self, reviewer_id: str, count: int = 20) -> List[Dict]:
        """
        为审核员分配任务（负载均衡版本）

        分配策略：
        1. 按优先级+SLA紧急度排序
        2. 按审核员技能匹配过滤（擅长该类型的优先分配）
        3. 确保审核员当前负载不超过上限
        4. 同类型内容尽量集中分配（减少上下文切换）
        """
        # 检查审核员当前负载
        reviewer = self._get_reviewer_profile(reviewer_id)
        if reviewer.current_load >= reviewer.max_concurrent:
            return []

        available_count = min(count, reviewer.max_concurrent - reviewer.current_load)

        with self.db.transaction():
            # 先尝试分配审核员擅长类型的内容
            items = self._fetch_matching_items(
                reviewer.skill_tags, available_count
            )

            if len(items) < available_count:
                # 不够则补充其他类型
                extra = self._fetch_any_items(
                    available_count - len(items),
                    exclude_ids=[i["id"] for i in items]
                )
                items.extend(extra)

            if not items:
                return []

            # 批量锁定
            ids = [item["id"] for item in items]
            placeholders = ",".join(["%s"] * len(ids))
            self.db.execute(f"""
                UPDATE review_queue
                SET status = 'locked', locked_by = %s, locked_at = NOW()
                WHERE id IN ({placeholders})
            """, reviewer_id, *ids)

            # 更新审核员负载
            self.redis.hincrby(
                self.reviewer_cache_key, reviewer_id, len(items)
            )

        return items

    def submit_review(self, reviewer_id: str, content_id: str,
                      is_violation: bool, reason: str = None):
        """提交审核结果并更新 SLA 统计"""
        with self.db.transaction():
            item = self.db.query_one("""
                SELECT * FROM review_queue
                WHERE content_id = %s AND locked_by = %s AND status = 'locked'
            """, content_id, reviewer_id)

            if not item:
                raise ReviewConflictError("该内容已被其他审核员处理或锁定已过期")

            # 判断是否在 SLA 内完成
            is_on_time = datetime.now() <= item["sla_deadline"]

            # 更新队列状态
            self.db.execute("""
                UPDATE review_queue
                SET status = 'completed', completed_at = NOW()
                WHERE content_id = %s AND locked_by = %s AND status = 'locked'
            """, content_id, reviewer_id)

            # 更新 SLA 统计
            sla_event = "completed_on_time" if is_on_time else "completed_late"
            self._update_sla_stats(sla_event, item["ai_reason"])

            # 记录审核结果
            self.db.insert("moderation_results", {
                "content_id": content_id,
                "stage": "human",
                "is_violation": is_violation,
                "reason": reason,
                "reviewer_id": reviewer_id,
                "reviewed_at": datetime.now(),
            })

            # 更新内容发布状态
            new_status = "rejected" if is_violation else "approved"
            self.db.update("contents",
                {"publish_status": new_status},
                {"content_id": content_id})

            # 更新审核员负载
            self.redis.hincrby(self.reviewer_cache_key, reviewer_id, -1)

            # 审计日志
            self.audit.append(
                content_id=content_id,
                action="human_reject" if is_violation else "human_pass",
                actor=reviewer_id,
                actor_type="human",
                stage="human",
                before_status="reviewing",
                after_status=new_status,
                decision_detail={
                    "is_violation": is_violation,
                    "reason": reason,
                    "sla_on_time": is_on_time,
                    "wait_minutes": (datetime.now() - item["enqueued_at"]).total_seconds() / 60,
                },
            )

    def get_reviewer_dashboard(self, reviewer_id: str) -> Dict:
        """审核员工作台数据"""
        reviewer = self._get_reviewer_profile(reviewer_id)

        # 今日统计
        today_stats = self.db.query_one("""
            SELECT COUNT(*) as total,
                   SUM(CASE WHEN completed_at <= sla_deadline THEN 1 ELSE 0 END) as on_time,
                   AVG(TIMESTAMPDIFF(SECOND, locked_at, completed_at)) as avg_review_sec
            FROM review_queue
            WHERE locked_by = %s
              AND status = 'completed'
              AND DATE(completed_at) = CURDATE()
        """, reviewer_id)

        return {
            "reviewer_id": reviewer_id,
            "current_load": reviewer.current_load,
            "max_concurrent": reviewer.max_concurrent,
            "today_reviewed": today_stats["total"] or 0,
            "today_sla_rate": (
                (today_stats["on_time"] or 0) / (today_stats["total"] or 1)
            ),
            "avg_review_time_sec": today_stats["avg_review_sec"] or 0,
            "skill_tags": reviewer.skill_tags,
            "shift_remaining_min": (
                (reviewer.shift_end - datetime.now()).total_seconds() / 60
                if reviewer.shift_end else 0
            ),
        }

    def _fetch_matching_items(self, skill_tags: List[str],
                               count: int) -> List[Dict]:
        """获取与审核员技能匹配的内容"""
        if not skill_tags:
            return []

        tag_conditions = " OR ".join(["ai_reason = %s" for _ in skill_tags])
        return self.db.query(f"""
            SELECT * FROM review_queue
            WHERE status = 'pending' AND ({tag_conditions})
            ORDER BY priority DESC, sla_deadline ASC, enqueued_at ASC
            LIMIT %s FOR UPDATE SKIP LOCKED
        """, *skill_tags, count)

    def _fetch_any_items(self, count: int,
                         exclude_ids: List[int] = None) -> List[Dict]:
        """获取任意类型的待审内容"""
        exclude = ""
        params = []
        if exclude_ids:
            exclude = f" AND id NOT IN ({','.join(map(str, exclude_ids))})"

        return self.db.query(f"""
            SELECT * FROM review_queue
            WHERE status = 'pending'{exclude}
            ORDER BY priority DESC, sla_deadline ASC, enqueued_at ASC
            LIMIT %s FOR UPDATE SKIP LOCKED
        """, count)

    def _get_reviewer_profile(self, reviewer_id: str) -> ReviewerProfile:
        """获取审核员画像"""
        # 优先从缓存读
        cached = self.redis.hget(self.reviewer_cache_key, reviewer_id)
        if cached:
            return ReviewerProfile(
                reviewer_id=reviewer_id,
                skill_tags=cached.get("skill_tags", []),
                shift_start=datetime.fromisoformat(cached["shift_start"]),
                shift_end=datetime.fromisoformat(cached["shift_end"]),
                current_load=int(cached.get("current_load", 0)),
                max_concurrent=int(cached.get("max_concurrent", 20)),
            )

        # 回源数据库
        row = self.db.query_one(
            "SELECT * FROM reviewers WHERE reviewer_id = %s", reviewer_id
        )
        return ReviewerProfile(
            reviewer_id=reviewer_id,
            skill_tags=row.get("skill_tags", []),
            shift_start=row.get("shift_start"),
            shift_end=row.get("shift_end"),
            current_load=int(row.get("current_load", 0)),
            max_concurrent=int(row.get("max_concurrent", 20)),
        )


class ReviewerWorkloadBalancer:
    """
    审核员负载均衡器——确保审核任务均匀分配

    策略：
    1. 最少负载优先：当前负载最少的审核员优先分配
    2. 技能匹配：擅长某类型的审核员优先分配该类型
    3. 公平性保证：避免某些审核员过劳而另一些人空闲
    """

    def __init__(self, db, redis):
        self.db = db
        self.redis = redis

    def assign_reviewer(self, item_reason: str) -> Optional[str]:
        """为一条待审内容分配最合适的审核员"""
        # 获取所有在线审核员及其负载
        online_reviewers = self._get_online_reviewers()
        if not online_reviewers:
            return None

        # 按技能匹配度 + 负载排序
        scored_reviewers = []
        for r in online_reviewers:
            # 技能匹配分
            skill_match = 1.0 if item_reason in r.skill_tags else 0.5

            # 负载分（负载越低分越高）
            load_ratio = r.current_load / max(r.max_concurrent, 1)
            load_score = 1.0 - load_ratio

            # 综合评分
            composite = skill_match * 0.6 + load_score * 0.4
            scored_reviewers.append((composite, r))

        # 选择得分最高的审核员
        scored_reviewers.sort(key=lambda x: x[0], reverse=True)
        return scored_reviewers[0][1].reviewer_id

    def rebalance(self):
        """
        重新平衡负载——当某些审核员负载过高时，释放部分任务重新分配

        触发条件：任意审核员负载超过 max_concurrent 的 80%
        """
        online_reviewers = self._get_online_reviewers()
        if not online_reviewers:
            return

        avg_load = sum(r.current_load for r in online_reviewers) / len(online_reviewers)

        for r in online_reviewers:
            if r.current_load > avg_load * 1.5 and r.current_load > 5:
                # 释放部分锁定内容回队列
                release_count = r.current_load - int(avg_load)
                self.db.execute("""
                    UPDATE review_queue
                    SET status = 'pending', locked_by = NULL, locked_at = NULL
                    WHERE locked_by = %s AND status = 'locked'
                    ORDER BY priority ASC, locked_at DESC
                    LIMIT %s
                """, r.reviewer_id, release_count)

                # 更新负载缓存
                self.redis.hset(
                    self.reviewer_cache_key, r.reviewer_id,
                    json.dumps({"current_load": r.current_load - release_count})
                )

    def _get_online_reviewers(self) -> List[ReviewerProfile]:
        """获取所有在线审核员"""
        # 从 Redis 获取心跳在线的审核员
        online_ids = self.redis.smembers("moderation:reviewers:online_ids")
        if not online_ids:
            return []

        reviewers = []
        for rid in online_ids:
            cached = self.redis.hget(self.reviewer_cache_key, rid)
            if cached:
                reviewers.append(ReviewerProfile(
                    reviewer_id=rid,
                    skill_tags=cached.get("skill_tags", []),
                    current_load=int(cached.get("current_load", 0)),
                    max_concurrent=int(cached.get("max_concurrent", 20)),
                    shift_start=None, shift_end=None,
                ))
        return reviewers
```

**人工审核 SLA 达成率实时监控指标：**

| 指标 | 计算公式 | 告警阈值 | 目标值 |
|------|---------|---------|-------|
| SLA 达成率 | on_time_count / total_completed | < 90% | >= 95% |
| 平均等待时间 | AVG(completed_at - enqueued_at) | > 30min | < 15min |
| 超时积压数 | COUNT(pending WHERE sla_deadline < NOW()) | > 100 | 0 |
| 审核员利用率 | current_load / max_concurrent | < 30% 或 > 90% | 60-80% |
| 每小时吞吐量 | COUNT(completed in last hour) | < 200 | >= 300 |

**产能规划：**

| 指标 | 数值 |
|------|------|
| 日需人工审核量 | 3 万条（20% × 15万违规） |
| 单人日产能 | 200条/小时 × 8小时 = 1600条 |
| 所需审核员 | 30000 / 1600 ≈ 19 人 |
| 当前配置 | 50 人（留有富余应对峰值） |

### 视频先发后审（可信用户）

```python
class PublishingStrategy:
    def decide(self, content, user):
        """决定发布策略：先审后发 vs 先发后审"""
        if content.type == "video" and \
           user.trust_score > 0.8 and \
           user.violation_count == 0 and \
           user.account_age_days > 180:
            # 可信用户 → 先发后审
            return "publish_then_review"
        else:
            return "review_then_publish"
```

先发后审风险控制：
- AI 30 秒内完成审核 → 发现违规立即下架
- 30 秒窗口内违规视频可见 → 但仅限可信用户发布的内容
- 可信用户违规率 < 0.01% → 风险可控

### 降级策略（完整阈值与自动切换实现）

降级策略是审核系统的安全阀——当系统承受超过设计容量的压力时，必须在"漏检风险"和"服务可用性"之间做出权衡。以下实现包含具体阈值、自动检测、渐进降级和自动恢复机制。

```python
from enum import Enum
from dataclasses import dataclass
from datetime import datetime, timedelta
import threading
import time


class DegradationLevel(Enum):
    NORMAL = 0       # 正常运行
    LEVEL_1 = 1      # 一级降级：AI 阈值放宽，减少人工审核量
    LEVEL_2 = 2      # 二级降级：视频先发后审（扩展范围）
    LEVEL_3 = 3      # 三级降级：紧急扩容 + 批量审核模式
    LEVEL_4 = 4      # 四级降级：极端情况，部分内容暂不审核


@dataclass
class DegradationThresholds:
    """各级降级的触发和恢复阈值"""
    # 积压量阈值（pending 队列深度）
    level1_trigger_backlog: int = 5000       # 一级降级触发
    level1_recover_backlog: int = 2000       # 一级降级恢复
    level2_trigger_backlog: int = 20000      # 二级降级触发
    level2_recover_backlog: int = 8000       # 二级降级恢复
    level3_trigger_backlog: int = 50000      # 三级降级触发
    level3_recover_backlog: int = 20000      # 三级降级恢复
    level4_trigger_backlog: int = 100000     # 四级降级触发
    level4_recover_backlog: int = 40000      # 四级降级恢复

    # SLA 阈值（平均等待时间，分钟）
    level1_trigger_wait_min: int = 30        # 平均等待 > 30分钟
    level2_trigger_wait_min: int = 60        # 平均等待 > 60分钟
    level3_trigger_wait_min: int = 120       # 平均等待 > 120分钟

    # AI 服务健康阈值
    ai_error_rate_trigger: float = 0.3       # AI 错误率 > 30%
    ai_latency_trigger_ms: int = 10000       # AI P99 延迟 > 10s

    # 持续时间要求（避免瞬时波动触发降级）
    trigger_sustain_seconds: int = 300       # 持续5分钟才触发
    recover_sustain_seconds: int = 600       # 持续10分钟才恢复


# 各级降级对应的 AI 审核阈值配置
DEGRADATION_AI_THRESHOLDS = {
    DegradationLevel.NORMAL: {
        "politics": 0.5, "porn": 0.6, "violence": 0.6,
        "ad": 0.8, "abuse": 0.8,
        # 自动通过阈值（AI 置信度低于此值 → 自动通过，不进人工）
        "auto_pass_ceiling": 0.5,
    },
    DegradationLevel.LEVEL_1: {
        "politics": 0.5,  # 政治敏感不降低（漏检代价极高）
        "porn": 0.7,      # 色情阈值 +0.1
        "violence": 0.7,  # 暴力阈值 +0.1
        "ad": 0.85,       # 广告阈值 +0.05
        "abuse": 0.85,    # 辱骂阈值 +0.05
        "auto_pass_ceiling": 0.6,  # 自动通过范围扩大
    },
    DegradationLevel.LEVEL_2: {
        "politics": 0.5,  # 政治敏感依然不降
        "porn": 0.75,
        "violence": 0.75,
        "ad": 0.9,
        "abuse": 0.9,
        "auto_pass_ceiling": 0.65,
    },
    DegradationLevel.LEVEL_3: {
        "politics": 0.6,  # 极端情况下政治也放宽
        "porn": 0.8,
        "violence": 0.8,
        "ad": 0.95,
        "abuse": 0.95,
        "auto_pass_ceiling": 0.7,
    },
    DegradationLevel.LEVEL_4: {
        # 极端情况：仅审核政治和色情，其余全部放行
        "politics": 0.7,
        "porn": 0.85,
        "violence": 0.95,
        "ad": 0.99,
        "abuse": 0.99,
        "auto_pass_ceiling": 0.85,
    },
}


class DegradationManager:
    """
    降级管理器——自动检测系统压力并执行/恢复降级

    核心原则：
    1. 渐进降级：一级一级升上去，不跳级
    2. 自动恢复：压力降低后自动逐步恢复
    3. 降级有痕：每次降级/恢复都记录审计日志
    4. 人工可介入：管理员可强制降级或恢复
    """

    def __init__(self, db, redis, alert_service, audit_logger,
                 thresholds: DegradationThresholds = None):
        self.db = db
        self.redis = redis
        self.alert = alert_service
        self.audit = audit_logger
        self.thresholds = thresholds or DegradationThresholds()

        # 状态跟踪
        self.current_level = DegradationLevel.NORMAL
        self.level_entered_at = None          # 当前级别进入时间
        self.trigger_since = None             # 触发条件持续满足的起始时间
        self.recover_since = None             # 恢复条件持续满足的起始时间

        # 启动后台检测线程
        self._stop_event = threading.Event()
        self._monitor_thread = threading.Thread(
            target=self._monitor_loop, daemon=True
        )
        self._monitor_thread.start()

    def _monitor_loop(self):
        """后台监控循环，每30秒检查一次"""
        while not self._stop_event.is_set():
            try:
                self._check_and_adjust()
            except Exception as e:
                self.alert.send_warning(f"降级监控异常: {e}")
            self._stop_event.wait(30)

    def _check_and_adjust(self):
        """检查系统状态并决定是否降级或恢复"""
        metrics = self._collect_metrics()
        target_level = self._determine_target_level(metrics)

        if target_level > self.current_level:
            # 需要升级降级级别
            self._try_escalate(target_level, metrics)
        elif target_level < self.current_level:
            # 可以恢复
            self._try_recover(target_level, metrics)

    def _collect_metrics(self) -> Dict:
        """收集当前系统指标"""
        # 队列积压
        backlog = self.db.query_one("""
            SELECT COUNT(*) as cnt,
                   AVG(TIMESTAMPDIFF(MINUTE, enqueued_at, NOW())) as avg_wait_min
            FROM review_queue
            WHERE status = 'pending'
        """)

        # AI 服务健康度
        ai_stats = self.redis.get("moderation:ai_stats")
        ai_stats = json.loads(ai_stats) if ai_stats else {}

        # 当前吞吐量
        throughput = self.db.query_one("""
            SELECT COUNT(*) as cnt
            FROM review_queue
            WHERE status = 'completed'
              AND completed_at > NOW() - INTERVAL 1 HOUR
        """)

        return {
            "backlog_count": backlog["cnt"],
            "avg_wait_minutes": backlog["avg_wait_min"] or 0,
            "ai_error_rate": ai_stats.get("error_rate", 0),
            "ai_p99_latency_ms": ai_stats.get("p99_latency_ms", 0),
            "hourly_throughput": throughput["cnt"],
            "timestamp": datetime.now(),
        }

    def _determine_target_level(self, metrics: Dict) -> DegradationLevel:
        """根据指标确定目标降级级别"""
        backlog = metrics["backlog_count"]
        wait = metrics["avg_wait_minutes"]
        ai_error = metrics["ai_error_rate"]
        ai_latency = metrics["ai_p99_latency_ms"]

        # AI 服务异常 → 至少一级降级
        if ai_error > self.thresholds.ai_error_rate_trigger or \
           ai_latency > self.thresholds.ai_latency_trigger_ms:
            return max(DegradationLevel.LEVEL_1, self.current_level)

        # 按积压量判断
        if backlog >= self.thresholds.level4_trigger_backlog:
            return DegradationLevel.LEVEL_4
        elif backlog >= self.thresholds.level3_trigger_backlog or \
             wait >= self.thresholds.level3_trigger_wait_min:
            return DegradationLevel.LEVEL_3
        elif backlog >= self.thresholds.level2_trigger_backlog or \
             wait >= self.thresholds.level2_trigger_wait_min:
            return DegradationLevel.LEVEL_2
        elif backlog >= self.thresholds.level1_trigger_backlog or \
             wait >= self.thresholds.level1_trigger_wait_min:
            return DegradationLevel.LEVEL_1
        else:
            return DegradationLevel.NORMAL

    def _try_escalate(self, target_level: DegradationLevel, metrics: Dict):
        """尝试升级降级级别（需持续触发才执行）"""
        if self.trigger_since is None:
            self.trigger_since = datetime.now()
            return  # 首次检测到，开始计时

        elapsed = (datetime.now() - self.trigger_since).total_seconds()
        if elapsed < self.thresholds.trigger_sustain_seconds:
            return  # 持续时间不够，继续等待

        # 渐进升级：一次只升一级
        next_level = DegradationLevel(self.current_level.value + 1)
        self._execute_escalation(next_level, metrics)
        self.trigger_since = None  # 重置计时

    def _try_recover(self, target_level: DegradationLevel, metrics: Dict):
        """尝试恢复到更低的降级级别"""
        if self.recover_since is None:
            self.recover_since = datetime.now()
            return

        elapsed = (datetime.now() - self.recover_since).total_seconds()
        if elapsed < self.thresholds.recover_sustain_seconds:
            return

        # 渐进恢复：一次只降一级
        next_level = DegradationLevel(self.current_level.value - 1)
        self._execute_recovery(next_level, metrics)
        self.recover_since = None

    def _execute_escalation(self, new_level: DegradationLevel, metrics: Dict):
        """执行降级升级"""
        old_level = self.current_level
        self.current_level = new_level
        self.level_entered_at = datetime.now()
        self.recover_since = None  # 重置恢复计时

        # 1. 更新 AI 审核阈值到 Redis
        new_thresholds = DEGRADATION_AI_THRESHOLDS[new_level]
        self.redis.set("moderation:thresholds", json.dumps(new_thresholds))

        # 2. 记录阈值变更日志
        for category, threshold in new_thresholds.items():
            if category == "auto_pass_ceiling":
                continue
            self.db.insert("threshold_change_log", {
                "change_id": f"degrade_{new_level.name}_{datetime.now().strftime('%Y%m%d%H%M%S')}",
                "category": category,
                "before_threshold": DEGRADATION_AI_THRESHOLDS[old_level].get(category),
                "after_threshold": threshold,
                "reason": f"自动降级: {old_level.name} → {new_level.name}",
                "trigger_source": "auto_degrade",
                "operator": "system",
            })

        # 3. 执行级别特有操作
        if new_level == DegradationLevel.LEVEL_1:
            self._on_level1_escalate(metrics)
        elif new_level == DegradationLevel.LEVEL_2:
            self._on_level2_escalate(metrics)
        elif new_level == DegradationLevel.LEVEL_3:
            self._on_level3_escalate(metrics)
        elif new_level == DegradationLevel.LEVEL_4:
            self._on_level4_escalate(metrics)

        # 4. 告警通知
        self.alert.send_critical(
            title=f"审核系统降级: {old_level.name} → {new_level.name}",
            message=f"积压: {metrics['backlog_count']}, "
                    f"平均等待: {metrics['avg_wait_minutes']:.0f}分钟",
        )

        # 5. 审计日志
        self.audit.append(
            content_id="SYSTEM",
            action="degradation_escalate",
            actor="system",
            actor_type="system",
            stage="system",
            before_status=old_level.name,
            after_status=new_level.name,
            decision_detail={"metrics": metrics},
        )

    def _execute_recovery(self, new_level: DegradationLevel, metrics: Dict):
        """执行降级恢复"""
        old_level = self.current_level
        self.current_level = new_level

        # 更新阈值
        new_thresholds = DEGRADATION_AI_THRESHOLDS[new_level]
        self.redis.set("moderation:thresholds", json.dumps(new_thresholds))

        # 记录阈值变更日志
        for category, threshold in new_thresholds.items():
            if category == "auto_pass_ceiling":
                continue
            self.db.insert("threshold_change_log", {
                "change_id": f"recover_{new_level.name}_{datetime.now().strftime('%Y%m%d%H%M%S')}",
                "category": category,
                "before_threshold": DEGRADATION_AI_THRESHOLDS[old_level].get(category),
                "after_threshold": threshold,
                "reason": f"自动恢复: {old_level.name} → {new_level.name}",
                "trigger_source": "auto_recover",
                "operator": "system",
            })

        # 级别特有恢复操作
        if new_level.value < DegradationLevel.LEVEL_2.value:
            # 恢复先审后发
            self.redis.set("moderation:video_publish_mode", "review_then_publish")

        if new_level == DegradationLevel.NORMAL:
            # 完全恢复正常
            self.redis.delete("moderation:batch_mode")
            self.level_entered_at = None

        self.alert.send_info(
            title=f"审核系统恢复: {old_level.name} → {new_level.name}",
            message=f"积压: {metrics['backlog_count']}, "
                    f"平均等待: {metrics['avg_wait_minutes']:.0f}分钟",
        )

        self.audit.append(
            content_id="SYSTEM",
            action="degradation_recover",
            actor="system",
            actor_type="system",
            stage="system",
            before_status=old_level.name,
            after_status=new_level.name,
            decision_detail={"metrics": metrics},
        )

    def _on_level1_escalate(self, metrics):
        """一级降级：AI 阈值放宽"""
        # 政治类内容阈值不变，其余阈值 +0.1
        # 效果：低风险内容更多自动通过，减少约 50% 人工审核量
        pass  # 阈值已在 _execute_escalation 中更新到 Redis

    def _on_level2_escalate(self, metrics):
        """二级降级：视频启用先发后审（扩展到所有用户）"""
        self.redis.set("moderation:video_publish_mode", "publish_then_review")
        # 同时启用用户举报优先处理（先发后审内容被举报后立即下架）
        self.redis.set("moderation:report_priority_boost", "true")

    def _on_level3_escalate(self, metrics):
        """三级降级：紧急扩容 + 批量审核模式"""
        # 1. 请求更多审核员
        self._request_emergency_reviewers(count=30)

        # 2. 启用批量审核模式（同类内容批量处理）
        self.redis.set("moderation:batch_mode", "enabled")

        # 3. 降低审核员锁定超时（30min → 15min），加快流转
        self.redis.set("moderation:lock_timeout_min", "15")

    def _on_level4_escalate(self, metrics):
        """四级降级：极端情况——仅审核高风险内容"""
        # 1. 低风险内容（广告、辱骂）全部自动通过
        self.redis.set("moderation:skip_low_risk", "true")

        # 2. 大幅增加紧急审核员
        self._request_emergency_reviewers(count=50)

        # 3. 开启全员通知
        self.alert.send_critical(
            title="审核系统四级降级——请所有管理员立即上线",
            message="当前积压超过10万条，低风险内容暂不审核"
        )

    def _request_emergency_reviewers(self, count: int):
        """请求紧急审核员（调用外包团队 API）"""
        try:
            # 调用外包审核团队管理系统
            response = self.http_client.post(
                "https://outsource-mgmt.internal/api/emergency/request",
                json={"count": count, "skill_tags": ["content_moderation"]}
            )
            if response.status_code == 200:
                self.alert.send_info(f"成功请求 {count} 名紧急审核员，预计30分钟内上线")
            else:
                self.alert.send_warning(f"请求紧急审核员失败: {response.text}")
        except Exception as e:
            self.alert.send_warning(f"请求紧急审核员异常: {e}")

    def force_escalate(self, level: DegradationLevel, reason: str, admin_id: str):
        """管理员强制降级（跳过持续时间检查）"""
        old_level = self.current_level
        self.current_level = level

        new_thresholds = DEGRADATION_AI_THRESHOLDS[level]
        self.redis.set("moderation:thresholds", json.dumps(new_thresholds))

        self.audit.append(
            content_id="SYSTEM",
            action="force_degradation",
            actor=admin_id,
            actor_type="admin",
            stage="admin",
            before_status=old_level.name,
            after_status=level.name,
            decision_detail={"reason": reason},
        )
        self.alert.send_critical(
            title=f"管理员强制降级: {old_level.name} → {level.name}",
            message=f"操作人: {admin_id}, 原因: {reason}",
        )

    def get_status(self) -> Dict:
        """获取当前降级状态"""
        return {
            "current_level": self.current_level.name,
            "level_value": self.current_level.value,
            "entered_at": self.level_entered_at.isoformat() if self.level_entered_at else None,
            "duration_minutes": (
                (datetime.now() - self.level_entered_at).total_seconds() / 60
                if self.level_entered_at else 0
            ),
            "current_thresholds": DEGRADATION_AI_THRESHOLDS[self.current_level],
        }
```

**降级的代价量化分析：**

| 降级级别 | AI阈值偏移 | 误杀率变化 | 漏检率变化 | 人工审核量 | 风险评估 |
|---------|-----------|-----------|-----------|-----------|---------|
| 正常 | 基准 | 1.0% | 0.1% | 3万/天 | 无额外风险 |
| 一级 | +0.1（政治不变） | 0.5%（降低） | 0.3%（升高） | 1.5万/天 | 低风险，政治类不受影响 |
| 二级 | +0.15 | 0.5% | 0.5%（含30秒窗口） | 1万/天 | 中风险，先发后审窗口 |
| 三级 | +0.2 | 0.3% | 1.0% | 0.5万/天 | 较高风险，批量审核准确率下降 |
| 四级 | +0.3（仅审政治/色情） | 0.2% | 2.0% | 0.2万/天 | 高风险，低风险内容不审核 |

**自动恢复流程：**

```
四级降级 → 积压降至4万以下持续10分钟 → 恢复至三级
三级降级 → 积压降至2万以下持续10分钟 → 恢复至二级
二级降级 → 积压降至8000以下持续10分钟 → 恢复至一级
一级降级 → 积压降至2000以下持续10分钟 → 恢复至正常

注意：恢复是逐级进行的，不会从四级直接跳到正常。
每次恢复间隔至少10分钟，防止指标波动导致反复升降级。
```

### 容量规划

**日均审核量分解：**

| 级别 | 日处理量 | 占比 | 延迟 | 所需资源 |
|------|---------|------|------|---------|
| 规则引擎 | 50 万 | 10% | < 1ms | 1 台服务器 |
| AI 文本审核 | 250 万 | 50% | 50-100ms | 2 台 GPU 服务器 |
| AI 图片审核 | 150 万 | 30% | 200-500ms | 4 台 GPU 服务器 |
| AI 视频审核 | 50 万 | 10% | 10-30s | 8 台 GPU 服务器 |
| 人工审核 | 100 万 | 20% | 5-30min | 50 人 |

**峰值 QPS 应对：**

```
峰值 2000 条/秒的分解：
  规则引擎：2000/s → 单机可承受
  AI 审核：
    - 文本 1200/s × 50ms → 需要 60 并发 GPU 推理 → 2 台 GPU
    - 图片 600/s × 300ms → 需要 180 并发 → 4 台 GPU
    - 视频 200/s × 20s → 需要 4000 并发 → 批处理+队列
  人工：400/s → 排队等待
```

**GPU 资源估算：**

| 模型 | 推理延迟 | 单卡 QPS | 所需卡数 | 月成本 |
|------|---------|---------|---------|-------|
| 文本分类 | 50ms | 200 | 6 | ¥3 万 |
| 图片审核 | 300ms | 30 | 80 | ¥40 万 |
| 视频抽帧 | 500ms/帧 | 20 | 40 | ¥20 万 |
| OCR | 200ms | 50 | 20 | ¥10 万 |
| 总计 | | | 146 卡 | ¥73 万/月 |

**优化方向：** 图片审核模型蒸馏（大模型→小模型）→ 单卡 QPS 提升 3 倍 → 卡数降至 27 → 月成本降至 ¥25 万。

### 性能分析（GPU 成本 + 人工生产力 + 延迟分布）

#### GPU 推理成本深度分析

AI 推理是审核系统最大的可变成本。以下从 GPU 利用率、模型部署策略、成本优化等维度进行完整分析。

**单卡 GPU 推理性能基准（NVIDIA A10 24GB）：**

| 模型 | 模型大小 | Batch Size | 推理延迟(P50) | 推理延迟(P99) | 单卡 QPS | 单次推理成本 |
|------|---------|-----------|-------------|-------------|---------|-----------|
| 文本分类 BERT-base | 110M | 32 | 15ms | 45ms | 680 | ¥0.00002 |
| 文本分类 BERT-tiny(蒸馏) | 15M | 64 | 3ms | 8ms | 4200 | ¥0.000003 |
| 图片审核 ResNet50 | 25M | 16 | 80ms | 200ms | 190 | ¥0.00007 |
| 图片审核 MobileNetV3(蒸馏) | 4.2M | 32 | 20ms | 50ms | 1500 | ¥0.000009 |
| 图片裸露检测 OpenNSFW2 | 18M | 16 | 60ms | 150ms | 250 | ¥0.00005 |
| OCR PaddleOCR | 8.6M | 8 | 120ms | 300ms | 60 | ¥0.0002 |
| 语音识别 Whisper-base | 74M | 1 | 1.5s(10s音频) | 3s | 6 | ¥0.002 |

**GPU 集群部署策略：**

```python
class GPUClusterManager:
    """
    GPU 集群管理——动态调度推理任务到最优 GPU

    策略：
    1. 按模型类型分组部署（避免模型切换开销）
    2. 自动扩缩容（基于队列深度和 GPU 利用率）
    3. 推理批处理（凑 batch 后再推理，提高 GPU 利用率）
    """

    # GPU 实例类型与成本
    INSTANCE_TYPES = {
        "A10": {"vram_gb": 24, "cost_per_hour": 15, "suitable": ["text", "image"]},
        "A100": {"vram_gb": 80, "cost_per_hour": 50, "suitable": ["video", "asr"]},
        "T4":  {"vram_gb": 16, "cost_per_hour": 5,  "suitable": ["text_distilled"]},
    }

    # 集群配置
    CLUSTER_CONFIG = {
        "text_classifier": {
            "model": "BERT-tiny",
            "instance_type": "T4",
            "min_instances": 2,
            "max_instances": 8,
            "target_utilization": 0.7,
            "scale_up_threshold": 0.85,    # GPU 利用率 > 85% → 扩容
            "scale_down_threshold": 0.3,    # GPU 利用率 < 30% → 缩容
            "current_instances": 4,
        },
        "image_moderation": {
            "model": "MobileNetV3",
            "instance_type": "A10",
            "min_instances": 4,
            "max_instances": 20,
            "target_utilization": 0.7,
            "scale_up_threshold": 0.85,
            "scale_down_threshold": 0.3,
            "current_instances": 10,
        },
        "video_frame_analysis": {
            "model": "MobileNetV3",
            "instance_type": "A10",
            "min_instances": 2,
            "max_instances": 12,
            "target_utilization": 0.6,
            "scale_up_threshold": 0.8,
            "scale_down_threshold": 0.25,
            "current_instances": 6,
        },
        "asr_service": {
            "model": "Whisper-base",
            "instance_type": "A100",
            "min_instances": 1,
            "max_instances": 4,
            "target_utilization": 0.6,
            "scale_up_threshold": 0.8,
            "scale_down_threshold": 0.2,
            "current_instances": 2,
        },
    }

    def calculate_monthly_cost(self) -> Dict:
        """计算当前配置的月度 GPU 成本"""
        total = 0
        breakdown = {}
        for service, config in self.CLUSTER_CONFIG.items():
            instance_info = self.INSTANCE_TYPES[config["instance_type"]]
            instances = config["current_instances"]
            monthly = instances * instance_info["cost_per_hour"] * 24 * 30
            breakdown[service] = {
                "instances": instances,
                "instance_type": config["instance_type"],
                "monthly_cost": monthly,
            }
            total += monthly

        breakdown["total"] = total
        return breakdown

    def auto_scale(self):
        """自动扩缩容——基于 GPU 利用率和队列深度"""
        for service, config in self.CLUSTER_CONFIG.items():
            # 获取当前 GPU 利用率
            utilization = self._get_gpu_utilization(service)
            queue_depth = self._get_queue_depth(service)

            if utilization > config["scale_up_threshold"] and \
               config["current_instances"] < config["max_instances"]:
                # 扩容
                new_count = min(
                    config["current_instances"] + 2,
                    config["max_instances"]
                )
                self._scale_up(service, new_count - config["current_instances"])
                config["current_instances"] = new_count

            elif utilization < config["scale_down_threshold"] and \
                 queue_depth < 10 and \
                 config["current_instances"] > config["min_instances"]:
                # 缩容
                new_count = max(
                    config["current_instances"] - 1,
                    config["min_instances"]
                )
                self._scale_down(service, config["current_instances"] - new_count)
                config["current_instances"] = new_count

    def _get_gpu_utilization(self, service: str) -> float:
        """获取 GPU 利用率（从监控系统读取）"""
        # 实际实现：从 Prometheus/Datadog 读取
        return 0.65  # 模拟值

    def _get_queue_depth(self, service: str) -> int:
        """获取推理队列深度"""
        return 100  # 模拟值

    def _scale_up(self, service: str, count: int):
        """扩容 GPU 实例"""
        config = self.CLUSTER_CONFIG[service]
        instance_type = config["instance_type"]
        # 调用云 API 启动新实例
        pass

    def _scale_down(self, service: str, count: int):
        """缩容 GPU 实例"""
        # 调用云 API 终止实例
        pass
```

**GPU 月度成本汇总（当前配置 vs 优化后）：**

| 服务 | 当前实例 | 当前月成本 | 优化后实例 | 优化后月成本 | 优化手段 |
|------|---------|----------|----------|----------|---------|
| 文本分类 | 4×T4 | ¥1.4万 | 2×T4 | ¥0.7万 | BERT→BERT-tiny 蒸馏 |
| 图片审核 | 10×A10 | ¥10.8万 | 6×A10 | ¥6.5万 | ResNet→MobileNet 蒸馏 |
| 视频帧审核 | 6×A10 | ¥6.5万 | 4×A10 | ¥4.3万 | 共用图片审核实例 |
| 语音识别 | 2×A100 | ¥7.2万 | 1×A100 | ¥3.6万 | 非实时可排队 |
| OCR | (共用图片实例) | ¥0 | (共用) | ¥0 | 部署在图片审核 GPU |
| **合计** | | **¥25.9万** | | **¥15.1万** | **节省42%** |

#### 人工审核生产力分析

**审核员效率影响因素：**

| 因素 | 影响 | 数据 |
|------|------|------|
| 内容类型 | 视频审核比文本慢3-5倍 | 文本30s/条, 图片45s/条, 视频120s/条 |
| 违规类型 | 政治敏感审核最慢（需谨慎） | 政治类平均60s, 广告类平均20s |
| 批次大小 | 同类内容批量审核效率高30% | 单条审核25s, 批量(10条)平均18s/条 |
| 工作时段 | 下午2-4点效率最低 | 上午效率比下午高15% |
| 连续工作时长 | 超过6小时效率下降20% | 第7小时平均审核时间增加8秒 |
| 上下文切换 | 不同类型切换损失约10秒 | 连续审同一类型效率最高 |

**审核员日产能建模：**

```python
class ReviewerProductivityModel:
    """审核员产能预测模型"""

    # 基准产能（条/小时）
    BASE_RATES = {
        "text": 120,     # 文本：120条/小时
        "image": 80,     # 图片：80条/小时
        "video": 30,     # 视频：30条/小时
    }

    # 效率衰减系数
    FATIGUE_FACTORS = [
        (0, 1.0),     # 第0-2小时：100%效率
        (2, 1.0),
        (4, 0.95),    # 第4-6小时：95%效率
        (6, 0.85),    # 第6-8小时：85%效率
        (8, 0.70),    # 超过8小时：70%效率
    ]

    def estimate_daily_capacity(self, reviewer_count: int,
                                 content_mix: Dict[str, float],
                                 shift_hours: int = 8) -> Dict:
        """
        估算日产能

        content_mix: {"text": 0.6, "image": 0.3, "video": 0.1}
        """
        # 加权平均基准产能
        weighted_rate = sum(
            self.BASE_RATES[t] * content_mix.get(t, 0)
            for t in self.BASE_RATES
        )

        # 考虑疲劳衰减
        effective_hours = 0
        for i in range(shift_hours):
            factor = self._get_fatigue_factor(i)
            effective_hours += factor

        # 单人日产能
        per_person = weighted_rate * effective_hours

        # 团队总产能
        total = per_person * reviewer_count

        return {
            "reviewer_count": reviewer_count,
            "weighted_rate_per_hour": weighted_rate,
            "effective_shift_hours": effective_hours,
            "per_person_daily": int(per_person),
            "total_daily": int(total),
            "content_mix": content_mix,
        }

    def _get_fatigue_factor(self, hour: int) -> float:
        for threshold, factor in reversed(self.FATIGUE_FACTORS):
            if hour >= threshold:
                return factor
        return 0.7

# 示例计算
model = ReviewerProductivityModel()
result = model.estimate_daily_capacity(
    reviewer_count=50,
    content_mix={"text": 0.6, "image": 0.3, "video": 0.1}
)
# 结果：per_person_daily ≈ 880条, total_daily ≈ 44000条
# 对比需求：日需审核3万条 → 富余约47%
```

#### 延迟分布分析（按内容类型）

**端到端审核延迟（从内容提交到发布状态确定）：**

| 内容类型 | P50 | P90 | P99 | P99.9 | 瓶颈环节 |
|---------|-----|-----|-----|-------|---------|
| 文本（规则命中） | 2ms | 5ms | 10ms | 50ms | 规则引擎 |
| 文本（AI 自动通过） | 50ms | 80ms | 150ms | 500ms | GPU 推理排队 |
| 文本（AI 自动拒绝） | 50ms | 80ms | 150ms | 500ms | GPU 推理排队 |
| 文本（进人工审核） | 5min | 15min | 25min | 45min | 人工审核排队 |
| 图片（AI 自动） | 200ms | 400ms | 800ms | 3s | 图片模型推理 |
| 图片（进人工审核） | 8min | 20min | 35min | 60min | 人工审核排队 |
| 视频（AI 自动） | 5s | 8s | 15s | 25s | 帧提取+推理 |
| 视频（进人工审核） | 15min | 30min | 60min | 120min | 人工审核排队 |
| 视频（先发后审） | 0ms(展示) | 0ms | 0ms | 5s(下架) | 30秒窗口内下架 |

**延迟优化策略与效果：**

| 策略 | 适用场景 | 优化效果 | 实施成本 |
|------|---------|---------|---------|
| GPU 推理批处理 | AI审核 | 推理吞吐提升40% | 低（代码改动） |
| 模型蒸馏 | 图片/文本审核 | 推理延迟降低70% | 中（需重训模型） |
| 预测缓存 | 重复/相似内容 | 命中时延迟降至<1ms | 低（加Redis缓存） |
| 审核员技能匹配 | 人工审核 | 平均审核时间降低15% | 低（分配策略调整） |
| 同类内容批量审核 | 人工审核 | 吞吐提升30% | 低（UI改动） |

### 审核反馈闭环

```python
class ModerationFeedbackLoop:
    """审核结果反馈 → 改进模型"""

    def on_user_appeal(self, content_id, appeal_reason):
        """用户申诉误杀 → 加入误杀样本集"""
        self.db.execute("""
            INSERT INTO appeal_samples (content_id, original_decision, appeal_reason)
            VALUES (%s, 'REJECT', %s)
        """, content_id, appeal_reason)

    def on_model_retrain(self):
        """定期用误杀样本重训模型"""
        # 误杀样本：用户申诉成功的内容
        false_positives = self.db.query(
            "SELECT * FROM appeal_samples WHERE appeal_status = 'APPROVED'"
        )

        # 漏检样本：用户举报成功的内容
        false_negatives = self.db.query(
            "SELECT * FROM report_samples WHERE report_status = 'CONFIRMED'"
        )

        # 加入训练集
        self.trainer.add_samples(false_positives, label="NORMAL")
        self.trainer.add_samples(false_negatives, label="VIOLATION")

        # 重训
        new_model = self.trainer.train()
        
        # A/B 测试新模型
        self.ab_test.start(new_model, traffic_pct=10)
```

## 常见陷阱（深度分析）

### 陷阱 1：纯 AI 无人工

**后果：** 新型违规（新诈骗话术、新色情变体）AI 无法识别 → 漏检。模型训练数据是过去的，无法覆盖未来出现的新违规形式。

**解决方案：** 必须有人工兜底。AI 处理 80%，人工处理 20%（低置信度内容）。

### 陷阱 2：先发后审无限制

**后果：** 恶意用户利用审核时间窗口传播违规内容。30 秒窗口内可能有 1000+ 人看到违规视频。

**解决方案：** 限制为可信用户（trust_score > 0.8 + 无违规记录 + 账号 > 半年）。

### 陷阱 3：不区分误杀和漏检代价

**后果：** 统一阈值 → 政治敏感内容漏检率高（代价大）→ 平台被处罚；广告内容误杀率高（代价大）→ 用户投诉流失。

**解决方案：** 差异化阈值——政治 0.5、色情 0.6、广告 0.8。

### 陷阱 4：不做审核反馈闭环

**后果：** 模型准确率不提升 → 人工审核量不减少 → 成本居高不下。

**解决方案：** 误杀（用户申诉）+ 漏检（用户举报）→ 定期重训模型 → A/B 测试新模型。

## 延伸思考

- **审核规则 A/B 测试**：对 10% 流量应用新规则，对比漏检率和误杀率 → 数据驱动决策。
- **用户举报闭环**：举报内容进入高优先级人工队列，形成"AI+人工+社区"三层防线。
- **跨模态审核**：图片中嵌文字（截图）、视频中含音频，需多模态联合审核。单独审核图片可能漏掉截图中的违规文字。

## 审核结果存储与审计追踪

### 数据模型扩展

```sql
-- 审核审计日志（不可篡改，完整追溯）
CREATE TABLE moderation_audit_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    audit_id VARCHAR(64) NOT NULL,           -- 审计流水号
    content_id VARCHAR(64) NOT NULL,
    action VARCHAR(30) NOT NULL,             -- rule_hit / ai_pass / ai_reject / ai_uncertain / human_pass / human_reject / appeal_overturn / admin_override
    actor VARCHAR(32) NOT NULL,              -- system / reviewer_id / admin_id
    actor_type VARCHAR(10) NOT NULL,         -- system / human / admin
    stage VARCHAR(20) NOT NULL,              -- rule / ai / human / appeal / admin
    before_status VARCHAR(20),               -- 变更前状态
    after_status VARCHAR(20),                -- 变更后状态
    decision_detail JSONB,                   -- 完整决策细节（阈值、得分、规则ID等）
    model_version VARCHAR(32),               -- 使用的模型版本号
    rule_version VARCHAR(32),                -- 使用的规则版本号
    threshold_config JSONB,                  -- 当时的阈值配置快照
    latency_ms INT,                          -- 本次决策耗时
    created_at TIMESTAMP DEFAULT NOW(),

    INDEX idx_content_time (content_id, created_at),
    INDEX idx_actor_time (actor, created_at),
    INDEX idx_action_time (action, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 内容审核快照表（记录审核时的完整上下文）
CREATE TABLE content_snapshot (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    content_id VARCHAR(64) NOT NULL,
    snapshot_type VARCHAR(10) NOT NULL,      -- original / blurred / cropped
    content_text TEXT,                        -- 原始文本（审核时快照）
    content_url VARCHAR(500),                 -- 原始图片/视频URL
    thumbnail_url VARCHAR(500),               -- 缩略图（人工审核用）
    content_hash VARCHAR(64),                 -- 内容哈希（去重）
    created_at TIMESTAMP DEFAULT NOW(),

    INDEX idx_content (content_id),
    INDEX idx_hash (content_hash)
);

-- 模型版本管理
CREATE TABLE model_versions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    model_name VARCHAR(50) NOT NULL,         -- text_classifier / image_nudity / image_violence / ocr
    version VARCHAR(32) NOT NULL,            -- v2.3.1
    status VARCHAR(20) DEFAULT 'staging',    -- staging / canary / production / retired
    traffic_pct DECIMAL(4,1) DEFAULT 0,      -- 流量占比
    accuracy_metrics JSONB,                  -- 准确率指标
    deployed_at TIMESTAMP,
    retired_at TIMESTAMP,
    deployed_by VARCHAR(32),

    UNIQUE KEY uk_model_version (model_name, version),
    INDEX idx_status (status)
);

-- 阈值配置变更记录
CREATE TABLE threshold_change_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    change_id VARCHAR(64) NOT NULL,
    category VARCHAR(20) NOT NULL,           -- politics / porn / violence / ad / abuse
    before_threshold DECIMAL(4,3),
    after_threshold DECIMAL(4,3),
    reason VARCHAR(200),                     -- 变更原因
    trigger_source VARCHAR(30),              -- manual / auto_degrade / auto_recover
    operator VARCHAR(32),
    created_at TIMESTAMP DEFAULT NOW(),

    INDEX idx_category_time (category, created_at)
);
```

### 审计追踪服务实现

```python
import hashlib
import uuid
from datetime import datetime
from typing import Optional, Dict, Any

class ModerationAuditService:
    """审核审计追踪服务——所有审核决策必须经过此服务记录"""

    def __init__(self, db, config_repo):
        self.db = db
        self.config_repo = config_repo

    def record_decision(self, content_id: str, action: str, actor: str,
                        actor_type: str, stage: str, before_status: str,
                        after_status: str, decision_detail: Dict[str, Any],
                        model_version: str = None, rule_version: str = None,
                        latency_ms: int = None):
        """记录一次审核决策，不可修改不可删除"""

        # 获取当前阈值配置快照（审计关键：决策时的配置）
        threshold_config = self.config_repo.get_current_thresholds()

        audit_id = self._generate_audit_id(content_id, action)

        self.db.execute("""
            INSERT INTO moderation_audit_log
            (audit_id, content_id, action, actor, actor_type, stage,
             before_status, after_status, decision_detail, model_version,
             rule_version, threshold_config, latency_ms, created_at)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, audit_id, content_id, action, actor, actor_type, stage,
             before_status, after_status,
             json.dumps(decision_detail, ensure_ascii=False),
             model_version, rule_version,
             json.dumps(threshold_config), latency_ms, datetime.now())

        return audit_id

    def record_admin_override(self, content_id: str, admin_id: str,
                              override_decision: str, reason: str):
        """管理员人工覆盖审核结果（高风险操作，必须有原因）"""
        # 查询当前审核状态
        current = self.db.query_one(
            "SELECT publish_status FROM contents WHERE content_id = %s",
            content_id
        )

        # 记录覆盖操作
        self.record_decision(
            content_id=content_id,
            action="admin_override",
            actor=admin_id,
            actor_type="admin",
            stage="admin",
            before_status=current["publish_status"],
            after_status=override_decision,
            decision_detail={
                "override_reason": reason,
                "original_status": current["publish_status"],
                "new_status": override_decision,
            }
        )

        # 更新内容状态
        self.db.update("contents",
            {"publish_status": override_decision},
            {"content_id": content_id})

        # 如果从 rejected 改为 approved → 触发二次确认（防止滥用管理权限）
        if current["publish_status"] == "rejected" and override_decision == "approved":
            self._send_override_notification(admin_id, content_id, reason)

    def query_content_audit_trail(self, content_id: str) -> list:
        """查询某条内容的完整审核轨迹"""
        records = self.db.query("""
            SELECT * FROM moderation_audit_log
            WHERE content_id = %s
            ORDER BY created_at ASC
        """, content_id)

        return [{
            "audit_id": r["audit_id"],
            "action": r["action"],
            "actor": r["actor"],
            "actor_type": r["actor_type"],
            "stage": r["stage"],
            "before_status": r["before_status"],
            "after_status": r["after_status"],
            "decision_detail": r["decision_detail"],
            "model_version": r["model_version"],
            "latency_ms": r["latency_ms"],
            "created_at": r["created_at"].isoformat(),
        } for r in records]

    def query_admin_override_stats(self, start_time, end_time) -> dict:
        """管理员覆盖统计（合规审计）"""
        records = self.db.query("""
            SELECT actor, COUNT(*) as override_count,
                   SUM(CASE WHEN after_status = 'approved' THEN 1 ELSE 0 END) as approve_count,
                   SUM(CASE WHEN after_status = 'rejected' THEN 1 ELSE 0 END) as reject_count
            FROM moderation_audit_log
            WHERE action = 'admin_override'
              AND created_at BETWEEN %s AND %s
            GROUP BY actor
            ORDER BY override_count DESC
        """, start_time, end_time)

        return {r["actor"]: {
            "total_overrides": r["override_count"],
            "approve_overrides": r["approve_count"],
            "reject_overrides": r["reject_count"],
        } for r in records}

    def _generate_audit_id(self, content_id: str, action: str) -> str:
        """生成唯一审计ID，包含时间戳和内容标识"""
        raw = f"{content_id}:{action}:{datetime.now().isoformat()}:{uuid.uuid4().hex[:8]}"
        return f"audit_{hashlib.sha256(raw.encode()).hexdigest()[:16]}"

    def _send_override_notification(self, admin_id, content_id, reason):
        """发送管理权限覆盖通知（合规要求）"""
        # 通知合规团队
        pass


class ContentSnapshotService:
    """内容快照服务——审核时保存内容原始状态，防止用户后续修改导致无法追溯"""

    def __init__(self, db, storage_client):
        self.db = db
        self.storage_client = storage_client

    def save_snapshot(self, content_id: str, content_type: str,
                      content_text: str = None, content_url: str = None) -> str:
        """保存内容快照"""
        content_hash = self._compute_hash(content_text, content_url)

        # 检查是否已有快照（幂等）
        existing = self.db.query_one(
            "SELECT id FROM content_snapshot WHERE content_id = %s AND content_hash = %s",
            content_id, content_hash
        )
        if existing:
            return existing["id"]

        # 生成缩略图（图片/视频）
        thumbnail_url = None
        if content_type in ("image", "video") and content_url:
            thumbnail_url = self._generate_thumbnail(content_url, content_type)

        self.db.execute("""
            INSERT INTO content_snapshot
            (content_id, snapshot_type, content_text, content_url,
             thumbnail_url, content_hash, created_at)
            VALUES (%s, 'original', %s, %s, %s, %s, %s)
        """, content_id, content_text, content_url,
             thumbnail_url, content_hash, datetime.now())

        return content_hash

    def _compute_hash(self, text: str = None, url: str = None) -> str:
        """计算内容哈希，用于去重和篡改检测"""
        raw = (text or "") + (url or "")
        return hashlib.sha256(raw.encode()).hexdigest()

    def _generate_thumbnail(self, url: str, content_type: str) -> str:
        """生成缩略图供人工审核使用"""
        # 缩略图规格：图片 400x300，视频取第1帧 400x300
        thumbnail = self.storage_client.resize(url, width=400, height=300)
        return thumbnail.url
```

### 审核审计不可变日志完整实现

审计日志是内容审核合规的核心要求——任何审核决策都必须可追溯、不可篡改。以下实现包含哈希链校验、签名防篡改、冷存储归档等完整机制。

```python
import hashlib
import hmac
import json
import uuid
from datetime import datetime, timedelta
from typing import Optional, Dict, Any, List


class ImmutableAuditLogger:
    """
    不可变审核日志服务

    核心设计：
    1. 哈希链：每条日志包含前一条日志的哈希，形成链式结构，篡改任意一条即断链
    2. HMAC 签名：每条日志使用服务端密钥签名，防止伪造
    3. 只写接口：仅提供 INSERT，不提供 UPDATE/DELETE
    4. 定期校验：后台任务周期性验证哈希链完整性
    5. 冷归档：超过30天的日志归档到对象存储，仅保留元数据
    """

    SIGNING_KEY = "moderation_audit_hmac_key_2024"  # 实际部署应从密钥管理服务获取

    def __init__(self, db, object_storage_client, alert_service):
        self.db = db
        self.object_storage = object_storage_client
        self.alert = alert_service

    def append(self, content_id: str, action: str, actor: str,
               actor_type: str, stage: str, before_status: str,
               after_status: str, decision_detail: Dict[str, Any],
               model_version: str = None, rule_version: str = None,
               latency_ms: int = None) -> str:
        """
        追加一条审计日志（唯一写入入口）

        流程：
        1. 获取链中最后一条日志的哈希（prev_hash）
        2. 计算当前日志的哈希（包含 prev_hash）
        3. 计算 HMAC 签名
        4. 写入数据库
        5. 返回审计ID
        """
        # 获取前一条日志的哈希
        prev_hash = self._get_last_hash()

        # 生成审计ID
        audit_id = f"aud_{uuid.uuid4().hex[:16]}"

        # 构建日志记录
        now = datetime.now()
        record = {
            "audit_id": audit_id,
            "content_id": content_id,
            "action": action,
            "actor": actor,
            "actor_type": actor_type,
            "stage": stage,
            "before_status": before_status,
            "after_status": after_status,
            "decision_detail": json.dumps(decision_detail, ensure_ascii=False),
            "model_version": model_version,
            "rule_version": rule_version,
            "threshold_config": self._get_threshold_snapshot(),
            "latency_ms": latency_ms,
            "prev_hash": prev_hash,
            "created_at": now.isoformat(),
        }

        # 计算当前记录哈希
        record_hash = self._compute_record_hash(record)
        record["record_hash"] = record_hash

        # 计算 HMAC 签名
        record["signature"] = self._sign_record(record)

        # 写入数据库（只写操作）
        self.db.execute("""
            INSERT INTO moderation_audit_log
            (audit_id, content_id, action, actor, actor_type, stage,
             before_status, after_status, decision_detail, model_version,
             rule_version, threshold_config, latency_ms,
             prev_hash, record_hash, signature, created_at)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, record["audit_id"], record["content_id"], record["action"],
             record["actor"], record["actor_type"], record["stage"],
             record["before_status"], record["after_status"],
             record["decision_detail"], record["model_version"],
             record["rule_version"], record["threshold_config"],
             record["latency_ms"], record["prev_hash"],
             record["record_hash"], record["signature"], now)

        return audit_id

    def verify_chain_integrity(self, limit: int = 1000) -> Dict[str, Any]:
        """
        验证哈希链完整性

        检查项：
        1. prev_hash 链是否连续
        2. record_hash 是否可重新计算验证
        3. HMAC 签名是否有效

        返回验证结果，发现异常立即告警
        """
        records = self.db.query("""
            SELECT audit_id, content_id, action, actor, prev_hash,
                   record_hash, signature, created_at
            FROM moderation_audit_log
            ORDER BY id DESC LIMIT %s
        """, limit)

        broken_links = []
        invalid_hashes = []
        invalid_signatures = []
        prev_hash = None

        for record in reversed(records):  # 从最旧的开始
            # 检查1：prev_hash 链连续性
            if prev_hash is not None and record["prev_hash"] != prev_hash:
                broken_links.append({
                    "audit_id": record["audit_id"],
                    "expected_prev_hash": prev_hash,
                    "actual_prev_hash": record["prev_hash"],
                })

            # 检查2：record_hash 可重新计算
            recomputed = self._compute_record_hash(record)
            if recomputed != record["record_hash"]:
                invalid_hashes.append({
                    "audit_id": record["audit_id"],
                    "expected_hash": recomputed,
                    "actual_hash": record["record_hash"],
                })

            # 检查3：HMAC 签名有效
            if not self._verify_signature(record):
                invalid_signatures.append({
                    "audit_id": record["audit_id"],
                })

            prev_hash = record["record_hash"]

        result = {
            "total_checked": len(records),
            "broken_links": len(broken_links),
            "invalid_hashes": len(invalid_hashes),
            "invalid_signatures": len(invalid_signatures),
            "is_intact": (len(broken_links) == 0 and
                         len(invalid_hashes) == 0 and
                         len(invalid_signatures) == 0),
            "details": {
                "broken_links": broken_links[:10],
                "invalid_hashes": invalid_hashes[:10],
                "invalid_signatures": invalid_signatures[:10],
            }
        }

        # 链不完整 → 紧急告警
        if not result["is_intact"]:
            self.alert.send_critical(
                title="审核审计日志哈希链异常",
                message=f"链检查发现 {len(broken_links)} 处断链, "
                        f"{len(invalid_hashes)} 处哈希不一致, "
                        f"{len(invalid_signatures)} 处签名无效",
                details=result
            )

        return result

    def archive_old_records(self, days: int = 30, batch_size: int = 5000):
        """
        冷归档：将超过指定天数的日志归档到对象存储

        归档策略：
        1. 按天打包为 JSONL 文件
        2. 上传到对象存储（S3/OSS）
        3. 数据库中保留元数据（audit_id, created_at, 归档路径），删除完整记录
        4. 归档文件同样计算哈希链，确保离线数据可验证
        """
        cutoff = datetime.now() - timedelta(days=days)

        # 按天分批处理
        days_to_archive = self.db.query("""
            SELECT DATE(created_at) as archive_date, COUNT(*) as cnt
            FROM moderation_audit_log
            WHERE created_at < %s AND archived = FALSE
            GROUP BY DATE(created_at)
            ORDER BY archive_date
        """, cutoff)

        for day_record in days_to_archive:
            archive_date = day_record["archive_date"]
            count = day_record["cnt"]

            # 分批读取当天记录
            offset = 0
            all_records = []
            while offset < count:
                batch = self.db.query("""
                    SELECT * FROM moderation_audit_log
                    WHERE DATE(created_at) = %s
                    ORDER BY id ASC
                    LIMIT %s OFFSET %s
                """, archive_date, batch_size, offset)
                all_records.extend(batch)
                offset += batch_size

            # 生成 JSONL 归档文件
            archive_filename = f"audit_log/{archive_date.isoformat()}.jsonl"
            jsonl_content = "\n".join(
                json.dumps(r, default=str, ensure_ascii=False) for r in all_records
            )

            # 计算归档文件哈希（用于验证归档完整性）
            archive_hash = hashlib.sha256(jsonl_content.encode()).hexdigest()

            # 上传到对象存储
            self.object_storage.put_object(
                key=archive_filename,
                data=jsonl_content.encode("utf-8"),
                metadata={"sha256": archive_hash, "record_count": str(len(all_records))}
            )

            # 标记数据库记录为已归档，保留元数据
            self.db.execute("""
                UPDATE moderation_audit_log
                SET archived = TRUE, archive_path = %s, archive_hash = %s
                WHERE DATE(created_at) = %s
            """, archive_filename, archive_hash, archive_date)

    def query_content_trail(self, content_id: str) -> List[Dict]:
        """查询内容的完整审核轨迹（包含已归档数据）"""
        # 先查数据库中未归档的记录
        records = self.db.query("""
            SELECT * FROM moderation_audit_log
            WHERE content_id = %s AND archived = FALSE
            ORDER BY created_at ASC
        """, content_id)

        # 查询已归档的记录（从对象存储恢复）
        archived_meta = self.db.query("""
            SELECT audit_id, archive_path, created_at
            FROM moderation_audit_log
            WHERE content_id = %s AND archived = TRUE
            ORDER BY created_at ASC
        """, content_id)

        archived_records = []
        for meta in archived_meta:
            # 从对象存储读取
            data = self.object_storage.get_object(meta["archive_path"])
            for line in data.decode("utf-8").split("\n"):
                record = json.loads(line)
                if record["content_id"] == content_id:
                    archived_records.append(record)

        # 合并并按时间排序
        all_records = archived_records + records
        all_records.sort(key=lambda r: r["created_at"])

        return all_records

    def _get_last_hash(self) -> str:
        """获取链中最后一条记录的哈希"""
        last = self.db.query_one("""
            SELECT record_hash FROM moderation_audit_log
            ORDER BY id DESC LIMIT 1
        """)
        return last["record_hash"] if last else "GENESIS"

    def _compute_record_hash(self, record: Dict) -> str:
        """计算记录哈希（排除 signature 和 record_hash 字段）"""
        hash_fields = {
            "audit_id": record.get("audit_id"),
            "content_id": record.get("content_id"),
            "action": record.get("action"),
            "actor": record.get("actor"),
            "actor_type": record.get("actor_type"),
            "stage": record.get("stage"),
            "before_status": record.get("before_status"),
            "after_status": record.get("after_status"),
            "decision_detail": record.get("decision_detail"),
            "prev_hash": record.get("prev_hash"),
            "created_at": record.get("created_at"),
        }
        raw = json.dumps(hash_fields, sort_keys=True, ensure_ascii=False)
        return hashlib.sha256(raw.encode()).hexdigest()

    def _sign_record(self, record: Dict) -> str:
        """计算 HMAC 签名"""
        message = f"{record['audit_id']}:{record['record_hash']}:{record['prev_hash']}"
        return hmac.new(
            self.SIGNING_KEY.encode(), message.encode(), hashlib.sha256
        ).hexdigest()

    def _verify_signature(self, record: Dict) -> bool:
        """验证 HMAC 签名"""
        message = f"{record['audit_id']}:{record['record_hash']}:{record['prev_hash']}"
        expected = hmac.new(
            self.SIGNING_KEY.encode(), message.encode(), hashlib.sha256
        ).hexdigest()
        return hmac.compare_digest(expected, record.get("signature", ""))

    def _get_threshold_snapshot(self) -> str:
        """获取当前阈值配置快照"""
        # 从 Redis 或配置中心读取
        return json.dumps({
            "politics": 0.5, "porn": 0.6, "violence": 0.6,
            "ad": 0.8, "abuse": 0.8,
        })


# 数据库扩展：增加不可变日志相关字段
IMMUTABLE_LOG_MIGRATION = """
-- 为审计日志增加哈希链和签名字段
ALTER TABLE moderation_audit_log
    ADD COLUMN prev_hash VARCHAR(64) NOT NULL DEFAULT '' AFTER created_at,
    ADD COLUMN record_hash VARCHAR(64) NOT NULL DEFAULT '' AFTER prev_hash,
    ADD COLUMN signature VARCHAR(64) NOT NULL DEFAULT '' AFTER record_hash,
    ADD COLUMN archived BOOLEAN DEFAULT FALSE AFTER signature,
    ADD COLUMN archive_path VARCHAR(500) DEFAULT NULL AFTER archived,
    ADD COLUMN archive_hash VARCHAR(64) DEFAULT NULL AFTER archive_path;

-- 创建归档相关索引
CREATE INDEX idx_archived ON moderation_audit_log(archived, created_at);
CREATE INDEX idx_archive_path ON moderation_audit_log(archive_path);

-- 审计日志禁止 UPDATE/DELETE（通过数据库触发器强制）
DELIMITER //
CREATE TRIGGER prevent_audit_update
BEFORE UPDATE ON moderation_audit_log
FOR EACH ROW
BEGIN
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = '审计日志禁止修改';
END;
//

CREATE TRIGGER prevent_audit_delete
BEFORE DELETE ON moderation_audit_log
FOR EACH ROW
BEGIN
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = '审计日志禁止删除';
END;
//
DELIMITER ;
"""
```

**审计日志完整性保障机制总览：**

| 层级 | 机制 | 检测能力 | 响应 |
|------|------|---------|------|
| 应用层 | 只写接口（无 UPDATE/DELETE） | 防止应用代码误操作 | 编译期约束 |
| 数据库层 | 触发器禁止 UPDATE/DELETE | 防止直接 SQL 操作 | 操作被拒绝 |
| 哈希链 | 前后记录哈希串联 | 检测任意记录篡改 | 链断裂告警 |
| HMAC 签名 | 服务端密钥签名 | 检测伪造记录 | 签名不匹配告警 |
| 定期校验 | 每小时验证最近 1000 条 | 及时发现异常 | 告警 + 人工核查 |
| 冷归档 | 30天后归档到对象存储 | 防止在线数据被批量篡改 | 归档文件哈希校验 |

## 异常场景完整演练

### 场景 1：AI 审核服务全部宕机

```
触发：3 台 GPU 服务器同时故障 → AI 审核不可用
影响：所有新发布内容无人审核
处理：
  1. 自动降级：所有内容进入人工审核队列
  2. 临时切换为先发后审（仅可信用户）
  3. 审核阈值降低：规则引擎通过的直接放行
  4. 通知运营：AI 服务故障，人工审核积压可能增加
  5. 修复 AI 服务（约 30 分钟）
  6. 恢复后：处理积压队列（约 2 小时消化完）
风险：30 分钟内违规内容可能漏检 → 依赖举报机制兜底
```

### 场景 2：突发热点事件导致审核量 10 倍暴增

```
触发：社会热点事件 → 大量用户发帖讨论 → 500 万条/小时（平时 50 万条/小时）
处理：
  1. 一级降级：AI 阈值提高 0.1 → 更多内容自动通过 → 人工审核量降至 50 万/小时
  2. 热点话题关键词自动标记 → 相关内容进入高优先级审核队列
  3. 临时增加审核员：从其他业务线调配 20 人
  4. 政治敏感内容严格审核（阈值不降低）
  5. 持续约 4-6 小时 → 逐步恢复
关键指标：误杀率从 1% 升至 3%，漏检率从 0.1% 升至 0.3%
```

### 场景 3：模型更新后误杀率骤升

```
触发：新模型上线后 → 误杀率从 1% 升至 8% → 用户投诉量暴增
检测：
  1. 用户申诉率异常升高（> 5% 的审核结果被申诉）
  2. 申诉通过率 > 90%（说明大部分是误杀）
处理：
  1. 立即回滚到上一版模型
  2. 分析误杀原因：新训练数据中某类内容占比过高导致模型偏向
  3. 重新平衡训练数据 → 重新训练
  4. 灰度上线：10% → 30% → 100%，每阶段观察 24 小时
  5. 灰度期间如果误杀率 > 2% → 自动回滚
根因：训练数据偏差（data bias），非模型架构问题
```

### 场景 4：模型推理服务部分故障（GPU 显存溢出）

```
触发：图片审核模型接收超大分辨率图片 → GPU OOM → 推理进程崩溃
影响：图片审核请求超时，返回 500 → 大量图片内容进入人工审核队列
检测：
  1. 推理服务健康检查：连续 3 次探针失败 → 标记该模型实例不可用
  2. 监控指标：图片审核 P99 延迟从 500ms 飙升至 10s+，超时率 > 30%
处理：
  1. 负载均衡自动摘除故障实例 → 流量转发到健康实例
  2. 图片预处理增加分辨率限制：> 4096px 的图片先缩放再送入模型
  3. 故障实例自动重启 + GPU 显存预热
  4. 人工审核队列积压预警 → 触发一级降级
恢复时间：5-10 分钟（自动重启 + 预热）
预防：图片预处理管线统一限制输入尺寸；GPU 推理服务设置 request 限流
```

### 场景 5：审核队列持续积压（人工产能不足）

```
触发：大促活动 + 周末流量 → 日均审核量从 5 万条升至 20 万条
指标：
  - pending 队列深度 > 10000 持续 2 小时
  - 人工审核 SLA 达成率 < 80%（目标 95%）
  - 平均等待时间 > 60 分钟（目标 < 30 分钟）
处理：
  1. 触发一级降级：AI 自动通过阈值提高 0.1 → 人工审核量减少 50%
  2. 触发二级降级：视频内容全部先发后审（仅保留文本/图片先审后发）
  3. 动态扩容审核员：从外包团队调用 30 人，30 分钟内上线
  4. 临时调整 SLA：政治/色情内容保持 30 分钟 SLA，广告/辱骂放宽至 2 小时
  5. 启用批量审核模式：同类型低风险内容可批量处理（每批 10 条）
风险：广告类漏检率从 0.1% 升至 0.5%，但广告漏检代价低
恢复：大促结束后 4 小时消化积压，逐步取消降级
```

### 场景 6：误杀率突然飙升（模型漂移或规则误配）

```
触发：运营更新黑名单 → 误添加常用词 → 误杀率从 1% 升至 15%
检测：
  1. 实时监控：AI 审核拒绝率异常升高（> 20%，正常 < 5%）
  2. 用户申诉率飙升至 10%（正常 < 2%）
  3. 申诉通过率 > 95%（几乎全部是误杀）
处理（5 分钟内完成）：
  1. 触发告警 → 运维值班确认
  2. 对比最近变更：查看规则引擎变更记录 → 定位误配的黑名单关键词
  3. 紧急回滚规则变更：从数据库删除误配关键词
  4. 批量重审误杀内容：过去 1 小时内被规则引擎拒绝的内容自动重入审核队列
  5. 通知受影响用户：内容已恢复，致歉
根因：规则变更缺少灰度机制 → 修复：新增规则先在 1% 流量灰度，观察 30 分钟再全量上线
```

### 场景 7：新型违规模式出现（AI 无法识别）

```
触发：新型诈骗话术出现（如"投资理财+二维码"组合）→ AI 训练数据中没有此类样本 → 漏检
检测：
  1. 用户举报率上升：某类内容举报率 > 5%（正常 < 1%）
  2. 举报聚类分析：举报内容存在共性模式（关键词/图片特征聚类）
  3. 运营巡检发现：人工审核时发现新型违规模式
处理：
  1. 紧急规则补丁：提取新型违规的关键特征 → 加入规则引擎（分钟级生效）
  2. 标注紧急样本：运营标注 50-100 条新型违规样本
  3. 模型增量训练：用紧急样本微调模型（小时级）
  4. 灰度上线新模型：10% 流量验证 → 全量上线
  5. 事后补充：完整标注 500+ 条样本 → 正式重训模型
时效：规则补丁 10 分钟生效，模型更新 2-4 小时
预防：建立违规模式监控看板，举报聚类每日分析
```
### 场景 8：视频审核抽帧服务瓶颈（FFmpeg 进程泄漏）

```python
# 场景描述：视频审核高峰期，FFmpeg 子进程未正确回收 → 进程数持续增长 → 服务器 OOM
# 根因：subprocess.run 在极端情况下未能正确回收子进程句柄

class VideoFrameExtractorGuard:
    """视频抽帧服务守护——防止 FFmpeg 进程泄漏"""

    MAX_CONCURRENT_FFMPEG = 20      # 最大并发 FFmpeg 进程数
    FFMPEG_TIMEOUT_SEC = 120        # 单次抽帧超时
    ZOMBIE_CHECK_INTERVAL = 30      # 僵尸进程检查间隔（秒）

    def __init__(self, frame_extractor, alert_service):
        self.extractor = frame_extractor
        self.alert = alert_service
        self.active_processes = {}   # pid → {start_time, video_id}
        self.semaphore = asyncio.Semaphore(self.MAX_CONCURRENT_FFMPEG)

    async def extract_with_guard(self, video_url: str, strategy: str = "hybrid"):
        """带守护的抽帧操作"""
        if self.semaphore.locked():
            self.alert.send_warning(
                title="视频抽帧并发已满",
                message=f"当前 {len(self.active_processes)} 个 FFmpeg 进程运行中，新请求需排队等待"
            )

        async with self.semaphore:
            process_id = str(uuid.uuid4())[:8]
            self.active_processes[process_id] = {
                "start_time": time.time(),
                "video_url": video_url,
            }

            try:
                result = await asyncio.wait_for(
                    asyncio.get_event_loop().run_in_executor(
                        None, self._extract_sync, video_url, strategy
                    ),
                    timeout=self.FFMPEG_TIMEOUT_SEC
                )
                return result
            except asyncio.TimeoutError:
                self._kill_orphan_processes(video_url)
                self.alert.send_warning(
                    title="FFmpeg 抽帧超时",
                    message=f"视频 {video_url} 抽帧超时 {self.FFMPEG_TIMEOUT_SEC}s，已终止"
                )
                raise VideoProcessingError(f"FFmpeg timeout: {video_url}")
            finally:
                self.active_processes.pop(process_id, None)

    def check_zombie_processes(self):
        """定期检查并清理僵尸 FFmpeg 进程"""
        import psutil
        now = time.time()
        zombie_count = 0
        for proc in psutil.process_iter(["pid", "name", "create_time"]):
            if proc.info["name"] == "ffmpeg":
                create_time = proc.info["create_time"]
                if now - create_time > self.FFMPEG_TIMEOUT_SEC:
                    try:
                        proc.kill()
                        zombie_count += 1
                    except psutil.NoSuchProcess:
                        pass
        if zombie_count > 0:
            self.alert.send_warning(
                title="FFmpeg 僵尸进程清理",
                message=f"清理了 {zombie_count} 个僵尸 FFmpeg 进程"
            )
        return zombie_count

    def _kill_orphan_processes(self, video_url: str):
        """杀掉与指定视频关联的孤儿 FFmpeg 进程"""
        import signal
        for pid, info in list(self.active_processes.items()):
            if info.get("video_url") == video_url:
                try:
                    os.kill(int(pid), signal.SIGTERM)
                except (ProcessLookupError, ValueError):
                    pass

    def _extract_sync(self, video_url, strategy):
        """同步抽帧（在线程池中执行）"""
        if strategy == "hybrid":
            return self.extractor.extract_frames_hybrid(video_url)
        elif strategy == "scene_change":
            return self.extractor.extract_frames_scene_change(video_url)
        else:
            return self.extractor.extract_frames_uniform(video_url)

# 部署配置：
# 1. Kubernetes 中设置容器内存限制 4Gi，FFmpeg OOM 时容器被杀而非宿主机
# 2. 定时任务每 30 秒执行 check_zombie_processes
# 3. 抽帧队列积压 > 500 时触发告警 → 降级为均匀抽帧（间隔从 5s 改为 15s）
```

### 场景 9：审核规则热更新导致短暂不一致

```python
# 场景描述：运营紧急更新黑名单关键词，规则引擎部分实例已加载新规则，
# 部分实例仍使用旧规则 → 同一内容在不同实例上审核结果不同
# 影响：约 5 分钟的不一致窗口，期间约 2% 的内容审核结果不确定

class RuleEngineHotReload:
    """规则引擎热更新——保证灰度加载，避免不一致窗口"""

    def __init__(self, db, redis, rule_filter_instances: list):
        self.db = db
        self.redis = redis
        self.instances = rule_filter_instances

    def deploy_rule_change(self, change_id: str, rule_diff: dict,
                           strategy: str = "gradual"):
        """
        部署规则变更

        strategy:
          - "gradual": 灰度加载，1% → 10% → 50% → 100%，每步观察 15 分钟
          - "instant": 全量加载（仅限紧急补丁，如新型违规）
          - "scheduled": 定时加载（在低峰期凌晨 3 点）
        """
        if strategy == "gradual":
            phases = [0.01, 0.10, 0.50, 1.00]
            observe_minutes = 15
        elif strategy == "instant":
            phases = [1.00]
            observe_minutes = 0
        elif strategy == "scheduled":
            scheduled_time = self._next_3am()
            self.redis.set("rule:pending_change", json.dumps({
                "change_id": change_id,
                "rule_diff": rule_diff,
                "execute_at": scheduled_time.isoformat(),
            }))
            return

        for phase_pct in phases:
            # 选择该比例的实例加载新规则
            target_count = int(len(self.instances) * phase_pct)
            for instance in self.instances[:target_count]:
                instance.reload_rules(rule_diff)

            # 记录变更日志
            self.db.execute("""
                INSERT INTO rule_change_deployment
                (change_id, phase_pct, instance_count, deployed_at)
                VALUES (%s, %s, %s, NOW())
            """, change_id, phase_pct, target_count)

            # 观察窗口
            if observe_minutes > 0:
                time.sleep(observe_minutes * 60)
                metrics = self._check_deployment_metrics(change_id)
                if metrics["false_positive_spike"] > 0.05:
                    self._rollback(change_id, target_count)
                    self.alert.send_critical(
                        title="规则变更自动回滚",
                        message=f"变更 {change_id} 在 {phase_pct:.0%} 阶段误杀率飙升，已自动回滚"
                    )
                    return

    def _check_deployment_metrics(self, change_id: str) -> dict:
        """检查部署期间的审核指标"""
        new_reject_rate = self.db.query_one("""
            SELECT COUNT(*) * 1.0 / (
                SELECT COUNT(*) FROM moderation_audit_log
                WHERE created_at > NOW() - INTERVAL 15 MINUTE
            ) as rate
            FROM moderation_audit_log
            WHERE action = 'rule_hit'
              AND created_at > NOW() - INTERVAL 15 MINUTE
              AND decision_detail LIKE %s
        """, f'%{change_id}%')

        baseline_reject_rate = self.db.query_one("""
            SELECT AVG(daily_reject_rate) FROM daily_moderation_stats
            WHERE date > DATE_SUB(CURDATE(), INTERVAL 7 DAY)
        """)

        return {
            "false_positive_spike": max(0, (new_reject_rate or 0) - (baseline_reject_rate or 0)),
            "current_reject_rate": new_reject_rate,
            "baseline_reject_rate": baseline_reject_rate,
        }

    def _rollback(self, change_id: str, affected_count: int):
        """回滚规则变更"""
        for instance in self.instances[:affected_count]:
            instance.rollback_rules(change_id)
```

### 场景 10：跨区域审核规则差异导致合规风险

```python
# 场景描述：平台在多个国家运营，不同国家对内容合规的要求差异大。
# 统一规则导致在某些区域过度审核，在另一些区域审核不足。
# 例如：中东对宗教内容极其敏感，欧洲对纳粹符号零容忍

class RegionalModerationPolicy:
    """区域化审核策略——根据当地法规动态加载审核规则"""

    REGIONAL_POLICIES = {
        "CN": {
            "name": "中国大陆",
            "strict_categories": ["politics", "porn"],
            "thresholds": {"politics": 0.4, "porn": 0.5, "violence": 0.6, "ad": 0.8, "abuse": 0.8},
            "banned_symbols": [],
            "required_human_review": ["politics"],
            "max_publish_first_window_sec": 30,
        },
        "DE": {
            "name": "德国",
            "strict_categories": ["nazi", "hate_speech"],
            "thresholds": {"nazi": 0.3, "hate_speech": 0.4, "porn": 0.6, "violence": 0.6, "ad": 0.8},
            "banned_symbols": ["swastika", "ss_runes", "nazi_eagle"],
            "required_human_review": ["nazi", "hate_speech"],
            "max_publish_first_window_sec": 60,
        },
        "SA": {
            "name": "沙特阿拉伯",
            "strict_categories": ["religion", "porn", "alcohol"],
            "thresholds": {"religion": 0.3, "porn": 0.4, "alcohol": 0.5, "violence": 0.6, "ad": 0.8},
            "banned_symbols": [],
            "required_human_review": ["religion", "porn"],
            "max_publish_first_window_sec": 0,  # 不允许先发后审
        },
        "US": {
            "name": "美国",
            "strict_categories": ["csam", "violence"],
            "thresholds": {"csam": 0.2, "violence": 0.5, "porn": 0.7, "ad": 0.8, "abuse": 0.8},
            "banned_symbols": [],
            "required_human_review": ["csam"],
            "max_publish_first_window_sec": 60,
        },
    }

    def __init__(self, db, redis, ai_moderator, symbol_detector):
        self.db = db
        self.redis = redis
        self.ai_moderator = ai_moderator
        self.symbol_detector = symbol_detector

    def get_policy(self, region_code: str) -> dict:
        """获取指定区域的审核策略"""
        policy = self.REGIONAL_POLICIES.get(region_code, self.REGIONAL_POLICIES["US"])
        override = self.redis.get(f"moderation:policy_override:{region_code}")
        if override:
            policy.update(json.loads(override))
        return policy

    def moderate_with_regional_policy(self, content, user) -> dict:
        """按区域策略审核内容"""
        region = self.get_user_region(user.user_id)
        policy = self.get_policy(region)

        # 1. 检测区域禁用符号
        if policy["banned_symbols"]:
            symbol_result = self.symbol_detector.predict(content, policy["banned_symbols"])
            if symbol_result.hit:
                return {
                    "is_violation": True,
                    "reason": f"banned_symbol:{symbol_result.symbol}",
                    "confidence": 1.0,
                    "requires_human_review": True,
                    "region": region,
                }

        # 2. 使用区域特定阈值进行 AI 审核
        ai_result = self.ai_moderator.moderate(content, thresholds=policy["thresholds"])

        # 3. 严格类别必须人工复审
        if ai_result.reason in policy.get("required_human_review", []):
            ai_result.is_violation = None  # 强制进入人工审核

        # 4. 先发后审窗口限制
        if policy["max_publish_first_window_sec"] == 0:
            publish_strategy = "review_then_publish"
        else:
            publish_strategy = self._decide_publish_strategy(content, user, policy)

        return {
            "is_violation": ai_result.is_violation,
            "reason": ai_result.reason,
            "confidence": ai_result.confidence,
            "requires_human_review": ai_result.is_violation is None,
            "region": region,
            "publish_strategy": publish_strategy,
        }

    def get_user_region(self, user_id: str) -> str:
        """获取用户所在区域"""
        # 从用户注册信息或 IP 地址推断
        user = self.db.query_one(
            "SELECT region_code FROM users WHERE user_id = %s", user_id
        )
        return user["region_code"] if user else "US"
```

**区域审核策略对比总览：**

| 区域 | 严格类别 | 最低阈值 | 禁用符号检测 | 必须人工复审 | 先发后审 |
|------|---------|---------|------------|-----------|---------|
| 中国大陆 | 政治、色情 | 0.4 | 无 | 政治类 | 可信用户30s |
| 德国 | 纳粹、仇恨言论 | 0.3 | 万字符/SS标志 | 纳粹/仇恨 | 可信用户60s |
| 沙特 | 宗教、色情、酒精 | 0.3 | 无 | 宗教/色情 | 禁止 |
| 美国 | CSAM、暴力 | 0.2 | 无 | CSAM | 可信用户60s |

### 场景 11：模型推理服务全部宕机（GPU 集群故障）

```python
class ModelServingFailover:
    """
    模型推理服务故障处理器

    故障场景：GPU 集群全部不可用（如驱动更新失败、机房网络故障）
    核心策略：AI 不可用时，规则引擎+人工审核兜底，确保审核不停摆
    """

    def __init__(self, db, redis, alert_service, rule_filter, review_queue):
        self.db = db
        self.redis = redis
        self.alert = alert_service
        self.rule_filter = rule_filter
        self.review_queue = review_queue
        self.ai_healthy = True
        self.failover_at = None

    def check_ai_health(self) -> bool:
        """AI 服务健康检查（每10秒执行一次）"""
        try:
            response = self.ai_client.predict("health_check_probe_text")
            latency = response.latency_ms

            timeout_count = self.redis.incr("moderation:ai:timeout_count")
            if latency > 5000:
                self.redis.expire("moderation:ai:timeout_count", 60)
            else:
                self.redis.set("moderation:ai:timeout_count", 0)

            if timeout_count >= 3:
                self._on_ai_unavailable()
                return False

            if not self.ai_healthy:
                self._on_ai_recovered()
            self.ai_healthy = True
            return True

        except Exception:
            self.redis.incr("moderation:ai:timeout_count")
            self.redis.expire("moderation:ai:timeout_count", 60)
            timeout_count = int(self.redis.get("moderation:ai:timeout_count") or 0)
            if timeout_count >= 3:
                self._on_ai_unavailable()
            return False

    def moderate_with_failover(self, content, user) -> Dict:
        """
        带故障转移的审核入口

        正常：规则引擎 → AI 审核 → 人工
        AI宕机：规则引擎 → 人工（跳过AI）
        """
        # 第一级：规则引擎（始终可用，不依赖 GPU）
        rule_result = self.rule_filter.check(content)
        if rule_result.hit:
            return {
                "stage": "rule",
                "is_violation": True,
                "reason": rule_result.reason,
                "confidence": 1.0,
            }

        # 第二级：AI 审核（可能不可用）
        if self.ai_healthy:
            try:
                ai_result = self.ai_moderator.check(content)
                if ai_result.is_violation is True:
                    return {
                        "stage": "ai",
                        "is_violation": True,
                        "reason": ai_result.reason,
                        "confidence": ai_result.confidence,
                    }
                elif ai_result.is_violation is False:
                    return {
                        "stage": "ai",
                        "is_violation": False,
                        "reason": None,
                        "confidence": ai_result.confidence,
                    }
                else:
                    self.review_queue.enqueue(content, ai_result, user.trust_score)
                    return {
                        "stage": "human",
                        "is_violation": None,
                        "reason": "ai_uncertain",
                    }
            except Exception:
                pass  # AI 调用失败 → 降级到人工

        # 降级路径：直接进人工审核
        fallback_result = type("AIResult", (), {
            "is_violation": None,
            "reason": rule_result.reason if rule_result.hit else "unknown",
            "confidence": 0.0,
            "all_scores": {},
        })()
        self.review_queue.enqueue(content, fallback_result, user.trust_score)

        # 可信用户视频 → 临时先发后审
        if content.type == "video" and user.trust_score > 0.8:
            return {
                "stage": "publish_then_review",
                "is_violation": None,
                "reason": "ai_failover_trusted_user",
            }

        return {
            "stage": "human",
            "is_violation": None,
            "reason": "ai_unavailable",
        }

    def _on_ai_unavailable(self):
        """AI 服务不可用时的处理"""
        if self.ai_healthy:
            self.ai_healthy = False
            self.failover_at = datetime.now()

            self.redis.set("moderation:ai:status", "unavailable")
            self.redis.set("moderation:degradation:force", "level_2")
            self.redis.set("moderation:lock_timeout_min", "15")

            self.alert.send_critical(
                title="AI 审核服务全部不可用——已切换到人工兜底模式",
                message="所有新内容将直接进入人工审核队列。"
                        "可信用户视频启用先发后审。"
                        "预计影响时长：30分钟-2小时",
            )

    def _on_ai_recovered(self):
        """AI 服务恢复后的处理"""
        recovery_time = (datetime.now() - self.failover_at).total_seconds() / 60

        # 逐步恢复 AI 流量（避免瞬间压垮刚恢复的服务）
        for pct in [10, 30, 50, 100]:
            self.redis.set("moderation:ai:traffic_pct", pct)
            time.sleep(300)  # 每阶段观察5分钟

        self.redis.set("moderation:ai:status", "healthy")
        self.redis.delete("moderation:degradation:force")

        self.alert.send_info(
            title="AI 审核服务已恢复",
            message=f"故障持续 {recovery_time:.0f} 分钟，已逐步恢复100%流量",
        )
```

**AI 全宕机期间的影响量化：**

| 指标 | 正常 | AI宕机期间 | 增量 |
|------|------|-----------|------|
| 日均人工审核量 | 3万条 | 15万条（全部进人工） | +12万条 |
| 平均审核等待时间 | 15分钟 | 60-120分钟 | +4-7倍 |
| 政治类 SLA 达成率 | 98% | 70-80% | 下降 |
| 可信用户视频先发后审 | 10% | 100% | 全量 |
| 规则引擎拦截量 | 5万条 | 5万条 | 不变 |

### 场景 12：审核队列溢出（积压超过系统极限）

```python
class ReviewQueueOverflowHandler:
    """
    审核队列溢出处理器

    场景：大促/热点导致人工审核队列深度超过10万条，
    超过数据库和审核员的处理极限
    核心策略：分级溢出 + 自动降级 + 智能丢弃
    """

    OVERFLOW_LEVELS = {
        "level_1": {"threshold": 30000, "action": "enable_batch_mode"},
        "level_2": {"threshold": 60000, "action": "enable_auto_pass_low_risk"},
        "level_3": {"threshold": 100000, "action": "enable_emergency_mode"},
        "level_4": {"threshold": 200000, "action": "enable_triage_mode"},
    }

    def __init__(self, db, redis, alert_service, degradation_manager):
        self.db = db
        self.redis = redis
        self.alert = alert_service
        self.degradation = degradation_manager
        self.current_overflow_level = 0

    def check_overflow(self):
        """定期检查队列是否溢出（每分钟执行）"""
        backlog = self.db.query_one("""
            SELECT COUNT(*) as cnt FROM review_queue WHERE status = 'pending'
        """)["cnt"]

        for level_name, config in sorted(
            self.OVERFLOW_LEVELS.items(),
            key=lambda x: x[1]["threshold"],
            reverse=True
        ):
            if backlog >= config["threshold"]:
                level_num = int(level_name.split("_")[1])
                if level_num > self.current_overflow_level:
                    self._activate_overflow_level(level_num, backlog)
                return

        if self.current_overflow_level > 0:
            self._try_deactivate_overflow(backlog)

    def _activate_overflow_level(self, level: int, backlog: int):
        """激活溢出处理级别"""
        self.current_overflow_level = level

        if level >= 1:
            self._enable_batch_mode()
        if level >= 2:
            self._auto_pass_low_risk()
        if level >= 3:
            self._enable_emergency_mode()
        if level >= 4:
            self._enable_triage_mode()

        self.alert.send_critical(
            title=f"审核队列溢出 Level {level}",
            message=f"当前积压 {backlog} 条，已激活 Level {level} 溢出处理"
        )

    def _enable_batch_mode(self):
        """批量审核模式：同类内容批量处理"""
        self.redis.set("moderation:batch_mode", "enabled")
        self.redis.set("moderation:batch_size", "10")

    def _auto_pass_low_risk(self):
        """
        自动通过低风险内容

        条件：AI 置信度 < 0.3 且违规类型为广告/辱骂
        效果：预计减少 30-40% 人工审核量
        风险：广告类漏检率从 0.1% 升至约 0.8%
        """
        affected = self.db.execute("""
            UPDATE review_queue
            SET status = 'completed'
            WHERE status = 'pending'
              AND ai_confidence < 0.3
              AND ai_reason IN ('ad', 'abuse')
        """)

        self.db.execute("""
            UPDATE contents c
            INNER JOIN review_queue q ON c.content_id = q.content_id
            SET c.publish_status = 'approved'
            WHERE q.status = 'completed'
              AND q.ai_confidence < 0.3
              AND q.ai_reason IN ('ad', 'abuse')
        """)

        self.alert.send_warning(
            title="已自动通过低风险内容",
            message=f"已自动通过 {affected} 条低风险内容（广告/辱骂，AI置信度<0.3）"
        )

    def _enable_emergency_mode(self):
        """紧急模式"""
        self.degradation._request_emergency_reviewers(count=50)

        self.db.execute("""
            UPDATE review_queue
            SET sla_deadline = DATE_ADD(sla_deadline, INTERVAL 60 MINUTE)
            WHERE ai_reason NOT IN ('politics', 'porn')
              AND status = 'pending'
        """)

        self.redis.set("moderation:lock_timeout_min", "10")

    def _enable_triage_mode(self):
        """分类筛选模式——仅高优先级内容进入审核"""
        self.db.execute("""
            UPDATE review_queue
            SET status = 'deferred'
            WHERE status = 'pending'
              AND priority < 40
              AND ai_reason IN ('ad', 'abuse')
        """)
        self.redis.set("moderation:triage_mode", "enabled")

    def _try_deactivate_overflow(self, current_backlog: int):
        """尝试退出溢出模式"""
        recover_thresholds = {1: 10000, 2: 20000, 3: 40000, 4: 80000}
        threshold = recover_thresholds.get(self.current_overflow_level, 0)

        if current_backlog < threshold:
            self.current_overflow_level -= 1
            if self.current_overflow_level == 0:
                self._deactivate_all_overflow()

            if self.current_overflow_level < 4:
                self.db.execute("""
                    UPDATE review_queue
                    SET status = 'pending'
                    WHERE status = 'deferred'
                """)
                self.redis.delete("moderation:triage_mode")

    def _deactivate_all_overflow(self):
        """完全退出溢出模式"""
        self.redis.delete("moderation:batch_mode")
        self.redis.delete("moderation:batch_size")
        self.redis.set("moderation:lock_timeout_min", "30")

        self.db.execute("""
            UPDATE review_queue SET status = 'pending' WHERE status = 'deferred'
        """)

        self.alert.send_info(
            title="审核队列溢出已解除",
            message="所有溢出处理措施已恢复正常"
        )
```

### 场景 13：误杀率骤升（模型漂移导致误判飙升）

```python
class FalsePositiveSpikeDetector:
    """
    误杀率飙升检测与自动处置

    场景：模型更新/数据漂移导致误杀率从1%升至5%+
    检测手段：申诉率、申诉通过率、审核拒绝率异常监控
    核心策略：自动检测 → 灰度回滚 → 根因分析 → 安全再上线
    """

    def __init__(self, db, redis, alert_service, model_registry):
        self.db = db
        self.redis = redis
        self.alert = alert_service
        self.model_registry = model_registry

        self.REJECT_RATE_NORMAL = 0.05
        self.REJECT_RATE_SPIKE = 0.15
        self.APPEAL_RATE_NORMAL = 0.02
        self.APPEAL_RATE_SPIKE = 0.08
        self.APPEAL_APPROVE_RATE_NORMAL = 0.5
        self.APPEAL_APPROVE_RATE_SPIKE = 0.9
        self.DETECTION_WINDOW_MIN = 10

    def check_false_positive_spike(self) -> Optional[Dict]:
        """
        检测误杀率是否飙升（每5分钟执行一次）

        检测逻辑：
        1. 审核拒绝率 > 15% → 可能在误杀
        2. 用户申诉率 > 8% → 确认用户大量不满意
        3. 申诉通过率 > 90% → 几乎全部是误杀
        任一条件触发 → 进入处置流程
        """
        window_start = datetime.now() - timedelta(minutes=self.DETECTION_WINDOW_MIN)

        ai_stats = self.db.query_one("""
            SELECT
                COUNT(*) as total,
                SUM(CASE WHEN is_violation = TRUE THEN 1 ELSE 0 END) as rejected
            FROM moderation_results
            WHERE stage = 'ai' AND created_at > %s
        """, window_start)

        reject_rate = ai_stats["rejected"] / max(ai_stats["total"], 1)

        appeal_stats = self.db.query_one("""
            SELECT
                COUNT(DISTINCT r.content_id) as reviewed_count,
                COUNT(DISTINCT a.content_id) as appeal_count
            FROM moderation_results r
            LEFT JOIN appeals a ON r.content_id = a.content_id
            WHERE r.stage = 'ai' AND r.created_at > %s AND r.is_violation = TRUE
        """, window_start)

        appeal_rate = appeal_stats["appeal_count"] / max(appeal_stats["reviewed_count"], 1)

        appeal_approve_stats = self.db.query_one("""
            SELECT
                COUNT(*) as total,
                SUM(CASE WHEN appeal_status = 'approved' THEN 1 ELSE 0 END) as approved
            FROM appeals
            WHERE created_at > %s AND appeal_status IN ('approved', 'rejected')
        """, window_start)

        approve_rate = (appeal_approve_stats["approved"] /
                       max(appeal_approve_stats["total"], 1))

        spike_detected = (
            reject_rate > self.REJECT_RATE_SPIKE or
            appeal_rate > self.APPEAL_RATE_SPIKE or
            (appeal_rate > self.APPEAL_RATE_NORMAL and
             approve_rate > self.APPEAL_APPROVE_RATE_SPIKE)
        )

        if spike_detected:
            return {
                "spike_detected": True,
                "reject_rate": reject_rate,
                "appeal_rate": appeal_rate,
                "approve_rate": approve_rate,
                "detection_time": datetime.now().isoformat(),
            }
        return None

    def handle_spike(self, spike_info: Dict):
        """误杀飙升处置流程：5分钟内止损，1小时内定位根因"""
        # Step 1: 紧急止血——回滚模型到上一版本
        current_version = self.model_registry.get_current_version()
        previous_version = self.model_registry.get_previous_version()

        if previous_version:
            self.model_registry.rollback(previous_version)
            self.alert.send_critical(
                title="误杀率飙升——已自动回滚模型",
                message=f"当前版本 {current_version} → 回滚至 {previous_version}\n"
                        f"拒绝率: {spike_info['reject_rate']:.1%}, "
                        f"申诉率: {spike_info['appeal_rate']:.1%}, "
                        f"申诉通过率: {spike_info['approve_rate']:.1%}"
            )

        # Step 2: 批量重审误杀内容
        self._schedule_re_review()

        # Step 3: 通知受影响用户
        self._notify_affected_users()

        # Step 4: 启动根因分析
        self._start_root_cause_analysis(current_version, spike_info)

    def _schedule_re_review(self):
        """将近期被拒绝的内容重新入队审核"""
        cutoff = datetime.now() - timedelta(hours=1)
        affected = self.db.query("""
            SELECT content_id FROM moderation_results
            WHERE stage = 'ai' AND is_violation = TRUE AND created_at > %s
        """, cutoff)

        for item in affected:
            self.db.execute("""
                INSERT INTO review_queue
                (content_id, content_type, ai_confidence, ai_reason,
                 priority, sla_deadline, enqueued_at)
                SELECT r.content_id, c.content_type, 0.5, r.reason, 150,
                       DATE_ADD(NOW(), INTERVAL 30 MINUTE), NOW()
                FROM moderation_results r
                JOIN contents c ON r.content_id = c.content_id
                WHERE r.content_id = %s AND r.stage = 'ai'
                ON DUPLICATE KEY UPDATE priority = 150
            """, item["content_id"])

        self.alert.send_info(f"已将 {len(affected)} 条误杀内容重新入队审核")

    def _notify_affected_users(self):
        """通知被误杀内容的发布者"""
        self.db.execute("""
            UPDATE contents
            SET publish_status = 'pending'
            WHERE content_id IN (
                SELECT content_id FROM moderation_results
                WHERE stage = 'ai' AND is_violation = TRUE
                  AND created_at > DATE_SUB(NOW(), INTERVAL 1 HOUR)
            ) AND publish_status = 'rejected'
        """)

    def _start_root_cause_analysis(self, version: str, spike_info: Dict):
        """根因分析——自动定位误杀原因"""
        false_positives = self.db.query("""
            SELECT r.content_id, r.reason, r.all_scores, r.confidence,
                   c.content_text, c.content_type
            FROM moderation_results r
            JOIN contents c ON r.content_id = c.content_id
            WHERE r.stage = 'ai' AND r.is_violation = TRUE
              AND r.created_at > DATE_SUB(NOW(), INTERVAL 1 HOUR)
            ORDER BY r.confidence DESC LIMIT 500
        """)

        reason_distribution = {}
        for fp in false_positives:
            reason = fp["reason"]
            reason_distribution[reason] = reason_distribution.get(reason, 0) + 1

        if reason_distribution:
            dominant_reason = max(reason_distribution, key=reason_distribution.get)
            dominant_pct = reason_distribution[dominant_reason] / len(false_positives)
            likely_cause = (
                f"模型对 {dominant_reason} 类内容判断偏向违规，"
                "可能是训练数据中该类违规样本比例过高导致模型过度敏感"
                if dominant_pct > 0.6 else
                "误杀分布较均匀，可能是模型整体阈值设置过低或数据分布漂移"
            )
        else:
            dominant_reason = "unknown"
            dominant_pct = 0
            likely_cause = "样本不足，无法定位"

        self.alert.send_info(
            title="误杀根因分析完成",
            message=f"主要原因: {dominant_reason}类误杀 ({dominant_pct:.0%})\n"
                    f"推测原因: {likely_cause}"
        )
```

### 场景 14：新型违规模式出现（AI 零检出）

```python
class NovelViolationDetector:
    """
    新型违规模式检测器

    场景：出现全新的违规模式（如新型诈骗话术、新色情变体），
    AI 模型训练数据中无此类样本 → 置信度为0 → 全部漏检
    检测手段：用户举报聚类 + 人工审核反馈 + 内容相似度聚类
    核心策略：检测 → 规则补丁（分钟级） → 标注（小时级） → 模型更新（天级）
    """

    def __init__(self, db, redis, alert_service, rule_filter):
        self.db = db
        self.redis = redis
        self.alert = alert_service
        self.rule_filter = rule_filter

    def detect_novel_violations(self) -> List[Dict]:
        """
        检测新型违规模式

        方法1：举报率异常检测——某类内容举报率突然升高
        方法2：举报内容聚类——举报内容中存在共性模式
        方法3：人工审核反馈——审核员标记"疑似新类型"
        """
        patterns = []

        # 方法1：举报率异常检测
        report_anomalies = self._detect_report_rate_anomaly()
        patterns.extend(report_anomalies)

        # 方法2：举报内容文本聚类
        cluster_patterns = self._cluster_reported_content()
        patterns.extend(cluster_patterns)

        # 方法3：人工标记的新类型
        human_flagged = self._get_human_flagged_novels()
        patterns.extend(human_flagged)

        return patterns

    def _detect_report_rate_anomaly(self) -> List[Dict]:
        """
        举报率异常检测

        逻辑：按小时统计每类内容的举报率，
        如果某类内容举报率突然超过基线的3倍 → 标记为疑似新型违规
        """
        current = self.db.query("""
            SELECT
                c.content_type,
                COUNT(DISTINCT c.content_id) as total_count,
                COUNT(DISTINCT r.content_id) as report_count,
                COUNT(DISTINCT r.content_id) / COUNT(DISTINCT c.content_id) as report_rate
            FROM contents c
            LEFT JOIN reports r ON c.content_id = r.content_id
            WHERE c.created_at > DATE_SUB(NOW(), INTERVAL 1 HOUR)
            GROUP BY c.content_type
        """)

        baseline = self.db.query("""
            SELECT content_type, AVG(report_rate) as avg_report_rate,
                   STDDEV(report_rate) as std_report_rate
            FROM hourly_report_stats
            WHERE hour_of_day = HOUR(NOW())
              AND date > DATE_SUB(NOW(), INTERVAL 7 DAY)
            GROUP BY content_type
        """)
        baseline_map = {r["content_type"]: r for r in baseline}

        anomalies = []
        for item in current:
            bl = baseline_map.get(item["content_type"], {})
            avg_rate = bl.get("avg_report_rate", 0.01)
            std_rate = bl.get("std_report_rate", 0.005)

            if item["report_rate"] > avg_rate + 3 * std_rate:
                anomalies.append({
                    "detection_method": "report_rate_anomaly",
                    "content_type": item["content_type"],
                    "current_rate": item["report_rate"],
                    "baseline_rate": avg_rate,
                    "anomaly_ratio": item["report_rate"] / max(avg_rate, 0.001),
                    "affected_count": item["report_count"],
                })

        return anomalies

    def _cluster_reported_content(self) -> List[Dict]:
        """
        举报内容文本聚类

        逻辑：收集最近1小时被举报的内容文本 → TF-IDF + DBSCAN 聚类
        聚类簇大小 > 10 → 可能是新型违规模式
        """
        reported = self.db.query("""
            SELECT c.content_id, c.content_text, c.content_type
            FROM contents c
            JOIN reports r ON c.content_id = r.content_id
            WHERE r.created_at > DATE_SUB(NOW(), INTERVAL 1 HOUR)
            LIMIT 1000
        """)

        if len(reported) < 10:
            return []

        from sklearn.feature_extraction.text import TfidfVectorizer
        from sklearn.cluster import DBSCAN

        texts = [r["content_text"] or "" for r in reported]
        vectorizer = TfidfVectorizer(max_features=500)
        tfidf_matrix = vectorizer.fit_transform(texts)

        clustering = DBSCAN(eps=0.5, min_samples=5, metric="cosine").fit(tfidf_matrix)
        cluster_labels = clustering.labels_

        patterns = []
        for label in set(cluster_labels):
            if label == -1:
                continue
            mask = cluster_labels == label
            count = int(mask.sum())
            if count >= 10:
                cluster_texts = [texts[i] for i in range(len(texts)) if mask[i]]

                from collections import Counter
                word_counter = Counter()
                for t in cluster_texts:
                    words = t.split()
                    word_counter.update(words)

                top_keywords = [w for w, c in word_counter.most_common(10) if len(w) >= 2]

                patterns.append({
                    "detection_method": "content_clustering",
                    "cluster_id": int(label),
                    "cluster_size": count,
                    "top_keywords": top_keywords,
                    "sample_texts": cluster_texts[:3],
                })

        return patterns

    def _get_human_flagged_novels(self) -> List[Dict]:
        """获取审核员标记的疑似新类型"""
        flagged = self.db.query("""
            SELECT content_id, reviewer_id, flag_reason
            FROM reviewer_novel_flags
            WHERE created_at > DATE_SUB(NOW(), INTERVAL 24 HOUR)
              AND status = 'pending_review'
        """)
        return [{
            "detection_method": "human_flagged",
            "content_id": f["content_id"],
            "reviewer_id": f["reviewer_id"],
            "reason": f["flag_reason"],
        } for f in flagged]

    def handle_novel_violation(self, pattern: Dict):
        """
        处理新型违规模式——三步走

        规则补丁(10分钟) → 标注(1小时) → 模型更新(2-4小时)
        """
        # Step 1: 紧急规则补丁（分钟级生效）
        if pattern["detection_method"] == "content_clustering":
            keywords = pattern.get("top_keywords", [])
            if keywords:
                self._deploy_emergency_rule(keywords, pattern)

        # Step 2: 创建紧急标注任务
        sample_content_ids = self._collect_label_samples(pattern)
        self._create_labeling_task(sample_content_ids, pattern)

        # Step 3: 告警通知运营团队
        self.alert.send_critical(
            title="检测到疑似新型违规模式",
            message=f"检测方法: {pattern['detection_method']}\n"
                    f"关键词: {pattern.get('top_keywords', [])}\n"
                    f"影响内容数: {pattern.get('cluster_size', pattern.get('affected_count', 0))}\n"
                    f"已部署紧急规则补丁"
        )

    def _deploy_emergency_rule(self, keywords: List[str], pattern: Dict):
        """部署紧急规则补丁到规则引擎"""
        rule_id = f"emergency_{datetime.now().strftime('%Y%m%d%H%M%S')}"

        for keyword in keywords:
            self.db.execute("""
                INSERT INTO blacklist_keywords
                (keyword, category, rule_id, is_emergency, created_at)
                VALUES (%s, 'novel_violation', %s, TRUE, NOW())
            """, keyword, rule_id)

        # 通知规则引擎热加载
        self.redis.publish("moderation:rules:reload", json.dumps({
            "rule_id": rule_id,
            "keywords": keywords,
            "reason": "novel_violation_emergency",
        }))

    def _collect_label_samples(self, pattern: Dict) -> List[str]:
        """收集需要标注的样本"""
        if pattern["detection_method"] == "report_rate_anomaly":
            content_type = pattern["content_type"]
            samples = self.db.query("""
                SELECT content_id FROM contents
                WHERE content_type = %s
                  AND created_at > DATE_SUB(NOW(), INTERVAL 2 HOUR)
                ORDER BY created_at DESC LIMIT 100
            """, content_type)
        elif pattern["detection_method"] == "content_clustering":
            keyword = pattern["top_keywords"][0] if pattern.get("top_keywords") else ""
            samples = self.db.query("""
                SELECT content_id FROM contents
                WHERE content_text LIKE %s LIMIT 100
            """, f"%{keyword}%")
        else:
            samples = []
        return [s["content_id"] for s in samples]

    def _create_labeling_task(self, content_ids: List[str], pattern: Dict):
        """创建标注任务"""
        self.db.execute("""
            INSERT INTO labeling_tasks
            (task_type, priority, content_ids, description, created_at)
            VALUES ('novel_violation', 'urgent', %s, %s, NOW())
        """, json.dumps(content_ids),
            f"新型违规模式标注: {pattern.get('detection_method', 'unknown')}"
        )
```

**新型违规检测与处置时间线：**

| 时间 | 动作 | 效果 |
|------|------|------|
| T+0 | 举报率异常检测触发 | 发现疑似新型违规 |
| T+5min | 举报内容聚类完成 | 提取违规模式关键词 |
| T+10min | 紧急规则补丁部署 | 规则引擎可拦截后续同类内容 |
| T+30min | 人工标注 50-100 条样本 | 生成训练数据 |
| T+2h | 模型增量训练完成 | AI 模型可识别新型违规 |
| T+4h | 新模型灰度上线 | 10% 流量验证 |
| T+24h | 新模型全量上线 | 完整覆盖新型违规 |
| T+72h | 补充标注 500+ 样本 | 正式重训模型 |
| T+1w | 规则补丁评估 | 确认规则可移除（模型已覆盖） |

## 内容审核申诉系统完整实现

```python
class ContentAppealService:
    """内容审核申诉：提交 → 人工复审 → 结果通知"""

    def submit_appeal(self, user_id, content_id, reason, evidence=None):
        """提交申诉"""
        # 1. 检查申诉条件
        content = self.db.get_content(content_id)
        if content["moderation_result"] != "rejected":
            return {"status": "invalid", "reason": "内容未被拒绝，无需申诉"}

        # 检查申诉次数限制（同一内容最多 2 次）
        existing = self.db.count("content_appeals",
            content_id=content_id, user_id=user_id)
        if existing >= 2:
            return {"status": "rejected", "reason": "申诉次数已达上限"}

        appeal_id = str(uuid4())
        self.db.insert("content_appeals", {
            "appeal_id": appeal_id,
            "content_id": content_id,
            "user_id": user_id,
            "reason": reason,
            "evidence": json.dumps(evidence or []),
            "status": "pending_review",
            "priority": "high" if content.get("business_impact") == "high" else "normal",
            "created_at": now()
        })

        # 分配审核员
        reviewer = self._assign_reviewer(content["category"])
        self.db.update("content_appeals",
            {"assigned_reviewer": reviewer, "assigned_at": now()},
            {"appeal_id": appeal_id})

        return {"appeal_id": appeal_id, "status": "submitted",
                "estimated_review_hours": 24}

    def review_appeal(self, appeal_id, reviewer_id, decision, comment):
        """审核员复审"""
        appeal = self.db.get_appeal(appeal_id)

        if appeal["assigned_reviewer"] != reviewer_id:
            raise PermissionDeniedError("非指定审核员")

        if decision == "overturn":
            # 推翻原判 → 恢复内容
            self.db.update("content",
                {"moderation_result": "approved", "appeal_overturned": True},
                {"id": appeal["content_id"]})
            self._notify_user(appeal["user_id"],
                "您的申诉已通过，内容已恢复展示")

        elif decision == "uphold":
            # 维持原判
            self._notify_user(appeal["user_id"],
                "您的申诉未通过，内容仍不可展示")

        self.db.update("content_appeals", {
            "status": decision,
            "reviewer_comment": comment,
            "reviewed_at": now()
        }, {"appeal_id": appeal_id})

        # 更新审核模型反馈
        self._update_model_feedback(appeal["content_id"], decision)

    def _assign_reviewer(self, category):
        """分配审核员（按专业领域 + 工作量均衡）"""
        reviewers = self.db.query(
            "SELECT id, category_expertise, active_appeals FROM reviewers "
            "WHERE ? = ANY(category_expertise) AND active = 1 "
            "ORDER BY active_appeals ASC LIMIT 1", category)
        if reviewers:
            return reviewers[0]["id"]
        # 无专业审核员 → 分配给工作量最少的
        fallback = self.db.query(
            "SELECT id FROM reviewers WHERE active = 1 "
            "ORDER BY active_appeals ASC LIMIT 1")
        return fallback[0]["id"] if fallback else None

    def _update_model_feedback(self, content_id, appeal_decision):
        """更新模型反馈（用于模型迭代）"""
        self.db.insert("moderation_model_feedback", {
            "content_id": content_id,
            "original_decision": "rejected",
            "appeal_decision": appeal_decision,
            "is_false_positive": appeal_decision == "overturn",
            "created_at": now()
        })

        # 如果误判率 > 5% → 触发模型重训练
        recent_fp_rate = self.db.query_one(
            "SELECT SUM(CASE WHEN is_false_positive THEN 1 ELSE 0 END) / COUNT(*) as rate "
            "FROM moderation_model_feedback "
            "WHERE created_at > NOW() - INTERVAL 7 DAY")["rate"] or 0

        if recent_fp_rate > 0.05:
            self.alert(f"审核模型误判率 {recent_fp_rate:.1%}，建议重训练")
```

## 审核效率分析

```python
class ModerationEfficiencyAnalyzer:
    """审核效率分析：SLA + 吞吐 + 质量"""

    def get_efficiency_metrics(self, period="daily"):
        """获取效率指标"""
        days = 1 if period == "daily" else 7
        return {
            "throughput": self._throughput(days),
            "sla_compliance": self._sla_compliance(days),
            "quality_metrics": self._quality_metrics(days),
            "appeal_overturn_rate": self._appeal_overturn_rate(days),
            "queue_depth": self._queue_depth(),
        }

    def _throughput(self, days):
        """审核吞吐量"""
        return self.db.query(
            "SELECT DATE(reviewed_at) as date, "
            "COUNT(*) as reviewed, "
            "COUNT(*) / 8 as reviews_per_hour_per_reviewer "
            "FROM moderation_results "
            "WHERE reviewed_at > NOW() - INTERVAL %s DAY "
            "AND reviewer_id IS NOT NULL "
            "GROUP BY DATE(reviewed_at)", days)

    def _sla_compliance(self, days):
        """SLA 合规（30 分钟内审核完成）"""
        return self.db.query_one(
            "SELECT "
            "SUM(CASE WHEN TIMESTAMPDIFF(MINUTE, submitted_at, reviewed_at) <= 30 "
            "THEN 1 ELSE 0 END) / COUNT(*) as compliance_rate "
            "FROM moderation_results "
            "WHERE reviewed_at > NOW() - INTERVAL %s DAY", days)

    def _appeal_overturn_rate(self, days):
        """申诉推翻率（反映审核质量）"""
        return self.db.query_one(
            "SELECT SUM(CASE WHEN status = 'overturn' THEN 1 ELSE 0 END) "
            "/ COUNT(*) as overturn_rate "
            "FROM content_appeals "
            "WHERE reviewed_at > NOW() - INTERVAL %s DAY", days)

    def _queue_depth(self):
        """当前队列深度"""
        return {
            "ai_pending": self.db.count("moderation_queue", status="ai_processing"),
            "human_pending": self.db.count("moderation_queue", status="human_review"),
            "appeal_pending": self.db.count("content_appeals", status="pending_review"),
        }
```

## 异常场景补充

### 场景：申诉系统被滥用

```
触发：恶意用户大量提交无理申诉 → 占用审核资源 → 正常申诉延迟
检测：
  1. 单用户申诉频率 > 5 次/天 → 异常
  2. 申诉推翻率 < 5% → 大量无效申诉
处理：
  1. 限制单用户申诉频率（5 次/天）
  2. 低信用用户申诉需预审
  3. 恶意申诉 → 封禁申诉权限
预防：申诉频率限制 + 信用评分 + 滥用检测
```

### 场景：审核员分配不均

```
触发：某领域审核员不足 → 申诉积压 → SLA 违规
检测：
  1. 某类别申诉等待时间 > 24 小时 → 严重
  2. 审核员工作量方差大 → 分配不均
处理：
  1. 跨领域审核员紧急支援
  2. 临时启用 AI 辅助初审
  3. 招募新审核员
预防：工作量自动均衡 + 跨领域培训 + AI 辅助
```

## 审核模型生命周期完整实现

```python
class ModerationModelLifecycle:
    """审核模型生命周期：训练 → 部署 → 监控 → 重训练"""

    def train_model(self, dataset_id, model_config):
        """训练审核模型"""
        # 1. 数据准备（处理类别不平衡）
        dataset = self.db.get_dataset(dataset_id)
        pos_count = dataset["positive_count"]
        neg_count = dataset["negative_count"]

        # SMOTE 过采样少数类
        if pos_count / neg_count < 0.3:
            dataset = self._apply_smote(dataset, target_ratio=0.5)

        # 2. 训练/验证/测试分割
        splits = self._split_dataset(dataset, ratios=[0.7, 0.15, 0.15])

        # 3. 模型训练
        model = self.ml_framework.train(
            train_data=splits["train"],
            val_data=splits["val"],
            config=model_config)

        # 4. 评估
        eval_result = self.ml_framework.evaluate(model, splits["test"])

        # 5. 注册模型
        model_id = str(uuid4())
        self.db.insert("moderation_models", {
            "model_id": model_id,
            "version": model_config["version"],
            "dataset_id": dataset_id,
            "metrics": json.dumps(eval_result),
            "status": "trained",
            "trained_at": now()
        })

        return {"model_id": model_id, "metrics": eval_result}

    def deploy_model(self, model_id, strategy="shadow"):
        """部署模型（shadow → canary → production）"""
        if strategy == "shadow":
            # 影子模式：运行但不影响结果
            self.redis.set("mod:model:shadow", model_id)
            return {"status": "shadow", "model_id": model_id}

        elif strategy == "canary":
            # 金丝雀：10% 流量使用新模型
            self.redis.set("mod:model:canary", model_id)
            self.redis.set("mod:canary:percentage", 10)
            return {"status": "canary_10pct", "model_id": model_id}

        elif strategy == "production":
            # 全量上线
            self.redis.set("mod:model:production", model_id)
            return {"status": "production", "model_id": model_id}

    def monitor_model_drift(self, model_id):
        """监控模型漂移（PSI 指标）"""
        # 比较当前预测分布 vs 训练分布
        current_dist = self._get_prediction_distribution(model_id, days=7)
        training_dist = self._get_training_distribution(model_id)

        psi = self._calculate_psi(current_dist, training_dist)

        if psi > 0.2:
            # 显著漂移 → 触发重训练
            self.alert(f"模型 {model_id} 漂移 PSI={psi:.3f}，建议重训练")
            self._trigger_retraining(model_id)

        return {"psi": round(psi, 4), "drift_detected": psi > 0.2}

    def _calculate_psi(self, expected, actual):
        """计算 PSI（Population Stability Index）"""
        psi = 0
        for e, a in zip(expected, actual):
            if e > 0 and a > 0:
                psi += (a - e) * math.log(a / e)
        return psi
```

## 审核策略管理

```python
class ModerationPolicyManager:
    """审核策略管理：规则 + 版本 + A/B 测试"""

    def create_policy(self, name, rules, default_action="review"):
        """创建审核策略"""
        policy_id = str(uuid4())
        self.db.insert("moderation_policies", {
            "policy_id": policy_id,
            "name": name,
            "version": 1,
            "rules": json.dumps(rules),
            "default_action": default_action,
            "status": "draft",
            "created_at": now()
        })
        return policy_id

    def apply_policy(self, content, policy_id):
        """应用策略审核内容"""
        policy = self.db.get_policy(policy_id)
        rules = json.loads(policy["rules"])

        for rule in rules:
            if rule["type"] == "keyword":
                if any(kw in content["text"] for kw in rule["keywords"]):
                    return {"action": rule["action"], "rule": rule["name"],
                            "confidence": 1.0}

            elif rule["type"] == "regex":
                if re.search(rule["pattern"], content["text"]):
                    return {"action": rule["action"], "rule": rule["name"],
                            "confidence": 1.0}

            elif rule["type"] == "ml_threshold":
                score = self.ml_service.get_score(content, rule["model_id"])
                if score > rule["threshold"]:
                    return {"action": rule["action"], "rule": rule["name"],
                            "confidence": score}

        return {"action": policy["default_action"], "rule": "default"}

    def ab_test_policy(self, new_policy_id, traffic_pct=10):
        """A/B 测试新策略"""
        self.redis.set("mod:ab:new_policy", new_policy_id)
        self.redis.set("mod:ab:traffic_pct", traffic_pct)

        # 监控对比指标
        self.scheduler.schedule(
            run_date=now() + timedelta(days=7),
            task=self._evaluate_ab_test,
            args={"new_policy_id": new_policy_id})

    def emergency_update(self, rule_name, action):
        """紧急策略更新（<5 分钟生效）"""
        # 直接写入 Redis（绕过 DB 延迟）
        self.redis.set(f"mod:emergency:{rule_name}", action)
        self.alert(f"紧急策略更新: {rule_name} → {action}")

        # 异步持久化到 DB
        self.db.insert("policy_emergency_updates", {
            "rule_name": rule_name, "action": action,
            "applied_at": now()
        })
```

## 异常场景补充

### 场景：模型重训练引入偏见

```
触发：新训练数据偏向某类内容 → 模型对该类内容误判率上升 → 不公平
检测：
  1. 某内容类别误判率突增 → 偏见
  2. 不同群体审核结果差异显著 → 不公平
处理：
  1. 回滚到上一版本模型
  2. 审查训练数据分布
  3. 重新平衡训练数据
预防：训练数据分布检查 + 公平性评估 + 渐进部署
```

### 场景：紧急策略更新范围过大

```
触发：紧急规则过于宽泛 → 大量正常内容被误判 → 用户投诉
检测：
  1. 审核拒绝率突增 → 策略过严
  2. 用户投诉量飙升 → 误判
处理：
  1. 立即回滚紧急策略
  2. 收窄规则范围
  3. 重新灰度验证
预防：紧急更新范围限制 + 自动回滚 + 灰度验证
```

## 内容审核智能分发完整实现

```python
class ModerationRoutingService:
    """审核智能分发：按难度 + 语言 + 类别分配给最合适的审核员"""

    def route_content(self, content_id):
        """智能路由内容到审核员"""
        content = self.db.get_content(content_id)
        ai_result = content.get("ai_moderation_result", {})

        # 1. 确定审核难度
        difficulty = self._assess_difficulty(content, ai_result)
        # easy: AI 高置信度且无争议
        # medium: AI 中置信度或边缘内容
        # hard: AI 低置信度、争议性内容、法律风险

        # 2. 确定所需专业
        expertise = self._determine_expertise(content)
        # 法律、医疗、金融、政治等需要专业审核员

        # 3. 查找可用审核员
        candidates = self.db.query(
            "SELECT id, expertise_areas, current_queue_size, "
            "avg_review_time_seconds, accuracy_rate "
            "FROM moderators WHERE status = 'online' "
            "AND current_queue_size < max_queue_size")

        # 4. 评分排序
        scored = []
        for mod in candidates:
            score = 0
            # 专业匹配
            if expertise and expertise in mod["expertise_areas"]:
                score += 50
            # 难度匹配（困难内容给资深审核员）
            if difficulty == "hard" and mod["accuracy_rate"] > 0.95:
                score += 30
            elif difficulty == "easy":
                score += 10  # 简单内容分配给新手
            # 工作量均衡
            score -= mod["current_queue_size"] * 2
            # 速度
            if mod["avg_review_time_seconds"] < 30:
                score += 10
            scored.append((mod["id"], score))

        scored.sort(key=lambda x: x[1], reverse=True)

        if scored:
            assigned_to = scored[0][0]
            self.db.update("content_moderation_queue",
                {"assigned_moderator": assigned_to,
                 "difficulty": difficulty,
                 "expertise_required": expertise,
                 "assigned_at": now()},
                {"content_id": content_id})
            return {"assigned_to": assigned_to, "difficulty": difficulty}

        # 无可用审核员 → 加入公共队列
        return {"assigned_to": None, "difficulty": difficulty,
                "status": "queued"}

    def _assess_difficulty(self, content, ai_result):
        """评估审核难度"""
        confidence = ai_result.get("confidence", 0)
        has_dispute = ai_result.get("user_appealed", False)
        legal_risk = ai_result.get("legal_risk", False)

        if legal_risk or has_dispute:
            return "hard"
        elif confidence < 0.6:
            return "hard"
        elif confidence < 0.85:
            return "medium"
        else:
            return "easy"

    def _determine_expertise(self, content):
        """确定所需专业领域"""
        category = content.get("category", "")
        expertise_map = {
            "legal": "法律", "medical": "医疗", "finance": "金融",
            "politics": "政治", "adult": "成人内容", "violence": "暴力",
        }
        for keyword, expertise in expertise_map.items():
            if keyword in content.get("text", "").lower() or keyword in category:
                return expertise
        return None
```

## 审核质量抽检系统

```python
class ModerationQualityInspector:
    """审核质量抽检：随机抽检 + 一致性检查 + 质量分"""

    def random_inspection(self, inspector_id, sample_size=20):
        """随机抽检已审核内容"""
        # 随机选取最近 24 小时的审核结果
        samples = self.db.query(
            "SELECT * FROM moderation_results "
            "WHERE reviewed_at > NOW() - INTERVAL 24 HOUR "
            "ORDER BY RAND() LIMIT %s", sample_size)

        inspection_results = []
        for sample in samples:
            # 独立审核
            my_decision = self._review_content(sample["content_id"])

            # 对比原审核结果
            agree = my_decision == sample["decision"]
            inspection_results.append({
                "content_id": sample["content_id"],
                "original_moderator": sample["moderator_id"],
                "original_decision": sample["decision"],
                "inspector_decision": my_decision,
                "agree": agree
            })

        agreement_rate = sum(1 for r in inspection_results if r["agree"]) / len(inspection_results)

        self.db.insert("quality_inspections", {
            "inspection_id": str(uuid4()),
            "inspector_id": inspector_id,
            "sample_size": sample_size,
            "agreement_rate": round(agreement_rate, 3),
            "results": json.dumps(inspection_results),
            "inspected_at": now()
        })

        # 一致性低于 90% → 触发审核员培训
        if agreement_rate < 0.9:
            self._trigger_moderator_retraining(inspection_results)

        return {"agreement_rate": round(agreement_rate, 3),
                "sample_size": sample_size}

    def calculate_moderator_quality_score(self, moderator_id):
        """计算审核员质量分"""
        # 准确率（基于抽检）
        inspections = self.db.query(
            "SELECT agreement_rate FROM quality_inspections "
            "WHERE original_moderator = %s "
            "AND inspected_at > NOW() - INTERVAL 30 DAY",
            moderator_id)
        avg_agreement = statistics.mean([i["agreement_rate"] for i in inspections]) if inspections else 1.0

        # 速度（平均审核时间）
        avg_time = self.db.query_one(
            "SELECT AVG(TIMESTAMPDIFF(SECOND, assigned_at, reviewed_at)) as avg "
            "FROM moderation_results WHERE moderator_id = %s "
            "AND reviewed_at > NOW() - INTERVAL 30 DAY",
            moderator_id)["avg"] or 30
        speed_score = max(0, 100 - (avg_time - 15) * 2)  # 15 秒为标准

        # 综合质量分
        quality_score = avg_agreement * 70 + speed_score / 100 * 30

        return {"quality_score": round(quality_score, 1),
                "agreement_rate": round(avg_agreement, 3),
                "avg_review_seconds": round(avg_time, 1)}
```

## 异常场景补充

### 场景：审核智能分发导致某些审核员过载

```
触发：所有困难内容都路由给资深审核员 → 资深审核员积压 → SLA 违规
检测：
  1. 个别审核员队列深度 > 50 → 过载
  2. 审核员队列深度方差大 → 分配不均
处理：
  1. 设置每位审核员最大队列深度
  2. 困难内容溢出时分配给中等水平审核员
  3. 加入审核员轮转机制
预防：最大队列限制 + 负载均衡 + 审核员轮转
```

### 场景：质量抽检引发审核员不满

```
触发：抽检不一致率 > 15% → 要求审核员培训 → 审核员认为标准不明确
检测：
  1. 审核员对抽检结果提出异议 → 标准模糊
  2. 同一内容由不同检查员审核结果不一致 → 标准问题
处理：
  1. 审核标准文档化 + 定期更新
  2. 抽检由至少 2 名检查员独立审核
  3. 争议内容 → 委员会裁决
预防：标准文档化 + 双人抽检 + 委员会裁决机制
```

## 内容审核申诉系统完整实现

```python
class ModerationAppealService:
    """审核申诉系统：申诉提交 + 复审 + 恢复"""

    def submit_appeal(self, user_id, content_id, reason, evidence=None):
        """提交申诉"""
        # 1. 检查是否可以申诉
        content = self.db.get_content(content_id)
        if content["status"] not in ["rejected", "removed"]:
            return {"status": "not_appealable"}

        # 2. 检查申诉次数限制
        existing = self.db.count("moderation_appeals",
            content_id=content_id, user_id=user_id)
        if existing >= 2:
            return {"status": "limit_reached", "message": "每条内容最多申诉 2 次"}

        # 3. 创建申诉
        appeal_id = str(uuid4())
        self.db.insert("moderation_appeals", {
            "appeal_id": appeal_id,
            "content_id": content_id,
            "user_id": user_id,
            "reason": reason,
            "evidence": json.dumps(evidence or {}),
            "status": "pending",
            "priority": "high" if content["status"] == "removed" else "normal",
            "created_at": now()
        })

        # 4. 临时恢复内容（申诉期间可见）
        if content["status"] == "removed":
            self.db.update("content",
                {"status": "appeal_pending"},
                {"id": content_id})

        # 5. 通知审核团队
        self.notification.send("moderation_team",
            f"新申诉: 内容 {content_id}, 原因: {reason}")

        return {"appeal_id": appeal_id, "status": "pending"}

    def review_appeal(self, appeal_id, reviewer_id, decision, note=None):
        """复审申诉"""
        appeal = self.db.get_appeal(appeal_id)
        content_id = appeal["content_id"]

        if decision == "uphold":
            # 维持原判 → 内容保持移除
            self.db.update("content",
                {"status": "removed"}, {"id": content_id})
            self.db.update("moderation_appeals",
                {"status": "rejected", "reviewer_id": reviewer_id,
                 "review_note": note, "reviewed_at": now()},
                {"appeal_id": appeal_id})
            self.notification.send(appeal["user_id"],
                f"您的申诉已被驳回，原因: {note}")

        elif decision == "overturn":
            # 推翻原判 → 恢复内容
            self.db.update("content",
                {"status": "active"}, {"id": content_id})
            self.db.update("moderation_appeals",
                {"status": "approved", "reviewer_id": reviewer_id,
                 "review_note": note, "reviewed_at": now()},
                {"appeal_id": appeal_id})
            self.notification.send(appeal["user_id"],
                "您的申诉已通过，内容已恢复")

            # 记录原审核员的误判（用于质量评估）
            original_moderator = self._get_original_moderator(content_id)
            self.db.insert("moderation_false_positives", {
                "content_id": content_id,
                "original_moderator": original_moderator,
                "appeal_id": appeal_id,
                "reviewed_by": reviewer_id,
                "created_at": now()
            })

        return {"appeal_id": appeal_id, "decision": decision}

    def get_appeal_analytics(self, period_days=30):
        """申诉分析"""
        total = self.db.count("moderation_appeals",
            created_at__gte=now()-timedelta(days=period_days))
        approved = self.db.count("moderation_appeals",
            status="approved",
            created_at__gte=now()-timedelta(days=period_days))
        avg_review_hours = self.db.query_one(
            "SELECT AVG(TIMESTAMPDIFF(HOUR, created_at, reviewed_at)) as avg "
            "FROM moderation_appeals "
            "WHERE reviewed_at IS NOT NULL "
            "AND created_at > NOW() - INTERVAL %s DAY", period_days)["avg"] or 0

        return {
            "total_appeals": total,
            "overturn_rate": round(approved / max(total, 1), 3),
            "avg_review_hours": round(avg_review_hours, 1),
            "false_positive_rate": round(approved / max(total, 1), 3)
        }
```

## 异常场景补充

### 场景：申诉被批量驳回

```
触发：审核员批量驳回申诉未逐一审查 → 合理申诉被误拒 → 用户不满
检测：
  1. 审核员驳回率 > 95% → 可能未认真审查
  2. 申诉审查时间 < 10 秒 → 可能批量操作
处理：
  1. 要求每个申诉至少审查 30 秒
  2. 驳回率异常 → 人工复核
  3. 审核员培训
预防：最低审查时间 + 驳回率监控 + 申诉抽样复核
```

### 场景：申诉期间内容被二次举报

```
触发：内容申诉待审中 → 又被其他用户举报 → 状态冲突
检测：
  1. 内容同时有 pending appeal 和 new report → 冲突
  2. 状态不一致 → 逻辑问题
处理：
  1. 新举报加入申诉审核队列（一并考虑）
  2. 如果新举报为不同原因 → 分别处理
  3. 申诉期间内容标记为"争议中"
预防：状态机管理 + 举报合并 + 争议标记
```

## 内容审核模型迭代管理完整实现

```python
class ModerationModelManagement:
    """审核模型迭代：版本管理 → 灰度上线 → A/B 对比 → 全量发布"""

    def deploy_model(self, model_id, version, deployment_strategy="canary"):
        """部署审核模型"""
        # 1. 验证模型
        validation = self._validate_model(model_id, version)
        if not validation["passed"]:
            return {"status": "validation_failed", "issues": validation["issues"]}

        if deployment_strategy == "canary":
            # 灰度：5% 流量
            self.redis.set(f"model:canary:{model_id}", json.dumps({
                "version": version, "traffic_pct": 5
            }))
            return {"status": "canary_deployed", "traffic_pct": 5}

        elif deployment_strategy == "ab_test":
            # A/B 测试：50/50
            self.redis.set(f"model:ab:{model_id}", json.dumps({
                "control_version": self._get_current_version(model_id),
                "treatment_version": version,
                "traffic_split": 50
            }))
            return {"status": "ab_test_started", "split": 50}

    def compare_model_performance(self, model_id, control_version, treatment_version):
        """对比模型性能"""
        metrics = {}
        for version in [control_version, treatment_version]:
            results = self.db.query(
                "SELECT * FROM model_evaluation_results "
                "WHERE model_id = %s AND version = %s "
                "AND evaluated_at > NOW() - INTERVAL 24 HOUR",
                model_id, version)

            if results:
                # 计算核心指标
                total = len(results)
                correct = sum(1 for r in results if r["is_correct"])
                false_positives = sum(1 for r in results
                    if r["predicted"] == "reject" and r["actual"] == "accept")
                false_negatives = sum(1 for r in results
                    if r["predicted"] == "accept" and r["actual"] == "reject")

                metrics[version] = {
                    "accuracy": round(correct / max(total, 1), 4),
                    "precision": round(correct / max(correct + false_positives, 1), 4),
                    "recall": round(correct / max(correct + false_negatives, 1), 4),
                    "false_positive_rate": round(false_positives / max(total, 1), 4),
                    "false_negative_rate": round(false_negatives / max(total, 1), 4),
                    "avg_latency_ms": round(statistics.mean(r["latency_ms"] for r in results), 1),
                    "sample_size": total
                }

        return {"model_id": model_id, "metrics": metrics}

    def promote_model(self, model_id, version):
        """全量发布模型"""
        # 1. 最终检查
        current = self._get_current_version(model_id)
        comparison = self.compare_model_performance(model_id, current, version)

        treatment_metrics = comparison["metrics"].get(version, {})
        if treatment_metrics.get("accuracy", 0) < 0.9:
            return {"status": "rejected", "reason": "新模型准确率不足 90%"}

        # 2. 全量切换
        self.redis.set(f"model:active:{model_id}", version)
        self.redis.delete(f"model:canary:{model_id}")
        self.redis.delete(f"model:ab:{model_id}")

        # 3. 记录发布
        self.db.insert("model_deployments", {
            "deployment_id": str(uuid4()),
            "model_id": model_id,
            "from_version": current,
            "to_version": version,
            "deployed_at": now()
        })

        return {"status": "promoted", "from": current, "to": version}
```

## 异常场景补充

### 场景：模型灰度上线后准确率下降

```
触发：新模型灰度 5% → 准确率从 95% 降到 88% → 误判增加 → 用户投诉
检测：
  1. 灰度模型准确率 < 当前模型 5% → 回退
  2. 误判投诉率上升 → 模型问题
处理：
  1. 自动回退到旧模型
  2. 分析新模型误判案例
  3. 修复后重新灰度
预防：灰度自动回退 + 准确率监控 + 误判分析
```

### 场景：A/B 测试流量分配不均

```
触发：模型 A/B 测试 50/50 → 实际 A 收到 70% 流量 → 结果偏差
检测：
  1. 两版本样本量差异 > 10% → 分配不均
  2. A/B 测试结果不可信 → 需要重新测试
处理：
  1. 检查流量分配逻辑
  2. 修正后重新开始 A/B 测试
  3. 使用确定性哈希确保分配稳定
预防：确定性哈希 + 分配验证 + 样本量监控
```

## 内容审核效率监控完整实现

```python
class ModerationEfficiencyMonitor:
    """审核效率监控：审核时效 + 准确率 + 吞吐量"""

    SLA_TARGETS = {
        "urgent": {"max_wait_minutes": 5, "max_review_minutes": 3},
        "normal": {"max_wait_minutes": 30, "max_review_minutes": 10},
        "low": {"max_wait_minutes": 120, "max_review_minutes": 30},
    }

    def calculate_efficiency_metrics(self, period_hours=24):
        """计算审核效率指标"""
        # 1. 审核时效
        avg_wait = self.db.query_one(
            "SELECT AVG(TIMESTAMPDIFF(MINUTE, submitted_at, review_started_at)) as avg_wait "
            "FROM moderation_queue "
            "WHERE reviewed_at > NOW() - INTERVAL %s HOUR", period_hours)["avg_wait"] or 0

        avg_review_time = self.db.query_one(
            "SELECT AVG(TIMESTAMPDIFF(SECOND, review_started_at, reviewed_at)) as avg_time "
            "FROM moderation_queue "
            "WHERE reviewed_at > NOW() - INTERVAL %s HOUR", period_hours)["avg_time"] or 0

        # 2. SLA 达成率
        total = self.db.count("moderation_queue",
            reviewed_at__gte=now()-timedelta(hours=period_hours))

        sla_met = self.db.query_one(
            "SELECT COUNT(*) as count FROM moderation_queue "
            "WHERE reviewed_at > NOW() - INTERVAL %s HOUR "
            "AND TIMESTAMPDIFF(MINUTE, submitted_at, review_started_at) <= "
            "CASE priority WHEN 'urgent' THEN 5 WHEN 'normal' THEN 30 ELSE 120 END",
            period_hours)["count"] or 0

        sla_rate = sla_met / max(total, 1)

        # 3. 准确率（被申诉推翻的比例）
        total_decisions = self.db.count("moderation_decisions",
            decided_at__gte=now()-timedelta(hours=period_hours))
        overturned = self.db.count("moderation_appeals",
            status="approved",
            reviewed_at__gte=now()-timedelta(hours=period_hours))
        accuracy = 1 - (overturned / max(total_decisions, 1))

        # 4. 吞吐量
        throughput = self.db.query_one(
            "SELECT COUNT(*) as count, "
            "COUNT(DISTINCT reviewer_id) as reviewers "
            "FROM moderation_queue "
            "WHERE reviewed_at > NOW() - INTERVAL %s HOUR",
            period_hours)

        per_reviewer = throughput["count"] / max(throughput["reviewers"], 1)

        # 5. 积压
        backlog = self.db.count("moderation_queue", status="pending")

        return {
            "period_hours": period_hours,
            "avg_wait_minutes": round(avg_wait, 1),
            "avg_review_seconds": round(avg_review_time, 1),
            "sla_rate": round(sla_rate, 3),
            "accuracy": round(accuracy, 3),
            "throughput_per_hour": round(throughput["count"] / period_hours, 1),
            "per_reviewer_per_hour": round(per_reviewer / period_hours, 1),
            "active_reviewers": throughput["reviewers"],
            "backlog": backlog,
            "estimated_clear_hours": round(backlog / max(throughput["count"] / period_hours, 1), 1)
        }

    def check_capacity_alert(self):
        """检查容量告警"""
        metrics = self.calculate_efficiency_metrics()

        alerts = []

        # 积压预警
        if metrics["backlog"] > 1000:
            alerts.append({"type": "backlog", "severity": "high",
                "message": f"积压 {metrics['backlog']} 条，预计 {metrics['estimated_clear_hours']} 小时清完"})

        # SLA 预警
        if metrics["sla_rate"] < 0.9:
            alerts.append({"type": "sla", "severity": "high",
                "message": f"SLA 达成率 {metrics['sla_rate']:.1%}，低于 90%"})

        # 准确率预警
        if metrics["accuracy"] < 0.95:
            alerts.append({"type": "accuracy", "severity": "medium",
                "message": f"审核准确率 {metrics['accuracy']:.1%}，低于 95%"})

        # 人力不足
        if metrics["avg_wait_minutes"] > 30:
            alerts.append({"type": "capacity", "severity": "medium",
                "message": f"平均等待 {metrics['avg_wait_minutes']} 分钟，需增加审核员"})

        return alerts
```

## 异常场景补充

### 场景：审核员疲劳导致准确率下降

```
触发：审核员连续工作 8 小时 → 准确率从 98% 降到 92% → 误判增加
检测：
  1. 审核员工作超过 6 小时后准确率下降 → 疲劳
  2. 同一审核员后半段推翻率上升 → 疲劳
处理：
  1. 审核员轮班制度（每 4 小时休息）
  2. 高风险内容分配给精力充沛的审核员
  3. 疲劳审核员审核的内容增加抽检
预防：轮班制度 + 风险分配 + 疲劳抽检
```

### 场景：AI 审核模型遇到新型违规

```
触发：新型违规内容（如深度伪造广告）→ AI 模型未训练过 → 漏放 → 平台风险
检测：
  1. 用户举报内容中 AI 判定为"通过"的比例上升 → 模型缺陷
  2. 新型违规模式在举报中集中出现 → 需要模型更新
处理：
  1. 人工审核补充（举报内容优先人工审核）
  2. 收集新型违规样本 → 重新训练模型
  3. 临时规则补充
预防：举报优先人工 + 模型持续训练 + 临时规则
```

## 内容审核申诉处理完整实现

```python
class ModerationAppealService:
    """审核申诉：申诉提交 → 复审 → 判决 → 恢复/维持"""

    APPEAL_STATUSES = {
        "submitted": "已提交",
        "under_review": "复审中",
        "approved": "申诉成功",
        "rejected": "申诉驳回",
        "escalated": "已升级",
    }

    def submit_appeal(self, user_id, content_id, original_decision, reason):
        """提交申诉"""
        # 1. 检查是否可申诉
        content = self.db.get_content(content_id)
        if not content:
            return {"status": "content_not_found"}

        # 2. 检查申诉次数限制（同一内容最多申诉 2 次）
        existing_appeals = self.db.count("moderation_appeals",
            content_id=content_id, user_id=user_id)
        if existing_appeals >= 2:
            return {"status": "appeal_limit_reached"}

        # 3. 检查冷却期（上次申诉需间隔 24 小时）
        last_appeal = self.db.query_one(
            "SELECT * FROM moderation_appeals "
            "WHERE content_id = %s AND user_id = %s "
            "ORDER BY created_at DESC LIMIT 1",
            content_id, user_id)
        if last_appeal:
            hours_since = (now() - last_appeal["created_at"]).total_seconds() / 3600
            if hours_since < 24:
                return {"status": "cooldown", "retry_after_hours": round(24 - hours_since, 1)}

        # 4. 创建申诉
        appeal_id = str(uuid4())
        self.db.insert("moderation_appeals", {
            "appeal_id": appeal_id,
            "user_id": user_id,
            "content_id": content_id,
            "original_decision": original_decision,
            "reason": reason,
            "status": "submitted",
            "priority": self._calculate_appeal_priority(content, original_decision),
            "created_at": now()
        })

        # 5. 分配复审员
        reviewer = self._assign_reviewer(original_decision)
        self.db.update("moderation_appeals",
            {"reviewer_id": reviewer["id"], "status": "under_review"},
            {"appeal_id": appeal_id})

        # 6. 通知用户
        self.notification.send(user_id, "申诉已提交，正在复审中")

        return {"appeal_id": appeal_id, "status": "submitted",
                "assigned_reviewer": reviewer["id"]}

    def review_appeal(self, appeal_id, reviewer_id, decision, note=None):
        """复审申诉"""
        appeal = self.db.get_appeal(appeal_id)

        if appeal["reviewer_id"] != reviewer_id:
            raise PermissionDeniedError("非指定复审员")

        if appeal["status"] != "under_review":
            return {"status": "cannot_review", "current_status": appeal["status"]}

        # 1. 更新申诉状态
        self.db.update("moderation_appeals",
            {"status": decision, "review_note": note,
             "reviewed_at": now()},
            {"appeal_id": appeal_id})

        # 2. 如果申诉成功 → 恢复内容
        if decision == "approved":
            self._restore_content(appeal["content_id"])

            # 记录原审核判定为误判
            self.db.update("moderation_decisions",
                {"overturned": True, "overturned_by": reviewer_id,
                 "overturned_at": now()},
                {"content_id": appeal["content_id"],
                 "decision": appeal["original_decision"]})

            # 更新审核员准确率
            original_reviewer = self.db.query_one(
                "SELECT reviewer_id FROM moderation_decisions "
                "WHERE content_id = %s AND decision = %s",
                appeal["content_id"], appeal["original_decision"])
            if original_reviewer:
                self._update_reviewer_accuracy(original_reviewer["reviewer_id"], overturned=True)

        elif decision == "rejected":
            # 维持原判
            self.notification.send(appeal["user_id"],
                "申诉已被驳回" + (f": {note}" if note else ""))

        elif decision == "escalated":
            # 升级处理
            senior_reviewer = self._assign_senior_reviewer()
            self.db.update("moderation_appeals",
                {"reviewer_id": senior_reviewer["id"]},
                {"appeal_id": appeal_id})

        # 3. 更新统计
        self._update_appeal_statistics(appeal_id, decision)

        return {"appeal_id": appeal_id, "decision": decision}

    def _restore_content(self, content_id):
        """恢复被下架的内容"""
        self.db.update("contents",
            {"status": "published", "restored_at": now()},
            {"id": content_id})

        content = self.db.get_content(content_id)
        self.notification.send(content["author_id"],
            "您的内容已恢复，申诉成功")

    def _calculate_appeal_priority(self, content, original_decision):
        """计算申诉优先级"""
        priority = "normal"

        # 高影响内容优先
        if content.get("view_count", 0) > 10000:
            priority = "high"

        # 严重处罚优先（封号 vs 删内容）
        if original_decision == "account_banned":
            priority = "urgent"

        # 创作者优先
        author = self.db.get_user(content["author_id"])
        if author and author.get("is_creator"):
            priority = "high"

        return priority

    def _update_reviewer_accuracy(self, reviewer_id, overturned):
        """更新审核员准确率"""
        window = now() - timedelta(days=30)
        total = self.db.count("moderation_decisions",
            reviewer_id=reviewer_id, decided_at__gte=window)
        overturned_count = self.db.count("moderation_decisions",
            reviewer_id=reviewer_id, overturned=True,
            decided_at__gte=window)

        accuracy = 1 - (overturned_count / max(total, 1))

        self.db.upsert("reviewer_accuracy", {
            "reviewer_id": reviewer_id,
            "total_decisions": total,
            "overturned_count": overturned_count,
            "accuracy": round(accuracy, 4),
            "updated_at": now()
        }, conflict_columns=["reviewer_id"])
```

## 异常场景补充

### 场景：申诉审核员与原审核员相同

```
触发：申诉分配给原审核员 → 自己推翻自己的判定 → 公正性存疑
检测：
  1. 复审员 ID 与原审核员 ID 相同 → 利益冲突
  2. 申诉成功率异常高 → 可能未认真复审
处理：
  1. 申诉分配排除原审核员
  2. 复审必须由不同审核员执行
  3. 严重申诉需两名审核员共同决定
预防：排除原审核员 + 双人复审 + 分配逻辑校验
```

### 场景：申诉积压导致用户等待过长

```
触发：申诉积压 5000 条 → 平均处理时间 7 天 → 用户不满 → 流失
检测：
  1. 申诉平均处理时间 > 48 小时 → 积压
  2. 申诉队列长度 > 1000 → 人力不足
处理：
  1. 自动化简单申诉（低风险内容自动恢复）
  2. 增加复审员
  3. 优先处理高优先级申诉
预防：自动处理 + 人力弹性 + 优先级队列
```

## 内容审核工作量预测与人力调度完整实现

```python
import math
import uuid
import logging
from enum import Enum
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Tuple
from datetime import datetime, timedelta
from collections import defaultdict

logger = logging.getLogger(__name__)


class ShiftType(Enum):
    MORNING = "morning"        # 06:00 - 14:00
    AFTERNOON = "afternoon"    # 14:00 - 22:00
    NIGHT = "night"            # 22:00 - 06:00
    ON_CALL = "on_call"        # 随时待命


class ReviewerStatus(Enum):
    AVAILABLE = "available"
    WORKING = "working"
    BREAK = "break"
    ON_CALL = "on_call"
    OFF_DUTY = "off_duty"


class ContentType(Enum):
    TEXT = "text"
    IMAGE = "image"
    VIDEO = "video"


@dataclass
class HourlyWorkload:
    """单小时的工作量预测结果。"""
    hour: int            # 0~23
    date: str            # YYYY-MM-DD
    predicted_volume: int
    confidence: float    # 0.0 ~ 1.0
    breakdown: Dict[str, int] = field(default_factory=dict)  # content_type -> volume


@dataclass
class ReviewerProfile:
    reviewer_id: str
    name: str
    status: ReviewerStatus = ReviewerStatus.OFF_DUTY
    shift_type: Optional[ShiftType] = None
    skills: List[str] = field(default_factory=list)  # e.g. ["text", "image", "video"]
    throughput_per_hour: Dict[str, int] = field(default_factory=dict)  # content_type -> items/hour
    max_working_hours: int = 8
    current_work_hours: float = 0.0
    phone: str = ""
    email: str = ""


@dataclass
class ShiftSchedule:
    schedule_id: str
    date: str             # YYYY-MM-DD
    shift_type: ShiftType
    reviewer_ids: List[str]
    required_count: int
    actual_count: int
    coverage_rate: float  # actual / required


@dataclass
class SpikeEvent:
    spike_id: str
    detected_at: datetime
    predicted_volume: int
    normal_volume: int
    spike_ratio: float    # predicted / normal
    activation_threshold: float  # ratio that triggers on-call
    activated: bool = False


class ModerationWorkloadPredictor:
    """内容审核工作量预测与人力调度服务，基于历史模式预测每小时审核量，
    计算所需审核员数量，编排排班计划，并处理突发流量通过待命机制。"""

    # 每种内容类型的平均审核耗时（秒）
    AVG_REVIEW_SECONDS = {
        ContentType.TEXT: 30,
        ContentType.IMAGE: 60,
        ContentType.VIDEO: 180,
    }

    # 审核员每班有效工作时间占比（扣除休息、切换成本）
    EFFECTIVE_WORK_RATIO = 0.75

    # 突发流量激活阈值
    SPIKE_ACTIVATION_RATIO = 2.0  # 实际量超过预测 2 倍触发待命

    # 时段权重（工作日 vs 周末 vs 节假日）
    DAY_TYPE_MULTIPLIERS = {
        "weekday": 1.0,
        "weekend": 0.65,
        "holiday": 0.4,
        "special_event": 2.5,
    }

    def __init__(self):
        # 历史小时级审核量：date_str -> hour -> volume
        self.historical_data: Dict[str, Dict[int, int]] = {}
        # 内容类型占比历史
        self.content_type_ratios: Dict[str, float] = {
            ContentType.TEXT.value: 0.50,
            ContentType.IMAGE.value: 0.30,
            ContentType.VIDEO.value: 0.20,
        }
        # 小时模式（24小时归一化分布）
        self.hourly_pattern: Dict[int, float] = self._init_default_hourly_pattern()
        # 审核员注册表
        self.reviewers: Dict[str, ReviewerProfile] = {}
        # 排班记录：date_str -> shift_type -> ShiftSchedule
        self.shift_schedules: Dict[str, Dict[ShiftType, ShiftSchedule]] = {}
        # 待命事件记录
        self.spike_events: Dict[str, SpikeEvent] = {}
        # 特殊事件日历：date_str -> "special_event"
        self.special_event_calendar: Dict[str, str] = {}
        # 时区偏移
        self.timezone_offset: int = 8  # UTC+8

    def _init_default_hourly_pattern(self) -> Dict[int, float]:
        """初始化默认的小时级流量分布模式（归一化到 0~1）。"""
        # 典型社交平台流量模式：早高峰、午间、晚高峰
        raw = {
            0: 0.02, 1: 0.01, 2: 0.01, 3: 0.01, 4: 0.01, 5: 0.02,
            6: 0.04, 7: 0.06, 8: 0.08, 9: 0.07, 10: 0.06, 11: 0.06,
            12: 0.07, 13: 0.06, 14: 0.05, 15: 0.05, 16: 0.05,
            17: 0.06, 18: 0.07, 19: 0.08, 20: 0.08, 21: 0.07,
            22: 0.05, 23: 0.03,
        }
        total = sum(raw.values())
        return {h: v / total for h, v in raw.items()}

    def load_historical_data(self, date_volume_map: Dict[str, Dict[int, int]]):
        """加载历史审核量数据，用于优化预测模型。"""
        self.historical_data.update(date_volume_map)
        self._recalculate_hourly_pattern()
        self._recalculate_content_ratios()
        logger.info(f"Loaded {len(date_volume_map)} days of historical data")

    def _recalculate_hourly_pattern(self):
        """根据历史数据重新计算小时级流量分布。"""
        if not self.historical_data:
            return

        hour_totals = defaultdict(float)
        for date_str, hourly in self.historical_data.items():
            for hour, volume in hourly.items():
                hour_totals[hour] += volume

        grand_total = sum(hour_totals.values())
        if grand_total > 0:
            self.hourly_pattern = {
                h: v / grand_total for h, v in hour_totals.items()
            }

    def _recalculate_content_ratios(self):
        """根据历史数据重新计算内容类型占比。"""
        # 在实际场景中从审核记录聚合
        # 这里保持默认值，可由外部调用更新
        pass

    def predict_hourly_workload(
        self, target_date: str, daily_total_estimate: Optional[int] = None
    ) -> List[HourlyWorkload]:
        """预测目标日期每小时的审核工作量。

        Args:
            target_date: 目标日期，格式 YYYY-MM-DD
            daily_total_estimate: 当日总审核量估算，若未提供则基于历史均值

        Returns:
            24 个 HourlyWorkload 对象，分别对应 0~23 时
        """
        # 判断日期类型
        day_type = self._classify_day_type(target_date)

        # 计算日总量
        if daily_total_estimate is None:
            daily_total_estimate = self._estimate_daily_total(target_date, day_type)

        # 应用日期类型系数
        adjusted_total = int(daily_total_estimate * self.DAY_TYPE_MULTIPLIERS[day_type])

        predictions = []
        for hour in range(24):
            ratio = self.hourly_pattern.get(hour, 0.01)
            predicted = max(1, int(adjusted_total * ratio))

            # 计算置信度（基于该时段历史数据方差）
            confidence = self._calculate_confidence(hour, target_date)

            # 按内容类型拆分
            breakdown = {}
            for ct, ct_ratio in self.content_type_ratios.items():
                breakdown[ct] = max(0, int(predicted * ct_ratio))

            predictions.append(HourlyWorkload(
                hour=hour,
                date=target_date,
                predicted_volume=predicted,
                confidence=confidence,
                breakdown=breakdown,
            ))

        logger.info(
            f"Predicted workload for {target_date} ({day_type}): "
            f"total={adjusted_total}, peak_hour={max(predictions, key=lambda p: p.predicted_volume).hour}"
        )
        return predictions

    def _classify_day_type(self, date_str: str) -> str:
        """判断日期类型（工作日/周末/节假日/特殊事件）。"""
        if date_str in self.special_event_calendar:
            return "special_event"

        dt = datetime.strptime(date_str, "%Y-%m-%d")
        # 简化：周六周日为周末
        if dt.weekday() >= 5:
            return "weekend"

        # 实际场景应接入节假日 API
        return "weekday"

    def _estimate_daily_total(self, target_date: str, day_type: str) -> int:
        """基于历史均值估算日总量。"""
        if not self.historical_data:
            # 默认值：5万/天
            return 50000

        # 取同类型日期的历史均值
        type_totals = []
        for date_str, hourly in self.historical_data.items():
            hist_type = self._classify_day_type(date_str)
            if hist_type == day_type:
                type_totals.append(sum(hourly.values()))

        if type_totals:
            return int(sum(type_totals) / len(type_totals))
        return sum(sum(h.values()) for h in self.historical_data.values()) // len(self.historical_data)

    def _calculate_confidence(self, hour: int, target_date: str) -> float:
        """计算预测置信度（基于历史方差）。"""
        if len(self.historical_data) < 3:
            return 0.5

        values = []
        for date_str, hourly in self.historical_data.items():
            if hour in hourly:
                values.append(hourly[hour])

        if len(values) < 2:
            return 0.5

        mean = sum(values) / len(values)
        variance = sum((v - mean) ** 2 for v in values) / len(values)
        cv = math.sqrt(variance) / max(1, mean)  # 变异系数
        # 变异系数越小置信度越高
        confidence = max(0.3, min(0.95, 1.0 - cv))
        return round(confidence, 2)

    def calculate_required_reviewers(
        self, hourly_predictions: List[HourlyWorkload]
    ) -> Dict[int, Dict]:
        """根据小时级预测量计算每时段所需的审核员数量。

        Returns:
            hour -> {required_count, by_content_type, estimated_queue_time}
        """
        requirements = {}

        for pred in hourly_predictions:
            # 按内容类型计算所需人时
            total_person_seconds = 0.0
            by_type = {}
            for ct, volume in pred.breakdown.items():
                ct_enum = ContentType(ct)
                review_time = self.AVG_REVIEW_SECONDS.get(ct_enum, 60)
                person_seconds = volume * review_time
                total_person_seconds += person_seconds
                # 该类型所需人数 = person_seconds / (3600 * effective_ratio)
                reviewers_needed = math.ceil(
                    person_seconds / (3600 * self.EFFECTIVE_WORK_RATIO)
                )
                by_type[ct] = reviewers_needed

            # 总需求人数（取各类型最大值的和，因为审核员可以交叉处理）
            required_count = math.ceil(
                total_person_seconds / (3600 * self.EFFECTIVE_WORK_RATIO)
            )

            # 估算排队时间（如果人手不足）
            capacity_per_reviewer = 3600 * self.EFFECTIVE_WORK_RATIO
            queue_time = max(
                0, (total_person_seconds - required_count * capacity_per_reviewer) / max(1, required_count)
            )

            requirements[pred.hour] = {
                "required_count": required_count,
                "by_content_type": by_type,
                "estimated_queue_seconds": round(queue_time, 1),
                "predicted_volume": pred.predicted_volume,
                "confidence": pred.confidence,
            }

        return requirements

    def schedule_shifts(
        self, target_date: str, requirements: Dict[int, Dict]
    ) -> List[ShiftSchedule]:
        """基于需求编排审核员排班。

        将 24 小时需求映射到三个班次，分配审核员以满足高峰需求。
        """
        # 按班次聚合需求
        shift_hours = {
            ShiftType.MORNING: list(range(6, 14)),
            ShiftType.AFTERNOON: list(range(14, 22)),
            ShiftType.NIGHT: list(range(22, 24)) + list(range(0, 6)),
        }

        schedules = []

        for shift_type, hours in shift_hours.items():
            # 取班次内最大需求作为该班次人数
            peak_requirement = 0
            for h in hours:
                req = requirements.get(h, {})
                peak_requirement = max(peak_requirement, req.get("required_count", 0))

            # 最低保障人数
            min_staffing = {
                ShiftType.MORNING: 5,
                ShiftType.AFTERNOON: 5,
                ShiftType.NIGHT: 3,
            }
            required_count = max(peak_requirement, min_staffing[shift_type])

            # 从可用审核员中分配
            assigned_ids = self._assign_reviewers_to_shift(
                target_date, shift_type, required_count
            )

            actual_count = len(assigned_ids)
            coverage_rate = actual_count / max(1, required_count)

            schedule = ShiftSchedule(
                schedule_id=f"sch_{target_date}_{shift_type.value}_{uuid.uuid4().hex[:6]}",
                date=target_date,
                shift_type=shift_type,
                reviewer_ids=assigned_ids,
                required_count=required_count,
                actual_count=actual_count,
                coverage_rate=round(coverage_rate, 2),
            )

            # 存储排班
            if target_date not in self.shift_schedules:
                self.shift_schedules[target_date] = {}
            self.shift_schedules[target_date][shift_type] = schedule

            schedules.append(schedule)

            if coverage_rate < 1.0:
                logger.warning(
                    f"Shift {shift_type.value} on {target_date} under-staffed: "
                    f"{actual_count}/{required_count} ({coverage_rate:.0%})"
                )

        return schedules

    def _assign_reviewers_to_shift(
        self, target_date: str, shift_type: ShiftType, required_count: int
    ) -> List[str]:
        """为指定班次分配审核员，优先分配技能匹配且工时未满的人员。"""
        candidates = []
        for rid, reviewer in self.reviewers.items():
            if reviewer.status == ReviewerStatus.OFF_DUTY:
                continue
            if reviewer.current_work_hours >= reviewer.max_working_hours:
                continue
            # 检查是否已排班
            already_scheduled = False
            day_schedules = self.shift_schedules.get(target_date, {})
            for existing_shift, sched in day_schedules.items():
                if rid in sched.reviewer_ids:
                    already_scheduled = True
                    break
            if already_scheduled:
                continue

            # 技能匹配度评分
            skill_score = self._calculate_skill_score(reviewer, shift_type)
            candidates.append((rid, skill_score))

        # 按评分降序排列
        candidates.sort(key=lambda x: x[1], reverse=True)

        # 取前 N 个
        assigned = [rid for rid, _ in candidates[:required_count]]

        # 更新审核员状态
        for rid in assigned:
            self.reviewers[rid].shift_type = shift_type
            self.reviewers[rid].current_work_hours += 8.0

        return assigned

    def _calculate_skill_score(self, reviewer: ReviewerProfile, shift_type: ShiftType) -> float:
        """计算审核员与班次的匹配度评分。"""
        score = 0.0
        # 多技能加分
        if "video" in reviewer.skills:
            score += 3.0  # 视频审核最稀缺
        if "image" in reviewer.skills:
            score += 2.0
        if "text" in reviewer.skills:
            score += 1.0
        # 夜班意愿
        if shift_type == ShiftType.NIGHT and "night_shift" in reviewer.skills:
            score += 2.0
        # 工时余量（工时少的优先，避免过劳）
        remaining_hours = reviewer.max_working_hours - reviewer.current_work_hours
        score += remaining_hours * 0.5
        return score

    def handle_spike(
        self, current_hour: int, actual_volume: int, predicted_volume: int
    ) -> SpikeEvent:
        """处理审核工作量突发激增，触发待命审核员激活。

        Args:
            current_hour: 当前小时
            actual_volume: 当前实际审核量
            predicted_volume: 预测的审核量

        Returns:
            SpikeEvent 记录
        """
        spike_ratio = actual_volume / max(1, predicted_volume)
        spike_id = f"spike_{datetime.now().strftime('%Y%m%d%H')}_{uuid.uuid4().hex[:6]}"

        event = SpikeEvent(
            spike_id=spike_id,
            detected_at=datetime.now(),
            predicted_volume=predicted_volume,
            normal_volume=predicted_volume,
            spike_ratio=round(spike_ratio, 2),
            activation_threshold=self.SPIKE_ACTIVATION_RATIO,
            activated=False,
        )

        # 超过阈值则激活待命人员
        if spike_ratio >= self.SPIKE_ACTIVATION_RATIO:
            event.activated = True
            activated_count = self._activate_on_call_reviewers()

            logger.warning(
                f"Spike detected! Ratio={spike_ratio:.1f}x, "
                f"activated {activated_count} on-call reviewers"
            )

            # 额外措施：降低自动化审核阈值，让机器承担更多
            self._adjust_auto_moderation_threshold(spike_ratio)

        self.spike_events[spike_id] = event
        return event

    def _activate_on_call_reviewers(self) -> int:
        """激活所有待命审核员。"""
        activated = 0
        for rid, reviewer in self.reviewers.items():
            if reviewer.status == ReviewerStatus.ON_CALL:
                reviewer.status = ReviewerStatus.AVAILABLE
                reviewer.shift_type = ShiftType.ON_CALL
                activated += 1
                # 发送通知（实际场景通过短信/电话/推送）
                logger.info(
                    f"On-call reviewer {reviewer.name} ({rid}) activated"
                )
        return activated

    def _adjust_auto_moderation_threshold(self, spike_ratio: float):
        """调整自动化审核阈值以应对突发流量。

        当人工审核能力不足时，适当放宽机器审核的置信度阈值，
        让更多内容通过机器审核直接放行，减轻人工压力。
        """
        # 将机器审核置信度阈值从 0.95 降低到 0.85
        # 仅在极端突发时启用
        if spike_ratio >= 3.0:
            logger.warning(
                f"Extreme spike ({spike_ratio:.1f}x): lowering auto-moderation "
                f"confidence threshold to 0.85"
            )
        elif spike_ratio >= 2.0:
            logger.info(
                f"Moderate spike ({spike_ratio:.1f}x): lowering auto-moderation "
                f"confidence threshold to 0.90"
            )

    def register_reviewer(
        self, name: str, skills: List[str], phone: str, email: str,
        max_working_hours: int = 8, throughput: Optional[Dict[str, int]] = None,
    ) -> ReviewerProfile:
        """注册审核员。"""
        rid = f"rev_{uuid.uuid4().hex[:8]}"
        reviewer = ReviewerProfile(
            reviewer_id=rid,
            name=name,
            skills=skills,
            phone=phone,
            email=email,
            max_working_hours=max_working_hours,
            throughput_per_hour=throughput or {
                ContentType.TEXT.value: 120,
                ContentType.IMAGE.value: 60,
                ContentType.VIDEO.value: 20,
            },
            status=ReviewerStatus.AVAILABLE,
        )
        self.reviewers[rid] = reviewer
        logger.info(f"Registered reviewer {name} ({rid}) with skills: {skills}")
        return reviewer

    def get_staffing_summary(self, target_date: str) -> Dict:
        """获取指定日期的人力排班摘要。"""
        day_schedules = self.shift_schedules.get(target_date, {})
        summary = {
            "date": target_date,
            "shifts": {},
            "total_assigned": 0,
            "total_required": 0,
            "overall_coverage": 0.0,
        }

        total_assigned = 0
        total_required = 0
        for shift_type, schedule in day_schedules.items():
            summary["shifts"][shift_type.value] = {
                "required": schedule.required_count,
                "assigned": schedule.actual_count,
                "coverage": schedule.coverage_rate,
                "reviewer_ids": schedule.reviewer_ids,
            }
            total_assigned += schedule.actual_count
            total_required += schedule.required_count

        summary["total_assigned"] = total_assigned
        summary["total_required"] = total_required
        summary["overall_coverage"] = (
            round(total_assigned / max(1, total_required), 2)
            if total_required > 0 else 0.0
        )
        return summary
```

## 异常场景补充

### 场景：预测偏差导致人力不足
```
触发条件：预测模型低估了审核量（如突发社会事件引发大量用户生成内容），实际审核量达到预测值的 3 倍以上，排班人力严重不足

检测机制：
  - 实时监控审核队列深度，队列积压超过 5000 条触发告警
  - 单条内容平均等待审核时间超过 10 分钟（SLA 为 5 分钟）
  - 审核员当前小时处理量达到饱和（有效工作占比 > 95%）
  - 自动审核驳回率上升（因置信度不足被送到人工队列的比例异常增加）
  - 对比实际量与预测量的偏差超过 50% 持续 1 小时

处理策略：
  - 立即激活所有待命审核员，通过短信/电话紧急召回
  - 调低机器审核置信度阈值，让高置信度内容自动放行（仅标记低置信度送人工）
  - 启动跨区域审核员支援：其他时区团队代为审核
  - 对非紧急内容（如举报复核）暂时延后处理，优先保障新增内容审核
  - 开启"快速审核模式"：简化审核流程，仅标注明确违规，模糊内容先放行后复核
  - 预测模型自动标记该时段异常，纳入后续训练数据

预防措施：
  - 建立社会热点事件监控（热搜/新闻 API），提前预判流量波动
  - 预测模型增加"事件驱动"特征维度，减少对纯时序模式的依赖
  - 每班保留 15% 的人力冗余作为缓冲
  - 设置阶梯式应急响应：1.5x 黄色预警、2x 橙色预警、3x 红色预警
  - 定期进行突发流量压力测试，验证待命激活流程
  - 与业务侧建立信息同步机制：营销活动、产品更新提前通知审核团队
```

### 场景：审核员临时请假
```
触发条件：审核员因个人原因（生病、家庭事务等）在班次开始前 2 小时内请假，导致该班次人力出现缺口

检测机制：
  - 请假系统实时同步到排班系统
  - 排班覆盖率低于 80% 自动告警
  - 审核员未在班次开始后 15 分钟内上线签到
  - 排班缺口导致队列积压超过阈值

处理策略：
  - 自动从同日其他班次的冗余人员中调配（如早班多出的人延后到午班）
  - 激活待命审核员填补缺口
  - 跨技能调配：将纯文本审核员临时安排处理图片内容（降低效率但保证覆盖）
  - 如果缺口 > 30%，启动紧急召回：联系近日休息的审核员加班
  - 临时提高机器审核比例：对低风险内容类型放宽机器审核阈值
  - 调整 SLA 预期：通知业务方审核延迟可能增加

预防措施：
  - 每班次设置 10~15% 的人力冗余
  - 建立审核员技能矩阵，确保任一岗位至少有 2 人具备相应技能（bus factor >= 2）
  - 排班系统支持"候补"机制：审核员可自愿标记可替补时段
  - 审核员请假需提前 24 小时（紧急情况除外），给调度系统反应时间
  - 维护外部审核外包商名录，紧急时可调用第三方审核力量
  - 定期轮岗培训，确保审核员技能多样化，提高调配灵活性
```
