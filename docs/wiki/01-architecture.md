# 01 · 项目概览与整体架构

> 源码版本：v0.1.23。相关文档：[Wiki 首页](README.md) · [主进程模块](02-main-process.md) · [运行时](03-agent-runtime.md)

## 1. 一句话架构

Douchat 是 **local-first** 的三进程 Electron 应用：**主进程**持有全部业务（认证、SQLite 存储、Agent 运行时、群组编排、IM/社交轮询），**渲染进程**是无 Node 权限的沙箱 React UI，两者只通过 preload 暴露的 `window.douchat` IPC 契约通信，数据以**单一全量快照 `AppSnapshot`** 推送同步。

## 2. 进程与目录分层

```text
┌─────────────────────────────────────────────────────────────────┐
│ Renderer（src/renderer，React 19，sandbox + contextIsolation）   │
│  App.tsx ─ 聊天/联系人/设置 三视图 · 状态源=AppSnapshot          │
└──────────────▲──────────────────────────────────────────────────┘
               │ window.douchat.*（invoke/send/on）
┌──────────────┴──────────────────────────────────────────────────┐
│ Preload（src/preload/index.ts）contextBridge 单一命名空间       │
└──────────────▲──────────────────────────────────────────────────┘
               │ ipcMain.handle('douchat:*') / webContents.send
┌──────────────┴──────────────────────────────────────────────────┐
│ Main（src/main/index.ts 装配，约 1350 行）                       │
│                                                                 │
│  DesktopAuth ──token──▶ DouchatRuntime ◀── RoutineScheduler     │
│  (safeStorage)             │  ├─ gateway.ts（douchat.ai /v1）   │
│  DouchatStore(SQLite) ◀────┘  ├─ customModels.ts（自定义端点）  │
│  LocalComputerProvider        ├─ desktopAgentExecutor（本地CLI）│
│  IMChannelManager             ├─ GroupGames / GroupDecision…    │
│  SocialClient                 └─ 工具：skill/computer/connany…  │
│  ConnanyManager / EmailConnectorManager                         │
│  DesktopUpdater · DiagnosticLog                                 │
└─────────────────────────────────────────────────────────────────┘
               ▲ 纯函数/类型复用
┌──────────────┴──────────────────────────────────────────────────┐
│ Shared（src/shared）：types.ts 数据契约 · bot/ 群组协作协议     │
│  （同时被 main 执行编排、被 renderer 展示复用）                 │
└─────────────────────────────────────────────────────────────────┘
```

各层职责：

| 层 | 目录 | 职责 |
| --- | --- | --- |
| 主进程 | `src/main/` | 应用生命周期、窗口管理、`douchat://` 深链、IPC 注册；全部业务逻辑与本地状态权威 |
| Preload | `src/preload/index.ts` | 唯一的 IPC 桥，暴露类型化的 `window.douchat`（契约定义在 `src/shared/types.ts` 的 `DouchatApi`） |
| 渲染层 | `src/renderer/` | React UI；无本地状态库，一切派生自快照 |
| 共享层 | `src/shared/` | 跨进程类型 + 可测试纯函数（提及解析、私信协议、任务图 DAG、游戏状态机、拼音分桶等） |

## 3. 关键设计决策

1. **单一快照 + 全量推送**：任何主进程变化 → `runtime.snapshot()` 聚合成 `AppSnapshot` → `broadcast()`（后台变化 100ms 节流）→ 所有窗口 `douchat:snapshot`。渲染层不做增量 diff 状态管理；大对象（技能资源文件）在快照中省略并按需拉取（`filesOmitted` + `getAgentSkill`）。
2. **密钥不出主进程**：登录 token 经 `safeStorage` 加密存于 `auth.json`；IM/邮件凭据同样加密落盘；渲染层只见布尔值（如 `EndpointSettings.hasApiKey`）。
3. **乐观并发**：`UpdateAgentInput` / `UpdateConversationInput` 等携带 `expectedRevision`，群游戏与工作流用 revision CAS 防并发覆盖。
4. **账号隔离**：本地数据按 `ownerId`（账号）划分域；`AccountContext{ownerId, sessionId}` 令牌防止回合执行中途切换账号；切换账号会停会话、关窗口、重建调度。
5. **失败安全（fail-closed）**：被中断的带工具群回复**绝不自动重跑**（副作用可能已发生），而是置 `paused` 等人工处理；权限默认 `ask`；云端 token 401 立即失效会话。

## 4. 一条消息的完整数据流

以"用户在群聊发一条消息"为例（本地群，云端 Agent）：

```text
React ChatPane
  → messageQueue.enqueue（每会话串行，等上一轮 activity 结束）
  → window.douchat.sendMessage(conversationId, text, images, files, mentions)
  → ipcMain.handle('douchat:send-message') → runtime.sendMessage()
      ├─ 共享群(remoteRoomId) → humanSender → SocialClient.sendMessage → 云端路由（不进本地编排）
      └─ 本地群 → performSendMessage
            ├─ resolveMentionedMembers / addressesEveryone（@提及解析，shared/bot/mentions.ts）
            ├─ store.addMessage（用户消息落库）+ store.createRun(trigger:'chat')
            └─ runGroupTurn
                  ├─ groupRoutingProfile：从成员 IDENTITY/SOUL/TOOLS.md 提取路由画像
                  ├─ directGroupDecision：行首 @单人 → 直通 single 路由
                  ├─ 否则 refreshGroupHealth（探测）→ 决策（Jev 服务 / leader 规划 / 云决策）
                  │     → GroupDecision{mode: none|single|parallel|sequential, tasks(DAG), …}
                  ├─ runGroupConversation（shared/bot/group.ts 执行器主循环）
                  │     └─ 每个成员回合 → runtime.runReply → performReply
                  │           ├─ 云端 Agent：session() → pi-agent-core Agent.prompt()
                  │           │    → models.streamSimple → gatewayProvider
                  │           │    → POST https://douchat.ai/v1/chat/completions（Bearer=登录 token）
                  │           └─ 本地 Agent：desktopAgentExecutor.run → CLI 进程（见 04）
                  ├─ 成员回复解析：splitBotReply 拆气泡；[[private:ID]] 私信投递；[[a2a:ID]] A2A 委派
                  └─ saveBubbles → store.addMessage
  → emit() → scheduleBroadcast（100ms 节流）→ broadcast('douchat:snapshot')
  → 渲染层 setSnapshot → 派生更新
```

单聊路径更简单：`runDirectTurn` → `runReply(direct)` → A2A 信封解析 → 有界综合回合。详见 [03 · Agent 运行时](03-agent-runtime.md)。

## 5. 模块分层图（主进程内部）

```text
index.ts（装配层：构造并接线所有模块，注册全部 IPC）
 ├─ DouchatStore（store.ts，node:sqlite 单文件 douchat.db）
 │   ├─ UserMemoryStore / GroupMemoryStore（记忆，含 Markdown 文件镜像）
 │   └─ 大量一次性迁移与 backfill（见 02）
 ├─ DouchatRuntime（runtime.ts，约 4300 行，应用大脑）
 │   ├─ gateway.ts（云模型 Provider） / customModels.ts（自定义 Provider）
 │   ├─ desktopAgentExecutor → localAgentRuntime.ts（本地 CLI，见 04）
 │   ├─ GroupGames / GroupDecisionService / GroupWorkflowJournal / groupHealth（见 05）
 │   ├─ LocalComputerProvider（computer.ts 浏览器/文件工具）
 │   └─ connectors：EmailConnectorManager + ConnanyManager（createTools 注入）
 ├─ RoutineScheduler（scheduler.ts）── runtime.runRoutine
 ├─ DesktopAuth（desktopAuth.ts + authProtocol.ts，PKCE + safeStorage）
 ├─ CustomModelStore（customModels.ts，key 加密）
 ├─ IMChannelManager（imChannels.ts，见 06）
 ├─ SocialClient（social.ts，见 06）
 ├─ DesktopUpdater（updater.ts，electron-updater）
 └─ DiagnosticLog（diagnostics.ts，JSONL 日志）
```

依赖注入方向：`index.ts` 构造各模块并互相接线（auth token → runtime/gateway；`runtime.setRoutineCreator` → scheduler；`runtime.setHumanSender` → social），形成少量受控的循环依赖。

## 6. 安全模型

| 维度 | 机制 | 代码位置 |
| --- | --- | --- |
| 渲染层沙箱 | `sandbox: true` / `contextIsolation: true` / `nodeIntegration: false`；CSP 禁外链脚本 | `src/main/index.ts` createWindow、`src/renderer/index.html` |
| IPC 来源校验 | 多数 handler 用 `isDouchatRenderer(event.sender)` 校验 sender 属于本应用窗口 | `src/main/index.ts` |
| 登录 token | `safeStorage.encryptString` 存 `auth.json`（0600，临时文件+rename 原子写）；token 从不进渲染层与 IPC 契约 | `src/main/desktopAuth.ts` |
| 打包版服务源锁定 | 打包构建忽略 `DOUCHAT_SERVICE_URL` 环境变量，防 shell 劫持 token | `src/main/authProtocol.ts` `normalizeWebAppUrl` |
| 本地文件工具 | 白名单根目录仅 Downloads/Desktop/Documents + 会话授权文件夹；`realpathSync` 防符号链接逃逸；移动不覆盖、不提供删除 | `src/main/computer.ts` `resolveAllowedPath` |
| 附件访问 | 渲染层经 `getAttachmentData(attachmentId)` 按需取字节；`douchat-file:` 协议经 `ownedDocumentPath` 校验 owner | `src/main/store.ts` |
| Agent 权限 | 9 类敏感能力（filesRead/filesWrite/network/browserControl/accountRead/accountWrite/automation/otherTools…）allow/ask/deny 三态，决策在 owner 主进程 `AgentPermissionBroker` | `src/main/agentPermissions.ts` |
| CLI 沙箱保留 | 本地 CLI 继续执行自己的登录与审批（Codex `sandbox workspace-write`、Claude 工具白名单），Douchat 不绕过 | `src/main/localAgentConnection.ts` |
| 诊断脱敏 | `redact()` 清除 URL 查询、Bearer、api_key、`sk-` 前缀密钥 | `src/main/diagnostics.ts` |
| 技能安全 | 技能从不执行脚本、不改权限；路径校验含 Windows 保留名 | `src/main/skillInstallation.ts`、`src/shared/agentCustomization.ts` |

## 7. 技术栈与版本

| 类别 | 选型 | 版本（package.json） |
| --- | --- | --- |
| 桌面框架 | Electron | 43.7.3 |
| 构建 | electron-vite / vite / TypeScript | 5.x / 7.x / 5.9 |
| UI | React + lucide-react + streamdown（Markdown 流式渲染） | 19.x |
| Agent 引擎 | `@earendil-works/pi-agent-core`（Agent 循环）+ `@earendil-works/pi-ai`（模型抽象） | 0.84.x |
| 存储 | `node:sqlite`（`DatabaseSync`，WAL） | Node 内置 |
| 测试 | Vitest（`vitest run src`，测试文件与源码同目录 `*.test.ts`） | 3.x |
| IM/邮件 | `@larksuiteoapi/node-sdk`（飞书 WS）、`imapflow`+`mailparser`+`nodemailer` | — |
| 打包/更新 | electron-builder + electron-updater（generic provider: cdn.douchat.ai） | 26.x / 6.x |
| 压缩/归档 | fflate / tar-stream / yauzl / yaml | — |

## 8. 深入阅读

- 认证与存储细节 → [02 · 主进程核心模块](02-main-process.md)
- 回合执行、超时/重试策略 → [03 · Agent 运行时](03-agent-runtime.md)
- 本地 CLI 如何被驱动 → [04 · 本地 Agent 子系统](04-local-agents.md)
- 群组决策与任务图 → [05 · 群组协作子系统](05-group-collaboration.md)
- 设计文档：`docs/local-first-architecture.md`、`docs/agent-permissions.md`
