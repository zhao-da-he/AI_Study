# ReAct 循环 / Supervisor 多智能体流程 / Command.PARENT / Handoff工具 / InjectedState 和 InjectedToolCallId 工具

> 把本项目里 `new_ctrip` 用到的 **7 个核心概念**完整讲透：
> 1. **ReAct 循环**——Agent 的"想→做→看"循环
> 2. **Supervisor 多智能体**——主管 + 5 个子代理的协作
> 3. **Command.PARENT**——子 agent "回家"的钥匙
> 4. **Handoff 工具**——主管的"转交任务"工具
> 5. **InjectedState / InjectedToolCallId**——LangGraph 自动注入的"隐藏参数"
> 6. **interrupt**——"AI 问你'行不行'"——敏感操作前的人工确认（必须 4 件套搭配）
> 7. **4 件套搭配**——`interrupt` + `graph.stream` + `get_state` + `Command(resume=...)`

---

## 📌 目录

- [1. ReAct 循环](#1-react-循环)
- [2. Supervisor 多智能体](#2-supervisor-多智能体)
- [3. Command.PARENT](#3-commandparent)
- [4. Handoff 工具](#4-handoff-工具)
- [5. InjectedState 和 InjectedToolCallId](#5-injectedstate-和-injectedtoolcallid)
- [6. 7 个概念怎么配合工作](#6-7-个概念怎么配合工作)
- [7. 一句话总结](#9-一句话总结)
- [8. interrupt 是什么，有什么作用](#8-interrupt-是什么有什么作用)
- [9. interrupt 需要 4 件套搭配使用](#9-interrupt-需要-4-件套搭配使用)
- [10. 全文档总结](#10-全文档总结)

---

## 1. ReAct 循环

### 1.1 一句话

> **ReAct 循环 = "推理 + 行动"自动循环**——
> LLM 不是一次给你答案，而是**反复循环**："思考 → 调用工具 → 看到结果 → 再思考 → …"直到任务完成。

### 1.2 单词拆解

| 字母 | 单词 | 含义 |
| --- | --- | --- |
| **Re** | Reason | 推理（"我应该做什么？"） |
| **Act** | Act | 行动（"调这个工具"） |

### 1.3 完整时序图（以"查北京天气"为例）

```
LLM 收到 "查北京天气"
   ↓
[第 1 轮] 思考(Reason)
   "用户想知道北京天气。我有 search_weather 工具。"
   ↓
   行动(Act)
   "调 search_weather(city='北京')"
   ↓
   工具返回："北京：晴，25度"
   ↓
[第 2 轮] 思考(Reason)
   "我已经拿到结果，可以给用户了。"
   ↓
   行动(Act)
   "不再调工具，直接给用户最终答案"
   ↓
返回："北京今天晴，25度"
```

**循环停止条件**：LLM 决定"我已经有足够信息了，不用再调工具"。

### 1.4 ReAct 循环里到底在循环什么？

| 步骤 | 谁在干 | 干什么 |
| --- | --- | --- |
| ① | LLM | **思考**："我该做什么？" |
| ② | LLM | **决策**：返回"调 X 工具"或"最终答案" |
| ③ | 框架 | **执行**工具调用 |
| ④ | 框架 | 把工具结果**追加**到 messages |
| ⑤ | **回到 ①** | 再让 LLM 想"现在结果够了没" |
| ⑥ | LLM | 直到返回"最终答案"（没工具调用） |
| ⑦ | 框架 | **结束**循环 |

### 1.5 项目里的 5 个子代理都用 ReAct

打开 [graph_chat/all_agent.py](../graph_chat/all_agent.py)：

```python
flight_booking_agent = create_react_agent(
    model=llm,
    tools=update_flight_tools,
    prompt="您是专门处理航班查询，改签政策查询，改签和预定的智能体(Agent)...",
    checkpointer=memory,
    name="flight_booking_agent",
)
```

`create_react_agent` 是 LangGraph 的"**预制 ReAct 智能体**"——ReAct 循环框架**内置**，你不用手写。

**完整流程（用户："我要改签到后天"）**：

```
LLM 看到 "我要改签到后天"
   ↓
[Reason]  "要改签 → 需要查原票 + 查新航班 + 调改签工具"
   ↓
[Act]     search_flights(...)     → 拿到航班列表
   ↓
[Reason]  "看哪些航班合适"
   ↓
[Act]     search_flights(new_date=...)
   ↓
[Reason]  "选 CA1231，更新机票"
   ↓
[Act]     update_ticket_to_new_flight(...) → 改签完成
   ↓
[Reason]  "任务完成"
   ↓
返回 "改签成功"
```

### 1.6 ReAct vs 普通 Function Calling

| 维度 | 普通 Function Calling | ReAct |
| --- | --- | --- |
| **调用次数** | 通常 1 次 | **可能多次**（循环） |
| **决策者** | 程序员写 if-else | **LLM 自己决定**每步调用什么 |
| **适合** | 一次回答就够的任务 | 复杂的多步任务 |
| **上下文** | 简单 | 累积"思考 + 行动 + 观察" |

---

## 2. Supervisor 多智能体

### 2.1 一句话

> **Supervisor = "总台经理"**——
> 1 个主管 agent + 5 个专业子 agent；主管自己**不干活**，只听用户说什么，然后调 Handoff 工具"派人"。

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

打开 [graph_chat/all_agent.py](../graph_chat/all_agent.py)：

| Agent | 工具 | 作用 |
| --- | --- | --- |
| `research_agent` | `MySearchTool` | 联网搜索（智谱 web_search） |
| `flight_booking_agent` | `search_flights`, `lookup_policy`, `update_ticket_to_new_flight`, `cancel_ticket` | 航班查/改签/退票 |
| `hotel_booking_agent` | `search_hotels`, `book_hotel`, `update_hotel`, `cancel_hotel` | 酒店查/订/改/取消 |
| `car_rental_booking_agent` | `search_car_rentals`, `book_car_rental`, `update_car_rental`, `cancel_car_rental` | 租车查/订/改/取消 |
| `excursion_booking_agent` | `search_trip_recommendations`, `book_excursion`, `update_excursion`, `cancel_excursion` | 旅行查/订/改/取消 |
| `supervisor_agent` | **5 个 Handoff 工具** | **调度员**——只分配任务 |

### 2.4 supervisor 的核心代码

```python
supervisor_agent = create_react_agent(
    model=llm,
    tools=[
        assign_to_research_agent,
        assign_to_flight_booking_agent,
        assign_to_hotel_booking_agent,
        assign_to_car_rental_booking_agent,
        assign_to_excursion_booking_agent,
    ],
    prompt=(
        "你是一个监督者或者管理者，管理五个智能体：\n"
        "- 网络搜索智能体：分配与网络搜索、数据查询相关的任务\n"
        "- 航班预订能体：分配与航班查询，预定，改签等相关的任务\n"
        "- 酒店预订智能体：分配与酒店查询，预定，修改订单等相关的任务\n"
        "- 汽车租赁预定智能体：分配与汽车租赁查询，预定，修改订单等相关的任务\n"
        "- 旅行产品预定智能体：分配与旅行推荐查询，预定，修改订单等相关的任务\n"
        "处理规则：\n"
        "1. 如果问题属于以下类别，直接回答：\n"
        "   - 可以根据上下文记录直接回答的内容\n"
        "   - 不需要工具的一般咨询（如'你好'）\n"
        "   - 确认类问题\n"
        "2. 其他情况按类型分配给对应智能体。\n"
        "3. 一次只分配一个任务给一个智能体。\n"
        "4. 不要自己执行需要工具的任务。\n"
    ),
    name="supervisor",
)
```

**4 条铁律**：

| 规则 | 含义 |
| --- | --- |
| **1. 这几类直接答** | 有上下文能答、纯问候、确认类问题——自己处理 |
| **2. 其它按类型分配** | 需要工具的任务 → 委派给对应子 agent |
| **3. 一次只分配一个** | 但 supervisor 的 ReAct 循环可以**连续多次调** |
| **4. 不要自己执行工具** | 主管只"调度"，不"干活" |

### 2.5 Supervisor 怎么决定调哪个 agent？

**完全靠 LLM 自己**——`create_react_agent` + `prompt` 提示词 + `tools` 工具列表 + ReAct 循环。

LLM 看着：
- 用户消息（"我要改签机票并订酒店"）
- 5 个 Handoff 工具的 `description`
- 当前 messages 历史

**LLM 自己决定**：
- 第一轮：调 `transfer_to_flight_booking_agent`
- 第二轮（子 agent 干完回来后）：调 `transfer_to_hotel_booking_agent`
- 第三轮：综合答案 → 最终回复

### 2.6 完整时序图（多 agent 协作）

```
用户："我要改签机票并订北京酒店"
   ↓
[fetch_user_info]（节点 1）
   ↓
[supervisor] — ReAct 循环第 1 轮
   ↓ 思考："改签 → flight_agent"
   ↓ 调 transfer_to_flight_booking_agent
   ↓
[flight_booking_agent]（节点 3）
   ↓ 调 search_flights → 调 update_ticket_to_new_flight → 改签完成
   ↓ Command(goto=PARENT) → 跳回 supervisor
   ↓
[supervisor] — ReAct 循环第 2 轮
   ↓ 思考："继续搞酒店"
   ↓ 调 transfer_to_hotel_booking_agent
   ↓
[hotel_booking_agent]（节点 4）
   ↓ 调 search_hotels → 触发 interrupt（人工确认）
   ↓ 用户输入 "y" → 续跑
   ↓ Command(PARENT) → 跳回 supervisor
   ↓
[supervisor] — ReAct 循环第 3 轮
   ↓ 综合 → 返回最终答案 → END
```

**supervisor 出现 3 次**——每次跳回都是 ReAct 循环的下一轮。

### 2.7 Supervisor 怎么支持多轮对话？

```python
# graph.py
while True:
    user_input = input('用户：')
    res = execute_graph(user_input)        # ← 每次输入触发一次图执行
```

**关键**：`checkpointer=memory` 保留 state——supervisor 在下一轮对话时**仍能看到**之前所有消息。

---

## 3. Command.PARENT

### 3.1 一句话

> **`Command.PARENT` = "跳出去，回到爸爸那"**——
> 子 agent 干完活用它告诉 LangGraph："**我干完了，跳回父图**（supervisor 所在）"。

### 3.2 Command 长什么样？

```python
from langgraph.types import Command

Command(
    goto="下一个节点名",         # 让图跳到某节点
    update={"字段": "值"},        # 顺便修改 state
    graph=Command.PARENT,        # ★ 跳到父图
)
```

### 3.3 `graph` 参数的 2 个值

| 值 | 含义 | 何时用 |
| --- | --- | --- |
| `Command.LOCAL`（默认） | **当前子图**内部跳转 | 子 agent 想叫子 agent 时 |
| `Command.PARENT` | **父图**（上一层） | 子 agent 想"回主管"时 |

### 3.4 为什么需要 `Command.PARENT`？

**默认行为**：子节点返回的 `Command(goto="supervisor")` 是**在当前子图里跳**——但 supervisor 不在子图里，会报 `KeyError`。

**解决**：

```python
return Command(
    goto="supervisor",
    update={...},
    graph=Command.PARENT,    # ← 跳回父图
)
```

### 3.5 项目里的实际用法

打开 [graph_chat/all_agent.py](../graph_chat/all_agent.py)：

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

### 3.6 完整跳转时序

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

---

## 4. Handoff 工具

### 4.1 一句话

> **Handoff 工具 = "转交任务"的工具**——
> supervisor 把活儿"交给"子 agent 的方式——**不是直接调函数**，而是**给 supervisor 一个"能调的工具"**。

### 4.2 为什么需要 Handoff？

| 没用 Handoff | 用 Handoff |
| --- | --- |
| supervisor 写 if-else 判断调谁 | LLM 自己决定调谁 |
| 手工塞 state | 框架自动管理 |
| 只能调一个就结束 | ReAct 循环可连续调多个 |

### 4.3 Handoff 工厂函数

打开 [graph_chat/all_agent.py](../graph_chat/all_agent.py)：

```python
def create_handoff_tool(*, agent_name: str, description: str | None = None):
    """创建一个用于将当前会话转接到指定代理的工具函数。"""
    name = f"transfer_to_{agent_name}"
    description = description or f"Ask {agent_name} for help."

    @tool(name, description=description)
    def handoff_tool(
            state: Annotated[MessagesState, InjectedState],     # ← 自动注入
            tool_call_id: Annotated[str, InjectedToolCallId],    # ← 自动注入
    ) -> Command:
        """执行实际的转接操作。"""
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

### 4.4 Handoff 工具的 3 大组成

| 组成 | 作用 | 项目里的体现 |
| --- | --- | --- |
| **① 名字** | LLM 知道工具是干嘛的 | `transfer_to_flight_booking_agent`（自动生成） |
| **② 描述** | 告诉 LLM 何时调 | `"将任务分配给：flight_booking_agent智能体。"` |
| **③ 函数体** | 被调用时返回 `Command`（跳走 + 更新 state） | `return Command(goto=..., graph=Command.PARENT)` |

### 4.5 5 个 Handoff 实例

```python
# Handoffs
assign_to_research_agent = create_handoff_tool(
    agent_name="research_agent",
    description="将任务分配给：research_agent智能体。",
)
assign_to_flight_booking_agent = create_handoff_tool(
    agent_name="flight_booking_agent",
    description="将任务分配给：flight_booking_agent智能体。",
)
assign_to_hotel_booking_agent = create_handoff_tool(
    agent_name="hotel_booking_agent",
    description="将任务分配给：hotel_booking_agent智能体。",
)
assign_to_car_rental_booking_agent = create_handoff_tool(
    agent_name="car_rental_booking_agent",
    description="将任务分配给：car_rental_booking_agent智能体。",
)
assign_to_excursion_booking_agent = create_handoff_tool(
    agent_name="excursion_booking_agent",
    description="将任务分配给：excursion_booking_agent智能体。",
)
```

**它们都是同一个工厂函数生成的**——只是 `agent_name` 不同。

### 4.6 Handoff 工具 vs 普通工具

| 维度 | 普通工具（如 search_hotels） | Handoff 工具 |
| --- | --- | --- |
| **干啥** | 真的查数据/做操作 | **不做事**——只是"跳走" |
| **结果** | 返回数据 | 返回 `Command`（跳到别处） |
| **谁来调** | supervisor 和 5 个子 agent 都会调 | **只给 supervisor 调** |

### 4.7 Handoff vs add_edge

| 维度 | `add_edge("A", "B")` | Handoff 工具 |
| --- | --- | --- |
| **时机** | 编译时就定好 | **运行时**由 LLM 决定 |
| **灵活性** | 死的 | 活的 |
| **判断者** | 程序员 | LLM |

---

## 5. InjectedState 和 InjectedToolCallId

### 5.1 一句话

> **`InjectedState` + `InjectedToolCallId` = LangGraph 自动注入的"隐藏参数"**——
> LLM **不用传**，框架**自动塞进来**。

### 5.2 对比

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

### 5.3 InjectedState 详解

```python
from typing import Annotated
from langgraph.graph import MessagesState

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

### 5.4 InjectedToolCallId 详解

```python
@tool
def my_tool(tool_call_id: Annotated[str, InjectedToolCallId]):
    return ToolMessage(
        content="工具的结果",
        tool_call_id=tool_call_id,    # ← 用这个 ID 回执
    )
```

**为什么需要它**：LangGraph 用"**调用 ID**"配对"**工具结果**"——AI 调了 `search_hotels("北京")`，LangGraph 给 ID `abc123`，**工具结果必须带 `abc123`**，AI 才知道"哦，原来这条结果对应刚才那个调用"。

**项目里的用法**：

```python
tool_message = {
    "role": "tool",
    "content": f"Successfully transferred to {agent_name}",
    "name": name,
    "tool_call_id": tool_call_id,    # ← ★ 用框架注入的 ID
}
```

### 5.5 Annotated 是什么？

```python
from typing import Annotated

def my_tool(
    state: Annotated[MessagesState, InjectedState],     # ← 双重标注
):
    pass
```

| 部分 | 作用 |
| --- | --- |
| `MessagesState` | 类型注解（IDE 能提示） |
| `InjectedState` | **元数据标记**——告诉 LangGraph "这个参数由你注入" |

**不写 `Annotated`** 只会标注类型，LangGraph 不知道要注入。**`InjectedState` 才是关键**。

### 5.6 完整时序

```
LLM 调 transfer_to_hotel_booking_agent({})
   ↓
LangGraph 框架：检查 handoff_tool 函数签名
   ↓ 发现有两个 Annotated[Injected...]
   ↓
LangGraph 自动准备：
   - state = 当前 state（messages）
   - tool_call_id = 自动生成 ID（如 "call_abc123"）
   ↓
调用 handoff_tool(state=state, tool_call_id="call_abc123")
   ↓
工具返回 Command(...)
   ↓
LangGraph 看到 tool_call_id "call_abc123"，知道这是对应那次调用的回执
```

### 5.7 3 个 Injected 类型对比

| Injected 类型 | 注入什么 | 谁需要用 |
| --- | --- | --- |
| **`InjectedState`** | 当前 state（messages） | 工具需要**读历史消息**时 |
| **`InjectedToolCallId`** | 这次工具调用的 ID | 工具需要**生成 ToolMessage 回执**时 |
| `InjectedStore` | 跨线程的长期存储 | 工具需要"持久记忆"时 |

---

## 6. 7 个概念怎么配合工作

### 6.1 完整时序图（7 概念一起跑）

```
用户："我要改签机票并订北京酒店"
   ↓
[fetch_user_info]（节点 1）
   ↓
[supervisor]（节点 2）—— ReAct 循环 第 1 轮
   ↓ 思考："需要改签 → flight_agent"
   ↓ 调工具：assign_to_flight_booking_agent
   ↓        ↑
   ↓        │ LangGraph 自动注入：
   ↓        │ - state = 当前 state
   ↓        │ - tool_call_id = "call_abc123"
   ↓        ↓
   ↓   handoff_tool 函数返回：
   ↓      Command(
   ↓          goto="flight_booking_agent",
   ↓          update={...messages + [转交消息]},
   ↓          graph=Command.PARENT,    ← ★ 跳回父图
   ↓      )
   ↓
[flight_booking_agent]（节点 3）—— ReAct 循环
   ↓ 思考："查航班"
   ↓ 调 search_flights → 拿到结果
   ↓ 思考："更新机票"
   ↓ 调 update_ticket_to_new_flight → 完成
   ↓ 返回 Command(goto="supervisor", graph=PARENT)   ← ★ 又跳回
   ↓
[supervisor]（节点 2 又来）—— ReAct 循环 第 2 轮
   ↓ 思考："继续搞酒店"
   ↓ 调工具：assign_to_hotel_booking_agent
   ↓        ↑ 同上：注入 state + tool_call_id
   ↓
[hotel_booking_agent]（节点 4）—— ReAct 循环
   ↓ 调 search_hotels → ★ 触发 interrupt
   ↓ 用户输入 "y" → 续跑
   ↓ 完成 → 返回 Command(PARENT) → 跳回 supervisor
   ↓
[supervisor]（节点 2 又来）—— ReAct 循环 第 3 轮
   ↓ 综合："航班改签 + 酒店预订完成"
   ↓ 返回最终答案 → END
```

### 6.2 每个概念在这一过程中扮演什么角色

| 概念 | 角色 | 在哪体现 |
| --- | --- | --- |
| **ReAct 循环** | supervisor 和子 agent 都在循环："想→做→看" | `create_react_agent` 自动跑 |
| **Supervisor 多智能体** | 主管+5 个子 agent 协作 | [graph_chat/all_agent.py](../graph_chat/all_agent.py) |
| **`Command.PARENT`** | 子 agent 干完活"回家"的钥匙 | handoff_tool 的 `return Command(...)` |
| **Handoff 工具** | supervisor 用来"派人"的工具 | 5 个 `assign_to_xxx_agent` |
| **`InjectedState`** | 工具能读 messages | `state: Annotated[MessagesState, InjectedState]` |
| **`InjectedToolCallId`** | 工具回执能配对 | `tool_call_id: Annotated[str, InjectedToolCallId]` |

### 6.3 调用链（谁负责什么）

```
LLM（supervisor）
   ↓ "调 transfer_to_hotel_booking_agent"
LangGraph 框架
   ↓ 自动注入 state + tool_call_id
handoff_tool 函数（业务逻辑）
   ↓ 返回 Command(goto, update, graph=PARENT)
LangGraph 框架
   ↓ 跳到 hotel_booking_agent
LLM（hotel agent）
   ↓ 干活（调 search_hotels 等）
LangGraph 框架
   ↓ Command(PARENT) 跳回 supervisor
```

**每一层只关心自己的事**：

| 层 | 关心什么 | 不关心什么 |
| --- | --- | --- |
| **LLM** | "我该调哪个工具" | state 怎么传、command 怎么写 |
| **LangGraph 框架** | state 怎么传、怎么跳 | LLM 该说什么 |
| **工具函数** | "我该改 state 哪部分、该跳哪" | LLM 的 prompt 是啥 |

---

## 7. 一句话总结

> **5 大概念 = 一次完整的多 agent 对话**：
>
> 1. **ReAct 循环** = "想→做→看"自动循环（框架内置）
> 2. **Supervisor** = 主管 + 5 个子 agent 协作（用 `create_react_agent`）
> 3. **`Command.PARENT`** = 子 agent "回家"的钥匙（`graph=Command.PARENT`）
> 4. **Handoff 工具** = 主管"派人"的工具（`create_handoff_tool` 生成）
> 5. **InjectedState + InjectedToolCallId** = LangGraph 自动注入的隐藏参数

**记忆口诀**：

> - **ReAct** = "想→做→看"
> - **Supervisor** = "主管 + 5 个子"
> - **`Command.PARENT`** = "跳出去，回爸爸那"
> - **Handoff** = "派人"
> - **Injected** = "LangGraph 自动塞的"

---

## 8. interrupt 是什么，有什么作用

### 8.1 一句话

> **`interrupt` = "AI 问你'行不行'，等你回答"**——
> 在 LangGraph 工具函数里调 `interrupt("问题")` → **图暂停**，等用户输入 `Command(resume=...)` 恢复。

### 8.2 `interrupt` 是什么？

`interrupt` 是 LangGraph 提供的一个**函数**——在工具内部调用它，**会暂停整个图的执行**，并把"问题"返回给外层调用方。

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

### 8.3 interrupt vs raise Exception

| 维度 | `raise Exception` | `interrupt(...)` |
| --- | --- | --- |
| **目的** | 报错，终止流程 | 暂停，**等用户回答** |
| **外部如何恢复** | catch 异常 | `Command(resume={...})` |
| **图能否继续** | ❌ 除非 catch | ✅ 继续 |
| **谁给答案** | 程序（catch 里） | 用户（`graph.stream` 调用方） |

**interrupt 不是异常**——它是 LangGraph 自家的"暂停-恢复协议"。

### 8.4 interrupt 的完整工作流

**时序图（一次完整的中断-恢复）**：

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
外层代码（你的 execute_graph）：
   current_state = graph.get_state(config)
   result = current_state.interrupts[0].value  # 拿到 "批准 y?"
   print(result)   # 显示给用户
   ↓
用户输入 "y"
   ↓
graph.stream(Command(resume={'answer': 'y'}), config)
   ↓
★ ★ ★ LangGraph 恢复 ★ ★ ★
   ↓
工具函数从 interrupt("批准 y?") 这一行**继续**往下跑
   ↓
user_response = 'y'  ← ★ 这里 interrupt() 返回 'y'
   ↓
if user_response == 'y':
    return "已执行"   ← 正常返回
```

**关键**：`interrupt(...)` 这个调用**返回一个值**——就是用户**回复的内容**（`'y'` 或 `'n'` 或更复杂的 dict）。

### 8.5 项目里的实际用法

**示例 1：[tools/search_tool.py](../tools/search_tool.py)**

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

        # 调智谱 web_search API
        response = zhipuai_client.web_search.web_search(
            search_engine="search_std",
            search_query=query
        )
        ...
```

**做了什么**：

| 行 | 作用 |
| --- | --- |
| `interrupt("AI大模型尝试调用...批准？")` | 暂停，提示用户 |
| `response = interrupt(...)` | **等用户回答**，用户的回复在 `response` 里 |
| `if response["answer"] == "y":` | 用户说 y → 继续执行搜索 |
| `else: return "...终止..."` | 用户说别的 → 不搜了，返回拒绝原因 |

**注意 `response` 是个 dict**——因为外层 `Command(resume={'answer': user_input})` 用 `{'answer': ...}` 包装了。

**示例 2：[tools/hotels_tools.py](../tools/hotels_tools.py)**

```python
@tool
def search_hotels(location=None, name=None) -> list[dict]:
    print('AI大模型尝试调用工具 `search_hotels`来查询酒店')
    response = interrupt(
        f"AI大模型尝试调用工具 `search_hotels`来查询酒店，\n"
        "请审核并选择：批准（y）或直接给我工具执行的答案。"
    )
    if response["answer"] == "y":
        pass
    else:
        return f"人工终止了该工具的调用，给出的理由或者答案是:{response['answer']}",
    # ... 查数据库
```

**和 search_tool 一模一样**——查酒店前也要人确认。

### 8.6 interrupt 和 Command.resume 的关系

| 方向 | 谁传什么 | 怎么传 |
| --- | --- | --- |
| **interrupt → 用户** | 工具调 `interrupt("问题")` | LangGraph 自动写到 `state.interrupts[].value` |
| **用户 → interrupt** | 外层调 `Command(resume={...})` | `resume` 的值就是工具里 `interrupt()` 的返回值 |

```python
# 工具侧
user_input = interrupt("批准 y?")        # ← 暂停，把"批准 y?"存起来
print(user_input)                          # ← 用户输入 "y" 后，这里拿到 "y"

# 外层（execute_graph）
graph.stream(Command(resume={'answer': 'y'}), config)  # ← 'y' 会变成 user_input 的值
```

**配对关系**：
- 工具 `interrupt("问题")` ↔ 外层 `Command(resume={'answer': '...'})`
- 工具拿到的 `user_input` = `Command(resume=...)` 的内容

### 8.7 interrupt 的 4 个要点

| 要点 | 内容 |
| --- | --- |
| **① 暂停图执行** | 让"自动跑"变成"停下来等用户" |
| **② 是个函数** | `interrupt("问题")` 而不是关键字 |
| **③ 有返回值** | 用户回复后，工具能拿到 |
| **④ 在工具里调** | 通常是 `@tool` 装饰的函数内部 |

### 8.8 一句话总结

> **`interrupt` = "AI 问你'行不行'，等你回答"**：
> - 在工具函数里 `interrupt("问题")` → **图暂停**
> - 工具的 `interrupt()` 调用**返回一个值**——就是用户回复的内容
> - 外层用 `Command(resume=...)` 恢复 + 传用户输入
> - 项目里 [search_tool.py](../tools/search_tool.py) 和 [hotels_tools.py](../tools/hotels_tools.py) 都用它——**敏感操作前先问用户**
> - 配合 `get_state(config).interrupts[0].value` 拿到问题内容

---

## 9. interrupt 需要 4 件套搭配使用

### 9.1 一句话

> **是的**——`interrupt` 是"半成品"，必须配合 **`Command(resume=...)` + `get_state` + `stream()` 循环** 4 件套才能工作。少一个都不行。

### 9.2 4 件套缺一不可

| # | 谁负责 | 做什么 | 不配合的后果 |
| --- | --- | --- | --- |
| ① | **工具函数里** | `response = interrupt("问题")` | ❌ 啥也不发生（甚至报错） |
| ② | **外层代码** | `graph.stream(Command(resume={'answer': user_input}), config)` | ❌ 图继续跑，不会停 |
| ③ | **外层代码** | `current_state = graph.get_state(config)` | ❌ 不知道"有没有中断" |
| ④ | **外层代码** | `if current_state.next: result = current_state.interrupts[0].value` | ❌ 拿不到"问题内容" |

### 9.3 4 件套完整工作流

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
[search_hotels 返回空（不是异常）]
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

### 9.4 最小可运行示例

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
            # 不打印最终答案，因为 AI 在等用户
        # else:
        #     print("AI: " + 最终答案)  # 可选
```

**6 个核心点**全部要写：

1. `interrupt(...)` 在工具里
2. `graph.stream(...)` 跑图
3. `graph.get_state(config)` 查状态
4. `state.next` 判断中断
5. `state.interrupts[0].value` 拿问题
6. `Command(resume={...})` 续跑

### 9.5 3 个常见误区

| 误区 | 为什么错 |
| --- | --- |
| ❌ "光写 `interrupt` 就能用" | interrupt 只是"暂停信号"，没外层响应就**永远停着** |
| ❌ "用 `raise Exception` 代替" | raise 出来的异常需要 catch，`interrupt` 是 LangGraph 自家协议 |
| ❌ "用 `add_edge` 写条件边就行" | 条件边是**编译时定死的**；`interrupt` 是**运行时**让 LLM 决定是否触发 |

### 9.6 4 件套的"角色分工"

| 角色 | 谁 | 工具 | 职责 |
| --- | --- | --- | --- |
| **暂停器** | 工具代码 | `interrupt(...)` | 暂停 + 抛问题 |
| **观察者** | 外层代码 | `get_state` | 查图当前状态 |
| **判断者** | 外层代码 | `if state.next` | 决定"是中断中吗" |
| **恢复器** | 外层代码 | `Command(resume={...})` | 续跑 + 传答案 |

### 9.7 一句话总结

> **是的，`interrupt` 必须 4 件套搭配**：
> 1. 工具里写 `interrupt("问题")` —— **暂停+抛问题**
> 2. 外层用 `graph.stream(...)` —— **跑图**
> 3. 外层用 `graph.get_state(config)` —— **查图状态**
> 4. 如果 `state.next` 非空，用 `Command(resume={...})` —— **续跑+传答案**
> 5. 用 `state.interrupts[0].value` —— **拿问题内容显示给用户**

**少了任何一步都会失败**——`interrupt` 只是"半成品协议"，必须 4 件套配合。

---

## 10. 全文档总结

### 10.1 7 大核心概念

| 概念 | 一句话 | 关键文件 |
| --- | --- | --- |
| **ReAct 循环** | "想→做→看"自动循环 | LangGraph 内置 |
| **Supervisor 多智能体** | 主管+5 个子 agent | [graph_chat/all_agent.py](../graph_chat/all_agent.py) |
| **`Command.PARENT`** | "跳出去，回到爸爸那" | 子 agent 干完活回主图 |
| **Handoff 工具** | "派人" | `create_handoff_tool` 工厂 |
| **InjectedState/InjectedToolCallId** | LangGraph 自动注入的隐藏参数 | `Annotated[..., InjectedState]` |
| **`interrupt`** | "AI 问你行不行" | 4 件套搭配用 |

### 10.2 记忆口诀

> - **ReAct** = "想→做→看"
> - **Supervisor** = "主管 + 5 个子"
> - **`Command.PARENT`** = "跳出去，回爸爸那"
> - **Handoff** = "派人"
> - **Injected** = "LangGraph 自动塞的"
> - **`interrupt`** 是嘴，`Command(resume=)` 是耳朵——少一边都聋
> - **`interrupt` 必须 4 件套搭配**才能工作

### 10.3 7 大概念怎么"串"起来

```
用户输入 "我要改签机票并订酒店"
   ↓
[fetch_user_info]（预加载背景）
   ↓
[supervisor] 调 handoff_tool（用 InjectedState 读消息）
   ↓ 跳到 flight_booking_agent（Command.PARENT 配合）
   ↓
[flight_booking_agent] ReAct 循环干活
   ↓ 完成 → Command(PARENT) 跳回 supervisor
   ↓
[supervisor] 又调 handoff_tool
   ↓ 跳到 hotel_booking_agent
   ↓
[hotel_booking_agent] 工具里调 interrupt("批准 y?")
   ↓ ★ 图暂停
   ↓
外层 execute_graph 用 4 件套恢复
   ↓ 用户输入 y
   ↓
[hotel_booking_agent] 续跑 → 完成 → Command(PARENT) 回 supervisor
   ↓
[supervisor] 综合答案 → END
```