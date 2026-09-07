# W2-D1 ｜ 数据源加固：多仓库、空周与异常兜底

> **Sprint 目标回顾**：本周让作品 1 从 MVP 变成「敢公开给别人用」的正式版。
> **前置**：W1 五张任务卡全部验收，`source=git` 基础链路可用。

## 今日验收

- [ ] `ALLOWED_REPOS` 支持多个仓库：并行读取、结果合并标注来源仓库
- [ ] 空提交周有兜底：git log 为空时给模型明确提示「本周无提交记录」，周报如实呈现而不是编造
- [ ] 异常兜底：git 命令失败（路径不存在/非 git 仓库）时降级为 paste 模式并把原因告知用户
- [ ] 超长输入保护：原始输入超过预算（如 4000 字符）时截断，且截断策略写进代码注释

## 任务卡

### T1 · 多仓库聚合（15min）

```ts
export async function readAllGitLogs(repos: string[], days = 7) {
  const results = await Promise.allSettled(repos.map(r => readGitLog(r, days)))
  return results.flatMap((r, i) =>
    r.status === 'fulfilled'
      ? r.value.map(c => ({ ...c, repo: path.basename(repos[i]) }))
      : [],   // 单仓库失败不拖垮整体：fail-soft
  )
}
```

**决策点**：`Promise.allSettled` 而非 `Promise.all`——三个仓库一个失败，周报不应该整体失败。多数据源的 AI 功能里，「部分成功」是常态，接口设计要承认这一点。

### T2 · 空周兜底（10min）

空数据时不要把空字符串丢给模型（会诱发编造），而是显式注入事实：

```ts
const material = commits.length
  ? renderCommits(commits)
  : '（本周该仓库无提交记录。summary 中如实说明，done 为空数组，不得虚构任何条目。）'
```

**原理**：模型对「空白」会本能补全。反幻觉不只在 prompt 里喊口号，更要在**数据层就消灭模糊**——这是 W1-D4 评测里「噪音用例」教你的事的延伸。

### T3 · 异常降级链（10min）

```
git 模式失败 → 提示原因 + 自动切回 paste 模式（用户输入仍可完成周报）
```

UI 上 toast 说明「自动读取提交失败（原因 xxx），已切换为手动粘贴」。**降级不是隐藏错误**，是给用户另一条路并如实告知。

### T4 · 输入预算保护（5min）

```ts
const MAX_INPUT_CHARS = 4000
// 截断策略：保留最早与最近各一半（周报两头都有信息价值），中间以 […省略…] 标记
```

## 关键决策与原理

1. **fail-soft 的边界**：单仓库失败可软处理，但所有来源都失败必须硬报错——「部分降级」与「静默吞错」一线之隔，判断标准是：用户能否感知并采取行动；
2. **空值即指令**：给 LLM 的输入里没有「无数据」的显式声明，模型就会自己脑补一个；数据工程的完整性直接决定 AI 输出的诚实度；
3. **截断策略是产品决策**：丢头还是丢尾还是抽稀？取决于下游任务更看重什么——写进注释， future 自己会感谢现在；
4. **错误信息是 UI 的一部分**：「失败了」和「因为 A 失败了，已帮你切到 B」是两个产品。

## 进阶挑战（选做）

- `git log --shortstat` 把每个提交的文件变更量解析出来，作为周报「工作量分布」的量化素材；
- 仓库路径支持 `~` 展开（Windows 下注意 `os.homedir()`）。

## 自测题（答完翻底部）

1. `Promise.all` 和 `Promise.allSettled` 在多仓库场景下的行为差异？
2. 为什么空数据要显式注入「无记录」声明而不是传空字符串？
3. 「降级」和「吞错」的区别是什么？判断标准？
4. 输入截断为什么「保头保尾」而不是只保留前面？
5. 本周无提交时，理想输出里 `done` 字段应该是什么？为什么？

---

## 答案与解析

1. `all` 任一失败整体 reject（一损俱损）；`allSettled` 返回每个的状态，允许部分成功——多源聚合场景几乎总该用它；
2. 空字符串在模型看来是「待补全的空白」，极易生成虚构内容；显式声明把「不确定性」变成「已知事实」，模型才能如实转述；
3. 吞错是用户无感知地换了行为；降级是**告知 + 给替代路径**。标准：用户能否知道发生了什么并有下一步选择；
4. 周报语境里周初的计划性内容和最近的工作都高价值，中段信息密度通常最低；「中间省略」还保留了「被截断」的可解释性；
5. 空数组。schema 里 `done` 是数组类型，空周如实输出空数组 + summary 说明，评测的 forbid 断言才能守住「不得虚构」这条线。

## 深入阅读

- MDN · Promise.allSettled：https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled
- 十二要素 · 可弃性（Disposability）与容错思想：https://12factor.net/zh_cn/
