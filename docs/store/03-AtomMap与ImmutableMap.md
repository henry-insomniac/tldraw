# AtomMap 与 ImmutableMap

AtomMap（src/lib/AtomMap.ts）
- Map 的响应式实现：每个值是一个独立 atom；键集合放在一个 `atoms` atom（ImmutableMap）。
- API 覆盖：`get/has/set/update/delete/deleteMany/clear/entries/keys/values/size/forEach/iterator`。
- 无捕获读：`__unsafe__getWithoutCapture/__unsafe__hasWithoutCapture`（热路径读不建立依赖）。
- 批量删除：`deleteMany(keys)` 利用 `withMutations` 一次性删除并将对应值原子置为 UNINITIALIZED，减少通知与分配。

ImmutableMap（src/lib/ImmutableMap.ts）
- 不可变 Map 的自实现，提供 `withMutations`、结构共享与高效迭代，服务于 AtomMap 内部。

设计权衡
- 记录级 atom 是细粒度响应的关键；当 map 规模很大时，避免“全表订阅”。
- 读写分离：热点读取用无捕获，响应式派生用 `get()`；批量变更用 `transact()` 包裹。

