# AI Agent 接口与架构交接

## 1. 交付目标

REVIEW 公司上下文补充（2026-09-25）：`context` 增加 `industry`、`base_currency`、`features` 和 `bank_accounts`，由 Backend 在提交时从该事务所的客户档案生成并存入 run 快照，重试及搜索轮次沿用同一快照。银行账户仅包含当前客户的启用账户（银行名称、尾四位、币种）。Agent 将上下文随请求传给远端模型，不另查数据库。兼容旧 run：前三个字段缺失时为 `null`，账户列表为空，表示上下文未提供，不代表公司没有相关业务或银行账户；不得据此虚构结论。`features` 仅包含系统维护的六项特征，没有 `has_employees`。行业编码为 `PROFESSIONAL_SERVICES / ONLINE_COMMERCE / PROJECT_ENGINEERING / TRADING_DISTRIBUTION / FOOD_BEVERAGE / SOFTWARE_SAAS / OTHER`。上传 CLASSIFY 协议不增加这些审核字段。

资料类型补充（2026-09-25）：创建请求支持 `BANK_STATEMENT`、`SALES_INVOICE`、`PURCHASE_INVOICE`、`RECEIPT`、`CREDIT_NOTE`、`SALES_REPORT`、`PAYMENT_PLATFORM_REPORT`、`SETTLEMENT_REPORT`、`PAYROLL_REPORT`、`EXPENSE_CLAIM`、`LOAN_STATEMENT`、`FX_ADVICE`、`PROGRESS_CLAIM`、`PAYMENT_CERTIFICATE`、`OPEN_ITEMS_REGISTER`、`OTHER`。v5 数据集的 `SUPPLIER_INVOICE` 对应系统的 `PURCHASE_INVOICE`；模型适配时须处理该映射，其余数据集类型保持同名。`PAYMENT_PLATFORM_REPORT` 是保留的综合平台报表类型，`SETTLEMENT_REPORT` 表示具体结算报告。此补充仅覆盖材料类别，不表示所有 case 的交易审核逻辑均已实现。

本文交给负责模型和 AI 服务的开发人员，说明训练完成的模型怎样接入资料收集系统。当前系统已经包含 `acc-system-agent` 适配层；推荐保留它，只实现远端模型服务，而不是在模型服务中复制业务状态机。

```text
Backend Worker
  → acc-system-agent（内网、无宿主机端口）
    → Remote Model Web API（AI 团队实现）
```

职责边界：

- Backend 是唯一业务状态机和事实源，拥有 PostgreSQL、Redis、任务重试、资料搜索、审核决定和通知记录；
- Agent 只读挂载 `/data/documents`，校验文件后调用模型，规范化并严格校验模型输出；
- Remote Model API 接收完整文件和结构化上下文，只返回分析结果；
- Agent 和 Remote Model 均不得连接业务 PostgreSQL/Redis，不得修改资料项或收集请求状态；
- AI 不得执行 `WAIVE` 或批准整单。Agent 按请求级 `review_preference` 决定明确补交或转人工，Backend 校验后才可自动退回；全部单项通过后，整单仍等待会计确认；
- 模型不可用时，上传后的手动分类和会计人工审核仍可继续。

现有契约的权威代码：

- `acc-system-agent/app/schemas.py`：`CLASSIFY`；
- `acc-system-agent/app/analysis_schemas.py`：`REVIEW`；
- `acc-system-backend/app/analysis_schemas.py`：Backend 保存的同版 `REVIEW` 契约。

如果代码与本文存在差异，以发布 SHA 中的 Pydantic schema 为准，并先协商升级 `schema_version`，不能单方面增加字段。

## 2. 推荐实现方式

AI 团队只需交付一个无状态 HTTP 服务：

```text
POST /analyze
  purpose=CLASSIFY → 快速分类 pipeline
  purpose=REVIEW   → OCR/字段提取 + 审核推理 pipeline

GET /health
  → 模型、推理运行时和必要依赖已就绪
```

建议内部保持以下简单分层：

```text
HTTP adapter
  → request/schema validator
  → document decoder + OCR/normalizer
  → purpose router
      ├─ fast classifier
      └─ review orchestrator
  → deterministic output validator
  → strict JSON response
```

首版不需要向量数据库、工作流引擎或业务数据库副本。需要更多资料时返回搜索动作，由 Backend 在授权范围内查找并开启下一轮。单副本可使用有界内存缓存处理幂等；多副本时可在 AI 服务内部使用 Redis，但 Redis 只能保存推理幂等结果，不能保存业务状态。

建议把 LLM/视觉模型推理与以下确定性逻辑分开：

- JSON schema 和枚举校验；
- 文件与输出 ID 对齐；
- 金额字符串规范化；
- 上传分类置信度校准；
- 幂等键冲突检查；
- 日志脱敏。

## 3. Backend → Agent 接口

### 3.1 健康检查

| 方法 | 路径 | 含义 |
| --- | --- | --- |
| `GET` | `/health/live` | 进程存活；不代表模型可用 |
| `GET` | `/health/ready` | 资料卷和远端模型均可用 |

未配置模型时，`/health/ready` 返回 `503 MODEL_NOT_CONFIGURED`。这不会使 Backend 健康检查失败，也不会关闭人工流程。

### 3.2 分析入口

```http
POST /v1/analyze
Content-Type: application/json
Idempotency-Key: <run_id>:<turn>
```

- 所有字段使用 `snake_case`；
- `schema_version` 当前固定为字符串 `"1"`；
- `CLASSIFY` 的 `turn` 固定为 `0`，幂等键为 `<run_id>:0`；
- `REVIEW` 的 `turn` 为 `0..3`，幂等键必须与请求一致；
- Backend 与 Agent 位于 Compose 内网，Agent 不映射宿主机端口。

Agent 接口接收 `storage_key`，并从只读卷读取文件。Remote Model API 不接收 `storage_key`，详见第 4 节。

## 4. Agent → Remote Model 接口

当前适配器使用以下暂定协议，AI 团队联调后应冻结为正式协议：

```http
POST ${MODEL_API_URL}
Authorization: Bearer ${MODEL_API_KEY}
Idempotency-Key: <run_id>:<turn>
Content-Type: application/json
```

Agent 会：

1. 校验路径、普通文件、MIME 文件头、大小和 SHA-256；
2. 删除每个 document 的 `storage_key`；
3. 增加 `content_base64`，内容为完整文件；
4. 原样转发 `document_id`、`original_name`、`content_type` 和 `sha256`；
5. 将 Remote Model 的 JSON 响应按严格 schema 校验后才返回 Backend。

远端健康检查：

```http
GET ${MODEL_HEALTH_URL}
Authorization: Bearer ${MODEL_API_KEY}
```

成功响应固定为：

```json
{"status":"ok"}
```

约束：

- 允许文件：PDF、PNG、JPEG；
- 单文件最大 25 MiB；
- 单请求最多 100 个文件、合计最大 100 MiB；
- 模型响应最大 1 MiB；
- 默认连接超时 10 秒、单轮推理超时 180 秒；
- Agent 不跟随 HTTP 重定向，也不在内部重试；
- 任意非 200 模型响应统一视为模型不可用，错误正文不会传给 Backend；
- 日志不得包含文件正文、Base64、API key、Authorization header 或完整模型响应。

## 5. `CLASSIFY`：上传时快速分类

上传阶段只回答“文件应该放到哪里”，不得执行主体、期间、金额或审核结论判断。

### 5.1 请求示例

Remote Model 实际收到的 document 使用 `content_base64`：

```json
{
  "schema_version": "1",
  "run_id": "11111111-1111-4111-8111-111111111111",
  "purpose": "CLASSIFY",
  "documents": [
    {
      "document_id": "22222222-2222-4222-8222-222222222222",
      "original_name": "august-bank-statement.pdf",
      "content_type": "application/pdf",
      "sha256": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
      "content_base64": "JVBERi0xLjcKLi4u"
    }
  ],
  "requirements": [
    {
      "id": "33333333-3333-4333-8333-333333333333",
      "document_type": "BANK_STATEMENT",
      "title": "Bank statement"
    }
  ]
}
```

### 5.2 响应示例

```json
{
  "schema_version": "1",
  "run_id": "11111111-1111-4111-8111-111111111111",
  "model_version": "document-classifier-2026-09-01",
  "classifications": [
    {
      "document_id": "22222222-2222-4222-8222-222222222222",
      "category": "REQUIREMENT",
      "document_type": "BANK_STATEMENT",
      "requirement_id": "33333333-3333-4333-8333-333333333333",
      "confidence": 0.994
    }
  ]
}
```

规则：

- `category` 只能是 `REQUIREMENT | OTHER | INVALID`；
- `REQUIREMENT` 必须设置已提供的 `requirement_id`；`OTHER/INVALID` 的 `requirement_id` 必须为 `null`；
- `document_type` 可为空，非空时最长 64 字符；
- `confidence` 必须为有限的 `0..1` 数值；
- `classifications` 必须逐个覆盖输入 document，数量、ID 和顺序完全一致；
- 不允许返回 extraction、finding、issue、金额、主体、期间或审核意见；
- 同一幂等键与相同请求必须返回一致结果；同一键对应不同请求必须报冲突。

客户会在 UI 中确认或拖动调整分类；确认前文件不进入正式资料清单。因此模型分类不是审核决定。

## 6. `REVIEW`：客户提交后的完整审核

### 6.1 请求结构

```json
{
  "schema_version": "1",
  "run_id": "44444444-4444-4444-8444-444444444444",
  "purpose": "REVIEW",
  "turn": 0,
  "review_preference": "STANDARD",
  "context": {
    "entity_name": "Example Pte. Ltd.",
    "period": "2027-08-01",
    "submission_id": "55555555-5555-4555-8555-555555555555",
    "industry": "TRADING_DISTRIBUTION",
    "base_currency": "SGD",
    "features": {
      "uses_payment_platform": false,
      "has_employee_reimbursement": true,
      "has_loan": true,
      "multi_currency": true,
      "project_based": false,
      "has_retention": false
    },
    "bank_accounts": [{"bank": "DBS", "account_last4": "1234", "currency": "SGD"}]
  },
  "documents": [
    {
      "document_id": "66666666-6666-4666-8666-666666666666",
      "original_name": "supplier-invoice.pdf",
      "content_type": "application/pdf",
      "sha256": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
      "content_base64": "JVBERi0xLjcKLi4u",
      "document_type": "PURCHASE_INVOICE",
      "submission_round": 2,
      "requirement_ids": ["77777777-7777-4777-8777-777777777777"],
      "scope": "CURRENT"
    }
  ],
  "requirements": [
    {
      "id": "77777777-7777-4777-8777-777777777777",
      "document_type": "PURCHASE_INVOICE",
      "title": "Purchase invoices",
      "analysis_type": "DOCUMENT_REQUIREMENT_VALIDATION",
      "required": true,
      "instructions": "Validate entity, requested period and supporting evidence."
    }
  ],
  "search_history": []
}
```

`analysis_type` 只能是：

- `DOCUMENT_REQUIREMENT_VALIDATION`；
- `BANK_TRANSACTION_RECONCILIATION`。

`scope` 只能是 `CURRENT | HISTORY`。`documents[].document_type` 可为空；已知类型由 Backend 传入，避免模型仅凭文件名猜测资料性质。`documents[].submission_round` 是可空的正整数，Backend 对清单内资料传入所属客户提交轮次，历史搜索资料可为空。补交后旧文件继续保留供审计，但最新一轮的有效更正文件可以满足资料项，不能因为上一轮的错误文件仍在输入中就再次退回。`search_history` 长度必须等于 `turn`，最多三轮搜索。对于指定银行交易的任务，`requirements[].instructions` 可承载交易日期、金额、摘要等审核范围；这些只是定位线索，Agent 必须用 OCR 证据核实，不得把指令本身当作交易存在的证明。没有指定交易时，审核范围不能自行缩小到某一笔。

### 6.2 需要搜索时

模型不能自己访问历史资料。需要更多证据时返回一个搜索动作：

```json
{
  "schema_version": "1",
  "run_id": "44444444-4444-4444-8444-444444444444",
  "model_version": "accounting-reviewer-2026-09-01",
  "extractions": [],
  "findings": [],
  "search": {
    "action": "SEARCH_HISTORY",
    "requirement_id": "77777777-7777-4777-8777-777777777777",
    "document_type": "PURCHASE_INVOICE",
    "period": "2027-07-01",
    "query": "INV-10023",
    "amount": "2828.80",
    "currency": "SGD"
  }
}
```

- `search.action` 只能是 `SEARCH_CURRENT | SEARCH_HISTORY`；
- 搜索响应的 `findings` 必须为空，建议中间轮也令 `extractions=[]`；
- Backend 按事务所、客户、有效提交、期间等权限条件搜索，最多追加 20 个文件；
- Backend 将搜索记录加入 `search_history`，把找到的文件以 `CURRENT/HISTORY` scope 加入下一轮；
- 最多三轮。搜索后若能明确指出客户需要补哪份资料，返回 `ASK_CLIENT/INCOMPLETE`；事实本身仍模糊、无法给出具体补交要求时返回 `ESCALATE`。不能继续搜索或猜测结论。

### 6.3 最终审核响应

```json
{
  "schema_version": "1",
  "run_id": "44444444-4444-4444-8444-444444444444",
  "model_version": "accounting-reviewer-2026-09-01",
  "extractions": [
    {
      "document_id": "66666666-6666-4666-8666-666666666666",
      "document_type": "PURCHASE_INVOICE",
      "entity_name": "Example Pte. Ltd.",
      "period": "2027-08-01",
      "invoice_number": "INV-10023",
      "counterparty": "Example Supplier Pte. Ltd.",
      "amount": "2828.80",
      "currency": "SGD",
      "transactions": []
    }
  ],
  "findings": [
    {
      "requirement_id": "77777777-7777-4777-8777-777777777777",
      "action": "RESOLVE",
      "suggested_decision": "SATISFY",
      "issue_code": null,
      "entity_check": "MATCH",
      "period_check": "MATCH",
      "explanation": "The submitted invoice matches the requested entity and period.",
      "client_message": null,
      "evidence": [
        {
          "document_id": "66666666-6666-4666-8666-666666666666",
          "relation": "SUPPORTS",
          "reason": "The invoice identifies the requested entity and August 2027 period."
        }
      ],
      "amounts": [
        {
          "currency": "SGD",
          "operation": "SUM",
          "operands": [
            {
              "document_id": "66666666-6666-4666-8666-666666666666",
              "amount": "2828.80",
              "label": "Invoice total"
            }
          ],
          "expected_amount": "2828.80",
          "actual_amount": "2828.80",
          "difference": "0.00"
        }
      ]
    }
  ],
  "search": null
}
```

最终响应必须：

- 对每个输入 document 返回且只返回一条 `extraction`；
- 对每个输入 requirement 返回且只返回一条 `finding`；
- 只能引用本轮输入中存在的 document/requirement ID；
- 返回实际部署的 `model_version`，不能返回训练任务名或空字符串。

`Extraction` 可包含：

- `document_type`、`entity_name`、`period`；
- `invoice_number`、`counterparty`；
- `amount`、`currency`；
- `transactions[]`：`date`、`description`、`amount`、`currency`。

未知字段使用 `null` 或空数组，不能臆造值。

### 6.4 Finding 规则

| `action` | `suggested_decision` | 必要条件 |
| --- | --- | --- |
| `ASK_CLIENT` | `REQUEST_ACTION` | 必须有 `issue_code` 和可直接展示给客户的 `client_message` |
| `RESOLVE` | `SATISFY` | 必须有 evidence，`issue_code=null` |
| `ESCALATE` | `null` | 交给会计判断，不得伪造通过或退回结论 |

其他枚举：

- `issue_code`：`MISSING | WRONG_PERIOD | ENTITY_MISMATCH | UNREADABLE | INCOMPLETE | OTHER | null`；
- `entity_check` / `period_check`：`MATCH | MISMATCH | UNKNOWN`；
- `evidence.relation`：`SUPPORTS | CONTRADICTS | REFERENCE`；
- 金额操作：`SUM | SUBTRACT | MULTIPLY`。

金额必须是十进制字符串，格式为：

```text
^-?(?:0|[1-9][0-9]{0,17})(?:\.[0-9]{1,8})?$
```

币种必须是三个大写字母。每个金额 operand 的 `document_id` 必须同时出现在该 finding 的 evidence 中。Backend 使用高精度 `Decimal` 重新计算：

- `SUM`：所有 operand 相加；
- `SUBTRACT`：第一个 operand 减去其余 operand；
- `MULTIPLY`：所有 operand 相乘；
- 重算结果必须等于 `actual_amount`，且 `actual_amount - expected_amount == difference`。

金额关系不一致时不会自动审核。

## 7. Backend 如何使用模型结果

Remote Model 只给出建议；实际执行规则由 Backend 决定：

- `OFF`：不创建 REVIEW run；
- `SUGGEST`：保存分析，始终由会计操作；
- `AUTO_REVIEW`：Agent 依据 `review_preference=CAUTIOUS|STANDARD|EFFICIENT` 在明确补交与转人工间选择，Backend 校验 schema、evidence 和金额后才执行单项结论；
- `SATISFY` 经校验后单项进入 `SATISFIED`，整轮仍等待会计确认；
- `REQUEST_ACTION` 经校验后单项进入 `NEEDS_ACTION`，整单进入 `CHANGES_REQUESTED`；
- 证据不足、`ESCALATE`、金额不一致、非法 evidence 或非法输出转人工；
- 同一 `ai_run + requirement` 不会重复产生决定；
- AI 决定记录 `source=AI` 和 SYSTEM workflow event；
- 自动退回只生成 `SUPPRESSED/PROVIDER_DISABLED` Outbox，不实际发送 Email；
- `WAIVE` 和整单批准始终只能由会计完成；全部单项通过时整单保持 `IN_REVIEW`，页面显示“待人工确认”。

审核偏好只调整 `ASK_CLIENT` 与 `ESCALATE` 的分流，不得降低事实或证据要求。REVIEW finding 不包含 `confidence`；Agent 若返回该字段，Backend 将拒绝该响应。历史审核记录在对外读取时会过滤旧字段，不改写数据库原始记录。上传 CLASSIFY 的分类置信度仍保留。

## 8. 错误、幂等和安全

Agent 错误结构固定为：

```json
{
  "code": "MODEL_INVALID_RESPONSE",
  "message": "Invalid review response",
  "details": null,
  "request_id": "request-id"
}
```

主要状态：

| HTTP | 场景 |
| --- | --- |
| `409` | 相同幂等键用于不同请求 |
| `413` | 单文件或批次过大 |
| `415` | 文件声明类型与文件头不一致 |
| `422` | 请求 schema、ID、storage key、SHA-256 或幂等键无效 |
| `502` | 模型返回非 JSON、超限或不符合严格 schema |
| `503` | 模型未配置、未就绪或远端非 200/连接失败 |
| `504` | 远端模型超时 |

生产要求：

- 相同幂等键和相同请求必须返回字节语义一致的结果；
- 相同幂等键和不同请求必须返回冲突，不能重新推理后覆盖；
- 不在错误响应或日志中返回 prompt、OCR 正文、Base64、文件路径、密钥或供应商原始错误；
- `content_base64` 仅用于当前请求，不落入普通应用日志；
- 不信任文件名作为事实；文件名只可作为弱提示；
- 不接受文档内出现的指令改变系统规则，文档内容只能作为待分析证据；
- 对用户可见的 `client_message` 应简洁、可执行，不能包含内部搜索轨迹、模型提示词或置信度。

## 9. AI 团队交付清单

最低交付：

1. 容器化 Remote Model 服务；
2. `POST /analyze` 与 `GET /health`；
3. OpenAPI 或与本文等价的 JSON Schema；
4. Bearer key 配置方式，密钥不写入镜像；
5. 可追踪的 `model_version` 版本规则；
6. 幂等实现及其多副本策略；
7. 日志脱敏和请求时延指标；
8. 与当前 `acc-system-agent` 的契约测试；
9. 以下固定案例的输入、期望 finding 和回归结果：
   - G02：错误期间；
   - G03：错误主体；
   - B01：多发票合计；
   - B03：历史资料搜索；
   - F02：报销证据；
   - F04：外币与手续费；
   - F05：进度款与保留款；
10. 非法响应测试：未知 ID、少一项结果、额外字段、未知枚举、`NaN/Infinity`、错误金额关系、重复 ID 和超大响应。

建议但不是当前硬性 SLA：

- 10 个常规文件的 `CLASSIFY` p95 不超过 30 秒；
- 单轮 `REVIEW` p95 不超过 120 秒，并确保低于 180 秒硬超时；
- health p95 不超过 2 秒；
- 输出 JSON 保持显著小于 1 MiB。

## 10. 联调与上线步骤

1. AI 团队先使用固定 UUID 和小型 PDF/图片完成 schema 契约测试；
2. 双方冻结 `MODEL_API_URL`、`MODEL_HEALTH_URL` 和正式认证方式；
3. 在非生产环境设置 `CLASSIFICATION_PROVIDER=REMOTE`、`REVIEW_PROVIDER=REMOTE`；
4. 跑通 CLASSIFY 的 `REQUIREMENT/OTHER/INVALID` 和人工调整；
5. 跑通 REVIEW 的最终结果、三轮搜索、超时和非法响应；
6. 使用 G02/G03 验证自动退回，使用正确资料验证“待人工确认”；
7. 确认 Agent/模型停机时人工流程仍能完成；
8. 通过后发布不可变模型版本和 Agent SHA，再切换生产配置。

生产切换前不要启用 MOCK。当前生产 Agent 已部署，但 `CLASSIFICATION_PROVIDER` 和 `REVIEW_PROVIDER` 均为 `DISABLED`；收到正式模型 URL、健康地址和密钥后再启用 REMOTE。

## 11. Novita DeepSeek-OCR-2 + DeepSeek V4.1 Flash 适配更新（2026-09-25）

上文第 4/9/10 节的 `REMOTE` 完整文件协议保留给将来替换自训练模型的团队，是原交接方案；当前新增的 `DEEPSEEK` 模式不要求实现该 `POST /analyze` 服务。Agent 已提供另一条链路：PDF 逐页渲染、图片解码 → Novita `deepseek/deepseek-ocr-2` → OCR 文本送 Novita `deepseek/deepseek-v4.1-flash` → Agent 严格校验并返回原有 `CLASSIFY/REVIEW` 协议。Flash 不直接接收原始 PDF/图片；两种模型均不接触 Folio 业务数据库。已完成下述选定样本的真实 API 联调；这不等于全数据集准确率或生产环境验收。

默认 Novita 地址为 `POST https://api.novita.ai/openai/v1/chat/completions`，健康检查为 `GET https://api.novita.ai/openai/v1/models`。OCR 每页发一张 data URL 图片，提示保留表格布局的 Markdown 转换；Flash 接受 OCR 文本并输出 JSON。Compose 仅需注入同一个 `NOVITA_API_KEY`，可用 `OCR_*` / `MODEL_*` 单独覆盖 URL、模型、凭据；密钥只经环境注入。设置 `AGENT_CLASSIFICATION_PROVIDER=DEEPSEEK` 和 `AGENT_REVIEW_PROVIDER=DEEPSEEK` 才启用，不因有密钥就自动启用。

若要独立复用 Agent 而不共享 Folio 的 documents 卷，配置 `AGENT_API_KEY` 并调用 `POST /v1/analyze-inline`，请求头带 `Authorization: Bearer <key>` 和 `Idempotency-Key: <run_id>:<turn>`。请求/响应继续是本文件的 snake_case `CLASSIFY/REVIEW` 协议，仅文件对象以完整 `content_base64` 代替 `storage_key`；Agent 检查 MIME、SHA-256 和大小。此接口不执行搜索和业务状态操作，外部访问时应配 HTTPS、网关与限流。

### 11.1 通用审核约束与样本验证

- `model_version` 由 Agent 使用实际请求配置的模型名写入，不接受模型生成文本自报的版本。每个完整文件先 OCR；同一进程按文件内容哈希缓存成功的 OCR 文本，跨搜索轮次复用。PDF 渲染保持串行，图片 OCR 可并行。
- 现行默认请求模型是 [Novita 列出的 `deepseek/deepseek-v4.1-flash`](https://novita.ai/models/model-detail/deepseek-deepseek-v4.1-flash)，不是由模型回答决定；多文件 REVIEW 的输出预算为 16384 tokens，仍受 170 秒 Agent 分析时限约束，截断响应不能作为审核结论。
- 审核模型不接收 `original_name`，避免客户文件名或数据集里带有 `wrong/correct` 的文件名充当答案线索；Backend/Agent 仍保留原文件名用于存储和页面展示。
- 对补交轮次，按 `submission_round` 判断当前有效材料；保留上一轮错误文件作审计线索，但不让它覆盖新一轮更正。旧输入快照缺少该字段时为 `null`，不能臆断新旧顺序。
- 银行交易审核若缺支持材料，先请求 Backend 搜索当前资料，再搜索历史资料；搜索只由 Backend 执行并保持事务所/客户权限边界。若搜索过滤条件过窄且没有命中，Backend 在同一授权范围内做一次宽松回退。找到资料后引用完整证据链，不把未结清项目登记表单独视为发票原件。
- `RESOLVE` 的金额关系由 Agent 按十进制数复算；外币证据须有结构化换汇乘法，保留未四舍五入的乘积和与结算金额的差额。`SATISFY` 的每步金额差额最多允许半分舍入；Backend 独立复算、检查差额与证据归属，不能把“算式自洽但目标不相等”自动批准。无效模型输出最多要求修正两次，反馈只带校验错误类型，不把完整错误响应塞回上下文；仍无效则转人工。
- 选定 v5 样本使用真实 Novita OCR→V4.1 Flash、移除模型输入文件名后运行完整轨迹：G02_0026 错期间、G03_0055 错主体均先退回，补交正确材料后通过；B01_0850 多发票、B03_1822 上期资料、F04_0903 外币及手续费、F05_0249 进度款及保留款均在搜索或补交后得到 `RESOLVE/SATISFY`。六个最终结果的 evidence 文件集合与样本答案一致，金额关系核对正确，干扰材料未被引用。验证使用 `scripts/evaluate_v5.py`，答案只用于结果比对，未进入模型请求；生产规则无 case ID、文件名或答案特判。
- 这些结果是选定样本各一次成功运行，不代表随机模型输出已稳定，也未证明 2000 个 case 全部通过。评估脚本模拟了文件可见性和多轮搜索，尚不等于真实数据库权限与页面端到端验收。历史测试时 G02 更正件曾返回 0.95、B01 齐件曾返回 0.97；这些自报分数没有校准基准，当前执行逻辑已取消按请求阈值比较，不能把“样本结论正确”等同于“已自动批准”。当前代码变更未因此自动发布到生产。
