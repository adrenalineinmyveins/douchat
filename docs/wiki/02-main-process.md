# 02 · 主进程核心模块

> 入口：[src/main/index.ts](../../src/main/index.ts)（约 1350 行，装配层）。相关：[01 架构](01-architecture.md) · [03 运行时](03-agent-runtime.md)

## 模块总览

| 文件 | 核心导出 | 一句话职责 |
| --- | --- | --- |
| [index.ts](../../src/main/index.ts) | — | 应用入口：窗口、深链、IPC 注册、模块装配与接线 |
| [store.ts](../../src/main/store.ts) | `DouchatStore` | SQLite 单文件持久层（douchat.db，WAL） |
| [userData.ts](../../src/main/userData.ts) | `applicationName` / `userDataDirectoryName` | 开发/生产互不污染的应用命名 |
| [desktopAuth.ts](../../src/main/desktopAuth.ts) | `DesktopAuth` | PKCE 登录、token 加密持久、资料/额度 API |
| [authProtocol.ts](../../src/main/authProtocol.ts) | `createDesktopLoginUrl` 等 | `douchat://` 协议 URL 纯函数（构造与解析） |
| [gateway.ts](../../src/main/gateway.ts) | `fetchGatewayModels` / `gatewayProvider` | OpenAI 兼容端点 → pi-ai Provider 适配 |
| [scheduler.ts](../../src/main/scheduler.ts) | `RoutineScheduler` / `nextRoutineOccurrence` | 轮询式定时任务调度 |
| [customModels.ts](../../src/main/customModels.ts) | `CustomModelStore` / `customModelProvider` | 自定义模型服务商（key 加密存储） |
| [internalMemory.ts](../../src/main/internalMemory.ts) | `internalMemorySnapshot` / `INTERNAL_MEMORY_POLICY` | 内部会话跨会话记忆快照 |
| [userMemoryStore.ts](../../src/main/userMemoryStore.ts) | `UserMemoryStore` | 双层记忆：SQLite + Markdown 文件镜像 |
| [diagnostics.ts](../../src/main/diagnostics.ts) | `DiagnosticLog` / `redact` | JSONL 诊断日志与脱敏 |
| [updater.ts](../../src/main/updater.ts) | `DesktopUpdater` / `safeUpdateError` | electron-updater 状态机封装 |

---

## 1. index.ts — 装配层

### 启动顺序（app ready 前）

1. `app.disableHardwareAcceleration()`（仅 win32）；`app.setName(applicationName(development))`；`app.setPath('userData', …)`
2. `configureLocalAgentRegistry` / `configureRemoteTransport` / `configureManagedNode` / `configureLocalWorkspaces`（注入 userData 路径）
3. `new CustomModelStore(…)`（注入 safeStorage codec）、`new DiagnosticLog(…)` + `crashReporter.start({ uploadToServer: false })`

### `app.whenReady()` 内的构造链

```text
Menu.setApplicationMenu(null)(win32) → configureMediaPermissions
→ store = DouchatStore.atUserData(…)，随即 store.setCurrentAccountId('')   // 识别会话前不跑旧账号任务
→ EmailConnectorManager → ConnanyManager → LocalComputerProvider
→ runtime = new DouchatRuntime(store, computer, scheduleBroadcast, CloudGatewayOptions, connectors)
   // CloudGatewayOptions = { baseUrl: chatApiBaseUrl(webAppUrl), resolveAccessToken: () => auth?.getAccessToken(),
   //                        onUnauthorized: () => auth.invalidateSession(), avatarFromImage }
→ runtime.notifyChangesWith(scheduleBroadcast)
→ imChannels = new IMChannelManager(…)          // IM 凭据同样 safeStorage 加密
→ scheduler = new RoutineScheduler(store, runtime, scheduleBroadcast, async () => (await auth.getUsageSummary()).credits)
→ runtime.setRoutineCreator(input => scheduler.createRoutine(input))   // 解除构造循环依赖
→ updater = new DesktopUpdater(…, busyTaskCount: () => runtime.snapshot().activity.length)
→ auth = new DesktopAuth(webAppUrl, authScheme, development, userDataPath, broadcastAuth)
→ social = new SocialClient(…)；runtime.setHumanSender((id, text, …) => social.sendMessage(…))
→ createWindow() → auth.initialize() → updater.startAutomaticChecks() → scheduler.start() → powerMonitor.on('resume')
```

### 关键函数

| 函数 | 说明 |
| --- | --- |
| `createWindow()` | 主窗口 1240×800，沙箱配置，preload `../preload/index.cjs` |
| `broadcast(snapshot)` | `notifyWindows(allWindows, 'douchat:snapshot', s)`；按"本地 Agent 是否 thinking"启停 `powerSaveBlocker` 防休眠 |
| `scheduleBroadcast()` | 节流：`BACKGROUND_BROADCAST_INTERVAL_MS = 100ms` |
| `broadcastAuth(state)` | 账号切换：停旧会话（`runtime.stopConversation/disposeAgent`）、关聊天/预览窗、`store.setCurrentAccountId`、`reloadCustomModels()`、`scheduler.accountChanged()`；登录成功后 `ensureDefaultCloudContact`（创建 Dr. Dou）+ 刷新内置 Agent → `runtime.connect()` + `runtime.greet(welcomeConversationId)` |
| `receiveAppUrl(url)` | 深链路由：connectors 回调 → `connany.handleReturn`；`douchat://group/open?room=` → `openPendingGroup`；`douchat://payment/callback?status=paid` → `pendingCreditsRefresh`；`douchat://auth/callback` → `auth.handleCallback` |
| `openChatWindow(conversationId)` / `openCodeArtifactWindow(input)` | 独立聊天窗口（Map 复用）与代码工件预览窗（iframe 拦 `will-frame-navigate`） |
| `push()` | 取 `runtime.snapshot()` → broadcast → 返回；绝大多数 mutation IPC handler 的返回值 |
| `isDouchatRenderer(contents)` | IPC 安全校验：sender 必须属于本应用的 BrowserWindow |
| `reloadCustomModels()` | `runtime.configureCustomModels(customModels.records(account), customModels.list(account).defaultModel)` |

### 生命周期收尾

`before-quit`：停更新检查、`cancelLocalModelQueries`、停 social/IM、释放 powerSaveBlocker、逐 agent `runtime.disposeAgent`、`disposeAllLocalAgentSessions`、`scheduler/computer.dispose`、`closeRemoteConnections`。`will-quit`：`store.close()`（checkpoint WAL）。渲染进程崩溃 → 诊断 + 恢复对话框（Ctrl/Cmd+Shift+L 打开日志目录）。

---

## 2. store.ts — DouchatStore（SQLite 持久层，约 2400 行）

### 设计

- 引擎：`node:sqlite` 的 `DatabaseSync`，单文件 `<userData>/douchat.db`，开 WAL + foreign_keys。
- 表：`meta`、`agents`、`conversations`、`messages`（级联删除 + `messages_topic` 索引）、`groupGames`、`groupWorkflows`、`privateMessages`、`routines`、`runs`、`runEvents`、`attachmentOwners`。
- **JSON 列模式**：路由/级联依赖的列为真实列并建索引，整条记录以 JSON 存于 `data` 列 —— 新增可选字段无需迁移。
- 常量：`LOCAL_DATA_VERSION = 1`（`PRAGMA user_version` 守卫）、`PRIVATE_MESSAGE_LIMIT = 400`、`RUN_LIMIT = 120`、`RUN_EVENT_LIMIT = 600`。

### 关键方法（按域）

| 域 | 方法 | 说明 |
| --- | --- | --- |
| 账号 | `setCurrentAccountId(id)` / `currentAccountId` / `captureAccountContext()` / `assertAccountContext(ctx)` | 切换时重置 `accountSessionId`；sessionId 令牌防回合中途换号 |
| Agent | `ensureDefaultCloudContact(accountId, binding, remoteManifest?)` | 每账号内置管理员 **Dr. Dou**（豆博士） |
| | `createAgent / updateAgent / deleteAgent` | admin 不可删；更新支持乐观锁 `expectedRevision`；技能目录物化到 `skills/skill-*` |
| 会话 | `createGroup` / `updateConversation` / `ensureDirectConversation` / `ensureIMConversation` | IM 三渠道与桌面共用同一直聊会话 |
| 话题 | `createTopic / renameTopic / deleteTopic / setActiveTopic` | 话题隔离运行时会话与历史 |
| 消息 | `recentMessages()`（每会话每话题最近 51 条）/ `messagePage()`（50 条分页）/ `contextMessages / contextPrivateMessages`（尊重 `contextReset`）/ `addMessage` / `resetConversationContext` | 快照只带最近 51 条，更早的分页拉取 |
| 附件 | `saveIMFile`（≤20MB）/ `saveImageAttachment`（≤8MB）/ `attachmentDataUrl` / `ownedDocumentPath` | 写 `attachments/`，返回 `douchat-file:` 链接；owner 表 + 目录校验防路径逃逸 |
| 例程 | `createRoutine(input, nextRunAt)` / `setRoutineEnabled` / `markRoutineTriggered` | — |
| 运行 | `createRun / updateRun / addRunEvent` | 每次触发的 TaskRun 及事件流 |
| 群组 | `decisionSettings/saveDecisionSettings`、`groupHealth/saveGroupHealth`、`commitGame(game, expectedRevision)`、`saveGroupWorkflow/groupWorkflows` | 游戏与工作流 revision CAS |
| 社交 | `syncFriendConversation` / `saveSocialTaskResult` / `socialTaskOutbox` | 好友房间同步、任务结果发件箱 |
| 端点 | `endpoint` getter/setter | OpenAI 兼容自定义端点（账号作用域 meta） |

### 构造期迁移链

`seed → removeUnusedLegacyDemo → backfillSystemAdminRoles → backfillAgentAvatarSeeds → backfillAccountOwnership → UserMemoryStore/GroupMemoryStore 初始化 → migrateLegacyUserMemories → migrateAccountSensitiveMeta → backfillAttachmentOwnership → backfillMessageSourceContent → backfillDeliveryReplies → recoverInterruptedRuns（崩溃时 queued/running 的 run 标记 failed；一次性例程延迟 60s 重试）→ mergeLegacyIMConversations`

另有 `EMBEDDED_BUILT_IN_AGENT_MANIFEST`：内嵌的离线兜底 Dr. Dou 清单。

---

## 3. userData.ts

| 函数 | 返回 | 说明 |
| --- | --- | --- |
| `applicationName(development)` | `'Douchat Dev' | 'Douchat'` | 同时决定 macOS Keychain 中 safeStorage 服务名，避免开发/正式版争抢凭据 |
| `userDataDirectoryName(development)` | `'douchat-dev' | 'douchat'` | `app.setPath('userData')` 使用 |

---

## 4. 认证：desktopAuth.ts + authProtocol.ts

### authProtocol.ts（纯函数）

| 导出 | 说明 |
| --- | --- |
| `desktopAuthScheme()` | 生产 `douchat`；开发 `douchat-dev` + loopback 回调 |
| `normalizeWebAppUrl(value, development)` | 默认 `https://douchat.ai`；**打包版忽略环境变量**（防 shell 劫持 token） |
| `chatApiBaseUrl(webAppUrl)` | `{origin}/v1` —— 云模型网关 baseUrl |
| `createDesktopLoginUrl(…)` | 登录页 `/desktop-auth?client_id=douchat-desktop&state=&code_challenge=&redirect_uri=`（PKCE S256） |
| `parseDesktopAuthCallback(input, expectedRedirectUri, expectedState)` | 校验协议/host/port/path、state 一致、code 为 43 位 `[A-Za-z0-9_-]` |
| `isDesktopAuthUrl / isDesktopCreditsUrl / parseDesktopGroupUrl / parseDesktopConnectorsUrl` | 各深链识别 |

### DesktopAuth 类（关键流程）

- **`startLogin()`**：生成 `codeVerifier`+`state`，`codeChallenge = sha256(verifier)`；开发模式 `startLoopbackCallback()` 在 `127.0.0.1:随机端口/auth/callback` 起本地 HTTP 服务；打包模式 redirectUri 为 `douchat://auth/callback`；`PendingFlow` 原子写 `auth-flow.json`（10 分钟有效）；`shell.openExternal` 打开浏览器。
- **`handleCallback(url)`**：`parseDesktopAuthCallback` → `exchangeCode`（`POST /api/desktop-auth/token`，token 必须以 `dch_` 开头）→ `saveCredential`（safeStorage 加密写 `auth.json`，0600，临时文件+rename）；Keychain 拒绝时降级为内存会话。
- **`initialize()`**：读 `auth.json` 解密 → `fetchUser`（`GET /api/desktop-auth/me`）；401/403 → 清凭据转 `signed-out`。
- **`getAccessToken()`**：仅供主进程模型传输；不进入任何 IPC 契约。
- 其它：`signOut`（`POST /api/desktop-auth/revoke`）、`invalidateSession`（runtime `onUnauthorized` 触发）、`refreshProfile / updateProfile`（PATCH `/me`；头像 `POST /api/desktop-auth/avatar` ≤2MB）、`getUsageSummary`（GET `/usage` → credits/planName）、`getBuiltInAgentManifest`（GET `/built-in-agents`，必须含 `dr-dou` admin）、`openBillingPortal / openSubscriptionPlans`。

---

## 5. gateway.ts — OpenAI 兼容网关（客户端）

把远端 OpenAI 兼容端点适配为 pi-ai `Provider`。**注意：本文件不含 HTTP 服务器**，`GET /v1/models`、`POST /v1/chat/completions` 是作为客户端消费的远端接口（默认 `https://douchat.ai/v1`）。

| 导出 | 说明 |
| --- | --- |
| `GatewayConfig` | `{ baseUrl, apiKey?, resolveApiKey?, authName?, authSource?, providerName?, assumeImageInput?, onUnauthorized? }` —— 第一方云用 `resolveApiKey`（每次请求重新解析 token，登出立即失效） |
| `fetchGatewayModels(config, signal?)` | `GET {baseUrl}/models`；401 触发 `notifyUnauthorized`；`chatModels()` 过滤仅保留 chat 类，按 ID 正则推断视觉/推理能力；`contextWindow` 固定 128000 |
| `gatewayProvider(models, config): Provider` | `createProvider({ id:'gateway', api:{ stream: watchedStream, … } })`；`monitoredFetch` 检查 401 → `config.onUnauthorized`（→ `auth.invalidateSession`） |
| `gatewayEnvConfig()` / `isGatewayConfig(config)` | `.env` 兜底（`GATEWAY_BASE_URL` / `GATEWAY_API_KEY`）与配置判定 |

---

## 6. scheduler.ts — RoutineScheduler

- **`nextRoutineOccurrence(schedule, after)`**（导出纯函数）：`once → runAt`；`interval → after + max(1, round(intervalMinutes))分钟`；`weekly →` 未来 7 天内第一个匹配 `days + HH:mm` 的时刻。
- `start() / dispose() / accountChanged() / checkNow()`（电源恢复后补跑）。
- `createRoutine(input)`：也是 `runtime.setRoutineCreator` 的回调 —— 聊天中创建的例程走同一路径。
- `scheduleNextTick()`：取启用例程最早 `nextRunAt`；定时器上限 `MAX_TIMER_DELAY = 60s` —— 长周期靠反复重排，睡眠唤醒后最多延迟 1 分钟。
- `runDueRoutines()`：到期 → `markRoutineTriggered`（写下次时间；`once` 触发即停用）→ 额度检查（仅确认 credits ≤ 0 时阻断**云端** Agent，余额服务故障不阻断）→ `pauseBeforeCreditlessRun`（停用 + 系统消息）→ `runtime.runRoutine(routine, 'schedule')`；一次性任务失败重试至多 `ONE_TIME_MAX_ATTEMPTS = 2` 次、延迟 60s。

---

## 7. customModels.ts — CustomModelStore

- `validateCustomProvider(input)`：id `[a-zA-Z0-9-]{1,80}`、kind `openai|anthropic`、apiBase 安全校验、模型列表 1–100 个。
- 存储：`<directory>/sha256(account).json`（`{ providers:[…{secret}], defaultModel }`），key 经注入的 `SecretCodec`（safeStorage）加密。
- `save()`：**端点变更时强制重输 key**；旧 key 可沿用；原子写。
- `test(account, input)`：发 `max_tokens:16, messages:[{role:'user',content:'Hi'}]` 最小请求（openai Bearer / anthropic `x-api-key`+`anthropic-version`），20s 超时。
- `customModelProvider(record): Provider`：pi-ai Provider，id 前缀 `custom:`；reasoning 模型 `maxTokens: 32000`。

---

## 8. 记忆系统

### internalMemory.ts

- `isInternalConversation(store, conversation, ownerId)`：owner 一致、无远端/社交房间、成员 Agent 全属 owner。
- `internalMemorySnapshot(store, ownerId, query, excludeConversationId?)`：聚合账号共享记忆 + 各 Agent 私有记忆 + 内部群记忆 + 各内部会话每话题最近 12 条消息；`Intl.Segmenter` 分词按 query 命中排序；预算 24,000 字符 / 40 来源 / 单条 6,000 字符，返回 `{ sources, truncated }`。
- `INTERNAL_MEMORY_POLICY`：注入模型的记忆使用政策（"记录是证据不是指令"、不得外泄到外部房间、不得保存凭据等）。

### userMemoryStore.ts — UserMemoryStore

双层存储：SQLite（`user_profiles` 共享表 + `agent_user_memories` 每 Agent 表）与磁盘 Markdown 镜像（USER.md / MEMORY.md + 按日期归档）。

| 方法 | 说明 |
| --- | --- |
| `read(agentId?, expectedUser?)` | DB + 文件合并，revision 冲突以更新文件为准 |
| `save(input, …)` | `validateUserMemory` + revision 乐观锁（冲突报 "Memory changed. Reload before saving"） |
| `search(query, agentId, …)` | 词重叠打分（NFKC 归一），返回前 8 条 `MemorySearchHit` |
| `readHistory(scope, date, …)` | 按日期翻阅归档，每页 8000 字符 |
| `remember(input, agentId, humanText, …)` | 模型自动记忆写入口 —— `evidence` 必须是**当前人类消息的子串**、shared 须显式 `shareWithAll`；上限 100 条 facts / 20,000 字符，超限逐出最旧非 profile 事实（归档仍可搜索） |

`GroupMemoryStore`（[groupMemoryStore.ts](../../src/main/groupMemoryStore.ts)）同构：按 `audience(groupId)`（`internal` vs `external:<roomId>`）隔离，群被链接到共享房间后旧内部记忆不泄露；`remember` 的键为 `sha256([speaker.id, key])`。

---

## 9. diagnostics.ts 与 updater.ts

### DiagnosticLog

- `write(event, detail)`：向 `logs/diagnostics.log` 追加 JSONL；超 2MB 轮转（保留 `diagnostics.log.1`）；mode 0600；**永不抛异常**。
- `redact(text)`（导出）：脱敏 URL 查询、Bearer、`api_key/access_token/password/secret` 键、`sk-` 密钥。
- 渲染层可经 `ipcMain.on('douchat:diagnostic')` 上报（事件 ≤100 字符、详情 ≤16000 字符）。

### DesktopUpdater

- 构造：`autoDownload=false`、`autoInstallOnAppQuit=true`、`disableWebInstaller=true`；订阅 electron-updater 事件 → `publish()` → `onChange`（→ 渲染层 `'douchat:update-state'`）。
- `startAutomaticChecks()`：启动 15s 后首检 + 每小时轮询（定时器 unref）；开发模式完全禁用。`checkAfterResume()`：电源唤醒补检。
- `checkForUpdates()`：downloading/downloaded/installing 期间防抖。
- `installUpdate()`：available → download；若 `busyTaskCount() > 0`（runtime 活动任务）则停留 downloaded 并提示 busyTasks；空闲则 `quitAndInstall(false, true)`。
- `safeUpdateError(cause)`（导出）：错误信息脱敏 token/signature。

---

## 10. IPC channel 命名规范

- 统一前缀 **`douchat:`** + kebab-case 动宾短语：`douchat:get-snapshot`、`douchat:send-message`、`douchat:create-agent`…
- 请求/响应用 `ipcMain.handle`；单向上报仅 `douchat:diagnostic`、`douchat:window-action`。
- 主→渲染推送：`douchat:snapshot`（100ms 节流全量快照）、`douchat:auth-state`、`douchat:update-state`、`douchat:credits-updated`、`douchat:connany-changed`。
- 完整 API 表面见 [08 · Preload 与 IPC 契约](08-ipc-preload.md)。
