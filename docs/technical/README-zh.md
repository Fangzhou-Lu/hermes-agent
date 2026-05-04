# Hermes Agent — 技术文档(中文导航)

> v0.12.0 技术参考文档的中文索引。完整内容仍以英文 `00-overview.md` 起始的 20 份文档为准。

本目录下的技术文档面向贡献者、打包者、插件作者以及任何需要深入理解
或扩展 Hermes Agent 代码的人。所有文档以英文撰写以便国际协作,但本
索引提供中文导航,帮助你快速定位。

## 文档目录

| # | 英文文件 | 主题 |
|---|----------|------|
| 00 | [00-overview.md](00-overview.md) | 文档总览 / 阅读路线 |
| 01 | [01-architecture.md](01-architecture.md) | 进程拓扑、Agent 主循环、请求生命周期、扩展点 |
| 02 | [02-modules.md](02-modules.md) | 模块参考(每个包、每个顶层文件) |
| 03 | [03-tools.md](03-tools.md) | 工具注册表、内置工具、终端 Backend、Skills 概览 |
| 04 | [04-gateway.md](04-gateway.md) | 消息网关、平台适配器、会话、投递、Cron 闭环 |
| 05 | [05-state-and-memory.md](05-state-and-memory.md) | SessionDB、记忆、上下文引擎、Curator |
| 06 | [06-providers.md](06-providers.md) | Provider Transport、Adapter、凭证池、限流、计费 |
| 07 | [07-developer-guide.md](07-developer-guide.md) | 开发环境、测试、贡献流程、发布 |
| 08 | [08-cli-reference.md](08-cli-reference.md) | CLI 全部子命令、61 个 slash 命令、键绑定 |
| 09 | [09-skills-system.md](09-skills-system.md) | Skill 文件格式、Hub、Guard、Sync、Curator(深度) |
| 10 | [10-batch-and-rl.md](10-batch-and-rl.md) | 批量轨迹生成、压缩、SWE Runner、Atropos RL |
| 11 | [11-mcp-and-acp.md](11-mcp-and-acp.md) | MCP(客户端 + 服务端)、ACP 服务端 |
| 12 | [12-tui-and-dashboard.md](12-tui-and-dashboard.md) | Ink TUI、tui_gateway、Web Dashboard、PTY 桥 |
| 13 | [13-security.md](13-security.md) | 威胁模型、审批、注入扫描、容器隔离、密钥脱敏 |
| 14 | [14-configuration-reference.md](14-configuration-reference.md) | 完整 `config.yaml` + `.env` 参考 |
| 15 | [15-data-flow.md](15-data-flow.md) | 20 张 ASCII 时序 / 数据流图 |
| 16 | [16-extension-recipes.md](16-extension-recipes.md) | 25 个扩展菜谱(新工具、新 Provider、新平台等) |
| 17 | [17-glossary-and-troubleshooting.md](17-glossary-and-troubleshooting.md) | 术语表、常见故障排查、定位索引 |
| 18 | [18-platforms-deep-dive.md](18-platforms-deep-dive.md) | 22 个平台适配器逐一详解 |
| 19 | [19-testing-and-observability.md](19-testing-and-observability.md) | 测试基础设施、日志、指标、追踪 |
| 20 | [20-voice-and-image.md](20-voice-and-image.md) | 语音模式、TTS、STT、图像生成流水线 |
| 21 | [21-browser-tools.md](21-browser-tools.md) | 浏览器工具家族、CDP、Provider 抽象、反爬 |
| 22 | [22-delegation-and-subagents.md](22-delegation-and-subagents.md) | 委派、子 Agent、MoA、Clarify、并行性 |
| 23 | [23-patterns-and-antipatterns.md](23-patterns-and-antipatterns.md) | 代码模式与反模式合集(reviewer 检查清单) |
| 24 | [24-adapters-deep-dive.md](24-adapters-deep-dive.md) | Anthropic / Codex / Bedrock / Gemini / Copilot 等 Adapter 逐项细节 |
| 25 | [25-fakes-and-test-harness.md](25-fakes-and-test-harness.md) | `tests/fakes/` 用法与 fixture 组合模式 |

## 按使用场景索引

### 我想了解 Hermes 整体架构

依次阅读 [00](00-overview.md) → [01](01-architecture.md) → [02](02-modules.md) →
[15](15-data-flow.md)。前两份给你术语和心智模型;模块参考是地图;数据流图把所有模块联系起来。

### 我想新增一个工具

阅读 [03-tools.md](03-tools.md) 第 1 节(注册表),然后 [16-extension-recipes.md](16-extension-recipes.md) 配方 1。

### 我想接入一个新的 LLM Provider

* 兼容 OpenAI 协议:[16](16-extension-recipes.md) 配方 2(只配置即可)。
* 自定义协议:[06-providers.md](06-providers.md) + [16](16-extension-recipes.md) 配方 3。

### 我想新增一个聊天平台(Telegram、Discord 之外)

阅读 [04-gateway.md](04-gateway.md) → [18-platforms-deep-dive.md](18-platforms-deep-dive.md) → [16](16-extension-recipes.md) 配方 4。
仓库根目录的 `gateway/platforms/ADDING_A_PLATFORM.md` 是 PR 检查清单。

### 我想加一个记忆 Provider 或 Context Engine

阅读 [05-state-and-memory.md](05-state-and-memory.md) → [16](16-extension-recipes.md) 配方 5–6。

### 我想给 CLI 加一个 slash 命令

[08-cli-reference.md](08-cli-reference.md) → [16](16-extension-recipes.md) 配方 9。

### 我想为我的项目定时让 Hermes 跑某个任务

[04-gateway.md](04-gateway.md) §9 + [16](16-extension-recipes.md) 配方 10。

### 我想集成到 VS Code / Zed / Cursor

* 编辑器要把 Hermes 当对话 Agent:[11-mcp-and-acp.md](11-mcp-and-acp.md) 第 3 节(ACP)。
* 编辑器只想用 Hermes 的会话搜索:[11](11-mcp-and-acp.md) 第 2 节(MCP)。

### 我想生成训练数据 / 跑 RL

[10-batch-and-rl.md](10-batch-and-rl.md) 起步,然后看 `environments/README.md` 与
`tinker-atropos/`、`datagen-config-examples/` 中的样例配置。

### 我在生产环境部署,关心安全性

[13-security.md](13-security.md) → [14-configuration-reference.md](14-configuration-reference.md) → 仓库根目录的 [SECURITY.md](../../SECURITY.md)。

### 出问题了,要先看哪里

* 概览症状 → 解决:[17-glossary-and-troubleshooting.md](17-glossary-and-troubleshooting.md) Part 2。
* 「报错信息出现在哪里」:[17](17-glossary-and-troubleshooting.md) Part 3「Quick locate index」。
* 日志:`~/.hermes/logs/agent.log`、`errors.log`、`gateway.log`。

## 关键文件速查

| 我要看的文件 | 作用 |
|--------------|------|
| `run_agent.py` | `AIAgent` 主循环(~14k 行) |
| `cli.py` | `HermesCLI` 交互式 REPL(~12k 行) |
| `model_tools.py` | 工具调度,异步桥接 |
| `toolsets.py` | Toolset 定义 |
| `hermes_state.py` | `SessionDB`(SQLite + FTS5) |
| `hermes_constants.py` | `get_hermes_home()`,Profile 路径 |
| `agent/transports/` | Provider 协议适配 |
| `agent/<provider>_adapter.py` | 厂商特殊处理 |
| `agent/context_compressor.py` | 默认上下文压缩 |
| `agent/curator.py` | 自治 Skill 维护 |
| `agent/credential_pool.py` | 多凭证轮转 |
| `tools/registry.py` | 工具注册中心(单例) |
| `tools/environments/` | 终端 Backend(local/docker/ssh/modal/...) |
| `gateway/run.py` | `GatewayRunner` |
| `gateway/platforms/` | 22 个平台适配器 |
| `gateway/delivery.py` | `DeliveryRouter` |
| `cron/scheduler.py` | Cron 调度器 |
| `cron/jobs.py` | 任务存储 |
| `acp_adapter/server.py` | ACP 服务端 |
| `mcp_serve.py` | MCP 服务端(stdio) |
| `tools/mcp_tool.py` | MCP 客户端 |
| `tui_gateway/server.py` | Python TUI 后端 |
| `ui-tui/src/app.tsx` | Ink TUI 前端 |
| `hermes_cli/web_server.py` | Dashboard FastAPI |
| `hermes_cli/main.py` | `hermes` 子命令分发 |
| `hermes_cli/commands.py` | `COMMAND_REGISTRY`(slash 命令) |
| `hermes_cli/config.py` | `DEFAULT_CONFIG`、`OPTIONAL_ENV_VARS` |
| `hermes_logging.py` | 集中日志 + 脱敏 |
| `utils.py` | 原子写、布尔解析等工具函数 |

## 用户视角的入口

* 仓库根 `README.md` — 用户视角的快速上手。
* `https://hermes-agent.nousresearch.com/docs/` — 完整用户文档(Docusaurus 站点,源在 `website/`)。
* `AGENTS.md` — AI 编码助手 / 贡献者必读;包含本文档以外的实战经验法则。
* `CONTRIBUTING.md` — 贡献流程、代码风格、PR 模板。
* `SECURITY.md` — 安全披露策略与威胁模型摘要。

## 版本

本批文档对应 `pyproject.toml` 中 `version = "0.12.0"`。Hermes 处于
快速迭代期,重大重构可能让 file:line 引用失效;**文件名/类名/函数名
基本稳定**,可作为最可靠的检索路径。

## 反馈

发现错误、过时或遗漏?在 GitHub 上提 Issue 或 PR。整套文档以纯 Markdown
保存于 `docs/technical/`,直接 `git diff` 即可改。

—

> 本索引是英文文档的目录与导航。文档正文未中文化,因为代码、错误信息、
> 配置键名都是英文,中英对照反而会造成歧义。如需汉化某一章节,欢迎
> 单独提 PR(请保留对应的英文原文版本以便国际协作)。
