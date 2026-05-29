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
        """审核员获取待审内容（跳过被锁定的行）"""
        return self.db.query("""
            SELECT * FROM review_queue
            WHERE status = 'PENDING'
            ORDER BY priority DESC, enqueued_at ASC
            LIMIT %s FOR UPDATE SKIP LOCKED
        """, count)
```

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

### 降级策略

```python
class ReviewQueueMonitor:
    def check_backlog(self):
        pending = self.db.query_one(
            "SELECT COUNT(*) FROM review_queue WHERE status = 'PENDING'"
        )

        if pending > 5000:
            # 一级降级：AI 通过阈值降低
            # 文本：0.90 → 0.80，图片：0.90 → 0.80
            self.adjust_thresholds(mode="high_load")
            self.notify_ops("审核积压 > 5000，已降低AI通过阈值")

        elif pending > 20000:
            # 二级降级：视频启用先发后审
            self.enable_publish_then_review(all_users=True)
            self.notify_ops("审核积压 > 20000，已启用先发后审")

        elif pending > 50000:
            # 三级降级：紧急扩容
            self.request_more_reviewers(count=20)
            self.notify_ops("审核积压 > 50000，已请求紧急扩容")
```

**降级的代价：**

| 降级级别 | AI阈值 | 误杀率变化 | 漏检率变化 |
|---------|-------|-----------|-----------|
| 正常 | 0.90 | 1% | 0.1% |
| 一级（阈值0.80） | 0.80 | 0.5%（降低） | 0.3%（升高） |
| 二级（先发后审） | 0.80 | 0.5% | 0.5%（30秒窗口） |

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