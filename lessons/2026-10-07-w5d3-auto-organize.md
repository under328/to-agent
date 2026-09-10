# W5-D3 ｜ 保存即整理：异步管线与标签治理

> **前置**：D2 录入管道可用。
> **今日一句话**：用户存完就走的体验背后，是一条「录入快、整理稳」的异步管线——这是 ai-notes 与普通笔记应用的技术分水岭。

## 今日验收

- [ ] 录入接口立即返回（<200ms 体感），整理管线异步执行，UI 显示「已入库 · 整理中…」
- [ ] 整理管线：flash 模型生成 `{ tags[], oneLineSummary }`，写入 notes 表，状态机 `pending → done / failed`
- [ ] failed 可重试（手动按钮即可，v1 不做自动重试队列）
- [ ] 标签治理：全局标签枚举表 + 新标签白名单确认机制生效

## 任务卡

### T1 · 异步化改造（15min）

```ts
// app/api/notes/route.ts
export async function POST(req: Request) {
  const note = await saveNote(await req.json())     // 同步：写库、去重、返回
  setTimeout(() => enrichNote(note.id).catch(console.error), 0)  // 异步：整理
  return Response.json({ id: note.id, status: 'captured' })
}
```

**决策点：fire-and-forget vs 持久队列？** 单用户本地跑、整理失败有状态字段和手动重试 → setTimeout 足够；什么时候必须上队列（BullMQ 类）——整理任务跨重启不能丢、或失败必须自动重试时。**现在的简单是清醒的选择，不是无知**：把升级条件写进代码注释（进程重启丢任务 → 升级队列），与 W5-D1 的 tags 迁移条件同款写法。

### T2 · 整理管线（20min）

```ts
async function enrichNote(id: number) {
  const note = getNote(id)
  const { object } = await generateObject({
    model: llm(process.env.LLM_MODEL_FLASH!),
    schema: z.object({
      tags: z.array(z.string()).max(3).describe('从现有标签表选择；都不合适则提议一个新标签'),
      oneLineSummary: z.string().max(50).describe('一句话摘要，≤30字，保留关键实体'),
    }),
    temperature: 0,
    prompt: `现有标签：${listTags()}\n\n笔记内容：${note.content.slice(0, 1500)}`,
  })
  updateNote(id, { ...object, embed_status: 'done' })
}
```

**细节**：把「现有标签表」喂进 prompt 是**封闭式分类**的关键——模型优先复用已有标签，标签体系才能收敛而不是每天膨胀出同义词；`max(3)` 防止「标签大杂烩」。

### T3 · 标签治理（10min）

模型提议的新标签不直接生效：进「待确认」状态，你在 UI 上每周点一次「收编/合并」（比如把「react-hooks」并进「react」）。**开放式标签必然碎片化**（大写小写、单复数、中英混写），完全放开三个月后标签系统就废了；完全封闭又失去灵活性——「提议 + 人工收编」是单用户产品跑得最久的折中。

### T4 · 状态机与失败可见（5min）

`pending / done / failed` 三态上 UI：整理中转圈、失败标红且可点重试。**异步管线的信任来自「状态可见」**——用户不知道后台发生了什么的系统，一旦失败就会被认为「整条管线坏了」。

## 关键决策与原理

1. **录入与整理分离是体验与可靠性的双赢**：录入路径只做确定性的本地写库（快、永不失败于网络），LLM 调用挪到异步——用户 3 秒录入的目标不因整理环节抖动而破灭；
2. **封闭式分类优于开放式生成**：把已有标签喂给模型 = 给它「菜单」而不是「白纸」，标签体系才会收敛；这条同样适用于未来的文件夹归类、笔记类型判断；
3. **治理机制是 AI 功能的隐藏工程**：模型输出（标签）也是需要治理的数据——「提议 + 收编」让模型效率与人的控制权共存，这比「全自动」和「全手动」都更接近真实产品的答案；
4. **状态机是异步系统的最小可观测性**：pending/done/failed 三个字段换来「失败可见、可重试」，投入产出比极高。

## 进阶挑战（选做）

- 整理成本入账：enrichNote 调用接成本日志（复用 W2-D3），跑一周后统计「每条笔记的整理均价」；
- 失败自动重试一次（间隔 30s），仍失败才转 failed——注意重试也要有上限。

## 自测题（答完翻底部）

1. 录入接口为什么要与整理管线分离？分开后各自的成功标准是什么？
2. 「把现有标签表喂进 prompt」解决什么问题？
3. 新标签「提议 + 人工收编」机制平衡了哪两边的矛盾？
4. fire-and-forget 的升级条件（什么信号出现时必须上队列）？
5. 状态机字段最小的三个状态是什么？为什么 failed 必须可见？

---

## 答案与解析

1. 分离让录入路径只依赖本地确定操作（快且稳），LLM 的延迟与失败被隔离在异步侧；成功标准：录入 <200ms 体感、整理成功率独立度量（不互相牵连）；
2. 封闭式分类：模型从既有标签中选择优先于发明新词，标签体系收敛、统计有意义，避免同义词碎片化；
3. 模型的灵活性（新内容可能真需要新维度）与人的控制权（体系不失控）——「提议」保留灵活，「收编」保留治理；
4. 任务不能丢（进程重启后 pending 永远 pending）或失败必须自动恢复时——单用户本地可容忍手动重试，多用户/SaaS 化后必须上持久队列；
5. pending/done/failed；失败不可见的异步系统在第一次失败后就会失去全部信任——用户无法区分「在整理」和「坏了」。

## 深入阅读

- MDN · setTimeout 与事件循环（理解 fire-and-forget 的边界）：https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Event_loop
- 昨天去重键（D2）与今天标签收编的共同思想：**模型/数据的自由度都要有治理边界**
