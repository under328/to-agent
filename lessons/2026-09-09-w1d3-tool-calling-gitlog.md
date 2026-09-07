# W1-D3 ｜ 数据源升级：git log 采集与 Tool Calling 的工程落地

> **前置**：D2 的结构化周报已跑通。
> **今日一句话**：你已懂 Agent 的工具循环，今天只补两件事——AI SDK 的 tool 落地细节，和「什么时候不该用 tool」的工程判断。

## 今日验收

- [ ] `/api/report` 支持 `source: 'paste' | 'git'`：git 模式读取指定仓库最近 7 天提交作为输入
- [ ] 提供一个可选 tool `searchCommits(keyword)`，模型可按需检索历史提交
- [ ] 安全边界：仓库路径白名单（env 配置）、`execFile` 而非 `exec`、3s 超时
- [ ] 两种 source 生成的周报都能通过 D2 的 schema 校验

## 任务卡

### T1 · 最重要的工程判断先做（10min，想清楚再动手）

**确定性流程 vs 模型自主决策**：

- 「读取最近 7 天 git log」是**确定性流程**——输入固定、步骤固定，直接写普通函数在生成前调用。用 tool 反而增加不确定性和延迟；
- 「按关键词检索历史提交」**是否需要、要搜什么词由模型判断**——这才是 tool 的用武之地。

> 面试高频：「你们哪些地方用了 tool calling？」——能答出「我们评估后大部分场景用确定性函数，只有 X 用了 tool」的候选人，比「全都用 Agent」的高一个段位。

### T2 · git log 采集器（15min）

```ts
import { execFile } from 'child_process'
import { promisify } from 'util'
const exec = promisify(execFile)

const ALLOWED_REPOS = (process.env.ALLOWED_REPOS ?? '').split(',') // 绝对路径白名单

export async function readGitLog(repoPath: string, days = 7) {
  if (!ALLOWED_REPOS.includes(repoPath)) throw new Error('repo not allowed')
  const since = new Date(Date.now() - days * 864e5).toISOString()
  const { stdout } = await exec(
    'git',
    ['log', `--since=${since}`, '--pretty=format:%h|%ad|%s', '--date=short'],
    { cwd: repoPath, timeout: 3000 },
  )
  return stdout.split('\n').filter(Boolean).map(l => {
    const [hash, date, subject] = l.split('|')
    return { hash, date, subject }
  })
}
```

把结果拼进 prompt（`source=git` 分支），走 D2 的同一套 `generateWithRetry`。

### T3 · 可选 tool：searchCommits（15min）

```ts
import { z } from 'zod'
import { tool, stepCountIs } from 'ai'
import { generateText } from 'ai'

const result = await generateText({
  model: llm(process.env.LLM_MODEL ?? 'glm-4-flash'),
  tools: {
    searchCommits: tool({
      description: '按关键词搜索该仓库的历史提交信息，返回 hash/日期/说明。当用户的问题涉及具体功能或模块时使用。',
      inputSchema: z.object({ keyword: z.string().describe('提交信息关键词，如 login、性能') }),
      execute: async ({ keyword }) => searchCommits(repoPath, keyword), // git log --grep
    }),
  },
  stopWhen: stepCountIs(5), // ai@4 旧名 maxSteps
  prompt: '本周登录模块做了哪些事？帮我把相关提交整理成周报要点。',
})
```

### T4 · 安全三件套（10min）

路径白名单（env 注入，永不信前端传来的路径）、`execFile` 数组参数（杜绝 shell 注入）、timeout 兜底。**给 LLM 用的任何工具都要当作「会被恶意调用的公网接口」来设防**——模型可能被提示注入诱导。

## 关键决策与原理

1. **tool 的 description 是给模型看的 API 文档**：写清「何时该用、输入是什么」，调用准确率直接由它决定；
2. **`stopWhen: stepCountIs(5)`**：限制工具循环轮次，防模型陷入无限「调用→观察」循环（成本与延迟的保险丝）；
3. **tool execute 永远在服务端跑**：和 Key 一样，工具能触达的文件系统/网络就是你的信任边界；
4. **提示注入意识**：git commit message 是用户可控文本，理论上可被构造诱导模型——本场景风险低，但要建立「工具返回内容也是不可信输入」的认知。

## 进阶挑战（选做）

- 多仓库聚合：`ALLOWED_REPOS` 支持多个仓库并行读取后合并（Promise.all）；
- 给 `readGitLog` 加「空提交周」的兜底文案（真实使用必遇到）。

## 自测题（答完翻底部）

1. 判断「直接函数调用 vs tool calling」的标准是什么？
2. tool 的 description 影响什么？写好它的要领？
3. 为什么用 `execFile` 不用 `exec`？
4. `stopWhen: stepCountIs(5)` 防的是什么问题？
5. 为什么「工具的返回值」也要当作不可信输入？

---

## 答案与解析

1. 流程是否确定：确定（步骤/参数可静态写死）→ 普通函数；需要模型根据上下文自主决定「要不要调、传什么参数」→ tool。能用确定性代码解决的不用 tool；
2. 影响模型选择与传参的准确率。要领：说清用途、适用场景、参数含义，像写一份给新同事的接口文档；
3. `exec` 把整条命令交给 shell 解释，拼接用户输入可构成命令注入；`execFile` 参数走数组，不经过 shell 解释；
4. 防模型无限工具循环（反复调用不收敛），它同时是成本和延迟的保险丝；
5. 工具读到的内容（commit message、网页、文件）可能包含攻击者预埋的指令（提示注入），模型可能把它当指令执行——所以进入上下文的外部内容都要有「这是数据不是指令」的隔离意识。

## 深入阅读

- AI SDK · Tool Calling：https://ai-sdk.dev/docs/ai-sdk-core/tools-and-tool-calling
- OWASP · LLM Prompt Injection：https://genai.owasp.org
