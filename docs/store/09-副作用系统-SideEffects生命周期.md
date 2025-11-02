# 副作用系统：StoreSideEffects 生命周期

位置：src/lib/StoreSideEffects.ts

能力
- 注册 before/after create/change/delete 处理器（可按 typeName）
- registerOperationCompleteHandler：批处理结束后统一回调（与 Editor 的用法类似）
- 开关：`setIsEnabled(bool)`；Store.atomic 可临时关闭 before* 回调

在 Store 中的调用点（src/lib/Store.ts）
- put：`handleBeforeChange/handleBeforeCreate` → 校验 → 写入 → `addDiffForAfterEvent` → history；after* 在 flush 时机统一触发
- remove：`handleBeforeDelete` 可阻止删除 → deleteMany → `addDiffForAfterEvent` → history
- atomic：嵌套/顶层事务收敛 `pendingAfterEvents`，在 `flushAtomicCallbacks` 统一执行 after* 与 operationCompleteHandlers

建议
- 业务一致性放到 after* 与 operationComplete 阶段，避免在 before* 中做复杂写入；
- 防环：注意 side effect 循环触发；必要时通过 `isEnabled` 或 `ChangeSource` 做保护；
- 批量：在 `atomic(runCallbacks=false)` 期间屏蔽 before*，待批量写完再手动处理所需副作用。

