# 06 · IM 通道、社交与连接器

> 装配位置：[src/main/index.ts](../../src/main/index.ts)。设计文档：[docs/im-channels.md](../im-channels.md) · [docs/social-chat.md](../social-chat.md) · [docs/connany-connectors.md](../connany-connectors.md)

## 1. IM 通道 — [imChannels.ts](../../src/main/imChannels.ts)（`IMChannelManager`）

### 接入方式：全部本地主进程直连各平台官方 API，**不经 Douchat 云端中转**

| 渠道 | 传输 | 要点 |
| --- | --- | --- |
| Telegram | 直连 `https://api.telegram.org/bot<token>`（Electron `net.fetch`），`getUpdates` 30s 长轮询 | `connect()` 先 `getMe` 校验并拒绝已配 Webhook 的机器人 |
| 飞书 | 官方 `@larksuiteoapi/node-sdk` 的 `WSClient` **WebSocket 长连接**（appId/appSecret），订阅 `im.message.receive_v1` | 自建应用模式 |
| 微信 | iLink bot 协议直连 `https://ilinkai.weixin.qq.com`：`get_bot_qrcode`/`get_qrcode_status` 扫码（会话 240s）换 `bot_token`，之后 `getupdates` 长轮询 | 域名白名单校验 |

### 核心机制

- **凭据安全**：token/appId/cursor/inbox 经 `Codec`（safeStorage）按账号 sha256 文件名存 `userData/im-channels/`；密钥永不进 renderer。
- **配对**：peer 未绑定时要求在平台私信发送 `/pair <pairingCode>`；绑定后仅接受该 peer 的消息。一个渠道机器人只能绑定一个联系人；一个联系人可同时接三个渠道。
- **接收与幂等**：`acceptFeishu/acceptTelegram/acceptWechat` → `enqueue`：**先持久化收据**（`runtime.receiveIMMessage` 生成确定性 receiptId，防平台重投导致工具重复执行）再推进游标；`drain` 每渠道并发 4；中断的 `running` 任务重投时只提示"请到桌面检查结果"，不重跑。
- **回复管线**：`reply` 回调 = `replyToIM(store, runtime, agent, thread, text, …)`（[imReply.ts](../../src/main/imReply.ts)）→ `store.ensureIMConversation`（三渠道与桌面共用同一直聊会话）→ `runtime.sendIMMessage`（独立 turn 会话 `direct:im:{conv}:{turnId}`）。
- **发送**：`formatIMMessages(bubble, 3500, plain)` 分片（同时产出 Telegram entities / 飞书 post 富文本 / 微信纯文本）；图片经 `sendImage`（Telegram multipart、飞书先传 `im/v1/images` 取 image_key、微信 AES-128-ECB 加密后 `getuploadurl`）。
- **输入状态**：`acquireTyping`（每聊天一个指示器，引用计数）+ `startIMTyping`（4s 刷新）；微信 typing_ticket、Telegram `sendChatAction`、飞书在原消息上加/删 `Typing` 表情回应。

### 配套模块

| 模块 | 职责 |
| --- | --- |
| [imFormatting.ts](../../src/main/imFormatting.ts) | `formatIMMessages(markdown, limit, plain)`：mdast(GFM) 解析为带样式 span，UTF-16 代理对安全分片；微信模式链接内联展开 |
| [imMedia.ts](../../src/main/imMedia.ts) | `IMMedia`/`MAX_IM_FILE_BYTES`(20MB)、`downloadMedia`（流式限长可取消）、`decryptWechatMedia`（AES-128-ECB）、`imageInput`（魔数嗅探 PNG/JPEG/GIF/WebP，8MB） |
| [imTyping.ts](../../src/main/imTyping.ts) | `startIMTyping(signal, update, clear, refreshMs)`：尽力而为、串行化保证晚启动必被清理 |

## 2. 社交子系统 — [social.ts](../../src/main/social.ts)（`SocialClient`，经云端）

好友、共享群、跨设备消息全部走 `{webAppUrl}/api/desktop-auth/social`（Bearer 桌面登录 token）。

### 域模型（[shared/social.ts](../../src/shared/social.ts)）

`SocialSnapshot`（friendships/rooms/userId）、`SocialRoom`（direct|group，members=SocialPerson，agents=SocialAgent）、`SocialTask`（含 claim）、`SocialAction` —— **14 种 action 判别联合**（lookup/request/respond/create-room/invite-members/remove-members/add-agent/remove-agent/messages/send/leave-room/rename-room/group-invite/add-members），全部操作走 `socialAction` 一个 IPC。

### 关键机制

- **目标解析**：`sharedGroupReplyTargets`（@提及 + `checkHumanAgentTargets` 跨 owner 权限检查：deny 拒绝、未同步 fail-closed、ask 在执行边界审批）；`@all` 仅群主。`socialFollowUpTarget`（[shared/socialFollowUp.ts](../../src/shared/socialFollowUp.ts)）：10 分钟窗口内、无显式 @、上一人类消息恰好一个成功 Agent 任务时，追问续给同一 Agent。
- **发送**：`sendMessage` → `exportSocialFiles`（`douchat-file:` 本地文件卡转便携 base64，**绝不发送本地路径**）→ 计算目标（默认静默 `agentIds: []`）→ 乐观本地插入（`deliveryState:'sending'`，幂等 pendingSends）→ `action:'send'`。
- **同步**：`syncInbox`：`revision` 游标跳过未变房间；增量按 (时间,ID) 游标 + 5s 重叠窗口防边界丢消息；5 分钟周期全量核对；4 房间并发；`watch` 长轮询（15s 挂起）+ 2s 轮询回退。
- **三循环**（`start()`）：`heartbeat`（10s，携带权限审批中任务）、`sync`、`poll`（2.5s 任务队列：`tasks → claim → executeSocialTask → complete`；结果先写本地 outbox `saveSocialTaskResult`，发布失败可重试而不重跑工具；认领即预写"设备中断"结果防盲目重试）。
- **任务执行**：`runtime.executeSocialTask` → `performSocialTask`：跨 owner 请求先 `permissions.authorize`；会话 key `social:{owner}:{agent}:room:{roomId}`；提示词要求公开回复、共享历史当不可信数据；本地 CLI Agent 可用 `[[douchat_call_group_agent]]` 委托其他群 Agent。
- **共享群策略（重要）**：共享群默认**显式调用** —— 普通消息/@all/对人类的回复一律 `agentIds: []`，不选 leader、不调决策模型（桌面策略）；本地纯自有 Agent 群保留模型驱动的完整调度（见 05）。群主不可退群；`group-invite` 生成 7 天邀请链接 `/join-group?room=&token=`。

### [socialFiles.ts](../../src/main/socialFiles.ts)

`decodeSocialFiles`（≤4 附件、总量 20MB、base64 严格校验）/ `exportSocialFiles`（本地文件卡 → 便携载荷 + 从文本移除链接）。

## 3. 连接器框架

### Connany — [connany.ts](../../src/main/connany.ts)（`ConnanyManager`，经云端）

第三方平台连接器框架（GitHub 等）：所有操作 `POST {webAppUrl}/api/desktop-auth/connectors`。

- `command(op)`：list/connect/reconnect/session/disconnect/rename/access/openAccess/events；`pollEvents()` 30s 轮询共享事件流（`connection.revoked` 等）。
- `createTools(agentId, hooks)` 为 Agent 生成工具：`request_connection`、`request_resource_access`、`connector_accounts`、`{connector}_list_tools`、`{connector}_call_tool`；写操作经 `hooks.confirmWrite` 人工确认；`reauth_required` 内联重授权。
- 深链回调：`douchat://connectors/callback?connany_session_id=` → `handleReturn`（OAuth 回到桌面）。
- 类型与开关：[shared/connany.ts](../../src/shared/connany.ts)（`ConnectorCommand`、`CONNECTORS_ENABLED`）。

### 邮件连接器 — [emailConnector.ts](../../src/main/emailConnector.ts)（本地直连）

- IMAP（imapflow，TLS 强校验）+ SMTP（nodemailer）；凭据 safeStorage 加密存 `email-connectors.json`（`accountId:connectorId` 键隔离）。
- `test/save/disconnect/snapshot`；`createTools(agentId)` 仅给授权且非本地 CLI 的 Agent 提供**只读** `email_search` / `email_read` 工具。

### TokenDance 授权 — [tokenDanceAuth.ts](../../src/main/tokenDanceAuth.ts)

`authorizeTokenDance(openExternal, signal, request)`：自定义模型服务商 PKCE 授权 —— 本地 `127.0.0.1` 随机端口回调服务器 + S256，打开 `https://tokendance.space/auth`，回调后换 key；verifier 与错误不进 URL/日志，10 分钟超时。由 IPC `douchat:authorize-tokendance` 调用。

## 4. 模块装配关系

```text
index.ts
├─ DouchatRuntime(…, connectors = { emailConnectors, connany })   // createTools 注入回合工具
├─ IMChannelManager(…, reply=replyToIM → runtime.sendIMMessage,
│                    received=runtime.receiveIMMessage, diagnostics)
├─ SocialClient(webAppUrl, auth, store, runtime) + runtime.setHumanSender(social.sendMessage)
├─ ConnanyManager(webAppUrl, auth, …) → 深链 handleReturn / pollEvents / createTools
└─ EmailConnectorManager(userData, codec, …) → createTools（只读邮件）
```

持久化：社交消息与任务结果走 `DouchatStore`（`syncFriendConversation/saveSocialTaskResult/socialTaskOutbox`）；IM 收件箱与游标走加密 JSON；群健康/工作流/游戏/私信见 [05](05-group-collaboration.md)。
