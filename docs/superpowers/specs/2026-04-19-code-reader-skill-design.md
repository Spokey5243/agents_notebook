---
name: Code Reader Skill 设计文档
description: 通用源码阅读工作流 skill，支持分层笔记、AI review、跨项目对比
---

# Code Reader Skill 设计文档

## Context

用户正在学习 harness management（agent 运行环境管理），需要深入阅读多个开源项目源码（claude-code、openclaw、codex 等），对比架构设计差异。用户是 Python 背景，首次尝试读 TypeScript 源码，遇到调用链深度迷失、耦合功能交叉引用困难等问题。

**目标**: 创建一个可复用的源码阅读 skill，帮助用户：
1. 系统化理解复杂项目架构
2. 建立可追溯的学习笔记（支持 review + error tracking）
3. 最终形成跨项目对比分析

---

## 设计决策总结

| 维度          | 决策                                                                               |
| ----------- | -------------------------------------------------------------------------------- |
| Skill 名称    | `code-reader`                                                                    |
| Skill 结构    | 单一 skill `code-reader`，内部按阶段切换（初始化→阅读→review→总结→对比）                              |
| 笔记结构        | 分层（L1黑盒/L2接口/L3实现）+ 项目内总结 + 跨项目对比                                                |
| Obsidian 集成 | 优先使用 obsidian skills，fallback 到直接文件操作                                            |
| Review 机制   | 划掉原结论 + 记录原因 + 新结论；双向触发（AI建议 + 用户主动）                                             |
| 启动方式        | 检测当前目录 + 确认                                                                      |
| 笔记路径        | `E:\JourneyIntoAI\DL_and_LLM\expirements\notebooks\LLM notebook\agents_notebook` |

---

## 笔记目录结构

```
agents_notebook/
├── {项目名}/                    # 如 claude-code/
│   ├── progress.md              # 阅读进度追踪
│   ├── summary.md               # 项目总结（万字级别）
│   └── modules/
│       ├── query.md             # 分层笔记
│       ├── tool.md
│       ├── permission.md
│       └── ...
├── openclaw/
├── codex/
└── cross-analysis/              # 跨项目对比
    ├── context-management.md
    ├── tool-orchestration.md
    ├── permission-system.md
    └── ...
```

---

## 分层笔记模板

每个模块笔记使用以下结构：

```markdown
# 模块名

> 项目: [[claude-code]]
> 文件: src/query.ts
> 状态: L1-complete / L2-complete / L3-complete

## L1 - 黑盒视角（第一轮）

### 职责
一句话描述这个模块做什么。

### 输入
- 主要参数/依赖

### 输出
- 返回值/副作用

### 调用时机
何时被使用，被谁触发。

### 调用方
- [[模块A]]
- [[模块B]]

### 黑盒调用记录
阅读过程中遇到的"外部调用"，暂不深入：
- `Permission.check()` → 返回 bool，作用是检查权限
- `Compact.run()` → 返回压缩后的消息列表

## L2 - 接口视角（第二轮）

### 核心函数
| 函数名 | 作用 | 关键参数 |
|--------|------|----------|
| `query()` | 主入口 | params, messages |
| `queryLoop()` | 内部循环 | state |

### 核心类型
```typescript
// 关键类型定义摘录
type State = { messages: Message[], ... }
```

## L3 - 实现视角（第三轮）

### 关键代码路径
核心逻辑的伪代码/流程图：

```
query() {
  while(true) {
    1. 构造 messagesForQuery
    2. 调用 API
    3. 处理 tool_use
    4. 递归或退出
  }
}
```

### 边界条件
- 错误处理逻辑
- 特殊情况分支

### 性能考量
- 缓存策略
- 异步设计

## Review 历史

### 2026-04-19 - L1 Review

#### 职责
~~query 是 agent 的心跳函数，每秒调用一次~~
> ❌ **错误理解**: 看到 `while (true)` 就联想到定时器，忽略了 `yield` 的语义。
> **正确理解**: query 是用户输入触发的主循环，处理一个完整 turn，通过 yield 流式返回事件。

## 疑问与待查
- [ ] `yield*` 和 `yield` 的区别是什么？
- [ ] compact 的触发阈值是多少？

## 相关模块
- [[permission]]
- [[compact]]
- [[tool]]
```

---

## 子流程设计

### 1. 初始化流程 (`init-project`)

**触发**: `/code-reader` 启动时，或用户明确说"开始读新项目"

**步骤**:
1. 检测当前工作目录，尝试识别项目名（从 package.json、git remote、文件夹名）
2. 提示用户确认："检测到你在 {项目名} 目录，要开始/继续阅读吗？"
3. 检查笔记目录是否已存在：
   - 存在 → 读取 progress.md，询问"继续上次进度还是重新开始？"
   - 不存在 → 创建目录结构 + progress.md 初始模板
4. 询问用户："你想先读哪个模块？"（建议从核心入口文件开始）

**progress.md 模板**:

```markdown
# {项目名} 阅读进度

## 当前状态
- 正在读: `query` (L1)
- 下一步: 完成 query L1 → review → query L2

## 已完成模块
| 模块 | L1 | L2 | L3 | Review |
|------|----|----|----|----|
| query | ✅ | ⏳ | ❌ | ✅ 2026-04-19 |

## 计划模块
- tool
- permission
- compact
```

---

### 2. 阅读流程 (`read-module`)

**触发**: 用户说"开始读 {模块名}" 或 "继续"

**步骤**:
1. 用户指定模块名（如 `query`）
2. AI 扫描模块核心文件，生成 L1 内容：
   - 职责一句话
   - 输入/输出
   - 调用时机
   - 黑盒调用记录（遇到外部依赖时，只记录"调了什么、返回什么"）
3. 写入 `modules/{模块名}.md`
4. 更新 `progress.md`
5. **关键节点**: 完成 L1 后，AI 主动提示："query 的 L1 已完成，需要 review 吗？"

**L1 阅读技巧**（AI 内部遵循）:
- 遇到不熟悉的 TS 语法时，先解释语法再继续
- 遇到外部调用时，只记录"黑盒信息"，不深入
- 遇到不理解的地方，标记为"疑问与待查"，不强行解释

---

### 3. Review 流程 (`review-module`)

**触发**:
- AI 主动建议（L1/L2 完成后）
- 用户主动发起："帮我 review query"

**步骤**:
1. AI 读取 `modules/{模块名}.md`
2. 结合源码验证每个结论
3. 发现错误时：
   - 原结论用 `~~删除线~~`
   - 添加错误原因分析
   - 写入正确结论
4. 更新笔记文件
5. 更新 progress.md 的 Review 列

---

### 4. 项目总结流程 (`project-summary`)

**触发**: 用户说"这个项目读完了，开始写总结"

**步骤**:
1. AI 汇总所有模块笔记
2. 生成架构总览图（使用 Obsidian canvas 或 mermaid）
3. 帮助用户撰写 `summary.md`（用户主导内容，AI 辅助组织结构）
4. 询问用户："是否开始跨项目对比？"

---

### 5. 跨项目对比流程 (`cross-analyze`)

**触发**: 用户说"对比 {项目A} 和 {项目B} 的 {模块}"

**步骤**:
1. AI 读取两个项目的对应模块笔记
2. 生成对比表格
3. 写入 `cross-analysis/{主题}.md`

---

## Skill Frontmatter

```yaml
---
name: code-reader
description: 通用源码阅读工作流。支持分层笔记(L1/L2/L3)、AI review、进度追踪、跨项目对比。用于学习复杂项目架构。
argument-hint: "[项目名 or 模块名]"
---
```

---

## 实现要点

1. **笔记路径**: 默认 `E:\JourneyIntoAI\DL_and_LLM\expirements\notebooks\LLM notebook\agents_notebook`

2. **阅读重点筛选**: 学习目标是**设计思想**，而非实现细节。AI 在阅读时需主动甄别：
   - **重点关注**: 架构设计、模块职责划分、数据流转、核心抽象、设计决策背后的权衡
   - **跳过忽略**:
     - 日志输出、调试代码、注释中的内部链接
     - 针对特定模型/厂商的优化（如 Anthropic 专属的 prompt cache）
     - 特定场景的边缘处理（如实验性 feature flag 分支）
     - 内部员工专属功能（`process.env.USER_TYPE === 'ant'`）
   - **甄别信号**:
     - 代码中有 `feature('XXX')` 且该 feature 是实验性 → 跳过
     - 注释提到 "ant-only" / "internal" → 跳过
     - 函数名含 `log` / `debug` / `trace` → 跳过
     - 模块名含 `analytics` / `telemetry` / `sentry` → 跳过

3. **Obsidian 集成**:
   - 优先使用 `obsidian-cli` skill（如果已加载）
   - Fallback: 直接用 Write/Edit 工具
   - wikilink 格式: `[[模块名]]`

4. **TS 语法解释**: 遇到陌生语法时，先解释再继续阅读
   - 常见新手困惑点: `yield*`, `?.`, `??`, 泛型, 类型守卫

5. **Review 格式**: 严格遵循"划掉 + 原因 + 新结论"格式

6. **Git 管理**:
   - 每次 review **前**: 提交当前状态，commit message: `pre-review: {模块名} {层级}`
   - 每次 review **后**: 提交修改，commit message: `post-review: {模块名} {层级}`
   - 用户负责定期推送到远程仓库

---

## 验证方式

1. 启动 skill，检测项目目录是否正确识别
2. 创建笔记结构，检查 progress.md 是否生成
3. 完成 query 模块的 L1，检查笔记内容是否符合模板
4. 发起 review，检查错误标记格式是否正确
5. 检查 Obsidian wikilink 是否可在客户端正常跳转