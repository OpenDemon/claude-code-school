# 第 4 章：对话引擎深度解析

> **本章目标**：理解 QueryEngine 如何管理多轮对话、构建上下文，以及系统提示词的模块化设计。

---

## 先用大白话理解

想象你在和一个翻译官合作：

每次你说一句话，翻译官不只是翻译你这一句，而是把**整个对话历史**都翻译给对方听——因为对方需要完整的上下文才能理解你现在说的话。

Claude Code 的 QueryEngine 就是这个翻译官。每次你发一条消息，它不只是把这条消息发给 AI，而是把**整个对话历史 + 系统规则 + 当前环境信息**打包在一起发过去。

这就是为什么 AI 能「记住」你之前说了什么——不是因为它有真正的记忆，而是因为每次都把历史记录一起发过去了。

---

## 4.1 QueryEngine 的职责

`QueryEngine.ts`（1,155 行）是会话管理的核心，负责：

- 维护对话历史（`Message[]` 数组）
- 构建每次 API 调用的完整上下文
- 管理并发请求（防止多个请求同时修改历史）
- 决定何时触发上下文压缩
- 处理会话的创建、恢复、保存

---

## 4.2 上下文的组成结构

每次调用 API，发送的不只是你的消息，而是一个精心构建的完整上下文：

```mermaid
graph LR
    subgraph 每次 API 调用的内容
        SP[系统提示词\n模块化组装]
        Env[环境信息\n目录 + git 状态]
        CM[CLAUDE.md\n项目规则]
        History[对话历史\n所有历史消息]
        Tools[工具列表\n66+ 工具定义]
        Current[当前消息\n你刚发的]
    end
```

这些内容按顺序拼接，形成一个可能长达数万 Token 的请求。

---

## 4.3 系统提示词的模块化设计

这是 Claude Code 最值得借鉴的设计之一：**系统提示词不是一大段文字，而是按功能分成独立模块，根据场景动态组合**。

### 系统提示词优先级

系统提示词的构建有严格的优先级，由 `buildEffectiveSystemPrompt()`（`src/utils/systemPrompt.ts`）实现：

```typescript
// 优先级从高到低：
// 0. overrideSystemPrompt — 完全覆盖（如 loop 模式）
// 1. coordinatorSystemPrompt — 协调器模式（Feature-gated）
// 2. agentSystemPrompt — Agent 定义的提示词
//    - Proactive 模式：追加到默认提示词后面
//    - 普通模式：替换默认提示词
// 3. customSystemPrompt — --system-prompt 参数指定
// 4. defaultSystemPrompt — 标准 Claude Code 提示词
// + appendSystemPrompt 始终追加到末尾（除 override 模式）
```

### 静态/动态边界标记

系统提示词中有一个关键的设计元素——`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`（`src/constants/prompts.ts`）。这是一个哨兵字符串，它将系统提示词数组分成两半：

- **边界之前**：核心指令、工具描述、安全规则等——对**所有用户的所有会话**都完全相同的内容
- **边界之后**：MCP 工具指令、输出风格、语言偏好等——因用户/会话而异的内容

为什么需要这个边界？因为它直接影响提示词缓存的效率。边界之前的静态部分可以使用 `scope: 'global'` 缓存，**跨所有用户共享**——这意味着全球数百万 Claude Code 用户可以共享同一份缓存的核心系统提示词。

### Section-Level 缓存

系统提示词的各个组成部分通过 `systemPromptSections.ts` 实现了 section 级别的缓存：

```typescript
// 计算一次，缓存到 /clear 或 /compact
systemPromptSection('toolInstructions', () => buildToolPrompt(...))

// 每轮重新计算，会破坏提示词缓存
DANGEROUS_uncachedSystemPromptSection(
  'modelOverride',
  () => getModelOverrideConfig(),
  'Live feature flags may change mid-session'  // 必须提供理由
)
```

`DANGEROUS_` 前缀是刻意为之的代码级警示——它提醒开发者：**这个 section 每轮都会重新计算，如果值发生变化会破坏提示词缓存**。开发者必须提供一个 `_reason` 参数解释为什么缓存破坏是必要的。

---

## 4.4 环境信息注入

每次对话开始时，Claude Code 会自动收集当前环境信息，注入到上下文中：

```typescript
// src/context.ts — getGitStatus()（memoize 缓存，每会话只计算一次）
export const getGitStatus = memoize(async (): Promise<string | null> => {
  const isGit = await getIsGit()
  if (!isGit) return null
  try {
    // 5 个 git 命令并行执行
    const [branch, mainBranch, status, log, userName] = await Promise.all([
      getBranch(),
      getDefaultBranch(),
      execFileNoThrow(gitExe(), ['--no-optional-locks', 'status', '--short'], ...)
        .then(({ stdout }) => stdout.trim()),
      execFileNoThrow(gitExe(), ['--no-optional-locks', 'log', '--oneline', '-n', '5'], ...)
        .then(({ stdout }) => stdout.trim()),
      execFileNoThrow(gitExe(), ['config', 'user.name'], ...)
        .then(({ stdout }) => stdout.trim()),
    ])
    // 状态截断至 2000 字符，防止大量未提交文件撑爆上下文
    const truncatedStatus = status.length > MAX_STATUS_CHARS
      ? status.substring(0, MAX_STATUS_CHARS) +
        '\n... (truncated because it exceeds 2k characters...)'
      : status
    return [
      `This is the git status at the start of the conversation.`,
      `Current branch: ${branch}`,
      `Main branch: ${mainBranch}`,
      ...(userName ? [`Git user: ${userName}`] : []),
      `Status:\n${truncatedStatus || '(clean)'}`,
      `Recent commits:\n${log}`,
    ].join('\n\n')
  } catch (error) {
    return null
  }
})
```

值得注意的设计细节：

- **`Promise.all` 并行**：5 个 git 命令同时执行，而不是串行等待——在大型仓库中可以节省数百毫秒
- **`--no-optional-locks`**：避免 git 命令获取锁导致与其他 git 操作冲突
- **`MAX_STATUS_CHARS = 2000`**：限制状态输出长度。想象一个有 500 个未提交文件的 monorepo——不截断的话，git status 本身就会消耗大量上下文预算

---

## 4.5 CLAUDE.md 的加载机制

CLAUDE.md 是 Claude Code 的**项目级指令文件**，类似于 `.editorconfig` 或 `.eslintrc`，但面向 AI Agent。

### 发现顺序

1. **管理策略文件**：从 MDM（移动设备管理）策略中读取的指令（如 `/etc/claude-code/CLAUDE.md`）
2. **用户主目录**：`~/.claude/CLAUDE.md` 下的全局配置
3. **项目文件**：从 CWD 向上遍历目录树，查找每一层的指令文件
4. **本地文件**：`CLAUDE.local.md`（不提交到 git 的个人指令）
5. **显式附加目录**：`--add-dir` 参数指定的额外目录

### 优先级排序

文件按从远到近的顺序加载——**靠近 CWD 的文件后加载**，因此优先级更高：

```
~/.claude/CLAUDE.md          ← 全局规则（最先加载，优先级最低）
    ↓
~/projects/CLAUDE.md         ← 父目录规则
    ↓
~/projects/myapp/CLAUDE.md   ← 项目根目录规则（最常用）
    ↓
~/projects/myapp/src/CLAUDE.md ← 子目录规则（最后加载，优先级最高）
```

### @include 指令

CLAUDE.md 文件可以通过 `@` 语法引用其他文件：

```markdown
# 项目指令
@./docs/coding-standards.md
@~/global-rules.md
@/etc/company-policy.md
```

- `@path`（无前缀）等同于 `@./path`，按相对路径解析
- `@~/path` 从用户主目录解析
- `@/path` 按绝对路径解析
- 只允许文本文件扩展名（.md、.txt 等），防止加载二进制文件
- 通过跟踪已处理文件防止循环引用

---

## 4.6 对话历史的管理

QueryEngine 维护一个 `Message[]` 数组，每条消息有三种角色：

| 角色 | 含义 | 示例 |
|------|------|------|
| `user` | 用户消息 | 你发的文字 |
| `assistant` | AI 响应 | AI 的回复（含工具调用） |
| `tool` | 工具结果 | 工具执行后的返回值 |

一次完整的工具调用在历史里看起来像这样：

```
user: "帮我读一下 config.json"
assistant: [工具调用: FileRead("config.json")]
tool: [工具结果: "{ 'name': 'myapp', 'version': '1.0' }"]
assistant: "config.json 的内容是：项目名称 myapp，版本 1.0"
```

### 消息规范化（normalizeMessagesForAPI）

`normalizeMessagesForAPI()`（`src/utils/messages.ts`，约 200 行）是消息发送前的关键处理步骤。主要处理步骤：

1. **附件重排序**：附件消息在内部可能出现在任意位置，但 API 要求它们在语义上关联的消息之前
2. **过滤虚拟消息**：标记为 `isVirtual` 的消息（如 REPL 内部工具调用的临时消息）被移除
3. **剥离内部元素**：从消息中移除 `tool_reference`（工具引用标记）、advisor blocks 等 API 不认识的内容
4. **合并分裂消息**：流式解析器可能将一个 API 响应拆分为多条具有相同 `message.id` 的消息，需要合并
5. **验证和修复配对**：API 要求每个 `tool_use` block 都有对应的 `tool_result`，此步骤检测并修复孤儿 block

**为什么这么复杂？** 因为 Claude API 对消息格式有严格要求：用户/助手消息必须交替出现、`tool_use`/`tool_result` 必须配对、thinking 块不能出现在不支持的位置。`normalizeMessagesForAPI` 是防御层——确保无论内部状态多混乱，API 始终收到合法的消息序列。

---

## 4.7 Token 使用追踪

QueryEngine 维护完整的 Token 使用统计：

```typescript
totalUsage: {
  input_tokens: 0,
  output_tokens: 0,
  cache_read_input_tokens: 0,
  cache_creation_input_tokens: 0,
  server_tool_use_input_tokens: 0,
}
```

- 每条 API 响应的 `message_delta` 事件中，`currentMessageUsage` 被更新
- `message_stop` 时，`currentMessageUsage` 通过 `accumulateUsage()` 累加到 `totalUsage`
- `getTotalCost()` 基于 `totalUsage` 和模型定价计算 USD 总成本
- 一旦 `getTotalCost() > maxBudgetUsd`，整个查询终止——这是防止意外高成本的安全机制

---

## 4.8 system-reminder 注入机制

Claude Code 需要在对话的各个位置注入系统级信息。`<system-reminder>` 是解决这个问题的统一机制：

```typescript
// 用户上下文前置（prependUserContext）
createUserMessage({
  content: `<system-reminder>
As you answer the user's questions, you can use the following context:
# claudeMd
${claudeMdContent}
# currentDate
Today's date is 2026-04-01.
IMPORTANT: this context may or may not be relevant to your tasks.
</system-reminder>`,
  isMeta: true,
})
```

XML 标签创建了一个清晰的语义边界。模型通过训练知道 `<system-reminder>` 内的内容是系统自动注入的元数据，而不是用户的直接输入。这使得系统可以在对话的**任意位置**注入上下文，而不会混淆消息的「发言者」身份。

---

## 4.9 并发控制

QueryEngine 使用一个简单但有效的锁机制，防止多个请求同时修改对话历史：

```typescript
class QueryEngine {
  private isProcessing = false;
  private queue: PendingRequest[] = [];

  async processMessage(message: string) {
    if (this.isProcessing) {
      // 排队等待
      return new Promise(resolve => {
        this.queue.push({ message, resolve });
      });
    }

    this.isProcessing = true;
    try {
      const result = await this.executeQuery(message);
      return result;
    } finally {
      this.isProcessing = false;
      // 处理队列中的下一个请求
      if (this.queue.length > 0) {
        const next = this.queue.shift();
        this.processMessage(next.message).then(next.resolve);
      }
    }
  }
}
```

---

## 4.10 设计洞察

1. **Memoize 保证幂等性**：`getSystemContext` 和 `getUserContext` 都是 memoized 的，每会话只计算一次
2. **缓存感知的上下文组装**：上下文的注入顺序（系统提示词在前、用户上下文在消息前）考虑了提示词缓存的命中率
3. **system-reminder 作为统一注入通道**：通过 XML 标签包装，在消息流的任意位置注入系统信息，而不混淆角色边界
4. **DANGEROUS_ 前缀作为代码级警示**：强迫开发者在破坏缓存时提供理由，防止无意识的性能损耗

---

> 下一章：[上下文工程与压缩 →](#/docs/05-context-compression)

---

## 4.11 双层生成器架构

Claude Code 的查询系统采用双层生成器架构：

```
QueryEngine（外层）
  └── query()（内层）
```

**QueryEngine**（`src/queryEngine.ts`）是会话级的管理器：
- 维护完整的对话历史（`messages`）
- 管理会话级状态（Token 使用统计、成本追踪、压缩历史）
- 提供 `processUserTurn()` 接口给上层（REPL、Bridge 模式）
- 处理会话恢复和 Worktree 管理

**`query()`**（`src/query.ts`）是单次循环的执行引擎：
- 是一个 `async function*`，通过 `yield` 逐步输出事件
- 管理 API 调用、工具执行、错误恢复
- 包含 7 个精确的继续点（Continue Sites）

这种分层的好处是：**关注点分离**。QueryEngine 不需要知道单次循环的细节（如何处理 PTL 错误、如何并行执行工具），query() 不需要知道会话级的状态（历史消息如何存储、成本如何计算）。

---

## 4.12 七个继续点（Continue Sites）

`query()` 循环有 7 个导致循环继续的位置，每个对应一种恢复策略：

| 继续原因 | 触发条件 | 处理方式 |
|---------|---------|---------|
| `next_turn` | 模型调用了工具 | 正常继续，带上工具结果 |
| `collapse_drain_retry` | PTL 错误 + Context Collapse 有暂存 | 提交折叠，释放 Token，重试 |
| `reactive_compact_retry` | PTL 错误 + Collapse 不够 | 强制全量摘要压缩，重试 |
| `max_output_tokens_escalate` | 输出 Token 不够 | 升级到 64K Token 限制 |
| `max_output_tokens_recovery` | 升级不可用/已用 | 注入续写提示，最多重试 3 次 |
| `stop_hook_blocking` | Stop Hook 阻止终止 | 继续执行 |
| `token_budget_continuation` | Token 预算续写 | 继续生成 |

### PTL（Prompt-Too-Long）恢复流程

```mermaid
flowchart TD
    PTL[PTL 错误发生] --> Phase1[Phase 1: Context Collapse 排水<br/>提交暂存的折叠]
    Phase1 --> Check1{释放了Token?}
    Check1 -->|是| Retry1[重试 API 调用<br/>transition=collapse_drain_retry]
    Check1 -->|否| Phase2[Phase 2: 反应式压缩<br/>强制全量摘要压缩]
    Phase2 --> Check2{压缩成功?}
    Check2 -->|是| Retry2[重试 API 调用<br/>transition=reactive_compact_retry]
    Check2 -->|否| Fail[yield 错误<br/>返回 prompt_too_long]
```

---

## 4.13 错误扣留策略（Withholding）

这是 Claude Code 最巧妙的设计之一：**可恢复的错误不立即 yield 给上层**。

### 工作原理

当出现 `prompt_too_long` 或 `max_output_tokens` 错误时，query() 不会立即通知调用方。它将错误推入 `assistantMessages` 但保留引用，然后运行恢复检查。如果恢复成功，错误**永远不会暴露给调用者**——用户完全感知不到中间的错误。

```typescript
// 错误扣留检测函数
function isWithheldMaxOutputTokens(
  msg: Message | StreamEvent | undefined,
): msg is AssistantMessage {
  return msg?.type === 'assistant' && msg.apiError === 'max_output_tokens'
}
```

### 一个实际场景

假设模型正在编辑一个大文件，生成了 16,000 Token 的输出后被 `max_output_tokens` 截断：

1. **错误发生**：API 返回 `stop_reason: 'max_output_tokens'`
2. **扣留而非暴露**：错误被包装为 `AssistantMessage`，推入消息列表但**不 yield** 给调用方
3. **恢复策略 1 — 升级**：检查是否可以升级到 `ESCALATED_MAX_TOKENS`（64K）。如果可以，直接用更大的 Token 限制重试
4. **恢复策略 2 — 续写**：如果升级不可用，注入一条 meta 用户消息 `"Output token limit hit. Resume directly from where you left off..."` 让模型从断点继续，最多重试 3 次
5. **成功恢复**：如果恢复成功，那条被扣留的错误消息永远不会 yield——用户感知到的是一次流畅的响应

只有当所有恢复尝试都失败时，错误才会被 yield 给上层。

---

## 4.14 Token 使用追踪

QueryEngine 维护完整的 Token 使用统计：

```typescript
totalUsage: {
  input_tokens: 0,
  output_tokens: 0,
  cache_read_input_tokens: 0,
  cache_creation_input_tokens: 0,
  server_tool_use_input_tokens: 0,
}
```

追踪机制：
- 每条 API 响应的 `message_delta` 事件中，`currentMessageUsage` 被更新
- `message_stop` 时，`currentMessageUsage` 通过 `accumulateUsage()` 累加到 `totalUsage`
- `getTotalCost()` 基于 `totalUsage` 和模型定价计算 USD 总成本
- 一旦 `getTotalCost() > maxBudgetUsd`，整个查询终止——这是防止意外高成本的安全机制

`cache_read_input_tokens` 和 `cache_creation_input_tokens` 的追踪对提示词缓存策略至关重要——它们告诉系统缓存是否在有效工作。缓存断裂检测就依赖这些数据来判断是否发生了缓存失效。

---

## 4.15 停止条件

循环在以下条件下终止：

1. **模型未调用工具**：返回纯文本响应，正常结束
2. **达到最大轮次**：`maxTurns` 限制
3. **USD 预算超限**：`getTotalCost() > maxBudgetUsd`
4. **用户中断**：`abortController.signal` 被触发
5. **不可恢复的错误**：PTL/MOT 恢复全部失败
6. **连续压缩失败**：3 次 autocompact 连续失败（熔断器）

> **为什么熔断阈值是 3 次？**
>
> `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`。源码注释引用了生产数据：*"1,279 sessions had 50+ consecutive failures (up to 3,272) in a single session, wasting ~250K API calls/day globally."* 没有这个熔断器之前，压缩一旦进入失败循环，会无限重试。3 次阈值在「给压缩服务恢复机会」和「避免资源浪费」之间取得平衡。

---

## 4.16 设计洞察（深度扩展）

**「异步生成器 vs 回调/事件」的架构选择**：`query()` 是一个 `async function*`，通过 `yield` 逐步输出事件。相比回调模式（如 EventEmitter），生成器有两个关键优势：（1）**背压控制**——消费端不处理完上一个事件，生产端不会继续执行，天然防止事件堆积；（2）**线性控制流**——循环的 7 个 continue site 可以用普通的 `state = { ... }; continue` 表达，不需要状态机的显式转换表。

**「错误扣留」的用户体验设计**：错误扣留不是「隐藏错误」，而是「在用户不需要知道的情况下自动修复错误」。这体现了「用户体验优先」的设计理念——用户关心的是任务是否完成，而不是系统内部发生了什么错误。只有当错误无法自动修复时，才需要告知用户。

**「Token 预算」的安全机制**：`maxBudgetUsd` 限制是一个「防止意外高成本」的安全机制。在 AI 系统中，一个失控的循环可能在几分钟内消耗数百美元的 API 费用。Token 预算机制确保即使系统进入异常状态，成本也有上限。这是「防御性设计」的体现。

---

> 下一章：[上下文工程与压缩 →](#/docs/05-context-compression)
