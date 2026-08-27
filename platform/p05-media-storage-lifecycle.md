# P05: 媒资存储与生命周期

## 业务场景

某统一内容平台的媒资存储域，管理运营第 3 年积累的 **534 PB 稳态逻辑存量**。这个域的唯一使命是**在满足读取延迟与可靠性要求的前提下，把每 GB 的月成本压到最低**——它是平台第二大成本项（占 20.3%，仅次于带宽的 73.0%）。

**已知数据：**
- 稳态逻辑存量：534 PB（长视频原片 137 PB + 图文 22 PB + 短视频原片 17 PB + 转码副本 53 PB + 直播转点播 15 PB + 其余素材 290 PB）
- 日入库：原片 0.6 PB + 转码副本 0.145 PB = 0.745 PB/天
- 日读取：媒体分发回源 4.8 PB（CDN 命中率 95%，即 96 PB 出流量中 5% 回源）
- 对象数量：约 1,240 亿个（含分片，平均对象 4.3 MB）
- 访问倾斜：1% 的内容承载 80% 的读取；90% 的内容 7 日读取次数 < 100
- 读取延迟要求：Hot < 30 ms、Warm < 120 ms、Cold < 800 ms（首字节）
- 可靠性要求：年数据丢失概率 < 1e-11（相当于 11 个 9 的持久性）
- 成本目标：≤ ¥1,258 万/月（等效逻辑单价 ¥0.0236/GB/月）
- 删除时效：用户删除后 24 小时内全副本不可读，7 天内物理擦除

**为什么存储成本容易被严重低估？**

三个叠加的认知陷阱，每一个都会让估算偏低数倍：

| 陷阱 | 错误算法 | 正确算法 | 偏差 |
|------|---------|---------|------|
| 用年增量代替稳态存量 | 只算今年新增的 569 PB | 算累积存量（第 3 年 534 PB，第 5 年约 830 PB） | 随年份放大 |
| 忘记冗余系数 | 534 PB × 单价 | 534 PB × **冗余 3.0x** × 单价 | **3 倍** |
| 用逻辑单价掩盖介质差异 | 统一 ¥0.12/GB/月 | 分介质：NVMe ¥0.040 / HDD ¥0.020 / 归档 ¥0.012（**物理**容量单价） | 3.3 倍 |

三者叠加：朴素估算 534 PB × ¥0.0236 = ¥1,258 万（正确的优化后值）被误当成"不优化也就这么多"，而**不优化的真实值是 ¥6,409 万——差 5 倍**。

**核心矛盾**：**成本与可访问性直接对立。** 让数据更便宜的每一个手段（降级介质、提高 EC 条带宽度、下沉到归档层）都同时让它更慢、更难恢复、更易在故障时不可用。而访问模式是**长尾且会突变**的——一条沉底两年的视频可能因一次热点在 10 分钟内变成全站最热。存储分层必须在"按当前热度优化成本"和"为热度突变保留弹性"之间取舍。

## 核心挑战

### 挑战 1：介质分层与 EC 的收益顺序被普遍搞反

工程师直觉认为 EC 是存储省钱的主要手段（"冗余从 3.0x 降到 1.25x，省 58%！"）。实算恰好相反：

| 手段 | 作用对象 | 节省 |
|------|---------|------|
| 介质分层（NVMe → 归档 HDD） | 85% 数据的**单价**（¥0.040 → ¥0.012，降 70%） | **¥4,197 万/月** |
| 冷层 EC(16,4) | 冷层的**冗余系数**（3.0x → 1.25x） | ¥953 万/月 |

**介质分层的收益是 EC 的 4.4 倍。** 搞反顺序的后果是：团队花半年上 EC（技术复杂、故障域大、恢复慢），却没先做成本最低、风险最小的介质分层。

### 挑战 2：冷热判据——热度不仅长尾，还会"复活"

访问分布极端倾斜（1% 内容占 80% 读取），这支持激进地把 85% 数据下沉到冷层。但冷层的代价是首字节 800 ms 与降级读需拉 16 个块。

问题在于**热度会复活**：

- 一条沉底两年的视频因社会热点被搬上热搜，10 分钟内从 0 QPS 涨到 8,000 QPS
- 一部老剧因续集上映，整季 40 集同时回暖
- 一个创作者突然爆火，其历史全部内容被翻出（可能上千条）

如果这些数据在冷层，用户体验是首帧从 800 ms 起步、且大量降级读把冷层集群打满。**冷层不是为高 QPS 设计的**——它的 IOPS 密度只有热层的 1/40。

难点：热度复活的**检测必须快于流量爬升**，而回迁（Cold → Hot）本身要读全量数据并重新编码，一条 4K 长视频回迁需 6 分钟、消耗大量带宽。

### 挑战 3：转码副本回收的误杀风险

P04 的 JIT 策略把转码副本从 350 PB 压到 53 PB，靠的是"7 日播放为 0 的非兜底档位直接删除"。但这个规则有三个误杀场景：

- **周期性内容**：每周更新的节目，第 8 天回收了第 1 天的档位，第 9 天用户追更时全部 JIT 重转
- **地域时差**：一条只在特定时区受欢迎的内容，统计窗口错位导致误判
- **季节性内容**：春节相关内容在 11 个月里播放为 0，第 12 个月爆发

误杀的代价不是数据丢失（可 JIT 重建），而是**JIT 风暴**——大量档位同时被重建，转码集群被挤满，新投稿转码 P95 击穿。

### 挑战 4：删除的完整性与合规

一份媒资在系统里有多少个副本？远超直觉：

```
一条长视频的物理副本清点：
  原片            × 3 副本（Hot）                      =  3
  转码产物 6 档   × 冷层 EC(16,4) 20 块              = 120 块
  跨地域灾备      × 2 地域                            × 2
  CDN 边缘缓存    × 约 180 个节点（命中过的）          = 180
  客户端离线下载  × 未知数量（用户设备上）             = ?
  转码中间产物    （正常已清理，异常时残留）            = ?
  数仓/备份快照   × 保留周期内的所有快照               = ?
  ─────────────────────────────────────────────────
  合计：数百个物理位置，其中 3 类不在自己的机房里
```

个保法与版权要求"删除即不可访问、限期内物理擦除"。漏掉任何一类都构成合规风险，而**最容易漏的恰是不在自己机房的那三类**（CDN 边缘、客户端离线、第三方备份）。

### 挑战 5：秒传与去重的正确性

秒传（客户端算 hash，命中则跳过上传）是省带宽与存储的重要手段——实测约 11% 的上传能命中去重（同一素材被多人搬运、同一创作者多端重复上传）。

但去重引入共享，共享引入引用计数，引用计数是**最容易出错的机制之一**：

- 计数泄漏（该减没减）→ 数据永不回收，存储持续泄漏
- 计数早减（不该减减了）→ 数据被误删，其他引用方的内容变成死链
- hash 碰撞 → 两个不同文件被当成同一个（SHA-256 碰撞概率可忽略，但**截断 hash 或弱 hash 不可忽略**）
- 并发（一方在删、一方在引用）→ 竞态导致上面两种错误

## 设计约束

- 稳态成本 ≤ ¥1,258 万/月（等效逻辑单价 ¥0.0236/GB/月）
- 读取首字节：Hot < 30 ms、Warm < 120 ms、Cold < 800 ms
- 年数据丢失概率 < 1e-11
- 热度复活的回迁必须在流量爬升到冷层容量上限前完成（约 90 秒窗口）
- 删除后 24 小时内全副本不可读，7 天内物理擦除，且过程可审计
- 任一单机房故障不导致数据不可读（跨地域至少 1 份可用副本）

## 请先独立思考（限时 40 分钟）

1. 534 PB 你会怎么分层？各层的判据是什么（按时间？按访问频次？按内容类型？）流转是单向下沉还是双向？
2. EC 参数 (k, m) 怎么选？条带越宽冗余越低，但代价是什么？为什么不用 EC(17,3) 把冗余压到 1.18x？
3. 一条冷层视频突然上热搜，10 分钟内 QPS 从 0 涨到 8,000。你怎么在流量爬升前把它回迁到热层？检测信号是什么？
4. 秒传的引用计数如何保证"不泄漏、不早减、并发安全"？如果计数错了，你怎么发现？

---

## 设计解析

### 三层存储架构：介质、冗余、访问路径

先给结论架构，再逐项论证与给实施步骤。

```
                     ┌─────────────────────────────────┐
   读请求 ──────────▶│  媒资访问网关 (Go)               │
                     │  · 按 asset_id 查分层元数据      │
                     │  · Hot/Warm 直读，Cold 走取回队列 │
                     │  · 热度打点（每次读上报）         │
                     └──────┬──────────────────────────┘
                            │
        ┌───────────────────┼───────────────────────┐
        ▼                   ▼                       ▼
┌───────────────┐  ┌───────────────┐  ┌──────────────────────┐
│ Hot  (3%)     │  │ Warm (12%)    │  │ Cold (85%)           │
│ 16.0 PB 逻辑  │  │ 64.1 PB 逻辑  │  │ 453.9 PB 逻辑        │
│ 3 副本 = 48PB │  │ 3 副本 =192PB │  │ EC(16,4) = 567 PB    │
│ NVMe SSD      │  │ HDD 7.2K      │  │ 归档 HDD (SMR)       │
│ ¥0.040/GB物理 │  │ ¥0.020/GB物理 │  │ ¥0.012/GB物理        │
│ ¥192 万/月    │  │ ¥384 万/月    │  │ ¥681 万/月           │
│ 首字节 <30ms  │  │ 首字节 <120ms │  │ 首字节 <800ms        │
│ IOPS 密度 1.0 │  │ 密度 0.15     │  │ 密度 0.025           │
└───────┬───────┘  └───────┬───────┘  └──────────┬───────────┘
        │                  │                     │
        │  ◀── 回迁（热度复活，90 秒内完成）─────┤
        └── 下沉（热度衰减，低峰期批量）────────▶│
```

**关键设计：三层的差异不只是单价，更是 IOPS 密度——冷层的 IOPS 密度只有热层的 1/40。这意味着冷层放不下热数据，不是因为慢，而是因为会被打满。**

#### 介质选型的论证

| 维度 | NVMe SSD | HDD 7.2K | 归档 HDD (SMR) | 磁带 (LTO-9) |
|------|---------|----------|---------------|-------------|
| 物理单价（¥/GB/月，含机架电力运维摊销） | 0.040 | 0.020 | 0.012 | 0.003 |
| 首字节延迟 | 0.2 ms | 8 ms | 15 ms | **40-180 秒**（需装载） |
| 单盘 IOPS | 500,000 | 180 | 120 | — |
| 单盘容量 | 7.68 TB | 20 TB | 26 TB | 45 TB |
| 顺序吞吐 | 6 GB/s | 260 MB/s | 190 MB/s | 400 MB/s |
| 随机小文件 | 极优 | 差 | **极差**（SMR 覆写惩罚） | 不可用 |
| 重建时间（单盘） | 1.5 h | 14 h | **26 h** | — |

**选择：Hot 用 NVMe、Warm 用 HDD 7.2K、Cold 用归档 HDD（SMR），不引入磁带**，理由：
1. 三档介质的单价梯度（0.040 / 0.020 / 0.012）配合访问倾斜（1% / 12% / 85% 的容量占比），加权单价降到 ¥0.0138/GB 物理——这是介质分层贡献 ¥4,197 万/月的来源
2. SMR 盘的覆写惩罚对媒资**恰好无害**——媒资是**一次写入、多次读取、永不修改**（WORM 语义）。SMR 的劣势场景是随机覆写，而这在媒资域根本不存在。用别人的劣势盘拿自己的优势价格
3. 不用磁带：单价虽然只有归档 HDD 的 1/4（省 ¥510 万/月），但首字节 40-180 秒完全无法满足 Cold 层 800 ms 的要求。磁带只适合"合规留存但几乎不读"的数据，而本平台的冷层仍有 5% 的读取占比

**但需要注意：** SMR 盘的顺序写要求意味着**写入必须由存储层做日志结构化聚合**——不能让上层随机写。实施上表现为：冷层只接受"整对象顺序写入"，且下沉任务必须按大块（≥ 256 MB）批量搬迁，不能逐个小对象搬。

#### 实施步骤：三层架构落地（12 周）

```
阶段 1（第 1-2 周）：元数据与打点先行——不动数据
  1.1 media_asset 表增加 storage_tier / tier_changed_at / heat_score 字段
  1.2 上线读取打点：媒资网关每次读取上报 (asset_id, ts, bytes, region)
      → Kafka → Flink 聚合成分钟级热度
  1.3 建立热度基线：观察 2 周，得到真实的访问分布曲线
  ★ 检验点：热度分布是否符合 1%/12%/85% 的假设？不符合则调整分层比例

阶段 2（第 3-5 周）：Warm 层上线——先做最安全的一档
  2.1 部署 HDD 集群（容量 = 预期 Warm 逻辑量 × 3 副本 × 1.3 余量）
  2.2 实现下沉任务：Hot → Warm，双写验证期 2 周
      · 先复制到 Warm，校验 checksum，读流量切到 Warm，观察 72h
      · 无异常后删除 Hot 副本
  2.3 实现回迁任务：Warm → Hot
  ★ 检验点：Warm 层首字节 P99 < 120 ms？下沉/回迁的数据校验零差异？

阶段 3（第 6-9 周）：Cold 层上线（不含 EC，先用 3 副本）
  3.1 部署归档 HDD 集群，仍用 3 副本（先验证介质与访问路径，不叠加 EC 风险）
  3.2 实现 Cold 取回队列 + 降级读兜底
  3.3 大规模下沉：按内容年龄从老到新，每天不超过总量的 2%
  ★ 检验点：此时已拿到 ¥4,197 万/月 的介质分层收益（EC 还没上）
     ——这是"先分层再 EC"的实际含义：8 周拿到 81% 的收益

阶段 4（第 10-12 周）：Cold 层改 EC(16,4)
  4.1 在新写入的冷数据上启用 EC，存量保持 3 副本
  4.2 存量按批转换（每批 5 PB，转换后校验可读性 + 模拟 4 块丢失的恢复）
  4.3 EC 与 3 副本共存期间，元数据记录每个对象的冗余方式
  ★ 检验点：模拟故障演练——随机杀 4 个节点，验证数据仍全部可读
```

**关键设计：EC 放在最后一个阶段，且前 3 阶段就已拿到 81% 的成本收益。** 这个排序让项目在第 9 周就能证明价值，即使第 4 阶段因技术风险延期或取消，也不影响主体收益。**把高收益低风险的事排在前面，是让长周期基础设施项目能活下来的关键。**

### EC 参数选择：为什么是 (16,4) 而不是 (17,3)

EC(k, m) 把对象切 k 个数据块 + m 个校验块，冗余系数 (k+m)/k，可容忍任意 m 块丢失。

| 参数 | 冗余 | 容错 | 跨节点 | 降级读需拉 | 冷层物理容量 | 月成本 |
|------|------|------|-------|-----------|------------|-------|
| 3 副本 | 3.000x | 2 | 3 | 1 | 1,362 PB | ¥1,634 万 |
| EC(4,2) | 1.500x | 2 | 6 | 4 | 681 PB | ¥817 万 |
| EC(8,3) | 1.375x | 3 | 11 | 8 | 624 PB | ¥749 万 |
| EC(12,4) | 1.333x | 4 | 16 | 12 | 605 PB | ¥726 万 |
| **EC(16,4)** | **1.250x** | **4** | **20** | **16** | **567 PB** | **¥681 万** |
| EC(17,3) | 1.176x | 3 | 20 | 17 | 534 PB | ¥641 万 |
| EC(20,4) | 1.200x | 4 | 24 | 20 | 545 PB | ¥654 万 |

**选择：EC(16,4)**，理由：
1. **容错 4 块是可靠性底线。** 目标年丢失概率 < 1e-11。归档 HDD 年故障率约 2.8%，26 小时的重建窗口内再坏 m 块的概率随 m 增大而急剧下降。实算容错 3 块时年丢失概率约 4e-11（不达标），容错 4 块约 6e-13（达标）。**EC(17,3) 虽然冗余更低，但容错只有 3 块，不满足可靠性要求——这是它被否决的决定性原因，而非成本。**
2. 20 个节点的故障域可接受。EC(20,4) 跨 24 节点，单对象受更多节点影响，尾延迟更差且集群扩容时的数据迁移量更大
3. 降级读拉 16 块 vs EC(12,4) 的 12 块——冷层读取占比仅 5%，且有 Warm 层与 CDN 挡在前面，降级读的绝对次数可控
4. 相比 EC(12,4) 每月多省 ¥45 万，相比 EC(8,3) 多省 ¥68 万

**但需要注意：** EC 的三个隐性代价必须提前设计，否则上线后会遇到麻烦：

| 隐性代价 | 表现 | 应对 |
|---------|------|------|
| 小对象放大 | 一个 1 MB 对象切 16 块，每块 64 KB，小于盘的最优 IO 单元，浪费 IOPS | **小对象不进 EC**：< 4 MB 的对象打包成 ≥ 256 MB 的大块后整块 EC |
| 重建放大 | 丢 1 块要读 16 块才能重建，重建流量是丢失量的 16 倍 | 限速重建 + 优先重建容错余量最小的条带 |
| 尾延迟 | 单对象读需 16 个节点都响应，任一慢节点拖累 P99 | 读 k+2 块（多读 2 块），先到 k 块即可解码，抛弃慢节点 |

第三项（多读 2 块）是最有效的实用技巧——用 12.5% 的额外读流量换掉长尾节点的影响，把 Cold 层读取 P99 从 2,100 ms 压到 780 ms。

#### 关键代码：EC 编解码与小对象打包

```python
import hashlib
import os


class ErasureCodedStore:
    """EC 存储层——冷层的读写实现

    三个关键设计：
      1. 小对象先打包（packing）再 EC，避免小块浪费 IOPS
      2. 读时多拉 2 块（hedged read），先到 k 块即解码，规避慢节点
      3. 写入前后都校验 checksum，防静默损坏
    """

    K = 16                      # 数据块数
    M = 4                       # 校验块数
    STRIPE_UNIT = 4 * 1024 * 1024      # 单块 4 MB → 单条带 64 MB 数据
    PACK_THRESHOLD = 4 * 1024 * 1024   # 小于此的对象走打包路径
    PACK_TARGET_SIZE = 256 * 1024 * 1024   # 打包目标大小 256 MB
    HEDGE_EXTRA = 2             # 多读的块数

    def __init__(self, node_pool, metadata_store, codec, config):
        self.nodes = node_pool          # 节点池，负责选节点与健康状态
        self.meta = metadata_store
        self.codec = codec              # Reed-Solomon 编解码器（C++ 加速）
        self.pack_buffer = config['pack_buffer']   # 打包缓冲区（持久化）

    # ── 写入 ──

    def put(self, asset_id, variant_key, data):
        """
        写入一个对象。小对象走打包，大对象直接 EC。
        返回定位信息，供 metadata 记录。
        """
        if len(data) < self.PACK_THRESHOLD:
            return self._put_packed(asset_id, variant_key, data)
        return self._put_striped(asset_id, variant_key, data)

    def _put_striped(self, asset_id, variant_key, data):
        """大对象：直接按条带 EC"""
        checksum = hashlib.sha256(data).hexdigest()
        stripes = []

        for stripe_idx, offset in enumerate(
                range(0, len(data), self.STRIPE_UNIT * self.K)):
            chunk = data[offset:offset + self.STRIPE_UNIT * self.K]

            # 1. 补齐到 K 个等长块（最后一条带需 padding）
            blocks = self._split_padded(chunk, self.K, self.STRIPE_UNIT)

            # 2. Reed-Solomon 编码出 M 个校验块
            parity = self.codec.encode(blocks, self.K, self.M)
            all_blocks = blocks + parity          # 共 K+M = 20 块

            # 3. 选 20 个不同节点（强制跨机架，同机架不超过 M 块）
            #    这条约束保证单机架整体故障时仍能恢复
            targets = self.nodes.select(
                count=self.K + self.M,
                max_per_rack=self.M,
                exclude_unhealthy=True)

            # 4. 并发写 20 块，每块带自己的 checksum
            written = self._write_blocks(all_blocks, targets, asset_id,
                                        variant_key, stripe_idx)

            # 5. 写入成功判据：至少 K+1 块成功（留 1 块余量再触发补写）
            if len(written) < self.K + 1:
                self._rollback(written)
                raise RuntimeError(
                    f"EC 写入失败：仅 {len(written)}/{self.K + self.M} 块成功")

            stripes.append({
                'stripe_index': stripe_idx,
                'blocks': written,
                'data_len': len(chunk),
            })

            # 6. 不足 K+M 块时，异步补写缺失块（不阻塞返回）
            if len(written) < self.K + self.M:
                self._schedule_repair(asset_id, variant_key, stripe_idx,
                                      missing=self.K + self.M - len(written))

        location = {
            'mode': 'ec_striped',
            'k': self.K, 'm': self.M,
            'stripe_unit': self.STRIPE_UNIT,
            'stripes': stripes,
            'total_len': len(data),
            'checksum': checksum,
        }
        self.meta.put_location(asset_id, variant_key, location)
        return location

    def _put_packed(self, asset_id, variant_key, data):
        """
        小对象：先追加到打包缓冲区，缓冲区满 256 MB 后整块 EC。
        对象在包内的位置记录为 (pack_id, offset, length)。
        """
        checksum = hashlib.sha256(data).hexdigest()

        # 1. 追加到当前打包缓冲（缓冲区本身是持久化的，防丢）
        pack_id, offset = self.pack_buffer.append(data)

        # 2. 元数据先记为 "packed_pending"——此时读取走缓冲区
        location = {
            'mode': 'ec_packed',
            'pack_id': pack_id,
            'offset': offset,
            'length': len(data),
            'checksum': checksum,
            'sealed': False,
        }
        self.meta.put_location(asset_id, variant_key, location)

        # 3. 缓冲区达阈值 → 封包并 EC（此时才真正落冷层）
        if self.pack_buffer.size(pack_id) >= self.PACK_TARGET_SIZE:
            self._seal_pack(pack_id)

        return location

    def _seal_pack(self, pack_id):
        """封包：把 256 MB 打包块作为一个大对象走 EC 路径"""
        pack_data = self.pack_buffer.read_all(pack_id)
        pack_location = self._put_striped('__pack__', pack_id, pack_data)

        # 更新包内所有对象的元数据为 sealed，指向包的 EC 位置
        members = self.pack_buffer.list_members(pack_id)
        self.meta.batch_update_pack(pack_id, pack_location, members)
        self.pack_buffer.release(pack_id)

    # ── 读取 ──

    def get(self, asset_id, variant_key, range_start=None, range_end=None):
        """读取对象。支持 Range 读（只解码涉及的条带）"""
        loc = self.meta.get_location(asset_id, variant_key)
        if loc is None:
            raise KeyError(f"{asset_id}/{variant_key} 不存在")

        if loc['mode'] == 'ec_packed':
            return self._get_packed(loc, range_start, range_end)
        return self._get_striped(loc, range_start, range_end)

    def _get_striped(self, loc, range_start, range_end):
        """
        条带读。关键优化：hedged read —— 发起 K+2 个请求，
        先到 K 个就解码，不等慢节点。
        """
        stripe_bytes = loc['stripe_unit'] * loc['k']
        start = range_start or 0
        end = range_end if range_end is not None else loc['total_len']

        # 只读涉及的条带（Range 读的关键——不必读全对象）
        first_stripe = start // stripe_bytes
        last_stripe = (end - 1) // stripe_bytes

        out = bytearray()
        for si in range(first_stripe, last_stripe + 1):
            stripe = loc['stripes'][si]
            blocks = self._read_stripe_hedged(stripe, loc['k'], loc['m'])
            decoded = self.codec.decode(blocks, loc['k'], loc['m'])
            out += decoded[:stripe['data_len']]

        # 截取 Range 内的部分
        offset_in_out = start - first_stripe * stripe_bytes
        return bytes(out[offset_in_out:offset_in_out + (end - start)])

    def _read_stripe_hedged(self, stripe, k, m):
        """
        并发读 k+HEDGE_EXTRA 块，先到 k 块即返回。
        这把 Cold 层 P99 从 2,100 ms 压到 780 ms——
        代价是 12.5% 的额外读流量。
        """
        import concurrent.futures

        # 优先选健康度高、负载低的块
        candidates = self.nodes.rank_by_health(stripe['blocks'])
        to_read = candidates[:k + self.HEDGE_EXTRA]

        collected = {}
        with concurrent.futures.ThreadPoolExecutor(max_workers=len(to_read)) as ex:
            futures = {ex.submit(self._read_block, b): b for b in to_read}
            for fut in concurrent.futures.as_completed(futures):
                b = futures[fut]
                try:
                    data = fut.result()
                except Exception:
                    continue        # 单块失败正常，靠冗余兜住
                if data is None:
                    continue
                collected[b['block_index']] = data
                # 攒够 k 块立即返回，剩余请求让它们自然结束
                if len(collected) >= k:
                    break

        if len(collected) < k:
            # 不足 k 块 → 该条带真的不可读，触发紧急修复
            self._schedule_repair(stripe.get('asset_id'), None,
                                  stripe['stripe_index'], urgent=True)
            raise RuntimeError(
                f"条带 {stripe['stripe_index']} 仅 {len(collected)}/{k} 块可读")

        return collected

    def _read_block(self, block):
        """读单块并校验 checksum——防静默数据损坏"""
        node = self.nodes.get(block['node_id'])
        data = node.read(block['path'])
        if data is None:
            return None
        actual = hashlib.crc32(data) & 0xFFFFFFFF
        if actual != block['crc32']:
            # 静默损坏：立即上报并隔离该块，返回 None 走冗余恢复
            self.nodes.report_corruption(block['node_id'], block['path'])
            return None
        return data

    # ── 辅助 ──

    def _split_padded(self, chunk, k, unit):
        """切成 k 个等长块，不足补零"""
        blocks = []
        for i in range(k):
            piece = chunk[i * unit:(i + 1) * unit]
            if len(piece) < unit:
                piece = piece + b'\x00' * (unit - len(piece))
            blocks.append(piece)
        return blocks

    def _write_blocks(self, blocks, targets, asset_id, variant_key, stripe_idx):
        """并发写块，返回成功的块信息"""
        import concurrent.futures
        written = []

        def write_one(idx_node):
            idx, node = idx_node
            data = blocks[idx]
            path = (f"ec/{asset_id}/{variant_key}/s{stripe_idx:05d}"
                    f"/b{idx:02d}")
            node.write(path, data)
            return {
                'block_index': idx,
                'node_id': node.node_id,
                'path': path,
                'crc32': hashlib.crc32(data) & 0xFFFFFFFF,
            }

        with concurrent.futures.ThreadPoolExecutor(max_workers=len(targets)) as ex:
            futures = [ex.submit(write_one, (i, n))
                       for i, n in enumerate(targets)]
            for fut in concurrent.futures.as_completed(futures):
                try:
                    written.append(fut.result())
                except Exception:
                    continue
        return written

    def _rollback(self, written):
        """写入失败的清理——避免残留块占用空间"""
        for b in written:
            try:
                self.nodes.get(b['node_id']).delete(b['path'])
            except Exception:
                pass    # 清理失败交给后台孤儿块巡检

    def _schedule_repair(self, asset_id, variant_key, stripe_idx,
                         missing=1, urgent=False):
        """加入修复队列。urgent 表示已接近不可读，优先处理"""
        self.meta.push_repair_task({
            'asset_id': asset_id,
            'variant_key': variant_key,
            'stripe_index': stripe_idx,
            'missing_blocks': missing,
            'urgent': urgent,
        })
```

**关键设计：`_read_stripe_hedged` 的"多读 2 块，先到 k 块即解码"是整个 EC 读路径最重要的一行优化。** EC 读取的固有问题是需要 k 个节点全部响应，尾延迟等于最慢节点的延迟。多读 2 块把"必须等最慢的第 16 个"变成"等最快的 16 个中的任意 16 个"，P99 从 2,100 ms 降到 780 ms。**这是分布式读取的通用技巧：用少量冗余请求换掉长尾。**

### 热度模型与分层流转：应对"热度复活"

回到挑战 2。分层的判据不能是简单的"内容年龄"，因为热度会复活。

#### 为什么按年龄分层是错的

```
两种分层判据在真实访问分布上的表现（抽样 1,000 万条内容 30 天）：

判据                    冷层命中率   冷层误放率   热度复活漏检
────────────────────────┼──────────┼──────────┼────────────
按内容年龄（>90天→冷）      82%        18%         41%
按累计播放量（<1000→冷）    88%        12%         33%
按近7日播放量               94%         6%         28%
按加权热度分（本文方案）      97%         3%          4%
```

"冷层误放率"指本该在热层的内容被放到冷层（用户遭遇 800 ms 首字节）。"热度复活漏检"指复活的内容未被及时回迁。**按年龄分层的漏检率 41%，意味着每 10 条复活内容有 4 条一直留在冷层被反复降级读——这是冷层集群被打满的主因。**

#### 加权热度分的设计

热度分要同时表达三件事：**最近有多热、衰减多快、有没有在加速**。

```
heat_score = w1 · 近1h读取(归一化)
           + w2 · 近24h读取(归一化)
           + w3 · 近7d读取(归一化)
           + w4 · 加速度项(近1h速率 / 近24h平均速率)
           + w5 · 内容类型基线(长视频长尾更长，基线更高)

权重（离线用"分层决策正确率"作目标回归得到）：
  w1 = 0.42   ← 近1h权重最高，这是捕捉"复活"的关键
  w2 = 0.24
  w3 = 0.11
  w4 = 0.18   ← 加速度项，让"正在爬升"的内容提前回迁
  w5 = 0.05

分层阈值：
  heat_score ≥ 0.60  → Hot
  0.18 ≤ score < 0.60 → Warm
  heat_score < 0.18   → Cold
```

**加速度项（w4）是应对热度复活的核心。** 一条刚开始爬升的内容，其近 1h 读取绝对值可能还很小（比如从 0 涨到 200 QPS），但**加速度极高**（近1h速率 / 近24h平均 = 无穷大）。只看绝对值会漏检，看加速度能在流量爬升的**早期**就触发回迁——这是抢到 90 秒回迁窗口的关键。

#### 关键代码：热度计算与分层决策

```python
import math
import time


class HeatScorer:
    """媒资热度打分器——驱动分层流转决策

    数据来源：媒资网关的每次读取打点 → Kafka → Flink 聚合成
    (asset_id, 1h读取, 24h读取, 7d读取) 写入 Redis。
    本类只做打分与阈值判定，不负责数据采集。
    """

    W_1H, W_24H, W_7D, W_ACCEL, W_TYPE = 0.42, 0.24, 0.11, 0.18, 0.05

    HOT_THRESHOLD = 0.60
    WARM_THRESHOLD = 0.18

    # 归一化基准：达到此读取量即视为满分 1.0（按经验分位数定）
    NORM_1H, NORM_24H, NORM_7D = 500, 6000, 30000

    # 内容类型基线（长视频长尾更长，给更高基线避免过早下沉）
    TYPE_BASELINE = {
        'long_video': 0.32,
        'short_video': 0.08,     # 短视频 48h 衰减 90%，基线低
        'article': 0.20,
        'live_replay': 0.12,
        'image': 0.15,
    }

    def __init__(self, redis_client, config):
        self.redis = redis_client
        self.accel_cap = config.get('accel_cap', 8.0)   # 加速度项上限，防除零爆炸

    def score(self, asset_id, content_type):
        """计算热度分。返回 (score, 建议层, 明细)"""
        stats = self._read_stats(asset_id)

        n1h = min(1.0, stats['read_1h'] / self.NORM_1H)
        n24h = min(1.0, stats['read_24h'] / self.NORM_24H)
        n7d = min(1.0, stats['read_7d'] / self.NORM_7D)

        # 加速度：近1h速率 相对 近24h平均速率的倍数
        avg_hourly_24h = stats['read_24h'] / 24.0
        if avg_hourly_24h < 0.5:
            # 近24h几乎没读 → 只要近1h有读就是强复活信号
            accel = self.accel_cap if stats['read_1h'] >= 5 else 0.0
        else:
            accel = min(self.accel_cap, stats['read_1h'] / avg_hourly_24h)
        n_accel = accel / self.accel_cap

        baseline = self.TYPE_BASELINE.get(content_type, 0.15)

        score = (self.W_1H * n1h + self.W_24H * n24h + self.W_7D * n7d
                 + self.W_ACCEL * n_accel + self.W_TYPE * baseline)

        tier = ('hot' if score >= self.HOT_THRESHOLD
                else 'warm' if score >= self.WARM_THRESHOLD
                else 'cold')

        return score, tier, {
            'n1h': round(n1h, 4), 'n24h': round(n24h, 4), 'n7d': round(n7d, 4),
            'accel': round(accel, 2), 'baseline': baseline,
        }

    def _read_stats(self, asset_id):
        """从 Redis 读聚合好的读取统计"""
        raw = self.redis.hgetall(f"heat:{asset_id}")
        return {
            'read_1h': int(raw.get(b'r1h', 0)),
            'read_24h': int(raw.get(b'r24h', 0)),
            'read_7d': int(raw.get(b'r7d', 0)),
        }

    def detect_revival(self, asset_id, content_type, current_tier):
        """
        专用的快速复活检测——比完整打分更激进，用于抢 90 秒窗口。
        每 15 秒对冷层活跃对象跑一遍（只跑近1h有读取的，量很小）。
        """
        if current_tier == 'hot':
            return None

        stats = self._read_stats(asset_id)
        r1h = stats['read_1h']
        avg_hourly_24h = stats['read_24h'] / 24.0

        # 三个独立的复活信号，任一触发即回迁（宁可多回迁，不可漏）
        signals = []

        # 信号1：绝对量突破——近1h读取超过冷层单对象承受上限
        if r1h >= 300:
            signals.append(('ABSOLUTE_SPIKE', r1h))

        # 信号2：加速度突破——相对24h基线放大 10 倍以上
        if avg_hourly_24h >= 0.5 and r1h / avg_hourly_24h >= 10:
            signals.append(('ACCELERATION', round(r1h / avg_hourly_24h, 1)))

        # 信号3：冷启动型复活——24h内几乎无读，但近1h突然有量
        if avg_hourly_24h < 0.5 and r1h >= 30:
            signals.append(('COLD_START_REVIVAL', r1h))

        if not signals:
            return None

        return {
            'asset_id': asset_id,
            'signals': signals,
            'target_tier': 'hot' if r1h >= 300 else 'warm',
            'urgency': 'immediate' if r1h >= 300 else 'normal',
        }
```

**为什么复活检测要独立于常规打分？** 因为两者的**执行频率与代价完全不同**：
- 常规打分：全量 1,240 亿对象，每 6 小时跑一遍，靠离线批处理
- 复活检测：只跑"近 1h 有读取的冷层对象"（约 40 万个），每 15 秒跑一遍

如果只靠常规打分，最坏情况要等 6 小时才发现复活——而流量爬升只需 10 分钟。**把"低频全量"与"高频增量"拆成两条路径，是所有大规模状态检测的通用模式。**

#### 分层流转执行器

```python
class TierMigrator:
    """分层流转执行器——下沉与回迁

    两个方向的设计原则完全不同：
      下沉：不着急，批量做，低峰期做，省成本优先
      回迁：抢时间，单对象立即做，占用带宽也要做，体验优先
    """

    # 下沉：批量、限速、低峰期
    DEMOTE_BATCH_SIZE = 500
    DEMOTE_DAILY_CAP_PB = 10.7          # 每天最多搬 2% 总量
    DEMOTE_WINDOW = (1, 7)               # 凌晨 1-7 点执行

    # 回迁：立即、不限速、允许抢占带宽
    PROMOTE_CONCURRENCY = 64
    PROMOTE_SLA_SEC = 90                 # 90 秒内完成，抢在流量爬升前

    # 抖动抑制：刚流转过的对象在冷静期内不再流转
    COOLDOWN_SEC = 3600

    def __init__(self, storage, metadata, scorer, redis_client, config):
        self.storage = storage
        self.meta = metadata
        self.scorer = scorer
        self.redis = redis_client
        self.alert = config['alert_service']

    # ── 回迁（Cold/Warm → Hot），时效优先 ──

    def promote(self, revival):
        """
        回迁单个对象。关键约束：90 秒内完成。
        实施要点：先复制，再切读，最后才删源——任何时刻都有可读副本。
        """
        asset_id = revival['asset_id']
        target = revival['target_tier']

        if not self._acquire_migration_lock(asset_id):
            return {'skipped': 'already_migrating'}

        started = time.time()
        try:
            loc = self.meta.get_location_all_variants(asset_id)

            # 1. 只回迁"实际被读的档位"，不回迁全部档位
            #    热点复活时用户集中看 720P，没必要把 4K 也搬上来
            hot_variants = self._get_recently_read_variants(asset_id)
            if not hot_variants:
                hot_variants = [self._default_variant(loc)]

            # 2. 并发复制到目标层（读冷层 + 写热层）
            copied = []
            for vk in hot_variants:
                data = self.storage.get_from_tier(asset_id, vk, 'cold')
                # 校验完整性——绝不把损坏数据搬到热层
                if not self._verify(data, loc[vk]['checksum']):
                    self.alert.fire("promote_checksum_mismatch", severity="P1",
                                    detail={'asset_id': asset_id, 'variant': vk})
                    continue
                self.storage.put_to_tier(asset_id, vk, data, target)
                copied.append(vk)

            if not copied:
                return {'failed': 'no_variant_copied'}

            # 3. 切换读路径（元数据原子更新）——此刻起读走热层
            self.meta.update_tier(asset_id, copied, target)

            # 4. 源副本不立即删除！保留 24 小时作为回退保险
            #    如果热层出问题，还能切回冷层
            self.redis.zadd("tier:pending_source_cleanup",
                            {f"{asset_id}": time.time() + 86400})

            elapsed = time.time() - started
            if elapsed > self.PROMOTE_SLA_SEC:
                self.alert.fire("promote_sla_breach", severity="P2", detail={
                    'asset_id': asset_id, 'elapsed_sec': round(elapsed, 1),
                    'variants': len(copied)})

            self._set_cooldown(asset_id)
            return {'promoted': copied, 'elapsed_sec': round(elapsed, 1)}

        finally:
            self._release_migration_lock(asset_id)

    # ── 下沉（Hot → Warm → Cold），成本优先 ──

    def demote_batch(self):
        """
        批量下沉。只在低峰窗口执行，且有日配额上限。
        实施要点：按"大块聚合"搬迁，适配 SMR 盘的顺序写要求。
        """
        hour = time.localtime().tm_hour
        if not (self.DEMOTE_WINDOW[0] <= hour < self.DEMOTE_WINDOW[1]):
            return {'skipped': 'outside_window'}

        moved_bytes = int(self.redis.get("tier:demote_today_bytes") or 0)
        cap_bytes = int(self.DEMOTE_DAILY_CAP_PB * 1e15)
        if moved_bytes >= cap_bytes:
            return {'skipped': 'daily_cap_reached'}

        # 1. 取候选：热度分低于阈值且不在冷静期
        candidates = self.meta.query_demote_candidates(
            limit=self.DEMOTE_BATCH_SIZE)

        # 2. 关键：按目标层 + 大小聚合成 ≥256MB 的批，顺序写冷层
        #    逐个小对象搬迁会让 SMR 盘的写放大到 3-5 倍
        batches = self._pack_into_batches(candidates,
                                          target_bytes=256 * 1024 * 1024)

        results = {'demoted': 0, 'bytes': 0, 'skipped': 0}
        for batch in batches:
            for item in batch:
                if self._in_cooldown(item['asset_id']):
                    results['skipped'] += 1
                    continue
                score, tier, _ = self.scorer.score(
                    item['asset_id'], item['content_type'])
                # 二次确认——候选是批量查出的，可能已过时
                if tier == item['current_tier']:
                    results['skipped'] += 1
                    continue

            ok_bytes = self._migrate_batch(batch)
            results['demoted'] += len(batch)
            results['bytes'] += ok_bytes

        self.redis.incrby("tier:demote_today_bytes", results['bytes'])
        return results

    def _migrate_batch(self, batch):
        """
        执行一批下沉。顺序：复制 → 校验 → 切读 → 延迟删源。
        与回迁相同的安全顺序，但可以慢慢做。
        """
        total = 0
        for item in batch:
            asset_id, vk = item['asset_id'], item['variant_key']
            src_tier, dst_tier = item['current_tier'], item['target_tier']

            data = self.storage.get_from_tier(asset_id, vk, src_tier)
            if not self._verify(data, item['checksum']):
                self.alert.fire("demote_checksum_mismatch", severity="P1",
                                detail={'asset_id': asset_id})
                continue

            self.storage.put_to_tier(asset_id, vk, data, dst_tier)
            self.meta.update_tier(asset_id, [vk], dst_tier)
            # 下沉的源副本保留 72 小时（比回迁更久，因为下沉更可能判断错）
            self.redis.zadd("tier:pending_source_cleanup",
                            {f"{asset_id}:{vk}:{src_tier}": time.time() + 259200})
            total += len(data)
            self._set_cooldown(asset_id)
        return total

    def cleanup_sources(self):
        """
        延迟删源：到期后才真正删除迁移前的副本。
        这个延迟窗口是分层流转最重要的安全网——
        任何判断错误都能在窗口内无损回退。
        """
        now = time.time()
        due = self.redis.zrangebyscore("tier:pending_source_cleanup", 0, now)
        deleted = 0
        for raw in due:
            key = raw.decode() if isinstance(raw, bytes) else raw
            parts = key.split(':')
            asset_id = parts[0]

            # 删前最后一次确认：目标层副本确实可读
            if not self._target_readable(asset_id, parts):
                self.alert.fire("cleanup_aborted_target_unreadable",
                                severity="P0", detail={'key': key})
                # 目标不可读 → 不删源，并把对象切回源层
                self.meta.rollback_tier(asset_id)
                self.redis.zrem("tier:pending_source_cleanup", raw)
                continue

            self.storage.delete_from_tier(asset_id, parts)
            self.redis.zrem("tier:pending_source_cleanup", raw)
            deleted += 1
        return {'deleted': deleted}

    # ── 抖动抑制 ──

    def _in_cooldown(self, asset_id):
        return self.redis.exists(f"tier:cooldown:{asset_id}")

    def _set_cooldown(self, asset_id):
        self.redis.setex(f"tier:cooldown:{asset_id}", self.COOLDOWN_SEC, "1")

    def _acquire_migration_lock(self, asset_id):
        """防止同一对象被并发迁移（下沉与回迁同时触发会导致数据错乱）"""
        return bool(self.redis.set(f"tier:lock:{asset_id}", "1",
                                   nx=True, ex=300))

    def _release_migration_lock(self, asset_id):
        self.redis.delete(f"tier:lock:{asset_id}")

    def _verify(self, data, expected_checksum):
        return (data is not None
                and hashlib.sha256(data).hexdigest() == expected_checksum)

    def _pack_into_batches(self, candidates, target_bytes):
        """把小对象聚合成大批，适配 SMR 顺序写"""
        batches, cur, cur_size = [], [], 0
        for c in sorted(candidates, key=lambda x: x['target_tier']):
            cur.append(c)
            cur_size += c['size_bytes']
            if cur_size >= target_bytes:
                batches.append(cur)
                cur, cur_size = [], 0
        if cur:
            batches.append(cur)
        return batches
```

**关键设计：迁移的四步顺序（复制 → 校验 → 切读 → 延迟删源）在两个方向上完全一致，且"延迟删源"是整个分层机制的安全网。** 任何时刻源副本与目标副本至少有一个可读；删源前还要再确认一次目标可读，不可读就回滚。**没有延迟删源窗口的分层系统，一次判断错误就是永久数据丢失。**

**抖动抑制（cooldown）同样不可省。** 没有它时，热度分在阈值附近震荡的对象会被反复搬迁——实测无 cooldown 时约 2.3% 的对象每天迁移 3 次以上，白白消耗 ¥40 万/月的迁移带宽且拉高延迟。1 小时冷静期把这个比例压到 0.04%。

### 转码副本回收：避免 JIT 风暴的误杀

回到挑战 3。回收规则"7 日播放为 0 的非兜底档位删除"有三类误杀场景，需要针对性豁免。

```python
class VariantRecycler:
    """转码副本回收器——把副本量从 350 PB 压到 53 PB，同时避免 JIT 风暴

    核心：不是简单的"7日无播放就删"，而是四层豁免规则 +
    回收后的 JIT 压力预算约束。
    """

    IDLE_DAYS_THRESHOLD = 7          # 基础判据：7 日无播放
    MIN_AGE_DAYS = 30                # 创建不足 30 天不回收（还在冷启动期）
    # 兜底档位永不回收——删了就没有任何可播档位
    PROTECTED_QUALITY = {360, 720}
    # 回收产生的 JIT 压力预算：每日重建量不超过转码集群 3% 算力
    JIT_BUDGET_DAILY_MINUTES = 12000

    def __init__(self, mysql_client, redis_client, config):
        self.mysql = mysql_client
        self.redis = redis_client
        self.alert = config['alert_service']

    def scan_and_recycle(self):
        """每日执行一次，在低峰期"""
        candidates = self._query_candidates()
        recycled, exempted = [], []
        jit_minutes_used = 0

        for c in candidates:
            reason = self._check_exemptions(c)
            if reason:
                exempted.append({'variant_id': c['variant_id'], 'reason': reason})
                continue

            # JIT 预算约束：估算这个档位若被重建需要多少转码算力
            rebuild_cost = self._estimate_rebuild_minutes(c)
            if jit_minutes_used + rebuild_cost > self.JIT_BUDGET_DAILY_MINUTES:
                # 超预算：今天不回收，留到明天
                exempted.append({'variant_id': c['variant_id'],
                                 'reason': 'JIT_BUDGET_EXHAUSTED'})
                continue

            self._recycle(c)
            recycled.append(c['variant_id'])
            jit_minutes_used += rebuild_cost

        return {
            'recycled': len(recycled),
            'exempted': len(exempted),
            'jit_budget_used_pct': round(
                jit_minutes_used / self.JIT_BUDGET_DAILY_MINUTES * 100, 1),
            'exemption_breakdown': self._group_reasons(exempted),
        }

    def _check_exemptions(self, c):
        """
        四层豁免规则。返回豁免原因，None 表示可回收。
        每条规则都对应一类真实的误杀场景。
        """
        # 豁免1：兜底档位——删了就没有可播档位，绝对不能删
        if c['quality_level'] in self.PROTECTED_QUALITY:
            return 'PROTECTED_FALLBACK_QUALITY'

        # 豁免2：太新——创建不足 30 天，还在冷启动分发期
        if c['age_days'] < self.MIN_AGE_DAYS:
            return 'TOO_YOUNG'

        # 豁免3：周期性内容——检测历史播放是否呈周期模式
        #        （每周更新的节目，7日窗口会误判为无播放）
        if self._is_periodic(c['asset_id']):
            return 'PERIODIC_CONTENT'

        # 豁免4：季节性内容——去年同期有显著播放
        #        （春节内容在 11 个月里播放为 0）
        if self._has_seasonal_pattern(c['asset_id']):
            return 'SEASONAL_CONTENT'

        # 豁免5：所属创作者近期爆火——其历史内容可能被批量翻出
        if self._author_recently_surged(c['author_id']):
            return 'AUTHOR_SURGE'

        return None

    def _is_periodic(self, asset_id):
        """
        周期性检测：取近 90 天的日播放序列，做自相关分析。
        若在 lag=7（周）或 lag=30（月）处自相关显著，判定为周期性。
        """
        series = self._daily_play_series(asset_id, days=90)
        if len(series) < 60 or sum(series) < 50:
            return False

        for lag in (7, 30):
            if self._autocorrelation(series, lag) > 0.45:
                return True
        return False

    def _autocorrelation(self, series, lag):
        """指定滞后的自相关系数"""
        n = len(series)
        if n <= lag:
            return 0.0
        mean = sum(series) / n
        var = sum((x - mean) ** 2 for x in series)
        if var <= 0:
            return 0.0
        cov = sum((series[i] - mean) * (series[i + lag] - mean)
                  for i in range(n - lag))
        return cov / var

    def _has_seasonal_pattern(self, asset_id):
        """季节性：去年同期(±14天)的播放量是否显著高于年均"""
        row = self.mysql.query("""
            SELECT
              AVG(CASE WHEN DAYOFYEAR(stat_date)
                       BETWEEN DAYOFYEAR(CURDATE())-14 AND DAYOFYEAR(CURDATE())+14
                       THEN play_count END) AS same_period_avg,
              AVG(play_count) AS yearly_avg
            FROM variant_daily_play
            WHERE asset_id=%s AND stat_date >= DATE_SUB(CURDATE(), INTERVAL 400 DAY)
        """, (asset_id,))
        if not row or not row['yearly_avg'] or row['same_period_avg'] is None:
            return False
        return row['same_period_avg'] > row['yearly_avg'] * 4

    def _author_recently_surged(self, author_id):
        """创作者近 3 日播放量是否为其 30 日均值的 5 倍以上"""
        surge = self.redis.get(f"author:surge_flag:{author_id}")
        return surge is not None

    def _estimate_rebuild_minutes(self, c):
        """估算该档位 JIT 重建所需转码分钟数（用于预算约束）"""
        # GPU 硬编约 0.3x 实时，按分辨率缩放
        dur_min = c['duration_ms'] / 60000.0
        scale = (c['quality_level'] / 1080.0) ** 1.8
        return dur_min * 0.3 * max(scale, 0.1)

    def _recycle(self, c):
        """
        执行回收。注意：不是物理删除，而是先标记 + 延迟物理删除。
        与分层流转同理——留回退窗口。
        """
        self.mysql.execute("""
            UPDATE media_variant
            SET status = 2, recycled_at = NOW()
            WHERE variant_id = %s AND status = 1
        """, (c['variant_id'],))

        # 物理删除延迟 7 天。这 7 天内若有 JIT 请求，直接"复活"元数据
        # 而无需重新转码——这是回收误判的免费保险
        self.redis.zadd("variant:pending_physical_delete",
                        {str(c['variant_id']): time.time() + 604800})

    def revive_if_pending(self, variant_id):
        """
        JIT 请求到来时先查：该档位是否在"已标记回收但未物理删除"状态？
        若是，直接恢复元数据，零成本。这条路径救回约 23% 的误判。
        """
        score = self.redis.zscore("variant:pending_physical_delete",
                                  str(variant_id))
        if score is None:
            return False

        affected = self.mysql.execute("""
            UPDATE media_variant SET status = 1, recycled_at = NULL
            WHERE variant_id = %s AND status = 2
        """, (variant_id,))
        if affected:
            self.redis.zrem("variant:pending_physical_delete", str(variant_id))
            # 记录误判，用于调优豁免规则
            self.redis.incr("variant:recycle_false_positive")
            return True
        return False

    def _query_candidates(self):
        return self.mysql.query("""
            SELECT v.variant_id, v.asset_id, v.quality_level, v.size_bytes,
                   a.duration_ms, a.author_id,
                   DATEDIFF(NOW(), v.created_at) AS age_days
            FROM media_variant v
            JOIN media_asset a ON a.asset_id = v.asset_id
            WHERE v.status = 1
              AND v.play_count_7d = 0
              AND v.gen_mode = 1
              AND DATEDIFF(NOW(), v.created_at) >= %s
            ORDER BY v.size_bytes DESC
            LIMIT 200000
        """, (self.MIN_AGE_DAYS,))

    def _daily_play_series(self, asset_id, days):
        rows = self.mysql.query("""
            SELECT stat_date, SUM(play_count) AS pc
            FROM variant_daily_play
            WHERE asset_id=%s AND stat_date >= DATE_SUB(CURDATE(), INTERVAL %s DAY)
            GROUP BY stat_date ORDER BY stat_date
        """, (asset_id, days))
        return [r['pc'] for r in rows]

    def _group_reasons(self, exempted):
        from collections import Counter
        return dict(Counter(e['reason'] for e in exempted))
```

**回收策略的效果对比：**

| 策略 | 回收量 | 误判率 | JIT 重建风暴 | 月节省 |
|------|-------|-------|------------|-------|
| 不回收 | 0 | — | 无 | ¥0 |
| 朴素（7 日无播放即删） | 312 PB | **18.4%** | **严重**（峰值占转码集群 31%） | ¥743 万 |
| + 兜底档位豁免 | 297 PB | 11.2% | 中等（19%） | ¥707 万 |
| + 周期/季节/爆火豁免 | 291 PB | 3.1% | 轻微（6%） | ¥693 万 |
| **+ JIT 预算约束 + 延迟物删** | **297 PB** | **0.9%** | **可控（≤3%）** | **¥699 万** |

**关键设计：最后一行的回收量反而回升到 297 PB，误判率却降到 0.9%。** 原因是"延迟物理删除 7 天"让误判可以零成本挽回（23% 的误判在这个窗口内被 JIT 请求救回），于是可以更激进地回收。**给决策留一个廉价的回退窗口，比把决策做得更准更有效。** 这与 P04 的"先廉价决策再按热度晋升"是同一个思路。

### 秒传与去重：引用计数的正确实现

回到挑战 5。约 11% 的上传能命中去重，年省约 24 PB 存储 + 大量上传带宽。但引用计数是最容易出错的机制之一。

#### 三个必须同时解决的正确性问题

| 问题 | 后果 | 根因 |
|------|------|------|
| 计数泄漏（该减没减） | 数据永不回收，存储持续泄漏 | 删除路径异常退出、消息丢失 |
| 计数早减（不该减减了） | 数据被误删，其他引用方内容变死链 | 并发删除、重复消费同一删除事件 |
| hash 碰撞 | 两个不同文件被当成同一个 | 用了弱 hash 或截断 hash |
| 引用-删除竞态 | A 在引用的同时 B 把计数减到 0 并删除 | 无原子性保护 |

#### 解决方案：二级校验 + 原子操作 + 对账兜底

```python
class MediaDeduplicator:
    """媒资去重与秒传——引用计数的正确实现

    三道防线：
      1. 强 hash（SHA-256 全长）+ 二级校验（大小 + 首尾块采样）防碰撞
      2. 原子引用计数（DB 行锁）+ 幂等键 防泄漏与早减
      3. 每日对账 兜住前两道的漏网
    """

    # 二级校验采样：除 SHA-256 外，再比对文件大小与首尾各 1MB 的 MD5
    # 这不是为了防 SHA-256 碰撞（概率可忽略），而是防"hash 计算实现有 bug"
    SAMPLE_SIZE = 1024 * 1024

    def __init__(self, mysql_client, redis_client, storage, config):
        self.mysql = mysql_client
        self.redis = redis_client
        self.storage = storage
        self.alert = config['alert_service']

    # ── 秒传检查 ──

    def check_instant_upload(self, content_hash, file_size,
                             head_md5, tail_md5):
        """
        客户端上传前调用。命中则返回 asset_id，客户端跳过上传。

        客户端需提供：SHA-256 全文 hash、文件大小、首尾 1MB 的 MD5。
        三者全部匹配才认定为同一文件——单靠 hash 不够稳健，
        因为客户端 hash 实现可能有 bug（分片顺序错、遗漏尾块等）。
        """
        row = self.mysql.query("""
            SELECT asset_id, src_size_bytes, head_md5, tail_md5, ref_count,
                   deleting
            FROM media_asset WHERE content_hash = %s
        """, (content_hash,))

        if not row:
            return {'hit': False}

        # 关键：正在删除中的资产不可被秒传命中
        # 否则新内容会引用一个即将被物理删除的资产
        if row['deleting']:
            return {'hit': False, 'reason': 'asset_deleting'}

        # 二级校验：任一不匹配则拒绝秒传，走正常上传
        if row['src_size_bytes'] != file_size:
            self.alert.fire("dedup_size_mismatch", severity="P1", detail={
                'content_hash': content_hash,
                'db_size': row['src_size_bytes'], 'client_size': file_size})
            return {'hit': False, 'reason': 'size_mismatch'}

        if row['head_md5'] != head_md5 or row['tail_md5'] != tail_md5:
            # 这是严重信号：SHA-256 相同但内容采样不同
            # 要么是 hash 实现有 bug，要么真碰撞（极不可能）
            self.alert.fire("dedup_sample_mismatch", severity="P0", detail={
                'content_hash': content_hash})
            return {'hit': False, 'reason': 'sample_mismatch'}

        return {'hit': True, 'asset_id': row['asset_id']}

    # ── 引用与解引用 ──

    def acquire_reference(self, asset_id, content_id, idempotency_key):
        """
        建立引用。必须与"内容落库"在同一事务内，否则会出现
        "内容已创建但引用未加"的泄漏（数据被回收后内容变死链）。

        idempotency_key 保证重复调用不重复加计数。
        """
        with self.mysql.transaction() as tx:
            # 1. 幂等：唯一键冲突说明已加过
            try:
                tx.execute("""
                    INSERT INTO media_asset_ref
                        (asset_id, content_id, idempotency_key, created_at)
                    VALUES (%s, %s, %s, NOW())
                """, (asset_id, content_id, idempotency_key))
            except Exception as e:
                if 'Duplicate entry' in str(e):
                    return {'ok': True, 'idempotent': True}
                raise

            # 2. 原子递增计数，并用 deleting=0 作为条件
            #    这一条件是防"引用-删除竞态"的关键：
            #    若删除流程已把 deleting 置 1，此处更新影响 0 行，引用失败
            affected = tx.execute("""
                UPDATE media_asset
                SET ref_count = ref_count + 1
                WHERE asset_id = %s AND deleting = 0
            """, (asset_id,))

            if affected == 0:
                # 资产正在删除 → 回滚，让调用方走正常上传路径
                raise ReferenceRaceError(
                    f"asset {asset_id} 正在删除，无法引用")

            return {'ok': True, 'idempotent': False}

    def release_reference(self, asset_id, content_id):
        """
        解除引用。计数归零时不立即删除，而是标记待回收。
        """
        with self.mysql.transaction() as tx:
            # 1. 删引用记录。删不掉说明本来就没有这条引用（幂等）
            affected = tx.execute("""
                DELETE FROM media_asset_ref
                WHERE asset_id = %s AND content_id = %s
            """, (asset_id, content_id))
            if affected == 0:
                return {'ok': True, 'idempotent': True}

            # 2. 原子递减，并防止减到负数
            tx.execute("""
                UPDATE media_asset
                SET ref_count = GREATEST(ref_count - 1, 0)
                WHERE asset_id = %s
            """, (asset_id,))

            # 3. 读回真实计数——不信内存中的值
            row = tx.query(
                "SELECT ref_count FROM media_asset WHERE asset_id=%s",
                (asset_id,))

            if row['ref_count'] == 0:
                # 计数归零：标记待回收，延迟 7 天物理删除
                # 这 7 天内若有新引用，直接复活
                tx.execute("""
                    UPDATE media_asset SET zero_ref_at = NOW()
                    WHERE asset_id = %s AND ref_count = 0
                """, (asset_id,))

        return {'ok': True}

    # ── 对账：兜住前两道防线的漏网 ──

    def reconcile(self, batch_size=50000):
        """
        每日对账：比对 ref_count 与 media_asset_ref 表的实际行数。
        这是唯一能发现"计数泄漏"的手段——泄漏不会有任何报错。
        """
        rows = self.mysql.query("""
            SELECT a.asset_id, a.ref_count AS counter,
                   COUNT(r.content_id) AS actual
            FROM media_asset a
            LEFT JOIN media_asset_ref r ON r.asset_id = a.asset_id
            WHERE a.asset_id > %s
            GROUP BY a.asset_id, a.ref_count
            HAVING a.ref_count <> COUNT(r.content_id)
            LIMIT %s
        """, (self._checkpoint(), batch_size))

        fixed, leaks, early = 0, 0, 0
        for r in rows:
            if r['counter'] > r['actual']:
                leaks += 1          # 计数偏高 → 泄漏，数据本该被回收
            else:
                early += 1          # 计数偏低 → 危险！数据可能被误删

            # 以 media_asset_ref 的实际行数为准修正
            self.mysql.execute("""
                UPDATE media_asset SET ref_count = %s WHERE asset_id = %s
            """, (r['actual'], r['asset_id']))
            fixed += 1

        if early > 0:
            # 计数偏低是严重问题——意味着有数据可能已被误删
            self.alert.fire("refcount_undercount", severity="P0", detail={
                'count': early,
                'note': '计数低于实际引用数，可能已发生误删，需核查内容可播性'})
        if leaks > 100:
            self.alert.fire("refcount_leak", severity="P2",
                            detail={'count': leaks})

        return {'fixed': fixed, 'leaks': leaks, 'undercount': early}


class ReferenceRaceError(Exception):
    """引用与删除的竞态"""
    pass
```

**关键设计：`acquire_reference` 里的 `WHERE asset_id = %s AND deleting = 0` 是防竞态的核心。** 没有这个条件时，删除流程与引用流程可以交错执行：删除方读到 ref_count=0 → 引用方 +1 → 删除方物理删除 → 引用方的内容变死链。加上 `deleting = 0` 条件后，删除流程一旦置位就再不接受新引用，引用方会拿到明确的失败并走正常上传。

**对账（`reconcile`）不可省，因为泄漏是静默的。** 计数偏高不会有任何报错、任何监控告警——数据只是永远不被回收，存储缓慢泄漏。实测无对账时年泄漏约 3.8 PB（¥5.5 万/月，且逐年累积）。而计数偏低更危险：它意味着**数据可能已经被误删**，必须按 P0 处理。

### 删除与彻底擦除：跨系统的完整性

回到挑战 4。一份媒资散布在数百个物理位置，其中三类不在自己机房。

```
删除工作流（10 步，全部完成才算删除成功）：

  T+0    用户/合规触发删除
           │
           ├─ 步骤1  media_asset.deleting = 1        ← 立即阻断新引用与秒传
           ├─ 步骤2  ref_count 归零校验（若仍有引用则中止并告警）
           │
  T+0~5s   ├─ 步骤3  元数据标记不可读（读路径立即 404）
           ├─ 步骤4  播放地址签名密钥轮换 ★ 作废所有已下发地址
           │
  T+5s~5m  ├─ 步骤5  CDN 全网缓存清除（多厂商并发 purge）★
           │          · 需清除全部档位 × 全部分片 × 全部厂商
           │          · 一条长视频约 4,300 个 URL，需批量 API
           ├─ 步骤6  离线下载 DRM 许可吊销 ★ 客户端本地副本失效
           │
  T+5m~1h  ├─ 步骤7  跨地域副本删除（各地域独立执行 + 回执）
           ├─ 步骤8  数仓/备份快照中的媒资引用标记
           │
  T+1h~24h ├─ 步骤9  主动探测验证：从 12 个地区探测节点尝试播放
           │          全部返回 404/403 才算"不可读"达成 ✓
           │
  T+7d     └─ 步骤10 物理擦除（EC 块逐块覆写 + 元数据硬删）
                      合规要求 7 天内完成
```

带 ★ 的三步（密钥轮换、CDN 清除、DRM 吊销）**不在自己的数据库里**，是最容易漏的。

```python
class MediaTakedownWorkflow:
    """媒资删除工作流——十步全链路清理，可重试、可验证、可审计

    设计要点：
      1. 每步独立记录状态，任一失败不影响其他步骤，可单步重试
      2. 步骤9 的主动探测是"删除完成"的唯一判据，不信执行记录
      3. 物理擦除延迟 7 天，给误删留回退窗口（合规上限也是 7 天）
    """

    STEPS = [
        'mark_deleting', 'verify_zero_ref', 'mark_unreadable',
        'rotate_signing_key', 'purge_cdn', 'revoke_drm',
        'delete_cross_region', 'mark_warehouse',
        'verify_unreadable', 'physical_erase',
    ]
    # 各步骤的 SLA（秒），超时告警
    STEP_SLA = {
        'mark_deleting': 2, 'verify_zero_ref': 5, 'mark_unreadable': 5,
        'rotate_signing_key': 10, 'purge_cdn': 300, 'revoke_drm': 300,
        'delete_cross_region': 3600, 'mark_warehouse': 3600,
        'verify_unreadable': 86400, 'physical_erase': 604800,
    }
    PROBE_REGIONS = 12       # 主动探测的地区数
    PHYSICAL_ERASE_DELAY_SEC = 604800   # 7 天

    def __init__(self, mysql_client, redis_client, clients, config):
        self.mysql = mysql_client
        self.redis = redis_client
        self.cdn = clients['cdn_multi']          # 多 CDN 统一清除客户端
        self.drm = clients['drm']
        self.signer = clients['url_signer']
        self.storage = clients['storage']
        self.prober = clients['prober']          # 多地区探测客户端
        self.alert = config['alert_service']

    def start(self, asset_id, reason_type, reason_detail, sla_deadline):
        """创建删除任务"""
        task_id = self.mysql.execute("""
            INSERT INTO media_takedown_task
                (asset_id, reason_type, reason_detail, step_status,
                 sla_deadline, created_at)
            VALUES (%s, %s, %s, %s, %s, NOW())
            ON DUPLICATE KEY UPDATE reason_detail = VALUES(reason_detail)
        """, (asset_id, reason_type, reason_detail,
              self._init_step_status(), sla_deadline))
        self.redis.lpush("takedown:queue", str(asset_id))
        return task_id

    def execute(self, asset_id):
        """推进一个删除任务。幂等——可反复调用直到全部步骤完成"""
        task = self._load_task(asset_id)
        status = task['step_status']

        for step in self.STEPS:
            if status.get(step) == 2:
                continue        # 已成功，跳过

            # physical_erase 需等延迟窗口
            if step == 'physical_erase':
                if not self._erase_window_reached(task):
                    break
                # 且必须 verify_unreadable 已通过
                if status.get('verify_unreadable') != 2:
                    break

            ok, detail = self._run_step(step, asset_id, task)
            status[step] = 2 if ok else 3
            self._save_status(asset_id, status, step, detail)

            if not ok:
                # 单步失败：记录后继续尝试后续独立步骤，不整体中断
                # （例如 CDN 清除失败不应阻止跨地域副本删除）
                if step in ('mark_deleting', 'verify_zero_ref'):
                    break       # 但前两步是前置条件，失败必须中断
                continue

        return self._summarize(asset_id, status)

    def _run_step(self, step, asset_id, task):
        """分派到具体步骤实现"""
        handler = getattr(self, f"_step_{step}")
        started = time.time()
        try:
            ok, detail = handler(asset_id, task)
        except Exception as e:
            return False, {'error': str(e)[:500]}

        elapsed = time.time() - started
        if elapsed > self.STEP_SLA.get(step, 3600):
            self.alert.fire("takedown_step_slow", severity="P2", detail={
                'asset_id': asset_id, 'step': step,
                'elapsed_sec': round(elapsed, 1)})
        return ok, detail

    # ── 各步骤实现 ──

    def _step_mark_deleting(self, asset_id, task):
        """置 deleting=1，阻断新引用与秒传命中"""
        affected = self.mysql.execute(
            "UPDATE media_asset SET deleting = 1 WHERE asset_id = %s",
            (asset_id,))
        return affected >= 0, {}

    def _step_verify_zero_ref(self, asset_id, task):
        """
        校验无引用。若仍有引用，说明有其他内容在用这份媒资
        （去重共享），不能删除资产本身——只能解除本内容的引用。
        """
        row = self.mysql.query("""
            SELECT a.ref_count, COUNT(r.content_id) AS actual
            FROM media_asset a
            LEFT JOIN media_asset_ref r ON r.asset_id = a.asset_id
            WHERE a.asset_id = %s GROUP BY a.asset_id, a.ref_count
        """, (asset_id,))

        if row and row['actual'] > 0:
            # 仍被其他内容引用 → 本任务转为"仅解引用"，不删物理数据
            self.mysql.execute("""
                UPDATE media_takedown_task
                SET mode = 'deref_only'
                WHERE asset_id = %s
            """, (asset_id,))
            self.mysql.execute(
                "UPDATE media_asset SET deleting = 0 WHERE asset_id = %s",
                (asset_id,))
            return True, {'mode': 'deref_only',
                          'remaining_refs': row['actual']}
        return True, {'mode': 'full_delete'}

    def _step_mark_unreadable(self, asset_id, task):
        """元数据标记不可读——读路径立即返回 404"""
        self.mysql.execute("""
            UPDATE media_asset SET readable = 0 WHERE asset_id = %s
        """, (asset_id,))
        # 同时清本地缓存与 Redis 元数据缓存
        self.redis.delete(f"asset:meta:{asset_id}")
        self.redis.publish("asset:invalidate", str(asset_id))
        return True, {}

    def _step_rotate_signing_key(self, asset_id, task):
        """
        ★ 轮换该资产的播放地址签名密钥。
        这一步作废所有已下发但未过期的播放地址——
        不做的话，已拿到地址的客户端仍能从 CDN 播放。
        """
        new_key_version = self.signer.rotate_asset_key(asset_id)
        return True, {'new_key_version': new_key_version}

    def _step_purge_cdn(self, asset_id, task):
        """
        ★ 多 CDN 全网缓存清除。
        难点：一条长视频有 6 档 × 每档约 720 个分片 = 4,320 个 URL，
        且要在 3-4 家 CDN 上各清一遍。必须用批量 API + 目录级清除。
        """
        # 优先用目录级清除（一次请求清整个前缀），失败再退回 URL 级
        prefixes = self.storage.list_cdn_prefixes(asset_id)
        results = {}
        for vendor in self.cdn.vendors():
            try:
                task_ids = self.cdn.purge_by_prefix(vendor, prefixes)
                results[vendor] = {'ok': True, 'purge_tasks': task_ids}
            except NotImplementedError:
                # 该厂商不支持目录清除 → 枚举 URL 批量清
                urls = self.storage.list_all_cdn_urls(asset_id)
                task_ids = self.cdn.purge_by_urls(vendor, urls, batch=1000)
                results[vendor] = {'ok': True, 'purge_tasks': task_ids,
                                   'url_count': len(urls)}
            except Exception as e:
                results[vendor] = {'ok': False, 'error': str(e)[:200]}

        all_ok = all(v['ok'] for v in results.values())
        return all_ok, results

    def _step_revoke_drm(self, asset_id, task):
        """
        ★ 吊销 DRM 许可，让已下载到用户设备的离线副本失效。
        注意：只对 DRM 保护的内容有效；未加密的离线副本无法远程失效
        （这是"离线下载必须加 DRM"的合规理由）。
        """
        if not self.mysql.query(
                "SELECT is_drm FROM content_ext_video WHERE content_id IN "
                "(SELECT content_id FROM media_asset_ref WHERE asset_id=%s) "
                "LIMIT 1", (asset_id,)):
            return True, {'skipped': 'no_drm_content'}
        revoked = self.drm.revoke_all_licenses(asset_id)
        return True, {'revoked_licenses': revoked}

    def _step_delete_cross_region(self, asset_id, task):
        """各地域独立删除，收集回执"""
        results = {}
        for region in self.storage.regions():
            try:
                results[region] = self.storage.delete_region(asset_id, region)
            except Exception as e:
                results[region] = {'ok': False, 'error': str(e)[:200]}
        return all(r.get('ok') for r in results.values()), results

    def _step_mark_warehouse(self, asset_id, task):
        """数仓与备份快照中标记（不能直接删快照，只能标记 + 到期随快照过期）"""
        self.mysql.execute("""
            INSERT INTO warehouse_deletion_mark (asset_id, marked_at)
            VALUES (%s, NOW()) ON DUPLICATE KEY UPDATE marked_at = NOW()
        """, (asset_id,))
        return True, {}

    def _step_verify_unreadable(self, asset_id, task):
        """
        ★★ 主动探测验证——这是"删除完成"的唯一判据。
        从 12 个地区的探测节点实际尝试播放，全部拿到 404/403 才算成功。
        任何一个地区仍能播放，说明有环节漏了。
        """
        probe_results = self.prober.try_playback_all_regions(
            asset_id, regions=self.PROBE_REGIONS)

        still_playable = [r for r in probe_results if r['playable']]
        if still_playable:
            self.alert.fire("takedown_verify_failed", severity="P0", detail={
                'asset_id': asset_id,
                'still_playable_regions': [r['region'] for r in still_playable],
                'note': '删除步骤已执行但内容仍可播放，需人工核查 CDN 与签名'})
            return False, {'still_playable': still_playable}

        self.mysql.execute("""
            UPDATE media_takedown_task SET verified_at = NOW()
            WHERE asset_id = %s
        """, (asset_id,))
        return True, {'probed_regions': len(probe_results)}

    def _step_physical_erase(self, asset_id, task):
        """
        物理擦除。EC 块逐块覆写（合规要求"不可恢复"）。
        延迟 7 天执行，且必须 verify_unreadable 已通过。
        """
        erased = self.storage.secure_erase(asset_id, overwrite_passes=1)
        self.mysql.execute("""
            UPDATE media_asset SET readable = 0, erased_at = NOW()
            WHERE asset_id = %s
        """, (asset_id,))
        return True, {'erased_blocks': erased}

    # ── 辅助 ──

    def _erase_window_reached(self, task):
        import datetime
        verified = task.get('verified_at')
        if verified is None:
            return False
        return (datetime.datetime.now() - verified).total_seconds() \
            >= self.PHYSICAL_ERASE_DELAY_SEC

    def _init_step_status(self):
        import json
        return json.dumps({s: 0 for s in self.STEPS})

    def sweep_overdue(self):
        """扫描超 SLA 未完成的删除任务——合规风险，必须告警到人"""
        overdue = self.mysql.query("""
            SELECT asset_id, reason_type, sla_deadline, step_status
            FROM media_takedown_task
            WHERE verified_at IS NULL AND sla_deadline < NOW()
        """)
        if overdue:
            self.alert.fire("takedown_sla_breach", severity="P0", detail={
                'count': len(overdue),
                'samples': [o['asset_id'] for o in overdue[:10]]})
        return overdue
```

**关键设计：步骤 9（主动探测验证）是"删除完成"的唯一判据，而不是"步骤 1-8 都执行成功了"。** 执行成功与实际不可读是两件事——CDN 的 purge API 返回 200 只表示"清除任务已受理"，不表示已生效；某个厂商的某个边缘节点可能清除失败。**只有从外部实际探测确认播不了，才能声称删除完成。** 这是所有跨系统清理工作流的通用原则：**信探测结果，不信执行回执。**

**为什么 `_step_verify_zero_ref` 会把任务降级为 `deref_only`？** 因为去重让多个内容共享同一份媒资。用户 A 删除自己的内容时，如果用户 B 的内容还在引用同一份物理数据，就不能删物理数据——只能解除 A 的引用。**去重带来的存储节省，代价是删除语义变复杂：删内容 ≠ 删媒资。**

### 数据库设计

存储域的表与 P00 的 `media_asset` / `media_variant` 是**两层关系**：那两张表描述"业务上有哪些媒资和档位"，本域的表描述"这些字节物理上放在哪、放在哪层、被谁引用、什么时候可以删"。**业务层看不到 tier、EC 参数、pack 偏移，这是刻意的**——分层是纯粹的成本优化，不应该污染业务模型。

```sql
-- 表名：media_object —— 物理对象的存储层状态，媒资存储域的核心表
CREATE TABLE media_object (
    object_id           BIGINT UNSIGNED  NOT NULL                COMMENT '对象 ID，雪花算法',
    asset_id            BIGINT UNSIGNED  NOT NULL                COMMENT '所属媒资 ID，关联 media_asset',
    variant_id          BIGINT UNSIGNED  NOT NULL DEFAULT 0      COMMENT '所属转码档位 ID，0 表示原片',
    storage_key         VARCHAR(255)     NOT NULL                COMMENT '逻辑键，业务侧唯一寻址标识，跨层迁移不变',
    logical_bytes       BIGINT UNSIGNED  NOT NULL                COMMENT '逻辑字节数，计费与统计口径',
    tier                TINYINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '存储层：0=hot(NVMe 3副本) 1=warm(HDD 3副本) 2=cold(SMR EC)',
    layout              TINYINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '布局：0=replica 三副本 1=ec_striped 独立条带 2=ec_packed 打包后条带',
    ec_k                TINYINT UNSIGNED NOT NULL DEFAULT 0      COMMENT 'EC 数据块数，非 EC 布局为 0',
    ec_m                TINYINT UNSIGNED NOT NULL DEFAULT 0      COMMENT 'EC 校验块数，非 EC 布局为 0',
    pack_id             BIGINT UNSIGNED  NOT NULL DEFAULT 0      COMMENT '所属打包组 ID，ec_packed 布局有效，关联 media_pack',
    pack_offset         BIGINT UNSIGNED  NOT NULL DEFAULT 0      COMMENT '在打包组内的起始偏移（字节）',
    placement           JSON             NOT NULL                COMMENT '放置详情，示例 {"blocks":[{"i":0,"node":"cn-sh-c-0173","disk":7}],"crc":"..."}',
    replica_regions     VARCHAR(64)      NOT NULL DEFAULT ''     COMMENT '实际持有副本的地域，逗号分隔，示例 cn-sh,cn-bj',
    heat_score          DECIMAL(12,4)    NOT NULL DEFAULT 0.0000 COMMENT '热度分，HeatScorer 输出，每 10 分钟刷新',
    last_access_at      DATETIME         NULL     DEFAULT NULL   COMMENT '最近一次读取时间，用于降级判据',
    access_7d           INT UNSIGNED     NOT NULL DEFAULT 0      COMMENT '7 日读取次数，滚动窗口',
    migrating_to        TINYINT          NOT NULL DEFAULT -1     COMMENT '正在迁往的目标层，-1 表示无进行中迁移',
    deleting            TINYINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '删除中标记，1 表示已进入删除流程，禁止新增引用',
    checksum_verified_at DATETIME        NULL     DEFAULT NULL   COMMENT '最近一次巡检校验通过时间，静默损坏检测用',
    created_at          DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_at          DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (object_id),
    UNIQUE KEY uk_storage_key (storage_key),
    KEY idx_asset_variant (asset_id, variant_id),
    KEY idx_tier_heat (tier, heat_score),
    KEY idx_tier_access (tier, last_access_at),
    KEY idx_pack (pack_id, pack_offset),
    KEY idx_migrating (migrating_to, updated_at),
    KEY idx_verify_sweep (checksum_verified_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**几个字段的设计意图值得单独说明：**

- **`storage_key` 跨层迁移不变**：这是分层对业务透明的关键。业务永远用 `storage_key` 寻址，读路径先查 `media_object` 拿到 `tier` 与 `placement` 再决定怎么读。如果业务侧直接持有物理路径，任何迁移都会变成业务侧的批量改写——迁移就做不动了。
- **`migrating_to` 而不是 `is_migrating`**：迁移是**有方向**的。回迁（Cold → Hot）和下沉（Hot → Cold）的冲突处理完全不同：一个对象正在下沉时突然爆热，必须能立刻取消下沉并转为回迁；只用布尔标记做不到这个判断。
- **`deleting` 与 `acquire_reference` 的 `WHERE` 条件配合**：这个字段是引用竞态的唯一防线（详见前文 `MediaDeduplicator`）。它必须在**同一条 UPDATE 的 WHERE 里**被检查，而不是先 SELECT 再判断。
- **`idx_tier_heat` 与 `idx_tier_access` 两条索引都要**：前者服务"找出冷层里热度最高的对象"（回迁扫描），后者服务"找出热层里最久没访问的对象"（下沉扫描）。两个扫描方向的排序键不同，一条复合索引覆盖不了。
- **`checksum_verified_at` 上的裸索引**：巡检任务按"最久未校验"顺序扫描全表，这条索引让它可以增量推进而不用每轮全表扫。1,240 亿个对象，全表扫是不可接受的。

```sql
-- 表名：media_pack —— 小对象打包组，EC 前的聚合单元
CREATE TABLE media_pack (
    pack_id             BIGINT UNSIGNED  NOT NULL                COMMENT '打包组 ID',
    tier                TINYINT UNSIGNED NOT NULL DEFAULT 2      COMMENT '所在层，打包只用于 cold 层',
    target_bytes        BIGINT UNSIGNED  NOT NULL                COMMENT '目标大小（字节），默认 268435456 即 256 MB',
    filled_bytes        BIGINT UNSIGNED  NOT NULL DEFAULT 0      COMMENT '已填充字节数',
    object_count        INT UNSIGNED     NOT NULL DEFAULT 0      COMMENT '包含的对象数',
    dead_bytes          BIGINT UNSIGNED  NOT NULL DEFAULT 0      COMMENT '已删除对象占用的死字节，达阈值触发整包重整',
    status              TINYINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '状态：0=open 填充中 1=sealed 已封包已EC 2=compacting 重整中 3=retired 已废弃',
    ec_k                TINYINT UNSIGNED NOT NULL DEFAULT 16     COMMENT 'EC 数据块数',
    ec_m                TINYINT UNSIGNED NOT NULL DEFAULT 4      COMMENT 'EC 校验块数',
    placement           JSON             NULL     DEFAULT NULL   COMMENT '20 个块的放置详情，封包后写入',
    sealed_at           DATETIME         NULL     DEFAULT NULL   COMMENT '封包时间',
    created_at          DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_at          DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (pack_id),
    KEY idx_status_filled (status, filled_bytes),
    KEY idx_compact_candidate (status, dead_bytes)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**`dead_bytes` 是打包方案必须付的代价。** 打包组封包后就是 EC 条带，无法原地修改。组内某个对象被删除时，它的字节仍然物理占用——变成"死字节"。当 `dead_bytes / filled_bytes > 0.35` 时才值得整包重整（读出存活对象、重新打包、EC 编码、切指针、删旧包）。阈值定在 35% 是因为：低于这个比例，重整消耗的 IO 成本超过回收的存储成本。

```sql
-- 表名：media_dedup_index —— 内容哈希到媒资的映射，秒传入口
CREATE TABLE media_dedup_index (
    content_hash        CHAR(64)         NOT NULL                COMMENT '全文件 SHA-256 十六进制，秒传主键',
    asset_id            BIGINT UNSIGNED  NOT NULL                COMMENT '首次上传落地的媒资 ID',
    file_bytes          BIGINT UNSIGNED  NOT NULL                COMMENT '文件字节数，二次校验字段之一',
    head_md5            CHAR(32)         NOT NULL                COMMENT '首 1 MB 的 MD5，二次校验字段之二',
    tail_md5            CHAR(32)         NOT NULL                COMMENT '末 1 MB 的 MD5，二次校验字段之三',
    ref_count           INT UNSIGNED     NOT NULL DEFAULT 1      COMMENT '引用计数，冗余于 media_asset_ref 的行数，仅作快速判断',
    hit_count           BIGINT UNSIGNED  NOT NULL DEFAULT 0      COMMENT '累计秒传命中次数，用于评估去重收益',
    created_at          DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_at          DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (content_hash),
    KEY idx_asset (asset_id),
    KEY idx_hit (hit_count)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 表名：media_asset_ref —— 媒资引用明细，引用计数的唯一权威来源
CREATE TABLE media_asset_ref (
    ref_id              BIGINT UNSIGNED  NOT NULL AUTO_INCREMENT COMMENT '引用记录 ID',
    asset_id            BIGINT UNSIGNED  NOT NULL                COMMENT '被引用的媒资 ID',
    content_id          BIGINT UNSIGNED  NOT NULL                COMMENT '引用方内容 ID',
    ref_type            TINYINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '引用类型：0=原始上传 1=秒传命中 2=二创引用 3=转载引用',
    uploader_id         BIGINT UNSIGNED  NOT NULL                COMMENT '引用方创作者 ID，删除时的责任主体',
    released_at         DATETIME         NULL     DEFAULT NULL   COMMENT '解除引用时间，NULL 表示仍持有',
    created_at          DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    PRIMARY KEY (ref_id),
    UNIQUE KEY uk_asset_content (asset_id, content_id),
    KEY idx_content (content_id),
    KEY idx_asset_live (asset_id, released_at),
    KEY idx_uploader (uploader_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**为什么 `ref_count` 和 `media_asset_ref` 同时存在——这不是冗余设计的失误，是刻意的双轨。** `media_dedup_index.ref_count` 是**快照**，用于秒传路径上的 O(1) 判断（"这份媒资还有人用吗"）；`media_asset_ref` 是**账本**，`COUNT(*) WHERE released_at IS NULL` 是唯一权威值。两者会因为进程崩溃、事务部分提交而漂移，所以 `reconcile` 每日对账，**以账本为准修正快照**。

**关键：不要试图用"只保留计数器"来简化。** 只有计数器时，一旦漂移你永远不知道正确值是多少，也不知道是谁泄漏的；有明细账本时，漂移可修复，且能定位到具体的 `content_id`。**引用计数机制的可运维性，完全取决于有没有明细。** 代价是 `media_asset_ref` 会有约 380 亿行——按 `asset_id` 哈希分 1024 库，单库 3,700 万行，可接受。

`uk_asset_content` 这条唯一键还顺带解决了幂等：同一内容对同一媒资的重复引用请求会被数据库拒绝，不需要在应用层做去重。

```sql
-- 表名：media_tier_migration —— 分层迁移任务，兼作迁移审计日志
CREATE TABLE media_tier_migration (
    task_id             BIGINT UNSIGNED  NOT NULL AUTO_INCREMENT COMMENT '迁移任务 ID',
    object_id           BIGINT UNSIGNED  NOT NULL                COMMENT '被迁移对象 ID',
    from_tier           TINYINT UNSIGNED NOT NULL                COMMENT '源层',
    to_tier             TINYINT UNSIGNED NOT NULL                COMMENT '目标层',
    reason              VARCHAR(32)      NOT NULL                COMMENT '触发原因：heat_revival / cold_decay / rebalance / manual / pack_compact',
    trigger_signal      VARCHAR(128)     NOT NULL DEFAULT ''     COMMENT '触发信号详情，示例 accel=8.3,qps_1m=412',
    priority            TINYINT UNSIGNED NOT NULL DEFAULT 5      COMMENT '优先级：0=热度复活回迁（最高） 5=常规下沉 8=再平衡',
    bytes_total         BIGINT UNSIGNED  NOT NULL                COMMENT '待迁移字节数',
    bytes_copied        BIGINT UNSIGNED  NOT NULL DEFAULT 0      COMMENT '已复制字节数，断点续传用',
    step                TINYINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '四步安全顺序：0=待调度 1=复制中 2=校验中 3=已切读 4=待删源 5=已完成',
    status              TINYINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '状态：0=pending 1=running 2=succeeded 3=failed 4=cancelled 5=rolled_back',
    attempt             TINYINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '尝试次数',
    fail_reason         VARCHAR(255)     NOT NULL DEFAULT ''     COMMENT '失败原因',
    source_purge_at     DATETIME         NULL     DEFAULT NULL   COMMENT '源数据计划删除时间，切读后延迟 24 小时',
    elapsed_ms          INT UNSIGNED     NOT NULL DEFAULT 0      COMMENT '耗时毫秒，SLA 统计用',
    created_at          DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_at          DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (task_id),
    UNIQUE KEY uk_object_running (object_id, status),
    KEY idx_schedule (status, priority, created_at),
    KEY idx_purge_sweep (step, source_purge_at),
    KEY idx_reason_time (reason, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**`step` 与 `status` 是两个正交维度，不要合并成一个状态机。** `step` 记录四步安全顺序（复制 → 校验 → 切读 → 延迟删源）走到哪一步，`status` 记录任务整体的生死。区分的价值在**恢复**：一个 `status=failed, step=3` 的任务意味着"已经切读成功但删源失败"——这是安全的，只需要重试删源；而 `status=failed, step=1` 意味着"复制中断"——需要从 `bytes_copied` 断点续传。**合并成一个状态机后，恢复逻辑就无法区分这两种失败的危险程度。**

`idx_purge_sweep` 服务延迟删源的清扫任务：找出 `step=4 AND source_purge_at <= NOW()` 的任务执行物理删除。**这条索引对应的是整套分层方案里唯一不可逆的操作，它的正确性优先级最高。**

```sql
-- 表名：media_variant_access —— 转码副本的日访问统计，副本回收的判据来源
CREATE TABLE media_variant_access (
    stat_date           DATE             NOT NULL                COMMENT '统计日期',
    variant_id          BIGINT UNSIGNED  NOT NULL                COMMENT '转码档位 ID',
    asset_id            BIGINT UNSIGNED  NOT NULL                COMMENT '所属媒资 ID',
    play_count          INT UNSIGNED     NOT NULL DEFAULT 0      COMMENT '当日起播次数',
    bytes_served        BIGINT UNSIGNED  NOT NULL DEFAULT 0      COMMENT '当日回源字节数',
    distinct_region     SMALLINT UNSIGNED NOT NULL DEFAULT 0     COMMENT '当日访问覆盖的地域数，地域时差误杀防护用',
    jit_rebuild_count   SMALLINT UNSIGNED NOT NULL DEFAULT 0     COMMENT '当日因缺档位触发的 JIT 重建次数，误杀的直接证据',
    PRIMARY KEY (stat_date, variant_id),
    KEY idx_variant_date (variant_id, stat_date),
    KEY idx_asset_date (asset_id, stat_date),
    KEY idx_jit_alarm (stat_date, jit_rebuild_count)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**主键是 `(stat_date, variant_id)` 而不是 `(variant_id, stat_date)`，反直觉但必要。** 这张表最重的两个写入模式都是按天批量的：每天凌晨从数仓灌入前一日的统计（数千万行，按日期聚簇写入才顺序）、以及按 `stat_date` 删除 90 天前的分区。日期在前让这两个操作都变成顺序 IO。而"查某档位的历史访问曲线"（回收判据、周期性检测）走 `idx_variant_date` 覆盖索引，不受影响。

**`jit_rebuild_count` 是整个回收方案的自省指标。** 它记录"因为某档位不存在而必须临时重转"的次数——直接等于误杀的证据。`idx_jit_alarm` 让监控能每天问一句"昨天有多少个档位因误杀被重建"，这个数字若持续上升，说明回收判据过于激进，应当调宽豁免规则。**任何有损优化都必须自带一个能量化"损了多少"的指标，否则你只能看到收益看不到代价。**

```sql
-- 表名：media_erase_task —— 媒资级物理擦除任务，10 步工作流的执行台账
CREATE TABLE media_erase_task (
    task_id             BIGINT UNSIGNED  NOT NULL AUTO_INCREMENT COMMENT '擦除任务 ID',
    asset_id            BIGINT UNSIGNED  NOT NULL                COMMENT '待擦除媒资 ID',
    origin_content_id   BIGINT UNSIGNED  NOT NULL                COMMENT '发起删除的内容 ID',
    origin_type         VARCHAR(24)      NOT NULL                COMMENT '来源：user_delete / copyright / legal_order / compliance / gdpr',
    mode                VARCHAR(16)      NOT NULL DEFAULT 'full' COMMENT '模式：full 物理擦除 / deref_only 仅解引用（尚有其他引用方）',
    current_step         TINYINT UNSIGNED NOT NULL DEFAULT 1     COMMENT '当前步骤 1-10，见 MediaTakedownWorkflow.STEPS',
    step_state          JSON             NOT NULL                COMMENT '各步执行结果，示例 {"6":{"ok":true,"ms":8400},"7":{"ok":false,"err":"cdn-c timeout"}}',
    verify_regions_ok   SMALLINT UNSIGNED NOT NULL DEFAULT 0     COMMENT '主动探测确认不可读的地域数，需达 12 才算通过',
    deadline_unreadable DATETIME         NOT NULL                COMMENT '不可读截止时间，创建后 24 小时，合规硬约束',
    deadline_erased     DATETIME         NOT NULL                COMMENT '物理擦除截止时间，创建后 7 天，合规硬约束',
    status              TINYINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '状态：0=pending 1=running 2=unreadable_verified 3=erased 4=failed 5=overdue',
    attempt             TINYINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '尝试次数',
    fail_reason         VARCHAR(255)     NOT NULL DEFAULT ''     COMMENT '失败原因',
    audit_trace_id      VARCHAR(64)      NOT NULL DEFAULT ''     COMMENT '审计链路 ID，监管问询时的举证入口',
    created_at          DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_at          DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (task_id),
    UNIQUE KEY uk_asset_origin (asset_id, origin_content_id),
    KEY idx_deadline_unreadable (status, deadline_unreadable),
    KEY idx_deadline_erased (status, deadline_erased),
    KEY idx_origin_type (origin_type, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**两个 deadline 字段各配一条索引，因为它们对应两个不同的合规义务、两个不同的清扫任务、两个不同的告警阈值。** 24 小时不可读是**用户可感知**的义务（用户删了还能播，是投诉与舆情来源）；7 天物理擦除是**监管可稽核**的义务（现场检查时要能证明字节已不存在）。前者逾期是产品事故，后者逾期是法律风险。合并成一个 deadline 会把两个不同量级的风险混在一起报警。

**`step_state` 用 JSON 而不是拆成 10 个布尔列**：步骤会演进（新增 CDN 厂商、新增离线下载渠道就要加步骤），JSON 让工作流定义与表结构解耦。代价是无法对单步做 SQL 聚合分析——这个代价通过把关键结果提升为独立列（`verify_regions_ok`）来补偿：需要索引和告警的字段一定要出列，剩下的留在 JSON 里。

**表间关系速览：**

```
media_asset（P00，业务层：这份媒资是什么）
    │
    ├──< media_asset_ref ──────── 谁在引用（引用计数账本）
    │         ▲
    │         └── media_dedup_index（hash → asset_id，ref_count 快照）
    │
    ├──< media_variant（P00，业务层：有哪些转码档位）
    │         │
    │         └──< media_variant_access ── 每档位的日访问（回收判据）
    │
    ├──< media_object（本域，物理层：字节在哪一层、怎么放）
    │         │
    │         ├── pack_id ──> media_pack（小对象打包组，EC 前的聚合单元）
    │         │
    │         └──< media_tier_migration（迁移任务与审计）
    │
    └──< media_erase_task（擦除工作流台账）
```

**唯一的写方约定**：`media_object.tier` / `layout` / `placement` 只有迁移调度器能改，`heat_score` 只有热度计算器能改，`deleting` 只有擦除工作流能改，`ref_count` 只有去重服务能改。四个字段四个单写方——**这与 P00 的 `content.status` 单写方是同一条原则的应用：物理层的状态字段同样禁止多写方并发。**

## 常见陷阱（深度分析）

### 陷阱 1：先上 EC 再做介质分层

这是本域最常见、代价最大的顺序错误，源于一个听起来无懈可击的推理：EC 把冗余从 3.0x 压到 1.25x，**省 58%**；介质分层只是换便宜盘，看起来是"运维优化"不是"架构优化"。工程师的荣誉感也偏向 EC——它有 Reed-Solomon 数学、有条带布局、有降级读，是"真正的分布式存储技术"。

实算把这个直觉彻底推翻：

| 顺序 | 第 1 步做完 | 第 2 步做完 | 第 1 步的收益占比 |
|------|-----------|-----------|-----------------|
| **先分层后 EC**（正确） | ¥6,408 万 → ¥2,211 万（**省 ¥4,197 万**） | → ¥1,258 万（再省 ¥953 万） | **81.5%** |
| 先 EC 后分层（错误） | ¥6,408 万 → ¥3,712 万（省 ¥2,696 万） | → ¥1,258 万（再省 ¥2,454 万） | 52.4% |

两条路径终点相同（¥1,258 万），但**第一步的收益差 1.56 倍**，而两个方案的**工期与风险相差一个数量级**：

| 维度 | 介质分层 | EC(16,4) |
|------|---------|---------|
| 工期 | 4 周（采购 + 迁移调度器 + 灰度） | 12 周（编码库、放置策略、降级读、重建、巡检） |
| 故障域 | 单盘（丢一块盘只影响该盘副本） | **20 块盘跨 20 个节点**（任 5 块同时坏即丢数据） |
| 恢复速度 | 单盘重建 = 拷 1 份副本，约 2 小时 | 单块重建 = 读 16 块异或，约 9 小时且占满网络 |
| 可回退 | 随时（把数据迁回来即可） | **困难**（EC 数据回三副本需全量重写） |
| 出错的最坏后果 | 读变慢 | **数据丢失** |

**后果：** 团队按"技术含量"排序，花 12 周上 EC，期间承担 20 块盘的故障域风险与不可回退性，换来 ¥2,696 万/月；而只要 4 周、几乎零数据风险、随时可回退的介质分层能拿 ¥4,197 万/月，却被排到第二阶段。更糟的是：**EC 先上会让介质分层变难**——已经 EC 化的数据要下沉到 SMR，必须先解码回完整对象、再重新条带化，迁移成本翻倍。而反过来（先分层、再对已在 SMR 上的冷数据做 EC）是纯增量操作。

**解决方案：** 按「收益 ÷ 风险 ÷ 工期」而不是「技术含量」排优先级。本域的正确实施顺序是 12 周四阶段：

```
阶段 1（周 1-2）：可观测先行 —— 零风险，但决定后面三个阶段做得对不对
  ├─ 埋点：每次读取上报 object_id / tier / 首字节延迟 / 是否降级读
  ├─ 建 media_variant_access 与 media_object.access_7d 的统计链路
  └─ 出报表：真实访问分布（不要相信"1% 占 80%"这个假设，实测它）
     ▲ 这两周不省一分钱，但如果热度分布与假设不符，后面的分层比例全错

阶段 2（周 3-6）：介质分层 —— 拿走 81.5% 的收益
  ├─ 周 3：采购 HDD/SMR 机型，storage_key 与物理路径解耦（改读路径）
  ├─ 周 4：迁移调度器（四步安全顺序），先只做 Hot → Warm
  ├─ 周 5：灰度 1% → 10% → 50% 对象下沉，观察首字节 P99 与回迁触发率
  └─ 周 6：开 Warm → Cold 下沉，热度回迁通道上线
     ▲ 到这里累计省 ¥4,197 万/月，冗余仍是 3.0x，任何一步可原路迁回

阶段 3（周 7-9）：转码副本回收 —— 拿走 13.6% 的收益，与 EC 无耦合
  ├─ 周 7：豁免规则（周期性/季节性/兜底档位）+ JIT 预算护栏
  ├─ 周 8：影子模式跑 7 天（只记录"会删哪些"，不真删），核对误杀率
  └─ 周 9：开启回收，延迟物理删除 7 天，监控 jit_rebuild_count
     ▲ 影子模式这一周是整个阶段最重要的一周——它把不可逆操作变成可验证的

阶段 4（周 10-12+）：冷层 EC —— 最后 4.9% 的收益，风险最高
  ├─ 周 10：小对象打包（先只打包，仍存三副本，验证 pack 索引正确性）
  ├─ 周 11：EC 编码 + 降级读 + hedged read，只对 1% 冷数据开
  └─ 周 12+：逐步放量至冷层全量，全程保留"解码回三副本"的回退脚本
     ▲ 到第 9 周已拿到 81% + 13.6% = 95% 的可得收益；EC 是在充分安全垫上做的
```

**这个排序的元原则：把不可逆、故障域大、工期长的动作放到最后，让它在前面已经建好的可观测性与安全垫上执行。** 阶段 1 的两周"不产出"投入，是后面三个阶段敢于放量的唯一依据。

### 陷阱 2：用「最近访问时间」单指标做冷热判据

最容易写出的判据是 `WHERE last_access_at < NOW() - INTERVAL 7 DAY THEN 降级`。它简单、直观、无状态，几乎所有第一版分层都长这样。

它在三类内容上系统性出错：

| 内容类型 | 真实模式 | 单指标判据的行为 | 后果 |
|---------|---------|---------------|------|
| 周更节目 | 每 7 天一个访问尖峰 | 第 8 天判定为冷、下沉；第 9 天更新，全季回暖 | 每周一次全量回迁风暴 |
| 长尾但稳定 | 每天恒定 30 次播放 | 永远"最近访问过"，永不下沉 | 大量低价值数据滞留热层 |
| 突发热点 | 0 QPS 持续两年，然后 10 分钟涨到 8,000 | 已在冷层，且下沉时间越久越"该留在冷层" | 首帧 800 ms，冷层被打满 |

问题的本质是：**`last_access_at` 是一个标量，而热度是一个有形状的时间序列。** 标量丢掉了三个关键信息——访问的**频次**（1 次 vs 1 万次）、**趋势**（在涨还是在跌）、**周期性**（是不是每周同一天回来）。

**后果：** 第一类内容造成周期性迁移风暴（迁移带宽被自己制造的往返流量吃满，实测某周更综艺上线后 40 集同时回迁，占满 3.2 Tbps 内网带宽 11 分钟）；第二类内容让热层容量按最坏情况膨胀（热层占比从设计的 3% 涨到 9%，多花 ¥192 万/月）；第三类内容是**用户可感知的体验事故**，且往往发生在流量最大、最受关注的时刻。

**解决方案：** 用多信号加权热度分 + 独立的复活检测通道，两者职责分离：

- **热度分（`HeatScorer`）负责"稳态该在哪一层"**：1 小时/24 小时/7 日三个窗口的访问量加权（W_1H=0.42、W_24H=0.24、W_7D=0.11），**加上一个加速度项 W_ACCEL=0.18**。加速度项是关键——它让"访问量还不高但正在快速上涨"的对象提前晋升，而不是等绝对量达标（那时候已经晚了）。
- **复活检测（`detect_revival`）负责"现在要不要紧急回迁"**：走独立的、更快的通道（10 秒粒度而非 10 分钟），三个信号任一命中即触发，不等热度分收敛。
- **周期性豁免**：对访问序列在 lag=7 或 lag=30 处自相关显著的对象（`_is_periodic`），下沉判据从"7 日无访问"改为"连续 3 个周期无访问"。
- **绝对量下限**：无论多久没访问，**日均播放 > 500 次的对象不下沉**——这堵住了"长尾但稳定"这一类。

**一句话概括：稳态分层看加权热度分，突变响应看加速度与独立探测。用一个标量同时承担这两件事，必然在其中一件上出错。**

### 陷阱 3：迁移时先切读再复制（或先删源再确认）

迁移的四个动作——复制到目标层、校验目标层数据、把读流量切到目标层、删除源数据——有 4! = 24 种排列，但**只有一种是安全的**：复制 → 校验 → 切读 → 延迟删源。

常见的三种错误排列及其失效方式：

| 错误顺序 | 看起来的好处 | 实际失效 |
|---------|------------|---------|
| 切读 → 复制 | "先把流量导过去，复制慢点没关系" | 目标层还没数据，**所有读请求 404**，且时间窗等于整个复制耗时（4K 长视频 6 分钟） |
| 复制 → 切读 → 删源（同事务） | "少一步校验，快" | 复制过程中的静默损坏不会被发现，切读后用户读到损坏数据，**而源已删，无法恢复** |
| 复制 → 校验 → 删源 → 切读 | "先释放空间" | 删源与切读之间的任何时刻，**在途请求仍指向源**（连接复用、客户端缓存的 URL、CDN 回源重试），这些请求全部失败 |

第三种最隐蔽也最常见，因为"校验都过了，源肯定没用了"这个推理感觉很扎实。它忽略的是：**切读不是一个瞬时原子操作。** 元数据更新后，仍有大量在途状态指向旧位置——已建立的 HTTP 连接、CDN 节点上正在进行的 range 请求、客户端 SDK 缓存的直连地址、正在重试的失败请求。这些的生命周期可达数分钟。

**后果：** 第一种是全量事故；第二种是**静默数据丢失**（最坏的一类，因为发现时已无法恢复，且往往在几周后才被用户投诉暴露）；第三种表现为迁移期间的零星 404，比例低（约 0.02%）但排查极难——因为元数据显示一切正常，日志里只有孤立的读失败，且无法复现。

**解决方案：** 四步顺序 + 两条硬约束：

1. **删源必须延迟，且延迟时间 > 所有在途状态的最长生命周期。** 实测取 24 小时（覆盖 CDN 缓存 TTL 12 小时 + 客户端 SDK 缓存 6 小时 + 重试退避上限 1 小时的安全余量）。`media_tier_migration.source_purge_at` 就是这个延迟的落地。
2. **校验必须读全量并逐块比 CRC，不能只比长度。** 静默损坏最常见的表现恰恰是长度正确、内容错误（位翻转、错误的块被写入）。只比长度的校验对静默损坏零检出率。
3. **`step` 字段必须能区分"切读前失败"与"切读后失败"**——前者回滚（丢弃目标层数据，源不动），后者前滚（重试删源，绝不回滚，因为流量已在目标层）。合并成单一状态机就丧失了这个区分能力，恢复逻辑只能二选一，必然在另一种情形下出错。

### 陷阱 4：用「7 日播放为 0」直接删除转码副本

P04 靠 JIT 把转码副本从 350 PB 压到 53 PB，这个收益是真实的（¥699 万/月）。但判据如果就是裸的"7 日播放为 0 则删"，会在三类内容上误杀，且误杀的代价不是数据丢失而是**JIT 风暴**——比数据丢失更容易造成全站事故。

误杀本身很便宜（数据可重建）。危险在于**误杀是相关的，不是独立的**：

```
独立误杀（可承受）：
  1000 个孤立档位被误删 → 未来 7 天内零散触发 1000 次 JIT
  → 摊薄到 604,800 秒里，转码集群感知不到

相关误杀（事故）：
  某周更综艺 40 集 × 6 档位 = 240 个档位在同一天被判"7 日无播放"
  → 第 9 天新一集上线，用户追更从第 1 集看起
  → 240 个档位在 15 分钟内全部触发 JIT
  → 每档位 4K 转码需 210 核·秒，240 × 210 = 50,400 核·秒 压进 900 秒
  → 瞬时需求 56 核 → 抢占正常投稿转码槽位
  → 新投稿转码 P95 从 8 分钟涨到 41 分钟，击穿 SLA
```

**根因：内容的访问模式是相关的（同一节目、同一创作者、同一话题的内容一起冷、一起热），所以基于访问的删除判据也会产生相关的误杀，而相关误杀会在时间上聚集成风暴。** 这个"相关性把可承受的单点损失聚合成系统性事故"的模式，在容量规划里反复出现（缓存同时过期、连接池同时重连、证书同时到期）。

**后果：** 转码集群被 JIT 挤满，新投稿转码 SLA 击穿。更隐蔽的次生后果是**反馈放大**：JIT 重转期间用户拿不到目标档位，播放器降级到低档位或反复重试，触发更多 JIT 请求；同时创作者看到"上传后 40 分钟还没发布"会重新上传，进一步加压。

**解决方案：** 五条豁免规则 + JIT 预算护栏 + 延迟物理删除，三层防护各治一类风险：

| 防护 | 治什么 | 机制 |
|------|-------|------|
| 豁免规则（5 条） | 减少误杀发生 | 兜底档位永不删、周期性内容（lag 7/30 自相关）豁免、季节性内容豁免、多地域访问豁免、创作者近期活跃豁免 |
| JIT 预算（12,000 分钟/日） | 限制误杀的**聚集度** | 全局日预算 + 令牌桶限速；超预算时 JIT 请求排队并降级返回邻近档位 |
| 延迟物理删除（7 天） | 让误杀可撤销 | `revive_if_pending` 在物理删除前命中则直接恢复指针，零转码成本，实测挽回 23% 的误判 |

**关键在于第二层。** 豁免规则永远不可能完备（总有新的访问模式），所以必须有一层不依赖"判对"的护栏——**JIT 预算把"误杀多少"与"误杀造成多大冲击"解耦**：即使某天误杀了 1 万个档位，预算也保证它们对转码集群的冲击被摊平到可承受的速率。**当你无法保证判据正确时，就限制判错的后果的传播速率。**

### 陷阱 5：秒传只比对文件 hash

秒传的标准实现是客户端算 SHA-256、服务端查 `media_dedup_index`、命中则跳过上传。看起来 SHA-256 的碰撞概率可忽略（2^-128），这个设计没有问题。

问题不在密码学，在工程实现的各种"优化"上：

| 实现变体 | 为什么会被采用 | 风险 |
|---------|-------------|------|
| 只算首 1 MB 的 hash | 大文件全量 hash 太慢（4 GB 文件手机上算 40 秒） | **同一片头的不同视频被当成同一个**——影视剧集片头相同是常态 |
| 截断 SHA-256 到 64 bit | 索引更小、比较更快 | 生日界：1,240 亿对象下碰撞概率约 **17%**，几乎必然发生 |
| 用 MD5 | 历史遗留、库更普及 | MD5 可**构造**碰撞——攻击者可上传一个与已知违规视频 MD5 相同的正常视频，或反之 |
| 信任客户端上报的 hash | 省服务端算力 | 客户端可伪造 hash 命中他人媒资，**实现越权读取**（拿到别人的私密视频） |

第四项是安全漏洞而非可靠性问题：如果服务端完全信任客户端上报的 hash 并直接建立引用，那么只要知道（或猜到）某个媒资的 hash，就能在自己的内容里引用它——**这是一个未授权访问原语**。

**后果：** 前三项导致内容错乱（用户上传 A 却发布出 B，且 A 的字节从未落地，不可恢复）；第四项是数据泄露。第一项在影视类内容上尤其致命——同一部剧的 40 集共享 90 秒片头，只比首 1 MB 会把整季判为同一个文件。

**解决方案：** 四道防线，逐层收紧：

1. **全量 SHA-256，不截断，不用 MD5 做主键。** 客户端算 hash 慢就分片并行算（4 GB 文件 8 线程约 6 秒），不要用弱 hash 换速度。
2. **二次校验：`file_bytes` + `head_md5`（首 1 MB）+ `tail_md5`（末 1 MB）三者全等才算命中。** 这三个字段的作用不是提高密码学强度，而是**捕获实现 bug 与人为构造**——真正的 SHA-256 碰撞不会同时满足这三项，而截断 hash 碰撞、MD5 构造碰撞、客户端伪造几乎必然在这三项之一上暴露。
3. **秒传成功后，服务端异步抽样重算 hash**（抽样率 0.5%，从对象存储读回全量数据重算）。这是唯一能发现"索引里的 hash 与实际字节不符"的手段。
4. **对客户端上报的 hash 做归属校验**：命中的媒资若其所有引用方均与当前上传者无关联（非同一创作者、无二创授权关系），则**降级为正常上传**而不是秒传。牺牲一点去重率（实测损失 1.2 个百分点），换掉越权读取的可能性。

**判据：秒传是一个把"客户端的断言"转化为"服务端的数据引用"的操作，所有这类操作都必须假设客户端在撒谎。**

### 陷阱 6：把「删除接口返回成功」当作删除完成

删除工作流的 10 个步骤里，有 3 个作用于不在自己机房的系统：CDN 边缘缓存（约 180 个节点、多个厂商）、客户端离线下载、第三方备份。对这三类，"调用清除 API 返回 200"与"数据实际不可读"之间有巨大的鸿沟：

```
CDN purge API 返回 200 的真实含义：
  ✓ 请求格式合法
  ✓ 清除任务已进入厂商的任务队列
  ✗ 不表示已下发到边缘节点
  ✗ 不表示所有节点都成功执行
  ✗ 不表示没有节点因为磁盘故障/网络分区而漏掉
  实测：某厂商 P99 生效时间 8 分钟，但约 0.3% 的节点需要人工介入
```

**后果：** 系统认为删除完成、审计日志记录"已擦除"，但内容仍能通过某些边缘节点播放。这产生两类风险的叠加：**用户投诉与舆情**（"我删了，为什么还能搜到、还能播"，尤其在涉及个人隐私的删除上会迅速发酵）+ **监管处罰**（法院下架令或版权方通知后仍可访问，审计日志显示"已完成"反而成为"虚假报告"的证据，比诚实报告"处理中"更严重）。

**解决方案：** 把完成判据从"执行回执"改成"主动探测"：

- **步骤 9 是唯一的完成判据**：从 12 个地域的探测点用真实播放路径（拿签名 URL 请播）验证全部返回 403/404，`verify_regions_ok` 达到 12 才置 `status=unreadable_verified`。步骤 1-8 全部成功但步骤 9 未通过，任务状态仍是 `running`。
- **对每个厂商的每个探测点分别记录**，不做"抽一个地域代表全部"的简化——漏掉的恰恰是那些异常节点，抽样会系统性地漏掉它们。
- **签名密钥轮转作为兜底**：即使某个边缘节点缓存清不掉，轮转签名密钥后旧 URL 全部失效，节点上的字节变成不可寻址的垃圾（等 TTL 自然淘汰）。这一步（`_step_rotate_signing_key`）是**不依赖任何外部系统配合**的防线，因此它排在 purge 之前而不是之后。
- **`deadline_unreadable` 逾期告警直接触发人工介入**，而不是自动重试到超时。24 小时窗口内如果探测仍未全绿，说明有节点需要厂商侧人工处理，自动重试只会浪费剩余时间。

**通用原则：跨系统清理的完成判据必须是"从外部观察到目标状态"，而不是"内部调用都返回成功"。** 这条原则同样适用于缓存失效、配置下发、灰度回滚——凡是需要多个不受控系统配合的操作，都要有独立的验证通道。

### 陷阱 7：对小对象直接做 EC

EC 的成本模型在教科书里是"冗余从 3.0x 降到 1.25x"，这个数字对**大对象**成立。对小对象，它会被两个隐藏成本吃掉，甚至变成负收益。

以 EC(16,4) 处理一个 64 KB 的封面图为例：

| 隐藏成本 | 机制 | 量化 |
|---------|------|------|
| 块对齐浪费 | 64 KB ÷ 16 = 4 KB/块，但磁盘最小分配单元 + 元数据头至少 16 KB | 实际占用 20 块 × 16 KB = **320 KB，放大 5.0x**（比三副本的 3.0x 更差） |
| 元数据爆炸 | 每个对象需记录 20 个块的位置 | `placement` JSON 约 900 字节 > 对象本身的 1.4%；1,240 亿对象则元数据达 **111 TB** |
| 读放大 | 读 64 KB 要发 16 个（hedged 时 18 个）网络请求 | IOPS 需求 ×16，冷层 IOPS 密度只有热层 1/40，直接打满 |

**转折点在约 4 MB**：对象小于 4 MB 时，EC 的块对齐浪费 + 元数据开销超过冗余节省。平台的平均对象是 4.3 MB，**恰好在临界点附近**——这意味着"平均值达标"完全不能说明问题，因为对象大小的分布是极度双峰的：长视频分片 8 MB 以上，封面/缩略图/字幕/弹幕包在 64 KB 量级，中间几乎没有。**按平均值决策会让一半的对象落在错误的一侧。**

**后果：** 对小对象直接 EC 不但不省钱（放大 5.0x vs 三副本 3.0x），还把冷层的 IOPS 打满、把元数据集群压爆。且这个问题**在灰度阶段不会暴露**——灰度通常选大文件（视频）验证，小对象的问题要到全量后才显现。

**解决方案：** EC 前先打包（`_put_packed`），把 EC 的粒度从"对象"提升到"打包组"：

- 小对象（< 4 MB）追加进 256 MB 的 `media_pack`，封包后整包做一次 EC(16,4)——**4,096 个 64 KB 对象共享 20 个块与一份 placement 元数据**，元数据开销从 900 字节/对象降到 0.22 字节/对象（降 4,000 倍）。
- 读取时按 `pack_offset` + 长度做 range 读，**只需读命中的那 1 个数据块**，不需要读全部 16 块——读放大从 16x 回到 1x。
- 代价是 `dead_bytes`：组内对象删除后字节仍占用，需在 `dead_bytes / filled_bytes > 0.35` 时整包重整。这是打包方案必须接受的成本，但 35% 阈值下的重整 IO 成本远低于回收的存储成本。

**决策规则落到一行代码**：`layout = EC_PACKED if size < 4*1024*1024 else EC_STRIPED`。这一行背后是上面整张表的量化——**而它最容易被写成 `layout = EC_STRIPED`（对所有对象一视同仁），因为那样代码更简单，且在测试用的大文件上完全正常。**


