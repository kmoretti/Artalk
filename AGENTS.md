# AGENTS.md

本文档为后续在该仓库工作的编码 Agent 提供项目结构、功能边界、运行方式和常见改动入口。它优先服务于“看懂本仓库、修改时找到正确位置、不破坏 Fork 定制功能”的目标。

## 项目定位

Artalk 是一个可自托管的评论系统：Go 后端负责数据、审核、通知和 API，前端提供嵌入网页的评论组件和管理端。本仓库是 `LiuShen-Fork/Artalk`，基于上游 `ArtalkJS/Artalk` 的个人维护 Fork。

- 当前版本基线：`v2.10.0`，版本号在 `internal/config/version.go`。
- Fork 不发布 npm 前端包；评论组件仍沿用上游客户端，本仓库重点维护后端、管理端和自托管部署。
- 本地和生成文档默认偏中文，配置模板、管理端默认语言和默认文案也以中文为主。
- 上游变更通过 `upstream` 远程同步。Rebase 上游后，最容易冲突的位置是审核管线、配置结构、管理端页面、发布工作流和 AI 相关服务，应重点检查这些区域。

## 技术栈与环境

| 部分 | 技术 |
| --- | --- |
| 后端 | Go 1.26.5、Cobra CLI、Fiber HTTP、GORM、koanf 配置 |
| 数据库 | SQLite（libtnb 纯 Go 驱动）、MySQL、PostgreSQL、SQL Server |
| 前端 | TypeScript、Vite、Vanilla JS 客户端；Vue 3 管理端 |
| 包管理 | pnpm 10.33.2（monorepo） |
| Node | >=22.19.0 |
| 文档 | VitePress（`docs/`） |
| 发布 | GitHub Actions + Docker，输出 `willowgod/artalk` 镜像 |

常用命令：

```powershell
# 启动后端（开发调试时不需要先构建前端）
go run . -c .\artalk.yml server --host 127.0.0.1 --port 23366

# 创建/修改管理员
go run . -c .\artalk.yml admin --name admin --email admin@example.com --password "强密码"

# 管理端开发服务器
pnpm dev:sidebar

# Go 测试
make test               # 等价于 go test ./...
pnpm test               # 运行 ui/artalk 客户端测试

# 构建前端产物并复制到 public（发布前需要）
make build-frontend
```

## 顶层目录

| 路径 | 作用 |
| --- | --- |
| `main.go` | 入口；`embed public/ i18n/ conf/`，通过 `internal/pkged` 提供给静态文件和语言包 |
| `cmd/` | CLI 命令：`server`、`admin`、`export`、`import`、`config`、`gen`、`upgrade`、`version` |
| `server/` | Fiber 应用、HTTP handler、middleware、公共响应/鉴权/参数工具 |
| `internal/core/` | `App` 容器、服务依赖注入、服务生命周期、默认服务实现 |
| `internal/config/` | Config 结构、YAML 解析、`ATK_` 环境变量、默认值 patch、配置模板 meta 生成 |
| `internal/db/` | 各数据库连接、DSN、SQLite/MySQL/Postgres/SQL Server 打开逻辑 |
| `internal/dao/` | DAO 层：GORM 查询、缓存适配、迁移、Cook 实体、页面/通知/投票辅助 |
| `internal/entity/` | GORM 模型和对外 Cooked 结构 |
| `internal/anti_spam/` | Akismet、腾讯云、阿里云、AI 审核、关键词审核、同步拦截 |
| `internal/auth/` | 社交登录、通用 OAuth 2.0 Provider、SSO、回调页 |
| `internal/captcha/` | 图片/Cloudflare Turnstile/ReCaptcha/HCaptcha/极验 |
| `internal/artransfer/` | 评论数据迁移导入导出 |
| `internal/email/`、`internal/notify_pusher/` | 邮件队列与管理员多渠道通知 |
| `internal/ip_region/` | ip2region 查询、IPv4/IPv6 分离、数据抓取 |
| `internal/cache/` | builtin/BigCache、Redis、Memcache 统一缓存 |
| `internal/limiter/` | 基于 IP 的评论/验证码操作限制 |
| `internal/i18n/`、`internal/log/`、`internal/template/`、`internal/utils/` | 语言包、日志、模板渲染、工具函数 |
| `ui/` | pnpm workspaces：`artalk` 客户端、`artalk-sidebar` 管理端、插件与 eslint 插件 |
| `test/` | Go 集成测试入口、fixtures、测试配置 |
| `docs/` | VitePress 文档站点 |
| `public/` | 可被服务端嵌入的生成产物：客户端 dist、sidebar、插件 |
| `conf/` | 多语言 YAML 配置模板 |
| `i18n/` | 后端本地化 YAML |

`public/sidebar` 和 `public/dist` 已在 `.gitignore` 中，属于生成物；它们由 `scripts/build-frontend.sh` 从 `ui/` 复制出来，`go build` 时嵌入。

## 后端生命周期

启动流程：

1. `cmd.New()` 预解析 `--config` / `--workdir`，确定配置文件路径和数据目录；无 `data/` 时可能切换到 XDG 数据目录。
2. `cmd.Launch()` 挂载命令并执行。普通命令在 `PreRun` 中 `config.NewFromFile()` 加载配置，创建 `core.NewApp(config)` 并 `Bootstrap()`。
3. `Bootstrap()` 依次初始化时区、i18n、日志、DAO（含数据库连接和 AutoMigrate）、缓存、配置同步和所有服务。
4. `server.Serve()` 创建 Fiber App、注册中间件和 `/api/v2` 路由，并监听 `conf.Host:conf.Port`。
5. 退出或 `settings` 接口应用新配置时触发 `OnTerminate` / `Restart`，关闭数据库、缓存并释放服务。

核心容器位于 `internal/core/base.go`：

- `App` 持有 `config`、`dao`、`cache`、`service map[string]Service`。
- 默认服务注入在 `NewApp()`：Email、IPRegion、Notify、AntiSpam、AIAssistant。
- 服务只实现 `Init()` / `Dispose()`，内部通过 `app.Dao()` 或泛型 `AppService[*xxx]` 协作。
- 新增业务服务时应在 `injectDefaultServices()` 中登记，并考虑是否需要在终止/重启时清理。

配置解析在 `internal/config/`：

- `Config` 结构定义在 `internal/config/config.go`，通过 koanf 读取 YAML，再合并 `ATK_` 环境变量。
- `internal/config/meta/` 从 Config 结构反射并解析配置模板注释，生成环境变量映射；生成结果在 `internal/config/cache.go`。
- 修改配置项时通常需要同步改 `internal/config/config.go`、`conf/artalk.example*.yml`，再运行 `go generate ./internal/config`。
- 许多默认值和兼容逻辑集中在 `internal/config/base.go` 的 patch 函数；Fork 的 AI 配置默认值也在这里。
- `gen config` 会按 `ATK_LOCALE` 生成带随机 `app_key` 的配置模板；`app_key` 同时用于 JWT 签名。

## HTTP 与鉴权

Fiber 路由入口在 `server/server.go`：

- 前缀为 `/api/v2`；另有 `/swagger/*`、静态资源和 `/` 到 `sidebar/` 的重定向。
- 全局中间件：RequestID、访问日志、Swagger、CORS、操作限流；调试模式额外启用 pprof。
- 公共接口包括评论创建/列表/获取、投票、页面 PV、统计、通知、上传、验证码、配置、版本、登录注册和用户接口。
- 管理接口通过 `common.AdminGuard()` 保护，包括评论更新/删除、站点/页面/用户管理、Dashboard、审核记录、AI 助手日志、缓存、邮件、设置和迁移。
- JWT 使用 HS256，密钥来自 `config.app_key`；Token 可从 query、form 或 `Authorization: Bearer` 获取，见 `server/common/auth.go`。
- 大多数 handler 只在文件内定义路由；`server/server.go` 的 `admin()` 是管理接口的注册清单。新增接口时先看 `server/handler/*.go` 中现有 handler 模式，再在 `server.go` 注册。

## 数据模型

主要 GORM 实体在 `internal/entity/`：

- `Site`：站点，`name` 唯一，`urls` 保存站点 URL。
- `Page`：评论页面，`key + site_name` 定位页面，含 `admin_only`、PV、投票数字段。
- `User`：评论用户/管理员；`Password` 支持 bcrypt 与旧 MD5/明文兼容格式。
- `Comment`：评论，含 `page_key`、`site_name`、`user_id`、父级 `rid`、根节点 `root_id`、待审/折叠/置顶/认证等状态。
- `Notify`：站内通知和邮件投递状态。
- `Vote`：评论/页面投票。
- `AuthIdentity`、`UserEmailVerify`：第三方身份和邮箱验证。
- Fork 新增：`ModerationLog`（审核结果记录）和 `AIAssistantLog`（AI 助手调用记录）。

`internal/dao/migrate.go` 使用 `AutoMigrate` 维护表结构，并处理历史数据迁移（`root_id` 递归 CTE、页面合并、删除外键约束）。新增实体后必须把模型加入 `MigrateModels()` 的 AutoMigrate 列表。

DAO 层同时支持可配置缓存：

- `internal/cache/base.go` 封装 BigCache/Redis/Memcache，序列化格式为 msgpack。
- DAO 内层缓存通过 `DaoCache` 适配；仅 `cache.enabled` 时才启用，启动时可按 `cache.warm_up` 预热。

## 评论发布链路

核心入口是 `server/handler/comment_create.go`，同步流程：

1. 校验昵称、邮箱、链接、内容、页面和站点。
2. 查找或创建 Page，检查 `admin_only`/管理员要求，并校验父评论。
3. 通过 JWT 取用户；未登录且允许匿名时创建/复用匿名 User。
4. 构造 `Comment`，非管理员且 `moderator.pending_default` 开启时默认待审。
5. Fork 的 `moderator.intercept` 在持久化前同步执行：命中关键词直接返回 HTTP 400，绝不落库，也不发通知，并记录 `ModerationLog`。
6. 保存评论；异步 `commentCreatedJobs()` 内执行页面标题抓取、反垃圾检查、通知推送和 AI 助手回复。
7. 返回 `CookedComment`，并在 `ip_region.enabled` 时附带 IP 属地。

反垃圾检查在 `internal/anti_spam/base.go`：

- 可同时启用 Akismet、腾讯云、阿里云、AI、关键词多个 checker，所有启用的 checker 都会执行。
- 结果为 block 时把评论改为待审；replace 时更新内容；API 失败时由 `moderator.api_fail_block` 决定放行或待审。
- 正常 pass 不写日志；block/error/replace/reject 写入 `ModerationLog`，默认保留 90 天。
- `internal/core/service_anti_spam.go` 负责把服务层结果持久化和管理日志保留。

Fork 的 AI 审核在 `internal/anti_spam/ai.go`：

- 支持 `responses`、`chat_completions`、`anthropic_messages`、`deepseek_json_output`、`jev` 等协议。
- 内置结构化输出 Schema，要求返回 `sensitive` 和 `reason`。
- 只把 Nickname 和正文一起送入审核，正文先转成纯文本，Markdown 图片/链接等会被归一化。
- `moderator.ai` 相关配置集中在 `internal/config/config.go`，默认值在 `base.go`。

## Fork 定制功能对照

| 功能 | 后端位置 | 配置 | 管理端 |
| --- | --- | --- | --- |
| AI 评论审核 | `internal/anti_spam/ai.go`、`internal/core/service_anti_spam.go` | `moderator.ai` | 设置页可编辑提示词 |
| 同步关键词拦截 | `internal/anti_spam/intercept.go`、`server/handler/comment_create.go` | `moderator.intercept` | 无独立管理页 |
| 审核记录 | `internal/entity/moderation_log.go`、`server/handler/moderation_log_*.go` | 自动记录 | `/moderation` |
| AI 评论助手 | `internal/core/service_ai_assistant.go`、`server/handler/ai_assistant_log_*.go` | `ai_assistant` | `/comments` AI 筛选、AI 助手日志 |
| 通用 OAuth 2.0 | `internal/auth/generic/`、`internal/auth/providers.go` | `auth.generic` | 登录弹窗显示 `label` |
| OIDC SSO 交换 | `server/handler/auth_sso_exchange.go` | `auth.sso` | 无独立管理页 |
| Lsky Pro 图床 | `server/handler/upload_lsky.go` | `img_upload.lsky` | 图片上传设置 |
| IPv6 属地 | `internal/ip_region/` | `ip_region.db_path_v6` | IP 属地展示 |
| 发布流程 | `.github/workflows/` | GitHub Actions inputs | 手动触发 |

AI 助手的行为要点：

- 收到“@助手名”或直接回复助手评论时触发；助手有独立用户身份，`User.IsAIAssistant = true`，不会给自己发通知。
- 上下文包括页面正文、当前评论和最多 20 条同线程历史，按时间顺序发送，便于 provider prefix cache。
- 默认每日 40 次、每用户每小时 5 次；超限时直接保存固定回复，不调用 AI 服务。
- 页面抓取失败或 AI 请求失败时保存固定兜底文案，并把详细错误写入 `AIAssistantLog`。
- AI 助手评论在页面评论列表可通过 `split_ai` 单独返回，管理端评论页也提供 AI 筛选。

通用 OAuth Provider 在 `internal/auth/generic/provider.go`：

- 通过 `goth.Provider` 接口实现标准 Authorization Code 流程。
- 从用户信息 JSON 中按常见字段/嵌套容器提取 ID、昵称、邮箱、头像和主页链接，不绑定固定 IdP。
- 配置项缺失或 URL 非法时只记录日志、不注册 Provider，所以配置错误不会导致完全启动失败。

## 前端结构

`ui/` 是 pnpm workspace：

- `ui/artalk`：嵌入网页的 Artalk 客户端。Vanilla TS + Vite，构建产物含 `Artalk` / `ArtalkLite` 双入口和 i18n。入口在 `src/artalk.ts`、`src/main.ts`；架构为 Context + DI + 插件系统，清单渲染、编辑器、列表分页、暗色、懒加载等功能都在 `src/` 子目录。
- `ui/artalk-sidebar`：Vue 3 管理端，主要页面在 `src/pages/`：Dashboard、评论、审核、页面、用户、站点、迁移、设置、登录。状态使用 Pinia，导航在 `AppNavigationMenu.ts`。
- 设置页从后端获取配置模板和当前 YAML，把 YAML 转化为树形控件；敏感字段由 `src/lib/settings-sensitive.ts` 隐藏。
- `ui/plugin-auth`、`ui/plugin-katex`、`ui/plugin-lightbox`：官方插件；`ui/plugin-kit` 是插件开发 SDK。
- `scripts/build-frontend.sh` 构建 `ui` 产物并复制到 `public/`，随后 Go 后端嵌入。修改管理端/客户端并在正式验证时需要运行它。

## 测试与验证

- Go 测试入口：`test/app.go` 创建临时 SQLite、加载 `test/fixtures/` YAML fixtures，再构造完整的 `core.App`。测试会 chdir 到仓库根目录并使用 `pkged.SetFS()` 加载目录文件。
- 常见运行方式：`go test ./...` 或 `make test`；CI 使用 `go vet ./...` + `gotestsum` 全量测试。
- 前端单元测试：`pnpm test`；客户端另有 Playwright e2e 脚本，见 `scripts/frontend-e2e-test.sh`。
- GitHub Actions 工作流在 `.github/workflows/`：`test-app.yml` 手动测试、`release.yml` 手动发布、`build*.yml` 构建和打包，发布镜像为 `willowgod/artalk`。
- 修改后端逻辑后至少跑相关包测试；修改管理端/客户端后至少跑 `vue-tsc` 或包内 lint/test。

## 常见改动入口

- 新增配置项：改 `internal/config/config.go` 的 `Config` 结构，补 `conf/artalk.example*.yml` 模板注释，然后 `go generate ./internal/config`；若要在管理端设置页可编辑，确认它出现在模板树中；若是密钥，追加到 `ui/artalk-sidebar/src/lib/settings-sensitive.ts`。
- 新增数据表：在 `internal/entity/` 建模型，加入 `internal/dao/migrate.go` 的 AutoMigrate 列表，并为列表查询 handler 增加分页/过滤。
- 新增 API：在 `server/handler/` 增加 handler 函数，按 `public` 或 `admin` 分组在 `server/server.go` 注册。
- 修改本地化文案：后端文案在 `i18n/*.yml`，前端文案在 `ui/artalk/src/i18n/` 和 `ui/artalk-sidebar/src/i18n/`；上游管理端语言文件与后端 i18n 是两套体系。
- 重建静态产物：运行 `make build-frontend` 或 `scripts/build-frontend.sh`，然后重新 `go build`；直接改 `public/` 不会持久生效，因为该目录产物被 gitignore。
- 修改配置文档：`make update-conf-docs` 会根据 meta 重新生成 `docs/docs/*/guide/env.md`。

## 需要注意的坑

- `cmd upgrade` 仍通过 GitHub Releases 检测 `ArtalkJS/Artalk`，不是本 Fork 的发布仓库；Fork 的 Docker 发布走 `.github/workflows/release.yml`。
- `app_key` 未配置时会随机生成，JWT 和登录态会因此不稳定，生产环境必须显式固定。
- `http.proxy_header` 旧版本默认是 `X-Forwarded-For`，直接暴露端口时可能导致 IP 伪造，新配置模板里应显式留空。
- 评论创建后的反垃圾检查是异步的，`intercept` 是同步硬拦截；两者语义不同，不要在异步反垃圾中依赖“未落库”。
- `settings` 接口会直接覆盖当前 YAML 并触发进程内 Restart，涉及数据库连接、缓存和服务的重新初始化；不要在热更新路径上假设单例全部保留。
- SQLite 已换为纯 Go 驱动，构建不需要 CGO；`Makefile` 默认 `CGO_ENABLED=0`。
- Fork 定制集中在审核、认证、上传、IP 属地、管理端和发布流程，同步上游时优先检查这些区域而不是整个仓库机械 rebase。