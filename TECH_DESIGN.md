# Pain-Point Digest 技术方案

## 1. 技术目标

Pain-Point Digest 的技术系统不是单纯的信息抓取器，而是一个面向独立开发者的“产品机会发现与快速验证系统”。

系统最终要支撑这条闭环：

```text
社区痛点发现 -> 机会评分 -> 验证包生成 -> MVP 开发辅助 -> 一键部署 -> 数据反馈 -> 是否继续投入
```

但 MVP 阶段只实现最短可验证链路：

```text
抓取 -> 预过滤 -> LLM 精筛 -> 结构化机会 -> 邮件日报
```

Phase 1 的技术目标是验证“痛点情报是否真的有用”，不提前建设完整 Web 平台、自动部署平台或复杂运维系统。

## 2. 阶段拆分

| 阶段 | 产品目标 | 技术目标 | 不做什么 |
| --- | --- | --- | --- |
| Phase 1 | 每天发现高质量产品机会 | 跑通自动抓取、AI 评分、邮件发送 | 不做 Web 看板、不做一键部署 |
| Phase 2 | 从日报升级为机会库 | 增加 Web 看板、搜索、语义去重、验证包生成 | 不做完整云平台 |
| Phase 3 | 辅助 AI 开发 MVP | 生成 PRD、任务拆解、Cursor/Claude Code Prompt、GitHub Issue | 不托管用户代码 |
| Phase 4 | 快速上线验证 | 集成 Vercel/Supabase/GitHub Actions，提供部署编排和轻运维 | 不自研 PaaS/IaaS |

## 3. Phase 1 总体架构

```text
GitHub Actions Cron
        |
        v
Python Pipeline
        |
        +--> HN Algolia API
        +--> Reddit Data API
        +--> V2EX RSS
        |
        v
Keyword Pre-filter
        |
        v
LLM Scoring
        |
        v
Turso (libSQL)
        |
        v
HTML Email Renderer
        |
        v
SMTP Provider
```

### 3.1 技术选型

| 层 | Phase 1 选型 | 原因 |
| --- | --- | --- |
| 语言 | Python 3.12 | 抓取、LLM 调用、邮件生成生态成熟 |
| 调度 | GitHub Actions cron | 零运维，适合早期自动任务 |
| 存储 | Turso（托管 libSQL） | 兼容 SQLite API，跨 GitHub Actions run 持久化，有免费 tier，无需等 Phase 2 才迁移 |
| LLM | DeepSeek / Anthropic | DeepSeek 控成本，Anthropic 做高质量备选 |
| 邮件 | SMTP | 实现简单，可接 Resend、Brevo、Gmail SMTP 等 |
| 模板 | Jinja2 | HTML 邮件模板清晰可维护 |
| 配置 | `.env` + GitHub Secrets | 本地开发和线上任务统一 |
| 运营管理 | Retool / Metabase 接 Turso | Phase 1 不自建管理后台，10 分钟接好，够管理早期 50 个订阅者 |

## 4. Phase 1 模块设计

### 4.1 数据抓取模块

职责：从合规数据源拉取候选帖子，统一转换为内部 `RawPost` 结构。

#### 扩展性设计

所有数据源继承统一抽象基类，新增来源只需新建一个文件，主流程不改：

```python
# sources/base.py
from abc import ABC, abstractmethod

class BaseSource(ABC):
    source_name: str

    @abstractmethod
    async def fetch(self) -> list[RawPost]:
        """拉取过去 24 小时的候选帖子"""
        ...

    @abstractmethod
    async def health_check(self) -> bool:
        """数据源是否可用"""
        ...
```

`main.py` 动态加载所有 source，单一数据源故障不影响整体：

```python
SOURCES: list[BaseSource] = [HNSource(), RedditSource(), V2EXSource()]

posts = []
for source in SOURCES:
    if await source.health_check():
        posts.extend(await source.fetch())
```

#### Phase 1 数据源

| 来源 | 接入方式 | 备注 |
| --- | --- | --- |
| Hacker News | Algolia HN API | 只取公开帖子和评论 |
| Reddit | Reddit Data API | 使用 OAuth，遵守 API 限制 |
| V2EX | RSS | 只读取公开 RSS |

#### 数据源扩展优先级（Phase 2+）

| 优先级 | 来源 | 接入方式 | 说明 |
| --- | --- | --- | --- |
| 高 | Indie Hackers | RSS | 公开可用 |
| 高 | Dev.to | 官方 API | 有 API Key，免费 |
| 高 | Product Hunt | 官方 API | 需申请 |
| 中 | X (Twitter) | 官方 API | Basic tier $100/月，等定价合理后接入 |
| 低 | Facebook | — | Graph API 几乎关闭，ToS 风险高，暂不接入 |

> **爬虫工具**：Phase 1 三个数据源全部是 JSON API 或 RSS，不需要 HTML 解析。Phase 2 若接入无官方 API 的来源（如部分论坛），可引入 **Scrapling**，但须在接入前评估平台 ToS 合规性。

统一结构：

```python
class RawPost:
    source: str
    source_id: str
    title: str
    url: str
    author: str | None
    content: str
    score: int | None
    comments_count: int | None
    published_at: datetime
    fetched_at: datetime
```

去重规则：

- `source + source_id` 作为硬去重主键
- Phase 1 不做语义去重
- 同一 URL 重复出现时只保留首次入库记录

### 4.2 关键词预过滤模块

职责：在调用 LLM 前降低噪音和成本。

预过滤信号：

- 抱怨类关键词：`frustrated`, `pain`, `annoying`, `hard to`, `can't find`, `有没有`, `太麻烦`
- 需求类关键词：`looking for`, `need a tool`, `alternative to`, `workflow`, `automate`
- 开发者场景关键词：`API`, `SDK`, `CLI`, `deploy`, `database`, `LLM`, `monitoring`, `billing`

输出：

```python
class CandidatePost(RawPost):
    matched_keywords: list[str]
    prefilter_score: float
```

策略：

- **时间窗口强制过滤（48 小时约束）**：只处理 `published_at` 在过去 48 小时内的帖子。原因：日报的核心验证动作是"去原帖回复"，超过 48 小时的帖子社区热度消退，回应率极低，验证动作失效。
- 命中强痛点关键词直接进入 LLM
- 只命中泛关键词时，需要满足最低热度阈值（HN points ≥ 10，Reddit upvotes ≥ 20）
- 每日进入 LLM 的候选数量设置上限（建议 ≤ 50 条），避免成本失控

### 4.3 LLM 精筛模块

职责：判断帖子是否包含真实痛点，并生成结构化产品机会。

#### 7 问题框架（替代原 5 维度评分）

每条机会必须能回答以下 7 个问题，答不了的字段视为不合格，整条机会不进日报：

| # | 问题 | 目的 |
| --- | --- | --- |
| Q1 | 这是哪个**具体人群**的痛？ | 确认目标用户清晰 |
| Q2 | **原文证据**是什么？（必须引用原帖用户原话） | 防止 LLM 编造需求 |
| Q3 | 他们现在用什么**替代方案**？ | 确认痛点真实存在 |
| Q4 | 为什么**现有方案不够好**？ | 明确市场缺口 |
| Q5 | **最小可验证产品**是什么？（1-2 人，4 周内） | 确认可做性 |
| Q6 | 去哪里找**第一批用户**？（需有明确社区入口） | 确认获客可行性 |
| Q7 | **48 小时内**怎么判断继续还是放弃？ | 强制行动闭环 |

综合评分 1-5 分，基于 7 个问题的回答质量加权计算：Q2（原文证据）和 Q7（48 小时验证方案）权重最高。

LLM 输出结构：

```json
{
  "is_pain_point": true,
  "score": 4.3,
  "summary": "用户在本地 LLM 推理时缺少轻量性能分析工具。",
  "target_user": "使用 Ollama / llama.cpp 的开发者",
  "user_quote": "I just want something like py-spy but for LLM inference.",
  "product_opportunity": "构建本地 LLM 推理性能分析 CLI。",
  "existing_alternatives": ["nvtop", "手动日志分析"],
  "mvp_scope": ["实时 token/s", "显存曲线", "延迟统计"],
  "suggested_stack": ["Python", "Rich", "Ollama API"],
  "validation_actions": [
    "在原帖回复 CLI 截图，观察是否有 10 个追问",
    "发布 3 题问卷收集当前调试方式"
  ],
  "acquisition_channels": ["HN 原帖", "Reddit r/LocalLLaMA", "GitHub README SEO"],
  "risk_notes": ["本地硬件差异可能导致兼容性成本高"]
}
```

强制拒绝条件（命中任一条直接返回 `is_pain_point: false`，不进入 7 问题评估）：

```
以下情况直接拒绝：
1. 纯吐槽，无具体工具 / 流程缺口（如"X 公司服务太差了"）
2. 痛点主体不是开发者或技术人员
3. 已有明显成熟解决方案（如 VS Code 原生功能、GitHub 内置能力）
4. 内容过于宏观，无法拆解为 1-2 人可做的 MVP（如"开发者教育需要改革"）
5. 无原文用户证据（仅有作者描述，无评论区真实用户声音）
6. 无明确社区获客入口（找不到第一批用户的地方）
```

80% 准确率定义：

- 上线前人工标注 **50 条样本**（好痛点 / 一般 / 无效各若干），作为固定 benchmark
- 准确率 = LLM 判断与人工判断一致的比率
- 每次修改 prompt 后重跑 benchmark，记录版本和准确率变化
- 达到 80% 后才进入正式运营，低于 70% 必须迭代

质量控制：

- 每日随机抽查 5 条 LLM 评分结果，不合格的记录到 benchmark 扩充池
- 进入邮件的机会必须包含用户原话、产品机会、验证动作三项

### 4.4 存储模块

Phase 1 使用 **Turso**（托管 libSQL，兼容 SQLite 语法）。选择原因：GitHub Actions 每次 run 是全新容器，本地 SQLite 文件无法跨 run 持久化；Turso 提供云端持久化，API 与标准 SQLite 完全兼容，迁移成本为零。

核心表：

```sql
CREATE TABLE raw_posts (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  source TEXT NOT NULL,
  source_id TEXT NOT NULL,
  title TEXT NOT NULL,
  url TEXT NOT NULL,
  author TEXT,
  content TEXT,
  score INTEGER,
  comments_count INTEGER,
  published_at TEXT NOT NULL,
  fetched_at TEXT NOT NULL,
  UNIQUE(source, source_id)
);

CREATE TABLE analyzed_opportunities (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  raw_post_id INTEGER NOT NULL,
  is_pain_point INTEGER NOT NULL,
  opportunity_score REAL NOT NULL,
  summary TEXT NOT NULL,
  target_user TEXT,
  user_quote TEXT,
  product_opportunity TEXT,
  existing_alternatives_json TEXT,
  mvp_scope_json TEXT,
  suggested_stack_json TEXT,
  validation_actions_json TEXT,
  acquisition_channels_json TEXT,
  risk_notes_json TEXT,
  llm_provider TEXT NOT NULL,
  llm_model TEXT NOT NULL,
  prompt_version TEXT NOT NULL,
  created_at TEXT NOT NULL,
  FOREIGN KEY(raw_post_id) REFERENCES raw_posts(id)
);

CREATE TABLE email_digests (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  digest_date TEXT NOT NULL UNIQUE,
  subject TEXT NOT NULL,
  html_body TEXT NOT NULL,
  sent_at TEXT,
  status TEXT NOT NULL
);

CREATE TABLE subscribers (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  email TEXT NOT NULL UNIQUE,
  subscribed_at TEXT NOT NULL,
  unsubscribe_token TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'active'  -- active | unsubscribed
);
```

### 4.5 订阅管理模块

Phase 1 需要的最小产品层：

- **订阅入口**：静态 Landing Page（Carrd / Tally 表单），收集邮箱后写入 `subscribers` 表
- **退订接口**：每封邮件底部带 `/unsubscribe?token=xxx` 链接（法律合规要求），点击后将 `status` 改为 `unsubscribed`
- **发送逻辑**：每日 pipeline 读取所有 `status = active` 的订阅者批量发送，而不是单个 `SMTP_TO` 环境变量

> Phase 1 不自建管理后台，用 **Retool 或 Metabase** 直接连 Turso 数据库，10 分钟搭出订阅者管理界面，满足早期 50-100 用户规模的运营需求。

### 4.6 邮件生成模块

职责：将每日 Top 5 机会渲染为 HTML 邮件。

规则：

- 每日 1-3 条，质量优先于数量
- Top 1-2 展示完整分析（7 问题框架全部字段）
- 第 3 条（若有）展示紧凑格式
- 当日无合格内容时停发，在邮件中说明原因（如"今日未发现通过质量门槛的机会"）
- 分数格式统一为 `X.X / 5`
- 每条原帖链接必须带 UTM 参数（`?utm_source=digest&utm_medium=email&utm_campaign=YYYY-MM-DD`），通过我们的跳转服务记录点击，这是北极星指标的数据来源
- 验证模板包链接：Tally 问卷模板、社区回复话术、Carrd 落地页模板，Phase 1 为静态预制链接，不调用 LLM 生成
- 邮件底部固定增长区：
  - "遇到过类似问题？直接回复这封邮件告诉我们。"（回复进创始人收件箱，零开发成本 UGC）
  - "觉得有用？转发给 1 个朋友是对我们最好的支持。"
- 邮件底部记录数据源覆盖情况和生成时间

多渠道扩展规划：

| 渠道 | 计划阶段 | 说明 |
| --- | --- | --- |
| Email | Phase 1 | 核心渠道 |
| Telegram Bot | Phase 2 | 复杂度低，3-4 天可接入，作为 Phase 2 第一个功能 |
| WhatsApp | Phase 3+ | Business API 审核周期长、成本高，暂缓 |
| Slack / 飞书 Webhook | Phase 2 | 面向团队用户，参见 PRD |

邮件发送失败处理：

- SMTP 失败时重试 3 次
- 失败后写入 `email_digests.status = failed`
- GitHub Actions job 失败，方便收到告警

## 5. 配置与密钥

环境变量：

```text
APP_ENV=development
TURSO_DATABASE_URL=
TURSO_AUTH_TOKEN=

LLM_PROVIDER=deepseek
DEEPSEEK_API_KEY=
ANTHROPIC_API_KEY=

REDDIT_CLIENT_ID=
REDDIT_CLIENT_SECRET=
REDDIT_USER_AGENT=

SMTP_HOST=
SMTP_PORT=
SMTP_USERNAME=
SMTP_PASSWORD=
SMTP_FROM=
SMTP_TO=  # 仅用于开发测试，生产环境从 subscribers 表读取
```

密钥管理：

- 本地使用 `.env`
- GitHub Actions 使用 GitHub Secrets
- 不提交真实密钥
- 日志中禁止打印 API Key、SMTP 密码、OAuth Token

## 6. GitHub Actions 工作流

每日定时执行：

```text
checkout repo
setup python
install dependencies
run database migrations (Turso)
run pipeline
```

建议 cron：

```yaml
on:
  schedule:
    - cron: "0 23 * * *"
```

对应北京时间早上 7 点生成前一日摘要。

## 7. Phase 2 技术演进：机会库与验证包

Phase 2 的核心是让痛点从一次性邮件变成可持续积累的机会资产。

新增能力：

| 能力 | 技术方案 |
| --- | --- |
| Web 看板 | Next.js 或 FastAPI + HTMX，优先简单可维护 |
| 主存储 | PostgreSQL |
| 搜索 | PostgreSQL full-text search 起步 |
| 语义去重 | pgvector 或独立 Vector DB |
| 验证包生成 | LLM 根据机会结构生成 Landing Page、问卷、社区回复 |
| 用户行为 | 跟踪点击、收藏、放弃、继续验证 |

验证包内容：

```text
Landing Page 文案
等待名单表单字段
Reddit/HN/V2EX 回复草稿
用户访谈问题
MVP 功能边界
定价假设
首批获客渠道
```

## 8. Phase 3 技术演进：AI 开发辅助

Phase 3 解决“从机会到可开发任务”的断层。

新增能力：

| 能力 | 技术方案 |
| --- | --- |
| PRD 生成 | 基于机会结构生成轻量 PRD |
| 任务拆解 | 生成 MVP issue list |
| Cursor/Claude Code Prompt | 生成可直接复制到 AI IDE 的开发上下文 |
| GitHub 集成 | 创建 repo、issues、项目 README |
| MCP Server | 允许 IDE Agent 查询痛点库和机会上下文 |

MCP Server 工具示例：

```text
search_opportunities(query, min_score, source)
get_opportunity(id)
generate_mvp_brief(opportunity_id)
generate_coding_prompt(opportunity_id, stack)
```

## 9. Phase 4 技术演进：一键上线与轻运维

Phase 4 不自研云平台，而是做成熟云服务的编排层。

首选集成：

| 能力 | 推荐服务 |
| --- | --- |
| 前端/全栈部署 | Vercel |
| 数据库/Auth/Storage | Supabase |
| 代码仓库 | GitHub |
| 支付 | Stripe |
| 域名/DNS | Cloudflare |
| 监控 | Sentry + UptimeRobot |
| 分析 | Plausible / PostHog |

一键上线流程：

```text
选择机会
生成 MVP 技术模板
创建 GitHub Repo
写入环境变量模板
连接 Vercel 项目
连接 Supabase 项目
部署 preview URL
生成验证链接
开启访问分析与错误监控
```

轻运维看板：

- 部署状态
- 最近一次构建日志
- 访问量
- 注册数
- 等待名单转化率
- 错误数
- 运行成本估算

## 10. 目录结构建议

Phase 1 推荐结构：

```text
.
├── PRD.md
├── TECH_DESIGN.md
├── pyproject.toml
├── .env.example
├── src/
│   └── pain_point_digest/
│       ├── config.py
│       ├── main.py
│       ├── sources/
│       │   ├── base.py
│       │   ├── hn.py
│       │   ├── reddit.py
│       │   └── v2ex.py
│       ├── filtering/
│       │   └── keyword_filter.py
│       ├── llm/
│       │   ├── client.py
│       │   └── prompts.py
│       ├── storage/
│       │   ├── db.py
│       │   └── migrations.py
│       ├── email/
│       │   ├── renderer.py
│       │   ├── sender.py
│       │   └── templates/
│       │       └── digest.html.j2
│       └── models.py
├── tests/
│   ├── test_keyword_filter.py
│   ├── test_email_renderer.py
│   └── test_llm_schema.py
└── .github/
    └── workflows/
        └── daily-digest.yml
```

## 11. 可观测性与运营指标

### 11.1 数据采集看板（Retool / Metabase）

每个数据源的采集结果在管理页面可见，查询来源是 `raw_posts` + `analyzed_opportunities` 两张表。

**数据源概览视图**（按 source 分组，每日刷新）：

| 列 | 数据来源 | 说明 |
| --- | --- | --- |
| 数据源 | `raw_posts.source` | HN / Reddit / V2EX |
| 今日抓取数 | COUNT by source + date | 判断来源是否正常 |
| 通过预过滤数 | 有 `matched_keywords` 的记录数 | 过滤率是否合理 |
| 进入 LLM 数 | JOIN `analyzed_opportunities` | LLM 调用量 |
| 平均评分 | AVG(`opportunity_score`) by source | 各来源内容质量对比 |
| 高分机会数（≥3） | COUNT where score ≥ 3 | 有效产出 |

**单条记录详情视图**：点击任意帖子可查看：
- 原始标题、正文、URL、作者
- 命中的关键词（`matched_keywords`）
- LLM 评分和结构化输出（summary、user_quote、product_opportunity 等）
- 是否进入邮件日报

### 11.2 运营指标

Phase 1 至少记录：

| 指标 | 类型 | 目标值 | 用途 |
| --- | --- | --- | --- |
| **原帖点击率** ⭐ | 北极星 | 激活期 ≥ 20%，留存期 ≥ 25% | 核心健康度，内容价值的真实代理变量 |
| 每日抓取帖子数（按来源） | 系统健康 | — | 判断各数据源是否稳定 |
| 预过滤后候选数 | 成本控制 | — | LLM 调用量是否在预算内 |
| LLM 调用成功率 | 系统健康 | ≥ 95% | 发现 API 或 Prompt 问题 |
| 平均机会评分 | 内容质量 | — | 观察评分分布是否合理 |
| 第 4 周邮件打开率 | 辅助监控 | ≥ 25% | 参考，不作为主要决策依据 |
| 每日邮件发送状态 | 系统健康 | 100% 成功 | 确保用户收到日报 |

点击追踪实现：每条原帖链接统一走我们的跳转路由（`/r?url=...&digest=YYYY-MM-DD&pos=1`），记录点击后 302 跳转原帖，Phase 2 再做完整事件表。

## 12. 关键风险与技术应对

| 风险 | 应对 |
| --- | --- |
| LLM 输出不稳定 | 使用 JSON Schema 校验，失败自动重试或降级 |
| 邮件内容变成“创业爽文” | 强制要求用户原话、验证动作、竞品/替代方案 |
| Reddit API 限制 | 做请求限速和数据源占比控制 |
| SQLite 在 GitHub Actions 中持久化麻烦 | 直接用 Turso（托管 libSQL），兼容 SQLite API，无需等 Phase 2 迁移 |
| 一键部署范围膨胀 | Phase 4 只做 Vercel/Supabase/GitHub 编排，不自研云 |

## 13. 近期实施顺序

建议按以下顺序开发：

1. 搭建 Python 项目结构和配置系统（Turso 连接、环境变量）
2. 人工标注 50 条样本，建立 LLM 评分 benchmark（先做，后续所有 prompt 迭代都依赖它）
3. 实现 HN + V2EX 抓取，Reddit 后置
4. 实现 Turso 表结构和入库去重
5. 实现关键词预过滤
6. 实现 LLM 结构化评分（含强制拒绝条件），对照 benchmark 验证准确率 ≥ 80%
7. 实现 HTML 邮件模板（含 UTM 跳转链接、验证模板包静态链接、底部增长区）
8. 实现 SMTP 发送（读取 subscribers 表，含退订链接）
9. 接入 GitHub Actions 定时任务
10. 接入 Retool / Metabase 查看采集看板
11. 运营 7 天，观察原帖点击率，迭代 Prompt
12. 点击率达标后再决定是否进入 Phase 2

