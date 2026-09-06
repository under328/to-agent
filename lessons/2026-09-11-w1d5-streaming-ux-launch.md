# W1-D5 ｜ 交付体验：流式 UX 与 Sprint 收官

> **前置**：D1~D4 全部验收。
> **今日一句话**：功能对了之后，把「体感」做对——流式输出、三态 UI、可发布的 README。

## 今日验收

- [ ] 周报生成改为**流式**：页面能边生成边渲染（至少 streamText 逐段输出摘要；进阶：streamObject 渐进渲染结构）
- [ ] 三态 UI 完整：生成中（骨架屏/禁用按钮）、失败（可重试+错误信息）、成功
- [ ] README 完成：三行定位 + 截图 + 本地运行方式
- [ ] 第 1 篇文章大纲完成（10 条目录级），存 `journal/2026-09-13-article-outline.md`
- [ ] 全部 commit + push

## 任务卡

### T1 · streamText 流式（15min）

服务端：

```ts
import { streamText } from 'ai'

export async function POST(req: Request) {
  const { input } = await req.json()
  const result = streamText({
    model: llm(process.env.LLM_MODEL ?? 'glm-4-flash'),
    prompt: buildSummaryPrompt(input),
  })
  return result.toTextStreamResponse() // text/event-stream 风格的流式响应
}
```

客户端（自定义场景手动读流比 useChat 更直观——useChat 绑定的是聊天消息协议，周报不是多轮对话）：

```ts
const res = await fetch('/api/summary', { method: 'POST', body: JSON.stringify({ input }) })
const reader = res.body!.getReader()
const decoder = new TextDecoder()
while (true) {
  const { done, value } = await reader.read()
  if (done) break
  setSummary(s => s + decoder.decode(value, { stream: true }))
}
```

### T2 · 三态 UI（15min）

`idle | generating | error | done` 状态机；generating 时按钮禁用 + 骨架；error 展示 message + 重试按钮。错误信息直接透传服务端 `{ error }`（D1 已规范）。

### T3 · README 与文章大纲（10min）

README 三行定位模板：`一句话是什么 / 解决什么问题 / 怎么跑起来`。文章大纲参考：痛点故事（每周五写周报）→ 为什么自己造 → 结构化输出 schema 设计 → tool calling 取舍 → 评测集方法 → 成本账 → 局限与下一步。

## 关键决策与原理

1. **流式的本质是感知延迟（TTFT，首字时间）**：完整生成要 5~10s，但首字 1s 内出现，用户耐心完全不同——前端工程师在流式 UX 上有天然主场优势，面试讲这个是加分项；
2. **手动 reader vs useChat**：useChat 管的是「消息数组」协议，适合聊天；单次生成类功能（周报/摘要）手写 reader 更简单可控——**选库前先看协议是否匹配问题**；
3. **流式中途断错的取舍**：已流出内容是丢弃还是保留？周报场景丢弃重试（内容短）；长文场景保留+续写。提前想清楚，别线上第一次报错时再想；
4. **streamObject 渐进渲染**（进阶）：结构化对象也能流式，每完成一个字段渲染一个——结构与体感兼得，代价是前端要处理「半成品对象」。

## 进阶挑战（选做）

- streamObject 接入：`streamObject({...}).partialObjectStream`，前端随字段到达渐次渲染；
- 生成完成后自动调用 `navigator.clipboard` 复制 Markdown 版。

## 自测题（答完翻底部）

1. 流式输出改善的是真实延迟还是感知延迟？为什么有效？
2. 什么场景该用 useChat，什么场景手动读流？
3. TTFT 是什么？受哪些因素影响？
4. 流式中途报错，保留还是丢弃已输出内容？决策依据是什么？
5. streamObject 相比 streamText 难在哪？

---

## 答案与解析

1. 感知延迟。总耗时不变，但首字时间从「整段生成完」提前到约 1s；人类对「开始响应」极敏感，等待焦虑大幅下降；
2. 多轮对话（协议=消息数组、自动管理历史）用 useChat；单次生成/结构化输出等自定义协议场景手写 reader，避免削足适履；
3. Time To First Token，从请求到首个 token 出现的时长；受网络 RTT、服务端排队、模型 prefill 长度（输入越长越慢）影响；
4. 看内容的「可续性」与重试成本：短内容丢弃重试最简单；长内容丢弃浪费大，考虑保留已确认部分或从断点续传；
5. 要处理「不完整对象」：字段陆续到达时前端需容忍缺字段的中间态渲染，且 schema 校验只能在流结束后做。

## Sprint 收官清单（周末完成）

- [ ] 文章初稿（对着大纲写，别追求完美，3000 字内）
- [ ] 邀 3 位朋友真实使用，反馈原话记进 journal/
- [ ] W2-D1 前把反馈 top3 列出来

## 深入阅读

- AI SDK · Streaming：https://ai-sdk.dev/docs/ai-sdk-core/streaming
- web.dev · Streams API：https://developer.mozilla.org/zh-CN/docs/Web/API/Streams_API
