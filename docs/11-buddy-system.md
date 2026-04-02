# 第 11 章：宠物系统与彩蛋

> **本章目标**：了解 Claude Code 隐藏的宠物系统（Buddy System）、18 种 AI 伴侣，以及工程师藏在代码里的彩蛋。

---

## 先用大白话理解

Claude Code 的工程师在一个没人会仔细看的角落，藏了一套完整的「宠物养成系统」。

这不是玩笑——源码里有完整的 18 种宠物物种定义、稀有度权重、五维属性系统、AI 生成的个性名字，甚至还有撸宠物的动画效果。

而且第一行代码是：`const SALT = 'friend-2026-401'`。2026 年 4 月 1 日，愚人节，源码泄露的第二天。

---

## 18 种 Buddy 物种

| 物种 | 英文 | 特点 |
|------|------|------|
| 猫头鹰 | Owl | 智慧型，调试力高 |
| 狐狸 | Fox | 机敏型，混乱度高 |
| 熊 | Bear | 稳重型，耐心高 |
| 龙 | Dragon | 传说级，全属性高 |
| 独角兽 | Unicorn | 传说级，稀有度极低 |
| 机器猫 | Robo-cat | 技术型，调试力极高 |
| … | … | … |

---

## 宠物的两层数据

Buddy 系统有一个精妙的设计：**骨骼（永不存盘）+ 灵魂（AI 生成后存盘）**。

### 骨骼层（由账号 ID 哈希确定）

```typescript
// 用账号 ID 作为随机种子，确保同一账号永远得到同一只宠物
const SALT = 'friend-2026-401';
const seed = hash(userId + SALT);

const buddy = {
  species: selectSpecies(seed),        // 物种（18 种）
  eyeType: selectEyeType(seed),        // 眼型（8 种）
  hat: selectHat(seed),                // 帽子（12 种，含「无」）
  rarity: calculateRarity(seed),       // 稀有度
  stats: {
    debugPower: rollStat(seed, 'debug'),    // 调试力
    patience: rollStat(seed, 'patience'),   // 耐心
    chaosLevel: rollStat(seed, 'chaos'),    // 混乱度
    wisdom: rollStat(seed, 'wisdom'),       // 智慧
    snarkiness: rollStat(seed, 'snark'),    // 毒舌度
  }
};
// 骨骼数据永远不存盘，每次从账号 ID 重新计算
```

### 灵魂层（AI 生成后存盘）

```typescript
// 第一次孵化时，调用 AI 生成宠物的名字和性格
const soulPrompt = `
你是一只 ${buddy.species}，调试力 ${buddy.stats.debugPower}，
混乱度 ${buddy.stats.chaosLevel}，毒舌度 ${buddy.stats.snarkiness}。
请生成：
1. 一个符合你性格的名字（2-3个字）
2. 一句话性格描述
3. 你最喜欢说的口头禅
`;

const soul = await callAI(soulPrompt);
// 灵魂存进配置文件，永久保存
saveToConfig({ buddySoul: soul });
```

---

## 稀有度分布

| 稀有度 | 概率 | 颜色 |
|--------|------|------|
| 普通 | 60% | 灰色 |
| 稀有 | 25% | 蓝色 |
| 史诗 | 14% | 紫色 |
| 传说 | 1% | 金色 |

---

## 如何触发宠物系统

```bash
# 在 Claude Code 中输入
/buddy          # 孵化/查看你的宠物
/buddy pet      # 撸宠物（触发爱心飘散动画）
/buddy stats    # 查看宠物五维属性
```

宠物会坐在界面角落，偶尔发出气泡评论，评论内容由 AI 根据当前任务上下文生成。

> **注意**：宠物系统目前在公开版本中还没有完全开放，需要特定的 Feature Flag 才能激活。

---

## 加载动画里的 200 个词

`spinnerVerbs.ts` 里有 200+ 个加载动画词汇，大部分是正经词，但夹杂着一批完全不正经的：

```
Beboppin'（踢踏舞）、Boondoggling（瞎折腾）、Canoodling（卿卿我我）、
Gallivanting（到处乱逛）、Jitterbugging（摇摆舞）、Lollygagging（游手好闲）、
Moonwalking（月球漫步）、Shenaniganing（胡闹）、Tomfoolering（耍蠢）、
Wibbling（发抖颤动）……
```

还有一个词：`Clauding`。

这些词没有任何功能意义，只是工程师在一个没人会仔细看的文件里，悄悄放进去的。这是一种态度：**做一个让人在意想不到的地方会心一笑的产品**。

---

## 隐藏命令目录

| 命令 | 说明 | 可用范围 |
|------|------|---------|
| `/ultraplan` | 三角色并行规划（架构师 + 风险分析师 + 任务拆解师） | Anthropic 内部 |
| `/ultrareview` | 深度代码审查，多维度分析 | Anthropic 内部 |
| `/bughunter` | 专门的 Bug 猎人模式 | Anthropic 内部 |
| `/buddy` | 宠物系统 | Feature Flag |
| `/doctor` | 环境诊断 | 公开可用 |

---

> 下一章：[源码泄露的完整故事 →](#/docs/12-leak-story)
