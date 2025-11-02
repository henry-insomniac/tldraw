# 历史缓冲与增量计算（HistoryBuffer）

位置：src/lib/HistoryBuffer.ts

能力
- 循环缓冲：保存最近 N 次 diff；若不足返回 `RESET_VALUE` 表示无法增量。
- API：`pushEntry(fromEpoch, toEpoch, diff)`、`getChangesSince(epoch)`、`clear()`。

与 Atom/Computed 的协作
- Atom.set 与 Computed 的求值在值变化时 push diff；当 `historyLength` 未配置则不创建缓冲，节省内存。
- Computed 支持 `withDiff(value, diff)` 将手工计算的 diff 放入缓冲（避免重复计算）。

使用建议
- 对于需要“时间差”或大对象局部更新的场景开启 `historyLength`；
- 不需要历史的信号避免开启，降低内存占用与写入开销。

