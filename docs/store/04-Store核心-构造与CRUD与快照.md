# Store 核心：构造、CRUD、快照/迁移、原子化

构造要点（src/lib/Store.ts:~300 起）
- 字段：
  - `id: string`
  - `records: AtomMap<Id, Record>`
  - `history: Atom<number, RecordsDiff>`（historyLength=1000）
  - `query: StoreQueries`
  - `sideEffects: StoreSideEffects`
  - `schema: StoreSchema`
  - `historyAccumulator` + `reactor`（节流推送）

CRUD
- put(records[])：
  - before：`handleBeforeChange/handleBeforeCreate`（按是否存在）
  - validate：`schema.validateRecord(phase, knownGood?)`
  - freeze：`devFreeze(record)`
  - 写入：`records.set(id, record)`；收集 additions/updates，`addDiffForAfterEvent`
  - 历史：`updateHistory({ added, updated, removed:{} })`
- update(id, updater)：语法糖（内部转 put）
- remove(ids[])：
  - before：`handleBeforeDelete` 可阻止删除
  - AtomMap.deleteMany → 构造 removed 映射 → `updateHistory`
- get/unsafeGetWithoutCapture：记录级读；推荐热点路径无捕获

快照/序列化与迁移
- serialize(scope='document')：按 scope 过滤，返回 `Record<Id, Record>`
- getStoreSnapshot(scope)：{ store: serialize(scope), schema: schema.serialize() }
- migrateSnapshot(snapshot)：schema.migrateStoreSnapshot → 返回迁移后的 { store, schema }
- loadStoreSnapshot(snapshot)：自动迁移 → 清空现有数据 → 重新 put，一般关闭副作用避免噪声

原子化（atomic）与变更源（ChangeSource）
- `atomic(fn, runCallbacks=true, isMergingRemoteChanges=false)`：
  - 支持嵌套；内部维护 `_isInAtomicOp`、`pendingAfterEvents`、`sideEffects.isEnabled` 切换
  - 合并来源：`isMergingRemoteChanges` 打上 'remote' 源头；否则 'user'
  - 结束时 `flushAtomicCallbacks` 触发 after* 与 history 推送
- addHistoryInterceptor：可拦截合并前的 entry（带上源）

监听与节流
- `listen(listener, filters?)`（在文件中具体实现位置可搜索）通过 `history` reactor 在“下一帧”推送合并后变更；支持过滤源与 scope。

最佳实践
- 大规模写入/迁移/远端合并使用 `atomic(..., runCallbacks=false, isMergingRemoteChanges=true)`，避免副作用风暴；
- 只在必要处监听 history（或使用 query 的增量接口），减少全局重算。

