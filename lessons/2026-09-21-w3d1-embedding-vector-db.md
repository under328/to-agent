# W3-D1 ｜ RAG Sprint · 向量地基：embedding 直觉与本地向量库

> **本周冲刺目标**：周日之前，「存 10 篇文档 → 问 3 个问题 → 带引用的回答」的检索内核可演示。
> **前置**：作品 1 项目的 LLM 调用已跑通（embedding 复用同一套 provider 配置）。
> **今天开工新项目**：`projects/knowledge`——检索内核独立成库，未来同时服务文档问答（W4）和作品 2（W5-W6）。

## 今日验收

- [ ] `projects/knowledge` 初始化：TS + `ai` + `@ai-sdk/openai-compatible` + `better-sqlite3` + `sqlite-vec`
- [ ] 本地向量库建表成功，插入 3 条文本的 embedding 并能查回
- [ ] 完成「相似句 vs 无关句」余弦距离小实验，把数字记进 journal
- [ ] 理解并能向别人解释：embedding 是什么、和关键词搜索的区别

## 任务卡

### T1 · embedding 直觉（10min，先建立心智模型）

embedding（向量化）= 把一段文字变成一个高维空间里的**坐标点**。语义相近的文本，坐标就相近。类比：把所有句子放进一个「意义地图」，「今天很热」和「今日气温高」是邻居，「今天很热」和「git rebase 教程」隔着整张地图。

- **关键词搜索**比的是「字面重合」：搜「温度高」找不到「今天很热」；
- **向量检索**比的是「意义距离」：能跨过措辞差异。

RAG（Retrieval-Augmented Generation，检索增强生成）= 先用向量检索找到相关资料，再把资料塞给 LLM 生成答案——治幻觉的地基工程。

### T2 · 初始化与建表（15min）

```bash
cd projects && pnpm init knowledge && cd knowledge
pnpm add ai @ai-sdk/openai-compatible dotenv better-sqlite3 sqlite-vec
pnpm add -D tsx @types/node
```

> ⚠️ **pnpm 10 的坑**：默认禁止依赖的编译脚本，`better-sqlite3` 不编译会直接 require 报错。执行 `pnpm approve-builds` 勾选 `better-sqlite3` 放行。

```ts
// src/db.ts
import Database from 'better-sqlite3'
import * as sqliteVec from 'sqlite-vec'

export const EMBED_DIM = 2048 // 必须与 embedding 模型输出维度一致（智谱 embedding-3，以平台文档为准）

export function openDb(file = 'knowledge.db') {
  const db = new Database(file)
  sqliteVec.load(db) // 加载向量扩展
  db.exec(`CREATE VIRTUAL TABLE IF NOT EXISTS chunks USING vec0(embedding float[${EMBED_DIM}])`)
  db.exec(`CREATE TABLE IF NOT EXISTS chunk_meta (
    id INTEGER PRIMARY KEY,
    source TEXT NOT NULL,
    heading TEXT,
    content TEXT NOT NULL
  )`)
  return db
}
```

**模式说明**：`chunks`（vec0 虚拟表）只存向量、用 rowid 对齐；正文与来源放普通表 `chunk_meta`——向量库管「找得到」，元数据表管「说得清」。

### T3 · 第一次向量化入库（10min）

```ts
// src/seed.ts
import 'dotenv/config'
import { embed } from 'ai'
import { createOpenAICompatible } from '@ai-sdk/openai-compatible'
import { openDb, EMBED_DIM } from './db'

const zhipu = createOpenAICompatible({ name: 'zhipu', baseURL: process.env.LLM_BASE_URL!, apiKey: process.env.LLM_API_KEY! })
const embedModel = zhipu.textEmbeddingModel('embedding-3') // 模型 ID 以平台文档为准

const texts = ['Vue 3 的响应式系统基于 Proxy 实现', 'React 使用 Virtual DOM 进行差分比对', '今天午饭吃了热干面']
const db = openDb()
for (const t of texts) {
  const { embedding } = await embed({ model: embedModel, value: t })
  if (embedding.length !== EMBED_DIM) throw new Error(`维度不符：${embedding.length} ≠ ${EMBED_DIM}`)
  const info = db.prepare('INSERT INTO chunk_meta(source, content) VALUES (?, ?)').run('seed', t)
  db.prepare('INSERT INTO chunks(rowid, embedding) VALUES (?, ?)').run(info.lastInsertRowid, JSON.stringify(embedding))
}
console.log('✅ 3 条向量已入库')
```

### T4 · 直觉小实验（5min）

写 `src/probe.ts`：把「前端框架的响应式原理」向量化，和库里 3 条逐一算余弦相似度（函数 5 行，明天正式封装）。预期：两条前端句子的分数显著高于热干面。**把三个数字抄进 journal**——这是你向量检索的第一组实验数据。

## 关键决策与原理

1. **为什么本地 sqlite-vec 起步而非云向量库**：零成本、数据不出本机、单机几万 chunk 内性能足够；规模化了再上云（迁移成本 = 导出重灌，架构不变）。**先用够用的，是工程成熟度而非妥协**；
2. **向量与元数据分表**：vec0 是专用虚拟表（只懂向量），元数据用普通表 JOIN——两边各干擅长的事；
3. **维度是强约束**：表结构的 `float[N]` 必须与模型输出维度一致，所以抽成 `EMBED_DIM` 常量并在入库时断言——错维度要到查询时才爆，断言让它死在入库时；
4. **embedding 模型与生成模型解耦**：一个负责「理解与检索」，一个负责「生成」，各自独立选型、独立换档（明天会反复用到这个分层）。

## 进阶挑战（选做）

- 把 `probe.ts` 的余弦函数独立成 `src/similarity.ts` 并写 2 个单测（同向量=1、正交≈0）；
- 查一下智谱 embedding-3 是否支持指定 `dimensions` 参数降维，记录结论。

## 自测题（答完翻底部）

1. embedding 和关键词搜索的本质区别？
2. RAG 的三个字母分别是什么？解决 LLM 的什么问题？
3. 为什么向量表和元数据表要分开？
4. `EMBED_DIM` 为什么要在入库时断言而不是查询时报错再说？
5. pnpm 10 下 better-sqlite3 装完跑不起来，第一反应查什么？

---

## 答案与解析

1. 关键词比字面重合，向量比语义距离——后者能跨措辞差异（同义、换说法）命中；
2. Retrieval-Augmented Generation，检索增强生成：先检索相关资料再生成，用「给模型看真实资料」治幻觉与知识过时；
3. vec0 虚拟表只为向量检索而生，塞业务字段反而受限；普通表管元数据与全文，rowid 关联——职责分离，换向量引擎时业务表不动；
4. 维度不匹配的向量写入可能被静默容忍或错误处理，污染整库；入库断言把错误前移到最早可发现点（fail fast）；
5. pnpm 10 默认拦截依赖的 install 脚本，原生模块（better-sqlite3）没编译成功——`pnpm approve-builds` 放行后重装。

## 深入阅读

- sqlite-vec：https://github.com/asg017/sqlite-vec
- AI SDK · Embeddings：https://ai-sdk.dev/docs/ai-sdk-core/embeddings
- 智谱 embedding-3 文档：https://docs.bigmodel.cn
