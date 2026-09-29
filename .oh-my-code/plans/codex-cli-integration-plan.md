# Codex CLI 接入方案与实施记录

> 状态：实现已完成并进入自测收尾。目标是 TradingAgents-Astock 将本机 Codex CLI 作为可选 LLM 后端。

## 目标和边界

假设“Codex CLI 能接入”指 **TradingAgents-Astock 把本机 Codex CLI 当作可选 LLM 后端**，继续通过现有 CLI、Web UI 和 Python 配置运行 A 股投研。若实际需要的是“在 Codex CLI 中调用投研能力”，应改为 MCP/Skill 入口方案，不能直接套用本方案。

- 保留现有分析师、辩论、决策、数据源、历史报告、checkpoint、所有现有 provider 和 Claude Agent SDK 路径的默认行为。
- `codex_cli` 是显式选择的可选 provider；默认配置不切换。第一版支持本机 ChatGPT 登录，API key 模式必须显式选择并告知按 API 计费。
- 不把 Codex 线程与 LangGraph checkpoint 混为同一状态；每次模型调用的完整输入来自 LangGraph 当前消息状态。
- 这一版不重写投研图、不新增交易能力、不迁移到 OpenAI API，也不使用已移除的 `codex mcp-server`。

## 已确认的代码事实

- `tradingagents/llm_clients/factory.py:16-66` 是 provider 唯一工厂，可惰性加载新客户端。
- `tradingagents/graph/trading_graph.py:201-305` 构造 quick/deep 客户端；`:350-452` 支持 `role_llms`；`:460-523` 统一传递超时和 provider 参数。
- `tradingagents/agents/analysts/market_analyst.py:74-108` 等 7 个分析师依赖 `prompt | llm.bind_tools(tools)`、`AIMessage.tool_calls` 和既有 LangGraph `ToolNode` 循环；`tradingagents/graph/setup.py:153-175` 定义该循环。
- `tradingagents/agents/utils/structured.py:31-73` 及 `tradingagents/agents/schemas.py:60-191` 要求 `with_structured_output(...).invoke(...)` 返回 Pydantic 对象。普通角色调用 `.invoke(...)` 并读取 `.content`。
- `tradingagents/llm_clients/claude_agent_sdk_client.py:189-234` 已处理字符串、LangChain 消息和 PromptValue；`:353-451` 展示三种调用形态的兼容接口。Claude 的工具桥接在 SDK 内循环，Codex 第一版采用**现有 LangGraph 外部 ToolNode 循环**，避免两套工具执行状态。
- Web 入口在 `web/components/sidebar.py:201-265`、`web/app.py:159-196`；交互式 CLI 入口在 `cli/utils.py:244-295`、`cli/main.py:561-622`；模型选项在 `tradingagents/llm_clients/model_catalog.py:11-153`。
- 依赖要求 Python >=3.10，`pyproject.toml:44-48` 已有可选 `[agentsdk]`；Codex 应独立可选，不影响基础安装。
- 本机 2026-09-29 的 `codex --version` 为 `0.153.4`，`codex login status` 显示 ChatGPT 登录。当前官方文档已有更新版本和 GPT-6 模型，模型可用性必须以实际 CLI、账号和工作区为准。

## 技术路线

1. **先做小型兼容性探针。** 在隔离的临时工作目录里，以 `codex exec --sandbox read-only --ephemeral --skip-git-repo-check --json` 验证三种结果：普通文本、`--output-schema` 的结构化 JSON、分析师工具调用协议。输入从 stdin 传，进程用参数数组启动，禁止 shell 拼接；捕获最终消息、退出码和 JSONL 错误事件。探针只用合成股票数据，不发真实行情请求。
2. **新增 `tradingagents/llm_clients/codex_cli_client.py`。** 实现 `BaseLLMClient`，返回 LangChain 兼容对象：`invoke` 返回 `AIMessage(content=...)`；`with_structured_output` 把三个现有 Pydantic schema 转为 Codex 可接受的 JSON Schema，读取结果后再 `model_validate`；`bind_tools` 返回 `Runnable`，要求 Codex 的一次回复明确选择“最终报告”或“工具调用”，工具名和参数先按绑定工具白名单及 Pydantic schema 校验，再生成 `AIMessage(tool_calls=...)` 交给现有 `ToolNode`。下一轮调用从 LangGraph 传入的 ToolMessage 恢复上下文。不要让 Codex 的内置 shell/文件工具替代项目行情工具。
3. **隔离与生命周期。** Codex 子进程以无项目文件的临时目录为 cwd，并使用 read-only sandbox。read-only 本身只限制写入，不足以隔离本机可读文件；因此每次调用还关闭 shell、MCP、插件、浏览器、图像、web search 等内置工具，并通过 `--ignore-user-config` 隔离用户级工具配置。启动前检查当前 CLI 是否暴露这些禁用开关；不满足就失败关闭。超时基于 `llm_timeout` 对每次 `codex exec` 生效，超时/取消时结束子进程并等待回收；限制 stdout/stderr 长度，错误日志脱敏，不记录 prompt、登录凭据或完整行情文本。不要用 `--yolo`。
4. **认证与计费边界。** 启动前检查 CLI 存在及登录状态；默认 ChatGPT 模式要求检测到 ChatGPT 登录，传给子进程的环境中清理 `OPENAI_API_KEY` / `CODEX_API_KEY` 等 API key 变量。显式 API key 模式读取 `CODEX_API_KEY` 或 `OPENAI_API_KEY`，但只将其以 `CODEX_API_KEY` 名称传给该进程，按量计费。认证失败、模式不符和额度耗尽时明确报错；不自动切换到现有 `openai` provider。具体账户可用模型及限额需以 CLI 实测。
5. **接入配置与界面。** 在工厂加 `codex_cli` 惰性分支；将 Codex 专属参数（CLI 路径、认证模式、模型、可选推理档位）与 OpenAI API 的 `base_url`、`api_key`、重试参数区分。Web 和交互式 CLI 增加 Codex CLI 选项及登录提示；模型允许“使用本机 CLI 默认值”和手填 ID，避免将当前滚动发布的模型固化为唯一可用值。Web 配置持久化应往返无损。`role_llms` 若选 `codex_cli` 也走同一客户端；用户原有 Claude override 仍按原逻辑优先。
6. **文档与依赖。** 更新 `README.md`、`README_en.md`、`.env.example` 和可选安装说明；若直接用已安装 `codex` 可执行文件，则不增加 Python 运行时依赖。文档清楚区分 `openai`（API）、`codex_cli`（本机 CLI 当前登录身份）与 `claude_agent_sdk`（Claude 订阅），包含登录、选择 provider、模型、超时、额度和排障示例。不要把用户 token 写进配置文件。

## 关键实现判断

- 首选 `codex exec`：官方 OpenAI 文档为其提供非交互模式、JSONL 事件、`--output-schema` 与最终消息文件；在当前 Python 项目中只需一个可选可执行文件。`openai-codex` Python SDK 是后续可选优化，官方文档称其稳定且随包携带匹配的 CLI runtime，但 SDK 的线程/工具调用能力需要另外验证，不能先假定可替换 LangGraph 工具循环。
- 工具调用应使用一个固定的严格 JSON 信封（例如 `kind`, `content`, `tool_calls`），复杂工具参数可以是 JSON 字符串并在本地解析及校验，以避开动态 JSON Schema 的兼容性问题。信封与 Pydantic 输出 schema 都必须通过真实 CLI 探针；若 CLI 对 schema 不兼容，先修正 schema 适配，不退化成用正则从自由文本猜工具调用。
- 每次 LLM 调用开新 `codex exec`，通过 LangGraph 消息传上下文，先保证 checkpoint 恢复正确。延迟和 CLI 启动开销是已知风险；完成正确性验证后再评估长驻 app-server/SDK，不能为了速度改变外部工具循环的语义。
- Codex 是代码代理，内置指令可能影响投研任务。用合成数据比较 3 类节点的输出质量，并检查没有写文件、执行交易或引入未授权数据源；提示词只能作为约束，不能替代沙箱和输出校验。

## 实施顺序及验收

| 步骤 | 交付 | 必须通过的验收 |
| --- | --- | --- |
| 0 基线 | 记录当前测试结果和 CLI 能力探针 | `python3 -m pytest tests -q` 的原有失败被记录；普通/结构化/工具信封探针均能在所用 CLI 版本跑通，否则先修订路线 |
| 1 核心适配 | 新客户端、工厂分支、配置边界 | 三种 `.invoke` 形态可用；工具名/参数非法时拒绝执行；超时、取消、非零退出、空回复和认证错误有明确结果；未选择 Codex 时不导入/启动它 |
| 2 入口和文档 | Web/CLI 选择、持久化、安装说明 | CLI 与 Web 都能选择 Codex；默认/现有 provider 的 UI、配置和启动行为不变；ChatGPT 与 API key 模式明确区分 |
| 3 回归与演练 | 单元测试和一条最小端到端分析 | 现有全量测试通过；新增测试覆盖消息序列化、三种调用、ToolNode 循环、结构化决定、子进程清理、认证/计费护栏、`role_llms`、Web 配置往返；用合成数据跑出完整报告和决策，checkpoint 恢复后不丢工具结果 |

适用质量维度：**向后兼容**（工厂、配置、CLI/Web 和 LangGraph 均为现有公开行为）、**进程/资源安全**（超时和取消必须回收子进程）、**I/O 与延迟**（每节点/工具轮次启动 CLI，记录最小演练耗时）；性能优化先以实测为据。此任务不改共享并发状态；无需并发竞态测试。可选功能如长驻 SDK、Codex 只覆盖 deep 节点、项目作为 MCP server 均不纳入第一版。

## GPT-6 Luna 执行委托（已完成）

按上面 0→3 顺序实施的委托已由 GPT-6 Luna 执行；主 agent 随后审查并补上了内置工具禁用、API Key 环境变量适配和 strict JSON Schema 转换，再执行实时 CLI 探针及回归测试。代码未提交，留给用户 review。

## 官方参考（核对日期：2026-09-29）

- [OpenAI Docs：Codex SDK](https://learn.chatgpt.com/docs/codex-sdk)
- [OpenAI Docs：Codex 非交互模式](https://learn.chatgpt.com/docs/non-interactive-mode)
- [OpenAI Docs：CLI 命令](https://learn.chatgpt.com/docs/developer-commands)
- [OpenAI Docs：Codex App Server](https://learn.chatgpt.com/docs/app-server)
- [OpenAI Docs：定价与认证模式](https://learn.chatgpt.com/docs/pricing)

## 实施与自测结果（2026-09-29）

- 新增 `codex_cli` provider，接入 factory、quick/deep 配置、`role_llms`、Web 侧栏和交互式 CLI；不改变默认 provider，不新增 Python 运行时依赖。
- 新客户端通过 `codex exec` 支持普通回复、strict JSON Schema 输出和 LangGraph 工具调用信封；Pydantic schema 的引用会内联，所有对象字段按 Codex strict schema 要求闭合，并在本地再次校验。
- 子进程关闭内置 shell、MCP、插件、浏览器、图像、web search 等工具，限制到空临时工作目录和 read-only sandbox；认证环境仅透传运行所需变量，ChatGPT 模式清掉 API keys，API 模式只注入 `CODEX_API_KEY`。支持超时/进程回收、输出大小限制、错误脱敏和 token/call 统计。
- 真实 CLI 探针：本机 Codex CLI `0.153.4`；合成 canary 验证关闭 shell 后无法读取外部测试文件；项目 `PortfolioDecision` strict 结构化输出与 `get_stock_data(ticker=600000)` 工具信封均成功。
- GPT-6 Sol 独立审查指出 stdin 大输入写入未受超时约束，以及 Claude 降级到 Codex 时丢失认证配置；两处已修复，并新增阻塞写入、子进程提前退出和降级认证回归测试。Sol 复核通过，未发现新的明确问题。
- 完整回归：`.venv/bin/python -m pytest tests/ -q` —— **527 passed, 14 skipped, 49 subtests passed, 5 warnings**。跳过项是项目已有的可选 Gemini / Claude SDK 依赖；5 条是现存 provider model catalog 警告。
- 当前账号拒绝显式 `gpt-6-sol` 与 `gpt-6-luna`，而 Codex CLI 内置默认模型探针可用；用户应留空使用 CLI 内置默认模型，或指定当前账号支持的 model ID。每次节点调用都启动 Codex 子进程，输入 token 开销高于直接 API client。Codex CLI 本身管理输出上限与网络重试，项目 `max_tokens`、`llm_max_retries` 和 `llm_retry_delay` 尚未映射到该 provider。
