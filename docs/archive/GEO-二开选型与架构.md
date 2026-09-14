# GEO 二次开发选型与架构方案

> 修正说明：本文**取代** `GEO-选型建议.md` 的结论。
> 前文把贵司定位为"用 GEO 工具优化自己官网的工厂"，实际是**贵司自己开发这套系统、拿开源项目做二次开发**。
> 定位不同，选型标准从"哪个开箱即用"变成"**哪个能当底座 fork**"——许可证和架构形态成为首要因素。

---

## 一、结论

| 角色 | 选型 | 许可证 | 理由 |
|---|---|---|---|
| **底座（主系统）** | `GEORank` | **Apache-2.0** | monorepo 平台形态 + 管理后台 + 已含向量库/对象存储，且 README 明确支持二次开发 |
| **GEO 引擎（嵌入）** | `geo-optimizer-skill` | **MIT** | 有 `CheckRegistry` 插件扩展点，可直接作为 Python 库补强诊断能力 |
| **排除** | `GEOFlow` | ~~AGPL-3.0~~ | **AGPL 对二次开发是硬约束**，详见第四节 |

**一句话**：用 `GEORank` 当地基盖楼，把 `geo-optimizer-skill` 当砖嵌进诊断模块。**别碰 AGPL。**

---

## 二、为什么底座从 geo-optimizer-skill 换成 GEORank

前文推荐 `geo-optimizer-skill` 是因为它"开箱即用"。但你们要**自己造系统**，标准变了：

| 维度 | `geo-optimizer-skill` | `GEORank` |
|---|---|---|
| 形态 | Python 包 + CLI（**引擎/库**） | monorepo 平台（前台 + 管理台 + 后端 + SDK） |
| 前端 | 仅一个 demo 页面 | Next.js 前台 + Next.js 管理后台 |
| 多用户/权限 | 无 | 有（`packages/auth` 会话与页面守卫） |
| 数据库 | 无（本地文件存历史） | PostgreSQL + Redis + Qdrant + Neo4j + MinIO |
| 异步任务 | 无 | Celery |
| 扩展方式 | 插件（`CheckRegistry`） | 频道/工具/模型/诊断规则可扩展 |
| 适合 | 嵌入别人的系统 | **作为系统本身被二次开发** |

你们要交付的是图里那套**完整门户**（诊断 → 推荐 → 引流），需要后台、用户、任务队列、存储——
这些 `geo-optimizer-skill` 都没有，你们得从零搭。而 `GEORank` 已经有了。

---

## 三、关键技术发现：以图搜图的地基已经存在

这是本次调研最有价值的一点。`GEORank` 的 `docker-compose.yml` 里**真实启用**的服务：

| 服务 | 镜像 | 对你们的意义 |
|---|---|---|
| **Qdrant** | `qdrant/qdrant:v1.12.1` | **向量库** —— 以图搜图的核心，已在跑 |
| **MinIO** | `minio/minio:RELEASE.2025-04-22` | **对象存储** —— 存款式图/样品图，已在跑 |
| **Redis** | `redis:8.10.0-alpine` | 缓存 + Celery broker |
| **Neo4j** | `neo4j:5.26.29-community` | 知识图谱 |
| **PostgreSQL** | `postgres:16.9-alpine` | 主库 |
| Traefik / nginx | v3.7.10 / 1.31.3 | 网关与静态服务 |

代码侧确认：`backend/app/services/vector_store.py` 实际 `from qdrant_client import QdrantClient`，
`requirements.txt` 锁定 `qdrant-client==1.12.1`——**不是依赖列表里挂着好看，是真的在用**。

更关键的是 AI 层配置项（`backend/app/main.py`）：

```python
{"key": "embedding_api_key",  ...},
{"key": "embedding_base_url", ...},   # ← 可指向自部署服务
{"key": "embedding_model",    "value": "text-embedding-3-small"},
{"key": "embedding_dimensions", "value": 1536},
```

**Embedding Provider 兼容 OpenAI 格式且 base_url 可配**——意味着你们可以把它指向**自部署的
CLIP / SigLIP / Qwen-VL 视觉 embedding 服务**，而不是只能调 OpenAI。

> 于是图片匹配这块的二次开发工作量，从"搭一套向量检索系统"降到
> **"给现有 embedding 通道加一条视觉向量通路"**。这是量级上的差别。

附带确认：诊断规则权重可通过 `GET/PUT /diagnostics/rules` 后台 API 调整
（`diagnostic_rule_weights`），且前台有 Playwright worker 做浏览器渲染抓取
（`scripts/crawler_import_smoke.py` 校验 crawler 镜像）——对付 JS 渲染的独立站是够的。

---

## 四、为什么必须排除 GEOFlow（AGPL-3.0）

`GEOFlow` 是这批项目里星最多的（3.6k★），功能也最完整，但它的许可证是 **AGPL-3.0**。

AGPL 与常见的 GPL 不同，它有个**网络服务条款（第 13 条）**：

> 即使用户只是通过网络与程序交互（不下载二进制），你也必须向他们提供完整源码，
> **包括你二次开发修改的部分**。

对贵司的具体影响：

- ❌ 想把这套系统做成**对外服务/产品**卖给别的工厂 → **必须开源你们的全部修改**，商业模式直接不成立
- ⚠️ 即使是**公司内部自用**，员工通过浏览器访问也算"网络交互"，理论上也要向员工提供源码
- ⚠️ AGPL 具有**传染性**，一旦代码混入你们的私有系统，法务风险会沿着调用链扩散

**这不是"注意一下"级别的问题，是选型红线。** 相比之下：

- `GEORank` = **Apache-2.0** → 允许闭源二次开发、允许商用、只需保留版权声明
- `geo-optimizer-skill` = **MIT** → 几乎无约束

除非贵司法务明确签字认可 AGPL，否则不要把它引入产品代码。

---

## 五、落地架构：图里三件事怎么拆

```
┌─────────────────────────────────────────────────────────────┐
│  底座：GEORank (Apache-2.0)                                  │
│  Next.js 前台/管理台 + FastAPI + Celery                       │
│  PostgreSQL · Redis · Qdrant · MinIO · Neo4j                 │
└─────────────────────────────────────────────────────────────┘
        │                    │                     │
        ▼                    ▼                     ▼
┌───────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ ① GEO 引擎     │  │ ② 以图搜图        │  │ ③ 企微引流        │
│               │  │                  │  │                  │
│ GEORank 诊断   │  │ Qdrant ✅ 已有    │  │ 完全自研          │
│ +             │  │ MinIO  ✅ 已有    │  │                  │
│ geo-optimizer │  │ Celery ✅ 已有    │  │ 企业微信 API：    │
│ -skill (MIT)  │  │                  │  │ · 客户联系        │
│ 插件补强       │  │ 新增：视觉        │  │ · 群活码          │
│               │  │ embedding 通路    │  │ · 渠道活码        │
│ 工作量：小     │  │ 工作量：中        │  │ 工作量：中        │
└───────────────┘  └──────────────────┘  └──────────────────┘
```

### ① GEO 引擎（工作量小）
`GEORank` 自带诊断模块（检查 Schema、页面结构、Meta、内容可读性、引用信号）。
用 `geo-optimizer-skill` 补强：它的 100 分模型更细，且 `CheckRegistry` 是公开扩展点：

```python
# geo-optimizer-skill 的分值分布（合计 100）
Robots.txt 18 · llms.txt 18 · Schema JSON-LD 16 · Meta 14
Content 12 · Signals 6 · AI Discovery 6 · Brand & Entity 10
```

**接入方式**：把 `geo-optimizer-skill` 作为 Python 依赖装进 `GEORank` 后端，
通过 `entry_points("geo_optimizer.checks")` 注册你们自己的检查项（比如"设计资质结构化"），
得分并入 GEORank 的诊断报告。

**注意**：`geo-optimizer-skill` 的 docstring 是**意大利语**（意大利团队项目），
`i18n` 模块有翻译。二次开发时注意这点。

### ② 以图搜图（工作量中）—— 地基已有，接视觉模型即可
需要新增的部分：
1. 款式库图片批量**向量化入库**（Celery 异步任务，MinIO 取图 → 视觉模型 → Qdrant 写入）
2. 买手上传图片 → 实时向量化 → Qdrant 相似度检索 → 返回匹配款式与打样方案
3. 在 GEORank 的 AI 层增加一条**视觉 embedding provider** 通道（现有配置是文本的）

> ⚠️ **强烈建议自部署视觉模型，不要用 Gemini API。**
> 你们是原创设计工厂，款式库是核心资产。把内部款式库喂给第三方模型，
> 等于把新款设计图交出去，可能进请求日志、可能被用于训练。
> 自部署 CLIP / SigLIP 做向量检索、或 Qwen-VL 做多模态，成本不高且数据不出内网。
> 而且既然 Qdrant 已经在跑，自部署方案和现有架构更贴合。

### ③ 企微引流（工作量中）
与 GEO 无关，纯自研。用企业微信「客户联系」+「群活码 / 渠道活码」API，
在独立站放带渠道参数的二维码，扫码后自动打标签并进对应客户群。
这部分**没有开源 GEO 项目能帮上忙**，别指望在 GEO 仓库里找。

---

## 六、二次开发前必须做的三件事

### 1. 剥离 GEORank 的 demo 数据（法务问题）
`GEORank` 的 **代码**是 Apache-2.0，但 `DATA_LICENSE.md` 明确写了：

> 仓库还包含公开专家资料和内置首页。这些材料可能包含**姓名、传记、肖像、书籍封面、logo、产品名**，
> 适用额外的权利边界。

即**代码可以随便用，但内置的专家频道/公司目录/首页内容不能商用**。
上线前必须清空 `data/public`、`runtime/homepages` 和专家频道相关数据，换成你们自己的。

### 2. 裁剪不需要的模块
`GEORank` 是个**通用** GEO 工作台，含「公司目录 / 专家频道 / 教程频道」——
这些是内容站功能，对"出口工厂获客"场景是冗余的。它有模块开关（后台「模块开关」设置），
先关掉，再决定是否删代码，避免后续 merge 上游时冲突。

### 3. 定好上游同步策略
你们要长期跟进 `GEORank` 上游更新（它 2 个月内 19 次提交，还在快速迭代）。
建议 **fork 后开自己的分支**，把定制尽量收敛到新文件/新模块，少改上游原有文件，
否则每次合并上游都是一场灾难。

---

## 七、待你确认的一个问题

**这套系统是只给贵司自己获客用，还是要做成产品卖给其他工厂/客户？**

- 若**只自用** → `GEORank` (Apache-2.0) 完全够，架构照上面走
- 若**要对外卖** → 更要在许可证上把死门，`GEORank` 仍是安全选择（Apache-2.0 允许商用闭源），
  但**必须确认上一条的 demo 数据已剥离**，否则等于拿别人的肖像和品牌去卖钱

两种情况下 `GEORank` 都是对的，但**对外卖时的数据剥离是硬性前提**。
