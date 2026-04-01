# Claude Code School 🏫

> **从零到一，深度解析 Claude Code 的内部架构**
>
> 面向零基础新手的 Claude Code 源码学习教程，先用大白话讲清楚概念，再带你看真实的代码实现。

[![在线阅读](https://img.shields.io/badge/在线阅读-教程网站-blue?style=flat-square)](https://claude-code-tutorial.manus.space)
[![源码参考](https://img.shields.io/badge/源码参考-sanbuphy/claude--code--source--code-orange?style=flat-square)](https://github.com/sanbuphy/claude-code-source-code)
[![License: MIT](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)

---

## 这个教程是什么？

2026 年 3 月 31 日，由于 CI/CD 流水线的一个配置失误，Anthropic 意外将 Claude Code 的完整 TypeScript 源码（59.8 MB 的 Source Map 文件）发布到了 npm 上。这份源码包含 **1,884 个源文件、394,222 行代码**，是迄今为止最完整的商业级 AI Agent 源码泄露事件。

本教程基于这份源码，从架构设计的角度，带你理解 Claude Code 是如何工作的。**不需要你是编程专家**——我们会先用生活化的比喻讲清楚每个概念，再逐步引入代码细节。

---

## 目录

- [第一章：Claude Code 是什么](#第一章claude-code-是什么)
- [第二章：整体架构一览](#第二章整体架构一览)
- [第三章：源码目录结构](#第三章源码目录结构)
- [第四章：代理循环——Claude Code 的心跳](#第四章代理循环claude-code-的心跳)
- [第五章：上下文加载——Claude 的"工作记忆"](#第五章上下文加载claude-的工作记忆)
- [第六章：工具执行模型——14 步流水线](#第六章工具执行模型14-步流水线)
- [第七章：对话引擎——query.ts 深度解析](#第七章对话引擎querysts-深度解析)
- [第八章：三层上下文压缩系统](#第八章三层上下文压缩系统)
- [第九章：工具系统与权限安全](#第九章工具系统与权限安全)
- [第十章：多 Agent 协作](#第十章多-agent-协作)
- [第十一章：MCP 集成](#第十一章mcp-集成)
- [第十二章：10 种运行模式](#第十二章10-种运行模式)
- [第十三章：为什么 Claude Code 这么强](#第十三章为什么-claude-code-这么强)
- [第十四章：关键模块导读](#第十四章关键模块导读)
- [延伸阅读](#延伸阅读)

---

## 第一章：Claude Code 是什么

### 先用大白话理解

你有没有用过 ChatGPT 或者 Claude？你问它一个问题，它给你一个答案，然后对话就结束了。它只能**说话**，不能帮你真正**做事**。

Claude Code 不一样。它不只是一个聊天机器人，它是一个**会用工具的 AI 助手**。就像一个真正的程序员坐在你旁边：

- 你说"帮我修复这个 bug"，它会**打开文件**、**读取代码**、**修改内容**、**运行测试**，然后告诉你结果
- 你说"帮我搭建一个网站"，它会**创建文件夹**、**写代码**、**安装依赖**、**启动服务器**

这种"能真正做事"的 AI，在技术上叫做 **AI Agent（人工智能代理）**。

### 技术定义

Claude Code 是 Anthropic 开发的一款**终端 AI 编程助手**，其核心特征是：

- **工具调用能力**：可以执行 Bash 命令、读写文件、搜索代码、访问网络
- **代理循环架构**：通过"感知→思考→行动→观察"的循环持续工作，直到任务完成
- **上下文感知**：自动读取项目文件、Git 历史、CLAUDE.md 配置，理解当前工作环境
- **多 Agent 协作**：支持多个 Claude 实例并行工作，分工完成复杂任务

### 代码规模

| 维度 | 数据 |
|------|------|
| 源文件总数 | 1,884 个 TypeScript 文件 |
| 代码行数 | 394,222 行 |
| 工具数量 | 54 个内置工具 |
| 特性标志 | 89 个 Feature Flags |
| 运行模式 | 10 种不同模式 |
| 最大文件 | main.tsx（4,683 行） |

---

## 第二章：整体架构一览

### 先用大白话理解

想象一家餐厅：

- **前台（TUI 界面）**：你和服务员（Claude）说话的地方，就是终端里那个漂亮的界面
- **厨房（查询引擎）**：接收你的需求，决定怎么做，协调所有资源
- **厨师们（工具系统）**：真正干活的人，每个工具就是一个专职厨师（有人专门切菜/读文件，有人专门炒菜/执行命令）
- **食材库（上下文系统）**：存放所有信息——你的代码、Git 历史、项目配置
- **餐厅规则（权限系统）**：哪些菜可以做，哪些需要经理批准，哪些绝对不能碰

### 五层架构

```
┌─────────────────────────────────────────────────────────┐
│                   第一层：用户界面层                        │
│         Ink (React for Terminal) + 自研 Yoga 布局引擎       │
├─────────────────────────────────────────────────────────┤
│                   第二层：协调控制层                        │
│    main.tsx (4683行) + QueryEngine.ts + commands.ts       │
├─────────────────────────────────────────────────────────┤
│                   第三层：核心引擎层                        │
│         query.ts (1729行) - AsyncGenerator 主循环          │
│      StreamingToolExecutor - 流式并发工具执行引擎            │
├─────────────────────────────────────────────────────────┤
│                   第四层：工具执行层                        │
│    54个工具 + toolExecution.ts (14步流水线) + 权限系统       │
├─────────────────────────────────────────────────────────┤
│                   第五层：基础服务层                        │
│   上下文系统 + 压缩系统 + MCP + 遥测 + 历史管理 + 认证       │
└─────────────────────────────────────────────────────────┘
```

### 技术栈

| 技术选型 | 具体实现 | 为什么这样选 |
|---------|---------|------------|
| 运行时 | **Bun**（非 Node.js） | 启动速度快 3-5 倍，原生 TypeScript 支持 |
| UI 框架 | **自定义 Ink**（React for Terminal） | 用 React 组件模型构建终端 UI |
| 布局引擎 | **自研 Yoga**（从 C++ 移植到纯 TS） | 无需 native 依赖，跨平台 |
| 状态管理 | **自研 Store**（类 Zustand + DeepImmutable） | 不可变状态，避免意外修改 |
| 打包器 | **Bun Bundler** | 支持编译时特性标志（Feature Flags） |
| 原生模块 | **Rust**（鼠标键盘）+ **Swift**（截屏） | 性能关键路径用原生语言 |

---

## 第三章：源码目录结构

### 先用大白话理解

就像一个大型公司的组织架构图，源码的目录结构告诉你"谁负责什么"。

### 顶层目录

```
src/
├── entrypoints/        # 入口点——不同模式的"大门"
│   ├── cli.tsx         # 命令行主入口
│   ├── sdk/            # SDK 模式入口
│   └── mcp.ts          # MCP 服务器模式入口
├── screens/            # 界面层——用户看到的东西
│   ├── REPL.tsx        # 主交互界面（最复杂的 UI 组件）
│   └── ...
├── tools/              # 工具层——Claude 能做的所有事情
│   ├── BashTool/       # 执行 Shell 命令
│   ├── FileReadTool/   # 读取文件
│   ├── FileWriteTool/  # 写入文件
│   ├── AgentTool/      # 启动子 Agent
│   └── ...（共 54 个）
├── services/           # 服务层——后台支撑系统
│   ├── api/            # API 通信（与 Claude 模型对话）
│   ├── mcp/            # MCP 协议实现
│   └── tools/          # 工具执行引擎
├── constants/          # 常量——系统提示词、配置
│   └── prompts.ts      # 核心系统提示词（最值得研究！）
├── utils/              # 工具函数——各种辅助功能
│   ├── permissions.ts  # 权限判定
│   ├── hooks.ts        # Hook 系统
│   └── settings/       # 配置管理
└── main.tsx            # 主协调器（4,683 行，整个系统的"大脑"）
```

### 最重要的 10 个文件

| 文件 | 行数 | 职责 |
|------|------|------|
| `main.tsx` | 4,683 | 主协调器：启动、配置、模式选择、会话管理 |
| `query.ts` | 1,729 | 查询循环：API 请求、压缩、重试、流式处理 |
| `QueryEngine.ts` | 1,295 | 查询引擎：状态管理、Token 预算、系统提示构建 |
| `Tool.ts` | 792 | 工具接口：泛型定义、权限声明、进度事件 |
| `commands.ts` | 754 | 命令注册表：100+ 斜杠命令的加载与过滤 |
| `history.ts` | 464 | 历史管理：JSONL 格式、并发安全写入 |
| `tools.ts` | 389 | 工具装配：条件编译、延迟加载、工具池构建 |
| `cost-tracker.ts` | 323 | 成本追踪：Token 统计、USD 计算 |
| `context.ts` | 189 | 上下文构建：Git 信息、CLAUDE.md、系统信息 |
| `permissions.ts` | ~300 | 权限判定：四层规则合并、拒绝追踪 |

---

## 第四章：代理循环——Claude Code 的心跳

### 先用大白话理解

想象你在玩一个游戏，游戏的核心循环是：

1. **看**：观察当前游戏状态（你在哪，周围有什么）
2. **想**：决定下一步怎么做
3. **做**：执行操作（移动、攻击、使用道具）
4. **看结果**：观察操作的效果
5. **重复**：回到第 1 步，直到游戏结束

Claude Code 的工作方式完全一样，只不过：

- **"看"** = 读取你的问题 + 当前项目状态
- **"想"** = Claude 模型分析，决定用哪个工具
- **"做"** = 执行工具（读文件、运行命令、写代码）
- **"看结果"** = 把工具的输出反馈给 Claude
- **"重复"** = 直到任务完成或需要你的输入

### 代理循环的代码实现

代理循环的核心在 `query.ts` 中，使用了 TypeScript 的 `AsyncGenerator` 模式：

```typescript
// query.ts - 简化版代理循环
async function* query(messages, tools, options) {
  while (true) {
    // 1. 调用 Claude API，获取流式响应
    const stream = await callClaudeAPI(messages, tools);

    // 2. 处理响应流
    for await (const event of stream) {
      if (event.type === 'text') {
        yield { type: 'text', content: event.text };  // 输出文字给用户看
      }

      if (event.type === 'tool_use') {
        // 3. Claude 决定使用工具
        const toolResult = await executeTool(event.tool, event.input);

        // 4. 把工具结果加入消息历史
        messages.push({ role: 'tool', content: toolResult });

        yield { type: 'tool_result', result: toolResult };
      }
    }

    // 5. 如果 Claude 说"我完成了"，退出循环
    if (lastResponse.stop_reason === 'end_turn') break;

    // 6. 否则继续循环（Claude 还有事要做）
  }
}
```

`AsyncGenerator` 的妙处在于：**它可以在循环过程中持续"产出"结果**，这样用户就能实时看到 Claude 正在做什么，而不是等所有事情做完才看到结果。

### 流式并发执行（StreamingToolExecutor）

Claude Code 有一个重要优化：**不等 API 响应完全结束，就开始执行工具**。

想象 Claude 在说："我需要先读文件 A，然后读文件 B，然后……"——普通实现会等 Claude 说完整句话再开始读文件。但 Claude Code 的 `StreamingToolExecutor` 在 Claude 刚说完"读文件 A"时就立刻开始读，不等后面的话。这让整体速度快了很多。

---

## 第五章：上下文加载——Claude 的"工作记忆"

### 先用大白话理解

你去一家新公司上班，第一天需要了解：公司的规章制度、你负责的项目情况、同事的工作风格……这些信息合在一起，才能让你高效工作。

Claude Code 每次启动时也会做类似的事情——它会主动收集当前工作环境的信息，这个过程叫做**上下文加载**。

### 两类上下文

**系统上下文**（`getSystemContext()`）——"当前项目是什么状态？"

Claude Code 不调用 `git` 命令行（太慢），而是**直接读取 `.git/` 目录下的文件**（< 1ms vs 10-50ms）：

```typescript
// context.ts - 直接读取 Git 文件，速度极快
const headContent = fs.readFileSync('.git/HEAD', 'utf8');      // 当前分支
const commitHash = fs.readFileSync('.git/refs/heads/main');    // 最新提交
```

收集的信息包括：当前 Git 分支、最近 5 条提交日志、未提交的变更摘要、操作系统信息。

**用户上下文**（`getUserContext()`）——"这个项目有什么特殊规则？"

Claude Code 会读取 `CLAUDE.md` 文件，这是你给 Claude 的"工作手册"。它支持三级层次合并：

```
~/.claude/CLAUDE.md          # 全局规则（对所有项目生效）
~/projects/CLAUDE.md         # 项目级规则（对当前项目生效）
~/projects/src/CLAUDE.md     # 子目录规则（对特定目录生效）
```

三个文件的内容会被合并，更具体的规则优先级更高。

### 系统提示词的动态构建

Claude Code 的系统提示词不是一段固定的文字，而是由多个**动态 Section** 拼接而成，每个 Section 都是一个惰性求值的函数：

```typescript
// 系统提示词由多个 Section 动态组合
const systemPrompt = [
  getCoreInstructionsSection(),    // 核心行为规范（静态缓存）
  getGitContextSection(),          // Git 状态（每次刷新）
  getClaudeMdSection(),            // CLAUDE.md 内容（文件变化时刷新）
  getMcpInstructionsSection(),     // MCP 工具说明（连接时注入）
  getToolSearchSection(),          // 工具搜索说明（按需加载）
].join('\n\n');
```

不同运行模式使用不同的 Section 组合——KAIROS 自主模式的系统提示词会包含："You are an autonomous agent. Use the available tools to do useful work..."

---

## 第六章：工具执行模型——14 步流水线

### 先用大白话理解

Claude Code 的工具执行**不是**"模型决定 → 直接跑函数"这么简单。每次工具调用都要经过一条 14 步的流水线，就像工厂的生产线，每一步都有严格的质检。

### 14 步工具执行流水线

`toolExecution.ts` 定义了完整的执行链路：

```
第 1 步：找 tool        → 根据名称在工具池中查找
第 2 步：解析 MCP 元数据 → 如果是 MCP 工具，解析额外信息
第 3 步：Input Schema 校验 → Zod 严格校验输入参数格式
第 4 步：validateInput  → 工具自定义的输入验证逻辑
第 5 步：推测性分类器检查 → 对 BashTool 启动 AI 安全分类器
第 6 步：运行 PreToolUse Hooks → 执行所有注册的前置钩子
第 7 步：解析 Hook 权限结果 → 综合 Hook 的权限建议
第 8 步：走权限决策    → 三级决策树（工具内部→规则→用户确认）
第 9 步：根据 updatedInput 修正输入 → Hook 可以修改工具输入
第 10 步：真正执行 tool.call() → 实际运行工具
第 11 步：记录 analytics/tracing → 遥测数据上报
第 12 步：运行 PostToolUse Hooks → 执行所有后置钩子
第 13 步：处理结构化输出 → 格式化 tool_result block
第 14 步：失败则走 PostToolUseFailure Hooks → 错误处理钩子
```

这条流水线保证了：**即使模型随便乱生成参数，也不会直接污染执行层**。

---

## 第七章：对话引擎——query.ts 深度解析

### 先用大白话理解

如果说代理循环是 Claude Code 的"心跳"，那么 `query.ts` 就是"心脏本身"——它是整个系统最核心的文件，1,729 行代码，负责所有与 Claude 模型的真实通信。

### QueryEngine.ts 的 Token 预算管理

`QueryEngine.ts` 负责计算每次请求能用多少 Token：

```
有效 Token 预算 = 模型上限 - 历史消息 Token - 工具 Schema Token - 保留缓冲区(13,000)
```

**为什么要保留 13,000 个 Token？** 这是经验值——为压缩请求本身和压缩后的首次回复预留空间。如果不留这个缓冲区，可能会出现"压缩请求本身就超出上限"的尴尬情况。

### 缓存断点策略（省钱神器）

Claude Code 会在消息序列的稳定位置插入**缓存断点**（Cache Breakpoints）。被缓存的内容在后续请求中只需要支付正常价格的 10%，多轮对话的成本可以降低 **70-90%**。

```typescript
// 在消息序列中插入缓存断点
messages[stablePosition].cache_control = { type: 'ephemeral' };
// 之后的请求中，这个位置之前的内容都从缓存读取，只需 1/10 的价格
```

### 消息规范化流程

每次发送给 API 之前，消息都要经过规范化处理：

1. **完整性检查**：确保消息序列符合 API 要求（不能连续两条 assistant 消息等）
2. **缓存断点插入**：在合适位置标记缓存断点
3. **工具引用清理**：删除已失效的工具调用引用
4. **Token 计数更新**：重新计算总 Token 数，决定是否需要压缩

---

## 第八章：三层上下文压缩系统

### 先用大白话理解

Claude 的"记忆"是有限的（就像人的短期记忆）。当对话变得很长时，早期的内容会超出 Claude 能处理的范围。Claude Code 用三层压缩系统来解决这个问题。

### 第一层：Snip 压缩（预防性）

**触发时机**：持续运行，静默执行，用户感知不到。

**工作原理**：扫描消息历史，找到"时间上远离当前对话焦点"的工具输出，将其替换为一行摘要：

```
原始内容（可能几百行）：
[FileReadTool 读取了 utils.ts 的完整内容]
function parseDate(str) { ... }
function formatCurrency(amount) { ... }
// ... 200 行代码

压缩后（1 行）：
[File content was read successfully - 200 lines]
```

**关键保护**：最近 N 轮对话的完整内容绝对不会被压缩，保证 Claude 对当前任务有完整记忆。

### 第二层：自动压缩（阈值触发）

**触发条件**：Token 用量达到 `有效上下文窗口 - 13,000` 时触发。

**工作原理**：发起一次独立的 LLM 调用（使用更快的模型），将整段对话历史压缩为结构化摘要：

```
压缩前：100 轮对话，50,000 个 Token
压缩后：1 段摘要，约 3,000 个 Token
节省：~94% 的 Token
```

**断路器设计**：如果连续 3 次压缩，每次回收的 Token 不到 500 个，压缩器自动禁用，避免无限循环。

### 第三层：响应式压缩（紧急兜底）

**触发条件**：API 返回 `prompt-too-long` 错误时触发。

**工作原理**：使用更激进的压缩策略，同时缩减 `maxOutputTokens`（最多重试 3 次）。

### 微压缩（精确外科手术）

`FileEditTool` 修改文件后，会精确删除之前 `FileReadTool` 读取该文件的工具结果——因为那个读取结果已经过时了，保留它只会浪费 Token。

---

## 第九章：工具系统与权限安全

### 先用大白话理解

Claude Code 有 54 个工具，就像一个工具箱里的各种工具。但不是所有工具都可以随便用——就像公司里有些设备需要申请才能使用，有些操作需要经理审批。

### 54 个工具分组

| 分组 | 工具 | 说明 |
|------|------|------|
| 文件操作 | FileReadTool, FileWriteTool, FileEditTool, GlobTool, GrepTool | 读写文件、搜索内容 |
| 命令执行 | BashTool, PowerShellTool, REPLTool | 执行 Shell 命令 |
| Agent 协作 | AgentTool, SendMessageTool | 启动子 Agent、发送消息 |
| Web 访问 | WebSearchTool, WebFetchTool | 搜索网络、抓取网页 |
| MCP 工具 | MCPTool, ListMcpResourcesTool, ReadMcpResourceTool | MCP 协议工具 |
| 任务管理 | TaskCreateTool, TaskGetTool, TaskListTool... | 异步任务管理 |
| 模式切换 | EnterPlanModeTool, ExitPlanModeTool... | 切换工作模式 |
| KAIROS 专属 | SleepTool, PushNotificationTool, SubscribePRTool... | 自主模式专用 |

### 延迟加载机制（工具搜索）

Claude Code 不会把所有 54 个工具的描述一次性发给模型（那会消耗大量 Token）。它使用了**延迟加载**策略：

- 模型初始只看到**约 15 个核心工具**
- 其余工具的描述缓存在 `ToolSearchTool` 内部
- 当模型需要特定能力时，调用 `ToolSearchTool` 进行关键词搜索

### BashTool 的 25 项安全检查

BashTool 是最危险的工具（可以执行任意命令），因此有 25 项安全检查，包括：

- **命令注入防护**：检测 `;`、`&&`、`||`、`|`、反引号等 Shell 元字符
- **文件系统破坏**：阻止 `rm -rf /`、`chmod -R 777` 等危险操作
- **网络管道执行**：阻止 `curl | sh`、`wget -O- | bash` 等远程代码执行
- **环境变量劫持**：检测 `PATH`、`LD_PRELOAD`、`DYLD_LIBRARY_PATH` 等被修改
- **迭代固定点算法**：处理嵌套包装（`sudo bash -c "env sh -c '...'"` 这类多层嵌套），连续两轮解析结果不再变化时才算收敛

### 权限三级决策树

每个工具操作都要经过三级决策：

```
第一级：工具内部检查
  └─ 工具自己判断这个操作是否明显安全/危险

第二级：规则匹配（四层来源，优先级合并）
  ├─ Settings 配置（settings.json 全局权限偏好）
  ├─ CLI 参数（命令行传入的权限覆盖）
  ├─ Command 参数（特定命令的权限上下文）
  └─ Session 状态（会话中动态积累的权限决策）

第三级：用户确认
  └─ 前两级都无法决定时，弹出提示让用户选择
```

**拒绝追踪机制**：连续拒绝 3 次 → 触发"策略回退提示"；累计拒绝 20 次 → 触发更强的回退信号（让 Claude 换一种方式完成任务）。

---

## 第十章：多 Agent 协作

### 先用大白话理解

一个人做不完的工作，可以分给多个人同时做。Claude Code 支持多个 Claude 实例并行工作，这就是**多 Agent 协作**。

### AgentTool：Agent 编排控制器

`AgentTool` 是多 Agent 系统的核心，它本质上是一个**编排控制器**，负责：

1. 判断是否需要启动子 Agent
2. 解析团队上下文（Team Context）
3. 选择合适的 Agent 类型
4. 决定是前台执行还是后台执行
5. 构建子 Agent 的系统提示和工具集
6. 调用 `runAgent()` 启动子 Agent

### Fork Path vs Normal Path

子 Agent 有两种启动路径，这是一个非常精妙的设计：

**Fork Path（分叉路径）**：
- 继承主线程的系统提示和工具集（保持 byte-identical）
- 目的是**最大化 Prompt Cache 命中率**
- 适用于：需要快速启动、复用主线程上下文的场景

**Normal Path（普通路径）**：
- 基于 `agentDefinition` 生成全新的系统提示
- 只给该 Agent 所需的上下文和工具
- 适用于：专职 Agent（Explore/Plan/Verification）

> **关键洞察**：Fork 不是"再开一个普通 Agent"，而是**为了 Cache 和 Context 继承专门优化过的执行路径**。普通人只想"子任务能跑"，Claude Code 想的是"子任务能跑，而且尽量复用主线程 Cache，不白烧 Token"。这就是产品级系统思维。

### 四种内置专职 Agent

| Agent | 文件 | 职责 |
|-------|------|------|
| Explore Agent | `built-in/exploreAgent.ts` | 探索代码库，不污染主线程 |
| Plan Agent | `built-in/planAgent.ts` | 制定实现方案，规划和实现分离 |
| Verification Agent | `built-in/verificationAgent.ts` | 验证结果，对抗"实现者 bias" |
| General Purpose Agent | `built-in/generalPurposeAgent.ts` | 通用子任务执行 |

---

## 第十一章：MCP 集成

### 先用大白话理解

MCP（Model Context Protocol）是 Anthropic 提出的一个开放协议，让 Claude 可以连接各种外部工具和数据源。就像手机的 App Store——你可以安装各种 App 来扩展手机功能，MCP 让你可以给 Claude 安装各种"能力插件"。

### MCP 不只是工具桥，还是行为说明注入通道

这是一个很多人不知道的细节。从 `prompts.ts` 可以看到：

```typescript
getMcpInstructionsSection()  // 把 MCP 服务器的 instructions 注入系统提示词
```

当一个 MCP 服务器连接时，它可以同时注入：
1. **新工具**（新的能力）
2. **如何使用这些工具的说明**（注入到系统提示词）

这让 MCP 的价值远高于简单的"工具注册表"。

### Agent-Specific MCP Servers

每个子 Agent 可以有自己专属的 MCP 服务器，通过 `initializeAgentMcpServers()` 实现：

- 从现有配置按名字引用服务器
- 在 frontmatter 里联定义 agent-specific MCP server
- 连接服务器、拉取工具、在 Agent 结束时做 cleanup

这意味着 Agent 不只是消费全局 MCP，它还可以带自己的外接能力。

---

## 第十二章：10 种运行模式

Claude Code 不只是一个终端聊天工具，它有 10 种不同的运行模式：

| 模式 | 启动方式 | 适用场景 |
|------|---------|---------|
| **交互模式** | `claude`（默认） | 日常使用，完整 TUI 界面 |
| **无头模式** | `claude -p "..."` | CI/CD 流水线，输出到 stdout |
| **Bridge 远程控制** | `claude bridge` | 手机/Web 远程连接 |
| **KAIROS 助手模式** | `claude assistant` | 7×24 自主运行 |
| **Coordinator 协调器** | `--agent-teams` | Leader Claude 协调多个 Worker |
| **Daemon 守护进程** | `--daemon-worker` | 后台常驻进程 |
| **语音模式** | `/voice` 命令 | 语音交互 |
| **SDK 模式** | `entrypoints/sdk/` | 程序化调用 |
| **MCP 服务器模式** | `claude mcp` | Claude Code 自身作为 MCP 服务器 |
| **SSH 远程模式** | `claude ssh <host>` | 连接远程服务器 |

### KAIROS 模式：7×24 自主运行

KAIROS 是 Claude Code 最神秘的特性之一。在这个模式下，Claude Code 可以**持续自主运行**，不需要用户每次输入。

KAIROS 模式的系统提示词包含：
```
You are an autonomous agent. Use the available tools to do useful work.
You will receive <tick> prompts that keep you alive between turns...
```

它有专属工具：`SleepTool`（暂停等待）、`PushNotificationTool`（推送通知给用户）、`SubscribePRTool`（订阅 PR 事件）、`KAIROS_DREAM`（自主做梦/记忆整理）。

---

## 第十三章：为什么 Claude Code 这么强

### 它不是一个 prompt，而是一个 operating model

很多人复刻 coding agent 时只会拿走：一个 system prompt + 一个文件编辑工具 + 一个 bash 工具 + 一个 CLI 壳。

但 Claude Code 真实的护城河是：

| 维度 | 普通实现 | Claude Code |
|------|---------|------------|
| Prompt 架构 | 一段固定文字 | 多动态 Section，按模式/状态动态组合 |
| 工具治理 | 直接调用函数 | 14 步流水线，完整的 runtime governance |
| 权限模型 | 没有或简单 | 三级决策树 + 四层规则 + 拒绝追踪 |
| Hook 策略层 | 没有 | PreToolUse/PostToolUse，可修改输入、阻止执行 |
| Agent 专业化 | 一个 Agent 做所有事 | Explore/Plan/Verification 明确分工 |
| Skill 工作流 | 没有 | markdown prompt bundle，可复用能力包 |
| Plugin 集成 | 没有 | Prompt + Metadata + Runtime Constraints |
| MCP 指令注入 | 只注册工具 | 同时注入工具 + 使用说明到系统提示词 |
| Prompt Cache 优化 | 没有 | 缓存断点 + fork path 共享 cache |
| Agent 生命周期 | 一次函数调用 | transcript/telemetry/cleanup/resume 完整生命周期 |

### 它把"好行为"制度化了

Claude Code 最大的优势之一，不是模型更聪明，而是：**它不把"好习惯"交给模型即兴发挥，而是写进 prompt 和 runtime 规则里**。

例如：不要乱加功能、不要过度抽象、不要赌重试被拒绝的工具、不要未验证就说成功、不要随便做风险操作、匹配 skill 时必须执行 skill、verification 不能只看代码必须跑命令……

这种制度化，会极大提高系统一致性。

### 它特别懂"上下文是稀缺资源"

源码中大量设计都在围绕上下文做优化：

- system prompt 动静边界（静态部分缓存，动态部分按需更新）
- prompt cache boundary（缓存断点，省 70-90% 费用）
- fork path 共享 cache（子 Agent 复用主线程缓存）
- skill 按需注入（不把所有 skill 都塞进系统提示词）
- MCP instructions 按连接状态注入（只注入当前连接的 MCP 说明）
- function result clearing（清理已过时的工具结果）
- 三层压缩系统（Snip + 自动压缩 + 响应式压缩）

这说明 Anthropic 不是把 Token 当免费空气，而是当 runtime 预算来管理。

---

## 第十四章：关键模块导读

### query.ts：主对话循环（最值得精读）

`query.ts` 是整个系统的核心，建议按以下顺序阅读：

1. 找到主函数 `query()` 的函数签名，理解它的输入和输出
2. 找到 `while (true)` 循环，理解代理循环的基本结构
3. 找到 `stream` 相关代码，理解流式处理
4. 找到 `tool_use` 处理逻辑，理解工具调用
5. 找到压缩触发逻辑，理解三层压缩系统

### bashSecurity.ts：安全防线

这个文件展示了 Anthropic 对安全的极度重视。25 项检查中最有意思的是**迭代固定点算法**：

```typescript
// 不断解包命令包装，直到结果不再变化
let prev = command;
while (true) {
  const unwrapped = unwrapCommand(prev);
  if (unwrapped === prev) break;  // 固定点：结果不再变化
  prev = unwrapped;
}
// 此时 prev 就是真正的底层命令
```

这保证了无论嵌套多少层 `sudo bash -c "env sh -c '...'"` 都能找到真实命令。

### autoCompact.ts：压缩决策

这个文件展示了系统如何在"保留信息"和"节省 Token"之间做权衡。断路器设计（连续 3 次压缩效果不佳就停止）是防止系统陷入无限循环的关键安全机制。

### permissions.ts：权限判定

这个文件展示了如何把复杂的权限逻辑组织成清晰的代码。四层规则合并（Settings → CLI → Command → Session）的设计让权限系统既灵活又可预测。

---

## 延伸阅读

### 源码仓库

- [sanbuphy/claude-code-source-code](https://github.com/sanbuphy/claude-code-source-code) — Claude Code v2.1.88 反编译源码，包含完整的 TypeScript 源文件

### 参考资料

- [Claude Code 0331 系统报告（手工川）](https://mp.weixin.qq.com/s) — 基于 Source Map 泄露版本的深度技术分析，30 章 6 附录
- [Claude Code 源码深度研究报告](https://github.com) — 聚焦 Agent 编排、Skills/MCP 生态的深度分析

### 官方文档

- [Claude Code 官方文档](https://docs.anthropic.com/en/docs/claude-code) — Anthropic 官方使用文档
- [Anthropic API 文档](https://docs.anthropic.com/en/api) — 包括 Tool Use、Streaming 等功能
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io) — MCP 协议官方规范

---

## 贡献

欢迎提交 Issue 和 Pull Request！如果你发现教程中有错误或不清晰的地方，请告诉我们。

## License

MIT License — 自由使用、修改和分发。
