# REPL

> 项目: [[claude-code-best]]
> 文件: src/screens/REPL.tsx (6314 行)
> 状态: L3-complete

## L1 - 黑盒视角

### 职责

**交互式命令行界面**。作为用户与 Claude Code 的主要交互入口，管理用户输入、消息显示、工具权限确认、命令执行、流式响应渲染等核心 UI 功能。

核心设计：**React/Ink 组件 + 状态管理 + query 循环**，实现"用户输入 → API 调用 → 流式响应 → 工具执行 → 结果渲染"的完整交互闭环。

### 输入

**组件 Props** (`Props`):
- `commands`: Command[] — 可用的斜杠命令列表
- `debug`: boolean — 调试模式
- `initialTools`: Tool[] — 初始工具列表
- `initialMessages`: MessageType[]? — 恢复会话时的初始消息
- `pendingHookMessages`: `Promise<HookResultMessage[]>`? — SessionStart hook 消息
- `mcpClients`: MCPServerConnection[]? — MCP 客户端连接
- `systemPrompt`: string? — 自定义系统提示词
- `appendSystemPrompt`: string? — 附加系统提示词
- `onBeforeQuery`: (input, messages) => `Promise<boolean>`? — 查询前回调
- `onTurnComplete`: (messages) => void? — turn 完成回调
- `disabled`: boolean? — 禁用输入
- `mainThreadAgentDefinition`: AgentDefinition? — 主线程 agent 定义
- `thinkingConfig`: ThinkingConfig — thinking 配置

**用户输入**:
- 键盘输入（prompt、命令、快捷键）
- 图片粘贴（通过 IDE selection）
- 历史消息选择（MessageSelector）

### 输出

**UI 渲染**:
- 消息列表（Messages 组件）
- 工具权限对话框（PermissionRequest）
- 流式响应进度（Spinner、streaming text）
- 任务列表（TaskListV2）
- 命令结果（notification、transcript entry）

**状态更新**:
- `AppState` 更新（消息、工具、权限、任务等）
- 历史记录写入（addToHistory）
- 会话文件更新（transcript）

### 调用时机

1. **启动** — CLI 入口 `main.tsx` 渲染 REPL 组件
2. **会话恢复** — `/resume` 加载历史消息后重新渲染
3. **子 agent 查看** — 查看 local agent task 时进入 teammate view

### 调用方

- [[main.tsx]] — CLI 启动入口
- [[ResumeConversation]] — 会话恢复
- [[InProcessTeammateTask]] — teammate 视图

### 黑盒调用记录

**核心 API 调用**:
- `query()` → `AsyncGenerator<QueryEvent>`, 主循环 API 调用
- `getTools()` → Tool[], 获取可用工具列表
- `assembleToolPool()` → Tool[], 组装工具池（含 MCP 工具）
- `processSessionStartHooks()` → `Promise<void>`, SessionStart hook 处理
- `executeSessionEndHooks()` → `Promise<void>`, SessionEnd hook 处理

**状态管理**:
- `useAppState()` → AppState slice, 状态读取
- `useSetAppState()` → SetAppState, 状态更新
- `useAppStateStore()` → Store, Zustand store

**消息操作**:
- `createUserMessage()` → UserMessage, 创建用户消息
- `createAssistantMessage()` → AssistantMessage, 创建 AI 消息
- `setMessages()` → void, 更新消息列表
- `addToHistory()` → void, 添加历史记录

**命令处理**:
- `commands.find()` → Command?, 查找匹配命令
- `matchingCommand.load()` → `Promise<CommandModule>`, 加载命令模块
- `mod.call()` → `Promise<JSX>`, 执行命令

**权限检查**:
- `canUseTool()` → `Promise<PermissionDecision>`, 工具权限检查
- `applyPermissionUpdate()` → PermissionContext, 应用权限更新

**其他**:
- `getSystemPrompt()` → string, 获取系统提示词模板
- `buildEffectiveSystemPrompt()` → string, 构建有效系统提示词
- `getSystemContext()` → string, 获取系统上下文
- `getUserContext()` → string, 获取用户上下文
- `getMemoryFiles()` → string[], 获取 memory 文件

### REPL 组件概览

REPL 是一个 6314 行的大型 React/Ink 组件，包含以下核心子系统：

| 子系统 | 职责 | 关键组件/Hook |
|--------|------|---------------|
| **消息管理** | 消息列表渲染、滚动、选择 | Messages, MessageSelector, VirtualMessageList |
| **输入处理** | 用户输入、命令解析、快捷键 | PromptInput, useInput, GlobalKeybindingHandlers |
| **API 循环** | query 调用、流式处理、工具执行 | query(), onQuery, onQueryEvent |
| **权限交互** | 工具权限确认、对话框 | PermissionRequest, useCanUseTool |
| **命令系统** | 斜杠命令执行、immediate/queued | commands, useCommandQueue |
| **MCP 集成** | MCP 客户端、工具、命令 | useMergedClients, useMergedTools |
| **任务管理** | agent 任务列表、状态 | TaskListV2, useTasksV2 |
| **通知系统** | 各种通知、提示 | useNotifications, addNotification |
| **会话管理** | 会话恢复、保存、标题 | sessionStorage, sessionTitle |
| **IDE 集成** | VS Code/Cursor 连接 | useIDEIntegration, useIdeSelection |

### 关键设计决策

**1. React/Ink 组件架构**

REPL 使用 Ink 框架（React for CLI）：
- 组件化 UI（Messages, PromptInput, PermissionRequest 等）
- hooks 状态管理（useAppState, useCommandQueue 等）
- 键盘输入处理（useInput, keybinding handlers）

**2. query 循环模式**

用户输入触发 `query()` 循环：
- `onSubmit()` → `onQuery()` → `query()` → `for await (event)` → `onQueryEvent()`
- 流式处理 API 响应
- 异步 generator 事件驱动

**3. 状态管理分层**

三层状态管理：
- **AppState**（Zustand store）— 全局状态（消息、工具、权限等）
- **Local state**（useState）— 组件状态（输入值、screen 等）
- **Refs**（useRef）— 非渲染状态（messagesRef, streamModeRef）

**4. 并发控制**

`QueryGuard` 防止并发 query：
- `tryStart()` → atomically check + transition idle→running
- `end()` → transition running→idle
- generation number 防止 stale finally 阻塞新 query

**5. immediate vs queued 命令**

两种命令执行模式：
- **immediate**: 立即执行（即使 query 进行中），`command.immediate: true`
- **queued**: 入队等待，query 完成后执行

**6. 流式渲染**

流式响应实时渲染：
- `streamingText` — 文本增量渲染
- `streamingToolUses` — tool_use 进度
- `streamingThinking` — thinking block 显示
- `Spinner` — 加载状态指示

**7. 消息选择模式**

transcript 模式支持消息选择：
- `[` 进入 transcript 视图
- `v` 进入选择模式
- `j/k` 移动选择
- 复制、导出等操作

**8. 远程模式支持**

三种远程模式：
- `--remote` (remoteSessionConfig) — Remote Control 模式
- `claude connect` (directConnectConfig) — Direct Connect 模式
- `claude ssh` (sshSession) — SSH 模式

**9. 窗口标题管理**

动态终端标题：
- `useTerminalTitle` 更新标题
- 默认: `Claude Code: {sessionTitle}`
- 动画: 旋转字符（响应时）

**10. 性能优化**

多项性能优化：
- `messagesRef` 避免 callback dep 变化
- `streamModeRef` 防止 onSubmit 重建
- `useDeferredValue` 延迟渲染
- `useMemo` 缓存计算结果
- virtual scroll（VirtualMessageList）

## L2 - 接口视角

### 核心函数

| 函数名 | 作用 | 关键参数 | 返回类型 |
|--------|------|----------|----------|
| `REPL()` | 主组件 | `Props` | `React.ReactNode` |
| `onSubmit()` | 用户输入提交 | `input`, `helpers`, `speculationAccept?` | `Promise<void>` |
| `onQuery()` | query 触发入口 | `newMessages`, `abortController`, `shouldQuery`, `additionalAllowedTools`, `mainLoopModelParam` | `Promise<void>` |
| `onQueryImpl()` | query 执行实现 | `messagesIncludingNewMessages`, `newMessages`, `abortController`, `shouldQuery`, `additionalAllowedTools`, `mainLoopModelParam`, `effort?` | `Promise<void>` |
| `onQueryEvent()` | 流式事件处理 | `event` | `void` |
| `getToolUseContext()` | 构建工具执行上下文 | `messages`, `newMessages`, `abortController`, `mainLoopModel` | `ProcessUserInputContext` |
| `canUseTool()` | 工具权限检查 | `toolName`, `toolUseID`, `input`, `context` | `Promise<PermissionDecision>` |

### 核心类型

```typescript
// Props - REPL 组件属性
export type Props = {
  commands: Command[]
  debug: boolean
  initialTools: Tool[]
  initialMessages?: MessageType[]
  pendingHookMessages?: Promise<HookResultMessage[]>
  initialFileHistorySnapshots?: FileHistorySnapshot[]
  initialContentReplacements?: ContentReplacementRecord[]
  initialAgentName?: string
  initialAgentColor?: AgentColorName
  mcpClients?: MCPServerConnection[]
  dynamicMcpConfig?: Record<string, ScopedMcpServerConfig>
  autoConnectIdeFlag?: boolean
  strictMcpConfig?: boolean
  systemPrompt?: string
  appendSystemPrompt?: string
  onBeforeQuery?: (input: string, newMessages: MessageType[]) => Promise<boolean>
  onTurnComplete?: (messages: MessageType[]) => void | Promise<void>
  disabled?: boolean
  mainThreadAgentDefinition?: AgentDefinition
  disableSlashCommands?: boolean
  taskListId?: string
  remoteSessionConfig?: RemoteSessionConfig
  directConnectConfig?: DirectConnectConfig
  sshSession?: SSHSession
  thinkingConfig: ThinkingConfig
}

// Screen - REPL 屏幕 mode
export type Screen = 'prompt' | 'transcript'

// SpinnerMode - 流式响应模式
type SpinnerMode = 'requesting' | 'responding' | 'tool-use'

// StreamingToolUse - 流式工具调用进度
type StreamingToolUse = {
  toolName: string
  toolUseID: string
  status: 'pending' | 'running' | 'completed' | 'error'
  input?: unknown
  result?: unknown
}

// StreamingThinking - 流式 thinking block
type StreamingThinking = {
  isStreaming: boolean
  streamingEndedAt?: number
  content?: string
}

// QueryGuard - 并发控制状态机
class QueryGuard {
  private _status: 'idle' | 'dispatching' | 'running'
  private _generation: number
  
  tryStart(): number | null  // 开始 query，返回 generation 或 null（已运行）
  end(generation: number): boolean  // 结束 query，检查 generation
  reserve(): boolean  // 队列处理预留
  cancelReservation(): void  // 取消预留
  
  subscribe: () => () => void  // React external store subscribe
  getSnapshot: () => boolean  // React external store snapshot
}

// ProcessUserInputContext - 工具执行上下文（简化版）
type ProcessUserInputContext = {
  abortController: AbortController
  options: {
    commands: Command[]
    tools: Tool[]
    debug: boolean
    verbose: boolean
    mainLoopModel: string
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    mcpResources: Record<string, ServerResource[]>
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinitionsResult
    customSystemPrompt?: string
    appendSystemPrompt?: string
    refreshTools?: () => Tool[]
  }
  getAppState: () => AppState
  setAppState: SetAppState
  messages: MessageType[]
  setMessages: (messages: MessageType[]) => void
  readFileState: FileStateCache
  setToolJSX: SetToolJSXFn
  addNotification?: (notif: Notification) => void
  appendSystemMessage?: (msg: SystemMessage) => void
  setStreamMode?: (mode: SpinnerMode) => void
  // ... 更多字段
}
```

### getToolUseContext 结构

`getToolUseContext()` 构建完整的工具执行上下文：

```text
getToolUseContext(messages, newMessages, abortController, mainLoopModel) {

  // 从 store 读取最新状态（避免 closure stale）
  const s = store.getState()

  // 动态计算工具列表（MCP 工具可能刚连接）
  const computeTools = () => {
    const state = store.getState()
    const assembled = assembleToolPool(state.toolPermissionContext, state.mcp.tools)
    const merged = mergeAndFilterTools(combinedInitialTools, assembled, state.toolPermissionContext.mode)
    if (mainThreadAgentDefinition) {
      return resolveAgentTools(mainThreadAgentDefinition, merged, false, true).resolvedTools
    }
    return merged
  }

  return {
    abortController,
    options: {
      commands,
      tools: computeTools(),
      debug,
      verbose: s.verbose,
      mainLoopModel,
      thinkingConfig: s.thinkingEnabled ? thinkingConfig : { type: 'disabled' },
      mcpClients: mergeClients(initialMcpClients, s.mcp.clients),
      mcpResources: s.mcp.resources,
      isNonInteractiveSession: false,
      agentDefinitions: allowedAgentTypes ? { ...s.agentDefinitions, allowedAgentTypes } : s.agentDefinitions,
      customSystemPrompt,
      appendSystemPrompt,
      refreshTools: computeTools,
    },
    getAppState: () => store.getState(),
    setAppState,
    messages,
    setMessages,
    updateFileHistoryState,
    updateAttributionState,
    openMessageSelector,
    readFileState,
    setToolJSX,
    addNotification,
    appendSystemMessage,
    sendOSNotification,
    setStreamMode,
    onCompactProgress,
    setInProgressToolUseIDs,
    resume,
    setConversationId,
    requestPrompt,
    contentReplacementState,
    // ... 更多字段
  }
}
```

### onQueryEvent 处理流程

`onQueryEvent()` 处理 query 返回的流式事件：

```text
onQueryEvent(event) {
  handleMessageFromStream(
    event,
    newMessage => {
      // 处理不同类型消息
      if (isCompactBoundaryMessage(newMessage)) {
        // 全屏模式：保留 pre-compact 消息
        // 普通模式：清空消息列表
        setMessages(...)
        setConversationId(randomUUID())  // 强制 row remount
        proactiveModule?.setContextBlocked(false)  // 恢复 ticks
      } else if (isEphemeralToolProgress(newMessage)) {
        // 替换而非追加（Sleep/Bash 每秒 emit）
        setMessages(oldMessages => {
          const last = oldMessages.at(-1)
          if (last?.parentToolUseID === newMessage.parentToolUseID && lastData?.type === newData.type) {
            const copy = oldMessages.slice()
            copy[copy.length - 1] = newMessage  // 替换最后一条
            return copy
          }
          return [...oldMessages, newMessage]
        })
      } else {
        setMessages(oldMessages => [...oldMessages, newMessage])
      }

      // 更新 proactive 状态
      if (newMessage.type === 'assistant' && newMessage.isApiErrorMessage) {
        proactiveModule?.setContextBlocked(true)  // API 错误阻断 ticks
      } else if (newMessage.type === 'assistant') {
        proactiveModule?.setContextBlocked(false)
      }

      // slave mode relay
      if (UDS_INBOX && newMessage.type === 'assistant') {
        relayPipeMessage({ type: 'stream' | 'error', data: text })
      }
    },
    newContent => setResponseLength(length => length + newContent.length),
    setStreamMode,
    setStreamingToolUses,
    tombstonedMessage => setMessages(old => old.filter(m => m !== tombstonedMessage)),
    setStreamingThinking,
    metrics => apiMetricsRef.current.push(metrics),
    onStreamingText,
  )
}
```

### onSubmit 流程

用户输入提交后的处理流程：

```text
onSubmit(input, helpers, speculationAccept?, options?) {

  // Step 1: 重置滚动位置
  repinScroll()

  // Step 2: 恢复 proactive mode
  if (PROACTIVE || KAIROS) proactiveModule?.resumeProactive()

  // Step 3: pipe routing（远程模式）
  if (routeToSelectedPipes(input)) {
    // 显示用户消息，清空输入，return
    return
  }

  // Step 4: immediate 命令处理
  if (input.trim().startsWith('/')) {
    const commandName = extractCommandName(input)
    const matchingCommand = commands.find(cmd => cmd.name === commandName)

    if (matchingCommand?.immediate || options?.fromKeybinding) {
      // 立即执行命令（即使 query 进行中）
      executeImmediateCommand()
      return
    }
  }

  // Step 5: idle-return 检查
  if (willowMode !== 'off' && meetsIdleThreshold) {
    // 显示 dialog/hint 提示用户重新开始
  }

  // Step 6: 标准提交流程
  handlePromptSubmit(input, helpers, ...)

  // Step 7: 调用 onQuery
  onQuery([userMessage], abortController, true, [], mainLoopModel)
}
```

### REPL 生命周期

```text
1. Mount 阶段
   - 初始化状态（useState, useRef）
   - 执行 SessionStart hooks（useDeferredHookMessages）
   - 启动各种 effect hooks（IDE, MCP, swarm 等）
   - 加载命令列表（useMergedCommands）

2. User Input 阶段
   - 用户输入（PromptInput 组件）
   - 解析命令/模式（immediate vs queued）
   - handleSubmit → onSubmit

3. Query 阶段
   - QueryGuard.tryStart() 并发检查
   - getToolUseContext() 构建上下文
   - for await (event of query({ ... }))
   - onQueryEvent(event) 处理流式事件

4. Tool Execution 阶段
   - tool_use block → findToolByName
   - canUseTool() 权限检查
   - PermissionRequest dialog（如需用户确认）
   - tool.call() 执行
   - tool_result 返回

5. Response 阶段
   - 流式响应渲染（streamingText）
   - thinking block 显示（streamingThinking）
   - Spinner 状态更新

6. Cleanup 阶段
   - QueryGuard.end()
   - resetLoadingState()
   - 历史记录写入
   - onTurnComplete callback

7. Unmount 阶段
   - executeSessionEndHooks()
   - 清理订阅
   - 保存会话状态
```

## L3 - 实现视角

### 关键代码路径

**REPL 组件初始化**:

```text
export function REPL({ commands: initialCommands, debug, initialTools, ... }: Props) {

  // Step 1: 状态初始化（useState, useRef）
  const [mainThreadAgentDefinition, setMainThreadAgentDefinition] = useState(initialMainThreadAgentDefinition)
  const [screen, setScreen] = useState<Screen>('prompt')
  const [streamMode, setStreamMode] = useState<SpinnerMode>('responding')
  const [streamingToolUses, setStreamingToolUses] = useState<StreamingToolUse[]>([])
  const [streamingThinking, setStreamingThinking] = useState<StreamingThinking | null>(null)
  const [localCommands, setLocalCommands] = useState(initialCommands)

  // Step 2: AppState 读取（Zustand slices）
  const toolPermissionContext = useAppState(s => s.toolPermissionContext)
  const verbose = useAppState(s => s.verbose)
  const mcp = useAppState(s => s.mcp)
  const plugins = useAppState(s => s.plugins)
  const tasks = useAppState(s => s.tasks)
  const setAppState = useSetAppState()

  // Step 3: Ref 初始化（非渲染状态）
  const queryGuard = React.useRef(new QueryGuard()).current  // 并发控制
  const messagesRef = useRef<MessageType[]>(initialMessages ?? [])  // 消息引用
  const streamModeRef = useRef(streamMode)  // 流模式引用
  streamModeRef.current = streamMode  // 每次渲染同步

  // Step 4: 工具/命令合并
  const localTools = useMemo(() => getTools(toolPermissionContext), [toolPermissionContext])
  const mergedTools = useMergedTools(combinedInitialTools, mcp.tools, toolPermissionContext)
  const commands = useMemo(() => disableSlashCommands ? [] : mergedCommands, [disableSlashCommands, mergedCommands])

  // Step 5: Effect hooks 初始化
  useEffect(() => {
    if (!isRemoteSession) void performStartupChecks(setAppState)
  }, [setAppState, isRemoteSession])

  // Step 6: 各种 hooks 调用
  useSwarmInitialization(setAppState, initialMessages, { enabled: !isRemoteSession })
  useIdeLogging(mcp.clients)
  useIdeSelection(mcp.clients, setIDESelection)
  useMcpConnectivityStatus({ mcpClients })
  useNotifications()
  // ... 约 30 个 hooks

  // Step 7: 核心回调定义
  const getToolUseContext = useCallback(...)
  const onQueryEvent = useCallback(...)
  const onQueryImpl = useCallback(...)
  const onQuery = useCallback(...)
  const onSubmit = useCallback(...)

  // Step 8: 返回 JSX
  return (
    <KeybindingSetup>
      <GlobalKeybindingHandlers>
        <CommandKeybindingHandlers>
          <Box>
            {/* Messages, PromptInput, PermissionRequest, etc. */}
          </Box>
        </CommandKeybindingHandlers>
      </GlobalKeybindingHandlers>
    </KeybindingSetup>
  )
}
```

**QueryGuard 并发控制**:

```text
class QueryGuard {
  private _status: 'idle' | 'dispatching' | 'running' = 'idle'
  private _generation = 0
  private _changed = createSignal()  // React external store signal

  tryStart(): number | null {
    if (this._status === 'running') return null  // 并发拒绝
    this._status = 'running'
    ++this._generation
    this._notify()
    return this._generation  // 返回新 generation number
  }

  end(generation: number): boolean {
    if (this._generation !== generation) return false  // stale finally
    if (this._status !== 'running') return false
    this._status = 'idle'
    this._notify()
    return true  // 当前 generation，执行 cleanup
  }

  reserve(): boolean {
    if (this._status !== 'idle') return false
    this._status = 'dispatching'
    this._notify()
    return true
  }

  cancelReservation(): void {
    if (this._status !== 'dispatching') return
    this._status = 'idle'
    this._notify()
  }

  // React external store API
  subscribe = () => this._changed.subscribe
  getSnapshot = () => this._status !== 'idle'
}

// 使用方式
const queryGuard = useRef(new QueryGuard()).current
const isQueryActive = useSyncExternalStore(queryGuard.subscribe, queryGuard.getSnapshot)

// onQuery 中
const thisGeneration = queryGuard.tryStart()
if (thisGeneration === null) {
  // 并发检测：enqueue message，return
  return
}

try {
  // ... query 执行
} finally {
  if (queryGuard.end(thisGeneration)) {
    // cleanup（只有当前 generation 执行）
  }
}
```

**messagesRef 模式**:

```text
// 问题：setMessages 在 callback deps 中导致每次消息更新重建 callback
// 解决：使用 ref 存储 messages，callback 读取 ref 而非 closure

// Wrapper: 同步更新 ref
const setMessages = useCallback((updater: (prev: MessageType[]) => MessageType[]) => {
  const prev = messagesRef.current
  const next = typeof updater === 'function' ? updater(prev) : updater
  messagesRef.current = next  // 立即同步 ref
  _setMessages(next)  // 触发 React re-render
}, [])

// callback 中读取 ref
const onSubmit = useCallback((input: string, helpers: PromptInputHelpers) => {
  // 读取 messagesRef.current（同步，无需 dep）
  const latestMessages = messagesRef.current
  
  // 不需要 messages 在 deps 中
}, [/* 没有 messages dep */])

// getToolUseContext 同样模式
const getToolUseContext = useCallback((messages, newMessages, abortController, mainLoopModel) => {
  // messages 参数来自 caller，不是 closure
  return { messages, ... }
}, [/* 没有 messages dep */])
```

**streamModeRef 模式**:

```text
// 问题：streamMode 在流式响应中频繁切换（~10x per turn）
// 如果 streamMode 在 onSubmit deps 中，会导致 onSubmit 重建

const [streamMode, setStreamMode] = useState<SpinnerMode>('responding')
const streamModeRef = useRef(streamMode)
streamModeRef.current = streamMode  // 每次渲染同步

// onSubmit 不依赖 streamMode，而是读取 ref
const onSubmit = useCallback((input: string, helpers: PromptInputHelpers) => {
  // 内部读取 streamModeRef.current（用于 debug/telemetry）
  const currentMode = streamModeRef.current
  
  // handleSubmit 内部也用 ref
}, [])  // 空 deps，永不重建

// setStreamMode 可安全调用，不触发 onSubmit 重建
```

**onQueryImpl 实现**:

```text
async function onQueryImpl(messagesIncludingNewMessages, newMessages, abortController, shouldQuery, additionalAllowedTools, mainLoopModelParam, effort) {

  // Step 1: IDE 准备
  if (shouldQuery) {
    const freshClients = mergeClients(initialMcpClients, store.getState().mcp.clients)
    void diagnosticTracker.handleQueryStart(freshClients)
    const ideClient = getConnectedIdeClient(freshClients)
    if (ideClient) void closeOpenDiffs(ideClient)
  }

  // Step 2: session title 生成（首次用户消息）
  if (!titleDisabled && !sessionTitle && !haikuTitleAttemptedRef.current) {
    const firstUserMessage = newMessages.find(m => m.type === 'user' && !m.isMeta)
    const text = getContentText(firstUserMessage?.message?.content)
    if (text && !isSynthetic(text)) {
      haikuTitleAttemptedRef.current = true
      void generateSessionTitle(text, signal).then(title => {
        if (title) setHaikuTitle(title)
      })
    }
  }

  // Step 3: 更新 allowedTools（skill-scoped）
  store.setState(prev => ({
    ...prev,
    toolPermissionContext: {
      ...prev.toolPermissionContext,
      alwaysAllowRules: {
        ...prev.toolPermissionContext.alwaysAllowRules,
        command: additionalAllowedTools,
      }
    }
  }))

  // Step 4: 构建 systemPrompt
  const systemPrompt = await buildEffectiveSystemPrompt({
    customSystemPrompt,
    appendSystemPrompt,
    mcpClients: freshClients,
    tools: computeTools(),
    thinkingConfig,
    ...options
  })

  // Step 5: 构建 userContext 和 systemContext
  const userContext = await getUserContext(getCwd(), { customSystemPrompt, appendSystemPrompt, ... })
  const systemContext = await getSystemContext(getCwd(), { mcpClients: freshClients, ... })

  // Step 6: 构建 toolUseContext
  const toolUseContext = getToolUseContext(messagesIncludingNewMessages, newMessages, abortController, mainLoopModelParam)

  // Step 7: 执行 query 循环
  queryCheckpoint('query_start')
  for await (const event of query({
    messages: messagesIncludingNewMessages,
    systemPrompt,
    userContext,
    systemContext,
    canUseTool,
    toolUseContext,
    querySource: getQuerySourceForREPL(),
  })) {
    onQueryEvent(event)  // 处理流式事件
  }

  // Step 8: API metrics 记录（ant-only）
  if (USER_TYPE === 'ant' && apiMetricsRef.current.length > 0) {
    const entries = apiMetricsRef.current
    const ttfts = entries.map(e => e.ttftMs)
    const otpsValues = entries.map(e => computeOTPS(e))
    setMessages(prev => [
      ...prev,
      createApiMetricsMessage({
        ttftMs: median(ttfts),
        otps: median(otpsValues),
        hookDurationMs: getTurnHookDurationMs(),
        toolDurationMs: getTurnToolDurationMs(),
        turnDurationMs: Date.now() - loadingStartTimeRef.current,
      })
    ])
  }

  // Step 9: 清理
  resetLoadingState()
  logQueryProfileReport()
  await onTurnComplete?.(messagesRef.current)
}
```

**immediate 命令执行**:

```text
onSubmit(input, helpers, speculationAccept?, options?) {
  
  // 检测命令
  if (input.trim().startsWith('/')) {
    const trimmedInput = expandPastedTextRefs(input, pastedContents).trim()
    const commandName = extractCommandName(trimmedInput)
    const matchingCommand = commands.find(cmd => isCommandEnabled(cmd) && cmd.name === commandName)

    // 判断 immediate
    const shouldTreatAsImmediate = queryGuard.isActive && (matchingCommand?.immediate || options?.fromKeybinding)

    if (matchingCommand && shouldTreatAsImmediate && matchingCommand.type === 'local-jsx') {
      // 立即执行（即使 query 进行中）
      const executeImmediateCommand = async () => {
        const onDone = (result, doneOptions) => {
          // 显示 notification
          addNotification({ key: `immediate-${matchingCommand.name}`, text: result, priority: 'immediate' })
          // 恢复 stashed prompt
          if (stashedPrompt !== undefined) {
            setInputValue(stashedPrompt.text)
            helpers.setCursorOffset(stashedPrompt.cursorOffset)
          }
        }

        const context = getToolUseContext(messagesRef.current, [], createAbortController(), mainLoopModel)
        const mod = await matchingCommand.load()
        const jsx = await mod.call(onDone, context, commandArgs)

        if (jsx && !doneWasCalled) {
          setToolJSX({ jsx, shouldHidePromptInput: false, isLocalJSXCommand: true })
        }
      }
      
      void executeImmediateCommand()
      return  // 不加入 history，不入队
    }
  }

  // 非 immediate：走标准提交流程
  handlePromptSubmit(input, helpers, ...)
}
```

**idle-return 检查**:

```text
onSubmit(input, helpers, ...) {
  
  // 检查 willow mode
  const willowMode = getFeatureValue_CACHED_MAY_BE_STALE('tengu_willow_mode', 'off')
  const idleThresholdMin = Number(process.env.CLAUDE_CODE_IDLE_THRESHOLD_MINUTES ?? 75)
  const tokenThreshold = Number(process.env.CLAUDE_CODE_IDLE_TOKEN_THRESHOLD ?? 100_000)

  if (
    willowMode !== 'off' &&
    !getGlobalConfig().idleReturnDismissed &&
    !skipIdleCheckRef.current &&
    !speculationAccept &&
    !input.trim().startsWith('/') &&
    meetsIdleThreshold
  ) {
    if (willowMode === 'dialog') {
      // 显示 IdleReturnDialog（blocking）
      setIdleReturnDialog({ threshold: idleThresholdMin, tokens: tokenThreshold })
      return
    } else if (willowMode === 'hint') {
      // 显示 notification hint
      addNotification({ key: 'idle-return-hint', text: '...', priority: 'normal' })
      idleHintShownRef.current = 'hint'
    }
  }

  // 用户可跳过检查
  skipIdleCheckRef.current = true  // 一次跳过后不再检查
}
```

### 边界条件

**1. Concurrent Query 检测**

当用户快速提交多个 prompt：
- `tryStart()` 返回 null → 检测并发
- 将 user message enqueue 到队列
- 等待当前 query 完成，队列处理器自动执行

设计原因：防止多个 API call 同时运行，保证顺序。

**2. Stale Finally Block**

用户取消 query 后立即提交新 query：
- 第一个 query 的 `finally` block 执行
- `end(generation)` 检查 generation number
- 如果 generation mismatch → 返回 false，跳过 cleanup
- 第二个 query 的 cleanup 执行

设计原因：防止 stale finally 阻塞新 query。

**3. Ephemeral Progress 替换**

Sleep/Bash 工具每秒 emit progress：
- 如果 append → messages 数组爆炸（13k+）
- 替换逻辑：检查 parentToolUseID 和 type
- 如果匹配 → 替换最后一条而非 append
- transcript 大小稳定

设计原因：O(n) 渲染性能，120MB transcript 问题。

**4. Compact Boundary 处理**

两种模式：
- **普通模式**：`setMessages(() => [newMessage])` 清空列表
- **全屏模式**：保留 pre-compact 消息作为 scrollback

`conversationId` bump 强制 row remount，防止 stale memoized rows。

**5. API Error 阻断 Proactive**

API 错误（auth, rate limit）时：
- `setContextBlocked(true)` 阻断 proactive ticks
- 防止 tick → error → tick 无限循环
- compact 成功或正常响应后恢复

**6. Skill-scoped AllowedTools**

slash command 传入 `additionalAllowedTools`：
- 写入 `toolPermissionContext.alwaysAllowRules.command`
- 下一 turn 自动清空（传入 []）

设计原因：skill frontmatter 限制工具，fork agent 也需继承。

**7. messagesRef 同步**

`setMessages` wrapper 确保 ref 同步：
- `messagesRef.current = next` 先执行
- `_setMessages(next)` 后执行
- callback 读取 ref 获得最新值

设计原因：避免 closure stale，React scheduler 延迟。

**8. Session Title 生成时机**

首次用户消息生成 title：
- 排除 synthetic 消息（command output, skill expansion）
- `haikuTitleAttemptedRef` 防止重复尝试
- 失败后重置 ref，允许下次尝试

**9. IDE Diff 关闭**

新 turn 开始时：
- `closeOpenDiffs(ideClient)` 关闭 VS Code diff panel
- 防止 stale diff 残留

**10. Remote Mode Pipe Routing**

`routeToSelectedPipes(input)` 检测：
- 如果路由成功 → 直接发送，不入队列
- 显示 user message，清空输入
- return early

### 性能考量

**1. messagesRef 模式**

测量：避免 callback 重建，减少 20-56ms GC pause

原理：
- setMessages 触发 re-render → callback deps 检查
- messages 在 deps 中 → callback 重建
- callback 重建 → 所有子组件 props 变化 → cascade re-render
- messagesRef 读取 ref → deps 不变 → callback stable

**2. streamModeRef 模式**

测量：streamMode ~10x/turn 切换，避免 10x callback 重建

原理：
- streamMode 在 deps 中 → 每次 flip 重建 onSubmit
- onSubmit 重建 → PromptInput props 变化
- ref 读取 → deps 空 → onSubmit stable

**3. computeTools 动态计算**

MCP server 异步连接：
- render 时 closure 捕获的 tools 可能 stale
- `computeTools()` 从 `store.getState()` 读取最新
- `getToolUseContext` 调用时动态计算

设计原因：MCP server 可在 render 后连接，tools 需要最新。

**4. Virtual Scroll**

`VirtualMessageList` 组件：
- 只渲染可视区域 rows
- O(visible) 而非 O(total) 渲染
- 支持数千条消息无卡顿

环境变量：`CLAUDE_CODE_DISABLE_VIRTUAL_SCROLL=1` 禁用

**5. Ephemeral Progress 替换**

测量：13k+ → 稳定数量

原理：
- append → messages.length 无限增长
- normalizeMessages, applyGrouping O(n) per render
- 替换 → length 稳定 → O(1) per render

**6. useState vs useAppState**

分层设计：
- **AppState**（Zustand）：全局共享状态，跨组件
- **useState**：组件私有状态，render 依赖
- **useRef**：非渲染状态，callback 读取

减少 AppState 订阅，优化 re-render 范围。

**7. useCallback deps 优化**

关键 callbacks deps 空或最小：
- `onSubmit` deps: []（使用 ref）
- `getToolUseContext` deps: 无 messages
- `onQueryEvent` deps: 只 set 函数

减少 cascade re-render。

**8. useDeferredValue**

`useDeferredHookMessages`：
- SessionStart hooks 消息延迟注入
- 不阻塞首屏渲染
- 用户立即看到 UI

**9. useMemo 工具缓存**

`localTools` memo：
- `getTools()` 计算开销大
- `toolPermissionContext` 变化才重新计算
- proactive/isBriefOnly 作为 dep

**10. queryCheckpoint**

query profiling：
- `queryCheckpoint('query_start')`, `queryCheckpoint('query_end')`
- 记录各阶段耗时
- `logQueryProfileReport()` 输出分析

性能分析工具，不影响生产性能。

## Review 历史

### 2026-04-19 - L3 Review

#### messagesRef 模式验证
源码 setMessages wrapper 正确（Line 1453-1456）：
- `const prev = messagesRef.current` ✅
- `const next = typeof action === 'function' ? action(messagesRef.current) : action` ✅
- `messagesRef.current = next` 先同步 ref ✅
- 注释确认 "previously cost 20-56ms per prompt; the 56ms case was a GC pause" ✅ (Line 3503-3505)

#### QueryGuard 状态机验证
源码 QueryGuard.ts 完整实现：
- `_status: 'idle' | 'dispatching' | 'running'` ✅ (Line 30)
- `_generation: number` ✅ (Line 31)
- `_changed = createSignal()` React external store ✅ (Line 32)
- `tryStart()` atomically check + transition ✅
- `end(generation)` stale finally protection ✅

#### Ephemeral Progress 替换验证
源码 onQueryEvent 逻辑正确（Line 3069-3094）：
- `isEphemeralToolProgress()` 检查 ✅
- 注释 "Sleep/Bash emit a tick per second" ✅
- 替换而非 append 逻辑 ✅

#### streamModeRef 模式验证
源码模式正确：
- `streamModeRef.current = streamMode` 每次渲染同步 ✅ (Line 1087)
- callback 读取 ref 而非 state ✅

#### Concurrent Query 检测验证
源码 onQuery 中：
- `const thisGeneration = queryGuard.tryStart()` ✅ (Line 3465)
- `if (thisGeneration === null)` → enqueue message ✅ (Line 3466-3483)
- `finally { if (queryGuard.end(thisGeneration)) ... }` ✅

#### idle-return 检查验证
源码 onSubmit 中：
- `willowMode !== 'off'` 检查 ✅
- `idleThresholdMin` 默认 75 分钟 ✅
- `dialog` vs `hint` mode ✅

#### immediate 命令验证
源码 onSubmit 中：
- `queryGuard.isActive && (matchingCommand?.immediate || options?.fromKeybinding)` ✅
- `executeImmediateCommand()` async 执行 ✅
- `return` early，不入队 ✅

#### 核心函数表格验证
源码函数签名验证：
- `getToolUseContext` 签名正确 (Line 2785-2921) ✅
- `onQueryEvent` 签名正确 (Line 3039-3165) ✅
- `onQueryImpl` 签名正确 (Line 3167-3439) ✅

#### QueryGuard 状态机验证
完整状态转换正确：
- 三状态：`idle` | `dispatching` | `running` ✅ (Line 30)
- `tryStart()` → 返回 generation 或 null ✅ (Line 61-67)
- `end(generation)` → 返回 boolean ✅ (Line 74-80)
- `reserve()` → dispatching ✅ (Line 38-43)
- `cancelReservation()` → idle ✅ (Line 49-53)

#### canUseTool 函数验证
源码 useCanUseTool.tsx：
- 返回类型 `CanUseToolFn` ✅ (Line 44-53)
- 参数：`tool`, `input`, `toolUseContext`, `assistantMessage`, `toolUseID`, `forceDecision?` ✅

#### getToolUseContext 返回结构验证
源码字段正确：
- `abortController`, `options`, `getAppState`, `setAppState` ✅
- `messages`, `setMessages`, `readFileState` ✅
- `setToolJSX`, `addNotification`, `setStreamMode` ✅
- `computeTools()` 动态计算工具列表 ✅

#### Props 类型验证
源码 Props 定义完整（Line 757-801）：
- 所有字段匹配笔记 ✅
- `thinkingConfig: ThinkingConfig` 必需字段 ✅ (Line 800)

#### onQueryEvent 流式处理验证
源码处理流程正确：
- `handleMessageFromStream` 调用 ✅ (Line 3041)
- compact boundary 消息处理 ✅ (Line 3044-3068)
- ephemeral progress 替换逻辑 ✅ (Line 3069-3094)
- API error 阻断 proactive ✅ (Line 3099-3107)

#### Props 结构验证
源码 Props 类型定义正确（Line 757-801）：
- `commands`, `debug`, `initialTools` ✅
- `initialMessages`, `pendingHookMessages` ✅
- `mcpClients`, `dynamicMcpConfig` ✅
- `systemPrompt`, `appendSystemPrompt` ✅
- `onBeforeQuery`, `onTurnComplete` ✅
- `thinkingConfig: ThinkingConfig` ✅ (Line 800)

#### QueryGuard 状态机验证
源码 QueryGuard.ts 状态机正确：
- 三状态：`idle` | `dispatching` | `running` ✅ (Line 30)
- `tryStart()` → 返回 generation number 或 null ✅ (Line 61-67)
- `end(generation)` → 检查是否当前 generation ✅ (Line 74-80)
- `reserve()`, `cancelReservation()` 用于队列处理 ✅

#### REPL 组件导入验证
- `import { QueryGuard } from '../utils/QueryGuard.js'` ✅ (Line 71)
- `const queryGuard = React.useRef(new QueryGuard()).current` ✅ (Line 1140)

#### 流式状态验证
- `useState<StreamingToolUse[]>` ✅ (Line 1088)
- `useState<StreamingThinking | null>` ✅ (Line 1089)
- `useState<SpinnerMode>('responding')` ✅ (Line 1077)

#### query 循环验证
源码调用流程正确：
- `for await (const event of query({ ... }))` ✅ (Line 3343)
- `onQueryEvent(event)` 处理每个事件 ✅ (Line 3352)
- `queryGuard.tryStart()` 并发检查 ✅ (Line 3465)

## 疑问与待查

- [ ] `onQueryEvent()` 如何处理不同类型的 query event？
- [ ] `getToolUseContext()` 如何构建工具执行上下文？
- [ ] `QueryGuard` 的完整实现细节？
- [ ] 流式响应的渲染机制？

## 相关模块

- [[query]]
- [[QueryEngine]]
- [[Tool]]
- [[permission]]
- [[message]]
- [[compact]]