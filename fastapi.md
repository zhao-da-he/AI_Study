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

## 📌 附录：常见问题答疑
Q1：FastAPI 接口"是不是和写好的代码连用"？
答：可以"连用"，也可以"直接写"。两种都常见。

模式	写法	适用
直接写（记事本案例）	业务代码和接口代码放在同一个文件	教学 / 小项目
分文件（本项目）	业务代码在 db/、接口代码在 api/	真实项目
Python



# ====== 直接写（教学用）======
@app.get("/hello")
def hello():
    return {"msg": "hi"}

# ====== 分文件（真实项目）======
# db/system_mgt/user_dao.py
def get_user_by_username(session, username):
    stmt = select(UserModel).where(UserModel.username == username)
    return session.execute(stmt).scalars().first()

# api/system_mgt/user_views.py
@router.post('/login/')
def login(obj_in, session: Session = Depends(get_db)):
    user = _dao.get_user_by_username(session, obj_in.username)  # 调 DAO
    ...
Q2：Pydantic 我没看到定义 / 调用？
Python



from pydantic import BaseModel              # ← 导入

class Note(BaseModel):                       # ← 继承 BaseModel
    title: str
    content: str
只是用 class X(BaseModel)，就告诉 Pydantic "我要校验这种数据"——前端发的字段类型不对，FastAPI 自动返回 422。

前端发来	Pydantic 校验
{"title": "今天", "content": "笔记"}	✅ 通过
{"title": 123}	❌ 自动返回 422 错误
app = FastAPI() 本身不会校验，只有"接口的参数类型是 BaseModel 子类"时 Pydantic 才会启动。所以"看不见 Pydantic 在跑"是正常的——它是后台默默干活。

Q3：app = FastAPI() 这个"店"是什么意思？
一句话：app 就是一个"装接口的容器"。

比喻	代码	含义
开一家空店	app = FastAPI()	创建一个 Web 应用对象
写菜单（菜名）	@app.get("/xxx")	注册一个 URL 路由
写做法	def xxx(): return ...	处理这个 URL 的逻辑
把店开张	uvicorn main:app	启动服务器监听请求
app 就像一个"空菜单本"，后面所有的 @app.get / @app.post 都是往这本菜单上贴菜。

Q4：Pydantic 定义数据长什么样，实际项目也只写类型就行吗？
答：对，实际项目就只写"类型 + 简单约束"。

Python



class CreateUserReq(BaseModel):
    username: str = Field(..., min_length=3, max_length=20)
    email: str = Field(...)
    age: int = Field(..., ge=0, le=150)
约束	作用	例子
min_length / max_length	字符串长度	min_length=6（密码至少 6 位）
ge / le / gt / lt	数字大小	ge=0, le=150（年龄 0-150）
regex	正则	regex=r"^1[3-9]\d{9}$"（手机号）
default	默认值	default=False
项目里 Pydantic 主要干两件事：① 前端发来数据时校验 → 不对就 422；② 返回数据时序列化 + 自动加示例到 Swagger。

Q5：notes = [] 实际项目用 MySQL 怎么用？
答：MySQL 确实是外部数据库，FastAPI 用 SQLAlchemy 跟它对话。

Python



# 1) 安装驱动：pip install sqlalchemy pymysql
DATABASE_URL = "mysql+pymysql://root:123123@127.0.0.1:3306/test_db4?charset=utf8mb4"

# 2) 创建 engine（连接池）
engine = create_engine(DATABASE_URL, pool_size=10)

# 3) 写 ORM 模型（每个 class = 一张表）
class Note(Base):
    __tablename__ = "t_note"
    id = Column(Integer, primary_key=True)
    title = Column(String(200))
    content = Column(Text)

# 4) 接口里用 session 操作数据库
@app.post("/notes")
def create_note(note: NoteIn, session: Session = Depends(get_db)):
    db_note = Note(title=note.title, content=note.content)
    session.add(db_note)
    session.commit()
    return {"id": db_note.id, "msg": "ok"}
MySQL 是另一个进程/服务器，FastAPI 通过 SQLAlchemy 这个"翻译官"跟它通信：




FastAPI（你的代码） 
   ↓ 用 SQLAlchemy 发 SQL
MySQL 服务器（另一个进程）
   ↓ 返回数据
SQLAlchemy 把"行"变成"对象"
   ↓
你的函数拿到 ORM 对象
本项目已经在用 MySQL 了：

db/__init__.py 里 create_engine(url, ...) 就是连 MySQL
db/system_mgt/models.py 里的 UserModel 就是表
接口里 _dao.create(session, obj_in) 就是写数据库
Q6：第 4 步"写接口"是在做什么？
答：每个 @app.xxx 就是"在菜单上加一道菜"。

Python



@app.get("/notes")                 # 加一道"查所有"的菜
def list_notes():
    return notes                   # 菜的内容 = "返回 notes 列表"
4 步加 4 道菜：

函数	菜名	客户点完这道菜后端返回什么
list_notes	GET /notes（查所有）	整个 notes 列表
create_note	POST /notes（新建）	{"id": ..., "msg": "ok"}
get_note	GET /notes/{id}（查单个）	notes[id] 那一条
delete_note	DELETE /notes/{id}（删除）	{"msg": "deleted"}
你看到的"几乎没做什么"是因为：这是个最简版，只演示"接口长什么样"。真实项目里函数里会做校验权限、查/改数据库、调其它服务等。

Q7：uvicorn main:app --reload 实际项目也是这样启动吗？
命令	场景
uvicorn main:app --reload	开发用（你写代码，保存就自动重启）
uvicorn main:app --host 0.0.0.0 --port 8000	测试 / 演示
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4	生产环境（开 4 个进程扛并发）
gunicorn main:app -w 4 -k uvicorn.workers.UvicornWorker	生产环境（更专业的进程管理）
参数：main = Python 文件名；app = FastAPI 实例变量名；--reload = 自动重启（开发用）；--host 0.0.0.0 = 监听所有网卡；--port 8000 = 监听 8000 端口。

Q8：engine 和 session 到底是什么关系？
答：engine 是"连接池"，sessionmaker 是"借窗口"，session 才是你真正用的"毛巾"。

Python



engine = create_engine(DATABASE_URL)        # ← 第 3 步：建连接池
session = Session(bind=engine)               # ← 第 5 步（注意：bind=engine）
生活类比：




engine  = "游泳池"
             ↓
session = "从池子里借出来的一条毛巾"
engine 是个"大池子"——它管的是"怎么连数据库、连哪个数据库、最多能开多少个连接"。
session 是个"小毛巾"——你每来一个请求，框架就从 engine 里借给你一条 session，用完归还。
所以 engine 不是"被 session 调用"——它是被 **sessionmaker 拿去做"模板"**用的。

本项目里（db/__init__.py）：

Python



engine = create_engine(url, echo=True, future=True, pool_size=10)
sm = sessionmaker(bind=engine, autoflush=True, autocommit=False)
sessionmaker 已经"绑"了 engine，所以后面 sm() 不用再传 engine：

Python



session = sm()             # ← 不需要再写 bind=engine
Q9：UserModel 用到 engine 了吗？
没有。 UserModel 只是定义"表长什么样"，不需要连接数据库。它的工作是：

Python



class UserModel(DBModelBase):
    """用户表（图纸）"""
    __tablename__ = "t_usermodel"
    username: Mapped[str] = mapped_column(String(20), unique=True, nullable=False)
    password: Mapped[str] = mapped_column(String(200), nullable=False)
这只是个"图纸"。它告诉 SQLAlchemy：

"我这张表叫 t_usermodel，有 username/password/phone/email/real_name/icon 这些字段。"

真的去数据库创建/查这张表，要用 session：

Python



# 创建一条记录
session.add(UserModel(username="zhangsan", password="123456", ...))
session.commit()

# 查询
user = session.query(UserModel).filter_by(username="zhangsan").first()
所以链路是：

角色	谁做	什么时候
engine	程序启动时建一次	全程就 1 个
sessionmaker（sm）	程序启动时建一次	全程就 1 个
session	每个请求通过 get_db() 借 1 条	每个请求 1 条
UserModel / Note	只是个"表结构图纸"	不参与连接
Q10：get_db()、login()、_dao.xxx() 各干什么？
它们是3 个完全不同的角色——一个送快递、一个点外卖、一个在后厨炒菜。

角色	谁	干什么	类比
get_db()	utils/dependencies.py	"借一条 session 给你"	快递员，把数据库连接送到你手上
login()	api/system_mgt/user_views.py（@router.post 装饰的）	接收前端请求、调 DAO、返回响应	餐厅服务员，接单 + 上菜
_dao.xxx()	db/system_mgt/user_dao.py	真的去查/写数据库	后厨，知道"这道菜具体怎么炒"
完整链路：

Python



# ========== 1) 后厨（db/system_mgt/user_dao.py）==========
class UserDao(BaseDAO):
    model = UserModel
    def get_user_by_username(self, session, username):
        stmt = select(self.model).where(self.model.username == username)
        return session.execute(stmt).scalars().first()

# ========== 2) 快递员（utils/dependencies.py）==========
def get_db():
    session = sm()        # 借一条
    yield session          # 送出去
    session.close()        # 用完还

# ========== 3) 服务员（api/system_mgt/user_views.py）==========
@router.post('/login/')
def login(obj_in: UserLoginSchema,                  # 接单
          session: Session = Depends(get_db)):     # 找快递员要 session
    user = _dao.get_user_by_username(session, obj_in.username)  # 找后厨炒菜
    if not user:
        raise HTTPException(401, "用户不存在")
    return {"token": create_token(...)}              # 菜做好，端给前端
3 者的关系 = 顾客 → 服务员 → 后厨。

Q11：是不是所有"查/改"都要通过 DAO？怎么改的也要写在 DAO 里？
答：是的，所有"碰数据库"的 SQL 都写在 DAO 里，接口不直接写 SQL。

通用 DAO 基类（db/dao.py） 已经写好了：

Python



class BaseDAO(Generic[ModelType, ...]):
    model = ModelType   # 子类指定具体表

    def get(self, session):                     # 查所有
        return session.scalars(select(self.model)).all()

    def get_by_id(self, session, pk):            # 按主键查
        return session.get(self.model, pk)

    def create(self, session, obj_in):           # 新增
        obj = self.model(**jsonable_encoder(obj_in))
        session.add(obj)
        session.commit()
        return obj

    def update(self, session, pk, obj_in):       # 修改
        obj = self.get_by_id(session, pk)
        for key, val in obj_in.dict(exclude_unset=True).items():
            setattr(obj, key, val)
        session.add(obj)
        session.commit()
        return obj

    def delete(self, session, pk):                # 删除
        obj = self.get_by_id(session, pk)
        session.delete(obj)
        session.commit()

    def deletes(self, session, ids):              # 批量删除
        stmt = delete(self.model).where(self.model.id.in_(ids))
        session.execute(stmt)
        session.commit()
所以"怎么改"都已经写好了——你只需要：

Python



_dao.update(session, pk=5, obj_in=UserUpdateSchema(name="新名字"))
_dao.delete(session, pk=5)
接口里就不写 SQL 了：

Python



@router.patch('/users/{pk}/')
def update_user(pk: int, user: UserUpdateSchema, session: Session = Depends(get_db)):
    return _dao.update(session, pk, user)
如果非要"自定义改法"怎么办？ 在 DAO 里加新方法：

Python



class UserDao(BaseDAO):
    def reset_password(self, session, user_id, new_password):
        """自定义方法：重置密码"""
        user = self.get_by_id(session, user_id)
        user.password = get_hashed_password(new_password)
        session.commit()
        return user
Q12：DAO 层和接口层的分工是什么？为什么封装在 DAO 里？
12.1 分工对比
关注点	接口层（api/.../user_views.py）	DAO 层（db/.../user_dao.py）
干什么	接单、上菜	备料、炒菜
关心	收什么数据、返回什么状态码、调哪个 DAO 方法	SQL 怎么写、表怎么查、事务怎么 commit
典型内容	@router.post()、参数校验、查 token、抛 HTTPException	session.query/add/commit、拼 SQL
出现频率	每个接口一个	多个接口共用
12.2 一个具体例子看清分工
Python



# ========= 接口层（服务员）=========
@router.post('/register/')
def create(obj_in: CreateOrUpdateUserSchema, session: Session = Depends(get_db)):
    # 1) 业务逻辑（加密密码、给默认值）
    if not obj_in.password:
        obj_in.password = str(settings.DEFAULT_PASSWORD)
    obj_in.password = get_hashed_password(obj_in.password)

    # 2) 调 DAO 写库
    return _dao.create(session, obj_in)

# ========= DAO 层（后厨）==========
class UserDao(BaseDAO[UserModel, ...]):
    def create(self, session, obj_in):
        """只负责'把这条记录写进数据库'"""
        obj = self.model(**jsonable_encoder(obj_in))
        session.add(obj)
        session.commit()
        return obj
你看到没：

接口层里 1 行 SQL 都没有，只关心"业务"（要不要默认密码、要不要加密）
DAO 层里 1 个 HTTPException 都没有，只关心"怎么写库"
12.3 为什么封装在 DAO 里？3 大理由
理由	没 DAO 的世界	有 DAO 的世界
1. 复用	改密码要 SQL，新建用户也要 SQL，复制 N 遍	写 1 次 DAO.create()，所有接口都能调
2. 测试方便	接口直接调数据库，单元测试很难	接口只调 DAO，测试时 mock 掉 DAO 就行
3. 改库方便	从 MySQL 换 PostgreSQL，要改 N 个接口	只改 DAO 那一层就行
12.4 一张图看清整个项目分层



┌─────────────────────────────────────┐
│  前端（浏览器/小程序）                │
└───────────────┬─────────────────────┘
                │ HTTP + JSON
                ▼
┌─────────────────────────────────────┐
│  接口层 api/.../user_views.py        │  ← 服务员：接单、上菜
│  - @router.post/get/...             │
│  - Pydantic 校验                     │
│  - 调 DAO                            │
│  - 返回 JSON                         │
└───────────────┬─────────────────────┘
                │ 调函数
                ▼
┌─────────────────────────────────────┐
│  DAO 层 db/.../user_dao.py            │  ← 后厨：备料、炒菜
│  - session.query/add/commit          │
│  - SQLAlchemy ORM 操作               │
│  - 通用 CRUD + 业务特殊方法          │
└───────────────┬─────────────────────┘
                │ ORM 翻译成 SQL
                ▼
┌─────────────────────────────────────┐
│  数据库 MySQL / SQLite                │  ← 仓库
└─────────────────────────────────────┘
Q13：FastAPI 中怎么管理登录密码和确保用户是同一个人？
本项目用了 3 个机制来"确保用户是同一个人"：

机制	作用	项目对应文件
1. 密码哈希	注册时把密码"加密"存库，登录时再校验	utils/password_hash.py
2. JWT Token	登录成功发个"通行证"，后续请求带它表示"我登录过"	utils/jwt_utils.py
3. 中间件校验	每个请求来时，框架自动检查"通行证"是否有效	utils/middlewares.py
第 1 步：用户注册 —— 密码加密后存数据库
Python



from utils.password_hash import get_hashed_password

@router.post('/register/')
def create(obj_in: CreateOrUpdateUserSchema, session: Session = Depends(get_db)):
    if not obj_in.password:
        obj_in.password = str(settings.DEFAULT_PASSWORD)         # 没传密码 → 用默认 123123

    # ✅ 关键：把明文密码加密后再入库
    obj_in.password = get_hashed_password(obj_in.password)

    return _dao.create(session, obj_in)                          # 存进数据库
get_hashed_password("123456") 返回什么？




明文：123456
   ↓ bcrypt 加盐 + 多轮哈希
密文：$2b$12$KIXxH8hG8yT9kT8VfN6mPe5Q...（每次都不一样，因为有"盐"）
为什么这么做？ 就算数据库被偷，黑客拿到的也是一堆乱码。

第 2 步：用户登录 —— 验证密码 + 发"通行证"（Token）
Python



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
第 3 步：用户请求接口 —— 带着"通行证"来
前端拿到 token 后，每次请求都放在 header 里：




GET /api/users/3/
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
这一步不需要接口函数自己处理——中间件自动搞定。

第 4 步：服务端校验 Token —— 中间件自动跑
Python



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
关键点：request.state.username = username —— 中间件把"当前用户是谁"塞到 request 上，视图函数可以读 request.state.username。

第 5 步：接口里读"当前用户"
Python



@router.post('/graph/')
def execute_graph(request: Request, obj_in: BaseGraphSchema):
    print('登陆之后的用户名： ' + request.state.username)   # ← 直接读
    # ...
所以"确认是同一个人"= request.state.username == "zhangsan" 一直成立（只要 token 没过期）。

完整流程图



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
4 个关键点速记
关键点	一句话
密码存库	永远存 bcrypt 哈希，绝不存明文
登录成功发什么	发一个 JWT token（30 分钟过期）
后续请求带什么	HTTP header Authorization: Bearer <token>
服务器怎么认人	中间件自动从 header 解码 → 写到 request.state.username
项目里几个小细节
① 双重登录入口
Python



# /api/login/  - 给前端小程序用的（JSON 请求体）
@router.post('/login/')
def login(obj_in: UserLoginSchema, ...): ...

# /api/auth/   - 给 Swagger 文档"Authorize"按钮用的（form 表单）
@router.post('/auth/')
def auth(form_data: OAuth2PasswordRequestForm = Depends(), ...): ...
② main.py 里强制"所有接口都要登录"
Python



self.app = FastAPI(dependencies=[Depends(my_oauth2)])
#                                ↑ 把 OAuth2 设成"全局依赖"：所有接口都先过 token 校验
白名单（config/development.yml）：

YAML



WHITE_LIST: ['/api/login', '/api/register', '/static', '/docs', '/swagger', '/openapi', '/api/auth']
#           ↑ 登录、注册、Swagger、静态文件不需要 token
③ 修改密码时的"漏勺"
Python



# user_dao.py 的 update 里：只更新"前端实际传的字段"
update_data = obj_in.dict(exclude_unset=True)
#                          ↑ 前端没传的字段（比如 password）不会变成默认值去"清空"数据库
🎯 一句话终极总结
FastAPI 项目的"用户管理"3 步走：

注册：get_hashed_password 把明文 → bcrypt 密文 存库
登录：用 verify_password 比对 → 验证通过 → create_token 发 JWT
后续请求：前端在 header 带 Authorization: Bearer <token>，中间件 verify_token 自动解码 → 把 username 塞到 request.state.username，接口函数就能读
关键设计：接口层不碰密码和 token——所有这些都由 utils/ 里的工具函数和中间件搞定，业务代码（api/、db/）只管"调 DAO"和"返回响应"。

业务分层：

接口层（api/） = 服务员：接单 + 上菜（参数校验、状态码、调 DAO）
DAO 层（db/） = 后厨：备料 + 炒菜（SQL、ORM、事务）
session 是个快递员，在 get_db() → 接口函数 → _dao.xxx() 之间来回送数据库连接
记住 3 件事：

engine = 连接池（启动时建 1 个）
sessionmaker = 借窗口（启动时建 1 个）
session = 临时借条（每个请求 1 条）
