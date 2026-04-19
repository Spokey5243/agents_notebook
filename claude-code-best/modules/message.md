# message

> 项目: [[claude-code-best]]
> 文件: src/types/message.ts, src/utils/messages.ts
> 状态: L1-complete

## L1 - 黑盒视角

### 职责

**消息类型系统**。定义对话中所有消息的类型（User、Assistant、System、Attachment、Progress 等），以及消息的创建、规范化、查询等工具函数。

核心设计：**联合类型 + 类型守卫 + 工厂函数**，实现"类型安全、统一结构、便捷构造"的消息管理。

### 输入

**类型定义**（静态，无输入）:
- 定义 `Message` 基类型及其子类型

**工厂函数参数**:
- `createUserMessage`: `content`, `isMeta`, `toolUseResult`, `origin`, `uuid` 等
- `createAssistantMessage`: `content`, `toolUseBlocks`, `thinkingBlocks`, `usage` 等
- `createProgressMessage`: `data`, `uuid`, `timestamp`

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

| 类型 | 说明 | 关键字段 |
|------|------|----------|
| `Message` | 基类型 | `type`, `uuid`, `isMeta`, `message` |
| `UserMessage` | 用户输入 | `type: 'user'`, `message.content`, `imagePasteIds` |
| `AssistantMessage` | AI 回复 | `type: 'assistant'`, `message.content`, `message.usage` |
| `SystemMessage` | 系统消息 | `type: 'system'`, 多种 subtype |
| `AttachmentMessage` | 附件 | `type: 'attachment'`, `attachment` |
| `ProgressMessage` | 进度 | `type: 'progress'`, `data` |
| `GroupedToolUseMessage` | 工具分组 | `type: 'grouped_tool_use'`, `toolName`, `messages` |
| `CollapsedReadSearchGroup` | 折叠组 | `type: 'collapsed_read_search'`, `readCount`, `searchCount` |

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

（待 L1 review 完成后填充）

## L3 - 实现视角

（待 L2 完成后填充）

## Review 历史

（待 review）

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