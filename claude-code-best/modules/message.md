# message

> 项目: [[claude-code-best]]
> 文件: src/types/message.ts, src/utils/messages.ts
> 状态: L3-complete

## L1 - 黑盒视角

### 职责

**消息类型系统**。定义对话中所有消息的类型（User、Assistant、System、Attachment、Progress 等），以及消息的创建、规范化、查询等工具函数。

核心设计：**联合类型 + 类型守卫 + 工厂函数**，实现"类型安全、统一结构、便捷构造"的消息管理。

### 输入

**类型定义**（静态，无输入）:
- 定义 `Message` 基类型及其子类型

**工厂函数参数**:
- `createUserMessage`: `content`, `isMeta`, `toolUseResult`, `origin`, `uuid` 等
- `createAssistantMessage`: `content`, `usage`, `isVirtual`
- `createProgressMessage`: `toolUseID`, `parentToolUseID`, `data`

### 输出

**类型定义**:
- `Message` — 基类型，所有消息的联合
- `UserMessage` — 用户消息
- `AssistantMessage` — AI 回复消息
- `SystemMessage` — 系统消息（多种 subtype）
- `AttachmentMessage` — 附件消息
- `ProgressMessage` — 进度消息
- `GroupedToolUseMessage` — 工具调用分组
- `CollapsedReadSearchGroup` — 折叠的 read/search 组

**工厂函数输出**:
- 返回对应类型的消息对象（带 `uuid`, `timestamp`, `type` 等）

### 调用时机

1. **对话构建** — 每轮对话都需要创建 UserMessage 和 AssistantMessage
2. **工具执行** — 工具调用创建 tool_use block，结果创建 tool_result
3. **压缩后恢复** — 创建 AttachmentMessage 恢复文件/skills
4. **进度显示** — 创建 ProgressMessage 通知 UI
5. **系统事件** — 创建各类 SystemMessage（compact boundary、error 等）

### 调用方

- [[query]] — 主循环创建消息、处理流式输出
- [[compact]] — 压缩后创建摘要消息、边界标记
- [[Tool]] — 工具执行创建 tool_result
- [[REPL]] — UI 显示消息、用户输入处理
- [[QueryEngine]] — 管理对话状态、消息队列

### 黑盒调用记录

**类型导入**:
- `ContentBlockParam`, `ContentBlock` → 来自 `@anthropic-ai/sdk`，API 的 block 类型
- `BetaUsage` → 来自 SDK，API 使用统计

**工具函数**:
- `randomUUID()` → 返回 UUID，生成唯一消息 ID
- `normalizeMessages()` → 返回 NormalizedMessage[]，规范化消息列表
- `getLastAssistantMessage()` → 返回 AssistantMessage，获取最后一条 AI 回复
- `isNotEmptyMessage()` → 返回 boolean，判断消息是否有内容
- `isToolUseRequestMessage()` → 返回 boolean，判断是否为工具调用请求
- `isToolUseResultMessage()` → 返回 boolean，判断是否为工具结果
- `createCompactBoundaryMessage()` → 返回 SystemCompactBoundaryMessage，创建压缩边界

**消息判断**:
- `isSyntheticMessage()` → 返回 boolean，判断是否为合成消息
- `hasToolCallsInLastAssistantTurn()` → 返回 boolean，判断最后是否有工具调用

### 消息类型概览

| 类型                         | 说明    | 关键字段                                                        |
| -------------------------- | ----- | ----------------------------------------------------------- |
| `Message`                  | 基类型   | `type`, `uuid`, `isMeta`, `message`                         |
| `UserMessage`              | 用户输入  | `type: 'user'`, `message.content`, `imagePasteIds`          |
| `AssistantMessage`         | AI 回复 | `type: 'assistant'`, `message.content`, `message.usage`     |
| `SystemMessage`            | 系统消息  | `type: 'system'`, 多种 subtype                                |
| `AttachmentMessage`        | 附件    | `type: 'attachment'`, `attachment`                          |
| `ProgressMessage`          | 进度    | `type: 'progress'`, `data`                                  |
| `GroupedToolUseMessage`    | 工具分组  | `type: 'grouped_tool_use'`, `toolName`, `messages`          |
| `CollapsedReadSearchGroup` | 折叠组   | `type: 'collapsed_read_search'`, `readCount`, `searchCount` |

### 关键设计决策

**1. 联合类型结构**

所有消息类型都扩展 `Message` 基类型，通过 `type` 字段区分：
- `type: 'user'` → UserMessage
- `type: 'assistant'` → AssistantMessage
- `type: 'system'` → SystemMessage（含多种 subtype）

**2. 类型守卫模式**

通过 `if (message.type === 'user')` 收窄类型，TypeScript 自动推断为 UserMessage。

**3. 消息内容结构**

`message.content` 可以是：
- `string` — 简单文本
- `ContentBlockParam[]` — block 数组（text, tool_use, tool_result, thinking 等）

**4. 消息生命周期**

消息从创建 → 入队列 → 发送 API → 流式返回 → 存储 → 压缩 → 恢复，贯穿整个对话流程。

**5. 扩展性设计**

`[key: string]: unknown` 允许动态添加字段，适应不同场景（hook 结果、压缩元数据等）。

## L2 - 接口视角

### 核心函数

| 函数名                                 | 作用         | 关键参数                                                   | 返回类型                           |            |
| ----------------------------------- | ---------- | ------------------------------------------------------ | ------------------------------ | ---------- |
| `createUserMessage()`               | 创建用户消息     | `content`, `isMeta`, `toolUseResult`, `origin`, `uuid` | `UserMessage`                  |            |
| `createAssistantMessage()`          | 创建 AI 消息   | `content`, `usage`, `isVirtual`                        | `AssistantMessage`             |            |
| `createProgressMessage()`           | 创建进度消息     | `toolUseID`, `parentToolUseID`, `data`                 | `ProgressMessage<P>`           |            |
| `createCompactBoundaryMessage()`    | 创建压缩边界     | `trigger`, `preTokens`, `lastPreCompactMessageUuid`    | `SystemCompactBoundaryMessage` |            |
| `normalizeMessages()`               | 规范化消息列表    | `messages: Message[]`                                  | `NormalizedMessage[]`          |            |
| `getLastAssistantMessage()`         | 获取最后 AI 消息 | `messages`                                             | `AssistantMessage              | undefined` |
| `isNotEmptyMessage()`               | 判断消息有内容    | `message`                                              | `boolean`                      |            |
| `isToolUseRequestMessage()`         | 判断工具调用请求   | `message`                                              | `boolean`（类型守卫）                |            |
| `isToolUseResultMessage()`          | 判断工具结果     | `message`                                              | `boolean`（类型守卫）                |            |
| `isSyntheticMessage()`              | 判断合成消息     | `message`                                              | `boolean`                      |            |
| `hasToolCallsInLastAssistantTurn()` | 判断最后有工具调用  | `messages`                                             | `boolean`                      |            |
| `deriveUUID()`                      | 派生 UUID    | `parentUUID`, `index`                                  | `UUID`                         |            |
| `deriveShortMessageId()`            | 生成短消息 ID   | `uuid`                                                 | `string`                       |            |
| `mergeUserMessages()`               | 合并用户消息     | `a`, `b`                                               | `UserMessage`                  |            |

### 核心类型

```typescript
// MessageType - 消息类型枚举
export type MessageType = 
  | 'user' 
  | 'assistant' 
  | 'system' 
  | 'attachment' 
  | 'progress' 
  | 'grouped_tool_use' 
  | 'collapsed_read_search'

// Message - 基类型（所有消息的联合）
export type Message = {
  type: MessageType
  uuid: UUID
  isMeta?: boolean              // 是否为元消息（不发送给 API）
  isCompactSummary?: boolean    // 是否为压缩摘要
  toolUseResult?: unknown       // 工具执行结果
  isVisibleInTranscriptOnly?: boolean
  attachment?: { type: string; toolUseID?: string; [key: string]: unknown }
  message?: {
    role?: string
    id?: string
    content?: MessageContent
    usage?: BetaUsage | Record<string, unknown>
    [key: string]: unknown
  }
  [key: string]: unknown        // 扩展字段
}

// UserMessage - 用户消息
export type UserMessage = Message & {
  type: 'user'
  message: NonNullable<Message['message']>
  imagePasteIds?: number[]      // 粘贴的图片 ID
}

// AssistantMessage - AI 回复消息
export type AssistantMessage = Message & {
  type: 'assistant'
  message: NonNullable<Message['message']>
}

// SystemMessage - 系统消息（多种 subtype）
export type SystemMessage = Message & { type: 'system' }
export type SystemCompactBoundaryMessage = Message & {
  type: 'system'
  compactMetadata: {
    preservedSegment?: {
      headUuid: UUID
      tailUuid: UUID
      anchorUuid: UUID
    }
  }
}

// AttachmentMessage - 附件消息
export type AttachmentMessage<T = { type: string }> = Message & {
  type: 'attachment'
  attachment: T
}

// ProgressMessage - 进度消息
export type ProgressMessage<T = unknown> = Message & {
  type: 'progress'
  data: T
}

// ContentItem - 内容元素（单个 block）
export type ContentItem = ContentBlockParam | ContentBlock

// MessageContent - 消息内容（string 或 block 数组）
export type MessageContent = string | ContentBlockParam[] | ContentBlock[]

// NormalizedMessage - 规范化消息（每个消息只有一个 content block）
export type NormalizedMessage = Message
export type NormalizedUserMessage = UserMessage
export type NormalizedAssistantMessage = AssistantMessage

// GroupedToolUseMessage - 工具调用分组
export type GroupedToolUseMessage = Message & {
  type: 'grouped_tool_use'
  toolName: string
  messages: NormalizedAssistantMessage[]
  results: NormalizedUserMessage[]
  displayMessage: NormalizedAssistantMessage | NormalizedUserMessage
}

// CollapsedReadSearchGroup - 折叠的 read/search 组
export type CollapsedReadSearchGroup = {
  type: 'collapsed_read_search'
  uuid: UUID
  searchCount: number
  readCount: number
  readFilePaths: string[]
  searchArgs: string[]
  messages: CollapsibleMessage[]
  displayMessage: CollapsibleMessage
}
```

### 函数签名详解

**类型守卫函数**（返回 `message is X` 类型）：

```typescript
// isToolUseRequestMessage - 判断是否为工具调用请求
export function isToolUseRequestMessage(
  message: Message,
): message is ToolUseRequestMessage  // 类型守卫返回类型

// isToolUseResultMessage - 判断是否为工具结果
export function isToolUseResultMessage(
  message: Message,
): message is ToolUseResultMessage

// getLastAssistantMessage - 从末尾查找最后一条 assistant 消息
export function getLastAssistantMessage(
  messages: Message[],
): AssistantMessage | undefined
```

## L3 - 实现视角

### 关键代码路径

**createUserMessage 实现**:

```text
createUserMessage({ content, isMeta, toolUseResult, ... }) {
  const m: UserMessage = {
    type: 'user',
    message: {
      role: 'user',
      content: content || NO_CONTENT_MESSAGE  // 确保不发送空消息
    },
    isMeta,
    isVisibleInTranscriptOnly,
    isVirtual,
    isCompactSummary,
    summarizeMetadata,
    uuid: uuid || randomUUID(),               // 自动生成或使用指定
    timestamp: timestamp || new Date().toISOString(),
    toolUseResult,
    mcpMeta,
    imagePasteIds,
    sourceToolAssistantUUID,
    permissionMode,
    origin,
  }
  return m
}
```

**createAssistantMessage 实现**:

```text
createAssistantMessage({ content, usage, isVirtual }) {
  return baseCreateAssistantMessage({
    content: typeof content === 'string'
      ? [{ type: 'text', text: content === '' ? NO_CONTENT_MESSAGE : content }]
      : content,
    usage,
    isVirtual,
  })
}

baseCreateAssistantMessage({ content, usage, ... }) {
  return {
    type: 'assistant',
    message: {
      id: randomUUID(),
      role: 'assistant',
      stop_reason: 'stop_sequence',
      content: content as ContentBlock[],
      usage,
    },
    uuid: randomUUID(),
    timestamp: new Date().toISOString(),
    ...
  }
}
```

**normalizeMessages 实现**:

```text
normalizeMessages(messages: Message[]): NormalizedMessage[] {
  let isNewChain = false
  
  return messages.flatMap(message => {
    switch (message.type) {
      case 'assistant': {
        // 将多 block 的 assistant 消息拆分成单 block 消息
        const content = Array.isArray(message.message.content) 
          ? message.message.content 
          : []
        isNewChain = isNewChain || content.length > 1
        
        return content.map((block, index) => ({
          type: 'assistant',
          message: { ...message.message, content: [block] },
          uuid: isNewChain ? deriveUUID(message.uuid, index) : message.uuid,
          ...
        }))
      }
      
      case 'user': {
        // 将 string content 转换为 [{ type: 'text', text: ... }]
        if (typeof message.message.content === 'string') {
          return [{
            ...message,
            message: {
              ...message.message,
              content: [{ type: 'text', text: message.message.content }]
            },
            uuid: isNewChain ? deriveUUID(message.uuid, 0) : message.uuid,
          }]
        }
        
        // 处理多 block user 消息（包括图片）
        isNewChain = isNewChain || (message.message.content?.length ?? 0) > 1
        let imageIndex = 0
        
        return (message.message.content ?? []).map((block, index) => {
          const isImage = block.type === 'image'
          const imageId = isImage && message.imagePasteIds
            ? message.imagePasteIds[imageIndex]
            : undefined
          if (isImage) imageIndex++
          
          return {
            ...createUserMessage({ content: [block], ... }),
            uuid: isNewChain ? deriveUUID(message.uuid, index) : message.uuid,
            imagePasteIds: imageId !== undefined ? [imageId] : undefined,
          }
        })
      }
      
      case 'attachment':  return [message]  // 保持原样
      case 'progress':    return [message]
      case 'system':      return [message]
    }
  })
}
```

**deriveUUID 实现**（确定性 UUID 派生）:

```text
deriveUUID(parentUUID: UUID, index: number): UUID {
  const hex = index.toString(16).padStart(12, '0')
  return `${parentUUID.slice(0, 24)}${hex}` as UUID
}

// 示例：
// parentUUID = 'abc-123-def-456'
// index = 0 → 'abc-123-def-456000000000000'
// index = 1 → 'abc-123-def-456000000000001'
```

**reorderMessagesInUI 实现**（UI 显示重排序）:

```text
reorderMessagesInUI(messages, syntheticStreamingToolUseMessages) {
  // 1. 按 tool use ID 分组
  const toolUseGroups = new Map<string, {
    toolUse: ToolUseRequestMessage | null
    preHooks: AttachmentMessage[]
    toolResult: NormalizedUserMessage | null
    postHooks: AttachmentMessage[]
  }>()
  
  // 2. 遍历消息，填充分组
  for (const message of messages) {
    if (isToolUseRequestMessage(message)) {
      const toolUseID = message.message.content[0]?.id
      toolUseGroups.get(toolUseID)!.toolUse = message
    }
    
    if (isHookAttachmentMessage(message) && message.attachment.hookEvent === 'PreToolUse') {
      toolUseGroups.get(toolUseID)!.preHooks.push(message)
    }
    
    // ... 类似处理 toolResult 和 postHooks
  }
  
  // 3. 按顺序构建结果：preHooks → toolUse → toolResult → postHooks
  return reorderedMessages
}
```

### 边界条件

**1. 空内容处理**:
- `content === ''` → 替换为 `NO_CONTENT_MESSAGE`
- `content === undefined` → 替换为 `NO_CONTENT_MESSAGE`

**2. UUID 派生规则**:
- 单 block 消息 → 保持原 UUID
- 多 block 消息（index > 0）→ 派生新 UUID
- `isNewChain` 标记：一旦遇到多 block，后续所有消息都需要派生

**3. 图片消息处理**:
- UserMessage 中的 `imagePasteIds` 数组对应图片 block
- 规范化时按顺序提取 `imagePasteIds[index]`

**4. 合成消息判断**:
- `isSyntheticMessage()` 检查 content 是否匹配特定常量（`INTERRUPT_MESSAGE`, `CANCEL_MESSAGE` 等）
- 合成消息不发给 API，只用于 UI 显示

**5. Hook 附件消息**:
- `AttachmentMessage` 的 `attachment.hookEvent` 区分 `PreToolUse` 和 `PostToolUse`
- UI 重排序时按 hook 类型分组显示

**6. 工具结果匹配**:
- `isToolUseResultMessage()` 通过 `tool_result` block 或 `toolUseResult` 字段判断
- `sourceToolAssistantUUID` 记录对应的 tool_use 消息 UUID

### 性能考量

**1. 规范化惰性**:
- 单 block 消息保持原 UUID，避免不必要的派生
- `isNewChain` 标记延迟设置，减少 UUID 计算

**2. findLast 优化**:
- `getLastAssistantMessage()` 使用 `Array.findLast()` 从末尾查找
- 比 `filter + last` 更高效（早期退出）

**3. 短消息 ID**:
- `deriveShortMessageId()` 取 UUID 前 10 位转 base36，得 6 位字符串
- 用于 snip 工具引用，减少 token

**4. 消息合并**:
- `mergeUserMessages()` 合并相邻用户消息
- 用于 tool result 合并，减少消息数量

**5. 类型守卫性能**:
- 类型守卫函数（`isToolUseRequestMessage` 等）使用简单条件判断
- 避免复杂类型解析，保持高性能

**6. 扩展字段**:
- `[key: string]: unknown` 允许动态字段，但 TypeScript 无法优化
- 工厂函数显式设置已知字段，减少动态访问

## Review 历史

### 2026-04-19 - L1 Review

#### createAssistantMessage 参数
~~`createAssistantMessage`: `content`, `toolUseBlocks`, `thinkingBlocks`, `usage` 等~~
> ❌ **参数错误**: 源码只有三个参数。
> **正确参数**: `content`, `usage`, `isVirtual`

#### createProgressMessage 参数
~~`createProgressMessage`: `data`, `uuid`, `timestamp`~~
> ❌ **参数错误**: 源码参数是工具调用 ID 相关。
> **正确参数**: `toolUseID`, `parentToolUseID`, `data`

### 2026-04-19 - L2 Review

#### CollapsedReadSearchGroup 类型定位
~~`CollapsedReadSearchGroup` — 折叠的 read/search 组（在类型定义列表中）~~
> ⚠️ **补充说明**: 源码显示该类型**不继承 `Message`**，是独立结构。
> **实际定位**: `CollapsedReadSearchGroup` 是 UI 渲染专用类型，被包含在 `RenderableMessage` 联合类型中，但不属于 Message 子类型体系。

#### ProgressMessage 类型定义
笔记中只写了 `type: 'progress'`, `data: T`，但工厂函数创建的对象还包含 `toolUseID` 和 `parentToolUseID`。
> ⚠️ **补充说明**: 类型定义只有核心字段，工厂函数会额外注入 `uuid`, `timestamp`, `toolUseID`, `parentToolUseID` 等字段（通过 `[key: string]: unknown` 扩展）。
> ❌ **参数错误**: 源码参数是工具调用 ID 相关。
> **正确参数**: `toolUseID`, `parentToolUseID`, `data`

## 疑问与待查

- [ ] `ContentBlockParam` 和 `ContentBlock` 的区别？
- [ ] `normalizeMessages()` 的具体逻辑？
- [ ] `CollapsedReadSearchGroup` 如何折叠多条消息？
- [ ] SystemMessage 的各种 subtype 如何区分？

## 相关模块

- [[query]]
- [[compact]]
- [[Tool]]
- [[REPL]]
- [[QueryEngine]]