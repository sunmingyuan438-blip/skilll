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

## 支付、会员和 Webhook

如果产品需要充值或会员：

- `billing_url` 是不透明的浏览器地址，只能跳转，不能解析、拼接参数或调用 `/mock-result`；
- 回跳或入口 `status=consumed` 不等于支付成功；必须用同一用户 Token 查询到 `payment_order.status=paid`，再读取钱包或会员；
- Webhook 使用当前环境 Secret 对原始请求体验签，校验 `environment`、`client_id`，按 `event_id` 去重；最终状态仍以查询接口为准；
- 积分账户与返佣人民币账户分离，不实现自动提现或分账。

## 上线验收

至少完成：

1. OpenAPI 请求和响应契约校验；
2. Sandbox OAuth 成功、拒绝、回调错误和 Token 兑换；
3. 模型调用成功、余额不足、超时、错误模型入口和幂等重试；
4. 数据 API 成功、上游失败、响应头记录和错误退费；
5. Webhook 验签、去重和重复投递；
6. 支付/会员状态以查询结果确认；
7. 敏感信息扫描，确认没有进入前端、URL、日志、源码或测试产物；
8. 正式环境端到端联调前，完成 Sandbox 与生产配置隔离检查。

