# 10 · 依赖关系

> 依据：[package.json](../../package.json)（v0.1.23）与源码 import 关系。相关：[01 架构](01-architecture.md)

## 1. 运行时依赖（dependencies）

| 依赖 | 版本 | 用途 | 使用位置 |
| --- | --- | --- | --- |
| `@earendil-works/pi-agent-core` | ^0.84.2 | Agent 回合引擎（`Agent`、`AgentTool`） | [runtime.ts](../../src/main/runtime.ts) |
| `@earendil-works/pi-ai` | ^0.84.2 | 模型抽象层（Provider/Model 注册表、openai-completions / anthropic-messages 流） | runtime.ts、gateway.ts、customModels.ts |
| `@larksuiteoapi/node-sdk` | ^1.74.0 | 飞书 WebSocket 长连接 | [imChannels.ts](../../src/main/imChannels.ts) |
| `@sinclair/typebox` | ^0.34.41 | JSON Schema 构造（群决策结构化输出） | [groupDecision.ts](../../src/main/groupDecision.ts) |
| `dotenv` | ^17.2.1 | `.env` 加载 | 主进程入口 |
| `electron-updater` | ^6.8.9 | 自动更新 | [updater.ts](../../src/main/updater.ts) |
| `fflate` | 0.8.3 | ZIP（技能上传/下载、Agent 档案） | renderer SkillDetailDialog、skillArchive、agentArchive |
| `highlight.js` | 11.11.1 | 代码高亮 | 渲染层 |
| `imapflow` / `mailparser` / `nodemailer` | — | 邮件连接器 IMAP/解析/SMTP | [emailConnector.ts](../../src/main/emailConnector.ts) |
| `mdast-util-from-markdown` / `mdast-util-gfm` / `micromark-extension-gfm` | — | Markdown/GFM 解析 | [imFormatting.ts](../../src/main/imFormatting.ts)、MessageMarkdown |
| `qrcode.react` | ^4.2.0 | 群邀请二维码 | GroupInviteDialog |
| `tar-stream` / `yauzl` | — | TAR.GZ / ZIP 解包 | archiveFiles、skillInstallation |
| `yaml` | 2.9.1 | YAML 解析（SKILL.md frontmatter、第三方档案、Gemini 图片策略 TOML 相关） | skillArchive、agentArchive、localAgentRuntime |

## 2. 开发依赖（devDependencies，要点）

| 依赖 | 用途 |
| --- | --- |
| `electron` 43.7.3 + `electron-vite` 5 + `vite` 7 | 框架与构建（main/preload/renderer 三目标） |
| `react` / `react-dom` 19 + `@types/react*` | UI |
| `streamdown` + `@streamdown/cjk` + `@streamdown/mermaid` | 流式 Markdown 渲染 |
| `lucide-react` | 图标 |
| `dompurify` + `jsdom` | HTML 消毒与测试环境 |
| `typescript` 5.9 | 类型检查（`tsconfig.node.json` / `tsconfig.web.json` 两个项目） |
| `vitest` 3 | 测试（`vitest run src`，与源码同目录 `*.test.ts(x)`） |
| `electron-builder` 26 | 打包（dmg/zip/nsis/AppImage/deb） |

## 3. 模块间依赖图（主进程）

```text
index.ts ─────────────────────────────────────────────────────┐
  │ 构造/接线                                                  │
  ▼                                                            │
DouchatStore ──组合──▶ UserMemoryStore / GroupMemoryStore      │
  ▲              └──── profileFiles（SOUL.md 物化）            │
  │                                                            │
DouchatRuntime ──▶ store（全部读写）                            │
  │  ├─▶ gateway.ts ──▶ pi-ai Provider（douchat.ai/v1）        │
  │  ├─▶ customModels.ts ──▶ pi-ai Provider（用户端点）         │
  │  ├─▶ desktopAgentExecutor ──▶ localAgentRuntime            │
  │  │        ├─▶ localAgents（目录/校验）                      │
  │  │        ├─▶ shellPath（登录 shell 环境）                  │
  │  │        ├─▶ localWorkspaces（目录+会话绑定）              │
  │  │        ├─▶ localProcessBudget（进程预算 8）              │
  │  │        ├─▶ localAgentConnection（codex/claude 私有协议） │
  │  │        │      └─▶ codexComputerUse                       │
  │  │        ├─▶ grokStream / geminiStream（流式解析）         │
  │  │        └─▶ remoteTransport / remoteScript（SSH）        │
  │  ├─▶ localSkillBridge / remoteSkillBridge（技能桥）        │
  │  ├─▶ agentPermissions.AgentPermissionBroker（审批）        │
  │  ├─▶ GroupGames / GroupDecisionService / GroupWorkflowJournal
  │  │        └─▶ shared/bot/*（决策/任务图/私信/A2A 纯函数）   │
  │  ├─▶ internalMemory / userMemoryStore（记忆注入）          │
  │  ├─▶ computer.LocalComputerProvider（浏览器/文件工具）     │
  │  └─▶ connectors = emailConnectors + connany（createTools） │
  │                                                            │
RoutineScheduler ──▶ store + runtime.runRoutine ◀─ setRoutineCreator
DesktopAuth ──▶ authProtocol + safeStorage ──token──▶ runtime/gateway
SocialClient ──▶ runtime.setHumanSender / executeSocialTask ◀──┘
IMChannelManager ──▶ replyToIM ──▶ runtime.sendIMMessage
DesktopUpdater / DiagnosticLog（横切）
```

**渲染层依赖**：`App.tsx` → components/* → `window.douchat`（preload 契约）+ `src/shared` 纯函数（排序/分桶/提及/预览）。渲染层**不直接 import 主进程任何模块**。

**shared 依赖方向**：`shared/bot/*`、`shared/*` 为叶子（不依赖 main/renderer）；`shared/types.ts` 被三方共同引用，是 IPC 与快照的唯一契约源。

## 4. 对 pi-agent-core / pi-ai 的分层

```text
DouchatRuntime（业务编排：会话缓存、记忆、权限、A2A、群决策、快照）
        │ streamFn = models.streamSimple(...)
        ▼
pi-ai（模型抽象：Provider 注册表 / openai-completions / anthropic-messages）
        │ gateway.ts、customModels.ts 把端点适配为 Provider 注入
        ▼
pi-agent-core（Agent 循环：prompt → 流式补全 → 工具执行 → 回填 → 继续）
```

- 云端/自定义模型 Agent 走 pi-agent-core `Agent`；本地 CLI Agent 不经过它，由 `AgentExecutor` 直接驱动 CLI 进程（但共享 runtime 的编排、权限与快照管线）。
- `AgentTool` 接口同时用于 pi-agent-core 工具与技能桥（`validateToolArguments` 校验后执行）。

## 5. 外部服务依赖

| 服务 | 协议/端点 | 用途 |
| --- | --- | --- |
| Douchat 服务（douchat.ai） | `POST /api/desktop-auth/token`、`GET /api/desktop-auth/me|usage|built-in-agents`、`/api/desktop-auth/social*`、`/api/desktop-auth/connectors` | 登录、资料、额度、社交、Connany |
| Douchat Chat API | `GET {origin}/v1/models`、`POST {origin}/v1/chat/completions` | 云模型目录与流式补全 |
| 云端决策代理 | `POST …/systemone`（Jev 协议） | 群决策（可选） |
| Telegram Bot API | `api.telegram.org/bot<token>` | IM 通道（本地直连） |
| 飞书开放平台 | WebSocket（node-sdk） | IM 通道（本地直连） |
| 微信 iLink | `ilinkai.weixin.qq.com` | IM 通道（本地直连，扫码） |
| skills.sh | HTTP | 技能搜索元数据 |
| npm registry / GitHub Releases / PyPI / x.ai | HTTP 只读 | CLI 版本检查 |
| cdn.douchat.ai | electron-updater generic provider | 自动更新 |
| OpenAI/Anthropic 兼容端点 | HTTP | 自定义模型 / 群决策 provider |
