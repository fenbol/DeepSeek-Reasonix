[Uploading DIRECTORY_ANALYSIS.md…]()
# Reasonix 目录分析报告

> **项目**: DeepSeek-Reasonix v1.16.0
> **语言**: Go 1.25 (toolchain go1.26.4) · TypeScript (前端/Workers)
> **定位**: DeepSeek 原生的 AI 编码代理，单一静态 Go 二进制，围绕 DeepSeek 前缀缓存优化令牌成本
> **模块路径**: reasonix (无 vanity URL)
> **许可**: MIT

---

## 目录

- [项目概览](#项目概览)
- [顶层目录速查表](#顶层目录速查表)
- [1. 根目录配置文件](#1-根目录配置文件)
- [2. internal/ — Go 核心内核](#2-internal--go-核心内核)
- [3. cmd/ — 入口点](#3-cmd--入口点)
- [4. desktop/ — Wails 桌面应用](#4-desktop--wails-桌面应用)
- [5. workers/ — Cloudflare Workers 服务](#5-workers--cloudflare-workers-服务)
- [6. site/ — Astro 静态站点](#6-site--astro-静态站点)
- [7. docs/ — 文档](#7-docs--文档)
- [8. npm/ — npm 分发包装](#8-npm--npm-分发包装)
- [9. tools/ — 辅助工具](#9-tools--辅助工具)
- [10. benchmarks/ — 端到端基准测试](#10-benchmarks--端到端基准测试)
- [11. scripts/ — 构建/发布/缓存脚本](#11-scripts--构建发布缓存脚本)
- [12. .github/ — CI/CD 流水线](#12-github--cicd-流水线)
- [架构依赖关系总览](#架构依赖关系总览)

---

## 项目概览

Reasonix 1.0 是从 TypeScript 0.x 版本完全用 Go 重写的 AI 编码代理。它的核心设计原则是：

1. **配置驱动** — Provider、agent、工具、插件全部在 reasonix.toml 中声明，无硬编码模型。
2. **缓存优先 (Cache-first)** — 系统提示前缀（基础提示 + 工具 + 记忆）必须在多轮对话间保持字节稳定，以维持 DeepSeek 自动前缀缓存的热度。
3. **传输无关** — 单一 control.Controller 位于所有前端之后（终端 TUI、HTTP/SSE serve、Wails 桌面应用）。
4. **零摩擦分发** — CGO_ENABLED=0 单一二进制，一条命令交叉编译到 6 个目标平台。
5. **插件驱动** — 外部工具作为子进程通过 stdio JSON-RPC (MCP 兼容) 运行，内置工具在编译时自注册。

### 三前端架构

```
                 ┌── internal/cli (终端 TUI — Bubble Tea)
control.Controller ├── internal/serve (HTTP/SSE 服务器)
                 └── desktop/ (Wails 桌面应用 — React + Go)
```

所有前端共享同一个 internal/control/controller.go 中的 Controller，它导入了 23 个内部包作为会话驱动。

---

## 顶层目录速查表

| 目录 | 技术栈 | 职责 |
|---|---|---|
| internal/ | Go | 核心内核 — 53 个包，覆盖 agent、provider、tool、config、memory、skill、plugin、serve、sandbox 等 |
| cmd/ | Go | 三个二进制入口：reasonix(CLI)、e2ebench(基准测试)、reasonix-plugin-example(MCP 插件示例) |
| desktop/ | Go + React/TS | Wails 桌面应用后端 + 前端 (Vite + React + TypeScript) |
| workers/ | TS (Hono) | 3 个 Cloudflare Workers：accounts(身份)、crash-report(崩溃+注册表)、forum(社区论坛) |
| site/ | Astro | 静态营销/文档/社区站点，部署到 GitHub Pages |
| docs/ | Markdown | 工程规范、用户指南、Bot 指南等（中英双语） |
| npm/ | JS | npm 包包装器，通过 npm i -g reasonix 分发 Go 二进制 |
| tools/ | Go | write_heartbeat 辅助工具 |
| benchmarks/ | Go + Shell | 端到端基准测试框架与任务集 |
| scripts/ | Bash + Node | 构建/发布/缓存守护/缓存影响检查脚本 |
| .github/ | YAML | CI/CD 工作流、Issue 模板、CODEOWNERS、dependabot |
| tutorial/ | Markdown | 本报告所在目录 |

---

## 1. 根目录配置文件

### go.mod

模块声明文件。模块路径为 reasonix，Go 1.25 最低要求。关键依赖揭示了系统的技术面：

| 关注点 | 依赖 |
|---|---|
| TUI 层 | charm.land/bubbles/v2、bubbletea/v2、lipgloss/v2、go-runewidth、uniseg |
| 语法/Markdown | chroma/v2 (语法高亮)、tree-sitter (JS/Python/Rust/TS)、goldmark (Markdown) |
| 配置 | BurntSushi/toml、joho/godotenv、yaml.v3 |
| 安全/密钥 | zalando/go-keyring (OS 密钥链)、golang.org/x/crypto |
| IM Bot | larksuite/oapi-sdk-go/v3 (飞书 SDK) |
| Shell 执行 | mvdan.cc/sh/v3 (沙箱化 shell)、sabhiram/go-gitignore、doublestar/v4 |
| 通知 | go-toast/v2 |
| 测试工具 | go.uber.org/goleak (goroutine 泄漏测试)、golang.org/x/mod |

纯 Go / CGO-free（CLI 构建路径），Wails 桌面构建除外（需 CGO 用于原生 webview）。

### Makefile

本地构建/测试/hook 编排，7 个目标：build、vet、fmt、test、hooks、cross、clean。

- build 编译两个二进制 (bin/reasonix + bin/reasonix-plugin-example)，CGO_ENABLED=0
- cross 手动循环 6 个 GOOS/GOARCH 组合到 dist/
- hooks 设置 core.hooksPath 为 .githooks
- 版本号通过 git describe --tags --always 注入 ldflags

### .golangci.yml

CI lint 配置 (golangci-lint v2)。启用 errcheck、govet、ineffassign、staticcheck、unused。

- errcheck 排除文件关闭、HTTP body 关闭等 — 务实地减少噪音
- staticcheck 启用 all 但退出 ST1005、ST1018、QF1001
- 测试范围内抑制 SA5011 (t.Fatal 守卫后的 nil-deref)
- max-issues-per-linter: 0 / max-same-issues: 0 — 无 issue 上限

### .goreleaser.yaml

CLI 发布自动化 (GoReleaser v2)。构建 reasonix for darwin/linux/windows x amd64/arm64，CGO_ENABLED=0，-trimpath，ldflag 版本注入。

- 归档：unix 用 tar.gz，Windows 用 zip，附 SHA256 校验文件
- Homebrew cask 发布到 esengine/homebrew-reasonix，post-install 运行 xattr -dr com.apple.quarantine 清除 Gatekeeper 隔离
- rc 标签不劫持稳定 tap

### reasonix.example.toml

详尽且注释丰富的参考配置（259 行），文档化了完整的配置面：

- [agent]: 系统提示、max_steps、压缩比率 (soft_compact_ratio=0.5, compact_ratio=0.8, compact_force_ratio=0.9, tool_result_snip_ratio=0.6) — 这些比率是缓存优先前缀稳定性策略的杠杆
- [[providers]]: 每个是厂商端点 (kind: openai | anthropic)，含 DeepSeek 多模型与 Anthropic Claude 示例
- [sandbox]: 工作空间限制 (workspace_root, allow_write, forbid_read)
- [[plugins]]: MCP stdio 服务器
- [bot]: 多通道 IM Bot (QQ/飞书/微信) 配置块
- 解析顺序: flag > ./reasonix.toml > ~/.reasonix/config.toml > 默认值

### .env.example

最小密钥模板 — 仅 DEEPSEEK_API_KEY 和 MIMO_API_KEY。头部明确说明只有密钥属于这里。

### REASONIX.md

项目"记忆"文件（Reasonix 的 CLAUDE.md 等价物），加载到每个 agent 会话的系统提示中。包含：

- 约定：Go 内核在 internal/，一个包一个关注点；单一 control.Controller；缓存优先规则
- 推送前 CI 模拟命令 (gofmt、vet、定向测试)
- 导入循环规则与测试循环陷阱
- 缓存影响 PR 元数据要求 (Cache-impact、Cache-guard、System-prompt-review 行)

### dev

启动 Wails 桌面开发服务器的 bash 包装器。从 root:branch 哈希派生确定性 Vite 端口 (5173 + hash%100) 以避免多工作树冲突。

### prod_test

本地生产风格桌面包构建脚本。自动检测平台，从分支+sha 合成 semver 版本，自动安装缺失工具 (pnpm、wails、create-dmg、nfpm、makensis)。

### .githooks/pre-push

Go 原生推送前门控，仅运行 go vet ./... — 快速跨平台检查。通过 make hooks 安装。

---

## 2. internal/ — Go 核心内核

internal/ 是项目的心脏，包含 53 个包。按功能关注点分组如下：

### 2.1 架构脊柱

依赖流：

```
config ─┐
boot ──► control ──► agent ──► provider (openai|anthropic)
   │       │   │       │   └──► tool ──► tool/builtin
   │       │   │       ├──► memory, memorycompiler, evidence, planmode, jobs, ...
   │       │   └──────► cli (TUI), serve (HTTP/SSE), acp (JSON-RPC), bot (IM gateways)
   │       └──────────► guardian (safety sub-agent), hook, permission, sandbox, checkpoint
```

关键包的文档注释定义了架构契约：

- **boot** (`internal/boot/boot.go:1-9`) — "将用户配置变成前端可驱动的 Controller 的唯一地方"
- **control** (`internal/control/controller.go:1-11`) — "传输无关的会话驱动器，每个前端背后的一个编排层"
- **control** 导入了 **23 个内部包** (`controller.go:32-56`)，是整个系统的依赖枢纽

### 2.2 Agent 核心与编排

| 包 | 职责 | 关键文件 |
|---|---|---|
| `agent` | Agent 主循环 — 推理、工具调用、压缩、缓存命中守护 | `agent.go`, `cache*.go`, `compact*.go`, `prune*.go`, `parallel_tasks.go`, `subagent_registry*.go`, `task.go`, `ask.go` |
| `control` | 传输无关的 Controller — 23 个内部包的依赖枢纽 | `controller.go` |
| `boot` | 配置到 Controller 的唯一装配点 | `boot.go` |
| `history` | 对话历史管理 — 工具输出、消息 | `tool.go` |
| `evidence` | 证据收集与验证 | — |
| `planmode` | 计划模式策略与标记 | `policy.go` (20KB), `marker_test.go`, `reconcile_test.go` |
| `checkpoint` | 检查点与回退 — 基于文件快照（非 git） | — |
| `guardian` | 安全子代理 — 危险操作审查 | — |
| `jobs` | 后台任务管理 | — |
| `hook` | 生命周期钩子 | — |

### 2.3 Provider（模型后端）

| 包 | 职责 | 关键文件 |
|---|---|---|
| `provider` | Provider 接口、注册表、重试、schema 规范化 | `provider.go` (23KB), `retry.go` (7KB), `schema_canonicalize.go` |
| `provider/openai` | OpenAI 兼容端点（DeepSeek 默认走此）— 模型获取、host 解析、思考/effort | `openai.go` (29KB), `fetch_models.go`, `host.go`, `think.go` |
| `provider/anthropic` | Anthropic Claude 端点 — extended thinking | `anthropic.go` (22KB) |

**设计亮点**：不导入任何 SDK，两个 provider 都使用手写 net/http 客户端。Provider 通过 init() 自注册，cmd/reasonix/main.go 通过 blank import 触发。

### 2.4 工具系统

| 包 | 职责 | 关键文件 |
|---|---|---|
| `tool` | 工具接口、注册表、契约、进度 | `tool.go` (10KB), `contract.go` (3KB), `progress.go` |
| `tool/builtin` | 全部内置工具实现（22+ 工具） | 见下表 |
| `tool/sessiontool` | 会话级工具管理 | `sessiontool.go` (7.7KB) |
| `lsp` | LSP 集成 — 定义、引用、悬停、诊断 | `tool.go` (3.5KB), 测试数据含 Go/Rust/TypeScript |

#### 内置工具清单 (tool/builtin/)

| 文件 | 工具 | 说明 |
|---|---|---|
| `bash.go` | bash | Shell 命令执行，超时、取消、环境变量 |
| `bgjobs.go` | 后台任务 | 后台进程管理 |
| `codeindex.go` | code_index | 代码符号索引（outline/search） |
| `codeindex_treesitter.go` | — | tree-sitter 支持的符号提取 |
| `completestep.go` | complete_step | 计划步骤完成签名 |
| `confine.go` | — | 工作空间限制 |
| `delete_range.go` | delete_range | 大范围删除 |
| `delete_symbol.go` | delete_symbol | Go AST 符号删除 |
| `editfile.go` | edit_file | 精确字符串替换 |
| `gitignore.go` | — | .gitignore 模式匹配 |
| `glob.go` | glob | 文件 glob 匹配 |
| `grep.go` | grep | 正则搜索 |
| `ls.go` | ls | 目录列表 |
| `movefile.go` | move_file | 文件移动/重命名 |
| `multiedit.go` | multi_edit | 原子多编辑 |
| `notebookedit.go` | notebook_edit | Jupyter notebook 编辑 |
| `preview.go` | — | 预览 |
| `readfile.go` | read_file | 文件读取（分页、UTF-16） |
| `todo.go` | todo_write | 任务列表管理 |
| `webfetch.go` | web_fetch | URL 抓取 |
| `workspace.go` | — | 工作空间管理 |
| `writefile.go` | write_file | 文件写入 |

测试覆盖充分，每个工具都有对应的 _test.go，还有大量边界测试（CRLF、UTF-16、SSRF、取消、超时等）。

### 2.5 配置与记忆

| 包 | 职责 | 关键文件 |
|---|---|---|
| `config` | 配置加载与解析 (TOML) — provider/agent/tool/sandbox/bot/skill 配置 | `config.go`, `system_prompt*.go` |
| `memory` | 分层记忆系统 — remember/forget/recall/quickadd、存储、约定 | `memory.go`, `store.go` (19KB), `remember.go`, `recall.go`, `forget.go`, `quickadd.go`, `doc.go` (9.7KB) |
| `memorycompiler` | 记忆编译/压缩/强化 — 运行时与回归测试 | `runtime.go` (95KB), `compression.go` (55KB) |
| `outputstyle` | 输出风格/persona | `outputstyle.go` |
| `skill` | 可调用 playbook 系统 — 内置技能、索引、工具 | `skill.go` (25KB), `builtins.go` (22KB), `tools.go` (25KB), `index.go` |
| `installsource` | 技能/MCP/插件安装源 | — |

### 2.6 安全与沙箱

| 包 | 职责 | 关键文件 |
|---|---|---|
| `permission` | 工具权限 — bash 分解、只读判定、重定向分析 | `permission.go` (21KB), `bash_decompose.go`, `bash_readonly.go`, `bash_redirect.go` |
| `sandbox` | 沙箱执行 — macOS seatbelt、shell 包装 | `sandbox.go`, `seatbelt_darwin.go`, `shell.go` |
| `shellsafe` | Shell 安全检查 — 重定向安全 | `shellsafe.go`, `bash_redirect.go` |
| `shellparse` | POSIX shell 解析 (mvdan.cc/sh) | `bash.go` (10KB) |

### 2.7 前端与传输

| 包 | 职责 | 关键文件 |
|---|---|---|
| `cli` | 终端 TUI 前端 — chat、bot、ACP、分支、自动计划、box 等 | `cli.go`, `chat_tui.go`, `bot.go`, `acp.go`, `branch.go`, `autoplan.go`, `box.go` |
| `serve` | HTTP/SSE 服务器 — auth、广播、标题缓存 | `serve.go` (37KB), `auth.go` (18KB), `broadcaster.go`, `titlecache.go`, `index.html` (84KB 嵌入式 Web UI) |
| `command` | 斜杠命令系统 | `slashtool.go` |
| `bot` | 多通道 IM Bot 网关 — 飞书/Lark/微信/QQ | — |

### 2.8 插件系统

| 包 | 职责 | 关键文件 |
|---|---|---|
| `plugin` | MCP 插件主机 — stdio/http 传输、懒加载、缓存、统计、并发 | `plugin.go` (43KB), `lazy.go` (14KB), `transport_stdio.go` (16KB), `transport_http.go`, `cache.go`, `stats.go`, `prompts.go`, `resources.go` |
| `pluginpkg` | 插件包管理 — 描述、规范化 | `pluginpkg.go` (21KB), `describe.go` |

### 2.9 实用工具包

| 包 | 职责 |
|---|---|
| `store` | 会话存储 |
| `migration` | 配置/数据迁移 (18KB) |
| `netclient` | HTTP 客户端工具 |
| `nilutil` | nil 工具函数 |
| `notify` | 跨平台通知 (macOS/Linux/Windows/other) |
| `proc` | 进程管理 — 隐藏、杀死、优先级、树、shell 路径探测 (平台分文件) |
| `retrieval` | BM25 检索 |
| `sysproxy` | 系统代理设置 (Windows/other) |
| `textutil` | 文本工具 — 字素 |
| `mcpdiag` | MCP 诊断 — auth |

---

## 3. cmd/ — 入口点

三个二进制，每个都是薄的 main，委托给 internal/。

### cmd/reasonix/main.go — 主 CLI 二进制

main 极其精简，通过 **blank import** 注册 provider 和工具，实际逻辑在 internal/cli.Run() 中。internal/cli 是大型包，包含 chat TUI、bot、ACP、分支、自动计划等。

### cmd/e2ebench/ — 端到端基准测试驱动 (4 文件)

两种模式：suite（运行任务套件）和 diff（PR-diff 测试生成评分）。每个任务复制种子 workdir/ 到临时目录，运行 reasonix run，评分时才复制 verify.sh（防止 agent 读取答案）。包含变异测试 — 用零值返回替换变更函数体，记录捕获/存活。

### cmd/reasonix-plugin-example/main.go — MCP 插件示例

参考 MCP stdio 插件 — 最小 JSON-RPC 2.0 服务器，文档化完整插件契约。区分带内工具错误与协议错误，stderr 保留给日志，stdout 保留给 JSON-RPC。

---

## 4. desktop/ — Wails 桌面应用

Wails v2 桌面应用，Go 后端 + React/TypeScript 前端，通过 Wails IPC 桥接。

### 4.1 Go 后端

| 文件 | 大小 | 职责 |
|---|---|---|
| `app.go` | 231KB | 核心 Wails 应用 — 会话管理、工具审批、设置、IPC 绑定 |
| `bot_connection_app.go` | 34KB | IM Bot 连接管理 |
| `bot_runtime_app.go` | 12KB | Bot 运行时 |
| `bot_event_sink.go` | 5KB | Bot 事件接收器 |
| `crash_app.go` | 10KB | 崩溃报告 |
| `devinfo*.go` | — | 设备信息（平台分文件：darwin/linux/windows） |
| `dotenv.go` | 3KB | .env 文件处理 |

测试覆盖极其充分：app_test.go (221KB)、app_autosave_test.go、app_session_dedup_test.go、bot_connection_app_test.go 等。

### 4.2 前端 (desktop/frontend/)

**技术栈**：Vite + React + TypeScript，pnpm 管理。

| 区域 | 关键文件 | 大小 | 说明 |
|---|---|---|---|
| 入口 | `src/App.tsx` | 162KB | 根组件 |
| 桥接 | `src/lib/bridge.ts` | 176KB | Wails IPC 桥接 — Go 方法绑定、事件订阅、dev mock |
| 状态管理 | `src/lib/useController.ts` | 91KB | 前端状态机，per-tab 状态，agent 事件流处理 |
| 设置 | `src/components/SettingsPanel.tsx` | 293KB | 海量设置 UI — provider/MCP/skill/plugin/bot/权限/沙箱 |
| 工作空间 | `src/components/WorkspacePanel.tsx` | 66KB | 工作空间面板 |
| 能力面板 | `src/components/CapabilitiesPanel.tsx` | 96KB | 能力面板 |
| 项目树 | `src/components/ProjectTree.tsx` | 84KB | 项目树 |
| 记忆面板 | `src/components/MemoryPanel.tsx` | 62KB | 记忆管理 |
| 对话 | `src/components/Transcript.tsx` | 49KB | 对话流 |
| 编辑器 | `src/components/Composer.tsx` | 108KB | 输入编辑器 |

**自定义功能**：`src/custom/features/heartbeat/` — Heartbeat 面板 (47KB)、桥接、CSS、类型。

**测试**：`src/__tests__/` 包含 60+ 测试文件，覆盖 UI 交互、快捷键、拖放、主题、审批模态框等。

### 4.3 构建资产 (desktop/build/)

| 平台 | 文件 | 说明 |
|---|---|---|
| Windows | `build/windows/icon.ico`, `build/windows/installer/project.nsi` | NSIS 安装程序 |
| Linux | `build/linux/nfpm.yaml`, `build/linux/reasonix.desktop` | nfpm 包 + .desktop 文件 |
| macOS | `build/darwin/entitlements.plist`, `build/darwin/icon.icns` | 签名权限 + 图标 |

### 4.4 代码签名 (desktop/cmd/sign/)

代码签名工具 (`main.go`, 7.5KB)，用于 Windows 构建的 SignPath 签名流程。

---

## 5. workers/ — Cloudflare Workers 服务

三个独立的 Cloudflare Workers，每个是私有 npm 包，部署到各自子域名和 D1 数据库。共享身份模型：id.reasonix.io (accounts worker) 是唯一身份权威，其他 worker 通过转发调用者的 rxid cookie / Bearer token 到 accounts /me 解析身份。

### 5.1 workers/accounts/ — 身份服务 (id.reasonix.io)

**用途**：邮件/密码认证、会话、邮件验证、密码重置、公共配置文件、RFC 8628 设备登录（CLI/desktop）。

**技术栈**：Hono v4 + Zod + Cloudflare Workers 类型，pnpm 管理。

**D1 数据库** (reasonix-accounts)，两份迁移：
- 0001_initial.sql: users、sessions (token_hash = sha256(pepper:token))、email_tokens (单次使用哈希令牌)
- 0002_device_grants.sql: device_grants (PK device_code_hash, status pending|approved|denied)

**API 路由**：POST /auth/{register,verify,login,logout,forgot,reset,resend-verification}、POST /device/{start,poll,approve,deny}、GET|PATCH|DELETE /me、GET /u/:handle

**安全模式**：
- Pepper 哈希令牌 — sessions/email_tokens/device_codes 只存 sha256(pepper:token)
- PBKDF2-HMAC-SHA256, 100k 迭代 (Cloudflare Workers 硬上限)
- 枚举安全注册 — 无论邮箱是否存在都返回相同响应
- 设备流 — device_code 由设备轮询，user_code 由人类在网页输入

### 5.2 workers/crash-report/ — 崩溃仪表板 + 技能/MCP 注册表 (crash.reasonix.io)

**用途**：两个折叠的关注点 — (1) 桌面崩溃/异常/反馈/性能报告的接收与仪表板；(2) 技能/MCP 注册表 API + 审核控制台。还代理桌面发布清单。

**技术栈**：Hono + Zod，npm 管理。崩溃仪表板渲染**服务端 HTML** (inline UI kit in shell.ts)；注册表子应用是纯 JSON API。

**两个 D1 数据库**：
- DB (reasonix-crash): groups (指纹崩溃组)、reports (原始报告)、pings (日活)、metrics + metric_users (opt-in 聚合指标)、access (仪表板认证)、audit_log
- REGISTRY_DB (reasonix-registry): packages (kind skill|mcp, slug, install_count, verified)、package_versions (不可变版本历史)、stars、events (发布/安装/星标动态)

**速率限制**：RATE_LIMITER (5/60s)、PING_LIMITER (30/60s)、METRICS_LIMITER (30/60s)、WRITE_LIMITER (30/60s)

**关键文件**：src/index.ts (39KB 单体)、src/stats.ts (45KB 仪表板图表)、src/shell.ts (28KB 内联 HTML)、src/registry/ (子应用)

### 5.3 workers/forum/ — 社区论坛 (forum.reasonix.io)

**用途**：讨论论坛 API — 分类、话题、帖子、举报、反应。

**技术栈**：Hono + Zod，pnpm 管理。

**D1 schema**：members (trust 0-4, role)、categories (min_trust_to_post)、topics、posts、reactions、flags、mod_log。种子分类：announcements、help、skills、show、feedback。

**反垃圾** (src/antispam.ts)：基于信任等级而非内容扫描。每日发帖上限按信任等级递增 (5/20/60/∞)。新成员不能发链接。4 个举报自动隐藏帖子。

---

## 6. site/ — Astro 静态站点

**用途**：营销/文档/社区/账户/技能前端，部署到 GitHub Pages。全程双语（EN/中文）— 每个标签是 `<span class="l-en">...</span><span class="l-zh">...</span>` 对，由共享 localStorage 切换。

**技术栈**：Astro v6.4.8 + @astrojs/sitemap + @fontsource-variable/{jetbrains-mono,outfit}。npm 管理。

**结构**：
- `layouts/`: Base.astro (全局 head + OG/Twitter meta)、Account.astro、Community.astro
- `pages/`: index.astro (28KB 着陆页)、docs.astro (47KB 单页文档)、skills.astro (32KB 注册表浏览器)、account.astro、login/register/forgot/reset/device.astro (认证流)、community/{index,topic,new,guidelines}.astro、404.astro
- `scripts/`: auth.js (10.7KB — accounts API 客户端)、community.js (16KB — 论坛客户端)、fx.js (8KB — 光标/滚动效果)、site.js (7KB — 导航/语言切换)
- `styles/`: global.css (40KB)、community.css (16KB)
- `data/`: community.json (贡献者+星标+合并PR)、contributors.json (回退快照)

**构建管道**：prebuild 运行 fetch-community.mjs 抓取实时 GitHub 贡献者/星标/合并PR — 从不失败构建（回退到提交的快照）。

---

## 7. docs/ — 文档

全部文档文件（标注中英双语对）：

| 文件 | 用途 |
|---|---|
| SPEC.md (40KB) | **工程规范/契约** — 设计原则、包布局、Provider/Tool 接口、注册表 |
| GUIDE.md (46KB) + GUIDE.zh-CN.md (43KB) | 日常配置与使用 — 配置解析、provider、权限/沙箱、MCP 插件、斜杠命令、@ 引用、双模型协作 |
| BOT_GUIDE.md (22KB) + BOT_GUIDE.zh-CN.md (20KB) | IM Bot 集成 — 飞书/Lark/微信/QQ 连接、审批、YOLO、命令 |
| CHECKPOINTS.md (6.5KB) | 检查点与回退设计 — 基于文件快照（非 git），每用户回合一个检查点 |
| CONFIG_PATHS.md (10.6KB) + .zh-CN.md | Reasonix 主目录结构 — config.toml、.env、会话、记忆位置 |
| MIGRATING.md (8.8KB) | 从 0.x TypeScript 迁移到 1.0 Go 重写 |
| TOOL_CONTRACT.md (7.6KB) + .zh-CN.md | Provider 可见内置工具名、只读标志、schema 快照守护 |
| PLUGIN_PACKAGES.md (8.8KB) + .zh-CN.md | 插件包 |
| RELEASING.md (4.3KB) | 发布流程 |
| REASONING_LANGUAGE.md (3.5KB) + .zh-CN.md | 推理语言偏好 |
| REASONING_PROVIDERS.md (3KB) | 推理 provider |
| SESSION_MEMORY_RETRIEVAL.md (9.3KB) | 会话记忆检索 |
| SESSION_REFERENCE_ARCHITECTURE.md (14.9KB) | 会话参考架构 |
| COLLABORATION_MODES.zh-CN.md (6.6KB) | 协作模式 |
| GOAL_ENFORCEMENT.zh-CN.md (4.3KB) | 目标执行 |
| TOOL_APPROVAL_MODES.zh-CN.md (6.5KB) | 工具审批模式 |
| DESKTOP_HOOKS.zh-CN.md (11.5KB) | 桌面钩子 |
| production_checklist.md (1.9KB) + .zh-CN.md | 生产检查清单 |
| assets/ | 截图、SVG、logo、预览图 |
| superpowers/ | 超能力文档子目录 |

---

## 8. npm/ — npm 分发包装

通过 `npm i -g reasonix` 分发 Go 二进制的 npm 包包装器。

- `npm/build.mjs` (4KB) — 构建脚本，从 GitHub Releases 下载对应平台的预编译二进制
- `npm/reasonix/package.json` — npm 包定义
- `npm/reasonix/bin/reasonix.js` (709B) — bin 入口，启动原生二进制

安装命令 `npm i -g reasonix` 在 1.0.0+ 交付 Go 二进制，0.x 是旧 TS 构建。

---

## 9. tools/ — 辅助工具

### tools/write_heartbeat/

- `main.go` (1.8KB) — heartbeat 任务写入工具
- `heartbeat-tasks.json` (535B) — heartbeat 任务定义

---

## 10. benchmarks/ — 端到端基准测试

### benchmarks/context-maintenance-e2e/

- `main.go` (8.7KB) — 上下文维护端到端基准测试

### benchmarks/e2e/

任务套件，每个任务包含 task.toml (任务定义) + verify.sh (验证脚本) + workdir/ (种子工作目录)：

| 任务 | 测试内容 |
|---|---|
| compaction | 压缩 — 6 个章节文件的故事，测试长上下文压缩 |
| fix-add-bug | 修复加法 bug — calc.py |
| fizzbuzz | FizzBuzz 经典编程题 |
| palindrome | 回文检测 |
| subagent-delegation | 子代理委托 — alpha/beta/gamma.txt 数据文件 |

---

## 11. scripts/ — 构建/发布/缓存脚本

| 脚本 | 用途 |
|---|---|
| `cache-guard.sh` | CI 门控 — 强制缓存稳定性不变量。运行缓存命中守护测试 |
| `check-cache-impact.sh` | PR 正文 linter — 分类变更文件是否触及缓存敏感路径，要求 PR 包含 Cache-impact/Cache-guard 元数据 |
| `desktop-build.sh` | 桌面构建脚本 (8KB) |
| `resolve-desktop-release.sh` | 桌面发布解析 (1.3KB) |
| `backfill-issue-labels.mjs` | Issue 标签回填 (Node 脚本, 4.5KB) |

---

## 12. .github/ — CI/CD 流水线

### 工作流 (.github/workflows/)

| 工作流 | 用途 |
|---|---|
| `ci.yml` | 主 CI — Go 构建、vet、lint (golangci-lint)、测试 |
| `release.yml` | CLI 发布 — GoReleaser 构建 6 平台二进制 |
| `release-npm.yml` | npm 包发布 |
| `release-desktop.yml` | 桌面应用发布 (19KB，最复杂) — Wails 构建、签名、打包 |
| `cache-impact.yml` | 缓存影响检查 — 运行 check-cache-impact.sh |
| `e2e-bot.yml` | 端到端 Bot 测试 (6.9KB) |
| `codeql.yml` | CodeQL 安全分析 |
| `pages.yml` | GitHub Pages 部署 (site/) |
| `deploy-accounts-worker.yml` | 部署 accounts Worker |
| `deploy-crash-worker.yml` | 部署 crash-report Worker |
| `deploy-forum-worker.yml` | 部署 forum Worker |
| `issue-auto-label.yml` | Issue 自动标签 |
| `issue-version-label.yml` | Issue 版本标签 |
| `pr-auto-label.yml` | PR 自动标签 |
| `pr-version-label.yml` | PR 版本标签 |

### 其他

- `CODEOWNERS` — 代码所有者定义
- `dependabot.yml` — Dependabot 依赖更新配置
- `labeler.yml` — 自动标签规则
- `ISSUE_TEMPLATE/` — bug 报告、功能请求模板
- `pull_request_template.md` — PR 模板
- `sponsor/` — 赞助二维码 (wechat-pay.jpg)

---

## 架构依赖关系总览

```
                     ┌──────────────────────────────────────────────┐
                     │                  reasonix.toml                 │
                     │            (配置: provider/agent/tool/         │
                     │             sandbox/bot/skill/plugin)          │
                     └──────────────────────┬───────────────────────┘
                                            │
                              ┌─────────────▼─────────────┐
                              │      internal/config       │
                              │    (TOML 解析 + 默认值)     │
                              └─────────────┬─────────────┘
                                            │
                              ┌─────────────▼─────────────┐
                              │      internal/boot         │
                              │  (配置 → Controller 装配)   │
                              └─────────────┬─────────────┘
                                            │
                              ┌─────────────▼─────────────┐
                              │   internal/control         │
                              │  (传输无关会话驱动器)       │
                              │  导入 23 个内部包          │
                              └──┬──────┬──────┬───────────┘
                                 │      │      │
              ┌──────────────────┤      │      ├──────────────────┐
              │                  │      │      │                  │
    ┌─────────▼─────────┐ ┌─────▼────┐ │ ┌───▼────────────┐ ┌────▼─────────┐
    │   internal/cli     │ │ internal/│ │  internal/serve │ │  desktop/    │
    │  (终端 TUI/Bubble)  │ │  bot     │ │  (HTTP/SSE)     │ │  (Wails App) │
    └───────────────────┘ └──────────┘ └────────────────┘ └──────────────┘
                             │
              ┌──────────────▼──────────────┐
              │       internal/agent         │
              │  (推理循环/压缩/缓存守护)    │
              └──┬───────────┬───────────┬──┘
                 │           │           │
        ┌────────▼───┐ ┌────▼────┐ ┌───▼────────────┐
        │  provider   │ │  tool   │ │ memory/skill/  │
        │ (openai/   │ │/builtin │ │ planmode/evidence│
        │  anthropic)│ │ (22+工具)│ │ checkpoint/    │
        └────────────┘ └────┬────┘ │ guardian/jobs  │
                            │      └────────────────┘
                   ┌────────▼────────┐
                   │ permission/      │
                   │ sandbox/shell    │
                   │ (安全层)         │
                   └─────────────────┘


    Cloud 服务层:
    ┌──────────────┐  ┌──────────────────┐  ┌──────────────┐
    │ workers/     │  │ workers/         │  │ workers/     │
    │ accounts     │  │ crash-report     │  │ forum        │
    │ (身份权威)   │  │ (崩溃+注册表)    │  │ (社区论坛)   │
    └──────┬───────┘  └────────┬─────────┘  └──────┬───────┘
           │                   │                   │
           └───────── id.reasonix.io /me ──────────┘
                      (共享身份解析)

    分发层:
    npm/ (npm 包装器) ──► GitHub Releases (GoReleaser)
    site/ (Astro 站点) ──► GitHub Pages
```

### 关键架构不变量

1. **缓存优先**：系统提示前缀必须字节稳定，由 `scripts/cache-guard.sh` 和 `internal/agent` 中的 `TestReleaseCacheHitGuard` 测试守护
2. **传输无关**：所有行为加到 `control.Controller`，不加到前端，三个前端同时继承
3. **单一二进制**：`CGO_ENABLED=0`，唯一依赖是 TOML 解析器（桌面构建除外）
4. **init() 自注册**：Provider 和工具通过 blank import + init() 注册，main 保持精简
5. **无 SDK**：Provider 使用手写 net/http 客户端，不导入厂商 SDK

---

*报告生成时间：基于 Reasonix v1.16.0 代码库分析*
