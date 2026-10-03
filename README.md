# ThreeStepAgent_LangGraph

基于 [LangGraph](https://github.com/langchain-ai/langgraph) 构建的**三步式（Three-Step）联网搜索问答 Agent**。

它把「理解问题 → 联网搜索 → 生成答案」拆成三个清晰的节点，用一张有状态的图把它们串起来，并内置了搜索失败时的降级策略。

---

## 一、项目概览

一个最小但完整的 RAG-style Agent 示例：

1. **理解查询**：让大模型读懂用户意图，并生成适合搜索引擎的关键词。
2. **联网搜索**：调用 [Tavily](https://tavily.com) API 获取真实、实时的网页结果。
3. **生成答案**：把搜索结果作为上下文交给大模型，综合生成最终回答。

相比「直接问大模型」，它能回答训练数据之外、需要实时信息的问题；相比「直接搜索」，它由 LLM 完成意图理解与结果归纳，答案更自然、更聚焦。

---

## 二、核心概念

| 概念 | 说明 |
| --- | --- |
| **State（状态）** | 贯穿整个流程的共享数据结构 `SearchState`，每个节点读取并更新它 |
| **Node（节点）** | 一个执行具体逻辑的函数，接收 state、返回要更新的字段 |
| **Edge（边）** | 定义节点之间的流转关系，本项目为线性流程 |
| **Checkpointer（检查点）** | 用 `InMemorySaver` 保存运行状态，支持多轮会话记忆 |

### 工作流

```text
        START
          │
          ▼
  ┌────────────────┐
  │  understand    │  步骤1：理解查询，生成搜索词
  │  (LLM)         │
  └────────────────┘
          │
          ▼
  ┌────────────────┐
  │     search     │  步骤2：调用 Tavily 联网搜索
  │  (Tavily API)  │
  └────────────────┘
          │
          ▼
  ┌────────────────┐
  │     answer     │  步骤3：基于结果生成答案
  │  (LLM)         │   └─ 若搜索失败 → 回退到模型自身知识
  └────────────────┘
          │
          ▼
         END
```

---

## 三、状态定义（`SearchState`）

`agent.py` 中定义的 `SearchState` 是全局共享的「白板」，字段含义如下：

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `messages` | `Annotated[list, add_messages]` | 对话消息列表。`add_messages` 让新消息**追加**而非覆盖，用于维护会话历史 |
| `user_query` | `str` | 经 LLM 理解后的用户需求总结 |
| `search_query` | `str` | 优化后用于 Tavily API 的搜索关键词 |
| `search_results` | `str` | Tavily 返回并格式化后的搜索结果文本 |
| `final_answer` | `str` | 最终生成的答案 |
| `step` | `str` | 步骤标记（`understood` / `searched` / `search_failed` / `completed`），供后续节点做分支判断 |

---

## 四、模块详解

### 1. `understand_query_node` —— 理解查询

`agent.py:40`

- **输入**：`state["messages"]` 中最后一条用户消息。
- **逻辑**：用一个提示词让 LLM 同时完成「总结需求」和「生成搜索词」，并按固定格式输出。
- **解析**：从返回文本中按 `搜索词：` 切分提取关键词；若解析失败，则退回使用原始问题。
- **输出**：写入 `user_query`、`search_query`、`step="understood"`，并追加一条 AI 消息。

> 意义：把自然语言问题翻译成搜索引擎更容易命中的查询，是提升检索质量的关键一步。

### 2. `tavily_search_node` —— 联网搜索

`agent.py:68`

- **输入**：`state["search_query"]`。
- **逻辑**：调用 `tavily_client.search(...)`，参数为 `search_depth="basic"`、`max_results=5`、`include_answer=True`。
- **结果格式化**：先取 Tavily 的 `answer` 作为摘要，再遍历 `results`，把每条结果的标题、正文摘要、来源 URL 拼成可读文本。
- **异常处理**：整个调用包在 `try/except` 中；失败时写入 `step="search_failed"`，让下游节点降级。
- **输出**：`search_results`、`step`、一条 AI 消息。

> 意义：这是 Agent「连接真实世界」的入口，也是唯一依赖外部网络与 API Key 的环节。

### 3. `generate_answer_node` —— 生成答案

`agent.py:104`

- **分支判断**：读取 `state["step"]`。
  - `search_failed` → **降级策略**：提示模型基于自身知识回答，保证搜索挂了也能给出结果。
  - 否则 → 把 `user_query` 和 `search_results` 一起交给模型，要求综合搜索结果作答。
- **输出**：`final_answer`、`step="completed"`，并追加 AI 消息。

> 意义：把「检索」和「生成」解耦，同时提供优雅降级，提升系统鲁棒性。

### 4. `create_search_assistant` —— 组装并编译图

`agent.py:127`

- 创建 `StateGraph(SearchState)`。
- `add_node` 注册三个节点。
- `add_edge` 定义线性顺序：`START → understand → search → answer → END`。
- `compile(checkpointer=InMemorySaver())` 编译成可执行图，并附带内存级状态保存。

### 5. 入口 `__main__`

`agent.py:146`

- 构建图，设置 `thread_id` 作为会话标识。
- 读取用户输入（为空时用默认问题）。
- `app.invoke(...)` 执行整个流程，最后打印 `final_answer`。

---

## 五、环境变量

复制 `.env.example` 为 `.env` 并填写：

| 变量 | 说明 |
| --- | --- |
| `LLM_MODEL_ID` | 模型名，默认 `gpt-4o-mini` |
| `LLM_API_KEY` | 大模型 API Key |
| `LLM_BASE_URL` | 兼容 OpenAI 协议的接口地址，默认 `https://api.openai.com/v1` |
| `TAVILY_API_KEY` | Tavily 搜索 API Key |

> `.env` 已被 `.gitignore` 忽略，密钥不会被提交。

---

## 六、快速开始

### 方式 A：使用 uv（推荐）

```bash
# 创建虚拟环境
uv venv

# 安装依赖（Windows 用 .venv\Scripts\Activate.ps1 激活）
uv pip install -r requirements.txt

# 运行
uv run python agent.py
```

### 方式 B：使用 pip

```bash
python -m venv .venv
.venv\Scripts\activate        # Windows
# source .venv/bin/activate   # macOS / Linux
pip install -r requirements.txt
python agent.py
```

运行后输入问题，例如「LangGraph 是什么？」，程序会依次搜索并输出最终答案。

---

## 七、项目结构

```text
ThreeStepAgent_LangGraph/
├── agent.py          # 核心：状态定义、三个节点、图组装与入口
├── requirements.txt  # Python 依赖
├── .env.example      # 环境变量模板
├── .gitignore        # 忽略 .env / .venv / __pycache__ 等
└── README.md
```

---

## 八、设计亮点

- **关注点分离**：理解、检索、生成各自独立，便于替换与测试。
- **优雅降级**：搜索失败自动回退到模型知识，不会中断对话。
- **状态可观测**：`step` 字段清晰标记流程所处阶段。
- **可扩展**：可轻松加入条件边（conditional edge）、多轮反思、多 Agent 协作等。

## 九、可扩展方向

- 用条件边让 `answer` 节点在结果不足时**回到 `search`** 重试。
- 接入长期记忆 / 数据库 Checkpointer，实现跨会话持久化。
- 增加网页抓取（crawler）节点，获取搜索结果的全文。
- 支持多工具调用（计算器、数据库、代码执行等）。

## 十、许可证

MIT
