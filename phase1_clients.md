# Phase 1 — 客户端接入 Gateway 的源码级解析

> **范围**：以 OpenClaw 仓库当前代码为依据，分析 4 类一线客户端 —— **Web Control UI**（`ui/`）、**macOS app**（`apps/macos`）、**iOS app**（`apps/ios`）、**Android app**（`apps/android`）—— 在与 **Node Gateway**（`src/gateway/`）建立连接、收发 RPC 与事件、维持会话上的实现，以及二次开发新客户端时需要做的最小集合改动。
>
> **不在范围**：通道（`extensions/<channel>/`，如 Telegram/WhatsApp）—— 它们是 Gateway 内的插件，不是"客户端"，连接关系与本文不同。

---

## 0. 共同基石：协议与 Client 注册表

四类客户端都走 **同一个 WebSocket 协议**，其权威定义在：

- `@/root/cursor_workspace/openclaw/src/gateway/protocol/index.ts:1-303`
- `@/root/cursor_workspace/openclaw/src/gateway/protocol/schema/frames.ts`
- `@/root/cursor_workspace/openclaw/src/gateway/protocol/client-info.ts:3-35`
- 文档：`@/root/cursor_workspace/openclaw/docs/gateway/protocol.md:9-185`

### 0.1 协议三大帧类型

```
{ "type": "req",   "id", "method", "params" }     // 客户端 → 网关
{ "type": "res",   "id", "ok",     "payload"|"error" }  // 网关 → 客户端
{ "type": "event", "event", "payload", "seq?" }   // 网关 → 客户端（含连接前的 connect.challenge）
```

`PROTOCOL_VERSION = 3`。所有客户端的第一帧 **必须是 `connect` 请求**，且必须先等到服务器推来的 `connect.challenge` 事件，把里面的 `nonce` 一起签进 `device.signature`。

### 0.2 Client ID 注册表（强约束）

`@/root/cursor_workspace/openclaw/src/gateway/protocol/client-info.ts:3-35`

```ts
export const GATEWAY_CLIENT_IDS = {
  WEBCHAT_UI: "webchat-ui",
  CONTROL_UI: "openclaw-control-ui",
  TUI: "openclaw-tui",
  WEBCHAT: "webchat",
  CLI: "cli",
  GATEWAY_CLIENT: "gateway-client",
  MACOS_APP: "openclaw-macos",
  IOS_APP: "openclaw-ios",
  ANDROID_APP: "openclaw-android",
  NODE_HOST: "node-host",
  TEST: "test",
  FINGERPRINT: "fingerprint",
  PROBE: "openclaw-probe",
} as const;

export const GATEWAY_CLIENT_MODES = {
  WEBCHAT: "webchat",
  CLI: "cli",
  UI: "ui",
  BACKEND: "backend",
  NODE: "node",
  PROBE: "probe",
  TEST: "test",
} as const;
```

`normalizeGatewayClientId()` 在 Gateway 侧用集合校验，**未注册的 ID 会被规范化为 `undefined`**——换言之，新客户端必须先把自己的 `clientId` 加进这个表才能被网关识别为一类已知客户端。

### 0.3 角色与作用域

`@/root/cursor_workspace/openclaw/docs/gateway/protocol.md:195-240`

- `role: "operator"` — 控制平面客户端（人类操作）。常用 scope：`operator.read / operator.write / operator.admin / operator.approvals / operator.pairing / operator.talk.secrets`。
- `role: "node"` — 设备代理（Agent 调用 `node.invoke`）。`scopes: []`，但要声明 `caps`、`commands`、`permissions`。

> 一个客户端可以**同时**以两种角色跑两条独立的 WS 连接（macOS app 和 Android app 都这么做：node 连接负责暴露设备能力，operator 连接负责 chat/config UI）。

### 0.4 设备身份（Ed25519 签名）

`@/root/cursor_workspace/openclaw/src/gateway/device-auth.ts:36-54`

```ts
export function buildDeviceAuthPayloadV3(params): string {
  return [
    "v3",
    params.deviceId,
    params.clientId,
    params.clientMode,
    params.role,
    params.scopes.join(","),
    String(params.signedAtMs),
    params.token ?? "",
    params.nonce,
    normalizeDeviceMetadataForAuth(params.platform),
    normalizeDeviceMetadataForAuth(params.deviceFamily),
  ].join("|");
}
```

四类客户端都 **必须用相同字符串拼装 + Ed25519 签名**，否则握手会以 `DEVICE_AUTH_SIGNATURE_INVALID` 失败（`docs/gateway/protocol.md:580-595`）。

### 0.5 握手期望响应 `hello-ok`

`@/root/cursor_workspace/openclaw/docs/gateway/protocol.md:74-143`

```json
{
  "type": "hello-ok",
  "protocol": 3,
  "server": { "version", "connId" },
  "features": { "methods": [...], "events": [...] },
  "snapshot": { "sessionDefaults": { "mainSessionKey": "main" }, ... },
  "policy": { "maxPayload", "maxBufferedBytes", "tickIntervalMs" },
  "auth": { "deviceToken", "role", "scopes", "deviceTokens?" },
  "canvasHostUrl?": "http://gateway:18789/__openclaw__/canvas/?capability=..."
}
```

客户端必须把 `hello-ok.auth.deviceToken` **持久化到本地（按 `deviceId+role` 索引）**，下次重连优先用它，避免再走 bootstrap pairing。

---

## 1. 四类客户端的"骨架对照表"

下面是按代码实际比对得到的差异矩阵：

| 维度                    | Web Control UI (`ui/`)                                                                        | macOS App (`apps/macos`)                                                                                                                 | iOS App (`apps/ios`)                                                                                                                                                           | Android App (`apps/android`)                                                                                                        |
| ----------------------- | --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| **语言/技术栈**         | TypeScript + Vite                                                                             | Swift / SwiftUI / SwiftPM                                                                                                                | Swift / SwiftUI / Xcode + xcodegen                                                                                                                                             | Kotlin / Jetpack Compose / Gradle                                                                                                   |
| **WS 客户端类**         | `GatewayBrowserClient` `@/root/cursor_workspace/openclaw/ui/src/ui/gateway.ts:291-657`        | `GatewayChannelActor`（共享）`@/root/cursor_workspace/openclaw/apps/shared/OpenClawKit/Sources/OpenClawKit/GatewayChannel.swift:165-496` | `GatewayChannelActor`（**与 macOS 共享**）+ `GatewayConnectionController` `@/root/cursor_workspace/openclaw/apps/ios/Sources/Gateway/GatewayConnectionController.swift:22-300` | `GatewaySession` `@/root/cursor_workspace/openclaw/apps/android/app/src/main/java/ai/openclaw/app/gateway/GatewaySession.kt:83-813` |
| **底层 WS 实现**        | 浏览器原生 `WebSocket`                                                                        | `URLSessionWebSocketTask`（可注入 `WebSocketSessioning`）                                                                                | 同左                                                                                                                                                                           | OkHttp 4 `WebSocket` + `WebSocketListener`                                                                                          |
| **`clientId`**          | `openclaw-control-ui`                                                                         | `openclaw-macos`                                                                                                                         | `openclaw-ios`                                                                                                                                                                 | `openclaw-android`                                                                                                                  |
| **典型 `clientMode`**   | `webchat`                                                                                     | `ui`（operator） / `node`（mac node mode）                                                                                               | `ui` / `node`                                                                                                                                                                  | `ui` / `node`                                                                                                                       |
| **角色**                | 仅 `operator`                                                                                 | `operator` + 可选 `node`                                                                                                                 | `operator` + `node`（默认 node 优先）                                                                                                                                          | `operator` + `node`                                                                                                                 |
| **同时连接数**          | 1                                                                                             | 2（operator + node 各一）                                                                                                                | 2（同上）                                                                                                                                                                      | 2（同上）                                                                                                                           |
| **设备身份存储**        | `IndexedDB` + `crypto.subtle` `@/root/cursor_workspace/openclaw/ui/src/ui/device-identity.ts` | Keychain（`DeviceIdentityStore` 共享 lib）                                                                                               | 同左（共享）                                                                                                                                                                   | EncryptedSharedPreferences `DeviceIdentityStore.kt`                                                                                 |
| **DeviceAuth 版本**     | v3（`buildDeviceAuthPayload` 直接复用 `src/gateway/device-auth.js`）                          | v3（`GatewayDeviceAuthPayload.buildV3`）                                                                                                 | 同左                                                                                                                                                                           | v3（`DeviceAuthPayload.buildV3` Kotlin 重写）                                                                                       |
| **TLS 证书 pinning**    | 浏览器自身 PKI；不支持指纹 pin                                                                | `GatewayTLSPinningSession`（手写 SecTrust）                                                                                              | 同左                                                                                                                                                                           | `GatewayTls.kt` + 自建 `X509TrustManager`                                                                                           |
| **Bonjour / mDNS 发现** | ❌ 无                                                                                         | ✅ `OpenClawDiscovery/` SwiftPM 子模块                                                                                                   | ✅ `GatewayDiscoveryModel` (NWBrowser)                                                                                                                                         | ✅ `GatewayDiscovery.kt`（Android NSD）                                                                                             |
| **节点 invoke 派发**    | ❌ 不接收（不是 node）                                                                        | `MacNodeRuntime` + `NodeMode/`                                                                                                           | `RootCanvas` + `Camera/Screen/...` 各 Handler                                                                                                                                  | `InvokeDispatcher.kt` 路由到 `Camera/Calendar/Photos/Sms/Motion/...Handler`                                                         |
| **重连策略**            | 指数退避 800ms→15s `gateway.ts:358-369`                                                       | watchdog 30s + 指数退避 + keepalive ping 15s `GatewayChannel.swift:259-369`                                                              | 同左（共享）                                                                                                                                                                   | `runLoop()` + 退避 `350·1.7^n` ≤ 8s `GatewaySession.kt:759-792`                                                                     |
| **APNs/FCM Push 接入**  | ❌                                                                                            | macOS app 直接使用 launchd 常驻，无 push                                                                                                 | ✅ APNs（直推或经 push relay）`apps/ios/Sources/Push/`                                                                                                                         | ✅ FCM（`apps/android/.../push/`）                                                                                                  |
| **进程角色**            | 浏览器中运行                                                                                  | 菜单栏常驻 + 托管 launchd Gateway                                                                                                        | 前台 app + ActivityWidget/Watch                                                                                                                                                | 前台 Service + 持久通知                                                                                                             |

---

## 2. 共同的 7 步握手流程（所有客户端都做这件事）

下面这套流程在四份代码里**完全一致**，只是宿主 API 不同。

```
┌──────────────────────────────────────────────────────────────────┐
│ Step 1. 解析 Gateway 端点（host/port/scheme）                       │
│   - WebUI: window.location 推导 + 显式 token/password              │
│   - macOS: GatewayEndpointStore（local/remote 两套）                │
│   - iOS:   GatewayServiceResolver + Bonjour                       │
│   - Android: GatewayEndpoint + NSD/Manual                         │
├──────────────────────────────────────────────────────────────────┤
│ Step 2. 加载/生成 Ed25519 设备身份                                   │
│   - publicKey / privateKey / deviceId(=publicKey 指纹)             │
├──────────────────────────────────────────────────────────────────┤
│ Step 3. 打开 WebSocket(ws://|wss://host:port)                      │
├──────────────────────────────────────────────────────────────────┤
│ Step 4. 等待 server 推来 event "connect.challenge"                  │
│   payload: { nonce, ts }                                          │
├──────────────────────────────────────────────────────────────────┤
│ Step 5. 用 buildDeviceAuthPayloadV3 拼字符串 + Ed25519 签名 nonce    │
├──────────────────────────────────────────────────────────────────┤
│ Step 6. 发送 req method="connect" params={                         │
│   minProtocol/maxProtocol = 3,                                    │
│   client = { id, mode, version, platform, deviceFamily, ... },    │
│   role, scopes, caps, commands, permissions,                      │
│   auth = { token | deviceToken | bootstrapToken | password },     │
│   device = { id, publicKey, signature, signedAt, nonce },         │
│   userAgent, locale                                               │
│ }                                                                 │
├──────────────────────────────────────────────────────────────────┤
│ Step 7. 收到 res ok + payload(hello-ok)                            │
│   - 持久化 hello-ok.auth.deviceToken（按 deviceId+role）            │
│   - 缓存 hello-ok.snapshot / canvasHostUrl / policy                │
│   - 之后用 res/event 多路复用所有 RPC 和推送                          │
└──────────────────────────────────────────────────────────────────┘
```

### 三份代码的实证对照

**WebUI** — `@/root/cursor_workspace/openclaw/ui/src/ui/gateway.ts:513-545`

```ts
private async sendConnect() {
  const plan = await this.buildConnectPlan();
  void this.request<GatewayHelloOk>("connect", this.buildConnectParams(plan))
    .then((hello) => this.handleConnectHello(hello, plan))
    .catch((err) => this.handleConnectFailure(err, plan));
}

private handleMessage(raw: string) {
  // ...
  if (frame.type === "event") {
    if (evt.event === "connect.challenge") {
      const nonce = (evt.payload as any).nonce;
      this.connectNonce = nonce;
      void this.sendConnect();
      return;
    }
  }
}
```

**macOS / iOS（共享 `OpenClawKit`）** — `@/root/cursor_workspace/openclaw/apps/shared/OpenClawKit/Sources/OpenClawKit/GatewayChannel.swift:371-471`

```swift
private func sendConnect() async throws {
  let options = self.connectOptions ?? GatewayConnectOptions(
    role: "operator",
    scopes: defaultOperatorConnectScopes,
    ...
    clientId: "openclaw-macos",
    clientMode: "ui",
    ...)
  let connectNonce = try await self.waitForConnectChallenge()
  let payload = GatewayDeviceAuthPayload.buildV3(...)
  let device  = GatewayDeviceAuthPayload.signedDeviceDictionary(payload, identity, ...)
  let frame = RequestFrame(type: "req", id: reqId, method: "connect",
                           params: ProtoAnyCodable(params))
  try await self.task?.send(.data(try self.encoder.encode(frame)))
  let response = try await self.waitForConnectResponse(reqId: reqId)
  try await self.handleConnectResponse(response, identity: identity, role: role)
}
```

**Android** — `@/root/cursor_workspace/openclaw/apps/android/app/src/main/java/ai/openclaw/app/gateway/GatewaySession.kt:384-431`

```kotlin
private suspend fun sendConnect(connectNonce: String) {
  val identity = identityStore.loadOrCreate()
  val storedToken = deviceAuthStore.loadToken(identity.deviceId, options.role)?.trim()
  val selectedAuth = selectConnectAuth(...)
  val payload = buildConnectParams(identity, connectNonce, selectedAuth)
  val res = request("connect", payload, timeoutMs = 12_000L)
  if (!res.ok) throw GatewayConnectFailure(res.error ?: ...)
  handleConnectSuccess(res, identity.deviceId, selectedAuth.authSource)
}
```

可以看出：**握手的字段、字段顺序、auth 优先级、签名 payload 结构**完全一致。这就是为什么 macOS 和 iOS 直接共享同一份 `GatewayChannelActor`（因为没有平台差异），而 Android 必须在 Kotlin 里把同一套逻辑重写一份。

---

## 3. 重要异同的逐项展开

### 3.1 相同：连接前 7 项契约

四份代码必须保持 byte-level 一致的部分：

1. **协议号** `minProtocol = maxProtocol = 3`。
2. **Client info 字段集**：`id`, `version`, `platform`, `mode`, `instanceId?`, `displayName?`, `deviceFamily?`, `modelIdentifier?`。
3. **设备身份**：Ed25519 keypair；`deviceId = SHA-256(publicKey) base64url 截断` 的指纹（具体在 `src/gateway/protocol/schema/primitives.ts`）。
4. **签名 payload v3 的字段顺序**（见 §0.4）。
5. **Auth 选择优先级**（`selectConnectAuth`）：
   `explicit shared token` > `explicit deviceToken (mismatch retry)` > `stored deviceToken (按 deviceId+role)` > `bootstrapToken` > `password` > 无 auth。
6. **`AUTH_TOKEN_MISMATCH` 一次性重试**：在受信任端点（loopback / 已 pin TLS）允许用 stored device token 试一次；失败就停止重连等待人工介入（`isNonRecoverableAuthError`）。
7. **持久化 `hello-ok.auth.deviceToken`**：按 `(deviceId, role)` 索引，bootstrap handoff token 只在 `wss` 或 loopback 上才允许保存（`shouldPersistBootstrapHandoffTokens`）。

### 3.2 相同：事件解析与 RPC 复用

四份代码都用 **`Map<requestId, Pending>`** 管理未完成 RPC，事件分发都遵循：

- `frame.type == "res"` → 找 pending 完成
- `frame.type == "event"` 且 `event == "connect.challenge"` → 触发握手
- `event == "node.invoke.request"` → 仅 node 角色处理（操作者忽略）
- 其它 event → 透传给上层订阅器

WebUI 的实现见 `gateway.ts:526-586`，Swift 的见 `GatewayChannel.swift` 的 `listen()` 循环，Android 的见 `GatewaySession.kt:619-666`。

### 3.3 相同：scope 过滤与 deviceToken handoff

四份代码都有相同的 **bootstrap 操作者 scope 白名单**：

```
operator.approvals / operator.read / operator.talk.secrets / operator.write
```

(`GatewayChannel.swift:557-573`、`GatewaySession.kt:439-454`、Web 端在服务端兜底校验)。这是 Gateway 协议规则的直接镜像（`docs/gateway/protocol.md:135-150`），客户端不能自行扩大。

### 3.4 异：客户端构建产物

| 客户端  | 构建命令                                  | 产物形态                                 | 启动入口                                                                                            |
| ------- | ----------------------------------------- | ---------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Web UI  | `pnpm ui:build`                           | 静态资源被 Gateway HTTP 服务器托管在 `/` | 浏览器加载 `@/root/cursor_workspace/openclaw/ui/index.html` → `ui/src/main.ts` → `ui/src/ui/app.ts` |
| macOS   | `scripts/restart-mac.sh` 或 `swift build` | `dist/OpenClaw.app`（菜单栏 app）        | `apps/macos/Sources/OpenClaw/AppState.swift` + `MenuBar.swift`                                      |
| iOS     | `pnpm ios:open` → Xcode                   | `.ipa`（TestFlight）                     | `apps/ios/Sources/OpenClawApp.swift`                                                                |
| Android | `pnpm android:bundle:release`             | `.aab`（Play / ThirdParty 双 flavor）    | `apps/android/app/src/main/java/ai/openclaw/app/MainActivity.kt`                                    |

### 3.5 异：发现机制

- **WebUI** 没有发现机制——浏览器 URL 已经直接给了 host:port。
- **macOS/iOS** 用 Apple `Network.framework` 的 NWBrowser 浏览 `_openclaw-gw._tcp` Bonjour 服务（`apps/macos/Sources/OpenClawDiscovery/` + `apps/ios/Sources/Gateway/GatewayDiscoveryModel.swift`），并且支持 wide-area DNS-SD（Tailscale 内网）。
- **Android** 用系统 NSD（`apps/android/.../gateway/GatewayDiscovery.kt`），同时在 API 33+ 要 `NEARBY_WIFI_DEVICES`，老版本要 `ACCESS_FINE_LOCATION`（`docs/platforms/android.md:189-194`）。

### 3.6 异：节点能力面（仅 node 角色）

WebUI 没有节点角色。其他三家都通过 `caps + commands + permissions` 在握手时声明能力，再在运行时分发 `node.invoke.request`：

**macOS** `@/root/cursor_workspace/openclaw/apps/macos/Sources/OpenClaw/NodeMode/MacNodeModeCoordinator.swift:62-107`

```swift
let connectOptions = GatewayConnectOptions(
  role: "node",
  scopes: [],
  caps: caps,                   // ["canvas","screen","browser?","camera?","location?"]
  commands: commands,           // ["canvas.present","camera.snap","system.run",...]
  permissions: permissions,     // {"screen.record": true/false, ...}
  clientId: "openclaw-macos",
  clientMode: "node",
  clientDisplayName: InstanceIdentity.displayName)
```

**Android** `@/root/cursor_workspace/openclaw/apps/android/app/src/main/java/ai/openclaw/app/node/ConnectionManager.kt:145-167`

```kotlin
fun buildNodeConnectOptions(): GatewayConnectOptions = GatewayConnectOptions(
  role = "node",
  scopes = emptyList(),
  caps = buildCapabilities(),     // 来自 InvokeCommandRegistry + runtimeFlags
  commands = buildInvokeCommands(),
  permissions = emptyMap(),
  client = buildClientInfo("openclaw-android", "node"),
  userAgent = buildUserAgent(),
)
```

派发器：`@/root/cursor_workspace/openclaw/apps/android/app/src/main/java/ai/openclaw/app/node/InvokeDispatcher.kt:85-100`：

```kotlin
suspend fun handleInvoke(command: String, paramsJson: String?) = when (command) {
  OpenClawCanvasCommand.Present.rawValue   -> canvasController.present(...)
  OpenClawCameraCommand.Snap.rawValue      -> cameraHandler.handleSnap(paramsJson)
  OpenClawSmsCommand.Send.rawValue         -> smsHandler.handleSmsSend(paramsJson)
  // ...总计 30+ 命令族
}
```

**iOS** 走 SwiftUI 视图层 + 各 Service（`apps/ios/Sources/{Camera,Screen,Location,Voice,...}/`），由 `RootCanvas.swift` 持有 WKWebView 渲染 `canvas.*`。

### 3.7 异：双会话（operator + node）协同

只有 macOS / iOS / Android 同时跑两条 WS：

- **node** 连接：暴露设备能力面，绝对不被 chat/agent 流量干扰。
- **operator** 连接：用户在 app 内的"聊天/设置"界面用，订阅 `chat`/`session.message`/`presence` 等事件。

iOS 把这件事做得最显式 —— `apps/ios/Sources/Model/NodeAppModel.swift:1995-2150` 同时维护 `nodeGatewayTask` 和 operator session 任务，独立断线重连。

### 3.8 异：Reconnect 与 keepalive

| 客户端    | 机制                                                                                                                                                                                  |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| WebUI     | `scheduleReconnect`：800ms 起，`*1.7` 上限 15s `@/root/cursor_workspace/openclaw/ui/src/ui/gateway.ts:358-369`；不发应用层 ping，靠浏览器 WS 自身                                     |
| macOS/iOS | `watchdogLoop` 30s 巡检 + `keepaliveLoop` 每 15s 发 `task.sendPing()` `@/root/cursor_workspace/openclaw/apps/shared/OpenClawKit/Sources/OpenClawKit/GatewayChannel.swift:267-368`     |
| Android   | `runLoop` + `okhttp.pingInterval(30s)`；退避 `min(8s, 350·1.7^n)` `@/root/cursor_workspace/openclaw/apps/android/app/src/main/java/ai/openclaw/app/gateway/GatewaySession.kt:759-792` |

---

## 4. 各客户端 RPC 方法面对照（典型子集）

四个客户端都直接读写同一组 RPC 方法。以下是各自实际**调用过**的方法（按文件 grep 结果汇总）：

| 方法族                                                      | WebUI           | macOS           | iOS        | Android    |
| ----------------------------------------------------------- | --------------- | --------------- | ---------- | ---------- |
| `connect` / `health` / `status`                             | ✅              | ✅              | ✅         | ✅         |
| `chat.history` / `chat.send` / `chat.abort` / `chat.inject` | ✅              | ✅              | ✅         | ✅         |
| `sessions.list/preview/create/delete/usage`                 | ✅              | ✅              | ✅         | ✅         |
| `config.get/set/patch/apply/schema`                         | ✅              | ✅              | ❌(只 get) | ❌(只 get) |
| `wizard.start/next/cancel/status`                           | ✅              | ✅ (CLI)        | ✅         | ❌         |
| `talk.config/mode/speak`                                    | ✅              | ✅              | ✅         | ✅         |
| `models.list / channels.status`                             | ✅              | ✅              | ✅         | ✅         |
| `cron.*`                                                    | ✅              | ✅              | ❌         | ❌         |
| `voicewake.get/set`                                         | ✅              | ✅              | ✅         | ✅         |
| `device.pair.list/approve/reject`                           | ✅              | ✅              | ❌         | ❌         |
| `node.pair.approve/reject`                                  | ✅              | ✅              | ❌         | ❌         |
| `node.invoke / node.invoke.result`                          | ❌（不是 node） | ✅（node mode） | ✅         | ✅         |
| `node.event` (上报)                                         | ❌              | ✅              | ✅         | ✅         |
| `exec.approval.*`                                           | ✅              | ✅              | ✅         | ❌         |

**结论**：

- WebUI 是"全量管理面"——几乎所有 RPC 它都用（毕竟一个浏览器要装下整套配置/聊天/审批）。
- macOS app 紧跟 WebUI，但额外承担"node mode"那一组。
- iOS / Android 是"用户日常 app"——chat + node 居多，配置面相对窄。

---

## 5. 二次开发：从零接入新客户端要做什么

下面给出"接入一个全新客户端"（无论平台是 Windows、TV、Watch、IoT、还是另一个 Web app）的最小步骤清单，按"必做"和"按需"分。

### 5.1 必做 7 步

1. **登记 Client ID**
   编辑 `@/root/cursor_workspace/openclaw/src/gateway/protocol/client-info.ts`，向 `GATEWAY_CLIENT_IDS` 添加新条目（如 `WINDOWS_APP: "openclaw-windows"`），同时更新 `GATEWAY_CLIENT_NAMES` 类型。
   - **测试同步更新**：`src/gateway/protocol/index.test.ts` 与 `src/gateway/protocol/client-info` 相关 unit。

2. **决定角色**
   - **如果做"管理面/聊天 UI"** → `role: "operator"`，按需声明 scopes（参考 WebUI 的 `CONTROL_UI_OPERATOR_SCOPES`）。
   - **如果做"设备代理"** → `role: "node"`，scopes 为空，但必须正确填 `caps` / `commands` / `permissions`，否则 Agent 无法把任务路由到你。

3. **生成并持久化 Ed25519 设备身份**
   - 平台原生 keypair API（iOS/macOS Keychain、Android Keystore、浏览器 `crypto.subtle`、Windows DPAPI 等）。
   - `deviceId` 计算方式必须与 Gateway 期望一致，参考 `@/root/cursor_workspace/openclaw/ui/src/ui/device-identity.ts` 或 `apps/shared/OpenClawKit/Sources/OpenClawKit/DeviceIdentityStore.swift`。

4. **实现握手协议**
   严格按 §2 的 7 步走。可对照其中一份现成实现做"功能镜像"：
   - JS / TS：抄 `@/root/cursor_workspace/openclaw/ui/src/ui/gateway.ts:291-657`。
   - Swift：直接 `import OpenClawKit` 复用 `GatewayChannelActor`（已经包好了）。
   - Kotlin：抄 `@/root/cursor_workspace/openclaw/apps/android/app/src/main/java/ai/openclaw/app/gateway/GatewaySession.kt`。
   - 其它语言：实现 v3 签名（见 `@/root/cursor_workspace/openclaw/src/gateway/device-auth.ts`）+ JSON over WS 即可。

5. **持久化 deviceToken**
   `(deviceId, role) → token` 是固定主键。参考 Android 的 `DeviceAuthTokenStore`（EncryptedSharedPreferences）或 macOS 的 `DeviceAuthStore`（Keychain）。**不要把 token 存进可被其他 app 读到的明文**。

6. **RPC + Event 多路复用**
   - 维护 `Map<reqId, Promise>`。
   - 实现 `request(method, params, timeoutMs)`。
   - 订阅 `event` 帧并做按事件名分发。

7. **重连策略**
   指数退避起步，遇到 `isNonRecoverableAuthError`（`PAIRING_REQUIRED`、`AUTH_TOKEN_MISSING`、`AUTH_PASSWORD_MISMATCH`、`AUTH_RATE_LIMITED` 等）就**停止自动重连，等待人工介入**。复制 WebUI 的 `isNonRecoverableAuthError` 名单即可。

### 5.2 视客户端类型选做

#### A. 做"操作面板/聊天 UI"（operator）

- 实现以下 RPC 调用：
  - `config.get/set/schema` → 配置编辑
  - `models.list` → 模型选择
  - `channels.status` → 通道状态
  - `chat.history/send/abort` 或更现代的 `sessions.preview/send/abort`
  - 订阅事件：`chat`、`session.message`、`session.tool`、`presence`、`tick`、`heartbeat`
- 处理服务器推来的 `connect.challenge` 之外，建议监听：
  - `health`：网关健康
  - `shutdown`：网关关停
  - `device.pair.requested`：批准其他设备配对（如果你有 `operator.pairing` 权限）

#### B. 做"设备节点"（node）

- 决定要暴露的 `caps`，从已有命名空间中选择：
  ```
  canvas | screen | camera | browser | location | voice |
  device | notifications | photos | contacts | calendar | callLog | sms | motion
  ```
- 在 `commands` 里声明你**真正实现了**的命令（不要谎报）。命令名都列在 `apps/android/app/src/main/java/ai/openclaw/app/node/InvokeCommandRegistry.kt` 这套常量中。
- 把节点端在 `event == "node.invoke.request"` 路径里实现一个分发器，处理完用 `request("node.invoke.result", { id, nodeId, ok, payload?, error? })` 回复。
- 上报状态（电量/位置/活动）调 `request("node.event", { event, payloadJSON })`。
- 处理审批：node 上的敏感命令（`system.run`、`sms.send` 等）会走 `exec.approval.*` 流，操作者审批后才会发回 `node.invoke.request`。

#### C. 做"既要 UI 又要节点"的复合 app

- 跑 **两条独立 WS**，各自有自己的 `GatewayChannelActor`/`GatewaySession`。
- 两个会话**共享同一个 deviceId / publicKey**，但分别按 `(deviceId, "operator")` 和 `(deviceId, "node")` 存自己的 deviceToken。
- 注意 pairing 一次性要审批两次（operator + node），UX 上可以预先告知用户。

### 5.3 跨平台都要做的"边角料"

- **Bonjour/mDNS 发现**（可选但强烈建议）：浏览 `_openclaw-gw._tcp` 服务类型；TXT 记录里有 `tlsFingerprintSha256`、`canvasPort` 等线索。
- **TLS 证书 pinning**（远程网关必做）：除 loopback 外，连 wss 时强烈建议 pin 指纹（参考 `apps/android/.../GatewayTls.kt`）。
- **`connect.challenge` timeout**：若 6s（macOS）/ 2s（Android）内没拿到 nonce，直接 abort 当前连接，避免握手 hang。
- **协议失配**：服务器返回 `MIN_PROTOCOL_TOO_HIGH` 时强制提示用户升级客户端，因为 Gateway 不会回退到 v<3。
- **Pairing 流程**：第一次连接通常会 `error.code == "PAIRING_REQUIRED"`；客户端需要 UX 引导用户在 Gateway 端 `openclaw devices approve <requestId>`。
- **国际化与 UA**：`connect.params.userAgent` + `locale` 在服务器侧用于审计与一些 i18n 路由，建议照实填。

### 5.4 仓库内需要碰到的文件清单（PR 级别）

| 改动点                                                        | 文件                                                                                         | 必要性                        |
| ------------------------------------------------------------- | -------------------------------------------------------------------------------------------- | ----------------------------- |
| Client ID 注册                                                | `@/root/cursor_workspace/openclaw/src/gateway/protocol/client-info.ts`                       | 必须                          |
| Client ID 单元测试                                            | 同目录 `*.test.ts`                                                                           | 建议                          |
| Schema 不需要改                                               | `src/gateway/protocol/schema/frames.ts`（除非新增字段）                                      | 仅当协议变更时                |
| 平台元数据签名（如果要新增 `platform` / `deviceFamily` 取值） | `src/gateway/device-metadata-normalization.ts`                                               | 仅当新平台                    |
| 文档登记                                                      | `@/root/cursor_workspace/openclaw/docs/platforms/index.md` + 新增 `docs/platforms/<your>.md` | 必须（如果对外公开）          |
| CODEOWNERS / labeler                                          | `.github/CODEOWNERS`、`.github/labeler.yml`                                                  | 必须（按根 `AGENTS.md` 要求） |
| 新客户端代码                                                  | `apps/<your>/` 新目录                                                                        | 必须                          |
| 共享 lib 复用                                                 | 如果是 Apple 平台，直接 `import OpenClawKit`（`apps/shared/OpenClawKit`）                    | 强烈建议                      |

---

## 6. 调试与验证手段

### 6.1 实时观察握手

在 Gateway 启动时加 `--verbose`，会打出每个 connect 的 `client.id / mode / role / scopes / authSource`：

```bash
pnpm openclaw gateway --port 18789 --verbose
```

### 6.2 手工握手（最快验证）

```bash
# 0. 用 macOS 自带的调试 CLI 单独跑握手，不需要启 macOS app
cd apps/macos
swift run openclaw-mac connect --json --client-id "openclaw-macos" \
  --client-mode ui --role operator --scopes operator.read,operator.write
```

参考 `@/root/cursor_workspace/openclaw/apps/macos/Sources/OpenClawMacCLI/ConnectCommand.swift:14-115`。

### 6.3 设备审批与 pairing 状态

```bash
openclaw devices list                    # 看待审批 / 已配对
openclaw devices approve <requestId>     # 批准
openclaw devices reject  <requestId>
openclaw nodes status                    # node 角色配对状态
openclaw gateway call node.list --params "{}"
```

### 6.4 错误码诊断

`hello-ok` 失败时，`error.details.code` 是分诊关键：

- `PAIRING_REQUIRED`：常态，让用户走 `openclaw devices approve`。
- `DEVICE_AUTH_NONCE_MISMATCH`：说明你的客户端复用了旧 nonce。
- `DEVICE_AUTH_SIGNATURE_INVALID`：v3 payload 字段顺序/编码错。
- `AUTH_TOKEN_MISMATCH` + `recommendedNextStep == "retry_with_device_token"`：在 trusted endpoint 上自动重试一次（按 §3.1.6）。
- `MIN_PROTOCOL_TOO_HIGH` / `MAX_PROTOCOL_TOO_LOW`：升级客户端。

完整表见 `@/root/cursor_workspace/openclaw/docs/gateway/protocol.md:580-595`。

### 6.5 协议契约测试

`src/gateway/protocol/talk-config.contract.test.ts`、`src/gateway/protocol/index.test.ts`、`test/extension-import-boundaries.test.ts` 等会兜底校验客户端不会偷偷扩大协议。新增 client 时跑一次：

```bash
pnpm test src/gateway/protocol
pnpm tsgo                                # 类型检查
pnpm check                               # full sweep before push
```

---

## 7. 风险与陷阱备忘录

1. **同步 / 异步混合签名**：`buildDeviceAuthPayloadV3` 必须按 v3 字段顺序（含 `platform`、`deviceFamily`）；漏字段在某些 Gateway 版本能过、新版本会拒。**不要省字段**。
2. **多次 `connect`**：必须保证每个 WS 连接只发一次 `connect`，否则会被 Gateway 关闭。Web UI 的 `connectSent` flag 和 macOS 的 `isConnecting` 都是为此存在。
3. **静态 + 动态导入混用**：尤其是 Web/Node 平台，遵守仓库 root `AGENTS.md` 的 `[INEFFECTIVE_DYNAMIC_IMPORT]` 规则，不要让同一 heavy 模块同时被 import 和 dynamic import。
4. **Bootstrap handoff token 的范围**：仅在 `wss` 或 loopback 端点持久化，否则中间人风险。本仓 macOS/iOS/Android 三家都已照做（§3.3）。
5. **`role=node` 的 scope 不能要求 `operator.*`**：否则 Gateway 直接拒绝 connect。
6. **`AUTH_TOKEN_MISMATCH` 重试只能一次**：四份代码都用 `deviceTokenRetryBudgetUsed` 标记，必须保留这个语义。
7. **TLS pinning 与 Bonjour TXT 不可信**：TXT 给的 fingerprint 仅作发现提示，正式 pin 必须从用户授权过的来源取得（参考 `apps/android/.../ConnectionManager.kt:57-87`）。
8. **不要用 `gateway-client` 这个 ID 当生产值**：它是给 SDK 使用者的兜底名，未来可能被收紧。请加自己的具名 ID。

---

## 附录 A — 关键文件速查

| 主题                        | 文件                                                                                                        |
| --------------------------- | ----------------------------------------------------------------------------------------------------------- |
| 协议主入口                  | `@/root/cursor_workspace/openclaw/src/gateway/protocol/index.ts`                                            |
| Client ID 注册              | `@/root/cursor_workspace/openclaw/src/gateway/protocol/client-info.ts`                                      |
| Frame schema                | `@/root/cursor_workspace/openclaw/src/gateway/protocol/schema/frames.ts`                                    |
| 设备身份签名                | `@/root/cursor_workspace/openclaw/src/gateway/device-auth.ts`                                               |
| 协议文档                    | `@/root/cursor_workspace/openclaw/docs/gateway/protocol.md`                                                 |
| **WebUI** WS 客户端         | `@/root/cursor_workspace/openclaw/ui/src/ui/gateway.ts`                                                     |
| **WebUI** 设备身份          | `@/root/cursor_workspace/openclaw/ui/src/ui/device-identity.ts`                                             |
| **macOS/iOS** 共享 WS actor | `@/root/cursor_workspace/openclaw/apps/shared/OpenClawKit/Sources/OpenClawKit/GatewayChannel.swift`         |
| **macOS** 单例封装          | `@/root/cursor_workspace/openclaw/apps/macos/Sources/OpenClaw/GatewayConnection.swift`                      |
| **macOS** node 协调器       | `@/root/cursor_workspace/openclaw/apps/macos/Sources/OpenClaw/NodeMode/MacNodeModeCoordinator.swift`        |
| **iOS** 连接控制器          | `@/root/cursor_workspace/openclaw/apps/ios/Sources/Gateway/GatewayConnectionController.swift`               |
| **iOS** 应用状态            | `@/root/cursor_workspace/openclaw/apps/ios/Sources/Model/NodeAppModel.swift`                                |
| **Android** WS 会话         | `@/root/cursor_workspace/openclaw/apps/android/app/src/main/java/ai/openclaw/app/gateway/GatewaySession.kt` |
| **Android** 连接管理器      | `@/root/cursor_workspace/openclaw/apps/android/app/src/main/java/ai/openclaw/app/node/ConnectionManager.kt` |
| **Android** invoke 派发器   | `@/root/cursor_workspace/openclaw/apps/android/app/src/main/java/ai/openclaw/app/node/InvokeDispatcher.kt`  |

---

## 附录 B — 新增客户端 PR 检查清单

- [ ] `clientId` 加到 `GATEWAY_CLIENT_IDS` 并通过单测
- [ ] 选定 `role` 与 `scopes`（写在文档里）
- [ ] 实现 Ed25519 keypair 持久化（用平台安全存储）
- [ ] 实现 v3 payload 签名（`platform` / `deviceFamily` 必填）
- [ ] 实现 `connect.challenge` → `connect` 七步握手
- [ ] 实现 `(deviceId, role)` token 持久化 + bootstrap handoff scope 过滤
- [ ] `AUTH_TOKEN_MISMATCH` 一次性重试（仅 trusted endpoint）
- [ ] 实现非可恢复错误清单与重连暂停
- [ ] 指数退避 + keepalive ping
- [ ] 远程端点 TLS 指纹 pinning
- [ ] 如果是 node：实现 `node.invoke.request` 分发与 `node.invoke.result` 回复
- [ ] 如果是 node：上报 `node.event`（电量/位置/能力变化）
- [ ] 写 `docs/platforms/<your>.md`，并更新 `docs/platforms/index.md`
- [ ] 更新 `.github/CODEOWNERS` 与 `.github/labeler.yml`
- [ ] 跑 `pnpm test src/gateway/protocol` + `pnpm tsgo` + 平台原生测试

---

> **下一阶段建议（phase 2）**：选择具体客户端类型（例如 Windows companion），按 §5 的清单走，并选定要暴露的 caps/commands 子集，落 `apps/windows/` 的目录骨架与第一份握手 PoC。
