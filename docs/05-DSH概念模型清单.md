# DSH 概念模型清单（出场自带部分）

> **口径**：只统计**随应用出场的 2 个 bundle**（`@deepseek-ai/dsh-base`、`@deepseek-ai/dsh-web-app`）及其内置 preset；第三方 bundle 不计入。
> `dsh-web-app` 自带 4 个 preset（`standard`、`ptc`、`minimal`、`cordis`），本文以 `standard` 为准（它启用压缩、子代理、任务板、skill、goal 等）。
> **日期**：2026-11-21
> 落点列中的名字是 Entry id（可在 `04` / 组合里查）或服务名。

---

## 一、真相层：一切从日志来

| 概念 | 是什么 | 落点 |
|---|---|---|
| **Session（会话）** | 一段 append-only 的事件日志，是唯一真源；持久化为 JSONL | `session`、`session-persistence-jsonl` |
| **SessionEvent** | 日志的事件类型：`turn/*`、`step/*`、`user/message`、`assistant/message`、`tool/call`、`tool/result`、`system/message`、插件自定义事件 | 同上 |
| **Turn（轮）** | 一次输入触发的处理区间：`turn/start` → … → `turn/end` | `agent-loop` |
| **Step（步）** | 轮内的一次模型调用及其工具执行：`step/start` → `step/end` | `agent-loop` |
| **Projection（投影）** | 从日志派生的"当前值"（标题、大纲、统计、任务板、待答问题…），可缓存、可订阅 | `session-projection`、`session-projection-cache` |
| **派生展示** | 会话标题、轮次大纲、统计 | `session-title`、`session-turn-outline`、`session-stats` |

## 二、执行体

| 概念 | 是什么 | 落点 |
|---|---|---|
| **Agent** | 绑定一个 Session 的执行体；`agent.id` 就是 SessionId；拥有自己的 `ctx` | `agent`（`ctx.agents`） |
| **Agent loop** | 具体 driver：认领输入 → 开 turn → 组 prompt → 调 LLM → 跑工具 → 写日志 | `agent-loop`（`ctx.agentLoop`） |
| **Inbox（入站队列）** | agent 自己的输入队列：`next-step`（steer）与 `next-turn`（queue）；有"已认领 / 未认领"之分 | `agent-loop` 内部 |
| **Agent 状态** | `running` / `inactive`，随 `agent/status` 事件变化 | `agent` |
| **中断** | 取消当前轮/当前任务；可保留队列（`keepInbox`） | `Agent.cancel`、`interrupt_agent` |
| **Scope（注册作用域）** | 按 agent 决定某条注册对谁可见 | `scope`（库，非服务） |

## 三、上下文与"记忆"

| 概念 | 是什么 | 落点 |
|---|---|---|
| **上下文（对话历史）** | **不是一个对象，而是每步从日志重建的模型输入**：system prompt + 历史消息 + 工具 schema | `agent-loop` + `system-prompt` |
| **System prompt** | 分节注册后组装的提示前缀（persona、AGENTS.md、时间、工具 schema 等） | `system-prompt`、`persona`、`agent-instructions`、`time-context` |
| **上下文预算** | 用 token 估算衡量（非精确值），用于触发压缩 | `token-meter` |
| **上下文压缩（compaction）** | 超阈值时先剪工具结果、再摘要；在同一个 open step 内恢复重试 | `compaction-basic`（standard preset 启用）、`command-compact` |
| **工具结果裁剪** | 头/中/尾保真裁剪 | `tool-result-pruner`（standard preset 启用） |
| **大结果外溢（spill）** | 超预算的工具结果落盘，留下可恢复指针 | `spill-policy`、`spill-local` |
| **图片外溢** | 超预算图片换占位并重试 | `compaction-image-offload` |
| **memory** | **不存在独立的 memory 子系统** | 见下方说明 |

### 关于 memory

DSH 没有 memory 服务，只有三样东西替代：

| 常说的记忆 | DSH 里的实际对应 |
|---|---|
| 短期记忆 | 当前上下文窗口（每步从日志重建的模型输入） |
| 长期记忆 | 会话日志（JSONL 持久化）+ 工作区文件（AGENTS.md 等）+ skill |
| 跨会话检索 | `session-query-sqlite`（SQLite FTS5） |

## 四、能力面

| 概念 | 是什么 | 落点 |
|---|---|---|
| **Tool（工具）** | 注册进注册表的可调用能力，带 schema 与执行管线（pre/post 钩子、并行、超时） | `tools`、`tool-executor-native` |
| **Skill（技能）** | 按需加载的指令集（不是工具），由 agent 主动读取 | `skill`、`tool-skill`、`skill-filesystem`、`skill-office` |
| **MCP 工具** | 外部 MCP 服务器提供的工具，桥接进 `ctx.tools` | `mcp-client`、`mcp-manager` |
| **Web / LSP / Office** | 搜索抓取、语言服务、文档处理 | `web`*、`lsp`*、`office-kit` |

## 五、协作与时间

| 概念 | 是什么 | 落点 |
|---|---|---|
| **Subagent（子代理）** | 由父创建，可续接、可冷恢复；有深度/容量上限、子代理目录、消息与打断 | `subagent`、`subagent-spawn-in-process`、`subagent-fork-in-process`、`tool-subagent-control` |
| **Team 任务板** | 子代理树共享的任务 DAG（认领、依赖、写域、CAS） | `agent-team`、`tool-agent-team` |
| **Goal（目标）** | 同会话的持久目标与自动续跑轮 | `goal`、`goal-round-driver` |
| **Plan mode（计划模式）** | 先出计划、经用户放行才执行 | `plan-mode` |
| **Todo** | 写进会话日志的待办（**不是**全局收件箱） | `tool-todo` |
| **Schedule（定时）** | 宿主级持久提醒，可投给已有或新会话 | `schedule` |
| **Jobs（后台任务）** | 后台进程注册表 | `jobs-local`、`tool-jobs` |
| **Terminal（终端）** | 会话内 PTY | `terminal-*`（minimal/standard preset 内） |

## 六、约束与信任

| 概念 | 是什么 | 落点 |
|---|---|---|
| **Approval（审批）** | 一次性权限决策，默认 fail-closed | `approval` |
| **Permission preset** | 一个"权限"选择，捆沙箱模式与审批策略 | `permission` |
| **Sandbox（沙箱）** | 对 bash/pwsh/fs 的强制约束（read-only / workspace-write / full） | `sandbox`、`sandbox-policy`、`bash-sandbox`、`fs-sandbox` |
| **Credentials / 授权** | 凭据文件，以及"通过对话取得凭据"的授权流程 | `credentials`、`authorization`、`deepseek-account` |
| **Workspace / 工作目录** | 会话的 cwd 与工作区注册表 | `working-directory`、`workspace` |

## 七、界面模型（web-app）

| 概念 | 是什么 | 落点 |
|---|---|---|
| **Slot（插槽）** | 插件挂 UI 的位置（侧边栏、会话区、输入区…） | `ui-layout`、各 `client-ui-*` |
| **Conversation / Composer / Queue** | 会话装配、输入框、待发队列 | `client-ui-conversation` |
| **Sidebar** | 会话多级树、状态点、分组 | `client-ui-sidebar` |
| **投影驱动 UI** | GoalBar、任务板、统计等直接读投影 | `ui-goal`、`ui-agent-team`、`ui-trajectory` |
| **结构化提问** | `ask_user_question`（仅主 agent 可用，子代理被拒） | `tool-ask-user`、`ui-user-questions` |
| **引用与附件** | `@文件` / `@会话`、文件上传 | `client-ui-reference`、`attachment-local` |

---

## 附：standard preset 实际挂载的能力（`dsh-web-app/presets/standard.patch.yml`）

```
persona · agent-instructions · time-context
tool-bash · tool-pwsh · tool-fs · tool-fs-search · tool-jobs · tool-schedule
skill-filesystem · tool-skill · tool-office
command-goal · tool-goal · planning · plan-mode
compaction · compaction-basic · command-compact · tool-result-pruner
delegation · tool-subagent · tool-subagent-control · tool-subagent-fork
workflow-ptc · tool-workflow · tool-ask-user · tool-todo · tool-web
present · tool-plugin-manager · tool-plugin-market
```

## 附：三个容易混的"inbox"

| 叫法 | 实际是什么 | 归属 |
|---|---|---|
| agent inbox | agent 自己的输入队列（steer / queue） | `agent-loop` 内部，日志之外 |
| todo | 写进会话日志的待办 | `tool-todo`（模型工具） |
| 全局收件箱 | 跨会话的人为待办面板 | 第三方插件 `dsh-todo-inbox`，非官方 |
