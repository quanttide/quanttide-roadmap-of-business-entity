# 量潮课堂接入账号与支付系统设计

> 把统一身份云 qtcloud-auth 与支付云 qtcloud-pay 接入课堂云 qtclass，形成「身份 → 课程 → 学习 → 付费 → 发券」的闭环。本文接口、路径、字段均来自三个仓库的真实代码（见文末参考文件），可直接作为开发清单。

## 目标与范围

| 系统 | 仓库 | 在闭环中的角色 |
|:-----|:-----|:---------------|
| 课堂云 | `quanttide/qtclass` | 聚合编排层，客户端唯一依赖的服务端 |
| 身份云 | `quanttide/qtcloud-auth` | 统一登录、JWT 签发、用户与角色 |
| 支付云 | `quanttide/qtcloud-pay` | 账户账本、下单结算、支付渠道、发券 |

核心原则是 **ID 同源直连**：三系统共享同一个身份键。

```text
auth.userinfo.sub == learn.learner.user_id == pay.account.customer_id
```

不需要映射表或同步作业，学习云与支付云面对同一套稳定的 `sub` 身份空间，思路与 qtclass 现有的 `Completion.criterion_id ↔ 课程云 Criterion.id` 同源直连一致。

---

## 现状盘点

### provider 服务端

- 已聚合课程云（`QTCLASS_API_BASE_URL/qtcloud-course`）与学习云（`/qtcloud-learn`），模式见 `internal/upstream/{upstream,learn}.go` 的 `CourseClient` 与 `LearnClient`
- 提供 `GET /courses`、`GET /courses/{id}/player`、`GET /player-data`，以及学习云代理（`/schedules`、`/tasks`、`/completions`、`/learners`）
- API 文档要求除健康检查与只读契约外均需 `Authorization: Bearer <token>`（qtcloud-auth 签发），但 provider 当前只把 token 透传给学习云，自身不校验
- 没有对接支付云的任何代码

### 客户端

- `AuthApi` 已直连 `https://api.quanttide.com/qtcloud-auth`，`POST /oauth/token`（password 模式），token 存 `SharedPreferences('qtclass_token')`，可从 JWT 解析 `sub`
- `LearnApi` 调 `https://api.quanttide.com/qtclass` 并带 Bearer；`ensureLearnerForCurrentUser()` 用 `sub` 找建 learner（`user_id = sub`），账号到学习的映射已通
- `course_data.dart` 的播放器数据已从 `assets/course.json` 改为可走 API

### 结论

登录链路已基本打通，缺口有二：

- provider 服务端未校验 auth 的 JWT，需新增校验并提取 `sub` 作为 learner 与 customer 身份
- 完全未接入支付云，无 PayClient、无钱包与订单路由、无客户端 PayApi

## 目标架构

```text
Studio / Site / Lite 客户端
  │ ① 登录 POST /oauth/token（password 模式）
  │      └─▶ qtcloud-auth（身份底座，RS256）签发 access_token
  │ ② 业务请求 Authorization: Bearer <token>，sub 为身份键
  ▼
qtclass provider（校验 JWT，sub 注入 context）
  ├─ 聚合代理 ──▶ qtcloud-course、qtcloud-learn
  └─ 钱包 / 订单 / 发券 ──▶ qtcloud-pay（账本、支付、发券）
```

- 客户端只依赖 qtclass provider，auth 与 pay 由 provider 代理聚合，与现有学习云代理一致
- 身份键 `sub` 在 provider 校验后注入 context，向下游 learn 与 pay 透传，或作为 `customer_id`

## 账号系统接入

### 服务端 JWT 校验

provider 目前没有校验逻辑，需新增 `AuthMiddleware`：

- auth 已升级 RS256 非对称签名，私钥 `JWT_PRIVATE_KEY` 在 auth 侧，公钥需在 qtclass 侧配置
- 配置项建议：`QTCLASS_AUTH_JWKS_URL`（auth 公钥端点）或 `QTCLASS_AUTH_PUBLIC_KEY`（base64(PEM) 单行，与 auth 一致以避免多行 env 问题）
- 校验通过后解析 claims，把 `sub` 注入 `request.Context`，供 handler 作为 `learner_id` 与 `customer_id`
- 行为对齐现有约定：缺失或无效、过期的 token 返回 401；只读契约（`/courses`、`/player-data`）放行

### 客户端

- `AuthApi` 已正确连接 qtcloud-auth，Site 与 Lite 复用同一登录入口即可，不必另写
- 利用 auth 已支持的 `refresh_token`（grant_type=refresh_token）做静默续期，避免学习或支付中途掉登录
- token 存储沿用 `SharedPreferences`，多端（studio、site、lite）统一 key

### 身份映射

| 下游 | 字段 | 取值 |
|:-----|:-----|:-----|
| 学习云 learner | `user_id` | `auth.sub`（`LearnApi.ensureLearnerForCurrentUser` 已实现） |
| 支付云 account | `customer_id` | `auth.sub`（新建账户时传入） |

## 支付系统接入

### 业务闭环

> 学生付费不是买断课程，而是记入账户额度；费用随学习进度逐次扣除；运营按交付发放折扣券与代金券。

对应 pay 内部链路：付费 → 充值登记（recharge）或下单结算（order）→ balance → 按学习写 consume 交易 → 完成发 coupon 与 voucher。

### 新增 PayClient

仿 `CourseClient` 与 `LearnClient` 的 upstream 模式，新增 `internal/upstream/pay.go`：

```go
type PayClient struct {
    BaseURL string // QTCLASS_API_BASE_URL 派生: <base>/qtcloud-pay
    HTTP    *http.Client
}

func NewPayClient(baseURL string) *PayClient {
    return &PayClient{BaseURL: trimSlash(baseURL), HTTP: &http.Client{}}
}
// 透传 Authorization，封装 GET/POST 到对应路径
```

`config.go` 增加：

```go
func (c *Config) PayBase() string {
    return strings.TrimRight(c.APIBaseURL, "/") + "/qtcloud-pay"
}
// 或独立变量 QTPAY_API_BASE_URL，避免与课程、学习云耦合
```

### 路由与代理

沿用 `handler/proxy.go` 的「客户端路径 → 上游路径」代理模式，新增支付路由：

| qtclass 路径 | 上游 pay 路径 | 说明 |
|:-------------|:-------------|:-----|
| `GET /wallet` | `GET /accounts/{id}` | 账户与余额，id 由 `sub` 解析 |
| `GET /wallet/transactions` | `GET /accounts/{id}/transactions` | 流水 |
| `GET /wallet/coupons` | `GET /accounts/{id}/coupons` | 我的券 |
| `GET /wallet/vouchers` | `GET /accounts/{id}/vouchers` | 我的代金券 |
| `POST /orders` | `POST /orders` | 报名下单并结算（ProductID 为课程或课时） |
| `GET /orders/{id}` | `GET /orders/{id}` | 订单明细 |
| `POST /pay` | `POST /pay` | 发起支付，返回微信 JSAPI 或支付宝调起参数 |
| `GET /pay/query/{id}` | `GET /query/{order_id}` | 支付状态查询 |

三条接入要点：

- 钱包类路由需身份：provider 校验 JWT 取 `sub`，查建 `account(customer_id=sub)` 后代理
- 按学习扣费：播放器每步完成回写（现有 `POST /completions`）后，provider 调 pay 写一条 consume 交易，按课时金额分账，对齐 pay 测试中「两节课各扣 10000」的模式
- 按交付发券：学员完成课程或任务时，provider 或 Studio 运营端调 `POST /accounts/{id}/coupons` 或 `/vouchers` 发放

### 概念映射

| 课堂概念 | 支付云实体 | 备注 |
|:---------|:-----------|:-----|
| 课程或课时 | `Product`（订单 `ProductID`） | 付费对象 |
| 学员 | `Account`（`CustomerID = sub`） | 虚拟钱包 |
| 报名付费 | `recharge` 或 `order` 结算 | 记额度 |
| 学习进度 | `consume` 交易 | 逐次扣费，余额投影 |
| 交付完成 | `coupon` 或 `voucher` 发放 | 按交付发券 |
| 余额不足 | HTTP 422 | 拒绝结算，订单不落库 |

### 客户端钱包与支付入口

qtcloud-pay 的 lite 已验证做法：登录复用 qtcloud-auth，`customerId` 取 `userinfo.sub`。qtclass 三端新增 `PayApi`：

```dart
// PayApi 复用 AuthApi.token 作为 Bearer
class PayApi {
  final String baseUrl; // QTCLASS_API_BASE_URL（走 qtclass provider 代理）
  Future<Map> wallet() => _get('/wallet');
  Future<Map> createOrder({required String productId, required int amount}) =>
      _post('/orders', {'product_id': productId, 'amount': amount});
  Future<Map> pay(String orderId) => _post('/pay', {'order_id': orderId});
}
```

微信 JSAPI 与支付宝网页支付由 `/pay` 返回前端调起参数，客户端 SDK 拉起；钱包页（余额、流水、券）直接读 `/wallet*` 代理。

---

## 配置与部署

### 环境变量

- qtclass provider 新增：`QTPAY_API_BASE_URL`（支付云基址，建议独立变量）；`QTCLASS_AUTH_JWKS_URL` 或 `QTCLASS_AUTH_PUBLIC_KEY`（auth 公钥，用于 RS256 验签）
- qtclass provider 现有：`QTCLASS_API_BASE_URL`、`LISTEN_ADDR`
- qtcloud-auth 现有：`JWT_PRIVATE_KEY`、`ADMIN_TOKEN`、`DB_*`、`SMS_DRIVER`
- qtcloud-pay 现有：`SECRET_KEY`、`DB_*`、渠道凭据（wechat/alipay），Bearer 或 `X-Admin-Token` 鉴权

### 网关路由

统一入口 `api.quanttide.com` 现有 `/qtclass`、`/qtcloud-auth`、`/qtcloud-course`、`/qtcloud-learn`；新增 `/qtcloud-pay`，与 provider 内部 pay 基址保持一致。

### 基础设施

- 复用 `qtclass/manifests/terraform`（FC 或容器 + OSS 或 CDN + 网关），为 qtcloud-pay 增加 Provider、函数与路由资源，参考 pay 的 `Makefile` 与 Docker 部署
- 密钥分发：auth 公钥下发给 qtclass，仅验签不可签发；pay 的 `SECRET_KEY` 用于 provider 调 pay 时携带 Bearer，pay 要求 401 与 403 fail-closed

## 落地步骤

1. Phase 0 准备：网关新增 `/qtcloud-pay` 路由；qtclass 注入 `QTPAY_API_BASE_URL` 与 auth 公钥；pay 起独立实例并打通 DB
2. Phase 1 账号服务端校验：provider 加 AuthMiddleware（RS256 验签 + `sub` 注入 context），先对非只读接口启用
3. Phase 2 支付只读接入：新增 PayClient 与钱包查询、流水、券代理路由，客户端 PayApi 钱包页，不影响交易
4. Phase 3 支付交易接入：`/orders`、`/pay`、按学习扣费、按交付发券，对接微信与支付宝渠道并验证回调入账
5. Phase 4 客户端支付入口：Site 价目与报名、Lite 任务付费、Studio 运营发券界面
6. Phase 5 联调与对账：跑 pay 的课堂场景端到端（付费、记额度、扣费、发券），用 `/reconcile/consistency` 与 `/accounts/{id}/statement` 校验

## 风险与注意点

- JWT 算法：auth 已为 RS256，qtclass 校验必须用公钥，不能用共享 secret（README 旧描述写 HS256，以 `main.go` 实际为准）
- 幂等：pay 所有写接口都有幂等键（充值 `recharge:{voucher_no}`、退款、发券批次号、订单号），provider 调用需带稳定幂等键，避免重试重复入账
- 金额单位：pay 账本内部用分，API 用元（两位小数），provider 聚合展示时统一换算，避免百倍误差
- 余额不足：pay 返回 422 且订单不落库，客户端需给出余额不足与充值引导
- 跨云一致性：completion 触发发券属跨服务写，建议完成记录落库后异步或补偿发券，不把发券放进学习云事务
- 安全：auth 与 pay 运维端点均 fail-closed（缺 token 或不匹配一律 403）；provider 代理只放行白名单路径，禁止把 `X-Admin-Token` 与管理端点透传给客户端

## 参考文件

- qtclass 服务端：`src/provider/README.md`、`docs/api-references/provider.md`、`docs/dev-guide/provider.md`、`cmd/server/main.go`、`internal/config/config.go`、`internal/upstream/{upstream,learn}.go`、`internal/handler/proxy.go`
- qtclass 客户端：`studio/lib/services/auth_api.dart`、`learn_api.dart`、`course_data.dart`
- qtcloud-auth：`src/provider/README.md`、`src/provider/cmd/server/main.go`
- qtcloud-pay：`src/provider/README.md`、`docs/index.md`、`account.md`、`order.md`、`billing.md`
