# 可修改点清单（二次开发学习地图）

> 这是我自己的学习笔记，**不是官方文档**。官方权威来源是 `docs/architecture.zh.md`。
> 本文件的目的：随时回来查「我想改 X，应该在哪一层、用哪个机制」。

## 一句话总纲

DeepSeek Harness 是「**一切皆插件**」：模型、工具、会话日志、agent loop 本身都是插件。
所以几乎没有「不能改」的地方，区别只是改的深浅。下面按「从浅到深」分 6 层。

---

## 第 0 层：不改代码，只改配置（最常用、最该先碰）

运行中的 `dsh` 就是一棵「插件树」，由几层按顺序叠起来：

- **profile**（`web` / `headless`）：声明用哪些 bundle
- **bundle**（组合包）：一捆配置 + 挂载代码
- **patch**（`cordis.patch.yml` / `--patch` 叠加）：按 id 定位某个插件、替换它整个配置

> 记住：`dsh --profile web --dump-config` 打印出来的**每一个条目，都能用自己的 patch 覆盖掉**。

这一层能改的：**换模型、加/减工具、换沙箱、改审批策略、开关遥测、改设置**——全都不用写 TypeScript。

---

## 第 1 层：加新东西（插件 / 工具 / 命令）

| 想加什么 | 在哪注册 | 通俗说法 |
|---|---|---|
| 新模型（换 provider） | `ctx.llm` 注册 adapter | 换个「脑子」 |
| 面向模型的新工具 | `ctx.tools` 注册，schema 自动进提示词 | 装个「新 App」 |
| 用户命令 | `ctx.commands` | 加个快捷指令 |
| 后台长任务 | `ctx.jobs` | 加个后台跑的任务 |
| 技能 skill | skill provider | 加一本「说明书」 |
| 工作流 workflow | workflow capability | 加一套固定流程 |
| Web 界面新节点 | `ConversationNodeDefinition` + renderer | 界面上多一种卡片 |

---

## 第 2 层：换掉某个能力的「背后实现」（seam 替换）

这是 Harness 最核心的设计：**很多能力是「三件套」——定义接口（Service Definition）、实现它（Provider）、用它（Consumer）**。
你只换 Provider，整个产品行为就跟着变。

能换的 Provider：文件系统 `ctx.fs`、shell `ctx.shell`、子进程 `ctx.subprocess`、终端 `ctx.terminals`、沙箱 `ctx.sandbox`、子智能体 `ctx.subagent`。

> 最漂亮的一点：把「文件系统 + 进程」指向远程沙箱，**Bash、PTY、LSP 就一起搬过去了**，不用逐个改。

---

## 第 3 层：拦截 / 控制流程（事件 = 钩子）

三类事件就是三类「钩子」：

- **会话事件**（`session/event`）：持久记录，重载后还在 → 改「日志怎么记 / 回放」
- **Agent 事件**（`agent/*`）：观察或拦截进行中的活 → 改「每一步干什么」
- **能力事件**（`fs/*`、`tools/*`、`telemetry/*`）：给某个 seam 挂策略

关键钩子：

- `agent/pre-step` —— 决定模型看到什么
- `agent/request`、`llm/stream` —— 拦截请求 / 流
- `tools/pre-execute` —— 工具执行前拦截，做审批 / 超时
- `agent/turn-stopping` —— 直接叫停一轮

**加审批、加超时、加安全策略、加「干活前先问你」——都在这层。**

---

## 第 4 层：改内核（agent loop / 核心包）

7 个核心包：`session`、`system-prompt`、`tools`、`agent`、`agent-loop`、`scope`、`llm`。

- 改提示词怎么组装 → `core/system-prompt`
- 改工具怎么注册 / 执行 → `core/tools`
- 改 agent loop 本身 → `core/agent-loop`（**改了要同步更新架构文档**）

> **红线**：任何「模型能看到的东西」都必须能从日志重建（「模型可见即已记录」）。
> 所以新增一个模型可见的输入，就得同步新增一个 `SessionEvent`。

---

## 第 5 层：嵌进别的系统（当引擎，不自己跑）

不用它的界面，把它藏后台：

- **headless** 模式：`dsh --profile headless "任务"`，跑一次就退
- **JSON-RPC / SDK**：`packages/sdk`，TypeScript 客户端
- **ACP server**：`packages/acp`，自动化专用协议
- **Python SDK**：`python/`
- **子智能体委派**：把一轮丢给另一个产品
- **自己写 UI**：驱动 `ctx.agents` + 从 `session/event` 渲染

---

## 对着我的两个目标看

| 我的目标 | 主要落在哪几层 |
|---|---|
| 目标 1：做 workbuddy 那样的通用工具 | 第 0、1、2、3 层（换模型、装工具、加审批、定制界面、打包） |
| 目标 2：当别的系统的大脑 | 第 5 层（SDK / ACP / headless 当引擎接进去） |

> 注：如果自建 loop 只缺某一块（比如「审批」或「日志回放」），可以只抄那一块的设计，不整体迁移。

---

## 下次深入学习从哪读（官方文档索引）

| 文档 | 讲什么 |
|---|---|
| `docs/architecture.zh.md` | 权威架构（本文的源头） |
| `docs/cordis-primer.md` | Cordis 入门（底层框架） |
| `docs/cookbook/extension-cookbook.md` | 功能 → 能力的映射索引 |
| `docs/cookbook/adding-a-tool.md` | 加工具分步指南 |
| `docs/cookbook/adding-an-llm-adapter.md` | 加模型适配器 |
| `docs/cookbook/adding-a-package.md` | 加一个包 |
| `docs/cookbook/adding-a-conversation-node.md` | 加 Chat 节点 |
| `docs/config-catalog.md` | 生成的配置字段目录 |

## 包位置速查（仓库布局，来自 AGENTS.md）

- `packages/core/` —— 产品 API 主干（session、system-prompt、tools、agent、agent-loop、scope）
- `packages/llm/` —— LLM 能力（Service Definition/Consumer + DeepSeek provider）
- `packages/shell/` `subprocess/` `terminal/` `fs/` `lsp/` —— 执行与文件相关能力
- `packages/skill/` `workflow/` `todo/` `plan/` —— 高阶能力
- `packages/subagent/` —— 子智能体
- `packages/session/` —— 持久会话数据
- `packages/interaction/` —— 审批 / 权限 / 命令 / ask-user
- `packages/sdk/` —— JSON-RPC 协议、server、TS 客户端
- `packages/acp/` —— Agent Client Protocol 自动化 server
- `packages/bundle/` —— 可安装的 profile patch 层
- `python/` —— Python SDK 与内置运行时
