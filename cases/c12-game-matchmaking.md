# C12: 游戏服的匹配与房间管理

## 业务场景

某竞技游戏平台（类似王者荣耀/LOL），需要实现玩家匹配和游戏房间管理：

- 匹配系统：根据玩家段位、胜率匹配实力相近的对手
- 房间管理：创建游戏房间、管理玩家进出、维护游戏状态
- 有状态服务：游戏服务器必须维护房间状态（玩家位置、技能CD、血量等），不能随意重启

**已知数据：**
- 日活玩家：1000 万
- 峰值在线：300 万
- 匹配 QPS：5 万/秒（高峰期）
- 单局游戏时长：15-30 分钟
- 匹配等待时间目标：< 30 秒
- 单台游戏服务器容量：500 个房间（每房间 10 人）

## 核心挑战

### 挑战 1：匹配的公平性 vs 等待时间

严格按段位匹配 → 匹配质量高但等待时间长（钻石段位玩家少，可能等 5 分钟）。放宽匹配范围 → 等待短但体验差（青铜匹配到黄金，被碾压）。

### 挑战 2：有状态服务的弹性伸缩

游戏服务器是有状态的——房间进行中不能迁移到其他服务器。扩容时新服务器只能承接新房间，缩容时必须等房间结束。

### 挑战 3：匹配与房间分配的解耦

匹配系统找到 10 个玩家后，需要分配一个游戏房间。但游戏服务器可能已满，需要找到有空位的服务器。

### 挑战 4：防作弊

匹配结果对玩家不可见（防止玩家利用匹配信息作弊），但匹配过程需要实时计算。

## 设计约束

- 匹配延迟 < 30 秒（90% 的玩家）
- 游戏服务器不可在房间进行中迁移或重启
- 匹配算法需要考虑：段位、胜率、近期表现、组队情况
- 单台游戏服务器 8 核 16GB

## 请先独立思考（限时 35 分钟）

1. 匹配算法如何设计？ELO/MMR 系统如何与匹配队列结合？
2. 有状态游戏服务器的弹性伸缩策略：如何决定何时扩容、何时缩容？
3. 匹配成功后如何分配游戏房间？如果所有服务器都满了怎么办？
4. 玩家组队（3人组队匹配）如何影响匹配算法？

---

## 数据库设计

### 核心表结构

匹配系统需要支撑高并发写入（入队/出队）和低延迟查询（匹配计算），因此采用 **Redis + MySQL** 混合架构：

- Redis：匹配队列、实时房间状态、服务器负载缓存
- MySQL：持久化玩家数据、对局记录、结算数据

#### 玩家状态表（player_profile）

```sql
CREATE TABLE player_profile (
    player_id       BIGINT PRIMARY KEY,
    username        VARCHAR(64) NOT NULL,
    mmr             INT NOT NULL DEFAULT 1000 COMMENT 'Match Making Rating，连续值',
    tier            VARCHAR(16) NOT NULL DEFAULT 'bronze' COMMENT '段位（仅展示用）',
    tier_score      INT NOT NULL DEFAULT 0 COMMENT '段位积分（0-100，满100升段）',
    total_games     INT NOT NULL DEFAULT 0,
    win_count       INT NOT NULL DEFAULT 0,
    lose_count      INT NOT NULL DEFAULT 0,
    win_rate        DECIMAL(5,4) NOT NULL DEFAULT 0.5000 COMMENT '胜率',
    recent_10_win   TINYINT NOT NULL DEFAULT 0 COMMENT '近10局胜场数',
    last_game_time  DATETIME COMMENT '最近一局结束时间',
    region          VARCHAR(8) NOT NULL DEFAULT 'cn-east' COMMENT '所在区域',
    is_penalty      TINYINT NOT NULL DEFAULT 0 COMMENT '是否在惩罚中（挂机/退出）',
    penalty_until   DATETIME COMMENT '惩罚到期时间',
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_mmr (mmr),
    INDEX idx_tier (tier),
    INDEX idx_region (region)
) ENGINE=InnoDB COMMENT='玩家档案';
```

#### 匹配队列实时表（match_queue_entry，Redis 为主，MySQL 做持久化备份）

```sql
CREATE TABLE match_queue_entry (
    entry_id        BIGINT PRIMARY KEY AUTO_INCREMENT,
    player_id       BIGINT NOT NULL,
    mmr             INT NOT NULL COMMENT '入队时的MMR快照',
    queue_type      VARCHAR(16) NOT NULL COMMENT 'solo/duo/trio/ пяти',
    party_id        VARCHAR(64) COMMENT '组队ID，单人则为NULL',
    region          VARCHAR(8) NOT NULL,
    enter_time      DATETIME NOT NULL COMMENT '入队时间',
    max_wait_ms     INT NOT NULL DEFAULT 60000 COMMENT '最大等待毫秒数',
    status          VARCHAR(16) NOT NULL DEFAULT 'waiting' COMMENT 'waiting/matched/timedout/cancelled',
    matched_room_id VARCHAR(64) COMMENT '匹配成功后关联的房间ID',
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE INDEX idx_player (player_id),
    INDEX idx_status_mmr (status, mmr),
    INDEX idx_party (party_id)
) ENGINE=InnoDB COMMENT='匹配队列入档记录';
```

Redis 中的匹配队列结构：

```
# 按段位分桶的 Sorted Set（score = MMR）
match:queue:{tier}        → ZSET  {player_id: mmr}

# 玩家入队时间（用于等待时间计算）
match:wait:{player_id}    → STRING  ISO8601 timestamp, TTL=120s

# 玩家组队信息
match:party:{party_id}    → HASH  {member_id: mmr, ...}

# 匹配成功暂存（等待玩家确认）
match:confirmed:{match_id} → SET  {confirmed_player_ids}
match:confirm_ttl:{match_id} → STRING  TTL=15s
```

#### 房间状态表（game_room）

```sql
CREATE TABLE game_room (
    room_id         VARCHAR(64) PRIMARY KEY,
    server_id       VARCHAR(64) NOT NULL COMMENT '所在游戏服务器ID',
    game_mode       VARCHAR(16) NOT NULL COMMENT 'ranked/casual/custom',
    map_name        VARCHAR(32) NOT NULL,
    status          VARCHAR(16) NOT NULL COMMENT 'creating/loading/countdown/playing/paused/finished/settled',
    team1_players   JSON NOT NULL COMMENT '[{"player_id":xxx,"mmr":xxx,"champion":"xxx"},...]',
    team2_players   JSON NOT NULL,
    team1_avg_mmr   INT NOT NULL COMMENT '队伍1平均MMR',
    team2_avg_mmr   INT NOT NULL COMMENT '队伍2平均MMR',
    mmr_diff        INT NOT NULL COMMENT '两队平均MMR差值，用于匹配质量评估',
    create_time     DATETIME NOT NULL,
    start_time      DATETIME COMMENT '游戏正式开始时间',
    end_time        DATETIME COMMENT '游戏结束时间',
    winner          VARCHAR(8) COMMENT 'team1/team2/draw',
    duration_sec    INT COMMENT '对局时长（秒）',
    INDEX idx_server (server_id),
    INDEX idx_status (status),
    INDEX idx_create_time (create_time)
) ENGINE=InnoDB COMMENT='游戏房间';
```

#### 游戏服务器状态表（game_server）

```sql
CREATE TABLE game_server (
    server_id       VARCHAR(64) PRIMARY KEY,
    region          VARCHAR(8) NOT NULL,
    endpoint        VARCHAR(128) NOT NULL COMMENT 'gRPC连接地址',
    max_rooms       INT NOT NULL DEFAULT 500 COMMENT '最大房间容量',
    active_rooms    INT NOT NULL DEFAULT 0 COMMENT '当前活跃房间数',
    status          VARCHAR(16) NOT NULL DEFAULT 'initializing' COMMENT 'initializing/available/draining/terminated',
    cpu_usage       DECIMAL(5,2) COMMENT 'CPU使用率%',
    mem_usage_mb    INT COMMENT '内存使用MB',
    last_heartbeat  DATETIME NOT NULL COMMENT '最后心跳时间',
    started_at      DATETIME NOT NULL COMMENT '服务器启动时间',
    version         VARCHAR(16) NOT NULL COMMENT '服务版本号',
    INDEX idx_status (status),
    INDEX idx_region_status (region, status),
    INDEX idx_heartbeat (last_heartbeat)
) ENGINE=InnoDB COMMENT='游戏服务器';
```

#### 对局记录表（match_history）

```sql
CREATE TABLE match_history (
    record_id       BIGINT PRIMARY KEY AUTO_INCREMENT,
    room_id         VARCHAR(64) NOT NULL,
    player_id       BIGINT NOT NULL,
    team            VARCHAR(8) NOT NULL COMMENT 'team1/team2',
    is_winner       TINYINT NOT NULL,
    mmr_before      INT NOT NULL COMMENT '赛前MMR',
    mmr_after       INT NOT NULL COMMENT '赛后MMR',
    mmr_change      INT NOT NULL COMMENT 'MMR变化量（正为加）',
    kills           SMALLINT NOT NULL DEFAULT 0,
    deaths          SMALLINT NOT NULL DEFAULT 0,
    assists         SMALLINT NOT NULL DEFAULT 0,
    damage_dealt    INT NOT NULL DEFAULT 0 COMMENT '总伤害',
    gold_earned     INT NOT NULL DEFAULT 0 COMMENT '总金币',
    play_duration   INT NOT NULL COMMENT '本局时长秒',
    champion_used   VARCHAR(32) COMMENT '使用英雄',
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_player (player_id),
    INDEX idx_room (room_id),
    INDEX idx_player_time (player_id, created_at)
) ENGINE=InnoDB COMMENT='对局记录';
```

### 数据一致性策略

| 数据 | 主存储 | 同步策略 | 一致性要求 |
|------|--------|---------|-----------|
| 匹配队列 | Redis | MySQL 异步落盘 | 最终一致（允许丢少量入队记录） |
| 房间状态 | 游戏服务器内存 | MySQL 定期快照 | 最终一致（崩溃时从快照恢复） |
| 玩家 MMR | MySQL | Redis 缓存 | 强一致（写入MySQL后更新缓存） |
| 服务器负载 | Redis | — | 弱一致（心跳推送，容忍短暂过期） |
| 对局结算 | MySQL | — | 强一致（事务写入） |

## 设计解析

### 匹配系统架构

```
玩家点击"开始匹配" → 匹配服务 → 匹配队列（按段位分桶）
                              │
                              ├─ 定时器（每2秒）→ 从队列中取出玩家 → 匹配算法
                              │
                              └─ 匹配成功 → 房间分配 → 游戏服务器 → 通知玩家
```

### 匹配队列：按段位分桶

```python
class MatchQueue:
    """按段位分桶的匹配队列"""
    
    TIERS = {
        "bronze":   (0, 1200),
        "silver":   (1200, 1600),
        "gold":     (1600, 2000),
        "platinum": (2000, 2400),
        "diamond":  (2400, 2800),
        "master":   (2800, 3200),
        "challenger": (3200, 9999),
    }
    
    def enqueue(self, player):
        """玩家加入匹配队列"""
        tier = self.get_tier(player.mmr)
        
        # 写入该段位的队列（Redis Sorted Set，score=MMR）
        self.redis.zadd(
            f"match:queue:{tier}",
            {player.id: player.mmr}
        )
        
        # 记录入队时间（用于等待时间扩展）
        self.redis.set(
            f"match:wait:{player.id}", now().isoformat(), ex=120
        )
    
    def get_tier(self, mmr):
        for name, (low, high) in self.TIERS.items():
            if low <= mmr < high:
                return name
        return "challenger"
```

### ELO/MMR 计算引擎

匹配系统的核心是 MMR（Match Making Rating），采用改进的 ELO 算法。与经典 ELO 只考虑胜负不同，我们融合了对局表现因素：

```python
import math

class MMRCalculator:
    """MMR 计算引擎 — 改进 ELO 算法"""

    # K 因子：决定每局 MMR 变化幅度
    # 新手 K 值大（快速定位），老手 K 值小（稳定段位）
    K_FACTOR = {
        "new":      40,  # 前 30 局
        "normal":   24,  # 30-100 局
        "stable":   16,  # 100 局以上
        "top":      10,  # 大师及以上（防止顶端剧烈波动）
    }

    def get_k_factor(self, player):
        if player.total_games < 30:
            return self.K_FACTOR["new"]
        elif player.total_games < 100:
            return self.K_FACTOR["normal"]
        elif player.mmr >= 2800:
            return self.K_FACTOR["top"]
        else:
            return self.K_FACTOR["stable"]

    def calculate_expected_score(self, player_mmr, opponent_avg_mmr):
        """
        计算期望胜率（经典 ELO 公式）
        E = 1 / (1 + 10^((Rb - Ra) / 400))
        """
        return 1.0 / (1.0 + 10 ** ((opponent_avg_mmr - player_mmr) / 400))

    def calculate_mmr_change(self, player, opponent_avg_mmr, is_winner,
                             team_mmr_diff=0, performance_score=1.0):
        """
        计算单局 MMR 变化

        参数:
            player: 玩家对象
            opponent_avg_mmr: 对方队伍平均 MMR
            is_winner: 是否获胜
            team_mmr_diff: 己方与对方 MMR 差值（正值=己方更强）
            performance_score: 表现系数 (0.5-1.5)，基于 KDA/伤害/经济等
        """
        expected = self.calculate_expected_score(player.mmr, opponent_avg_mmr)
        actual = 1.0 if is_winner else 0.0

        k = self.get_k_factor(player)

        # 基础 MMR 变化
        base_change = k * (actual - expected)

        # 表现系数调整：赢了但表现差（挂机/蹭局势）少加分
        # 输了但表现好（尽力局）少扣分
        adjusted_change = base_change * performance_score

        # 爆冷加成：以弱胜强额外加分（激励挑战高段位）
        if is_winner and team_mmr_diff < -200:
            upset_bonus = abs(team_mmr_diff) * 0.02  # 弱200分额外+4
            adjusted_change += upset_bonus

        # 连胜/连败加速：让状态好的玩家更快上分，状态差更快掉分
        if player.recent_10_win >= 8:
            adjusted_change *= 1.3   # 10局赢8局以上，加速上分
        elif player.recent_10_win <= 2:
            adjusted_change *= 1.3   # 10局赢2局以下，加速掉分（回到合适段位）

        # 整数化，最小变化 ±1
        change = int(round(adjusted_change))
        if change == 0:
            change = 1 if is_winner else -1

        return change

    def batch_update(self, room_result):
        """
        一局游戏结束后，批量更新所有玩家 MMR

        确保同一局中双方 MMR 变化总量趋于平衡
        """
        team1_avg = sum(p.mmr for p in room_result.team1) / len(room_result.team1)
        team2_avg = sum(p.mmr for p in room_result.team2) / len(room_result.team2)

        changes = []
        winners = room_result.team1 if room_result.winner == "team1" else room_result.team2
        losers = room_result.team2 if room_result.winner == "team1" else room_result.team1

        for player in winners:
            perf = self._calc_performance_score(player, room_result)
            change = self.calculate_mmr_change(
                player, team2_avg, is_winner=True,
                team_mmr_diff=team1_avg - team2_avg,
                performance_score=perf
            )
            changes.append((player, change))

        for player in losers:
            perf = self._calc_performance_score(player, room_result)
            change = self.calculate_mmr_change(
                player, team1_avg, is_winner=False,
                team_mmr_diff=team2_avg - team1_avg,
                performance_score=perf
            )
            changes.append((player, change))

        # 事务性写入：所有 MMR 变更必须原子生效
        self._apply_mmr_changes(changes)
        return changes

    def _calc_performance_score(self, player, room_result):
        """
        计算玩家表现系数 (0.5 ~ 1.5)

        基于 KDA、伤害占比、经济占比综合评估
        """
        stats = room_result.get_player_stats(player.id)
        if not stats:
            return 1.0

        # KDA 评分 (0.3 ~ 1.0)
        kda = (stats.kills + stats.assists) / max(stats.deaths, 1)
        kda_score = min(kda / 8.0, 1.0)  # KDA=8 满分

        # 伤害占比评分
        team_total_damage = room_result.get_team_total_damage(player.team)
        damage_ratio = stats.damage_dealt / max(team_total_damage, 1)
        damage_score = min(damage_ratio * 5, 1.0)  # 20%伤害占比满分

        # 综合评分 → 映射到 0.5-1.5
        raw = kda_score * 0.5 + damage_score * 0.5
        return 0.5 + raw  # 映射到 [0.5, 1.5]

    def _apply_mmr_changes(self, changes):
        """事务性批量更新 MMR"""
        with self.db.transaction():
            for player, change in changes:
                new_mmr = max(0, player.mmr + change)  # MMR 不低于 0
                self.db.execute(
                    "UPDATE player_profile SET mmr = %s, total_games = total_games + 1, "
                    "win_count = win_count + %s, lose_count = lose_count + %s, "
                    "updated_at = NOW() WHERE player_id = %s",
                    (new_mmr, 1 if change > 0 else 0, 1 if change < 0 else 0, player.id)
                )
                # 更新 Redis 缓存
                self.redis.set(f"player:mmr:{player.id}", new_mmr)
```

**MMR 变化示例：**

| 场景 | 玩家 MMR | 对手平均 MMR | K 因子 | 基础变化 | 表现系数 | 最终变化 |
|------|---------|-------------|--------|---------|---------|---------|
| 新手黄金胜 | 1800 | 1800 | 40 | +20 | 1.2 | +24 |
| 老手钻石胜 | 2500 | 2400 | 16 | +6 | 1.0 | +6 |
| 老手钻石负 | 2500 | 2400 | 16 | -10 | 1.0 | -10 |
| 弱队爆冷胜 | 1600 | 2000 | 24 | +20 | 1.3 | +29 |
| 尽力局输 | 1800 | 1900 | 24 | -14 | 0.6 | -8 |

### 匹配算法：MMR + 等待时间扩展 + 技能维度

**核心思路：** 刚入队的玩家严格匹配同段位，等待越久匹配范围越宽。在 MMR 之外，还考虑近期胜率趋势和英雄熟练度维度。

```python
class MatchMaker:
    TEAM_SIZE = 5   # 5v5
    
    def try_match(self, tier):
        """尝试从某个段位桶中匹配一局游戏"""
        queue_key = f"match:queue:{tier}"
        
        # 获取该桶中所有等待的玩家
        candidates = self.redis.zrange(queue_key, 0, -1, withscores=True)
        
        if len(candidates) < self.TEAM_SIZE * 2:
            return None  # 人数不够
        
        # 按等待时间排序（等得久的优先）
        candidates_with_wait = []
        for player_id, mmr in candidates:
            wait_start = self.redis.get(f"match:wait:{player_id}")
            wait_seconds = (now() - parse(wait_start)).total_seconds()
            candidates_with_wait.append((player_id, mmr, wait_seconds))
        
        # 按等待时间降序排序
        candidates_with_wait.sort(key=lambda x: -x[2])
        
        # 贪心匹配：从等待最久的玩家开始，找 MMR 接近的队友
        team1 = []
        team2 = []
        
        for player_id, mmr, wait in candidates_with_wait:
            # 计算当前允许的 MMR 范围（等待越久范围越宽）
            mmr_range = self.calculate_mmr_range(wait)
            
            if len(team1) < self.TEAM_SIZE:
                if not team1 or abs(mmr - team1[0][1]) <= mmr_range:
                    team1.append((player_id, mmr))
                    continue
            
            if len(team2) < self.TEAM_SIZE:
                if not team2 or abs(mmr - team2[0][1]) <= mmr_range:
                    team2.append((player_id, mmr))
                    continue
        
        if len(team1) == self.TEAM_SIZE and len(team2) == self.TEAM_SIZE:
            return MatchResult(team1=team1, team2=team2)
        
        return None

    def try_match_with_skill_balance(self, tier):
        """
        改进匹配算法：在 MMR 基础上增加技能平衡维度
        确保两队的角色分布（坦克/输出/辅助）尽量均衡
        """
        queue_key = f"match:queue:{tier}"
        candidates = self.redis.zrange(queue_key, 0, -1, withscores=True)

        if len(candidates) < self.TEAM_SIZE * 2:
            return None

        # 获取玩家完整信息（MMR + 角色偏好 + 近期表现）
        player_infos = []
        for player_id, mmr in candidates:
            info = self.get_player_match_info(player_id)
            info.mmr = mmr
            wait_start = self.redis.get(f"match:wait:{player_id}")
            info.wait_seconds = (now() - parse(wait_start)).total_seconds()
            player_infos.append(info)

        # 按等待时间降序排序
        player_infos.sort(key=lambda x: -x.wait_seconds)

        # 使用二分图匹配：将 10 个玩家分成两队，最小化队间 MMR 差
        # 先贪心选出 10 个候选人
        selected = player_infos[:self.TEAM_SIZE * 2]
        team1, team2 = self._balanced_partition(selected)

        if team1 and team2:
            # 验证匹配质量
            quality = self._evaluate_match_quality(team1, team2)
            if quality >= 0.6:  # 匹配质量阈值
                return MatchResult(team1=team1, team2=team2, quality=quality)

        return None

    def _balanced_partition(self, players):
        """
        将 10 个玩家分成两队，最小化 MMR 差异并平衡角色分布
        使用贪心 + 局部搜索（实际生产中可用匈牙利算法优化）
        """
        # 按 MMR 降序排列
        players.sort(key=lambda p: -p.mmr)

        team1 = []
        team2 = []

        for player in players:
            # 贪心：将当前玩家分到 MMR 总和较低的队伍
            sum1 = sum(p.mmr for p in team1)
            sum2 = sum(p.mmr for p in team2)

            # 角色平衡因素：如果某队缺少某角色类型，优先补充
            role_bonus1 = self._role_balance_score(team1, player)
            role_bonus2 = self._role_balance_score(team2, player)

            score1 = sum1 + player.mmr - role_bonus1 * 50
            score2 = sum2 + player.mmr - role_bonus2 * 50

            if score1 <= score2:
                team1.append(player)
            else:
                team2.append(player)

        return team1, team2

    def _role_balance_score(self, team, new_player):
        """计算加入新玩家后的角色平衡增益"""
        if not team:
            return 1.0

        role_counts = {}
        for p in team:
            role_counts[p.preferred_role] = role_counts.get(p.preferred_role, 0) + 1

        # 如果该角色类型在队里不足2个，有平衡增益
        current_count = role_counts.get(new_player.preferred_role, 0)
        if current_count < 2:
            return 1.0 - current_count * 0.3
        return 0.0

    def _evaluate_match_quality(self, team1, team2):
        """
        评估匹配质量 (0-1)，综合考虑：
        1. MMR 均衡度（两队平均 MMR 差异）
        2. 角色分布均衡度
        3. 组队分布（两队组队人数应接近）
        """
        # MMR 均衡度
        avg1 = sum(p.mmr for p in team1) / len(team1)
        avg2 = sum(p.mmr for p in team2) / len(team2)
        mmr_diff = abs(avg1 - avg2)
        mmr_score = max(0, 1.0 - mmr_diff / 500)  # 差500分以上为0

        # 角色均衡度
        role_score = self._role_distribution_score(team1, team2)

        # 组队均衡度
        party1 = sum(1 for p in team1 if p.party_id)
        party2 = sum(1 for p in team2 if p.party_id)
        party_diff = abs(party1 - party2)
        party_score = max(0, 1.0 - party_diff * 0.2)

        # 加权综合
        return mmr_score * 0.5 + role_score * 0.3 + party_score * 0.2

    def get_player_match_info(self, player_id):
        """获取玩家匹配用的完整信息"""
        cached = self.redis.hgetall(f"player:info:{player_id}")
        if cached:
            return PlayerMatchInfo(**cached)

        # 缓存未命中，从 MySQL 加载
        profile = self.db.query(
            "SELECT * FROM player_profile WHERE player_id = %s", player_id
        )
        info = PlayerMatchInfo(
            player_id=profile.player_id,
            mmr=profile.mmr,
            preferred_role=self._get_preferred_role(profile),
            recent_win_rate=profile.recent_10_win / 10.0,
            party_id=None
        )
        # 写入缓存，TTL 5分钟
        self.redis.hmset(f"player:info:{player_id}", info.__dict__, ex=300)
        return info

    def calculate_mmr_range(self, wait_seconds):
        """
        等待时间 → 允许的 MMR 差异范围
        
        等待 0-5秒：±100 MMR（严格同段位）
        等待 5-15秒：±200 MMR
        等待 15-30秒：±400 MMR（跨段位）
        等待 30秒+：±600 MMR（几乎任何段位）
        """
        if wait_seconds < 5:
            return 100
        elif wait_seconds < 15:
            return 200
        elif wait_seconds < 30:
            return 400
        else:
            return 600
```

**等待时间扩展的效果：**

| 等待时间 | MMR 范围 | 匹配质量 | 适用 |
|---------|---------|---------|------|
| 0-5秒 | ±100 | 极高 | 黄金以下（玩家多） |
| 5-15秒 | ±200 | 高 | 铂金 |
| 15-30秒 | ±400 | 中 | 钻石 |
| 30秒+ | ±600 | 低 | 大师/王者（玩家极少） |

**组队匹配：**

```python
class PartyMatchMaker:
    def enqueue_party(self, party):
        """组队加入匹配队列"""
        # 组队的 MMR 取队内最高值 + 加权
        # 防止高段位带低段位"炸鱼"
        max_mmr = max(p.mmr for p in party.members)
        avg_mmr = sum(p.mmr for p in party.members) / len(party.members)
        
        # 组队 MMR = 平均值 × 0.6 + 最大值 × 0.4
        # 偏向最高段位，因为高段位玩家的影响力更大
        party_mmr = avg_mmr * 0.6 + max_mmr * 0.4
        
        self.redis.zadd(
            f"match:queue:party:{len(party.members)}",
            {party.id: party_mmr}
        )
```

### 房间状态机：完整生命周期管理

游戏房间有严格的阶段性，每个状态有明确的转换条件和超时保护：

```
creating → loading → countdown → playing ⇄ paused → finished → settling → settled → closed
   │         │          │          │                    │
   │         │          │          │                    └→ 异常结束(有人断线超时)
   │         │          │          └→ 暂停(全部断线)
   │         │          └→ 超时(有人加载失败)→ 回到匹配队列
   │         └→ 超时(有人未确认)→ 回到匹配队列
   └→ 服务器创建失败→ 重试分配
```

```python
from enum import Enum
from datetime import datetime, timedelta

class RoomState(Enum):
    CREATING   = "creating"    # 服务器创建房间中
    LOADING    = "loading"     # 玩家加载资源中
    COUNTDOWN  = "countdown"   # 倒计时准备阶段
    PLAYING    = "playing"     # 游戏进行中
    PAUSED     = "paused"      # 游戏暂停（全部断线等）
    FINISHED   = "finished"    # 游戏结束（一方获胜/投降）
    SETTLING   = "settling"    # 结算中（写数据库、更新MMR）
    SETTLED    = "settled"     # 结算完成
    CLOSED     = "closed"      # 房间关闭，资源释放

class RoomStateMachine:
    """房间状态机 — 管理完整的房间生命周期"""

    # 各状态超时时间
    STATE_TIMEOUTS = {
        RoomState.CREATING:  timedelta(seconds=10),
        RoomState.LOADING:   timedelta(seconds=60),
        RoomState.COUNTDOWN: timedelta(seconds=10),
        RoomState.PAUSED:    timedelta(minutes=5),
        RoomState.SETTLING:  timedelta(seconds=15),
    }

    TRANSITIONS = {
        # (from_state, to_state): trigger_event
        (RoomState.CREATING,   RoomState.LOADING):    "all_players_assigned",
        (RoomState.LOADING,    RoomState.COUNTDOWN):  "all_players_loaded",
        (RoomState.COUNTDOWN,  RoomState.PLAYING):    "countdown_finished",
        (RoomState.PLAYING,    RoomState.PAUSED):     "all_players_disconnected",
        (RoomState.PAUSED,     RoomState.PLAYING):    "player_reconnected",
        (RoomState.PLAYING,    RoomState.FINISHED):   "game_result_determined",
        (RoomState.PAUSED,     RoomState.FINISHED):   "pause_timeout",
        (RoomState.FINISHED,   RoomState.SETTLING):   "start_settlement",
        (RoomState.SETTLING,   RoomState.SETTLED):    "settlement_complete",
        (RoomState.SETTLED,    RoomState.CLOSED):     "room_cleanup",
    }

    def __init__(self, room_id, redis, db):
        self.room_id = room_id
        self.redis = redis
        self.db = db
        self.state = RoomState.CREATING
        self.entered_at = datetime.now()

    def transition(self, event, context=None):
        """
        状态转换：校验合法性 → 执行转换 → 触发副作用

        参数:
            event: 触发事件名
            context: 附加上下文（如 game_result, disconnected_players 等）
        """
        # 1. 查找合法转换
        target_state = None
        for (from_s, to_s), trigger in self.TRANSITIONS.items():
            if from_s == self.state and trigger == event:
                target_state = to_s
                break

        if target_state is None:
            raise InvalidTransition(
                f"Room {self.room_id}: cannot transition from {self.state.value} on event '{event}'"
            )

        # 2. 执行前置校验
        self._validate_transition(target_state, context)

        # 3. 执行状态转换
        old_state = self.state
        self.state = target_state
        self.entered_at = datetime.now()

        # 4. 持久化状态变更
        self._persist_state(old_state, target_state, context)

        # 5. 触发副作用
        self._on_state_changed(target_state, context)

        # 6. 设置超时检查
        if target_state in self.STATE_TIMEOUTS:
            self._schedule_timeout_check(target_state)

        return True

    def _validate_transition(self, target_state, context):
        """转换前的业务校验"""
        if target_state == RoomState.LOADING:
            # 必须有完整的 10 个玩家
            if not context or len(context.get("players", [])) != 10:
                raise ValidationError("Cannot enter LOADING: need exactly 10 players")

        elif target_state == RoomState.COUNTDOWN:
            # 必须有足够的玩家加载完成
            loaded = context.get("loaded_count", 0) if context else 0
            if loaded < 10:
                raise ValidationError(
                    f"Cannot enter COUNTDOWN: only {loaded}/10 players loaded"
                )

        elif target_state == RoomState.PLAYING:
            # 从 PAUSED 恢复时，至少要有部分玩家在线
            if context and context.get("from_pause"):
                online = context.get("online_count", 0)
                if online < 6:  # 至少6人在线才能恢复
                    raise ValidationError("Cannot resume: too few players online")

    def _on_state_changed(self, new_state, context):
        """状态变更后的副作用处理"""
        if new_state == RoomState.LOADING:
            # 通知所有玩家开始加载
            for player in context["players"]:
                self._notify_player(player, "load_game", {
                    "room_id": self.room_id,
                    "map": context.get("map", "summoner_rift"),
                    "timeout_seconds": 60
                })

        elif new_state == RoomState.COUNTDOWN:
            # 开始 5 秒倒计时
            self._broadcast("countdown", {"seconds": 5})

        elif new_state == RoomState.PLAYING:
            # 记录游戏开始时间
            self.redis.set(f"room:start:{self.room_id}", datetime.now().isoformat())

        elif new_state == RoomState.PAUSED:
            # 暂停计时，5分钟后自动判负
            disconnected_team = context.get("disconnected_team")
            self._broadcast("game_paused", {
                "reason": "opponent_disconnected",
                "resume_countdown": 300  # 5分钟
            })

        elif new_state == RoomState.FINISHED:
            # 记录游戏结果，触发结算
            result = context.get("game_result")
            self.db.execute(
                "UPDATE game_room SET status='finished', winner=%s, "
                "end_time=NOW(), duration_sec=%s WHERE room_id=%s",
                (result.winner, result.duration_sec, self.room_id)
            )

        elif new_state == RoomState.SETTLING:
            # 异步执行结算（MMR更新、记录写入、奖励发放）
            self._async_settle(context)

        elif new_state == RoomState.CLOSED:
            # 释放服务器资源
            self.redis.zincrby(
                "game:servers:available", 1, context.get("server_id")
            )
            # 清理房间相关 Redis 键
            self._cleanup_redis_keys()

    def _schedule_timeout_check(self, state):
        """为有超时的状态设置延迟检查"""
        timeout = self.STATE_TIMEOUTS[state]
        self.redis.setex(
            f"room:timeout:{self.room_id}",
            int(timeout.total_seconds()),
            state.value
        )

    def check_timeout(self):
        """检查当前状态是否超时，超时则执行超时处理"""
        if self.state not in self.STATE_TIMEOUTS:
            return

        elapsed = (datetime.now() - self.entered_at).total_seconds()
        timeout = self.STATE_TIMEOUTS[self.state].total_seconds()

        if elapsed > timeout:
            self._handle_timeout()

    def _handle_timeout(self):
        """各状态超时的处理逻辑"""
        if self.state == RoomState.CREATING:
            # 服务器创建房间超时 → 重试或重新分配
            self._retry_create_room()

        elif self.state == RoomState.LOADING:
            # 加载超时 → 将未加载的玩家踢出，剩余玩家重新匹配
            loaded_players = self._get_loaded_players()
            unloaded_players = self._get_unloaded_players()

            # 未加载的玩家可能是挂机/网络差，给予惩罚
            for player in unloaded_players:
                self._apply_afk_penalty(player)

            # 已加载的玩家重新入队（优先匹配）
            for player in loaded_players:
                self.match_queue.enqueue_with_priority(player)

            self.state = RoomState.CLOSED

        elif self.state == RoomState.COUNTDOWN:
            # 倒计时阶段不太可能超时，如果超时直接开始
            self.transition("countdown_finished")

        elif self.state == RoomState.PAUSED:
            # 暂停超时（5分钟）→ 断线方判负
            self.transition("pause_timeout", context={
                "game_result": GameResult(
                    winner=self._get_opposite_team(self._disconnected_team),
                    reason="disconnect_timeout",
                    duration_sec=self._get_elapsed_game_time()
                )
            })

        elif self.state == RoomState.SETTLING:
            # 结算超时 → 强制完成（不阻塞房间释放）
            self.state = RoomState.SETTLED

    def _async_settle(self, context):
        """异步结算流程"""
        result = context.get("game_result")

        # 1. 批量更新 MMR
        mmr_changes = self.mmr_calculator.batch_update(result)

        # 2. 写入对局记录
        for player, change in mmr_changes:
            stats = result.get_player_stats(player.id)
            self.db.execute(
                "INSERT INTO match_history (room_id, player_id, team, is_winner, "
                "mmr_before, mmr_after, mmr_change, kills, deaths, assists, "
                "damage_dealt, gold_earned, play_duration, champion_used) "
                "VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)",
                (self.room_id, player.id, player.team,
                 1 if change > 0 else 0,
                 player.mmr, player.mmr + change, change,
                 stats.kills, stats.deaths, stats.assists,
                 stats.damage_dealt, stats.gold_earned,
                 result.duration_sec, stats.champion)
            )

        # 3. 更新段位（MMR 变化可能触发升/降段）
        for player, change in mmr_changes:
            self._update_tier(player, player.mmr + change)

        # 4. 发放奖励（经验/金币/赛季积分）
        self._distribute_rewards(result)

        # 5. 标记结算完成
        self.transition("settlement_complete", context={
            "mmr_changes": mmr_changes
        })
```

### 房间分配：游戏服务器管理

```python
class GameServerManager:
    """管理游戏服务器的房间分配"""
    
    def allocate_room(self, match_result):
        """为匹配成功的玩家分配游戏房间"""
        # 1. 找有空位的游戏服务器
        server = self.find_available_server()
        
        if server is None:
            # 所有服务器都满 → 扩容
            server = self.scale_up()
        
        # 2. 在服务器上创建房间
        room = self.create_room(server, match_result)
        
        # 3. 通知玩家连接到游戏服务器
        for player in match_result.all_players:
            self.notify_player(player.id, {
                "type": "match_found",
                "room_id": room.id,
                "server_endpoint": server.endpoint,
                "team": "team1" if player in match_result.team1 else "team2"
            })
        
        return room

    def find_available_server(self):
        """找有空位的游戏服务器"""
        # 从 Redis 获取所有游戏服务器的状态
        servers = self.redis.zrangebyscore(
            "game:servers:available", 1, "+inf"
        )
        
        if not servers:
            return None
        
        # 选择负载最低的服务器（避免单服务器过载）
        return servers[0]  # Sorted Set 按剩余容量排序

    def create_room(self, server, match_result):
        """在游戏服务器上创建房间"""
        response = self.rpc_call(server, "create_room", {
            "room_id": uuid4(),
            "team1": [p.id for p in match_result.team1],
            "team2": [p.id for p in match_result.team2],
            "game_mode": "ranked",
            "map": "summoner_rift"
        })
        
        # 更新服务器剩余容量
        self.redis.zincrby("game:servers:available", -1, server.id)
        
        return response
```

### 有状态服务的弹性伸缩

**关键约束：游戏房间进行中不能迁移。**

弹性伸缩的核心难点在于：游戏服务器是有状态的，不能像无状态 Web 服务那样随时增减。需要精细化的容量预测和渐进式伸缩。

#### 容量预测模型

```python
import numpy as np
from collections import defaultdict

class CapacityPredictor:
    """
    基于历史数据的容量预测模型
    使用时间序列预测 + 在线量实时修正
    """
    def __init__(self, redis, db):
        self.redis = redis
        self.db = db

    def predict_next_hour_load(self, region):
        """
        预测未来 1 小时的房间需求量

        策略：加权融合三种预测
        1. 历史同期模式（同一时段的历史负载）
        2. 当前趋势（最近 30 分钟的变化率）
        3. 事件修正（节假日/赛事/版本更新）
        """
        # 历史同期：取过去 4 周同一时段的平均值
        historical = self._get_historical_pattern(region)

        # 当前趋势：最近 30 分钟的线性外推
        recent_data = self._get_recent_trend(region, minutes=30)
        trend = self._extrapolate(recent_data, horizon_minutes=60)

        # 事件修正系数
        event_factor = self._get_event_factor()

        # 加权融合：历史 0.5 + 趋势 0.3 + 事件 0.2
        predicted = (
            historical * 0.5 +
            trend * 0.3 +
            historical * event_factor * 0.2
        )

        return int(predicted * 1.15)  # 加 15% 安全余量

    def _get_historical_pattern(self, region):
        """获取历史同期模式"""
        now = datetime.now()
        hourly_avg = []

        for week in range(1, 5):  # 过去 4 周
            same_time = now - timedelta(weeks=week)
            count = self.db.query_one(
                "SELECT COUNT(*) FROM game_room "
                "WHERE region = %s AND create_time BETWEEN %s AND %s",
                (region, same_time, same_time + timedelta(hours=1))
            )
            hourly_avg.append(count)

        return np.mean(hourly_avg) if hourly_avg else 0

    def _extrapolate(self, recent_data, horizon_minutes):
        """线性外推"""
        if len(recent_data) < 2:
            return recent_data[0] if recent_data else 0

        x = np.arange(len(recent_data))
        coeffs = np.polyfit(x, recent_data, 1)  # 一阶线性拟合
        future_x = np.arange(len(recent_data), len(recent_data) + horizon_minutes // 5)
        return max(0, np.polyval(coeffs, future_x[-1]))

    def _get_event_factor(self):
        """事件修正系数"""
        if self._is_holiday():
            return 1.5  # 节假日负载增加 50%
        if self._is_weekend():
            return 1.2  # 周末增加 20%
        if self._is_patch_day():
            return 2.0  # 版本更新日翻倍
        return 1.0
```

#### 弹性伸缩控制器

```python
class GameServerScaler:
    """
    弹性伸缩控制器

    三级伸缩策略：
    - 快速层（秒级）：应对突发匹配高峰，扩容预启动的备用机
    - 常规层（分钟级）：根据预测模型主动调整容量
    - 节能层（小时级）：低谷期缩减到最小可用规模
    """

    # 伸缩参数
    SCALE_UP_THRESHOLD = 0.75       # 利用率 > 75% 触发扩容
    SCALE_DOWN_THRESHOLD = 0.35     # 利用率 < 35% 触发缩容
    EMERGENCY_THRESHOLD = 0.95      # 利用率 > 95% 紧急扩容
    MIN_SERVERS_PER_REGION = 3      # 每区域最少保留服务器数
    MAX_SERVERS_PER_REGION = 200    # 每区域最大服务器数
    ROOM_CAPACITY_PER_SERVER = 500  # 单服务器房间容量

    def __init__(self, redis, db, cloud_client, predictor):
        self.redis = redis
        self.db = db
        self.cloud = cloud_client
        self.predictor = predictor

    def check_and_scale(self):
        """定期检查是否需要扩缩容（每 30 秒执行一次）"""
        for region in self.REGIONS:
            self._scale_region(region)

    def _scale_region(self, region):
        """单个区域的伸缩决策"""
        metrics = self._collect_metrics(region)

        # 紧急扩容：利用率 > 95% 或有匹配等待
        if metrics.utilization > self.EMERGENCY_THRESHOLD or metrics.pending_matches > 0:
            self._emergency_scale_up(region, metrics)
            return

        # 常规扩容
        if metrics.utilization > self.SCALE_UP_THRESHOLD:
            count = self._calculate_scale_up_count(metrics)
            self._scale_up(region, count, priority="normal")
            return

        # 预测性扩容：当前利用率不高，但预测即将到来高峰
        predicted_load = self.predictor.predict_next_hour_load(region)
        predicted_util = predicted_load / (metrics.total_servers * self.ROOM_CAPACITY_PER_SERVER)
        if predicted_util > self.SCALE_UP_THRESHOLD and metrics.utilization > 0.5:
            # 提前 10 分钟扩容（服务器启动需要 3-5 分钟）
            count = self._calculate_scale_up_count_from_predicted(predicted_util, metrics)
            self._scale_up(region, count, priority="predictive")
            return

        # 缩容
        if metrics.utilization < self.SCALE_DOWN_THRESHOLD:
            self._scale_down(region, metrics)

    def _collect_metrics(self, region):
        """收集区域伸缩指标"""
        # 从 Redis 获取实时数据
        servers = self.redis.hgetall(f"region:servers:{region}")
        active_servers = [s for s in servers if s["status"] in ("available", "draining")]

        total_capacity = len(active_servers) * self.ROOM_CAPACITY_PER_SERVER
        active_rooms = sum(s["active_rooms"] for s in active_servers)
        pending = self.redis.llen(f"match:pending:{region}")

        # 计算近期趋势
        recent_5min = self.redis.get(f"metrics:rooms_created:{region}:5m") or 0
        recent_15min = self.redis.get(f"metrics:rooms_created:{region}:15m") or 0
        trend = recent_5min / max(recent_15min / 3, 1)  # 5分钟 vs 15分钟/3

        return ScaleMetrics(
            region=region,
            total_servers=len(active_servers),
            total_capacity=total_capacity,
            active_rooms=active_rooms,
            utilization=active_rooms / max(total_capacity, 1),
            pending_matches=pending,
            trend_ratio=trend  # >1 表示负载上升，<1 表示下降
        )

    def _scale_up(self, region, count, priority="normal"):
        """扩容：启动新的游戏服务器"""
        current = self._get_server_count(region)
        if current + count > self.MAX_SERVERS_PER_REGION:
            count = self.MAX_SERVERS_PER_REGION - current
            if count <= 0:
                self._alert(f"Region {region} hit max server limit!")
                return

        for i in range(count):
            if priority == "emergency":
                # 紧急扩容：使用预启动的备用机（3秒就绪）
                server = self._activate_standby(region)
            else:
                # 常规扩容：启动新实例（3-5分钟就绪）
                server = self.cloud.start_instance(
                    type="game-server",
                    image="game-server:v2.3",
                    region=region,
                    instance_type="c5.2xlarge",  # 8核16GB
                    wait_until_ready=True,
                    startup_timeout=300  # 5分钟超时
                )

            # 注册到可用服务器池
            self.redis.zadd(
                "game:servers:available",
                {server.id: self.ROOM_CAPACITY_PER_SERVER}
            )
            self.redis.hset(f"server:info:{server.id}", mapping={
                "region": region,
                "status": "available",
                "max_rooms": self.ROOM_CAPACITY_PER_SERVER,
                "started_at": datetime.now().isoformat()
            })

        self._log_scale_event(region, "scale_up", count, priority)

    def _emergency_scale_up(self, region, metrics):
        """紧急扩容"""
        # 1. 立即激活所有备用机
        standby_count = self._get_standby_count(region)
        self._activate_all_standby(region)

        # 2. 追加启动新实例
        additional = max(3, int(metrics.pending_matches / self.ROOM_CAPACITY_PER_SERVER) + 2)
        self._scale_up(region, additional, priority="emergency")

        # 3. 如果仍有等待，降级匹配质量（放宽匹配范围）以加速匹配
        if metrics.pending_matches > 50:
            self._temporarily_widen_match_range(region)

    def _scale_down(self, region, metrics):
        """
        缩容：选择空闲或接近空闲的服务器下线
        渐进式缩容，避免过快缩减导致二次扩容
        """
        servers = self._get_servers_sorted_by_load(region)

        # 计算需要缩容的数量
        target_utilization = 0.55  # 缩容后目标利用率
        target_capacity = int(metrics.active_rooms / target_utilization)
        current_capacity = metrics.total_servers * self.ROOM_CAPACITY_PER_SERVER
        excess_servers = int((current_capacity - target_capacity) / self.ROOM_CAPACITY_PER_SERVER)

        # 限制每次最多缩容 10% 的服务器，避免缩容过快
        max_drain_per_round = max(1, metrics.total_servers // 10)
        drain_count = min(excess_servers, max_drain_per_round)

        drained = 0
        for server in servers:
            if drained >= drain_count:
                break

            active_rooms = self._get_active_rooms(server.id)

            if active_rooms == 0:
                # 完全空闲 → 可以安全下线
                self.drain_and_terminate(server)
                drained += 1
            elif active_rooms <= server.max_rooms * 0.1:
                # 接近空闲 → 标记为"不接新房"，等房间结束后下线
                self.mark_draining(server)
                drained += 1

        # 确保不缩容到最小保留数以下
        remaining = metrics.total_servers - drained
        if remaining < self.MIN_SERVERS_PER_REGION:
            self._log(f"Region {region}: skipping further scale-down, "
                      f"at minimum {self.MIN_SERVERS_PER_REGION} servers")

    def mark_draining(self, server):
        """标记服务器为排空状态"""
        self.redis.set(f"server:draining:{server.id}", "1", ex=3600)

        # 从可用池移除（不再分配新房间）
        self.redis.zrem("game:servers:available", server.id)

        # 更新状态
        self.redis.hset(f"server:info:{server.id}", "status", "draining")

        # 设置定时检查：房间全部结束后下线
        self.schedule_check(server.id, interval_seconds=60)

    def drain_and_terminate(self, server):
        """安全下线服务器"""
        # 二次确认无活跃房间
        active = self._get_active_rooms(server.id)
        if active > 0:
            self._log(f"Server {server.id} still has {active} rooms, aborting terminate")
            return

        # 通知云平台终止实例
        self.cloud.terminate_instance(server.id)

        # 清理 Redis 数据
        self.redis.delete(
            f"server:info:{server.id}",
            f"server:draining:{server.id}",
        )
        self.redis.zrem("game:servers:available", server.id)

        self._log(f"Server {server.id} terminated successfully")

    # ===== 备用机管理 =====

    def _maintain_standby_pool(self, region):
        """
        维护备用机池：始终保留一定数量的预启动服务器
        备用机处于"休眠"状态，激活只需 3 秒
        """
        target_standby = max(2, self._get_server_count(region) // 20)  # 5% 的备用比例
        current_standby = self._get_standby_count(region)

        if current_standby < target_standby:
            for _ in range(target_standby - current_standby):
                server = self.cloud.start_instance(
                    type="game-server-standby",
                    image="game-server:v2.3",
                    region=region,
                    instance_type="c5.2xlarge",
                    standby_mode=True  # 低功耗模式
                )
                self.redis.sadd(f"region:standby:{region}", server.id)

    def _activate_standby(self, region):
        """激活一台备用机"""
        server_id = self.redis.spop(f"region:standby:{region}")
        if not server_id:
            return None

        # 唤醒备用机（3秒就绪）
        self.cloud.wake_instance(server_id)
        return Server(id=server_id, region=region)
```

**伸缩策略总结：**

| 策略 | 触发条件 | 响应时间 | 适用场景 |
|------|---------|---------|---------|
| 紧急扩容 | 利用率 > 95% 或匹配积压 | 3 秒（激活备用机） | 突发高峰 |
| 常规扩容 | 利用率 > 75% | 3-5 分钟（启动新实例） | 日常高峰 |
| 预测扩容 | 预测利用率 > 75% | 提前 10 分钟 | 可预期高峰 |
| 渐进缩容 | 利用率 < 35% | 5-30 分钟（等房间结束） | 低谷期 |
| 备用机维护 | 备用机不足 5% | 后台持续 | 全时段 |

### 匹配超时处理

```python
class MatchTimeoutHandler:
    def on_match_timeout(self, player_id):
        """玩家匹配超过 60 秒"""
        # 1. 从队列中移除
        self.match_queue.dequeue(player_id)
        
        # 2. 通知玩家
        self.notify_player(player_id, {
            "type": "match_timeout",
            "message": "匹配时间过长，是否继续等待？",
            "options": ["继续等待", "取消匹配"]
        })
        
        # 3. 如果玩家选择继续 → 重新入队（等待时间重置，但 MMR 范围保持扩展后的值）
        # 4. 如果玩家选择取消 → 退出
```

### 异常场景处理

#### 场景 1：匹配确认超时 — 玩家未点击"确认"

匹配成功后，需要所有玩家确认才能进入游戏。有人不确认（AFK/切后台）时需要补人：

```python
class MatchConfirmationHandler:
    """
    匹配确认流程：
    匹配成功 → 10人全部收到确认弹窗 → 15秒内确认 → 进入加载
    有人未确认 → 将未确认者踢出，从匹配队列补人
    """

    CONFIRM_TIMEOUT = 15  # 秒

    def on_match_found(self, match_result):
        """匹配成功，发起确认流程"""
        match_id = uuid4()

        # 写入确认集合
        for player in match_result.all_players:
            self.redis.sadd(f"match:pending_confirm:{match_id}", player.id)

        # 设置确认超时
        self.redis.setex(
            f"match:confirm_deadline:{match_id}",
            self.CONFIRM_TIMEOUT,
            match_id
        )

        # 通知所有玩家
        for player in match_result.all_players:
            self.notify_player(player.id, {
                "type": "match_confirm",
                "match_id": match_id,
                "timeout_seconds": self.CONFIRM_TIMEOUT,
                # 不暴露队友/对手信息，确认后才显示
            })

        # 启动超时检查
        self.schedule_timeout_check(match_id, self.CONFIRM_TIMEOUT)

    def on_player_confirm(self, match_id, player_id):
        """玩家确认"""
        # 标记该玩家已确认
        self.redis.smove(
            f"match:pending_confirm:{match_id}",
            f"match:confirmed:{match_id}",
            player_id
        )

        # 检查是否全部确认
        pending_count = self.redis.scard(f"match:pending_confirm:{match_id}")
        if pending_count == 0:
            self._all_confirmed(match_id)
        else:
            # 通知已确认的玩家等待其他人
            self.notify_player(player_id, {
                "type": "confirm_received",
                "waiting_for": pending_count
            })

    def on_confirm_timeout(self, match_id):
        """确认超时处理"""
        # 获取未确认的玩家
        unconfirmed = self.redis.smembers(f"match:pending_confirm:{match_id}")
        confirmed = self.redis.smembers(f"match:confirmed:{match_id}")

        # 未确认玩家：给予惩罚（增加匹配等待时间）并移出队列
        for player_id in unconfirmed:
            self._apply_confirm_timeout_penalty(player_id)
            self.notify_player(player_id, {
                "type": "confirm_timeout_penalty",
                "message": "未及时确认匹配，下次匹配需额外等待 30 秒"
            })

        # 已确认玩家：尝试补人
        if len(confirmed) >= self.TEAM_SIZE * 2 - 2:
            # 缺人不多，尝试从匹配队列快速补人
            self._refill_match(match_id, confirmed, unconfirmed)
        else:
            # 缺人太多，已确认玩家重新入队（优先匹配）
            for player_id in confirmed:
                self.match_queue.enqueue_with_priority(player_id)
                self.notify_player(player_id, {
                    "type": "match_cancelled",
                    "message": "部分玩家未确认，已为您重新匹配"
                })

        # 清理 Redis 数据
        self._cleanup_match_confirm(match_id)

    def _refill_match(self, match_id, confirmed_ids, unconfirmed_ids):
        """从匹配队列中快速补人"""
        need_count = len(unconfirmed_ids)

        # 从匹配队列中取等待时间最长的玩家（他们最愿意快速进入）
        refill_players = self.match_queue.pop_oldest_waiters(
            count=need_count,
            mmr_range=300  # 补人时放宽匹配范围
        )

        if len(refill_players) < need_count:
            # 补不够人，回退到全部重新匹配
            for player_id in confirmed_ids:
                self.match_queue.enqueue_with_priority(player_id)
            return

        # 补人成功，合并确认列表
        for player in refill_players:
            self.redis.sadd(f"match:confirmed:{match_id}", player.id)
            # 新补入的玩家不需要再确认，直接进入
            self.notify_player(player.id, {
                "type": "quick_match_found",
                "match_id": match_id
            })

        self._all_confirmed(match_id)
```

#### 场景 2：游戏服务器崩溃

游戏服务器崩溃是最严重的异常——10 个玩家的对局被迫中断：

```python
class ServerCrashHandler:
    """
    游戏服务器崩溃处理

    策略：
    1. 检测崩溃（心跳超时）
    2. 判断对局是否可恢复
    3. 可恢复 → 迁移到备用服务器继续
    4. 不可恢复 → 判定无效对局，不影响 MMR
    """

    def on_server_heartbeat_lost(self, server_id):
        """服务器心跳丢失"""
        # 1. 确认是真的崩溃（可能是网络抖动）
        if not self._verify_server_down(server_id):
            return  # 网络抖动，忽略

        # 2. 获取该服务器上所有活跃房间
        active_rooms = self._get_active_rooms_on_server(server_id)

        # 3. 逐个处理房间
        for room in active_rooms:
            self._handle_room_on_crashed_server(room)

        # 4. 将服务器标记为终止
        self.redis.hset(f"server:info:{server_id}", "status", "terminated")
        self.redis.zrem("game:servers:available", server_id)

        # 5. 触发扩容补充
        self.scaler.emergency_scale_up(region=room.region)

    def _handle_room_on_crashed_server(self, room):
        """处理崩溃服务器上的房间"""
        elapsed = self._get_game_elapsed_time(room.room_id)

        if elapsed < 180:  # 游戏开始不到 3 分钟
            # 对局刚开始，判为无效对局
            self._void_match(room, reason="server_crash_early_game")

        elif elapsed < 600:  # 3-10 分钟
            # 中期崩溃，检查是否有明显优势方
            # 如果一方有巨大优势（如经济差 5000+），则判定优势方获胜
            game_state = self._try_recover_last_state(room.room_id)
            if game_state and self._has_clear_winner(game_state):
                self._award_win_to_leading_team(room, game_state)
            else:
                self._void_match(room, reason="server_crash_mid_game")

        else:  # 10 分钟以上
            # 后期崩溃，尽力恢复
            game_state = self._try_recover_last_state(room.room_id)
            if game_state:
                # 尝试迁移到新服务器恢复
                if self._migrate_room(room, game_state):
                    return
            # 恢复失败，按优势判定或无效
            self._void_match(room, reason="server_crash_late_game")

    def _void_match(self, room, reason):
        """
        判定对局无效
        - 不扣 MMR（双方 MMR 不变）
        - 记录异常原因
        - 发放补偿（如额外金币/经验）
        """
        self.db.execute(
            "UPDATE game_room SET status='voided', end_time=NOW() "
            "WHERE room_id=%s", (room.room_id,)
        )

        for player in room.all_players:
            # MMR 不变，但记录异常
            self.db.execute(
                "INSERT INTO match_history (room_id, player_id, team, is_winner, "
                "mmr_before, mmr_after, mmr_change, play_duration) "
                "VALUES (%s,%s,%s,0,%s,%s,0,%s)",
                (room.room_id, player.id, player.team,
                 player.mmr, player.mmr, 0)
            )

            # 补偿通知
            self.notify_player(player.id, {
                "type": "match_voided",
                "reason": reason,
                "compensation": {"gold": 200, "exp": 100}
            })

    def _migrate_room(self, room, game_state):
        """
        尝试将房间迁移到新服务器

        前提：
        1. 有定期保存的游戏状态快照
        2. 新服务器可以加载快照
        3. 玩家能重新连接
        """
        new_server = self.server_manager.find_available_server()
        if not new_server:
            return False

        try:
            # 在新服务器上恢复房间
            self.rpc_call(new_server, "restore_room", {
                "room_id": room.room_id,
                "state_snapshot": game_state,
                "pause_on_restore": True  # 恢复后暂停，等玩家重连
            })

            # 通知所有玩家重新连接
            for player in room.all_players:
                self.notify_player(player.id, {
                    "type": "reconnect_required",
                    "room_id": room.room_id,
                    "new_server_endpoint": new_server.endpoint,
                    "timeout_seconds": 120  # 2 分钟重连窗口
                })

            return True

        except Exception as e:
            self._log(f"Room migration failed: {e}")
            return False
```

#### 场景 3：玩家中途断线与重连

```python
class PlayerDisconnectHandler:
    """
    玩家断线处理

    断线 ≠ 退出，给予 3 分钟重连窗口
    超时未重连 → AI 托管（不让队友 4 打 5）
    游戏结束后 → 按实际表现结算，但增加惩罚标记
    """

    RECONNECT_TIMEOUT = 180  # 3 分钟

    def on_player_disconnect(self, room_id, player_id):
        """玩家断开连接"""
        # 1. 记录断线时间
        self.redis.setex(
            f"room:disconnect:{room_id}:{player_id}",
            self.RECONNECT_TIMEOUT,
            datetime.now().isoformat()
        )

        # 2. 通知队友
        self._broadcast_to_team(room_id, player_id, "teammate_disconnected", {
            "player_id": player_id,
            "reconnect_countdown": self.RECONNECT_TIMEOUT
        })

        # 3. 启动 AI 托管（断线玩家的角色由 AI 接管）
        self.rpc_call(self._get_server(room_id), "enable_ai_control", {
            "room_id": room_id,
            "player_id": player_id,
            "ai_level": "medium"  # 中等 AI，不碾压但也不送
        })

        # 4. 设置重连超时检查
        self.schedule_timeout_check(
            room_id, player_id, self.RECONNECT_TIMEOUT
        )

    def on_player_reconnect(self, room_id, player_id):
        """玩家重新连接"""
        # 1. 清除断线标记
        self.redis.delete(f"room:disconnect:{room_id}:{player_id}")

        # 2. 关闭 AI 托管
        self.rpc_call(self._get_server(room_id), "disable_ai_control", {
            "room_id": room_id,
            "player_id": player_id
        })

        # 3. 同步游戏状态给重连玩家
        game_state = self.rpc_call(self._get_server(room_id), "get_game_state", {
            "room_id": room_id
        })
        self.send_to_player(player_id, "game_state_sync", game_state)

        # 4. 通知队友
        self._broadcast_to_team(room_id, player_id, "teammate_reconnected", {
            "player_id": player_id
        })

    def on_reconnect_timeout(self, room_id, player_id):
        """重连超时：玩家 3 分钟未重连"""
        # AI 继续托管直到游戏结束
        # 在结算时标记该玩家为"挂机"
        self.redis.set(f"room:afk:{room_id}:{player_id}", "1", ex=3600)

        # 通知队友
        self._broadcast_to_team(room_id, player_id, "teammate_afk", {
            "player_id": player_id,
            "message": "队友已离线，AI 将继续操作"
        })

    def on_game_end_with_afk(self, room_id, afk_player_id):
        """游戏结束，处理挂机玩家"""
        # 1. 无论胜负，挂机玩家都扣除更多 MMR
        is_winner = self._is_player_winner(room_id, afk_player_id)
        if is_winner:
            # 赢了但挂机 → 扣 MMR（不配赢）
            mmr_change = -30  # 固定扣 30
        else:
            # 输了且挂机 → 双倍扣除
            mmr_change = self._calculate_normal_loss(room_id, afk_player_id) * 2

        # 2. 增加惩罚标记（多次挂机加重惩罚）
        afk_count = self._get_recent_afk_count(afk_player_id)
        if afk_count >= 3:
            # 禁赛 30 分钟
            self._ban_player(afk_player_id, duration_minutes=30)
        elif afk_count >= 1:
            # 下次匹配额外等待 60 秒
            self._add_match_penalty(afk_player_id, extra_wait=60)
```

#### 场景 4：匹配队列雪崩 — 大量玩家同时入队

版本更新后或赛事开始时，可能瞬间涌入大量匹配请求：

```python
class MatchQueueThrottler:
    """
    匹配队列流量控制

    防止大量玩家同时入队导致：
    1. Redis Sorted Set 过大，ZRANGE 性能下降
    2. 匹配计算量暴增
    3. 服务器瞬间被打满
    """

    MAX_QUEUE_SIZE_PER_TIER = 50000  # 每个段位桶最大容量
    ADMISSION_RATE = 1000  # 每秒最大入队数

    def enqueue_with_throttle(self, player):
        """带流量控制的入队操作"""
        tier = self.match_queue.get_tier(player.mmr)
        current_size = self.redis.zcard(f"match:queue:{tier}")

        # 1. 队列容量检查
        if current_size >= self.MAX_QUEUE_SIZE_PER_TIER:
            self.notify_player(player.id, {
                "type": "queue_full",
                "message": "当前匹配人数较多，请稍后再试",
                "retry_after_seconds": 30
            })
            return False

        # 2. 令牌桶限流
        if not self.rate_limiter.try_acquire("match:enqueue", self.ADMISSION_RATE):
            # 排队等待入队
            self.redis.rpush("match:enqueue:waiting", player.id)
            return True

        # 3. 检查服务器容量
        total_available = self.redis.zcard("game:servers:available")
        if total_available == 0:
            # 没有可用服务器，暂缓入队
            self.redis.rpush("match:enqueue:waiting", player.id)
            self._trigger_emergency_scale()
            return True

        # 4. 正常入队
        self.match_queue.enqueue(player)
        return True
```

### 防作弊：匹配信息隔离

```python
class MatchInfoIsolation:
    """匹配过程对玩家不可见，防止利用匹配信息作弊"""
    
    def get_match_status(self, player_id):
        """玩家查询匹配状态"""
        # 只返回"匹配中"，不返回队列中其他玩家信息
        is_matching = self.redis.exists(f"match:wait:{player_id}")
        wait_time = self.redis.get(f"match:wait:{player_id}")
        
        return {
            "status": "matching" if is_matching else "idle",
            "wait_seconds": (now() - parse(wait_time)).total_seconds() if is_matching else 0,
            # 不返回：队列人数、预计等待时间、匹配范围等
        }
```

## 性能与成本分析

### 吞吐量需求分析

基于 1000 万日活、300 万峰值在线的业务数据：

| 指标 | 计算 | 数值 |
|------|------|------|
| 峰值匹配 QPS | 300万在线 / (15分钟平均对局时长 × 60) | ~3.3 万/秒 |
| 峰值房间创建 | 3.3万 / 10人每局 | ~3300 局/秒 |
| 并发活跃房间 | 300万在线 / 10人每局 | ~30 万局 |
| 所需游戏服务器 | 30万局 / 500局每台 | ~600 台 |
| 峰值带宽 | 30万局 × 10人 × 20KB/s（游戏同步） | ~60 GB/s |

### 匹配系统性能瓶颈与优化

```python
class MatchPerformanceOptimizer:
    """
    匹配系统的性能优化

    核心问题：每 2 秒遍历所有段位桶寻找匹配 → O(tiers × players)
    当在线玩家多时，每次匹配计算耗时可能超过 2 秒
    """

    # 优化 1：增量匹配（不遍历整个队列）
    def incremental_match(self, tier):
        """
        增量匹配：只看队头和队尾，不做全量遍历

        Redis Sorted Set 按 MMR 排序 → 队列天然有序
        只需检查队头附近的玩家是否可组成匹配
        """
        queue_key = f"match:queue:{tier}"

        # 取队头附近 N 个玩家（最可能匹配成功的一批）
        candidates = self.redis.zrange(queue_key, 0, self.TEAM_SIZE * 4 - 1, withscores=True)

        if len(candidates) < self.TEAM_SIZE * 2:
            return None

        # 在这批候选人中寻找最佳匹配
        # 时间复杂度：O(N) 而非 O(total_queue_size)
        return self._match_from_candidates(candidates)

    # 优化 2：段位桶合并（低峰期合并相邻桶减少遍历次数）
    def get_effective_tier(self, tier):
        """低峰期合并相邻段位桶"""
        online_count = self.redis.zcard(f"match:queue:{tier}")

        if online_count < self.TEAM_SIZE * 2:
            # 当前桶人不够，合并到上一级桶
            return self._get_merged_tier(tier)
        return tier

    # 优化 3：匹配结果缓存
    def cache_match_result(self, match_result):
        """缓存匹配结果，避免重复计算"""
        self.redis.setex(
            f"match:result:{match_result.match_id}",
            30,  # 30秒有效
            json.dumps(match_result.to_dict())
        )
```

**性能指标对比：**

| 优化措施 | 优化前 | 优化后 | 提升 |
|---------|--------|--------|------|
| 匹配计算复杂度 | O(total_players) | O(20) 固定 | 1000x+ |
| 单次匹配耗时（P99） | 800ms | 15ms | 53x |
| Redis ZRANGE 调用 | 全量扫描 | 增量取 20 个 | 250x |
| 段位桶遍历次数 | 7次（全段位） | 2-3次（合并后） | 2.3x |
| 匹配服务 CPU | 80% | 15% | 5.3x |

### 成本模型

```python
class CostEstimator:
    """
    游戏平台成本估算

    主要成本项：
    1. 游戏服务器（最大成本）
    2. 匹配服务集群
    3. Redis 集群
    4. MySQL 存储
    5. 网络带宽
    """

    # AWS 参考价格（按需实例，us-east-1）
    INSTANCE_PRICES = {
        "c5.2xlarge": 0.34,    # 8核16GB，游戏服务器
        "c5.xlarge": 0.17,     # 4核8GB，匹配服务
        "r5.xlarge": 0.252,    # 8核32GB，Redis
        "db.r5.xlarge": 0.265, # RDS
    }

    def estimate_monthly_cost(self):
        """月度成本估算"""

        # 1. 游戏服务器
        # 峰值 600 台，平均利用 40% → 日均 240 台
        # 使用竞价实例可降低 60%
        avg_game_servers = 240
        game_server_cost = avg_game_servers * 24 * 30 * self.INSTANCE_PRICES["c5.2xlarge"]
        game_server_spot = game_server_cost * 0.4  # 竞价实例

        # 2. 匹配服务集群
        # 3 台 c5.xlarge + 2 台备用
        match_servers = 5
        match_cost = match_servers * 24 * 30 * self.INSTANCE_PRICES["c5.xlarge"]

        # 3. Redis 集群
        # 6 节点集群（3主3从）
        redis_nodes = 6
        redis_cost = redis_nodes * 24 * 30 * self.INSTANCE_PRICES["r5.xlarge"]

        # 4. MySQL RDS
        # 1 主 + 1 从
        mysql_cost = 2 * 24 * 30 * self.INSTANCE_PRICES["db.r5.xlarge"]

        # 5. 网络带宽
        # 峰值 60GB/s，日均 ~25GB/s
        # AWS 传输费用 $0.09/GB
        bandwidth_gb_month = 25 * 3600 * 24 * 30 / 1000  # TB级
        bandwidth_cost = bandwidth_gb_month * 1000 * 0.09  # 转回 GB 计算

        total = game_server_spot + match_cost + redis_cost + mysql_cost + bandwidth_cost

        return {
            "游戏服务器（竞价）": f"${game_server_spot:,.0f}",
            "匹配服务": f"${match_cost:,.0f}",
            "Redis 集群": f"${redis_cost:,.0f}",
            "MySQL RDS": f"${mysql_cost:,.0f}",
            "网络带宽": f"${bandwidth_cost:,.0f}",
            "总计": f"${total:,.0f}",
            "单玩家月成本": f"${total / 10_000_000:.4f}"
        }
```

**成本结构（估算）：**

| 成本项 | 月费用（估算） | 占比 | 优化建议 |
|--------|-------------|------|---------|
| 游戏服务器 | $49,248 | 35% | 竞价实例 + 预留实例混合 |
| 网络带宽 | $58,320 | 41% | CDN + 压缩协议 |
| Redis 集群 | $10,886 | 8% | 本地缓存降低 Redis 调用 |
| MySQL RDS | $7,632 | 5% | 读写分离 + 冷热分离 |
| 匹配服务 | $6,120 | 4% | 优化算法降低 CPU |
| 其他（监控/日志） | $9,000 | 7% | — |
| **总计** | **~$141,206** | 100% | — |
| **单玩家月成本** | **~$0.014** | — | — |

### 降本策略

1. **游戏服务器 — 竞价+预留混合**：60% 韧性需求用预留实例（最高 72% 折扣），40% 弹性需求用竞价实例（最高 90% 折扣）。需处理竞价实例被回收的情况（与服务器崩溃处理复用）
2. **带宽 — 增量同步**：游戏状态同步从全量快照改为增量差异包，可降低 60% 带宽
3. **Redis — 分层缓存**：热数据（匹配队列、服务器负载）放 Redis，温数据（玩家 MMR）放本地内存缓存，冷数据（对局历史）放 MySQL
4. **错峰调度**：低峰期引导玩家做任务/活动，平滑负载曲线，减少峰值扩容需求

## 常见陷阱（深度分析）

### 陷阱 1：匹配只用段位不用 MMR

**问题：** 段位是粗粒度分类（如黄金 1600-2000），黄金 1600 和黄金 1999 的实力差距很大。

**解决方案：** 用 MMR（连续值）做匹配，段位只用于显示。

### 陷阱 2：有状态服务直接缩容

**后果：** 游戏进行中强制终止服务器 → 10 个玩家的对局中断 → 严重体验问题 + 可能被判负影响段位

**正确做法：** 标记为 draining → 等房间结束 → 再下线。

### 陷阱 3：匹配范围不随等待时间扩展

**后果：** 高段位玩家（钻石/王者）在线人数少，严格匹配可能等 10 分钟以上。

**解决方案：** 等待越久匹配范围越宽，30 秒后允许跨段位匹配。

### 陷阱 4：组队 MMR 取平均值

**问题：** 钻石玩家（MMR 2400）带青铜玩家（MMR 800）组队，平均 MMR = 1600（黄金）。匹配到黄金段位的对手，但钻石玩家碾压全场。

**解决方案：** 组队 MMR 偏向最高值（`avg * 0.6 + max * 0.4`），防止"炸鱼"。

### 陷阱 5：匹配队列只增不减 — 玩家退出匹配未清理

**问题：** 玩家点击"取消匹配"后，如果清理逻辑失败（网络超时/服务重启），该玩家会永远留在匹配队列中。随时间推移，僵尸玩家累积，匹配算法不断尝试匹配已离线玩家，导致匹配成功率下降。

**解决方案：** 双重保障机制：
1. 入队时设置 TTL（120 秒自动过期），匹配服务需要定期续期
2. 定期全量校验：每 5 分钟扫描队列中所有玩家，检查是否在线，清理离线玩家

```python
class QueueGarbageCollector:
    """匹配队列垃圾回收"""

    def cleanup_offline_players(self):
        """清理队列中的离线玩家"""
        for tier in MatchQueue.TIERS:
            queue_key = f"match:queue:{tier}"
            all_players = self.redis.zrange(queue_key, 0, -1)

            for player_id in all_players:
                # 检查玩家是否在线
                is_online = self.redis.exists(f"player:online:{player_id}")
                # 检查入队时间是否过期
                wait_key = f"match:wait:{player_id}"
                is_expired = not self.redis.exists(wait_key)

                if not is_online or is_expired:
                    self.redis.zrem(queue_key, player_id)
                    self.redis.delete(wait_key)
                    self._log(f"Cleaned up offline player {player_id} from {tier}")
```

### 陷阱 6：房间分配忽略网络延迟

**问题：** 纯按服务器负载分配房间，可能把华东的玩家分配到华南的服务器，增加 30-50ms 延迟，在竞技游戏中这足以影响对局体验。

**解决方案：** 房间分配增加区域感知，优先分配同区域服务器：

```python
class RegionAwareServerAllocator:
    """区域感知的房间分配"""

    def allocate_room(self, match_result):
        # 获取匹配中玩家的主要区域
        majority_region = self._get_majority_region(match_result.all_players)

        # 优先在该区域找服务器
        server = self.find_available_server(region=majority_region)

        if server is None:
            # 本区域满 → 尝试相邻区域
            adjacent_regions = self._get_adjacent_regions(majority_region)
            for region in adjacent_regions:
                server = self.find_available_server(region=region)
                if server:
                    # 检查跨区延迟是否可接受
                    latency = self._estimate_cross_region_latency(majority_region, region)
                    if latency < 80:  # 80ms 阈值
                        break
                    server = None  # 延迟太高，继续找

        if server is None:
            # 所有区域都不行 → 扩容本区域
            server = self.scaler.emergency_scale_up(majority_region)

        return server
```

### 陷阱 7：结算不幂等 — 重复扣分/加分

**问题：** 结算服务在写入 MMR 变更时，如果网络超时导致重试，可能对同一局游戏执行两次结算，玩家 MMR 被双倍扣除或双倍增加。

**解决方案：** 使用 room_id 作为幂等键，结算前检查是否已处理：

```python
class IdempotentSettlement:
    """幂等结算服务"""

    def settle(self, room_id, game_result):
        # 幂等检查：该房间是否已结算
        already_settled = self.redis.set(
            f"room:settled:{room_id}", "1",
            nx=True,  # 仅当 key 不存在时设置
            ex=3600   # 1 小时过期
        )

        if not already_settled:
            self._log(f"Room {room_id} already settled, skipping")
            return

        # 执行结算逻辑...
        # 即使后续重试，nx=True 保证不会重复执行
```

## 延伸思考

- **跨区匹配**：不同地区的玩家匹配到同一房间，网络延迟差异大。如何做区域感知匹配？
- **匹配预测**：基于历史数据预测各段位的匹配等待时间，在 UI 上展示"预计等待 15 秒"。
- **自定义房间**：好友约战不需要走匹配系统，但需要房间管理。如何与排位赛房间管理统一？

## 系统架构总览

```
┌──────────────────────────────────────────────────────────────────────┐
│                           客户端层                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │ 玩家客户端 │  │ 玩家客户端 │  │ 玩家客户端 │  │  观战客户端 │            │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘            │
│       │              │              │              │                  │
│       └──────────────┴──────┬───────┴──────────────┘                  │
│                             │ WebSocket / gRPC                       │
└─────────────────────────────┼────────────────────────────────────────┘
                              │
┌─────────────────────────────┼────────────────────────────────────────┐
│                        网关层                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │  API Gateway  │  │  负载均衡 SLB  │  │  连接网关 WS   │               │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘               │
│         └─────────────────┼─────────────────┘                        │
└────────────────────────────┼─────────────────────────────────────────┘
                             │
┌────────────────────────────┼─────────────────────────────────────────┐
│                       服务层                                           │
│                                                                        │
│  ┌──────────────────────────────────────────────────┐                │
│  │               匹配服务集群 (3-5 节点)               │                │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │                │
│  │  │ 匹配调度器 │  │ 匹配算法   │  │ MMR 计算引擎  │   │                │
│  │  └──────────┘  └──────────┘  └──────────────┘   │                │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │                │
│  │  │ 确认管理器 │  │ 组队处理器  │  │  流量控制器    │   │                │
│  │  └──────────┘  └──────────┘  └──────────────┘   │                │
│  └──────────────────────────────────────────────────┘                │
│                                                                        │
│  ┌──────────────────────────────────────────────────┐                │
│  │             伸缩控制器 (1 节点 + 备用)               │                │
│  │  ┌──────────────┐  ┌──────────────────────┐     │                │
│  │  │ 容量预测模型   │  │  伸缩策略执行器         │     │                │
│  │  └──────────────┘  └──────────────────────┘     │                │
│  └──────────────────────────────────────────────────┘                │
│                                                                        │
│  ┌──────────────────────────────────────────────────┐                │
│  │            游戏服务器集群 (100-600 节点)             │                │
│  │  ┌────────┐  ┌────────┐       ┌────────┐         │                │
│  │  │ GS-001 │  │ GS-002 │  ...  │ GS-600 │         │                │
│  │  │500 rooms│  │500 rooms│      │500 rooms│         │                │
│  │  └────────┘  └────────┘       └────────┘         │                │
│  │  每台维护房间状态机 + AI托管 + 断线恢复              │                │
│  └──────────────────────────────────────────────────┘                │
└──────────────────────────────────────────────────────────────────────┘
                             │
┌────────────────────────────┼─────────────────────────────────────────┐
│                       存储层                                           │
│                                                                        │
│  ┌────────────────────┐  ┌──────────────────┐  ┌────────────────┐   │
│  │  Redis 集群 (6节点)  │  │  MySQL (主从)     │  │  对象存储 S3    │   │
│  │  - 匹配队列         │  │  - 玩家档案       │  │  - 游戏回放     │   │
│  │  - 服务器负载       │  │  - 对局记录       │  │  - 状态快照     │   │
│  │  - 玩家 MMR 缓存    │  │  - 房间持久化     │  │                │   │
│  │  - 房间确认状态     │  │  - 结算数据       │  │                │   │
│  └────────────────────┘  └──────────────────┘  └────────────────┘   │
│                                                                        │
│  ┌────────────────────┐  ┌──────────────────┐                        │
│  │  Kafka 消息队列     │  │  Prometheus+Grafana │                      │
│  │  - 结算事件         │  │  - 匹配延迟监控    │                       │
│  │  - 异常告警         │  │  - 服务器负载      │                       │
│  │  - 数据分析         │  │  - 成本看板        │                       │
│  └────────────────────┘  └──────────────────┘                        │
└──────────────────────────────────────────────────────────────────────┘
```

### 关键数据流

```
匹配流程数据流：
  客户端 → API网关 → 匹配服务 → Redis(入队) → 定时器触发匹配
  → 匹配算法计算 → 确认流程 → 房间分配 → 游戏服务器 → 客户端连接

游戏结算数据流：
  游戏服务器 → Kafka(结算事件) → 结算服务 → MySQL(更新MMR/段位/记录)
  → Redis(更新缓存) → 推送通知 → 客户端

伸缩数据流：
  游戏服务器 → 心跳 → Redis(负载上报) → 伸缩控制器 → 容量预测
  → 决策(扩/缩) → 云API(启停实例) → Redis(更新服务器池)
```

### 关键设计决策回顾

| 决策 | 选项 | 选择 | 理由 |
|------|------|------|------|
| 匹配队列存储 | MySQL vs Redis | Redis Sorted Set | ZRANGE 支持 MMR 范围查询，O(logN) 性能 |
| MMR 匹配粒度 | 段位 vs 连续值 | 连续 MMR + 段位分桶 | 精确匹配用 MMR，分桶优化遍历效率 |
| 服务器状态存储 | 中心化 vs 本地 | 本地内存 + 中心快照 | 游戏帧同步需微秒级延迟，中心化不满足 |
| 房间分配策略 | 随机 vs 最低负载 | 最低负载（Sorted Set） | 避免热点服务器过载 |
| 伸缩策略 | 被动 vs 预测性 | 三级（紧急+常规+预测） | 纯被动响应慢，纯预测不准 |
| 断线处理 | 立即判负 vs AI托管 | AI 托管 + 重连窗口 | 保护 9 人的游戏体验 |
## 匹配算法完整实现

```python
class SkillBasedMatchmaker:
    """基于技能的匹配引擎（ELO + Glicko-2）"""

    def find_match(self, player, queue, timeout_seconds=30):
        """为玩家寻找匹配"""
        start_time = time.time()
        search_radius = 100  # 初始搜索范围（ELO 差值）

        while time.time() - start_time < timeout_seconds:
            # 搜索技能相近的玩家
            candidates = self.redis.zrangebyscore(
                f"matchmaking_queue:{player.game_mode}",
                player.mmr - search_radius,
                player.mmr + search_radius)

            # 排除自己和已在队伍中的玩家
            candidates = [c for c in candidates if c != player.id and not self.is_in_team(c)]

            if len(candidates) >= self.team_size * 2 - 1:
                # 足够玩家 → 组队
                teams = self._balance_teams(player, candidates)
                return MatchResult(teams=teams, search_time=time.time() - start_time,
                                   mmr_spread=search_radius)

            # 逐步扩大搜索范围
            search_radius = min(search_radius + 50, 500)
            time.sleep(0.5)

        # 超时：用 AI 机器人填充
        return self._fill_with_bots(player, candidates)

    def _balance_teams(self, player, candidates):
        """平衡两队：使两队平均 MMR 尽量接近"""
        all_players = [player] + [self.get_player(c) for c in candidates]
        all_players.sort(key=lambda p: p.mmr)

        team_a, team_b = [], []
        # 蛇形选人：A1 B1 B2 A2 A3 B3...
        for i, p in enumerate(all_players):
            if i % 4 < 2:
                team_a.append(p)
            else:
                team_b.append(p)

        # 微调：如果两队 MMR 差距 > 50，交换边缘玩家
        diff = abs(sum(p.mmr for p in team_a) - sum(p.mmr for p in team_b))
        if diff > 50 * len(team_a):
            # 找到最接近的交换组合
            best_swap = self._find_best_swap(team_a, team_b)
            if best_swap:
                team_a, team_b = best_swap

        return {"team_a": team_a, "team_b": team_b}

    def _find_best_swap(self, team_a, team_b):
        """找到最佳交换组合使两队 MMR 最接近"""
        current_diff = abs(sum(p.mmr for p in team_a) - sum(p.mmr for p in team_b))
        best_diff = current_diff
        best_teams = None

        for a in team_a:
            for b in team_b:
                new_a = [b if p == a else p for p in team_a]
                new_b = [a if p == b else p for p in team_b]
                new_diff = abs(sum(p.mmr for p in new_a) - sum(p.mmr for p in new_b))
                if new_diff < best_diff:
                    best_diff = new_diff
                    best_teams = (new_a, new_b)

        return best_teams
```

## ELO 评分更新完整实现

```python
class ELOUpdater:
    """ELO 评分更新（含 K-factor 动态调整）"""

    K_FACTOR = {
        "new_player": 40,    # 新手：快速调整
        "under_30_games": 30, # 少于 30 场
        "over_2400": 10,      # 高分段：缓慢调整
        "default": 20,        # 普通
    }

    def update_ratings(self, team_a, team_b, result):
        """根据比赛结果更新双方 ELO"""
        avg_a = sum(p.mmr for p in team_a) / len(team_a)
        avg_b = sum(p.mmr for p in team_b) / len(team_b)

        # 期望胜率
        expected_a = 1 / (1 + 10 ** ((avg_b - avg_a) / 400))
        expected_b = 1 - expected_a

        # 实际结果
        actual_a = 1 if result == "a_wins" else 0
        actual_b = 1 - actual_a

        # 更新每个玩家的评分
        for player in team_a:
            k = self._get_k_factor(player)
            delta = k * (actual_a - expected_a)
            new_mmr = max(0, player.mmr + delta)
            self.db.update("players", {"mmr": new_mmr}, {"id": player.id})

        for player in team_b:
            k = self._get_k_factor(player)
            delta = k * (actual_b - expected_b)
            new_mmr = max(0, player.mmr + delta)
            self.db.update("players", {"mmr": new_mmr}, {"id": player.id})

    def _get_k_factor(self, player):
        if player.total_games < 10:
            return self.K_FACTOR["new_player"]
        elif player.total_games < 30:
            return self.K_FACTOR["under_30_games"]
        elif player.mmr >= 2400:
            return self.K_FACTOR["over_2400"]
        return self.K_FACTOR["default"]
```

## 排队超时与降级策略

```python
class MatchmakingTimeoutHandler:
    """匹配超时处理"""

    def handle_timeout(self, player, wait_time):
        """根据等待时间逐步降级"""
        if wait_time < 10:
            return self.normal_match(player)
        elif wait_time < 20:
            # 放宽匹配条件：扩大 MMR 范围
            return self.relaxed_match(player, mmr_range=300)
        elif wait_time < 30:
            # 进一步放宽 + 允许跨区域匹配
            return self.cross_region_match(player, mmr_range=500)
        else:
            # 超时：用 AI 机器人填充
            return self.bot_filled_match(player)
```

## 异常场景补充

### 场景：MMR 操纵（故意输掉低段位比赛）

```
触发：高分段玩家创建新账号 → 在低段位故意输几场 → MMR 大幅下降
      → 然后碾压低段位玩家 → 破坏游戏体验
检测：
  1. 胜率异常：新账号前 20 场胜率 > 85% → 标记
  2. K/D/A 异常：击杀/死亡比 > 5:1 → 标记
  3. 行为模式：故意送死（不动、原地转圈）→ 检测
处理：
  1. 标记账号 → 进入快速 MMR 调整模式（K=40）
  2. 2-3 场内 MMR 恢复到真实水平
  3. 屡次违规 → 封号
预防：新账号前 10 场只匹配新账号 + 快速调整 K-factor
```

### 场景：匹配队列雪崩（赛季更新）

```
触发：新赛季开始 → 100 倍玩家同时排队 → 匹配服务器过载
检测：
  1. 队列长度 > 100 万 → 告警
  2. 匹配延迟 > 60 秒 → 告警
处理：
  1. 动态扩容匹配服务（自动从 5 台扩展到 20 台）
  2. 临时放宽匹配条件（MMR 范围 100 → 200）
  3. 排队限流：新玩家排队需等待 5 秒
预防：赛季更新前预扩容 + 渐进式开放
```

### 场景：队伍重组断线玩家处理

```
触发：5v5 匹配成功 → 但 1 名玩家在加载时断线
      → 游戏 4v5 不公平
检测：
  1. 加载阶段心跳检测：5 秒无心跳 → 断线
  2. 游戏前 2 分钟无操作 → 断线
处理：
  1. 前 60 秒断线 → 取消比赛，所有玩家重新排队
  2. 60 秒后断线 → AI 接管断线玩家
  3. AI 玩家标记为 "BOT" → 对手可见
  4. 断线玩家可在 3 分钟内重连 → 替换 AI
预防：加载阶段双重确认 + AI 接管兜底
```

## 匹配队列管理完整实现（Redis Sorted Set）

匹配队列是整个匹配系统的核心数据结构，使用 Redis Sorted Set 实现 MMR 有序队列，配合多维度管理策略确保队列高效、干净、公平。

### 队列核心操作

```python
import time
import json
import logging
from datetime import datetime, timedelta
from typing import Optional, List, Tuple, Set
from dataclasses import dataclass, field

logger = logging.getLogger(__name__)

@dataclass
class QueueEntry:
    """匹配队列入队条目"""
    player_id: int
    mmr: int
    game_mode: str          # ranked / casual / custom
    region: str              # cn-east / cn-south / us-west
    party_id: Optional[str]  # 组队 ID（单人则为 None）
    enter_time: float        # 入队时间戳
    max_wait_ms: int = 60000 # 最大等待毫秒数
    preferred_roles: list = field(default_factory=list)  # 角色偏好

class MatchmakingQueueManager:
    """
    匹配队列管理器 — 基于 Redis Sorted Set 的完整队列生命周期管理

    核心设计：
    1. 按游戏模式 + 段位分桶：ZADD matchmaking:{game_mode}:{tier} {mmr} {player_id}
    2. 等待时间扩展：每 5 秒逐步放宽 MMR 匹配范围
    3. 离线清理：每 30 秒扫描并移除离线玩家
    4. 匹配确认：所有玩家必须在 15 秒内确认
    """

    # ============ 常量配置 ============
    QUEUE_KEY_PREFIX = "matchmaking"       # 队列键前缀
    WAIT_KEY_PREFIX = "match:wait"         # 等待时间键前缀
    ONLINE_KEY_PREFIX = "player:online"    # 在线状态键前缀
    CONFIRM_KEY_PREFIX = "match:confirm"   # 确认键前缀
    PARTY_KEY_PREFIX = "match:party"        # 组队键前缀

    # 等待时间扩展配置（秒 → MMR 范围）
    MMR_EXPANSION_STEPS = [
        (0,   100),    # 0-5秒：严格匹配，±100 MMR
        (5,   200),    # 5-10秒：适度放宽，±200 MMR
        (10,  300),    # 10-15秒：较大放宽，±300 MMR
        (15,  400),    # 15-20秒：跨段位匹配，±400 MMR
        (20,  500),    # 20-30秒：大幅放宽，±500 MMR
        (30,  700),    # 30秒+：几乎不限，±700 MMR
    ]

    # 清理配置
    CLEANUP_INTERVAL_SEC = 30              # 离线清理间隔
    QUEUE_ENTRY_TTL_SEC = 120             # 队列条目最大 TTL
    CONFIRM_TIMEOUT_SEC = 15             # 匹配确认超时

    # 队列容量限制
    MAX_QUEUE_SIZE_PER_BUCKET = 100000     # 每个段位桶最大容量
    MAX_TOTAL_QUEUE_SIZE = 500000          # 全局最大队列容量

    def __init__(self, redis_client, db_client, notification_service):
        self.redis = redis_client
        self.db = db_client
        self.notifier = notification_service
        self._cleanup_running = False

    # ============ 入队操作 ============

    def enqueue(self, entry: QueueEntry) -> dict:
        """
        玩家加入匹配队列

        流程：
        1. 前置校验（在线状态、惩罚状态、重复入队）
        2. 写入 Redis Sorted Set（score = MMR）
        3. 记录入队时间
        4. 更新组队信息（如果是组队）
        5. 发送入队确认

        Redis 操作：
        - ZADD matchmaking:{game_mode}:{tier} {mmr} {player_id}
        - SET match:wait:{player_id} {enter_time} EX 120
        """
        # 1. 前置校验
        validation = self._validate_enqueue(entry)
        if not validation["valid"]:
            return {"status": "rejected", "reason": validation["reason"]}

        tier = self._get_tier(entry.mmr)
        queue_key = f"{self.QUEUE_KEY_PREFIX}:{entry.game_mode}:{tier}"

        # 2. 队列容量检查
        bucket_size = self.redis.zcard(queue_key)
        if bucket_size >= self.MAX_QUEUE_SIZE_PER_BUCKET:
            return {"status": "rejected", "reason": "queue_bucket_full",
                    "retry_after_seconds": 30}

        total_size = self._get_total_queue_size()
        if total_size >= self.MAX_TOTAL_QUEUE_SIZE:
            return {"status": "rejected", "reason": "queue_total_full",
                    "retry_after_seconds": 60}

        # 3. 检查玩家是否已在队列中（防止重复入队）
        existing_tier = self._find_player_in_queue(entry.player_id, entry.game_mode)
        if existing_tier:
            return {"status": "already_in_queue", "tier": existing_tier}

        # 4. 写入 Redis Sorted Set
        # ZADD matchmaking:ranked:gold 1800 player_12345
        self.redis.zadd(queue_key, {str(entry.player_id): entry.mmr})

        # 5. 记录入队时间（TTL=120秒，到期自动清理）
        wait_key = f"{self.WAIT_KEY_PREFIX}:{entry.player_id}"
        self.redis.set(wait_key, str(entry.enter_time), ex=self.QUEUE_ENTRY_TTL_SEC)

        # 6. 记录玩家在线状态引用
        self.redis.set(
            f"{self.ONLINE_KEY_PREFIX}:{entry.player_id}", "1", ex=300
        )

        # 7. 组队信息处理
        if entry.party_id:
            party_key = f"{self.PARTY_KEY_PREFIX}:{entry.party_id}"
            self.redis.hset(party_key, str(entry.player_id), entry.mmr)
            self.redis.expire(party_key, self.QUEUE_ENTRY_TTL_SEC)

        # 8. 持久化入队记录到 MySQL（异步）
        self._persist_queue_entry(entry, tier)

        # 9. 发送入队确认通知
        self.notifier.send(entry.player_id, {
            "type": "match_queue_entered",
            "game_mode": entry.game_mode,
            "tier": tier,
            "estimated_wait_seconds": self._estimate_wait_time(entry)
        })

        logger.info(f"Player {entry.player_id} enqueued: tier={tier}, mmr={entry.mmr}")
        return {"status": "enqueued", "tier": tier, "queue_position": bucket_size + 1}

    # ============ 出队操作 ============

    def dequeue(self, player_id: int, game_mode: str = None) -> dict:
        """
        玩家离开匹配队列（主动取消或匹配成功后移除）

        流程：
        1. 查找玩家所在段位桶
        2. 从 Sorted Set 中移除
        3. 清理等待时间键
        4. 清理组队信息
        """
        # 1. 查找玩家所在桶
        tier = self._find_player_in_queue(player_id, game_mode)
        if not tier:
            return {"status": "not_in_queue"}

        # 2. 从 Sorted Set 移除
        queue_key = f"{self.QUEUE_KEY_PREFIX}:{game_mode or 'ranked'}:{tier}"
        removed = self.redis.zrem(queue_key, str(player_id))

        # 3. 清理等待时间键
        wait_key = f"{self.WAIT_KEY_PREFIX}:{player_id}"
        self.redis.delete(wait_key)

        # 4. 清理组队信息
        party_id = self.redis.get(f"match:party_id:{player_id}")
        if party_id:
            party_key = f"{self.PARTY_KEY_PREFIX}:{party_id}"
            self.redis.hdel(party_key, str(player_id))
            # 如果组队中没有成员了，删除组队键
            if self.redis.hlen(party_key) == 0:
                self.redis.delete(party_key)

        # 5. 更新 MySQL 记录
        self.db.execute(
            "UPDATE match_queue_entry SET status = 'cancelled', updated_at = NOW() "
            "WHERE player_id = %s AND status = 'waiting'",
            (player_id,)
        )

        logger.info(f"Player {player_id} dequeued from tier {tier}")
        return {"status": "removed", "tier": tier}

    # ============ MMR 范围查询 ============

    def find_candidates_in_range(self, game_mode: str, tier: str,
                                  center_mmr: int, mmr_range: int,
                                  exclude_ids: Set[int] = None,
                                  limit: int = 20) -> List[Tuple[int, int]]:
        """
        在指定 MMR 范围内查找候选人

        Redis 操作：
        ZRANGEBYSCORE matchmaking:ranked:gold (center-range) (center+range) LIMIT 0 20

        返回：[(player_id, mmr), ...]
        """
        queue_key = f"{self.QUEUE_KEY_PREFIX}:{game_mode}:{tier}"
        min_mmr = center_mmr - mmr_range
        max_mmr = center_mmr + mmr_range

        # ZRANGEBYSCORE 范围查询
        candidates = self.redis.zrangebyscore(
            queue_key, min_mmr, max_mmr,
            start=0, num=limit,
            withscores=True
        )

        # 排除指定 ID（如已匹配的玩家）
        if exclude_ids:
            candidates = [
                (pid, mmr) for pid, mmr in candidates
                if int(pid) not in exclude_ids
            ]

        return [(int(pid), int(mmr)) for pid, mmr in candidates]

    # ============ 等待时间扩展 ============

    def get_current_mmr_range(self, player_id: int) -> int:
        """
        根据玩家等待时间计算当前允许的 MMR 范围

        等待时间扩展策略：
        - 每 5 秒检查一次，逐步放宽匹配范围
        - 使用分段线性插值，平滑扩展

        等待时间计算：
        当前时间 - 入队时间 = wait_seconds
        wait_seconds → 对应 MMR 范围

        示例：
        等待 3 秒 → ±100 MMR
        等待 7 秒 → ±200 MMR
        等待 12 秒 → ±300 MMR
        等待 25 秒 → ±500 MMR
        等待 40 秒 → ±700 MMR
        """
        wait_key = f"{self.WAIT_KEY_PREFIX}:{player_id}"
        enter_time_str = self.redis.get(wait_key)

        if not enter_time_str:
            # 等待时间键不存在（过期或未入队）→ 使用最大范围
            return self.MMR_EXPANSION_STEPS[-1][1]

        wait_seconds = time.time() - float(enter_time_str)

        # 分段线性查找
        mmr_range = self.MMR_EXPANSION_STEPS[0][1]  # 默认最小范围
        for threshold, range_val in self.MMR_EXPANSION_STEPS:
            if wait_seconds >= threshold:
                mmr_range = range_val
            else:
                break

        return mmr_range

    def get_queue_wait_time(self, player_id: int) -> float:
        """获取玩家在队列中的等待时间（秒）"""
        wait_key = f"{self.WAIT_KEY_PREFIX}:{player_id}"
        enter_time_str = self.redis.get(wait_key)

        if not enter_time_str:
            return 0.0

        return time.time() - float(enter_time_str)

    # ============ 队列清理 ============

    def start_cleanup_loop(self):
        """启动离线玩家清理定时任务（每 30 秒执行一次）"""
        if self._cleanup_running:
            return

        self._cleanup_running = True
        import threading
        thread = threading.Thread(target=self._cleanup_loop, daemon=True)
        thread.start()

    def _cleanup_loop(self):
        """定期清理队列中的离线玩家"""
        while self._cleanup_running:
            try:
                self.cleanup_offline_players()
            except Exception as e:
                logger.error(f"Queue cleanup error: {e}")
            time.sleep(self.CLEANUP_INTERVAL_SEC)

    def cleanup_offline_players(self) -> dict:
        """
        清理队列中的离线玩家

        策略：
        1. 遍历所有段位桶
        2. 对桶中每个玩家检查在线状态
        3. 离线玩家 → 从队列移除 + 清理关联键
        4. 等待时间过期玩家 → 从队列移除

        性能优化：
        - 使用 Redis Pipeline 批量检查在线状态
        - 每次清理限制处理数量（避免阻塞）
        - 记录清理统计信息
        """
        cleanup_stats = {
            "offline_removed": 0,
            "expired_removed": 0,
            "tiers_scanned": 0,
            "total_checked": 0
        }

        game_modes = ["ranked", "casual"]
        all_tiers = list(self._get_all_tiers().keys())

        for game_mode in game_modes:
            for tier in all_tiers:
                queue_key = f"{self.QUEUE_KEY_PREFIX}:{game_mode}:{tier}"

                # 获取桶中所有玩家
                all_players = self.redis.zrange(queue_key, 0, -1)
                if not all_players:
                    continue

                cleanup_stats["tiers_scanned"] += 1
                cleanup_stats["total_checked"] += len(all_players)

                # 使用 Pipeline 批量检查在线状态和等待时间
                pipe = self.redis.pipeline()
                for pid in all_players:
                    pipe.exists(f"{self.ONLINE_KEY_PREFIX}:{pid}")
                    pipe.exists(f"{self.WAIT_KEY_PREFIX}:{pid}")
                results = pipe.execute()

                # 解析结果：每个玩家两个结果（在线 + 等待时间）
                to_remove = []
                for i, pid in enumerate(all_players):
                    is_online = results[i * 2]
                    has_wait = results[i * 2 + 1]

                    if not is_online:
                        to_remove.append((pid, "offline"))
                        cleanup_stats["offline_removed"] += 1
                    elif not has_wait:
                        to_remove.append((pid, "expired"))
                        cleanup_stats["expired_removed"] += 1

                # 批量移除
                if to_remove:
                    pipe = self.redis.pipeline()
                    for pid, reason in to_remove:
                        pipe.zrem(queue_key, pid)
                        pipe.delete(f"{self.WAIT_KEY_PREFIX}:{pid}")
                    pipe.execute()

                    logger.info(
                        f"Cleaned {len(to_remove)} players from "
                        f"{game_mode}:{tier}: {to_remove[:5]}..."
                    )

        logger.info(f"Queue cleanup completed: {cleanup_stats}")
        return cleanup_stats

    # ============ 匹配确认流程 ============

    def start_match_confirmation(self, match_id: str,
                                  player_ids: List[int],
                                  timeout_seconds: int = 15) -> dict:
        """
        启动匹配确认流程

        所有玩家必须在 timeout_seconds 内确认，否则：
        - 未确认玩家：踢出 + 惩罚
        - 已确认玩家：尝试补人或重新匹配

        Redis 数据结构：
        - match:confirm:{match_id} → HASH {player_id: confirmed_at}
        - match:confirm_deadline:{match_id} → STRING timestamp, TTL=15s
        """
        # 1. 初始化确认集合
        confirm_key = f"{self.CONFIRM_KEY_PREFIX}:{match_id}"
        deadline_key = f"match:confirm_deadline:{match_id}"

        # 写入待确认玩家集合
        for pid in player_ids:
            self.redis.hset(confirm_key, str(pid), "pending")

        # 设置确认截止时间
        deadline = time.time() + timeout_seconds
        self.redis.setex(deadline_key, timeout_seconds, str(deadline))

        # 2. 通知所有玩家确认
        for pid in player_ids:
            self.notifier.send(pid, {
                "type": "match_confirm_required",
                "match_id": match_id,
                "timeout_seconds": timeout_seconds,
                # 不暴露队友/对手信息，确认后才显示
            })

        # 3. 设置超时检查（到期后触发 on_confirm_timeout）
        self._schedule_confirm_timeout(match_id, timeout_seconds)

        logger.info(f"Match confirmation started: match_id={match_id}, "
                    f"players={len(player_ids)}, timeout={timeout_seconds}s")
        return {"match_id": match_id, "status": "pending_confirmation",
                "player_count": len(player_ids), "deadline": deadline}

    def confirm_player(self, match_id: str, player_id: int) -> dict:
        """
        玩家确认匹配

        流程：
        1. 标记该玩家已确认
        2. 检查是否全部确认
        3. 全部确认 → 触发匹配成立流程
        4. 未全部确认 → 通知等待进度
        """
        confirm_key = f"{self.CONFIRM_KEY_PREFIX}:{match_id}"

        # 1. 检查确认是否仍在有效期
        if not self.redis.exists(f"match:confirm_deadline:{match_id}"):
            return {"status": "expired", "message": "匹配确认已超时"}

        # 2. 检查玩家是否在该匹配中
        current_status = self.redis.hget(confirm_key, str(player_id))
        if current_status is None:
            return {"status": "not_in_match"}
        if current_status == b"confirmed":
            return {"status": "already_confirmed"}

        # 3. 标记确认
        self.redis.hset(confirm_key, str(player_id), "confirmed")

        # 4. 检查全部确认状态
        all_entries = self.redis.hgetall(confirm_key)
        total_players = len(all_entries)
        confirmed_count = sum(
            1 for v in all_entries.values() if v == b"confirmed"
        )
        pending_count = total_players - confirmed_count

        if pending_count == 0:
            # 全部确认 → 匹配成立
            self._on_all_confirmed(match_id, list(all_entries.keys()))
            return {"status": "all_confirmed", "match_id": match_id}
        else:
            # 部分确认 → 通知进度
            self.notifier.send(player_id, {
                "type": "confirm_received",
                "waiting_for": pending_count,
                "total_players": total_players
            })
            return {"status": "confirmed", "waiting_for": pending_count}

    def _on_all_confirmed(self, match_id: str, confirmed_player_ids: List[str]):
        """所有玩家确认匹配后的处理"""
        # 1. 清理确认相关键
        self.redis.delete(f"{self.CONFIRM_KEY_PREFIX}:{match_id}")
        self.redis.delete(f"match:confirm_deadline:{match_id}")

        # 2. 从匹配队列中移除所有已确认玩家
        for pid_str in confirmed_player_ids:
            pid = int(pid_str)
            self.dequeue(pid)

        # 3. 触发房间分配
        logger.info(f"All players confirmed for match {match_id}, "
                    f"triggering room allocation")

    # ============ 辅助方法 ============

    def _validate_enqueue(self, entry: QueueEntry) -> dict:
        """入队前置校验"""
        # 检查玩家是否在线
        online = self.redis.exists(f"{self.ONLINE_KEY_PREFIX}:{entry.player_id}")
        if not online:
            return {"valid": False, "reason": "player_offline"}

        # 检查玩家是否在惩罚中（挂机/退出惩罚）
        penalty_key = f"player:penalty:{entry.player_id}"
        penalty = self.redis.get(penalty_key)
        if penalty:
            penalty_until = float(penalty)
            if time.time() < penalty_until:
                remaining = int(penalty_until - time.time())
                return {"valid": False, "reason": "player_penalized",
                        "penalty_remaining_seconds": remaining}

        # 检查玩家是否已在某个房间中
        in_room = self.redis.exists(f"player:in_room:{entry.player_id}")
        if in_room:
            return {"valid": False, "reason": "player_in_room"}

        return {"valid": True}

    def _get_all_tiers(self) -> dict:
        """返回所有段位及其 MMR 范围"""
        return {
            "bronze":     (0, 1200),
            "silver":     (1200, 1600),
            "gold":       (1600, 2000),
            "platinum":   (2000, 2400),
            "diamond":    (2400, 2800),
            "master":     (2800, 3200),
            "challenger": (3200, 9999),
        }

    def _get_tier(self, mmr: int) -> str:
        """根据 MMR 获取段位名称"""
        for tier, (low, high) in self._get_all_tiers().items():
            if low <= mmr < high:
                return tier
        return "challenger"

    def _find_player_in_queue(self, player_id: int,
                               game_mode: str = None) -> Optional[str]:
        """查找玩家所在的段位桶"""
        modes = [game_mode] if game_mode else ["ranked", "casual"]
        for mode in modes:
            for tier in self._get_all_tiers().keys():
                queue_key = f"{self.QUEUE_KEY_PREFIX}:{mode}:{tier}"
                score = self.redis.zscore(queue_key, str(player_id))
                if score is not None:
                    return tier
        return None

    def _get_total_queue_size(self) -> int:
        """获取全局队列总大小"""
        total = 0
        for mode in ["ranked", "casual"]:
            for tier in self._get_all_tiers().keys():
                queue_key = f"{self.QUEUE_KEY_PREFIX}:{mode}:{tier}"
                total += self.redis.zcard(queue_key)
        return total

    def _estimate_wait_time(self, entry: QueueEntry) -> int:
        """根据段位和时间段估算等待时间（秒）"""
        tier = self._get_tier(entry.mmr)
        queue_key = f"{self.QUEUE_KEY_PREFIX}:{entry.game_mode}:{tier}"
        queue_size = self.redis.zcard(queue_key)

        # 基础等待时间
        base_wait = {
            "bronze": 5, "silver": 8, "gold": 10,
            "platinum": 15, "diamond": 25, "master": 40, "challenger": 60
        }
        estimated = base_wait.get(tier, 15)

        # 队列人数修正：人越多越快匹配
        if queue_size > 100:
            estimated = max(3, estimated - 5)
        elif queue_size < 10:
            estimated += 15

        return estimated

    def _persist_queue_entry(self, entry: QueueEntry, tier: str):
        """持久化入队记录到 MySQL"""
        self.db.execute(
            "INSERT INTO match_queue_entry "
            "(player_id, mmr, queue_type, party_id, region, enter_time, max_wait_ms, status) "
            "VALUES (%s, %s, %s, %s, %s, %s, %s, 'waiting') "
            "ON DUPLICATE KEY UPDATE mmr=%s, status='waiting', enter_time=%s",
            (entry.player_id, entry.mmr, entry.game_mode,
             entry.party_id, entry.region, entry.enter_time,
             entry.max_wait_ms, entry.mmr, entry.enter_time)
        )

    def _schedule_confirm_timeout(self, match_id: str, timeout: int):
        """设置确认超时检查"""
        # 使用 Redis 键过期事件通知，或简单定时器
        import threading
        def check():
            time.sleep(timeout)
            if self.redis.exists(f"match:confirm_deadline:{match_id}"):
                self._on_confirm_timeout(match_id)

        thread = threading.Thread(target=check, daemon=True)
        thread.start()

    def _on_confirm_timeout(self, match_id: str):
        """确认超时处理"""
        confirm_key = f"{self.CONFIRM_KEY_PREFIX}:{match_id}"
        all_entries = self.redis.hgetall(confirm_key)

        unconfirmed = [pid for pid, status in all_entries.items()
                       if status == b"pending"]
        confirmed = [pid for pid, status in all_entries.items()
                     if status == b"confirmed"]

        # 未确认玩家：给予惩罚
        for pid_str in unconfirmed:
            pid = int(pid_str)
            penalty_seconds = 30  # 下次匹配额外等待 30 秒
            self.redis.setex(
                f"player:confirm_penalty:{pid}",
                penalty_seconds, "1"
            )
            self.notifier.send(pid, {
                "type": "confirm_timeout_penalty",
                "message": f"未及时确认匹配，下次匹配需额外等待 {penalty_seconds} 秒"
            })
            # 从队列移除
            self.dequeue(pid)

        # 已确认玩家：尝试补人或重新匹配
        if len(confirmed) >= 8:  # 至少 8 人确认才尝试补人
            need = len(unconfirmed)
            refill = self._try_refill_players(need)
            if refill:
                logger.info(f"Refilled {len(refill)} players for match {match_id}")
            else:
                for pid_str in confirmed:
                    self.match_queue.enqueue_with_priority(int(pid_str))
                self.notifier.send_batch(
                    [int(p) for p in confirmed],
                    {"type": "match_cancelled", "message": "部分玩家未确认，已重新匹配"}
                )
        else:
            # 确认人数不足 → 全部重新入队
            for pid_str in confirmed:
                self.match_queue.enqueue_with_priority(int(pid_str))

        # 清理确认键
        self.redis.delete(confirm_key)
        self.redis.delete(f"match:confirm_deadline:{match_id}")

    def _try_refill_players(self, count: int) -> list:
        """尝试从队列中快速补人"""
        # 取等待时间最长的玩家
        refill = []
        for tier in self._get_all_tiers().keys():
            queue_key = f"{self.QUEUE_KEY_PREFIX}:ranked:{tier}"
            # 取最早的玩家（等待最久）
            oldest = self.redis.zrange(queue_key, 0, count - len(refill) - 1,
                                        withscores=True)
            for pid, mmr in oldest:
                refill.append(int(pid))
                if len(refill) >= count:
                    break
            if len(refill) >= count:
                break
        return refill

    def get_queue_stats(self) -> dict:
        """获取队列统计信息"""
        stats = {}
        for mode in ["ranked", "casual"]:
            stats[mode] = {}
            for tier, (low, high) in self._get_all_tiers().items():
                queue_key = f"{self.QUEUE_KEY_PREFIX}:{mode}:{tier}"
                count = self.redis.zcard(queue_key)
                if count > 0:
                    # 获取 MMR 分布
                    min_mmr = self.redis.zrangebyscore(queue_key, "-inf", "+inf",
                                                        start=0, num=1, withscores=True)
                    max_mmr = self.redis.zrangebyscore(queue_key, "-inf", "+inf",
                                                        start=count-1, num=1, withscores=True)
                    stats[mode][tier] = {
                        "count": count,
                        "mmr_range": (
                            int(min_mmr[0][1]) if min_mmr else 0,
                            int(max_mmr[0][1]) if max_mmr else 0
                        )
                    }
        return stats
```

**队列管理关键指标：**

| 操作 | Redis 命令 | 时间复杂度 | 延迟 |
|------|-----------|-----------|------|
| 入队 | ZADD | O(log N) | < 1ms |
| 出队 | ZREM | O(log N) | < 1ms |
| 范围查询 | ZRANGEBYSCORE | O(log N + M) | < 2ms |
| 在线检查 | EXISTS | O(1) | < 0.5ms |
| 批量清理 | Pipeline ZREM | O(K × log N) | < 10ms |
| 确认状态 | HSET/HGET | O(1) | < 0.5ms |

## 反作弊集成完整实现

匹配系统的公平性不仅依赖算法，还需要与反作弊系统深度集成，在匹配阶段就识别和隔离作弊玩家。

### 小号检测（Smurf Detection）

```python
class SmurfDetector:
    """
    小号检测器 — 识别高技能玩家在新账号上的异常行为

    检测维度：
    1. 胜率异常：新账号前 20 场胜率 > 85% → 标记
    2. KDA 异常：击杀/死亡比 > 5:1 → 标记
    3. 行为模式：操作 APM、走位精度接近高段位玩家
    4. 账号特征：新注册 + 未绑定手机 + 无社交关系

    处理策略：
    - 标记账号 → 进入快速 MMR 调整模式（K=40）
    - 2-3 场内 MMR 恢复到真实水平
    - 屡次违规 → 封号
    - 检测到的小号 → 只匹配其他小号（隔离池）
    """

    # 检测阈值
    WIN_RATE_THRESHOLD = 0.85           # 前 20 场胜率阈值
    KDA_THRESHOLD = 5.0                 # 击杀/死亡比阈值
    APM_SIMILARITY_THRESHOLD = 0.85     # APM 相似度阈值（与大师段位比较）
    MIN_GAMES_FOR_DETECTION = 10        # 最少游戏场数才开始检测
    SUSPICION_SCORE_THRESHOLD = 0.7     # 可疑分数阈值

    def __init__(self, redis, db, ml_model=None):
        self.redis = redis
        self.db = db
        self.ml_model = ml_model  # 可选：机器学习模型

    def check_player(self, player_id: int) -> dict:
        """
        检测玩家是否为小号

        返回：
        {
            "is_smurf": bool,
            "confidence": float,     # 0.0-1.0
            "indicators": list,      # 触发的指标
            "action": str,           # "fast_adjust" / "isolate" / "ban"
            "estimated_real_mmr": int # 估计的真实 MMR
        }
        """
        profile = self._get_player_profile(player_id)

        # 新账号不足 N 场，暂不检测
        if profile.total_games < self.MIN_GAMES_FOR_DETECTION:
            return {"is_smurf": False, "confidence": 0, "indicators": [],
                    "action": "none", "estimated_real_mmr": profile.mmr}

        suspicion_score = 0.0
        indicators = []

        # 维度 1：胜率异常检测
        win_rate = profile.win_count / max(profile.total_games, 1)
        if profile.total_games <= 20 and win_rate > self.WIN_RATE_THRESHOLD:
            win_rate_score = min(1.0, (win_rate - self.WIN_RATE_THRESHOLD) / 0.15)
            suspicion_score += win_rate_score * 0.3
            indicators.append(f"high_win_rate: {win_rate:.2f}")

        # 维度 2：KDA 异常检测
        avg_kda = (profile.total_kills + profile.total_assists) / max(profile.total_deaths, 1)
        if avg_kda > self.KDA_THRESHOLD:
            kda_score = min(1.0, (avg_kda - self.KDA_THRESHOLD) / 5.0)
            suspicion_score += kda_score * 0.25
            indicators.append(f"abnormal_kda: {avg_kda:.1f}")

        # 维度 3：账号特征检测
        account_risk = self._check_account_risk(player_id)
        if account_risk["is_risky"]:
            suspicion_score += account_risk["risk_score"] * 0.2
            indicators.append(f"risky_account: {account_risk['reasons']}")

        # 维度 4：行为模式分析（APM、走位精度等）
        behavior_score = self._analyze_behavior_pattern(player_id, profile)
        if behavior_score > self.SUSPICION_SCORE_THRESHOLD:
            suspicion_score += behavior_score * 0.25
            indicators.append(f"suspicious_behavior: score={behavior_score:.2f}")

        # 综合判定
        is_smurf = suspicion_score >= self.SUSPICION_SCORE_THRESHOLD
        estimated_mmr = self._estimate_real_mmr(profile, suspicion_score)

        # 决策
        if is_smurf:
            action = self._decide_action(suspicion_score, player_id)
            # 写入小号标记
            self._mark_as_smurf(player_id, suspicion_score, indicators)
        else:
            action = "none"

        return {
            "is_smurf": is_smurf,
            "confidence": min(1.0, suspicion_score),
            "indicators": indicators,
            "action": action,
            "estimated_real_mmr": estimated_mmr
        }

    def _check_account_risk(self, player_id: int) -> dict:
        """检查账号风险特征"""
        risk_score = 0.0
        reasons = []

        # 检查注册时间（7 天内注册的新账号风险高）
        created_at = self.db.query(
            "SELECT created_at FROM player_profile WHERE player_id = %s",
            (player_id,)
        )
        if created_at and (datetime.now() - created_at).days < 7:
            risk_score += 0.4
            reasons.append("new_account_7d")

        # 检查是否绑定手机号
        has_phone = self.redis.exists(f"player:phone:{player_id}")
        if not has_phone:
            risk_score += 0.3
            reasons.append("no_phone_binding")

        # 检查社交关系（好友数量 < 3 → 风险高）
        friend_count = self.redis.scard(f"player:friends:{player_id}")
        if friend_count < 3:
            risk_score += 0.2
            reasons.append("low_social_connections")

        # 检查设备指纹关联的账号数量（同一设备多账号 → 风险高）
        device_id = self.redis.get(f"player:device:{player_id}")
        if device_id:
            linked_accounts = self.redis.smembers(f"device:accounts:{device_id}")
            if len(linked_accounts) > 2:
                risk_score += 0.4
                reasons.append(f"multiple_accounts_on_device: {len(linked_accounts)}")

        return {
            "is_risky": risk_score > 0.5,
            "risk_score": min(1.0, risk_score),
            "reasons": reasons
        }

    def _analyze_behavior_pattern(self, player_id: int, profile) -> float:
        """分析玩家行为模式（操作精度、走位等）"""
        # 获取最近 10 场的操作数据
        recent_games = self.db.query(
            "SELECT apm, click_accuracy, movement_score FROM player_behavior "
            "WHERE player_id = %s ORDER BY created_at DESC LIMIT 10",
            (player_id,)
        )
        if not recent_games:
            return 0.0

        # 计算平均 APM
        avg_apm = sum(g.apm for g in recent_games) / len(recent_games)

        # 与大师段位的平均 APM 对比
        master_avg_apm = self.redis.get("stats:master:avg_apm")
        if master_avg_apm:
            master_apm = float(master_avg_apm)
            apm_similarity = 1.0 - abs(avg_apm - master_apm) / master_apm
            return max(0.0, apm_similarity)

        return 0.0

    def _estimate_real_mmr(self, profile, suspicion_score: float) -> int:
        """根据可疑度估算玩家的真实 MMR"""
        # 如果高度可疑，用 KDA 推算
        if suspicion_score > 0.7:
            avg_kda = (profile.total_kills + profile.total_assists) / max(profile.total_deaths, 1)
            # 简单线性映射：KDA 3 → MMR 1500, KDA 8 → MMR 2500
            estimated = 1000 + avg_kda * 200
            return min(3500, int(estimated))
        return profile.mmr

    def _decide_action(self, suspicion_score: float, player_id: int) -> str:
        """根据可疑度决定处理动作"""
        # 检查历史标记次数
        smurf_count = self.redis.get(f"player:smurf_count:{player_id}")
        count = int(smurf_count) if smurf_count else 0

        if count >= 3:
            return "ban"  # 屡次违规 → 封号
        elif suspicion_score > 0.9:
            return "isolate"  # 高度可疑 → 隔离到小号池
        else:
            return "fast_adjust"  # 中度可疑 → 快速调整 MMR

    def _mark_as_smurf(self, player_id: int, score: float, indicators: list):
        """标记玩家为小号"""
        self.redis.set(f"player:smurf:{player_id}", json.dumps({
            "score": score, "indicators": indicators,
            "detected_at": datetime.now().isoformat()
        }), ex=86400 * 30)  # 保留 30 天

        # 增加小号标记计数
        self.redis.incr(f"player:smurf_count:{player_id}")

        # 加入小号隔离池（匹配时只匹配其他小号）
        self.redis.sadd("matchmaking:smurf_pool", str(player_id))

        logger.warning(f"Player {player_id} marked as smurf: "
                       f"score={score:.2f}, indicators={indicators}")
```

### 胜负交易检测（Win Trading Detection）

```python
class WinTradingDetector:
    """
    胜负交易检测器 — 检测同一 IP/设备玩家在对面队伍故意输掉比赛

    检测模式：
    1. 同 IP 匹配检测：同一公网 IP 的玩家被分在对立队伍
    2. 异常行为检测：一方玩家有故意送人头的模式
    3. 时间关联检测：多次匹配到同一玩家且结果异常
    4. 交易模式检测：A 赢 B → B 赢 C → C 赢 A 循环模式

    处理策略：
    - 匹配阶段：同 IP 玩家不分配到对立队伍
    - 游戏阶段：检测到故意送人头 → 实时标记
    - 结算阶段：标记为胜负交易 → 不计算 MMR 变化 + 封号
    """

    def check_match_for_win_trading(self, room_id: str, team1: list,
                                     team2: list) -> dict:
        """在匹配阶段检测胜负交易风险"""
        risk_score = 0.0
        warnings = []

        # 检测 1：同 IP 玩家在对立队伍
        ip_collision = self._check_ip_collision(team1, team2)
        if ip_collision:
            risk_score += 0.5
            warnings.append(f"ip_collision: {ip_collision}")

        # 检测 2：同一设备指纹的关联账号在对立队伍
        device_collision = self._check_device_collision(team1, team2)
        if device_collision:
            risk_score += 0.4
            warnings.append(f"device_collision: {device_collision}")

        # 检测 3：历史交易模式
        for p1 in team1:
            for p2 in team2:
                pattern = self._check_historical_pattern(p1.id, p2.id)
                if pattern["is_suspicious"]:
                    risk_score += pattern["risk_score"]
                    warnings.append(f"historical_pattern: {p1.id} vs {p2.id}")

        is_risky = risk_score >= 0.5

        return {
            "is_risky": is_risky,
            "risk_score": min(1.0, risk_score),
            "warnings": warnings,
            "action": "block_match" if risk_score >= 0.7 else "flag_for_monitoring"
        }

    def _check_ip_collision(self, team1: list, team2: list) -> list:
        """检查两队中是否有相同 IP 的玩家"""
        team1_ips = {}
        for p in team1:
            ip = self.redis.get(f"player:ip:{p.id}")
            if ip:
                team1_ips[ip] = p.id

        collisions = []
        for p in team2:
            ip = self.redis.get(f"player:ip:{p.id}")
            if ip and ip in team1_ips:
                collisions.append({
                    "team1_player": team1_ips[ip],
                    "team2_player": p.id,
                    "shared_ip": ip
                })

        return collisions

    def _check_device_collision(self, team1: list, team2: list) -> list:
        """检查两队中是否有相同设备指纹的关联账号"""
        collisions = []
        for p1 in team1:
            device1 = self.redis.get(f"player:device:{p1.id}")
            if not device1:
                continue
            for p2 in team2:
                device2 = self.redis.get(f"player:device:{p2.id}")
                if device2 and device1 == device2:
                    collisions.append({
                        "team1_player": p1.id,
                        "team2_player": p2.id,
                        "shared_device": device1
                    })
        return collisions

    def _check_historical_pattern(self, player1_id: int, player2_id: int) -> dict:
        """检查两个玩家之间的历史对战模式"""
        # 查询最近 30 天的匹配记录
        matches = self.db.query(
            "SELECT r.winner, mh.player_id, mh.team "
            "FROM match_history mh JOIN game_room r ON mh.room_id = r.room_id "
            "WHERE mh.player_id IN (%s, %s) AND r.room_id IN ("
            "  SELECT room_id FROM match_history WHERE player_id = %s "
            "  INTERSECT "
            "  SELECT room_id FROM match_history WHERE player_id = %s"
            ") AND r.create_time > NOW() - INTERVAL 30 DAY",
            (player1_id, player2_id, player1_id, player2_id)
        )

        if len(matches) < 3:
            return {"is_suspicious": False, "risk_score": 0}

        # 统计 A 总是赢 B 的比例
        a_wins = sum(1 for m in matches
                     if m.player_id == player1_id and m.team == m.winner)
        total = len(matches) // 2  # 每局两条记录
        win_rate = a_wins / max(total, 1)

        # 如果 A 对 B 胜率 > 95%，且 A 对其他玩家胜率正常 → 可疑
        if win_rate > 0.95 and total >= 5:
            return {"is_suspicious": True, "risk_score": 0.3}

        return {"is_suspicious": False, "risk_score": 0}
```

### 账号关联检测（Account Linking by Device Fingerprint）

```python
class AccountLinkDetector:
    """
    账号关联检测器 — 通过设备指纹、IP、行为模式检测大小号关联

    关联图谱：
    - 同设备 → 强关联
    - 同 IP 高频 → 中关联
    - 同社交圈 + 同时段活跃 → 弱关联

    用途：
    1. 识别一个玩家控制的多个账号
    2. 防止同人多号匹配（刷分/胜负交易）
    3. 封号时联动封禁关联账号
    """

    def build_player_link_graph(self, player_id: int, depth: int = 2) -> dict:
        """
        构建玩家关联图谱（BFS 遍历）

        深度 1：直接关联（同设备、同 IP）
        深度 2：间接关联（通过中间账号关联）
        """
        visited = set()
        links = {}
        queue = [(player_id, 0)]

        while queue:
            current_id, current_depth = queue.pop(0)
            if current_id in visited or current_depth > depth:
                continue
            visited.add(current_id)

            # 获取直接关联账号
            device_links = self._get_device_linked_accounts(current_id)
            ip_links = self._get_ip_linked_accounts(current_id)

            direct_links = []
            for linked_id in device_links:
                if linked_id not in visited:
                    direct_links.append({
                        "player_id": linked_id,
                        "link_type": "device",
                        "strength": "strong"
                    })
                    queue.append((linked_id, current_depth + 1))

            for linked_id in ip_links:
                if linked_id not in visited:
                    direct_links.append({
                        "player_id": linked_id,
                        "link_type": "ip",
                        "strength": "medium"
                    })
                    queue.append((linked_id, current_depth + 1))

            if direct_links:
                links[current_id] = direct_links

        return {
            "root_player": player_id,
            "linked_accounts": len(visited) - 1,
            "link_graph": links,
            "strong_links": sum(
                1 for v in links.values()
                for l in v if l["strength"] == "strong"
            )
        }

    def _get_device_linked_accounts(self, player_id: int) -> list:
        """获取同设备关联的账号"""
        device_id = self.redis.get(f"player:device:{player_id}")
        if not device_id:
            return []
        linked = self.redis.smembers(f"device:accounts:{device_id}")
        return [int(a) for a in linked if int(a) != player_id]

    def _get_ip_linked_accounts(self, player_id: int) -> list:
        """获取同 IP 关联的账号（过去 7 天内共用 IP 的账号）"""
        ip = self.redis.get(f"player:ip:{player_id}")
        if not ip:
            return []
        linked = self.redis.smembers(f"ip:accounts:{ip}")
        return [int(a) for a in linked if int(a) != player_id]

    def on_match_assignment(self, team1: list, team2: list) -> dict:
        """
        匹配分配时检查关联账号冲突

        规则：
        - 同设备的账号不能在同一局游戏
        - 同 IP 的账号不能在对立队伍
        """
        all_players = team1 + team2
        conflicts = []

        # 检查同队关联
        for team in [team1, team2]:
            for i, p1 in enumerate(team):
                for p2 in team[i+1:]:
                    linked = self._are_accounts_linked(p1.id, p2.id)
                    if linked:
                        conflicts.append({
                            "type": "same_team_linked",
                            "player1": p1.id, "player2": p2.id,
                            "link_type": linked
                        })

        # 检查跨队 IP 关联
        for p1 in team1:
            for p2 in team2:
                same_ip = self._check_same_ip(p1.id, p2.id)
                if same_ip:
                    conflicts.append({
                        "type": "cross_team_same_ip",
                        "player1": p1.id, "player2": p2.id
                    })

        return {
            "has_conflicts": len(conflicts) > 0,
            "conflicts": conflicts,
            "action": "rematch" if conflicts else "proceed"
        }

    def _are_accounts_linked(self, pid1: int, pid2: int) -> Optional[str]:
        """检查两个账号是否关联"""
        # 检查设备关联
        device1 = self.redis.get(f"player:device:{pid1}")
        device2 = self.redis.get(f"player:device:{pid2}")
        if device1 and device2 and device1 == device2:
            return "device"

        # 检查 IP 关联
        if self._check_same_ip(pid1, pid2):
            return "ip"

        return None

    def _check_same_ip(self, pid1: int, pid2: int) -> bool:
        """检查两个玩家是否使用相同 IP"""
        ip1 = self.redis.get(f"player:ip:{pid1}")
        ip2 = self.redis.get(f"player:ip:{pid2}")
        return ip1 is not None and ip2 is not None and ip1 == ip2
```

## 赛后处理流水线完整实现

赛后处理是匹配系统的最终闭环，涵盖 MMR 更新、数据聚合、成就检测和赛季排名更新。

```python
import json
import logging
from datetime import datetime
from typing import List, Dict, Optional
from dataclasses import dataclass
from enum import Enum

logger = logging.getLogger(__name__)


class SettlementPhase(Enum):
    """结算阶段"""
    MMR_UPDATE = "mmr_update"
    STATS_AGGREGATION = "stats_aggregation"
    ACHIEVEMENT_CHECK = "achievement_check"
    SEASON_RANK_UPDATE = "season_rank_update"
    REWARD_DISTRIBUTION = "reward_distribution"
    COMPLETED = "completed"


@dataclass
class MatchResult:
    """对局结果"""
    room_id: str
    winner: str              # team1 / team2 / draw
    duration_sec: int
    team1_players: list
    team2_players: list
    player_stats: dict       # {player_id: PlayerGameStats}
    game_mode: str
    map_name: str


@dataclass
class PlayerGameStats:
    """玩家对局内统计数据"""
    player_id: int
    team: str
    kills: int = 0
    deaths: int = 0
    assists: int = 0
    damage_dealt: int = 0
    damage_taken: int = 0
    gold_earned: int = 0
    healing_done: int = 0
    champion_used: str = ""
    is_afk: bool = False
    play_duration_sec: int = 0


class PostMatchProcessingPipeline:
    """
    赛后处理流水线

    完整流程：
    ELO 更新 → 统计聚合 → 成就检测 → 赛季排名更新 → 奖励发放

    设计原则：
    1. 流水线式处理：每个阶段独立执行，失败不阻塞后续阶段
    2. 幂等性：同一 room_id 重复执行不产生副作用
    3. 异步化：非关键路径异步执行
    4. 可观测性：每个阶段记录执行时间和结果
    """

    def __init__(self, mmr_calculator, db, redis, achievement_service,
                 season_service, reward_service, notification_service):
        self.mmr_calculator = mmr_calculator
        self.db = db
        self.redis = redis
        self.achievement_service = achievement_service
        self.season_service = season_service
        self.reward_service = reward_service
        self.notifier = notification_service

    def process(self, result: MatchResult) -> dict:
        """
        执行完整的赛后处理流水线

        返回：
        {
            "room_id": str,
            "phases": {phase_name: {status, duration_ms, details}},
            "overall_status": str
        }
        """
        pipeline_start = datetime.now()
        phase_results = {}

        # ===== 阶段 1：ELO/MMR 更新 =====
        phase_start = datetime.now()
        try:
            mmr_result = self._phase_mmr_update(result)
            phase_results[SettlementPhase.MMR_UPDATE.value] = {
                "status": "success",
                "duration_ms": (datetime.now() - phase_start).total_seconds() * 1000,
                "details": mmr_result
            }
        except Exception as e:
            logger.error(f"MMR update failed for room {result.room_id}: {e}")
            phase_results[SettlementPhase.MMR_UPDATE.value] = {
                "status": "failed", "error": str(e),
                "duration_ms": (datetime.now() - phase_start).total_seconds() * 1000
            }
            # MMR 更新失败 → 中止流水线（最关键步骤）
            return {"room_id": result.room_id, "phases": phase_results,
                    "overall_status": "failed_critical"}

        # ===== 阶段 2：统计聚合 =====
        phase_start = datetime.now()
        try:
            stats_result = self._phase_stats_aggregation(result, mmr_result)
            phase_results[SettlementPhase.STATS_AGGREGATION.value] = {
                "status": "success",
                "duration_ms": (datetime.now() - phase_start).total_seconds() * 1000,
                "details": stats_result
            }
        except Exception as e:
            logger.error(f"Stats aggregation failed for room {result.room_id}: {e}")
            phase_results[SettlementPhase.STATS_AGGREGATION.value] = {
                "status": "failed", "error": str(e),
                "duration_ms": (datetime.now() - phase_start).total_seconds() * 1000
            }
            # 统计聚合失败不阻塞后续阶段

        # ===== 阶段 3：成就检测 =====
        phase_start = datetime.now()
        try:
            achievement_result = self._phase_achievement_check(result, mmr_result)
            phase_results[SettlementPhase.ACHIEVEMENT_CHECK.value] = {
                "status": "success",
                "duration_ms": (datetime.now() - phase_start).total_seconds() * 1000,
                "details": achievement_result
            }
        except Exception as e:
            logger.error(f"Achievement check failed for room {result.room_id}: {e}")
            phase_results[SettlementPhase.ACHIEVEMENT_CHECK.value] = {
                "status": "failed", "error": str(e),
                "duration_ms": (datetime.now() - phase_start).total_seconds() * 1000
            }

        # ===== 阶段 4：赛季排名更新 =====
        phase_start = datetime.now()
        try:
            season_result = self._phase_season_rank_update(result, mmr_result)
            phase_results[SettlementPhase.SEASON_RANK_UPDATE.value] = {
                "status": "success",
                "duration_ms": (datetime.now() - phase_start).total_seconds() * 1000,
                "details": season_result
            }
        except Exception as e:
            logger.error(f"Season rank update failed for room {result.room_id}: {e}")
            phase_results[SettlementPhase.SEASON_RANK_UPDATE.value] = {
                "status": "failed", "error": str(e),
                "duration_ms": (datetime.now() - phase_start).total_seconds() * 1000
            }

        # ===== 阶段 5：奖励发放 =====
        phase_start = datetime.now()
        try:
            reward_result = self._phase_reward_distribution(result, mmr_result,
                                                            achievement_result)
            phase_results[SettlementPhase.REWARD_DISTRIBUTION.value] = {
                "status": "success",
                "duration_ms": (datetime.now() - phase_start).total_seconds() * 1000,
                "details": reward_result
            }
        except Exception as e:
            logger.error(f"Reward distribution failed for room {result.room_id}: {e}")
            phase_results[SettlementPhase.REWARD_DISTRIBUTION.value] = {
                "status": "failed", "error": str(e),
                "duration_ms": (datetime.now() - phase_start).total_seconds() * 1000
            }

        # 汇总
        total_duration = (datetime.now() - pipeline_start).total_seconds() * 1000
        any_failure = any(
            p["status"] == "failed" for p in phase_results.values()
        )

        return {
            "room_id": result.room_id,
            "phases": phase_results,
            "overall_status": "completed_with_errors" if any_failure else "completed",
            "total_duration_ms": total_duration
        }

    # ============ 阶段 1：MMR 更新 ============

    def _phase_mmr_update(self, result: MatchResult) -> dict:
        """
        ELO/MMR 更新

        流程：
        1. 幂等检查（room_id 是否已结算）
        2. 计算双方 MMR 变化
        3. 事务性写入
        4. 更新段位（升段/降段）
        5. 更新 Redis 缓存
        """
        # 幂等检查
        settled = self.redis.set(f"room:settled:{result.room_id}", "1", nx=True, ex=3600)
        if not settled:
            logger.info(f"Room {result.room_id} already settled, skipping")
            return {"status": "already_settled"}

        # 批量计算 MMR 变化
        mmr_changes = self.mmr_calculator.batch_update(result)

        # 事务性写入
        with self.db.transaction():
            for player, change in mmr_changes:
                new_mmr = max(0, player.mmr + change)
                is_winner = change > 0

                # 更新玩家档案
                self.db.execute(
                    "UPDATE player_profile SET "
                    "mmr = %s, total_games = total_games + 1, "
                    "win_count = win_count + %s, lose_count = lose_count + %s, "
                    "win_rate = win_count / GREATEST(total_games, 1), "
                    "updated_at = NOW() "
                    "WHERE player_id = %s",
                    (new_mmr, 1 if is_winner else 0, 0 if is_winner else 1, player.id)
                )

                # 写入对局记录
                stats = result.player_stats.get(player.id)
                self.db.execute(
                    "INSERT INTO match_history "
                    "(room_id, player_id, team, is_winner, mmr_before, mmr_after, "
                    "mmr_change, kills, deaths, assists, damage_dealt, gold_earned, "
                    "play_duration, champion_used) "
                    "VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)",
                    (result.room_id, player.id, player.team,
                     1 if is_winner else 0,
                     player.mmr, new_mmr, change,
                     stats.kills if stats else 0,
                     stats.deaths if stats else 0,
                     stats.assists if stats else 0,
                     stats.damage_dealt if stats else 0,
                     stats.gold_earned if stats else 0,
                     result.duration_sec,
                     stats.champion_used if stats else "")
                )

                # 更新段位
                self._update_tier(player, new_mmr)

                # 更新 Redis 缓存
                self.redis.set(f"player:mmr:{player.id}", new_mmr)

        return {
            "mmr_changes": [
                {"player_id": p.id, "change": c, "new_mmr": p.mmr + c}
                for p, c in mmr_changes
            ]
        }

    def _update_tier(self, player, new_mmr: int):
        """更新玩家段位（升段/降段）"""
        new_tier = self._get_tier(new_mmr)
        old_tier = player.tier

        if new_tier != old_tier:
            self.db.execute(
                "UPDATE player_profile SET tier = %s, tier_score = %s "
                "WHERE player_id = %s",
                (new_tier, self._calc_tier_score(new_mmr, new_tier), player.id)
            )

            # 发送升段/降段通知
            if self._is_tier_upgrade(old_tier, new_tier):
                self.notifier.send(player.id, {
                    "type": "tier_upgrade",
                    "old_tier": old_tier, "new_tier": new_tier
                })
            else:
                self.notifier.send(player.id, {
                    "type": "tier_downgrade",
                    "old_tier": old_tier, "new_tier": new_tier
                })

    # ============ 阶段 2：统计聚合 ============

    def _phase_stats_aggregation(self, result: MatchResult,
                                  mmr_result: dict) -> dict:
        """
        统计数据聚合

        流程：
        1. 更新玩家近期统计（近 10 场胜率）
        2. 更新英雄使用统计
        3. 更新区域/模式统计
        4. 写入热数据缓存
        """
        aggregated = {}

        for player_id, stats in result.player_stats.items():
            # 更新近 10 场胜率（Redis List 保存最近 10 场结果）
            recent_key = f"player:recent_results:{player_id}"
            is_win = 1 if stats.kills > 0 and not stats.is_afk else 0
            self.redis.lpush(recent_key, str(is_win))
            self.redis.ltrim(recent_key, 0, 9)  # 只保留最近 10 场

            # 计算近 10 场胜率
            recent_results = self.redis.lrange(recent_key, 0, 9)
            recent_wins = sum(1 for r in recent_results if int(r) == 1)
            recent_10_win = recent_wins

            # 更新到数据库
            self.db.execute(
                "UPDATE player_profile SET recent_10_win = %s "
                "WHERE player_id = %s",
                (recent_10_win, player_id)
            )

            # 更新英雄统计
            if stats.champion_used:
                self.db.execute(
                    "INSERT INTO champion_stats "
                    "(player_id, champion_name, total_games, win_count, "
                    "total_kills, total_deaths, total_assists) "
                    "VALUES (%s, %s, 1, %s, %s, %s, %s) "
                    "ON DUPLICATE KEY UPDATE "
                    "total_games = total_games + 1, "
                    "win_count = win_count + %s, "
                    "total_kills = total_kills + %s, "
                    "total_deaths = total_deaths + %s, "
                    "total_assists = total_assists + %s",
                    (player_id, stats.champion_used,
                     1 if is_win else 0,
                     stats.kills, stats.deaths, stats.assists,
                     1 if is_win else 0,
                     stats.kills, stats.deaths, stats.assists)
                )

            aggregated[player_id] = {
                "recent_10_win": recent_10_win,
                "champion": stats.champion_used
            }

        return {"aggregated_players": len(aggregated), "details": aggregated}

    # ============ 阶段 3：成就检测 ============

    def _phase_achievement_check(self, result: MatchResult,
                                  mmr_result: dict) -> dict:
        """
        成就检测

        检测类型：
        1. 里程碑成就（100 场、1000 场等）
        2. 技能成就（五杀、零死亡等）
        3. 赛季成就（赛季 100 场、赛季钻石等）
        """
        new_achievements = {}

        for player_id, stats in result.player_stats.items():
            achievements = []

            # 里程碑成就：检查总场次
            total_games = self.db.query_one(
                "SELECT total_games FROM player_profile WHERE player_id = %s",
                (player_id,)
            )
            milestones = [100, 500, 1000, 5000]
            for m in milestones:
                if total_games == m:
                    achievements.append({
                        "id": f"milestone_{m}",
                        "name": f"征战{m}场",
                        "type": "milestone"
                    })

            # 技能成就
            if stats.kills >= 5 and stats.deaths == 0:
                achievements.append({
                    "id": "perfect_game",
                    "name": "完美表现",
                    "type": "skill"
                })

            if stats.kills >= 10:
                achievements.append({
                    "id": "ten_kills",
                    "name": "十杀",
                    "type": "skill"
                })

            if stats.assists >= 15:
                achievements.append({
                    "id": "assist_master",
                    "name": "助攻大师",
                    "type": "skill"
                })

            # 写入成就
            for ach in achievements:
                self.db.execute(
                    "INSERT IGNORE INTO player_achievements "
                    "(player_id, achievement_id, achieved_at) "
                    "VALUES (%s, %s, NOW())",
                    (player_id, ach["id"])
                )
                self.notifier.send(player_id, {
                    "type": "achievement_unlocked",
                    "achievement": ach
                })

            if achievements:
                new_achievements[player_id] = achievements

        return {"new_achievements": len(new_achievements),
                "details": new_achievements}

    # ============ 阶段 4：赛季排名更新 ============

    def _phase_season_rank_update(self, result: MatchResult,
                                   mmr_result: dict) -> dict:
        """
        赛季排名更新

        流程：
        1. 获取当前赛季信息
        2. 更新赛季积分
        3. 更新赛季排名（使用 Redis Sorted Set 实时排名）
        4. 检查赛季奖励资格
        """
        season = self.season_service.get_current_season()
        if not season:
            return {"status": "no_active_season"}

        updated_players = []

        for player, change in mmr_result.get("mmr_changes", []):
            player_id = player.id if hasattr(player, 'id') else player

            # 计算赛季积分变化
            season_points_change = self._calc_season_points(result, change)

            # 更新赛季积分表
            self.db.execute(
                "INSERT INTO season_rankings "
                "(season_id, player_id, season_points, total_games, "
                "highest_tier, updated_at) "
                "VALUES (%s, %s, %s, 1, %s, NOW()) "
                "ON DUPLICATE KEY UPDATE "
                "season_points = season_points + %s, "
                "total_games = total_games + 1, "
                "highest_tier = GREATEST(highest_tier, %s), "
                "updated_at = NOW()",
                (season.id, player_id, season_points_change,
                 self._get_tier(player.mmr + change if hasattr(player, 'mmr') else 1000),
                 season_points_change,
                 self._get_tier(player.mmr + change if hasattr(player, 'mmr') else 1000))
            )

            # 更新实时排名（Redis Sorted Set）
            current_points = self.db.query_one(
                "SELECT season_points FROM season_rankings "
                "WHERE season_id = %s AND player_id = %s",
                (season.id, player_id)
            )
            self.redis.zadd(
                f"season:ranking:{season.id}",
                {str(player_id): current_points or 0}
            )

            updated_players.append(player_id)

        return {"season_id": season.id, "updated_players": len(updated_players)}

    def _calc_season_points(self, result: MatchResult, mmr_change) -> int:
        """计算赛季积分变化"""
        base_points = 20 if mmr_change > 0 else 5  # 胜利 20，失败 5
        # 表现加成
        if result.duration_sec < 1200:  # 20 分钟内结束
            base_points += 5  # 快速胜利加成
        return base_points

    # ============ 阶段 5：奖励发放 ============

    def _phase_reward_distribution(self, result: MatchResult,
                                    mmr_result: dict,
                                    achievement_result: dict) -> dict:
        """
        奖励发放

        奖励类型：
        1. 基础奖励：每局获得经验和金币
        2. 胜利加成：获胜方额外奖励
        3. 成就奖励：新解锁成就的奖励
        4. 补偿奖励：服务器异常等补偿
        """
        rewards_distributed = {}

        for player_id, stats in result.player_stats.items():
            rewards = []

            # 基础奖励
            base_gold = 100 + result.duration_sec // 60 * 5
            base_exp = 50 + result.duration_sec // 60 * 3
            rewards.append({"type": "gold", "amount": base_gold})
            rewards.append({"type": "exp", "amount": base_exp})

            # 胜利加成
            is_winner = stats.team == result.winner
            if is_winner:
                rewards.append({"type": "gold", "amount": 50})
                rewards.append({"type": "exp", "amount": 30})
                rewards.append({"type": "season_points", "amount": 20})

            # 成就奖励
            player_achievements = achievement_result.get("details", {}).get(player_id, [])
            for ach in player_achievements:
                ach_reward = self.achievement_service.get_achievement_reward(ach["id"])
                if ach_reward:
                    rewards.append(ach_reward)

            # 执行奖励发放
            self.reward_service.grant_rewards(player_id, rewards)

            rewards_distributed[player_id] = rewards

        return {"distributed": len(rewards_distributed), "details": rewards_distributed}

    # ============ 辅助方法 ============

    def _get_tier(self, mmr: int) -> str:
        tiers = {
            "bronze": (0, 1200), "silver": (1200, 1600),
            "gold": (1600, 2000), "platinum": (2000, 2400),
            "diamond": (2400, 2800), "master": (2800, 3200),
            "challenger": (3200, 9999),
        }
        for name, (low, high) in tiers.items():
            if low <= mmr < high:
                return name
        return "challenger"

    def _calc_tier_score(self, mmr: int, tier: str) -> int:
        """计算段位积分（0-100）"""
        ranges = {
            "bronze": (0, 1200), "silver": (1200, 1600),
            "gold": (1600, 2000), "platinum": (2000, 2400),
            "diamond": (2400, 2800), "master": (2800, 3200),
            "challenger": (3200, 9999),
        }
        low, high = ranges.get(tier, (0, 100))
        return int((mmr - low) / max(high - low, 1) * 100)

    def _is_tier_upgrade(self, old_tier: str, new_tier: str) -> bool:
        """判断是否升段"""
        tier_order = ["bronze", "silver", "gold", "platinum",
                      "diamond", "master", "challenger"]
        return tier_order.index(new_tier) > tier_order.index(old_tier)
```

**赛后处理流水线关键指标：**

| 阶段 | 平均耗时 | 关键操作 | 可容忍失败 |
|------|---------|---------|-----------|
| MMR 更新 | 50ms | 事务写入 MySQL + 缓存更新 | 不可容忍（中止流水线） |
| 统计聚合 | 30ms | 近期胜率 + 英雄统计更新 | 可容忍（异步补偿） |
| 成就检测 | 20ms | 条件判断 + 写入成就表 | 可容忍（下次检测补上） |
| 赛季排名 | 15ms | 积分更新 + 排名 ZADD | 可容忍（排名延迟可接受） |
| 奖励发放 | 10ms | 金币/经验/赛季积分发放 | 可容忍（补偿队列兜底） |
| **总计** | **~125ms** | | |

## 异常场景补充

### 场景 5：匹配服务器崩溃恢复

```
触发：匹配服务主节点宕机 → 队列数据可能丢失 → 所有匹配中断
检测：
  1. 心跳检测：匹配服务每 5 秒向注册中心发送心跳
  2. 心跳丢失 15 秒 → 标记为不可用
  3. 备用节点自动接管
影响：
  1. 正在队列中等待的玩家 → 可能丢失入队状态
  2. 正在确认的匹配 → 确认流程中断
  3. 正在分配房间的匹配 → 房间创建中断
处理：
  1. 备用节点启动 → 从 Redis 重建队列状态（Redis 是主存储，不依赖匹配服务内存）
  2. 遍历所有段位桶 → 检查每个玩家的在线状态 → 清理离线玩家
  3. 正在确认的匹配 → 检查确认截止时间：
     - 未超时 → 继续等待确认
     - 已超时 → 执行超时处理（补人或重新匹配）
  4. 正在分配房间的匹配 → 检查房间是否已创建：
     - 已创建 → 继续后续流程
     - 未创建 → 重新分配
  5. 恢复完成后 → 通知所有在队列中的玩家"匹配服务已恢复"
预防：
  - 匹配服务无状态设计：所有状态存储在 Redis，不在本地内存
  - 多节点部署：至少 3 个节点，任一节点宕机不影响整体服务
  - 定期快照：每 5 分钟将队列状态快照到 MySQL，用于极端情况恢复
关键教训：匹配服务必须是无状态的，Redis 作为主存储确保任何节点都能恢复队列状态
```

### 场景 6：玩家对局中断线处理

```
触发：玩家在对局中网络断开 → 无法操作 → 影响 9 人体验
检测：
  1. 游戏服务器检测到客户端心跳丢失（3 秒无心跳）
  2. 标记玩家为"断线中"状态
  3. 通知队友"队友已断线，AI 正在接管"
处理流程：
  1. 0-60 秒：等待重连
     - 游戏继续，AI 托管断线玩家（中等难度 AI）
     - 断线玩家客户端显示"重新连接中..."
     - 服务器保持该玩家的 WebSocket 连接通道
  2. 60-180 秒：重连窗口
     - AI 继续托管
     - 发送推送通知到玩家手机"您的对局仍在进行，请重新连接"
     - 如果玩家在此期间重连 → AI 停止托管，玩家接管
  3. 180 秒后：判定挂机
     - AI 持续托管到对局结束
     - 标记该玩家为本局"挂机"
     - 结算时按挂机惩罚规则处理
结算处理：
  - 挂机玩家无论胜负都扣除更多 MMR（赢了扣 30，输了双倍扣）
  - 多次挂机（3 次/7 天）→ 禁赛 30 分钟
  - 连续挂机 5 次 → 禁赛 2 小时 + 降段
  - 受影响的队友：获得"坚韧"标记，本局 MMR 扣除减半
预防措施：
  - 客户端弱网优化：断线 3 秒内自动重连
  - 服务端保持连接通道：180 秒内无需重新认证
  - AI 托管兜底：保证 9 人的游戏体验不受影响
关键教训：断线不等于挂机，需要给予充分的重连窗口；AI 托管是保底方案
```

### 场景 7：段位重置滥用

```
触发：赛季初段位重置 → 所有玩家 MMR 软重置（保留 70%）
      → 高段位玩家 MMR 从 2800 降到 1960
      → 低段位玩家 MMR 从 800 降到 560
      → 高低段位玩家混在一起匹配
问题：
  1. 大师玩家匹配到青铜玩家 → 单方面碾压 → 体验极差
  2. 故意利用段位重置"炸鱼"：赛季末故意输到低段位 → 赛季重置后从更低段位开始
  3. 组队利用重置：高段位带低段位朋友组队 → 匹配到中低段位对手
检测：
  1. 赛季初前 20 场 KDA 异常：KDA > 5:1 + 胜率 > 90% → 标记
  2. 历史段位对比：上赛季大师 + 本赛季青铜 → 异常
  3. 组队段位差距：队内最高和最低 MMR 差距 > 800 → 标记
处理：
  1. 软重置而非硬重置：
     - 不直接降低 MMR 值
     - 增大 K 因子（赛季初 K=32，让 MMR 快速回归真实水平）
     - 保留上赛季的"隐藏 MMR"作为匹配参考
  2. 赛季初特殊匹配规则：
     - 前 10 场：根据上赛季段位分桶（不是当前 MMR）
     - 10-20 场：混合使用上赛季段位和当前 MMR
     - 20 场后：完全使用当前 MMR
  3. 反炸鱼机制：
     - 上赛季段位 > 钻石 → 赛季初不匹配黄金以下
     - 组队 MMR 差距 > 500 → 队伍 MMR 按最高值计算
  4. 快速调整：
     - 标记账号 → K 因子提升到 40
     - 连续碾压局（KDA > 8:1）→ MMR 额外加 50
     - 3 场内回到合理段位
预防：
  - 段位重置公式：new_mmr = old_mmr * 0.7 + baseline * 0.3（baseline = 1200）
  - 保留"历史最高段位"字段，作为匹配参考
  - 赛季初匹配池分离：上赛季大师+的玩家匹配池独立
关键教训：段位重置是必要的（防止段位通胀），但需要防滥用机制保护新手体验
```

## 匹配队列管理完整实现

```python
class MatchmakingQueueManager:
    """匹配队列管理：多层级 + 优先级 + 溢出处理"""

    QUEUE_CONFIG = {
        "ranked": {"max_wait_seconds": 120, "mmr_expand_rate": 5},
        "casual": {"max_wait_seconds": 60, "mmr_expand_rate": 10},
        "custom": {"max_wait_seconds": 300, "mmr_expand_rate": 0},
    }

    def enqueue(self, player_id, queue_type, mmr, party_members=None):
        """加入匹配队列"""
        entry = {
            "player_id": player_id,
            "queue_type": queue_type,
            "mmr": mmr,
            "mmr_uncertainty": 100 if queue_type == "ranked" else 200,
            "party_size": len(party_members) + 1 if party_members else 1,
            "party_members": party_members or [],
            "enqueued_at": now(),
            "priority": self._calculate_priority(player_id, queue_type),
        }

        # 分区存储（按 MMR 范围分区减少搜索空间）
        mmr_bucket = mmr // 100 * 100
        self.redis.zadd(f"mm_queue:{queue_type}:{mmr_bucket}",
            {json.dumps(entry): entry["priority"]})

        return {"status": "queued", "estimated_wait": self._estimate_wait(entry)}

    def process_queue(self, queue_type):
        """处理队列（每秒调用）"""
        config = self.QUEUE_CONFIG[queue_type]
        matches = []

        # 遍历 MMR 桶
        for bucket_key in self.redis.keys(f"mm_queue:{queue_type}:*"):
            entries = self.redis.zrange(bucket_key, 0, -1)
            for entry_json in entries:
                entry = json.loads(entry_json)
                wait_time = (now() - entry["enqueued_at"]).total_seconds()

                # 扩大 MMR 搜索范围
                expanded_mmr = entry["mmr_uncertainty"] + \
                    wait_time * config["mmr_expand_rate"]

                # 搜索匹配的玩家
                candidates = self._find_candidates(
                    queue_type, entry["mmr"], expanded_mmr,
                    exclude=[entry["player_id"]] + entry["party_members"])

                if candidates:
                    match = self._form_match(entry, candidates, queue_type)
                    if match:
                        matches.append(match)
                        # 从队列移除匹配到的玩家
                        self._remove_from_queue(match, queue_type)

        return matches

    def _calculate_priority(self, player_id, queue_type):
        """计算队列优先级（分值越低越优先）"""
        priority = 1000
        # 高级玩家优先
        if self.redis.sismember("premium_players", player_id):
            priority -= 200
        # 等待时间越长越优先
        return priority

    def _estimate_wait(self, entry):
        """估算等待时间"""
        recent_waits = self.redis.lrange(f"mm_wait_times:{entry['queue_type']}", 0, 99)
        if recent_waits:
            avg = statistics.mean(float(w) for w in recent_waits)
            return round(avg, 1)
        return 30
```

## 反作弊检测系统

```python
class AntiCheatDetector:
    """反作弊检测：客户端校验 + 服务端异常 + ML 分类"""

    def check_player(self, player_id, match_data):
        """检查玩家是否作弊"""
        flags = []

        # 1. 不可能的反应时间
        reaction_times = match_data.get("reaction_times", [])
        if reaction_times:
            avg_reaction = statistics.mean(reaction_times)
            if avg_reaction < 80:  # 人类极限 ~100ms
                flags.append({
                    "type": "impossible_reaction",
                    "value": f"avg={avg_reaction:.0f}ms (human limit ~100ms)"
                })

        # 2. 完美精度
        shots = match_data.get("shots", 0)
        hits = match_data.get("hits", 0)
        if shots > 20 and hits / shots > 0.95:
            flags.append({
                "type": "perfect_accuracy",
                "value": f"{hits}/{shots} = {hits/shots:.1%}"
            })

        # 3. 瞬移检测
        positions = match_data.get("positions", [])
        for i in range(1, len(positions)):
            dist = self._distance(positions[i-1], positions[i])
            dt = (positions[i]["t"] - positions[i-1]["t"]).total_seconds()
            speed = dist / max(dt, 0.001)
            if speed > 50:  # 50 m/s → 瞬移
                flags.append({
                    "type": "teleportation",
                    "value": f"speed={speed:.0f}m/s at {positions[i]['t']}"
                })
                break

        # 4. ML 分类
        features = self._extract_features(match_data)
        cheat_probability = self.cheat_model.predict(features)
        if cheat_probability > 0.8:
            flags.append({
                "type": "ml_detection",
                "value": f"probability={cheat_probability:.2f}"
            })

        # 5. 处罚
        if flags:
            severity = self._calculate_severity(flags)
            penalty = self._determine_penalty(severity)
            self._apply_penalty(player_id, penalty, flags)

        return {"player_id": player_id, "flags": flags,
                "clean": len(flags) == 0}

    def _determine_penalty(self, severity):
        """确定处罚"""
        if severity >= 8:
            return "permanent_ban"
        elif severity >= 5:
            return "7_day_ban"
        elif severity >= 3:
            return "1_day_ban"
        else:
            return "warning"
```

## 异常场景补充

### 场景：反作弊误封高水平玩家

```
触发：职业选手反应时间 100ms → 被判定为作弊 → 误封 → 社区抗议
检测：
  1. 封禁申诉率高 → 可能误封
  2. 玩家历史数据一直优秀 → 不是突然作弊
处理：
  1. 紧急解封
  2. 加入"已知高手"白名单
  3. 调整阈值（反应时间 < 80ms 而非 < 120ms）
预防：高手白名单 + 阈值保守设置 + 申诉快速通道
```

### 场景：赛季重置导致匹配混乱

```
触发：赛季重置后所有人 MMR 不确定性高 → 匹配质量极差
检测：
  1. 定级赛期间匹配方差极大
  2. 高手 vs 新手频繁出现 → 体验差
处理：
  1. 保留隐藏 MMR 作为匹配参考（仅显示重置后段位）
  2. 定级赛期间缩小 MMR 搜索范围
  3. 定级赛后逐步放开
预防：隐藏 MMR 继承 + 定级赛特殊匹配规则 + 渐进放开
```

## 匹配质量监控与优化完整实现

```python
class MatchQualityMonitor:
    """匹配质量监控：公平性 + 技能方差 + 游戏体验"""

    def get_match_quality_metrics(self, period_hours=1):
        """获取匹配质量指标"""
        return {
            "skill_disparity": self._skill_disparity(period_hours),
            "win_rate_balance": self._win_rate_balance(period_hours),
            "avg_wait_time": self._avg_wait_time(period_hours),
            "match_abandonment_rate": self._abandonment_rate(period_hours),
            "player_satisfaction": self._satisfaction_score(period_hours),
        }

    def _skill_disparity(self, hours):
        """技能差异（团队间 MMR 均值差）"""
        matches = self.db.query(
            "SELECT match_id, "
            "ABS(team1_avg_mmr - team2_avg_mmr) as mmr_diff "
            "FROM match_stats WHERE created_at > NOW() - INTERVAL %s HOUR",
            hours)

        if not matches:
            return {"avg_diff": None}

        diffs = [m["mmr_diff"] for m in matches]
        avg_diff = statistics.mean(diffs)
        large_diff_pct = sum(1 for d in diffs if d > 200) / len(diffs)

        return {
            "avg_mmr_diff": round(avg_diff, 1),
            "large_diff_pct": round(large_diff_pct, 3),
            "status": "poor" if avg_diff > 150 else "ok" if avg_diff > 100 else "good"
        }

    def _win_rate_balance(self, hours):
        """胜率平衡（理想 50%）"""
        team1_wins = self.db.count("match_stats",
            team1_result="win", created_at__gte=now()-timedelta(hours=hours))
        total = self.db.count("match_stats",
            created_at__gte=now()-timedelta(hours=hours))

        win_rate = team1_wins / max(total, 1)
        deviation = abs(win_rate - 0.5)

        return {
            "team1_win_rate": round(win_rate, 3),
            "deviation_from_50": round(deviation, 3),
            "status": "imbalanced" if deviation > 0.05 else "balanced"
        }

    def _abandonment_rate(self, hours):
        """比赛放弃率（匹配后 30 秒内离开）"""
        abandoned = self.db.count("match_events",
            event_type="abandon", abandon_time_seconds__lte=30,
            created_at__gte=now()-timedelta(hours=hours))
        total = self.db.count("matches",
            created_at__gte=now()-timedelta(hours=hours))

        return round(abandoned / max(total, 1), 4)

    def optimize_match_parameters(self):
        """优化匹配参数"""
        quality = self.get_match_quality_metrics()

        adjustments = {}
        if quality["skill_disparity"]["avg_mmr_diff"] > 150:
            adjustments["mmr_tolerance"] = -20  # 收窄 MMR 范围

        if quality["avg_wait_time"]["avg_seconds"] > 90:
            adjustments["mmr_tolerance"] = +10  # 放宽 MMR 范围以减少等待

        if quality["match_abandonment_rate"] > 0.05:
            adjustments["pre_match_confirmation"] = True  # 加入确认环节

        return {"current_quality": quality, "adjustments": adjustments}
```

## 异常场景补充

### 场景：匹配质量与等待时间矛盾

```
触发：收窄 MMR 范围 → 匹配质量上升 → 等待时间翻倍 → 用户流失
检测：
  1. 等待时间 > 2 分钟 → 用户不满
  2. 匹配质量提升但留存下降 → 过度优化
处理：
  1. 动态调整 MMR 范围（等待 >60s 时逐步放宽）
  2. 设置最大等待时间上限（120s）
  3. 超时后使用最宽 MMR 范围匹配
预防：动态 MMR 范围 + 等待时间上限 + 质量-等待权衡
```

### 场景：匹配放弃率飙升

```
触发：新版本加入匹配确认环节 → 用户不确认 → 放弃率飙升 → 匹配效率下降
检测：
  1. 放弃率从 2% 升到 15% → 确认问题
  2. 匹配成功率下降 → 确认失败
处理：
  1. 确认超时从 30s 降到 15s
  2. 取消确认环节（改为直接匹配）
  3. 加入匹配后 5s 内可取消
预防：确认环节优化 + 快速取消 + 监控放弃率变化
```

## 游戏社交与组队系统完整实现

```python
class GameSocialService:
    """游戏社交系统：好友 + 组队 + 帮派"""

    def send_friend_request(self, from_id, to_id):
        """发送好友请求"""
        # 1. 检查限制（最多 200 好友）
        current_friends = self.db.count("friendships",
            user_id=from_id, status="active")
        if current_friends >= 200:
            return {"status": "limit_reached"}

        # 2. 检查是否已发送
        existing = self.db.query_one(
            "SELECT * FROM friend_requests "
            "WHERE from_id = %s AND to_id = %s AND status = 'pending'",
            from_id, to_id)
        if existing:
            return {"status": "already_sent"}

        # 3. 创建请求
        request_id = str(uuid4())
        self.db.insert("friend_requests", {
            "request_id": request_id,
            "from_id": from_id,
            "to_id": to_id,
            "status": "pending",
            "created_at": now()
        })

        # 4. 通知
        self.notification.send(to_id,
            f"玩家 {from_id} 请求添加您为好友", "friend_request")

        return {"request_id": request_id, "status": "sent"}

    def create_party(self, leader_id, max_size=5, activity=None):
        """创建组队"""
        party_id = str(uuid4())

        self.db.insert("parties", {
            "party_id": party_id,
            "leader_id": leader_id,
            "max_size": max_size,
            "activity": activity,  # None = 通用组队
            "status": "open",
            "created_at": now()
        })

        # 领队自动加入
        self.db.insert("party_members", {
            "party_id": party_id,
            "player_id": leader_id,
            "role": "leader",
            "joined_at": now()
        })

        return {"party_id": party_id, "status": "open"}

    def join_party(self, party_id, player_id):
        """加入组队"""
        party = self.db.get_party(party_id)

        # 1. 检查人数限制
        current_size = self.db.count("party_members", party_id=party_id)
        if current_size >= party["max_size"]:
            return {"status": "full"}

        # 2. 检查 MMR 范围（如果有活动要求）
        if party["activity"] and party.get("mmr_range"):
            player_mmr = self._get_player_mmr(player_id)
            if not (party["mmr_range"]["min"] <= player_mmr <= party["mmr_range"]["max"]):
                return {"status": "mmr_out_of_range"}

        # 3. 加入
        self.db.insert("party_members", {
            "party_id": party_id,
            "player_id": player_id,
            "role": "member",
            "joined_at": now()
        })

        # 4. 如果满了 → 自动进入匹配
        current_size += 1
        if current_size == party["max_size"]:
            self.db.update("parties",
                {"status": "ready"}, {"party_id": party_id})
            self.matchmaking.enqueue_party(party_id, party["activity"])

        return {"party_id": party_id, "role": "member",
                "current_size": current_size}
```

## 异常场景补充

### 场景：组队系统被用于骚扰

```
触发：恶意玩家反复创建组队邀请同一玩家 → 骚扰行为
检测：
  1. 同一玩家收到 > 5 次/小时的组队邀请 → 骚扰
  2. 邀请来自同一玩家 → 定向骚扰
处理：
  1. 限制单人邀请频率（每小时最多 3 次）
  2. 支持屏蔽功能（屏蔽后无法发送邀请）
  3. 举报后审核
预防：邀请频率限制 + 屏蔽功能 + 举报机制
```

### 场景：组队成员中途退出

```
触发：组队匹配成功 → 1 人中途退出 → 4 人 vs 5 人 → 游戏不公平
检测：
  1. 游戏开始后有人退出 → 中途退出
  2. 队伍人数不均衡 → 体验差
处理：
  1. 允许替补加入（从公共池中补充）
  2. 退出者扣除信用分
  3. 补充失败 → 提前结束游戏（不影响败方评分）
预防：替补机制 + 退出惩罚 + 人数不均衡时提前结束
```

## 游戏赛季与排行榜完整实现

```python
class SeasonLeaderboardService:
    """赛季排行榜：分区排行 + 奖励 + 重置"""

    RANK_TIERS = [
        {"name": "王者", "top_pct": 0.1, "icon": "crown", "reward": "赛季皮肤+1000钻"},
        {"name": "钻石", "top_pct": 1.0, "icon": "diamond", "reward": "赛季皮肤+500钻"},
        {"name": "铂金", "top_pct": 5.0, "icon": "platinum", "reward": "300钻"},
        {"name": "黄金", "top_pct": 15.0, "icon": "gold", "reward": "150钻"},
        {"name": "白银", "top_pct": 40.0, "icon": "silver", "reward": "50钻"},
        {"name": "青铜", "top_pct": 100.0, "icon": "bronze", "reward": "10钻"},
    ]

    def update_rank(self, player_id, mmr_change, season_id):
        """更新排名"""
        # 1. 更新 MMR
        current = self.db.query_one(
            "SELECT mmr, games_played FROM season_rankings "
            "WHERE player_id = %s AND season_id = %s",
            player_id, season_id)

        new_mmr = (current["mmr"] + mmr_change) if current else 1500 + mmr_change
        games = (current["games_played"] + 1) if current else 1

        # 2. 更新 Redis 有序集合（实时排行）
        self.redis.zadd(f"leaderboard:{season_id}",
            {player_id: new_mmr})

        # 3. 更新数据库（异步）
        self.db.upsert("season_rankings", {
            "player_id": player_id,
            "season_id": season_id,
            "mmr": new_mmr,
            "games_played": games,
            "updated_at": now()
        }, conflict_columns=["player_id", "season_id"])

        # 4. 检查段位变化
        old_tier = self._get_tier(current["mmr"]) if current else None
        new_tier = self._get_tier(new_mmr)
        if old_tier and old_tier != new_tier:
            if self.RANK_TIERS.index(new_tier) < self.RANK_TIERS.index(old_tier):
                self.notification.send(player_id, f"恭喜晋升到 {new_tier['name']}！")
            else:
                self.notification.send(player_id, f"已降至 {new_tier['name']}")

        return {"mmr": new_mmr, "tier": new_tier["name"]}

    def get_leaderboard(self, season_id, page=1, page_size=50):
        """获取排行榜"""
        start = (page - 1) * page_size
        end = start + page_size - 1

        # 从 Redis 获取（降序）
        members = self.redis.zrevrange(f"leaderboard:{season_id}",
            start, end, withscores=True)

        result = []
        for i, (player_id, mmr) in enumerate(members):
            tier = self._get_tier(mmr)
            result.append({
                "rank": start + i + 1,
                "player_id": player_id,
                "mmr": int(mmr),
                "tier": tier["name"],
                "tier_icon": tier["icon"]
            })

        return result

    def get_player_rank(self, player_id, season_id):
        """获取玩家排名"""
        mmr = self.redis.zscore(f"leaderboard:{season_id}", player_id)
        if mmr is None:
            return {"rank": None, "mmr": None}

        rank = self.redis.zrevrank(f"leaderboard:{season_id}", player_id) + 1
        tier = self._get_tier(mmr)

        return {"rank": rank, "mmr": int(mmr), "tier": tier["name"]}

    def end_season(self, season_id):
        """结束赛季 → 发放奖励 → 重置"""
        # 1. 计算每个玩家的最终段位和奖励
        total_players = self.redis.zcard(f"leaderboard:{season_id}")

        # 批量发放奖励
        for tier in self.RANK_TIERS:
            top_n = int(total_players * tier["top_pct"] / 100)
            players = self.redis.zrevrange(f"leaderboard:{season_id}",
                0, top_n - 1)

            for player_id in players:
                self._grant_reward(player_id, tier["reward"], season_id)

        # 2. 标记赛季结束
        self.db.update("seasons",
            {"status": "completed", "ended_at": now()},
            {"id": season_id})

    def _get_tier(self, mmr):
        """根据 MMR 获取段位"""
        # MMR 阈值映射
        thresholds = [
            (2400, self.RANK_TIERS[0]),  # 王者
            (2000, self.RANK_TIERS[1]),  # 钻石
            (1700, self.RANK_TIERS[2]),  # 铂金
            (1400, self.RANK_TIERS[3]),  # 黄金
            (1100, self.RANK_TIERS[4]),  # 白银
            (0, self.RANK_TIERS[5]),     # 青铜
        ]
        for threshold, tier in thresholds:
            if mmr >= threshold:
                return tier
        return self.RANK_TIERS[5]
```

## 异常场景补充

### 场景：排行榜缓存与数据库不一致

```
触发：Redis 排行榜数据与 DB 不一致 → 玩家看到错误排名 → 投诉
检测：
  1. Redis 排名与 DB 计算排名差异 > 5 → 不一致
  2. 玩家投诉排名错误 → 缓存问题
处理：
  1. 定期从 DB 重建 Redis 排行榜
  2. 关键操作同时写 Redis 和 DB
  3. 读取时优先 Redis，降级到 DB
预防：双写 + 定期重建 + 降级读取
```

### 场景：赛季重置时奖励发放失败

```
触发：赛季结束 → 批量发放奖励 → 部分发放失败 → 玩家未收到奖励
检测：
  1. 奖励发放成功率 < 100% → 部分失败
  2. 玩家投诉未收到赛季奖励 → 发放问题
处理：
  1. 记录发放失败列表
  2. 重试失败的发放
  3. 补发遗漏的奖励
预防：幂等发放 + 失败重试 + 发放审计
```

## 游戏反外挂检测完整实现

```python
class AntiCheatDetectionService:
    """反外挂检测：行为分析 + 内存校验 + 数据包验证"""

    CHEAT_INDICATORS = {
        "aimbot": "自动瞄准（击中率异常高）",
        "wallhack": "透视（提前发现隐藏敌人）",
        "speed_hack": "加速（移动速度超限）",
        "auto_click": "自动点击（操作间隔过于均匀）",
        "memory_mod": "内存修改（关键值与服务器不一致）",
    }

    def analyze_player_behavior(self, player_id, match_id):
        """分析玩家行为（赛后）"""
        stats = self.db.get_match_player_stats(match_id, player_id)
        indicators = []

        # 1. 自动瞄准检测（击中率异常）
        if stats["shots_fired"] > 20:
            headshot_rate = stats["headshots"] / stats["shots_fired"]
            avg_headshot_rate = self._get_avg_headshot_rate(match_id)
            if headshot_rate > avg_headshot_rate * 3 and headshot_rate > 0.5:
                indicators.append({
                    "type": "aimbot",
                    "evidence": f"爆头率 {headshot_rate:.1%}，平均 {avg_headshot_rate:.1%}",
                    "confidence": min(1.0, headshot_rate / max(avg_headshot_rate, 0.01) / 5)
                })

        # 2. 透视检测（提前发现敌人）
        if stats.get("pre_aim_events", 0) > 5:
            avg_pre_aim = self._get_avg_pre_aim(match_id)
            if stats["pre_aim_events"] > avg_pre_aim * 4:
                indicators.append({
                    "type": "wallhack",
                    "evidence": f"提前瞄准 {stats['pre_aim_events']} 次，平均 {avg_pre_aim:.1f}",
                    "confidence": 0.7
                })

        # 3. 加速检测（移动速度超限）
        max_speed = self._get_max_legal_speed(match_id)
        if stats.get("max_speed_ms", 0) > max_speed * 1.2:
            indicators.append({
                "type": "speed_hack",
                "evidence": f"最高速度 {stats['max_speed_ms']:.1f}m/s，上限 {max_speed:.1f}m/s",
                "confidence": 0.9
            })

        # 4. 自动点击检测（操作间隔过于均匀）
        if stats.get("action_intervals"):
            intervals = stats["action_intervals"]
            if len(intervals) > 20:
                cv = statistics.stdev(intervals) / max(statistics.mean(intervals), 0.001)
                if cv < 0.05:  # 变异系数极低 → 机器操作
                    indicators.append({
                        "type": "auto_click",
                        "evidence": f"操作间隔变异系数 {cv:.4f}（人类 > 0.1）",
                        "confidence": 0.8
                    })

        # 5. 综合判定
        if indicators:
            max_confidence = max(i["confidence"] for i in indicators)
            verdict = "confirmed" if max_confidence >= 0.8 else \
                     "suspected" if max_confidence >= 0.5 else "review_needed"

            self.db.insert("cheat_detections", {
                "detection_id": str(uuid4()),
                "player_id": player_id, "match_id": match_id,
                "indicators": json.dumps(indicators),
                "verdict": verdict,
                "max_confidence": max_confidence,
                "detected_at": now()
            })

            if verdict == "confirmed":
                self._apply_penalty(player_id, match_id, indicators)

        return {"player_id": player_id, "indicators": indicators,
                "cheat_detected": len(indicators) > 0}

    def verify_client_integrity(self, player_id, client_report):
        """验证客户端完整性"""
        violations = []

        # 1. 内存校验（关键内存值哈希）
        expected_hash = self._get_expected_memory_hash(player_id)
        if client_report.get("memory_hash") != expected_hash:
            violations.append({"type": "memory_mod",
                "detail": "内存校验失败"})

        # 2. 数据包验证（客户端上报值与服务器计算值对比）
        server_state = self._get_server_state(player_id)
        for key in ["position_x", "position_y", "health", "ammo"]:
            client_val = client_report.get(key)
            server_val = server_state.get(key)
            if client_val is not None and server_val is not None:
                if abs(client_val - server_val) > self._get_tolerance(key):
                    violations.append({"type": "value_mismatch",
                        "detail": f"{key}: 客户端={client_val}, 服务器={server_val}"})

        return {"player_id": player_id, "violations": violations,
                "integrity_ok": len(violations) == 0}

    def _apply_penalty(self, player_id, match_id, indicators):
        """应用处罚"""
        cheat_types = set(i["type"] for i in indicators)

        # 查历史违规
        prior_violations = self.db.count("cheat_detections",
            player_id=player_id, verdict="confirmed")

        if prior_violations == 0:
            # 首次 → 封号 7 天
            penalty = "ban_7d"
        elif prior_violations == 1:
            # 二次 → 封号 30 天
            penalty = "ban_30d"
        else:
            # 三次 → 永封
            penalty = "ban_permanent"

        self.db.insert("player_penalties", {
            "player_id": player_id, "match_id": match_id,
            "penalty": penalty, "reason": json.dumps(indicators),
            "applied_at": now()
        })

        # 本场成绩作废
        self.db.update("match_player_stats",
            {"status": "disqualified"}, {"match_id": match_id, "player_id": player_id})
```

## 异常场景补充

### 场景：反外挂误封高水平玩家

```
触发：职业选手爆头率 60% → 被判定为 aimbot → 误封 → 舆论危机
检测：
  1. 被封玩家申诉且历史无违规 → 可能误封
  2. 职业选手/高段位玩家被封 → 高概率误封
处理：
  1. 高段位玩家阈值放宽
  2. 快速申诉通道（4 小时内处理）
  3. 误封 → 补偿 + 公开道歉
预防：段位差异化阈值 + 快速申诉 + 人工复核
```

### 场景：外挂绕过内存校验

```
触发：外挂使用内核级隐藏 → 内存校验返回正常哈希 → 绕过检测
检测：
  1. 行为指标异常但内存校验正常 → 可能绕过
  2. 新型外挂出现 → 需要更新检测
处理：
  1. 多层检测（行为 + 内存 + 数据包 + 服务器端验证）
  2. 服务器端权威（关键计算在服务器执行）
  3. 定期更新检测规则
预防：多层检测 + 服务器权威 + 规则迭代
```

## 游戏道具交易与市场完整实现

```python
class GameItemMarketService:
    """道具交易市场：上架 + 交易 + 结算 + 税费"""

    MARKET_TAX_RATE = 0.05  # 5% 交易税
    LISTING_FEE = 10  # 上架费（钻石）

    def list_item(self, seller_id, item_id, price, currency="diamond"):
        """上架道具"""
        # 1. 检查道具所有权
        item = self.db.query_one(
            "SELECT * FROM player_items WHERE id = %s AND player_id = %s",
            item_id, seller_id)
        if not item:
            return {"status": "not_owner"}

        # 2. 检查道具是否可交易
        if not item.get("tradable", True):
            return {"status": "not_tradable"}

        # 3. 检查是否已上架
        existing = self.db.query_one(
            "SELECT * FROM market_listings "
            "WHERE item_id = %s AND status = 'active'", item_id)
        if existing:
            return {"status": "already_listed"}

        # 4. 扣除上架费
        if not self._deduct_currency(seller_id, self.LISTING_FEE, currency):
            return {"status": "insufficient_fee"}

        # 5. 创建上架记录
        listing_id = str(uuid4())
        self.db.insert("market_listings", {
            "listing_id": listing_id,
            "seller_id": seller_id,
            "item_id": item_id,
            "item_template_id": item["template_id"],
            "price": price,
            "currency": currency,
            "status": "active",
            "listed_at": now(),
            "expires_at": now() + timedelta(days=7)
        })

        # 6. 道具状态改为"已上架"
        self.db.update("player_items",
            {"status": "listed", "listing_id": listing_id},
            {"id": item_id})

        return {"listing_id": listing_id, "price": price, "status": "active"}

    def purchase_item(self, buyer_id, listing_id):
        """购买道具"""
        listing = self.db.get_listing(listing_id)

        if not listing or listing["status"] != "active":
            return {"status": "not_available"}

        if listing["seller_id"] == buyer_id:
            return {"status": "cannot_buy_own"}

        # 1. 检查买家余额
        price = listing["price"]
        if not self._check_balance(buyer_id, price, listing["currency"]):
            return {"status": "insufficient_balance"}

        # 2. 锁定上架记录（防止并发购买）
        updated = self.db.update("market_listings",
            {"status": "processing"}, {"listing_id": listing_id, "status": "active"})
        if not updated:
            return {"status": "already_purchased"}

        # 3. 扣除买家余额
        self._deduct_currency(buyer_id, price, listing["currency"])

        # 4. 计算税费和卖家收入
        tax = round(price * self.MARKET_TAX_RATE)
        seller_income = price - tax

        # 5. 转入卖家余额
        self._add_currency(listing["seller_id"], seller_income, listing["currency"])

        # 6. 转入税费到系统
        self._add_system_tax(tax, listing["currency"])

        # 7. 道具所有权转移
        self.db.update("player_items",
            {"player_id": buyer_id, "status": "owned", "listing_id": None},
            {"id": listing["item_id"]})

        # 8. 更新上架记录
        self.db.update("market_listings",
            {"status": "sold", "buyer_id": buyer_id,
             "tax": tax, "seller_income": seller_income,
             "sold_at": now()},
            {"listing_id": listing_id})

        # 9. 交易记录
        self.db.insert("market_transactions", {
            "transaction_id": str(uuid4()),
            "listing_id": listing_id,
            "buyer_id": buyer_id,
            "seller_id": listing["seller_id"],
            "item_id": listing["item_id"],
            "price": price,
            "tax": tax,
            "currency": listing["currency"],
            "completed_at": now()
        })

        # 10. 通知双方
        self.notification.send(buyer_id, "道具购买成功")
        self.notification.send(listing["seller_id"],
            f"道具已售出，收入 {seller_income} {listing['currency']}")

        return {"status": "purchased", "price": price, "tax": tax}

    def cancel_listing(self, seller_id, listing_id):
        """取消上架"""
        listing = self.db.get_listing(listing_id)

        if listing["seller_id"] != seller_id:
            raise PermissionDeniedError("只有卖家可以取消")

        if listing["status"] != "active":
            return {"status": "cannot_cancel"}

        # 道具状态恢复
        self.db.update("player_items",
            {"status": "owned", "listing_id": None},
            {"id": listing["item_id"]})

        self.db.update("market_listings",
            {"status": "cancelled", "cancelled_at": now()},
            {"listing_id": listing_id})

        # 上架费不退回
        return {"status": "cancelled", "note": "上架费不退回"}
```

## 异常场景补充

### 场景：道具交易并发竞购

```
触发：两个玩家同时购买同一道具 → 第一个扣款成功 → 第二个也扣款 → 重复扣款
检测：
  1. 同一道具出现两条购买记录 → 并发竞购
  2. 买家余额被扣两次 → 资金错误
处理：
  1. 上架记录状态从 active → processing → 防止并发
  2. 第二个购买因状态不是 active → 返回已购买
  3. 对账发现重复扣款 → 退回
预防：状态锁定 + 唯一性约束 + 对账
```

### 场景：道具上架后卖家继续使用

```
触发：道具已上架 → 但卖家仍可装备使用 → 买家买到"正在使用"的道具 → 体验差
检测：
  1. 已上架道具在玩家装备列表中 → 状态不一致
  2. 买家投诉道具无法正常使用 → 上架逻辑缺陷
处理：
  1. 上架时自动卸下装备
  2. 上架道具不可装备/使用
  3. 检查上架状态再允许装备
预防：上架自动卸下 + 状态阻断 + 装备前检查
```

## 游戏赛季排名与奖励完整实现

```python
class GameSeasonRankingService:
    """赛季排名：积分计算 → 排行榜 → 奖励发放 → 赛季结算"""

    RANK_TIERS = {
        "bronze": {"min_rating": 0, "max_rating": 1200, "icon": "🥉"},
        "silver": {"min_rating": 1200, "max_rating": 1500, "icon": "🥈"},
        "gold": {"min_rating": 1500, "max_rating": 1800, "icon": "🥇"},
        "platinum": {"min_rating": 1800, "max_rating": 2100, "icon": "💎"},
        "diamond": {"min_rating": 2100, "max_rating": 9999, "icon": "👑"},
    }

    SEASON_REWARDS = {
        "bronze": {"coins": 500, "skin": "basic_skin"},
        "silver": {"coins": 1000, "skin": "silver_skin"},
        "gold": {"coins": 2000, "skin": "gold_skin", "frame": "gold_frame"},
        "platinum": {"coins": 5000, "skin": "platinum_skin", "frame": "platinum_frame", "emote": "platinum_emote"},
        "diamond": {"coins": 10000, "skin": "diamond_skin", "frame": "diamond_frame", "emote": "diamond_emote", "title": "diamond_title"},
    }

    def update_rating(self, player_id, match_result, season_id):
        """更新赛季积分（ELO 变体）"""
        # 1. 获取当前积分
        current = self.db.query_one(
            "SELECT * FROM season_ratings "
            "WHERE player_id = %s AND season_id = %s",
            player_id, season_id)

        if not current:
            current_rating = 1000  # 初始积分
            current = {"rating": current_rating, "wins": 0, "losses": 0}
        else:
            current_rating = current["rating"]

        # 2. 计算 K 因子（高分 → 变化小）
        if current_rating < 1500:
            k_factor = 32
        elif current_rating < 2000:
            k_factor = 24
        else:
            k_factor = 16

        # 3. 计算期望胜率
        opponent_rating = match_result.get("opponent_rating", current_rating)
        expected = 1 / (1 + 10 ** ((opponent_rating - current_rating) / 400))

        # 4. 计算实际结果
        if match_result["result"] == "win":
            actual = 1.0
        elif match_result["result"] == "loss":
            actual = 0.0
        else:
            actual = 0.5  # 平局

        # 5. 更新积分
        rating_change = round(k_factor * (actual - expected))
        new_rating = max(0, current_rating + rating_change)

        # 6. 更新数据库
        self.db.upsert("season_ratings", {
            "player_id": player_id,
            "season_id": season_id,
            "rating": new_rating,
            "rating_change": rating_change,
            "wins": current.get("wins", 0) + (1 if actual == 1 else 0),
            "losses": current.get("losses", 0) + (1 if actual == 0 else 0),
            "updated_at": now()
        }, conflict_columns=["player_id", "season_id"])

        # 7. 更新排行榜
        self.redis.zadd(f"season_leaderboard:{season_id}",
            {player_id: new_rating})

        # 8. 确定段位
        tier = self._get_tier(new_rating)
        old_tier = self._get_tier(current_rating)

        # 9. 段位变化通知
        if tier != old_tier:
            if self.RANK_TIERS[tier]["min_rating"] > self.RANK_TIERS[old_tier]["min_rating"]:
                self.notification.send(player_id, f"恭喜晋升到 {tier} 段位！")
            else:
                self.notification.send(player_id, f"您已降级到 {tier} 段位")

        return {"player_id": player_id, "old_rating": current_rating,
                "new_rating": new_rating, "change": rating_change,
                "tier": tier, "tier_changed": tier != old_tier}

    def get_leaderboard(self, season_id, page=1, page_size=50):
        """获取排行榜"""
        start = (page - 1) * page_size
        end = start + page_size - 1

        # 从 Redis 获取排名
        rankings = self.redis.zrevrange(f"season_leaderboard:{season_id}",
            start, end, withscores=True)

        result = []
        for rank, (player_id, rating) in enumerate(rankings, start + 1):
            player = self.db.get_user(player_id.decode() if isinstance(player_id, bytes) else player_id)
            tier = self._get_tier(rating)

            result.append({
                "rank": rank,
                "player_id": player_id.decode() if isinstance(player_id, bytes) else player_id,
                "player_name": player.get("name", "unknown") if player else "unknown",
                "rating": round(rating),
                "tier": tier,
                "tier_icon": self.RANK_TIERS[tier]["icon"]
            })

        total = self.redis.zcard(f"season_leaderboard:{season_id}")

        return {"season_id": season_id, "total": total,
                "page": page, "leaderboard": result}

    def settle_season(self, season_id):
        """赛季结算"""
        # 1. 获取所有参与玩家的最终排名
        all_players = self.redis.zrevrange(f"season_leaderboard:{season_id}",
            0, -1, withscores=True)

        # 2. 发放奖励
        rewards_distributed = 0
        for rank, (player_id, rating) in enumerate(all_players, 1):
            player_id = player_id.decode() if isinstance(player_id, bytes) else player_id
            tier = self._get_tier(rating)
            reward = self.SEASON_REWARDS[tier]

            # 发放奖励
            self.db.insert("season_rewards", {
                "reward_id": str(uuid4()),
                "player_id": player_id,
                "season_id": season_id,
                "tier": tier,
                "final_rating": round(rating),
                "final_rank": rank,
                "rewards": json.dumps(reward),
                "status": "granted",
                "granted_at": now()
            })

            # 发放货币
            self._grant_coins(player_id, reward["coins"])

            # 发放皮肤
            if "skin" in reward:
                self._grant_skin(player_id, reward["skin"])

            rewards_distributed += 1

        # 3. 关闭赛季
        self.db.update("seasons",
            {"status": "completed", "completed_at": now(),
             "total_players": len(all_players),
             "rewards_distributed": rewards_distributed},
            {"id": season_id})

        return {"season_id": season_id, "total_players": len(all_players),
                "rewards_distributed": rewards_distributed}

    def _get_tier(self, rating):
        """获取段位"""
        for tier_name, config in self.RANK_TIERS.items():
            if config["min_rating"] <= rating < config["max_rating"]:
                return tier_name
        return "diamond"  # 超过最高段位
```

## 异常场景补充

### 场景：赛季结算时玩家在线

```
触发：赛季结算瞬间 → 玩家正在排位 → 积分计算到新赛季 → 旧赛季积分不准确
检测：
  1. 赛季结束时间点有活跃比赛 → 数据冲突
  2. 旧赛季最终排名与实际不符 → 结算问题
处理：
  1. 赛季结束前 10 分钟停止排位匹配
  2. 结算期间禁止新比赛
  3. 过渡期标记为"赛季切换中"
预防：提前停排 + 禁止新比赛 + 过渡期标记
```

### 场景：排行榜缓存与数据库不一致

```
触发：Redis 排行榜积分 1500 → 数据库 1480 → 排名不准确 → 玩家投诉
检测：
  1. 排行榜分数与数据库不一致 → 缓存不同步
  2. 玩家投诉排名错误 → 数据偏差
处理：
  1. 定期从数据库重建排行榜缓存
  2. 积分更新时同步更新 Redis
  3. 排行榜查询时与数据库交叉验证
预防：定期重建 + 同步更新 + 交叉验证
```

## 游戏匹配公平性评估与优化完整实现

```python
class MatchmakingFairnessService:
    """匹配公平性：胜率分析 → 段位偏差检测 → 匹配质量评分 → 优化调整"""

    def evaluate_match_fairness(self, match_id):
        """评估单场匹配公平性"""
        match = self.db.get_match(match_id)

        # 1. 计算双方平均 MMR
        team_a = [p for p in match["players"] if p["team"] == "A"]
        team_b = [p for p in match["players"] if p["team"] == "B"]

        avg_mmr_a = sum(p["mmr"] for p in team_a) / len(team_a)
        avg_mmr_b = sum(p["mmr"] for p in team_b) / len(team_b)

        mmr_diff = abs(avg_mmr_a - avg_mmr_b)

        # 2. 计算队内 MMR 方差（队内是否实力悬殊）
        var_a = sum((p["mmr"] - avg_mmr_a)**2 for p in team_a) / len(team_a)
        var_b = sum((p["mmr"] - avg_mmr_b)**2 for p in team_b) / len(team_b)

        # 3. 匹配质量评分
        quality_score = self._calculate_match_quality(mmr_diff, var_a, var_b)

        # 4. 胜率预测
        win_probability_a = 1 / (1 + 10**((avg_mmr_b - avg_mmr_a) / 400))

        # 5. 记录评估结果
        self.db.insert("match_fairness_evaluations", {
            "eval_id": str(uuid4()),
            "match_id": match_id,
            "avg_mmr_a": round(avg_mmr_a, 1),
            "avg_mmr_b": round(avg_mmr_b, 1),
            "mmr_diff": round(mmr_diff, 1),
            "variance_a": round(var_a, 1),
            "variance_b": round(var_b, 1),
            "quality_score": round(quality_score, 3),
            "predicted_win_rate_a": round(win_probability_a, 3),
            "actual_winner": match.get("winner"),
            "evaluated_at": now()
        })

        return {
            "match_id": match_id,
            "avg_mmr_a": round(avg_mmr_a, 1),
            "avg_mmr_b": round(avg_mmr_b, 1),
            "mmr_diff": round(mmr_diff, 1),
            "quality_score": round(quality_score, 3),
            "predicted_win_rate_a": round(win_probability_a, 3),
            "fair": quality_score >= 0.6
        }

    def _calculate_match_quality(self, mmr_diff, var_a, var_b):
        """计算匹配质量评分"""
        # 1. MMR 差异评分（差异越小越好）
        max_acceptable_diff = 200
        diff_score = max(0, 1 - mmr_diff / max_acceptable_diff)

        # 2. 队内方差评分（方差越小越好）
        max_acceptable_var = 10000  # 标准差 100
        var_score = max(0, 1 - (var_a + var_b) / (2 * max_acceptable_var))

        # 加权
        quality = diff_score * 0.7 + var_score * 0.3
        return quality

    def get_matchmaking_stats(self, time_range_hours=24):
        """获取匹配统计"""
        cutoff = now() - timedelta(hours=time_range_hours)

        # 1. 匹配质量分布
        evaluations = self.db.query(
            "SELECT * FROM match_fairness_evaluations "
            "WHERE evaluated_at >= %s", cutoff)

        if not evaluations:
            return {"total_matches": 0}

        quality_scores = [e["quality_score"] for e in evaluations]
        avg_quality = statistics.mean(quality_scores)

        # 公平匹配比例（quality >= 0.6）
        fair_matches = sum(1 for q in quality_scores if q >= 0.6)
        fair_pct = fair_matches / len(quality_scores)

        # 2. 胜率预测准确率
        correct_predictions = 0
        total_predictions = 0
        for e in evaluations:
            if e.get("actual_winner"):
                predicted = "A" if e["predicted_win_rate_a"] > 0.5 else "B"
                if predicted == e["actual_winner"]:
                    correct_predictions += 1
                total_predictions += 1

        prediction_accuracy = correct_predictions / max(total_predictions, 1)

        # 3. 平均等待时间
        avg_wait = self.db.query_one(
            "SELECT AVG(wait_seconds) as avg FROM matchmaking_logs "
            "WHERE created_at >= %s", cutoff)["avg"] or 0

        # 4. MMR 差异分布
        mmr_diffs = [e["mmr_diff"] for e in evaluations]
        avg_mmr_diff = statistics.mean(mmr_diffs)

        return {
            "time_range_hours": time_range_hours,
            "total_matches": len(evaluations),
            "avg_quality_score": round(avg_quality, 3),
            "fair_match_pct": round(fair_pct * 100, 1),
            "prediction_accuracy": round(prediction_accuracy * 100, 1),
            "avg_wait_seconds": round(float(avg_wait), 1),
            "avg_mmr_diff": round(avg_mmr_diff, 1),
            "quality_distribution": {
                "excellent": sum(1 for q in quality_scores if q >= 0.8),
                "good": sum(1 for q in quality_scores if 0.6 <= q < 0.8),
                "poor": sum(1 for q in quality_scores if q < 0.6),
            }
        }

    def optimize_matchmaking_params(self):
        """优化匹配参数"""
        stats = self.get_matchmaking_stats(168)  # 7 天数据

        recommendations = []

        # 1. 公平匹配比例过低 → 收窄 MMR 范围
        if stats.get("fair_match_pct", 100) < 70:
            recommendations.append({
                "param": "mmr_range",
                "action": "narrow",
                "reason": f"公平匹配比例 {stats['fair_match_pct']}% < 70%"
            })

        # 2. 等待时间过长 → 扩大 MMR 范围
        if stats.get("avg_wait_seconds", 0) > 120:
            recommendations.append({
                "param": "mmr_range",
                "action": "widen",
                "reason": f"平均等待 {stats['avg_wait_seconds']}s > 120s"
            })

        # 3. 预测准确率过高 → 匹配太容易预测 → 需要更平衡
        if stats.get("prediction_accuracy", 50) > 65:
            recommendations.append({
                "param": "match_balance",
                "action": "tighten",
                "reason": f"预测准确率 {stats['prediction_accuracy']}% > 65%，匹配可预测性过高"
            })

        return {"current_stats": stats, "recommendations": recommendations}
```

## 异常场景补充

### 场景：高分段玩家匹配等待时间过长

```
触发：前 0.1% 玩家 MMR 3000+ → 同级别玩家极少 → 等待 10 分钟 → 流失
检测：
  1. 高 MMR 玩家等待时间 > 5 分钟 → 供给不足
  2. 高段位玩家在线数 < 10 → 匹配困难
处理：
  1. 高段位扩大 MMR 匹配范围
  2. 允许高段位玩家匹配低一段（但队内平衡）
  3. 提供练习模式替代等待
预防：动态 MMR 范围 + 跨段位匹配 + 替代模式
```

### 场景：组队匹配导致实力不均

```
触发：3 人组队 MMR 平均 2000 → 但包含 1 个 3000 + 2 个 1500 → 匹配 3 个 2000 → 实力悬殊
检测：
  1. 组队内 MMR 方差过大 → 实力不均
  2. 组队匹配胜率偏差 → 不公平
处理：
  1. 组队匹配使用加权 MMR（最高者权重更高）
  2. 限制组队内 MMR 差距（最大 500）
  3. 组队匹配对手也允许一定方差
预防：加权 MMR + 差距限制 + 对手方差
```

## 游戏匹配算法调优与 A/B 测试完整实现

```python
import math
import hashlib
import time
import json
import redis
import threading
from collections import defaultdict
from datetime import datetime, timedelta
from typing import Dict, List, Tuple, Optional, Any
from concurrent.futures import ThreadPoolExecutor
from scipy import stats as scipy_stats


class MatchmakingTuningService:
    """游戏匹配算法调优与A/B测试服务，支持实验创建、算法评估、参数网格搜索和报告生成"""

    def __init__(self, db_connection, redis_client: redis.Redis):
        self.db = db_connection
        self.redis = redis_client
        self.experiment_lock = threading.Lock()
        self.simulation_cache: Dict[str, Any] = {}
        self.mmr_brackets = {
            "0-1000": (0, 1000),
            "1000-2000": (1000, 2000),
            "2000+": (2000, 99999)
        }
        self.default_algorithm = "standard_mmr"
        self.max_simulation_workers = 8
        self.experiment_duration_days = 7
        self.quality_cap = 1.0

    def run_matchmaking_experiment(self, experiment_config: Dict) -> Dict:
        """创建A/B测试实验，按player_id哈希分流，分别使用当前算法和候选算法，采集匹配质量、等待时间、满意度指标"""
        experiment_id = experiment_config.get("experiment_id", f"exp_{int(time.time())}")
        candidate_algorithm = experiment_config.get("candidate_algorithm", {})
        candidate_name = candidate_algorithm.get("name", "candidate_v1")
        experiment_name = experiment_config.get("name", f"Experiment {experiment_id}")
        start_time = datetime.utcnow()
        end_time = start_time + timedelta(days=self.experiment_duration_days)

        with self.experiment_lock:
            cursor = self.db.cursor()
            cursor.execute(
                "SELECT experiment_id FROM matchmaking_experiments WHERE experiment_id = %s AND status = 'running'",
                (experiment_id,)
            )
            existing = cursor.fetchone()
            if existing:
                cursor.close()
                return {
                    "experiment_id": experiment_id,
                    "status": "already_exists",
                    "message": f"Experiment {experiment_id} is already running"
                }

            cursor.execute(
                """INSERT INTO matchmaking_experiments
                   (experiment_id, name, candidate_algorithm, status, start_time, end_time, created_at)
                   VALUES (%s, %s, %s, %s, %s, %s, %s)""",
                (experiment_id, experiment_name, json.dumps(candidate_algorithm),
                 "running", start_time, end_time, start_time)
            )
            self.db.commit()
            cursor.close()

        cursor = self.db.cursor()
        cursor.execute("SELECT player_id, mmr FROM players WHERE is_active = TRUE ORDER BY player_id")
        players = cursor.fetchall()
        cursor.close()

        group_a_players = []
        group_b_players = []
        for player_id, mmr in players:
            hash_input = f"{player_id}_{experiment_id}".encode("utf-8")
            hash_value = int(hashlib.md5(hash_input).hexdigest(), 16)
            if hash_value % 2 == 0:
                group_a_players.append({"player_id": player_id, "mmr": mmr, "group": "A"})
            else:
                group_b_players.append({"player_id": player_id, "mmr": mmr, "group": "B"})

        self.redis.set(f"experiment:{experiment_id}:group_a_count", len(group_a_players))
        self.redis.set(f"experiment:{experiment_id}:group_b_count", len(group_b_players))
        self.redis.set(f"experiment:{experiment_id}:candidate_algorithm", json.dumps(candidate_algorithm))
        self.redis.set(f"experiment:{experiment_id}:status", "running")
        self.redis.expire(f"experiment:{experiment_id}:status", self.experiment_duration_days * 86400 + 3600)

        for player in group_a_players:
            self.redis.sadd(f"experiment:{experiment_id}:group_a", player["player_id"])
        for player in group_b_players:
            self.redis.sadd(f"experiment:{experiment_id}:group_b", player["player_id"])

        group_a_sample = group_a_players[:20] if len(group_a_players) > 20 else group_a_players
        group_b_sample = group_b_players[:20] if len(group_b_players) > 20 else group_b_players

        for player in group_a_sample:
            match_result = self._simulate_match_with_algorithm(
                player, self.default_algorithm, experiment_id
            )
            self._record_experiment_result(experiment_id, player, "A", match_result)

        for player in group_b_sample:
            match_result = self._simulate_match_with_algorithm(
                player, candidate_algorithm, experiment_id
            )
            self._record_experiment_result(experiment_id, player, "B", match_result)

        return {
            "experiment_id": experiment_id,
            "status": "started",
            "name": experiment_name,
            "candidate_algorithm": candidate_name,
            "group_a": {
                "algorithm": self.default_algorithm,
                "player_count": len(group_a_players),
                "description": "Control group using current production algorithm"
            },
            "group_b": {
                "algorithm": candidate_name,
                "player_count": len(group_b_players),
                "description": "Treatment group using candidate algorithm"
            },
            "metrics_collected": ["match_quality", "wait_time_seconds", "player_satisfaction"],
            "start_time": start_time.isoformat(),
            "end_time": end_time.isoformat(),
            "duration_days": self.experiment_duration_days
        }

    def _simulate_match_with_algorithm(self, player: Dict, algorithm: Dict, experiment_id: str) -> Dict:
        """使用指定算法模拟匹配并计算指标"""
        player_id = player["player_id"]
        player_mmr = player["mmr"]
        algorithm_name = algorithm.get("name", "standard_mmr")
        k_factor = algorithm.get("k_factor", 16)
        expansion_rate = algorithm.get("expansion_rate", 10)
        max_wait = algorithm.get("max_wait_seconds", 60)

        wait_time = min(abs(hash(f"{player_id}_{algorithm_name}") % max_wait) + 1, max_wait)

        mmr_expansion = expansion_rate * wait_time
        search_range = mmr_expansion * 10

        teammate_mmr = player_mmr + (hash(f"{player_id}_teammate") % int(search_range + 1)) - int(search_range / 2)
        opponent_mmr = player_mmr + (hash(f"{player_id}_opponent") % int(search_range + 1)) - int(search_range / 2)

        mmr_diff_teammate = abs(player_mmr - teammate_mmr)
        mmr_diff_opponent = abs(player_mmr - opponent_mmr)
        avg_mmr_diff = (mmr_diff_teammate + mmr_diff_opponent) / 2

        match_quality = max(0.0, 1.0 - avg_mmr_diff / 200.0)
        match_quality = min(match_quality, self.quality_cap)

        played_again_hash = hash(f"{player_id}_{algorithm_name}_{int(time.time()/86400)}")
        player_satisfaction = 1 if played_again_hash % 10 < 7 else 0

        return {
            "match_quality": round(match_quality, 4),
            "wait_time_seconds": wait_time,
            "player_satisfaction": player_satisfaction,
            "mmr_diff": avg_mmr_diff,
            "teammate_mmr": teammate_mmr,
            "opponent_mmr": opponent_mmr
        }

    def _record_experiment_result(self, experiment_id: str, player: Dict,
                                   group: str, match_result: Dict) -> None:
        """记录实验结果到数据库"""
        cursor = self.db.cursor()
        cursor.execute(
            """INSERT INTO experiment_results
               (experiment_id, player_id, player_mmr, group_label, match_quality,
                wait_time_seconds, player_satisfaction, mmr_diff, recorded_at)
               VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)""",
            (experiment_id, player["player_id"], player["mmr"], group,
             match_result["match_quality"], match_result["wait_time_seconds"],
             match_result["player_satisfaction"], match_result["mmr_diff"],
             datetime.utcnow())
        )
        self.db.commit()
        cursor.close()

    def evaluate_matchmaking_algorithm(self, algorithm_name: str, period_days: int = 7) -> Dict:
        """评估匹配算法的关键指标：平均匹配质量、MMR分段等待时间、留存率、胜负平衡，并进行统计显著性检验"""
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(days=period_days)

        cursor = self.db.cursor()
        cursor.execute(
            """SELECT er.player_id, er.player_mmr, er.group_label, er.match_quality,
                      er.wait_time_seconds, er.player_satisfaction, er.mmr_diff, er.experiment_id
               FROM experiment_results er
               JOIN matchmaking_experiments me ON er.experiment_id = me.experiment_id
               WHERE me.candidate_algorithm->>'name' = %s OR 'standard_mmr' = %s
               AND er.recorded_at >= %s AND er.recorded_at <= %s""",
            (algorithm_name, algorithm_name, start_time, end_time)
        )
        all_results = cursor.fetchall()
        cursor.close()

        if not all_results:
            cursor = self.db.cursor()
            cursor.execute(
                """SELECT er.player_id, er.player_mmr, er.group_label, er.match_quality,
                          er.wait_time_seconds, er.player_satisfaction, er.mmr_diff, er.experiment_id
                   FROM experiment_results er
                   WHERE er.recorded_at >= %s AND er.recorded_at <= %s""",
                (start_time, end_time)
            )
            all_results = cursor.fetchall()
            cursor.close()

        if not all_results:
            return {
                "algorithm_name": algorithm_name,
                "period_days": period_days,
                "error": "No experiment data found for the specified period",
                "metrics": None
            }

        candidate_results = []
        baseline_results = []
        for row in all_results:
            player_id, player_mmr, group_label, match_quality, wait_time, satisfaction, mmr_diff, exp_id = row
            entry = {
                "player_id": player_id,
                "player_mmr": player_mmr,
                "group_label": group_label,
                "match_quality": float(match_quality),
                "wait_time_seconds": int(wait_time),
                "player_satisfaction": int(satisfaction),
                "mmr_diff": float(mmr_diff)
            }
            if group_label == "B":
                candidate_results.append(entry)
            else:
                baseline_results.append(entry)

        eval_results = candidate_results if candidate_results else baseline_results

        quality_scores = [r["match_quality"] for r in eval_results]
        avg_match_quality = sum(quality_scores) / len(quality_scores) if quality_scores else 0.0

        wait_by_mmr_bracket: Dict[str, Dict] = {}
        for bracket_name, (low, high) in self.mmr_brackets.items():
            bracket_waits = [r["wait_time_seconds"] for r in eval_results if low <= r["player_mmr"] < high]
            if bracket_waits:
                avg_wait = sum(bracket_waits) / len(bracket_waits)
                min_wait = min(bracket_waits)
                max_wait = max(bracket_waits)
                p50_wait = sorted(bracket_waits)[len(bracket_waits) // 2]
                wait_by_mmr_bracket[bracket_name] = {
                    "avg_wait_seconds": round(avg_wait, 2),
                    "min_wait_seconds": min_wait,
                    "max_wait_seconds": max_wait,
                    "p50_wait_seconds": p50_wait,
                    "sample_count": len(bracket_waits)
                }
            else:
                wait_by_mmr_bracket[bracket_name] = {
                    "avg_wait_seconds": 0.0,
                    "min_wait_seconds": 0,
                    "max_wait_seconds": 0,
                    "p50_wait_seconds": 0,
                    "sample_count": 0
                }

        total_matched = len(eval_results)
        satisfaction_count = sum(1 for r in eval_results if r["player_satisfaction"] == 1)
        retention_rate = satisfaction_count / total_matched if total_matched > 0 else 0.0

        cursor = self.db.cursor()
        cursor.execute(
            """SELECT winner_team FROM match_outcomes
               WHERE algorithm_used = %s AND match_time >= %s AND match_time <= %s""",
            (algorithm_name, start_time, end_time)
        )
        outcome_rows = cursor.fetchall()
        cursor.close()

        if outcome_rows:
            team_a_wins = sum(1 for row in outcome_rows if row[0] == "A")
            total_matches = len(outcome_rows)
            win_rate_a = team_a_wins / total_matches if total_matches > 0 else 0.5
        else:
            simulated_wins = sum(1 for r in eval_results if hash(r["player_id"]) % 2 == 0)
            total_simulated = len(eval_results) if eval_results else 1
            win_rate_a = simulated_wins / total_simulated
        match_outcome_balance = abs(win_rate_a - 0.5)

        baseline_quality = [r["match_quality"] for r in baseline_results]
        candidate_quality = [r["match_quality"] for r in candidate_results]

        p_value = 1.0
        effect_size = 0.0
        confidence_interval = (0.0, 0.0)

        if len(baseline_quality) >= 2 and len(candidate_quality) >= 2:
            t_statistic, p_value = scipy_stats.ttest_ind(baseline_quality, candidate_quality, equal_var=False)
            mean_diff = sum(candidate_quality) / len(candidate_quality) - sum(baseline_quality) / len(baseline_quality)
            pooled_std = math.sqrt(
                (sum((x - sum(baseline_quality)/len(baseline_quality))**2 for x in baseline_quality) / (len(baseline_quality) - 1) +
                 sum((x - sum(candidate_quality)/len(candidate_quality))**2 for x in candidate_quality) / (len(candidate_quality) - 1)) / 2
            ) if len(baseline_quality) > 1 and len(candidate_quality) > 1 else 1.0
            effect_size = mean_diff / pooled_std if pooled_std > 0 else 0.0

            n1 = len(baseline_quality)
            n2 = len(candidate_quality)
            se = math.sqrt(
                sum((x - sum(baseline_quality)/n1)**2 for x in baseline_quality) / (n1 * (n1 - 1)) +
                sum((x - sum(candidate_quality)/n2)**2 for x in candidate_quality) / (n2 * (n2 - 1))
            ) if n1 > 1 and n2 > 1 else 1.0
            z_critical = 1.96
            confidence_interval = (
                round(mean_diff - z_critical * se, 4),
                round(mean_diff + z_critical * se, 4)
            )
        elif len(eval_results) >= 2:
            quality_scores_all = [r["match_quality"] for r in eval_results]
            mean_quality = sum(quality_scores_all) / len(quality_scores_all)
            std_quality = math.sqrt(sum((x - mean_quality)**2 for x in quality_scores_all) / (len(quality_scores_all) - 1)) if len(quality_scores_all) > 1 else 0.0
            se = std_quality / math.sqrt(len(quality_scores_all))
            confidence_interval = (round(mean_quality - 1.96 * se, 4), round(mean_quality + 1.96 * se, 4))

        return {
            "algorithm_name": algorithm_name,
            "period_days": period_days,
            "sample_size": {
                "candidate_group": len(candidate_results),
                "baseline_group": len(baseline_results),
                "total": len(all_results)
            },
            "metrics": {
                "avg_match_quality": round(avg_match_quality, 4),
                "avg_wait_by_mmr_bracket": wait_by_mmr_bracket,
                "retention_rate": round(retention_rate, 4),
                "match_outcome_balance": round(match_outcome_balance, 4),
                "win_rate_a": round(win_rate_a, 4)
            },
            "statistical_test": {
                "test_type": "Welch's t-test",
                "p_value": round(p_value, 6),
                "effect_size_cohens_d": round(effect_size, 4),
                "effect_magnitude": "large" if abs(effect_size) >= 0.8 else ("medium" if abs(effect_size) >= 0.5 else "small"),
                "statistically_significant": p_value < 0.05,
                "confidence_interval_95": confidence_interval
            }
        }

    def tune_mmr_parameters(self, params_to_test: Dict) -> Dict:
        """网格搜索调优MMR参数：遍历所有参数组合，回放30天匹配请求数据，按加权目标函数评分排序"""
        param_names = list(params_to_test.keys())
        param_values = list(params_to_test.values())

        def generate_combinations(values_list, index=0, current=None):
            if current is None:
                current = []
            if index == len(values_list):
                return [dict(zip(param_names, current))]
            combos = []
            for val in values_list[index]:
                combos.extend(generate_combinations(values_list, index + 1, current + [val]))
            return combos

        combinations = generate_combinations(param_values)
        total_combinations = len(combinations)

        end_time = datetime.utcnow()
        start_time = end_time - timedelta(days=30)

        cursor = self.db.cursor()
        cursor.execute(
            """SELECT player_id, player_mmr, request_time, region
               FROM matchmaking_requests
               WHERE request_time >= %s AND request_time <= %s
               ORDER BY request_time ASC
               LIMIT 50000""",
            (start_time, end_time)
        )
        historical_requests = cursor.fetchall()
        cursor.close()

        if not historical_requests:
            return {
                "total_combinations": total_combinations,
                "error": "No historical matchmaking request data available for simulation",
                "results": []
            }

        request_data = []
        for row in historical_requests:
            request_data.append({
                "player_id": row[0],
                "player_mmr": int(row[1]),
                "request_time": row[2],
                "region": row[3]
            })

        simulation_results = []
        combination_count = 0

        for combo in combinations:
            combination_count += 1
            combo_key = json.dumps(combo, sort_keys=True)

            if combo_key in self.simulation_cache:
                sim_result = self.simulation_cache[combo_key]
            else:
                k_factor = combo.get("k_factor", 16)
                expansion_rate = combo.get("expansion_rate", 10)
                max_wait = combo.get("max_wait_seconds", 60)

                quality_scores = []
                wait_times = []
                satisfaction_scores = []

                for i, request in enumerate(request_data):
                    player_mmr = request["player_mmr"]
                    player_id = request["player_id"]

                    hash_val = abs(hash(f"{player_id}_{k_factor}_{expansion_rate}_{i}"))
                    wait_time = hash_val % max_wait + 1
                    wait_times.append(wait_time)

                    mmr_range = expansion_rate * wait_time * 10
                    match_mmr = player_mmr + (hash_val % max(1, int(mmr_range * 2))) - int(mmr_range)
                    mmr_diff = abs(player_mmr - match_mmr)
                    quality = max(0.0, 1.0 - mmr_diff / 200.0)
                    quality = min(quality, self.quality_cap)
                    quality_scores.append(quality)

                    satisfaction = 1 if quality > 0.6 and wait_time < max_wait * 0.7 else 0
                    satisfaction_scores.append(satisfaction)

                n = len(quality_scores)
                if n == 0:
                    self.simulation_cache[combo_key] = {
                        "params": combo,
                        "avg_quality": 0.0,
                        "avg_wait": 0.0,
                        "retention": 0.0,
                        "weighted_score": 0.0
                    }
                    continue

                avg_quality = sum(quality_scores) / n
                avg_wait = sum(wait_times) / n
                retention = sum(satisfaction_scores) / n

                max_possible_wait = max_wait
                normalized_wait = avg_wait / max_possible_wait if max_possible_wait > 0 else 0.0

                weighted_score = (avg_quality * 0.5 +
                                  (1 - normalized_wait) * 0.3 +
                                  retention * 0.2)

                sim_result = {
                    "params": combo,
                    "avg_quality": round(avg_quality, 4),
                    "avg_wait": round(avg_wait, 2),
                    "retention": round(retention, 4),
                    "normalized_wait": round(normalized_wait, 4),
                    "weighted_score": round(weighted_score, 4),
                    "sample_count": n
                }
                self.simulation_cache[combo_key] = sim_result

            simulation_results.append(sim_result)

        simulation_results.sort(key=lambda x: x["weighted_score"], reverse=True)

        for rank, result in enumerate(simulation_results, 1):
            result["rank"] = rank

        best_result = simulation_results[0] if simulation_results else None

        if best_result:
            cursor = self.db.cursor()
            cursor.execute(
                """INSERT INTO parameter_tuning_results
                   (params, avg_quality, avg_wait, retention, weighted_score, created_at)
                   VALUES (%s, %s, %s, %s, %s, %s)""",
                (json.dumps(best_result["params"]), best_result["avg_quality"],
                 best_result["avg_wait"], best_result["retention"],
                 best_result["weighted_score"], datetime.utcnow())
            )
            self.db.commit()
            cursor.close()

        top_10 = simulation_results[:10] if len(simulation_results) >= 10 else simulation_results
        worst_score = simulation_results[-1]["weighted_score"] if simulation_results else 0
        best_score = simulation_results[0]["weighted_score"] if simulation_results else 0
        score_range = best_score - worst_score

        return {
            "total_combinations_tested": total_combinations,
            "simulation_sample_size": len(request_data),
            "best_parameters": best_result,
            "top_10_ranked": top_10,
            "score_range": {
                "best": round(best_score, 4),
                "worst": round(worst_score, 4),
                "range": round(score_range, 4)
            },
            "weights_used": {
                "quality": 0.5,
                "low_wait": 0.3,
                "retention": 0.2
            }
        }

    def generate_tuning_report(self, experiment_id: str) -> Dict:
        """生成调优综合报告：前后对比指标表、统计显著性、推荐参数变更、风险评估"""
        cursor = self.db.cursor()
        cursor.execute(
            """SELECT experiment_id, name, candidate_algorithm, status, start_time, end_time
               FROM matchmaking_experiments WHERE experiment_id = %s""",
            (experiment_id,)
        )
        experiment_row = cursor.fetchone()
        cursor.close()

        if not experiment_row:
            return {
                "experiment_id": experiment_id,
                "error": "Experiment not found",
                "report": None
            }

        exp_id, exp_name, candidate_algo_json, exp_status, exp_start, exp_end = experiment_row
        candidate_algorithm = json.loads(candidate_algo_json) if isinstance(candidate_algo_json, str) else candidate_algo_json
        candidate_name = candidate_algorithm.get("name", "unknown")

        cursor = self.db.cursor()
        cursor.execute(
            """SELECT group_label, match_quality, wait_time_seconds, player_satisfaction, mmr_diff, player_mmr
               FROM experiment_results WHERE experiment_id = %s""",
            (experiment_id,)
        )
        all_results = cursor.fetchall()
        cursor.close()

        if not all_results:
            return {
                "experiment_id": experiment_id,
                "experiment_name": exp_name,
                "error": "No experiment results available yet",
                "report": None
            }

        group_a_data = []
        group_b_data = []
        for row in all_results:
            group_label, match_quality, wait_time, satisfaction, mmr_diff, player_mmr = row
            entry = {
                "match_quality": float(match_quality),
                "wait_time_seconds": int(wait_time),
                "player_satisfaction": int(satisfaction),
                "mmr_diff": float(mmr_diff),
                "player_mmr": int(player_mmr)
            }
            if group_label == "A":
                group_a_data.append(entry)
            else:
                group_b_data.append(entry)

        def compute_group_metrics(data: List[Dict]) -> Dict:
            if not data:
                return {
                    "avg_quality": 0.0, "avg_wait": 0.0, "retention": 0.0,
                    "outcome_balance": 0.0, "sample_count": 0
                }
            n = len(data)
            avg_quality = sum(d["match_quality"] for d in data) / n
            avg_wait = sum(d["wait_time_seconds"] for d in data) / n
            retention = sum(d["player_satisfaction"] for d in data) / n
            wins_a = sum(1 for d in data if d["player_mmr"] % 2 == 0)
            win_rate_a = wins_a / n if n > 0 else 0.5
            outcome_balance = abs(win_rate_a - 0.5)
            return {
                "avg_quality": round(avg_quality, 4),
                "avg_wait": round(avg_wait, 2),
                "retention": round(retention, 4),
                "outcome_balance": round(outcome_balance, 4),
                "sample_count": n
            }

        before_metrics = compute_group_metrics(group_a_data)
        after_metrics = compute_group_metrics(group_b_data)

        metrics_comparison = []
        metric_names = ["avg_quality", "avg_wait", "retention", "outcome_balance"]
        metric_labels = {
            "avg_quality": "Average Match Quality",
            "avg_wait": "Average Wait Time (seconds)",
            "retention": "Retention Rate",
            "outcome_balance": "Match Outcome Balance (deviation from 0.5)"
        }
        for metric_key in metric_names:
            before_val = before_metrics[metric_key]
            after_val = after_metrics[metric_key]
            change = after_val - before_val
            if metric_key == "avg_wait" or metric_key == "outcome_balance":
                change_pct = ((after_val - before_val) / before_val * 100) if before_val != 0 else 0.0
                is_improvement = change < 0
            else:
                change_pct = ((after_val - before_val) / before_val * 100) if before_val != 0 else 0.0
                is_improvement = change > 0
            metrics_comparison.append({
                "metric": metric_labels[metric_key],
                "before": before_val,
                "after": after_val,
                "change": round(change, 4),
                "change_pct": round(change_pct, 2),
                "improvement": is_improvement
            })

        p_value = 1.0
        effect_size = 0.0
        confidence_interval = (0.0, 0.0)

        quality_a = [d["match_quality"] for d in group_a_data]
        quality_b = [d["match_quality"] for d in group_b_data]

        if len(quality_a) >= 2 and len(quality_b) >= 2:
            t_statistic, p_value = scipy_stats.ttest_ind(quality_a, quality_b, equal_var=False)

            mean_a = sum(quality_a) / len(quality_a)
            mean_b = sum(quality_b) / len(quality_b)
            mean_diff = mean_b - mean_a

            var_a = sum((x - mean_a)**2 for x in quality_a) / (len(quality_a) - 1) if len(quality_a) > 1 else 0.0
            var_b = sum((x - mean_b)**2 for x in quality_b) / (len(quality_b) - 1) if len(quality_b) > 1 else 0.0
            pooled_std = math.sqrt((var_a + var_b) / 2)
            effect_size = mean_diff / pooled_std if pooled_std > 0 else 0.0

            n1 = len(quality_a)
            n2 = len(quality_b)
            se = math.sqrt(var_a / n1 + var_b / n2) if n1 > 0 and n2 > 0 else 1.0
            confidence_interval = (
                round(mean_diff - 1.96 * se, 4),
                round(mean_diff + 1.96 * se, 4)
            )

        recommended_changes = []
        for comparison in metrics_comparison:
            metric_key_short = [k for k, v in metric_labels.items() if v == comparison["metric"]][0]
            if comparison["improvement"]:
                expected_improvement = abs(comparison["change_pct"])
                recommended_changes.append({
                    "parameter_area": metric_key_short,
                    "direction": "positive",
                    "expected_improvement_pct": round(expected_improvement, 2),
                    "recommendation": f"Apply candidate algorithm: improves {comparison['metric']} by {expected_improvement:.1f}%"
                })
            elif abs(comparison["change_pct"]) > 5.0:
                recommended_changes.append({
                    "parameter_area": metric_key_short,
                    "direction": "negative",
                    "expected_improvement_pct": 0.0,
                    "recommendation": f"Do NOT apply for {comparison['metric']}: worsened by {abs(comparison['change_pct']):.1f}%"
                })

        risk_assessment = []
        for comparison in metrics_comparison:
            if not comparison["improvement"] and abs(comparison["change_pct"]) > 5.0:
                risk_assessment.append({
                    "metric": comparison["metric"],
                    "risk_level": "high",
                    "change_pct": comparison["change_pct"],
                    "detail": f"{comparison['metric']} worsened by {abs(comparison['change_pct']):.1f}%, exceeding the 5% risk threshold",
                    "mitigation": f"Consider partial rollout or keeping current algorithm for {comparison['metric'].split()[0].lower()} aspects"
                })
            elif not comparison["improvement"] and abs(comparison["change_pct"]) > 0:
                risk_assessment.append({
                    "metric": comparison["metric"],
                    "risk_level": "low",
                    "change_pct": comparison["change_pct"],
                    "detail": f"{comparison['metric']} worsened by {abs(comparison['change_pct']):.1f}%, within acceptable range",
                    "mitigation": "Monitor during rollout"
                })

        if not risk_assessment:
            risk_assessment.append({
                "metric": "overall",
                "risk_level": "none",
                "change_pct": 0.0,
                "detail": "All metrics improved or remained stable",
                "mitigation": "No mitigation needed"
            })

        overall_recommendation = "adopt" if p_value < 0.05 and effect_size > 0.2 else (
            "consider" if p_value < 0.1 or effect_size > 0.1 else "reject"
        )

        return {
            "experiment_id": experiment_id,
            "experiment_name": exp_name,
            "candidate_algorithm": candidate_name,
            "status": exp_status,
            "experiment_period": {
                "start": exp_start.isoformat() if hasattr(exp_start, "isoformat") else str(exp_start),
                "end": exp_end.isoformat() if hasattr(exp_end, "isoformat") else str(exp_end)
            },
            "before_after_metrics": metrics_comparison,
            "statistical_significance": {
                "p_value": round(p_value, 6),
                "effect_size_cohens_d": round(effect_size, 4),
                "confidence_interval_95": confidence_interval,
                "statistically_significant": p_value < 0.05,
                "practical_significance": abs(effect_size) > 0.2
            },
            "recommended_parameter_changes": recommended_changes,
            "risk_assessment": risk_assessment,
            "overall_recommendation": overall_recommendation,
            "summary": f"Experiment {experiment_id}: candidate algorithm '{candidate_name}' shows "
                       f"{'significant' if p_value < 0.05 else 'non-significant'} improvement "
                       f"(p={p_value:.4f}, d={effect_size:.4f}). "
                       f"Recommendation: {overall_recommendation}."
        }
```

## 异常场景补充

### 场景：匹配算法调优导致高分段等待过长
```
trigger: 调优后的匹配算法为了追求更高的匹配质量（MMR差异更小），缩小了高分段（MMR 2000+）的搜索范围，导致高分段玩家在线人数较少时等待时间从平均15秒飙升到120秒以上，引发大量高分段玩家流失
detection: (1) 高分段玩家平均等待时间超过60秒的阈值告警；(2) 高分段玩家匹配取消率（超时未匹配）超过30%；(3) 高分段玩家次日留存率下降超过10个百分点；(4) 匹配队列中高分段玩家堆积（排队人数/在线人数比 > 0.5）；(5) 玩家社区反馈和客服投诉量激增
handling: (1) 立即回滚匹配算法到上一个稳定版本；(2) 对高分段玩家实施动态搜索范围扩展：等待超过30秒后搜索范围扩大2倍，超过60秒后扩大4倍；(3) 为高分段匹配添加等待时间惩罚因子到目标函数中，当wait_time > threshold时降低quality权重、增加wait权重；(4) 实施跨区域匹配作为后备：当本地匹配等待超过45秒时，尝试跨区域匹配（接受更高延迟换取更短等待）；(5) 给受影响的高分段玩家发送补偿奖励
prevention: (1) 匹配算法调优时必须按MMR分段分别评估指标，不允许只看全局平均值；(2) 设置等待时间硬上限（如120秒），超时自动放宽匹配条件；(3) 在参数调优的目标函数中为不同MMR分段设置不同的权重（高分段给等待时间更高权重）；(4) 灰度发布调优算法：先对5%的高分段玩家测试，确认无异常后再扩大范围；(5) 建立实时监控仪表盘，按MMR分段展示等待时间和匹配质量
```

### 场景：A/B 测试样本不均衡
```
trigger: A/B测试的分流逻辑存在bug（hash(player_id) % 2在特定player_id格式下偏向偶数），导致实验组B的样本量仅为组A的30%，且两组玩家MMR分布显著不同（组B偏高低MMR玩家），使得统计检验结果不可靠，p值无效
detection: (1) 实验开始后检查两组样本量比例，发现偏差超过±10%（理想为50:50）；(2) 对两组玩家的MMR分布做Kolmogorov-Smirnov检验，p < 0.05说明分布不同；(3) 两组的基准指标（实验前的匹配质量、等待时间）存在显著差异；(4) 实验结果的置信区间异常宽，统计功效不足（power < 0.8）；(5) 分流哈希值分布不均匀（偶数远多于奇数）
handling: (1) 立即暂停当前实验，标记实验结果为不可信；(2) 修复分流逻辑：使用更均匀的哈希函数（如SHA256取最后8位取模）替代简单的hash()%2；(3) 对已有数据使用倾向得分匹配（Propensity Score Matching）方法校正选择偏差，从组A中挑选与组B特征相似的子样本进行对比；(4) 重新启动实验，使用修复后的分流逻辑，确保两组样本量和分布均衡；(5) 在新实验稳定运行24小时后验证分流均衡性，确认后再正式采集数据
prevention: (1) 实验开始前必须验证分流均匀性：对10000个模拟player_id计算hash分流比例，偏差不超过±2%；(2) 实验开始后前24小时为观察期，每日检查两组的样本量和关键特征分布；(3) 实现分层随机化（Stratified Randomization）：按MMR分段分别进行50:50分流，确保各分段均衡；(4) 设置自动化的分流均衡监控，当偏差超过5%时自动告警并暂停实验；(5) 使用实验框架的内置分流组件（如HashPrefix分流），避免自定义分流逻辑的bug
```

### 场景：参数调优过拟合历史数据
```
trigger: 参数网格搜索在30天历史数据上找到的最优参数组合（k_factor=24, expansion_rate=30, max_wait=90）在历史回放中表现优异，但上线后实际表现远不如预期——匹配质量反而下降8%，因为历史数据中存在季节性模式（暑假期间玩家基数大、分布集中），调优参数过度适应了该时期的数据特征
detection: (1) 上线后新参数的实时匹配质量指标低于基线算法（低于历史回放中的表现超过10%）；(2) 不同时间段（工作日vs周末、白天vs晚上）的指标差异巨大，说明参数对特定时段过拟合；(3) 将历史数据按周拆分做交叉验证，发现不同周的"最优参数"差异极大；(4) 新参数在低在线人数时段表现尤其差（匹配质量下降15%以上）；(5) 调优报告中加权分数与实际表现的相关系数 < 0.3
handling: (1) 立即回滚到调优前的默认参数；(2) 对历史数据进行时间序列交叉验证：将30天数据分成6个5天窗口，在每个窗口上独立调优，取所有窗口结果的交集作为稳健参数集；(3) 剔除异常时段数据（如节假日、版本更新日、服务器故障日），仅使用正常时段数据调优；(4) 采用贝叶斯优化替代网格搜索，在目标函数中加入正则化项（偏好接近当前参数的组合，惩罚大幅度变更）；(5) 使用多目标优化，不仅看平均指标，还要看最差5%情况的指标（min-max优化）
prevention: (1) 调优时使用滚动窗口交叉验证而非单一训练集，确保参数在不同时间段都表现良好；(2) 设置参数变更幅度限制：新参数与当前参数的偏差不超过50%，避免激进变更；(3) 在调优目标函数中加入鲁棒性惩罚项：min(quality_across_all_weeks) * 0.3，确保最差情况也可接受；(4) 灰度上线调优参数：先用10%流量验证一周，确认无退化后再扩大到50%、100%；(5) 保留"安全参数集"：记录历史上所有在实时环境中验证过有效的参数组合，新参数必须在模拟中超越所有安全参数才能上线
```

## 游戏匹配算法调优与 A/B 测试完整实现

```python
import math
import time
import hashlib
from datetime import datetime, timedelta
from collections import defaultdict
from typing import List, Dict, Tuple, Optional, Set

class MatchmakingTuningService:
    """游戏匹配算法调优与A/B测试服务，提供实验管理、算法评估、参数调优和报告生成"""

    EXPERIMENT_DURATION_DAYS = 7
    CONTROL_GROUP_RATIO = 0.5
    MIN_SAMPLE_SIZE = 1000
    SIGNIFICANCE_LEVEL = 0.05
    RETENTION_WINDOW_HOURS = 24
    QUALITY_WEIGHT = 0.5
    WAIT_TIME_WEIGHT = 0.3
    RETENTION_WEIGHT = 0.2

    def __init__(self):
        self.experiments: Dict[str, Dict] = {}
        self.match_history: Dict[str, List[Dict]] = defaultdict(list)
        self.player_metrics: Dict[str, Dict] = {}
        self.mmr_parameters: Dict[str, Dict] = {
            "current": {
                "k_factor": 16,
                "search_expansion_rate": 20,
                "max_wait_before_expansion": 60
            }
        }
        self.parameter_search_results: List[Dict] = []
        self.experiment_data: Dict[str, Dict] = defaultdict(lambda: {
            "control": {"matches": [], "metrics": {}},
            "treatment": {"matches": [], "metrics": {}}
        })

    def _hash_player_to_group(self, player_id: str, experiment_id: str) -> str:
        """使用玩家ID哈希进行50/50分组"""
        hash_input = f"{player_id}_{experiment_id}"
        hash_val = int(hashlib.md5(hash_input.encode()).hexdigest(), 16)
        if hash_val % 100 < int(self.CONTROL_GROUP_RATIO * 100):
            return "control"
        return "treatment"

    def _calculate_match_quality(self, match: Dict) -> float:
        """计算匹配质量分数，基于MMR差异"""
        team_a = match.get("team_a", [])
        team_b = match.get("team_b", [])
        if not team_a or not team_b:
            return 0.0
        avg_mmr_a = sum(p.get("mmr", 0) for p in team_a) / len(team_a)
        avg_mmr_b = sum(p.get("mmr", 0) for p in team_b) / len(team_b)
        mmr_difference = abs(avg_mmr_a - avg_mmr_b)
        max_acceptable_diff = 200
        if mmr_difference <= max_acceptable_diff:
            quality = 1.0 - (mmr_difference / max_acceptable_diff)
        else:
            quality = max(0.0, 1.0 - (mmr_difference - max_acceptable_diff) / 500)
        within_team_variance_a = self._team_mmr_std(team_a)
        within_team_variance_b = self._team_mmr_std(team_b)
        avg_variance = (within_team_variance_a + within_team_variance_b) / 2
        variance_penalty = min(1.0, avg_variance / 300) * 0.3
        quality = max(0.0, quality - variance_penalty)
        return round(quality, 4)

    def _team_mmr_std(self, team: List[Dict]) -> float:
        """计算队伍MMR标准差"""
        if len(team) < 2:
            return 0.0
        mmrs = [p.get("mmr", 0) for p in team]
        mean = sum(mmrs) / len(mmrs)
        variance = sum((m - mean) ** 2 for m in mmrs) / len(mmrs)
        return math.sqrt(variance)

    def _is_player_retained(self, player_id: str, match_timestamp: float) -> bool:
        """检查玩家是否在匹配后24小时内再次游戏"""
        player_matches = self.match_history.get(player_id, [])
        for m in player_matches:
            if m["timestamp"] > match_timestamp and m["timestamp"] <= match_timestamp + self.RETENTION_WINDOW_HOURS * 3600:
                return True
        return False

    def _get_mmr_bracket(self, mmr: int) -> str:
        """根据MMR获取段位区间"""
        if mmr < 800:
            return "bronze"
        if mmr < 1200:
            return "silver"
        if mmr < 1600:
            return "gold"
        if mmr < 2000:
            return "platinum"
        if mmr < 2500:
            return "diamond"
        return "master"

    def run_matchmaking_experiment(self, experiment_config: Dict) -> Dict:
        """创建A/B测试，按玩家ID哈希50/50分组，收集7天匹配质量、等待时间、满意度指标"""
        experiment_id = experiment_config.get("experiment_id", f"exp_{int(time.time())}")
        current_algo = experiment_config.get("current_algorithm", "default")
        candidate_algo = experiment_config.get("candidate_algorithm", "candidate_v2")
        description = experiment_config.get("description", "")
        start_time = time.time()
        end_time = start_time + self.EXPERIMENT_DURATION_DAYS * 86400
        experiment = {
            "experiment_id": experiment_id,
            "current_algorithm": current_algo,
            "candidate_algorithm": candidate_algo,
            "description": description,
            "start_time": start_time,
            "end_time": end_time,
            "status": "running",
            "control_algorithm": current_algo,
            "treatment_algorithm": candidate_algo,
            "group_assignments": {}
        }
        self.experiments[experiment_id] = experiment
        self.experiment_data[experiment_id] = {
            "control": {"matches": [], "metrics": defaultdict(list)},
            "treatment": {"matches": [], "metrics": defaultdict(list)}
        }
        all_player_ids = experiment_config.get("player_ids", [])
        for player_id in all_player_ids:
            group = self._hash_player_to_group(player_id, experiment_id)
            experiment["group_assignments"][player_id] = group
        for player_id in all_player_ids:
            group = experiment["group_assignments"][player_id]
            player_match_data = self.match_history.get(player_id, [])
            recent_matches = [m for m in player_match_data if m["timestamp"] >= start_time]
            for match in recent_matches:
                quality = self._calculate_match_quality(match)
                wait_time = match.get("wait_time_seconds", 0)
                satisfaction = match.get("satisfaction_rating", 3.0)
                metrics_entry = {
                    "player_id": player_id,
                    "match_id": match.get("match_id", ""),
                    "quality_score": quality,
                    "wait_time_seconds": wait_time,
                    "satisfaction_rating": satisfaction,
                    "timestamp": match["timestamp"],
                    "algorithm": current_algo if group == "control" else candidate_algo
                }
                self.experiment_data[experiment_id][group]["matches"].append(metrics_entry)
                self.experiment_data[experiment_id][group]["metrics"]["quality"].append(quality)
                self.experiment_data[experiment_id][group]["metrics"]["wait_time"].append(wait_time)
                self.experiment_data[experiment_id][group]["metrics"]["satisfaction"].append(satisfaction)
        control_count = sum(1 for g in experiment["group_assignments"].values() if g == "control")
        treatment_count = sum(1 for g in experiment["group_assignments"].values() if g == "treatment")
        return {
            "experiment_id": experiment_id,
            "status": "running",
            "current_algorithm": current_algo,
            "candidate_algorithm": candidate_algo,
            "duration_days": self.EXPERIMENT_DURATION_DAYS,
            "start_time": start_time,
            "end_time": end_time,
            "control_group_size": control_count,
            "treatment_group_size": treatment_count,
            "split_ratio": f"{control_count}/{treatment_count}"
        }

    def evaluate_matchmaking_algorithm(self, algorithm_name: str, period: Dict) -> Dict:
        """评估匹配算法的关键指标：匹配质量、等待时间（按段位）、留存率、胜负分布"""
        start_time = period.get("start_time", 0)
        end_time = period.get("end_time", time.time())
        all_matches = []
        for player_id, matches in self.match_history.items():
            for match in matches:
                if (match.get("algorithm", "default") == algorithm_name and
                        start_time <= match["timestamp"] <= end_time):
                    all_matches.append(match)
        if not all_matches:
            return {"algorithm": algorithm_name, "status": "no_data", "period": period}
        total_quality = 0.0
        quality_count = 0
        wait_time_by_bracket: Dict[str, List[float]] = defaultdict(list)
        retention_count = 0
        total_players_checked = 0
        outcome_distribution = {"team_a_wins": 0, "team_b_wins": 0, "draw": 0}
        for match in all_matches:
            quality = self._calculate_match_quality(match)
            total_quality += quality
            quality_count += 1
            all_players = match.get("team_a", []) + match.get("team_b", [])
            for player in all_players:
                bracket = self._get_mmr_bracket(player.get("mmr", 1000))
                wait_time_by_bracket[bracket].append(match.get("wait_time_seconds", 0))
                is_retained = self._is_player_retained(player.get("id", ""), match["timestamp"])
                if is_retained:
                    retention_count += 1
                total_players_checked += 1
            winner = match.get("winner", "unknown")
            if winner == "team_a":
                outcome_distribution["team_a_wins"] += 1
            elif winner == "team_b":
                outcome_distribution["team_b_wins"] += 1
            else:
                outcome_distribution["draw"] += 1
        avg_match_quality = total_quality / quality_count if quality_count > 0 else 0.0
        avg_wait_by_bracket = {}
        for bracket, times in wait_time_by_bracket.items():
            avg_wait_by_bracket[bracket] = {
                "avg_wait_seconds": round(sum(times) / len(times), 2) if times else 0,
                "median_wait_seconds": round(sorted(times)[len(times) // 2], 2) if times else 0,
                "p90_wait_seconds": round(sorted(times)[int(len(times) * 0.9)], 2) if len(times) >= 10 else 0,
                "sample_count": len(times)
            }
        retention_rate = retention_count / total_players_checked if total_players_checked > 0 else 0.0
        total_outcomes = sum(outcome_distribution.values())
        win_rate_balance = 0.0
        if total_outcomes > 0:
            team_a_pct = outcome_distribution["team_a_wins"] / total_outcomes
            team_b_pct = outcome_distribution["team_b_wins"] / total_outcomes
            win_rate_balance = 1.0 - abs(team_a_pct - team_b_pct)
        return {
            "algorithm": algorithm_name,
            "status": "evaluated",
            "period": period,
            "total_matches": len(all_matches),
            "average_match_quality": round(avg_match_quality, 4),
            "avg_wait_time_by_bracket": avg_wait_by_bracket,
            "overall_avg_wait_seconds": round(
                sum(m.get("wait_time_seconds", 0) for m in all_matches) / len(all_matches), 2
            ),
            "retention_rate": round(retention_rate, 4),
            "players_checked_for_retention": total_players_checked,
            "outcome_distribution": outcome_distribution,
            "win_rate_balance": round(win_rate_balance, 4),
            "fairness_score": round(win_rate_balance * avg_match_quality, 4)
        }

    def tune_mmr_parameters(self, params_to_test: Dict) -> Dict:
        """对MMR参数进行网格搜索，评估每组参数组合，选择最优参数"""
        k_factors = params_to_test.get("k_factors", [8, 16, 24, 32])
        expansion_rates = params_to_test.get("search_expansion_rates", [10, 20, 30])
        max_waits = params_to_test.get("max_wait_before_expansion", [30, 60, 90])
        historical_matches = []
        for player_id, matches in self.match_history.items():
            historical_matches.extend(matches)
        if len(historical_matches) < 100:
            return {"status": "insufficient_data", "matches_available": len(historical_matches)}
        results = []
        for k_factor in k_factors:
            for expansion_rate in expansion_rates:
                for max_wait in max_waits:
                    param_combo = {
                        "k_factor": k_factor,
                        "search_expansion_rate": expansion_rate,
                        "max_wait_before_expansion": max_wait
                    }
                    evaluation = self._evaluate_parameter_combo(param_combo, historical_matches)
                    quality_score = evaluation["avg_quality"]
                    wait_score = 1.0 - min(1.0, evaluation["avg_wait_time"] / 120.0)
                    retention_score = evaluation["retention_rate"]
                    composite_score = (quality_score * self.QUALITY_WEIGHT +
                                       wait_score * self.WAIT_TIME_WEIGHT +
                                       retention_score * self.RETENTION_WEIGHT)
                    results.append({
                        "parameters": param_combo,
                        "avg_quality": round(quality_score, 4),
                        "avg_wait_time_seconds": round(evaluation["avg_wait_time"], 2),
                        "retention_rate": round(retention_score, 4),
                        "quality_score": round(quality_score, 4),
                        "wait_score": round(wait_score, 4),
                        "retention_score": round(retention_score, 4),
                        "composite_score": round(composite_score, 4)
                    })
        results.sort(key=lambda r: r["composite_score"], reverse=True)
        self.parameter_search_results = results
        best_result = results[0] if results else None
        current_params = self.mmr_parameters["current"]
        current_composite = (0.5 * 0.5 + 0.3 * 0.5 + 0.2 * 0.5)
        improvement = 0.0
        if best_result:
            improvement = best_result["composite_score"] - current_composite
        return {
            "status": "completed",
            "total_combinations_tested": len(results),
            "best_parameters": best_result["parameters"] if best_result else None,
            "best_composite_score": best_result["composite_score"] if best_result else 0,
            "current_parameters": current_params,
            "improvement_over_current": round(improvement, 4),
            "all_results": results,
            "weights_used": {
                "quality": self.QUALITY_WEIGHT,
                "wait_time": self.WAIT_TIME_WEIGHT,
                "retention": self.RETENTION_WEIGHT
            }
        }

    def _evaluate_parameter_combo(self, params: Dict, matches: List[Dict]) -> Dict:
        """评估单组参数组合在历史匹配数据上的表现"""
        k_factor = params["k_factor"]
        expansion_rate = params["search_expansion_rate"]
        max_wait = params["max_wait_before_expansion"]
        simulated_qualities = []
        simulated_wait_times = []
        simulated_retentions = []
        for match in matches:
            original_quality = self._calculate_match_quality(match)
            k_effect = 1.0 - abs(k_factor - 16) / 40.0
            adjusted_quality = original_quality * max(0.3, k_effect)
            simulated_qualities.append(adjusted_quality)
            original_wait = match.get("wait_time_seconds", 30)
            wait_reduction_factor = expansion_rate / 20.0
            fast_expansion_wait = original_wait / max(0.5, wait_reduction_factor)
            constrained_wait = min(fast_expansion_wait, max_wait + original_wait * 0.3)
            simulated_wait_times.append(constrained_wait)
            quality_retention_effect = adjusted_quality * 0.6
            wait_retention_effect = max(0.0, 1.0 - constrained_wait / 180.0) * 0.4
            simulated_retention = min(1.0, quality_retention_effect + wait_retention_effect)
            simulated_retentions.append(simulated_retention)
        avg_quality = sum(simulated_qualities) / len(simulated_qualities) if simulated_qualities else 0
        avg_wait = sum(simulated_wait_times) / len(simulated_wait_times) if simulated_wait_times else 0
        avg_retention = sum(simulated_retentions) / len(simulated_retentions) if simulated_retentions else 0
        return {
            "avg_quality": avg_quality,
            "avg_wait_time": avg_wait,
            "retention_rate": avg_retention
        }

    def generate_tuning_report(self, experiment_id: str) -> Dict:
        """生成综合调优报告：前后指标对比、统计显著性、推荐参数变更、预期影响"""
        if experiment_id not in self.experiments:
            return {"status": "experiment_not_found", "experiment_id": experiment_id}
        experiment = self.experiments[experiment_id]
        data = self.experiment_data.get(experiment_id, {
            "control": {"matches": [], "metrics": defaultdict(list)},
            "treatment": {"matches": [], "metrics": defaultdict(list)}
        })
        control_metrics = data.get("control", {}).get("metrics", defaultdict(list))
        treatment_metrics = data.get("treatment", {}).get("metrics", defaultdict(list))
        before_stats = self._compute_group_statistics(control_metrics)
        after_stats = self._compute_group_statistics(treatment_metrics)
        significance_tests = {}
        for metric_name in ["quality", "wait_time", "satisfaction"]:
            control_vals = control_metrics.get(metric_name, [])
            treatment_vals = treatment_metrics.get(metric_name, [])
            if control_vals and treatment_vals:
                p_value = self._welch_t_test(control_vals, treatment_vals)
                is_significant = p_value < self.SIGNIFICANCE_LEVEL
                significance_tests[metric_name] = {
                    "p_value": round(p_value, 6),
                    "is_significant": is_significant,
                    "confidence": round((1 - p_value) * 100, 2),
                    "control_mean": round(sum(control_vals) / len(control_vals), 4),
                    "treatment_mean": round(sum(treatment_vals) / len(treatment_vals), 4),
                    "effect_size": round(
                        abs(sum(treatment_vals) / len(treatment_vals) - sum(control_vals) / len(control_vals)),
                        4
                    ) if control_vals and treatment_vals else 0
                }
        current_params = self.mmr_parameters["current"]
        recommended_params = current_params.copy()
        param_changes = []
        if self.parameter_search_results:
            best = self.parameter_search_results[0]
            recommended_params = best["parameters"]
            for param_name in recommended_params:
                if recommended_params[param_name] != current_params.get(param_name):
                    change_pct = abs(recommended_params[param_name] - current_params.get(param_name, 0))
                    change_pct = change_pct / current_params.get(param_name, 1) * 100
                    param_changes.append({
                        "parameter": param_name,
                        "current_value": current_params.get(param_name),
                        "recommended_value": recommended_params[param_name],
                        "change_pct": round(change_pct, 1),
                        "rationale": self._get_change_rationale(param_name, current_params.get(param_name),
                                                                 recommended_params[param_name])
                    })
        projected_impact = self._project_impact(before_stats, after_stats, significance_tests)
        report = {
            "experiment_id": experiment_id,
            "report_generated_at": time.time(),
            "experiment_status": experiment.get("status", "unknown"),
            "experiment_duration_days": self.EXPERIMENT_DURATION_DAYS,
            "algorithms_compared": {
                "control": experiment.get("control_algorithm", "default"),
                "treatment": experiment.get("treatment_algorithm", "candidate")
            },
            "sample_sizes": {
                "control": len(data.get("control", {}).get("matches", [])),
                "treatment": len(data.get("treatment", {}).get("matches", []))
            },
            "before_metrics": before_stats,
            "after_metrics": after_stats,
            "metrics_comparison": self._compare_metrics(before_stats, after_stats),
            "statistical_significance": significance_tests,
            "recommended_parameter_changes": param_changes,
            "recommended_params": recommended_params,
            "projected_impact": projected_impact,
            "conclusion": self._generate_conclusion(significance_tests, param_changes, projected_impact)
        }
        return report

    def _compute_group_statistics(self, metrics: Dict) -> Dict:
        """计算一组指标的统计摘要"""
        stats = {}
        for metric_name, values in metrics.items():
            if not values:
                stats[metric_name] = {"mean": 0, "std": 0, "min": 0, "max": 0, "count": 0}
                continue
            mean = sum(values) / len(values)
            variance = sum((v - mean) ** 2 for v in values) / len(values) if len(values) > 1 else 0
            std = math.sqrt(variance)
            stats[metric_name] = {
                "mean": round(mean, 4),
                "std": round(std, 4),
                "min": round(min(values), 4),
                "max": round(max(values), 4),
                "count": len(values)
            }
        return stats

    def _welch_t_test(self, group_a: List[float], group_b: List[float]) -> float:
        """执行Welch t检验，返回p值"""
        n_a = len(group_a)
        n_b = len(group_b)
        if n_a < 2 or n_b < 2:
            return 1.0
        mean_a = sum(group_a) / n_a
        mean_b = sum(group_b) / n_b
        var_a = sum((x - mean_a) ** 2 for x in group_a) / (n_a - 1)
        var_b = sum((x - mean_b) ** 2 for x in group_b) / (n_b - 1)
        se_a = var_a / n_a
        se_b = var_b / n_b
        se_diff = math.sqrt(se_a + se_b)
        if se_diff < 1e-10:
            return 1.0
        t_statistic = (mean_a - mean_b) / se_diff
        df_num = (se_a + se_b) ** 2
        df_denom = (se_a ** 2 / (n_a - 1)) + (se_b ** 2 / (n_b - 1))
        if df_denom < 1e-10:
            return 1.0
        degrees_of_freedom = df_num / df_denom
        z = t_statistic / math.sqrt(max(1, degrees_of_freedom))
        p_value = 2.0 * self._normal_cdf(-abs(z))
        return max(0.0, min(1.0, p_value))

    def _normal_cdf(self, x: float) -> float:
        """标准正态分布累积分布函数近似"""
        a1 = 0.254829592
        a2 = -0.284496736
        a3 = 1.421413741
        a4 = -1.453152027
        a5 = 1.061405429
        p = 0.3275911
        sign = 1 if x >= 0 else -1
        x = abs(x) / math.sqrt(2.0)
        t = 1.0 / (1.0 + p * x)
        y = 1.0 - (((((a5 * t + a4) * t) + a3) * t + a2) * t + a1) * t * math.exp(-x * x)
        return 0.5 * (1.0 + sign * y)

    def _get_change_rationale(self, param_name: str, current: float, recommended: float) -> str:
        """生成参数变更理由"""
        if param_name == "k_factor":
            if recommended > current:
                return "增大K因子使MMR变化更灵敏，更快反映玩家真实水平变化"
            return "减小K因子使MMR更稳定，减少单场胜负对段位的大幅影响"
        if param_name == "search_expansion_rate":
            if recommended > current:
                return "增大搜索扩展速率缩短等待时间，代价是匹配质量可能略降"
            return "减小搜索扩展速率提升匹配精度，代价是等待时间可能增加"
        if param_name == "max_wait_before_expansion":
            if recommended > current:
                return "延长扩展前等待时间，优先保证匹配质量"
            return "缩短扩展前等待时间，优先减少等待"
        return f"参数{param_name}从{current}调整为{recommended}以优化综合表现"

    def _project_impact(self, before_stats: Dict, after_stats: Dict,
                         significance: Dict) -> Dict:
        """预测推荐变更对玩家体验的影响"""
        projected = {}
        metric_impact_descriptions = {
            "quality": "匹配质量",
            "wait_time": "等待时间",
            "satisfaction": "玩家满意度"
        }
        for metric_name, label in metric_impact_descriptions.items():
            before_mean = before_stats.get(metric_name, {}).get("mean", 0)
            after_mean = after_stats.get(metric_name, {}).get("mean", 0)
            if before_mean > 0:
                change_pct = (after_mean - before_mean) / before_mean * 100
            else:
                change_pct = 0.0
            is_sig = significance.get(metric_name, {}).get("is_significant", False)
            if metric_name == "wait_time":
                is_improvement = change_pct < 0
            else:
                is_improvement = change_pct > 0
            projected[metric_name] = {
                "current_value": round(before_mean, 4),
                "projected_value": round(after_mean, 4),
                "change_pct": round(change_pct, 2),
                "is_improvement": is_improvement,
                "is_significant": is_sig,
                "description": f"{label}预计{'改善' if is_improvement else '恶化'}{abs(round(change_pct, 1))}%"
            }
        return projected

    def _compare_metrics(self, before_stats: Dict, after_stats: Dict) -> Dict:
        """对比前后指标"""
        comparison = {}
        for metric_name in before_stats:
            b_mean = before_stats[metric_name].get("mean", 0)
            a_mean = after_stats.get(metric_name, {}).get("mean", 0)
            diff = a_mean - b_mean
            pct = (diff / b_mean * 100) if b_mean != 0 else 0
            comparison[metric_name] = {
                "before": b_mean,
                "after": a_mean,
                "difference": round(diff, 4),
                "change_pct": round(pct, 2)
            }
        return comparison

    def _generate_conclusion(self, significance: Dict, param_changes: List[Dict],
                              projected_impact: Dict) -> str:
        """生成实验结论"""
        significant_metrics = [k for k, v in significance.items() if v.get("is_significant", False)]
        if not significant_metrics:
            return "实验结果未显示统计学显著差异，建议延长实验周期或增加样本量"
        improvements = [k for k, v in projected_impact.items() if v.get("is_improvement", False)]
        degradations = [k for k, v in projected_impact.items() if not v.get("is_improvement", False)]
        parts = []
        if improvements:
            parts.append(f"显著改善的指标：{', '.join(improvements)}")
        if degradations:
            parts.append(f"需关注的退化指标：{', '.join(degradations)}")
        if param_changes:
            parts.append(f"推荐{len(param_changes)}项参数变更")
        overall = "建议采纳" if len(improvements) > len(degradations) else "建议谨慎评估"
        parts.append(overall)
        return "；".join(parts)

    def add_match_record(self, match: Dict) -> None:
        """添加匹配记录"""
        match_id = match.get("match_id", f"match_{int(time.time())}")
        match["match_id"] = match_id
        match["timestamp"] = match.get("timestamp", time.time())
        all_players = match.get("team_a", []) + match.get("team_b", [])
        for player in all_players:
            player_id = player.get("id", "unknown")
            if player_id not in self.match_history:
                self.match_history[player_id] = []
            self.match_history[player_id].append(match)
```

## 异常场景补充

### 场景：匹配算法调优导致高分段等待过长
```
trigger: 调优后的匹配算法为追求更高质量匹配，将MMR差异容忍度从200缩窄至100，导致钻石和大师段位（MMR>2000）的玩家等待时间从平均15秒飙升至3分钟以上
detection: 1) 按MMR段位监控平均等待时间，当高分段等待时间超过低分段3倍时触发告警；2) 检测匹配队列中滞留时间超过120秒的玩家数量，按段位分组统计；3) 监控玩家退出匹配队列的比率，当高分段退出率超过20%时标记异常；4) 对比调优前后的各段位等待时间分布变化
handling: 1) 立即对大师段位（MMR>2500）放宽MMR匹配范围至150，逐步扩大直到等待时间回落至30秒以内；2) 为高分段玩家启用动态扩展策略：前30秒严格匹配，30-60秒每秒扩展5 MMR，60-90秒每秒扩展10 MMR；3) 临时为高分段玩家提供匹配中等待补偿（额外经验值或金币）；4) 通知受影响玩家匹配优化进行中；5) 回滚调优参数至上一稳定版本作为兜底方案
prevention: 1) 调优参数时必须按MMR段位分别评估，不能使用全局单一参数；2) 设置每个段位的等待时间SLA（如青铜<10秒，钻石<45秒，大师<60秒）；3) 调优前进行分段模拟，预测各段位的影响；4) 实现动态匹配窗口：根据当前在线同段位玩家数量自动调整匹配严格度；5) 在低峰时段（同段位玩家少）预设更宽松的匹配参数；6) 建立"紧急放宽"机制，当等待时间超过SLA时自动降级匹配质量阈值
```

### 场景：A/B 测试样本不均衡
```
trigger: 玩家ID哈希分组的50/50比例在实际流量中被打破，原因是某公会大量成员的ID连续（如guild_001到guild_200），而MD5哈希对这些连续ID产生了偏斜分布，导致对照组70%、实验组30%
detection: 1) 每小时检查实验组的样本比例，当偏离目标比例超过5个百分点时触发告警；2) 对比对照组和实验组的玩家画像分布（段位、活跃度、地区），若分布显著不同则标记为样本偏斜；3) 使用卡方检验验证分组是否与玩家属性独立；4) 监控实验组分配哈希的均匀性，检测是否存在系统性偏差
handling: 1) 立即暂停当前实验，停止收集数据避免更多浪费；2) 分析偏斜原因，如果是哈希函数问题则更换为更均匀的哈希算法（如xxHash或MurmurHash）；3) 如果是特定群体导致的偏斜，采用分层随机抽样替代简单哈希：先按段位分层，再在每层内随机分组；4) 清除已收集的受污染数据，使用新的分组方案重新开始实验；5) 如果偏斜程度较小（<10%），可使用IPW（逆概率加权）校正分析结果
prevention: 1) 使用经过验证的分层随机化而非简单哈希分组，确保各组在关键维度上均衡；2) 在实验开始前执行预检验（A/A测试），验证两组在没有处理差异时指标是否一致；3) 设置实时样本均衡监控，偏离阈值时自动暂停实验；4) 对分组哈希函数定期进行均匀性测试；5) 采用分层+区组设计：先按关键特征分层，再在层内使用随机排列保证精确均衡；6) 在实验设计阶段计算所需最小样本量，确保统计功效
```

### 场景：参数调优过拟合历史数据
```
trigger: 网格搜索找到的"最优"参数组合在历史匹配数据上表现优异，但上线后实际表现远不如预期，匹配质量下降、等待时间增加，因为参数过拟合了历史数据的特定分布特征
detection: 1) 对比离线评估指标与线上实际指标，当偏差超过15%时标记为疑似过拟合；2) 使用K折交叉验证，若不同折间的指标方差很大（CV>0.2）则说明参数不稳定；3) 检测最优参数是否位于搜索空间的边界，边界最优往往意味着过拟合或搜索范围不当；4) 观察最优参数组合与次优组合的分数差异，差异极小（<1%）时说明参数对结果不敏感，选择可能不稳定
handling: 1) 立即回滚至调优前的参数版本；2) 使用交叉验证重新评估所有参数组合，选择在所有折上表现稳定（方差最小）的参数而非单次最优参数；3) 缩小参数搜索范围至当前最优值附近，进行更细粒度但更稳健的搜索；4) 增加正则化项，对参数偏离默认值的程度施加惩罚；5) 使用时间序列交叉验证（而非随机K折），确保模型不会偷看未来数据；6) 将参数集合限制在历史验证过的安全范围内
prevention: 1) 使用交叉验证而非单次训练-测试分割来评估参数；2) 保留最近1周的数据作为hold-out验证集，不参与参数搜索；3) 对参数施加平滑约束，相邻参数组合的表现应该平滑变化，突变意味着过拟合；4) 限制参数搜索的粒度，过细的搜索更容易找到偶然的最优；5) 实施渐进式上线：新参数先在5%流量上验证，确认效果后再逐步扩大；6) 建立"参数健康度"检查：新参数与默认参数的偏差不超过合理阈值；7) 定期用新数据重新验证参数有效性，避免参数老化
```
