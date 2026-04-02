# 第 17 章：用户体验与终端渲染

> **本章目标**：理解 Claude Code 如何用 React 构建终端 UI，以及流式输出、Vim 模式等体验细节的实现原理。

---

## 17.1 先用大白话理解

你在终端里使用 Claude Code，看到的那些流畅的动画、实时的流式输出、漂亮的语法高亮——这些不是用 `console.log` 实现的。

Claude Code 在终端里运行了一个**完整的 React 应用**。

这听起来很奇怪，但有充分的理由：终端 UI 的复杂度（流式 Markdown 渲染、虚拟滚动、多种动画状态、权限弹窗、Vim 模式、搜索高亮）已经超过了命令式 ANSI 输出能合理管理的范围。

---

## 17.2 Ink：终端里的 React

Claude Code 使用 **Ink 终端渲染器**（基于 React），核心模块 `src/ink/ink.tsx` 达 251KB。

### 为什么选 React？

在终端中使用 React 看起来像「杀鸡用牛刀」，但如果你了解 Claude Code 的 UI 复杂度，就会理解这个选择的必然性：

**1. 声明式消除 ANSI 状态管理**

终端 UI 的底层是 ANSI 转义序列——颜色、粗体、光标位置都需要手动跟踪。命令式编程需要维护「当前在哪一行、什么颜色是激活的、上一帧哪些区域需要擦除」这些状态，组件间的状态耦合会迅速失控。React 的声明式模型让开发者只需描述「UI 应该长什么样」，渲染器自动处理差异更新。

**2. 组件模型天然支持组合**

`ToolUseLoader`、`SpinnerGlyph`、`PermissionRequest`、`Markdown` 都是独立组件，可以嵌套组合。不需要协调「Spinner 在第几行、权限弹窗弹出时 Spinner 是否要让位」这些位置计算——Flexbox 布局引擎自动处理。

**3. Reconciliation 最小化终端写入**

终端不像浏览器有 GPU 加速渲染——每个字符的写入都是一次 I/O 操作。如果每帧都全量重绘，哪怕只改了一个字符也要重写整个屏幕，会产生明显闪烁。React Reconciler 自动 diff 前后两帧，只更新真正变化的部分。

**4. 复用 React 生态**

Hooks（`useState`、`useMemo`、`useEffect`）、Context（全局状态共享）、Memo（避免不必要渲染）——这些年积累的 React 优化模式直接可用。

代价是 251KB 的自研渲染器代码。但考虑到替代方案——用命令式 ANSI 输出手动管理这个复杂度的 UI——这个代价完全值得。

---

## 17.3 渲染流水线

```mermaid
flowchart LR
    React["React 组件树"] --> Reconciler["React Reconciler\n协调器\nreconciler.ts"]
    Reconciler --> Yoga["Yoga Layout\nFlexbox布局\n→ 终端坐标"]
    Yoga --> Render["renderNodeToOutput\nDOM → Screen Buffer\n63KB"]
    Render --> Diff["Diff Detection\n对比前一帧\n仅更新变化部分"]
    Diff --> ANSI["ANSI Output\n生成终端转义序列\noutput.ts 26KB"]
    ANSI --> Terminal["终端输出"]
```

每个阶段都有明确的职责：

- **React Reconciler**（`reconciler.ts`）：标准 React 协调过程，将组件树的变更转换为对内部 DOM 节点的操作。关键点是它只标记需要更新的节点，不会触碰未变化的部分。
- **Yoga Layout**：终端 UI 和 Web 布局面临同样的问题——内容动态变化、宽度不固定、需要嵌套。Yoga 是 Facebook 开源的 Flexbox 布局引擎（WebAssembly 版本），提供了经过实战检验的布局计算能力。
- **Diff Detection**：Screen Buffer 逐 cell 对比前一帧，只有值或样式真正发生变化的 cell 才会生成 ANSI 输出序列。这是流畅体验的关键——用户在快速滚动或流式输出时不会看到闪烁。
- **Blitting 优化**：对于前一帧中完全没变化的连续行，直接从旧的 Screen Buffer 复制（blit），跳过 cell 级别的比较。这在大量静态内容 + 少量动态内容的场景下效果显著。
- **ANSI Output**（`output.ts`）：将样式化的 cell 转换为终端转义序列，处理 256 色、TrueColor、粗体/斜体/下划线等样式的编码，以及 OSC 8 超链接协议。

---

## 17.4 内存优化：对象池

终端应用与 Web 应用有一个关键区别：它们可能连续运行数小时。一个持续数百轮对话的会话中，短生命周期的字符串和样式对象会给垃圾回收器带来巨大压力。

Screen Buffer（`src/ink/screen.ts`，49KB）借鉴了游戏引擎的**对象池（Object Pooling）**技术，使用三种池来避免重复创建对象：

| 对象池 | 作用 | 优化手段 |
|--------|------|---------|
| CharPool | 重复字符 intern 化 | ASCII 快速路径：直接数组查找（`chars[charCode]`），不需要 Map 查询 |
| StylePool | 重复样式 intern 化 | 位打包存储样式元数据（颜色、粗体等编码到一个整数中） |
| HyperlinkPool | 重复 URL intern 化 | URL 去重，数千个 cell 指向同一个超链接只存一份 |

「Intern 化」的意思是：屏幕上可能有 10000 个 cell 显示相同的白色普通字符 "a"，但它们共享同一个 CharPool 条目，而不是各自创建一个字符串对象。

---

## 17.5 核心组件

| 组件 | 功能 |
|------|------|
| `App.tsx` (98KB) | 根组件，键盘/鼠标/焦点事件分发 |
| `Box.tsx` | Flexbox 布局容器 |
| `Text.tsx` | 样式化文本渲染 |
| `ScrollBox.tsx` | 可滚动容器（支持文本选择） |
| `Button.tsx` | 交互式按钮（焦点/点击） |
| `AlternateScreen.tsx` | 全屏模式 |
| `Ansi.tsx` | ANSI 转义码解析为 React Text |

---

## 17.6 Context 系统

Claude Code 的终端 UI 使用 5 个 React Context 向深层组件树提供全局状态，避免逐层传递 props：

```typescript
AppContext              // 全局应用状态（会话、配置、权限模式）
TerminalFocusContext    // 终端窗口焦点状态（用于暂停/恢复动画）
TerminalSizeContext     // 终端视口尺寸（行×列，响应式布局）
StdinContext            // 标准输入流（键盘事件源）
ClockContext            // 动画时钟（统一调度渲染帧）
```

这些 Context 的设计遵循「终端即浏览器」的理念。例如 `TerminalSizeContext` 在终端窗口尺寸变化时会触发 Yoga 重新计算布局，类似于浏览器中的 `resize` 事件驱动 CSS 重排。`TerminalFocusContext` 则在用户切换到其他窗口时暂停动画和流式输出的渲染，减少不必要的 CPU 开销。

---

## 17.7 自定义 Hooks 库

在 Context 系统之上，Claude Code 封装了一组自定义 Hooks，每个 Hook 封装了终端 I/O 的一个复杂性维度：

```typescript
useInput(handler)           // 全局键盘事件监听（支持 Kitty 扩展键码）
useSelection()              // 文本选择状态管理（选区起止、选中内容）
useSearchHighlight(query)   // 搜索高亮渲染（匹配位置追踪 + 当前焦点）
useAnimationFrame(callback) // 帧调度（与 ClockContext 同步，避免不必要渲染）
useTerminalFocus()          // 终端焦点事件（窗口切换时暂停流式输出）
useTerminalViewport()       // 视口尺寸响应（触发 Yoga 重新布局）
```

**`useAnimationFrame(intervalMs)` 的设计**：所有动画组件（Spinner、Shimmer、Blink）不各自维护定时器，而是订阅同一个 `ClockContext` 提供的时钟源。当 `intervalMs` 为 `null` 时，组件自动取消订阅——这就是暂停的实现方式（终端失去焦点时，`useTerminalFocus()` 返回 false，动画 Hook 将 intervalMs 设为 null）。

**`useBlink(enabled)` 的同步设计**：所有闪烁的元素天然同步，因为它们使用同一个数学公式从共享时钟推导状态：

```typescript
const isVisible = Math.floor(time / BLINK_INTERVAL_MS) % 2 === 0
```

不需要一个「闪烁协调器」来同步多个组件——它们读同一个 `time`，用同一个公式，结果自然一致。BLINK_INTERVAL_MS = 600ms（300ms 亮、300ms 暗）——快到能表示「进行中」，慢到不会让人觉得刺眼。

---

## 17.8 流式输出

Claude Code 的流式输出不是「等完了再显示」，而是**真正的实时流式渲染**。

从 API 到用户终端，整个链路基于 `async function*` 异步生成器：

```
API SSE → callModel() → query() → QueryEngine → REPL → Ink 渲染器
     ↓          ↓            ↓           ↓          ↓
   chunk      yield       yield       yield     React 更新
```

每个 Token 从 API 返回的瞬间就开始渲染，用户可以实时看到模型的「思考过程」。

### 流式事件类型

| 事件类型 | 来源 | 处理 |
|---------|------|------|
| `text_delta` | 模型输出文本 | 追加到当前消息，触发 Markdown 重渲染 |
| `input_json_delta` | 工具调用参数（流式） | 累积 JSON 片段，完成后解析 |
| `thinking_delta` | 扩展思考内容 | 显示在可折叠的「思考」区域 |
| `message_start` | 新消息开始 | 创建新的消息容器 |
| `message_stop` | 消息结束 | 触发工具执行 |

### Markdown 流式渲染

Markdown 渲染面临一个特殊挑战：**流式输入的 Markdown 在任意时刻都可能是不完整的**。

例如，模型正在输出一个代码块：

```
第 1 个 token：```
第 2 个 token：python
第 3 个 token：\n
第 4 个 token：def
...
```

在代码块完整之前，渲染器不知道这是代码块还是普通文本。Claude Code 的 Markdown 渲染器使用**流式解析**策略：

1. 维护一个「未完成块」缓冲区
2. 对于确定完整的块（段落、标题），立即渲染
3. 对于可能未完成的块（代码块、列表），先以「进行中」状态渲染，完成后切换到「完成」状态

---

## 17.9 Vim 模式

Claude Code 支持完整的 Vim 键位——不是「类 Vim」，而是真正的 Normal/Insert/Visual 模式切换。

```typescript
// Vim 模式状态机
type VimMode = 'normal' | 'insert' | 'visual' | 'command'

// 模式切换规则
'i' in Normal → Insert
'Escape' in Insert → Normal
'v' in Normal → Visual
':' in Normal → Command
```

**为什么实现 Vim 模式？**

Claude Code 的目标用户是开发者，而大量开发者使用 Vim/Neovim。在终端中提供 Vim 键位，让这些用户不需要切换肌肉记忆。

**实现挑战**：Vim 模式需要拦截所有键盘事件，在「Vim 输入」和「应用输入」之间路由。`useInput()` Hook 的设计支持这种路由——它接受一个 `isActive` 参数，当 Vim 模式激活时，Vim 处理器的 `isActive` 为 true，其他处理器的 `isActive` 为 false。

---

## 17.10 权限弹窗

当 Claude Code 需要执行危险操作时，会弹出权限确认弹窗：

```
╭─────────────────────────────────────────────────────╮
│ Claude wants to run a command                       │
│                                                     │
│ rm -rf node_modules                                 │
│                                                     │
│ [y] Yes  [n] No  [a] Always allow  [d] Deny always │
╰─────────────────────────────────────────────────────╯
```

这个弹窗是一个 React 组件（`PermissionRequest.tsx`），使用 `AlternateScreen` 覆盖在当前输出之上。

**四个选项的设计**：

| 选项 | 含义 | 持久化 |
|------|------|--------|
| `y` (Yes) | 本次允许 | 不持久化 |
| `n` (No) | 本次拒绝 | 不持久化 |
| `a` (Always allow) | 总是允许此类操作 | 写入 `.claude/settings.json` |
| `d` (Deny always) | 总是拒绝此类操作 | 写入 `.claude/settings.json` |

「Always allow」和「Deny always」的选择会被持久化到权限规则中，下次遇到相同类型的操作时不再询问。

---

## 17.11 设计洞察

**「终端即浏览器」的设计哲学**：Claude Code 把终端当成一个功能受限的浏览器来对待——它有「DOM」（Yoga 节点树）、「CSS」（Flexbox 布局）、「事件系统」（键盘/鼠标事件）、「渲染引擎」（Ink Reconciler）。这种类比让 Web 开发者能够用熟悉的思维模型理解终端 UI 的实现。

**对象池是「长寿命应用」的必要优化**：Web 应用通常在用户刷新页面时重置内存。终端应用可能连续运行数小时，垃圾回收压力完全不同。对象池（CharPool、StylePool、HyperlinkPool）是从游戏引擎借鉴的技术，在 GC 压力大的场景下效果显著。

**统一时钟的优雅性**：所有动画组件共享同一个 `ClockContext` 时钟源，通过数学公式推导状态而不是各自维护定时器。这不仅保证了动画同步，也让「暂停所有动画」变成了一个简单的操作——只需停止时钟，所有订阅者自动停止。

---

> 下一章：[宠物系统与彩蛋 →](#/docs/11-buddy-system)

---

## 17.12 工具状态可视化

每个工具调用前面都有一个状态指示器——一个小圆点 `●`，通过颜色和闪烁编码状态：

| 状态 | 颜色 | 动画 | 含义 |
|------|------|------|------|
| 未完成 + 动画中 | 暗淡 | 闪烁（600ms 周期） | 正在执行 |
| 未完成 + 排队中 | 暗淡 | 静止 | 等待执行 |
| 成功完成 | 绿色 | 静止 | 已完成 |
| 出错 | 红色 | 静止 | 执行失败 |

当多个工具并行执行时，所有「正在执行」的圆点会**同步闪烁**——它们在同一瞬间亮起或熄灭。这不是靠一个「闪烁协调器」实现的，而是利用了 `useBlink` Hook 的数学同步：所有组件共享同一个时钟源，通过 `Math.floor(Date.now() / 600) % 2` 计算当前帧，保证完全同步。

同步闪烁给用户一种「系统在统一运作」的感觉，比各自独立闪烁更有秩序感。

---

## 17.13 Diff 渲染系统

当工具执行文件编辑时，Claude Code 会渲染一个类似 `git diff` 的差异视图，让用户在确认前看到具体改动。

差异计算基于 `structuredPatch`：

```typescript
structuredPatch(filePath, filePath, oldContent, newContent, {
  context: 3,      // 改动前后各展示 3 行上下文（与 git diff 一致）
  timeout: 5000    // 防止极端 diff 计算阻塞
})
```

渲染使用 `StructuredDiffList` 组件：删除行红色、新增行绿色、上下文行灰色，并附带行号。代码块支持语法高亮（通过 `cli-highlight` + `highlight.js`，支持 180+ 种语言）。

对于大文件，不会加载整个文件到内存，而是根据编辑位置分块读取，只加载改动周围的上下文区域。Diff 组件使用 React `Suspense` 模式——异步加载文件内容和计算 patch 期间显示 `"…"` 占位符，加载完成后替换为完整 diff 视图。

---

## 17.14 错误处理与自动恢复

Claude Code 的错误处理策略是「尽可能自动恢复，实在不行才告诉用户」。

### 重试策略详解

核心重试参数：

| 参数 | 值 | 说明 |
|------|-----|------|
| DEFAULT_MAX_RETRIES | 10 | 最大重试次数 |
| BASE_DELAY_MS | 500 | 基础延迟 |
| MAX_529_RETRIES | 3 | 连续 529 错误触发降级的阈值 |

退避策略采用**指数退避 + 随机抖动**：

| 重试次数 | 延迟（约） |
|---------|-----------|
| 第 1 次 | 500ms |
| 第 2 次 | 1s |
| 第 3 次 | 2s |
| 第 4 次 | 4s |
| 第 7+ 次 | 32s（上限封顶） |

每次延迟额外叠加 ±25% 的随机抖动，防止多个客户端在同一时刻重试造成「惊群效应」。

### 前台 vs 后台查询：避免级联放大

**不是所有查询都值得重试**。只有用户直接等待结果的前台查询才重试 529：

```typescript
const FOREGROUND_529_RETRY_SOURCES = new Set([
  'repl_main_thread',    // 用户在等模型回复
  'sdk',                 // SDK 调用
  'agent:default',       // Agent 子任务
  'compact',             // 上下文压缩
  // ...
])
```

后台查询——摘要生成、标题建议、命令补全建议——**在收到 529 时立即放弃，不重试**。理由：在容量紧张期间，每次重试都是对 API 网关的 3-10 倍放大。后台查询对用户不可见，放弃后台查询，把容量留给前台查询。

### 连接恢复

长时间运行的会话中，HTTP keep-alive 连接可能在服务端超时失效。当客户端试图在已失效的连接上发送请求时，会收到 `ECONNRESET` 或 `EPIPE` 错误。

Claude Code 的处理：

```
ECONNRESET / EPIPE 检测
  → disableKeepAlive()    // 禁用连接池复用
  → 获取新的 client 实例  // 建立全新连接
  → 重试请求
```

这是一个「自我修复」的设计：第一次 ECONNRESET 失败后，后续所有请求都使用新连接，不会重复遇到同样的问题。

### 用户无感知的自动恢复

| 错误类型 | 自动恢复策略 |
|---------|------------|
| PTL（Prompt Too Long） | 解析错误消息提取超出量，触发上下文压缩 |
| Max-Output-Tokens | 自动升级 Token 限制或注入续写提示 |
| API 5xx | 指数退避重试（最多 10 次） |
| ECONNRESET | 禁用 Keep-Alive + 新连接重试 |
| OAuth 过期 | 检测 401 → 自动刷新 Token → 新 client 重试 |

---

## 17.15 模型降级通知

当连续 3 次 529 错误触发模型降级时：

```
连续 3 次 529 → 抛出 FallbackTriggeredError(originalModel, fallbackModel)
  → 清除之前的 assistant 消息（避免降级模型看到高级模型的输出格式）
  → 剥离思考签名块（降级模型可能不支持）
  → yield 系统消息告知用户降级
  → 用降级模型重试
```

用户会看到一条系统消息说明模型已降级，但不需要任何操作。

---

## 17.16 进度消息流

工具在执行过程中可以通过 `yield { type: 'progress', content }` 发射进度事件（例如 Bash 工具流式输出 stdout/stderr）。这些事件沿着流式管线一路传递到 REPL 组件，通过 `renderToolResultMessage(content, progress)` 渲染为工具输出区域的实时更新。

这意味着用户运行一个 `npm install` 时，不是等 30 秒后一次性看到全部输出，而是实时看到每一行包安装日志。这种即时反馈大幅降低了「Agent 在干嘛？为什么这么久？」的焦虑感。

---

## 17.17 设计洞察（扩展）

**「负载感知的重试策略」**：前台/后台查询的差异化重试策略，是一种「系统自我保护」机制。在容量紧张时，主动放弃非关键查询，把资源留给关键路径。这个设计原则在分布式系统中被称为「load shedding」（负载卸载），是高可用系统的标准实践。

**「自我修复」的连接管理**：ECONNRESET 处理的「禁用 Keep-Alive + 新连接」策略，是一个「一次失败，永久修复」的设计。它不是每次失败都尝试修复，而是在第一次失败后改变策略，避免重复遇到同样的问题。这种「从失败中学习」的设计，比「无限重试」更优雅。

**「同步闪烁」的视觉语言**：多个工具并行执行时的同步闪烁，是一个微妙但重要的 UX 设计——它向用户传达「这些操作是协调一致的，不是各自独立的」。这种视觉语言的设计，需要对用户的认知模型有深刻理解。

---

> 下一章：[宠物系统与彩蛋 →](#/docs/11-buddy-system)

---

## 17.18 Vim 模式

`src/vim/` 实现了终端输入的 Vim 键绑定（总计约 40KB），使习惯 Vim 的用户可以在 Claude Code 的输入框中使用熟悉的编辑模式。

### 四模式状态机

```mermaid
stateDiagram-v2
    [*] --> Normal
    Normal --> Insert: i, a, o, A, I, O
    Insert --> Normal: Escape
    Normal --> Visual: v, V
    Visual --> Normal: Escape
    Normal --> Command: :
    Command --> Normal: Escape, Enter
```

- **Normal 模式**：默认模式，用于导航和操作组合
- **Insert 模式**：文本输入模式，行为与普通编辑器一致
- **Visual 模式**：文本选择模式，支持字符选择（`v`）和行选择（`V`）
- **Command 模式**：命令行模式，通过 `:` 进入

### 操作符、移动和文本对象

Vim 的核心语法是「操作符 + 移动/文本对象」的组合：

| 操作符 | 键 | 功能 |
|--------|-----|------|
| Delete | d | 删除（dw=删除单词, dd=删除行） |
| Yank | y | 复制（yw=复制单词, yy=复制行） |
| Change | c | 修改（cw=修改单词, cc=修改行） |
| Paste | p/P | 粘贴（p=光标后, P=光标前） |

文本对象分为 `inner`（内部）和 `a`（包含分隔符）两种：

| 文本对象 | 键 | 说明 |
|----------|-----|------|
| inner word | iw | 单词内部（不含空格） |
| a word | aw | 整个单词（含尾随空格） |
| inner quotes | i", i' | 引号内部内容 |
| inner parens | i(, i{ | 括号/花括号内部 |

操作符、移动和文本对象三者可以自由组合：`diw` = 删除单词内部，`ci"` = 修改引号内的内容，`ya{` = 复制花括号内（含花括号）的内容。

---

## 17.19 虚拟消息列表

对于一个可能持续数百轮的对话，如果同时渲染所有消息，会面临严重的性能问题：每个 `MessageRow` 需要 Yoga 布局计算、Markdown 解析、可能的语法高亮——全量渲染数百条消息意味着数秒的渲染时间和持续增长的内存占用。

`useVirtualScroll`（`src/hooks/useVirtualScroll.ts`）的方案是：**只挂载视口可见范围 + 上下缓冲区内的消息**，其余用空白 Spacer 占位保持滚动高度。

关键常量的设计推理：

| 常量 | 值 | 为什么 |
|------|-----|--------|
| `DEFAULT_ESTIMATE` | 3 行 | **故意偏低**。高估会导致空白，低估只是多挂载几条消息，代价很小 |
| `OVERSCAN_ROWS` | 80 行 | **很宽裕**。真实消息高度可以是估计值的 10 倍（一个长工具输出可能占 30+ 行） |
| `SCROLL_QUANTUM` | 40 行 | 量化 `scrollTop`，避免每个滚轮 tick 都触发完整的 React commit |
| `SLIDE_STEP` | 25 项 | **每次 commit 最多新挂载 25 项**，避免一次挂载 194 项造成 290ms 同步阻塞 |
| `MAX_MOUNTED_ITEMS` | 300 项 | React fiber 分配的硬上限，防止极端情况下内存爆炸 |

终端 resize 时的处理：不是清空缓存重新测量（那会导致约 600ms 的渲染高峰），而是按列数比例**缩放**已缓存的高度。缩放值不完全精确，但在下一次 Yoga 布局时会被真实高度覆盖。

---

## 17.20 权限确认的 200ms 防误触

`PermissionRequest` 的 200ms 防误触不是一个「nice to have」——它是一个**安全关键设计**。

场景：用户正在快速输入一段话。Agent 此时决定执行一个 Bash 命令，弹出了权限确认对话框。如果对话框弹出后立即响应按键，用户的下一个 Enter（本意是换行或提交消息）就会被解读为「Allow」——意外地批准了一个可能修改文件系统的操作。

200ms 的设计依据：人类快速打字的击键间隔通常在 50-150ms 之间。200ms 的延迟确保用户已经停止打字动作（视觉上注意到弹窗出现），然后才开始接受输入。这个值不能太长（否则影响想要快速确认的用户），也不能太短（否则无法防止误触）。

---

## 17.21 终端协议支持

`src/ink/termio/` 处理底层终端协议，支持多种高级特性：

| 特性 | 协议 | 说明 |
|------|------|------|
| 超链接 | OSC 8 | 可点击的链接 |
| 鼠标追踪 | Mode-1003/1000 | 移动/点击事件 |
| 键盘 | Kitty Protocol | 扩展键码 |
| 文本选择 | 自定义 | 单词/行吸附 |
| 搜索高亮 | 自定义 | 带位置追踪 |
| 双向文本 | bidi.ts | RTL 语言支持 |

**Kitty 键盘协议**：得益于 Kitty 键盘协议，Claude Code 能区分传统终端无法分辨的按键组合（如 Ctrl+Shift+A 与 Ctrl+A），提供更精细的快捷键支持。

**和弦快捷键（Chord）**：多键组合，如 `ctrl+k ctrl+s` 需要依次按下两组按键才触发。这借鉴了 VS Code 的设计。终端环境下可用的键组合远比 GUI 应用少（很多组合被终端仿真器或 shell 占用），和弦机制通过序列组合扩展了可用的快捷键空间。

---

## 17.22 REPL 主界面

`src/screens/REPL.tsx`（895KB）是整个应用的主要交互界面。它集成了：

- 流式消息处理（`handleMessageFromStream`）
- 工具执行编排
- 权限请求处理（`PermissionRequest` 组件）
- 消息压缩（`partialCompactConversation`）
- 搜索历史（`useSearchInput`）
- 会话恢复和 Worktree 管理
- 后台任务协调
- 成本追踪和速率限制
- 虚拟滚动（`VirtualMessageList`）

### 核心依赖组件

| 组件 | 功能 |
|------|------|
| `Messages` | 对话历史渲染（支持 Markdown、代码高亮、工具调用展示） |
| `PromptInput` | 用户输入控件（多行编辑、自动补全、Vim 模式切换） |
| `VirtualMessageList` | 虚拟滚动（只渲染可见区域，支持数百条消息） |
| `PermissionRequest` | 权限确认 UI（Allow/Deny 按钮 + 200ms 防误触） |

---

## 17.23 设计洞察（深度扩展）

**「虚拟滚动」的不对称误差选择**：`DEFAULT_ESTIMATE = 3 行` 故意偏低，因为高估和低估的代价不对称。高估会导致视觉空白（用户体验差），低估只是多挂载几条消息（性能损耗小）。在设计估算系统时，选择「代价更小的误差方向」是一种重要的工程智慧。

**「200ms 防误触」的人机工程学**：这个设计体现了对用户行为的深刻理解——人类的打字节奏（50-150ms 击键间隔）和视觉注意力切换（需要 100-200ms 感知弹窗出现）是可以被量化的。将这些人机工程学数据转化为具体的系统参数，是「以用户为中心」设计的体现。

**「Vim 模式」的用户分层**：Vim 模式的实现体现了「用户分层」的设计思想——大多数用户不需要 Vim 模式，但对于习惯 Vim 的用户来说，它是不可或缺的效率工具。通过可选的 Vim 模式，Claude Code 同时服务了两类用户，而不是为了「简洁」而牺牲高级用户的体验。

---

> 下一章：[宠物系统与彩蛋 →](#/docs/11-buddy-system)
