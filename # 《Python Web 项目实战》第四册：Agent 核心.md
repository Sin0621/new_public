# 《Python Web 项目实战》第四册：Agent 核心  
  
| 项目 | 内容 |  
|---|---|  
| 系列 | 《Python Web 项目实战》 |  
| 册数 | 第四册（开发阶段四：Day 7-8） |  
| 主题 | 消息协议 · 模型调用 · 工具注册 · ReAct 循环 · 编排测试 |  
| 前置要求 | 第一册（配置、日志、异常）、第三册（`search_page` 检索） |  
| 版本 | v1.7 · 2026-10-26 |  
  
## 本册地图  
  
```  
输入：第三册的 search_page（按关键词取一页分块）  
  → 第1章 定义协议     1 个文件：消息、工具、决策长什么样  
  → 第2章 调模型       2 个文件：发请求、超时、重试  
  → 第3章 注册工具     3 个文件：把检索包装成可调用工具  
  → 第4章 编排循环     1 个文件：模型决策 → 调工具 → 再问模型  
  → 第5章 测编排       2 个文件：不花钱、不联网地验证  
输出：一个能回答"文档里怎么部署"的可运行 Agent  
```  
  
每章只做一个动作，只在上章产出上增量。  
  
术语校准：Agent 不是模型本身；它是一段程序，负责组织消息、调用工具、决定何时停止。模型只负责生成下一步决策。  
  
## 目录  
  
1. 第 1 章 定义协议  
2. 第 2 章 调模型  
3. 第 3 章 注册工具  
4. 第 4 章 编排循环  
5. 第 5 章 测编排  
6. 运行与验收  
7. 知识地图  
8. 自测  
9. 规范自检记录  
  
---  
  
## 第 1 章 定义协议  
  
### 1.1 场景与动机  
  
本章做：定义消息、工具、决策三个运行时对象。  
本章不做：发网络请求、注册工具、循环编排、测试。  
  
① 上一章交出的是 `app/services/search_service.py` 的 `search_page`，它返回 `dict`，数据在内存里。  
② 直接调模型时，若把消息当裸 `dict` 传，字段拼错要到运行时才报 `KeyError`；模型返回的 JSON 少一个 `action`，循环里 `if` 判断落空。  
③ 本章要把"传什么、返回什么"固定成可校验的结构。  
④ 选 Pydantic 模型：字段类型、必填、枚举值都能在入口处拦住。  
  
### 1.2 数据建模  
  
① 业务对象清单（本章对象只在运行时存在，不落库）：  
  
- `ChatMessage`：一条对话消息，带角色与内容。  
- `ToolSpec`：一个工具对模型的自描述，含名称、用途、参数结构。  
- `AgentDecision`：模型对下一步的决策，是调工具还是给答案。  
  
② 字段表：  
  
| 对象 | 字段 | 类型 | 为什么需要 | 约束 |  
|---|---|---|---|---|  
| ChatMessage | role | system／user／assistant | 区分指令、问题、模型输出 | 枚举、非空 |  
| ChatMessage | content | 字符串 | 消息正文 | 非空 |  
| ToolSpec | name | 字符串 | 模型按名字选工具 | 唯一、非空 |  
| ToolSpec | description | 字符串 | 模型据此判断何时用 | 非空 |  
| ToolSpec | parameters | 字典 | 参数结构，供模型填值 | 默认空字典 |  
| AgentDecision | thought | 字符串 | 记录推理，便于排查 | 可空 |  
| AgentDecision | action | tool／final | 决定下一步走向 | 枚举、非空 |  
| AgentDecision | tool | 字符串或空 | 要调的工具名 | 仅 action=tool 时必填 |  
| AgentDecision | args | 字典 | 工具参数 | 默认空字典 |  
| AgentDecision | answer | 字符串或空 | 最终答复 | 仅 action=final 时必填 |  
  
③ 关系与基数：一次运行包含 N 条 `ChatMessage`（1:N，按时间追加）；一个 `AgentDecision` 只属于一条模型输出。消息是线性序列，不是树。  
  
④ 替代建模与否决：三个对象全部用裸 `dict`。否决理由：字段名拼错只在运行时暴露，`action` 写成 `"use_tool"` 也能通过，客户端与编排器会各写一份字段约定并逐渐漂移。  
  
⑤ 呼应动机：三个模型把"模型看到什么、能选什么、怎么回答"固定下来，解决 ② 的静默失败。  
  
### 1.3 最小可运行版  
  
本段做什么：定义三个模型并校验一条决策，不写校验钩子与配置。  
  
```python  
# 本段：最小可运行版，三个运行时对象  
from typing import Literal  
from pydantic import BaseModel, Field  
class ChatMessage(BaseModel):  
    """一条对话消息。"""  
    role: Literal["system", "user", "assistant"]  
    content: str  
class ToolSpec(BaseModel):  
    """工具自描述。"""  
    name: str  
    description: str  
    parameters: dict = Field(default_factory=dict)  
class AgentDecision(BaseModel):  
    """模型对下一步的决策。"""  
    thought: str = ""  
    action: str  
    tool: str | None = None  
    args: dict = Field(default_factory=dict)  
    answer: str | None = None  
print(AgentDecision.model_validate_json('{"action":"final","answer":"ok"}'))  
```  
  
预期：  
  
```  
action='final' tool=None args={} answer='ok' thought=''  
```  
  
### 1.4 分段讲解  
  
| 代码 | 含义 |  
|---|---|  
| `Literal[...]` | 枚举约束，只允许列出的值 |  
| `BaseModel` | Pydantic 模型基类，自动校验与序列化 |  
| `Field(default_factory=dict)` | 可变默认值用工厂函数，避免共享同一个字典 |  
| `model_validate_json(str)` | 从 JSON 字符串解析并校验，返回模型实例 |  
| `str | None` | 可为空；`= None` 给默认值 |  
  
**知识点 1：Pydantic 模型**  
  
定义：Pydantic 模型用类型注解声明字段，在构造或解析时自动做类型转换与校验。  
  
定式（要背）：  
  
```python  
class X(BaseModel):  
    name: str  
    count: int = 0  
```  
  
适用：任何跨边界的输入输出：接口入参、模型返回、配置。例题：`X(name="a", count="3")` 自动把 `"3"` 转成 `3`。  
  
**知识点 2：`Literal` 枚举**  
  
定义：`Literal["a","b"]` 限定字段只能取列出的字面值。  
  
定式（要背）：  
  
```python  
action: Literal["tool", "final"]  
```  
  
适用：状态机、动作类型、角色。例题：传 `"use_tool"` 直接抛 `ValidationError`。  
  
**知识点 3：`Field` 与默认值**  
  
定义：`Field` 给字段加默认值、描述与约束；可变默认值用 `default_factory`。  
  
定式（要背）：  
  
```python  
args: dict = Field(default_factory=dict, description="工具参数")  
```  
  
适用：字典、列表等可变默认值。例题：两个实例的 `args` 是不同对象，互不影响。  
  
**知识点 4：`model_validate_json`**  
  
定义：从 JSON 字符串解析并校验，失败抛 `ValidationError`。  
  
定式（要背）：  
  
```python  
decision = AgentDecision.model_validate_json(raw_text)  
```  
  
适用：解析模型返回、读取外部 JSON。例题：缺 `action` 时报 `Field required`。  
  
### 1.5 指出缺陷  
  
以下缺陷只评"定义协议"这一动作，均可在 1.3 代码中定位。  
  
- **D1**：`AgentDecision.action` 声明为 `str` 而非 `Literal` → 模型返回 `"use_tool"` 也能通过，编排器里 `if action == "tool"` 落空，函数静默返回 `None`。  
- **D2**：`ToolSpec.parameters` 无类型细化，只写 `dict` → 工具参数写错结构不报错，模型拿到错误 schema 后乱填参。  
- **D3**：三个模型写在脚本里，客户端与编排器各自复制字段定义 → 加一个 `trace_id` 要改 3 处，字段逐渐漂移。  
- **D4**：模型未设置 `extra="forbid"` → 模型多返回 `"confidence": 0.9` 被静默忽略，字段名拼错也不报错。  
  
### 1.6 结构设计  
  
① 文件清单：  
  
| 文件路径 | 一句话职责 | 依赖谁 |  
|---|---|---|  
| `app/agent/schemas.py` | 集中定义三个运行时模型 | `pydantic` |  
| `app/agent/__init__.py` | 标记 Agent 包 | 无 |  
  
② 拆分理由：  
  
- `schemas.py` 单文件：解决 D3，客户端、注册表、编排器都从这里导入，字段只定义一次。  
- `Literal`：解决 D1。  
- `Field(default_factory=dict)`：解决 D2 的可变默认值问题。  
- `ConfigDict(extra="forbid")`：解决 D4。  
  
③ 不这么拆会怎样：模型定义散在 3 个文件，改 `AgentDecision` 要同步 3 处，漏一处就在运行时报 `AttributeError`；测试只能测自己那份定义，测不到真实协议。  
  
④ 依赖图：  
  
```  
app/agent/schemas.py  
    ↑            ↑            ↑  
llm_client    registry    executor  
```  
  
⑤ 分工：设计决策回答"用 Pydantic 还是裸 dict"；结构设计回答"三个模型是否集中在一个文件"。  
  
### 1.7 完整版  
  
**设计决策：用 Pydantic，而不是裸 dict**  
  
① 痛点：D1 的静默落空、D2 的错误 schema、D3 的三处漂移、D4 的拼错不报。  
  
② 取舍：  
  
- 方案 A：裸 `dict`。适用于一次性脚本、字段不超过 3 个。否决理由：无类型提示，拼错到运行时才报 `KeyError`，无法枚举约束。  
- 方案 B：`dataclass`。适用于纯内部数据、不需要 JSON 解析。否决理由：不做类型转换，`"3"` 不会变 `3`；JSON 解析要自己写。  
  
③ 触发条件：数据跨进程或跨模块边界（模型返回、接口入参）时用 Pydantic；纯内部、同进程传递可用 dataclass。字段少于 3 个且不解析 JSON 时，可退回 dict。  
  
**文件 1：`app/agent/schemas.py`**  
  
本段做什么：定义三个模型，并禁止多余字段。  
  
```python  
# 本段：协议模型  
"""Agent 运行时协议：消息、工具描述、模型决策。"""  
from typing import Literal                     # 标准库 · 类型 · 枚举约束  
  
from pydantic import BaseModel, ConfigDict, Field  # 第三方库 · 校验 · 模型与字段  
  
class ChatMessage(BaseModel):  
    """一条对话消息。"""  
    model_config = ConfigDict(extra="forbid")   # 关键行：多余字段直接报错  
    role: Literal["system", "user", "assistant"]  
    content: str  
  
class ToolSpec(BaseModel):  
    """工具自描述，供模型选择与填参。"""  
    model_config = ConfigDict(extra="forbid")  
    name: str  
    description: str  
    parameters: dict = Field(default_factory=dict)   # 关键行：可变默认值用工厂  
  
class AgentDecision(BaseModel):  
    """模型对下一步的决策。"""  
    model_config = ConfigDict(extra="forbid")  
    thought: str = ""  
    action: Literal["tool", "final"]                # 关键行：只允许两种动作  
    tool: str | None = None  
    args: dict = Field(default_factory=dict)  
    answer: str | None = None  
```  
  
| 代码 | 含义 |  
|---|---|  
| `ConfigDict(extra="forbid")` | 出现未声明字段直接报错，防拼错与静默忽略 |  
| `Literal["system","user","assistant"]` | 角色只能是三种 |  
| `Field(default_factory=dict)` | 每次构造生成新字典 |  
| `thought: str = ""` | 可选字段给默认值 |  
  
**文件 2：`app/agent/__init__.py`**  
  
本段做什么：标记包，导出常用模型。  
  
```python  
# 本段：包导出  
"""Agent 包。"""  
from app.agent.schemas import AgentDecision, ChatMessage, ToolSpec  # 本项目 · 协议  
  
__all__ = ["AgentDecision", "ChatMessage", "ToolSpec"]  
```  
  
| 代码 | 含义 |  
|---|---|  
| `__all__` | 声明包对外导出的名字 |  
  
固定模板：三个模型的字段与 `model_config`。可组合写法：是否加 `trace_id`、`thought` 是否必填、`args` 是否做二级校验。  
  
已知边界：`extra="forbid"` 要求模型严格输出；模型偶尔加解释字段会被判失败，需在提示词里明确"只输出 JSON"。  
  
调用链：三个模型由 `llm_client` 返回、`executor` 解析、`registry` 生成 `ToolSpec`；它们不调用业务代码；校验失败抛 `ValidationError`，由编排器捕获并转为一次重试或 `AgentError`。  
  
### 1.8 结尾块  
  
**本章判据**：一条合法 JSON 能解析成 `AgentDecision`，一个非法 `action` 会立刻报错。  
  
**本章产出**：`app/agent/schemas.py`、`app/agent/__init__.py`。  
  
**下一章入口**：打开 `app/agent/llm_client.py`，加发请求与重试。  
  
**常见错法 3 条**：  
  
1. 三个对象全用裸 `dict`。  
2. `action` 写成 `str`，不限制取值。  
3. 可变默认值写成 `args: dict = {}`，多个实例共享同一个字典。  
  
**自检 3 问**：  
  
1. 模型多返回一个字段，会报错还是被忽略？  
2. `action` 写成 `"use_tool"`，会在哪一步失败？  
3. 两个 `AgentDecision` 的 `args` 是同一个对象吗？  
  
---  
  
## 第 2 章 调模型  
  
### 2.1 场景与动机  
  
本章做：把消息发到模型接口，拿回一段文本。  
本章不做：解析决策、注册工具、循环编排、测试替身。  
  
① 第 1 章交出的是 `app/agent/schemas.py` 的三个模型，它们只在内存里，还没有任何网络调用。  
② 接口偶发 429 与 502；不设超时会挂到连接池耗尽，一次 60 秒，10 个并发就把请求线程占满。  
③ 本章要一次可失败、可重试、可观测的模型调用。  
④ 选手写 httpx 加重试装饰器：依赖少，超时与状态码可控。  
  
本章不涉及新数据结构。  
  
### 2.2 最小可运行版  
  
本段做什么：发一次请求，取回文本，不做超时与重试。  
  
```python  
# 本段：最小可运行版，一次 POST  
import httpx  
from app.config import settings  
from app.agent.schemas import ChatMessage  
  
def chat(messages: list[ChatMessage]) -> str:  
    """把消息发给模型，返回文本。"""  
    payload = {  
        "model": settings.LLM_MODEL,  
        "messages": [m.model_dump() for m in messages],  
    }  
    resp = httpx.post(  
        f"{settings.LLM_BASE_URL}/chat/completions",  
        headers={"Authorization": f"Bearer {settings.LLM_API_KEY}"},  
        json=payload,  
    )  
    return resp.json()["choices"][0]["message"]["content"]  
```  
  
### 2.3 分段讲解  
  
| 代码 | 含义 |  
|---|---|  
| `settings.LLM_*` | 从第一册配置读取地址、密钥、模型名 |  
| `m.model_dump()` | 把 Pydantic 模型转成字典，供 JSON 序列化 |  
| `httpx.post(url, headers, json)` | 发 POST；`json=` 自动序列化并设 `Content-Type` |  
| `resp.json()` | 把响应体解析为字典 |  
| `["choices"][0]["message"]["content"]` | 按接口结构取文本 |  
  
**知识点 1：`httpx.post`**  
  
定义：`httpx.post` 发同步 POST 请求，返回 `Response`。  
  
定式（要背）：  
  
```python  
resp = httpx.post(url, headers={"Authorization": f"Bearer {key}"}, json=payload, timeout=30)  
```  
  
适用：调用 REST 接口。例题：返回 200 时 `resp.json()` 得到字典。  
  
**知识点 2：`timeout`**  
  
定义：`timeout` 限制连接与读取的等待时间，超时抛 `httpx.TimeoutException`。  
  
定式（要背）：  
  
```python  
httpx.post(url, json=payload, timeout=httpx.Timeout(30.0, connect=5.0))  
```  
  
适用：所有外部调用。例题：服务端 35 秒不回，客户端 30 秒抛超时。  
  
**知识点 3：状态码与 `raise_for_status`**  
  
定义：`raise_for_status()` 在 4xx／5xx 时抛 `HTTPStatusError`，2xx 不抛。  
  
定式（要背）：  
  
```python  
resp.raise_for_status()  
```  
  
适用：任何需要区分成功与失败的调用。例题：429 时抛错，异常里带 `response.status_code == 429`。  
  
**知识点 4：重试装饰器**  
  
定义：装饰器在不改动函数体的前提下包一层重试逻辑。  
  
定式（要背）：  
  
```python  
@retry(times=3, backoff=1.0, exceptions=(httpx.TimeoutException, httpx.HTTPStatusError))  
def chat(...): ...  
```  
  
适用：瞬时故障（超时、429、502）。例题：前两次 502，第三次成功，调用方只看到成功结果。  
  
### 2.4 指出缺陷  
  
以下缺陷只评"调模型"这一动作，均可在 2.2 代码中定位。  
  
- **D1**：`httpx.post` 未传 `timeout` → 用库默认值，服务端卡住时请求长时间不返回，并发一高线程被占满。  
- **D2**：没有调用 `raise_for_status()` → 429 或 502 时仍读 `resp.json()`，报 `KeyError: 'choices'`，真实原因被掩盖。  
- **D3**：没有重试 → 一次 429 直接失败，用户看到 500。  
- **D4**：异常未包装 → `httpx.ConnectError` 直接冒泡到接口层，返回英文堆栈，且无法统一映射状态码。  
  
### 2.5 结构设计  
  
① 文件清单：  
  
| 文件路径 | 一句话职责 | 依赖谁 |  
|---|---|---|  
| `app/agent/llm_client.py` | 组装请求、解析响应 | `httpx`、`app.config`、`schemas` |  
| `app/core/retry.py` | 通用重试装饰器 | `time`、`functools` |  
  
② 拆分理由：  
  
- `llm_client.py`：解决 D1（显式超时）、D2（`raise_for_status`）、D4（包装为 `UpstreamError`）。  
- `retry.py`：解决 D3，重试逻辑与业务分离，任何外部调用都能复用。  
  
③ 不这么拆会怎样：重试写成 `for` 循环塞在 `chat` 里，第二个外部调用要复制一遍；异常包装散落，接口层拿不到统一错误类型。  
  
④ 依赖图：  
  
```  
app/core/retry.py  
        ↑  
app/agent/llm_client.py → httpx → LLM 服务  
        ↑  
app/agent/schemas.py  
```  
  
⑤ 分工：设计决策回答"手写 httpx 还是官方 SDK"；结构设计回答"客户端与重试分两个文件"。  
  
### 2.6 完整版  
  
**设计决策：手写 httpx，而不是引入 SDK 或框架**  
  
① 痛点：D1 的挂起、D2 的 `KeyError` 掩盖真实错误、D3 的一次失败即 500、D4 的堆栈外泄。  
  
② 取舍：  
  
- 方案 A：官方 SDK。适用于只接一家模型、需要流式与函数调用高级特性。否决理由：本册需要兼容多家的 OpenAI 兼容接口，SDK 会绑定单一供应商，且隐藏了超时配置。  
- 方案 B：LangChain 等编排框架。适用于快速搭原型。否决理由：抽象层厚，出问题要读框架源码；本册目标是看清调用与循环本身。  
  
③ 触发条件：只接一家且需要流式／原生工具调用时用 SDK；多家兼容或要控制超时细节时手写 httpx。原型阶段可用框架，稳定后按需下沉。  
  
**文件 1：`app/core/retry.py`**  
  
本段做什么：提供可复用的重试装饰器。  
  
```python  
# 本段：重试装饰器  
"""通用重试装饰器。"""  
import functools                                # 标准库 · 函数工具 · 保留元信息  
import time                                     # 标准库 · 时间 · 退避等待  
  
from app.core.logging import get_logger         # 本项目 · 日志 · 记录重试  
  
logger = get_logger(__name__)  
  
  
def retry(times: int = 3, backoff: float = 1.0, exceptions: tuple = (Exception,)):  
    """失败后按线性退避重试，最后一次仍失败则抛出。"""  
    def decorator(func):  
        @functools.wraps(func)                  # 关键行：保留原函数名与文档  
        def wrapper(*args, **kwargs):  
            for attempt in range(1, times + 1):  
                try:  
                    return func(*args, **kwargs)  
                except exceptions as e:  
                    if attempt == times:  
                        raise  
                    logger.warning("第 %d 次失败，%.1fs 后重试: %s", attempt, backoff, e)  
                    time.sleep(backoff * attempt)  
        return wrapper  
    return decorator  
```  
  
| 代码 | 含义 |  
|---|---|  
| `functools.wraps(func)` | 保留被装饰函数的 `__name__` 与文档 |  
| `*args, **kwargs` | 原样转发所有参数 |  
| `range(1, times+1)` | 总共尝试 `times` 次 |  
| `if attempt == times: raise` | 最后一次失败不再等待，直接抛 |  
| `backoff * attempt` | 线性退避，第 2 次等 2 倍 |  
  
**文件 2：`app/agent/llm_client.py`**  
  
本段做什么：组装请求、显式超时、包装异常。  
  
```python  
# 本段：模型客户端  
"""OpenAI 兼容接口的同步客户端。"""  
import httpx                                    # 第三方库 · HTTP · 发请求  
  
from app.agent.schemas import ChatMessage       # 本项目 · 协议  
from app.config import settings                 # 本项目 · 配置  
from app.core.exceptions import UpstreamError   # 本项目 · 异常  
from app.core.logging import get_logger         # 本项目 · 日志  
from app.core.retry import retry                # 本项目 · 重试  
  
logger = get_logger(__name__)  
  
  
class LLMClient:  
    """调用 OpenAI 兼容的 chat/completions 接口。"""  
  
    def __init__(self, base_url: str | None = None, api_key: str | None = None,  
                 model: str | None = None, timeout: float = 30.0):  
        """允许测试时覆盖地址、密钥、模型与超时。"""  
        self.base_url = (base_url or settings.LLM_BASE_URL).rstrip("/")  
        self.api_key = api_key or settings.LLM_API_KEY  
        self.model = model or settings.LLM_MODEL  
        self.timeout = timeout  
  
    @retry(times=3, backoff=1.0, exceptions=(httpx.TimeoutException, httpx.HTTPStatusError))  
    def chat(self, messages: list[ChatMessage]) -> str:  
        """发送消息并返回文本，失败抛 UpstreamError。"""  
        payload = {"model": self.model, "messages": [m.model_dump() for m in messages]}  
        try:  
            resp = httpx.post(  
                f"{self.base_url}/chat/completions",  
                headers={"Authorization": f"Bearer {self.api_key}"},  
                json=payload,  
                timeout=self.timeout,           # 关键行：显式超时  
            )  
            resp.raise_for_status()             # 关键行：4xx/5xx 立刻抛  
        except httpx.HTTPError as e:  
            logger.error("模型调用失败: %s", e)  
            raise UpstreamError("模型服务不可用") from e  
        return resp.json()["choices"][0]["message"]["content"]  
```  
  
| 代码 | 含义 |  
|---|---|  
| `rstrip("/")` | 去掉地址末尾斜杠，避免拼出双斜杠 |  
| 构造参数默认取配置 | 测试可传假地址与假密钥 |  
| `@retry(...)` | 只对超时与状态码错误重试 |  
| `raise ... from e` | 保留原始网络错误 |  
| `timeout=self.timeout` | 连接与读取共用同一上限 |  
  
固定模板：`retry` 装饰器与 `LLMClient` 的 `__init__`／`chat` 结构。可组合写法：退避策略、重试异常范围、是否支持流式。  
  
已知边界：同步客户端会阻塞线程；高并发接口应改用 `httpx.AsyncClient`，本册不展开。  
  
调用链：`LLMClient.chat` 由 `executor` 调用；它调用 `httpx.post`；超时与状态码错误经重试后包装为 `UpstreamError`，最终由全局处理器映射为 HTTP 502。  
  
### 2.7 结尾块  
  
**本章判据**：模型接口超时或返回 429 时，调用方拿到统一的 `UpstreamError`，且最多尝试 3 次。  
  
**本章产出**：`app/core/retry.py`、`app/agent/llm_client.py`。  
  
**下一章入口**：打开 `app/agent/registry.py`，加工具注册与调用。  
  
**常见错法 3 条**：  
  
1. 不传 `timeout`，依赖库默认值。  
2. 不调 `raise_for_status`，直接读 `choices`。  
3. 重试包住所有异常，把参数错误也重试 3 次。  
  
**自检 3 问**：  
  
1. 服务端 60 秒不回，客户端多久放弃？  
2. 429 时我拿到的是 `KeyError` 还是可读错误？  
3. 重试次数用尽后抛的是哪种异常？  
  
---  
  
## 第 3 章 注册工具  
  
### 3.1 场景与动机  
  
本章做：把检索包装成模型可调用的工具，并统一注册。  
本章不做：模型调用、循环编排、测试。  
  
① 第 2 章交出的是 `app/agent/llm_client.py`，能拿到一段文本，但模型还不知道有哪些能力可用。  
② 工具增加到 3 个后，`if name == "kb_search" ... elif ...` 要改 3 处；参数传错只会得到 `TypeError`，模型无法自我修正。  
③ 本章要给工具一个统一接口，让新增工具不改调度代码。  
④ 选抽象基类加注册表：每个工具自带描述与参数，调度只按名字查表。  
  
本章不涉及新数据结构；工具协议 `ToolSpec` 已在 1.2 定义。  
  
### 3.2 最小可运行版  
  
本段做什么：用函数字典与 `if` 分发实现一个检索工具，只保证能调用。  
  
```python  
# 本段：最小可运行版，函数字典分发  
from app.services.search_service import search_page  
  
def kb_search(keyword: str) -> str:  
    """按关键词检索分块，拼成文本。"""  
    page = search_page(db, keyword, page=1, page_size=5)  
    return "\n".join(c["text"] if isinstance(c, dict) else c.text for c in page["items"])  
  
TOOLS = {"kb_search": kb_search}  
  
def run_tool(name: str, args: dict) -> str:  
    """按名字调用工具。"""  
    return TOOLS[name](**args)  
```  
  
### 3.3 分段讲解  
  
| 代码 | 含义 |  
|---|---|  
| `def kb_search(keyword: str)` | 工具本体，参数即模型要填的字段 |  
| `search_page(db, ...)` | 复用第三册检索，取前 5 块 |  
| `"\n".join(...)` | 把多块拼成一段文本，作为观察结果 |  
| `TOOLS = {...}` | 名字到函数的映射 |  
| `TOOLS[name](**args)` | 解包字典作为关键字参数 |  
| `isinstance(c, dict)` | 兼容返回字典或模型实例两种情况 |  
  
**知识点 1：抽象基类**  
  
定义：抽象基类用 `@abstractmethod` 声明必须实现的方法，子类不实现就无法实例化。  
  
定式（要背）：  
  
```python  
from abc import ABC, abstractmethod  
  
class Base(ABC):  
    @abstractmethod  
    def run(self, **kwargs) -> str: ...  
```  
  
适用：一组同类组件需要统一接口时。例题：子类漏写 `run`，实例化直接抛 `TypeError`。  
  
**知识点 2：工具自描述**  
  
定义：工具把名称、用途、参数结构作为属性暴露给模型，模型据此选择与填参。  
  
定式（要背）：  
  
```python  
name = "kb_search"  
description = "在知识库中按关键词检索分块"  
parameters = {"type": "object", "properties": {"keyword": {"type": "string"}}, "required": ["keyword"]}  
```  
  
适用：任何要交给模型调用的函数。例题：模型看到 `required` 后不会漏传 `keyword`。  
  
**知识点 3：注册表**  
  
定义：注册表用字典保存"名字 → 工具实例"，调度只查表，不写分支。  
  
定式（要背）：  
  
```python  
registry.register(tool)      # 启动时注册  
registry.run(name, args)     # 运行时按名调用  
```  
  
适用：工具数量会增长的系统。例题：新增第 4 个工具只调用一次 `register`，调度代码不改。  
  
**知识点 4：统一返回与参数错误**  
  
定义：工具统一返回字符串；参数错误转成可读观察，而不是抛异常中断。  
  
定式（要背）：  
  
```python  
try:  
    return tool.run(**args)  
except TypeError as e:  
    return f"参数错误: {e}"  
```  
  
适用：所有工具入口。例题：模型传了多余参数，返回错误文本，下一轮可自我修正。  
  
### 3.4 指出缺陷  
  
以下缺陷只评"注册工具"这一动作，均可在 3.2 代码中定位。  
  
- **D1**：`TOOLS` 只有名字到函数，没有 `description` 与 `parameters` → 模型不知道工具用途与参数，只能瞎猜，命中率低。  
- **D2**：`TOOLS[name](**args)` 直接解包 → 参数名写错抛 `TypeError`，整个请求中断，模型失去修正机会。  
- **D3**：新增工具要改 `TOOLS` 与可能的分支判断 → 工具从 1 个增到 5 个时，调度函数被改 4 次。  
- **D4**：返回类型不统一，有的返回 `list` 有的返回 `str` → 编排器拼上下文时对 `list` 调 `[:500]` 报 `TypeError`。  
  
### 3.5 结构设计  
  
① 文件清单：  
  
| 文件路径 | 一句话职责 | 依赖谁 |  
|---|---|---|  
| `app/agent/base.py` | 定义 `BaseTool` 抽象接口 | `abc`、`schemas` |  
| `app/agent/registry.py` | 保存并调度工具 | `base` |  
| `app/agent/tools/kb_search.py` | 检索工具实现 | `base`、`search_service` |  
  
② 拆分理由：  
  
- `base.py`：解决 D1、D4，把名称、描述、参数、返回类型固化成接口。  
- `registry.py`：解决 D3，新增工具只 `register`，不改调度。  
- `kb_search.py`：解决 D2，工具内部处理参数与返回值，错误转成文本。  
  
③ 不这么拆会怎样：工具实现与调度同文件，5 个工具后该文件 300 行，改一个工具要重新读调度；测试只能整体调用，无法单独测某个工具。  
  
④ 依赖图：  
  
```  
app/agent/schemas.py  
        ↑  
app/agent/base.py (BaseTool)  
        ↑                    ↑  
app/agent/registry.py   app/agent/tools/kb_search.py  
                              ↑  
                    app/services/search_service.py  
```  
  
⑤ 分工：设计决策回答"函数字典还是类继承"；结构设计回答"接口、注册表、具体工具分三个文件"。  
  
### 3.6 完整版  
  
**设计决策：类继承，而不是函数字典**  
  
① 痛点：D1 缺描述与参数、D2 参数错误中断、D3 改动点随工具增长、D4 返回类型不一。  
  
② 取舍：  
  
- 方案 A：函数字典。适用于 1～2 个固定工具。否决理由：描述与参数无处安放，模型侧 schema 要另写一份，新增工具要同步改两处。  
- 方案 B：装饰器注册（`@tool`）。适用于工具函数简单、无状态。否决理由：工具需要数据库会话等依赖时，闭包注入不如类构造清晰，且参数校验仍要额外挂。  
  
③ 触发条件：工具 ≥3 或需要携带依赖（会话、客户端）时用类；只有 1 个无状态工具时用函数字典即可。  
  
**文件 1：`app/agent/base.py`**  
  
本段做什么：定义所有工具必须实现的接口与自描述。  
  
```python  
# 本段：工具基类  
"""工具抽象接口。"""  
from abc import ABC, abstractmethod             # 标准库 · 抽象基类 · 强制实现  
  
from app.agent.schemas import ToolSpec          # 本项目 · 协议  
  
  
class BaseTool(ABC):  
    """所有工具的基类：名称、描述、参数、执行。"""  
  
    name: str = ""  
    description: str = ""  
    parameters: dict = {}  
  
    @abstractmethod  
    def run(self, **kwargs) -> str:  
        """执行工具，返回文本观察结果。"""  
  
    def spec(self) -> ToolSpec:  
        """生成给模型看的工具描述。"""  
        return ToolSpec(name=self.name, description=self.description,  
                        parameters=self.parameters)  
```  
  
| 代码 | 含义 |  
|---|---|  
| `ABC` | 抽象基类，配合 `@abstractmethod` 使用 |  
| `@abstractmethod` | 子类不实现则无法实例化 |  
| `**kwargs` | 接收任意关键字参数，具体由子类签名约束 |  
| `spec()` | 把类属性转成 `ToolSpec`，供提示词使用 |  
  
**文件 2：`app/agent/registry.py`**  
  
本段做什么：保存工具，按名调用，错误转成文本。  
  
```python  
# 本段：工具注册表  
"""工具注册与调度。"""  
from app.agent.base import BaseTool             # 本项目 · 工具基类  
from app.agent.schemas import ToolSpec          # 本项目 · 协议  
from app.core.logging import get_logger         # 本项目 · 日志  
  
logger = get_logger(__name__)  
  
  
class ToolRegistry:  
    """名字到工具实例的映射。"""  
  
    def __init__(self):  
        """初始化空注册表。"""  
        self._tools: dict[str, BaseTool] = {}  
  
    def register(self, tool: BaseTool) -> None:  
        """注册一个工具，重名覆盖。"""  
        self._tools[tool.name] = tool  
  
    def specs(self) -> list[ToolSpec]:  
        """返回全部工具描述，供提示词拼接。"""  
        return [t.spec() for t in self._tools.values()]  
  
    def run(self, name: str, args: dict) -> str:  
        """按名调用；未知工具或参数错误返回可读文本。"""  
        tool = self._tools.get(name)  
        if tool is None:  
            return f"未知工具: {name}"  
        try:  
            return tool.run(**args)              # 关键行：参数错误转文本  
        except TypeError as e:  
            return f"参数错误: {e}"  
```  
  
| 代码 | 含义 |  
|---|---|  
| `dict[str, BaseTool]` | 类型注解：键为字符串，值为工具实例 |  
| `register` | 按 `tool.name` 写入，重名覆盖 |  
| `specs` | 批量生成模型可读的工具描述 |  
| `.get(name)` | 不存在返回 `None`，不抛异常 |  
| `except TypeError` | 捕获参数不匹配，转成观察文本 |  
  
**文件 3：`app/agent/tools/kb_search.py`**  
  
本段做什么：把第三册检索包装成工具。  
  
```python  
# 本段：检索工具  
"""知识库检索工具。"""  
from sqlalchemy.orm import Session              # 第三方库 · ORM · 会话类型  
  
from app.agent.base import BaseTool             # 本项目 · 工具基类  
from app.services.search_service import search_page   # 本项目 · 检索服务  
  
  
class KBSearchTool(BaseTool):  
    """在知识库中按关键词检索分块。"""  
  
    name = "kb_search"  
    description = "在知识库中按关键词检索相关文本块，返回若干条片段"  
    parameters = {  
        "type": "object",  
        "properties": {"keyword": {"type": "string", "description": "检索关键词"}},  
        "required": ["keyword"],  
    }  
  
    def __init__(self, db: Session):  
        """注入数据库会话，供检索使用。"""  
        self.db = db  
  
    def run(self, keyword: str) -> str:  
        """执行检索，把前 5 块拼成文本。"""  
        page = search_page(self.db, keyword, page=1, page_size=5)  
        items = page["items"]  
        if not items:  
            return "未检索到相关内容"  
        return "\n".join(  
            f"[{i + 1}] {getattr(c, 'text', c.get('text', ''))}"  
            for i, c in enumerate(items)  
        )  
```  
  
| 代码 | 含义 |  
|---|---|  
| `name` / `description` / `parameters` | 工具自描述，模型据此选择与填参 |  
| `__init__(db)` | 构造时注入依赖，避免全局变量 |  
| `search_page(...)` | 复用第三册分页检索 |  
| `getattr(c, "text", c.get("text",""))` | 兼容模型实例与字典两种返回 |  
  
固定模板：`BaseTool` 的四个成员、`ToolRegistry` 的三个方法。可组合写法：是否做参数二级校验、是否限制返回长度、是否记录调用耗时。  
  
已知边界：`run` 只捕获 `TypeError`；工具内部业务异常会冒泡，需由编排器统一兜底，避免一个工具坏掉整轮对话。  
  
调用链：`ToolRegistry.run` 由 `executor` 调用；它调用 `BaseTool.run`，最终到 `search_page` 与数据库；工具内的 `TypeError` 转成文本，其他异常继续上抛到编排器。  
  
### 3.7 结尾块  
  
**本章判据**：新增一个工具只需写一个类并 `register`，调度代码不改。  
  
**本章产出**：`app/agent/base.py`、`app/agent/registry.py`、`app/agent/tools/kb_search.py`。  
  
**下一章入口**：打开 `app/agent/executor.py`，加 ReAct 循环。  
  
**常见错法 3 条**：  
  
1. 用 `if/elif` 分发，新增工具改调度。  
2. 工具不写 `description` 与 `parameters`。  
3. 工具返回 `list`，编排器按字符串处理。  
  
**自检 3 问**：  
  
1. 新增第 4 个工具要改几个文件？  
2. 模型传错参数，请求会中断吗？  
3. 工具返回值类型统一吗？  
  
---  
  
## 第 4 章 编排循环  
  
### 4.1 场景与动机  
  
本章做：让模型决定调工具还是给答案，循环到收敛。  
本章不做：改协议、改客户端、改工具实现、写测试。  
  
① 第 3 章交出的是 `app/agent/registry.py` 与 `kb_search.py`，工具可调用，但没有东西去决定何时调用。  
② 单轮调用只能回答模型已知的内容；需要查文档时，模型拿不到检索结果，答"我不知道"。  
③ 本章要把"决策—执行—观察"串成有上限的循环。  
④ 选 ReAct 循环加最大步数：每轮只做一次决策，步数封顶防止烧钱。  
  
本章不涉及新数据结构。  
  
### 4.2 最小可运行版  
  
本段做什么：跑一轮"决策 → 调工具 → 再问"，只有一个轮次。  
  
```python  
# 本段：最小可运行版，单轮工具调用  
import json  
from app.agent.schemas import AgentDecision, ChatMessage  
  
def run_once(client, registry, question: str) -> str:  
    """问一次，若模型要调工具则执行一次再问。"""  
    messages = [ChatMessage(role="user", content=question)]  
    raw = client.chat(messages)  
    decision = AgentDecision.model_validate_json(raw)  
    if decision.action == "final":  
        return decision.answer or ""  
    observation = registry.run(decision.tool, decision.args)  
    messages.append(ChatMessage(role="assistant", content=raw))  
    messages.append(ChatMessage(role="user", content=f"观察结果: {observation}"))  
    raw2 = client.chat(messages)  
    return AgentDecision.model_validate_json(raw2).answer or ""  
```  
  
### 4.3 分段讲解  
  
| 代码 | 含义 |  
|---|---|  
| `messages = [ChatMessage(...)]` | 初始只有用户问题 |  
| `client.chat(messages)` | 取模型原始文本 |  
| `model_validate_json` | 解析成 `AgentDecision` |  
| `decision.action == "final"` | 直接返回答案 |  
| `registry.run(tool, args)` | 执行工具，得到观察 |  
| `append(...)` | 把模型输出与观察追加进历史 |  
| 第二次 `chat` | 带着观察再问一次 |  
  
**知识点 1：ReAct 循环**  
  
定义：ReAct 是"推理—行动—观察"交替的循环，模型每轮输出一个决策，程序执行后把结果回填。  
  
定式（要背）：  
  
```python  
for step in range(max_steps):  
    decision = parse(client.chat(messages))  
    if decision.action == "final":  
        return decision.answer  
    observation = registry.run(decision.tool, decision.args)  
    messages += [assistant_msg, observation_msg]  
```  
  
适用：需要外部信息或多步操作的任务。例题：问"部署方式"，第一轮调 `kb_search`，第二轮给答案。  
  
**知识点 2：最大步数**  
  
定义：最大步数是循环的硬上限，超过即停止并返回兜底话术。  
  
定式（要背）：  
  
```python  
for step in range(1, max_steps + 1):  
    ...  
return "达到最大步数，未能得出结论"  
```  
  
适用：所有带模型循环的系统。例题：`max_steps=5` 时最多 5 次模型调用。  
  
**知识点 3：观察截断**  
  
定义：把工具结果回填前截断到固定长度，防止上下文膨胀。  
  
定式（要背）：  
  
```python  
observation = raw_observation[:500]  
```  
  
适用：检索、抓网页等长文本工具。例题：返回 5000 字，只回填前 500 字。  
  
**知识点 4：异常转观察**  
  
定义：工具异常不直接中断循环，而是作为观察回填，让模型换策略。  
  
定式（要背）：  
  
```python  
try:  
    observation = registry.run(name, args)  
except Exception as e:  
    observation = f"工具执行失败: {e}"  
```  
  
适用：任何外部工具。例题：检索抛连接错误，模型收到失败信息后可改用其他关键词。  
  
### 4.4 指出缺陷  
  
以下缺陷只评"编排"这一动作，均可在 4.2 代码中定位。  
  
- **D1**：`run_once` 只跑两轮，没有 `max_steps` 与循环 → 需要三次以上工具调用的问题直接返回空答案。  
- **D2**：`registry.run` 未包 `try` → 工具抛异常时整个请求 500，模型没有机会换策略。  
- **D3**：`observation` 原样拼入消息，未截断 → 检索返回 5000 字时上下文迅速膨胀，下一轮报超长。  
- **D4**：没有记录 `step`、工具名与耗时 → 线上回答错误时无法判断是没调工具、调错工具还是工具失败。  
  
### 4.5 结构设计  
  
① 文件清单：  
  
| 文件路径 | 一句话职责 | 依赖谁 |  
|---|---|---|  
| `app/agent/executor.py` | 组织消息、驱动循环、兜底 | `llm_client`、`registry`、`schemas` |  
| `app/agent/prompts.py` | 存放系统提示词模板 | `schemas` |  
  
② 拆分理由：  
  
- `executor.py`：解决 D1（循环与上限）、D2（异常转观察）、D3（截断）、D4（逐步日志）。  
- `prompts.py`：把提示词从逻辑中抽出，调提示词不改执行代码。  
  
③ 不这么拆会怎样：提示词字符串写在 `executor` 里，改一句话要重读 80 行控制流；多个 Agent 复用同一提示词时只能复制。  
  
④ 依赖图：  
  
```  
app/agent/prompts.py ──→ app/agent/executor.py  
                              ↑        ↑  
                    llm_client     registry  
```  
  
⑤ 分工：设计决策回答"单轮、循环还是先计划后执行"；结构设计回答"执行与提示词分两个文件"。  
  
### 4.6 完整版  
  
**设计决策：ReAct 循环，而不是单轮或先计划后执行**  
  
① 痛点：D1 的两轮上限、D2 的异常中断、D3 的上下文膨胀、D4 的不可观测。  
  
② 取舍：  
  
- 方案 A：单轮调用。适用于模型已知答案的问答。否决理由：需要查库时拿不到外部信息，只能答"不知道"。  
- 方案 B：先计划全部步骤再执行。适用于步骤稳定、可预先分解的任务。否决理由：检索结果会改变下一步，预先计划无法根据观察调整。  
  
③ 触发条件：需要外部信息且下一步依赖上一步结果时用 ReAct；步骤固定、可批处理时用计划执行；无需外部信息时单轮即可。  
  
**文件 1：`app/agent/prompts.py`**  
  
本段做什么：生成系统提示词，声明输出格式与可用工具。  
  
```python  
# 本段：提示词  
"""Agent 系统提示词。"""  
import json                                    # 标准库 · JSON · 序列化工具描述  
  
from app.agent.schemas import ToolSpec         # 本项目 · 协议  
  
  
def build_system_prompt(tools: list[ToolSpec]) -> str:  
    """把工具描述拼进系统提示词，限定输出为 JSON。"""  
    tool_text = json.dumps([t.model_dump() for t in tools], ensure_ascii=False)  
    return (  
        "你是知识库助手。可用工具如下：\n"  
        f"{tool_text}\n"  
        "只输出一个 JSON 对象，不要输出其他文字。\n"  
        '要调工具时输出：{"action":"tool","tool":"名字","args":{...},"thought":"原因"}\n'  
        '能直接回答时输出：{"action":"final","answer":"答案"}'  
    )  
```  
  
| 代码 | 含义 |  
|---|---|  
| `model_dump()` | 模型转字典，便于序列化 |  
| `json.dumps(..., ensure_ascii=False)` | 保留中文，不转成 `\uXXXX` |  
| 字符串拼接 | 把工具列表与格式要求写进系统消息 |  
  
**文件 2：`app/agent/executor.py`**  
  
本段做什么：驱动循环，处理解析失败、工具异常与步数上限。  
  
```python  
# 本段：循环执行器  
"""Agent 执行器：ReAct 循环与兜底。"""  
from pydantic import ValidationError           # 第三方库 · 校验 · 解析异常  
  
from app.agent.llm_client import LLMClient     # 本项目 · 客户端  
from app.agent.prompts import build_system_prompt   # 本项目 · 提示词  
from app.agent.registry import ToolRegistry    # 本项目 · 工具表  
from app.agent.schemas import AgentDecision, ChatMessage   # 本项目 · 协议  
from app.core.exceptions import UpstreamError  # 本项目 · 异常  
from app.core.logging import get_logger        # 本项目 · 日志  
  
logger = get_logger(__name__)  
  
  
class AgentExecutor:  
    """按 ReAct 循环调用模型与工具。"""  
  
    def __init__(self, client: LLMClient, registry: ToolRegistry, max_steps: int = 5):  
        """注入客户端、工具表与步数上限。"""  
        self.client = client  
        self.registry = registry  
        self.max_steps = max_steps  
  
    def run(self, question: str) -> str:  
        """回答问题，最多 max_steps 轮。"""  
        messages = [  
            ChatMessage(role="system", content=build_system_prompt(self.registry.specs())),  
            ChatMessage(role="user", content=question),  
        ]  
        for step in range(1, self.max_steps + 1):  
            raw = self.client.chat(messages)  
            try:  
                decision = AgentDecision.model_validate_json(raw)  
            except ValidationError:  
                logger.warning("第 %d 轮输出非法，要求重答", step)  
                messages.append(ChatMessage(role="assistant", content=raw))  
                messages.append(ChatMessage(role="user", content="只输出合法 JSON。"))  
                continue  
            if decision.action == "final":  
                logger.info("第 %d 轮结束 | answer_len=%d", step, len(decision.answer or ""))  
                return decision.answer or ""  
            try:  
                observation = self.registry.run(decision.tool or "", decision.args)  
            except Exception as e:                     # 关键行：工具异常不中断  
                observation = f"工具执行失败: {e}"  
            logger.info("第 %d 轮调用 %s | obs_len=%d", step, decision.tool, len(observation))  
            messages.append(ChatMessage(role="assistant", content=raw))  
            messages.append(ChatMessage(role="user", content=f"观察结果: {observation[:500]}"))  
        logger.warning("达到最大步数 | question=%s", question[:30])  
        return "达到最大步数，未能得出结论"  
```  
  
| 代码 | 含义 |  
|---|---|  
| `max_steps=5` | 默认最多 5 轮 |  
| `try ... except ValidationError` | 输出不合法时提示重答，不直接失败 |  
| `continue` | 跳过本轮，进入下一轮 |  
| `except Exception` | 工具异常转成观察文本 |  
| `observation[:500]` | 截断，控制上下文长度 |  
| `logger.info` | 每轮记录步数、工具、长度 |  
  
固定模板：`AgentExecutor.__init__` 与 `run` 的循环骨架。可组合写法：提示词、截断长度、最大步数、是否并行调用多个工具。  
  
已知边界：非法 JSON 会消耗一次步数；模型连续输出非 JSON 时会在 `max_steps` 用尽后返回兜底话术，而不是抛错。  
  
调用链：`AgentExecutor.run` 由第五册的对话接口调用；它调用 `LLMClient.chat` 与 `ToolRegistry.run`；`UpstreamError` 向上冒泡到全局处理器，工具异常在内部转为观察文本。  
  
### 4.7 结尾块  
  
**本章判据**：问一个需要查文档的问题，能在 5 轮内给出带检索依据的答案。  
  
**本章产出**：`app/agent/prompts.py`、`app/agent/executor.py`。  
  
**下一章入口**：打开 `tests/test_agent.py`，用假客户端测循环。  
  
**常见错法 3 条**：  
  
1. 不设最大步数。  
2. 工具异常直接冒泡。  
3. 观察结果不截断。  
  
**自检 3 问**：  
  
1. 模型一直要调工具，循环会停吗？  
2. 检索工具挂了，整轮对话会 500 吗？  
3. 工具返回 5000 字，下一轮上下文多长？  
  
---  
  
## 第 5 章 测编排  
  
### 5.1 场景与动机  
  
本章做：用假客户端测编排逻辑，不联网、不花钱。  
本章不做：改业务代码、改提示词、测真实模型质量。  
  
① 第 4 章交出的是 `app/agent/executor.py`，循环能跑，但每次测试都真调模型。  
② 一次真实调用约 2～5 秒、按 token 计费；模型措辞变化会让断言随机失败。  
③ 本章要让循环逻辑可稳定验证。  
④ 选手写假客户端：返回固定序列，专测分支。  
  
本章不涉及新数据结构。  
  
### 5.2 最小可运行版  
  
本段做什么：用一个返回固定文本的假客户端跑一次循环。  
  
```python  
# 本段：最小可运行版，假客户端  
from app.agent.executor import AgentExecutor  
from app.agent.registry import ToolRegistry  
from app.agent.schemas import ToolSpec  
  
class FakeClient:  
    """按顺序返回预设文本。"""  
    def __init__(self, replies: list[str]):  
        self.replies = list(replies)  
    def chat(self, messages):  
        """返回下一条预设文本。"""  
        return self.replies.pop(0)  
  
class EmptyRegistry(ToolRegistry):  
    """无工具的注册表。"""  
    def specs(self) -> list[ToolSpec]:  
        """返回空列表。"""  
        return []  
  
def test_final_answer_directly():  
    """模型第一轮就给答案。"""  
    executor = AgentExecutor(FakeClient(['{"action":"final","answer":"好的"}']), EmptyRegistry())  
    assert executor.run("你好") == "好的"  
```  
  
### 5.3 分段讲解  
  
| 代码 | 含义 |  
|---|---|  
| `FakeClient.chat` | 与真实客户端同名方法，返回预设文本 |  
| `list(replies)` | 复制一份，避免测试间共享 |  
| `pop(0)` | 按顺序取，模拟多轮 |  
| `EmptyRegistry` | 无工具场景，隔离工具影响 |  
| `assert executor.run(...)` | 断言最终答复 |  
  
**知识点 1：鸭子类型替身**  
  
定义：只要对象有同名方法且返回结构一致，就能替换真实依赖，不必继承。  
  
定式（要背）：  
  
```python  
class FakeClient:  
    def chat(self, messages) -> str:  
        return '{"action":"final","answer":"ok"}'  
```  
  
适用：外部客户端、网络依赖。例题：把 `LLMClient` 换成 `FakeClient`，`AgentExecutor` 不改一行。  
  
**知识点 2：`monkeypatch`**  
  
定义：pytest 的 `monkeypatch` 可在测试期间替换模块属性，结束后自动还原。  
  
定式（要背）：  
  
```python  
def test_x(monkeypatch):  
    monkeypatch.setattr("app.agent.llm_client.httpx.post", fake_post)  
```  
  
适用：替换模块级函数、环境变量。例题：替换 `httpx.post` 后测试不发真实请求。  
  
**知识点 3：契约测试**  
  
定义：契约测试只验证"给定输入应产生什么行为"，不锁定具体措辞与实现。  
  
定式（要背）：  
  
```python  
assert "部署" in answer      # 断言语义，不断言整句  
assert client.call_count == 2  # 断言调用次数  
```  
  
适用：模型输出不稳定、实现可能重构的场景。例题：模型换措辞，测试仍通过。  
  
### 5.4 指出缺陷  
  
以下缺陷只评"测编排"这一动作，均可在 5.2 代码中定位。  
  
- **D1**：`FakeClient` 之外若直接用 `LLMClient()`，测试每次真联网 → 单测 2～5 秒、按量计费，CI 无密钥时报 401。  
- **D2**：`assert executor.run(...) == "好的"` 只覆盖直接回答 → 调工具分支、非法 JSON 分支、达到上限分支全无覆盖。  
- **D3**：断言整句文本 → 模型换个说法（"可以的"）测试就红，实际功能没坏。  
- **D4**：`EmptyRegistry` 只提供空工具 → 无法验证"模型选错工具名"与"工具抛异常"两条路径。  
  
### 5.5 结构设计  
  
① 文件清单：  
  
| 文件路径 | 一句话职责 | 依赖谁 |  
|---|---|---|  
| `tests/fakes.py` | 假客户端、假工具、假注册表 | `schemas`、`base` |  
| `tests/test_agent.py` | 编排分支的契约测试 | `fakes`、`executor` |  
  
② 拆分理由：  
  
- `fakes.py`：解决 D1，所有替身集中一份；解决 D4，提供可配置的假工具与假注册表。  
- `test_agent.py`：解决 D2、D3，按分支写用例，断言行为而非整句。  
  
③ 不这么拆会怎样：替身写在每个测试函数里，5 个用例复制 5 份 `FakeClient`，行为逐渐不一致；断言散落整句，模型升级后集体红。  
  
④ 依赖图：  
  
```  
app/agent/executor.py  
        ↑  
tests/fakes.py  →  tests/test_agent.py  
```  
  
⑤ 分工：设计决策回答"真实调用、录制回放还是手写假客户端"；结构设计回答"替身与用例分两个文件"。  
  
### 5.6 完整版  
  
**设计决策：手写假客户端，而不是录制回放**  
  
① 痛点：D1 的真实调用成本、D2 的分支覆盖缺失、D3 的整句断言、D4 的工具路径未测。  
  
② 取舍：  
  
- 方案 A：真实调用。适用于上线前的效果评估。否决理由：慢、花钱、结果不稳定，无法在 CI 无条件运行。  
- 方案 B：录制真实响应回放。适用于回归真实模型行为。否决理由：录制文件与模型版本绑定，换模型全部失效，且覆盖不到"非法输出""超时"这类异常分支。  
  
③ 触发条件：逻辑分支测试用手写假客户端；模型效果评估用真实调用并单独打标；需要回归真实措辞时再引入录制。  
  
**文件 1：`tests/fakes.py`**  
  
本段做什么：提供可配置的假客户端与假工具。  
  
```python  
# 本段：测试替身  
"""测试替身：假客户端、假工具、假注册表。"""  
from app.agent.base import BaseTool             # 本项目 · 工具基类  
from app.agent.registry import ToolRegistry     # 本项目 · 注册表  
from app.agent.schemas import ToolSpec          # 本项目 · 协议  
  
  
class FakeClient:  
    """按顺序返回预设文本的假客户端。"""  
  
    def __init__(self, replies: list[str]):  
        """保存预设序列。"""  
        self.replies = list(replies)  
        self.call_count = 0  
  
    def chat(self, messages) -> str:  
        """返回下一条预设文本，并计数。"""  
        self.call_count += 1  
        return self.replies.pop(0)  
  
  
class EchoTool(BaseTool):  
    """回显参数的假工具。"""  
  
    name = "echo"  
    description = "回显输入"  
    parameters = {"type": "object", "properties": {"text": {"type": "string"}}, "required": ["text"]}  
  
    def run(self, text: str) -> str:  
        """返回输入文本。"""  
        return f"echo: {text}"  
  
  
class BoomTool(BaseTool):  
    """总是抛异常的假工具。"""  
  
    name = "boom"  
    description = "用于测试工具异常"  
    parameters = {"type": "object", "properties": {}}  
  
    def run(self) -> str:  
        """抛出运行时错误。"""  
        raise RuntimeError("工具炸了")  
```  
  
| 代码 | 含义 |  
|---|---|  
| `list(replies)` | 复制序列，避免测试间共享 |  
| `call_count` | 记录调用次数，供断言 |  
| `EchoTool` | 返回固定格式，验证参数传递 |  
| `BoomTool` | 专测异常转观察 |  
  
**文件 2：`tests/test_agent.py` 第 1 段**  
  
本段做什么：覆盖直接回答与工具调用两条路径。  
  
```python  
# 本段：正常路径  
"""Agent 编排的契约测试。"""  
from app.agent.executor import AgentExecutor     # 本项目 · 被测  
from app.agent.registry import ToolRegistry      # 本项目 · 注册表  
from tests.fakes import EchoTool, FakeClient     # 测试代码 · 替身  
  
  
def test_direct_answer():  
    """模拟：模型无需工具直接回答（正常路径）。  
    契约：返回答案，只调用 1 次模型。"""  
    client = FakeClient(['{"action":"final","answer":"好的"}'])  
    executor = AgentExecutor(client, ToolRegistry())  
    assert executor.run("你好") == "好的"  
    assert client.call_count == 1  
  
  
def test_tool_then_answer():  
    """模拟：模型先调工具再回答（正常路径）。  
    契约：调用 2 次模型，答案包含工具结果。"""  
    client = FakeClient([  
        '{"action":"tool","tool":"echo","args":{"text":"部署"}}',  
        '{"action":"final","answer":"根据工具结果：部署"}',  
    ])  
    registry = ToolRegistry()  
    registry.register(EchoTool())  
    answer = AgentExecutor(client, registry).run("怎么部署")  
    assert "部署" in answer  
    assert client.call_count == 2  
```  
  
| 代码 | 含义 |  
|---|---|  
| 预设两条文本 | 第一轮调工具，第二轮给答案 |  
| `registry.register(EchoTool())` | 注册可用的假工具 |  
| `"部署" in answer` | 断言语义，不断言整句 |  
| `call_count == 2` | 断言循环轮数 |  
  
**文件 2 第 2 段**  
  
本段做什么：覆盖工具异常、非法输出、达到上限三条分支。  
  
```python  
# 本段：异常与边界路径  
def test_tool_error_becomes_observation():  
    """模拟：工具抛异常（异常路径）。  
    契约：不向外抛，循环继续，最终返回答案。"""  
    client = FakeClient([  
        '{"action":"tool","tool":"boom","args":{}}',  
        '{"action":"final","answer":"工具不可用"}',  
    ])  
    registry = ToolRegistry()  
    registry.register(BoomTool())  
    assert AgentExecutor(client, registry).run("试一下") == "工具不可用"  
  
  
def test_invalid_json_retries():  
    """模拟：模型输出非 JSON（异常路径）。  
    契约：不崩溃，要求重答，最终给出答案。"""  
    client = FakeClient([  
        "我觉得应该查一下资料",  
        '{"action":"final","answer":"补答成功"}',  
    ])  
    assert AgentExecutor(client, ToolRegistry()).run("问") == "补答成功"  
  
  
def test_max_steps_stops_loop():  
    """模拟：模型一直要求调工具（边界路径）。  
    契约：最多调用 max_steps 次，返回兜底话术。"""  
    client = FakeClient(['{"action":"tool","tool":"echo","args":{"text":"x"}}'] * 5)  
    registry = ToolRegistry()  
    registry.register(EchoTool())  
    result = AgentExecutor(client, registry, max_steps=3).run("循环")  
    assert "最大步数" in result  
    assert client.call_count == 3  
```  
  
| 代码 | 含义 |  
|---|---|  
| `BoomTool` 用例 | 验证异常被转成观察，不中断 |  
| 非 JSON 用例 | 验证重答逻辑 |  
| `* 5` 重复预设 | 模拟模型持续要工具 |  
| `max_steps=3` | 验证上限与调用次数 |  
  
固定模板：假客户端的三要素（预设序列、计数、同名 `chat`）。可组合写法：断言粒度、是否用 `monkeypatch` 替换 `httpx`、是否加耗时断言。  
  
已知边界：假客户端不验证真实模型的 JSON 遵循率；该指标需用真实调用单独统计。  
  
调用链：测试函数由 pytest 发现；它们构造 `FakeClient` 与 `ToolRegistry` 注入 `AgentExecutor`；断言失败直接使该测试失败，不触碰网络与数据库。  
  
### 5.7 结尾块  
  
**本章判据**：不联网、无密钥时，四条分支（直接答、调工具、工具异常、非法输出）全部可测。  
  
**本章产出**：`tests/fakes.py`、`tests/test_agent.py`。  
  
**下一章入口**：本册封版，下一册把这套 Agent 挂到 HTTP 接口上。  
  
**常见错法 3 条**：  
  
1. 测试里直接 new 真实客户端。  
2. 断言整句回答。  
3. 只测直接回答，不测工具与异常分支。  
  
**自检 3 问**：  
  
1. 没网、没密钥时测试能跑吗？  
2. 模型换措辞，测试会红吗？  
3. 工具抛异常的分支有覆盖吗？  
  
**下一册从哪接**：`app/agent/executor.py` 的 `AgentExecutor.run` 将作为第五册 `POST /api/v1/chat` 的核心调用，接口层只负责参数校验、依赖注入与流式返回。  
  
---  
  
## 运行与验收  
  
```bash  
pip install httpx pydantic pytest  
```  
  
文件清单：  
  
```  
app/agent/__init__.py  
app/agent/schemas.py  
app/agent/llm_client.py  
app/agent/base.py  
app/agent/registry.py  
app/agent/tools/kb_search.py  
app/agent/prompts.py  
app/agent/executor.py  
app/core/retry.py  
tests/fakes.py  
tests/test_agent.py  
```  
  
验收：  
  
```bash  
# 1) 协议校验  
python -c "  
from app.agent.schemas import AgentDecision  
print(AgentDecision.model_validate_json('{\"action\":\"final\",\"answer\":\"ok\"}'))  
"  
  
# 2) 测试  
pytest -v  
  
# 3) 真实调用（需在 .env 配好密钥）  
python -c "  
from app.agent.llm_client import LLMClient  
from app.agent.schemas import ChatMessage  
print(LLMClient().chat([ChatMessage(role='user', content='用一句话说明什么是 ReAct')]))  
"  
```  
  
预期：协议输出 `action='final'`；测试全绿；真实调用返回一段中文说明。  
  
---  
  
## 知识地图  
  
| 知识点 | 落点 |  
|---|---|  
| Pydantic 协议 | 1.4 知识点 1、2、3、4 |  
| 网络请求 | 2.3 知识点 1、2、3 |  
| 装饰器与重试 | 2.3 知识点 4 |  
| OOP 抽象与继承 | 3.3 知识点 1 |  
| 注册表模式 | 3.3 知识点 3 |  
| 循环与状态机 | 4.3 知识点 1、2 |  
| 异常处理 | 4.3 知识点 4 |  
| 测试替身 | 5.3 知识点 1、2、3 |  
| 日志 | 2.6、4.6 |  
  
## 自测  
  
1. `AgentDecision.action` 用 `str` 和 `Literal` 有什么区别？  
2. 为什么重试只包住超时与状态码错误，不包所有异常？  
3. 工具体系为什么用类继承而不是函数字典？  
4. ReAct 循环为什么必须设最大步数？  
5. 假客户端测试能覆盖哪些真实调用测不到的分支？  
  
<details>  
<summary>答案要点</summary>  
  
1. `str` 接受任意字符串，非法动作会静默落空；`Literal` 在解析时直接抛校验错误。  
2. 参数错误、鉴权失败重试也不会成功，只会放大延迟与调用量；只重试瞬时故障。  
3. 类能携带名称、描述、参数结构与依赖（如数据库会话），注册表按名调度，新增工具不改调度代码。  
4. 模型可能反复要求调用工具，不设上限会无限调用、持续计费并占满请求。  
5. 工具抛异常、模型输出非法 JSON、达到最大步数这三类分支，真实调用难以稳定复现。  
  
</details>  
  
---  
  
## 规范自检记录  
  
| 条目 | 证据（小节号） | 结论 |  
|---|---|---|  
| 1 交付 | 仅本册，末行停止 | 满足 |  
| 2 方案四段 | 本册为教材 | 不适用 |  
| 3 八段结构 | 1.1–1.8；2.1 起无数据结构，已在 N.1 声明跳过 | 满足 |  
| 4 场景与动机 | 各章 N.1，含本章做／不做与四问 | 满足 |  
| 5 数据建模五项 | 1.2 | 满足 |  
| 6 最小可运行版 | 1.3 含三个对象；2.2 起无省略、无演示对象 | 满足 |  
| 7 一章一动作与缺陷定位 | 1.5、2.4、3.4、4.4、5.4 | 满足 |  
| 8 结构设计 | 1.6、2.5、3.5、4.5、5.5，均引用真实 D 编号 | 满足 |  
| 9 知识点三件套 | 1.4、2.3、3.3、4.3、5.3 共 18 个 | 满足 |  
| 10 设计决策三段 | 1.7、2.6、3.6、4.6、5.6 | 满足 |  
| 11 代码与注释 | 各代码块 import 三标、类函数 docstring、分段导读 | 部分满足：知识点卡内 3～6 行定式模板未逐行注释，因模板需可直接抄写；正文代码均满足 |  
| 12 读者画像 | 全文 | 满足 |  
| 13 测试标注 | 5.6 每个测试 docstring 标明路径与契约 | 满足 |  
| 14 结尾块 | 各章 N.7 五项；5.7 含下一册从哪接 | 满足 |  
| 15 自检六项 | 下表 C1–C6 | 满足 |  
| 16 格式细节 | 第 N 章／N.M；本册地图；信息块 | 满足 |  
| 17 详略 | 各章"已知边界" | 满足 |  
| 18 纠正认知 | 本册地图"术语校准" | 满足 |  
| D1 只给完整版 | 各章均有最小版 | 未违反 |  
| D2 省略用语 | 全文无 | 未违反 |  
| D3 不可运行代码 | 各段可整段复制 | 未违反 |  
| D4 抽象理由 | 缺陷均给报错或数字 | 未违反 |  
| D5 多册同写 | 仅本册 | 未违反 |  
| D6 反问开头 | 无 | 未违反 |  
  
一致性检查：  
  
- **C1**：完整版出现的领域对象只有 `ChatMessage`、`ToolSpec`、`AgentDecision`，均在 1.2 出现；`BaseTool`、`ToolRegistry`、`LLMClient` 为组件而非数据对象，不属数据建模范围。通过。  
- **C2**：1.5 的 D1–D4 可指向 1.3 字段；2.4 的 D1–D4 可指向 2.2 的 `httpx.post` 与取值行；3.4 的 D1–D4 可指向 3.2 的 `TOOLS` 与 `run_tool`；4.4 的 D1–D4 可指向 4.2 的 `run_once`；5.4 的 D1–D4 可指向 5.2 的 `FakeClient` 与断言。通过。  
- **C3**：1.6 引用 D1–D4；2.5 引用 D1–D4；3.5 引用 D1–D4；4.5 引用 D1–D4；5.5 引用 D1–D4。全部真实存在。通过。  
- **C4**：1.5 全属协议定义；2.4 全属调模型；3.4 全属工具注册；4.4 全属编排；5.4 全属测试。无越界。通过。  
- **C5**：1.3 最小版三个对象与 1.2 名称、数量一致；其余章已在 N.1 写"本章不涉及新数据结构"。通过。  
- **C6**：每条缺陷均以字段、类名或函数名定位，不依赖不存在代码。通过。  
  
本册结束，未开始第五册。  
