# W3-D3 ｜ 检索接口：从「问一句」到「命中最相关的 5 块」

> **前置**：D1 向量库 + D2 切块器就绪。
> **今日一句话**：把「向量化 → 相似度 → 排序」封装成一个 `search()`——检索内核的正式 API。

## 今日验收

- [ ] `src/similarity.ts`：手写余弦相似度（含归一化），带 3 个单测
- [ ] `src/search.ts`：`search(query, { k })` 两套实现——学习版（JS 全量算余弦）与生产版（sqlite-vec KNN）
- [ ] CLI 可用：`pnpm search "响应式原理"` 输出 top-5（含来源、heading、相似度分数）
- [ ] 两种实现的结果一致（同一个 query 对比验证，允许浮点尾差）

## 任务卡

### T1 · 手写余弦（10min）

```ts
export function cosine(a: number[], b: number[]): number {
  let dot = 0, normA = 0, normB = 0
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i]; normA += a[i] * a[i]; normB += b[i] * b[i]
  }
  return dot / (Math.sqrt(normA) * Math.sqrt(normB))
}
```

**为什么要手写一遍**：2048 维循环 5 分钟写完，但从此余弦相似度对你是「具体的东西」而不是「调库」——面试聊「为什么用余弦不用欧氏」时，手写过的人和背答案的人语气不一样。

### T2 · 学习版检索（10min）

```ts
export async function searchBrute(db: Database, embedModel: EmbeddingModel, query: string, k = 5) {
  const { embedding: qvec } = await embed({ model: embedModel, value: query })
  const rows = db.prepare('SELECT id, embedding FROM chunks').all() as { id: number; embedding: string }[]
  const scored = rows.map(r => ({ id: r.id, score: cosine(qvec, JSON.parse(r.embedding)) }))
  return scored.sort((a, b) => b.score - a.score).slice(0, k)
}
```

### T3 · 生产版：sqlite-vec KNN（10min）

```ts
export function searchKnn(db: Database, qvec: number[], k = 5) {
  return db.prepare(`
    SELECT m.id, m.source, m.heading, m.content, v.distance
    FROM chunks v JOIN chunk_meta m ON m.id = v.rowid
    WHERE v.embedding MATCH ? AND v.k = ?
    ORDER BY distance
  `).all(JSON.stringify(qvec), k)
}
```

**决策点（今日核心）**：暴力版 O(N) 全量算一遍——1 万块毫无压力、且逻辑全透明；vec0 的 MATCH 走向量索引，规模化后的正路。**两个都留着**：学习版是你的「参照实现」，生产版结果可疑时用它对拍。这种「简单实现验证复杂实现」的手法是检索系统的通用调试术。

### T4 · CLI 封装（10min）

```bash
pnpm search "响应式原理是怎么实现的"
# 输出：score 0.62 [vue-notes.md | 响应式系统] Vue 3 的响应式基于…
```

`search()` 的签名从此冻结为 `{ query, k } → Chunk[]`——上层（W4 的问答、作品 2 的笔记对话）只依赖这个接口，底层换索引/换模型它们无感。**接口先行，实现可换**。

## 关键决策与原理

1. **余弦 vs 点积 vs 欧氏**：余弦比方向不管长度（文本长度不应影响相似度）；向量归一化后余弦与点积等价——多数系统内部用点积算，语义上都是余弦；
2. **相似度分数要透出**：top-5 每条带 score，是 D4 定拒答阈值、D5 调参数的数据基础。API 设计阶段就把「调优要用的数据」暴露出来；
3. **`k` 是权衡旋钮**：k 小→ precision 高但易漏；k 大→ 召回高但注入噪声且费 token。先固定 k=5 跑通，D5 评测再回来动它；
4. **查询也要 embedding**：query 与文档必须用**同一个** embedding 模型——跨模型比距离没有意义（两套地图坐标系不同）。

## 进阶挑战（选做）

- 对拍脚本：随机 20 个 query，断言学习版与生产版 top-5 的 id 集合一致；
- `search()` 加 `minScore` 参数（低于阈值的块直接丢弃）。

## 自测题（答完翻底部）

1. 为什么文本相似度用余弦而不是欧氏距离？
2. 「归一化后余弦与点积等价」意味着什么？
3. 为什么要同时保留暴力版和 KNN 版两套实现？
4. query 和文档用了不同 embedding 模型会怎样？
5. `search()` 接口为什么现在就冻结签名？

---

## 答案与解析

1. 欧氏受向量模长影响（长文本天然模长大），余弦只比方向——文本检索要的是「语义方向一致」，不受长度干扰；
2. 归一化（模长=1）后 `cos = dot`，所以工程上存归一化向量、用点积计算（更快），语义仍是余弦相似度；
3. 暴力版逻辑透明、是正确性基准；KNN 版高性能但实现黑盒——简单实现对拍复杂实现，是定位索引类 bug 的通用手法；
4. 两个模型的向量空间互不相通（维度都可能不同），算出的「距离」没有语义，检索结果≈随机——检索质量突然变差时第一个查「是否混用了模型」；
5. W4 的问答层和作品 2 都会依赖它；先冻结接口，底层实现（换索引、加 rerank、换模型）就不会波及上层——依赖倒置在检索系统里的落地。

## 深入阅读

- sqlite-vec · KNN 查询：https://github.com/asg017/sqlite-vec/blob/main/examples/simple-node/demo.ts
- AI SDK · embed()：https://ai-sdk.dev/docs/ai-sdk-core/embeddings#embed
