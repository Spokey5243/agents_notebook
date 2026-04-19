# compact

> 项目: [[claude-code-best]]
> 文件: src/services/compact/
> 状态: L1-complete

## L1 - 黑盒视角

### 职责

**上下文压缩系统**。当对话历史超过 token 阈值时，调用 LLM 生成摘要，替换旧消息以释放上下文空间，同时保留关键信息（文件状态、skills、plan 等）。

核心设计：**多策略压缩 + 压缩后恢复 + Hook 集成**，实现"自动触发、智能摘要、信息保留"的上下文管理。

### 输入

**compactConversation 参数**:
- `messages`: Message[] — 待压缩的对话历史
- `context`: ToolUseContext — 工具执行上下文
- `cacheSafeParams`: CacheSafeParams — fork context 参数（用于保留 cache）
- `suppressFollowUpQuestions`: boolean — 是否抑制后续问题
- `customInstructions`: string? — 自定义压缩指令
- `isAutoCompact`: boolean — 是否自动压缩
- `recompactionInfo`: RecompactionInfo? — 重压缩信息

**autoCompactIfNeeded 参数**:
- `messages`: Message[]
- `context`: ToolUseContext
- `cacheSafeParams`: CacheSafeParams
- `querySource`: QuerySource
- `tracking`: AutoCompactTrackingState | undefined
- `snipTokensFreed`: number — snip 释放的 token 数

### 输出

**CompactionResult**:
- `summaryMessages`: UserMessage[] — 摘要消息列表（用户消息数组）
- `attachments`: AttachmentMessage[] — 压缩后重新注入的附件
- `hookResults`: HookResultMessage[] — hook 结果消息
- `messagesToKeep`: Message[]? — 需保留的消息
- `boundaryMarker`: SystemCompactBoundaryMessage — 边界标记消息
- `preCompactTokenCount`: number — 压缩前 token 数
- `postCompactTokenCount`: number — 压缩后 token 数（含 input_tokens）
- `truePostCompactTokenCount`: number — 压缩后真实输出 token 数
- `compactionUsage`: APIUsage? — API 使用统计

### 调用时机

1. **自动触发** — `query.ts` 每轮循环检查 `shouldAutoCompact()`，超阈值时调用
2. **手动触发** — 用户执行 `/compact` 命令
3. **响应式触发** — API 返回 prompt-too-long 错误时尝试恢复（reactiveCompact，stub）
4. **Session Memory** — fork agent 使用 session memory compact

### 调用方

- [[query]] — 主循环中自动压缩和恢复
- [[commands]] — `/compact` 命令入口
- [[AgentTool]] — fork agent 使用 session memory compact

### 黑盒调用记录

**API 调用**:
- `streamCompactSummary()` → 返回 AssistantMessage，调用 LLM 生成摘要
- `getCompactPrompt()` → 返回 prompt string，生成压缩提示词

**Hook 系统**:
- `executePreCompactHooks()` → 返回 `{ newCustomInstructions, userDisplayMessage }`，执行压缩前 hook
- `executePostCompactHooks()` → 执行压缩后 hook

**Token 计算**:
- `tokenCountWithEstimation()` → 返回 token 数，估算消息 token
- `getEffectiveContextWindowSize()` → 返回有效窗口大小
- `getAutoCompactThreshold()` → 返回压缩阈值

**压缩后恢复**:
- `createPostCompactFileAttachments()` → 返回 AttachmentMessage[]，恢复文件状态
- `createSkillAttachmentIfNeeded()` → 返回 AttachmentMessage?，恢复 skills
- `createPlanAttachmentIfNeeded()` → 返回 AttachmentMessage?，恢复 plan

**其他**:
- `stripImagesFromMessages()` → 返回 Message[]，移除图片减少 token
- `runPostCompactCleanup()` → 执行压缩后清理
- `buildPostCompactMessages()` → 返回 Message[]，构建压缩后消息列表

### 子模块概览

| 模块                      | 状态     | 职责                       |
| ----------------------- | ------ | ------------------------ |
| compact.ts              | ✅ 完整   | 主压缩逻辑                    |
| autoCompact.ts          | ✅ 完整   | 自动压缩触发判断                 |
| microCompact.ts         | ✅ 完整   | 微压缩 tool result（时间/大小触发） |
| sessionMemoryCompact.ts | ✅ 完整   | session memory 压缩策略      |
| prompt.ts               | ✅ 完整   | 压缩提示词生成                  |
| grouping.ts             | ✅ 完整   | 消息按 API round 分组         |
| reactiveCompact.ts      | ❌ Stub | 响应式压缩（未实现）               |
| snipCompact.ts          | ❌ Stub | snip 压缩（未实现）             |
| cachedMicrocompact.ts   | ❌ Stub | 缓存微压缩（未实现）               |

### 关键常量

```typescript
AUTOCOMPACT_BUFFER_TOKENS = 13_000       // 自动压缩缓冲区
WARNING_THRESHOLD_BUFFER_TOKENS = 20_000 // 警告阈值缓冲区
ERROR_THRESHOLD_BUFFER_TOKENS = 20_000   // 错误阈值缓冲区
MANUAL_COMPACT_BUFFER_TOKENS = 3_000     // 手动压缩缓冲区
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000   // 摘要最大输出 token
POST_COMPACT_TOKEN_BUDGET = 50_000       // 压缩后恢复 token 预算
POST_COMPACT_MAX_FILES_TO_RESTORE = 5    // 最大恢复文件数
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3 // 连续失败最大次数（熔断）
```

### 关键设计决策

**1. 多策略分层**

- **autoCompact**: 主动检测，超阈值自动触发
- **microCompact**: 轻量级，只压缩 tool result
- **sessionMemoryCompact**: 保留最后 N 条消息，生成摘要
- **snipCompact**: 直接删除旧消息（stub，未实现）

**2. 压缩后恢复**

压缩后自动注入关键信息：
- 最近编辑的文件状态（最多 5 个）
- 已加载的 skills
- 当前 plan 状态
- Hook 结果

**3. 熔断机制**

`consecutiveFailures >= 3` 时停止尝试自动压缩，防止 API 调用浪费。

**4. prompt-too-long 重试**

压缩请求本身超限时，truncateHeadForPTLRetry() 删除最旧消息后重试（最多 2 次）。

## L2 - 接口视角

（待 L1 review 完成后填充）

## L3 - 实现视角

（待 L2 完成后填充）

## Review 历史

### 2026-04-19 - L1 Review

#### summaryMessages 类型
~~`summaryMessages`: Message[] — 摘要消息列表~~
> ⚠️ **类型简化**: 源码定义为 `UserMessage[]`，是 Message 的子类型。
> **实际类型**: `UserMessage[]`（用户消息数组，不含其他消息类型）

## 疑问与待查

- [ ] microCompact 的具体触发条件是什么？
- [ ] sessionMemoryCompact 和普通 compact 的区别？
- [ ] POST_COMPACT_TOKEN_BUDGET 如何分配给各类恢复内容？
- [ ] cachedMicrocompact（缓存编辑）的设计原理？

## 相关模块

- [[query]]
- [[Tool]]
- [[hook]]
- [[sessionMemory]]