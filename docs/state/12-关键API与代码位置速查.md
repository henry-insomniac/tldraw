# 关键 API 与代码位置速查（起始行）

- Atom 实现/工厂：packages/state/src/lib/Atom.ts（~1–440）
- Computed 实现/装饰器：packages/state/src/lib/Computed.ts（~1–560 及后续）
- 依赖捕获：packages/state/src/lib/capture.ts
- EffectScheduler/react/reactor：packages/state/src/lib/EffectScheduler.ts
- 事务/epoch：packages/state/src/lib/transactions.ts
- 历史缓冲：packages/state/src/lib/HistoryBuffer.ts
- 工具：helpers/constants/isSignal/warnings：packages/state/src/lib/*.ts
- 本地持久化：packages/state/src/lib/localStorageAtom.ts

建议配合 ripgrep：
- `rg -n "advanceGlobalEpoch|getGlobalEpoch|haveParentsChanged" packages/state/src/lib`
- `rg -n "startCapturingParents|maybeCaptureParent|stopCapturingParents" packages/state/src/lib`
- `rg -n "react\(|reactor\(" packages/state/src/lib`

