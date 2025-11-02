# 事务一致性：ChangeSource 与监听过滤

ChangeSource（src/lib/Store.ts）
- `'user'`：本地用户触发；`isMergingRemoteChanges=false`
- `'remote'`：远端同步合入；`isMergingRemoteChanges=true`（atomic 参数）

atomic 嵌套与回调
- 顶层：建立 `pendingAfterEvents`、切换 sideEffects 开关、可标记 remote 合入；
- 嵌套：不创建新的 pending 队列；可临时关闭 before*（仅从开→关，避免开关反复）
- 退出：`flushAtomicCallbacks` 执行 after* 与 operationComplete，并通过 HistoryAccumulator 推送 history

监听过滤（StoreListenerFilters）
- 结构：`{ source: 'user'|'remote'|'all', scope: RecordScope|'all' }`
- 作用：`store.listen(listener, filters)` 仅接收匹配源与作用域的变更，降低下游负担

建议
- 远端合并：总是放在 `atomic(..., runCallbacks=false, isMergingRemoteChanges=true)`，写完再统一处理业务后置；
- 本地 UI 只监听 `source:'user'`，避免 UI 因远端同步频繁重绘。

