# tool

> 项目: [[claude-code-best]]
> 文件: src/Tool.ts, src/tools.ts
> 状态: L1-complete

## L1 - 黑盒视角

### 职责

**工具系统**。定义工具的接口、执行上下文、权限检查、结果处理等核心类型，以及工具的注册、组装、过滤等管理逻辑。

核心设计：**泛型接口 + Zod Schema + 权限集成**，实现"类型安全、schema 验证、权限控制"的工具管理。

### 输入

**Tool 类型参数**（泛型）:
- `Input`: Zod schema 类型，定义工具输入参数
- `Output`: 工具输出类型
- `P`: 进度数据类型（ToolProgressData 子类型）

**Tool.call 参数**:
- `args`: `z.infer<Input>` — 工具输入参数
- `context`: ToolUseContext — 工具执行上下文
- `canUseTool`: CanUseToolFn — 权限检查函数
- `parentMessage`: AssistantMessage — 父消息（包含 tool_use）
- `onProgress`: `ToolCallProgress<P>`? — 进度回调

**getAllBaseTools 参数**:
- 无参数，根据环境变量和 feature flags 返回工具列表

### 输出

**`ToolResult<T>`**:
- `data: T` — 工具输出数据
- `newMessages`: Message[]? — 新增消息（如权限提示）
- `contextModifier`: (context) => context? — 上下文修改器
- `mcpMeta`: MCP metadata? — MCP 协议元数据

**Tools**:
- `readonly Tool[]` — 工具列表

### 调用时机

1. **API 响应处理** — 收到 tool_use block 后，通过 `findToolByName()` 找到工具，调用 `call()`
2. **工具注册** — `getAllBaseTools()` 在启动时组装工具列表
3. **权限过滤** — `getTools()` 根据 permissionContext 过滤可用工具
4. **MCP 工具集成** — MCP server 连接后动态添加工具

### 调用方

- [[query]] — 处理 tool_use，调用 `runTools()`
- [[REPL]] — 初始化时组装工具列表
- [[QueryEngine]] — 管理工具状态
- [[permission]] — 权限检查调用 `checkPermissions()`
- [[MCP]] — MCP 工具集成

### 黑盒调用记录

**类型导入**:
- `z.infer<Input>` → Zod schema 推断类型
- `ToolResultBlockParam`, `ToolUseBlockParam` → 来自 `@anthropic-ai/sdk`
- `PermissionResult` → 权限检查结果

**工具函数**:
- `findToolByName()` → 返回 Tool | undefined，按名称查找工具
- `toolMatchesName()` → 返回 boolean，检查名称匹配（支持别名）
- `getAllBaseTools()` → 返回 Tools，获取所有基础工具
- `getTools()` → 返回 Tools，获取可用工具（权限过滤）
- `filterToolsByDenyRules()` → 返回 T[]，按拒绝规则过滤
- `getEmptyToolPermissionContext()` → 返回空权限上下文

**权限相关**:
- `checkPermissions()` → 返回 `Promise<PermissionResult>`，工具权限检查
- `validateInput()` → 返回 `Promise<ValidationResult>`，输入验证

**其他**:
- `isConcurrencySafe()` → 返回 boolean，判断是否并发安全
- `isReadOnly()` → 返回 boolean，判断是否只读
- `isDestructive()` → 返回 boolean?，判断是否破坏性操作
- `isEnabled()` → 返回 boolean，判断是否启用

### 工具概览

按功能分类（从 getAllBaseTools 提取）：

| 分类 | 工具 |
|------|------|
| **文件操作** | FileReadTool, FileEditTool, FileWriteTool |
| **搜索** | GlobTool, GrepTool, WebSearchTool, ToolSearchTool |
| **执行** | BashTool, PowerShellTool?, REPLTool? |
| **Agent** | AgentTool, TaskCreateTool, TaskUpdateTool, TaskGetTool, TaskListTool, TaskStopTool, TaskOutputTool |
| **规划** | EnterPlanModeTool, ExitPlanModeV2Tool, VerifyPlanExecutionTool? |
| **Web/MCP** | WebFetchTool, ListMcpResourcesTool, ReadMcpResourceTool |
| **调度** | CronCreateTool, CronDeleteTool, CronListTool |
| **其他** | SkillTool, AskUserQuestionTool, TodoWriteTool, ConfigTool, LSPTool, NotebookEditTool |
| **实验性** | MonitorTool?, RemoteTriggerTool?, SleepTool?, SnipTool?, WorkflowTool? |

### 关键设计决策

**1. 泛型接口设计**

`Tool<Input, Output, P>` 使用三个泛型参数：
- `Input`: Zod schema 类型，定义输入结构
- `Output`: 输出类型，可以是任意类型
- `P`: 进度数据类型，用于 `onProgress` 回调

**2. Zod Schema 验证**

- `inputSchema`: 定义输入参数 schema
- `outputSchema`: 定义输出 schema（可选）
- `z.infer<Input>`: 自动推断输入类型

**3. 权限集成**

- `checkPermissions()`: 工具特定的权限逻辑
- `validateInput()`: 输入验证（如路径检查）
- `isDestructive()`: 破坏性操作标记
- `isReadOnly()`: 只读操作标记

**4. 并发安全**

- `isConcurrencySafe()`: 判断是否可并发执行
- `contextModifier`: 非并发安全工具可修改上下文

**5. 工具别名**

- `aliases`: 支持工具重命名后的向后兼容
- `toolMatchesName()`: 同时检查主名称和别名

**6. 工具延迟加载**

- `shouldDefer`: 工具延迟加载标记
- `alwaysLoad`: 强制立即加载（不延迟）
- `ToolSearchTool`: 延迟加载工具的搜索工具

**7. MCP 兼容**

- `isMcp`: MCP 工具标记
- `mcpInfo`: MCP server 和工具名称信息
- `inputJSONSchema`: MCP 工具可直接使用 JSON Schema

**8. 进度回调**

- `onProgress`: 工具执行过程中通知进度
- `ToolProgress<P>`: 进度数据结构
- `ProgressMessage`: UI 显示进度消息

**9. 结果渲染**

- `mapToolResultToToolResultBlockParam()`: 将输出转为 API 格式
- `renderToolResultMessage()`: UI 渲染工具结果
- `maxResultSizeChars`: 结果大小限制

## L2 - 接口视角

### 核心函数

| 函数名 | 作用 | 关键参数 | 返回类型 |
|--------|------|----------|----------|
| `toolMatchesName()` | 检查工具名匹配（含别名） | `tool`, `name` | `boolean` |
| `findToolByName()` | 按名称查找工具 | `tools`, `name` | `Tool | undefined` |
| `getAllBaseTools()` | 获取所有基础工具 | 无 | `Tools` |
| `getTools()` | 获取可用工具（权限过滤） | `permissionContext` | `Tools` |
| `filterToolsByDenyRules()` | 按拒绝规则过滤 | `tools`, `permissionContext` | `T[]` |
| `Tool.call()` | 执行工具 | `args`, `context`, `canUseTool`, `parentMessage`, `onProgress` | `Promise<ToolResult<Output>>` |
| `Tool.checkPermissions()` | 工具权限检查 | `input`, `context` | `Promise<PermissionResult>` |
| `Tool.validateInput()` | 输入验证 | `input`, `context` | `Promise<ValidationResult>` |
| `Tool.description()` | 生成工具描述 | `input`, `options` | `Promise<string>` |
| `Tool.prompt()` | 生成工具提示词 | `options` | `Promise<string>` |

### 核心类型

```typescript
// Tool 泛型接口
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData
> = {
  name: string
  aliases?: string[]
  inputSchema: Input
  inputJSONSchema?: ToolInputJSONSchema  // MCP 工具用
  outputSchema?: z.ZodType<unknown>
  maxResultSizeChars: number
  
  // 核心方法
  call(args, context, canUseTool, parentMessage, onProgress): Promise<ToolResult<Output>>
  description(input, options): Promise<string>
  prompt(options): Promise<string>
  checkPermissions(input, context): Promise<PermissionResult>
  validateInput?(input, context): Promise<ValidationResult>
  
  // 属性判断方法
  isConcurrencySafe(input): boolean
  isEnabled(): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean | undefined
  interruptBehavior?(): 'cancel' | 'block'
  
  // 延迟加载
  shouldDefer?: boolean
  alwaysLoad?: boolean
  
  // MCP 兼容
  isMcp?: boolean
  mcpInfo?: { serverName: string; toolName: string }
  
  // 结果渲染
  mapToolResultToToolResultBlockParam(content, toolUseID): ToolResultBlockParam
  renderToolResultMessage?(content, progressMessages, options): React.ReactNode
  renderToolUseMessage(input, options): React.ReactNode
  
  // 其他
  getPath?(input): string
  userFacingName(input): string
  getActivityDescription?(input): string | null
  toAutoClassifierInput(input): unknown
}

// Tools 类型（工具集合）
export type Tools = readonly Tool[]

// ToolResult 结构
export type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext
  mcpMeta?: { _meta?: Record<string, unknown>; structuredContent?: Record<string, unknown> }
}

// ValidationResult
export type ValidationResult =
  | { result: true }
  | { result: false; message: string; errorCode: number }

// ToolCallProgress 回调类型
export type ToolCallProgress<P extends ToolProgressData = ToolProgressData> = 
  (progress: ToolProgress<P>) => void

// ToolProgress 进度数据
export type ToolProgress<P extends ToolProgressData> = {
  toolUseID: string
  data: P
}

// AnyObject（Zod schema 约束）
export type AnyObject = z.ZodType<{ [key: string]: unknown }>
```

### ToolUseContext 关键字段

```typescript
export type ToolUseContext = {
  // 配置选项
  options: {
    commands: Command[]
    debug: boolean
    mainLoopModel: string
    tools: Tools
    verbose: boolean
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    mcpResources: Record<string, ServerResource[]>
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinitionsResult
    maxBudgetUsd?: number
    customSystemPrompt?: string
    appendSystemPrompt?: string
    querySource?: QuerySource
    refreshTools?: () => Tools
  }
  
  // 核心状态
  abortController: AbortController
  messages: Message[]
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  
  // UI 回调
  setToolJSX?: SetToolJSXFn
  addNotification?: (notif: Notification) => void
  appendSystemMessage?: (msg: SystemMessage) => void
  sendOSNotification?: (opts) => void
  setStreamMode?: (mode: SpinnerMode) => void
  onCompactProgress?: (event: CompactProgressEvent) => void
  
  // 权限相关
  setInProgressToolUseIDs: (f: (prev: Set<string>) => Set<string>) => void
  setHasInterruptibleToolInProgress?: (v: boolean) => void
  requireCanUseTool?: boolean
  toolDecisions?: Map<string, { source: string; decision: 'accept' | 'reject'; timestamp: number }>
  
  // 进度回调请求
  requestPrompt?: (sourceName: string) => (request: PromptRequest) => Promise<PromptResponse>
  
  // Agent 相关
  agentId?: AgentId
  agentType?: string
  
  // 文件历史和归属
  updateFileHistoryState: (updater) => void
  updateAttributionState: (updater) => void
  
  // 其他
  fileReadingLimits?: { maxTokens?: number; maxSizeBytes?: number }
  globLimits?: { maxResults?: number }
  queryTracking?: QueryChainTracking
  langfuseTrace?: LangfuseSpan | null
  contentReplacementState?: ContentReplacementState
  renderedSystemPrompt?: SystemPrompt
}
```

### 工具生命周期

```text
1. 注册阶段
   getAllBaseTools() → 根据环境/feature 组装工具列表
   getTools(permissionContext) → 权限过滤 → API 请求的 tools 参数

2. 调用阶段
   API 返回 tool_use → findToolByName() 查找
   → validateInput() 验证输入
   → checkPermissions() 权限检查
   → canUseTool() hook/用户确认
   → call() 执行工具
   → onProgress 进度回调（可选）
   → mapToolResultToToolResultBlockParam() 转换结果
   → tool_result 发送给 API

3. 渲染阶段
   renderToolUseMessage() 渲染 tool_use 消息
   renderToolUseProgressMessage() 渲染进度（可选）
   renderToolResultMessage() 渲染 tool_result（可选）
```

## L3 - 实现视角

（待 L2 完成后填充）

## Review 历史

### 2026-04-19 - L1 Review

#### 工具分类表完整性
笔记中工具分类表遗漏了部分工具：
- BriefTool（未分类）
- SendUserFileTool（实验性）
- SubscribePRTool（实验性）
- ReviewArtifactTool（实验性）
- PushNotificationTool（实验性）
- TerminalCaptureTool（实验性）
> ⚠️ **补充说明**: 笔记分类表只列出了主要工具，遗漏的是少数实验性/条件加载工具。不影响理解核心设计。

#### ToolUseContext 字段数量
笔记疑问："ToolUseContext 的完整字段列表？"
> ⚠️ **验证结果**: 源码显示 ToolUseContext 约 50 个字段（options、abortController、messages、各种 state setter、回调等）。笔记标记为疑问是合理的，字段确实太多不适合在 L1 全部列举。

#### 其余内容验证
以下内容经源码验证全部正确：
- Tool.call 参数签名 ✅
- `ToolResult<T>` 结构 ✅
- TOOL_DEFAULTS 默认值（isConcurrencySafe=false 等） ✅
- aliases/toolMatchesName 机制 ✅
- shouldDefer/alwaysLoad 延迟加载 ✅
- MCP 兼容（isMcp, mcpInfo, inputJSONSchema） ✅
- 进度回调（`ToolCallProgress<P>`） ✅
- 结果渲染（mapToolResultToToolResultBlockParam） ✅

### 2026-04-19 - L2 Review

#### 核心函数表格验证
所有函数签名经源码验证正确：
- `toolMatchesName()` 签名 ✅
- `findToolByName()` 签名 ✅
- `Tool.call()` 参数顺序 ✅
- `interruptBehavior()` 返回类型 `'cancel' | 'block'` ✅

#### ToolUseContext 关键字段
笔记列举了约 30 个关键字段，源码实际约 50 个字段。笔记简化是合理的（只列核心字段）。

#### 工具生命周期流程
调用阶段的步骤顺序验证正确：
- validateInput → checkPermissions → canUseTool → call → mapToolResultToToolResultBlockParam

## 疑问与待查

- [ ] `ToolUseContext` 的完整字段列表？
- [ ] `checkPermissions()` 和通用权限检查的关系？
- [ ] `contextModifier` 的具体使用场景？
- [ ] 延迟加载工具（`shouldDefer`）的触发机制？

## 相关模块

- [[query]]
- [[permission]]
- [[message]]
- [[MCP]]
- [[REPL]]