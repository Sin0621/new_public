
# 《Python Web 项目实战》第一册：把骨架搭起来

> **怎么读这本书**
> - 每章结构固定：**需求 → 先看能跑的最小版（A 版）→ 逐行讲 → 发现它的毛病 → 升级成项目版（B 版）→ 对照总结**
> - 每章末尾有「本章只需记住 3 句话」，背这 3 句就够。
> - 代码一定要**亲手敲一遍**，不要只看。看会 ≠ 会。
> - 遇到看不懂的符号，先回到第 0 章查。

---

# 第 0 章 看代码前必须补的 5 个概念

这一章不讲项目，只讲"读懂任何 Python 代码"需要的底层常识。**这 5 个概念不通，后面每一行都会卡住。**

## 0.1 库、模块、import —— 工具箱比喻

Python 自带一批工具箱（叫**标准库**），比如 `logging`（日志）、`os`（操作系统）、`time`（时间）。
你也可以用 `pip install` 装别人做好的工具箱（叫**第三方库**），比如 `fastapi`、`pydantic`。

`import` 的意思就是：**把这个工具箱搬到当前文件里来用。**

三种写法，含义不同：


# 写法1：整个工具箱搬进来，用的时候要写"工具箱名.工具名"
import logging
logging.getLogger()          # logging 工具箱 → 里面的 getLogger 工具

# 写法2：只从工具箱里拿某几个工具，直接放桌上，用的时候不用写工具箱名
from fastapi import FastAPI, APIRouter
app = FastAPI()              # 直接写 FastAPI，不用写 fastapi.FastAPI

# 写法3：工具箱里还有"抽屉"（子模块），从抽屉里拿工具
from logging.handlers import RotatingFileHandler
#          ↑包名    ↑子模块        ↑具体工具


**记住这个对应关系：**

| 代码 | 中文意思 |
|---|---|
| `import logging` | 把 `logging` 工具箱搬来 |
| `from logging.handlers import RotatingFileHandler` | 从 `logging` 的 `handlers` 抽屉里，拿 `RotatingFileHandler` 这个工具 |
| `fastapi` 是什么 | 一个第三方工具箱的名字 |

## 0.2 点号 `.` 是什么意思 —— 就是"的"

Python 里几乎所有"归属关系"都用点号表示。**读的时候把 `.` 读成"的"。**


logging.getLogger()
# 读作：logging 的 getLogger（工具）

logger.info("你好")
# 读作：logger 的 info（动作）

request.state.request_id
# 读作：request 的 state 的 request_id


所以看到一串点号不要慌，**一层一层从右往左读**："request_id 属于 state，state 属于 request"。

## 0.3 类、实例、属性、方法

你说"class 是类创建"——对，但补一句准确的：

- `class` 定义的是**类（图纸）**
- `类名()` 加括号，是**造一个实例（按图纸盖出的房子）**
- 类里面定义的变量叫**属性**（房子的特征：颜色、面积）
- 类里面定义的函数叫**方法**（房子能做的事：开门、关灯）


class Dog:                  # 图纸
    def __init__(self, name):   # 构造方法：盖房子时自动执行
        self.name = name        # 属性
    def bark(self):             # 方法
        return "汪汪"

d = Dog("旺财")             # 按图纸盖一间房 → 实例
print(d.name)               # 旺财（属性）
print(d.bark())             # 汪汪（方法）


**术语纠正**（很重要，不然看文档会迷）：`def` 创建的是**函数**；写在 `class` 里面的 `def` 才叫**方法**。日常口头混用没关系，但看官方文档时要分清。

还有 `class Settings(BaseSettings):` 括号里的东西叫**继承**，意思是"Settings 在 BaseSettings 的基础上改造"。读作："Settings 是一种 BaseSettings"。

## 0.4 函数调用里的三种写法


def greet(name, greeting="你好", times=1):   # name 必填；后两个有默认值
    return greeting * times

greet("小明")                     # 只用位置：小明
greet("小明", "早上好")            # 按顺序给
greet("小明", times=3)            # 关键字参数：明确说 times 是 3，跳过 greeting


- `greeting="你好"` 这种**带默认值**的参数，调用时可以不给。
- 调用时写 `times=3` 叫**关键字参数**，好处是不用记顺序。项目代码里大量的 `maxBytes=...`、`backupCount=...` 就是这个。
- `-> str`、`: str` 叫**类型注解**，只是给人看的提示，Python 不会强制检查。

## 0.5 两个"看起来吓人"的语法

**（1）装饰器 `@xxx`**


@router.get("/health")
async def health_check():
    ...


先别管原理，**把它读成标签**：`@router.get("/health")` 就是给下面这个函数贴一张标签，写着"**这个函数负责处理访问 /health 的请求**"。

框架看到标签，就知道该在什么时候调用它。**你写的函数本身没变，是标签赋予了它额外身份。**

**（2）`async def` 和 `await`**


async def health_check():
    ...


读作："这是一个**异步函数**"。异步函数的特点是：**它在等外部东西（网络、磁盘、数据库）的时候，会主动让出 CPU 去干别的活**，所以一个服务器能同时接待很多请求。

初学者只要记两条规则：

1. FastAPI 的接口函数，一律写 `async def`。
2. 在 `async def` 里面调用另一个 `async def` 函数时，前面要加 `await`（意思是"在这儿等它的结果"）。

> 现在不用理解原理，会用就行。这是本册唯一"先背规则再理解"的地方。

---

# 第 1 章 写一个 `logger.py`

## 1.1 我们的需求

程序跑起来后，我想知道：什么时候启动的、谁访问了哪个接口、哪里出错了。
**不能靠 `print`**，因为它：① 没有时间；② 没有级别；③ 关不掉；④ 不能存进文件。

## 1.2 A 版：能跑的最小日志（5 行）


import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(name)s | %(message)s",
)

logger = logging.getLogger(__name__)

logger.info("服务启动了")
logger.warning("磁盘快满了")
logger.debug("这条不会显示，因为级别不够")


**逐行讲：**

| 代码 | 在干什么 |
|---|---|
| `import logging` | 搬来标准库里的日志工具箱 |
| `logging.basicConfig(...)` | 一次性配置"根日志器"（相当于总开关） |
| `level=logging.INFO` | 只显示 INFO 及以上级别。级别从低到高：DEBUG < INFO < WARNING < ERROR |
| `format="%(asctime)s ..."` | 日志的排版模板。`%(xxx)s` 是 logging 内置的占位符，不是字符串格式化 |
| `%(asctime)s` | 时间 |
| `%(levelname)s` | 级别名（INFO / WARNING） |
| `%(name)s` | 哪个模块打的日志 |
| `%(message)s` | 你写的那句话 |
| `logging.getLogger(__name__)` | 拿到"属于当前这个文件"的日志器。`__name__` 是 Python 自动填的当前模块名 |
| `logger.info(...)` | 打一条 INFO 日志 |

**动手实验**：把上面存成 `demo_logger.py`，运行 `python demo_logger.py`，你会看到类似：


2026-10-19 16:20:31,512 | INFO | __main__ | 服务启动了
2026-10-19 16:20:31,512 | WARNING | __main__ | 磁盘快满了


注意 `DEBUG` 那条没出来——因为级别设的是 INFO。

## 1.3 A 版的 3 个毛病

1. **不能重复配置**：`basicConfig` 只有第一次调用生效，第二次调用完全不做事。
   → 后果：项目用 `--reload` 改代码时，模块会被重新导入，日志就乱了。
2. **日志只在控制台**，关掉终端就没了。要存文件。
3. **文件会无限变大**，跑一个月能到几个 G。要能自动切分。

## 1.4 B 版：项目版（逐行讲）


"""日志配置模块。"""
import logging
import sys
from logging.handlers import RotatingFileHandler
from pathlib import Path

from app.config import settings

LOG_FORMAT = "%(asctime)s | %(levelname)-8s | %(name)s | %(message)s"


def setup_logging() -> None:
    root_logger = logging.getLogger()          # ① 不带参数 = 拿"根日志器"（总管家）

    if root_logger.handlers:                   # ② 已经配过了就直接退出（解决毛病1）
        return
    
    root_logger.setLevel(settings.LOG_LEVEL.upper())
    
    formatter = logging.Formatter(LOG_FORMAT)  # ③ 先做一张"排版模板"
    
    console_handler = logging.StreamHandler(sys.stdout)   # ④ 造一个"输出到控制台"的工人
    console_handler.setFormatter(formatter)               # ⑤ 给工人发模板
    root_logger.addHandler(console_handler)               # ⑥ 让管家雇这个工人
    
    log_dir = Path(settings.LOG_DIR)
    log_dir.mkdir(parents=True, exist_ok=True)            # ⑦ 目录不存在就建
    
    file_handler = RotatingFileHandler(
        filename=log_dir / "app.log",
        maxBytes=10 * 1024 * 1024,        # 10MB
        backupCount=5,                    # 最多留 5 个旧文件
        encoding="utf-8",                 # 不写中文乱码
    )
    file_handler.setFormatter(formatter)
    root_logger.addHandler(file_handler)


def get_logger(name: str) -> logging.Logger:
    return logging.getLogger(name)


**关键概念：`Handler` 是什么？**

把它想成"**打工人**"。日志消息是"活儿"，`logger` 是"老板"，`Handler` 是"具体干活的工人"。

- `StreamHandler(sys.stdout)` → 负责把日志**打印到屏幕**
- `RotatingFileHandler(...)` → 负责把日志**写进文件**，写满 10MB 就换个新文件

老板（`logger`）可以同时雇好几个工人，所以一条日志能同时出现**在屏幕上和文件里**。

**再看几个新符号：**

| 代码 | 含义 |
|---|---|
| `logging.getLogger()` | 括号里不填 = 拿根日志器（总管一切） |
| `root_logger.handlers` | 根日志器身上的"工人列表"，是个列表 |
| `if root_logger.handlers: return` | 列表非空 = 已经雇过人了 = 不用再配。**这就是解决毛病 1 的办法** |
| `.upper()` | 把字符串变大写，"info" → "INFO" |
| `logging.Formatter(LOG_FORMAT)` | 造一张排版模板 |
| `setFormatter(formatter)` | 把模板发给某个工人 |
| `addHandler(handler)` | 给日志器雇一个工人 |
| `Path("logs")` | 造一个路径对象（比字符串拼接更安全） |
| `log_dir / "app.log"` | 路径对象用 `/` 拼路径，等价于 `"logs/app.log"` |
| `.mkdir(parents=True, exist_ok=True)` | 建目录；`parents=True` 连父目录一起建；`exist_ok=True` 已存在也不报错 |
| `maxBytes` / `backupCount` | 关键字参数，含义见注释 |

## 1.5 A/B 对照

| 需求 | A 版 | B 版 | 差在哪 |
|---|---|---|---|
| 打日志 | `basicConfig` | `setup_logging()` | B 把"打包做法"拆开，可重复调用 |
| 输出到屏幕 | `basicConfig` 自动做 | `StreamHandler` | 手动指定输出目标 |
| 输出到文件 | ❌ | `RotatingFileHandler` | 新增 |
| 文件不无限大 | ❌ | `maxBytes + backupCount` | 新增 |
| 重复配置不报错 | ❌ | `if handlers: return` | 新增 |

> **本质**：`basicConfig` 是"一键傻瓜模式"，内部帮你 `addHandler` 了一个 `StreamHandler` 并配好格式。B 版只是**把这个过程摊开**，再加两个功能。**不是两套知识。**

### 本章只需记住 3 句话

1. 日志 = 老板（logger）+ 工人（Handler），`addHandler` 就是雇工人。
2. `Formatter` 是排版模板，要发给每个工人。
3. 配置只做一次，开头先判 `if logger.handlers: return`。

---

# 第 2 章 写一个 `config.py`

## 2.1 需求

数据库地址、端口、API Key 这些东西**不能写死在代码里**——本地开发和线上不一样，密钥更不能提交到 Git。

## 2.2 A 版：用 `os.getenv` 读环境变量


import os

APP_NAME = os.getenv("APP_NAME", "InsightAgent")
PORT = int(os.getenv("PORT", "8000"))
DEBUG = os.getenv("DEBUG", "false").lower() == "true"

print(APP_NAME, PORT, DEBUG)


**逐行讲：**

| 代码 | 含义 |
|---|---|
| `import os` | 搬来"操作系统"工具箱，它能读环境变量 |
| `os.getenv("APP_NAME", "InsightAgent")` | 读名为 `APP_NAME` 的环境变量；读不到就用默认值 `InsightAgent` |
| 环境变量 | 操作系统里存的键值对，程序启动时能读到。`export APP_NAME=xx` 就是设一个 |
| `int(...)` | 环境变量读出来永远是**字符串**，`"8000"` 要手动转成数字 `8000` |
| `.lower() == "true"` | 字符串没有布尔类型，只能自己判断 |

## 2.3 A 版的 4 个毛病

1. **每个变量都要手写类型转换**（`int(...)`），多了就烦，忘了就出 bug。
2. **`.env` 文件不会被自动读**——`os.getenv` 只认真正的环境变量，不认文件。
3. **拼错没提示**：写 `os.getenv("PROT")` 不会报错，只会静默返回默认值。
4. **没人知道项目到底有哪些配置项**，散落在各处。

## 2.4 B 版：pydantic-settings（逐行讲）

先装库：`pip install pydantic-settings`


"""全局配置模块。"""
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",        # 从 .env 文件读
        extra="ignore",         # .env 里多出来的键，忽略，不报错
        case_sensitive=False,   # 变量名不区分大小写
    )

    APP_NAME: str = "InsightAgent"
    APP_ENV: str = "dev"
    DEBUG: bool = True
    PORT: int = 8000
    
    LOG_LEVEL: str = "INFO"
    LOG_DIR: str = "logs"


settings = Settings()


**逐行讲：**

| 代码 | 含义 |
|---|---|
| `from pydantic_settings import BaseSettings` | 从一个叫 pydantic_settings 的第三方工具箱里，拿 `BaseSettings` 这个工具 |
| `BaseSettings` | 一个**专门的父类**：谁继承它，谁就自动获得"从环境变量/.env 读值"的能力 |
| `class Settings(BaseSettings):` | 定义自己的配置类，继承上面的能力 |
| `model_config = SettingsConfigDict(...)` | Pydantic v2 的固定写法，用来告诉父类"去哪读、怎么读"。当成**固定模板背下来** |
| `env_file=".env"` | 去当前目录的 `.env` 文件读 |
| `APP_NAME: str = "InsightAgent"` | **这就是一个配置项**：变量名 = 环境变量名，类型 = 自动转换规则，等号后 = 默认值 |
| `DEBUG: bool = True` | 注意：`.env` 里写 `DEBUG=true`，它会自动变成布尔 `True`，**不用你手写转换** |
| `settings = Settings()` | 造一个实例。整个项目共用这一个 |

**它自动解决了 A 版的 4 个毛病：**

1. 类型转换 → 看类型注解，自动做。
2. 读 `.env` → `env_file` 指定了。
3. 拼错/类型错 → 启动时立刻报错（比如 `PORT: int` 却给了 `"abc"`）。
4. 配置项一览无余 → 都在这个类里。

**用法（其他文件里）：**


from app.config import settings

print(settings.PORT)      # 8000，而且是 int 类型


> `from app.config import settings` 的意思是：从 `app` 文件夹下的 `config.py` 文件里，导入 `settings` 这个变量。**`app.config` 就是"app 目录里的 config 模块"**，点号表示目录层级。

## 2.5 A/B 对照

| 需求 | A 版 | B 版 |
|---|---|---|
| 读配置 | `os.getenv` | `Settings` 类字段 |
| 类型转换 | 手动 `int()` | 自动 |
| 读 `.env` | ❌ | `env_file=".env"` |
| 写错有提示 | ❌ | ✅ 启动报错 |
| 配置集中 | ❌ | ✅ 一个类看全 |

### 本章只需记住 3 句话

1. 配置 = 一个继承 `BaseSettings` 的类，**字段就是配置项**。
2. `SettingsConfigDict(env_file=".env")` 是固定模板，照抄。
3. 全项目只 `from app.config import settings` 这一句，别再写 `os.getenv`。

---

# 第 3 章 写一个 `exceptions.py`

## 3.1 先补两个概念

**（1）HTTP 状态码**：服务器回应请求时，会给一个三位数字。

| 码 | 含义 | 例子 |
|---|---|---|
| 200 | 成功 | 正常返回数据 |
| 400 | 你传的参数不对 | 文件类型不支持 |
| 404 | 找不到 | 查的文档不存在 |
| 422 | 参数格式校验失败 | 该填数字你填了字母 |
| 500 | 服务器自己出 bug 了 | 代码报错 |
| 502 | 上游服务挂了 | 调 LLM 超时 |

**（2）`raise` 和 `try/except`**

- `raise 某个异常` = **主动抛出**一个错误，程序会中断，往上找谁接。
- `try: ... except 某异常: ...` = **接住**错误，不让它中断程序。


try:
    x = 1 / 0
except ZeroDivisionError:
    print("不能除以 0")


`Exception` 是 Python 里**所有异常的父类**。所以"继承 Exception"就是"我是一个异常"。

## 3.2 A 版：直接在接口里 `raise HTTPException`


from fastapi import FastAPI, HTTPException

app = FastAPI()

@app.get("/items/{item_id}")
def get_item(item_id: int):
    if item_id != 1:
        raise HTTPException(status_code=404, detail="找不到这个物品")
    return {"id": 1, "name": "苹果"}


**逐行讲：**

| 代码 | 含义 |
|---|---|
| `from fastapi import FastAPI, HTTPException` | 从 fastapi 工具箱拿两个工具 |
| `app = FastAPI()` | 造一个应用实例（整个 Web 服务的主体） |
| `@app.get("/items/{item_id}")` | 贴标签：这个函数负责处理"GET 请求 /items/某个id" |
| `{item_id}` | 路径里的占位符，会被真实值替换 |
| `def get_item(item_id: int)` | 函数参数自动从路径里取值，并按 `int` 转成数字 |
| `raise HTTPException(404, "找不到")` | 抛一个框架认识的错误，框架会自动返回 404 JSON |
| `return {...}` | 正常返回时，FastAPI 自动把 dict 转成 JSON |

## 3.3 A 版的 3 个毛病

1. **每个接口都要写一遍** `raise HTTPException(...)`，重复。
2. **返回格式不统一**：`HTTPException` 返回 `{"detail": "..."}`，但参数校验错误返回的是另一种结构，前端要写两套解析。
3. **500 错误没人管**：代码真出 bug 时，可能把堆栈信息暴露给用户（安全隐患）。

## 3.4 B 版：自定义异常 + 全局处理器（逐行讲）


from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse


class AppError(Exception):              # ① 定义一个"业务异常"
    status_code = 400                   # 类属性：默认状态码
    code = "BAD_REQUEST"                # 类属性：给前端看的错误码
    message = "请求错误"                 # 类属性：默认提示

    def __init__(self, message: str | None = None):
        self.message = message or self.message
        super().__init__(self.message)


class NotFoundError(AppError):          # ② 继承，只改几个类属性
    status_code = 404
    code = "NOT_FOUND"


async def app_error_handler(request: Request, exc: AppError):    # ③ 处理函数
    return JSONResponse(
        status_code=exc.status_code,
        content={"code": exc.code, "message": exc.message},
    )


def register(app: FastAPI):             # ④ 注册
    app.add_exception_handler(AppError, app_error_handler)


**逐行讲：**

| 代码 | 含义 |
|---|---|
| `class AppError(Exception)` | 定义异常类，继承内置的 `Exception` |
| `status_code = 400` | **类属性**：写在类里、所有实例共享的变量 |
| `def __init__(self, message=None)` | 构造方法，造实例时自动执行。`message=None` = 可选 |
| `message or self.message` | 传了 message 就用传的，没传就用类属性的默认值 |
| `super().__init__(self.message)` | 调用父类 `Exception` 的构造，让 `str(exc)` 能拿到消息 |
| `class NotFoundError(AppError)` | 继承 `AppError`，只覆盖两个类属性，**一行逻辑都不用写** |
| `async def app_error_handler(request, exc)` | 处理函数。签名固定：第一个是请求，第二个是异常 |
| `JSONResponse(...)` | 造一个真正的 HTTP 响应对象 |
| `exc.status_code` | 从异常实例上取状态码 |
| `app.add_exception_handler(AppError, app_error_handler)` | 配对："以后谁抛 AppError，就交给这个函数处理" |

**使用效果：**


raise NotFoundError("文档 123 不存在")
# 自动返回：HTTP 404, {"code": "NOT_FOUND", "message": "文档 123 不存在"}


**业务代码从"写响应"变成了"只抛异常"**——这是这套设计的核心价值。

> 完整版还会再加两个处理器（参数校验错误、未知异常兜底）。**它们不是新知识**，只是把 `add_exception_handler` 再调用两次、换个异常类型。理解上面这 5 步，剩下都是重复。

### 本章只需记住 3 句话

1. 自定义异常 = 继承 `Exception`，加几个类属性，不写逻辑。
2. 处理函数签名固定为 `(request, exc)`，返回 `JSONResponse`。
3. `add_exception_handler(异常类, 函数)` 把两者配对，之后业务代码只管 `raise`。

---

# 第 4 章 路由与入口：`health.py` + `main.py`

## 4.1 A 版：全写在一个 `main.py` 里


from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
async def health_check():
    return {"status": "ok"}

@app.get("/items")
async def list_items():
    return {"items": []}


**逐行讲：**

| 代码 | 含义 |
|---|---|
| `app = FastAPI()` | 造应用实例 |
| `@app.get("/health")` | 标签：GET 请求访问 `/health` 时，执行下面这个函数 |
| `async def health_check()` | 异步函数（见 0.5） |
| `return {...}` | 返回 dict，FastAPI 自动转成 JSON |

## 4.2 A 版的毛病

接口一多，`main.py` 会变成几千行。**所有东西堆在一个文件里，改一个接口要滚半天。**

## 4.3 B 版：用 `APIRouter` 拆文件

**`health.py`**


from fastapi import APIRouter
from pydantic import BaseModel

router = APIRouter()           # ① 造一个"路由分组"


class HealthResponse(BaseModel):      # ② 出参模型
    status: str
    app_name: str


@router.get("/health", response_model=HealthResponse)
async def health_check():
    return HealthResponse(status="ok", app_name="InsightAgent")


**逐行讲：**

| 代码 | 含义 |
|---|---|
| `APIRouter()` | 一个"小型 app"，专门装一组接口。可以理解为**文件夹** |
| `class HealthResponse(BaseModel)` | 继承 Pydantic 的 `BaseModel`，定义一个"数据形状"。不是配置，是**数据模板** |
| `status: str` | 这个字段必须是字符串 |
| `response_model=HealthResponse` | 声明出参格式：FastAPI 会按这个模板校验并生成文档 |
| `@router.get(...)` | 和 `@app.get` 用法完全一样，只是挂到了 router 上 |

**`main.py`**


from fastapi import FastAPI
from app.api.v1.health import router as health_router

app = FastAPI(title="InsightAgent")

app.include_router(health_router, prefix="/api/v1")


**逐行讲：**

| 代码 | 含义 |
|---|---|
| `from app.api.v1.health import router` | 从 `app/api/v1/health.py` 里导入 `router` 这个变量 |
| `as health_router` | 起个别名，避免和别的 router 重名 |
| `include_router(router, prefix="/api/v1")` | 把这个分组挂到 app 上，并给所有接口统一加前缀 |
| `prefix="/api/v1"` | 最终地址变成 `/api/v1/health`。`v1` 是版本号，以后升级开 `v2`，老接口不受影响 |

> **`as` 是什么意思**：`import X as Y` = "把 X 拿进来，但我在这文件里管它叫 Y"。纯粹是起外号。

## 4.4 `main.py` 的完整职责（装配车间）


from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.api.v1.health import router as health_router
from app.core.exceptions import register_exception_handlers
from app.core.logging import setup_logging, get_logger

logger = get_logger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    setup_logging()                      # 启动时：配日志
    logger.info("服务启动")
    yield                                # 分隔线：上面是启动，下面是关闭
    logger.info("服务关闭")


app = FastAPI(lifespan=lifespan)

app.add_middleware(                      # 中间件：每个请求都要先过的一道关
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(health_router, prefix="/api/v1")
register_exception_handlers(app)


**新概念只两个：**

**（1）`lifespan` = 启动/关闭钩子**


@asynccontextmanager
async def lifespan(app):
    # yield 之前：服务启动时执行一次
    yield
    # yield 之后：服务关闭时执行一次


`yield` 就是"分隔线"。以后数据库连接池、向量库，都在 `yield` 前初始化、`yield` 后关闭。

**（2）中间件 = 安检门**

每个请求进来都要先过它，出去也要再过一次。典型用途：加请求 ID、算耗时、跨域。


@app.middleware("http")
async def my_middleware(request, call_next):
    # 请求进来时做的事
    response = await call_next(request)   # 放行，交给后面的接口
    # 响应出去时做的事
    return response


`call_next(request)` = "**让请求继续往下走**"，返回值是接口的响应。

### 本章只需记住 3 句话

1. `APIRouter` 是接口的文件夹，`include_router` 是把它挂到 app 上。
2. `main.py` 只做装配：建 app、挂中间件、挂路由、注册异常。
3. `lifespan` 用 `yield` 分隔启动和关闭；中间件用 `call_next` 放行。

---

# 第 5 章 动手跑起来（完整流程）


# 1) 建目录
mkdir insight-agent && cd insight-agent
mkdir -p app/api/v1 app/core tests logs

# 2) 建虚拟环境并激活
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

# 3) 装依赖
pip install fastapi uvicorn pydantic-settings

# 4) 建 .env
echo "APP_NAME=InsightAgent" > .env
echo "PORT=8000" >> .env

# 5) 启动
uvicorn app.main:app --reload --port 8000


**验证：**

1. 浏览器打开 `http://127.0.0.1:8000/api/v1/health` → 看到 `{"status":"ok",...}`
2. 打开 `http://127.0.0.1:8000/docs` → 看到自动生成的接口文档（FastAPI 白送的）
3. 看终端 → 有启动日志和访问日志
4. 看 `logs/app.log` → 同样的日志也写进文件了

**每个目录的作用：**

| 路径 | 干什么 |
|---|---|
| `app/` | 主代码 |
| `app/api/v1/` | 接口层，按版本分 |
| `app/core/` | 全局设施（日志、异常） |
| `logs/` | 日志输出 |
| `.env` | 本地配置（不提交 Git） |
| `.venv/` | 虚拟环境（不提交 Git） |

---

# 第 6 章 自测（做完这一册，答这 10 题）

1. `import logging` 和 `from fastapi import FastAPI` 有什么区别？
2. `logging.getLogger(__name__)` 里的 `__name__` 是什么？
3. 日志里 `Handler` 是什么角色？为什么要 `addHandler`？
4. `basicConfig` 有什么局限？项目版怎么绕过的？
5. `BaseSettings` 的字段为什么能自动变成 `int` / `bool`？
6. `from app.config import settings` 这句里的点号表示什么？
7. `raise` 和 `return` 有什么区别？
8. 为什么处理函数要写成 `(request, exc)` 两个参数？
9. `APIRouter` 和 `FastAPI` 实例是什么关系？
10. `@router.get("/health")` 这行"标签"到底干了什么？

<details>
<summary>答案要点（先自己答，再点开）</summary>

1. 前者搬整个工具箱，用时要写 `logging.xxx`；后者只拿一个工具，直接用 `FastAPI`。
2. Python 自动填的当前模块名，即文件名。
3. 干活的工人；`addHandler` 就是给日志器雇工人，可以雇多个（屏幕+文件）。
4. 只能生效一次；项目版改成手写 `Handler`，并加 `if handlers: return` 防重复。
5. `BaseSettings` 会读字段的类型注解，据此做类型转换。
6. 目录层级：`app` 文件夹下的 `config` 模块。
7. `raise` 抛出异常、中断流程；`return` 正常返回结果。
8. `request` 拿请求信息，`exc` 是抛出的异常实例，处理函数要用它取状态码和消息。
9. `APIRouter` 是接口分组（小文件夹），`FastAPI` 是应用主体；用 `include_router` 把前者挂到后者。
10. 把这个函数注册成一个处理 `GET /health` 的路由，框架据此在收到请求时调用它。

</details>

---

# 下一册预告

第二册：**文件加载 + 文本清洗与分块**。那里会正式用上：
- `open()` 和 `with`（文件读写）
- `try/except`（异常处理）
- `re` 正则库（`re.sub` 清标签、`re.split` 断句）
- 生成器 `yield`（逐块处理大文件）

要我现在开始写第二册吗？还是先把第一册的代码亲手敲一遍、把 10 道题做完？

