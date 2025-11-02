# 变更 Diff 与历史累积器

RecordsDiff（src/lib/RecordsDiff.ts）
- 结构：`{ added: Record<Id, R>, updated: Record<Id, [from,to]>, removed: Record<Id, R> }`
- 工具：合并/压缩（`squashRecordDiffs`）与 apply/逆向等（见文件内导出）

HistoryEntry 与 Accumulator（src/lib/Store.ts:~1600 以后）
- HistoryEntry：`{ changes: RecordsDiff, source: 'user'|'remote' }`
- HistoryAccumulator：
  - `add(entry)`：暂存并通知拦截器
  - `flush()`：相邻同源合并（`squashHistoryEntries`）→ 清空 → 返回合并结果
  - `clear()/hasChanges()`：工具方法

推送与节流
- Store 在写入后调用 `updateHistory(diff)` 将 entry 放入 accumulator；
- 有 reactor（`throttleToNextFrame`）在下一帧 `flush()` 并写入 `history` atom，触发监听回调；
- 合并策略避免“同一帧多次用户写入”导致的碎片化通知；remote 与 user 相邻分块分别合并。

最佳实践
- 批量写入应在 `atomic` 内完成，尽量减少帧间切换；
- 监听方可以通过 filters 仅取 `source:'user'` 或 `scope:'document'`，降低处理成本。

