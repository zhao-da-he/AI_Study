# LangGraph 完全入门指南（小白版）

> 写给"完全没接触过 LangGraph"的同学。这份文档会**手把手**告诉你：
> - LangGraph 到底是什么、解决什么问题
> - 它和普通 Python 写法、LangChain 写法有什么区别
> - 怎么读懂别人写的 LangGraph 智能体代码
> - 怎么自己写一个最简的 LangGraph 智能体

---

## 📌 目录

- [1. LangGraph 是什么？](#1-langgraph-是什么)
- [2. 为什么需要 LangGraph？](#2-为什么需要-langgraph)
- [3. 3 个核心概念：State / Node / Edge](#3-3-个核心概念state--node--edge)
- [4. 看图说话：一张 LangGraph 状态机长什么样](#4-看图说话一张-langgraph-状态机长什么样)
- [5. 5 分钟速通：写一个"笑话 + 翻译"双节点工作流](#5-5-分钟速通写一个笑话--翻译双节点工作流)
- [6. 进阶：条件分支、中断、人工确认](#6-进阶条件分支中断人工确认)
- [7. 看懂本项目里那张大图](#7-看懂本项目里那张大图)
- [8. 怎么读懂别人写的 LangGraph 代码（7 步法）](#8-怎么读懂别人写的-langgraph-代码7-步法)
- [9. 常见概念速查表](#9-常见概念速查表)

---

## 1. LangGraph 是什么？

**一句话**：LangGraph 是一个"**用写流程图的方式来编排 AI 智能体**"的 Python 库。

你可以把它理解成：

> 把"让 AI 干活"这件事，从"写一堆 if-else + while 循环"，
> 升级成"画一张流程图（节点 + 边）"，让框架按图执行。

来源：LangChain 团队出品，是 [LangChain](https://python.langchain.com/) 生态里专门负责"**智能体流程编排**"的子库。

---

## 2. 为什么需要 LangGraph？

在 LangGraph 出现之前，让一个 AI 智能体"调用工具"通常是这样写的：

```python
# ❌ 传统写法：纯 Python 流程
def run_agent(user_input):
    messages = [user_input]
    for i in range(10):  # 最多循环 10 次
        response = llm.invoke(messages)
        if response.tool_calls:
            for call in response.tool_calls:
                result = tool_registry[call.name].invoke(call.args)
                messages.append(ToolMessage(result))
        else:
            return response.content
    return "超出最大步数"
```

**问题**：
- 循环嵌套、状态散落一地
- 想加个"问用户确认"、想加个"分支判断"非常麻烦
- 多智能体协作（主代理 + 子代理）几乎写不下去

**LangGraph 的解决方案**：

```
你只需要"画图"：
   ┌─────┐    ┌─────┐    ┌─────┐
   │ 问  │ →  │ 思考 │ →  │ 调工具 │ →  ...  →  │ 答  │
   └─────┘    └─────┘    └─────┘
   ↓
   State (共享状态) 在节点之间自动流动
```

每一格（"节点"）就是一小段 Python 函数；每条箭头（"边"）就是"我执行完去哪儿"。

---

## 3. 3 个核心概念：State / Node / Edge

| 概念 | 通俗解释 | 对应代码 |
| --- | --- | --- |
| **State（状态）** | 一张"公共白板"，所有节点都从这上面读、往这上面写 | `class State(TypedDict)` |
| **Node（节点）** | 一段干活的 Python 函数（输入 state，输出"对 state 的修改"） | `def my_node(state): return {...}` |
| **Edge（边）** | 节点之间的箭头：决定"我执行完去下一个节点" | `builder.add_edge("A", "B")` |

**记忆口诀**：
- **State** 是 **共享变量**
- **Node** 是 **函数**
- **Edge** 是 **流程**

### 3.1 详解 State

State 用 `TypedDict` 定义，**每个字段是节点之间共享的数据**。

```python
from typing import TypedDict, Annotated
from langgraph.graph import add_messages

class State(TypedDict):
    # messages 字段：所有聊天历史（自动用 add_messages reducer 追加）
    messages: Annotated[list, add_messages]
    # user_info 字段：用户信息
    user_info: str
    # dialog_state 字段：当前在哪个子代理（栈）
    dialog_state: list[str]
```

**关键点**：
- `Annotated[list, add_messages]` 里的 `add_messages` 是个 **reducer**——告诉 LangGraph：节点返回新消息时，**追加**到 messages 里，**不是覆盖**。
- State 就是普通的 dict，节点读 `state['xxx']`、返回 `{...}` 表示"修改这些字段"。

### 3.2 详解 Node

Node 就是个**普通的 Python 函数**，签名约定：

```python
def my_node(state: State) -> dict:
    """读 state，干活，返回要修改的字段"""
    user_input = state['user_input']
    result = do_something(user_input)
    return {'output': result}    # 返回 dict，LangGraph 会合并回 state
```

LangGraph 拿到返回值后，**自动**把它合并到 state 里：
- `Annotated[list, add_messages]` 的字段 -> 追加
- 其它字段 -> 直接覆盖

### 3.3 详解 Edge

边分两种：

| 类型 | 写法 | 含义 |
| --- | --- | --- |
| **固定边** | `builder.add_edge("A", "B")` | A 执行完 **一定** 去 B |
| **条件边** | `builder.add_conditional_edges("A", router_fn, {...})` | A 执行完看 `router_fn` 的返回值决定去哪 |

```python
# 固定
builder.add_edge("assistant", END)        # 执行完 -> 结束

# 条件
builder.add_conditional_edges(
    "assistant",                             # 起点节点
    lambda state: "tools" if state.get("tool_calls") else END,
    {"tools": "tools", END: END}             # 可能的目标（key 是 router 的返回值）
)
```

---

## 4. 看图说话：一张 LangGraph 状态机长什么样

```
   ┌──────────┐
   │   START  │ （图起点）
   └────┬─────┘
        ↓
   ┌──────────┐
   │ 读取用户 │  （Node：fetch_user_info）
   │  航班信息 │
   └────┬─────┘
        ↓
   ┌──────────┐    调工具     ┌────────┐
   │  主助理  │  ─────────→  │  工具  │
   │ (主思考) │  ←─────────  │  (执行) │
   └────┬─────┘    工具结果    └────────┘
        ↓
     决定下一步（条件边）：
       - 调用了"跳转子代理"工具？→ 进入子代理
       - 调用了普通工具？       → 主助理工具节点
       - 都没调？              → 结束
```

每个方框就是一个 **Node**（Python 函数）；每条箭头就是一条 **Edge**（可能是固定的、也可能是带条件的）。

---

## 5. 5 分钟速通：写一个"笑话 + 翻译"双节点工作流

我们来写一个**最简单的 LangGraph** —— 让 AI 先讲个笑话，再翻译成英文。

```python
# ---------- 1. 定义 State ----------
from typing import TypedDict, Annotated
from langgraph.graph import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]   # 聊天历史

# ---------- 2. 定义节点 ----------
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o-mini")

def tell_joke(state: State):
    """节点1：让 AI 讲个笑话"""
    response = llm.invoke(state["messages"] + [{"role": "user", "content": "讲个笑话"}])
    return {"messages": [response]}

def translate_to_english(state: State):
    """节点2：把上一步的笑话翻译成英文"""
    response = llm.invoke(state["messages"] + [{"role": "user", "content": "翻译成英文"}])
    return {"messages": [response]}

# ---------- 3. 画图 ----------
from langgraph.graph import StateGraph, START, END

builder = StateGraph(State)
builder.add_node("joke", tell_joke)                # 注册节点
builder.add_node("translate", translate_to_english)
builder.add_edge(START, "joke")                   # 起点 → joke
builder.add_edge("joke", "translate")              # joke → translate
builder.add_edge("translate", END)                 # translate → 结束

# ---------- 4. 编译（让图"动起来"）----------
graph = builder.compile()

# ---------- 5. 调用 ----------
result = graph.invoke({"messages": [{"role": "user", "content": "你好"}]})
print(result["messages"][-1].content)
```

**输出**：
```
为什么程序员总是穿黑色衣服？
Because they don't like "light mode"... 😄

翻译: Why do programmers always wear black?
Because they don't like "light mode"...
```

**短短 20 行**就完成了一个"AI 流程"！

---

## 6. 进阶：条件分支、中断、人工确认

### 6.1 条件分支

```python
def should_continue(state: State) -> str:
    """判断下一步去哪"""
    last_msg = state["messages"][-1]
    if last_msg.tool_calls:
        return "tools"     # 有工具调用 -> 去工具节点
    return END             # 没工具调用 -> 结束

builder.add_conditional_edges(
    "assistant",           # 起点节点
    should_continue,       # 路由函数：返回字符串
    {"tools": "tools", END: END}   # 字符串 -> 目标节点 的映射
)
```

**这就是"让 AI 自己决定下一步去哪"的关键。**

### 6.2 中断（Interrupt）+ 人工确认

对于"敏感操作"（删数据、转账、改机票），你**一定要在执行前停下来问用户**：

```python
graph = builder.compile(
    checkpointer=memory,   # 内存检查点：保存"暂停时的现场"
    interrupt_before=[
        "delete_user",      # 在跑 delete_user 节点之前，强制停下来
    ]
)
```

`graph.stream(...)` 时：
1. 跑到 `delete_user` 之前，**暂停**，返回一个"中断"事件
2. 你的代码看到中断，去**问用户**：你确认要删除吗？
3. 用户说"是" -> 你再调 `graph.stream(None, config)` **继续**
4. 用户说"否" -> 你可以构造一个"假 ToolMessage"骗过去，让流程转向别的节点

**项目里的实际场景**（来自 `finally_graph.py`）：
```python
graph = builder.compile(
    checkpointer=memory,
    interrupt_before=[
        "update_flight_sensitive_tools",   # 改签前停下
        "book_hotel_sensitive_tools",      # 订酒店前停下
        "book_car_rental_sensitive_tools", # 租车前停下
        "book_excursion_sensitive_tools",   # 旅行前停下
    ]
)
```

这样前端就有机会显示"⚠️ 即将执行改签操作，是否同意？" 的提示。

### 6.3 工具调用的完整流程

这是 LangGraph 智能体最常见的模式：

```
   ┌──────────┐
   │ 用户输入  │
   └────┬─────┘
        ↓
   ┌──────────┐        ┌──────────┐
   │ 思考(LLM) │ ─────→ │ 调工具    │ ← LangGraph 自动调用 tool
   │           │ ←───── │ 拿结果    │
   └──┬────┬──┘        └──────────┘
      │    │
   还要调? │
      ↓    ↓否
   (回思考)  END
```

**怎么告诉 LangGraph 哪些函数是"工具"**？
```python
from langchain_core.tools import tool

@tool
def search_weather(city: str) -> str:
    """查询天气"""
    return f"{city}: 25度, 晴"

tools = [search_weather]              # 工具列表
llm_with_tools = llm.bind_tools(tools)  # 把工具"绑"给大模型
```

LLM 决定"我应该调 search_weather(city='北京')" 时，LangGraph 会**自动**调你的 Python 函数，把结果塞回 state，让 LLM 看结果再继续。

---

## 7. 看懂本项目里那张大图

打开 [`graph_chat/finally_graph.py`](../graph_chat/finally_graph.py)，你会看到一张"主代理 + 4 个子代理"的大图。**用我们刚学的概念来解读**：

```
   START
     ↓
   fetch_user_info     ← 第一个节点：查用户航班信息
     ↓
   (条件路由)
   ↓                ↓                ↓                ↓
 primary_      enter_update_   enter_book_      enter_book_     enter_book_
 assistant      flight          car_rental       hotel           excursion
     ↓                ↓                ↓                ↓                ↓
 (路由主代理   航班子代理      租车子代理      酒店子代理    旅行子代理
  的下一步)
   ↓
   (主代理工具：搜索、查航班、查政策)
   ↓
 END（如果不需要调子代理）
```

**关键代码片段**（[finally_graph.py](../graph_chat/finally_graph.py)）：

```python
# 1. 创建图
builder = StateGraph(State)

# 2. 添节点
builder.add_node('fetch_user_info', get_user_info)
builder.add_node('primary_assistant', CtripAssistant(assistant_runnable))
builder.add_node("primary_assistant_tools", ...)

# 3. 子图（4 个）也是 StateGraph，最后合并进来
builder = build_flight_graph(builder)     # 航班
builder = builder_hotel_graph(builder)     # 酒店
builder = build_car_graph(builder)        # 租车
builder = builder_excursion_graph(builder) # 旅行

# 4. 加边
builder.add_edge(START, 'fetch_user_info')         # 起点
builder.add_conditional_edges(                    # 主代理的条件路由
    'primary_assistant', route_primary_assistant,
    ['enter_update_flight', 'enter_book_car_rental',
     'enter_book_hotel', 'enter_book_excursion',
     'primary_assistant_tools', END]
)

# 5. 编译
graph = builder.compile(
    checkpointer=memory,
    interrupt_before=[...]   # 敏感操作前停下
)
```

**接下来看子图怎么构造**（[build_child_graph.py](../graph_chat/build_child_graph.py)）：

每个子代理都是同样的"套路"：
```python
def build_flight_graph(builder):
    # 1. 入口节点（主代理跳进来）
    builder.add_node("enter_update_flight", create_entry_node("航班助理", "update_flight"))
    # 2. 助理节点（跟用户对话）
    builder.add_node("update_flight", CtripAssistant(update_flight_runnable))
    # 3. 工具节点
    builder.add_node("update_flight_safe_tools", ...)
    builder.add_node("update_flight_sensitive_tools", ...)
    # 4. 退出节点（"我不干了，回主助理"）
    builder.add_node("leave_skill", pop_dialog_state)
    # 5. 边
    builder.add_edge("enter_update_flight", "update_flight")
    builder.add_edge("update_flight_sensitive_tools", "update_flight")
    builder.add_edge("update_flight_safe_tools", "update_flight")
    builder.add_edge("leave_skill", "primary_assistant")  # 回主助理
    # 6. 条件边：路由函数根据 LLM 调用的工具名决定走哪
    builder.add_conditional_edges("update_flight", route_update_flight, [...])
    return builder
```

读懂这个套路，你就读懂了**全部 4 个子代理**。

---

## 8. 怎么读懂别人写的 LangGraph 代码（7 步法）

看到一段 LangGraph 代码，按这 7 步读，**永远不出错**：

### Step 1: 找 `State` 定义
```python
class State(TypedDict):
    messages: ...
    user_info: ...
```
→ 问：**节点之间传什么数据？** 字段就是数据。

### Step 2: 找所有 `add_node(...)`
```python
builder.add_node("A", function_a)
builder.add_node("B", function_b)
```
→ 问：**有哪几个节点？每个节点是干什么的？**
→ 然后**逐个看对应的函数**：输入是什么、输出是什么。

### Step 3: 找所有 `add_edge(..., ...)`
```python
builder.add_edge("A", "B")    # A → B
```
→ 问：**哪些节点是"必须按顺序走"的？**

### Step 4: 找所有 `add_conditional_edges(...)`
```python
builder.add_conditional_edges(
    "A",                   # 起点
    router_fn,             # 路由函数
    {"key1": "B", "key2": "C"}   # 目标映射
)
```
→ 问：**A 执行完后**：
> - 看路由函数 `router_fn` 的 return 值（字符串）；
> - 在"目标映射"里查这个字符串对应的节点；
> - 去那个节点。

### Step 5: 找 `compile(...)`
```python
graph = builder.compile(
    checkpointer=...,        # 是否支持"中断恢复"？
    interrupt_before=[...],  # 哪些节点前会暂停？
)
```
→ 问：**运行时会暂停吗？在哪里暂停？**

### Step 6: 看 `invoke` / `stream` 调用
```python
graph.invoke({"messages": [...]})
# 或
for event in graph.stream({...}):
    ...
```
→ 问：**入口参数是什么？返回值是什么结构？**

### Step 7: 看"工具节点"是怎么构造的
```python
builder.add_node("tools", create_tool_node_with_fallback([tool1, tool2]))
```
→ 工具节点 = LangGraph **自动**调 LLM 决定要用的工具。
→ `create_tool_node_with_fallback` = 出错时给 LLM 返回错误信息，让它重试。

---

## 9. 常见概念速查表

| 概念 | 含义 | 备注 |
| --- | --- | --- |
| `StateGraph` | 用来"画图"的类 | 比喻：画板 |
| `State` | TypedDict，定义节点间共享数据 | 比喻：白板 |
| `Node` | 一段 Python 函数 | 比喻：工序 |
| `Edge` | 节点之间的箭头 | 比喻：流水线方向 |
| `add_node(name, fn)` | 注册一个节点 | `name` 必须在图里唯一 |
| `add_edge("A", "B")` | A → B（固定） | 不可跳过 |
| `add_conditional_edges(...)` | A → ?（看返回值） | 路由函数返回字符串 |
| `START` / `END` | 图的起点 / 终点 | 特殊常量 |
| `compile()` | 把 builder 变成可执行的图 | 必须调 |
| `checkpointer` | "暂停现场"的存哪 | `MemorySaver` = 内存 |
| `interrupt_before` | 在哪个节点前暂停 | 列表 |
| `tool_calls` | LLM 决定要调的工具 | 在 AI Message 上 |
| `AIMessage` / `HumanMessage` / `ToolMessage` | 三种消息 | LangChain 自带 |
| `add_messages` | reducer：追加消息而不是覆盖 | 配合 `Annotated[list, add_messages]` |
| `ToolNode` | LangGraph 自带的"自动调工具"节点 | 一行代码 |
| `bind_tools([...])` | 把工具列表"绑"给 LLM | LLM 才知道自己能调什么 |

---

## 🎯 总结

| 你想做什么 | 你要写什么 |
| --- | --- |
| 定义节点间传什么 | `class State(TypedDict)` |
| 一个节点做什么 | 写个 `def my_node(state): return {...}` |
| 节点 A 之后接什么 | `add_edge("A", "B")` 或 `add_conditional_edges("A", fn, ...)` |
| 把图跑起来 | `graph = builder.compile()` |
| 让 AI 在改数据前停下来等用户 | `compile(interrupt_before=[...])` |
| 让 LLM 自己决定调哪些工具 | `llm.bind_tools([...])` + 工具节点 |

**一句话总结**：
> LangGraph = **State（白板） + Node（工序） + Edge（流水线方向）**
> 你只需要定义这三样，框架会**自动**帮你跑起来。

---

## 📌 附录：常见问题答疑

### Q1：`builder = StateGraph(State)` 是不是"注册 State 的节点"？

**答：不是。** `StateGraph(State)` 的意思是告诉 LangGraph：

> "我这张图里要传的数据，长 `State` 那个样子。"

| 代码 | 作用 | 比喻 |
| --- | --- | --- |
| `class State(TypedDict)` | **定义**数据长什么样 | 设计"快递包裹"里能放什么 |
| `builder = StateGraph(State)` | **告诉框架**：以后按 State 这个形状传数据 | 给工厂说"我要做这种快递" |
| `builder.add_node("X", fn)` | 注册一个**节点** | 工厂里加一个工人 |
| `builder.add_edge(...)` | 注册一条**流水线方向** | 给工人画工序箭头 |
| `builder.compile()` | 把 builder 变成可执行的 graph | 把图纸落地成真正能跑的车间 |

所以"为什么 `builder` 写法和 `add_node` 不一样"——**它们干的不是同一件事**：

```python
# 1) 先跟 LangGraph 声明："我的图里要传这些字段"
builder = StateGraph(State)

# 2) 然后往图上加节点、加边
builder.add_node(...)
builder.add_edge(...)
```

---

### Q2：什么叫"让图动起来"？

**答：** 你写好图纸之后，**还没有真在跑**——它只是"一堆对象在内存里"。
"让它动起来" = 把图纸**编译**成"可执行程序"。

```python
graph = builder.compile()
# ↑ 这一步 = 把 builder 变成 graph
#   之后 graph 才能"接收请求、按照你画的图去执行"
```

举个生活类比：

- `builder` = 工厂的**设计图纸**
- `builder.compile()` = 按图纸把**真实工厂**建出来
- `graph.invoke(...)` = 工厂接到订单，**开始生产**

所以这句话的"动" = "可以接收用户输入、按图执行了"。

`compile()` 还能加参数（这就是你后面会看到的"敏感操作前停下"）：

```python
graph = builder.compile(
    checkpointer=memory,           # 内存检查点：保存"暂停现场"
    interrupt_before=[             # 哪些节点前强制停下
        "update_flight_sensitive_tools"
    ]
)
```

---

### Q3：`state["messages"] + [{"role": "user", "content": "讲个笑话"}]` 是什么意思？

**一句话：把"用户最新的话"塞进对话历史，然后让 AI 接着聊。**

#### ① `state["messages"]` 是什么？

`state` 是当前图里**所有节点共享的那张白板**。`state["messages"]` 就是白板上的"聊天记录"——之前所有节点产生的对话：

```python
state = {
    "messages": [
        HumanMessage("你好"),       # 之前用户说的
        AIMessage("你好呀！"),       # 之前 AI 回的
        ...
    ]
}
```

#### ② `+ [...]` 是什么意思？

把"**用户最新的要求**"追加到聊天记录后面：

```python
追加的 = {"role": "user", "content": "讲个笑话"}
# 格式来自 OpenAI 的 Chat Completions API：
#   role=user   -> "这是用户说的话"
#   content=... -> "说了什么"
```

`+` 在 Python 列表里就是"把两个列表拼起来"。

#### ③ 合起来给 LLM 干啥？

`llm.invoke(全部聊天历史 + 最新指令)` —— **让 AI 看着之前所有对话 + 用户最新一句话，给出回复**：

```python
完整传过去的 messages = [
    HumanMessage("你好"),                                # 之前
    AIMessage("你好呀！"),                                # 之前
    {"role": "user", "content": "讲个笑话"}              # ← 刚追加的
]
# LLM 看完整段历史后，生成一条新回复 AIMessage("为什么程序员...")
```

**为什么这么麻烦**？因为 LangGraph 的 messages 是个 reducer（`Annotated[list, add_messages]`），节点**只能返回"新增"**——不能直接 `state["messages"] = [...]` 覆盖。
所以模式是固定的：

```python
def my_node(state):
    # 1) 读：拿出"当前所有历史"
    history = state["messages"]
    # 2) 追加：把用户新指令接上
    new_input = history + [{"role": "user", "content": "..."}]
    # 3) 调 LLM
    response = llm.invoke(new_input)
    # 4) 返回"新增的消息"（add_messages 会自动把它加到 state）
    return {"messages": [response]}
```

---

### Q4：这个"笑话+翻译"案例是不是固定边？

**是的——你理解得 100% 正确！** 完整流程：

```
1. 定义 State 类              → "这张图要传什么数据"
2. builder = StateGraph(State) → "画图板准备好"
3. builder.add_node(...)      → "在图上画两个工序"
4. builder.add_edge(...)      → "画箭头：工序顺序是固定的"
5. builder.compile()          → "把图纸变成能跑的工厂"
6. graph.invoke(...)          → "开始干活"
```

加上更直观的图：

```
         你的代码                           实际意义
         ──────                           ──────
   ┌─────────────────┐
   │ class State:      │   定义"快递包裹"长什么样：
   │   messages: list  │   里面能放聊天记录
   │   user_info: str  │   能放用户信息
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │ StateGraph(State) │   声明"按上面那种包裹传数据"
   │   = 准备画图板   │
   └────────┬────────┘
            │
            ├─ add_node("joke", tell_joke)        ← 注册工人 A
            │
            ├─ add_node("translate", translate)   ← 注册工人 B
            │
            ├─ add_edge(START, "joke")            ← 入口 → A
            │
            ├─ add_edge("joke", "translate")     ← A → B（固定！）
            │
            └─ add_edge("translate", END)        ← B → 出口
                                                ▲
                                  重点：这条箭头是"固定边"，
                                       走完 joke 一定去 translate
```

**和"条件边"对比一下**：

```python
# ✅ 这个例子（固定边）：joke 跑完一定去 translate
builder.add_edge("joke", "translate")

# ✅ 如果要"条件边"：AI 觉得好笑才翻译，不好笑就重讲
def router(state):
    if "好笑" in state["messages"][-1].content:
        return "translate"
    else:
        return "joke"   # 重讲

builder.add_conditional_edges(
    "joke",                                       # 起点
    router,                                       # 路由函数
    {"translate": "translate", "joke": "joke"}   # 路由函数返回值 -> 目标
)
```

**"笑话+翻译"这个例子里没用到条件边**，所以 3 条边全是 `add_edge` 写的固定边，跟你理解的一模一样。✅

---

## 🎯 一句话终极总结

> LangGraph = **State（白板） + Node（工序） + Edge（流水线方向）**：
> - `StateGraph(State)` = 声明"我这张图要传什么数据"
> - `add_node` 才是注册节点
> - `compile()` = 让图"动起来"
> - `state["messages"] + [...]` = 拿"已有聊天"+"用户新指令"一起喂给 LLM
> - `add_edge` = 固定箭头，`add_conditional_edges` = 条件箭头（按路由函数返回值选路）