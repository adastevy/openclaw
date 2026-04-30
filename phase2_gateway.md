# Phase 2 — Gateway 组件深度分析

## 1. 结论先行

你的理解 **方向正确，但需要补完一半**。准确描述：

> Gateway 是 OpenClaw 的 **常驻守护进程**，承担三个角色：
>
> 1. **入口路由器**：接收所有客户端（CLI、Web UI、macOS/iOS/Android、Webchat、Channels 入站消息）的 WebSocket 连接，按 RPC 方法分发到本进程内 handler。
> 2. **Agent 调度器/宿主**：把 `chat.send` / `agent` 类请求翻译成对内置 Pi Coding Agent 的调用，并把 Agent 的流式输出广播回订阅的客户端。
> 3. **设备能力代理（Node Broker）**：把 Agent 想做的本机/移动端动作（屏幕、相机、定位、Canvas、APNs 唤醒…）以 `node.invoke` 形式转发到 `role: node` 远端节点，再把结果回填给 Agent。

也就是说 Gateway **不是单向 "客户端 → Agent" 转发器**，而是 **双向 broker**：对 Agent 来说既是上游入口（请求进来），也是下游出口（去调动设备/通道）。

---

## 2. 进程拓扑

```
                       ┌──────────────────────────────────────────────────┐
   Operator clients    │                  GATEWAY  (one per host)         │   role:node clients
   (CLI / WebUI /      │                                                  │   (macOS / iOS / Android /
    macOS app /        │   ┌──────────┐   ┌────────────────┐   ┌───────┐  │    headless)
    iOS / Android /    │   │  ws/http │ → │ handleGateway  │ → │ Agent │  │
    webchat)           │   │  server  │   │ Request +      │   │ Run-  │  │
            ───WS──►   │   │ + auth   │   │ coreGateway    │ ← │ time  │  │   ◄──WS──
                       │   │ + scopes │   │ Handlers table │   │ (Pi)  │  │
                       │   └──────────┘   └───────┬────────┘   └───┬───┘  │
                       │                          │                │      │
   Channels ingress    │   ┌──────────────────┐   │                │      │
   (Telegram, WA,      │   │ channel plugins  │ ──┘    node.invoke │      │
   Slack, Discord,     │   │ (in-process)     │ ◄────── dispatch ──┘      │
   iMessage, Signal,   │   └──────────────────┘                           │
   Matrix, Webchat)    └──────────────────────────────────────────────────┘
```

不变量（来自 `@/root/cursor_workspace/openclaw/docs/concepts/architecture.md:143-147`）：

- 每个主机 **唯一一个 Gateway** 控制 Baileys/WhatsApp session。
- 第一帧必须是 `connect`，否则硬断连。
- Events **不重放**，客户端断流后需自刷新。

---

## 3. 输入侧 — 客户端如何进入 Gateway

### 3.1 握手

入口：`@/root/cursor_workspace/openclaw/src/gateway/server/ws-connection.ts:180-425`。每条新连接立即推送 `connect.challenge`：

```@/root/cursor_workspace/openclaw/src/gateway/server/ws-connection.ts:258-263
    const connectNonce = randomUUID();
    send({
      type: "event",
      event: "connect.challenge",
      payload: { nonce: connectNonce, ts: Date.now() },
    });
```

客户端必须在 `handshakeTimer` 超时前回 `connect` req，包含：

- **`role`**：`operator`（控制面）或 `node`（设备节点）
- **`scopes`**：`operator.read|write|admin|pairing`
- **`client.{id,mode,platform,version,deviceFamily,...}`**：自描述
- **`device`**：Ed25519 身份 + 对 `connectNonce` 的签名（v3 payload 还绑 `platform` + `deviceFamily`）
- **`auth`**：共享密钥 / device-token / bootstrap-token / Tailscale 头，取决于 `gateway.auth.mode`

握手成功后 Gateway 回 `hello-ok`：`features.methods`、`features.events`、`presence` 快照、`health` 快照、`policy.{maxPayload,maxBufferedBytes,tickIntervalMs}`。

`role: node` 的连接会立即注册到 `NodeRegistry`：

```@/root/cursor_workspace/openclaw/src/gateway/server/ws-connection/message-handler.ts:1325-1330
        if (role === "node") {
          const context = buildRequestContext();
          const nodeSession = context.nodeRegistry.register(nextClient, {
            remoteIp: reportedClientIp,
          });
```

之后该节点可被 `node.invoke` 按 `nodeId` 寻址。

### 3.2 后续帧只接受 `req`

握手后非 `req` 帧一律 `INVALID_REQUEST`。合法 req 直接喂入 `handleGatewayRequest`：

```@/root/cursor_workspace/openclaw/src/gateway/server/ws-connection/message-handler.ts:1504-1516
      void (async () => {
        await handleGatewayRequest({
          req,
          respond,
          client,
          isWebchatConnect,
          extraHandlers,
          context: buildRequestContext(),
        });
      })().catch((err) => {
        logGateway.error(`request handler failed: ${formatForLog(err)}`);
        respond(false, undefined, errorShape(ErrorCodes.UNAVAILABLE, formatForLog(err)));
      });
```

### 3.3 中央分发 — `handleGatewayRequest`

`@/root/cursor_workspace/openclaw/src/gateway/server-methods.ts` 是请求总线：

1. **`authorizeGatewayMethod(req.method, client)`**：基于 `client.connect.role` + `scopes` 校验调用许可。`node` 角色只能调极少数方法（如 `node.event` / `node.invoke.result`），`operator` 按 scope 决定能不能 `chat.send` / `agent` / `config.set` 等。
2. **rate limit**（按方法/连接/IP，可选）。
3. 在 `coreGatewayHandlers[req.method]` 里查 handler；找不到 → `INVALID_REQUEST`。
4. 用 `withPluginRuntimeGatewayRequestScope(...)` 包一层插件运行时上下文，再 `await handler(...)`。

`coreGatewayHandlers` 是把若干模块的 handler 表合并而成（chat、agent、send、config、channels、device pairing、cron、tasks、talk、nodes…）。

### 3.4 通道入站 (channels) 的 "影子入口"

`extensions/<channel>/`（Telegram、WhatsApp、Slack、Discord、Signal、iMessage、Matrix、Webchat）都跑在 Gateway 进程内。当用户在 Telegram 给机器人发消息时：

1. 通道插件收到外部消息，构造 `MsgContext`。
2. 调 `dispatchInboundMessage(ctx, cfg, dispatcher)`（`@/root/cursor_workspace/openclaw/src/auto-reply/dispatch.ts:50-69`）。
3. 这条路径 **不经过 WebSocket / `handleGatewayRequest`**，但终点和 `chat.send` 完全相同 —— 进入同一个 Agent runtime。

所以 Gateway 的"输入"有两类：**显式 RPC** + **通道入站事件**。它们汇入同一个 Agent 调度模型。

---

## 4. Agent 调度路径 — `chat.send` / `agent` 怎么变成模型调用

`agent` 方法的核心 handler 在 `@/root/cursor_workspace/openclaw/src/gateway/server-methods/agent.ts:321-1024`：

```@/root/cursor_workspace/openclaw/src/gateway/server-methods/agent.ts:321-325
export const agentHandlers: GatewayRequestHandlers = {
  agent: async ({ params, respond, context, client, isWebchatConnect }) => {
    const p = params;
    if (!validateAgentParams(p)) {
```

主要工作（顺序简化）：

1. **参数校验**：`validateAgentParams`（Ajv，从 `protocol/index.ts` 编译的 schema）。
2. **scope 检查**：是否 owner / 是否允许 `modelOverride` / 是否允许 `/new` `/reset`（`resolveSenderIsOwnerFromClient` 等）。
3. **解析 sessionKey + agentId**：`resolveAgentIdFromSessionKey`、`resolveExplicitAgentSessionKey`、`resolveAgentMainSessionKey`，决定要写到哪个会话目录。
4. **附件归一化**：`parseMessageWithAttachments` + `normalizeRpcAttachmentsToChatAttachments` 把内联 base64 / mediaRef 统一成 chat attachments，超大或失败走 `MediaOffloadError`。
5. **会话条目合并**：`mergeSessionEntry` + `updateSessionStore`，写入会话元数据（model、provider、deliveryContext、spawnedBy、subagentRole…）。
6. **`/new` `/reset`**：`performGatewaySessionReset` 清掉旧 turn，必要时通过 `runSessionResetFromAgent` 立即跑一次 startup prompt。
7. **构造 ingressOpts** 并把请求转交 Agent 运行时：调用栈 `agentCommandFromIngress` → `defaultRuntime` → `runAgentAttempt`（`@/root/cursor_workspace/openclaw/src/agents/command/attempt-execution.ts:232`） → `runEmbeddedPiAgent` → `@mariozechner/pi-coding-agent` 的 `SessionManager`。
8. **流式事件回放**：Agent 内部 `emitAgentEvent` 把 `assistant`、`tool`、`lifecycle`、`thinking` 等流推到 `infra/agent-events`，再由 broadcast 层以 `agent` 事件形式推给订阅了该 `runId`/`sessionKey` 的 operator 客户端。
9. **最终响应**：handler 返回 `respond(true, { runId, status, summary })`；中途也允许早 ack（`status:"accepted"`）让 UI 立即拿到 `runId`。
10. **投递（可选）**：当 `request.deliver === true`，`resolveAgentDeliveryPlan` + `deliverOutboundPayloads` 还会把 Agent 输出的最终消息回投到原通道（如 Telegram 群）。

`chat.send` 走的是基本一致的内部流程（同样进入 `dispatchInboundMessage`），区别在于它面向 chat-style UI 的事件命名空间（`chat` 事件、transcript history）。

> 这是你最关心那个问题的答案：**是的**，`chat.send` / `agent` / 通道入站消息最终都被 Gateway 翻译成对 Agent 运行时的调用；Agent 在同一进程里跑，再通过事件流回到 Gateway，再广播到客户端。

---

## 5. 反向通道 — Agent 如何让 Gateway 替它做事

Agent 的 tool 调用并非什么都自己做。三类典型动作要 **借 Gateway 出门**：

### 5.1 Node 设备命令 — `node.invoke`

`@/root/cursor_workspace/openclaw/src/gateway/server-methods/nodes.ts:860-1100`：handler 收到 `node.invoke {nodeId, command, params}` 后：

1. 在 `context.nodeRegistry` 里找该 `nodeId`。
2. 没在线 → 检查是否有 APNs/FCM 注册 → `sendApnsBackgroundWake({wakeReason:"node.invoke"})` 唤醒 → `waitForNodeReconnect` 轮询。
3. 在线后调 `context.nodeRegistry.invoke(...)`：

   ```@/root/cursor_workspace/openclaw/src/gateway/node-registry.ts:135-158
       const ok = this.sendEventToSession(node, "node.invoke.request", payload);
       ...
       return await new Promise<NodeInvokeResult>((resolve, reject) => {
         const timer = setTimeout(() => {
           this.pendingInvokes.delete(requestId);
           resolve({ ok: false, error: { code: "TIMEOUT", message: "node invoke timed out" } });
         }, timeoutMs);
         this.pendingInvokes.set(requestId, { nodeId, command, resolve, reject, timer });
       });
   ```

   节点收到 `node.invoke.request` event，执行命令（屏幕截图、定位、Canvas 渲染…），再 **以 RPC `node.invoke.result` req** 把结果发回 Gateway。

4. `handleNodeInvokeResult` 通过 `pendingInvokes.get(id)` 找到那个 Promise，resolve，于是 operator 端的 `node.invoke` res 才返回。

这就是 Gateway 作为"双向 broker"最直观的体现：**operator → Gateway → node**（事件方向是 event 推送），**node → Gateway → operator**（response 方向是 RPC res）。

### 5.2 通道出站 — `send` / `deliverOutboundPayloads`

`@/root/cursor_workspace/openclaw/src/gateway/server-methods/send.ts` 的 `send` handler 把 operator 或 Agent 想发的消息走 `deliverOutboundPayloads`：解析目标通道（Telegram/WA/...），调用通道插件 SDK，落 `outbound` 事件，最后给客户端 res `{messageId, channel, ...}`。

### 5.3 广播事件 — `createGatewayBroadcaster`

`@/root/cursor_workspace/openclaw/src/gateway/server-broadcast.ts:93-186`：

```@/root/cursor_workspace/openclaw/src/gateway/server-broadcast.ts:175-185
  const broadcast: GatewayBroadcastFn = (event, payload, opts) =>
    broadcastInternal(event, payload, opts);

  const broadcastToConnIds: GatewayBroadcastToConnIdsFn = (event, payload, connIds, opts) => {
    if (connIds.size === 0) {
      return;
    }
    broadcastInternal(event, payload, opts, connIds);
  };
```

广播按 **per-event scope guard** 过滤客户端：`agent`/`chat` 事件只发给具备 `READ_SCOPE` 的 operator；`device.pair.*` 给 pairing scope；`plugin.*` 命名空间允许插件自定义。这保证一个未授权或 node-only 客户端不会偷听别人会话。

---

## 6. 输入 / 输出全景表

| 方向   | 协议帧                                                  | 触发者                 | Gateway 行为                                                     | 终点                 |
| ------ | ------------------------------------------------------- | ---------------------- | ---------------------------------------------------------------- | -------------------- |
| **入** | `req:connect`                                           | 任何客户端             | 校验签名/auth/scope，注册 client（必要时 NodeRegistry）          | 自身                 |
| **入** | `req:chat.send` / `req:agent`                           | operator               | `handleGatewayRequest` → `agent.ts` → `runAgentAttempt`          | Pi Agent             |
| **入** | `req:send`                                              | operator / Agent       | 通道插件出站                                                     | 外部通道             |
| **入** | `req:node.invoke`                                       | operator / Agent tool  | `nodes.ts` → `nodeRegistry.invoke` → `node.invoke.request` event | Node 客户端          |
| **入** | `req:node.invoke.result`                                | node 客户端            | `handleNodeInvokeResult` 解决 pending Promise                    | 还在 await 的 caller |
| **入** | `req:node.event`                                        | node 客户端            | `server-node-events.ts` 处理 voice/exec/notification 等推送      | Agent + broadcast    |
| **入** | 通道入站消息                                            | 通道插件（in-process） | `dispatchInboundMessage`                                         | Pi Agent             |
| **出** | `res`                                                   | —                      | `handleGatewayRequest.respond` 发回原连接                        | 调用方               |
| **出** | `event:agent` / `chat` / `presence` / `tick` / `health` | broadcaster            | scope-filtered 广播                                              | 订阅 operator        |
| **出** | `event:node.invoke.request`                             | NodeRegistry           | 单播给目标 node                                                  | Node 客户端          |
| **出** | `event:device.pair.*`                                   | pairing 流程           | pairing-scope 客户端                                             | 配对 UI              |

---

## 7. 关键源码入口（速查）

- `@/root/cursor_workspace/openclaw/src/gateway/server.impl.ts` — 服务初始化、HTTP/WS 监听、插件运行时挂载、生命周期。
- `@/root/cursor_workspace/openclaw/src/gateway/server/ws-connection.ts` — 单连接生命周期、challenge、超时、close 清理。
- `@/root/cursor_workspace/openclaw/src/gateway/server/ws-connection/message-handler.ts` — 帧解析、`connect` 处理、req → `handleGatewayRequest`。
- `@/root/cursor_workspace/openclaw/src/gateway/server-methods.ts` — `authorizeGatewayMethod` + `coreGatewayHandlers` + `handleGatewayRequest`。
- `@/root/cursor_workspace/openclaw/src/gateway/server-methods/agent.ts` — `agent` / `agent.identity.get` / `agent.wait`。
- `@/root/cursor_workspace/openclaw/src/gateway/server-methods/chat.ts` — chat 事务（history、abort、inject、send 的 chat 视角）。
- `@/root/cursor_workspace/openclaw/src/gateway/server-methods/send.ts` — 出站消息投递。
- `@/root/cursor_workspace/openclaw/src/gateway/server-methods/nodes.ts` — `node.*` 全套（含 `node.invoke` / `node.invoke.result`）。
- `@/root/cursor_workspace/openclaw/src/gateway/node-registry.ts` — node 在线表 + `pendingInvokes` Promise 桥。
- `@/root/cursor_workspace/openclaw/src/gateway/server-broadcast.ts` — 事件广播 + 作用域过滤。
- `@/root/cursor_workspace/openclaw/src/gateway/server-node-events.ts` — 节点上行事件（voicewake、exec、notification…）的服务端处理。
- `@/root/cursor_workspace/openclaw/src/gateway/protocol/index.ts` — 协议 schema 与运行时校验器。
- `@/root/cursor_workspace/openclaw/src/auto-reply/dispatch.ts` — 通道入站 → Agent 的统一调度入口。
- `@/root/cursor_workspace/openclaw/src/agents/command/attempt-execution.ts` — `runAgentAttempt` / Pi embedded runner。

---

## 8. 回到你的原问题

> "Gateway 接收不同客户端，然后给到后端 Agent 执行 — 是这样的处理逻辑吗？"

**是，但不止**：

- ✅ **接收**：CLI / Web UI / macOS / iOS / Android / Webchat 通过 WS `connect` + RPC 进入；通道（Telegram 等）通过 in-process `dispatchInboundMessage` 进入。两条路径都汇聚到 Gateway 的同一个调度层。
- ✅ **派给 Agent**：`agent` / `chat.send` / 通道入站都最终调到 `runAgentAttempt` → Pi 运行时；Agent 流式输出经 `infra/agent-events` 广播回客户端。
- ➕ **同时反向调度设备**：Agent 工具调用（`node.invoke`）通过 Gateway 的 `NodeRegistry` 单播给 `role: node` 设备并 await 结果——这部分常被忽略，但和"接收客户端"对称重要。
- ➕ **同时管出站**：通道发回消息（`send` / `deliverOutboundPayloads`）、event 广播（presence/health/tick）、device pairing 流程也都由 Gateway 收口。

更准确的一句话总结：

> **Gateway = 控制面 RPC 路由 + Agent runtime 宿主 + 设备 broker + 通道枢纽**，是 OpenClaw 进程模型里 _唯一_ 长驻的协调中心。
