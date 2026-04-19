# Claude Code 源码阅读笔记

> **源码来源**: https://github.com/claude-code-best/claude-code
>
> Claude Code 是 Anthropic 官方的 CLI Agent 工具，本项目通过逆向工程/decompiled 方式系统性阅读其核心模块。

## 项目概述

Claude Code 是一个成熟的 CLI Agent，支持：
- REPL 交互式对话
- 多工具并发执行
- 权限管理系统
- Hook 事件机制
- Agent 协作（多 Agent）
- 流式输出与中断控制

## 阅读进度

| 模块 | L1 | L2 | L3 | Review | 状态 |
|------|----|----|----|----|------|
| query | ✅ | ✅ | ✅ | ✅ | 完成 |
| tool | ✅ | ✅ | ✅ | ✅ | 完成 |
| permission | ✅ | ✅ | ✅ | ✅ | 完成 |
| hook | ✅ | ✅ | ✅ | ✅ | 完成 |
| REPL | ✅ | ✅ | ✅ | ✅ | 完成 |
| message | ✅ | ✅ | ✅ | ✅ | 完成 |
| compact | ✅ | ✅ | ✅ | ✅ | 完成 |
| abortController | ✅ | ✅ | ✅ | ✅ L1+L2 | 完成 |

## 核心架构

### 主循环流程

```
用户输入 → onSubmit → onQuery → query() → queryLoop()
                                              ↓
                                      API Streaming
                                              ↓
                                      Tool Execution
                                              ↓
                                      Permission + Hook
                                              ↓
                                      返回结果 → UI渲染
```

### 核心模块职责

| 模块 | 职责 |
|------|------|
| query | 主循环，处理 API streaming 和 tool execution |
| tool | 工具注册和执行，55+ 个工具 |
| permission | 权限检查，多模式（default/bypass/auto） |
| hook | 事件系统，28 种事件类型 |
| REPL | UI 层，用户输入和消息渲染 |
| message | 消息类型系统，联合类型 + 类型守卫 |
| compact | 对话压缩，多种压缩策略 |
| abortController | 中断控制，支持 Ctrl+C 和父子级联 |

### 流式传输机制

```
API 返回 SSE 流 → SDK buffer 检查 → 增量事件 → 聚合 → AssistantMessage
```

关键技术：
- Transfer-Encoding: chunked
- text/event-stream
- WeakRef 内存安全

### 权限系统设计

```
三层规则：allow / deny / ask
优先级：deny > ask > allow
模式：default / acceptEdits / bypassPermissions / dontAsk / auto
```

### Hook 事件类型

| 类别 | 事件 |
|------|------|
| 工具 | PreToolUse, PostToolUse |
| LLM | Stop, PrePrompt |
| 权限 | PermissionPrompt, PermissionAccepted |
| Agent | PreAgentTask, PostAgentTask |

## 笔记目录结构

```
claude-code-best/
├── progress.md              # 进度追踪
├── modules/
│   ├── query.md             # 主循环
│   ├── tool.md              # 工具系统
│   ├── permission.md        # 权限管理
│   ├── hook.md              # Hook 事件
│   ├── REPL.md              # REPL 界面
│   ├── message.md           # 消息类型
│   ├── compact.md           # 压缩机制
│   └── abortController.md   # 中断控制
└── summary.md               # 待撰写
```

## 关键设计决策

### 1. Discriminated Union 而非继承

```typescript
type Message = { type: MessageType, uuid: UUID, ... }
type UserMessage = Message & { type: 'user', content: ... }
```

原因：API 返回 JSON，无运行时继承链。

### 2. 异步流式 + AbortController

```typescript
for await (const event of stream) { yield event }
if (signal.aborted) { return { reason: 'aborted_streaming' } }
```

原因：实时输出 + 可中断 + 并发工具执行。

### 3. Permission bypass 免疫

某些检查即使 bypass 模式也执行：
- deny 规则
- safetyCheck（敏感路径）

原因：用户显式配置优先于模式。

### 4. Hook 与中断分离

- AbortController：强制中断 LLM 输出
- Stop Hook：LLM 完成后处理

原因：Hook 不能中断正在进行的输出。

## 学习要点

### TypeScript 模式

| 模式 | 用途 |
|------|------|
| Discriminated Union | 消息类型、权限决策 |
| async generator | 流式输出 |
| WeakRef | 内存安全 |
| Ref-based state | 避免闭包 stale |

### Agent 架构模式

| 模式 | Claude Code 实现 |
|------|------------------|
| 主循环 | queryLoop + yield |
| 工具执行 | StreamingToolExecutor（并发） |
| 权限管理 | 三层规则 + AI classifier |
| 中断控制 | AbortController + 父子级联 |

## 下一步

- [ ] 撰写 summary.md（万字级别）
- [ ] QueryEngine 模块
- [ ] StreamingToolExecutor 模块
- [ ] forkedAgent 模块

---

**源码来源声明**: 本项目笔记基于 https://github.com/claude-code-best/claude-code 的逆向工程/decompiled 源码，用于学习和理解 Agent 架构设计。