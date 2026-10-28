# 《Python Web 项目实战》第五册：HTTP 接口  
  
| 项目 | 内容 |  
|---|---|  
| 系列 | 《Python Web 项目实战》 |  
| 册数 | 第五册（开发阶段五：Day 9-10） |  
| 主题 | 接口契约 · 依赖注入 · 错误映射 · 流式返回 · 接口测试 |  
| 前置要求 | 第一册（路由、异常骨架）、第四册（`AgentExecutor`） |  
| 版本 | v1.9 · 2026-10-28 |  
  
## 本册地图  
  
| 章 | 当前解决点 | 产出文件 |  
|---|---|---|  
| 第1章 挂接口 | 打开 `app/api/v1/chat.py`，把 Agent 挂到 HTTP 接口 | `app/api/v1/schemas.py`、`app/api/v1/chat.py` |  
| 第2章 错误映射 | 打开 `app/core/exceptions.py`，让错误映射成正确的状态码 | `app/core/exceptions.py`、`app/main.py` |  
| 第3章 流式返回 | 打开 `app/api/v1/chat.py`，把一次性响应改成流式返回 | `app/agent/executor.py`、`app/api/v1/chat.py` |  
| 第4章 测接口 | 打开 `tests/test_chat_api.py`，用 TestClient 固定接口行为 | `tests/test_chat_api.py`、`tests/conftest.py` |  
| 下一册 | 打开 `Dockerfile`，把服务打包成可部署镜像 | — |  
  
本册贯穿同一句真实提问：**「这个项目部署时用哪个端口？」**  
正确答案仍是 **8000**。  
  
流程：先让问题能被 HTTP 调起；再让失败有正确的状态码；再让用户不必干等；最后用测试把接口行为固定住。  
  
术语校准：接口层只做三件事——校验入参、调下层、把结果或错误翻译成 HTTP。业务判断不写在这里。  
  
## 目录  
  
1. 第 1 章 挂接口  
2. 第 2 章 错误映射  
3. 第 3 章 流式返回  
4. 第 4 章 测接口  
5. 运行与验收  
6. 知识地图  
7. 自测  
8. 规范自检记录  
  
---  
  
## 第 1 章 挂接口  
  
### 1.0 进度卡  
  
**① 章节清单与状态**  
  
🔄 第1章 挂接口 ← 本章  
⏳ 第2章 错误映射  
⏳ 第3章 流式返回  
⏳ 第4章 测接口  
  
**② 数据现在在哪**  
  
`chunks` 表里有分块，`kb_search` 与测试能读；模型看不到它——`AgentExecutor` 只在 Python 脚本与测试里被调用，HTTP 层没有任何入口。  
  
**③ 本章做完后**  
  
`POST /api/v1/chat` 能返回答案，`curl` 与前端都能调用；`chunks` 表经 `kb_search` 对模型可见；但任何错误都返回 500。  
  
**④ 下一章要做**  
  
打开 `app/core/exceptions.py`，让错误映射成正确的状态码  
  
### 1.1 场景与动机  
  
本章做：把 `AgentExecutor` 挂到 `POST /api/v1/chat`。  
本章不做：错误映射、流式返回、接口测试。  
  
**业务落差演示（没有 HTTP 接口时）**  
  
- 同一句提问：`这个项目部署时用哪个端口？`  
- 当前结果：只能在项目目录执行 `python -c "...AgentExecutor().run('这个项目部署时用哪个端口？')"` 得到「8000」；浏览器或 `curl http://127.0.0.1:8000/api/v1/chat` 返回 `404 Not Found`。  
- 知识来源：知识库（chunks 表，经 `kb_search` 检索）。  
- 落差：答案存在，但只有装好环境、会跑脚本的人拿得到；产品没有入口。  
  
① 第 4 章交出的是 `app/agent/executor.py` 与 `tests/`，Agent 能在脚本和测试里跑，但只能被 Python 代码调用。  
② 换一台电脑、换一个前端、换一个同事，都要重新配环境才能问一句；10 个用户就是 10 份环境。  
③ 本章要给这个问题一个 URL。  
④ 选 FastAPI 路由加依赖注入：校验与文档自动生成，会话按请求创建。  
  
### 1.2 数据建模  
  
① 业务对象清单（只在请求与响应期间存在，不落库）：  
  
- `ChatRequest`：一次聊天请求，含问题与检索条数。  
- `ChatResponse`：一次聊天响应，含答案与耗时。  
  
② 字段表：  
  
| 对象 | 字段 | 类型 | 为什么需要 | 约束 |  
|---|---|---|---|---|  
| ChatRequest | question | 字符串 | 用户的问题 | 非空、1～2000 字 |  
| ChatRequest | top_k | 整数 | 控制检索条数，影响上下文长度 | 1～20，默认 5 |  
| ChatResponse | answer | 字符串 | 最终答复 | 非空 |  
| ChatResponse | elapsed_ms | 整数 | 前端显示耗时、排查慢请求 | ≥0 |  
  
③ 关系与基数：一次请求对应一次响应，1:1。没有持久化关系，所以不建表。  
  
④ 替代建模与否决：把问题放在查询参数里（`GET /chat?q=...`）。否决理由：长问题会超 URL 长度限制，含换行的文本无法直接拼进 URL，也无法在请求体里扩展 `top_k` 等字段。  
  
⑤ 呼应动机：`question` 有非空与长度约束，空问题在入口就返回 422，不会传到模型那里浪费一次调用。  
  
### 1.3 最小可运行版  
  
本段做什么：定义两个请求响应模型，并建一个能返回答案的路由。  
  
```python  
# 本段：最小可运行版，两个 DTO 加一个路由  
from pydantic import BaseModel  
from fastapi import APIRouter, Depends  
from sqlalchemy.orm import Session  
from app.db.session import get_db  
from app.agent.executor import AgentExecutor  
from app.agent.llm_client import LLMClient  
from app.agent.registry import ToolRegistry  
from app.agent.tools.kb_search import KBSearchTool  
  
class ChatRequest(BaseModel):  
    """一次聊天请求。"""  
    question: str  
    top_k: int = 5  
  
class ChatResponse(BaseModel):  
    """一次聊天响应。"""  
    answer: str  
  
router = APIRouter(prefix="/chat")  
  
@router.post("", response_model=ChatResponse)  
def chat(req: ChatRequest, db: Session = Depends(get_db)) -> ChatResponse:  
    """接收问题，调用 Agent，返回答案。"""  
    registry = ToolRegistry()  
    registry.register(KBSearchTool(db))  
    executor = AgentExecutor(LLMClient(), registry)  
    return ChatResponse(answer=executor.run(req.question))  
```  
  
### 1.4 分段讲解  
  
| 代码 | 含义 |  
|---|---|  
| `class ChatRequest(BaseModel)` | 请求体模型，字段即 JSON 字段 |  
| `router = APIRouter(prefix="/chat")` | 建路由分组，所有路径自动加 `/chat` 前缀 |  
| `@router.post("", response_model=ChatResponse)` | 声明 POST 接口与出参模型 |  
| `db: Session = Depends(get_db)` | 依赖注入：由框架调用 `get_db` 并把会话传进来 |  
| `KBSearchTool(db)` | 把当前请求的会话注入工具 |  
| `return ChatResponse(...)` | 返回模型实例，框架自动转 JSON 并校验 |  
  
**知识点 1：依赖注入 `Depends`**  
  
定义：`Depends(函数)` 让框架在调用路由前先执行该函数，把返回值作为参数传入。  
  
定式（要背）：  
  
```python  
def route(db: Session = Depends(get_db)):  
    ...  
```  
  
适用：数据库会话、当前用户、客户端实例。例题：`get_db` 每个请求执行一次，请求结束自动关闭。  
  
**知识点 2：请求与响应模型**  
  
定义：请求模型声明 JSON 入参的字段与约束；响应模型声明出参结构，并由 `response_model` 校验。  
  
定式（要背）：  
  
```python  
class Req(BaseModel):  
    question: str  
  
@router.post("", response_model=Resp)  
def route(req: Req) -> Resp: ...  
```  
  
适用：所有对外接口。例题：请求缺 `question` 时自动返回 422，不进函数体。  
  
**知识点 3：路由前缀**  
  
定义：`APIRouter(prefix=...)` 给组内所有路径加统一前缀，`include_router` 时再叠加应用级前缀。  
  
定式（要背）：  
  
```python  
router = APIRouter(prefix="/chat")  
app.include_router(router, prefix="/api/v1")  
```  
  
适用：按业务或版本分组路由。例题：最终路径为 `/api/v1/chat`。  
  
**知识点 4：每个请求一个执行器**  
  
定义：执行器不跨请求共享，因为它持有当前请求的数据库会话。  
  
定式（要背）：  
  
```python  
registry = ToolRegistry()  
registry.register(KBSearchTool(db))  
executor = AgentExecutor(LLMClient(), registry)  
```  
  
适用：工具依赖请求级资源时。例题：两个并发请求各自拿到不同会话，互不干扰。  
  
### 1.5 指出缺陷  
  
以下缺陷只评「挂接口」这一动作，均可在 1.3 代码中定位。  
  
- **D1**：`ChatRequest.question` 只写 `str`，没有长度与非空约束 → 空字符串也会进入模型，浪费一次调用并返回「请提供问题」之类无用答案。  
- **D2**：`ChatResponse` 只有 `answer`，没有 `elapsed_ms` → 前端无法显示耗时，慢请求只能靠猜。  
- **D3**：`AgentExecutor(LLMClient(), registry)` 在函数内每次新建 → 每个请求都新建 HTTP 客户端与连接，配置重复读取；测试也无法替换。  
- **D4**：路由直接 `return ChatResponse(...)`，异常未处理 → 模型超时、数据库报错都变成框架默认 500，响应体是 `{"detail":"Internal Server Error"}`，用户拿不到任何可操作信息。  
  
### 1.6 结构设计  
  
① 文件清单：  
  
| 文件路径 | 一句话职责 | 依赖谁 |  
|---|---|---|  
| `app/api/v1/schemas.py` | 定义请求与响应模型 | `pydantic` |  
| `app/api/v1/chat.py` | 路由与依赖提供者 | `schemas`、`agent.*`、`db.session` |  
| `app/main.py`（修改） | 挂载 chat 路由 | `chat.router` |  
  
② 拆分理由：  
  
- `schemas.py`：解决 D1、D2，字段约束与响应字段集中一处，接口与测试共用同一份定义。  
- `chat.py`：解决 D3，把执行器构建收进 `get_executor` 依赖，路由只调用。  
- `main.py`：注册路由，解决「接口存在但没有暴露」。  
  
③ 不这么拆会怎样：模型写在路由文件里，测试导入路由就会连带导入 Agent 与数据库；执行器构建散在多个接口，改工具注册要动 5 处。  
  
④ 依赖图：  
  
```  
app/api/v1/schemas.py  
        ↑  
app/api/v1/chat.py  →  app/agent/executor.py  
        ↑                     ↑  
app/main.py            app/db/session.py  
```  
  
⑤ 分工：设计决策回答「用 GET 还是 POST、是否加 DTO」；结构设计回答「模型、路由、挂载分三处」。  
  
### 1.7 完整版  
  
**设计决策：POST + Pydantic 模型，而不是 GET 查询参数**  
  
① 痛点：D1 的空问题进模型、D2 的耗时不可见、D3 的客户端重复创建、D4 的错误不可读。  
  
② 取舍：  
  
- 方案 A：`GET /chat?q=...`。适用于一句话、无副作用的简单查询。否决理由：长问题受 URL 长度限制（约 2000 字符），换行与特殊字符要反复转义，扩展字段只能不断加查询参数。  
- 方案 B：`POST` + 裸 `dict`。适用于内部临时接口。否决理由：无字段校验，缺字段要到函数体内才报 `KeyError`，也无法生成接口文档。  
  
③ 触发条件：请求体含长文本或多个字段时用 POST + 模型；只有一两个短参数、且无副作用时可用 GET。  
  
**文件 1：`app/api/v1/schemas.py`**  
  
本段做什么：定义请求与响应模型，给字段加约束。  
  
```python  
# 本段：接口模型  
"""聊天接口的请求与响应模型。"""  
from pydantic import BaseModel, ConfigDict, Field   # 第三方库 · 校验 · 模型与字段  
  
  
class ChatRequest(BaseModel):  
    """一次聊天请求。"""  
    model_config = ConfigDict(extra="forbid")  
    question: str = Field(min_length=1, max_length=2000, description="用户问题")  
    top_k: int = Field(default=5, ge=1, le=20, description="检索条数")  
  
  
class ChatResponse(BaseModel):  
    """一次聊天响应。"""  
    answer: str  
    elapsed_ms: int = Field(ge=0, description="处理耗时，毫秒")  
```  
  
| 代码 | 含义 |  
|---|---|  
| `min_length=1` | 空字符串直接 422 |  
| `ge` / `le` | 大于等于 / 小于等于 |  
| `description` | 出现在自动生成的接口文档里 |  
| `extra="forbid"` | 多传字段直接报错 |  
  
**文件 2：`app/api/v1/chat.py`**  
  
本段做什么：建路由，用依赖提供执行器。  
  
```python  
# 本段：聊天路由  
"""聊天接口。"""  
import time                                        # 标准库 · 时间 · 计耗时  
  
from fastapi import APIRouter, Depends             # 第三方库 · Web · 路由与依赖  
from sqlalchemy.orm import Session                 # 第三方库 · ORM · 会话类型  
  
from app.agent.executor import AgentExecutor       # 本项目 · 执行器  
from app.agent.llm_client import LLMClient         # 本项目 · 客户端  
from app.agent.registry import ToolRegistry        # 本项目 · 工具表  
from app.agent.tools.kb_search import KBSearchTool # 本项目 · 检索工具  
from app.api.v1.schemas import ChatRequest, ChatResponse   # 本项目 · 模型  
from app.db.session import get_db                  # 本项目 · 会话依赖  
  
router = APIRouter(prefix="/chat", tags=["chat"])  
_client = LLMClient()                              # 关键行：客户端进程级复用  
  
  
def get_executor(db: Session = Depends(get_db)) -> AgentExecutor:  
    """为每个请求构建执行器，工具绑定当前会话。"""  
    registry = ToolRegistry()  
    registry.register(KBSearchTool(db))  
    return AgentExecutor(_client, registry)  
  
  
@router.post("", response_model=ChatResponse, summary="问答")  
def chat(req: ChatRequest, executor: AgentExecutor = Depends(get_executor)) -> ChatResponse:  
    """接收问题，调用 Agent，返回答案与耗时。"""  
    start = time.perf_counter()  
    answer = executor.run(req.question)  
    return ChatResponse(answer=answer, elapsed_ms=int((time.perf_counter() - start) * 1000))  
```  
  
| 代码 | 含义 |  
|---|---|  
| `_client = LLMClient()` | 模块加载时建一次，复用连接与配置 |  
| `get_executor` | 依赖函数，返回请求级执行器 |  
| `Depends(get_executor)` | 框架先调依赖，再把结果传给路由 |  
| `perf_counter()` | 高精度计时，用于 `elapsed_ms` |  
  
**文件 3：`app/main.py`（追加两行）**  
  
本段做什么：把 chat 路由挂到应用。  
  
```python  
# 本段：挂载路由  
from app.api.v1 import chat                        # 本项目 · 聊天路由  
  
# 在 app = FastAPI(...) 之后：  
app.include_router(chat.router, prefix="/api/v1")  # 关键行：最终路径 /api/v1/chat  
```  
  
| 代码 | 含义 |  
|---|---|  
| `include_router(...)` | 把分组挂到应用，叠加前缀 |  
  
固定模板：`ChatRequest` / `ChatResponse` 的字段与约束、`get_executor` 依赖、路由签名。可组合写法：`top_k` 上限、是否加 `conversation_id`、是否记录用户。  
  
已知边界：`_client` 在模块导入时创建，若无密钥也不会报错，直到第一次调用才失败；这属于第 2 章的错误映射范围。  
  
调用链：`chat` 由 FastAPI 路由调用；它调用 `get_executor`，后者调用 `get_db`、`KBSearchTool`、`AgentExecutor`；`AgentExecutor.run` 内部再调 `LLMClient` 与工具；异常向上冒泡到第 2 章的全局处理器。  
  
### 1.8 结尾块  
  
**本章判据**：`curl -X POST .../api/v1/chat` 带上问题，能拿到 `{"answer": "...", "elapsed_ms": ...}`。  
  
**本章产出**：`app/api/v1/schemas.py`、`app/api/v1/chat.py`，修改 `app/main.py`。  
  
**下一章入口**：打开 `app/core/exceptions.py`，让错误映射成正确的状态码  
  
**常见错法 3 条**：  
  
1. 用 GET 传长问题。  
2. 请求模型不写约束，空问题进模型。  
3. 每个请求新建 `LLMClient`。  
  
**自检 3 问**：  
  
1. 空问题会在哪一步被拦住？  
2. 两个并发请求会共用同一个数据库会话吗？  
3. 前端能拿到这次请求的耗时吗？  
  
---  
  
## 第 2 章 错误映射  
  
### 2.0 进度卡  
  
**① 章节清单与状态**  
  
✅ 第1章 挂接口 → `app/api/v1/schemas.py`、`app/api/v1/chat.py`  
🔄 第2章 错误映射 ← 本章  
⏳ 第3章 流式返回  
⏳ 第4章 测接口  
  
**② 数据现在在哪**  
  
`POST /api/v1/chat` 能返回答案，`curl` 与前端都能调用；`chunks` 表经 `kb_search` 对模型可见；但任何错误都返回 500。  
  
**③ 本章做完后**  
  
错误已分层：参数错 422、文档不存在 404、模型不可用 502、未知 500，响应体统一为 `{code, message}`；答案仍一次性返回。  
  
**④ 下一章要做**  
  
打开 `app/api/v1/chat.py`，把一次性响应改成流式返回  
  
### 2.1 场景与动机  
  
本章做：给接口错误定义状态码与统一响应体。  
本章不做：改接口契约、改流式、写测试。  
  
**业务落差演示（错误没有分类时）**  
  
- 同一句提问：`这个项目部署时用哪个端口？`  
- 当前结果：把 `.env` 里的 `LLM_API_KEY` 改成空值后再问，接口返回 `500`，响应体是 `{"detail":"Internal Server Error"}`，服务端日志里是一整段 `httpx.HTTPStatusError: Client error '401 Unauthorized'`。  
- 知识来源：联网（真实模型调用失败）。  
- 落差：用户不知道是自己问错了、服务挂了，还是密钥过期；前端只能弹「服务器错误」，重试也没有意义。  
  
① 第 1 章交出的是 `app/api/v1/chat.py`，路由能答，但所有失败都是一个 500。  
② 一次模型 401、一次参数为空、一次文档不存在，用户看到的都是同一句「服务器错误」，客服无法分流。  
③ 本章要给不同失败不同的状态码与可读信息。  
④ 选自定义异常加全局处理器：业务代码只抛，翻译集中一处。  
  
本章不涉及新数据结构。  
  
### 2.2 最小可运行版  
  
本段做什么：在路由里用 `try/except` 把所有异常都转成 500。  
  
```python  
# 本段：最小可运行版，统一 500  
from fastapi.responses import JSONResponse  
  
@router.post("", response_model=ChatResponse)  
def chat(req: ChatRequest, executor=Depends(get_executor)):  
    """接收问题，任何异常都返回 500。"""  
    try:  
        return ChatResponse(answer=executor.run(req.question), elapsed_ms=0)  
    except Exception as e:  
        return JSONResponse(status_code=500, content={"code": "ERROR", "message": str(e)})  
```  
  
### 2.3 分段讲解  
  
| 代码 | 含义 |  
|---|---|  
| `try/except Exception` | 捕获所有异常 |  
| `JSONResponse(status_code=500, ...)` | 手动构造响应 |  
| `str(e)` | 把异常转成字符串 |  
| `content={...}` | 自定义响应体 |  
  
**知识点 1：HTTP 状态码语义**  
  
定义：状态码表达"这次请求属于哪一类结果"，客户端据此决定重试、提示还是改参数。  
  
定式（要背）：  
  
```python  
400 参数不合法 ／ 404 资源不存在 ／ 422 字段校验失败 ／ 502 上游不可用 ／ 500 未知错误  
```  
  
适用：所有接口的错误分支。例题：模型超时用 502，而不是 500，网关才会按上游故障处理。  
  
**知识点 2：`HTTPException`**  
  
定义：FastAPI 内置异常，抛出后自动转成 `{"detail": ...}` 响应。  
  
定式（要背）：  
  
```python  
raise HTTPException(status_code=404, detail="文档不存在")  
```  
  
适用：单个接口的简单错误。例题：查不到文档时抛 404，响应体为 `{"detail":"文档不存在"}`。  
  
**知识点 3：全局异常处理器**  
  
定义：`app.add_exception_handler(异常类, 函数)` 把某类异常统一翻译成响应，业务代码只负责抛。  
  
定式（要背）：  
  
```python  
app.add_exception_handler(AppError, app_error_handler)  
```  
  
适用：多种错误需要统一格式时。例题：任何 `AppError` 子类都返回 `{code, message}`。  
  
**知识点 4：统一错误响应结构**  
  
定义：所有错误响应使用同一组字段，前端只需写一套解析。  
  
定式（要背）：  
  
```python  
{"code": "UPSTREAM_ERROR", "message": "模型服务不可用"}  
```  
  
适用：任何对外 API。例题：前端按 `code` 判断是否可重试，不解析自然语言。  
  
### 2.4 指出缺陷  
  
以下缺陷只评「错误映射」这一动作，均可在 2.2 代码中定位。  
  
- **D1**：`except Exception` 把所有错误映射为 `500` → 参数为空、文档不存在、模型 401 都是 500，客户端无法区分「改参数」与「稍后重试」。  
- **D2**：`"message": str(e)` 直接把异常文本返回 → `httpx` 的 401 响应里可能带请求地址与头部片段，泄漏内部信息。  
- **D3**：捕获逻辑写在路由里 → 第 3 章再加流式接口时要复制一遍，两个接口的错误格式会漂移。  
- **D4**：未处理 `RequestValidationError` → 字段校验失败返回框架默认的 `{"detail":[{"loc":...}]}`，与 `{code, message}` 两套结构并存。  
  
### 2.5 结构设计  
  
① 文件清单：  
  
| 文件路径 | 一句话职责 | 依赖谁 |  
|---|---|---|  
| `app/core/exceptions.py` | 异常类与全局处理器 | `fastapi`、`logging` |  
| `app/main.py`（修改） | 注册处理器 | `exceptions` |  
| `app/api/v1/chat.py`（修改） | 路由删掉 `try/except` | `exceptions` |  
  
② 拆分理由：  
  
- `exceptions.py`：解决 D1、D2、D4，状态码与文案集中定义，响应体只在一处生成。  
- `main.py`：解决注册缺失，处理器不注册等于没写。  
- `chat.py`：解决 D3，路由不再捕获，新增流式接口自动复用同一套映射。  
  
③ 不这么拆会怎样：每个路由一套 `try/except`，第 3 章的流式接口复制第二套，两个接口的字段名一个叫 `message` 一个叫 `msg`，前端要写两套判断。  
  
④ 依赖图：  
  
```  
app/core/exceptions.py  
        ↑  
app/main.py（注册）  
        ↑  
app/api/v1/chat.py（只 raise，不捕获）  
```  
  
⑤ 分工：设计决策回答「用 HTTPException 还是自定义异常体系」；结构设计回答「异常类、注册、路由三处如何分工」。  
  
### 2.6 完整版  
  
**设计决策：自定义异常 + 全局处理器，而不是逐接口 HTTPException**  
  
① 痛点：D1 全是 500、D2 泄漏内部文本、D3 捕获重复、D4 两套响应结构。  
  
② 取舍：  
  
- 方案 A：每个接口 `raise HTTPException`。适用于只有 1～2 个接口。否决理由：状态码与文案散落，新增接口要重复判断；响应体是 `detail`，与自定义结构不一致。  
- 方案 B：只在网关层统一映射。适用于服务内部约定已稳定。否决理由：网关看不到业务异常类型，只能按状态码猜，且本地开发时没有网关。  
  
③ 触发条件：接口 ≥3 或错误类型 ≥3 时引入统一体系；单接口脚本用 `HTTPException` 即可。  
  
**文件 1：`app/core/exceptions.py`（完整版）**  
  
本段做什么：定义异常层级，并把每类异常翻译成状态码与统一响应体。  
  
```python  
# 本段：异常层级  
"""业务异常与全局处理器。"""  
from fastapi import FastAPI, Request               # 第三方库 · Web · 应用与请求  
from fastapi.exceptions import RequestValidationError   # 第三方库 · Web · 校验异常  
from fastapi.responses import JSONResponse         # 第三方库 · Web · JSON 响应  
  
from app.core.logging import get_logger            # 本项目 · 日志  
  
logger = get_logger(__name__)  
  
  
class AppError(Exception):  
    """所有业务异常的基类。"""  
    status_code = 500  
    code = "INTERNAL_ERROR"  
    message = "服务器内部错误"  
  
    def __init__(self, message: str | None = None):  
        """允许覆盖默认文案。"""  
        self.message = message or self.message  
        super().__init__(self.message)  
  
  
class NotFoundError(AppError):  
    """资源不存在。"""  
    status_code = 404  
    code = "NOT_FOUND"  
  
  
class InvalidInputError(AppError):  
    """业务参数不合法。"""  
    status_code = 400  
    code = "INVALID_INPUT"  
  
  
class UpstreamError(AppError):  
    """上游服务不可用。"""  
    status_code = 502  
    code = "UPSTREAM_ERROR"  
```  
  
| 代码 | 含义 |  
|---|---|  
| `status_code` / `code` / `message` | 类属性作为默认值，子类只覆盖差异 |  
| `message or self.message` | 传了用传的，没传用默认 |  
| `super().__init__(...)` | 让 `str(exc)` 可读 |  
  
**文件 1 第 2 段**  
  
本段做什么：写三个处理器，统一响应体。  
  
```python  
# 本段：处理器  
async def app_error_handler(request: Request, exc: AppError) -> JSONResponse:  
    """业务异常转统一响应。"""  
    logger.warning("业务异常 | path=%s | code=%s", request.url.path, exc.code)  
    return JSONResponse(status_code=exc.status_code,  
                        content={"code": exc.code, "message": exc.message})  
  
  
async def validation_error_handler(request: Request, exc: RequestValidationError) -> JSONResponse:  
    """字段校验失败转 422，结构与业务错误一致。"""  
    first = exc.errors()[0] if exc.errors() else {}  
    loc = ".".join(str(p) for p in first.get("loc", []))  
    return JSONResponse(status_code=422,  
                        content={"code": "VALIDATION_ERROR",  
                                 "message": f"参数校验失败: {loc}"})  
  
  
async def unhandled_error_handler(request: Request, exc: Exception) -> JSONResponse:  
    """未知异常记堆栈，对外只给笼统信息。"""  
    logger.exception("未捕获异常 | path=%s", request.url.path)  
    return JSONResponse(status_code=500,  
                        content={"code": "INTERNAL_ERROR", "message": "服务器内部错误"})  
  
  
def register_exception_handlers(app: FastAPI) -> None:  
    """把三个处理器注册到应用。"""  
    app.add_exception_handler(AppError, app_error_handler)  
    app.add_exception_handler(RequestValidationError, validation_error_handler)  
    app.add_exception_handler(Exception, unhandled_error_handler)  
```  
  
| 代码 | 含义 |  
|---|---|  
| `logger.warning` | 业务异常属预期分支，不记 error |  
| `exc.errors()[0]` | 取第一条校验错误 |  
| `logger.exception` | 自动附带完整堆栈 |  
| `add_exception_handler` | 异常类与处理函数配对 |  
  
**文件 2：`app/api/v1/chat.py`（路由删掉 try）**  
  
本段做什么：让路由只负责调用，异常交给处理器。  
  
```python  
# 本段：精简后的路由  
@router.post("", response_model=ChatResponse, summary="问答")  
def chat(req: ChatRequest, executor: AgentExecutor = Depends(get_executor)) -> ChatResponse:  
    """接收问题，调用 Agent；异常由全局处理器翻译。"""  
    start = time.perf_counter()  
    answer = executor.run(req.question)             # 关键行：不捕获，直接上抛  
    return ChatResponse(answer=answer, elapsed_ms=int((time.perf_counter() - start) * 1000))  
```  
  
| 代码 | 含义 |  
|---|---|  
| 无 `try/except` | 业务异常冒泡到全局处理器 |  
| `executor.run(...)` | `UpstreamError` 等异常由下层抛出 |  
  
固定模板：异常类三属性、三个处理器、注册函数。可组合写法：错误码枚举、是否带 `details`、是否上报监控。  
  
已知边界：`Exception` 兜底处理器在部分 ASGI 服务器上仍会打印一条服务端日志，这是预期行为；对外响应不受影响。  
  
调用链：路由抛出 `UpstreamError`；`register_exception_handlers` 在 `main.py` 启动时完成注册；处理器调用 `JSONResponse` 返回统一体；未捕获异常进 `unhandled_error_handler` 并记堆栈。  
  
### 2.7 结尾块  
  
**本章判据**：空问题返回 422、模型不可用返回 502、未知错误返回 500，三者响应体都是 `{code, message}`。  
  
**本章产出**：`app/core/exceptions.py`、修改 `app/main.py` 与 `app/api/v1/chat.py`。  
  
**下一章入口**：打开 `app/api/v1/chat.py`，把一次性响应改成流式返回  
  
**常见错法 3 条**：  
  
1. 所有异常都返回 500。  
2. 把 `str(e)` 原样返回给客户端。  
3. 校验错误用框架默认结构，与自定义结构并存。  
  
**自检 3 问**：  
  
1. 用户能只看 `code` 决定是否重试吗？  
2. 501 与 502 在我这里分别代表什么？  
3. 错误响应里会带出内部地址吗？  
  
---  
  
## 第 3 章 流式返回  
  
### 3.0 进度卡  
  
**① 章节清单与状态**  
  
✅ 第1章 挂接口 → `app/api/v1/schemas.py`、`app/api/v1/chat.py`  
✅ 第2章 错误映射 → `app/core/exceptions.py`、`app/main.py`  
🔄 第3章 流式返回 ← 本章  
⏳ 第4章 测接口  
  
**② 数据现在在哪**  
  
错误已分层：参数错 422、文档不存在 404、模型不可用 502、未知 500，响应体统一为 `{code, message}`；答案仍一次性返回。  
  
**③ 本章做完后**  
  
`POST /api/v1/chat/stream` 以 SSE 逐段返回，首字约 0.8 秒到达，总耗时不变；前端可用 `EventSource` 接收。  
  
**④ 下一章要做**  
  
打开 `tests/test_chat_api.py`，用 TestClient 固定接口行为  
  
### 3.1 场景与动机  
  
本章做：把答案改成流式事件返回。  
本章不做：改错误映射、改协议、写测试。  
  
**业务落差演示（一次性返回时）**  
  
- 同一句提问：`这个项目部署时用哪个端口？`  
- 当前结果：点击发送后页面空白约 **8.4 秒**（一次 ReAct 两轮 + 检索），然后整段答案一次性出现。  
- 知识来源：知识库（chunks 表，经 `kb_search` 检索）。  
- 落差：用户以为卡死，会刷新或重复点击；同一次请求被提交 2～3 次，模型调用与费用翻倍。  
  
① 第 2 章交出的是 `app/api/v1/chat.py`，错误已分层，但用户仍要干等完整答案。  
② 8.4 秒里没有任何反馈；刷新一次就多一次检索与模型调用。  
③ 本章要让过程可见。  
④ 选 SSE 加生成器：浏览器原生支持，逐段推送，不改协议。  
  
本章不涉及新数据结构；SSE 是传输格式，不是新对象。  
  
### 3.2 最小可运行版  
  
本段做什么：用 `StreamingResponse` 逐段返回三条纯文本。  
  
```python  
# 本段：最小可运行版，逐段返回  
import time  
from fastapi.responses import StreamingResponse  
  
def fake_events():  
    """产出三条事件文本。"""  
    for text in ["正在思考", "正在检索", "答案：8000"]:  
        yield f"data: {text}\n\n"  
        time.sleep(0.1)  
  
@router.post("/stream")  
def chat_stream(req: ChatRequest):  
    """以 SSE 格式返回。"""  
    return StreamingResponse(fake_events(), media_type="text/event-stream")  
```  
  
### 3.3 分段讲解  
  
| 代码 | 含义 |  
|---|---|  
| `def fake_events(): ... yield` | 生成器函数，调用时不执行，迭代时才逐条产出 |  
| `yield f"data: {text}\n\n"` | SSE 格式：`data:` 开头，两个换行结束一条 |  
| `StreamingResponse(生成器, media_type=...)` | 边产出边发送 |  
| `text/event-stream` | SSE 的 MIME 类型，浏览器据此识别 |  
  
**知识点 1：`StreamingResponse`**  
  
定义：`StreamingResponse` 接收一个可迭代对象，逐块发送，不等待全部内容就绪。  
  
定式（要背）：  
  
```python  
return StreamingResponse(gen(), media_type="text/event-stream")  
```  
  
适用：长耗时、需要进度反馈的接口。例题：首段在 0.1 秒到达，末段在 3 秒到达，客户端可边收边显示。  
  
**知识点 2：生成器与 `yield`**  
  
定义：含 `yield` 的函数返回生成器，每次迭代产出下一段，状态保留。  
  
定式（要背）：  
  
```python  
def gen():  
    yield "a"  
    yield "b"  
```  
  
适用：流式输出、大文件、分批处理。例题：`for x in gen()` 依次得到 `a`、`b`。  
  
**知识点 3：SSE 格式**  
  
定义：SSE 每条消息以 `data:` 开头，以空行结束；浏览器用 `EventSource` 解析。  
  
定式（要背）：  
  
```python  
yield f"data: {json.dumps(event, ensure_ascii=False)}\n\n"  
```  
  
适用：服务端单向推送进度与结果。例题：收到 `data: {"type":"final",...}` 时更新界面。  
  
**知识点 4：事件类型**  
  
定义：在同一流里用 `type` 字段区分状态、工具调用与最终答案。  
  
定式（要背）：  
  
```python  
{"type": "status", "message": "正在思考"}  
{"type": "tool", "tool": "kb_search"}  
{"type": "final", "answer": "8000"}  
```  
  
适用：多阶段任务的进度展示。例题：前端收到 `tool` 时显示"正在查资料"，收到 `final` 时替换为答案。  
  
### 3.4 指出缺陷  
  
以下缺陷只评「流式」这一动作，均可在 3.2 代码中定位。  
  
- **D1**：`fake_events` 的内容写死，与真实执行器无关 → 用户看到「答案：8000」是假数据，问别的问题也返回同一句。  
- **D2**：事件只有纯文本，没有 `type` → 前端无法区分「进度」与「最终答案」，只能把每段都当答案拼接。  
- **D3**：生成器不处理异常 → 执行器在第二轮抛 `UpstreamError` 时，连接直接断开，前端收不到任何错误事件，只看到加载圈一直转。  
- **D4**：没有发送结束标记 → 客户端无法判断流已正常结束，只能靠超时。  
  
### 3.5 结构设计  
  
① 文件清单：  
  
| 文件路径 | 一句话职责 | 依赖谁 |  
|---|---|---|  
| `app/agent/executor.py`（修改） | 增加 `stream`，按轮次产出事件 | `llm_client`、`registry` |  
| `app/api/v1/chat.py`（修改） | 新增 `/stream` 路由，把事件转成 SSE | `executor` |  
  
② 拆分理由：  
  
- `executor.stream`：解决 D1、D2、D3，事件由真实循环产生，类型明确，异常可转成 `error` 事件。  
- 路由层：解决 D4，统一加结束标记与 SSE 编码，执行器不关心传输格式。  
  
③ 不这么拆会怎样：在路由里复制一遍 ReAct 循环，两份循环逻辑会逐渐不同；把 SSE 编码写进执行器，执行器就绑定了 HTTP，测试与非 Web 调用都要改。  
  
④ 依赖图：  
  
```  
app/agent/executor.py (stream)  
        ↑  
app/api/v1/chat.py (/stream → SSE)  
```  
  
⑤ 分工：设计决策回答「轮询、WebSocket 还是 SSE」；结构设计回答「事件由执行器产、编码由路由做」。  
  
### 3.6 完整版  
  
**设计决策：SSE，而不是轮询或 WebSocket**  
  
① 痛点：D1 的假数据、D2 的无类型、D3 的断开、D4 的无结束标记；业务上是 8.4 秒空白与重复提交。  
  
② 取舍：  
  
- 方案 A：前端轮询任务状态。适用于无法保持长连接的场景。否决理由：每次轮询都有一次 HTTP 开销，间隔难定；短则浪费，长则延迟。  
- 方案 B：WebSocket。适用于双向实时通信。否决理由：本场景只有服务端单向推送，WebSocket 要额外处理握手、心跳与重连，复杂度高于收益。  
  
③ 触发条件：单向推送进度或结果时用 SSE；需要双向通信（如协作编辑）时才用 WebSocket；无法长连接时用轮询。  
  
**文件 1：`app/agent/executor.py`（新增 `stream`）**  
  
本段做什么：按轮次产出状态、工具与最终答案事件。  
  
```python  
# 本段：流式执行  
from collections.abc import Iterator               # 标准库 · 类型 · 迭代器注解  
  
    def stream(self, question: str) -> Iterator[dict]:  
        """按轮次产出事件，最后一条为 final 或 error。"""  
        messages = [  
            ChatMessage(role="system", content=build_system_prompt(self.registry.specs())),  
            ChatMessage(role="user", content=question),  
        ]  
        try:  
            for step in range(1, self.max_steps + 1):  
                yield {"type": "status", "step": step, "message": "正在思考"}  
                raw = self.client.chat(messages)  
                try:  
                    decision = AgentDecision.model_validate_json(raw)  
                except ValidationError:  
                    messages.append(ChatMessage(role="assistant", content=raw))  
                    messages.append(ChatMessage(role="user", content="只输出合法 JSON。"))  
                    continue  
                if decision.action == "final":  
                    yield {"type": "final", "answer": decision.answer or ""}  
                    return  
                yield {"type": "tool", "step": step, "tool": decision.tool}  
                try:  
                    observation = self.registry.run(decision.tool or "", decision.args)  
                except Exception as e:  
                    observation = f"工具执行失败: {e}"  
                messages.append(ChatMessage(role="assistant", content=raw))  
                messages.append(ChatMessage(role="user", content=f"观察结果: {observation[:500]}"))  
            yield {"type": "final", "answer": "达到最大步数，未能得出结论"}  
        except UpstreamError as e:  
            yield {"type": "error", "message": str(e)}   # 关键行：异常也走事件  
```  
  
| 代码 | 含义 |  
|---|---|  
| `Iterator[dict]` | 返回迭代器，每次一个事件字典 |  
| `yield {"type":"status"}` | 先告诉前端"在思考" |  
| `yield {"type":"tool"}` | 告诉前端正在查什么 |  
| `yield {"type":"final"}` | 结束事件，带答案 |  
| `except UpstreamError` | 上游失败也转成事件，不直接断流 |  
  
**文件 2：`app/api/v1/chat.py`（新增流式路由）**  
  
本段做什么：把事件编码成 SSE 并返回。  
  
```python  
# 本段：SSE 路由  
import json                                        # 标准库 · JSON · 事件序列化  
  
from fastapi.responses import StreamingResponse    # 第三方库 · Web · 流式响应  
  
  
def _to_sse(events) -> Iterator[str]:  
    """把事件字典编码为 SSE 文本。"""  
    for event in events:  
        yield f"data: {json.dumps(event, ensure_ascii=False)}\n\n"  
    yield "data: [DONE]\n\n"                       # 关键行：结束标记  
  
  
@router.post("/stream", summary="流式问答")  
def chat_stream(req: ChatRequest, executor: AgentExecutor = Depends(get_executor)):  
    """按 SSE 逐段返回状态与答案。"""  
    return StreamingResponse(  
        _to_sse(executor.stream(req.question)),  
        media_type="text/event-stream",            # 关键行：SSE 类型  
        headers={"Cache-Control": "no-cache"},  
    )  
```  
  
| 代码 | 含义 |  
|---|---|  
| `_to_sse` | 生成器，逐条编码 |  
| `json.dumps(..., ensure_ascii=False)` | 保留中文 |  
| `\n\n` | SSE 一条消息的结束符 |  
| `[DONE]` | 自定义结束标记，前端识别后关闭连接 |  
| `Cache-Control: no-cache` | 防止代理缓存流 |  
  
固定模板：`stream` 的事件产出顺序、`_to_sse` 的编码、`StreamingResponse` 的三个参数。可组合写法：是否心跳、是否带 `step_id`、是否压缩。  
  
已知边界：本册流的是"轮次事件"，不是逐字 token；要做逐字输出，需把 `LLMClient.chat` 换成流式接口，本册不展开。  
  
调用链：`chat_stream` 由路由调用；它调用 `_to_sse`，后者迭代 `executor.stream`；`executor.stream` 调用 `LLMClient.chat` 与 `ToolRegistry.run`；`UpstreamError` 在生成器内转成 `error` 事件，不再交给全局处理器。  
  
### 3.7 结尾块  
  
**本章判据**：`curl -N` 能看到多条 `data:`，首条在 1 秒内到达，最后一条是 `final` 或 `error`，末尾有 `[DONE]`。  
  
**本章产出**：修改 `app/agent/executor.py`、`app/api/v1/chat.py`。  
  
**下一章入口**：打开 `tests/test_chat_api.py`，用 TestClient 固定接口行为  
  
**常见错法 3 条**：  
  
1. 用假数据冒充流式。  
2. 事件不带 `type`，前端无法区分进度与答案。  
3. 流里抛异常直接断连，前端只看到转圈。  
  
**自检 3 问**：  
  
1. 第一条事件多久到达？  
2. 上游失败时，前端能收到错误事件吗？  
3. 前端怎么知道流结束了？  
  
---  
  
## 第 4 章 测接口  
  
### 4.0 进度卡  
  
**① 章节清单与状态**  
  
✅ 第1章 挂接口 → `app/api/v1/schemas.py`、`app/api/v1/chat.py`  
✅ 第2章 错误映射 → `app/core/exceptions.py`、`app/main.py`  
✅ 第3章 流式返回 → `app/agent/executor.py`、`app/api/v1/chat.py`  
🔄 第4章 测接口 ← 本章  
  
**② 数据现在在哪**  
  
`POST /api/v1/chat/stream` 以 SSE 逐段返回，首字约 0.8 秒到达，总耗时不变；前端可用 `EventSource` 接收。  
  
**③ 本章做完后**  
  
`pytest` 覆盖 200、422、502 与流式四类路径，全程不联网；接口行为可回归。  
  
**④ 下一章要做**  
  
打开 `Dockerfile`，把服务打包成可部署镜像  
  
### 4.1 场景与动机  
  
本章做：用 TestClient 固定接口行为。  
本章不做：改业务逻辑、改流式协议、测真实模型质量。  
  
**业务落差演示（接口没有测试时）**  
  
- 同一句提问：`这个项目部署时用哪个端口？`  
- 当前结果：CI 环境没有 `LLM_API_KEY`，登录态接口的冒烟测试在调用模型处返回 `502`，整条流水线红灯；为了让它变绿，有人把断言改成「只要不是 500 就算通过」，错误映射就此失去保护。  
- 知识来源：联网（真实模型）。  
- 落差：不是接口坏了，而是「对错无法判断」；一旦有人改坏状态码，没有任何东西会拦住。  
  
① 第 3 章交出的是 `app/api/v1/chat.py`，接口与流式都能用，但验证靠手动 `curl`。  
② 手动验证一次要 8 秒；改一处错误映射要重新点 4 种情况，漏一种就上线。  
③ 本章要让这些判断自动执行。  
④ 选 TestClient 加依赖覆盖：进程内调用，不启服务、不联网。  
  
本章不涉及新数据结构。  
  
### 4.2 最小可运行版  
  
本段做什么：用 TestClient 直接请求真实接口，不替换任何依赖。  
  
```python  
# 本段：最小可运行版，直连真实依赖  
from fastapi.testclient import TestClient  
from app.main import app  
  
def test_chat_ok():  
    """请求真实接口，断言返回 200。"""  
    with TestClient(app) as client:  
        resp = client.post("/api/v1/chat", json={"question": "这个项目部署时用哪个端口？"})  
    assert resp.status_code == 200  
```  
  
### 4.3 分段讲解  
  
| 代码 | 含义 |  
|---|---|  
| `TestClient(app)` | 进程内测试客户端，无需启动 uvicorn |  
| `with ... as client` | 进入时触发 `lifespan`，退出时清理 |  
| `client.post(url, json=...)` | 发请求，`json=` 自动序列化 |  
| `resp.status_code` | 断言状态码 |  
  
**知识点 1：`TestClient`**  
  
定义：`TestClient` 用进程内 ASGI 调用应用，不经过网络端口。  
  
定式（要背）：  
  
```python  
with TestClient(app) as client:  
    resp = client.post("/api/v1/chat", json={...})  
```  
  
适用：接口级测试。例题：`resp.status_code` 为 200，耗时通常小于 50 毫秒。  
  
**知识点 2：`dependency_overrides`**  
  
定义：用替身替换指定依赖函数，优先于真实依赖执行，测试结束需清空。  
  
定式（要背）：  
  
```python  
app.dependency_overrides[get_executor] = lambda: FakeExecutor()  
app.dependency_overrides.clear()  
```  
  
适用：替换数据库、外部客户端、执行器。例题：覆盖后接口拿到假执行器，不触碰模型与数据库。  
  
**知识点 3：接口测试的断言对象**  
  
定义：接口测试断言的是状态码、响应体结构与关键字段，不是内部实现。  
  
定式（要背）：  
  
```python  
assert resp.status_code == 422  
assert resp.json()["code"] == "VALIDATION_ERROR"  
```  
  
适用：所有接口用例。例题：空问题返回 422 且 `code` 固定，改内部校验方式不影响测试。  
  
**知识点 4：流式响应的读取**  
  
定义：`TestClient` 可用 `client.stream(...)` 逐块读取响应体。  
  
定式（要背）：  
  
```python  
with client.stream("POST", "/api/v1/chat/stream", json={...}) as r:  
    body = "".join(r.iter_text())  
```  
  
适用：SSE、大文件下载。例题：`body` 中含 `data: {"type": "final"`。  
  
### 4.4 指出缺陷  
  
以下缺陷只评「测接口」这一动作，均可在 4.2 代码中定位。  
  
- **D1**：`TestClient(app)` 不替换执行器 → 测试真调模型，CI 无密钥时返回 502，用例失败；本地每次约 8 秒。  
- **D2**：只断言 `status_code == 200` → 422、502、500 三条错误分支无覆盖，改坏状态码不会被发现。  
- **D3**：断言对象是整体响应 → 响应体新增一个字段就红，实际行为没变。  
- **D4**：没有流式用例 → `/stream` 的事件格式、结束标记被改坏时无人拦截。  
  
### 4.5 结构设计  
  
① 文件清单：  
  
| 文件路径 | 一句话职责 | 依赖谁 |  
|---|---|---|  
| `tests/conftest.py`（修改） | 提供 `client` 与假执行器 | `app.main`、`agent.executor` |  
| `tests/test_chat_api.py` | 四类接口契约测试 | `conftest` |  
  
② 拆分理由：  
  
- `conftest.py`：解决 D1，统一覆盖依赖，所有接口测试共用；解决 D4，假执行器同时提供 `stream`。  
- `test_chat_api.py`：解决 D2、D3，按状态码与事件类型分用例，断言结构与关键字段。  
  
③ 不这么拆会怎样：每个测试文件各自写覆盖与清理，漏掉 `clear()` 会污染后续用例；假执行器复制多份，`stream` 行为逐渐不一致。  
  
④ 依赖图：  
  
```  
app/main.py  
    ↑  
tests/conftest.py (client, fake executor)  
    ↑  
tests/test_chat_api.py  
```  
  
⑤ 分工：设计决策回答「真实服务、进程内客户端还是 mock 网络」；结构设计回答「夹具与用例分两个文件」。  
  
### 4.6 完整版  
  
**设计决策：TestClient + 依赖覆盖，而不是启动真实服务**  
  
① 痛点：D1 的真实调用、D2 的分支缺失、D3 的脆弱断言、D4 的流式未测。  
  
② 取舍：  
  
- 方案 A：`requests` 打真实端口。适用于上线前端到端验收。否决理由：要启服务、占端口、清理进程；CI 中端口冲突会随机失败。  
- 方案 B：mock `httpx.post`。适用于只测客户端。否决理由：绕过路由、依赖注入与异常处理器，测不到本册要保证的东西。  
  
③ 触发条件：接口契约与错误映射用 TestClient；跨服务链路用真实环境冒烟；只测单个客户端时 mock 网络。  
  
**文件 1：`tests/conftest.py`（追加）**  
  
本段做什么：提供带假执行器的 `client`。  
  
```python  
# 本段：接口测试夹具  
import pytest                                      # 第三方库 · 测试框架 · fixture  
from fastapi.testclient import TestClient          # 第三方库 · 测试客户端  
  
from app.api.v1.chat import get_executor           # 本项目 · 被覆盖的依赖  
from app.main import app                           # 本项目 · 应用  
  
  
class FakeExecutor:  
    """替代真实执行器，按预设返回。"""  
  
    def __init__(self, answer: str = "应用端口是 8000", error: Exception | None = None):  
        """保存答案或待抛异常。"""  
        self.answer = answer  
        self.error = error  
  
    def run(self, question: str) -> str:  
        """返回答案或抛异常。"""  
        if self.error:  
            raise self.error  
        return self.answer  
  
    def stream(self, question: str):  
        """产出状态与最终答案事件。"""  
        yield {"type": "status", "message": "正在思考"}  
        yield {"type": "final", "answer": self.answer}  
  
  
@pytest.fixture()  
def client():  
    """提供覆盖了执行器的测试客户端。"""  
    app.dependency_overrides[get_executor] = lambda: FakeExecutor()  
    with TestClient(app) as c:  
        yield c  
    app.dependency_overrides.clear()               # 关键行：清空，避免污染  
```  
  
| 代码 | 含义 |  
|---|---|  
| `FakeExecutor` | 与真实执行器同名方法，可直接替换 |  
| `error` 参数 | 按需模拟 502 |  
| `dependency_overrides[...]` | 用 lambda 返回替身 |  
| `clear()` | 测试结束恢复真实依赖 |  
  
**文件 2：`tests/test_chat_api.py` 第 1 段**  
  
本段做什么：覆盖成功与参数错误。  
  
```python  
# 本段：200 与 422  
"""聊天接口契约测试。"""  
import pytest                                      # 第三方库 · 测试框架 · raises  
  
from app.core.exceptions import UpstreamError      # 本项目 · 异常  
from app.api.v1.chat import get_executor           # 本项目 · 依赖  
from tests.conftest import FakeExecutor            # 测试代码 · 替身  
  
  
def test_chat_ok(client):  
    """模拟：正常提问（正常路径）。  
    契约：200，响应含 answer 与 elapsed_ms。"""  
    resp = client.post("/api/v1/chat", json={"question": "这个项目部署时用哪个端口？"})  
    assert resp.status_code == 200  
    body = resp.json()  
    assert "8000" in body["answer"]  
    assert body["elapsed_ms"] >= 0  
  
  
def test_empty_question_returns_422(client):  
    """模拟：空问题（异常路径）。  
    契约：422，code 为 VALIDATION_ERROR。"""  
    resp = client.post("/api/v1/chat", json={"question": ""})  
    assert resp.status_code == 422  
    assert resp.json()["code"] == "VALIDATION_ERROR"  
```  
  
| 代码 | 含义 |  
|---|---|  
| `json={"question": ...}` | 请求体 |  
| `"8000" in body["answer"]` | 断言语义，不断言整句 |  
| `elapsed_ms >= 0` | 断言字段存在且合法 |  
| 空字符串 | 触发 `min_length=1` |  
  
**文件 2 第 2 段**  
  
本段做什么：覆盖上游失败与流式。  
  
```python  
# 本段：502 与流式  
def test_upstream_error_returns_502(client):  
    """模拟：模型不可用（异常路径）。  
    契约：502，code 为 UPSTREAM_ERROR，响应体不含堆栈。"""  
    from app.main import app  
    app.dependency_overrides[get_executor] = lambda: FakeExecutor(error=UpstreamError("模型服务不可用"))  
    resp = client.post("/api/v1/chat", json={"question": "这个项目部署时用哪个端口？"})  
    assert resp.status_code == 502  
    assert resp.json()["code"] == "UPSTREAM_ERROR"  
    assert "Traceback" not in resp.text  
  
  
def test_stream_events(client):  
    """模拟：流式提问（正常路径）。  
    契约：多条 data 事件，最后含 final 与 [DONE]。"""  
    with client.stream("POST", "/api/v1/chat/stream",  
                       json={"question": "这个项目部署时用哪个端口？"}) as r:  
        body = "".join(r.iter_text())  
    assert r.status_code == 200  
    assert '"type": "status"' in body  
    assert '"type": "final"' in body  
    assert "[DONE]" in body  
```  
  
| 代码 | 含义 |  
|---|---|  
| 覆盖为带 `error` 的替身 | 模拟上游故障 |  
| `"Traceback" not in resp.text` | 断言不泄漏堆栈 |  
| `client.stream(...)` | 读取流式响应 |  
| `iter_text()` | 逐块取文本 |  
| `[DONE]` | 断言结束标记存在 |  
  
固定模板：`FakeExecutor` 的 `run`／`stream`、`client` 夹具的覆盖与清理、四类断言。可组合写法：`raise_server_exceptions`、是否加耗时上限断言、是否测 404。  
  
已知边界：TestClient 的流式读取会一次性缓冲完整响应，测不出真实网络下的逐块到达时间；首字延迟需在真实环境测。  
  
调用链：`client` 夹具由 pytest 注入；`dependency_overrides` 让路由拿到 `FakeExecutor`；请求经路由、异常处理器后返回；断言失败直接使该用例失败，不触碰模型与数据库。  
  
### 4.7 结尾块  
  
**本章判据**：无网络、无密钥时，200、422、502、流式四类用例全部通过。  
  
**本章产出**：修改 `tests/conftest.py`，新增 `tests/test_chat_api.py`。  
  
**下一章入口**：打开 `Dockerfile`，把服务打包成可部署镜像  
  
**常见错法 3 条**：  
  
1. 测试直连真实模型。  
2. 只断言 200，不测错误分支。  
3. 覆盖依赖后忘记 `clear()`。  
  
**自检 3 问**：  
  
1. CI 没有密钥时用例能通过吗？  
2. 把 502 改成 500，会有用例红吗？  
3. 流式的结束标记被删掉，能发现吗？  
  
**下一册从哪接**：打开 `Dockerfile`，把服务打包成可部署镜像  
  
---  
  
## 运行与验收  
  
```bash  
pip install fastapi uvicorn httpx pydantic-settings pytest  
```  
  
文件清单：  
  
```  
app/api/v1/schemas.py  
app/api/v1/chat.py  
app/core/exceptions.py  
app/agent/executor.py  
app/main.py  
tests/conftest.py  
tests/test_chat_api.py  
```  
  
验收：  
  
```bash  
uvicorn app.main:app --reload --port 8000  
  
# 1) 普通问答  
curl -X POST http://127.0.0.1:8000/api/v1/chat \  
  -H "Content-Type: application/json" \  
  -d '{"question":"这个项目部署时用哪个端口？"}'  
  
# 2) 参数错误  
curl -i -X POST http://127.0.0.1:8000/api/v1/chat \  
  -H "Content-Type: application/json" -d '{"question":""}'  
  
# 3) 流式（-N 关闭缓冲）  
curl -N -X POST http://127.0.0.1:8000/api/v1/chat/stream \  
  -H "Content-Type: application/json" \  
  -d '{"question":"这个项目部署时用哪个端口？"}'  
  
# 4) 测试  
pytest -v  
```  
  
预期：普通问答返回含 `8000` 的 JSON；空问题返回 422 且 `code=VALIDATION_ERROR`；流式依次出现 `status`、`tool`、`final`、`[DONE]`；测试全绿。  
  
---  
  
## 知识地图  
  
| 知识点 | 落点 |  
|---|---|  
| 路由与依赖注入 | 1.4 知识点 1、3 |  
| 请求响应模型 | 1.4 知识点 2 |  
| HTTP 状态码 | 2.3 知识点 1 |  
| 全局异常处理 | 2.3 知识点 3、2.6 |  
| 统一响应结构 | 2.3 知识点 4 |  
| 流式响应 | 3.3 知识点 1 |  
| 生成器 | 3.3 知识点 2 |  
| SSE 协议 | 3.3 知识点 3、4 |  
| 接口测试 | 4.3 知识点 1、2、3、4 |  
  
## 自测  
  
1. `Depends(get_db)` 每个请求执行几次？会话什么时候关闭？  
2. 为什么空问题返回 422 而不是 400？  
3. 502 与 500 在语义上有什么区别？  
4. SSE 一条消息的结束符是什么？为什么需要 `[DONE]`？  
5. `dependency_overrides` 用完为什么必须 `clear()`？  
  
<details>  
<summary>答案要点</summary>  
  
1. 每个请求一次；请求结束（`get_db` 的 `finally`）关闭。  
2. 422 表示字段级校验失败，是框架与 OpenAPI 的约定；400 用于业务规则不合法。  
3. 502 表示上游服务不可用，客户端可稍后重试；500 表示本服务未知错误。  
4. 两个换行 `\n\n`；`[DONE]` 是自定义结束标记，让前端区分"正常结束"与"连接断开"。  
5. 覆盖是全局字典，不清空会污染后续用例，导致真实依赖被替身顶替。  
  
</details>  
  
---  
  
## 规范自检记录  
  
| 条目 | 证据（小节号） | 结论 |  
|---|---|---|  
| 1 交付 | 仅本册，末行停止 | 满足 |  
| 2 方案四段 | 本册为教材 | 不适用 |  
| 3 八段结构 + 进度卡 | 各章 N.0；1.1–1.8、2.1–2.7 等 | 满足 |  
| 4 场景与动机 + 业务落差 | 各章 N.1 均含落差演示与来源标注 | 满足 |  
| 5 数据建模五项 | 1.2 | 满足 |  
| 6 最小可运行版 | 1.3 含两个 DTO；其余章无省略 | 满足 |  
| 7 一章一动作与缺陷定位 | 1.5、2.4、3.4、4.4 | 满足 |  
| 8 结构设计 | 1.6、2.5、3.5、4.5，均引用真实 D 编号 | 满足 |  
| 9 知识点三件套 | 1.4、2.3、3.3、4.3 共 16 个 | 满足 |  
| 10 设计决策三段 | 1.7、2.6、3.6、4.6 | 满足 |  
| 11 代码与注释 | 各代码块 import 三标、类函数 docstring、分段导读 | 部分满足：知识点卡内 3～6 行定式模板未逐行注释，因模板需可直接抄写；正文代码均满足 |  
| 12 读者画像 | 全文 | 满足 |  
| 13 测试标注 | 4.6 每个测试 docstring 标明路径与契约 | 满足 |  
| 14 结尾块 | 各章 N.7 五项；4.7 含下一册从哪接 | 满足 |  
| 15 自检十项 | 下表 C1–C10 | 满足 |  
| 16 导航表与格式 | 本册地图为三列导航表；第 N 章／N.M | 满足 |  
| 17 详略 | 各章「已知边界」 | 满足 |  
| 18 纠正认知 | 本册地图「术语校准」 | 满足 |  
| D1 只给完整版 | 各章均有最小版 | 未违反 |  
| D2 省略用语 | 全文无 | 未违反 |  
| D3 不可运行代码 | 各段可整段复制 | 未违反 |  
| D4 抽象理由 | 落差均为用户可感知症状，工程收益标为次要 | 未违反 |  
| D5 多册同写 | 仅本册 | 未违反 |  
| D6 反问开头 | 无 | 未违反 |  
  
一致性检查：  
  
- **C1**：完整版出现的领域对象只有 `ChatRequest`、`ChatResponse`，均在 1.2 出现；`FakeExecutor` 为测试替身，不属领域对象。通过。  
- **C2**：1.5 的 D1–D4 指向 1.3 的 `ChatRequest`、`ChatResponse`、`AgentExecutor(...)`、`return` 行；2.4 指向 2.2 的 `except Exception` 与 `str(e)`；3.4 指向 3.2 的 `fake_events` 与 `yield`；4.4 指向 4.2 的 `TestClient` 与 `assert`。通过。  
- **C3**：1.6 引用 D1–D4；2.5、3.5、4.5 均引用 D1–D4，全部真实存在。通过。  
- **C4**：1.5 全属挂接口；2.4 全属错误映射；3.4 全属流式；4.4 全属接口测试。无越界。通过。  
- **C5**：1.3 最小版两个对象与 1.2 名称、数量一致；其余章已在 N.1 写「本章不涉及新数据结构」。通过。  
- **C6**：每条缺陷均以字段、类名或函数名定位。通过。  
- **C7**：1.0④＝导航表第2行＝1.8「打开 `app/core/exceptions.py`，让错误映射成正确的状态码」；2.0④＝第3行＝2.7；3.0④＝第4行＝3.7；4.0④＝「下一册」行＝4.7 及「下一册从哪接」。一字不差。通过。  
- **C8**：2.0②＝1.0③；3.0②＝2.0③；4.0②＝3.0③，逐句相同；1.0②以第四册封版状态为基线。通过。  
- **C9**：1.1 给出「只能脚本调用、HTTP 404」（知识库，经 Agent 检索）；2.1 给出「401 变 500，堆栈外泄」（联网）；3.1 给出「8.4 秒空白、重复提交」（知识库）；4.1 给出「CI 无密钥 502、断言被放宽」（联网）。来源均已标注。通过。  
- **C10**：各章 N.0② 均写了两件事：位置（`POST /api/v1/chat`、`chunks` 表、错误层、SSE 接口）与可读者（`curl`、前端、模型、pytest）；1.0 明写「模型看不到它」。通过。  
  
本册结束，未开始第六册。  
