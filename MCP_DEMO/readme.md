# MCP_DEMO 项目详解

> 本项目是 **AI 大模型工程师 · 11_基于 MCP 的 Agent 开发** 课程的配套示例代码。从最基础的 **Function Calling** 出发，逐步演进到 **LangChain Agent → LangGraph Agent → MCP（Model Context Protocol）服务端 / 客户端**，完整演示"如何让大模型调用外部工具"这一主线技术栈。
>
> 通俗理解：整个项目就是讲一件事 —— **怎么让大模型"长出手脚"，能查天气、能算数学、能读文件**。4 个目录就像是 4 个版本升级的"技能点"。

---

## 📑 目录（点击跳转）

| 章节 | 内容简介 |
|------|---------|
| [1. 项目目录结构（详细版）](#1-项目目录结构详细版) | 4 个目录 + 通用文件的整体结构与定位 |
| [2. 环境与依赖](#2-环境与依赖) | Python 版本、依赖包、`.env` 配置 |
| [3. 全局公共模块（项目最底层的"基础设施"）](#3-全局公共模块项目最底层的基础设施) | `env_utils.py`、`zhipu_ai.py` 详解 |
| [4. ① FC 模块：手把手教你"调一次工具"](#4--fc-模块手把手教你调一次工具) | `fc_demo.py` / `fc_demo2.py` / `fc_demo3.py` |
| &nbsp;&nbsp;&nbsp;&nbsp;[4.1 fc_demo.py —— OpenAI 风格 + 假天气](#41-fc_demopy--openai-风格--假天气) | 工具调用的 4 步流程图 |
| &nbsp;&nbsp;&nbsp;&nbsp;[4.2 fc_demo2.py —— 工具内部接真搜索](#42-fc_demo2py--工具内部接真搜索) | 把假数据换成智谱真实搜索 |
| &nbsp;&nbsp;&nbsp;&nbsp;[4.3 fc_demo3.py —— 智谱自带联网搜索](#43-fc_demo3py--智谱自带联网搜索) | 启用平台内置 web_search 工具 |
| [5. ② Agent 模块：让模型"自己决定"调什么](#5--agent-模块让模型自己决定调什么) | `zhipu_demo.py` LangChain Agent + 多轮记忆 |
| [6. ③ LangGraph + MCP 模块：流程图 + 远程工具](#6--langgraph--mcp-模块流程图--远程工具) | `agent_mcp.py` / `graph_mcp.py` |
| &nbsp;&nbsp;&nbsp;&nbsp;[6.1 agent_mcp.py —— 自动挡版](#61-agent_mcppy--自动挡版) | LangGraph ReAct Agent 调 MCP |
| &nbsp;&nbsp;&nbsp;&nbsp;[6.2 graph_mcp.py —— 手动挡版](#62-graph_mcppy--手动挡版) | 自定义状态图（资源 → 工具） |
| [7. ④ MCP 模块：服务端 + 多种客户端](#7--mcp-模块服务端--多种客户端) | 服务端定义 + 5 个文件的完整说明 |
| &nbsp;&nbsp;&nbsp;&nbsp;[7.1 main.py —— 服务端启动入口](#71-mainpy--服务端启动入口) | 启动命令 + import 注册技巧 |
| &nbsp;&nbsp;&nbsp;&nbsp;[7.2 mcp_server.py —— 服务端定义（核心）](#72-mcp_serverpy--服务端定义核心) | FastMCP 实例 + 工具 + 资源 |
| &nbsp;&nbsp;&nbsp;&nbsp;[7.3 mcp_tools.py —— 额外工具（数学）](#73-mcp_toolspy--额外工具数学) | add / multiply 工具注册 |
| &nbsp;&nbsp;&nbsp;&nbsp;[7.4 agent_client.py —— LangChain 版客户端](#74-agent_clientpy--langchain-版客户端) | LangChain + MCP 客户端 |
| &nbsp;&nbsp;&nbsp;&nbsp;[7.5 fastmcp_client.py —— 原生 fastmcp 客户端](#75-fastmcp_clientpy--原生-fastmcp-客户端) | 原生 RPC 调用方式 |
| [8. 整体调用关系图](#8-整体调用关系图) | 客户端 → MCP 服务端 → 工具/资源 的全链路 |
| [9. 启动与运行示例](#9-启动与运行示例) | 服务端启动 + 4 个客户端 + 4 个独立 demo |
| [10. 关键技术点速记](#10-关键技术点速记) | 10 个核心概念的速查表 |
| [11. 学习路径建议](#11-学习路径建议) | 推荐 4 步学习顺序 |
| [12. 常见问题](#12-常见问题) | 5 类常见问题及解决方法 |

---

## 1. 项目目录结构（详细版）

```text
MCP_DEMO/
│
├── FC/                      # ① 第一阶段：Function Calling（最原始的"调工具"方式）
│   ├── __init__.py          #    模块说明文件（空文件，让 Python 把 FC 识别成一个包）
│   ├── fc_demo.py           #    入门：手写一遍"问天气"的完整流程（工具返回假数据）
│   ├── fc_demo2.py          #    进阶：把假数据换成真实的智谱联网搜索
│   └── fc_demo3.py          #    跳阶：不写工具，直接用智谱平台自带联网搜索
│
├── agent_demo/              # ② 第二阶段：LangChain 智能体
│   ├── __init__.py          #    模块说明文件
│   └── zhipu_demo.py        #    用 LangChain 一行代码搞定 Agent + 多轮对话记忆
│
├── langgraph_mcp/           # ③ 第三阶段：LangGraph 状态图 + MCP
│   ├── __init__.py          #    模块说明文件
│   ├── agent_mcp.py         #    "自动挡"：用 LangGraph 内置 ReAct Agent 调 MCP 工具
│   └── graph_mcp.py         #    "手动挡"：自己画状态图（节点1读资源，节点2调工具）
│
├── mcp_demo/                # ④ 第四阶段：MCP 服务端 + 各种客户端
│   ├── __init__.py          #    模块说明文件
│   ├── main.py              #    服务端启动入口（python -m mcp_demo.main 就跑起来）
│   ├── mcp_server.py        #    服务端定义：用 FastMCP 注册工具（搜索）和资源（邮箱/分类）
│   ├── mcp_tools.py         #    额外工具：注册 add（加法）和 multiply（乘法）
│   ├── agent_client.py      #    客户端 A：LangChain 风格的 MCP 客户端
│   └── fastmcp_client.py    #    客户端 B：原生 fastmcp 客户端（不依赖 LangChain）
│
├── env_utils.py             # ⑤ 通用：读取 .env 文件里的 API Key（OpenAI / DeepSeek / 智谱）
├── zhipu_ai.py              # ⑤ 通用：初始化智谱 AI 客户端（其他文件统一从这里导入）
├── requirements.txt         # ⑤ 通用：依赖清单（pip install -r requirements.txt）
└── docs/
    └── readme.md            # 你正在看的这份文档
```

### 一句话总结 4 个阶段

| 目录 | 解决的问题 | 一句话类比 |
|------|----------|----------|
| **FC/** | 工具怎么"手动调用一次" | 学开车 —— 手动挡，一步一步自己操作 |
| **agent_demo/** | 让模型"自己决定"调什么工具 | 学开车 —— 自动挡，告诉目的地就行 |
| **langgraph_mcp/** | 把多个步骤串成"流程图" | 学开车 —— 提前规划路线（先去加油站→再上高速） |
| **mcp_demo/** | 把工具"开放给别人用" | 学开车 —— 开出租车，让别人也能上你的车 |

---

## 2. 环境与依赖

### 2.1 Python 环境

- 推荐 **Python 3.11+**（项目里能看到 `cpython-311` 缓存文件，说明是 3.11 写的）。

### 2.2 安装依赖

```bash
pip install -r requirements.txt
```

核心依赖对照表：

| 包名 | 它是干嘛的 | 主要用在哪些文件 |
|------|----------|----------------|
| `openai` | 调 OpenAI（兼容中转）的大模型 | `FC/fc_demo.py`、`FC/fc_demo2.py` |
| `zhipuai` | 智谱 AI 原生 SDK，用来调"联网搜索"等特殊接口 | `FC/fc_demo3.py`、`mcp_demo/mcp_server.py`、`zhipu_ai.py` |
| `langchain` / `langchain-core` / `langchain-openai` | LangChain 框架本体 | `agent_demo/zhipu_demo.py`、`mcp_demo/agent_client.py` |
| `langchain-community` | 提供 `ChatMessageHistory` 等社区组件 | `agent_demo/zhipu_demo.py` |
| `langgraph` | 状态图引擎（"流程图"） | `langgraph_mcp/*.py` |
| `mcp` | MCP 协议官方实现 | `mcp_demo/mcp_server.py` |
| `fastmcp` | 更易用的 MCP 服务端 / 客户端 | `mcp_demo/fastmcp_client.py` |
| `langchain-mcp-adapters` | 把 MCP 工具"翻译"成 LangChain 能用的工具 | `mcp_demo/agent_client.py`、`langgraph_mcp/*.py` |

### 2.3 环境变量（API Key 配置）

在项目**根目录**新建一个 `.env` 文件，写入下面的内容（把 xxxx 换成你自己的 Key）：

```env
OPENAI_API_KEY=sk-xxxxxx
DEEPSEEK_API_KEY=sk-xxxxxx
ZHIPU_API_KEY=xxxxxxxxxxxxxxxx
```

`env_utils.py` 会自动加载这个文件，并把 3 个 Key 暴露成 3 个变量。其他文件统一 `from env_utils import ZHIPU_API_KEY` 就行。

> ⚠️ **小提示**：`load_dotenv(override=True)` 会让 .env 里的值覆盖系统环境变量，这样不同电脑配置不同 Key 时很方便。

---

## 3. 全局公共模块（项目最底层的"基础设施"）

这两个文件被其他所有模块依赖，必须放在项目根目录。

### 3.1 [env_utils.py](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/env_utils.py) —— "钥匙盒"

- **作用**：从 `.env` 文件里把 3 个 API Key 读出来，存成 3 个变量。
- **包含内容**：
  - `import os` 和 `from dotenv import load_dotenv`
  - `load_dotenv(override=True)` —— 加载 .env
  - `OPENAI_API_KEY` / `DEEPSEEK_API_KEY` / `ZHIPU_API_KEY` 三个变量
- **通俗理解**：相当于酒店前台保管所有房卡，你办入住时统一从前台拿。
- **谁会用到**：所有需要 API Key 的文件（几乎所有 demo 都用）。

### 3.2 [zhipu_ai.py](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/zhipu_ai.py) —— "工厂车间"

- **作用**：项目启动时就把两个客户端创建好，其他文件直接 import 即可，不用每次都新建。
- **包含内容**：
  - `zhipuai_client = ZhipuAI(api_key=ZHIPU_API_KEY)` —— 智谱官方 SDK 客户端，用来调"联网搜索"等特殊接口
  - `llm = ChatOpenAI(temperature=0, model='glm-4-air-250414', ...)` —— LangChain 风格的大模型对象
- **关键参数**：
  - `temperature=0`：模型不随机，回答稳定
  - `model='glm-4-air-250414'`：智谱的 GLM-4-Air 模型
  - `base_url`：智谱兼容 OpenAI 协议的接口地址
- **谁会用到**：`agent_demo/`、`langgraph_mcp/`、`mcp_demo/agent_client.py`。

---

## 4. ① FC 模块：手把手教你"调一次工具"

> **目标**：理解"工具描述 → 模型返回 `tool_calls` → 开发者执行工具 → 二次对话"的标准流程。

### 4.1 [fc_demo.py](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/FC/fc_demo.py) —— OpenAI 风格 + 假天气

**它里面有什么**：

1. **`get_weather(location)` 函数**：模拟一个天气查询工具，返回写死的假数据（22°C、晴天、3级风）。真实项目应该接和风天气 API。
2. **`tools` 列表**：用 JSON Schema 描述工具，声明工具名、描述、参数（`location` 是必填的字符串）。
3. **`OpenAI` 客户端**：通过中转服务 `https://xiaoai.plus/v1` 创建。
4. **第一轮调用**：把"今天北京的天气怎么样？"连同工具描述发给模型，看模型怎么决定。
5. **解析 `tool_calls`**：取出模型想调的工具名和参数，手动调用 `get_weather("北京")`。
6. **第二轮调用**：把"用户问题 + 工具调用记录 + 工具结果"再发给模型，得到自然语言回答。

**核心 4 步流程**（看懂这张图就懂 FC）：

```
┌─────────────┐    1. 问问题     ┌─────────┐
│   用户      │ ──────────────→ │  GPT    │
└─────────────┘                 │  模型   │
                                └────┬────┘
                                     │ 2. 决定调工具 get_weather("北京")
                                     ▼
                                ┌─────────┐
                                │  本地   │  3. 执行工具
                                │  函数   │
                                └────┬────┘
                                     │ 4. 把工具结果再喂给模型
                                     ▼
                                ┌─────────┐
                                │  模型   │  5. 生成自然语言回答
                                └─────────┘
```

### 4.2 [fc_demo2.py](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/FC/fc_demo2.py) —— 工具内部接真搜索

**它里面有什么**：

- 与 `fc_demo.py` 的代码结构几乎完全一样。
- **唯一区别**：`get_weather()` 内部不再返回假数据，而是真的去调 `zhipuai_client.web_search.web_search(search_engine="search-std", search_query="北京，今天的天气情况")`，把搜索结果拼成字符串返回。

**想表达的概念**：工具函数内部可以是任何外部 API —— 天气、数据库、地图、翻译……只要你能用代码实现，就能让大模型调用。

### 4.3 [fc_demo3.py](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/FC/fc_demo3.py) —— 智谱自带联网搜索

**它里面有什么**：

- 不写 `tools` 列表（不自己定义工具）。
- 直接在 `tools` 参数里启用智谱**内置**的 `web_search` 类型工具。
- 关键配置：
  - `search_engine="search_pro_sogou"` —— 用搜狗专业版搜索引擎
  - `search_prompt="..."` —— 让模型扮演"财经分析师"，整理搜索结果
- 一次性得到整理好的答案，不用二次调用。

**想表达的概念**：如果不想自己造工具，可以让大模型直接用平台自带的搜索能力（省事）。

---

## 5. ② Agent 模块：让模型"自己决定"调什么

> **目标**：用 LangChain 框架简化 FC 那一堆样板代码，并加入"多轮对话记忆"。

### 5.1 [zhipu_demo.py](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/agent_demo/zhipu_demo.py) —— LangChain Agent + 多轮记忆

**它里面有什么**：

1. **`SearchInput(BaseModel)`**：用 pydantic 定义工具参数的 schema（一个 `query` 字符串）。
2. **`@tool('my_search_tool')`** 装饰的 `my_search` 函数：和 FC 的工具函数类似，但通过装饰器注册成 LangChain 工具。
3. **`ChatPromptTemplate`**：提示词模板，包含 system、chat_history 占位、用户输入、agent_scratchpad（Agent 思考过程）。
4. **`create_tool_calling_agent(llm, tools, prompt)`**：把"大模型 + 工具 + 提示词"打包成一个 Agent。
5. **`AgentExecutor`**：Agent 的"执行器"，负责反复调大模型、调工具、直到任务完成。
6. **`store` 字典 + `get_session_history` 函数**：按 `session_id` 存不同的对话历史。
7. **`RunnableWithMessageHistory`**：把 Executor 包装一层，自动注入历史记录。
8. **两次 `agent_with_history.invoke(...)`**：演示"先自我介绍 → 再问出生那年发生的大事"，Agent 能记住 "1983年"。

**通俗理解**：FC 是你"手把手"指挥模型；这里是给模型一个"工具箱 + 一本对话笔记"，让它自己看着办。

---

## 6. ③ LangGraph + MCP 模块：流程图 + 远程工具

> **目标**：用 LangGraph 把多个步骤串成流程图，并接入 MCP 服务器（远程的工具箱）。

### 6.1 [agent_mcp.py](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/langgraph_mcp/agent_mcp.py) —— 自动挡版

**它里面有什么**：

- **`mcp_server_config`**：MCP 服务器连接信息（`http://localhost:8008/sse`，用 SSE 协议）。
- **`@asynccontextmanager make_agent()`**：异步上下文管理器，进入时连接 MCP 服务器、构造 Agent，退出时自动断开。
- **`MultiServerMCPClient({'lx_mcp': config})`**：langchain-mcp-adapters 提供的客户端，可以同时连多个 MCP 服务。
- **`create_react_agent(llm, tools=client.get_tools())`**：LangGraph 内置的 ReAct（推理+行动）Agent。
- **`main()`**：示范一次调用 `'计算一下(3+6)的结果'`，会触发 `mcp_tools.py` 里的 `add` 工具。

**关键代码（一看就懂）**：

```python
async with MultiServerMCPClient({'lx_mcp': mcp_server_config}) as client:
    agent = create_react_agent(llm, tools=client.get_tools())  # 工具 = MCP 服务器上所有的工具
    resp = await agent.ainvoke({'messages': '计算一下(3+6)的结果'})
```

### 6.2 [graph_mcp.py](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/langgraph_mcp/graph_mcp.py) —— 手动挡版

**它里面有什么**：

- **`MyState(TypedDict)`**：自定义状态结构（`email` + `messages` 列表，用 `add_messages` 表示追加消息而不是覆盖）。
- **`async_node(state)`**：节点 1，连接 MCP，调用工具生成回答。
- **`async_resource(state)`**：节点 0，连接 MCP，**读取资源**（用户邮箱 `datas://users/567/email`）。
- **`StateGraph(MyState)`**：建一张空的状态图。
- **`add_node` / `set_entry_point` / `add_edge` / `END`**：注册节点、设置入口、连边。
- **`graph.compile()`**：编译成可执行的工作流。
- **`execute_graph()`**：交互式 REPL（输入 `q` 退出）。
- **`_print_event()`**：流式输出事件，做 HTML 美化和超长截断。

**工作流图**：

```
START → resource(读 MCP 资源,得到 email) → agent(调 MCP 工具,生成回答) → END
```

**通俗理解**：`agent_mcp.py` 是开自动挡直接到目的地；这里是手动挡，先去加油站（读资源）、再上高速（调工具）。

---

## 7. ④ MCP 模块：服务端 + 多种客户端

> **目标**：把工具/资源用标准协议开放出来，让任何兼容 MCP 的客户端（Claude、Cursor、LangChain……）都能调用。

### 7.1 [main.py](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/mcp_demo/main.py) —— 服务端启动入口

**它里面有什么**（只有 5 行）：

```python
from mcp_demo.mcp_server import mcp_server   # 拿到 FastMCP 实例
import mcp_demo.mcp_tools                    # 关键！这一行只是为了"触发" add/multiply 注册

if __name__ == '__main__':
    mcp_server.run(transport='sse')          # 用 SSE 协议启动，监听 :8008
```

**为什么要 `import mcp_demo.mcp_tools`？**

因为 `mcp_tools.py` 里的 `@mcp_server.tool()` 装饰器**只有在该文件被 import 时才会执行**。不 import，add/multiply 就不会被注册到 mcp_server 上。

### 7.2 [mcp_server.py](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/mcp_demo/mcp_server.py) —— 服务端定义（核心）

**它里面有什么**：

- **`mcp_server = FastMCP(name='lx-mcp', instructions='我自己的MCP服务', port=8008)`**：创建一个 FastMCP 实例。
  - `name`：服务端的名字（客户端会看到）
  - `instructions`：给客户端看的说明
  - `port=8008`：监听端口（必须和客户端配置一致）

- **`@mcp_server.tool('my_search_tool')`** 装饰的 `my_search(query)` 函数：
  - **工具**：调智谱联网搜索
  - 客户端可以传入 query（搜索关键词），得到搜索结果

- **`@mcp_server.resource("datas://users/{user_id}/email")`** 装饰的 `get_user_email(user_id)` 函数：
  - **资源**：根据用户 ID 返回邮箱
  - `{user_id}` 是路径参数，客户端请求 `datas://users/123/email` 就会拿到 `alice@example.com`
  - 假数据：123→alice，456→bob，其他→not_found

- **`@mcp_server.resource("data://product-categories")`** 装饰的 `get_categories()` 函数：
  - **资源**：返回商品分类列表（Electronics / Books / Home Goods）
  - 静态数据，没有路径参数

**三类 MCP 原语对照**：

| 原语 | 装饰器 | 用途 | 本项目例子 |
|------|--------|------|----------|
| **Tool** | `@mcp_server.tool()` | 能"主动调用"的函数 | `my_search`、`add`、`multiply` |
| **Resource** | `@mcp_server.resource()` | 只能"按 URI 读取"的数据 | `datas://users/...`、`data://...` |
| **Prompt** | `@mcp_server.prompt()` | 预设的提示词模板（本项目未演示） | - |

### 7.3 [mcp_tools.py](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/mcp_demo/mcp_tools.py) —— 额外工具（数学）

**它里面有什么**（只有 3 个东西）：

- `from mcp_demo.mcp_server import mcp_server` —— 拿到服务端实例
- `@mcp_server.tool()` 装饰的 `add(a, b)`：加法
- `@mcp_server.tool()` 装饰的 `multiply(a, b)`：乘法

**想表达的概念**：工具多了以后，按功能拆到不同文件（比如搜索一个文件、数学一个文件），维护起来更清晰。所有被装饰的函数都会被登记到同一个 `mcp_server` 上。

### 7.4 [agent_client.py](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/mcp_demo/agent_client.py) —— LangChain 版客户端

**它里面有什么**：

- **`mcp_server_config`**：连接信息（同上）
- **`prompt`**：和 `agent_demo` 里的几乎一样（system + chat_history + human + agent_scratchpad）
- **`client_call()` 异步函数**，做了 3 件事：
  1. `client.get_tools()` —— 列出 MCP 服务器提供的所有工具
  2. `client.get_resources('lx_mcp', uris='datas://users/567/email')` —— 读一个资源
  3. `create_tool_calling_agent(llm, tools, prompt)` + `AgentExecutor` —— 让 Agent 自动用工具
- **示例问题**：`'请计算:10和89的乘积'` 会触发 `multiply` 工具

**与 `fastmcp_client.py` 的区别**：本文件让大模型自动决定调什么工具；`fastmcp_client.py` 是手写代码直接调。

### 7.5 [fastmcp_client.py](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/mcp_demo/fastmcp_client.py) —— 原生 fastmcp 客户端

**它里面有什么**：

- `from fastmcp import Client` + `from fastmcp.client import SSETransport`
- `async with Client(SSETransport(url='http://localhost:8008/sse')) as client:` 连接
- 三个核心 RPC：
  - `client.list_tools()` —— 列出所有工具
  - `client.read_resource('datas://users/567/email')` —— 读一个资源
  - `client.call_tool(name='add', arguments={"a": 23, "b": 11})` —— 调一个工具

**适用场景**：
- 不需要大模型参与（比如测试 MCP 服务）
- 集成测试
- 其他语言写的客户端（只要也支持 MCP 协议就能连）

---

## 8. 整体调用关系图

```text
                    ┌──────────────────────────────┐
                    │        用户 / 应用层         │
                    └──────────────┬───────────────┘
                                   │
            ┌──────────────────────┼──────────────────────┐
            ▼                      ▼                      ▼
   fastmcp_client.py        agent_client.py        langgraph_mcp/*
   (原生 fastmcp 客户端)    (LangChain 客户端)     (LangGraph 客户端)
            │                      │                      │
            └─────────── MultiServerMCPClient (SSE, :8008) ─┘
                                   │
                                   ▼
                       ┌────────────────────┐
                       │   mcp_server       │  FastMCP('lx-mcp')
                       │   (mcp_server.py)  │
                       └─────────┬──────────┘
                                 │ 装饰器注册
              ┌──────────────────┼─────────────────────────┐
              ▼                  ▼                          ▼
         my_search           add / multiply            resources(email,
       (mcp_server.py)       (mcp_tools.py)           categories ...)
```

---

## 9. 启动与运行示例

### 9.1 启动 MCP 服务端（必须先做）

```bash
# 在项目根目录
python -m mcp_demo.main
```

启动成功后，服务器监听 `http://localhost:8008/sse`。可以另开终端用 `curl -N http://localhost:8008/sse` 简单验证。

### 9.2 运行各客户端（另开终端）

```bash
# 原生 fastmcp 客户端（不依赖大模型）
python -m mcp_demo.fastmcp_client

# LangChain + MCP 客户端（会自动调用 multiply 工具）
python -m mcp_demo.agent_client

# LangGraph ReAct Agent + MCP
python -m langgraph_mcp.agent_mcp

# LangGraph 状态图 + MCP（命令行 REPL，输入 q 退出）
python -m langgraph_mcp.graph_mcp
```

### 9.3 单独体验 FC / Agent（不需要启动服务端）

```bash
python -m FC.fc_demo        # OpenAI 风格 FC（假天气）
python -m FC.fc_demo2       # OpenAI 风格 + 智谱真实搜索
python -m FC.fc_demo3       # 智谱原生 web_search 工具
python -m agent_demo.zhipu_demo   # LangChain Agent + 多轮记忆
```

> 💡 **Windows 建议用 `python -m 包.模块`** 的形式，而不是 `cd FC && python fc_demo.py`，避免相对导入问题。

---

## 10. 关键技术点速记

| 概念 | 出现位置 | 备注 |
|------|---------|------|
| **JSON Schema 描述工具** | `FC/fc_demo.py`、`FC/fc_demo2.py` | `tools=[{"type":"function","function":{...}}]` |
| **原生 SDK Web Search** | `FC/fc_demo3.py`、`mcp_demo/mcp_server.py` | `zhipuai_client.web_search.web_search(...)` |
| **LangChain `@tool`** | `agent_demo/zhipu_demo.py` | `args_schema=SearchInput` 提供结构化参数 |
| **`create_tool_calling_agent`** | `agent_demo/zhipu_demo.py`、`mcp_demo/agent_client.py` | LangChain Agent 工厂 |
| **`RunnableWithMessageHistory`** | `agent_demo/zhipu_demo.py` | 用 `session_id` 维护多轮对话 |
| **LangGraph ReAct** | `langgraph_mcp/agent_mcp.py`、`graph_mcp.py` | `create_react_agent` / `StateGraph` |
| **`MultiServerMCPClient`** | `mcp_demo/agent_client.py`、`langgraph_mcp/*` | 同时连接多个 MCP 服务 |
| **FastMCP 装饰器** | `mcp_demo/mcp_server.py`、`mcp_tools.py` | `@mcp_server.tool()` / `@mcp_server.resource()` |
| **SSE 协议** | 所有客户端 → `:8008/sse` | 也可改成 `stdio`，但 SSE 更利于跨进程 |
| **流式事件打印** | `langgraph_mcp/graph_mcp.py` | `astream(stream_mode="values")` + `_printed` 去重 |

---

## 11. 学习路径建议

按下面这个顺序看代码，会越来越顺：

1. **先跑 FC**：
   - 把 `fc_demo.py / fc_demo2.py / fc_demo3.py` 三个文件分别执行
   - 观察 `tool_calls`、二次对话与内置工具的差异
   - **理解目标**：工具调用到底是怎么"来回跑"的

2. **再跑 Agent**：
   - 看 `zhipu_demo.py`
   - 理解 `@tool`、`create_tool_calling_agent`、`RunnableWithMessageHistory` 的角色
   - 观察多轮记忆效果（同一个 `session_id`）

3. **然后 LangGraph**：
   - 先看 `agent_mcp.py` 的极简 ReAct（自动挡）
   - 再看 `graph_mcp.py` 的状态图版本（手动挡）
   - **理解目标**：节点 / 边 / 状态 / 资源

4. **最后 MCP**：
   - 先启动 `mcp_demo.main` 作为服务端
   - 用 `fastmcp_client.py` 直接验证工具/资源（不依赖大模型）
   - 用 `agent_client.py` / `agent_mcp.py` 把 MCP 工具接入上层 Agent
   - 在 `mcp_tools.py` 里**继续添加工具**，重启服务后观察客户端能否自动发现

---

## 12. 常见问题

| 问题 | 原因 / 解决 |
|------|------------|
| **客户端连不上服务** | 检查 `mcp_demo/main.py` 是否在跑、`:8008` 端口是否被占用；防火墙 / 代理是否拦截 SSE 长连接 |
| **工具没出现** | `main.py` 里必须 `import mcp_demo.mcp_tools`，否则 `add` / `multiply` 不会被注册 |
| **`OPENAI_API_KEY` 用的是中转** | `fc_demo.py` / `fc_demo2.py` 的 `base_url="https://xiaoai.plus/v1"` 是中转服务，请自行替换或确保本地代理可用 |
| **依赖装不上** | `mcp==1.6.0` / `fastmcp==2.2.5` 对 Python 版本敏感，遇到编译错误请升级到 Python 3.11+ |
| **想加新工具** | 在 `mcp_demo/mcp_tools.py` 里加一个新函数，用 `@mcp_server.tool()` 装饰，重启服务即可 |

---

> **文档维护提示**：随着课程迭代，`mcp_demo` 模块很可能继续增加工具/资源。如需更新本文档，只需同步刷新第 7 节与第 8 节的对应部分即可。
