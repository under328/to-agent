# W1-D4 ｜ 质量引擎：评测集——从「感觉不错」到「有据可依」

> **前置**：D2/D3 的生成链路可用。
> **今日一句话**：AI 应用与传统应用最大的差异是「输出质量不可断言」。评测集（evals）就是 AI 功能的单元测试+回归测试，也是 2026 年面试的核心考点。

## 今日验收

- [ ] `eval/cases.ts`：10 个真实脱敏样例（3 类输入：规整列表 / 口语碎片 / 含噪音闲聊）
- [ ] `eval/run.ts`：跑分脚本，输出每例得分与总分表格
- [ ] 评分维度：结构合法性（schema 过没过）+ 关键点命中率（mustInclude）+ 禁止项（forbid，如「不得出现输入中没有的会议」）
- [ ] `eval/SCORES.md`：记录 prompt v1 与 v2 的分数对比（唯一变量：加「不得编造未提及事项」约束）

## 任务卡

### T1 · 设计用例（10min）

```ts
export type Case = {
  name: string
  input: string
  mustInclude: string[]   // 关键词至少命中 N 个（宽松 includes 匹配）
  forbid?: string[]       // 输出中不允许出现
}

export const cases: Case[] = [
  {
    name: '规整列表',
    input: '周一：完成登录页重构，修复 3 个样式 bug\n周二：参加需求评审会',
    mustInclude: ['登录', '评审'],
    forbid: ['上线'],   // 输入没提上线，编了就扣分
  },
  // …再写 9 个：口语碎片 4 个、含噪音 3 个（闲聊/抱怨混入）、边界 2 个（空输入、超长输入）
]
```

**用例设计要点**：三类覆盖——正常、难解析（口语碎片）、有陷阱（噪音与诱导编造）。用你**真实工作记录脱敏**后的内容，别编——真实数据才暴露真实问题。

### T2 · 跑分脚本（20min）

```ts
// eval/run.ts
for (const c of cases) {
  const { object } = await generateWithRetry(c.input)   // 复用 D2 的函数
  const pass = reportSchema.safeParse(object).success
  const hits = c.mustInclude.filter(k => JSON.stringify(object).includes(k))
  const bans = (c.forbid ?? []).filter(k => JSON.stringify(object).includes(k))
  score(c.name, { pass, hitRate: hits.length / c.mustInclude.length, violations: bans })
}
printTable() // 每例得分 + 总分
```

### T3 · 迭代一轮 prompt（10min）

v2 只改一处：system 里加「**只能使用输入中明确提及的事项，禁止编造或补充**」。重跑，把两版分数记进 `SCORES.md`。

**唯一变量原则**：一次只改一处，否则分数变化无法归因——这条纪律和你写业务代码 A/B 实验一模一样。

## 关键决策与原理

1. **为什么评测先于 UI 打磨**：AI 应用最大风险是「质量不可知」——UI 再好，编造一次关键事项用户就流失。先有度量，再谈优化；
2. **为什么 v1 用关键词断言不用 LLM 打分**：断言便宜、稳定、可解释、秒级跑完——**跑得起的测试才会被经常跑**。LLM-as-judge 是 W2 的升级项，不是起步项；
3. **回归思维**：以后每次改 prompt / 换模型 / 升级 SDK，必跑全套评测——没有评测集的 AI 应用，每次改动都是赌博；
4. **评测也是成本预算单位**：10 例 × flash 模型 ≈ 忽略不计的成本，让你敢于高频迭代。

## 进阶挑战（选做）

- 加「稳定性维度」：同一样例跑 3 次，对比输出 JSON 一致率（temperature 0 也不保证 100% 一致，量化它）；
- 把跑分脚本挂成 `pnpm eval`，纳入提交习惯。

## 自测题（答完翻底部）

1. 评测集和单元测试的相似与不同？
2. 三类用例（规整/碎片/噪音）分别在测什么能力？
3. 为什么 v1 评测用关键词断言而不是 LLM-as-judge？
4. 「唯一变量原则」在 prompt 迭代中如何落实？
5. 换模型供应商时，评测集扮演什么角色？

---

## 答案与解析

1. 相同：自动化断言、防回归、可持续运行；不同：传统输出是确定函数结果，AI 输出是概率分布——所以评测断言的是「关键属性」（结构、要点覆盖、禁忌）而非逐字符相等；
2. 规整测基线能力；碎片测信息抽取与归纳（真实用户输入常态）；噪音测抗诱导与不编造（安全底线）；
3. 断言确定性 100%、零额外成本、结果可解释，能把「每改必跑」变成习惯；judge 本身也是模型输出，有噪声有成本，适合作为 v2 补充而非起点；
4. 每轮只改 prompt 的一处（如加一条禁令），改完重跑全套记录分数；分数不动说明改动无效，退回；
5. 模型迁移的验收标准——同一套评测跑两个模型，分数对比决定是否切换，把「听说 X 模型强」变成「X 在我的任务上高 Y 分」。

## 深入阅读

- AI SDK · Evals 指南：https://ai-sdk.dev/docs/foundations/evals
- Hamel Husain · Your AI Product Needs Evals：https://hamel.dev/blog/posts/evals/
