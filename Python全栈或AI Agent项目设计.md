# Python全栈或AI Agent项目设计

## 1. Python全栈或AI Agent项目设计

**system**: 

```
You are a helpful assistant.
```

**user**: 

```
根据我已学的14天Python内容（基础语法到web开发，测试部署），设计一个全栈或ai agent方向的完整项目，要求：
1.涵盖已学的知识点（环境搭建，控制流程，数据结构，函数结构，oop，文件异常，高级特性，正则，并发，网络，数据库，web开发，测试调试，项目部署）
2.项目结构清晰，能帮助我将零散的知识点整合成“拿到需求就能写”的方法论
3.输出内容应当包含：项目概述，技术栈，功能模块划分各模块对应的知识点，开发步骤建议等
请直接输出项目方案
```

**assistant**: 

```
I'll first record your learning background for future sessions, then output the full project plan.
```

Tool Call: save_memory (state: result)
Args:
  {
    "content": "用户已完成 14 天 Python 学习，覆盖：环境搭建、控制流程、数据结构、函数、OOP、文件与异常、高级特性、正则、并发、网络、数据库、Web 开发、测试调试、项目部署。正在做阶段性的整合项目（倾向全栈或 AI Agent 方向），偏好\"拿到需求就能写\"的方法论式输出与清晰的项目结构。"
  }
Result:
  {
    "message": "Memory saved with id mmva9ew1bsn6b. It will be loaded in future agent sessions."
  }

```
# 项目方案：InsightAgent —— 私有知识库 AI Agent 平台（全栈 + AI Agent）

> 选型说明：这个项目**同时覆盖全栈与 AI Agent 两条线**。它有个天然好处——Agent 的"工具调用、外部 API 调用、并发、异常重试"正好把你学过的所有零散点串成一条真实业务链，而不是为练而练的 demo。

---

## 一、项目概述

### 1.1 一句话定位
用户上传 PDF/Markdown/网页链接 → 系统解析入库 → 通过 Web 界面向 Agent 提问 → Agent 自主决定"查本地知识库 / 联网搜索 / 调计算工具" → 流式返回带引用来源的答案，并留存全部会话历史。

### 1.2 业务价值（面试可讲）
- 解决"资料散落、找不到、记不住"的问题，是一个**可演示、可部署、有真实用户**的产品，而不是玩具。
- 技术上是典型的 **RAG + Agent + 全栈工程化** 组合，是当下企业最主流的落地形态。

### 1.3 核心能力清单
| 能力 | 说明 |
|---|---|
| 文档入库 | 上传/爬取 → 清洗 → 分块 → 向量化 → 落库 |
| 语义检索 | 向量相似 + 关键词混合召回，返回 Top-K 片段 |
| Agent 编排 | LLM 自主选择工具（知识库检索、网页抓取、计算器、时间） |
| 流式对话 | SSE/WebSocket 实时逐字返回 + 来源引用 |
| 会话管理 | 多会话、历史消息、Token 统计 |
| 可观测 | 结构化日志、请求链路 ID、调用耗时统计 |
| 工程化 | pytest 测试、Docker 部署、CI 自动跑测 |

---

## 二、技术栈

| 层 | 选型 | 理由 |
|---|---|---|
| 语言/环境 | Python 3.11+、venv 或 uv、`.env` + python-dotenv | 环境隔离与配置管理 |
| Web 框架 | **FastAPI** + Uvicorn | 异步原生、自动 Swagger 文档，适合 Agent 长请求 |
| 数据校验 | Pydantic v2 | 请求/响应模型，贯穿全项目 |
| 前端 | Jinja2 模板 + 原生 JS（进阶：Vue3 + Vite） | 轻量起步，避免学习负担爆炸 |
| ORM/DB | SQLAlchemy 2.0 + **SQLite**(开发) / **PostgreSQL**(生产) | 先把 ORM 吃透，再切库 |
| 向量存储 | 起步 **Chroma**（本地文件）/ 进阶 pgvector | 不必一上来搞重型方案 |
| LLM 接入 | OpenAI 兼容接口（httpx 异步调用，可换 DeepSeek/Qwen 本地 Ollama） | 统一协议，方便 mock 测试 |
| 爬取 | httpx + asyncio + BeautifulSoup | 异步并发 + HTML 清洗 |
| 任务并发 | `asyncio` + `concurrent.futures`（CPU 型分块/嵌入） | 覆盖两种并发范式 |
| 测试 | pytest + pytest-asyncio + respx/monkeypatch + coverage | mock LLM 是重点 |
| 部署 | Docker + docker-compose + Nginx + GitHub Actions | 完整上线链路 |
| 日志 | logging + 结构化 JSON（可选 loguru） | 调试能力 |

---

## 三、项目结构


insight-agent/
├── app/
│   ├── main.py                 # FastAPI 入口，注册路由/中间件/异常处理器
│   ├── config.py               # Pydantic Settings 读 .env，全局配置单例
│   ├── core/
│   │   ├── logging.py          # 日志配置（控制流程/文件异常）
│   │   ├── exceptions.py       # 自定义异常层级 + 全局异常处理器
│   │   └── security.py         # 简单鉴权/API Key 校验
│   ├── models/                 # SQLAlchemy ORM 模型（OOP + 数据库）
│   │   ├── document.py
│   │   ├── chunk.py
│   │   └── conversation.py
│   ├── schemas/                # Pydantic DTO（请求/响应）
│   ├── db/
│   │   ├── session.py          # 引擎/会话工厂/依赖注入
│   │   └── init_db.py
│   ├── services/
│   │   ├── loader.py           # 文件/网页加载（文件异常 + 正则清洗）
│   │   ├── splitter.py         # 文本分块（生成器 + 正则）
│   │   ├── embedder.py         # 向量化（并发批处理）
│   │   ├── retriever.py        # 抽象基类 + 多种检索实现（OOP 多态）
│   │   └── crawler.py          # 异步并发爬虫（asyncio + httpx）
│   ├── agent/
│   │   ├── base.py             # BaseTool 抽象类 / ToolRegistry
│   │   ├── tools/              # kb_search / web_search / calculator / clock
│   │   ├── llm_client.py       # LLM 调用封装（重试装饰器）
│   │   └── executor.py         # Agent 主循环（ReAct：思考→调用→观察→回答）
│   ├── api/
│   │   ├── deps.py
│   │   └── v1/
│   │       ├── documents.py    # 上传/列表/删除
│   │       ├── chat.py         # 对话（含 SSE 流式）
│   │       └── health.py
│   ├── templates/              # Jinja2 前端页面
│   └── static/                 # JS/CSS
├── tests/
│   ├── conftest.py             # fixture：测试库、假 LLM、TestClient
│   ├── unit/                   # splitter / retriever / tools
│   └── integration/            # API 端到端
├── data/                       # 上传文件、Chroma 持久化目录（gitignore）
├── scripts/                    # 初始化、批量导入、压测脚本
├── .env.example
├── requirements.txt  /  pyproject.toml
├── Dockerfile
├── docker-compose.yml
└── .github/workflows/ci.yml


**这套分层的意义**：`api → services → models/db`，每层只依赖下一层。以后拿到任何需求，先问自己"它属于哪一层"，工程就不会乱——这就是可复用的方法论骨架。

---

## 四、功能模块 ↔ 知识点映射

| 模块 | 具体要做的事 | 对应知识点 | 练法要点 |
|---|---|---|---|
| **M0 项目骨架** | 建 venv、装依赖、`.env` 配置、Git 仓库 | 环境搭建 | 用 Pydantic Settings 统一管配置，杜绝硬编码 |
| **M1 文件加载器** | 读 PDF/TXT/MD，编码容错，大文件流式读 | 文件与异常、高级特性 | 自定义 `FileParseError`；生成器逐行 yield，内存不爆 |
| **M2 文本清洗与分块** | 去 HTML 标签、去多余空行、按句子/长度分块并保留重叠 | **正则**、控制流程、数据结构 | `re.sub` 清标签、`re.split` 断句；循环写滑窗切分 |
| **M3 异步爬虫** | 并发抓取一批 URL、超时/重试/限速、域名去重 | **并发**、网络、控制流程 | `asyncio.gather` + `Semaphore` 限流；`set` 去重；指数退避重试 |
| **M4 向量化与入库** | 批量嵌入、写 SQLite/PG、事务提交、幂等更新 | 数据库、OOP | SQLAlchemy 关系建模（Document 1-N Chunk）；`session.commit()` 与回滚 |
| **M5 检索层** | 抽象 `BaseRetriever` → 向量检索/关键词检索/混合检索三种实现 | **OOP**（继承、抽象类、多态）、数据结构 | 用 Top-K 堆或 `sorted(key=)` 排序；接口一致便于替换 |
| **M6 Agent 与工具** | ReAct 循环、工具注册表、参数校验、最大步数保护 | OOP、函数、异常、控制流程 | `BaseTool` 抽象 + 注册表模式；`try/except` 包裹工具调用，失败反馈给 LLM |
| **M7 LLM 客户端** | 统一封装、超时、重试、限流、Token 统计 | 函数结构（**装饰器**）、网络、异常 | 写 `@retry(times=3, backoff=2)` 和 `@timer` 装饰器，闭包实现 |
| **M8 Web API** | 上传、检索、对话（SSE 流式）、历史查询 | **Web 开发**、网络 | FastAPI 依赖注入、`StreamingResponse`、中间件加请求 ID |
| **M9 前端界面** | 聊天窗、上传区、引用来源卡片、Markdown 渲染 | Web 开发 | Jinja2 + fetch + `EventSource` 接收流 |
| **M10 测试与调试** | 单测 + 集成测试、mock LLM、覆盖率、日志排查 | **测试调试** | `monkeypatch` 假 LLM；fixture 建测试库；`breakpoint()` 调试 |
| **M11 部署上线** | Dockerfile、compose、Nginx 反代、CI、云主机 | **项目部署** | 多阶段构建瘦身镜像；健康检查；GitHub Actions 跑 pytest |

> 知识点全覆盖校验：环境搭建(M0) ✅ 控制流程(M2/M3/M6) ✅ 数据结构(M2/M3/M5) ✅ 函数结构(M7) ✅ OOP(M4/M5/M6) ✅ 文件异常(M1/M6) ✅ 高级特性(M1/M2/M7) ✅ 正则(M2) ✅ 并发(M3/M4) ✅ 网络(M3/M7/M8) ✅ 数据库(M4) ✅ Web 开发(M8/M9) ✅ 测试调试(M10) ✅ 部署(M11) ✅

---

## 五、"拿到需求就能写"的方法论（贯穿全程的六步法）

每次做新功能都套这套流程，做完这个项目你会形成肌肉记忆：

1. **拆需求 → 定验收**：一句话需求拆成 3~5 条可验证的验收标准（"能上传 PDF 并返回 chunk 数"）。
2. **建模 → 画数据结构**：先定 Pydantic 输入输出，再定数据库表。**数据结构对了，代码就顺了。**
3. **分层 → 定归属**：是 API 层、service 层还是 agent 层？不确定就说明职责没想清。
4. **先跑通主干，再补分支**：先写 happy path（能跑通），再补异常、重试、边界。
5. **边写边测**：核心纯函数（分块、清洗、排序）先写测试，再写实现。
6. **可观测 + 可部署**：加日志 → 加健康检查 → 能 Docker 起来，才算完成。

---

## 六、开发步骤建议（建议 12~16 天，每天 2~4 小时）

### 阶段一：地基（Day 1-2）
- 建仓库、虚拟环境、目录骨架、`config.py`、日志、全局异常处理
- **验收**：`uvicorn app.main:app --reload` 启动，`/health` 返回 200，Swagger 能打开

### 阶段二：数据入口（Day 3-4）
- M1 文件加载 + M2 文本清洗与分块（正则重点在这里）
- **验收**：上传一个 PDF，日志打印出 chunk 列表和数量；分块函数有单测

### 阶段三：数据库与检索（Day 5-6）
- M4 ORM 建模 + 入库；M5 检索抽象与两种实现
- **验收**：`/documents` 能列出文档，`/search?q=xxx` 能返回相关片段

### 阶段四：并发爬虫（Day 7）
- M3 异步抓取 + 清洗 + 入库，串起前面所有模块
- **验收**：给 20 个 URL，5 秒内完成抓取入库，失败的有日志有重试

### 阶段五：Agent 核心（Day 8-10）
- M7 LLM 客户端（装饰器重试）→ M6 工具 + ReAct 主循环
- **验收**：提问"我们文档里提到的部署方式是什么"能触发 `kb_search` 并给出带引用的回答

### 阶段六：Web 与前端（Day 11-12）
- M8 上传/对话接口 + SSE 流式；M9 前端聊天页
- **验收**：浏览器里能上传文档、聊天、看到逐字输出和来源卡片

### 阶段七：测试与调试（Day 13-14）
- M10 补齐单测/集成测试、mock LLM、覆盖率 > 70%，练一次用日志 + breakpoint 定位真实 bug
- **验收**：`pytest -v` 全绿，覆盖率报告可看

### 阶段八：部署上线（Day 15-16）
- M11 Dockerfile + compose（app + db + nginx）、CI 流水线、部署到云主机或 Render/Fly.io
- **验收**：公网 IP 可访问，`docker compose up -d` 一键起服务，push 代码自动跑测试

---

## 七、避坑与加分项

**避坑**
- 别一上来做多用户权限、复杂前端，**先让主线闭环**。
- LLM 调用**必须可 mock**，否则测试慢且花钱。
- 分块策略（大小、重叠）直接决定效果，多试 2~3 组参数。
- API Key 一律走 `.env`，`.env` 进 `.gitignore`。

**加分**（做完主线再加）
- 换成 PostgreSQL + pgvector，写 Alembic 迁移
- 加 Redis 缓存检索结果与对话上下文
- 加 `pyproject.toml` + ruff/black 代码规范
- 加简单的 RAG 评估脚本（召回率/命中率）
- 前端升级 Vue3 + 组件化

---

## 八、若想要更轻的替代方案

如果觉得 AI 部分压力大，可做**纯全栈版**：`任务协作平台`（FastAPI + SQLAlchemy + JWT 鉴权 + Vue 前端 + Docker 部署 + pytest），知识点覆盖几乎一致，只是把 M5/M6/M7 换成"权限系统 + 业务逻辑层"。建议先按主方案走到阶段四，你会发现自己已经有能力驾驭它了。

---

需要的话，我可以继续帮你做三件事：① 输出阶段的 Day 1 详细任务清单和首个可运行代码骨架；② 把 M2 分块 / M3 异步爬虫 / M6 ReAct 循环给出核心代码范式；③ 生成本项目的 `requirements.txt` 与 `Dockerfile`。你想先要哪个？
```

**user**: 

```
按照开发步骤建议生成阶段一的代码，需要：代码注释清晰，调用库，方法的代码注释作用或用法，确保看完能记住遇到这个阶段的需求该怎么写的方法论
```

**assistant**: 

```
# 阶段一（Day 1-2）：项目地基 —— 可运行的空壳骨架

## 0. 本阶段目标与验收标准

| 目标 | 验收标准 |
|---|---|
| 环境隔离 | `pip install -r requirements.txt` 一次成功，无版本冲突 |
| 配置集中 | 所有可变项来自 `.env`，代码里**零** `os.getenv()` 散落 |
| 日志可用 | 控制台 + 文件双输出，带时间/级别/模块名 |
| 异常统一 | 任何错误都返回统一 JSON 结构，不暴露堆栈给用户 |
| 服务可起 | `uvicorn app.main:app --reload` 启动，`/health` 返回 200，Swagger 能打开 |
| 有测试 | `pytest` 一条命令跑通第一个用例 |

> **阶段一的方法论一句话**：先搭"骨架 + 三个全局设施（配置/日志/异常）"，再写业务。这三样是所有后续模块的地基，后面 10 个阶段都在复用它们。

---

## 1. 目录结构（本阶段实际创建的文件）


insight-agent/
├── app/
│   ├── __init__.py
│   ├── main.py              # ① 应用入口：装配车间
│   ├── config.py            # ② 全局配置：唯一配置源
│   ├── core/
│   │   ├── __init__.py
│   │   ├── logging.py       # ③ 日志设施
│   │   └── exceptions.py    # ④ 异常体系 + 全局处理器
│   └── api/
│       ├── __init__.py
│       └── v1/
│           ├── __init__.py
│           └── health.py    # ⑤ 第一个路由：健康检查
├── tests/
│   ├── __init__.py
│   └── test_health.py       # ⑥ 第一个测试
├── logs/                    # 日志输出目录（运行时自动创建）
├── .env                     # 本地真实配置（进 .gitignore，绝不提交）
├── .env.example             # 配置模板（提交 Git，给协作者看）
├── .gitignore
├── requirements.txt
└── README.md


---

## 2. 环境搭建命令（照抄即可）


mkdir insight-agent && cd insight-agent
python3 -m venv .venv

# macOS / Linux
source .venv/bin/activate
# Windows (PowerShell)
# .venv\Scripts\Activate.ps1

pip install --upgrade pip
pip install -r requirements.txt   # 文件内容见下
cp .env.example .env              # 复制模板，改成本地值
git init && git add . && git commit -m "chore: 阶段一项目骨架"


**`requirements.txt`**


# ---- Web 框架层 ----
fastapi==0.115.0            # 主框架：路由、依赖注入、自动 Swagger 文档
uvicorn[standard]==0.30.6   # ASGI 服务器：真正跑 FastAPI 的进程（--reload 靠它）
python-multipart==0.0.9     # 解析文件上传（不给它装，UploadFile 会直接报错）

# ---- 配置校验层 ----
pydantic==2.9.2             # 数据校验与序列化（贯穿全项目的"数据类型安全网"）
pydantic-settings==2.5.2    # 让 Pydantic 模型能从 .env / 环境变量读配置

# ---- 网络与工具 ----
httpx==0.27.2               # 异步 HTTP 客户端（阶段七调 LLM、阶段四爬虫都用它）
python-dotenv==1.0.1        # 读取 .env（pydantic-settings 的底层依赖，显式写上便于理解）

# ---- 测试层 ----
pytest==8.3.3               # 测试框架
pytest-asyncio==0.24.0      # 让 pytest 能跑 async def 测试函数
pytest-cov==5.0.0           # 覆盖率报告


**方法论要点**：每个依赖后面写一行注释说明"为什么需要它"。半年后回来看，你会感谢自己——**依赖清单本身就是一份架构说明**。

---

## 3. `.env.example` 与 `.gitignore`


# ---- .env.example：配置模板（放进 Git，供他人复制）----
# 用法：cp .env.example .env，然后按本地情况修改 .env 里的值

APP_NAME=InsightAgent
APP_ENV=dev                 # dev / test / prod
DEBUG=true
HOST=0.0.0.0
PORT=8000

LOG_LEVEL=INFO              # DEBUG / INFO / WARNING / ERROR
LOG_DIR=logs

# 阶段三之后才会用到，先占位（配置项"预留"是好习惯，接口不用改）
DATABASE_URL=sqlite:///./data/insight.db
VECTOR_STORE_DIR=./data/chroma

# 保密项：真实 Key 只写在 .env，绝不写进代码和 Git
LLM_API_KEY=sk-xxx
LLM_BASE_URL=https://api.openai.com/v1
LLM_MODEL=gpt-4o-mini



# ---- .gitignore ----
.venv/
__pycache__/
*.py[cod]
.env                 # 关键：环境变量文件绝不提交
logs/
data/
.pytest_cache/
.coverage
htmlcov/
.idea/
.vscode/
.DS_Store


---

## 4. `app/config.py` —— 全局唯一配置源


"""全局配置模块。

【方法论】
配置只在一处读取、一处暴露。其他任何模块一律写 `from app.config import settings`，
绝不允许出现散落的 os.getenv("XXX")。
好处：① 换环境只改 .env，不动代码；② 配置项有类型和默认值，写错立刻报错；
     ③ 测试时可以只覆盖这一个对象。

【用法】
    from app.config import settings
    print(settings.PORT)          # 8000
"""
from functools import lru_cache

from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """应用配置类。每个字段 = 一个配置项，字段的类型注解就是校验规则。"""

    # model_config 是 Pydantic v2 的固定写法：定义"这个类怎么从外部读数据"
    model_config = SettingsConfigDict(
        env_file=".env",          # 从 .env 文件读取（本地开发）
        env_file_encoding="utf-8",
        case_sensitive=False,     # 环境变量名大小写不敏感：APP_PORT 和 app_port 都认
        extra="ignore",           # 忽略 .env 里多余的键，避免加个变量就启动失败
    )

    # ---------- 应用信息 ----------
    APP_NAME: str = "InsightAgent"
    # Literal 表示"只能取这几个值"，写错会在启动时立刻报错（比运行时才发现强得多）
    APP_ENV: str = Field(default="dev", pattern="^(dev|test|prod)$")
    DEBUG: bool = True            # 注意：Pydantic 会把字符串 "true" 自动转成布尔 True
    HOST: str = "0.0.0.0"         # 0.0.0.0 = 监听所有网卡（Docker 里必须这么写）
    PORT: int = 8000

    # ---------- 日志 ----------
    LOG_LEVEL: str = "INFO"
    LOG_DIR: str = "logs"         # 日志文件目录，阶段一自动创建

    # ---------- 外部资源（阶段三/七启用，阶段一先占位）----------
    DATABASE_URL: str = "sqlite:///./data/insight.db"
    VECTOR_STORE_DIR: str = "./data/chroma"

    # ---------- 大模型（保密信息，无默认值）----------
    # 写成可选类型 + 默认 None：阶段一没配 Key 也能正常启动，不阻塞开发
    LLM_API_KEY: str | None = None
    LLM_BASE_URL: str = "https://api.openai.com/v1"
    LLM_MODEL: str = "gpt-4o-mini"

    @property
    def is_dev(self) -> bool:
        """便捷属性：判断是否开发环境。用法：if settings.is_dev: ..."""
        return self.APP_ENV == "dev"


@lru_cache
def get_settings() -> Settings:
    """获取配置单例。

    【lru_cache 的作用】带缓存：第一次调用真正实例化并在内部记住结果，
    之后每次调用直接返回同一个对象 —— 天然实现"单例"，避免到处重复读 .env。
    【测试友好】测试里需要临时改配置时，可调用 get_settings.cache_clear()
    清掉缓存，再重新读取 —— 这是 lru_cache 选它的核心理由。
    """
    return Settings()


# 模块级单例：全项目统一从这里导入，例如 from app.config import settings
settings = get_settings()


**记忆口诀**：**配置类 = 字段 + 类型 + 默认值；读一次，导处用，永不散落 `getenv`。**

---

## 5. `app/core/logging.py` —— 日志设施


"""日志配置模块。

【方法论】日志只在程序启动时配置一次，之后全项目用 logging.getLogger(__name__) 拿 logger。
① 不要用 print（没有级别、没有时间、无法关掉、无法落文件）；
② 不要在业务代码里重复 addHandler（会造成每条日志打印 N 遍）；
③ 日志文件名建议直接用模块名，出问题时能一眼定位到文件。

【用法】
    from app.core.logging import get_logger
    logger = get_logger(__name__)
    logger.info("用户上传文件: %s", filename)   # 用 %s 占位，不要写 f-string
"""
import logging
import sys
from logging.handlers import RotatingFileHandler
from pathlib import Path

from app.config import settings

# 统一格式：时间 | 级别 | 模块名 | 消息
# %(name)s 就是 getLogger(__name__) 传进去的模块名，这是定位问题的关键字段
LOG_FORMAT = "%(asctime)s | %(levelname)-8s | %(name)s | %(message)s"
DATE_FORMAT = "%Y-%m-%d %H:%M:%S"


def setup_logging() -> None:
    """初始化全局日志（只在应用启动时调用一次）。

    做三件事：① 定级别；② 加"控制台"处理器；③ 加"文件"处理器（自动切割）。
    """
    root_logger = logging.getLogger()

    # 【幂等保护】uvicorn --reload 会重复导入模块，若不检查就会重复添加 handler，
    # 导致同一条日志打印多次。这是新手最常见的日志坑，务必加这个判断。
    if root_logger.handlers:
        return

    root_logger.setLevel(settings.LOG_LEVEL.upper())

    formatter = logging.Formatter(LOG_FORMAT, datefmt=DATE_FORMAT)

    # ---------- 处理器 1：输出到控制台（开发时看）----------
    # sys.stdout 比 sys.stderr 更友好（Docker logs 抓取更干净）
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setFormatter(formatter)
    root_logger.addHandler(console_handler)

    # ---------- 处理器 2：输出到文件，并按大小自动轮转 ----------
    log_dir = Path(settings.LOG_DIR)
    log_dir.mkdir(parents=True, exist_ok=True)   # 目录不存在就建，已存在不报错

    # RotatingFileHandler = 日志文件写满后自动改名并新建，防止单文件无限膨胀
    # maxBytes=10MB 后切分，backupCount=5 表示最多保留 5 个历史文件
    file_handler = RotatingFileHandler(
        filename=log_dir / "app.log",
        maxBytes=10 * 1024 * 1024,
        backupCount=5,
        encoding="utf-8",       # 不写中文会乱码
    )
    file_handler.setFormatter(formatter)
    root_logger.addHandler(file_handler)

    # 降噪：把第三方库的日志级别调高，避免 httpx/matplotlib 之类刷屏
    logging.getLogger("httpx").setLevel(logging.WARNING)

    logging.getLogger(__name__).info(
        "日志初始化完成 | 环境=%s | 级别=%s | 目录=%s",
        settings.APP_ENV, settings.LOG_LEVEL, settings.LOG_DIR,
    )


def get_logger(name: str) -> logging.Logger:
    """获取 logger 的便捷函数（本项目统一入口）。

    参数 name 传 __name__ 即可，这样日志里会显示 "app.api.v1.health" 这样的模块路径，
    排查问题时能直接知道是哪个文件打的日志。
    """
    return logging.getLogger(name)


**记忆口诀**：**配置一次，取之 `getLogger(__name__)`；handler 加前先判空。**

---

## 6. `app/core/exceptions.py` —— 异常体系与全局处理器


"""异常体系与全局异常处理器。

【方法论：异常分三层，永不裸奔】
第一层——业务异常（自己抛）：可预期，如"文档不存在""LLM 调用失败"。
        特点：带 status_code + code + 人类可读的 message，前端能直接展示。
第二层——参数校验异常（FastAPI 自动抛）：用户传错参数，交给框架的 ValidationError。
第三层——未知异常（SystemExit/Exception 兜底）：真正出 bug 了，记完整堆栈到日志，
        但只给用户返回"服务器内部错误"，不泄露实现细节。

核心思想：**业务代码只负责 raise，格式统一的工作交给全局处理器。**
好处：新增一个错误场景时，只需定义一个异常类，不用改任何响应逻辑。
"""
from fastapi import FastAPI, Request, status
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

from app.core.logging import get_logger

logger = get_logger(__name__)


# ============================================================
# 第一部分：业务异常类（只有一个基类带逻辑，子类只需声明差异）
# ============================================================
class AppError(Exception):
    """所有业务异常的基类。

    【用法】
        raise AppError("出错了")                        # 用默认 status/code
        raise NotFoundError("文档 123 不存在")           # 用子类的默认值
    """

    # 类属性作为默认值：子类只需覆盖想要的字段，代码极简 —— 这是 OOP 继承的典型用法
    status_code: int = status.HTTP_500_INTERNAL_SERVER_ERROR
    code: str = "INTERNAL_ERROR"
    message: str = "服务器内部错误"

    def __init__(self, message: str | None = None, *, details: dict | None = None):
        # 允许调用时覆盖默认 message（传了就用自己的，没传就用类属性）
        self.message = message or self.message
        self.details = details or {}      # 附加信息，如 {"doc_id": 123}
        super().__init__(self.message)    # 让 str(exc) 也能拿到消息，方便日志


# 下面这些子类：只改类属性，不写一行方法体 —— 极简且语义清晰
class NotFoundError(AppError):
    """资源不存在。例：raise NotFoundError("文档 123 不存在")"""
    status_code = status.HTTP_404_NOT_FOUND
    code = "NOT_FOUND"


class InvalidInputError(AppError):
    """业务层参数非法（框架校验管不到的规则，比如"文件类型不支持"）。"""
    status_code = status.HTTP_400_BAD_REQUEST
    code = "INVALID_INPUT"


class UpstreamError(AppError):
    """调用外部服务失败（LLM API、爬虫目标站）。阶段七会大量用到。"""
    status_code = status.HTTP_502_BAD_GATEWAY
    code = "UPSTREAM_ERROR"


# ============================================================
# 第二部分：统一响应结构
# ============================================================
def _error_response(status_code: int, code: str, message: str, details: dict) -> JSONResponse:
    """把异常渲染成统一 JSON：{code, message, details}。

    为什么统一结构？前端只需写一次错误处理逻辑，不用为每个接口适配不同格式。
    """
    return JSONResponse(
        status_code=status_code,
        content={"code": code, "message": message, "details": details},
    )


# ============================================================
# 第三部分：处理器函数（在 main.py 里注册到 app 上）
# ============================================================
async def app_error_handler(request: Request, exc: AppError) -> JSONResponse:
    """处理自己抛的业务异常。

    【为什么记 warning 而不是 error】业务异常是"预期内的正常分支"（比如用户查了不存在的文档），
    不算系统故障，用 warning 级别，避免污染 error 日志、影响告警。
    """
    logger.warning(
        "[%s] 业务异常 | %s %s | code=%s | msg=%s",
        getattr(request.state, "request_id", "-"),
        request.method, request.url.path, exc.code, exc.message,
    )
    return _error_response(exc.status_code, exc.code, exc.message, exc.details)


async def validation_error_handler(
    request: Request, exc: RequestValidationError
) -> JSONResponse:
    """处理 FastAPI 自动抛出的参数校验异常（如缺字段、类型不对）。

    默认响应体的结构是 list[dict]，很不友好；这里压平成一句话，并保留原始错误明细。
    """
    errors = exc.errors()
    first = errors[0] if errors else {}
    # loc 形如 ("body", "content")，拼成 "body.content" 便于定位
    location = ".".join(str(p) for p in first.get("loc", []))
    message = f"参数校验失败: {location} - {first.get('msg', '未知错误')}"

    logger.warning("[参数校验] %s %s | %s", request.method, request.url.path, message)
    return _error_response(422, "VALIDATION_ERROR", message, {"errors": errors})


async def unhandled_error_handler(request: Request, exc: Exception) -> JSONResponse:
    """兜底：处理所有未预料到的异常。

    【关键】必须 logger.exception(...)，它会自动带上完整堆栈 —— 这是我们排查 bug 的唯一线索。
    同时对外只返回笼统信息，绝不把堆栈/表名/SQL 返回给客户端（安全红线）。
    """
    logger.exception(
        "[%s] 未捕获异常 | %s %s",
        getattr(request.state, "request_id", "-"),
        request.method, request.url.path,
    )
    return _error_response(500, "INTERNAL_ERROR", "服务器内部错误", {})


def register_exception_handlers(app: FastAPI) -> None:
    """把所有异常处理器注册到 FastAPI 应用上（main.py 启动时调用一次）。

    【用法】把异常类型和处理函数配对注册。之后任何地方 raise AppError 的子类，
    FastAPI 都会自动走到对应的 handler —— 这就是"业务代码只管 raise"的实现机制。
    """
    app.add_exception_handler(AppError, app_error_handler)
    app.add_exception_handler(RequestValidationError, validation_error_handler)
    app.add_exception_handler(Exception, unhandled_error_handler)


**记忆口诀**：**业务抛异常，全局来兜底；堆栈进日志，笼统给用户。**

---

## 7. `app/api/v1/health.py` —— 第一个路由


"""健康检查路由。放第一个接口，因为它是"服务是否活着"的最小验证。

【方法论：路由文件只做三件事】
① 声明路径与方法；② 用 Pydantic 声明入参/出参；③ 把逻辑委托给 service 层。
阶段一还没有 service，所以 /health 直接返回；从阶段三开始，所有路由都变成"薄壳"。
"""
from fastapi import APIRouter
from pydantic import BaseModel

from app.config import settings
from app.core.logging import get_logger

logger = get_logger(__name__)

# APIRouter = 路由分组器。好处：按业务模块拆文件，最后在 main.py 统一挂载
router = APIRouter(tags=["系统"])


class HealthResponse(BaseModel):
    """响应模型（Pydantic）。

    【为什么必须定义它】① 自动生成 Swagger 文档；② 保证出参字段和类型稳定不跑偏；
    ③ 前端可据此生成 TS 类型。任何接口都应该有明确的响应模型。
    """
    status: str
    app_name: str
    env: str
    version: str


@router.get("/health", response_model=HealthResponse, summary="健康检查")
async def health_check() -> HealthResponse:
    """服务存活探针。

    用途：① 本地验证服务是否起来了；② Docker/K8s 健康检查；③ 云平台探活。
    注意：返回类型写成 HealthResponse，配合 response_model，FastAPI 会自动做校验。
    """
    logger.debug("健康检查被调用")
    return HealthResponse(
        status="ok",
        app_name=settings.APP_NAME,
        env=settings.APP_ENV,
        version="0.1.0",
    )


---

## 8. `app/main.py` —— 应用入口（装配车间）


"""FastAPI 应用入口。

【方法论：main.py 只做四件事，永远是"装配车间"而不是"生产车间"】
① 创建 app（含 lifespan 生命周期管理）；
② 挂中间件（请求 ID、耗时统计、CORS）；
③ 注册路由（把各模块 router 挂上来）；
④ 注册异常处理器。
业务逻辑一律不写在这里。以后加功能，是"新增一个 router 文件 + 这里挂一行"，
而不是往 main.py 里堆代码 —— 这就是项目能长大的原因。
"""
import time
import uuid
from contextlib import asynccontextmanager

from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware

from app.api.v1 import health
from app.config import settings
from app.core.exceptions import register_exception_handlers
from app.core.logging import get_logger, setup_logging

logger = get_logger(__name__)


# ---------------- 生命周期管理 ----------------
@asynccontextmanager
async def lifespan(app: FastAPI):
    """应用生命周期：yield 前是启动逻辑，yield 后是关闭清理逻辑。

    【为什么用它而不是 @app.on_event("startup")】on_event 已被官方废弃；
    lifespan 用一个函数同时表达"启动-运行-关闭"三阶段，还支持依赖注入，是现在的标准写法。

    【以后会往这里加什么】
    启动：初始化数据库连接池、加载向量库、预热 LLM 客户端
    关闭：关闭连接池、flush 日志 —— 保证资源不泄漏
    """
    # ---- 启动阶段 ----
    setup_logging()                       # 日志必须最先初始化，否则后面打的日志会丢
    logger.info(
        "🚀 %s 启动中 | env=%s | debug=%s", settings.APP_NAME, settings.APP_ENV, settings.DEBUG
    )
    yield
    # ---- 关闭阶段 ----
    logger.info("👋 %s 已关闭", settings.APP_NAME)


# ---------------- 创建应用 ----------------
app = FastAPI(
    title=settings.APP_NAME,
    version="0.1.0",
    description="私有知识库 AI Agent 平台",
    # DEBUG 时开启交互式文档；生产环境关掉可以减少信息暴露
    docs_url="/docs" if settings.DEBUG else None,
    redoc_url="/redoc" if settings.DEBUG else None,
    lifespan=lifespan,
)

# ---------------- 中间件 ----------------
# CORS：允许前端页面（不同端口/域名）跨域调用本 API。不加的话浏览器会直接拦截请求。
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"] if settings.is_dev else ["https://your-domain.com"],  # 生产必须收紧
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


@app.middleware("http")
async def access_log_middleware(request: Request, call_next):
    """请求上下文中间件：给每个请求分配 request_id，并记录耗时。

    【作用】① request_id 串起一次请求的所有日志，排查问题时能一眼筛出同一次调用；
    ② 耗时统计让你提前发现慢接口（阶段八调 LLM 时尤其重要）。

    【用法说明】函数签名固定为 (request, call_next)：
    先处理入站请求 → 调用 call_next 交给下游 → 再处理出站响应。
    request.state 是 Starlette 提供的"请求级存储"，可跨函数传递数据。
    """
    # 优先用上游（如 Nginx）传下来的 ID，实现全链路追踪；没有就自己生成
    request_id = request.headers.get("X-Request-ID") or uuid.uuid4().hex[:8]
    request.state.request_id = request_id

    start = time.perf_counter()            # perf_counter 精度最高，专用于测耗时
    response = await call_next(request)
    cost_ms = (time.perf_counter() - start) * 1000

    response.headers["X-Request-ID"] = request_id   # 回写给客户端，便于报障时对日志
    logger.info(
        "[%s] %s %s -> %s | %.1fms",
        request_id, request.method, request.url.path, response.status_code, cost_ms,
    )
    return response


# ---------------- 注册路由 ----------------
# prefix="/api/v1" 做版本管理：以后接口不兼容升级时，开 /api/v2 而不用改老接口
app.include_router(health.router, prefix="/api/v1")

# ---------------- 注册异常处理器 ----------------
register_exception_handlers(app)


---

## 9. `tests/test_health.py` —— 第一个测试


"""阶段一的测试：验证服务骨架可用。

【方法论：测试从第一天开始写，不要等"功能全做完再补"】
理由：地基测试极便宜，但它能保证后续每次改动（加中间件、改异常处理）
都没有把服务搞崩 —— 这就是回归保护网。
"""
from fastapi.testclient import TestClient

from app.main import app


def test_health_check_ok():
    """健康检查应返回 200 且 status=ok。

    TestClient 是 FastAPI 提供的测试客户端：它不需要启动真实服务器，
    直接以进程内方式调用 app，所以测试快且不占用端口。
    【用法】用 with 语句创建，会正确触发 lifespan 的启动/关闭逻辑。
    """
    with TestClient(app) as client:
        response = client.get("/api/v1/health")

    assert response.status_code == 200
    body = response.json()
    assert body["status"] == "ok"
    assert body["app_name"] == "InsightAgent"


def test_request_id_header_returned():
    """每个响应都应带上 X-Request-ID 头（中间件生效验证）。"""
    with TestClient(app) as client:
        response = client.get("/api/v1/health")

    assert "X-Request-ID" in response.headers


def test_404_uses_unified_response_shape():
    """访问不存在的路径，应返回统一 JSON 结构，而不是 HTML 错误页。"""
    with TestClient(app) as client:
        response = client.get("/api/v1/not-exist")

    assert response.status_code == 404


---

## 10. 验收操作（跑完这几条，阶段一就算交付）


# 1) 启动服务（--reload：改代码自动重启，开发必备）
uvicorn app.main:app --reload --port 8000

# 2) 新开一个终端，验证接口
curl http://127.0.0.1:8000/api/v1/health
# 期望：{"status":"ok","app_name":"InsightAgent","env":"dev","version":"0.1.0"}

# 3) 看响应头里有没有 X-Request-ID
curl -i http://127.0.0.1:8000/api/v1/health | grep -i x-request-id

# 4) 浏览器打开交互式文档，点开 /health 直接试
open http://127.0.0.1:8000/docs

# 5) 跑测试并看覆盖率
pytest -v --cov=app --cov-report=term-missing

# 6) 确认日志落盘
cat logs/app.log


同时你会观察到：`logs/app.log` 里有启动日志、请求访问日志；控制台里没有重复打印（幂等保护生效）。

---

## 11. 阶段一方法论总结卡（背下这 6 条）

| # | 口诀 | 对应文件 | 遇到同类需求怎么做 |
|---|---|---|---|
| 1 | **配置集中一处，字段带类型和默认值** | `config.py` | 新建 `XxxSettings(BaseSettings)`，`.env` 加一条，代码里导 `settings` |
| 2 | **日志配一次，`getLogger(__name__)` 取用，加 handler 前判空** | `core/logging.py` | 用 `%s` 占位、不用 f-string；异常用 `logger.exception` 带堆栈 |
| 3 | **异常分业务/校验/未知三层，业务只管 raise，全局统一响应** | `core/exceptions.py` | 新错误场景 = 新继承 `AppError` 的类 + 类属性，不改响应逻辑 |
| 4 | **入口只做装配：app、中间件、路由、异常处理器** | `main.py` | 加功能 = 新增 router 文件 + 这里加一行 include_router |
| 5 | **路由是薄壳：声明路径/出入参模型，逻辑交给 service** | `api/v1/health.py` | 每个接口都定义 `response_model`（Pydantic 出参模型） |
| 6 | **测试从第一天开始，TestClient 进程内调用** | `tests/` | 新接口必配一个 happy-path 用例，用 `with TestClient(app)` |

**一句话串起来**：
> 建骨架 → 集中配置（`settings`）→ 装日志（`setup_logging`）→ 定异常（`AppError` + handler）→ 挂路由（`APIRouter`）→ 加中间件（request_id + 耗时）→ 写第一个测试。
> **这套流程对任何 Python Web 项目都通用，换框架也只需替换第 4~5 步的具体写法。**

---

## 12. 阶段一交给你自己做的两个小练习

1. **加一个 `/api/v1/ping` 接口**，返回当前服务器时间（练"新增路由 + 挂载"的完整流程）。
2. **故意制造一个异常**（例如在接口里 `raise NotFoundError("测试")`，再 `raise ValueError("boom")`），观察两种情况的响应体和日志差异——亲手感受"业务异常 vs 未知异常"的处理区别。

完成后我们进入**阶段二（Day 3-4）：文件加载 + 文本清洗与分块**，那里是正则和文件异常的主战场。要我继续输出阶段二吗？
```

**user**: 

```
读了第一阶段的代码后，发现一些问题：除了整体的框架该怎么做之外，对于代码完全看不懂，疑问一：配置，日志，异常，路由代码中导入的类是不是固定的，包括调用的方法，如果只告诉让我写个logger.py我都不知道要导哪些包和哪些方法调用，没有个固定的模板可套
```

**assistant**: 

```
我先记录你这个学习痛点，后续代码讲解会按“可背模板 + 导入逐行解释”的方式来写。
```

Tool Call: save_memory (state: result)
Args:
  {
    "content": "用户 Python 基础较薄弱：能理解项目框架思路，但看具体代码时看不懂 import 了哪些包、类和方法从哪来，需要\"可背的最小模板 + 每个 import 对应什么角色\"级别的讲解，而不是直接给完整工程代码。讲解代码时应先给 10 行左右的骨架模板，再说明扩展项。"
  }
Result:
  {
    "message": "Memory saved with id mmveza9ltsp9l. It will be loaded in future agent sessions."
  }

```
# 直接回答你的疑问一

## 先给结论

**你问“import 的类是不是固定的”，答案是：库和 API 是固定的，但“写法”不唯一。**

分三层看：

| 层面 | 是否固定 | 说明 |
|---|---|---|
| 用哪个库 | ✅ 固定 | 写日志就是 `logging`，读配置就是 `pydantic-settings`，写路由就是 `fastapi`。这是“选型”，阶段一已经替你定了 |
| 库里的类名/函数名 | ✅ 固定 | `getLogger`、`StreamHandler`、`BaseSettings`、`APIRouter` 这些是官方 API，不会变。查文档能查到 |
| 怎么组合它们 | ❌ 不固定 | 同一条日志可以 `basicConfig` 一行搞定，也可以手写 handler 精细控制。两种都对 |

所以 **你不需要背 import 列表**，你需要背的是：**“这个文件要完成几件事，每件事对应一个什么对象”。** 对象定了，import 自然就出来了。

我下面给你一套 **推导法 + 4 个 10 行模板 + 一张速查表**。

---

## 一、万能推导法：从“中文需求”推到 import

不要一上来想“我要 import 什么”，而是先写中文任务清单，再把**名词**圈出来。

以写一个 `logger.py` 为例，你的真实需求是：

> 我要一个日志器 → 能设置级别 → 能输出到控制台 → 能格式化（时间、级别、模块、消息）

把名词翻译成英文，基本就是类名：

| 中文名词 | 英文对象 | 来自哪个包 |
|---|---|---|
| 日志器 | `Logger` / `getLogger()` | `logging` |
| 级别 | `INFO` 等常量 | `logging` |
| 输出到控制台 | `StreamHandler` | `logging` |
| 输出到文件 | `FileHandler` | `logging` |
| 文件太大自动切分 | `RotatingFileHandler` | `logging.handlers` |
| 格式化 | `Formatter` | `logging` |
| 输出流 | `sys.stdout` | `sys` |
| 文件路径 | `Path` | `pathlib` |

**规律：类名几乎就是英文直译。** `Request`=请求，`JSONResponse`=JSON 响应，`StreamHandler`=流处理器，`BaseSettings`=配置基类。你只要知道“我要一个什么东西”，英文名八九不离十，再靠 IDE 补全确认。

**这就是方法论第一句：先想“我要什么对象”，再想“它在哪个包”，永远不要反过来死记 import。**

---

## 二、4 个最小可背模板（先背这个，再看项目版）

我阶段一给你的版本是“生产可用版”，所以多了轮转、幂等、异常分层这些东西。**你现在需要的是先背 10 行骨架**，骨架会了，再去理解多出来的每一块在补什么。

### 模板 1：`logger.py`（8 行核心）


import logging
import sys

def setup_logging(level: str = "INFO") -> None:
    """配置日志：设置级别 + 输出到控制台 + 定格式。"""
    logging.basicConfig(
        level=level,                                      # 低于这个级别的日志不输出
        format="%(asctime)s | %(levelname)s | %(name)s | %(message)s",  # 占位符是 logging 内置的
        handlers=[logging.StreamHandler(sys.stdout)],     # 输出目标：控制台
    )

def get_logger(name: str) -> logging.Logger:
    """业务代码统一用它拿 logger，name 传 __name__。"""
    return logging.getLogger(name)


**这就是全部。** 记住 3 个调用：

1. `logging.basicConfig(...)` —— 一次性配置“根日志器”，内部自动帮你 `addHandler`。
2. `logging.StreamHandler(sys.stdout)` —— “往控制台输出”这个动作的对象。
3. `logging.getLogger(__name__)` —— 拿到本模块的日志器。

**然后项目版多出来的东西，逐个对应一个需求**（这样你就能“看懂”而不是“背下”）：

| 项目版多出来的代码 | 它解决什么问题 |
|---|---|
| `if root_logger.handlers: return` | `basicConfig` 只能生效一次，`--reload` 重复导入会导致日志重复。所以改成手写 handler + 判空 |
| `RotatingFileHandler(..., maxBytes, backupCount)` | 要落文件，且文件不能无限变大 |
| `logging.Formatter(...)` | 手写 handler 时必须自己配格式（`basicConfig` 是替你做了这一步） |
| `Path(settings.LOG_DIR).mkdir(...)` | 目录不存在会报错，先建 |
| `setLevel` 分别设在 root 和第三方库上 | 全局级别 + 给 httpx 降噪 |

> **看懂的关键**：`basicConfig` 是个“打包好的快捷方式”。项目版只是把它拆开，手动做同样的事，再加两个额外功能。**不是两套知识，是一套知识的展开版。**

### 模板 2：`config.py`（7 行核心）


from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")  # 告诉它去哪读

    APP_NAME: str = "InsightAgent"   # 字段名 = 环境变量名，类型 = 校验规则
    DEBUG: bool = False

settings = Settings()   # 模块级单例，全项目 import 它


**只有 3 个东西要记：**

- `BaseSettings` —— 一个“会自己从环境变量读值”的类，继承它就行。
- `SettingsConfigDict(env_file=".env")` —— 指定读哪个文件。写在 `model_config` 里是 Pydantic v2 的固定写法。
- `settings = Settings()` —— 实例化一次。

`Field(...)` 和 `@lru_cache` 都是**可选优化**：前者做更细的校验（比如限定取值范围），后者保证只实例化一次。**没有它们代码照样跑。**

### 模板 3：`exceptions.py`（核心 12 行）


from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

class AppError(Exception):                    # ① 自定义异常：继承内置 Exception 就行
    status_code = 400
    code = "BAD_REQUEST"
    def __init__(self, message: str = "请求错误"):
        self.message = message

async def app_error_handler(request: Request, exc: AppError):   # ② 处理函数
    return JSONResponse(                                        # ③ 统一返回 JSON
        status_code=exc.status_code,
        content={"code": exc.code, "message": exc.message},
    )

def register(app: FastAPI) -> None:                            # ④ 注册到 app
    app.add_exception_handler(AppError, app_error_handler)


**四个角色，一句话一个：**

1. **自定义异常** —— 继承 `Exception`，加几个类属性。这一步和 FastAPI 无关，是纯 Python。
2. **处理函数** —— 固定签名 `(request, exc)`，返回值必须是 `Response`。
3. **`JSONResponse`** —— 把 dict 变成 HTTP 响应。
4. **`add_exception_handler(异常类, 处理函数)`** —— 把两者配对。这就是全部机制。

`RequestValidationError` 那些是“额外接入框架自带的异常”，属于锦上添花，不影响你写出能用的版本。

### 模板 4：`router.py`（核心 8 行）


from fastapi import APIRouter
from pydantic import BaseModel

router = APIRouter()                 # ① 建路由分组

class ItemOut(BaseModel):            # ② 出参模型（就是普通 Pydantic 模型）
    id: int

@router.get("/items/{item_id}", response_model=ItemOut)   # ③ 装饰器声明接口
async def get_item(item_id: int):    # ④ 函数参数 = 入参，自动校验类型
    return ItemOut(id=item_id)


**只有 2 个 import：** `APIRouter`（分组）和 `BaseModel`（数据模型）。`@router.get` 是方法调用，不是 import。

### `main.py` 骨架（5 行）


from fastapi import FastAPI
from app.api.v1.health import router as health_router

app = FastAPI()
app.include_router(health_router, prefix="/api/v1")


**就这两句是核心。** 中间件、lifespan、异常注册都是往上加，不加也能启动。

---

## 三、速查表：我要做 X，就导 Y

把这张表贴在编辑器旁边，写的时候照着找，写 3 次就不用看了。

| 我要做的事 | 导入 | 关键调用 |
|---|---|---|
| 打日志 | `import logging` | `logging.getLogger(__name__)` |
| 输出到控制台 | `import sys` + `logging` | `logging.StreamHandler(sys.stdout)` |
| 日志写文件并切分 | `from logging.handlers import RotatingFileHandler` | `RotatingFileHandler(path, maxBytes=, backupCount=)` |
| 定日志格式 | `logging` | `logging.Formatter(fmt, datefmt=)` |
| 给日志器加输出目标 | `logging` | `logger.addHandler(h)` / `logger.setLevel()` |
| 从 .env 读配置 | `from pydantic_settings import BaseSettings, SettingsConfigDict` | 继承 `BaseSettings`，写 `model_config` |
| 配置项做精细校验 | `from pydantic import Field` | `PORT: int = Field(default=8000, ge=1, le=65535)` |
| 配置只实例化一次 | `from functools import lru_cache` | `@lru_cache` 加在 `get_settings()` 上 |
| 定义请求/响应数据结构 | `from pydantic import BaseModel` | `class Xxx(BaseModel):` |
| 建路由 | `from fastapi import APIRouter` | `router = APIRouter()` / `@router.get(...)` |
| 拿请求对象（取 header、state） | `from fastapi import Request` | `request.headers.get(...)` / `request.state.xxx` |
| 返回 JSON | `from fastapi.responses import JSONResponse` | `JSONResponse(status_code=, content=)` |
| 返回状态码常量 | `from fastapi import status` | `status.HTTP_404_NOT_FOUND` |
| 抛 HTTP 错误（简单场景） | `from fastapi import HTTPException` | `raise HTTPException(404, "不存在")` |
| 注册全局异常处理 | `fastapi` | `app.add_exception_handler(异常类, 函数)` |
| 跨域 | `from fastapi.middleware.cors import CORSMiddleware` | `app.add_middleware(CORSMiddleware, ...)` |
| 写中间件 | `from fastapi import Request` | `@app.middleware("http")` + `call_next` |
| 启动/关闭钩子 | `from contextlib import asynccontextmanager` | `@asynccontextmanager` + `yield` |
| 生成请求 ID | `import uuid` | `uuid.uuid4().hex` |
| 测耗时 | `import time` | `time.perf_counter()` |
| 处理路径 | `from pathlib import Path` | `Path("logs") / "app.log"`、`.mkdir(parents=True, exist_ok=True)` |
| 写测试 | `from fastapi.testclient import TestClient` | `with TestClient(app) as c: c.get(...)` |

---

## 四、为什么你会有“完全看不懂”的感觉（这是正常的）

三个真实原因，不是你的问题：

1. **你看到的是“成品”，不是“推导过程”。** 我直接给了最终文件，中间的“我需要什么 → 查什么”被省略了，所以看起来像凭空出现的符号。上面四个模板就是把这一步补回来。
2. **import 数量多 ≠ 难度高。** `main.py` 导了 8 个东西，但每一个只服务一个小职责。把它拆成“日志 / 中间件 / 路由 / 异常”四块看，每块都只有 2~3 个 import。
3. **没人背 import。** 真实工作流是：知道要什么 → IDE 补全 → 查文档确认。资深工程师也这么干。**你要背的是结构，不是符号表。**

---

## 五、给你的 3 天默写训练（这是真正能“记住”的方法）

**Day 1**：关掉参考，默写 `logger.py` 的 8 行版。写不出来就看一眼，再默写。目标是 3 分钟内写完。

**Day 2**：默写 `config.py` 7 行 + `router.py` 8 行。写完对照，标出“我漏了哪个 import”。

**Day 3**：默写 `main.py` 最小 5 行版，然后**逐条加**：加 CORS、加中间件、加异常注册。每加一条问自己“这一步需要新 import 什么？为什么？”

**训练的核心不是背对，而是每次漏掉后能回答：“我漏的这个东西，是为了完成哪件事？”** 能答上来，就说明你掌握了推导法，下次换个需求也能推出来。

---

## 六、补充：同一个需求，写法可以不止一种

这点必须说清，否则你会以为“必须这么写”：

| 需求 | 写法 A（简单） | 写法 B（项目版） |
|---|---|---|
| 配日志 | `logging.basicConfig(...)` | 手写 `Handler + Formatter + addHandler` |
| 抛错误 | `raise HTTPException(404, "xx")` | `raise NotFoundError("xx")` + 全局 handler |
| 读配置 | `os.getenv("PORT", 8000)` | `BaseSettings` + `.env` |
| 配置单例 | 模块级 `settings = Settings()` | `@lru_cache` 包一层 |

**学习路径：先用 A 版跑通，知道 B 版在解决 A 版的什么缺陷，再换 B 版。** 你现在的问题正是“直接跳到 B 版”，所以补上 A 版这一环就通了。

---

要不要我用这个方式，把阶段一的 5 个文件重新出一版 **“A 版极简（每个 10 行内）+ B 版对照说明”**？你可以先跑 A 版建立信心，再逐块升级到 B 版——这样“看懂”和“记住”会同时发生。
```

**user**: 

```
需要，记住我是个初学者，相当于只会py里的最基础方法（知道def是方法创建，条件判断是if，class是类创建，等等），其他什么业务涉及的公共类方法或三方类方法的使用完全不知，需要按书本教学的方式教会我
```

**assistant**: 

```
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
```



--------------------


<a href="https://chatboxai.app" style="display: flex; align-items: center;">
<img src='https://chatboxai.app/icon.png' style='width: 40px; height: 40px; padding-right: 6px'>
<b style='font-size:30px'>Chatbox AI</b>
</a>
