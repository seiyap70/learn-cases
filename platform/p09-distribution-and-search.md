# P09: 分发决策——多形态混排、内容池与内容检索

## 业务场景

某统一内容平台的分发决策域，负责回答一个问题：**这一屏 20 个位置，从 40 亿候选内容里放什么。** 难点不在"推荐算法"——单形态的召回/粗排/精排/重排是成熟工程；难点在于这 20 个位置要同时承载**图文、短视频、长视频、直播**四种形态，而这四种形态的收益量纲、反馈周期、供给速度、商业模式**没有一项可比**。

**已知数据：**
- Feed 请求 QPS：均值 1.85 万 / 峰值 6.5 万，每次返回 20 条
- 日曝光量：320 亿次（Feed 16 亿次 × 20 条）
- 日播放量（VV）：160 亿，日总消费时长 5,121 亿秒（人均 42.7 分钟）
- 可召回候选池：40 亿条（未下架、未过期、质量达标）
- 日新增内容：图文 800 万、短视频 500 万、长视频 5 万、直播 40 万场（合计 1,345 万）
- 端到端延迟预算：Feed 请求 P99 < 200 ms（其中混排求解 ≤ 12 ms）
- 搜索请求：日 1.51 亿次（渗透率 18% × 人均 4.2 次），均值 1,750 QPS / 峰值 6,100 QPS
- 向量索引规模：160 亿条向量（40 亿内容 × 4 个关键帧），768 维
- 冷启动预算：总曝光的 5%（16 亿次/日）

**四种形态的结构性差异——这张表是全篇所有设计的出发点：**

| 形态 | 日新增 | 存量占比 | 曝光占比 | VV 占比 | 时长占比 | 单位曝光时长（EEV） |
|------|-------|---------|---------|---------|---------|-------------------|
| 图文 | 800 万 | 62% | 34% | 20.0% | 15.6% | 7.35 s |
| 短视频 | 500 万 | 35% | 52% | 73.8% | 50.7% | 15.6 s |
| 长视频 | 5 万 | 2.4% | **9%** | 4.4% | **24.6%** | **43.8 s** |
| 直播 | 40 万场 | 0.6% | 5% | 1.9% | 9.1% | 29.1 s |

**读这张表要读出的东西：长视频用 2.4% 的存量、9% 的曝光，贡献了 24.6% 的消费时长——它的单位曝光时长价值是短视频的 2.8 倍、图文的 6.0 倍。**

那为什么不把 Feed 全换成长视频？因为这张表还藏着三个反向约束：

- **供给不够**：日新增 5 万条。若把曝光占比提到 52%（166 亿次/日），每条长视频日均要被曝光 33 万次——远超其可被消费的次数，会退化为同一批内容反复刷屏。
- **决策成本高**：消费一条长视频要用户愿意投入 20 分钟。连续 3 条长视频后的跳出率是连续 3 条短视频的 4.1 倍。
- **时长不是唯一目标**：直播的单位曝光打赏 GMV（¥0.42）是长视频的 42 倍；图文的评论率是短视频的 2.3 倍，对社区氛围与次日留存的贡献无法用时长衡量。

**核心矛盾**：**四种形态在多个不可比的目标上各有优势，而它们要抢同一个 20 位的容器。** 纯数据驱动的排序会必然收敛到"即时反馈信号最强"的那一种（短视频），因为它的标签最快成熟、样本最多、方差最小——这不是模型选出的最优解，是**训练数据的结构偏差**选出的解。

**本篇与单形态推荐的边界**：召回/粗排/精排的三阶段架构、特征平台、Embedding 训练、冷启动的经典手法，都是**单形态推荐的成熟问题**，本篇不重写。本篇只解决统一平台特有的三件事——**跨形态可比性、内容池的流量分配、跨形态检索**。

## 核心挑战

### 挑战 1：四种形态的收益量纲不可比

推荐系统的排序需要一个标量分数。但四种形态的"好"是用不同单位度量的：

| 形态 | 主目标 | 度量单位 | 为什么不能跨形态比 |
|------|-------|---------|-----------------|
| 图文 | 阅读完成 + 互动 | 完读率、评论率 | 一篇图文的"完读"是 25 秒，一条长视频的"完播"是 45 分钟 |
| 短视频 | 完播 + 时长 | 完播率、播放时长 | 完播率 80% 的 15 秒视频 = 12 秒；完播率 20% 的 45 分钟视频 = 540 秒 |
| 长视频 | 追更 + 会员转化 | 集数进度、付费率 | 价值大部分在**未来**（追完整季、续订会员），本次曝光只是入口 |
| 直播 | 停留 + 打赏 | 同时在线、GMV | **有时效硬约束**：下播后价值归零，无法延后消费 |

一个具体的排序难题：

```
候选 A：15 秒短视频，预估完播率 0.82 → 预估时长 12.3 s
候选 B：45 分钟长剧集第 3 集，预估播放 0.19 → 预估时长 513 s
候选 C：正在直播的演唱会，预估停留 0.11 → 预估时长 162 s，但 40 分钟后下播
候选 D：图文长测评，预估完读 0.44 → 预估时长 27 s，但评论率 0.061（是 A 的 5.2 倍）

按预估时长排：B > C > A > D
按完播/完读率排：A > D > B > C
按互动率排：D > C > A > B
按商业化价值排：C > B > A > D

四个口径给出四个完全不同的顺序，且没有一个是错的。
```

**难点不是"选哪个口径"，而是这四个口径的权重本身随用户、时段、场景而变**——通勤时段的用户不适合长视频，睡前用户适合；新用户需要高完播率内容建立习惯，老用户可以承受长决策成本。**权重是一个需要在线学习的函数，不是一组常量。**

### 挑战 2：标签成熟期差 240 倍，造成系统性偏斜

模型用什么标签训练，就会优化什么。而四种形态的标签成熟速度差了两个数量级：

| 形态 | 标签成熟期 | 曝光后 24h 累积占最终值 | 曝光后 72h 占最终值 |
|------|-----------|---------------------|-------------------|
| 短视频 | 约 30 秒 | 97% | 100% |
| 图文 | 约 5 分钟 | 96% | 100% |
| 直播 | 场次结束（均 2.4 小时） | 100%（当日必然结束） | 100% |
| 长视频 | **约 72 小时**（分段追看） | **31%** | 92% |

用 T+1 的标签训练（这是绝大多数推荐系统的做法，因为要保证模型时效性），长视频的真实价值被系统性低估：

```
低估幅度 = 1 − (长视频 24h 标签成熟度 ÷ 短视频 24h 标签成熟度)
        = 1 − (0.31 ÷ 0.97)
        = 68%
```

**模型看到的长视频价值只有真实值的 32%。** 这个偏差不是随机噪声，是**恒定方向的系统性偏差**，且它会自我强化：

```
T+1 标签低估长视频 68%
  → 模型降低长视频曝光
  → 长视频样本量减少（从 9% 降到 6%）
  → 长视频的预估方差变大，置信度降低
  → 排序时进一步被保守压制（不确定性惩罚）
  → 曝光继续下降 …
  → 长视频创作者（专业机构、影视版权方）看到分发量下滑，减少供给
  → 供给减少 → 存量占比下降 → 候选池里更找不到好的长视频
  → 平台内容结构单一化，用户时长天花板下降
```

**这是一个跨越「模型 → 曝光 → 供给 → 内容生态」的正反馈退化环，且每一环单独看都"符合数据"。** 它不会触发任何告警——所有在线指标（CTR、完播率、人均时长短期值）都在变好。

### 挑战 3：混排是带约束的装箱问题，不是排序问题

工程师的默认实现是"算分 → 排序 → 取 Top 20"。但真实的混排必须同时满足一组约束：

```
20 个位置上的约束清单（全部同时生效）：
  形态配额     图文 5-9 条、短视频 8-12 条、长视频 1-3 条、直播 0-2 条
  多样性       同一创作者 ≤ 2 条、同一话题 ≤ 3 条、同一形态连续 ≤ 3 条
  位置约束     直播只能出现在 1-8 位（时效性，越后越可能已下播）
  供给保量     冷启动池内容 ≥ 2 条（否则新内容永无曝光机会）
  商业约束     第 4、11、18 位为广告位，不参与自然排序
  体验约束     首屏 3 条内至少 1 条高确定性内容（防止首屏就跳出）
  合规约束     敏感话题内容 ≤ 1 条，且不得相邻
```

"排序 + Top 20"无法表达任何一条约束。加上约束后，它变成一个**带多重线性约束的最优选择问题**——精确解是 NP-hard，而在线预算只有 **12 ms**，QPS 峰值 **6.5 万**。

更麻烦的是：**约束之间会冲突且无解。** 例如某用户的候选池里长视频只有 1 条且质量很差，"长视频 ≥ 1 条"与"质量下限"直接矛盾。**在线求解器必须能在无解时给出可解释的降级顺序，而不是抛异常或返回空。**

### 挑战 4：冷启动预算除以内容量，得到的样本数不足以做质量判断

冷启动池拿到总曝光的 5%（16 亿次/日），日新增内容 1,345 万条，于是：

```
单条冷启动曝光量 = 16 亿 ÷ 1,345 万 ≈ 120 次
```

120 次曝光够不够判断一条内容的质量？做个统计检验：

```
假设：判断"播放率 55% 的好内容"与"播放率 50% 的平庸内容"
      p = 0.5，需检出差异 Δ = 0.05
      α = 0.05（双侧），power = 0.80

  所需样本量 n = 2 × (1.96 + 0.84)² × 0.25 ÷ 0.05²
                = 2 × 7.84 × 0.25 ÷ 0.0025
                = 1,568 次曝光

  实际预算 120 次 → 差 13.1 倍
  120 次曝光下的 95% 置信区间：50% ± 8.9 个百分点
```

**冷启动的 120 次曝光，置信区间宽到无法区分 Top 20% 与中位数内容。** 这个结论很反直觉，但它是纯算术——不管用什么模型、什么特征，样本量摆在那里。

**推论：冷启动阶段的晋升决策必然是高噪声的。** 试图用这 120 次的后验数据做精细排序是自欺欺人。它只能做一件统计上站得住脚的事——**排除明显有问题的内容**（播放率 < 15% vs 50%，差异 35 个百分点，120 次样本足以在 α=0.001 下检出）。精细判断必须靠**先验**：创作者历史表现、内容理解特征、相似内容的表现。

### 挑战 5：跨形态检索的可比性问题与混排同源

搜索看起来是另一个域，但它的核心难题与混排完全同构。用户搜「周杰伦」：

| 可能的意图 | 最佳形态 | 判据 |
|-----------|---------|------|
| 想听歌 / 看 MV | 长视频 | 官方 MV、完整版 |
| 想看名场面 | 短视频 | 演唱会切片、综艺片段 |
| 想看乐评 / 歌单 | 图文 | 深度长文 |
| 想看有人正在唱 | 直播 | **正在开播的翻唱间** |

**同一个 query 在四种形态上的"最佳结果"完全不同，而搜索结果页需要把它们排在一个列表里——这就是混排问题，只是多了一个 query 条件。**

在此之上，视频还有一个图文没有的困难：**文本可检索性天生匮乏**。

```
一篇图文的可检索文本：标题 28 字 + 正文 1,800 字 = 1,828 字
一条长视频的可检索文本：标题 24 字 + 简介 60 字 = 84 字

信息量差 21.8 倍。
仅靠标题+简介，视频的召回率上限约 0.34（实测）——
2/3 的相关视频因为"标题里没有这个词"而检索不到。
```

必须从媒体内容本身抽取文本：ASR 语音转写、OCR 字幕/画面文字、视觉标签。一条 8 分钟长视频 ASR 出约 1,200 字，是标题+简介的 14 倍。**视频搜索的召回率上限，本质上由 ASR/OCR 的覆盖率与准确率决定，而不是由检索引擎决定。**

## 设计约束

- Feed 请求 P99 < 200 ms，其中混排求解 ≤ 12 ms（峰值 6.5 万 QPS）
- 形态曝光占比必须在小时粒度收敛到目标值 ±2 个百分点
- 长视频与直播的曝光占比不得因短期指标优化而低于保量下限（长视频 ≥ 7%、直播 ≥ 3%）
- 冷启动池必须保证每条新内容在 24 小时内获得 ≥ 100 次曝光
- 搜索 P99 < 350 ms，Recall@100 ≥ 0.90
- 约束冲突无解时必须降级而非报错，且降级顺序可配置、可审计

## 请先独立思考（限时 45 分钟）

1. 四种形态的分数如何统一到可比的量纲？如果用"预估消费时长"，长视频会不会通吃？如果加形态系数，这些系数从哪里来、怎么更新？
2. 20 个位置 + 7 类约束 + 12 ms 预算，你怎么求解？如果放弃精确解，放弃的是什么、保住的是什么？
3. 长视频标签 72 小时才成熟，而模型必须 T+1 训练。你怎么在不牺牲模型时效性的前提下消除 68% 的低估？
4. 冷启动只给得起 120 次曝光，而统计显著需要 1,568 次。你用这 120 次做什么判断、不做什么判断？
5. 搜索「周杰伦」时，正在直播的翻唱间应该排第几？它的排序分数怎么与官方 MV 比较？

---

## 设计解析

### 混排的问题定义：从「排序」到「带约束装箱」

先把问题写清楚。这一步比选算法重要——**大多数混排系统的问题不是求解器不好，是问题定义错了。**

```
输入：
  候选集 C = {c₁ … cₙ}，n ≈ 800（精排输出）
  每个候选有：形态 f(c) ∈ {ARTICLE, SHORT, LONG, LIVE}
             多目标预估向量 y(c) = [时长, 互动, 商业化, 留存, 满意度]
             元信息（创作者、话题、内容池等级、时效截止时间）
  位置集 P = {p₁ … p₂₀}，位置有权重 w(p)（第 1 位权重 1.00，第 20 位 0.21）

输出：
  分配方案 A: P → C ∪ {∅}

目标：
  maximize  Σ_p w(p) · V(A(p))          ← V 是统一价值函数（下一节定义）

约束：
  形态配额   ∀f: min_f ≤ |{p : f(A(p))=f}| ≤ max_f
  创作者     ∀u: |{p : author(A(p))=u}| ≤ 2
  话题       ∀t: |{p : topic(A(p))=t}| ≤ 3
  连续形态   ∀i: ¬(f(A(pᵢ))=f(A(pᵢ₊₁))=f(A(pᵢ₊₂)))
  直播位置   f(A(p))=LIVE ⟹ index(p) ≤ 8
  时效       f(A(p))=LIVE ⟹ deadline(A(p)) > now + 5min
  冷启保量   |{p : pool(A(p))=COLD_START}| ≥ 2
  广告位     index(p) ∈ {4,11,18} ⟹ A(p) 由广告系统填充
```

**这个形式化本身就澄清了三件事：**

1. **目标里有 `w(p)`——位置权重**。这意味着"把最好的内容放第 1 位"不一定最优：如果第 1 位放长视频会让后面 19 个位置的形态配额受挤压，总价值可能反而下降。**排序的贪心最优 ≠ 装箱的全局最优。**
2. **约束里有"连续形态"这种涉及位置相邻关系的项**，它让问题不再是简单的背包，而是带序列约束的分配——这是"排序 + 过滤"这类实现根本表达不了的。
3. **`min_f` 下限的存在，是整个混排设计里唯一能打断挑战 2 那个退化环的机制**。它是一个**非数据驱动的、人为设定的结构性保护**。所有纯优化的方案都会把它优化掉。

**判据：如果你的混排代码里没有一处是"违背短期指标但必须保留"的硬约束，那么挑战 2 描述的生态退化正在发生，只是还没被观测到。**

### 统一价值函数：EEV 与形态修正

需要一个把四种形态映射到同一实数轴的函数 `V(c)`。分三层构造。

**第一层：期望消费时长（EEV, Expected Engagement Value）。** 这是唯一天然可比的物理量——秒。

```python
class ValueEstimator:
    """统一价值函数：把四种形态的多目标预估折算到可比标量

    分三层：
      1. EEV     —— 期望消费时长（秒），唯一天然可比的物理量
      2. 多目标   —— 互动/商业化/留存折算成"等效秒数"
      3. 形态修正 —— 修正标签成熟度偏差与体验成本
    """

    # 各形态的标签成熟度修正系数：T+1 标签 → 最终值的外推倍数
    # 来源：label_maturity_curve 表，按形态 × 内容子类分桶，每周重算
    MATURITY_MULTIPLIER = {
        "ARTICLE": 1.04,   # 图文 24h 成熟度 96%
        "SHORT":   1.03,   # 短视频 24h 成熟度 97%
        "LIVE":    1.00,   # 直播当日必然结束，无需外推
        "LONG":    3.23,   # 长视频 24h 成熟度 31% → 1/0.31，最关键的一个系数
    }

    # 体验成本系数：消费该形态需要用户付出的决策成本，折算为价值折扣
    # 长视频要求用户投入 20 分钟，这个"承诺"本身有成本
    DECISION_COST = {
        "ARTICLE": 0.98,
        "SHORT":   1.00,   # 基准：零决策成本，划到就播
        "LIVE":    0.94,   # 需要接受"无法暂停/回看"
        "LONG":    0.86,   # 需要接受长时间投入
    }

    # 等效秒数换算率：把非时长目标折算成秒
    # 来源：离线因果实验（每单位目标变化对次日留存的贡献 ÷ 每秒时长对次日留存的贡献）
    EQUIV_SECONDS = {
        "interact":  42.0,   # 一次评论/转发 ≈ 42 秒消费时长的留存价值
        "gmv":       18.0,   # 每 ¥1 打赏/带货 GMV ≈ 18 秒
        "retention":  0.0,   # 留存已通过上面两项间接计入，避免重复
        "satisfy":   26.0,   # 一次显式正反馈（关注/收藏）≈ 26 秒
    }

    NEGATIVE_PENALTY = 380.0   # 一次显式负反馈（不感兴趣/举报）扣 380 等效秒

    def estimate(self, cand, user_ctx):
        """返回统一价值标量（等效秒数）"""
        # 1. 基础 EEV：预估播放率 × 预估单次消费时长
        eev = cand.pred_play_rate * cand.pred_duration_sec

        # 2. 标签成熟度外推 —— 消除挑战 2 描述的 68% 系统性低估
        #    注意：外推系数按形态 × 内容子类取，不用全局均值（见陷阱 3）
        eev *= self.maturity_of(cand)

        # 3. 多目标折算成等效秒数
        eev += cand.pred_interact_rate * self.EQUIV_SECONDS["interact"]
        eev += cand.pred_gmv * self.EQUIV_SECONDS["gmv"]
        eev += cand.pred_satisfy_rate * self.EQUIV_SECONDS["satisfy"]
        eev -= cand.pred_negative_rate * self.NEGATIVE_PENALTY

        # 4. 体验成本折扣（形态固有 × 用户当前场景）
        eev *= self.DECISION_COST[cand.form]
        eev *= self.scene_fitness(cand.form, user_ctx)

        # 5. 形态影子价格 —— 由配额约束的对偶变量决定，见下节
        #    这是把"全局配额约束"下沉到"单条打分"的桥梁
        eev *= self.shadow_price.get(cand.form, 1.0)

        return eev

    def maturity_of(self, cand):
        """标签成熟度外推系数：按形态 × 内容子类分桶

        为什么不能用形态级全局均值：剧集的追更曲线（3 天看完）与
        纪录片（30 天看完）差异巨大，全局均值会高估长尾、低估爆款
        """
        key = (cand.form, cand.sub_type)
        return self.maturity_table.get(key) or self.MATURITY_MULTIPLIER[cand.form]

    def scene_fitness(self, form, user_ctx):
        """场景适配度：同一形态在不同场景下的价值不同

        这是挑战 1 里"权重随用户/时段变化"的落地。
        通勤时段（移动、碎片时间）不适合长视频；睡前时段适合。
        """
        # 1. 网络环境：弱网下长视频体验差，价值打折
        if user_ctx.net_type in ("2G", "3G") and form in ("LONG", "LIVE"):
            return 0.55
        # 2. 时段：由离线统计的「形态 × 时段」完播率相对值查表
        base = self.scene_table[(form, user_ctx.hour_bucket, user_ctx.motion_state)]
        # 3. 新用户保护：前 7 天优先建立"刷得爽"的习惯，压制高决策成本形态
        if user_ctx.days_since_reg < 7 and form == "LONG":
            base *= 0.72
        return base
```

**这个类里最重要的一行是 `MATURITY_MULTIPLIER["LONG"] = 3.23`。** 它把挑战 2 那个 68% 的系统性低估直接乘回来。而它最容易被漏掉——因为漏掉它之后，所有在线指标（CTR、完播率、短期人均时长）都会**变好**，只有长视频供给侧的指标会在**三个月后**开始下滑。

**为什么负反馈的惩罚（380 秒）比任何正向收益都大一个量级？** 因为负反馈的代价不在本次曝光，在**用户对推荐系统的信任度**——一次"不感兴趣"会显著降低该用户后续 7 天的 Feed 消费深度。用等效秒数表达时必须把这个跨期影响算进去，否则模型会为了短期时长而容忍高负反馈内容。

### 形态影子价格：把在线约束求解降级为全局统计控制

现在处理挑战 3 的 12 ms 预算。核心思路是**把"每请求硬满足配额"放松成"全站小时级统计满足配额"**。

数学依据是拉格朗日松弛。原问题的形态配额约束 `Σ_c x_c·1[f(c)=f] ≤ max_f` 引入对偶变量 λ_f 后，进入目标函数变成 `V(c) − λ_{f(c)}`。**关键观察：λ_f 是一个全局量，不依赖单个请求。** 于是：

```
朴素做法（每请求求解）：
  每个请求跑一次约束优化 → 6.5 万 QPS × 求解耗时 40 ms = 需 2,600 核，且 P99 超预算

影子价格做法：
  λ_f 由一个全局控制器按小时更新（离线，与请求无关）
  每个请求只做：打分（已含 λ_f）→ 贪心装箱 → 8.4 ms
  单请求可能违反配额，全站小时级配额精确达标
```

**放弃了什么、保住了什么，要说清楚：**

| | 每请求求解 | 影子价格 + 贪心 |
|---|---|---|
| 单请求配额 | 硬保证 | **可能违反**（某个用户这一刷可能 0 条长视频） |
| 全站小时配额 | 硬保证 | 收敛到 ±1.3 个百分点（实测） |
| 求解耗时 | 40 ms | **8.4 ms** |
| 峰值算力 | 2,600 核 | **546 核** |
| 无解处理 | 抛异常/回退 | 天然无"无解"（约束变成软惩罚） |

**单请求违反配额是可以接受的，且反而更好。** 用户 A 这一刷没有长视频，用户 B 这一刷有 3 条——只要全站占比达标，且个体层面由用户长期兴趣决定谁多谁少，这比"每个人每刷都强塞 1 条长视频"的体验更好。**硬配额下沉到每个请求，是把统计目标误当成个体约束。**

```python
class ShadowPriceController:
    """形态影子价格控制器：用 PID 把实际曝光占比拉到目标占比

    每 5 分钟执行一次，与在线请求完全解耦。
    输出的 λ_f 通过配置中心下发，在线侧只做乘法。
    """

    TARGET_SHARE = {              # 目标曝光占比
        "ARTICLE": 0.34,
        "SHORT":   0.52,
        "LONG":    0.09,
        "LIVE":    0.05,
    }
    FLOOR_SHARE = {               # 保量下限：跌破则强制拉升，不受 PID 平滑约束
        "ARTICLE": 0.28,
        "SHORT":   0.40,
        "LONG":    0.07,          # 打断挑战 2 退化环的硬底
        "LIVE":    0.03,
    }

    KP, KI, KD = 0.85, 0.12, 0.30   # PID 系数，由离线仿真调参
    LAMBDA_MIN, LAMBDA_MAX = 0.45, 2.60   # 影子价格上下限，防失控
    INTEGRAL_CLAMP = 0.40                 # 积分项限幅，防积分饱和
    CONTROL_PERIOD_SEC = 300              # 控制周期 5 分钟

    def tick(self):
        """一个控制周期"""
        actual = self.metrics.form_share(window_sec=self.CONTROL_PERIOD_SEC)
        new_lambda = {}

        for form, target in self.TARGET_SHARE.items():
            err = target - actual.get(form, 0.0)          # 正=曝光不足，需拉升

            # 1. 积分项累积 + 限幅（防积分饱和：长期偏差会让积分项爆掉）
            self.integral[form] = self._clamp(
                self.integral[form] + err * self.CONTROL_PERIOD_SEC / 3600.0,
                -self.INTEGRAL_CLAMP, self.INTEGRAL_CLAMP)

            # 2. 微分项用误差变化率，抑制超调
            deriv = (err - self.prev_err[form]) / (self.CONTROL_PERIOD_SEC / 3600.0)
            self.prev_err[form] = err

            # 3. PID 输出映射为乘性影子价格（乘性而非加性，保持尺度无关）
            adjust = self.KP * err + self.KI * self.integral[form] + self.KD * deriv
            lam = self._clamp(self.lambda_[form] * (1.0 + adjust / max(target, 1e-6)),
                              self.LAMBDA_MIN, self.LAMBDA_MAX)

            # 4. 保量下限硬干预：跌破 floor 时绕过 PID，直接跳到上限
            #    理由见下方说明——生态保护不能等 PID 慢慢收敛
            if actual.get(form, 0.0) < self.FLOOR_SHARE[form]:
                lam = self.LAMBDA_MAX
                self.alarm.fire("form_share_below_floor", form=form,
                                actual=actual.get(form), floor=self.FLOOR_SHARE[form])

            new_lambda[form] = lam

        # 5. 归一化：影子价格只有相对大小有意义，归一化避免整体漂移
        #    否则四个 λ 可能一起涨，等价于什么都没调，还破坏了 EEV 的绝对量纲
        mean = sum(new_lambda.values()) / len(new_lambda)
        new_lambda = {f: v / mean for f, v in new_lambda.items()}

        self.lambda_ = new_lambda
        self.config_center.publish("feed.form_shadow_price", new_lambda,
                                   ttl_sec=self.CONTROL_PERIOD_SEC * 4)
        return new_lambda

    @staticmethod
    def _clamp(v, lo, hi):
        return max(lo, min(hi, v))
```

**为什么保量下限要绕过 PID 直接跳到上限？** PID 是为**平滑收敛**设计的，它会花几个周期慢慢把偏差纠回来。但保量下限保护的是**生态**——长视频曝光跌破 7% 意味着创作者已经在感受分发下滑，这个损失是不可逆的（创作者一旦流失不会因为下周恢复配额就回来）。**平滑收敛对可逆的偏差是优点，对不可逆的损失是缺陷。** 判据：**控制器的响应速度应当匹配被控对象的不可逆程度，而不是统一追求平滑。**

**为什么要归一化（第 5 步）？** 这是一个真实踩过的坑：不归一化时，如果四种形态的实际占比同时偏低（比如广告位占比临时调高，挤压了全部自然内容），PID 会把四个 λ 一起拉高。**四个都乘 2.0 等价于什么都没做**（相对顺序不变），但 `V(c)` 的绝对值翻倍了——而下游有若干依赖 EEV 绝对值的逻辑（如"低于 8 等效秒的候选直接丢弃"的阈值），它们会集体失效。**任何只有相对意义的量，都必须显式归一化，否则绝对值会随控制过程漂移并污染下游阈值。**

### 在线混排求解器：贪心 + 位置感知 + 可解释降级

有了含影子价格的分数，在线求解退化为一次贪心装箱。但贪心的顺序与降级策略里有几处非平凡的设计。

```python
class FeedMixer:
    """在线混排求解器：贪心装箱 + 硬约束校验 + 可解释降级

    预算 12 ms / 请求，峰值 6.5 万 QPS。
    """

    SLOT_COUNT = 20
    AD_SLOTS = (3, 10, 17)              # 0-indexed 的广告位：第 4、11、18 位
    LIVE_MAX_INDEX = 7                  # 直播只能出现在前 8 位（0-indexed ≤ 7）
    LIVE_MIN_REMAIN_SEC = 300           # 直播剩余时长下限：< 5 分钟不再曝光
    MAX_PER_AUTHOR = 2
    MAX_PER_TOPIC = 3
    MAX_CONSECUTIVE_FORM = 3            # 同形态最多连续 3 条
    MIN_COLD_START = 2                  # 冷启动池保量
    FIRST_SCREEN = 3                    # 首屏条数
    FIRST_SCREEN_MIN_CONFIDENCE = 0.72  # 首屏至少 1 条置信度 ≥ 0.72 的内容

    # 无解时的降级顺序：从最先放弃的排到最后放弃的
    # 这个顺序是产品决策，不是技术决策——它编码了"哪些约束更重要"
    DEGRADE_ORDER = (
        "topic_diversity",      # 1. 先放弃话题多样性（用户最不易感知）
        "consecutive_form",     # 2. 再放弃"同形态不连续"
        "author_diversity",     # 3. 再放弃创作者去重
        "form_quota_max",       # 4. 再放弃形态配额上限
        "cold_start_floor",     # 5. 再放弃冷启动保量
        "first_screen_conf",    # 6. 再放弃首屏置信度
        # form_quota_min 与 compliance 永不放弃 —— 宁可返回不足 20 条
    )

    def mix(self, candidates, user_ctx, trace):
        # 1. 预过滤：硬性不可用的候选先剔除，减少后续遍历量
        pool = self._prefilter(candidates, user_ctx)

        # 2. 打分并排序（分数已含形态影子价格）
        for c in pool:
            c.score = self.value_estimator.estimate(c, user_ctx)
        pool.sort(key=lambda c: c.score, reverse=True)

        # 3. 先满足下限约束（保量），再做自由贪心
        #    顺序很关键：如果先自由贪心填满 20 位再回头补下限，
        #    补的时候只能替换已选内容，会破坏已建立的多样性状态
        state = _MixState(self.SLOT_COUNT, self.AD_SLOTS)
        self._fill_floors(pool, state, user_ctx, trace)

        # 4. 自由贪心填充剩余位置
        self._fill_greedy(pool, state, user_ctx, trace)

        # 5. 若仍有空位，按降级顺序逐条放宽约束重试
        relaxed = []
        for rule in self.DEGRADE_ORDER:
            if state.is_full():
                break
            relaxed.append(rule)
            state.relax(rule)
            self._fill_greedy(pool, state, user_ctx, trace)

        if relaxed:
            trace.note("mix_degraded", rules=relaxed, filled=state.filled_count())
            self.metrics.incr("feed.mix.degrade", tags={"rules": ",".join(relaxed)})

        # 6. 位置感知重排：在已选集合内部优化位置分配
        #    贪心是"按分数依次占位"，但高分内容不一定该放第 1 位
        result = self._assign_positions(state.selected(), user_ctx)

        # 7. 最终校验：永不放弃的约束若被违反，宁可返回不足 20 条
        return self._final_guard(result, trace)

    def _prefilter(self, candidates, user_ctx):
        """预过滤：剔除硬性不可用候选

        放在最前面是为了减少打分量——打分是最贵的一步（要查影子价格、
        场景表、成熟度表），过滤掉 1/3 候选能省 1/3 的打分开销。
        """
        out = []
        for c in candidates:
            # 1. 直播时效：剩余时长不足则丢弃（曝光了也来不及看）
            if c.form == "LIVE" and c.remain_sec < self.LIVE_MIN_REMAIN_SEC:
                continue
            # 2. 已读去重：近 30 天曝光过且未消费的不再出
            if self.seen_filter.contains(user_ctx.uid, c.content_id):
                continue
            # 3. 合规：地域封禁、年龄分级、用户屏蔽
            if not self.compliance.visible(c, user_ctx):
                continue
            # 4. 内容池等级：低于用户当前可见等级的不出
            if c.pool_level < user_ctx.min_pool_level:
                continue
            out.append(c)
        return out

    def _fill_floors(self, pool, state, user_ctx, trace):
        """先满足下限约束：形态配额下限 + 冷启动保量

        为什么下限要先填：下限是"必须有"，上限是"不能超"。
        先填必须有的，剩下的位置做自由竞争，才不会出现
        "20 位填满后发现缺 1 条长视频，只能踢掉一条高分短视频"的情况。
        """
        # 1. 形态配额下限
        for form, min_n in self.quota.min_by_form(user_ctx).items():
            need = min_n - state.count_form(form)
            for c in pool:
                if need <= 0:
                    break
                if c.form != form or state.contains(c):
                    continue
                if state.try_place(c, check_max_quota=False):
                    need -= 1
            if need > 0:
                # 候选池里这个形态的内容不够 —— 供给侧问题，需要告警
                trace.note("form_floor_unmet", form=form, short_by=need)
                self.metrics.incr("feed.mix.floor_unmet", tags={"form": form})

        # 2. 冷启动保量
        need = self.MIN_COLD_START - state.count_pool("COLD_START")
        for c in pool:
            if need <= 0:
                break
            if c.pool != "COLD_START" or state.contains(c):
                continue
            if state.try_place(c, check_max_quota=False):
                need -= 1

    def _fill_greedy(self, pool, state, user_ctx, trace):
        """自由贪心：按分数依次尝试放入，通过约束校验则占位"""
        for c in pool:
            if state.is_full():
                return
            if state.contains(c):
                continue
            state.try_place(c)

    def _assign_positions(self, selected, user_ctx):
        """位置感知重排：已选内容集合固定，优化它们的位置分配

        这一步解决"贪心按分数占位"的两个问题：
          1. 高分长视频放第 1 位会挤压首屏的形态多样性
          2. 直播必须在前 8 位，但贪心可能把它放到第 12 位

        用的是位置权重 × 分数的匹配，而不是纯分数排序。
        规模只有 20×20，可以做精确的匈牙利匹配（约 0.4 ms）。
        """
        # 1. 硬位置约束先钉死
        pinned = {}
        for c in selected:
            if c.form == "LIVE":
                pinned[c] = self._best_free_index(pinned, limit=self.LIVE_MAX_INDEX)

        # 2. 剩余内容与剩余位置做加权匹配
        #    收益矩阵 gain[i][j] = score_i × pos_weight_j × form_pos_affinity
        free_items = [c for c in selected if c not in pinned]
        free_slots = [i for i in range(self.SLOT_COUNT)
                      if i not in self.AD_SLOTS and i not in pinned.values()]
        gain = [[c.score * self.POS_WEIGHT[j] *
                 self._form_pos_affinity(c.form, j, user_ctx)
                 for j in free_slots] for c in free_items]
        matching = hungarian_max(gain)          # 20×20 精确匹配，约 0.4 ms

        # 3. 首屏置信度兜底：若首屏 3 条都是低置信内容，换入一条高置信的
        layout = self._materialize(pinned, free_items, free_slots, matching)
        return self._ensure_first_screen_confidence(layout)
```

**约束状态机 `_MixState`：全部约束逻辑的实际所在**

`FeedMixer` 读起来很简洁，是因为所有约束判断都被下沉到了 `_MixState`。这个类值得单独实现——**贪心装箱的正确性与耗时，几乎完全由它的增量维护是否正确决定。**

```python
class _MixState:
    """混排的约束状态机：增量维护全部约束，try_place 必须是 O(1)

    为什么必须 O(1)：贪心要遍历 800 个候选、每个候选尝试放入一次，
    若每次都全量重算约束（遍历已选 20 条统计形态/作者/话题），
    则是 800 × 20 = 1.6 万次比较。看起来不多，但乘以 6.5 万 QPS
    就是每秒 10.4 亿次 —— 这一步会独占整个 12 ms 预算。
    增量维护把它降到 800 次 O(1) 判断。
    """

    def __init__(self, slot_count, ad_slots, quota, limits):
        self.slot_count = slot_count
        self.ad_slots = set(ad_slots)
        self.quota = quota                      # {form: (min, max)}
        self.limits = limits                    # 作者/话题/连续数上限
        # 可用位置（升序），广告位不参与自然内容装箱
        self.free = [i for i in range(slot_count) if i not in self.ad_slots]
        # provisional 序列：贪心阶段的临时顺序，_assign_positions 会重排
        self.seq = []                           # [(slot_index, cand)]
        self.chosen_ids = set()
        # 增量计数器 —— 这四个 dict 是 O(1) 的全部来源
        self.form_count = defaultdict(int)
        self.author_count = defaultdict(int)
        self.topic_count = defaultdict(int)
        self.pool_count = defaultdict(int)
        # 已放宽的规则集合
        self.relaxed = set()

    # ---------- 查询接口 ----------
    def is_full(self):
        return not self.free

    def contains(self, c):
        return c.content_id in self.chosen_ids

    def count_form(self, form):
        return self.form_count[form]

    def count_pool(self, pool):
        return self.pool_count[pool]

    def filled_count(self):
        return len(self.seq)

    def selected(self):
        return [c for _, c in self.seq]

    def relax(self, rule):
        """放宽一条约束。注意：放宽是单向的，本次请求内不会再收紧"""
        self.relaxed.add(rule)

    # ---------- 核心：约束校验 + 占位 ----------
    def try_place(self, c, check_max_quota=True):
        """尝试把候选 c 放入下一个可用位置。通过全部约束才占位。

        返回 True 表示已占位。调用方不需要知道是哪条约束拒绝了 ——
        但被拒原因会累积到 reject_stats，用于事后诊断"为什么这一刷
        只有 12 条内容"。
        """
        slot = self.free[0]
        reason = self._violates(c, slot, check_max_quota)
        if reason:
            self.reject_stats[reason] += 1
            return False

        self.free.pop(0)
        self.seq.append((slot, c))
        self.chosen_ids.add(c.content_id)
        self.form_count[c.form] += 1
        self.author_count[c.author_id] += 1
        self.topic_count[c.topic_id] += 1
        self.pool_count[c.pool] += 1
        return True

    def _violates(self, c, slot, check_max_quota):
        """返回被违反的约束名，或 None 表示全部通过

        校验顺序按「判断成本升序 + 拒绝率降序」排列：
        最便宜且最常拒绝的先判，让大多数候选在前两项就被短路掉。
        """
        # 1. 位置类硬约束（最便宜，且永不放宽）
        if c.form == "LIVE":
            if slot > self.limits["live_max_index"]:
                return "live_position"
            if c.remain_sec < self.limits["live_min_remain_sec"]:
                return "live_freshness"

        # 2. 形态配额上限（拒绝率最高的一项，尤其在长视频供给充足时）
        if check_max_quota and "form_quota_max" not in self.relaxed:
            _, max_n = self.quota.get(c.form, (0, self.slot_count))
            if self.form_count[c.form] >= max_n:
                return "form_quota_max"

        # 3. 创作者去重
        if "author_diversity" not in self.relaxed:
            if self.author_count[c.author_id] >= self.limits["max_per_author"]:
                return "author_diversity"

        # 4. 话题去重
        if "topic_diversity" not in self.relaxed:
            if self.topic_count[c.topic_id] >= self.limits["max_per_topic"]:
                return "topic_diversity"

        # 5. 连续形态：只看 provisional 序列的尾部 k-1 条
        #    这是唯一依赖「顺序」而非「计数」的约束，也是唯一
        #    会被 _assign_positions 破坏的约束 —— 见下方说明
        if "consecutive_form" not in self.relaxed:
            k = self.limits["max_consecutive_form"]
            tail = [cand.form for _, cand in self.seq[-(k - 1):]]
            if len(tail) == k - 1 and all(f == c.form for f in tail):
                return "consecutive_form"

        return None
```

**这个类里有一个必须显式处理的不一致：`consecutive_form` 在贪心阶段按 provisional 顺序校验，但 `_assign_positions` 之后位置全变了。** 也就是说贪心阶段辛苦维持的"同形态不连续 3 条"，可能在匈牙利匹配重排位置之后被破坏。

这不是一个可以忽略的边角问题——匹配的目标函数里没有任何一项关心相邻关系，所以它**很容易**产出连续 4 条短视频的布局。三种处理方式：

```
方案 A：把连续约束编码进匈牙利匹配
  ✗ 匈牙利算法求解的是线性分配问题，目标函数必须可分解为 Σ gain[i][j]
    而"相邻不同形态"是位置对之间的耦合项，无法表达
  → 需要换成 ILP 或带约束的束搜索，耗时从 0.4 ms 涨到 6+ ms，超预算

方案 B：匹配后做局部修复（选定）
  ✓ 匹配完成后扫一遍布局，发现连续 k 条同形态则做一次相邻交换
  ✓ 交换只在"同形态块"与"最近的异形态项"之间进行，代价可控
  ✓ 实测修复触发率 11.3%，平均交换 1.4 次，耗时 P99 0.12 ms
  ✗ 交换会略微降低目标值（实测 −0.31%），因为破坏了最优匹配

方案 C：放弃连续约束
  ✗ 连续 5 条短视频会显著抬升跳出率（实测 +2.1pp）
```

**选择方案 B**，实现如下。这是一个典型的"用后处理修复换求解器简单性"的取舍：

```python
    def repair_consecutive(self, layout, max_consecutive):
        """匹配后的连续形态修复：滑窗检测 + 最近异形态交换

        layout: [cand or None]，下标即位置。广告位为 None。
        返回修复后的 layout（原地修改）。

        为什么放在匹配之后而不是之前：匹配保证了"内容与位置"的全局
        最优对应，修复只做局部微调。反过来（先保证不连续再匹配）会
        让匹配的可行域被切碎，反而丢掉更多目标值。
        """
        n = len(layout)
        i = 0
        swaps = 0
        while i <= n - max_consecutive:
            window = [layout[j] for j in range(i, i + max_consecutive)]
            if any(c is None for c in window):          # 跨广告位不算连续
                i += 1
                continue
            if len({c.form for c in window}) > 1:       # 窗口内有异形态，合法
                i += 1
                continue

            # 窗口内 max_consecutive 条同形态 —— 找最近的异形态项交换
            bad_form = window[0].form
            donor = self._nearest_other_form(layout, i + max_consecutive - 1, bad_form)
            if donor is None:
                # 整个布局只有这一种形态（供给极度不足），无法修复
                # 记录但不阻断 —— 此时形态配额下限必然也没满足，已有告警
                self.trace.note("consecutive_unrepairable", form=bad_form, at=i)
                break

            tgt = i + max_consecutive - 1                # 换掉窗口最后一条
            layout[tgt], layout[donor] = layout[donor], layout[tgt]
            swaps += 1
            # 交换可能在 donor 处引入新的连续块，回退窗口重新检查
            i = max(0, min(i, donor - max_consecutive + 1))

        if swaps:
            self.metrics.observe("feed.mix.consecutive_repair_swaps", swaps)
        return layout

    @staticmethod
    def _nearest_other_form(layout, from_index, bad_form):
        """从 from_index 向两侧扩散，找最近的异形态位置

        向后优先：往后换对已确定的首屏体验影响更小
        """
        n = len(layout)
        for d in range(1, n):
            for j in (from_index + d, from_index - d):
                if 0 <= j < n and layout[j] is not None and layout[j].form != bad_form:
                    return j
        return None
```

**`i = max(0, min(i, donor - max_consecutive + 1))` 这一行回退是必需的，也是最容易写错的地方。** 交换把一条异形态内容移到了 `tgt`，同时把一条 `bad_form` 内容移到了 `donor`——**后者可能在 `donor` 附近制造出一个新的连续块**。如果只是 `i += 1` 继续向前扫，新产生的违规（在已扫过的位置）就永远不会被发现。**判据：任何"就地修复"的扫描循环，都必须考虑修复动作是否会在已扫描区域引入新违规；如果会，扫描位置必须回退到受影响的最早点。** 这类 bug 在测试里极难暴露——它需要一个"交换后恰好形成新连续块"的特定布局，随机测试命中率不到 2%，必须专门构造用例。

**三处值得单独说明的设计：**

**（1）为什么下限约束要先填（`_fill_floors` 在 `_fill_greedy` 之前）？** 这是贪心算法里一个通用但容易搞错的顺序问题。如果先自由贪心填满 20 位，再发现"缺 1 条长视频"，此时只能**替换**——踢掉一条已选内容。而踢掉哪一条会连带影响多样性状态（踢掉的那条可能是某话题的唯一代表），需要回溯校验，复杂度和出错概率都陡增。**先填下限的本质是：把"必须满足的约束"变成初始条件，而不是事后修补的目标。**

**（2）为什么降级顺序是配置而不是代码？** `DEGRADE_ORDER` 编码的是"哪些约束更重要"，这是**产品决策**。话题多样性放第一个放弃，是因为用户几乎感知不到"这 20 条里有 4 条同话题"；而首屏置信度放最后，是因为首屏体验直接决定跳出率。这个排序会随产品阶段变化（拉新期首屏更重要，成熟期多样性更重要），**必须可配置、可灰度、可审计**——每次降级都写 trace 并打点，才能回答"上周三下午为什么大量用户的 Feed 里没有冷启动内容"。

**（3）为什么形态配额下限不可放弃，宁可返回不足 20 条？** 这是本篇最重要的一个取舍。返回 18 条内容会让该次请求的曝光量下降 10%，是个可测量的短期损失。但放弃 `form_quota_min` 会打开挑战 2 的退化环——**而退化环的损失是不可逆且不可测量的**（创作者流失不会出现在任何在线指标里）。**在"可测量的小损失"与"不可测量的不可逆损失"之间，必须选前者。** 工程直觉恰恰相反，因为可测量的损失会立刻出现在报表上、被追问，而不可测量的损失没有人为它负责。

**位置亲和度与首屏兜底：`_assign_positions` 的两个补充实现**

匈牙利匹配的收益矩阵里有一项 `_form_pos_affinity`，它是"形态 × 位置"的适配度。这一项的存在，是**位置感知重排相对纯分数排序的全部价值来源**——没有它，加权匹配的结果与按分数降序排列完全等价（因为 `gain[i][j] = score_i × w_j` 时，最优匹配必然是分数最高的配权重最高的）。

```python
    # 形态 × 位置段 的完播率相对值（离线统计，每周重算）
    # 行=形态，列=位置段 [0-2 首屏, 3-7 次屏, 8-13 中段, 14-19 尾段]
    # 数值 = 该形态在该位置段的完播率 / 该形态的全局完播率
    FORM_POS_AFFINITY = {
        "SHORT":   (1.06, 1.02, 0.98, 0.94),   # 平坦：短视频对位置不敏感
        "ARTICLE": (0.92, 1.01, 1.04, 1.03),   # 首屏偏弱：用户刚进来不想读长文
        "LONG":    (0.81, 1.09, 1.07, 0.96),   # ★ 首屏最差、次屏最好
        "LIVE":    (1.14, 1.05, 0.72, 0.55),   # 陡降：直播的价值高度依赖靠前曝光
    }
    POS_SEGMENT_BOUNDS = (3, 8, 14)            # 位置段边界

    def _form_pos_affinity(self, form, slot_index, user_ctx):
        """形态在特定位置的适配度

        LONG 在首屏只有 0.81、在次屏有 1.09 —— 差 34.6%。
        这解释了为什么"把最高分内容放第 1 位"是错的：
        一条长视频分数最高，但放第 1 位它自己的完播率就掉 19%，
        还挤掉了本该占首屏的短视频（首屏亲和 1.06）。
        """
        seg = bisect.bisect_right(self.POS_SEGMENT_BOUNDS, slot_index)
        base = self.FORM_POS_AFFINITY[form][seg]

        # 冷启动内容前置：L0 内容放靠后位置会拿不到有效曝光
        # （尾段位置权重 0.21，119 次曝光预算会被浪费掉大半）
        if getattr(self, "_is_cold", None) and slot_index >= self.POS_SEGMENT_BOUNDS[1]:
            base *= 0.88

        # 弱网下把长视频往后放：给预加载留时间
        if user_ctx.net_type in ("2G", "3G") and form == "LONG" and slot_index < 3:
            base *= 0.70
        return base

    def _ensure_first_screen_confidence(self, layout):
        """首屏置信度兜底：首屏至少 1 条高置信内容

        为什么需要这一步：探索性内容（冷启动、Thompson 采样采到高值的
        长尾内容）的预估方差极大。若首屏 3 条恰好全是探索内容，
        本次请求的跳出概率显著上升（实测 +4.7pp）。

        这是「探索必须做，但不能全押在首屏」—— 探索的成本应当
        分摊到用户注意力较不敏感的位置上。
        """
        first = [c for c in layout[:self.FIRST_SCREEN] if c is not None]
        if any(c.confidence >= self.FIRST_SCREEN_MIN_CONFIDENCE for c in first):
            return layout

        # 首屏全是低置信 —— 从后段找一条高置信内容换上来
        donor_idx = None
        for j in range(self.FIRST_SCREEN, len(layout)):
            c = layout[j]
            if c is not None and c.confidence >= self.FIRST_SCREEN_MIN_CONFIDENCE:
                donor_idx = j
                break
        if donor_idx is None:
            # 整个布局没有高置信内容 —— 候选集质量问题，不是混排能解决的
            self.metrics.incr("feed.mix.no_confident_candidate")
            return layout

        # 换掉首屏里置信度最低的那条（而不是分数最低的）
        victim_idx = min(
            (j for j in range(self.FIRST_SCREEN) if layout[j] is not None),
            key=lambda j: layout[j].confidence)
        layout[victim_idx], layout[donor_idx] = layout[donor_idx], layout[victim_idx]
        self.metrics.incr("feed.mix.first_screen_rescue")
        return layout
```

**`FORM_POS_AFFINITY["LONG"] = (0.81, 1.09, 1.07, 0.96)` 这一行是位置感知重排存在的理由。** 长视频在首屏的完播率只有其全局均值的 81%，在次屏却有 109%——**同一条内容，换个位置，价值差 34.6%**。原因是行为学的：用户刚打开 App 时处于"扫描模式"，倾向于快速划过；滑过两三条之后进入"投入模式"，才愿意接受长内容的时间承诺。

**而这个 34.6% 的差异是纯分数排序完全看不见的**——它不在候选的任何特征里，它是"候选 × 位置"的交互项。**判据：当某个决策的收益依赖"选择"与"安放位置"的交互时，把它拆成"先选后排"两个独立的贪心步骤会系统性地丢掉这部分收益；必须在第二步显式建模交互项。**

**`_ensure_first_screen_confidence` 换掉的是"置信度最低"而非"分数最低"的那条**，这个细节容易写反。首屏的问题是**方差过大**，不是分数过低——一条分数很高但方差极大的探索内容，正是要被换掉的对象。按分数换会把它留下（它分数高），问题依然存在。**用哪个维度筛选，必须与被解决的问题维度一致。**

### 标签成熟度矫正：外推 + 在线校准双轨

`MATURITY_MULTIPLIER["LONG"] = 3.23` 这个系数不能是硬编码常量——它会漂移（内容结构变化、播放器改版、追更行为演变都会改变成熟曲线）。需要一套自动维护它的机制。

```
方案对比：如何消除长视频 68% 的标签低估

方案 A：延迟到 T+3 训练
  ✓ 标签真实（92% 成熟度）
  ✗ 模型时效性损失 3 天 —— 冷启动内容全部失效，热点响应失效
  ✗ 无法用于直播（直播内容 3 天后已无价值）
  结论：不可行

方案 B：只对长视频延迟，短视频仍 T+1（双模型）
  ✓ 各形态标签都真实
  ✗ 两个模型的分数不可比 —— 回到挑战 1 的原点，且更糟（连量纲基准都不同）
  ✗ 特征平台需维护两套时间切片，成本翻倍
  结论：不可行

方案 C：T+1 标签 + 成熟度外推系数（选定）
  ✓ 模型时效性不损失
  ✓ 单模型，分数天然可比
  ✗ 外推系数本身可能错，且错了很难发现
  → 用 T+7 真值做在线校准，闭环修正（下方实现）

方案 D：直接预测最终值（把"72 小时后的总时长"作为回归目标）
  ✓ 概念最干净
  ✗ 训练样本必须等 72 小时 —— 退化为方案 A
  ✗ 目标方差极大，模型难收敛
  结论：与方案 A 同构，不可行
```

**选择：方案 C（T+1 外推 + T+7 校准）**，理由：

1. **它是唯一不牺牲模型时效性的方案**。而时效性对内容平台是刚性需求——冷启动与热点响应都依赖它。
2. **外推是一个比预测更简单的问题**。预测"这条内容最终会被看多久"很难；估计"这类内容 24 小时的观看量占最终值的百分之几"是一个稳定的统计量（分桶后周环比波动 < 4%）。**把难问题拆成"易预测的部分 × 稳定的统计系数"，是处理延迟反馈的通用手法。**
3. **它有闭环校准通道**。T+7 的真值总会到来，可以用它检验外推系数并自动修正——这是方案 A/B/D 都不具备的自纠错能力。

**但需要注意：**

- **外推系数必须按「形态 × 内容子类」分桶，不能用形态级均值**（详见陷阱 3）。剧集追更 3 天看完、纪录片 30 天看完，均值会同时高估长尾、低估爆款。
- **外推会放大预估误差**。乘以 3.23 意味着长视频的预估误差也放大 3.23 倍，方差变大。必须配合不确定性惩罚，否则高方差候选会靠"运气好的预估"挤进 Top 20。
- **新内容子类没有历史成熟曲线**，只能用父类系数，且必须标记为低置信、限制其曝光上限，直到积累够样本。

```python
class MaturityCalibrator:
    """标签成熟度外推系数的维护与在线校准

    双轨：
      轨 1（外推）—— 用历史成熟曲线，把 T+1 部分标签外推到最终值，供训练用
      轨 2（校准）—— 用 T+7 真值反查外推准确度，自动修正系数
    """

    HORIZON_HOURS = (24, 48, 72, 168)      # 观测窗口：1/2/3/7 天
    FINAL_HORIZON = 168                     # 视 T+7 为最终值
    MIN_SAMPLES_PER_BUCKET = 3000            # 分桶最小样本量，不足则回退父类
    MAX_MULTIPLIER = 6.0                     # 外推系数上限，防异常桶爆炸
    DRIFT_ALARM_THRESHOLD = 0.15             # 系数周环比漂移超 15% 则告警
    RECOMPUTE_CRON = "0 4 * * 1"             # 每周一 04:00 重算

    def recompute(self):
        """重算全部分桶的成熟度系数（每周一次）"""
        new_table, low_conf = {}, []

        for bucket in self.enumerate_buckets():          # (form, sub_type)
            # 1. 取 T-14 ~ T-7 曝光的内容（保证已有 7 天完整观测）
            rows = self.warehouse.query_maturity(bucket, days_ago=(14, 7))
            if len(rows) < self.MIN_SAMPLES_PER_BUCKET:
                low_conf.append(bucket)
                continue

            # 2. 成熟度 = 24h 累积时长 / 168h 累积时长
            #    用中位数而非均值：时长分布极度右偏，均值被爆款主导
            ratios = [r.dur_24h / r.dur_168h for r in rows if r.dur_168h > 0]
            maturity_24h = median(ratios)

            # 3. 外推系数 = 1 / 成熟度，并限幅
            mult = min(1.0 / max(maturity_24h, 1e-3), self.MAX_MULTIPLIER)

            # 4. 漂移检测：与上周系数比，超阈值则告警但仍采用新值
            #    告警而不阻断的理由：漂移可能是真实的行为变化（如播放器改版），
            #    阻断会让系数永久停留在旧值；但必须让人看到
            old = self.table.get(bucket)
            if old and abs(mult - old) / old > self.DRIFT_ALARM_THRESHOLD:
                self.alarm.fire("maturity_drift", bucket=bucket,
                                old=round(old, 3), new=round(mult, 3))

            new_table[bucket] = mult

        # 5. 低样本桶回退到父类（形态级）系数，并标记低置信
        for bucket in low_conf:
            form = bucket[0]
            new_table[bucket] = self.form_level_multiplier(form, new_table)
            self.low_confidence.add(bucket)      # 下游据此限制曝光上限

        self.publish(new_table)
        return new_table

    def online_check(self):
        """在线校准：用已到期的 T+7 真值检验外推是否准确（每日执行）

        这是整个方案的自纠错通道。没有它，外推系数错了不会有任何信号——
        因为训练与在线预估用的是同一个（错的）系数，指标看起来完全自洽。
        """
        report = []
        for bucket in self.enumerate_buckets():
            # 1. 取 8 天前曝光、已有完整 T+7 数据的样本
            rows = self.warehouse.query_maturity(bucket, days_ago=(9, 8))
            if len(rows) < self.MIN_SAMPLES_PER_BUCKET // 3:
                continue

            # 2. 用当前系数外推 24h 标签，与 168h 真值比
            mult = self.table.get(bucket, 1.0)
            predicted = [r.dur_24h * mult for r in rows]
            actual = [r.dur_168h for r in rows]

            # 3. 关键指标是「有偏无偏」，不是"误差大小"
            #    外推必然有误差（单条不可能准），但整体不能有系统性偏向
            bias = (sum(predicted) - sum(actual)) / sum(actual)
            report.append((bucket, bias, len(rows)))

            if abs(bias) > 0.10:
                self.alarm.fire("maturity_biased", bucket=bucket,
                                bias=round(bias, 3), samples=len(rows))
        return report
```

**`online_check` 里最关键的一句是"关键指标是有偏无偏，不是误差大小"。** 外推单条内容的最终时长必然不准（方差很大，无法避免）；但**整体的偏差必须接近零**——因为排序只关心相对大小，随机误差会被大量样本抵消，而系统性偏差会直接扭曲形态间的相对权重。**判据：评估一个用于排序的估计量时，看它的偏差（bias）而不是它的均方误差（MSE）。** 这两个指标经常指向不同的方案选择。

**没有 `online_check` 会发生什么？** 训练用外推系数 3.23，在线预估也用 3.23，模型学到的长视频价值与在线打分完全自洽——**所有离线与在线指标都正常，AUC 正常，校准曲线正常**。唯一能暴露系数错误的信号，是把外推值与真实的 T+7 值直接对比。**当预测与评估用了同一个错误假设时，任何自洽性检查都无法发现错误，只有引入外部真值才能。**

### 内容池与流量分配：120 次曝光能做什么判断

P00 定义了分级曝光池（把"发/不发"变成"发多少"）。本节解决分配问题：**每个池分多少流量，以及池间晋升/降级的判据。**

```
五级内容池与流量分配（日总曝光 320 亿）

  池等级          内容量        流量占比    日曝光       单条日均曝光
  ─────────────────────────────────────────────────────────────────
  L0 冷启动池     1,345 万/日    5.0%      16.0 亿        119
  L1 小流量池       210 万        8.0%      25.6 亿      1,219
  L2 中流量池        48 万       17.0%      54.4 亿     11,333
  L3 大流量池        11 万       34.0%     108.8 亿     98,909
  L4 精选池         1.4 万       31.0%      99.2 亿    708,571
  广告位             —           5.0%      16.0 亿        —
  ─────┼───────────────────────────────────────────────────────────
  合计                          100.0%     320.0 亿
```

**这张表读出的第一件事：单条曝光量从 L0 到 L4 放大了 5,954 倍。** 池等级不是标签，是**指数级的流量杠杆**——这解释了为什么池晋升判据的正确性如此关键，也解释了为什么它是刷量攻击的首要目标。

**第二件事回到挑战 4：L0 的单条曝光量只有 119 次，而统计显著需要 1,568 次。** 所以 L0 → L1 的晋升**不可能是精细的质量排序**。必须重新定义 L0 的职责：

```
L0 冷启动池的职责重定义

  ✗ 不是"评估内容质量并排序"      —— 119 次样本做不到，置信区间 ±8.9pp
  ✓ 是"以 119 次为代价排除明显有问题的内容"

  119 次曝光能可靠检出的差异（α=0.001, power=0.9）：
    播放率 50% vs 15%   → 差 35pp，n=48 即可检出       ✓ 可靠
    播放率 50% vs 35%   → 差 15pp，n=260 才够          ✗ 不可靠
    播放率 50% vs 45%   → 差 5pp，n=1568 才够          ✗ 完全不可靠

  → L0 的唯一硬判据：播放率 < 15% 或负反馈率 > 8% 则淘汰
  → 其余内容的晋升排序，必须由先验主导
```

先验从哪里来？三个来源，加权融合成一个经验贝叶斯的先验分布：

```python
class PoolPromotionJudge:
    """内容池晋升判据：经验贝叶斯 + Thompson 采样

    核心认知：L0 的 119 次曝光只够做"排除"，不够做"排序"。
    排序必须由先验主导，后验只做修正。
    """

    # L0 的硬淘汰线：这两个差异足够大，119 次样本可靠检出
    L0_KILL_PLAY_RATE   = 0.15
    L0_KILL_NEGATIVE    = 0.08

    # 先验强度（等效曝光次数）：先验相当于多少次观测
    # 设为 480 意味着：L0 的 119 次后验只占 119/(119+480) = 20% 的权重
    # 这个数字是刻意设大的 —— 它诚实地反映了 119 次样本的信息量之低
    PRIOR_STRENGTH = 480

    # 三类先验的权重，由离线回归拟合（对最终 L3 表现的解释力占比）
    W_CREATOR   = 0.52    # 创作者历史表现 —— 解释力最强
    W_CONTENT   = 0.31    # 内容理解特征（多模态模型打分）
    W_SIMILAR   = 0.17    # 相似内容表现（同话题/同题材近期均值）

    EXPLORE_RATIO = 0.18       # L1/L2 中留给探索的流量比例
    MIN_L0_IMPRESSIONS = 100   # 未达此曝光量不做晋升判断（约束要求 24h ≥ 100）

    def judge_l0(self, content, stat):
        """L0 → L1 判定"""
        # 1. 曝光量不足则继续观察，不做判断
        if stat.impressions < self.MIN_L0_IMPRESSIONS:
            return "OBSERVING"

        # 2. 硬淘汰：这是 119 次样本唯一能可靠支撑的结论
        if stat.play_rate < self.L0_KILL_PLAY_RATE:
            return "KILLED"
        if stat.negative_rate > self.L0_KILL_NEGATIVE:
            return "KILLED"

        # 3. 经验贝叶斯：先验 + 后验融合
        prior_mean = self._prior_mean(content)
        n = stat.impressions
        alpha = prior_mean * self.PRIOR_STRENGTH + stat.plays
        beta  = (1 - prior_mean) * self.PRIOR_STRENGTH + (n - stat.plays)
        posterior_mean = alpha / (alpha + beta)

        # 4. Thompson 采样决定晋升：从后验分布采样而非用点估计
        #    这让"先验高但后验一般"的内容仍有机会，也让"先验低但后验惊艳"
        #    的内容能突围 —— 探索与利用的自然平衡
        sampled = beta_sample(alpha, beta)
        threshold = self.dynamic_threshold("L0_TO_L1")   # 由 L1 容量反推的分位阈值

        if sampled >= threshold:
            return "PROMOTE"
        # 5. 不晋升但也不淘汰的内容进"长尾池"，靠搜索/关注流获得零散曝光
        return "TAIL"

    def _prior_mean(self, content):
        """三源先验融合"""
        # 1. 创作者历史：该创作者近 90 天同形态内容的播放率（收缩到全站均值）
        creator = self.creator_stats.shrunk_play_rate(
            content.author_id, content.form, shrink_to=self.global_mean)
        # 2. 内容理解：多模态模型对内容本身的质量打分（不依赖任何曝光数据）
        content_score = self.content_understanding.quality_prior(content.content_id)
        # 3. 相似内容：同话题 + 同题材近 7 天新内容的播放率均值
        similar = self.similar_stats.play_rate(content.topic_id, content.genre_id)

        return (self.W_CREATOR * creator +
                self.W_CONTENT * content_score +
                self.W_SIMILAR * similar)

    def dynamic_threshold(self, transition):
        """晋升阈值由下一级池的容量反推，而非固定值

        L1 容量 210 万，日新增 1,345 万，则 L0→L1 晋升率约 15.6%
        → 阈值取采样值的 84.4 分位

        为什么阈值必须动态：内容供给量是波动的（节假日、活动期可能翻倍）。
        固定阈值会导致池容量随供给波动 —— 供给翻倍时 L1 塞进两倍内容，
        每条的流量腰斩，整个池的评估质量崩塌。
        """
        capacity = self.pool_capacity[transition]
        inflow = self.inflow_estimator.predict(transition)     # 预测未来 1 小时流入
        rate = min(capacity / max(inflow, 1), 1.0)
        return self.quantile_cache.get(transition, 1.0 - rate)
```

**`PRIOR_STRENGTH = 480` 这个数字是整个类里最诚实的一处设计。** 它明确地说：L0 那 119 次曝光只值 20% 的权重，剩下 80% 来自先验。工程师的本能会把这个值设小（"我们要相信真实数据，不要相信模型先验"），但那恰恰是错的——**119 次观测的信息量客观上就很低，把它当成主导信号是把噪声当信号。** 判据：**先验强度应当由后验样本量决定（设为"后验样本量的 4 倍"是一个稳健起点），而不是由对先验的信心决定。**

**为什么用 Thompson 采样而不是直接按后验均值排序？** 按均值排序会让先验高的内容（大 V、热门题材）永久占据晋升名额，新人内容永无出头之日——**先验主导的排序会固化先验**。Thompson 采样从后验分布采样，方差大的内容（新人、小众题材，先验不确定）有更大概率被采到高值，从而获得晋升机会。**这是在"先验必须主导"与"先验不能固化"之间的唯一解法：让不确定性本身成为探索的驱动力。**

**`dynamic_threshold` 里那段注释指出的问题很容易被忽略：** 晋升阈值写成固定值（如"播放率 > 42% 则晋升"）时，一旦内容供给翻倍（节假日、平台活动），晋升的绝对条数也翻倍，L1 池被塞进两倍内容。但 L1 的流量占比是固定的 8%，于是**每条内容的曝光量腰斩**——从 1,219 次降到 610 次，L1 的评估置信度随之崩塌，连带 L1→L2 的判断全部失准。**容量固定的池，其入口阈值必须由容量反推，不能是绝对值。**

### 内容检索：可检索文本的构造是召回率的真正上限

搜索链路的架构（query 理解 → 多路召回 → 融合排序）是成熟工程。本节只讲统一平台特有的两个问题：**视频的可检索文本从哪来**，以及**跨形态结果如何排在一个列表里**。

先量化第一个问题。视频的召回率上限完全由可检索文本的覆盖决定：

```
视频召回率随可检索文本来源的累积提升（实测，Recall@50，人工标注 1.2 万 query）

  仅标题 + 简介（84 字）                          0.34   ├────────┤
  + ASR 语音转写（长视频约 1,200 字）              0.79   ├───────────────────┤
  + OCR 画面文字与硬字幕（约 180 字）              0.84   ├────────────────────┤
  + 视觉标签（物体/场景/人脸，约 40 个标签）        0.87   ├─────────────────────┤
  + 用户行为文本（评论/弹幕高频词，约 300 字）      0.91   ├──────────────────────┤
  ─────────────────────────────────────────────────────────────────────────
  图文基线（标题 + 正文 1,828 字）                 0.94   ├───────────────────────┤
```

**ASR 一项就把召回率从 0.34 拉到 0.79（+132%）——它是视频搜索里投入产出比最高的单项，没有任何检索引擎的优化能接近这个量级。** 这与工程直觉相反：搜索团队的注意力通常放在召回策略、排序模型、向量索引上，而真正的瓶颈在**内容理解侧的文本供给**。

而它的成本低得出乎意料：

| 内容理解任务 | 日处理量 | 单位算力 | 日算力 | 常驻卡数 | 月成本 |
|-------------|---------|---------|-------|---------|--------|
| ASR 语音转写 | 3.45 亿秒音频 | RTF 0.02 | 690 万 GPU·秒 | 80 | ¥64 万 |
| OCR 画面文字 | 2,020 万关键帧 | 0.05 GPU·秒/帧 | 101 万 GPU·秒 | 12 | ¥9.6 万 |
| 视觉 Embedding | 2,020 万关键帧 | 0.02 GPU·秒/帧 | 40 万 GPU·秒 | 5 | ¥4.0 万 |
| **合计** | — | — | — | **97** | **¥77.6 万** |

**¥77.6 万/月 占平台总成本（¥1.17 亿）的 0.66%，换来视频召回率 +156%（0.34 → 0.87）。** 这是全平台性价比最高的投入之一，而它容易被推迟——因为它的收益体现在"搜索能搜到了"，而不体现在任何单一 KPI 上；且它属于内容理解域，预算和搜索域的 KPI 不在同一个部门。**跨域的高收益投入，最容易因为组织边界而被搁置——这是康威定律的成本面。**

**可检索文档的构造：为什么不能把这些文本直接拼起来**

拿到 ASR、OCR、视觉标签、行为文本之后，最直觉的做法是全部拼进一个大字段送进倒排索引。**这会让召回率上升，但让精度崩塌**，原因在 BM25 的长度归一化：

```
拼接前后的字段长度与 BM25 影响

  图文：标题 32 字 + 正文 1,828 字                     → doc_len ≈ 1,860
  短视频（拼接前）：标题 26 字 + 简介 58 字             → doc_len ≈ 84
  短视频（拼接后）：+ ASR 340 + OCR 180 + 标签 40
                    + 弹幕高频词 300                    → doc_len ≈ 944  （×11.2）

  BM25 的 tf 归一化项：tf / (tf + k₁·(1 − b + b·doc_len/avg_len))
  取 k₁=1.2, b=0.75，avg_len 从 84 涨到 944 时：

    标题命中一次（tf=1）的得分贡献
      拼接前   1 / (1 + 1.2 × (0.25 + 0.75 × 84/84))   = 0.455
      拼接后   1 / (1 + 1.2 × (0.25 + 0.75 × 944/944)) = 0.455   ← 看似不变

    但「标题命中」与「ASR 命中」现在不可区分了：
      用户搜「红烧肉」
        候选 A：标题《红烧肉的三种做法》，ASR 里也提到 4 次   → tf=5
        候选 B：标题《我的一天》，ASR 里主播随口说了 5 次红烧肉 → tf=5
      → BM25 得分相同，而 B 完全不该被召回
```

**根因：拼接把"这个词出现在哪个字段"这一信息彻底丢掉了，而字段本身携带了强相关性信号**——标题里的词是作者的主动声明，ASR 里的词可能只是随口一提。解法是多字段索引（BM25F）+ 差异化字段权重：

```python
class SearchableDocBuilder:
    """可检索文档构造：多字段 + 差异化权重 + 噪声清洗

    输出送入 ES 的 multi_field 文档，而不是单个拼接大字段。
    字段权重的量级差异（标题 12.0 vs 弹幕 0.8，差 15 倍）
    是精度的主要来源。
    """

    # 字段权重：由「该字段命中时的人工相关性标注均值」拟合
    # 15 倍的跨度不是拍的 —— 标题命中的相关率 0.81，弹幕命中只有 0.09
    FIELD_WEIGHTS = {
        "title":        12.0,   # 作者主动声明，最强信号
        "title_pinyin":  6.0,   # 拼音容错，权重减半（易误召回）
        "summary":       4.5,   # 简介
        "topic_tags":    4.0,   # 结构化话题标签
        "ocr_subtitle":  3.2,   # 硬字幕：视频的"正文"，密度高且与画面对齐
        "asr_text":      2.4,   # 语音转写：召回贡献最大，但精度最低
        "visual_tags":   2.0,   # 视觉标签（物体/场景）
        "ocr_screen":    1.6,   # 非字幕的画面文字（水印/弹窗/背景招牌，噪声多）
        "author_name":   3.0,
        "behavior_text": 0.8,   # 评论/弹幕高频词：只用于兜底召回
    }

    ASR_MIN_CONFIDENCE = 0.62      # ASR 分句置信度下限，低于此丢弃
    OCR_MIN_CONFIDENCE = 0.70      # OCR 置信度下限（比 ASR 高：OCR 错字更伤）
    OCR_MIN_AREA_RATIO = 0.004     # 文字框面积占比下限，滤掉水印/台标
    BEHAVIOR_MIN_DF = 8            # 弹幕词最小出现次数，滤掉个人化表达
    BEHAVIOR_TOP_K = 60            # 行为文本只保留 Top 60 高频词
    ASR_MAX_CHARS = 4000           # ASR 截断上限（长视频可达 3 万字）

    def build(self, content, understanding):
        """content: 内容元信息  understanding: 内容理解域的产出"""
        doc = {
            "content_id":   content.content_id,
            "form":         content.form,
            "title":        self._normalize(content.title),
            "title_pinyin": to_pinyin(content.title),
            "summary":      self._normalize(content.summary or ""),
            "topic_tags":   " ".join(content.topic_tags),
            "author_name":  content.author_name,
            # 结构化过滤字段（不参与打分，只做 filter）
            "pool_level":   content.pool_level,
            "publish_at":   content.publish_at,
            "duration_sec": content.duration_sec,
        }
        if content.form == "ARTICLE":
            doc["body"] = self._normalize(content.body)
            return doc

        # ---- 视频/直播：从内容理解产出构造可检索文本 ----
        doc["asr_text"]      = self._build_asr(understanding.asr_segments)
        doc["ocr_subtitle"], doc["ocr_screen"] = self._split_ocr(understanding.ocr_boxes)
        doc["visual_tags"]   = self._build_visual_tags(understanding.visual_labels)
        doc["behavior_text"] = self._build_behavior(content.content_id)
        # 帧级时间戳单独存，供检索命中后做跳转定位（见陷阱 7）
        doc["frame_offsets"] = understanding.keyframe_offsets
        return doc

    def _build_asr(self, segments):
        """ASR 清洗：置信度过滤 + 口语噪声去除 + 连续重复合并

        原始 ASR 有三类噪声，不处理会显著拉低精度：
          1. 低置信分句（背景音乐段落常被识别成随机词）
          2. 口语填充词（"那个"、"就是说"、"然后呢"）—— 占长视频 ASR 的 11%
          3. 连续重复（"好好好好好"，直播里尤其多）
        """
        kept = []
        for seg in segments:
            if seg.confidence < self.ASR_MIN_CONFIDENCE:
                continue
            text = FILLER_RE.sub("", seg.text)          # 去口语填充词
            text = REPEAT_RE.sub(r"\1", text)           # "好好好好" → "好"
            text = text.strip()
            if len(text) >= 2:
                kept.append(text)

        joined = " ".join(kept)
        if len(joined) <= self.ASR_MAX_CHARS:
            return joined

        # 超长截断：不能简单取前 4000 字（视频开头常是片头/自我介绍）
        # 按分句 TF-IDF 权重取信息量最高的片段，保留原始时序
        return self._extractive_truncate(kept, self.ASR_MAX_CHARS)

    def _split_ocr(self, boxes):
        """OCR 必须拆成「硬字幕」与「画面文字」两个字段

        权重差 2 倍（3.2 vs 1.6），因为两者的相关性完全不同：
          硬字幕 —— 是内容的台词，等价于正文
          画面文字 —— 水印、台标、路牌、弹窗广告，大部分是噪声

        判据：位于画面下部 1/4、跨帧位置稳定、且横向居中的文字框是字幕。
        """
        subtitle, screen = [], []
        for b in boxes:
            if b.confidence < self.OCR_MIN_CONFIDENCE:
                continue
            if b.area_ratio < self.OCR_MIN_AREA_RATIO:   # 滤掉台标/水印
                continue
            is_subtitle = (b.center_y > 0.75 and
                           abs(b.center_x - 0.5) < 0.22 and
                           b.cross_frame_stable)
            (subtitle if is_subtitle else screen).append(b.text)

        # 字幕去重：相邻帧的同一句字幕会被反复识别
        return " ".join(dedup_consecutive(subtitle)), " ".join(set(screen))

    def _build_behavior(self, content_id):
        """行为文本：从评论/弹幕提取高频词

        这是唯一「随时间变化」的可检索字段 —— 它需要增量重建。
        重建策略：内容发布后 24h / 72h / 7d 各重建一次，之后每月一次。
        为什么不实时：倒排索引的更新成本远高于收益，且行为文本
        权重只有 0.8，抖动对结果影响很小。
        """
        words = self.behavior_store.top_words(
            content_id, min_df=self.BEHAVIOR_MIN_DF, top_k=self.BEHAVIOR_TOP_K)
        # 过滤纯情绪词（"哈哈哈"、"泪目"）—— 它们不携带内容信息
        return " ".join(w for w in words if w not in EMOTION_STOPWORDS)
```

**`_split_ocr` 把 OCR 拆成两个字段，是这个类里收益最集中的一处。** 未拆分时，视频搜索会被水印和台标严重污染——一个带"XX 电视台"水印的视频，搜"XX 电视台"会命中全部该来源的内容，而这几乎从不是用户意图。拆分后画面文字权重降到 1.6，且**用了三个几何 + 时序特征来判定字幕**（下部 1/4、横向居中、跨帧稳定）。这三个条件都很朴素，但组合起来的字幕识别准确率是 0.94，而任何单一条件都不到 0.7。

**`_build_behavior` 的注释里藏着一条重要的工程判断：行为文本是唯一会随时间变化的可检索字段，但它的权重只有 0.8，所以不值得实时更新。** 这个推理链条值得显式化——**字段的更新频率应当由「权重 × 变化率」共同决定，而不是只看变化率。** 一个高频变化但低权重的字段，实时更新是纯粹的成本浪费；工程上常见的错误是"因为它在变，所以要实时同步"。

**向量索引的规模与压缩取舍：**

```
向量索引容量测算

  向量数：40 亿内容 × 4 个关键帧 = 160 亿条，768 维
  ─────────────────────────────────────────────────────────
  原始 fp16          768 × 2 B = 1,536 B/条 → 24.6 TB
  + HNSW 图结构      × 1.6                  → 39.4 TB   ✗ 放不进内存
  ─────────────────────────────────────────────────────────
  IVF-PQ 压缩        96 B/条（m=96, 8bit）  → 1.54 TB   ✓ 24 台 × 64 GB
  ─────────────────────────────────────────────────────────
  代价：Recall@100 从 0.98 降到 0.91（压缩损失 7 个百分点）
```

7 个百分点的召回损失能不能补回来？可以，用**两阶段 ANN**：

```python
class MultiModalRetriever:
    """多模态向量检索：IVF-PQ 粗筛 + 原始向量精排

    单机内存放不下 39 TB 的 HNSW，只能 PQ 压缩到 1.54 TB；
    但 PQ 损失 7pp 召回，用小规模精排补回来。
    """

    NPROBE = 48                  # IVF 探测的聚类桶数，召回与延迟的主旋钮
    COARSE_TOPK = 500            # PQ 粗筛取回条数
    RERANK_TOPK = 100            # 用原始向量精排的条数（受 SSD 随机读延迟约束）
    VECTOR_BYTES = 1536          # 原始向量字节数（768 维 fp16）
    SSD_RANDOM_READ_MS = 0.10    # 单次 SSD 随机读延迟

    def search(self, query_vec, form_filter=None, top_k=50):
        # 1. PQ 粗筛：在压缩向量上算近似距离，取 Top 500
        #    这一步在内存里完成，24 台机器并行，约 8 ms
        coarse = self.pq_index.search(query_vec, k=self.COARSE_TOPK,
                                      nprobe=self.NPROBE, filt=form_filter)

        # 2. 精排：只对 Top 100 从 SSD 读原始向量重算精确距离
        #    为什么只取 100 而不是 500：500 次随机读 = 50 ms，超预算；
        #    100 次 = 10 ms，可接受。Recall@100 从 0.91 回到 0.948
        head = coarse[:self.RERANK_TOPK]
        exact_vecs = self.vector_store.batch_get(
            [c.vec_id for c in head])              # 100 次随机读，约 10 ms
        for c, v in zip(head, exact_vecs):
            c.dist = exact_distance(query_vec, v)
        head.sort(key=lambda c: c.dist)

        # 3. 尾部（101-500）保留 PQ 的近似距离，但打上低置信标记
        #    融合排序时对低置信候选施加折扣，避免近似误差污染头部
        tail = coarse[self.RERANK_TOPK:]
        for c in tail:
            c.approx = True

        # 4. 帧级结果聚合到内容级：一条内容有 4 个关键帧，取最优帧
        #    注意去重必须在聚合后做 —— 否则同一内容的 4 帧会占 4 个坑
        return self._aggregate_to_content(head + tail, top_k)

    def _aggregate_to_content(self, frame_hits, top_k):
        """帧级 → 内容级聚合

        一条视频的 4 个关键帧可能都命中，需要聚合。
        聚合函数用 max 而不是 mean：
          用户搜"爆炸场面"，一条 40 分钟电影里只有 1 帧是爆炸，
          mean 会把它稀释到排不上，max 才是对的语义。
        """
        best = {}
        for h in frame_hits:
            cur = best.get(h.content_id)
            if cur is None or h.dist < cur.dist:
                best[h.content_id] = h
        out = sorted(best.values(), key=lambda h: h.dist)
        return out[:top_k]
```

**`_aggregate_to_content` 用 max 而不是 mean，这个选择决定了视频搜索的语义。** 用 mean 意味着"整条内容整体上有多像 query"，用 max 意味着"内容里存在一个片段很像 query"。对视频搜索，**后者才是用户要的**——搜"爆炸场面"的人想要那 3 秒，不在乎另外 39 分 57 秒是什么。用 mean 会让所有长视频的相似度被稀释，系统性地偏向短视频。**聚合函数的选择不是实现细节，它定义了检索的语义，且会引入形态偏向。**

**跨形态结果排序：query 条件下的形态权重**

搜索的排序面对的是混排的同一个问题，但多了 query 这个条件。解法也同构——**只是影子价格换成了 query 意图相关的形态权重**：

```python
class SearchFormWeighting:
    """query 条件下的形态权重：搜索版的"影子价格"

    与 Feed 混排的区别：
      Feed  —— 形态权重由全站配额目标决定（供给侧驱动）
      搜索  —— 形态权重由 query 意图决定（需求侧驱动）
    """

    # 意图 → 形态偏好的先验，从历史点击日志的形态分布学习
    # 每周重算，冷启动 query 用意图类别的均值
    DEFAULT_PRIOR = {"ARTICLE": 0.25, "SHORT": 0.35, "LONG": 0.25, "LIVE": 0.15}

    LIVE_FRESHNESS_BOOST = 2.4     # 正在开播的直播加权（时效性溢价）
    LIVE_MIN_VIEWERS = 50          # 低于此在线人数的直播不参与搜索（防空房）
    MIN_CLICK_SAMPLES = 200        # query 级形态分布的最小样本量

    def weights_for(self, query, query_intent):
        # 1. 高频 query 用自身的历史形态点击分布（最准）
        if self.click_stats.samples(query) >= self.MIN_CLICK_SAMPLES:
            w = self.click_stats.form_distribution(query)
        # 2. 低频 query 回退到意图类别的分布
        #    意图类别由 query 理解模块给出：人物/影视剧/知识/商品/事件…
        else:
            w = self.intent_stats.form_distribution(query_intent) or self.DEFAULT_PRIOR

        # 3. 时效性意图（"…直播"、"…在线"、正在发生的事件）额外抬升直播
        if query_intent.is_realtime or self.event_detector.is_hot_now(query):
            w = dict(w)
            w["LIVE"] = w.get("LIVE", 0.15) * self.LIVE_FRESHNESS_BOOST
            w = self._normalize(w)
        return w
```

**多路召回的融合：为什么搜索不能照搬 Feed 的加权求和**

有了形态权重，还需要把多路召回的结果融合成一个列表。这里与 Feed 混排有一个**结构性差异**，照搬会出严重问题：

```
Feed 混排：  V(c) = EEV(c) × 形态权重 × 场景系数        —— 全部项加权求和/相乘
搜索排序：   ？

朴素照搬：  score = w_rel · 相关性 + w_eev · EEV × 形态权重
  ✗ 致命缺陷：高 EEV 可以补偿低相关性

  用户搜「Python 装饰器教程」
    候选 A：相关性 0.95，EEV 40 秒（冷门但精准的技术讲解）
    候选 B：相关性 0.31，EEV 480 秒（爆款搞笑视频，标题含"Python"）
    取 w_rel=100, w_eev=0.5：
      A = 100×0.95 + 0.5×40  =  115
      B = 100×0.31 + 0.5×480 =  271   ← B 排第一

  用户搜技术教程，返回搞笑视频 —— 这不是"排序不够好"，是搜索坏了。
```

**根因：相关性与质量在搜索里不是同一个量纲的两个加数，而是「门控」与「排序」两个不同角色。** Feed 里用户没有明确意图，所以所有信号都可以互相补偿；搜索里用户给出了明确意图，**相关性是一个必须满足的前置条件，不可被任何其他信号补偿**：

```python
class CrossFormFusion:
    """多路召回融合 + 跨形态排序

    两阶段结构（关键）：
      阶段 1  相关性门控 —— 硬阈值，不可被质量补偿
      阶段 2  门内排序   —— 相关性 × 质量 × 形态权重
    """

    RRF_K = 60                    # RRF 平滑常数，业界标准值
    REL_GATE_STRICT = 0.62        # 高频 query 的相关性门槛（有充足点击反馈校准）
    REL_GATE_LOOSE  = 0.45        # 长尾 query 的门槛（相关性模型本身不确定，放宽）
    REL_GATE_MIN_RESULTS = 8      # 门控后不足 8 条则逐步降低门槛（宁可放宽也不能空结果）
    QUALITY_EXPONENT = 0.35       # 质量项的指数压缩，见下方说明

    # 各召回通道的 RRF 权重（不是分数权重，是排名权重）
    CHANNEL_WEIGHTS = {
        "bm25f":        1.00,     # 多字段文本精确匹配
        "vector_text":  0.85,     # 文本语义向量
        "vector_multi": 0.70,     # 多模态向量（画面/音频）
        "tag_exact":    0.60,     # 结构化标签精确匹配
        "personalized": 0.45,     # 个性化召回（用户历史相似）
    }

    def fuse(self, channel_results, query, query_intent, user_ctx):
        # ---- 阶段 0：RRF 融合多路排名 ----
        # 为什么用 RRF（Reciprocal Rank Fusion）而不是分数归一化：
        #   BM25 分数域是 [0, ~40] 且随 query 词数漂移；
        #   余弦相似度域是 [-1, 1]；标签匹配是布尔。
        #   逐 query 做 min-max 归一化在「某通道只返回 3 条」时极不稳定
        #   （3 条的极差不能代表分布）。RRF 只用排名，天然免疫量纲问题。
        fused = defaultdict(float)
        for channel, hits in channel_results.items():
            cw = self.CHANNEL_WEIGHTS.get(channel, 0.5)
            for rank, h in enumerate(hits, start=1):
                fused[h.content_id] += cw / (self.RRF_K + rank)

        cands = self.hydrate(fused.keys())          # 批量补齐元信息与预估

        # ---- 阶段 1：相关性门控（硬阈值，不可补偿）----
        gate = (self.REL_GATE_STRICT
                if self.click_stats.samples(query) >= 200
                else self.REL_GATE_LOOSE)
        passed = [c for c in cands if c.relevance >= gate]

        # 门控后结果太少：逐级放宽，但每次放宽都记录
        # 空结果比不相关结果更糟 —— 前者用户直接流失，后者至少有挽回机会
        while len(passed) < self.REL_GATE_MIN_RESULTS and gate > 0.20:
            gate -= 0.08
            passed = [c for c in cands if c.relevance >= gate]
            self.metrics.incr("search.gate_relaxed")

        # ---- 阶段 2：门内排序 ----
        form_w = self.form_weighting.weights_for(query, query_intent)
        for c in passed:
            # 相关性仍参与排序（门控只是下限，不是把相关性丢掉）
            rel = c.relevance
            # 质量项做指数压缩：EEV 在形态间差 20 倍，不压缩会让
            # 长视频靠时长碾压全部短内容 —— 而搜索场景下用户
            # 要的是「最匹配的」，不是「最长的」
            quality = (c.eev ** self.QUALITY_EXPONENT)
            # RRF 分数作为多通道共识度的信号：多个通道都召回的内容更可信
            consensus = fused[c.content_id]

            c.final = (rel ** 1.6) * quality * form_w.get(c.form, 0.25) \
                      * (1.0 + 2.2 * consensus) * self.freshness(c, query_intent)

        passed.sort(key=lambda c: c.final, reverse=True)
        return self._diversify_forms(passed, form_w)

    def _diversify_forms(self, ranked, form_w):
        """形态配额的软化版本：滑窗内形态占比不得超过权重的 2 倍

        搜索的形态多样性约束比 Feed 弱得多 —— 如果 query 意图
        明确指向某个形态（搜"XX 直播"），就该全是直播。
        所以这里只做「防止单一形态完全垄断」的兜底，
        不做 Feed 那样的保量下限。
        """
        out, window, counts = [], deque(maxlen=10), defaultdict(int)
        deferred = []
        for c in ranked:
            cap = max(2, int(10 * form_w.get(c.form, 0.25) * 2.0))
            if counts[c.form] >= cap:
                deferred.append(c)                  # 暂缓，不丢弃
                continue
            out.append(c)
            if len(window) == window.maxlen:
                counts[window[0].form] -= 1
            window.append(c)
            counts[c.form] += 1
        return out + deferred
```

**`QUALITY_EXPONENT = 0.35` 这个指数压缩是搜索与 Feed 最实质的一处差异。** Feed 里 EEV 直接线性参与打分（我们**就是**要最大化消费时长）；搜索里如果也线性参与，长视频的 EEV（均值 480 秒）会碾压短视频（均值 24 秒）——20 倍的差距足以让任何相关性优势失效。取 0.35 次幂后，20 倍差距被压缩到 `20^0.35 = 2.9` 倍，与相关性项（`rel^1.6`，域内差异约 2 倍）处在同一量级。

**判据：当两个信号需要相乘但动态范围差一个数量级以上时，必须先做幂压缩把它们拉到可比的动态范围，否则动态范围大的那个信号会独占决策权——而这个失效是静默的，表现为"另一个信号的权重调了也没用"。** 调参时的典型症状是"我把相关性权重从 1 调到 10 都没变化"，此时问题不在权重而在动态范围。

**`deferred` 而不是丢弃，是 `_diversify_forms` 里的一个小而重要的选择。** 被形态配额挤掉的候选放到列表尾部而非丢掉——搜索结果有翻页，第 2 页仍需要内容。Feed 混排可以丢（只有 20 个位置），搜索不能丢（用户可能翻到第 5 页）。**同一个多样性约束，在"固定坑位"与"无限滚动"两种消费形态下的正确实现不同：前者是过滤，后者是重排。**

**这一节的收尾结论：混排与搜索面对的是同一个核心难题——跨形态可比性。** 区别只在权重的来源：Feed 的形态权重由**供给侧配额**驱动（全站需要 9% 的长视频曝光），搜索的形态权重由**需求侧意图**驱动（这个 query 的用户想看什么形态）。**两者共用同一套统一价值函数与同一套形态权重乘法机制，只替换权重的生成器**——这是把两个看似独立的域统一到一个抽象下的收益：一套价值函数、一套评估体系、一套 A/B 框架。

### 热榜：加速度、跨圈层扩散度与反刷榜

热榜的位置价值极高——榜单入口日 PV 约 8,000 万，Top 10 单条日均获得约 300 万次曝光，是 L4 精选池平均值的 4.2 倍。**任何流量杠杆达到这个量级的入口，都会成为操纵的首要目标。**

**热度定义：加速度主导，而不是绝对量。**

这里要与 P05 的 `HeatScorer` 做个明确区分——两者名字像、公式像，但**目标完全相反**：

| | P05 存储热度 | P09 榜单热度 |
|---|---|---|
| 目的 | 预测**未来**访问量，决定放哪一层 | 发现**正在**变热的内容，决定上不上榜 |
| 偏好 | 稳定的高访问（可靠地留在热层） | **不稳定的快速上涨**（新鲜度） |
| 对"持续高热"的态度 | 正是想要的 | 已经热了很久的不该再上榜（占位） |
| 错误代价 | 冷层被打满（性能问题） | 榜单被刷（信任问题） |

**同一个"热度"词在两个域里指向相反的偏好，这是跨域设计时最容易踩的语义陷阱**——如果两个团队复用了同一个"热度分"服务，其中一方必然拿到错的语义。

**反刷榜：绝对量指标全部可刷，结构性指标不可刷。**

```
刷量能轻易伪造的指标（全部不可用作榜单主判据）
  播放次数        买量即可
  点赞/收藏数     买量即可
  评论数          买量即可（且可生成合理文本）
  完播率          脚本控制播放时长即可，甚至比真实用户更"完美"
  播放增速        买量的时间分布可以任意编排

刷量难以伪造的结构性指标（榜单主判据）
  独立真实用户数            需要真实设备与真实账号历史
  跨圈层扩散度              ★ 最难伪造 —— 见下方定义
  自然搜索量的同步上涨      用户主动搜索该内容的名字，买量买不到
  外部平台的同步提及        跨平台的舆情同步，成本极高
  负反馈率的正常水位        刷量账号不会点"不感兴趣"，导致负反馈率异常偏低（反向信号）
```

**跨圈层扩散度是反刷榜的核心指标**，它的定义与为什么难伪造：

```
扩散度 = 内容触达的社交圈层数 ÷ 触达用户数

  社交圈层 = 关注关系图上的连通社区（Louvain 社区发现，离线每日更新）
  全站约 240 万个社区，中位社区规模 78 人

  真实爆款：10 万次播放来自 3.2 万个不同社区 → 扩散度 0.32
  刷量内容：10 万次播放来自   180 个社区 → 扩散度 0.0018

  差 178 倍 —— 这是所有反刷指标里区分度最大的一个
```

**为什么扩散度难伪造：** 伪造它需要控制大量**分属不同社交社区**的账号。而社交社区是由真实的关注关系形成的——批量注册的账号天然聚集在少数几个社区里（它们互相关注、被同一批人关注、行为模式相同，社区发现算法会把它们归到一起）。**要伪造扩散度，攻击者必须先伪造一个覆盖数万个真实社区的社交网络，其成本远超刷量收益。**

**这背后是一个通用的反作弊原则：可被单个参与者独立产生的指标必然可刷；只能由多个独立参与者的结构关系产生的指标难刷。** 播放次数是前者（一个账号刷一次），扩散度是后者（需要真实的社交结构）。设计反作弊指标时，应当优先寻找**结构性的、关系型的、需要多方共谋才能伪造的**信号。

```python
class HotRankScorer:
    """榜单热度打分：加速度主导 + 结构性反刷 + 冷却机制

    与 P05 的存储热度是不同语义 —— 这里偏好"正在快速上涨的新内容"，
    而 P05 偏好"稳定的高访问"。切勿复用。
    """

    W_ACCEL     = 0.44    # 加速度（增速的增速）—— 主导项，识别"正在起飞"
    W_VELOCITY  = 0.21    # 增速（单位时间增量）
    W_SPREAD    = 0.23    # 跨圈层扩散度 —— 反刷主判据
    W_EXTERNAL  = 0.12    # 外部信号（自然搜索量上涨 + 跨平台提及）

    SPREAD_MIN          = 0.06     # 扩散度硬门槛，低于此直接不予上榜
    NEGATIVE_RATE_MIN   = 0.004    # 负反馈率下限：过低反而可疑（刷量账号不点负反馈）
    NEGATIVE_RATE_MAX   = 0.055    # 负反馈率上限：过高说明内容有问题
    COOLDOWN_HOURS      = 48       # 同一内容上榜后的冷却期，防长期霸榜
    COOLDOWN_DECAY      = 0.35     # 冷却期内的分数折扣
    MAX_PER_AUTHOR      = 2        # 同一创作者榜上最多 2 条
    MIN_UNIQUE_USERS    = 20000    # 独立真实用户数下限

    def score(self, content, series):
        """series 是最近 6 小时、10 分钟粒度的时序统计"""
        # 1. 结构性硬门槛：不满足则直接淘汰，不进入打分
        #    这些是"一票否决"项，不参与加权 —— 加权会让高分掩盖异常
        if series.spread_ratio < self.SPREAD_MIN:
            return self._reject(content, "spread_too_low", series.spread_ratio)
        if series.unique_users < self.MIN_UNIQUE_USERS:
            return self._reject(content, "insufficient_unique_users")
        if not (self.NEGATIVE_RATE_MIN <= series.negative_rate <= self.NEGATIVE_RATE_MAX):
            return self._reject(content, "abnormal_negative_rate", series.negative_rate)

        # 2. 加速度：用二阶差分，对数化压缩量级差异
        #    对数化的理由：绝对增量在头部内容间差几个数量级，
        #    不压缩会让榜单永远被超大体量内容占据
        v = self._velocity(series)                  # 一阶：每 10 分钟增量
        a = self._acceleration(series)              # 二阶：增量的增量
        s = (self.W_ACCEL    * log1p_signed(a) +
             self.W_VELOCITY * log1p_signed(v) +
             self.W_SPREAD   * series.spread_ratio * 100 +
             self.W_EXTERNAL * log1p_signed(series.external_signal))

        # 3. 冷却：已上榜过的内容打折，给新内容让位
        #    没有这一步，榜单会退化为"长期热门列表"，失去发现价值
        hours_since = self.rank_history.hours_since_last_ranked(content.content_id)
        if hours_since is not None and hours_since < self.COOLDOWN_HOURS:
            s *= self.COOLDOWN_DECAY

        return s

    def _reject(self, content, reason, value=None):
        """拒绝上榜并记录 —— 审计与调参都依赖这条日志"""
        self.audit.log("hotrank_rejected", content_id=content.content_id,
                       reason=reason, value=value)
        self.metrics.incr("hotrank.reject", tags={"reason": reason})
        return float("-inf")
```

**加速度的信噪比问题：为什么不能用二阶差分**

`_acceleration` 的教科书实现是二阶差分 `x[t] − 2x[t−1] + x[t−2]`。**对计数型时序，这个实现基本不可用**——它的噪声会完全淹没信号：

```
设 10 分钟增量近似 Poisson(λ)，头部候选内容典型 λ = 5,000

  一阶差分噪声  std = √(2λ)  = √10,000  =  100
  二阶差分噪声  std = √(6λ)  = √30,000  =  173

  而"正在起飞"的真实加速度典型值约 200~600
  → 信噪比只有 1.2~3.5，且二阶差分放大噪声的系数（√6）
    比一阶（√2）大 1.73 倍

  后果：榜单在相邻两个 10 分钟周期之间剧烈抖动，
        实测 Top 20 的周期间变动率高达 41%（用户感知为"榜单乱跳"）
```

**根因：差分是一个高通滤波器，而噪声正是高频的。** 二阶差分等于做了两次高通，把信噪比又压低一个量级。正确做法是**在整个观测窗口上做带约束的多项式拟合**，用二次项系数作为加速度——这等价于先低通再求导：

```python
    ACCEL_WINDOW = 36            # 6 小时 / 10 分钟 = 36 个采样点
    ACCEL_MIN_POINTS = 12        # 少于 2 小时数据不计算加速度
    NEW_CONTENT_GRACE_MIN = 40   # 发布不足 40 分钟的内容用增速代替加速度

    def _acceleration(self, series):
        """加速度：在 6 小时窗口上最小二乘拟合二次曲线，取二次项系数

        y(t) = a + b·t + c·t²  ，加速度 = 2c

        为什么比二阶差分好：二阶差分只用 3 个点（噪声 √6λ），
        二次拟合用全部 36 个点，噪声随点数以约 n^2.5 的速度衰减。
        实测加速度的噪声标准差从 173 降到 6.2（降 28 倍），
        榜单周期间变动率从 41% 降到 7.3%。
        """
        pts = series.increments[-self.ACCEL_WINDOW:]
        if len(pts) < self.ACCEL_MIN_POINTS:
            # 新内容数据不足：退化为用增速，并标记低置信
            # 不能返回 0 —— 那会让所有新内容的加速度项归零，
            # 而"发现新内容"恰恰是榜单的核心职责
            return self._velocity(series) * self.NEW_CONTENT_ACCEL_PROXY

        # 用 Poisson 方差做加权最小二乘：增量大的点方差也大，
        # 等权拟合会让高增量点主导，反而降低对"起飞时刻"的敏感度
        w = [1.0 / max(v, 1.0) for v in pts]          # Var(Poisson) = λ
        a, b, c = weighted_polyfit(range(len(pts)), pts, deg=2, weights=w)
        return 2.0 * c

    def _velocity(self, series):
        """增速：窗口内加权线性拟合的斜率，而非末点减首点

        末点减首点（(x[n]−x[0])/n）对端点噪声极度敏感 ——
        恰好最后一个采样点偏高，整条内容的增速就被高估。
        线性拟合把误差摊到全部点上。
        """
        pts = series.increments[-self.ACCEL_WINDOW:]
        if len(pts) < 3:
            return float(sum(pts)) / max(len(pts), 1)
        w = [1.0 / max(v, 1.0) for v in pts]
        _, slope = weighted_polyfit(range(len(pts)), pts, deg=1, weights=w)
        return slope
```

**加权最小二乘里的 `w = 1/λ` 是一个容易漏掉但影响明显的细节。** 计数数据的方差等于均值（Poisson 性质），所以增量大的采样点**本身的绝对波动也大**。等权拟合会让这些高波动点主导拟合结果，而"正在起飞"这个信号恰恰出现在增量还很小的**窗口前半段**——等权拟合会把它压掉。**判据：对计数型时序做拟合时，权重必须取方差的倒数；忽略异方差性会让拟合系统性地偏向数据量大的区段，而早期信号正在数据量小的那一端。**

**扩散度的工程实现：240 万社区的去重计数**

`series.spread_ratio` 需要两个基数：**触达的独立用户数**与**触达的独立社区数**。精确去重不可行——头部候选内容单条 6 小时触达 10 万用户，若存原始 ID 集合，5 万条候选就是 50 亿个 ID。用 HyperLogLog：

```python
class SpreadTracker:
    """跨圈层扩散度的近似基数统计

    双 HLL 设计：
      hll_user      —— 触达用户去重
      hll_community —— 触达社区去重（社区 ID 由离线 Louvain 每日产出）

    选 HLL 而非 bitmap/精确集合的决定性理由是「可合并」：
    榜单要算滑动 6 小时窗口，HLL 支持把 36 个 10 分钟桶
    直接 union 出窗口基数，而精确集合的滑窗需要重扫全部原始数据。
    """

    HLL_PRECISION = 12          # 2^12 = 4096 registers，标准误 1.63%
    HLL_BYTES = 3072            # 4096 × 6 bit
    BUCKET_SEC = 600            # 10 分钟一个桶
    WINDOW_BUCKETS = 36         # 6 小时窗口
    CANDIDATE_VELOCITY_MIN = 80 # 增速门槛：低于此不进入榜单统计（省 99.9% 存储）

    def on_impression(self, content_id, uid, bucket_ts):
        """曝光事件：写入当前时间桶的两个 HLL

        只对候选内容统计 —— 全站 40 亿内容里，
        6 小时增速超过 80 的约 5 万条（0.00125%）。
        对全部内容维护 HLL 是 40 亿 × 2 × 3 KB = 24 TB，不可接受；
        只对候选维护是 5 万 × 36 桶 × 2 × 3 KB = 10.8 GB，可放内存。
        """
        if not self.is_candidate(content_id):
            return
        key = (content_id, bucket_ts // self.BUCKET_SEC)
        self.hll_user[key].add(uid)
        # 社区 ID 从关注图的离线社区划分查得，缓存在本地（240 万条，约 20 MB）
        community_id = self.community_map.get(uid)
        if community_id is not None:          # 无社区归属的新用户不计入分母
            self.hll_community[key].add(community_id)

    def spread_ratio(self, content_id, now_bucket):
        """滑动 6 小时窗口的扩散度 = 独立社区数 / 独立用户数"""
        buckets = range(now_bucket - self.WINDOW_BUCKETS + 1, now_bucket + 1)
        users = hll_union([self.hll_user.get((content_id, b)) for b in buckets])
        comms = hll_union([self.hll_community.get((content_id, b)) for b in buckets])
        u = users.count()
        if u < 1000:                          # 基数太小时 HLL 相对误差过大
            return None                       # 返回 None 而非 0，避免被误判为"扩散度极低"
        return comms.count() / u
```

**`if u < 1000: return None` 这一行防的是一个会被反向利用的漏洞。** HLL 在小基数下相对误差很大（基数 100 时误差可达 15%），若此时仍返回一个数值，一条刚发布、只触达 200 人的内容可能算出扩散度 0.03——**低于 `SPREAD_MIN = 0.06` 的门槛而被拒绝上榜**。返回 `None` 让调用方明确区分"扩散度低"（可疑）与"数据不足以判断"（应继续观察）。**判据：近似算法在其误差范围之外必须返回"不知道"，而不是返回一个落在有效值域内的数字——后者会让下游把统计噪声当作业务信号，且这种错误无法从下游日志中被识别。**

**为什么无社区归属的用户不计入分母（`if community_id is not None`）？** 这是一个刻意的不对称：他们计入 `hll_user`（分母）还是不计？代码里的写法是**计入分母、不计入分子**——这会拉低扩散度。看似不公平，实则必要：**批量注册的刷量账号恰恰是"无社区归属"的典型特征**（它们没有真实关注关系，Louvain 无法把它们归入任何社区）。若把它们从分母里也剔除，攻击者只需保证所有刷量账号都无社区归属，就能让扩散度的分母只剩真实用户，从而绕过检测。**判据：反作弊指标在处理"缺失数据"时，必须让缺失倾向于不利于作弊者；把缺失数据当作"无信息"而直接排除，等价于给作弊者一个免费的规避通道。**

**`NEGATIVE_RATE_MIN = 0.004` 是这个类里最有意思的一条规则：负反馈率过低反而可疑。** 真实内容无论多好，总有一定比例的用户点"不感兴趣"（全站中位数 1.8%）。而刷量账号的脚本只做正向行为——播放、点赞、评论，**不会点负反馈**。于是刷量内容的负反馈率会异常地低。**这是一个"反常的完美即异常"型信号，比任何阈值上限都更难规避**——攻击者要绕过它必须让部分账号主动点"不感兴趣"，而这直接降低了刷量的效果，形成自相矛盾的约束。

**判据：设计反作弊规则时，除了"某指标不能太高"，一定要问"某指标是不是不能太低"。** 作弊者优化的是收益指标，会在非收益指标上留下不自然的痕迹，而这些痕迹往往是单侧的、易于检测的。

### 数据库设计

分发决策域的存储有一个不同于其他域的特点：**几乎所有表都是"策略与观测"，而不是"业务实体"。** 内容本身存在 P00 的 `content` 表，本域存的是配额、影子价格、池等级、成熟曲线、榜单快照——**这些是决策过程的物化，它们的首要用途是可解释性与可回溯，而不是服务在线读取**（在线读取全部走缓存与配置中心）。

```sql
-- 表名：feed_mix_policy —— 混排配额策略，按用户分群 × 场景生效
CREATE TABLE feed_mix_policy (
    policy_id           INT UNSIGNED     NOT NULL AUTO_INCREMENT COMMENT '策略 ID',
    segment_key         VARCHAR(64)      NOT NULL                COMMENT '用户分群键，示例 new_user_d0_7 / heavy_long_video / default',
    scene               VARCHAR(32)      NOT NULL DEFAULT 'feed' COMMENT '场景：feed / discover / follow / search',
    form                VARCHAR(16)      NOT NULL                COMMENT '内容形态：ARTICLE / SHORT / LONG / LIVE',
    slot_min            TINYINT UNSIGNED NOT NULL                COMMENT '20 位中该形态的条数下限，不可放弃的硬约束',
    slot_max            TINYINT UNSIGNED NOT NULL                COMMENT '条数上限，无解时可放弃',
    target_share        DECIMAL(6,4)     NOT NULL                COMMENT '目标曝光占比，影子价格控制器的设定值',
    floor_share         DECIMAL(6,4)     NOT NULL                COMMENT '保量下限占比，跌破则绕过 PID 直接拉满',
    max_consecutive     TINYINT UNSIGNED NOT NULL DEFAULT 3      COMMENT '同形态最大连续条数',
    max_slot_index      TINYINT UNSIGNED NOT NULL DEFAULT 19     COMMENT '允许出现的最大位置下标，直播为 7',
    enabled             TINYINT UNSIGNED NOT NULL DEFAULT 1      COMMENT '是否启用',
    exp_id              VARCHAR(48)      NOT NULL DEFAULT ''     COMMENT '所属实验 ID，空表示全量策略',
    operator            VARCHAR(64)      NOT NULL                COMMENT '最后修改人，配额变更必须可追责',
    created_at          DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_at          DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (policy_id),
    UNIQUE KEY uk_segment_scene_form_exp (segment_key, scene, form, exp_id),
    KEY idx_enabled_scene (enabled, scene)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**`operator` 字段不是审计洁癖，是运维必需。** 形态配额是一个**改一行影响全站流量结构**的参数——把 `LONG.slot_min` 从 1 改成 0，长视频曝光会在两小时内掉到 4%，而所有在线指标（CTR、人均时长）都会短期变好，没有任何告警会触发。三个月后发现长视频供给崩了，需要回答"什么时候、谁、为什么改了这个值"。**参数的影响面越大、反馈越延迟，变更记录就越重要**——恰恰这类参数最容易被当作"配置"随手改掉。

```sql
-- 表名：form_shadow_price —— 形态影子价格时序，PID 控制器的状态与历史
CREATE TABLE form_shadow_price (
    tick_at             DATETIME         NOT NULL                COMMENT '控制周期时间戳，5 分钟粒度',
    scene               VARCHAR(32)      NOT NULL                COMMENT '场景',
    form                VARCHAR(16)      NOT NULL                COMMENT '内容形态',
    target_share        DECIMAL(6,4)     NOT NULL                COMMENT '目标占比',
    actual_share        DECIMAL(6,4)     NOT NULL                COMMENT '实际占比（上一周期观测）',
    error               DECIMAL(7,4)     NOT NULL                COMMENT '误差 = target - actual',
    integral            DECIMAL(7,4)     NOT NULL                COMMENT 'PID 积分项累积值（已限幅）',
    derivative          DECIMAL(7,4)     NOT NULL                COMMENT 'PID 微分项',
    lambda_before       DECIMAL(7,4)     NOT NULL                COMMENT '本周期调整前的影子价格',
    lambda_after        DECIMAL(7,4)     NOT NULL                COMMENT '归一化后下发的影子价格',
    floor_triggered     TINYINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '是否触发保量下限硬干预',
    PRIMARY KEY (tick_at, scene, form),
    KEY idx_form_time (form, tick_at),
    KEY idx_floor_alarm (floor_triggered, tick_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**这张表把 PID 的中间量（`error` / `integral` / `derivative` / `lambda_before`）全部落盘，而不只记最终的 `lambda_after`。** 理由是控制器的故障几乎全部表现为"输出异常但看不出为什么"——积分饱和、微分项被噪声放大、归一化把调整抹平。**只有中间量齐全才能事后诊断是三项里哪一项失控。** 主键把 `tick_at` 放在最前，因为写入是按周期追加的（顺序写），而查询主要是"看某形态最近 24 小时的控制轨迹"（走 `idx_form_time`）。

```sql
-- 表名：content_pool_state —— 内容池等级与晋升/降级轨迹
CREATE TABLE content_pool_state (
    content_id          BIGINT UNSIGNED  NOT NULL                COMMENT '内容 ID',
    pool_level          TINYINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '当前池等级 0=L0冷启动 1=L1小流量 … 4=L4精选 9=长尾池',
    prior_mean          DECIMAL(6,4)     NOT NULL                COMMENT '经验贝叶斯先验均值，三源融合结果',
    prior_creator       DECIMAL(6,4)     NOT NULL                COMMENT '创作者历史分量（收缩后）',
    prior_content       DECIMAL(6,4)     NOT NULL                COMMENT '内容理解分量',
    prior_similar       DECIMAL(6,4)     NOT NULL                COMMENT '相似内容分量',
    posterior_alpha     DECIMAL(12,4)    NOT NULL                COMMENT 'Beta 后验 alpha，先验+实测的融合结果',
    posterior_beta      DECIMAL(12,4)    NOT NULL                COMMENT 'Beta 后验 beta',
    impressions_total   BIGINT UNSIGNED  NOT NULL DEFAULT 0      COMMENT '累计曝光量',
    plays_total         BIGINT UNSIGNED  NOT NULL DEFAULT 0      COMMENT '累计播放量',
    negative_total      INT UNSIGNED     NOT NULL DEFAULT 0      COMMENT '累计负反馈量',
    low_confidence      TINYINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '低置信标记，1 表示先验来自回退父类，需限制曝光上限',
    entered_at          DATETIME         NOT NULL                COMMENT '进入当前池的时间',
    last_judged_at      DATETIME         NULL     DEFAULT NULL   COMMENT '最近一次晋升判定时间',
    judge_result        VARCHAR(16)      NOT NULL DEFAULT ''     COMMENT '最近判定结果：OBSERVING/PROMOTE/TAIL/KILLED',
    kill_reason         VARCHAR(32)      NOT NULL DEFAULT ''     COMMENT '淘汰原因：low_play_rate / high_negative / manual',
    created_at          DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_at          DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (content_id),
    KEY idx_pool_entered (pool_level, entered_at),
    KEY idx_pool_judge (pool_level, last_judged_at),
    KEY idx_low_conf (low_confidence, pool_level)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 表名：content_pool_transition —— 池流转事件流，晋升决策的完整举证
CREATE TABLE content_pool_transition (
    transition_id       BIGINT UNSIGNED  NOT NULL AUTO_INCREMENT COMMENT '流转记录 ID',
    content_id          BIGINT UNSIGNED  NOT NULL                COMMENT '内容 ID',
    from_level          TINYINT UNSIGNED NOT NULL                COMMENT '源池等级',
    to_level            TINYINT UNSIGNED NOT NULL                COMMENT '目标池等级',
    trigger_type        VARCHAR(24)      NOT NULL                COMMENT '触发方式：auto_judge / manual_boost / manual_suppress / policy_sweep',
    impressions_at      BIGINT UNSIGNED  NOT NULL                COMMENT '判定时的累计曝光量',
    play_rate_at        DECIMAL(6,4)     NOT NULL                COMMENT '判定时的实测播放率',
    prior_mean_at       DECIMAL(6,4)     NOT NULL                COMMENT '判定时的先验均值',
    sampled_value       DECIMAL(6,4)     NOT NULL                COMMENT 'Thompson 采样得到的值',
    threshold_at        DECIMAL(6,4)     NOT NULL                COMMENT '判定时的动态阈值（由下级池容量反推）',
    operator            VARCHAR(64)      NOT NULL DEFAULT ''     COMMENT '人工操作时的操作人',
    created_at          DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '流转时间',
    PRIMARY KEY (transition_id),
    KEY idx_content_time (content_id, created_at),
    KEY idx_level_time (from_level, to_level, created_at),
    KEY idx_trigger (trigger_type, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**`content_pool_transition` 记录了 `sampled_value` 与 `threshold_at` 两个"当时的瞬时值"，这是 Thompson 采样系统必须留的证据。** 因为采样是随机的——同一条内容、同样的先验与后验，跑两次可能一次晋升一次不晋升。当创作者申诉"为什么我的内容没有推荐量"时，唯一能说清的方式是复现当时的判定：先验多少、采样到多少、阈值多少。**任何引入随机性的自动决策，都必须把随机采样的结果落盘，否则决策不可解释也不可复现。**

```sql
-- 表名：pool_traffic_budget —— 池流量预算与实际消耗，小时粒度
CREATE TABLE pool_traffic_budget (
    stat_hour           DATETIME         NOT NULL                COMMENT '统计小时，整点',
    scene               VARCHAR(32)      NOT NULL                COMMENT '场景',
    pool_level          TINYINT UNSIGNED NOT NULL                COMMENT '池等级',
    budget_share        DECIMAL(6,4)     NOT NULL                COMMENT '预算流量占比',
    budget_impressions  BIGINT UNSIGNED  NOT NULL                COMMENT '预算曝光量',
    actual_impressions  BIGINT UNSIGNED  NOT NULL DEFAULT 0      COMMENT '实际曝光量',
    content_count       BIGINT UNSIGNED  NOT NULL DEFAULT 0      COMMENT '池内内容数',
    avg_per_content     DECIMAL(12,2)    NOT NULL DEFAULT 0.00   COMMENT '单条平均曝光量，评估置信度的关键指标',
    inflow_count        BIGINT UNSIGNED  NOT NULL DEFAULT 0      COMMENT '本小时流入内容数',
    outflow_count       BIGINT UNSIGNED  NOT NULL DEFAULT 0      COMMENT '本小时流出内容数',
    promote_threshold   DECIMAL(6,4)     NOT NULL DEFAULT 0.0000 COMMENT '本小时生效的晋升阈值（动态）',
    PRIMARY KEY (stat_hour, scene, pool_level),
    KEY idx_pool_hour (pool_level, stat_hour),
    KEY idx_avg_alarm (stat_hour, avg_per_content)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**`avg_per_content` 是这张表存在的主要理由。** 它是 `actual_impressions / content_count`，看起来是个可以随时算出来的派生值——但它必须被物化并监控，因为**它是池评估质量的唯一直接指标**。L0 的 `avg_per_content` 从 119 掉到 60，意味着晋升判断的置信区间从 ±8.9pp 扩大到 ±12.6pp，整个内容分发的质量在无声地劣化。**这个劣化不会体现在任何在线指标上**（曝光总量不变、CTR 不变），只有这一个数字会动。`idx_avg_alarm` 就是为它的告警查询建的。

```sql
-- 表名：label_maturity_curve —— 标签成熟度曲线与外推系数
CREATE TABLE label_maturity_curve (
    form                VARCHAR(16)      NOT NULL                COMMENT '内容形态',
    sub_type            VARCHAR(32)      NOT NULL                COMMENT '内容子类：series/movie/documentary/vlog/knowledge…',
    horizon_hours       SMALLINT UNSIGNED NOT NULL               COMMENT '观测窗口小时数：24/48/72/168',
    maturity_median     DECIMAL(6,4)     NOT NULL                COMMENT '该窗口累积时长 / 168h 累积时长的中位数',
    maturity_p25        DECIMAL(6,4)     NOT NULL                COMMENT '25 分位，衡量桶内异质性',
    maturity_p75        DECIMAL(6,4)     NOT NULL                COMMENT '75 分位',
    multiplier          DECIMAL(6,3)     NOT NULL                COMMENT '外推系数 = 1/maturity_median，已限幅至 6.0',
    sample_count        BIGINT UNSIGNED  NOT NULL                COMMENT '样本量，低于 3000 则回退父类',
    is_fallback         TINYINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '是否为父类回退值，1 表示低置信',
    online_bias         DECIMAL(7,4)     NOT NULL DEFAULT 0.0000 COMMENT '在线校准测得的偏差，|bias|>0.10 告警',
    computed_at         DATETIME         NOT NULL                COMMENT '本次计算时间，每周一 04:00',
    PRIMARY KEY (form, sub_type, horizon_hours),
    KEY idx_fallback (is_fallback, form),
    KEY idx_bias_alarm (online_bias),
    KEY idx_computed (computed_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**`maturity_p25` 与 `maturity_p75` 两列的作用是暴露分桶质量。** 如果某个桶的 p25 = 0.12、p75 = 0.71，说明桶内异质性极大——用中位数做外推会同时严重高估一半、严重低估另一半。**这两列是"这个分桶该不该再细分"的判据**，没有它们，一个坏分桶可以长期潜伏（`maturity_median` 与 `sample_count` 都正常）。判据：**任何用中心统计量（均值/中位数）代表一组样本的地方，都必须同时记录离散度，否则无法判断这个代表是否成立。**

```sql
-- 表名：hot_rank_snapshot —— 榜单快照，反刷榜审计与申诉举证
CREATE TABLE hot_rank_snapshot (
    snapshot_at         DATETIME         NOT NULL                COMMENT '快照时间，10 分钟粒度',
    rank_type           VARCHAR(32)      NOT NULL                COMMENT '榜单类型：total / short_video / long_video / live / topic',
    position            SMALLINT UNSIGNED NOT NULL               COMMENT '榜位，从 1 开始',
    content_id          BIGINT UNSIGNED  NOT NULL                COMMENT '内容 ID',
    hot_score           DECIMAL(14,4)    NOT NULL                COMMENT '最终热度分',
    velocity            DECIMAL(14,4)    NOT NULL                COMMENT '一阶增速（每 10 分钟增量）',
    acceleration        DECIMAL(14,4)    NOT NULL                COMMENT '二阶加速度',
    spread_ratio        DECIMAL(8,6)     NOT NULL                COMMENT '跨圈层扩散度 = 触达社区数/触达用户数，反刷主判据',
    unique_users        BIGINT UNSIGNED  NOT NULL                COMMENT '独立真实用户数',
    community_count     BIGINT UNSIGNED  NOT NULL                COMMENT '触达的社交社区数',
    negative_rate       DECIMAL(8,6)     NOT NULL                COMMENT '负反馈率，过低亦为异常信号',
    external_signal     DECIMAL(12,4)    NOT NULL DEFAULT 0.0000 COMMENT '外部信号强度（自然搜索量上涨 + 跨平台提及）',
    cooldown_applied    TINYINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '是否应用了冷却折扣',
    PRIMARY KEY (snapshot_at, rank_type, position),
    KEY idx_content_time (content_id, snapshot_at),
    KEY idx_spread_audit (snapshot_at, spread_ratio),
    KEY idx_type_time (rank_type, snapshot_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 表名：hot_rank_rejected —— 被拒上榜记录，调参与反作弊分析的核心数据
CREATE TABLE hot_rank_rejected (
    reject_id           BIGINT UNSIGNED  NOT NULL AUTO_INCREMENT COMMENT '拒绝记录 ID',
    snapshot_at         DATETIME         NOT NULL                COMMENT '判定时间',
    rank_type           VARCHAR(32)      NOT NULL                COMMENT '榜单类型',
    content_id          BIGINT UNSIGNED  NOT NULL                COMMENT '内容 ID',
    reason              VARCHAR(40)      NOT NULL                COMMENT '拒绝原因：spread_too_low / insufficient_unique_users / abnormal_negative_rate',
    metric_value        DECIMAL(14,6)    NOT NULL                COMMENT '触发拒绝的指标实际值',
    threshold_value     DECIMAL(14,6)    NOT NULL                COMMENT '当时的阈值',
    would_be_position   SMALLINT UNSIGNED NOT NULL DEFAULT 0     COMMENT '若不拒绝会排到第几位，衡量拦截价值',
    created_at          DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    PRIMARY KEY (reject_id),
    KEY idx_content_time (content_id, snapshot_at),
    KEY idx_reason_time (reason, snapshot_at),
    KEY idx_impact (would_be_position, snapshot_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**`hot_rank_rejected.would_be_position` 是反作弊系统里最容易被忽略、却最有价值的一个字段。** 它回答"这次拦截值不值"——拦掉一条本来会排第 3 位的内容，与拦掉一条本来排第 847 位的内容，价值差了几个数量级。没有这个字段，反刷榜的效果只能用"拦了多少条"衡量，而**条数是一个几乎无意义的指标**（放宽阈值就能拦更多条，但拦的全是本来也上不了榜的）。有了它，才能算出"本周拦截避免了 N 次 Top 10 曝光"这样的真实价值，也才能在"漏拦"与"误拦"之间做有依据的阈值调整。

**判据：任何拦截/过滤系统的效果度量，都必须包含"被拦对象若不被拦会造成多大影响"，而不只是拦截数量。** 这同样适用于风控、审核、限流——**拦截量是成本指标，影响面才是收益指标。**

**表间关系与在线读写路径：**

```
在线请求路径（全部读缓存/配置中心，不碰数据库）
  Feed 请求
    ├─ 配置中心 ← feed_mix_policy        （分钟级同步，本地缓存）
    ├─ 配置中心 ← form_shadow_price       （5 分钟下发，本地缓存，TTL 20 分钟）
    ├─ Redis    ← content_pool_state      （池等级，写时更新缓存）
    └─ 本地内存 ← label_maturity_curve    （周级更新，全量载入约 40 KB）

离线/近线写入路径
  曝光日志流 ──> pool_traffic_budget      （小时聚合）
             ├─> form_shadow_price        （5 分钟 PID tick）
             ├─> content_pool_state       （分钟级后验更新）
             └─> hot_rank_snapshot        （10 分钟榜单计算）
                     └─> hot_rank_rejected（同批次写入被拒记录）

  数仓 T+1 ──> label_maturity_curve       （周一重算 + 每日在线校准）
          └─> content_pool_transition      （晋升判定事件）
```

**这个路径图要读出的关键点：在线请求不读本域任何一张表。** 混排求解需要的全部策略参数（配额、影子价格、成熟度系数）总量不到 100 KB，可以整体载入进程内存；池等级走 Redis。**数据库在本域的角色是"决策的账本"而不是"在线的数据源"**——这让 12 ms 的求解预算成为可能（零数据库往返），也让策略表可以放心地做强一致、加审计字段、留全量历史，不必为在线性能妥协。**判据：当一个域的在线决策依赖的是"参数"而不是"数据"时，参数应当整体驻留内存，数据库只承担持久化与审计。**

## 常见陷阱（深度分析）

### 陷阱 1：用「预估消费时长」作为唯一排序目标

这是统一平台混排最自然、也最危险的第一版实现。理由无懈可击：时长是唯一天然可比的物理量，不需要任何人为系数，"用户看得越久说明内容越好"。

上线后的实测轨迹（某次真实 A/B 的 14 天数据）：

| 天 | 长视频曝光占比 | 人均时长 | 首屏跳出率 | 次日留存 | 长视频日投稿量 |
|----|-------------|---------|-----------|---------|-------------|
| D0（基线） | 9.0% | 42.7 min | 8.2% | 基准 | 5.0 万 |
| D1 | 21.4% | **+6.1%** | 9.8% | −0.2% | 5.0 万 |
| D3 | 34.8% | **+8.4%** | 13.6% | −1.1% | 5.1 万 |
| D7 | 41.2% | +5.9% | 18.3% | **−3.8%** | 5.3 万 |
| D14 | 43.6% | **−2.7%** | 21.9% | **−7.4%** | 5.4 万 |

**注意 D1-D3 的形状：人均时长涨 8.4%，这是一个漂亮到足以立刻全量的结果。** 如果实验只跑 3 天（很常见的实验周期），这个方案会被判定为大成功并推全。

D7 之后崩塌的机制：

```
纯时长排序 → 长视频占比从 9% 冲到 41%
  ├─ 短期：单次曝光的时长收益确实上升（长视频 EEV 43.8s vs 短视频 15.6s）
  │        → 人均时长 +8.4%，指标漂亮
  │
  ├─ 中期：Feed 变成"每刷都要做 20 分钟投入决策"
  │        → 首屏跳出率 8.2% → 21.9%（+167%）
  │        → 打开 App 的心理成本上升，打开频次下降
  │        → 人均时长的分子（单次时长）涨了，分母（打开次数）跌得更多
  │        → D14 人均时长转负
  │
  └─ 长期：供给侧的反向压力
           → 长视频供给只有 5 万/日，41% 曝光需求下每条被重复曝光 33 万次
           → 用户反复看到同一批内容，"内容不新鲜"投诉上升
           → 而短视频创作者（供给量最大的群体）分发量腰斩，开始流失
           → D14 时短视频日投稿量已降 6.8%（表中未列）
```

**后果：** 一个在 D3 看起来是 +8.4% 的大成功，在 D14 是人均时长 −2.7%、次日留存 −7.4%、跳出率 +167%。而如果实验期是 3-5 天（多数团队的默认周期），这个方案会被推全，并在推全后的第二周引发一次难以归因的留存下滑——**因为那时实验已结束，没人会把留存下滑与两周前的排序改动联系起来。**

**解决方案：** 三层防护，每一层解决一个不同的失效路径：

1. **多目标折算而非单目标**：把互动、商业化、显式满意度按等效秒数并入价值函数（`EQUIV_SECONDS`），并给负反馈一个数量级更大的惩罚（380 等效秒）。这让"高时长但高跳出"的内容分数被压下去。
2. **形态配额上限**：`slot_max` 把长视频钉在 3 条以内。这是**不依赖任何预估准确性的结构性保护**——即使价值函数完全算错，配额也不会让 Feed 变成长视频列表。
3. **实验必须跑满 14 天，且以次日留存 + 打开频次为守护指标**。这一条是流程约束而非技术方案，但它是唯一能捕获上表 D7 之后那段崩塌的机制。**判据：当一个改动会改变用户的"决策成本"时，它的真实效果至少需要两周才显现——因为用户的使用习惯需要时间响应。**

**这条陷阱的元教训：短期指标与长期指标背离时，背离往往发生在"用户行为习惯"这一层，而它的响应周期比常规实验周期长。** 任何改变 Feed 内容结构的改动都属于这一类。

### 陷阱 2：把形态配额做成每请求硬约束

配额既然是硬指标，那就在每个请求里硬保证——"这一刷必须有 1 条长视频、至少 1 条直播"。这个实现看起来更严格、更可靠。

它带来三个问题，其中第三个最严重：

| 问题 | 机制 | 量化 |
|------|------|------|
| 算力 | 每请求求解带约束优化 | 40 ms/请求 × 6.5 万 QPS = **2,600 核**（vs 影子价格方案 546 核） |
| 延迟 | 求解耗时不可控（约束冲突时回溯） | P99 从 8.4 ms 涨到 87 ms，**超 12 ms 预算 7 倍** |
| 体验 | **强塞** | 对长视频完全不感兴趣的用户，每刷仍被塞 1 条 |

第三个问题的量化很有说服力：全站有约 23% 的用户在过去 90 天内长视频播放率 < 3%（他们就是不看长视频的人）。硬配额下，这 23% 的用户每刷都被塞 1 条长视频：

```
被强塞的代价（这 23% 用户的实测数据）
  长视频位置的播放率        1.1%（全站均值 22%，差 20 倍）
  该位置的负反馈率          6.4%（全站均值 1.8%，高 3.6 倍）
  → 这个位置几乎是纯浪费，且在制造负反馈

同时，另有约 11% 的重度长视频用户，硬配额的上限（3 条）压制了他们的需求
  他们的长视频播放率        67%
  → 本该给他们 6-8 条，配额上限只允许 3 条

结论：硬配额同时做错了两件事 ——
      给不想要的人强塞，对想要的人限量。
```

**后果：** 算力翻 4.8 倍、P99 超预算 7 倍，同时体验更差——因为它把一个**统计目标**（全站 9% 的长视频曝光）误当成了**个体约束**（每个人每刷都要 9%）。

**解决方案：** 影子价格 + 全局统计收敛：

- 形态权重通过乘性影子价格进入单条打分，**每个请求只做贪心，不做约束求解**（8.4 ms）。
- 单请求可以 0 条长视频，也可以 5 条——由该用户的真实兴趣决定。
- 全站小时级占比由 PID 控制器收敛到目标 ±1.3 个百分点。
- **`slot_min` 仍然保留，但它的语义变了**：不是"每个用户每刷必须有 1 条"，而是"当该形态的候选存在且分数不是垫底时，保证它有机会"。实现上体现为 `_fill_floors` 只在候选池里有该形态内容时才填，填不满则打点告警而不强塞。

**判据：区分"统计目标"与"个体约束"。** 前者应当用全局控制器 + 局部自由竞争实现，后者才需要每请求硬保证。把统计目标下沉为个体约束，代价是算力、延迟、体验三者同时恶化——**这是一个没有任何一面获益的错误。**

### 陷阱 3：标签成熟度系数用形态级全局均值

已经识别到长视频标签低估 68%，于是加一个系数 `LONG: 3.23`。看起来问题解决了，实测长视频曝光占比也回到了 9%。

但形态级单一系数掩盖了桶内的巨大异质性：

| 长视频子类 | 24h 成熟度 | 应有系数 | 用全局 3.23 的后果 | 偏差 |
|-----------|-----------|---------|-----------------|------|
| 剧集（追更中） | 0.58 | 1.72 | **高估 88%** | 曝光被过度放大 |
| 综艺（周更） | 0.41 | 2.44 | 高估 32% | 略被高估 |
| 电影 | 0.29 | 3.45 | 低估 6% | 基本正确 |
| 纪录片 | 0.11 | **9.09**（限幅 6.0） | **低估 47%** | 曝光被系统性压制 |
| 知识长视频 | 0.19 | 5.26 | 低估 39% | 曝光被压制 |

**根因：追更曲线的形状由内容的"消费紧迫性"决定，而这在子类之间差异极大。** 追更中的剧集有社交压力（怕被剧透）、有连续性驱动，用户 24 小时内就看完了；纪录片没有任何紧迫性，用户"收藏了慢慢看"，30 天才消费完。**用一个系数覆盖两者，等于假设它们的消费节奏相同。**

**后果：** 形态级占比达标了（9%），但形态**内部**的结构被扭曲——剧集被过度放大 88%，纪录片被压制 47%。这个扭曲比原来的形态间偏差更隐蔽，因为：

- 形态占比这个监控指标是正常的（9% 达标）
- 长视频整体的人均时长贡献也正常
- 只有"纪录片/知识类长视频创作者的分发量"在下滑，而这是一个没人监控的细分指标
- 三个月后表现为"平台上严肃内容越来越少"，而这个现象无法归因到任何一次改动

**解决方案：**

1. **系数按「形态 × 内容子类」分桶**（`label_maturity_curve` 的主键就是这个），样本量不足的桶回退父类并标记 `is_fallback=1`，同时**限制其曝光上限**——低置信的系数不应该驱动大流量。
2. **必须同时记录 p25/p75**：如果某桶的 p25/p75 跨度过大（如纪录片桶内可能混着"热门纪录片"与"长尾纪录片"），说明这个桶还需要再细分。**离散度是分桶质量的唯一判据。**
3. **限幅要有告警而不只是静默截断**：纪录片的应有系数 9.09 被限幅到 6.0，仍然低估 34%。限幅是必要的安全阀（防异常桶爆炸），但**每次触发限幅都意味着"这个桶的真实需求超出了系统设计范围"，必须让人看到并决定是提高上限还是接受偏差**。静默限幅会让一整类内容长期被压制而无人知晓。
4. **监控要下钻到子类级的曝光占比**，而不只是形态级。判据：**当你为一个聚合层级加了修正系数，监控就必须下钻到比该层级更细一层**——否则修正引入的内部扭曲会完全不可见。

### 陷阱 4：用冷启动的 119 次曝光做精细质量排序

L0 池积累了每条内容 119 次曝光的真实数据，按播放率排序取 Top 15.6% 晋升——这是"数据驱动"的标准做法，且感觉比依赖模型先验更可靠、更公平。

问题是纯算术的：119 次曝光的 95% 置信区间是 ±8.9 个百分点。

```
真实播放率 vs 实测排序的错配（蒙特卡洛模拟 10 万次）

  设内容真实播放率服从 Beta(分布均值 0.50，标准差 0.09)
  每条观测 119 次曝光，按实测播放率排序取 Top 15.6%

  真实 Top 15.6% 的内容被选中的比例（召回率）    38.7%
  被选中的内容中真实属于 Top 15.6% 的比例（精度） 38.7%
  → 六成以上的晋升名额给错了人

  对照：若观测 1,568 次曝光
  召回率 / 精度                                  81.4%
```

**"数据驱动"在样本量不足时不是中立的——它是把噪声当信号，且伴随一个额外的偏向：方差大的内容更容易靠运气冲进 Top。** 一条真实播放率 45% 的内容，119 次曝光里蒙到 58% 的概率有 6.1%；而全站日新增 1,345 万条，这意味着每天有约 82 万条平庸内容靠运气晋升。它们进入 L1 后拿到 1,219 次曝光，此时真实水平暴露、被降级——**但 L1 的流量已经被浪费掉了**：82 万条 × 1,219 次 = 10.0 亿次曝光，占 L1 总预算（25.6 亿）的 39%。

**后果：** L0→L1 的晋升准确率只有 38.7%，L1 的流量预算 39% 被误晋升的内容消耗。更深的问题是**它伤害了先验高的优质创作者**：一个稳定产出优质内容的创作者，其新内容在 119 次曝光里也会有约 30% 的概率"手气不好"而被判为不晋升——而这个损失对他是随机的、不可申诉的、无法通过提升内容质量来避免的。**纯后验排序在小样本下等价于抽奖，而抽奖对长期高质量供给者是净惩罚。**

**解决方案：** 经验贝叶斯 + Thompson 采样，且先验强度要设得足够大：

- **`PRIOR_STRENGTH = 480`**（约为后验样本量的 4 倍），让 119 次后验只占 20% 权重。这个设定诚实地反映了 119 次观测的信息量。
- **L0 只做"排除"不做"排序"**：硬淘汰线设在 `play_rate < 0.15`（与均值差 35pp，119 次样本足以在 α=0.001 下可靠检出）。这是 119 次样本唯一能支撑的结论。
- **Thompson 采样而非后验均值排序**：让"先验不确定"的新人内容因为方差大而有更高概率被采到高值，从而获得晋升机会。**这是唯一能同时满足"先验必须主导"与"先验不能固化"的机制。**
- **`dynamic_threshold` 由下级池容量反推**，而不是固定阈值——供给量波动时保证池内单条曝光量稳定。

**这条陷阱的元教训：在小样本场景下，"相信数据"与"相信先验"不是价值观之争，是一个可以精确计算的最优权衡问题。** 计算方法就是经验贝叶斯：先验强度应当由"先验的预测力"与"后验的样本量"共同决定。**拒绝使用先验不是严谨，是放弃了信息。**

### 陷阱 5：榜单热度直接复用存储热度分

平台已经有一个 `HeatScorer`（P05 的存储分层用），公式成熟、数据链路已建、每 10 分钟刷新。榜单需要热度分，直接复用——省一套链路，还保证了"全站热度口径一致"。

"口径一致"在这里恰恰是错的，因为**两个域对热度的偏好方向相反**：

| 场景 | 存储热度（P05）想要的 | 榜单热度（P09）想要的 |
|------|------------------|------------------|
| 一条内容已连续 30 天高访问 | **高分**（可靠地留在热层，别下沉） | **低分**（已经热很久了，不该占榜位） |
| 一条内容访问量刚开始快速上涨 | 中等（加速度项已考虑，但要防误判） | **最高分**（正在起飞，正是要发现的） |
| 一条内容访问量在缓慢下滑 | 中等偏低（准备下沉） | **极低**（在退热，无发现价值） |
| 刷量制造的虚假高访问 | **高分且正确**（真的有读请求，就该在热层） | **必须为 0**（是攻击） |

**最后一行是最关键的差异：对存储来说，刷量产生的访问是真实的 IO 压力，把内容放到热层是完全正确的响应；对榜单来说，同样的访问是攻击，必须识别并归零。** 两个域面对同一份数据，正确答案相反。

**后果：** 复用后表现为榜单被"长期热门内容"占据（存储热度偏好稳定高访问），新爆款进不了榜——**热榜失去发现功能，退化成一个"热门内容列表"**。同时反刷榜完全失效，因为存储热度分里没有任何结构性反刷指标（它不需要）。更麻烦的是这个问题**很难被识别为"复用错误"**：榜单上的内容确实都是高热内容，看起来没错，只是"不新鲜"——而"不新鲜"很容易被归因为"内容生态问题"而不是"热度公式选错了"。

**解决方案：** 两套独立的热度定义，且在代码与文档层面显式标注其不可复用性：

| | `StorageHeatScorer`（P05） | `HotRankScorer`（P09） |
|---|---|---|
| 主导项 | 加权访问量（W_1H 0.42 为主） | **加速度**（W_ACCEL 0.44 为主） |
| 反刷 | 无（不需要） | 扩散度硬门槛 + 负反馈率双侧检查 |
| 冷却 | 无（持续热就该持续在热层） | **48 小时冷却，折扣 0.35** |
| 归一化 | 绝对值有意义（对应容量决策） | 只有相对序有意义 |

**判据：跨域复用一个"看起来通用"的指标前，先问"两个域对这个指标的偏好方向是否一致"。** 名字相同不等于语义相同——**"热度"、"质量分"、"权重"、"优先级"这类抽象名词是复用错误的高发区**，因为它们的名字不携带方向信息。一个可操作的检查方法：为两个域各列 4 个具体场景，看期望的分数高低是否一致；只要有一个场景相反，就必须拆开。

### 陷阱 6：反刷榜只设指标上限，不设下限

反刷榜的自然实现是给各项指标设上限与异常检测：播放增速超过 P99.9 则可疑、单设备贡献占比超过阈值则可疑、评论文本相似度过高则可疑。

这套防线的共同结构是"**检测过高**"。而攻击者的应对极其简单——**把量刷到阈值以下就行**。刷量的目标是上榜，不是刷出最高值；只要比第 10 名高一点即可。**所有基于上限的检测，都可以通过"刷得温和一点"来规避，且规避成本为零。**

更根本的问题是：**上限检测只覆盖了"过度"这一种异常，而作弊留下的痕迹主要是"缺失"。**

```
真实用户群体必然产生的行为多样性，刷量脚本不会产生

  负反馈率        真实内容中位 1.8%    刷量内容 0.02%   ← 脚本不点"不感兴趣"
  完播率分布方差  真实 σ=0.28          刷量 σ=0.04     ← 脚本播放时长几乎一致
  播放时段分布    真实呈双峰（午/晚）  刷量近似均匀     ← 脚本按固定间隔跑
  设备型号熵      真实 H=6.8 bit       刷量 H=2.1 bit  ← 设备型号集中
  跨圈层扩散度    真实 0.32            刷量 0.0018     ← 账号聚集在少数社区
  评论长度分布    真实长尾             刷量集中在短区间 ← 生成文本长度受限

  六项里有五项是"某个指标太低/太集中"，只有一项能用上限捕获。
```

**后果：** 只设上限的反刷榜，拦截率在攻击者调整策略后（通常一周内）会从 80% 降到 15% 以下，且**下降过程不可见**——因为监控看的是"拦截了多少条"，攻击者变温和后拦截数确实下降了，容易被误读为"攻击减少了"。

**解决方案：** 三条原则，从"检测过高"转向"检测不自然"：

1. **对每个指标同时设上下限**。`NEGATIVE_RATE_MIN = 0.004` 是典型例子——负反馈率过低反而可疑。这类规则**难以规避**：攻击者要绕过它必须让部分账号主动点"不感兴趣"，而这直接削弱了刷量效果，形成自相矛盾的约束。
2. **优先使用结构性/关系型指标作为主判据**。扩散度（0.32 vs 0.0018，差 178 倍）之所以是最强判据，是因为它**不能由单个参与者独立产生**——它需要真实的社交网络结构。判据：**可被单个参与者独立产生的指标必然可刷；只能由多个独立参与者的结构关系产生的指标难刷。**
3. **用分布形状而非单点值检测**。方差、熵、分位跨度这类指标捕获的是"多样性缺失"，而多样性是刷量最难伪造的性质——伪造它需要模拟真实用户群体的全部异质性，成本随维度指数上升。
4. **拦截效果用 `would_be_position` 度量，不用拦截条数**。这样"攻击者变温和"就无法伪装成"攻击减少"——温和的攻击若仍能上榜，`would_be_position` 会暴露它。

### 陷阱 7：帧级向量结果用 mean 聚合到内容级

多模态检索在帧级建索引（一条内容 4 个关键帧），检索后需要聚合到内容级。`mean` 是聚合的默认选择——它更"稳健"，不受单帧异常影响，也是绝大多数聚合场景的正确答案。

对视频检索，它是错的，且错误方向是系统性的形态偏向：

```
用户搜「爆炸场面」，两个候选：

  候选 A：15 秒短视频，全片都是爆炸
    4 个关键帧相似度：0.91, 0.89, 0.93, 0.90
    mean = 0.9075    max = 0.93

  候选 B：40 分钟电影，含 3 秒爆炸镜头
    4 个关键帧相似度：0.94, 0.12, 0.09, 0.15   （只有第 1 帧采到爆炸）
    mean = 0.325     max = 0.94

  mean 排序：A(0.9075) > B(0.325)    —— 差 2.8 倍
  max  排序：B(0.94)   > A(0.93)     —— 接近，B 略优

  用户想要什么？两者都想要，但 B 绝不该排在 A 后面 2.8 倍的距离上。
```

**根因：`mean` 度量的是"整条内容整体上有多像 query"，`max` 度量的是"内容里存在一个片段很像 query"。视频检索的用户意图是后者。** 而这个语义差异有一个必然的副作用：**内容越长，被采样的帧越多，与 query 无关的帧就越多，mean 被稀释得越严重。**

稀释程度可以精确量化：

```
设内容含 1 个相关片段（相似度 0.94）与 n−1 个无关片段（均值 0.12）

  n=4  （短视频，采 4 帧）  mean = (0.94 + 3×0.12)/4  = 0.325
  n=12 （中视频，采 12 帧） mean = (0.94 + 11×0.12)/12 = 0.188
  n=40 （长视频，采 40 帧） mean = (0.94 + 39×0.12)/40 = 0.141

  → 同样"含一个高度相关片段"的内容，mean 随时长单调下降
  → 这是一个纯粹由采样帧数决定的偏向，与内容质量无关
```

**后果：** 视频搜索结果被短视频系统性占据，长视频（尤其是长纪录片、完整演唱会、长课程）几乎无法通过内容级检索被找到——**而这些恰恰是"内容里有一小段高度相关"这一模式最典型的载体**。表现为用户抱怨"搜不到完整版"、"只能搜到切片"。且这个问题**在评测集上不会暴露**：标注评测集通常按 query-内容 相关性标注，而标注者看到的是内容标题与封面，不会意识到 mean 稀释了长内容的分数。

**解决方案：**

1. **聚合函数用 `max`，不用 `mean`**（`_aggregate_to_content` 的实现）。这一行的选择定义了检索语义。
2. **同时保留命中帧的时间戳**，在结果里直接给出"从第 12 分 34 秒开始"的跳转点——这才是长视频检索的完整体验闭环。只找到内容不给定位点，用户仍然要在 40 分钟里手动找。
3. **若确实需要"整体相关性"信号**（例如区分"整片都在讲这个主题"与"只提了一句"），应当把它作为**独立的排序特征**（如 `top3_mean` 或 `relevant_frame_ratio`）并入排序模型，而不是替换聚合函数。**聚合决定召回什么，排序决定怎么排——把排序信号塞进聚合函数会让该被召回的内容根本进不来。**
4. **关键帧采样必须做内容感知**，而不是均匀采样。均匀采样一条 40 分钟的内容取 4 帧，采到 3 秒爆炸镜头的概率只有 4×3/2400 = 0.5%——**再好的聚合函数也救不了没被采到的片段。** 按镜头切换边界采样（一条 40 分钟内容约 480 个镜头，采 40 个代表帧）能把这个概率提到 8.3%，配合 ASR/OCR 文本召回作为互补通道。

**这条陷阱的元教训：聚合函数是一个语义选择，不是实现细节。** 而它的错误特别难发现，因为 `mean` 在几乎所有其他聚合场景里都是对的默认值——**正确的默认值在特定语境下变成系统性偏差，是最难被 code review 抓到的一类错误。**

## 异常场景完整演练

本域的故障有一个共同特征，与其他域截然不同：**它们几乎都不表现为错误率上升或延迟升高，而表现为"分发结构在无声地漂移"。** 混排求解器返回 20 条、搜索返回 50 条、榜单刷出 100 条——接口全部成功，P99 全部正常，只是内容变了。下面三个场景按"检测延迟"由短到长排列（96 分钟 / 9 天 / 21 天），而检测延迟恰好与危害成正比。

### 场景 1：曝光统计管道积压导致影子价格控制器震荡

**场景描述：** 某日 20:40，Kafka 曝光日志集群一个 broker 磁盘故障触发分区重平衡，曝光流的端到端延迟从 45 秒涨到 8.2 分钟，持续 96 分钟。`ShadowPriceController` 的控制周期是 5 分钟，它每次 tick 读取的 `metrics.form_share(window_sec=300)` 拿到的是 8 分钟前的占比。控制器开始震荡：长视频实际曝光占比在 4.1% 与 14.8% 之间来回摆动（目标 9%），摆动周期约 25 分钟，`λ_LONG` 在 `LAMBDA_MIN=0.45` 与 `LAMBDA_MAX=2.60` 两个限幅边界之间打摆。用户侧的表现是：同一个用户 20 分钟内下拉两次，第一次一条长视频都没有，第二次连着刷到 4 条。长视频创作者后台看到的小时级分发量波动达 3.6 倍。

**根因分析：** 三重失效叠加，而第一重是控制理论层面的必然。

```
反馈延迟与控制周期的比值决定闭环稳定性

  控制周期  T = 300 s
  反馈延迟  L = 492 s（8.2 分钟）
  → L/T = 1.64

  经典结论：带死区时间 L 的一阶系统，PID 稳定的必要条件约为
           L/T < 0.5（无微分项时可放宽到 ~1.0）
  → 1.64 远超稳定边界，且 KD=0.30 的微分项使情况恶化：
    微分项对"陈旧误差的变化率"求导，等于在放大一个 8 分钟前的趋势

  震荡周期的理论值 ≈ 2 × (L + T) = 2 × 792 s = 26.4 分钟
  实测 25 分钟 —— 与理论吻合，确认是反馈延迟导致的极限环
```

第二重：**`tick()` 从未校验反馈数据的新鲜度。** 它信任 `metrics.form_share()` 返回的是"刚刚的占比"，而这个假设在监控管道劣化时静默失效。第三重：**λ 触达限幅边界时没有告警。** `_clamp` 把失控的输出裁剪成了"看起来合法"的值——限幅本是防护措施，却掩盖了正在发生的失控。

**检测机制：**

```python
class ControlLoopHealthMonitor:
    """控制回路健康度监控：检测震荡、限幅触边、反馈陈旧

    这三项是所有闭环控制器的通用故障模式，与被控对象无关。
    本监控独立于控制器进程运行，读 form_shadow_price 表的时序。
    """

    OSCILLATION_WINDOW = 8         # 检查最近 8 个控制周期（40 分钟）
    SIGN_FLIP_ALERT = 4            # 误差符号翻转 ≥4 次判定为震荡
    CLAMP_STREAK_ALERT = 3         # 连续 3 周期触边即告警
    FEEDBACK_AGE_ALERT_SEC = 120   # 反馈延迟超过 0.4×控制周期告警
    AMPLITUDE_ALERT = 0.03         # 实际占比峰谷差超过 3pp 判定为震荡

    def check(self, scene="feed"):
        findings = []
        for form in ("ARTICLE", "SHORT", "LONG", "LIVE"):
            series = self.mysql.query("""
                SELECT tick_at, error, integral, lambda_after,
                       actual_share, target_share, floor_triggered
                  FROM form_shadow_price
                 WHERE scene = %s AND form = %s
                 ORDER BY tick_at DESC LIMIT %s
            """, (scene, form, self.OSCILLATION_WINDOW))
            if len(series) < self.OSCILLATION_WINDOW:
                continue

            # 1. 震荡检测：误差符号翻转次数 + 实际占比振幅
            #    单看符号翻转会误报（正常收敛也会在目标附近小幅翻转），
            #    必须与振幅联合判断 —— 小振幅的符号翻转是健康的收敛态
            flips = sum(1 for a, b in zip(series, series[1:])
                        if a['error'] * b['error'] < 0)
            shares = [r['actual_share'] for r in series]
            amplitude = max(shares) - min(shares)
            if flips >= self.SIGN_FLIP_ALERT and amplitude >= self.AMPLITUDE_ALERT:
                findings.append(self._alert(
                    "control_oscillation", form=form, flips=flips,
                    amplitude=round(amplitude, 4),
                    period_min=round(2 * self.OSCILLATION_WINDOW * 5 / max(flips, 1), 1)))

            # 2. 限幅触边：λ 长期贴边说明控制器已失去调节能力
            #    这是"控制器还在跑但已经不起作用"的唯一可观测信号
            streak = 0
            for r in series:
                at_bound = (r['lambda_after'] <= ShadowPriceController.LAMBDA_MIN + 1e-6
                            or r['lambda_after'] >= ShadowPriceController.LAMBDA_MAX - 1e-6)
                if not at_bound:
                    break
                streak += 1
            if streak >= self.CLAMP_STREAK_ALERT:
                findings.append(self._alert(
                    "shadow_price_saturated", form=form, streak=streak,
                    lambda_value=series[0]['lambda_after'],
                    hint="模型预估偏移超出控制器量程，λ 已无调节余量"))

            # 3. 积分饱和：积分项贴住限幅，说明长期偏差未被消除
            if abs(series[0]['integral']) >= ShadowPriceController.INTEGRAL_CLAMP - 1e-6:
                findings.append(self._alert(
                    "integral_windup", form=form, integral=series[0]['integral']))

        # 4. 反馈新鲜度：与被控对象无关的独立检查，最先触发
        age = self.metrics.pipeline_lag_sec("impression_stream")
        if age > self.FEEDBACK_AGE_ALERT_SEC:
            findings.append(self._alert(
                "feedback_stale", age_sec=age,
                hint=f"反馈延迟 {age}s 与控制周期 300s 之比 "
                     f"{age / 300:.2f}，>0.5 时 PID 不再稳定"))
        return findings
```

**解决方案：** 控制器必须对自己的输入做**有效性校验**，并在输入不可信时**冻结而非调整**。

```python
class ShadowPriceControllerV2(ShadowPriceController):
    """加固后的影子价格控制器

    三条加固原则：
      1. 反馈不新鲜时冻结输出（宁可不调，不可乱调）
      2. 死区抑制噪声驱动的无效调整
      3. 检测到震荡时自动增益退让，而不是等人来调参
    """

    MAX_FEEDBACK_AGE_SEC = 150     # 0.5 × 控制周期，超过则冻结
    DEADBAND = 0.005               # 误差 < 0.5pp 不调整
    GAIN_BACKOFF = 0.5             # 震荡时增益折半
    GAIN_RECOVER = 1.15            # 稳定后逐步恢复，恢复比退让慢
    MAX_STEP_RATIO = 0.25          # 单周期 λ 变化幅度上限 25%

    def tick(self):
        # ---- 前置门禁 1：反馈新鲜度 ----
        # 这是整个加固里最关键的一行。陈旧反馈下的"正确控制算法"
        # 会稳定地产生错误输出 —— 算法没问题，输入的时间语义错了。
        age = self.metrics.pipeline_lag_sec("impression_stream")
        if age > self.MAX_FEEDBACK_AGE_SEC:
            self.alarm.fire("shadow_price_frozen_stale_feedback",
                            age_sec=age, threshold=self.MAX_FEEDBACK_AGE_SEC)
            self._persist_tick(frozen=True, reason="stale_feedback")
            return self.lambda_          # 保持上一周期的 λ 不变

        # ---- 前置门禁 2：样本量充足性 ----
        # 低流量时段（凌晨 4 点）的占比统计本身方差极大，
        # 用它调整会把统计噪声当成系统偏差
        sample = self.metrics.impression_count(window_sec=self.CONTROL_PERIOD_SEC)
        if sample < self.MIN_SAMPLE_FOR_CONTROL:      # 200 万次曝光
            self._persist_tick(frozen=True, reason="insufficient_sample")
            return self.lambda_

        actual = self.metrics.form_share(window_sec=self.CONTROL_PERIOD_SEC)
        new_lambda, saturated = {}, []

        for form, target in self.TARGET_SHARE.items():
            err = target - actual.get(form, 0.0)

            # ---- 死区：小误差不调整 ----
            # 没有死区时，控制器会不停地追逐统计噪声，
            # 表现为 λ 持续小幅抖动，下游缓存反复失效
            if abs(err) < self.DEADBAND:
                new_lambda[form] = self.lambda_[form]
                self.integral[form] *= 0.9        # 死区内缓慢泄放积分
                self.prev_err[form] = err
                continue

            self.integral[form] = self._clamp(
                self.integral[form] + err * self.CONTROL_PERIOD_SEC / 3600.0,
                -self.INTEGRAL_CLAMP, self.INTEGRAL_CLAMP)

            # ---- 微分项：仅在反馈足够新鲜时启用 ----
            # 微分项是对趋势求导，而趋势的时间戳一旦不可信，
            # 微分就是在放大一个过期的方向
            if age <= self.CONTROL_PERIOD_SEC * 0.2:
                deriv = ((err - self.prev_err[form])
                         / (self.CONTROL_PERIOD_SEC / 3600.0))
            else:
                deriv = 0.0
            self.prev_err[form] = err

            g = self.gain_scale[form]             # 震荡退让后的增益系数
            adjust = g * (self.KP * err
                          + self.KI * self.integral[form]
                          + self.KD * deriv)
            raw = self.lambda_[form] * (1.0 + adjust / max(target, 1e-6))

            # ---- 步长限制：单周期变化不超过 25% ----
            # 这让"从正常值跳到限幅边界"至少需要 4 个周期，
            # 给监控留出反应窗口 —— 突变是不可观测的，渐变是可观测的
            lo = self.lambda_[form] * (1 - self.MAX_STEP_RATIO)
            hi = self.lambda_[form] * (1 + self.MAX_STEP_RATIO)
            lam = self._clamp(self._clamp(raw, lo, hi),
                              self.LAMBDA_MIN, self.LAMBDA_MAX)

            if lam <= self.LAMBDA_MIN + 1e-6 or lam >= self.LAMBDA_MAX - 1e-6:
                saturated.append(form)

            # ---- 保量下限仍然绕过全部平滑机制 ----
            # 死区、步长限制、增益退让都是为"可逆偏差"设计的平滑措施；
            # 生态保护是不可逆的，必须保留直达通道（见前文 §形态影子价格）
            if actual.get(form, 0.0) < self.FLOOR_SHARE[form]:
                lam = self.LAMBDA_MAX
                self.alarm.fire("form_share_below_floor", form=form,
                                actual=actual.get(form), floor=self.FLOOR_SHARE[form])

            new_lambda[form] = lam

        # ---- 震荡自适应：连续符号翻转则增益退让 ----
        self._update_gain_scale()

        # ---- 限幅饱和告警：控制器已失去权威，必须让人知道 ----
        if saturated:
            self.alarm.fire("shadow_price_saturated", forms=saturated,
                            hint="λ 触边，形态偏差已超出控制器量程，"
                                 "需检查上游模型预估分布是否漂移")

        mean = sum(new_lambda.values()) / len(new_lambda)
        new_lambda = {f: v / mean for f, v in new_lambda.items()}
        self.lambda_ = new_lambda
        self.config_center.publish("feed.form_shadow_price", new_lambda,
                                   ttl_sec=self.CONTROL_PERIOD_SEC * 4)
        self._persist_tick(frozen=False, reason="")
        return new_lambda

    def _update_gain_scale(self):
        """震荡检测与增益退让：让控制器自己降低攻击性

        退让快、恢复慢（0.5 vs 1.15）是刻意的不对称 ——
        震荡的代价（用户体验抖动）远大于收敛慢的代价。
        """
        for form in self.TARGET_SHARE:
            hist = self.err_history[form][-6:]
            flips = sum(1 for a, b in zip(hist, hist[1:]) if a * b < 0)
            if flips >= 3:
                self.gain_scale[form] = max(0.25,
                                            self.gain_scale[form] * self.GAIN_BACKOFF)
                self.metrics.incr("shadow_price.gain_backoff", tags={"form": form})
            elif flips == 0:
                self.gain_scale[form] = min(1.0,
                                            self.gain_scale[form] * self.GAIN_RECOVER)
```

**四项加固措施：**

| 措施 | 具体做法 | 拦住本次事故的哪一环 |
|------|---------|-------------------|
| 反馈新鲜度门禁 | `pipeline_lag > 150 s` 则冻结 λ，保持上周期值 | **根本性拦截**：8.2 分钟延迟直接触发冻结，震荡不会发生 |
| 单周期步长限制 25% | λ 从正常值到限幅需 ≥4 个周期 | 把突变变成渐变，监控有 20 分钟反应窗口 |
| 限幅触边告警 | 连续 3 周期贴边即 P1 告警 | 限幅不再掩盖失控 |
| 震荡自适应增益退让 | 6 周期内符号翻转 ≥3 次则增益折半 | 兜底：即使前三项失效，振幅也会自行收敛 |

**关键设计：控制器必须校验反馈数据的时间语义，而不只是数值有效性。** 本次事故里 `form_share()` 返回的每一个数字都是"合法的占比"——在 0 到 1 之间、四项加和为 1、无 NaN。任何数值校验都会放行。**错的是它的时间戳，而时间戳恰恰是唯一没被校验的维度。** 判据：**闭环控制系统的输入校验必须包含"这个观测有多旧"，因为控制算法的稳定性证明全部建立在"反馈及时"这一隐含前提上——而这个前提由监控管道的健康度决定，不由控制器自己决定。**

**这条推广到所有反馈型系统**：自动扩缩容（依赖负载指标）、动态限流（依赖 QPS 统计）、自适应超时（依赖延迟分位）、本域的池晋升阈值（依赖流入量预测）——全部有同一个失效模式。**凡是"读指标、做决策、影响指标"的闭环，指标管道的延迟就是系统的死区时间，必须被显式度量并纳入决策门禁。**

### 场景 2：投稿洪峰击穿内容池容量假设，冷启动评估质量静默崩塌

**场景描述：** 平台三周年活动上线，投稿激励翻倍，日投稿量从 1,345 万涨到 4,100 万（3.05 倍），持续 9 天。L0 冷启动池的流量占比是固定的 5%（16 亿次曝光/日），于是单条冷启动曝光量从 119 次掉到 **39 次**。全部在线指标毫无变化——曝光总量恒定、CTR 持平、人均时长持平、接口成功率 100%。9 天后活动结束，运营复盘时发现：活动期投稿的内容进入 L3 大流量池的比例只有平时的 41%，而这批内容在后续自然分发中的表现与平时投稿**没有统计差异**——说明不是内容变差了，是**筛选机制失效了**。

**根因分析：** 单条样本量从 119 掉到 39，直接击穿了 L0 硬淘汰判据的统计前提。

```
L0 硬淘汰线 play_rate < 0.15 的误杀率随样本量变化
（真实播放率 0.30 的正常内容被误判为 KILLED 的概率）

  n = 119：期望播放数 35.7，std = √(119×0.3×0.7) = 4.99
           淘汰阈值 = 119 × 0.15 = 17.85
           z = (17.85 − 35.7) / 4.99 = −3.58   → P ≈ 0.017%

  n = 39： 期望播放数 11.7，std = √(39×0.3×0.7)  = 2.86
           淘汰阈值 = 39 × 0.15 = 5.85
           z = (5.85 − 11.7) / 2.86  = −2.05    → P ≈ 2.0%

  误杀率放大 118 倍
  绝对量：4,100 万 × 2.0% = 82 万条/日 被误杀（平时 0.23 万条/日）
```

第二重失效在 `dynamic_threshold()`。它用 `inflow_estimator.predict()` 预测未来 1 小时流入量，而这个预测器是用常态数据训练的，对 3 倍洪峰的预测滞后约 5 小时。预测偏低 → `rate = capacity / inflow` 偏高 → 晋升阈值偏低 → **L1 池被塞进 2.7 倍内容**。L1 的流量占比同样固定（8%），于是 L1 单条曝光量从 1,219 掉到 451，L1→L2 的判断跟着失准，误差沿池链条逐级传递放大。

第三重是可观测性缺口：**唯一会动的数字是 `pool_traffic_budget.avg_per_content`。** 前文 §数据库设计 里把这一列物化并建 `idx_avg_alarm` 索引正是为此——但当时只建了索引，没建告警规则。

**检测机制：**

```python
class PoolStatisticalPowerMonitor:
    """内容池统计功效监控：把"评估质量"本身变成一等 SLO

    核心思路：池的健康度不是"曝光量是否达标"（那永远达标，
    因为流量占比是固定的），而是"单条样本量是否足以支撑判据"。
    这个监控计算的是判据的统计功效，而不是业务指标。
    """

    # 各池判据所需的最小样本量，由判据要检出的效应量反推
    REQUIRED_N = {
        0: 100,      # L0：检出 50% vs 15%（Δ=35pp），α=0.001/power=0.9 需 48，取 100 留余量
        1: 480,      # L1：检出 50% vs 35%（Δ=15pp）需 260，取 480
        2: 1568,     # L2：检出 50% vs 45%（Δ=5pp）需 1,568
        3: 1568,
    }
    POWER_FLOOR = 0.80           # 统计功效下限，跌破即告警
    MISKILL_RATE_ALERT = 0.005   # L0 误杀率超 0.5% 告警

    def check(self, scene="feed"):
        rows = self.mysql.query("""
            SELECT pool_level, budget_impressions, actual_impressions,
                   content_count, avg_per_content, inflow_count, promote_threshold
              FROM pool_traffic_budget
             WHERE scene = %s AND stat_hour = DATE_FORMAT(NOW(), '%%Y-%%m-%%d %%H:00:00')
        """, (scene,))

        findings = []
        for r in rows:
            level = r['pool_level']
            need = self.REQUIRED_N.get(level)
            if need is None:
                continue
            actual_n = float(r['avg_per_content'])

            # 1. 样本量充足性 —— 最直接的指标
            if actual_n < need:
                findings.append(self._alert(
                    "pool_sample_starved", level=level,
                    actual_n=round(actual_n, 1), required_n=need,
                    ratio=round(actual_n / need, 2),
                    ci_halfwidth_pp=round(
                        100 * 1.96 * math.sqrt(0.25 / max(actual_n, 1)), 1),
                    hint="池内容量超出流量承载，判据已失去统计效力"))

            # 2. L0 误杀率：把统计量翻译成业务语言，这是能让人行动的形式
            #    "样本量 39 < 100" 不会让人紧张，"每天误杀 82 万条" 会
            if level == 0:
                mk = self._miskill_rate(actual_n, true_rate=0.30,
                                        kill_threshold=PoolPromotionJudge.L0_KILL_PLAY_RATE)
                if mk > self.MISKILL_RATE_ALERT:
                    findings.append(self._alert(
                        "l0_miskill_rate_high", level=0,
                        miskill_rate=round(mk, 5),
                        daily_miskilled=int(mk * r['content_count']),
                        hint="降低淘汰阈值或扩大 L0 流量占比，二者择一"))

            # 3. 流入速率突变：领先指标，比样本量劣化早约 1 小时
            baseline = self._inflow_baseline(level, days=7)
            if baseline and r['inflow_count'] > baseline * 1.8:
                findings.append(self._alert(
                    "pool_inflow_surge", level=level,
                    inflow=r['inflow_count'], baseline=int(baseline),
                    surge_ratio=round(r['inflow_count'] / baseline, 2)))
        return findings

    @staticmethod
    def _miskill_rate(n, true_rate, kill_threshold):
        """正态近似计算误杀率：真实播放率为 true_rate 的内容，
        在 n 次曝光下被判定为 play_rate < kill_threshold 的概率"""
        if n < 1:
            return 1.0
        sd = math.sqrt(n * true_rate * (1 - true_rate))
        if sd == 0:
            return 0.0
        z = (n * kill_threshold - n * true_rate) / sd
        return normal_cdf(z)
```

**解决方案：** 两条互补的路径，核心认知是**评估容量是一种有限资源**。

```python
class PoolAdmissionController:
    """L0 准入控制：把"等比例摊薄"改为"排队准入"

    这是本场景最关键的设计决策。面对供给洪峰有两种选择：

      A. 等比例摊薄（原实现）：全部内容立即进 L0，每条分到 39 次曝光
         → 代价：全体内容的评估质量下降，误杀率放大 118 倍
         → 这个代价落在"所有创作者"头上，且不可见

      B. 排队准入（本实现）：按承载力放行，超出部分排队
         → 代价：部分内容的评估被延迟数小时
         → 这个代价落在"部分内容"头上，可度量、可告知、可补偿

    B 优于 A 的理由：评估质量的下降是不可恢复的（内容被误杀就没了），
    而评估延迟是可恢复的（晚几小时进池，结果一样）。
    判据：在"降低全体质量"与"延迟部分处理"之间，永远选后者。
    """

    MIN_IMPRESSIONS_PER_CONTENT = 100    # 设计约束：24h 内 ≥ 100 次曝光
    ELASTIC_MAX_SHARE = 0.09             # L0 流量占比的弹性上限（常态 5%）
    QUEUE_MAX_WAIT_SEC = 20 * 3600       # 排队上限 20 小时，留 4 小时余量满足 24h 约束
    PRIORITY_CREATOR_TIERS = ("S", "A")  # 高等级创作者优先出队

    def plan_hour(self, scene="feed"):
        """每小时执行一次：决定本小时放行多少内容进 L0"""
        # 1. 本小时 L0 可用曝光量（含弹性借调）
        total = self.forecast.hourly_impressions(scene)
        base_share = self.policy.pool_share(scene, level=0)          # 0.05
        pending = self.queue.size() + self.forecast.hourly_new_content()

        # 2. 先算按基线占比能承载多少条
        capacity_base = int(total * base_share / self.MIN_IMPRESSIONS_PER_CONTENT)

        # 3. 承载不足时，向 L3/L4 借流量（弹性扩张）
        #    为什么从 L3/L4 借：它们的单条曝光量是 9.9 万与 70.9 万，
        #    削减 10% 对单条评估质量的影响可忽略（样本量远超判据需求），
        #    而这 10% 对 L0 是成倍的增益。
        #    判据：跨池调剂流量时，从"样本量远超需求"的池借，
        #          边际损失接近零 —— 统计功效对样本量是饱和的。
        share = base_share
        if pending > capacity_base:
            need_share = pending * self.MIN_IMPRESSIONS_PER_CONTENT / total
            share = min(self.ELASTIC_MAX_SHARE, max(base_share, need_share))
            if share > base_share:
                self.policy.borrow_share(scene, from_levels=(3, 4),
                                         to_level=0, amount=share - base_share)
                self.metrics.gauge("pool.l0.elastic_share", share)

        capacity = int(total * share / self.MIN_IMPRESSIONS_PER_CONTENT)

        # 4. 仍然不足则排队 —— 这是与原实现的本质区别
        admitted = self.queue.pop_n(capacity, priority=self._priority_key)
        overflow = pending - len(admitted)
        if overflow > 0:
            self.metrics.gauge("pool.l0.queue_depth", self.queue.size())
            self.alarm.fire("l0_admission_queued", overflow=overflow,
                            queue_depth=self.queue.size(),
                            est_wait_min=self._est_wait_min(),
                            hint="投稿洪峰超出评估承载力，已排队而非摊薄")

        # 5. 排队超时保护：接近 24h 约束时无条件放行并降低判据强度
        #    这一步是诚实的降级 —— 明确标记这批内容"未经充分评估"，
        #    而不是假装评估过了
        expired = self.queue.pop_expired(self.QUEUE_MAX_WAIT_SEC)
        for c in expired:
            self.pool_state.mark_low_confidence(c.content_id, reason="admission_timeout")
        return admitted + expired

    def _priority_key(self, content):
        """出队优先级：先验置信度高的先进池

        理由：先验强的内容（S/A 级创作者）本来就主要靠先验晋升，
        L0 的后验只占 20% 权重（PRIOR_STRENGTH=480），
        它们从 L0 快速通过的信息损失最小。
        而先验弱的新人内容更依赖真实曝光反馈，值得等一个样本量充足的窗口。
        """
        tier = self.creator_stats.tier(content.author_id)
        return (0 if tier in self.PRIORITY_CREATOR_TIERS else 1,
                -self.pool_state.prior_mean(content.content_id),
                content.created_at)
```

配套地，硬淘汰判据必须**随样本量动态化**——用单侧置信下界而不是点估计：

```python
def should_kill(self, stat):
    """样本量自适应的硬淘汰判据

    原实现：play_rate < 0.15 则淘汰        —— 点估计，样本量敏感
    新实现：播放率的 99.9% 单侧置信上界 < 0.15 才淘汰 —— 样本量自适应

    效果：n=39 时置信区间宽，上界很难低于 0.15，自然不淘汰（宁可漏放）；
          n=119 时区间收窄，判据恢复原有效力。
    同一行代码在任何样本量下都保持误杀率 ≤ 0.1%，无需人工调阈值。
    """
    n, k = stat.impressions, stat.plays
    if n < PoolPromotionJudge.MIN_L0_IMPRESSIONS:
        return False
    # Wilson 上界（z = 3.09 对应单侧 99.9%）
    upper = wilson_upper_bound(k, n, z=3.09)
    return upper < PoolPromotionJudge.L0_KILL_PLAY_RATE
```

**四项加固措施：**

| 措施 | 具体做法 | 拦住本次事故的哪一环 |
|------|---------|-------------------|
| 统计功效 SLO 化 | `avg_per_content` 对比判据所需样本量，跌破即告警，并翻译成"日误杀条数" | 洪峰第 1 小时即告警（原本 9 天无感知） |
| L0 排队准入 | 按承载力放行，超出排队，20 小时超时兜底 | **根本性拦截**：单条曝光量恒定在 ≥100，判据不失效 |
| 流量占比弹性 | 洪峰时 L0 从 5% 弹性借到 9%（从 L3/L4 借） | 减少排队深度，把 3 倍洪峰的排队压到 1.2 小时 |
| 淘汰判据用置信下界 | Wilson 上界 < 0.15 才淘汰，替代点估计 | 兜底：即使样本量掉到 39，误杀率仍 ≤ 0.1% |

**关键设计：固定比例的资源分配 + 弹性的需求 = 静默的质量崩塌。** 这个模式极其常见且极难发现，因为**分配的资源总量始终"达标"**——L0 拿到了承诺的 5% 流量，一次不少。崩塌发生在"人均"这一层，而人均不是任何人的 KPI。

**判据：当一个系统的质量取决于"单位对象获得的资源量"，而资源总量固定、对象数量弹性时，必须在入口限流，不能等比例摊薄。** 同构的场景遍布各处：人工审核队列（审核员固定、送审量弹性 → 单条审核时长被摊薄 → 漏审率上升）、A/B 实验流量（实验位固定、实验数弹性 → 单实验样本量摊薄 → 结论不可信）、客服工单、灰度观察窗口。**这些系统的共同特征是：过载的表现不是拒绝服务，而是服务质量的无声下降——而无声的质量下降不会触发任何以"成功率"和"吞吐"为核心的监控体系。**

### 场景 3：精排模型迭代触发形态退化环，全部在线指标向好而生态崩塌

**场景描述：** 精排模型 v47 上线。离线评估全绿：全局 AUC 从 0.742 涨到 0.746（+0.4%），GAUC、NDCG@20、Logloss 全部改善。灰度 5% 观察 6 小时，在线指标全面向好：CTR +2.1%、人均时长 +1.3%、完播率 +3.8%、次日留存 +0.4%。实验平台判定显著正向，全量。

全量后 40 分钟，`form_share_below_floor` 告警触发：长视频曝光占比 6.8%，跌破保量下限 7%。值班同学核对了一圈——CTR、时长、留存、成功率、延迟全部正常或更好，唯一异常就是这条占比告警。他判定为"新模型让用户更爱看短视频，属预期内的结构变化"，静音了告警。

`ShadowPriceController` 按设计把 `λ_LONG` 一路拉到限幅 `LAMBDA_MAX = 2.60`，长视频曝光占比回升到 8.9%，看起来控制器把问题解决了。但接下来三周发生的事情是：

```
T+0     模型 v47 全量，长视频形态内排序质量塌陷（当时无人知晓）
T+40m   占比跌破 7% 告警 → 被判为误报静音
T+3h    λ_LONG 触限幅，曝光占比被强行拉回 8.9%（表面恢复）
T+2d    长视频实测完播率下降 11.2%，均播时长下降 8.4%
        → ValueEstimator 的形态修正系数在线校准据此下调 LONG 的 EEV
        → 长视频得分再降，需要更高的 λ 才能维持占比（λ 已在限幅）
T+5d    长视频实际占比滑到 7.4% 并持续下滑，控制器已无余量
T+11d   长视频创作者后台分发量周环比 −34%，客服工单出现"限流"投诉
T+21d   长视频日投稿从 5 万掉到 3.1 万（−38%）
        长视频消费时长占比从 24.6% 掉到 17.2%
        人均总时长首次转负（−0.9%）→ 此时才启动排查
        而模型已迭代到 v51，v47 早已不在线上，无法简单回滚
```

**根因分析：** 这正是挑战 2 描述的退化环真实发生，但触发点比预想的更隐蔽——**不是标签成熟度问题，而是全局 AUC 这个指标本身的结构缺陷。**

关键在于：**全局 AUC 是"形态间可分性"与"形态内可分性"的加权混合，而这个加权由样本对的数量分布决定，不由业务重要性决定。** 把全部正负样本对按"两条内容是否同形态"分成两类：

```
全局 AUC 的分层分解（按样本对类型）

  形态间对（正负样本来自不同形态）  占全部样本对 68%
  形态内对（正负样本来自同一形态）  占全部样本对 32%

  全局 AUC = 0.68 × AUC_形态间 + 0.32 × AUC_形态内

  v46（旧）：0.68 × 0.770 + 0.32 × 0.683 = 0.742
  v47（新）：0.68 × 0.792 + 0.32 × 0.648 = 0.746
                     ↑ +2.2pp            ↑ −3.5pp        ↑ +0.4pp

  → 新模型在 68% 的样本对上变好、在 32% 的样本对上变差，
    加权后全局指标上升 —— 而排序系统真正需要的恰恰是那 32%
```

再往下拆到单形态，问题彻底暴露：`AUC_形态内` 里长视频这一档从 0.681 掉到 0.612（**−10.1%**），短视频从 0.694 微升到 0.701。

为什么模型会这样学？因为**它是被目标函数诱导的，不是 bug**。训练用的是全局 pairwise 目标，每个样本对贡献相同的权重。模型容量有限时，优化器必然把容量投向能带来最多损失下降的方向——而 68% 的样本对是跨形态的。**"判断这条内容是不是短视频"是一个远比"判断这条长视频好不好"更容易学、且回报更高的任务。** v47 恰好新增了几个与形态强相关的上下文交叉特征，模型顺势把容量从"形态内区分好坏"迁移到了"形态间区分类型"。

第二重失效，也是危害最大的一重：**影子价格是形态级的一维乘数，它只能调整形态之间的相对位置，无法修复形态内部的排序塌陷。** 拉高 `λ_LONG` 的效果是"让更多长视频进入 20 个位置"，但**进来的是哪几条长视频，由已经塌陷的形态内排序决定**——接近随机抽取。于是长视频的实测消费表现必然变差。

第三重是自动化机制的反噬：`ValueEstimator` 的形态修正系数带在线校准（用实测 EEV 纠正预估偏差）。它观测到"长视频的实测完播率和时长确实下降了"，于是忠实地下调了长视频的价值系数。**校准机制无法区分"这个形态确实变差了"与"我们对这个形态的排序能力变差了"——这两种情况在数据上完全一样。** 于是自动校准把模型退化学成了形态退化，形成二阶正反馈：排序塌陷 → 实测变差 → 价值系数下调 → 得分更低 → 需要更大的 λ → λ 已限幅 → 占比下滑。

第四重是组织性失效，且它是可预测的：**当一条告警与其他所有指标的方向矛盾时，它一定会被静音。** 值班同学的推理没有错——在他能看到的全部证据下，"用户更爱看短视频"是最合理的解释。缺的不是判断力，是**告警本身没有携带"我预期与业务指标背离"这一元信息**。

**检测机制：** 核心只有一件事——**把 AUC 按形态分层度量，并把分层 AUC 作为模型上线的一等门禁**。逻辑本身很简单，重要的是它必须是阻断性的而不是观察性的：

```python
class StratifiedAUCGate:
    """模型上线门禁：分层 AUC 不得回退

    全局 AUC 上升不能作为放行依据 —— 它允许"提升形态间可分性、
    牺牲形态内可分性"这种对排序系统有害的交易。
    真正的门禁指标是每个形态内部的 AUC。
    """

    WITHIN_FORM_DROP_LIMIT = 0.005    # 任一形态的形态内 AUC 回退超 0.5pp 即阻断
    LOW_SHARE_FORMS = ("LONG", "LIVE")  # 曝光占比小的形态，样本少、最容易被牺牲
    LOW_SHARE_DROP_LIMIT = 0.002      # 对它们用更严的阈值（0.2pp）

    def evaluate(self, new_model, base_model, eval_set):
        report, blockers = {}, []
        for form in ("ARTICLE", "SHORT", "LONG", "LIVE"):
            # 关键：只用同形态的样本对计算 AUC，把形态间可分性完全排除
            pairs = eval_set.within_form_pairs(form)
            new_auc = pairwise_auc(new_model, pairs)
            base_auc = pairwise_auc(base_model, pairs)
            drop = base_auc - new_auc
            limit = (self.LOW_SHARE_DROP_LIMIT if form in self.LOW_SHARE_FORMS
                     else self.WITHIN_FORM_DROP_LIMIT)
            report[form] = {"new": new_auc, "base": base_auc, "drop": drop}
            if drop > limit:
                blockers.append((form, round(drop, 4), limit))

        # 全局 AUC 只作为参考记录，不参与放行判断
        report["_global"] = {"new": pairwise_auc(new_model, eval_set.all_pairs()),
                             "base": pairwise_auc(base_model, eval_set.all_pairs())}
        return {"passed": not blockers, "blockers": blockers, "report": report}
```

配套的三项监控，都是在线的：**λ 限幅触边**（场景 1 已实现，本场景下它是"控制器已失去权威"的唯一信号）；**形态内在线排序质量**——按形态分别统计"曝光位次与实际消费的秩相关"，长视频这一项在 v47 上线 40 分钟内就从 0.31 掉到 0.19；**供给侧周指标**——分形态投稿量、创作者活跃数、头部创作者留存，它们是最终受害者，也是最迟钝的指标，但必须存在，因为它们是唯一能证明"这不是用户偏好变化而是系统故障"的证据。

**解决方案：** 四条，按拦截时序排列。

第一，**分层 AUC 门禁前置到模型上线前**，且对低占比形态用更严的阈值。理由是不对称的：长视频占 9% 曝光，其样本对只占形态内对的一小部分，全局指标对它的损失几乎不敏感——**越是"全局指标不敏感"的子群体，越需要专门的门禁保护它**。

第二，**λ 触限幅时冻结形态修正系数的在线校准**。这是打断二阶反馈环的关键一步，逻辑用几行就能说清：

```python
def should_freeze_calibration(self, form):
    """λ 贴住限幅时，实测 EEV 不再是"这个形态的真实价值"的无偏观测

    因为此时形态的曝光是被控制器强行拉上去的，进入曝光的内容
    由（可能已塌陷的）形态内排序决定 —— 观测样本被系统自身污染了。
    继续校准就是把自己的故障学成对世界的认识。
    """
    saturated = self.shadow_price.saturated_periods(form)      # 连续触边周期数
    return saturated >= 3
```

**这条原则可以推得更远：任何"用系统自身产出的数据来校准系统自身参数"的闭环，都必须有一个开关，在系统处于异常态时切断学习。** 否则系统会把故障期的数据当作正常观测，把暂时的失效固化成永久的先验——这是所有自学习系统最危险的失效模式，且它造成的损害在故障修复后依然存在。

第三，**告警必须携带预期的指标背离说明**。`form_share_below_floor` 的告警文案改为显式声明：

```
【P1】长视频曝光占比 6.8% 跌破保量下限 7%

⚠ 本告警预期与消费侧指标方向相反：CTR / 人均时长 / 完播率
  很可能同时向好。这是本告警的正常表现，不是误报依据。
  形态占比下滑的收益体现在当期消费指标，代价体现在 3-6 周后的
  供给侧（投稿量、创作者留存），当期指标无法反映它。

❌ 不得以"业务指标正常"为由静音本告警。
✓ 排查路径：① 分形态 AUC 是否回退 ② λ 是否触限幅
            ③ 近 7 日该形态投稿量趋势
✓ 静音需二级审批，并强制填写 3 周后的复核时间
```

这不是文档问题，是**告警设计问题**。一条与其他所有指标矛盾的告警，如果不自带"为什么它会矛盾"的解释，等于没有这条告警。

第四，**实验准入必须包含结构指标**，且结构指标不能与消费指标做加权平均——必须是独立的一票否决项。具体清单：分形态曝光占比偏移不得超过 ±1.5pp、分形态内 AUC 不得回退、λ 不得触边、分形态投稿量的 21 天趋势纳入长周期复核。

**四项加固措施：**

| 措施 | 具体做法 | 拦住本次事故的哪一环 |
|------|---------|-------------------|
| 分层 AUC 上线门禁 | 形态内 AUC 回退 >0.5pp（低占比形态 >0.2pp）即阻断 | **根本性拦截**：v47 的 LONG 形态内 AUC −6.9pp，上线前即被拒 |
| λ 触边冻结在线校准 | 连续 3 周期贴边则停止形态系数校准 | 打断二阶反馈环，把 21 天的不可逆损害压到可回滚范围 |
| 告警自带背离说明 | 明确声明"预期与消费指标相反"，静音需二级审批 | 避免组织性静音，把检测延迟从 21 天压到 40 分钟 |
| 结构指标一票否决 | 占比偏移 / 分层 AUC / λ 饱和三项独立门禁，不参与加权 | 兜底：全量阶段仍会被拦下 |

**关键设计：全局指标是分层指标的加权混合，权重由样本量分布决定，因此它可以在任何一层退化的同时上升。** v47 不是一个坏模型，它在 68% 的样本对上确实更好——**它只是在一个错误的目标上被优化，而这个错误由指标定义引入，不由模型引入。** 判据：**当系统对 N 个子群体分别做决策时，模型质量必须按子群体分别度量并分别设门禁。** 如果一个模型的放行依据里只有全局指标，那么它必然会以牺牲小样本子群体为代价换取全局数字——这不是可能发生，而是**优化器的必然行为**，因为那正是你让它最大化的东西。

**这条陷阱的适用范围远超推荐系统**：多语言 NLP 模型（高资源语言吞掉低资源语言的容量）、多地域风控模型（大市场吞掉小市场）、多品类预测模型、任何做了"统一大模型替代 N 个专用模型"的重构。**统一模型的收益是真实的，但它的代价永远落在样本量最小的那个子群体上，而全局指标在结构上看不见这个代价。**
