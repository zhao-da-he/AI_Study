# Python with 语句完全学习指南

> 本文档系统整理 Python 中 `with` 语句的所有用法，从基础语法到高级应用，适合新手小白系统学习。

---

## 目录

1. [什么是 with 语句](#一什么是-with-语句)
2. [为什么需要 with 语句](#二为什么需要-with-语句)
3. [with 语句基础语法](#三with-语句基础语法)
4. [上下文管理协议](#四上下文管理协议)
5. [常见内置上下文管理器](#五常见内置上下文管理器)
6. [自定义上下文管理器（类实现）](#六自定义上下文管理器类实现)
7. [使用 contextlib 模块简化](#七使用-contextlib-模块简化)
8. [常见应用场景](#八常见应用场景)
9. [异常处理详解](#九异常处理详解)
10. [with 与 try/finally 的对比](#十with-与-tryfinally-的对比)
11. [高级用法](#十一高级用法)
12. [注意事项与最佳实践](#十二注意事项与最佳实践)

---

## 一、什么是 with 语句

`with` 语句是 Python 中的一种**上下文管理协议（Context Management Protocol）**，用于简化资源管理代码。它可以确保代码块执行完毕后，**无论是否发生异常**，都能正确地清理资源（如关闭文件、释放锁、断开数据库连接等）。

### 核心思想

- **进入时**：自动调用对象的 `__enter__()` 方法，完成资源的获取（如打开文件）
- **退出时**：自动调用对象的 `__exit__()` 方法，完成资源的释放（如关闭文件）

---

## 二、为什么需要 with 语句

### 没有 with 时的写法（不推荐）

```python
# 打开文件
file = open("test.txt", "r", encoding="utf-8")
try:
    content = file.read()
    print(content)
finally:
    # 无论是否发生异常，都要关闭文件
    file.close()
```

### 使用 with 的写法（推荐）

```python
# 自动管理资源，简洁且安全
with open("test.txt", "r", encoding="utf-8") as file:
    content = file.read()
    print(content)
# 文件已自动关闭
```

**优势对比：**

| 对比项 | try/finally | with |
|--------|-------------|------|
| 代码量 | 多 | 少 |
| 可读性 | 一般 | 高 |
| 异常安全性 | 需手动保证 | 自动保证 |
| 嵌套使用 | 复杂 | 简洁 |

---

## 三、with 语句基础语法

### 1. 单个上下文管理器

```python
with 表达式 [as 变量]:
    代码块
```

### 2. 多个上下文管理器（Python 3.10+ 推荐）

```python
with (
    表达式1 as 变量1,
    表达式2 as 变量2,
):
    代码块
```

### 3. 多个上下文管理器（Python 3.10 之前）

```python
with 表达式1 as 变量1, 表达式2 as 变量2:
    代码块
```

### 4. 基础示例

```python
# 单个
with open("data.txt", "r") as f:
    data = f.read()

# 多个
with open("input.txt", "r") as fin, open("output.txt", "w") as fout:
    fout.write(fin.read())
```

---

## 四、上下文管理协议

任何实现了以下两个魔术方法的对象都可以被 `with` 语句使用：

### 1. `__enter__(self)`

- **调用时机**：进入 `with` 代码块时调用一次
- **返回值**：可以返回任何对象，通常返回 self 或资源对象
- **as 变量**：赋值的变量就是 `__enter__` 的返回值

### 2. `__exit__(self, exc_type, exc_val, exc_tb)`

- **调用时机**：退出 `with` 代码块时调用（包括异常退出）
- **参数**：
  - `exc_type`：异常类型（无异常时为 None）
  - `exc_val`：异常实例（无异常时为 None）
  - `exc_tb`：异常回溯信息（无异常时为 None）
- **返回值**：
  - 返回 True：吞掉异常，不会向外传播
  - 返回 False/None：异常正常向上抛出

### 3. 自定义示例（最简实现）

```python
class MyContext:
    def __enter__(self):
        print("进入 with 代码块")
        return self  # 返回值赋给 as 变量

    def __exit__(self, exc_type, exc_val, exc_tb):
        print("退出 with 代码块")
        # 返回 False 表示不处理异常
        return False


with MyContext() as ctx:
    print("执行主体逻辑")
    print("ctx is:", ctx)
```

**输出：**

```
进入 with 代码块
执行主体逻辑
ctx is: <__main__.MyContext object at 0x...>
退出 with 代码块
```

---

## 五、常见内置上下文管理器

### 1. 文件操作

```python
# 读取文件
with open("file.txt", "r", encoding="utf-8") as f:
    content = f.read()

# 写入文件
with open("output.txt", "w", encoding="utf-8") as f:
    f.write("Hello World")

# 读取行
with open("file.txt", "r", encoding="utf-8") as f:
    for line in f:
        print(line.strip())
```

### 2. 线程锁

```python
import threading

lock = threading.Lock()

with lock:
    # 同一时刻只有一个线程能执行这段代码
    print("线程安全的代码块")
# 锁自动释放
```

### 3. 数据库连接

```python
import sqlite3

# sqlite3.Connection 和 sqlite3.Cursor 都支持 with
with sqlite3.connect("test.db") as conn:
    with conn:  # 自动提交/回滚
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM users")
        rows = cursor.fetchall()
        print(rows)
```

### 4. 网络请求（urllib）

```python
from urllib.request import urlopen

with urlopen("https://www.example.com") as response:
    html = response.read().decode("utf-8")
    print(len(html))
# 连接自动关闭
```

### 5. decimal 精度设置

```python
from decimal import localcontext, Decimal

with localcontext() as ctx:
    ctx.prec = 50  # 设置高精度
    result = Decimal("1") / Decimal("7")
    print(result)
# 退出后精度恢复默认值
```

### 6. timeit 计时

```python
import timeit

with timeit.Timer("sum(range(100))") as timer:
    time_taken = timer.timeit(number=1000)
    print(f"耗时：{time_taken} 秒")
```

### 7. mock.patch 测试

```python
from unittest.mock import patch

with patch("module.function") as mock_func:
    mock_func.return_value = 42
    # 测试代码
# 自动恢复原函数
```

### 8. redirect_stdout/stderr

```python
import io
from contextlib import redirect_stdout

f = io.StringIO()
with redirect_stdout(f):
    print("这段输出不会显示在控制台")
# 退出后 print 恢复

output = f.getvalue()
print("捕获到的输出：", output)
```

---

## 六、自定义上下文管理器（类实现）

### 示例 1：计时器

```python
import time


class Timer:
    def __enter__(self):
        self.start = time.perf_counter()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.end = time.perf_counter()
        self.elapsed = self.end - self.start
        print(f"耗时：{self.elapsed:.4f} 秒")
        return False  # 不吞掉异常


with Timer() as t:
    total = sum(range(1000000))
print("计算结果：", total)
```

### 示例 2：临时切换工作目录

```python
import os


class ChangeDir:
    def __init__(self, path):
        self.new_path = path
        self.original_path = None

    def __enter__(self):
        self.original_path = os.getcwd()
        os.chdir(self.new_path)
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        os.chdir(self.original_path)
        return False


with ChangeDir("/tmp"):
    print("当前目录：", os.getcwd())
    # 在 /tmp 下执行操作
print("回到原目录：", os.getcwd())
```

### 示例 3：异常处理型上下文管理器

```python
class SuppressErrors:
    """吞掉指定异常"""
    def __init__(self, *exceptions):
        self.exceptions = exceptions

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        # 如果发生的异常在允许列表中，返回 True 吞掉
        if exc_type and issubclass(exc_type, self.exceptions):
            print(f"已吞掉异常：{exc_type.__name__}: {exc_val}")
            return True
        return False


with SuppressErrors(ValueError, TypeError):
    int("abc")  # ValueError 被吞掉
print("继续执行")
```

---

## 七、使用 contextlib 模块简化

对于简单的上下文管理器，可以使用 `contextlib` 模块避免写完整的类。

### 1. `@contextmanager` 装饰器

```python
from contextlib import contextmanager


@contextmanager
def my_context():
    # 相当于 __enter__
    print("开始")
    yield "返回值"  # yield 的值赋给 as 变量
    # 相当于 __exit__
    print("结束")


with my_context() as value:
    print(f"收到：{value}")
```

**输出：**

```
开始
收到：返回值
结束
```

### 2. 实际应用：计时器（简化版）

```python
from contextlib import contextmanager
import time


@contextmanager
def timer(name="block"):
    start = time.perf_counter()
    yield
    elapsed = time.perf_counter() - start
    print(f"[{name}] 耗时：{elapsed:.4f} 秒")


with timer("求和"):
    total = sum(range(1000000))
print("结果：", total)
```

### 3. 实际应用：临时目录

```python
from contextlib import contextmanager
import os
import tempfile


@contextmanager
def temporary_directory():
    path = tempfile.mkdtemp()
    try:
        yield path
    finally:
        # 清理临时目录
        import shutil
        shutil.rmtree(path, ignore_errors=True)


with temporary_directory() as tmp:
    print("临时目录：", tmp)
    filepath = os.path.join(tmp, "data.txt")
    with open(filepath, "w") as f:
        f.write("hello")
# 退出后临时目录被自动删除
```

### 4. 实际应用：数据库会话

```python
from contextlib import contextmanager
import sqlite3


@contextmanager
def database_session(db_path):
    conn = sqlite3.connect(db_path)
    try:
        yield conn
        conn.commit()  # 正常退出时提交
    except Exception:
        conn.rollback()  # 异常时回滚
        raise
    finally:
        conn.close()


with database_session("test.db") as conn:
    cursor = conn.cursor()
    cursor.execute("CREATE TABLE IF NOT EXISTS t (id INTEGER)")
    cursor.execute("INSERT INTO t VALUES (1)")
# 自动提交或回滚，并关闭连接
```

### 5. `closing()` 函数

用于给只有 `close()` 方法的对象（如 `urllib`）添加上下文管理能力：

```python
from contextlib import closing
from urllib.request import urlopen

with closing(urlopen("https://www.example.com")) as response:
    page = response.read()
```

### 6. `suppress()` 函数

```python
from contextlib import suppress

# 等价于：
# try:
#     os.remove("somefile.txt")
# except FileNotFoundError:
#     pass

with suppress(FileNotFoundError):
    os.remove("somefile.txt")
```

### 7. `ExitStack` 动态管理多个上下文

```python
from contextlib import ExitStack

files = ["file1.txt", "file2.txt", "file3.txt"]

with ExitStack() as stack:
    # 动态打开多个文件，退出时全部关闭
    file_objects = [stack.enter_context(open(f, "r")) for f in files]
    for f in file_objects:
        print(f.read())
# 所有文件自动关闭
```

### 8. `redirect_stdout` / `redirect_stderr`

```python
import io
from contextlib import redirect_stdout

buf = io.StringIO()
with redirect_stdout(buf):
    print("不会显示到屏幕")
print("捕获内容：", buf.getvalue())
```

---

## 八、常见应用场景

### 1. 文件读写（最常用）

```python
# JSON 文件
import json

with open("config.json", "r", encoding="utf-8") as f:
    config = json.load(f)

with open("output.json", "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)
```

### 2. CSV 文件

```python
import csv

with open("data.csv", "r", encoding="utf-8") as f:
    reader = csv.reader(f)
    for row in reader:
        print(row)

with open("output.csv", "w", encoding="utf-8", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["name", "age"])
    writer.writerow(["Alice", 30])
```

### 3. 数据库连接

```python
# pymysql
import pymysql

with pymysql.connect(host="localhost", user="root", password="pwd", db="test") as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT * FROM users LIMIT 10")
        results = cur.fetchall()
```

### 4. Redis 连接

```python
import redis

with redis.Redis(host="localhost", port=6379) as r:
    r.set("key", "value")
    print(r.get("key"))
```

### 5. 线程/进程池

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

with ThreadPoolExecutor(max_workers=5) as executor:
    futures = [executor.submit(pow, i, 2) for i in range(10)]
    for f in futures:
        print(f.result())
```

### 6. HTTP 请求（requests）

> 注：requests 的 Session 不是原生上下文管理器，但可以通过 close() 自定义

```python
import requests

# 推荐方式：手动管理
session = requests.Session()
try:
    response = session.get("https://api.example.com")
    print(response.json())
finally:
    session.close()

# 或者包装一下
from contextlib import contextmanager

@contextmanager
def requests_session():
    s = requests.Session()
    try:
        yield s
    finally:
        s.close()

with requests_session() as s:
    s.get("https://api.example.com")
```

### 7. 临时修改环境变量

```python
import os
from contextlib import contextmanager

@contextmanager
def set_env(key, value):
    old_value = os.environ.get(key)
    os.environ[key] = value
    try:
        yield
    finally:
        if old_value is None:
            os.environ.pop(key, None)
        else:
            os.environ[key] = old_value


with set_env("API_KEY", "test-key"):
    print(os.environ["API_KEY"])
# 恢复原值
print(os.getenv("API_KEY"))  # None
```

### 8. 资源加锁/解锁

```python
import threading

class SharedResource:
    def __init__(self):
        self.lock = threading.Lock()
        self.value = 0

    def increment(self):
        with self.lock:
            # 临界区
            temp = self.value
            temp += 1
            self.value = temp
```

---

## 九、异常处理详解

### `__exit__` 方法的返回值含义

```python
class Context:
    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is None:
            print("正常退出")
        else:
            print(f"异常退出：{exc_type.__name__}: {exc_val}")
        # 返回 True：吞掉异常，不向外传播
        # 返回 False/None：异常向上传播
        return False


# 正常退出
with Context():
    print("执行代码")

# 异常退出（异常会被传播）
with Context():
    raise ValueError("测试异常")
# 报错：ValueError: 测试异常
```

### 典型模式：在 `__exit__` 中处理异常

```python
class Transaction:
    """事务管理器：异常时回滚，否则提交"""
    def __enter__(self):
        # 开始事务
        print("事务开始")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is None:
            # 无异常，提交
            print("事务提交")
        else:
            # 有异常，回滚
            print(f"事务回滚：{exc_type.__name__}")
        return False  # 不吞掉异常，让上层知道


with Transaction():
    print("执行业务")
    # 如果此处抛异常，自动回滚
```

### contextmanager 的异常处理

```python
from contextlib import contextmanager

@contextmanager
def transaction():
    print("开始")
    try:
        yield
    except Exception as e:
        print(f"回滚：{e}")
        raise  # 重新抛出
    else:
        print("提交")


with transaction():
    print("业务逻辑")
```

---

## 十、with 与 try/finally 的对比

### 等价关系

```python
# with 写法
with EXPRESSION as TARGET:
    BODY

# 等价的 try/finally 写法（伪代码）
mgr = (EXPRESSION)
enter = type(mgr).__enter__
exit = type(mgr).__exit__
value = enter(mgr)
try:
    TARGET = value
    BODY
finally:
    exc = True
    try:
        if not (value is None):
            exit(mgr, *sys.exc_info())
        else:
            exit(mgr)
    except:
        exc = False
        raise
    finally:
        if exc:
            ...
```

### 选择建议

| 场景 | 推荐 |
|------|------|
| 仅做资源清理 | `with` |
| 复杂的异常处理 | `try/except/finally` |
| 资源清理 + 异常处理 | `with` + `try/except` 配合 |
| 简单的条件判断 | `if` 或 `try/except` |

### 混合使用

```python
with open("data.txt", "r") as f:
    try:
        data = f.read()
        process(data)
    except ValueError as e:
        # 业务逻辑异常处理
        log_error(e)
```

---

## 十一、高级用法

### 1. 异步 with 语句（async with）

用于异步上下文管理器（实现 `__aenter__` 和 `__aexit__` 方法）：

```python
import aiohttp

async def fetch():
    async with aiohttp.ClientSession() as session:
        async with session.get("https://api.example.com") as resp:
            data = await resp.json()
            return data

# 在异步环境中运行
import asyncio
result = asyncio.run(fetch())
print(result)
```

### 2. 异步上下文管理器自定义

```python
import asyncio


class AsyncTimer:
    async def __aenter__(self):
        self.start = asyncio.get_event_loop().time()
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        elapsed = asyncio.get_event_loop().time() - self.start
        print(f"异步耗时：{elapsed:.4f}s")
        return False


async def main():
    async with AsyncTimer():
        await asyncio.sleep(1)
        print("异步任务完成")

asyncio.run(main())
```

### 3. `@asynccontextmanager` 装饰器

```python
from contextlib import asynccontextmanager
import asyncio


@asynccontextmanager
async def async_timer():
    start = asyncio.get_event_loop().time()
    try:
        yield
    finally:
        elapsed = asyncio.get_event_loop().time() - start
        print(f"耗时：{elapsed:.4f}s")


async def main():
    async with async_timer():
        await asyncio.sleep(1)

asyncio.run(main())
```

### 4. 嵌套 with 语句

```python
with open("input.txt", "r") as fin:
    with open("output.txt", "w") as fout:
        fout.write(fin.read().upper())
```

### 5. 动态数量上下文管理器（ExitStack）

```python
from contextlib import ExitStack


def process_files(file_list):
    with ExitStack() as stack:
        # 根据动态列表打开文件
        files = [stack.enter_context(open(f, "r")) for f in file_list]
        # 处理文件
        for f in files:
            print(f.name)
    # 所有文件自动关闭


process_files(["a.txt", "b.txt", "c.txt"])
```

### 6. with 与生成器

```python
from contextlib import contextmanager

@contextmanager
def ignored(*exceptions):
    """忽略指定异常"""
    try:
        yield
    except exceptions:
        pass

# 使用
with ignored(OSError):
    os.remove("nonexistent.txt")
```

### 7. 上下文管理器作为装饰器

```python
from contextlib import contextmanager

@contextmanager
def debug_logging():
    print("[DEBUG] 开始执行")
    yield
    print("[DEBUG] 执行结束")


class my_decorator:
    """可以用 with 实现的装饰器（不常见，了解即可）"""
    def __init__(self, func):
        self.func = func

    def __enter__(self):
        return self.func

    def __exit__(self, *args):
        return False
```

### 8. 复用上下文管理器

```python
from contextlib import contextmanager

@contextmanager
def trace():
    print("进入")
    try:
        yield
    finally:
        print("离开")

# 多次复用
with trace():
    print("A")

with trace():
    print("B")
```

---

## 十二、注意事项与最佳实践

### 1. ✅ 应该做的事

#### （1）总是使用 with 管理需要关闭的资源

```python
# ✅ 推荐
with open("file.txt") as f:
    data = f.read()

# ❌ 不推荐
f = open("file.txt")
data = f.read()
f.close()  # 万一上面抛异常，文件未关闭
```

#### （2）处理可能为 None 的上下文管理器

```python
# file 可能为 None
with contextlib.nullcontext(file) as f:
    if f:
        process(f)
```

#### （3）合理使用 contextlib 简化代码

```python
# ✅ 推荐：contextmanager 装饰器
@contextmanager
def my_ctx():
    setup()
    try:
        yield
    finally:
        teardown()
```

### 2. ❌ 避免的做法

#### （1）不要在 `__enter__` 中做复杂业务

```python
# ❌ 不推荐
class BadContext:
    def __enter__(self):
        # 大量业务逻辑
        send_request()
        process_data()
        return self

    def __exit__(self, *args):
        return False

# ✅ 推荐：__enter__ 只做初始化，业务逻辑放在 with 块中
class GoodContext:
    def __enter__(self):
        self.resource = acquire_resource()
        return self

    def __exit__(self, *args):
        release_resource(self.resource)
        return False
```

#### （2）不要吞掉所有异常

```python
# ❌ 不推荐
def __exit__(self, *args):
    return True  # 吞掉所有异常，调试困难

# ✅ 推荐：仅在特定场景吞掉特定异常
def __exit__(self, exc_type, exc_val, exc_tb):
    if exc_type and issubclass(exc_type, self.suppressed_exceptions):
        log_warning(f"Suppressed: {exc_val}")
        return True
    return False
```

#### （3）不要在 `with` 块外使用 `as` 变量

```python
# ❌ 错误用法
with open("file.txt") as f:
    pass
print(f.read())  # NameError: f 未定义

# ✅ 正确：把使用范围限制在 with 块内
with open("file.txt") as f:
    content = f.read()
    print(content)
```

#### （4）注意 `__exit__` 中的异常吞掉行为

```python
class Demo:
    def __exit__(self, exc_type, exc_val, exc_tb):
        # 即使 __exit__ 自己抛异常，原异常也会被传播
        if exc_type:
            print(f"原异常：{exc_type.__name__}")
        # 如果此处返回 True，原异常被吞掉
        return True

with Demo():
    raise ValueError("测试")
# ValueError 不会传播到外层，因为 __exit__ 返回了 True
```

### 3. 🐛 常见陷阱

#### （1）文件编码问题

```python
# ❌ 不指定编码，跨平台可能报错
with open("file.txt") as f:
    data = f.read()

# ✅ 明确指定编码
with open("file.txt", encoding="utf-8") as f:
    data = f.read()
```

#### （2）不支持 with 的对象

```python
# requests.Session 不是原生上下文管理器
import requests

session = requests.Session()
# with session:  # ❌ AttributeError

# ✅ 用 closing 或 contextmanager 包装
from contextlib import closing

with closing(requests.Session()) as s:
    s.get("https://example.com")
```

#### （3）生成器函数中 yield 的陷阱

```python
from contextlib import contextmanager

@contextmanager
def broken_ctx():
    print("setup")
    yield  # 第一次
    yield  # ❌ 多次 yield 会导致后续 yield 不执行清理
    print("teardown")
```

### 4. 📋 速查表

| 用途 | 推荐写法 |
|------|---------|
| 文件操作 | `with open(...) as f:` |
| 线程锁 | `with lock:` |
| 数据库连接 | `with connection:` |
| 临时环境 | `@contextmanager` |
| 忽略异常 | `contextlib.suppress` |
| 关闭任意对象 | `contextlib.closing` |
| 异步资源 | `async with` |
| 动态多资源 | `contextlib.ExitStack` |

---

## 附录：完整实战示例

### 综合示例：安全的文件处理器

```python
from contextlib import contextmanager
import os
import json
import logging


@contextmanager
def safe_file_operation(filepath, mode="r", encoding="utf-8"):
    """安全的文件操作上下文管理器"""
    file = None
    try:
        file = open(filepath, mode, encoding=encoding)
        yield file
    except FileNotFoundError as e:
        logging.error(f"文件未找到：{filepath}")
        raise
    except PermissionError as e:
        logging.error(f"权限不足：{filepath}")
        raise
    except Exception as e:
        logging.error(f"文件操作失败：{e}")
        raise
    finally:
        if file and not file.closed:
            file.close()
            logging.info(f"文件已关闭：{filepath}")


# 使用
with safe_file_operation("config.json") as f:
    config = json.load(f)
    print(config)
```

### 综合示例：数据库事务

```python
from contextlib import contextmanager
import logging


@contextmanager
def db_transaction(connection):
    """数据库事务管理"""
    cursor = connection.cursor()
    try:
        yield cursor
        connection.commit()
        logging.info("事务已提交")
    except Exception as e:
        connection.rollback()
        logging.error(f"事务已回滚：{e}")
        raise
    finally:
        cursor.close()


# 使用
with db_transaction(conn) as cur:
    cur.execute("INSERT INTO users (name) VALUES (?)", ("Alice",))
    cur.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
# 自动提交或回滚
```

---

## 参考资料

- Python 官方文档：[Context Managers](https://docs.python.org/3/reference/datamodel.html#context-managers)
- Python 官方文档：[contextlib 模块](https://docs.python.org/3/library/contextlib.html)
- PEP 343：[The with statement](https://peps.python.org/pep-0343/)

---

> 文档持续完善中，建议配合实际代码练习理解。