# Phase 2 - Gateway 客户端接入与复用分析（源码版）

## 1. 结论

针对你当前的问题（WebUI、macOS 等客户端接入）：

- 在 **控制面客户端** 维度（WebUI、macOS、CLI、自定义 App），Gateway 后端核心可视为 **100% 复用**。
- 复用前提：客户端严格按 Gateway 现有协议接入（WebSocket 握手、角色/作用域、请求帧、事件帧、鉴权与设备签名）。
- 你需要开发的是“客户端协议适配层”，而不是改 Gateway 核心执行链路。

对应源码证据：

- 统一协议与校验：`src/gateway/protocol/index.ts`
- 协议帧定义：`src/gateway/protocol/schema/frames.ts`
- 统一请求分发：`src/gateway/server-methods.ts`
- 方法与事件目录：`src/gateway/server-methods-list.ts`
- Agent 执行入口链路：`src/gateway/server-methods/agent.ts`、`src/agents/command/attempt-execution.ts`

---

## 2. 当前架构中的标准接入方式

### 2.1 标准路径 A：WebSocket 控制面接入（WebUI/macOS/CLI）

这是最标准、功能最完整的客户端接入方式。

1. 建立 WS 连接到 Gateway

- 服务端连接处理：`src/gateway/server/ws-connection.ts`
- 连接建立后服务端先发 `connect.challenge`（nonce 挑战）

2. 客户端发送 `connect` 请求（首个业务帧）

- `ConnectParams` schema：`src/gateway/protocol/schema/frames.ts`
- 包含：`minProtocol/maxProtocol`、`client`、`role`、`scopes`、`caps`、`commands`、`permissions`、`auth`、`device`

3. 服务端返回 `hello-ok`

- `HelloOk` schema：`src/gateway/protocol/schema/frames.ts`
- 返回：协议版本、能力清单（methods/events）、快照（presence/health）、策略（payload/tick）

4. 客户端按 `req/res/event` 帧进行调用与订阅

- 帧格式：`src/gateway/protocol/schema/frames.ts`
- 方法分发：`src/gateway/server-methods.ts`
- 方法目录：`src/gateway/server-methods-list.ts`

### 2.2 标准路径 B：HTTP OpenAI 兼容接入（工具/第三方前端）

适合不做 WS 协议实现、只想通过 OpenAI 兼容接口调用模型能力的客户端。

- HTTP 入口：`src/gateway/server-http.ts`
- 关键路径：
  - `/v1/chat/completions`
  - `/v1/responses`
  - `/v1/models`
  - `/v1/embeddings`
- 处理器：
  - `src/gateway/openai-http.ts`
  - `src/gateway/openresponses-http.ts`
  - `src/gateway/models-http.ts`
  - `src/gateway/embeddings-http.ts`

结论：

- 需要完整会话控制、事件订阅、节点能力编排：选 WS。
- 只需模型推理接口：可选 HTTP OpenAI 兼容层。

---

## 3. 核心功能模块梳理（复用边界）

### 3.1 协议与校验层（客户端无关）

- 协议定义：`src/gateway/protocol/schema/*.ts`
- 运行时校验器：`src/gateway/protocol/index.ts`（Ajv compile）

特点：

- 所有客户端共享同一协议定义与校验规则。
- 客户端差异只体现在 `client.id/mode/role/scopes`，不改变后端核心执行逻辑。

### 3.2 请求路由与权限层（客户端无关）

- 总分发：`src/gateway/server-methods.ts`
- 方法总表：`src/gateway/server-methods-list.ts`
- 作用域控制：`src/gateway/method-scopes.ts`

特点：

- 统一做 method 级授权、角色校验、scope 校验。
- WebUI/macOS/CLI 只是不同调用方，不是不同后端路径。

### 3.3 会话与状态层（客户端无关）

- 会话方法：`src/gateway/server-methods/sessions.ts`
- 会话 schema：`src/gateway/protocol/schema/sessions.ts`
- 会话键与归一化：`src/routing/session-key.ts`

特点：

- session 统一由 Gateway 管理，客户端只提供 key/params。
- 适合多端共享同一会话（WebUI 与 macOS 同步观察/控制）。

### 3.4 Agent 调度层（客户端无关）

- 入口方法：`agent` / `chat.send`（见 `src/gateway/server-methods/agent.ts`、`src/gateway/server-methods/chat.ts`）
- 执行尝试：`src/agents/command/attempt-execution.ts`
- 运行上下文：`src/agents/command/types.ts`（`AgentRunContext`）

特点：

- Agent 执行依赖统一参数模型，不绑定具体客户端。
- `messageChannel` 是语义标签，不是强耦合客户端分支。

### 3.5 事件广播层（客户端无关）

- 广播能力：`src/gateway/server-broadcast.ts`
- 事件目录：`src/gateway/server-methods-list.ts` 的 `GATEWAY_EVENTS`

特点：

- 统一事件总线，按连接能力和 scopes 过滤下发。
- 客户端实现差异仅在“订阅哪些 event”和“如何渲染”。

---

## 4. 不同客户端接入时的输入/输出接口规范

## 4.1 WebSocket 通用帧规范

请求帧：

```json
{ "type": "req", "id": "string", "method": "string", "params": {} }
```

响应帧：

```json
{ "type": "res", "id": "string", "ok": true, "payload": {} }
```

事件帧：

```json
{ "type": "event", "event": "string", "payload": {}, "seq": 1, "stateVersion": {} }
```

来源：`src/gateway/protocol/schema/frames.ts`

## 4.2 握手输入（connect）规范

核心字段：

- `minProtocol` / `maxProtocol`
- `client.id/version/platform/mode`
- `role`（operator 或 node）
- `scopes`（如 `operator.read`、`operator.write`）
- `auth`（token/password/deviceToken/bootstrapToken）
- `device`（id/publicKey/signature/signedAt/nonce）

来源：`src/gateway/protocol/schema/frames.ts`

## 4.3 握手输出（hello-ok）规范

核心字段：

- `protocol`
- `server.version/server.connId`
- `features.methods/events`
- `snapshot`（presence/health/stateVersion）
- `policy.maxPayload/maxBufferedBytes/tickIntervalMs`
- 可选 `auth.deviceToken`

来源：`src/gateway/protocol/schema/frames.ts`

## 4.4 业务输入接口（常用）

- `chat.send` / `chat.abort` / `chat.history`：`src/gateway/protocol/schema/logs-chat.ts`
- `agent` / `agent.wait` / `agent.identity.get`：`src/gateway/protocol/schema/agent.ts`
- `sessions.*`：`src/gateway/protocol/schema/sessions.ts`
- `send` / `message.action`：`src/gateway/protocol/schema/agent.ts`

## 4.5 业务输出接口（常用事件）

常见事件族：

- `agent`
- `chat`
- `session.message`
- `session.tool`
- `sessions.changed`
- `presence`
- `health`
- `tick`
- `shutdown`

来源：`src/gateway/server-methods-list.ts`

## 4.6 HTTP 输入/输出规范（OpenAI 兼容）

输入：标准 HTTP JSON body（OpenAI 兼容格式）

输出：

- 非流式 JSON
- 流式 SSE（`data: ...`）

来源：`src/gateway/openai-http.ts`、`src/gateway/openresponses-http.ts`

---

## 5. 新客户端二次开发需要重点关注的技术细节

### 5.1 必做事项

1. 严格实现 connect 挑战签名流程

- 必须先收 `connect.challenge`，再用 nonce 签名回传。
- 参考：`src/gateway/server/ws-connection.ts`、`src/gateway/client.ts`

2. 处理协议版本协商

- `minProtocol/maxProtocol` 必须覆盖服务端支持版本。
- 参考：`src/gateway/protocol/schema/protocol-schemas.ts`

3. 使用 idempotencyKey（有副作用的方法）

- 如 `agent`、`send`、`chat.send` 等要安全重试。
- 对应 schema 可见 `idempotencyKey` 字段。

4. 处理事件序列和断流恢复

- 事件可能不重放，客户端需在断线后主动刷新状态。
- 参考：`docs/concepts/architecture.md`

5. 按 `hello-ok.policy` 限制 payload 和缓冲

- 客户端发送/接收都要遵守服务端策略。

### 5.2 鉴权与安全

- 远程接入优先 `wss://`，并可启用 TLS 指纹校验。
- 认证模式受 Gateway 配置影响：token/password/trusted-proxy/device token。
- 参考：`src/gateway/auth.ts`、`src/gateway/client.ts`

### 5.3 角色与权限模型

- 控制面客户端通常 `role=operator`。
- 设备执行端是 `role=node`，且需声明 caps/commands/permissions。
- 方法调用受 role + scope 双重限制。
- 参考：`src/gateway/server-methods.ts`、`src/gateway/method-scopes.ts`

### 5.4 实施建议（新客户端最小可用路径）

建议按这个顺序实现：

1. `connect` + `hello-ok` 解析
2. 通用 `req/res` 调用器（带超时与重试）
3. 事件分发器（`event` 路由 + seq 监控）
4. 首批方法：`health`、`models.list`、`chat.send`、`chat.abort`
5. 会话增强：`sessions.*`、`agent.wait`
6. 高级能力：`node.*`、`tools.catalog`、`skills.*`

---

## 6. 针对你的问题的最终判断

- 对 WebUI、macOS 这类客户端：**Gateway 后端核心机制（协议处理、路由调度、状态管理、Agent 调度）可以按现状完整复用**。
- 新客户端开发重点不在改后端，而在实现一个高质量的协议客户端层：
  - 连接与安全
  - 鉴权与设备签名
  - req/res/event 生命周期管理
  - 断线恢复与状态重建

一句话总结：

> 你现在需要做的是“客户端协议工程”，不是“Gateway 核心重构”。

---

## 7. 合并补充 - Gateway 的双向 Broker 视角

为避免误解，这里补一层执行视角：Gateway 不只是“客户端请求转发给 Agent”，还负责把 Agent 的能力请求路由到节点或通道，再回流结果。

### 7.1 运行时角色

- 入口路由器：接收 WS/HTTP/通道入站，统一进入方法分发。
- Agent 宿主：`agent`、`chat.send` 等请求最终进入同一 Agent 运行链。
- 设备能力代理：`node.invoke` 下发到 `role=node` 客户端，`node.invoke.result` 回填。
- 通道枢纽：统一出站 `send` 与事件广播（`agent/chat/presence/health`）。

关键源码：

- WS 连接与挑战握手：`src/gateway/server/ws-connection.ts`
- 请求分发总线：`src/gateway/server-methods.ts`
- Agent 方法族：`src/gateway/server-methods/agent.ts`
- Chat 方法族：`src/gateway/server-methods/chat.ts`
- Node 方法族：`src/gateway/server-methods/nodes.ts`
- 广播总线：`src/gateway/server-broadcast.ts`

### 7.2 输入/输出链路矩阵（工程实施版）

| 方向 | 入口            | 协议/接口                  | Gateway 处理                      | 输出                    |
| ---- | --------------- | -------------------------- | --------------------------------- | ----------------------- |
| 输入 | WebUI/macOS/CLI | WS `req`                   | `handleGatewayRequest` 路由与鉴权 | `res` + `event`         |
| 输入 | 第三方工具      | HTTP `/v1/*`               | OpenAI/OpenResponses 兼容处理器   | JSON/SSE                |
| 输入 | 通道插件        | `dispatchInboundMessage`   | 统一进入 auto-reply/agent 链      | 通道回发/事件           |
| 输出 | Agent 流        | `agent` 事件               | 广播并按 scope 过滤               | 控制面客户端            |
| 输出 | 节点调用        | `node.invoke.request` 事件 | NodeRegistry 选路与等待结果       | `node.invoke.result`    |
| 输出 | 消息投递        | `send` / delivery          | 渠道插件出站适配                  | Telegram/Slack 等外部面 |

### 7.3 新客户端最易踩坑点（落地清单）

- 不要跳过 `connect.challenge`：签名必须绑定服务端 nonce。
- 不要硬编码能力清单：以 `hello-ok.features.methods/events` 为运行时发现源。
- 不要忽略 `hello-ok.policy`：按服务端上限控制 payload 与缓存。
- 不要把 WS 与 HTTP 混用为同一语义层：WS 负责控制面全功能，HTTP 主要是模型兼容 API。
- 不要依赖事件重放：断线后应主动刷新会话/状态（例如 `sessions.*`、`health`）。

---

## 8. 交付说明

- 主文档：`phase2_gateway_client_integration.md`
- 补充来源：`phase2_gateway.md`（关键内容已并入本文件）
