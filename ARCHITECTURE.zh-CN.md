# OpenClaw 系统架构总览

OpenClaw 是一个**本地优先（local-first）的个人 AI 助理控制平面**：单进程 Gateway 守护进程把"消息通道 ↔ AI Agent ↔ 工具/沙箱 ↔ 客户端 UI/移动端节点"全部缝合在一起。整体架构是一个**插件化 monorepo**——核心保持薄、可扩展能力（消息通道、模型 provider、媒体生成、记忆等）以**插件（`extensions/`）**形式注入；插件只能通过 `@openclaw/plugin-sdk` 这条公共契约接入核心。

## 一、仓库顶层结构与模块划分

下面把仓库顶层目录映射到系统功能模块。

| 顶层目录 | 角色 | 主要由什么提供服务 |
|---|---|---|
| `src/` | **核心运行时（Core）**：CLI、Gateway 守护进程、Agent 执行循环、通道核心、插件加载器、配置/安全/会话/沙箱 | 见下方"核心模块拆分" |
| `extensions/` | **官方捆绑插件（Bundled Plugins）**：所有 messaging channel、模型 provider、媒体生成、记忆引擎、QA 工具等 | 每个子目录就是一个独立 npm 包，按 `openclaw.plugin.json` + `register(api)` 注册能力 |
| `packages/` | **可发布 SDK 包** | `@openclaw/plugin-sdk`（插件契约）、`@openclaw/memory-host-sdk`（记忆引擎契约）、`@openclaw/plugin-package-contract`（包级元数据契约）|
| `ui/` | **Control UI（Web 前端）**：Vite 单页应用，连 Gateway WebSocket 当作运维/聊天面板 | `ui/src/ui/app.ts`、`app-gateway.ts`、`controllers/`、`views/` |
| `apps/` | **伴生原生应用** | `apps/macos/`（菜单栏 + Voice Wake/Canvas）、`apps/ios/`、`apps/android/`、`apps/macos-mlx-tts/`，全部以 **node** 角色经 WebSocket 接入 Gateway |
| `Swabble/` | macOS Swift 工具/库（被 macOS 应用复用） | Swift 包 |
| `docs/` | **Mintlify 文档站**（外部文档源） | Markdown |
| `scripts/` | 构建/CI/i18n/Docker/烟囱测试/发布脚本 | `*.mjs`/`*.ts`/`*.sh` |
| `qa/` | QA 场景脚本与凭证代理 | `qa/scenarios/`、`qa/convex-credential-broker/` |
| `test/` | 跨模块集成测试与共享 helpers | `test/helpers/`、`test/scripts/`、`test/mocks/` |
| `skills/` | 内置 Skills（提示/工作流模板） | `<skill>/SKILL.md` |
| `openclaw.mjs` + `src/entry.ts` + `src/index.ts` | **唯一二进制入口** `bin: openclaw` | `openclaw.mjs`、`src/entry.ts`、`src/index.ts` |
| 根级 `Dockerfile*`、`fly.*.toml`、`render.yaml`、`docker-compose.yml` | 部署目标（容器、Fly.io、Render、SSH dev box） | – |

## 二、核心运行时（`src/`）模块拆分

`src/` 内部是一个由"控制平面 + Agent 引擎 + 通道核心 + 插件加载器"组成的有向架构。下面只列**顶层模块**与各自的关键文件/子目录。

### 1. 进程入口与 CLI（控制平面入口）

- **进程引导**：`src/entry.ts` 处理 argv 规范化、profile/容器目标解析、守护进程 respawn；`src/index.ts` 是 npm 包入口（CLI 模式 / 库模式分流）。
- **CLI 命令体系**：`src/cli/`（约 376 文件）。`src/cli/run-main.ts` 是 Commander 主入口；按子命令拆分为 `gateway-cli/`、`daemon-cli/`、`nodes-cli/`、`cron-cli/`、`update-cli/`、`plugins-cli.ts`、`devices-cli.ts`、`models-cli.ts`、`channel-auth.ts`、`config-cli.ts`、`secrets-cli.ts`、`security-cli.ts`、`exec-approvals-cli.ts`、`docs-cli.ts`、`tui-cli.ts` 等。
- **TUI**：`src/tui/`（终端 UI）。
- **Daemon 管理**（launchd/systemd）：`src/daemon/`。

### 2. Gateway（系统的运行时心脏）

`src/gateway/`（约 595 文件）是 OpenClaw 的**唯一控制平面**。它对外是一个 WebSocket+HTTP 服务器（默认 `127.0.0.1:18789`），承载：

- **WS 服务器与协议**：`src/gateway/server.impl.ts`、`server-http.ts`、`server-methods/`（按 RPC 方法切分）、`server-ws-runtime.ts`、`ws-log.ts`。
- **协议契约（codegen 源）**：`src/gateway/protocol/index.ts`、`protocol/schema/` —— TypeBox schema 生成 JSON Schema 与 Swift 模型，供客户端/节点共享。
- **认证 / 授权 / 设备配对**：`auth.ts`、`auth-rate-limit.ts`、`startup-auth.ts`、`connection-auth.ts`、`device-auth.ts`、`origin-check.ts`、`role-policy.ts`、`method-scopes.ts`、`probe-auth.ts`。
- **会话管理 / 历史 / 转录文件**：`session-utils*.ts`、`session-history-state.ts`、`sessions-history-http.ts`、`sessions-resolve.ts`、`session-reset-service.ts`、`cli-session-history.*`。
- **通道生命周期与健康监控**：`server-channels.ts`、`channel-health-monitor.ts`、`channel-health-policy.ts`。
- **节点注册（mac/iOS/Android）**：`node-registry.ts`、`server-node-events.ts`、`node-catalog.ts`、`node-command-policy.ts`、`server-mobile-nodes.ts`。
- **OpenAI/MCP HTTP 兼容层（让 OpenClaw 自身扮演 LLM Server）**：`openai-http.ts`、`openresponses-http.ts`、`embeddings-http.ts`、`mcp-http.*`、`models-http.ts`、`tools-invoke-http.ts`。
- **Control UI 静态托管**：`control-ui.ts`、`control-ui-csp.ts`、`control-ui-routing.ts`、`server-control-ui-root.ts` —— 把构建好的 `ui/` SPA 挂在 Gateway 同端口。
- **Cron / 钩子 / 审批 / 凭证**：`server-cron.ts`、`hooks*.ts`、`exec-approval-*.ts`、`credentials*.ts`、`credential-planner.ts`。
- **配置热重载与重启**：`config-reload*.ts`、`server-restart-sentinel.ts`、`server-startup-*.ts`、`startup-tasks.ts`。
- **VoiceClaw 实时语音**：`src/gateway/voiceclaw-realtime/`。

> 设计：**一台主机一个 Gateway**；它是唯一持有 WhatsApp Baileys session/Canvas host/Cron 调度器的进程。`src/gateway/AGENTS.md` 强制热路径不得加载完整插件 runtime，只用插件提供的轻量 artifact。

### 3. Agent 引擎（src/agents/）

`src/agents/`（约 1439 文件）是 LLM 驱动循环、工具调用、子 agent 派生、会话编排的实现。

- **Agent 主循环**：`agent-command.ts`、`pi-embedded.ts`、`pi-embedded-runner/` —— 基于 [pi-mono](https://github.com/badlogic/pi-mono) 的内嵌 runner。
- **CLI Agent 后端（Codex / Claude / OpenCode 等）**：`cli-runner/`、`cli-runner.ts`、`cli-backends.ts`、`cli-credentials.ts`、`acp-spawn.ts`（ACP/ACPX 桥接）。
- **模型选择 / 鉴权 / 失效切换**：`model-selection*.ts`、`model-catalog.ts`、`model-fallback*.ts`、`model-auth*.ts`、`auth-profiles/`、`auth-profiles*.ts`、`api-key-rotation.ts`。
- **Provider 传输与流**：`provider-*.ts`、`anthropic-transport-stream.ts`、`openai-transport-stream.ts`、`openai-ws-*.ts`、`simple-completion-*.ts`。
- **工具系统**：`src/agents/tools/`（≈119 文件）+ `pi-tools.*`、`pi-tool-definition-adapter.ts`、`bash-tools.*`、`tool-policy*.ts`、`tool-display*.ts`、`apply-patch.ts`、`channel-tools.ts`。
- **子 agent / 多 session 编排**：`subagent-*.ts`、`spawn-requester-origin.ts`、`session-*.ts`、`compaction.ts`、`workspace*.ts`。
- **沙箱（host/docker/openshell/SSH）**：`src/agents/sandbox/`、`sandbox*.ts`。
- **Skills / Workspace bootstrap / 系统提示**：`skills*.ts`、`bootstrap-*.ts`、`system-prompt*.ts`、`workspace*.ts`、`identity*.ts`。
- **MCP 客户端**：`mcp-*.ts`、`mcp-stdio*.ts`、`mcp-transport*.ts`。
- **Agent harness 抽象**：`harness/`、`harness-runtimes.ts`。

### 4. 通道核心（`src/channels/`）

通道实现的"**核心契约层**"。具体协议（Telegram/Discord/Slack…）的实现位于 `extensions/<channel>/`，但所有通道都遵循 `src/channels/` 定义的合同：

- **类型契约**：`src/channels/plugins/types.plugin.ts`、`types.core.ts`、`types.adapters.ts` — 与 SDK 的 `src/plugin-sdk/channel-contract.ts` 互相对齐。
- **会话/对话/绑定解析**：`session*.ts`、`conversation-*.ts`、`thread-binding*.ts`、`thread-bindings-*.ts`。
- **入站去抖、典型事件、状态反应**：`inbound-debounce-policy.ts`、`status-reactions.ts`、`ack-reactions.ts`、`typing*.ts`。
- **路由 / 目标 / 提及门控 / 允许列表**：`targets.ts`、`mention-gating.ts`、`allow-from.ts`、`allowlist-match.ts`、`command-gating.ts`。
- **草稿/流式预览**：`draft-stream-*.ts`、`draft-preview-finalizer.ts`。
- **Web 通道实现**：`src/channels/web/`、根级 `src/channel-web.ts`。

`src/channels/AGENTS.md` 明确：**插件作者不得直接 import `src/channels/**`**，必须走 `openclaw/plugin-sdk/*`。

### 5. 插件加载器与注册中心（`src/plugins/`）

OpenClaw 的"manifest-first 控制平面"实现。`src/plugins/`（约 441 文件）负责：

- **发现 / 启用 / 安装**：`discovery.ts`、`enable.ts`、`install.ts`、`uninstall.ts`、`update.ts`、`marketplace.ts`、`clawhub.ts`（与 ClawHub 注册中心交互）。
- **Manifest 解析与注册**：`manifest.ts`、`manifest-registry.ts`、`bundle-manifest.ts`、`bundled-plugin-metadata.ts`、`bundled-runtime-deps.ts`。
- **运行时加载**：`loader.ts`（≈120 KB）使用 jiti 在主进程内加载原生插件；`runtime/`、`runtime.ts`、`active-runtime-registry.ts`、`captured-registration.ts`。
- **能力路由（capability registries）**：`providers.ts`、`provider-runtime.ts`、`provider-discovery.ts`、`provider-catalog.ts`、`web-fetch-providers.runtime.ts`、`web-search-providers.runtime.ts`、`document-extractors.runtime.ts`、`memory-runtime.ts`、`memory-embedding-providers.ts`、`channel-catalog-registry.ts`、`compaction-provider.ts`。
- **激活计划器**：`activation-planner.ts`、`activation-context.ts` —— 在加载完整 runtime 之前先回答"这条命令/通道需要哪些插件"。
- **钩子系统（before/after agent/model/tool/install …）**：`hooks.ts`、`hook-types.ts`、`hook-runner-global.ts`、`wired-hooks-*.test.ts`。
- **配置激活与契约校验**：`config-activation-shared.ts`、`config-state.ts`、`config-policy.ts`、`config-schema.ts`、`schema-validator.ts`、`doctor-contract-registry.ts`。
- **CLI 注册（让插件贡献 root 命令）**：`commands.ts`、`cli.ts`、`cli-registry-loader.ts`。

### 6. 公共插件 SDK（`src/plugin-sdk/` + `packages/plugin-sdk/`）

这是**核心与所有插件之间唯一允许跨越的边界**。

- 实现源在 `src/plugin-sdk/`（约 449 个文件，每个对应一个 export subpath）。
- `packages/plugin-sdk/package.json` 把它们暴露为 `@openclaw/plugin-sdk/*`；根 `package.json` `exports` 中 200+ 条目同样地映射为发布形态 `openclaw/plugin-sdk/*`，对应 `dist/plugin-sdk/*`。
- 关键契约：`plugin-entry.ts`（`OpenClawPluginDefinition`/`register(api)`）、`provider-entry.ts`、`channel-contract.ts`、`channel-entry-contract.ts`、`core.ts`（`OpenClawPluginApi`）、各种 `*-runtime.ts` 懒加载边界。

### 7. 跨切关注点（核心横向能力）

| 关注点 | 模块 |
|---|---|
| **配置（解析/校验/迁移）** | `src/config/`（≈310 文件）|
| **安全（凭证、SSRF、沙箱策略）** | `src/security/`、`src/secrets/`、`src/proxy-capture/` |
| **基础设施（错误、env、日志、warning filter、IPC bridge）** | `src/infra/`、`src/process/`、`src/logger.ts`、`src/logging/` |
| **Hooks（用户自定义钩子运行时）** | `src/hooks/` |
| **Cron 调度** | `src/cron/` |
| **会话存储与转录** | `src/sessions/`、`src/trajectory/` |
| **媒体（图像/音乐/视频/媒体理解）** | `src/media/`、`src/media-generation/`、`src/media-understanding/`、`src/image-generation/`、`src/music-generation/`、`src/video-generation/` |
| **TTS / Realtime voice / Realtime transcription** | `src/tts/`、`src/realtime-voice/`、`src/realtime-transcription/` |
| **Web 抓取/搜索 / 链接理解** | `src/web-fetch/`、`src/web-search/`、`src/link-understanding/` |
| **Memory 主机** | `src/memory-host-sdk/`（绑定 `@openclaw/memory-host-sdk`），具体引擎插件在 `extensions/memory-core`、`extensions/memory-lancedb`、`extensions/memory-wiki`、`extensions/active-memory` |
| **MCP（Model Context Protocol）服务端/客户端** | `src/mcp/`，HTTP 入口在 `src/gateway/mcp-http.*` |
| **Canvas 主机（agent 可控的 HTML/A2UI 表面）** | `src/canvas-host/`（`server.ts`、`a2ui.ts`），HTTP 路由 `/__openclaw__/canvas/`、`/__openclaw__/a2ui/` |
| **Node host（移动节点对端）** | `src/node-host/` |
| **状态/状态机/路由/任务** | `src/status/`、`src/run-state-machine.ts`、`src/routing/`、`src/tasks/`、`src/flows/` |
| **配对/QR/设备**| `src/pairing/` |
| **Wizard/Onboarding** | `src/wizard/` |
| **ACP/ACPX 协议** | `src/acp/`、`extensions/acpx/` |
| **AutoReply 引擎（解析、回复策略）** | `src/auto-reply/`（≈477 文件）|
| **Context Engine** | `src/context-engine/` |
| **Markdown 渲染/分块** | `src/markdown/` |

## 三、扩展层（`extensions/`）— 真正的功能宽度

`extensions/` 下的每个目录是一个独立 npm 包，通过 `openclaw.plugin.json` 声明 `id`/`activation`/能力，再通过 `index.ts → register(api)` 在加载时调用 `api.registerProvider/registerChannel/registerSpeechProvider/...` 把自身挂入核心注册表。按"业务能力"分组：

- **Messaging Channels（≈30 个）**：`telegram/`、`discord/`、`slack/`、`whatsapp/`、`signal/`、`imessage/`、`bluebubbles/`、`matrix/`、`feishu/`、`googlechat/`、`google-meet/`、`mattermost/`、`msteams/`、`line/`、`nostr/`、`tlon/`、`twitch/`、`irc/`、`nextcloud-talk/`、`synology-chat/`、`qqbot/`、`webhooks/`、`zalo/`、`zalouser/`、`open-prose/`、`voice-call/`（电话桥）、`qa-channel/` …
- **LLM Providers（≈40 个）**：`openai/`、`anthropic/`、`anthropic-vertex/`、`google/`、`amazon-bedrock/`、`amazon-bedrock-mantle/`、`microsoft/`、`microsoft-foundry/`、`openrouter/`、`mistral/`、`moonshot/`、`deepseek/`、`groq/`、`fireworks/`、`together/`、`huggingface/`、`xai/`、`zai/`、`qwen/`、`kimi-coding/`、`alibaba/`、`byteplus/`、`volcengine/`、`tencent/`、`xiaomi/`、`venice/`、`perplexity/`、`vercel-ai-gateway/`、`cloudflare-ai-gateway/`、`litellm/`、`ollama/`、`lmstudio/`、`vllm/`、`sglang/`、`gradium/`、`copilot-proxy/`、`github-copilot/`、`opencode/`、`opencode-go/`、`codex/`、`kilocode/`、`tokenjuice/`、`synthetic/`、`chutes/`、`stepfun/`、`arcee/`、`nvidia/`、`qianfan/`、`minimax/` …
- **Speech / Realtime / Voice**：`elevenlabs/`、`deepgram/`、`microsoft/`（部分）、`openai/`（realtime 子集）、`speech-core/`、`talk-voice/`。
- **Media generation / understanding**：`fal/`、`runway/`、`comfy/`、`vydra/`、`image-generation-core/`、`media-understanding-core/`、`video-generation-core/`、`document-extract/`、`web-readability/`。
- **Web 抓取/搜索**：`brave/`、`duckduckgo/`、`exa/`、`firecrawl/`、`searxng/`、`tavily/`、`voyage/`。
- **Memory 引擎**：`memory-core/`、`memory-lancedb/`、`memory-wiki/`、`active-memory/`。
- **Tools / 系统能力**：`browser/`（CDP 浏览器自动化）、`openshell/`（SSH 沙箱）、`acpx/`（ACP eXternal）、`device-pair/`、`bonjour/`（Gateway 发现）、`phone-control/`、`skill-workshop/`、`webhooks/`、`thread-ownership/`、`diffs/`、`diagnostics-otel/`、`llm-task/`、`lobster/`。
- **QA / 测试基础设施**：`qa-channel/`、`qa-lab/`、`qa-matrix/`、`test-support/`。

这些插件之间**互相不能直接 import**（`extensions/AGENTS.md` 强约束），必须经核心 + SDK。

## 四、前端、移动端与外部集成入口

### Control UI（Web 前端）

- 入口：`ui/index.html` → `ui/src/main.ts` → `ui/src/ui/app.ts`。
- 与后端的桥：`ui/src/ui/app-gateway.ts` + `ui/src/ui/gateway.ts`，使用 Gateway 同源 WebSocket（`/ws`）+ HTTP（OpenAI 兼容、配置、设备 pair、控制 UI auth）。
- 配置/会话/聊天/插件/Canvas/通道等视图位于 `ui/src/ui/views/`、`controllers/`、`chat/`。
- 构建产物由 Gateway 的 `control-ui.ts` 直接 serve，所以 Web UI 与 Gateway **同端口同源**，没有独立后端。

### 原生伴生应用（皆为 Gateway WebSocket "node" 角色）

- macOS（`apps/macos/`）：菜单栏控制 + Voice Wake + Canvas + WebChat。
- iOS（`apps/ios/`）：节点 + 推送 + Canvas surface。
- Android（`apps/android/`）：节点 + Connect/Chat/Voice + Camera/Screen capture。
- 它们走 `src/gateway/protocol/index.ts` 定义的同一份协议，TypeBox → JSON Schema → Swift/Kotlin 模型 codegen。

### 外部服务集成入口（**只通过插件**）

- **LLM 提供商**：`extensions/<provider>/index.ts` 调用 `api.registerProvider(...)`，鉴权与流由 SDK `provider-auth-runtime`、`provider-stream-shared` 等承载。
- **消息平台 SDK / 反向网关**：每个 channel 插件持有第三方 SDK（grammY/Baileys/discord.js/@buape/carbon/@tloncorp/api/matrix-sdk-crypto-nodejs/…），通过 SDK 的 `channel-contract.ts` 暴露给核心。
- **Memory / 嵌入 / 向量库**：`memory-host-sdk` 公共契约 + `extensions/memory-*` 实现（LanceDB、Wiki、Active）。
- **MCP 服务**：作为 server（`src/gateway/mcp-http.*` + `src/mcp/`）和 client（`src/agents/mcp-*.ts`）双栖。
- **OAuth / Subscription**：ChatGPT/Codex 在 `extensions/codex`、`extensions/openai`；ClawHub 在 `src/plugins/clawhub.ts` + `extensions/`。
- **诊断 / 观测**：`extensions/diagnostics-otel/`。

## 五、模块依赖关系（运行时调用图）

下面用一张层级依赖图表达**主要数据/控制流**：

```
                ┌─────────────────────────────────────────────┐
                │   External users & systems                  │
                │  Telegram/Slack/Discord/WhatsApp/...        │
                │  OpenAI/Anthropic/Google/...  MCP clients   │
                └──────────────┬──────────────────┬──────────-┘
                               │ inbound           │ HTTP (LLM/MCP/OpenAI-compat)
                               ▼                   ▼
                        ┌────────────────────────────────────┐
                        │   extensions/<channel>             │   <── outbound replies
                        │   extensions/<provider>            │
                        └────────────┬───────────────────────┘
                                     │ register(api) via @openclaw/plugin-sdk
                                     ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │                         src/plugins/  (loader + registries)         │
   │  manifest → discovery → activation-planner → loader.ts (jiti)       │
   │  ──> capability registries: providers / channels / web / memory ... │
   └──────────────┬───────────────────────────────┬─────────────────────--┘
                  │                               │
                  ▼                               ▼
         ┌─────────────────────┐        ┌──────────────────────────┐
         │  src/channels/      │        │  src/agents/             │
         │  conversation/      │◄──────►│  pi-embedded-runner,     │
         │  routing/           │ tools  │  cli-runner (codex/...), │
         │  thread-bindings/   │        │  model-fallback,         │
         │  inbound debounce   │        │  subagents, hooks,       │
         │  draft streaming    │        │  sandbox, skills, MCP    │
         └─────────┬───────────┘        └──────────┬──────────────-┘
                   │                               │
                   └────────────┬──────────────────┘
                                ▼
                      ┌────────────────────────────────────────────┐
                      │   src/gateway/  (the only daemon)         │
                      │   server.impl.ts + server-http.ts +       │
                      │   server-methods/, protocol/, hooks,      │
                      │   sessions, cron, control-ui, mcp-http,   │
                      │   openai-http, node-registry,             │
                      │   canvas-host (mounted on same port)      │
                      └─────────────┬─────────────────────────────-┘
                                    │ WebSocket + HTTP (127.0.0.1:18789)
              ┌─────────────────────┼─────────────────────────┐
              ▼                     ▼                         ▼
     ┌────────────────┐   ┌──────────────────┐     ┌────────────────────┐
     │ ui/  Control   │   │ openclaw CLI     │     │ apps/  macOS/iOS/  │
     │  UI (browser)  │   │ (src/cli)        │     │  Android (nodes)   │
     └────────────────┘   └──────────────────┘     └────────────────────┘
```

关键依赖规则（强制约束）：

- **核心 → 插件**：永远只读插件 manifest + 通过 capability 调用，**绝不 deep import** `extensions/*/src/**`。核心使用 `*-public-artifacts.ts`、`<id>-api.ts` 等轻量 artifact 走热路径。
- **插件 → 核心**：只能 import `openclaw/plugin-sdk/*`、自己包内 `api.ts`/`runtime-api.ts`，不可 import `src/**`、其他插件、或 `src/plugin-sdk-internal/**`。
- **CLI/UI/Apps → Gateway**：统一走 `src/gateway/protocol/` 定义的 RPC + Event 协议；首帧必须是 `connect`，握手后才允许其他方法（见 `docs/concepts/architecture.md`）。
- **`src/agents/` → `src/channels/` / `src/plugins/`**：通过依赖注入和 `*-runtime.ts` 懒加载边界，避免热路径触发完整插件 runtime（`src/agents/AGENTS.md` 性能护栏）。

## 六、典型一次入站消息的端到端调用流

1. 第三方平台（如 Telegram）→ `extensions/telegram/src/inbound.ts` 收到 webhook/long-poll。
2. 插件适配为 SDK 中的 `InboundEnvelope`，交给 `src/channels/`：去抖、典型解析、路由、提及/允许列表/命令门控、会话/线程绑定。
3. `src/auto-reply/` + `src/agents/agent-command.ts` 决定使用哪个 agent + harness（`pi-embedded-runner` 或 `cli-runner`）。
4. `src/agents/model-selection.ts` + `src/plugins/providers.runtime.ts` 解析 provider/model；`src/plugins/provider-runtime.ts` 调用 `extensions/<provider>` 的注册函数发起请求。
5. 流式 token 经 `src/plugin-sdk/provider-stream-shared.ts` → `src/agents/pi-embedded-subscribe.ts` → `src/channels/draft-stream-*.ts` → 通道插件的 `outbound`/`actions.handleAction("message")` 出站。
6. 全程事件由 `src/gateway/server-broadcast.ts` 推到所有订阅的 WS 客户端（Web UI / CLI / mac app / iOS / Android）。
7. 工具调用（bash/edit/browser/canvas/sessions_spawn/...）走 `src/agents/pi-tools.*` + `src/agents/sandbox/`，必要时进 `extensions/browser`、`extensions/openshell`、Docker。
8. 持久化由 `src/gateway/session-utils.fs.ts` 写入 `~/.openclaw/`；记忆通过 `src/memory-host-sdk` + `extensions/memory-*`。

---

## 总结要点

- **架构模式**：`Local-first daemon` + `manifest-first 插件控制平面` + `严格的 SDK 边界` + `WebSocket 单端口控制平面`。
- **核心本身刻意保持薄**：协议、Agent 循环、通道契约、插件加载、安全/会话/沙箱在 `src/`；所有"业务宽度"（通道、模型、媒体、记忆、工具）在 `extensions/`。
- **插件契约唯一入口**：`@openclaw/plugin-sdk`（源码 `src/plugin-sdk/`，发布映射在 `package.json` exports）。
- **三个主要客户端形态都共享同一份 WebSocket 协议**：Control UI（`ui/`）、CLI（`src/cli/`）、原生应用（`apps/`）。
- **单进程多重身份**：Gateway 同时扮演 (a) 通道桥（出入站消息），(b) Agent 编排器，(c) Web/MCP/OpenAI-兼容 HTTP 服务端，(d) Canvas/A2UI host，(e) 节点注册中心，(f) Cron 调度器，(g) 控制面板静态资源服务器。
- **高保真扩展点**：`api.registerProvider/registerChannel/registerSpeechProvider/registerMediaUnderstandingProvider/registerImageGenerationProvider/registerMusicGenerationProvider/registerVideoGenerationProvider/registerWebFetchProvider/registerWebSearchProvider/registerCliBackend/registerGatewayDiscoveryService/registerHook/registerService/registerCli/...`（详见 `docs/plugins/architecture.md`）。
