# C14: 内容审核的多级流水线

## 业务场景

某社交平台，日新增 500 万条 UGC 内容（文本、图片、视频），需要审核后才能公开展示。审核目标：违规内容（色情、暴力、政治敏感、广告、辱骂）的漏检率 < 0.1%，误杀率 < 1%。

**为什么不是简单的 AI 判定通过/拒绝？**

- AI 置信度 < 90% 的内容需要人工复审——约 20% 的内容需要人工介入
- 人工审核产能有限（50 人 × 200 条/小时 = 1 万条/天），但日均违规约 15 万条
- 视频审核需要抽帧分析，5 分钟视频 = 30 帧逐一分析
- 误杀的代价（用户流失）vs 漏检的代价（监管处罚）的权衡

**已知数据：**
- 日新增内容：500 万条
- 峰值 QPS：2000 条/秒
- 内容类型：文本 60%、图片 30%、视频 10%
- AI 审核延迟：文本 < 100ms，图片 < 500ms，视频 < 30s
- 人工审核延迟：5-30 分钟
- 违规率：约 3%

## 核心挑战

### 挑战 1：AI+人工混合审核的流水线

AI 毫秒级，人工分钟级。如何编排使高置信度内容秒级通过，低置信度内容进入人工队列不积压？

### 挑战 2：视频审核的延迟

5 分钟视频抽帧+AI 分析需要 30 秒。30 秒内不可见 → 用户体验差。但不审核就展示 → 违规视频已传播。

### 挑战 3：误杀与漏检的权衡

降低漏检率 → 提高敏感度 → 误杀率上升 → 用户抱怨
降低误杀率 → 降低敏感度 → 漏检率上升 → 监管处罚

## 设计约束

- 所有内容必须审核后公开展示
- 人工审核积压 < 2 小时
- 审核结果可追溯
- 规则更新即时生效

## 请先独立思考（限时 30 分钟）

1. 设计多级审核流水线：AI 高置信度秒级通过，低置信度进人工队列。
2. 视频审核如何平衡延迟和安全？是否可以先展示再审核？
3. 人工审核积压时如何降级？

---

## 设计解析

### 多级审核流水线

```
用户发布内容 → 第一级：规则过滤（黑名单/正则）→ 命中 → 直接拒绝
                                    → 未命中 ↓
               → 第二级：AI审核 → 高置信违规 → 拒绝
                                 → 高置信正常 → 通过
                                 → 低置信 ↓
               → 第三级：人工审核 → 违规/正常
```

```python
class ContentModerationPipeline:
    def moderate(self, content):
        # 第一级：规则过滤（< 1ms）
        rule_result = self.rule_filter.check(content)
        if rule_result.hit:
            return ModerationResult(decision="REJECT", reason=rule_result.reason, level=1)

        # 第二级：AI 审核
        ai_result = self.ai_moderator.check(content)
        if ai_result.confidence > 0.95 and ai_result.is_violation:
            return ModerationResult(decision="REJECT", reason=ai_result.reason, level=2)
        elif ai_result.confidence > 0.90 and not ai_result.is_violation:
            return ModerationResult(decision="APPROVE", level=2)
        else:
            # 第三级：人工审核
            self.enqueue_human_review(content, ai_result)
            return ModerationResult(decision="PENDING", level=3)
```

### 第一级：规则过滤

```python
class RuleFilter:
    def __init__(self):
        self.blacklist_words = self.load_blacklist()      # 敏感词库
        self.regex_rules = self.load_regex_rules()        # 正则规则（手机号、URL等）
    
    def check(self, content):
        # 黑名单词匹配（AC 自动机，O(N)扫描）
        for word in self.ac_automaton.search(content.text):
            return RuleResult(hit=True, reason=f"blacklist:{word}")
        
        # 正则匹配（如手机号、微信号）
        for rule in self.regex_rules:
            if re.search(rule.pattern, content.text):
                return RuleResult(hit=True, reason=f"regex:{rule.name}")
        
        return RuleResult(hit=False)
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
        
        max_label = max(scores, key=scores.get)
        max_score = scores[max_label]
        
        return AIModerationResult(
            is_violation=max_score > 0.5,
            reason=max_label,
            confidence=max_score,
            all_scores=scores
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
        
        max_label = max(results, key=results.get)
        return AIModerationResult(
            is_violation=results[max_label] > 0.5,
            reason=max_label,
            confidence=results[max_label],
            all_scores=results
        )

    def video_moderation(self, content):
        """视频抽帧 + 图片审核"""
        frames = self.extract_keyframes(content.video_url, interval=10)  # 每10秒一帧
        
        max_score = 0
        violation_label = None
        for frame in frames:
            result = self.image_moderation(Image(content=frame))
            if result.confidence > max_score and result.is_violation:
                max_score = result.confidence
                violation_label = result.reason
        
        # 音频审核（语音转文字）
        audio_text = self.speech_to_text(content.video_url)
        if audio_text:
            audio_result = self.text_moderation(Text(content=audio_text))
            if audio_result.confidence > max_score:
                max_score = audio_result.confidence
                violation_label = audio_result.reason
        
        return AIModerationResult(is_violation=max_score > 0.5, reason=violation_label, confidence=max_score)
```

### 人工审核队列

```python
class HumanReviewQueue:
    def enqueue(self, content, ai_result):
        priority = self.calculate_priority(ai_result)
        self.db.execute("""
            INSERT INTO review_queue (content_id, ai_confidence, ai_reason, priority, enqueued_at)
            VALUES (%s, %s, %s, %s, NOW())
        """, content.id, ai_result.confidence, ai_result.reason, priority)

    def calculate_priority(self, ai_result):
        """优先级：政治敏感 > 色情 > 暴力 > 广告 > 辱骂"""
        priority_map = {"politics": 100, "porn": 80, "violence": 60, "ad": 40, "abuse": 20}
        return priority_map.get(ai_result.reason, 10) + ai_result.confidence * 50

    def get_next_batch(self, reviewer_id, count=20):
        """审核员获取待审内容（跳过被锁定的行）"""
        return self.db.query("""
            SELECT * FROM review_queue
            WHERE status = 'PENDING'
            ORDER BY priority DESC, enqueued_at ASC
            LIMIT %s FOR UPDATE SKIP LOCKED
        """, count)
```

### 视频先发后审（可信用户）

```python
class PublishingStrategy:
    def decide(self, content, user):
        if content.type == "video" and user.trust_score > 0.8 and user.violation_count == 0:
            # 可信用户 → 先发后审
            return "publish_then_review"
        else:
            return "review_then_publish"
```

先发后审风险控制：AI 30 秒内完成审核 → 发现违规立即下架。

### 降级策略

```python
class ReviewQueueMonitor:
    def check_backlog(self):
        pending = self.db.query_one("SELECT COUNT(*) FROM review_queue WHERE status = 'PENDING'")
        
        if pending > 5000:
            self.adjust_thresholds(mode="high_load")     # AI 通过阈值 0.90 → 0.80
        elif pending > 20000:
            self.adjust_thresholds(mode="emergency")      # AI 通过阈值 0.80 → 0.70
```

## 常见陷阱（深度分析）

### 陷阱 1：纯 AI 无人工

新型违规（新诈骗话术、新色情变体）AI 无法识别 → 漏检。必须有人工兜底。

### 陷阱 2：先发后审无限制

恶意用户利用审核时间窗口传播违规内容。必须限制为可信用户。

### 陷阱 3：不区分误杀和漏检代价

对于政治敏感内容，漏检代价远高于误杀 → 阈值应更敏感。
对于广告内容，误杀代价较高（用户正常推广被删）→ 阈值应更宽松。

## 延伸思考

- **审核规则 A/B 测试**：对 10% 流量应用新规则，对比漏检率和误杀率。
- **用户举报闭环**：举报内容进入高优先级人工队列，形成"AI+人工+社区"三层防线。
- **跨模态审核**：图片中嵌文字（截图）、视频中含音频，需多模态联合审核。