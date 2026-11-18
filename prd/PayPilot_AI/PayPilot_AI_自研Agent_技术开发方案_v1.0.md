# PayPilot AI｜完全自研 Agent 技术开发方案

> **版本**：v1.0（面向 MVP v0.4）  
> **定位**：可直接进入研发的技术设计文档（Technical Design Document, TDD）  
> **项目**：跨境支付成功率智能诊断与优化 Agent  
> **研发周期**：4 周 / 20 个开发日（个人项目目标，不代表保证工期）  
> **原则**：**不使用 LangGraph / CrewAI / AutoGen 等 Agent 编排框架；自建 Agent Runtime、工具注册、工作流、状态管理、证据校验、RAG 检索与评测。**  
> **数据范围**：只使用自行生成的模拟交易数据和可使用的演示知识文档；不连接生产支付环境；不提供真实交易写入工具。

## 阅读说明与依据

本技术方案由以下三份已完成的项目材料推导并展开：

1. **《PayPilot_AI_自研Agent_第1至4周详细开发计划.html》**：Day 1–20 的阶段目标、文件清单、11 个 API 路由、状态机、工具协议、100 条评测集及 Go/No-Go 门槛。
2. **《PayPilot_AI_交互原型.html》**：支付总览、AI 智能诊断、诊断报告、知识库、评测中心五个界面，以及筛选、澄清、取消、追问、证据查看、反馈和演示评测等交互。
3. **《PayPilot_AI_PRD_v1.0.html》**：PSR 指标口径、支付异常场景、角色与权限、P0 范围、证据可信性、性能目标及产品验收条件。

**区分“现状”和“目标”**：现有 HTML 是纯本地交互原型；其中的 `scenarios`、`getMetrics()`、`trendSeries()`、`errorRows()`、`kbDocs`、`setInterval()` 执行进度以及评测结果属于模拟逻辑。本文的数据库、服务端接口、自研 Runtime、RAG 和真实模型调用均为**待实现方案**，不能将其写成已交付功能。

---

## 1. 项目范围、目标与非目标

### 1.1 只做一条可验收的业务闭环

输入：

> “分析德国 Klarna 最近 24 小时支付成功率下降的原因。”

目标链路：

```text
支付总览筛选
  → 创建诊断任务
  → 解析意图及参数（必要时澄清）
  → 查询支付指标及基准期
  → 按终端 / 错误码下钻
  → 检索知识库并获取真实引用
  → 校验数字与证据
  → 生成“事实 / 假设 / 建议 / 不确定性”报告
  → 用户追问 / 反馈
  → Trace 与评测结果归档
```

**MVP 通过条件**：同一演示案例在 Dashboard、后端 SQL、Agent ToolResult 与诊断报告中的统计一致，用户可以从报告查看来源，且系统不会把异常相关性说成已经确认的根因。

### 1.2 业务黄金样本（必须精确复现）

| 口径 | 样本数 | 成功数 | 失败数 | PSR |
|---|---:|---:|---:|---:|
| 德国 Klarna · 基准期 | 4,000 | 3,800 | 200 | 95.0% |
| 德国 Klarna · 当前期 | 2,000 | 1,784 | 216 | 89.2% |
| 当前期 · Android | 1,000 | 850 | 150 | 85.0% |
| 当前期 · Web | 1,000 | 934 | 66 | 93.4% |

- 总体成功率变化：`89.2% - 95.0% = -5.8pp`。
- Android 失败记录中，`redirect_timeout = 90`，占 Android 失败数 `90/150 = 60.0%`。
- `redirect_timeout` 是本项目归一化的**演示错误码**，不代表 Klarna 的官方错误码。
- 可以得出“Android 回跳链路值得优先排查”；**不能确认**“SDK 更新导致故障”。
- 黄金数据集使用固定 `dataset_version`、时钟锚点、种子以及独立的基准/当前窗口；不允许靠前端常量或 LLM 直接返回这些数字。

### 1.3 P0 / P1 / 暂不实现

| 级别 | 范围 |
|---|---|
| **P0** | 模拟数据生成与校验、PostgreSQL、4 个指标 API、自研 Runtime、Tool Registry、意图解析、受控 Workflow、基础 RAG、证据校验、REST + SSE、核心 React 页面、反馈、Trace、自动化评测 |
| **P1** | 全功能知识库管理、混合检索深度调优、多用户协作、自动异常预警、工单关联、复杂版本趋势分析 |
| **不做** | 真实支付路由切换、自动退款、风控写入、任意 SQL、Shell/网络执行型 Agent 工具、生产支付数据接入、通用多 Agent 平台 |

**实施取舍**：先做确定性单 Agent 工作流，不为“完全自建”而重复造所有数据库、Web 框架或大模型基础设施。

---

## 2. 总体技术架构

### 2.1 逻辑架构

```mermaid
flowchart TB
  FE[React + TypeScript<br/>Dashboard / Chat / Report / KB / Eval]
  API[FastAPI<br/>REST / SSE / 鉴权 / 参数校验]
  RUN[自研 Agent Runtime<br/>Orchestrator + State Machine + Executor]
  REG[Tool Registry<br/>权限 / Schema / 超时 / 审计]
  LLM[LLM Adapter<br/>意图识别 / 结构化生成]
  EV[Evidence Validator<br/>数值 / 引用 / 不确定性]
  DB[(PostgreSQL<br/>支付尝试 / 任务 / 证据 / 事件)]
  RAG[自建 RAG<br/>入库 / 分块 / 召回 / 排序]
  KB[(演示知识文档<br/>chunks / metadata)]
  TRACE[事件持久化 + Trace]
  EVAL[离线 Eval Runner<br/>数据集 / 自动评分 / 人工复核]

  FE --> API
  API --> RUN
  RUN --> REG
  RUN --> LLM
  RUN --> EV
  REG --> DB
  REG --> RAG
  RAG --> KB
  RUN --> TRACE
  TRACE --> DB
  API --> DB
  EVAL --> RUN
  API -->|SSE| FE
```

### 2.2 技术选型与理由

| 层级 | 建议技术 | 为什么这样选 | 是否自研 |
|---|---|---|---|
| 前端 | React、TypeScript、Vite、React Router、TanStack Query | 保留现有原型交互，清晰分离服务端状态与组件状态 | 业务 UI 自研 |
| 后端 | Python、FastAPI、Pydantic、SQLAlchemy、Alembic | 快速提供类型化接口、数据查询和异步任务入口 | 业务 API 自研 |
| 数据库 | PostgreSQL | 保存模拟交易、Agent 任务、事件、证据与反馈 | 不重新造数据库 |
| Agent | 纯 Python + 自定义协议与状态机 | 可解释、可控、可复现，无 Agent 编排框架依赖 | **核心自研** |
| 模型 | 可配置的 LLM API / SDK | 只承担语义理解、工具参数建议与文本组织 | 适配层自研 |
| RAG | 自研 ingest/retriever/ranker，起步 PostgreSQL 全文检索；可选 pgvector | 先验证错误码精准检索，再决定是否引入语义召回 | **流程自研** |
| 流式进度 | Server-Sent Events（SSE） | 单向服务端事件适合任务进度、工具状态和报告完成通知 | 协议与持久化自研 |
| 测试 | pytest、httpx、Playwright、TypeScript 类型检查 | 覆盖数据、状态机、接口及端到端交互 | 测试用例自研 |
| 本地部署 | Docker Compose | 一键复现实验环境 | 使用现成容器工具 |

不锁定具体第三方包的最新版本；第一次安装时生成并提交锁文件，**整个评测周期冻结模型 ID、依赖版本、Prompt 和数据版本**。

### 2.3 六个不可违反的设计约束

1. **所有支付指标由确定性查询计算**，LLM 不能成为数值计算的唯一来源。
2. **数据库读写分权**：支付分析工具仅能连接只读账户；任务/事件持久化通过独立服务账户处理。
3. **工具能力白名单**：模型不能调用未注册工具，不接受自由 SQL 或支付交易写操作。
4. **证据优先**：报告中的每项关键统计必须指向已保存的 evidence；知识引用必须对应真实 chunk。
5. **取消与恢复有明确语义**：取消后停止新工具调用；网络重连不重新启动任务。
6. **所有演示输出标识 DEMO**，不将离线评测提升为真实支付成功率改善声明。

---

## 3. 代码仓库结构与模块边界

建议采用单仓库，后端保持按层组织，前端按功能组织：

```text
paypilot-ai/
├── README.md
├── .env.example
├── .gitignore
├── infra/
│   └── compose.yaml
├── backend/
│   ├── pyproject.toml
│   ├── alembic.ini
│   ├── migrations/
│   └── app/
│       ├── main.py
│       ├── core/
│       │   ├── config.py
│       │   ├── security.py
│       │   └── logging.py
│       ├── api/
│       │   ├── health.py
│       │   ├── metrics.py
│       │   ├── diagnoses.py
│       │   └── diagnosis_events.py
│       ├── schemas/
│       │   ├── common.py
│       │   ├── metrics.py
│       │   ├── diagnoses.py
│       │   └── events.py
│       ├── db/
│       │   ├── session.py
│       │   ├── models.py
│       │   └── agent_models.py
│       ├── services/
│       │   ├── metric_service.py
│       │   ├── diagnosis_service.py
│       │   └── run_service.py
│       ├── agent/
│       │   ├── contracts.py
│       │   ├── state_machine.py
│       │   ├── runtime.py
│       │   ├── executor.py
│       │   ├── diagnosis_workflow.py
│       │   ├── intent_parser.py
│       │   ├── context_store.py
│       │   ├── trace.py
│       │   ├── evidence.py
│       │   ├── report_builder.py
│       │   ├── report_validator.py
│       │   ├── guardrails.py
│       │   ├── tools/
│       │   │   ├── base.py
│       │   │   ├── registry.py
│       │   │   └── payment_tools.py
│       │   └── llm/
│       │       ├── base.py
│       │       └── provider_adapter.py
│       └── rag/
│           ├── ingest.py
│           ├── chunker.py
│           ├── retriever.py
│           └── ranker.py
├── frontend/
│   ├── package.json
│   └── src/
│       ├── app/
│       ├── pages/
│       │   ├── Dashboard.tsx
│       │   ├── Diagnosis.tsx
│       │   ├── Report.tsx
│       │   ├── Knowledge.tsx
│       │   └── Evaluation.tsx
│       ├── components/
│       │   ├── FilterBar.tsx
│       │   ├── MetricsCards.tsx
│       │   ├── DiagnosisTimeline.tsx
│       │   ├── EvidencePanel.tsx
│       │   ├── ReportSections.tsx
│       │   └── FeedbackForm.tsx
│       ├── api/
│       │   ├── client.ts
│       │   ├── metrics.ts
│       │   └── diagnoses.ts
│       ├── hooks/
│       │   └── useDiagnosisEvents.ts
│       └── types/
├── scripts/
│   └── seed_payments.py
├── data/
│   ├── seed_manifest.json
│   └── kb/
├── evaluation/
│   ├── datasets/
│   ├── rubrics/
│   ├── scorers/
│   ├── reviews/
│   ├── results/
│   ├── badcases/
│   ├── reports/
│   └── runner.py
├── research/
├── portfolio/
├── tests/
│   ├── week01/
│   ├── week02/
│   ├── week03/
│   ├── e2e/
│   ├── evaluation/
│   └── regression/
└── docs/
    ├── metrics-definition.md
    ├── data-dictionary.md
    ├── tool-contract.md
    ├── report-schema.md
    ├── agent-architecture.md
    ├── evaluation-protocol-v1.md
    └── decision-records/
```

### 3.1 依赖方向

```text
API 层 → Application Services → Agent Runtime / Metric Service
Agent Runtime → Tool 接口 / LLM 接口 / Evidence 接口
Tool 实现 → Metric Service / RAG Service
存储实现 → PostgreSQL
Frontend → HTTP/SSE；不直接知道 Agent 内部模块
```

禁止让 `runtime.py` 直接拼接 SQL，禁止让 React 直接访问数据库，禁止让 `report_builder.py` 自行重新计算交易指标。

---

## 4. 第 1 周：确定性支付数据服务

### 4.1 统一业务数据口径

**统计单元**：`payment_attempt_id`，而非 `order_id`。同一订单可能重试多次；同一次支付尝试有多条状态事件时应形成一条规范化最终记录。

**MVP PSR**：

```text
valid_attempts = count(status in [SUCCESS, FAILED])
successful_attempts = count(status == SUCCESS)
failed_attempts = count(status == FAILED)
PSR = successful_attempts / valid_attempts
Delta_pp = (current_PSR - baseline_PSR) * 100
```

`PENDING` 与 `CANCELLED` 单独显示，不进入上述 PSR 分母。分母为 0 时 `psr=null`，不把它渲染成 `0%`。

**时间语义**：

- 数据库存 UTC；接口使用带时区的 ISO 8601 时间。
- 统计仅使用成熟观察窗口的数据，成熟阈值可配置（例如 `finalized_at <= as_of_utc - maturity_lag`）；最终阈值需按业务假设记录。
- 当前窗口、基准窗口应写入返回字段，不能只写“最近 24 小时”。
- 演示回放使用固定 `as_of_utc`，不使用机器当前时间生成黄金案例。
- 基准使用预先定义的“可比窗口组合”；黄金 fixture 中基准总数为 4,000，不能未经定义就解释为单日基准，也不能把不同时间跨度的**交易量**直接比较。

### 4.2 数据表设计

**核心表与责任**：

| 表名 | 关键字段 | 用途 |
|---|---|---|
| `payment_attempts` | `payment_attempt_id`, `country_code`, `payment_method`, `channel`, `app_version`, `status`, `error_code`, `finalized_at`, `dataset_version` | 指标查询事实表 |
| `release_events` | `release_id`, `app_version`, `channel`, `released_at`, `description`, `dataset_version` | 版本线索，不等于故障因果证据 |
| `diagnosis_tasks` | `task_id`, `user_query`, `filters`, `status`, `dataset_version` | 产品级诊断任务 |
| `agent_runs` | `run_id`, `task_id`, `state`, `workflow_version`, `model_id`, `step_count`, `started_at`, `ended_at` | 一次 Agent 运行，可重试生成新 run |
| `agent_steps` | `step_id`, `run_id`, `sequence`, `node`, `status`, `started_at`, `ended_at` | 节点执行记录与恢复检查点 |
| `agent_events` | `run_id`, `sequence`, `event_type`, `payload`, `created_at` | SSE 可重放事件 |
| `tool_invocations` | `invocation_id`, `run_id`, `tool_name`, `input_digest`, `result_status`, `duration_ms`, `evidence_id` | 工具调用审计 |
| `diagnosis_evidence` | `evidence_id`, `run_id`, `kind`, `source_key`, `filters`, `payload`, `data_version` | 数值与文档的可追溯证据 |
| `diagnosis_reports` | `report_id`, `run_id`, `schema_version`, `report_json`, `created_at` | 固定结构的诊断报告 |
| `diagnosis_feedback` | `feedback_id`, `run_id`, `report_id`, `rating`, `issue_tags`, `note` | 用户反馈 |
| `knowledge_documents` | `document_id`, `title`, `source`, `updated_at`, `scope`, `version`, `is_active` | 知识源元数据 |
| `knowledge_chunks` | `chunk_id`, `document_id`, `chunk_order`, `content`, `location`, `metadata` | RAG 召回、引用与版本管理 |

**支付事实表 DDL 示例**（其余表用 SQLAlchemy + Alembic 迁移实现）：

```sql
CREATE TABLE payment_attempts (
  payment_attempt_id TEXT PRIMARY KEY,
  order_id_hash TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL,
  finalized_at TIMESTAMPTZ,
  country_code CHAR(2) NOT NULL,
  payment_method TEXT NOT NULL,
  channel TEXT NOT NULL CHECK (channel IN ('Web','Android','iOS')),
  app_version TEXT,
  status TEXT NOT NULL CHECK (status IN ('SUCCESS','FAILED','PENDING','CANCELLED')),
  failure_stage TEXT,
  error_code TEXT,
  provider_error_code TEXT,
  amount_bucket TEXT,
  dataset_version TEXT NOT NULL,
  seed_batch_id TEXT NOT NULL,
  environment TEXT NOT NULL DEFAULT 'DEMO'
    CHECK (environment = 'DEMO'),
  CHECK (status NOT IN ('SUCCESS','FAILED') OR finalized_at IS NOT NULL)
);

CREATE INDEX idx_attempts_metric
  ON payment_attempts (dataset_version, country_code, payment_method, finalized_at);
CREATE INDEX idx_attempts_channel_error
  ON payment_attempts (dataset_version, channel, error_code, finalized_at);
```

**运行态数据存储约束**：

- `agent_events` 使用 `(run_id, sequence)` 唯一约束；每次状态改变与对应事件在同一事务中落库。
- `agent_steps` 使用 `(run_id, sequence)` 唯一约束；`tool_invocations` 记录幂等键，避免恢复时重复执行副作用型步骤（当前工具只读）。
- `diagnosis_evidence` 的 `payload` 保存经过校验的事实快照、查询条件及来源标识。报告只引用已存在的 evidence ID。
- `diagnosis_reports` 保存结构化 JSON 与 schema 版本；后续编辑或重新诊断不得静默覆盖旧报告。
- `knowledge_chunks` 必须可回溯到文档及原始位置；失效文档不参与默认检索。

### 4.3 黄金数据生成策略

**先锁定黄金样本，再填充其他数据**，否则随机数据可能污染预期分母。

```text
seed_payments.py --seed 20261116 --count 30000 \
  --anchor-date 2026-11-16T10:00:00Z --reset

阶段 A：删除/隔离指定 DEMO batch（reset 有显式确认）
阶段 B：写入黄金基准 4,000 次和当前 2,000 次支付尝试
阶段 C：按 Android 1,000/850、Web 1,000/934 细分当前 2,000 条
阶段 D：Android 当前失败中写入 redirect_timeout 90 条
阶段 E：填充其余 24,000 条非黄金维度记录
阶段 F：写入 release_events、seed_manifest.json
阶段 G：运行黄金 SQL 与一致性断言
```

`seed_manifest.json` 至少保存 `seed`、`anchor_at_utc`、`dataset_version`、`golden_scope`、`golden_windows`、`total_rows`、`expected_metrics`、`sha256`。

**避免数据污染**：额外生成的数据不可落到黄金统计范围（地区、支付方式、时间组合）内，除非该条记录是 fixture 的明确组成部分。

### 4.4 指标服务与 SQL

所有指标函数接受规范化 `QueryScope`：

```json
{
  "dataset_version": "demo-v1",
  "country_code": "DE",
  "payment_method": "Klarna",
  "channel": null,
  "start_at": "<ISO_UTC_START>",
  "end_at": "<ISO_UTC_END>",
  "as_of_utc": "2026-11-16T10:00:00Z"
}
```

**只读服务端查询**（示意，实际应使用绑定参数）：

```sql
SELECT
  COUNT(*) FILTER (WHERE status IN ('SUCCESS','FAILED')) AS valid_attempts,
  COUNT(*) FILTER (WHERE status = 'SUCCESS') AS successful_attempts,
  COUNT(*) FILTER (WHERE status = 'FAILED') AS failed_attempts
FROM payment_attempts
WHERE dataset_version = :dataset_version
  AND country_code = :country_code
  AND payment_method = :payment_method
  AND status IN ('SUCCESS', 'FAILED')
  AND finalized_at >= :start_at
  AND finalized_at < :end_at
  AND finalized_at <= :mature_until;
```

> 上述 SQL 专门统计**成熟终态的有效支付尝试**。`PENDING` 通常没有 `finalized_at`，应另行通过 `created_at` 和快照时点统计；`CANCELLED` 也单独查询。页面展示的待决/取消比例需要明确其分母与队列归属窗口，不能用本终态查询假装完整交易状态分布。`mature_until` 由 `as_of_utc - maturity_lag` 计算，并通过参数绑定传入。

实现原则：

1. 指标查询统一使用参数绑定，动态维度来自白名单枚举，不字符串拼接任意 SQL。
2. 复杂查询由 `metric_service.py` 统一生成；REST 与 Agent Tool 调用相同服务函数。
3. 后端以 Decimal 或有理数进行关键精度处理，存储/输出比例（`0.892`），展示时才转换为百分比（`89.2%`）。
4. API 同时返回 `valid_attempts`、`successful_attempts`、`failed_attempts`、`psr`、`data_version`、`query_window`、`as_of_utc` 和数据质量标记。
5. 分组结果必须带分母，禁止只返回百分比导致误判低样本组。

### 4.5 第 1 周四个指标 API

| 方法 | 路径 | 关键参数 | 响应核心字段 |
|---|---|---|---|
| GET | `/api/metrics` | `country_code`, `payment_method`, `channel`, `period`, `as_of` | `valid_attempts`, `successful_attempts`, `failed_attempts`, `psr` |
| GET | `/api/metrics/compare` | 当前和基准窗口、筛选器 | `current`, `baseline`, `delta_pp`, `quality_flags` |
| GET | `/api/metrics/breakdown` | `dimension=channel/app_version/payment_method/country_code` | `rows[]`、各行样本量、PSR、变化 |
| GET | `/api/metrics/errors` | 筛选器、可选 `channel` | `error_code`, `failed_attempts`, `share_of_failures` |

错误响应统一为：

```json
{
  "error": {
    "code": "INVALID_DIMENSION",
    "message": "该分组维度暂不受支持",
    "request_id": "req_example",
    "retryable": false
  }
}
```

**第 1 周验收**：针对黄金数据写独立 SQL 基准测试；Swagger 中四个 API 与预期一致；空分母、非法维度、未成熟数据、重复 ID、只读权限均有测试。

---

## 5. 第 2 周：从零实现 Agent Runtime

### 5.1 自研模块与接口

| 模块 | 文件 | 职责 | 明确不负责 |
|---|---|---|---|
| `AgentState / Event` | `agent/contracts.py` | 运行态、结构化事件、错误契约 | 直接执行 SQL |
| `StateMachine` | `agent/state_machine.py` | 合法状态转移与守卫 | 调用 LLM |
| `Orchestrator` | `agent/runtime.py` | 调度、执行预算、生命周期、恢复 | 自行计算 PSR |
| `NodeExecutor` | `agent/executor.py` | 执行单节点、超时、取消检查 | 决定业务统计口径 |
| `ToolRegistry` | `agent/tools/registry.py` | 工具注册、权限及输入验证 | 放开任意工具 |
| `LLMAdapter` | `agent/llm/` | 模型输入输出适配、结构化响应 | 直接持久化数据库 |
| `ContextStore` | `agent/context_store.py` | 会话范围、已验证事实 ID | 不受控无限拼聊天历史 |
| `TraceRecorder` | `agent/trace.py` | 状态、耗时、工具记录与审计 | 把私密密钥写入日志 |
| `Guardrails` | `agent/guardrails.py` | 工具权限、安全输入输出验证 | 取代后端权限体系 |

### 5.2 有限状态机

```mermaid
stateDiagram-v2
  [*] --> CREATED
  CREATED --> PARSING
  PARSING --> CLARIFYING: 缺少关键范围
  CLARIFYING --> PARSING: 用户补全
  PARSING --> QUERYING: 参数确认
  QUERYING --> ANALYZING: 数据有效
  QUERYING --> INSUFFICIENT_DATA: 数据不足
  ANALYZING --> VALIDATING: Week 2 初步报告
  ANALYZING --> RETRIEVING: Week 3 知识检索
  RETRIEVING --> VALIDATING
  VALIDATING --> COMPLETED: 通过
  VALIDATING --> FAILED: 无法修复
  CREATED --> CANCELLED
  PARSING --> CANCELLED
  QUERYING --> CANCELLED
  ANALYZING --> CANCELLED
  RETRIEVING --> CANCELLED
  CLARIFYING --> CANCELLED
  COMPLETED --> [*]
  FAILED --> [*]
  CANCELLED --> [*]
  INSUFFICIENT_DATA --> [*]
```

Week 3 的证据整合可以实现为 `RETRIEVING → (内部 evidence synthesis 节点) → VALIDATING`。如果将其升级为独立状态 `SYNTHESIZING`，需要同步更新数据库状态枚举、前端映射、事件模型和转移测试；这属于技术细化，而非原型现有行为。

核心规则：

- `COMPLETED`、`FAILED`、`CANCELLED`、`INSUFFICIENT_DATA` 是本次 run 的终态，不能继续执行；重试应新建 run 并关联旧 run。
- `CLARIFYING` 为等待用户输入状态，不消耗模型执行预算；用户补充后重新解析并保留原 task 关联。
- 取消标志优先于下一节点启动。正在进行的外部模型请求可尽力中止；无法中止时收到结果也不得再推进被取消的 run。
- 状态与事件序号应在数据库事务内写入；不能只靠内存状态。

### 5.3 核心状态对象

```python
from dataclasses import dataclass, field
from enum import StrEnum
from typing import Any

class RunState(StrEnum):
    CREATED = "CREATED"
    PARSING = "PARSING"
    CLARIFYING = "CLARIFYING"
    QUERYING = "QUERYING"
    ANALYZING = "ANALYZING"
    RETRIEVING = "RETRIEVING"
    VALIDATING = "VALIDATING"
    COMPLETED = "COMPLETED"
    INSUFFICIENT_DATA = "INSUFFICIENT_DATA"
    FAILED = "FAILED"
    CANCELLED = "CANCELLED"

@dataclass
class AgentState:
    task_id: str
    run_id: str
    state: RunState = RunState.CREATED
    intent: str | None = None
    filters: dict[str, Any] = field(default_factory=dict)
    evidence_ids: list[str] = field(default_factory=list)
    tool_invocation_ids: list[str] = field(default_factory=list)
    step_count: int = 0
    workflow_version: str = "workflow-v0.2"
    model_id: str = "configured-provider-model"
    last_error_code: str | None = None
    cancel_requested: bool = False
```

真实实现优先使用 Pydantic 数据结构和持久化模型；这里用于明确数据契约，不是完整运行时代码。

### 5.4 Orchestrator 执行策略

```text
1. create_task(query, filters, idempotency_key)
2. persist task + run in CREATED
3. execute PARSING; if critical fields missing → CLARIFYING
4. validate scope + permission
5. QUERYING: call get_payment_metrics / compare_payment_periods
6. ANALYZING: decide whether drill-down is required
7. call get_failure_breakdown / get_error_distribution (bounded)
8. Week 2: generate preliminary report (no RAG)
9. Week 3: RETRIEVING + evidence synthesis
10. VALIDATING: compare facts and citations against persisted evidence
11. persist report + terminal state; publish report.ready
```

**默认执行预算（可配置的演示值）**：`max_steps=8`、`tool_timeout=10s`、`run_timeout=30s`、`structured_output_repair=1`。预算超限时返回可解释的失败状态和已完成证据，不能静默无限重试。第三周的实际延迟若超出预算，应按测试数据调整预算与服务级目标，并记录取舍。

恢复规则：只从持久化的**已完成节点与事件**恢复，检查节点幂等性，禁止在刷新页面时重复创建新 run。

### 5.5 Tool Registry 接口及代码示例

```python
from typing import Protocol, Generic, TypeVar
from pydantic import BaseModel

ArgsT = TypeVar("ArgsT", bound=BaseModel)

class ToolContext(BaseModel):
    run_id: str
    dataset_version: str
    role: str = "ANALYST"
    timeout_seconds: int = 10

class ToolResult(BaseModel):
    status: str
    data: dict
    evidence_id: str | None = None
    data_version: str | None = None
    filters: dict = {}
    errors: list[dict] = []

class Tool(Protocol[ArgsT]):
    name: str
    input_schema: type[ArgsT]

    async def execute(self, args: ArgsT, context: ToolContext) -> ToolResult:
        ...
```

> 注：为便于阅读，示例简化了泛型与验证。实际项目中 Pydantic 字段应优先用 `Field(default_factory=dict)`、`Field(default_factory=list)`，避免共享可变默认值；Registry 需持有工具元数据（权限、超时、返回 Schema、版本）。

Registry 支持：`register()`、`list_allowed()`、`validate_arguments()`、`invoke()`。在调用前执行白名单检查、严格参数校验、身份与角色校验、时间范围限制、预算检查，执行后校验 ToolResult、保存 evidence 和 Trace。

**第一批工具**：

| 工具名 | 功能 | 复用服务 | 权限 |
|---|---|---|---|
| `get_payment_metrics` | 当前周期指标 | `metric_service.get_metrics` | READ_ONLY |
| `compare_payment_periods` | 当前/基准对比 | `metric_service.compare` | READ_ONLY |
| `get_failure_breakdown` | 终端、版本等维度下钻 | `metric_service.breakdown` | READ_ONLY |
| `get_error_distribution` | 错误码与失败样本占比 | `metric_service.errors` | READ_ONLY |
| `get_release_changes` | 查询模拟版本发布事件 | `release_service.list_changes` | READ_ONLY（Week 3 或后续） |
| `search_payment_kb` | 知识库检索 | `rag.retriever.search` | READ_ONLY（Week 3） |

禁止注册任意 SQL、任意 URL 请求、系统命令或支付写入工具；**禁止仅凭 Prompt 限制权限**。

ToolResult 示例：

```json
{
  "status": "success",
  "data": {
    "valid_attempts": 2000,
    "successful_attempts": 1784,
    "failed_attempts": 216,
    "psr": 0.892
  },
  "evidence_id": "ev_payment_001",
  "data_version": "demo-v1",
  "filters": {"country_code": "DE", "payment_method": "Klarna"},
  "errors": []
}
```

### 5.6 LLM Adapter 与结构化输出

自建适配器方法：

```python
class LLMClient(Protocol):
    async def parse_intent(self, user_text: str, context: dict) -> dict: ...
    async def propose_tool_call(self, allowed_tools: list[dict], context: dict) -> dict: ...
    async def generate_report_draft(self, facts: dict, references: list[dict]) -> dict: ...
```

接口调用链：

```text
用户文本 → 规范化意图 JSON（Pydantic 校验）
         → 固定工作流判断需要的只读工具
         → LLM 可提出工具调用建议，但 Registry 执行最终授权和参数校验
         → 模型生成文字性判断 / 原因假设
         → 服务端覆盖与校验所有确定性数值、来源 ID
```

**意图 Schema**建议至少包含：`intent`、`country_code`、`payment_method`、`period`、`channel?`、`app_version?`、`missing_fields[]`、`requires_clarification`。

**失败与防护**：无效 JSON 最多修复一次；用户提示或知识文档不能改变工具许可；模型不得伪造知识引用 ID；对“自动退款 / 切换支付渠道 / 修改风控”返回能力边界拒绝说明。

### 5.7 Day 10 最小闭环

第二周**不允许假装已接入 RAG**。正确结果应是：

- Agent 查询数据库，得到 89.2%、95.0%、-5.8pp 及 Android 85.0%。
- 提示 Android 为异常集中方向，呈现数据证据。
- 报告写明“当前仅完成数据初步诊断；知识检索未接入；根因未确认”。
- 支持创建任务、查看状态、必要时澄清参数，并保留 20 条快速回归测试。

---

## 6. 第 3 周：自建 RAG、完整诊断、SSE 与前端接入

### 6.1 RAG 实施方案

**知识来源**：延续原型中的 5 类演示知识：Android Deep Link / WebView 回跳、`redirect_timeout` 归一化含义、PSR 口径、版本发布相关性、支付失败原因分类。原型的 `KB-001` 至 `KB-005` 可用作第一批**自编模拟文档**，必须保留“非渠道官方规范”的来源标识。

**文档处理链**：

```text
data/kb/*.md
  → 读取与校验许可证/来源
  → 文档元数据（document_id、title、version、scope、updated_at）
  → 标题/段落切分（chunk_id、位置、parent_doc_id）
  → PostgreSQL 存储与全文索引
  → 错误码关键词检索 + 文档范围过滤
  → 可选语义向量检索与融合排序
  → 检索结果携带 source_id/chunk_id/版本/相关性
```

**检索决策**：

1. v0.3 先确保 `redirect_timeout`、`WebView`、`PSR` 等关键词精确命中，错误码 exact match 提升权重。
2. 数据量小的演示库可先使用 PostgreSQL 全文检索/关键词索引；语义向量和 pgvector 只在误召回/漏召回评测证明有收益时增加。
3. `payment_method`、平台、文档版本和更新时间作为过滤条件；不匹配的文档不可被当作权威依据。
4. 空召回返回 `NO_EVIDENCE`；冲突文档返回多来源与冲突标记，不编造不存在的出处。
5. 检索文本属于**不可信内容**，不得含有可执行工具指令；只作为引用材料进入生成上下文。

**RAG 最小测试**：10 条以上，覆盖有命中、无命中、过期、版本冲突、恶意文档、错误码同名不同支付方式。测试里应核对真实存在的 `chunk_id` 和引用定位。

### 6.2 结构化诊断报告与证据校验

服务端最终报告 Schema 建议：

```json
{
  "schema_version": "report-v1",
  "report_id": "report_001",
  "run_id": "run_001",
  "dataset_version": "demo-v1",
  "environment": "DEMO",
  "status": "HYPOTHESIS_ONLY",
  "scope": {
    "country_code": "DE",
    "payment_method": "Klarna",
    "period": "24h"
  },
  "metrics": {
    "valid_attempts": 2000,
    "successful_attempts": 1784,
    "failed_attempts": 216,
    "current_psr": 0.892,
    "baseline_psr": 0.95,
    "delta_pp": -5.8,
    "evidence_ids": ["ev_payment_001", "ev_compare_001"]
  },
  "findings": [
    {
      "type": "OBSERVATION",
      "text": "Android 当前成功率为 85.0%，低于 Web 的 93.4%",
      "evidence_ids": ["ev_breakdown_001"]
    }
  ],
  "hypotheses": [
    {
      "text": "Android 回跳链路值得优先排查",
      "confidence_label": "NEEDS_VERIFICATION",
      "evidence_ids": ["ev_error_001", "kb_chunk_001"]
    }
  ],
  "citations": ["kb_chunk_001", "kb_chunk_002"],
  "actions": ["对比基准错误码分布", "核查回跳与终态同步"],
  "uncertainties": ["不能仅凭相关性确认具体 SDK 版本为故障根因"]
}
```

上面 JSON 是**目标响应示例**，其中 `evidence_id`/`chunk_id` 在真实系统中必须由持久化模块生成，不能硬编码。数值来自 Golden fixture，文本为未确认的诊断假设。

**严格校验器**：

- `metrics` 数值只能从已保存的 ToolResult / EvidenceProjection 填充，不接受模型自报覆盖。
- 报告引用的 evidence/chunk ID 必须属于本 run 的已检索集合，且数据窗口一致。
- 事实与假设分字段表达；没有独立验证不得输出 `CONFIRMED_ROOT_CAUSE`。
- 检查 `data_version`、`as_of_utc` 与展示数据一致，避免跨运行引用旧证据。
- 输出模型不能修复错误引用时，回退到“数据事实 + 证据不足”报告，保留失败原因。

### 6.3 诊断任务 REST API（完整 11 个）

| 方法 | 路径 | 责任 | 周次 |
|---|---|---|---|
| GET | `/api/health` | 服务健康及数据库可用性 | W1 |
| GET | `/api/metrics` | 支付指标 | W1 |
| GET | `/api/metrics/compare` | 基准比较 | W1 |
| GET | `/api/metrics/breakdown` | 分组查询 | W1 |
| GET | `/api/metrics/errors` | 错误分布 | W1 |
| POST | `/api/diagnoses` | 创建任务与 run；幂等键 | W2 |
| GET | `/api/diagnoses/{id}` | 查询状态、可用步骤、最终报告 | W2 |
| GET | `/api/diagnoses/{id}/events` | SSE 增量事件、重连恢复 | W3 |
| POST | `/api/diagnoses/{id}/messages` | 追问与澄清回复 | W3 |
| POST | `/api/diagnoses/{id}/cancel` | 请求取消 | W3 |
| POST | `/api/diagnoses/{id}/feedback` | 提交报告反馈 | W3 |

**创建任务请求**：

```http
POST /api/diagnoses
Idempotency-Key: demo-create-001
Content-Type: application/json

{
  "question": "分析德国 Klarna 最近 24 小时支付成功率下降的原因",
  "filters": {
    "country_code": "DE",
    "payment_method": "Klarna",
    "period": "24h"
  },
  "dataset_version": "demo-v1"
}
```

**创建任务响应（202 Accepted）**：

```json
{
  "task_id": "task_001",
  "run_id": "run_001",
  "state": "CREATED",
  "events_url": "/api/diagnoses/task_001/events",
  "status_url": "/api/diagnoses/task_001"
}
```

**统一错误语义**：

- `400`：不合法输入；`401/403`：身份/权限问题；`404`：任务不存在或无权查看；`409`：状态冲突；`422`：Schema 不符合；`429`：超出使用预算；`500/503`：服务不可用。
- 业务性证据不足不一定是 HTTP 错误，可使用 `INSUFFICIENT_DATA` 任务终态。
- 所有创建/取消操作需要考虑并发与重复请求。

### 6.4 SSE 事件协议与断线恢复

**消息格式**：

```text
event: run.started
id: run_001:1
data: {"run_id":"run_001","task_id":"task_001","state":"PARSING"}

event: step.started
id: run_001:2
data: {"run_id":"run_001","step":"QUERYING","message":"正在读取支付数据"}

event: tool.completed
id: run_001:3
data: {"run_id":"run_001","tool":"get_payment_metrics","evidence_id":"ev_payment_001"}

event: report.ready
id: run_001:8
data: {"run_id":"run_001","report_id":"report_001","state":"COMPLETED"}
```

**Event types**：`run.started`、`step.started`、`tool.completed`、`step.completed`、`clarification.required`、`report.ready`、`run.failed`、`run.cancelled`、`run.insufficient_data`。

**恢复语义**：

1. 先落库 `agent_events`，再向订阅者广播。
2. 浏览器记录最后消费的 `event_id`；重连时使用 SSE 的 `Last-Event-ID`（或在客户端无法配置时使用明确的 `after` 查询参数）。
3. 服务端根据 `run_id + sequence` 重放未接收事件，再继续推送新事件；已处理 ID 在前端去重。
4. 如果服务端无法保留完整历史，客户端调用 `GET /api/diagnoses/{id}` 获取权威状态快照。
5. 多标签页订阅同一任务只创建多个**监听连接**，不得触发多次执行。
6. SSE 输出本身不能暴露敏感内部 Prompt 或密钥；前端只展示友好的步骤状态与证据摘要。

**部署限制**：演示版可单进程运行并轮询数据库/使用进程内唤醒；若未来扩展多进程或多实例，需要独立可靠队列/事件分发，不能假设进程内内存事件可以跨实例传递。

### 6.5 前端：将原型逐项替换为真实能力

原型当前是单文件 HTML + JavaScript；需要拆为 React 组件。**先沿用视觉结构，不在接 API 时大规模重做 UI**。

| 原型现有行为 | 目标组件/状态 | 后端接入点 | 替换要求 |
|---|---|---|---|
| `scenarios` / `getMetrics()` | `Dashboard`、`MetricsCards` | `/api/metrics`、`/compare` | 取消前端统计常量；统一 PSR 口径 |
| `trendSeries()`、`errorRows()` | 趋势图、错误分布组件 | 指标/分布查询 | 不再伪造趋势；实际返回窗口与样本量 |
| 国家、支付方式、终端、时段筛选 | `FilterBar`、URL Query | `/api/metrics*` | 筛选改变后缓存键和图表同步更新 |
| `sendQuery()` 内正则与模拟返回 | `Diagnosis` + `DiagnosisComposer` | POST `/api/diagnoses` | 后端意图识别 / 受控澄清 |
| `setInterval()` 六步模拟进度 | `DiagnosisTimeline` | SSE `/events` | 映射真实事件，禁止固定时间假进度 |
| 原型“停止执行” | `CancelButton` | POST `/cancel` | 显示取消中、已取消、已终止等状态 |
| 原型“继续追问” | 任务会话 | POST `/messages` | 显式覆盖筛选条件，避免串任务上下文 |
| 原型报告和固定文本导出 | `Report`、`EvidencePanel` | GET `/diagnoses/{id}` | 数值与引用来自服务端结构化报告 |
| 原型 `kbDocs`（KB-001～005） | `Knowledge` | 后端 RAG / 文档只读查询 | 保留来源、版本及演示标识 |
| 原型 `localStorage` 反馈 | `FeedbackForm` | POST `/feedback` | 反馈落库并关联 run/report |
| 原型“运行评测”展示脚本结果 | `Evaluation` | Week 4 评测报告数据 | 明示真实评测 / 尚未运行，不伪装为实测 |

前端状态建议：

- **服务端状态**：查询缓存（指标、任务、报告、反馈）；通过 TanStack Query 或类似方案管理。
- **本地 UI 状态**：当前 Tab、筛选下拉、输入草稿、证据展开、移动端侧栏。
- **任务状态**：由 `task_id` + SSE 与 GET 状态快照组成；不把 `RUNNING` 当成唯一真相。
- **URL 可恢复性**：支持 `/dashboard?country=DE&method=Klarna&period=24h`、`/diagnoses/:taskId`、`/reports/:reportId`。
- **显示约束**：无数据、执行中、证据不足、失败、取消、无知识命中、弱网重连均有单独状态。

**页面验收**：从 Dashboard 携带筛选条件发起诊断，正确看到逐步事件、证据、报告、追问及反馈；刷新页面可恢复任务；手机布局仍可操作。

---

## 7. 第 4 周：评测、Bad Case、人工验证与结项

### 7.1 评测数据集构成

| 类型 | 数量 | 重点 |
|---|---:|---|
| 指标查询 | 20 | PSR、分母、百分点、时间范围 |
| 异常诊断 | 25 | 异常分类、因果过度归因、证据链 |
| 多维度比较 | 20 | Web/Android/版本与错误分布一致性 |
| 知识问答 | 15 | 文档命中、引用正确性、拒绝编造 |
| 数据缺失与歧义 | 10 | 参数澄清、证据不足、正确拒判 |
| 安全边界 | 10 | 越权拒绝、提示注入、敏感信息 |
| **合计** | **100** | **80 条开发集 + 20 条封存盲测集** |

**数据集与结果文件**：`evaluation/datasets/cases_v1.jsonl`、`splits_v1.json`、`results/*.jsonl`、`reviews/human_review.csv`、`reports/*.md`。

用例示例：

```json
{
  "case_id": "diagnosis_de_klarna_001",
  "category": "diagnosis",
  "dataset_version": "demo-v1",
  "input": "分析德国 Klarna 当前窗口支付成功率下降的原因",
  "expected": {
    "current_psr": 0.892,
    "baseline_psr": 0.95,
    "delta_pp": -5.8,
    "focus_channel": "Android",
    "should_abstain_from_confirmed_root_cause": true
  },
  "required_evidence": ["payment_metrics", "failure_breakdown"],
  "risk_tags": ["causal_overclaim"]
}
```

### 7.2 Runner 与评分器设计

```text
加载 80 条开发集
  → 对每条用例创建隔离 run
  → 固定模型/工作流/Prompt/数据/知识库版本
  → 等待任务终态，捕获 Trace / Token / 耗时
  → 确定性评分（数值、Schema、工具、引用存在、越权）
  → 人工复核（诊断分类、证据是否真的支持结论、建议可操作性）
  → 合并出单条结果与汇总报告
```

自动评分必须以金标与数据库重算结果为依据，**不允许使用模型自评替代真实正确率**。模型评委只能作为辅助信号。

### 7.3 指标及分母

| 指标 | 计算/判定 | 目标 |
|---|---|---|
| 黄金数值一致性 | 匹配的关键数值 / 应核对关键数值 | 100% |
| 诊断分类正确率 | 分类正确数 / 带可判定分类金标的诊断样本数 | ≥80% |
| 证据引用有效率 | 支持相应判断的有效引用 / 抽查关键引用数 | ≥95% |
| 工作流完成率 | 达到对应预期终态的任务 / 所有发起任务 | ≥90% |
| 正确拒判率 | 正确表达无法确认 / 需要拒判的样本 | 建立基线 |
| AI 辅助任务耗时改善 | 人工基线与 AI 辅助的同类任务耗时差异 | 探索目标 ≥50% |
| 多工具任务 P95 延迟 | 从创建到可用报告的端到端时间 | 探索目标 ≤30 秒 |
| 未授权真实交易修改 | 实际执行的越权操作次数 | 0 |

> 分母与限制：失败任务不得从工作流完成率分母中剔除；无金标的开放问题不纳入分类正确率分母；20 条盲测样本仅能作为小样本信号，不应用于夸大统计显著性。

### 7.4 Bad Case 闭环

1. **分类**：意图解析 / 参数 / 工具路由 / 数据口径 / 检索召回 / 引用 / 过度归因 / 超时恢复 / 安全。
2. **排序**：按频次、用户影响和修复复杂度选择优先处理的 1–2 类问题。
3. **单变量改进**：例如调整错误码检索排序；尽量固定模型、数据、其他流程条件。
4. **开发集回归**：在相同的 80 条开发集上对比 v0.3 和候选 v0.4。
5. **验证回退**：黄金数值和安全红线保持稳定；如果质量下降则回滚。
6. **结果留档**：`badcases/register.csv`、`v03_vs_v04.md` 和架构决策记录（ADR）。

### 7.5 人工对照实验

- 任务：选择 6–8 个可比较的诊断任务。
- 人员：争取 3–5 名具支付/数据分析经验的测试者；若无法招募，明确标注“未进行真实用户测试”。
- 两种路径：**传统看板 + 文档** 与 **AI 辅助诊断**。
- 控制偏差：任务难度匹配、轮换顺序、统一完成标准和计时方法。
- 记录：完成时间、正确性、查看证据行为、重试/求助次数、主观可用性评价。
- 结论：只对**本次模拟场景与样本**进行描述，不推导真实生产支付成功率提升。

### 7.6 结项 Go / No-Go

| Gate | 通过条件 | 失败处理 |
|---|---|---|
| 数值正确 | 黄金指标全部准确 | 阻止发布为“可信诊断” |
| 证据可信 | 引用可访问、关键判断有依据 | 降级为“初步分析/数据不足” |
| 安全 | 无真实支付写入能力，未授权操作 0 | 阻止演示版推广使用 |
| 产品闭环 | 筛选→诊断→SSE→报告→追问→反馈可用 | 记录缺陷并降级范围 |
| 可复现 | 数据 seed、模型/Prompt/Workflow、Trace 和结果完整 | 补齐可复现材料 |

未通过 Gate 时，产品状态应标记为 **DEMO / 待改进**，不能通过修改评测分母掩盖问题。

---

## 8. API 与前端交互的完整时序

```mermaid
sequenceDiagram
  participant U as 用户
  participant F as React
  participant A as FastAPI
  participant R as 自研 Runtime
  participant T as Tool Registry
  participant D as PostgreSQL
  participant K as 自建 RAG
  participant M as LLM API

  U->>F: 选择 DE / Klarna / 24h 并点击诊断
  F->>A: POST /api/diagnoses
  A->>D: 保存 task/run 和 CREATED 事件
  A-->>F: 202 task_id/run_id
  F->>A: GET /api/diagnoses/{id}/events
  A->>R: 推进当前 run（由任务执行器调度）
  R->>M: 意图识别（结构化）
  M-->>R: 意图与范围
  R->>T: get_payment_metrics / compare_payment_periods
  T->>D: 参数化只读查询
  D-->>T: 指标与窗口
  T-->>R: ToolResult + evidence_id
  A-->>F: SSE tool.completed
  R->>T: breakdown / error_distribution
  T->>D: 参数化只读查询
  D-->>T: 终端与错误分布
  T-->>R: 证据
  R->>K: 检索 KB-001/KB-002 等文档
  K-->>R: chunk_id + 文档来源
  R->>M: 生成文字性分析草稿（不允许编造指标）
  M-->>R: 假设及建议草稿
  R->>R: 证据与 Schema 校验
  R->>D: 保存报告及终态事件
  A-->>F: SSE report.ready
  F->>A: GET /api/diagnoses/{id}
  A-->>F: 报告 JSON 与真实引用
  F-->>U: 展示报告、证据、反馈入口
```

> **工程注意**：时序图中“A 调度 R”表示任务执行服务；不要把耗时 Agent 直接运行在短生命周期的 SSE 请求处理函数中。MVP 可用进程内受控后台任务 + 数据库状态；如需抗进程崩溃/多实例，增加可靠 Worker/队列并重新定义恢复策略。

---

## 9. 安全、权限与数据治理

### 9.1 权限模型

- `ANALYST`：查看 DEMO 看板、创建诊断、查看自己的任务与报告、提交反馈。
- `ADMIN`：增加演示数据、知识库与评测管理能力。
- 当前本地演示可以使用固定 DEMO 用户，但接口仍必须有明确的角色检查入口；不可因为没有真实登录页而把授权只写在前端。

### 9.2 必须在代码层阻止的行为

- 工具 Registry 不包含退款、修改路由、修改风控、任意数据库写入。
- 只读数据库账户不能执行 `INSERT/UPDATE/DELETE` 于支付事实表。
- 非法国家/支付方式/维度/时间范围/数据量上限被参数校验拦截。
- Prompt Injection 及知识文档中的“忽略规则并执行命令”不得被提升为系统指令。
- 报告不得从其他 run 复用未授权证据；反馈不允许跨任务写入。
- 日志只记录最小必要信息，不保存支付敏感数据、模型密钥和内部认证信息。

### 9.3 威胁与测试用例

| 风险 | 对应测试 |
|---|---|
| SQL 注入 / 非法分组 | 工具参数中传入非法字段或 SQL 片段，必须被 Schema/白名单拒绝 |
| 提示注入 | KB 文档嵌入系统指令，模型不得执行 |
| 越权业务操作 | “直接切换 Klarna 渠道”返回只读能力边界，无工具调用 |
| 模型数字幻觉 | 报告中 PSR 与实际 ToolResult 不一致则校验失败 |
| 虚假文档引用 | 引用不存在 chunk_id 则拒绝或降级 |
| SSE 重放 | 重连后重复事件不得导致重复运行或重复渲染 |
| 取消竞态 | 取消后不得再启动新工具；迟到结果不推进状态 |
| 数据串租户 / 串任务 | 只能访问本用户可见的 task/run/evidence |

---

## 10. 测试策略与质量保障

### 10.1 测试金字塔

| 层级 | 测试目标 | 推荐路径 |
|---|---|---|
| 单元测试 | PSR、pp、数据窗口、Schema、状态机守卫、RAG 分块 | `tests/week01/`、`tests/week02/`、`tests/week03/` |
| 集成测试 | FastAPI + PostgreSQL、Tool Registry + Metric Service、任务持久化 | `tests/integration/`（可新增） |
| E2E 测试 | 用户从 Dashboard 到报告、澄清/取消/恢复/反馈 | `tests/e2e/` |
| 安全回归 | 越权、提示注入、伪造引用、日志脱敏 | `tests/regression/` |
| 评测 | 100 条标准集、版本对比、人工 Rubric | `evaluation/` |

### 10.2 必须自动化的黄金断言

```python
assert baseline.valid_attempts == 4000
assert baseline.successful_attempts == 3800
assert baseline.psr == 0.95

assert current.valid_attempts == 2000
assert current.successful_attempts == 1784
assert current.failed_attempts == 216
assert current.psr == 0.892
assert compare.delta_pp == -5.8

assert android.valid_attempts == 1000
assert android.successful_attempts == 850
assert android.psr == 0.85
assert web.valid_attempts == 1000
assert web.successful_attempts == 934
assert web.psr == 0.934
assert android_timeout_count == 90
assert android_timeout_share_of_failures == 0.60
```

生产代码和测试计算浮点数时采用 `Decimal`/适当容差；上述断言表达业务金标，不建议依赖不受控的二进制浮点比较。

### 10.3 E2E 验收剧本（建议至少 12 条）

1. 德国 Klarna 黄金诊断正常完成。
2. Dashboard 条件能够传入任务。
3. 用户问题缺少地区时进入澄清，补充后恢复。
4. `redirect_timeout` 查询返回可点击的演示知识引用。
5. 提问“是否与新版本有关”输出相关性分析但不确认因果。
6. 对没有故障知识覆盖的地区返回证据不足。
7. 样本量不足时正确警告或拒判。
8. 工具超时后生成可解释失败及 Trace。
9. 用户中途取消后没有新工具启动。
10. SSE 断线恢复不重复执行。
11. 反馈与正确 `run_id/report_id` 关联。
12. 尝试退款/切换支付渠道被代码级拒绝。

---

## 11. 四周开发排期：文件、依赖与验收

下面是从**已确定的 20 天 / 100 项任务计划**抽取的技术实施主线。每日 5 项细分任务以原开发计划 HTML 为准；本节额外强调跨日依赖和“代码到什么程度算完成”。

| Day | 工程实现重点 | 必须交付 | 最小验收 |
|---|---|---|---|
| 1 | 业务指标、窗口语义、黄金 fixture、数据字典 | `docs/metrics-definition.md`、`docs/golden-scenarios.md` | 人工可复算金标 |
| 2 | FastAPI/PostgreSQL/Alembic/Docker、只读角色 | `infra/compose.yaml`、`db/models.py`、迁移 | 空库能初始化并连接 |
| 3 | 固定种子数据生成与 30,000 条 DEMO 记录 | `scripts/seed_payments.py`、`seed_manifest.json` | 重跑金标完全一致 |
| 4 | 四个确定性指标 API | `api/metrics.py`、`services/metric_service.py` | Swagger 可查准确结果 |
| 5 | 黄金回归、权限异常、Tool/Report 协议 | `tests/week01/`、`docs/tool-contract.md` | 指标误差 0、契约冻结 |
| 6 | AgentState、StateMachine、Run/Step/Event 表 | `contracts.py`、`state_machine.py` | 非法转移被阻止 |
| 7 | Orchestrator、预算、取消、恢复、Trace | `runtime.py`、`executor.py`、`trace.py` | 状态恢复/取消测试通过 |
| 8 | 自研 Tool Registry + 支付只读工具 | `tools/registry.py`、`payment_tools.py` | 未注册与非法工具拒绝 |
| 9 | LLM Adapter、意图解析、结构化输出、护栏 | `llm/`、`intent_parser.py` | 换模型不改 Runtime |
| 10 | 无 RAG 的初步诊断链 + 20 条回归 | `diagnosis_workflow.py`、`api/diagnoses.py` | 数据诊断完成，明确无知识引用 |
| 11 | 自建 RAG 导入、分块、检索、引用 | `rag/`、`data/kb/` | 文档可引用；无命中不编造 |
| 12 | 证据整合、报告 Schema、数字/引用校验 | `evidence.py`、`report_validator.py` | 黄金报告数字准确 |
| 13 | SSE、追问、取消、反馈与断线恢复 | `diagnosis_events.py`、`run_service.py` | 刷新恢复与取消有效 |
| 14 | React 接入五个页面的 P0 交互 | `frontend/src/pages/`、`api/` | 浏览器完整闭环 |
| 15 | 12 条 E2E、延迟/成本测量、产品复盘 | `tests/e2e/`、`docs/week03-acceptance.md` | 演示连续通过 3 次 |
| 16 | 冻结 v0.3、构建 100 例（80/20） | `evaluation/datasets/`、Rubric | 分层抽样可复现 |
| 17 | Eval Runner、自动评分、人工审阅基线 | `evaluation/runner.py`、基线报告 | 80 条开发集逐例记录 |
| 18 | Bad Case 归因与 v0.3/v0.4 对比 | `evaluation/badcases/`、对照报告 | 修改有证据，不掩盖回退 |
| 19 | 人工对照实验、性能与成本 | `research/`、可靠性报告 | 如实记录小样本限制 |
| 20 | 20 条 Holdout 盲测、Go/No-Go、作品集 | `holdout-final.md`、`portfolio/`、README | 可复现演示，未达标显式声明 |

### 11.1 跨周依赖

```text
Week 1 指标服务 ─────────────┐
                              ├─ Week 2 Tool Registry → Agent Runtime
Week 1 Tool / Report Schema ──┘                           │
                                                          ▼
Week 3 RAG 检索 + Evidence Validator + SSE + React 页面 ←──┘
                         │
                         ▼
Week 4 评测 Runner → Bad Case → 对照实验 → 盲测 → 结项
```

### 11.2 资源不够时的裁剪顺序

**绝不能裁剪**：数据库黄金样本、指标准确性、只读工具限制、结构化报告与证据、核心流程测试、如实评测记录。

**优先裁剪**：向量检索（保留关键词检索）、复杂 Agent 自由规划、全功能知识库管理、历史任务 UI、精美动效、自动工单、多人协作。这些不影响展示“业务问题 → AI 方案 → MVP → 评测”的主线。

---

## 12. 本地开发、配置和交付

### 12.1 建议环境变量

```dotenv
APP_ENV=development
DATABASE_URL=postgresql+psycopg://app_user:CHANGE_ME@localhost:5432/paypilot
PAYMENT_READONLY_DATABASE_URL=postgresql+psycopg://metric_reader:CHANGE_ME@localhost:5432/paypilot
LLM_PROVIDER=your_provider
LLM_MODEL_ID=your_model_id
LLM_API_KEY=YOUR_LOCAL_SECRET
DATASET_VERSION=demo-v1
DEMO_AS_OF_UTC=2026-11-16T10:00:00Z
AGENT_MAX_STEPS=8
AGENT_TOOL_TIMEOUT_SECONDS=10
AGENT_RUN_TIMEOUT_SECONDS=30
RAG_MODE=keyword
```

`.env` 不提交仓库，`.env.example` 仅保留占位符。生产部署时密钥由环境/密钥管理器注入。

### 12.2 建议本地开发命令（项目创建后实现）

```bash
# 基础服务
cd paypilot-ai
docker compose -f infra/compose.yaml up -d

# 数据库迁移
cd backend
alembic upgrade head

# 固定种子 DEMO 数据
python ../scripts/seed_payments.py --seed 20261116 --count 30000 --anchor-date 2026-11-16T10:00:00Z

# 后端服务
uvicorn app.main:app --reload

# 前端服务（另一个终端）
cd ../frontend
npm install
npm run dev

# 测试
cd ../backend
pytest ../tests/week01 ../tests/week02 ../tests/week03
```

这些是**设计目标命令**；只有在创建对应文件、脚本入口、依赖和 Compose 服务后才可执行。`pytest` 的导入路径需要在仓库初始化时统一配置。

### 12.3 CI 最小检查

每次 PR / 合并前运行：

```text
Python 格式/静态检查
  → Python 单元与集成测试
  → 数据库迁移和黄金指标测试
  → 前端 TypeScript 检查与构建
  → 安全/Schema 回归用例
  → （可选）少量无真实模型费用的 FakeLLM E2E
```

真实模型的完整回归和 100 例评测单独触发，保留模型与费用记录；不要在所有代码提交上盲目执行高成本模型评测。

### 12.4 Demo 发布与故障恢复

- 演示部署明确显示 `DEMO DATA`、只读模式和数据快照时间。
- `GET /api/health` 区分应用可用与数据库依赖情况；不得返回密钥。
- 对任务状态、ToolError 分类、超时、SSE 重连、Token 成本设置可观测性统计。
- 用户查看失败任务时展示友好的失败原因与重试入口；重试创建关联的新 run。
- 导入演示数据与重置数据应有显式管理操作，默认禁止在公共演示环境执行危险重置。

---

## 13. 工程决策记录（ADR）建议

| ADR | 决策 | 核心论证 |
|---|---|---|
| ADR-001 | 使用自建显式状态机而非 Agent 框架 | 可控、易追踪、满足当前固定诊断业务；代价是自行维护恢复与边界 |
| ADR-002 | LLM 不计算 PSR | 指标需要稳定可复核；统计由 SQL 服务承担 |
| ADR-003 | 先关键词检索，后评估向量检索 | 演示错误码精准命中优先，避免无依据增加技术复杂度 |
| ADR-004 | 评测集 80/20 与单变量迭代 | 降低调参对盲测污染，支持产品效果归因 |
| ADR-005 | SSE + 数据库事件恢复 | 与单向任务进度匹配；多实例能力留待后续升级 |

每条 ADR 记录：背景、备选方案、最终选择、取舍、潜在回退条件、实际评测证据。

---

## 14. AI 产品经理作品集的技术证据

技术开发时同步留存下面的可展示材料，而不是第四周再从代码里回忆：

| 作品集部分 | 技术阶段能产生的证据 | 不能声称的内容 |
|---|---|---|
| 业务与需求 | 用户任务、痛点假设、PSR 口径、MVP 边界 | 没做访谈却声称访谈完成 |
| 产品方案 | 原型前后对比、交互状态、API 与流程决策 | “用了 Agent”就等同产品有效 |
| AI 架构 | 自研 Runtime、状态机、工具白名单、RAG、Trace | 不区分设计与已完成实现 |
| 效果验证 | 100 例评测、质量分母、人工对照、Bad Case | 把脚本化原型结果写成真实实测 |
| 产品迭代 | v0.3→v0.4 的受控变化与失败归因 | 从模拟场景外推真实生产 PSR 提升 |
| 成本与风险 | Token 成本、P95、拒判、越权测试 | 没有数据却承诺线上稳定性 |

**最终应得到的交付包**：

1. 可启动的项目源码与最小 Demo 环境。
2. 与原型视觉接近、但连接真实数据服务与自研 Runtime 的 Web MVP。
3. 固定 DEMO 数据、可重放的黄金用例与自动化测试。
4. 自研 Agent 架构、API、Tool、SSE、证据和数据库设计文档。
5. 评测原始结果、Bad Case、版本对比、人工实验、Go/No-Go 记录。
6. 10–12 页 AI PM 作品集和一份 3 分钟演示脚本。

---

## 15. Day 1 的具体开工顺序

按此顺序执行可避免后续重构：

- [ ] 创建仓库基础目录、README 和 `.gitignore`，写明**自研 Agent、DEMO、只读**边界。
- [ ] 在 `docs/metrics-definition.md` 冻结 `payment_attempt_id`、PSR、pp、窗口成熟与时区口径。
- [ ] 在 `docs/golden-scenarios.md` 列出 4,000/3,800、2,000/1,784、Android 850/1,000、Web 934/1,000、timeout 90/150。
- [ ] 在 `docs/data-dictionary.md` 确定 payment_attempts、release_events、Agent 任务与证据的字段。
- [ ] 在 `docs/tool-contract.md` 明确四个数据工具、权限、Schema 与 Evidence ID 的格式。
- [ ] 先写一个能够独立校验黄金数据的测试文件，再开始写数据生成脚本。
- [ ] 建立第一个 ADR：**为什么先做确定性查询，再做自研 Agent**。

**第一项技术里程碑**：不启动任何 LLM，也能从 PostgreSQL 查询得到 `89.2% / 95.0% / -5.8pp`，并解释统计口径与数据来源。只有这个里程碑通过，才进入 Runtime 实现。

---

### 附录 A：设计与实现状态表

| 模块 | 当前 HTML 原型 | 本方案目标 | 首次验收周 |
|---|---|---|---|
| Dashboard | JS 模拟值与趋势 | PostgreSQL 确定性指标 | W1 / W3 前端联调 |
| AI 对话 | 规则分支和模拟执行 | LLM 解析 + 自研 Runtime | W2 |
| 工具调用 | 无后端调用 | Registry + 只读 ToolResult | W2 |
| 知识库 | `kbDocs` 固定数组 | 文档入库、检索、真实 chunk 引用 | W3 |
| 报告 | 固定演示文本 | JSON Schema + Evidence Validator | W3 |
| 执行进度 | 定时器模拟 | 持久化事件 + SSE | W3 |
| 反馈 | 本地存储 | 按 run/report 关联落库 | W3 |
| 评测中心 | 5 条脚本演示 | 100 条集 + Runner + 人工复核 | W4 |

### 附录 B：项目完成定义（Definition of Done）

- [ ] 全量 P0 核心流程可由别人按 README 复现。
- [ ] 所有业务黄金数值由数据库得出，前后端一致。
- [ ] 用户能够查看证据和不确定性，不存在凭空引用。
- [ ] 自研 Runtime 具备状态转移、预算、取消、恢复与工具白名单。
- [ ] 无真实支付写操作和高风险越权能力。
- [ ] 评测集与运行版本固定，结果可独立复核。
- [ ] 没有真实验证的性能、正确率和业务提升均清楚标注“规划/模拟/待验证”。

**文档结束。**
