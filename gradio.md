# Gradio 完全入门指南（小白版）

> 写给"完全没接触过 Gradio"的同学。本文档会**手把手**告诉你：
> - Gradio 是什么、解决什么问题
> - 它和 Streamlit / Flask / FastAPI 有什么区别
> - 怎么写一个能跑的 Gradio 应用
> - 怎么读懂别人写的 Gradio 代码
> - 怎么用它把 AI 模型做成"网页应用"

---

## �� 目录

- [1. Gradio 是什么？](#1-gradio-是什么)
- [2. 为什么要用 Gradio？](#2-为什么要用-gradio)
- [3. 5 分钟速通：写一个聊天界面](#3-5-分钟速通写一个聊天界面)
- [4. 9 个核心概念](#4-9-个核心概念)
- [5. 常用组件速查](#5-常用组件速查)
- [6. 进阶玩法](#6-进阶玩法)
- [7. 看懂本项目里的 Gradio 代码](#7-看懂本项目里的-gradio-代码)
- [8. 怎么读懂别人写的 Gradio 代码（5 步法）](#8-怎么读懂别人写的-gradio-代码5-步法)
- [9. 常见概念速查表](#9-常见概念速查表)

---

## 1. Gradio 是什么？

**一句话**：Gradio 是一个"**用 Python 函数直接生成网页界面**"的库。

它干这一件事：
> 你写一个 Python 函数 → Gradio 自动生成一个网页 → 用户在网页里输入 → Gradio 把输入传给你的函数 → 你的函数返回结果 → Gradio 自动展示给用户。

**场景**（本项目）：你写了一个 LangGraph AI 助手（在 Python 里跑），Gradio 让用户在浏览器聊天框里跟它对话。

### 1.1 最直观的对比

**没有 Gradio**（纯 Python）：
```python
# 想让用户用 AI，必须自己写网页、写 JS、写后端...
# 至少几百行代码
```

**用 Gradio**：
```python
import gradio as gr

def chat(message, history):
    return f"你说的是：{message}"

gr.ChatInterface(chat).launch()
# → 浏览器自动打开聊天界面
```

**3 行 = 一个能用的聊天网页** ✓

---

## 2. 为什么要用 Gradio？

| 工具 | 适合谁 | 写法 | 风格 |
| --- | --- | --- | --- |
| **Gradio** | 演示 / 给 AI 模型做界面 | `gr.Interface(fn, inputs, outputs)` | **3 行出界面** |
| **Streamlit** | 数据看板 / 内部工具 | `st.write(...)` | 像写 Word |
| **Flask / FastAPI** | 真实后端 API | 路由 + 视图函数 | 要自己写前端 |
| **Django** | 完整 Web 项目 | MTV 架构 | 大而全 |

**Gradio 的 4 大优势**：

1. **3 行出网页**——不用 HTML/CSS/JS
2. **自动适配各种输入**——文本框 / 文件上传 / 麦克风 / 摄像头 全自动
3. **支持大模型**——专门为 ML / LLM 设计的
4. **一键分享**——可以生成临时公网链接

---

## 3. 5 分钟速通：写一个聊天界面

### 3.1 安装

```bash
pip install gradio
```

### 3.2 最简单的例子

```python
# app.py
import gradio as gr

def greet(name, intensity):
    """你的业务函数"""
    return "Hello, " + name + "!" * intensity

# 创建界面
demo = gr.Interface(
    fn=greet,                              # 处理函数
    inputs=[                                # 输入组件列表
        gr.Textbox(label="你的名字"),
        gr.Slider(minimum=1, maximum=10, value=3, label="感叹号数量"),
    ],
    outputs=gr.Textbox(label="问候"),       # 输出组件
    title="我的第一个 Gradio",
    description="这是一个最简示例",
)

# 启动
demo.launch()
```

跑起来后：

1. 终端打印 `Running on local URL: http://127.0.0.1:7860`
2. **自动打开浏览器**
3. 你看到一个网页，名字+感叹号 → 输出问候语

### 3.3 跑起来的效果

- 左：两个输入框（文本框 + 滑块）
- 右：一个输出文本框
- 点 "Submit" → 调你的 `greet()` 函数 → 显示结果

---

## 4. 9 个核心概念

| 概念 | 通俗解释 | 代码对应 |
| --- | --- | --- |
| **Interface** | "整个界面"对象 | `gr.Interface(...)` |
| **fn** | 处理用户输入的 Python 函数 | `fn=my_function` |
| **inputs** | 一组输入组件（文本框、上传框...） | `inputs=[gr.Textbox(), gr.Slider()]` |
| **outputs** | 一组输出组件（文本、图像、音频） | `outputs=gr.Textbox()` |
| **Component（组件）** | 单独的输入/输出小部件 | `gr.Textbox / gr.Image / gr.Audio` |
| **Blocks** | 用"积木"自己拼界面 | `with gr.Blocks() as demo:` |
| **launch()** | 启动服务（生成 URL） | `demo.launch()` |
| **share=True** | 生成临时公网链接 | `demo.launch(share=True)` |
| **ChatInterface** | 专门做聊天界面的快捷类 | `gr.ChatInterface(fn=chat_fn)` |

### 4.1 `Interface` vs `Blocks` vs `ChatInterface`

```python
# ============ 1) Interface：最简单，按 inputs / outputs 自动布局 ============
gr.Interface(fn=greet, inputs=["text"], outputs="text")

# ============ 2) Blocks：自己摆位置 ============
with gr.Blocks() as demo:
    gr.Markdown("# 我的应用")
    with gr.Row():                     # 一行
        inp = gr.Textbox(label="输入")
        out = gr.Textbox(label="输出")
    btn = gr.Button("运行")
    btn.click(fn=greet, inputs=inp, outputs=out)

# ============ 3) ChatInterface：专为聊天设计 ============
gr.ChatInterface(fn=chatbot).launch()
```

### 4.2 函数签名约定（**这是最常出错的地方**）

Gradio 通过**函数参数名**和**返回值**决定输入/输出：

```python
# 单输入单输出
def fn(name):                  # 一个参数 -> 一个输入框
    return f"hi {name}"

# 多输入
def fn(name, age):            # 两个参数 -> 两个输入框
    return f"{name} is {age}"

# 多输出（按位置对应）
def fn(text):
    return text.upper(), text.lower()   # 返回元组 -> 两个输出框

# 聊天（特殊签名）
def chat(message, history):    # 第一参=用户消息，第二参=历史
    return "回复"
```

**关键**：参数**名字**不重要，**顺序**和**数量**才重要。

---

## 5. 常用组件速查

### 5.1 输入组件

| 组件 | 用途 | 示例 |
| --- | --- | --- |
| `gr.Textbox` | 文本输入框 | `gr.Textbox(placeholder="请输入", lines=3)` |
| `gr.Number` | 数字输入 | `gr.Number(value=10, label="数量")` |
| `gr.Slider` | 滑块 | `gr.Slider(minimum=0, maximum=100, step=5)` |
| `gr.Checkbox` | 复选框 | `gr.Checkbox(label="同意", value=True)` |
| `gr.Radio` | 单选 | `gr.Radio(choices=["A", "B", "C"])` |
| `gr.Dropdown` | 下拉 | `gr.Dropdown(choices=["北京", "上海"])` |
| `gr.Image` | 图片上传 | `gr.Image(type="filepath")` |
| `gr.File` | 文件上传 | `gr.File()` |
| `gr.Audio` | 音频 | `gr.Audio(type="filepath")` |
| `gr.Dataframe` | 表格 | `gr.Dataframe(value=df)` |

### 5.2 输出组件

| 组件 | 用途 | 示例 |
| --- | --- | --- |
| `gr.Textbox` | 显示文本 | `gr.Textbox(label="结果")` |
| `gr.Image` | 显示图片 | `gr.Image()` |
| `gr.Audio` | 播放音频 | `gr.Audio()` |
| `gr.Dataframe` | 显示表格 | `gr.Dataframe()` |
| `gr.Label` | 显示标签+置信度 | `gr.Label(num_top_classes=3)` |
| `gr.JSON` | 显示 JSON | `gr.JSON()` |
| `gr.HTML` | 显示 HTML | `gr.HTML()` |
| `gr.Plot` | 显示 matplotlib 图表 | `gr.Plot(fig)` |

### 5.3 布局组件

| 组件 | 用途 |
| --- | --- |
| `gr.Tab` / `gr.Tabs` | 选项卡 |
| `gr.Row` | 水平排成一行 |
| `gr.Column` | 垂直排成一列 |
| `gr.Group` | 一组（视觉上分组） |
| `gr.Accordion` | 可折叠面板 |

---

## 6. 进阶玩法

### 6.1 流式输出（边生成边显示）

```python
import gradio as gr
import time

def slow_stream(message):
    """每 0.1 秒返回一个字符"""
    for i in range(len(message)):
        time.sleep(0.1)
        yield message[:i+1]   # yield 不是 return，是"流式"

demo = gr.Interface(slow_stream, "textbox", "textbox", stream=True)
demo.launch()
```

`yield` 关键字 → **流式响应** → 像 ChatGPT 一样**一个字一个字**蹦出来。

### 6.2 多步骤交互（按钮触发）

```python
import gradio as gr

def analyze(text):
    return f"分析结果：{text[::-1]}"   # 字符串反转

with gr.Blocks() as demo:
    inp = gr.Textbox(label="输入")
    out = gr.Textbox(label="结果")
    btn = gr.Button("分析")
    # 把按钮的 click 事件接到函数
    btn.click(fn=analyze, inputs=inp, outputs=out)

demo.launch()
```

### 6.3 聊天 + 多模态（图片 + 文本）

```python
import gradio as gr

def chat_with_image(message, history, image):
    """Gradio 自动把上传的图片作为额外参数"""
    if image is not None:
        return f"我看到图片：{image.name}，你说的：{message}"
    return f"你说：{message}"

demo = gr.ChatInterface(
    fn=chat_with_image,
    additional_inputs=gr.Image(type="filepath"),
)
demo.launch()
```

### 6.4 一键分享公网链接

```python
demo.launch(share=True)
# 输出：
# Running on local URL:  http://127.0.0.1:7860
# Running on public URL: https://xxxxxx.gradio.live   ← 任何人能访问，72 小时有效
```

---

## 7. 看懂本项目里的 Gradio 代码

打开 [graph_chat/graph_gradio.py](../../graph_chat/graph_gradio.py)：

```python
import gradio as gr
from langgraph.checkpoint.memory import MemorySaver
# ...

# 全局唯一实例
memory = MemorySaver()
graph = builder.compile(
    checkpointer=memory,
    interrupt_before=[...]
)

# 函数定义
def execute_graph(chat_bot, history):
    """用户问 → graph 工作流 → 返回结果"""
    # 1. 用历史最后一条当输入
    message = history[-1]['content'] if history else ""
    # 2. 调 graph
    for event in graph.stream({'messages': ('user', message)}, config):
        # ... 解析 event，提取 AI 回复
    # 3. 把回复追加到 history
    return "", history + [{'role': 'user', 'content': message},
                           {'role': 'assistant', 'content': result}]

# 关键：用 gr.Blocks + gr.Chatbot 自己拼界面
with gr.Blocks(title='携程AI智能助手', css=css) as instance:
    gr.Label('携程AI智能助手')                      # 顶部标题
    chatbot = gr.Chatbot(type='messages', height=350)  # 聊天显示区
    input_textbox = gr.Textbox(label='请输入你的问题')  # 输入框
    # 提交事件链：输入 → 清空输入框 → 更新聊天
    input_textbox.submit(
        execute_graph,                  # 处理函数
        [input_textbox, chatbot],       # 输入
        [input_textbox, chatbot]        # 输出
    )

instance.launch()
```

### 7.1 逐块拆解

| 代码 | 含义 |
| --- | --- |
| `with gr.Blocks(...) as instance:` | "画布"——接下来开始往里放组件 |
| `gr.Label(...)` | 顶部大标题 |
| `gr.Chatbot(type='messages', ...)` | 聊天显示区（type='messages' 表示按"用户/AI"气泡显示） |
| `gr.Textbox(...)` | 用户输入框 |
| `input_textbox.submit(execute_graph, ...)` | **关键**：提交时触发函数，第一个返回值赋给输入框（清空），第二个给聊天框 |
| `instance.launch()` | 启动 |

---

## 8. 怎么读懂别人写的 Gradio 代码（5 步法）

### Step 1：找 `gr.XXX` 看"用了哪种界面"
```python
gr.Interface(...)             # → 自动布局
gr.ChatInterface(...)         # → 聊天
with gr.Blocks() as demo:     # → 自己拼
```

### Step 2：找函数（关键）
```python
def fn(inputs...):
    return outputs
```
- 参数 = 输入
- 返回 = 输出
- 看参数顺序 = 看输入框顺序
- 看返回元组 = 看有几个输出框

### Step 3：找事件触发
```python
btn.click(fn=..., inputs=..., outputs=...)   # 按钮
textbox.submit(fn=..., inputs=..., outputs=...)  # 回车
slider.change(fn=..., inputs=..., outputs=...)   # 滑动
dropdown.select(fn=..., inputs=..., outputs=...)  # 选中
```

### Step 4：找 `launch()`
```python
demo.launch()
demo.launch(server_name="0.0.0.0", server_port=8080)
demo.launch(share=True)         # 生成公网链接
```

### Step 5：找 `gr.ChatInterface(fn=chatbot)` 这种"快捷版"
- 它封装了所有输入输出
- 看 `fn` 即可
- 看 `additional_inputs=[...]` 看有没有额外输入

---

## 9. 常见概念速查表

| 概念 | 含义 | 代码示例 |
| --- | --- | --- |
| `gr.Interface` | 最简界面（自动布局） | `gr.Interface(fn, inputs, outputs)` |
| `gr.Blocks` | 自定义布局 | `with gr.Blocks() as demo: ...` |
| `gr.ChatInterface` | 聊天界面 | `gr.ChatInterface(fn)` |
| `gr.Tab` / `gr.Tabs` | 选项卡 | `with gr.Tabs(): with gr.Tab("A"): ...` |
| `gr.Row` / `gr.Column` | 行/列布局 | `with gr.Row(): ...` |
| `gr.Textbox` | 文本框 | `gr.Textbox(label=..., lines=3)` |
| `gr.Image` | 图片上传/显示 | `gr.Image(type="filepath")` |
| `gr.Audio` | 音频 | `gr.Audio(type="filepath")` |
| `gr.Button` | 按钮 | `gr.Button("提交")` |
| `gr.Slider` | 滑块 | `gr.Slider(0, 100)` |
| `gr.Chatbot` | 聊天显示 | `gr.Chatbot(type='messages')` |
| `.click()` | 按钮点击事件 | `btn.click(fn, inputs, outputs)` |
| `.submit()` | 表单提交 | `textbox.submit(fn, inputs, outputs)` |
| `.change()` | 值变化事件 | `slider.change(fn, inputs, outputs)` |
| `stream=True` | 流式输出 | `gr.Interface(..., stream=True)` |
| `share=True` | 生成公网链接 | `demo.launch(share=True)` |

---

## �� 一句话总结

> **Gradio = "给 Python 函数自动生成网页界面"**：
> - 写个 `def fn(x): return ...` → 用 `gr.Interface(fn, inputs, outputs)` 包一下 → 浏览器自动能调
> - 跟 **LangGraph 配合**：把 LangGraph 编译出来的 `graph` 包进 Gradio → 一个能聊天的 AI 网页就出来了
> - 一行 `demo.launch(share=True)` 就能让全世界的人用你的 AI（72 小时临时链接）
> 
> **记住 3 件事**：
> - 函数 = 业务（你写）
> - `Interface/Blocks/ChatInterface` = 界面（Gradio 包）
> - `launch()` = 启动（自动开浏览器）

---

## �� 附录：常见问题答疑

### Q1：Gradio 是不是生成页面就是 UI 页面？

**答：是的**，Gradio 生成的就是 **UI 页面**——但比普通网页有 3 个区别：

#### 1️⃣ 它生成的"是什么页面"

```
gr.Interface(fn=...).launch()
       ↓
自动打开浏览器，访问 http://127.0.0.1:7860
       ↓
你看到的是这样的页面：
   ┌─────────────────────────────┐
   │  �� 你的标题                  │
   │  ┌──────────┐  ┌──────────┐ │
   │  │ 输入框 #1 │  │ 输出框   │ │
   │  └──────────┘  └──────────┘ │
   │       [Submit 按钮]          │
   └─────────────────────────────┘
       ↑
   用户在浏览器里输入 → 浏览器调你的 Python 函数 → 显示结果
```

**它就是 UI**——但**是自动生成的 UI**，不是手写 HTML/CSS/JS 那种。

#### 2️⃣ 跟"普通 UI"的 3 个区别

| 区别 | Gradio | 普通前端（Vue/React） |
| --- | --- | --- |
| **写 UI 的方式** | 写 Python 函数 + `gr.Textbox()` 等 | 手写 HTML/CSS/JS |
| **后端** | **同一个 Python 进程** | 单独的 Node.js / Java 服务 |
| **速度** | 3 行出界面 | 几十~几百行起步 |

**用 Vue 写一个同样的聊天页面**至少需要：

- `index.html`（HTML 结构）
- `style.css`（样式）
- `main.js`（逻辑）
- `package.json` + `npm install` 包管理
- **Node.js 服务器**

**用 Gradio 写**：

```python
import gradio as gr

def chat(message, history):
    return f"你说的是：{message}"

gr.ChatInterface(chat).launch()   # ← 完事
```

**3 行 = 一个完整的可聊天网页**。

#### 3️⃣ Gradio 生成的 UI 是什么风格

**4 种风格，对应 4 个类**：

| 类 | 风格 | 适合 |
| --- | --- | --- |
| `gr.Interface` | 经典"左输入 → 右输出" | 简单工具（翻译、识别） |
| `gr.ChatInterface` | **聊天界面**（用户/AI 气泡） | ✅ AI 对话（本项目） |
| `gr.Blocks` | **自由布局**（自己摆位置） | 复杂页面（多组件） |
| `gr.Tabs` | 选项卡 | 多功能应用 |

**本项目用的是 `gr.Blocks + gr.Chatbot` 自己拼**，所以它长这样：

```
   ┌─────────────────────────────────┐
   │      携程 AI 智能助手             │   ← gr.Label 顶部标题
   ├─────────────────────────────────┤
   │  ┌───────────────────────────┐  │
   │  │ 用户：你好                 │  │
   │  │ AI：你好呀！我是...        │  │   ← gr.Chatbot 聊天显示区
   │  └───────────────────────────┘  │
   ├─────────────────────────────────┤
   │  请输入你的问题：                │   ← gr.Textbox 输入框
   │  [___________________] [回车]  │
   └─────────────────────────────────┘
```

#### 4️⃣ 它跟你说的"普通 UI 页面"的对比

**如果你说的"普通 UI 页面"指的是 HTML 写的：**

```
Gradio 生成的页面 = 普通 HTML 页面（只是它帮你自动生成）
                          ↑
                  你不用写 HTML 也能得到
```

打开浏览器**右键 → 查看网页源代码**，你会看到 Gradio 生成的页面的确是一堆 `<div>` / `<button>` / `<script>`——**它是真 HTML**，只是**生成方式自动**了。

#### 5️⃣ 一句话总结

> **Gradio = UI 生成器**：
> - 你写 Python 函数
> - Gradio 自动生成对应的 HTML 网页（前端）
> - 同时也帮你处理"用户在网页输入 → 调 Python 函数 → 把结果显示到网页上"这条链路
> - 所以 Gradio 是**"Python 写 UI 页面"的最快路径**——3 行出一个完整网页 ✓

> **对比记忆**：
> - **FastAPI** = 后端 API（给小程序/App 调）
> - **Gradio** = 前端 UI 页面（给浏览器用）
> - **LangGraph** = AI 智能体流程（业务逻辑）
>
> 本项目三者都有：LangGraph 做 AI → FastAPI 提供 HTTP 接口 → Gradio 提供网页聊天界面