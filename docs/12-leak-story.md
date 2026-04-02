# 第 12 章：源码泄露的完整故事

> **本章目标**：了解 Claude Code 源码是如何被发现的，以及这个事件背后的技术原理和工程启示。

---

## 72 小时时间线

```
2025年3月某日凌晨 2:14
Chaofan Shou（一位安全研究员）在分析 Claude Code 的 npm 包时，
发现了一个异常：JavaScript bundle 里包含了 Source Map 文件的引用。

↓

Source Map 是什么？
开发时，工程师写的是清晰的 TypeScript 源码。
发布时，代码被压缩混淆成一行难以阅读的 JavaScript。
Source Map 是一个「地图文件」，记录了压缩代码和原始源码的对应关系，
让开发者在调试时能看到原始代码。

↓

问题在哪里？
Source Map 里包含了一个 Cloudflare R2 存储桶的 URL。
Chaofan 访问了这个 URL……
完整的 TypeScript 源码就在那里，没有任何访问控制。

↓

72 小时内
消息在 Twitter/X 上扩散，数十位研究员开始分析源码。
Anthropic 关闭了公开访问，但镜像已经到处都是。

↓

2026年4月1日（愚人节）
工程师在 companion.ts 第一行写下：
const SALT = 'friend-2026-401';
```

---

## 技术原理：Source Map 是什么？

```javascript
// 原始 TypeScript 源码（开发时）
async function executeQuery(
  messages: Message[],
  tools: Tool[]
): Promise<AssistantMessage> {
  const response = await anthropicClient.messages.create({
    model: 'claude-opus-4',
    messages: messages,
    tools: tools,
  });
  return response;
}

// 压缩后的 JavaScript（发布时）
async function a(b,c){return await d.e.create({model:'claude-opus-4',messages:b,tools:c})}

// Source Map 文件（本应只在开发环境使用）
{
  "version": 3,
  "sources": ["src/query.ts"],
  "mappings": "...",  // 压缩代码和原始代码的对应关系
  "sourceRoot": "https://r2.cloudflarestorage.com/claude-code-sources/"
  //                                              ↑ 这个 URL 不应该公开
}
```

---

## 为什么会发生这个错误？

最可能的原因：构建脚本在生产环境构建时，**意外地把开发环境的 Source Map 配置包含进去了**。

这是一个非常常见的工程失误，特别是在构建配置复杂的大型项目中。

```javascript
// vite.config.ts 中的一个配置错误示例
export default {
  build: {
    sourcemap: true,  // ← 这行本应该是 false 或 'hidden'
    // 'true' = 生成 Source Map 并在 bundle 中引用
    // 'hidden' = 生成 Source Map 但不在 bundle 中引用
    // false = 不生成 Source Map
  }
}
```

---

## 工程启示

这个事件对所有做 AI 产品的工程师都有直接的教训：

| 教训 | 具体措施 |
|------|---------|
| 构建产物审查 | 发布前检查 bundle 中是否包含不应公开的 URL 或路径 |
| Source Map 策略 | 生产环境用 `hidden` 而不是 `true`，或完全禁用 |
| 存储桶权限 | 开发资产的存储桶默认私有，不要依赖「没人知道 URL」 |
| 自动化检查 | 在 CI/CD 中加入「敏感信息扫描」步骤 |

---

## 事件的积极影响

尽管是一次安全失误，这次泄露对整个 AI 工程社区产生了积极影响：

- 让外界第一次看到了工业级 AI Agent 的真实实现方式
- 推动了 MCP、多 Agent 协作等概念的传播
- 激励了大量开源 AI 工具的开发
- 本教程的存在，也是这次事件的间接结果

---

> 回到开始：[Claude Code 是什么 →](docs/00-what-is-claude-code.md)
