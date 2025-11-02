# 效果调度：EffectScheduler、react 与 reactor

位置：src/lib/EffectScheduler.ts

概念
- react(name, fn, { scheduleEffect? })：立即创建反应（依赖追踪），当依赖变化时调度执行；可自定义调度器（如 requestAnimationFrame）。
- reactor(name, fn, opts)：返回具备 start/stop 控制的反应器，适合组件挂载/卸载周期管理。
- 清理：内部维护 cleanup 队列；事务/错误时保证不泄漏。

实践
- DOM 更新类副作用建议通过 `scheduleEffect: raf` 合并；
- 在组件层优先使用 reactor，可避免重复 start；
- 注意避免在反应函数内写入其依赖信号造成循环。

