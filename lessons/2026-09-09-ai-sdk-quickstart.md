# W1-3 ｜ Vercel AI SDK Quickstart：你的第一次 LLM 调用

> 课程表来源：curriculum.md · W1 学习条目 ③ ｜ 建议 08:10~09:00 使用
> 前置：昨天的 Key 已放进 `.env` 并通过 `check-key.mjs` 验证。

## 今日目标（3 条）

1. 安装 **AI SDK**（`ai`）和 **OpenAI 兼容 Provider**，理解各自职责；
2. 跑通 `generateText`：发一问、收一答；
3. 看懂返回里的 `usage`，建立 token 消耗的直觉。

## 概念讲解

**AI SDK 是什么？** Vercel 出品的 TypeScript 库，把各家 LLM 的 HTTP API 翻译成统一接口。类比：**axios 之于 HTTP**——你写一次调用逻辑，底层换哪家服务都不用改业务代码。

**为什么选它作为你的主线？** 三个理由：①TypeScript 原生，你的前端技能直接复用；②从命令行脚本到 Next.js 网页是同一套 API 无缝升级（W3 就会用到）；③它目前是 AI 应用层事实标准之一，简历上认。

**`generateText`**：最朴素的调用——把 prompt 发过去，等模型把**完整回答**一次性返回。（边生成边返回的 `streamText` 是 W3 的主题，今天不碰。）

**两个包的分工**：`ai` 是核心（generateText 等方法），`@ai-sdk/openai-compatible` 是适配器（负责跟具体平台的 HTTP API 对话）。国内主流平台（智谱/DeepSeek/Kimi）都提供 OpenAI 兼容接口，所以一个适配器全够用。

## 动手清单

```bash
cd projects/playground
pnpm add ai @ai-sdk/openai-compatible
pnpm add -D tsx     # tsx：直接运行 TypeScript 的工具
```

新建 `src/gen.ts`：

```ts
import 'dotenv/config'
import { generateText } from 'ai'
import { createOpenAICompatible } from '@ai-sdk/openai-compatible'

const zhipu = createOpenAICompatible({
  name: 'zhipu',
  baseURL: 'https://open.bigmodel.cn/api/paas/v4', // 以平台文档为准
  apiKey: process.env.ZHIPU_API_KEY ?? '',
})

const { text, usage } = await generateText({
  model: zhipu('glm-4-flash'), // 模型 ID 以平台「模型列表」文档为准
  prompt: '用一句话向前端工程师解释什么是 LLM。',
})

console.log('回答：', text)
console.log('消耗：', usage) // inputTokens / outputTokens / totalTokens
```

```bash
pnpm tsx src/gen.ts
```

**常见报错对照**：`401` → Key 不对（回看 W1-2）；`404` → 模型 ID 或 baseURL 写错（去平台文档核对）；`could not connect` → 网络/代理问题。

**看懂 `usage`**：inputTokens 是你的问题占多少 token，outputTokens 是回答占多少——**计费按两者之和**，而 output 通常更贵。从今天起养成习惯：每次调用都瞟一眼 usage。

## 自测题（先答，再对答案）

1. `ai` 和 `@ai-sdk/openai-compatible` 两个包各自的职责是什么？
2. `generateText` 的调用是「一次性返回完整回答」还是「边生成边返回」？
3. `usage.totalTokens` 由哪两部分组成？哪部分单价通常更高？
4. 调用报 404，最可能写错的是哪两个东西？
5. 为什么国内平台接 AI SDK 用 `openai-compatible` 这一个适配器就够？

---

## 答案与解析

1. `ai` 提供统一的核心方法（generateText / 后续的 streamText 等）；`@ai-sdk/openai-compatible` 是适配器，负责按目标平台的 HTTP 协议发请求、解析响应。
2. 一次性返回。它对应人的体验是「转圈半天蹦出一整段」；边生成边返回的是 `streamText`（W3 学）。
3. inputTokens + outputTokens；output 更贵（生成比理解算力开销大）。
4. 模型 ID（如 `glm-4-flash` 拼写/版本）和 baseURL（平台接口地址）。
5. 国内主流平台都主动兼容 OpenAI 的接口格式（行业事实标准），所以一个通用适配器即可对接全部，换平台只改 baseURL 和 Key。

## 参考资料

- AI SDK 文档：https://ai-sdk.dev/docs/introduction （英文，配沉浸式翻译看 Getting Started 即可）
- 智谱模型列表（查模型 ID）：https://docs.bigmodel.cn

## 明日预告

W1-4：模型、Provider、Prompt 三个词到底什么关系？为什么「换模型不改代码」是你未来最重要的护城河。
