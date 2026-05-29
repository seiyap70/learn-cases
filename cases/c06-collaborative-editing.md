# C06: 在线协同编辑的冲突解决

## 业务场景

某在线文档平台（类似 Google Docs），支持多人同时编辑同一份文档。协同编辑是文档平台区别于传统文档软件的核心竞争力——如果多人无法同时编辑，用户就会退回到"一人编辑发邮件"的传统模式。

**核心需求：**
- 多个用户同时编辑同一份文档，所有人的修改都能正确合并
- 光标位置实时同步，用户能看到其他人的编辑光标和选中范围
- 离线编辑支持：用户断网后可以继续编辑，重新上线后自动合并
- 历史版本管理：可以回溯到任意历史版本

**已知数据：**
- 单文档同时编辑人数：通常 2-20 人，极端 100 人
- 文档大小：通常 < 100KB（约 3 万字），极端 10MB（长文档/含图片）
- 编辑延迟要求：本地操作 < 50ms 响应（即时渲染），远端同步 < 500ms
- 网络环境：用户可能使用不稳定的移动网络（丢包率 1-5%，延迟 100-500ms）
- 编辑频率：平均每 2 秒一次操作（输入一个字符、删除、格式修改等）

**为什么这是专家级难题？**

协同编辑是分布式系统中极少数"没有银弹"的问题之一。核心矛盾在于：
- **并发修改同一位置**——两个用户同时改同一个词，结果应该是什么？
- **因果顺序**——A 的修改基于文档版本 v1，B 的修改基于文档版本 v2，A 的修改在 v2 上可能不再适用
- **离线分歧**——两个用户各自离线编辑了 30 分钟，各自产生了 500 次修改，如何合并？

## 核心挑战

### 挑战 1：并发修改同一位置

**最简单的冲突场景：**

```
文档初始内容："Hello World"

用户A：将 "World" 改为 "Earth"  → "Hello Earth"
用户B：将 "World" 改为 "Mars"   → "Hello Mars"

两个操作同时提交，最终结果应该是什么？
- "Hello Earth"（A 的修改优先）？
- "Hello Mars"（B 的修改优先）？
- "Hello EarthMars"（两者都保留）？
```

没有"正确答案"，只有"收敛保证"——所有用户最终看到的文档内容必须完全一致。

### 挑战 2：操作变换的传递性问题

OT（Operational Transformation）的核心思想是对操作做"变换"，使得操作在不同上下文中产生相同效果。但变换必须满足数学性质（传递性、收敛性），否则会导致文档不一致。

Google Docs 团队花了数年时间才调试出正确的 OT 算法。

### 挑战 3：离线编辑后的合并

两个用户离线期间各自做了大量修改，重新上线后需要合并。OT 方案需要回溯所有历史操作重新变换，计算量大且容易出错。CRDT 方案天然支持离线合并，但需要理解 CRDT 的数学原理。

## 设计约束

- 客户端为 Web 浏览器（WebSocket 通信）
- 服务端需要持久化文档，重启后状态不能丢失
- 不允许使用文件锁（同一时刻只允许一人编辑）——业务要求必须支持并发编辑
- 合并后的文档不能丢失用户内容（除非两个用户确实改了同一个字）
- 文档格式支持：纯文本 + 基本格式（加粗/斜体/标题）

## 请先独立思考（限时 45 分钟）

1. 设计一个数据结构来表示"插入字符 'A' 到位置 5"这个操作。考虑：如果另一个操作同时删除了位置 3 的字符，这个操作应该如何调整？
2. CRDT 为什么能保证"最终收敛"？用你自己的话解释：两个用户各做 10 次编辑，文档最终一定会一致吗？
3. 光标位置如何在别人的编辑后保持正确？如果 A 在 B 的光标前插入 10 个字，B 的光标应该怎么移动？
4. 如果不用 OT 也不用 CRDT，有没有更简单的方案？它的问题是什么？

---

## 设计解析

### 方案选型：为什么选 CRDT（Yjs）而非 OT

**OT 的原理和问题：**

OT 的核心是：当两个操作并发执行时，对后到的操作做"变换"（Transformation），使其在前一个操作的结果上正确执行。

```
初始文档："ABC"
操作1：在位置1插入'X' → "AXBC"
操作2：在位置2插入'Y' → "ABYC"

两个操作并发，需要变换：
  操作1基于初始文档执行 → "AXBC"
  操作2需要变换：位置2 → 位置3（因为操作1在位置1插入了字符，后面的位置要+1）
  变换后的操作2：在位置3插入'Y' → "AXBYC"

结果：两个用户都看到 "AXBYC"
```

**OT 的致命问题：**

1. **变换函数极难正确实现** —— 对于富文本（含格式），变换函数需要处理几十种操作的组合（插入 vs 删除 vs 格式变更 vs 段落分割...），组合爆炸
2. **需要中心化服务器** —— OT 的变换通常在服务端执行（因为需要全局操作顺序），服务端成为瓶颈和单点
3. **离线支持困难** —— 离线期间的操作需要与在线期间的所有操作逐一变换，复杂度 O(N²)
4. **调试困难** —— Google Docs 的 OT 实现花了数年调试，文档不一致的 bug 极难重现

**CRDT 的原理：**

CRDT（Conflict-free Replicated Data Types）是一类数学上保证"最终收敛"的数据结构。核心思想：**每个操作携带足够的元数据，使得任何两个副本独立执行相同的操作序列后，状态一定相同。**

```
CRDT 不做"变换"，而是让操作本身携带足够信息来定位：

传统操作：在位置5插入'A'
  → 问题：位置5是相对于某个文档版本的，不同版本的位置5可能指向不同位置

CRDT 操作：在 Item(id=[A,3]) 之后插入 Item(id=[B,1], content='A')
  → Item 的 id 全局唯一，且有序
  → 无论在哪个副本上执行，都会找到 Item(id=[A,3]) 并在其后插入
  → 如果两个并发的插入都在 Item(id=[A,3]) 之后，按 id 排序决定先后
```

**CRDT 的收敛保证：**
- 每个 Item 有唯一 ID（clientId + 逻辑时钟）
- 相同 ID 的 Item 在所有副本中是同一个
- 并发冲突通过 ID 排序确定唯一顺序
- 无需服务端仲裁，纯数学保证

**选择 CRDT 的核心理由：**

| 维度 | OT | CRDT |
|------|-----|------|
| 离线支持 | 困难（需回溯历史变换） | 天然支持（CRDT 保证最终一致） |
| 服务端角色 | 必须参与冲突解决 | 仅做消息中转 |
| 实现复杂度 | 变换函数极难调对 | 数学保证正确性 |
| 性能 | 操作少时快 | 元数据开销大（需优化） |

### Yjs 的 CRDT 实现：深入原理

Yjs 将文档建模为**有序的双向链表**，每个字符是一个 Item 节点：

```typescript
class Item {
  id: [clientId: number, clock: number]  // 唯一标识
  left: Item | null    // 左侧邻居（插入位置）
  right: Item | null   // 右侧邻居
  content: any         // 字符内容或格式信息
  deleted: boolean     // 墓碑标记（删除不真正移除）
}
```

**插入操作的详细过程：**

```
文档初始状态（双向链表）：
  HEAD ↔ [H] ↔ [e] ↔ [l] ↔ [l] ↔ [o] ↔ TAIL

用户 A（clientId=1）在位置 2 插入 'X'：
  1. 找到位置 2 的 Item：[e]（id=[0,1]）
  2. 创建新 Item：id=[1,5], left=[e], right=[l], content='X'
  3. 链表变为：HEAD ↔ [H] ↔ [e] ↔ [X] ↔ [l] ↔ [l] ↔ [o] ↔ TAIL

用户 B（clientId=2）同时在位置 2 插入 'Y'：
  1. 找到位置 2 的 Item：[e]（id=[0,1]）——同样的左侧邻居
  2. 创建新 Item：id=[2,3], left=[e], right=[l], content='Y'
  3. 链表变为：HEAD ↔ [H] ↔ [e] ↔ [Y] ↔ [l] ↔ [l] ↔ [o] ↔ TAIL

现在两个副本不一致！合并时：
  - 副本A有 [X]（left=[e], right=[l]）
  - 副本B有 [Y]（left=[e], right=[l]）
  - 两者都在 [e] 和 [l] 之间
  - 按 id 排序：[1,5] < [2,3] → [X] 在 [Y] 前面
  - 最终链表：HEAD ↔ [H] ↔ [e] ↔ [X] ↔ [Y] ↔ [l] ↔ [l] ↔ [o] ↔ TAIL

所有副本执行相同的排序规则，最终结果一致。
```

**删除操作：墓碑标记**

```
删除 'X'：不真正移除 Item，而是标记 deleted=true

Item: id=[1,5], content='X', deleted=true

渲染时跳过 deleted=true 的 Item，但 Item 仍在链表中：
  - 保证其他 Item 的 left/right 引用不会断开
  - 光标可以定位到已删除的位置旁边
```

**为什么墓碑不会无限增长？**

Yjs 的 GC（Garbage Collection）机制：
- 当所有客户端都已确认某个 Item 被删除后（即没有客户端还持有该 Item 的未合并操作）
- 可以安全地移除该 Item，修复链表指针
- GC 是渐进式的，不需要全局暂停

### 服务端设计

```python
class CollaborationServer:
    """协同编辑服务端——仅做消息中转和持久化"""

    def __init__(self):
        self.rooms = {}  # docId → set of websocket connections
        self.document_store = DocumentStore()

    async def handle_connection(self, websocket, doc_id):
        # 1. 加入文档房间
        if doc_id not in self.rooms:
            self.rooms[doc_id] = set()
        self.rooms[doc_id].add(websocket)

        try:
            # 2. 同步：发送文档当前状态给新加入的客户端
            doc_state = await self.document_store.get_state_vector(doc_id)
            await websocket.send(json.dumps({
                "type": "sync-step-1",
                "docState": doc_state
            }))

            # 3. 消息循环
            async for message in websocket:
                data = json.loads(message)

                if data["type"] == "update":
                    # 客户端发来的编辑操作
                    update = data["update"]  # Uint8Array (Yjs 编码格式)

                    # 4. 持久化（异步，不阻塞转发）
                    await self.document_store.append_update(doc_id, update)

                    # 5. 广播给同一文档的其他客户端
                    for conn in self.rooms[doc_id]:
                        if conn != websocket:
                            await conn.send(json.dumps({
                                "type": "update",
                                "update": update
                            }))

                elif data["type"] == "sync-step-2":
                    # 客户端响应同步，发送差异操作
                    await self.document_store.append_update(doc_id, data["update"])
                    # 广播差异给其他客户端
                    for conn in self.rooms[doc_id]:
                        if conn != websocket:
                            await conn.send(json.dumps({
                                "type": "update",
                                "update": data["update"]
                            }))

        finally:
            self.rooms[doc_id].remove(websocket)
```

**服务端职责的极简性：**
- 不解析操作内容
- 不做冲突解决
- 只做消息转发 + 持久化
- 如果服务端宕机，客户端可以直连（P2P 模式，CRDT 仍保证收敛）

### 文档持久化

**写入策略：操作日志 + 定期快照**

```python
class DocumentStore:
    def __init__(self, redis, db):
        self.redis = redis
        self.db = db

    async def append_update(self, doc_id, update_bytes):
        """追加写入操作日志"""
        # Redis List 缓冲（快速写入）
        await self.redis.rpush(f"doc:updates:{doc_id}", update_bytes.hex())

        # 异步落库（每 100 条或每 5 秒批量写入）
        # ...

    async def get_state_vector(self, doc_id):
        """获取文档当前状态向量"""
        # 1. 读取最新快照
        snapshot = await self.db.query_one(
            "SELECT snapshot_data, snapshot_clock FROM document_snapshots "
            "WHERE doc_id = %s ORDER BY created_at DESC LIMIT 1", doc_id
        )

        if snapshot is None:
            return None  # 新文档

        # 2. 读取快照之后的增量操作
        updates = await self.redis.lrange(f"doc:updates:{doc_id}", 0, -1)

        # 3. 在 Yjs Doc 上重放
        doc = YDoc()
        doc.apply_update(base64.b64decode(snapshot.snapshot_data))
        for update_hex in updates:
            doc.apply_update(bytes.fromhex(update_hex))

        return doc.get_state_vector()

    async def create_snapshot(self, doc_id):
        """定期创建快照（减少重放量）"""
        doc = await self._rebuild_document(doc_id)

        await self.db.execute(
            "INSERT INTO document_snapshots (doc_id, snapshot_data, created_at) "
            "VALUES (%s, %s, NOW())",
            doc_id, doc.encode_state_as_update()
        )

        # 清理已快照的操作日志
        await self.redis.delete(f"doc:updates:{doc_id}")
```

**快照频率：** 每 1000 次操作或每 5 分钟，以先到者为准。活跃文档约 30 次/小时操作，5 分钟约 2-3 次，快照频率合理。

### 光标同步

**光标位置映射为 CRDT Item 引用：**

```typescript
class RemoteCursor {
  userId: string;
  anchor: Item | null;  // 选区起点（CRDT Item 引用）
  head: Item | null;    // 选区终点

  // 当其他用户的编辑改变了文档结构时
  // Item 引用自动调整（因为 Item 不会被删除，只是标记 deleted）
  // 所以光标始终指向正确的逻辑位置
}
```

**为什么不用位置索引（如"第 50 个字符"）？**

```
文档："Hello World"
用户B的光标在位置5（'W' 前面）

用户A在位置2插入 "XYZ"：
  文档变为："HeXYZllo World"

如果光标用位置索引：
  用户B的光标仍在位置5 → 现在指向 'l' 而非 'W' → 错误！

如果光标用 Item 引用：
  用户B的光标指向 Item('W') → 插入后仍指向 'W' → 正确！
```

**光标数据的传输：**

```json
{
  "type": "cursor",
  "docId": "doc_123",
  "userId": "user_456",
  "anchor": {"client": 1, "clock": 42},
  "head": {"client": 1, "clock": 42}
}
```

光标更新频率约 5 次/秒（用户移动鼠标时），数据量极小（约 50 字节），对系统负担可忽略。

### 离线编辑与重连合并

**离线流程：**

```
1. 用户A断网 → Yjs Doc 继续在本地记录操作
   （本地操作的 id 使用 clientId + 递增 clock，无需服务端分配）

2. 用户A离线期间做了 100 次编辑
   → 本地 Yjs Doc 有 100 个新 Item
   → 这些 Item 的 id = [A, clock_50] 到 [A, clock_149]

3. 用户A重新上线
   → 发送 sync-step-1：告知服务器自己的状态向量
   → 服务器返回差异操作（离线期间其他人的编辑）
   → 用户A应用差异操作（CRDT 自动合并）
   → 用户A同时将自己的离线操作发送给服务器
   → 服务器广播给其他客户端

4. 合并结果：
   → 所有副本执行相同操作序列 → CRDT 保证一致
   → 无需特殊处理
```

**这正是 CRDT 相对 OT 的最大优势——离线合并不是特殊场景，而是 CRDT 的默认行为。**

### 性能优化

**问题：大文档（10MB，约 300 万字符）的 CRDT 元数据开销**

- 每个字符的 Item 包含：id(8B) + left/right引用(16B) + content(1-4B) + deleted(1B) ≈ 30 字节
- 300 万字符 × 30 字节 = 90MB 内存（仅元数据）
- 100 人同时编辑 → 9GB 服务器内存（仅一个文档）

**优化策略：**

**1. 字符合并（Item Merging）**

```
连续输入 "Hello" 不是 5 个 Item，而是 1 个 Item：
  Item: id=[1,10], content="Hello", left=..., right=...

只有在 "Hello" 中间插入字符时才分裂：
  用户B在 "Hello" 的 'l' 前插入 'X' →
  Item1: id=[1,10], content="He", left=..., right=Item3
  Item2: id=[2,5], content="X", left=Item1, right=Item3
  Item3: id=[1,12], content="llo", left=Item2, right=...
```

**优化效果：** 正常编辑场景下，文档中的 Item 数量约等于字符数 × 0.1（因为大部分是连续输入），内存减少 90%。

**2. 增量同步**

Yjs 只传输差异操作（编码为 Uint8Array），不传输完整文档状态。

```
同步协议：
  客户端A的状态：包含 Item [1,1] 到 [1,100] 和 [2,1] 到 [2,50]
  客户端B的状态：包含 Item [1,1] 到 [1,80] 和 [2,1] 到 [2,60]

  差异：A 缺少 [2,51] 到 [2,60]，B 缺少 [1,81] 到 [1,100]
  交换差异：约 1KB（而非完整文档的 10MB）
```

**3. 懒加载**

大文档按段落分片：
- 初始只加载前 20 个段落
- 用户滚动时异步加载后续段落
- 每个段落是独立的 CRDT 子文档

**4. 定期 GC**

```python
class YjsGarbageCollector:
    def collect(self, doc):
        """清理已删除的墓碑 Item"""
        for item in doc.items:
            if item.deleted and self.all_clients_seen(item):
                # 所有客户端都已确认此 Item 被删除
                # 安全移除，修复链表指针
                if item.left:
                    item.left.right = item.right
                if item.right:
                    item.right.left = item.left
                doc.items.remove(item)
```

GC 条件：所有连接的客户端的状态向量都超过了被删除 Item 的 clock。需要跟踪每个客户端的状态向量。

## 常见陷阱（深度分析）

### 陷阱 1：OT 变换函数有 bug

**具体的 bug 类型：TP1 违反**

OT 需要满足 TP1 性质：`transform(transform(op1, op2), transform(op2, op1))` 的结果在两个副本上一致。

```
假设 transform 函数对"删除"操作的处理有 bug：

op1: 删除位置 3 的字符
op2: 删除位置 4 的字符

正确的变换：
  transform(op1, op2) = 删除位置3（op2不影响op1的位置）
  transform(op2, op1) = 删除位置3（op1删除了前面的字符，op2位置-1）

如果变换函数忘记处理"删除影响后续删除位置"的情况：
  transform(op2, op1) = 删除位置4（错误！应该减1）

结果：副本A删除位置3+位置4，副本B删除位置3+位置3 → 文档不一致
```

这类 bug 在 OT 中极难发现，因为只在不特定的操作组合下才出现。

### 陷阱 2：CRDT 不做 GC

墓碑无限增长的实际影响：

```
活跃文档，每天编辑 1000 次（约 500 次插入、300 次删除、200 次格式修改）
删除的 Item 变为墓碑，永不移除

一年后：
  活跃 Item：约 180,000 个
  墓碑 Item：约 110,000 个
  元数据大小：约 8.7MB（含墓碑）

不做 GC → 文档大小持续增长，加载和同步变慢
```

### 陷阱 3：光标用位置索引

位置索引在别人编辑后跳到错误位置的具体场景：

```
文档："ABCDEFGHIJ"
用户B光标在位置5（'F' 前面）

用户A删除位置1-3（"BCD"）→ 文档变为："AEFGHIJ"
位置5 现在指向 'I' 而非 'F' → 光标跳到错误位置

用 Item 引用：光标指向 Item('F') → 删除后 Item('F') 仍在链表中
  → Item('A').right = Item('F')（BCD 被标记 deleted）
  → 光标仍在 'F' 前面 → 正确
```

### 陷阱 4：大文档全量同步

10MB 文档全量同步的影响：

```
用户打开文档 → 下载完整 Yjs Doc → 10MB+ 元数据
首次加载延迟：10MB / 1Mbps = 80 秒

使用增量同步：
  客户端发送状态向量（约 100 字节）
  服务器计算差异 → 通常 < 100KB
  加载延迟：< 1 秒
```

## 延伸思考

- **建议模式（Suggest Mode）**：编辑以建议形式呈现，文档所有者可以接受/拒绝。实现方式：建议操作的 Item 标记为 `suggested=true`，渲染时用不同颜色显示。接受时去掉标记，拒绝时标记 `deleted=true`。
- **表格协同编辑**：表格的行列增删与内容编辑的冲突更复杂——删除一列时，其他用户正在编辑该列的单元格如何处理？Yjs 有专门的 Y.Array 和 Y.Map 类型处理结构化数据。
- **端到端加密**：如果文档内容在客户端加密后才上传，服务端只能转发密文。CRDT 的操作编码可以被加密，但服务端无法做快照合并。解决方案：客户端做快照，服务端只存储加密后的快照。