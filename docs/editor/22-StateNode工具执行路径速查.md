# StateNode 工具执行路径速查

目标：快速明白“事件如何进入工具状态机、如何迁移子状态、工具如何读写 Editor/Store”。

核心文件
- 根状态：`packages/editor/src/lib/editor/tools/RootState.ts`
- 基类：`packages/editor/src/lib/editor/tools/StateNode.ts`
- 常用工具目录（选择、框选、变换等）：`packages/editor/src/lib/editor/tools/`

事件入口与派发
- React 捕获 DOM → `editor.dispatch(info)` → `editor._flushEventForTick(info)` → `root.handleEvent(info)`
- 定位：`packages/editor/src/lib/editor/Editor.ts:10177, 10209, 10696`

当前工具与路径
- 读：`editor.getCurrentTool()` / `editor.getCurrentToolId()`
- 路径：`editor.getPath()`（如 `select.brushing`）
- 切换：`editor.setCurrentTool('hand' | 'select' | ...)`

常见子状态（以 Select 为例）
- `Idle`：空闲、命中测试与 hover
- `Brushing`：框选
- `Translating`：拖拽移动
- `Resizing`：八向缩放
- `Rotating`：旋转
- `EditingShape`：文本/形状内部编辑

如何“看见”状态迁移（最小插桩）
1) 在 `StateNode.ts` 基类中，给 `transition(id, info)` 与 `onEnter/onExit` 打点：
```ts
// [debug]工具状态迁移（勿提交）
protected transition(id: string, info?: any) {
  try { console.debug('[tool.transition]', this.getPath?.(), '→', id, info?.name ?? info?.type ?? ''); } catch {}
  return super.transition(id, info)
}
onEnter = () => { try { console.debug('[tool.enter]', this.getPath?.()); } catch {} }
onExit  = () => { try { console.debug('[tool.exit ]', this.getPath?.()); } catch {} }
```

2) 在 `RootState.ts` 的 `handleEvent` 起始处增加：
```ts
// [debug]工具事件
try { console.debug('[tool.event]', info.type, (info as any).name ?? ''); } catch {}
```

3) 预期：
- 切换工具时出现 `tool.exit/enter/transition`
- 拖拽/框选时显示 `select.idle → select.brushing → select.idle` 等序列

“只在某工具打点”的做法（更干净）
- 复制一个工具，或在目标工具（如 `SelectTool`）的关键子状态覆写 `onEnter/onExit/onPointerDown/Move/Up` 加入 `console.debug`。
- 不改基类，避免影响全部工具。

快速编程实验（不进 UI）
- 使用测试编辑器：`packages/editor/src/lib/test/TestEditor.ts`
  ```ts
  import { TestEditor } from '.../TestEditor'
  const editor = new TestEditor()
  editor.setCurrentTool('select')
  editor.dispatch({ type: 'pointer', target:'canvas', name:'pointer_down', point:{x:0,y:0} })
  // 断言选区/状态迁移...
  ```

常见定位指引
- “这个事件为什么没生效？”
  - 看 `editor.wasEventAlreadyHandled(e)` 与 `markEventAsHandled`（DOM 层可能已拦截）
  - 查看 tool `isPenMode` / 只读 / 锁定 等前置条件
- “为什么进不了某子状态？”
  - 检查 `transition` 的条件（命中、阈值、键位）与 `canEnter`/`canExit`
  - 关注 Editor 中的阈值：`dragDistanceSquared`、`coarseDragDistanceSquared`

清理
- 搜索 `// [debug]` 并移除所有插桩

