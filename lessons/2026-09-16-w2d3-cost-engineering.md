# W2-D3 ｜ 成本工程 v1：模型路由与缓存

> **前置**：D1/D2 已验收。
> **今日一句话**：AI 应用的毛利率是在架构期决定的——今天给作品 1 装上「路由 + 缓存」两台省油器。

## 今日验收

- [ ] 模型路由：按任务难度分级调用不同档位模型（如摘要/正文用主力模型，tag 归类用 flash 级）
- [ ] exact 缓存：相同输入命中缓存直接返回（不调 API），缓存层独立成模块
- [ ] **prompt 版本号参与缓存 key**——prompt 改版自动失效，绝不返回旧版 prompt 的结果
- [ ] 成本日志：每次调用记录 `{ route, model, inputTokens, outputTokens, cached }`，提供 `pnpm cost` 汇总今日账单

## 任务卡

### T1 · 模型路由（15min）

```ts
const ROUTES = {
  'tag-classify': { model: env('LLM_MODEL_FLASH'), maxOutput: 100 },   // 枚举归类，flash 足够
  'report-core':  { model: env('LLM_MODEL_MAIN'),  maxOutput: 1500 },  // 摘要与归纳，用主力
} as const

type RouteName = keyof typeof ROUTES
```

**路由判断标准（面试可直接引用）**：任务类型（分类/抽取 → 小模型；多约束生成/长上下文归纳 → 大模型）+ 评测分数（小模型在该子任务评测达标就用小模型）。**路由表里每条路由都应有评测分数背书**——这把 D2 的评测体系变成了成本决策的依据，两个模块咬合成一个系统。

### T2 · exact 缓存（15min）

```ts
import { createHash } from 'crypto'
import { readFileSync, writeFileSync, existsSync } from 'fs'

const PROMPT_VERSION = 'v2.1' // 每次改 prompt 必须升版本

function cacheKey(input: string, route: RouteName) {
  const normalized = input.replace(/\s+/g, ' ').trim() // 规范化空白，提高命中率
  return createHash('sha256').update(`${PROMPT_VERSION}|${route}|${normalized}`).digest('hex')
}
// 命中：读缓存文件/SQLite；未命中：调用后写回
```

**决策点**：为什么 `PROMPT_VERSION` 必须进 key——改了 prompt 不升版本 = 缓存返回旧逻辑结果，这类 bug 线上极难排查。版本号是「逻辑变更」的显式声明。

### T3 · 成本日志（10min）

每次调用 append 一行 JSONL（`logs/cost-2026-09-16.jsonl`），`pnpm cost` 读取汇总：

```
route         calls  cached  tokens(in/out)      est.cost
report-core       7       2     9.2k / 4.1k       ¥0.13
tag-classify     14       9     2.0k / 0.3k       ¥0.00
```

**原理**：看不见的成本管不住。这个 10 分钟写的脚本，会让你对「每个功能值多少钱」建立直觉——W8 的成本看板就是它的放大版。

## 关键决策与原理

1. **成本优化的顺序**：先砍调用量（缓存/合并请求）→ 再降单次单价（路由小模型）→ 最后才谈压缩 prompt。顺序反了白费劲；
2. **exact 缓存的命中原题**：周报场景同一份 git log 一天内会重复生成（改了又改）——这正是缓存价值最高的模式：**用户会重试的功能，缓存收益指数级放大**；
3. **规范化提高命中**：尾部空格、换行差异不应导致缓存 miss；但注意别过度规范化（语义缓存才解决「换了说法」的命中，W8 讲）；
4. **缓存有效期**：git log 输入以「天」为天然周期，`key` 里带上日期即可实现日级失效，无需 TTL 系统。

## 进阶挑战（选做）

- 缓存命中时在 UI 上标注「来自缓存」（用户知情权 + 调试便利）；
- 统计「缓存节约的金额」累计展示——这个数字将来就是文章里的素材。

## 自测题（答完翻底部）

1. 成本优化的正确顺序是什么？为什么？
2. 为什么 PROMPT_VERSION 必须参与缓存 key？
3. 判断「子任务能否路由到小模型」的依据是什么？
4. exact 缓存为什么先做输入规范化？
5. 本项目的缓存如何实现「日级失效」而不用引入 TTL？

---

## 答案与解析

1. 先减量（缓存/合并）、再降单价（路由）、最后压缩 prompt——减量是一次性架构收益，单价受限于模型市场价，而 prompt 压缩收益最小且可能伤质量；
2. prompt 就是「程序逻辑」，改版不升版本等于部署新代码却保留旧二进制缓存——用户拿到新旧混合的结果，且无法复现排查；
3. 该子任务的独立评测集分数达标（如 tag 归类准确率 ≥ 某阈值）即可路由小模型——用 D2 的评测体系给路由决策背书；
4. 用户输入的空白/换行差异是无关噪声，规范化能把「语义相同的输入」合并到同一 key，显著提高命中率且零风险；
5. 把输入素材的日期（如 git log 的 --since 起点）编进缓存 key：新的一天输入必然变化，key 天然不同，等效于按日失效。

## 深入阅读

- AI SDK · Prompt Caching（平台级缓存能力）：https://ai-sdk.dev/docs/ai-sdk-core/caching
- OpenAI · Prompt Caching 指南（思路通用）：https://platform.openai.com/docs/guides/prompt-caching
