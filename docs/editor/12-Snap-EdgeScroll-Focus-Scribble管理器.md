# Snap / EdgeScroll / Focus / Scribble 管理器

SnapManager（吸附）
- 提供吸附线与指示器，`snaps.getIndicators()` 被 UI 订阅以渲染（见 DefaultCanvas）。
- 配合选择/变换工具，实时计算吸附目标与对齐信息。

EdgeScrollManager（边缘滚动）
- 鼠标到画布边缘时自动滚动相机；与手势/相机状态机共用 TickManager 时间源。

FocusManager（焦点）
- 统一管理 Editor 的焦点状态，协调键盘事件与工具切换。

ScribbleManager（涂鸦）
- 用于刷选/临时轨迹显示，随 `tick` 推进衰减/清理。

设计要点
- 管理器是“横切关注点”的实现位置，尽量保持 Editor 内核的纯度。
- 与 UI 的交汇通过只读派生（如 getIndicators）实现，避免在 React 层计算业务数据。

