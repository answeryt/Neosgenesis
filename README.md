# Neosgenesis (Multi-Agent Framework)

<p align="center">
  <img src="Image 2025年11月16日 16_14_49.png" alt="neosgenesis logo" width="220">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/python-3.10%2B-3776AB?logo=python&logoColor=white" alt="Python 3.10+">
</p>

<p align="center">
  把复杂的上下文工程「文件化、模板化、可视化」，让多阶段多 Agent 工作流变得可理解、可复制、好调试。
</p>

---

## 目录

- [项目简介](#项目简介)
- [整体架构与流程](#整体架构与流程)
- [文档驱动与上下文检索机制](#文档驱动与上下文检索机制)
- [模型封装与调用说明](#模型封装与调用说明)
- [目录结构概览](#目录结构概览)
- [安装与配置](#安装与配置)
- [快速开始：运行全流程](#快速开始运行全流程)
- [能力库与策略库维护建议](#能力库与策略库维护建议)
- [测试与基准评估](#测试与基准评估)
- [适用场景与扩展方向](#适用场景与扩展方向)

---

## 项目简介

`neosgenesis` 是一个以 **多阶段、多智能体协作** 为核心的自动问题求解与自我进化系统。  
它希望把原本“黑箱”的 **上下文工程（prompt / tool / 文档编排）透明化、文件化、模板化**，让开发者和普通用户都能**看得见、改得动、复用得起**整条思考链路。

系统将一次任务从「理解问题 → 选择策略 → 生成执行计划 → 执行记录与复盘 → 能力/策略库升级」拆解为多个 Agent，并统一写入一份协作表单（Markdown 文档），通过文档检索与标记块管理上下文，实现类似 RAG 的增强推理，但完全基于文件系统与 Markdown，而非传统向量库。

### 项目特点（为什么它适合做“上下文工程脚手架”）

- **上下文工程完全透明**  
  - 所有提示词、阶段划分、工具调用计划、执行结果，全部落在可视的 Markdown 文档里（`finish_form/*.md`、`ability_library`、`strategy_library` 等）。  
  - 不需要猜“模型背后做了什么”，你可以直接看到：每一步是如何理解任务、选策略、拆步骤、调用工具的。

- **把复杂的 prompt / context 工程拆成可维护的“小文件”**  
  - 用目录 + Markdown + 标记块替代“超长、难维护的 mega prompt”：能力库、策略库、执行计划模板都拆分在独立文件。  
  - 修改或扩展上下文，只需要改对应的 `.md` 文件，而不是重写一大段提示词。

- **面向“小白”的工程化体验：低代码、强结构**  
  - 想跑一个完整的“上下文工程 + 任务求解”流程，只需要：  
    1. 填 `.env`（API Key）；  
    2. 按 README 安装依赖；  
    3. 运行一条命令：`python -m workflow.full_pipeline_runner ...`。  
  - 上手时可以只把它当成一个“可编辑的工作流模板”：不懂 LLM 细节，也可以通过修改 Markdown 和少量配置，做出效果不错的上下文工程。

- **以文档为中心，而不是以代码为中心**  
  - 绝大部分“智能行为的配置”体现在文档结构和内容里（协作表模板、能力条目、策略条目、工具目录），代码更多扮演“执行器”和“文档编辑器”的角色。  
  - 这让团队协作更容易：产品、运营、领域专家可以直接在文档层面参与“上下文工程”的设计，而不必改代码。

- **工程路径简单，但可逐步变“高级”**  
  - 初学者：只需要改模板和文案，就能搭出自己的“多阶段 Agent 工作流”。  
  - 进阶：可以替换模型、增加 MCP 工具、扩展新的 Stage；  
  - 高阶：可以把自己的领域知识沉淀进能力库 / 策略库，构建专属的“团队大脑”。

核心目标：

- **任务求解标准化**：所有阶段在同一份 `finish_form/*.md` 协作表中协作，便于回溯与审计。
- **能力与策略可演化**：通过能力库（`ability_library`）与策略库（`strategy_library`）的升级 Agent，使系统随使用持续进化。
- **工具增强推理**：内置 MCP 工具（如 Tavily 搜索、代码解释器）和工具目录文档，让 Agent 在规划与执行阶段可以“知道自己有哪些工具可用”。

---

## 整体架构与流程

整体流程由 `workflow/full_pipeline_runner.py` 中的 `FullPipelineRunner` 串联，默认使用 DeepSeek 模型：

1. **模板生成与表单定位**
   - 使用 `Document_Checking.TemplateGenerationAgent` 确保 `finish_form/` 下存在足够的标准模板实例。
   - `_prepare_finish_form_document()` 选择最新的 Markdown 文件作为本轮任务协作表单。
   - 从协作表中提取 `## 索引` 段落作为索引快照，为 Stage 1 提供结构化视图。

2. **Stage 1：元能力分析（MetacognitiveAnalysisAgent）**
   - 代码：`stage1_agent/Metacognitive_Analysis_agnet.py`，CLI：`stage1_agent/main.py`  
   - 提示词：`stage1_agent/reasoner.md`
   - 职责：围绕当前任务做**自我能力分析**，输出：
     - `problem_type`（任务类型与挑战点）
     - `required_capabilities`（所需能力，引用 `ability_library/core_capabilities.md` 中的 A1–H2 等编号）
     - `complexity_assessment`、`timeliness_and_knowledge_boundary`、`content_quality` 等字段
   - 上下文来源：
     - 协作表索引快照 + 用户附加上下文（`context_snapshot`）
     - 能力库（按编号检索）
   - 输出写入协作表的 Stage 1 段落（预留标记 `<!-- STAGE1_ANALYSIS_START --> ...` 之间）。

3. **Stage 2-A：候选策略筛选（CandidateSelectionAgent）**
   - 代码：`stage2_candidate_agent/Candidate_Selection_agent.py`，CLI：`stage2_candidate_agent/main.py`  
   - 提示词：`stage2_candidate_agent/selector.md`
   - 职责：根据 Stage 1 输出，在 `strategy_library/strategy.md` 中检索并筛选出 2–3 条候选策略：
     - 引用策略编号（如 P1、I2、D1 等）
     - 说明每个策略如何覆盖 `problem_type` 与 `required_capabilities`
   - 输出写入协作表 “阶段二-A：候选策略产出” 区域（`<!-- STAGE2A_ANALYSIS_START --> ...`）。

4. **Stage 2-B：策略批判与改造（StrategySelectionAgent）**
   - 代码：`stage2_agent/Strategy_Selection_agent.py`，CLI：`stage2_agent/main.py`  
   - 提示词：`stage2_agent/verifire.md`
   - 职责：作为“策略批判家与方法设计师”，对候选策略进行深度评估和改造：
     - 对每个候选策略分析优势、劣势、缺口、能力匹配度；
     - 决定是否融合多个策略，形成唯一的 `refined_strategy`；
     - 生成交接说明(`handover_notes`)，为 Stage 3 提供拆解依据。
   - 输出写入协作表 “阶段二-B：策略遴选” 区域（`<!-- STAGE2B_ANALYSIS_START --> ...`）。

5. **Stage 2-C：策略库升级（Stage2CapabilityUpgradeAgent）**
   - 代码：`stage2_capability_upgrade_agent/stage2_capability_upgrade_agent.py`，CLI：`stage2_capability_upgrade_agent/main.py`
   - 继承自通用 `CapabilityUpgradeAgent`，但指向策略库文件 `strategy_library/strategy.md`：
     - 读取策略库生成快照，注入 system prompt；
     - 结合 Stage 1 + Stage 2-B 的文本，评估是否需要新增/调整策略；
     - 生成 Markdown 补丁并自动写入 `strategy_library/strategy.md`（可配置是否 auto_apply）。
   - 同时通过 `workflow/finish_form_utils.update_form_section` 把评估内容写回协作表 “阶段二-C：能力升级评估” 区域。

6. **Stage 3：执行步骤规划（Stage3ExecutionAgent）**
   - 代码：`stage3_agent/Step_agent.py`，CLI：`stage3_agent/main.py`  
   - 提示词：`stage3_agent/step.md`
   - 职责：将最终策略 `refined_strategy` 拆解为具体可执行计划：
     - 为每个 `key_step` 生成一组子步骤（step_id、actions、expected_output、quality_checks 等）；
     - 映射 Stage 1 中的 `required_capabilities` 和风险到执行步骤；
     - 嵌入工具使用规划（引用工具目录 / MCP 工具，标注工具名称、调用目的和结果挂载路径）。
   - 默认工具目录由 `tool_catalog.load_tool_catalog()` 解析 `tools/tool_catalog.md` 获得，作为 `tool_catalog` 字符串列表传入。
   - 输出写入协作表 “阶段三：执行步骤规划（Step_agent）” 区域。

7. **Stage 4：执行记录与复盘（Stage4ExecutorAgent）**
   - 代码：`stage4_agent/Executor_agent.py`，CLI：`stage4_agent/main.py`  
   - 提示词：`stage4_agent/executor.md`
   - 职责：作为执行阶段的“任务落实代理”，负责：
     - 执行准备检查（对照 Stage 3 前提条件与资源需求）；
     - 逐步执行 `execution_plan.steps`，记录实际耗时、结果、偏差与补救措施；
     - 按 Stage 2/3 的 `failure_indicators` 与 `quality_checks` 监控风险；
     - 在总结部分评估目标达成度，并给上游阶段返回经验与改进建议。
   - 输出写入协作表 “执行阶段：任务落实（Executor）” 区域，并附加 `**最终答案**: ...` 回答用户目标。

8. **能力库升级（CapabilityUpgradeAgent）**
   - 代码：`capability_upgrade_agent/capability_upgrade_agent.py`
   - 职责：面向能力库 `ability_library/core_capabilities.md`：
     - 读取能力库（按 A1–H2 编号组织）形成快照；
     - 根据 Stage1 元分析判断现有能力是否覆盖任务挑战；
     - 必要时生成新的能力定义 Markdown 段落，并追加写入能力库。

最终，`FullPipelineRunner.run()` 会返回一个包含各阶段原始输出的字典，并在终端打印协作表路径及升级提示。

---

## 文档驱动与上下文检索机制

本项目没有使用传统向量检索，而是通过 **文件系统 + Markdown 结构 + 标记块** 来实现上下文管理与“文档检索”：

- **协作表单（finish_form）**
  - 所有阶段共享一份或多份 `finish_form/*.md` 文件，结构由 `form_templates/standard template.md` 定义。
  - `Document_Checking.TemplateGenerationAgent` 确保当文档数量低于阈值（默认 8）时自动复制标准模板生成新的表单，并在文末附加“完成文档位置索引”。
  - `FullPipelineRunner` 通过扫描目录 + 修改时间定位“当前协作表”，并通过正则提取 `## 索引` 段落。

- **标记块读写**
  - 阶段间通过 HTML 注释标记划分写入区域，例如：
    - `<!-- STAGE1_ANALYSIS_START --> ... <!-- STAGE1_ANALYSIS_END -->`
    - `<!-- STAGE2A_ANALYSIS_START --> ...`
    - `<!-- STAGE3_PLAN_START --> ...` 等等。
  - `stage2_agent/main.py` 等脚本使用 `_extract_between_markers` / `_replace_between_markers` 在标记块间读写内容。
  - 通用工具 `workflow/finish_form_utils.update_form_section` 支持：
    - 按 `marker_name` 替换已有块；
    - 若块不存在，则在指定 `header` 后插入；
    - 若 header 也不存在，则追加到文档末尾。

- **能力库与策略库快照**
  - 能力库：`ability_library/core_capabilities.md`  
    - 被 `CapabilityUpgradeAgent` 读取，拼接为“能力库快照”，注入 system prompt。
  - 策略库：`strategy_library/strategy.md`  
    - 被 `Stage2CapabilityUpgradeAgent` 读取，形成“策略库快照”，注入其 `thinking.md` 提示词中。
  - 若快照长度超过配置的 `max_library_chars`，会按字符截断并在尾部附上“内容已截断，请查看原始 Markdown 文件”的提示。

- **工具文档与工具目录**
  - 核心工具说明：`MCP/tool.md`（如 `MCP.code_interpreter`、`MCP.tavily`）  
    用于指导 Stage 3 / Stage 4 在规划或执行时如何选择合适工具。
  - 工具目录：`tools/tool_catalog.md`  
    - 由 `tool_catalog.load_tool_catalog()` 解析成字符串列表（形如 `"Tavily MCP · search: 调用 Tavily 在线搜索..."`），作为 `tool_catalog` 输入传给 Agent。
  - Tavily MCP 集成：`MCP/tavily.py`  
    - 封装 Tavily 官方 MCP Server，提供 `discover_tavily_tool_catalog`、`get_default_tavily_search_tool` 等辅助函数。

通过上述机制，系统实现了一种**以 Markdown 为中心的上下文检索与知识增强**：Agent 通过文件路径、索引小节和标记块来查找上下文，而不是通过嵌入检索。

---

## 模型封装与调用说明

### 抽象基类：ChatModelBase

- 文件：`model/_model_base.py`
- 提供统一接口：
  - 属性：`model_name`、`stream`
  - 抽象方法：`__call__(...) -> ChatResponse | AsyncGenerator[ChatResponse, None]`
  - 工具方法：`_validate_tool_choice`、`_attach_contract_payload`（兼容旧的“payload contract”接口）
- 所有具体模型（DeepSeek、OpenAI 等）均继承此类，Agent 仅需要关心“传入 messages / 工具列表，返回统一的 `ChatResponse`”。

### DeepSeekChatModel：主力后端

- 文件：`model/_deepseek_model.py`
- 功能：
  - 对 DeepSeek Chat Completions API (`/chat/completions`) 进行最小包装；
  - 支持 `reasoning_effort`（`low`/`medium`/`high`），可控制推理强度；
  - 支持超时与指数退避重试（`timeout`、`max_retries`、`retry_base_delay`、`retry_backoff_factor`）。
- 调用方式：
  1. Agent 构造 messages：
     - `{"role": "system", "content": "<系统提示词 + 能力/策略库快照等>"}`
     - `{"role": "user", "content": "<任务描述 + 文档写入指引 + 结构化字段>"}`
  2. 调用模型：
     - `response = await self._model(messages=messages, **kwargs)`
  3. 模型内部：
     - 使用 `httpx.AsyncClient.post()` 发送 JSON 请求；
     - 解析返回 JSON 中的 `message.reasoning_content`（填入 `ResponseBlock(type="thinking")`）和 `message.content`（填入 `ResponseBlock(type="text")`）；
     - 构造 `ChatUsage`（输入/输出 token 统计）；
     - 封装为 `ChatResponse(content=blocks, usage=usage, raw=data)`。
  4. 上层 Agent 根据需要将 `ChatResponse` 转为纯文本（例如 `FullPipelineRunner._normalize_stage_output`）或从中抽取 Markdown 补丁等。

### OpenAIChatModel：扩展与兼容

- 文件：`model/_openai_model.py`
- 提供对 OpenAI Chat API（以及兼容 API，例如部分 Qwen）的一致封装：
  - 支持流式输出（返回 `AsyncGenerator[ChatResponse, None]`）；
  - 支持 tool calling（函数调用）和工具选择模式（`auto`/`none`/`any`/`required` 或具体函数名）；
  - 支持结构化输出（`structured_model: Type[BaseModel]`），通过 Pydantic 自动将输出解析为预定义 schema。
- 当前主流程默认使用 DeepSeek，但已有足够接口支持未来部分阶段迁移到 OpenAI / Qwen 等模型。

---

## 目录结构概览

仅列出与核心流程最相关的目录和文件：

- `workflow/`
  - `full_pipeline_runner.py`：全流程调度器（Stage 1–4 + 能力/策略库升级）。
  - `finish_form_utils.py`：协作表标记块写入工具。
  - `capability_upgrade_workflow.py`：能力升级相关辅助流程（如有）。
- `Document_Checking/`
  - `template_generation.py`：根据标准模板维护 `finish_form/` 文档数量与索引。
- `form_templates/`
  - `standard template.md`：统一协作表结构定义。
  - `template_generation_agent.py`：另一套模板生成封装（面向简单场景）。
- `finish_form/`
  - `auto_generated_template_*.md`：实际协作表单实例（程序运行时不断生成）。
- `ability_library/`
  - `core_capabilities.md`：核心能力库（A1–H2 等）。
- `strategy_library/`
  - `strategy.md`：策略库（P1、I1、D1、E1 等，以及派生策略）。
- `MCP/`
  - `tavily.py`：Tavily MCP 客户端封装。
  - `code_interpreter.py`：代码解释器 MCP 客户端封装。
  - `tool.md`：核心工具使用说明文档。
- `tools/`
  - `tool_catalog.md`：可用工具目录（供 Stage3/4 使用）。
- `stage1_agent/`
  - `Metacognitive_Analysis_agnet.py`：Stage 1 Agent。
  - `reasoner.md`：Stage 1 提示词。
  - `main.py`：交互式命令行入口。
- `stage2_candidate_agent/`
  - `Candidate_Selection_agent.py`：Stage 2-A Agent。
  - `selector.md`：候选策略筛选提示词。
  - `main.py`：命令行入口。
- `stage2_agent/`
  - `Strategy_Selection_agent.py`：Stage 2-B Agent。
  - `verifire.md`：策略批判/选择提示词。
  - `main.py`：Stage 2 流水线（2A→2B→2C）专用脚本。
- `stage2_capability_upgrade_agent/`
  - `stage2_capability_upgrade_agent.py`：策略库升级 Agent。
  - `thinking.md`：Stage 2-C 提示词。
  - `main.py`：命令行入口。
- `stage3_agent/`
  - `Step_agent.py`：Stage 3 Agent。
  - `step.md`：执行步骤规划提示词。
  - `main.py`：命令行入口。
- `stage4_agent/`
  - `Executor_agent.py`：Stage 4 Agent。
  - `executor.md`：执行记录与复盘提示词。
  - `main.py`：命令行入口。
- `capability_upgrade_agent/`
  - `capability_upgrade_agent.py`：通用能力库升级 Agent。
  - `thinking.md`：能力库升级提示词。
- `model/`
  - `_model_base.py`：模型基类。
  - `_deepseek_model.py`：DeepSeek Chat 实现。
  - `_openai_model.py`：OpenAI Chat 实现。
  - `_model_response.py`、`_model_usage.py`：统一响应与用量结构。
- `bamboogle_benchmark/`、`AIME-2025/`
  - 基准测试与评估用数据与脚本（例如 `run_full_pipeline_benchmark.py`）。

---

## 安装与配置

### 环境要求

- Python 3.10+（推荐 3.11 或 3.12）
- 操作系统：Windows / macOS / Linux

### 依赖安装

在项目根目录下（即包含本 README 的目录）执行：

```bash
pip install -r requirements.txt
```

> 若仓库中尚未提供 `requirements.txt`，可根据实际使用的包（如 `httpx`、`python-dotenv`、`openai`、`mcp` 相关依赖等）自行创建。

### 环境变量

项目主要依赖以下环境变量（使用 `python-dotenv` 自动从 `.env` 读取）：

- `DEEPSEEK_API_KEY`：DeepSeek 模型调用所需的 API Key（必需）。
- `TAVILY_API_KEY`：Tavily 搜索服务 API Key（必需，用于访问 Tavily MCP 远程服务，不再提供任何硬编码默认密钥）。
- 其他可选：
  - `TAVILY_MCP_TRANSPORT`、`TAVILY_MCP_URL`、`TAVILY_MCP_HEADERS` 等，用于自定义 Tavily MCP 客户端配置。

建议在项目根目录创建 `.env` 文件，示例：

```env
DEEPSEEK_API_KEY=your_deepseek_key_here
TAVILY_API_KEY=your_tavily_key_here
```

---

## 快速开始：运行全流程

### 1. 使用 FullPipelineRunner（推荐）

在项目根目录运行：

```bash
python -m workflow.full_pipeline_runner --objective "用一句话描述你的任务目标"
```

可选参数：

- `--context`：附加上下文说明；
- `--candidate-limit`：Stage 2-A 候选策略数量上限；
- `--finish-dir`：自定义 `finish_form` 目录；
- `--template`：自定义标准模板路径；
- `--encoding`：协作表编码（默认 `utf-8`）；
- `--api-key` / `--model` / `--base-url` / `--reasoning-effort`：模型配置；
- `--no-strategy-auto-apply`：禁用 Stage 2-C 策略库自动写入；
- `--auto-apply-capability`：启用能力库自动写入。

运行结束后终端会输出：

- 协作表路径（相对项目根目录），可在编辑器中直接查看多阶段输出；
- 是否生成策略库/能力库升级补丁的提示。

### 2. 单阶段交互调试

如果你只想调试某个阶段，可以直接运行对应 Agent 的 CLI：

- Stage 1：

```bash
python -m stage1_agent.main
```

- Stage 2-A 候选策略：

```bash
python -m stage2_candidate_agent.main
```

- Stage 2 全流水线（2A→2B→2C）：

```bash
python -m stage2_agent.main
```

- Stage 2-C 策略库升级（单独运行）：

```bash
python -m stage2_capability_upgrade_agent.main
```

- Stage 3 执行步骤规划：

```bash
python -m stage3_agent.main
```

- Stage 4 执行记录与复盘：

```bash
python -m stage4_agent.main
```

> 这些 CLI 均为交互式，会在终端提示输入目标、上下文、JSON 片段等，适合单步调试和 prompt 调优。

---

## 能力库与策略库维护建议

- **能力库（`ability_library/core_capabilities.md`）**
  - 按分类 A–H（语言、推理、知识、工具、验证、创新、跨模态、伦理与安全）维护能力条目；
  - 每条能力包含：编号、适配问题类型、能力说明、典型示例；
  - 所有新增能力应通过 `CapabilityUpgradeAgent` 生成 Markdown 段落再追加，避免手工编辑破坏结构。

- **策略库（`strategy_library/strategy.md`）**
  - 按 P、I、X、C、F、D、E、L 等分类维护策略：
    - 例如规划策略（P1/P2）、信息管理（I1–I4）、决策策略（D1/D2）、执行与验证（E1）、复盘（L1）等；
  - Stage 2-A / 2-B 会引用策略编号和名称；
  - 策略库的新增/修改应通过 Stage 2-C 升级 Agent 完成。

---

## 测试与基准评估

- `test/` 目录中包含若干单元测试：
  - `test_capability_upgrade_agent.py`
  - `test_stage2_capability_upgrade_agent.py`
  - `test_strategy_selection_workflow.py`
  - 等等，可用于验证核心 Agent 的行为。
- `bamboogle_benchmark/` 与 `AIME-2025/` 提供了一些外部基准数据与评估脚本（例如 `run_full_pipeline_benchmark.py`），可以用来比较不同模型配置、策略或能力库版本下的表现。

---

## 适用场景与扩展方向

适用场景：

- 复杂任务拆解与执行（如项目排期、研究问题求解、系统设计与验证等）；
- 需要明确“Agent 自我能力边界”的高风险任务（如事实核验、合规与伦理评估）；
- 需要形成长期可追溯知识库（能力/策略）的多轮协作场景。

扩展方向：

- 将部分阶段切换到其他大模型（例如 OpenAI、Qwen），利用 `OpenAIChatModel` 的 tool calling 或结构化输出能力；
  - 在配置中调整 `model_name` 与 `base_url` 即可；必要时扩展 Agent 的配置结构。
- 引入更多 MCP 工具（代码执行、数据库查询、内部文档搜索等），并在 `tools/tool_catalog.md` 与 `MCP/tool.md` 中完善说明；
- 对协作表模板进行定制，以适配特定领域（如医学、法律、金融）的规范化输出。

如果你在接入自己的任务或模型时遇到问题，可以优先查看：

- `workflow/full_pipeline_runner.py` 中的调度逻辑；
- 对应阶段 Agent 的 `*_agent.py` 文件与提示词 `.md` 文件；
- `model/` 目录中模型封装与返回结构定义。



