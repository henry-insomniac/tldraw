# tldraw Editor 包架构深度分析

> 本文档深度探讨 @tldraw/editor 包的技术架构、Canvas 与 React 交互机制、以及系统设计考量

**目录版本**: v3.15.1
**最后更新**: 2025-10-27
**作者**: 架构分析文档

---

## 第一部分：架构概览

### 1.1 核心设计理念

tldraw editor 是一个**最小化基础编辑引擎**，采用以下核心设计原则：

#### 关键设计特点：

- **无形状耦合**: 核心引擎不包含任何具体形状实现（矩形、圆形等）
- **无UI耦合**: 没有内置UI工具栏、菜单等
- **完全可扩展**: 通过 ShapeUtil、BindingUtil、StateNode 等扩展点
- **高性能**: 使用信号系统、viewport culling、CSS transforms 等优化

### 1.2 系统分层架构

```
┌─────────────────────────────────────────────────────┐
│                  React 层                            │
│      (TldrawEditor 组件、Hooks、生命周期管理)       │
└─────────────────────────────────────────────────────┘
                        ↕
┌─────────────────────────────────────────────────────┐
│         Editor 核心编排层                            │
│  (事件分发、状态管理、API 入口)                      │
└─────────────────────────────────────────────────────┘
                        ↕
┌─────────────────────────────────────────────────────┐
│         响应式状态层 (@tldraw/state)                │
│  (Atoms、Computed、Events、Epochs)                  │
└─────────────────────────────────────────────────────┘
                        ↕
┌─────────────────────────────────────────────────────┐
│         数据持久化层 (@tldraw/store)                │
│  (Document Store、IndexedDB、Migrations)           │
└─────────────────────────────────────────────────────┘
                        ↕
┌─────────────────────────────────────────────────────┐
│         扩展层                                       │
│  (ShapeUtil、BindingUtil、Tools、Managers)         │
└─────────────────────────────────────────────────────┘
```

---

## 第二部分：响应式状态系统深度解析

### 2.1 @tldraw/state 信号系统

#### 2.1.1 核心概念：Epoch 和依赖追踪

tldraw 使用基于 **epoch** 的全局版本控制来追踪变更：

```typescript
// 全局单调递增的版本号
let globalEpoch = 0

// 当状态改变时
atom.set(newValue) // 触发 globalEpoch++

// Computed 通过比较 epoch 判断依赖是否改变
if (parentEpoch < globalEpoch) {
	// 需要重新计算
	recompute()
}
```

**优点**：

- 精确的依赖追踪，无需手动声明
- 自动化的缓存失效
- 高效的增量更新

#### 2.1.2 Atom - 可变状态容器

```typescript
class Atom<Value, Diff> {
	// 当前值
	private current: Value

	// 历史缓冲区（用于diff计算）
	historyBuffer?: HistoryBuffer<Diff>

	// 最后变更的 epoch
	lastChangedEpoch: number

	// 依赖此 Atom 的子信号
	children: Set<Child>

	set(value: Value, diff?: Diff) {
		// 1. 检查值是否真的改变
		if (equals(current, value)) return

		// 2. 前进全局 epoch
		advanceGlobalEpoch()

		// 3. 记录历史
		if (historyBuffer) {
			historyBuffer.pushEntry(
				lastChangedEpoch,
				getGlobalEpoch(),
				diff ?? computeDiff(current, value)
			)
		}

		// 4. 更新值和时间戳
		lastChangedEpoch = getGlobalEpoch()
		current = value

		// 5. 通知所有依赖者
		notifyChildren()
	}
}
```

**关键优化**：

- 值相等时为 no-op（防止级联更新）
- Diff 计算支持增量更新
- 历史缓冲区限制大小防止内存泄漏

#### 2.1.3 Computed - 派生状态

```typescript
class Computed<Value, Diff> {
	// 派生函数
	private derive: (prevValue, lastComputedEpoch) => Value

	// 缓存的计算值
	private state: Value

	// 追踪的父信号及其 epoch
	parentSet: Set<Signal>
	parentEpochs: number[]

	// 是否有活跃的监听者
	get isActivelyListening() {
		return !children.isEmpty
	}

	get(): Value {
		// 1. 检查是否需要重新计算
		if (!haveDependenciesChanged()) {
			return this.state
		}

		// 2. 开始捕获依赖
		startCapturingParents(this)

		// 3. 执行派生函数
		const result = this.derive(this.state, lastComputedEpoch)
		const newState = result.value

		// 4. 检查值是否真的改变
		if (!this.isEqual(newState, this.state)) {
			this.state = newState
			notifyChildren()
		}

		// 5. 停止捕获
		stopCapturingParents()

		return this.state
	}
}
```

**关键特性**：

- 懒评估（lazy evaluation）：只在被访问时计算
- 自动依赖追踪：通过访问 signal.get() 自动建立依赖关系
- 智能缓存：只有依赖改变时才重新计算
- 活跃监听检测：如果没有订阅者，可以被 GC

#### 2.1.4 React 集成：useValue 和 useQuickReactor

```typescript
// useValue: 读取 signal 值，订阅变更
export function useValue<Value>(name: string, fn: () => Value, deps: unknown[]): Value {
	const { $val, subscribe, getSnapshot } = useMemo(() => {
		// 创建或获取 computed signal
		const $val = computed(name, fn)

		return {
			$val,
			// 通过 react() 订阅变更
			subscribe: (notify: () => void) => {
				return react(`useValue(${name})`, () => {
					try {
						$val.get()
					} catch {
						// 忽略错误
					}
					notify()
				})
			},
			// useSyncExternalStore 需要的快照
			getSnapshot: () => $val.lastChangedEpoch,
		}
	}, deps)

	// 使用 React 的外部存储 hook 进行订阅
	useSyncExternalStore(subscribe, getSnapshot, getSnapshot)

	return $val.__unsafe__getWithoutCapture()
}

// useQuickReactor: 高性能的响应式副作用
export function useQuickReactor(name: string, effect: () => void, deps: unknown[]) {
	useLayoutEffect(() => {
		// 创建响应式反应，每当依赖改变时自动执行
		return react(name, effect)
	}, deps)
}
```

**React 集成策略**：

- `useValue`: 用于读取响应式状态并触发 React 重渲染
- `useQuickReactor`: 用于副作用，不需要 React 重渲染
- 使用 `useSyncExternalStore` 实现 React 18 并发安全性
- 分离"信号计算"和"React 渲染"关注点

---

## 第三部分：Editor 核心编排层

### 3.1 Editor 类架构

```typescript
export class Editor extends EventEmitter<TLEventMap> {
	// 核心依赖
	readonly store: TLStore
	readonly shapeUtils: Map<string, ShapeUtil>
	readonly bindingUtils: Map<string, BindingUtil>
	readonly tools: StateNode
	readonly user: TLUser

	// 响应式状态（sample）
	private _camera: Atom<TLCamera>
	private _selectedShapeIds: Atom<TLShapeId[]>
	private _currentPageId: Atom<TLPageId>
	private _instanceState: Atom<TLInstance>

	// 派生状态
	getSelectedShapes = computed(() => {
		return this._selectedShapeIds
			.get()
			.map((id) => this.store.get(id))
			.filter(isShape)
	})

	getSelectionPageBounds = computed(() => {
		return calcSelectionBounds(this.getSelectedShapes.get())
	})

	// 管理器（处理特定子领域）
	readonly snaps: SnapManager
	readonly history: HistoryManager
	readonly fonts: FontManager
	readonly text: TextManager
	readonly click: ClickManager

	constructor(options: TLEditorOptions) {
		// 初始化核心系统
		this.initializeStore()
		this.initializeShapeUtils()
		this.initializeTools()
		this.initializeManagers()
		this.setupEventHandling()
	}
}
```

### 3.2 事件处理流程

```
用户交互（指针、键盘等）
         ↓
React 事件处理器（useCanvasEvents）
         ↓
editor.dispatch(eventInfo)
         ↓
RootState.handleEvent(eventInfo)
         ↓
当前 StateNode.handleEvent(eventInfo)
         ↓
执行对应的 onPointerDown/onKeyDown 等
         ↓
修改响应式状态（store、camera 等）
         ↓
@computed 自动重新计算
         ↓
useValue 检测到变更
         ↓
React 重渲染相应组件
         ↓
DOM/Canvas 更新
```

### 3.3 事件类型和信息对象

```typescript
// 基础事件信息
interface TLEventInfo {
  type: 'pointer' | 'keyboard' | 'wheel' | 'tick' | 'pinch'
  target: 'canvas' | 'shape-handle'
  name: string  // 'pointer_down', 'pointer_move', etc.
}

// 指针事件（最常用）
interface TLPointerEventInfo extends TLEventInfo {
  type: 'pointer'
  name: 'pointer_down' | 'pointer_move' | 'pointer_up' | 'right_click'
  x: number  // 页面坐标
  y: number
  canvasX: number  // Canvas/工作区坐标
  canvasY: number
  shapeId?: TLShapeId  // 命中的形状
  handle?: TLHandle  // 命中的把手
  ...
}

// 状态节点中的处理
class MyTool extends StateNode {
  onPointerDown(info: TLPointerEventInfo) {
    const { canvasX, canvasY } = info

    // 创建形状、更新状态等
    editor.createShape({
      id: createShapeId(),
      type: 'my-shape',
      x: canvasX,
      y: canvasY
    })

    // 转移到子状态
    this.transition('dragging', info)
  }
}
```

---

## 第四部分：Canvas 与 React 深度交互

### 4.1 渲染管道架构

```typescript
// DefaultCanvas 组件 - 整体渲染容器
export function DefaultCanvas() {
  const editor = useEditor()

  return (
    <div className="tl-canvas">
      {/* SVG 上下文和定义 */}
      <svg className="tl-svg-context">
        <defs>
          {/* 形状的 SVG 定义（patterns, masks 等）*/}
          {shapeSvgDefs}
        </defs>
      </svg>

      {/* 背景图层 */}
      {Background && <Background />}

      {/* 网格 */}
      <GridWrapper />

      {/* HTML 形状图层（通过 CSS transform 定位）*/}
      <div className="tl-html-layer tl-shapes">
        {/* 每个形状由 Shape 组件渲染 */}
        <ShapesToDisplay />
        {/* 选择背景 */}
        <SelectionBackgroundWrapper />
      </div>

      {/* 覆盖层（指示器、手柄等）*/}
      <div className="tl-overlays">
        <BrushWrapper />
        <ShapeIndicators />
        <HandlesWrapper />
        <SnapIndicatorWrapper />
      </div>
    </div>
  )
}
```

### 4.2 相机和坐标转换系统

```typescript
// 相机状态（响应式）
interface TLCamera {
  x: number      // 视口 X 偏移
  y: number      // 视口 Y 偏移
  z: number      // 缩放级别
}

// 坐标转换 - 四个坐标系统
enum CoordinateSpace {
  Screen,   // 浏览器视口坐标
  Canvas,   // Canvas/工作区坐标（相对于 Canvas 元素）
  Page,     // 页面坐标（应用相机变换）
  Local     // 本地形状坐标（相对于形状父元素）
}

// 转换函数
class Editor {
  screenToCanvas(point: Vec): Vec {
    return point  // 直接映射（canvas 覆盖整个视口）
  }

  canvasToPage(point: Vec): Vec {
    const { x, y, z } = this.camera
    return {
      x: (point.x - x) / z,
      y: (point.y - y) / z
    }
  }

  pageToCanvas(point: Vec): Vec {
    const { x, y, z } = this.camera
    return {
      x: point.x * z + x,
      y: point.y * z + y
    }
  }

  getShapePageTransform(shapeId: TLShapeId): Mat {
    const shape = this.getShape(shapeId)
    // 计算形状的变换矩阵（位置、旋转、缩放）
    return Mat.compose(
      Mat.translate(shape.x, shape.y),
      Mat.rotate(shape.rotation),
      ...
    )
  }
}

// React 中应用相机变换
function DefaultCanvas() {
  useQuickReactor(
    'position layers',
    function positionLayersWhenCameraMoves() {
      const { x, y, z } = editor.getCamera()

      // 使用 CSS transform 移动整个形状图层
      const transform = `scale(${z}) translate(${x}px, ${y}px)`
      rHtmlLayer.current.style.transform = transform
    },
    [editor]
  )
}
```

**坐标系设计要点**：

- 分离多个坐标系统便于不同操作的计算
- Screen → Canvas 映射简单（Canvas 覆盖全视口）
- Canvas → Page 应用相机反向变换
- Page → Local 应用形状变换的逆矩阵

### 4.3 形状渲染和更新机制

```typescript
// Shape 组件 - 单个形状的渲染
export const Shape = memo(function Shape({
  id,
  shape,
  util,
  index,
  backgroundIndex,
  opacity
}) {
  const editor = useEditor()
  const containerRef = useRef<HTMLDivElement>(null)

  // 追踪变换、剪切路径等需要频繁更新的属性
  useQuickReactor(
    'set shape stuff',
    () => {
      const shape = editor.getShape(id)

      // 1. 页面变换（应用相机和形状变换）
      const pageTransform = editor.getShapePageTransform(id)
      const transform = Mat.toCssString(pageTransform)
      setStyleProperty(containerRef.current, 'transform', transform)

      // 2. 几何边界（宽高）
      const bounds = editor.getShapeGeometry(shape).bounds
      setStyleProperty(containerRef.current, 'width', bounds.width + 'px')
      setStyleProperty(containerRef.current, 'height', bounds.height + 'px')

      // 3. 剪切路径
      const clipPath = editor.getShapeClipPath(id) ?? 'none'
      setStyleProperty(containerRef.current, 'clip-path', clipPath)
    },
    [editor]
  )

  // 不常变化的属性（z-index、opacity）
  useLayoutEffect(() => {
    setStyleProperty(containerRef.current, 'z-index', index)
    setStyleProperty(containerRef.current, 'opacity', opacity)
  }, [index, opacity])

  // 是否被视口剪切（culling）
  useQuickReactor(
    'set display',
    () => {
      const isCulled = editor.getCulledShapes().has(id)
      setStyleProperty(containerRef.current, 'display',
        isCulled ? 'none' : 'block')
    },
    [editor]
  )

  return (
    <div ref={containerRef} className="tl-shape-wrapper">
      {/* 背景组件（如果定义）*/}
      {util.backgroundComponent && (
        <InnerShapeBackground shape={shape} util={util} />
      )}
      {/* 主体组件 */}
      <InnerShape shape={shape} util={util} />
    </div>
  )
})

// InnerShape - 形状内容的实际渲染
export const InnerShape = memo(
  function InnerShape({ shape, util }) {
    return useStateTracking(
      'InnerShape:' + shape.type,
      () => {
        // 总是从 store 获取最新的形状数据，避免陈旧数据
        const latestShape = editor.store
          .unsafeGetWithoutCapture(shape.id)

        // 调用 ShapeUtil 的渲染方法
        return util.component(latestShape)
      },
      [util, shape.id]
    )
  },
  // 自定义比较：只有形状内容改变时才重新渲染
  (prev, next) => areShapesContentEqual(prev.shape, next.shape)
)
```

**性能优化策略**：

- 分离频繁变化（transform）和不常变化（opacity）的更新
- 使用 `useQuickReactor` 进行 DOM 操作，避免 React 重渲染
- 直接 DOM 操作（setStyleProperty）比 React 状态更新快
- Viewport culling 隐藏不可见形状（display: none）
- Shape memo + areShapesContentEqual 防止不必要的渲染

### 4.4 指示器系统（Indicators）

```typescript
// Indicators 是形状选中和交互时的视觉反馈
// 通过 SVG 渲染，高性能

function ShapeIndicators() {
  const editor = useEditor()

  return editor.getSelectedShapeIds().map(id => (
    <ShapeIndicator key={id} shapeId={id} />
  ))
}

// ShapeIndicator 使用 ShapeUtil 提供的指示器
function ShapeIndicator({ shapeId }) {
  const editor = useEditor()
  const shape = editor.getShape(shapeId)
  const util = editor.getShapeUtil(shape)
  const transform = editor.getShapePageTransform(shapeId)

  return (
    <g transform={Mat.toCssString(transform)}>
      {/* 形状 util 定义的指示器 SVG */}
      {util.indicator(shape)}
    </g>
  )
}

// 在 ShapeUtil 中定义指示器
class MyShapeUtil extends ShapeUtil {
  indicator(shape: MyShape) {
    const bounds = this.getGeometry(shape).bounds
    return (
      <rect
        x={bounds.minX}
        y={bounds.minY}
        width={bounds.width}
        height={bounds.height}
        fill="none"
        stroke="blue"
        strokeWidth="2"
      />
    )
  }
}
```

---

## 第五部分：工具系统和状态机

### 5.1 StateNode 层级状态机

```typescript
// 工具通过 StateNode 的层级实现复杂交互

export class SelectTool extends StateNode {
	static id = 'select'
	static children = () => [IdlingState, BrushingState, TranslatingState]
	static initial = 'idling'

	onEnter() {
		editor.setCursor({ type: 'default' })
	}
}

// 子状态
export class IdlingState extends StateNode {
	static id = 'idling'

	onPointerDown(info: TLPointerEventInfo) {
		if (info.shapeId) {
			// 点击形状
			editor.select(info.shapeId)
			this.parent.transition('translating', info)
		} else {
			// 点击空白区域，开始框选
			this.parent.transition('brushing', info)
		}
	}
}

export class BrushingState extends StateNode {
	static id = 'brushing'

	private brushStart: Vec

	onEnter(info: TLPointerEventInfo) {
		this.brushStart = { x: info.canvasX, y: info.canvasY }
		editor.setBrush(this.brushStart)
	}

	onPointerMove(info: TLPointerEventInfo) {
		// 更新选择框
		editor.setBrush({
			...this.brushStart,
			width: info.canvasX - this.brushStart.x,
			height: info.canvasY - this.brushStart.y,
		})
	}

	onPointerUp() {
		// 完成选择，返回到 idling
		editor.clearBrush()
		this.parent.transition('idling')
	}

	onExit() {
		editor.clearBrush()
	}
}

export class TranslatingState extends StateNode {
	static id = 'translating'

	private initialPosition: Vec
	private initialPagePosition: Vec

	onEnter(info: TLPointerEventInfo) {
		this.initialPosition = { x: info.canvasX, y: info.canvasY }
		this.initialPagePosition = editor.screenToPage(this.initialPosition)
	}

	onPointerMove(info: TLPointerEventInfo) {
		const delta = {
			x: (info.canvasX - this.initialPosition.x) / editor.zoomLevel,
			y: (info.canvasY - this.initialPosition.y) / editor.zoomLevel,
		}

		editor.updateSelectedShapes((shapes) =>
			shapes.map((shape) => ({
				...shape,
				x: shape.x + delta.x,
				y: shape.y + delta.y,
			}))
		)
	}

	onPointerUp() {
		// 完成拖动
		editor.history.commit()
		this.parent.transition('idling')
	}
}
```

**状态机设计要点**：

- 层级结构：RootState → Tools → ChildStates
- 事件在激活的状态链中传播（叶 → 根）
- onEnter/onExit 钩子用于状态初始化和清理
- 状态转移通过 transition() 方法
- 每个状态可以处理多种事件（pointer、keyboard、tick）

### 5.2 StateNode 中的响应式状态

```typescript
// StateNode 使用 Atom 追踪状态机状态
class StateNode {
	private _isActive = atom<boolean>('toolIsActive' + this.id, false)
	private _current = atom<StateNode | undefined>('toolState' + this.id, undefined)

	private _path = computed('toolPath' + this.id, () => {
		const current = this.getCurrent()
		return this.id + (current ? `.${current.getPath()}` : '')
	})

	// 状态查询
	getIsActive(): boolean {
		return this._isActive.get()
	}

	getCurrent(): StateNode | undefined {
		return this._current.get()
	}

	getPath(): string {
		return this._path.get()
	}

	// 状态转移
	transition(id: string, info: any = {}) {
		const path = id.split('.')
		let currState = this

		for (let i = 0; i < path.length; i++) {
			const id = path[i]
			const prevChildState = currState.getCurrent()
			const nextChildState = currState.children?.[id]

			if (prevChildState?.id !== nextChildState.id) {
				prevChildState?.exit(info, id)
				currState._current.set(nextChildState)
				nextChildState.enter(info, prevChildState?.id || 'initial')
			}

			currState = nextChildState
		}
	}
}
```

---

## 第六部分：形状系统和扩展点

### 6.1 ShapeUtil 接口设计

```typescript
export abstract class ShapeUtil<Shape extends TLUnknownShape> {
	constructor(public editor: Editor) {}

	// ========== 必须实现 ==========

	/**
	 * 获取形状的默认属性
	 */
	abstract getDefaultProps(): Shape['props']

	/**
	 * 获取形状的几何表示
	 * 用于碰撞检测、选择、绑定等
	 */
	abstract getGeometry(shape: Shape, opts?: TLGeometryOpts): Geometry2d

	/**
	 * 返回形状的 React 组件（HTML 元素）
	 * 这是形状最终在 canvas 上的样子
	 */
	abstract component(shape: Shape): ReactElement

	/**
	 * 返回形状选中时的指示器（SVG 元素）
	 */
	abstract indicator(shape: Shape): ReactElement

	// ========== 交互回调 ==========

	onBeforeCreate?(next: Shape): Shape | void
	onBeforeUpdate?(prev: Shape, next: Shape): Shape | void

	onResizeStart?(shape: Shape): TLShapePartial<Shape> | void
	onResize?(shape: Shape, info: TLResizeInfo<Shape>): TLShapePartial<Shape> | void
	onResizeEnd?(initial: Shape, current: Shape): TLShapePartial<Shape> | void

	onTranslateStart?(shape: Shape): TLShapePartial<Shape> | void
	onTranslate?(initial: Shape, current: Shape): TLShapePartial<Shape> | void
	onTranslateEnd?(initial: Shape, current: Shape): TLShapePartial<Shape> | void

	onRotateStart?(shape: Shape): TLShapePartial<Shape> | void
	onRotate?(initial: Shape, current: Shape): TLShapePartial<Shape> | void
	onRotateEnd?(initial: Shape, current: Shape): TLShapePartial<Shape> | void

	onHandleDragStart?(shape: Shape, info: TLHandleDragInfo<Shape>): TLShapePartial<Shape> | void
	onHandleDrag?(shape: Shape, info: TLHandleDragInfo<Shape>): TLShapePartial<Shape> | void
	onHandleDragEnd?(current: Shape, info: TLHandleDragInfo<Shape>): TLShapePartial<Shape> | void

	onDoubleClick?(shape: Shape): TLShapePartial<Shape> | void
	onClick?(shape: Shape): TLShapePartial<Shape> | void

	// ========== 功能查询 ==========

	canResize(shape: Shape): boolean {
		return true
	}
	canRotate(shape: Shape): boolean {
		return true
	}
	canBind(opts: TLShapeUtilCanBindOpts): boolean {
		return true
	}
	canEdit(shape: Shape): boolean {
		return false
	}
	canCrop(shape: Shape): boolean {
		return false
	}

	// ========== 导出和 SVG ==========

	toSvg?(shape: Shape, ctx: SvgExportContext): ReactElement | Promise<ReactElement>
	getCanvasSvgDefs(): TLShapeUtilCanvasSvgDef[] {
		return []
	}

	// ========== 其他 ==========

	getHandles?(shape: Shape): TLHandle[]
	getFontFaces(shape: Shape): TLFontFace[] {
		return []
	}
	getText(shape: Shape): string | undefined {
		return undefined
	}
	getInterpolatedProps?(startShape: Shape, endShape: Shape, progress: number): Shape['props']
}
```

### 6.2 ShapeUtil 实现示例

```typescript
// 实现一个自定义"评论框"形状
type CommentShape = TLBaseShape<'comment', {
  w: number
  h: number
  text: string
  color: string
}>

class CommentShapeUtil extends ShapeUtil<CommentShape> {
  static type = 'comment' as const

  getDefaultProps(): CommentShape['props'] {
    return {
      w: 200,
      h: 100,
      text: '',
      color: '#ffd700'
    }
  }

  getGeometry(shape: CommentShape) {
    return new Rectangle2d({
      width: shape.props.w,
      height: shape.props.h
    })
  }

  component(shape: CommentShape) {
    return (
      <div
        className="comment-box"
        style={{
          width: shape.props.w,
          height: shape.props.h,
          backgroundColor: shape.props.color,
          padding: 8,
          borderRadius: 4,
          position: 'relative'
        }}
      >
        <textarea
          value={shape.props.text}
          onChange={(e) => {
            this.editor.updateShape({
              id: shape.id,
              type: 'comment',
              props: { ...shape.props, text: e.currentTarget.value }
            })
          }}
          style={{
            width: '100%',
            height: '100%',
            border: 'none',
            background: 'transparent',
            resize: 'none'
          }}
        />
      </div>
    )
  }

  indicator(shape: CommentShape) {
    return (
      <rect
        width={shape.props.w}
        height={shape.props.h}
        fill="none"
        stroke="#999"
        strokeWidth="1"
        rx="4"
      />
    )
  }

  onResize(shape: CommentShape, info: TLResizeInfo<CommentShape>) {
    return {
      props: {
        w: Math.max(100, info.initialBounds.width * info.scaleX),
        h: Math.max(50, info.initialBounds.height * info.scaleY)
      }
    }
  }

  canEdit() { return true }
  canCrop() { return false }

  getText(shape: CommentShape) {
    return shape.props.text
  }
}

// 注册形状
<TldrawEditor
  shapeUtils={[CommentShapeUtil]}
  // ...
/>
```

---

## 第七部分：性能优化深度分析

### 7.1 Viewport Culling（视口剪切）

```typescript
class Editor {
	/**
	 * 获取当前视口内的形状
	 * 只有这些形状会被实际渲染
	 */
	getVisibleShapeIds(): TLShapeId[] {
		const viewportBounds = this.getViewportPageBounds()

		return this.store
			.allShapes()
			.filter((shape) => {
				const bounds = this.getShapePageBounds(shape)
				return bounds && viewportBounds.intersects(bounds)
			})
			.map((shape) => shape.id)
	}

	/**
	 * 获取被视口剪切的形状
	 * 这些形状设置 display: none
	 */
	getCulledShapes(): Set<TLShapeId> {
		const visibleIds = new Set(this.getVisibleShapeIds())
		const allIds = new Set(this.store.allShapes().map((s) => s.id))

		return new Set([...allIds].filter((id) => !visibleIds.has(id)))
	}
}

// React 中使用
function Shape({ id }) {
	useQuickReactor(
		'set display',
		() => {
			const isCulled = editor.getCulledShapes().has(id)
			// display: none 比完全删除 DOM 快得多
			setStyleProperty(containerRef.current, 'display', isCulled ? 'none' : 'block')
		},
		[editor]
	)
}
```

**优点**：

- DOM 节点仍然存在（保留索引、引用等）
- 只是隐藏（display: none），渲染流程跳过
- 对于大型文档显著减少渲染成本

### 7.2 几何缓存

```typescript
class Editor {
	// 几何缓存：形状 ID → 几何对象
	private _geometryCache = new ComputedCache<TLShapeId, Geometry2d>('geometry', (shapeId) => {
		const shape = this.store.get(shapeId)
		if (!shape || !isShape(shape)) return undefined

		const shapeUtil = this.getShapeUtil(shape)
		return shapeUtil.getGeometry(shape)
	})

	getShapeGeometry(shape: TLShape): Geometry2d {
		return this._geometryCache.get(shape.id)!
	}

	// 当形状改变时，缓存自动失效
	// ComputedCache 使用形状的 lastChangedEpoch
}
```

**设计要点**：

- Geometry 计算是昂贵操作（尤其对复杂形状）
- ComputedCache 基于信号系统，自动追踪形状变更
- 形状改变 → shape atom epoch 前进 → 缓存失效 → 下次访问重新计算

### 7.3 CSS Transform 优化

```typescript
// ❌ 不好：重排（reflow）和重绘（repaint）
shape.style.left = x + 'px'
shape.style.top = y + 'px'

// ✅ 好：只触发合成（composition），GPU 加速
shape.style.transform = `translate(${x}px, ${y}px)`

// ✅ 更好：仅改变 GPU 属性，性能最佳
shape.style.transform = `translate3d(${x}px, ${y}px, 0)`
```

**tldraw 的应用**：

```typescript
// 形状变换通过 CSS transform 应用
const pageTransform = editor.getShapePageTransform(shapeId)
setStyleProperty(containerRef.current, 'transform', Mat.toCssString(pageTransform))

// 相机移动也是 transform
const { x, y, z } = editor.getCamera()
const transform = `scale(${z}) translate(${x}px, ${y}px)`
setStyleProperty(rHtmlLayer.current, 'transform', transform)
```

**性能收益**：

- 避免布局重算（计算昂贵）
- GPU 加速变换
- 60fps 流畅交互

### 7.4 React 优化

```typescript
// ❌ 问题：Shape 组件在任何形状改变时重新渲染
function ShapesContainer() {
  const shapes = useValue('shapes', () => editor.getShapes(), [editor])

  return shapes.map(shape => (
    <Shape key={shape.id} shape={shape} />
  ))
}

// ✅ 解决：逐个追踪形状，只重新渲染改变的形状
function ShapesToDisplay() {
  const renderingShapes = useValue(
    'rendering shapes',
    () => editor.getRenderingShapes(),
    [editor]
  )

  return renderingShapes.map(result => (
    <Shape key={result.id} {...result} />
  ))
}

// ✅ Shape 组件：细粒度重渲染控制
export const Shape = memo(
  function Shape({ id, shape, util }) {
    // ... 组件代码
  },
  // 自定义比较：只在形状内容改变时重新渲染
  (prev, next) =>
    areShapesContentEqual(prev.shape, next.shape) &&
    prev.util === next.util
)

// ✅ InnerShape：使用 useStateTracking 实现最细粒度的响应
export const InnerShape = memo(
  function InnerShape({ shape, util }) {
    return useStateTracking(
      'InnerShape:' + shape.type,
      () => {
        // 总是从 store 获取最新形状
        const latestShape = editor.store
          .unsafeGetWithoutCapture(shape.id)
        // 调用 util 的 component 方法
        return util.component(latestShape)
      },
      [util, shape.id]
    )
  },
  (prev, next) => areShapesContentEqual(prev.shape, next.shape)
)
```

**优化层次**：

1. **React 级别**: memo + 自定义比较
2. **副作用级别**: useQuickReactor（直接 DOM 操作）
3. **样式级别**: CSS transform（GPU 加速）
4. **渲染级别**: Culling（隐藏不可见形状）

---

## 第八部分：管理器系统

### 8.1 核心管理器概览

```typescript
class Editor {
	// 核心管理器
	readonly snaps: SnapManager // 形状对齐吸附
	readonly history: HistoryManager // 撤销/重做
	readonly fonts: FontManager // 字体加载和管理
	readonly text: TextManager // 文本测量和输入
	readonly click: ClickManager // 多击检测（点击/双击/三击）
	readonly edge: EdgeScrollManager // 视口边缘自动滚动
	readonly focus: FocusManager // 焦点和键盘事件
	readonly scribble: ScribbleManager // 涂鸦/笔刷绘制
	readonly tick: TickManager // 动画帧管理
	readonly user: UserPreferencesManager // 用户偏好设置
}
```

### 8.2 SnapManager 实现示例

```typescript
class SnapManager {
	private _snapIndicators = atom<SnapIndicator[]>('snap indicators', [])

	/**
	 * 在拖动/调整大小时自动吸附到其他形状
	 */
	snapPoint(
		point: Vec,
		options: {
			snapDistance?: number
			excludeIds?: TLShapeId[]
		} = {}
	): { snappedPoint: Vec; snapLines: Line[] } {
		const snapDistance = options.snapDistance ?? 8 / editor.zoomLevel
		const excludeIds = new Set(options.excludeIds)

		let snappedX = point.x
		let snappedY = point.y
		const snapLines: Line[] = []

		// 检查所有其他形状的边界
		for (const shape of editor.getShapes()) {
			if (excludeIds.has(shape.id)) continue

			const bounds = editor.getShapePageBounds(shape)
			if (!bounds) continue

			// X 轴吸附
			const xDist = Math.min(
				Math.abs(point.x - bounds.minX),
				Math.abs(point.x - bounds.maxX),
				Math.abs(point.x - bounds.centerX)
			)

			if (xDist < snapDistance) {
				// 确定吸附到哪条线
				if (Math.abs(point.x - bounds.minX) < Math.abs(point.x - bounds.maxX)) {
					snappedX = bounds.minX
				} else {
					snappedX = bounds.maxX
				}

				snapLines.push({
					type: 'vertical',
					x: snappedX,
					y1: Math.min(point.y, bounds.minY),
					y2: Math.max(point.y, bounds.maxY),
				})
			}

			// Y 轴吸附（类似）
			// ...
		}

		return {
			snappedPoint: { x: snappedX, y: snappedY },
			snapLines,
		}
	}

	/**
	 * 显示吸附线指示器
	 */
	getIndicators(): SnapIndicator[] {
		return this._snapIndicators.get()
	}
}

// 在拖动工具中使用
class TranslatingState extends StateNode {
	onPointerMove(info: TLPointerEventInfo) {
		let newPoint = editor.screenToPage({ x: info.x, y: info.y })

		// 应用吸附
		const { snappedPoint, snapLines } = editor.snaps.snapPoint(newPoint, {
			excludeIds: editor.getSelectedShapeIds(),
		})

		newPoint = snappedPoint

		// 显示吸附线
		editor.snaps._snapIndicators.set(snapLines)

		// 移动形状
		editor.updateSelectedShapes(/* ... */)
	}
}
```

---

## 第九部分：重新设计方案

### 9.1 架构改进方向

#### 9.1.1 现有架构的优势

✅ **分离关注点**

- Core Engine（无形状、无UI）
- 响应式状态系统（自动依赖追踪）
- 可插拔的扩展系统

✅ **性能优化**

- Epoch 系统精确追踪变更
- Viewport culling
- CSS transform 优化
- 细粒度响应式更新

✅ **可扩展性**

- ShapeUtil、BindingUtil、StateNode 扩展点
- Tool 状态机灵活性
- Manager 模式处理特定子领域

#### 9.1.2 改进建议

**方向1：增强状态管理的原子性**

```typescript
// 目前的问题：多个 atom 更新可能不同步
editor.updateSelectedShapeIds(['shape1', 'shape2']) // atom 1
editor.setCamera({ x: 100, y: 200 }) // atom 2

// 改进：使用事务确保原子性
editor.transaction(() => {
	editor.updateSelectedShapeIds(['shape1', 'shape2'])
	editor.setCamera({ x: 100, y: 200 })
	// 两个更新作为单一 epoch 变更
})
```

**方向2：改进形状的增量更新**

```typescript
// 目前：频繁的完整形状对象替换
editor.updateShape({
	id: shapeId,
	type: 'rect',
	props: { ...shape.props, width: 100 },
})

// 改进：支持更细粒度的 patch 更新
editor.patchShape(shapeId, {
	props: { width: 100 }, // 只更新改变的属性
})

// 实现：Immer 或类似库实现结构共享
```

**方向3：增强工具系统**

```typescript
// 目前问题：工具状态机比较复杂
// 改进：引入更高阶的工具抽象

class DrawingTool extends Tool {
	// 声明式定义状态流转
	states = {
		idle: { on: { pointerDown: 'drawing' } },
		drawing: { on: { pointerMove: 'drawing', pointerUp: 'idle' } },
	}

	// 简化事件处理
	onStateChange(from: string, to: string) {}
}
```

**方向4：改进渲染性能**

```typescript
// 目前：所有形状 DOM 节点始终存在
// 改进：考虑虚拟化（virtualization）

class VirtualizedShapeContainer {
	// 只渲染视口内的形状 DOM 节点
	// 不可见形状创建虚拟占位符
	renderVisibleShapes() {}
}

// 权衡：需要考虑滚动/平移的流畅性
```

### 9.2 分层架构优化

```
改进前（单层 Signal 系统）：
┌─────────────────────────────────────────┐
│   所有状态都是基础 Atom/Computed         │
│   （相机、选择、工具状态、形状数据等）   │
└─────────────────────────────────────────┘

改进后（多层次抽象）：
┌─────────────────────────────────────────┐
│   第3层：业务逻辑状态                    │
│   （工具状态、交互状态）                 │
├─────────────────────────────────────────┤
│   第2层：编辑器状态                      │
│   （相机、选择、viewport 等）            │
├─────────────────────────────────────────┤
│   第1层：文档数据（Store）               │
│   （形状、绑定、页面等）                 │
├─────────────────────────────────────────┤
│   第0层：基础信号系统                    │
│   （Atom、Computed、Events）             │
└─────────────────────────────────────────┘
```

### 9.3 集成第三方库的考量

**WebGPU 渲染层**（长期方向）

```typescript
// 目前：HTML/SVG/CSS 混合渲染
// 前景：使用 WebGPU 进行高性能图形渲染

class WebGPURenderer {
	// 将形状渲染卸载到 GPU
	// 支持更复杂的效果（阴影、模糊等）
	// 更好的大型文档性能
}
```

**Immer 用于不可变更新**

```typescript
// 改进形状更新的易用性
import produce from 'immer'

editor.updateShape(shapeId, (draft) => {
	draft.props.width = 100
	draft.props.color = '#ff0000'
})
```

**Web Workers 用于后台计算**

```typescript
// 卸载昂贵计算
class ComputeWorker {
	// 几何计算
	// 碰撞检测
	// 导出处理
}
```

---

## 第十部分：关键技术决策和权衡

### 10.1 为什么选择 React？

| 方面         | React                     | 替代方案       | 权衡                         |
| ------------ | ------------------------- | -------------- | ---------------------------- |
| **组件系统** | 声明式、基于 JSX          | Vue/Svelte     | React 生态最大，学习资源多   |
| **状态管理** | useSyncExternalStore 集成 | 自定义方案     | 内置支持外部状态，无需 Redux |
| **性能**     | Concurrent、Suspense      | 自定义调度     | React 18+ 很好的并发支持     |
| **学习曲线** | 中等                      | 陡峭（自定义） | 权衡生产力和灵活性           |

### 10.2 为什么用 Signals 而不是 Redux？

| 指标             | Signals     | Redux                     |
| ---------------- | ----------- | ------------------------- |
| **自动依赖追踪** | ✅ 是       | ❌ 需要 selectors         |
| **性能**         | ✅ 精确订阅 | ⚠️ 每次 dispatch 遍历     |
| **学习成本**     | ✅ 低       | ❌ 高（actions/reducers） |
| **时间旅行调试** | ⚠️ 有限     | ✅ 完善支持               |
| **生态**         | ⚠️ 小       | ✅ 大                     |

**结论**: Signals 更适合实时交互应用（即时响应、低延迟）。

### 10.3 为什么 CSS Transform 而不是 Canvas？

| 方面             | CSS Transform   | Canvas 绘制       |
| ---------------- | --------------- | ----------------- |
| **HTML 元素**    | ✅ 完全支持     | ❌ 需要特殊处理   |
| **文本编辑**     | ✅ 原生输入框   | ❌ 需要自定义输入 |
| **可访问性**     | ✅ 屏幕阅读器   | ❌ 完全自定义     |
| **交互**         | ✅ 点击、焦点等 | ⚠️ 需要手动实现   |
| **性能**         | ✅ GPU 加速     | ⚠️ CPU 密集       |
| **最终像素控制** | ❌ 受限         | ✅ 完全控制       |

**结论**: 混合方案（HTML + SVG indicators）最佳。

### 10.4 为什么用 Viewport Culling 而不是虚拟化？

| 考量           | Culling     | 虚拟化          |
| -------------- | ----------- | --------------- |
| **实现复杂度** | ✅ 简单     | ❌ 复杂         |
| **大型文档**   | ✅ 可处理   | ✅ 可处理       |
| **交互流畅度** | ✅ 很好     | ⚠️ 需要调优     |
| **内存占用**   | ⚠️ 更多 DOM | ✅ 更少         |
| **代码稳定性** | ✅ 成熟     | ❌ 更多边界情况 |

**结论**: 对 tldraw 的用例，culling 足够好，虚拟化复杂度不值得。

---

## 第十一部分：深度技术细节

### 11.1 全局 Epoch 系统的精妙设计

```typescript
// 核心机制
let globalEpoch = 0

function advanceGlobalEpoch() {
	globalEpoch++
}

// Atom 记录最后变更的 epoch
class Atom {
	lastChangedEpoch = globalEpoch

	set(value) {
		if (value === current) return // no-op

		advanceGlobalEpoch()
		current = value
		lastChangedEpoch = globalEpoch

		notifyChildren()
	}
}

// Computed 检查依赖的 epoch
class Computed {
	parentEpochs: number[] = []
	lastCheckedEpoch = globalEpoch

	haveParentsChanged(): boolean {
		for (let i = 0; i < this.parents.length; i++) {
			if (this.parentEpochs[i] < this.parents[i].lastChangedEpoch) {
				return true
			}
		}
		return false
	}

	get(): Value {
		if (!haveParentsChanged()) {
			return this.state // 缓存命中
		}

		// 重新计算...
		this.state = derive()
		return this.state
	}
}
```

**精妙之处**：

- 单个单调递增的版本号（globalEpoch）
- 无需维护复杂的依赖图
- 比较操作 O(1)（只比较数字）
- 自动处理间接依赖

### 11.2 事务系统中的原子性

```typescript
class Editor {
	private transactionDepth = 0
	private pendingUpdates: Array<() => void> = []

	/**
	 * 事务内所有更新作为单一 epoch 变更
	 */
	transaction<T>(fn: () => T): T {
		this.transactionDepth++

		try {
			return fn()
		} finally {
			this.transactionDepth--

			if (this.transactionDepth === 0) {
				// 事务完成，提交所有更新
				advanceGlobalEpoch()
				// 所有 Atom 更新现在共享相同的 epoch
			}
		}
	}

	// 或者使用 transact 函数
	batch(fn: () => void) {
		transact(fn) // from @tldraw/state
	}
}
```

### 11.3 内存管理和 GC 友好性

```typescript
// Computed 支持垃圾回收
class Computed {
	// 当没有活跃订阅者时可以被 GC
	get isActivelyListening(): boolean {
		return !this.children.isEmpty
	}

	// 可选：释放缓存
	dispose() {
		this.state = UNINITIALIZED
		this.children.clear()
	}
}

// React 中使用 WeakMap 避免内存泄漏
class ShapeGeometryCache {
	private cache = new WeakMap<TLShape, Geometry2d>()

	// shape 被删除时，缓存条目自动清理
	get(shape: TLShape): Geometry2d {
		if (!this.cache.has(shape)) {
			this.cache.set(shape, computeGeometry(shape))
		}
		return this.cache.get(shape)!
	}
}

// 历史缓冲区限制大小
class HistoryBuffer<T> {
	private entries: T[] = []
	private maxSize: number

	pushEntry(entry: T) {
		this.entries.push(entry)
		if (this.entries.length > this.maxSize) {
			this.entries.shift() // 丢弃最旧的
		}
	}
}
```

### 11.4 形状变换矩阵计算

```typescript
// 页面变换 = 相机变换 × 形状变换
class Editor {
	getShapePageTransform(shapeId: TLShapeId): Mat {
		const shape = this.getShape(shapeId)
		const parent = this.getShape(shape.parentId)

		// 1. 本地变换（相对于父元素）
		let transform = Mat.compose(
			Mat.translate(shape.x, shape.y),
			Mat.rotate(shape.rotation ?? 0),
			Mat.scale(shape.scaleX ?? 1, shape.scaleY ?? 1)
		)

		// 2. 层级变换（应用所有祖先的变换）
		let current = parent
		while (current && isShape(current)) {
			const parentTransform = Mat.compose(
				Mat.translate(current.x, current.y),
				Mat.rotate(current.rotation ?? 0)
			)
			transform = Mat.multiply(parentTransform, transform)
			current = this.getShape(current.parentId)
		}

		// 3. 相机变换
		const { x: cx, y: cy, z } = this.getCamera()
		const cameraTransform = Mat.compose(Mat.translate(cx, cy), Mat.scale(z, z))

		return Mat.multiply(cameraTransform, transform)
	}

	/**
	 * 反向变换：获取本地坐标
	 */
	pageToShapePoint(shapeId: TLShapeId, pagePoint: Vec): Vec {
		const transform = this.getShapePageTransform(shapeId)
		const inverse = Mat.invert(transform)
		return Mat.applyToPoint(inverse, pagePoint)
	}
}
```

### 11.5 撤销/重做历史管理

```typescript
class HistoryManager {
	private stack: Record<TLRecord, TLRecord>[] = []
	private pointer = -1

	/**
	 * 记录当前状态快照
	 */
	record(snapshot: Record<TLShapeId, TLShape>) {
		// 从当前指针处丢弃所有前向历史
		this.stack = this.stack.slice(0, this.pointer + 1)

		// 添加新记录
		this.stack.push(snapshot)
		this.pointer++

		// 限制历史大小（防止内存爆炸）
		if (this.stack.length > MAX_HISTORY_SIZE) {
			this.stack.shift()
			this.pointer--
		}
	}

	/**
	 * 撤销：回到上一个记录
	 */
	undo() {
		if (this.pointer <= 0) return
		this.pointer--
		const snapshot = this.stack[this.pointer]
		this.applySnapshot(snapshot)
	}

	/**
	 * 重做：向前到下一个记录
	 */
	redo() {
		if (this.pointer >= this.stack.length - 1) return
		this.pointer++
		const snapshot = this.stack[this.pointer]
		this.applySnapshot(snapshot)
	}

	private applySnapshot(snapshot: Record<TLShapeId, TLShape>) {
		// 使用 diff 而不是完整替换（更高效）
		const diff = computeDiff(this.currentSnapshot, snapshot)
		this.store.applyChanges(diff)
	}
}
```

---

## 第十二部分：未来展望和建议

### 12.1 短期改进（3-6 个月）

1. **增强工具系统**
   - 引入高阶工具抽象
   - 简化状态机定义
   - 添加工具链支持

2. **改进文本编辑**
   - 升级 Tiptap 版本
   - 支持 RTL 文本
   - 改进 IME 输入法支持

3. **性能监控**
   - 添加性能追踪 API
   - 集成 Web Vitals
   - 暴露性能指标

### 12.2 中期目标（6-12 个月）

1. **渲染层优化**
   - 考虑 WebGPU 实验性支持
   - 评估虚拟化收益
   - 优化大型文档渲染

2. **协作增强**
   - 改进并发编辑冲突解决
   - 添加变更追踪 API
   - 支持版本控制集成

3. **开发者体验**
   - 改进调试工具
   - 添加 DevTools 浏览器扩展
   - 完善错误提示

### 12.3 长期愿景（1+ 年）

1. **跨平台支持**
   - React Native 移植
   - 原生应用支持
   - 移动端优化

2. **插件系统**
   - 完善插件 API
   - 添加插件市场
   - 安全沙箱隔离

3. **AI 集成**
   - 智能布局建议
   - 内容生成辅助
   - 协作助手

---

## 总结

### 关键架构特点

✅ **分离关注点**：Core Engine 无形状耦合，通过扩展点实现灵活性

✅ **精妙的信号系统**：基于 epoch 的自动依赖追踪，避免手动订阅

✅ **多层次性能优化**：从系统架构到单个 DOM 操作都经过精心设计

✅ **渐进增强**：基础引擎可扩展到完整编辑系统

### 最佳实践建议

1. **设计 ShapeUtil 时，专注于几何和交互**，UI 交给 component() 方法
2. **使用事务确保多个更新的原子性**，避免中间状态暴露
3. **充分利用 useQuickReactor，直接 DOM 操作快于 React 状态**
4. **理解坐标系统，正确使用 screen/canvas/page/local 转换**
5. **添加自定义工具时，理解 StateNode 的状态流转模式**

### 推荐学习路径

1. 阅读 `@tldraw/state` 源码，理解信号系统
2. 学习 `Editor` 类的初始化和事件循环
3. 实现一个简单的 ShapeUtil，体验渲染系统
4. 创建一个简单的工具（如 TextTool），体验 StateNode
5. 研究性能优化策略，特别是 culling 和 transform

---

**文档完成时间**: 2025-10-27
**知识库**: tldraw 官方文档 + 源码分析
