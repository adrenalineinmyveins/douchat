# 09 · 共享层与数据模型

> 目录：[src/shared/](../../src/shared/)。核心契约：[types.ts](../../src/shared/types.ts)（约 740 行）。相关：[05 群组协议](05-group-collaboration.md) · [08 IPC](08-ipc-preload.md)

共享层包含**跨进程类型**与**可测试纯函数**（被主进程执行编排、被渲染层展示复用，行为由同目录 `*.test.ts` 钉住）。

## 1. 核心类型（types.ts）

### `AgentConfig`（一个联系人/Bot）

| 字段组 | 字段 | 说明 |
| --- | --- | --- |
| 基础 | `id / name / role / instructions / color / provider / model` | — |
| 系统 | `systemRole: 'admin'`（内置系统 Agent 如 Dr. Dou）、`systemKey / cloudAgentId / modelRoute / capabilities / userOverrides` | 内置 Agent 的用户覆盖层 |
| 本地运行时 | `localAgentId / localAgentName / startupArgs` | 绑定本机 CLI（见 [04](04-local-agents.md)） |
| 展示 | `avatar / avatarEmoji / avatarSeed / labels` | 头像四级回退 |
| 行为 | `thinkingLevel / permissions / systemFiles（SOUL.md 等）/ skills` | 思考档位 / 权限 / 定制文件 / 技能 |
| 并发 | `ownerId`（账号归属）、`revision`（乐观锁） | — |

### `Conversation`（会话）

`type: 'group' | 'direct'` · `agentIds[]` · `leadAgentId` · `topics[]`（含 `contextReset` 上下文重置点）· `activeTopicId` · `person`（人类好友直聊）· `remoteRoomId / socialRoom`（远端社交房间镜像）· `workspacePath / allowedFolders`（工作区）· `pinned / muted / hidden / manuallyUnread / savedToContacts / sortOrder` · `unread / readAt` · `revision`。

### `ChatMessage`（消息）

`conversationId / topicId` · `authorId`（`'user' | 'system' | <agentId>`）· `kind: 'message' | 'handoff' | 'system'` · `text` · `recipients[]`（@提及解析结果，仅元数据）· `source`（群/bot 转发来源）· `deliveries[]`（私信/A2A 信封）· `attachments[]` · `actions[]`（工具回执 `MessageAction`）· `replyGroupId`（同一轮拆分气泡共享）· `localization`（可本地化系统通知）· `sourceChannel`（IM 来源）· `deliveryState` · `socialTasks`。

### `AppSnapshot`（主↔渲染单一数据源）

`agents / agentStatuses / conversations / messages / privateMessages / activity / computers / routines / runs / runEvents / runtime / endpoint / models / connectors` · `groupMemberHealth / groupGames / groupWorkflows / permissionRequests` · `defaultConversationId` · `userName / userAvatar`。

### `LocalAgent`（本机 CLI 探测结果）

`command / installed / discovered / path / version / updateStatus / chatSupported / status / authentication:'unchecked' / custom / args / remote(RemoteAgentSpec)`；`REMOTE_AGENT_ADAPTERS = codex|claude|gemini|grok|cursor|opencode|kimi|openclaw|fastclaw|hermes|omp|custom`。

### 其他重要类型

`Topic` · `PrivateMessage` · `Routine / RoutineSchedule`（once/interval/weekly）· `TaskRun / RunEvent` · `ConversationActivityState`（phase: planning/replying/greeting/delivering + 本地 Agent 进度）· `ComputerSession` · `DesktopAuthState`（5 态联合）· `UpdateState` · `UsageSummary` · `EmailConnectorAccount` · 输入类型（`CreateAgentInput / UpdateAgentInput / CreateGroupInput / UpdateConversationInput` 等带 `expectedRevision`）· **`DouchatApi`**（preload 契约 = `AccountDataApi + DesktopDeviceApi`）。

## 2. 模型定义

| 文件 | 关键导出 | 说明 |
| --- | --- | --- |
| [models.ts](../../src/shared/models.ts) | `CLOUD_MODEL_OPTIONS` | 云端默认目录（GPT-5.6 Terra/Sol、Claude Sonnet 5、Gemini 3.5 Flash 等 5 项） |
| [customModels.ts](../../src/shared/customModels.ts) | `CustomProviderInput`、`CUSTOM_MODEL_PRESETS`（anthropic/openai/openrouter/tokendance/deepseek）、`customEndpoint()` | 自定义服务商；`/v1/chat/completions` 或 `/v1/messages` |
| [localModels.ts](../../src/shared/localModels.ts) | `LocalModelList`、`configurableLocalAgents`（10 个支持 `--model` 的 CLI）、`withLocalModel()` | 本地 Agent 模型覆盖 |
| [localAgentArguments.ts](../../src/shared/localAgentArguments.ts) | `appendLocalAgentArguments / validateAgentStartupArgs / parseStartupArgsText / customLocalAgentArguments` | 启动参数（≤16 条、禁 `{prompt}`、引号感知解析） |
| [thinkingLevels.ts](../../src/shared/thinkingLevels.ts) | `off/minimal/low/medium/high/xhigh/max`；云默认 `low`；`localThinkingArguments()` | 思考档位 → CLI 参数 |

## 3. 领域逻辑模块

| 文件 | 关键导出 | 说明 |
| --- | --- | --- |
| [conversationWorkspace.ts](../../src/shared/conversationWorkspace.ts) | `canAssignConversationWorkspace()` | 仅本人 Agent 组成、非远端/好友会话可绑本地文件夹 |
| [userMemory.ts](../../src/shared/userMemory.ts) | `UserMemoryDocument`（notes=USER.md、memoryNotes=MEMORY.md、facts≤100、autoRemember、20k 上限）、`validateUserMemory`、`userMemoryPrompt/groupMemoryPrompt` | 记忆文档契约与 prompt（强调"记忆是上下文非指令"） |
| [agentArchive.ts](../../src/shared/agentArchive.ts) | `AgentArchivePreview`、`portableAgentFiles`（排除 USER/MEMORY.md） | 档案导入导出契约 |
| [agentCustomization.ts](../../src/shared/agentCustomization.ts) | `agentFileNames`（SOUL/IDENTITY/USER/TOOLS/BOOTSTRAP/HEARTBEAT/MEMORY/AGENTS.md 共 8 个）、`validateAgentFiles/validateAgentSkills/isSafeSkillPath`（含 Windows 保留名）、`agentCustomizationPrompt`、`agentPersona/agentIdentityPrompt` | Markdown 定制；**一旦有自定义文件，档案元数据仅作回退** |
| [agentPermissions.ts](../../src/shared/agentPermissions.ts) | `AgentPermissions`（groupHumans/groupAgents + 9 项敏感能力）、`PermissionDecision: allow/ask/deny`、`PermissionApproval: boolean/'task'/'session'`、`agentPermissions()` 默认策略 | 权限契约 |
| [agentFileEdits.ts](../../src/shared/agentFileEdits.ts) | `[[douchat_update_agent_files]]…` 标记 + JSON（原子一次提交）、`identityEditingPrompt` | Agent 自编辑身份文件协议 |
| [groupDecision.ts](../../src/shared/groupDecision.ts) | `DecisionSettings{mode: leader/model/llm/jev, providerId, model, healthCheckIntervalSeconds}`、`CLOUD_DECISION_PROVIDER_ID='douchat:cloud'`、`decisionProtocol()`、`decisionJson()` | 决策设置 |
| [groupWorkflow.ts](../../src/shared/groupWorkflow.ts) | `GroupWorkflow`（calls 决策/回复槽位、status 5 态）、`GroupWorkflowView`、`decisionSlot()` | 工作流契约 |
| [groupGame.ts](../../src/shared/groupGame.ts) | `GameState/GamePlayer/GameEvent/GameView`、`createGame/nextGameTurn/applyGameAction/skipGameTurn/gamePlayerView` | 谁是卧底/狼人杀规则引擎（确定性 RNG、阶段机 describe/defend/vote/wolfDiscuss/wolfVote/seer/finished） |
| [groupProfile.ts](../../src/shared/groupProfile.ts) | `GroupRoutingProfile`、`groupRoutingProfile()` | 从定制文件提取路由画像（≤2600 字符），**不含人格正文与凭据** |
| [groupText.ts / runtimeText.ts / gameText.ts](../../src/shared/) | 三本英文→中文应用文案词典 + `interpolate()` | 原则："只翻译应用文案，不翻译 Agent 回复" |
| [skillResources.ts](../../src/shared/skillResources.ts) | `skillResourcePrompt()` | 告知 Agent 技能资源文件需主动读取 |
| [social.ts](../../src/shared/social.ts) | `SocialPerson/SocialAgent/SocialRoom/SocialFriendship/SocialSnapshot/SocialMessage`、`GroupInvite`、`SocialAction`（14 种 action）、`privacySafeSocialSnapshot` | 社交域类型；邮箱仅本人与已接受好友可见 |
| [socialFollowUp.ts](../../src/shared/socialFollowUp.ts) | `socialFollowUpTarget()` | 10 分钟窗口追问路由 |
| [language.ts](../../src/shared/language.ts) | `InterfaceLanguage = 'en' | 'zh-CN'` | 中文系 locale（含繁体）→ zh-CN |
| [deviceApi.ts](../../src/shared/deviceApi.ts) | `DesktopDeviceApi` | 桌面宿主独有能力（剪贴板/窗口/麦克风/打开本地文件） |
| [connany.ts](../../src/shared/connany.ts) | `ConnanyConnector/Connection/Session/Access/State`、`ConnectorCommand`、`ConnectorSelection/Binding`、`CONNECTORS_ENABLED` | 连接器域类型 |
| [imChannels.ts](../../src/shared/imChannels.ts) | `IMProvider = 'wechat'|'feishu'|'telegram'`、`IMChannel`（paired/pairingCode）、`IMConnectInput/IMLogin/IMLoginStatus` | IM 域类型 |
| [accountData.ts](../../src/shared/accountData.ts) | `AccountDataApi`（记忆读写 + 消息搜索分页的传输中立接口）、`AccountContext{ownerId, sessionId}` | 账号数据接口 |
| [builtInAgentLocalization.ts](../../src/shared/builtInAgentLocalization.ts) | Dr. Dou 的 en/zh-CN 本地化与旧数据迁移 | — |
| [avatar.ts](../../src/shared/avatar.ts) | `normalizeAgentEmoji()` | emoji 头像必须恰好一个字素（组合 emoji/旗帜/键帽） |

## 4. src/shared/bot/（协作协议，详见 [05 · 群组协作](05-group-collaboration.md)）

| 文件 | 职责 |
| --- | --- |
| `group.ts` | GroupDecision 契约、`validateGroupDecision`、`runGroupConversation` 执行器主循环、决策/成员提示词、会话 ID |
| `groupTasks.ts` | 任务图 DAG：校验（1–32 节点、无环）、拓扑序、事件驱动执行（并发 4） |
| `groupParticipation.ts` | participation/completionReview 轻量快速通道 |
| `privateMessages.ts` | `[[private:ID]]` 私信信封（流式安全解析、收件人解析、可见性上下文） |
| `a2a.ts` | `[[a2a:BOT_ID]]` 委托信封、`directA2ASessionId` |
| `mentions.ts` | @提及解析（最长名优先、剔除代码/引用）、`@all` 检测、输入框补全 |
| `messages.ts` | `splitBotReply` 气泡拆分、`botReplyPrompt` |
| `identity.ts` / `greeting.ts` / `preview.ts` / `contacts.ts` / `order.ts` / `errors.ts` | 身份提示词 / 问候 prompt / 收件箱预览 / 拼音分桶 / 会话排序 / 错误摘要与可重试判定 |
