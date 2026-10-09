# 07 · 渲染层（React UI）

> 入口：[src/renderer/src/main.tsx](../../src/renderer/src/main.tsx) → [App.tsx](../../src/renderer/src/App.tsx)（约 750 行）。相关：[08 IPC 契约](08-ipc-preload.md) · [09 数据模型](09-shared-data-models.md)

## 1. 入口与骨架

- **index.html**：单页 `<div id="root">`；内置严格 CSP（`default-src 'self'; script-src 'self'; … img-src 'self' data: https:`）。
- **main.tsx**：`import './diagnostics'`（先装全局 error/unhandledrejection 上报钩子）→ `<StrictMode><ChatErrorBoundary root><App/></ChatErrorBoundary></StrictMode>`；把 `window.douchat.platform` 写入 `document.documentElement.dataset.platform`（Windows 禁 backdrop-filter）。
- **App.tsx**：按 URL 分流 —— `?artifact=<id>` 渲染独立代码预览窗 `CodeArtifactWindow`；否则 `WorkspaceApp`。

### WorkspaceApp 主要 state

| State | 作用 |
| --- | --- |
| `snapshot: AppSnapshot \| null` | **全局唯一数据源**（agents/conversations/messages/activity/routines/runs/models…），`getSnapshot` 拉取 + `onSnapshot` 订阅推送 |
| `authState: DesktopAuthState` | 登录态（checking/signed-out/waiting/error/signed-in） |
| `activeId` / `topic` | 当前会话与话题（detached 窗口从 `?conversation=` 取） |
| `view: 'chats' \| 'contacts'` | AppRail 视图切换 |
| `dialog: Dialog`（判别联合） | 模态路由：agent-permissions/self-profile/add-friend/friend-profile/member-profile/im-channels/local-model/bot/add-members/remove-members/group/endpoint |
| `settingsOpen` / `settingsTab` | 设置面板 10 个标签页 |
| `messageQueue` + `queuedMessages` | 会话级排队发消息 |
| `socialSnapshot` | 社交快照（3s 轮询） |
| `localAgents: LocalAgent[]` | 启动时 `detectLocalAgents()`（20s 超时） |
| `showInspector` / `inspectorAgentId` | 右侧详情栏 |

### 布局

```text
AppRail（导航：聊天/联系人/设置，未读角标）
├─ view==='contacts'：ContactList + ContactCard / SocialWorkspace
└─ view==='chats'：BotInbox（会话列表）
                    + ChatPane（聊天主面板）
                    + InspectorRail（会话详情侧栏，Esc 关闭）
DialogErrorBoundary 包裹全部模态 → ToastNotice（2.6s 自动消失）
```

### 消息发送/接收

- **发送**：`send()` → `messageQueue.enqueue(conversationId, 摘要, 实际调用, ready条件)` —— ready = 该会话无 `activity`（上一轮结束）；队列每会话串行；执行时校验 owner 与话题未切换；最终 `window.douchat.sendMessage(id, text, images, files, mentions)`；失败回队首可重试。
- **接收**：主进程 `broadcast('douchat:snapshot')` → `setSnapshot` → `useMemo` 按当前会话/话题派生 → 渲染；打开且聚焦的会话自动 `markConversationRead`。
- 其他：IME 组合态拦截回车（[ime.ts](../../src/renderer/src/ime.ts)）；Cmd/Ctrl+F 聚焦搜索；点数错误自动打开"额度"设置页；`withAccountIdentity` 保证 UI 身份以账号数据为准。

## 2. 组件清单（src/renderer/src/components/）

### 聊天类

| 组件 | 职责 |
| --- | --- |
| [ChatPane.tsx](../../src/renderer/src/components/ChatPane.tsx) | 聊天主面板：消息流（送达/私信/工具 action 回执）、输入框（@补全、粘贴图片 ≤4 张/8MB、附件）、图片画廊、排队消息、活动状态与本地进度、停止/清空/重置上下文 |
| [MessageMarkdown.tsx](../../src/renderer/src/components/MessageMarkdown.tsx) | Streamdown Markdown 渲染（CJK 断行 + Mermaid）；`douchat-file:` 链接渲染为"打开本地文件"按钮 |
| [CodeArtifact.tsx](../../src/renderer/src/components/CodeArtifact.tsx) | 消息内文件级代码块（≥18 行或 ≥1200 字节或带 filename）→ 卡片，点击独立窗口打开 |
| [CodeArtifactWindow.tsx](../../src/renderer/src/components/CodeArtifactWindow.tsx) | `?artifact=<id>` 独立预览窗宿主 |
| [BotInbox.tsx](../../src/renderer/src/components/BotInbox.tsx) | 会话列表：搜索、筛选（all/unread/direct/group）、新建菜单、右键菜单（置顶/免打扰/编辑/删除/独立窗口）、bot 摘要预览 |
| [ChatErrorBoundary.tsx](../../src/renderer/src/components/ChatErrorBoundary.tsx) | 聊天区与全局 React 错误边界 |

### 联系人类

| 组件 | 职责 |
| --- | --- |
| [ContactList.tsx](../../src/renderer/src/components/ContactList.tsx) | 内置/星标/自定义 bot（拼音 A–Z 分组）/群/好友，带搜索 |
| [ContactCard.tsx](../../src/renderer/src/components/ContactCard.tsx) | 联系人详情卡（好友资料、bot 模型标签、群资料）；`SelfProfileCard` |
| [MemberProfilePopover.tsx](../../src/renderer/src/components/MemberProfilePopover.tsx) | 锚定式成员资料弹层 |
| [ContactKindBadge.tsx](../../src/renderer/src/components/ContactKindBadge.tsx) | 联系人种类徽标（本地/远程/云端） |
| [InvitationAvatar.tsx](../../src/renderer/src/components/InvitationAvatar.tsx) | 群邀请海报 SVG 生成 |
| [AddFriendModal.tsx](../../src/renderer/src/components/AddFriendModal.tsx) | 按邮箱查找/发送好友请求 |
| [SocialWorkspace.tsx](../../src/renderer/src/components/SocialWorkspace.tsx) | 社交工作区：好友直聊/群/申请（2s 轮询 messages）、代表自己的 Agent 发言、建群/邀请 |

### 设置类

| 组件 | 职责 |
| --- | --- |
| [SettingsPanel.tsx](../../src/renderer/src/components/SettingsPanel.tsx) | 全局设置（NativeDialog 980×720），10 标签：profile/general/usage/automation/models/connectors/agents/scheduling/memory/about；含本地 Agent 扫描列表 |
| [AgentSettingsDialog.tsx](../../src/renderer/src/components/AgentSettingsDialog.tsx) | 单 Agent 设置（8 标签：profile/customize/models/skills/permissions/channels/memory/advanced），dirty 追踪 + 统一保存锁 |
| [AgentCustomization.tsx](../../src/renderer/src/components/AgentCustomization.tsx) | `AgentFilesPanel`（编辑 SOUL/IDENTITY/TOOLS/BOOTSTRAP/AGENTS/HEARTBEAT.md + 只读 USER/MEMORY.md）与 `AgentSkillsPanel` |
| [AgentPermissions.tsx](../../src/renderer/src/components/AgentPermissions.tsx) | `AgentPermissionsDialog`（三态设置）与 `AgentPermissionPrompt`（运行中审批：Bash、桌面应用、连接器授权） |
| [CustomModelSettings.tsx](../../src/renderer/src/components/CustomModelSettings.tsx) | 自定义服务商管理（预设 Anthropic/OpenAI/OpenRouter/TokenDance/DeepSeek，OAuth 或 API key，默认模型，连通测试） |
| [CustomModelSelection.tsx](../../src/renderer/src/components/CustomModelSelection.tsx) | 云 vs 自定义模型两级选择器 |
| [LocalModelDialog.tsx](../../src/renderer/src/components/LocalModelDialog.tsx) | 本地 CLI 模型选择或云/自定义模型 + 思考等级 |
| [LocalAgentEditor.tsx](../../src/renderer/src/components/LocalAgentEditor.tsx) | 新增/编辑自定义本地 Agent（本机命令或 SSH 远程，host 来自 `listSshHosts`） |
| [LocalAgentSelect.tsx](../../src/renderer/src/components/LocalAgentSelect.tsx) | 本地 Agent 运行时下拉（BotModal 内用） |
| [DecisionSettings.tsx](../../src/renderer/src/components/DecisionSettings.tsx) | 群决策设置：默认（leader）vs 云端决策模型，健康检查间隔，连通测试 |
| [SchedulingSettings.tsx](../../src/renderer/src/components/SchedulingSettings.tsx) | 调度设置外壳（内嵌 DecisionSettings） |
| [UserMemoryPanel.tsx](../../src/renderer/src/components/UserMemoryPanel.tsx) | 记忆编辑：关于我/Agent 专属/群记忆三模式；事实条目、按日期历史、自动记忆开关 |
| [AgentArchivePanel.tsx](../../src/renderer/src/components/AgentArchivePanel.tsx) | Agent 配置 ZIP/TAR.GZ 导入（预览/警告/确认替换）与导出 |
| [ConnanyPanel.tsx](../../src/renderer/src/components/ConnanyPanel.tsx) | Connany 连接器：列表、OAuth 会话轮询（5s）、资源授权、断开重连；错误码翻译产品文案 |
| [ConnectorsPanel.tsx](../../src/renderer/src/components/ConnectorsPanel.tsx) | 邮件（IMAP/SMTP）连接器管理与授权 |
| [ConversationWorkspaceSetting.tsx](../../src/renderer/src/components/ConversationWorkspaceSetting.tsx) | 会话工作区（本地文件夹）：仅"全员为本人 Agent 的私聊/群"可用 |
| [LoginScreen.tsx](../../src/renderer/src/components/LoginScreen.tsx) | 登录屏：浏览器 OAuth、语言切换 |

### 群组类

| 组件 | 职责 |
| --- | --- |
| [GroupInviteDialog.tsx](../../src/renderer/src/components/GroupInviteDialog.tsx) | 邀请链接生成/轮换、二维码海报（canvas → 复制/保存）、过期检测 |
| [dialogs.tsx](../../src/renderer/src/components/dialogs.tsx) | `BotModal`（创建/编辑 Agent：头像/名称/指令/标签/模型来源/启动参数）、`GroupModal`（建群/编辑，成员与 lead 选择）、`AddMembersModal`、`EndpointModal` |
| [GroupMemoryDialog.tsx](../../src/renderer/src/components/GroupMemoryDialog.tsx) | 群记忆对话框（封装 UserMemoryPanel） |

### 技能类

| 组件 | 职责 |
| --- | --- |
| [SkillUploadDialog.tsx](../../src/renderer/src/components/SkillUploadDialog.tsx) | 技能 ZIP 上传（≤64MB、须含 SKILL.md） |
| [SkillDetailDialog.tsx](../../src/renderer/src/components/SkillDetailDialog.tsx) | 技能详情：文件树、Markdown 预览、源码/图片切换、ZIP 下载（fflate） |
| [SkillSourceCode.tsx](../../src/renderer/src/components/SkillSourceCode.tsx) / [skillMarkdown.ts](../../src/renderer/src/components/skillMarkdown.ts) | 只读高亮视图 / remark 插件 |

### 通用基础设施

| 模块 | 职责 |
| --- | --- |
| [common.tsx](../../src/renderer/src/components/common.tsx) | 头像四级回退（图片→emoji→CLI logo→种子生成脸）、`agentDisplayName/conversationDisplayName`、`conversationMembers`、时间格式化、`SidebarResizer`、调色板 |
| [NativeDialog.tsx](../../src/renderer/src/components/NativeDialog.tsx) | 经 `resizeDialog` 同步主进程窗口尺寸的原生对话框容器 |
| [AgentDialogSurface.tsx](../../src/renderer/src/components/AgentDialogSurface.tsx) | 嵌入态对话框表面 + `EmbeddedAgentSettings` context |
| [DialogErrorBoundary.tsx](../../src/renderer/src/components/DialogErrorBoundary.tsx) | 对话框错误边界 |
| [RemoteMark.tsx](../../src/renderer/src/components/RemoteMark.tsx) | 远程 Agent 标记与全局 localAgents 注册表 |
| [SvgPreview.tsx](../../src/renderer/src/components/SvgPreview.tsx) | SVG 代码内联预览 |
| [AppRail.tsx](../../src/renderer/src/components/AppRail.tsx) / [InspectorRail.tsx](../../src/renderer/src/components/InspectorRail.tsx) | 左导航 / 右详情栏 |

## 3. 渲染层辅助模块（src/renderer/src/）

| 模块 | 职责 |
| --- | --- |
| [preferences.ts](../../src/renderer/src/preferences.ts) | localStorage `douchat.general`：语言/外观/字号；`usePreferences`（useSyncExternalStore）；`t()/tr()` 翻译（聚合 shared 的 groupText/runtimeText/gameText 词典） |
| [messageQueue.ts](../../src/renderer/src/messageQueue.ts) | `MessageQueue` 类：按会话串行的待发队列 |
| [accountIdentity.ts](../../src/renderer/src/accountIdentity.ts) | `withAccountIdentity(snapshot, user)`：以登录账号覆盖旧本地缓存 |
| [agentIcons.ts](../../src/renderer/src/agentIcons.ts) | `import.meta.glob` 导入 `assets/agents/*` 为 id→URL 映射 |
| [avatarFile.ts](../../src/renderer/src/avatarFile.ts) | `readAvatarFile()`：中心裁剪 256×256 PNG data URL |
| [diagnostics.ts](../../src/renderer/src/diagnostics.ts) | 渲染层诊断上报（≤15000 字符） |
| [generatedAvatar.tsx](../../src/renderer/src/generatedAvatar.tsx) | 种子确定性生成头像（108 调色板 + xorshift PRNG） |
| [ime.ts](../../src/renderer/src/ime.ts) | `isImeCommitEnter()`：输入法组合期 Enter |
| [sidebarWidth.ts](../../src/renderer/src/sidebarWidth.ts) | 侧栏宽度（220–360px，默认 244），CSS 变量 |
| [speechRecognition.ts](../../src/renderer/src/speechRecognition.ts) | Web Speech API 垫片（语音输入当前隐藏） |

## 4. 状态管理哲学

1. **无客户端状态库**：一切派生自 `AppSnapshot` 全量快照（mutation IPC 直接返回新快照做同步路径）。
2. **密钥不落渲染层**：只见 `hasApiKey` 布尔值。
3. **附件按需取**：本地文件路径不暴露，字节经 `getAttachmentData(attachmentId)` 取。
4. **共享纯函数双用**：排序/分桶/提及解析等同时被主进程与渲染层引用，行为由 `*.test.ts` 钉住。
