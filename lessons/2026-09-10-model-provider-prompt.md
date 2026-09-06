# W1-4 ｜ 模型、Provider 与 Prompt：三个核心概念的关系

> 课程表来源：curriculum.md · W1 学习条目 ④ ｜ 建议 08:10~09:00 使用
> 前置：昨天的 `gen.ts` 能跑通。

## 今日目标（3 条）

1. 分清 **Provider / Model / Prompt** 三层概念，不再混用；
2. 理解 `system`（系统提示词）和 `prompt`（用户输入）的分工；
3. 用对照实验感受 `temperature` 参数对输出的影响。

## 概念讲解

用「快递」类比三层结构：

- **Provider（提供商适配层）** = 快递公司网点：负责「对接哪家服务」。它是一层代码适配器（如 `createOpenAICompatible`），把你的调用翻译成目标平台的 HTTP 请求；
- **Model（模型）** = 具体的车型：同一网点下有不同车型（`glm-4-flash`、`glm-4.7`……），能力、速度、价格不同。**模型 ID 由平台文档定义，是字符串**；
- **Prompt（提示词）** = 你填的快递单：`system` 是「对配送员的长期要求」（角色、规则、边界），`prompt`/`messages` 是「这一单具体要送什么」。

**为什么这个分层重要？** 因为 2026 年模型迭代极快，**「换模型不改业务代码」** 是 AI 应用工程师的护城河：Provider 层换 baseURL/Key，业务代码一行不动——这正是 AI SDK 帮你隔离出来的能力。

**`system` 的意义**：把「这个应用是什么人设、有什么规矩」和「用户这次说了什么」分开。以后你做的每个 AI 功能，第一步都是写 system——它是产品的一部分，不是临时对话。

**`temperature`（温度）**：控制随机性。低（趋近 0）→ 稳定、可复现，适合周报生成、信息抽取；高 → 发散、有创意，适合起名、头脑风暴。取值范围随平台不同（常见 0~1 或 0~2），数值越大越发散。

## 动手清单：对照实验

新建 `src/lab.ts`——同一个问题，跑两档温度对比输出：

```ts
import 'dotenv/config'
import { generateText } from 'ai'
import { createOpenAICompatible } from '@ai-sdk/openai-compatible'

const zhipu = createOpenAICompatible({
  name: 'zhipu',
  baseURL: 'https://open.bigmodel.cn/api/paas/v4',
  apiKey: process.env.ZHIPU_API_KEY ?? '',
})

const question = '给一个「把零散工作记录自动整理成周报」的工具想 3 个产品名。'

for (const temperature of [0.1, 0.9]) {
  const { text } = await generateText({
    model: zhipu('glm-4-flash'),
    temperature,
    prompt: question,
  })
  console.log(`—— temperature = ${temperature} ——\n${text}\n`)
}
```

```bash
pnpm tsx src/lab.ts
```

观察：低温两次运行结果趋同、保守；高温更发散、可能有惊喜也可能跑偏。**再把同样的循环跑第二遍**，验证「低温可复现」这个说法。

## 自测题（先答，再对答案）

1. Provider、Model、Prompt 三层分别对应快递类比里的什么？
2. 换一家模型平台（如智谱换 DeepSeek），理论上要改哪几处、不用改哪几处？
3. `system` 提示词和用户 `prompt` 的本质区别是什么？
4. 做一个「从合同里抽取甲乙双方名称」的功能，temperature 应该调高还是调低？为什么？
5. 模型 ID 写错会发生什么？去哪里查正确写法？

---

## 答案与解析

1. Provider=网点（对接哪家服务），Model=车型（具体型号与参数），Prompt=快递单（本次输入内容）。
2. 改：Provider 的 baseURL、apiKey、模型 ID 字符串；不改：业务代码里所有 generateText 调用与 prompt——这就是分层隔离的价值。
3. system 是应用层面的长期设定（人设/规则/输出格式），对每次请求都生效；prompt 是用户当次的临时输入。前者属于产品，后者属于会话。
4. 调低。信息抽取要的是准确、稳定、可复现，发散性只会在「给产品起名」这类创造性任务里是优点。
5. 平台返回 404/模型不存在类错误；查平台官方「模型列表」文档（如 docs.bigmodel.cn），ID 必须逐字符一致。

## 参考资料

- Prompt 工程中文指南：https://www.promptingguide.ai/zh
- AI SDK · settings/provider 概念：https://ai-sdk.dev/docs/foundations/providers-and-models

## 明日预告

W1-5：实验课——改造 system prompt 三种写法，亲手建立「提示词手感」，并写下你的第一篇实验日志。
