# AGENTS.md

本文件面向在本仓库工作的 AI Agent（Claude Code / Codex / Cursor / Pi 等），说明工程结构、常用命令、代码规范与必须遵守的原则。人类贡献者请先读 [`README_CN.md`](README_CN.md) 与 [`website-docs/`](website-docs/README.md)。

`cli/` 是独立 Go module，另有更详细的 [`cli/AGENTS.md`](cli/AGENTS.md)（CLI 线协议契约），改 CLI 前必读。

---

## 基本原则（Agent 必须遵守）

1. **不要自动提交 git，只能由人工手动提交。**
   - 禁止执行 `git commit`、`git commit --amend`、`git tag`、`git push`、`git reset`、`git rebase`、`git checkout --`、`git stash drop` 等任何会改变仓库历史或远端状态的命令。
   - 允许的只读 git 操作：`git status`、`git diff`、`git log`、`git show`、`git blame`、`git ls-files`。
   - 完成改动后，请把「改了哪些文件、验证命令与结果」汇报清楚，由人工审阅后自行提交。
   - 不要绕过 hooks（不要使用 `--no-verify`、`SKIP_HOOKS=1`）；hooks 只在人工提交时由人工决定是否跳过。
2. **不要动 secrets**：`.env` 未被 Git 跟踪（`.gitignore` 忽略所有隐藏文件，仅保留 `.env.example`）。不要把真实密钥、token、内网地址写进任何被跟踪的文件，也不要把 `.env` 内容贴进日志、提交信息或 PR 描述。
3. **改动范围最小化**：只改与任务相关的文件，不做顺手重构、不重排格式、不批量重命名；不要为了"顺便优化"而扩大 diff。
4. **不要手改生成物**：见「生成物与产物」一节，生成物由脚本产出。
5. **先验证再声称完成**：任何"已修复/已通过"的结论都必须有实际命令输出支撑（见「验证清单」）。
6. **不确定就先问**：涉及产品行为、接口兼容性、数据迁移、部署方式的取舍，先向人类确认，不要自行假设。

---

## 项目概览

**WeKnora（维娜拉）** 是腾讯开源的企业级知识管理框架：文档解析 → 向量化 → 混合检索 → LLM 推理，同时提供 **RAG 问答**、**Agent 智能推理（工具 / 技能 / 沙箱 / 长期记忆）** 与 **自动 Wiki** 三种能力，三者共享同一知识库。当前版本 `0.8.2`（见 `VERSION`）。

多进程组成：

| 组件 | 目录 | 技术栈 | 说明 |
| --- | --- | --- | --- |
| 主后端 `app` | `cmd/server` + `internal/` | Go 1.26，需 CGO（DuckDB / sqlite-vec） | HTTP API（Gin）+ Agent 引擎 + 任务队列 |
| 文档解析 `docreader` | `docreader/` | Python ≥3.10.18 + gRPC，依赖用 `uv` | 独立进程/容器，被后端通过 gRPC 调用 |
| 前端 `frontend` | `frontend/` | Vue 3.5 + Vite 7 + TS + TDesign + Pinia | SPA，同时含嵌入 Widget 与 Wails 桌面端复用代码 |
| CLI `weknora` | `cli/` | Go 1.26（独立 module） | Agent 友好的命令行与 MCP 工具面 |
| 内置 MCP Server | `internal/mcpserver/` | Go | `/mcp/<endpoint_id>` Streamable HTTP |
| Go SDK | `client/` | Go（独立 module） | 供外部程序调用 API |
| 小程序 | `miniprogram/` | 微信小程序 | 移动端轻量客户端 |
| dsh 插件 | `packages/dsh-weknora/` | TypeScript | DeepSeek Harness 只读工具插件 |
| ~~Python MCP 服务~~ | `mcp-server/` | Python | **已弃用**，仅保留历史版本发布，不要再新增功能 |

## 仓库结构

| 路径 | 内容 |
| --- | --- |
| `cmd/server/` | 后端入口：`main.go`（gin 模式、启动）、`bootstrap.go`、`listen.go`、`shutdown_test.go` |
| `cmd/desktop/`、`cmd/download/`、`cmd/milvus-migrate/` | 桌面端打包、Lite 下载器、Milvus 迁移工具 |
| `internal/container/` | 依赖注入装配（`uber/dig`），`BuildContainer` 是全部服务的组装点 |
| `internal/router/` | 路由注册（`router.go` + 按域拆分的 `routes_*.go`），RBAC 与中间件挂载 |
| `internal/handler/` | HTTP 层：参数校验、DTO 转换、SSE 输出；`dto/`、`session/` 为子包 |
| `internal/application/service/` | 业务逻辑层（约 400 个文件，按领域命名） |
| `internal/application/repository/` | 数据访问层 |
| `internal/types/`、`internal/types/interfaces/` | 领域模型与接口定义（Service/Repository 接口集中在 `interfaces/`） |
| `internal/agent/` | Agent 引擎：`engine.go`、`think.go`、`act.go`、`compaction/`、`tools/`、`skills/`、`approval/`、`token/` |
| `internal/models/` | 模型厂商体系：`providers/`、`catalog/`、`parity/`、`api/`、`chat/`、`embedding/`、`rerank/`、`asr/`、`vlm/` |
| `internal/infrastructure/` | `chunker/`（分块策略）、`docparser/`、`web_search/`、`web_fetch/` |
| `internal/datasource/` | 外部数据源连接器（飞书 / Confluence / GitLab / Notion / 语雀 / 钉钉文档 / RSS…）与调度 |
| `internal/im/` | IM 渠道适配器（企业微信 / 飞书 / Slack / Telegram / 钉钉…），`adapter.go` 是统一接口 |
| `internal/mcp/`、`internal/mcpserver/` | MCP 客户端（外部服务）与内置 MCP 服务端 |
| `internal/sandbox/`、`internal/localsandbox/` | Docker / E2B / Cube 沙箱与 Lite 本机沙箱（Seatbelt） |
| `internal/database/` | 数据库连接与迁移执行，迁移文件在 `migrations/versioned/`（另有 `sqlite/`、`mysql/`、`paradedb/`） |
| `internal/config/`、`config/` | 配置加载（viper + `.env`）；`config/config.yaml`、`builtin_agents.yaml`、`agent_type_presets.yaml`、`prompt_templates/*.yaml` |
| `internal/runtime/`、`internal/logger/`、`internal/tracing/` | 启动环境横幅、日志、Langfuse 链路追踪 |
| `frontend/src/` | `api/`、`stores/`、`views/`、`components/`、`i18n/`、`router/`、`wailsjs/` |
| `website-docs/` | **产品文档唯一来源**（入门/架构/功能/API/客户端/开发 六大板块）+ 官网主页 |
| `docs/` | 仅保留 Swagger 生成物（`docs.go`、`swagger.json/yaml`）、`LITE.md`、图片与 `poc/`；新文档不要放这里 |
| `deploy/`、`helm/`、`docker/`、`docker-compose*.yml` | 部署：Compose、K8s Helm、镜像 Dockerfile |
| `scripts/` | 开发/构建/迁移/校验脚本；`scripts/git-hooks/` 为仓库 Git hooks |
| `migrations/`、`testdata/`、`examples/`、`dataset/`、`misc/`、`Formula/` | 迁移、测试数据、示例、Homebrew formula |

## 架构与数据流

- 分层严格单向：`router → handler → service → repository → DB`；跨层只依赖 `internal/types/interfaces` 中定义的接口，实现在 `internal/container` 里用 dig 装配。新增服务时：先定接口 → 写实现 → 在 `container.BuildContainer` 注册 → 在 `internal/router/routes_*.go` 挂路由。
- 数据库默认 `DB_DRIVER=postgres`（ParadeDB/pgvector），Lite 模式用 `sqlite`；向量库 `RETRIEVE_DRIVER` 支持 postgres / ES / OpenSearch / Milvus / Weaviate / Qdrant / Doris / 腾讯云 VectorDB，存储后端支持本地 / COS / MinIO / S3 / TOS / OSS / KS3 / OBS。
- 文档处理链路：上传 → 对象存储 → 任务队列（asynq）→ docreader 解析（anydoc 处理 Office）→ 分块 → embedding → 向量/关键词索引 → 检索（混合 + 重排 + 可选 GraphRAG）。
- 配置来源：`.env`（约 150 个环境变量，参考 `.env.example` 的注释）> `config/*.yaml` 默认值。**提示词内容优先放在 `config/prompt_templates/*.yaml`，不要硬编码到 Go 里**（代码中通过 `xxx_id` 引用）。

## 常用命令

### 开发模式（推荐，本地跑后端/前端 + Docker 跑基础设施）

```bash
cp .env.example .env     # 必需；.env.local 可选覆盖
make dev-start           # 启动 ParadeDB/Postgres + Redis + docreader(+Langfuse)
make dev-app             # 另开终端：本地运行后端（.air.toml 支持热重载）
make dev-frontend        # 另开终端：前端 dev server（5173，代理 /api,/files）
make dev-status / dev-logs / dev-stop / dev-restart
```

端口：后端 `8080`、前端 dev `5173`、`vite preview` `4173`、Langfuse `3000`、Web UI（容器）`80`。

### 构建 / 测试 / 检查（根 module）

```bash
go build ./cmd/server          # 只编译后端（CI 同款，推荐用于快速校验）
make build                     # 构建 ./WeKnora
make test                      # go test -v ./...
go test -count=1 ./internal/application/service/...   # 定向包测试
go test -run TestXxx ./internal/xxx/                  # 单个测试
make fmt                       # go fmt ./...  ⚠ 全仓格式化，仅工作区干净时运行
make lint                      # golangci-lint run
golangci-lint run --new-from-rev=origin/main ./...    # 只报本次新增问题（PR 口径）
make docs                      # 重新生成 Swagger（改了 @Router/@Param 注释后必须执行）
```

### 其他模块

```bash
cd frontend && npm install && npm run dev          # 前端开发
cd frontend && npm test && npm run type-check && npm run build
../scripts/verify_frontend_pr.sh                   # 前端 CI 同款（test→type-check→build）
cd docreader && uv sync && uv run python -m unittest discover -s tests -v
cd cli && make build && make test && make lint
cd packages/dsh-weknora && npm test
node tests/miniprogram/miniprogram.test.js          # 小程序集成测试
```

### 数据库迁移

```bash
make migrate-up        # 应用迁移（scripts/migrate.sh，读取 .env）
make migrate-down      # 回滚
make migrate-create NAME=xxx   # 新建一对 up/down SQL
```

迁移文件放 `migrations/versioned/`，命名 `0000NN_<name>.up.sql` / `.down.sql`，**up 与 down 必须成对且可逆**；SQLite 兼容性需在 `internal/database/` 的相关测试中确认。

### 许可证与模型目录校验

```bash
scripts/check-license-bundle.sh        # 新增/升级依赖后必跑
make model-catalog-check               # 模型厂商目录不变量校验
make model-catalog-generate            # 重新生成模型目录（models.json）
```

## 代码规范

- **格式化**：Go 用 **gofumpt**（比 gofmt 更严格），行宽上限 **120**（`.golangci.yml`，启用 `lll`、`govet`、`revive`）。改动过的 Go 文件提交前必须格式化。
- **命名与注释**：导出符号要有注释；包的注释写在 `package` 前；Go 代码注释与产品文案遵循仓库既有语言习惯（Go 注释多为中文，标识符英文）。
- **提交信息**：遵循 [Conventional Commits](https://www.conventionalcommits.org/)（`feat:` / `fix:` / `docs:` / `test:` / `refactor:` / `chore:`，可带 scope 如 `fix(embed):`）——**注意这只约束人工提交，Agent 不执行提交**。
- **前端 i18n**：新增界面文案必须补齐 5 个 locale（`frontend/src/i18n/locales/`：`zh-CN`、`en-US`、`ja-JP`、`ko-KR`、`ru-RU`），并跑 `npm run check-i18n`；Langfuse/审计等注册表类文案另有对应测试。
- **错误处理**：返回错误用仓库既有包装风格（`internal/errors/`），对外 API 错误结构不要擅自改动；不要吞掉错误、不要用 panic 处理可预期失败。
- **接口兼容**：已发布的 HTTP 路由、CLI 输出信封（`cli/AGENTS.md` 的线协议）、配置项与环境变量默认视为兼容契约；破坏性变更必须显式标注并说明迁移方式。
- **测试**：优先单元测试，善用 `go-sqlmock`、`miniredis`、内存 SQLite；涉及 SQL 的改动要同时考虑 postgres 与 sqlite 两条路径。测试文件与被测文件同目录，命名 `xxx_test.go`。
- **依赖**：新增第三方依赖要同步更新 `THIRD_PARTY_NOTICES.md` 与 `licenses/`，跑 `scripts/check-license-bundle.sh`。
- **Git hooks**：`./scripts/install-git-hooks.sh` 可把 `core.hooksPath` 指向 `scripts/git-hooks`（pre-commit 空白字符 + gofmt + golangci-lint；pre-push 按 CI 跑 go vet/test/build）。若改动这批 hooks，需跑 `scripts/test-git-hooks.sh`。

## 生成物与产物（不要手改）

| 文件 / 目录 | 生成方式 |
| --- | --- |
| `docs/docs.go`、`docs/swagger.json`、`docs/swagger.yaml` | `make docs`（swag），由 router/handler 注释生成 |
| `config/models.json` | `make model-catalog-generate`（`MODELS_CONFIG` 可覆盖） |
| `frontend/dist/`、`web/`、`WeKnora`、`WeKnora-lite`、`tmp/` | 构建产物，已被 `.gitignore` 忽略 |
| `licenses/`、`THIRD_PARTY_NOTICES.md` | `scripts/copy-licenses.sh` 等脚本维护 |

## 验证清单

按改动范围选择最小但有说服力的组合，并在汇报中给出实际命令与输出：

| 改动范围 | 建议验证 |
| --- | --- |
| 单个 Go 包 | `gofmt -l <files>` → `go vet <pkg>` → `go test -count=1 <pkg>` → `go build ./cmd/server` |
| 跨包 / 核心链路 | `golangci-lint run --new-from-rev=origin/main ./...` + 相关包 `go test` + `go build ./cmd/server` |
| 前端 | `scripts/verify_frontend_pr.sh`（test → type-check → build），UI 改动附截图 |
| docreader | `uv run python -m unittest discover -s tests -v` |
| CLI | `cd cli && go vet ./... && go test -count=1 ./...` |
| 路由 / Swagger 注释 | `make docs` 并提交生成物差异 |
| 迁移 | up + down 均可执行；`internal/database/` 测试通过 |
| 依赖 / 许可证 | `scripts/check-license-bundle.sh` |

若全仓检查因无关的基线问题或环境依赖失败（例如缺少本地基础设施），如实说明失败命令与原因，同时提供本次改动范围内的定向验证结果。

## 参考文档

- [`README_CN.md`](README_CN.md) / [`README.md`](README.md)：项目介绍、部署、贡献流程
- [`website-docs/06-development/01-dev-guide.md`](website-docs/06-development/01-dev-guide.md)：开发环境、Makefile 全览、测试体系、CI 与提交
- [`website-docs/06-development/03-extension-points.md`](website-docs/06-development/03-extension-points.md)：9 类扩展点（解析器 / 分块 / 检索 / 模型厂商 / 联网搜索 / 数据源 / IM / Agent 工具 / 存储后端）的新增步骤
- [`website-docs/06-development/02-database-schema.md`](website-docs/06-development/02-database-schema.md)、[`05-agent-prompts.md`](website-docs/06-development/05-agent-prompts.md)、[`04-sandbox-deployment.md`](website-docs/06-development/04-sandbox-deployment.md)
- [`cli/AGENTS.md`](cli/AGENTS.md)：CLI 线协议与退出码契约
- [`CHANGELOG.md`](CHANGELOG.md)：版本变更（`### BREAKING` 为破坏性变更权威来源）
- `.env.example`：全部环境变量及注释