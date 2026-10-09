# 03 · Agent 运行时（DouchatRuntime）

> 核心文件：[src/main/runtime.ts](../../src/main/runtime.ts)（约 4300 行，应用大脑）。相关：[02 主进程](02-main-process.md) · [04 本地 Agent](04-local-agents.md) · [05 群组](05-group-collaboration.md)

## 1. 定位与分层

DouchatRuntime 是业务编排层，其下是两个库：

- **`@earendil-works/pi-agent-core`** 提供 `Agent`（回合引擎）：`session.prompt()` → 流式模型调用 → 工具执行 → 结果回填 → 继续循环；`session.continue()` / `session.abort()` / `session.state.messages` / `agent.subscribe(event)`。
- **`@earendil-works/pi-ai`** 提供模型抽象：`builtinModels()` 注册表，`setProvider/deleteProvider/getModel/checkAuth/streamSimple`。gateway 与自定义模型都以 `Provider` 形式注入；构造 Agent 的 `streamFn` 委托 `models.streamSimple(...)`。

```ts
export class DouchatRuntime {
  constructor(
    store: DouchatStore,
    computer: ComputerProvider,
    onChange: (snapshot: AppSnapshot) => void,
    cloudGateway?: CloudGatewayOptions,      // { baseUrl, resolveAccessToken, onUnauthorized, avatarFromImage }
    connectors: ConnectorProvider = emptyConnectors,  // { revision?, snapshot, createTools, prepare?, createLocalTools? }
    localExecutor: AgentExecutor = desktopAgentExecutor // 本地 CLI 执行器
  )
}
```

## 2. 关键公开方法

| 方法 | 说明 |
| --- | --- |
| `snapshot(): AppSnapshot` | 聚合 agents/conversations/messages(每话题最近 51 条)/privateMessages/activity/computers/routines/runs/runEvents/agentStatuses/runtime/endpoint/models/connectors 等 |
| `connect(): Promise<void>` | `gatewayConfig()`（云 token → 自定义端点 → .env）→ `fetchGatewayModels` → `models.setProvider(gatewayProvider(...))`；`connectionGeneration` 防竞态；失败记 `connectionError`，从不假装连接成功 |
| `setEndpoint(input) / testEndpoint(input)` | 保存/试连自定义 OpenAI 端点 |
| `sendMessage(conversationId, text, inputImages?, files?, mentions?)` | 渲染层发消息入口：`enqueueAgent('conversation:{id}')` 串行化；远端共享房间转 `humanSender`；文件经 `store.saveIMFile` 转链接 |
| `sendIMMessage(...) / receiveIMMessage(...)` | IM 渠道回合：每回合隔离会话 `direct:im:{conversationId}:{turnId}`（共享 transcript、不共享可变模型会话） |
| `runRoutine(routine, trigger)` | 例程执行：系统公告 → `createRun` → `runReply('routine:{id}', 'direct')` → `saveBubbles` |
| `greet(conversationId)` | 新会话/话题主动问候：Dr. Dou 首次无网直接落库内置中英文引导文案；否则 8s 超时 toolsDisabled 生成回合 |
| `recoverGroupWorkflows()` | 重启后恢复/暂停中断群工作流 |
| `prewarmConversation(id)` | 用户即将发消息前预热本地 Agent 进程（`localExecutor.prepare`） |
| `stopConversation / resetConversation / disposeAgent / workspaceChanged / refreshAgent` | 生命周期控制：中止相关 abort、清理会话 key 前缀、释放本地进程 |
| `configureCustomModels(records, defaultModel)` | 注册 `customModelProvider`、失效旧 provider、重绑 followDefaultModel 的 Agent |
| `customAgentModel / cloudAgentModel / defaultCloudAgentModel` | 供 create/update-agent IPC 计算**受信任**的 `{provider, model}` 绑定（渲染层不能走私模型选择） |
| `setHumanSender / setRoutineCreator / notifyChangesWith / setInterfaceLanguage / resolveAgentPermission` | 回注接口（SocialClient / Scheduler / 节流广播 / 语言 / 权限审批） |
| `executeSocialTask(...)` | 共享房间中被认领的委派任务执行（见 06） |
| `saveDecisionSettings / testDecisionSettings / getCloudDecisionModels` | 群决策设置（leader 模式或外部决策 provider） |

## 3. 私有核心链路（消息流转）

### 3.1 `enqueueAgent(key, task, wait?)`

按 key（agent 或 conversation）的 Promise 链串行化任务；排队超时默认 180s，可取消。

### 3.2 `runReply(options)` — 安全外壳

`store.captureAccountContext()` 捕获账号上下文 → 入队 → 队内 `assertAccountContext` 二次校验 → **永不抛出**（失败以 `{ text:'', error }` 返回）；登记 `pendingReplies` 供 `stopConversation` 取消。

### 3.3 `performReply({...})` — 单个 Agent 的一次模型回合（最核心）

**前置**：`statuses='thinking'`、`busyAgents` 计数、`activeConversation/activeTopic/activeRun`、`memoryTurns`、`permissions.beginTask`。

**记忆注入**：私聊 `userMemoryPrompt(shared, personal, writable)`；群聊 `groupMemoryPrompt(...)`；内部会话追加 `INTERNAL_MEMORY_POLICY + internalMemorySnapshot(...)`；本地 Agent 额外做 `userMemories.search` 预取。

**本地 Agent 分支**（`config.localAgentId`）：
1. `remoteAgentSpec` 判定远端 SSH → 工作区锁 `acquireWorkspace`（`workspaceLocks` 同一文件夹全局互斥）
2. 技能桥 `openLocalSkillBridge` / `openRemoteSkillBridge`（把 Douchat 工具经 loopback/socket 桥暴露给 CLI）
3. 组装大 prompt：身份（SOUL.md 等）、技能资源、记忆指令 `MEMORY_OPEN/CLOSE`、身份编辑 `FILE_EDIT_OPEN/CLOSE`、例程指令 `[[douchat_create_routine]]`、远程宿主声明等
4. `withReplyDeadline(() => localExecutor.run(...))` —— 超时 `LOCAL_REPLY_TIMEOUT_MS = 15min`
5. 后处理：保存产物图片/文件；**例程指令二次校验**（用只读 `localExecutor.run` 让原模型判定 `{"authorized":true/false}`，fail-closed）；身份编辑与记忆编辑落库并追加回执。

**云端分支**：
1. `this.session(config, sessionKey, context, toolsDisabled)` 取得/创建 pi-agent-core `Agent`
2. 每回合重建 systemPrompt（基础提示 + 记忆提示 + 工作区提示）
3. `waitForResponse(() => session.prompt(prompt, images))`：以"模型无活动"计时（工具执行/等待审批不计静默）；`CHAT_REPLY_TIMEOUT_MS = 120s`，控制器 `CONTROLLER_REPLY_TIMEOUT_MS = 30s`
4. **瞬时错误重试**：`MAX_TRANSIENT_REPLY_RETRIES = 3`，指数退避 `400ms × 2^n`，剔除失败占位消息后 `session.continue()`
5. **空回复恢复**：有工具结果但无文本时，一次性继续提示
6. 从 `session.state.messages` 提取最后一条 assistant 文本。

**收尾 `finish()`**：拼接 `generatedFiles` 链接、`takeToolActions`、工具回退文本；finally 清理活跃状态并 `retainIdleSession`（**5 分钟空闲驱逐，热会话上限 32**）。

### 3.4 `session(config, sessionKey, context, toolsDisabled)` — 会话缓存与工具装配

会话缓存键 `modelBinding = provider/model#thinking#connectorRevision`（变化才重建）。按 context 装配工具集：

| context | 工具集 |
| --- | --- |
| `controller` | 无工具（纯 JSON 规划） |
| `group` | `userMemoryTool + internalMemoryTool + routineTool + skillTools + skillInstallationTools + artifactTools + computer.createTools + connectors.createTools`（无 agent 管理工具，防他人改动联系人） |
| `direct` | group 全部 + `messageAgentTool`（A2A）+ `groupMessagingTools` + `memoryRetrievalTools` + `agentFileTools` + `agentManagementTools`（Dr. Dou 管联系人） |

所有工具经 `guardedTools` 包装：共享房间调用者需 `permissions.authorize(...)` 审批；computer 文件工具受 owner 权限位约束。`agent.subscribe` 把 `tool_execution_start/end` 映射为消息上的 `toolActions`（工具动作气泡）、`store.addRunEvent`、`setActivity`。

> 注意：`workspaceTools.createWorkspaceTools` 只给**云端** Agent（本地 CLI 直接用自己的 shell 在 cwd 工作）。

### 3.5 `performSendMessage` → `runDirectTurn` / `runGroupTurn`

- `performSendMessage`：@提及解析（`resolveMentionedMembers`/`addressesEveryone`）→ `store.addMessage` → `store.createRun({trigger:'chat'})` → 分派 → `dispatchGroupPosts`（群工具产生的待发帖子）→ 更新 run 状态。
- **`runDirectTurn`**：`directReplyPrompt`（带历史，IM/新会话附 transcript）+ `directA2ASourcePrompt` → `runReply(context:'direct', sessionKey:'direct:{conversationId}:{topicId}')` → `a2aReplyMessages` 解析 A2A 信封（信封永不入 transcript）→ `saveBubbles`（`splitBotReply` 拆气泡、@提及收件人、错误气泡）→ 逐条 `deliverA2A`（目标 Agent 私聊落库并触发其回合）→ **有界综合回合**（`handoff-summary:{...}` 会话，toolsDisabled，汇总各 Agent 回复）。
- **`runGroupTurn`**：构建 `BotGroup` 与 `GroupHealth` → 直接 @ 提及走 `directGroupDecision` 快路径 → `GroupWorkflowJournal` → 决策（leader 模式：群成员作 controller，JSON 校验 `validateGroupDecision`，最多 2 次格式纠错；或 `GroupDecisionService.decide` 外部决策）→ 决策失败降级（`DecisionEscalation` / `firstGroupPlan` 并发竞选）→ `runGroupConversation` 执行成员回合。详见 [05 · 群组协作](05-group-collaboration.md)。

## 4. 会话 key 体系（隔离设计）

| key 模式 | 用途 |
| --- | --- |
| `direct:{conversationId}:{topicId}` | 私聊（话题隔离） |
| `direct:im:{conversationId}:{turnId}` | IM 回合（每回合独立，共享 transcript） |
| `group:{…}` / `groupMemberSessionId` 等 | 群协调 / 成员发言互不复用，也不复用 bot 私聊会话 |
| `groupControllerSessionId` | 群决策 controller（无工具） |
| `handoff-summary:{…}` | A2A 综合回合 |
| `directA2ASessionId(source, target, topic?)` | 按源/目标对隔离的 A2A 会话 |
| `routine:{routineId}` | 例程 |
| `social:{owner}:{agent}:room:{roomId}` | 共享房间任务 |
| `game:{key}` | 群游戏回合 |

## 5. 超时与重试常量速查

| 常量 | 值 | 场景 |
| --- | --- | --- |
| `LOCAL_REPLY_TIMEOUT_MS` | 15 min | 本地 Agent 单回合上限 |
| `CHAT_REPLY_TIMEOUT_MS` | 120 s | 云端聊天静默超时（工具执行不计） |
| `CONTROLLER_REPLY_TIMEOUT_MS` | 30 s | 群决策控制器超时 |
| `MAX_TRANSIENT_REPLY_RETRIES` | 3 | 瞬时错误重试（指数退避 400ms×2^n） |
| 空闲会话驱逐 | 5 min / 上限 32 | 热模型会话 LRU |
| `BACKGROUND_BROADCAST_INTERVAL_MS` | 100 ms | 快照广播节流 |

## 6. 其他导出

- `botDescription(agent)`：Agent 的模型可见描述。
- `plainRemoteLinks(text)`：把远端 Agent 回复中的本地文件链接降级为纯文本（不泄露本机路径）。
- 群游戏装配：构造函数中 `new GroupGames(store, { language, changed, activity, reply })`，游戏回合复用 `runReply`；自定义模型 Agent 走 `groupDecisionService.complete`（无工具直连）。
