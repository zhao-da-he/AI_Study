# LangGraph 的多智能体方案 — 疑惑解答（按流程顺序）

> 把你之前在 [docs/readme.md](../readme.md) 和 [docs/ReAct 循环、Supervisor 多智能体流程、Command.PARENT、Handoff工具、InjectedState 和 InjectedToolCallId工具.md](../ReAct%20循环、Supervisor%20多智能体流程、Command.PARENT、Handoff工具、InjectedState%20和%20InjectedToolCallId工具.md) 里问过的问题，**按"图执行一遍"的先后顺序**重新排版。
> 这样从"图启动 → 用户输入 → 调度 → 工具 → 中断 → 恢复"一条线看下来更顺。

---

## 📌 完整对话流程时序图（先看这张图理解整体）

```
[图启动]
   ↓
fetch_user_info 节点  ← Q1 背景预加载
   ↓
supervisor 节点      ← Q2 主管是什么
   ↓ 调工具：assign_to_xxx_agent  ← Q3 Handoff 工具
   ↓        ↑
   ↓        │ LangGraph 自动注入 state + tool_call_id  ← Q4 Injected 参数
   ↓
[子 agent 节点]
   ↓ 调业务工具（如 search_hotels）  ← Q5 SearchArgs 是什么
   ↓
★ 触发 interrupt("批准 y?")  ← Q6 interrupt 是什么
   ↓
[图暂停]
   ↓
外层 execute_graph 用 4 件套恢复  ← Q7 interrupt 需要 4 件套
   ↓
子 agent 干完 → Command(PARENT) 跳回 supervisor  ← Q8 Command.PARENT
   ↓
[supervisor 综合答案]
   ↓
END
```

**按这个顺序看下面 8 个问题最清晰。**

---

## 📌 目录

- [Q1：fetch_user_info 是什么](#q1fetch_user_info-是什么)
- [Q2：Supervisor 多智能体](#q2supervisor-多智能体)
- [Q3：Handoff 工具是什么](#q3handoff-工具是什么)
- [Q4：InjectedState 和 InjectedToolCallId](#q4injectedstate-和-injectedtoolcallid)
- [Q5：SearchArgs 是什么](#q5searchargs-是什么)
- [Q6：interrupt 是什么](#q6interrupt-是什么)
- [Q7：interrupt 需要 4 件套搭配使用](#q7interrupt-需要-4-件套搭配使用)
- [Q8：Command.PARENT](#q8commandparent)
- [完整数据流与项目文件结构](#完整数据流与项目文件结构)
- [一句话总结](#一句话总结)

---

## Q1：fetch_user_info 是什么

> **引用的代码**：[readme.md](../readme.md) 第 4 大核心特性 = "fetch_user_info 第一站"（第 46 行）

### 1.1 一句话

> **`fetch_user_info` = 图启动时的预加载**——
> 1. 是图的**第一个节点**（`START` 之后）
> 2. **用户输入之前**就主动查用户航班
> 3. 把结果**写进 messages**，给所有 Agent 看
> 4. **只查一次**（用 `id='user_info_success'` 去重）

### 1.2 完整时序

```
图启动
   ↓
[fetch_user_info] 节点
   ↓
def get_user_info(state, config):
   ↓ 第一次：state.messages 空，跳过缓存
   ↓
flight_data = fetch_user_flight_information(config)   # ← 真的查 DB
   ↓
format_flight_info(flight_data)   # ← 把字典变中文文本
   ↓
return {"messages": [flight_message]}   # ← partial update
```

### 1.3 核心价值

| 价值 | 体现 |
| --- | --- |
| **上下文增强** | 所有 Agent 都能看到用户背景，不用每次问 |
| **去重** | 同一会话只查一次，不浪费 SQL |
| **错误处理** | 查不到时返回"未找到您的航班信息" |
| **统一消息格式** | 后续 Agent 看到的是结构化的 AIMessage |

### 1.4 在 graph.py 里的注册

```python
graph = (
    StateGraph(MessagesState)
    .add_node('fetch_user_info', get_user_info)        # ← 注册这个函数
    .add_node(supervisor_agent, destinations=(...))
    ...
    .add_edge(START, 'fetch_user_info')              # ← 图入口→ 查用户
    .add_edge('fetch_user_info', 'supervisor')        # ← 查完→ 去主管
    .compile(checkpointer=memory)
)
```

**完整代码讲解**：[readme.md](../readme.md) **§ 4.2 [graph_chat/fetch_user_info_node.py](../graph_chat/fetch_user_info_node.py)**。

---

## Q2：Supervisor 多智能体

### 2.1 一句话

> **Supervisor = "总台经理"**——1 个主管 agent + 5 个专业子 agent；主管**不干活**，只听用户说什么，然后调 Handoff 工具"派人"。

### 2.2 架构

```
主图 (supervisor 在这)
   │
   ├── research_agent            (网搜)
   ├── flight_booking_agent      (航班)
   ├── hotel_booking_agent       (酒店)
   ├── car_rental_booking_agent  (租车)
   └── excursion_booking_agent   (旅行)
```

### 2.3 6 个 Agent 速览

| Agent | 工具 | 作用 |
| --- | --- | --- |
| `research_agent` | `MySearchTool` | 联网搜索 |
| `flight_booking_agent` | `search_flights`, `lookup_policy`, `update_ticket_to_new_flight`, `cancel_ticket` | 航班查/改签/退票 |
| `hotel_booking_agent` | `search_hotels`, `book_hotel`, `update_hotel`, `cancel_hotel` | 酒店 |
| `car_rental_booking_agent` | `search_car_rentals`, `book_car_rental`, `update_car_rental`, `cancel_car_rental` | 租车 |
| `excursion_booking_agent` | `search_trip_recommendations`, `book_excursion`, `update_excursion`, `cancel_excursion` | 旅行 |
| `supervisor_agent` | **5 个 Handoff 工具** | 调度员 |

### 2.4 supervisor 4 条铁律

```python
prompt=(
    "你是一个监督者或者管理者，管理五个智能体：\n"
    "- 网络搜索智能体：分配与网络搜索、数据查询相关的任务\n"
    "- 航班预订能体：分配与航班查询，预定，改签等相关的任务\n"
    ...
    "处理规则：\n"
    "1. 如果问题属于以下类别，直接回答：\n"
    "2. 其他情况按类型分配给对应智能体。\n"
    "3. 一次只分配一个任务给一个智能体。\n"
    "4. 不要自己执行需要工具的任务。\n"
)
```

**完整代码**：[readme.md](../readme.md) **§ 4.1.1-4.1.6**。

---

## Q3：Handoff 工具是什么

> **引用的代码**：[readme.md](../readme.md) 第 90 行 — "all_agent.py # ★ 6 个 Agent + 5 个 Handoff 工具"

### 3.1 一句话

> **Handoff 工具 = "转交任务"的工具**——supervisor 把活儿"交给"子 agent 的方式——**不是直接调函数**，而是**给 supervisor 一个"能调的工具"**。

### 3.2 工厂函数

打开 [graph_chat/all_agent.py](../graph_chat/all_agent.py)：

```python
def create_handoff_tool(*, agent_name: str, description: str | None = None):
    name = f"transfer_to_{agent_name}"
    description = description or f"Ask {agent_name} for help."

    @tool(name, description=description)
    def handoff_tool(
            state: Annotated[MessagesState, InjectedState],
            tool_call_id: Annotated[str, InjectedToolCallId],
    ) -> Command:
        tool_message = {
            "role": "tool",
            "content": f"Successfully transferred to {agent_name}",
            "name": name,
            "tool_call_id": tool_call_id,
        }
        return Command(
            goto=agent_name,
            update={**state, "messages": state["messages"] + [tool_message]},
            graph=Command.PARENT,
        )

    return handoff_tool
```

### 3.3 5 个 Handoff 实例

```python
assign_to_research_agent = create_handoff_tool(agent_name="research_agent", ...)
assign_to_flight_booking_agent = create_handoff_tool(agent_name="flight_booking_agent", ...)
assign_to_hotel_booking_agent = create_handoff_tool(agent_name="hotel_booking_agent", ...)
assign_to_car_rental_booking_agent = create_handoff_tool(agent_name="car_rental_booking_agent", ...)
assign_to_excursion_booking_agent = create_handoff_tool(agent_name="excursion_booking_agent", ...)
```

### 3.4 Handoff vs 普通工具

| 维度 | 普通工具（如 search_hotels） | Handoff 工具 |
| --- | --- | --- |
| **干啥** | 真的查数据/做操作 | **不做事**——只是"跳走" |
| **结果** | 返回数据 | 返回 `Command`（跳到别处） |
| **谁来调** | supervisor 和 5 个子 agent 都会调 | **只给 supervisor 调** |

### 3.5 Handoff vs add_edge

| 维度 | `add_edge("A", "B")` | Handoff 工具 |
| --- | --- | --- |
| **时机** | 编译时就定好 | **运行时**由 LLM 决定 |
| **灵活性** | 死的 | 活的 |

**完整讲解**：[ReAct...InjectedToolCallId 工具.md § 4 Handoff 工具](#4-handoff-工具)。

---

## Q4：InjectedState 和 InjectedToolCallId

> **引用的代码**：[readme.md](../readme.md) 第 90 行的 `all_agent.py` — 内部 `handoff_tool` 函数签名

### 4.1 一句话

> **`InjectedState` + `InjectedToolCallId` = LangGraph 自动注入的"隐藏参数"**——LLM **不用传**，框架**自动塞进来**。

### 4.2 对比

```python
# 普通工具参数：LLM 必须传
@tool
def search_hotels(location: str) -> list:
    """location 由 LLM 传"""
    ...

# 注入参数：LLM 不用传
@tool
def handoff_tool(
    state: Annotated[MessagesState, InjectedState],     # ← 框架自动注入
    tool_call_id: Annotated[str, InjectedToolCallId],    # ← 框架自动注入
):
    ...
```

### 4.3 InjectedState 详解

```python
@tool
def my_tool(state: Annotated[MessagesState, InjectedState]):
    messages = state["messages"]    # ← 直接读当前所有消息
    return ...
```

**项目里的用法**（handoff_tool 内部）：

```python
return Command(
    goto=agent_name,
    update={**state, "messages": state["messages"] + [tool_message]},  # ← 用 state 追加消息
    graph=Command.PARENT,
)
```

### 4.4 InjectedToolCallId 详解

```python
@tool
def my_tool(tool_call_id: Annotated[str, InjectedToolCallId]):
    return ToolMessage(
        content="工具的结果",
        tool_call_id=tool_call_id,    # ← 用这个 ID 回执
    )
```

**项目里的用法**：

```python
tool_message = {
    "role": "tool",
    "content": f"Successfully transferred to {agent_name}",
    "name": name,
    "tool_call_id": tool_call_id,    # ← ★ 用框架注入的 ID
}
```

### 4.5 Annotated 是什么？

```python
def my_tool(
    state: Annotated[MessagesState, InjectedState],     # ← 双重标注
):
    pass
```

| 部分 | 作用 |
| --- | --- |
| `MessagesState` | 类型注解（IDE 能提示） |
| `InjectedState` | **元数据标记**——告诉 LangGraph "这个参数由你注入" |

**完整讲解**：[ReAct...InjectedToolCallId 工具.md § 5 InjectedState 和 InjectedToolCallId](#5-injectedstate-和-injectedtoolcallid)。

---

## Q5：SearchArgs 是什么

> **引用的代码**：[readme.md](../readme.md) 第 904-905 行 — `tools/search_tool.py` 里的 `class SearchArgs(BaseModel)`

### 5.1 一句话

> **`SearchArgs` = "工具的参数说明书"**——告诉 LLM "这个工具**接受什么参数**、**每个参数是什么意思**"。

### 5.2 长什么样

```python
from pydantic import BaseModel, Field

class SearchArgs(BaseModel):
    query: str = Field(description="需要进行网络搜索的信息。")
```

| 部分 | 含义 |
| --- | --- |
| `BaseModel` | 继承 Pydantic 基类——这是"参数模型" |
| `query: str` | **参数名 + 类型**——LLM 调工具时要传 `query="北京天气"` |
| `Field(description=...)` | 描述——告诉 LLM 这个参数是干嘛的 |

### 5.3 为什么要单独定义 `SearchArgs`？

```python
class MySearchTool(BaseTool):
    name: str = "search_tool"
    description: str = '搜索互联网上公开内容的工具'
    return_direct: bool = False
    args_schema: Type[BaseModel] = SearchArgs       # ← ★ 这里要用 Pydantic 类
```

**关键属性 `args_schema: Type[BaseModel] = SearchArgs`** 告诉 LangChain：

- 这个工具接受**什么参数**（类名）
- 每个参数**叫什么、什么类型、什么含义**（类里的字段）

### 5.4 完整时序

```
LLM 收到工具描述（自动生成）
   ↓
"工具: search_tool
  描述: 搜索互联网上公开内容的工具
  参数:
    - query (string): 需要进行网络搜索的信息。   ← ★ 来自 SearchArgs.query
"
   ↓
LLM 决定调工具
   ↓
输出（function call）：
{
  "name": "search_tool",
  "arguments": {
    "query": "北京天气怎么样"      ← ★ 符合 SearchArgs 的 schema
  }
}
   ↓
LangChain 自动：
   1) 解析 arguments → 创建 SearchArgs(query="北京天气怎么样") 实例
   2) 自动校验：query 必须是 str ✓
   3) 调 MySearchTool._run(query="北京天气怎么样")
```

### 5.5 3 种工具写法对比

| 写法 | 怎么传参 | 用 `SearchArgs` 吗 |
| --- | --- | --- |
| **`@tool` 装饰器** | 写函数参数 | ❌ 不需要 |
| **`BaseTool` 子类**（本项目） | 写 `args_schema = SearchArgs` | ✅ **必须** |
| **`StructuredTool.from_function()`** | 显式传 `args_schema=...` | ✅ **必须** |

---

## Q6：interrupt 是什么

> **引用的代码**：[readme.md](../readme.md) 第 504 行 — `current_state.interrupts[0].value`

### 6.1 一句话

> **`interrupt` = "AI 问你'行不行'，等你回答"**——
> 在 LangGraph 工具函数里调 `interrupt("问题")` → **图暂停**，等用户输入 `Command(resume=...)` 恢复。

### 6.2 `interrupt` 是什么？

```python
from langgraph.types import interrupt

@tool
def my_sensitive_tool():
    user_response = interrupt("你确认要执行吗？")
    if user_response == "y":
        return "已执行"
    return "已取消"
```

**核心**：`interrupt` 抛出一个**特殊异常**（LangGraph 内部机制），把控制权**还给外层 `graph.stream(...)`**。

### 6.3 interrupt vs raise Exception

| 维度 | `raise Exception` | `interrupt(...)` |
| --- | --- | --- |
| **目的** | 报错，终止流程 | 暂停，**等用户回答** |
| **外部如何恢复** | catch 异常 | `Command(resume={...})` |

### 6.4 interrupt 的完整工作流

```
工具函数被调用
   ↓
代码跑到 interrupt("批准 y?") 这一行
   ↓
★ ★ ★ LangGraph 暂停当前节点 ★ ★ ★
   ↓
把 "批准 y?" 写到 state.interrupts[0].value
   ↓
控制权返回到 graph.stream() 这一层
   ↓
graph.stream() 看到"有 interrupt" → 不再往后跑
   ↓
外层代码：
   current_state = graph.get_state(config)
   result = current_state.interrupts[0].value  # 拿到 "批准 y?"
```

**关键**：`interrupt(...)` 这个调用**返回一个值**——就是用户**回复的内容**。

### 6.5 项目里的实际用法

打开 [tools/search_tool.py](../tools/search_tool.py)：

```python
class MySearchTool(BaseTool):
    def _run(self, query) -> str:
        # ★ 关键：在执行网络搜索前"人工确认"
        print('AI大模型尝试调用工具 `search_tool`来完成数据搜索')
        response = interrupt(
            f"AI大模型尝试调用工具 `search_tool`来完成数据搜索，\n"
            "请审核并选择：批准（y）或直接给我工具执行的答案。"
        )
        if response["answer"] == "y":
            pass  # 同意 → 继续
        else:
            return f"人工终止了该工具的调用，给出的理由或者答案是:{response['answer']}"
```

---

## Q7：interrupt 需要 4 件套搭配使用

### 7.1 一句话

> **是的**——`interrupt` 是"半成品"，必须配合 **`Command(resume=...)` + `get_state` + `stream()` 循环** 4 件套才能工作。少一个都不行。

### 7.2 4 件套缺一不可

| # | 谁负责 | 做什么 | 不配合的后果 |
| --- | --- | --- | --- |
| ① | **工具函数里** | `response = interrupt("问题")` | ❌ 啥也不发生（甚至报错） |
| ② | **外层代码** | `graph.stream(Command(resume={'answer': user_input}), config)` | ❌ 图继续跑，不会停 |
| ③ | **外层代码** | `current_state = graph.get_state(config)` | ❌ 不知道"有没有中断" |
| ④ | **外层代码** | `if current_state.next: result = current_state.interrupts[0].value` | ❌ 拿不到"问题内容" |

### 7.3 完整时序

```
[第 1 轮：用户说"订北京酒店"]
   ↓
execute_graph("订北京酒店")
   ↓
graph.stream({"messages": ("user", "订北京酒店")}, config)   ← ① 流式跑
   ↓
[fetch_user_info] → [supervisor] → [hotel_booking_agent]
   ↓
search_hotels 工具里跑到 interrupt("批准 y?")  ← ② ★ 暂停
   ↓
graph.stream() 检测到 interrupt → 停止往下跑
   ↓
execute_graph：
   current_state = graph.get_state(config)    ← ③ 查图状态
   if current_state.next:                  ← ④ 判断有没有中断
       result = current_state.interrupts[0].value   ← ⑤ 拿问题
   ↓
print(result)  # "批准 y?"
   ↓
用户输入 "y"
   ↓
[第 2 轮]
execute_graph("y")
   ↓
graph.stream(Command(resume={'answer': 'y'}), config)  ← ⑥ 续跑 + 传答案
   ↓
search_hotels 从 interrupt() 那一行继续
   ↓
user_input = 'y'  ← interrupt() 返回 'y'
   ↓
工具继续执行
```

### 7.4 最小可运行示例

```python
# ========== 工具层 ==========
from langgraph.types import interrupt
from langchain_core.tools import tool

@tool
def my_sensitive_tool():
    user_input = interrupt("你确认吗？")
    return f"用户说：{user_input}"


# ========== 外层（缺一不可）==========
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.types import Command

# 1) 注册节点
graph = StateGraph(MessagesState)
graph.add_node("my_tool", my_sensitive_tool)
graph.add_edge(START, "my_tool")
graph.add_edge("my_tool", END)
graph = graph.compile()

# 2) 跑图
def run():
    while True:
        user_input = input("你：")
        
        # ★ ③ 先看 state
        state = graph.get_state(config)
        
        if state.next:                          # ★ ④ 在中断
            # ★ ⑥ 续跑 + 传用户输入
            graph.stream(Command(resume={'answer': user_input}), config)
        else:
            # ② 正常开新一轮
            graph.stream({"messages": ("user", user_input)}, config)
        
        # ★ ⑤ 跑完看有没有新中断
        state = graph.get_state(config)
        if state.next:
            question = state.interrupts[0].value
            print(f"AI 问你：{question}")
```

---

## Q8：Command.PARENT

> **引用的代码**：[readme.md](../readme.md) 第 944-952 行 — `create_handoff_tool` 返回的 `Command(goto=..., graph=Command.PARENT, ...)`、第 1190-1191 行 `Command.PARENT 跳回 supervisor`

### 8.1 一句话

> **`Command.PARENT` = "跳出去，回到爸爸那"**——
> 当子节点（子代理）干完活，用 `Command(goto=..., graph=Command.PARENT)` 告诉 LangGraph "**我干完了，跳回父图**"。

### 8.2 Command 长什么样？

```python
from langgraph.types import Command

Command(
    goto="下一个节点名",         # 让图跳到某节点
    update={"字段": "值"},        # 顺便修改 state
    graph=Command.PARENT,        # ★ 跳到父图
)
```

### 8.3 `graph` 参数的 2 个值

| 值 | 含义 | 何时用 |
| --- | --- | --- |
| `Command.LOCAL`（默认） | **当前子图**内部跳转 | 子 agent 想叫子 agent 时 |
| `Command.PARENT` | **父图**（上一层） | 子 agent 想"回主管"时 |

### 8.4 为什么需要 `Command.PARENT`？

**默认行为**：子节点返回的 `Command(goto="supervisor")` 是**在当前子图里跳**——但 supervisor 不在子图里，会报 `KeyError`。

**解决**：

```python
return Command(
    goto="supervisor",
    update={...},
    graph=Command.PARENT,    # ← 跳回父图
)
```

### 8.5 项目里的实际用法

```python
@tool(name, description=description)
def handoff_tool(
        state: Annotated[MessagesState, InjectedState],
        tool_call_id: Annotated[str, InjectedToolCallId],
) -> Command:
    """执行实际的转接操作。"""
    tool_message = {
        "role": "tool",
        "content": f"Successfully transferred to {agent_name}",
        "name": name,
        "tool_call_id": tool_call_id,
    }
    return Command(
        goto=agent_name,                  # ← 目标 agent 名
        update={**state, "messages": state["messages"] + [tool_message]},
        graph=Command.PARENT,              # ← ★ 跳回父图
    )
```

### 8.6 完整跳转时序

```
[supervisor]
   ↓
   调 handoff_tool → Command(goto="hotel_booking_agent", graph=PARENT)
   ↓
[hotel_booking_agent]
   ↓ 干活 → 处理完
   ↓
   返回 Command(goto="supervisor", graph=PARENT)
   ↓
★ ★ ★ PARENT 跳回 supervisor ★ ★ ★
   ↓
[supervisor]
   ↓ 继续 ReAct 循环
```

### 8.7 完整讲解

[ReAct...InjectedToolCallId 工具.md § 3 Command.PARENT](#3-commandparent)

---

## 完整数据流与项目文件结构

### 8 大概念怎么"串"起来

```
用户输入 "我要改签机票并订酒店"
   ↓
[fetch_user_info]（预加载背景）        ← Q1
   ↓
[supervisor] 调 handoff_tool（用 InjectedState 读消息）  ← Q2, Q3, Q4
   ↓ 跳到 flight_booking_agent（Command.PARENT 配合）  ← Q8
   ↓
[flight_booking_agent] ReAct 循环干活
   ↓ 完成 → Command(PARENT) 跳回 supervisor
   ↓
[supervisor] 又调 handoff_tool
   ↓ 跳到 hotel_booking_agent
   ↓
[hotel_booking_agent] 工具里调 interrupt("批准 y?")  ← Q6
   ↓ ★ 图暂停
   ↓
外层 execute_graph 用 4 件套恢复  ← Q7
   ↓ 用户输入 y
   ↓
[hotel_booking_agent] 续跑 → 完成 → Command(PARENT) 回 supervisor
   ↓
[supervisor] 综合答案 → END
```

### 8 个问题与文件位置的对照

| 问题 | 主要文件 | 重点章节 |
| --- | --- | --- |
| Q1 fetch_user_info | [graph_chat/fetch_user_info_node.py](../graph_chat/fetch_user_info_node.py) | [readme.md § 4.2](../readme.md) |
| Q2 Supervisor | [graph_chat/all_agent.py](../graph_chat/all_agent.py) | [readme.md § 4.1.6](../readme.md) |
| Q3 Handoff 工具 | [graph_chat/all_agent.py](../graph_chat/all_agent.py) | [ReAct... § 4](#4-handoff-工具) |
| Q4 InjectedState/ToolCallId | [graph_chat/all_agent.py](../graph_chat/all_agent.py) | [ReAct... § 5](#5-injectedstate-和-injectedtoolcallid) |
| Q5 SearchArgs | [tools/search_tool.py](../tools/search_tool.py) | [readme.md § 4.8.5](../readme.md) |
| Q6 interrupt | [tools/search_tool.py](../tools/search_tool.py) | [ReAct... § 8](#8-interrupt-是什么有什么作用) |
| Q7 interrupt 4 件套 | [graph_chat/graph.py](../graph_chat/graph.py) | [ReAct... § 9](#9-interrupt-需要-4-件套搭配使用) |
| Q8 Command.PARENT | [graph_chat/all_agent.py](../graph_chat/all_agent.py) | [ReAct... § 3](#3-commandparent) |

### 项目目录结构（按流程顺序）

```
new_ctrip/
├── graph_chat/
│   ├── all_agent.py                    ← Q2, Q3, Q4, Q8（核心）
│   ├── fetch_user_info_node.py         ← Q1
│   ├── graph.py                         ← Q7（execute_graph）
│   ├── my_llm.py
│   ├── env_utils.py
│   ├── draw_png.py
│   └── my_print.py
├── tools/
│   ├── search_tool.py                  ← Q5, Q6（interrupt 实际应用）
│   ├── hotels_tools.py
│   ├── flights_tools.py
│   ├── car_tools.py
│   ├── trip_tools.py
│   ├── retriever_vector.py
│   ├── location_trans.py
│   ├── init_db.py
│   └── __init__.py
├── order_faq.md
├── travel_new.sqlite
└── docs/
    └── ReAct 循环、Supervisor 多智能体流程、Command.PARENT、Handoff工具、InjectedState 和 InjectedToolCallId工具.md
    └── LangGraph的多智能体方案疑惑解答.md   ← 你正在看的
```

---

## 一句话总结

> **8 个问题按图执行的先后顺序**：
>
> 1. **fetch_user_info**（Q1）→ 预加载用户航班背景
> 2. **Supervisor**（Q2）→ 主管开始 ReAct 循环
> 3. **Handoff 工具**（Q3）→ supervisor 用来"派人"
> 4. **InjectedState/ToolCallId**（Q4）→ 工具读 state、写回执
> 5. **SearchArgs**（Q5）→ 工具的"参数说明书"
> 6. **interrupt**（Q6）→ 敏感工具前"暂停问用户"
> 7. **4 件套搭配**（Q7）→ interrupt 必须的 4 个机制
> 8. **Command.PARENT**（Q8）→ 子 agent 干完活"回家"的钥匙
>
> **记忆口诀**：
> - **fetch_user_info** = "图启动时预加载"
> - **Supervisor** = "主管 + 5 个子"
> - **Handoff** = "派人"
> - **Injected** = "LangGraph 自动塞的"
> - **Command.PARENT** = "跳出去，回爸爸那"
> - **interrupt** 是嘴，`Command(resume=)` 是耳朵——少一边都聋
> - **SearchArgs** = "工具的参数说明书"