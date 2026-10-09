# Douchat Code Wiki

本 Wiki 基于 `v0.1.23` 源码逐文件分析生成，覆盖项目整体架构、模块职责、关键类与函数、依赖关系与运行方式。所有函数名、类名、常量均直接取自源码，可直接作为代码导航与二次开发的参考。

## 项目是什么

**Douchat** 是一个 Electron 桌面应用（macOS / Windows / Linux），定位为"AI Agent 协作工作台"：把独立 AI Agent 当作联系人，通过直聊、群聊、话题、`@` 提及、私信与 Agent-to-Agent 委派进行协作。

- **云 Agent**：登录 [douchat.ai](https://douchat.ai) 账号后，运行在 Douchat 托管模型上（OpenAI 兼容 Chat API）。
- **本地 Agent**：复用本机已安装的 CLI 工具（Claude Code、Codex、Gemini、Grok Build、OpenClaw、Hermes、OpenCode、Cursor、Kimi、OMP、Fastclaw），不复制凭据。
- **自定义模型**：任意 OpenAI / Anthropic 兼容服务商（OpenRouter、TokenDance、DeepSeek 等）。

技术栈：**Electron 43 + electron-vite 5 + React 19 + TypeScript 5.9 + node:sqlite + Vitest**，Agent 引擎基于 `@earendil-works/pi-agent-core` 与 `@earendil-works/pi-ai`。许可证 AGPL-3.0（另提供商业授权）。

## 目录结构

```text
src/main/          Electron 主进程：认证、SQLite 存储、Agent 运行时、群组编排、IM/社交/连接器（约 150 个 ts 文件）
src/preload/       沙箱渲染进程的 typed IPC 桥（window.douchat 命名空间）
src/renderer/      React 应用：聊天、联系人、设置等 UI 组件
src/shared/        主/渲染进程共享的数据类型与协作协议纯函数（含 bot/ 协议核心）
resources/         应用图标、macOS 签名 entitlements
scripts/           dev-host 准备、图标生成、发布脚本、群组验证场景
docs/              设计文档（本 Wiki 位于 docs/wiki/）
```

## Wiki 导航

| 文档 | 内容 |
| --- | --- |
| [01 · 项目概览与整体架构](01-architecture.md) | 进程模型、模块分层图、一条消息的完整数据流、安全模型 |
| [02 · 主进程核心模块](02-main-process.md) | index.ts 装配、DouchatStore 存储、DesktopAuth 认证、gateway 网关、scheduler 调度、记忆与更新 |
| [03 · Agent 运行时](03-agent-runtime.md) | DouchatRuntime 核心流程：performReply、会话缓存、工具装配、超时与重试 |
| [04 · 本地 Agent 子系统](04-local-agents.md) | CLI 检测、11 个 CLI 适配器、持久连接协议、工作区、进程预算、技能桥 |
| [05 · 群组协作子系统](05-group-collaboration.md) | 群决策（Jev/LLM 规划）、任务图 DAG、健康与隔离、工作流日志、群游戏、私信/A2A 协议 |
| [06 · IM 通道、社交与连接器](06-im-social-connectors.md) | 微信/飞书/Telegram 接入、好友与共享群、邮件/Connany 连接器 |
| [07 · 渲染层 UI](07-renderer.md) | App.tsx 结构、组件清单与职责、状态管理、消息队列 |
| [08 · Preload 与 IPC 契约](08-ipc-preload.md) | window.douchat API 表面、channel 命名规范、推送事件 |
| [09 · 共享层与数据模型](09-shared-data-models.md) | AgentConfig/Conversation/ChatMessage/AppSnapshot 等核心类型、bot/ 协议助手 |
| [10 · 依赖关系](10-dependencies.md) | 运行时/开发依赖清单、模块间依赖图、对 pi-agent-core 的分层 |
| [11 · 开发、构建与发布](11-development.md) | 环境要求、npm 脚本、环境变量、数据目录、打包与自动更新 |

## 推荐阅读路线

- **快速了解**：01 架构 → 11 运行方式
- **理解消息流转**：01 → 03（运行时）→ 05（群组）
- **做本地 Agent 相关开发**：04 → 10（依赖）→ 09（类型）
- **做 UI 开发**：07 → 08（IPC 契约）→ 09（类型）

## 与其他文档的关系

- `README.md`：面向用户的产品介绍与环境变量说明
- `docs/*.md`：各子系统的设计笔记（如 `docs/group-orchestration-design.md`、`docs/local-agent-runtime.md`），Wiki 在相应章节会引用
- `CONTRIBUTING.md`：贡献流程与 PR 规范
