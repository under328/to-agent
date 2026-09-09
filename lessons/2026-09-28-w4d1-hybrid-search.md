# W4-D1 ｜ 混合检索：FTS5 关键词 + 向量排序，RRF 融合

> **本周冲刺目标**：把 W3 的检索内核升级为「文档问答应用」并上线，3 个真实用户。
> **前置**：`projects/knowledge` 全链路可用（W3 五张卡全部验收）。

## 今日验收

- [ ] `chunks_fts` FTS5 全文索引表建立（trigram tokenizer），入库时与向量表双写同步
- [ ] `hybridSearch(query, k)`：向量 top-20 + FTS top-20 → RRF 融合 → 最终 top-k
- [ ] 对比实验：≥3 个「精确术语类问题」（含型号/编号/专有名词），hybrid vs 纯向量的 hit@3 记入 `SCORES.md`
- [ ] 能说清：什么问题向量检索会输给关键词检索

## 任务卡

### T1 · 为什么混合（10min）

两个检索器是性格互补的同事：

- **向量检索** = 「懂意思的朋友」：说「怎么让页面不卡」能找到「性能优化」，但对 `PR-1234`、`glm-4-flash` 这类**精确符号**，语义平均化反而稀释信号；
- **关键词检索（BM25/FTS5）** = 「Ctrl+F」：精确术语一击即中，但「换个说法」就抓瞎。

生产级检索几乎都是混合的——今天把另一半装上。

### T2 · FTS5 建表与中文坑（15min）

```ts
db.exec(`
  CREATE VIRTUAL TABLE IF NOT EXISTS chunks_fts USING fts5(
    content, source, heading,
    tokenize = 'trigram'
  )
`)
// 入库时双写：向量表 + FTS 表（写在同一个 seed/ingest 函数里，保证一致性）
db.prepare('INSERT INTO chunks_fts(content, source, heading) VALUES (?, ?, ?)')
  .run(content, source, heading)
```

**关键坑（面试可讲）**：SQLite FTS5 默认分词器按空格切词，**中文直接失灵**（整句变成一个 token）。不引外部分词器的最稳解法是 `tokenize='trigram'`——按三字滑窗建索引，中文子串匹配立即可用。代价：query 至少 3 个字符，且索引体积偏大。查「PR-12」这类短编号时要留意。

### T3 · RRF 融合（15min）

```ts
// Reciprocal Rank Fusion：只看排名不算分数，天然回避两套分数量纲问题
const RRF_K = 60
function rrf(rankings: { id: number; rank: number }[][]) {
  const scores = new Map<number, number>()
  for (const ranking of rankings) {
    for (const { id, rank } of ranking) {
      scores.set(id, (scores.get(id) ?? 0) + 1 / (RRF_K + rank))
    }
  }
  return [...scores.entries()].sort((a, b) => b[1] - a[1])
}
// hybridSearch：向量排名 + FTS(bm25) 排名 → rrf → top-k → 回表取 chunk
```

**决策点**：为什么用 RRF 而不是「向量分 × 0.7 + BM25 分 × 0.3」的加权融合——余弦分数和 BM25 分数量纲完全不同，权重调参是无底洞；RRF 只用排名（rank-based），对分数分布鲁棒、几乎免调参。`60` 是论文沿用的平滑常数：让「两路都排前」的文档显著胜出，单路冠军不至于碾压。

### T4 · 对比实验（10min）

挑 3 个纯向量会翻车的问题（精确型号、编号、人名缩写类），分别跑 `search()` 与 `hybridSearch()`，hit@3 记入 SCORES.md。预期：hybrid 在这组上明显胜出，而「换说法类」问题两者打平——这正是混合的意义：**不丢分，只补分**。

## 关键决策与原理

1. **双写而非触发器**：SQLite 触发器对 FTS5 虚拟表可用，但双写写在同一个 ingest 函数里更显式、更好测——一致性责任放在业务代码而非数据库魔法；
2. **trigram 是中文 FTS 的免费午餐**：免去 jieba 等外挂分词依赖，代价是短查询限制与索引膨胀——记录这个取舍，比「能跑」更重要；
3. **融合层是策略层**：今天 RRF，明天也许加权重或过滤规则，融合函数独立成模块，检索器增减不动上层；
4. **实验先于上线路径**：hybrid 不是免费午餐（多一路查询、多一份索引），评测数据证明收益后再让它成为默认路径——你已经有评测集，这是你的特权。

## 进阶挑战（选做）

- 给 hybridSearch 加 `weights: [向量权重, 关键词权重]` 参数，与 RRF 对比一轮；
- 记录一条调研笔记：libsql/Turso 的原生向量能力与 sqlite-vec 的关系。

## 自测题（答完翻底部）

1. 向量检索在哪类查询上是短板？为什么？
2. FTS5 默认分词器对中文的问题是什么？trigram 如何解决、代价是什么？
3. RRF 相比加权分数融合的优势是什么？常数 60 起什么作用？
4. 「双写同步」失败会有什么后果？如何防御？
5. 什么数据能证明「hybrid 该成为默认路径」？

---

## 答案与解析

1. 精确符号类查询（型号/编号/代码标识/人名）：这些 token 语义信息稀薄，向量空间里与无关文本距离近，语义泛化反成噪声；关键词精确匹配完胜；
2. 默认按空白切词，中文整句成单 token 导致几乎无法命中；trigram 按三字滑窗索引，子串匹配可用；代价：查询需 ≥3 字符、索引体积增大；
3. RRF 只用排名，规避两套分数量纲不可比的问题、免调参；60 是平滑常数，压制单路名次差距、奖励多路一致靠前的文档；
4. 两表不一致 = 部分文档「向量找得到、关键词找不到」或反之，检索结果静默缺失；防御：双写放同一事务（better-sqlite3 的 transaction），ingest 失败整体回滚；
5. 评测集上 hybrid 在整体 hit@k 不降的前提下、在短板类问题上显著提升，且成本（延迟/存储）可接受——让 SCORES.md 的数据拍板，不是感觉。

## 深入阅读

- SQLite FTS5 文档：https://www.sqlite.org/fts5.html
- RRF 原论文（Cormack et al.）：https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf
