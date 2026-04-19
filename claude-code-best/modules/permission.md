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

（待 L1 review 完成后填充）

## L3 - 实现视角

（待 L2 完成后填充）

## Review 历史

（待 review）

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