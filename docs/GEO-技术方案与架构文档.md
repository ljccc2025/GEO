# GEO 作战平台 — 全栈技术方案与架构文档

> 本文档仿照《SQL AI 智批改软件 — 全栈技术方案与架构文档》的结构编写。
> 配套设计规格：`docs/superpowers/specs/2026-09-14-geo-platform-design.md`

---

## 一、需求深度复述与分析

### 1.1 功能模块分析

| 功能模块 | 需求描述 | 技术要求 | 优先级 |
|---------|---------|---------|--------|
| 公司知识库 | 录入设计能力、资质、获奖、合作品牌、案例 | 结构化切片 + 向量召回；来源可核验 | P0 |
| 内容生产 | 批量生成面向海外买手的专业内容 | LLM 生成 + 知识库 RAG 约束 | P0 |
| AI 质检门禁 | 发布前逐篇检查，不通过则拦在草稿 | 知识证据/引文/广告规则/法规依据四维检查 | P0 |
| 目标问题管理 | 管理「希望 AI 推荐我们」的提问清单 | 关键词库 + 后台一键触发采集 | P0 |
| AI 可见性采集 | 向海外 AI 引擎提问，记录回答原文 | 检索型引擎接入 + 多引擎并行采样 | P0 |
| 可见性分析 | 计算可见率、排名、情感、信源偏好 | 品牌别名字符串匹配 + 信源归并 | P0 |
| 竞品检测 | 从 AI 回答中识别竞品 | LLM 抽取 + 竞品库 | P1 |
| 公司结构化档案 | 把公司事实变成 AI 能读懂的字段 | Schema.org Organization 输出 | P0 |
| 站点技术层输出 | 每个分发站点自带 Schema/sitemap/llms.txt | 与站点包生成器集成 | P0 |
| 多站点分发 | 内容投放到官网/托管站点/第三方 | 四条通道 + 失败冷却 | P1 |
| 独立效果验收 | 用第三方工具交叉验证效果 | MIT 工具，独立于主系统 | P1 |
| 权限与审计 | 谁能改什么、改过什么可回溯 | 角色过滤 + 操作日志 | P1 |
| 队列可观测 | 任务积压可见，失败可重试 | 按队列名监控 5 个 worker 服务 | P1 |

### 1.2 非功能需求分析

| 需求维度 | 具体要求 | 技术挑战 |
|---------|---------|---------|
| 部署形态 | 公司自有服务器，Docker Compose，内网访问 | 12 个服务 / 默认 13 进程的编排与依赖顺序 |
| 企业级治理 | 权限、审计、备份、恢复点、可升级 | 复用底座既有能力，不重复造 |
| 网络边界 | 内网访问；但需出站访问海外 AI API | 国内服务器访问 api.openai.com 等需额外网络条件（**未验证假设 A1**） |
| 许可证合规 | AGPL-3.0 边界必须守住 | 公开站点只发静态产物，不暴露运行实例 |
| 可配置性 | 品类/能力点/目标问题全部数据化 | 业务信息未定（P6），必须零代码扩展 |
| 成本可控 | 海外 API 按次计费，采样是定时任务 | 复用配额机制，按月核算 |
| 可验收性 | 能客观判断「AI 是否推荐我们」 | AI 回答有随机性，单次采样不构成结论 |
| 可维护性 | 上游迭代快（近两月 408 次提交） | 定制收敛到新文件，对上游文件改动最小化 |

### 1.3 隐含技术挑战

| 挑战 | 描述 | 影响范围 |
|-----|------|---------|
| 品牌实体归并 | AI 靠 `sameAs` 把 LinkedIn、B2B 店铺、官网认作同一实体；当前 Schema 只有一个 `name` | 结构化输出、可见性判定 |
| 采样源与市场错配 | 底座只支持豆包/DeepSeek，目标市场在欧美，测错市场等于没有验收标准 | 可见性模块整体 |
| 单路径调度 | 采样调度是 if/else 链，只走一条路径；新引擎加 Client 也不会被调用 | 引擎接入 |
| 品牌判定是字符串匹配 | 判定「AI 是否提及我们」靠别名匹配，配置不当则测的是开源项目名 | 全部验收标准 |
| SSRF 白名单 | 出站域名白名单写死，扩白名单是安全敏感改动 | 海外引擎接入的前置 |
| AI 回答随机性 | 同一问题多次询问结果不同，趋势才有意义 | 验收方法设计 |

### 1.4 本系统与通用 GEO SaaS / 内容农场的本质区别

| 对比维度 | 本系统 | 通用 GEO SaaS | 内容农场 |
|---------|--------|--------------|---------|
| 服务对象 | 只服务本公司获客（对内单租户） | 多租户，服务多个客户 | 流量变现 |
| 内容来源 | 公司真实知识库 + 来源可核验 | 客户提供 | 抓取/编造 |
| 质量门禁 | 发布前四维强制检查，不通过不发 | 通常无 | 无 |
| 结构化输出 | 完整 Organization Schema（含 `sameAs`/`award`/`brand`） | 部分支持 | 无 |
| 效果验收 | 海外引擎多引擎采样 + 第三方工具交叉验证 | 平台自带指标 | 看流量 |
| 反馈闭环 | 可见率下降 → 定位文章 → 生成优化任务 | 少有 | 无 |
| 长期性 | 真实内容 × 广泛分发，可长期有效 | 有效 | 会被过滤，域名受损 |

---

## 二、终极技术栈选型（带具体版本及决策矩阵）

### 2.1 底座平台

| 维度 | 详情 |
|------|------|
| **我们的选择** | **GEOFlow 3.1.0**（`yaojingang/GEOFlow`，commit `f7c75e6`） |
| 备选方案 | GEORank（Apache-2.0）、elmo（MIT）、纯自研 |
| 决定性理由 | 1) 它本身就是一个 **Web 应用**——管理后台 + 内容生产 + 质检 + 分发 + 可见性分析，正是要交付的形态；2) 能力闭环最全，其余候选只做诊断或监测，不含内容生产流水线；3) 企业级治理现成（权限/审计/签名更新/备份/恢复点/防 SSRF）；4) **配置驱动**——知识库、关键词库、标题库都是后台可管理的表，契合「业务信息未定」；5) 支持简体中文界面 |
| 关键应用点 | Laravel 应用主体、Admin UI V3（37 模块 / 141 模板）、26 套站点主题、24 张内置帮助截图 |
| ⚠ 约束 | **AGPL-3.0**。内网自用可控；公开站点只发静态产物，不暴露运行实例 |

### 2.2 运行时与语言

| 维度 | 详情 |
|------|------|
| **我们的选择** | **PHP 8.3+（Docker 镜像默认 PHP 8.4）+ Laravel** |
| 备选方案 | 用 GEORank 走 Python/FastAPI 栈；或纯自研选 Node/Go |
| 决定性理由 | 1) 二次开发的成本主要在读懂并复用既有模块，而不是选一门顺手的语言；2) 底座的 408 次提交与 323 个测试文件都是 PHP，换栈等于全部重来；3) 团队技术栈不受限（用户明确表示可按实际需要选），因此以**复用最大化**为准；4) Docker 部署下本地无需安装 PHP |
| 关键应用点 | `app/Services/GeoFlow/` 业务层、`app/Models/` 数据层、`app/Http/Controllers/Admin/` 后台层 |

### 2.3 主数据库与向量存储

| 维度 | 详情 |
|------|------|
| **我们的选择** | **PostgreSQL 16 + pgvector 扩展** |
| 备选方案 | MySQL + 独立向量库（Qdrant/Milvus） |
| 决定性理由 | 1) 底座原生要求 PostgreSQL，且推荐 pgvector 镜像；2) 知识库切片（`chunk_*` 字段）直接依赖向量能力；3) **业务数据与向量同库**，避免两套存储的一致性问题与额外运维；4) 全部业务资产（知识库、文章、可见性数据）集中在 PostgreSQL，备份即备份一切 |
| 关键应用点 | 知识库切片与语义召回、可见性数据、分发日志、公司档案 |

### 2.4 缓存与队列

| 维度 | 详情 |
|------|------|
| **我们的选择** | **Redis 8**（队列 + 缓存 + 运行状态） |
| 备选方案 | 数据库队列、RabbitMQ |
| 决定性理由 | 1) 底座已按 Redis 设计队列；2) 内容生产、AI 质检、可见性采集全部是长耗时任务，必须有独立 worker；3) 队列**按业务域拆分**是底座既有设计（质量检查、优化、知识各自独立队列），避免长任务阻塞短任务 |
| 关键应用点 | 5 个 worker 服务消费不同队列（详见模块 01） |

### 2.5 容器编排

| 维度 | 详情 |
|------|------|
| **我们的选择** | **Docker Compose**（`docker-compose.prod.yml`） |
| 备选方案 | Kubernetes、裸机部署 |
| 决定性理由 | 1) 底座官方支持 Compose，生产用 Nginx + php-fpm；2) 单服务器、内网、单租户场景下 K8s 属于过度设计（YAGNI）；3) Compose 的编排复杂度与运维人力匹配 |
| 关键应用点 | 12 个服务、默认 13 个进程；依赖顺序 postgres/redis → init → app → web/queue |

### 2.6 前端构建

| 维度 | 详情 |
|------|------|
| **我们的选择** | **Vite 7 + Tailwind CSS 4**（底座内置） |
| 备选方案 | 不改造，直接用底座产物 |
| 决定性理由 | 1) 二次开发若涉及后台界面样式，需能重新构建；2) 底座已配好 `vite.config.js` 与 `laravel-vite-plugin`；3) 附带 Vditor（Markdown 编辑器）、CropperJS（图片裁剪）、Laravel Echo + Pusher（实时） |
| 关键应用点 | `npm run build` 产出前端资源 |

### 2.7 内容生产模型（LLM 接入）

| 维度 | 详情 |
|------|------|
| **我们的选择** | **底座兼容的多 Provider 策略**，后台配置 API 池，支持轮询与故障转移 |
| 备选方案 | 单一供应商硬编码 |
| 决定性理由 | 1) 底座 AI 层兼容 OpenAI 格式的 Chat 与 Embedding Provider，后台可配 base_url / model / dimensions；2) 多 Provider + 轮询避免单点限流导致生产停摆；3) 写作模型与质检模型可分开配置 |
| 关键应用点 | `app/Http/Controllers/Admin/AiModelController.php`、后台「AI 配置器」 |

### 2.8 AI 可见性采样引擎（海外）★ 核心二次开发项

| 维度 | 详情 |
|------|------|
| **我们的选择** | **Perplexity + OpenAI，两者均建模为检索型 `AiSourceProvider`** |
| 备选方案 | (a) 只接 Perplexity；(b) 建模为模型调用型 `AiModel`（无检索）；(c) 继续用豆包/DeepSeek |
| 决定性理由 | 1) 目标市场在欧美，测国内引擎等于没有验收标准；2) **必须是检索型**——验收意图是「买手提问时 AI 是否找到并推荐我们」，无检索的模型调用测的是「训练语料里有没有我们」，是另一个问题；3) 检索型会产生 `ai_visibility_sources`，那是信源与引用排名分析的**唯一数据来源**；4) Perplexity 自带网页检索 API，OpenAI 走 Responses API + `web_search` 工具 |
| 关键应用点 | `app/Services/GeoFlow/AiVisibility/` 下新增 Client；`AiVisibilityRun::SAMPLE_PROVIDERS` 挂常量；采样调度与配置解析同步扩展 |
| ⚠ 前置 | 出站域名白名单必须放行新端点（安全敏感，见模块 06） |

### 2.9 结构化数据输出

| 维度 | 详情 |
|------|------|
| **我们的选择** | **Schema.org Organization JSON-LD**，独立服务渲染 |
| 备选方案 | 把公司信息写进知识库正文，靠 AI 自行理解 |
| 决定性理由 | 1) 需求原文要求「提供**结构化**设计资质与合规标准」；2) 自由文本里出现「红点奖」三个字，与 `award` 字段是两个量级的信息——后者 AI 能直接消费；3) `sameAs` 是实体归并的关键，缺失则 AI 无法把网络上零散的公司信息归并为同一实体；4) 独立成服务便于单元测试与复用 |
| 关键应用点 | 新增 `OrganizationSchemaBuilder`；替换 `DistributionTargetSitePackageBuilder` 中仅含 `name` 的实现（约 L3322，**定点替换，不重构该 3796 行文件**） |

### 2.10 品牌身份

| 维度 | 详情 |
|------|------|
| **我们的选择** | **环境变量配置**（`SITE_NAME` / `SITE_FULL_NAME` / `APP_NAME` / `SITE_URL`） |
| 备选方案 | 改代码让判定读公司档案 |
| 决定性理由 | 1) 底座判定「AI 是否提及我们」是**纯字符串匹配**，默认值是 `GEOFlow` 和 `localhost`；2) 不配置则基线测的是开源项目名，全部验收标准失效；3) 本期以环境变量为度量口径、公司档案为输出口径是**有意的临时双源**——在没有真实数据前合并两者只会增加改动面（YAGNI） |
| 关键应用点 | `AiVisibilityAnalyticsService::brandAliases()` L940-959、`ownedHosts()` L964-975；配置项 `config/geoflow.php` L76-80 |

### 2.11 独立效果验收工具

| 维度 | 详情 |
|------|------|
| **我们的选择** | **geo-optimizer-skill**（MIT，Python 3.9+） |
| 备选方案 | 只用底座自带的可见性看板 |
| 决定性理由 | 1) 只用底座看板等于**让做工具的人用自己的工具验收自己**——可见性模块与内容策略是同一套假设，很可能同时「看起来有效」；2) 该工具的引擎清单正是 `OAI-SearchBot` / `PerplexityBot` / `ClaudeBot` / `Google-Extended`；3) MIT 协议，无任何义务；4) `geo citations --runs 5` 支持多次采样给置信区间 |
| 关键应用点 | `geo audit` 官网技术层体检；`geo citations` 独立问答验证；100 分制评分（抓取类占 42 分） |

### 2.12 测试与质量保证

| 维度 | 详情 |
|------|------|
| **我们的选择** | **PHPUnit**，遵循底座既有惯例（`tests/Feature` / `tests/Unit`） |
| 备选方案 | 只做手工验证 |
| 决定性理由 | 1) 底座有 **323 个测试文件**（Feature 182 · Unit 119 · PostgreSQL 13 · Support 5 · Fixtures 2 · Performance 1 · 顶层 1）与 CI，不写测试就是破坏惯例；2) 海外引擎 Client 涉及外部 HTTP，必须 mock，否则测试不稳定且产生费用；3) 安全敏感的端点策略改动尤其需要断言保护 |
| 关键应用点 | 新增 Client 单元测试（mock HTTP）、Schema 渲染单元测试、采集链路 Feature 测试 |

---
## 三、高保真系统架构与模块详设

### 3.1 架构全景图（容器模型）

```
                     公司内网（不对公众暴露）
  ┌──────────────────────────────────────────────────────────────┐
  │  浏览器：仅公司内部运营同事                                     │
  │      │                                                        │
  │      ▼                                                        │
  │  ┌────────┐   ┌─────────┐                                    │
  │  │  web   │──▶│   app   │  Nginx + php-fpm / Laravel 应用主体  │
  │  └────────┘   └────┬────┘                                    │
  │                    │                                          │
  │        ┌───────────┼───────────┬──────────────┐              │
  │        ▼           ▼           ▼              ▼              │
  │  ┌──────────┐ ┌─────────┐ ┌────────┐   ┌───────────┐        │
  │  │ postgres │ │  redis  │ │ init   │   │ scheduler │        │
  │  │ +pgvector│ │队列/缓存│ │迁移初始化│  │ 定时任务  │        │
  │  └──────────┘ └────┬────┘ └────────┘   └───────────┘        │
  │                    │                                          │
  │     ┌──────────────┴───────────────┐                        │
  │     ▼              ▼               ▼                        │
  │  ┌────────┐  ┌──────────────┐  ┌──────────────────┐        │
  │  │ queue  │  │ai-quality-*  │  │ai-optimization-* │        │
  │  │系统/分发│  │质检(默认2副本)│  │内容优化          │        │
  │  └────────┘  └──────────────┘  └──────────────────┘        │
  │     ▼                                                        │
  │  ┌───────────────┐        ┌────────┐                        │
  │  │knowledge-queue│        │ reverb │  实时推送              │
  │  └───────────────┘        └────────┘                        │
  └──────────────────────────────────────────────────────────────┘
         │ 出站（需要网络条件，假设 A1 未验证）
         ▼
   ┌──────────────┬────────────────┬──────────────────┐
   │ 写作/质检模型 │  ★ 海外采样引擎 │  分发渠道         │
   │ (OpenAI 兼容) │ Perplexity /    │ 托管站点/WordPress│
   │               │ OpenAI          │ 通用 HTTP API     │
   └──────────────┴────────────────┴──────────────────┘
```

> 图中 `default` 未出现：它是 compose **网络名**，不是服务。
> 12 个服务 / 默认 13 个进程（`ai-quality-queue` 默认 2 副本）。

### 3.2 全部功能模块深度分解（16 个核心模块）

图例：★ 新增 ｜ ◆ 改造 ｜ ● 复用（底座既有，不改代码）

---

#### 模块 M01: 部署编排与队列拓扑  ◆ 改造

| 属性 | 内容 |
|------|------|
| **模块ID** | M01 |
| **物理文件路径** | `docker-compose.prod.yml`、`.env.prod` |
| **核心职责** | 定义 12 个服务、依赖顺序、健康检查与卷挂载；把队列按业务域拆分 |
| **对外API** | `docker compose --env-file .env.prod -f docker-compose.prod.yml up -d` |
| **内部技术** | Docker Compose, Nginx 1.x, php-fpm, PostgreSQL 16 + pgvector, Redis 8 |
| **交互流程** | 构建镜像 → 起 postgres/redis → init 跑迁移 → 起 app → 起 web 与各 worker → scheduler 接管定时任务 |

**🔧 核心技术栈**:
- `postgres:16-alpine` + pgvector 扩展 — 主库与向量存储
- `redis:8` — 队列 broker、缓存、运行状态
- 5 个 worker 服务，**命令并不统一**（监控必须按队列名）
- `ai-quality-queue` 默认 `replicas: 2`

**🎯 推荐Skills**: `terraform-infrastructure`（编排概念参考）、`senior-fullstack`（部署架构）

**关键：5 个 worker 服务与它们的实际命令**

| 服务 | 实际命令 |
|---|---|
| `queue` | `queue:work --queue=system-updates,geoflow,distribution,theme-replication,default` |
| `ai-quality-queue` | `geoflow:work-ai-quality front` |
| `ai-quality-backfill-queue` | `geoflow:work-ai-quality backfill` |
| `ai-optimization-queue` | `geoflow:work-ai-optimization` |
| `knowledge-queue` | `queue:work --queue=knowledge` |

> 另有 `scheduler`（`schedule:work`）与 `reverb`（`reverb:start`）。
> **监控告警要按队列积压量设计，不能按容器名**——三个 worker 根本不用 `queue:work`。

---

#### 模块 M02: 品牌身份配置  ◆ 改造（配置，非编码）

| 属性 | 内容 |
|------|------|
| **模块ID** | M02 |
| **物理文件路径** | `.env.prod`（`SITE_NAME` / `SITE_FULL_NAME` / `APP_NAME` / `SITE_URL`）；读取方 `app/Services/Admin/Analytics/AiVisibilityAnalyticsService.php` |
| **核心职责** | 让系统知道「本公司叫什么」，这是所有可见性验收的前提 |
| **对外API** | 无代码 API；通过环境变量生效 |
| **内部技术** | 纯字符串匹配（`brandAliases()` / `ownedHosts()`），非语义识别 |
| **交互流程** | 配置 env → 重启服务 → 用一条已知会提及本公司的问句跑采集 → 确认看板能识别出本品牌 |

**🔧 核心技术栈**:
- `config/geoflow.php` L76-80 — 品牌配置项与默认值
- `AiVisibilityAnalyticsService::brandAliases()` L940-959 — 别名集合
- `AiVisibilityAnalyticsService::ownedHosts()` L964-975 — 自有域名集合

**🎯 推荐Skills**: `chinese-documentation`（配置说明撰写）

**⚠ 这是 M0 的前置条件，不配置则一切验收失效**

```php
brandAliases() -> config('geoflow.site_name')      // 默认 'GEOFlow'
                 config('geoflow.site_full_name')  // 默认 'GEOFlow'
                 'GEOFlow'                          // 硬编码
                 config('app.name')
ownedHosts()   -> config('geoflow.site_url')       // 默认 'http://localhost'
                 config('app.url')
```

公司当前**没有域名**，因此 `ownedHosts` 为空，判定 **100% 依赖别名字符串**。
别名必须覆盖海外买手可能使用的各种写法（公司英文全称、简称、品牌名、常见拼写变体）。

---

#### 模块 M03: AI 可见性采集调度  ◆ 改造

| 属性 | 内容 |
|------|------|
| **模块ID** | M03 |
| **物理文件路径** | `app/Services/GeoFlow/AiVisibility/AiVisibilityCollectionService.php` |
| **核心职责** | 把目标问题分发给各采样引擎，收集回答 |
| **对外API** | `collect(SystemAiIdentity $identity, string $keyword): array<string, AiVisibilityRun>` |
| **内部技术** | 当前是 if/else 单路径链，按优先级只走**一条**分支 |
| **交互流程** | 命令 `geoflow:ai-visibility:collect` 或后台入口 → `CollectAiVisibilityKeywordJob` → `collect()` → 选中引擎 → 写 `ai_visibility_runs` |

**🔧 核心技术栈**:
- `AiVisibilityCollectionService::collect()` L20-67 — 待改造的唯一入口
- `CollectAiVisibilityKeywordJob` — 异步执行，`ShouldBeUnique` + `ShouldQueue`

**🎯 推荐Skills**: `systematic-debugging`（调度不生效排查）、`test-driven-development`

**改造要求**：让新引擎可被选中，并明确**同一目标问题对两个引擎各采一次**。

这是原设计的漏洞：只新增 Client 而不改这个入口，新 Client **永远不会被调用**。

---

#### 模块 M04: Perplexity 采样 Client  ★ 新增

| 属性 | 内容 |
|------|------|
| **模块ID** | M04 |
| **物理文件路径** | `app/Services/GeoFlow/AiVisibility/PerplexitySearchClient.php`（命名待定） |
| **核心职责** | 向 Perplexity 提问并取回带网页引用的回答 |
| **对外API** | 由 `AiVisibilityService::run*` 包装后调用 |
| **内部技术** | 检索型 `AiSourceProvider`（endpoint + api_key + daily_limit） |
| **交互流程** | 取配置 → HTTP 请求 → 归一化结果 → 落库 `ai_visibility_runs` + `ai_visibility_sources` |

**🔧 核心技术栈**:
- 参照既有 `DoubaoSearchCustomClient` 的结构
- `AiVisibilityResultNormalizer` — 结果归一化
- `AiVisibilityHttpClientFactory` — HTTP 客户端工厂

**🎯 推荐Skills**: `python-pro`（HTTP 客户端健壮性参考）、`systematic-debugging`

**必须产生 `ai_visibility_sources`** —— 那是信源与引用排名分析的唯一数据来源。
若实现成无检索的普通调用，本模块等于白做。

---

#### 模块 M05: OpenAI 采样 Client  ★ 新增

| 属性 | 内容 |
|------|------|
| **模块ID** | M05 |
| **物理文件路径** | `app/Services/GeoFlow/AiVisibility/OpenAiResponsesClient.php`（命名待定） |
| **核心职责** | 用 OpenAI Responses API + `web_search` 工具取回带引用的回答 |
| **对外API** | 由 `AiVisibilityService::run*` 包装后调用 |
| **内部技术** | 检索型 `AiSourceProvider`；`web_search` 工具启用 |
| **交互流程** | 取配置 → Responses API 请求（带 web_search）→ 解析引用 → 落库 |

**🔧 核心技术栈**:
- OpenAI Responses API + `web_search` 工具
- 参照既有 Client 的错误处理与超时约定
- 复用 `AiVisibilityResultNormalizer`

**🎯 推荐Skills**: `api-patterns`（外部 API 集成）、`systematic-debugging`

**不得**退化为普通 chat/completions 调用——那测的是「训练语料里有没有我们」，不产生信源数据，与验收意图不符。

---

#### 模块 M06: 出站端点策略  ◆ 改造（⚠ 安全敏感）

| 属性 | 内容 |
|------|------|
| **模块ID** | M06 |
| **物理文件路径** | `app/Services/GeoFlow/AiVisibility/AiProviderEndpointPolicy.php` |
| **核心职责** | 限制可见性模块可以访问哪些外部端点（防 SSRF） |
| **对外API** | `acceptsModelApi(string $bindingType, string $url): bool`、`acceptsSearchApi(string $url): bool` |
| **内部技术** | HTTPS 域名白名单，支持子域匹配 |
| **交互流程** | 取配置 URL → 解析 host → 与白名单比对 → 通过/拒绝 |

**🔧 核心技术栈**:
- 当前白名单：`ark => volces.com`、`deepseek => deepseek.com`、搜索 `feedcoopapi.com`
- **必须新增**：`api.perplexity.ai`、`api.openai.com`

**🎯 推荐Skills**: `api-security-best-practices`、`receiving-code-review`

**当前白名单是写死的**，新增端点会被直接拒绝。

**硬性约束**：
- 项目规则明示「绝不允许绕过防 SSRF 保护」，**不得**放宽为通配或跳过校验
- 必须配单元测试：断言新端点被允许、**同时断言非白名单端点仍被拒绝**
- 按项目既有安全验证流程走

---

#### 模块 M07: 可见性配置解析  ◆ 改造

| 属性 | 内容 |
|------|------|
| **模块ID** | M07 |
| **物理文件路径** | `app/Services/GeoFlow/AiVisibility/AiVisibilityConfigurationResolver.php` |
| **核心职责** | 决定用哪个模型/搜索源来做采样 |
| **对外API** | `searchProvider()`、`arkModel()`、`deepSeekModel()`、`status()` |
| **内部技术** | 站点设置表（`SiteSetting`）存配置键；`searchProvider()` 当前**硬编码** `provider_key = doubao_search_custom` |
| **交互流程** | 读站点设置 → 校验端点策略 → 校验已存 API Key → 返回可用资源 |

**🔧 核心技术栈**:
- 现有配置键仅 2 个：`ARK_MODEL_SETTING_KEY`、`DEEPSEEK_MODEL_SETTING_KEY`
- 需为两个新引擎新增配置键并纳入解析

**🎯 推荐Skills**: `systematic-debugging`

**改造要求**：新增配置键、扩展 `searchProvider()` 使其不再硬编码单一 provider_key，并让 `status()` 能反映新引擎的可用性（后台「去配置」提示依赖它）。

---

#### 模块 M08: 公司档案实体  ★ 新增

| 属性 | 内容 |
|------|------|
| **模块ID** | M08 |
| **物理文件路径** | `database/migrations/*_create_organization_profiles_table.php`、`app/Models/OrganizationProfile.php` |
| **核心职责** | 以**结构化字段**承载公司事实，供 Schema 输出消费 |
| **对外API** | Eloquent 模型；字段见下 |
| **内部技术** | 单行/单例语义；字段强类型 |
| **交互流程** | 后台录入 → 落库 → 被 Schema 渲染服务读取 |

**🔧 核心技术栈**:
- Laravel Migration + Eloquent
- 字段对齐 Schema.org Organization 词汇表

**🎯 推荐Skills**: `python-pro`（数据建模参考）、`software-architecture`

**字段清单（R2.2 要求覆盖）**

| 字段 | Schema.org 对应 | 说明 |
|---|---|---|
| `legal_name` / `alternate_name` | `name` / `alternateName` | 公司法定名与别名 |
| `url` | `url` | 官网 |
| `logo` | `logo` | 绝对 URL |
| `description` | `description` | 公司简介 |
| `address` | `address` | 结构化地址 |
| `contact_point` | `contactPoint` | 联系方式 |
| **`same_as`** | **`sameAs`** | **★ 关键**：LinkedIn / B2B 店铺 / 官网等外部权威链接 |
| **`award`** | **`award`** | 设计获奖记录 |
| **`brand` / `member_of`** | `brand` / `memberOf` | 合作品牌、所属协会 |
| **`has_credential`** | `hasCredential` | 设计资质与认证（ISO/BSCI 等） |
| `knows_about` | `knowsAbout` | 专业领域（工业设计、耳塞） |
| `founding_date` | `foundingDate` | 成立时间 |

---

#### 模块 M09: 公司档案后台管理  ★ 新增

| 属性 | 内容 |
|------|------|
| **模块ID** | M09 |
| **物理文件路径** | `app/Http/Controllers/Admin/OrganizationProfileController.php` + `resources/views/admin/organization-profile/*.blade.php` + 路由 |
| **核心职责** | 让运营同事在后台维护公司档案 |
| **对外API** | 后台 CRUD 路由 |
| **内部技术** | 沿用 Admin UI V3 的布局与权限中间件 |
| **交互流程** | 打开页面 → 编辑各字段 → 保存 → 校验（URL 必须绝对地址）→ 落库 |

**🔧 核心技术栈**:
- Blade 模板 + Admin UI V3 布局
- 复用既有权限中间件与审计日志

**🎯 推荐Skills**: `laravel-best-practices`（底座内置 skill）、`vue-best-practices`（不适用，底座用 Blade）

**约束**：只新增文件，不改动底座既有的站点设置等后台模块，降低与上游的冲突面。

---

#### 模块 M10: Organization Schema 渲染服务  ★ 新增

| 属性 | 内容 |
|------|------|
| **模块ID** | M10 |
| **物理文件路径** | `app/Services/GeoFlow/OrganizationSchemaBuilder.php` |
| **核心职责** | 把公司档案渲染为完整的 Organization JSON-LD |
| **对外API** | `build(OrganizationProfile $profile): array` |
| **内部技术** | 纯函数式：输入档案，输出 JSON-LD 数组，无副作用 |
| **交互流程** | 读档案 → 组装 JSON-LD → 返回（由站点生成器序列化注入） |

**🔧 核心技术栈**:
- PHP 数组 → `jsonLdScript()` 序列化（底座既有辅助函数）
- 字段为空时**不得输出空数组或 null**（会污染结构化数据）

**🎯 推荐Skills**: `test-driven-development`、`verification-before-completion`

**必须可独立单元测试**：给定档案对象，断言输出结构与字段。

**为空字段的处理规则**：字段无值时**整体省略该键**，而不是输出 `[]` 或 `null` —— 后者会被 Schema 校验器判为无效。

---

#### 模块 M11: 站点 JSON-LD 注入  ◆ 改造（定点替换）

| 属性 | 内容 |
|------|------|
| **模块ID** | M11 |
| **物理文件路径** | `app/Services/GeoFlow/DistributionTargetSitePackageBuilder.php` 约 L3322 |
| **核心职责** | 在生成的站点中注入完整 Organization 结构化数据 |
| **对外API** | 内部私有方法 |
| **内部技术** | 该文件 **3796 行**，属巨型文件 |
| **交互流程** | 生成站点包 → 组装 head → 输出 JSON-LD（当前仅 `@type` + `name`） |

**🔧 核心技术栈**:
- 当前实现：`"publisher" => ["@type"=>"Organization", "name"=>$settings['site_name']]`
- 改为调用 `OrganizationSchemaBuilder`

**🎯 推荐Skills**: `anti-entropy-governance`（避免扩散改动面）、`first-principles-review`

**硬性约束：只做定点替换，不重构该 3796 行文件。**

原范围包含「不做无关重构」。同时注意：全仓库仅此一处生成 Organization JSON-LD，改动面可控。

站点包支持 `front_mode = static`（默认），公开站点只发静态产物 —— 这也是 AGPL 边界的落地方式（见 §5.2）。

---

#### 模块 M12: 知识库与内容生产  ● 复用

| 属性 | 内容 |
|------|------|
| **模块ID** | M12 |
| **物理文件路径** | `app/Models/KnowledgeBase.php`、`app/Models/EnterpriseKnowledge*.php`、`app/Jobs/*` |
| **核心职责** | 承载公司知识，驱动批量内容生产 |
| **对外API** | 后台界面 + 任务队列 |
| **内部技术** | 结构化切片 + 语义规划 + 向量召回 + 稳定回退 |
| **交互流程** | 录入知识 → 切片 → 向量化 → 选题 → AI 写作 → 产出文章草稿 |

**🔧 核心技术栈**:
- `knowledge_bases` 表含 `business_line` / `source_url` / `source_type` / `risk_level` / `effective_date` / `chunk_*`
- 企业知识含版本（`EnterpriseKnowledgeRevision`）与来源（`EnterpriseKnowledgeSource`）

**🎯 推荐Skills**: （无需开发，运营录入即可）

**业务信息全部配置化在这里落地**：品类写进 `business_line`，能力点写进企业知识，资质写进知识库并带来源 URL —— 换品类只加记录，不改代码。

---

#### 模块 M13: AI 质检门禁  ● 复用

| 属性 | 内容 |
|------|------|
| **模块ID** | M13 |
| **物理文件路径** | `app/Models/ArticleAiQualityCheck.php`、`ArticleRiskScan.php` |
| **核心职责** | 发布前按知识证据、引文、广告规则、法规依据逐项检查 |
| **对外API** | 任务队列驱动，结果写入文章记录 |
| **内部技术** | 分项评分 + 原文定位 + 法规依据 + 修改建议 + 历史结果 |
| **交互流程** | 文章完成 → 质检任务 → 未通过则停留草稿 → 人工放行 → 才可发布 |

**🔧 核心技术栈**:
- `article_ai_quality_checks` / `article_ai_quality_check_sources`
- `article_risk_scans` — 风险扫描

**🎯 推荐Skills**: （无需开发）

**这个模块是「真实内容」与「内容农场」的分界线**。证据不足的文章被强制拦在草稿阶段 —— 这也是本系统能长期有效、不被 AI 引擎过滤的关键。

---

#### 模块 M14: 多站点分发  ● 复用

| 属性 | 内容 |
|------|------|
| **模块ID** | M14 |
| **物理文件路径** | `app/Models/DistributionChannel.php`、`DistributionTargetSitePackageBuilder.php` |
| **核心职责** | 把已发布文章投放到官网与第三方渠道 |
| **对外API** | 四条通道：托管站点 / 目标站点包 / WordPress REST / 通用 HTTP API |
| **内部技术** | 失败冷却 + 技术预检 + 状态对账 + 生命周期管理 |
| **交互流程** | 文章发布 → 选择渠道 → 分发任务 → 落 `distribution_logs` → 失败按冷却重试 |

**🔧 核心技术栈**:
- `distribution_channels` / `article_distributions` / `distribution_logs`
- 人工发布平台：知乎·小红书·微博·公众号·抖音·B站·**Reddit·X·LinkedIn**·自定义

**🎯 推荐Skills**: （无需开发）

**Reddit 值得特别关注** —— 它是 AI 训练语料与 Perplexity 检索的高权重来源，海外买手也常在 Reddit 问采购问题，底座已内置该渠道。

---

#### 模块 M15: 可见性分析与看板  ● 复用（受 M02 影响）

| 属性 | 内容 |
|------|------|
| **模块ID** | M15 |
| **物理文件路径** | `app/Services/Admin/Analytics/AiVisibilityAnalyticsService.php`、`AiVisibilityAnalyticsController.php` |
| **核心职责** | 计算可见率、排名、情感、信源偏好、采样质量；竞品检测 |
| **对外API** | 后台 `/admin/ai-visibility` 页面；`POST ai-visibility/collect` |
| **内部技术** | `SAMPLE_PROVIDERS` 常量驱动的分析管道（8 个文件引用，**新引擎挂常量后自动生效**） |
| **交互流程** | 采集落库 → 分析管道 → 看板展示 → 竞品检测任务 |

**🔧 核心技术栈**:
- `ai_visibility_runs` / `ai_visibility_sources` / `ai_visibility_competitors`
- 目标问题来自关键词库，单次最多 50 条，单条限 100 字符（**超长静默过滤**）

**🎯 推荐Skills**: （无需开发）

**收益**：因为引擎清单集中在 `SAMPLE_PROVIDERS` 一个常量里，8 个分析类文件**一行都不用改**。

---

#### 模块 M16: 独立效果验收工具  ★ 外部（独立部署）

| 属性 | 内容 |
|------|------|
| **模块ID** | M16 |
| **物理文件路径** | 独立 Python 环境；`pip install geo-optimizer-skill` |
| **核心职责** | 用第三方工具交叉验证「AI 是否推荐我们」 |
| **对外API** | `geo audit` / `geo citations` / `geo track` |
| **内部技术** | MIT 协议；100 分制评分模型 |
| **交互流程** | 安装 → `geo audit --url` 体检官网 → `geo citations --brand --topic --runs 5` 独立问答 → 对比底座看板 |

**🔧 核心技术栈**:
- `geo audit` — 100 分制（Robots.txt 18 / llms.txt 18 / Schema 16 / Meta 14 / Content 12 / Signals 6 / AI Discovery 6 / Brand & Entity 10）
- `geo citations --runs 5` — 同一问题采样 5 次给置信区间
- 爬虫清单：`OAI-SearchBot` / `PerplexityBot` / `ClaudeBot` / `Google-Extended`

**🎯 推荐Skills**: `verification-before-completion`

**为什么必须有它**：只用底座看板等于让做工具的人用自己的工具验收自己 —— 可见性模块与内容策略是同一套假设，很可能同时「看起来有效」。独立工具是**交叉验证**。

---

## 四、项目工程化终极指南

### 4.1 完整项目目录树

```
D:\上班的东西\geo\                     ← 本项目仓库（ljccc2025/GEO）
│
├── README.md                            # 项目定位、结构、进度
├── .gitignore                           # 排除 9 个上游参考仓库
│
├── docs/
│   ├── superpowers/specs/
│   │   └── 2026-09-14-geo-platform-design.md   # ★ 设计规格
│   ├── GEO-技术方案与架构文档.md               # ★ 本文档
│   └── archive/                                # 已作废的早期文档
│
├── GEO-开源项目调研.md                   # 9 个开源 GEO 项目横向调研
├── GEO-最终方案.md                       # 三个子项目总体方案
│
├── GEOFlow/                             # ← 上游底座（gitignore，需单独 clone）
│   ├── app/
│   │   ├── Services/GeoFlow/AiVisibility/   # ★ M1 改造主战场
│   │   │   ├── AiVisibilityCollectionService.php   ◆ M03
│   │   │   ├── AiVisibilityService.php             ◆ 加 run* 方法
│   │   │   ├── AiVisibilityConfigurationResolver.php ◆ M07
│   │   │   ├── AiProviderEndpointPolicy.php        ◆ M06（安全敏感）
│   │   │   ├── DoubaoSearchCustomClient.php        （参照样板）
│   │   │   ├── DeepSeekAnalysisClient.php          （参照样板）
│   │   │   ├── AiVisibilityResultNormalizer.php    （复用）
│   │   │   └── AiVisibilityHttpClientFactory.php   （复用）
│   │   ├── Services/GeoFlow/
│   │   │   ├── DistributionTargetSitePackageBuilder.php  ◆ M11（3796 行，只定点替换）
│   │   │   └── OrganizationSchemaBuilder.php              ★ M10
│   │   ├── Models/
│   │   │   ├── AiVisibilityRun.php                 ◆ M03 挂常量
│   │   │   ├── AiSourceProvider.php                （复用配额机制）
│   │   │   ├── KnowledgeBase.php                   ● M12
│   │   │   └── OrganizationProfile.php             ★ M08
│   │   ├── Http/Controllers/Admin/
│   │   │   ├── AiVisibilityAnalyticsController.php （复用）
│   │   │   └── OrganizationProfileController.php   ★ M09
│   │   └── Console/Commands/
│   │       └── GeoFlowCollectAiVisibilityCommand.php （复用）
│   ├── database/migrations/             # ★ 新增 organization_profiles 迁移
│   ├── resources/views/admin/
│   │   └── organization-profile/        # ★ M09 后台 Blade
│   ├── tests/                           # 323 个测试文件，新代码必须配套
│   ├── lang/zh_CN/                      # 简体中文界面
│   ├── docker-compose.prod.yml          # ◆ M01
│   └── .env.prod                        # ◆ M02 品牌身份
│
└── _logs/                               # 本地工作产物（gitignore）
```

### 4.2 环境变量与配置

```bash
# ==============================
# GEO 作战平台 - 环境变量配置
# 基于 GEOFlow .env.prod 扩展
# ==============================

# ---- ★ 品牌身份（M02，最重要）----
# 可见性判定靠别名做字符串匹配。不配置则基线测的是 'GEOFlow'，一切验收失效。
SITE_NAME=YourCompanyEnglishName
SITE_FULL_NAME=YourCompanyEnglishName Co., Ltd.
APP_NAME=YourCompanyEnglishName
# 域名确定后填写；决定 ownedHosts，用于识别「本站被 AI 引用为信源」
SITE_URL=https://your-domain.example
APP_URL=https://your-domain.example

# ---- 数据库 ----
DB_CONNECTION=pgsql
DB_HOST=postgres
DB_PORT=5432
DB_DATABASE=geoflow
DB_USERNAME=geoflow
DB_PASSWORD=<强密码，不要用默认值>
# pgvector 扩展：使用 pgvector 镜像或在 init 阶段执行 CREATE EXTENSION vector

# ---- Redis ----
REDIS_HOST=redis
REDIS_PORT=6379

# ---- ★ 海外采样引擎（M04/M05）----
# 在后台「AI 配置器」中配置，或以站点设置写入；API Key 走 AiSourceProvider 加密存储
# 注意：出站域名需先在 AiProviderEndpointPolicy 白名单中放行（M06）
PERPLEXITY_API_KEY=<在后台配置，勿写入代码库>
OPENAI_API_KEY=<在后台配置，勿写入代码库>
# 成本保护：AiSourceProvider.daily_limit 限制每引擎每日调用次数

# ---- 队列副本 ----
# ai-quality-queue 默认 2 副本，按服务器规格调整
AI_QUALITY_QUEUE_REPLICAS=2

# ---- 匿名统计 ----
# 默认为关闭。启用后只发送固定白名单字段，业务内容与密钥不会进入载荷。
# GEOFLOW_ANONYMOUS_TELEMETRY=false
```

> ⚠ **密钥纪律**：API Key 一律通过后台配置界面写入（底座加密存储），**不得**提交到代码库。
> 本仓库为公开仓库，任何写进文件的密钥都视为已泄露。

### 4.3 从零启动的「傻瓜式」指南

```bash
# ============ 前置条件 ============
# 1) 一台公司服务器（内网可达即可）
# 2) 安装 Docker Engine + Compose 插件
# 3) 预期能访问海外 AI API（假设 A1，未验证，见 §5.2）

# ---- 步骤 0：先验证网络（M0 第一步，最重要）----
# 这一步不通过，后面全白做
curl -s -o /dev/null -w '%{http_code}\n' https://api.perplexity.ai
curl -s -o /dev/null -w '%{http_code}\n' https://api.openai.com/v1/models
# 返回 401/404 都算通（说明能连上，只是没带密钥）；超时/连接失败说明网络不通

# ---- 步骤 1：获取底座代码 ----
git clone https://github.com/yaojingang/GEOFlow.git
cd GEOFlow
git checkout f7c75e6        # 锁定本项目基于的提交

# ---- 步骤 2：准备生产配置 ----
cp .env.prod.example .env.prod
# 编辑 .env.prod，重点是 §4.2 的「品牌身份」四项

# ---- 步骤 3：构建并启动 ----
docker compose --env-file .env.prod -f docker-compose.prod.yml build
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d postgres redis
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d init      # 跑迁移
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d --remove-orphans \
  app web queue ai-quality-queue ai-quality-backfill-queue \
  ai-optimization-queue knowledge-queue scheduler reverb

# ---- 步骤 4：确认服务健康 ----
docker compose --env-file .env.prod -f docker-compose.prod.yml ps
# 预期：12 个服务 Up；ai-quality-queue 显示 2 个副本

# ---- 步骤 5：验证品牌身份（M0 验收）----
# 先录一条目标问题到关键词库，然后触发采集
docker compose --env-file .env.prod -f docker-compose.prod.yml exec app \
  php artisan geoflow:ai-visibility:collect "your target question"
# 打开后台 /admin/ai-visibility，确认能识别出本品牌

# ---- 步骤 6：建立基线（M1 验收 AC2）----
# 换成海外引擎后重复步骤 5，记录项目起点的可见率
```

### 4.4 常用命令速查

```bash
# ---- 服务 ----
docker compose -f docker-compose.prod.yml ps                    # 状态
docker compose -f docker-compose.prod.yml logs -f app           # 应用日志
docker compose -f docker-compose.prod.yml restart queue         # 重启队列

# ---- ★ 队列积压（最需要盯的地方，按队列名而非容器名）----
docker compose -f docker-compose.prod.yml exec app php artisan queue:monitor \
  system-updates,geoflow,distribution,theme-replication,default,knowledge

# ---- 可见性采集 ----
php artisan geoflow:ai-visibility:collect "<目标问题>"
php artisan geoflow:detect-competitors                  # 竞品检测

# ---- 独立验收（geo-optimizer-skill，MIT）----
geo audit --url https://your-domain.example             # 官网技术层 0-100 分
geo citations --brand "YourBrand" --domain your-domain.example \
  --topic "original design earplug manufacturers in China" --runs 5
geo track --url https://your-domain.example --report --output ./geo-report.html

# ---- 测试（改动后必跑）----
docker compose -f docker-compose.prod.yml exec app php artisan test
docker compose -f docker-compose.prod.yml exec app ./vendor/bin/phpunit --filter=AiVisibility
```

---

## 五、关键设计决策补充

### 5.1 数据流向图

```
 ① 录入
   运营同事 ──▶ 后台
               ├─ 知识库（事实/资质/获奖/合作品牌，带来源 URL）
               ├─ 企业知识（能力点，带版本）
               ├─ 关键词库（目标问题，≤50 条/次，≤100 字符/条）
               └─ 公司档案 ★（结构化字段）
                    │
 ② 生产             ▼
   知识切片 ──▶ 向量化(pgvector) ──▶ 任务选题 ──▶ AI 写作
                                                     │
                                          ┌──────────┘
                                          ▼
                                    AI 质检门禁（四维检查）
                                          │
                            不通过 ───────┴───────▶ 停留草稿
                                          │ 通过
                                          ▼
                                     人工审核放行
                                          │
 ③ 分发                                   ▼
   托管站点 / 目标站点包 / WordPress REST / 通用 HTTP API
        │
        └─ 每个站点自动带 Schema + sitemap + llms.txt
           ★ 注入完整 Organization（含 sameAs/award/brand/hasCredential）
                                          │
 ④ 度量                                   ▼
   关键词库目标问题 ──▶ ★ 海外引擎采样（Perplexity + OpenAI）
                            │
                            ├─▶ ai_visibility_runs    （回答原文）
                            └─▶ ai_visibility_sources （信源与引用）
                                     │
                                     ▼
                            可见率 / 排名 / 情感 / 信源偏好 / 采样质量
                                     │
 ⑤ 反馈                              ▼
   可见率下降 ──▶ 人工定位相关文章 ──▶ 生成优化任务 ──▶ 回到 ②
```

**⑤ 为人工触发**（YAGNI）：自动阈值需要先积累数据才能确定取值，当前基线未知，任何阈值都是猜测。

### 5.2 安全模型

| 维度 | 措施 |
|------|------|
| **网络边界** | 内网访问，不对公众暴露任何运行实例 |
| **AGPL 边界** | 公开站点只发 `front_mode=static` 的**静态产物**（输出物，非衍生作品）；不部署含 GEOFlow front controller 的动态站点给公众 |
| **SSRF 防护** | `AiProviderEndpointPolicy` 白名单制。扩白名单**不得**放宽为通配或跳过校验；配套单元测试断言「新端点放行 + 非白名单仍拒绝」 |
| **密钥管理** | API Key 经后台加密存储（`AiSourceProvider.api_key` 在模型 `$hidden` 中）；**禁止**提交到代码库 |
| **成本防护** | `AiSourceProvider` 的 `daily_limit` / `used_today` / `total_used` 配额；防止采样任务失控产生费用 |
| **权限** | 后台入口按角色过滤，敏感操作需超级管理员 |
| **审计** | 操作日志、任务状态历史、人工发布回执与审计记录 |
| **出站管控** | 统一的出站请求策略（限制私网访问、重定向、响应体大小） |
| **更新安全** | Updater 使用签名包 + 本地 Unix socket，高风险操作需管理员密码 + 6 位验证器授权码 |
| **匿名统计** | 默认关闭；启用后只发固定白名单字段，业务内容/账号/域名/密钥不进载荷 |

### 5.3 性能与稳定性策略

| 策略 | 说明 |
|------|------|
| **队列按域拆分** | 5 个 worker 服务消费不同队列，长耗时任务（AI 质检）不阻塞短任务（分发） |
| **队列副本** | `ai-quality-queue` 默认 2 副本，质检是生产瓶颈 |
| **异步优先** | 内容生产、质检、分发、可见性采集全部走队列，Web 请求不阻塞 |
| **唯一任务** | `CollectAiVisibilityKeywordJob` 实现 `ShouldBeUnique`，防止同一问题重复采样烧钱 |
| **失败冷却** | 分发渠道级失败冷却，避免对故障渠道持续重试 |
| **向量召回回退** | 知识库语义召回失败时有稳定回退路径，不中断生产 |
| **静态产出** | 分发的站点以静态模式产出，无运行时开销，AI 爬虫抓取成本最低 |
| **采样须多次** | AI 回答有随机性，**单次采样不构成结论**；验收建立在多次采样的趋势上 |

---

## 附：本文档依据的代码事实

以下均在 `GEOFlow` commit `f7c75e6` 中实际核实：

| 事实 | 位置 |
|---|---|
| AI 可见性仅支持 3 个国内提供商 | `app/Models/AiVisibilityRun.php` L21-29 |
| 采样调度为单路径 if/else 链 | `AiVisibilityCollectionService::collect()` L20-67 |
| 出站域名白名单写死 | `AiProviderEndpointPolicy.php` L10-25 |
| 配置解析仅 2 个配置键，搜索源硬编码 | `AiVisibilityConfigurationResolver.php` L15-40 |
| Organization schema 只有 `name`（全仓库仅此一处） | `DistributionTargetSitePackageBuilder.php` 约 L3322 |
| 该文件 3796 行 | `wc -l` 实测 |
| 品牌判定为纯字符串匹配，默认值 GEOFlow / localhost | `AiVisibilityAnalyticsService.php` L940-975 |
| 品牌配置项与默认值 | `config/geoflow.php` L76-80 |
| 目标问题限 50 条 / 100 字符，**超长静默过滤** | `AiVisibilityAnalyticsController.php` L73-86 |
| 配额机制原生存在 | `AiSourceProvider.php` `$fillable` |
| 知识库含业务线与来源治理字段 | `KnowledgeBase.php` `$fillable` |
| 后台支持简体中文 | `lang/zh_CN/` |
| 12 个服务、默认 13 进程、5 个 worker 服务 | `docker-compose.prod.yml` |
| 323 个测试文件 | `tests/` 实测 |
| 8 个文件引用 `SAMPLE_PROVIDERS` | 全仓库 grep |
| 分发站点支持 `front_mode=static` | `DistributionTargetSitePackageBuilder` |