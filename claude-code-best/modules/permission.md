# permission

> 项目: [[claude-code-best]]
> 文件: src/types/permissions.ts, src/utils/permissions/permissions.ts, src/utils/permissions/PermissionMode.ts
> 状态: L1-in-progress

## L1 - 黑盒视角

### 职责

**权限管理系统**。定义权限模式、规则、决策类型，以及工具调用的权限检查流程。支持多种权限模式（default、acceptEdits、bypassPermissions、dontAsk、plan、auto），规则匹配（allow/deny/ask），以及 AI classifier 自动决策。

核心设计：**多模式 + 规则优先级 + AI classifier**，实现"灵活控制、规则匹配、智能自动审批"的权限管理。

### 输入

**权限检查参数** (`hasPermissionsToUseTool`):
- `tool`: Tool — 待检查的工具
- `input`: object — 工具输入参数
- `context`: ToolUseContext — 工具执行上下文（含 permissionContext）
- `assistantMessage`: AssistantMessage — 父消息
- `toolUseID`: string — 工具调用 ID

**权限上下文** (`ToolPermissionContext`):
- `mode`: PermissionMode — 当前权限模式
- `additionalWorkingDirectories`: Map — 额外允许的工作目录
- `alwaysAllowRules`: ToolPermissionRulesBySource — 允许规则
- `alwaysDenyRules`: ToolPermissionRulesBySource — 拒绝规则
- `alwaysAskRules`: ToolPermissionRulesBySource — 询问规则
- `isBypassPermissionsModeAvailable`: boolean — bypass 是否可用

### 输出

**PermissionResult/PermissionDecision**:
- `behavior`: 'allow' | 'deny' | 'ask' | 'passthrough' — 权限行为
- `message`: string? — 拒绝/询问消息
- `updatedInput`: object? — 修改后的输入
- `decisionReason`: PermissionDecisionReason? — 决策原因
- `suggestions`: PermissionUpdate[]? — 建议的权限更新
- `contentBlocks`: ContentBlockParam[]? — 附加内容块

### 调用时机

1. **工具调用前** — 每次工具执行前必须通过权限检查
2. **工具列表过滤** — `getTools()` 根据 deny rules 过滤可用工具
3. **Agent 启动** — fork agent 时继承或修改权限上下文
4. **MCP 工具集成** — MCP 工具也受权限规则约束

### 调用方

- [[tool]] — 工具执行前调用 `hasPermissionsToUseTool`
- [[query]] — 处理 tool_use 前检查权限
- [[REPL]] — 权限提示 UI 渲染
- [[MCP]] — MCP 工具权限过滤

### 黑盒调用记录

**类型导入**:
- `PermissionBehavior` → 'allow' | 'deny' | 'ask'
- `PermissionMode` → 权限模式枚举
- `ToolPermissionContext` → 权限上下文结构

**权限检查函数**:
- `hasPermissionsToUseTool()` → 返回 `Promise<PermissionDecision>`, 核心权限检查入口
- `toolAlwaysAllowedRule()` → 返回 `PermissionRule | null`, 检查工具是否被 allow 规则匹配
- `getDenyRuleForTool()` → 返回 `PermissionRule | null`, 检查工具是否被 deny 规则匹配
- `getAskRuleForTool()` → 返回 `PermissionRule | null`, 检查工具是否被 ask 规则匹配

**规则管理**:
- `getAllowRules()` → 返回 `PermissionRule[]`, 获取所有允许规则
- `getDenyRules()` → 返回 `PermissionRule[]`, 获取所有拒绝规则
- `getAskRules()` → 返回 `PermissionRule[]`, 获取所有询问规则
- `getRuleByContentsForTool()` → 返回 `Map<string, PermissionRule>`, 获取工具的规则内容映射

**模式相关**:
- `permissionModeTitle()` → 返回 string, 获取模式显示标题
- `isDefaultMode()` → 返回 boolean, 判断是否默认模式
- `toExternalPermissionMode()` → 返回 ExternalPermissionMode, 转换为外部可见模式

**其他**:
- `filterDeniedAgents()` → 返回 `T[]`, 过滤被 deny 的 agent
- `createPermissionRequestMessage()` → 返回 string, 创建权限请求消息

### 权限模式概览

| 模式 | 说明 | 行为 |
|------|------|------|
| **default** | 默认模式 | 无规则匹配时询问用户 |
| **acceptEdits** | 接受编辑 | 自动允许工作目录内的文件编辑 |
| **bypassPermissions** | 绕过权限 | 自动允许所有操作（危险） |
| **dontAsk** | 不询问 | 无规则匹配时自动拒绝 |
| **plan** | 规划模式 | 规划期间特殊处理 |
| **auto** | 自动模式（实验性） | 使用 AI classifier 自动决策 |

### 权限规则概览

**规则结构**: `ToolName(ruleContent)` 例如 `Bash(git *)`

| 规则类型 | 说明 | 示例 |
|----------|------|------|
| **工具级** | 匹配整个工具 | `Bash` |
| **参数级** | 匹配工具参数 | `Bash(git status)` |
| **MCP 级** | 匹配 MCP server | `mcp__server1` |
| **Agent 级** | 匹配特定 Agent | `Agent(Explore)` |
| **通配符** | 匹配所有工具 | `Bash(*)` |

### 关键设计决策

**1. 多模式分层**

- **default**: 基础模式，无匹配时询问
- **acceptEdits**: 快速路径，工作目录编辑自动允许
- **bypassPermissions**: 完全放开（危险，需 feature gate）
- **dontAsk**: 反向模式，无匹配时拒绝
- **auto**: AI classifier 自动审批（实验性）

**2. 规则优先级**

规则按 source 优先级排序：
- `userSettings` > `projectSettings` > `localSettings` > `flagSettings` > `policySettings`
- `cliArg` > `command` > `session`

**3. 三层规则体系**

- **alwaysAllowRules**: 自动允许
- **alwaysDenyRules**: 自动拒绝（优先于 allow）
- **alwaysAskRules**: 强制询问

**4. 模式转换**

- `dontAsk` 模式将 `ask` 转换为 `deny`
- `bypassPermissions` 模式将所有结果转换为 `allow`
- `auto` 模式使用 classifier 处理 `ask`

**5. deny 优先原则**

deny 规则优先于 allow 规则：
- 工具级 deny → 工具不可见
- 参数级 deny → 特定操作被拒绝

**6. Classifier 自动审批**

auto 模式使用 AI classifier：
- 分析工具调用上下文
- 判断是否安全可自动允许
- 支持 fail-closed（classifier 不可用时拒绝）

**7. 熔断机制**

连续拒绝达到阈值后：
- `shouldFallbackToPrompting()` 返回 true
- auto 模式退回到询问用户

**8. Hook 集成**

权限检查与 hook 系统：
- `PreToolUse` hook 可修改权限决策
- `PermissionRequest` hook 处理 headless agent

**9. 工作目录扩展**

`additionalWorkingDirectories` 扩展权限范围：
- 允许访问项目外的目录
- source 记录来源

**10. MCP 工具权限**

MCP 工具规则匹配：
- `mcp__server__tool` 格式
- server 级规则 `mcp__server` 匹配所有工具
- 通配符 `mcp__server__*` 同样效果

## L2 - 接口视角

### 核心函数

| 函数名 | 作用 | 关键参数 | 返回类型 |
|--------|------|----------|----------|
| `hasPermissionsToUseTool()` | 权限检查入口 | `tool`, `input`, `context`, `assistantMessage`, `toolUseID` | `Promise<PermissionDecision>` |
| `hasPermissionsToUseToolInner()` | 内部检查逻辑 | `tool`, `input`, `context` | `Promise<PermissionDecision>` |
| `toolAlwaysAllowedRule()` | 检查工具 allow 规则 | `context`, `tool` | `PermissionRule | null` |
| `getDenyRuleForTool()` | 检查工具 deny 规则 | `context`, `tool` | `PermissionRule | null` |
| `getAskRuleForTool()` | 检查工具 ask 规则 | `context`, `tool` | `PermissionRule | null` |
| `getAllowRules()` | 获取所有 allow 规则 | `context` | `PermissionRule[]` |
| `getDenyRules()` | 获取所有 deny 规则 | `context` | `PermissionRule[]` |
| `getAskRules()` | 获取所有 ask 规则 | `context` | `PermissionRule[]` |
| `toolMatchesRule()` | 工具匹配规则 | `tool`, `rule` | `boolean` |
| `createPermissionRequestMessage()` | 创建请求消息 | `toolName`, `decisionReason` | `string` |
| `permissionModeTitle()` | 获取模式标题 | `mode` | `string` |
| `isDefaultMode()` | 判断默认模式 | `mode` | `boolean` |

### 核心类型

```typescript
// PermissionMode - 权限模式
export type PermissionMode = 
  | 'default' 
  | 'acceptEdits' 
  | 'bypassPermissions' 
  | 'dontAsk' 
  | 'plan' 
  | 'auto'  // 实验性

// PermissionBehavior - 权限行为
export type PermissionBehavior = 'allow' | 'deny' | 'ask'

// PermissionDecision - 权限决策（三种行为）
export type PermissionDecision<Input = { [key: string]: unknown }> =
  | PermissionAllowDecision<Input>
  | PermissionAskDecision<Input>
  | PermissionDenyDecision

// PermissionAllowDecision
export type PermissionAllowDecision<Input> = {
  behavior: 'allow'
  updatedInput?: Input
  userModified?: boolean
  decisionReason?: PermissionDecisionReason
  toolUseID?: string
  acceptFeedback?: string
  contentBlocks?: ContentBlockParam[]
}

// PermissionAskDecision
export type PermissionAskDecision<Input> = {
  behavior: 'ask'
  message: string
  updatedInput?: Input
  decisionReason?: PermissionDecisionReason
  suggestions?: PermissionUpdate[]
  blockedPath?: string
  metadata?: PermissionMetadata
  pendingClassifierCheck?: PendingClassifierCheck
  contentBlocks?: ContentBlockParam[]
}

// PermissionDenyDecision
export type PermissionDenyDecision = {
  behavior: 'deny'
  message: string
  decisionReason: PermissionDecisionReason
  toolUseID?: string
}

// PermissionRule - 权限规则
export type PermissionRule = {
  source: PermissionRuleSource
  ruleBehavior: PermissionBehavior
  ruleValue: PermissionRuleValue
}

// PermissionRuleValue - 规则值
export type PermissionRuleValue = {
  toolName: string
  ruleContent?: string  // 如 "git *" for Bash(git *)
}

// PermissionRuleSource - 规则来源
export type PermissionRuleSource =
  | 'userSettings'
  | 'projectSettings'
  | 'localSettings'
  | 'flagSettings'
  | 'policySettings'
  | 'cliArg'
  | 'command'
  | 'session'

// ToolPermissionContext - 权限上下文
export type ToolPermissionContext = {
  readonly mode: PermissionMode
  readonly additionalWorkingDirectories: ReadonlyMap<string, AdditionalWorkingDirectory>
  readonly alwaysAllowRules: ToolPermissionRulesBySource
  readonly alwaysDenyRules: ToolPermissionRulesBySource
  readonly alwaysAskRules: ToolPermissionRulesBySource
  readonly isBypassPermissionsModeAvailable: boolean
  readonly strippedDangerousRules?: ToolPermissionRulesBySource
  readonly shouldAvoidPermissionPrompts?: boolean
  readonly awaitAutomatedChecksBeforeDialog?: boolean
  readonly prePlanMode?: PermissionMode
}

// PermissionDecisionReason - 决策原因
export type PermissionDecisionReason =
  | { type: 'rule'; rule: PermissionRule }
  | { type: 'mode'; mode: PermissionMode }
  | { type: 'subcommandResults'; reasons: Map<string, PermissionResult> }
  | { type: 'permissionPromptTool'; permissionPromptToolName: string; toolResult: unknown }
  | { type: 'hook'; hookName: string; hookSource?: string; reason?: string }
  | { type: 'asyncAgent'; reason: string }
  | { type: 'sandboxOverride'; reason: 'excludedCommand' | 'dangerouslyDisableSandbox' }
  | { type: 'classifier'; classifier: string; reason: string }
  | { type: 'workingDir'; reason: string }
  | { type: 'safetyCheck'; reason: string; classifierApprovable: boolean }
  | { type: 'other'; reason: string }
```

### 权限检查流程

```text
hasPermissionsToUseToolInner() 内部流程：

Step 1: 前置检查（bypass 模式也执行）
  1a. 工具级 deny 规则 → 直接 deny
  1b. 工具级 ask 规则 → 返回 ask（除非 sandbox 自动允许）
  1c. tool.checkPermissions() → 工具特定权限逻辑
  1d. 工具返回 deny → 直接 deny
  1e. requiresUserInteraction() → 强制 ask
  1f. 内容级 ask 规则 → 强制 ask
  1g. safetyCheck → 强制 ask（bypass 免疫）

Step 2: 模式处理
  2a. bypassPermissions 模式 → allow
  2b. 工具级 allow 规则 → allow

Step 3: 结果转换
  passthrough → ask
  返回最终 decision
```

### 规则匹配逻辑

```text
toolMatchesRule() 匹配规则：

1. 无 ruleContent → 工具级匹配
   - rule.toolName === tool.nameForRuleMatch
   
2. MCP server 级匹配
   - ruleInfo.serverName === toolInfo.serverName
   - ruleInfo.toolName === undefined 或 '*'
   
3. 规则示例：
   - "Bash" → 匹配整个 BashTool
   - "Bash(git status)" → 匹配特定命令
   - "mcp__server1" → 匹配 server1 的所有工具
   - "mcp__server1__*" → 同上（通配符）
```

## L3 - 实现视角

（待 L2 完成后填充）

## Review 历史

### 2026-04-19 - L1 Review

#### 权限模式列表验证
笔记中权限模式列表正确：
- `EXTERNAL_PERMISSION_MODES`: acceptEdits, bypassPermissions, default, dontAsk, plan ✅
- `INTERNAL_PERMISSION_MODES`: 上述 + auto（实验性） ✅

#### 函数签名验证
- `hasPermissionsToUseTool` 参数签名正确 ✅
- `ToolPermissionContext` 字段列表正确 ✅

#### 其余内容验证
- 规则结构 `ToolName(ruleContent)` ✅
- 三层规则体系 ✅
- deny 优先原则 ✅

## 疑问与待查

- [ ] `hasPermissionsToUseToolInner` 的完整流程？
- [ ] classifier 的触发条件和决策逻辑？
- [ ] `applyPermissionUpdate` 如何更新规则？
- [ ] safetyCheck 的具体检查内容？

## 相关模块

- [[tool]]
- [[hook]]
- [[query]]
- [[MCP]]
- [[REPL]]