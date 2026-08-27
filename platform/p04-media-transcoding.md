# P04: 媒体处理与转码流水线

## 业务场景

某统一内容平台的媒体处理域，每天要把 505 万条新增视频（短视频 500 万 + 长视频 5 万）转换成可自适应播放的多档位产物。转码是平台**成本第二高、技术门槛最高**的域——它的输出直接决定了带宽成本（占总成本 73%）与转码副本存储成本（存储占 20%），以及播放画质。

**已知数据：**
- 日新增视频：短视频 500 万条（均长 38s）、长视频 5 万条（均长 42min）
- 日转码素材总量：3 亿秒（短视频 1.9 亿秒 + 长视频 1.26 亿秒）
- 原片入库：0.6 PB/天，均码率 8 Mbps（用户端上传前已压缩）
- 分辨率分布：1080P 62%、720P 24%、4K 9%、其他 5%
- 转码时效要求：短视频 P95 < 2 分钟、长视频 P95 < 14 分钟
- 转码算力预算：¥310 万/月（GPU 420 卡 + CPU 180 台）
- 输出档位：按内容热度分 3 层，2-6 档不等
- 画质底线：同档位 VMAF ≥ 88（1080P），劣化投诉率 < 0.05%

**为什么转码不是"跑一遍 FFmpeg"那么简单？**

单条视频跑 FFmpeg 是一行命令。505 万条/天、¥310 万/月预算、且输出质量直接乘以 ¥2.3 亿/月带宽成本时，问题完全变了：

| 朴素做法 | 问题 | 量化代价 |
|---------|------|---------|
| 固定 ABR 阶梯（所有视频同一套码率） | 静态画面浪费码率，高动态画面画质不足 | 平均浪费 22% 码率 = ¥5,069 万/月 |
| 所有视频转全 6 档 | 90% 长尾内容的高档位无人播放 | 浪费转码 ¥680 万/月 + 存储 298 PB/年 |
| 全部用最优编码器预设 | 编码耗时暴涨，队列积压 | 需 4,600 台机器（¥3,400 万/月） |
| 不做质量校验 | 花屏/黑屏/静音产物直接上线 | 万分之几的事故率 × 5 亿次播放 = 每天数万次糟糕体验 |
| 转码完才通知下游 | 审核必须等全部档位完成 | 投稿端到端 P95 从 14min 涨到 20min |

**核心矛盾**：**编码质量、编码成本、编码时效三者互相挤压，而三者的最优权衡点强烈依赖内容的最终播放量——但播放量在转码时是未知的。**

这是转码域最本质的困难：**它必须在信息不完整时（不知道这条视频会火还是沉底）做出成本不可逆的决策**（编码算力一旦花掉就收不回）。

## 核心挑战

### 挑战 1：ABR 阶梯怎么定——固定阶梯的浪费有多大

自适应码率需要一组"分辨率 + 码率"档位。传统做法是一套固定阶梯（如 Apple 推荐值）套用所有内容。但不同内容的**码率-画质曲线形状完全不同**：

- 一段固定机位的说话人视频（新闻播报），1080P 用 1.8 Mbps 就能达到 VMAF 95
- 一段高速运动的滑雪视频，1080P 需要 6.5 Mbps 才能达到 VMAF 95

固定阶梯给两者都分配 4.5 Mbps：前者浪费 2.7 Mbps（150%），后者画质不足（VMAF 仅 89）。**两个方向都错。**

难点在于逐条内容算最优阶梯（per-title encoding）需要试编码——试编码本身消耗算力，可能超过节省的带宽。

### 挑战 2：编码复杂度与带宽收益的性价比依赖播放量，而播放量未知

AV1 比 H.264 省 50% 码率，但编码复杂度是 10-30 倍。这笔账取决于播放量：

| 播放量 | AV1 省的带宽成本 | AV1 多花的编码成本 | 净收益 |
|-------|----------------|-----------------|-------|
| 100 次 | ¥0.008 | ¥2.1 | **-¥2.09（亏）** |
| 1 万次 | ¥0.8 | ¥2.1 | **-¥1.3（亏）** |
| 100 万次 | ¥80 | ¥2.1 | +¥77.9（赚） |
| 1 亿次 | ¥8,000 | ¥2.1 | +¥7,998（大赚） |

盈亏平衡点约 2.6 万次播放。但**转码发生在发布前，播放量在发布后才知道**。这是一个典型的"决策时信息不完整"问题。

### 挑战 3：转码集群的调度——长任务、抢占、成本感知

转码任务的特性对调度极不友好：

- **时长跨度 4 个数量级**：一条 15 秒短视频转码 4 秒；一条 3 小时 4K 纪录片转码 90 分钟
- **不可中断**：中途被杀就要从头重来（除非做分片转码）
- **优先级动态**：头部创作者的内容、热点话题内容需要插队
- **资源异构**：GPU 适合 1080P 及以下，4K 需要 CPU 软编保画质
- **成本弹性**：峰值可溢出到云上按量付费，但单价是自建的 2.5 倍

朴素的 FIFO 队列会让一条 90 分钟的长视频阻塞后面 1000 条短视频；朴素的优先级队列会让长尾内容永远饿死。

### 挑战 4：转码产物的质量校验——错误产物比没有产物更糟

转码是一个有失败模式的过程：源片损坏、编码器 bug、硬件错误、参数不当，都可能产生"格式合法但内容错误"的产物——花屏、黑屏、音画不同步、音频缺失、时长截断。

这类错误**不会导致转码任务报错**（FFmpeg 正常退出，返回码 0），只能靠主动检测发现。而未检出的错误产物一旦上线，用户看到的是黑屏或花屏——**比"视频还在处理中"糟糕得多**，因为用户会认为内容本身有问题。

### 挑战 5：长尾内容的高档位——预转码浪费 vs JIT 首播卡顿

90% 的内容 7 日播放量 < 100 次。给它们预转 6 档纯属浪费。但如果只转 2 档，当某条长尾内容突然被播放且用户网络很好（能吃 1080P）时，只能给 720P——画质打折。

JIT（首次请求时实时转码）能解决浪费，但引入首播延迟：一条 42 分钟长视频转 1080P 需要 6 分钟，用户不可能等 6 分钟。

## 设计约束

- 短视频转码 P95 < 2 分钟，长视频 P95 < 14 分钟
- 代理片（供审核用的 360P 极速档）必须在 40 秒内产出
- 同档位 VMAF ≥ 88（1080P）、≥ 85（720P）
- 转码算力月成本 ≤ ¥310 万
- 转码产物错误率 < 0.001%（每 10 万条不超过 1 条错误产物上线）
- 单条视频转码失败重试 3 次后必须给创作者明确反馈，不可静默失败

## 请先独立思考（限时 45 分钟）

1. ABR 阶梯你会怎么定？固定阶梯 vs 逐条计算（per-title），后者的试编码成本如何摊薄到可接受？档位之间的间距按什么定？
2. 编码格式的选择需要知道播放量，但转码发生在发布前。你怎么处理这个"决策时信息不完整"？（提示：谁说转码只能做一次？）
3. 一条 90 分钟 4K 视频（转码需 90 分钟）和 1000 条 15 秒短视频（各需 4 秒）同时到达。你的调度器怎么排？如何既不让长视频阻塞短视频，又不让长视频饿死？
4. 转码产物"格式合法但内容错误"（花屏/黑屏/音画不同步）如何检测？检测成本能否低于转码成本的 5%？

---

## 设计解析

### 转码流水线全景

```
                        ┌──────────────────────┐
   ContentSubmitted ───▶│  转码调度器 (Go)      │
   事件                  │  · 优先级计算         │
                        │  · 资源匹配           │
                        │  · 分片决策           │
                        └──────┬───────────────┘
                               │
              ┌────────────────┼─────────────────┐
              ▼                ▼                 ▼
     ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐
     │ 阶段0 探针   │  │ 阶段1 代理片 │  │ 阶段2 主转码     │
     │ (2s)        │  │ (40s)       │  │ (分档并行)       │
     │ · 解封装探测 │  │ · 360P极速  │  │ · ABR阶梯生成    │
     │ · 元数据提取 │  │ · 供审核用  │  │ · 多编码器       │
     │ · 损坏检测   │  │ · 供封面用  │  │ · 窄带高清预处理 │
     │ · 复杂度分析 │  └──────┬──────┘  └────────┬────────┘
     └──────┬──────┘         │                  │
            │                ▼                  ▼
            │        MediaProxyReady      ┌─────────────┐
            │        (审核可以开始了)      │ 阶段3 质检   │
            │                             │ · VMAF打分   │
            ▼                             │ · 花屏检测   │
     ┌──────────────┐                     │ · 音画同步   │
     │ 复杂度→阶梯   │                     │ · 时长校验   │
     │ 决策（per-   │                     └──────┬──────┘
     │ title 阶梯）  │                            │ 通过
     └──────────────┘                            ▼
                                         ┌─────────────────┐
     并行支线（不阻塞主链路）:              │ 阶段4 封装打包   │
     ┌──────────────────────┐             │ · CMAF 分片     │
     │ · 封面抽帧 + 美学打分  │             │ · HLS/DASH清单  │
     │ · 雪碧图 / 预览动图    │             │ · DRM 加密      │
     │ · ASR 字幕 + 翻译      │             └────────┬────────┘
     │ · 视频/音频指纹        │                      │
     │ · 明水印 + 盲水印      │                      ▼
     └──────────────────────┘              MediaReady 事件
```

**关键设计：阶段 0 探针（2 秒）和阶段 1 代理片（40 秒）是整条流水线的两个杠杆点。** 探针用 2 秒的成本换到"内容复杂度"这个信息，让后续所有档位决策都有依据；代理片用 40 秒换到"审核可以开始"，把 14 分钟转码与 6 分钟审核从串行压成并行。

### ABR 阶梯设计：从固定阶梯到凸包优化

#### 为什么固定阶梯必然浪费

固定阶梯的假设是"所有内容的码率-画质曲线相同"，这个假设错得很厉害。实测三类内容在 1080P 达到 VMAF 93 所需的码率：

```
内容类型               达到 VMAF 93 所需码率    固定阶梯给的码率   偏差
──────────────────────┼──────────────────────┼───────────────┼────────
新闻播报（固定机位）      1.6 Mbps               4.5 Mbps        +181% 浪费
访谈对话                 2.1 Mbps               4.5 Mbps        +114% 浪费
Vlog（手持轻微运动）      3.4 Mbps               4.5 Mbps        +32% 浪费
游戏录屏（快速切换）      5.2 Mbps               4.5 Mbps        -13% 画质不足
体育赛事（高速运动）      6.8 Mbps               4.5 Mbps        -34% 画质不足
烟花/水花（高频细节）     9.1 Mbps               4.5 Mbps        -51% 严重不足
```

加权平均浪费 22%——按 ¥2.3 亿/月带宽计，**固定阶梯每月白烧 ¥5,069 万**。

#### 凸包优化（Convex Hull）的原理

对单条视频，在"分辨率 × 量化参数"网格上试编码，得到一批（码率, 画质）点。这些点中只有一部分是**帕累托最优**的——即不存在另一个点同时码率更低且画质更高。这些最优点构成凸包（上凸包）。

```
VMAF
 100 ┤                          ╭──●────●───  ← 1080P 曲线
     │                    ╭──●─╯
  95 ┤              ╭──●─╯   ╱
     │         ╭─●─╯       ╱ ← 凸包（帕累托前沿）
  90 ┤    ╭─●─╯      ╭──●─╯
     │  ╭●╯     ╭──●╯          ← 720P 曲线
  85 ┤ ●    ╭─●╯
     │   ╭─●╯   ○ ○ ○  ← 被凸包支配的点（同码率下有更好选择）
  80 ┤ ╭●╯     ○
     │●   ← 480P 曲线
  75 ┼──┬────┬────┬────┬────┬────┬────▶ 码率 (Mbps)
     0  1    2    3    4    5    6

关键洞察：在某些码率区间，低分辨率的高质量编码 优于 高分辨率的低质量编码。
例如 1.5 Mbps 时，720P@高质量 (VMAF 88) 优于 1080P@低质量 (VMAF 82)。
固定阶梯强行在 1.5 Mbps 给 1080P，是错的。
```

阶梯选点规则：沿凸包取点，相邻档位满足两个条件之一即可作为一档——
1. VMAF 差值 ≥ 6（低于 6 人眼难辨，多一档纯属浪费）
2. 码率比值 ≥ 1.5（低于 1.5 时 ABR 切换收益不足）

#### 试编码成本的摊薄：用探针替代全量试编码

朴素的 per-title encoding 要跑完整网格试编码（如 5 分辨率 × 6 CRF = 30 次编码），成本是正常转码的 30 倍——完全不可行。

**解决方案：用 2 秒探针提取"空间复杂度 + 时间复杂度"两个特征，再用预训练模型预测凸包，而非实测凸包。**

```python
class ComplexityProbe:
    """转码前的内容复杂度探针——用 2 秒换到阶梯决策所需的信息

    核心思路：不做试编码（成本 30x），而是抽稀疏帧算复杂度特征，
    再用离线训练好的模型预测该内容的码率-画质曲线。
    """

    SAMPLE_FRAMES = 24          # 全片均匀抽 24 帧（覆盖不同场景）
    PROBE_TIMEOUT_SEC = 3       # 探针超时即放弃，退回固定阶梯

    def __init__(self, ffmpeg_client, model_client, config):
        self.ffmpeg = ffmpeg_client
        self.model = model_client       # 离线训练的复杂度→阶梯预测模型
        self.fallback_ladder = config['default_ladder']

    def probe(self, src_url, duration_ms):
        """返回内容复杂度特征 + 预测的最优阶梯"""
        try:
            # 1. 解封装探测：拿元数据（不解码，极快）
            meta = self.ffmpeg.probe(src_url)

            # 2. 抽稀疏帧解码（24 帧，而非全片）
            frames = self.ffmpeg.extract_frames(
                src_url, count=self.SAMPLE_FRAMES,
                timeout=self.PROBE_TIMEOUT_SEC)

            # 3. 空间复杂度：帧内细节丰富度（用 DCT 高频能量占比近似）
            spatial = self._spatial_complexity(frames)

            # 4. 时间复杂度：帧间运动量（用相邻抽样帧的 SAD 近似）
            temporal = self._temporal_complexity(frames)

            # 5. 场景切换密度（影响 GOP 结构与关键帧分配）
            scene_cuts = self._scene_cut_density(frames, duration_ms)

            features = {
                'spatial_complexity': spatial,       # 0-1
                'temporal_complexity': temporal,     # 0-1
                'scene_cut_per_min': scene_cuts,
                'src_width': meta['width'],
                'src_height': meta['height'],
                'src_bitrate': meta['bitrate'],
                'src_fps': meta['fps'],
                'grain_level': self._film_grain(frames),   # 胶片颗粒/噪点
                'has_text_overlay': self._detect_text(frames),  # 字幕/文字
            }

            # 6. 预测凸包 → 生成阶梯（模型在离线用真实试编码数据训练）
            ladder = self.model.predict_ladder(features)
            return features, ladder

        except Exception as e:
            # 探针失败不能阻塞转码——退回固定阶梯
            return None, self.fallback_ladder

    def _spatial_complexity(self, frames):
        """空间复杂度：高频能量占比越高，越需要码率"""
        import numpy as np
        scores = []
        for f in frames:
            gray = f.mean(axis=2) if f.ndim == 3 else f
            # 用梯度幅值均值近似细节丰富度（比完整 DCT 快 20 倍）
            gy, gx = np.gradient(gray)
            scores.append(float(np.sqrt(gx ** 2 + gy ** 2).mean()))
        # 归一化到 0-1（255 是经验上界）
        return min(1.0, float(np.mean(scores)) / 40.0)

    def _temporal_complexity(self, frames):
        """时间复杂度：相邻帧差越大，运动越剧烈，越需要码率"""
        import numpy as np
        if len(frames) < 2:
            return 0.0
        diffs = []
        for a, b in zip(frames[:-1], frames[1:]):
            diffs.append(float(np.abs(a.astype(float) - b.astype(float)).mean()))
        return min(1.0, float(np.mean(diffs)) / 30.0)

    def _scene_cut_density(self, frames, duration_ms):
        """场景切换密度（每分钟切换次数）"""
        import numpy as np
        if len(frames) < 2:
            return 0.0
        cuts = 0
        for a, b in zip(frames[:-1], frames[1:]):
            if float(np.abs(a.astype(float) - b.astype(float)).mean()) > 45:
                cuts += 1
        # 抽样帧覆盖全片，按比例外推
        minutes = max(duration_ms / 60000.0, 0.1)
        return cuts / minutes * (len(frames) / max(len(frames), 1))

    def _film_grain(self, frames):
        """噪点水平——噪点消耗大量码率但对观感贡献小，是降码率的重点"""
        import numpy as np
        scores = []
        for f in frames:
            gray = f.mean(axis=2) if f.ndim == 3 else f
            # 高斯模糊后的残差近似噪点
            blurred = self._box_blur(gray, 3)
            scores.append(float(np.abs(gray - blurred).mean()))
        return min(1.0, float(np.mean(scores)) / 12.0)

    def _box_blur(self, img, k):
        import numpy as np
        pad = k // 2
        padded = np.pad(img, pad, mode='edge')
        out = np.zeros_like(img, dtype=float)
        for dy in range(k):
            for dx in range(k):
                out += padded[dy:dy + img.shape[0], dx:dx + img.shape[1]]
        return out / (k * k)

    def _detect_text(self, frames):
        """检测是否有文字/字幕叠加——文字区域需要更高码率保清晰"""
        # 生产实现用轻量 OCR 检测器；此处用边缘密度的水平分布近似
        import numpy as np
        for f in frames:
            gray = f.mean(axis=2) if f.ndim == 3 else f
            bottom = gray[int(gray.shape[0] * 0.75):, :]
            gy, gx = np.gradient(bottom)
            if float(np.sqrt(gx ** 2 + gy ** 2).mean()) > 55:
                return True
        return False
```

**探针的成本收益：**

| 项 | 数值 |
|---|------|
| 探针耗时 | 2 秒/条（抽 24 帧解码 + 特征计算） |
| 探针算力成本 | ¥0.0008/条 → 505 万条/天 = ¥12 万/月 |
| 带来的码率节省 | 平均 19%（相比固定阶梯） |
| 带宽节省 | ¥2.3 亿 × 19% = **¥4,370 万/月** |
| **收益比** | **364 : 1** |

**关键设计：用 2 秒探针 + 离线训练的预测模型替代 30 倍成本的实测试编码，把 per-title encoding 的收益（19% 码率节省）以 1/364 的成本拿到手。**

**但需要注意：** 预测模型有误差。对预测置信度低的内容（特征落在训练分布边缘，如极端噪点、HDR、动画风格），退回到"保守阶梯"（码率上浮 15%）而不是硬用预测值——**宁可多花一点带宽，不可让画质掉到投诉线以下**。同时这些内容进入模型的再训练样本池。

#### 阶梯生成的实现

```python
class LadderGenerator:
    """ABR 阶梯生成器——按预测凸包选点"""

    # 档位候选（分辨率, 最小码率, 最大码率）
    RESOLUTION_TIERS = [
        (360,  240,   800),
        (480,  400,  1200),
        (720,  800,  3000),
        (1080, 1500,  8000),
        (1440, 3000, 14000),
        (2160, 6000, 32000),
    ]
    MIN_VMAF_GAP = 6.0        # 相邻档位最小 VMAF 差
    MIN_BITRATE_RATIO = 1.5   # 相邻档位最小码率比
    TARGET_TOP_VMAF = 95.0    # 最高档位目标画质

    def generate(self, features, src_height, tier_level):
        """
        tier_level: 1=头部内容(6档), 2=常规(4档), 3=长尾(2档)
        返回档位列表，每档含分辨率、码率、编码器、预设
        """
        # 1. 上限不超过源分辨率（禁止上采样——纯粹浪费码率）
        candidates = [t for t in self.RESOLUTION_TIERS if t[0] <= src_height]
        if not candidates:
            candidates = [self.RESOLUTION_TIERS[0]]

        # 2. 按复杂度预测每个分辨率达到目标画质所需的码率
        points = []
        for height, min_br, max_br in candidates:
            br = self._predict_bitrate(features, height, self.TARGET_TOP_VMAF)
            br = max(min_br, min(max_br, br))
            vmaf = self._predict_vmaf(features, height, br)
            points.append({'height': height, 'bitrate': br, 'vmaf': vmaf})

        # 3. 取凸包（剔除被支配的点）
        hull = self._convex_hull(points)

        # 4. 按 tier 决定档位数量，从凸包上等间距取点
        target_count = {1: 6, 2: 4, 3: 2}[tier_level]
        ladder = self._select_rungs(hull, target_count)

        # 5. 为每档分配编码器与预设（呼应分层编码策略）
        for rung in ladder:
            rung.update(self._assign_codec(rung, tier_level))

        return ladder

    def _convex_hull(self, points):
        """取(码率, VMAF)平面的上凸包——帕累托最优点集"""
        pts = sorted(points, key=lambda p: p['bitrate'])
        hull = []
        for p in pts:
            # 剔除：码率更高但 VMAF 不更高的点（被支配）
            while hull and hull[-1]['vmaf'] >= p['vmaf']:
                hull.pop()
            hull.append(p)
        return hull

    def _select_rungs(self, hull, target_count):
        """从凸包选档位：满足 VMAF 间距或码率比约束"""
        if len(hull) <= target_count:
            return list(hull)
        selected = [hull[0]]                  # 最低档必选（兜底）
        for p in hull[1:]:
            last = selected[-1]
            vmaf_gap = p['vmaf'] - last['vmaf']
            br_ratio = p['bitrate'] / max(last['bitrate'], 1)
            if vmaf_gap >= self.MIN_VMAF_GAP or br_ratio >= self.MIN_BITRATE_RATIO:
                selected.append(p)
            if len(selected) >= target_count:
                break
        if hull[-1] not in selected:
            selected[-1] = hull[-1]           # 最高档必选（保上限画质）
        return selected

    def _assign_codec(self, rung, tier_level):
        """按 tier 与分辨率分配编码器、预设、硬件"""
        # 4K 及以上强制 CPU 软编（GPU 硬编在 4K 画质损失可感知）
        if rung['height'] >= 2160:
            return {'codec': 'h265', 'preset': 'slow', 'hardware': 'cpu'}
        if tier_level == 1:      # 头部内容：多编码器 + 慢预设
            return {'codec': 'h265+av1', 'preset': 'slow', 'hardware': 'gpu'}
        if tier_level == 2:      # 常规：H.264 + H.265
            return {'codec': 'h264+h265', 'preset': 'medium', 'hardware': 'gpu'}
        return {'codec': 'h264', 'preset': 'veryfast', 'hardware': 'gpu'}

    def _predict_bitrate(self, features, height, target_vmaf):
        """预测达到目标 VMAF 所需码率（离线模型的在线推理，此处给简化公式）"""
        # 基线：分辨率的像素数正比
        pixels = height * height * 16 / 9
        base = pixels / 1000.0
        # 复杂度修正：空间与时间复杂度都推高码率需求
        c = (1 + 1.7 * features['temporal_complexity']
               + 1.1 * features['spatial_complexity']
               + 0.6 * features['grain_level'])
        # 文字叠加需额外码率保清晰
        if features.get('has_text_overlay'):
            c *= 1.12
        # 目标画质修正（VMAF 每高 1 分，码率约需 +7%）
        q = 1.07 ** (target_vmaf - 93)
        return int(base * c * q)

    def _predict_vmaf(self, features, height, bitrate):
        """给定码率预测 VMAF（用于凸包构建）"""
        import math
        need = self._predict_bitrate(features, height, 93.0)
        # 码率-画质是对数饱和关系
        return 93.0 + 14.0 * math.log(max(bitrate, 1) / max(need, 1), 2) / 3.0
```

### 分层编码策略：用"二次转码"解决信息不完整

回到挑战 2：编码格式的最优选择依赖播放量，而播放量在转码时未知。

**关键洞察：谁说转码只能做一次？** 把一次不可逆决策改成"先廉价决策，观察后再补"，就化解了信息不完整。

```
三层编码策略 + 热度晋升机制：

  发布时（信息不完整）→ 一律按 Tier 3 廉价编码
    └─ H.264，2 档（360P/720P），veryfast 预设
       编码成本：0.15 核时/分钟素材
       覆盖：全部 505 万条/天
       理由：此时不知道会不会火，先用最便宜的方式让内容可播

  发布后 6 小时（有初步数据）→ 播放 > 5000 的晋升 Tier 2
    └─ 补 H.265，扩到 4 档（+480P/1080P），medium 预设
       编码成本：0.9 核时/分钟素材
       覆盖：约 4.2 万条/天（0.8%）
       理由：已证明有流量，H.265 省 35% 码率在此播放量下回本

  发布后 24 小时（数据充分）→ 播放 > 100 万的晋升 Tier 1
    └─ 补 AV1，扩到 6 档，+窄带高清预处理，slow 预设
       编码成本：14 核时/分钟素材
       覆盖：约 1,500 条/天（0.03%）
       理由：这 0.03% 的内容将承载 60% 的播放量，
             在它们身上花 100 倍编码算力仍然大赚
```

**晋升机制的成本收益测算：**

| 策略 | 编码成本/月 | 带宽成本/月 | 合计/月 |
|------|-----------|-----------|--------|
| 全部 Tier 3（只 H.264 2 档） | ¥95 万 | ¥18,400 万 | ¥18,495 万 |
| 全部 Tier 1（一律最优编码） | ¥5,400 万 | ¥11,520 万 | ¥16,920 万 |
| 发布时按预测分层（预测会错） | ¥680 万 | ¥13,100 万 | ¥13,780 万 |
| **发布后按实际热度晋升** | **¥310 万** | **¥12,890 万** | **¥13,200 万** |

**"发布后按实际热度晋升"比"发布时按预测分层"每月省 ¥580 万**——因为预测播放量的准确率有限（AUC 约 0.78），预测错的代价是双向的（给冷内容做了贵编码 + 给热内容做了廉价编码）。而**等 6 小时拿到真实数据，准确率是 100%**。

**关键设计：当一个决策依赖未知信息时，先做最廉价的可逆决策，等信息到位后补做。这比努力提高预测精度更有效。**

**但需要注意：** 晋升期间（发布后 0-6 小时）内容只有 H.264 2 档。而短视频的黄金流量窗口恰好是发布后 2 小时——这段时间用的是最差的编码。实测这带来 8% 的额外带宽消耗（相比一开始就用 H.265）。这个代价是可接受的，因为：
- 只影响 0.8% 会晋升的内容在前 6 小时的流量（占总流量约 4%）
- 4% 流量 × 8% 额外码率 = 0.32% 总带宽 ≈ ¥74 万/月
- 相比"给 99.2% 的内容都做贵编码"的 ¥370 万/月浪费，明显更优

### 窄带高清：在同码率下拿更高画质

窄带高清不是单一技术，而是**编码前预处理 + 编码内决策优化**的组合。核心思想是"把码率花在人眼敏感的地方"。

| 技术 | 原理 | 码率节省 | 算力代价 |
|------|------|---------|---------|
| 自适应降噪 | 噪点消耗大量码率但无观感价值，编码前去除 | 8-14% | +15% |
| 去块/去振铃预处理 | 源片已有压缩伪影，直接再编码会放大 | 3-6% | +8% |
| ROI 感知码率分配 | 人脸/文字区域多给码率，背景少给 | 6-11% | +22% |
| 自适应量化（AQ） | 平坦区域用大量化步长，细节区用小步长 | 4-8% | +5% |
| 场景自适应 GOP | 场景切换处放关键帧，而非固定间隔 | 3-5% | +3% |
| 心理视觉优化（psy-rd） | 保留主观纹理感而非追求 PSNR | 5-9% | +12% |
| **组合（非线性叠加）** | — | **20-24%** | **+80%** |

**为什么组合效果不是简单相加？** 因为各技术优化的是同一批码率。降噪去掉的噪点码率，AQ 本来也会部分节省。实测组合收益约为单项之和的 62%。

```python
class NarrowbandHDPipeline:
    """窄带高清处理流水线——同码率下提升画质，或同画质下降低码率

    只对 Tier 1（头部内容）启用：算力代价 +80%，
    但这批内容承载 60% 播放量，带宽收益远超算力成本。
    """

    def __init__(self, ffmpeg_client, roi_detector, config):
        self.ffmpeg = ffmpeg_client
        self.roi = roi_detector          # 人脸/文字检测模型
        self.config = config

    def build_filter_chain(self, features, target_height):
        """根据探针特征动态构建滤镜链——不是固定一套参数套所有内容"""
        filters = []

        # 1. 降噪：强度与噪点水平挂钩（噪点少的内容不降噪，避免损失细节）
        grain = features['grain_level']
        if grain > 0.35:
            # 高噪点：强降噪，收益最大
            filters.append(f"hqdn3d=luma_spatial={4 * grain:.1f}:"
                           f"chroma_spatial={3 * grain:.1f}:"
                           f"luma_tmp={6 * grain:.1f}:chroma_tmp={4 * grain:.1f}")
        elif grain > 0.15:
            filters.append(f"hqdn3d=luma_spatial={2 * grain:.1f}:chroma_spatial=1.5")
        # grain <= 0.15：不降噪。强行降噪会磨平真实细节

        # 2. 去压缩伪影：源码率低说明已被压过，需要去块
        if features['src_bitrate'] < 2_000_000:
            filters.append("deblock=filter=weak:block=8")

        # 3. 缩放：用 Lanczos（比默认 bicubic 保留更多细节）
        filters.append(f"scale=-2:{target_height}:flags=lanczos")

        # 4. 胶片颗粒合成（AV1 专用）：编码前去噪，解码后合成回来
        #    这是 AV1 的杀手特性——噪点不占码率，但观感保留
        if self.config.get('codec') == 'av1' and grain > 0.25:
            filters.append(f"# film_grain_synthesis level={int(grain * 50)}")

        return ",".join(filters)

    def build_encoder_params(self, features, codec, bitrate_kbps):
        """构建编码器参数——AQ / psy-rd / GOP 均按内容特征调整"""
        params = {}

        if codec in ('h264', 'h265'):
            # 自适应量化模式：
            #   mode 1 = 方差based（适合一般内容）
            #   mode 3 = 自动方差 + 偏向暗部（适合暗场景多的内容）
            params['aq-mode'] = 3 if features['spatial_complexity'] > 0.5 else 1
            # AQ 强度：细节丰富的内容用更强 AQ
            params['aq-strength'] = round(0.8 + 0.6 * features['spatial_complexity'], 2)

            # 心理视觉优化：保主观纹理。文字内容要降低（避免文字边缘振铃）
            psy = 1.0
            if features.get('has_text_overlay'):
                psy = 0.6
            params['psy-rd'] = f"{psy}:0.15"

            # GOP：场景切换密集时缩短 GOP，让 ABR 切换更平滑
            cuts = features['scene_cut_per_min']
            gop_sec = 2 if cuts > 20 else 4
            params['keyint'] = int(gop_sec * features.get('src_fps', 30))
            params['scenecut'] = 40          # 启用场景切换检测插关键帧
            params['min-keyint'] = int(params['keyint'] / 4)

            # B 帧：静态内容多用 B 帧省码率；高动态内容减少 B 帧
            params['bframes'] = 3 if features['temporal_complexity'] > 0.6 else 8

        elif codec == 'av1':
            params['enable-qm'] = 1          # 量化矩阵
            params['film-grain'] = int(features['grain_level'] * 50)
            params['tune'] = 'ssim'
            params['enable-tf'] = 1          # 时域滤波

        return params

    def build_roi_map(self, src_url, duration_ms):
        """
        ROI 码率分配：检测人脸与文字区域，生成 qp offset 图。
        人脸区域 qp -4（更高质量），背景 qp +2（更低质量）。
        净效果：同总码率下主观画质提升，等效省 6-11% 码率。
        """
        # 每 2 秒抽一帧做检测（ROI 位置变化不会太快）
        sample_interval_ms = 2000
        roi_maps = []
        for ts in range(0, duration_ms, sample_interval_ms):
            frame = self.ffmpeg.extract_frame_at(src_url, ts)
            regions = self.roi.detect(frame)      # 返回 [(x,y,w,h,type)]
            qp_offsets = []
            for (x, y, w, h, rtype) in regions:
                offset = -4 if rtype == 'face' else (-3 if rtype == 'text' else 0)
                qp_offsets.append({'rect': (x, y, w, h), 'qp_offset': offset})
            roi_maps.append({'timestamp_ms': ts, 'regions': qp_offsets})
        return roi_maps
```

**关键设计：滤镜与编码参数全部由探针特征驱动，而非固定一套配置。** 一套固定参数在某些内容上有效、在另一些上有害——比如对本身干净的内容强行降噪会磨平细节，反而降低画质。**"按内容自适应"是窄带高清与普通转码的本质区别。**

### 转码集群调度：长任务、抢占、成本感知

回到挑战 3。转码任务时长跨 4 个数量级，朴素 FIFO 会让一条 90 分钟长视频阻塞后面 1000 条短视频。

#### 分片转码：把长任务变成短任务

**核心手段是分片。** 把长视频按 GOP 边界切成 2-5 分钟的片段，各片段独立并行转码，最后拼接。这一步同时解决三个问题：

```
分片转码的收益：

  未分片（一条 90 分钟 4K）
    └─ 单机串行 90 分钟 → 期间占用 1 张 GPU 卡
       · 阻塞：后续任务等 90 分钟
       · 失败代价：中途失败重来 90 分钟
       · 无法抢占：杀掉就全丢

  分片后（切成 30 片 × 3 分钟）
    └─ 30 片分发到 30 张卡并行 → 墙钟 3 分钟 + 拼接 40 秒
       · 阻塞：单片只占 3 分钟，调度粒度细
       · 失败代价：只重做失败的那 1 片
       · 可抢占：高优任务来了，让出部分片段的卡
       · 墙钟从 90 分钟降到 3.7 分钟（24 倍加速）
```

**分片的技术前提与坑：**

| 要求 | 原因 | 不满足的后果 |
|------|------|------------|
| 切点必须在关键帧（IDR）边界 | 非关键帧处切分会导致该片首帧无法解码 | 片段开头花屏 |
| 各片段用相同编码参数与 GOP 结构 | 参数不一致会导致拼接处画质跳变 | 观众可见的"一卡一卡"感 |
| 各片段码率控制需用固定 CRF 而非 ABR | ABR 的码率控制是全片统计的，分片后各片独立统计会不一致 | 片段间码率跳变 |
| 音频单独整轨转码，不分片 | 音频分片拼接会有爆音 | 拼接处爆音 |
| 拼接用 concat 协议而非重编码 | 重编码会二次损失画质 | 画质劣化 |

**关键设计：音频不分片。** 视频分 30 片并行，音频作为第 31 个任务整轨转码（音频转码极快，42 分钟音频只需 25 秒）。这避免了音频拼接的爆音问题，代价可忽略。

#### 调度器实现

```python
import heapq


class TranscodeScheduler:
    """转码集群调度器——多级优先级 + 分片 + 抢占 + 成本感知

    调度目标（按优先级）：
      1. 代理片任务永不排队（审核链路阻塞代价最高）
      2. 短视频 P95 < 2 分钟
      3. 长视频 P95 < 14 分钟
      4. 长尾任务不饿死（老化提权）
      5. 在满足以上前提下最小化成本（优先自建，溢出才上云）
    """

    # 优先级定义（数字越小越优先）
    PRIO_PROXY = 0          # 代理片：审核依赖，绝对最高
    PRIO_HOT_CREATOR = 1    # 头部创作者
    PRIO_SHORT_VIDEO = 2    # 短视频（时效敏感）
    PRIO_PROMOTION = 3      # 热度晋升的二次转码
    PRIO_LONG_VIDEO = 4     # 长视频
    PRIO_BACKFILL = 5       # 补转码/回填（可无限延后）

    # 分片阈值：超过此时长才分片（短视频分片的开销大于收益）
    SHARD_THRESHOLD_SEC = 300
    SHARD_TARGET_SEC = 180          # 每片目标时长 3 分钟
    MAX_SHARDS_PER_TASK = 60        # 单任务最多 60 片（防止极长视频占满集群）

    # 老化提权：等待超过阈值则每档提升一级，防止饿死
    AGING_INTERVAL_SEC = 300

    # 成本感知：自建集群利用率超此值才允许溢出到云
    CLOUD_OVERFLOW_THRESHOLD = 0.88

    def __init__(self, redis_client, mysql_client, cluster_manager, config):
        self.redis = redis_client
        self.mysql = mysql_client
        self.cluster = cluster_manager
        self.cloud = config['cloud_transcode_client']
        self.alert = config['alert_service']

    # ── 任务入队 ──

    def submit(self, task):
        """
        task: {content_id, asset_id, duration_sec, src_height,
               tier_level, author_level, is_proxy, ladder}
        """
        prio = self._calc_priority(task)

        # 1. 决定是否分片
        if (not task['is_proxy']
                and task['duration_sec'] > self.SHARD_THRESHOLD_SEC):
            shards = self._plan_shards(task)
            self._enqueue_shards(task, shards, prio)
        else:
            self._enqueue_single(task, prio)

    def _calc_priority(self, task):
        """优先级计算——代理片 > 头部创作者 > 短视频 > 晋升 > 长视频 > 回填"""
        if task['is_proxy']:
            return self.PRIO_PROXY
        if task.get('is_backfill'):
            return self.PRIO_BACKFILL
        if task.get('is_promotion'):
            return self.PRIO_PROMOTION
        if task['author_level'] >= 5:
            return self.PRIO_HOT_CREATOR
        if task['duration_sec'] <= 120:
            return self.PRIO_SHORT_VIDEO
        return self.PRIO_LONG_VIDEO

    def _plan_shards(self, task):
        """
        规划分片：必须切在关键帧边界。
        关键帧位置来自阶段0探针提取的 GOP 索引。
        """
        keyframes = self._get_keyframe_timestamps(task['asset_id'])
        if not keyframes:
            return None      # 拿不到关键帧索引，退回不分片

        shards, start_idx = [], 0
        while start_idx < len(keyframes) - 1:
            start_ts = keyframes[start_idx]
            # 找到距 start_ts 约 SHARD_TARGET_SEC 的下一个关键帧
            end_idx = start_idx
            for i in range(start_idx + 1, len(keyframes)):
                if keyframes[i] - start_ts >= self.SHARD_TARGET_SEC:
                    end_idx = i
                    break
            else:
                end_idx = len(keyframes) - 1

            if end_idx == start_idx:
                end_idx = min(start_idx + 1, len(keyframes) - 1)

            shards.append({
                'shard_index': len(shards),
                'start_ts': start_ts,
                'end_ts': keyframes[end_idx],
            })
            start_idx = end_idx

            if len(shards) >= self.MAX_SHARDS_PER_TASK:
                # 超过上限：剩余部分合成一个大片（宁可慢也不占满集群）
                shards.append({
                    'shard_index': len(shards),
                    'start_ts': keyframes[start_idx],
                    'end_ts': None,          # 到结尾
                })
                break

        return shards

    def _enqueue_shards(self, task, shards, prio):
        """分片入队 + 登记拼接依赖"""
        if shards is None:
            return self._enqueue_single(task, prio)

        # 登记父任务，记录需要拼接的片数
        self.mysql.execute("""
            INSERT INTO transcode_task
                (content_id, asset_id, task_type, priority, shard_total,
                 shard_done, status, ladder_json)
            VALUES (%s, %s, 'sharded', %s, %s, 0, 'running', %s)
        """, (task['content_id'], task['asset_id'], prio, len(shards),
              self._json(task['ladder'])))

        # 每档位 × 每分片 = 一个独立子任务
        for rung in task['ladder']:
            for shard in shards:
                self._push(prio, {
                    'content_id': task['content_id'],
                    'asset_id': task['asset_id'],
                    'rung': rung,
                    'shard': shard,
                    'est_cost_sec': self._estimate_cost(task, rung, shard),
                })

    def _push(self, prio, subtask):
        """
        推入优先级队列。用 Redis ZSET，score = 优先级 * 1e10 + 入队时间戳。
        这样同优先级内部按 FIFO，不同优先级严格分层。
        """
        import time
        score = prio * 1e10 + time.time()
        self.redis.zadd("transcode:queue", {self._json(subtask): score})

    # ── 任务出队与资源匹配 ──

    def acquire(self, worker):
        """
        worker 拉取任务。资源匹配规则：
          · 4K 及以上 → 只给 CPU worker（GPU 硬编 4K 画质损失可感知）
          · 1080P 及以下 → 优先 GPU worker
          · AV1 → 只给支持 AV1 硬编的卡，或 CPU
        """
        for _ in range(50):        # 最多试探 50 个候选任务
            items = self.redis.zpopmin("transcode:queue", 1)
            if not items:
                return None
            raw, score = items[0]
            subtask = self._parse(raw)

            if self._match(worker, subtask):
                self._mark_running(subtask, worker)
                return subtask

            # 不匹配：放回队列（保持原 score，不影响其排序）
            self.redis.zadd("transcode:queue", {raw: score})

        return None

    def _match(self, worker, subtask):
        """worker 能力与任务需求匹配"""
        rung = subtask['rung']
        if rung['height'] >= 2160 and worker['type'] != 'cpu':
            return False
        if 'av1' in rung['codec'] and not worker.get('supports_av1'):
            return worker['type'] == 'cpu'
        return True

    # ── 抢占 ──

    def try_preempt(self, incoming_prio):
        """
        高优任务到达但无空闲资源时，抢占低优任务。
        只抢占 PRIO_BACKFILL 与 PRIO_LONG_VIDEO 的分片任务
        （分片任务被杀只损失 3 分钟，未分片长任务不抢占）。
        """
        if incoming_prio > self.PRIO_SHORT_VIDEO:
            return None        # 只有短视频及更高优才触发抢占

        victims = self.cluster.list_running(
            min_priority=self.PRIO_LONG_VIDEO, sharded_only=True)
        if not victims:
            return None

        # 选择已运行时间最短的（损失最小）
        victim = min(victims, key=lambda v: v['elapsed_sec'])
        if victim['elapsed_sec'] > 60:
            return None        # 已跑 60 秒以上，杀掉不划算，让它跑完

        self.cluster.kill(victim['task_id'])
        # 被抢占的任务重新入队，并提升优先级（避免反复被抢占而饿死）
        self._push(max(self.PRIO_SHORT_VIDEO, victim['priority'] - 1),
                   victim['subtask'])
        return victim['worker']

    # ── 老化提权（防饿死）──

    def age_queue(self):
        """每 60 秒执行：等待过久的任务提升优先级"""
        import time
        now = time.time()
        # 扫描队列中等待超过阈值的任务
        candidates = self.redis.zrange("transcode:queue", 0, 2000,
                                       withscores=True)
        for raw, score in candidates:
            prio = int(score // 1e10)
            enqueued_at = score - prio * 1e10
            waited = now - enqueued_at
            if prio <= self.PRIO_SHORT_VIDEO:
                continue        # 高优任务无需提权
            steps = int(waited // self.AGING_INTERVAL_SEC)
            if steps <= 0:
                continue
            new_prio = max(self.PRIO_SHORT_VIDEO, prio - steps)
            if new_prio != prio:
                self.redis.zadd("transcode:queue",
                                {raw: new_prio * 1e10 + enqueued_at})

    # ── 成本感知的云溢出 ──

    def check_cloud_overflow(self):
        """
        自建集群饱和时溢出到云。关键是"只溢出，不迁移"——
        云单价是自建 2.5 倍，只用于消峰，不承担基线负载。
        """
        util = self.cluster.utilization()
        backlog = self.redis.zcard("transcode:queue")
        throughput = self.cluster.throughput_per_sec()
        drain_sec = backlog / max(throughput, 0.001)

        # 双条件：自建接近饱和 且 积压确实消化不掉
        if util < self.CLOUD_OVERFLOW_THRESHOLD or drain_sec < 600:
            return 0

        # 计算需要溢出多少任务才能把消化时长压到 10 分钟内
        target_drain = 600
        excess = int(backlog - throughput * target_drain)
        excess = max(0, min(excess, 20000))     # 单次溢出上限，防成本失控

        # 只溢出高优任务（付了 2.5 倍价钱要买到时效）
        overflowed = 0
        for _ in range(excess):
            items = self.redis.zpopmin("transcode:queue", 1)
            if not items:
                break
            subtask = self._parse(items[0][0])
            self.cloud.submit(subtask)
            overflowed += 1

        if overflowed > 0:
            cost = overflowed * 0.08        # 估算单任务云成本
            self.alert.fire("cloud_overflow", severity="P2", detail={
                'tasks': overflowed, 'est_cost_yuan': round(cost, 2),
                'cluster_util': round(util, 3), 'drain_sec': int(drain_sec),
            })
        return overflowed

    # ── 分片拼接 ──

    def on_shard_done(self, content_id, rung_key, shard_index, output_key):
        """
        单个分片完成。所有分片就绪后触发拼接。
        用 Redis 原子递增判断"是否最后一片"，避免竞争。
        """
        done_key = f"transcode:shards:{content_id}:{rung_key}"
        self.redis.hset(done_key, shard_index, output_key)
        self.redis.expire(done_key, 86400)

        done_count = self.redis.hlen(done_key)
        total = int(self.redis.hget(
            f"transcode:meta:{content_id}", "shard_total") or 0)

        if total > 0 and done_count == total:
            # 最后一片：按 index 排序后 concat 拼接（不重编码）
            parts = self.redis.hgetall(done_key)
            ordered = [parts[str(i).encode()].decode()
                       for i in range(total)]
            self._concat(content_id, rung_key, ordered)

    def _concat(self, content_id, rung_key, ordered_parts):
        """用 FFmpeg concat demuxer 拼接——纯复制流，不重编码，无画质损失"""
        self.cluster.submit_concat_job({
            'content_id': content_id,
            'rung_key': rung_key,
            'parts': ordered_parts,
            'mode': 'stream_copy',      # -c copy，关键：不重编码
        })

    # ── 辅助 ──

    def _estimate_cost(self, task, rung, shard):
        """估算子任务耗时，用于容量规划与超时判定"""
        dur = ((shard['end_ts'] or task['duration_sec']) - shard['start_ts'])
        # 实时倍率：GPU 硬编 0.3x，CPU 软编 3.0x，按分辨率缩放
        ratio = 3.0 if rung['height'] >= 2160 else 0.3
        scale = (rung['height'] / 1080.0) ** 1.8
        preset_factor = {'veryfast': 0.5, 'medium': 1.0, 'slow': 3.2}.get(
            rung.get('preset', 'medium'), 1.0)
        return dur * ratio * scale * preset_factor

    def _json(self, obj):
        import json
        return json.dumps(obj, ensure_ascii=False, sort_keys=True)

    def _parse(self, raw):
        import json
        return json.loads(raw if isinstance(raw, str) else raw.decode())
```

**调度效果对比：**

| 调度策略 | 短视频 P95 | 长视频 P95 | 长尾饿死率 | GPU 利用率 |
|---------|-----------|-----------|-----------|-----------|
| 纯 FIFO | 18 min | 22 min | 0% | 71% |
| 纯优先级（无老化） | 52 s | 11 min | **8.2%** | 82% |
| 优先级 + 分片 | 48 s | **3.7 min** | 6.1% | 86% |
| **优先级 + 分片 + 老化 + 抢占** | **44 s** | **3.9 min** | **0.02%** | **91%** |

**关键设计：分片把长视频墙钟从 22 分钟压到 3.7 分钟（24 倍），老化提权把饿死率从 8.2% 压到 0.02%。两者都是必需的——只做优先级会饿死长尾，只做分片解决不了优先级倒挂。**

### JIT 按需转码：长尾高档位的解法

回到挑战 5：长尾内容预转全档位是浪费（298 PB/年），只转 2 档则高网速用户拿不到高清。

**解法是 JIT + 邻档兜底 + 异步补齐三件套：**

```
用户请求一条长尾内容的 1080P（该档位未预转码）：

  T+0ms    播放服务查 media_variant → 1080P 状态为 'not_generated'
             │
             ├─ 立即返回最接近的已有档位（720P）+ 标记 upgrading=true
             │  用户 0 延迟起播（关键：绝不让用户等）
             │
             └─ 同时投递 JIT 转码任务（优先级 = PRIO_SHORT_VIDEO）
                   │
  T+40s        ├─ 短视频：JIT 完成 → 推送"高清已就绪"给客户端
                │    客户端在下一个分片边界无缝切到 1080P
                │
  T+3.9min     └─ 长视频（分片并行）：完成 → 同样无缝切换
                     用户此时可能已经看完了——但档位已落盘，
                     下一个观众直接享受 1080P

  副作用（正向）：JIT 生成的档位会落盘并计入 play_count_7d，
                 若持续有人播放则保留，7 日无播放则回收。
                 这形成了"按真实需求分配存储"的自适应闭环。
```

```python
class JITTranscodeService:
    """按需转码服务——长尾内容的高档位在首次请求时生成"""

    # 邻档兜底：请求档位不可用时的降级顺序
    FALLBACK_CHAIN = {
        2160: [1440, 1080, 720, 480, 360],
        1440: [1080, 720, 480, 360],
        1080: [720, 1440, 480, 360],      # 优先降级，其次升级
        720:  [480, 1080, 360],
        480:  [360, 720],
        360:  [480, 720],
    }
    # 同一档位的 JIT 请求去重窗口（防止爆款瞬间提交上千个相同任务）
    DEDUP_WINDOW_SEC = 600
    # JIT 触发的最小请求数：单个用户的请求不值得触发转码
    MIN_REQUESTS_TO_TRIGGER = 3

    def __init__(self, redis_client, mysql_client, scheduler, config):
        self.redis = redis_client
        self.mysql = mysql_client
        self.scheduler = scheduler

    def resolve_playback(self, content_id, asset_id, requested_height,
                         user_context):
        """
        返回可立即播放的档位。核心原则：绝不让用户等待转码。
        """
        available = self._get_available_variants(asset_id)

        # 1. 请求的档位已就绪 → 直接返回
        if requested_height in available:
            return {'height': requested_height,
                    'variant': available[requested_height],
                    'upgrading': False}

        # 2. 未就绪 → 按兜底链找最接近的可用档位
        fallback = None
        for h in self.FALLBACK_CHAIN.get(requested_height, []):
            if h in available:
                fallback = h
                break
        if fallback is None:
            # 连兜底都没有——说明转码彻底失败，返回错误让客户端提示
            return {'error': 'NO_PLAYABLE_VARIANT'}

        # 3. 累计请求计数，达阈值才触发 JIT（避免为单次请求转码）
        should_trigger = self._count_and_check(
            asset_id, requested_height)

        if should_trigger:
            self._submit_jit(content_id, asset_id, requested_height)

        return {'height': fallback,
                'variant': available[fallback],
                'upgrading': should_trigger,
                'upgrade_target': requested_height}

    def _count_and_check(self, asset_id, height):
        """请求计数 + 去重：达到阈值且未在转码中才触发"""
        # 去重：已在转码中则不重复提交
        lock_key = f"jit:inflight:{asset_id}:{height}"
        if self.redis.exists(lock_key):
            return True        # 已在转码，返回 upgrading=true 让客户端等推送

        cnt_key = f"jit:demand:{asset_id}:{height}"
        cnt = self.redis.incr(cnt_key)
        self.redis.expire(cnt_key, self.DEDUP_WINDOW_SEC)

        if cnt < self.MIN_REQUESTS_TO_TRIGGER:
            return False

        # SET NX 保证只有一个请求成功触发转码
        if self.redis.set(lock_key, "1", nx=True, ex=3600):
            return True
        return True

    def _submit_jit(self, content_id, asset_id, height):
        """提交 JIT 转码任务"""
        meta = self.mysql.query(
            "SELECT duration_ms, src_height, src_bitrate FROM media_asset "
            "WHERE asset_id=%s", (asset_id,))
        # JIT 任务用 veryfast 预设——此时时效优先于码率效率
        self.scheduler.submit({
            'content_id': content_id,
            'asset_id': asset_id,
            'duration_sec': meta['duration_ms'] / 1000.0,
            'src_height': meta['src_height'],
            'tier_level': 3,
            'author_level': 0,
            'is_proxy': False,
            'ladder': [{'height': height, 'codec': 'h264',
                        'preset': 'veryfast', 'hardware': 'gpu',
                        'bitrate': self._default_bitrate(height)}],
        })

    def on_jit_ready(self, content_id, asset_id, height, variant_key):
        """JIT 完成：落库 + 推送客户端无缝切换"""
        self.mysql.execute("""
            UPDATE media_variant SET status=1, manifest_key=%s, gen_mode=2
            WHERE asset_id=%s AND quality_level=%s
        """, (variant_key, asset_id, self._level(height)))

        self.redis.delete(f"jit:inflight:{asset_id}:{height}")

        # 推送给正在观看该内容且请求过此档位的客户端
        self.redis.publish(f"quality_ready:{content_id}", self._json({
            'height': height, 'manifest_key': variant_key,
        }))
```

**JIT 的成本收益：**

| 项 | 全档位预转码 | 2 档预转 + JIT |
|---|-------------|---------------|
| 转码算力 | ¥1,120 万/月 | ¥310 万/月 |
| 转码副本存储 | 350 PB/年 | 53 PB/年 |
| 存储成本 | ¥824 万/月 | ¥125 万/月 |
| 高档位可用性 | 100% 立即 | 首次请求降级，40s-4min 后就绪 |
| 受影响请求占比 | 0% | 约 3.2% 的请求首次拿到降级档位 |
| **合计成本** | **¥1,944 万/月** | **¥435 万/月（省 78%）** |

**但需要注意：** JIT 的"首次请求降级"有一个恶性场景——**冷内容突然爆红**。一条沉底半年的视频突然上热搜，瞬间涌入 50 万请求，全部拿到 720P 降级档位，且 JIT 任务因优先级不够而排队。

应对：把"JIT 请求数激增"作为热度信号接入调度器——同一 asset 的 JIT 需求计数在 60 秒内超过 1000，则该任务优先级直接提到 `PRIO_HOT_CREATOR`，并同时触发 Tier 晋升（补全档位 + H.265）。**JIT 的需求信号本身就是最准的热度信号，比播放量统计更实时。**

### 转码产物质量校验

回到挑战 4：转码可能产出"格式合法但内容错误"的产物，FFmpeg 返回码 0，只能靠主动检测。

```python
class TranscodeQualityGate:
    """转码产物质检闸门——拦住花屏/黑屏/音画不同步/时长异常的产物

    成本约束：质检算力必须 < 转码算力的 5%。
    因此全部检测都基于稀疏抽帧，不做全片解码。
    """

    # 检测阈值
    VMAF_MIN = {360: 78, 480: 82, 720: 85, 1080: 88, 1440: 89, 2160: 90}
    DURATION_TOLERANCE = 0.02        # 时长偏差容忍 2%
    BLACK_FRAME_RATIO_MAX = 0.03     # 黑帧占比上限 3%
    AV_SYNC_TOLERANCE_MS = 120       # 音画同步容忍 120ms
    VMAF_SAMPLE_FRAMES = 12          # VMAF 只抽 12 帧算（全片算太贵）

    def __init__(self, ffmpeg_client, config):
        self.ffmpeg = ffmpeg_client
        self.alert = config['alert_service']

    def check(self, src_url, out_url, expected_height, expected_duration_ms):
        """
        返回 (通过?, 失败项列表)。任一项失败则产物不上线。
        """
        failures = []

        # ── 检测 1：容器与流完整性（最便宜，先做）──
        meta = self.ffmpeg.probe(out_url)
        if meta is None:
            return False, ['PROBE_FAILED']

        if not meta.get('video_streams'):
            failures.append('NO_VIDEO_STREAM')
        if not meta.get('audio_streams') and expected_duration_ms > 3000:
            failures.append('NO_AUDIO_STREAM')

        # ── 检测 2：分辨率与时长 ──
        if meta.get('height') != expected_height:
            failures.append(
                f"HEIGHT_MISMATCH(expect={expected_height},got={meta.get('height')})")

        dur_diff = abs(meta['duration_ms'] - expected_duration_ms)
        if dur_diff / max(expected_duration_ms, 1) > self.DURATION_TOLERANCE:
            failures.append(
                f"DURATION_MISMATCH(expect={expected_duration_ms},"
                f"got={meta['duration_ms']})")

        # ── 检测 3：黑帧/纯色帧检测（花屏与黑屏的主要表现）──
        black_ratio = self._black_frame_ratio(out_url, meta['duration_ms'])
        if black_ratio > self.BLACK_FRAME_RATIO_MAX:
            failures.append(f"TOO_MANY_BLACK_FRAMES({black_ratio:.3f})")

        # ── 检测 4：花屏检测（块状伪影异常）──
        if self._detect_corruption(out_url, meta['duration_ms']):
            failures.append('VISUAL_CORRUPTION')

        # ── 检测 5：音频静音检测 ──
        if meta.get('audio_streams'):
            silence_ratio = self._silence_ratio(out_url, meta['duration_ms'])
            if silence_ratio > 0.95:
                failures.append(f"AUDIO_ALL_SILENT({silence_ratio:.3f})")

        # ── 检测 6：音画同步 ──
        sync_offset = self._av_sync_offset(out_url)
        if abs(sync_offset) > self.AV_SYNC_TOLERANCE_MS:
            failures.append(f"AV_DESYNC({sync_offset}ms)")

        # ── 检测 7：VMAF 客观画质（最贵，放最后，前面失败就不做了）──
        if not failures:
            vmaf = self._sampled_vmaf(src_url, out_url)
            threshold = self.VMAF_MIN.get(expected_height, 85)
            if vmaf < threshold:
                failures.append(f"LOW_VMAF({vmaf:.1f}<{threshold})")

        return len(failures) == 0, failures

    def _black_frame_ratio(self, url, duration_ms):
        """抽 30 帧检测纯黑/纯色帧占比"""
        frames = self.ffmpeg.extract_frames(url, count=30)
        black = 0
        for f in frames:
            if f is None:
                continue
            # 亮度均值极低 且 方差极小 → 纯黑帧
            mean = float(f.mean())
            std = float(f.std())
            if mean < 12 and std < 6:
                black += 1
        return black / max(len(frames), 1)

    def _detect_corruption(self, url, duration_ms):
        """
        花屏检测：花屏的特征是出现规律的 8x8/16x16 块状边界。
        用水平/垂直方向上"块边界处梯度显著高于块内梯度"来识别。
        """
        import numpy as np
        frames = self.ffmpeg.extract_frames(url, count=16)
        suspicious = 0
        for f in frames:
            if f is None:
                continue
            gray = f.mean(axis=2) if f.ndim == 3 else f
            h, w = gray.shape
            if h < 32 or w < 32:
                continue
            # 计算列方向的相邻差
            col_diff = np.abs(np.diff(gray, axis=1)).mean(axis=0)
            # 块边界位置（每 16 列）
            boundary = col_diff[15::16]
            interior = np.delete(col_diff, np.arange(15, len(col_diff), 16))
            if len(boundary) == 0 or len(interior) == 0:
                continue
            # 块边界梯度显著高于块内 → 强块效应 → 疑似花屏
            if boundary.mean() > interior.mean() * 3.2:
                suspicious += 1
        # 超过 1/4 的抽样帧疑似花屏才判定（避免误杀本身块效应重的低码率内容）
        return suspicious > len(frames) * 0.25

    def _silence_ratio(self, url, duration_ms):
        """音频静音占比——用 FFmpeg silencedetect 滤镜"""
        result = self.ffmpeg.detect_silence(url, noise_db=-50, min_dur=0.5)
        silent_ms = sum(s['duration_ms'] for s in result)
        return silent_ms / max(duration_ms, 1)

    def _av_sync_offset(self, url):
        """音画同步偏移：比较视频与音频流的首个 PTS"""
        meta = self.ffmpeg.probe(url, show_streams=True)
        v_start = meta['video_streams'][0].get('start_time_ms', 0)
        a_start = (meta['audio_streams'][0].get('start_time_ms', 0)
                   if meta.get('audio_streams') else 0)
        return int(v_start - a_start)

    def _sampled_vmaf(self, src_url, out_url):
        """
        抽样 VMAF：只在 12 个时间点各取 1 秒计算，取均值。
        全片 VMAF 成本约为转码的 40%，抽样后降到 2%。
        """
        return self.ffmpeg.vmaf(
            src_url, out_url,
            sample_points=self.VMAF_SAMPLE_FRAMES,
            sample_duration_sec=1.0)

    def on_failure(self, content_id, rung, failures):
        """质检失败的处置：分类重试"""
        # 可通过换参数重试解决的
        RETRIABLE = {'VISUAL_CORRUPTION', 'AV_DESYNC', 'PROBE_FAILED'}
        # 提高码率可解决的
        BITRATE_FIXABLE = {f for f in failures if f.startswith('LOW_VMAF')}
        # 源片问题，重试无用
        SOURCE_ISSUE = {'NO_VIDEO_STREAM', 'DURATION_MISMATCH'}

        fset = set(failures)
        if fset & SOURCE_ISSUE:
            return {'action': 'FAIL_PERMANENT',
                    'notify_creator': True,
                    'message': '源视频文件异常，请重新上传'}
        if BITRATE_FIXABLE:
            return {'action': 'RETRY',
                    'adjust': {'bitrate_multiplier': 1.25}}
        if fset & RETRIABLE:
            return {'action': 'RETRY',
                    'adjust': {'preset': 'medium', 'hardware': 'cpu'}}
        return {'action': 'RETRY', 'adjust': {}}
```

**质检的成本占比：**

| 检测项 | 单条成本 | 占转码成本 |
|-------|---------|-----------|
| 容器/流完整性（probe） | ¥0.00002 | 0.01% |
| 分辨率/时长 | 包含在 probe | 0% |
| 黑帧检测（抽 30 帧） | ¥0.0003 | 0.15% |
| 花屏检测（抽 16 帧） | ¥0.0004 | 0.20% |
| 静音检测 | ¥0.0002 | 0.10% |
| 音画同步 | 包含在 probe | 0% |
| 抽样 VMAF（12 点） | ¥0.0068 | 3.40% |
| **合计** | **¥0.0077** | **3.86%** |

**关键设计：检测顺序按"成本从低到高"排列，前面失败就不做后面的。** VMAF 占质检成本的 88%，所以它必须放在最后——只有通过了所有廉价检测的产物才值得花钱算 VMAF。这个排序让实际平均成本远低于 3.86%（因为约 0.4% 的产物在廉价检测阶段就被拦下）。

**但需要注意：** 抽样 VMAF 会漏掉"局部严重劣化"——比如全片只有第 37 分钟有 10 秒花屏，12 个抽样点恰好都没落在那里。应对是**抽样点非均匀分布**：在场景切换处、高运动段、码率突变处加密采样（这些位置最容易出问题），而非均匀抽样。这把漏检率从 1.2% 降到 0.09%。

### 并行支线：封面、字幕、水印、DRM

这四项与主转码链路**并行执行，不阻塞 MediaReady**。它们的共同特点是"缺失不影响播放，但影响体验或合规"。

| 支线 | 输入 | 耗时 | 缺失后果 | 兜底策略 |
|------|------|------|---------|---------|
| 封面抽帧 + 美学打分 | 代理片 | 8 s | 无封面，Feed 展示空白 | 用第 1 秒帧兜底 |
| 雪碧图（进度条预览） | 代理片 | 12 s | 拖进度条无预览 | 降级为无预览 |
| ASR 字幕 | 音频轨 | 45 s | 无字幕 | 用户可手动关闭，非阻塞 |
| 字幕翻译（多语言） | ASR 结果 | 30 s | 无外语字幕 | 仅影响出海用户 |
| 明水印 | 各档位 | 与转码同步 | 内容被搬运无标识 | 必须有，否则无法追溯 |
| 盲水印（溯源） | 各档位 | +6% 转码耗时 | 泄露源无法定位 | 仅 Tier 1 内容必须 |
| DRM 加密 | 打包后分片 | 18 s | 付费内容可被下载 | **付费内容必须有，不可降级** |
| 视频/音频指纹 | 代理片 | 15 s | 无法做版权比对 | 异步补，不阻塞 |

**封面选择的美学打分**是一个容易被低估的收益点——封面直接决定 Feed 点击率。实测"随机取第 1 帧"与"美学打分选最优帧"的 CTR 差异达 23%。

```python
class CoverSelector:
    """智能封面选择——从候选帧中选点击率最高的

    评分维度不是"好看"，而是"能带来点击"。这两者不完全等价：
    清晰的人脸特写往往比构图精美的风景 CTR 更高。
    """

    CANDIDATE_COUNT = 30          # 抽 30 帧做候选
    # 各维度权重（离线用真实 CTR 数据回归得到）
    WEIGHTS = {
        'sharpness': 0.18,        # 清晰度（模糊帧 CTR 极低）
        'face_quality': 0.26,     # 人脸大小与正面度（最强正相关）
        'colorfulness': 0.12,     # 色彩丰富度
        'brightness_fit': 0.10,   # 亮度适中（过暗过曝都差）
        'composition': 0.09,      # 构图（主体居中/三分法）
        'text_penalty': -0.15,    # 帧内已有大量文字 → 与标题重复，降权
        'motion_blur_penalty': -0.20,   # 运动模糊 → 强降权
    }

    def __init__(self, ffmpeg_client, face_detector, config):
        self.ffmpeg = ffmpeg_client
        self.face = face_detector

    def select(self, proxy_url, duration_ms, exclude_head_ms=1000):
        """
        从代理片中选最优封面。
        exclude_head_ms: 跳过开头（片头/黑场常在开头）
        """
        frames = self.ffmpeg.extract_frames_with_ts(
            proxy_url, count=self.CANDIDATE_COUNT,
            start_ms=exclude_head_ms,
            end_ms=max(duration_ms - 500, exclude_head_ms + 1000))

        scored = []
        for frame, ts in frames:
            if frame is None:
                continue
            score = self._score(frame)
            scored.append({'timestamp_ms': ts, 'score': score,
                           'frame': frame})

        if not scored:
            return None

        scored.sort(key=lambda x: x['score'], reverse=True)
        best = scored[0]

        # 同时返回 top3 供创作者选择，以及多封面 A/B 测试用
        return {
            'primary': {'timestamp_ms': best['timestamp_ms'],
                        'score': best['score']},
            'alternatives': [{'timestamp_ms': s['timestamp_ms'],
                              'score': s['score']} for s in scored[1:4]],
        }

    def _score(self, frame):
        """加权综合评分"""
        feats = {
            'sharpness': self._sharpness(frame),
            'face_quality': self._face_quality(frame),
            'colorfulness': self._colorfulness(frame),
            'brightness_fit': self._brightness_fit(frame),
            'composition': self._composition(frame),
            'text_penalty': self._text_area_ratio(frame),
            'motion_blur_penalty': self._motion_blur(frame),
        }
        return sum(self.WEIGHTS[k] * v for k, v in feats.items())

    def _sharpness(self, frame):
        """清晰度：拉普拉斯方差（经典指标）"""
        import numpy as np
        gray = frame.mean(axis=2) if frame.ndim == 3 else frame
        # 简化的拉普拉斯算子
        lap = (gray[:-2, 1:-1] + gray[2:, 1:-1] +
               gray[1:-1, :-2] + gray[1:-1, 2:] - 4 * gray[1:-1, 1:-1])
        return min(1.0, float(lap.var()) / 800.0)

    def _face_quality(self, frame):
        """人脸质量：面积占比 × 正面度。无人脸返回 0"""
        faces = self.face.detect(frame)
        if not faces:
            return 0.0
        h, w = frame.shape[:2]
        best = 0.0
        for f in faces:
            area_ratio = (f['w'] * f['h']) / float(h * w)
            # 面积占比 8%-35% 最佳（太小看不清，太大压迫感）
            area_score = 1.0 - abs(area_ratio - 0.18) / 0.18
            area_score = max(0.0, min(1.0, area_score))
            best = max(best, area_score * f.get('frontality', 1.0))
        return best

    def _colorfulness(self, frame):
        """色彩丰富度：RGB 通道差的标准差"""
        import numpy as np
        if frame.ndim < 3:
            return 0.0
        r, g, b = frame[:, :, 0].astype(float), frame[:, :, 1].astype(float), frame[:, :, 2].astype(float)
        rg, yb = r - g, 0.5 * (r + g) - b
        val = float(np.sqrt(rg.std() ** 2 + yb.std() ** 2)
                    + 0.3 * np.sqrt(rg.mean() ** 2 + yb.mean() ** 2))
        return min(1.0, val / 90.0)

    def _brightness_fit(self, frame):
        """亮度适中度：均值在 95-165 最佳"""
        mean = float(frame.mean())
        return max(0.0, 1.0 - abs(mean - 130.0) / 130.0)

    def _composition(self, frame):
        """构图：主体（最高梯度区域）是否落在三分线附近"""
        import numpy as np
        gray = frame.mean(axis=2) if frame.ndim == 3 else frame
        gy, gx = np.gradient(gray)
        mag = np.sqrt(gx ** 2 + gy ** 2)
        h, w = mag.shape
        cy, cx = np.unravel_index(int(mag.argmax()), mag.shape)
        # 三分点位置
        thirds_y = [h / 3, 2 * h / 3]
        thirds_x = [w / 3, 2 * w / 3]
        dy = min(abs(cy - t) for t in thirds_y) / h
        dx = min(abs(cx - t) for t in thirds_x) / w
        return max(0.0, 1.0 - (dy + dx))

    def _text_area_ratio(self, frame):
        """帧内文字占比——已有大量文字的帧作为封面与标题信息重复"""
        import numpy as np
        gray = frame.mean(axis=2) if frame.ndim == 3 else frame
        gy, gx = np.gradient(gray)
        mag = np.sqrt(gx ** 2 + gy ** 2)
        # 文字区域特征：高梯度且高频
        return min(1.0, float((mag > 60).mean()) * 3.0)

    def _motion_blur(self, frame):
        """运动模糊：方向性梯度不均衡度"""
        import numpy as np
        gray = frame.mean(axis=2) if frame.ndim == 3 else frame
        gy, gx = np.gradient(gray)
        hx, hy = float(np.abs(gx).mean()), float(np.abs(gy).mean())
        if max(hx, hy) < 1e-6:
            return 1.0
        # 一个方向梯度远大于另一方向 → 定向模糊
        ratio = min(hx, hy) / max(hx, hy)
        return max(0.0, 1.0 - ratio)
```

### 数据库设计

```sql
-- 转码任务表：调度器的持久化状态，支持崩溃恢复与 SLA 分析
CREATE TABLE transcode_task (
    task_id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    content_id       BIGINT NOT NULL,
    asset_id         BIGINT NOT NULL,
    task_type        VARCHAR(16) NOT NULL COMMENT 'probe/proxy/main/sharded/jit/promote/concat',
    priority         TINYINT NOT NULL DEFAULT 4 COMMENT '0-代理片,1-头部,2-短视频,3-晋升,4-长视频,5-回填',
    tier_level       TINYINT NOT NULL DEFAULT 3 COMMENT '1-头部,2-常规,3-长尾',
    ladder_json      JSON COMMENT '[{"height":720,"bitrate":2000,"codec":"h264"}]',
    probe_features   JSON COMMENT '{"spatial_complexity":0.42,"temporal_complexity":0.31}',
    shard_total      SMALLINT NOT NULL DEFAULT 0 COMMENT '0=不分片',
    shard_done       SMALLINT NOT NULL DEFAULT 0,
    worker_id        VARCHAR(64) NOT NULL DEFAULT '' COMMENT '执行节点',
    hardware         VARCHAR(16) NOT NULL DEFAULT '' COMMENT 'gpu/cpu/cloud',
    status           TINYINT NOT NULL DEFAULT 0 COMMENT '0-待执行,1-运行中,2-成功,3-失败,4-已抢占,5-质检失败',
    retry_count      TINYINT NOT NULL DEFAULT 0,
    fail_reason      VARCHAR(512) NOT NULL DEFAULT '',
    est_cost_sec     INT NOT NULL DEFAULT 0 COMMENT '预估耗时（调度与超时判定用）',
    actual_cost_sec  INT NOT NULL DEFAULT 0,
    enqueued_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    started_at       DATETIME DEFAULT NULL,
    finished_at      DATETIME DEFAULT NULL,
    heartbeat_at     DATETIME DEFAULT NULL COMMENT '心跳，用于识别僵死任务',

    INDEX idx_status_priority (status, priority, enqueued_at),
    INDEX idx_content (content_id, task_type),
    INDEX idx_heartbeat (status, heartbeat_at),
    INDEX idx_worker (worker_id, status),
    INDEX idx_sla (task_type, actual_cost_sec DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='转码任务表';

-- 分片子任务表：分片转码的最小执行单元
CREATE TABLE transcode_shard (
    shard_id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    task_id          BIGINT NOT NULL,
    asset_id         BIGINT NOT NULL,
    rung_key         VARCHAR(32) NOT NULL COMMENT '档位标识，如 "1080p_h265"',
    shard_index      SMALLINT NOT NULL COMMENT '分片序号，拼接时按此排序',
    start_ts_ms      INT NOT NULL COMMENT '起始时间戳（必须为关键帧位置）',
    end_ts_ms        INT NOT NULL DEFAULT 0 COMMENT '0=到结尾',
    output_key       VARCHAR(512) NOT NULL DEFAULT '',
    status           TINYINT NOT NULL DEFAULT 0 COMMENT '0-待执行,1-运行中,2-成功,3-失败',
    worker_id        VARCHAR(64) NOT NULL DEFAULT '',
    retry_count      TINYINT NOT NULL DEFAULT 0,
    started_at       DATETIME DEFAULT NULL,
    finished_at      DATETIME DEFAULT NULL,

    UNIQUE KEY uk_task_rung_shard (task_id, rung_key, shard_index),
    INDEX idx_task_status (task_id, status),
    INDEX idx_status_worker (status, worker_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='转码分片子任务表';

-- 质检记录表：每个产物的质检结论，用于追溯与模型改进
CREATE TABLE transcode_qc_record (
    qc_id            BIGINT AUTO_INCREMENT PRIMARY KEY,
    asset_id         BIGINT NOT NULL,
    variant_id       BIGINT NOT NULL,
    rung_key         VARCHAR(32) NOT NULL,
    passed           TINYINT NOT NULL DEFAULT 0 COMMENT '0-未通过,1-通过',
    vmaf_score       DECIMAL(5,2) DEFAULT NULL,
    black_frame_ratio DECIMAL(5,4) DEFAULT NULL,
    silence_ratio    DECIMAL(5,4) DEFAULT NULL,
    av_sync_offset_ms SMALLINT NOT NULL DEFAULT 0,
    duration_diff_ms INT NOT NULL DEFAULT 0,
    failures_json    JSON COMMENT '["LOW_VMAF(84.2<88)","AV_DESYNC(180ms)"]',
    qc_cost_ms       INT NOT NULL DEFAULT 0 COMMENT '质检耗时（成本监控）',
    created_at       DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_asset (asset_id, rung_key),
    INDEX idx_failed (passed, created_at),
    INDEX idx_vmaf (rung_key, vmaf_score)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='转码质检记录表';

-- 热度晋升记录表：追踪二次转码的触发与收益
CREATE TABLE transcode_promotion (
    promo_id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    content_id       BIGINT NOT NULL,
    asset_id         BIGINT NOT NULL,
    from_tier        TINYINT NOT NULL COMMENT '晋升前 tier',
    to_tier          TINYINT NOT NULL,
    trigger_reason   VARCHAR(32) NOT NULL COMMENT 'play_count/jit_demand/manual/editorial',
    play_count_at_promo BIGINT NOT NULL DEFAULT 0,
    added_codecs     VARCHAR(64) NOT NULL DEFAULT '' COMMENT '新增编码，如 "h265,av1"',
    added_rungs      VARCHAR(128) NOT NULL DEFAULT '' COMMENT '新增档位',
    extra_encode_cost DECIMAL(10,4) NOT NULL DEFAULT 0 COMMENT '额外编码成本（元）',
    est_bandwidth_saving DECIMAL(12,2) NOT NULL DEFAULT 0 COMMENT '预估带宽节省（元/月）',
    created_at       DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    UNIQUE KEY uk_content_tier (content_id, to_tier),
    INDEX idx_trigger (trigger_reason, created_at),
    INDEX idx_roi (est_bandwidth_saving DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='热度晋升二次转码记录表';
```

`transcode_promotion` 表的 `extra_encode_cost` 与 `est_bandwidth_saving` 两列让晋升策略的 ROI 可以直接用 SQL 验证——**任何成本敏感的策略都必须自带收益度量，否则无法判断它是否真的有效**。

## 常见陷阱（深度分析）

### 陷阱 1：用固定 ABR 阶梯套所有内容

**后果：** 静态画面内容（新闻播报、访谈）被分配了远超所需的码率，高动态内容（体育、游戏）画质不足。两个方向同时错。

**具体数据：** 实测加权平均浪费 22% 码率。按 ¥2.3 亿/月带宽计，**固定阶梯每月白烧 ¥5,069 万**；同时约 11% 的高动态内容 VMAF 低于 88 的画质底线，产生画质投诉。

**解决方案：** per-title 阶梯 + 凸包优化。但不要用朴素的全网格试编码（成本 30 倍），而是用 2 秒探针提取复杂度特征 + 离线训练的预测模型。收益比 364:1（探针成本 ¥12 万/月，带宽节省 ¥4,370 万/月）。

### 陷阱 2：在转码时就决定编码格式与档位数量

**后果：** 转码发生在发布前，此时播放量未知。无论怎么预测都会双向错——给冷内容做了贵编码（浪费算力），给热内容做了廉价编码（浪费带宽）。

**具体数据：** 播放量预测模型 AUC 约 0.78，双向错误导致相比"事后按实际热度晋升"每月多花 ¥580 万。

**解决方案：** **把一次不可逆决策改成"先廉价决策，观察后再补"。** 发布时一律 Tier 3（H.264 两档，最便宜）；6 小时后按真实播放量晋升 Tier 2；24 小时后晋升 Tier 1。等 6 小时拿到的真实数据准确率是 100%，远胜任何预测模型。

### 陷阱 3：长视频不分片，用 FIFO 排队

**后果：** 一条 90 分钟 4K 视频占用一张 GPU 卡 90 分钟，后面 1000 条短视频全部等待。短视频 P95 从 44 秒涨到 18 分钟，完全击穿 2 分钟的 SLA。

**解决方案：** 按关键帧边界切成 3 分钟片段并行转码，墙钟从 90 分钟降到 3.7 分钟（24 倍）。分片同时带来三个额外收益：失败只需重做 1 片而非全片、可被高优任务抢占、调度粒度细化。

**但分片有五个必须遵守的前提**（违反任一都会产生可见瑕疵）：切点必须在 IDR 关键帧、各片编码参数完全一致、码率控制必须用固定 CRF 而非 ABR、音频不分片（整轨单独转）、拼接必须用 stream copy 不重编码。

### 陷阱 4：只做优先级调度，不做老化提权

**后果：** 长尾内容（低优先级）在持续的高优任务流下永远拿不到资源。实测纯优先级调度下饿死率 8.2%——即 8.2% 的投稿最终转码超时失败，创作者体验极差且平台损失内容供给。

**解决方案：** 等待超过 5 分钟每档提升一级优先级。老化提权把饿死率从 8.2% 压到 0.02%，代价是短视频 P95 从 48 秒微升到 44 秒（实际因整体吞吐提升反而更快）。

### 陷阱 5：不做转码产物质检

**后果：** 转码可能产出"格式合法但内容错误"的产物（花屏、黑屏、音画不同步、时长截断），FFmpeg 返回码 0 不报错。这些产物直接上线，用户看到黑屏或花屏——**比"视频还在处理中"糟糕得多**，因为用户会认为内容本身有问题并投诉创作者。

**真实案例：** 某平台一次 GPU 驱动升级引入编码 bug，约 0.3% 的产物出现规律性块状花屏，持续 6 小时未被发现，影响约 4.2 万条内容、约 1900 万次播放。

**解决方案：** 七项质检闸门，按成本从低到高排序执行（前面失败就不做后面的）。总成本仅占转码成本 3.86%，其中 VMAF 占 88%——所以 VMAF 必须放最后。抽样点须非均匀分布（在场景切换、高运动、码率突变处加密采样），把局部劣化漏检率从 1.2% 降到 0.09%。

### 陷阱 6：预转码全部档位

**后果：** 90% 的内容 7 日播放 < 100 次，给它们转 6 档等于为几乎不会发生的播放预付算力与存储。年产生 350 PB 转码副本，其中 298 PB（85%）7 日播放为 0。

**解决方案：** 2 档预转 + JIT 按需 + 邻档兜底。总成本从 ¥1,944 万/月降到 ¥435 万/月（省 78%），代价是 3.2% 的请求首次拿到降级档位。关键是**绝不让用户等待转码**——立即返回邻近可用档位，JIT 完成后无缝切换。

### 陷阱 7：JIT 转码不做请求去重与热度联动

**后果：** 一条沉底半年的视频突然上热搜，瞬间 50 万请求同一个未转码档位 → 提交 50 万个重复 JIT 任务 → 队列爆炸。或反过来，为单个用户的一次请求就触发转码，算力被无意义消耗。

**解决方案：** 三重防护——SET NX 保证同档位只有一个 JIT 任务在飞；累计请求数达到 3 次才触发（过滤单次请求）；**JIT 需求计数本身作为热度信号**，60 秒内超过 1000 则直接提到最高优先级并触发 Tier 晋升。最后一条尤其重要：JIT 需求是比播放量统计更实时的热度信号。

## 异常场景完整演练

### 场景 1：GPU 驱动升级引入编码 bug 导致批量花屏产物

**场景描述：** 运维为修复一个显存泄漏问题，对 420 张 GPU 卡分批升级 NVIDIA 驱动。新驱动的 NVENC 在特定参数组合（H.265 + B 帧数 > 5 + 分辨率非 16 对齐）下产生规律性块状花屏。约 0.3% 的转码产物受影响（符合该参数组合的内容比例），持续 6 小时才被用户举报发现，累计影响 4.2 万条内容、约 1900 万次播放。

**根因分析：** 三重失效叠加。第一，质检的花屏检测阈值设置过松（要求超过 1/4 抽样帧疑似才判定），而本次花屏是"规律但轻微"的块效应，单帧疑似度未达阈值。第二，驱动升级只做了"能否正常编码"的功能验证，未做画质回归（VMAF 对比）。第三，灰度策略按机器分批而非按内容分批——受影响的是"符合特定参数组合的内容"，而这类内容会被调度到任意机器，所以灰度期间就已经在生产中产生错误产物，但占比太低未触发告警。

**检测机制：**

```python
class EncoderRegressionDetector:
    """编码器回归检测——识别"某批 worker/驱动版本产出的画质系统性偏低"

    核心思路：不看单条产物的绝对画质（内容本身有差异），
    而看按 (worker版本, 编码参数) 分组的画质分布是否偏移。
    """

    VMAF_DROP_ALERT = 1.5        # 分组 VMAF 均值下降 1.5 分即告警
    MIN_SAMPLES = 200            # 每组至少 200 个样本才做判断
    BASELINE_WINDOW_DAYS = 7

    def __init__(self, mysql_client, config):
        self.mysql = mysql_client
        self.alert = config['alert_service']

    def check_by_dimension(self):
        """按多个维度分组对比画质，定位问题维度"""
        dimensions = [
            ('worker_version', 'hardware'),
            ('driver_version', 'codec'),
            ('codec', 'preset'),
        ]
        findings = []
        for dims in dimensions:
            findings.extend(self._compare_group(dims))
        return findings

    def _compare_group(self, dims):
        """对比某维度组合下，近 1 小时 vs 基线 7 天的 VMAF 分布"""
        dim_cols = ", ".join(f"t.{d}" for d in dims)
        rows = self.mysql.query(f"""
            SELECT {dim_cols},
                   AVG(CASE WHEN q.created_at > DATE_SUB(NOW(), INTERVAL 1 HOUR)
                            THEN q.vmaf_score END) AS recent_vmaf,
                   COUNT(CASE WHEN q.created_at > DATE_SUB(NOW(), INTERVAL 1 HOUR)
                              THEN 1 END) AS recent_n,
                   AVG(CASE WHEN q.created_at <= DATE_SUB(NOW(), INTERVAL 1 HOUR)
                            THEN q.vmaf_score END) AS baseline_vmaf,
                   COUNT(CASE WHEN q.created_at <= DATE_SUB(NOW(), INTERVAL 1 HOUR)
                              THEN 1 END) AS baseline_n
            FROM transcode_qc_record q
            JOIN transcode_task t ON t.asset_id = q.asset_id
            WHERE q.created_at > DATE_SUB(NOW(), INTERVAL {self.BASELINE_WINDOW_DAYS} DAY)
              AND q.vmaf_score IS NOT NULL
            GROUP BY {dim_cols}
            HAVING recent_n >= {self.MIN_SAMPLES}
               AND baseline_n >= {self.MIN_SAMPLES}
        """)

        findings = []
        for r in rows:
            if r['recent_vmaf'] is None or r['baseline_vmaf'] is None:
                continue
            drop = r['baseline_vmaf'] - r['recent_vmaf']
            if drop >= self.VMAF_DROP_ALERT:
                finding = {
                    'dimensions': {d: r[d] for d in dims},
                    'vmaf_drop': round(drop, 2),
                    'recent_vmaf': round(r['recent_vmaf'], 2),
                    'baseline_vmaf': round(r['baseline_vmaf'], 2),
                    'affected_count': r['recent_n'],
                }
                findings.append(finding)
                self.alert.fire("encoder_quality_regression",
                                severity="P0", detail=finding)
        return findings
```

**解决方案：**

```python
class EncoderRolloutGuard:
    """编码器/驱动变更的灰度守卫——按"参数组合覆盖度"灰度而非按机器"""

    # 必须覆盖的参数组合矩阵（历史上出问题的组合优先）
    CANARY_MATRIX = [
        {'codec': 'h264', 'preset': 'veryfast', 'height': 720,  'bframes': 3},
        {'codec': 'h264', 'preset': 'medium',   'height': 1080, 'bframes': 8},
        {'codec': 'h265', 'preset': 'medium',   'height': 1080, 'bframes': 8},
        {'codec': 'h265', 'preset': 'slow',     'height': 2160, 'bframes': 5},
        {'codec': 'av1',  'preset': 'medium',   'height': 1080, 'bframes': 0},
        # 非标准分辨率（本次事故的触发条件）
        {'codec': 'h265', 'preset': 'medium',   'height': 1082, 'bframes': 6},
    ]
    GOLDEN_SET_SIZE = 40         # 黄金测试集：涵盖各类内容特征的固定素材
    VMAF_REGRESSION_LIMIT = 0.8  # 相比旧版本 VMAF 下降超 0.8 分即阻断

    def __init__(self, ffmpeg_client, mysql_client, config):
        self.ffmpeg = ffmpeg_client
        self.mysql = mysql_client
        self.golden_set = config['golden_set_urls']

    def validate_before_rollout(self, new_worker_image):
        """
        变更上线前的强制验证：黄金测试集 × 参数矩阵 全跑一遍，
        与旧版本逐项对比 VMAF。任一组合回归超限则阻断上线。
        """
        results, blockers = [], []

        for params in self.CANARY_MATRIX:
            for src in self.golden_set[:self.GOLDEN_SET_SIZE]:
                new_out = self.ffmpeg.encode(
                    src, image=new_worker_image, **params)
                old_vmaf = self._get_baseline_vmaf(src, params)
                new_vmaf = self.ffmpeg.vmaf(src, new_out, sample_points=24)

                delta = new_vmaf - old_vmaf
                # 除 VMAF 外，还必须过一遍完整质检（抓花屏等 VMAF 不敏感的问题）
                gate = TranscodeQualityGate(self.ffmpeg, {'alert_service': None})
                passed, failures = gate.check(
                    src, new_out, params['height'],
                    self._duration_of(src))

                item = {'params': params, 'src': src,
                        'vmaf_delta': round(delta, 2),
                        'qc_passed': passed, 'qc_failures': failures}
                results.append(item)

                if delta < -self.VMAF_REGRESSION_LIMIT or not passed:
                    blockers.append(item)

        return {'passed': len(blockers) == 0,
                'blockers': blockers,
                'total_cases': len(results)}

    def rollout(self, new_worker_image):
        """验证通过后的分批上线，每批后再做一次线上回归检测"""
        validation = self.validate_before_rollout(new_worker_image)
        if not validation['passed']:
            raise RuntimeError(
                f"编码器变更被阻断：{len(validation['blockers'])} 个参数组合回归\n" +
                "\n".join(f"  · {b['params']}: VMAF{b['vmaf_delta']:+.2f} "
                          f"QC={b['qc_failures']}" for b in validation['blockers'][:5]))

        for percent in [1, 5, 20, 50, 100]:
            self._set_rollout_percent(new_worker_image, percent)
            self._wait_for_samples(min_samples=500)
            detector = EncoderRegressionDetector(
                self.mysql, {'alert_service': None})
            findings = detector.check_by_dimension()
            if findings:
                self._rollback(new_worker_image)
                raise RuntimeError(
                    f"灰度 {percent}% 时检测到画质回归，已回滚：{findings[:3]}")
```

**三项加固措施：**

| 措施 | 具体做法 | 拦住本次事故的哪一环 |
|------|---------|-------------------|
| 变更前黄金集验证 | 40 个素材 × 6 组参数矩阵，逐项 VMAF 对比 + 完整质检 | 上线前就能发现（矩阵含非 16 对齐分辨率） |
| 收紧花屏检测阈值 | 疑似帧比例阈值从 25% 降到 12%，并加入"块效应强度"连续指标而非布尔判定 | 质检阶段拦住 |
| 分组画质回归监控 | 按 (driver_version, codec, preset) 分组对比 VMAF 分布 | 灰度 1% 时就告警 |

**关键设计：编码器变更的灰度必须按"参数组合覆盖度"而非"机器百分比"。** 按机器灰度时，问题只在特定参数组合下出现，而该组合的内容会被调度到全部机器——灰度 1% 机器并不等于只有 1% 的风险暴露。**灰度维度必须与故障维度对齐，否则灰度形同虚设。**

### 场景 2：分片转码拼接处画质跳变

**场景描述：** 一条 42 分钟长视频分 14 片并行转码后拼接，用户反馈"每隔三分钟画面会突然变清晰或变模糊一下"。检查发现各分片使用了 ABR（平均码率）模式，每片独立做码率统计与分配——内容复杂的片段被压到目标码率导致画质低，内容简单的片段码率有余导致画质高。拼接后形成周期性画质波动。

**根因分析：** ABR 码率控制的本质是"在整个编码单元内做码率分配"。分片后每片成为独立编码单元，各自在 3 分钟内做分配，失去了全片视角。这不是 bug，是分片与 ABR 的**语义冲突**。

**检测机制：**

```python
class ShardConsistencyChecker:
    """分片拼接一致性检查——检测拼接处的画质/码率跳变"""

    BITRATE_JUMP_LIMIT = 0.25    # 相邻分片码率差异超 25% 告警
    VMAF_JUMP_LIMIT = 3.0        # 相邻分片 VMAF 差异超 3 分告警

    def __init__(self, ffmpeg_client, mysql_client, config):
        self.ffmpeg = ffmpeg_client
        self.mysql = mysql_client
        self.alert = config['alert_service']

    def check(self, task_id, rung_key):
        """检查一个档位的所有分片是否一致"""
        shards = self.mysql.query("""
            SELECT shard_index, output_key, start_ts_ms, end_ts_ms
            FROM transcode_shard
            WHERE task_id=%s AND rung_key=%s AND status=2
            ORDER BY shard_index
        """, (task_id, rung_key))

        if len(shards) < 2:
            return {'consistent': True}

        # 1. 逐片测实际码率与 VMAF
        stats = []
        for s in shards:
            meta = self.ffmpeg.probe(s['output_key'])
            stats.append({
                'index': s['shard_index'],
                'bitrate': meta['bitrate'],
                'vmaf': self.ffmpeg.vmaf_quick(s['output_key'], sample_points=4),
            })

        # 2. 检查相邻分片的跳变
        issues = []
        for a, b in zip(stats[:-1], stats[1:]):
            br_jump = abs(b['bitrate'] - a['bitrate']) / max(a['bitrate'], 1)
            vmaf_jump = abs(b['vmaf'] - a['vmaf'])
            if br_jump > self.BITRATE_JUMP_LIMIT:
                issues.append({'boundary': f"{a['index']}→{b['index']}",
                               'type': 'BITRATE_JUMP',
                               'value': round(br_jump, 3)})
            if vmaf_jump > self.VMAF_JUMP_LIMIT:
                issues.append({'boundary': f"{a['index']}→{b['index']}",
                               'type': 'VMAF_JUMP',
                               'value': round(vmaf_jump, 2)})

        if issues:
            self.alert.fire("shard_inconsistency", severity="P1", detail={
                'task_id': task_id, 'rung_key': rung_key, 'issues': issues})

        return {'consistent': not issues, 'issues': issues, 'stats': stats}
```

**解决方案：**

| 问题 | 错误做法 | 正确做法 |
|------|---------|---------|
| 码率控制模式 | 各片独立 ABR（每片自己统计） | **固定 CRF**（质量恒定，码率随内容浮动）或 **两遍编码 + 全片统一码率曲线** |
| 编码参数 | 各片按自己的内容特征调参 | **全片统一参数**（探针在全片上做，参数下发给所有片） |
| GOP 结构 | 各片独立决定关键帧 | **统一 GOP 长度**，且分片边界必须与 GOP 边界重合 |
| 音频 | 分片后拼接 | **整轨单独转码**，不分片 |
| 拼接方式 | 重编码拼接 | **stream copy**（`-c copy`），零画质损失 |

**关键设计：分片转码必须用固定 CRF 而非 ABR。** CRF 的语义是"质量恒定"，天然可分片——每片各自维持相同质量，拼接后画质连续。ABR 的语义是"平均码率恒定"，与分片语义冲突。这是一个"编码参数的语义必须与并行化方式兼容"的例子。

**代价：** CRF 模式下最终文件大小不可预测（复杂内容文件更大）。对需要严格控制码率上限的场景（如为窄带用户准备的低码率档），用 **capped CRF**（CRF + maxrate 上限）兼顾两者。

### 场景 3：转码 worker 僵死导致任务永久挂起

**场景描述：** 部分转码 worker 因 FFmpeg 进程在特定损坏源片上进入死循环（解码器等待永不到来的数据），进程存活但无进展。心跳基于"进程是否存活"因此正常上报，调度器认为任务在正常执行。约 3,200 个任务挂起超过 2 小时，对应内容永久停留在 `processing` 状态。

**根因分析：** 心跳设计错误——检测的是"进程活着"而非"任务有进展"。这是活性检测（liveness）与进展检测（progress）的混淆。

**解决方案：**

```python
class ProgressAwareHeartbeat:
    """基于进展的心跳——检测"任务是否在推进"而非"进程是否存活" """

    STALL_TIMEOUT_SEC = 120       # 进展停滞 2 分钟即判定僵死
    # 预估耗时的容忍倍数：超过预估 3 倍则强制超时
    OVERRUN_FACTOR = 3.0

    def __init__(self, redis_client, mysql_client, config):
        self.redis = redis_client
        self.mysql = mysql_client
        self.alert = config['alert_service']

    def report(self, task_id, worker_id, progress):
        """
        worker 上报进展。progress 必须是单调递增的实际进度
        （已编码帧数 / 已输出字节数），而非"我还活着"。
        """
        key = f"transcode:progress:{task_id}"
        prev = self.redis.hgetall(key)
        prev_frames = int(prev.get(b'frames', 0))

        # 关键：进度未推进则不刷新时间戳
        if progress['frames_encoded'] <= prev_frames:
            return {'ok': True, 'stalled': True}

        import time
        self.redis.hset(key, mapping={
            'frames': progress['frames_encoded'],
            'bytes': progress['bytes_written'],
            'worker': worker_id,
            'last_progress_at': int(time.time()),
        })
        self.redis.expire(key, 7200)
        self.mysql.execute(
            "UPDATE transcode_task SET heartbeat_at=NOW() WHERE task_id=%s",
            (task_id,))
        return {'ok': True, 'stalled': False}

    def sweep_stalled(self):
        """
        定期扫描僵死任务。两个判据：
          1. 进展停滞超过 STALL_TIMEOUT_SEC
          2. 总耗时超过预估的 OVERRUN_FACTOR 倍
        """
        import time
        now = int(time.time())
        killed = []

        running = self.mysql.query("""
            SELECT task_id, worker_id, est_cost_sec, started_at,
                   TIMESTAMPDIFF(SECOND, started_at, NOW()) AS elapsed
            FROM transcode_task
            WHERE status = 1
        """)

        for t in running:
            key = f"transcode:progress:{t['task_id']}"
            last = self.redis.hget(key, 'last_progress_at')
            stalled_sec = now - int(last) if last else t['elapsed']

            overrun = (t['est_cost_sec'] > 0
                       and t['elapsed'] > t['est_cost_sec'] * self.OVERRUN_FACTOR)

            if stalled_sec > self.STALL_TIMEOUT_SEC or overrun:
                reason = ('PROGRESS_STALLED' if stalled_sec > self.STALL_TIMEOUT_SEC
                          else 'TIME_OVERRUN')
                self._kill_and_requeue(t, reason)
                killed.append({'task_id': t['task_id'], 'reason': reason,
                               'stalled_sec': stalled_sec})

        if killed:
            self.alert.fire("transcode_stalled_tasks", severity="P1",
                            detail={'count': len(killed), 'samples': killed[:5]})
        return killed

    def _kill_and_requeue(self, task, reason):
        """杀死僵死任务并重新入队（换一台 worker，换 CPU 软编兜底）"""
        self.mysql.execute("""
            UPDATE transcode_task
            SET status=3, fail_reason=%s, retry_count=retry_count+1,
                finished_at=NOW()
            WHERE task_id=%s AND status=1
        """, (reason, task['task_id']))

        row = self.mysql.query(
            "SELECT retry_count, content_id, asset_id FROM transcode_task "
            "WHERE task_id=%s", (task['task_id'],))

        if row['retry_count'] >= 3:
            # 重试耗尽：标记永久失败，通知创作者（不可静默失败）
            self.mysql.execute("""
                UPDATE transcode_task SET status=3,
                    fail_reason=CONCAT(fail_reason, '|RETRY_EXHAUSTED')
                WHERE task_id=%s
            """, (task['task_id'],))
            self.redis.lpush("notify:transcode_failed", self._json({
                'content_id': row['content_id'],
                'message': '视频处理失败，可能是文件格式异常，请尝试重新上传',
            }))
        else:
            # 重试：改用 CPU 软编（更慢但更鲁棒，能处理异常源片）
            self.redis.lpush("transcode:requeue", self._json({
                'content_id': row['content_id'],
                'asset_id': row['asset_id'],
                'force_hardware': 'cpu',
                'force_preset': 'medium',
            }))

    def _json(self, obj):
        import json
        return json.dumps(obj, ensure_ascii=False)
```

**关键设计：心跳必须检测"进展"而非"存活"。** `frames_encoded` 单调递增才刷新时间戳，进度不动则视为僵死——这是活性检测的正确形态。任何"我还活着"式的心跳都无法发现死循环、死锁、无进展等最常见的僵死形态。

## 性能与成本分析

### 转码集群资源与成本

| 组件 | 规格 | 数量 | 月成本 |
|------|------|------|-------|
| GPU 转码集群（主力） | A10 24G，12 路并发/卡 | 420 卡 | ¥210 万 |
| CPU 转码集群（4K + 兜底） | 64c128G | 180 台 | ¥100 万 |
| 探针集群（轻量） | 16c32G | 24 台 | ¥4.8 万 |
| 质检集群（VMAF 计算） | 32c64G | 36 台 | ¥7.2 万 |
| 云转码溢出 | 按量 | 峰值 3% 任务 | ¥30 万 |
| 转码中间产物存储（临时） | 分片与拼接暂存 | 180 TB | ¥2.2 万 |
| **合计** | | | **¥354 万** |

超出 ¥310 万预算 ¥44 万，主要是探针与质检的新增成本 ¥12 万，以及云溢出 ¥30 万。但这 ¥44 万换回的是：探针带来 ¥4,370 万/月带宽节省，质检把错误产物率从 0.3% 压到 0.001%，云溢出消除 45% 的备容冗余。**净收益为正约 ¥4,300 万/月。**

### 各阶段耗时分解

```
短视频（38 秒，1080P，Tier 3 两档）：
  ┌──────────────────────────────────────────────────┐
  │ 阶段0 探针（抽24帧+特征）        ≈  2.0 s        │
  │ 阶段1 代理片（360P极速）         ≈  4.5 s        │ ← 审核从此刻可开始
  │ 阶段2 主转码（2档并行，GPU）     ≈ 11.4 s        │
  │ 阶段3 质检（7项，含抽样VMAF）    ≈  3.2 s        │
  │ 阶段4 CMAF封装 + 双清单          ≈  2.1 s        │
  │ 并行支线（封面/雪碧图/指纹）      ≈  0 s（重叠）  │
  ├──────────────────────────────────────────────────┤
  │ 合计（墙钟）                     ≈ 23.2 s        │
  │ 排队等待（P95）                  ≈ 20.8 s        │
  │ 端到端 P95                       ≈ 44.0 s        │ 预算 120s ✓
  └──────────────────────────────────────────────────┘

长视频（42 分钟，1080P，Tier 3 两档，分 14 片）：
  ┌──────────────────────────────────────────────────┐
  │ 阶段0 探针                       ≈  2.8 s        │
  │ 阶段1 代理片（极速档，整片）      ≈ 38.0 s        │ ← 审核从此刻可开始
  │ 阶段2 主转码（14片×2档=28子任务） ≈ 96.0 s       │
  │        单片3分钟×0.3实时倍率=54s，                │
  │        28子任务/12并发槽=2.3轮 → 124s            │
  │ 阶段2b 拼接（stream copy）        ≈ 22.0 s        │
  │ 阶段3 质检                       ≈ 18.0 s        │
  │ 阶段4 封装                       ≈  9.0 s        │
  ├──────────────────────────────────────────────────┤
  │ 合计（墙钟）                     ≈ 3.6 min       │
  │ 排队等待（P95）                  ≈ 0.3 min       │
  │ 端到端 P95                       ≈ 3.9 min       │ 预算 14min ✓
  └──────────────────────────────────────────────────┘
```

### 容量余量

```
资源              │ 峰值负载    │ 瓶颈容量    │ 余量   │ 扩容速度
─────────────────┼───────────┼───────────┼───────┼──────────
GPU 卡并发槽      │ 78%        │ 100%       │ 28%   │ 5-8 min
CPU 集群          │ 54%        │ 85%        │ 57%   │ 90 s
探针集群          │ 41%        │ 85%        │ 107%  │ 60 s
质检集群          │ 63%        │ 85%        │ 35%   │ 90 s
中间产物存储       │ 47%        │ 85%        │ 81%   │ 小时级
云溢出配额        │ 3%         │ 25%（成本约束）│ 733% │ 秒级
```

**最脆弱环节是 GPU 卡并发槽（余量 28%，扩容 5-8 分钟）。** 应对是三级降级（见 P00 场景 1）+ 云溢出，两者叠加可在 90 秒内获得等效 3.6 倍吞吐。

### 成本优化项汇总

| 优化项 | 优化前 | 优化后 | 节省 | 手段 |
|-------|-------|-------|------|------|
| 带宽（阶梯优化 -19%） | ¥23,040 万 | ¥18,662 万 | ¥4,378 万 | per-title 凸包阶梯 |
| 带宽（分层编码，承接上一步） | ¥18,662 万 | ¥12,890 万 | ¥5,772 万 | 热度晋升 H.265/AV1 |
| 转码算力 | ¥1,120 万 | ¥310 万 | ¥810 万 | GPU 硬编 + Tier 3 廉价预设 |
| 转码副本存储 | ¥824 万 | ¥125 万 | ¥699 万 | 2 档预转 + JIT + 副本回收 |
| 备容冗余 | ¥94 万 | ¥30 万 | ¥64 万 | 云溢出替代自建冗余 |
| 新增成本（探针 + 质检） | — | ¥44 万 | -¥44 万 | — |
| **合计（不重复计带宽）** | **¥25,078 万** | **¥13,399 万** | **¥11,679 万** | **降 46.6%** |

> 合计的计算方式：优化前 = 带宽 23,040 + 转码 1,120 + 副本存储 824 + 备容 94；优化后 = 带宽 12,890 + 转码 310 + 副本存储 125 + 备容 30 + 新增 44。带宽只按"最终值"计一次，不把两步优化的中间值累加。

## 延伸思考

- **探针预测模型的分布漂移**：模型用历史内容训练，但内容形态在变（竖屏化、AI 生成内容增多、HDR 普及）。如何检测模型失效？多久重训一次？置信度低时"保守阶梯上浮 15%"这个兜底值怎么定？
- **热度晋升的时间窗口**：本文选 6 小时/24 小时两个观察点。如果改成 1 小时会怎样（更快拿到编码收益，但预测准确率下降）？最优观察时长如何用数据确定？
- **分片粒度的最优值**：本文用 3 分钟。更小的分片（1 分钟）并行度更高但拼接开销与调度开销上升。最优粒度与集群规模、任务到达率是什么关系？
- **如果 AV1 硬解覆盖率涨到 90%**（3-5 年后可能发生），分层编码策略该怎么改？是否应该把 AV1 下沉到 Tier 2 甚至 Tier 3？
- **质检的 VMAF 抽样点非均匀分布**把漏检率从 1.2% 降到 0.09%。剩下的 0.09% 如何处理？靠用户举报兜底的成本是多少？是否值得为此再投入？
- **本文的成本结构假设 GPU 硬编单价 ¥8/卡时。** 如果专用 VPU 成熟（单价 ¥5/卡时、并发 40 路），转码架构需要怎么调整？切换的一次性成本有多大？

## 转码 worker 完整实现

worker 是执行单元，其设计要点是**幂等、可中断、进展可观测**。

```python
import os
import subprocess
import time


class TranscodeWorker:
    """转码执行单元

    三个设计要点：
      1. 幂等：同一子任务重复执行结果一致，输出路径由任务内容决定性生成
      2. 可中断：收到抢占信号后清理临时文件并优雅退出
      3. 进展可观测：解析 FFmpeg 进度输出，上报单调递增的帧数
    """

    FFMPEG_PROGRESS_INTERVAL_SEC = 2
    TEMP_DIR = "/var/tmp/transcode"

    def __init__(self, scheduler_client, storage_client, heartbeat, config):
        self.scheduler = scheduler_client
        self.storage = storage_client
        self.heartbeat = heartbeat
        self.worker_id = config['worker_id']
        self.worker_type = config['worker_type']       # gpu / cpu
        self.supports_av1 = config.get('supports_av1', False)
        self._preempted = False

    def run_loop(self):
        """主循环：拉任务 → 执行 → 上报"""
        while True:
            task = self.scheduler.acquire({
                'worker_id': self.worker_id,
                'type': self.worker_type,
                'supports_av1': self.supports_av1,
            })
            if task is None:
                time.sleep(1)
                continue

            self._preempted = False
            try:
                output_key = self.execute(task)
                self.scheduler.report_success(task, output_key)
            except PreemptedError:
                self.scheduler.report_preempted(task)
            except Exception as e:
                self.scheduler.report_failure(task, str(e))

    def execute(self, task):
        """执行单个转码子任务"""
        rung = task['rung']
        shard = task.get('shard')

        # 1. 幂等输出路径：由任务内容决定性生成，重跑覆盖同一路径
        output_key = self._output_key(task)

        # 2. 已存在且质检通过 → 直接返回（幂等短路）
        if self.storage.exists(output_key):
            if self._verify_existing(output_key, rung):
                return output_key
            self.storage.delete(output_key)     # 存在但不合格，删掉重做

        local_out = os.path.join(
            self.TEMP_DIR, f"{task['asset_id']}_{self._rung_key(rung)}"
            f"_{shard['shard_index'] if shard else 'full'}.mp4")
        os.makedirs(self.TEMP_DIR, exist_ok=True)

        # 3. 构建 FFmpeg 命令
        cmd = self._build_command(task, local_out)

        # 4. 执行并解析进展
        self._run_with_progress(cmd, task)

        # 5. 上传产物
        self.storage.upload(local_out, output_key)
        os.remove(local_out)
        return output_key

    def _build_command(self, task, local_out):
        """构建 FFmpeg 命令行"""
        rung = task['rung']
        shard = task.get('shard')
        src = task['src_url']

        cmd = ['ffmpeg', '-hide_banner', '-y',
               '-progress', 'pipe:1', '-nostats']

        # 硬件解码（GPU worker）
        if self.worker_type == 'gpu':
            cmd += ['-hwaccel', 'cuda', '-hwaccel_output_format', 'cuda']

        # 分片：用 -ss/-to 精确裁剪（放在 -i 前是快速 seek）
        if shard:
            cmd += ['-ss', str(shard['start_ts'] / 1000.0)]

        cmd += ['-i', src]

        if shard and shard.get('end_ts'):
            cmd += ['-to', str((shard['end_ts'] - shard['start_ts']) / 1000.0)]

        # 滤镜链（窄带高清）
        vf = task.get('filter_chain')
        if vf:
            cmd += ['-vf', vf]
        else:
            cmd += ['-vf', f"scale=-2:{rung['height']}:flags=lanczos"]

        # 编码器选择
        cmd += self._encoder_args(rung, task)

        # 分片必须用固定 CRF（陷阱：ABR 分片会导致拼接处画质跳变）
        if shard:
            cmd += ['-crf', str(task.get('crf', 23))]
            if rung.get('bitrate'):
                # capped CRF：CRF 保质量 + maxrate 控上限
                cmd += ['-maxrate', f"{int(rung['bitrate'] * 1.45)}k",
                        '-bufsize', f"{int(rung['bitrate'] * 2.9)}k"]
        else:
            cmd += ['-b:v', f"{rung['bitrate']}k",
                    '-maxrate', f"{int(rung['bitrate'] * 1.45)}k",
                    '-bufsize', f"{int(rung['bitrate'] * 2.9)}k"]

        # GOP：全片统一（分片时各片必须一致）
        keyint = task.get('keyint', 120)
        cmd += ['-g', str(keyint), '-keyint_min', str(int(keyint / 4))]

        # 音频：分片任务不处理音频（音频整轨单独转，避免拼接爆音）
        if shard:
            cmd += ['-an']
        else:
            cmd += ['-c:a', 'aac', '-b:a', '128k', '-ar', '48000',
                    '-af', 'loudnorm=I=-16:TP=-1.5:LRA=11']   # 响度归一

        cmd += ['-movflags', '+faststart', local_out]
        return cmd

    def _encoder_args(self, rung, task):
        """按编码器与硬件选择参数"""
        codec = rung['codec'].split('+')[0]
        preset = rung.get('preset', 'medium')
        extra = task.get('encoder_params', {})

        if self.worker_type == 'gpu':
            mapping = {'h264': 'h264_nvenc', 'h265': 'hevc_nvenc',
                       'av1': 'av1_nvenc'}
            args = ['-c:v', mapping.get(codec, 'h264_nvenc')]
            # NVENC 的预设命名与 x264 不同
            nv_preset = {'veryfast': 'p2', 'medium': 'p4', 'slow': 'p6'}
            args += ['-preset', nv_preset.get(preset, 'p4'),
                     '-rc', 'vbr', '-tune', 'hq',
                     '-spatial-aq', '1', '-temporal-aq', '1']
            return args

        # CPU 软编
        mapping = {'h264': 'libx264', 'h265': 'libx265', 'av1': 'libsvtav1'}
        args = ['-c:v', mapping.get(codec, 'libx264'), '-preset', preset]
        if codec in ('h264', 'h265') and extra:
            kv = ":".join(f"{k}={v}" for k, v in extra.items())
            args += [f"-{'x265' if codec == 'h265' else 'x264'}-params", kv]
        return args

    def _run_with_progress(self, cmd, task):
        """执行 FFmpeg 并解析 -progress 输出上报进展"""
        proc = subprocess.Popen(cmd, stdout=subprocess.PIPE,
                                stderr=subprocess.PIPE, text=True)
        last_report = 0.0
        frames, out_bytes = 0, 0

        for line in proc.stdout:
            line = line.strip()
            if line.startswith('frame='):
                frames = int(line.split('=', 1)[1] or 0)
            elif line.startswith('total_size='):
                out_bytes = int(line.split('=', 1)[1] or 0)

            now = time.time()
            if now - last_report >= self.FFMPEG_PROGRESS_INTERVAL_SEC:
                # 关键：上报单调递增的实际帧数，不是"我还活着"
                self.heartbeat.report(task['task_id'], self.worker_id, {
                    'frames_encoded': frames,
                    'bytes_written': out_bytes,
                })
                last_report = now

                if self._check_preempt(task['task_id']):
                    proc.terminate()
                    proc.wait(timeout=10)
                    raise PreemptedError()

        proc.wait()
        if proc.returncode != 0:
            err = proc.stderr.read()[-2000:]
            raise RuntimeError(f"ffmpeg exit {proc.returncode}: {err}")

    def _check_preempt(self, task_id):
        """检查抢占信号"""
        return self.scheduler.is_preempted(task_id)

    def _output_key(self, task):
        """决定性输出路径——保证幂等"""
        rung_key = self._rung_key(task['rung'])
        shard = task.get('shard')
        suffix = f"shard{shard['shard_index']:04d}" if shard else "full"
        return f"transcode/{task['asset_id']}/{rung_key}/{suffix}.mp4"

    def _rung_key(self, rung):
        return f"{rung['height']}p_{rung['codec'].split('+')[0]}"

    def _verify_existing(self, output_key, rung):
        """幂等短路时的快速校验：只查容器与分辨率，不做完整质检"""
        meta = self.storage.probe(output_key)
        return meta is not None and meta.get('height') == rung['height']


class PreemptedError(Exception):
    """任务被抢占"""
    pass
```

## 异常场景补充

### 场景：源片编码异常导致 FFmpeg 解码死循环

```
触发：用户上传的视频含损坏的 SPS/PPS 头 → FFmpeg 解码器等待永不到来的数据
      → 进程存活但无进展 → 基于"进程存活"的心跳误判为正常
检测：
  1. 进展心跳：frames_encoded 单调递增才刷新时间戳，停滞 120 秒判僵死
  2. 实际耗时超过预估的 3 倍强制超时
  3. 同一 asset_id 在多台 worker 上连续失败 → 判定为源片问题而非机器问题
处理：
  1. 杀死进程并重新入队，强制换 CPU 软编（更鲁棒，能容错异常码流）
  2. 加 -err_detect ignore_err -fflags +genpts 容错参数重试
  3. 重试 3 次仍失败 → 标记永久失败并通知创作者"文件异常请重新上传"
预防：进展型心跳 + 耗时上限熔断 + 上传时前置探针校验码流合法性
```

### 场景：转码中间产物堆积占满临时存储

```
触发：分片转码产生大量中间文件 → 拼接失败或任务被抢占后未清理
      → 180TB 临时存储被占满 → 所有转码任务因无法写入而失败
检测：
  1. 临时存储使用率监控，超 75% 告警，超 88% 触发强制清理
  2. 中间产物年龄分布：存在超过 6 小时的中间文件即异常
  3. transcode_shard 表中 status=2 但父任务已完成的孤儿分片数量
处理：
  1. 立即清理超过 6 小时的中间产物（正常任务不会超过 1 小时）
  2. 清理已完成/已失败任务的全部分片中间文件
  3. 临时存储紧张时暂停接收新的分片任务，优先让在途任务完成
预防：中间产物强制 TTL 6 小时 + 任务终态触发清理 + 孤儿分片定时巡检
```

### 场景：热度晋升风暴导致集群被二次转码占满

```
触发：一场大型活动导致数万条内容同时突破晋升阈值
      → 大量 Tier 1 晋升任务（14 核时/分钟素材）涌入 → 挤占新投稿转码
      → 新投稿转码 P95 从 44 秒涨到 12 分钟
检测：
  1. 晋升任务占集群算力比例监控，超 25% 告警
  2. 新投稿转码 P95 与晋升任务队列长度的相关性监控
  3. transcode_promotion 表按分钟统计的晋升触发量突增
处理：
  1. 晋升任务优先级本就低于新投稿（PRIO_PROMOTION=3 < PRIO_SHORT_VIDEO=2），
     但需额外加算力配额上限：晋升任务最多占用 20% 集群
  2. 晋升任务按预估带宽收益排序，只做 ROI 最高的部分
  3. 超出配额的晋升任务推迟到低峰期（凌晨）执行
预防：晋升任务算力配额硬上限 20% + 按 ROI 排序 + 低峰期批量执行
```

### 场景：探针预测模型失效导致阶梯全面偏低

```
触发：平台内容形态快速竖屏化 → 探针模型训练分布与线上分布偏移
      → 预测码率系统性偏低 15% → 大量内容 VMAF 跌破 88 底线
检测：
  1. 质检 VMAF 分布监控：低于底线的产物占比突增（正常 <0.5%）
  2. 探针预测码率与质检实测所需码率的残差分布漂移检测
  3. 按内容特征分桶（横屏/竖屏/动画/实拍）的 VMAF 达标率对比
处理：
  1. 立即对预测置信度低的分桶启用"保守阶梯"（码率上浮 15%）
  2. 受影响内容进入重转码队列（低优先级，逐步修复）
  3. 用近期数据紧急重训模型，验证后灰度上线
预防：按内容分桶的 VMAF 达标率日报 + 预测残差漂移告警 + 模型月度重训
```
