# 会计事务所资料收集系统 - 后端开发文档

## 1. 文档用途

本文把[系统架构设计](./README_系统架构设计.md)拆成可执行的后端开发阶段。每个阶段必须独立可部署、可验证；未通过当前阶段验收，不进入下一阶段。

后端仓库：`acc-system-backend`

技术基线：

- Python 3.12、uv、FastAPI、Pydantic；
- SQLAlchemy 2、Alembic、psycopg 3、PostgreSQL；
- redis-py、Redis；
- pytest、FastAPI TestClient/httpx；
- Docker、Docker Compose、Nginx、GitHub Actions、Amazon ECR。

首版使用同步 SQLAlchemy 与 psycopg，API 的同步路由由 FastAPI 线程池执行。不要在同一业务链路混用同步和异步数据库会话；实际压测证明数据库等待成为瓶颈后，再评估 async SQLAlchemy。

## 2. 开发与验收规则

### 2.1 通用完成标准

每个阶段同时满足以下条件才算完成：

1. Alembic 可以从空数据库升级到最新版本，连续执行两次不会失败；
2. 独立测试环境内 `uv run pytest` 全部通过，测试数据库使用 PostgreSQL，不用 SQLite 代替；
3. OpenAPI 与实际响应一致，接口错误统一为 `{ code, message, details, request_id }`；
4. 新增写接口至少覆盖成功、无权限、非法状态和重复请求；
5. 日志不包含密码、JWT、Cookie、文件内容或 AI 原始敏感数据；
6. 容器镜像可以由固定 Git SHA 重建，依赖锁定在 `uv.lock`；
7. 验收记录保存测试日期、提交 SHA、镜像标签、执行人和失败截图/日志。

### 2.2 数据库规则

- 只通过 Alembic 修改结构，不在应用启动时调用 `create_all()`；
- 业务时间使用 `timestamptz` 并保存 UTC；
- 金额使用 `numeric(20, 4)`，不使用浮点数；
- 工作流状态使用文本列加 `CHECK`；
- 所有租户查询从当前认证主体取得 `firm_id`，不相信请求体中的租户字段；
- 工作流实体通过 `version_id_col` 防止旧数据覆盖，并用数据库行锁保护跨表完成条件。

### 2.3 建议命令

```bash
uv sync --frozen
uv run alembic upgrade head
uv run pytest tests/test_health.py
uv run uvicorn app.main:app --reload
docker compose --env-file deploy/.env.prod -f deploy/compose.yml up -d
```

生产环境变量文件只保存在服务器，不提交 Git。

B2 账户测试会清空数据库和 Redis，完整 `uv run pytest` 必须使用后端 README 中的独立测试容器：`ENVIRONMENT=test`、数据库名 `acc_test`、Redis DB `15`。禁止对开发验收或生产数据库执行；测试入口会先验证这三个条件。

## 3. 阶段总览

| 阶段 | 后端交付结果 | 前端依赖 |
| --- | --- | --- |
| B1 部署基线 | 镜像、Compose、Nginx、迁移、健康检查、ECR 发布可用 | F1 |
| B2 账户与租户 | JWT、邀请、用户、客户、成员和分配权限闭环 | F2、F3 |
| B3 收集请求 | 创建、发布、复制、查询、取消、事件与并发保护 | F4 |
| B4 文件与客户提交 | 隔离上传、扫描、下载、历史检索和首次提交 | F5 |
| B5 审核与审批 | 单项审核、多轮补交、证据关联、批准、撤回与关闭 | F6 |
| B6 通知、AI 与上线加固 | Outbox、Email/飞书适配、AI 接口、恢复和全量验收 | F6 |

B1 与 F1 是同一个部署阶段的两部分，可以并行开发；早期可暂用占位前端镜像验证反向代理，但阶段验收必须换成 F1 的真实镜像。

## 4. B1：部署基线

### 4.1 阶段目标

先打通从代码到服务器的完整发布链路。本阶段不实现账户和业务接口，只保证所有运行组件能构建、发布、迁移、启动、检查和回滚。

### 4.2 交付内容

后端仓库至少包含：

```text
app/
  main.py
  config.py
  db.py
  worker.py
migrations/
tests/
deploy/
  compose.yml
  .env.example
  nginx/
    Dockerfile
    nginx.conf
Dockerfile
.dockerignore
pyproject.toml
uv.lock
.github/workflows/build-image.yml
```

具体要求：

- `GET /api/v1/health/live` 只检查 API 进程；
- `GET /api/v1/health/ready` 检查 PostgreSQL 和 Redis，依赖异常时返回 `503`；
- Alembic 有可执行的初始 revision；
- backend、worker、migrate 使用同一个后端镜像，通过不同命令启动；
- Worker 首版只建立进程、信号处理、数据库/Redis 连接和轮询框架，不实现业务任务；
- Compose 启动 nginx、frontend、backend、worker、migrate、postgres、redis；ClamAV 使用可选 profile，B4 再启用；
- 只有 Nginx 映射宿主机端口，PostgreSQL 和 Redis 不映射公网端口；
- Nginx 将 `/api/` 转发至 backend，其他路径转发至 frontend；
- GitHub Actions 使用 OIDC 获取 AWS 临时凭证；后端仓库分别构建 backend 与 nginx 镜像；
- 镜像推送 Git SHA 不可变标签，可额外更新 `main` 标签，但部署只引用 SHA；
- 服务器按 `pull → migrate → up -d` 顺序发布。

阶段外内容：业务表、JWT、消息队列、对象存储和 Kubernetes 均不在 B1 实现。

### 4.3 验收前置

- AWS 中已创建 backend、frontend、nginx 三个 ECR repository；
- GitHub 仓库已配置 `AWS_ROLE_ARN`、`AWS_REGION` 和 ECR repository 变量；
- 验收服务器已安装 Docker Engine 与 Compose plugin；
- 服务器准备了生产 `.env`、域名和 TLS 证书；若暂时没有域名，可先在受限测试网络使用 HTTP 验收，其余项目不变。

### 4.4 可验收场景

| ID | 场景与操作 | 预期结果 |
| --- | --- | --- |
| B1-A1 | 本地构建 backend 与 nginx 镜像，再启动完整 Compose | 所有必要容器进入 healthy/running；migrate 容器以 0 退出 |
| B1-A2 | 请求 `/api/v1/health/live` 和 `/api/v1/health/ready` | 两者返回 `200`，响应带 `request_id`；ready 明确显示 PostgreSQL、Redis 可用 |
| B1-A3 | 暂停 Redis 后再次请求两个健康接口 | live 仍为 `200`，ready 为 `503`；恢复 Redis 后 ready 自动恢复 `200` |
| B1-A4 | 请求 `/` 与一个不存在的前端路由 `/login` | 均由前端容器返回 SPA；`/api/` 不会落到前端 fallback |
| B1-A5 | 从服务器外扫描公开端口 | 只有预期的 `80/443` 可访问，PostgreSQL、Redis、backend 和 worker 无公网端口 |
| B1-A6 | 连续执行两次 `docker compose run --rm migrate` | 两次均成功，第二次无待执行迁移且不修改已有结构 |
| B1-A7 | 向 `main` 推送一次仅后端变更和一次仅 Nginx 变更 | 对应 workflow 测试通过，ECR 出现正确的 Git SHA 镜像；不相关镜像无需重复构建 |
| B1-A8 | 服务器把 `.env` 中镜像标签固定为本次 SHA，执行发布命令 | 拉取、迁移和启动成功；重启服务器后服务与 PostgreSQL/Redis 数据卷仍存在 |
| B1-A9 | 检查镜像历史和容器环境 | 镜像中没有 `.env`、AWS 密钥、数据库密码或 JWT 密钥 |

### 4.5 阶段完成标志

使用一个新的 Git SHA，可以在空服务器上仅靠 Compose 和服务器环境变量恢复相同运行环境；后续阶段不再改变基本部署拓扑。

## 5. B2：账户、JWT 与租户权限

### 5.1 阶段目标

完成事务所员工和客户联系人的账户闭环，并建立后续所有接口共用的租户授权边界。

### 5.2 交付内容

- `firms`、`users`、`firm_members`、`clients`、`client_members`、`client_assignments`、`user_invites` 表及复合外键；
- 一个账号在 MVP 中只能属于一个事务所，同一事务所内可关联多个客户；
- Argon2id 密码哈希、短期 access JWT、HttpOnly refresh JWT Cookie；
- Redis refresh session、轮换、撤销、登录/重置限流；
- `current_principal` 依赖，以及 FIRM_ADMIN、ACCOUNTANT、CLIENT_ADMIN、CLIENT_SUBMITTER 权限检查；
- 登录、refresh、logout、me、修改密码、邀请接受和密码重置接口；
- 员工、客户、客户成员和会计分配管理接口；
- 密码重置申请使用统一响应，token 只保存哈希且只能消费一次；
- 所有按 id 查询先限定 `firm_id`/`client_id`，越权资源统一按既定策略返回 `403` 或 `404`。

### 5.3 可验收场景

| ID | 场景与操作 | 预期结果 |
| --- | --- | --- |
| B2-A1 | 管理员邀请会计，会计使用一次性 token 设置密码并登录 | 邀请只可使用一次；登录返回 access JWT 并设置 refresh Cookie；`GET /me` 返回正确角色和事务所 |
| B2-A2 | access JWT 过期后调用 refresh，再用旧 refresh JWT 重试 | 首次刷新成功并轮换 token；旧 refresh JWT 失效 |
| B2-A3 | 用户登出后继续使用旧 access/refresh token | refresh 失败；同一 `sid` 的受保护请求也失败 |
| B2-A4 | 管理员禁用某用户或用户修改密码 | 该用户已有 refresh session 全部撤销，后续登录遵循最新状态 |
| B2-A5 | ACCOUNTANT 访问未分配客户，客户端用户猜测其他客户/事务所 id | 不返回目标数据，也不能通过写接口建立跨租户关系 |
| B2-A6 | CLIENT_ADMIN 邀请本客户联系人；CLIENT_SUBMITTER 尝试同一操作 | 前者成功，后者返回权限错误 |
| B2-A7 | 对存在和不存在的邮箱申请密码重置 | HTTP 状态和响应正文一致，不能据此枚举账号 |
| B2-A8 | 对登录、邀请接受、密码重置连续超限请求 | 返回明确限流错误；正常用户和管理员仍可从日志定位 `request_id` |

### 5.4 阶段完成标志

后续路由只需依赖 `current_principal` 和资源范围检查，不再自行解析 JWT 或复制角色判断。

### 5.5 本轮验收记录（2026-09-17）

- 交付提交：`9221ffb1af062cc5c977c58ad460832a347b5e0a`；配套前端 `3801ff0c93a5a5922426fe666b221408c6cc2dbc`。
- 在独立临时 PostgreSQL / Redis 上执行全量 pytest：**16 passed**；从空库连续两次 `alembic upgrade head` 成功，第二次无待执行迁移。
- 覆盖邀请单次/过期/替换、生产重置统一响应、重置与改密/禁用后的多会话撤销、登出旧 token、租户/分配/客户端角色限制、认证限流与 request ID。
- 补齐 refresh 会话索引续期、登出撤销失败返回 503、422 不回显密码/token 输入；测试在操作数据库前拒绝非测试配置。
- Compose 后端信任内部代理头，Nginx 覆盖外部传入的 X-Forwarded-For；生产需 HTTPS、正确的 `FRONTEND_ORIGIN` 和 `COOKIE_SECURE=true`。
- 本地后端/Worker 镜像 `acc-system-backend:local` 使用提交 SHA 构建；数据库和管理员密码保留，本轮不更新 Lightsail。
- 邀请 token 暂由授权接口返回，重置 token 仅非生产环境返回；Email 实际发送、并发压力测试和生产上线加固不计入本轮已验证内容。

## 6. B3：收集请求与资料要求

### 6.1 阶段目标

让会计人员能够创建、编辑、发布、复制、查看和取消月度收集请求，并建立可靠的状态、事件、幂等和并发基础。

### 6.2 交付内容

- `collection_requests`、`requirements`、`workflow_events`、`idempotency_records` 表；
- 请求与 requirements 在同一事务创建；
- `DRAFT → OPEN` 发布流程，及 DRAFT/OPEN/IN_REVIEW/CHANGES_REQUESTED 的取消规则；
- 请求列表分页与客户、期间、状态、负责人、截止日筛选；
- 复制上月请求时只复制仍适用的要求，不复制文档、审核结论和事件；
- 已发布要求不物理删除；审核阶段新增项标记 `FOLLOW_UP`；
- `collection_requests.version` 与 `requirements.version` 乐观并发控制；
- 发布、取消、复制等写操作使用 `Idempotency-Key`；
- 当前状态与 workflow event 在同一事务提交；notification outbox 到 B6 再加入同一事务。

### 6.3 可验收场景

| ID | 场景与操作 | 预期结果 |
| --- | --- | --- |
| B3-A1 | ACCOUNTANT 为已分配客户创建请求和三项 requirement | 一次事务全部成功；任一 requirement 非法时整笔回滚 |
| B3-A2 | 未分配会计创建请求；FIRM_ADMIN 执行同一操作 | 未分配会计失败，管理员成功 |
| B3-A3 | 编辑 DRAFT 后发布，再尝试修改或删除初始要求 | 发布成功并写事件；发布后的非法编辑被拒绝 |
| B3-A4 | 使用同一 `Idempotency-Key` 重复发布同一请求 | 返回第一次结果，只产生一次状态变化和一次发布事件 |
| B3-A5 | 使用相同 key 发送不同请求体 | 返回 `409`，原请求结果不变 |
| B3-A6 | 两个客户端基于同一 version 修改请求 | 第一个成功并递增 version；第二个收到 `409` 和最新资源 |
| B3-A7 | 复制上月请求到新期间 | requirements 被复制，文档、审核决定和历史事件未复制；同客户同期间不能重复创建 |
| B3-A8 | 按客户、期间、状态和截止日期组合筛选列表 | 只返回当前事务所且符合条件的数据，分页总数正确 |

### 6.4 阶段完成标志

会计人员可以发布一张真实可访问的月度资料清单，且重复点击、并发编辑和跨租户请求不会破坏状态。

### 6.5 本地验收候选（2026-09-17，待用户验收）

- 本地 Compose 已执行 `0004_b3_collections` 迁移并重建 backend/worker，`/api/v1/health/ready` 正常；未更新 Lightsail。
- 已覆盖请求与要求原子创建、客户分配权限、草稿编辑、发布/取消、幂等重放、版本冲突、复制和组合筛选。
- 自动检查：后端 `uv run pytest -q` **28 passed**，`uv run alembic check` 无模型漂移。

## 7. B4：文件、客户门户与首次提交

### 7.1 阶段目标

完成客户从看到资料要求、上传文件到首次提交审核的闭环，并使文件在扫描、存储、历史搜索和下载过程中保持安全和可恢复。补交轮次在 B5 的退回流程中验收。

### 7.2 交付内容

- `submissions`、`documents`、`requirement_documents` 表及文件处理租约字段；
- portal 请求列表/详情，仅返回当前客户可见字段；
- `multipart/form-data` 分块上传、大小限制、扩展名/MIME/magic bytes 校验、SHA-256；
- `.part → QUARANTINED → AVAILABLE/FAILED/EXCLUDED` 生命周期；
- Worker 使用 `FOR UPDATE SKIP LOCKED`、`locked_by`、`locked_until`、attempts 和退避领取扫描任务；
- 生产环境启用 ClamAV；扫描通过前文件不可下载、预览或发送给 AI；
- 最终文件不可变，原始文件名只作展示，不参与磁盘路径；
- FastAPI 授权后使用 `X-Accel-Redirect`，Nginx `internal` location 只读发送最终文件；
- 当前及历史材料按客户、期间、资料类型查询；
- 每个请求同一时刻最多一个 DRAFT submission，提交后产生不可变轮次；
- 客户提交时把请求从 `OPEN/CHANGES_REQUESTED` 原子迁移到 `IN_REVIEW` 并写入事件；
- portal 使用独立 Pydantic 响应模型，不返回 `internal_note`、AI 原始输出或其他客户数据。

### 7.3 可验收场景

| ID | 场景与操作 | 预期结果 |
| --- | --- | --- |
| B4-A1 | 客户上传一个允许类型的文件 | 接口返回 `202` 和 document id；初始为 `QUARANTINED`，扫描后变为 `AVAILABLE` |
| B4-A2 | 上传超限、伪造扩展名、危险类型和 EICAR 测试文件 | 请求或扫描失败，文件不可下载；失败原因可审计但不泄漏服务器路径 |
| B4-A3 | 在扫描完成前调用下载，或直接访问 Nginx 内部文件路径 | 两种方式都无法取得文件 |
| B4-A4 | 扫描通过后由有权限用户下载，再由其他客户用户猜测相同 document id | 前者获得原文件且哈希一致；后者无权访问 |
| B4-A5 | Worker 移动文件后、回写状态前被终止 | 租约过期后任务重试，依据最终文件和哈希补写 `AVAILABLE`，不产生第二份原件 |
| B4-A6 | 同一客户再次上传相同 SHA-256 文件并关联另一要求 | 系统提示重复并允许关联已有证据；不同事务所绝不复用文件 |
| B4-A7 | 客户在草稿轮次上传多个文件、逻辑排除其中一个并提交 | 只提交有效文档；submission 变为 `SUBMITTED`，原轮次不再可编辑 |
| B4-A8 | 会计按历史期间和 document type 搜索 | 可找到当前客户历史资料，不会把历史文件复制成新文件，也不会返回其他客户资料 |

### 7.4 阶段完成标志

关闭 AI、Email 和飞书后，客户仍能安全完成首次上传和提交，会计仍能查阅当前及历史资料。

## 8. B5：审核、补交与审批闭环

### 8.1 阶段目标

完成从客户提交到 `READY_FOR_BOOKKEEPING`、撤回批准和关闭的全部人工流程，并落实参考业务案例要求的证据关系和审计历史。

### 8.2 交付内容

- `review_decisions` 表；
- `SUPPORTS`、`CONTRADICTS`、`REFERENCE` 多对多证据关系；
- 单项 `SATISFY`、`REQUEST_ACTION`、`WAIVE` 审核动作；
- `client_message` 与 `internal_note` 分离；
- `IN_REVIEW → CHANGES_REQUESTED → IN_REVIEW` 多轮补交；
- 审核中新增 `FOLLOW_UP` 要求时自动退回客户；
- 只有全部必填项为 `SATISFIED/WAIVED` 才能批准；
- `READY_FOR_BOOKKEEPING → IN_REVIEW` 仅 FIRM_ADMIN 可撤回并填写原因；
- `READY_FOR_BOOKKEEPING → CLOSED`，关闭后更正必须创建新请求；
- 状态迁移统一按“请求 → 要求 → 文档”加锁，并写入不可变事件；
- 所有审核接口检查 version，防止两名会计互相覆盖。

### 8.3 可验收场景

| ID | 场景与操作 | 预期结果 |
| --- | --- | --- |
| B5-A1 | 客户依次提交缺失、错主体、错期间和正确文件 | requirement 依次保持/进入 `NEEDS_ACTION`，最后可变为 `SATISFIED`；全部文件和每轮决定均保留 |
| B5-A2 | 会计退回后客户创建下一轮 submission 并再次提交 | `round_no` 递增，前几轮、错误文件和公开说明仍可追溯 |
| B5-A3 | 多张发票共同支持一笔付款，或一张发票对应多笔付款 | 多份文件可关联同一 requirement，关系和审核决定可追溯 |
| B5-A4 | 使用上期发票支持本期付款 | 通过 `REFERENCE/SUPPORTS` 关联历史文档，不复制文件 |
| B5-A5 | `REQUEST_ACTION` 缺少 issue_code/client_message，或 `WAIVE` 缺少原因 | 后端拒绝请求，不产生部分状态更新 |
| B5-A6 | 任一必填 requirement 未满足时调用 approve | 返回业务错误，请求保持 `IN_REVIEW` |
| B5-A7 | 全部必填项满足或豁免后批准 | 原子进入 `READY_FOR_BOOKKEEPING`，写入批准人、时间和事件 |
| B5-A8 | 两名会计分别退回和批准同一 version | 只能有一个成功；另一个收到 `409`，不会错误批准 |
| B5-A9 | 普通会计与 FIRM_ADMIN 分别尝试撤回批准 | 普通会计失败；管理员填写原因后回到 `IN_REVIEW` 并留下事件 |
| B5-A10 | 关闭请求后尝试重新打开或修改审核 | 被拒绝；可通过新请求继续更正 |
| B5-A11 | 客户读取请求详情和事件 | 只能看到公开说明，看不到 `internal_note` 和 AI 原始输出 |

### 8.4 阶段完成标志

没有 AI 和通知服务时，账户、创建、上传、补交、审核、批准与关闭已经形成完整可用的业务闭环。

## 9. B6：通知、AI 接口与上线加固

### 9.1 阶段目标

在不改变业务事实源的前提下接入异步通知和 AI 建议接口，并完成故障恢复、备份和发布前回归。

### 9.2 交付内容

- `notification_outbox`、`ai_runs` 表和租约字段；
- Worker 短事务领取任务，外部 Email/飞书/AI 调用不持有数据库锁；
- `REQUEST_PUBLISHED`、`DUE_SOON`、`OVERDUE`、`CHANGES_REQUESTED`、`READY_FOR_BOOKKEEPING` 事件；
- 先实现一个 Email provider；飞书只实现相同接口或配置关闭，不阻塞首版；
- `(firm_id, dedupe_key)` 唯一约束和发送前状态复核；
- Outbox 明确为至少一次投递，渠道支持幂等键时透传，但不承诺跨系统 exactly-once；
- `POST /collection-requests/{id}/ai-runs` 与 `GET /ai-runs/{id}`；
- AI 请求/响应 schema 校验、短期受限下载地址、超时、重试和失败状态；
- AI 输出只保存建议，不能直接修改 requirement 或 collection 状态；
- JSON 结构化日志、积压/失败/磁盘/备份告警；
- PostgreSQL 备份恢复和资料卷备份恢复演练；
- 对架构文档第 15 节及本文件全部场景执行回归。

### 9.3 可验收场景

| ID | 场景与操作 | 预期结果 |
| --- | --- | --- |
| B6-A1 | 发布请求并触发 Email 通知 | 业务状态与 outbox 同一事务提交；Worker 成功发送并标记 `SENT` |
| B6-A2 | Email 服务不可用后恢复 | 任务按退避策略重试，API 上传、审核和批准不受影响 |
| B6-A3 | Worker 领取通知或 AI 任务后被强制终止 | `locked_until` 过期后其他 Worker 可重领，attempts 和错误记录正确 |
| B6-A4 | 定时任务多次扫描同一逾期请求 | 同一提醒周期只产生一个 dedupe_key；接受外部发送成功但回写失败时可能重复投递 |
| B6-A5 | AI 返回合法建议 | `ai_runs` 保存输入快照和输出；requirement 状态不自动变化，必须由会计确认 |
| B6-A6 | AI 超时、不可用或返回非法枚举/其他客户文档 id | run 进入可重试或 FAILED，非法输出不落入业务状态，人工流程继续可用 |
| B6-A7 | 从 PostgreSQL 备份和资料卷备份恢复到空环境 | 请求、事件、文件元数据与原文件哈希一致，授权下载正常 |
| B6-A8 | 使用前一版本 SHA 执行回滚 | 应用可启动；数据库迁移遵循向后兼容策略，不需要危险的自动 downgrade |
| B6-A9 | 执行全量权限、并发、上传、审批和日志检查 | 所有阶段回归通过，日志与响应中无敏感字段泄漏 |

### 9.4 阶段完成标志

系统满足首版上线条件。飞书、SQLAdmin、S3、Celery、PostgreSQL RLS 和自定义流程引擎继续延后，只有真实需求或容量指标出现时再增加。

## 10. 阶段验收记录模板

每完成一个阶段，在 issue/release 中保存：

```text
阶段：B1/B2/...
提交 SHA：
backend 镜像：
nginx 镜像：
数据库 revision：
验收环境：
自动测试结果：
手工场景结果：
未完成项：
验收人：
日期：
```
