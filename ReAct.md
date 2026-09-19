# ReAct 完全学习指南（新手向）

> 本文档专为**新手**撰写，把 ReAct 从"是什么"到"怎么用"一次讲清楚。
> 读完这篇，你不仅能理解 ReAct 的原理，还能看懂本项目 `langgraph_mcp/agent_mcp.py` 里 `create_react_agent` 那行代码到底在干嘛。

---

## 📑 目录（点击跳转）

- [一、ReAct 是什么？](#一react-是什么)
- [二、为什么要用 ReAct？](#二为什么要用-react)
- [三、ReAct 的核心思想](#三react-的核心思想)
  - [3.1 三个关键词：Thought / Action / Observation](#31-三个关键词thought--action--observation)
  - [3.2 ReAct 的循环流程](#32-react-的循环流程)
- [四、一个完整例子](#四一个完整例子)
  - [4.1 场景：问"北京今天多少度？"](#41-场景问北京今天多少度)
  - [4.2 伪代码逐步拆解](#42-伪代码逐步拆解)
- [五、ReAct 的提示词模板](#五react-的提示词模板)
- [六、ReAct vs 其他方式](#六react-vs-其他方式)
- [七、LangGraph 中的 ReAct：create_react_agent](#七langgraph-中的-reactcreate_react_agent)
- [八、本项目里的 ReAct 实战](#八本项目里的-react-实战)
  - [8.1 agent_mcp.py —— 最简 ReAct + MCP](#81-agent_mcppy--最简-react--mcp)
  - [8.2 graph_mcp.py —— 自定义状态图（也包含 ReAct 节点）](#82-graph_mcppy--自定义状态图也包含-react-节点)
- [九、动手跑一跑](#九动手跑一跑)
- [十、常见疑问](#十常见疑问)

---

## 一、ReAct 是什么？

**ReAct** = **Re**asoning + **Act**ing，中文是"**推理 + 行动**"。

它是一种让大模型"**边想边做、做完再想**"的工作方式，由 Princeton 大学的 Shunyu Yao 等人在 2022 年提出（论文《ReAct: Synergizing Reasoning and Acting in Language Models》）。

**一句话定义**：

> ReAct 让大模型在回答问题前，先输出"思考过程"（Thought），再决定"要不要调用工具"（Action），拿到"工具结果"（Observation）后再继续思考，如此循环，直到能给出最终答案为止。

---

## 二、为什么要用 ReAct？

在 ReAct 出现之前，有两种主流做法，但都有问题：

| 做法 | 优点 | 缺点 |
|------|------|------|
| **纯推理（Chain-of-Thought）** | 让模型"想清楚再答"，复杂问题表现好 | 模型容易"瞎想"，编造不存在的事实 |
| **纯行动（Action-only）** | 直接调工具，结果可靠 | 不会推理，遇到多步任务就懵 |

**ReAct 把两者结合**：

- 先**推理**决定下一步做什么（避免乱调工具）
- 再**行动**真正调用工具（结果真实可靠）
- 拿到结果后**再推理**判断任务完成没（决定是否继续）

**通俗类比**：

```
纯推理 ≈ 一个"纸上谈兵"的将军，只会在沙盘上推演
纯行动 ≈ 一个"只会按按钮"的士兵，没有判断力
ReAct   ≈ 一个能"想一步做一步"的指挥官
```

---

## 三、ReAct 的核心思想

### 3.1 三个关键词：Thought / Action / Observation

ReAct 把大模型的每一步输出拆成三个部分：

| 关键词 | 中文 | 是什么 | 例子 |
|--------|------|--------|------|
| **Thought** | 思考 | 大模型的"心里话"，说明自己为什么这么做 | "用户问的是天气，我需要先查一下" |
| **Action** | 行动 | 大模型决定**调用哪个工具** + **传什么参数** | `Action: get_weather[北京]` |
| **Observation** | 观察 | **工具执行后返回的结果**（由我们填回去） | "北京今天 22°C，晴天" |

**注意**：
- **Thought** 和 **Action** 是大模型自己输出的
- **Observation** 是**工具的真实返回结果**（由代码塞回去，不是模型编的）

### 3.2 ReAct 的循环流程

```
用户提问：今天北京多少度？我明天想去爬山，适合吗？
   ↓
┌────────────────────────────────────────────────────┐
│  ReAct 循环                                          │
│                                                     │
│  Thought 1: 用户问了两个问题，天气和爬山建议。           │
│             我得先查天气，才能判断爬山是否合适。          │
│             → 我应该调 get_weather 工具                │
│  Action 1:  get_weather[北京]                         │
│  Observation 1: 北京今天 22°C，晴天，西北风 3 级        │
│                                                     │
│  Thought 2: 天气不错，适合爬山。可以给建议了。           │
│             → 不需要再调工具了                          │
│  Final Answer: 北京今天 22°C 晴天，很适合爬山……         │
└────────────────────────────────────────────────────┘
```

**关键点**：

1. **循环直到出现 Final Answer**：模型可以反复思考、调工具，直到拿到所有需要的信息
2. **每次循环都用尽三个关键词**：少一个就可能让模型跑偏
3. **Thought 是"自言自语"**：这一段不会给用户看，只是给模型自己理思路

---

## 四、一个完整例子

### 4.1 场景：问"北京今天多少度？"

我们假设有 2 个工具：
- `get_weather(city)` —— 查天气
- `multiply(a, b)` —— 乘法

### 4.2 伪代码逐步拆解

```python
# ============ 第 1 步：用户问问题 ============
user_input = "北京今天多少度？"

# ============ 第 2 步：第 1 轮 ReAct 循环 ============
# 把"用户问题 + 工具说明 + 历史"打包，发给大模型
response_1 = llm.invoke(prompt_with_user_input_and_tools)

# 模型输出（伪）：
# Thought: 用户问天气，我得调 get_weather
# Action: get_weather[北京]
# → 解析出 action_name = "get_weather", action_args = {"city": "北京"}

# ============ 第 3 步：执行工具 ============
observation_1 = get_weather("北京")  # 执行真实函数
# 返回："22°C，晴天"

# ============ 第 4 步：第 2 轮 ReAct 循环 ============
# 把"用户问题 + 第1轮 thought/action + observation_1"再发给大模型
response_2 = llm.invoke(prompt_with_history_and_observation)

# 模型输出（伪）：
# Thought: 拿到天气数据了，可以回答用户了
# Final Answer: 北京今天 22°C，晴天，适合出门～

# ============ 第 5 步：返回最终答案 ============
final_answer = "北京今天 22°C，晴天，适合出门～"
```

**实际打印出来大概长这样**：

```text
Thought 1: 用户问的是天气，需要调用 get_weather 工具。
Action 1: get_weather
Action Input 1: {"city": "北京"}
Observation 1: 22°C，晴天

Thought 2: 已经拿到天气数据，可以直接回答用户了。
Final Answer: 北京今天 22°C，晴天，很适合出门哦～
```

---

## 五、ReAct 的提示词模板

ReAct 之所以能跑起来，靠的是一段精心设计的**提示词**，把"思考+行动+观察"的格式灌给模型。下面是一段简化版的提示词（实际框架里会有更复杂的版本）：

```text
你可以使用以下工具来回答问题：

工具 1: get_weather(city: str)
  描述：查询指定城市的天气
工具 2: multiply(a: int, b: int)
  描述：计算两个数字相乘

请按以下格式回答：

Thought: 你应该思考下一步要做什么
Action: 工具名
Action Input: 工具的参数（JSON 格式）
Observation: 工具返回的结果（这一行由系统填，不要自己写）
... (Thought/Action/Observation 可以重复 N 次)
Thought: 我现在知道最终答案了
Final Answer: 给用户的最终回答

开始！

Question: 北京今天多少度？
Thought:
```

**关键设计**：

1. **明确告诉模型三种格式**：Thought / Action / Final Answer
2. **强调 Observation 由系统填**：避免模型自己编造结果
3. **预留"循环"语义**："Thought/Action/Observation 可以重复 N 次"
4. **结尾"开始！"**：让模型从这里开始输出

---

## 六、ReAct vs 其他方式

| 方式 | 推理能力 | 工具调用 | 多步任务 | 适用场景 |
|------|---------|---------|---------|---------|
| **普通对话** | ❌ 弱 | ❌ 无 | ❌ 不行 | 闲聊 |
| **Chain-of-Thought (CoT)** | ✅ 强 | ❌ 无 | ⚠️ 容易编 | 数学题、推理题 |
| **Function Calling (FC)** | ❌ 弱 | ✅ 强 | ⚠️ 单步 | 单次查数据 |
| **ReAct** | ✅ 强 | ✅ 强 | ✅ 强 | **复杂多步任务** |

**一句话区别**：

- FC = 让模型**一次**决定调什么工具
- ReAct = 让模型**反复**思考 + 调工具，直到问题解决

---

## 七、LangGraph 中的 ReAct：create_react_agent

LangGraph 已经把 ReAct 封装好了，你只需要一行代码就能用：

```python
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(llm, tools=[...])   # 创建 ReAct Agent
resp = agent.invoke({"messages": "计算一下(3+6)"})   # 调用
```

### 它帮你做了什么？

| 步骤 | 手动写 ReAct 你要做的 | create_react_agent 帮你做的 |
|------|--------------------|---------------------------|
| ① 构造提示词模板 | 自己拼接 Thought/Action/Observation 格式 | ✅ 自动注入 |
| ② 调用大模型 | 手写 invoke + 解析响应 | ✅ 自动 |
| ③ 解析 Thought / Action | 用正则提取 "Action: xxx" | ✅ 自动 |
| ④ 执行工具 | 手动调用对应函数 | ✅ 自动 |
| ⑤ 把 Observation 塞回上下文 | 手动拼接 messages | ✅ 自动 |
| ⑥ 判断是否到 Final Answer | 检查响应里有没有 "Final Answer:" | ✅ 自动 |
| ⑦ 决定是否继续循环 | if 判断 + 递归 | ✅ 自动 |

**所以 create_react_agent 就是一个"开箱即用的 ReAct 循环机器"**。

---

## 八、本项目里的 ReAct 实战

回到 [readme.md](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/docs/readme.md) 第 447 行：

```
| **LangGraph ReAct** | `langgraph_mcp/agent_mcp.py`、`graph_mcp.py` | `create_react_agent` / `StateGraph` |
```

两个文件用了不同的"姿势"来用 ReAct：

### 8.1 [agent_mcp.py](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/langgraph_mcp/agent_mcp.py) —— 最简 ReAct + MCP

**关键代码**（节选）：

```python
async with MultiServerMCPClient({'lx_mcp': mcp_server_config}) as client:
    # 这一行就是"ReAct 的核心"
    agent = create_react_agent(llm, tools=client.get_tools())

    # 调用 Agent（ReAct 循环在内部自动跑）
    resp = await agent.ainvoke({'messages': '计算一下(3+6)的结果'})
```

**ReAct 在这里的体现**：

1. `client.get_tools()` —— 从 MCP 服务器拿所有工具（add、multiply、my_search_tool）
2. `create_react_agent(llm, tools=...)` —— 把这些工具喂给 ReAct Agent
3. 用户问 `"计算一下(3+6)的结果"` —— ReAct 会自动：
   - Thought：用户要做加法，调 add 工具
   - Action：add(3, 6)
   - Observation：9
   - Final Answer：3+6=9

### 8.2 [graph_mcp.py](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/langgraph_mcp/graph_mcp.py) —— 自定义状态图（也包含 ReAct 节点）

**关键代码**（节选）：

```python
async def async_node(state: MyState):
    """节点：调用 ReAct Agent 处理用户消息"""
    async with MultiServerMCPClient({'lx_mcp': mcp_server_config}) as client:
        agent = create_react_agent(llm, tools=client.get_tools())
        # 这里也调了 create_react_agent —— ReAct 是节点内部的实现
        resp = await agent.ainvoke(state)
        return resp
```

**ReAct 在这里的体现**：

- `async_node` 这个节点**内部**用的还是 ReAct Agent
- 但是外层用 `StateGraph` 把"读资源"和"调 Agent"两个步骤**手动串起来**

**对比**：

```
agent_mcp.py：直接一个 ReAct Agent 干完所有事
              ┌──────────────────┐
              │  ReAct Agent     │ ← 一个节点搞定一切
              └──────────────────┘

graph_mcp.py：把 ReAct Agent 嵌进状态图
              ┌──────────┐    ┌──────────────────┐
              │ resource │ →  │  ReAct Agent     │
              └──────────┘    └──────────────────┘
                读邮箱              调工具回答
```

---

## 九、动手跑一跑

想看 ReAct 实际打印的 Thought/Action/Observation？把 [agent_mcp.py](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/langgraph_mcp/agent_mcp.py) 改两行：

```python
# 改前
resp = await agent.ainvoke({'messages': '计算一下(3+6)的结果'})
print(resp)

# 改后（流式输出，能看到 ReAct 每一步）
async for event in agent.astream({'messages': '计算一下(3+6)的结果'}):
    print(event)
    print('---')
```

运行后你会看到类似输出：

```text
{'agent': {'messages': [AIMessage(content='Thought: 用户要做加法...')]}}
---
{'tools': {'messages': [ToolMessage(content='9')]}}
---
{'agent': {'messages': [AIMessage(content='Final Answer: 3+6=9')]}}
```

---

## 十、常见疑问

### Q1：ReAct 和普通 Agent 有什么区别？

**A**：ReAct 是一种**具体的 Agent 实现方式**（"推理+行动"循环）。其他 Agent 范式还包括：
- **Plan-and-Execute**（先规划再执行）
- **Reflexion**（带自我反思）
- **BabyAGI**（任务拆解 + 优先级队列）

LangGraph 的 `create_react_agent` 默认走的就是 ReAct 路线。

### Q2：ReAct 会"卡死"循环吗？

**A**：会。所以一般要设两个保护：
1. **最大循环次数**（比如最多 5 轮）
2. **超时时间**（比如 30 秒）

LangGraph 的 `AgentExecutor` 默认 `max_iterations=15`，超过会强制结束。

### Q3：Observation 一定要是真的吗？

**A**：**必须是真实的工具结果**，不能让模型自己编。ReAct 的精髓就是"用真实数据修正推理"。

### Q4：本项目里 ReAct 用的是哪个模型？

**A**：智谱的 `glm-4-air-250414`（在 [zhipu_ai.py](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/zhipu_ai.py) 里配置）。

ReAct 本身和模型无关，**任何能遵循指令格式的 LLM 都能跑**。

---

## 附录：相关文档跳转

- 项目整体介绍：[readme.md](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/docs/readme.md)
- 项目文件分类：[项目整理.md](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/docs/%E9%A1%B9%E7%9B%AE%E6%95%B4%E7%90%86.md)
- GitHub 推送流程：[GitHub代码提交文件.md](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/docs/GitHub%E4%BB%A3%E7%A0%81%E6%8F%90%E4%BA%A4%E6%96%87%E4%BB%B6.md) / [git clone上传方式.md](file:///D:/BaiduNetdiskDownload/AI%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%B7%A5%E7%A8%8B%E5%B8%88/02_%E5%BA%94%E7%94%A8%E7%AF%87/11_%E5%9F%BA%E4%BA%8EMCP%E7%9A%84Agent%E5%BC%80%E5%8F%91/00_%E8%AF%BE%E7%A8%8B%E8%B5%84%E6%96%99/MCP_DEMO/MCP_DEMO/docs/git%20clone%E4%B8%8A%E4%BC%A0%E6%96%B9%E5%BC%8F.md)

---

> **最后一句话**：ReAct 不是神秘的算法，它只是把大模型的"心里话"（Thought）和"动手做"（Action）写在了 prompt 里，再用工具的真实结果（Observation）去修正它。仅此而已。
