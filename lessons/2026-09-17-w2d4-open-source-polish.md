# W2-D4 ｜ 开源打磨：让作品配得上简历里的链接

> **前置**：功能与成本层就绪。
> **今日一句话**：面试官点开你 GitHub 链接的平均停留时间不到 30 秒——今天把 30 秒内的每一屏都打磨到位。

## 今日验收

- [ ] README 重写：三行定位（是什么/解决什么/怎么跑）+ 截图 + Quick Start ≤ 5 步
- [ ] 演示 GIF 一张（生成过程流式输出），放在 README 首屏
- [ ] LICENSE（MIT）+ `.env.example` 完善 + 安全自查通过（仓库历史无任何 Key）
- [ ] （可选）自定义域名或干净的 Vercel 域名，README 挂「在线体验」链接

## 任务卡

### T1 · README 首屏（15min）

结构模板：

```markdown
# AI 周报生成器（weekly-report）
> 把 git 提交与零散记录，一键变成结构化周报。在线体验：xxx
![demo](docs/demo.gif)
## Quick Start（≤5 步）
## 工程亮点（给读源码的人）
- 结构化输出：zod schema + 失败自修复重试
- 评测驱动：20 例回归集 + LLM-as-judge 双轨评分
- 成本工程：模型路由 + 版本化缓存
```

**决策点**：「工程亮点」一节是给面试官看的——功能谁都能做，**方法论才是稀缺品**。这一节直接决定了这个仓库是「玩具」还是「作品」。

### T2 · 录 GIF（10min）

Windows 工具：ScreenToGif（轻量）或 OBS 录制转 GIF。脚本：打开页面 → 粘贴记录 → 点击生成 → 流式输出完整呈现 → 复制 Markdown。**控制在 10 秒内**，GIF 体积 < 3MB（放 `docs/` 目录，别塞仓库根目录）。

### T3 · 安全自查（10min）

```bash
# 历史提交里扫密钥（.env 曾被提交过就会在这里现形）
git log -p --all | grep -iE "sk-[a-z0-9]{20,}|api[_-]?key\s*=" || echo "✅ 干净"
# 确认 .env 从未被追踪
git log --all --full-history -- "*.env" --oneline
```

**红线**：发现历史泄露 → 吊销 Key + 考虑 rewrite 历史（`git filter-repo`）或干脆重建仓库。

### T4 · 元数据（5min）

MIT LICENSE、`package.json` 的 description/repository 字段、GitHub 仓库 About 填写 + topics（`ai`、`nextjs`、`vercel-ai-sdk`、`weekly-report`）——topics 决定搜索可发现性。

## 关键决策与原理

1. **README 面向两类读者写两遍**：使用者只要 Quick Start；读源码的人（面试官）要看到架构与方法论——把「工程亮点」放首屏第三屏，就是给后者的高速路；
2. **开源前的信任成本**：一个泄露的 Key 能抵消所有工程亮点，安全自查必须是发布前固定仪式；
3. **GIF > 文字**：AI 应用的核心体验是「生成过程」，静态截图传达不了流式——录屏脚本要提前设计（这本身也是一次产品演示彩排）；
4. **topics 与可发现性**：开源的第二价值是被搜到，第一价值的 50% 也来自「有人因为搜索找到你」——W10 增长周会复用今天的基础。

## 进阶挑战（选做）

- README 加「Deploy with Vercel」一键部署按钮（用户 fork 即跑，降低试用门槛）；
- 加英文版 README.Abstract（README.zh-CN.md + README.md 双语）——为简历的英文阅读能力做物证。

## 自测题（答完翻底部）

1. README 里「工程亮点」一节主要写给谁？为什么？
2. 安全自查的两条命令分别在查什么？
3. 发现 Key 曾被提交到历史，处置顺序是什么？
4. 演示 GIF 的录制脚本为什么要提前设计？
5. GitHub topics 的工程价值是什么？

---

## 答案与解析

1. 写给技术面试官/资深同行——功能描述人人会写，schema/评测/成本这套方法论才是区分「玩具与作品」的信号，也是面试深挖的入口；
2. 第一条扫全部提交历史 diff 中的密钥特征串；第二条查 `.env` 文件是否在任何提交里被追踪过——两条分别抓「内容泄露」和「文件泄露」；
3. 立即吊销该 Key（视为已泄露）→ 清理或重建仓库历史 → 排查泄露期间账单 → 复盘防止再犯；
4. GIF 只有 10 秒，必须呈现核心价值（流式生成）；现场随手录 often 卡顿/超长/没拍到关键交互，脚本化 = 一次产品叙事彩排；
5. topics 是 GitHub 搜索与话题浏览的索引维度，直接决定陌生人能否通过 `vercel-ai-sdk` 这类标签发现你的仓库——影响开源的第二增长曲线。

## 深入阅读

- GitHub · About 仓库与 topics 文档：https://docs.github.com/zh/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository
- git filter-repo（历史清理）：https://github.com/newren/git-filter-repo
