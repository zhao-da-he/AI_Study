# MCP 完全学习指南（新手小白版）

> 本文档面向 **AI 工程师新手**，从零开始系统讲解 MCP（Model Context Protocol）。
> 配套本项目 [test_mcp/](../test_mcp/) 目录的代码实战，所有跳转链接使用 `file:///` 协议。

---

## 📚 目录

- [一、MCP 是什么？](#一mcp-是什么)
- [二、为什么要用 MCP？](#二为什么要用-mcp)
- [三、MCP 核心概念](#三mcp-核心概念)
- [四、MCP 工作流程](#四mcp-工作流程)
- [五、本项目 MCP 文件详解](#五本项目-mcp-文件详解)
- [六、本项目 MCP 代码逐行讲解](#六本项目-mcp-代码逐行讲解)
- [七、如何运行本项目的 MCP 实验？](#七如何运行本项目的-mcp-实验)
- [八、MCP 跟 LangChain Tool 的区别](#八mcp-跟-langchain-tool-的区别)
- [九、MCP 进阶：stdio 模式](#九mcp-进阶stdio-模式)
- [十、MCP 跟其他协议对比](#十mcp-跟其他协议对比)
- [十一、常见问题 FAQ](#十一常见问题-faq)
- [十二、学习路线图](#十二学习路线图)

---

## 一、MCP 是什么？

### 1.1 一句话定义

> **MCP = Model Context Protocol（模型上下文协议）**
> Anthropic 公司（Claude 母公司）2024 年推出的 **AI 工具共享标准**

### 1.2 通俗类比

| 类比 | 解释 |
| --- | --- |
| 🔌 **USB-C 接口** | 不管什么设备，都能用同一种线 |
| 🏪 **工具出租店** | AI 来借工具，店家统一管理 |
| 📞 **114 查号台** | 一个号码能查到各种服务 |
| 🛒 **外卖平台** | 餐厅不用对接每个 App，统一通过平台 |

### 1.3 核心思想

```
没有 MCP 之前：
   AI 框架 A  ──→ 工具 1（要专门写对接代码）
   AI 框架 B  ──→ 工具 1（要重新写一遍！）
   AI 框架 C  ──→ 工具 1（又要重新写！）

有 MCP 之后：
   AI 框架 A  ─┐
   AI 框架 B  ─┼──→ MCP 协议 ──→ 工具 1（一次开发，所有框架通用）
   AI 框架 C  ─┘
```

### 1.4 MCP 出现的背景

| 问题 | 解决 |
| --- | --- |
| 每个 AI 框架都要单独对接工具 | 统一标准 |
| 工具升级要改 N 个框架 | 改一次就行 |
| 跨框架工具难共享 | 一次开发，到处用 |
| 团队/企业想共享工具 | 大家用同一套接口 |

> 💡 类比：HTTP 协议让任何浏览器都能访问任何网站，**MCP 让任何 AI 框架都能使用任何工具**。

---

## 二、为什么要用 MCP？

### 2.1 不使用 MCP 的痛苦

```python
# 项目 A：自己接"联网搜索"
from langchain_community.tools import TavilySearchResults
tavily = TavilySearchResults(max_results=2)

# 项目 C：也要接（代码重复！）
from langchain_community.tools import TavilySearchResults
tavily = TavilySearchResults(max_results=2)   # 又写一遍！

# 项目 D：还想升级搜索功能
# ↓ 要改 N 个项目 😩
```

### 2.2 使用 MCP 的好处

```python
# 工具店只写一次（test_mcp/mcp_server.py）
@mcp.tool()
def my_search(query: str) -> str:
    """搜索互联网上的内容"""
    return zhipu_client.web_search.web_search(search_query=query)

# 项目 A、B、C 都连这一个店
async with MultiServerMCPClient({
    "weather": {"url": "http://localhost:8000/sse"}
}) as client:
    tools = client.get_tools()    # A、B、C 都拿同一份工具

# 工具升级，只改 mcp_server.py 一处 🎉
```

### 2.3 MCP 的核心优势

| 优势 | 说明 |
| --- | --- |
| **一次开发** | 工具写一次，所有 AI 都能用 |
| **跨框架** | LangChain、Claude Desktop、Cursor 都能用 |
| **跨语言** | 服务端用 Python，客户端用 Node.js 也行 |
| **标准化** | 不用每家 AI 公司自己定义工具协议 |
| **生态化** | Anthropic、第三方都在贡献 MCP 服务器 |

---

## 三、MCP 核心概念

### 3.1 两个角色

| 角色 | 英文 | 干什么 | 类比 | 本项目文件 |
| --- | --- | --- | --- | --- |
| **MCP 服务器** | MCP Server | 提供工具 | 🏪 工具店 | [test_mcp/mcp_server.py](../test_mcp/mcp_server.py) |
| **MCP 客户端** | MCP Client | 调用工具 | 🛒 借工具的人 | [test_mcp/agent_client.py](../test_mcp/agent_client.py)、[test_mcp/mcp_app.py](../test_mcp/mcp_app.py) |

### 3.2 三种"可调用的东西"

| 类型 | 英文 | 说明 | 本项目例子 |
| --- | --- | --- | --- |
| **工具** | Tool | 可执行的函数 | `add(a, b)`、`my_search(query)` |
| **资源** | Resource | 可读取的数据 | `get_user_email(user_id)` |
| **提示词** | Prompt | 预设的 Prompt 模板 | （本项目没演示） |

### 3.3 通信协议

| 协议 | 英文 | 用途 | 特点 |
| --- | --- | --- | --- |
| **SSE** | Server-Sent Events | 远程通信（HTTP） | 服务器主动推数据 |
| **stdio** | Standard I/O | 本地进程间通信 | 最快，不需要网络 |
| **HTTP** | HTTP | 标准 HTTP（未来） | 通用 |

> 本项目用 **SSE**（远程通信），适合不同机器之间调用。

### 3.4 SSE 是什么？（小白科普）

> **SSE = Server-Sent Events**（服务器推送事件）

| 普通 HTTP | SSE |
| --- | --- |
| 客户端请求一次，服务器返回一次 | 服务器可以**多次推**数据给客户端 |
| 类比：发短信 | 类比：直播 |

> 用 SSE，AI 可以实时看到工具调用的进度。

---

## 四、MCP 工作流程

### 4.1 4 步流程

```
Step 1: 启动 mcp_server.py（店家开门，端口 8000）
   ↓
Step 2: 跑 agent_client.py 或 mcp_app.py（顾客进店）
   ↓
Step 3: 客户端通过 MCP 协议，自动发现店家提供的工具
   ↓
Step 4: 用户提问 → AI 自动选用合适的工具 → 返回结果
```

### 4.2 详细时序图

```
┌──────────┐         ┌──────────────┐         ┌──────────┐
│ MCP 客户端│         │  MCP 服务器   │         │   AI     │
│ (Python) │         │  (FastMCP)    │         │ (GPT)    │
└────┬─────┘         └──────┬───────┘         └────┬─────┘
     │                     │                     │
     │  1. 连接（list_tools）│                     │
     ├────────────────────>│                     │
     │                     │                     │
     │  返回工具列表         │                     │
     │<────────────────────┤                     │
     │  [add, my_search,    │                     │
     │   get_user_email]    │                     │
     │                     │                     │
     │  2. 用户提问："计算 2+3"                     │
     ├─────────────────────────────────────────>│
     │                     │                     │
     │                     │  AI 决定调 add(2,3)  │
     │                     │                     │
     │  3. 调用 add 工具   │                     │
     ├────────────────────>│                     │
     │                     │                     │
     │                     │  执行 add(2,3)      │
     │                     │  返回 5              │
     │                     │                     │
     │  4. 拿到结果         │                     │
     │<────────────────────┤                     │
     │                     │                     │
     │  5. 把结果给 AI      │                     │
     ├─────────────────────────────────────────>│
     │                     │                     │
     │                     │  AI 生成最终答案    │
     │                     │  "2+3=5"            │
     │<─────────────────────────────────────────┤
```

### 4.3 一句话总结

> 客户端连服务器 → 服务器亮出工具菜单 → AI 看菜单选工具 → 调用工具 → 返回答案

---

## 五、本项目 MCP 文件详解

### 5.1 文件清单

[test_mcp/](../test_mcp/) 目录有 4 个文件：

| 文件 | 角色 | 干什么 | 怎么用 |
| --- | --- | --- | --- |
| [test_mcp/mcp_server.py](../test_mcp/mcp_server.py) | 🏪 **店家** | 提供 3 个工具 + 1 个资源 | `python test_mcp/mcp_server.py` |
| [test_mcp/agent_client.py](../test_mcp/agent_client.py) | 🛒 **借工具的人（命令行）** | 用 Python 借工具 | `python test_mcp/agent_client.py` |
| [test_mcp/mcp_app.py](../test_mcp/mcp_app.py) | 🛒 **借工具的人（网页）** | 用 Gradio 借工具 | `python test_mcp/mcp_app.py` |
| [test_mcp/zhipu_agent.py](../test_mcp/zhipu_agent.py) | 🏠 **不借，自己买** | 直接 import 工具，不用 MCP | `python test_mcp/zhipu_agent.py` |

### 5.2 文件之间的关系

```
┌─────────────────────────────────────────────┐
│                                             │
│   mcp_server.py（店家）                       │
│   ┌──────────────────────────┐               │
│   │ Tools:                   │               │
│   │  - my_search_tool        │               │
│   │  - add                   │               │
│   │  - multiply              │               │
│   │ Resources:               │               │
│   │  - get_user_email        │               │
│   └──────────────────────────┘               │
│            ↓ 监听 8000 端口                   │
│                                             │
│   agent_client.py（顾客 - 命令行）             │
│   ┌──────────────────────────┐               │
│   │ 1. 连店家（localhost:8000）│               │
│   │ 2. 拿到 3 个工具           │               │
│   │ 3. AI 自动选用             │               │
│   └──────────────────────────┘               │
│                                             │
│   mcp_app.py（顾客 - 网页）                   │
│   ┌──────────────────────────┐               │
│   │ 同上 + Gradio 聊天界面    │               │
│   └──────────────────────────┘               │
│                                             │
│   zhipu_agent.py（自给自足版）                │
│   ┌──────────────────────────┐               │
│   │ 不连店家，直接 import 工具│ ← 对照实验     │
│   └──────────────────────────┘               │
│                                             │
└─────────────────────────────────────────────┘
```

---

## 六、本项目 MCP 代码逐行讲解

### 6.1 [test_mcp/mcp_server.py](../test_mcp/mcp_server.py) —— 店家

#### 导入

```python
from mcp.server.fastmcp import FastMCP        # MCP 服务端框架
from zhipuai import ZhipuAI                    # 智谱 AI 客户端
from utils.env_utils import ZHIPU_API_KEY       # 智谱 API Key
```

> 从 `utils.env_utils` 读 API Key（不放代码里）。

#### 创建 MCP 实例

```python
mcp = FastMCP("Math")                           # 店名叫 "Math"
zhipu_client = ZhipuAI(api_key=ZHIPU_API_KEY, base_url='https://open.bigmodel.cn/api/paas/v4/')
```

> 类比：**给工具店取名字 + 准备工具原料**。

#### 注册工具 1：联网搜索

```python
@mcp.tool(name='my_search_tool', description='搜索互联网上的内容')
def my_search(query: str) -> str:
    """搜索互联网上的内容"""
    response = zhipu_client.web_search.web_search(
        search_engine="search-pro",
        search_query=query
    )
    if response.search_result:
        return "\n\n".join([d.content for d in response.search_result])
    return '没有搜索到任何内容！'
```

| 装饰器 | 含义 |
| --- | --- |
| `@mcp.tool(name='my_search_tool')` | 把这个函数注册成 MCP 工具，名字叫 `my_search_tool` |
| `description` | 描述这个工具干啥（LLM 靠这个判断"该不该用"） |

#### 注册工具 2：加法

```python
@mcp.tool()
def add(a: int, b: int) -> int:
    """加法运算: 计算两个数字相加"""
    return a + b
```

#### 注册工具 3：乘法

```python
@mcp.tool()
def multiply(a: int, b: int) -> int:
    """乘法运算：计算两个数字相乘"""
    return a * b
```

#### 注册资源：查邮箱

```python
@mcp.resource("datas://users/{user_id}/email", name='get_user_email')
async def get_user_email(user_id: str) -> str:
    """检索给定用户ID的电子邮件地址"""
    emails = {"123": "alice@example.com", "456": "bob@example.com"}
    return emails.get(user_id, "not_found@example.com")
```

> 资源（Resource）跟工具（Tool）的区别：资源是"读数据"，工具是"执行操作"。

#### 启动服务

```python
if __name__ == "__main__":
    mcp.run(transport='sse')       # 用 SSE 协议，端口默认 8000
```

> 类比：**打开店门营业**。

### 6.2 [test_mcp/agent_client.py](../test_mcp/agent_client.py) —— 借工具（命令行）

#### 连接 MCP 服务

```python
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient

weather_server_config = {
    "url": "http://localhost:8000/sse",
    "transport": "sse"
}

async def main():
    async with MultiServerMCPClient({
        "weather": weather_server_config
    }) as client:
        tools = client.get_tools()        # 拿到店家提供的工具
```

> `MultiServerMCPClient` 可以同时连多个 MCP 服务器（用字典的 key 区分）。

#### 让 AI 用这些工具

```python
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from llm_models.all_llm import llm

prompt = ChatPromptTemplate.from_messages([
    ('system', '你是一个智能助手，尽可能的调用工具回答用户的问题'),
    MessagesPlaceholder(variable_name='chat_history', optional=True),
    ('human', '{input}'),
    MessagesPlaceholder(variable_name='agent_scratchpad', optional=True),
])

agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools)
```

#### 测试 3 个问题

```python
response1 = await executor.ainvoke({"input": "计算 2 和 4的乘积"})     # → multiply
response2 = await executor.ainvoke({"input": "计算 6+19的结果"})         # → add
response3 = await executor.ainvoke({"input": "今天，北京的天气怎么样？"}) # → my_search_tool
```

### 6.3 [test_mcp/mcp_app.py](../test_mcp/mcp_app.py) —— 借工具（网页）

跟 `agent_client.py` 几乎一样，**多了 Gradio 界面**：

```python
import gradio as gr

with gr.Blocks(title='调用MCP服务的Agent项目', css=css) as instance:
    chatbot = gr.Chatbot(type='messages', height=450)
    input_textbox = gr.Textbox(label='请输入你的问题📝')
    
    input_textbox.submit(do_graph, [input_textbox, chatbot], [input_textbox, chatbot]) \
                   .then(execute_graph, chatbot, chatbot)

if __name__ == '__main__':
    instance.launch(debug=True)
```

> 类比：把"命令行聊天"包装成"网页聊天"。

### 6.4 [test_mcp/zhipu_agent.py](../test_mcp/zhipu_agent.py) —— 自给自足版

> **不走 MCP**，直接 import 工具

```python
from langchain_core.tools import tool
from zhipuai import ZhipuAI
from utils.env_utils import ZHIPU_API_KEY

zhipu_client = ZhipuAI(api_key=ZHIPU_API_KEY, base_url='https://open.bigmodel.cn/api/paas/v4/')

class SearchInput(BaseModel):
    query: str = Field(description='需要搜索的内容或者关键词')

@tool('my_search_tool', args_schema=SearchInput)
def my_search(query: str) -> str:
    """搜索互联网上的内容"""
    response = zhipu_client.web_search.web_search(
        search_engine="search-std",
        search_query=query
    )
    if response.search_result:
        return "\n\n".join([d.content for d in response.search_result])
    return '没有搜索到任何内容！'
```

> 跟 `mcp_server.py` 里的 `my_search` 功能一样，但**不用开 MCP 服务**，直接调用。

---

## 七、如何运行本项目的 MCP 实验？

### 7.1 三步走

#### Step 1：启动 MCP 服务（店家开门）

```powershell
cd D:\BaiduNetdiskDownload\AI大模型工程师\02_应用篇\10_RAG企业知识库项目\00_课程资料\RAG_PROJECT2\RAG_PROJECT

python test_mcp/mcp_server.py
```

输出：

```
INFO:     Started server process
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000
```

#### Step 2（新终端）：运行命令行客户端

```powershell
python test_mcp/agent_client.py
```

输出：

```
[Tool: multiply(2, 4) → 8]
2 乘 4 等于 8

[Tool: add(6, 19) → 25]
6 加 19 等于 25

[Tool: my_search_tool('北京天气') → ...]
今天北京天气晴朗...
```

#### Step 3（可选）：运行 Gradio 客户端

```powershell
python test_mcp/mcp_app.py
```

浏览器打开 `http://localhost:7860`，看到聊天界面。

### 7.2 完整流程图

```
终端 1:                    终端 2:
python mcp_server.py    →  python agent_client.py
   ↓                         ↓
启动 8000 端口               连接 localhost:8000
   ↓                         ↓
等待客户端连接                拿到 3 个工具
   ↓                         ↓
收到请求 → 执行工具 → 返回     调用 AI → 自动选工具 → 调用 → 出答案
```

---

## 八、MCP 跟 LangChain Tool 的区别

| 维度 | **LangChain Tool** | **MCP Tool** |
| --- | --- | --- |
| **调用方式** | `tool.invoke(args)` | 远程 HTTP（`http://...`） |
| **跨进程** | ❌ 必须同进程 | ✅ 可以跨机器 |
| **跨语言** | ❌ 只支持 Python | ✅ 任何语言都能写服务端 |
| **标准** | LangChain 私有 | 行业开放标准 |
| **生态** | LangChain 工具库 | Anthropic + 社区 |
| **适合** | 单一项目 | 多项目/团队共享 |

### 通俗对比

| LangChain Tool | MCP Tool |
| --- | --- |
| 🏠 家里自用工具 | 🏪 店里租的工具 |
| 用完放回柜子 | 用完店家管 |
| 只能本家用 | 谁都能租 |

### 何时用哪个？

| 场景 | 推荐 |
| --- | --- |
| 个人项目、单一项目 | LangChain Tool（更简单） |
| 团队协作、多项目共享 | MCP Tool |
| 想用现成的工具（搜索、文件） | 找现成的 MCP 服务器 |
| 跨语言、跨机器 | MCP（必须） |

---

## 九、MCP 进阶：stdio 模式

### 9.1 stdio 是什么？

> **stdio = Standard I/O**（标准输入输出）
> 客户端和服务器**通过命令行通信**，不走网络

### 9.2 SSE vs stdio

| | **SSE（远程）** | **stdio（本地）** |
| --- | --- | --- |
| 通信方式 | HTTP | 命令行 |
| 速度 | 较慢（网络） | **最快**（内存） |
| 部署 | 需要服务器 | 一行命令启动 |
| 适合 | 跨机器 | 本地进程间 |
| 配置 | URL | stdio 标记 |

### 9.3 stdio 模式的客户端配置

```python
mcp_server_config = {
    "command": "python",                    # 启动命令
    "args": ["test_mcp/mcp_server.py"],    # 参数
    "transport": "stdio"                   # 通信方式
}
```

> **不需要手动启动服务器**！客户端会自动用 `python test_mcp/mcp_server.py` 启动服务端进程。

### 9.4 SSE 模式的配置（远程）

```python
mcp_server_config = {
    "url": "http://localhost:8000/sse",
    "transport": "sse"
}
```

> **需要先手动 `python mcp_server.py`** 启动服务端。

### 9.5 本项目为什么用 SSE？

- 本项目演示**远程通信**（更通用）
- 实际生产中 SSE 更常见（多个客户端连一个服务端）
- stdio 主要用于本地开发

---

## 十、MCP 跟其他协议对比

### 10.1 MCP vs OpenAI Function Calling

| 维度 | Function Calling | MCP |
| --- | --- | --- |
| 出品 | OpenAI | Anthropic |
| 范围 | 工具调用（一次） | 工具共享（持续） |
| 标准 | OpenAI 私有 | 开放 |
| 生态 | GPT 系列 | Claude + LangChain + 社区 |

> **Function Calling 是 MCP 的子集**。MCP 是更上层的协议。

### 10.2 MCP vs REST API

| 维度 | REST API | MCP |
| --- | --- | --- |
| 调用者 | 程序员手动 | AI 自动 |
| 接口 | URL + HTTP 方法 | 工具列表 + 参数 |
| 适合 | 业务系统对接 | AI 应用集成 |

### 10.3 MCP vs gRPC

| 维度 | gRPC | MCP |
| --- | --- | --- |
| 性能 | 极快 | 中等 |
| 复杂度 | 高 | 低 |
| 适合 | 微服务 | AI 工具集成 |

### 10.4 总结

```
OpenAI FC  ←── 单次调用工具
MCP       ←── AI 工具共享标准（FC 的超集）
REST API  ←── 业务系统对接
gRPC      ←── 高性能微服务
WebSocket ←── 实时双向通信
```

> **MCP 的定位：AI 时代的"USB-C 接口"**。

---

## 十一、常见问题 FAQ

### Q1：MCP 只能用 Claude 吗？

**不是**！MCP 是开放协议：

| 支持 MCP 的客户端 |
| --- |
| Claude Desktop ✅ |
| Cursor（代码编辑器）✅ |
| LangChain ✅ |
| Cline（前 Claude Dev）✅ |
| 自研 AI 系统 ✅ |

### Q2：MCP 安全吗？

| 风险 | 解决 |
| --- | --- |
| 未授权访问 | 加 API Key、OAuth |
| 中间人攻击 | 用 HTTPS |
| 服务端不可信 | 验证来源 |

### Q3：MCP 跟 LangChain 的 @tool 装饰器有啥区别？

```python
# LangChain 的 @tool（本地）
@tool
def add(a: int, b: int) -> int:
    return a + b

# MCP 的 @mcp.tool（远程）
@mcp.tool()
def add(a: int, b: int) -> int:
    return a + b
```

代码看起来一样，但：
- LangChain 的 @tool：函数在当前进程
- MCP 的 @mcp.tool：函数在另一个进程（或机器）

### Q4：MCP 服务器怎么调试？

```python
@mcp.tool()
def add(a, b):
    print(f"add 被调用: a={a}, b={b}")    # 加 print 调试
    return a + b
```

或者用 MCP Inspector（官方调试工具）：

```bash
npx @modelcontextprotocol/inspector
```

### Q5：MCP 跟 Function Calling 冲突吗？

**不冲突**！MCP 底层通常**基于** Function Calling 实现。

```
MCP Server → Function Calling → LLM
```

### Q6：MCP 服务端用什么语言写？

| 语言 | 支持情况 |
| --- | --- |
| Python | ✅（本项目用） |
| TypeScript | ✅（官方支持） |
| Go | ✅ |
| Java | ✅ |
| Rust | ✅ |

### Q7：MCP 服务端必须用 SSE 吗？

**不是**。本项目演示用 SSE，还有：

| 模式 | 启动方式 |
| --- | --- |
| `sse` | 远程 HTTP（端口） |
| `stdio` | 客户端自动启动子进程 |

### Q8：本项目不启动 MCP 服务能跑 RAG 吗？

**能**！MCP 是**附加功能**，不影响主流程。

主流程入口：

```bash
python graph2/graph_gradio.py    # V2 工作流 + Gradio（不用 MCP）
```

### Q9：生产环境怎么部署 MCP？

- 用 Docker 打包服务端
- 用 nginx/Envoy 做反向代理
- 配 HTTPS + API Key
- 用 K8s 做高可用

### Q10：MCP 跟 WebSocket 对比？

| | MCP | WebSocket |
| --- | --- | --- |
| 用途 | AI 工具调用 | 实时双向通信 |
| 协议 | MCP | WS |
| 适合 | AI 应用 | 聊天/游戏 |

---

## 十二、学习路线图

### 阶段 1：理解 MCP 是什么（半天）

```
□ 读懂本文档第一、二章
□ 理解"USB-C"类比
□ 知道 MCP 是 Anthropic 提出的
```

### 阶段 2：跑通本项目实验（1 小时）

```
□ 读懂 [test_mcp/mcp_server.py](../test_mcp/mcp_server.py)
□ 启动服务端
□ 跑通客户端
□ 试着用 3 个工具
```

### 阶段 3：理解 MCP 协议（半天）

```
□ 看官方文档：https://modelcontextprotocol.io
□ 学习 JSON-RPC 2.0（底层协议）
□ 理解 SSE / stdio 区别
```

### 阶段 4：写自己的 MCP 服务器（半天）

```python
from mcp.server.fastmcp import FastMCP
mcp = FastMCP("MyTools")

@mcp.tool()
def my_tool(arg: str) -> str:
    """我的自定义工具"""
    return f"处理 {arg}"

mcp.run(transport='stdio')
```

### 阶段 5：集成到 Claude Desktop（1 小时）

```json
# ~/Library/Application Support/Claude/claude_desktop_config.json
{
  "mcpServers": {
    "my_server": {
      "command": "python",
      "args": ["my_server.py"]
    }
  }
}
```

### 阶段 6：生产部署（进阶）

```
□ Docker 化
□ HTTPS 证书
□ K8s 集群
□ 监控告警
```

---

## 终极类比

把 MCP 比作"**AI 工具界的 USB-C**"：

| MCP | USB-C |
| --- | --- |
| 🔌 一种接口标准 | 🔌 一种接口标准 |
| 🤖 AI 框架都能用 | 📱 手机都能用 |
| 🛠️ 各种工具都能接 | 💾 各种设备都能接 |
| 🔗 一次开发，多处用 | 🔗 一根线，所有设备通用 |

> 🎯 **一句话总结**：
>
> **MCP = AI 工具共享的统一标准**（Anthropic 提出）。
> 让任何 AI 框架都能用任何工具，不用重复对接代码。
> 本项目 [test_mcp/](../test_mcp/) 演示了完整的"店家 + 顾客"流程。