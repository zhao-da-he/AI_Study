# RAG_PROJECT 项目详解（readme2.md）

> 本文档是对 [readme.md](./readme.md) 的**完整重写版**，对整个 RAG_PROJECT 工程进行了更细致、更深入、逐文件级别的梳理。
> 项目主题：**半导体 / 芯片制造领域的企业级 RAG（检索增强生成）系统**。
> 技术栈核心：**LangChain + LangGraph + Milvus + Gradio + MCP**。

---

## 📚 目录

- [一、项目是什么](#一项目是什么)
- [二、技术栈全景](#二技术栈全景)
- [三、目录结构（树状图）](#三目录结构树状图)
- [四、运行入口与启动方式汇总](#四运行入口与启动方式汇总)
- [五、根目录文件逐个解读](#五根目录文件逐个解读)
- [六、子项目逐文件解读](#六子项目逐文件解读)
  - [6.1 utils —— 基础设施](#61-utils--基础设施)
  - [6.2 llm_models —— 模型与工具封装](#62-llm_models--模型与工具封装)
  - [6.3 documents —— 文档解析与 Milvus 写入](#63-documents--文档解析与-milvus-写入)
  - [6.4 tools —— Agent 检索工具](#64-tools--agent-检索工具)
  - [6.5 agent —— 基础 Tool Calling Agent](#65-agent--基础-tool-calling-agent)
  - [6.6 graph —— LangGraph 自评估 RAG（V1）](#66-graph--langgraph-自评估-ragv1)
  - [6.7 graph2 —— LangGraph 高级 RAG（V2：混合检索 + 路由 + 自评估）](#67-graph2--langgraph-高级-ragv2混合检索--路由--自评估)
  - [6.8 search_tool —— Tavily 搜索测试](#68-search_tool--tavily-搜索测试)
  - [6.9 test_load —— 文档加载器实验](#69-test_load--文档加载器实验)
  - [6.10 test_milvus —— Milvus 原生 API 实验](#610-test_milvus--milvus-原生-api-实验)
  - [6.11 test_vector —— 向量检索实验](#611-test_vector--向量检索实验)
  - [6.12 test_mcp —— MCP 工具服务实验](#612-test_mcp--mcp-工具服务实验)
- [七、核心数据流（End-to-End）](#七核心数据流end-to-end)
- [八、依赖与环境变量](#八依赖与环境变量)
- [九、数据集说明](#九数据集说明)
- [十、可视化产物](#十可视化产物)
- [十一、扩展与工程化建议](#十一扩展与工程化建议)

---

## 一、项目是什么

`RAG_PROJECT` 是一个面向半导体行业的 **企业级 RAG 综合实战工程**，演示了一条完整的工业级链路：

```
非结构化文档(MD/PDF)
   ↓ 解析
Document 元素列表（UnstructuredMarkdownLoader）
   ↓ 合并 / 语义切分
chunk 文档（SemanticChunker + 父子合并）
   ↓ Embedding（BGE / OpenAI）
   ↓ 写入 Milvus（稠密 HNSW + 稀疏 BM25 + RRF）
向量数据库 t_collection01
   ↓
Query → Query Router
   ├─→ 命中主题 → Milvus 混合检索
   │       ↓
   │     Documents Relevance Grader
   │       ↓ (yes)
   │     Generate Answer
   │       ↓
   │     Hallucination & Answer Grader（Self-RAG）
   └─→ 未命中主题 → Tavily Web Search → Generate
```

**典型应用问题**：

- "什么是 EUV 光刻机？"
- "现在最先进的纳米级清洗技术是什么？"
- "介绍一下光刻机有哪几种？"

---

## 二、技术栈全景

| 层级 | 组件 | 用途 |
| --- | --- | --- |
| LLM | `ChatOpenAI(gpt-4o-mini)` | 主推理模型 |
| LLM 备选 | `ChatOpenAI(deepseek-chat)` | 已注释，可切换 DeepSeek |
| Embedding | `HuggingFaceEmbeddings(BAAI/bge-small-zh-v1.5)` | 离线中文向量 |
| Embedding | `OpenAIEmbeddings` | 备用语义切分 |
| 向量库 | `pymilvus + langchain_milvus` | 稠密+稀疏混合索引 |
| 文档解析 | `unstructured / UnstructuredMarkdownLoader` | 元素级解析 |
| 文本切分 | `langchain_experimental.SemanticChunker` | 语义切分 |
| Agent | `create_tool_calling_agent` / `AgentExecutor` | 工具调用 |
| 工作流 | `langgraph.StateGraph` | DAG 编排 |
| 评估 | `llm.with_structured_output(Pydantic)` | 二元评分 |
| Web Search | `TavilySearchResults` | 兜底搜索 |
| MCP | `mcp.server.fastmcp + langchain_mcp_adapters` | 远程工具服务 |
| UI | `gradio` | 聊天界面 |
| 日志 | `loguru` | 彩色日志 |
| 配置 | `python-dotenv` | 环境变量 |

完整依赖见 [requirements.txt](../requirements.txt)。

---

## 三、目录结构（树状图）

```
RAG_PROJECT/
├── agent/
│   ├── __init__.py
│   └── rag_agent.py               # 基础 Tool Calling Agent
│
├── documents/
│   ├── __init__.py
│   ├── markdown_parser.py         # MD 解析 + 父子合并 + 语义切分
│   ├── milvus_db.py               # Milvus 建表 / 写入 / 检索
│   └── write_milvus.py            # 多进程批量入库
│
├── graph/
│   ├── __init__.py
│   ├── agent_node.py              # LLM 决定是否调用工具
│   ├── generate_node.py           # 基于检索结果生成答案
│   ├── get_human_message.py       # 取出最后一个 HumanMessage
│   ├── graph1.py                  # V1 主入口（StateGraph 编排 + REPL）
│   ├── graph_state1.py            # V1 状态定义 + Grade 模型
│   ├── graph_rag1-2.png           # V1 流程图
│   └── rewrite_node.py            # 文档不相关时改写问题
│
├── graph2/
│   ├── __init__.py
│   ├── generate_node2.py          # V2 生成节点（带格式化的 docs）
│   ├── grade_answer_chain.py      # 答案质量评分（yes/no）
│   ├── grade_documents_node.py    # 文档过滤节点
│   ├── grade_hallucinations_chain.py  # 幻觉检测评分
│   ├── grader_chain.py            # 文档相关性评分（yes/no）
│   ├── graph_2.py                 # V2 主入口（StateGraph 编排 + REPL）
│   ├── graph_gradio.py            # V2 Gradio 聊天界面
│   ├── graph_rag2.png             # V2 流程图
│   ├── graph_state2.py            # V2 状态定义
│   ├── query_route_chain.py       # 路由器：vectorstore vs web_search
│   ├── retriever_node.py          # 调用 Milvus 检索
│   ├── transform_query_node.py    # 问题改写（带 transform_count）
│   └── web_search_node.py         # 调用 Tavily
│
├── llm_models/
│   ├── __init__.py
│   ├── all_llm.py                 # ChatOpenAI + Tavily 封装
│   └── embeddings_model.py        # OpenAI / BGE Embedding
│
├── search_tool/
│   ├── __init__.py
│   └── test_search.py             # Tavily 简单调用测试
│
├── test_load/
│   ├── __init__.py
│   ├── demo1.py                   # PyPDFLoader（每页 1 Doc）
│   ├── demo2.py                   # UnstructuredLoader hi_res + 输出 JSON
│   ├── demo4.py                   # UnstructuredMarkdownLoader elements
│   └── dome3.py                   # 自定义 JSON → Document
│
├── test_mcp/
│   ├── __init__.py
│   ├── agent_client.py            # 命令行 MCP Agent 客户端
│   ├── mcp_app.py                 # MCP Agent + Gradio
│   ├── mcp_server.py              # FastMCP 服务端
│   └── zhipu_agent.py             # 本地版（不通过 MCP）
│
├── test_milvus/
│   ├── __init__.py
│   └── demo1.py                   # MilvusClient 原生增删查
│
├── test_vector/
│   ├── __init__.py
│   ├── demo1.py                   # BGE Embedding 调用示例
│   └── demo2.py                   # Milvus BM25 Function 全文搜索
│
├── tools/
│   ├── __init__.py
│   └── retriever_tools.py         # Milvus 检索 → LangChain Tool
│
├── utils/
│   ├── __init__.py
│   ├── env_utils.py               # .env 加载 + 常量
│   ├── log_utils.py               # loguru 封装
│   └── print_utils.py             # 事件流打印辅助
│
├── datas/                         # 数据集（MD/PDF/JSON）
│
├── docs/
│   ├── 9.11--RAG企业知识库项目.pdf
│   ├── 9.12--RAG企业知识库项目.pdf
│   ├── 9.14--RAG企业知识库项目.pdf
│   ├── 9.15--RAG企业知识库项目.pdf
│   ├── readme.md                  # 上一版文档
│   └── readme2.md                 # 本文档
│
├── draw_png.py                    # 把 LangGraph 导出为 PNG
├── graph_rag1.png                 # 根目录的 V1 流程图
├── graph_rag2.png                 # 根目录的 V2 流程图
├── main.py                        # PyCharm 默认模板
├── requirements.txt               # 依赖锁定
└── t.py                           # 测试 LangSmith Hub
```

---

## 四、运行入口与启动方式汇总

| 子项目 | 命令 | 说明 |
| --- | --- | --- |
| 基础 Agent | `python agent/rag_agent.py` | 单轮/多轮 REPL（`session_id='zs123'`） |
| LangGraph V1 | `python graph/graph1.py` | 自评估 RAG 命令行 |
| LangGraph V2 | `python graph2/graph_2.py` | 混合检索+路由+自评估 |
| Gradio（V2） | `python graph2/graph_gradio.py` | 浏览器聊天界面 |
| Milvus 入库（小） | `python documents/milvus_db.py` | 单进程建表+插入 |
| Milvus 入库（大） | `python documents/write_milvus.py` | 多进程批量入库 |
| MCP 服务端 | `python test_mcp/mcp_server.py` | 默认 SSE 端口 8000 |
| MCP 客户端（命令行） | `python test_mcp/agent_client.py` | 调用 add/multiply/search |
| MCP 客户端（Gradio） | `python test_mcp/mcp_app.py` | 浏览器聊天界面 |
| LangSmith Hub | `python t.py` | 拉取 `rlm/rag-prompt` |

---

## 五、根目录文件逐个解读

### 5.1 [main.py](../main.py)

PyCharm 创建的默认模板，仅打印 `Hi, PyCharm`。**不是项目入口**。

### 5.2 [requirements.txt](../requirements.txt)

依赖锁定文件（143 个包），关键依赖：
- `langchain==0.3.23` / `langgraph==0.3.30` / `langchain-milvus==0.1.9`
- `pymilvus==2.5.6`
- `unstructured==0.17.2` / `langchain-unstructured==0.1.6`
- `gradio`（间接由 `langchain` 生态拉入）
- `zhipuai`（MCP 服务端智谱搜索用）
- `loguru==0.7.3` / `python-dotenv==1.1.0`

### 5.3 [draw_png.py](../draw_png.py)

```python
def draw_graph(graph, file_name: str):
    try:
        mermaid_code = graph.get_graph().draw_mermaid_png()
        with open(file_name, "wb") as f:
            f.write(mermaid_code)
    except Exception as e:
        log.exception(e)
```

被 [graph/graph1.py](../graph/graph1.py) 和 [graph2/graph_2.py](../graph2/graph_2.py) 调用，把 LangGraph 渲染成 Mermaid PNG。

### 5.4 [t.py](../t.py)

极简测试：从 LangSmith Hub 拉取 `rlm/rag-prompt`，验证 Hub 连通性。

### 5.5 根目录 PNG

- [graph_rag1.png](../graph_rag1.png) —— V1 流程图（备份）
- [graph_rag2.png](../graph_rag2.png) —— V2 流程图（备份）
- [graph/graph_rag1-2.png](../graph/graph_rag1-2.png) —— V1 流程图（最新导出）
- [graph2/graph_rag2.png](../graph2/graph_rag2.png) —— V2 流程图（最新导出）

---

## 六、子项目逐文件解读

### 6.1 utils —— 基础设施

#### [utils/env_utils.py](../utils/env_utils.py)

| 行号 | 内容 | 说明 |
| --- | --- | --- |
| L1-L4 | `import os` + `from dotenv import load_dotenv` | 加载 `.env` |
| L5 | `load_dotenv(override=True)` | 覆盖已有环境变量 |
| L7-L9 | 读取 `OPENAI_API_KEY / DEEPSEEK_API_KEY / ZHIPU_API_KEY` | 三个平台 Key |
| L11 | `MILVUS_URI = 'http://1.95.116.112:19530'` | 远程 Milvus 地址 |
| L13 | `COLLECTION_NAME = 't_collection01'` | 全局唯一 Collection 名 |

> ⚠️ **建议改为本地 Milvus Lite**：安装 `pymilvus[milvus_lite]`，把 URI 改为 `./milvus.db`。

#### [utils/log_utils.py](../utils/log_utils.py)

`MyLogger` 单例：
- 默认输出到 stdout（带颜色、模块名、行号）。
- 文件输出已注释，需要时取消注释 `self.logger.add(log_file_path, ...)` 即可。
- 暴露全局 `log = MyLogger().get_logger()`。

#### [utils/print_utils.py](../utils/print_utils.py)

`_print_event(event, _printed, max_length=1500)`：
- 从 LangGraph 的 `stream_mode='values'` 事件中取出最后一条消息；
- 超过 1500 字符自动截断并追加 `" ... （已截断）"`；
- 用 `_printed` 集合去重，避免重复打印。

### 6.2 llm_models —— 模型与工具封装

#### [llm_models/all_llm.py](../llm_models/all_llm.py)

```python
llm = ChatOpenAI(
    temperature=0,
    model='gpt-4o-mini',
    api_key=OPENAI_API_KEY,
    base_url="https://xiaoai.plus/v1"          # 第三方代理
)
web_search_tool = TavilySearchResults(max_results=2)
```

被 [graph2/web_search_node.py](../graph2/web_search_node.py) 和所有需要 LLM 的节点调用。

#### [llm_models/embeddings_model.py](../llm_models/embeddings_model.py)

- `openai_embedding` —— 用于 SemanticChunker（语义切分）。
- `bge_embedding` —— `BAAI/bge-small-zh-v1.5`，CPU 推理，512 维。

> `bge_embedding` 被 [documents/milvus_db.py](../documents/milvus_db.py) 的 `Milvus(...)` 和 [tools/retriever_tools.py](../tools/retriever_tools.py) 使用。

### 6.3 documents —— 文档解析与 Milvus 写入

#### [documents/markdown_parser.py](../documents/markdown_parser.py)

**类 `MarkdownParser`** —— 完整流水线：

```
Markdown 文件
   ↓ UnstructuredMarkdownLoader(mode='elements', strategy='fast')
docs（每个 element 一个 Document）
   ↓ merge_title_content：Title/NarrativeText 父子合并
merged_data（带 title 字段的 content Document）
   ↓ text_chunker：>5000 字符走 SemanticChunker
chunk_documents
```

**关键方法**：

| 方法 | 行号 | 作用 |
| --- | --- | --- |
| `__init__` | L14-L17 | 初始化 `SemanticChunker(openai_embedding, percentile)` |
| `text_chunker` | L19-L26 | 长度阈值 5000 的二次切分 |
| `parse_markdown_to_documents` | L29-L39 | 完整流水线（带日志） |
| `parse_markdown` | L41-L51 | Unstructured 懒加载 |
| `merge_title_content` | L53-L80 | 父子合并核心算法 |
| `__main__` | L83-L91 | 调试入口（注意：硬编码 `E:\my_project\RAG_PROJECT\datas\md\tech_report_0tfhhamx.md`） |

**`merge_title_content` 算法详解**：

1. 遍历所有 element document；
2. 若是 `NarrativeText` 且没有 `parent_id` → 直接作为独立正文加入结果；
3. 若是 `Title` → 记入 `parent_dict[element_id]`，`title` 元数据保存标题文字；如果该 Title 有 parent，则说明是"嵌套标题"，把当前标题拼接到父标题的 `page_content` 后；
4. 若是其他类别（ListItem / Table 等）且有 `parent_id` → 把内容追加到父 Title 的 `page_content`，并把父的 `category` 改为 `'content'`；
5. 最后把 `parent_dict` 中的所有 Title 父文档加入结果。

#### [documents/milvus_db.py](../documents/milvus_db.py)

**类 `MilvusVectorSave`** —— Milvus 写入与连接封装。

**Collection 字段定义**（[milvus_db.py L20-L33](../documents/milvus_db.py#L20-L33)）：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | INT64 (PK, auto) | 主键 |
| `text` | VARCHAR(6000) + jieba analyzer | 原文（BM25 输入） |
| `category` | VARCHAR(1000) | 'Title' / 'content' |
| `source` | VARCHAR(1000) | 来源 |
| `filename` | VARCHAR(1000) | 文件名 |
| `filetype` | VARCHAR(1000) | 文件类型 |
| `title` | VARCHAR(1000) | 所属标题 |
| `category_depth` | INT64 | 标题层级深度 |
| `sparse` | SPARSE_FLOAT_VECTOR | BM25 稀疏向量（Function 自动生成） |
| `dense` | FLOAT_VECTOR(512) | BGE 稠密向量 |

**索引**：
- sparse：`SPARSE_INVERTED_INDEX` + `BM25` + DAAT_MAXSCORE，k1=1.2, b=0.75；
- dense：`HNSW` + `IP` + M=16, efConstruction=64。

**`create_connection`**（[L78-L88](../documents/milvus_db.py#L78-L88)）：

```python
self.vector_store_saved = Milvus(
    embedding_function=bge_embedding,
    collection_name=COLLECTION_NAME,
    builtin_function=BM25BuiltInFunction(),
    vector_field=['dense', 'sparse'],
    consistency_level="Strong",
    auto_id=True,
    connection_args={"uri": MILVUS_URI}
)
```

**`__main__`**：完整演示 解析 → 建表 → 入库 → describe/list/query。

#### [documents/write_milvus.py](../documents/write_milvus.py)

**多进程 Pipeline**（[L12-L49](../documents/write_milvus.py#L12-L49)）：

| 进程 | 函数 | 职责 |
| --- | --- | --- |
| 解析进程 | `file_parser_process` | 扫描目录 → `MarkdownParser` → 每 20 条入队 |
| 写入进程 | `milvus_writer_process` | 从队列取批 → `MilvusVectorSave.add_documents` |

终止信号：队列收到 `None` 终止。

`__main__`：默认 `md_dir = 'E:\my_project\langchain_demo01\md'`，需根据实际环境修改。

### 6.4 tools —— Agent 检索工具

#### [tools/retriever_tools.py](../tools/retriever_tools.py)

```python
mv = MilvusVectorSave()
mv.create_connection()
retriever = mv.vector_store_saved.as_retriever(
    search_type='similarity',
    search_kwargs={
        "k": 3,
        "score_threshold": 0.1,
        "ranker_type": "rrf",
        "ranker_params": {"k": 100},
        'filter': {"category": "content"}     # 过滤掉 Title
    }
)
retriever_tool = create_retriever_tool(
    retriever,
    'rag_retriever',
    '搜索并返回关于 ‘半导体和芯片’ 的信息, 内容涵盖：半导体和芯片的封装、测试、光刻胶等'
)
```

> 关键点：`filter={"category": "content"}` 排除标题类文档，只保留正文；`ranker_type="rrf"` 启用 RRF 融合。

### 6.5 agent —— 基础 Tool Calling Agent

#### [agent/rag_agent.py](../agent/rag_agent.py)

**执行流程**：

1. `create_tool_calling_agent(llm, [retriever_tool], prompt)` 创建 Agent；
2. `AgentExecutor(agent=agent, tools=[retriever_tool])` 执行器；
3. `store: dict` 内存会话存储；
4. `get_session_history(session_id)` 返回/创建 `ChatMessageHistory`；
5. `RunnableWithMessageHistory` 包装执行器，启用多轮对话；
6. 调用：`agent_with_history.invoke({'input': '什么是EUV光刻机？'}, config={'configurable': {'session_id': 'zs123'}})`。

**核心特性**：
- LangChain `MessagesPlaceholder` 支持 `chat_history` 与 `agent_scratchpad`；
- 会话记忆基于内存字典（重启即失）。

### 6.6 graph —— LangGraph 自评估 RAG（V1）

#### 6.6.1 [graph/graph_state1.py](../graph/graph_state1.py)

- `AgentState`：TypedDict，`messages` 字段用 `add_messages` reducer；
- `Grade`：Pydantic 模型，`binary_score: str`（'yes' / 'no'）。

#### 6.6.2 [graph/agent_node.py](../graph/agent_node.py)

```python
def agent_node(state: AgentState):
    messages = state["messages"]
    model = llm.bind_tools([retriever_tool])
    response = model.invoke([messages[-1]])
    return {"messages": [response]}
```

把最近一条 HumanMessage 喂给绑定了工具的 LLM，决定是否调用工具。

#### 6.6.3 [graph/generate_node.py](../graph/generate_node.py)

```python
rag_chain = prompt | llm | StrOutputParser()
response = rag_chain.invoke({"context": docs, "question": question})
```

简单 prompt 模板："请根据以下检索到的上下文内容回答问题。如果不知道答案，请直接说明。回答保持简洁。"

#### 6.6.4 [graph/rewrite_node.py](../graph/rewrite_node.py)

```python
msg = [HumanMessage(content=f"""分析输入并尝试理解潜在的语义意图/含义。
    这是初始问题: {question}
    请提出一个改进后的问题: """)]
response = llm.invoke(msg)
return {"messages": [response]}
```

把"用户消息"改写为"AI 改写问题"，然后路由回 `agent`。

#### 6.6.5 [graph/get_human_message.py](../graph/get_human_message.py)

```python
def get_last_human_message(messages):
    for message in reversed(messages):
        if isinstance(message, HumanMessage):
            return message
    raise ValueError("No HumanMessage found in the messages list")
```

反向遍历取最后一个 HumanMessage。

#### 6.6.6 [graph/graph1.py](../graph/graph1.py)

**完整工作流编排**：

```
START → agent → (tools_condition) → retrieve → grade_documents → (yes) generate → END
                                  (END)      ↑                 → (no) rewrite → agent (循环)
```

- `tools_condition`（LangGraph 内置）：判断 LLM 是否请求工具调用；
- `MemorySaver()`：启用 checkpoint（按 `thread_id`）；
- `config = {"configurable": {"thread_id": uuid4()}}`：每个进程一个会话；
- `while True + input('用户：')`：REPL 循环；
- `_print_event` 流式打印每个节点状态。

### 6.7 graph2 —— LangGraph 高级 RAG（V2：混合检索 + 路由 + 自评估）

#### 6.7.1 [graph2/graph_state2.py](../graph2/graph_state2.py)

```python
class GraphState(TypedDict):
    question: str
    transform_count: int
    generation: str
    documents: List[Document]
```

注意：相比 V1，V2 **没有用 messages 列表**，而是显式字段，便于评估节点访问。

#### 6.7.2 [graph2/query_route_chain.py](../graph2/query_route_chain.py)

`question_router_chain` —— 路由器：

```python
class RouteQuery(BaseModel):
    datasource: Literal["vectorstore", "web_search"] = Field(...)

system = """你是一个擅长将用户问题路由到向量知识库或网络搜索的专家。
向量知识库包含与半导体材料，芯片制造，光刻技术相关的文档。
对于这些主题的问题请使用向量知识库，其他情况使用网络搜索。"""
```

被 [graph2/graph_2.py](../graph2/graph_2.py) 的 `route_question` 节点调用。

#### 6.7.3 [graph2/web_search_node.py](../graph2/web_search_node.py)

```python
def web_search(state):
    question = state["question"]
    docs = web_search_tool.invoke({"query": question})
    web_results = "\n".join([d["content"] for d in docs])
    web_results = Document(page_content=web_results)
    return {"documents": web_results, "question": question}
```

把 Tavily 结果合并为单 Document。

#### 6.7.4 [graph2/retriever_node.py](../graph2/retriever_node.py)

```python
def retrieve(state):
    documents = retriever.invoke(state["question"])
    return {"documents": documents, "question": question}
```

`retriever` 来自 [tools/retriever_tools.py](../tools/retriever_tools.py)，使用 Milvus RRF 混合检索。

#### 6.7.5 [graph2/grader_chain.py](../graph2/grader_chain.py)

`retrieval_grader_chain` —— 文档相关性评分（yes/no）。

#### 6.7.6 [graph2/grade_documents_node.py](../graph2/grade_documents_node.py)

对每个 document 调一次 `retrieval_grader_chain`，仅保留 `binary_score == 'yes'` 的；其余丢弃。

#### 6.7.7 [graph2/transform_query_node.py](../graph2/transform_query_node.py)

```python
better_question = question_rewriter.invoke({"question": question})
return {"documents": documents, "question": better_question, "transform_count": transform_count+1}
```

**关键点**：递增 `transform_count`，防止无限循环。

#### 6.7.8 [graph2/generate_node2.py](../graph2/generate_node2.py)

```python
def format_docs(docs):
    if isinstance(docs, list):
        return "\n\n".join(doc.page_content for doc in docs)
    else:
        return "\n\n" + docs.page_content

rag_chain = prompt | llm | StrOutputParser()
generation = rag_chain.invoke({"context": format_docs(documents), "question": question})
```

> 与 V1 的 `generate_node` 不同：V2 显式处理 list/单个 doc 两种情况。

#### 6.7.9 [graph2/grade_hallucinations_chain.py](../graph2/grade_hallucinations_chain.py)

```python
class GradeHallucinations(BaseModel):
    binary_score: str = Field(description="回答是否基于事实，取值为'yes'或'no'")
```

Prompt：评估生成内容是否基于/支持于给定事实集。

#### 6.7.10 [graph2/grade_answer_chain.py](../graph2/grade_answer_chain.py)

```python
class GradeAnswer(BaseModel):
    binary_score: str = Field(description="回答是否解决了问题，取值为'yes'或'no'")
```

#### 6.7.11 [graph2/graph_2.py](../graph2/graph_2.py)

**V2 完整工作流**：

```python
workflow.add_conditional_edges(START, route_question,
    {"web_search": "web_search", "vectorstore": "retrieve"})

workflow.add_edge("web_search", "generate")
workflow.add_edge("retrieve", "grade_documents")

workflow.add_conditional_edges('grade_documents', decide_to_generate)
# decide_to_generate:
#   - filtered_docs 为空 且 transform_count >= 2  → "web_search"
#   - filtered_docs 为空                          → "transform_query"
#   - filtered_docs 非空                          → "generate"

workflow.add_conditional_edges("generate", grade_generation_v_documents_and_question,
    {"not supported": "generate", "useful": END, "not useful": "transform_query"})

workflow.add_edge("transform_query", "retrieve")
```

**`grade_generation_v_documents_and_question`** 三态决策：

| 幻觉 | 答案匹配 | 路由 |
| --- | --- | --- |
| no | — | `not supported` → 重试 generate |
| yes | no | `not useful` → transform_query |
| yes | yes | `useful` → END |

#### 6.7.12 [graph2/graph_gradio.py](../graph2/graph_gradio.py)

Gradio 聊天界面：

- `execute_graph(chat_bot)` 异步执行工作流，追加 AI 回复；
- `do_graph(user_input, chat_bot)` 提交输入框；
- `css`：背景色 `#7FFFD4`，textarea 字号 24px；
- `instance.launch(debug=True)`。

### 6.8 search_tool —— Tavily 搜索测试

#### [search_tool/test_search.py](../search_tool/test_search.py)

占位测试文件（实际内容为模块级 `pass` 风格，主体 Tavily 封装已迁至 [llm_models/all_llm.py](../llm_models/all_llm.py)）。

### 6.9 test_load —— 文档加载器实验

| 文件 | 加载器 | 关键参数 | 输出 |
| --- | --- | --- | --- |
| [test_load/demo1.py](../test_load/demo1.py) | `PyPDFLoader` | `file_path` | 每页 1 Doc |
| [test_load/demo2.py](../test_load/demo2.py) | `UnstructuredLoader` | `strategy='hi_res', coordinates=True, partition_via_api=False` | 元素级 + JSON 输出 |
| [test_load/demo4.py](../test_load/demo4.py) | `UnstructuredMarkdownLoader` | `mode='elements', strategy='fast'` | 元素级 |
| [test_load/dome3.py](../test_load/dome3.py) | 自定义 `load_doc_from_json` | — | JSON → Document |

**`demo2.py` 的特殊处理**：
- `UnstructuredLoader(api_key='IhWKAZRBmZ14c8tmCsOLabqwIKLJ2e')` 演示 partition_via_api 风格（实际 `partition_via_api=False` 走本地）；
- 每个 element 写入 `datas/output/<page_number>_<counter>.json`；
- 演示从结果中筛 `category == 'Table'` 并打印 `text_as_html`。

### 6.10 test_milvus —— Milvus 原生 API 实验

#### [test_milvus/demo1.py](../test_milvus/demo1.py)

**演示完整 CRUD**：
- `MilvusClient(uri='http://1.95.116.112:19530')`
- `client.drop_collection('demo_collection')`
- `create_collection(name, dimension=384)`
- `insert(data)`
- `search(data, filter='subject == "history"', limit=2)`
- `query(filter=...)`
- `delete(filter=...)`

数据：3 段历史主题文本，384 维随机向量。

### 6.11 test_vector —— 向量检索实验

#### [test_vector/demo1.py](../test_vector/demo1.py)

`MilvusClient` + BM25 Function 全文搜索：
- 字段：`id, text(VARCHAR(2000), enable_analyzer=True), sparse(SPARSE_FLOAT_VECTOR)`；
- BM25 Function：`input=text, output=sparse, type=BM25, k1=1.6, b=0.75`；
- Collection 名：`t_demo2`；
- `search` 时 `drop_ratio_search=0.2` 忽略最低 20% 词权重。

#### [test_vector/demo2.py](../test_vector/demo2.py)

BGE Embedding 调用示例：
- `bge_embedding.embed_documents([...])` → 5 个 512 维向量；
- `bge_embedding.embed_query(...)` → 单条向量。

#### [test_milvus/demo1.py](../test_milvus/demo1.py)（同路径，含 9 个测试函数）

| test | 内容 |
| --- | --- |
| test1 | `similarity_search_with_score` 带过滤 |
| test2 | 创建 BM25 collection（'demo'） |
| test3 | 写入数据（LangChain `Milvus`） |
| test4 | LangChain BM25 全文搜索 |
| test5 | `MilvusClient.search` BM25 搜索 |
| test6 | `query` 过滤 category=='Title' |
| test7 | `pymilvus.AnnSearchRequest` + `RRFRanker(60)` 混合检索 |
| test8 | LangChain `similarity_search` + RRF |
| test9 | LangChain `as_retriever` + RRF + 过滤 |

### 6.12 test_mcp —— MCP 工具服务实验

#### [test_mcp/mcp_server.py](../test_mcp/mcp_server.py)

**FastMCP 服务端**：

| 注册 | 类型 | 说明 |
| --- | --- | --- |
| `mcp = FastMCP("Math")` | 实例 | MCP Server 名称 |
| `my_search` | Tool | 智谱 AI `web_search` 联网搜索 |
| `add` | Tool | 加法 |
| `multiply` | Tool | 乘法 |
| `get_user_email` | Resource | `datas://users/{user_id}/email` |
| `mcp.run(transport='sse')` | 启动 | 默认端口 8000 |

> `mcp.tool(name='my_search_tool', ...)` 显式命名；`add/multiply` 使用默认函数名。

#### [test_mcp/agent_client.py](../test_mcp/agent_client.py)

**命令行 MCP 客户端**：
- `MultiServerMCPClient({"weather": mcp_server_config})`；
- `tools = client.get_tools()`；
- `agent = create_tool_calling_agent(llm, tools, prompt)`；
- 三个测试问题：`计算 2 和 4的乘积` / `计算 6+19的结果` / `今天，北京的天气怎么样？`。

#### [test_mcp/mcp_app.py](../test_mcp/mcp_app.py)

**MCP + Gradio 聊天界面**：
- 与 `agent_client.py` 逻辑相同，但入口改为 Gradio 的 `Chatbot`；
- `chatbot` 类型 `messages`，高度 450；
- CSS 与 `graph_gradio.py` 一致。

#### [test_mcp/zhipu_agent.py](../test_mcp/zhipu_agent.py)

**本地工具版本（不走 MCP）**：
- 用 `@tool('my_search_tool', args_schema=SearchInput)` 把智谱搜索封装为 LangChain Tool；
- `agent_with_history` 部分被注释掉，只保留单轮 `executor.invoke({'input': '什么是EUV光刻机？'})`。

> 该文件主要演示"不用 MCP 时如何直接把智谱搜索做成 LangChain 工具"。

---

## 七、核心数据流（End-to-End）

### 7.1 离线建库

```
datas/md/*.md
  ↓ UnstructuredMarkdownLoader (mode=elements, strategy=fast)
  ↓ MarkdownParser.merge_title_content（父子合并）
  ↓ SemanticChunker（>5000 字符二次切分）
  ↓ MarkdownParser.text_chunker
chunk_documents (LangChain Document 列表)
  ↓ write_milvus.py: 多进程分批（每批 20 条）
  ↓ MilvusVectorSave.add_documents
  ↓ langchain_milvus.Milvus（自动 Embedding + BM25 Function）
Milvus t_collection01
  - dense: BAAI/bge-small-zh-v1.5 512 维（HNSW, IP）
  - sparse: BM25 稀疏向量（SPARSE_INVERTED_INDEX）
  - metadata: category, title, filename 等
```

### 7.2 在线问答（V2）

```
用户输入
  ↓ query_route_chain
  ├─ vectorstore → retrieve_node (Milvus 混合检索 RRF)
  │     ↓
  │   grade_documents_node（逐 doc 评分，过滤）
  │     ↓ decide_to_generate
  │     ├─ 全不相关 + transform_count >= 2 → web_search
  │     ├─ 全不相关                        → transform_query
  │     └─ 有相关                          → generate_node
  │                                          ↓
  │                                       grade_generation_v_documents_and_question
  │                                          ├─ not supported → generate（重试）
  │                                          ├─ not useful     → transform_query
  │                                          └─ useful         → END
  │
  └─ web_search → web_search_node (Tavily)
                    ↓
                  generate_node → 评估 → END
```

---

## 八、依赖与环境变量

### 8.1 安装

```bash
pip install -r requirements.txt
```

如果 `pymilvus` 安装失败，可换 `pip install pymilvus[milvus_lite]` 跑本地版。

### 8.2 环境变量（`.env` 文件放项目根目录）

```ini
OPENAI_API_KEY=sk-xxxx
DEEPSEEK_API_KEY=sk-xxxx
ZHIPU_API_KEY=xxxx
```

> `TAVILY_API_KEY` 在使用 `TavilySearchResults` 时也需配置（默认未在 .env 中显式设置，Tavily 会自动从环境变量读取）。

### 8.3 Milvus

默认 URI：`http://1.95.116.112:19530`（远程服务器）。
本地替换：

```python
# utils/env_utils.py
MILVUS_URI = './milvus.db'   # Milvus Lite
```

---

## 九、数据集说明

### 9.1 `datas/md/`

| 文件 | 类型 | 用途 |
| --- | --- | --- |
| `operational_faq.md` | 运维 FAQ | 解析测试 |
| `overview.md` | 概览文档 | 解析测试 |
| `performance_faq.md` | 性能 FAQ | 解析测试 |
| `product_faq.md` | 产品 FAQ | 解析测试 |
| `tech_report_0tfhhamx.md` | 技术报告 | 主入库文件 |
| `tech_report_0ui655n3.md` | 技术报告 | 主入库文件 |
| `troubleshooting.md` | 排错文档 | 解析测试 |

### 9.2 `datas/layout-parser-paper.pdf`

学术论文，用于 [test_load/demo1.py](../test_load/demo1.py) 和 [demo2.py](../test_load/demo2.py) 测试 PDF 解析。

### 9.3 `datas/output/`

`test_load/demo2.py` 通过 UnstructuredLoader 输出的元素级 JSON 文件：
- 命名格式：`<page_number>_<counter>.json`；
- 涵盖 1 ~ 16 页，共 186 个元素；
- 可用 [test_load/dome3.py](../test_load/dome3.py) 的 `load_doc_from_json` 反序列化为 Document。

---

## 十、可视化产物

| 文件 | 来源 | 用途 |
| --- | --- | --- |
| [graph_rag1.png](../graph_rag1.png) | 课程资料 | V1 流程图（备份） |
| [graph_rag2.png](../graph_rag2.png) | 课程资料 | V2 流程图（备份） |
| [graph/graph_rag1-2.png](../graph/graph_rag1-2.png) | `draw_png.py` 导出 | V1 最新 |
| [graph2/graph_rag2.png](../graph2/graph_rag2.png) | `draw_png.py` 导出 | V2 最新 |

**重新生成**：

```python
# graph/graph1.py 取消注释
draw_graph(graph, 'graph_rag1-2.png')

# graph2/graph_2.py 取消注释
draw_graph(graph, 'graph_rag2.png')
```

---

## 十一、扩展与工程化建议

1. **会话持久化**：[graph/graph1.py](../graph/graph1.py) 用 `MemorySaver`，生产环境建议换 `SqliteSaver` / `PostgresSaver`。
2. **流式输出**：把 `stream_mode='messages'` 接到 Gradio `stream` 接口，实现 token 级流式。
3. **多模态**：Unstructured `hi_res` 策略已演示（[demo2.py](../test_load/demo2.py)），可扩展到图片/表格问答。
4. **多租户**：Milvus 支持 `partition_key`，可按业务线切分。
5. **可观测性**：开启 LangSmith（[t.py](../t.py) 已验证连通），追踪每次工作流的 token / latency。
6. **评估体系**：可基于 RAGAS / TruLens 在 `graph2` 上做离线评测。
7. **更智能的改写**：`transform_query_node` 当前只做"问题改写"，可改为多 Query 召回 + 去重。
8. **生产级部署**：用 FastAPI 包装工作流 + LangServe 暴露 API；用 Docker Compose 编排 Milvus。

---

> 📌 最小跑通链路：
> 1. `pip install -r requirements.txt`
> 2. 写 `.env`（OPENAI_API_KEY / ZHIPU_API_KEY）
> 3. `python documents/write_milvus.py`（建库）
> 4. `python graph2/graph_gradio.py`（启动 Web UI）
> 5. 浏览器访问 `http://localhost:7860`，输入"什么是 EUV 光刻机？" 测试。
