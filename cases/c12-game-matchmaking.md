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

### 匹配算法：MMR + 等待时间扩展

**核心思路：** 刚入队的玩家严格匹配同段位，等待越久匹配范围越宽。

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

```python
class GameServerScaler:
    def check_and_scale(self):
        """定期检查是否需要扩缩容"""
        total_capacity = self.get_total_capacity()    # 总房间容量
        active_rooms = self.get_active_rooms()        # 进行中的房间数
        pending_matches = self.get_pending_matches()  # 等待分配的匹配数
        
        utilization = active_rooms / total_capacity
        
        # 扩容条件：利用率 > 80% 或有等待分配的匹配
        if utilization > 0.8 or pending_matches > 0:
            self.scale_up(count=self.calculate_scale_count(utilization))
        
        # 缩容条件：利用率 < 40% 且无即将结束的房间高峰
        elif utilization < 0.4:
            self.scale_down()

    def scale_up(self, count):
        """扩容：启动新的游戏服务器"""
        for _ in range(count):
            server = self.cloud.start_instance(
                type="game-server",
                image="game-server:v2.3",
                wait_until_ready=True  # 等待服务器启动完成
            )
            
            # 注册到可用服务器池
            self.redis.zadd(
                "game:servers:available",
                {server.id: server.max_rooms}  # 剩余容量 = 最大房间数
            )

    def scale_down(self):
        """缩容：选择空闲服务器下线"""
        servers = self.get_all_servers()
        
        for server in servers:
            active_rooms = self.get_active_rooms(server.id)
            
            if active_rooms == 0:
                # 完全空闲 → 可以安全下线
                self.drain_and_terminate(server)
            elif active_rooms < server.max_rooms * 0.1:
                # 接近空闲 → 标记为"不接新房"，等房间结束后下线
                self.mark_draining(server)
                # 不再分配新房间到该服务器
                self.redis.zrem("game:servers:available", server.id)

    def mark_draining(self, server):
        """标记服务器为排空状态"""
        self.redis.set(f"server:draining:{server.id}", "1", ex=3600)
        
        # 设置定时检查：房间全部结束后下线
        self.schedule_check(server.id, interval_seconds=60)
```

**缩容的安全保证：**
1. 标记为 draining 的服务器不再接收新房间
2. 进行中的房间继续运行直到游戏结束
3. 所有房间结束后才真正下线
4. 最长等待时间 = 单局最长时长（30 分钟）

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

## 延伸思考

- **跨区匹配**：不同地区的玩家匹配到同一房间，网络延迟差异大。如何做区域感知匹配？
- **匹配预测**：基于历史数据预测各段位的匹配等待时间，在 UI 上展示"预计等待 15 秒"。
- **自定义房间**：好友约战不需要走匹配系统，但需要房间管理。如何与排位赛房间管理统一？