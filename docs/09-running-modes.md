# 第 9 章：10 种运行模式

> **本章目标**：了解 Claude Code 的 10 种运行模式，以及 89 个 Feature Flag 背后的工程哲学。

---

## 先用大白话理解

同一个人，在不同场合会有不同的工作方式：在会议室里是「讨论模式」，在工位上是「专注模式」，在紧急情况下是「救火模式」。

Claude Code 也一样——根据不同的使用场景，它有 10 种不同的运行模式，每种模式下的行为、权限、工具集都不一样。

---

## 10 种运行模式

| 模式 | 触发方式 | 核心特征 |
|------|---------|---------|
| **Interactive** | 默认启动 | 交互式对话，每步可确认 |
| **Task** | `--print` 标志 | 非交互式，执行完退出 |
| **KAIROS** | Feature Flag | 完全自主，最小化人工干预 |
| **Coordinator** | `--coordinator` | 纯指挥官，只分配任务 |
| **Subagent** | 被父 Agent 启动 | 执行单一子任务 |
| **Swarm Member** | Swarm 初始化 | 点对点协作网络中的节点 |
| **Headless** | `--headless` | 无 UI，适合 CI/CD |
| **Pipe** | stdin 输入 | 管道模式，接收流式输入 |
| **Review** | `--review` | 只读模式，只分析不修改 |
| **Doctor** | `/doctor` 命令 | 诊断模式，检查环境问题 |

---

## 最重要的两种模式对比

### Interactive 模式（日常使用）

```
用户输入 → AI 思考 → 执行工具（危险操作需确认）→ 显示结果 → 等待用户
```

特点：
- 每个危险操作都会弹出确认框
- 用户可以随时中断（Ctrl+C）
- 对话历史在会话内持久化
- 支持 `/` 斜杠命令

### KAIROS 模式（自主执行）

```
用户设定目标 → AI 自主规划 → 批量执行 → 完成后汇报
```

特点：
- 最小化人工干预（只在真正必要时才问用户）
- 有内置的自我检查机制
- 会生成详细的执行日志
- 适合长时间运行的复杂任务

---

## 89 个 Feature Flag

Claude Code 有 89 个 Feature Flag，控制各种功能的开关。这些 Flag 分为几类：

| 类别 | 数量 | 示例 |
|------|------|------|
| 实验性功能 | ~30 | `experimental_multi_file_edit` |
| 内部功能 | ~20 | `coordinator_mode`、`swarm_enabled` |
| 性能优化 | ~15 | `parallel_tool_execution` |
| UI 功能 | ~10 | `buddy_system`（宠物系统）|
| 安全控制 | ~14 | `strict_bash_validation` |

**编译时 Feature Gate**：内部功能（如协调器、Swarm）在公开版本的构建过程中被**物理删除**，而不是运行时隐藏。这意味着即使你拿到了二进制文件，也无法通过修改配置来启用这些功能。

---

## 模式切换的实现

```typescript
// 根据启动参数和 Feature Flag 决定运行模式
function determineRunMode(args: Args, flags: FeatureFlags): RunMode {
  if (args.headless) return RunMode.Headless;
  if (args.coordinator && flags.coordinator_mode) return RunMode.Coordinator;
  if (args.print) return RunMode.Task;
  if (flags.kairos_mode) return RunMode.KAIROS;
  return RunMode.Interactive; // 默认
}
```

---

> 下一章：[记忆系统与 CLAUDE.md →](#/docs/10-memory-claude-md)
