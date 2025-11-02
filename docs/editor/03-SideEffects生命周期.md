# SideEffects 生命周期

SideEffects 负责在 Store 操作完成后统一触发形状/绑定的后置逻辑，保证数据一致性与派生更新。

核心挂载点
- `operationComplete` 汇总后置处理：packages/editor/src/lib/editor/Editor.ts:451
  - 清空临时集合（deletedShapeIds/invalidParents/invalidBindingTypes 等）
  - 对无效父亲调用 `util.onChildrenChange` 并 `updateShapes`
  - 对绑定类型统一调用 `util.onOperationComplete`
  - 对记录的删除绑定调用 `util.onAfterDelete(opts)`
  - 最后 `emit('update')` 触发外部订阅

形状生命周期
- `shape.afterChange(shapeBefore, shapeAfter)` packages/editor/src/lib/editor/Editor.ts:492
  - 找出与该 shape 相关的所有 bindings → 标记 `invalidBindingTypes`
  - 分别调用 `onAfterChangeFromShape`、`onAfterChangeToShape`（reason: 'self'）
  - 当父级变化时，递归后代并以 reason:'ancestry' 通知绑定侧
  - 若跨页移动，清理其他页面的 PageState（选区/擦除/hover/编辑/聚焦）

- `shape.beforeDelete(shape)` packages/editor/src/lib/editor/Editor.ts:602
  - 记录将删除的 shapeId，父级标为 `invalidParents`
  - 收集涉及的 bindingId，分别调用 `onBeforeIsolate*/onBeforeDelete*`
  - 统一 `deleteBindings()` 后更新 PageState

绑定生命周期（由 shape 钩子触发 + operationComplete）
- `onAfterChangeFromShape/onAfterChangeToShape`：自/他端形状参数变化
- `onBeforeIsolateToShape/onBeforeIsolateFromShape`：父子/依赖关系变化前的隔离处理
- `onBeforeDeleteFromShape/onBeforeDeleteToShape`：删除前的两端回调
- `onAfterDelete(opts)`：删除完成后清理收尾
- `onOperationComplete()`：一次事务合并后的批量后处理

设计要点
- 生命周期解耦：删除/变更时只记录必要信息，最终在 `operationComplete` 统一收敛，避免重复计算与抖动。
- 绑定一致性：任何形状的父子/位置/参数变化都可能影响绑定，需完整通知两端 util。
- PageState 清理：跨页或删除后若不清理，UI 会出现“悬空”状态（选中/编辑等）。

