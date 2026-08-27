# Ceph 集群搭建与配置（媒资存储域配套运维手册）

> 本文是 [P05 媒资存储与生命周期](p05-media-storage-lifecycle.md) 的**落地配套**：P05 讲"字节怎么管理、怎么分层、怎么删除"，本文讲"Ceph 集群怎么搭、每个配置干什么、在哪里执行"。
>
> 配套结论（来自 P05 讨论）：三档介质 = Ceph 三个池（hot NVMe 三副本 / warm HDD 三副本 / cold SMR EC16,4），自研层只通过 RGW S3 接口交互，迁移调度器用 `CopyObject(DestinationStorageClass=…)` 驱动，**不碰 RADOS/OSD/CRUSH**。
>
> 版本基线：Ceph Reef（18.x）+ cephadm（容器化部署）。

## 一、全局架构

### 1.1 逻辑架构

```
                 ┌────────────────────────────────────────────────┐
                 │              自研层（业务，P05 代码）              │
                 │  存储调度器 / 打包器 / 去重 / 删除工作流 / 合规擦除  │
                 └────────────────────────────────────────────────┘
                                    │  S3 API（boto3）
                                    ▼
                 ┌────────────────────────────────────────────────┐
                 │           RGW（S3 门面，N 个实例 + LB）           │
                 │  StorageClass: STANDARD / STANDARD_IA / GLACIER  │
                 └───────────────┬──────────────┬──────────────┘
                                 │              │
               ┌─────────────────┘              └─────────────────┐
               ▼  STANDARD                        ▼  GLACIER        ▼  STANDARD_IA
     ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐
     │  hot.rep 池      │  │  warm.rep 池     │  │  cold.ec 池           │
     │  NVMe  3 副本    │  │  HDD   3 副本    │  │  SMR   EC(16,4)       │
     │  PG=4096         │  │  PG=4096         │  │  PG=4096              │
     └────────┬─────────┘  └────────┬─────────┘  └──────────┬───────────┘
              │                     │                       │
        ┌─────▼─────┐        ┌──────▼─────┐        ┌────────▼────────┐
        │ CRUSH 规则 │        │ CRUSH 规则  │        │ CRUSH 规则       │
        │ take nvme  │        │ take hdd   │        │ take smr, k16m4  │
        └─────┬─────┘        └──────┬─────┘        └────────┬────────┘
              └─────────────────────┴────────────────────────┘
                                   ▼
                     ┌──────────────────────────┐
                     │        RADOS 层           │
                     │  OSD（每盘一守护进程）       │
                     │  Monitor×3 + MGR          │
                     │  PG→OSD 映射、自愈、scrub   │
                     └──────────────────────────┘
                                   │
                              NVMe/HDD/SMR 物理盘
```

### 1.2 节点角色规划

| 角色 | 数量 | 机型 | 职责 |
|------|------|------|------|
| Monitor + MGR | 3 | 小规格独立机 | 集群一致性、成员管理、监控（quorum ≥ 2） |
| OSD（hot） | 若干 | NVMe | 承载 hot.rep 池，热数据 3 副本 |
| OSD（warm） | 若干 | HDD | 承载 warm.rep 池，温数据 3 副本 |
| OSD（cold） | **≥ 20** | SMR | 承载 cold.ec 池，EC(16,4) 的硬约束（20 台故障域） |
| RGW | ≥ 3（前置 LB） | 通用 | S3 网关，storage class 分发 |

**EC(16,4) + `crush-failure-domain=host` 的硬前提：SMR 机型必须 ≥ 20 台**（每个条带 20 块 shard 落在 20 台不同 host）。机器不够，要么放宽故障域到 `osd`，要么降 k/m——都牺牲可靠性，不要为了省机器做。

### 1.3 网络规划（先定，别后补）

| 网络 | CIDR 示例 | 用途 | 带宽要求 |
|------|----------|------|---------|
| 公共网络 public_network | 192.168.20.0/24 | 客户端（RGW 接入）、Monitor/MGR 管理 | ≥ 10G |
| 集群网络 cluster_network | 192.168.21.0/24 | OSD 间复制、EC 重建、backfill | **≥ 25G** |

**集群网络必须独立**：EC 重建时一块 shard 丢失要读 16 块（16x 放大），backfill/重建流量会占满网络；与客户端流量混用会让重建期间客户端体验崩掉。

### 1.4 命令执行位置的总约定（全篇通用）

| 操作 | 在哪执行 |
|------|---------|
| 集群管理（`ceph ...` / `ceph orch ...` / `ceph osd ...` / `ceph config set ...`） | **任意管理节点**（bootstrap 节点或其上 `cephadm shell`） |
| RGW 管理（`radosgw-admin ...`） | **任意 RGW 节点**（或管理节点，需能连 RGW 的 admin socket） |
| 节点本机操作（装 cephadm、看日志） | **对应节点本机**（root/sudo） |
| 客户端 S3 操作 | 应用节点（boto3 / aws cli） |

> cephadm 下**不直接改 `/etc/ceph/ceph.conf`**，所有配置统一用 `ceph config set <who> <key> <value>` 下发给集群并持久化。这是 cephadm 与老手动的最大区别。

---

## 二、部署前置

```bash
# 每个节点：装 cephadm + 时间同步 + 主机名解析
apt install -y cephadm chrony
chronyc makestep                     # 时钟必须一致，Ceph 强依赖时钟
# /etc/hosts 或 DNS 解析所有节点名（不能用 IP 代替 hostname，OSD 标识靠 hostname）
# 每节点挂载：OSD 盘直接整盘，不要分区；数据盘不挂文件系统（cephadm 会接管）
```

## 三、集群初始化

```bash
# 3.1 首个节点 bootstrap（★ 在第一个管理节点上执行）
cephadm bootstrap \
    --mon-ip 192.168.10.11 \
    --apply-mon 3 \
    --initial-dashboard-password 'change-me'

# 3.2 结果验证（★ 管理节点）
ceph -s            # 应看到 HEALTH_OK，mon 3，mgr 1
```

| 参数 | 作用 |
|------|------|
| `--mon-ip` | 指定本机 Monitor 的 IP（公共网络地址） |
| `--apply-mon 3` | 立刻把 quorum 扩到 3 个 mon（防单点） |
| `--initial-dashboard-password` | Dashboard（Web 管理）初始密码 |

## 四、节点加入与网络分离

```bash
# 4.1 把其余节点加入集群（★ 管理节点）
ceph orch host add osd-hot-01 192.168.10.21
ceph orch host add osd-hot-02 192.168.10.22
ceph orch host add osd-warm-01 192.168.10.31
ceph orch host add osd-cold-01 192.168.10.41
ceph orch host add rgw-01      192.168.10.51
...                                    # 所有节点

# 4.2 网络分离（★ 管理节点，global 作用域）
ceph config set global public_network 192.168.20.0/24
ceph config set global cluster_network 192.168.21.0/24
# 注意：mon_osd 的 public/cluster 地址在 bootstrap 时已绑定；
# 此设置在首次加入的 OSD 生效，历史 mon 地址如不符需单独改（一般 bootstrap 就对了）
```

| 配置 | 作用 | 生效范围 |
|------|------|---------|
| `public_network` | 声明客户端/管理面网段，OSD 用此网段的 IP 与 mon、客户端通信 | global（所有守护进程） |
| `cluster_network` | 声明 OSD 间数据面网段，复制/EC/重建全走此网，**与客户端流量隔离** | global |

## 五、OSD 部署与设备类隔离（三档介质的关键）

```bash
# 5.1 部署全部可用盘（★ 管理节点）
ceph orch apply osd --all-available-devices

# 更精细：按节点/标签分批（避免一次性全量）
ceph orch apply osd -i osd_spec.yaml      # yaml 里按 hosts + 过滤条件分批
# 例：只认未格式化整盘
# service_type: osd
# service_id: cold-osds
# placement: {hosts: [osd-cold-01..20]}
# data_devices: {all: true}

# 5.2 设备类确认（NVMe/HDD 自动识别；SMR 手动归类）
ceph osd crush class ls                   # 应看到 nvme / hdd / smr

# SMR 机型上的盘 cephadm 可能识别为 hdd，需手动改为 smr：
for osd in $(ceph osd ls-tree osd-cold-01); do
    ceph osd crush set-device-class smr "$osd"
done

# 5.3 校验 OSD 状态与设备类分布
ceph osd tree                             # 按 host 看每台 OSD 与 class
ceph osd df                               # 容量视图
```

| 动作 | 作用 | 在哪执行 |
|------|------|---------|
| `orch apply osd` | 声明式部署 OSD，cephadm 自动创建并接管整盘 | 管理节点 |
| `osd crush set-device-class smr` | 手动把 OSD 归为 smr 类，供 CRUSH 规则按介质选盘 | 管理节点 |
| `osd crush class ls` | 验证三类设备齐全（nvme/hdd/smr） | 管理节点 |

> **设备类是"三档介质"能否成立的根基**：CRUSH 规则 `take <class>` 选盘，靠的就是每块盘被正确归类。归类错了，热数据会落进 SMR 盘。

## 六、CRUSH 规则 + EC Profile + 三个池

### 6.1 EC Profile（★ 管理节点）

```bash
ceph osd erasure-code-profile set cold_ec_16_4 \
    k=16 \
    m=4 \
    stripe_unit=4194304 \
    crush-failure-domain=host \
    crush-device-class=smr \
    plugin=isa
```

| 参数 | 作用 |
|------|------|
| `k=16 m=4` | 16 数据块 + 4 校验块，冗余 1.25x，容错任意 4 块（对应 P05 的 EC(16,4)） |
| `stripe_unit=4194304` | 单 shard 4MB（对齐 P05 的 `STRIPE_UNIT=4MB`，条带宽 64MB） |
| `crush-failure-domain=host` | 20 块 shard 分散到 20 台不同 host（可靠性的硬约束） |
| `crush-device-class=smr` | 只允许数据落在 SMR 盘上（三档介质隔离） |
| `plugin=isa` | 用 ISA 加速库的 RS 实现（Intel），性能优于默认 jerasure |

### 6.2 CRUSH 规则（★ 管理节点）

```bash
# hot/warm 是副本池规则（按设备类选盘）
ceph osd crush rule create-replicated hot-rule  default host nvme
ceph osd crush rule create-replicated warm-rule default host hdd
# cold 的 EC 规则由 profile 建池时自动生成（cold_ec_16_4）
```

| 规则 | 作用 |
|------|------|
| `hot-rule` | 副本池规则：数据只落在 nvme 设备类，host 故障域 |
| `warm-rule` | 同上，hdd 设备类 |
| `cold_ec_16_4`（隐式） | EC 规则：smr 设备类 + host 故障域 |

### 6.3 建池（★ 管理节点）

```bash
# 副本池
ceph osd pool create hot.rep  4096 replicated hot-rule
ceph osd pool set hot.rep  size 3 min_size 2
ceph osd pool create warm.rep 4096 replicated warm-rule
ceph osd pool set warm.rep size 3 min_size 2

# EC 池（冗余由 profile 决定，不用设 size）
ceph osd pool create cold.ec 4096 erasure cold_ec_16_4

# PG 自动扩缩（全局开，见配置清单）
```

| 池 | 冗余 | 介质 | 作用 |
|----|------|------|------|
| `hot.rep` | 3 副本（min_size 2） | NVMe | 热数据，P05 的 Hot 层 |
| `warm.rep` | 3 副本（min_size 2） | HDD | 温数据，P05 的 Warm 层 |
| `cold.ec` | EC(16,4)=1.25x | SMR | 冷数据，P05 的 Cold 层 |

> `size 3 / min_size 2`：3 副本下可容忍 2 盘同时故障；min_size 2 是"只剩 2 副本仍可写"的降级下限（再低宁可不写，防止丢最后一致）。**min_size 不是想设多低就多低——它是数据安全的最后防线。**

## 七、RGW 部署与 Storage Class 绑定

```bash
# 7.1 部署 RGW（★ 管理节点）
ceph orch apply rgw default --replicas 3
# 前置一个负载均衡（nginx/LB），对外暴露 7480

# 7.2 绑定三个 storage class → 三个池（★ RGW 节点或管理节点）
radosgw-admin zone placement modify \
    --rgw-zone default --placement-id default-placement \
    --storage-class STANDARD    --data-pool-pool hot.rep
radosgw-admin zone placement modify \
    --rgw-zone default --placement-id default-placement \
    --storage-class STANDARD_IA --data-pool-pool warm.rep
radosgw-admin zone placement modify \
    --rgw-zone default --placement-id default-placement \
    --storage-class GLACIER     --data-pool-pool cold.ec

# 7.3 提交配置（★ RGW 节点）
radosgw-admin period update --commit

# 7.4 创建 S3 账号（★ RGW 节点，供自研层 boto3 使用）
radosgw-admin user create --uid=media-svc \
    --display-name='media-storage-svc' \
    --access-key=MEDIAAK... --secret-key=...
# 返回的 access_key/secret 写入自研配置中心
```

| 命令 | 作用 | 在哪执行 |
|------|------|---------|
| `orch apply rgw` | 容器化部署 RGW 实例 | 管理节点 |
| `zone placement modify` | 把 storage class 绑到数据池（storage class ↔ 池 一一对应） | RGW 节点 |
| `period update --commit` | 把 zone 配置提交到 period 并全网生效 | RGW 节点 |
| `user create` | 建 S3 账号（媒体服务专用 key） | RGW 节点 |

> **校验 storage class 是否生效**（★ RGW 节点）：
> ```bash
> radosgw-admin zone placement get
> # 应看到 STANDARD→hot.rep / STANDARD_IA→warm.rep / GLACIER→cold.ec
> ```

---

## 八、配置清单：每个配置的作用与执行位置

以下按"作用域"分组。**执行位置统一在管理节点（`ceph config set` 类）或 RGW 节点（`radosgw-admin` 类）**，不再逐条重复说明；每组给出生效范围。

### 8.1 全局配置（global，作用于所有守护进程）

| 配置项 | 作用 | 命令 |
|--------|------|------|
| `public_network` | 客户端/管理面网段 | `ceph config set global public_network 192.168.20.0/24` |
| `cluster_network` | OSD 数据面网段，隔离复制/EC 流量 | `ceph config set global cluster_network 192.168.21.0/24` |
| `osd_pool_default_size` | 新建池默认副本数（=3） | `ceph config set global osd_pool_default_size 3` |
| `osd_pool_default_min_size` | 默认 min_size（=2） | `ceph config set global osd_pool_default_min_size 2` |
| `osd_pool_default_pg_autoscale_mode` | 开 PG 自动扩缩，免手工算 PG | `ceph config set global osd_pool_default_pg_autoscale_mode on` |
| `mon_osd_full_ratio` | OSD 容量 95% 视为 full，**阻塞写入** | `ceph config set mon mon_osd_full_ratio 0.95` |
| `mon_osd_backfillfull_ratio` | OSD 92% 暂停 backfill（防重建把盘写满） | `ceph config set mon mon_osd_backfillfull_ratio 0.92` |
| `osd_pool_default_crush_rule` | 默认 CRUSH 规则（兜底，新池不指定时用） | `ceph config set global osd_pool_default_crush_rule hot-rule` |

**执行位置：管理节点 `ceph` CLI。生效范围：`global` = 全部守护进程（含未来新增）。**

### 8.2 Monitor 配置（mon 子系统）

| 配置项 | 作用 | 命令 |
|--------|------|------|
| `mon_max_pg_per_osd` | 每 OSD 最多 PG 数上限，防 PG 爆炸拖垮 mon | `ceph config set mon mon_max_pg_per_osd 400` |
| `mon_pg_warn_max_object_skew` | 对象分布倾斜告警阈值 | `ceph config set mon mon_pg_warn_max_object_skew 5` |
| `mon_warn_on_slow_ping_ratio` | 慢 OSD 检测灵敏度（对 EC 条带读重要） | `ceph config set mon mon_warn_on_slow_ping_ratio 0.05` |

**执行位置：管理节点。生效范围：mon 子系统（Monitor 进程）。**

### 8.3 OSD 配置（osd 子系统）——媒体场景最需要调的一组

| 配置项 | 作用 | 命令 |
|--------|------|------|
| `osd_max_backfills` | 单 OSD 并发 backfill 数（**=2**，防 EC 重建 16x 流量打满集群网） | `ceph config set osd osd_max_backfills 2` |
| `osd_recovery_max_active` | 单 OSD 并发恢复数（=2，同上） | `ceph config set osd osd_recovery_max_active 2` |
| `osd_scrub_begin_hour` / `osd_scrub_end_hour` | 常规 scrub 窗口（凌晨 2-6 点，避开播放高峰） | `ceph config set osd osd_scrub_begin_hour 2`<br>`ceph config set osd osd_scrub_end_hour 6` |
| `osd_deep_scrub_interval` | 深 scrub 周期（14 天；深 scrub 读全量数据，太勤伤带宽） | `ceph config set osd osd_deep_scrub_interval 1209600` |
| `osd_scrub_during_recovery` | 恢复期间暂停 scrub（资源让给重建） | `ceph config set osd osd_scrub_during_recovery false` |
| `osd_fast_fail_on_corruption` | 检测到磁盘损坏立即标 OSD 为 failed（不等超时） | `ceph config set osd osd_fast_fail_on_corruption true` |
| `osd_pool_default_min_size` | 已在 global；OSD 级可覆盖单池 | — |
| `osd_heartbeat_grace` | OSD 心跳超时（慢盘/GC 抖动时调大） | `ceph config set osd osd_heartbeat_grace 25` |

**执行位置：管理节点。生效范围：osd 子系统（所有 OSD 进程）。**
**针对 SMR 冷池的额外建议**：SMR 顺序写盘，随机覆写放大严重——`osd_max_backfills 1`（冷池 OSD 用 `ceph config set osd.0 osd_max_backfills 1` 按实例覆盖）。

### 8.4 RGW 配置（rgw 子系统）

| 配置项 | 作用 | 命令 |
|--------|------|------|
| `rgw_bucket_index_max_shards` | 大桶索引分片数（**=64**，应对 1,240 亿对象的桶索引压力） | `ceph config set rgw rgw_bucket_index_max_shards 64` |
| `rgw_multipart_min_part_size` | multipart 最小段大小（=4MB，对齐自研打包粒度） | `ceph config set rgw rgw_multipart_min_part_size 4194304` |
| `rgw_max_chunk_size` | 单段读取/写入块大小（默认即可，大对象读调大） | `ceph config set rgw rgw_max_chunk_size 4194304` |
| `rgw_enable_usage_log` | 开启用量日志（自研成本归因/对账的数据源） | `ceph config set rgw rgw_enable_usage_log true` |
| `rgw_lc_debug_interval` | lifecycle 调试周期（上线期观察用，生产关） | `ceph config set rgw rgw_lc_debug_interval 60` |
| `rgw_frontends` | RGW HTTP 前端（=beast，端口 7480） | `ceph config set rgw rgw_frontends beast port=7480` |

**执行位置：管理节点（`ceph config set rgw ...`）。生效范围：rgw 子系统（所有 RGW 实例）。**

### 8.5 池级配置（pool 参数，随池走）

| 配置项 | 作用 | 命令 |
|--------|------|------|
| `size / min_size` | 副本池冗余与降级下限 | `ceph osd pool set hot.rep size 3`<br>`ceph osd pool set hot.rep min_size 2` |
| `pg_autoscale_mode` | 池内 PG 自动扩缩 | `ceph osd pool set hot.rep pg_autoscale_mode on` |
| `pg_num / pgp_num` | PG 数量（建池时定，扩可、缩难） | 建池时 `--pg_num` |
| `application` | 标记用途（rgw），便于管理 | `ceph osd pool application enable hot.rep rgw` |

**执行位置：管理节点。生效范围：对应池。**

### 8.6 RGW 运维类（radosgw-admin，执行于 RGW 节点）

| 命令 | 作用 |
|------|------|
| `radosgw-admin zone placement get` | 查看 storage class ↔ 池绑定 |
| `radosgw-admin bucket check` | 桶索引一致性检查（配合自研 ref_count 对账） |
| `radosgw-admin sync status` | 跨地域（multisite）复制收敛状态 |
| `radosgw-admin md export` | 导出 RGW 元数据（配置/账号备份） |
| `radosgw-admin user info --uid=...` | 查 S3 账号 |
| `radosgw-admin lc list` | 查看 lifecycle 任务 |

### 8.7 加密落盘（合规擦除的前提，见 P05 讨论）

| 方式 | 作用 | 在哪执行 |
|------|------|---------|
| OSD 级 dmcrypt（部署时启用） | 数据落盘即加密；"不可恢复"靠密钥销毁而非覆写 | 管理节点，部署 OSD 时（`--all-available-devices` + 加密选项 / 独立加密 key） |
| `ceph auth` 管理 | 控制谁连集群 | 管理节点 |

---

## 九、与自研层的接口契约（写进配置中心的常量）

| 契约项 | 值 | 自研层用途 |
|--------|-----|-----------|
| endpoint | `https://rgw.svc:7480` | boto3 S3 客户端连接 |
| bucket 命名 | `media-{region}-{tier}` | storage_key = `bucket/key` |
| storage class 常量 | `STANDARD` / `STANDARD_IA` / `GLACIER` | 迁移调度器 `CopyObject(DestinationStorageClass=…)` |
| S3 凭据 | media-svc 的 access/secret（每服务一 key） | 签名访问 |
| ETag 校验策略 | multipart 对象的 ETag 非简单 MD5 | 迁移校验须自研抽样 Range 读重算 hash，勿只信 ETag |

## 十、上线后运维清单

| 周期 | 事项 | 命令/工具 | 在哪执行 |
|------|------|----------|---------|
| 实时 | 容量水位告警（>85%） | Dashboard / `ceph df` | 管理节点 |
| 实时 | 慢 OSD 检测（`osd perf` 异常即隔离） | `ceph osd perf` | 管理节点 |
| 每日 | 集群健康检查 | `ceph -s` + Dashboard 告警 | 管理节点 |
| 每日 | 桶一致性抽查 | `radosgw-admin bucket check` | RGW 节点 |
| 月度 | 故障演练：主动坏一块盘，验证 EC 重建 | `ceph osd out <id>` | 管理节点 |
| 月度 | 跨地域收敛演练 | `radosgw-admin sync status` | RGW 节点 |
| 升级时 | 小版本升级（RGW 与 OSD 分批） | `ceph orch upgrade start` | 管理节点 |

## 十一、常见故障速查

| 现象 | 可能原因 | 排查/处置 |
|------|---------|----------|
| `HEALTH_ERR` 满盘 | full_ratio 命中 | 扩容 + 查是否有泄漏对象（配合 P05 reconcile） |
| 重建缓慢、客户端卡 | backfill 抢带宽 | 调小 `osd_max_backfills` / `osd_recovery_max_active` |
| 某池读慢但集群健康 | 设备类归类错（数据落在错误介质） | `ceph osd crush class ls` + `ceph osd tree` 核对 |
| scrub 拖垮高峰播放 | scrub 窗口不对 | 调 `osd_scrub_begin_hour/end_hour` |
| multipart 上传 500 | 段大小 < `rgw_multipart_min_part_size` | 对齐自研打包粒度（4MB） |

---

**一份文档收尾的结论**：本方案的关键路径是「**bootstrap → 加节点 → OSD 设备类（nvme/hdd/smr）→ CRUSH 规则 → 三池（hot/warm/cold）→ RGW storage class 绑定**」六步，配通后自研层即获得 P05 的三档介质能力。**绝大多数调优集中在 `osd` 子系统（重建限速、scrub 窗口）和 `rgw` 子系统（桶索引分片、multipart 对齐）**——这两组参数就是媒体存储场景与通用场景的差异所在。