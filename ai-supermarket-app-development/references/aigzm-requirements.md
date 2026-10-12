# AIGZM / AI 超市开发约束

更新时间：2026-10-12。正式开发前必须重新读取官方实时文档，不要把下面的动态模型、数据接口清单或价格当作永久快照。

官方入口：

- 文档导航：https://aigzm.com/docs
- 机器索引：https://aigzm.com/llms.txt
- 完整接入文档：https://aigzm.com/llms-full.txt
- OpenAPI 契约：https://aigzm.com/openapi.json
- 开发者安全接入指南：https://aigzm.com/developer-integration-guide.md
- 实时模型索引：https://aigzm.com/v1/platform/models/documentation
- 实时模型 JSON：https://aigzm.com/v1/platform/models
- 模型价格：https://aigzm.com/v1/platform/models/pricing
- 实时数据 API 索引：https://aigzm.com/v1/market/documentation
- 数据 API JSON 索引：https://aigzm.com/v1/market/endpoints

## 文档优先级

字段、路径、请求体或响应结构冲突时，按以下顺序判断：

1. `openapi.json`：HTTP 路径、方法、字段、鉴权和响应契约；
2. 实时模型文档和逐模型文档：当前可用模型、`model_key`、分类、请求体与限制；
3. 实时数据 API 文档和逐接口文档：数据接口清单、调用路径、参数、积分和限制；
4. `developer-integration-guide.md`：业务流程、安全、OAuth、支付、Webhook、幂等和错误处理；
5. 官方示例：只用于理解调用方式，不能覆盖前四项。

不得根据字段名猜测未公开能力；不得自造接口、模型、会员列表、支付状态或跨应用用户查询。

## 身份和凭据

先根据产品目标选择一种身份模式，并写入设计决策：

- 开发者自己的模型或数据 API 消费：账号级 Developer API Key；
- 终端用户登录、读取其资料/会员/积分：OAuth 2.1 Authorization Code + PKCE；
- 代表终端用户消费其积分：OAuth Access Token + `points.spend`；
- 为终端用户发起充值/会员购买：OAuth Access Token + `billing.initiate`。

不要混用 Cookie、Developer API Key、Client Secret 和 OAuth Access Token。API Key 创建后只显示一次明文，必须服务端保存，并调用：

```http
POST /v1/developer/api-keys/{id}/confirm-delivery
Content-Type: application/json
{}
```

请求头只能二选一：`Authorization: Bearer <key>` 或 `X-API-Key: <key>`，不能同时发送。任何 Key、Secret、Access Token、授权码和敏感模型输入都不能出现在前端、URL、源码、浏览器持久存储或日志中。

OAuth 必须使用 PKCE S256；服务端校验 `state`、`nonce`、issuer 和 audience，并在服务端兑换授权码。应用内外部身份键使用 `issuer + sub`，不要用手机号、邮箱或平台内部 ID 绑定用户。

## 模型和数据 API

模型可用性必须从实时索引读取，不能保存静态模型白名单，也不能用展示标题代替 `model_key`。OpenAI 兼容入口的基本形态是：

- Base URL：平台 `/v1`；
- 对话/代码：`POST /v1/chat/completions`；
- Embeddings：`POST /v1/embeddings`；
- 图片生成：`POST /v1/images/generations`；
- 图片编辑：`POST /v1/images/edits`；
- 语音识别：`POST /v1/audio/transcriptions`；
- 每次业务请求生成唯一幂等键；同一业务重试复用同一个键和完全相同的请求体。

数据 API 的逐接口清单只从 `/v1/market/documentation` 或其 JSON 索引读取。实际调用路径使用索引中已经拼好的 `/v1/data/{supplier_slug}/{upstream_path}`；GET 参数放 query，POST 参数放 JSON。响应体原样透传上游，平台计费、接口键、预留单号和错误来源通过 `X-IdP-*` 响应头传递。

## 超时、错误和重试

默认设计和验收至少覆盖：

- 文本/代码：客户端超时不低于 120 秒；
- Embeddings：不低于 90 秒；
- 图片：不低于 300 秒，推荐 600 秒；
- 长语音：不低于 180 秒，推荐 300 秒；
- 数据 API：不低于 45 秒。

固定处理以下边界：

- `409 api_key_delivery_pending`：先完成 Key 激活，不当作平台故障；
- `400 ambiguous_credentials`：删除其中一套鉴权头；
- `400 unsupported_model_format`：按模型分类选择正确入口；
- `402 insufficient_balance`：余额不足，不实现信用额度兜底；
- `502 supplier_unavailable`：保留 `X-IdP-Reservation-Id` 等信息供核对，不盲目重试；
- 网络重试必须遵守幂等键和完全相同请求体。

## Sandbox、上线和 AI 超市展示

要发布到 AI 超市的应用，先创建 Sandbox，完成 Sandbox OAuth 与基础接入，再通过平台的正式申请流程提交审核。Sandbox 与正式环境的 OAuth、JWKS、Cookie、Redis、数据库、Webhook、日志、凭据和测试数据必须隔离；测试凭据不得迁移到正式环境。

应用配置至少核对：

- 生产使用 HTTPS 的 `homepage_url`；
- OAuth `redirect_uris` 完整匹配并加入白名单；
- 支付 `billing_return_uris` 完整匹配并加入白名单；
- 申请的 Scope 是已登记 Scope 的子集；
- 最低会员等级、应用名称和公开入口；
- Webhook 是服务端 HTTPS 地址，不是前端页面；
- 变更首页、Scope、回调白名单或支付回跳地址会触发下架、撤销 Token、重审和 Secret 轮换风险。

运行时应用只能处理当前完成授权的用户。不得拉取、搜索、导出、关联全量会员、其他应用用户或跨应用身份。平台公开应用列表只返回当前 `active` 应用的公开字段。

## 会员信息、钱包和用户隔离

会员和钱包只读取当前已授权用户，不能做会员列表、全量导出、按手机号/邮箱/用户名查询或跨应用同步。会员中心负责升级、续费和降级，当前不开放开发者直接修改会员等级的接口。

```http
GET /v1/oauth/userinfo
Authorization: Bearer <ACCESS_TOKEN>
```

需要 `identity` Scope。应用内部使用 `issuer + sub` 作为外部身份键，不使用手机号、昵称或邮箱作为唯一键。

```http
GET /v1/user/membership
Authorization: Bearer <ACCESS_TOKEN>
```

需要 `membership.read`，只返回实时 `member_level`。

```http
GET /v1/user/membership/details
Authorization: Bearer <ACCESS_TOKEN>
```

需要 `membership.read`，返回当前等级、套餐权益、当前购买周期和预约周期。会员详情来自当前数据库状态，禁止缓存。

```http
GET /v1/user/wallet
Authorization: Bearer <ACCESS_TOKEN>
```

需要 `points.read`，返回当前用户的 `sub`、会员等级、积分余额、冻结额、可用额和更新时间。`available` 已经是可用积分，接入方不得再次从 `balance` 中扣减；钱包响应禁止缓存。

## 支付、充值、会员购买和 Webhook

在开发者中心先把精确回跳地址加入 Client 的 `billing_return_uris`。创建支付入口需要 `billing.initiate`，请求体只能包含 `intent` 和 `return_uri`：

```http
POST /v1/user/billing-entries
Authorization: Bearer <ACCESS_TOKEN>
Content-Type: application/json

{ "intent": "recharge", "return_uri": "https://app.example.com/payment-complete" }
```

会员升级或续费把 `intent` 改为 `membership`。响应会返回 `entry_id`、短时单次 `billing_url` 和过期时间。浏览器跳转 `billing_url`，支付完成后用同一用户 Token 查询：

```http
GET /v1/user/billing-entries/{entry_id}
Authorization: Bearer <ACCESS_TOKEN>
```

只有查询结果确认 `payment_order.status=paid` 后，才能读取钱包或会员详情。回跳参数、`status=paid` 或 `status=consumed` 本身都不能当作最终支付凭证。`billing_url` 是不透明地址，不能解析、拼接参数或调用 `/mock-result`。

### AI 超市应用一次性购买

如果应用在 AI 超市按一次性购买收费，价格由平台审核、收款和退款，应用不自行接支付：

```http
POST /v1/developer/clients/{client_id}/commercial-offer
GET /v1/platform/apps/{client_id}
GET /v1/user/app-entitlement
Authorization: Bearer <ACCESS_TOKEN>
```

`GET /v1/user/app-entitlement` 不需要额外 Scope，只能查询当前应用的权益。根据 `access` 为 `granted` 或 `purchase_required` 决定是否放行；响应禁止缓存。应用一次性价格不包含模型 Token 消耗，模型调用仍按公开模型价格另计。退款后刷新 Token 可能返回 `400 invalid_grant`，应用应清理本地登录态并引导用户重新授权。

### Webhook 验签

每个 Client 可以配置一个 HTTPS 服务端 Webhook。事件可能包括 `billing.payment.paid`、`billing.payment.refunded`、`membership.changed`、`points.confirmed` 和 `points.released`。平台发送 `X-IdP-Event-Id`、`X-IdP-Timestamp` 与 `X-IdP-Signature: v1=<hex>`。

应用必须对原始请求体按以下形式使用 HMAC-SHA256 验签：

```text
HMAC-SHA256(webhook_secret, "<timestamp>.<raw request body>")
```

先校验时间窗口和签名，再解析 JSON；按 `event_id` 做持久化唯一去重。2xx 表示成功，失败或超时最多重试 8 次；最终支付或会员状态仍以查询接口为准。Webhook 负载不包含内部用户 ID、手机号、邮箱、Token、密码、Secret 或完整钱包流水。

## 应用托管、部署和数据库

应用托管目前是开发者中心“应用托管”页面中的平台流程，不是可以自行猜测的 API 流程：

1. 提交开通申请，选择套餐、模板（`static_site` / `nodejs` / `python`）、应用标识和用途说明；
2. 等待 Admin 审核，通过后进入待部署状态并按套餐月度扣积分；
3. 将项目打成 `.zip` 上传，平台自动构建镜像、启动容器和健康检查；
4. 后续重新上传 `.zip` 可重新部署，构建或健康检查失败会自动回滚到上一个可用版本；也可以手动唤醒、下线或销毁实例。

出站网络默认全部禁止，只自动放行应用自己的公开子域名。调用第三方 API 前必须联系管理员放行目标地址。

### 数据库连接

每个托管实例有独立 MySQL 数据库和专属账号：

- `nodejs` 和 `python` 应用在容器内读取平台自动注入的 `DATABASE_URL`（标准 `mysql://` 连接串）；
- `static_site` 不使用数据库；
- 数据库不对公网开放，本机不能用 MySQL 客户端直连；
- 本机运行的 AI 工具通过以下接口管理或查询自己名下的托管数据库：

```http
POST /v1/developer/hosting-databases/{instanceId}/sql-query
X-API-Key: <DEVELOPER_API_KEY>
Content-Type: application/json

{ "sql": "SELECT * FROM your_table LIMIT 20" }
```

单次 SQL 查询超过 10 秒会中止，结果最多 200 行。`SELECT`/`SHOW` 返回 `columns`/`rows`，写入或 DDL 语句返回 `affected_rows`。数据库连接接口使用账号级 Developer API Key，Key 仍必须服务端保存。

### 托管打包和运行要求

- 只收不超过 20 MB 的 `.zip`；压缩包最外层必须直接有入口文件，不能再套一层项目目录；
- 不打包 `node_modules`、`.git`、`.venv`、`__pycache__` 和 `.env`；密钥放实例详情页环境变量，最多 30 个、单值不超过 4 KB，修改后必须重新部署；
- 平台自动注入 `PORT=8080` 和 `DATABASE_URL`，不能自定义 `PORT`；
- `nodejs` 使用 `node:20-alpine`、`npm ci --ignore-scripts`、`npm run build --if-present`，必须有根目录 `package.json`、一致的 `package-lock.json` 和 `scripts.start`；
- `python` 使用 `python:3.12-slim`、`pip install --no-deps -r requirements.txt`，必须有完整版本依赖和根目录 `Procfile`，其中包含监听 `0.0.0.0:$PORT` 的 `web:` 命令；
- `static_site` 不执行构建，上传的必须是构建产物，`index.html` 必须在 zip 最外层；
- 容器文件系统只读，只有 `/tmp` 可写且会随重启清空；业务数据放 MySQL，不写 SQLite、日志或上传文件到本地磁盘；
- 启动后 60 秒内必须在 `0.0.0.0:8080` 让 `GET /` 返回 2xx；应用先监听，再做迁移、seed、缓存预热和外部连接；
- 支持 `SIGTERM` 并在 10 秒内退出；长连接、定时任务或后台队列需要常驻套餐；
- 空闲挂起后，页面导航会由平台唤醒；前端 API 可能收到带 `Retry-After` 和 `X-Idp-Wake-Page: 1` 的 503，只有幂等请求可以等待后重试一次。

### 托管环境变量导入

应用托管详情页可以从应用中心导入接入配置。推荐变量名：`AIGZM_CLIENT_ID`、`AIGZM_CLIENT_SECRET`（密钥）、`AIGZM_REDIRECT_URI`、`AIGZM_BILLING_RETURN_URI`、`AIGZM_WEBHOOK_URL`、`AIGZM_WEBHOOK_SECRET`（密钥）、`AIGZM_API_KEY`（密钥）和 `AIGZM_EXPECTED_ISSUER`。

导入 Client Secret、Webhook Secret 或 API Key 会生成新值并可能使旧值失效；必须准备部署切换流程，并在本地或其他服务器同步更新。环境变量改完后必须重新部署才会生效。

## 上线验收

至少完成：

1. OpenAPI 请求和响应契约校验；
2. Sandbox OAuth 成功、拒绝、回调错误和 Token 兑换；
3. 模型调用成功、余额不足、超时、错误模型入口和幂等重试；
4. 数据 API 成功、上游失败、响应头记录和错误退费；
5. 会员等级、会员详情、钱包余额和应用购买权益读取，并确认响应不缓存；
6. 充值、会员购买、一次性应用购买、退款后的重新授权和支付回跳核验；
7. Webhook 原始正文验签、时间窗口、去重和重复投递；
8. 托管 zip 大小、根目录入口、模板构建、`PORT`、健康检查、只读文件系统和环境变量检查；
9. 容器内通过 `DATABASE_URL` 连接 MySQL，本机通过 `sql-query` 的 SELECT/写入/超时边界；
10. 空闲唤醒 503 的幂等重试和常驻/挂起套餐选择；
11. 敏感信息扫描，确认没有进入前端、URL、日志、源码或测试产物；
12. 正式环境端到端联调前，完成 Sandbox 与生产配置、数据库、Webhook、凭据和日志隔离检查。

