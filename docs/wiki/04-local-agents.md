# 04 · 本地 Agent 子系统

> 把本机已安装的 AI CLI（Claude Code、Codex、Gemini 等 11 种）作为聊天联系人驱动。相关设计文档：[docs/local-agent-runtime.md](../local-agent-runtime.md) · [docs/local-workspaces.md](../local-workspaces.md) · [docs/local-agents 相关](../README.md)

## 1. 四层结构

```text
注册与检测层   localAgents.ts（目录/自定义注册表/检测）· shellPath.ts（登录 shell PATH）·
               localAgentUpdates.ts / localAgentMaintenance.ts（版本与升级）
执行层         shared/agentExecutor.ts（AgentExecutor 接口）→ desktopAgentExecutor.ts（默认实现）
               → localAgentRuntime.ts（一次性适配 + 持久连接池）→ localAgentConnection.ts（codex/claude 私有协议）
资源管理层     localWorkspaces.ts（工作目录与原生会话绑定）· localProcessBudget.ts（进程预算=8）
宿主能力层     localSkillBridge.ts（localhost 工具桥）· skillTools/skillInstallation/skillArchive ·
               workspaceTools.ts / computer.ts · agentPermissions.ts · agentCustomization / agentArchive
```

## 2. CLI 目录与检测 — [localAgents.ts](../../src/main/localAgents.ts)

`localAgentCatalog` 内置 11 个 CLI：`claude`（Claude Code）、`codex`、`gemini`、`grok`（Grok Build）、`openclaw`、`hermes`、`opencode`、`cursor`（命令 `cursor-agent`）、`kimi`、`omp`、`fastclaw`。

| 导出 | 说明 |
| --- | --- |
| `configureLocalAgentRegistry(userDataPath)` | 自定义注册表 `<userData>/local-agents.json`（0600 原子写，`mutateRegistry` 串行化） |
| `validateLocalAgentInput(input)` | 名称 ≤80 / 命令 ≤2048 / 参数 ≤128 / 头像 data URL；远程走 `normalizeRemoteSpec` |
| `addCustomLocalAgent / updateLocalAgent / removeCustomLocalAgent` | custom 上限 64，id `custom:<uuid>`；不允许在本机/远程间切换 |
| `resolveLocalAgentDraft(id, input, signal)` | 不落盘的草稿解析（编辑器预览/连接测试） |
| `detectLocalAgents(dependencies, onlyId?)` | 整体 15s 超时；每项并行 `resolveExecutable` + `findDesktopApp` + `executableVersion`（`--version`，6s）；状态 `ready / desktop-only / not-found` |
| `cachedRemoteAgentSpec / remoteAgentSpec` | 远程（SSH）Agent 规格 |
| `validateLocalAgent(id)` | 运行前必查（存在且已安装） |

**shellPath.ts**（登录 shell 环境修复）：

- `loginShellValue(expression)`：`zsh/bash -lic` 优先（非交互 zsh 跳过 `~/.zshrc`，而 CLI 安装器常写在那里）→ `-lc` 回退；`__termany_shell__` 哨兵行防 rc 噪声。
- `loginShellCliEnvironment()`：白名单 17 个变量（`ANTHROPIC_*`、`GEMINI_*`、`GOOGLE_*` 等）仅传子进程、从不落盘。
- `executableEnvironment() / spawnEnvironment()`：登录 shell PATH + 常规安装位置（nvm/pnpm/mise/asdf/volta/fnm/bun/cargo/homebrew…）。
- `resolveExecutable(command)`：X_OK + shebang 解释器检查（uv/pipx 场景）；Windows `where.exe`；`codex` 找不到时回退 ChatGPT.app 捆绑引擎（[codexDesktop.ts](../../src/main/codexDesktop.ts)）。

## 3. 执行层接口 — [shared/agentExecutor.ts](../../src/shared/agentExecutor.ts)

```ts
interface AgentExecutor {
  run(config, prompt, signal?, images?, options?): Promise<LocalAgentReply>;
  prepare?(…); releaseIdleConnections?(…); disposeAgent(agentId); resetConversation(…);
}
```

关键类型：`LocalProgress`（phase: `connecting|ready|working|waiting|approval`）、`LocalToolApproval`、`NativeAppSession`（Computer Use 会话授权）、`LocalRunOptions`（`sessionKey / transient / workspaceDirectory / continuationPrompt / onProgress / onApproval / claudeAccountLogin / freshSessionRetry` 等）、`LocalAgentReply`（text/images/远程 outbox 文件）。

[desktopAgentExecutor.ts](../../src/main/desktopAgentExecutor.ts) 是唯一默认实现，绑定 `runLocalAgent / prepareLocalAgentSession / disposeLocalAgentSessions / resetLocalAgentConversation / releaseIdleLocalAgentConnections`。

## 4. 运行时 — [localAgentRuntime.ts](../../src/main/localAgentRuntime.ts)（核心，约 870 行）

### 主入口 `runLocalAgent(config, prompt, signal?, inputImages?, options?)`

1. `withAgentOverride`：`custom:` 前缀或 startupArgs 时解析覆盖。
2. `connectionKind()` 判定：**codex/claude 且非 transient → 持久连接**；其余 → 一次性进程（需 `processBudget.acquire`）；远程规格 → `executeRemoteAgent`（SSH）。

### 一次性执行 `executeLocalAgent`

`validateLocalAgent` → `spawnEnvironment()` → 工作区（`localWorkspace`，无 sessionKey/transient 时 `mkdtemp('douchat-agent-')`）→ 输入图片落盘（自定义文件夹用 `.douchat-input-*` 隐藏目录）→ `spawnOwnedProcess` 启动（Gemini 附 `geminiImagePolicy` TOML）→ stdout/stderr 收集（8MB 上限；Grok/Gemini 用流解析器提供 1s 级进度，其余 15s 心跳）→ `localAgentText` 解析 + 图片收集 → finally 清理临时文件。

### 持久连接回合 `runConnectedAgent`

`connectionPlan` 计算连接键 → 池中查活连接（busy 报 "This local agent conversation is already working"）→ `LocalAgentConnection.connect`（失败即驱逐）→ `connection.turn(...)`（warm 连接直接用 `continuationPrompt` 不重发全文）→ 审批经 `onApproval` → `AgentPermissionBroker.authorize` → 失败处理（Claude session 丢失 → `workspace.remember(undefined)` + `freshSessionRetry`；凭据冲突 → `claudeAccountLogin: true` 重试）→ finally `retainIdle`（**空闲 10 分钟、上限 4 条**）。

### 其它关键导出

| 导出 | 说明 |
| --- | --- |
| `localAgentArgs(id, prompt, output, appOwnedWorkspace)` | 每 CLI 的一次性参数适配器（见 §6 表） |
| `localAgentText(id, stdout)` | 输出解析（Gemini/Grok 流式聚合；Kimi JSONL；OpenCode 事件流；fastclaw/hermes/omp 原文；其余 JSON 字段） |
| `localAgentExitError(agent, code, stdout, stderr)` | 非零退出归因（剥 ANSI，优先 stdout JSON 错误） |
| `localAgentEnvironment / shouldRetryClaudeWithAccountLogin` | Claude 账号登录回退：删 `ANTHROPIC_API_KEY` 冲突变量后用 claude.ai 登录重试（唯一允许的自动重试） |
| `localAgentExecutable(id, installedPath)` | macOS Grok 优先用 `~/.douchat/local-tools/grok/<arch>/grok`（socket-deny 兼容补丁版，见 patches/grok-build） |
| 图片收集 | `codexThreadId`+`~/.codex/generated_images`、`grokGeneratedImages`（严格限定 `~/.grok/sessions` 当前会话防逃逸）、`geminiImageFiles`（`nanobanana-output` 前后快照差集）；限额 4 张 / 单张 8MB / 总 20MB |
| `prepareLocalAgentSession` | 预热（有空闲槽位才拉起，不驱逐在用会话） |
| 生命周期 | `disposeAllLocalAgentSessions / disposeLocalAgentSessions(agentId) / resetLocalAgentConversation(conversationId, topicId?) / releaseIdleLocalAgentConnections` |

## 5. 私有 stdio 协议 — [localAgentConnection.ts](../../src/main/localAgentConnection.ts)

`LocalAgentConnection` 类（`kind: 'codex' | 'claude'`），每会话一条私有 stdio 连接、无 TCP 监听：

| 方法 | 说明 |
| --- | --- |
| `connect(path, cwd, env, model?, discoveryOnly?, resume?, …)` | **Codex**：`codex app-server` JSON-RPC over stdio，`initialize` → `thread/start`（`ephemeral: !resume`，`sandbox: workspace-write`，`web_search: live`）或 `thread/resume`（not found/active writer 时 forget 重启）。**Claude**：`claude -p --input-format stream-json --output-format stream-json --verbose --include-partial-messages`，resume 用 `--resume <sid>`，否则 `--no-session-persistence`；审批开启 `--permission-prompt-tool stdio`，否则 `--permission-mode dontAsk`；工具白名单 `--allowedTools WebSearch,WebFetch` |
| `turn(prompt, signal, progress?, onApproval?)` | 一轮对话：Codex `turn/start`；Claude 写 `{"type":"user",…}` 帧。监听 Claude `system/init`（捕获 session_id 并 `rememberThread`）、`stream_event`、`assistant`、`result`；Codex `item/started|item/completed`（活动标签）与 `turn/completed`。进度 15s 心跳 + 1s 节流；`turnInProgress` 拒绝并发复用 |
| `approveComputerUse` / `approveClaudeTool` | 审批转发：Codex `mcpServer/elicitation/request`（仅 Computer Use 服务器：`cua_repl/computer-use/computer_use`）；Claude `can_use_tool` control_request → allow/deny |
| `models()` | 模型发现（Claude `initialize` 的 models；Codex `model/list` 分页） |
| `request / write / read` | JSONL 帧编解码（单帧 64MB、RPC 60s 确认超时） |
| `close(error?, graceful?)` | 优雅关闭或强杀（`kill(-pid, SIGKILL)` / Windows `taskkill.exe /T /F`） |

进程管理：`spawnOwnedProcess`（detached 独立进程组 + `ownedProcesses` 跟踪）、`stopLocalProcess`（关 stdin → SIGTERM → SIGKILL）、`killAllLocalProcesses`（退出兜底）。Computer Use 清单预查 `loadComputerUseInventory`（`mcpServerStatus/list`，5 分钟缓存）。

## 6. CLI 适配器一览

| CLI | 一次性参数（`localAgentArgs`）要点 | 输出解析 | 持久协议 | 特点 |
| --- | --- | --- | --- | --- |
| `codex` | `exec --json --skip-git-repo-check --ephemeral --sandbox workspace-write … --output-last-message <file> -`（提示走 stdin） | 读 `--output-last-message` 文件 | ✅ app-server JSON-RPC | thread 会话恢复；生成图片收集 |
| `claude` | `-p --output-format json --allowedTools WebSearch,WebFetch -- <prompt>` | JSON `result` | ✅ stream-json 双向 | 账号登录回退；Bash 桥 curl 豁免 |
| `gemini` | `-p <p> --output-format stream-json` | GeminiStream 聚合 | ❌ | `GEMINI_CLI_TRUST_WORKSPACE=true`；Nano Banana 图片策略 |
| `grok` | `--no-auto-update -p … --output-format streaming-json --permission-mode dontAsk --tools read_file,list_dir,grep,web_search,web_fetch,image_gen,image_edit --sandbox strict` | GrokStream | ❌ | 只给安全工具集；图片严格校验会话目录 |
| `cursor` | `[--trust] --print --output-format json --mode ask -- <p>` | JSON | ❌ | 紧凑工作区路径（信任标记 NAME_MAX） |
| `opencode` | `run --format json -- <p>` | 事件流 `type==='text'` | ❌ | 模型 `owner/name` |
| `kimi` | `--prompt <p> --output-format stream-json` | JSONL assistant 拼接 | ❌ | 版本查 PyPI |
| `openclaw` | `agent exec --message-file - --json --code-mode direct`（stdin） | stdout JSON envelope | ❌ | 模型 `models list --json` |
| `hermes` | `--oneshot <p>` | 原文 | ❌ | 版本查 GitHub Releases |
| `omp` | `--print --mode text --no-session --no-tools <p>` | 原文 | ❌ | 模型 `models --json` |
| `fastclaw` | `chat --query <p>` | 原文 | ❌ | 支持 `{prompt}` 启动参数展开 |
| 自定义 | 用户命令 + `customLocalAgentArguments`（`{prompt}` 占位） | stdout 原文 | 远程 codex/claude 可走 | 可为 SSH 远程 Agent |

模型与思考档位：`withLocalModel(id, args, model)`（`--model` 插在 `--` 前）、`withLocalThinking`（codex `-c model_reasoning_effort=…`、claude `--effort …`，档位定义在 [shared/thinkingLevels.ts](../../src/shared/thinkingLevels.ts)）。

## 7. 工作区 — [localWorkspaces.ts](../../src/main/localWorkspaces.ts)

- 根目录 `<userData>/local-workspaces/`；`validateWorkspaceFolder(path)` 拒绝磁盘根/home/系统目录/Douchat 数据目录。
- **`localWorkspace(config, sessionKey?, customDirectory?, placement?)`**：
  - 会话记录 `sessions/<sha256(owner,agent,sessionKey)>.json`：`{owner, agent, sessionKey, generation, fingerprint, thread?, claudeAccountLogin?}`（原子写）。
  - `fingerprint = localAgentId + instructions + role + name + model + thinkingLevel (+自定义文件夹)` —— **人格/模型变化保留文件但失效原生线程绑定**（换新进程、续聊转写）。
  - 文件目录 `files/<hash(owner)>/<hash(agent)>/<id>/<generation>`；Cursor 用紧凑 `cursor/<id>/<generation>`。
  - 返回的 `remember(thread?)` 在 generation/fingerprint 已变化（清空会话）时**拒绝写入** —— 防止迟到的完成结果复活已清除的上下文。
- `resetLocalWorkspaces(owner, matches)`：换新 generation、清 thread（文件保留）。
- `openableWorkspace(config, sessionKey)`：只读解析最新可打开工作区。

## 8. 进程预算 — [localProcessBudget.ts](../../src/main/localProcessBudget.ts)

`LocalProcessBudget(8)`：FIFO 等待队列；满员时调用 `pressure` 回调（runtime 注入 `evictIdleConnection` 驱逐空闲连接腾位）；lease 保持到进程真正关闭才释放。

## 9. 技能桥 — [localSkillBridge.ts](../../src/main/localSkillBridge.ts)

`openLocalSkillBridge(tools, signal)`：`127.0.0.1` 随机端口 HTTP 服务器（仅 `POST /tools` + Bearer 32 字节随机 token、拒绝带 Origin 的请求、单并发、3MB 上限），`validateToolArguments` 校验后执行宿主 `AgentTool`；返回 `{ close, port, token, tools, isBridgeCommand, prompt }` —— `prompt` 告诉 CLI 用原生 shell/HTTP 工具 curl 该端点安装技能等；**回合结束即关闭，token 不复用**。远程 Agent 走 `openRemoteSkillBridge`（SSH 隧道转发到本机桥，[remoteSkillBridge.ts](../../src/main/remoteSkillBridge.ts)）。

`isBridgeCurl(command, endpoint, token)`：严格白名单解析 curl 命令，供 Claude `Bash` 审批豁免。

## 10. 权限代理 — [agentPermissions.ts](../../src/main/agentPermissions.ts)

- `toolCapability(name)`：工具名 → 敏感能力（filesRead/filesWrite/network/browserControl/accountRead/accountWrite/automation/otherTools）。
- `reusableScope(input)`：可复用授权范围（URL origin、绝对只读路径、新建文件、邮箱账号+文件夹）；写已有文件/发送/点击永远单次。
- **`AgentPermissionBroker`**：决策全在 owner 主进程。`beginTask/endTask`（任务级授权集）、`resolve(id, allow)`（`true/false/'task'/'session'`）、`authorize(config, input, signal?, forceAsk?, taskId?, native?)` —— pending 上限 20、10 分钟过期；`NativeAppSession`（Computer Use app 授权）跨轮次按 `appId+capability+requester` 复用；账号切换立即失效。默认策略 `agentPermissions()`（[shared/agentPermissions.ts](../../src/shared/agentPermissions.ts)）：敏感能力默认 `ask`，群成员默认 `allow`。

## 11. 一轮本地 Agent 对话完整流程

```text
runtime.performReply（本地分支）
 1. 远程判定 remoteAgentSpec → 工作区锁 acquireWorkspace
 2. openLocalSkillBridge（bridgeTools = 技能安装 + 技能读取 + 产物 + 连接器工具）
 3. promptParts：身份（SOUL.md/IDENTITY.md…）+ 技能资源 + 记忆指令 + 身份编辑通道
    + update_user_memory + 例程指令 + 图像约束 + workspace 说明 + 用户 prompt
    （冷直聊连接附最近历史 ≤24000 字符 "Conversation so far"）
 4. localExecutor.run(config, prompt, abort.signal, images, {sessionKey, …})
      ├─ 持久路径 runConnectedAgent：connectionPlan → 池 → connect → turn → 审批回调 → retainIdle
      ├─ 一次性路径 executeLocalAgent：validate → spawnEnvironment → workspace → argv 组装
      │   → spawnOwnedProcess → stdout 收集 → localAgentText → 图片收集
      └─ 远程路径 executeRemoteAgent：probeRemoteAgent → RemoteRun.prepare → ssh stdin/argv → outbox 文件
 5. 后处理：saveImageAttachment；plainRemoteLinks；localRoutineDirectives 二次校验（fail-closed）；
    身份编辑（[[douchat_update_agent_files]]）与记忆编辑验证落库 + 回执
```

## 12. 周边模块速览

| 模块 | 职责 |
| --- | --- |
| [localAgentModels.ts](../../src/main/localAgentModels.ts) | `listLocalAgentModels(id)`（去重并发、15s 超时）：codex/claude 走连接 `models()`；openclaw/omp/其余跑 CLI models 命令；远程走 `discoverRemoteModels`（同目录命令经 SSH） |
| [localAgentUpdates.ts](../../src/main/localAgentUpdates.ts) | `latestAgentVersion(id)`：npm registry / GitHub Releases / PyPI / `x.ai/cli/stable` / cursor 安装脚本；`checkLocalAgentUpdates` → `updateStatus: unknown/available/current` |
| [localAgentMaintenance.ts](../../src/main/localAgentMaintenance.ts) | `maintenancePlan/resolveMaintenancePlan/prepareNpmMaintenance`：按安装来源（npm 全局/Homebrew/winget/update 子命令/官方脚本）生成升级命令 |
| [localAgentTest.ts](../../src/main/localAgentTest.ts) | `testLocalAgent(id, input)`：30s 超时，提示词要求**精确回复 `DOUCHAT_OK`**（防把 `--help` 横幅当成功） |
| [skillTools.ts](../../src/main/skillTools.ts) | `createSkillTools` → `list_skill_files` / `read_skill_file`（只读已验证打包快照内文件） |
| [skillArchive.ts](../../src/main/skillArchive.ts) | `parseSkillArchive`：ZIP/TAR.GZ，要求 SKILL.md + YAML frontmatter（name ≤100 / description ≤5000），≤50 个技能 |
| [skillInstallation.ts](../../src/main/skillInstallation.ts) | `searchSkills`（skills.sh API）、`loadSkillSource`（GitHub tar.gz / 本地目录 / ZIP / TAR.GZ）、`createSkillInstallationTools`（search/list/install/create）；从不执行脚本 |
| [agentArchive.ts](../../src/main/agentArchive.ts) | `exportAgentArchive`（ZIP：agent.json v2 + Markdown + skills/）、`parseAgentArchive`（自有 v1/v2 + 第三方 Hermes/Workspace 格式） |
| [agentArtifacts.ts](../../src/main/agentArtifacts.ts) | 回合产物（图片/文件）保存与消息链接 |
| [managedNode.ts](../../src/main/managedNode.ts) | 托管 Node 运行 npm 维护命令 |
| [terminalLauncher.ts](../../src/main/terminalLauncher.ts) | "在终端打开" CLI 登录会话 |
| [workspaceTools.ts](../../src/main/workspaceTools.ts) | `createWorkspaceTools(directory, lock)`：list/read/write 工作区文件（仅云端 Agent）；写需 `overwrite=true`、O_EXCL、按文件夹锁串行 |
| [computer.ts](../../src/main/computer.ts) | `LocalComputerProvider`：每 Agent 独立隐藏 BrowserWindow（`partition: persist:douchat-agent-<agentId>`）；`navigate/observe/click/type/scroll/capture` + 9 个 `computer_*` 工具；文件白名单 Downloads/Desktop/Documents + 授权文件夹 |
| [remoteTransport.ts](../../src/main/remoteTransport.ts) 等 | SSH 远程 Agent：`RemoteRun` 服务器目录、outbox 文件交付、`launchScript` |
