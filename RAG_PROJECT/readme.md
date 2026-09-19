# RAG 企业知识库项目（RAG_PROJECT）

> 一个面向**半导体 / 芯片制造**领域的 **企业级 RAG（Retrieval-Augmented Generation，检索增强生成）项目**。
> 项目以 LangChain + LangGraph + Milvus 为核心技术栈，覆盖了从文档解析、向量入库、混合检索、Agent 编排、自评估、网页搜索到 MCP 服务调用的完整 RAG 落地链路。

---

## 📑 目录（Table of Contents）

- [一、项目概览](#一项目概览)
- [二、技术栈与依赖](#二技术栈与依赖)
- [三、目录结构总览](#三目录结构总览)
- [四、子项目一览](#四子项目一览)
  - [子项目 1：环境与基础工具（utils）](#子项目-1环境与基础工具utils)
  - [子项目 2：LLM 与 Embedding 模型封装（llm_models）](#子项目-2llm-与-embedding-模型封装llm_models)
  - [子项目 3：文档解析与切分（documents）](#子项目-3文档解析与切分documents)
  - [子项目 4：Milvus 向量库构建与混合检索](#子项目-4milvus-向量库构建与混合检索)
  - [子项目 5：基础 RAG Agent（Tool Calling Agent）](#子项目-5基础-rag-agenttool-calling-agent)
  - [子项目 6：自评估 RAG 工作流（graph）](#子项目-6自评估-rag-工作流graph)
  - [子项目 7：高级 RAG 工作流 —— 混合检索 + 路由 + 自评估（graph2）](#子项目-7高级-rag-工作流--混合检索--路由--自评估graph2)
  - [子项目 8：Gradio Web UI（混合检索 + 自评估）](#子项目-8gradio-web-ui混合检索--自评估)
  - [子项目 9：网络搜索工具测试（search_tool）](#子项目-9网络搜索工具测试search_tool)
  - [子项目 10：文档加载器实验（test_load）](#子项目-10文档加载器实验test_load)
  - [子项目 11：Milvus 基础实验（test_milvus）](#子项目-11milvus-基础实验test_milvus)
  - [子项目 12：向量检索基础实验（test_vector）](#子项目-12向量检索基础实验test_vector)
  - [子项目 13：MCP 工具调用（test_mcp）](#子项目-13mcp-工具调用test_mcp)
- [五、流程图与可视化](#五流程图与可视化)
- [六、运行准备](#六运行准备)
- [七、数据集说明（datas/）](#七数据集说明datas)
- [八、课程笔记（docs/）](#八课程笔记docs)
- [九、扩展与延伸](#九扩展与延伸)

---

## 一、项目概览

本项目是一个 **以"半导体行业企业知识库"为主题** 的 RAG 综合实战工程，重点演示了：

1. **非结构化文档解析**：Markdown、PDF 通过 Unstructured / PyPDF 等加载器解析。
2. **语义切分 + 父子文档合并**：使用 LangChain `SemanticChunker` 与 Unstructured 的元素元数据合并标题与正文。
3. **混合检索（Hybrid Search）**：在 Milvus 中同时构建 **稠密向量（BGE）+ 稀疏向量（BM25）**，使用 RRF（Reciprocal Rank Fusion）排序。
4. **Agent 工具调用**：基于 LangChain `create_tool_calling_agent`，把检索工具暴露给大模型。
5. **LangGraph 工作流**：基于 LangGraph 编排"agent → 检索 → 文档相关性评估 → 改写 → 生成"流程。
6. **自评估 RAG（Self-RAG）**：对生成结果进行 **幻觉检测（hallucination）+ 答案相关性（answer grading）**。
7. **路由 + Web 搜索**：当问题不属于知识库范围时自动路由到 Web Search（Tavily）。
8. **MCP 工具服务**：通过 MCP（Model Context Protocol）暴露搜索/计算工具给 LangChain Agent。
9. **Gradio UI**：将 LangGraph 工作流接入 Web 聊天界面。

> 应用场景举例：用户提出"什么是 EUV 光刻机？"或"现在最先进的纳米级清洗技术是什么？"，系统优先从企业知识库（向量库）检索半导体技术报告，再由 LLM 生成答案。

---

## 二、技术栈与依赖

完整依赖见 [requirements.txt](../requirements.txt)。

| 类别 | 主要库 |
| --- | --- |
| LLM 框架 | `langchain`, `langchain-core`, `langchain-community`, `langchain-openai` |
| Agent / Graph | `langgraph`, `langgraph-checkpoint`, `langgraph-prebuilt` |
| 向量数据库 | `pymilvus`, `langchain-milvus` |
| Embedding | `langchain-huggingface` (BAAI/bge-small-zh-v1.5) + `langchain-openai` |
| 文档解析 | `unstructured`, `langchain-unstructured`, `pypdf` |
| 文本切分 | `langchain-experimental` (SemanticChunker), `langchain-text-splitters` |
| Web 搜索 | `langchain-community.tools.TavilySearchResults` |
| MCP | `langchain-mcp-adapters`, `mcp`, `zhipuai` |
| Web UI | `gradio` |
| 日志 | `loguru` |
| 工具 | `python-dotenv`, `numpy`, `pandas`, `requests`, `pydantic` |

---

## 三、目录结构总览

```
RAG_PROJECT/
├── agent/                     # 基础 Tool Calling Agent 子项目
├── documents/                 # 文档解析、切分、Milvus 封装
├── graph/                     # LangGraph 工作流1（自评估 RAG）
├── graph2/                    # LangGraph 工作流2（混合检索 + 路由 + 自评估）
├── llm_models/                # LLM、Embedding、WebSearch 工具封装
├── search_tool/               # 网络搜索工具测试
├── test_load/                 # 文档加载器实验（PDF / MD / JSON）
├── test_mcp/                  # MCP 服务端与客户端实验
├── test_milvus/               # Milvus 基础实验
├── test_vector/               # 向量检索（BM25 / 稀疏向量）实验
├── tools/                     # 检索工具（Agent 工具调用）
├── utils/                     # 工具类：日志 / 环境变量 / 打印
├── datas/                     # 数据集（MD 文档 + PDF + JSON 输出）
├── docs/                      # 课程笔记 PDF + 本 README
├── draw_png.py                # 工作流可视化（生成 Mermaid PNG）
├── graph_rag1.png             # graph 工作流图
├── graph_rag2.png             # graph2 工作流图
├── main.py                    # PyCharm 默认入口（demo）
├── t.py                       # LangSmith Hub Prompt 测试
└── requirements.txt           # 依赖列表
```

---

## 四、子项目一览

> 以下每个子项目都给出了**目标、关键文件、核心代码片段、与其它模块的关系**，并附上文件跳转链接。

---

### 子项目 1：环境与基础工具（utils）

**目标**：统一管理项目中的环境变量、日志、打印工具。

**关键文件**：

- [env_utils.py](../utils/env_utils.py) —— 读取 `.env` 中的 API Key、Milvus URI、Collection 名。
- [log_utils.py](../utils/log_utils.py) —— 基于 `loguru` 的彩色日志输出。
- [print_utils.py](../utils/print_utils.py) —— LangGraph 事件流打印辅助。

**核心代码片段**：

```python
# utils/env_utils.py
load_dotenv(override=True)
OPENAI_API_KEY = os.getenv('OPENAI_API_KEY')
DEEPSEEK_API_KEY = os.getenv('DEEPSEEK_API_KEY')
ZHIPU_API_KEY = os.getenv('ZHIPU_API_KEY')
MILVUS_URI = 'http://1.95.116.112:19530'
COLLECTION_NAME = 't_collection01'
```

**作用**：被 [llm_models](../llm_models)、[documents](../documents)、[graph](../graph) 等几乎所有模块依赖，是整个项目的"基础设施"。

---

### 子项目 2：LLM 与 Embedding 模型封装（llm_models）

**目标**：集中封装 ChatOpenAI、OpenAI Embedding、HuggingFace BGE Embedding、Tavily 搜索等外部模型。

**关键文件**：

- [all_llm.py](../llm_models/all_llm.py) —— `llm`（gpt-4o-mini）+ `web_search_tool`（Tavily）。
- [embeddings_model.py](../llm_models/embeddings_model.py) —— `openai_embedding` 和 `bge_embedding`。

**核心代码片段**：

```python
# llm_models/all_llm.py
llm = ChatOpenAI(
    temperature=0,
    model='gpt-4o-mini',
    api_key=OPENAI_API_KEY,
    base_url="https://xiaoai.plus/v1"
)
web_search_tool = TavilySearchResults(max_results=2)
```

```python
# llm_models/embeddings_model.py
bge_embedding = HuggingFaceEmbeddings(
    model_name="BAAI/bge-small-zh-v1.5",
    model_kwargs={"device": "cpu"},
    encode_kwargs={"normalize_embeddings": True}
)
```

**被谁调用**：[documents/milvus_db.py](../documents/milvus_db.py)（bge_embedding）、[graph/*](../graph)、[graph2/*](../graph2)、[tools/retriever_tools.py](../tools/retriever_tools.py)。

---

### 子项目 3：文档解析与切分（documents）

**目标**：将 Markdown 技术文档解析为 LangChain `Document` 对象，并按"语义 + 父子文档"策略切分。

**关键文件**：

- [markdown_parser.py](../documents/markdown_parser.py) —— 加载 `.md`、合并标题与内容、做语义切分。
- [milvus_db.py](../documents/milvus_db.py) —— 定义 Milvus Collection、创建稠密/稀疏索引、写入文档。
- [write_milvus.py](../documents/write_milvus.py) —— **多进程** 批量解析 + 写入 Milvus。

**核心流程（markdown_parser.py）**：

1. `UnstructuredMarkdownLoader(mode='elements', strategy='fast')` 把每个元素切分为 `Document`。
2. `merge_title_content`：根据 `category` 和 `parent_id` 把"标题元素"和后续"正文元素"合并到父 Title 文档中，并保留独立 `NarrativeText`。
3. `text_chunker`：对长度 > 5000 的文档使用 `SemanticChunker(percentile)` 二次切分。

**核心流程（milvus_db.py）**：

- 创建 Collection 字段：`id, text, category, source, filename, filetype, title, category_depth, sparse, dense`。
- 添加 BM25 Function 自动生成稀疏向量。
- 索引：稀疏向量用 `SPARSE_INVERTED_INDEX`（BM25），稠密向量用 HNSW（IP）。
- 通过 `langchain_milvus.Milvus` 包装为 LangChain VectorStore，支持混合检索（RRF）。

**核心流程（write_milvus.py）**：

- 进程1：`file_parser_process` 扫描目录 → 调用 `MarkdownParser` → 每 20 个文档入队。
- 进程2：`milvus_writer_process` 从队列取批 → 调用 `MilvusVectorSave.add_documents`。
- 用 `multiprocessing.Queue` 解耦解析与写入，提升海量数据入库效率。

**被谁调用**：[tools/retriever_tools.py](../tools/retriever_tools.py)、[agent/rag_agent.py](../agent/rag_agent.py)、[test_vector/*](../test_vector)、[test_milvus/*](../test_milvus)。

---

### 子项目 4：Milvus 向量库构建与混合检索

**目标**：构建稠密+稀疏混合索引，提供统一检索入口。

**入口**：[documents/milvus_db.py](../documents/milvus_db.py) 的 `MilvusVectorSave`。

**关键能力**：

| 能力 | 方法 | 说明 |
| --- | --- | --- |
| 建表 | `create_collection()` | 定义字段、BM25 Function、HNSW 索引 |
| 建连接 | `create_connection()` | 通过 langchain_milvus 创建 VectorStore |
| 写入 | `add_documents(docs)` | 写入 LangChain Document |
| 检索 | `as_retriever()` / `similarity_search_with_score()` | 支持 RRF 排序 + 过滤 |

**检索示例**（[tools/retriever_tools.py](../tools/retriever_tools.py)）：

```python
retriever = mv.vector_store_saved.as_retriever(
    search_type='similarity',
    search_kwargs={
        "k": 3,
        "score_threshold": 0.1,
        "ranker_type": "rrf",
        "ranker_params": {"k": 100},
        'filter': {"category": "content"}
    }
)
```

---

### 子项目 5：基础 RAG Agent（Tool Calling Agent）

**目标**：使用 LangChain **Tool Calling Agent** + 多轮会话记忆，回答基于知识库的问题。

**关键文件**：[agent/rag_agent.py](../agent/rag_agent.py)

**核心流程**：

1. 通过 `create_retriever_tool` 把 Milvus 检索器包装成 Tool。
2. `create_tool_calling_agent(llm, [retriever_tool], prompt)` 创建 Agent。
3. `AgentExecutor` 执行调用。
4. 使用 `RunnableWithMessageHistory` + 内存字典 `store` 实现 **多轮会话记忆**（`session_id='zs123'`）。

**被谁调用**：直接 `python agent/rag_agent.py` 运行示例：`什么是EUV光刻机？`。

---

### 子项目 6：自评估 RAG 工作流（graph）

**目标**：基于 **LangGraph** 编排"agent → 检索 → 文档相关性评估 → 改写 → 生成"的循环工作流，并支持 Checkpoint 持久化。

**关键文件**：

- [graph/graph_state1.py](../graph/graph_state1.py) —— `AgentState`（messages 列表）+ `Grade` Pydantic 模型。
- [graph/agent_node.py](../graph/agent_node.py) —— Agent 节点，调用 `llm.bind_tools`。
- [graph/generate_node.py](../graph/generate_node.py) —— 基于检索结果生成最终答案。
- [graph/rewrite_node.py](../graph/rewrite_node.py) —— 在文档不相关时改写问题。
- [graph/get_human_message.py](../graph/get_human_message.py) —— 从消息列表中取出最后一个 HumanMessage。
- [graph/graph1.py](../graph/graph1.py) —— 工作流主入口（编排 + 循环 REPL）。

**工作流结构**：

```
START
  ↓
[agent]            —— 决定调用工具 / 结束
  ↓ tools_condition
[retrieve]         —— ToolNode(retriever_tool)
  ↓
[grade_documents]  —— yes → generate；no → rewrite
  ↓
[generate] → END
[rewrite] → [agent] (循环)
```

**配套可视化**：[graph/graph_rag1-2.png](../graph/graph_rag1-2.png)（执行 `draw_png.draw_graph(graph, 'graph_rag1-2.png')` 生成）。

**亮点**：

- 使用 `MemorySaver` 实现 **checkpoint**，每个 thread 一份独立会话。
- `tools_condition` 自动判断 LLM 是否需要调用工具。
- `grade_documents` 是基于 LLM with structured output 的二元评分器（yes/no）。

**被谁调用**：直接 `python graph/graph1.py` 进入 REPL 多轮对话。

---

### 子项目 7：高级 RAG 工作流 —— 混合检索 + 路由 + 自评估（graph2）

**目标**：在 `graph` 基础上增加 **问题路由（Query Routing）+ Web Search + 自我评估（生成是否基于文档/是否回答问题）** 的完整 Self-RAG 工作流。

**关键文件**：

| 文件 | 作用 |
| --- | --- |
| [graph2/graph_2.py](../graph2/graph_2.py) | 工作流主入口与三个路由函数 |
| [graph2/graph_state2.py](../graph2/graph_state2.py) | `GraphState`（question/transform_count/generation/documents） |
| [graph2/query_route_chain.py](../graph2/query_route_chain.py) | 路由器：判断走 web_search 还是 vectorstore |
| [graph2/web_search_node.py](../graph2/web_search_node.py) | 调用 Tavily 搜索 |
| [graph2/retriever_node.py](../graph2/retriever_node.py) | 调用 Milvus 混合检索 |
| [graph2/grade_documents_node.py](../graph2/grade_documents_node.py) | 过滤不相关文档 |
| [graph2/grader_chain.py](../graph2/grader_chain.py) | 文档相关性评分（yes/no） |
| [graph2/transform_query_node.py](../graph2/transform_query_node.py) | 改写问题（带 transform_count 计数） |
| [graph2/generate_node2.py](../graph2/generate_node2.py) | 基于上下文生成最终答案 |
| [graph2/grade_hallucinations_chain.py](../graph2/grade_hallucinations_chain.py) | 幻觉检测 |
| [graph2/grade_answer_chain.py](../graph2/grade_answer_chain.py) | 答案与问题匹配度评分 |

**工作流结构**：

```
START
  ↓ route_question
 ┌──────────────┐                ┌─────────────┐
 │ web_search   │                │ retrieve    │
 └──────┬───────┘                └──────┬──────┘
        ↓                              ↓
   [generate]                  [grade_documents]
        ↓                              ↓
        ↓                    decide_to_generate()
        ↓                       ↓           ↓
   grade_generation      [generate]   [transform_query]
        ↓                              (count>=2 → web_search)
   ┌────┼────────────┐                       ↓
   ↓    ↓            ↓                   [retrieve] (循环)
useful  not useful   not supported
   ↓     ↓            ↓
  END  [transform]   [generate] (重试)
        ↓
      [retrieve]
```

**三大评估器（GRADER 设计）**：

1. **retrieval_grader_chain**（[grader_chain.py](../graph2/grader_chain.py)）：评估检索文档是否相关。
2. **hallucination_grader_chain**（[grade_hallucinations_chain.py](../graph2/grade_hallucinations_chain.py)）：评估生成是否基于事实（避免幻觉）。
3. **answer_grader_chain**（[grade_answer_chain.py](../graph2/grade_answer_chain.py)）：评估生成是否回答了用户问题。

**循环控制**：
- 文档不相关 → `transform_query`（带 `transform_count` 计数器）；
- 当 `transform_count >= 2` 时自动转 `web_search`；
- 答案没用 → `transform_query` 重写问题并重新检索；
- 答案不基于文档 → 重试 `generate`；
- 评估通过 → END。

**配套可视化**：[graph2/graph_rag2.png](../graph2/graph_rag2.png)。

**被谁调用**：
- CLI：直接 `python graph2/graph_2.py`。
- Web UI：`graph2/graph_gradio.py`（[子项目 8](#子项目-8gradio-web-ui混合检索--自评估)）。

---

### 子项目 8：Gradio Web UI（混合检索 + 自评估）

**目标**：将 [子项目 7](#子项目-7高级-rag-工作流--混合检索--路由--自评估graph2) 的 LangGraph 工作流包装为 Gradio 聊天界面。

**关键文件**：[graph2/graph_gradio.py](../graph2/graph_gradio.py)

**核心功能**：

```python
with gr.Blocks(title='混合检索+自评估RAG', css=css) as instance:
    chatbot = gr.Chatbot(type='messages', height=350, label='AI客服')
    input_textbox = gr.Textbox(label='请输入你的问题📝', value='')
    input_textbox.submit(do_graph, [input_textbox, chatbot], [input_textbox, chatbot]) \
                   .then(execute_graph, chatbot, chatbot)
```

**运行**：`python graph2/graph_gradio.py`，浏览器访问 `http://localhost:7860`。

---

### 子项目 9：网络搜索工具测试（search_tool）

**目标**：测试 LangChain 的 Tavily 网络搜索能力，备用做"知识库外问题"的兜底。

**关键文件**：[search_tool/test_search.py](../search_tool/test_search.py)

> 实际使用 Tavily 的封装已并入 [llm_models/all_llm.py](../llm_models/all_llm.py) 中的 `web_search_tool`。

---

### 子项目 10：文档加载器实验（test_load）

**目标**：对比不同文档加载器（PyPDF / Unstructured / UnstructuredMarkdown / 自定义 JSON）。

**关键文件**：

| 文件 | 加载器 | 输出 |
| --- | --- | --- |
| [test_load/demo1.py](../test_load/demo1.py) | `PyPDFLoader` | 每页 1 个 Document |
| [test_load/demo2.py](../test_load/demo2.py) | `UnstructuredLoader` (hi_res, coordinates) | 元素级 + 坐标 |
| [test_load/demo4.py](../test_load/demo4.py) | `UnstructuredMarkdownLoader` (elements, fast) | 元素级（Title/NarrativeText/Table 等） |
| [test_load/dome3.py](../test_load/dome3.py) | 自定义 JSON → `Document` | 读取 [datas/output](../datas/output) 下的 JSON |

> demo2 会把每个 element 写入 `datas/output/<page>_<idx>.json`，便于调试元素级元数据。

---

### 子项目 11：Milvus 基础实验（test_milvus）

**目标**：演示 **PyMilvus 原生 API** 的基础操作（create/insert/search/query/delete）。

**关键文件**：[test_milvus/demo1.py](../test_milvus/demo1.py)

**演示要点**：
- `MilvusClient(uri='http://1.95.116.112:19530')` 连接远程 Milvus。
- 384 维随机向量 + 三段文本，演示 `insert/search/query/delete`。
- 过滤条件 `subject == 'history'`。

---

### 子项目 12：向量检索基础实验（test_vector）

**目标**：演示 **稀疏向量（BM25）+ 稠密向量** 混合检索的多种实现方式。

**关键文件**：

- [test_vector/demo1.py](../test_vector/demo1.py) —— `MilvusClient` BM25 Function 全文搜索。
- [test_vector/demo2.py](../test_vector/demo2.py) —— `HuggingFaceEmbeddings(BGE)` 调用示例。
- [test_milvus/test_search.py](../test_milvus/demo1.py) —— 涵盖 `similarity_search / similarity_search_with_score / hybrid_search / langchain hybrid` 等 9 个测试函数。

**重点 test 函数**：

| 函数 | 说明 |
| --- | --- |
| `test1` | LangChain 相似度搜索（带 score） |
| `test2` | 创建 BM25 collection |
| `test3` | 写入数据 |
| `test4` | LangChain BM25 全文搜索 |
| `test5` | `MilvusClient.search` BM25 搜索 |
| `test7` | `pymilvus.AnnSearchRequest` + RRF 混合检索 |
| `test8` | LangChain `similarity_search` + RRF |
| `test9` | LangChain `as_retriever` + RRF + 过滤 |

---

### 子项目 13：MCP 工具调用（test_mcp）

**目标**：通过 **MCP（Model Context Protocol）** 把自定义工具（搜索、加减乘、用户邮箱）暴露给 LangChain Agent。

**关键文件**：

| 文件 | 角色 |
| --- | --- |
| [test_mcp/mcp_server.py](../test_mcp/mcp_server.py) | **MCP 服务端**，注册 `my_search_tool`（智谱 AI 联网搜索）、`add` / `multiply` 工具，以及 `get_user_email` Resource |
| [test_mcp/mcp_app.py](../test_mcp/mcp_app.py) | **Agent + Gradio** 前端，通过 `MultiServerMCPClient` 远程连接 MCP |
| [test_mcp/agent_client.py](../test_mcp/agent_client.py) | **命令行版** Agent 客户端，演示 SSE 远程调用 |
| [test_mcp/zhipu_agent.py](../test_mcp/zhipu_agent.py) | **本地工具版本**（不通过 MCP），把智谱 AI 搜索作为本地 Tool |

**MCP Server 关键代码**：

```python
@mcp.tool(name='my_search_tool', description='搜索互联网上的内容')
def my_search(query: str) -> str:
    response = zhipu_client.web_search.web_search(
        search_engine="search-pro",
        search_query=query
    )
    return "\n\n".join([d.content for d in response.search_result])

@mcp.tool()
def add(a: int, b: int) -> int:
    return a + b
```

**MCP Client 关键代码**：

```python
weather_server_config = {
    "url": "http://localhost:8000/sse",
    "transport": "sse"
}
async with MultiServerMCPClient({"weather": weather_server_config}) as client:
    tools = client.get_tools()
    agent = create_tool_calling_agent(llm, tools, prompt)
    executor = AgentExecutor(agent=agent, tools=tools)
    response = await executor.ainvoke({"input": "计算 2 和 4的乘积"})
```

**运行步骤**：
1. 启动服务端：`python test_mcp/mcp_server.py`（默认 SSE，端口 8000）。
2. 启动客户端：`python test_mcp/agent_client.py` 或 `python test_mcp/mcp_app.py`。

---

## 五、流程图与可视化

| 图 | 路径 | 来源 |
| --- | --- | --- |
| LangGraph 工作流1 | [graph_rag1-2.png](../graph/graph_rag1-2.png) | 由 [draw_png.draw_graph](../draw_png.py) 生成，源自 [graph/graph1.py](../graph/graph1.py) |
| LangGraph 工作流2 | [graph_rag2.png](../graph2/graph_rag2.png) | 同上，源自 [graph2/graph_2.py](../graph2/graph_2.py) |
| 工作流根目录图 | [graph_rag1.png](../graph_rag1.png)、[graph_rag2.png](../graph_rag2.png) | 课程资料中已导出的 PNG |

**重新生成**：

```bash
# 在 graph/graph1.py 取消注释 draw_graph 行
python graph/graph1.py
# 在 graph2/graph_2.py 取消注释 draw_graph 行
python graph2/graph_2.py
```

---

## 六、运行准备

1. **克隆项目并安装依赖**：

```bash
pip install -r requirements.txt
```

2. **配置 `.env`**（项目根目录）：

```ini
OPENAI_API_KEY=sk-...
DEEPSEEK_API_KEY=sk-...
ZHIPU_API_KEY=...
```

3. **启动 Milvus**（远程或本地均可），修改 [utils/env_utils.py](../utils/env_utils.py) 中的 `MILVUS_URI`。

4. **构建向量库**：

```bash
python documents/write_milvus.py    # 多进程批量入库
# 或小批量验证
python documents/milvus_db.py
```

5. **启动对话**：

```bash
# 基础 Agent
python agent/rag_agent.py

# LangGraph 自评估 RAG（命令行）
python graph/graph1.py
python graph2/graph_2.py

# Gradio Web UI
python graph2/graph_gradio.py
python test_mcp/mcp_app.py
```

---

## 七、数据集说明（datas/）

`datas/` 下存放了项目使用的全部数据。

| 子目录 | 文件 | 说明 |
| --- | --- | --- |
| `datas/md/` | `operational_faq.md` / `overview.md` / `performance_faq.md` / `product_faq.md` / `tech_report_0tfhhamx.md` / `tech_report_0ui655n3.md` / `troubleshooting.md` | 半导体行业技术报告 / FAQ（被 UnstructuredMarkdownLoader 解析） |
| `datas/layout-parser-paper.pdf` | —— | Layout-Parser 论文，用于 `test_load/demo1.py` & `demo2.py` |
| `datas/output/` | `1_0.json` ~ `16_186.json` | `test_load/demo2.py` 通过 UnstructuredLoader 输出的元素级 JSON（按页号_序号命名） |

---

## 八、课程笔记（docs/）

`docs/` 目录下保存了完整的课程笔记 PDF：

- [9.11--RAG企业知识库项目.pdf](./9.11--RAG企业知识库项目.pdf)
- [9.12--RAG企业知识库项目.pdf](./9.12--RAG企业知识库项目.pdf)
- [9.14--RAG企业知识库项目.pdf](./9.14--RAG企业知识库项目.pdf)
- [9.15--RAG企业知识库项目.pdf](./9.15--RAG企业知识库项目.pdf)

> 这些 PDF 是扫描件，本 README 已基于代码反推还原了对应章节的核心内容。

---

## 九、扩展与延伸

- **多模态**：可接入 Unstructured 的 `hi_res` 策略解析 PDF 中的图表（已通过 [test_load/demo2.py](../test_load/demo2.py) 输出元素级 JSON）。
- **多租户**：可在 Milvus 中按 `partition_key` 切分不同业务线。
- **会话持久化**：[graph/graph1.py](../graph/graph1.py) 使用 `MemorySaver`；可替换为 `SqliteSaver`/`PostgresSaver` 实现跨进程持久化。
- **流式输出**：可把 LangGraph 的 `stream_mode='messages'` 接到 Gradio 的 `stream` 接口，实现 token 级流式。
- **可观测性**：可通过 `LANGSMITH_*` 环境变量开启 LangSmith 追踪（[t.py](../t.py) 测试了 `hub.pull("rlm/rag-prompt")`）。
- **更多评估**：可在 `graph2` 基础上加入 RAGAS / TruLens 等自动评估。

---

> 📌 如果你只想跑通最小链路：先 `pip install -r requirements.txt` → 配置 `.env` → `python documents/write_milvus.py` 建库 → `python graph2/graph_gradio.py` 启动 UI。
