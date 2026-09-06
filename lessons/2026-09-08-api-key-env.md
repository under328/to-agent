# W1-2 ｜ API Key 与 .env：凭证管理与安全底线

> 课程表来源：curriculum.md · W1 学习条目 ② ｜ 建议 08:10~09:00 使用
> 今天起需要一把「真钥匙」：注册一个 LLM 平台拿 API Key（见动手清单第 0 步）。

## 今日目标（3 条）

1. 拿到你的第一个 **API Key**（推荐智谱开放平台，GLM 系列有免费额度；DeepSeek 也可以）；
2. 掌握 `.env` + `process.env` 的标准凭证管理用法；
3. 建立**红线意识**：API Key 永远不进 Git、不进截图、不进聊天记录。

## 概念讲解

**API Key 是什么？** 一串证明「你是谁」的字符串，平台靠它记账扣费。类比：**一张绑了自动扣款的银行卡号**——谁能拿到，谁就能花你的钱（或免费额度）。

**为什么不能写死在代码里？** 代码会进 Git、会被开源、会被复制。Key 一旦进了提交历史，**视为永久泄露**（历史可以翻旧账），正确做法是把 Key 放在**环境变量**里，代码只负责「读」，不负责「存」。这也是著名的 12-Factor 应用原则的第一条：**配置与代码分离**。

**`.env` 文件惯例**：

- `.env` —— 真正的钥匙，写实际值，**必须进 `.gitignore`**；
- `.env.example` —— 钥匙的「外壳模板」，只有变量名没有值，提交进 Git，告诉协作者需要配哪些变量。

**泄露了怎么办？** 立即去平台控制台**吊销并重新生成**（轮换）。速度要快，别不好意思。

## 动手清单

```bash
# 0. 拿 Key（10 分钟）
#    智谱：open.bigmodel.cn → 注册 → 控制台 → API Keys → 创建并复制
#    （备选 DeepSeek：platform.deepseek.com，流程类似）

cd projects/playground
pnpm add dotenv

# 1. 创建 .env（把 sk-xxx 换成你的真 Key）
echo 'ZHIPU_API_KEY=sk-xxx' > .env

# 2. 创建模板（这个可以提交）
echo 'ZHIPU_API_KEY=你的key' > .env.example

# 3. 红线检查：确认 .env 被 Git 忽略（工作区根目录的 .gitignore 已包含 .env）
cd ../..
git check-ignore projects/playground/.env && echo "✅ 已被忽略，安全"
```

验证脚本 `projects/playground/check-key.mjs`：

```js
import 'dotenv/config'

const key = process.env.ZHIPU_API_KEY
if (!key) {
  console.error('❌ 未找到 ZHIPU_API_KEY，检查 .env 是否存在、变量名是否一致')
  process.exit(1)
}
const mask = (s) => `${s.slice(0, 4)}****${s.slice(-4)}`
console.log('✅ 已加载 Key:', mask(key))
```

```bash
node check-key.mjs   # 看到打码的 Key 即达标
```

## 自测题（先答，再对答案）

1. API Key 本质上证明了两件事，是哪两件？
2. 为什么「Key 写在代码里且已提交 Git」等于「必须吊销重办」？
3. `.env` 和 `.env.example` 的分工分别是什么？哪个进 Git？
4. 代码里读取 `.env` 中变量的标准方式是？（Node 生态）
5. 发现 Key 被传到了公开仓库，正确的处置顺序是什么？

---

## 答案与解析

1. 身份（这个请求是谁发的）+ 计费（额度记在谁头上）。
2. Git 历史可追溯，删掉当前文件不代表历史里没有；任何能翻仓库历史的人都能拿到 Key，所以只能吊销作废，不能当没事。
3. `.env` 存真实值、**绝不进 Git**；`.env.example` 只存变量名模板、**进 Git**，作用是告诉协作者/未来的自己「这里需要配哪些变量」。
4. `process.env.变量名`（配合 `dotenv` 在入口 `import 'dotenv/config'` 加载 `.env`）。
5. 立即吊销 → 生成新 Key → 清理历史（或直接弃用仓库）→ 排查泄露期间的用量账单。

## 参考资料

- dotenv：https://github.com/motdotla/dotenv
- 12-Factor · 配置：https://12factor.net/zh_cn/config
- 智谱开放平台：https://open.bigmodel.cn （DeepSeek：https://platform.deepseek.com）

## 明日预告

W1-3：安装 Vercel AI SDK，发出你人生第一次代码级 LLM 调用（`generateText`）——今天拿到的 Key 明天就派上用场。
