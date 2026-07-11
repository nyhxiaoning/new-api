# New API 项目分析

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
