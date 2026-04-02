# 第 9 章：10 种运行模式

> **本章目标**：了解 Claude Code 的 10 种运行模式，以及 89 个 Feature Flag 背后的工程哲学。

---

## 9.1 先用大白话理解

同一个人，在不同场合会有不同的工作方式：在会议室里是「讨论模式」，在工位上是「专注模式」，在紧急情况下是「救火模式」。

Claude Code 也一样——根据不同的使用场景，它有 10 种不同的运行模式，每种模式下的行为、权限、工具集都不一样。

---

## 9.2 10 种运行模式

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

## 9.3 最重要的两种模式对比

### Interactive 模式（日常使用）

```
用户输入 → AI 思考 → 执行工具（危险操作需确认）→ 显示结果 → 等待用户
```

特点：
- 每个危险操作都会弹出确认框
- 用户可以随时中断（Ctrl+C）
- 对话历史在会话内持久化
- 支持 `/` 斜杠命令

Interactive 模式是大多数用户日常使用的模式。它的核心设计原则是**人类始终在控制循环中**：AI 可以自主读文件、搜索代码，但在写文件、执行命令等危险操作前，必须获得用户确认。

### KAIROS 模式（自主执行）

```
用户设定目标 → AI 自主规划 → 批量执行 → 完成后汇报
```

特点：
- 最小化人工干预（只在真正必要时才问用户）
- 有内置的自我检查机制
- 会生成详细的执行日志
- 适合长时间运行的复杂任务

KAIROS 模式是 Claude Code 最强大也最危险的模式。它的设计目标是让 AI 能够**完成需要数小时人工操作的复杂任务**，比如重构整个代码库、迁移数据库 schema、修复所有测试失败。

> **注意**：KAIROS 模式在公开版本中通过编译时 Feature Gate 被移除。这不是运行时隐藏，而是物理删除——即使你拿到了源码，也需要特定的构建配置才能启用它。

### Task 模式（CI/CD 集成）

```bash
# 在 CI/CD 流水线中使用
claude --print "检查这个 PR 是否有安全漏洞" --output json
```

Task 模式专为程序化调用设计：
- 接受单次查询，执行完毕后退出
- 支持 JSON 输出格式，方便脚本解析
- 不需要终端 UI，适合无头环境
- 可以通过 `--max-turns` 限制最大循环次数

---

## 9.4 89 个 Feature Flag

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

## 9.5 模式切换的实现

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

模式切换不仅仅是改变一个变量——它会影响整个系统的行为：

```typescript
// 不同模式下的权限策略
const permissionPolicy = {
  [RunMode.Interactive]: 'confirm-dangerous',    // 危险操作需确认
  [RunMode.KAIROS]:      'auto-approve-all',     // 全部自动批准
  [RunMode.Task]:        'confirm-destructive',  // 只有破坏性操作需确认
  [RunMode.Review]:      'read-only',            // 只读，不允许任何写操作
  [RunMode.Headless]:    'inherit-from-config',  // 从配置文件读取
};
```

---

## 9.6 编译时 Feature Gate 的工程哲学

89 个 Feature Flag 中，有一类特殊的——**编译时 Feature Gate**。这些 Flag 不是运行时开关，而是在构建时就决定代码是否存在。

```typescript
// Bun 的编译时 Feature Flag
import { feature } from 'bun';

// 如果构建时 coordinator_mode = false
// 这整个 if 块将被物理删除
if (feature('coordinator_mode')) {
  // 协调器相关代码
  // 在外部发布版本中，这里的代码完全不存在
}
```

这种设计的好处：

1. **安全性**：即使用户反编译了二进制文件，也看不到内部功能的代码
2. **性能**：删除的代码不占用任何运行时内存，也不会影响启动速度
3. **代码库统一**：内部功能和外部功能共用同一个代码库，不需要维护两个分支

这也解释了为什么 Claude Code 的公开版本和内部版本看起来很不一样——它们从同一个源码库构建，但使用了不同的 Feature Flag 组合。

---

## 9.7 Headless 模式与 CI/CD 集成

Headless 模式是 Claude Code 在自动化场景中的主要使用方式：

```yaml
# GitHub Actions 示例
- name: Claude Code Review
  run: |
    claude --headless \
      --print "Review this PR for security issues and code quality" \
      --output json \
      --max-turns 10 \
      > review.json
    
    # 解析 JSON 输出
    python3 parse_review.py review.json
```

Headless 模式的特点：
- 无终端 UI，所有输出通过 stdout/stderr
- 支持 JSON 格式输出，方便脚本解析
- 可以设置超时时间（`--timeout`）
- 可以限制最大工具调用次数（`--max-turns`）
- 退出码反映任务状态（0=成功，非0=失败）

---

> 下一章：[记忆系统与 CLAUDE.md →](#/docs/10-memory-claude-md)
