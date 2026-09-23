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
| B6 AI 审核与上线加固 | AI 策略、任务队列、Agent 接入、证据搜索、自动单项审核和 Outbox | F7、A1-A3 |

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
- 同一客户、同一期间只允许一个未取消请求；取消记录保留，且可为该期间重新创建请求；
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
| B3-A9 | 取消请求后，为同一客户、同一期间重新创建请求 | 取消记录继续保留；新请求创建成功；新请求未取消前再次创建返回 `COLLECTION_EXISTS` |

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
- 文件分类接口：首版根据文件名返回 `REQUIREMENT/OTHER/INVALID`、requirement id 和置信度，保持固定 schema 以便后续替换为真实 AI；
- `multipart/form-data` 分块上传、大小限制、扩展名/MIME/magic bytes 校验、SHA-256；
- `.part → QUARANTINED → AVAILABLE/FAILED/EXCLUDED` 生命周期；
- Worker 使用 `FOR UPDATE SKIP LOCKED`、`locked_by`、`locked_until`、attempts 和退避领取扫描任务；
- 生产环境启用 ClamAV；扫描通过前文件不可下载、预览或发送给 AI；
- 最终文件不可变，原始文件名只作展示，不参与磁盘路径；
- FastAPI 授权后使用 `X-Accel-Redirect`，Nginx `internal` location 只读发送最终文件；
- 当前及历史材料按客户、期间、资料类型查询；
- 每个请求同一时刻最多一个 DRAFT submission，提交后产生不可变轮次；
- submission 支持可选的客户备注，备注随轮次保存，历史轮次提交后不可修改；
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
| B4-A7 | 客户在草稿轮次上传多个文件、逻辑排除其中一个、填写备注并提交 | 只提交有效文档；备注随轮次保存；submission 变为 `SUBMITTED`，原轮次及文件不再可编辑 |
| B4-A8 | 提交多个文件名请求自动分类 | 返回与输入顺序一致的分类、requirement id 和置信度；不支持的类型或空文件返回 `INVALID`，其他未命中文件返回 `OTHER` |

### 7.4 阶段完成标志

关闭 AI、Email 和飞书后，客户仍能安全完成首次上传和提交，会计仍能查阅当前请求资料。

> B4 的按文件名模拟分类是当时的可验收占位实现。B6 接入后由 Agent 读取完整文件执行 `CLASSIFY`，不再使用该占位逻辑。

## 8. B5：审核、补交与审批闭环

### 8.1 阶段目标

完成从客户提交到 `READY_FOR_BOOKKEEPING`、撤回批准和关闭的全部人工流程，并落实参考业务案例要求的证据关系和审计历史。

### 8.2 交付内容

- `review_decisions` 表；
- `SUPPORTS`、`CONTRADICTS`、`REFERENCE` 多对多证据关系；
- 保存单项审核时自动记录当前轮资料项的可用文件；
- 单项 `SATISFY`、`REQUEST_ACTION`、`WAIVE` 审核动作；
- `client_message` 与 `internal_note` 分离；
- `IN_REVIEW → CHANGES_REQUESTED → IN_REVIEW` 多轮补交；
- portal 以各轮 link 叠加出当前有效文件视图；补交时排除旧文件通过新轮次 tombstone 实现，不修改历史 submission；
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
| B5-A2 | 会计退回后客户追加新文件并从本轮清单移除旧文件，再次提交 | 新文件追加展示，`round_no` 递增；旧 submission 不被修改，错误文件和公开说明仍可追溯 |
| B5-A3 | 多张发票共同支持一笔付款，或一张发票对应多笔付款 | 多份文件可关联同一 requirement，关系和审核决定可追溯 |
| B5-A4 | `REQUEST_ACTION` 缺少 issue_code/client_message，或 `WAIVE` 缺少原因 | 后端拒绝请求，不产生部分状态更新 |
| B5-A5 | 任一必填 requirement 未满足时调用 approve | 返回业务错误，请求保持 `IN_REVIEW` |
| B5-A6 | 全部必填项满足或豁免后批准 | 原子进入 `READY_FOR_BOOKKEEPING`，写入批准人、时间和事件 |
| B5-A7 | 两名会计分别退回和批准同一 version | 只能有一个成功；另一个收到 `409`，不会错误批准 |
| B5-A8 | 普通会计与 FIRM_ADMIN 分别尝试撤回批准 | 普通会计失败；管理员填写原因后回到 `IN_REVIEW` 并留下事件 |
| B5-A9 | 关闭请求后尝试重新打开或修改审核 | 被拒绝；可通过新请求继续更正 |
| B5-A10 | 客户读取请求详情和事件 | 只能看到公开说明，看不到 `internal_note` 和 AI 原始输出 |

### 8.4 阶段完成标志

没有 AI 和通知服务时，账户、创建、上传、补交、审核、批准与关闭已经形成完整可用的业务闭环。

### 8.5 本地验收候选（2026-09-22）

- 已实现单项满足、要求补交、豁免，以及公开说明和内部备注隔离；
- 已实现多轮补交、证据关系、历史材料搜索、批准、管理员撤回批准和关闭；
- 后端自动检查：`33 passed`，`uv run alembic check` 无模型漂移。

F6/B5 中“当前轮文件自动记录为审核证据”是人工闭环的已验收基线；B6/F7 接入 AI 后改为由 AI 或会计显式选择 evidence，保留原有历史决定。

## 9. B6：AI 审核与上线加固

### 9.1 阶段目标

在不改变 PostgreSQL 业务事实源的前提下接入 `acc-system-agent`：上传阶段只快速分类，客户提交后再完整审核；后端校验 Agent 输出后按请求阈值自动执行单项决定，整单批准仍由会计完成。

### 9.2 交付内容

- Alembic 增加 AI 策略、文档分析字段、`ai_runs`、AI 来源决定、SYSTEM 工作流事件和 Outbox；`collection_requests` 使用 `ai_mode`、`ai_satisfy_threshold`、`ai_request_action_threshold`，新请求默认 `AUTO_REVIEW + 0.980/0.980`，已有请求迁移为 `SUGGEST`；
- `requirements.analysis_type` 为内部分析提示，不是创建/编辑 API 的用户输入：银行对账单默认 `BANK_TRANSACTION_RECONCILIATION`，其他资料默认 `DOCUMENT_REQUIREMENT_VALIDATION`，创建/新增/修改/复制共用规则；银行对账不要求会计在 `criteria` 手填目标交易。提交后 REVIEW 才提取对账单中的交易并匹配完整支持资料，银行资料同时仍需做主体、期间和完整性检查；
- `documents` 增加 `document_type`、`entity_name`、`period` 和 `extracted_data`；上传分类只确认 `document_type`，其余字段在提交后审核写入；
- `ai_runs` 支持 `CLASSIFY | REVIEW` 和 `DRAFT | QUEUED | PROCESSING | SUCCEEDED | FAILED | CANCELLED`，保存模型版本、输入快照、输出、租约和错误；
- Worker 在短事务中使用 `FOR UPDATE SKIP LOCKED` 领取任务，Agent HTTP 调用在事务外执行，最多重试三次；
- 批量分类 run 提供创建、上传、启动、查询、确认和取消接口；未确认文件不进入正式清单，失败时允许客户手动分类；
- 客户提交时原子进入 `IN_REVIEW` 并创建 `REVIEW` run；提交前不做主体、期间、金额或证据审核；
- 按客户、期间、资料类型、编号、对手方和金额搜索当前/历史资料；Agent 最多发起三轮 `SEARCH_CURRENT/SEARCH_HISTORY`；
- 后端校验 Agent schema、run id、枚举、租户/客户 evidence 归属，并用 `Decimal` 重算结构化金额关系；
- 达到当前请求阈值的 `SATISFY/REQUEST_ACTION` 可生成 `source=AI` 的单项决定；`WAIVE`、低置信度、非法证据、计算不一致和 `ESCALATE` 转人工；
- AI 状态迁移产生 `actor_type=SYSTEM` 的工作流事件；人工覆盖继续记录真实用户 actor，二者都只追加；
- 人工审核显式提交 evidence 及其 `SUPPORTS/CONTRADICTS/REFERENCE` 关系，可追加新决定覆盖 AI 当前结果；
- 任一必填项被自动退回时，整单与决定、事件和 Outbox 在同一事务进入 `CHANGES_REQUESTED`；AI 永远不自动豁免或批准整单；
- `notification_outbox` 按邮箱为客户管理员和提交人去重；首版不实现发信 provider，任务直接记为 `SUPPRESSED/PROVIDER_DISABLED`；
- JSON 结构化日志、AI 积压/失败/磁盘/备份告警，以及 PostgreSQL/资料卷恢复演练。

### 9.3 可验收场景

| ID | 场景与操作 | 预期结果 |
| --- | --- | --- |
| B6-A1 | 批量上传后 Agent 返回分类，但客户尚未确认 | 文件不进入正式资料清单，不写入审核状态或提取字段 |
| B6-A2 | 客户确认、调整、取消或在 Agent 失败后手动分类 | 人工确认结果为准；取消不合入清单；AI 故障不阻塞提交 |
| B6-A3 | 客户未提交时检查文档 | 不执行主体、期间、金额、证据或历史搜索逻辑 |
| B6-A4 | G02/G03 错期间/错主体提交达到自动退回阈值 | 单项进入 `NEEDS_ACTION`，整单与决定、事件和 Outbox 原子进入 `CHANGES_REQUESTED` |
| B6-A5 | B01 多发票合计与 B03 历史资料搜索 | 证据关系正确，历史文件不被复制，不返回其他客户资料 |
| B6-A6 | F04 外币/手续费或 F05 进度款/保留款 | Backend 使用 `Decimal` 重算；模型差额错误时转人工而不改状态 |
| B6-A7 | finding 低于阈值、evidence 属于其他租户/客户或 Agent 返回非法 schema | 拒绝自动执行，run 进入可重试/失败或人工队列，无部分业务更新 |
| B6-A8 | Worker 领取 run 后被终止或重放同一 run | 租约过期后可重领；同一 `ai_run + requirement` 不产生重复决定 |
| B6-A9 | AI 将所有必填项判定为满足 | 各单项可进入 `SATISFIED`，但整单仍保持 `IN_REVIEW`，必须由会计批准 |
| B6-A10 | AI 自动要求补交 | 为有效客户管理员/提交人按邮箱去重创建 `SUPPRESSED/PROVIDER_DISABLED` Outbox，不实际发信 |
| B6-A11 | 会计显式选择 evidence 并提交新决定 | 新的人工决定成为当前结果，AI 和历史人工决定仍可追溯 |
| B6-A12 | Agent/远端模型停机后完成上传、手动分类、人工审核和批准 | 人工业务闭环继续可用，AI 失败不被显示成审批失败 |

### 9.4 B6.1 + A1 本地验收候选（2026-09-22，待用户验收）

本批只交付 AI 数据模型、配置 API 与 Agent 部署基线；B6.2–B6.4 的真实分类、搜索、自动决定和通知记录生成尚未启用。

| ID | 验收内容 | 预期结果 |
| --- | --- | --- |
| B6.1-A1 | 升级已有本地数据库 | revision 为 `0009_b6_ai_baseline`；既有请求为 `SUGGEST`，原业务状态、版本、文件和审核历史保留 |
| B6.1-A2 | 使用 API 创建、修改、复制草稿 | 新请求默认 `AUTO_REVIEW`、双阈值 `0.980`；复制保留策略；阈值范围 `0.500–1.000`、最多三位小数；发布后不可修改策略 |
| B6.1-A3（历史技术检查，现已调整） | 保存对账资料要求 | 初版曾要求手填 `analysis_type` 和 `criteria.target_transaction`；此设计已撤销，当前按业务资料类型设置内部提示，不要求手填交易 |
| B6.1-A4 | 校验 AI 任务、决定与事件约束 | AI run 受事务所/请求/提交轮次外键约束；同一 run/资料项不得重复决定；AI 不得 `WAIVE`；SYSTEM 无用户 actor，人工历史保留真实 actor |
| B6.1-A5 | 启动及停止 Agent | Agent live 为 200，未配模型时 ready 返回 `MODEL_NOT_CONFIGURED`；停止 Agent 后 Backend ready 和既有人工流程仍可用 |
| B6.1-A6 | 检查 Agent 容器 | 无宿主机端口，资料卷只读，无 PostgreSQL/Redis/JWT 凭据；文件越界、符号链接及错误 SHA-256 被拒绝 |

- 本地后端与 Agent 镜像版本：`b6.1-a1-local`；Vite 仍使用 `http://localhost:5173` 并代理本地后端。
- 自动检查：后端 **36 passed**，Agent **15 passed**；Alembic 无模型漂移；已验证旧 schema 升级、空 AI 数据时迁移往返和策略迁移规则。
- 本地实际迁移保留 5 个请求、32 条人工审核记录；5 个既有请求均为 `SUGGEST / 0.980 / 0.980`，AI run 和 Outbox 均为空。
- 已验证 Backend → Agent 的 live/ready、Agent 只读挂载、无数据库凭据，以及 Agent 停机后 Backend 仍 ready。
- 当前没有远端模型 URL/协议/密钥，模型成功、超时、非法响应由测试 transport 验证；A1 `/v1/analyze` 只验证请求和文件边界，配置模型后仍明确返回 `ANALYSIS_NOT_IMPLEMENTED`，不伪造分类或审核成功。
- Agent ECR Action、IAM/OIDC 模板已编写；本批未推送代码、未实际发布 Agent ECR 镜像，未修改线上 AWS 配置。
- 本地操作命令见后端仓库 `deploy/README.md` 的 **Agent baseline (B6.1 + A1)**；本次数据库备份位于 `/tmp/acc-b61-backup-NXtVdJ/database.dump`（临时目录，不作为长期备份）。

### 9.5 B6.2 + A2 + F7.1/F7.2 本地功能（2026-09-22，用户验收通过）

B6.1/A1 记录保留为历史技术检查，本次向用户交付页面功能，不要求用户调用 API。迁移 `0010_classification_confirmation` 为 run 增加 `confirmed_at` / `confirmation`，分析结果和用户确认分开保存。

- `POST /api/v1/portal/collection-requests/{id}/classification-runs` 创建 DRAFT；现有上传接口附 `classification_run_id` 暂存完整文件，不创建 submission/link。
- 同一 run 的 `GET` 查询，以及 `POST /start`、`/cancel`、`/manual`、`/confirm`。状态为 `DRAFT → QUEUED → PROCESSING → SUCCEEDED/FAILED`，取消为 `CANCELLED`；合入使用独立 `confirmed_at`，不添加新状态。
- Worker 等安全扫描完成后才调用 Agent；有租约、有限重试和过期结果丢弃。取消/改为手动后到达的旧结果不能覆盖 run。
- 人工确认逐文件 `REQUIREMENT/OTHER/INVALID`，严格校验所有 document id、资料项归属、可编辑状态、扫描结果；同样的确认幂等重放，不同确认返回冲突。
- 分类输出只含类别、目标、文档类型、置信度；客户看不到原始 Agent 输出、storage key、内部上下文或密钥。无 REVIEW run、自动审核决定、通知发送。
- 取消保留已扫描但未关联的文件作为暂存记录，不进入清单/提交；当前未实现孤立暂存文件清理，后续按保留策略增加清理任务，不直接删除共享去重文件。
- 本地 `AGENT_CLASSIFICATION_PROVIDER=MOCK`，镜像版本 `b6.2-a2-local`；生产默认 DISABLED，Agent 拒绝生产 MOCK。真实模型接入仍待 URL/协议/密钥，非本次页面验收前置条件。
- 数据库迁移前备份：`/tmp/acc-b62-backup-rSDmj2/database.dump`。未推送代码、未变更线上。
- 交互修正：创建、编辑不接收 `analysis_type`，也不再依赖目标交易字段；原请求的内部字段和历史 `target_transaction` 不批量重写。复制时重新计算分析提示并去掉旧单笔交易目标。后续 REVIEW 必须根据本次提交的银行对账单提取交易，不把历史手填目标当作本轮事实。
- 自动检查：后端 **41 passed**、Agent **23 passed**、前端 **46 passed**，前端 lint/类型检查/构建通过，Alembic 无模型漂移；覆盖扫描前不分类、失败手动确认、取消后迟到结果丢弃、幂等合入、陌生客户拒绝、已满足项拒绝修改。
- 本地 Compose 实测两个完整 PDF 经扫描后由 Agent 返回 `MOCK/SUCCEEDED`；取消前后清单均为 0 文件。另一次确认重放两次后仍仅 2 个文件，submission 为 DRAFT、请求为 OPEN，未触发完整审核。
- 浏览器已走通创建/配置/保存/回显/发布及客户正式清单展示；浏览器扩展未开放本地文件访问，自动文件选择被权限阻止，因此 Dialog 的完整浏览器上传操作尚待补测；其状态交互已由前端自动测试覆盖，不将此项写为浏览器通过。

### 9.6 B6.3 + A3 + F7.3 第一批（2026-09-23，用户验收通过）

- 提交事务中原子创建 REVIEW run（OFF 除外）；不可因 AI 失败回滚客户提交。输入快照包含截止本轮的有效已提交文件和内部分析提示，不使用历史手填 `target_transaction`。
- Worker 单轮 210 秒租约、Agent HTTP 180 秒超时，每轮最多 3 次尝试；最多 3 轮搜索，持久化当前输入快照、搜索上下文和终态输出。过期 worker 不覆盖新租约结果；请求状态或最新提交改变时丢弃迟到结果。
- 同一客户、同一事务所、有效已提交文件范围内按资料类型、期间、文件名/编号/主体/对手方以及金额/币种搜索；历史仅早于当前请求期间。单次最多追加 20 文件，总计 100 文件/100 MiB。金额查询目前按规范十进制字符串精确匹配，不做模糊容差或向量搜索。
- Backend-Agent REVIEW 的严格 snake_case 协议见两个仓库一致的 `app/analysis_schemas.py`。最终结果必须覆盖全部输入资料项/文件，未知 id、跨客户/事务所证据、非法枚举和非十进制金额拒绝；每个金额操作数必须关联 evidence 文件。
- `SUM/SUBTRACT/MULTIPLY` 使用高精度 Decimal 重算 actual 和 difference；不一致记录 `AMOUNT_MISMATCH`，低于请求阈值记录 `LOW_CONFIDENCE`。算术通过不等于原始凭证真实或财务结论成立。
- 成功后持久化模型版本、提取字段、findings、证据和搜索轮次；不改 requirement/collection 审核状态，不创建 ReviewDecision 或 Outbox。本批仅分析建议；B6.4 再启用自动决定与人工显式 evidence。
- 员工 `GET /collection-requests/{id}/review-runs` 查询（不返回 storage_key 或输入快照）；`POST /collection-requests/{id}/review-runs/{run_id}/retry` 仅重试当前提交失败 run，保留旧记录并复用正在处理的新 run。客户端只返回 PROCESSING/AWAITING_ACCOUNTANT。
- 无新增数据库迁移，使用 `0010_classification_confirmation`。本地镜像 `b6.3-a3-local`；`AGENT_REVIEW_PROVIDER=MOCK`，生产默认 DISABLED 且禁止 MOCK。未提交、未发布线上。
- 用户验收覆盖 AI 分析状态、单项建议一键填写、审核结论重新编辑、全部资料审核完成前禁止下一步、旧补交请求中未审核资料的兼容恢复，以及底部浮动审批操作条。
- 自动检查：Backend **52 passed**，Agent **33 passed**，Frontend **49 passed**，前端 lint/类型检查/构建通过；覆盖超时重试、旧租约、迟到结果、金额差错、三轮搜索上限、跨客户/事务所隔离、OFF 模式、轮次切换和展示转义。
- Compose 实测：隔离客户 `AI Review Demo` 的 2027 年 3 月提交 10 份参考样例，搜索找到 2 月请求中的 1 份历史文件；生成 7 个 finding，状态仍 IN_REVIEW，无 AI 审核决定。本地会计账号 `review@test.com / review123`，原验收客户账号 `ai@test.com / ai123`；模型数据均为明确标注的固定模拟场景，不是数据集准确率验收。

### 9.7 阶段完成标志

系统完成上传快速分类、提交后完整审核、自动单项决定和人工兜底；整单批准仍由会计完成。Email/飞书 provider、向量数据库、SQLAdmin、S3、Celery、PostgreSQL RLS 和自定义流程引擎继续延后。

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
