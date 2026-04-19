# Agent 源码阅读笔记仓库

系统性阅读和记录主流 Agent 项目源码，理解架构设计和实现细节。

## 仓库结构

```
agents_notebook/
├── {项目名}/                    # 每个项目的笔记
│   ├── progress.md              # 阅读进度追踪
│   ├── summary.md               # 项目总结（万字级别）
│   └── modules/
│       ├── query.md             # 分层笔记
│       ├── tool.md
│       └── ...
├── skill/                       # 使用的 skill 定义
│   └── code-reader.md           # 源码阅读工作流 skill
├── docs/                        # superpowers 生成的文档（已 gitignore）
│   └── superpowers/
│       ├── plans/
│       └── specs/
└── cross-analysis/              # 跨项目对比（待创建）
    ├── context-management.md
    ├── tool-orchestration.md
    └── ...
```

## 阅读方法

### 分层笔记（L1/L2/L3）

| 层级 | 视角 | 内容 |
|------|------|------|
| L1 | 黑盒视角 | 职责、输入输出、调用时机、调用方、黑盒调用记录 |
| L2 | 接口视角 | 核心函数表格、核心类型定义 |
| L3 | 实现视角 | 关键代码路径、边界条件、性能考量 |

### AI Review 验证

每层完成后进行验证：
1. Git commit（pre-review）
2. 回到源码验证每个结论
3. 发现错误时标记修正（划掉原结论 + 原因 + 新结论）
4. Git commit（post-review）

### 使用的 Skill

详见 [skill/code-reader.md](skill/code-reader.md)

## 已完成项目

| 项目 | 状态 | 完成模块 |
|------|------|----------|
| claude-code-best | 进行中 | 8 模块（query, tool, permission, hook, REPL, message, compact, abortController） |

## 计划项目

- openclaw
- hermes
- deerflow
- codex-cli

## 笔记特点

- **分层渐进**: 从黑盒到接口再到实现，逐层深入
- **AI Review**: 每层验证，避免错误理解累积
- **设计导向**: 关注架构设计，而非实现细节
- **跨项目对比**: 最终形成跨项目的架构对比分析

## Git 提交规范

```
pre-review: {模块名} {层级}
post-review: {模块名} {层级} - {修正数量} 处修正 / all verified
```

---

**仓库创建时间**: 2026-04-19