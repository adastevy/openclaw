# Phase 3 - Plugins / Channels 统一输入输出接口规范（源码锚定）

## 1. 目的与结论

本文回答一个核心问题：

- 不同 channel（Telegram、WhatsApp、未来自定义 channel）是否存在统一输入输出规范，能稳定接入同一个 Agent + Gateway 体系？

结论：**存在统一规范，而且是分层规范**。

- 平台侧（Telegram/WhatsApp）可以有不同原生协议和流程。
- 进入 OpenClaw 核心后，必须收敛到统一 I/O 契约：
  - 入站统一到 `MsgContext` / `FinalizedMsgContext`
  - 出站统一到 `ReplyPayload` + `ChannelOutboundAdapter`
  - 插件统一实现 `ChannelPlugin` 合同

关键源码锚点：

- `src/channels/plugins/types.plugin.ts`
- `src/channels/plugins/types.core.ts`
- `src/channels/plugins/types.adapters.ts`
- `src/auto-reply/templating.ts`
- `src/auto-reply/dispatch.ts`
- `src/auto-reply/reply-payload.ts`
- `src/channels/plugins/outbound.types.ts`
- `src/plugin-sdk/inbound-reply-dispatch.ts`
- `src/plugin-sdk/direct-dm.ts`
- `src/infra/outbound/deliver-types.ts`

---

## 2. 三层边界（你拆分 Phase3 的基准）

### 2.1 Plugins 层（扩展机制）

- 通过 `api.registerChannel(...)` 注册 channel 能力。
- 入口建议用 `defineChannelPluginEntry(...)`。
- 文档：`docs/plugins/sdk-entrypoints.md`

### 2.2 Channels 层（平台适配）

- 每个 channel 实现一个 `ChannelPlugin`。
- 负责平台特有解析、路由、线程、出站发送。
- 合同类型：`src/channels/plugins/types.plugin.ts`

### 2.3 Agent / Core 层（统一执行）

- 不感知 Telegram/WhatsApp SDK 差异。
- 只吃标准化上下文（入站）并产出标准化 payload（出站）。
- 核心入口：`dispatchInboundMessage(...)` in `src/auto-reply/dispatch.ts`

---

## 3. 统一输入规范（Channel -> Agent）

## 3.1 统一输入对象

统一入站对象是：`MsgContext`（可先构建后再 finalize）。

定义：`src/auto-reply/templating.ts`

这不是某个 channel 专属类型，而是全 channel 共用的 Agent 入站上下文模型。

## 3.2 最小可用字段（工程建议）

从 `src/plugin-sdk/direct-dm.ts` 的标准组装路径可抽出稳定最小集（`finalizeInboundContext(...)` 前后一致）：

- 消息主体：
  - `Body`
  - `BodyForAgent`
  - `RawBody`
  - `CommandBody`
- 路由与会话：
  - `SessionKey`
  - `AccountId`
  - `ChatType`
  - `From`
  - `To`
- 发送者与消息标识：
  - `SenderId`
  - `MessageSid`
  - `MessageSidFull`
  - `Timestamp`
- 面信息（影响路由和策略）：
  - `Provider`
  - `Surface`
  - `OriginatingChannel`
  - `OriginatingTo`

源码参考：`src/plugin-sdk/direct-dm.ts:106`

## 3.3 标准入站调用链

推荐复用 SDK 提供的公共链路，而不是每个插件自己拼：

1. 把平台 webhook / event 解析为内部消息
2. 组装 `MsgContext`（必要时先走 `finalizeInboundContext`）
3. 记录会话（`recordInboundSession`）
4. 分发到 Agent 回复流程（`dispatchReplyWithBufferedBlockDispatcher`）

标准封装：

- `recordInboundSessionAndDispatchReply(...)`
- `dispatchInboundReplyWithBase(...)`

源码：`src/plugin-sdk/inbound-reply-dispatch.ts:104`

## 3.4 统一入站分发入口

最终都进入：

- `dispatchInboundMessage(...)` in `src/auto-reply/dispatch.ts:50`

该函数签名固定为：

- `ctx: MsgContext | FinalizedMsgContext`
- `cfg: OpenClawConfig`
- `dispatcher: ReplyDispatcher`

这就是“不同 channel 进入同一 Agent 架构”的硬边界。

---

## 4. 统一输出规范（Agent -> Channel）

## 4.1 核心统一输出对象

核心输出对象：`ReplyPayload`

定义：`src/auto-reply/reply-payload.ts:7`

核心通用字段：

- `text`
- `mediaUrl` / `mediaUrls`
- `presentation`（通道无关富表现）
- `delivery`（通道无关投递偏好）
- `replyToId`
- `audioAsVoice`
- `channelData`（允许 channel 扩展）

要点：

- `text/media/presentation/delivery` 是通用契约。
- `channelData` 是差异化扩展槽，避免核心类型频繁变更。

## 4.2 Channel 出站适配合同

所有 channel 都通过 `ChannelOutboundAdapter` 对接核心发送链路。

定义：`src/channels/plugins/outbound.types.ts:75`

常见适配函数：

- `sendPayload`
- `sendFormattedText`
- `sendFormattedMedia`
- `sendText`
- `sendMedia`
- `sendPoll`

补充能力：

- `chunker` / `textChunkLimit`
- `normalizePayload`
- `renderPresentation`
- `resolveTarget`
- `targetsMatchForReplySuppression`

## 4.3 统一发送结果

发送结果统一回传 `OutboundDeliveryResult`：

- `channel`
- `messageId`
- 可选 `chatId/channelId/conversationId/toJid/pollId`
- 可选 `meta`（channel 扩展字段）

定义：`src/infra/outbound/deliver-types.ts:3`

---

## 5. 插件层统一注册规范（自定义 channel 必须满足）

## 5.1 Manifest 约束

`openclaw.plugin.json` 至少要声明：

- `id`
- `channels`
- `configSchema`

channel 插件建议补全：

- `channelConfigs.<channel-id>.schema`
- `channelEnvVars`（若存在 env 驱动配置）

文档：`docs/plugins/manifest.md`

## 5.2 Entry 约束

推荐：

- 主入口：`defineChannelPluginEntry(...)`
- setup 入口：`defineSetupPluginEntry(...)`

文档：`docs/plugins/sdk-entrypoints.md`

## 5.3 ChannelPlugin 合同

`ChannelPlugin` 统一合同定义于：

- `src/channels/plugins/types.plugin.ts:53`

常见必须实现面（按接入复杂度递增）：

- `config`
- `setup`
- `outbound`
- `gateway`（如需账号启动/停止、二维码登录等）
- `messaging`（目标解析、session grammar、payload transform）
- `threading`
- `security`
- `pairing`
- `actions`（共享 `message` 工具下的 channel 动作）

---

## 6. “统一”与“差异”的精确划分

## 6.1 强统一（必须一致）

- 入站边界对象：`MsgContext` / `FinalizedMsgContext`
- 入站调度入口：`dispatchInboundMessage(...)`
- 出站边界对象：`ReplyPayload`
- 出站适配合同：`ChannelOutboundAdapter`
- 插件主合同：`ChannelPlugin`

## 6.2 可差异（允许不同）

- 平台 SDK（Bot API、Webhook、长连接）
- 平台 ID 语义（jid/chatId/topic/thread）
- 目标解析规则（`normalizeTarget` / `parseExplicitTarget`）
- 线程模型与 reply 语义
- 原生能力扩展（poll/reaction/media 细节）

所以准确说法是：

- **统一的是核心契约，不是平台实现细节。**

---

## 7. 你接入自定义 channel 的落地清单（可执行）

1. 新建插件包并声明 `openclaw.plugin.json`（含 `channels` + `channelConfigs`）。
2. 用 `defineChannelPluginEntry(...)` 暴露 `ChannelPlugin`。
3. 实现 `config` + `setup` + `outbound` 三个最小面。
4. 入站消息统一映射为 `MsgContext`，通过 `inbound-reply-dispatch` 进入标准链路。
5. 出站统一接收 `ReplyPayload`，在 `ChannelOutboundAdapter` 内做平台适配。
6. 若有平台特有字段，放在 `ReplyPayload.channelData` 或 `OutboundDeliveryResult.meta`。
7. 增补 `messaging` 适配（target 解析、session grammar、thread 规则）。
8. 最后再扩展 `pairing/security/actions/gateway` 高级面。

---

## 8. 给你当前疑问的最终回答（可直接复用）

是的，存在你要的规范。

但它不是“所有 channel 外部 API 完全一致”，而是“所有 channel 与 OpenClaw 核心交互的接口一致”：

- 入站统一到 `MsgContext`；
- 出站统一到 `ReplyPayload` + `ChannelOutboundAdapter`；
- 插件统一到 `ChannelPlugin` 合同。

因此你后续接入自定义 channel 时，只要严格落在这三层统一边界，就能无缝接入现有 Agent + Gateway 架构。
