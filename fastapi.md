# FastAPI 完全入门指南（小白版）

> 写给"完全没接触过 Web 后端"的程序员。这份文档会**手把手**告诉你：
> - FastAPI 是什么、解决什么问题
> - 它和 Flask / Django 有什么区别
> - 它是怎么"跑起来一个网站"的
> - 前后端是怎么"对接"的（小程序、Web 浏览器、App、第三方）
> - 看懂本项目里所有的 FastAPI 代码

---

## 📌 目录

- [1. FastAPI 是什么？](#1-fastapi-是什么)
- [2. 为什么要用 FastAPI？](#2-为什么要用-fastapi)
- [3. 5 个核心概念](#3-5-个核心概念)
- [4. 10 分钟入门：写一个"记事本"API](#4-10-分钟入门写一个记事本api)
- [5. FastAPI 是怎么"跑起来"的](#5-fastapi-是怎么跑起来)
- [6. 前后端是怎么"对接"的](#6-前后端是怎么对接的)
- [7. FastAPI 的高级特性](#7-fastapi-的高级特性)
- [8. 看懂本项目里的 FastAPI 代码](#8-看懂本项目里的-fastapi-代码)
- [9. 常见概念速查表](#9-常见概念速查表)

---

## 1. FastAPI 是什么？

**一句话**：FastAPI 是一个**用来写"网站后端 API"的 Python 框架**。

它做这一件事：
> 让你用很少的 Python 代码，就能"开一个网站"——前端（小程序、浏览器、App）可以用 HTTP 协议来调你的函数。

**来源**：作者 Sebastián Ramírez（tiangolo），2018 年发布，现在（2026）已经是 Python 后端最流行的框架之一。

### 1.1 一个最直观的对比

**没有 FastAPI**（用 Python 标准库写一个"能访问的接口"）：
```python
# 至少 30 行：要自己处理 HTTP 协议、解析 URL、拼 JSON、设置 Content-Type ...
from http.server import BaseHTTPRequestHandler, HTTPServer
import json

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/hello":
            self.send_response(200)
            self.send_header("Content-Type", "application/json")
            self.end_headers()
            self.wfile.write(json.dumps({"msg": "hi"}).encode())

HTTPServer(("localhost", 8000), Handler).serve_forever()
```

**用 FastAPI**：
```python
from fastapi import FastAPI
app = FastAPI()

@app.get("/hello")
def hello():
    return {"msg": "hi"}

# 启动：uvicorn main:app --reload
```

**3 行，效果一样，FastAPI 还自动给你 Swagger 文档。**

---

## 2. 为什么要用 FastAPI？

| 框架 | 特点 | 适合谁 |
| --- | --- | --- |
| **Django** | 大而全、自带 ORM/Admin/表单 | 做完整的传统网站 |
| **Flask** | 极简、灵活、啥都要自己装 | 想要高度自定义 |
| **FastAPI** | 自动文档、类型提示、高性能 | 写 **API 后端**（不写 HTML 页面） |

**FastAPI 的 4 大优势**：

1. **自动生成 API 文档** —— 你写了函数，Swagger 文档自动出来
2. **基于类型提示** —— 函数的参数类型就是 API 的入参校验
3. **性能极高** —— 底层用 Starlette + uvicorn，跟 Node.js/Go 差不多
4. **异步原生** —— `async def` 直接支持，适合 IO 密集型（调 LLM、查数据库）

---

## 3. 5 个核心概念

| 概念 | 通俗解释 | 代码对应 |
| --- | --- | --- |
| **App** | "店"本身 | `app = FastAPI()` |
| **路由（Router）** | "店里的菜单"——每个 URL 对应一个函数 | `@app.get("/xxx")` |
| **请求（Request）** | 前端发过来的"问题" | `请求体、URL 参数、Header` |
| **响应（Response）** | 后端返回给前端的"答案" | `return {...}` |
| **依赖注入（Depends）** | 调函数前，框架自动给你准备好"原料" | `def fn(x = Depends(get_db))` |

**一句话记忆**：
> FastAPI = **把"URL 路径"和"Python 函数"绑在一起**，函数返回啥，前端就拿到啥。

### 3.1 装饰器：决定"什么 URL 触发什么函数"

```python
@app.get("/hello")            # GET /hello      -> 触发 hello()
def hello():
    return {"msg": "hi"}

@app.post("/user")            # POST /user     -> 触发 create_user()
def create_user():
    return {"ok": True}
```

| 装饰器 | HTTP 方法 | 用途 |
| --- | --- | --- |
| `@app.get(...)` | GET | 查 |
| `@app.post(...)` | POST | 新建 |
| `@app.put(...)` | PUT | 整体更新 |
| `@app.patch(...)` | PATCH | 部分更新 |
| `@app.delete(...)` | DELETE | 删除 |

---

## 4. 10 分钟入门：写一个"记事本"API

我们用 FastAPI 写一个**最完整**的迷你项目：用户的记事本，可以增删改查。

### 4.1 安装

```bash
pip install fastapi uvicorn pydantic
```

| 包 | 作用 |
| --- | --- |
| `fastapi` | 框架本身 |
| `uvicorn` | 真正"跑起来"的服务器（FastAPI 只负责定义接口） |
| `pydantic` | 数据校验（FastAPI 强依赖） |

### 4.2 写代码

```python
# main.py
from fastapi import FastAPI
from pydantic import BaseModel

# 1. 创建"店"
app = FastAPI()

# 2. 定义"数据长什么样"（类似"类"，但带自动校验）
class Note(BaseModel):
    title: str
    content: str

# 3. 准备一个内存"数据库"
notes = []  # 实际项目用 MySQL/PostgreSQL/SQLite

# 4. 写接口

# 4.1 GET /notes —— 查所有
@app.get("/notes")
def list_notes():
    return notes

# 4.2 POST /notes —— 新建一个
@app.post("/notes")
def create_note(note: Note):                 # ← 注意：直接把 Note 当参数！
    notes.append(note)
    return {"id": len(notes) - 1, "msg": "ok"}

# 4.3 GET /notes/{id} —— 查单个
@app.get("/notes/{note_id}")
def get_note(note_id: int):                  # ← 自动从 URL 提取
    return notes[note_id]

# 4.4 DELETE /notes/{id}
@app.delete("/notes/{note_id}")
def delete_note(note_id: int):
    notes.pop(note_id)
    return {"msg": "deleted"}
```

### 4.3 跑起来

```bash
uvicorn main:app --reload
```

- `main` = `main.py` 文件名
- `app` = 文件里 `app = FastAPI()` 这个变量
- `--reload` = 代码改了自动重启（开发用）

跑起来后访问 `http://127.0.0.1:8000/docs` 就能看到自动生成的 **Swagger 文档**，可以在线试接口。

---

## 5. FastAPI 是怎么"跑起来"的

很多人搞不清 **"写完代码"到"网站能访问"** 之间发生了什么。一图讲清楚：

```
你写的 main.py
   │
   │  uvicorn main:app
   ▼
┌──────────────────────────────┐
│        uvicorn（服务器）         │  ← 监听 8000 端口，接收 HTTP 请求
│   ┌──────────────────────┐    │
│   │   FastAPI（路由）      │    │  ← 看 URL，把请求分发给对应函数
│   └─────┬────────────────┘    │
│         │                     │
│   ┌─────▼────────────┐        │
│   │  Pydantic（校验） │        │  ← 检查前端传的数据格式对不对
│   └─────┬────────────┘        │
│         │                     │
│   ┌─────▼────────────┐        │
│   │  你的函数        │        │  ← 真正干活
│   └─────┬────────────┘        │
│         │                     │
│   ┌─────▼────────────┐        │
│   │  Pydantic（序列化）│        │  ← 函数返回的 dict 变 JSON
│   └─────┬────────────┘        │
│         │                     │
│   ┌─────▼────────────┐        │
│   │  uvicorn 返回      │        │  ← 加上 Content-Type 等头，发回给前端
│   └──────────────────┘        │
└──────────────────────────────┘
           │
           ▼
        前端 / 小程序 / App / 浏览器 / 第三方服务器
```

**简单说**：
> uvicorn = 接待员（管网络）
> FastAPI = 服务员（管分单）
> Pydantic = 质检员（管数据）
> 你的函数 = 大厨（管做饭）

### 5.1 启动命令详解

```bash
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

| 参数 | 含义 |
| --- | --- |
| `main` | Python 文件名（不带 .py） |
| `app` | 文件里 FastAPI 实例的变量名 |
| `--host 0.0.0.0` | 监听所有网卡（否则默认只本机能访问） |
| `--port 8000` | 监听 8000 端口 |
| `--reload` | 代码改动自动重启（开发用） |

---

## 6. 前后端是怎么"对接"的

这是你问的**最关键**的问题。**FastAPI 跟小程序 / App / 浏览器 怎么配合？**

### 6.1 答案：**它们都用 HTTP 协议**

不管前端是：
- 浏览器（JS 用 `fetch`）
- 小程序（用 `wx.request`）
- 微信 H5（用 `jQuery.ajax` 或 `axios`）
- App（用 `OkHttp` / `Alamofire`）
- 第三方服务器（用 Python `requests` / Go `http.Client` / curl）

**它们发的请求格式都长一样**：

```
POST /api/login HTTP/1.1
Host: 127.0.0.1:8000
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

{
  "username": "zhangsan",
  "password": "123456"
}
```

后端 FastAPI 收到后：
1. 解析 URL `/api/login` → 找到 `login` 函数
2. 解析请求体 → Pydantic 校验（必须有 username + password）
3. 跑函数 → 返回 dict
4. 序列化成 JSON → 加 Header → 发回去

### 6.2 完整对接示例

#### 6.2.1 浏览器（JavaScript）

```javascript
// 前端 JS 代码
async function login() {
    const response = await fetch('http://127.0.0.1:8000/api/login', {
        method: 'POST',                       // HTTP 方法
        headers: {
            'Content-Type': 'application/json',
            'Authorization': 'Bearer ' + token  // 如果需要登录态
        },
        body: JSON.stringify({                // 请求体
            username: 'zhangsan',
            password: '123456'
        })
    });
    const data = await response.json();      // 解析响应
    console.log(data);
}
```

#### 6.2.2 微信小程序

```javascript
// 小程序的 .js 文件
wx.request({
    url: 'http://127.0.0.1:8000/api/login',  // 后端地址
    method: 'POST',
    header: {
        'Content-Type': 'application/json'
    },
    data: {                                    // 请求体（自动转 JSON）
        username: 'zhangsan',
        password: '123456'
    },
    success: (res) => {                       // 成功回调
        console.log(res.data);                // res.data 就是后端返回的 dict
    }
});
```

> ⚠️ **小程序的坑**：必须去微信公众平台把 `127.0.0.1:8000` 加到"request 合法域名"里（开发期可以勾"不校验合法域名"）。

#### 6.2.3 微信 H5 / 移动端

跟浏览器 JS 一模一样，用 `fetch` / `axios`。

#### 6.2.4 App（Android Kotlin）

```kotlin
val client = OkHttpClient()
val body = "{\"username\":\"zhangsan\",\"password\":\"123456\"}".toRequestBody("application/json".toMediaType())
val request = Request.Builder()
    .url("http://your-server.com/api/login")
    .post(body)
    .build()
client.newCall(request).execute().use { 
    println(it.body?.string())  // 拿到后端返回的 JSON 字符串
}
```

### 6.3 数据格式约定（最常见的"对接协议"）

后端返回的 JSON 一般长这样：

```json
{
    "code": 0,            // 0=成功，其他=失败
    "msg": "ok",
    "data": {              // 真正的业务数据
        "user": {
            "id": 3,
            "username": "zhangsan"
        }
    }
}
```

FastAPI 默认**不强制这个格式**，你直接 `return {"key": "value"}` 就行，前端拿到的就是 `{"key": "value"}`。

如果你想统一格式，参考本项目 [`utils/handler_error.py`](../utils/handler_error.py) 的做法。

### 6.4 跨域问题（CORS）

**问题**：你的网站在 `localhost:8080`，API 在 `localhost:8000`，浏览器会拒绝 JS 调 API（"跨域"）。

**解决**：后端声明"我允许谁访问我"。

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:8080"],   # 允许的源
    allow_credentials=True,                     # 允许带 cookie
    allow_methods=["*"],                        # 允许所有方法
    allow_headers=["*"],                        # 允许所有请求头
)
```

本项目里 [`utils/cors.py`](../utils/cors.py) 就是干这个的。

---

## 7. FastAPI 的高级特性

### 7.1 Pydantic 数据校验

Pydantic 让你**用类来定义"前端应该传什么"**：

```python
from pydantic import BaseModel, Field

class UserIn(BaseModel):
    username: str = Field(..., min_length=3, max_length=20)   # 3-20 字符
    age: int = Field(..., ge=0, le=150)                       # 0-150 之间
    email: str                                                   # 自动按 email 格式校验

@app.post("/user")
def create(user: UserIn):                                       # ← Pydantic 自动校验
    return user
```

如果前端发 `"age": -1`，FastAPI 会**自动**返回 422 错误 + 哪里错了，**不用你写一行校验代码**。

### 7.2 依赖注入（Depends）

**问题**：每个接口都要查数据库，怎么避免重复代码？

```python
# ❌ 重复
@app.get("/a")
def a(session = Session()):           # 每次手动建 session
    ...

@app.get("/b")
def b(session = Session()):
    ...

# ✅ 用 Depends
def get_db():
    db = Session()
    try:
        yield db
    finally:
        db.close()

@app.get("/a")
def a(db = Depends(get_db)):           # 框架自动注入
    ...

@app.get("/b")
def b(db = Depends(get_db)):
    ...
```

**效果**：请求来了 -> 框架自动调 `get_db()` -> 你的函数拿到 db -> 函数跑完 -> 自动 close。

本项目 [`utils/dependencies.py`](../utils/dependencies.py) 就是这套路。

### 7.3 中间件（Middleware）

**中间件 = 在每个请求"路过"的钩子函数**。可以做：
- 鉴权（验证 token）
- 记日志
- 限流
- 修改请求/响应

```python
@app.middleware("http")
async def add_header(request, call_next):
    response = await call_next(request)
    response.headers["X-Powered-By"] = "FastAPI"
    return response
```

本项目 [`utils/middlewares.py`](../utils/middlewares.py) 就是用中间件做 token 校验。

### 7.4 异常处理

```python
from fastapi import HTTPException

@app.get("/user/{user_id}")
def get_user(user_id: int):
    user = db.get(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="用户不存在")  # 自动返回 404
    return user
```

本项目 [`utils/handler_error.py`](../utils/handler_error.py) 用 `app.add_exception_handler(...)` 把所有异常统一成 `{"detail": "..."}` 格式。

### 7.5 后台任务

```python
from fastapi import BackgroundTasks

def send_email(to: str):
    # 耗时的操作
    ...

@app.post("/register")
def register(bg: BackgroundTasks):
    bg.add_task(send_email, "user@example.com")   # 注册完再发邮件，不阻塞
    return {"msg": "ok"}
```

### 7.6 异步

```python
@app.get("/slow")
async def slow():                       # async def
    result = await some_io_call()         # 等 IO 完再继续
    return result
```

适合"要等数据库 / 远程 API / LLM"的场景。

### 7.7 WebSocket

```python
@app.websocket("/ws")
async def ws(websocket: WebSocket):
    await websocket.accept()
    while True:
        data = await websocket.receive_text()
        await websocket.send_text(f"received: {data}")
```

适合"实时聊天、股票行情、游戏"等长连接。

---

## 8. 看懂本项目里的 FastAPI 代码

打开 [main.py](../main.py)：

```python
from fastapi import FastAPI, Depends
from starlette.staticfiles import StaticFiles
from config import settings
from utils import handler_error, cors, middlewares
from config.log_config import init_log
from api import routers
from utils.docs_oauth2 import MyOAuth2PasswordBearer

class Server:
    def __init__(self):
        init_log()
        my_oauth2 = MyOAuth2PasswordBearer(tokenUrl='/api/auth/', schema='JWT')
        self.app = FastAPI(dependencies=[Depends(my_oauth2)])
        self.app.mount('/static', StaticFiles(directory='static'), name='my_static')

    def init_app(self):
        handler_error.init_handler_errors(self.app)
        middlewares.init_middleware(self.app)
        cors.init_cors(self.app)
        routers.init_routers(self.app)

    def run(self):
        self.init_app()
        uvicorn.run(app=self.app, host=settings.HOST, port=settings.PORT)

if __name__ == '__main__':
    Server().run()
```

**用我们刚学的概念解读**：

| 代码 | 含义 |
| --- | --- |
| `init_log()` | 配置日志（看 utils 里的 log_config） |
| `MyOAuth2PasswordBearer(...)` | 让 Swagger 有"Authorize"按钮，且绕开登录死循环 |
| `FastAPI(dependencies=[Depends(my_oauth2)])` | 创建"店"，并要求**所有接口都要登录** |
| `app.mount('/static', ...)` | 把 `static/` 目录暴露成 `http://xxx/static/` |
| `handler_error.init_handler_errors(app)` | 装"统一异常处理" |
| `middlewares.init_middleware(app)` | 装"token 校验中间件" |
| `cors.init_cors(app)` | 装"跨域" |
| `routers.init_routers(app)` | 把"用户管理"和"工作流"的接口全注册进来 |
| `uvicorn.run(...)` | 启动服务器 |

### 8.1 看接口（[`api/system_mgt/user_views.py`](../api/system_mgt/user_views.py)）

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from utils.dependencies import get_db
from api.system_mgt.user_schemas import UserLoginSchema, UserLoginRspSchema

router = APIRouter()

@router.post('/login/', response_model=UserLoginRspSchema)
def login(obj_in: UserLoginSchema, session: Session = Depends(get_db)):
    user = dao.get_user_by_username(session, obj_in.username)
    if not user:
        raise HTTPException(status_code=401, detail="用户不存在")
    if not verify_password(obj_in.password, user.password):
        raise HTTPException(status_code=401, detail="密码错误")
    return {"id": user.id, "username": user.username, "token": create_token(...)}
```

**逐行解读**：

| 行 | 含义 |
| --- | --- |
| `APIRouter()` | 创建一个"分菜单" |
| `@router.post('/login/', ...)` | 挂在 POST /login 下；返回值用 UserLoginRspSchema 校验 |
| `obj_in: UserLoginSchema` | 前端发的 JSON 自动用 UserLoginSchema 校验 |
| `session: Session = Depends(get_db)` | 框架自动注入数据库 session |
| `raise HTTPException(...)` | 抛异常，FastAPI 自动转成 JSON 错误响应 |
| `return {...}` | 函数的返回值就是接口的 JSON 响应 |

### 8.2 看 Pydantic 模型（[`api/system_mgt/user_schemas.py`](../api/system_mgt/user_schemas.py)）

```python
class UserLoginSchema(BaseModel):
    username: str = Field(description='用户名')
    password: str = Field(description='密码')

class UserLoginRspSchema(UserSchema):
    token: str
```

**逐行解读**：

| 行 | 含义 |
| --- | --- |
| `class UserLoginSchema(BaseModel)` | 一个"数据形状"，名字叫 UserLoginSchema |
| `username: str` | 必填字符串字段 |
| `Field(description=...)` | Swagger 文档里会显示这个描述 |
| `class UserLoginRspSchema(UserSchema)` | 继承 UserSchema（所有用户字段）并加 token 字段 |
| `token: str` | 必填字符串字段 |

---

## 9. 常见概念速查表

| 概念 | 含义 | 代码示例 |
| --- | --- | --- |
| `FastAPI()` | 创建应用 | `app = FastAPI()` |
| `@app.get/post/...` | 注册路由 | `@app.get("/xxx")` |
| `BaseModel` | Pydantic 数据模型 | `class X(BaseModel): ...` |
| `Field(...)` | 给字段加元数据 | `name: str = Field(..., min_length=1)` |
| `HTTPException` | 抛 HTTP 错误 | `raise HTTPException(404, "Not Found")` |
| `Depends(...)` | 依赖注入 | `def fn(x = Depends(get_x)): ...` |
| `Query(...)` | URL 查询参数 | `def fn(q: str = Query(...)): ...` |
| `Path(...)` | URL 路径参数 | `def fn(id: int = Path(...)): ...` |
| `Body(...)` | 请求体 | `def fn(data: X = Body(...)): ...` |
| `Header(...)` | 请求头 | `def fn(x: str = Header(...)): ...` |
| `Cookie(...)` | Cookie | `def fn(x: str = Cookie(...)): ...` |
| `BackgroundTasks` | 后台任务 | `bg.add_task(fn, arg)` |
| `UploadFile` | 上传文件 | `def fn(f: UploadFile): ...` |
| `Form(...)` | 表单字段（multipart） | `def fn(name: str = Form(...)): ...` |
| `APIRouter` | 子路由 | `router = APIRouter()` |
| `middleware('http')` | 中间件 | `@app.middleware("http")` |
| `add_exception_handler` | 异常处理 | `app.add_exception_handler(X, fn)` |
| `include_router` | 挂载子路由 | `app.include_router(router, prefix='/api')` |
| `response_model` | 声明响应模型 | `@app.get('/', response_model=X)` |
| `status_code` | 声明状态码 | `@app.post('/', status_code=201)` |
| `tags` | Swagger 分组 | `@app.get('/', tags=['用户管理'])` |
| `uvicorn.run` | 启动服务器 | `uvicorn.run(app, host, port)` |

---

## 🎯 一句话总结

> **FastAPI = 用 Python 函数 + 装饰器写"网站后端 API"**：
> - `@app.get("/url")` 决定"哪个 URL 触发哪个函数"
> - 函数参数 + Pydantic = 自动接收 + 校验前端传的数据
> - 函数返回值 = 自动转成 JSON 发回前端
> - `uvicorn` 启动后，前端（浏览器/小程序/App）用 HTTP 协议调它

**对接任何前端**：
> 浏览器、小程序、App、第三方服务器 都会发 HTTP 请求，
> 你只管把函数写好、返回 dict，剩下的 FastAPI + uvicorn 自动帮你处理。