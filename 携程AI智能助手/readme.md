# 携程 AI 智能助手 (Ctrip Assistant)

一个基于 **LangGraph 多智能体工作流** + **FastAPI** + **Gradio** 构建的携程旅行 AI 客服助手项目。

用户可以像和真人客服对话一样，向助手提出机票查询、改签、退票、租车、酒店预订、旅行推荐等需求；助手在内部通过"主助理 + 4 个专门子助理"的多智能体协作完成查询、预订、改签等操作，并在涉及"写入/改签"类敏感动作前**强制要求人工确认（human-in-the-loop）**。

---

## 目录

- [项目特性](#项目特性)
- [项目结构](#项目结构)
- [核心流程图](#核心流程图)
- [快速开始](#快速开始)
- [配置说明](#配置说明)
- [API 接口文档](#api-接口文档)
- [工作流（LangGraph）详解](#工作流langgraph-详解)
  - [状态（State）](#状态state)
  - [工具（Tools）](#工具tools)
  - [主助理 vs 子助理](#主助理-vs-子助理)
  - [子图构建](#子图构建)
  - [敏感工具的人工审批](#敏感工具的人工审批)
- [Gradio WebUI](#gradio-webui)
- [FastAPI 服务](#fastapi-服务)
- [认证 / 授权](#认证--授权)
- [常见问题](#常见问题)
- [附录：文件索引](#附录文件索引)

---

## 项目特性

- ✅ 基于 **LangGraph** 的多智能体（Multi-Agent）工作流，支持"主助理委派 + 子助理执行"
- ✅ 4 大业务子助理：航班更新/取消、租车预订、酒店预订、旅行推荐
- ✅ 内置 **Tavily 网络搜索** + **公司政策 RAG 检索**（基于 `order_faq.md` 向量化）
- ✅ **人工确认机制（Human-in-the-loop）**：所有"改/删"类敏感工具执行前必须用户输入 `y` 确认
- ✅ 工具分层：每个子助理都分为 `safe_tools`（只读）与 `sensitive_tools`（写操作）两类节点
- ✅ 双前端：**Gradio** 调试页面 + **FastAPI** 接口对外提供服务（含 Swagger）
- ✅ JWT 鉴权、白名单中间件、CORS 全局跨域、统一异常处理
- ✅ SQLite 模拟数据 + pandas 自动对齐"当前时间"，方便演示

---

## 项目结构

```
ctrip_assistant/
├── main.py                         # FastAPI 入口
├── config/                         # 配置（日志、YAML）
│   ├── __init__.py                 # Dynaconf 配置加载
│   ├── development.yml             # 开发配置
│   ├── log_config.py               # 日志 dictConfig
│   └── production.yml              # 生产配置（当前未启用）
├── api/                            # FastAPI 路由层
│   ├── __init__.py
│   ├── routers.py                  # 总路由聚合
│   ├── schemas.py                  # 通用 Pydantic Schema（InDBMixin + 泛型）
│   ├── system_mgt/                 # 用户管理模块
│   │   ├── user_schemas.py
│   │   └── user_views.py
│   └── graph_api/                  # 工作流调用接口
│       ├── graph_schemas.py
│       └── graph_views.py
├── db/                             # 数据库 ORM 层
│   ├── __init__.py                 # engine / sessionmaker / DBModelBase
│   ├── dao.py                      # 泛型 BaseDAO（增删改查）
│   └── system_mgt/
│       ├── models.py               # UserModel
│       └── user_dao.py
├── graph_chat/                     # LangGraph 工作流核心
│   ├── assistant.py                # 主助理（CtripAssistant 类 + Runnable）
│   ├── agent_assistant.py          # 4 个子助理的 Runnable
│   ├── base_data_model.py          # 子助理跳转 / CompleteOrEscalate 的 Pydantic 模型
│   ├── build_child_graph.py        # 4 个子图构建函数
│   ├── entry_node.py               # 子助理入口节点工厂
│   ├── state.py                    # State TypedDict + 对话栈
│   ├── finally_graph.py            # 最终工作流组装（编译 graph）
│   ├── graph_gradio.py             # Gradio 入口（带前端 UI）
│   ├── draw_png.py                 # 工作流图 PNG 导出
│   ├── log_utils.py                # loguru 日志
│   ├── llm_tavily.py               # LLM（ChatOpenAI）+ Tavily 工具
│   ├── 第一个流程图.py               # 早期迭代：单助理 + 中断
│   ├── 第二个流程图.py               # 中期迭代：单助理 + 安全/敏感分流
│   ├── 第三个流程图.py               # 中期迭代：主助理 + 4 子助理
│   └── graph1/2/3/8.*              # 工作流流程图 PNG/JPG
├── tools/                          # LangChain Tools 工具集
│   ├── __init__.py                 # 数据库路径
│   ├── flights_tools.py            # 航班相关工具
│   ├── hotels_tools.py             # 酒店工具
│   ├── car_tools.py                # 租车工具
│   ├── trip_tools.py               # 旅行推荐工具
│   ├── retriever_vector.py         # 公司政策 RAG 检索
│   ├── location_trans.py           # 中英文城市名映射
│   ├── tools_handler.py            # ToolNode + 异常 fallback
│   └── init_db.py                  # 数据库"时间对齐"脚本
├── utils/                          # FastAPI 通用工具
│   ├── cors.py                     # CORS 跨域
│   ├── middlewares.py              # JWT 校验中间件
│   ├── jwt_utils.py                # JWT 编解码
│   ├── password_hash.py            # bcrypt 密码哈希
│   ├── dependencies.py             # DB session 依赖
│   ├── handler_error.py            # 全局异常处理
│   └── docs_oauth2.py              # Swagger OAuth2 适配
├── static/                         # 静态资源（头像等）
├── order_faq.md                    # 公司政策原始文档（RAG 语料）
├── travel_new.sqlite               # 当前使用的数据库
└── travel2.sqlite                  # 数据库备份（用于重置）
```

---

## 核心流程图

- 早期流程图 1：[graph1.png](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/graph1.png)
- 早期流程图 2：[graph2.png](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/graph2.png)
- 早期流程图 3：[graph3.png](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/graph3.png)
- 最终多代理流程图：[graph8.jpg](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/graph8.jpg)

工作流顶层走向：

```
START → fetch_user_info → route_to_workflow ─┐
                                              ├─→ primary_assistant
                                              │     ├─ tools_condition: 主助理工具 → 主助理
                                              │     ├─ ToFlightBookingAssistant  → enter_update_flight
                                              │     ├─ ToBookCarRental           → enter_book_car_rental
                                              │     ├─ ToHotelBookingAssistant   → enter_book_hotel
                                              │     └─ ToBookExcursion           → enter_book_excursion
                                              │
                                              └─ 子助理内：safe_tools / sensitive_tools（敏感工具前 interrupt_before）
                                                          → 子助理节点 → 循环 / leave_skill → primary_assistant
```

---

## 快速开始

### 1. 安装依赖

```bash
pip install fastapi uvicorn sqlalchemy pydantic python-jose[cryptography] passlib[bcrypt] \
            langchain langchain-openai langchain-community langgraph tavily-python pandas
```

### 2. 配置 LLM

修改 [graph_chat/llm_tavily.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/llm_tavily.py) 中 `llm = ChatOpenAI(...)` 的 `openai_api_base` / `openai_api_key`（默认指向本地 `http://localhost:6006/v1` 的 Qwen-7B），同时检查 `TAVILY_API_KEY`。

### 3. 准备数据库

首次运行前可以调用 `tools/init_db.py` 让航班时间与"现在"对齐：

```bash
python -m tools.init_db
```

### 4. 启动 Gradio WebUI（推荐调试）

```bash
python -m graph_chat.graph_gradio
```

浏览器打开 `http://127.0.0.1:7860` 即可对话（默认 `passenger_id="3442 587242"`）。

### 5. 启动 FastAPI 服务

```bash
python main.py
```

访问 Swagger 文档：`http://127.0.0.1:8000/docs`，先用 `/api/login/` 或 `/api/auth/` 获取 Token，再调用 `/api/graph/`。

---

## 配置说明

所有配置位于 [config/development.yml](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/config/development.yml)，通过 [config/__init__.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/config/__init__.py) 中的 Dynaconf 加载：

| 配置项 | 含义 | 默认值 |
| --- | --- | --- |
| `LOG_LEVEL` | 全局日志级别 | `INFO` |
| `HOST` / `PORT` | 服务监听地址 | `127.0.0.1` / `8000` |
| `ORIGINS` | CORS 允许的前端源 | 见 yml |
| `DATABASE` | MySQL 配置（当前主要服务用的是 SQLite） | 见 yml |
| `JWT_SECRET_KEY` | JWT 签名密钥 | 见 yml |
| `ALGORITHM` | JWT 算法 | `HS256` |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | Token 过期时间 | `30` |
| `WHITE_LIST` | 无需鉴权的接口路径正则 | 见 yml |
| `DEFAULT_PASSWORD` | 新建用户默认密码 | `123123` |

可通过环境变量覆盖：

- `EMP_CONF_*`（前缀）：覆盖同名配置项
- `EMP_ENV=production`：启用 `production.yml`

---

## API 接口文档

服务启动后可在 `/docs` 查看完整 Swagger 文档。入口聚合见 [api/routers.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/api/routers.py)。

### 1. 用户管理（[api/system_mgt/user_views.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇\09_携程AI智能助手项目\00_课程资料\ctrip_assistant+webui+fastapi代码\ctrip_assistant\api\system_mgt\user_views.py)）

| 方法 | 路径 | 说明 | 鉴权 |
| --- | --- | --- | --- |
| GET | `/api/users/getUsers/` | 查询所有用户（仅返回 username + id） | ✅ |
| GET | `/api/users/{pk}/` | 根据主键查询用户 | ✅ |
| POST | `/api/register/` | 用户注册（密码可选，默认 `123123`） | ❌（白名单） |
| POST | `/api/login/` | 用户登录，返回 JWT | ❌（白名单） |
| POST | `/api/auth/` | Swagger 表单 OAuth2 提交入口 | ❌（白名单） |
| PATCH | `/api/users/{pk}/` | 修改用户 | ✅ |
| POST | `/api/users/delete/` | 批量删除用户 | ✅ |

> 涉及数据模型：[user_schemas.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/api/system_mgt/user_schemas.py)

### 2. 工作流调用（[api/graph_api/graph_views.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/api/graph_api/graph_views.py)）

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/api/graph/` | 调用 LangGraph 工作流，返回 AI 最后一条回复 |

请求体（[BaseGraphSchema](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/api/graph_api/graph_schemas.py#L21-L24)）：

```json
{
  "user_input": "帮我预订苏黎世的一家酒店",
  "config": {
    "configurable": {
      "passenger_id": "3442 587242",
      "thread_id": "uuid-xxx"
    }
  }
}
```

特殊交互：

- 输入 `y` 表示**确认**当前敏感操作（继续 `graph.stream(None, config)`）。
- 其他输入会被当作新的用户提问，喂给主助理。

---

## 工作流（LangGraph）详解

### 状态（State）

定义：[graph_chat/state.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/state.py)

```python
class State(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]                   # LangGraph 消息列表
    user_info: str                                                       # 通过 fetch_user_info 注入
    dialog_state: Annotated[list[Literal[...]], update_dialog_stack]      # 栈：assistant/update_flight/...
```

`update_dialog_stack` 用于维护"当前正在哪个子助理"——主助理调用 `pop` 时弹出栈，子助理进入时压栈。

### 工具（Tools）

所有工具均通过 `@tool` 装饰器注册，参数描述由 LLM 读取：

| 模块 | 工具 |
| --- | --- |
| [flights_tools.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/tools/flights_tools.py) | `fetch_user_flight_information`、`search_flights`、`update_ticket_to_new_flight`、`cancel_ticket` |
| [hotels_tools.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/tools/hotels_tools.py) | `search_hotels`、`book_hotel`、`update_hotel`、`cancel_hotel` |
| [car_tools.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/tools/car_tools.py) | `search_car_rentals`、`book_car_rental`、`update_car_rental`、`cancel_car_rental` |
| [trip_tools.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/tools/trip_tools.py) | `search_trip_recommendations`、`book_excursion`、`update_excursion`、`cancel_excursion` |
| [retriever_vector.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/tools/retriever_vector.py) | `lookup_policy`（基于 `order_faq.md` 的本地向量检索） |

通用工具节点包装：[tools_handler.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/tools/tools_handler.py) 中 `create_tool_node_with_fallback` 在工具抛错时回填 `ToolMessage` 错误消息，避免崩溃。

### 主助理 vs 子助理

- **主助理**：[assistant.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/assistant.py) 中 `assistant_runnable`
  - 工具：`tavily_tool`、`search_flights`、`lookup_policy` + 4 个 `To*Assistant` 跳转工具
  - 只负责：查询、检索、对话管理；**不做任何写操作**
- **子助理**：[agent_assistant.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/agent_assistant.py) 中 4 个 `*_runnable`
  - 各自绑定对应业务工具集 + `CompleteOrEscalate`
  - 当不需要当前任务时调用 `CompleteOrEscalate` 退回主助理

通用执行包装类：[CtripAssistant](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/assistant.py#L21-L56)，对空响应自动追加 "请提供一个真实的输出作为回应" 提示，确保不返回空内容。

### 子图构建

[build_child_graph.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/build_child_graph.py) 提供 4 个 `build_*_graph` 函数，每个子助理的模板：

```
enter_* (create_entry_node) → * (CtripAssistant) 
                              ├─ 条件分支 ─→ *_safe_tools       → *
                              ├─ 条件分支 ─→ *_sensitive_tools   → *
                              └─ CompleteOrEscalate → leave_skill → primary_assistant
```

入口节点工厂 [create_entry_node](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/entry_node.py#L6-L39)：在切换时往 messages 里塞一条 ToolMessage 提醒子助理"你现在是 XXX"，并把 `dialog_state` 压栈。

### 敏感工具的人工审批

[finally_graph.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/finally_graph.py#L107-L115) 编译时：

```python
graph = builder.compile(
    checkpointer=memory,
    interrupt_before=[
        "update_flight_sensitive_tools",
        "book_car_rental_sensitive_tools",
        "book_hotel_sensitive_tools",
        "book_excursion_sensitive_tools",
    ]
)
```

运行到任一 `*_sensitive_tools` 之前，LangGraph 会暂停。前端（Gradio / FastAPI）检测到 `current_state.next` 非空后提示用户：

> "AI助手马上根据你要求，执行相关操作。您是否批准上述操作？输入'y'继续；否则，请说明您请求的更改。"

用户输入 `y` 才会真正调用 `graph.stream(None, config)` 继续，否则新一轮 user 消息被发回主助理。

---

## Gradio WebUI

[graph_chat/graph_gradio.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/graph_gradio.py)

- 使用 `gr.Blocks` 自定义 UI
- 关键函数：
  - `do_graph(user_input, chat_bot)`：把用户消息 push 到 chat history
  - `execute_graph(chat_bot)`：调用 `graph.stream`，抓取最后一条 `AIMessage.content`
- 默认 `passenger_id = "3442 587242"`、`thread_id = uuid.uuid4()`
- 启动：`python -m graph_chat.graph_gradio`

---

## FastAPI 服务

入口：[main.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/main.py)

`Server.run()` 顺序：

1. `init_log()` 加载日志配置（[config/log_config.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/config/log_config.py)）
2. 创建 `FastAPI`，全局依赖 `MyOAuth2PasswordBearer` 让所有接口在 Swagger 中需要鉴权
3. 挂载 `/static` 目录
4. 注册异常处理、CORS、中间件、路由

`MyOAuth2PasswordBearer` ([utils/docs_oauth2.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/utils/docs_oauth2.py)) 重写了 OAuth2，对 `WHITE_LIST` 中的路径直接放空 Token，避免登录接口死循环。

---

## 认证 / 授权

- 密码哈希：[password_hash.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/utils/password_hash.py)（bcrypt）
- Token 编解码：[jwt_utils.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/utils/jwt_utils.py)
- 鉴权中间件：[middlewares.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/utils/middlewares.py)
  - 命中 `WHITE_LIST` 直接放行
  - 解析 `Authorization: Bearer <token>`，校验 `exp`、提取 `sub` 中的 username 写入 `request.state.username`
- ORM：[db/__init__.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/db/__init__.py) + [user_dao.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/db/system_mgt/user_dao.py)
- 全局异常：[handler_error.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/utils/handler_error.py)

---

## 常见问题

**Q1：启动后工具查询不到任何航班？**
请先执行 `python -m tools.init_db` 重置数据库 + 把航班时间推到"现在"。

**Q2：怎么切换 LLM？**
打开 [graph_chat/llm_tavily.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/llm_tavily.py)，把注释里 GLM-4 / GPT-4o / Claude / DeepSeek 任一 `llm = ChatOpenAI(...)` 取消注释，并替换 `api_key` 与 `base_url`。

**Q3：如何扩展新的子助理？**
1. 在 [graph_chat/base_data_model.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/base_data_model.py) 加一个跳转模型（如 `ToNewBusinessAssistant`）。
2. 在 [graph_chat/agent_assistant.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/agent_assistant.py) 仿照模板添加 `*_prompt` 和 `*_runnable`。
3. 在 [graph_chat/build_child_graph.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/build_child_graph.py) 加 `build_*_graph(builder)`。
4. 在 [finally_graph.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/finally_graph.py) 把新子图挂上 + 在 `route_primary_assistant` 增加分支 + 在 `interrupt_before` 中加入敏感工具名。

**Q4：Gradio 与 FastAPI 共用一个 graph 吗？**
是的，二者都引用 [finally_graph.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/finally_graph.py) 中编译出的 `graph` 单例（Gradio 自己有一份拷贝，但底层逻辑完全一致）。

---

## 📚 附录：常见问题汇总（FastAPI 用户登录 + LangGraph 工作流）

> 把本项目里"用户登录检测 + 网络配置 + LangGraph 内部机制"的所有问答整理成册，方便查阅。

### Q1：FastAPI 中怎么管理登录密码和确保用户是同一个人？

> **引用的代码**：[docs/fastapi.md](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/docs/fastapi.md#L1112-L1118)

本项目用了 **3 个机制**来"确保用户是同一个人"：

| 机制 | 作用 | 项目对应文件 |
| --- | --- | --- |
| **1. 密码哈希** | 注册时把密码"加密"存库，登录时再校验 | [utils/password_hash.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/utils/password_hash.py) |
| **2. JWT Token** | 登录成功发个"通行证"，后续请求带它表示"我登录过" | [utils/jwt_utils.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/utils/jwt_utils.py) |
| **3. 中间件校验** | 每个请求来时，框架自动检查"通行证"是否有效 | [utils/middlewares.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/utils/middlewares.py) |

#### 第 1 步：用户注册 —— 密码加密后存数据库

> **引用的代码**：[docs/fastapi.md](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/docs/fastapi.md#L1118-L1144)

```python
from utils.password_hash import get_hashed_password

@router.post('/register/')
def create(obj_in: CreateOrUpdateUserSchema, session: Session = Depends(get_db)):
    if not obj_in.password:
        obj_in.password = str(settings.DEFAULT_PASSWORD)         # 没传密码 → 用默认 123123

    # ✅ 关键：把明文密码加密后再入库
    obj_in.password = get_hashed_password(obj_in.password)

    return _dao.create(session, obj_in)                          # 存进数据库
```

**`get_hashed_password("123456")` 返回什么？**

```
明文：123456
   ↓ bcrypt 加盐 + 多轮哈希
密文：$2b$12$KIXxH8hG8yT9kT8VfN6mPe5Q...（每次都不一样，因为有"盐"）
```

**为什么这么做？** 就算数据库被偷，黑客拿到的也是一堆乱码。

#### 第 2 步：用户登录 —— 验证密码 + 发"通行证"（Token）

> **引用的代码**：[docs/fastapi.md](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/docs/fastapi.md#L1144-L1171)

```python
from utils.password_hash import verify_password
from utils.jwt_utils import create_token

@router.post('/login/')
def login(obj_in: UserLoginSchema, session: Session = Depends(get_db)):
    # 1) 用用户名去数据库找这个用户
    user = _dao.get_user_by_username(session, obj_in.username)
    if not user:
        raise HTTPException(401, "用户不存在")           # 找不到 → 401

    # 2) 拿用户输入的明文密码 + 库里存的密文，对一下
    if not verify_password(obj_in.password, user.password):
        raise HTTPException(401, "密码错误")           # 对不上 → 401

    # 3) 都对 → 生成"通行证"（JWT）返回给前端
    return {
        "id": user.id,
        "username": user.username,
        "phone": user.phone,
        "real_name": user.real_name,
        "token": create_token(str(user.id) + ':' + user.username)
    }
```

#### 第 3 步：用户请求接口 —— 带着"通行证"来

> **引用的代码**：[docs/fastapi.md](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/docs/fastapi.md#L1171-L1182)

**前端拿到 token 后，每次请求都放在 header 里：**

```
GET /api/users/3/
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**这一步不需要接口函数自己处理——中间件自动搞定。**

#### 第 4 步：服务端校验 Token —— 中间件自动跑

> **引用的代码**：[docs/fastapi.md](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/docs/fastapi.md#L1182-L1219)

```python
async def verify_token(request, call_next):
    # 1) 白名单里的接口（登录、Swagger、静态文件）直接放行
    path = request.get('path')
    for p in settings.WHITE_LIST:
        if re.match(p, path):
            return await call_next(request)        # 不查 token

    # 2) 其它接口：从 header 里取 token
    authorization = request.headers.get('Authorization')
    if not authorization:
        return auth_error                          # 没带 → 401

    token = authorization.split(' ')[1]          # 取出真正的 token

    try:
        # 3) 解码 + 校验有效期
        res_dict = jwt.decode(token, JWT_SECRET_KEY, algorithms=[ALGORITHM])

        # 4) 验证通过 → 把 username 塞到 request.state 上
        username = res_dict.get('sub').split(':')[1]
        request.state.username = username          # 后面视图函数能拿到

        # 5) 验证时间
        if datetime.fromtimestamp(res_dict.get('exp')) < datetime.now():
            return auth_error                      # 过期了 → 401

        # 6) 通过！继续往后走（到路由函数）
        return await call_next(request)
    except ExpiredSignatureError:
        return auth_error                          # 过期
```

**关键点**：`request.state.username = username` —— **中间件把"当前用户是谁"塞到 request 上**，视图函数可以读 `request.state.username`。

#### 第 5 步：接口里读"当前用户"

> **引用的代码**：[docs/fastapi.md](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/docs/fastapi.md#L1219-L1230)

```python
@router.post('/graph/')
def execute_graph(request: Request, obj_in: BaseGraphSchema):
    print('登陆之后的用户名： ' + request.state.username)   # ← 直接读
    # ...
```

**所以"确认是同一个人"= `request.state.username == "zhangsan"` 一直成立**（只要 token 没过期）。

#### 完整流程图

> **引用的代码**：[docs/fastapi.md](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/docs/fastapi.md#L1230-L1261)

```
用户第一次：
   注册 POST /register {"username":"zs","password":"123"}
      ↓
   1. Pydantic 校验
   2. get_hashed_password("123") → 密文存到数据库
   ↓
   登录 POST /login {"username":"zs","password":"123"}
      ↓
   1. 数据库查用户
   2. verify_password("123", 密文) → True
   3. create_token("3:zhangsan") → 返回 token
   ↓
   浏览器把 token 存到 localStorage

用户后续每次：
   GET /api/users/3/
   Header: Authorization: Bearer eyJhbG...
      ↓
   中间件 verify_token 自动跑：
   1. 拿 token → jwt.decode
   2. 查有效期 → 没过期
   3. request.state.username = "zhangsan"
      ↓
   视图函数执行（自动拿到 request.state.username）
      ↓
   返回 JSON
```

#### 4 个关键点速记

> **引用的代码**：[docs/fastapi.md](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/docs/fastapi.md#L1261-L1272)

| 关键点 | 一句话 |
| --- | --- |
| **密码存库** | 永远存 `bcrypt 哈希`，**绝不**存明文 |
| **登录成功发什么** | 发一个 **JWT token**（30 分钟过期） |
| **后续请求带什么** | HTTP header `Authorization: Bearer <token>` |
| **服务器怎么认人** | **中间件**自动从 header 解码 → 写到 `request.state.username` |

#### 项目里几个小细节

> **引用的代码**：[docs/fastapi.md](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/docs/fastapi.md#L1272-L1298)

##### ① 双重登录入口

```python
# /api/login/  - 给前端小程序用的（JSON 请求体）
@router.post('/login/')
def login(obj_in: UserLoginSchema, ...): ...

# /api/auth/   - 给 Swagger 文档"Authorize"按钮用的（form 表单）
@router.post('/auth/')
def auth(form_data: OAuth2PasswordRequestForm = Depends(), ...): ...
```

##### ② [main.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/main.py) 里强制"所有接口都要登录"

```python
self.app = FastAPI(dependencies=[Depends(my_oauth2)])
#                                ↑ 把 OAuth2 设成"全局依赖"：所有接口都先过 token 校验
```

白名单（[config/development.yml](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/config/development.yml)）：

```yaml
WHITE_LIST: ['/api/login', '/api/register', '/static', '/docs', '/swagger', '/openapi', '/api/auth']
#           ↑ 登录、注册、Swagger、静态文件不需要 token
```

##### ③ 修改密码时的"漏勺"

```python
# user_dao.py 的 update 里：只更新"前端实际传的字段"
update_data = obj_in.dict(exclude_unset=True)
#                          ↑ 前端没传的字段（比如 password）不会变成默认值去"清空"数据库
```

#### 网络配置（YAML 里的配置项）

> **引用的代码**：[docs/readme.md](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/docs/readme.md#L179-L179) 和 [docs/readme.md](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/docs/readme.md#L204-L204)

```yaml
# config/development.yml
LOG_LEVEL: INFO
HOST: 127.0.0.1          ← 服务器监听哪个 IP
PORT: 8000              ← 服务器监听哪个端口
ORIGINS: ['http://localhost:8080', ...]   ← 允许哪些前端域名跨域访问

DATABASE:                ← 数据库连接配置
  DRIVER: mysql
  HOST: 127.0.0.1
  PORT: 3306
  USERNAME: root
  PASSWORD: 123123
  NAME: test_db4

JWT_SECRET_KEY: 09d25e094faa6ca2556c818166b7a9563b93f7099f6f0f4caa6cf63b88e8d3e7
ALGORITHM: HS256
ACCESS_TOKEN_EXPIRE_MINUTES: 30

WHITE_LIST: ['/api/login', '/api/register', '/static', '/docs', '/swagger', '/openapi', '/api/auth']
```

**这些配置是怎么用上的？**

```python
# main.py
def run(self):
    uvicorn.run(app=self.app, host=settings.HOST, port=settings.PORT)
                            # ↑ 读 yml 里的 HOST 和 PORT
                            # 启动后监听 127.0.0.1:8000
```

```python
# api/routers.py
app.include_router(router_v1(), prefix='/api')
                              # ↑ 路由自动加 /api 前缀
                              # 所以 user_views 里写 '/login/' 实际是 '/api/login/'
```

### Q2：第 3 步"请求带 token"具体在做什么？

> **引用的代码**：[docs/fastapi.md](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/docs/fastapi.md#L1219-L1230)

**这一段在做什么？**

```
   浏览器                            FastAPI 后端
   │                                       │
   │  --- 步骤 1（注册）已经发生过：用户"zs"存在数据库 ---
   │  --- 步骤 2（登录）已经发生过：浏览器拿到了 token ---
   │
   │  现在：用户点击了"获取我的资料"按钮
   │                                       │
   │  GET /api/users/3/                    │
   │  Header: Authorization: Bearer eyJ... │
   ├──────────────────────────────────────►
   │                                       │
   │                  ┌─────────────────┐
   │                  │ verify_token     │  ← 中间件（自动跑）
   │                  │ 中间件自动：     │
   │                  │ 1. 解码 token   │
   │                  │ 2. 查有效期     │
   │                  │ 3. 拿 username  │
   │                  └────────┬────────┘
   │                           │
   │                  request.state.username = "zs"
   │                           │
   │                  ┌────────▼────────┐
   │                  │ 视图函数        │  ← 你的业务代码
   │                  │ execute_graph() │
   │                  │ 读 state.username│
   │                  │ 干活...         │
   │                  └────────┬────────┘
   │                           │
   │   {"id":3,...}             │
   ◄─────────────────────────────┘
```

**关键**：`Authorization: Bearer <token>` 是 token 所在的地方。

`verify_token` 中间件**自动**校验 → 把当前用户名写进 `request.state` → 视图函数就能知道"当前是谁在操作"。

### Q3：State 经历了什么、存了什么东西？

> **引用的代码**：[docs/readme.md](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/docs/readme.md#L245-L245)

[graph_chat/state.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/state.py)：

```python
class State(TypedDict):
    messages: Annotated[list, add_messages]   # 消息列表
    user_info: str                            # 用户信息
    dialog_state: Annotated[list, update_dialog_stack]  # 当前在哪个子助理
```

**State 是个 TypedDict，定义了 3 个字段**——但**初始是空的**。

| 阶段 | `messages` | `user_info` | `dialog_state` |
| --- | --- | --- | --- |
| **初始** | `[用户消息]` | `""` | `[]` |
| ① fetch_user_info | 同上 | `"航班信息..."` | `[]` |
| ② primary_assistant 思考 | +`[AI 调工具消息]` | 同上 | `[]` |
| ③ entry_node 跳转 | +`[ToolMessage 跳转提示]` | 同上 | `["update_flight"]` |
| ④ update_flight 干活 | +`[工具结果, AI 回复]` | 同上 | `["update_flight"]` |
| ⑤ sensitive_tools 改签 | +`[工具结果, AI 确认]` | 同上 | `["update_flight"]` |
| ⑥ leave_skill 弹栈 | 同上 | 同上 | `[]` ← 弹回 |
| **最终** | `[完整对话历史]` | `"航班信息..."` | `[]` |

**一句话总结**：

> **State 是工作流的"白板"**——你不用手动管 state，**每个节点返回 dict**就行，LangGraph 自动合并。
>
> - `messages` 不断**追加**用户/AI/工具消息
> - `user_info` 从空 → 一次性塞进"航班信息"
> - `dialog_state` 从 `[]` → 压栈 `["update_flight"]` → 用完弹回 `[]`

### Q4：state 和 dialog-state 是什么关系？

> **引用的代码**：[docs/readme.md](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/docs/readme.md#L245-L245)

**State 是整个"白板"**，**dialog_state 是白板上的一个小格子**——专门记录"现在 AI 正在和哪个子助理对话"。

| 概念 | 比喻 | 是什么 |
| --- | --- | --- |
| **State** | 整张白板 | 工作流中所有节点共享的"数据包"（含 `messages` + `user_info` + `dialog_state`） |
| **dialog_state** | 白板上的一个小格子 | 专门记录"现在在哪个子代理"（栈） |

**State ⊇ dialog_state**（包含关系）。

[graph_chat/state.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/state.py)：

```python
def update_dialog_stack(left: list[str], right: Optional[str]) -> list[str]:
    """更新对话状态栈"""
    if right is None:
        return left                      # 不变
    if right == "pop":
        return left[:-1]                # 弹栈
    return left + [right]                # 压栈
```

**3 种行为**：

| right 的值 | 行为 | 类比 |
| --- | --- | --- |
| `None` | 保持不变 | 工人路过，但没改变工位 |
| `"pop"` | 弹栈顶 | 下班离开工位 |
| `"update_flight"` | 压栈 | 进入"航班"工位 |

**记忆口诀**：

> - **State = 整个教室**（学生+课程+黑板+课表）
> - **dialog_state = 课表**（"现在在上哪节课"）
> - **State 包含 dialog_state**（小格子在大白板上）

### Q5：主助理和子助理各包含什么？`runnable` 是干什么的？

> **引用的代码**：[docs/readme.md](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/docs/readme.md#L274-L274)

[graph_chat/assistant.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/assistant.py)：

```python
class CtripAssistant:
    def __init__(self, runnable: Runnable):
        self.runnable = runnable

primary_assistant_prompt = ChatPromptTemplate.from_messages([...]).partial(time=datetime.now())
primary_assistant_tools = [tavily_tool, search_flights, lookup_policy]

assistant_runnable = primary_assistant_prompt | llm.bind_tools(
    primary_assistant_tools + [ToFlightBookingAssistant, ToBookCarRental, ToHotelBookingAssistant, ToBookExcursion]
)
```

**主助理的"职责"**：

| 主助理**会** | 主助理**不会** |
| --- | --- |
| ✅ 听懂用户说什么 | ❌ 改签机票 |
| ✅ 决定交给哪个子助理 | ❌ 订酒店 |
| ✅ 查公司政策（只读） | ❌ 取消订单 |
| ✅ 联网搜索 | ❌ 直接改数据库 |

> **总结**：主助理是"调度员"——只负责听、分、查，**不做写操作**。

**`runnable` = "工人技能包"**（人设 + 大脑 + 工具）：

```python
runnable = prompt | llm.bind_tools([tool1, tool2, ...])
```

**4 个子助理的 runnable**（[graph_chat/agent_assistant.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/agent_assistant.py)）：

| runnable | 能干啥 |
| --- | --- |
| `update_flight_runnable` | 改签/退票 |
| `book_hotel_runnable` | 订/改/取消酒店 |
| `book_car_rental_runnable` | 订/改/取消租车 |
| `book_excursion_runnable` | 订/改/取消旅行 |

### Q6：`CtripAssistant` 与 `runnable` / `assistant_runnable` / 4 个子助理 runnable 的关系？

> **引用的代码**：[docs/readme.md](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/docs/readme.md#L274-L274)

```
                    CtripAssistant
                  ┌─────────────────┐
                  │  这是个"包装盒"  │
                  │  __init__(runnable)│ ← 收 1 个工人
                  │  __call__(state)  │ ← LangGraph 调它时干活
                  └────────┬────────┘
                           │ 里面装 1 个 runnable
                           ↓
        ┌──────────────────────────────────────────┐
        │      5 个 runnable（5 个工人）              │
        ├──────────────────────────────────────────┤
        │   assistant_runnable                ← 主助理 │
        │   update_flight_runnable            ← 航班  │
        │   book_hotel_runnable              ← 酒店  │
        │   book_car_rental_runnable          ← 租车  │
        │   book_excursion_runnable          ← 旅行  │
        └──────────────────────────────────────────┘
```

| 关系 | 含义 |
| --- | --- |
| **`CtripAssistant` = 包装盒** | 把任意一个 runnable 包成 LangGraph 节点 |
| **`runnable` = 工人** | 提示词 + 大模型 + 工具的打包 |
| **主助理** = `CtripAssistant(assistant_runnable)` | 装的是"调度员" |
| **4 个子助理** = 4 个 `CtripAssistant(xxx_runnable)` | 装的是 4 个"工程师" |

**所以总共：1 个盒子类 + 5 个工人 + 5 个 LangGraph 节点**。

### Q7：`graph = builder.compile()` 中的 `compile` 是什么意思？

> **引用的代码**：[docs/readme.md](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/docs/readme.md#L377-L377)

[graph_chat/finally_graph.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/finally_graph.py)：

```python
graph = builder.compile(
    checkpointer=memory,
    interrupt_before=[
        "update_flight_sensitive_tools",
        "book_hotel_sensitive_tools",
        "book_car_rental_sensitive_tools",
        "book_excursion_sensitive_tools",
    ]
)
```

**`builder.compile()` = "把图纸变成真实工厂"**——`builder` 只是"图纸"，**跑不起来**；调 `compile()` 后变成能"接收请求、按图执行"的 `graph`。

`compile()` 一口气干 4 件事：

| 步骤 | 干了啥 | 类比 |
| --- | --- | --- |
| ① 校验图结构 | 检查节点间有没有孤立、边连不连通 | 设计院审核图纸 |
| ② 冻结所有节点 | 把"函数引用"和"状态 schema"锁住 | 把图纸装订成册 |
| ③ 装检查点 | 给"暂停/恢复"能力就位 | 工厂装 UPS 备用电源 |
| ④ 装中断规则 | 在敏感节点前自动停下 | 在敏感工位装"暂停按钮" |

**记忆口诀**：

> - `builder` = 图纸
> - `compile()` = 组装
> - `graph` = 能跑的工厂
> - `graph.stream(...)` = 工厂开始生产

### Q8：`stream` 和 `invoke` 有什么区别？

> **引用的代码**：[docs/readme.md](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/docs/readme.md#L377-L377)

| 维度 | `invoke` | `stream` |
| --- | --- | --- |
| **类比** | 看完整电影 | 看直播 |
| **返回** | 一次性返回最终结果 | **边走边返回**，每个节点跑完就吐一次 |
| **何时拿结果** | 等所有节点跑完 | 每个节点跑完立刻拿 |
| **大模型场景** | 不适合 | ✅ **最适合**（一个字一个字蹦出来） |
| **能否中断？** | ❌ 不能 | ✅ 能（敏感工具前停下） |
| **代码量** | 1 行 | 需要 for 循环处理 |

**项目里**用 `stream`——因为要支持"敏感工具前停下来问用户"。

```python
# 用 stream
events = graph.stream(
    {"messages": [("user", "你好")]},
    config,
    stream_mode="values"
)
for event in events:
    # 每次循环 = 一个节点跑完后的 state
    ...
```

**`stream_mode` 的 2 种模式**：

| 模式 | 每次吐的什么 | 适合 |
| --- | --- | --- |
| `"values"` | **完整 state 快照**（默认） | 想看每个节点的完整 state |
| `"updates"` | **只吐这一节点的输出**（增量） | 想精确知道"谁在改 state" |

**选择指南**：

| 场景 | 用什么 |
| --- | --- |
| **AI 聊天**（一个字一个字蹦） | ✅ stream |
| **敏感操作**（要能停下来问用户） | ✅ stream |
| **批处理**（跑 1000 条数据） | ✅ invoke |
| **单元测试** | ✅ invoke |

**记忆口诀**：

> - `invoke` = 看电影（一次性出结果）
> - `stream` = 看直播（边演边看，可以中途问问题）

---

## 附录：文件索引

| 路径 | 作用 |
| --- | --- |
| [main.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/main.py) | FastAPI 服务入口 |
| [api/routers.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/api/routers.py) | 总路由聚合 |
| [api/schemas.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/api/schemas.py) | 通用 Pydantic 基类与泛型 |
| [api/system_mgt/user_views.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/api/system_mgt/user_views.py) | 用户模块视图 |
| [api/system_mgt/user_schemas.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/api/system_mgt/user_schemas.py) | 用户模块 Schema |
| [api/graph_api/graph_views.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/api/graph_api/graph_views.py) | 工作流调用视图 |
| [api/graph_api/graph_schemas.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/api/graph_api/graph_schemas.py) | 工作流调用 Schema |
| [config/__init__.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/config/__init__.py) | Dynaconf 配置加载 |
| [config/development.yml](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/config/development.yml) | 开发配置 |
| [config/log_config.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/config/log_config.py) | 日志配置 |
| [db/__init__.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/db/__init__.py) | SQLAlchemy engine/session |
| [db/dao.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/db/dao.py) | 通用 DAO 泛型类 |
| [db/system_mgt/models.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/db/system_mgt/models.py) | 用户 ORM 模型 |
| [db/system_mgt/user_dao.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/db/system_mgt/user_dao.py) | 用户 DAO |
| [graph_chat/assistant.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/assistant.py) | 主助理 Runnable + CtripAssistant 类 |
| [graph_chat/agent_assistant.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/agent_assistant.py) | 4 个子助理 Runnable |
| [graph_chat/base_data_model.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/base_data_model.py) | 子助理跳转/CompleteOrEscalate 模型 |
| [graph_chat/build_child_graph.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/build_child_graph.py) | 4 个子图构建 |
| [graph_chat/entry_node.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/entry_node.py) | 子助理入口节点工厂 |
| [graph_chat/state.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/state.py) | 工作流 State |
| [graph_chat/finally_graph.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/finally_graph.py) | 总工作流编译入口 |
| [graph_chat/graph_gradio.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/graph_gradio.py) | Gradio 前端 |
| [graph_chat/draw_png.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/draw_png.py) | 工作流 PNG 导出 |
| [graph_chat/log_utils.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/log_utils.py) | loguru 日志封装 |
| [graph_chat/llm_tavily.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/llm_tavily.py) | LLM 实例 + Tavily |
| [graph_chat/第一个流程图.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/第一个流程图.py) | 单助理 + interrupt 实验 |
| [graph_chat/第二个流程图.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/第二个流程图.py) | 单助理 + safe/sensitive 分流 |
| [graph_chat/第三个流程图.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/graph_chat/第三个流程图.py) | 主助理 + 4 子助理 |
| [tools/__init__.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/tools/__init__.py) | 数据库路径配置 |
| [tools/flights_tools.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/tools/flights_tools.py) | 航班工具集 |
| [tools/hotels_tools.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/tools/hotels_tools.py) | 酒店工具集 |
| [tools/car_tools.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/tools/car_tools.py) | 租车工具集 |
| [tools/trip_tools.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/tools/trip_tools.py) | 旅行推荐工具集 |
| [tools/retriever_vector.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/tools/retriever_vector.py) | 公司政策 RAG 检索 |
| [tools/location_trans.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/tools/location_trans.py) | 中英文城市名映射 |
| [tools/tools_handler.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/tools/tools_handler.py) | 工具节点 + fallback |
| [tools/init_db.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/tools/init_db.py) | 数据库时间对齐 |
| [utils/cors.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/utils/cors.py) | CORS |
| [utils/middlewares.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/utils/middlewares.py) | JWT 校验中间件 |
| [utils/jwt_utils.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/utils/jwt_utils.py) | JWT 编解码 |
| [utils/password_hash.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/utils/password_hash.py) | bcrypt 密码哈希 |
| [utils/dependencies.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/utils/dependencies.py) | DB session 依赖 |
| [utils/handler_error.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/utils/handler_error.py) | 全局异常处理 |
| [utils/docs_oauth2.py](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/utils/docs_oauth2.py) | Swagger OAuth2 适配 |
| [order_faq.md](file:///d:/BaiduNetdiskDownload/AI大模型工程师/02_应用篇/09_携程AI智能助手项目/00_课程资料/ctrip_assistant+webui+fastapi代码/ctrip_assistant/order_faq.md) | 公司政策 RAG 语料 |

---

> 项目核心亮点：基于 LangGraph 的多智能体协作 + 严格的人工确认机制 + 工具分层（safe / sensitive）+ RAG 政策检索。