# 依赖捕获：capture 与父子关系

位置：src/lib/capture.ts

机制
- 全局捕获栈：`startCapturingParents(child)` 入栈，`maybeCaptureParent(parent)` 在 `parent.get()` 时注册依赖，`stopCapturingParents()` 出栈。
- Child 结构：保存 `parents/parentEpochs`；Parent 保存 `children: ArraySet<Child>`。
- 清理：当 child 不再被任何人引用（children 为空）或依赖变化时，会修剪父关系；降低内存与重算成本。

与 Computed 协作
- Computed 在每次求值之前 `startCapturingParents(this)`，之后 `stopCapturingParents()`；在 finally 中通过 `maybeCaptureParent(this)` 确保“被动读取”也记录依赖。
- `haveParentsChanged(this)`（helpers.ts）通过比较 `parentEpochs` 与父的 `lastChangedEpoch` 快速判定是否需要重算。

最佳实践
- 避免在副作用中直接 `get()` 大量信号导致多余依赖；必要时在副作用中用无捕获读取。
- 谨慎设计 computed 的父集合大小，必要时分层拆分。

