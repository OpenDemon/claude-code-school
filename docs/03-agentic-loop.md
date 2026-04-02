# 第 3 章：代理循环——AI 的心跳

> **本章目标**：理解 Claude Code 最核心的运行机制——「思考 → 行动 → 观察」的无限循环。

---

## 先用大白话理解

想象一个厨师接到一道菜的订单：「做一份红烧肉」。

一个**普通厨师**会这样做：想好所有步骤，一次性列出来，然后开始做。

一个**有经验的厨师**会这样做：先焯水，尝一下，加调料，再尝，调整，继续炖，再尝……每一步都根据上一步的结果来决定下一步。

Claude Code 就是第二种厨师。它不是「想好了再做」，而是「做一步，看结果，再做下一步」。这个「做一步、看结果、再做下一步」的循环，就是**代理循环（Agentic Loop）**。

---

## 循环的本质

代理循环的核心逻辑极其简单，用伪代码表示就是：

```
while (任务未完成) {
    思考：下一步应该做什么？
    行动：调用工具执行操作
    观察：看工具返回了什么结果
    把结果加入记忆，继续循环
}
```

这个循环在源码 `src/query.ts` 里实现，是整个系统最重要的文件（1,728 行）。

---

## 循环的完整流程

```mermaid
flowchart TD
    Start([用户输入]) --> Build[构建上下文\n历史 + 系统提示词 + 工具列表]
    Build --> Check{上下文是否\n接近限制?}
    Check -->|是| Compress[触发压缩\n4 级流水线]
    Compress --> API
    Check -->|否| API[调用 Anthropic API\n流式接收响应]
    API --> Parse{解析响应\n类型}
    Parse -->|纯文本| Display[显示给用户\n循环结束]
    Parse -->|工具调用| Security{安全检查\n5 层防御}
    Security -->|拒绝| Reject[拒绝执行\n告知原因]
    Reject --> API
    Security -->|需要确认| Confirm[弹出确认框\n等待用户]
    Confirm -->|用户拒绝| Reject
    Confirm -->|用户同意| Execute
    Security -->|自动通过| Execute[执行工具\n获取结果]
    Execute --> AddHistory[结果加入对话历史]
    AddHistory --> Build

    style Start fill:#e8f5e9
    style Display fill:#e8f5e9
    style Security fill:#ffebee
    style Compress fill:#e3f2fd
    style Execute fill:#fff3e0
```

---

## 源码核心片段

`query.ts` 的核心循环结构（简化版）：

```typescript
// src/query.ts
async function* query(
  messages: Message[],
  tools: Tool[],
  options: QueryOptions
): AsyncGenerator<AssistantMessage | ToolResult> {

  while (true) {
    // 1. 检查是否需要压缩上下文
    if (shouldCompress(messages)) {
      messages = await compress(messages);
    }

    // 2. 调用 API，流式接收响应
    const response = await callAPI(messages, tools);

    // 3. 解析响应
    for await (const chunk of response) {
      yield chunk; // 实时流式输出给用户
    }

    // 4. 如果没有工具调用，任务完成，退出循环
    if (!response.hasToolUse) {
      break;
    }

    // 5. 执行所有工具调用
    const toolResults = await executeTools(response.toolUses);

    // 6. 把工具结果加入历史，继续循环
    messages = [...messages, response, ...toolResults];
  }
}
```

注意这是一个 `async function*`（异步生成器）。这意味着每生成一个 Token，就立刻 `yield` 出去显示给用户，而不是等全部生成完再显示。

---

## 工具预执行：为什么感觉这么快？

Claude Code 有一个聪明的优化：**在 AI 还在「说话」的时候，就开始执行工具了**。

普通做法：
```
AI 生成完整响应（5-30 秒）→ 解析工具调用 → 执行工具（1 秒）→ 显示结果
```

Claude Code 的做法：
```
AI 开始生成响应 → 检测到工具调用标记 → 立刻开始执行工具
                                        ↓（并行）
                   AI 继续生成剩余文本（5-30 秒）
                                        ↓
                   工具执行完成（1 秒）→ 结果已就绪
```

利用模型生成的 5-30 秒窗口，把约 1 秒的工具延迟完全隐藏了。

---

## 循环的终止条件

循环在以下情况下退出：

| 情况 | 说明 |
|------|------|
| AI 不再调用工具 | 正常完成，AI 认为任务已完成 |
| 达到最大轮次限制 | 防止无限循环，默认上限约 200 轮 |
| 用户主动中断（Ctrl+C） | 立即停止，清理资源 |
| 发生不可恢复的错误 | API 错误、权限错误等 |
| 上下文压缩失败 | 极端情况，无法继续 |

---

## 并行工具执行

当 AI 在一次响应中调用多个工具时，Claude Code 会智能地决定哪些可以并行执行：

```typescript
// 只读工具可以并行执行（不会互相影响）
const readOnlyTools = toolCalls.filter(t => t.tool.isReadOnly);
const writeTools = toolCalls.filter(t => !t.tool.isReadOnly);

// 并行执行所有只读工具
const readResults = await Promise.all(
  readOnlyTools.map(t => executeToolCall(t))
);

// 串行执行写操作工具（防止冲突）
const writeResults = [];
for (const toolCall of writeTools) {
  writeResults.push(await executeToolCall(toolCall));
}
```

这个设计让多文件读取操作可以同时进行，大幅提升速度。

---

## 你学到了什么

代理循环是 Claude Code 最核心的设计。它的本质是一个 `while(true)` 循环，每次迭代包含「思考 → 行动 → 观察」三步。工具预执行、并行工具调度、流式输出这三个优化，让这个循环既快速又流畅。

---

> 下一章：[对话引擎深度解析 →](docs/04-query-engine.md)
