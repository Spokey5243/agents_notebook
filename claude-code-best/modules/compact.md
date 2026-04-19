# compact

> 项目: [[claude-code-best]]
> 文件: src/services/compact/
> 状态: L3-complete

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

### 核心函数

| 函数名                               | 作用                   | 关键参数                                                        |
| --------------------------------- | -------------------- | ----------------------------------------------------------- |
| `compactConversation()`           | 主压缩入口，生成摘要           | `messages`, `context`, `cacheSafeParams`, `isAutoCompact`   |
| `autoCompactIfNeeded()`           | 自动压缩判断和执行            | `messages`, `toolUseContext`, `tracking`, `snipTokensFreed` |
| `shouldAutoCompact()`             | 判断是否需要自动压缩           | `messages`, `model`, `querySource`, `snipTokensFreed`       |
| `getAutoCompactThreshold()`       | 计算压缩阈值               | `model`                                                     |
| `getEffectiveContextWindowSize()` | 获取有效上下文窗口            | `model`                                                     |
| `groupMessagesByApiRound()`       | 按 API round 分组消息     | `messages`                                                  |
| `truncateHeadForPTLRetry()`       | prompt-too-long 截断重试 | `messages`, `ptlResponse`                                   |
| `buildPostCompactMessages()`      | 构建压缩后消息列表            | `result: CompactionResult`                                  |
| `stripImagesFromMessages()`       | 移除图片减少 token         | `messages`                                                  |
| `getCompactPrompt()`              | 获取压缩提示词              | `customInstructions?`                                       |
| `partialCompactConversation()`    | 部分压缩（pivot 位置）       | `allMessages`, `pivotIndex`, `direction`                    |

### 核心类型

```typescript
// CompactionResult - 压缩结果
export interface CompactionResult {
  boundaryMarker: SystemMessage              // 边界标记
  summaryMessages: UserMessage[]             // 摘要消息
  attachments: AttachmentMessage[]           // 恢复附件
  hookResults: HookResultMessage[]           // hook 结果
  messagesToKeep?: Message[]                 // 保留消息
  userDisplayMessage?: string                // 用户显示消息
  preCompactTokenCount?: number              // 压缩前 token
  postCompactTokenCount?: number             // 压缩后 token
  truePostCompactTokenCount?: number         // 真实输出 token
  compactionUsage?: ReturnType<typeof getTokenUsage>  // API 使用统计
}

// AutoCompactTrackingState - 自动压缩追踪状态
export type AutoCompactTrackingState = {
  compacted: boolean           // 本轮是否压缩
  turnCounter: number          // 对话轮次
  turnId: string               // 每轮唯一 ID
  consecutiveFailures?: number // 连续失败次数（熔断）
}

// RecompactionInfo - 重压缩信息
export type RecompactionInfo = {
  isRecompactionInChain: boolean       // 是否链内重压缩
  turnsSincePreviousCompact: number    // 上次压缩后轮次
  previousCompactTurnId?: string       // 上次压缩 turn ID
  autoCompactThreshold: number         // 压缩阈值
  querySource?: QuerySource            // 调用来源
}

// SessionMemoryCompactConfig - session memory 压缩配置
export type SessionMemoryCompactConfig = {
  minTokens: number            // 最小保留 token
  minTextBlockMessages: number // 最小保留消息数
  maxTokens: number            // 最大保留 token（硬上限）
}
```

## L3 - 实现视角

### 关键代码路径

**compactConversation 主流程**:

```text
compactConversation(messages, context, ...) {
  // 1. 前置检查
  if (messages.length === 0) throw Error
  
  preCompactTokenCount = tokenCountWithEstimation(messages)
  
  // 2. 执行 preCompact hooks
  hookResult = await executePreCompactHooks({ trigger, customInstructions }, ...)
  customInstructions = mergeHookInstructions(customInstructions, hookResult.newCustomInstructions)
  
  // 3. 构建 summary request
  compactPrompt = getCompactPrompt(customInstructions)
  summaryRequest = createUserMessage({ content: compactPrompt })
  
  // 4. 循环尝试生成摘要（处理 prompt-too-long）
  for (;;) {
    summaryResponse = await streamCompactSummary({ messages, summaryRequest, ... })
    summary = getAssistantMessageText(summaryResponse)
    
    if (!summary?.startsWith(PROMPT_TOO_LONG_ERROR_MESSAGE)) break
    
    // prompt-too-long: 截断最旧消息重试（最多 2 次）
    ptlAttempts++
    truncated = truncateHeadForPTLRetry(messages, summaryResponse)
    if (!truncated) throw Error  // 无法恢复
    messagesToSummarize = truncated
  }
  
  // 5. 创建附件（恢复文件、skills、plan）
  attachments = [
    ...createPostCompactFileAttachments(appState),
    ...createSkillAttachmentIfNeeded(appState),
    ...createPlanAttachmentIfNeeded(appState),
  ]
  
  // 6. 创建边界标记
  boundaryMarker = createCompactBoundaryMessage({ ... })
  
  // 7. 执行 postCompact hooks
  await executePostCompactHooks({ trigger, ... }, ...)
  
  // 8. 返回结果
  return {
    boundaryMarker,
    summaryMessages: [summaryRequest, createUserMessage({ content: summary })],
    attachments,
    hookResults,
    ...
  }
}
```

**autoCompactIfNeeded 主流程**:

```text
autoCompactIfNeeded(messages, toolUseContext, tracking, ...) {
  // 1. 检查禁用状态
  if (isEnvTruthy(DISABLE_COMPACT)) return { wasCompacted: false }
  
  // 2. 熔断检查
  if (tracking?.consecutiveFailures >= MAX_CONSECUTIVE_FAILURES) {
    return { wasCompacted: false }
  }
  
  // 3. 判断是否需要压缩
  shouldCompact = await shouldAutoCompact(messages, model, querySource, ...)
  if (!shouldCompact) return { wasCompacted: false }
  
  // 4. 构建 recompactionInfo
  recompactionInfo = {
    isRecompactionInChain: tracking?.compacted === true,
    turnsSincePreviousCompact: tracking?.turnCounter ?? -1,
    autoCompactThreshold: getAutoCompactThreshold(model),
    ...
  }
  
  // 5. 执行压缩
  result = await compactConversation(messages, context, ..., true, recompactionInfo)
  
  // 6. 后置清理
  await runPostCompactCleanup(...)
  
  // 7. 返回结果
  return { wasCompacted: true, compactionResult: result }
}
```

**shouldAutoCompact 判断逻辑**:

```text
shouldAutoCompact(messages, model, ...) {
  // 1. 检查禁用
  if (!isAutoCompactEnabled()) return false
  
  // 2. Context collapse 模式冲突检查（互斥）
  if (isContextCollapseEnabled()) return false
  
  // 3. Token 计算
  tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
  threshold = getAutoCompactThreshold(model)
  
  // 4. 判断是否超阈值
  { isAboveAutoCompactThreshold } = calculateTokenWarningState(tokenCount, model)
  return isAboveAutoCompactThreshold
}
```

### 边界条件

**1. 空消息处理**:
- `messages.length === 0` → 抛出 `ERROR_MESSAGE_NOT_ENOUGH_MESSAGES`

**2. prompt-too-long 恢复**:
- 压缩请求本身超限时，`truncateHeadForPTLRetry()` 截断最旧 API round
- 最大重试次数 `MAX_PTL_RETRIES = 2`
- 截断失败 → 抛出错误，用户被阻塞

**3. 熔断机制**:
- `consecutiveFailures >= 3` → 停止尝试自动压缩
- 防止无效 API 调用浪费（曾有 1279 个 session 连续失败 50+ 次）

**4. Context Collapse 互斥**:
- `isContextCollapseEnabled() === true` → 禁用 autoCompact
- collapse 是另一种上下文管理方式，两者不应同时触发

**5. 消息分组边界**:
- `groupMessagesByApiRound()` 以 assistant message id 变化为边界
- 保证每个 group 是 API-safe 的（tool_use 都有对应的 tool_result）

**6. 部分压缩方向**:
- `direction = 'from'`: 压缩 pivot 后的消息，保留之前（cache 有效）
- `direction = 'up_to'`: 压缩 pivot 前的消息，保留之后（cache 无效）

### 性能考量

**1. Prompt Cache Sharing**:
- `tengu_compact_cache_prefix` feature gate 控制是否共享 cache
- Fork agent 路径复用父对话的 prompt cache（默认启用）
- 98% 的不共享路径是 cache miss，成本高

**2. 异步流式处理**:
- `streamCompactSummary()` 使用流式 API 调用
- 通过 `setStreamMode('requesting')` 显示进度
- `context.onCompactProgress` 回调通知 UI

**3. Token 预算分配**:
- `POST_COMPACT_TOKEN_BUDGET = 50_000`
  - `POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000`（单文件上限）
  - `POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000`（skills 总预算）
  - `POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000`（单 skill 上限）
- 每类恢复内容有独立预算，防止单类过大

**4. MicroCompact 轻量压缩**:
- 只压缩特定 tool result（FILE_READ, SHELL, GLOB 等）
- 时间触发：`TIME_BASED_MC_CLEARED_MESSAGE` 清除旧内容
- 大小触发：超过阈值时截断

**5. SessionMemoryCompact 特化**:
- 保留最后 N 条消息 + session memory 摘要
- 配置：`minTokens: 10_000`, `minTextBlockMessages: 5`, `maxTokens: 40_000`
- 用于 fork agent 的上下文管理

**6. 图片移除优化**:
- `stripImagesFromMessages()` 将 `[image]` 替换为文本标记 `[image]`
- 减少 token 数，防止压缩请求本身超限

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