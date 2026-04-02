# Claude Code School

> **从零到一，读懂 Claude Code 的架构与源码**

一份面向零基础读者的 Claude Code 源码架构学习教程。基于 Source Map 泄露版本的完整 TypeScript 源码，从「AI 是什么」讲起，一步步带你理解工业级 AI Agent 的真实实现方式。

**在线阅读** → [opendemon.github.io/claude-code-school](https://opendemon.github.io/claude-code-school/)

---

## 谁应该读这份教程？

| 读者类型 | 收获 |
|---------|------|
| **AI 产品经理** | 理解 Claude Code 的核心能力边界，做出更准确的产品判断 |
| **AI 应用开发者** | 学习工业级 Agent 的架构模式，用于自己的产品设计 |
| **提示词工程师** | 看懂 Anthropic 如何设计模块化提示词和验证角色 |
| **技术好奇者** | 了解一个真实的 AI 产品是怎么构建的 |

**前置要求**：用过终端，知道「代码」是什么，不需要会写代码。

---

## 关键数字

| 指标 | 数值 |
|------|------|
| 核心源码文件 | ~150 个 TypeScript 文件 |
| 内置工具数量 | 66+ 个 |
| Feature Flag 数量 | 89 个 |
| 运行模式数量 | 10 种 |
| 安全检查层数 | 5 层（BashTool） |
| 上下文压缩级别 | 4 级流水线 |
| Buddy 物种数量 | 18 种 |

---

## 阅读建议

**只有 10 分钟？** → 从 [快速入门](docs/quick-start.md) 开始，了解最核心的 5 个概念

**想系统学习？** → 按章节顺序从第 0 章读到第 12 章，每章约 15-20 分钟

**只对某个主题感兴趣？** → 直接跳到对应章节，每章都是独立的

---

## 目录

### 入门
- [快速入门（10 分钟版）](docs/quick-start.md)
- [第 0 章：Claude Code 是什么](docs/00-what-is-claude-code.md)

### 核心架构
- [第 1 章：整体架构一览](docs/01-architecture-overview.md)
- [第 2 章：源码目录结构](docs/02-source-structure.md)
- [第 3 章：代理循环——AI 的心跳](docs/03-agentic-loop.md)
- [第 4 章：对话引擎深度解析](docs/04-query-engine.md)
- [第 5 章：上下文工程与压缩](docs/05-context-compression.md)

### 能力系统
- [第 6 章：工具系统与权限安全](docs/06-tools-permissions.md)
- [第 7 章：多 Agent 协作架构](docs/07-multi-agent.md)
- [第 8 章：MCP 集成与扩展](docs/08-mcp-integration.md)

### 进阶与彩蛋
- [第 9 章：10 种运行模式](docs/09-running-modes.md)
- [第 10 章：记忆系统与 CLAUDE.md](docs/10-memory-claude-md.md)
- [第 11 章：宠物系统与彩蛋](docs/11-buddy-system.md)
- [第 12 章：源码泄露的完整故事](docs/12-leak-story.md)

### 参考
- [关键数据速查表](docs/reference-key-numbers.md)

---

## 源码来源

本教程基于以下公开资料整理：

- **源码仓库**：[sanbuphy/claude-code-source-code](https://github.com/sanbuphy/claude-code-source-code) — 基于 Source Map 泄露的完整 TypeScript 源码还原
- **参考分析**：Claude Code 0331 系统报告（手工川）— 基于泄露版本的深度技术分析，30 章 6 附录

---

## License

MIT © OpenDemon
