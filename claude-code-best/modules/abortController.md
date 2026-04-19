# abortController

> 项目: [[claude-code-best]]
> 文件: src/utils/abortController.ts, src/hooks/useCancelRequest.ts, src/screens/REPL.tsx, src/query.ts
> 状态: L1-in-progress

## L1 - 黑盒视角

### 职责

**中断控制机制**。提供创建 AbortController、父子级联中断、以及在整个 query 流程中检查和响应中断的能力。支持用户 Ctrl+C 取消、后台切换等多种中断场景。

### 输入

**工具函数参数**:
- `createAbortController`: `maxListeners` (可选，默认 50)
- `createChildAbortController`: `parent: AbortController`, `maxListeners` (可选)

**触发中断**:
- `abortController.abort(reason)` — reason 可以是 'user-cancel', 'interrupt', 'background'

### 输出

**工具函数返回**:
- `createAbortController`: `AbortController` 对象
- `createChildAbortController`: 子 `AbortController` 对象

**query 返回值**:
- `{ reason: 'aborted_streaming' }` — API streaming 期间中断
- `{ reason: 'aborted_tools' }` — 工具执行期间中断

### 调用时机

1. **REPL 启动 query** → 创建 AbortController，传给 toolUseContext
2. **用户 Ctrl+C/Escape** → 触发 abort('user-cancel')
3. **query 每个阶段** → 检查 `signal.aborted`
4. **后台切换 Ctrl+B** → abort('background')
5. **子 Agent fork** → createChildAbortController(parent)

### 调用方

- [[REPL]] — 创建和管理 abortController 状态
- [[query]] — 检查 abort 状态，返回中断原因
- [[hook]] — useCancelRequest 处理用户取消
- [[StreamingToolExecutor]] — 工具执行检查 abort
- [[forkedAgent]] — 子 Agent 创建 child abortController

### 黑盒调用记录

**AbortController API（Web 内置）**:
- `controller.abort(reason)` → 设置 signal.aborted=true, signal.reason=reason
- `signal.aborted` → 返回 boolean，检查是否已中断
- `signal.reason` → 返回中断原因（'user-cancel', 'interrupt', 'background' 等）

**工具函数**:
- `createAbortController()` → 返回 AbortController，设置 maxListeners
- `createChildAbortController(parent)` → 返回子 AbortController，父中断时自动传播到子

**query 检查点**:
- API streaming 前 → 检查是否可添加 tool_use block
- API streaming 结束 → yield 中断消息，return aborted_streaming
- 工具执行后 → yield 中断消息，return aborted_tools

### 关键设计决策

**1. 父子级联传播**

```
父 abort → 子自动 abort（单向）
子 abort → 不影响父
```

设计原因：父任务取消时，所有子任务应取消；但子任务单独取消不应影响父。

**2. WeakRef 内存安全**

```typescript
const weakChild = new WeakRef(child)
const weakParent = new WeakRef(parent)
```

父只持有子的 WeakRef，子被丢弃后可 GC，不会因父长期存活而泄漏。

**3. reason 区分中断场景**

| reason        | 场景               | UI行为                    |
| ------------- | ---------------- | ----------------------- |
| 'user-cancel' | 用户主动 Ctrl+C      | 显示中断消息                  |
| 'interrupt'   | submit-interrupt | 不显示（后续有 queued message） |
| 'background'  | Ctrl+B 后台切换      | 不显示                     |

**4. 多层检查**

```
API streaming → 添加 tool_use block 时检查
API streaming 结束 → 处理 abort
工具执行 → 每个阶段检查
compact → tryReactiveCompact 参数
```

**5. maxListeners 防止警告**

默认 50 个 listener，防止多个组件监听 abort 时触发 MaxListenersExceededWarning。

**6. Fast path 优化**

```typescript
if (parent.signal.aborted) {
  child.abort(parent.signal.reason)  // 立即设置，不添加 listener
  return child
}
```

父已中断时，直接设置子，跳过 listener 注册。

**7. reason 不可覆盖**

```typescript
child.abort('child first')  // child reason = 'child first'
parent.abort('parent later')  // parent reason = 'parent later'
// child.reason 仍为 'child first'，不覆盖
```

子已中断后，父的中断传播不改变子的 reason。

## L2 - 接口视角

### 核心函数

| 函数名 | 作用 | 关键参数 | 返回类型 |
|--------|------|----------|----------|
| `createAbortController` | 创建 AbortController | `maxListeners?: number` | `AbortController` |
| `createChildAbortController` | 创建子 AbortController | `parent: AbortController`, `maxListeners?: number` | `AbortController` |
| `abortController.abort` | 触发中断 | `reason?: string` | void |
| `signal.aborted` | 检查中断状态 | 无 | `boolean` |
| `signal.reason` | 获取中断原因 | 无 | `string` |

### 核心类型

```typescript
// AbortController（Web API 内置）
interface AbortController {
  signal: AbortSignal
  abort(reason?: any): void
}

interface AbortSignal {
  aborted: boolean
  reason: any
  addEventListener(type: 'abort', listener: Function, options?: { once?: boolean }): void
  removeEventListener(type: 'abort', listener: Function): void
}

// Claude Code 工具函数
export function createAbortController(maxListeners?: number): AbortController
export function createChildAbortController(parent: AbortController, maxListeners?: number): AbortController

// Query 返回值
type Terminal = 
  | { reason: 'aborted_streaming' }
  | { reason: 'aborted_tools' }
  | { reason: 'completed' }
  | ...
```

### AbortController 在 toolUseContext 中的位置

```typescript
type ToolUseContext = {
  abortController: AbortController  // ← 核心
  options: {
    tools: Tool[]
    mainLoopModel: string
    ...
  }
  getAppState: () => AppState
  setAppState: (updater) => void
  ...
}
```

### AbortController 生命周期

```
┌─────────────────────────────────────────────────────────────┐
│ AbortController 生命周期                                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. REPL 创建                                                │
│     const [abortController, setAbortController] = useState  │
│                                                             │
│  2. 传递到 toolUseContext                                    │
│     getToolUseContext(messages, [], abortController)        │
│                                                             │
│  3. 传递到 query                                             │
│     query({ toolUseContext, ... })                          │
│                                                             │
│  4. API 调用传递 signal                                      │
│     callModel({ signal: abortController.signal })           │
│                                                             │
│  5. 用户取消触发 abort                                        │
│     useCancelRequest → handleCancel → abortController.abort │
│                                                             │
│  6. query 检查并返回                                         │
│     if (signal.aborted) → return { reason: 'aborted_...' }  │
│                                                             │
│  7. 清理                                                     │
│     setAbortController(null)                                │
│     resetLoadingState()                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## L3 - 实现视角

### 关键代码路径

**createAbortController 实现**:

```text
createAbortController(maxListeners = 50) {
  const controller = new AbortController()
  setMaxListeners(maxListeners, controller.signal)
  return controller
}
```

**createChildAbortController 实现**:

```text
createChildAbortController(parent, maxListeners) {
  const child = createAbortController(maxListeners)
  
  // Fast path: 父已中断
  if (parent.signal.aborted) {
    child.abort(parent.signal.reason)
    return child
  }
  
  // WeakRef 防止内存泄漏
  const weakChild = new WeakRef(child)
  const weakParent = new WeakRef(parent)
  
  // 父中断时传播到子
  const handler = propagateAbort.bind(weakParent, weakChild)
  parent.signal.addEventListener('abort', handler, { once: true })
  
  // 子中断时清理父的 listener
  child.signal.addEventListener('abort', removeAbortHandler.bind(...), { once: true })
  
  return child
}

// 模块级函数，避免闭包分配
propagateAbort(weakParent, weakChild) {
  const parent = weakParent.deref()
  weakChild.deref()?.abort(parent?.signal.reason)
}

removeAbortHandler(weakParent, weakHandler) {
  const parent = weakParent.deref()
  const handler = weakHandler.deref()
  if (parent && handler) {
    parent.signal.removeEventListener('abort', handler)
  }
}
```

**REPL 创建和传递**:

```text
// REPL.tsx
const [abortController, setAbortController] = useState<AbortController | null>(null)
const abortControllerRef = useRef<AbortController | null>(null)
abortControllerRef.current = abortController

const getToolUseContext = useCallback((messages, newMessages, abortController, mainLoopModel) => {
  return {
    abortController,  // ← 传入
    options: { ... },
    ...
  }
}, [...])

// onQueryImpl
const toolUseContext = getToolUseContext(messages, [], abortController, mainLoopModel)

for await (const event of query({ toolUseContext, ... })) {
  onQueryEvent(event)
}
```

**用户取消触发**:

```text
// useCancelRequest.ts
const handleCancel = useCallback(() => {
  if (abortSignal !== undefined && !abortSignal.aborted) {
    logEvent('tengu_cancel', cancelProps)
    setToolUseConfirmQueue(() => [])
    onCancel()  // → cancelRequest()
    return
  }
}, [abortSignal, ...])

// REPL.tsx cancelRequest
const cancelRequest = useCallback(() => {
  abortController?.abort('user-cancel')
  setAbortController(null)
  void mrOnTurnComplete(messagesRef.current, true)
}, [abortController, ...])
```

**query.ts 检查点**:

```text
// API 调用传递 signal
signal: toolUseContext.abortController.signal

// 添加 tool_use block 时检查
if (!toolUseContext.abortController.signal.aborted) {
  streamingToolExecutor.addTool(toolBlock, assistantMessage)
}

// API streaming 结束检查
if (toolUseContext.abortController.signal.aborted) {
  if (streamingToolExecutor) {
    for await (const update of streamingToolExecutor.getRemainingResults()) {
      if (update.message) yield update.message
    }
  }
  
  if (toolUseContext.abortController.signal.reason !== 'interrupt') {
    yield createUserInterruptionMessage({ toolUse: false })
  }
  return { reason: 'aborted_streaming' }
}

// 工具执行后检查
if (toolUseContext.abortController.signal.aborted) {
  if (toolUseContext.abortController.signal.reason !== 'interrupt') {
    yield createUserInterruptionMessage({ toolUse: true })
  }
  return { reason: 'aborted_tools' }
}
```

### 边界条件

**1. 父已中断时创建子**:

```typescript
parent.abort('pre-abort')
const child = createChildAbortController(parent)
// child 立即被中断，不注册 listener
```

**2. 子先中断，父后中断**:

```typescript
child.abort('child first')
parent.abort('parent later')
// child.reason = 'child first'（不覆盖）
// parent.reason = 'parent later'
```

**3. 多个子独立**:

```typescript
const child1 = createChildAbortController(parent)
const child2 = createChildAbortController(parent)
child1.abort()
// child2 不受影响，parent 不受影响
```

**4. 祖孙三代级联**:

```typescript
const grandparent = createAbortController()
const parent = createChildAbortController(grandparent)
const child = createChildAbortController(parent)
grandparent.abort()
// parent、child 都被中断
```

**5. reason 为 'interrupt' 时不显示中断消息**:

```typescript
if (signal.reason !== 'interrupt') {
  yield createUserInterruptionMessage()
}
// submit-interrupt 场景，后续有 queued message 提供上下文
```

**6. abortController 为 null 时**:

```typescript
// REPL 状态: abortController = null
// canCancelRunningTask = false
// 用户按键不触发 cancel
```

### 性能考量

**1. WeakRef 防止内存泄漏**:

```
问题: 父长期存活，注册了子的 abort listener
      子被丢弃后，listener 仍存在 → 内存泄漏

解决: WeakRef 持有子和父
      子被 GC 后，WeakRef.deref() 返回 undefined
      listener 不阻塞 GC
```

**2. 模块级函数避免闭包分配**:

```typescript
// 每次调用 createChildAbortController 不创建新函数
// propagateAbort 和 removeAbortHandler 是模块级，复用
const handler = propagateAbort.bind(weakParent, weakChild)
// 只创建 bound 函数，不创建 closure
```

**3. once: true 自动清理**:

```typescript
parent.signal.addEventListener('abort', handler, { once: true })
// 触发一次后自动移除，不需手动清理
```

**4. Fast path 跳过 listener 注册**:

```typescript
if (parent.signal.aborted) {
  child.abort(parent.signal.reason)
  return child  // 不注册 listener
}
```

**5. maxListeners 防止警告**:

```
默认 10 个 listener 超过会警告
Claude Code 多组件监听同一 signal
设置 maxListeners=50 避免警告
```

## Review 历史

## 疑问与待查

- [ ] AbortController 与 Hook 的 StopUserCancelled 事件的关系？
- [ ] 子 Agent fork 时如何创建 child abortController？
- [ ] StreamingToolExecutor 如何处理 abort？

## 相关模块

- [[REPL]]
- [[query]]
- [[hook]]
- [[StreamingToolExecutor]]
- [[forkedAgent]]