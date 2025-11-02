# 状态与 Store 集成

全局状态分三层：Document、Instance、PageState，均存储于 Store（@tldraw/store）并由 signals 驱动。

Document（全局设置）
- 读取：`getDocumentSettings()` packages/editor/src/lib/editor/Editor.ts:1472
- 更新：`updateDocumentSettings(partial)` packages/editor/src/lib/editor/Editor.ts:1481

Instance（本地实例）
- 读取：`getInstanceState()` packages/editor/src/lib/editor/Editor.ts:1498
- 更新：`updateInstanceState(partial, opts?)` packages/editor/src/lib/editor/Editor.ts:1510
  - 特殊键 `isChangingStyle` 自动 1s 复位，防止 UI 闪烁（:1516）。
- 游标：`setCursor(partial)` packages/editor/src/lib/editor/Editor.ts:1559
- 菜单：`menus = tlmenus.forContext(this.contextId)` packages/editor/src/lib/editor/Editor.ts:1549

PageState（每页 UI 状态）
- 列表：`getPageStates()` packages/editor/src/lib/editor/Editor.ts:1571
- 当前页面：`getCurrentPageState()` packages/editor/src/lib/editor/Editor.ts:1585
- 更新当前页：`updateCurrentPageState(partial)` packages/editor/src/lib/editor/Editor.ts:1606
- 选择：
  - 读取：`getSelectedShapeIds()`（:1626）、`getSelectedShapes()`（:1636）
  - 设置：`setSelectedShapes(ids)`（:1653）

数据一致性与清理
- 跨页移动/删除需清理各 PageState 子字段（选区/擦除/悬停/编辑/聚焦组）：`cleanupInstancePageState`（:398 起），在 SideEffects 的 `shape.afterChange/beforeDelete` 和 `operationComplete` 中统一触发。

设计要点
- 所有 UI/交互相关状态（如选择/编辑/hover/brush/光标等）皆存于 Store，React 层仅订阅变化；历史选项多为 `history:'ignore'`，避免污染文档历史。
- 通过 `@computed` 包装的读取 API，自动利用缓存与依赖跟踪实现最小重算。

