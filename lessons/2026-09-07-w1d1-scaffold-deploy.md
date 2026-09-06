# W1-D1 ｜ Build Sprint · 作品1奠基：从零到线上 Hello-LLM

> **本周冲刺目标**：周日之前，「AI 周报生成器」MVP 核心链路可演示、线上可访问。
> **前置假设**：Git/TS 熟练，Next.js 生疏也没关系——「让 ZCode 结对生成」本身就是今天的训练内容。
> **版本注意**：代码以 `ai@5` 为准（`pnpm list ai` 查版本）；若装到 v4，个别 API 名不同，以实际版本文档为准。

## 今日验收（全部勾上才算完成）

- [ ] `projects/weekly-report` 初始化完成：Next.js App Router + TypeScript + Tailwind
- [ ] 部署到 Vercel，有一个可访问的线上 URL（周末前完成也可，优先级最低）
- [ ] 页面按钮 → `POST /api/llm-test` → 页面显示一句 LLM 生成的话
- [ ] Key 只存在于 `.env.local`，全仓库 `grep` 不到任何硬编码密钥

## 任务卡

### T1 · 初始化（10min）

```bash
cd projects
pnpm create next-app@latest weekly-report --ts --tailwind --eslint --app --src-dir --import-alias "@/*"
cd weekly-report
pnpm add ai @ai-sdk/openai-compatible
pnpm add -D tsx
```

### T2 · 密钥进服务端（5min）

创建 `.env.local`（Next 约定自动加载，且默认在 `.gitignore` 里）：

```bash
LLM_API_KEY=sk-xxx
LLM_BASE_URL=https://open.bigmodel.cn/api/paas/v4
LLM_MODEL=glm-4-flash
```

**决策点**：抽成 `LLM_BASE_URL / LLM_MODEL / LLM_API_KEY` 三件套而不是写死智谱——换 DeepSeek/Kimi 只改 env，业务代码零改动。这是你上周已有结论的工程化落地。

### T3 · 第一个 API Route + 页面（20min）

`src/app/api/llm-test/route.ts`：

```ts
import { generateText } from 'ai'
import { createOpenAICompatible } from '@ai-sdk/openai-compatible'

const llm = createOpenAICompatible({
  name: 'llm',
  baseURL: process.env.LLM_BASE_URL ?? '',
  apiKey: process.env.LLM_API_KEY ?? '',
})

export async function POST() {
  try {
    const { text } = await generateText({
      model: llm(process.env.LLM_MODEL ?? 'glm-4-flash'),
      prompt: '用一句话鼓励一位正在转型 AI 应用工程师的前端开发者。',
    })
    return Response.json({ text })
  } catch (e) {
    return Response.json({ error: e instanceof Error ? e.message : 'unknown' }, { status: 500 })
  }
}
```

`src/app/page.tsx`：一个按钮 + `useState`，点击 `fetch('/api/llm-test', { method: 'POST' })` 显示返回文本。（让 ZCode 生成这段 UI，你负责读懂并手敲一遍。）

### T4 · 部署（10min，可挪到周末）

GitHub 建仓推送 → Vercel Import → 在 Dashboard 的 Environment Variables 里配置三件套（**注意**：`.env.local` 不会被上传，线上必须单独配）。

## 关键决策与原理（面试也会问）

1. **Key 为什么必须在 route handler（服务端）**：任何出现在客户端 bundle 的东西都能被用户扒出来，`NEXT_PUBLIC_` 前缀的变量等于公开；
2. **为什么用 POST 不用 GET**：该操作有副作用（产生计费），GET 会被浏览器预取/缓存误触发；
3. **`createOpenAICompatible` 在模块顶层创建一次**：复用连接与配置，不要每个请求 new 一个；
4. **错误返回规范化**：永远返回 `{ error }` 结构，前端才有统一处理路径——从第一个接口就养成习惯。

## 进阶挑战（选做）

- 给 route 加 `export const maxDuration = 60`，查一下 Vercel 免费档函数超时是多少秒；
- 把错误处理抽成 `wrap()` 高阶函数，未来所有 API route 复用。

## 自测题（答完翻底部）

1. `NEXT_PUBLIC_` 前缀的环境变量有什么风险？
2. LLM 调用接口为什么选 POST？
3. 部署到 Vercel 后 `.env.local` 会生效吗？正确做法是什么？
4. `createOpenAICompatible` 的 `baseURL` 指向的是什么？
5. 从智谱换到 DeepSeek，需要改动哪几处？

---

## 答案与解析

1. Next 会把这类变量**内联进客户端 bundle**，任何访问者查看源码即可拿到——绝不能放 Key；
2. 有副作用（计费）的操作不应被幂等语义的 GET 承载，避免预取/缓存误触发和 CSRF 面扩大；
3. 不会。`.env.local` 只在本地生效，线上必须在 Vercel Dashboard 配 Environment Variables（区分 Production/Preview）；
4. 目标平台的 **OpenAI 兼容接口根路径**（如 `https://open.bigmodel.cn/api/paas/v4`），SDK 会在其后拼接 `/chat/completions` 等端点；
5. 只改 `.env.local`（或 Vercel 环境变量）里的 `LLM_BASE_URL / LLM_API_KEY / LLM_MODEL` 三个值，代码零改动。

## 深入阅读

- AI SDK · Next.js Quickstart：https://ai-sdk.dev/docs/getting-started/nextjs-app-router
- Vercel · Environment Variables：https://vercel.com/docs/environment-variables
