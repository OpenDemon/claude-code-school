# 第 2 章：源码目录结构

> **本章目标**：了解 512,000 行代码是怎么组织的，找到你感兴趣的部分在哪里。

---

## 先用大白话理解

想象一个大型超市的仓库：

- **入口区（screens/）**：顾客进来看到的界面，收银台、服务台
- **核心运营（query.ts、QueryEngine.ts）**：整个超市的运营中枢，决定怎么接待顾客、处理订单
- **工具区（tools/）**：各种专业设备，扫码枪、收银机、叉车……
- **安保室（utils/bash/）**：监控系统，防止有人偷东西或做危险的事
- **仓储系统（services/compact/）**：货架管理，东西太多了就整理压缩
- **供应商对接（services/mcp/）**：和外部供应商（第三方工具）的对接系统
- **员工手册（constants/prompts.ts）**：写着所有员工（AI）的行为规范

---

## 顶层目录结构

```
src/
├── query.ts              ← 核心查询循环（1,728 行，最重要的文件）
├── QueryEngine.ts        ← 会话引擎（1,155 行）
├── Tool.ts               ← 工具接口定义（~400 行）
├── tools.ts              ← 工具注册表（~200 行）
├── context.ts            ← 上下文构建（190 行）
│
├── screens/              ← 终端 UI 界面
│   ├── REPL.tsx          ← 主界面（核心交互）
│   └── ...
│
├── tools/                ← 66+ 内置工具实现
│   ├── BashTool/         ← 命令执行工具（含安全模块）
│   ├── FileReadTool/     ← 文件读取
│   ├── FileEditTool/     ← 文件编辑
│   ├── GlobTool/         ← 文件搜索
│   ├── GrepTool/         ← 内容搜索
│   ├── AgentTool/        ← 子 Agent 调度
│   └── ...
│
├── services/             ← 核心服务
│   ├── api/
│   │   └── claude.ts     ← API 调用逻辑（3,419 行，最大的文件）
│   ├── compact/
│   │   └── compact.ts    ← 4 级压缩引擎（1,705 行）
│   └── mcp/              ← MCP 集成
│
├── utils/                ← 工具函数
│   ├── bash/
│   │   └── bashSecurity.ts ← Bash 安全验证（23 项检查）
│   ├── settings/         ← 配置管理
│   ├── hooks/            ← Hook 系统
│   └── swarm/            ← Swarm 多 Agent
│
├── coordinator/          ← 协调器模式（多 Agent）
├── memdir/               ← 记忆系统
├── skills/               ← 技能系统（斜杠命令）
├── buddy/                ← 宠物系统（彩蛋）
│
└── constants/
    └── prompts.ts        ← 系统提示词（AI 的行为规范）
```

---

## 最重要的 10 个文件

按重要性排序，这 10 个文件包含了 Claude Code 最核心的逻辑：

| 排名 | 文件 | 行数 | 为什么重要 |
|------|------|------|-----------|
| 1 | `src/query.ts` | 1,728 | **Agent 循环的心脏**，包含核心 `while(true)` 循环 |
| 2 | `src/services/api/claude.ts` | 3,419 | API 调用、流式处理、工具预执行 |
| 3 | `src/services/compact/compact.ts` | 1,705 | 4 级压缩引擎，解决上下文超限问题 |
| 4 | `src/QueryEngine.ts` | 1,155 | 会话管理，历史记录，多轮对话 |
| 5 | `src/utils/bash/bashSecurity.ts` | ~800 | Bash 安全验证，23 项检查 |
| 6 | `src/constants/prompts.ts` | ~600 | 系统提示词，AI 的行为规范 |
| 7 | `src/Tool.ts` | ~400 | 工具接口定义，所有工具的基础 |
| 8 | `src/coordinator/coordinator.ts` | ~500 | 多 Agent 协调器 |
| 9 | `src/memdir/` | — | 记忆系统，跨会话知识管理 |
| 10 | `src/hooks/` | — | Hook 系统，25 种扩展事件 |

---

## 按功能模块分类

### 用户界面层

`screens/REPL.tsx` 是用户看到的终端界面。它基于 React + Ink（一个把 React 用在终端里的框架）构建，支持流式输出、代码高亮、权限确认对话框等复杂 UI 元素。

### 核心循环层

`query.ts` 是整个系统的心脏。它的核心是一个 `while(true)` 循环，每次迭代：检查是否需要压缩上下文 → 调用 API → 处理响应 → 执行工具（如果有）→ 判断是否继续。

`QueryEngine.ts` 是循环的管理者。它维护对话历史、管理会话状态、处理并发请求、决定何时触发压缩。

### 工具层

`tools/` 目录下有 66+ 个工具，每个工具都是一个独立的模块，实现同一套接口（`Tool.ts` 定义）。工具分为几类：

- **文件操作**：FileRead、FileEdit、FileWrite、FileDelete
- **搜索**：Grep（内容搜索）、Glob（文件名搜索）
- **命令执行**：BashTool（含完整安全系统）
- **Agent 调度**：AgentTool（启动子 Agent）
- **MCP 桥接**：动态加载第三方工具

### 安全层

`utils/bash/bashSecurity.ts` 是整个安全系统中最复杂的部分。它使用 tree-sitter 把 Bash 命令解析成抽象语法树（AST），然后对语法树进行 23 项安全检查，而不是简单的字符串匹配。

### 服务层

`services/api/claude.ts` 处理所有与 Anthropic API 的通信，包括流式响应处理、工具预执行、错误重试、Token 限制管理。

`services/compact/compact.ts` 实现 4 级压缩引擎，在上下文接近限制时自动压缩历史记录。

### 扩展层

`memdir/` 实现记忆系统，把用户偏好、项目规则、历史教训存储为结构化数据，支持语义召回。

`skills/` 实现技能系统，用户可以通过斜杠命令（如 `/commit`）调用预定义的工作流。

`hooks/` 实现 Hook 系统，允许用户在工具执行前后插入自定义逻辑。

---

## 代码规模感受

512,000 行代码是什么概念？

- 如果每天读 1,000 行，需要 512 天才能读完
- 最大的单个文件（`claude.ts`）有 3,419 行，相当于一本薄薄的技术书
- 整个系统有 1,884 个 TypeScript 文件

但这份教程不需要你读完所有代码。我们会聚焦在最关键的几个文件，理解核心设计思路。

---

> 下一章：[代理循环：AI 的心跳 →](docs/03-agentic-loop.md)
