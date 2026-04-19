# hook

> 项目: [[claude-code-best]]
> 文件: src/types/hooks.ts, src/utils/hooks.ts, src/utils/hooks/hookEvents.ts
> 状态: L3-complete

## L1 - 黑盒视角

### 职责

**Hook 系统**。定义生命周期钩子事件，执行用户配置的 shell 命令或回调函数，处理同步/异步 hook 结果，聚合多个 hook 的输出。

核心设计：**事件驱动 + JSON 协议 + 异步支持**，实现"生命周期扩展、权限干预、状态注入"的可插拔机制。

### 输入

**Hook 执行参数**:
- `hookEvent`: HookEvent — 钩子事件类型
- `hookInput`: HookInput — 事件输入数据（含 tool_name、input 等）
- `context`: ToolUseContext — 工具执行上下文
- `abort`: AbortSignal — 取消信号
- `hooksConfig`: HookMatcher[] — hook 配置列表

**HookInput 结构**（按事件类型）:
- `PreToolUseHookInput`: `session_id`, `transcript_path`, `tool_name`, `tool_input`
- `PostToolUseHookInput`: 上述 + `tool_result`, `tool_result_is_error`
- `SessionStartHookInput`: `session_id`, `transcript_path`, `cwd`, `hook_event_name`
- `UserPromptSubmitHookInput`: `session_id`, `transcript_path`, `prompt`
- 其他事件各有特定字段

### 输出

**HookResult**:
- `message`: Message? — hook 输出消息（显示在 transcript）
- `systemMessage`: Message? — 系统消息（不发给 API）
- `blockingError`: HookBlockingError? — 阻塞错误
- `outcome`: 'success' | 'blocking' | 'non_blocking_error' | 'cancelled'
- `preventContinuation`: boolean? — 是否阻止继续执行
- `permissionBehavior`: 'ask' | 'deny' | 'allow' | 'passthrough'? — 权限决策
- `additionalContext`: string? — 额外上下文注入
- `updatedInput`: Record? — 修改工具输入

**HookJSONOutput**（hook 命令输出）:
- Sync: `{ continue, suppressOutput, decision, reason, ... }`
- Async: `{ async: true, asyncTimeout }`

### 调用时机

按生命周期事件触发：

| 事件                     | 触发时机                  |
| ---------------------- | --------------------- |
| **PreToolUse**         | 工具执行前，可拦截/修改输入        |
| **PostToolUse**        | 工具执行后，可修改输出           |
| **PostToolUseFailure** | 工具执行失败后               |
| **SessionStart**       | 会话开始时                 |
| **SessionEnd**         | 会话结束时                 |
| **UserPromptSubmit**   | 用户提交消息时               |
| **PreCompact**         | 压缩前                   |
| **PostCompact**        | 压缩后                   |
| **PermissionRequest**  | 权限请求时（headless agent） |
| **PermissionDenied**   | 权限被拒绝后                |
| **Notification**       | 通知事件                  |
| **Stop**               | 停止事件                  |
| **SubagentStart**      | 子 agent 启动时           |
| **SubagentStop**       | 子 agent 结束时           |

### 调用方

- [[query]] — 处理 tool_use 前后触发 PreToolUse/PostToolUse
- [[permission]] — 权限检查调用 PermissionRequest hooks
- [[compact]] — 压缩前后触发 PreCompact/PostCompact
- [[REPL]] — 会话启动时触发 SessionStart
- [[Agent]] — 子 agent 启动触发 SubagentStart

### 黑盒调用记录

**类型导入**:
- `HookEvent` → 钩子事件枚举
- `HookInput` → 各事件的输入结构
- `HookJSONOutput` → hook 命令的 JSON 输出

**执行函数**:
- `executeHooks()` → 执行多个 hook，返回聚合结果
- `executePreToolUseHooks()` → 返回 `AggregatedHookResult`，PreToolUse 专用
- `executePostToolUseHooks()` → PostToolUse 专用
- `executeSessionStartHooks()` → SessionStart 专用
- `executePermissionRequestHooks()` → PermissionRequest 专用

**配置函数**:
- `getHooksConfigFromSnapshot()` → 返回 hook 配置
- `shouldSkipHookDueToTrust()` → 返回 boolean，检查 workspace trust

**事件广播**:
- `emitHookStarted()` → 广播 hook 开始事件
- `emitHookResponse()` → 广播 hook 结果事件
- `registerPendingAsyncHook()` → 注册异步 hook

### Hook 事件概览

```text
HOOK_EVENTS = [
  'PreToolUse', 'PostToolUse', 'PostToolUseFailure',
  'Notification', 'UserPromptSubmit',
  'SessionStart', 'SessionEnd',
  'Stop', 'StopFailure',
  'SubagentStart', 'SubagentStop',
  'PreCompact', 'PostCompact',
  'PermissionRequest', 'PermissionDenied',
  'Setup', 'TeammateIdle',
  'TaskCreated', 'TaskCompleted',
  'Elicitation', 'ElicitationResult',
  'ConfigChange', 'WorktreeCreate', 'WorktreeRemove',
  'InstructionsLoaded', 'CwdChanged', 'FileChanged'
]
```

### 关键设计决策

**1. 同步/异步模式**

- **同步 hook**: 执行完成后返回结果，阻塞后续流程
- **异步 hook**: 返回 `{ async: true }`，后台执行，不阻塞

异步 hook 用于长时间操作（如文件监控、后台任务）。

**2. JSON 协议**

hook 命令输出 JSON 格式：
```json
{
  "continue": true,
  "suppressOutput": false,
  "decision": "approve",
  "reason": "...",
  "additionalContext": "..."
}
```

使用 Zod schema 验证，无效 JSON 视为非阻塞错误。

**3. matcher 过滤**

hook 配置包含 matcher，只匹配特定条件：
- `matcher: "Bash"` → 只匹配 BashTool
- `matcher: "Bash(git *)"` → 只匹配 git 命令
- 无 matcher → 匹配所有

**4. 权限干预**

`PreToolUse` hook 可返回权限决策：
- `decision: "approve"` → 自动允许
- `decision: "block"` → 自动拒绝

hook 可覆盖权限系统，实现自定义审批逻辑。

**5. 输入修改**

`PreToolUse` hook 可返回 `updatedInput`：
- 修改工具输入参数
- 注入额外字段

例如：自动添加环境变量到 Bash 命令。

**6. Workspace Trust**

所有 hook 需要 workspace trust：
- `shouldSkipHookDueToTrust()` 检查
- 防止未信任目录执行恶意 hook

安全设计：hook 可执行任意命令，必须受信任。

**7. 多 hook 聚合**

同一事件可有多个 hook：
- 顺序执行（PreToolUse）
- 并行执行（SessionEnd）
- 聚合结果：合并 additionalContext、取最严格 permissionBehavior

**8. 超时控制**

- 默认超时：`TOOL_HOOK_EXECUTION_TIMEOUT_MS = 10 * 60 * 1000` (10 分钟)
- SessionEnd 超时：`1500ms`（快速关闭）
- 可通过环境变量覆盖

**9. 进度通知**

`HookProgress` 类型：
- UI 显示 hook 执行进度
- `type: 'hook_progress'`, `hookEvent`, `hookName`, `command`

**10. Session Hook**

会话级别的临时 hook：
- `getSessionHooks()` — 获取当前会话 hook
- 通过 `/hook` 命令动态添加
- 会话结束时清除

## L2 - 接口视角

### 核心函数

| 函数名 | 作用 | 关键参数 | 返回类型 |
|--------|------|----------|----------|
| `executeHooks()` | 通用 hook 执行 | `hookInput`, `toolUseID`, `matchQuery`, `signal`, `timeoutMs` | `AsyncGenerator<AggregatedHookResult>` |
| `executePreToolUseHooks()` | PreToolUse 执行 | `toolName`, `toolInput`, `toolUseID`, `context` | `AsyncGenerator<AggregatedHookResult>` |
| `executePostToolUseHooks()` | PostToolUse 执行 | `toolName`, `toolInput`, `toolResult`, `context` | `AsyncGenerator<AggregatedHookResult>` |
| `executeSessionStartHooks()` | SessionStart 执行 | `cwd`, `context` | `AsyncGenerator<AggregatedHookResult>` |
| `executePermissionRequestHooks()` | PermissionRequest 执行 | `toolName`, `toolUseID`, `input`, `context` | `AsyncGenerator<AggregatedHookResult>` |
| `executePreCompactHooks()` | PreCompact 执行 | `trigger`, `customInstructions` | `AsyncGenerator<AggregatedHookResult>` |
| `executePostCompactHooks()` | PostCompact 执行 | `trigger`, `context` | `AsyncGenerator<AggregatedHookResult>` |
| `getMatchingHooks()` | 获取匹配的 hook | `appState`, `sessionId`, `hookEvent`, `hookInput` | `HookMatcher[]` |
| `shouldSkipHookDueToTrust()` | 检查 workspace trust | 无 | `boolean` |
| `emitHookStarted()` | 广播开始事件 | `hookId`, `hookName`, `hookEvent` | `void` |
| `emitHookResponse()` | 广播结果事件 | `hookId`, `hookName`, `output` | `void` |

### 核心类型

```typescript
// HookEvent - 钩子事件类型（28 种）
export type HookEvent = 
  | 'PreToolUse' | 'PostToolUse' | 'PostToolUseFailure'
  | 'Notification' | 'UserPromptSubmit'
  | 'SessionStart' | 'SessionEnd'
  | 'Stop' | 'StopFailure'
  | 'SubagentStart' | 'SubagentStop'
  | 'PreCompact' | 'PostCompact'
  | 'PermissionRequest' | 'PermissionDenied'
  | 'Setup' | 'TeammateIdle'
  | 'TaskCreated' | 'TaskCompleted'
  | 'Elicitation' | 'ElicitationResult'
  | 'ConfigChange' | 'WorktreeCreate' | 'WorktreeRemove'
  | 'InstructionsLoaded' | 'CwdChanged' | 'FileChanged'

// HookResult - 单个 hook 结果
export type HookResult = {
  message?: Message
  systemMessage?: Message
  blockingError?: HookBlockingError
  outcome: 'success' | 'blocking' | 'non_blocking_error' | 'cancelled'
  preventContinuation?: boolean
  stopReason?: string
  permissionBehavior?: 'ask' | 'deny' | 'allow' | 'passthrough'
  hookPermissionDecisionReason?: string
  additionalContext?: string
  initialUserMessage?: string
  updatedInput?: Record<string, unknown>
  updatedMCPToolOutput?: unknown
  permissionRequestResult?: PermissionRequestResult
  retry?: boolean
}

// AggregatedHookResult - 多个 hook 聚合结果
export type AggregatedHookResult = {
  message?: Message
  blockingErrors?: HookBlockingError[]
  preventContinuation?: boolean
  stopReason?: string
  hookPermissionDecisionReason?: string
  permissionBehavior?: PermissionResult['behavior']
  additionalContexts?: string[]
  initialUserMessage?: string
  updatedInput?: Record<string, unknown>
  updatedMCPToolOutput?: unknown
  permissionRequestResult?: PermissionRequestResult
  retry?: boolean
}

// HookJSONOutput - hook 命令输出
export type HookJSONOutput = SyncHookJSONOutput | AsyncHookJSONOutput

export type SyncHookJSONOutput = {
  async?: false
  continue?: boolean      // 是否继续执行（默认 true）
  suppressOutput?: boolean // 是否隐藏输出
  stopReason?: string
  decision?: 'approve' | 'block'
  reason?: string
  systemMessage?: string
  hookSpecificOutput?: { hookEventName: string; ... }
}

export type AsyncHookJSONOutput = {
  async: true
  asyncTimeout?: number
}

// HookCallback - 回调类型 hook
export type HookCallback = {
  type: 'callback'
  callback: (input, toolUseID, abort, hookIndex?, context?) => Promise<HookJSONOutput>
  timeout?: number
  internal?: boolean
}

// HookProgress - 进度消息
export type HookProgress = {
  type: 'hook_progress'
  hookEvent: HookEvent
  hookName: string
  command: string
  promptText?: string
  statusMessage?: string
}

// HookBlockingError - 阻塞错误
export type HookBlockingError = {
  blockingError: string
  command: string
}

// PermissionRequestResult - 权限请求结果
export type PermissionRequestResult =
  | { behavior: 'allow'; updatedInput?: Record<string, unknown>; updatedPermissions?: PermissionUpdate[] }
  | { behavior: 'deny'; message?: string; interrupt?: boolean }
```

### HookInput 结构

```typescript
// PreToolUseHookInput
type PreToolUseHookInput = {
  hook_event_name: 'PreToolUse'
  session_id: string
  transcript_path: string
  tool_name: string
  tool_input: Record<string, unknown>
}

// PostToolUseHookInput
type PostToolUseHookInput = PreToolUseHookInput & {
  tool_result: string
  tool_result_is_error?: boolean
}

// SessionStartHookInput
type SessionStartHookInput = {
  hook_event_name: 'SessionStart'
  session_id: string
  transcript_path: string
  cwd: string
}

// UserPromptSubmitHookInput
type UserPromptSubmitHookInput = {
  hook_event_name: 'UserPromptSubmit'
  session_id: string
  transcript_path: string
  prompt: string
}

// PermissionRequestHookInput
type PermissionRequestHookInput = {
  hook_event_name: 'PermissionRequest'
  session_id: string
  transcript_path: string
  tool_name: string
  tool_input: Record<string, unknown>
  permission_mode?: string
  permission_suggestions?: PermissionUpdate[]
}
```

### Hook 执行流程

```text
executeHooks() 内部流程：

1. 前置检查
   - shouldDisableAllHooksIncludingManaged() → 禁用则退出
   - CLAUDE_CODE_SIMPLE → simple mode 退出
   - shouldSkipHookDueToTrust() → 无信任则退出

2. 获取匹配 hook
   - getMatchingHooks(appState, sessionId, hookEvent, hookInput)
   - 按 matcher 筛选

3. 快速路径（全部 internal callback）
   - 直接调用 callback，不启动 span/progress
   - 测量耗时，返回

4. 常规路径
   - 启动 hookSpan（beta tracing）
   - 创建 combinedAbortSignal
   - 遍历 matchingHooks:
     a. 获取 hook 配置（command/callback）
     b. emitHookStarted()
     c. 执行 hook（shell/callback）
     d. 解析 JSON 输出
     e. 处理异步 hook → registerPendingAsyncHook()
     f. 聚合结果
   - yield AggregatedHookResult

5. 结束
   - endHookSpan()
   - 记录 analytics
```

### matcher 匹配逻辑

```text
getMatchingHooks() 匹配逻辑：

1. 获取 hook 配置快照
   - getHooksConfigFromSnapshot()
   - 包含 user/project/local/flag/policy sources

2. 过滤条件
   - hookEvent 匹配
   - matcher 匹配（如有）
   
3. matcher 语法
   - 无 matcher → 匹配所有事件
   - "Bash" → 匹配 BashTool
   - "Bash(git *)" → 匹配 git 命令
   - 支持通配符和模式

4. Session hook 合并
   - getSessionHooks() → 会话临时 hook
   - 与配置快照合并
```

## L3 - 实现视角

### 关键代码路径

**executeHooks 主流程**:

```text
async function* executeHooks({
  hookInput, toolUseID, matchQuery, signal, timeoutMs, toolUseContext, ...
}): AsyncGenerator<AggregatedHookResult> {

  // Step 1: 前置检查
  if (shouldDisableAllHooksIncludingManaged()) return
  if (CLAUDE_CODE_SIMPLE) return
  if (shouldSkipHookDueToTrust()) return  // 安全：未信任 workspace 不执行

  // Step 2: 获取匹配 hook
  const matchingHooks = await getMatchingHooks(appState, sessionId, hookEvent, hookInput)
  if (matchingHooks.length === 0) return

  // Step 3: 快速路径（全部 internal callback）
  if (userHooks.length === 0) {
    // 只有 internal callback（如 sessionFileAccessHooks）
    // 跳过 span/progress/abortSignal/processHookJSONOutput
    // 测量耗时，直接调用 callback，return
    for (const { hook } of matchingHooks) {
      if (hook.type === 'callback') {
        await hook.callback(hookInput, toolUseID, signal, i, context)
      }
    }
    return  // 不 yield，直接退出
  }

  // Step 4: 常规路径
  const hookSpan = startHookSpan(...)  // beta tracing
  const batchStartTime = Date.now()

  // Step 4a: yield progress message（每个 hook 一个）
  for (const { hook } of matchingHooks) {
    yield { message: { type: 'progress', data: { type: 'hook_progress', ... } } }
  }

  // Step 4b: 惰性 stringify hookInput（共享给所有 command/prompt hooks）
  let jsonInputResult: { ok: true; value: string } | { ok: false; error } | undefined
  function getJsonInput() {
    if (jsonInputResult !== undefined) return jsonInputResult
    return (jsonInputResult = { ok: true, value: jsonStringify(hookInput) })
  }

  // Step 4c: 并行执行所有 hook（每个 hook 一个 async generator）
  const hookPromises = matchingHooks.map(async function* ({ hook, ... }): AsyncGenerator<HookResult> {
    const { signal: abortSignal, cleanup } = createCombinedAbortSignal(signal, { timeoutMs })

    if (hook.type === 'callback') {
      yield executeHookCallback(...).finally(cleanup)
      return
    }

    if (hook.type === 'function') {
      yield executeFunctionHook(...)
      return
    }

    // Command/Prompt/Agent/HTTP hooks 需要 jsonInput
    const jsonInput = getJsonInput().value

    if (hook.type === 'prompt') {
      yield execPromptHook(...)
      cleanup()
      return
    }

    if (hook.type === 'agent') {
      yield execAgentHook(...)
      cleanup()
      return
    }

    if (hook.type === 'http') {
      emitHookStarted(hookId, hookName, hookEvent)
      const httpResult = await execHttpHook(...)
      emitHookResponse(...)
      yield httpResult
      cleanup()
      return
    }

    // Command hook（最常见）
    emitHookStarted(hookId, hookName, hookEvent)
    const { stdout, stderr, status, aborted, backgrounded } = await execCommandHook(...)

    if (backgrounded) {
      // 异步 hook：已注册到 AsyncHookRegistry
      yield { outcome: 'success', hook }
      cleanup()
      return
    }

    emitHookResponse(...)

    if (status === 2) {
      // Exit code 2 = blocking error
      yield { blockingError: { blockingError: stderr || stdout, command }, outcome: 'blocking', hook }
      cleanup()
      return
    }

    if (aborted) {
      yield { outcome: 'cancelled', hook }
      cleanup()
      return
    }

    // 解析 JSON 输出
    const parsed = parseHookOutput(stdout)
    if (parsed.json) {
      yield processHookJSONOutput({ json: parsed.json, ... })
    } else {
      yield { message: ..., outcome: 'non_blocking_error', hook }
    }
    cleanup()
  })

  // Step 5: 聚合结果（遍历所有 hook generator）
  const aggregated: AggregatedHookResult = {}
  for await (const hookResult of all(hookPromises)) {
    // 合并 additionalContext
    if (hookResult.additionalContext) {
      aggregated.additionalContexts = [...(aggregated.additionalContexts || []), hookResult.additionalContext]
    }

    // 取最严格 permissionBehavior
    if (hookResult.permissionBehavior) {
      const current = aggregated.permissionBehavior
      if (!current || current === 'allow' || current === 'passthrough') {
        aggregated.permissionBehavior = hookResult.permissionBehavior
      }
    }

    // 合并 blockingErrors
    if (hookResult.blockingError) {
      aggregated.blockingErrors = [...(aggregated.blockingErrors || []), hookResult.blockingError]
    }

    // preventContinuation: 任一 hook 设置则整体阻止
    if (hookResult.preventContinuation) {
      aggregated.preventContinuation = true
    }

    // updatedInput: 最后一个有效值覆盖
    if (hookResult.updatedInput) {
      aggregated.updatedInput = hookResult.updatedInput
    }
  }

  // Step 6: yield 聚合结果
  yield aggregated

  // Step 7: 结束
  endHookSpan(hookSpan)
  const totalDurationMs = Date.now() - batchStartTime
  logEvent('tengu_repl_hook_finished', { hookName, numCommands, totalDurationMs, ... })
}
```

**getMatchingHooks 实现**:

```text
async function getMatchingHooks(appState, sessionId, hookEvent, hookInput, tools): Promise<MatchedHook[]> {

  // Step 1: 获取 hook 配置
  const hookMatchers = getHooksConfig(appState, sessionId, hookEvent)

  // Step 2: 确定 matchQuery（按事件类型）
  let matchQuery: string | undefined
  switch (hookInput.hook_event_name) {
    case 'PreToolUse': matchQuery = hookInput.tool_name  // 匹配工具名
    case 'PostToolUse': matchQuery = hookInput.tool_name
    case 'SessionStart': matchQuery = hookInput.source   // 匹配来源
    case 'PreCompact': matchQuery = hookInput.trigger    // 匹配触发器
    case 'FileChanged': matchQuery = basename(hookInput.file_path)  // 匹配文件名
    ...
  }

  // Step 3: 过滤 matcher
  const filteredMatchers = matchQuery
    ? hookMatchers.filter(m => !m.matcher || matchesPattern(matchQuery, m.matcher))
    : hookMatchers

  // Step 4: 展开 hooks（每个 matcher 可含多个 hook）
  const matchedHooks = filteredMatchers.flatMap(matcher => {
    const pluginRoot = matcher.pluginRoot
    const skillRoot = matcher.skillRoot
    const hookSource = pluginRoot ? `plugin:${matcher.pluginName}` : skillRoot ? `skill:...` : 'settings'
    return matcher.hooks.map(hook => ({ hook, pluginRoot, skillRoot, hookSource }))
  })

  // Step 5: 快速路径（全部 callback/function）
  if (matchedHooks.every(m => m.hook.type === 'callback' || m.hook.type === 'function')) {
    return matchedHooks  // 不需要 dedup
  }

  // Step 6: deduplication（按类型分别去重）
  const uniqueCommandHooks = Array.from(new Map(
    matchedHooks.filter(m => m.hook.type === 'command')
      .map(m => [hookDedupKey(m, `${m.hook.shell ?? 'bash'}\0${m.hook.command}`), m])
  ).values())

  const uniquePromptHooks = Array.from(new Map(
    matchedHooks.filter(m => m.hook.type === 'prompt')
      .map(m => [hookDedupKey(m, `${m.hook.prompt}`), m])
  ).values())

  // ... 类似处理 agent/http hooks

  const callbackHooks = matchedHooks.filter(m => m.hook.type === 'callback')
  const functionHooks = matchedHooks.filter(m => m.hook.type === 'function')

  // Step 7: 合并返回
  return [...uniqueCommandHooks, ...uniquePromptHooks, ...uniqueAgentHooks, ...uniqueHttpHooks, ...callbackHooks, ...functionHooks]
}
```

**execCommandHook 实现**:

```text
async function execCommandHook(hook, hookEvent, hookName, jsonInput, signal, hookId, ...): Promise<{
  stdout, stderr, output, status, aborted?, backgrounded?
}> {

  // Step 1: Shell 选择
  const shellType = hook.shell ?? DEFAULT_HOOK_SHELL  // 'bash' | 'powershell'
  const isPowerShell = shellType === 'powershell'

  // Step 2: Windows 路径转换（bash 用 POSIX path）
  const toHookPath = isWindows && !isPowerShell
    ? (p: string) => windowsPathToPosixPath(p)  // C:\Users\foo -> /c/Users/foo
    : (p: string) => p

  // Step 3: 变量替换
  let command = hook.command
  if (pluginRoot) {
    command = command.replace(/\$\{CLAUDE_PLUGIN_ROOT\}/g, () => toHookPath(pluginRoot))
    if (pluginId) {
      command = command.replace(/\$\{CLAUDE_PLUGIN_DATA\}/g, () => toHookPath(getPluginDataDir(pluginId)))
    }
    command = substituteUserConfigVariables(command, pluginOpts)
  }

  // Step 4: Windows .sh 自动 prepend bash
  if (isWindows && !isPowerShell && command.trim().match(/\.sh(\s|$|")/)) {
    if (!command.trim().startsWith('bash ')) {
      command = `bash ${command}`
    }
  }

  // Step 5: CLAUDE_CODE_SHELL_PREFIX 包装
  const finalCommand = !isPowerShell && process.env.CLAUDE_CODE_SHELL_PREFIX
    ? formatShellPrefixCommand(process.env.CLAUDE_CODE_SHELL_PREFIX, command)
    : command

  // Step 6: 构建 env vars
  const envVars = {
    ...subprocessEnv(),
    CLAUDE_PROJECT_DIR: toHookPath(projectDir),
    CLAUDE_ENV_FILE: getHookEnvFilePath(hookIndex),  // hook 写入 env var 定义
  }

  // Step 7: 执行 shell 命令
  const shellCommand = wrapSpawn({
    command: finalCommand,
    shell: isPowerShell ? 'pwsh' : undefined,
    signal: abortSignal,
    timeoutMs: hook.timeout ? hook.timeout * 1000 : timeoutMs,
    env: envVars,
    stdin: jsonInput,  // hook input 作为 stdin
  })

  const result = await shellCommand.result

  // Step 8: 处理异步 hook
  if (parsed.json.async === true) {
    executeInBackground({ processId, hookId, shellCommand, asyncResponse: parsed.json, ... })
    return { stdout: '', stderr: '', output: '', status: 0, backgrounded: true }
  }

  // Step 9: 返回结果
  return {
    stdout: shellCommand.taskOutput.getStdout(),
    stderr: shellCommand.taskOutput.getStderr(),
    output: stdout + stderr,
    status: result.code,
    aborted: result.aborted,
  }
}
```

**AsyncHookRegistry 实现**:

```text
// 全局注册表
const pendingHooks = new Map<string, PendingAsyncHook>()

function registerPendingAsyncHook({ processId, hookId, asyncResponse, ... }): void {
  const timeout = asyncResponse.asyncTimeout || 15000  // 默认 15s

  // 启动进度轮询（每秒 emit progress event）
  const stopProgressInterval = startHookProgressInterval({ hookId, getOutput: ... })

  pendingHooks.set(processId, {
    processId, hookId, hookName, hookEvent, command,
    startTime: Date.now(),
    timeout,
    responseAttachmentSent: false,
    shellCommand,
    stopProgressInterval,
  })
}

async function checkForAsyncHookResponses(): Promise<Array<{
  processId, response, hookName, stdout, stderr, exitCode
}>> {
  const responses = []
  const hooks = Array.from(pendingHooks.values())

  for (const hook of hooks) {
    if (hook.shellCommand.status !== 'completed') continue

    const stdout = await hook.shellCommand.taskOutput.getStdout()
    if (hook.responseAttachmentSent || !stdout.trim()) {
      pendingHooks.delete(hook.processId)
      continue
    }

    // 解析 stdout 中的 JSON（跳过 async 行）
    for (const line of stdout.split('\n')) {
      if (line.trim().startsWith('{')) {
        const parsed = jsonParse(line.trim())
        if (!('async' in parsed)) {
          response = parsed
          break
        }
      }
    }

    hook.responseAttachmentSent = true
    await finalizeHook(hook, exitCode, outcome)
    responses.push({ processId, response, ... })
    pendingHooks.delete(hook.processId)
  }

  return responses
}

async function finalizePendingAsyncHooks(): Promise<void> {
  for (const hook of pendingHooks.values()) {
    if (hook.shellCommand?.status === 'completed') {
      await finalizeHook(hook, result.code, 'success')
    } else {
      hook.shellCommand?.kill()
      await finalizeHook(hook, 1, 'cancelled')
    }
  }
  pendingHooks.clear()
}
```

**processHookJSONOutput 实现**:

```text
function processHookJSONOutput({ json, command, hookName, ... }): Partial<HookResult> {
  const result: Partial<HookResult> = {}

  // Step 1: continue 字段
  if (json.continue === false) {
    result.preventContinuation = true
    if (json.stopReason) result.stopReason = json.stopReason
  }

  // Step 2: decision 字段（approve/block）
  if (json.decision) {
    switch (json.decision) {
      case 'approve': result.permissionBehavior = 'allow'
      case 'block':
        result.permissionBehavior = 'deny'
        result.blockingError = { blockingError: json.reason || 'Blocked by hook', command }
    }
  }

  // Step 3: hookSpecificOutput（按事件类型）
  if (json.hookSpecificOutput) {
    switch (json.hookSpecificOutput.hookEventName) {
      case 'PreToolUse':
        if (json.hookSpecificOutput.permissionDecision) {
          result.permissionBehavior = json.hookSpecificOutput.permissionDecision
        }
        if (json.hookSpecificOutput.updatedInput) {
          result.updatedInput = json.hookSpecificOutput.updatedInput
        }
        result.additionalContext = json.hookSpecificOutput.additionalContext
        break

      case 'SessionStart':
        result.initialUserMessage = json.hookSpecificOutput.initialUserMessage
        result.watchPaths = json.hookSpecificOutput.watchPaths  // FileChanged 监控路径
        break

      case 'PostToolUse':
        result.updatedMCPToolOutput = json.hookSpecificOutput.updatedMCPToolOutput
        break

      case 'PermissionRequest':
        result.permissionRequestResult = json.hookSpecificOutput.decision
        break
      ...
    }
  }

  // Step 4: 创建 attachment message
  result.message = result.blockingError
    ? createAttachmentMessage({ type: 'hook_blocking_error', ... })
    : createAttachmentMessage({ type: 'hook_success', content: '', ... })

  return result
}
```

### 边界条件

**1. Exit Code 语义**

- `exit 0` → success
- `exit 2` → **blocking error**（约定：阻塞后续流程）
- 其他 → non_blocking_error

设计原因：允许 hook 通过 exit code 控制流程，无需解析 JSON。

**2. JSON 输出验证**

使用 Zod schema 验证：
- 无效 JSON → 视为 plain text，outcome 为 `non_blocking_error`
- schema mismatch → 记录 validationError，继续执行（不阻塞）

Fail-open 设计：hook 错误不应阻止主流程。

**3. Matcher 通配符匹配**

`matchesPattern(matchQuery, matcher)`:
- `matcher: "Bash"` → 精确匹配 `tool_name: "Bash"`
- `matcher: "Bash(git *)"` → 正则匹配 `tool_name` + `input.command`
- 无 matcher → 匹配所有事件

**4. Hook Deduplication**

同类型 hook 按 command/prompt/url 去重：
- Key: `pluginRoot + command` (namespaced)
- 不同 plugin 可有相同 command（不冲突）
- 同一 plugin 重复定义 → 后者覆盖

**5. 异步 Hook 超时**

- 默认 `asyncTimeout: 15000` (15s)
- 超时后 hook 继续运行，但 progress interval 停止
- `checkForAsyncHookResponses()` 检查完成状态

**6. Workspace Trust 安全**

`shouldSkipHookDueToTrust()` 检查：
- Interactive mode → 必须 trust accepted
- Non-interactive (SDK) → 信任隐式，跳过检查

历史漏洞：SessionEnd/SubagentStop 在 trust dialog 前执行。

**7. Internal Hook 快速路径**

全部为 internal callback 时：
- 跳过 JSON stringify（无 command hooks）
- 跳过 span/progress/emitHookStarted
- 直接调用 callback，测量总耗时

优化：减少 ~70% overhead（PostToolUse internal hooks 常见）。

**8. Hook Input 惰性 Stringify**

`getJsonInput()`:
- 第一次调用时 stringify
- 缓存结果，共享给所有 command/prompt hooks
- 错误时 yield non_blocking_error，不阻塞其他 hooks

**9. 进度消息 Yield**

每个 hook yield 一个 progress message：
- `{ type: 'progress', data: { type: 'hook_progress', hookEvent, hookName, command } }`
- UI 显示 "Running hook: ..."
- 不影响聚合结果

**10. Abort Signal 组合**

`createCombinedAbortSignal(signal, { timeoutMs })`:
- 外部 signal（用户取消）
- 内部 timeout（hook 超时）
- 任一触发则 abort

cleanup() 在 finally 中调用，确保资源释放。

### 性能考量

**1. Internal Hook 快速路径**

测量：6.01µs → ~1.8µs per PostToolUse hit (-70%)

优化点：
- 跳过 JSON stringify
- 跳过 span/progress/emitHookResponse
- 直接调用 callback（无 shell spawn）

**2. Hook Input 惰性 Stringify**

共享给所有 command hooks：
- N hooks → 1 stringify
- callback hooks 不触发 stringify

**3. 并行执行**

所有 hooks 并行执行（Promise.all 风格）：
- `all(hookPromises)` 遍历 generator
- 每个 hook 有独立 timeout
- 不互相阻塞

**4. Deduplication 快速路径**

全部 callback/function 时跳过：
- 6-pass filter + 4×Map + 4×Array.from
- 44x faster in microbench

**5. 进度轮询 Interval**

`startHookProgressInterval`:
- 默认 1000ms 轮询
- `interval.unref()` 防止阻塞进程退出
- output 未变化时跳过 emit

**6. Shell 命令执行**

`wrapSpawn` 使用 `child_process.spawn`：
- 流式 stdout/stderr（实时捕获）
- TaskOutput 累积数据
- background mode 持续运行

**7. Hook Span Tracing**

Beta tracing 启用时：
- `startHookSpan` / `endHookSpan` 记录 OTel event
- 生产环境默认关闭（减少 overhead）

**8. Analytics 采样**

`tengu_run_hook` 只记录 user hooks：
- internal hooks 不计入（不影响 metrics）
- hookTypeCounts 分类统计

**9. Windows 路径缓存**

`windowsPathToPosixPath` LRU-500 缓存：
- `C:\Users\foo` → `/c/Users/foo`
- 重复转换无开销

**10. Event Handler Pending Buffer**

`pendingEvents` 数组：
- handler 注册前的事件暂存
- MAX_PENDING_EVENTS = 100（防止内存泄漏）
- 注册后批量发送

## Review 历史

### 2026-04-19 - L1 Review

#### HOOK_EVENTS 列表验证
笔记中 HOOK_EVENTS 列表正确：
- 28 个事件类型 ✅
- 包含 PreToolUse, PostToolUse, SessionStart, PermissionRequest 等 ✅

#### 超时常量验证
- `TOOL_HOOK_EXECUTION_TIMEOUT_MS = 10 * 60 * 1000` (10 分钟) ✅
- SessionEnd 超时 `1500ms` ✅

#### HookResult 类型验证
字段列表正确：
- `outcome`: 'success' | 'blocking' | 'non_blocking_error' | 'cancelled' ✅
- `permissionBehavior` 字段存在 ✅

#### 其余内容验证
- 同步/异步模式设计 ✅
- matcher 过滤机制 ✅
- Workspace Trust 安全检查 ✅

### 2026-04-19 - L3 Review

#### executeHooks 流程验证
源码步骤正确：
- Step 1: shouldDisableAllHooksIncludingManaged() ✅
- Step 3: 快速路径（userHooks.length === 0）✅
- Step 4: 并行执行 hookPromises ✅
- Step 5: 聚合结果逻辑 ✅

#### Internal Hook 快速路径验证
源码注释确认：
- `6.01µs → ~1.8µs per PostToolUse hit (-70%)` ✅ (Line 2178)
- 跳过 span/progress/abortSignal/processHookJSONOutput ✅

#### Deduplication 快速路径验证
源码注释确认：
- `44x faster in microbench` ✅ (Line 1860)
- Skip 6-pass filter + 4×Map + 4×Array.from ✅

#### Exit Code 语义验证
源码确认：
- `exit code 2` = blocking error ✅ (Line 237, 2786, 3469)
- `outcome: 'blocking'`, `'cancelled'`, `'non_blocking_error'` ✅

#### matchesPattern 实现验证
源码逻辑正确：
- 无 matcher 或 `*` → true ✅
- 管道分隔 `Bash|Read` → 多个精确匹配 ✅
- 正则匹配 `Bash(git *)` ✅

#### AsyncHookRegistry 验证
源码确认：
- 默认 timeout 15000ms ✅ (Line 51)
- `checkForAsyncHookResponses()` 流程 ✅
- `finalizePendingAsyncHooks()` 清理逻辑 ✅

#### TIMEOUT 常量验证
- `TOOL_HOOK_EXECUTION_TIMEOUT_MS = 10 * 60 * 1000` ✅ (Line 167)
- `SESSION_END_HOOK_TIMEOUT_MS_DEFAULT = 1500` ✅ (Line 176)

#### executeHooks 流程验证
源码步骤正确：
- shouldDisableAllHooksIncludingManaged() 检查 ✅
- getMatchingHooks() 匹配逻辑 ✅
- internal callback 快速路径 ✅
- emitHookStarted/emitHookResponse 广播 ✅

#### HookInput 结构验证
各事件输入字段正确：
- PreToolUseHookInput: session_id, tool_name, tool_input ✅
- PostToolUseHookInput: 增加 tool_result ✅
- PermissionRequestHookInput: 增加 permission_mode ✅

## 疑问与待查

- [ ] `executeHooks()` 的完整执行流程？
- [ ] matcher 匹配逻辑的具体实现？
- [ ] 异步 hook 的结果如何回传？
- [ ] hook 与 permission 的交互机制？

## 相关模块

- [[tool]]
- [[permission]]
- [[query]]
- [[compact]]
- [[REPL]]