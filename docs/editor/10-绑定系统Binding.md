# 绑定系统（Binding）

绑定描述形状间关系（如箭头 → 目标形状）。由 `BindingUtil` 承载行为定义，贯穿变更/删除生命周期。

注册与获取
- 注册：构造时注入并保存在 `bindingUtils`（packages/editor/src/lib/editor/Editor.ts:376）。
- 获取：`getBindingUtil(type|binding)`（packages/editor/src/lib/editor/Editor.ts:171）

生命周期
- 变更：`shape.afterChange` → `onAfterChangeFromShape/onAfterChangeToShape`（packages/editor/src/lib/editor/Editor.ts:492 起）
- 父系变化：以 reason:'ancestry' 通知后代相关绑定（:515 起）。
- 删除：`shape.beforeDelete` → `onBeforeIsolate*/onBeforeDelete*`，随后统一 `deleteBindings()`；在 `operationComplete` 中调用 `onAfterDelete`（:602 起；:479 起）。
- 完成：`onOperationComplete()`（operation 收敛回调，:470 起）。

设计要点
- 绑定是 Editor 的“二级约束层”，需跟随形状层级/几何变化保持同步。
- 生命周期回调必须幂等、防重入；在批处理事务中只记录必要信息，收敛后统一处理。

