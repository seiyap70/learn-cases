# C01: 秒杀系统的流量洪峰

## 业务场景

某电商平台每年举办"双11大促"，核心玩法是限时秒杀：每场次放出 1000 件商品，定价仅为日常价的 10%，吸引数十万用户在同一秒点击抢购。

秒杀是平台最重要的获客和促活手段——每个场次的 UV 占当日总 UV 的 30%，秒杀页面的转化率是普通页面的 20 倍。因此秒杀系统不能简单粗暴地"挂个维护页"，必须在保证系统稳定的同时尽可能多地把商品卖出去。

**已知数据：**
- 秒杀场次峰值 QPS：50 万/s（根据去年数据，50 万用户在秒杀开始的 1 秒内同时点击）
- 商品库存：1000 件
- 用户数：3000 万注册用户
- 页面静态资源：CDN 加速
- 秒杀结果需 3 秒内反馈给用户
- 每个用户每次秒杀只能购买 1 件

**流量分布特征：**
- 秒杀开始前 10 分钟：大量用户停留在秒杀页面反复刷新
- 秒杀开始的第 0-1 秒：50 万 QPS 的洪峰
- 第 1-3 秒：流量迅速回落到 10 万 QPS（大量用户看到"售罄"后离开）
- 第 3 秒以后：恢复到 1 万 QPS（只有少数成功者仍在页面操作）

**这意味着：真正的压力只有 3 秒，但这是 50 万 QPS × 3 秒 = 150 万次请求的瞬间洪峰。**

## 核心挑战

### 挑战 1：流量是库存的 500 倍

50 万 QPS vs 1000 件库存。99.8% 的请求注定失败。问题不是"如何处理这些请求"，而是"如何避免这些请求到达需要真正处理它们的地方"。

如果把 50 万 QPS 放到后端，即使只是做一次"检查库存是否充足"的 Redis 查询，也需要 5 个 Redis 节点（每节点 10 万 QPS）。如果请求打到 MySQL，数据库立即崩溃。

**核心思路：层层过滤，越外层越粗粒度，越内层越精确。每一层的目标都是让下一层的流量减少一个数量级。**

### 挑战 2：绝对不能超卖

1000 件商品卖出 1001 件 = 严重的客诉 + 信任危机。在极高并发下保证不超卖，需要仔细分析并发扣减的竞态条件。

### 挑战 3：防止机器刷单

如果不做防刷，黄牛可以用脚本在秒杀开始的第 0.1 秒内发出 1 万次请求，把 1000 件商品全部抢走。普通用户完全无法竞争。

### 挑战 4：3 秒内反馈结果

用户提交秒杀请求后，3 秒内必须知道结果（成功/失败）。如果超过 3 秒用户会反复点击，进一步放大流量。

## 设计约束

- 后端服务部署在云上，可弹性扩缩容，但扩容需 2-3 分钟生效（秒杀洪峰只有 3 秒，扩容来不及）
- 数据库为 MySQL，单机写入上限约 5000 QPS，读上限约 1 万 QPS
- Redis 集群可用，单节点约 10 万 QPS（读写混合）
- 不可使用"排队页面让用户等待"的方案（业务方要求即时反馈，不能让用户盯着一个加载动画）
- 秒杀商品为实物商品，需要创建真实订单并扣减库存
- 订单创建后 15 分钟内未支付自动取消，库存回滚
- 秒杀结果需要持久化，不能仅依赖 Redis 内存状态
- 每个用户每个场次只能参与一次，且只能购买 1 件

## 请先独立思考（限时 30 分钟）

1. 50 万 QPS 只需成功处理 1000 次，剩下的 49.9 万次请求如何在不消耗后端资源的情况下返回"未中签"？
2. 库存扣减的原子性如何保证？写一个你认为无超卖风险的扣减方案，然后自己尝试构造一个并发场景看是否能超卖。
3. 秒杀请求提交后，用户如何获知结果？长轮询、WebSocket、短轮询各有什么利弊？考虑一下如果 50 万用户同时轮询结果，系统是否承受得住。
4. 如果 Redis 集群在秒杀开始的瞬间挂了，系统的行为应该是什么？

---

## 设计解析

### 整体架构：七层过滤漏斗

```
原始流量 50万QPS
  │
  ▼ 第一层：前端随机抽签        → 过滤98%  → 1万QPS
  │
  ▼ 第二层：CDN + 页面控制       → 过滤30%  → 7000QPS
  │
  ▼ 第三层：Nginx 限流           → 过滤50%  → 3500QPS
  │
  ▼ 第四层：网关校验+风控        → 过滤60%  → 1400QPS
  │
  ▼ 第五层：Redis原子预扣        → 扣减1000次 → 1000个成功请求
  │
  ▼ 第六层：消息队列异步         → 削峰缓冲
  │
  ▼ 第七层：MySQL创建订单        → 1000笔订单
```

下面逐层详细分析。

### 第一层：前端随机抽签（过滤 98%）

**原理：** 秒杀只有 1000 件，50 万人抢，成功率约 0.2%。既然 99.8% 的人注定失败，为什么不让 98% 的人在前端就"认命"？

**实现：**

秒杀页面加载时，从服务端获取一个"抽签概率"参数：

```
GET /api/seckill/config?itemId=123
Response:
{
  "itemId": 123,
  "startTime": 1715000000000,
  "drawProbability": 0.02,    // 2% 的请求被允许发出
  "drawSeed": "a3f8c2..."     // 服务端种子（防篡改）
}
```

用户点击"抢购"按钮后，前端执行：

```javascript
function onSeckillClick(itemId) {
  const config = seckillConfig[itemId];

  // 1. 检查是否在秒杀时间内
  if (Date.now() < config.startTime) {
    showToast("秒杀尚未开始");
    return;
  }

  // 2. 随机抽签
  const draw = hashUserId + config.drawSeed;  // 用用户ID+种子生成确定性随机数
  if (draw % 100 >= config.drawProbability * 100) {
    // 98% 的用户直接提示"未中签"，不发请求
    showToast("手慢了，下次再来");
    return;
  }

  // 3. 中签的 2% 用户发出请求
  submitSeckillRequest(itemId);
}
```

**关键细节：**

- 抽签概率可动态调整：如果实际库存是 1 万件而非 1000 件，概率可以调高到 20%
- 使用 `userId + seed` 做确定性哈希，同一个用户在同一个秒杀场次中每次点击的结果相同（避免用户反复点击提高概率）
- 前端抽签只是一种"善意的欺骗"——它不是安全机制，真正的限流在后端。但它的作用是把后端压力从 50 万 QPS 降到 1 万 QPS

**这层过滤的争议：** 你可能会问——让 98% 的用户在前端就失败，公平吗？

答案是：这 98% 的用户即使发了请求也几乎不可能抢到（1000/500000 = 0.2% 成功率），前端抽签只是提前告知结果，省去了他们等待 3 秒的焦虑。实际上用户体验是变好的。

### 第二层：CDN + 页面控制（过滤 30%）

**1. 秒杀按钮时间控制**

秒杀按钮的显示由服务端下发的时间戳控制，而不是前端本地时间：

```
秒杀页面 JS 逻辑：
  - 页面加载时获取服务端时间 serverTime
  - 计算本地时间与服务器时间的偏移：offset = serverTime - Date.now()
  - 判断是否到秒杀时间：Date.now() + offset >= seckillStartTime
  - 未到时间：按钮灰显，倒计时
  - 到时间：按钮激活
```

这样即使用户修改本地系统时间，也无法提前点击。

**2. 秒杀 URL 动态生成**

秒杀接口的 URL 不是固定的，而是在秒杀开始时动态下发：

```
秒杀开始前：前端没有提交接口的 URL
秒杀开始时：服务端通过长连接推送 URL → /api/seckill/submit/a3f8c2e1
秒杀结束后：该 URL 失效
```

好处：脚本无法提前知道提交接口的 URL，必须先访问秒杀页面获取。

**3. 请求频率控制**

前端在用户点击后禁用按钮 5 秒（防止连点），且同一用户 10 秒内不会发出第二次请求。

经过这层，1 万 QPS → 约 7000 QPS（过滤掉 URL 不对的、时间不对的、重复提交的请求）。

### 第三层：Nginx 限流（过滤 50%）

Nginx 是后端的第一道防线，用 `limit_req` 模块做精确限流：

```nginx
# 秒杀接口的限流配置
limit_req_zone $binary_remote_addr zone=seckill_ip:10m rate=1r/s;
limit_req_zone $server_name zone=seckill_total:10m rate=7000r/s;

location /api/seckill/submit/ {
    # 单 IP 每秒最多 1 次请求
    limit_req zone=seckill_ip burst=1 nodelay;

    # 秒杀接口总 QPS 上限 7000（Redis 可承受的安全水位）
    limit_req zone=seckill_total burst=500 nodelay;

    # 超出限流的请求直接返回
    limit_req_status 429;

    proxy_pass http://gateway;
}
```

**限流数值的推算：**

为什么总 QPS 上限设为 7000 而不是更高？

- 后端 Redis 集群（3 节点）的安全水位：3 × 10 万 = 30 万 QPS
- 但 Redis 不只为秒杀服务，还要支撑其他业务
- 分配给秒杀的 Redis 容量：1 万 QPS（留足余量）
- 再考虑网关到 Redis 之间还有校验逻辑，Nginx 限流设为 7000 给下游留余量

超出 7000 QPS 的请求返回 HTTP 429，前端显示"活动太火爆，请稍后再试"。

经过这层：7000 QPS → 约 3500 QPS。

### 第四层：网关校验 + 风控（过滤 60%）

网关是应用层的第一道防线，做业务校验：

**1. 登录态校验**

```
检查 Cookie/Token 是否有效 → 未登录直接拒绝
```

**2. 秒杀资格校验**

```
该用户是否有此场次的参与资格？
  - 是否在黑名单中（历史违规用户）
  - 是否已经参与过本场次秒杀（一人一次）
  - 是否满足活动条件（如新用户专享）
```

**3. 风控规则**

```
风控打分模型：
  - 同一设备指纹 10 秒内多次请求 → 高风险 → 拦截
  - 同一 IP 段短时大量请求（代理/机房 IP）→ 高风险 → 拦截
  - 请求头缺失/异常（脚本特征）→ 中风险 → 拦截
  - 用户注册时间 < 1 天 + 无购物记录 → 中风险 → 人机验证
```

风控判定的结果：
- 低风险 → 放行
- 中风险 → 要求完成人机验证（滑块/图形验证码）
- 高风险 → 直接拦截

人机验证是拦截脚本的有效手段：真实用户可以完成滑块验证，但脚本在 1 秒内难以通过。通过人机验证会增加 1-2 秒延迟，但这恰好是给真实用户的优势——脚本无法在秒杀开始的第 0 秒就提交请求。

经过这层：3500 QPS → 约 1400 QPS。

### 第五层：Redis 原子预扣库存（1000 次成功）

这是最关键的一层——1000 件库存的原子扣减。

**为什么不能用简单的 DECR？**

先看一个错误方案：

```lua
-- 错误方案：GET + DECR 分两步
local stock = redis.call('GET', KEYS[1])   -- 线程A读到 1
if tonumber(stock) > 0 then
    redis.call('DECR', KEYS[1])             -- 线程A还没DECR，线程B也读到 1
    return 1
end
return 0
```

并发场景下，线程 A 和线程 B 同时读到 stock = 1，都判断 > 0，都执行 DECR，结果 stock 变成 -1，**超卖**。

**正确方案：Lua 脚本原子操作**

```lua
-- 正确方案：GET + DECR 在同一个 Lua 脚本中原子执行
-- Redis 保证 Lua 脚本执行期间不会被其他命令打断
local stock = redis.call('GET', KEYS[1])
if tonumber(stock) > 0 then
    redis.call('DECR', KEYS[1])
    return 1  -- 扣减成功
end
return 0  -- 库存不足，扣减失败
```

**更进一步：防止单用户重复扣减**

上面的脚本有个问题——如果同一个用户的请求因为网络重试被执行了两次，会扣减两次库存。

解决方案：在 Lua 脚本中加入用户维度的幂等检查：

```lua
-- 幂等扣减脚本
local stock_key = KEYS[1]           -- stock:item123
local user_key = KEYS[2]            -- seckill:user:item123  (Set 类型)
local user_id = ARGV[1]

-- 1. 检查该用户是否已经参与过
if redis.call('SISMEMBER', user_key, user_id) == 1 then
    return -1  -- 重复请求
end

-- 2. 检查并扣减库存
local stock = redis.call('GET', stock_key)
if tonumber(stock) > 0 then
    redis.call('DECR', stock_key)
    redis.call('SADD', user_key, user_id)  -- 标记该用户已参与
    return 1  -- 成功
end
return 0  -- 库存不足
```

**数据预热：** 秒杀开始前 10 分钟，将库存数据写入 Redis：

```
SET stock:item123 1000
DEL seckill:user:item123   -- 清空用户参与记录
```

**Redis 内存估算：**
- stock_key: 1 个 key，可忽略
- user_key: 最多 1000 个成员（只有成功的人），每个 userId 约 20 字节 → 20KB
- 完全可接受

**扣减结果：** 1400 QPS 中，前 1000 次扣减成功，后 400 次返回"库存不足"。成功的 1000 个请求进入下一层。

#### 完整版 Lua 脚本：原子预扣 + 回滚支持

上面的幂等扣减脚本解决了核心问题，但生产环境还需要处理"预扣后订单创建失败需要回滚"的场景。以下是完整版脚本：

**脚本 1：原子预扣库存（seckill_deduct.lua）**

```lua
-- 秒杀原子预扣脚本（完整版）
-- KEYS[1]: stock:{sessionId}           库存 key
-- KEYS[2]: seckill:users:{sessionId}    已参与用户集合 (Set)
-- KEYS[3]: seckill:reserved:{sessionId} 预扣记录 Hash (userId -> requestId)
-- ARGV[1]: userId
-- ARGV[2]: requestId                   请求唯一ID，用于回滚时匹配
-- ARGV[3]: 当前时间戳(毫秒)            防止过期请求

-- 返回值: 1=成功, 0=库存不足, -1=重复参与, -2=请求过期

local stock_key    = KEYS[1]
local users_key    = KEYS[2]
local reserved_key = KEYS[3]
local user_id      = ARGV[1]
local request_id   = ARGV[2]
local now_ms       = tonumber(ARGV[3])

-- 1. 检查是否已参与过（幂等）
if redis.call('SISMEMBER', users_key, user_id) == 1 then
    -- 同一用户已成功预扣过，检查是否同一次请求
    local existing_req = redis.call('HGET', reserved_key, user_id)
    if existing_req == request_id then
        return 1  -- 同一请求重复提交，返回成功（幂等）
    end
    return -1  -- 不同请求，说明重复参与
end

-- 2. 检查库存
local stock = redis.call('GET', stock_key)
if not stock or tonumber(stock) <= 0 then
    return 0  -- 库存不足
end

-- 3. 原子扣减 + 标记用户 + 记录预扣
redis.call('DECR', stock_key)
redis.call('SADD', users_key, user_id)
redis.call('HSET', reserved_key, user_id, request_id)

-- 4. 设置预扣记录过期时间（15分钟，与支付超时对齐）
-- 防止预扣记录永不过期导致内存泄漏
redis.call('EXPIRE', reserved_key, 900)

return 1  -- 预扣成功
```

**脚本 2：库存回滚（seckill_rollback.lua）**

当订单创建失败（如 MySQL 写入异常）或支付超时时，需要回滚 Redis 预扣的库存：

```lua
-- 秒杀库存回滚脚本
-- KEYS[1]: stock:{sessionId}           库存 key
-- KEYS[2]: seckill:users:{sessionId}    已参与用户集合
-- KEYS[3]: seckill:reserved:{sessionId} 预扣记录 Hash
-- ARGV[1]: userId
-- ARGV[2]: requestId                   必须与预扣时的 requestId 匹配
-- ARGV[3]: reason                      回滚原因: "order_failed" / "pay_timeout"

-- 返回值: 1=回滚成功, 0=未找到预扣记录, -1=requestId不匹配(可能重复回滚)

local stock_key    = KEYS[1]
local users_key    = KEYS[2]
local reserved_key = KEYS[3]
local user_id      = ARGV[1]
local request_id   = ARGV[2]
local reason       = ARGV[3]

-- 1. 检查预扣记录是否存在
local existing_req = redis.call('HGET', reserved_key, user_id)
if not existing_req then
    return 0  -- 没有预扣记录，无需回滚
end

-- 2. 校验 requestId，防止误回滚
if existing_req ~= request_id then
    return -1  -- requestId 不匹配，可能是并发问题或重复回滚
end

-- 3. 原子回滚：恢复库存 + 移除用户标记 + 删除预扣记录
redis.call('INCR', stock_key)
redis.call('SREM', users_key, user_id)
redis.call('HDEL', reserved_key, user_id)

-- 4. 记录回滚日志（可选，用于审计排查）
-- 写入一个 List 记录回滚事件，TTL 1小时
local log_key = "seckill:rollback_log:" .. string.match(stock_key, "stock:(.+)")
redis.call('RPUSH', log_key,
    string.format("%s|%s|%s|%s", user_id, request_id, reason, ARGV[4] or "0"))
redis.call('EXPIRE', log_key, 3600)

return 1  -- 回滚成功
```

**Java 调用示例：**

```java
@Service
public class SeckillStockService {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    // 预扣脚本 SHA 缓存（避免每次传脚本正文）
    private String deductScriptSha;
    private String rollbackScriptSha;

    @PostConstruct
    public void init() {
        // 秒杀开始前加载脚本到 Redis，缓存 SHA
        deductScriptSha = redisTemplate.scriptLoad(loadScript("seckill_deduct.lua"));
        rollbackScriptSha = redisTemplate.scriptLoad(loadScript("seckill_rollback.lua"));
    }

    /**
     * 原子预扣库存
     * @return 1=成功, 0=库存不足, -1=重复参与
     */
    public int deductStock(Long sessionId, Long userId, String requestId) {
        List<String> keys = Arrays.asList(
            "stock:" + sessionId,
            "seckill:users:" + sessionId,
            "seckill:reserved:" + sessionId
        );
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(deductScriptSha, Long.class),
            keys,
            userId.toString(),
            requestId,
            String.valueOf(System.currentTimeMillis())
        );
        return result != null ? result.intValue() : 0;
    }

    /**
     * 回滚库存（订单创建失败或支付超时时调用）
     */
    public boolean rollbackStock(Long sessionId, Long userId, String requestId, String reason) {
        List<String> keys = Arrays.asList(
            "stock:" + sessionId,
            "seckill:users:" + sessionId,
            "seckill:reserved:" + sessionId
        );
        try {
            Long result = redisTemplate.execute(
                new DefaultRedisScript<>(rollbackScriptSha, Long.class),
                keys,
                userId.toString(),
                requestId,
                reason,
                String.valueOf(System.currentTimeMillis())
            );
            return result != null && result == 1L;
        } catch (Exception e) {
            // Redis 调用失败，记录到本地日志，后续人工对账
            log.error("库存回滚失败, sessionId={}, userId={}, requestId={}",
                sessionId, userId, requestId, e);
            return false;
        }
    }

    /**
     * 使用 EVALSHA 调用时如果 Redis 重启导致 SHA 失效，降级为 EVAL
     */
    private <T> T executeWithFallback(String sha, String scriptSource,
                                       RedisScript<T> script, List<String> keys,
                                       String... args) {
        try {
            return redisTemplate.execute(
                new DefaultRedisScript<>(sha, script.getResultType()), keys, args);
        } catch (RedisSystemException e) {
            // SHA 不存在，重新加载脚本
            String newSha = redisTemplate.scriptLoad(scriptSource);
            return redisTemplate.execute(
                new DefaultRedisScript<>(newSha, script.getResultType()), keys, args);
        }
    }
}
```

**回滚场景梳理：**

| 触发条件 | 回滚方式 | 调用时机 |
|---------|---------|---------|
| 订单创建失败（MySQL 异常） | 同步回滚 | 订单服务 catch 异常后立即调用 |
| 支付超时（15分钟未支付） | 异步回滚 | 延迟消息触发，检查订单状态后调用 |
| 用户主动取消订单 | 同步回滚 | 用户点击取消后调用 |
| 对账发现差异 | 补偿回滚 | 对账脚本自动修正，并告警人工复核 |

### 第六层：消息队列异步削峰

1000 个成功扣减的请求不直接写 MySQL，而是投入消息队列（Kafka/RocketMQ）：

```
秒杀服务 → Kafka Topic "seckill-orders" → 订单服务消费
```

**为什么需要消息队列缓冲？**

- 1000 个请求如果在同一秒内打到 MySQL，MySQL 要执行 1000 次 INSERT + 1000 次 UPDATE
- MySQL 单机写入上限 5000 QPS，1000 次写入没问题
- 但如果 MySQL 此时有其他业务写入（普通订单），加上秒杀的 1000 次写入，可能接近上限
- 消息队列可以平滑写入节奏：订单服务以 200 条/秒的速度消费，5 秒内处理完毕

**消息体：**
```json
{
  "requestId": "req-uuid-001",    // 幂等键
  "userId": "U12345",
  "itemId": "item123",
  "sessionId": 100,
  "seckillPrice": 99.00,
  "timestamp": 1715000000123
}
```

**完整的秒杀服务端代码（Java / Spring Boot）：**

```java
@RestController
@RequestMapping("/api/seckill")
public class SeckillController {

    @Autowired
    private SeckillStockService stockService;

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * 秒杀提交接口
     * 流量经过四层过滤后到达此处，QPS 约 1400
     */
    @PostMapping("/submit/{urlToken}")
    public ResponseEntity<SeckillResponse> submit(
            @PathVariable String urlToken,
            @RequestBody SeckillRequest request,
            @RequestHeader("X-User-Id") Long userId) {

        // 1. 验证 URL Token 是否有效（动态 URL 校验）
        String validToken = redisTemplate.opsForValue()
            .get("seckill:url:" + request.getSessionId());
        if (validToken == null || !validToken.equals(urlToken)) {
            return ResponseEntity.ok(SeckillResponse.fail("链接已失效"));
        }

        // 2. 检查秒杀是否在进行中
        String status = redisTemplate.opsForValue()
            .get("seckill:session:" + request.getSessionId() + ":status");
        if (!"ACTIVE".equals(status)) {
            return ResponseEntity.ok(SeckillResponse.fail("秒杀未开始或已结束"));
        }

        // 3. 生成请求唯一ID（用于幂等）
        String requestId = "req-" + request.getSessionId() + "-" + userId + "-" +
            System.currentTimeMillis();

        // 4. Redis 原子预扣库存
        int deductResult = stockService.deductStock(
            request.getSessionId(), userId, requestId);

        if (deductResult == -1) {
            return ResponseEntity.ok(SeckillResponse.fail("您已参与过本场次"));
        }
        if (deductResult == 0) {
            // 库存不足，立即写入失败结果
            redisTemplate.opsForValue().set(
                "seckill:result:" + userId + ":" + request.getSessionId(),
                "soldout", 5, TimeUnit.MINUTES);
            return ResponseEntity.ok(SeckillResponse.pending("手慢了，商品已抢光"));
        }

        // 5. 预扣成功，写入"处理中"状态
        redisTemplate.opsForValue().set(
            "seckill:result:" + userId + ":" + request.getSessionId(),
            "processing", 5, TimeUnit.MINUTES);

        // 6. 异步投递消息到 Kafka
        SeckillOrderMessage message = new SeckillOrderMessage();
        message.setRequestId(requestId);
        message.setUserId(userId);
        message.setItemId(request.getItemId());
        message.setSessionId(request.getSessionId());
        message.setSeckillPrice(request.getSeckillPrice());
        message.setTimestamp(System.currentTimeMillis());

        try {
            // 使用 requestId 作为 key，保证同一用户的消息落到同一分区（有序性）
            kafkaTemplate.send("seckill-orders",
                requestId, JSON.toJSONString(message)).get(1, TimeUnit.SECONDS);
        } catch (Exception e) {
            // Kafka 投递失败，回滚 Redis 库存
            log.error("消息投递失败, 回滚库存, requestId={}", requestId, e);
            stockService.rollbackStock(request.getSessionId(), userId, requestId, "mq_send_failed");
            redisTemplate.opsForValue().set(
                "seckill:result:" + userId + ":" + request.getSessionId(),
                "fail", 5, TimeUnit.MINUTES);
            return ResponseEntity.ok(SeckillResponse.fail("系统繁忙，请稍后重试"));
        }

        // 7. 返回"处理中"，前端开始轮询
        return ResponseEntity.ok(SeckillResponse.pending("正在处理中，请稍候"));
    }
}
```

**Kafka Topic 配置建议：**

```properties
# seckill-orders Topic 配置
num.partitions=6          # 6个分区，支持并行消费
replication.factor=3      # 3副本保证可靠性
min.insync.replicas=2     # 最少2个同步副本
retention.ms=86400000     # 保留24小时

# 生产者配置
acks=all                  # 等待所有 ISR 确认
retries=3                 # 重试3次
max.in.flight.requests.per.connection=1  # 保证顺序性
enable.idempotence=true   # 开启幂等

# 消费者配置
auto.offset.reset=earliest
enable.auto.commit=false  # 手动提交offset
max.poll.records=50       # 每次最多拉50条（控制消费速率）
```

6 个分区 + 限流消费（50 条/批次，间隔 250ms）→ 消费速率约 200 条/秒，1000 条消息 5 秒内处理完毕，不会对 MySQL 造成压力。

### 第七层：MySQL 创建订单

#### 完整数据库设计

秒杀系统涉及的核心表结构如下，所有表均基于 MySQL 8.0+ 设计，使用 InnoDB 引擎。

**1. 商品表（products）**

```sql
CREATE TABLE `products` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '商品ID',
    `name` VARCHAR(200) NOT NULL COMMENT '商品名称',
    `description` TEXT COMMENT '商品描述',
    `original_price` DECIMAL(10,2) NOT NULL COMMENT '原价',
    `image_url` VARCHAR(500) DEFAULT NULL COMMENT '主图URL',
    `category_id` BIGINT UNSIGNED DEFAULT NULL COMMENT '分类ID',
    `status` TINYINT NOT NULL DEFAULT 1 COMMENT '状态: 0=下架 1=上架',
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (`id`),
    INDEX `idx_category` (`category_id`),
    INDEX `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品表';
```

**2. 秒杀场次表（flash_sale_sessions）**

```sql
CREATE TABLE `flash_sale_sessions` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '场次ID',
    `product_id` BIGINT UNSIGNED NOT NULL COMMENT '商品ID',
    `seckill_price` DECIMAL(10,2) NOT NULL COMMENT '秒杀价',
    `total_stock` INT UNSIGNED NOT NULL COMMENT '总库存',
    `available_stock` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '可用库存(冗余,对账用)',
    `start_time` DATETIME NOT NULL COMMENT '开始时间',
    `end_time` DATETIME NOT NULL COMMENT '结束时间',
    `limit_per_user` TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '每用户限购数量',
    `status` TINYINT NOT NULL DEFAULT 0 COMMENT '状态: 0=未开始 1=进行中 2=已结束 3=已取消',
    `min_account_age_days` INT UNSIGNED NOT NULL DEFAULT 7 COMMENT '最低账号注册天数',
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (`id`),
    UNIQUE INDEX `uk_product_session` (`product_id`, `start_time`),
    INDEX `idx_start_time` (`start_time`),
    INDEX `idx_status` (`status`),
    INDEX `idx_end_time` (`end_time`),
    CONSTRAINT `chk_time_range` CHECK (`end_time` > `start_time`),
    CONSTRAINT `chk_stock_positive` CHECK (`total_stock` > 0),
    CONSTRAINT `chk_price_positive` CHECK (`seckill_price` > 0)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='秒杀场次表';
```

**3. 订单表（orders）**

```sql
CREATE TABLE `orders` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '订单ID',
    `request_id` VARCHAR(64) NOT NULL COMMENT '请求唯一ID(幂等键)',
    `user_id` BIGINT UNSIGNED NOT NULL COMMENT '用户ID',
    `session_id` BIGINT UNSIGNED NOT NULL COMMENT '秒杀场次ID',
    `product_id` BIGINT UNSIGNED NOT NULL COMMENT '商品ID',
    `seckill_price` DECIMAL(10,2) NOT NULL COMMENT '秒杀成交价',
    `quantity` TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '购买数量',
    `status` TINYINT NOT NULL DEFAULT 0 COMMENT '订单状态: 0=待支付 1=已支付 2=已发货 3=已完成 4=已取消 5=超时关闭',
    `pay_deadline` DATETIME NOT NULL COMMENT '支付截止时间',
    `paid_at` DATETIME DEFAULT NULL COMMENT '实际支付时间',
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (`id`),
    UNIQUE INDEX `uk_request_id` (`request_id`),
    UNIQUE INDEX `uk_user_session` (`user_id`, `session_id`),
    INDEX `idx_user_id` (`user_id`),
    INDEX `idx_session_id` (`session_id`),
    INDEX `idx_status_created` (`status`, `created_at`),
    INDEX `idx_pay_deadline` (`pay_deadline`),
    CONSTRAINT `chk_quantity` CHECK (`quantity` > 0),
    CONSTRAINT `chk_price` CHECK (`seckill_price` > 0)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='秒杀订单表';
```

`uk_request_id` 保证消息重复消费不会创建重复订单；`uk_user_session` 保证同一用户同一场次只能下一单。

**4. 库存快照表（inventory_snapshots）**

```sql
CREATE TABLE `inventory_snapshots` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '快照ID',
    `session_id` BIGINT UNSIGNED NOT NULL COMMENT '秒杀场次ID',
    `product_id` BIGINT UNSIGNED NOT NULL COMMENT '商品ID',
    `initial_stock` INT UNSIGNED NOT NULL COMMENT '初始库存',
    `current_stock` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '当前库存',
    `reserved_stock` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '预扣库存(Redis已扣,MySQL尚未确认)',
    `sold_count` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '已售数量',
    `snapshot_time` DATETIME NOT NULL COMMENT '快照时间',
    `reconciled` TINYINT NOT NULL DEFAULT 0 COMMENT '是否已对账: 0=否 1=是',
    `discrepancy` INT DEFAULT NULL COMMENT '差异数量(对账发现)',
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    PRIMARY KEY (`id`),
    UNIQUE INDEX `uk_session_snapshot` (`session_id`, `snapshot_time`),
    INDEX `idx_reconciled` (`reconciled`),
    CONSTRAINT `chk_initial_stock` CHECK (`initial_stock` >= 0),
    CONSTRAINT `chk_current_stock` CHECK (`current_stock` >= 0)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='库存快照表(对账用)';
```

**5. 库存变更流水表（inventory_change_log）**

```sql
CREATE TABLE `inventory_change_log` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '流水ID',
    `session_id` BIGINT UNSIGNED NOT NULL COMMENT '秒杀场次ID',
    `product_id` BIGINT UNSIGNED NOT NULL COMMENT '商品ID',
    `order_id` BIGINT UNSIGNED DEFAULT NULL COMMENT '关联订单ID',
    `change_type` TINYINT NOT NULL COMMENT '变更类型: 1=预扣 2=确认扣减 3=回滚释放 4=超时释放',
    `change_amount` INT NOT NULL COMMENT '变更数量(正=增加,负=减少)',
    `stock_before` INT UNSIGNED NOT NULL COMMENT '变更前库存',
    `stock_after` INT UNSIGNED NOT NULL COMMENT '变更后库存',
    `operator` VARCHAR(64) NOT NULL DEFAULT 'system' COMMENT '操作来源',
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    PRIMARY KEY (`id`),
    INDEX `idx_session_product` (`session_id`, `product_id`),
    INDEX `idx_order_id` (`order_id`),
    INDEX `idx_created_at` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='库存变更流水表';
```

该表记录每一次库存变动，用于对账、审计和异常恢复。秒杀结束后可通过流水表完整还原库存变更过程。

#### 订单创建流程

订单服务从 Kafka 消费消息，创建订单：

```sql
-- 1. 创建订单（幂等：requestId 唯一约束）
INSERT INTO orders (request_id, user_id, session_id, product_id, seckill_price,
                    status, pay_deadline, created_at)
VALUES ('req-uuid-001', 12345, 100, 123, 99.00, 0,
        DATE_ADD(NOW(), INTERVAL 15 MINUTE), NOW())
ON DUPLICATE KEY UPDATE id=id;  -- 重复消费时忽略

-- 2. 扣减 MySQL 库存（乐观锁兜底）
UPDATE flash_sale_sessions
SET available_stock = available_stock - 1
WHERE id = 100 AND available_stock > 0;

-- 3. 记录库存变更流水
INSERT INTO inventory_change_log (session_id, product_id, order_id,
                                   change_type, change_amount, stock_before, stock_after)
SELECT 100, 123, LAST_INSERT_ID(), 2, -1,
       available_stock + 1, available_stock
FROM flash_sale_sessions WHERE id = 100;
```

**为什么 MySQL 还需要检查 `available_stock > 0`？**

Redis 已经保证只成功扣减 1000 次，MySQL 不应该再超卖。但作为防御性编程，如果 Redis 和 MySQL 之间出现不一致（如 Redis 预扣了但消息重复投递），MySQL 的 `available_stock > 0` 检查是最后一道防线。

**完整的订单消费者代码（Java / Spring Boot）：**

```java
@Component
@KafkaListener(topics = "seckill-orders", groupId = "order-service")
public class SeckillOrderConsumer {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Autowired
    private SeckillStockService stockService;  // Redis 回滚服务

    @Value("${seckill.pay.timeout.minutes:15}")
    private int payTimeoutMinutes;

    @KafkaListener
    public void handleSeckillOrder(ConsumerRecord<String, String> record,
                                    Acknowledgment ack) {
        SeckillOrderMessage msg = parseMessage(record.value());

        try {
            // 1. 幂等创建订单（事务内）
            createOrderInTransaction(msg);

            // 2. 发送支付超时延迟消息
            sendPayTimeoutMessage(msg);

            // 3. 更新 Redis 结果状态
            updateResultStatus(msg.getUserId(), msg.getSessionId(), "success");

        } catch (DuplicateKeyException e) {
            // requestId 重复，说明消息重复消费，忽略即可
            log.info("重复消息, requestId={}", msg.getRequestId());

        } catch (Exception e) {
            // 订单创建失败，回滚 Redis 预扣库存
            log.error("订单创建失败, 开始回滚库存, requestId={}", msg.getRequestId(), e);
            boolean rolledBack = stockService.rollbackStock(
                msg.getSessionId(), msg.getUserId(),
                msg.getRequestId(), "order_failed"
            );
            if (!rolledBack) {
                // 回滚也失败了，记录到补偿表，后续人工处理
                saveCompensationRecord(msg, e.getMessage());
            }

            // 更新 Redis 结果状态为失败
            updateResultStatus(msg.getUserId(), msg.getSessionId(), "fail");
        }

        // 手动提交 offset
        ack.acknowledge();
    }

    @Transactional
    public void createOrderInTransaction(SeckillOrderMessage msg) {
        // 创建订单
        jdbcTemplate.update(
            "INSERT INTO orders (request_id, user_id, session_id, product_id, " +
            "seckill_price, status, pay_deadline, created_at) " +
            "VALUES (?, ?, ?, ?, ?, 0, DATE_ADD(NOW(), INTERVAL ? MINUTE), NOW()) " +
            "ON DUPLICATE KEY UPDATE id=id",
            msg.getRequestId(), msg.getUserId(), msg.getSessionId(),
            msg.getItemId(), msg.getSeckillPrice(), payTimeoutMinutes
        );

        // 扣减 MySQL 库存（乐观锁）
        int updated = jdbcTemplate.update(
            "UPDATE flash_sale_sessions SET available_stock = available_stock - 1 " +
            "WHERE id = ? AND available_stock > 0",
            msg.getSessionId()
        );

        if (updated == 0) {
            // MySQL 库存已为 0，说明 Redis-MySQL 不一致
            // 这笔订单仍然创建（Redis 已承诺成功），但需要告警
            log.warn("MySQL库存不足但Redis已预扣, sessionId={}, userId={}",
                msg.getSessionId(), msg.getUserId());
            alertInconsistency(msg.getSessionId(), msg.getUserId());
        }
    }

    /**
     * 发送支付超时延迟消息（RocketMQ 延迟消息）
     */
    private void sendPayTimeoutMessage(SeckillOrderMessage msg) {
        // RocketMQ 16级延迟: 1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
        // 选择 15m 级别（level=14）
        Message timeoutMsg = new Message(
            "seckill-pay-timeout",
            msg.getRequestId().getBytes()
        );
        timeoutMsg.setDelayTimeLevel(14);  // 约 15 分钟延迟
        // rocketMQTemplate.send(timeoutMsg);
    }
}
```

### 结果推送：短轮询方案

**为什么选短轮询而不是 WebSocket？**

| 方案 | 优点 | 缺点 | 适用性 |
|------|------|------|--------|
| WebSocket | 实时推送 | 需要 50 万个 WebSocket 连接；秒杀只有 3 秒，连接建连本身就耗时 | 不适合 |
| 长轮询 | 较实时 | 服务端需要 hold 住连接，占用线程资源 | 可选 |
| 短轮询 | 实现简单，无状态 | 有延迟（取决于轮询间隔） | 最适合 |

**选择短轮询的理由：**
- 秒杀场景下，50 万用户可能同时轮询。但 98% 的用户在前端已经过滤，实际轮询的只有约 1 万人
- 短轮询每次请求是无状态的，不会占住连接
- 轮询间隔 500ms，3 秒内最多 6 次请求，1 万人 × 6 次 = 6 万次查询，Redis 轻松承受

**实现：**

用户提交秒杀请求后，前端开始轮询结果：

```javascript
function pollResult(itemId) {
  const maxAttempts = 6;   // 最多轮询 6 次
  const interval = 500;    // 每 500ms 一次
  let attempts = 0;

  const timer = setInterval(() => {
    fetch(`/api/seckill/result?itemId=${itemId}`)
      .then(res => res.json())
      .then(data => {
        if (data.status !== 'pending') {
          // 有结果了
          clearInterval(timer);
          showResult(data.status);  // success / fail / soldout
        }
      });

    attempts++;
    if (attempts >= maxAttempts) {
      clearInterval(timer);
      showToast("查询超时，请稍后在订单列表查看");
    }
  }, interval);
}
```

**服务端：** 秒杀服务将结果写入 Redis：

```
# 成功
SET seckill:result:U12345:item123 "success" EX 300   # 5 分钟过期

# 失败（库存不足）
SET seckill:result:U12345:item123 "soldout" EX 300
```

### Redis-MySQL 数据一致性对账

秒杀结束后，Redis 的库存应该为 0，MySQL 的库存应该减少了 1000。但可能出现不一致：

**不一致场景分析：**

| 场景 | Redis 状态 | MySQL 状态 | 原因 |
|------|-----------|-----------|------|
| 正常 | stock=0 | 扣了1000 | 正常 |
| Redis 扣减但消息丢失 | stock=0 | 扣了990 | Kafka 消息丢失（极少发生） |
| 消息重复消费 | stock=0 | 扣了1000 | 订单表 requestId 唯一约束兜底，实际不会多扣 |
| Redis 故障后恢复 | stock=1000 | 扣了1000 | Redis 从 RDB 恢复到秒杀前状态 |

**对账方案：**

```
秒杀结束后 5 分钟，运行对账脚本：

1. 从订单表统计：SELECT COUNT(*) FROM orders WHERE item_id = 'item123' AND status != 'CANCELLED'
   → 假设是 1000

2. 从 MySQL 库存表读取：SELECT stock FROM inventory WHERE item_id = 'item123'
   → 假设原始库存 1000，当前应为 0

3. 校验：原始库存 - 订单数 = 当前库存
   → 1000 - 1000 = 0 ✓

4. 如果不一致：
   → 以订单数为准，修正 MySQL 库存
   → 记录差异日志，人工复核
```

### 兜底方案：故障场景逐个分析

**场景 1：Redis 集群挂掉**

- 影响：无法执行预扣，秒杀无法进行
- 应对：Nginx 层配置降级开关，Redis 不可用时秒杀接口直接返回"系统繁忙"
- 绝对不能让流量直接打到 MySQL——50 万 QPS 会瞬间打垮数据库，影响全站

```
# Nginx 降级配置
location /api/seckill/submit/ {
    # Redis 健康检查
    set $redis_healthy 1;  # 由配置中心动态下发

    if ($redis_healthy = 0) {
        return 503 "系统繁忙，请稍后再试";
    }

    proxy_pass http://gateway;
}
```

**场景 2：Kafka 挂掉**

- 影响：消息无法投递，订单无法创建
- 应对：秒杀服务本地缓冲消息（内存队列），Kafka 恢复后批量投递
- 1000 条消息在内存中占用极小，不会 OOM

**场景 3：MySQL 挂掉**

- 影响：订单无法落库
- 应对：Kafka 中的消息不消费（等 MySQL 恢复后继续消费）
- 用户侧：Redis 中已经记录了"成功"状态，用户看到的是"秒杀成功，订单创建中"
- MySQL 恢复后消费积压消息，创建订单

**场景 4：全部正常但流量超出预期**

- 应对：Nginx 层的 `limit_req` 已经限流在 7000 QPS，超出部分返回 429
- 这是设计就考虑的：宁可让部分用户看到"太火爆"，也不能让后端崩溃导致所有人都无法使用

### 深度异常场景分析

上面四个场景是最基本的故障分析，生产环境中还有更复杂的异常场景需要逐一分析和应对。

#### 场景 5：Redis 主从切换（秒杀进行中 Failover）

**场景描述：** 秒杀进行到第 0.5 秒时，Redis 主节点因负载过高触发 sentinel 自动故障转移，从节点升级为主节点。切换期间有 1-3 秒不可用窗口。

**风险分析：**

```
时间线：
  T=0s    秒杀开始，Redis 主节点正常工作
  T=0.5s  主节点 CPU 100%，sentinel 判定主观下线
  T=1.0s  sentinel 判定客观下线，开始选举新主
  T=2.0s  新主就绪，客户端连接切换
  T=2.5s  服务恢复正常

问题1：切换期间 1-3 秒的请求如何处理？
  → 秒杀服务 catch Redis 连接异常，返回"系统繁忙"，不回滚也不扣减

问题2：新主节点的数据是否完整？
  → 如果旧主在切换前有部分写操作尚未同步到从节点，这些操作会丢失
  → 最坏情况：Redis 已扣减了 800 件，但新主只有 700 件的记录
  → 结果：库存从 200 变成 300，多了 100 件可售

问题3：旧主恢复后会不会脑裂？
  → Redis Sentinel 会在旧主恢复后将其降级为从节点
  → 但如果旧主在断开期间继续处理了请求（脑裂窗口），这些数据会被覆盖
```

**应对方案：**

```java
@Service
public class RedisFailoverHandler {

    private static final int MAX_RETRIES = 2;
    private static final long RETRY_INTERVAL_MS = 100;

    /**
     * 带故障转移重试的 Redis 操作
     * 核心原则：宁可少卖，不可超卖
     */
    public int deductWithFailover(Long sessionId, Long userId, String requestId) {
        for (int attempt = 0; attempt <= MAX_RETRIES; attempt++) {
            try {
                int result = stockService.deductStock(sessionId, userId, requestId);
                return result;
            } catch (RedisConnectionFailureException e) {
                log.warn("Redis连接失败, 第{}次重试, sessionId={}", attempt + 1, sessionId);
                if (attempt < MAX_RETRIES) {
                    try { Thread.sleep(RETRY_INTERVAL_MS); } catch (InterruptedException ignored) {}
                }
            } catch (RedisCommandTimeoutException e) {
                log.warn("Redis命令超时, sessionId={}", sessionId);
                return 0;  // 超时不重试，返回库存不足（宁可少卖）
            }
        }
        // 重试用尽，触发降级
        return 0;  // 返回库存不足，保护后端
    }
}
```

**Redis 故障转移后的数据修复：**

```
秒杀结束后对账脚本额外检查：
1. 检查 Redis 的 stock 值是否 > 0（正常应为 0）
   → 如果 > 0，说明故障转移导致部分扣减丢失
2. 以 MySQL 订单数为准修正 Redis
3. 如果 Redis stock=0 但 MySQL 订单 < 1000，说明有用户看到了成功但订单未创建
   → 这些用户需要在订单列表中展示"订单创建中"状态
   → 对账脚本自动补偿创建订单或通知用户
```

#### 场景 6：消息队列积压

**场景描述：** 订单服务由于 MySQL 慢查询导致消费速度下降，Kafka 中积压了 5000 条消息（多个场次叠加）。

**风险分析：**

```
正常消费速率：200 条/秒
积压 5000 条 → 需要 25 秒消化
→ 这 25 秒内新秒杀场次的订单创建延迟
→ 用户轮询结果超时，看到"查询超时"
```

**应对方案——消费端动态扩容：**

```java
@Component
public class KafkaLagMonitor {

    @Autowired
    private AdminClient kafkaAdminClient;

    @Autowired
    private KafkaConsumerManager consumerManager;

    /**
     * 每 10 秒检查一次消费者积压
     * 如果积压超过阈值，动态增加消费者实例
     */
    @Scheduled(fixedDelay = 10000)
    public void monitorConsumerLag() {
        try {
            Map<TopicPartition, OffsetAndMetadata> offsets = kafkaAdminClient
                .listConsumerGroupOffsets("order-service")
                .partitionsToOffsetAndMetadata().get();

            long totalLag = calculateTotalLag(offsets);

            if (totalLag > 2000) {
                // 积压超过 2000，告警
                log.warn("Kafka消费积压严重, lag={}, 触发扩容", totalLag);
                // 通知 Kubernetes HPA 扩容订单服务 Pod
                consumerManager.scaleUp(3);  // 扩到 3 个消费实例
            } else if (totalLag > 500) {
                // 积压 500-2000，轻度告警
                log.warn("Kafka消费积压, lag={}", totalLag);
                consumerManager.scaleUp(1);  // 扩 1 个实例
            } else if (totalLag < 100) {
                // 积压消除，缩容
                consumerManager.scaleDown();
            }
        } catch (Exception e) {
            log.error("Kafka积压监控异常", e);
        }
    }

    private long calculateTotalLag(
            Map<TopicPartition, OffsetAndMetadata> committedOffsets) {
        // 获取各分区的最新 offset，计算 lag
        long totalLag = 0;
        for (Map.Entry<TopicPartition, OffsetAndMetadata> entry :
                committedOffsets.entrySet()) {
            long committed = entry.getValue().offset();
            // 获取分区最新 offset
            Map<TopicPartition, Long> endOffsets = kafkaAdminClient
                .listOffsets(Map.of(entry.getKey(),
                    OffsetSpec.latest())).all().get();
            Long endOffset = endOffsets.get(entry.getKey());
            if (endOffset != null) {
                totalLag += endOffset - committed;
            }
        }
        return totalLag;
    }
}
```

**注意事项：** Kafka 消费者实例数不能超过分区数（6 个分区最多 6 个消费者），因此分区数要提前规划好。如果分区数不够，需要临时创建新 Topic 扩分区。

#### 场景 7：超卖检测与恢复

**场景描述：** 对账发现 MySQL 订单数 > 原始库存（如 1002 > 1000），说明发生了超卖。

**超卖根因排查清单：**

```
1. Lua 脚本被绕过？
   → 检查是否有代码路径绕过了 Lua 脚本直接操作 Redis
   → 检查 Lua 脚本是否有逻辑漏洞

2. Redis-MySQL 双写不一致？
   → 检查 Kafka 消息是否重复投递导致 MySQL 多次扣减
   → 检查 ON DUPLICATE KEY UPDATE 是否生效

3. Redis 故障转移导致数据回滚？
   → 检查秒杀期间是否有 Redis 主从切换
   → 切换后库存值是否异常增大

4. 管理后台手动操作？
   → 检查是否有运营人员在秒杀期间手动增加库存
   → 检查是否有后台任务修改了库存数据
```

**超卖恢复流程：**

```java
@Service
public class OversellRecoveryService {

    /**
     * 超卖恢复：当订单数 > 库存时的补偿逻辑
     *
     * 策略：按"先下单先得"原则，超出的订单标记为"超卖取消"
     * 并发放补偿优惠券
     */
    @Transactional
    public void recoverOversell(Long sessionId, int oversellCount) {
        // 1. 找出超出的订单（按创建时间排序，取消最晚的）
        List<Order> oversellOrders = jdbcTemplate.query(
            "SELECT * FROM orders WHERE session_id = ? AND status != 4 " +
            "ORDER BY created_at DESC LIMIT ?",
            orderRowMapper, sessionId, oversellCount
        );

        // 2. 逐个取消超卖订单
        for (Order order : oversellOrders) {
            // 取消订单
            jdbcTemplate.update(
                "UPDATE orders SET status = 4, cancel_reason = 'oversell' " +
                "WHERE id = ?", order.getId()
            );

            // 退还库存
            jdbcTemplate.update(
                "UPDATE flash_sale_sessions SET available_stock = available_stock + 1 " +
                "WHERE id = ?", sessionId
            );

            // 发放补偿优惠券
            couponService.issueCompensationCoupon(
                order.getUserId(), order.getSeckillPrice().doubleValue(),
                "秒杀超卖补偿"
            );

            // 通知用户
            notificationService.notifyOversellCancellation(
                order.getUserId(), order.getId()
            );

            log.info("超卖订单已取消, orderId={}, userId={}", order.getId(), order.getUserId());
        }

        // 3. 记录超卖事件
        jdbcTemplate.update(
            "INSERT INTO inventory_change_log (session_id, change_type, " +
            "change_amount, operator) VALUES (?, 4, ?, 'oversell_recovery')",
            sessionId, oversellCount
        );

        // 4. 告警
        alertService.sendOversellAlert(sessionId, oversellCount);
    }
}
```

#### 场景 8：支付超时级联效应

**场景描述：** 1000 个秒杀成功用户中，600 个在 15 分钟内未支付（秒杀用户冲动消费后后悔比例高）。600 件库存回滚后需要二次销售，但秒杀活动已结束。

**问题链：**

```
600 人未支付
  → 600 件库存回滚到 Redis 和 MySQL
  → 但秒杀活动已结束，用户无法购买
  → 库存白白浪费
  → 商家损失严重（600 件商品以秒杀价备货但未售出）
```

**应对方案——二次销售机制：**

```java
@Service
public class PaymentTimeoutHandler {

    @Autowired
    private SeckillStockService stockService;

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    /**
     * 处理支付超时：取消订单 + 库存回滚 + 触发二次销售
     */
    @Transactional
    public void handlePaymentTimeout(Long orderId, Long sessionId,
                                      Long userId, String requestId) {
        // 1. 检查订单是否确实未支付（防止并发问题）
        Order order = orderRepository.findById(orderId);
        if (order.getStatus() != OrderStatus.PENDING_PAYMENT) {
            return;  // 已支付或已取消，忽略
        }

        // 2. 取消订单
        order.setStatus(OrderStatus.PAYMENT_TIMEOUT);
        order.setCancelReason("timeout");
        orderRepository.save(order);

        // 3. 回滚 Redis 库存
        boolean rolledBack = stockService.rollbackStock(
            sessionId, userId, requestId, "pay_timeout");

        // 4. 回滚 MySQL 库存
        jdbcTemplate.update(
            "UPDATE flash_sale_sessions SET available_stock = available_stock + 1 " +
            "WHERE id = ?", sessionId);

        // 5. 判断是否需要触发二次销售
        int remainingStock = jdbcTemplate.queryForObject(
            "SELECT available_stock FROM flash_sale_sessions WHERE id = ?",
            Integer.class, sessionId);

        if (remainingStock > 50 && isSessionStillActive(sessionId)) {
            // 库存回滚量较大且活动未结束，触发二次销售
            triggerResale(sessionId, remainingStock);
        } else if (remainingStock > 0 && !isSessionStillActive(sessionId)) {
            // 活动已结束但仍有库存，创建补售场次
            createMakeupSession(sessionId, remainingStock);
        }
    }

    /**
     * 触发二次销售：重新开放库存
     */
    private void triggerResale(Long sessionId, int stock) {
        // 更新 Redis 库存（让新用户可以抢购）
        redisTemplate.opsForValue().set("stock:" + sessionId, String.valueOf(stock));

        // 清除部分用户参与记录（允许之前未成功的用户重试）
        // 注意：之前成功的用户仍然在 Set 中，不能重复购买

        // 通知前端刷新页面
        notificationService.broadcastResaleNotification(sessionId, stock);
    }

    /**
     * 创建补售场次：活动结束后仍有库存，开放限时补售
     */
    private void createMakeupSession(Long originalSessionId, int stock) {
        // 创建一个新的补售场次，价格可以略高于秒杀价
        FlashSaleSession makeupSession = new FlashSaleSession();
        makeupSession.setProductId(/* 从原场次获取 */);
        makeupSession.setTotalStock(stock);
        makeupSession.setStartTime(LocalDateTime.now().plusMinutes(5));  // 5分钟后开始
        makeupSession.setEndTime(LocalDateTime.now().plusHours(1));
        // ... 其他字段
        sessionRepository.save(makeupSession);

        log.info("创建补售场次, sessionId={}, stock={}", makeupSession.getId(), stock);
    }
}
```

**支付超时率的优化：**

```
问题：秒杀场景的支付超时率远高于普通订单（60% vs 5%）
原因：用户冲动点击后后悔、来不及付款、网络问题

优化措施：
1. 缩短支付时限：从 15 分钟缩短到 5 分钟
   → 减少库存锁定时间，加快库存回滚速度
   → 前端显示倒计时，营造紧迫感

2. 提前预热支付：秒杀开始前要求用户预先绑定支付方式
   → 减少支付环节的操作步骤
   → 支持指纹/面容一键支付

3. 分批释放：不是 15 分钟后统一释放，而是按下单时间逐个到期释放
   → 每释放一件就立即开放一件，增加二次销售机会

4. 阶梯式库存保护：初始只放出 800 件，预留 200 件给超时回滚后的二次销售
   → 缺点：初始可售量减少，用户感知"更难抢"
   → 权衡：根据历史超时率决定预留比例
```

### 防刷方案细化

| 层级 | 防刷手段 | 拦截对象 |
|------|---------|---------|
| 前端 | 动态 URL + 时间控制 | 低级脚本 |
| 前端 | 抽签概率控制 | 所有用户（含脚本） |
| Nginx | IP 限流 1次/秒 | 同一 IP 的高频请求 |
| 网关 | 设备指纹检测 | 模拟器/自动化工具 |
| 网关 | 人机验证（滑块） | 疑似脚本 |
| 网关 | 用户行为分析 | 注册即秒杀的"秒注号" |
| Redis | 用户去重（Set） | 同一用户重复请求 |

**多账号协同刷单**的防御：如果黄牛控制了 1 万个真实账号，每个账号发 1 次请求，上述手段都难以识别。这种情况下：
- 限制新注册用户参与秒杀（注册 < 7 天的账号不可参与）
- 分析账号画像：无购物记录、无浏览历史的账号降低中签概率
- 最终手段：提高抽签概率中的"用户质量权重"——老用户/活跃用户的中签概率更高

### 反作弊系统完整实现

上面的防刷方案是按层级列举的，这里给出完整的代码级实现，涵盖机器人检测、IP 限流、设备指纹三大核心能力。

#### 1. 机器人检测服务

通过分析请求行为特征，区分真实用户和脚本。脚本的典型特征：请求间隔极其均匀、缺少浏览器特有请求头、鼠标/触屏轨迹缺失。

```java
@Service
public class BotDetectionService {

    // 脚本特征：常见 HTTP 客户端的 User-Agent 关键词
    private static final Set<String> BOT_UA_KEYWORDS = Set.of(
        "python", "curl", "wget", "httpclient", "okhttp",
        "requests", "aiohttp", "java/", "go-http", "apache"
    );

    // 浏览器必须携带的请求头（脚本通常会省略）
    private static final Set<String> REQUIRED_HEADERS = Set.of(
        "accept", "accept-language", "accept-encoding", "referer"
    );

    /**
     * 综合判定请求是否来自机器人
     * @return 风险评分 0-100，>=70 判定为机器人
     */
    public int calculateBotScore(SeckillRequestContext ctx) {
        int score = 0;

        // 规则1：User-Agent 检测（权重 30 分）
        String ua = ctx.getHeader("user-agent");
        if (ua == null || ua.isEmpty()) {
            score += 30;  // 无 UA，几乎肯定是脚本
        } else {
            String uaLower = ua.toLowerCase();
            for (String keyword : BOT_UA_KEYWORDS) {
                if (uaLower.contains(keyword)) {
                    score += 30;
                    break;
                }
            }
        }

        // 规则2：缺少浏览器特征请求头（权重 20 分）
        int missingHeaders = 0;
        for (String header : REQUIRED_HEADERS) {
            if (ctx.getHeader(header) == null) {
                missingHeaders++;
            }
        }
        score += missingHeaders * 5;  // 每缺一个 5 分，最多 20 分

        // 规则3：请求间隔分析（权重 25 分）
        // 正常用户点击间隔随机（500ms-5s），脚本间隔均匀（<100ms）
        List<Long> recentIntervals = ctx.getRecentRequestIntervals();
        if (recentIntervals != null && recentIntervals.size() >= 3) {
            double avgInterval = recentIntervals.stream()
                .mapToLong(Long::longValue).average().orElse(1000);
            double stdDev = calculateStdDev(recentIntervals);

            if (avgInterval < 100) {
                score += 25;  // 平均间隔 < 100ms，几乎肯定是脚本
            } else if (avgInterval < 500 && stdDev < 50) {
                score += 20;  // 间隔短且均匀，高度疑似脚本
            } else if (stdDev < 10 && avgInterval < 2000) {
                score += 15;  // 间隔过于均匀，可能是定时器驱动
            }
        }

        // 规则4：Cookie 完整性（权重 15 分）
        // 真实浏览器会自动携带之前服务端设置的 Cookie
        if (ctx.getCookie("_fingerprint") == null) {
            score += 8;   // 缺少指纹 Cookie
        }
        if (ctx.getCookie("_session") == null) {
            score += 7;   // 缺少会话 Cookie
        }

        // 规则5：Referer 检查（权重 10 分）
        String referer = ctx.getHeader("referer");
        if (referer == null || !referer.contains("shop.example.com/seckill")) {
            score += 10;  // 非秒杀页面跳转来的
        }

        return Math.min(score, 100);
    }

    private double calculateStdDev(List<Long> values) {
        double mean = values.stream().mapToLong(Long::longValue).average().orElse(0);
        double variance = values.stream()
            .mapToDouble(v -> Math.pow(v - mean, 2))
            .average().orElse(0);
        return Math.sqrt(variance);
    }
}
```

#### 2. IP 限流服务（滑动窗口 + 地域检测）

单纯的固定窗口限流容易被绕过（窗口边界突发），采用滑动窗口算法，并结合 IP 地域信息检测机房/代理 IP。

```java
@Service
public class IpRateLimitService {

    @Autowired
    private StringRedisTemplate redisTemplate;

    // 机房 IP 段（CIDR），这些 IP 段几乎不可能有真实用户
    private static final List<String> DATACENTER_CIDRS = List.of(
        "10.0.0.0/8",       // 内网（不应该出现在公网请求中）
        "172.16.0.0/12",    // 内网
        "192.168.0.0/16",   // 内网
        // 云服务商公网段（示例）
        "47.92.0.0/14",     // 阿里云华北
        "39.96.0.0/13",     // 阿里云华东
        "106.52.0.0/15",    // 腾讯云
        "121.4.0.0/15",     // 腾讯云
        "43.128.0.0/13"     // AWS 亚太
    );

    private final SubnetUtils[] datacenterSubnets;

    public IpRateLimitService() {
        datacenterSubnets = DATACENTER_CIDRS.stream()
            .map(cidr -> new SubnetUtils(cidr))
            .peek(su -> su.setInclusiveHostCount(true))
            .toArray(SubnetUtils[]::new);
    }

    /**
     * 滑动窗口限流：检查该 IP 是否超过速率限制
     * @param ip 客户端 IP
     * @param windowSeconds 窗口大小（秒）
     * @param maxRequests 窗口内最大请求数
     * @return true=允许，false=限流
     */
    public boolean allowRequest(String ip, int windowSeconds, int maxRequests) {
        String key = "seckill:ip_rate:" + ip;
        long now = System.currentTimeMillis();

        // Redis Sorted Set 实现滑动窗口
        // 分值 = 时间戳，成员 = 时间戳 + 随机后缀（防止同一毫秒请求覆盖）
        String uniqueMember = now + ":" + UUID.randomUUID().toString().substring(0, 4);

        // Lua 脚本保证原子性
        String luaScript = """
            local key = KEYS[1]
            local window_start = tonumber(ARGV[1])
            local max_req = tonumber(ARGV[2])
            local now_ms = tonumber(ARGV[3])
            local member = ARGV[4]

            -- 移除窗口外的旧记录
            redis.call('ZREMRANGEBYSCORE', key, '-inf', window_start)

            -- 获取当前窗口内的请求数
            local count = redis.call('ZCARD', key)

            if count >= max_req then
                return 0  -- 限流
            end

            -- 添加当前请求
            redis.call('ZADD', key, now_ms, member)
            redis.call('EXPIRE', key, math.ceil((now_ms - window_start) / 1000) + 1)

            return 1  -- 允许
            """;

        long windowStart = now - windowSeconds * 1000L;
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(luaScript, Long.class),
            List.of(key),
            String.valueOf(windowStart),
            String.valueOf(maxRequests),
            String.valueOf(now),
            uniqueMember
        );
        return result != null && result == 1L;
    }

    /**
     * 检测 IP 是否来自数据中心（云服务器/代理）
     * @return true=数据中心IP，高度疑似刷单
     */
    public boolean isDatacenterIp(String ip) {
        try {
            for (SubnetUtils.SubnetInfo subnet : Arrays.stream(datacenterSubnets)
                    .map(SubnetUtils::getInfo).toArray(SubnetUtils.SubnetInfo[]::new)) {
                if (subnet.isInRange(ip)) {
                    return true;
                }
            }
        } catch (Exception e) {
            return false;
        }
        return false;
    }

    /**
     * IP 段聚集检测：同一 IP 段（/24）在短时间内大量请求
     * 如果一个 /24 网段在 10 秒内超过 50 次请求，高度疑似代理集群
     */
    public boolean isIpSegmentConcentrated(String ip) {
        String segment = ip.replaceAll("(\\d+\\.\\d+\\.\\d+)\\.\\d+", "$1");
        String key = "seckill:ip_segment:" + segment;
        Long count = redisTemplate.opsForValue().increment(key);
        if (count != null && count == 1) {
            redisTemplate.expire(key, 10, TimeUnit.SECONDS);
        }
        return count != null && count > 50;
    }
}
```

#### 3. 设备指纹生成与校验

设备指纹是识别多账号同一设备的关键手段。前端采集浏览器特征，服务端生成指纹并比对。

**前端指纹采集（JavaScript）：**

```javascript
/**
 * 设备指纹采集模块
 * 采集浏览器的硬件和软件特征，生成一个相对稳定的指纹
 * 注意：指纹不是100%不变的（浏览器升级、设置变更会改变指纹），
 * 但在秒杀场景下，同一设备短时间内指纹不会变
 */
class DeviceFingerprint {

    static async generate() {
        const components = {};

        // 1. 画布指纹（Canvas Fingerprint）
        // 不同硬件/驱动渲染出的图像有微小差异
        components.canvas = this.getCanvasFingerprint();

        // 2. WebGL 指纹
        components.webgl = this.getWebGLFingerprint();

        // 3. 音频指纹（AudioContext）
        components.audio = await this.getAudioFingerprint();

        // 4. 屏幕特征
        components.screen = {
            width: screen.width,
            height: screen.height,
            colorDepth: screen.colorDepth,
            pixelRatio: window.devicePixelRatio
        };

        // 5. 时区与语言
        components.locale = {
            timezone: Intl.DateTimeFormat().resolvedOptions().timeZone,
            language: navigator.language,
            languages: navigator.languages?.join(',')
        };

        // 6. 已安装的插件
        components.plugins = navigator.plugins?.length || 0;

        // 7. 平台与架构
        components.platform = navigator.platform;
        components.arch = navigator.userAgent.match(/\(([^)]+)\)/)?.[1] || '';

        // 8. 触摸支持
        components.touch = {
            maxTouchPoints: navigator.maxTouchPoints,
            touchEvent: 'ontouchstart' in window
        };

        // 9. 硬件并发数（CPU 核心数）
        components.cores = navigator.hardwareConcurrency || 0;

        // 10. 内存（部分浏览器支持）
        components.memory = navigator.deviceMemory || 0;

        // 将所有特征拼接后哈希
        const rawString = JSON.stringify(components);
        const hash = await this.sha256(rawString);

        return hash;
    }

    static getCanvasFingerprint() {
        const canvas = document.createElement('canvas');
        canvas.width = 200;
        canvas.height = 50;
        const ctx = canvas.getContext('2d');
        ctx.textBaseline = 'top';
        ctx.font = '14px Arial';
        ctx.fillStyle = '#f60';
        ctx.fillRect(125, 1, 62, 20);
        ctx.fillStyle = '#069';
        ctx.fillText('SeckillFP!@#', 2, 15);
        ctx.fillStyle = 'rgba(102,204,0,0.7)';
        ctx.fillText('SeckillFP!@#', 4, 17);
        return canvas.toDataURL();
    }

    static getWebGLFingerprint() {
        try {
            const canvas = document.createElement('canvas');
            const gl = canvas.getContext('webgl');
            const debugInfo = gl.getExtension('WEBGL_debug_renderer_info');
            return {
                vendor: debugInfo ? gl.getParameter(debugInfo.UNMASKED_VENDOR_WEBGL) : '',
                renderer: debugInfo ? gl.getParameter(debugInfo.UNMASKED_RENDERER_WEBGL) : ''
            };
        } catch (e) {
            return { vendor: '', renderer: '' };
        }
    }

    static async getAudioFingerprint() {
        try {
            const ctx = new AudioContext();
            const oscillator = ctx.createOscillator();
            const analyser = ctx.createAnalyser();
            oscillator.type = 'triangle';
            oscillator.frequency.setValueAtTime(10000, ctx.currentTime);
            oscillator.connect(analyser);
            analyser.connect(ctx.createScriptProcessor(4096, 1, 1));
            oscillator.start(0);
            const result = analyser.frequencyBinCount + ':' + ctx.sampleRate;
            oscillator.stop();
            ctx.close();
            return result;
        } catch (e) {
            return '';
        }
    }

    static async sha256(message) {
        const msgBuffer = new TextEncoder().encode(message);
        const hashBuffer = await crypto.subtle.digest('SHA-256', msgBuffer);
        const hashArray = Array.from(new Uint8Array(hashBuffer));
        return hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
    }
}

// 使用：在秒杀请求中附带设备指纹
async function submitSeckillRequest(itemId) {
    const fingerprint = await DeviceFingerprint.generate();
    fetch('/api/seckill/submit/' + dynamicUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
            itemId: itemId,
            fingerprint: fingerprint,
            timestamp: Date.now()
        })
    });
}
```

**服务端指纹校验：**

```java
@Service
public class DeviceFingerprintService {

    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * 校验设备指纹，返回风险等级
     * @param fingerprint 前端传来的设备指纹哈希
     * @param userId      用户ID
     * @return 风险等级：LOW / MEDIUM / HIGH
     */
    public RiskLevel checkFingerprint(String fingerprint, Long userId) {
        if (fingerprint == null || fingerprint.length() != 64) {
            // SHA-256 结果应为 64 位十六进制，长度不对说明被篡改
            return RiskLevel.HIGH;
        }

        // 检查1：同一设备是否关联了多个用户
        String deviceUsersKey = "seckill:device_users:" + fingerprint;
        Long deviceUserCount = redisTemplate.opsForSet().size(deviceUsersKey);
        redisTemplate.opsForSet().add(deviceUsersKey, userId.toString());
        redisTemplate.expire(deviceUsersKey, 1, TimeUnit.HOURS);

        if (deviceUserCount != null && deviceUserCount > 3) {
            // 同一设备关联超过 3 个用户，高度疑似多账号刷单
            return RiskLevel.HIGH;
        } else if (deviceUserCount != null && deviceUserCount > 1) {
            // 同一设备 2-3 个用户，可能是家人共用设备
            return RiskLevel.MEDIUM;
        }

        // 检查2：同一用户是否频繁更换设备
        String userDevicesKey = "seckill:user_devices:" + userId;
        Long userDeviceCount = redisTemplate.opsForSet().size(userDevicesKey);
        redisTemplate.opsForSet().add(userDevicesKey, fingerprint);
        redisTemplate.expire(userDevicesKey, 1, TimeUnit.HOURS);

        if (userDeviceCount != null && userDeviceCount > 2) {
            // 短时间内换了 2+ 台设备，可能是 token 泄露被多设备使用
            return RiskLevel.MEDIUM;
        }

        return RiskLevel.LOW;
    }

    enum RiskLevel {
        LOW,     // 放行
        MEDIUM,  // 要求人机验证
        HIGH     // 直接拦截
    }
}
```

#### 4. 网关风控集成

将上述三大检测能力在网关层统一编排：

```java
@Component
public class SeckillRiskControlFilter implements GlobalFilter, Ordered {

    @Autowired
    private BotDetectionService botDetectionService;

    @Autowired
    private IpRateLimitService ipRateLimitService;

    @Autowired
    private DeviceFingerprintService fingerprintService;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String path = request.getPath().value();

        // 只拦截秒杀提交接口
        if (!path.startsWith("/api/seckill/submit/")) {
            return chain.filter(exchange);
        }

        String ip = extractClientIp(request);
        String fingerprint = request.getHeaders().getFirst("X-Device-Fingerprint");
        String userId = exchange.getAttribute("userId");

        // 第一关：IP 数据中心检测（最快，纯内存判断）
        if (ipRateLimitService.isDatacenterIp(ip)) {
            return reject(exchange, 403, "ACCESS_DENIED", "异常访问环境");
        }

        // 第二关：IP 段聚集检测
        if (ipRateLimitService.isIpSegmentConcentrated(ip)) {
            return reject(exchange, 429, "RATE_LIMITED", "请求过于频繁");
        }

        // 第三关：IP 滑动窗口限流（10 秒内最多 3 次）
        if (!ipRateLimitService.allowRequest(ip, 10, 3)) {
            return reject(exchange, 429, "RATE_LIMITED", "操作过于频繁");
        }

        // 第四关：设备指纹检测
        DeviceFingerprintService.RiskLevel deviceRisk =
            fingerprintService.checkFingerprint(fingerprint, Long.valueOf(userId));
        if (deviceRisk == DeviceFingerprintService.RiskLevel.HIGH) {
            return reject(exchange, 403, "ACCESS_DENIED", "设备异常");
        }
        if (deviceRisk == DeviceFingerprintService.RiskLevel.MEDIUM) {
            // 中风险：要求人机验证
            return requireCaptcha(exchange);
        }

        // 第五关：机器人检测（最重，放在最后）
        SeckillRequestContext ctx = buildContext(request, ip, userId);
        int botScore = botDetectionService.calculateBotScore(ctx);
        if (botScore >= 70) {
            return reject(exchange, 403, "ACCESS_DENIED", "请求异常");
        }
        if (botScore >= 50) {
            return requireCaptcha(exchange);
        }

        // 通过所有风控检查
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -1;  // 在所有过滤器之前执行
    }

    private String extractClientIp(ServerHttpRequest request) {
        String xff = request.getHeaders().getFirst("X-Forwarded-For");
        if (xff != null && !xff.isEmpty()) {
            return xff.split(",")[0].trim();
        }
        String realIp = request.getHeaders().getFirst("X-Real-IP");
        if (realIp != null && !realIp.isEmpty()) {
            return realIp;
        }
        return request.getRemoteAddress().getAddress().getHostAddress();
    }

    private Mono<Void> reject(ServerWebExchange exchange, int status,
                               String code, String message) {
        exchange.getResponse().setStatusCode(HttpStatus.valueOf(status));
        exchange.getResponse().getHeaders().setContentType(MediaType.APPLICATION_JSON);
        String body = String.format("{\"code\":\"%s\",\"message\":\"%s\"}", code, message);
        DataBuffer buffer = exchange.getResponse().bufferFactory().wrap(body.getBytes());
        return exchange.getResponse().writeWith(Mono.just(buffer));
    }

    private Mono<Void> requireCaptcha(ServerWebExchange exchange) {
        exchange.getResponse().setStatusCode(HttpStatus.OK);
        exchange.getResponse().getHeaders().setContentType(MediaType.APPLICATION_JSON);
        String body = "{\"code\":\"CAPTCHA_REQUIRED\",\"message\":\"请完成人机验证\"}";
        DataBuffer buffer = exchange.getResponse().bufferFactory().wrap(body.getBytes());
        return exchange.getResponse().writeWith(Mono.just(buffer));
    }
}
```

### 完整时序图

```
用户A点击抢购
  │
  ├─ 前端抽签：中签 ✓
  │
  ├─ 提交请求 → Nginx
  │                ├─ IP限流检查 ✓
  │                ├─ 总QPS限流检查 ✓
  │                └─ 转发到网关
  │
  ├─ 网关校验
  │   ├─ 登录态检查 ✓
  │   ├─ 资格校验 ✓
  │   ├─ 风控检查 ✓
  │   └─ 转发到秒杀服务
  │
  ├─ 秒杀服务
  │   ├─ Redis Lua 脚本：库存>0 → 扣减成功 ✓
  │   ├─ 写入结果到 Redis：success
  │   └─ 投递消息到 Kafka
  │
  ├─ 前端轮询结果
  │   └─ 读取 Redis：success ✓
  │
  └─ 用户看到"秒杀成功"

  （异步）订单服务消费 Kafka → MySQL 创建订单
```

### 性能与成本计算（详细分析）

#### 各层 QPS 承载能力

| 层级 | 组件 | QPS 承载能力 | 实际峰值 | 水位 | 备注 |
|------|------|-------------|---------|------|------|
| L1 | 前端（浏览器） | 不限 | 50万（原始） | N/A | 前端过滤后降至1万 |
| L2 | CDN | 100万+ | 1万 | 1% | 静态资源走CDN，动态请求透传 |
| L3 | Nginx | 5万（单机） | 1万 | 20% | 2台 Nginx 做 HA |
| L4 | 网关（Spring Cloud Gateway） | 1万（单机） | 3500 | 35% | 3个网关实例 |
| L5 | Redis Cluster（3节点） | 30万 | 1400 | 0.5% | 秒杀只占用极小比例 |
| L6 | Kafka（6分区3副本） | 10万/分区 | 1000 | 0.2% | 远超需求 |
| L7 | MySQL | 5000写/10000读 | 200写/秒 | 4% | 限流消费后压力极小 |

**关键结论：** 系统瓶颈不在任何单层，而在于流量的"漏斗效应"是否生效。只要前端和 Nginx 层的过滤正常工作，后端各层的负载都远低于容量上限。

#### Redis 内存详细计算

单场次秒杀的 Redis 内存占用：

```
1. 库存 key
   key:   "stock:10001"           → 约 12 字节
   value: "1000"                  → 约 4 字节
   overhead（Redis 对象头等）      → 约 80 字节
   小计: ~96 字节

2. 用户去重 Set（最多 1000 成员）
   key:   "seckill:users:10001"   → 约 22 字节
   每个成员: userId 字符串         → 约 20 字节
   1000 成员: 1000 × (20 + 32)    → 约 52KB（含 Set 结构开销）
   小计: ~52KB

3. 预扣记录 Hash（最多 1000 条）
   key:   "seckill:reserved:10001" → 约 27 字节
   每条: field(userId 20B) + value(requestId 40B) → ~60 字节
   1000 条: 1000 × (60 + 32)      → 约 90KB
   小计: ~90KB

4. 结果 key（最多 1000 条成功 + 400 条失败）
   key:   "seckill:result:U12345:10001" → 约 35 字节
   value: "success" / "soldout"         → 约 7 字节
   1400 条: 1400 × (42 + 80)            → 约 170KB
   小计: ~170KB

5. 回滚日志 List
   key:   "seckill:rollback_log:10001"  → 约 33 字节
   假设 50 条回滚: 50 × 100 字节         → ~5KB
   小计: ~5KB

6. URL Token key
   key:   "seckill:url:10001"           → 约 20 字节
   value: 随机 token                     → 约 20 字节
   小计: ~120 字节

单场次总计: ~320KB
10 场次同时进行: ~3.2MB
```

**结论：** 即使同时进行 100 场秒杀，Redis 内存占用也仅约 32MB，远低于 Redis 通常配置的数 GB 内存上限。内存不是瓶颈。

#### Kafka 吞吐量分析

```
Topic 配置：seckill-orders
- 分区数：6
- 副本数：3
- 消息大小：约 200 字节/条（JSON 格式）

单分区吞吐量：
- 生产者：约 5万条/秒（小消息场景）
- 消费者：约 3万条/秒（单消费者）

6 分区总吞吐量：
- 生产：30万条/秒 → 远超 1000 条/场次的需求
- 消费：18万条/秒 → 远超需求

磁盘占用（保留 24 小时）：
- 1000 条 × 200 字节 = 200KB/场次
- 100 场次/天 = 20MB/天
- 3 副本 = 60MB/天
- 保留 24 小时 = 60MB
- 完全可忽略
```

#### 网络带宽计算

```
入站（用户请求）：
- 前端过滤后约 1 万 QPS
- 平均请求大小：1KB（含 Header + Body + Cookie）
- 峰值入站带宽：1万 × 1KB = 10MB/s = 80Mbps

出站（响应 + 轮询）：
- 秒杀提交响应：1万 QPS × 0.5KB = 5MB/s
- 结果轮询：2万 QPS × 0.5KB = 10MB/s
- CDN 静态资源：由 CDN 承载，不经过源站
- 峰值出站带宽：15MB/s = 120Mbps

内网带宽（服务间通信）：
- 秒杀服务 → Redis：1400 QPS × 0.5KB = 0.7MB/s
- 秒杀服务 → Kafka：1000 条 × 0.2KB = 0.2MB/s
- 订单服务 → MySQL：200 QPS × 1KB = 0.2MB/s
- 内网总带宽：约 1.1MB/s = 8.8Mbps → 完全无压力
```

#### 基础设施成本估算（月度）

| 资源 | 规格 | 数量 | 月费 | 备注 |
|------|------|------|------|------|
| Redis Cluster | 4GB 内存/节点 | 3 | $300 | 可复用现有集群 |
| Kafka Cluster | 4核8GB/节点 | 3 | $250 | 复用现有集群 |
| MySQL | 8核16GB RDS | 1 | $200 | 复用现有数据库 |
| Nginx | 2核4GB | 2 | $60 | 与其他服务共用 |
| 网关 | 4核8GB | 3 | $180 | 可弹性扩缩 |
| 秒杀服务 | 4核8GB | 3 | $180 | 可弹性扩缩 |
| 订单服务 | 4核8GB | 2 | $120 | 可弹性扩缩 |
| **合计** | | | **~$1290/月** | **大部分可复用现有资源** |

**秒杀专属增量成本：**

如果大部分基础设施已存在，秒杀系统实际新增成本极低：

```
新增资源：
- 秒杀服务 3 实例（按需，仅大促期间启动）→ 约 $5/天
- 额外 Redis 内存 100MB → 几乎为零
- Kafka 额外 Topic → 几乎为零

结论：秒杀系统的主要成本不在硬件，而在开发和维护人力。
一次大促（3天）的额外基础设施成本 < $50。
```

#### 容量规划：扩展到更大规模

| 场景 | 库存量 | 峰值QPS | Redis节点 | Kafka分区 | MySQL |
|------|--------|---------|-----------|----------|-------|
| 当前（单场次1000件） | 1,000 | 50万 | 3 | 6 | 单机 |
| 中型（10场次×5000件） | 50,000 | 100万 | 6 | 12 | 单机 |
| 大型（100场次×1万件） | 1,000,000 | 300万 | 12 | 24 | 分库（4库） |
| 超大型（双11主会场） | 10,000,000 | 1000万 | 30+ | 60+ | 分库（16库） |

**扩展瓶颈不在 Redis 或 Kafka，而在 MySQL 的写入能力。** 当库存量达到百万级时，需要引入分库分表（按 sessionId 分片）或批量写入（攒批 INSERT）。

## 分布式锁：Redlock 完整实现

在秒杀系统中，某些操作需要分布式锁保护，例如：秒杀场次的预热（防止重复预热）、对账脚本的执行（防止并发对账）、库存回滚（防止并发回滚导致超加）。这里给出基于 Redlock 算法的完整实现。

### 为什么需要分布式锁？

秒杀系统是多实例部署的，单机的 `synchronized` 或 `ReentrantLock` 无法跨实例生效。典型场景：

```
场景：秒杀场次预热
  - 预热操作：将 MySQL 库存写入 Redis + 清除旧数据 + 设置状态
  - 如果两个实例同时执行预热，可能出现：
    实例A 读取 MySQL stock=1000，写入 Redis
    实例B 读取 MySQL stock=1000，写入 Redis（重复操作）
    实例A 清除用户 Set
    实例B 清除用户 Set（重复操作，但无副作用）
  - 虽然这个场景不会造成数据错误，但预热操作包含多次 Redis 写入，
    如果中途被另一个实例打断，可能产生中间状态
```

### Redlock 算法原理

Redlock 由 Redis 作者 Antirez 提出，核心思想是在多个独立的 Redis 实例上同时加锁，只有获得大多数实例的锁才算成功。

```
算法步骤（假设 5 个 Redis 实例，需要获得 3 个以上的锁）：

1. 获取当前时间 T1
2. 依次向 5 个实例发送 SET lock_key unique_value NX PX timeout 命令
3. 计算获取锁耗时：elapsed = T2 - T1
4. 如果成功获得 >= 3 个实例的锁，且 elapsed < 锁的超时时间，则锁获取成功
5. 锁的有效时间 = 原始超时 - elapsed
6. 如果未获得足够的锁，向所有实例发送释放锁请求
```

**为什么不用单节点 Redis 锁？** 单节点故障时锁丢失，可能导致两个实例同时持有锁。Redlock 通过多实例投票解决此问题。

### 完整 Redlock 实现

```java
@Component
public class RedisDistributedLock {

    private final List<JedisPool> jedisPools;  // 5个独立的 Redis 连接池
    private static final int QUORUM = 3;        // 需要获得 3 个以上的锁
    private static final int RETRY_COUNT = 3;   // 加锁重试次数
    private static final long RETRY_DELAY_MS = 200;  // 重试间隔
    private static final long CLOCK_DRIFT_MS = 50;   // 时钟漂移补偿

    public RedisDistributedLock(List<String> redisNodes) {
        // 初始化 5 个独立的 Redis 连接池（非 Cluster，各自独立）
        jedisPools = redisNodes.stream()
            .map(node -> {
                String[] parts = node.split(":");
                return new JedisPool(new JedisPoolConfig(),
                    parts[0], Integer.parseInt(parts[1]));
            })
            .collect(Collectors.toList());
    }

    /**
     * 尝试获取分布式锁
     * @param lockKey    锁的 key
     * @param lockValue  锁的唯一值（用于安全释放，防止误解锁）
     * @param expireMs   锁的超时时间（毫秒）
     * @param timeoutMs  获取锁的超时时间（毫秒）
     * @return 是否成功获取锁
     */
    public boolean tryLock(String lockKey, String lockValue,
                           long expireMs, long timeoutMs) {
        long deadline = System.currentTimeMillis() + timeoutMs;

        for (int retry = 0; retry < RETRY_COUNT; retry++) {
            long lockAcquiredCount = 0;
            long startTime = System.currentTimeMillis();

            // 向所有实例请求加锁
            for (JedisPool pool : jedisPools) {
                try (Jedis jedis = pool.getResource()) {
                    String result = jedis.set(lockKey, lockValue, "NX", "PX", expireMs);
                    if ("OK".equals(result)) {
                        lockAcquiredCount++;
                    }
                } catch (Exception e) {
                    // 单个实例失败不影响整体，继续尝试其他实例
                    log.warn("Redis实例加锁失败", e);
                }
            }

            long elapsed = System.currentTimeMillis() - startTime;

            // 判断是否成功获得大多数锁
            if (lockAcquiredCount >= QUORUM
                && elapsed < expireMs - CLOCK_DRIFT_MS) {
                // 锁获取成功
                log.info("分布式锁获取成功, key={}, acquired={}/{}",
                    lockKey, lockAcquiredCount, jedisPools.size());
                return true;
            }

            // 加锁失败，释放已获得的锁
            unlockInternal(lockKey, lockValue);

            // 检查是否超过获取超时
            if (System.currentTimeMillis() >= deadline) {
                return false;
            }

            // 等待后重试
            try {
                Thread.sleep(Math.min(RETRY_DELAY_MS,
                    deadline - System.currentTimeMillis()));
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return false;
            }
        }

        return false;
    }

    /**
     * 释放分布式锁（使用 Lua 脚本保证原子性）
     * 只有持有锁的客户端才能释放，防止误解锁
     */
    public boolean unlock(String lockKey, String lockValue) {
        return unlockInternal(lockKey, lockValue);
    }

    private boolean unlockInternal(String lockKey, String lockValue) {
        // Lua 脚本：检查值是否匹配，匹配才删除
        String unlockScript = """
            if redis.call('GET', KEYS[1]) == ARGV[1] then
                return redis.call('DEL', KEYS[1])
            else
                return 0
            end
            """;

        int successCount = 0;
        for (JedisPool pool : jedisPools) {
            try (Jedis jedis = pool.getResource()) {
                Object result = jedis.eval(unlockScript,
                    Collections.singletonList(lockKey),
                    Collections.singletonList(lockValue));
                if (Long.valueOf(1L).equals(result)) {
                    successCount++;
                }
            } catch (Exception e) {
                log.warn("Redis实例解锁失败", e);
            }
        }
        return successCount >= 1;  // 释放至少 1 个实例即算成功
    }

    /**
     * 带自动续期的分布式锁（看门狗机制）
     * 适用于执行时间不确定的任务（如对账脚本）
     */
    public AutoRenewLock tryLockWithRenewal(String lockKey, String lockValue,
                                             long expireMs, long timeoutMs) {
        boolean locked = tryLock(lockKey, lockValue, expireMs, timeoutMs);
        if (!locked) {
            return null;
        }

        // 启动续期线程
        ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
        long renewInterval = expireMs / 3;  // 每隔 1/3 过期时间续期一次

        scheduler.scheduleAtFixedRate(() -> {
            for (JedisPool pool : jedisPools) {
                try (Jedis jedis = pool.getResource()) {
                    // 只有值匹配才续期
                    String currentValue = jedis.get(lockKey);
                    if (lockValue.equals(currentValue)) {
                        jedis.pexpire(lockKey, expireMs);
                    }
                } catch (Exception e) {
                    log.warn("锁续期失败", e);
                }
            }
        }, renewInterval, renewInterval, TimeUnit.MILLISECONDS);

        return new AutoRenewLock(lockKey, lockValue, scheduler);
    }

    /**
     * 自动续期锁的包装类
     */
    public static class AutoRenewLock implements AutoCloseable {
        private final String lockKey;
        private final String lockValue;
        private final ScheduledExecutorService scheduler;
        private final RedisDistributedLock lockService;

        public AutoRenewLock(String lockKey, String lockValue,
                              ScheduledExecutorService scheduler) {
            this.lockKey = lockKey;
            this.lockValue = lockValue;
            this.scheduler = scheduler;
            this.lockService = null;  // 由外部注入
        }

        @Override
        public void close() {
            // 停止续期
            scheduler.shutdown();
            try {
                scheduler.awaitTermination(5, TimeUnit.SECONDS);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            // 释放锁
            if (lockService != null) {
                lockService.unlock(lockKey, lockValue);
            }
        }

        public void setLockService(RedisDistributedLock lockService) {
            // 通过 setter 注入，避免构造函数循环依赖
        }
    }
}
```

### 分布式锁在秒杀系统中的应用场景

```java
@Service
public class SeckillLockService {

    @Autowired
    private RedisDistributedLock distributedLock;

    /**
     * 场景1：秒杀场次预热（防止多实例重复预热）
     */
    public boolean warmUpSession(Long sessionId) {
        String lockKey = "lock:warmup:" + sessionId;
        String lockValue = UUID.randomUUID().toString();

        try {
            if (distributedLock.tryLock(lockKey, lockValue, 30000, 5000)) {
                // 获得锁，执行预热
                doWarmUp(sessionId);
                return true;
            }
            return false;  // 另一个实例正在预热
        } finally {
            distributedLock.unlock(lockKey, lockValue);
        }
    }

    /**
     * 场景2：对账脚本执行（防止并发对账）
     * 对账可能执行较长时间，使用带续期的锁
     */
    public void runReconciliation(Long sessionId) {
        String lockKey = "lock:reconcile:" + sessionId;
        String lockValue = UUID.randomUUID().toString();

        try (RedisDistributedLock.AutoRenewLock lock =
                distributedLock.tryLockWithRenewal(lockKey, lockValue, 30000, 5000)) {
            if (lock != null) {
                doReconcile(sessionId);
            } else {
                log.info("对账脚本已在其他实例执行, sessionId={}", sessionId);
            }
        }
    }

    /**
     * 场景3：库存回滚（防止并发回滚导致库存超加）
     *
     * 注意：正常流程下的库存回滚由 Lua 脚本保证原子性，不需要分布式锁。
     * 这里针对的是"补偿回滚"场景：对账发现差异后需要修正，可能多个
     * 管理员或多个定时任务同时触发修正。
     */
    public boolean compensateStock(Long sessionId, int delta, String reason) {
        String lockKey = "lock:compensate:" + sessionId;
        String lockValue = UUID.randomUUID().toString();

        try {
            if (distributedLock.tryLock(lockKey, lockValue, 10000, 3000)) {
                doCompensate(sessionId, delta, reason);
                return true;
            }
            return false;
        } finally {
            distributedLock.unlock(lockKey, lockValue);
        }
    }
}
```

### Redlock 的争议与取舍

**Martin Kleppmann 的批评：** Redlock 依赖系统时钟的准确性，如果某个 Redis 实例的时钟发生跳跃（如 NTP 同步），可能导致锁提前过期。此外，进程暂停（GC、SWAP）也可能导致持有锁的客户端误以为锁仍有效。

**Antirez 的回应：** Redlock 通过 `CLOCK_DRIFT_MS` 补偿时钟漂移，且锁的超时时间远大于 GC 暂停时间（30 秒 vs 几百毫秒），在实际场景中足够安全。

**秒杀场景的取舍：**

```
秒杀系统中使用分布式锁的场景特点：
1. 锁的持有时间短（预热 < 30秒，对账 < 60秒）
2. 锁的粒度粗（按场次加锁，不是按用户加锁）
3. 竞争不激烈（只有 3-5 个实例参与竞争）
4. 后果不严重（即使锁失效导致并发预热，最坏结果是重复操作，不会超卖）

结论：在秒杀系统中，Redlock 足够安全。
真正需要强一致性的库存扣减操作，已由 Redis Lua 脚本保证原子性，不依赖分布式锁。
```

## 常见陷阱（深度分析）

### 陷阱 1：库存直接扣 MySQL

**错误做法：** `UPDATE inventory SET stock = stock - 1 WHERE item_id = ?`

**后果：** 50 万 QPS 的 UPDATE 请求打到 MySQL。MySQL 单机写入上限约 5000 QPS，50 万 QPS 意味着：
- 连接池瞬间耗尽
- InnoDB 行锁竞争导致大量线程等待
- 最终数据库 OOM 或磁盘 IO 打满，全站不可用

**为什么不能加缓存解决？** 缓存可以缓解读压力，但扣减库存是写操作，必须保证一致性，缓存无法替代。

### 陷阱 2：Redis DECR 不用 Lua

**错误做法：**
```python
stock = redis.get("stock:item123")  # 步骤1
if int(stock) > 0:
    redis.decr("stock:item123")     # 步骤2
```

**超卖构造：**
```
时刻 T1: 请求A 执行步骤1，读到 stock=1
时刻 T2: 请求B 执行步骤1，读到 stock=1
时刻 T3: 请求A 执行步骤2，stock 变为 0
时刻 T4: 请求B 执行步骤2，stock 变为 -1  ← 超卖！
```

### 陷阱 3：前端不过滤，后端扛全量

**错误逻辑：** "前端过滤可以被绕过，所以不值得做"

**反驳：** 前端过滤的目的不是安全，而是降低后端压力。即使 10% 的用户绕过了前端过滤（使用脚本），后端流量也只有 5 万 QPS 而非 50 万 QPS。安全性由后端限流和风控保证，但前端过滤是性价比最高的降流手段。

### 陷阱 4：忽略对账

**风险：** 如果 Redis 预扣了 1000 次但 Kafka 消息丢失了 10 条，MySQL 只创建了 990 笔订单。结果：10 个用户看到"秒杀成功"但没有订单。

**对账方案**已在上方详述。关键原则：**Redis 的扣减结果不是最终真相，MySQL 的订单记录才是。**

### 陷阱 5：秒杀页面静态化导致库存状态不准

**错误做法：** 将秒杀页面完全静态化推到 CDN，不包含任何动态库存信息。用户看到"有货"但实际已售罄，大量无效请求涌入。

**正确做法：** 秒杀页面 HTML 骨架走 CDN 缓存，但库存状态通过独立的轻量 API 实时获取。该 API 只返回一个布尔值（有货/无货），由 Redis 直接响应，QPS 承受能力极强。

```
秒杀页面加载：
  1. CDN 返回 HTML 骨架（缓存 10 分钟）
  2. 前端并行请求 /api/seckill/stock_status?itemId=123
  3. 服务端：GET stock:item123 → 返回 {"hasStock": true/false}
  4. 前端根据返回值显示按钮状态

优势：
  - 库存状态接口极轻（1 次 Redis GET），可承受百万级 QPS
  - 售罄后按钮立即灰显，避免无效提交
  - HTML 缓存不受库存变化影响，CDN 命中率高
```

### 陷阱 6：秒杀接口返回库存余量

**风险：** 接口返回当前库存余量（如"还剩 3 件"），黄牛可以通过高频轮询判断库存消耗速度，在库存即将耗尽时精准投入大量请求。

**正确做法：** 接口只返回"有货/无货"的布尔值，不暴露具体库存数量。即使需要显示库存，也只显示粗粒度信息（如"库存紧张"而非"还剩 3 件"）。

### 陷阱 7：忽略秒杀结束后的资源清理

**风险：** 秒杀结束后，Redis 中的库存 key、用户去重 Set、结果 key 等数据如果不清理，会持续占用内存。长期积累后可能导致 Redis 内存不足。

**正确做法：** 所有秒杀相关的 Redis key 必须设置 TTL，并在秒杀结束后主动清理。

```
资源清理策略：
1. 短期 key（结果查询、URL Token）：TTL 5 分钟，自动过期
2. 中期 key（库存、用户去重 Set）：TTL 30 分钟，秒杀结束后 30 分钟自动清理
3. 长期 key（预扣记录 Hash）：TTL 15 分钟，与支付超时对齐
4. 秒杀结束后运行清理脚本：主动 DEL 所有相关 key，不等 TTL

清理脚本：
  1. 查询已结束的秒杀场次（status=2 且 end_time < NOW() - 30分钟）
  2. 批量删除：stock:{sessionId}, seckill:users:{sessionId}, seckill:reserved:{sessionId}
  3. 更新场次状态为"已清理"
```

## 延伸思考

- **如果秒杀商品不是 1000 件而是 100 万件**（库存充足型秒杀）：前端抽签概率应提高（不需要过滤 98%），但后端 MySQL 的写入压力会增大（100 万次 INSERT），需要分库分表或批量写入
- **秒杀后 15 分钟未支付自动释放库存**：使用延迟消息队列（RocketMQ 延迟消息 / Redis 延迟队列），到期后检查订单状态，未支付则取消订单并 Redis `INCR` 恢复库存
- **多机房部署**：Redis Cluster 跨机房同步有延迟（主从复制 1-5ms），但秒杀请求通常有地域性（北京用户访问北京机房），可以在每个机房维护独立的库存配额（如北京 400 件、上海 300 件、广州 300 件），各机房独立扣减

## 完整系统运行演练

### 正常流程：从用户点击到订单创建

以下是一次完整秒杀请求从发起到完成的全链路时间线分析：

```
T=0ms    用户点击"抢购"按钮
  │
  ├─ 前端抽签判定：耗时 < 1ms（纯本地计算）
  │  → 中签概率 2%，98% 的用户在此结束
  │
  ├─ 前端构造请求：耗时 1-2ms
  │  → 附带设备指纹、时间戳、动态 URL Token
  │
T=2ms    请求发出，到达 CDN/Nginx
  │
  ├─ CDN 透传：耗时 ~5ms（网络传输）
  │
  ├─ Nginx 限流检查：耗时 < 1ms
  │  → IP限流 + 总QPS限流，超出部分立即返回 429
  │
T=8ms    请求到达网关
  │
  ├─ 登录态校验：耗时 ~2ms（JWT 解析，无远程调用）
  ├─ 资格校验：耗时 ~3ms（Redis 查询黑名单 + 参与记录）
  ├─ 风控检查：耗时 ~5ms（IP检测 + 设备指纹 + 机器人评分）
  │  → 中风险用户额外需要 1-2 秒完成人机验证
  │
T=18ms   请求到达秒杀服务
  │
  ├─ Redis Lua 预扣：耗时 ~1ms（单次 Lua 执行）
  │  → 前 1000 次成功，后续返回库存不足
  │  → 成功的用户写入 "processing" 状态到 Redis
  │
  ├─ 构造 Kafka 消息：耗时 ~1ms
  ├─ Kafka 投递：耗时 ~5ms（网络传输 + Broker 确认）
  │  → 投递失败时回滚 Redis 库存
  │
T=25ms   秒杀服务返回 "正在处理中"
  │
  ├─ 前端收到响应：耗时 ~5ms（网络回传）
  │
T=30ms   前端开始轮询结果
  │
  ├─ 第1次轮询（500ms后）：seckill:result → "processing"（订单还在创建）
  ├─ 第2次轮询（1000ms后）：seckill:result → "processing"（MySQL 写入进行中）
  ├─ 第3次轮询（1500ms后）：seckill:result → "success"（订单已创建）
  │
T~1500ms 用户看到"秒杀成功"页面

（异步）订单消费流程：
T=25ms   → T~200ms  Kafka 消费者拉取消息
T~200ms  → T~250ms  创建订单（MySQL INSERT）
T~250ms  → T~300ms  扣减 MySQL 库存（MySQL UPDATE）
T~300ms  → T~350ms  更新 Redis 结果为 "success"
```

**总延迟分析：**
- 幸运用户（第一次轮询就查到结果）：约 500-1000ms
- 正常用户：约 1500-2000ms
- 最慢用户（MySQL 写入在消费队列中排队）：约 3000ms
- 超时用户（6 次轮询都没查到结果）：提示"查询超时，请稍后在订单列表查看"

### 全链路压测方案

```
压测目标：验证系统在 50 万 QPS 下的各层过滤效果

1. 前端模拟层：
   - 使用 50 万个虚拟用户，每个用户有独立 userId 和设备指纹
   - 模拟正常用户的点击行为（间隔 1-5 秒，随机分布）
   - 模拟脚本用户（间隔 < 100ms，固定分布，无浏览器特征）

2. Nginx 层：
   - 使用 wrk 或 JMeter 发起 50 万 QPS 的 HTTP 请求
   - 验证 limit_req 是否正确限流到 7000 QPS
   - 验证 429 响应的比例

3. 网关层：
   - 模拟各种风控场景：数据中心 IP、多账号同设备、无 UA 请求
   - 验证风控评分和拦截比例

4. Redis 层：
   - 使用 redis-benchmark 模拟 7000 QPS 的 Lua 脚本执行
   - 验证原子性和幂等性
   - 模拟 Redis 故障转移

5. Kafka 层：
   - 模拟 1000 条消息的投递和消费
   - 验证消费速率和幂等性

6. MySQL 层：
   - 模拟 200 QPS 的订单创建
   - 验证唯一约束的幂等效果

7. 全链路：
   - 从前端模拟到 MySQL 创建的完整链路
   - 验证各层过滤比例是否符合预期
   - 验证最终订单数是否等于 1000（无超卖）
```

### 系统监控与告警

```
秒杀系统需要监控的关键指标：

1. 流量指标
   - 原始请求 QPS（Nginx 统计）
   - 通过各层过滤后的 QPS（网关统计）
   - 429 响应数量和比例

2. Redis 指标
   - Lua 脚本执行次数和延迟
   - 库存扣减成功/失败比例
   - Redis 内存使用量
   - Redis 连接数

3. Kafka 指标
   - 消息生产速率和延迟
   - 消费者 lag（积压量）
   - 消息投递成功率

4. MySQL 指标
   - 活跃连接数
   - 写入 QPS
   - 慢查询数量
   - 行锁等待时间

5. 业务指标
   - 库存剩余数量
   - 订单创建成功率
   - 支付转化率
   - 支付超时率

告警规则：
   - Redis 库存 <= 50 且仍有请求 → 即将售罄告警
   - Kafka lag > 500 → 消费积压告警
   - MySQL 写入 QPS > 1000 → 数据库压力告警
   - 订单创建成功率 < 95% → 业务异常告警
   - 支付超时率 > 30% → 超时率告警
```
## 限流与降级策略完整实现

```python
class FlashSaleRateLimiter:
    """秒杀限流：多级限流策略"""

    def check_access(self, user_id, activity_id):
        """多级限流检查"""
        # Level 1: 全局活动限流（令牌桶）
        global_key = f"rate_limit:activity:{activity_id}"
        global_count = self.redis.incr(global_key)
        if global_count == 1:
            self.redis.expire(global_key, 1)
        if global_count > 50000:  # 5万 QPS 全局上限
            return {"allowed": False, "reason": "system_busy"}

        # Level 2: 用户级限流（滑动窗口）
        user_key = f"rate_limit:user:{user_id}:{activity_id}"
        user_count = self.redis.incr(user_key)
        if user_count == 1:
            self.redis.expire(user_key, 1)
        if user_count > 5:  # 每用户每秒最多 5 次
            return {"allowed": False, "reason": "too_frequent"}

        # Level 3: 黑名单检查（刷单用户）
        if self.redis.sismember("blacklist", user_id):
            return {"allowed": False, "reason": "banned"}

        return {"allowed": True}

    def degrade_on_overload(self, activity_id):
        """过载降级策略"""
        current_qps = self.monitor.get_qps(activity_id)
        if current_qps > 100000:
            # 极端过载 → 排队模式
            self.redis.set(f"degrade:{activity_id}", "queue_mode", ex=60)
            return "queue_mode"
        elif current_qps > 50000:
            # 过载 → 验证码模式
            self.redis.set(f"degrade:{activity_id}", "captcha_mode", ex=60)
            return "captcha_mode"
        return "normal"
```

## 库存超卖防护完整实现

```python
class StockSafetyGuard:
    """库存超卖防护"""

    def deduct_stock(self, activity_id, sku_id, quantity):
        """安全扣减库存"""
        # 1. Redis 原子预扣减（DECRBY）
        stock_key = f"stock:{activity_id}:{sku_id}"
        remaining = self.redis.decrby(stock_key, quantity)
        if remaining < 0:
            # 库存不足 → 回滚预扣减
            self.redis.incrby(stock_key, quantity)
            return {"success": False, "reason": "out_of_stock"}

        # 2. 异步同步到数据库
        self.db.execute(
            "UPDATE flash_sale_stock SET available = available - %s "
            "WHERE activity_id = %s AND sku_id = %s AND available >= %s",
            quantity, activity_id, sku_id, quantity)

        # 3. 如果数据库扣减失败（乐观锁）→ 回滚 Redis
        if self.db.rowcount == 0:
            self.redis.incrby(stock_key, quantity)
            return {"success": False, "reason": "stock_conflict"}

        return {"success": True, "remaining": remaining}
```

## 异常场景补充

### 场景：Redis 库存与数据库不一致

```
触发：Redis 预扣减成功但数据库扣减失败 → 数据不一致
检测：
  1. 定期对账：Redis stock vs DB stock
  2. 差异 > 10 → 告警
处理：
  1. 以数据库为准 → 修正 Redis
  2. 记录差异日志用于排查
  3. 活动结束后做最终对账
预防：每秒对账 + 异步修复机制
```

### 场景：秒杀链接提前泄露

```
触发：秒杀 URL 在活动开始前被泄露 → 提前涌入流量
检测：
  1. 活动开始前收到该活动的请求 → 链接泄露
  2. 非预期 QPS 增长 → 告警
处理：
  1. 活动链接包含动态 token（每小时更换）
  2. 活动未开始 → 所有请求返回"活动未开始"
  3. 更换泄露的 URL token
预防：动态 URL token + 时间窗口校验
```

### 场景：用户重复下单

```
触发：用户快速点击两次 → 创建两个订单
检测：
  1. 同一用户同一活动 → 幂等键检查
  2. order_id = hash(user_id + activity_id + sku_id)
处理：
  1. 创建订单前检查是否已有待支付订单
  2. 已有 → 返回已有订单（不重复创建）
  3. 前端防抖 + 后端幂等键双重保护
预防：幂等键 UNIQUE 约束 + 前端点击防抖
```

## 订单创建幂等完整实现

```python
class IdempotentOrderCreator:
    """幂等订单创建：防止重复下单"""

    def create_order(self, user_id, activity_id, sku_id, quantity):
        """幂等创建订单"""
        # 1. 生成幂等键
        idempotency_key = hashlib.md5(
            f"{user_id}:{activity_id}:{sku_id}".encode()).hexdigest()

        # 2. 检查幂等键是否已存在
        existing = self.redis.get(f"idempotent:{idempotency_key}")
        if existing:
            # 已有订单 → 返回已有结果
            return {"order_id": existing, "status": "already_exists"}

        # 3. 创建订单（在事务中）
        with self.db.transaction():
            try:
                order_id = str(uuid4())
                self.db.insert("orders", {
                    "order_id": order_id,
                    "user_id": user_id,
                    "activity_id": activity_id,
                    "sku_id": sku_id,
                    "quantity": quantity,
                    "status": "pending_payment",
                    "created_at": now(),
                    "expire_at": now() + timedelta(minutes=15)  # 15分钟未支付自动取消
                })

                # 4. 扣减库存（乐观锁）
                affected = self.db.execute(
                    "UPDATE flash_sale_stock SET available = available - %s "
                    "WHERE activity_id = %s AND sku_id = %s AND available >= %s",
                    quantity, activity_id, sku_id, quantity)
                if affected == 0:
                    raise StockNotEnoughError("库存不足")

                # 5. 写入幂等键（同一事务）
                self.db.insert("order_idempotency", {
                    "idempotency_key": idempotency_key,
                    "order_id": order_id,
                    "created_at": now()
                })

            except IntegrityError:
                # 幂等键唯一约束冲突 → 订单已存在
                existing_order = self.db.query_one(
                    "SELECT order_id FROM order_idempotency "
                    "WHERE idempotency_key = %s", idempotency_key)
                return {"order_id": existing_order["order_id"], "status": "already_exists"}

        # 6. 缓存幂等键到 Redis
        self.redis.setex(f"idempotent:{idempotency_key}", 3600, order_id)

        return {"order_id": order_id, "status": "created"}

    def cancel_expired_orders(self):
        """自动取消超时未支付订单"""
        expired = self.db.query(
            "SELECT * FROM orders WHERE status = 'pending_payment' "
            "AND expire_at <= NOW()")
        for order in expired:
            # 1. 恢复库存
            self.db.execute(
                "UPDATE flash_sale_stock SET available = available + %s "
                "WHERE activity_id = %s AND sku_id = %s",
                order["quantity"], order["activity_id"], order["sku_id"])
            self.redis.incrby(f"stock:{order['activity_id']}:{order['sku_id']}",
                order["quantity"])
            # 2. 取消订单
            self.db.update("orders",
                {"status": "cancelled", "cancelled_at": now()},
                {"order_id": order["order_id"]})
```

## 反机器人系统

```python
class AntiBotSystem:
    """反机器人：设备指纹 + 行为分析 + CAPTCHA"""

    def check_request(self, request):
        """检查请求是否来自机器人"""
        score = 0  # 风险分：0=安全，100=确定机器人

        # 1. 设备指纹检查
        fingerprint = request.headers.get("X-Device-Fingerprint")
        if not fingerprint:
            score += 30  # 无指纹 → 可疑

        # 2. 请求间隔检查（同一设备）
        last_request = self.redis.get(f"last_req:{fingerprint}")
        if last_request:
            interval = now().timestamp() - float(last_request)
            if interval < 0.1:  # 100ms 内重复请求 → 机器人
                score += 50
        self.redis.setex(f"last_req:{fingerprint}", 60, str(now().timestamp()))

        # 3. IP 信誉检查
        ip = request.client_ip
        ip_rep = self.redis.get(f"ip_rep:{ip}")
        if ip_rep == "bad":
            score += 40

        # 4. 行为模式分析
        click_pattern = self.redis.get(f"clicks:{fingerprint}")
        if click_pattern and self._is_machine_pattern(json.loads(click_pattern)):
            score += 30

        # 判定
        if score >= 80:
            return {"action": "block", "reason": "bot_detected", "score": score}
        elif score >= 50:
            return {"action": "captcha", "reason": "suspicious", "score": score}
        return {"action": "allow", "score": score}

    def _is_machine_pattern(self, clicks):
        """检测点击模式是否为机器（间隔过于均匀）"""
        if len(clicks) < 5:
            return False
        intervals = [clicks[i+1] - clicks[i] for i in range(len(clicks)-1)]
        avg = statistics.mean(intervals)
        std = statistics.stdev(intervals)
        return std / avg < 0.05  # 变异系数 < 5% → 机器
```

## 异常场景补充

### 场景：幂等键冲突

```
触发：两个不同用户生成相同的幂等键 → 第二个用户无法下单
原因：hash(user_id + activity_id + sku_id) 碰撞
概率：MD5 碰撞概率极低，但用户 ID 格式异常可能增加概率
处理：
  1. 幂等键增加更多维度（timestamp 窗口）
  2. 冲突时检查：如果已有订单属于同一用户 → 返回已有
  3. 如果属于不同用户 → 重新生成幂等键（加随机盐）
预防：幂等键包含足够唯一信息 + 冲突检测
```

### 场景：反机器人误杀

```
触发：正常用户被反机器人系统拦截 → 无法参与秒杀
检测：
  1. 用户申诉"无法下单" → 误杀检查
  2. CAPTCHA 通过但仍被拦截 → 系统过严
处理：
  1. 降低风险阈值（80 → 90）
  2. 增加 CAPTCHA 通道：被拦截用户可通过 CAPTCHA 解除
  3. 人工审核：标记误杀案例 → 调整规则
预防：风险阈值动态调整 + CAPTCHA 解除机制 + 误杀率监控
```

## 订单创建流水线（完整幂等实现）

```python
import hashlib
import json
import time
from datetime import datetime, timedelta
from typing import Dict, Optional, Tuple
from enum import Enum
from uuid import uuid4


class OrderStatus(Enum):
    PENDING_PAYMENT = "pending_payment"   # 待支付
    PAID = "paid"                         # 已支付
    SHIPPED = "shipped"                   # 已发货
    CANCELLED = "cancelled"               # 已取消
    CANCELLED_TIMEOUT = "cancelled_timeout"  # 超时取消
    REFUNDED = "refunded"                 # 已退款


class OrderCreationPipeline:
    """订单创建完整流水线：幂等键 + Redis 去重 + DB 唯一约束 + 支付超时自动取消"""

    def __init__(self, redis_client, db_client, mq_producer, stock_service):
        self.redis = redis_client
        self.db = db_client
        self.mq = mq_producer
        self.stock = stock_service
        self.pay_timeout_min = 15

    def create_order(self, user_id: int, activity_id: int, sku_id: int,
                     quantity: int, idempotency_key: str = None,
                     request_context: Dict = None) -> Dict:
        """创建订单（完整幂等流程）

        幂等保证层级：
        1. 客户端传入 idempotency_key（推荐）
        2. Redis 去重（快速路径）
        3. DB 唯一约束（最终保证）
        """
        # 1. 生成幂等键（客户端未提供则自动生成）
        if not idempotency_key:
            idempotency_key = self._generate_idempotency_key(
                user_id, activity_id, sku_id)

        # 2. Redis 快速去重检查
        redis_key = f"order:idempotent:{idempotency_key}"
        existing_order_id = self.redis.get(redis_key)
        if existing_order_id:
            order = self.db.query(
                "SELECT * FROM orders WHERE order_id = %s",
                existing_order_id.decode() if isinstance(existing_order_id, bytes) else existing_order_id)
            if order:
                return {
                    "order_id": order["order_id"],
                    "status": order["status"],
                    "is_duplicate": True,
                    "message": "订单已存在",
                }

        # 3. 检查是否有同用户同活动的待支付订单
        pending = self.db.query(
            "SELECT * FROM orders WHERE user_id = %s AND activity_id = %s "
            "AND sku_id = %s AND status = 'pending_payment' LIMIT 1",
            user_id, activity_id, sku_id)
        if pending:
            return {
                "order_id": pending["order_id"],
                "status": "pending_payment",
                "is_duplicate": True,
                "message": "已有待支付订单",
            }

        # 4. 创建订单（事务内保证原子性）
        order_id = str(uuid4())
        now = datetime.now()
        expire_at = now + timedelta(minutes=self.pay_timeout_min)

        try:
            with self.db.transaction():
                # 4a. 插入订单记录
                self.db.insert("orders", {
                    "order_id": order_id,
                    "user_id": user_id,
                    "activity_id": activity_id,
                    "sku_id": sku_id,
                    "quantity": quantity,
                    "status": OrderStatus.PENDING_PAYMENT.value,
                    "idempotency_key": idempotency_key,
                    "expire_at": expire_at,
                    "created_at": now,
                })

                # 4b. 插入幂等记录（唯一约束保证）
                self.db.insert("order_idempotency", {
                    "idempotency_key": idempotency_key,
                    "order_id": order_id,
                    "user_id": user_id,
                    "created_at": now,
                })

                # 4c. 扣减数据库库存（乐观锁）
                affected = self.db.execute(
                    "UPDATE flash_sale_stock SET available = available - %s "
                    "WHERE activity_id = %s AND sku_id = %s AND available >= %s",
                    quantity, activity_id, sku_id, quantity)
                if affected == 0:
                    raise StockNotEnoughError("库存不足")

        except IntegrityError as e:
            # DB 唯一约束冲突 → 订单已存在
            existing = self.db.query(
                "SELECT order_id FROM order_idempotency "
                "WHERE idempotency_key = %s", idempotency_key)
            if existing:
                return {
                    "order_id": existing["order_id"],
                    "status": "already_exists",
                    "is_duplicate": True,
                }
            raise

        # 5. 缓存幂等键到 Redis（TTL = 支付超时 + 缓冲）
        self.redis.setex(redis_key, self.pay_timeout_min * 60 + 300, order_id)

        # 6. 发送支付超时延迟消息
        self.mq.send_delay_message(
            topic="order-timeout-check",
            key=order_id,
            body={"order_id": order_id, "expire_at": expire_at.isoformat()},
            delay_ms=self.pay_timeout_min * 60 * 1000,
        )

        # 7. 发送订单创建事件
        self.mq.send("order-created", {
            "order_id": order_id,
            "user_id": user_id,
            "activity_id": activity_id,
            "sku_id": sku_id,
            "expire_at": expire_at.isoformat(),
        })

        return {
            "order_id": order_id,
            "status": OrderStatus.PENDING_PAYMENT.value,
            "is_duplicate": False,
            "expire_at": expire_at.isoformat(),
            "pay_timeout_min": self.pay_timeout_min,
        }

    def cancel_expired_orders(self):
        """自动取消超时未支付订单（定时任务，每分钟执行）"""
        now = datetime.now()
        expired_orders = self.db.query(
            "SELECT * FROM orders WHERE status = 'pending_payment' "
            "AND expire_at <= %s LIMIT 500", now)

        cancelled_count = 0
        for order in expired_orders:
            try:
                with self.db.transaction():
                    # 1. 更新订单状态
                    self.db.update("orders", {
                        "status": OrderStatus.CANCELLED_TIMEOUT.value,
                        "cancelled_at": now,
                        "cancel_reason": "payment_timeout",
                    }, {"order_id": order["order_id"]})

                    # 2. 恢复数据库库存
                    self.db.execute(
                        "UPDATE flash_sale_stock SET available = available + %s "
                        "WHERE activity_id = %s AND sku_id = %s",
                        order["quantity"], order["activity_id"], order["sku_id"])

                # 3. 恢复 Redis 库存
                self.redis.incrby(
                    f"stock:{order['activity_id']}:{order['sku_id']}",
                    order["quantity"])

                # 4. 删除幂等键缓存（允许用户重新下单）
                idempotency_key = order.get("idempotency_key")
                if idempotency_key:
                    self.redis.delete(f"order:idempotent:{idempotency_key}")

                # 5. 发送订单取消通知
                self.mq.send("order-cancelled", {
                    "order_id": order["order_id"],
                    "user_id": order["user_id"],
                    "reason": "payment_timeout",
                })

                cancelled_count += 1

            except Exception as e:
                # 单个订单取消失败不影响其他订单
                self._log_cancel_error(order["order_id"], str(e))

        return {"cancelled_count": cancelled_count}

    def pay_order(self, order_id: str, payment_info: Dict) -> Dict:
        """订单支付"""
        order = self.db.query("SELECT * FROM orders WHERE order_id = %s", order_id)
        if not order:
            return {"status": "error", "reason": "order_not_found"}

        if order["status"] != OrderStatus.PENDING_PAYMENT.value:
            return {"status": "error", "reason": f"invalid_status:{order['status']}"}

        if datetime.now() > order["expire_at"]:
            return {"status": "error", "reason": "order_expired"}

        # 更新订单状态
        self.db.update("orders", {
            "status": OrderStatus.PAID.value,
            "paid_at": datetime.now(),
            "payment_info": json.dumps(payment_info),
        }, {"order_id": order_id})

        return {"status": "paid", "order_id": order_id}

    def _generate_idempotency_key(self, user_id: int, activity_id: int,
                                   sku_id: int) -> str:
        """生成幂等键"""
        raw = f"{user_id}:{activity_id}:{sku_id}:{time.strftime('%Y%m%d%H')}"
        return hashlib.md5(raw.encode()).hexdigest()

    def _log_cancel_error(self, order_id: str, error: str):
        """记录取消失败日志"""
        self.redis.lpush("order_cancel_errors", json.dumps({
            "order_id": order_id,
            "error": error,
            "timestamp": datetime.now().isoformat(),
        }))
```

## 秒杀活动管理系统

```python
import json
import time
from datetime import datetime, timedelta
from typing import Dict, List, Optional
from enum import Enum
from dataclasses import dataclass, field


class ActivityStatus(Enum):
    DRAFT = "draft"             # 草稿
    PREVIEW = "preview"         # 预览（仅管理员可见）
    ACTIVE = "active"           # 进行中
    ENDED = "ended"             # 已结束
    CANCELLED = "cancelled"     # 已取消


@dataclass
class FlashSaleActivity:
    """秒杀活动"""
    activity_id: int
    name: str
    status: ActivityStatus
    sku_id: int
    total_stock: int
    seckill_price: float        # 秒杀价
    original_price: float       # 原价
    start_time: datetime
    end_time: datetime
    max_per_user: int = 1       # 每人限购
    preallocated_stock: Dict[int, int] = field(default_factory=dict)  # 预分配库存


class FlashSaleActivityManager:
    """秒杀活动管理：生命周期、库存预分配、结果通知"""

    def __init__(self, db_client, redis_client, mq_producer, notify_service):
        self.db = db_client
        self.redis = redis_client
        self.mq = mq_producer
        self.notify = notify_service

    def create_activity(self, name: str, sku_id: int, total_stock: int,
                        seckill_price: float, original_price: float,
                        start_time: datetime, end_time: datetime,
                        max_per_user: int = 1) -> Dict:
        """创建秒杀活动（草稿状态）"""
        activity_id = self.db.insert("flash_sale_activities", {
            "name": name,
            "status": ActivityStatus.DRAFT.value,
            "sku_id": sku_id,
            "total_stock": total_stock,
            "available_stock": total_stock,
            "seckill_price": seckill_price,
            "original_price": original_price,
            "start_time": start_time,
            "end_time": end_time,
            "max_per_user": max_per_user,
            "created_at": datetime.now(),
        })

        return {"activity_id": activity_id, "status": "draft"}

    def preview_activity(self, activity_id: int) -> Dict:
        """预览活动（管理员审核用）"""
        activity = self._get_activity(activity_id)
        if not activity:
            return {"error": "activity_not_found"}

        # 切换到预览状态
        self.db.update("flash_sale_activities", {
            "status": ActivityStatus.PREVIEW.value,
        }, {"activity_id": activity_id})

        # 生成预览链接（含动态 token，防泄露）
        preview_token = self._generate_preview_token(activity_id)
        preview_url = f"https://app.example.com/seckill/preview?token={preview_token}"

        return {
            "activity_id": activity_id,
            "status": "preview",
            "preview_url": preview_url,
            "name": activity["name"],
            "total_stock": activity["total_stock"],
            "seckill_price": activity["seckill_price"],
            "start_time": activity["start_time"].isoformat(),
        }

    def activate_activity(self, activity_id: int) -> Dict:
        """激活活动（进入进行中状态）"""
        activity = self._get_activity(activity_id)
        if not activity:
            return {"error": "activity_not_found"}

        if activity["status"] not in (ActivityStatus.DRAFT.value,
                                       ActivityStatus.PREVIEW.value):
            return {"error": f"invalid_status:{activity['status']}"}

        now = datetime.now()

        # 1. 预分配库存到 Redis
        self._preallocate_stock(activity_id, activity["total_stock"],
                                 activity["sku_id"])

        # 2. 更新活动状态
        self.db.update("flash_sale_activities", {
            "status": ActivityStatus.ACTIVE.value,
            "activated_at": now,
        }, {"activity_id": activity_id})

        # 3. 缓存活动配置到 Redis（供网关快速读取）
        config_key = f"seckill:config:{activity_id}"
        self.redis.setex(config_key, 86400, json.dumps({
            "activity_id": activity_id,
            "sku_id": activity["sku_id"],
            "total_stock": activity["total_stock"],
            "seckill_price": activity["seckill_price"],
            "max_per_user": activity["max_per_user"],
            "start_time": activity["start_time"].isoformat(),
            "end_time": activity["end_time"].isoformat(),
        }))

        # 4. 发送活动开始通知
        self.notify.broadcast(activity_id, {
            "type": "seckill_start",
            "activity_id": activity_id,
            "name": activity["name"],
            "seckill_price": activity["seckill_price"],
        })

        return {"activity_id": activity_id, "status": "active"}

    def end_activity(self, activity_id: int) -> Dict:
        """结束活动"""
        activity = self._get_activity(activity_id)
        if not activity:
            return {"error": "activity_not_found"}

        now = datetime.now()

        # 1. 更新状态
        self.db.update("flash_sale_activities", {
            "status": ActivityStatus.ENDED.value,
            "ended_at": now,
        }, {"activity_id": activity_id})

        # 2. 清理 Redis 缓存
        self.redis.delete(f"seckill:config:{activity_id}")
        self.redis.delete(f"stock:{activity_id}:{activity['sku_id']}")

        # 3. 最终对账
        reconciliation = self._reconcile_stock(activity_id)

        # 4. 发送活动结束通知 + 结果推送
        self._push_results_to_users(activity_id)

        return {
            "activity_id": activity_id,
            "status": "ended",
            "reconciliation": reconciliation,
        }

    def _preallocate_stock(self, activity_id: int, total_stock: int,
                            sku_id: int):
        """库存预分配：将库存加载到 Redis"""
        stock_key = f"stock:{activity_id}:{sku_id}"

        # 使用 Lua 脚本保证原子性：仅在 key 不存在时设置
        lua_script = """
        if redis.call('EXISTS', KEYS[1]) == 0 then
            redis.call('SET', KEYS[1], ARGV[1])
            redis.call('EXPIRE', KEYS[1], 86400)
            return 1
        end
        return 0
        """
        result = self.redis.eval(lua_script, 1, stock_key, total_stock)

        if result == 0:
            # Redis 已有库存数据，检查一致性
            current = int(self.redis.get(stock_key))
            if current != total_stock:
                # 以数据库为准，更新 Redis
                self.redis.set(stock_key, total_stock)

        # 记录库存快照（用于对账）
        self.redis.set(f"stock:snapshot:{activity_id}", total_stock, ex=86400)

    def _reconcile_stock(self, activity_id: int) -> Dict:
        """库存最终对账"""
        activity = self._get_activity(activity_id)

        # Redis 中的剩余库存
        redis_remaining = int(self.redis.get(
            f"stock:{activity_id}:{activity['sku_id']}") or 0)

        # 数据库中的已售数量
        sold_count = self.db.query(
            "SELECT COUNT(*) as cnt FROM orders "
            "WHERE activity_id = %s AND status NOT IN ('cancelled', 'cancelled_timeout')",
            activity_id)["cnt"]

        # 初始库存
        total_stock = activity["total_stock"]

        # 一致性检查
        db_remaining = total_stock - sold_count
        is_consistent = (redis_remaining == db_remaining)

        if not is_consistent:
            # 记录不一致
            self.db.insert("stock_reconciliation_log", {
                "activity_id": activity_id,
                "total_stock": total_stock,
                "redis_remaining": redis_remaining,
                "db_remaining": db_remaining,
                "sold_count": sold_count,
                "discrepancy": redis_remaining - db_remaining,
                "checked_at": datetime.now(),
            })

        return {
            "is_consistent": is_consistent,
            "total_stock": total_stock,
            "sold_count": sold_count,
            "redis_remaining": redis_remaining,
            "db_remaining": db_remaining,
        }

    def _push_results_to_users(self, activity_id: int):
        """推送秒杀结果给用户"""
        # 查询所有成功下单的用户
        successful_orders = self.db.query(
            "SELECT DISTINCT user_id FROM orders "
            "WHERE activity_id = %s AND status IN ('pending_payment', 'paid', 'shipped')",
            activity_id)

        for order in successful_orders:
            self.notify.send(order["user_id"], {
                "type": "seckill_result",
                "activity_id": activity_id,
                "result": "success",
            })

    def _get_activity(self, activity_id: int) -> Optional[Dict]:
        """获取活动信息"""
        cache_key = f"activity:{activity_id}"
        cached = self.redis.get(cache_key)
        if cached:
            return json.loads(cached)

        row = self.db.query(
            "SELECT * FROM flash_sale_activities WHERE activity_id = %s",
            activity_id)
        if row:
            self.redis.setex(cache_key, 60, json.dumps(row, default=str))
        return row

    def _generate_preview_token(self, activity_id: int) -> str:
        """生成预览 token（防泄露）"""
        import hashlib
        raw = f"preview:{activity_id}:{time.time()}"
        return hashlib.sha256(raw.encode()).hexdigest()[:32]
```

## 反机器人完整系统

```python
import hashlib
import json
import time
import statistics
from typing import Dict, List, Optional, Tuple
from dataclasses import dataclass, field
from enum import Enum


class RiskAction(Enum):
    ALLOW = "allow"           # 放行
    CAPTCHA = "captcha"       # 需要验证码
    CHALLENGE = "challenge"   # 需要高级验证
    BLOCK = "block"           # 拦截


@dataclass
class DeviceFingerprint:
    """设备指纹"""
    fingerprint_id: str
    user_agent: str
    screen_resolution: str
    timezone: str
    language: str
    platform: str             # Win/Mac/Linux/iOS/Android
    plugins_hash: str         # 浏览器插件哈希
    canvas_hash: str          # Canvas 指纹哈希
    webgl_hash: str           # WebGL 渲染器哈希
    audio_hash: str           # AudioContext 指纹
    font_list_hash: str       # 系统字体列表哈希
    created_at: float = 0


@dataclass
class BehaviorProfile:
    """行为画像"""
    fingerprint_id: str
    click_intervals: List[float] = field(default_factory=list)    # 点击间隔
    request_intervals: List[float] = field(default_factory=list)  # 请求间隔
    scroll_events: int = 0         # 滚动事件数
    mouse_move_distance: float = 0 # 鼠标移动距离
    keypress_count: int = 0        # 按键次数
    page_stay_time: float = 0      # 页面停留时间
    form_focus_events: int = 0     # 表单聚焦事件


class AntiBotSystem:
    """反机器人系统：设备指纹 + 行为分析 + CAPTCHA 挑战 + IP 信誉"""

    def __init__(self, redis_client, db_client, captcha_service, ip_reputation):
        self.redis = redis_client
        self.db = db_client
        self.captcha = captcha_service
        self.ip_rep = ip_reputation

        # 风险阈值配置
        self.threshold_captcha = 50   # >= 50 分 → 需要 CAPTCHA
        self.threshold_challenge = 70 # >= 70 分 → 需要高级验证
        self.threshold_block = 90     # >= 90 分 → 直接拦截

    def check_request(self, request) -> Dict:
        """检查请求风险等级"""
        score = 0
        reasons = []

        # 1. 设备指纹检查
        fp_score, fp_reasons = self._check_device_fingerprint(request)
        score += fp_score
        reasons.extend(fp_reasons)

        # 2. 请求间隔分析
        interval_score, interval_reasons = self._check_request_interval(request)
        score += interval_score
        reasons.extend(interval_reasons)

        # 3. IP 信誉检查
        ip_score, ip_reasons = self._check_ip_reputation(request)
        score += ip_score
        reasons.extend(ip_reasons)

        # 4. 行为模式分析
        behavior_score, behavior_reasons = self._check_behavior_pattern(request)
        score += behavior_score
        reasons.extend(behavior_reasons)

        # 综合判定
        score = min(100, score)
        if score >= self.threshold_block:
            action = RiskAction.BLOCK
        elif score >= self.threshold_challenge:
            action = RiskAction.CHALLENGE
        elif score >= self.threshold_captcha:
            action = RiskAction.CAPTCHA
        else:
            action = RiskAction.ALLOW

        # 记录风险日志
        self._log_risk(request, score, action, reasons)

        return {
            "action": action.value,
            "score": score,
            "reasons": reasons,
            "captcha_required": action in (RiskAction.CAPTCHA, RiskAction.CHALLENGE),
        }

    def _check_device_fingerprint(self, request) -> Tuple[int, List[str]]:
        """设备指纹检查"""
        score = 0
        reasons = []

        fp_id = request.headers.get("X-Device-Fingerprint")
        if not fp_id:
            score += 30
            reasons.append("no_fingerprint")
            return score, reasons

        # 检查指纹是否在黑名单
        if self.redis.sismember("fingerprint:blacklist", fp_id):
            score += 80
            reasons.append("fingerprint_blacklisted")
            return score, reasons

        # 检查指纹关联的用户数（同一设备多账号 = 可疑）
        associated_users = self.redis.scard(f"fp:users:{fp_id}")
        if associated_users > 5:
            score += 40
            reasons.append(f"too_many_users:{associated_users}")
        elif associated_users > 3:
            score += 15
            reasons.append(f"multiple_users:{associated_users}")

        # 检查指纹关联的秒杀活动数（同一设备抢多个活动）
        activity_count = self.redis.scard(f"fp:activities:{fp_id}")
        if activity_count > 10:
            score += 30
            reasons.append(f"too_many_activities:{activity_count}")

        # 指纹伪造检测（Canvas/WebGL 哈希一致性）
        fingerprint_data = self.redis.get(f"fingerprint:data:{fp_id}")
        if fingerprint_data:
            stored = json.loads(fingerprint_data)
            current_canvas = request.headers.get("X-Canvas-Hash", "")
            current_webgl = request.headers.get("X-WebGL-Hash", "")
            if stored.get("canvas_hash") != current_canvas:
                score += 20
                reasons.append("canvas_hash_mismatch")
            if stored.get("webgl_hash") != current_webgl:
                score += 20
                reasons.append("webgl_hash_mismatch")

        return score, reasons

    def _check_request_interval(self, request) -> Tuple[int, List[str]]:
        """请求间隔分析"""
        score = 0
        reasons = []

        fp_id = request.headers.get("X-Device-Fingerprint", request.client_ip)
        now = time.time()

        # 记录请求时间
        interval_key = f"req:intervals:{fp_id}"
        self.redis.lpush(interval_key, now)
        self.redis.ltrim(interval_key, 0, 19)  # 保留最近 20 次
        self.redis.expire(interval_key, 300)    # 5 分钟过期

        # 获取最近的请求间隔
        intervals_raw = self.redis.lrange(interval_key, 0, 19)
        if len(intervals_raw) >= 3:
            timestamps = [float(t) for t in intervals_raw]
            intervals = [timestamps[i] - timestamps[i+1] for i in range(len(timestamps)-1)]

            # 检查间隔过短
            if intervals and min(intervals) < 0.05:  # 50ms 内
                score += 50
                reasons.append(f"request_too_fast:{min(intervals)*1000:.0f}ms")

            # 检查间隔过于均匀（机器人特征）
            if len(intervals) >= 5:
                avg = statistics.mean(intervals)
                if avg > 0:
                    std = statistics.stdev(intervals)
                    cv = std / avg  # 变异系数
                    if cv < 0.05:  # 变异系数 < 5% → 机器行为
                        score += 40
                        reasons.append(f"uniform_intervals:cv={cv:.3f}")

        return score, reasons

    def _check_ip_reputation(self, request) -> Tuple[int, List[str]]:
        """IP 信誉检查"""
        score = 0
        reasons = []
        ip = request.client_ip

        # 检查 IP 黑名单
        if self.redis.sismember("ip:blacklist", ip):
            score += 60
            reasons.append("ip_blacklisted")
            return score, reasons

        # 检查 IP 信誉分数
        rep_key = f"ip:reputation:{ip}"
        rep_score = self.redis.get(rep_key)
        if rep_score:
            rep = int(rep_score)
            if rep < 30:
                score += 40
                reasons.append(f"ip_bad_reputation:{rep}")
            elif rep < 60:
                score += 15
                reasons.append(f"ip_suspicious_reputation:{rep}")

        # 检查 IP 关联请求数（同一 IP 短时大量请求）
        ip_req_key = f"ip:req_count:{ip}"
        req_count = self.redis.incr(ip_req_key)
        if req_count == 1:
            self.redis.expire(ip_req_key, 60)
        if req_count > 100:
            score += 30
            reasons.append(f"ip_high_frequency:{req_count}/min")
        elif req_count > 50:
            score += 15
            reasons.append(f"ip_elevated_frequency:{req_count}/min")

        # 检查是否为已知代理/VPN IP
        if self.ip_rep.is_proxy(ip):
            score += 20
            reasons.append("proxy_ip")

        # 检查是否为数据中心 IP
        if self.ip_rep.is_datacenter(ip):
            score += 30
            reasons.append("datacenter_ip")

        return score, reasons

    def _check_behavior_pattern(self, request) -> Tuple[int, List[str]]:
        """行为模式分析"""
        score = 0
        reasons = []

        fp_id = request.headers.get("X-Device-Fingerprint", "")
        behavior_key = f"behavior:{fp_id}"

        # 获取行为画像
        behavior_data = self.redis.get(behavior_key)
        if not behavior_data:
            return score, reasons

        profile = json.loads(behavior_data)

        # 点击模式检测
        clicks = profile.get("click_intervals", [])
        if len(clicks) >= 5:
            avg_interval = statistics.mean(clicks)
            if avg_interval < 0.1:  # 平均点击间隔 < 100ms → 机器
                score += 30
                reasons.append("click_too_fast")
            if len(clicks) >= 5:
                std = statistics.stdev(clicks)
                if std / max(avg_interval, 0.001) < 0.05:
                    score += 25
                    reasons.append("click_pattern_too_uniform")

        # 缺少人类行为特征
        if profile.get("page_stay_time", 0) < 1.0:  # 停留不到 1 秒
            score += 15
            reasons.append("too_short_stay")

        if profile.get("mouse_move_distance", 0) == 0:
            score += 10
            reasons.append("no_mouse_movement")

        if profile.get("scroll_events", 0) == 0 and profile.get("page_stay_time", 0) > 3:
            # 停留超过 3 秒但无滚动 → 脚本驻留
            score += 10
            reasons.append("no_scroll_with_long_stay")

        return score, reasons

    def verify_captcha(self, fp_id: str, captcha_token: str) -> Dict:
        """验证 CAPTCHA"""
        result = self.captcha.verify(captcha_token)

        if result["valid"]:
            # CAPTCHA 通过，降低该指纹的风险评分
            self.redis.setex(f"captcha:passed:{fp_id}", 3600, "1")
            # 如果之前被拦截，释放拦截
            self.redis.delete(f"blocked:{fp_id}")
            return {"verified": True}
        else:
            return {"verified": False, "reason": result.get("reason", "invalid")}

    def record_behavior(self, fp_id: str, behavior: Dict):
        """记录用户行为数据"""
        behavior_key = f"behavior:{fp_id}"
        existing = self.redis.get(behavior_key)
        profile = json.loads(existing) if existing else {
            "click_intervals": [],
            "request_intervals": [],
            "scroll_events": 0,
            "mouse_move_distance": 0,
            "keypress_count": 0,
            "page_stay_time": 0,
            "form_focus_events": 0,
        }

        # 更新行为数据
        if "click_interval" in behavior:
            profile["click_intervals"].append(behavior["click_interval"])
            profile["click_intervals"] = profile["click_intervals"][-20:]  # 保留最近 20 次

        if "scroll" in behavior:
            profile["scroll_events"] += 1

        if "mouse_distance" in behavior:
            profile["mouse_move_distance"] += behavior["mouse_distance"]

        if "keypress" in behavior:
            profile["keypress_count"] += 1

        if "stay_time" in behavior:
            profile["page_stay_time"] = behavior["stay_time"]

        self.redis.setex(behavior_key, 600, json.dumps(profile))

    def _log_risk(self, request, score: int, action: RiskAction,
                  reasons: List[str]):
        """记录风险日志"""
        log_key = f"risk:log:{time.strftime('%Y%m%d')}"
        log_entry = {
            "ip": request.client_ip,
            "fp": request.headers.get("X-Device-Fingerprint", ""),
            "score": score,
            "action": action.value,
            "reasons": reasons,
            "timestamp": time.time(),
        }
        self.redis.lpush(log_key, json.dumps(log_entry))
        self.redis.ltrim(log_key, 0, 99999)  # 保留最近 10 万条
```

## 异常场景补充

### 场景：幂等键碰撞

```
触发：两个不同用户生成了相同的幂等键 → 第二个用户无法下单
原因：hash(user_id + activity_id + sku_id) 发生 MD5 碰撞
概率：MD5 碰撞概率极低（2^64 分之一），但以下场景可能增加风险：
  1. 用户 ID 格式异常（如包含特殊字符导致哈希输入相同）
  2. 幂等键生成算法缺陷（如缺少时间窗口区分）
检测：
  1. 幂等键插入失败（IntegrityError）→ 检查是否属于同一用户
  2. 用户反馈"无法下单"但库存充足 → 排查幂等键冲突
处理：
  1. 幂等键冲突时检查 user_id：
     - 同一用户 → 返回已有订单（正常幂等）
     - 不同用户 → 碰撞，为第二个用户生成新幂等键（加随机盐）
  2. 新幂等键 = hash(original_key + random_salt)
  3. 记录碰撞事件用于监控
预防：幂等键包含足够唯一维度 + 碰撞自动重试 + 碰撞率监控告警
```

### 场景：反机器人误杀真实用户

```
触发：正常用户被反机器人系统判定为机器人 → 无法参与秒杀
常见场景：
  1. 老年用户操作缓慢但点击模式不自然 → 触发行为分析告警
  2. 企业网络多用户共享出口 IP → IP 高频告警
  3. 手机浏览器指纹不完整 → 无指纹告警
  4. 使用无障碍工具的用户 → 行为模式异常
检测：
  1. 用户投诉"无法下单" → 误杀排查
  2. CAPTCHA 通过但仍被拦截 → 阈值过严
  3. 活动结束后成功订单率 < 预期 → 可能误杀过多
  4. 申诉通道请求量激增 → 误杀率高
处理：
  1. 短期：降低风险阈值（block 90→95，captcha 50→60）
  2. 开放 CAPTCHA 申诉通道：被拦截用户可通过 CAPTCHA 证明人类身份
  3. IP 共享场景：对同一 IP 不同设备指纹 → 分别评分而非整体限流
  4. 误杀用户白名单：CAPTCHA 通过 3 次以上的指纹加入信任列表
  5. 长期：收集误杀案例 → 优化规则权重 → 模型重训练
预防：
  - 风险阈值动态调整（根据实时误杀率反馈）
  - 多层验证而非一刀切拦截
  - 误杀率监控面板（目标 < 0.1%）
  - 人工审核申诉 + 规则持续优化
```

## 秒杀实时数据分析看板

### 看板架构概述

秒杀活动的运营需要秒级甚至亚秒级的数据反馈，传统 T+1 报表无法满足需求。实时看板基于流计算引擎（Flink）构建，从订单、库存、支付、用户行为四个数据源实时汇聚指标，推送到前端 WebSocket 展示。

**核心指标体系：**

| 指标类别 | 指标名称 | 计算方式 | 刷新频率 |
|---------|---------|---------|---------|
| 流量 | 当前 QPS | 滑动窗口 1s 内请求总数 | 1s |
| 流量 | 页面 UV | Redis HyperLogLog 去重 | 5s |
| 交易 | 每秒订单数 | 1s 窗口内订单创建数 | 1s |
| 交易 | 库存剩余 | Redis 实时库存值 | 1s |
| 交易 | 支付成功率 | 支付成功数 / 支付请求数（1分钟窗口） | 5s |
| 转化 | 转化漏斗 | 各步骤去重用户数 | 5s |
| 渠道 | 渠道转化率 | 按渠道分组的下单/UV | 10s |
| 地域 | 订单地域分布 | 按省份聚合订单数 | 10s |
| 设备 | 设备类型分布 | 按设备类型分组统计 | 30s |

### 实时指标计算引擎

```python
import time
import json
from collections import defaultdict
from datetime import datetime, timedelta


class SeckillAnalyticsEngine:
    """秒杀实时数据分析引擎"""

    def __init__(self, redis_client, kafka_consumer, flink_gateway):
        self.redis = redis_client
        self.kafka = kafka_consumer
        self.flink = flink_gateway
        # 滑动窗口状态
        self.qps_window = []           # [(timestamp, request_count)]
        self.order_window = []         # [(timestamp, order_count)]
        self.payment_window = []       # [(timestamp, payment_event)]
        self.funnel_state = {          # 转化漏斗状态
            "page_view": set(),        # 页面浏览用户
            "click": set(),            # 点击抢购用户
            "order": set(),            # 下单用户
            "pay": set(),              # 支付用户
            "confirm": set(),          # 确认收货用户
        }
        self.channel_stats = defaultdict(lambda: {"uv": set(), "orders": 0, "paid": 0})
        self.geo_stats = defaultdict(int)       # 省份 → 订单数
        self.device_stats = defaultdict(int)    # 设备类型 → 订单数
        self.peak_qps_timeline = []             # 峰值 QPS 时间线（秒级）
        self.window_size = 60  # 窗口大小（秒）

    def on_request(self, event):
        """处理请求事件"""
        ts = event["timestamp"]
        user_id = event["user_id"]
        channel = event.get("channel", "app")  # app / web / mini_program

        # 更新 QPS 窗口
        self.qps_window.append((ts, 1))
        self._cleanup_window(self.qps_window, ts)

        # 更新漏斗：页面浏览
        self.funnel_state["page_view"].add(user_id)

        # 更新渠道 UV
        self.channel_stats[channel]["uv"].add(user_id)

    def on_click(self, event):
        """处理点击抢购事件"""
        user_id = event["user_id"]
        self.funnel_state["click"].add(user_id)

    def on_order_created(self, event):
        """处理订单创建事件"""
        ts = event["timestamp"]
        user_id = event["user_id"]
        channel = event.get("channel", "app")
        province = event.get("province", "unknown")
        device_type = event.get("device_type", "unknown")

        # 更新订单窗口
        self.order_window.append((ts, 1))
        self._cleanup_window(self.order_window, ts)

        # 更新漏斗
        self.funnel_state["order"].add(user_id)

        # 更新渠道统计
        self.channel_stats[channel]["orders"] += 1

        # 更新地域分布
        self.geo_stats[province] += 1

        # 更新设备分布
        self.device_stats[device_type] += 1

    def on_payment(self, event):
        """处理支付事件"""
        ts = event["timestamp"]
        user_id = event["user_id"]
        channel = event.get("channel", "app")
        success = event.get("success", False)

        # 更新支付窗口
        self.payment_window.append((ts, 1 if success else 0))
        self._cleanup_window(self.payment_window, ts)

        if success:
            self.funnel_state["pay"].add(user_id)
            self.channel_stats[channel]["paid"] += 1

    def on_confirm(self, event):
        """处理确认收货事件"""
        user_id = event["user_id"]
        self.funnel_state["confirm"].add(user_id)

    def get_current_metrics(self):
        """获取当前实时指标快照"""
        now = time.time()

        # 当前 QPS（1 秒滑动窗口）
        current_qps = sum(count for ts, count in self.qps_window if now - ts <= 1)

        # 每秒订单数
        orders_per_second = sum(count for ts, count in self.order_window if now - ts <= 1)

        # 库存剩余
        session_id = self._get_active_session()
        inventory_remaining = int(self.redis.get(f"stock:{session_id}") or 0)

        # 支付成功率（1 分钟窗口）
        recent_payments = [s for ts, s in self.payment_window if now - ts <= 60]
        payment_success_rate = (
            sum(recent_payments) / len(recent_payments) * 100
            if recent_payments else 0
        )

        # 记录峰值 QPS 时间线
        self.peak_qps_timeline.append({
            "timestamp": datetime.now().isoformat(),
            "qps": current_qps,
            "orders_ps": orders_per_second,
        })
        # 只保留最近 3600 秒的时间线
        if len(self.peak_qps_timeline) > 3600:
            self.peak_qps_timeline = self.peak_qps_timeline[-3600:]

        return {
            "current_qps": current_qps,
            "orders_per_second": orders_per_second,
            "inventory_remaining": inventory_remaining,
            "payment_success_rate": round(payment_success_rate, 2),
            "timestamp": datetime.now().isoformat(),
        }

    def get_conversion_funnel(self):
        """获取转化漏斗数据"""
        return {
            "page_view": len(self.funnel_state["page_view"]),
            "click": len(self.funnel_state["click"]),
            "order": len(self.funnel_state["order"]),
            "pay": len(self.funnel_state["pay"]),
            "confirm": len(self.funnel_state["confirm"]),
            "rates": {
                "pv_to_click": self._calc_rate("page_view", "click"),
                "click_to_order": self._calc_rate("click", "order"),
                "order_to_pay": self._calc_rate("order", "pay"),
                "pay_to_confirm": self._calc_rate("pay", "confirm"),
                "overall": self._calc_rate("page_view", "pay"),
            }
        }

    def get_channel_performance(self):
        """获取渠道转化表现"""
        result = {}
        for channel, stats in self.channel_stats.items():
            uv = len(stats["uv"])
            result[channel] = {
                "uv": uv,
                "orders": stats["orders"],
                "paid": stats["paid"],
                "order_conversion_rate": round(stats["orders"] / uv * 100, 2) if uv else 0,
                "pay_conversion_rate": round(stats["paid"] / uv * 100, 2) if uv else 0,
            }
        return result

    def get_geo_distribution(self):
        """获取订单地域分布（Top 10）"""
        sorted_geo = sorted(self.geo_stats.items(), key=lambda x: x[1], reverse=True)
        return [{"province": p, "orders": c} for p, c in sorted_geo[:10]]

    def get_device_breakdown(self):
        """获取设备类型分布"""
        total = sum(self.device_stats.values())
        return {
            device: {"count": count, "percentage": round(count / total * 100, 2)}
            for device, count in self.device_stats.items()
        }

    def get_peak_qps_timeline(self, last_n_seconds=300):
        """获取峰值 QPS 时间线（秒级粒度）"""
        return self.peak_qps_timeline[-last_n_seconds:]

    def _calc_rate(self, from_step, to_step):
        """计算漏斗转化率"""
        from_count = len(self.funnel_state[from_step])
        to_count = len(self.funnel_state[to_step])
        return round(to_count / from_count * 100, 2) if from_count else 0

    def _cleanup_window(self, window, current_ts, max_age=None):
        """清理过期窗口数据"""
        age = max_age or self.window_size
        cutoff = current_ts - age
        while window and window[0][0] < cutoff:
            window.pop(0)

    def _get_active_session(self):
        """获取当前活跃秒杀场次"""
        return self.redis.get("seckill:active_session") or "0"


class SeckillDashboardPusher:
    """秒杀看板数据推送器：将指标推送到前端 WebSocket"""

    def __init__(self, analytics_engine, ws_manager):
        self.engine = analytics_engine
        self.ws_manager = ws_manager
        self.push_interval = 1.0  # 推送间隔（秒）

    async def push_loop(self):
        """持续推送实时指标到前端"""
        while True:
            metrics = self.engine.get_current_metrics()
            funnel = self.engine.get_conversion_funnel()
            channel = self.engine.get_channel_performance()

            payload = {
                "type": "seckill_realtime",
                "metrics": metrics,
                "funnel": funnel,
                "channel": channel,
            }

            await self.ws_manager.broadcast(json.dumps(payload))
            time.sleep(self.push_interval)

    async def push_geo_and_device(self):
        """低频推送：地域和设备分布（每 10 秒）"""
        while True:
            geo = self.engine.get_geo_distribution()
            device = self.engine.get_device_breakdown()
            timeline = self.engine.get_peak_qps_timeline(last_n_seconds=60)

            payload = {
                "type": "seckill_geo_device",
                "geo_distribution": geo,
                "device_breakdown": device,
                "qps_timeline": timeline,
            }

            await self.ws_manager.broadcast(json.dumps(payload))
            time.sleep(10)
```

### 看板 SQL 查询模板（Flink SQL）

```sql
-- 秒杀实时 QPS 指标（1 秒滚动窗口）
CREATE VIEW seckill_qps_metrics AS
SELECT
    TUMBLE_START(event_time, INTERVAL '1' SECOND) AS window_start,
    COUNT(*) AS qps,
    COUNT(DISTINCT user_id) AS uv
FROM seckill_request_stream
GROUP BY TUMBLE(event_time, INTERVAL '1' SECOND);

-- 秒杀转化漏斗（5 秒刷新）
CREATE VIEW seckill_funnel AS
SELECT
    COUNT(DISTINCT CASE WHEN event_type = 'page_view' THEN user_id END) AS pv_users,
    COUNT(DISTINCT CASE WHEN event_type = 'click' THEN user_id END) AS click_users,
    COUNT(DISTINCT CASE WHEN event_type = 'order' THEN user_id END) AS order_users,
    COUNT(DISTINCT CASE WHEN event_type = 'pay' THEN user_id END) AS pay_users,
    COUNT(DISTINCT CASE WHEN event_type = 'confirm' THEN user_id END) AS confirm_users
FROM seckill_event_stream
WHERE session_id = CURRENT_SESSION_ID();

-- 渠道转化率（10 秒刷新）
CREATE VIEW seckill_channel_conversion AS
SELECT
    channel,
    COUNT(DISTINCT user_id) AS uv,
    COUNT(DISTINCT CASE WHEN event_type = 'order' THEN user_id END) AS order_users,
    COUNT(DISTINCT CASE WHEN event_type = 'pay' THEN user_id END) AS paid_users,
    ROUND(
        COUNT(DISTINCT CASE WHEN event_type = 'order' THEN user_id END)
        * 100.0 / NULLIF(COUNT(DISTINCT user_id), 0), 2
    ) AS order_rate,
    ROUND(
        COUNT(DISTINCT CASE WHEN event_type = 'pay' THEN user_id END)
        * 100.0 / NULLIF(COUNT(DISTINCT user_id), 0), 2
    ) AS pay_rate
FROM seckill_event_stream
GROUP BY channel;
```

## 订单取消与退款流水线

### 取消与退款流程概述

秒杀订单的生命周期中，取消和退款是不可避免的环节。秒杀场景的特殊性在于：库存极其有限，取消释放的库存需要立即可供其他用户抢购；支付方式可能涉及多渠道拆分支付（优惠券 + 余额 + 信用卡），退款需要按渠道分别处理。

**核心规则：**
- 秒杀订单取消窗口：5 分钟（比普通订单 15 分钟更短，防止占库存）
- 取消后库存立即恢复，并触发排队通知
- 拆分支付退款：按支付渠道原路退回，优惠券退回有效期
- 余额退款：即时到账；信用卡退款：1-3 工作日
- 取消原因追踪：用于运营分析和风控

### 订单取消服务

```python
import time
import uuid
from enum import Enum
from datetime import datetime, timedelta
from typing import Optional


class CancellationReason(Enum):
    USER_INITIATED = "user_initiated"         # 用户主动取消
    PAYMENT_TIMEOUT = "payment_timeout"       # 支付超时
    RISK_BLOCK = "risk_block"                 # 风控拦截
    SYSTEM_ERROR = "system_error"             # 系统异常
    FRAUD_DETECTED = "fraud_detected"         # 欺诈检测


class SeckillOrderCancellationService:
    """秒杀订单取消服务"""

    CANCEL_WINDOW_SECONDS = 300  # 5 分钟取消窗口
    FRAUD_PATTERN_WINDOW = 3600  # 欺诈模式检测窗口（1 小时）

    def __init__(self, db, redis, stock_service, payment_service, notification_service):
        self.db = db
        self.redis = redis
        self.stock_service = stock_service
        self.payment_service = payment_service
        self.notification_service = notification_service
        # 分布式锁：防止取消和支付并发
        self.lock_prefix = "lock:cancel:"

    def cancel_order(self, order_id: str, user_id: str,
                     reason: CancellationReason = CancellationReason.USER_INITIATED) -> dict:
        """
        取消秒杀订单
        返回: {"success": bool, "message": str, "refund_details": dict}
        """
        # 1. 获取分布式锁，防止与支付流程并发
        lock_key = f"{self.lock_prefix}{order_id}"
        lock_acquired = self.redis.set(lock_key, "1", nx=True, ex=30)
        if not lock_acquired:
            return {"success": False, "message": "订单正在处理中，请稍后重试"}

        try:
            # 2. 查询订单信息
            order = self.db.query_one(
                "SELECT * FROM seckill_orders WHERE order_id = %s AND user_id = %s",
                order_id, user_id
            )
            if not order:
                return {"success": False, "message": "订单不存在"}

            # 3. 检查订单状态
            if order["status"] == "cancelled":
                return {"success": False, "message": "订单已取消"}
            if order["status"] == "shipped":
                return {"success": False, "message": "订单已发货，无法取消"}
            if order["status"] == "confirmed":
                return {"success": False, "message": "订单已确认收货，请申请退款"}

            # 4. 检查取消窗口（仅对用户主动取消检查）
            if reason == CancellationReason.USER_INITIATED:
                elapsed = (datetime.now() - order["created_at"]).total_seconds()
                if elapsed > self.CANCEL_WINDOW_SECONDS:
                    return {"success": False, "message": "已超过取消时限（5分钟）"}

            # 5. 欺诈取消检测
            fraud_check = self._check_fraud_cancellation(user_id, order["session_id"])
            if fraud_check["is_fraud"]:
                # 记录欺诈行为，但仍然允许取消（不阻止用户，但记录风控）
                self._record_fraud_event(user_id, order_id, fraud_check["pattern"])

            # 6. 执行取消（数据库事务）
            refund_details = self._execute_cancellation(order, reason)

            # 7. 原子恢复库存
            stock_restored = self.stock_service.rollback_stock(
                session_id=order["session_id"],
                user_id=user_id,
                request_id=order["request_id"],
                reason=f"order_cancelled:{reason.value}"
            )

            if not stock_restored:
                # 库存恢复失败 → 告警，人工介入
                self._alert_stock_restore_failed(order_id, order["session_id"])

            # 8. 通知排队用户
            self.notification_service.notify_waitlist(order["session_id"])

            # 9. 记录取消原因
            self._record_cancellation_reason(order_id, user_id, reason)

            return {
                "success": True,
                "message": "订单已取消" + ("，退款将在1-3个工作日内到账" if order["status"] == "paid" else ""),
                "refund_details": refund_details,
                "stock_restored": stock_restored,
            }

        finally:
            self.redis.delete(lock_key)

    def _execute_cancellation(self, order, reason):
        """执行取消操作（数据库事务）"""
        refund_details = None

        with self.db.transaction() as tx:
            # 更新订单状态
            tx.execute(
                "UPDATE seckill_orders SET status = 'cancelled', "
                "cancelled_at = %s, cancel_reason = %s WHERE order_id = %s",
                datetime.now(), reason.value, order["order_id"]
            )

            # 如果已支付，发起退款
            if order["status"] == "paid":
                refund_details = self.payment_service.initiate_refund(
                    order_id=order["order_id"],
                    amount=order["pay_amount"],
                    refund_reason=f"seckill_cancel:{reason.value}"
                )
                # 记录退款单
                tx.execute(
                    "INSERT INTO seckill_refunds (refund_id, order_id, amount, "
                    "status, created_at) VALUES (%s, %s, %s, 'processing', %s)",
                    refund_details["refund_id"],
                    order["order_id"],
                    order["pay_amount"],
                    datetime.now()
                )

            # 更新秒杀参与记录
            tx.execute(
                "UPDATE seckill_participation SET status = 'cancelled' "
                "WHERE session_id = %s AND user_id = %s",
                order["session_id"], order["user_id"]
            )

        return refund_details

    def _check_fraud_cancellation(self, user_id, session_id):
        """
        欺诈取消检测：检测"取消后重新抢购"模式
        典型欺诈：用户抢到后取消，再抢另一个SKU，或利用取消窗口反复占用库存
        """
        window_start = datetime.now() - timedelta(seconds=self.FRAUD_PATTERN_WINDOW)

        # 查询用户最近的取消和下单记录
        recent_cancels = self.db.query(
            "SELECT order_id, session_id, sku_id, cancelled_at "
            "FROM seckill_orders WHERE user_id = %s AND status = 'cancelled' "
            "AND cancelled_at > %s ORDER BY cancelled_at DESC",
            user_id, window_start
        )

        recent_orders = self.db.query(
            "SELECT order_id, session_id, sku_id, created_at "
            "FROM seckill_orders WHERE user_id = %s AND status IN ('paid', 'pending') "
            "AND created_at > %s ORDER BY created_at DESC",
            user_id, window_start
        )

        # 模式 1：取消后立即重新下单（30秒内）
        cancel_rebuy = False
        for cancel in recent_cancels:
            for new_order in recent_orders:
                gap = (new_order["created_at"] - cancel["cancelled_at"]).total_seconds()
                if 0 < gap < 30 and cancel["sku_id"] != new_order["sku_id"]:
                    cancel_rebuy = True
                    break

        # 模式 2：频繁取消（1小时内取消 3 次以上）
        frequent_cancel = len(recent_cancels) >= 3

        # 模式 3：取消后抢同一场次不同商品（换SKU行为）
        same_session_diff_sku = False
        for cancel in recent_cancels:
            for new_order in recent_orders:
                if (cancel["session_id"] == new_order["session_id"]
                        and cancel["sku_id"] != new_order["sku_id"]):
                    same_session_diff_sku = True

        is_fraud = cancel_rebuy or frequent_cancel or same_session_diff_sku
        pattern = []
        if cancel_rebuy:
            pattern.append("cancel_rebuy")
        if frequent_cancel:
            pattern.append("frequent_cancel")
        if same_session_diff_sku:
            pattern.append("same_session_diff_sku")

        return {"is_fraud": is_fraud, "pattern": pattern}


class SeckillRefundService:
    """秒杀退款服务：处理拆分支付的多渠道退款"""

    # 退款时效 SLA
    REFUND_SLA = {
        "balance": {"description": "余额退款", "sla": "即时", "max_seconds": 5},
        "coupon": {"description": "优惠券退回", "sla": "即时", "max_seconds": 5},
        "credit_card": {"description": "信用卡退款", "sla": "1-3工作日", "max_seconds": 259200},
        "alipay": {"description": "支付宝退款", "sla": "即时-2小时", "max_seconds": 7200},
        "wechat_pay": {"description": "微信退款", "sla": "即时-2小时", "max_seconds": 7200},
    }

    def __init__(self, db, redis, payment_gateways):
        self.db = db
        self.redis = redis
        self.payment_gateways = payment_gateways  # {"credit_card": CCPaymentGateway, ...}

    def initiate_refund(self, order_id: str, amount: int, refund_reason: str) -> dict:
        """
        发起退款：解析拆分支付，按渠道分别退款
        amount: 退款金额（分）
        返回: {"refund_id": str, "channels": [...], "total_amount": int}
        """
        refund_id = f"RF_{uuid.uuid4().hex[:16]}"

        # 查询支付拆分明细
        payment_splits = self.db.query(
            "SELECT * FROM payment_splits WHERE order_id = %s "
            "AND status = 'success' ORDER BY split_index",
            order_id
        )

        if not payment_splits:
            raise ValueError(f"订单 {order_id} 无有效支付记录")

        # 验证退款总额 = 支付总额
        total_paid = sum(p["amount"] for p in payment_splits)
        if amount != total_paid:
            raise ValueError(
                f"退款金额 {amount} 与支付总额 {total_paid} 不匹配"
            )

        # 按渠道创建退款子单
        channel_refunds = []
        for split in payment_splits:
            channel = split["channel"]
            channel_refund_id = f"CRF_{uuid.uuid4().hex[:16]}"

            channel_refund = {
                "channel_refund_id": channel_refund_id,
                "refund_id": refund_id,
                "order_id": order_id,
                "channel": channel,
                "amount": split["amount"],
                "original_payment_id": split["payment_id"],
                "status": "pending",
                "sla": self.REFUND_SLA.get(channel, {}).get("sla", "未知"),
            }

            # 余额和优惠券 → 即时退款
            if channel in ("balance", "coupon"):
                result = self._instant_refund(channel_refund)
                channel_refund["status"] = result["status"]
                channel_refund["completed_at"] = datetime.now().isoformat()
            else:
                # 信用卡/第三方支付 → 异步退款
                self._async_refund(channel_refund)

            channel_refunds.append(channel_refund)

            # 记录退款子单
            self.db.execute(
                "INSERT INTO refund_channels (channel_refund_id, refund_id, "
                "order_id, channel, amount, original_payment_id, status, sla, created_at) "
                "VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)",
                channel_refund_id, refund_id, order_id, channel,
                split["amount"], split["payment_id"],
                channel_refund["status"], channel_refund["sla"],
                datetime.now()
            )

        # 优惠券退回有效期
        coupon_splits = [s for s in payment_splits if s["channel"] == "coupon"]
        for coupon in coupon_splits:
            self._restore_coupon(coupon["payment_id"], order_id)

        return {
            "refund_id": refund_id,
            "total_amount": amount,
            "channels": channel_refunds,
        }

    def _instant_refund(self, channel_refund: dict) -> dict:
        """即时退款：余额 / 优惠券"""
        channel = channel_refund["channel"]
        amount = channel_refund["amount"]
        user_id = self._get_user_id(channel_refund["order_id"])

        if channel == "balance":
            # 余额退款：原子操作
            with self.db.transaction() as tx:
                tx.execute(
                    "UPDATE user_balance SET balance = balance + %s "
                    "WHERE user_id = %s", amount, user_id
                )
                tx.execute(
                    "INSERT INTO balance_log (user_id, amount, type, "
                    "ref_id, created_at) VALUES (%s, %s, 'refund', %s, %s)",
                    user_id, amount, channel_refund["channel_refund_id"],
                    datetime.now()
                )
            return {"status": "success"}

        elif channel == "coupon":
            # 优惠券退回：恢复有效期
            coupon_id = self._get_coupon_id(channel_refund["original_payment_id"])
            new_expiry = datetime.now() + timedelta(days=7)  # 退回后有效期 7 天
            self.db.execute(
                "UPDATE user_coupons SET status = 'active', expiry = %s "
                "WHERE coupon_id = %s AND user_id = %s",
                new_expiry, coupon_id, user_id
            )
            return {"status": "success"}

        return {"status": "failed"}

    def _async_refund(self, channel_refund: dict):
        """异步退款：信用卡 / 支付宝 / 微信"""
        channel = channel_refund["channel"]
        gateway = self.payment_gateways.get(channel)
        if not gateway:
            raise ValueError(f"不支持的退款渠道: {channel}")

        # 提交退款请求到支付网关
        gateway.submit_refund(
            refund_id=channel_refund["channel_refund_id"],
            original_payment_id=channel_refund["original_payment_id"],
            amount=channel_refund["amount"],
            reason="seckill_order_cancel"
        )

    def check_refund_status(self, refund_id: str) -> dict:
        """查询退款状态"""
        channels = self.db.query(
            "SELECT * FROM refund_channels WHERE refund_id = %s", refund_id
        )

        all_completed = all(c["status"] == "success" for c in channels)
        any_failed = any(c["status"] == "failed" for c in channels)

        return {
            "refund_id": refund_id,
            "status": "completed" if all_completed else ("partial_failed" if any_failed else "processing"),
            "channels": [
                {
                    "channel": c["channel"],
                    "amount": c["amount"],
                    "status": c["status"],
                    "sla": c["sla"],
                    "completed_at": c.get("completed_at"),
                }
                for c in channels
            ],
        }

    def _restore_coupon(self, payment_id: str, order_id: str):
        """恢复优惠券有效期"""
        coupon_usage = self.db.query_one(
            "SELECT coupon_id, user_id FROM coupon_usage "
            "WHERE payment_id = %s AND order_id = %s",
            payment_id, order_id
        )
        if coupon_usage:
            self.db.execute(
                "UPDATE user_coupons SET status = 'active', "
                "expiry = DATE_ADD(NOW(), INTERVAL 7 DAY) "
                "WHERE coupon_id = %s AND user_id = %s",
                coupon_usage["coupon_id"], coupon_usage["user_id"]
            )

    def _get_user_id(self, order_id: str) -> str:
        """从订单获取用户 ID"""
        order = self.db.query_one(
            "SELECT user_id FROM seckill_orders WHERE order_id = %s", order_id
        )
        return order["user_id"]

    def _get_coupon_id(self, payment_id: str) -> str:
        """从支付记录获取优惠券 ID"""
        usage = self.db.query_one(
            "SELECT coupon_id FROM coupon_usage WHERE payment_id = %s", payment_id
        )
        return usage["coupon_id"]
```

### 取消原因追踪表

```sql
CREATE TABLE seckill_cancellation_logs (
    log_id            BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id          VARCHAR(64) NOT NULL COMMENT '订单ID',
    user_id           BIGINT NOT NULL COMMENT '用户ID',
    session_id        BIGINT NOT NULL COMMENT '秒杀场次ID',
    cancel_reason     VARCHAR(32) NOT NULL COMMENT '取消原因枚举',
    cancel_source     VARCHAR(16) NOT NULL COMMENT '取消来源: user/system/risk',
    time_to_cancel_ms INT NOT NULL COMMENT '从下单到取消的毫秒数',
    was_paid          TINYINT NOT NULL COMMENT '取消时是否已支付',
    refund_initiated  TINYINT NOT NULL COMMENT '是否发起退款',
    stock_restored    TINYINT NOT NULL COMMENT '库存是否成功恢复',
    fraud_flag        TINYINT DEFAULT 0 COMMENT '是否标记为欺诈取消',
    fraud_pattern     VARCHAR(128) COMMENT '欺诈模式: cancel_rebuy/frequent_cancel',
    created_at        DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_session_reason (session_id, cancel_reason),
    INDEX idx_user_time (user_id, created_at),
    INDEX idx_fraud (fraud_flag, created_at)
) ENGINE=InnoDB COMMENT='秒杀取消原因追踪';
```

## 异常场景补充

### 场景：看板数据延迟导致运营盲区

```
触发：秒杀活动开始后，实时看板指标延迟 > 30秒 → 运营无法实时决策
原因：
  1. Flink 消费 Kafka 积压（Kafka 消费者 lag > 100万条）
  2. Redis 指标查询超时（看板频繁轮询导致 Redis 压力大）
  3. WebSocket 推送堆积（前端连接数 > 10万，推送线程池耗尽）
  4. 数据采集链路中某个环节 GC 停顿（Flink TaskManager Full GC）
检测：
  1. 看板推送延迟 = 当前时间 - 指标时间戳 > 10秒 → 告警
  2. Kafka consumer lag 监控 → lag 持续增长
  3. Flink checkpoint 耗时 > 正常值 3 倍 → 反压
  4. 前端心跳检测 → 连接断开或数据不更新
影响：
  1. 运营无法判断是否需要扩容 → 系统可能在过载
  2. 无法及时发现支付成功率异常 → 漏处理支付故障
  3. 库存数据不准 → 误判售罄或误判有库存
处理：
  1. 降级方案：看板切换到 Redis 直接查询模式（跳过 Flink）
  2. 减少 Flink 窗口精度：1秒窗口 → 5秒窗口（减少计算量）
  3. WebSocket 推送降级：全量推送 → 仅推送关键指标
  4. 增加看板数据本地缓存：前端缓存最近一次数据，超时不显示"实时"
  5. 排查根因：检查 Flink 反压、Kafka 分区均衡、Redis 慢查询
预防：
  - 看板数据链路独立部署（不与秒杀核心链路共享资源）
  - Flink 消费者预留 2 倍消费能力
  - WebSocket 推送采用分层架构（10个推送节点，每节点1万连接）
  - 关键指标双通道：Flink + Redis 直接查询互为备份
  - 看板延迟监控（延迟 > 5秒 P1 告警，> 30秒 P0 告警）
```

### 场景：退款流水线故障需人工介入

```
触发：退款子单全部或部分失败 → 退款卡在 processing 状态
典型场景：
  1. 信用卡网关超时 → 退款请求丢失，银行侧可能已退也可能未退
  2. 余额退款数据库事务失败 → 金额未到账但退款单标记为 processing
  3. 优惠券退回时优惠券已被删除/过期 → 无法恢复
  4. 拆分支付部分退款成功、部分失败 → 退款不完整
检测：
  1. 退款状态轮询：processing 状态超过 SLA 时长 → 告警
     - 余额退款 > 10秒仍在 processing → 异常
     - 信用卡退款 > 3天仍在 processing → 异常
  2. 退款成功率监控：成功率 < 99% → 告警
  3. 对账差异：退款总额 ≠ 支付总额 → 差异告警
处理：
  1. 信用卡网关超时：
     - 查询银行侧退款状态（退款查询接口）
     - 银行确认已退 → 更新状态为 success
     - 银行确认未退 → 重新提交退款
     - 银行无法确认 → 标记为 manual_intervention，人工对账
  2. 余额退款事务失败：
     - 检查余额操作幂等性（退款子单号做幂等键）
     - 重试退款操作（最多3次）
     - 仍失败 → 人工处理（财务手动调账）
  3. 优惠券退回失败：
     - 替代方案：发放等值新优惠券 + 延长有效期
     - 记录差异到 compensation_pending 表
  4. 部分退款成功：
     - 继续重试失败渠道（最多3次，间隔递增）
     - 全部重试失败 → 创建人工工单
     - 工单包含：订单详情、已退金额、未退金额、失败渠道
预防：
  - 退款操作幂等设计（channel_refund_id 作为幂等键）
  - 退款状态机 + 超时自动重试（指数退避）
  - 退款对账系统：每日与银行/支付渠道对账
  - 退款 SLA 监控 + 人工工单自动创建
  - 灰度退款：大额退款先人工审核再自动执行
```

## 秒杀防刷与风控完整实现

```python
class FlashSaleRiskControlService:
    """秒杀风控：多层防刷 + 风险评分 + 挑战验证"""

    RISK_LEVELS = {
        "low": {"score_range": (0, 30), "action": "pass"},
        "medium": {"score_range": (30, 70), "action": "challenge"},
        "high": {"score_range": (70, 100), "action": "block"},
    }

    def evaluate_risk(self, user_id, activity_id, request):
        """评估风险"""
        score = 0

        # 1. IP 频率检测（同一 IP > 10 次/分钟 → 高风险）
        ip = request.get("ip")
        ip_count = self.redis.incr(f"risk:ip:{ip}:{now().strftime('%Y%m%d%H%M')}")
        if ip_count > 10:
            score += 40
        elif ip_count > 5:
            score += 15

        # 2. 设备指纹检测
        device_id = request.get("device_fingerprint")
        if device_id:
            device_count = self.redis.incr(f"risk:device:{device_id}:{now().strftime('%Y%m%d%H%M')}")
            if device_count > 5:
                score += 35
            elif device_count > 3:
                score += 10

            # 检测模拟器/多开
            if self._is_emulator(device_id):
                score += 30

        # 3. 行为分析（请求间隔过短 → 机器人）
        last_request = self.redis.get(f"risk:last_request:{user_id}")
        if last_request:
            interval_ms = (now() - datetime.fromisoformat(last_request.decode())).total_seconds() * 1000
            if interval_ms < 100:  # < 100ms → 机器人
                score += 50
            elif interval_ms < 500:
                score += 20

        self.redis.setex(f"risk:last_request:{user_id}", 60, now().isoformat())

        # 4. 账号年龄（新号 → 高风险）
        account_age_days = self._get_account_age(user_id)
        if account_age_days < 1:
            score += 25
        elif account_age_days < 7:
            score += 10

        # 5. 历史违规
        violations = self.db.count("risk_violations", user_id=user_id)
        score += min(30, violations * 10)

        # 6. 下单频率（同一用户同一活动）
        user_activity_count = self.redis.incr(f"risk:user_activity:{user_id}:{activity_id}")
        if user_activity_count > 3:
            score += 30

        # 确定风险等级
        score = min(100, score)
        if score < 30:
            level = "low"
        elif score < 70:
            level = "medium"
        else:
            level = "high"

        action = self.RISK_LEVELS[level]["action"]

        # 记录风控日志
        self.db.insert("risk_evaluation_log", {
            "user_id": user_id, "activity_id": activity_id,
            "risk_score": score, "risk_level": level,
            "action": action, "ip": ip,
            "device_id": device_id, "evaluated_at": now()
        })

        return {"risk_score": score, "risk_level": level, "action": action}

    def handle_risk_action(self, user_id, risk_result, request):
        """执行风控动作"""
        action = risk_result["action"]

        if action == "block":
            # 直接拦截
            self.redis.setex(f"risk:blocked:{user_id}", 3600, "1")
            return {"status": "blocked", "message": "请求异常，请稍后再试"}

        elif action == "challenge":
            # 挑战验证（滑块/短信验证码）
            challenge_type = self._select_challenge_type(request)
            challenge_id = str(uuid4())

            if challenge_type == "slider":
                # 生成滑块验证
                puzzle = self._generate_slider_puzzle()
                self.redis.setex(f"challenge:{challenge_id}", 120,
                    json.dumps({"user_id": user_id, "type": "slider",
                               "answer": puzzle["answer"]}))

            elif challenge_type == "sms":
                # 发送短信验证码
                code = str(random.randint(100000, 999999))
                user = self.db.get_user(user_id)
                self.sms.send(user["phone"], f"验证码: {code}")
                self.redis.setex(f"challenge:{challenge_id}", 300,
                    json.dumps({"user_id": user_id, "type": "sms",
                               "code": code}))

            return {"status": "challenge_required",
                    "challenge_id": challenge_id,
                    "challenge_type": challenge_type}

        return {"status": "pass"}

    def verify_challenge(self, challenge_id, user_response):
        """验证挑战"""
        challenge_data = self.redis.get(f"challenge:{challenge_id}")
        if not challenge_data:
            return {"status": "expired"}

        challenge = json.loads(challenge_data)

        if challenge["type"] == "slider":
            # 验证滑块位置（容忍 5px 误差）
            if abs(float(user_response) - challenge["answer"]) < 5:
                self.redis.delete(f"challenge:{challenge_id}")
                return {"status": "verified"}
        elif challenge["type"] == "sms":
            if user_response == challenge["code"]:
                self.redis.delete(f"challenge:{challenge_id}")
                return {"status": "verified"}

        # 验证失败 → 记录
        self.redis.incr(f"challenge_fail:{challenge['user_id']}")
        return {"status": "failed"}

    def _is_emulator(self, device_id):
        """检测模拟器"""
        device = self.db.query_one(
            "SELECT * FROM device_fingerprints WHERE device_id = %s", device_id)
        if not device:
            return False
        indicators = ["root_access", "debug_mode", "vm_detection",
                      "sensor_anomaly", "clipboard_shared"]
        flags = sum(1 for i in indicators if device.get(i))
        return flags >= 2
```

## 异常场景补充

### 场景：风控误杀正常用户

```
触发：公司内网 50 人同时抢购 → 同一 IP → 被判定为高风险 → 全部拦截
检测：
  1. 同一活动拦截率 > 20% → 可能误杀
  2. 用户投诉无法参与 → 误杀
处理：
  1. IP 频率阈值区分场景（公司网络 vs 住宅网络）
  2. 被拦截用户可申诉 + 人工审核
  3. 降低 IP 维度权重，增加设备+行为维度
预防：多维度综合评分 + IP 阈值场景化 + 申诉通道
```

### 场景：风控规则被绕过

```
触发：刷单团队使用代理 IP 池 + 设备农场 + 模拟人工间隔 → 绕过风控
检测：
  1. 中奖用户中设备指纹重复率高 → 刷单
  2. 下单后立即转卖 → 非自用
处理：
  1. 增加行为生物识别（点击模式、滑动轨迹）
  2. 中奖后实名验证
  3. 限制转售（7 天内不可转赠）
预防：行为生物识别 + 实名验证 + 转售限制
```

## 秒杀防刷与风控完整实现

```python
import time
import hashlib
import logging
import threading
from dataclasses import dataclass, field
from typing import List, Optional, Dict, Tuple, Set
from enum import Enum
from collections import defaultdict, deque
from datetime import datetime, timedelta

logger = logging.getLogger(__name__)


class RiskLevel(Enum):
    """风险等级"""
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"


class BlockDecision(Enum):
    """风控决策"""
    PASS = "pass"
    CHALLENGE = "challenge"
    BLOCK = "block"


class RiskControlError(Exception):
    """风控服务异常"""
    pass


class UserBlockedError(Exception):
    """用户被风控拦截"""
    pass


@dataclass
class RiskScore:
    """风险评估结果"""
    user_id: str
    total_score: int  # 0-100, 越高风险越大
    level: RiskLevel
    decision: BlockDecision
    factors: Dict[str, int] = field(default_factory=dict)
    detail: str = ""
    timestamp: datetime = None

    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = datetime.now()


@dataclass
class IPFrequencyRecord:
    """IP频率记录"""
    ip: str
    request_times: deque = field(default_factory=lambda: deque(maxlen=1000))
    blocked_until: Optional[datetime] = None

    def recent_count(self, window_seconds: int = 60) -> int:
        cutoff = datetime.now() - timedelta(seconds=window_seconds)
        return sum(1 for t in self.request_times if t > cutoff)


@dataclass
class DeviceFingerprint:
    """设备指纹"""
    fingerprint_hash: str
    user_agents: Set[str] = field(default_factory=set)
    user_ids: Set[str] = field(default_factory=set)
    request_times: deque = field(default_factory=lambda: deque(maxlen=500))
    blocked_until: Optional[datetime] = None

    def recent_count(self, window_seconds: int = 60) -> int:
        cutoff = datetime.now() - timedelta(seconds=window_seconds)
        return sum(1 for t in self.request_times if t > cutoff)


@dataclass
class BehaviorProfile:
    """用户行为画像"""
    user_id: str
    page_view_times: deque = field(default_factory=lambda: deque(maxlen=200))
    click_intervals: deque = field(default_factory=lambda: deque(maxlen=100))
    order_intervals: deque = field(default_factory=lambda: deque(maxlen=50))
    mouse_movement_entropy: float = 1.0  # 鼠标移动信息熵, 机器人通常极低
    average_page_stay_seconds: float = 0.0

    def is_suspicious(self) -> Tuple[bool, str]:
        """检测行为是否可疑"""
        # 点击间隔过于均匀(机器人特征)
        if len(self.click_intervals) >= 5:
            intervals = list(self.click_intervals)[-20:]
            if intervals:
                avg = sum(intervals) / len(intervals)
                variance = sum((x - avg) ** 2 for x in intervals) / len(intervals)
                if variance < 0.01 and avg < 2.0:
                    return True, "click_interval_too_uniform"

        # 页面停留时间极短
        if self.average_page_stay_seconds > 0 and self.average_page_stay_seconds < 0.5:
            return True, "page_stay_too_short"

        # 鼠标移动信息熵极低(自动化脚本)
        if self.mouse_movement_entropy < 0.1:
            return True, "mouse_entropy_too_low"

        return False, ""


class FlashSaleRiskControlService:
    """
    秒杀防刷与风控服务

    实现多层防刷检测:
    - 第一层: IP频率限制
    - 第二层: 设备指纹识别
    - 第三层: 用户行为分析
    风险评分0-100, 高风险自动拦截, 中风险触发验证码挑战
    """

    # 风控阈值配置
    IP_RATE_LIMIT_PER_MINUTE = 30
    IP_RATE_LIMIT_PER_SECOND = 5
    IP_BLOCK_DURATION_MINUTES = 30
    DEVICE_MAX_USERS_PER_FINGERPRINT = 3
    DEVICE_RATE_LIMIT_PER_MINUTE = 20
    DEVICE_BLOCK_DURATION_MINUTES = 60
    USER_MAX_ORDERS_PER_SESSION = 1

    # 风险评分权重
    IP_SCORE_WEIGHT = 30
    DEVICE_SCORE_WEIGHT = 35
    BEHAVIOR_SCORE_WEIGHT = 35

    # 风险阈值
    BLOCK_THRESHOLD = 70       # >=70 直接拦截
    CHALLENGE_THRESHOLD = 40   # 40-69 触发验证码
    # <40 正常通过

    def __init__(self):
        self._ip_records: Dict[str, IPFrequencyRecord] = {}
        self._device_records: Dict[str, DeviceFingerprint] = {}
        self._behavior_profiles: Dict[str, BehaviorProfile] = {}
        self._blocked_users: Dict[str, datetime] = {}
        self._challenge_sessions: Dict[str, Dict] = {}
        self._lock = threading.Lock()
        self._stats = defaultdict(int)

    def evaluate(
        self,
        user_id: str,
        ip: str,
        device_fingerprint: str,
        user_agent: str = "",
        request_timestamp: Optional[datetime] = None,
    ) -> RiskScore:
        """
        执行风控评估主流程

        Args:
            user_id: 用户ID
            ip: 请求来源IP
            device_fingerprint: 设备指纹哈希
            user_agent: 浏览器UA
            request_timestamp: 请求时间

        Returns:
            RiskScore: 风险评估结果
        """
        now = request_timestamp or datetime.now()
        self._stats["total_evaluations"] += 1

        # 0. 检查用户是否已被封禁
        if self._is_user_blocked(user_id, now):
            self._stats["blocked_hits"] += 1
            return RiskScore(
                user_id=user_id,
                total_score=100,
                level=RiskLevel.CRITICAL,
                decision=BlockDecision.BLOCK,
                factors={"previously_blocked": 100},
                detail=f"用户{user_id}处于封禁期内",
            )

        # 1. IP频率检测
        ip_score, ip_factors = self._check_ip_frequency(ip, now)

        # 2. 设备指纹检测
        device_score, device_factors = self._check_device_fingerprint(
            device_fingerprint, user_id, user_agent, now
        )

        # 3. 行为分析
        behavior_score, behavior_factors = self._check_user_behavior(
            user_id, now
        )

        # 4. 计算综合风险分
        total_score = min(100, int(
            ip_score * (self.IP_SCORE_WEIGHT / 100)
            + device_score * (self.DEVICE_SCORE_WEIGHT / 100)
            + behavior_score * (self.BEHAVIOR_SCORE_WEIGHT / 100)
        ))

        # 5. 合并因子
        all_factors = {}
        all_factors.update(ip_factors)
        all_factors.update(device_factors)
        all_factors.update(behavior_factors)

        # 6. 决策
        if total_score >= self.BLOCK_THRESHOLD:
            level = RiskLevel.HIGH if total_score < 90 else RiskLevel.CRITICAL
            decision = BlockDecision.BLOCK
            self._block_user(user_id, now)
            self._stats["auto_blocked"] += 1
        elif total_score >= self.CHALLENGE_THRESHOLD:
            level = RiskLevel.MEDIUM
            decision = BlockDecision.CHALLENGE
            self._initiate_challenge(user_id, now)
            self._stats["challenged"] += 1
        else:
            level = RiskLevel.LOW
            decision = BlockDecision.PASS
            self._stats["passed"] += 1

        result = RiskScore(
            user_id=user_id,
            total_score=total_score,
            level=level,
            decision=decision,
            factors=all_factors,
            timestamp=now,
        )

        logger.info(
            f"风控评估: user={user_id}, ip={ip}, score={total_score}, "
            f"decision={decision.value}, factors={all_factors}"
        )
        return result

    def handle_challenge_response(
        self,
        user_id: str,
        challenge_token: str,
        challenge_answer: str,
    ) -> bool:
        """
        处理验证码挑战响应

        Returns:
            True表示验证通过, False表示验证失败
        """
        session = self._challenge_sessions.get(user_id)
        if not session:
            logger.warning(f"用户{user_id}无有效挑战会话")
            return False

        expected_token = session.get("token")
        if challenge_token != expected_token:
            logger.warning(f"用户{user_id}挑战token不匹配")
            return False

        # 检查是否超时(5分钟内有效)
        created = session.get("created_at")
        if created and datetime.now() - created > timedelta(minutes=5):
            del self._challenge_sessions[user_id]
            return False

        # 简化的验证码校验(实际应接入验证码服务)
        is_correct = self._verify_challenge_answer(
            session.get("challenge_type"), challenge_answer, session
        )

        if is_correct:
            del self._challenge_sessions[user_id]
            self._stats["challenge_passed"] += 1
            logger.info(f"用户{user_id}挑战验证通过")
        else:
            self._stats["challenge_failed"] += 1
            # 连续失败3次则封禁
            fail_count = session.get("fail_count", 0) + 1
            session["fail_count"] = fail_count
            if fail_count >= 3:
                self._block_user(user_id, datetime.now())
                del self._challenge_sessions[user_id]
                logger.warning(f"用户{user_id}连续验证失败{fail_count}次,已封禁")
            else:
                logger.info(f"用户{user_id}验证失败,累计{fail_count}次")

        return is_correct

    def record_behavior(
        self,
        user_id: str,
        event_type: str,
        event_data: Optional[Dict] = None,
    ):
        """记录用户行为事件,用于行为分析"""
        with self._lock:
            if user_id not in self._behavior_profiles:
                self._behavior_profiles[user_id] = BehaviorProfile(
                    user_id=user_id
                )
            profile = self._behavior_profiles[user_id]
            now = datetime.now()

            if event_type == "page_view":
                if profile.page_view_times:
                    interval = (
                        now - profile.page_view_times[-1]
                    ).total_seconds()
                    profile.click_intervals.append(interval)
                profile.page_view_times.append(now)

            elif event_type == "order":
                if profile.order_intervals:
                    last = profile.order_intervals[-1]
                    interval = (now - last).total_seconds()
                profile.order_intervals.append(now)

            elif event_type == "mouse_move":
                if event_data and "entropy" in event_data:
                    # 指数移动平均更新鼠标信息熵
                    alpha = 0.3
                    profile.mouse_movement_entropy = (
                        alpha * event_data["entropy"]
                        + (1 - alpha) * profile.mouse_movement_entropy
                    )

            elif event_type == "page_stay":
                if event_data and "duration" in event_data:
                    alpha = 0.2
                    profile.average_page_stay_seconds = (
                        alpha * event_data["duration"]
                        + (1 - alpha) * profile.average_page_stay_seconds
                    )

    def _check_ip_frequency(
        self, ip: str, now: datetime
    ) -> Tuple[int, Dict[str, int]]:
        """IP频率检测,返回(风险分0-100, 因子详情)"""
        factors = {}

        with self._lock:
            if ip not in self._ip_records:
                self._ip_records[ip] = IPFrequencyRecord(ip=ip)
            record = self._ip_records[ip]
            record.request_times.append(now)

        # 检查是否在封禁期
        if record.blocked_until and now < record.blocked_until:
            factors["ip_blocked"] = 100
            return 100, factors

        # 每秒频率
        per_second = record.recent_count(window_seconds=1)
        if per_second > self.IP_RATE_LIMIT_PER_SECOND:
            factors["ip_rate_per_second"] = 80
            self._block_ip(ip, now)
            return 80, factors

        # 每分钟频率
        per_minute = record.recent_count(window_seconds=60)
        if per_minute > self.IP_RATE_LIMIT_PER_MINUTE:
            ratio = per_minute / self.IP_RATE_LIMIT_PER_MINUTE
            score = min(80, int(ratio * 40))
            factors["ip_rate_per_minute"] = score
            if ratio > 2.0:
                self._block_ip(ip, now)
            return score, factors

        # 正常范围,给一个基础分
        if per_minute > self.IP_RATE_LIMIT_PER_MINUTE * 0.7:
            factors["ip_rate_elevated"] = 20
            return 20, factors

        return 0, factors

    def _check_device_fingerprint(
        self,
        fingerprint: str,
        user_id: str,
        user_agent: str,
        now: datetime,
    ) -> Tuple[int, Dict[str, int]]:
        """设备指纹检测"""
        factors = {}

        with self._lock:
            if fingerprint not in self._device_records:
                self._device_records[fingerprint] = DeviceFingerprint(
                    fingerprint_hash=fingerprint
                )
            record = self._device_records[fingerprint]
            record.user_ids.add(user_id)
            if user_agent:
                record.user_agents.add(user_agent)
            record.request_times.append(now)

        # 检查封禁期
        if record.blocked_until and now < record.blocked_until:
            factors["device_blocked"] = 100
            return 100, factors

        # 同一设备关联过多用户(刷单特征)
        user_count = len(record.user_ids)
        if user_count > self.DEVICE_MAX_USERS_PER_FINGERPRINT:
            score = min(90, 30 + (user_count - self.DEVICE_MAX_USERS_PER_FINGERPRINT) * 15)
            factors["device_multi_user"] = score
            if user_count > self.DEVICE_MAX_USERS_PER_FINGERPRINT * 3:
                self._block_device(fingerprint, now)
            return score, factors

        # 设备请求频率过高
        per_minute = record.recent_count(window_seconds=60)
        if per_minute > self.DEVICE_RATE_LIMIT_PER_MINUTE:
            score = min(80, int(per_minute / self.DEVICE_RATE_LIMIT_PER_MINUTE * 40))
            factors["device_rate_high"] = score
            return score, factors

        # UA切换频繁(模拟器特征)
        if len(record.user_agents) > 5:
            factors["device_ua_switch"] = 40
            return 40, factors

        return 0, factors

    def _check_user_behavior(
        self, user_id: str, now: datetime
    ) -> Tuple[int, Dict[str, int]]:
        """用户行为分析"""
        factors = {}

        profile = self._behavior_profiles.get(user_id)
        if not profile:
            return 0, factors

        is_suspicious, reason = profile.is_suspicious()
        if is_suspicious:
            score_map = {
                "click_interval_too_uniform": 70,
                "page_stay_too_short": 50,
                "mouse_entropy_too_low": 80,
            }
            score = score_map.get(reason, 30)
            factors[f"behavior_{reason}"] = score
            return score, factors

        # 检查订单频率
        if len(profile.order_intervals) >= 2:
            recent_orders = list(profile.order_intervals)[-5:]
            if len(recent_orders) >= 2:
                intervals = [
                    (recent_orders[i] - recent_orders[i - 1]).total_seconds()
                    for i in range(1, len(recent_orders))
                ]
                if intervals and min(intervals) < 1.0:
                    factors["behavior_order_too_fast"] = 60
                    return 60, factors

        return 0, factors

    def _is_user_blocked(self, user_id: str, now: datetime) -> bool:
        blocked_until = self._blocked_users.get(user_id)
        if blocked_until and now < blocked_until:
            return True
        if blocked_until:
            del self._blocked_users[user_id]
        return False

    def _block_user(self, user_id: str, now: datetime):
        self._blocked_users[user_id] = now + timedelta(
            minutes=self.IP_BLOCK_DURATION_MINUTES
        )
        logger.warning(f"用户{user_id}已被风控封禁")

    def _block_ip(self, ip: str, now: datetime):
        record = self._ip_records.get(ip)
        if record:
            record.blocked_until = now + timedelta(
                minutes=self.IP_BLOCK_DURATION_MINUTES
            )
        logger.warning(f"IP {ip}已被风控封禁")

    def _block_device(self, fingerprint: str, now: datetime):
        record = self._device_records.get(fingerprint)
        if record:
            record.blocked_until = now + timedelta(
                minutes=self.DEVICE_BLOCK_DURATION_MINUTES
            )
        logger.warning(f"设备{fingerprint[:8]}...已被风控封禁")

    def _initiate_challenge(self, user_id: str, now: datetime):
        token = hashlib.sha256(
            f"{user_id}:{now.timestamp()}:{id(self)}".encode()
        ).hexdigest()[:32]
        self._challenge_sessions[user_id] = {
            "token": token,
            "created_at": now,
            "challenge_type": "captcha",
            "fail_count": 0,
        }

    def _verify_challenge_answer(
        self, challenge_type: str, answer: str, session: Dict
    ) -> bool:
        """简化验证码校验,实际应接入专业验证码服务"""
        return len(answer) >= 4  # 占位实现

    def get_stats(self) -> Dict[str, int]:
        return dict(self._stats)

    def unblock_user(self, user_id: str) -> bool:
        """手动解封用户(运维操作)"""
        if user_id in self._blocked_users:
            del self._blocked_users[user_id]
            self._stats["manual_unblocked"] += 1
            logger.info(f"用户{user_id}已手动解封")
            return True
        return False
```

## 异常场景补充

### 场景：风控误杀正常用户

```
触发条件: 正常用户因共享网络(如公司/校园WiFi出口IP相同)导致IP频率超限被拦截;
          或同一设备登录多个家庭成员账号触发设备指纹多用户检测;
          或用户操作习惯特殊(快速浏览、低鼠标移动)被行为模型判定为可疑
检测手段: 1. 监控拦截率指标——正常业务拦截率应<1%,若超过2%则触发告警;
          2. 用户投诉渠道——当收到"无法参与秒杀"的客诉时,查询风控日志确认是否被误拦;
          3. 风控复核看板——每日生成被拦截用户清单,按历史购买记录、账号注册时长、
             实名认证状态等维度排序,高价值用户被拦时自动标记需人工复核;
          4. 对比同一用户在秒杀外的行为模式——若非秒杀时段行为完全正常,则大概率误杀
处理方式: 1. 立即通过unblock_user接口解封被误杀用户,并补偿秒杀参与资格(如下一期优先通道);
          2. 对共享IP场景引入IP白名单机制——企业用户可提交IP段申请白名单;
          3. 对家庭共享设备场景,增加"设备-用户"绑定确认流程——首次在新设备登录时
             通过短信验证码确认身份,确认后该用户不计入设备多用户计数;
          4. 行为模型校准——收集误杀案例作为负样本重新训练,调整阈值;
          5. 增加"二次确认"通道——被拦截用户可通过实名认证+人脸识别绕过风控
预防措施: 1. 引入多维度交叉验证——单一维度(如仅IP)超标不直接拦截,需至少两个维度同时异常;
          2. 对已知共享网络IP段(通过WHOIS和IP地理信息库识别)提高频率阈值;
          3. 灰度发布风控规则——新规则先对5%流量生效,观察误杀率后再全量;
          4. 建立"用户信任等级"体系——根据账号年龄、历史购买、实名认证等计算信任分,
             高信任用户降低风控敏感度;
          5. 风控决策可解释性——每个拦截必须输出具体触发因子,便于后续审计和调优
```

### 场景：风控规则被绕过

```
触发条件: 黑产使用代理IP池轮换绕过IP频率限制;使用设备指纹篡改工具(如Xposed框架)
          伪造不同设备指纹;使用AI模拟人类鼠标移动和点击行为绕过行为检测;
          通过分布式低频攻击(每个IP/设备保持在阈值以下但总量仍很大)
检测手段: 1. 监控秒杀成功率异常——若某商品秒杀成功率远高于历史同级别活动,
          且中奖用户集中在新注册/低活跃账号,则疑似被刷;
          2. 事后聚类分析——对所有成功下单用户按IP段/设备特征/注册时间聚类,
          发现异常聚集则回溯风控日志;
          3. 蜜罐检测——在秒杀页面植入隐藏的蜜罐字段(人类不可见但爬虫会填写),
          填写蜜罐字段的请求直接标记为机器人;
          4. 对比自然流量与秒杀流量的用户画像分布——若秒杀流量中机器特征占比异常升高则告警;
          5. 通过设备环境检测(WebView特征、传感器数据、电池状态一致性)识别模拟器
处理方式: 1. 紧急升级风控规则——发现绕过后立即调整阈值或增加检测维度;
          2. 事后回溯——对已确认黑产账号的订单执行取消并释放库存;
          3. 封禁IP段——对聚类分析发现的代理IP段执行大范围封禁;
          4. 引入前端JS动态挑战——每次秒杀前下发动态混淆的JS验证脚本,
          计算结果必须与服务端一致才能提交(对抗自动化工具);
          5. 对分布式低频攻击,引入全局速率令牌桶——不限单IP频率,而是限制
          整个秒杀活动的总请求吞吐量,超出则排队
预防措施: 1. 多层纵深防御——不依赖单一检测维度,即使IP和设备维度都被绕过,
          行为分析+蜜罐+JS挑战仍可拦截;
          2. 风控规则动态化——规则阈值不固定,基于实时流量自动调整,
          使黑产无法预测精确阈值;
          3. 设备环境完整性检测——采集设备传感器(陀螺仪/加速度计)数据、
          GPU渲染指纹、电池充电状态等,检测模拟器特征;
          4. 引入验证码服务升级——从简单图形验证码升级为行为验证(滑动拼图/点选)，
          并定期更换验证码类型;
          5. 与行业风控情报平台对接——共享已知黑产IP/设备指纹库,
          实现跨平台联合防刷
```

## 秒杀库存动态调整与防超卖完整实现

```python
class InventoryDynamicAdjustmentService:
    """库存动态调整：动态库存分配 → 超卖防护 → 库存回收 → 限流策略"""

    def allocate_dynamic_inventory(self, activity_id, total_stock, channels):
        """按渠道动态分配库存"""
        # 1. 获取各渠道历史销量权重
        channel_weights = {}
        for channel in channels:
            historical_sales = self.db.query_one(
                "SELECT COALESCE(SUM(quantity), 0) as total "
                "FROM flash_sale_orders "
                "WHERE channel = %s "
                "AND created_at > NOW() - INTERVAL 30 DAY",
                channel["id"])["total"]

            channel_weights[channel["id"]] = max(float(historical_sales), 1)

        # 2. 归一化权重
        total_weight = sum(channel_weights.values())
        allocations = {}
        remaining = total_stock

        for i, (channel_id, weight) in enumerate(
                sorted(channel_weights.items(), key=lambda x: -x[1])):
            if i == len(channels) - 1:
                # 最后一个渠道分配剩余
                allocations[channel_id] = remaining
            else:
                allocated = int(total_stock * weight / total_weight)
                allocations[channel_id] = allocated
                remaining -= allocated

        # 3. 设置各渠道的 Redis 库存
        for channel_id, stock in allocations.items():
            self.redis.set(f"flash_stock:{activity_id}:{channel_id}", stock)

        # 4. 保留应急库存（5%）
        emergency_stock = int(total_stock * 0.05)
        self.redis.set(f"flash_stock:{activity_id}:emergency", emergency_stock)

        # 5. 记录分配方案
        self.db.insert("inventory_allocations", {
            "allocation_id": str(uuid4()),
            "activity_id": activity_id,
            "total_stock": total_stock,
            "channel_allocations": json.dumps(allocations),
            "emergency_stock": emergency_stock,
            "created_at": now()
        })

        return {"activity_id": activity_id, "allocations": allocations,
                "emergency_stock": emergency_stock}

    def deduct_stock_with_oversell_protection(self, activity_id, channel_id,
                                               quantity, user_id):
        """防超卖扣库存（Lua 原子操作）"""
        # 1. Lua 脚本保证原子性
        lua_script = """
        local stock_key = KEYS[1]
        local user_key = KEYS[2]
        local quantity = tonumber(ARGV[1])
        local user_limit = tonumber(ARGV[2])

        -- 检查用户购买限制
        local user_bought = tonumber(redis.call('GET', user_key) or '0')
        if user_bought + quantity > user_limit then
            return {-1, 'user_limit_exceeded'}
        end

        -- 检查库存
        local stock = tonumber(redis.call('GET', stock_key) or '0')
        if stock < quantity then
            return {-2, 'insufficient_stock'}
        end

        -- 扣减库存
        redis.call('DECRBY', stock_key, quantity)
        redis.call('INCRBY', user_key, quantity)

        return {stock - quantity, 'success'}
        """

        stock_key = f"flash_stock:{activity_id}:{channel_id}"
        user_key = f"flash_user_bought:{activity_id}:{user_id}"

        # 获取用户限购数
        activity = self.db.get_flash_sale_activity(activity_id)
        user_limit = activity.get("per_user_limit", 1)

        result = self.redis.eval(lua_script, 2, stock_key, user_key,
                                 quantity, user_limit)

        remaining_stock = result[0]
        status = result[1]

        if status == "success":
            # 记录扣减日志
            self.db.insert("stock_deduction_log", {
                "log_id": str(uuid4()),
                "activity_id": activity_id,
                "channel_id": channel_id,
                "user_id": user_id,
                "quantity": quantity,
                "remaining_stock": remaining_stock,
                "created_at": now()
            })

            return {"status": "success", "remaining_stock": remaining_stock}

        elif status == "user_limit_exceeded":
            return {"status": "user_limit_exceeded",
                    "limit": user_limit}

        elif status == "insufficient_stock":
            # 尝试使用应急库存
            return self._try_emergency_stock(activity_id, quantity, user_id)

    def _try_emergency_stock(self, activity_id, quantity, user_id):
        """尝试使用应急库存"""
        emergency_key = f"flash_stock:{activity_id}:emergency"

        remaining = self.redis.decrby(emergency_key, quantity)

        if remaining >= 0:
            return {"status": "success_from_emergency",
                    "remaining_emergency": remaining}
        else:
            # 应急库存也不够 → 回滚
            self.redis.incrby(emergency_key, quantity)
            return {"status": "sold_out"}

    def reclaim_stock(self, activity_id, order_id, channel_id, quantity, reason):
        """回收库存（订单取消/支付超时）"""
        # 1. 回收渠道库存
        stock_key = f"flash_stock:{activity_id}:{channel_id}"
        new_stock = self.redis.incrby(stock_key, quantity)

        # 2. 清除用户购买记录
        # （仅在完全取消时清除，部分退款不清除限购记录）

        # 3. 记录回收
        self.db.insert("stock_reclaim_log", {
            "reclaim_id": str(uuid4()),
            "activity_id": activity_id,
            "order_id": order_id,
            "channel_id": channel_id,
            "quantity": quantity,
            "reason": reason,
            "new_stock_level": new_stock,
            "reclaimed_at": now()
        })

        # 4. 通知等待中的用户（如果有排队）
        self._notify_waitlisted_users(activity_id, quantity)

        return {"activity_id": activity_id, "reclaimed_quantity": quantity,
                "new_stock": new_stock}

    def _notify_waitlisted_users(self, activity_id, quantity):
        """通知等待中的用户库存有恢复"""
        # 从等待队列取出用户
        waitlist_key = f"flash_waitlist:{activity_id}"

        for _ in range(quantity):
            user_id = self.redis.lpop(waitlist_key)
            if not user_id:
                break

            uid = user_id.decode() if isinstance(user_id, bytes) else user_id
            self.notification.send(uid,
                f"秒杀商品库存有恢复，请尽快下单！")

    def get_realtime_stock_status(self, activity_id):
        """获取实时库存状态"""
        channels = self.db.query(
            "SELECT DISTINCT channel_id FROM stock_deduction_log "
            "WHERE activity_id = %s", activity_id)

        status = {}
        for ch in channels:
            key = f"flash_stock:{activity_id}:{ch['channel_id']}"
            stock = self.redis.get(key)
            status[ch["channel_id"]] = int(stock) if stock else 0

        emergency = self.redis.get(f"flash_stock:{activity_id}:emergency")
        status["emergency"] = int(emergency) if emergency else 0

        total_remaining = sum(v for v in status.values())

        return {
            "activity_id": activity_id,
            "total_remaining": total_remaining,
            "channel_breakdown": status,
            "is_sold_out": total_remaining == 0
        }

    def adjust_rate_limit_by_stock(self, activity_id):
        """根据剩余库存动态调整限流"""
        status = self.get_realtime_stock_status(activity_id)
        remaining = status["total_remaining"]

        activity = self.db.get_flash_sale_activity(activity_id)
        total_stock = activity["total_stock"]

        remaining_pct = remaining / max(total_stock, 1)

        # 库存越少 → 限流越严（减少无效请求）
        if remaining_pct > 0.5:
            rate_limit = 1000  # 1000 QPS
        elif remaining_pct > 0.2:
            rate_limit = 500
        elif remaining_pct > 0.05:
            rate_limit = 200
        elif remaining_pct > 0:
            rate_limit = 50
        else:
            rate_limit = 0  # 售罄

        # 更新限流配置
        self.redis.set(f"flash_rate_limit:{activity_id}", rate_limit)

        return {"activity_id": activity_id,
                "remaining_pct": round(remaining_pct * 100, 1),
                "rate_limit_qps": rate_limit}
```

## 异常场景补充

### 场景：库存分配与实际销量严重不匹配

```
触发：渠道 A 分配 80% 库存 → 实际渠道 A 只卖了 30% → 渠道 B 库存不足 → 销量损失
检测：
  1. 某渠道库存消耗率 < 30% 而另一渠道售罄 → 分配不均
  2. 活动开始 10 分钟后各渠道消耗速率差异 > 3 倍 → 需重新分配
处理：
  1. 活动开始后动态调整：每 5 分钟检查各渠道消耗速率
  2. 消耗慢的渠道库存自动转移到消耗快的渠道
  3. 转移需原子操作（先扣后加）
预防：动态再分配 + 定时检查 + 原子转移
```

### 场景：Redis 库存与数据库不一致

```
触发：Redis 扣减成功 → 数据库写入订单失败 → Redis 库存少了但订单不存在 → 库存泄漏
检测：
  1. 定时对账：Redis 库存 + 已生成订单数 ≠ 总库存 → 不一致
  2. 库存负数 → 超卖或泄漏
处理：
  1. 订单创建失败 → 自动回收 Redis 库存
  2. 定时对账修复（每分钟）
  3. 数据库事务包裹扣减+订单创建
预防：失败回滚 + 定时对账 + 事务保护
```

## 秒杀订单防刷与风控完整实现

```python
class FlashSaleAntiFraudService:
    """秒杀防刷：风控识别 → 行为分析 → 限流策略 → 黑名单管理"""

    RISK_FACTORS = {
        "new_account": {"weight": 20, "description": "新注册账户（< 7 天）"},
        "no_purchase_history": {"weight": 15, "description": "无购买历史"},
        "high_cancel_rate": {"weight": 25, "description": "历史取消率 > 50%"},
        "same_device_multi_account": {"weight": 30, "description": "同设备多账号"},
        "proxy_ip": {"weight": 20, "description": "代理 IP"},
        "bot_behavior": {"weight": 35, "description": "机器行为特征"},
        "velocity_anomaly": {"weight": 25, "description": "请求频率异常"},
    }

    def evaluate_risk(self, user_id, activity_id, request_context):
        """评估用户风险"""
        risk_score = 0
        risk_details = []

        # 1. 新账户检查
        user = self.db.get_user(user_id)
        if user and (now() - user["created_at"]).days < 7:
            risk_score += self.RISK_FACTORS["new_account"]["weight"]
            risk_details.append({
                "factor": "new_account",
                "detail": f"注册 {((now() - user['created_at']).days)} 天"
            })

        # 2. 购买历史检查
        purchase_count = self.db.count("orders", user_id=user_id, status="completed")
        if purchase_count == 0:
            risk_score += self.RISK_FACTORS["no_purchase_history"]["weight"]
            risk_details.append({"factor": "no_purchase_history"})

        # 3. 取消率检查
        total_orders = self.db.count("orders", user_id=user_id)
        cancelled_orders = self.db.count("orders", user_id=user_id, status="cancelled")
        if total_orders > 3:
            cancel_rate = cancelled_orders / total_orders
            if cancel_rate > 0.5:
                risk_score += self.RISK_FACTORS["high_cancel_rate"]["weight"]
                risk_details.append({
                    "factor": "high_cancel_rate",
                    "detail": f"取消率 {cancel_rate:.0%}"
                })

        # 4. 同设备多账号
        device_id = request_context.get("device_id")
        if device_id:
            accounts_on_device = self.redis.scard(f"device_accounts:{device_id}")
            if accounts_on_device > 2:
                risk_score += self.RISK_FACTORS["same_device_multi_account"]["weight"]
                risk_details.append({
                    "factor": "same_device_multi_account",
                    "detail": f"同设备 {accounts_on_device} 个账号"
                })

        # 5. 代理 IP 检测
        ip_address = request_context.get("ip_address")
        if ip_address and self._is_proxy_ip(ip_address):
            risk_score += self.RISK_FACTORS["proxy_ip"]["weight"]
            risk_details.append({"factor": "proxy_ip", "detail": ip_address})

        # 6. 机器行为检测
        if self._detect_bot_behavior(user_id, request_context):
            risk_score += self.RISK_FACTORS["bot_behavior"]["weight"]
            risk_details.append({"factor": "bot_behavior"})

        # 7. 请求频率异常
        request_count = self.redis.incr(f"request_count:{user_id}:{activity_id}")
        self.redis.expire(f"request_count:{user_id}:{activity_id}", 60)

        if request_count > 10:
            risk_score += self.RISK_FACTORS["velocity_anomaly"]["weight"]
            risk_details.append({
                "factor": "velocity_anomaly",
                "detail": f"1 分钟内 {request_count} 次请求"
            })

        # 8. 确定风险等级
        if risk_score >= 60:
            risk_level = "high"
            action = "block"
        elif risk_score >= 30:
            risk_level = "medium"
            action = "challenge"  # 需要验证码
        else:
            risk_level = "low"
            action = "allow"

        return {
            "user_id": user_id,
            "activity_id": activity_id,
            "risk_score": risk_score,
            "risk_level": risk_level,
            "action": action,
            "risk_details": risk_details
        }

    def handle_risky_request(self, user_id, activity_id, risk_evaluation):
        """处理风险请求"""
        action = risk_evaluation["action"]

        if action == "block":
            # 直接阻断
            self.db.insert("fraud_blocks", {
                "block_id": str(uuid4()),
                "user_id": user_id,
                "activity_id": activity_id,
                "risk_score": risk_evaluation["risk_score"],
                "risk_details": json.dumps(risk_evaluation["risk_details"]),
                "created_at": now()
            })

            # 加入活动黑名单
            self.redis.sadd(f"flash_blacklist:{activity_id}", user_id)

            return {"status": "blocked", "reason": "risk_too_high"}

        elif action == "challenge":
            # 需要验证码
            challenge_id = str(uuid4())
            captcha_type = self._select_captcha_type(risk_evaluation["risk_score"])

            self.db.insert("fraud_challenges", {
                "challenge_id": challenge_id,
                "user_id": user_id,
                "activity_id": activity_id,
                "captcha_type": captcha_type,
                "risk_score": risk_evaluation["risk_score"],
                "status": "pending",
                "created_at": now()
            })

            return {"status": "challenge_required",
                    "challenge_id": challenge_id,
                    "captcha_type": captcha_type}

        else:
            return {"status": "allowed"}

    def verify_challenge(self, challenge_id, user_response):
        """验证挑战"""
        challenge = self.db.get_fraud_challenge(challenge_id)

        if not challenge or challenge["status"] != "pending":
            return {"status": "invalid_challenge"}

        # 验证超时（5 分钟）
        if (now() - challenge["created_at"]).total_seconds() > 300:
            self.db.update("fraud_challenges",
                {"status": "expired"},
                {"challenge_id": challenge_id})
            return {"status": "expired"}

        # 验证答案
        is_correct = self._validate_captcha(challenge["captcha_type"], user_response)

        if is_correct:
            self.db.update("fraud_challenges",
                {"status": "passed"},
                {"challenge_id": challenge_id})

            # 临时降低风险（5 分钟内有效）
            self.redis.setex(
                f"challenge_passed:{challenge['user_id']}:{challenge['activity_id']}",
                300, "1")

            return {"status": "passed", "activity_id": challenge["activity_id"]}
        else:
            self.db.update("fraud_challenges",
                {"status": "failed"},
                {"challenge_id": challenge_id})

            return {"status": "failed"}

    def detect_scalper_pattern(self, activity_id):
        """检测黄牛模式"""
        # 1. 同一收货地址多账号下单
        orders = self.db.query(
            "SELECT shipping_address, COUNT(DISTINCT user_id) as user_count "
            "FROM flash_sale_orders "
            "WHERE activity_id = %s AND status != 'cancelled' "
            "GROUP BY shipping_address "
            "HAVING user_count > 2 "
            "ORDER BY user_count DESC",
            activity_id)

        scalper_addresses = []
        for order in orders:
            scalper_addresses.append({
                "address_hash": hashlib.sha256(
                    order["shipping_address"].encode()).hexdigest()[:8],
                "account_count": order["user_count"],
                "type": "same_address_multi_account"
            })

        # 2. 同 IP 多账号
        ip_groups = self.db.query(
            "SELECT ip_address, COUNT(DISTINCT user_id) as user_count "
            "FROM flash_sale_orders "
            "WHERE activity_id = %s AND status != 'cancelled' "
            "GROUP BY ip_address "
            "HAVING user_count > 3 "
            "ORDER BY user_count DESC",
            activity_id)

        scalper_ips = []
        for group in ip_groups:
            scalper_ips.append({
                "ip": group["ip_address"],
                "account_count": group["user_count"],
                "type": "same_ip_multi_account"
            })

        # 3. 下单时间过于集中（毫秒级差异）
        rapid_orders = self.db.query(
            "SELECT user_id, created_at, "
            "TIMESTAMPDIFF(MICROSECOND, created_at, "
            "  LAG(created_at) OVER (ORDER BY created_at)) as time_diff_us "
            "FROM flash_sale_orders "
            "WHERE activity_id = %s AND status != 'cancelled' "
            "ORDER BY created_at "
            "LIMIT 1000",
            activity_id)

        bot_users = set()
        for order in rapid_orders:
            if order.get("time_diff_us") and order["time_diff_us"] < 100000:  # < 100ms
                bot_users.add(order["user_id"])

        return {
            "activity_id": activity_id,
            "scalper_address_count": len(scalper_addresses),
            "scalper_addresses": scalper_addresses,
            "scalper_ip_count": len(scalper_ips),
            "scalper_ips": scalper_ips,
            "suspected_bot_users": len(bot_users),
            "bot_user_ids": list(bot_users)[:20]
        }

    def _is_proxy_ip(self, ip_address):
        """检测代理 IP"""
        # 检查已知代理 IP 库
        return self.redis.sismember("proxy_ip_list", ip_address)

    def _detect_bot_behavior(self, user_id, context):
        """检测机器行为"""
        # 1. 请求间隔过于均匀（人类不可能是完全均匀的）
        recent_intervals = self.redis.lrange(f"request_intervals:{user_id}", 0, 9)
        if len(recent_intervals) >= 5:
            intervals = [float(i) for i in recent_intervals]
            mean = statistics.mean(intervals)
            stdev = statistics.stdev(intervals) if len(intervals) > 1 else 0

            # 变异系数 < 0.1 → 太均匀 → 可能是机器人
            if mean > 0 and stdev / mean < 0.1:
                return True

        # 2. User-Agent 不常见
        ua = context.get("user_agent", "")
        known_bots = ["python", "curl", "wget", "httpclient", "okhttp"]
        if any(bot in ua.lower() for bot in known_bots):
            return True

        return False

    def _select_captcha_type(self, risk_score):
        """选择验证码类型"""
        if risk_score >= 50:
            return "slider_puzzle"  # 高风险 → 滑块拼图
        else:
            return "image_select"   # 中风险 → 图片选择

    def _validate_captcha(self, captcha_type, user_response):
        """验证验证码"""
        if captcha_type == "slider_puzzle":
            return user_response.get("accuracy", 0) > 0.8
        elif captcha_type == "image_select":
            return user_response.get("correct", False)
        return False
```

## 异常场景补充

### 场景：验证码被自动化破解

```
触发：黄牛使用 AI 识别验证码 → 滑块拼图和图片选择都能自动破解 → 防护失效
检测：
  1. 验证码通过率突然上升 → 可能被破解
  2. 同类设备全部通过验证 → 自动化
处理：
  1. 升级验证码难度（增加干扰元素）
  2. 切换验证码类型（行为验证码 + 短信验证）
  3. 高风险用户需人脸验证
预防：难度升级 + 多类型验证 + 人脸验证
```

### 场景：正常用户被误判为黄牛

```
触发：家庭共享 WiFi → 同 IP 5 个账号 → 被判定为黄牛 → 订单被取消
检测：
  1. 用户投诉订单被取消 → 误判
  2. 申诉通过率高 → 误判严重
处理：
  1. 同 IP 判定增加其他因素（设备、收货地址）
  2. 提供申诉通道（24 小时内处理）
  3. 申诉成功 → 恢复订单 + 补偿
预防：多因素判定 + 申诉通道 + 订单恢复
```

### 场景：黑名单数据被污染

```
触发：代理 IP 库误添加正常 IP 段 → 大量正常用户被拦截 → 订单量暴跌
检测：
  1. 拦截率突然上升 → 可能误封
  2. 投诉量暴增 → 黑名单问题
处理：
  1. 黑名单变更需审批
  2. 定期清理黑名单（30 天未命中则移除）
  3. 提供快速解封通道
预防：审批机制 + 定期清理 + 快速解封
```
