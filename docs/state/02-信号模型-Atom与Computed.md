# 信号模型：Atom 与 Computed

Atom（src/lib/Atom.ts）
- 可变信号：`set(value, diff?) / update(updater)`；写入前比较（`isEqual ?? equals`）短路；写入推进 `advanceGlobalEpoch()`。
- 依赖：`get()` 中 `maybeCaptureParent(this)`；`children: ArraySet<Child>` 维护反向依赖。
- 历史：可选 `historyLength` → `HistoryBuffer.pushEntry(prevEpoch, now, diff ?? computeDiff(...) ?? RESET_VALUE)`。
- getDiffSince(epoch)：如果 `epoch >= lastChangedEpoch` 返回 `EMPTY_ARRAY`，否则从 `historyBuffer` 查询；无历史返回 `RESET_VALUE`。
- 无捕获：`__unsafe__getWithoutCapture(ignoreErrors?)` 返回当前值，不建立依赖。

Computed（src/lib/Computed.ts）
- derive：`(prevValue|UNINITIALIZED, lastComputedEpoch) => Value | withDiff(Value, Diff)`；支持增量计算
- 访问：`__unsafe__getWithoutCapture` 路径下：
  - 若 parents 未变更且缓存新鲜 → 直接返回；
  - 否则 `startCapturingParents()`，运行 derive，`stopCapturingParents()`，写入 state 与 `lastChangedEpoch`，更新 historyBuffer；错误时重置为 `UNINITIALIZED` 并清空历史。
- 捕获：`get()` 中先 `__unsafe__...()`，finally 中 `maybeCaptureParent(this)` 保证被动依赖也被记录。
- 活跃监听：`isActivelyListening = !children.isEmpty`；反压策略：在 reacting 时只要 `lastTraversedEpoch < getReactionEpoch()` 则允许跳过重算（避免重复）。
- 装饰器：`@computed` 支持 TC39/legacy 方法/Getter；`getComputedInstance(obj, 'method')` 可取底层 computed 实例。

增量与历史
- `withDiff(value, diff)` 搭配 `historyLength` 打开增量；无增量时 diff 记录为 `RESET_VALUE`；
- `getDiffSince(epoch)` 先无捕获求值以确保有最新父依赖，再查历史。

最佳实践
- 在 computed 内组合使用 `someAtom.get()`（建立依赖）与 `unsafe__withoutCapture(() => otherAtom.get())`（不建立依赖）；
- 对大对象的 computed 打开 `historyLength` 并在 derive 中返回 `withDiff`。

