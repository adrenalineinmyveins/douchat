# 08 · Preload 与 IPC 契约

> 桥文件：[src/preload/index.ts](../../src/preload/index.ts)。契约类型：`DouchatApi`（[src/shared/types.ts](../../src/shared/types.ts)）。相关：[02 主进程](02-main-process.md) · [07 渲染层](07-renderer.md)

## 1. 机制

- `contextBridge.exposeInMainWorld('douchat', api)` 暴露**单一命名空间 `window.douchat`**。
- channel 命名：**`douchat:` 前缀 + kebab-case 动宾短语**（如 `douchat:send-message`）。
- 请求/响应：`ipcRenderer.invoke` ↔ `ipcMain.handle`；单向上报：`ipcRenderer.send`（仅 `douchat:diagnostic`、`douchat:window-action`）；订阅：`ipcRenderer.on` 并返回取消函数。
- 主→渲染推送事件：`douchat:snapshot`（100ms 节流全量快照）、`douchat:auth-state`、`douchat:update-state`、`douchat:credits-updated`、`douchat:connany-changed`。
- 多数 mutation handler 返回 `push()` —— 最新 `AppSnapshot`，渲染层即时 `setSnapshot`。
- 安全：handler 侧 `isDouchatRenderer(event.sender)` 校验来源；敏感操作附原生确认对话框。

## 2. API 表面（window.douchat，按域分组）

### 快照与消息

`getSnapshot` · `sendMessage` · `prewarmConversation` · `stopConversation` · `clearConversation` · `resetConversationContext` · `deleteMessage` · `searchMessages` · `getMessagePage` · `getAttachmentData` · `markConversationRead` · `markAllConversationsRead`

### 会话与话题

`deleteConversation` · `setConversationPinned` · `updateConversation` · `createGroup` · `createTopic` · `renameTopic` · `deleteTopic` · `setActiveTopic` · `openConversationWindow` · `openConversationWorkspace` · `chooseConversationWorkspace` · `clearConversationWorkspace`

### Agent

`createAgent` · `updateAgent` · `deleteAgent` · `startDirectChat` · `resolveAgentPermission` · `exportAgentArchive` · `parseAgentArchive` · `getAgentSkill`

### 本地 Agent 运行时

`detectLocalAgents` · `maintainLocalAgent` · `listLocalAgentModels` · `openLocalAgentTerminal` · `addCustomLocalAgent` · `updateLocalAgent` · `removeCustomLocalAgent` · `testLocalAgent` · `cancelLocalAgentTest` · `listSshHosts`

### 模型与端点

`setEndpoint` · `testEndpoint` · `getCustomModels` · `saveCustomModels` · `testCustomModel` · `authorizeTokenDance` · `cancelTokenDanceAuthorization`

### 群决策

`getDecisionSettings` · `saveDecisionSettings` · `testDecisionSettings` · `getCloudDecisionModels`

### IM 通道

`listIMChannels` · `connectIMChannel` · `disconnectIMChannel` · `startIMLogin` · `cancelIMLogin` · `pollIMLogin`

### 记忆

`getGroupMemory` · `saveGroupMemory` · `getUserMemory` · `saveUserMemory`

### 社交

`getSocialSnapshot` · `socialAction`（好友/房间全部操作走这一个 action 判别联合，见 [06](06-im-social-connectors.md)）

### 连接器

`connanyCommand` · `connanySelect` · `testEmailConnector` · `saveEmailConnector` · `disconnectEmailConnector`

### 账号 / 计费 / 更新

`getAuthState` · `startLogin` · `cancelLogin` · `retryAuth` · `signOut` · `refreshProfile` · `updateProfile` · `getUsageSummary` · `consumeCreditsReturn` · `openSubscriptionPlans` · `openBillingPortal` · `getUpdateState` · `checkForUpdates` · `installUpdate`

### 设备能力（`DesktopDeviceApi`，[shared/deviceApi.ts](../../src/shared/deviceApi.ts)）

`platform` · `windowAction` · `resizeDialog` · `copyText` · `copyAttachment` · `copyImageData` · `copyInvitationImage` · `saveInvitationImage` · `openLocalFile` · `requestMicrophoneAccess` · `openMicrophoneSettings`

### 诊断 / 自动化 / 代码预览 / 语言

`reportDiagnostic` · `openDiagnosticLogs` · `createRoutine` · `deleteRoutine` · `setRoutineEnabled` · `runRoutineNow` · `openCodeArtifact` · `getCodeArtifact` · `setInterfaceLanguage`

### 订阅事件（返回 unsubscribe）

`onAuthState` · `onSnapshot` · `onCreditsUpdated` · `onUpdateState` · `onConnanyChanged`

## 3. 新增 IPC 的步骤

1. 在 `src/shared/types.ts` 的 `DouchatApi` 增加方法签名（保持 preload/主进程/渲染层三方类型一致）。
2. `src/preload/index.ts`：实现方法（`invoke('douchat:kebab-name', args…)`）。
3. `src/main/index.ts`：`ipcMain.handle('douchat:kebab-name', handler)` —— 校验 `isDouchatRenderer`，mutation 返回 `push()`。
4. 渲染层调用 `window.douchat.xxx(...)`；全局类型经 [env.d.ts](../../src/renderer/src/env.d.ts) 声明。

## 4. 类型检查

`npm run typecheck` = `tsc --noEmit -p tsconfig.node.json`（main+preload）+ `tsc --noEmit -p tsconfig.web.json`（renderer+shared）。
