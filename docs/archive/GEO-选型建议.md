# GEO 项目选型建议（面向「全球 AI 获客与智能搜索门户」）

> 输入：公司规划图第 5 项「全球 AI 获客与智能搜索门户（GEO 布局）」
> 场景：原创设计出口工厂 · 跨境 B2B · 目标是让海外买手通过 AI 搜索找到并信任我们
> 日期：2026-09-14

---

## 一、结论

**主力工具选 `geo-optimizer-skill`**，配套 `elmo` 做持续监测看板。

理由不是"它星多"，而是**它的能力边界和你图里的诉求逐条重合**：

### 1. 目标 AI 引擎完全对口

`geo-optimizer-skill` 的 `SKILL.md` 明确列出的爬虫/引擎清单：

| 爬虫 | 对应引擎 |
|------|----------|
| `OAI-SearchBot` | ChatGPT Search 引用 |
| `PerplexityBot` | Perplexity 答案引用 |
| `ClaudeBot` | Claude 网页引用 |
| `Google-Extended` | Gemini / AI Overviews |

这正是你图里写的「ChatGPT Search / Perplexity / Google AI 以及市面上大部分 AI」。
其余候选项目（GEORank 等）也提这些引擎，但只有它把**每个引擎的爬虫准入**做进了评分。

### 2. 100 分评分模型逐条命中你的原话

| 你在图里的原话 | 对应评分维度 | 分值 |
|---|---|---|
| 「方便让 AI 可以抓取我们公司的数据」 | Robots.txt 18 + llms.txt 18 + AI Discovery 6 | **42** |
| 「提供结构化设计资质与合规标准」 | Schema JSON-LD 16 + Brand & Entity 10 | **26** |
| 「AI 首页直接推荐我们」 | Meta Tags 14 + Content 12 + Signals 6 | **32** |

「抓取」这一项占了 42 分，说明这个工具的核心假设就是"先让 AI 爬得到，再谈推荐"——
和你们规划里的因果链完全一致。

### 3. 落地成本最低

- **MIT 协议**，商用无开源义务、无传染性
- `pip install geo-optimizer-skill` 一行装完，或 `uvx` 免安装直接跑
- **CLI 全部免费**，不强制购买任何在线服务
- 只有 `geo citations`（真实问 AI 引擎）需要自带一个 Perplexity 或 OpenAI API Key

### 4. 你要的是"优化自己官网"，不是"做 GEO 服务卖给别人"

这是选型的**关键分水岭**：

- 审计修复型工具（`geo-optimizer-skill`）→ 适合"我就一个官网，我要它被 AI 推荐"
- SaaS 平台型项目（`GEOFlow` / `GEORank` / `elmo`）→ 适合"我要管理多品牌/多客户，对外提供 GEO 服务"

你们属于前者。用平台型项目属于杀鸡用牛刀，还要额外背上数据库、模型 API、运维成本。

---

## 二、落地路径（可直接执行）

```bash
pip install geo-optimizer-skill

# 第 1 步：给官网做一次基线体检，拿到 0-100 分和整改清单
geo audit --url https://你们的域名.com

# 第 2 步：整站扫描，按最差的页面优先排序
geo audit --sitemap https://你们的域名.com/sitemap.xml --max-urls 25

# 第 3 步：自动生成缺失的 robots.txt / llms.txt / schema / meta
geo fix --url https://你们的域名.com --apply
geo llms --base-url https://你们的域名.com --output ./public/llms.txt

# 第 4 步：验证 AI 到底推不推荐你们 —— 这一步直接对应图里那句
#   「海外买手提问『推荐中国优秀原创设计出口工厂』时，AI 首页直接推荐我们」
geo citations --brand "你们的品牌" --domain 你们的域名.com \
  --topic "original design apparel manufacturers in China" --runs 5

# 第 5 步：建立持续监测，出 HTML 趋势报告
geo track --url https://你们的域名.com --report --output ./geo-report.html
```

> `--runs 5` 会同一问题采样 5 次给置信区间——AI 回答每次都不一样，
> 单次结果不能当结论，这点对"老板要看效果"的场景很重要。

---

## 三、配套与备选

### 配套：`elmo` —— 持续监测看板
`geo track` 出的是报告，不是看板。如果想要一个**可以随时打开看趋势**的界面：
- MIT 协议，Docker Compose 自托管，数据存在自己的 PostgreSQL
- 后台 worker 按计划定时跑 prompt，抓取 ChatGPT / Claude / Perplexity / Gemini / Google AI Overviews 的**真实回答原文**并入库
- 自托管免费（官方云版 $29/月，可不用）

### 备选：`GEORank` —— 中文工作台
如果团队更看重**中文界面**和"诊断→问答→方案→拓词"的完整工作流（含 30/60/90 天行动方案），
`GEORank` 是 Apache-2.0、中文原生、可私有化部署。代价是更重（FastAPI + Next.js + 数据库 + 模型 API），
更适合当作**市场部内部工作台**而非工程工具。**建议先跑通 `geo-optimizer-skill`，确有需要再上它。**

### 辅助：`notfair-plugin`
如果内容团队已经在用 Claude Code / Codex 写文案，装上它可以让 AI 按 GEO 规则产出内容。

### 暂不建议
- **`GEOFlow`（3.6k★）**：**AGPL-3.0** 有传染性；且它是"内容工程 + 多站点分发"平台（PHP/Laravel），
  面向的是批量运营内容站群。除非你们打算做大规模内容矩阵，否则过重。**若将来要基于它做对外服务，
  AGPL 会要求你开源自己的修改**——这是法务必须提前评估的点。
- **`AutoGEO`**：ICLR'26 学术框架，需要 GPU 做 GRPO 训练。除非要做研究或自训改写模型。
- **`geo-optimizer-go`**：Go 库，给已有 Go 后端嵌 GEO 能力用的。

---

## 四、两个必须提醒的问题

### ⚠️ 1. 开源 GEO 只覆盖你图里 3 件事中的第 1 件

| 图中的事 | 能否用现成开源 GEO 项目解决 |
|---|---|
| ① GEO 生成式引擎优化 | ✅ 可以，`geo-optimizer-skill` 直接覆盖 |
| ② 独立站多模态以图搜图（Gemini 视觉匹配内部款式库） | ❌ **没有**现成开源 GEO 项目做这个，需自研 |
| ③ 引流进企业微信客户群 | ❌ 需自研 / 接 SCRM，与 GEO 无关 |

也就是说：**GEO 开源项目能帮你"被 AI 推荐"，但"以图搜图"和"企微引流"这两段得自己写。**
建议把这三件事拆成独立项目排期，不要指望一个 GEO 工具全包。

### ⚠️ 2. 「上传样品图给 Gemini 匹配」有知识产权风险

你们是**原创设计**工厂，款式库是核心资产。让海外买手把图片传给 Gemini、
再把你们的内部款式库喂进去做匹配，等于**把新款设计图交到 Google 手里**。

一旦进了第三方模型的请求日志，就可能被用于训练或存在泄露风险——对靠原创设计吃饭的工厂，
这个代价可能远高于省下的开发成本。

**建议替代方案**：用可自部署的开源视觉模型做匹配（CLIP / SigLIP 系列做向量检索，
或本地部署 Qwen-VL 等多模态模型）。以图搜图本质是**向量相似度检索**，
自建一套的成本并不高，而且款式库始终不出内网。
如果必须用 Gemini，至少要做：图片脱敏、只传局部细节、签订数据处理条款。

---

## 五、一句话总结

> **先用 `geo-optimizer-skill` 把官网的 GEO 基线分打出来，这是本周就能做完的事；
> 以图搜图和企微引流另立项目；Gemini 方案建议换成自部署视觉模型。**
