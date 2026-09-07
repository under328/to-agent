# W2-D2 ｜ 评测升级：20 例回归集 + LLM-as-judge 双轨评分

> **前置**：W1-D4 的 10 例断言评测已运行。
> **今日一句话**：断言便宜但不全面，judge 全面但有噪声——工程答案是双轨并行，各管一段。

## 今日验收

- [ ] 评测集扩到 20 例：新增「多来源混合」「含敏感信息（应被脱敏）」「超长输入截断后」三类
- [ ] `eval/judge.ts`：用 LLM 按固定 rubric 打分（准确性/完整性/简洁性，各 1~5 分）+ 一句话理由
- [ ] 双轨报告：断言分（硬性，一票否决）+ judge 分（软性，趋势参考）同表输出
- [ ] `SCORES.md` 记录：v2 prompt 的断言分与 judge 分基线

## 任务卡

### T1 · 补三类用例（10min）

- 多来源混合：paste 文本 + 模拟 git log 合并输入 → 测「来源归并与去重」；
- 敏感信息：输入含手机号/内网 IP → `forbid` 断言输出中不得出现原样敏感串（隐私红线用例）；
- 截断输入：超过 4000 字符的长输入 → 测摘要完整性（昨日截断策略的回归验证）。

### T2 · LLM-as-judge（20min）

```ts
const judgeSchema = z.object({
  accuracy: z.number().min(1).max(5),   // 周报内容是否忠于输入、无编造
  completeness: z.number().min(1).max(5), // 输入要点是否遗漏
  conciseness: z.number().min(1).max(5),  // 是否简洁无废话
  reason: z.string(),
})

const judgement = await generateObject({
  model: judgeModel,          // 见下方决策点
  schema: judgeSchema,
  temperature: 0,
  prompt: `你是严格的技术周报评审。评分对象（周报）：\n${report}\n\n原始输入：\n${input}\n\n按定义逐项打分并给一句话理由。`,
})
```

**rubric 三条定义要写进 prompt**（每档分数的判据），否则 judge 每次用自己的标准，分数不可比。

### T3 · 双轨报告（10min）

`run.ts` 输出一张表：每例的断言结果（pass/违规词）+ judge 三维分 + judge 一句话理由。`SCORES.md` 追加 v2 基线行。

## 关键决策与原理

1. **judge 模型的选择**：避免用被测模型本身评自己（self-preference bias，自评偏高）。实操：被测用 A 家 flash，judge 用 B 家（或同家旗舰档）；
2. **judge 分数有方差**：temperature 0 也不保证稳定，**同一例跑 3 次取中位数**再入表——评测脚本自己也要可信；
3. **断言与 judge 的分工**：结构、禁编造、敏感信息这些「硬红线」必须断言（确定、免费、可解释）；「写得好不好」这种主观维度交给 judge。反过来（全用 judge）你会失去「为什么挂」的可解释性；
4. **judge 的成本账**：20 例 × judge 调用 ≈ 每轮几分钱，但它让你敢于随时改 prompt——这是整个体系里 ROI 最高的几分钟钱。

## 进阶挑战（选做）

- judge 输出按例对比两版 prompt（A/B 对比 prompt），让它输出 `preferred: 'v1' | 'v2' | 'tie'`——配对比较比绝对打分更稳；
- 把评测脚本接到 `package.json` 的 `prepush` 钩子上（想 push 先过评测）。

## 自测题（答完翻底部）

1. 为什么不建议用被测模型自己当 judge？
2. rubric 为什么要写明每档判据？
3. 断言评测和 judge 评测各自最适合什么维度？
4. judge 分数为什么要多次取中位数？
5. 「配对比较」相比「绝对打分」的优势是什么？

---

## 答案与解析

1. 模型对自身输出有系统性偏好（风格熟悉=打分偏高），自评会引入方向性偏差，对比实验失真；
2. 不写判据的 rubric 等于让 judge 每次随机选一把尺子，分数横向不可比，趋势图全是噪声；
3. 断言管客观红线（结构、禁词、敏感信息、要点命中）；judge 管主观质量（准确性、简洁性、语气）；
4. 单次 judge 是有噪声的随机变量，中位数对离群值稳健；不处理噪声的评测会引导你「优化噪声」而不是优化 prompt；
5. 同一输入两版输出直接比高下，judge 只需相对判断（容易且稳），回避了绝对分数的标尺漂移问题。

## 深入阅读

- Hamel Husain · Evals（judge 设计章节）：https://hamel.dev/blog/posts/evals/
- AI SDK · Evals：https://ai-sdk.dev/docs/foundations/evals
