# 05 · 群组协作子系统

> 核心协议：[src/shared/bot/](../../src/shared/bot/)（纯函数，被主进程执行、被渲染层复用）。编排装配在 [runtime.ts](../../src/main/runtime.ts) 的 `runGroupTurn`。设计文档：[docs/group-orchestration-design.md](../group-orchestration-design.md)

## 1. 分层原则（源自设计文档，代码已印证）

> 消息可见性（存储）→ 语义判断（Jev 或 LLM controller）→ 任务规划（Leader 规划会话）→ 发言与执行（确定性调度器）→ 活动规则（游戏状态机）

不允许一个模型同时成为唯一状态存储与最终执行权威。当前实现两种决策模式：**默认（leader 成员规划）** 与 **决策模型（model/jev 外部服务）**。

## 2. 协作协议核心 — [shared/bot/group.ts](../../src/shared/bot/group.ts)

### `GroupDecision`（分派决策契约）

字段：`mode: 'none'|'single'|'parallel'|'sequential'`、`memberIds`、`triggerMessageIds`、`participationOnly`（每成员一次个人贡献，缺席者跳过不冒名）、`participantScope`、`supervise`、`requireSummary`、`assignments`（成员→交付物）、`publicDeliverables`、`addressedMemberId`、`waitForHuman`、`leaderFirst`、`tasks`（DAG 任务图）、`recoveryAction`（skip/replace/pause）。

### 关键导出

| 导出 | 说明 |
| --- | --- |
| `validateGroupDecision(raw, group, context)` | **只验证、绝不推断**的契约校验器：任务图与 memberIds 一致、不可用成员不得派工、triggerMessageIds 必须在可见消息/私信信封中 |
| `explicitGroupDecision / directGroupDecision` | @提及结构提示；后者：人类消息**行首 @ 单一成员且无其他显式提及**时绕过规划直通 single 路由 |
| `groupConversationContinuity` + `GROUP_CONTINUITY_INSTRUCTION` | 识别"上一轮唯一发言人"作为会话延续伙伴（followup） |
| `runGroupConversation({...})` | **执行器主循环**（见 §4） |
| `groupDecisionPrompt(group, context, coordinator)` | 分派策略提示词（"你是调度策略不是参与者"），输出 minified JSON |
| `groupConversationPrompt(group, speaker, messages, turn, privateMessages)` | 成员回复提示词，内含消息传输契约（见 §3） |
| `groupTopicSessionId / groupControllerSessionId / groupMemberSessionId` | 会话隔离：协调、成员发言互不复用 |
| `GROUP_MAX_TURNS = 16`（库默认；runtime 传 `maxTurns: 128`）、`sharedGroupMessages`（48k 预算裁剪） | 上限守护 |

## 3. 消息传输契约（协议标记）

| 标记 | 含义 | 解析 |
| --- | --- | --- |
| `<!-- message_break -->` | 拆分多条气泡（≤4 条/轮） | `splitBotReply`（[bot/messages.ts](../../src/shared/bot/messages.ts)：显式分隔符 > 段落 > 句子，代码围栏保护） |
| `@name` / `@all` | 可执行公开交接（下一轮触发） | `mentionedMembers`（NFKC 归一、最长名优先防 @Ann 命中 @Anna；剔除代码/引用/链接，[bot/mentions.ts](../../src/shared/bot/mentions.ts)） |
| `[[private:MEMBER_ID]]…[[/private]]` | 私信投递给群成员；`[[private:human]]` 给用户 | `parsePrivateReply` / `privateReplyDeliveries`（[bot/privateMessages.ts](../../src/shared/bot/privateMessages.ts)；**流式安全**：残缺开头标记也扣留；≤20 块/块 ≤12000 字符） |
| `[[private-info:ID]]…` | 仅告知不触发行动（intent: `inform` vs `request`） | 同上，`PrivateDelivery.intent` |
| `[[a2a:BOT_ID]]…[[/a2]]` | 直聊场景 Bot 间私有委托 | `parseA2AReply` / `a2aReplyMessages`（[bot/a2a.ts](../../src/shared/bot/a2a.ts)；只接受 availableBots 里的不透明 id） |
| `[[douchat_silent]]` | 成员选择沉默（被直接点名时） | runtime：输出恰为该标记且无附件/动作 → 静默返回 |

私信投递后：接收成员的该投递成为其 trigger（`handoffsFrom`），其下一轮的 `privateContext` 才能读到正文 —— **controller 只见信封，正文仅收发双方可见**。

## 4. 群消息一轮完整流转

以本地群（无 `remoteRoomId`）为例：

1. **入口** `runtime.sendMessage` → `performSendMessage`：@提及解析（`resolveMentionedMembers`/`addressesEveryone`）→ `store.addMessage` → `createRun` → `runGroupTurn`。
2. **构造上下文**：
   - `groupRoutingProfile`（[shared/groupProfile.ts](../../src/shared/groupProfile.ts)）：只提取成员 IDENTITY/SOUL/AGENTS/TOOLS.md 中明确标注的 routing/capabilities 段落 + 技能描述 + 权限（≤2600 字符），**人物散文不外泄**。
   - 有 `waiting` 工作流：用户消息标记 `resumesGroupTask` 拼接早前待澄清任务。
   - `directGroupDecision`：行首 @单人直通 single（跳过健康探测）。
   - 否则 `refreshGroupHealth`（`probeGroupMember` 发送 "Reply exactly PONG" 无工具探测）→ `rankGroupMembers` 排序 → `store.contextPrivateMessages` → 创建 `GroupWorkflow` + `GroupWorkflowJournal`。
3. **决策**（decide 回调）：
   - `settings.mode === 'model'` → `decisionProvider(settings)` → `GroupDecisionService.decide(...)`（按 `decisionProtocol(model)` 选 `jev()` 或 `plan()`）。
   - leader 模式 / 回退：`firstGroupPlan`（[groupPlanning.ts](../../src/main/groupPlanning.ts)：`GROUP_PLANNING_BUDGET_MS = 180s`、attempt 60s）让候选成员在 `groupControllerSessionId` 控制会话（无工具）用 `groupDecisionPrompt` 产出 JSON → `validateGroupDecision`；失败计入 `failedControllers` 并发系统提示换协调者。
   - 每次决策经 `journal.call(decisionSlot(context, version), 'decision', …)` **幂等化**；`decision.leaderMemberId` 变化 → 持久化 `leadAgentId` 并广播 "Leader elected"。
4. **执行循环 `runGroupConversation`**（[bot/group.ts](../../src/shared/bot/group.ts)）：
   - 初始决策：`none` → 结束；`waitForHuman/leaderFirst` → leader 单人；`sequential` 且 lead 不在首位 → `deferredInitialDecision`（lead 先开场）。
   - `tasks` 图 → `executeGroupTaskGraph` 流式执行；否则 `parallel`（并发 ≤4）或 `sequential`（`handoffsFrom` 把前序输出中的 @提及与非 inform 私信作为后续 trigger）。
   - 成员失败 → `executeWithFailover` → `decide(recoveryContext)` → `recoveryAction: 'skip'`（绝不冒名）/`'replace'`（校验 `requiredCapabilities` 权限）/`'pause'`；失败成员 `quarantine` 隔离。
   - 批次结束 supervised → 再 `decide`（continuation）；`requireSummary` 补 lead `finalize` 轮。
5. **成员回复**（reply 回调）：`groupConversationPrompt` + `botReplyPrompt` → `runReply`（会话 key = `groupMemberSessionId` + workflow/task 后缀；`onProgress` → `journal.progress`）→ 私信剥离（`privateReplyDeliveries`）→ `storePrivateDeliveries` / `deliverToHumanInbox` → 公开气泡 `saveBubbles`。
   - 错误但有部分输出 → 剥离私有块保存公开部分后**暂停**（工具可能已执行，不自动重跑）。
   - 修复轮：`publicDeliverable` 缺公开文本或 `waitForHuman` 缺问号 → 一次无工具 repair。
6. **收尾**：`journal.finish(...)`；`participationOnly` 轮发"应到 X 实到 Y"系统提示。
7. **重启恢复**：`runtime.recoverGroupWorkflows()` 重放 `running` 工作流：存在不确定的 running reply 步骤/附件任务/花名册已变 → 置 `paused` 提示人工检查；否则 `resume=workflow` 重新进入（journal 重放已完成步骤）。

## 5. 决策服务 — [groupDecision.ts](../../src/main/groupDecision.ts)

`GroupDecisionService`，三种后端：

| 方法 | 说明 |
| --- | --- |
| `decide(settings, provider, group, context, signal)` | 入口；失败计数熔断（401/403 冷却 300s，连续 3 次失败冷却 60s） |
| `plan(provider, model, …)` | chat/completions 路径：严格 JSON Schema（tasks DAG、mode 枚举），2 次纠正机会；`mode==='none'` 首轮需二次 `needsReply` 审计；lightweight 先走 `participationPrompt`/`completionReviewPrompt` 快速通道 |
| `jev(...)` | System One/Jev 结构化判断协议（`{model, state, questions}` POST 到 `.../systemone`，choice/noul 题型，route 枚举 none/single/ordered/parallel/followup/plan）。**置信度门槛**：choice 概率 ∈ [0.8,1]；noul ≥0.8 选中/≤0.2 排除；不确定抛 `DecisionEscalation`（携带 `leaderMemberId` + `routeHint`，交回 leader 规划） |
| `recover(...)` | 失败槽恢复决策（仅 skip/replace/pause） |
| `complete(...)` | 底层补全（Anthropic/OpenAI 兼容；OpenRouter `openRouterReasoning` 关闭可选 reasoning 防吃光 8192 输出预算） |

云端决策代理：`CLOUD_DECISION_PROVIDER_ID = 'douchat:cloud'`（[shared/groupDecision.ts](../../src/shared/groupDecision.ts)，含 `DecisionSettings`、`decisionProtocol(model)` 按 `jev-*` 模型名切协议）。

## 6. 任务图 — [bot/groupTasks.ts](../../src/shared/bot/groupTasks.ts)

`GroupTask{ id, memberId, instruction, dependsOn, expectedOutput, requiredCapabilities, publicDeliverable }`。

- `validateGroupTasks`：1–32 节点、DAG 无环、deny 权能拒绝。
- `executeGroupTaskGraph`：事件驱动 DAG —— 并发 4、同成员节点串行、完成即解锁子节点、失败即停止但排空在飞任务。
- `groupTaskEvidence`：确定性任务证据（只有公开输出）。

## 7. 健康与隔离 — [groupHealth.ts](../../src/main/groupHealth.ts)

- `MemberHealth{ fingerprint, status: healthy|unavailable|unknown, latencyMs, failures, … }`；`refreshGroupHealth`：有界批探测（并发 8、总预算 6s、默认间隔 300s）；配置指纹变化重置 unknown。
- `rankGroupMembers(members, health, task, preferred?)`：可用性 ±1000 → 任务相关度（`Intl.Segmenter` 分词匹配 profile，封顶 60）→ 延迟对数惩罚 → preferred +3 → activeTasks 惩罚。**仅决定"先问谁当策略执行者"，选举和派工仍由决策完成**。
- **隔离（quarantine）**：实现于 `runGroupConversation` 闭包 —— 加入 unavailable 集合、`onUnavailable` 系统通知、重算 lead。
- **替补 lead**：`DecisionEscalation.leaderMemberId` → 回退路径首选该成员；`recoveryAction='replace'` 校验权能权限。

## 8. 工作流日志 — [groupWorkflow.ts](../../src/main/groupWorkflow.ts) + [shared/groupWorkflow.ts](../../src/shared/groupWorkflow.ts)

- `GroupWorkflowJournal.call(key, kind: 'decision'|'reply', execute)`：已完成步骤直接 `structuredClone` 重放；`running` 的 reply 步骤抛错要求人工检查；reply 步骤 15s 心跳证明执行器存活。`progress(key)`（工具进度脉冲）/ `finish(status)`（running/waiting/completed/paused/cancelled）。
- shared 侧：`GroupWorkflow` 类型、`workflowView`（UI 投影）、`decisionSlot(context, version)`（因果幂等决策槽 ID）。

## 9. 群游戏 — [groupGames.ts](../../src/main/groupGames.ts) + [shared/groupGame.ts](../../src/shared/groupGame.ts)

- 玩法：`undercover`（谁是卧底）与 `werewolf`（狼人杀，含狼人私聊频道 `wolfDiscuss`、预言家 `seer`）。
- 状态机在 shared（纯函数）：`createGame / nextGameTurn / applyGameAction / skipGameTurn / gamePlayerView`（按身份投影，秘密不越权）。
- `GroupGames` 类：`start`（仅本地群、仅云端 Agent —— 本地 CLI 因无法隔离工具被 `GameRuleError` 拒绝；可含 human 玩家）、`act`（人类动作）、`control(pause|resume|cancel)`、`recover()`、`pump/execute`（预算 300 slot，每 slot 3 次尝试、2 次无效输出即 skip；`revision` CAS 经 `store.commitGame`）。
- 玩家回合提示：只输出一个 JSON（`speak: {"text"}` / `vote|inspect: {"targetId"}`），无工具，150 字以内。

## 10. 群记忆 — [groupMemoryStore.ts](../../src/main/groupMemoryStore.ts)

SQLite 表 `group_memories(userId, groupId, data)`；`read/save/remember`。关键隔离：`audience(groupId)` 区分 `internal` 与 `external:<remoteRoomId|socialRoom.id|groupId>` —— 群被链接到共享房间后旧内部记忆不泄露。`remember` 强约束：`evidence` 必须出现在当前人类消息原文中；键 `sha256([speaker.id, key])` 不同说话人不冲突。

## 11. 辅助协议（bot/ 其余）

| 文件 | 关键导出 | 功能 |
| --- | --- | --- |
| [bot/groupParticipation.ts](../../src/shared/bot/groupParticipation.ts) | `participationPrompt/Decision`、`completionReviewPrompt/Decision` | 轻量快速通道：全员报数式贡献判定、完成审查（complete/waiting/continue） |
| [bot/messages.ts](../../src/shared/bot/messages.ts) | `BOT_MESSAGE_BREAK`、`splitBotReply`、`botReplyPrompt` | 气泡拆分与 IM 风格契约 |
| [bot/identity.ts](../../src/shared/bot/identity.ts) | `botIdentityPrompt` | bot 档案（name/description/labels）注入为"当前配置"指令 |
| [bot/contacts.ts](../../src/shared/bot/contacts.ts) | `contactSections/compareContacts/matchesContactQuery` | 拼音 A–Z 分桶（`Intl.Collator('zh-Hans-u-co-pinyin')`） |
| [bot/order.ts](../../src/shared/bot/order.ts) | `compareConversationActivity/Organization` | 会话列表排序（运行中置顶） |
| [bot/errors.ts](../../src/shared/bot/errors.ts) | `summarizeRuntimeError`、`isRetryableRuntimeError`、`isDouchatCreditError` | CLI 错误堆叠摘要 / 可重试判定 / 点数错误识别 |
| [bot/greeting.ts](../../src/shared/bot/greeting.ts) | `botGreetingPrompt` | 新话题主动问候 prompt |
| [bot/preview.ts](../../src/shared/bot/preview.ts) | `markdownPreview/latestAssistantPreview` | 收件箱单行预览 |
