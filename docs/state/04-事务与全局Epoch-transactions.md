# 事务与全局 Epoch（transactions.ts）

位置：src/lib/transactions.ts

核心概念
- 全局 epoch：`advanceGlobalEpoch/getGlobalEpoch`；每次 Atom.set/Computed 求值更新都会前进，用于脏检查。
- Reacting 状态：`getIsReacting/getReactionEpoch`；在反应执行期间，Computed 可用 `lastTraversedEpoch` 避免重复遍历。
- 事务：
  - `transact(fn)`：批量变更原子值，期间仅在最后统一触发依赖；
  - `transaction((rollback) => { ...; rollback() })`：支持显式回滚；
  - 维护 `initialAtomValues`（旧值）、`cleanupReactors`（副作用清理）。

行为
- 在事务内的多次 set，不会为每次 set 单独触发一轮完整的反应图；
- 出错时回滚到进入事务前的值；
- 事务退出时推进 epoch 并调度需要执行的反应。

建议
- UI 大批量写入/合并远端数据时总是包裹在 `transact` 中；
- 在计算/副作用内不要开启嵌套事务，避免复杂时序；
- 通过 `getIsReacting` 避免在反应执行期间触发新的反应导致循环。

