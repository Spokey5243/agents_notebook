# hook

> 项目: [[claude-code-best]]
> 文件: src/types/hooks.ts, src/utils/hooks.ts, src/utils/hooks/hookEvents.ts
> 状态: L1-in-progress

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

（待 L1 review 完成后填充）

## L3 - 实现视角

（待 L2 完成后填充）

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