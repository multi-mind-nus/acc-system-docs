# 会计事务所资料收集系统 - Agent 开发文档

## 1. 文档用途

本文把[系统架构设计](./README_系统架构设计.md)中的 AI 边界拆成 `acc-system-agent` 的可执行开发阶段。Agent 是本地 Harness：它读取只读资料卷、调用远端已训练模型并返回规范化结果，不连接 PostgreSQL/Redis，不修改业务状态。

Agent 仓库：`acc-system-agent`

技术基线：Python 3.12、uv、FastAPI、Pydantic、httpx、pytest、Docker、GitHub Actions 和 Amazon ECR。

## 2. 通用开发与验收规则

1. Backend-Agent JSON 统一使用 `snake_case`，错误响应固定为 `{ code, message, details, request_id }`；
2. Agent 只接受内网请求，不映射宿主机端口，资料卷只读挂载到 `/data/documents`；
3. `storage_key` 解析后必须仍位于资料根目录，读取前校验 SHA-256；
4. 日志只记录 request/run/document id、状态、耗时和错误码，不记录文件正文、API key 或模型原始敏感输出；
5. Agent 不做后台重试，超时或远端错误立即返回给 Backend Worker 统一重试；
6. 每阶段完成时 `uv run pytest`、镜像构建和对应真实 Backend 联合验收均通过。

## 3. 阶段总览

| 阶段 | Agent 交付结果 | 系统依赖 |
| --- | --- | --- |
| A1 服务与部署基线 | FastAPI、健康检查、只读文件边界、远端模型客户端、镜像与 ECR | B6.1 |
| A2 完整文件快速分类 | `purpose=CLASSIFY`，只返回资料类别 | B6.2、F7.2 |
| A3 提交后完整审核 | `purpose=REVIEW`，字段、findings、evidence、搜索动作和金额关系 | B6.3-B6.4、F7.3 |

## 4. A1：服务与部署基线

### 4.1 阶段目标

建立可独立构建、发布、配置和健康检查的 Agent 服务，先锁定文件与远端模型信任边界。

### 4.2 交付内容

- `uv + FastAPI` 项目、锁定依赖、结构化日志和统一错误响应；
- `GET /health/live` 只检查进程，`GET /health/ready` 检查配置与远端模型可达性；
- `POST /v1/analyze`，并要求 `Idempotency-Key: <run_id>:<turn>`；
- `DOCUMENT_PATH=/data/documents`、`MODEL_API_URL`、`MODEL_HEALTH_URL`、`MODEL_API_KEY`、`MODEL_CONNECT_TIMEOUT_SECONDS=10`、`MODEL_REQUEST_TIMEOUT_SECONDS=180`；
- 路径标准化、越界防护、文件存在性、大小、内容类型和 SHA-256 校验；
- Dockerfile、`.dockerignore`、GitHub Action；main/tag 使用 AWS OIDC 推送 ECR `acc-system-agent:<git-sha>`；
- Compose `agent` 服务仅加入 internal network，只读挂载 documents 卷，不挂载隔离区、数据库或 Redis 凭据。

### 4.3 可验收场景

| ID | 场景与操作 | 预期结果 |
| --- | --- | --- |
| A1-A1 | 启动 Agent 并请求 live/ready | live 只反映进程；ready 能区分远端模型可用/不可用 |
| A1-A2 | 传入 `../`、绝对路径、符号链接越界或错误哈希 | 请求被拒绝，不读取资料根目录外文件 |
| A1-A3 | 远端模型超时或返回非 JSON | Agent 返回稳定错误码，不内部重试或泄漏密钥 |
| A1-A4 | 检查 Compose 端口、挂载和日志 | 无公网/宿主机端口，资料卷只读，日志无文件正文和 secret |

### 4.4 完成标志

Agent 可由固定 SHA 镜像在 Compose 内启动，Backend 可通过 `AGENT_URL=http://agent:8000` 访问，远端故障不会暴露本地文件或凭据。

### 4.5 本地验收候选（2026-09-22，待用户验收）

- uv/FastAPI、锁文件、非 root Docker image、健康检查、结构化日志、统一错误及 ECR workflow 已实现；本地镜像 `acc-system-agent:local` 的版本为 `b6.1-a1-local`。
- **15 passed**，覆盖文件越界/符号链接/非普通文件、大小/MIME/SHA-256、错误幂等键、模型缺省、超时/网络错误/非法 JSON/重定向，以及错误响应和日志脱敏。
- Backend → Agent live 为 200；未配置真实模型，ready 正确返回 `503 MODEL_NOT_CONFIGURED`。容器无宿主机端口、资料卷只读、无 DB/Redis 凭据；Agent 停机不影响 Backend readiness。
- `MODEL_HEALTH_URL` 为显式健康地址，暂定使用 Bearer 授权的 `GET` 并返回 `200 {"status":"ok"}`；真实供应方协议尚未提供，后续按实际接口适配，不能将当前测试 transport 视为真实模型联调完成。
- A1 的 `/v1/analyze` 只实现请求、幂等键及文件校验；未配置模型返回 `MODEL_NOT_CONFIGURED`，配置后返回 `501 ANALYSIS_NOT_IMPLEMENTED`。A2/A3 再加入 purpose-specific schema、推理和幂等结果复用。
- ECR workflow/部署模板已就绪，真实 GitHub 发布及 AWS 权限配置待验收后部署时执行；A2/A3 尚未实施。

## 5. A2：完整文件快速分类

### 5.1 阶段目标

读取完整 PDF/图片并调用远端模型，只返回客户确认分类所需的最小结果，不提前执行审核。

### 5.2 协议

```json
{
  "schema_version": "1",
  "run_id": "uuid",
  "purpose": "CLASSIFY",
  "requirements": [
    {"id": "uuid", "document_type": "SUPPLIER_INVOICE", "title": "Supplier invoices"}
  ],
  "documents": [
    {"document_id": "uuid", "storage_key": "firm/client/uuid", "content_type": "application/pdf", "sha256": "hex", "original_name": "invoice.pdf"}
  ]
}
```

响应只允许：

```json
{
  "schema_version": "1",
  "run_id": "uuid",
  "model_version": "model-v1",
  "classifications": [
    {
      "document_id": "uuid",
      "category": "REQUIREMENT",
      "document_type": "SUPPLIER_INVOICE",
      "requirement_id": "uuid",
      "confidence": 0.96
    }
  ]
}
```

`category` 只允许 `REQUIREMENT | OTHER | INVALID`。响应不得包含 findings、evidence、主体/期间/金额提取、搜索动作或审核决定。

### 5.3 可验收场景

| ID | 场景与操作 | 预期结果 |
| --- | --- | --- |
| A2-A1 | 同时分类多个 PDF/图片 | 每个输入 document id 恰好返回一条结果，无重复/未知 id |
| A2-A2 | 远端返回未知 requirement、错误 run id、越界置信度或审核字段 | Agent 拒绝响应并返回稳定 schema 错误 |
| A2-A3 | 重放相同 `Idempotency-Key` 与请求 | 返回一致结果，不产生第二套业务副作用 |
| A2-A4 | 检查分类输出 | 不包含审核 finding、搜索动作或完整字段提取 |

### 5.4 完成标志

Agent 能使用完整文件返回最小分类 schema，对远端模型的额外或非法输出 fail closed。

### 5.5 A2 本地功能（2026-09-22，用户验收通过）

- 与 B6.2/F7.1/F7.2 同批交付，完整读取只读卷上的扫描后文件再分类；不做审核、搜索或业务状态持久化。
- `CLASSIFICATION_PROVIDER=DISABLED|MOCK|REMOTE`；Compose 由 `AGENT_CLASSIFICATION_PROVIDER` 同步。MOCK 仅非生产环境使用，按文件名/内容关键词产生确定性示例，模型版本 `mock-classifier-v1`，页面明确显示模拟标识；这不是训练后的模型。
- REMOTE 暂定 Bearer JSON POST，文件 `storage_key` 替换为完整 `content_base64`，保留其他元数据；响应严格 snake_case，拒绝未知 id/错误顺序/未知资料项/越界置信度/额外 finding。真实协议未提供，当前仅使用可控 transport 测试，不宣称真实模型联调完成。
- 单批上限 100 文件/100 MiB；响应上限 1 MiB。单进程缓存最近 128 个成功结果及请求摘要，冲突幂等键拒绝；缓存重启/淘汰后可能重复推理，生产模型需支持透传的幂等键，Backend 持久结果和确认幂等保证业务不重复合入。
- Agent 自动测试 **23 passed**；`REVIEW` 尚未实施。A1 的 501 记录为当时基线，不再描述当前 CLASSIFY 能力。

## 6. A3：提交后完整审核

### 6.1 阶段目标

在客户提交后返回字段提取、requirement findings、evidence、客户补交消息、搜索动作和结构化金额关系。Agent 仅返回分析，不执行搜索或修改业务状态；Backend 执行授权搜索后把结果加入下一轮输入。

`analysis_type` 是后端提供的内部分析提示，不由会计选择，也不表示两种检查互斥。银行对账单仍需主体/期间/完整性校验；对账交易的日期、描述、金额、币种来自提交后的完整文件提取，再匹配本次及授权历史支持资料。不得要求创建请求时提供 `criteria.target_transaction`，也不得将旧请求遗留的手填交易自动当作本轮提取事实。

### 6.2 协议

`purpose=REVIEW` 允许动作：

```text
SEARCH_CURRENT | SEARCH_HISTORY | ASK_CLIENT | RESOLVE | ESCALATE
```

finding 至少包含 `requirement_id`、`action`、`suggested_decision`、`issue_code`、`evidence[]` 和面向客户的 `client_message`。`MISSING/INCOMPLETE` 还必须用 `requested_document_type` 指明客户需要上传的资料类型；补交 finding 必须归到清单中同类型的 requirement，不存在对应资料项或归属错误时拒绝自动补交并转人工。REVIEW finding 不包含 `confidence`；多余字段将被拒绝。金额关系使用十进制字符串：

```json
{
  "currency": "SGD",
  "operation": "SUM",
  "operands": [
    {"document_id": "invoice-uuid-1", "amount": "1246.02", "label": "Invoice 1"},
    {"document_id": "invoice-uuid-2", "amount": "1582.78", "label": "Invoice 2"}
  ],
  "expected_amount": "2828.80",
  "actual_amount": "2828.80",
  "difference": "0.00"
}
```

Agent 校验输出结构，Backend 负责证据归属、`Decimal` 重算与状态迁移。`review_preference` 由 Backend 传入，用于指导 Agent 在 `ASK_CLIENT` 与 `ESCALATE` 间分流，不能降低事实和证据要求。

### 6.3 可验收场景

| ID | 场景与操作 | 预期结果 |
| --- | --- | --- |
| A3-A1 | G02/G03 错期间/错主体 | 返回稳定 issue code、明确客户消息和可校验 evidence |
| A3-A2 | B01 多张发票支持一笔付款 | 返回全部支持文件与结构化合计关系 |
| A3-A3 | B03 需要上期证据 | 先返回 `SEARCH_HISTORY`；Backend 提供搜索结果后再解决或要求客户补交 |
| A3-A4 | F02/F04/F05 报销、外币、进度款/保留款 | 返回用于 Backend 重算的完整 operands 与证据关系 |
| A3-A5 | 远端返回未知 action/issue code、错误 evidence id 类型或非十进制金额 | Agent 拒绝响应，不伪造默认审核结果 |
| A3-A6 | 检查 Agent 容器与日志 | 无数据库连接，无业务写入，无文件正文或模型敏感输出 |

### 6.4 完成标志

G02、G03、B01、B03、F02、F04 和 F05 的固定模型桩联合验收通过；Agent 只返回结构化分析，Backend 仍是证据、自动执行和整单批准的唯一业务边界。

### 6.5 A3 第一批实现记录（2026-09-23，用户验收通过）

- REVIEW 使用独立严格 schema；完整文件读取、路径/MIME/SHA-256 校验和响应上限与 CLASSIFY 一致。协议定义为 `app/analysis_schemas.py`，Backend 保存同一契约，所有 JSON 仍为 snake_case。
- 接受 context（主体/期间/提交 id）、内部 requirements、文件与资料项关联、CURRENT/HISTORY 范围、turn 和 search_history。最终响应要求每文件一份 extraction、每资料项一份 finding；搜索响应不能混入最终 finding。
- finding 包含 action、suggested_decision、issue_code、requested_document_type、主体/期间检查、解释、客户建议、evidence 和金额关系；动作限定 SEARCH_CURRENT/SEARCH_HISTORY/ASK_CLIENT/RESOLVE/ESCALATE，不接受 WAIVE/APPROVE，也不包含审核 confidence。
- 金额十进制字符串支持 SUM/SUBTRACT/MULTIPLY，每个 operand 带 document_id、amount、label；用于合计、扣款、汇率换算等关系，Backend 再独立校验。Agent 不执行搜索或业务状态修改。
- `REVIEW_PROVIDER=DISABLED|MOCK|REMOTE`，Compose 对应 `AGENT_REVIEW_PROVIDER`。MOCK 仅非生产使用，文件名 G02/G03/B01/B03/F02/F04/F05 选择固定样例，返回 `mock-reviewer-v1`。模拟的主体/期间/交易/金额仅验证页面与协议，不是真实提取；普通文件返回未知字段并 ESCALATE，不臆造识别结果。
- REMOTE 复用 Bearer 完整文件 JSON transport；真实模型协议仍待提供。单进程最近 128 结果缓存，相同 run:turn 返回一致响应，冲突内容拒绝，重启/多副本依赖远端幂等键与 Backend 持久化去重。
- 自动测试 **33 passed**，包括固定场景、幂等冲突、非法 schema/枚举/证据/金额、完整文件远端传输和生产拒绝 MOCK。A3 的真实模型准确率与供应方联调尚未验收。

## 7. Novita OCR 与 Flash 接入增量（2026-09-25）

保留 A1–A3 的历史验收记录；本节是其后的真实服务适配，不能把此前 MOCK 场景视为模型准确率验收。

- 新增 `DEEPSEEK` provider。图片及 PDF 的每一页先经 Novita `deepseek/deepseek-ocr-2` 识别，再把 OCR 文本交给 Novita Flash。初始联调使用 V4，现行默认模型为 `deepseek/deepseek-v4.1-flash`；两次调用均使用其 OpenAI 兼容 Chat Completions。Agent 不部署模型权重，也不连接业务数据库。
- `CLASSIFY` 只产生类别、目标资料项和分类置信度；`REVIEW` 产生提取、finding、evidence、搜索动作和金额关系，不包含 `confidence`。模型输出继续经过严格 schema 校验，Backend 继续负责授权搜索、Decimal 重算和业务状态。OCR 失败、截断或未配置时必须显式失败，不能以文件名模拟结果冒充真实审核。
- Compose 用 `NOVITA_API_KEY` 为 OCR 与 Flash 提供同一凭据；可选 `OCR_*` 和 `MODEL_*` 覆盖各自 URL、模型与凭据。默认地址均为 `https://api.novita.ai/openai/v1/chat/completions`。初始接入时真实端到端测试尚未完成，后续选定样本结果见 7.1；密钥不保存在仓库或文档。
- REVIEW 调用启用 V4.1 Flash 的普通 Thinking（`reasoning.effort=low`），用于提升复杂证据关系和严格 JSON 输出的稳定性；上传 CLASSIFY 保持 Non-thinking，以维持快速分类。Thinking 不能替代 schema 与 Backend 证据校验。
- REVIEW 单轮 OCR 与推理共享 290 秒预算，Backend 等待 300 秒并持有 330 秒任务租约；CLASSIFY 仍使用 150 秒预算。预算增加用于容纳 Thinking，不改变最多三轮搜索或失败重试规则。
- 新增 `POST /v1/analyze-inline`：以 `content_base64` 代替共享卷路径，使用 `AGENT_API_KEY` Bearer 鉴权；两种 purpose 使用同一输入/输出业务协议。面向外部复用时需置于 HTTPS、限流和可信网关之后。原 `/v1/analyze` 保持 Folio 内网接口。
- 新增测试覆盖 PNG 与真实 PDF 页渲染、OCR 先于 Flash、OCR 文本而非原始二进制进入 Flash、独立 API 鉴权、校验和、审核 ESCALATE。仍需用 Novita 实际调用及 G02/G03/B01/B03/F02/F04/F05 材料做准确率和延迟验收。
- 初始接入验证记录：49 个自动测试通过，Agent Docker 镜像构建通过；Novita 真实调用已跑通图片 OCR→分类、图片 OCR→审核、G02 错期间 PDF（`ASK_CLIENT/WRONG_PERIOD`）和 G03 错主体 PDF（`ASK_CLIENT/ENTITY_MISMATCH`）。G03 曾有一次结构不合约的输出被拒绝；调整提示后重测通过。该记录不代表后续样本或稳定准确率的验收状态。

### 7.1 基于选定真实样本的通用改进与回归

- 不按 case ID 或文件名决定真实审核结论。任务提供的指定交易仅作为审核范围，须从银行文件 OCR 核实；未指定交易时不能猜测其中一笔为目标。
- Backend 在 REVIEW 文件引用中携带 `submission_round`。旧轮错误文件继续留作审计；新轮有经 OCR 验证的有效更正材料时，Agent 以最新轮次判断该资料项，不再因旧文件仍在输入里重复退回。
- 对缺失支持资料先搜索当前和历史记录；未结清项目登记表只作辅助，不能代替原始发票。外币换算、手续费、进度款保留款均须给出完整证据链和可用 `Decimal` 复算的结构化金额关系。模型生成的 `model_version` 不可信，由 Agent 按实际请求模型配置写入。
- 审核模型不接收原文件名，避免数据集的 `wrong/correct` 文件名泄露答案；Backend 仍保留文件名供展示和审计。若未结清登记表的发票编号与已引用的原始发票匹配，必须把登记表一并列入历史证据；不相关登记表不引用。
- Agent 的不合约结果最多要求模型修正两次，仍不合约则显式失败。相同文件的成功 OCR 文本按哈希在单进程缓存；PDF 原生渲染串行，图片 OCR 可并行。Backend 的历史/当前搜索遇到过窄元数据过滤零命中时，在相同事务所、客户和搜索范围内重试宽松查询。
- V4.1 Flash 的多文件 REVIEW 将输出预算从 8192 提到 16384；输出截断仍被拒绝，不返回半份 JSON。修正请求只包含简短校验错误，不回灌完整错误响应。`SATISFY` 除算式自洽外，还要求每步金额差额不超过半分舍入容差；Backend 独立检查这一点。
- 用 `REVIEW_PROVIDER=DEEPSEEK uv run python -m scripts.evaluate_v5 --phase trajectory <case_dir>` 可复现多轮流程；ground truth 仅用于评价，不发给模型。使用 V4.1 Flash 且不向模型传文件名，选定六个样本各有一次完整轨迹通过：G02_0026/G03_0055 错材料退回后补交通过；B01_0850/B03_1822/F04_0903/F05_0249 完成搜索/补交后的 `RESOLVE/SATISFY`，最终 evidence 文件集合和金额关系符合答案。F04/F05 包含干扰材料，未被引用。B03 的登记表必须与所引用发票编号匹配且被列入证据。
- 历史测试时 Agent 60 passed，Backend 新增搜索/文件类型/金额差额单元测试 3 passed。F02 和全数据集尚未评价；随机输出稳定性、延迟、成本及生产联调未验收。当时模型给出的 G02 更正件 0.95、B01 齐件 0.97 会因旧请求阈值转人工；当前已取消该自报分数门槛，一次样本通过仍不等于生产可自动放行。

### 7.2 报销收据证据覆盖回归

- 通用规则：`EXPENSE_CLAIM` 的每个报销行必须有不同的原始 `RECEIPT` 文件支持；报销单本身不是收据。自动满足相关资料项时，报销单行项目、总额、收据提取金额和 `SUM` operands 必须一致，相关 finding 必须引用完整证据链。Agent 和 Backend 使用同一校验契约，避免模型自称通过就写入审核决定。
- 当前资料搜索后，若具体缺失材料已明确，就形成 `ASK_CLIENT/INCOMPLETE`；不再为了已明确的当期补交事项强制查历史。证据仍不清楚或可能涉及上期义务时，历史搜索和人工升级仍可用。
- 使用 F02_0002 的实际 PDF 与三项必交资料模拟真实请求：缺第二张收据时，真实 Novita OCR→Flash 返回搜索当前资料，随后银行对账单与报销单使用 `ESCALATE/INCOMPLETE` 等待支持，只有收据项使用 `ASK_CLIENT/INCOMPLETE`，且 `requested_document_type=RECEIPT`；补齐后一次真实调用返回三项 `SATISFY`，两张收据分别匹配两笔报销并作为合计 operands。一次补齐材料调用曾输出截断并超时，不能据此声称随机稳定性或生产时延已验收。测试不在生产规则里使用 case ID、文件名或答案。
- 跨资料项补交规则：银行对账发现缺发票时，银行 finding 使用 `ESCALATE` 等待支持，发票 finding 使用 `ASK_CLIENT`，不能把补交动作挂在银行单上并同时满足发票项。B01_0850 缺一张发票的真实 OCR→Flash 回归得到银行 `ESCALATE/INCOMPLETE`、发票 `ASK_CLIENT/MISSING`；后端状态测试确认只有发票项进入 `NEEDS_ACTION`。

## 8. 阶段验收记录模板

```text
阶段：A1/A2/A3
Agent 提交 SHA：
Agent 镜像：
Backend 提交 SHA/镜像：
远端模型版本：
验收环境：
自动测试结果：
手工场景结果：
未完成项：
验收人：
日期：
```
