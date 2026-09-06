# W1-D2 ｜ 核心引擎：结构化输出——让 LLM 产出「能用的数据」

> **前置**：D1 的线上 Hello-LLM 已跑通。
> **今日一句话**：把「自由文本」升级为「带 schema 校验的结构化数据」——这是 AI 功能从 demo 变成产品的分水岭。

## 今日验收

- [ ] 首页输入框：粘贴零散工作记录 → `POST /api/report` → 页面渲染出结构化周报
- [ ] zod schema 包含：`weekOf`、`summary`、`done[]`（title/detail/tags）、`risks[]`（desc/level 枚举）、`nextPlan[]`
- [ ] 校验/生成失败自动重试 1 次，且**把错误信息喂回给模型**修复
- [ ] 服务端日志打印每次调用的 inputTokens / outputTokens

## 任务卡

### T1 · Schema 设计（10min，先想后写）

```ts
import { z } from 'zod'

const reportSchema = z.object({
  weekOf: z.string().describe('本周起始日期 YYYY-MM-DD'),
  summary: z.string().describe('一句话总结，≤50字'),
  done: z.array(z.object({
    title: z.string(),
    detail: z.string().optional(),
    tags: z.array(z.enum(['开发', '会议', '学习', '其他'])),
  })),
  risks: z.array(z.object({
    desc: z.string(),
    level: z.enum(['高', '中', '低']),
  })),
  nextPlan: z.array(z.string()),
})
```

**设计决策（想清楚再写代码）**：
- `tags`/`level` 用 **enum** 而不是自由 string：枚举 = 前端可枚举渲染 + 统计可聚合，自由文本两头都不讨好；
- `detail` 用 `optional()`：输入里没提就别让模型编——**schema 本身就是反幻觉的第一道闸**；
- 每个字段写 `describe()`：它会进入给模型的提示，等于「自带文档」。

### T2 · generateObject（15min）

```ts
import { generateObject } from 'ai'
import { NoObjectGeneratedError } from 'ai'

const { object, usage } = await generateObject({
  model: llm(process.env.LLM_MODEL ?? 'glm-4-flash'),
  schema: reportSchema,
  temperature: 0,
  prompt: buildReportPrompt(rawInput), // 粘贴的零散记录
})
```

### T3 · 失败重试（10min）

```ts
async function generateWithRetry(input: string, maxRetry = 1) {
  let feedback = ''
  for (let i = 0; i <= maxRetry; i++) {
    try {
      return await generateObject({
        model: llm(process.env.LLM_MODEL ?? 'glm-4-flash'),
        schema: reportSchema,
        temperature: 0,
        prompt: feedback
          ? `${buildReportPrompt(input)}\n\n上次输出未通过校验：${feedback}。请修复后重新输出。`
          : buildReportPrompt(input),
      })
    } catch (e) {
      if (NoObjectGeneratedError.isInstance(e) && i < maxRetry) {
        feedback = e.text?.slice(0, 500) ?? '输出不符合 schema'
        continue
      }
      throw e
    }
  }
  throw new Error('unreachable')
}
```

**决策点**：重试必须**带失败原因**重问（自我修复），原样重问只是再抽一次卡。

### T4 · UI 渲染（10min）

结构化数据渲染成卡片列表；加一个「复制为 Markdown」按钮（客户端把 object 转 MD 文本）。生成周报正好是你周一早上要用的工作流——自己先用起来。

## 关键决策与原理

1. **schema-first 而非 prompt-first**：类型即文档、校验即测试、字段即 UI 契约——三合一；
2. **temperature 0**：信息整理类任务要稳定可复现，不要创意；
3. **成本直觉**：本功能单次约 1k input + 500 output tokens，flash 级模型成本可忽略；同一功能换旗舰模型约贵 30~100 倍——**为功能选模型档位**是核心工程判断；
4. **`describe()` 不是注释**：它真实地进入模型上下文，是提示词工程的一部分。

## 进阶挑战（选做）

- 把 `buildReportPrompt` 抽成独立模块并加一段 3 样例的 few-shot（明天评测时验证它值不值）；
- 记录：同一段输入跑 3 次，对比输出一致率。

## 自测题（答完翻底部）

1. schema 里 enum 和 optional 各自防的是什么问题？
2. `generateObject` 相比 `generateText` + 手写 JSON.parse，解决了什么？
3. 重试时为什么要带上次失败信息？
4. 这个功能为什么 temperature 0 而不是 0.9？
5. 同样的功能，什么时候值得换旗舰模型？

---

## 答案与解析

1. enum 防输出「不可枚举/不可聚合」（下游渲染与统计依赖有限集合）；optional 防「输入里没有信息却被编造」（幻觉）；
2. 把「模型输出 → JSON 解析 → zod 校验」合成一步，解析失败/不符 schema 时抛结构化错误，且部分 Provider 走原生 JSON/工具调用通道，成功率远高于自由文本+祈祷；
3. 模型能看到「错在哪」就能针对性自我修复（如「缺 nextPlan 字段」），盲重试成功率不提升；
4. 信息整理任务的评价标准是准确、稳定、可复现；发散性只在创意任务里有价值；
5. 当 flash 级模型在该任务的评测集上分数不达标、且该功能商业价值足够覆盖成本差时——用评测分数决策，不用感觉。

## 深入阅读

- AI SDK · Structured Output：https://ai-sdk.dev/docs/ai-sdk-core/generating-structured-data
- zod：https://zod.dev
