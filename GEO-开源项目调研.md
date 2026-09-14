# 开源 GEO（Generative Engine Optimization）项目调研

> 调研时间：2026-09-14 · 工作目录：`D:\上班的东西\geo`
> 全部通过 GitHub API 检索，**完整克隆**（含全部 git 历史，无 submodule / 无 LFS 依赖）

## 一、结论

"GEO" 在开源圈有三层含义，本次按你的选择聚焦 **生成式引擎优化（Generative Engine Optimization）**。
该方向 2025—2026 年爆发式增长，生态已相当成熟：从「审计打分」到「内容改写」，从「排名监测」到
「学术可复现框架」都有成熟开源实现，且多数在 2026 年 9 月仍有活跃提交。

共克隆 **9 个**代表性项目，总计约 **413 MB**。

## 二、项目清单

| # | 目录 | Stars | 技术栈 | 许可证 | 体积 | 提交数 | 最后提交 | 定位 |
|---|------|-------|--------|--------|------|--------|----------|------|
| 1 | `GEOFlow` | 3630 | PHP 8.3 / Laravel | AGPL-3.0 | 111M | 408 | 2026-09-14 | GEO 内容工程 + 多站点分发平台，含浏览器扩展、AI 质检、托管站点 |
| 2 | `notfair-plugin` | 3783 | TypeScript | MIT | 28M | 394 | 2026-09-10 | SEO/GEO/广告投放的 **AI Agent 技能包**（Claude Code / Codex / Cursor 插件） |
| 3 | `geo-optimizer-skill` | 795 | Python 3.9+ | MIT | 71M | 882 | 2026-09-14 | AEO/GEO 审计·优化·引用追踪工具包，已发布 PyPI，兼容 MCP |
| 4 | `GEORank` | 463 | Python / FastAPI + Next.js | Apache-2.0 | 20M | 19 | 2026-08-12 | GEO 排名诊断工作台（pnpm monorepo，含 CLI 与 skills） |
| 5 | `getcito` | 414 | TypeScript | 自定义 | 9.3M | 41 | 2026-08-27 | 自托管 AI 可见性追踪与优化平台 |
| 6 | `elmo` | 328 | TypeScript (Turborepo) | MIT | 31M | 980 | 2026-09-11 | 开源 AI 可见性追踪平台，监控 ChatGPT/Claude/Perplexity/Gemini 引用 |
| 7 | `AutoGEO` | 216 | Python (LLaMA-Factory + open-r1) | MIT | 90M | 25 | 2026-06-13 | **ICLR'26 论文官方实现**，规则挖掘 + GRPO 训练内容改写模型 |
| 8 | `geo-optimizer-go` | 191 | Go | MIT | 5.6M | 76 | 2026-03-27 | Go 可插拔 GEO 优化框架，内置 Structure/Schema/AnswerFirst/Authority/FAQ 策略 |
| 9 | `eGEOagents` | 184 | Python | MIT | 49M | 46 | 2026-09-14 | E-GEO：GEO/AEO agent 工具包，PyPI `egeo`，配套 arXiv 2511.20867 |

## 三、按用途选型建议

### 想快速给网站做一次 GEO 体检 → `geo-optimizer-skill`
一条命令把站点按 0—100 分打分，指出 AI 搜索引擎能否抓取/理解/引用，并追踪 ChatGPT、Perplexity、
Gemini、Claude、Google AI Overviews 的实际引用情况。MIT 协议、有 PyPI 包、带 SCORING_RUBRIC.md
评分标准，是上手成本最低的一个。

```bash
cd geo-optimizer-skill && pip install -e . && geo-optimizer audit https://your-site.com
```

### 想要一套可自托管的产品级平台 → `GEOFlow` / `GEORank` / `elmo` / `getcito`
- `GEOFlow`：星最多、功能最全（内容生成 → 质检 → 多站点分发闭环），但 **AGPL-3.0**，商用需注意开源义务。
- `GEORank`：Apache-2.0 最友好，FastAPI + Next.js，定位「诊断工作台」，代码量适中易读。
- `elmo`：MIT，980 次提交最活跃，Turborepo 结构规范，工程化程度高（有 e2e、biome、changeset）。
- `getcito`：与 elmo 同源思路，体量最小（9.3M），适合快速读懂整体架构。

### 想让 AI Agent 直接具备 GEO 能力 → `notfair-plugin`
不是传统软件，而是把 SEO/GEO 工作流写成可读的 `SKILL.md`，直接装进 Claude Code / Codex / Cursor。
适合你已经用 AI Agent 做内容，想让它懂 GEO 规则。

### 想做研究 / 训练自己的 GEO 模型 → `AutoGEO`
ICLR'26 论文《What Generative Search Engines Like and How to Optimize Web Content Cooperatively》
官方代码。三件套：规则提取（自动挖掘生成引擎的内容偏好）→ AutoGEO_API（基于规则的 prompt 改写）
→ 用 GRPO 强化学习微调。带 `run_cold_start.sh` / `run_grpo.sh` 复现脚本。

### 想在 Go 服务里嵌入 GEO 优化 → `geo-optimizer-go`
唯一一个 Go 实现，提供 `Strategy` 接口，可直接作为库集成进自己的后端。

## 四、目录结构说明

```
D:\上班的东西\geo\
├── GEOFlow/              # 1  PHP/Laravel     AGPL-3.0
├── notfair-plugin/       # 2  TypeScript      MIT
├── geo-optimizer-skill/  # 3  Python          MIT
├── GEORank/              # 4  Python+Next.js  Apache-2.0
├── getcito/              # 5  TypeScript      自定义
├── elmo/                 # 6  TypeScript      MIT
├── AutoGEO/              # 7  Python          MIT
├── geo-optimizer-go/     # 8  Go              MIT
├── eGEOagents/           # 9  Python          MIT
├── _logs/                # 各仓库 git clone 日志
└── GEO-开源项目调研.md    # 本文件
```

## 五、完整性校验

对每个仓库执行了 `git rev-list --count HEAD`、`git fsck`、`git rev-parse HEAD`：

- 9/9 仓库分支正常解析（8 个 `main`，`getcito` 为 `master`）
- 9/9 `git fsck` 无错误输出
- 扫描 `.gitmodules` → 无 submodule，无未初始化内容
- 扫描 `.gitattributes` → 无 git-lfs 过滤器，无未解析的指针文件
- 均为完整克隆（非浅克隆），可自由 `git log` / `git checkout` 历史版本

## 六、备注

- `GEOFlow` 与 `GEORank` 同属作者 yaojingang，两者定位互补（前者做内容生产与分发，后者做诊断分析）。
- `elmo` 与 `getcito` 功能高度重叠（都是 AI 可见性追踪平台），可对比阅读。
- 检索时另有两个高星非代码仓库可供延伸阅读：
  `amplifying-ai/awesome-generative-engine-optimization`（506★）、`luka2chat/awesome-geo`（143★）。
- 若后续需要，可用 `git -C <目录> pull` 更新，或 `git remote -v` 查看原始地址。
