# 11 · 开发、构建与发布

> 参考：[README.md](../../README.md) · [CONTRIBUTING.md](../../CONTRIBUTING.md) · [docs/releasing.md](../releasing.md)。构建配置：[electron.vite.config.ts](../../electron.vite.config.ts) · [package.json](../../package.json) `build` 段

## 1. 环境要求

- **Node.js ≥ 22.12**（用到 `node:sqlite`）、**npm ≥ 10**
- macOS：Xcode Command Line Tools（签名/公证构建）
- 可选：douchat.ai 账号（云 Agent）、本地 Agent CLI（Claude Code / Codex / Gemini…）

## 2. 快速开始

```bash
git clone https://github.com/thinkany-ai/douchat.git
cd douchat
npm ci
cp .env.example .env
npm run dev
```

点击 **Get started** → 浏览器登录 douchat.ai 账号 → 应用自动返回（`douchat://` 协议回调；开发版走 `douchat-dev` + loopback `127.0.0.1` 回调服务）。

### npm 脚本

| 脚本 | 作用 |
| --- | --- |
| `npm run dev` | 先 `prepare:dev-host`（scripts/prepare-electron-dev-host.mjs）再 `electron-vite dev --watch` 热重载（DEV 徽标图标、`douchat-dev` 数据目录） |
| `npm run typecheck` | `tsc --noEmit -p tsconfig.node.json && tsc --noEmit -p tsconfig.web.json` |
| `npm test` | `vitest run src`（测试与源码同目录 `*.test.ts(x)`） |
| `npm run build` | `electron-vite build` → 产物 `out/{main,preload,renderer}` |
| `npm run preview` | 预览生产 bundle |
| `npm run package` | `build` + `electron-builder --dir`（当前平台 unpacked） |
| `npm run package:mac` | 未签名 Apple Silicon + Intel DMG/ZIP（**仅供冒烟测试**，禁签名/公证/hardenedRuntime） |
| `npm run package:win` | x64 NSIS 安装器 |
| `npm run package:linux` | x64 AppImage + deb |
| `npm run release:mac` | 签名发布构建（维护者用，见 docs/releasing.md） |

构建细节（electron.vite.config.ts）：preload 输出 **CJS**（`index.cjs`）；renderer 固定端口 dev 5275 / preview 5274（避开 Termany 5174 与 Vite 5173）；`optimizeDeps.include` 预热 React/dialog 依赖。

### 环境变量（.env）

| 变量 | 用途 | 默认 |
| --- | --- | --- |
| `DOUCHAT_SERVICE_URL` | 开发登录与 Cloud Chat 源 | `https://douchat.ai`（**打包版忽略此变量**） |
| `DOUCHAT_WEB_URL` | 上者别名 | — |
| `GATEWAY_BASE_URL` / `GATEWAY_API_KEY` | 测试用 OpenAI 兼容网关 / 未登录脚本运行时 | — |
| `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` / `GOOGLE_API_KEY` / `OPENROUTER_API_KEY` / `DEEPSEEK_API_KEY` | 可选直连 provider 凭据 | — |

网关 live 测试（缺任一变量自动跳过）：

```bash
GATEWAY_BASE_URL=https://gateway.example/v1 \
GATEWAY_API_KEY=replace-with-a-local-secret \
npx vitest run src/main/gateway.live.test.ts
```

## 3. 数据目录（本地优先）

| 数据 | 位置 |
| --- | --- |
| 全部本地数据 | `~/Library/Application Support/douchat`（开发 `douchat-dev`；Windows/Linux 对应 userData 目录） |
| 主数据库 | `<userData>/douchat.db`（SQLite WAL） |
| 登录凭据 | `<userData>/auth.json`（safeStorage 加密，0600） |
| 日志/崩溃 | `<userData>/logs/`（diagnostics.log + crashes；Ctrl/Cmd+Shift+L 或 Settings → About 打开） |
| 本地工作区 | `<userData>/local-workspaces/`（会话记录 sessions/*.json + files/…） |
| 自定义 Agent 注册表 | `<userData>/local-agents.json`（0600） |
| IM 通道凭据 | `<userData>/im-channels/`（加密 JSON） |
| 邮件连接器 | `<userData>/email-connectors.json`（加密） |
| 自定义模型 | `<userData>/custom-models/sha256(account).json`（key 加密） |
| 附件 | `<userData>/attachments/` |

删除 userData 目录即重置本地档案（好友/共享群下次登录从服务端同步回来）。云端持有：账号资料、额度、好友、共享群消息。

## 4. 提交前检查

```bash
npm run typecheck && npm test && npm run build
```

PR 规范（CONTRIBUTING.md）：从 `dev` 分支拉出；保持聚焦；UI 变更附截图/录屏；**绝不提交真实凭据**；`git commit -s` 签署 DCO；贡献遵循 AGPL-3.0 + 商业双授权条款。

## 5. 测试体系

- 运行器：Vitest（jsdom 环境），`vitest run src` 全量。
- **同目录测试**：`src/main/*.test.ts`（约 100 个）、`src/renderer/src/**/*.test.tsx`、`src/shared/**/*.test.ts` —— 行为规范即测试（如 `bot.test.ts` 钉住协作协议）。
- 场景验证脚本：`scripts/group-scenarios.ts`、`group-health-scenarios.ts`、`group-policy-scenarios.ts` + `docs/validation/*.json`（群编排/健康/策略的录制验证）。
- live 测试：`gateway.live.test.ts`（需环境变量）。

## 6. 打包与发布

- 产物目录 `release/<version>/`（electron-builder 配置在 package.json `build` 段：appId `ai.thinkany.douchat`、asar、协议 `douchat://`、languages en/zh_CN）。
- macOS：DMG + ZIP（x64/arm64），hardenedRuntime + entitlements（[resources/entitlements.mac.plist](../../resources/entitlements.mac.plist)），公证；NSMicrophoneUsageDescription 等隐私描述内置。
- Windows：NSIS x64；Linux：AppImage + deb x64。
- 自动更新：electron-updater generic provider `https://cdn.douchat.ai`（channel latest）；主进程 `DesktopUpdater` 状态机（busyTasks 时延迟安装）；CI 发布流见 [.github/workflows/release.yml](../../.github/workflows/release.yml) 与 [publish-cdn.yml](../../.github/workflows/publish-cdn.yml)。
- 维护者签名发布流程：[docs/releasing.md](../releasing.md)。

## 7. 常见开发任务入口

| 任务 | 起点 |
| --- | --- |
| 新增 IPC 能力 | `shared/types.ts DouchatApi` → `preload/index.ts` → `main/index.ts` handler（见 [08](08-ipc-preload.md)） |
| 新增群协议能力 | `shared/bot/*`（纯函数 + 测试）→ `runtime.ts runGroupTurn/runDirectTurn` 接线 |
| 新增本地 CLI 适配器 | `localAgents.ts` catalog → `localAgentRuntime.ts` 的 `localAgentArgs/localAgentText`（见 [04](04-local-agents.md)） |
| 新增工具（云端 Agent） | `runtime.ts session()` 工具装配 + `guardedTools` 包装 |
| 新增设置页 | `SettingsPanel.tsx` 标签 + 对应组件 |
