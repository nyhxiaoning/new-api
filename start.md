# New API 项目分析

## 本地开发部署文档
[本地开发部署文档：](https://docs.newapi.pro/zh/docs/installation/deployment-methods/local-development)

### 前端部署说明：
第三部分：

- 新版前端 (default):
- 旧版前端（class）


### 后段部署说明：

开发环境默认使用SQLite，
```
PORT=3000
SQL_DSN=root:password@tcp(localhost:3306)/new-api   # 如使用MySQL，取消注释并修改
# REDIS_CONN_STRING=redis://localhost:6379         # 如使用Redis，取消注释并修改
```


#### 后端实时调试：
go run main.go --log-dir ./logs



## 核心问题：如何统一管理API网关服务，假设我想要加入kimi的厂商，为我们的一个小组提供5个账号服务访问，请给出NewAPI平台使用方式，这里的kimiapi地址：https://api.moonshot.cn/v1

- 三步：创建渠道，创建用户，然后通过openAI客户端调用；
- 加入一个kimi的请求的提供商和地址。
访问api地址：https://api.moonshot.cn/v1

这里如何统一管理转发配置的API的厂家的网关？

Kimi (Moonshot) 不需要任何开发，NewAPI 已有完整适配器。全流程就是纯配置操作：

1. 管理面板 → 渠道管理 → 新增5条 Moonshot 渠道，分组都设为 kimi-team
注意：这里的分组创建：console/setting?tab=ratio，创建支付设置位置，这里选择：分组管理创建。

2. 管理面板 → 用户管理 → 新建用户，分组设为 kimi-team，设置配额
3. 用户拿到 Token → 用 OpenAI 兼容客户端连接 http://your-newapi:3000/v1
4. 新建套餐：套餐这里需要配置订阅讨论的确定输入隐私。
注意：创建套餐后，可以购买创建
5. 通过新用户订阅的内容，比如：admin2，密码nyh123456，进入后。
```
curl http://localhost:3000/v1/chat/completions \
  -H "Authorization: Bearer sk-xxxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "kimi-k2.5",
    "messages": [{"role": "user", "content": "你好"}]
  }'




```

NewAPI 自动完成负载均衡（5个账号轮询）、故障切换、用量统计和计费。你无需关心底层哪个 Key 被用到。

### 订阅套餐与Token的关系：用户如何通过Token使用订阅

核心链路：**渠道绑定分组 → 套餐升级用户分组 → Token继承分组 → 匹配渠道**

```
                   渠道1(kimi-team) ──┐
                   渠道2(kimi-team) ──┤
                   渠道3(kimi-team) ──┤  ← 渠道绑定到分组
                   渠道4(kimi-team) ──┤
                   渠道5(kimi-team) ──┘
                                        │
                 订阅套餐(UpgradeGroup=kimi-team)
                                        │
                 用户购买订阅 → 用户分组自动变为 kimi-team
                                        │
                 用户创建Token → Token继承用户分组 kimi-team
                                        │
                 用户用Token调API → 分发器匹配kimi-team渠道 → 负载均衡
```

#### 具体操作步骤

**第一步：创建套餐时设升级分组**

在管理面板 → 套餐管理 → 新增套餐，关键字段：

| 字段 | 值 | 说明 |
|---|---|---|
| 升级分组 (UpgradeGroup) | `kimi-team` | 用户购买后自动升级到此分组 |
| 降级分组 (DowngradeGroup) | `default` | 订阅过期后用户回退到默认分组 |
| 总配额 (TotalAmount) | `1000000` | 订阅包含的额度（可周期性重置） |
| 配额重置周期 | `monthly` | 每月重置额度 |
| 允许余额兜底 | true | 额度用完后可从钱包余额扣 |

**第二步：用户购买套餐**

- 方式A：用户自行在面板点击"购买套餐" → 余额扣费或支付 → 订阅激活
- 方式B：管理员在"用户管理 → 操作为用户订阅套餐" → 无支付直接绑定

**第三步：用户创建Token并开始使用**

用户购买后，系统会自动：
1. 将用户的 `group` 从默认分组升级到 `kimi-team`
2. 用户已有的Token（如果有）仍然可用—Token的 `group` 为空时自动继承用户分组
3. 也可以新建一个Token，分组字段留空即可

用户拿到Token后使用：
```bash
curl http://localhost:3000/v1/chat/completions \
  -H "Authorization: Bearer <用户的Token>" \
  -H "Content-Type: application/json" \
  -d '{"model": "kimi-k2.5", "messages": [{"role":"user","content":"hi"}]}'
```

分发器会：
1. 读取Token → 找到用户 → 找到用户分组 `kimi-team`
2. 在所有 `kimi-team` 分组的渠道中选可用的一条
3. 如果配额用完 → 检查 `AllowWalletOverflow`，允许则从钱包余额扣

**第四步：订阅过期时的自动降级**

套餐过期后，系统自动：
1. 标记订阅状态为 `expired`
2. 用户分组回退到 `DowngradeGroup`（如 `default`）
3. 用户的Token仍然可用，但只能匹配 `default` 分组的渠道
4. 续费后分组自动升回 `kimi-team`


### （1）关于其中使用：订阅套餐创建和变更已锁定，管理员需先在支付设置中确认合规声明。
操作路径：

1. 登录管理面板 → 系统设置 → 计费设置（Billing） → 支付网关（Payment Gateway）
2. 在支付网关配置区域，会有一个 合规声明确认 的对话框/按钮
3. 阅读合规提醒并勾选确认
4. 确认后，所有被锁的订阅操作（创建套餐、修改套餐、变更订阅等）会自动解锁

### （2）管理员未开启在线支付功能，请联系管理员配置。
所以你的完整配置路径应该是：
增加一个配置：DEV_ENABLE_PAYMENT=true，true跳过检查

1. 管理面板 → 系统设置 → Billing → Payment Gateway
2. 先点 合规声明确认（确认弹窗）
3. 然后在同一页面配置至少一个真实的支付网关（如 Stripe），填入 API Secret、Webhook Secret、Price ID 等
4. 保存后刷新，订阅管理和充值功能才会解锁

关键逻辑在 controller/payment_webhook_availability.go：每种支付方式同时需要 compliance_confirmed == true + 凭证不为空。只确认合规但没填任何支付凭证，计费和订阅功能仍然不可用。

### （3）Dev 模式下打通完整流程

设置 `DEV_ENABLE_PAYMENT=true` 后，大部分开关已经跳过。但订阅购买流程中还有两个硬编码检查会导致卡住：

**错误1："套餐金额过低"**
创建套餐时 `PriceAmount` 需要 ≥ 0.01。在管理面板创建套餐时把价格设为 ≥ 0.01 即可（如 1 USD）。如果设为 0 会被这个检查拦住。

**错误2："当前管理员未配置支付信息"**
`GetEpayClient()` 检查 `EpayId/EpayKey/PayAddress` 三个配空就返回 nil。这个不走 `DevEnablePayment` 开关。

**Dev 模式已修复的行为**（当前代码已改好）：
- `GetEpayClient()` → 在 dev 模式下用虚拟参数也能创建 client，不再卡住
- `SubscriptionRequestEpay()` → 在 dev 模式下不发起真实支付，而是直接调用 `AdminBindSubscription` 绑定套餐

**本地测试完整流程就三步：**
```bash
# 1. 启动（打开 dev 支付模式）
DEV_ENABLE_PAYMENT=true go run main.go --log-dir ./logs

# 2. 管理面板操作
#    - 系统设置 → Billing → Payment Gateway → 合规声明确认
#    - 套餐管理 → 新增套餐（金额设 ≥ 0.01，如 1 USD）
#    - 套餐管理 → 点击购买 → 用余额支付或走 Epay

# 3. 验证：用户该套餐的状态变为"已订阅"
```

> 注意：前端"购买套餐"按钮走的是余额支付（余额充足时直接扣减）
> 或 Epay 支付（触发 `SubscriptionRequestEpay`，dev 模式自动模拟绑定）。
> 走"管理 → 为用户订阅套餐"则无需支付，直接绑定。


## 支付网关配置指南

所有支付网关统一在 **管理面板 → 系统设置 → Billing → Payment Gateway** 配置。配置前必须先确认合规声明。

### 通用前提：合规声明

```
管理面板 → 系统设置 → Billing → Payment Gateway
→ 在页面底部找到"合规声明"区域
→ 阅读并勾选确认
→ 保存
```

只有确认后，所有支付方式才会生效（`controller/payment_webhook_availability.go` 中每个 `is*Enabled()` 都先检查 `compliance_confirmed`）。

> **Dev 模式**：`DEV_ENABLE_PAYMENT=true go run main.go` 可跳过合规检查和凭证检查。

---

### 1. 易支付 (Epay)

**适用场景**：国内用户，支持支付宝、微信等本地支付方式。

| 配置字段 | 说明 | 获取方式 |
|---------|------|---------|
| 支付地址 (PayAddress) | 易支付站点地址，如 `https://epay.example.com` | 易支付商户后台 |
| 商户ID (EpayId) | 商户号 | 易支付商户后台 |
| 商户密钥 (EpayKey) | API 密钥 | 易支付商户后台 |
| 回调地址 (CustomCallbackAddress) | 支付成功后的回调地址，如 `https://your-api.com` | 你的 NewAPI 部署地址 |
| 支付方式 (PayMethods) | 可用支付通道（支付宝/微信/QQ钱包等） | 自行配置 JSON |
| 最低充值 (MinTopUp) | 单笔最低充值金额 | 自行设置的阈值 |
| 汇率 (USDExchangeRate) | 1 USD = ? CNY | 默认 7.3 |

**后端逻辑**（`controller/topup.go:136`）：
```go
func GetEpayClient() *epay.Client {
    // 检查 PayAddress / EpayId / EpayKey 是否为空
    // 用 epay.NewClient 创建支付客户端
}
```

**配置示例**：
```
PayAddress:        https://pay.your-epay.com
EpayId:            10001
EpayKey:           abcdef1234567890
CustomCallbackAddress: https://newapi.example.com
PayMethods:        [{"name":"支付宝","icon":"SiAlipay","type":"alipay"},
                   {"name":"微信支付","icon":"SiWechat","type":"wxpay"}]
MinTopUp:          1
```

> 注意：所有 Epay 回调接口前缀必须与 `CustomCallbackAddress` 一致。应用内回调路由：
> - `/api/user/epay/notify` — 充值异步通知
> - `/api/user/epay/return` — 充值同步跳回
> - `/api/subscription/epay/notify` — 订阅异步通知
> - `/api/subscription/epay/return` — 订阅同步跳回

---

### 2. Stripe

**适用场景**：海外用户，支持国际信用卡支付。

| 配置字段 | 说明 | 获取方式 |
|---------|------|---------|
| API Secret (StripeApiSecret) | `sk_live_xxx` 或 `sk_test_xxx` | Stripe Dashboard → Developers → API keys |
| Webhook Secret (StripeWebhookSecret) | `whsec_xxx` | Stripe Dashboard → Developers → Webhooks → 添加 endpoint |
| Price ID (StripePriceId) | `price_xxx` | Stripe Dashboard → Products → 创建产品 → 获取 Price ID |
| 单价 (StripeUnitPrice) | 每单位配额对应的 USD 金额 | 自行设置 |
| 最低充值 (StripeMinTopUp) | 单笔最低充值金额（USD） | 默认 1 |
| 折扣码 (StripePromotionCodesEnabled) | 是否启用折扣码 | true / false |

**后端逻辑**（`controller/payment_webhook_availability.go:14`）：
```go
func isStripeTopUpEnabled() bool {
    return StripeApiSecret != "" && StripeWebhookSecret != "" && StripePriceId != ""
}
```

**Webhook 配置步骤**：
1. 在 Stripe Dashboard 创建 Webhook endpoint：`https://your-newapi.com/api/stripe/webhook`
2. 监听事件：`checkout.session.completed`、`invoice.paid`
3. Stripe 返回 `whsec_xxx` 签名密钥，填入 Webhook Secret 字段

**配置示例**：
```
StripeApiSecret:          sk_live_xxxxxxxxxxxxxxxx
StripeWebhookSecret:      whsec_xxxxxxxxxxxxxxxx
StripePriceId:            price_xxxxxxxxxxxxx
StripeUnitPrice:          8.0
StripeMinTopUp:           1
StripePromotionCodesEnabled: false
```

---

### 3. Creem

**适用场景**：面向开发者的支付平台，支持一键购买 API 产品。

| 配置字段 | 说明 | 获取方式 |
|---------|------|---------|
| API Key (CreemApiKey) | Creem 平台 API 密钥 | Creem Dashboard |
| Webhook Secret (CreemWebhookSecret) | Webhook 签名密钥 | Creem Dashboard → Webhooks |
| 产品列表 (CreemProducts) | JSON 数组，定义可购买的产品 | 自行配置 |
| 测试模式 (CreemTestMode) | 沙箱环境开关 | true / false |

**后端逻辑**（`controller/payment_webhook_availability.go:31`）：
```go
func isCreemTopUpEnabled() bool {
    return CreemApiKey != "" && CreemProducts != "" && CreemProducts != "[]"
}
```

**Webhook 配置步骤**：在 Creem Dashboard 添加 Webhook: `https://your-newapi.com/api/creem/webhook`

**配置示例**：
```
CreemApiKey:           creem_sk_xxxxxxxxxxxx
CreemWebhookSecret:    whsec_xxxxxxxxxxxx
CreemProducts:         [{"name":"基础包","price":5,"credits":1000},
                        {"name":"高级包","price":20,"credits":5000}]
CreemTestMode:         false
```

---

### 4. Waffo

**适用场景**：企业级独立结算平台，功能包括商户管理、门店、产品编码。

分为两种模式：**标准 Waffo** 和 **Waffo Pancake**。

#### 4a. 标准 Waffo

| 配置字段 | 说明 |
|---------|------|
| WaffoEnabled | 启用开关 |
| API Key (WaffoApiKey) | API 密钥 |
| Private Key (WaffoPrivateKey) | 私钥 |
| Public Cert (WaffoPublicCert) | 公钥证书 |
| 沙箱模式 (WaffoSandbox) | 沙箱启用后使用沙箱密钥 |
| 沙箱 API Key (WaffoSandboxApiKey) | 沙箱环境密钥 |
| 沙箱 Private Key (WaffoSandboxPrivateKey) | 沙箱环境私钥 |
| 沙箱 Public Cert (WaffoSandboxPublicCert) | 沙箱环境公钥 |
| 商户ID (WaffoMerchantId) | 商户唯一标识 |
| 货币 (WaffoCurrency) | 结算货币，默认 USD |
| 单价 (WaffoUnitPrice) | 1.0 |
| 最低充值 (WaffoMinTopUp) | 默认 1 |
| 支付方式 (WaffoPayMethods) | JSON 配置可用支付方式 |
| 回调地址 (WaffoNotifyUrl) | 异步通知地址 |
| 返回地址 (WaffoReturnUrl) | 同步跳回地址 |
| 订阅返回地址 (WaffoSubscriptionReturnUrl) | 订阅成功跳回 |

**后端逻辑**（`controller/payment_webhook_availability.go:49`）：
```go
func isWaffoTopUpEnabled() bool {
    return complianceConfirmed && WaffoEnabled && webhookConfigured
}
```

#### 4b. Waffo Pancake

Waffo Pancake 是简化版 Waffo，支持产品编码直接结算。

| 配置字段 | 说明 | 获取方式 |
|---------|------|---------|
| 商户ID (WaffoPancakeMerchantID) | 商户标识 | Waffo Pancake 后台 |
| 私钥 (WaffoPancakePrivateKey) | API 签名私钥 | Waffo Pancake 后台 |
| 返回地址 (WaffoPancakeReturnURL) | 回调地址 | 你的 NewAPI 地址 |
| 单价 (WaffoPancakeUnitPrice) | 1.0 | 自行设置 |
| 最低充值 (WaffoPancakeMinTopUp) | 默认 1 | 自行设置 |
| 门店ID (WaffoPancakeStoreID) | 自动分配，只读 | 配置成功后自动生成 |
| 产品ID (WaffoPancakeProductID) | 产品编码 | 配置成功后自动生成 |

**后端逻辑**（`controller/payment_webhook_availability.go:76`）：
```go
func isWaffoPancakeTopUpEnabled() bool {
    return complianceConfirmed && MerchantID != "" && PrivateKey != "" && ProductID != ""
}
```

---

### 支付方式总览比较

| 特性 | 易支付 (Epay) | Stripe | Creem | Waffo | Waffo Pancake |
|-----|:---:|:---:|:---:|:---:|:---:|
| 国内支付（支付宝/微信） | ✓ | ✗ | ✗ | ✓ | ✓ |
| 国际信用卡 | ✗ | ✓ | ✓ | ✓ | ✓ |
| 订阅账单 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 一键充值 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 沙箱测试 | ✗ | ✓ | ✓ | ✓ | ✓ |
| Webhook 配置 | 内建路由 | 需额外配置 | 需额外配置 | 需额外配置 | 自动签发 |
| 配置复杂度 | 低 | 中 | 低 | 高 | 低 |

### 花一分钟快速验证支付配置

以 Epay 为例的完整端到端测试：
```
1. 确认合规声明
2. 填入 EpayId / EpayKey / PayAddress
3. 设置 CustomCallbackAddress（通常就是 NewAPI 的服务器地址）
4. 保存
5. 用户登录 → 钱包 → 点击充值 → 应能看到支付宝/微信支付按钮
6. 点击下单 → 跳转到易支付付款页 → 完成支付 → 自动回调 → 余额到账
```


## 1. 项目定位

**Next-Generation LLM Gateway and AI Asset Management System**（新一代大模型网关与 AI 资产管理系统）

这是一个开源的 AI API 网关/代理项目，将 40+ 上游 AI 提供商（OpenAI、Claude、Gemini、Azure、AWS Bedrock、DeepSeek 等）统一聚合在单一 API 背后。功能范围包括：

- 统一 API 代理转发（兼容 OpenAI 格式输出）
- 用户管理与组织级鉴权
- 用量计费与成本核算（预扣费、结算、退款）
- 速率限制与渠道分发
- 管理面板（Web Dashboard）
- 多主题 UI（default + classic 两套前端）
- Electron 桌面客户端
- 国际化（后端 en/zh，前端 en/zh/fr/ru/ja/vi）

许可证：AGPL-3.0，提供商用授权

---

## 2. 技术栈

### 后端

| 类别     | 技术                                                             |
| -------- | ---------------------------------------------------------------- |
| 语言     | Go 1.25+                                                        |
| Web 框架 | Gin (gin-gonic/gin)                                              |
| ORM      | GORM v2                                                          |
| 数据库   | SQLite（glebarez/sqlite）、MySQL、PostgreSQL、ClickHouse（可选） |
| 缓存     | Redis (go-redis/v8) + 内存缓存 (samber/hot)                      |
| 鉴权     | JWT (golang-jwt)、WebAuthn/Passkeys (go-webauthn)                |
| OAuth    | GitHub、Discord、OIDC、LinuxDO、WeChat、Telegram                 |
| 支付     | Stripe、Epay、Creem、Waffo                                       |
| 测试     | testify (require/assert)                                         |
| 其他     | Casbin (权限)、shopspring/decimal、go-i18n、prometheus           |

### 前端

| 类别       | 技术                                                                                    |
| ---------- | --------------------------------------------------------------------------------------- |
| 包管理     | Bun (workspace)                                                                         |
| 框架       | React 19 + TypeScript                                                                   |
| 构建       | Rsbuild                                                                                 |
| 路由       | @tanstack/react-router                                                                  |
| 数据请求   | @tanstack/react-query、axios、Zustand (状态管理)                                        |
| UI 库      | Base UI (headless)、Tailwind CSS                                                         |
| 图标       | lucide-react、@hugeicons/core-free-icons                                                |
| 表单       | React Hook Form + Zod                                                                   |
| 表格       | @tanstack/react-table、@tanstack/react-virtual                                           |
| 图表       | @visactor/vchart                                                                        |
| 国际化     | i18next + react-i18next（7 种语言）                                                      |
| 代码质量   | oxlint、oxfmt、TypeScript (tsgo)、Prettier                                              |

---

## 3. 目录结构

```
├── main.go                 # 入口：初始化资源、启动 Gin 服务
├── router/                 # HTTP 路由定义
│   ├── api-router.go       #   REST API 路由 (/api/...)
│   ├── relay-router.go     #   AI 模型代理路由 (/v1/...)
│   ├── channel-router.go   #   渠道相关路由
│   ├── video-router.go     #   视频相关路由
│   ├── web-router.go       #   前端静态文件服务
│   └── dashboard.go        #   仪表盘路由
├── controller/             # 请求处理器（46 个文件，23259 行）
│   ├── channel.go          #   2170 行 — 最大文件
│   ├── user.go             #   1480 行
│   └── ...
├── service/                # 业务逻辑层
│   ├── channel*.go         #   渠道选择、亲和性缓存
│   ├── billing*.go         #   计费逻辑
│   └── ...
├── model/                  # 数据模型与数据库访问（GORM）
│   ├── user.go, channel.go, log.go, token.go, ...
│   └── main.go             #   DB 初始化、GORM 公共配置
├── relay/                  # AI 模型代理核心
│   ├── channel/            #   40+ 提供商适配器
│   │   ├── openai/         #   OpenAI 格式
│   │   ├── claude/         #   Anthropic Claude
│   │   ├── gemini/         #   Google Gemini
│   │   ├── aws/            #   AWS Bedrock
│   │   ├── vertex/         #   GCP Vertex AI
│   │   └── ...
│   ├── helper/             #   请求验证、适配
│   └── common/             #   中继公共逻辑
├── middleware/             # Gin 中间件（认证、限流、CORS、日志）
├── dto/                    # 数据传输对象（请求/响应结构体）
├── types/                  # 类型定义
├── constant/               # 常量定义
├── common/                 # 共享工具库
├── setting/                # 配置管理（模型比率、系统设置等）
├── i18n/                   # 后端国际化（en, zh）
├── oauth/                  # OAuth 提供商实现
├── pkg/                    # 包（billingexpr, cachex, perf_metrics）
├── logger/                 # 日志
├── web/                    # 前端（Bun workspace）
│   ├── default/            #   React 19 + Rsbuild + Tailwind（主前端）
│   │   └── src/
│   │       ├── routes/     #   TanStack Router 路由文件
│   │       ├── features/   #   功能模块（auth, channels, dashboard, ...）
│   │       ├── components/ #   通用组件
│   │       ├── lib/        #   工具函数
│   │       ├── hooks/      #   自定义 Hooks
│   │       ├── stores/     #   Zustand 状态管理
│   │       ├── context/    #   React Context 提供者
│   │       ├── i18n/       #   国际化文件（7 种语言）
│   │       └── styles/     #   全局样式
│   └── classic/            #   React 18 + Vite + Semi Design（旧版前端）
├── docs/                   # 文档
├── electron/               # Electron 桌面客户端
├── bin/                    # 构建脚本
└── data/                   # 运行时数据
```

**架构模式**：Router → Controller → Service → Model（分层架构）

---

## 4. 命令速查和启动说明

### 后端

| 命令                       | 说明                                     |
| -------------------------- | ---------------------------------------- |
| `go run main.go`           | 启动 API 服务                            |
| `go build -o new-api .`    | 编译                                     |
| `go test ./...`            | 运行所有测试（75 个测试文件）             |
| `make dev-api`             | Docker 启动开发数据库环境                 |
| `make start-api`           | 本地启动 API（go run）                   |
| `make dev-api-rebuild`     | 重建并启动 Docker API 服务               |

### 前端（重要：必须用 Bun，不是 npm）

**为什么 `cd web && npm run dev` 不行？**
- 项目用 **Bun** 包管理器，不是 npm。`web/` 下没有 `package-lock.json`，也没有 `dev` 脚本。
- `web/package.json` 是 **workspace 根配置**（只声明 workspaces 和版本目录），真正的脚本在 `web/default/package.json` 里。

**常见问题：Bun workspace 导致字体文件加载失败**

报错信息：
```
Module not found: Can't resolve '../../node_modules/@fontsource-variable/lora/files/lora-vietnamese-wght-normal.woff2'
```

原因：Bun workspace 把所有依赖提升（hoist）到 `web/node_modules/`，但 CSS（`src/styles/fonts.css`）中用 `../../node_modules/` 硬编码路径引用字体文件，Rsbuild 在 `web/default/` 下找不到。

修复方式：
```bash
mkdir -p web/default/node_modules
ln -s ../../node_modules/@fontsource-variable web/default/node_modules/@fontsource-variable
```

**正确启动方式：**

```bash
# 首次：安装所有前端依赖（在 web/ 目录执行，仅一次）
cd web && bun install

# 启动默认主题开发服务器（端口 5173）
cd web/default && bun run dev

# 一键启动（项目根目录执行 make）
make dev-web
```

**如果非要 `cd web` 后直接启动**，可以借助 Bun workspace：
```bash
cd web && bun run --filter default dev
```

所有命令必须在 `web/default/` 下执行（或通过 `--filter` 指定），因为只有 `web/default/package.json` 定义了 `dev`/`build`/`typecheck` 等脚本：

| 命令                                                       | 说明                    |
| ---------------------------------------------------------- | ----------------------- |
| `cd web/default && bun run dev`                            | 启动开发服务器（5173）   |
| `cd web/default && bun run build`                         | 生产构建                |
| `cd web/default && bun run typecheck`                     | TypeScript 类型检查     |
| `cd web/default && bun run lint`                          | 代码检查（oxlint）      |
| `cd web/default && bun run format`                        | 代码格式化              |
| `cd web/default && bun run i18n:sync`                     | 同步国际化文件          |
| `cd web/default && bun run knip`                          | 死代码检测              |
| 或通过 make：`make dev-web`                                | 自动安装依赖并启动 dev  |

### 全栈

| 命令                 | 说明                       |
| -------------------- | -------------------------- |
| `make all`           | 构建前后端并启动           |
| `make dev`           | 启动 Docker + 前端开发模式 |
| `make build-all-web` | 构建两套前端主题           |
| `make reset-setup`   | 重置初始化向导状态         |

---

## 5. 新增页面流程

### 后端 API（如 `/api/xxx`）

1. **DTO 定义**：在 `dto/` 下定义请求/响应结构体
2. **Controller**：在 `controller/` 下新增 `.go` 文件，处理 HTTP 请求（参数解析、验证、调用 Service）
3. **Service**：在 `service/` 下新增业务逻辑
4. **Model**：如需新数据表，在 `model/` 下定义 GORM 模型
5. **Router**：在 `router/api-router.go` 注册路由，挂载中间件
6. **国际化**：如有提示消息，在 `i18n/locales/` 添加翻译

### 前端页面

1. **Route**：在 `web/default/src/routes/` 下使用 `createFileRoute` 定义路由文件（支持文件路由）
2. **Feature**：在 `web/default/src/features/` 下创建对应功能目录，内含 `components/`、`lib/`、`hooks/`、`api.ts`、`types.ts` 等
3. **i18n**：页面用英文 key 调用 `t('xxx')`，在 `web/default/src/i18n/locales/en.json` 中添加翻译，运行 `bun run i18n:sync` 同步
4. **导航**：如果需要在侧边栏显示，参考现有路由布局（`features/home` 用于首页，`_authenticated` 用于需要登录的布局）

### 完整链路示例

```
用户请求 → router (Gin) → middleware (认证/限流) → controller (解析/验证)
 → service (业务逻辑) → model (数据库操作) → 响应
                                                        ↓
前端页面 ← route (TanStack Router) ← feature (API 请求) ← Backend API
```

---

## 6. 维护风险

### 🔴 高风险

1. **直接使用 `encoding/json`** — AGENTS.md 明确规定所有 JSON 操作必须通过 `common.Marshal/Unmarshal`，但至少有 5+ 个 controller 文件（channel.go、channel-billing.go、channel-test.go、console_migrate.go、deployment.go 等）直接 `import "encoding/json"`。统一封装提供了自定义序列化行为，绕过意味着这些行为在这些路径上是缺失的。

2. **`controller/channel.go` 达 2170 行** — 单文件远超合理范围（建议 <500 行），函数和状态之间没有模块边界，严重影响可维护性和可测试性。

3. **`VERSION` 文件为空** — 根目录的 `VERSION` 文件是 0 字节，但 Dockerfile 和 makefile 都读取 `cat VERSION` 作为构建版本号。这意味着所有构建产物都会使用空字符串替代预期版本。

4. **两套前端主题维护负担** — `web/default/`（React 19 + Rsbuild）和 `web/classic/`（React 18 + Vite + Semi Design）两套 UI 代码库，需要同步功能和修复，维护成本几乎是双倍的。

### 🟡 中风险

1. **测试覆盖率偏低** — 75 个 Go 测试文件对应 646 个源码文件（约 12%），前端 995 个文件零测试文件。计费逻辑、渠道分发、认证等关键路径缺少充分的测试覆盖。

2. **大依赖树** — `go.mod` 164 行，依赖大量第三方库（包括间接依赖），依赖更新和安全补丁跟踪成本高。

3. **`go.sum` 达 310K** — 表示第三方依赖链非常长，增加了构建安全审计的复杂度。

4. **三数据库兼容性** — 同时支持 SQLite、MySQL、PostgreSQL（还有可选的 ClickHouse），ORM 无法完全屏蔽 SQL 方言差异，存在隐式的兼容性风险。

5. **TODO/FIXME 遗留** — 代码中存在 7 处 TODO/FIXME 标记，说明有已知未完成的技术债务。

6. **部分 controller 文件过大** — 除 channel.go 外，user.go (1480行)、channel-test.go (1064行) 明显偏大，逻辑耦合度高。

### 🟢 低风险

1. **Go 版本前沿** — 要求 Go 1.25+（远高于当前稳定版），可能导致某些第三方库不兼容。

2. **Electron 客户端** — `electron/` 目录独立存在但功能状态未明，可能是实验性功能。

3. **AI 辅助代码声明** — PR 模板要求声明 AI 生成代码，增加了协作审核的复杂度。

4. **`.gitignore` 中的特例文件** — 明确排除了 `token_estimator_test.go`（测试文件被排除很奇怪）和 `service/relayconvert/` 下的本地测试文件，说明存在非标准测试实践。
