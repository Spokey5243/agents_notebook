# REPL

> 项目: [[claude-code-best]]
> 文件: src/screens/REPL.tsx (6314 行)
> 状态: L1-complete

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

（待 L1 完成后填充）

## L3 - 实现视角

（待 L2 完成后填充）

## Review 历史

### 2026-04-19 - L1 Review

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