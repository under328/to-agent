# W1-5 ｜ 实验课：改造示例，建立「提示词手感」

> 课程表来源：curriculum.md · W1 学习条目 ⑤ ｜ 建议 08:10~09:00 使用
> 今天是 W1 收官课：跑通 → 改动 → 观察 → 记录，这个循环将贯穿你整个转型期。

## 今日目标（3 条）

1. 掌握读官方示例的四步法（跑通→改动→观察→记录）；
2. 亲手对比 **system prompt 三种写法**对输出的影响；
3. 写下你的第一篇实验日志（journal/ 的正确打开方式）。

## 概念讲解

**为什么「改造示例」比「看教程」有效？** 你是重度 AI 工具用户，早已体会过：看懂 ≠ 会做。编程知识只有经过「自己改一刀、观察输出变化」才会长在身上。以后每篇课程都按这个四步法走。

**system prompt 三板斧**（今后所有 AI 功能设计的底层框架）：

1. **无设定**：直接问——输出随机、风格漂移，质量靠运气；
2. **角色设定**：「你是资深技术主编」——风格开始稳定；
3. **角色 + 输出约束**：规定格式（如 JSON）、字段、边界——输出从「能看」变成「能用」。**工程上真正重要的是第 3 级**：能被程序解析的输出才叫产出。

**控制变量法**：一次只改一处，否则你永远不知道是哪个改动起了作用——这也是做 AI 功能调试（乃至未来面试聊「你怎么调优 RAG」）的基本功。

## 动手清单

新建 `src/experiments.ts`：

```ts
import 'dotenv/config'
import { generateText } from 'ai'
import { createOpenAICompatible } from '@ai-sdk/openai-compatible'

const zhipu = createOpenAICompatible({
  name: 'zhipu',
  baseURL: 'https://open.bigmodel.cn/api/paas/v4',
  apiKey: process.env.ZHIPU_API_KEY ?? '',
})

const question = '把今天的工作记录整理一下：上午修了登录页 bug，下午开了需求评审会。'

const systems = {
  无设定: '',
  角色版: '你是严谨的技术项目经理，输出精炼、有条理。',
  约束版:
    '你是严谨的技术项目经理。把工作记录整理成 Markdown，固定输出两个小节：\n' +
    '## 今日完成（列表，每条一句话）\n## 明日建议（最多 2 条）\n不要输出其他任何内容。',
}

for (const [name, system] of Object.entries(systems)) {
  const { text } = await generateText({
    model: zhipu('glm-4-flash'),
    ...(system ? { system } : {}),
    prompt: question,
  })
  console.log(`\n========== ${name} ==========\n${text}`)
}
```

```bash
pnpm tsx src/experiments.ts
```

**观察重点**：无设定的输出像聊天；角色版像人写的汇报；约束版是**结构稳定、能被程序解析**的产出——这个差别就是你未来吃饭的手艺。

## 写第一篇实验日志

新建 `journal/2026-09-11-experiment.md`（模板照抄）：

```markdown
# 实验日志 2026-09-11：system prompt 三板斧
- 改了什么：system 从无 → 角色 → 角色+输出约束
- 观察到什么：（用自己的话写 3 行以内）
- 结论/新疑问：（想继续验证什么）
```

**周末任务（W1 产出验收）**：把 playground 里散落的脚本整理好、README 写两行说明、commit + push——学习仓库已就绪 ✅，这一步算完成半个里程碑。

## 自测题（先答，再对答案）

1. 读官方示例的四步法是什么？
2. system prompt 三板斧分别是什么？哪一级才是「工程可用」的分水岭？为什么？
3. 「控制变量法」在调试 AI 功能时为什么重要？
4. 为什么说「能被程序解析的输出才叫产出」？
5. `...(system ? { system } : {})` 这个展开写法的作用是什么？

---

## 答案与解析

1. 跑通 → 改动（一次一处）→ 观察（对比输出）→ 记录（实验日志）。
2. 无设定 / 角色设定 / 角色+输出约束；分水岭是第三级——前两级输出风格不稳定、格式不可预测，只有加了输出约束，程序才能稳定解析，功能才能上线。
3. AI 输出受多个因素叠加影响，一次改多处就无法归因；控制变量才能定位「哪个改动有效」，这也是面试聊调优时的基本素养。
4. 上线的 AI 功能下游都是代码：UI 渲染、入库、统计——它们需要确定的结构（如固定小节、JSON 字段）；自由文本只能给人看，不能被程序消费。
5. 条件展开：system 为空串时不传 `system` 字段（等价于无设定组），保证三组实验只差「是否有 system」这一个变量。

## 参考资料

- AI SDK 官方示例库：https://github.com/vercel/ai/tree/main/examples
- Prompt 交互式教程（中文）：https://www.promptingguide.ai/zh

## 下周预告

W2：多轮对话——messages 数组、上下文窗口、历史裁剪。你将做出一个能「记住上文」的终端聊天器。
