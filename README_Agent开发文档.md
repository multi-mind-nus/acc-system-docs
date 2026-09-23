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

finding 至少包含 `requirement_id`、`suggested_decision`、`issue_code`、`confidence`、`evidence[]` 和面向客户的 `client_message`。金额关系使用十进制字符串：

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

Agent 校验输出结构，Backend 负责证据归属、`Decimal` 重算、置信度阈值与状态迁移。

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
- finding 包含 action、suggested_decision、issue_code、置信度、主体/期间检查、解释、客户建议、evidence 和金额关系；动作限定 SEARCH_CURRENT/SEARCH_HISTORY/ASK_CLIENT/RESOLVE/ESCALATE，不接受 WAIVE/APPROVE。
- 金额十进制字符串支持 SUM/SUBTRACT/MULTIPLY，每个 operand 带 document_id、amount、label；用于合计、扣款、汇率换算等关系，Backend 再独立校验。Agent 不执行搜索或业务状态修改。
- `REVIEW_PROVIDER=DISABLED|MOCK|REMOTE`，Compose 对应 `AGENT_REVIEW_PROVIDER`。MOCK 仅非生产使用，文件名 G02/G03/B01/B03/F02/F04/F05 选择固定样例，返回 `mock-reviewer-v1`。模拟的主体/期间/交易/金额仅验证页面与协议，不是真实提取；普通文件返回未知字段并 ESCALATE，不臆造识别结果。
- REMOTE 复用 Bearer 完整文件 JSON transport；真实模型协议仍待提供。单进程最近 128 结果缓存，相同 run:turn 返回一致响应，冲突内容拒绝，重启/多副本依赖远端幂等键与 Backend 持久化去重。
- 自动测试 **33 passed**，包括固定场景、幂等冲突、非法 schema/枚举/证据/金额、完整文件远端传输和生产拒绝 MOCK。A3 的真实模型准确率与供应方联调尚未验收。

## 7. 阶段验收记录模板

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
