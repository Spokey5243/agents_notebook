# query

> 项目: [[claude-code-best]]
> 文件: src/query.ts
> 状态: L2-complete

## L1 - 黑盒视角

### 职责

**Agent 主循环入口**。处理用户输入触发的一个完整 turn（包含多轮 tool call），通过 `yield` 流式返回事件给调用者，最终返回结束原因。

核心设计：**异步生成器 + yield 流式输出 + while(true) 循环**，实现"一个入口、流式输出、状态驱动"的 agent 运行模式。

### 输入

**QueryParams**:
- `messages`: Message[] — 当前对话历史
- `systemPrompt`: SystemPrompt — 系统提示词
- `userContext`: { [k: string]: string } — 用户上下文信息
- `systemContext`: { [k: string]: string } — 系统上下文信息
- `canUseTool`: CanUseToolFn — 工具权限检查函数
- `toolUseContext`: ToolUseContext — 工具执行上下文（核心对象，包含 tools、options、abortController 等）
- `fallbackModel`: string? — 备用模型
- `querySource`: QuerySource — 调用来源标识
- `maxOutputTokensOverride`: number? — 输出 token 上限覆盖
- `maxTurns`: number? — 最大轮次限制
- `skipCacheWrite`: boolean? — 是否跳过缓存写入
- `taskBudget`: { total: number }? — API 任务预算
- `deps`: QueryDeps? — 依赖注入（用于测试/替换核心函数）

### 输出

**流式输出**（通过 yield）:
- `StreamEvent` — 流式事件
- `RequestStartEvent` — 请求开始事件
- `Message` — 消息（AssistantMessage, UserMessage 等）
- `TombstoneMessage` — 墓碑消息（标记删除）
- `ToolUseSummaryMessage` — 工具调用摘要

**最终返回**（Terminal）:
- `reason`: 结束原因 — `'completed'` | `'aborted_streaming'` | `'aborted_tools'` | `'blocking_limit'` | `'max_turns'` | `'prompt_too_long'` | `'model_error'` | `'stop_hook_prevented'` | `'hook_stopped'` | `'image_error'`

### 调用时机

1. **用户输入触发** — REPL 主界面收到用户输入后调用
2. **子 agent 调用** — AgentTool 创建子 agent 时调用（fork 场景）
3. **恢复场景** — `/resume` 或 session_memory 恢复时调用
4. **compact 场景** — 压缩 agent 调用时使用 fork context

### 调用方

- [[REPL]] — 主界面屏幕，处理用户交互
- [[QueryEngine]] — 高层封装，管理对话状态
- [[AgentTool]] — 创建子 agent 执行任务

### 黑盒调用记录

阅读过程中遇到的"外部调用"，暂不深入：

**上下文管理**:
- `deps.autocompact()` → 返回 `{ compactionResult, consecutiveFailures }`，压缩超限上下文
- `deps.microcompact()` → 返回 `{ messages, compactionInfo }`，微压缩 tool result
- `applyToolResultBudget()` → 返回 `messages`，应用工具结果大小预算
- `getMessagesAfterCompactBoundary()` → 返回 `messages`，获取未压缩消息
- `contextCollapse.applyCollapsesIfNeeded()` → 返回 `{ messages }`，应用上下文折叠

**API 调用**:
- `deps.callModel()` → 返回流式消息，调用 LLM API
- `prependUserContext()` → 返回 `messages`，添加用户上下文
- `appendSystemContext()` → 返回 `systemPrompt`，添加系统上下文

**工具执行**:
- `runTools()` → 返回异步迭代器，执行工具调用
- `StreamingToolExecutor` → 流式工具执行器类
- `findToolByName()` → 返回 Tool，按名查找工具

**Hook 系统**:
- `handleStopHooks()` → 返回 `{ preventContinuation, blockingErrors }`，处理停止 hook
- `executeStopFailureHooks()` → 执行失败 hook
- `executePostSamplingHooks()` → 执行采样后 hook

**状态追踪**:
- `notifyCommandLifecycle()` → 通知命令生命周期（started/completed）
- `queryCheckpoint()` → 性能检查点标记
- `logEvent()` → 记录事件
- `createTrace()` / `endTrace()` → Langfuse trace 管理

**权限系统**:
- `canUseTool` → 函数，检查工具是否可用
- `toolUseContext.abortController.signal` → 中断信号

**其他**:
- `getRuntimeMainLoopModel()` → 返回模型名，获取当前模型
- `calculateTokenWarningState()` → 返回 `{ isAtBlockingLimit }`，计算 token 状态
- `tokenCountWithEstimation()` → 返回 token 数，估算消息 token
- `createBudgetTracker()` → 返回预算追踪器
- `checkTokenBudget()` → 返回预算决策，检查 token 预算

### 关键设计决策

**1. 异步生成器架构**

使用 `async function*` + `yield` 实现流式输出，调用者通过 `for await (const event of query(...))` 消费事件。这种设计：
- 调用者可以实时处理事件（显示 UI、处理工具）
- 不阻塞 agent 主循环
- 支持中断（abortController）

**2. 委托生成器 (`yield*`)**

`terminal = yield* queryLoop(...)` 将 queryLoop 的所有 yield 值直接传递给调用者，最后获取 return 值。这避免了多层嵌套。

**3. 状态驱动循环**

使用 `State` 类型承载循环状态，通过 `state = { ... }` 更新状态并 `continue` 重新循环。这避免了复杂的状态管理逻辑。

**4. 依赖注入 (`deps`)**

通过 `deps` 参数注入核心函数（callModel、autocompact、microcompact），支持测试替换和生产环境配置。

## L2 - 接口视角

### 核心函数

| 函数名                                                             | 作用                                  | 关键参数                                                                    |
| --------------------------------------------------------------- | ----------------------------------- | ----------------------------------------------------------------------- |
| `query(params)`                                                 | 主入口，导出的异步生成器                        | `params: QueryParams` — 包含 messages、systemPrompt、toolUseContext 等       |
| `queryLoop(params, consumedCommandUuids)`                       | 内部循环，实际执行逻辑                         | `params: QueryParams`, `consumedCommandUuids: string[]` — 已消费命令 UUID 列表 |
| `yieldMissingToolResultBlocks(assistantMessages, errorMessage)` | 生成缺失的 tool_result 消息                | `assistantMessages: AssistantMessage[]`, `errorMessage: string`         |
| `isWithheldMaxOutputTokens(msg)`                                | 判断是否为 withheld max_output_tokens 错误 | `msg: Message \| StreamEvent \| undefined`                              |

### 核心类型

```typescript
// QueryParams - 主入口参数类型
export type QueryParams = {
  messages: Message[]
  systemPrompt: SystemPrompt
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  canUseTool: CanUseToolFn
  toolUseContext: ToolUseContext
  fallbackModel?: string
  querySource: QuerySource
  maxOutputTokensOverride?: number
  maxTurns?: number
  skipCacheWrite?: boolean
  taskBudget?: { total: number }
  deps?: QueryDeps
}

// State - 循环状态类型（mutable）
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined
}

// query 函数签名
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent
  | RequestStartEvent
  | Message
  | TombstoneMessage
  | ToolUseSummaryMessage,
  Terminal
>

// queryLoop 函数签名
async function* queryLoop(
  params: QueryParams,
  consumedCommandUuids: string[],
): AsyncGenerator<
  | StreamEvent
  | RequestStartEvent
  | Message
  | TombstoneMessage
  | ToolUseSummaryMessage,
  Terminal
>
```

### 函数调用关系

```
query(params)
  ├── ownsTrace = !params.toolUseContext.langfuseTrace
  ├── langfuseTrace = params.toolUseContext.langfuseTrace ?? createTrace() ?? null
  └── terminal = yield* queryLoop(params, consumedCommandUuids)  ← 委托生成器
          └── finally: endTrace(langfuseTrace)
  └── notifyCommandLifecycle(uuid, 'completed')
  └── return terminal
```

### 常量

```typescript
const MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3  // max_output_tokens 恢复最大尝试次数
```

## L3 - 实现视角

（待 L2 完成后填充）

### 关键代码路径

```
query() {
  while(true) {
    ...
  }
}
```

### 边界条件

- {错误处理}

### 性能考量

- {缓存/异步}

## Review 历史

### 2026-04-19 - L1 Review

#### 输出 - Terminal 类型

~~`reason`: 结束原因 — `'completed'` | `'aborted_streaming'` | `'aborted_tools'` | `'blocking_limit'` | `'max_turns'` | `'prompt_too_long'` | `'model_error'` | `'stop_hook_prevented'` | `'hook_stopped'` | `'image_error'`~~

> ❌ **错误理解**: 假设 Terminal 有明确的类型定义，但 `src/query/transitions.ts` 显示 `Terminal = any` 是 stub 类型。
> **正确理解**: Terminal 目前是 stub 类型，实际返回值由 query 函数内部约定。从源码 grep 可见，reason 值包括：
> - `'completed'` — 正常完成
> - `'aborted_streaming'` — 流式输出中断
> - `'aborted_tools'` — 工具执行中断
> - `'blocking_limit'` — 达到阻塞限制
> - `'max_turns'` — 达到最大轮次（附带 `turnCount`）
> - `'prompt_too_long'` — prompt 过长
> - `'model_error'` — 模型错误（附带 `error`）
> - `'stop_hook_prevented'` — stop hook 阻止继续
> - `'hook_stopped'` — hook 停止
> - `'image_error'` — 图片错误

### 2026-04-19 - L2 Review

#### isWithheldMaxOutputTokens - 类型守卫

表格描述："判断是否为 withheld max_output_tokens 错误"

> ⚠️ **补充说明**: 该函数是 TypeScript **类型守卫**（Type Guard）。返回类型 `msg is AssistantMessage` 表示：当函数返回 `true` 时，TypeScript 会将 `msg` 的类型从 `Message | StreamEvent | undefined` 收窄为 `AssistantMessage`。
>
> **实现逻辑**: `msg?.type === 'assistant' && msg.apiError === 'max_output_tokens'`

---

## 疑问与待查

- [ ] `yield*` 和 `yield` 的区别？（已解答：yield* 委托生成器）
- [ ] compact 的触发阈值是多少？
- [ ] `toolUseContext` 对象的完整结构是什么？
- [ ] `deps` 的默认值 `productionDeps()` 包含什么？

## 相关模块

- [[REPL]]
- [[QueryEngine]]
- [[Tool]]
- [[compact]]
- [[hook]]
- [[permission]]