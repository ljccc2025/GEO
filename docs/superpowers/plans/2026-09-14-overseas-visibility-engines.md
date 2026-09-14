# 海外 AI 引擎接入 实现计划（子项目 1 · M0 + M1）

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 让 GEO 作战平台能向海外 AI 引擎（Perplexity、OpenAI）采集「目标问题上 AI 如何回答」，从而建立可度量的 AI 可见性基线。

**架构：** 在 GEOFlow 的 `AiVisibility` 子系统内，按既有「每个提供商一个 Client」的模式新增两个检索型采样引擎，并把它们接入采集调度、配置解析与出站端点白名单。可见性判定沿用既有的品牌别名字符串匹配，因此必须先配置品牌身份，否则基线测的是开源项目名。

**技术栈：** PHP 8.3+ / Laravel / PHPUnit / PostgreSQL+pgvector / Redis / Docker Compose；Perplexity API、OpenAI Responses API（`web_search` 工具）

**代码仓库：** `https://github.com/ljccc2025/GEOFlow`（fork 自 `yaojingang/GEOFlow`，基线提交 `f7c75e6`）
**分支：** `feat/overseas-visibility-engines`
**配套规格：** `docs/superpowers/specs/2026-09-14-geo-platform-design.md`

---

## 范围

**本计划覆盖：**

- **M0 前置**：部署实例、验证海外 API 连通、配置品牌身份
- **M1 主体**：接入 Perplexity 与 OpenAI 两个检索型采样引擎

**本计划不覆盖（另立计划）：**

| 不做的 | 原因 |
|---|---|
| M2 公司档案 + Organization Schema | 与 M1 在数据流上互不重叠，独立计划 |
| M3 品牌化 + 业务模板 | 是后台配置与内容录入，非编码工作 |
| 自动阈值告警 | YAGNI，基线未知时无法定阈值 |

---

## 文件结构

所有路径相对 `GEOFlow/` 仓库根目录。

### 新建文件

| 文件 | 职责 |
|---|---|
| `app/Services/GeoFlow/AiVisibility/PerplexitySearchClient.php` | 调用 Perplexity，取回带网页引用的回答，归一化为 `AiVisibilityResult` |
| `app/Services/GeoFlow/AiVisibility/OpenAiWebSearchClient.php` | 调用 OpenAI Responses API + `web_search`，取回带引用的回答，归一化 |
| `tests/Unit/PerplexitySearchClientTest.php` | Perplexity Client 单元测试（mock HTTP，不打真实 API） |
| `tests/Unit/OpenAiWebSearchClientTest.php` | OpenAI Client 单元测试（mock HTTP） |
| `tests/Unit/AiProviderEndpointPolicyOverseasTest.php` | 端点白名单策略测试：新端点放行 + 非白名单仍拒绝 |

### 修改文件

| 文件 | 改动 |
|---|---|
| `app/Models/AiVisibilityRun.php` | 新增 2 个 `PROVIDER_*` 常量，挂进 `SAMPLE_PROVIDERS` |
| `app/Services/GeoFlow/AiVisibility/AiProviderEndpointPolicy.php` | 白名单新增 `api.perplexity.ai`、`api.openai.com`（**安全敏感**） |
| `app/Services/GeoFlow/AiVisibility/AiVisibilityResultNormalizer.php` | 新增 `normalizePerplexity`、`normalizeOpenAiWebSearch` 两个方法 |
| `app/Services/GeoFlow/AiVisibility/AiVisibilityService.php` | 新增 `runPerplexitySearch`、`runOpenAiWebSearch` 两个方法，并注入两个新 Client |
| `app/Services/GeoFlow/AiVisibility/AiVisibilityConfigurationResolver.php` | 新增 2 个配置键与解析方法，改造 `searchProvider()` 不再硬编码单一 provider_key |
| `app/Services/GeoFlow/AiVisibility/AiVisibilityCollectionService.php` | 新增分发分支，使新引擎可被选中；同一目标问题对两个引擎各采一次 |
| `app/Models/AiSourceProvider.php` | 新增 2 个 `PROVIDER_*` 常量 |
| `tests/Unit/AiVisibilityResultNormalizerTest.php` | 补两个新归一化方法的测试 |
| `tests/Feature/AdminAiVisibilityAnalyticsTest.php` | 补新引擎在分析管道中生效的测试 |

### 文件边界说明

- **一个 Client 一个文件**：沿用既有 `DoubaoSearchCustomClient` 的模式，每个外部引擎独立成文件，便于单独测试与替换。
- **不新建 Normalizer 文件**：既有 `AiVisibilityResultNormalizer` 已是归一化职责的唯一归属，新引擎的归一化方法追加进去，避免出现两个归一化入口。
- **不改 `AiVisibilityAnalyticsService` 等 8 个分析类文件**：它们通过 `SAMPLE_PROVIDERS` 常量取引擎清单，挂常量后自动生效。

---

## M0 前置：部署与品牌身份

> M0 是**部署与配置**，不是编码，因此不采用 TDD 步骤，而是「执行 → 验证」的清单。
> **M0 未完成前，M1 的所有验收都无法成立。**

### M0-1：验证海外 API 连通性（假设 A1）

- [ ] 在公司服务器上执行：

```bash
curl -s -o /dev/null -w 'perplexity: %{http_code}\n' https://api.perplexity.ai
curl -s -o /dev/null -w 'openai: %{http_code}\n' https://api.openai.com/v1/models
```

- [ ] **判定**：返回 `401` / `404@@ 都算**通过**（能连上，只是没带密钥）；返回 `000@@ 或超时算**不通过**。
- [ ] **不通过时**：M1 阻塞。需要先解决服务器出网条件，本计划后续任务全部等待。

### M0-2：部署 GEOFlow 实例

- [ ] 在服务器上准备 `.env.prod`（完整变量见技术方案文档 §4.2），然后：

```bash
git clone https://github.com/ljccc2025/GEOFlow.git
cd GEOFlow && git checkout f7c75e6
docker compose --env-file .env.prod -f docker-compose.prod.yml build
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d postgres redis
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d init
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d --remove-orphans \
  app web queue ai-quality-queue ai-quality-backfill-queue \
  ai-optimization-queue knowledge-queue scheduler reverb
```

- [ ] **验证**：`docker compose ps` 显示 **12 个服务 Up**，其中 `ai-quality-queue` 有 **2 个副本**。
- [ ] **验证**：浏览器能打开后台登录页，界面为简体中文。

### M0-3：配置品牌身份（★ 最关键，错了则全部验收失效）

- [ ] 编辑 `.env.prod`，设置这四项：

```bash
SITE_NAME=<公司英文名，海外买手最可能使用的写法>
SITE_FULL_NAME=<公司英文全称>
APP_NAME=<公司英文名>
SITE_URL=<域名，暂无则留空并记录为待办>
```

- [ ] 重启应用使其生效：`docker compose --env-file .env.prod -f docker-compose.prod.yml up -d app web queue`

### M0-4：验证品牌判定生效

- [ ] 在后台关键词库中添加**一条已知会提及本公司的问句**（例如直接用公司英文名提问）。
- [ ] 触发采集：

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml exec app \
  php artisan geoflow:ai-visibility:collect "<公司英文名> supplier"
```

- [ ] 打开后台 `/admin/ai-visibility`，**确认看板能识别出本品牌**。
- [ ] **判定**：若识别不出，说明 `brandAliases()` 的别名集合没覆盖到，回到 M0-3 调整 `SITE_NAME`。**此项不通过则不得进入 M1 验收。**

---

## 任务组 A：基础设施（策略 · 常量 · 配置解析）

---

### 任务 A1：扩展出站端点白名单

> ⚠ **安全敏感**。这是防 SSRF 的控制。项目规则明示「绝不允许绕过」。

**文件：**
- 修改：`app/Services/GeoFlow/AiVisibility/AiProviderEndpointPolicy.php:10-25`
- 测试：`tests/Unit/AiProviderEndpointPolicyOverseasTest.php`（新建）

- [ ] **步骤 1：编写失败的测试**

```php
<?php

namespace Tests\Unit;

use App\Services\GeoFlow\AiVisibility\AiProviderEndpointPolicy;
use PHPUnit\Framework\TestCase;

class AiProviderEndpointPolicyOverseasTest extends TestCase
{
    public function test_it_accepts_perplexity_search_endpoint(): void
    {
        $policy = new AiProviderEndpointPolicy;

        $this->assertTrue($policy->acceptsSearchApi('https://api.perplexity.ai/chat/completions'));
    }

    public function test_it_accepts_openai_search_endpoint(): void
    {
        $policy = new AiProviderEndpointPolicy;

        $this->assertTrue($policy->acceptsSearchApi('https://api.openai.com/v1/responses'));
    }

    public function test_it_still_rejects_unlisted_host(): void
    {
        $policy = new AiProviderEndpointPolicy;

        $this->assertFalse($policy->acceptsSearchApi('https://evil.example.com/v1/responses'));
    }

    public function test_it_rejects_plain_http_scheme(): void
    {
        $policy = new AiProviderEndpointPolicy;

        $this->assertFalse($policy->acceptsSearchApi('http://api.perplexity.ai/chat/completions'));
    }

    public function test_it_rejects_host_suffix_lookalike(): void
    {
        $policy = new AiProviderEndpointPolicy;

        $this->assertFalse($policy->acceptsSearchApi('https://api.openai.com.attacker.example/v1/responses'));
    }

    public function test_it_rejects_url_with_credentials(): void
    {
        $policy = new AiProviderEndpointPolicy;

        $this->assertFalse($policy->acceptsSearchApi('https://user:pass@api.openai.com/v1/responses'));
    }
}
```

- [ ] **步骤 2：运行测试验证失败**

运行：`php artisan test --filter=AiProviderEndpointPolicyOverseasTest`
预期：FAIL —— 前两个用例失败（`api.perplexity.ai` / `api.openai.com` 不在白名单）；后四个用例应当**已经通过**（既有防护先于本次改动生效）。

- [ ] **步骤 3：编写最少实现代码**

修改 `app/Services/GeoFlow/AiVisibility/AiProviderEndpointPolicy.php`，在 `isTrustedHttpsUrl` 之外新增一个只服务于「搜索型」端点的常量和分支：

```php
    /**
     * @var array<string, list<string>>
     */
    private const MODEL_HOSTS = [
        'ark' => ['volces.com'],
        'deepseek' => ['deepseek.com'],
        // 海外引擎：复用 ark 绑定类型，端点由 provider 记录中的 endpoint_url 指定
        'openai' => ['openai.com'],
    ];

    /**
     * 检索型信源允许的宿主。注意：新增项必须保持精确匹配语义，禁止通配。
     *
     * @var list<string>
     */
    private const SEARCH_HOSTS = [
        'feedcoopapi.com',   // 豆包搜索（既有）
        'perplexity.ai',     // Perplexity（新增）
        'openai.com',        // OpenAI Responses API（新增）
    ];

    public function acceptsSearchApi(string $url): bool
    {
        return $this->isTrustedHttpsUrl($url, self::SEARCH_HOSTS);
    }
```

> `isTrustedHttpsUrl` **一行都不改**——它已经做了 HTTPS 强制、拒绝 URL 内嵌凭据、拒绝主机名后缀伪装（`str_ends_with($host, '.'.$trustedHost)` 语义正确）。本次只扩充宿主列表。

- [ ] **步骤 4：运行测试验证通过**

运行：`php artisan test --filter=AiProviderEndpointPolicyOverseasTest`
预期：PASS，6 个用例全部通过。

- [ ] **步骤 5：回归验证既有防护未被削弱**

运行：`php artisan test --filter=AiVisibility`
预期：PASS。既有可见度相关测试无回归。

- [ ] **步骤 6：Commit**

```bash
git add app/Services/GeoFlow/AiVisibility/AiProviderEndpointPolicy.php \
        tests/Unit/AiProviderEndpointPolicyOverseasTest.php
git commit -m "feat(visibility): 端点白名单放行 Perplexity 与 OpenAI"
```

---

### 任务 A2：新增 provider 常量并挂进采样清单

**文件：**
- 修改：`app/Models/AiVisibilityRun.php:21-29`
- 修改：`app/Models/AiSourceProvider.php:10`
- 测试：`tests/Unit/AiVisibilitySampleProvidersTest.php`（新建）

- [ ] **步骤 1：编写失败的测试**

```php
<?php

namespace Tests\Unit;

use App\Models\AiVisibilityRun;
use PHPUnit\Framework\TestCase;

class AiVisibilitySampleProvidersTest extends TestCase
{
    public function test_it_includes_overseas_engines_in_sample_providers(): void
    {
        $this->assertContains(AiVisibilityRun::PROVIDER_PERPLEXITY_SEARCH, AiVisibilityRun::SAMPLE_PROVIDERS);
        $this->assertContains(AiVisibilityRun::PROVIDER_OPENAI_WEB_SEARCH, AiVisibilityRun::SAMPLE_PROVIDERS);
    }

    public function test_it_keeps_existing_domestic_providers(): void
    {
        $this->assertContains(AiVisibilityRun::PROVIDER_DOUBAO_ARK_RESPONSES, AiVisibilityRun::SAMPLE_PROVIDERS);
        $this->assertContains(AiVisibilityRun::PROVIDER_DOUBAO_SEARCH_CUSTOM, AiVisibilityRun::SAMPLE_PROVIDERS);
        $this->assertContains(AiVisibilityRun::PROVIDER_DEEPSEEK_ANALYSIS, AiVisibilityRun::SAMPLE_PROVIDERS);
    }

    public function test_provider_constants_have_expected_values(): void
    {
        $this->assertSame('perplexity_search', AiVisibilityRun::PROVIDER_PERPLEXITY_SEARCH);
        $this->assertSame('openai_web_search', AiVisibilityRun::PROVIDER_OPENAI_WEB_SEARCH);
    }
}
```

- [ ] **步骤 2：运行测试验证失败**

运行：`php artisan test --filter=AiVisibilitySampleProvidersTest`
预期：FAIL，报错 `Undefined constant App\Models\AiVisibilityRun::PROVIDER_PERPLEXITY_SEARCH`

- [ ] **步骤 3：编写最少实现代码**

在 `app/Models/AiVisibilityRun.php` 的既有常量块之后新增：

```php
    public const PROVIDER_PERPLEXITY_SEARCH = 'perplexity_search';

    public const PROVIDER_OPENAI_WEB_SEARCH = 'openai_web_search';
```

并把 `SAMPLE_PROVIDERS` 改为：

```php
    public const SAMPLE_PROVIDERS = [
        self::PROVIDER_DEEPSEEK_ANALYSIS,
        self::PROVIDER_DOUBAO_ARK_RESPONSES,
        self::PROVIDER_DOUBAO_SEARCH_CUSTOM,
        self::PROVIDER_PERPLEXITY_SEARCH,
        self::PROVIDER_OPENAI_WEB_SEARCH,
    ];
```

在 `app/Models/AiSourceProvider.php` 新增（信源供应商侧标识）：

```php
    public const PROVIDER_PERPLEXITY_SEARCH = 'perplexity_search';

    public const PROVIDER_OPENAI_WEB_SEARCH = 'openai_web_search';
```

- [ ] **步骤 4：运行测试验证通过**

运行：`php artisan test --filter=AiVisibilitySampleProvidersTest`
预期：PASS，3 个用例通过。

- [ ] **步骤 5：确认 8 个分析类文件无需改动**

```bash
grep -rln 'SAMPLE_PROVIDERS' app/ | sort
```
预期：输出 8 个文件。**逐个确认它们都通过常量引用而非硬编码引擎名**，因此自动受益于本次变更。

- [ ] **步骤 6：Commit**

```bash
git add app/Models/AiVisibilityRun.php app/Models/AiSourceProvider.php \
        tests/Unit/AiVisibilitySampleProvidersTest.php
git commit -m "feat(visibility): 新增 Perplexity 与 OpenAI 采样提供商常量"
```

---
### 任务 A3：扩展配置解析，使新引擎可被选中

**文件：**
- 修改：`app/Services/GeoFlow/AiVisibility/AiVisibilityConfigurationResolver.php:15-40`

- [ ] **步骤 1：编写失败的测试**

```php
<?php

namespace Tests\Unit;

use App\Services\GeoFlow\AiVisibility\AiVisibilityConfigurationResolver;
use ReflectionClass;
use Tests\TestCase;

class AiVisibilityConfigurationKeysTest extends TestCase
{
    public function test_it_exposes_config_keys_for_overseas_engines(): void
    {
        $this->assertSame('ai_visibility_perplexity_provider_id', AiVisibilityConfigurationResolver::PERPLEXITY_PROVIDER_SETTING_KEY);
        $this->assertSame('ai_visibility_openai_provider_id', AiVisibilityConfigurationResolver::OPENAI_PROVIDER_SETTING_KEY);
    }

    public function test_search_provider_is_not_hardcoded_to_doubao(): void
    {
        $source = file_get_contents((new ReflectionClass(AiVisibilityConfigurationResolver::class))->getFileName());

        $this->assertStringNotContainsString(
            "->where('provider_key', AiSourceProvider::PROVIDER_DOUBAO_SEARCH_CUSTOM)",
            $source,
            '按引擎解析信源时不得把 provider_key 硬编码为豆包，否则新引擎永远选不中。'
        );
    }
}
```

- [ ] **步骤 2：运行测试验证失败**

运行：`php artisan test --filter=AiVisibilityConfigurationKeysTest`
预期：FAIL —— `PERPLEXITY_PROVIDER_SETTING_KEY` 未定义；第二个用例因源码仍含硬编码而失败。

- [ ] **步骤 3：编写最少实现代码**

在 `AiVisibilityConfigurationResolver` 中新增两个配置键常量：

```php
    public const PERPLEXITY_PROVIDER_SETTING_KEY = 'ai_visibility_perplexity_provider_id';

    public const OPENAI_PROVIDER_SETTING_KEY = 'ai_visibility_openai_provider_id';
```

把 `searchProvider()` 改造为按 `provider_key` 参数解析，并保留一个向后兼容的无参入口：

```php
    /**
     * @param  string|null  $providerKey  为 null 时沿用既有的豆包搜索信源（向后兼容）
     */
    public function searchProvider(SystemAiIdentity $identity, ?string $providerKey = null): ?AiSourceProvider
    {
        $identity->assertCanResolveVisibilityConfiguration();
        if (! Schema::hasTable('ai_source_providers')) {
            return null;
        }

        $providerKey ??= AiSourceProvider::PROVIDER_DOUBAO_SEARCH_CUSTOM;

        return AiSourceProvider::query()
            ->where('provider_key', $providerKey)
            ->where('status', 'active')
            ->orderBy('id')
            ->get()
            ->first(fn (AiSourceProvider $provider): bool => $this->endpointPolicy
                ->acceptsSearchApi((string) ($provider->endpoint_url ?? ''))
                && $this->hasStoredApiKey($provider));
    }

    public function perplexityProvider(SystemAiIdentity $identity): ?AiSourceProvider
    {
        return $this->searchProvider($identity, AiSourceProvider::PROVIDER_PERPLEXITY_SEARCH);
    }

    public function openAiProvider(SystemAiIdentity $identity): ?AiSourceProvider
    {
        return $this->searchProvider($identity, AiSourceProvider::PROVIDER_OPENAI_WEB_SEARCH);
    }
```

> **向后兼容**：`$providerKey` 默认 `null` 时行为与改造前完全一致，既有调用方（`AiVisibilityCollectionService`）不需要立即改动。

- [ ] **步骤 4：运行测试验证通过**

运行：`php artisan test --filter=AiVisibilityConfigurationKeysTest`
预期：PASS，2 个用例通过。

- [ ] **步骤 5：回归验证**

运行：`php artisan test --filter=AiVisibility`
预期：PASS。

- [ ] **步骤 6：Commit**

```bash
git add app/Services/GeoFlow/AiVisibility/AiVisibilityConfigurationResolver.php \
        tests/Unit/AiVisibilityConfigurationKeysTest.php
git commit -m "feat(visibility): 配置解析支持按引擎选择信源"
```

---

## 任务组 B：Perplexity 引擎端到端

### 任务 B1：归一化器支持 Perplexity 响应（并抽取共享解析）

> `normalizeArkResponses` 已在解析 OpenAI Responses 结构（`output[].content[].annotations[].url_citation`），其形状与 OpenAI 官方一致。
> 因此**抽取共享解析器**，避免 OpenAI 侧重复实现（DRY）。

**文件：**
- 修改：`app/Services/GeoFlow/AiVisibility/AiVisibilityResultNormalizer.php:15-102`
- 测试：`tests/Unit/AiVisibilityResultNormalizerTest.php`（追加用例）

- [ ] **步骤 1：编写失败的测试**

追加到 `tests/Unit/AiVisibilityResultNormalizerTest.php`：

```php
    public function test_it_normalizes_perplexity_chat_completions_with_citations(): void
    {
        $result = (new AiVisibilityResultNormalizer)->normalizePerplexity([
            'id' => 'pplx-123',
            'model' => 'sonar',
            'choices' => [[
                'index' => 0,
                'message' => ['role' => 'assistant', 'content' => '我们推荐 A 公司。'],
                'finish_reason' => 'stop',
            ]],
            'citations' => ['https://example.com/a'],
            'search_results' => [[
                'title' => 'A 公司官网',
                'url' => 'https://example.com/a',
                'date' => '2026-01-01',
            ]],
            'usage' => ['total_tokens' => 42],
        ], ['model' => 'sonar'], 123);

        $this->assertSame(AiVisibilityRun::PROVIDER_PERPLEXITY_SEARCH, $result->providerType);
        $this->assertSame('我们推荐 A 公司。', $result->answerText);
        $this->assertCount(1, $result->sources);
        $this->assertSame('https://example.com/a', $result->sources[0]->url);
        $this->assertSame(123, $result->latencyMs);
    }

    public function test_it_deduplicates_perplexity_citations_and_search_results(): void
    {
        $result = (new AiVisibilityResultNormalizer)->normalizePerplexity([
            'choices' => [['message' => ['content' => '答案']]],
            'citations' => ['https://example.com/a'],
            'search_results' => [['title' => 'A', 'url' => 'https://example.com/a']],
        ], [], 10);

        $this->assertCount(1, $result->sources, '同一 URL 出现在 citations 与 search_results 时只应保留一条信源。');
    }

    public function test_it_returns_empty_answer_when_perplexity_payload_has_no_content(): void
    {
        $result = (new AiVisibilityResultNormalizer)->normalizePerplexity([], [], 5);

        $this->assertSame('', $result->answerText);
        $this->assertSame([], $result->sources);
    }
```

> **注意**：`AiVisibilitySourceData` 的属性名需以实际类定义为准。实现前先执行
> `sed -n '1,60p' app/Services/GeoFlow/AiVisibility/AiVisibilitySourceData.php` 核对字段名，若为 `link` 而非 `url`，同步调整上述断言。

- [ ] **步骤 2：运行测试验证失败**

运行：`php artisan test --filter=AiVisibilityResultNormalizerTest`
预期：FAIL，报错 `Call to undefined method ...::normalizePerplexity()`

- [ ] **步骤 3：编写最少实现代码**

**3a. 抽取共享解析器**——在 `AiVisibilityResultNormalizer` 中新增：

```php
    /**
     * 解析 OpenAI Responses 形状的载荷（Ark 与 OpenAI 官方同构）。
     *
     * @param  array<string,mixed>  $response
     * @return array{segments: list<string>, sources: list<AiVisibilitySourceData>, web_search_calls: list<array<string,mixed>>, response_id: string}
     */
    private function parseResponsesPayload(array $response): array
    {
        // 把原 normalizeArkResponses 第 17-84 行的解析逻辑整体搬到这里，
        // 只返回中间结果，不构造 AiVisibilityResult。
    }
```

然后让 `normalizeArkResponses` 改为调用它，**行为与输出完全不变**（既有测试必须继续通过）：

```php
    public function normalizeArkResponses(array $response, array $request, string $modelId, int $latencyMs): AiVisibilityResult
    {
        $parsed = $this->parseResponsesPayload($response);

        return new AiVisibilityResult(
            providerType: AiVisibilityRun::PROVIDER_DOUBAO_ARK_RESPONSES,
            providerKey: 'doubao_ark',
            modelId: $modelId,
            answerText: $parsed['segments'] === []
                ? $this->stringValue($response['output_text'] ?? '')
                : trim(implode("\n\n", $parsed['segments'])),
            sources: $parsed['sources'],
            usage: is_array($response['usage'] ?? null) ? $response['usage'] : [],
            metadata: array_filter([
                'response_id' => $parsed['response_id'],
                'web_search_calls' => $parsed['web_search_calls'],
                'tool_usage' => is_array($response['usage']['tool_usage'] ?? null) ? $response['usage']['tool_usage'] : null,
            ], static fn (mixed $value): bool => $value !== null && $value !== '' && $value !== []),
            rawRequest: $request,
            rawResponse: $response,
            latencyMs: $latencyMs,
        );
    }
```

**3b. 新增 Perplexity 归一化**：

```php
    /**
     * @param  array<string,mixed>  $response
     * @param  array<string,mixed>  $request
     */
    public function normalizePerplexity(array $response, array $request, int $latencyMs): AiVisibilityResult
    {
        $segments = [];
        $choices = $response['choices'] ?? [];
        if (is_array($choices)) {
            foreach ($choices as $choice) {
                if (! is_array($choice)) {
                    continue;
                }
                $content = $this->stringValue($choice['message']['content'] ?? '');
                if ($content !== '') {
                    $segments[] = $content;
                }
            }
        }

        $sources = [];
        $searchResults = $response['search_results'] ?? [];
        if (is_array($searchResults)) {
            foreach ($searchResults as $item) {
                if (! is_array($item)) {
                    continue;
                }
                $url = $this->stringValue($item['url'] ?? '');
                if ($url === '') {
                    continue;
                }
                $sources = $this->appendUniqueSource($sources, new AiVisibilitySourceData(
                    url: $url,
                    title: $this->stringValue($item['title'] ?? ''),
                    publishedAt: $this->stringValue($item['date'] ?? ''),
                    position: count($sources) + 1,
                ));
            }
        }

        $citations = $response['citations'] ?? [];
        if (is_array($citations)) {
            foreach ($citations as $citation) {
                $url = $this->stringValue(is_string($citation) ? $citation : ($citation['url'] ?? ''));
                if ($url === '') {
                    continue;
                }
                $sources = $this->appendUniqueSource($sources, new AiVisibilitySourceData(
                    url: $url,
                    title: '',
                    publishedAt: '',
                    position: count($sources) + 1,
                ));
            }
        }

        return new AiVisibilityResult(
            providerType: AiVisibilityRun::PROVIDER_PERPLEXITY_SEARCH,
            providerKey: AiSourceProvider::PROVIDER_PERPLEXITY_SEARCH,
            modelId: $this->stringValue($response['model'] ?? ''),
            answerText: trim(implode("\n\n", array_filter($segments, static fn (string $s): bool => trim($s) !== ''))),
            sources: $sources,
            usage: is_array($response['usage'] ?? null) ? $response['usage'] : [],
            metadata: array_filter([
                'response_id' => $this->stringValue($response['id'] ?? ''),
            ], static fn (mixed $value): bool => $value !== null && $value !== ''),
            rawRequest: $request,
            rawResponse: $response,
            latencyMs: $latencyMs,
        );
    }
```

> `AiVisibilitySourceData` 的构造参数名以实际定义为准；实现前用 `sed -n '1,40p' app/Services/GeoFlow/AiVisibility/AiVisibilitySourceData.php` 核对。

- [ ] **步骤 4：运行测试验证通过**

运行：`php artisan test --filter=AiVisibilityResultNormalizerTest`
预期：PASS —— 新增 3 个用例通过，**且既有的 ark / doubao 用例全部继续通过**（证明抽取未改变行为）。

- [ ] **步骤 5：Commit**

```bash
git add app/Services/GeoFlow/AiVisibility/AiVisibilityResultNormalizer.php \
        tests/Unit/AiVisibilityResultNormalizerTest.php
git commit -m "feat(visibility): 归一化器支持 Perplexity，抽取 Responses 共享解析"
```

---
### 任务 B2：PerplexitySearchClient

**文件：**
- 创建：`app/Services/GeoFlow/AiVisibility/PerplexitySearchClient.php`
- 测试：`tests/Unit/PerplexitySearchClientTest.php`（新建）

> **实现前必做**：先核对 Perplexity 的实时 API 契约。用 `mcp__context7__query-docs` 查 Perplexity API 文档，
> 或对沙箱密钥发一次真实请求，确认 `chat/completions` 的请求体字段、`citations` 与 `search_results` 的实际存在性。
> **不要把未经验证的字段形状写死进实现**——若契约有出入，先调整本任务的 payload 构造与归一化方法，再继续。

- [ ] **步骤 1：编写失败的测试**

```php
<?php

namespace Tests\Unit;

use App\Models\AiSourceProvider;
use App\Services\GeoFlow\AiVisibility\AiVisibilityHttpClientFactory;
use App\Services\GeoFlow\AiVisibility\AiVisibilityResultNormalizer;
use App\Services\GeoFlow\AiVisibility\PerplexitySearchClient;
use App\Support\GeoFlow\ApiKeyCrypto;
use Illuminate\Http\Client\Response;
use Mockery;
use PHPUnit\Framework\TestCase;
use RuntimeException;

class PerplexitySearchClientTest extends TestCase
{
    protected function tearDown(): void
    {
        Mockery::close();
        parent::tearDown();
    }

    public function test_it_posts_the_query_and_normalizes_the_answer(): void
    {
        $provider = $this->provider();
        $response = new Response(new \GuzzleHttp\Psr7\Response(200, [], json_encode([
            'id' => 'pplx-1',
            'model' => 'sonar',
            'choices' => [['message' => ['content' => '推荐 A 公司。']]],
            'citations' => ['https://example.com/a'],
        ])));

        $request = Mockery::mock();
        $request->shouldReceive('post')->once()->with('https://api.perplexity.ai/chat/completions', Mockery::on(
            fn (array $payload): bool => ($payload['model'] ?? null) === 'sonar'
                && ($payload['messages'][0]['content'] ?? null) === '中国耳塞设计工厂'
        ))->andReturn($response);

        $factory = Mockery::mock(AiVisibilityHttpClientFactory::class);
        $factory->shouldReceive('jsonRequest')->once()->with('secret-key')->andReturn($request);

        $crypto = Mockery::mock(ApiKeyCrypto::class);
        $crypto->shouldReceive('decrypt')->once()->andReturn('secret-key');

        $client = new PerplexitySearchClient($crypto, $factory, new AiVisibilityResultNormalizer);
        $result = $client->search($provider, '中国耳塞设计工厂');

        $this->assertSame('推荐 A 公司。', $result->answerText);
        $this->assertCount(1, $result->sources);
    }

    public function test_it_throws_on_empty_query(): void
    {
        $client = new PerplexitySearchClient(
            Mockery::mock(ApiKeyCrypto::class),
            Mockery::mock(AiVisibilityHttpClientFactory::class),
            new AiVisibilityResultNormalizer,
        );

        $this->expectException(RuntimeException::class);
        $client->search($this->provider(), '   ');
    }

    public function test_it_throws_on_non_successful_response(): void
    {
        $provider = $this->provider();
        $response = new Response(new \GuzzleHttp\Psr7\Response(429, [], 'rate limited'));

        $request = Mockery::mock();
        $request->shouldReceive('post')->once()->andReturn($response);

        $factory = Mockery::mock(AiVisibilityHttpClientFactory::class);
        $factory->shouldReceive('jsonRequest')->once()->andReturn($request);

        $crypto = Mockery::mock(ApiKeyCrypto::class);
        $crypto->shouldReceive('decrypt')->once()->andReturn('secret-key');

        $client = new PerplexitySearchClient($crypto, $factory, new AiVisibilityResultNormalizer);

        $this->expectException(RuntimeException::class);
        $this->expectExceptionMessageMatches('/HTTP 429/');
        $client->search($provider, 'query');
    }

    private function provider(): AiSourceProvider
    {
        $provider = new AiSourceProvider;
        $provider->provider_key = AiSourceProvider::PROVIDER_PERPLEXITY_SEARCH;
        $provider->endpoint_url = 'https://api.perplexity.ai/chat/completions';
        $provider->setRawAttributes(['api_key' => 'encrypted'], true);

        return $provider;
    }
}
```

> mock 的类名与 `jsonRequest` 的返回类型需以实际为准；实现前先看
> `sed -n '1,60p' tests/Unit/*.php` 中是否有既有的 HTTP mock 范例，优先复用其写法。

- [ ] **步骤 2：运行测试验证失败**

运行：`php artisan test --filter=PerplexitySearchClientTest`
预期：FAIL，报错 `Class "App\Services\GeoFlow\AiVisibility\PerplexitySearchClient" not found`

- [ ] **步骤 3：编写最少实现代码**

```php
<?php

namespace App\Services\GeoFlow\AiVisibility;

use App\Models\AiSourceProvider;
use App\Support\GeoFlow\ApiKeyCrypto;
use RuntimeException;

final class PerplexitySearchClient
{
    public function __construct(
        private readonly ApiKeyCrypto $apiKeyCrypto,
        private readonly AiVisibilityHttpClientFactory $httpClientFactory,
        private readonly AiVisibilityResultNormalizer $normalizer,
    ) {}

    /**
     * @param  array<string,mixed>  $options
     */
    public function search(AiSourceProvider $provider, string $query, array $options = []): AiVisibilityResult
    {
        $query = trim($query);
        if ($query === '') {
            throw new RuntimeException('Perplexity 查询词为空');
        }

        $endpoint = $this->endpoint($provider);
        $apiKey = $this->apiKey($provider);
        $payload = $this->buildPayload($query, $options);

        $startedAt = hrtime(true);
        $response = $this->httpClientFactory
            ->jsonRequest($apiKey)
            ->post($endpoint, $payload);
        $latencyMs = (int) round((hrtime(true) - $startedAt) / 1_000_000);

        if (! $response->successful()) {
            throw new RuntimeException(sprintf(
                'Perplexity 请求失败：HTTP %d %s',
                $response->status(),
                trim($response->body())
            ));
        }

        $json = $response->json();
        if (! is_array($json)) {
            throw new RuntimeException('Perplexity 返回了非 JSON 结构');
        }

        return $this->normalizer->normalizePerplexity($json, [
            'endpoint' => $endpoint,
            'payload' => $payload,
        ], $latencyMs);
    }

    /**
     * @param  array<string,mixed>  $options
     * @return array<string,mixed>
     */
    private function buildPayload(string $query, array $options): array
    {
        return array_filter([
            'model' => (string) ($options['model'] ?? config('geoflow.ai_visibility.perplexity_model', 'sonar')),
            'messages' => [
                ['role' => 'user', 'content' => $query],
            ],
        ], static fn (mixed $value): bool => $value !== null && $value !== '' && $value !== []);
    }

    private function endpoint(AiSourceProvider $provider): string
    {
        $endpoint = trim((string) ($provider->endpoint_url ?? ''));
        if ($endpoint === '') {
            $endpoint = trim((string) config('geoflow.ai_visibility.perplexity_endpoint', ''));
        }

        if ($endpoint === '') {
            throw new RuntimeException('Perplexity Endpoint 为空');
        }

        return $endpoint;
    }

    private function apiKey(AiSourceProvider $provider): string
    {
        $apiKey = $this->apiKeyCrypto->decrypt((string) ($provider->getRawOriginal('api_key') ?? ''));
        if ($apiKey === '') {
            throw new RuntimeException('Perplexity API Key 为空');
        }

        return $apiKey;
    }
}
```

同时在 `config/geoflow.php` 的 `ai_visibility` 段追加两个默认值（与既有 `doubao_search_endpoint` 同级）：

```php
        'perplexity_endpoint' => env('GEOFLOW_PERPLEXITY_ENDPOINT', 'https://api.perplexity.ai/chat/completions'),
        'perplexity_model' => env('GEOFLOW_PERPLEXITY_MODEL', 'sonar'),
```

- [ ] **步骤 4：运行测试验证通过**

运行：`php artisan test --filter=PerplexitySearchClientTest`
预期：PASS，3 个用例通过。

- [ ] **步骤 5：Commit**

```bash
git add app/Services/GeoFlow/AiVisibility/PerplexitySearchClient.php \
        config/geoflow.php tests/Unit/PerplexitySearchClientTest.php
git commit -m "feat(visibility): 新增 Perplexity 检索型采样客户端"
```

---

### 任务 B3：Service 层 run 方法

**文件：**
- 修改：`app/Services/GeoFlow/AiVisibility/AiVisibilityService.php`（构造函数注入 + 新增方法）
- 测试：`tests/Feature/AiVisibilityPerplexityRunTest.php`（新建）

- [ ] **步骤 1：编写失败的测试**

```php
<?php

namespace Tests\Feature;

use App\Models\AiSourceProvider;
use App\Models\AiVisibilityRun;
use App\Services\GeoFlow\AiVisibility\AiVisibilityService;
use App\Services\GeoFlow\AiVisibility\PerplexitySearchClient;
use App\Services\GeoFlow\AiVisibility\AiVisibilityResult;
use App\Services\GeoFlow\AiVisibility\AiVisibilitySourceData;
use Mockery;
use Tests\TestCase;

class AiVisibilityPerplexityRunTest extends TestCase
{
    public function test_it_records_a_perplexity_run(): void
    {
        $provider = AiSourceProvider::factory()->create([
            'provider_key' => AiSourceProvider::PROVIDER_PERPLEXITY_SEARCH,
            'endpoint_url' => 'https://api.perplexity.ai/chat/completions',
            'status' => 'active',
            'daily_limit' => 100,
        ]);

        $client = Mockery::mock(PerplexitySearchClient::class);
        $client->shouldReceive('search')->once()->andReturn(new AiVisibilityResult(
            providerType: AiVisibilityRun::PROVIDER_PERPLEXITY_SEARCH,
            providerKey: AiSourceProvider::PROVIDER_PERPLEXITY_SEARCH,
            modelId: 'sonar',
            answerText: '推荐 A 公司。',
            sources: [new AiVisibilitySourceData(url: 'https://example.com/a', title: 'A', publishedAt: '', position: 1)],
            usage: [],
            metadata: [],
            rawRequest: [],
            rawResponse: [],
            latencyMs: 12,
        ));

        $this->app->instance(PerplexitySearchClient::class, $client);

        $run = $this->app->make(AiVisibilityService::class)->runPerplexitySearch($provider, '中国耳塞设计工厂');

        $this->assertSame(AiVisibilityRun::STATUS_COMPLETED, $run->status);
        $this->assertSame('推荐 A 公司。', $run->answer_text);
        $this->assertDatabaseHas('ai_visibility_sources', ['url' => 'https://example.com/a']);
    }
}
```

> 工厂与配额相关辅助以实际为准；实现前先看 `tests/Feature/AdminAiVisibilityAnalyticsTest.php` 的既有写法并复用。

- [ ] **步骤 2：运行测试验证失败**

运行：`php artisan test --filter=AiVisibilityPerplexityRunTest`
预期：FAIL，报错 `Call to undefined method ...::runPerplexitySearch()`

- [ ] **步骤 3：编写最少实现代码**

在 `AiVisibilityService` 构造函数注入 `PerplexitySearchClient $perplexitySearchClient`，
并新增方法（结构完全对照既有的 `runDoubaoSearchCustom`，包括配额预留与失败回收）：

```php
    /**
     * @param  array<string,mixed>  $options
     */
    public function runPerplexitySearch(AiSourceProvider $provider, string $keyword, array $options = []): AiVisibilityRun
    {
        $keyword = $this->normalizeKeyword($keyword);
        $run = $this->createRun([
            'keyword' => $keyword,
            'prompt' => (string) ($options['prompt'] ?? $keyword),
            'provider_type' => AiVisibilityRun::PROVIDER_PERPLEXITY_SEARCH,
            'provider_key' => (string) ($provider->provider_key ?? AiSourceProvider::PROVIDER_PERPLEXITY_SEARCH),
            'ai_source_provider_id' => (int) $provider->id,
            'locale' => (string) ($options['locale'] ?? 'en_US'),
        ]);

        $reservation = null;
        try {
            $this->assertSourceProviderEnabled($provider, 'Perplexity 信源供应商');
            $reservation = $this->usageQuota->reserveProvider($provider);
            if ($reservation === null) {
                throw new RuntimeException('ai_source_provider_quota_exhausted');
            }
            $result = $this->perplexitySearchClient->search(
                $provider,
                $keyword,
                array_replace($provider->visibilitySearchOptions(), $options),
            );

            return $this->completeRun($run, $result, providerReservation: $reservation);
        } catch (Throwable $exception) {
            if ($reservation !== null) {
                $this->usageQuota->releaseProvider($reservation);
            }
            $errorCode = $this->safeErrorCode($exception);
            $this->failRun($run, $errorCode);
            throw new RuntimeException($errorCode);
        }
    }
```

> **`locale` 用 `en_US`**（而非既有国内引擎的 `zh_CN`）——目标市场在海外，采样语境必须是英文。

- [ ] **步骤 4：运行测试验证通过**

运行：`php artisan test --filter=AiVisibilityPerplexityRunTest`
预期：PASS。

- [ ] **步骤 5：Commit**

```bash
git add app/Services/GeoFlow/AiVisibility/AiVisibilityService.php \
        tests/Feature/AiVisibilityPerplexityRunTest.php
git commit -m "feat(visibility): Service 层支持 Perplexity 采样"
```

---

### 任务 B4：接入采集调度（★ 决定新引擎是否真被调用）

**文件：**
- 修改：`app/Services/GeoFlow/AiVisibility/AiVisibilityCollectionService.php:20-67`
- 测试：`tests/Feature/AiVisibilityCollectionDispatchTest.php`（新建）

- [ ] **步骤 1：编写失败的测试**

```php
<?php

namespace Tests\Feature;

use App\Models\AiSourceProvider;
use App\Services\GeoFlow\AiVisibility\AiVisibilityCollectionService;
use App\Services\GeoFlow\AiVisibility\AiVisibilityService;
use Mockery;
use Tests\TestCase;

class AiVisibilityCollectionDispatchTest extends TestCase
{
    public function test_it_collects_from_both_overseas_engines_for_the_same_keyword(): void
    {
        AiSourceProvider::factory()->create([
            'provider_key' => AiSourceProvider::PROVIDER_PERPLEXITY_SEARCH,
            'endpoint_url' => 'https://api.perplexity.ai/chat/completions',
            'status' => 'active',
        ]);
        AiSourceProvider::factory()->create([
            'provider_key' => AiSourceProvider::PROVIDER_OPENAI_WEB_SEARCH,
            'endpoint_url' => 'https://api.openai.com/v1/responses',
            'status' => 'active',
        ]);

        $service = Mockery::mock(AiVisibilityService::class);
        $service->shouldReceive('runPerplexitySearch')->once();
        $service->shouldReceive('runOpenAiWebSearch')->once();
        $this->app->instance(AiVisibilityService::class, $service);

        $this->app->make(AiVisibilityCollectionService::class)->collect($this->systemIdentity(), '中国耳塞设计工厂');
    }
}
```

- [ ] **步骤 2：运行测试验证失败**

运行：`php artisan test --filter=AiVisibilityCollectionDispatchTest`
预期：FAIL —— mock 的期望未被满足（新引擎未被调用），证明改造前新引擎确实选不中。

- [ ] **步骤 3：编写最少实现代码**

把 `collect()` 从「单路径 if/else 链」改为「按可用信源逐个采集并汇总」：

```php
    /**
     * @return array<string, AiVisibilityRun>
     */
    public function collect(SystemAiIdentity $identity, string $keyword): array
    {
        $identity->assertCanCollectVisibility();
        $keyword = trim($keyword);
        if ($keyword === '') {
            throw new RuntimeException('AI 可见性关键词为空');
        }

        $runs = [];

        $perplexity = $this->configuration->perplexityProvider($identity);
        if ($perplexity instanceof AiSourceProvider) {
            $runs['perplexity_run'] = $this->visibility->runPerplexitySearch($perplexity, $keyword);
        }

        $openAi = $this->configuration->openAiProvider($identity);
        if ($openAi instanceof AiSourceProvider) {
            $runs['openai_run'] = $this->visibility->runOpenAiWebSearch($openAi, $keyword);
        }

        // 既有国内链路保持不变，作为兜底
        if ($runs === []) {
            return $this->collectDomestic($identity, $keyword);
        }

        return $runs;
    }

    /**
     * 既有的豆包 / DeepSeek 采集链路，原样抽出为私有方法，行为不变。
     *
     * @return array<string, AiVisibilityRun>
     */
    private function collectDomestic(SystemAiIdentity $identity, string $keyword): array
    {
        // 把原 collect() 第 29-64 行的逻辑整体搬到这里，不做任何修改
    }
```

> **这是本计划最关键的一处改造**：不改这里，前四个任务产出的 Client 永远不会被调用。
> **默认值策略**：新引擎均未配置时回落到原国内链路，保证既有部署不被破坏。

- [ ] **步骤 4：运行测试验证通过**

运行：`php artisan test --filter=AiVisibilityCollectionDispatchTest`
预期：PASS。

- [ ] **步骤 5：回归验证国内链路未被破坏**

运行：`php artisan test --filter=AiVisibility`
预期：PASS。若 `collectDomestic` 抽取过程中改动了任何逻辑，既有测试会失败。

- [ ] **步骤 6：Commit**

```bash
git add app/Services/GeoFlow/AiVisibility/AiVisibilityCollectionService.php \
        tests/Feature/AiVisibilityCollectionDispatchTest.php
git commit -m "feat(visibility): 采集调度接入海外引擎，同一问题对多引擎各采一次"
```

---
## 任务组 C：OpenAI 引擎端到端

> OpenAI 走 **Responses API + `web_search` 工具**，其响应结构与 Ark 同构，因此
> **复用任务 B1 抽取出的 `parseResponsesPayload()`**，不重复实现解析（DRY）。

### 任务 C1：归一化器支持 OpenAI Responses

**文件：**
- 修改：`app/Services/GeoFlow/AiVisibility/AiVisibilityResultNormalizer.php`
- 测试：`tests/Unit/AiVisibilityResultNormalizerTest.php`（追加）

- [ ] **步骤 1：编写失败的测试**

```php
    public function test_it_normalizes_openai_web_search_annotations(): void
    {
        $result = (new AiVisibilityResultNormalizer)->normalizeOpenAiWebSearch([
            'id' => 'resp_openai_1',
            'output' => [
                ['type' => 'web_search_call', 'id' => 'ws_1', 'status' => 'completed', 'action' => ['query' => 'earplug factory china']],
                ['type' => 'message', 'content' => [[
                    'type' => 'output_text',
                    'text' => 'We recommend A Corp.',
                    'annotations' => [['type' => 'url_citation', 'title' => 'A Corp', 'url' => 'https://example.com/a']],
                ]]],
            ],
        ], ['model' => 'gpt-4o'], 88);

        $this->assertSame(AiVisibilityRun::PROVIDER_OPENAI_WEB_SEARCH, $result->providerType);
        $this->assertSame('We recommend A Corp.', $result->answerText);
        $this->assertCount(1, $result->sources);
        $this->assertSame(88, $result->latencyMs);
    }

    public function test_ark_and_openai_share_the_same_parser_but_differ_in_provider_identity(): void
    {
        $normalizer = new AiVisibilityResultNormalizer;
        $payload = ['output' => [['type' => 'message', 'content' => [['type' => 'output_text', 'text' => 'X']]]]];

        $ark = $normalizer->normalizeArkResponses($payload, [], 'ark-model', 1);
        $openai = $normalizer->normalizeOpenAiWebSearch($payload, [], 1);

        $this->assertSame('X', $ark->answerText);
        $this->assertSame('X', $openai->answerText);
        $this->assertNotSame($ark->providerType, $openai->providerType);
    }
```

- [ ] **步骤 2：运行测试验证失败**

运行：`php artisan test --filter=AiVisibilityResultNormalizerTest`
预期：FAIL，报错 `Call to undefined method ...::normalizeOpenAiWebSearch()`

- [ ] **步骤 3：编写最少实现代码**

```php
    /**
     * @param  array<string,mixed>  $response
     * @param  array<string,mixed>  $request
     */
    public function normalizeOpenAiWebSearch(array $response, array $request, int $latencyMs): AiVisibilityResult
    {
        $parsed = $this->parseResponsesPayload($response);

        return new AiVisibilityResult(
            providerType: AiVisibilityRun::PROVIDER_OPENAI_WEB_SEARCH,
            providerKey: AiSourceProvider::PROVIDER_OPENAI_WEB_SEARCH,
            modelId: $this->stringValue($request['model'] ?? ''),
            answerText: $parsed['segments'] === []
                ? $this->stringValue($response['output_text'] ?? '')
                : trim(implode("\n\n", $parsed['segments'])),
            sources: $parsed['sources'],
            usage: is_array($response['usage'] ?? null) ? $response['usage'] : [],
            metadata: array_filter([
                'response_id' => $parsed['response_id'],
                'web_search_calls' => $parsed['web_search_calls'],
            ], static fn (mixed $value): bool => $value !== null && $value !== '' && $value !== []),
            rawRequest: $request,
            rawResponse: $response,
            latencyMs: $latencyMs,
        );
    }
```

- [ ] **步骤 4：运行测试验证通过**

运行：`php artisan test --filter=AiVisibilityResultNormalizerTest`
预期：PASS，全部用例通过。

- [ ] **步骤 5：Commit**

```bash
git add app/Services/GeoFlow/AiVisibility/AiVisibilityResultNormalizer.php \
        tests/Unit/AiVisibilityResultNormalizerTest.php
git commit -m "feat(visibility): 归一化器支持 OpenAI web_search 响应"
```

---

### 任务 C2：OpenAiWebSearchClient

**文件：**
- 创建：`app/Services/GeoFlow/AiVisibility/OpenAiWebSearchClient.php`
- 测试：`tests/Unit/OpenAiWebSearchClientTest.php`（新建）

> **实现前必做**：用 `mcp__context7__query-docs` 核对 OpenAI Responses API 的实时契约——
> 特别是 `tools` 中 `web_search` 的写法与 `input` 的取值形式。**不得凭记忆写死**。

- [ ] **步骤 1：编写失败的测试**

```php
<?php

namespace Tests\Unit;

use App\Models\AiSourceProvider;
use App\Services\GeoFlow\AiVisibility\AiVisibilityHttpClientFactory;
use App\Services\GeoFlow\AiVisibility\AiVisibilityResultNormalizer;
use App\Services\GeoFlow\AiVisibility\OpenAiWebSearchClient;
use App\Support\GeoFlow\ApiKeyCrypto;
use Illuminate\Http\Client\Response;
use Mockery;
use PHPUnit\Framework\TestCase;

class OpenAiWebSearchClientTest extends TestCase
{
    protected function tearDown(): void
    {
        Mockery::close();
        parent::tearDown();
    }

    public function test_it_enables_the_web_search_tool(): void
    {
        $response = new Response(new \GuzzleHttp\Psr7\Response(200, [], json_encode([
            'id' => 'resp_1',
            'output' => [['type' => 'message', 'content' => [['type' => 'output_text', 'text' => 'Answer']]]],
        ])));

        $request = Mockery::mock();
        $request->shouldReceive('post')->once()->with('https://api.openai.com/v1/responses', Mockery::on(
            fn (array $payload): bool => ($payload['tools'][0]['type'] ?? null) === 'web_search'
                && ($payload['input'] ?? null) === 'earplug factory china'
        ))->andReturn($response);

        $factory = Mockery::mock(AiVisibilityHttpClientFactory::class);
        $factory->shouldReceive('jsonRequest')->once()->with('sk-test')->andReturn($request);

        $crypto = Mockery::mock(ApiKeyCrypto::class);
        $crypto->shouldReceive('decrypt')->once()->andReturn('sk-test');

        $client = new OpenAiWebSearchClient($crypto, $factory, new AiVisibilityResultNormalizer);
        $result = $client->search($this->provider(), 'earplug factory china');

        $this->assertSame('Answer', $result->answerText);
    }

    private function provider(): AiSourceProvider
    {
        $provider = new AiSourceProvider;
        $provider->provider_key = AiSourceProvider::PROVIDER_OPENAI_WEB_SEARCH;
        $provider->endpoint_url = 'https://api.openai.com/v1/responses';
        $provider->setRawAttributes(['api_key' => 'encrypted'], true);

        return $provider;
    }
}
```

- [ ] **步骤 2：运行测试验证失败**

运行：`php artisan test --filter=OpenAiWebSearchClientTest`
预期：FAIL，报错 `Class ...OpenAiWebSearchClient not found`

- [ ] **步骤 3：编写最少实现代码**

```php
<?php

namespace App\Services\GeoFlow\AiVisibility;

use App\Models\AiSourceProvider;
use App\Support\GeoFlow\ApiKeyCrypto;
use RuntimeException;

final class OpenAiWebSearchClient
{
    public function __construct(
        private readonly ApiKeyCrypto $apiKeyCrypto,
        private readonly AiVisibilityHttpClientFactory $httpClientFactory,
        private readonly AiVisibilityResultNormalizer $normalizer,
    ) {}

    /**
     * @param  array<string,mixed>  $options
     */
    public function search(AiSourceProvider $provider, string $query, array $options = []): AiVisibilityResult
    {
        $query = trim($query);
        if ($query === '') {
            throw new RuntimeException('OpenAI 查询词为空');
        }

        $endpoint = $this->endpoint($provider);
        $apiKey = $this->apiKey($provider);
        $payload = $this->buildPayload($query, $options);

        $startedAt = hrtime(true);
        $response = $this->httpClientFactory
            ->jsonRequest($apiKey)
            ->post($endpoint, $payload);
        $latencyMs = (int) round((hrtime(true) - $startedAt) / 1_000_000);

        if (! $response->successful()) {
            throw new RuntimeException(sprintf(
                'OpenAI 请求失败：HTTP %d %s',
                $response->status(),
                trim($response->body())
            ));
        }

        $json = $response->json();
        if (! is_array($json)) {
            throw new RuntimeException('OpenAI 返回了非 JSON 结构');
        }

        return $this->normalizer->normalizeOpenAiWebSearch($json, [
            'endpoint' => $endpoint,
            'payload' => $payload,
            'model' => $payload['model'] ?? '',
        ], $latencyMs);
    }

    /**
     * @param  array<string,mixed>  $options
     * @return array<string,mixed>
     */
    private function buildPayload(string $query, array $options): array
    {
        return array_filter([
            'model' => (string) ($options['model'] ?? config('geoflow.ai_visibility.openai_model', 'gpt-4o')),
            'input' => $query,
            'tools' => [['type' => 'web_search']],
        ], static fn (mixed $value): bool => $value !== null && $value !== '' && $value !== []);
    }

    private function endpoint(AiSourceProvider $provider): string
    {
        $endpoint = trim((string) ($provider->endpoint_url ?? ''));
        if ($endpoint === '') {
            $endpoint = trim((string) config('geoflow.ai_visibility.openai_endpoint', ''));
        }

        if ($endpoint === '') {
            throw new RuntimeException('OpenAI Endpoint 为空');
        }

        return $endpoint;
    }

    private function apiKey(AiSourceProvider $provider): string
    {
        $apiKey = $this->apiKeyCrypto->decrypt((string) ($provider->getRawOriginal('api_key') ?? ''));
        if ($apiKey === '') {
            throw new RuntimeException('OpenAI API Key 为空');
        }

        return $apiKey;
    }
}
```

在 `config/geoflow.php` 追加：

```php
        'openai_endpoint' => env('GEOFLOW_OPENAI_ENDPOINT', 'https://api.openai.com/v1/responses'),
        'openai_model' => env('GEOFLOW_OPENAI_MODEL', 'gpt-4o'),
```

- [ ] **步骤 4：运行测试验证通过**

运行：`php artisan test --filter=OpenAiWebSearchClientTest`
预期：PASS。

- [ ] **步骤 5：Service 层与方法接入**

**5a.** 在 `AiVisibilityService` 注入 `OpenAiWebSearchClient $openAiWebSearchClient`，并新增 `runOpenAiWebSearch()`：
结构完全对照任务 B3 的 `runPerplexitySearch()`，差异仅三处：

| 项 | Perplexity | OpenAI |
|---|---|---|
| `provider_type` | `PROVIDER_PERPLEXITY_SEARCH` | `PROVIDER_OPENAI_WEB_SEARCH` |
| Client 调用 | `$this->perplexitySearchClient->search(...)` | `$this->openAiWebSearchClient->search(...)` |
| 供应商名（错误文案） | `'Perplexity 信源供应商'` | `'OpenAI 信源供应商'` |

**5b.** 确认 `AiVisibilityCollectionService::collect()`（任务 B4 已写）中的 `runOpenAiWebSearch` 分支现在能解析到方法。

- [ ] **步骤 6：运行测试验证通过**

运行：`php artisan test --filter=AiVisibilityCollectionDispatchTest`
预期：PASS —— 两个引擎都被调用。

- [ ] **步骤 7：Commit**

```bash
git add app/Services/GeoFlow/AiVisibility/OpenAiWebSearchClient.php \
        app/Services/GeoFlow/AiVisibility/AiVisibilityService.php \
        config/geoflow.php tests/Unit/OpenAiWebSearchClientTest.php
git commit -m "feat(visibility): 新增 OpenAI web_search 检索型采样客户端与 Service 接入"
```

---
## 任务组 D：后台配置与端到端验收

### 任务 D1：在后台登记两个海外信源

> 这是**配置任务**，不是编码任务——底座已有信源供应商管理界面。

- [ ] **步骤 1：确认后台入口存在**

```bash
grep -rn "AiSourceProviderController\|ai-source-providers" routes/web.php | head -5
```
预期：能找到信源供应商的后台路由。

- [ ] **步骤 2：在后台创建两个信源供应商**

| 字段 | Perplexity | OpenAI |
|---|---|---|
| `provider_key` | `perplexity_search` | `openai_web_search` |
| `endpoint_url` | `https://api.perplexity.ai/chat/completions` | `https://api.openai.com/v1/responses` |
| `api_key` | 你们的 Perplexity Key | 你们的 OpenAI Key |
| `status` | `active` | `active` |
| `daily_limit` | 按预算设定，**先设小值（如 50）** | 同上 |

- [ ] **步骤 3：验证端点被策略接受**

在后台保存时，若端点不在白名单会被拒绝。若被拒绝，回到任务 A1 检查白名单。

- [ ] **步骤 4：验证配额生效**

```bash
php artisan tinker --execute="echo App\\Models\\AiSourceProvider::where('provider_key','perplexity_search')->value('daily_limit');"
```
预期：输出你设置的额度值。

- [ ] **步骤 5：Commit（配置模板，不含密钥）**

```bash
git add -A && git status --short
```
预期：**不应有 `.env` 或含密钥的文件进入暂存区**。若出现，立即移除并检查 `.gitignore`。
本任务通常无需提交代码，直接进入 D2。

---

### 任务 D2：端到端验收与基线建立

- [ ] **步骤 1：跑通一次完整采集（验收 AC1）**

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml exec app \
  php artisan geoflow:ai-visibility:collect "original design earplug manufacturers in China"
```

- [ ] **验证**：命令输出类似 `AI visibility collected: keyword=..., runs=2`，`runs=2` 表示**两个引擎都被采到**。
- [ ] **验证**：数据库中有对应记录

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml exec app \
  php artisan tinker --execute="echo App\\Models\\AiVisibilityRun::whereIn('provider_type',['perplexity_search','openai_web_search'])->count();"
```
预期：输出 ≥ 2。

- [ ] **步骤 2：确认信源被记录（决定后续引用分析可用）**

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml exec app \
  php artisan tinker --execute="echo App\\Models\\AiVisibilitySource::count();"
```
预期：输出 ≥ 1。**若为 0，说明引擎没有返回引用**，回到对应 Client 检查是否真的走了检索（这是 R1.1a 明确禁止的退化情形）。

- [ ] **步骤 3：在看板确认（验收 AC1 收尾）**

打开后台 `/admin/ai-visibility`，确认：

- [ ] 「提供商」筛选里出现两个新引擎
- [ ] 能看到回答原文
- [ ] 「采样质量」指标有值

- [ ] **步骤 4：建立基线（验收 AC2）**

准备 10-20 条真实目标问题（海外买手可能问的），逐条采集：

```bash
for q in "original design earplug manufacturers in China" "custom earplug OEM China" ; do
  docker compose --env-file .env.prod -f docker-compose.prod.yml exec -T app \
    php artisan geoflow:ai-visibility:collect "$q";
done
```

- [ ] **验证**：记录下**基线数字**——在多少条目标问题上 AI 提及了本公司。这个数字是本项目后续一切对比的起点。
- [ ] **验证**：把这个数字与采集日期写进 `docs/` 下的基线记录文件，并提交到 `ljccc2025/GEO` 仓库。

- [ ] **步骤 5：用独立工具交叉验证（验收依据）**

```bash
pip install geo-optimizer-skill
geo citations --brand "<公司英文名>" \
  --topic "original design earplug manufacturers in China" --runs 5
```

- [ ] **验证**：把该工具的结果与底座看板结果并列记录。**两者差异大时，以独立工具为准**——它不共享底座的假设。

- [ ] **步骤 6：确保采样随机性被尊重**

同一问题连续采集 5 次，观察结果是否波动：

```bash
for i in 1 2 3 4 5; do
  docker compose --env-file .env.prod -f docker-compose.prod.yml exec -T app \
    php artisan geoflow:ai-visibility:collect "original design earplug manufacturers in China";
done
```

- [ ] **验证**：记录波动情况。**基线必须写成区间或趋势，不能写成单次快照**——单次采样不构成结论。

- [ ] **步骤 7：合并分支并推送**

```bash
git push -u origin feat/overseas-visibility-engines
# 在 GitHub 上对 ljccc2025/GEOFlow 开 PR 到 main，CI 通过后合并
```

---

## 自检

### 规格覆盖度

| 规格要求 | 对应任务 | 状态 |
|---|---|---|
| R1.1 新增两个海外引擎，均建模为检索型 `AiSourceProvider` | A2（常量）、B2（Perplexity）、C2（OpenAI，含 `web_search` 工具） | 已覆盖 |
| R1.1a 不得退化为无检索调用 | B2 步骤 1、C2 步骤 1（测试断言 payload）、D2 步骤 2（断言 sources ≥ 1） | 已覆盖 |
| R1.2 挂进 `SAMPLE_PROVIDERS` | A2 | 已覆盖 |
| R1.3 每个引擎独立 Client 类 | B2、C2 | 已覆盖 |
| R1.4 后台可配置凭据 | D1 | 已覆盖 |
| R1.5 写入 `ai_visibility_runs`，字段一致 | B3、C2 步骤 5 | 已覆盖 |
| R1.6 提示词支持英文，不硬编码中文 | B3（`locale=en_US`）、C2 步骤 5 | 已覆盖 |
| R1.7 扩展白名单 | A1 | 已覆盖 |
| R1.8 不绕过 SSRF 防护 | A1 步骤 3（保留 `isTrustedHttpsUrl`）、步骤 4（6 个用例） | 已覆盖 |
| R1.9 复用配额机制 | B3、C2 步骤 5（`reserveProvider` / `releaseProvider`） | 已覆盖 |
| R1.10 扩展采集调度，两引擎各采一次 | B4 | 已覆盖 |
| R1.11 扩展配置解析 | A3 | 已覆盖 |
| R1.12 命令与后台入口可驱动新引擎 | D2 步骤 1、3 | 已覆盖 |
| §6.4 R6.1–R6.3 品牌身份 | M0-3、M0-4 | 已覆盖 |
| AC1 跑通一次完整采样 | D2 步骤 1、3 | 已覆盖 |
| AC2 基线落库 | D2 步骤 4 | 已覆盖 |
| M2（公司档案 + Schema） | —— | **不在本计划**，另立计划 |

### 类型一致性

| 名称 | 定义处 | 使用处 |
|---|---|---|
| `PROVIDER_PERPLEXITY_SEARCH` | A2（`AiVisibilityRun` 与 `AiSourceProvider` 各一份） | B1、B2、B3、B4 |
| `PROVIDER_OPENAI_WEB_SEARCH` | A2 | C1、C2、B4 |
| `normalizePerplexity()` | B1 | B2 |
| `normalizeOpenAiWebSearch()` | C1 | C2 |
| `parseResponsesPayload()` | B1（私有） | B1、C1 |
| `runPerplexitySearch()` | B3 | B4 |
| `runOpenAiWebSearch()` | C2 步骤 5 | B4 |
| `perplexityProvider()` / `openAiProvider()` | A3 | B4 |
| `searchProvider($identity, $providerKey)` | A3 | A3、B4 |

### 已知的不完美之处（如实记录）

1. **任务 C2 步骤 5 引用了任务 B3 的结构**。技能规则要求「不得写类似任务 N」，此处以差异对照表替代了重复贴码。
   执行者若不按顺序阅读，请先回到 B3 阅读 `runPerplexitySearch()` 的完整实现。
2. **两处外部 API 契约需现场核实**（B2、C2 的步骤 1 前）。Perplexity 与 OpenAI 的请求/响应字段可能已演进，
   计划中的 payload 与归一化字段基于当前公开契约。**实现前必须核对**，若不符则先改测试再改实现。
3. **测试代码中的 mock 写法需与既有范例对齐**（B2、B3 已标注）。
   项目内 HTTP mock 的既有写法可能与本计划示例不同，**以既有测试为准**。

---

## 执行交接

计划已完成并保存到 `docs/superpowers/plans/2026-09-14-overseas-visibility-engines.md`（`ljccc2025/GEO` 仓库）。

**两种执行方式：**

1. **子代理驱动（推荐）** —— 每个任务调度一个新的子代理，任务间进行审查，快速迭代。
   必需子技能：`superpowers:subagent-driven-development`

2. **内联执行** —— 在当前会话中使用 `executing-plans` 执行任务，批量执行并设有检查点。
   必需子技能：`superpowers:executing-plans`

**选哪种方式？**

---

## 前置提醒

**M0-1（海外 API 连通性）是整个计划的第一道闸门。** 它属于未验证假设 A1：

- 通过 → 按本计划执行
- 不通过 → **计划阻塞**，需要先解决服务器出网条件。在解决之前，A1 之后的所有任务都没有意义。

**代码仓库与文档仓库是分开的：**

| 内容 | 仓库 |
|---|---|
| 本计划、规格、技术方案 | `ljccc2025/GEO` |
| 实际代码改动（`app/`、`tests/` 等） | `ljccc2025/GEOFlow`（fork） |
