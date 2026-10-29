# 交付物 A：需求文档  
  
## 0. 术语与边界  
  
本文件使用四个严格区分的词，全文不得混用：  
  
| 术语 | 含义 |  
|---|---|  
| 骨架段 | 单章内不超过 15 行的最小演示，只为跑通一个概念 |  
| 本章成品 | 单章内的最终形态，只在该章这一跳范围内可用 |  
| 可运行项目 | 全书合并后端到端能跑：启动服务 → 提问 → 检索 → 作答 → 返回 |  
| 生产级 | 含鉴权、限流、监控告警、迁移、备份、多副本；本次不做 |  
  
本次目标：**可运行项目**。共 40 个步骤，每步对应代码不超过 30 行，全项目代码控制在 1500 行以内。  
  
技术栈固定：Python 3.11 / FastAPI / SQLAlchemy / SQLite / httpx / pydantic / pytest。  
  
---  
  
## 1. 项目一句话定位  
  
InsightAgent 是一个把本地资料文件切片入库、再让模型按关键词检索后作答的最小全栈服务。  
  
---  
  
## 2. 最终目录树  
  
```  
insight-agent/  
├── app/  
│   ├── __init__.py                  # 标记 app 为包  
│   ├── config.py                    # 读 .env，暴露 settings  
│   ├── main.py                      # 创建应用、挂路由、注册异常处理器  
│   ├── core/  
│   │   ├── __init__.py              # 标记包  
│   │   ├── logging.py               # 日志初始化与 get_logger  
│   │   ├── exceptions.py            # 业务异常与全局处理器  
│   │   └── retry.py                 # 通用重试装饰器  
│   ├── db/  
│   │   ├── __init__.py              # 标记包  
│   │   ├── base.py                  # 引擎与模型基类  
│   │   ├── init_db.py               # 建表函数  
│   │   └── session.py               # 会话工厂与 get_db  
│   ├── models/  
│   │   ├── __init__.py              # 集中导入两张表  
│   │   ├── document.py              # documents 表  
│   │   └── chunk.py                 # chunks 表  
│   ├── repositories/  
│   │   ├── __init__.py              # 标记包  
│   │   ├── document_repo.py         # 文档增删查  
│   │   └── chunk_repo.py            # 分块检索与计数  
│   ├── services/  
│   │   ├── __init__.py              # 标记包  
│   │   ├── loader.py                # 资料文件读成纯文本  
│   │   ├── cleaner.py               # 去标签、压空白、压空行  
│   │   ├── splitter.py              # 按句切分并保留重叠  
│   │   ├── ingest.py                # 先删后插批量入库  
│   │   └── search_service.py        # 分页检索与总数组装  
│   ├── agent/  
│   │   ├── __init__.py              # 标记包并导出协议模型  
│   │   ├── schemas.py               # 消息、工具、决策模型  
│   │   ├── llm_client.py            # 模型 HTTP 客户端  
│   │   ├── base.py                  # 工具抽象基类  
│   │   ├── registry.py              # 工具注册表  
│   │   ├── prompts.py               # 系统提示词  
│   │   ├── executor.py              # ReAct 循环与 stream  
│   │   └── tools/  
│   │       ├── __init__.py          # 标记包  
│   │       └── kb_search.py         # 知识库检索工具  
│   └── api/  
│       ├── __init__.py              # 标记包  
│       └── v1/  
│           ├── __init__.py          # 标记包  
│           ├── schemas.py           # 聊天请求与响应模型  
│           ├── health.py            # 健康检查路由  
│           └── chat.py              # 聊天与流式路由  
├── tests/  
│   ├── __init__.py                  # 标记包  
│   ├── conftest.py                  # 内存库、假执行器、测试客户端  
│   ├── test_pipeline.py             # 读→洗→切→存→查  
│   └── test_api.py                  # 接口与流式  
├── scripts/  
│   ├── __init__.py                  # 标记包  
│   ├── import_docs.py               # 命令行导入资料文件  
│   └── ask.py                       # 命令行问一句  
├── data/  
│   └── .gitkeep                     # 数据库与示例资料目录  
├── requirements.txt                 # 锁定版本依赖  
└── .env.example                     # 环境变量清单  
```  
  
---  
  
## 3. 离线链：建库（O-01 ～ O-17）  
  
### O-01 建立目录与依赖清单  
  
- **做什么**：创建目录结构并写入锁定版本的依赖清单。  
- **为什么**：不装依赖直接导入会报 `ModuleNotFoundError: No module named 'fastapi'`。  
- **输入**：无。  
- **输出**：包目录与 `requirements.txt`，供后续所有步骤使用。  
- **验收**：`pip install -r requirements.txt && python -c "import fastapi, sqlalchemy, httpx, pydantic"` 退出码 0。  
- **文件/知识点**：`requirements.txt`｜第一册 第1章（环境搭建）  
  
### O-02 声明环境变量  
  
- **做什么**：写 `.env.example` 并复制为 `.env`。  
- **为什么**：缺 `LLM_API_KEY` 时首个请求返回 `401 Unauthorized`，且不知道少哪一项。  
- **输入**：无。  
- **输出**：6 个变量名与本地 `.env`。  
- **验收**：`grep -c '=' .env.example` 输出 `6`。  
- **文件/知识点**：`.env.example`｜第一册 第2章（配置）  
  
### O-03 读取配置  
  
- **做什么**：用 `BaseSettings` 从 `.env` 读出配置单例。  
- **为什么**：端口与数据库地址写死在代码里，换环境就要改代码。  
- **输入**：`.env`。  
- **输出**：`settings`，供全项目导入。  
- **验收**：`python -c "from app.config import settings; print(settings.PORT)"` 输出 `8000`。  
- **文件/知识点**：`app/config.py`｜第一册 第2章（配置）  
  
### O-04 初始化日志  
  
- **做什么**：配置控制台与轮转文件双输出。  
- **为什么**：没有日志时接口 500 只剩框架默认堆栈，定位不到是哪个文件。  
- **输入**：`settings.LOG_LEVEL`、`settings.LOG_DIR`。  
- **输出**：全局 logger 与 `logs/app.log`。  
- **验收**：`python -c "from app.core.logging import setup_logging,get_logger; setup_logging(); get_logger('demo').info('ok')"`，输出含 `ok`。  
- **文件/知识点**：`app/core/logging.py`｜第一册 第1章（日志）  
  
### O-05 定义业务异常  
  
- **做什么**：定义 `AppError` 及 404／400／502 三个子类。  
- **为什么**：不分类时所有失败都返回 500，客户端无法判断该改参数还是该重试。  
- **输入**：异常消息。  
- **输出**：带 `status_code`、`code`、`message` 的异常类型。  
- **验收**：`python -c "from app.core.exceptions import UpstreamError; e=UpstreamError(); print(e.status_code, e.code)"` 输出 `502 UPSTREAM_ERROR`。  
- **文件/知识点**：`app/core/exceptions.py`｜第一册 第3章、第五册 第2章  
  
### O-06 创建引擎与模型基类  
  
- **做什么**：建 SQLite 引擎并定义 `Base`。  
- **为什么**：每个模型文件各自建引擎，会重复开连接池，并发时 `sqlite3.OperationalError: database is locked`。  
- **输入**：`settings.DATABASE_URL`。  
- **输出**：`engine`、`Base`。  
- **验收**：`python -c "from app.db.base import engine; print(engine.url)"` 输出 `sqlite:///./data/insight.db`。  
- **文件/知识点**：`app/db/base.py`｜第三册 第1章  
  
### O-07 定义文档表  
  
- **做什么**：定义 `documents` 表模型。  
- **为什么**：没有这张表，`chunks.document_id` 没有指向，删文档后分块成孤儿。  
- **输入**：`Base`。  
- **输出**：`Document` 类。  
- **验收**：`python -c "from app.models.document import Document; print(Document.__tablename__)"` 输出 `documents`。  
- **文件/知识点**：`app/models/document.py`｜第三册 第1章  
  
### O-08 定义分块表  
  
- **做什么**：定义 `chunks` 表、外键、索引与唯一约束。  
- **为什么**：缺唯一约束时重复导入一次，条数从 500 变 1000，检索结果重复。  
- **输入**：`Base`、`Document`。  
- **输出**：`Chunk` 类。  
- **验收**：`python -c "from app.models.chunk import Chunk; print([c.name for c in Chunk.__table__.constraints])"` 含 `uq_chunk_doc_index`。  
- **文件/知识点**：`app/models/chunk.py`｜第三册 第1章、第3章  
  
### O-09 集中注册模型并建表  
  
- **做什么**：在包入口导入两张表，并提供 `init_db()`。  
- **为什么**：不导入模型，`create_all` 只建已加载的表，运行时 `no such table: chunks`。  
- **输入**：两个模型类。  
- **输出**：`init_db()` 与两张已建好的表。  
- **验收**：`python -c "from app.db.init_db import init_db; init_db(); import sqlite3; print(sorted(r[0] for r in sqlite3.connect('data/insight.db').execute(\"select name from sqlite_master where type='table'\")))"` 含 `chunks`、`documents`。  
- **文件/知识点**：`app/models/__init__.py`、`app/db/init_db.py`｜第三册 第1章  
  
### O-10 读取资料文件  
  
- **做什么**：把资料文件读成字符串，并包装读取失败。  
- **为什么**：直接读不存在或编码不符的文件，会抛 `FileNotFoundError` 或 `UnicodeDecodeError`，接口层无从处理。  
- **输入**：资料文件路径。  
- **输出**：纯文本或 `FileParseError`。  
- **验收**：`printf '服务监听 8000。\n' > data/sample.txt && python -c "from app.services.loader import load_text; print(load_text('data/sample.txt').strip())"` 输出该句。  
- **文件/知识点**：`app/services/loader.py`｜第二册 第1章、第2章  
  
### O-11 清洗文本  
  
- **做什么**：去 HTML 标签、压连续空白、压多余空行。  
- **为什么**：不清洗时切片里混入 `<p>` 等标签，检索命中大量噪声。  
- **输入**：纯文本。  
- **输出**：干净文本。  
- **验收**：`python -c "from app.services.cleaner import clean_text; print(clean_text('<p>a</p>'))"` 输出 `a`。  
- **文件/知识点**：`app/services/cleaner.py`｜第二册 第3章、第4章  
  
### O-12 切分文本  
  
- **做什么**：按句末标点切句，再按长度装箱并保留重叠。  
- **为什么**：定长硬切会把一句话切成两半，检索到的块缺主语。  
- **输入**：干净文本。  
- **输出**：`list[dict]`，每项含 `index`、`text`、`char_count`。  
- **验收**：`python -c "from app.services.splitter import split_text; c=split_text('一句话。'*10,size=20,overlap=5); print(len(c), sorted(c[0]))"` 输出块数与三个键。  
- **文件/知识点**：`app/services/splitter.py`｜第二册 第5章  
  
### O-13 创建会话工厂  
  
- **做什么**：建 `SessionLocal` 与每请求一个会话的 `get_db`。  
- **为什么**：会话创建散在各处且忘记关闭，50 并发后报 `database is locked`。  
- **输入**：`engine`。  
- **输出**：`SessionLocal`、`get_db`。  
- **验收**：`python -c "from app.db.session import SessionLocal; s=SessionLocal(); print(s.is_active); s.close()"` 输出 `True`。  
- **文件/知识点**：`app/db/session.py`｜第三册 第2章  
  
### O-14 文档仓储  
  
- **做什么**：实现新增、按主键查、列表、删除四个函数。  
- **为什么**：路由里直接写会话，8 个接口会变成 48 行重复样板。  
- **输入**：`Session`、文档字段。  
- **输出**：`Document` 实例或布尔值。  
- **验收**：`python -c "from app.db.init_db import init_db; init_db(); from app.db.session import SessionLocal; from app.repositories.document_repo import create_document; s=SessionLocal(); print(create_document(s,'a.txt','data/a.txt').id); s.close()"` 输出 `1`。  
- **文件/知识点**：`app/repositories/document_repo.py`｜第三册 第2章  
  
### O-15 批量入库  
  
- **做什么**：先删该文档旧块，再一次性批量插入新块。  
- **为什么**：逐条提交 500 块约 2.3 秒；重复导入不先删，条数会翻倍。  
- **输入**：`Session`、`document_id`、`list[dict]`。  
- **输出**：写入块数。  
- **验收**：连续调用两次后 `python -c "...print(len(...))"` 计数仍为 `3`（完整命令见 B 阶段脚本）。  
- **文件/知识点**：`app/services/ingest.py`｜第三册 第3章  
  
### O-16 导入脚本  
  
- **做什么**：提供 `python -m scripts.import_docs <文件>` 一条命令完成读、洗、切、入库。  
- **为什么**：没有入口，资料进不了库，在线链永远查不到东西。  
- **输入**：资料文件路径。  
- **输出**：`documents` 1 行、`chunks` N 行。  
- **验收**：`python -m scripts.import_docs data/sample.txt` 输出 `imported 1 document, N chunks` 且 N ≥ 1。  
- **文件/知识点**：`scripts/import_docs.py`｜第二册 第7章、第三册 第3章  
  
### O-17 离线链验收  
  
- **做什么**：核对库中行数与切片内容。  
- **为什么**：不核对时可能出现「脚本成功但入库 0 条」，后面问答永远为空。  
- **输入**：`data/insight.db`。  
- **输出**：两张表的行数与一条切片。  
- **验收**：`sqlite3 data/insight.db "select count(*) from documents; select count(*) from chunks;"` 输出两行，第二行 ≥ 1。  
- **文件/知识点**：无新增文件｜第三册 第1章  
  
---  
  
## 4. 在线链：问答（Q-01 ～ Q-11）  
  
### Q-01 定义 Agent 协议模型  
  
- **做什么**：定义 `ChatMessage`、`ToolSpec`、`AgentDecision`。  
- **为什么**：裸字典传消息时字段拼错只在运行时 `KeyError`；`action` 写成 `"use_tool"` 也能通过。  
- **输入**：模型返回的 JSON、工具描述。  
- **输出**：三个可校验的 Pydantic 模型。  
- **验收**：`python -c "from app.agent.schemas import AgentDecision; print(AgentDecision.model_validate_json('{\"action\":\"final\",\"answer\":\"ok\"}').answer)"` 输出 `ok`。  
- **文件/知识点**：`app/agent/schemas.py`、`app/agent/__init__.py`｜第四册 第1章  
  
### Q-02 重试装饰器  
  
- **做什么**：写一个按次数与退避重试的装饰器。  
- **为什么**：模型接口偶发 429，不重试时一次抖动就让用户看到 500。  
- **输入**：被装饰函数、次数、退避秒数。  
- **输出**：包装后的函数。  
- **验收**：`python -c "from app.core.retry import retry; f=retry(times=2)(lambda: 1); print(f())"` 输出 `1`。  
- **文件/知识点**：`app/core/retry.py`｜第四册 第2章  
  
### Q-03 模型客户端  
  
- **做什么**：用 httpx 发 `chat/completions` 请求，带超时、状态码检查与异常包装。  
- **为什么**：不设超时会挂到线程耗尽；不检查状态码时 401 被读成 `KeyError: 'choices'`。  
- **输入**：`list[ChatMessage]`。  
- **输出**：模型文本或 `UpstreamError`。  
- **验收**：`python -c "from app.agent.llm_client import LLMClient; print(LLMClient().timeout)"` 输出 `30.0`。  
- **文件/知识点**：`app/agent/llm_client.py`｜第四册 第2章  
  
### Q-04 工具基类  
  
- **做什么**：定义 `BaseTool`，包含名称、描述、参数与 `run`。  
- **为什么**：没有统一接口，每加一个工具都要改调度分支。  
- **输入**：无。  
- **输出**：抽象基类。  
- **验收**：`python -c "from app.agent.base import BaseTool; T=type('T',(BaseTool,),{'name':'t','description':'d','parameters':{},'run':lambda self,**k:'ok'}); print(T().run())"` 输出 `ok`。  
- **文件/知识点**：`app/agent/base.py`｜第四册 第3章  
  
### Q-05 工具注册表  
  
- **做什么**：用字典保存工具，按名调用，未知工具返回可读文本。  
- **为什么**：直接 `TOOLS[name]` 时名字写错抛 `KeyError`，整轮对话中断。  
- **输入**：工具实例、名字、参数。  
- **输出**：工具返回值或错误文本。  
- **验收**：`python -c "from app.agent.registry import ToolRegistry; print(ToolRegistry().run('x',{}))"` 输出 `未知工具: x`。  
- **文件/知识点**：`app/agent/registry.py`｜第四册 第3章  
  
### Q-06 分块检索仓储  
  
- **做什么**：实现按关键词检索与命中计数两条查询。  
- **为什么**：10 万行上 `LIKE '%部署%'` 无索引可用，单次约 1.2 秒。  
- **输入**：`Session`、关键词、`limit`、`offset`。  
- **输出**：`Chunk` 列表与总数。  
- **验收**：`python -c "from app.db.session import SessionLocal; from app.repositories.chunk_repo import count_chunks; s=SessionLocal(); print(count_chunks(s,'8000')); s.close()"` 输出 ≥ 1。  
- **文件/知识点**：`app/repositories/chunk_repo.py`｜第三册 第4章  
  
### Q-07 检索服务  
  
- **做什么**：把页码换算成 `offset`，返回当前页数据与总数。  
- **为什么**：不分页时一次返回 10 万条约 40MB，前端直接卡死。  
- **输入**：`Session`、关键词、页码、每页条数。  
- **输出**：`{items, total, page, page_size}`。  
- **验收**：`python -c "from app.db.session import SessionLocal; from app.services.search_service import search_page; s=SessionLocal(); print(search_page(s,'8000')['total']); s.close()"` 输出 ≥ 1。  
- **文件/知识点**：`app/services/search_service.py`｜第三册 第4章  
  
### Q-08 知识库检索工具  
  
- **做什么**：把 `search_page` 包装成 `kb_search` 工具。  
- **为什么**：模型看不到 `chunks` 表，没有工具时只能凭训练知识答「通常 80 或 443」，与本项目 8000 不符。  
- **输入**：关键词、`Session`。  
- **输出**：拼接后的切片文本。  
- **验收**：`python -c "from app.db.session import SessionLocal; from app.agent.tools.kb_search import KBSearchTool; s=SessionLocal(); print(KBSearchTool(s).run('8000')[:20]); s.close()"` 输出含 `8000` 的片段。  
- **文件/知识点**：`app/agent/tools/kb_search.py`、`app/agent/tools/__init__.py`｜第四册 第3章  
  
### Q-09 系统提示词  
  
- **做什么**：把工具列表与输出格式写进系统消息。  
- **为什么**：不限定输出格式时模型回一段散文，`model_validate_json` 直接抛 `ValidationError`。  
- **输入**：`list[ToolSpec]`。  
- **输出**：系统提示词字符串。  
- **验收**：`python -c "from app.agent.prompts import build_system_prompt; print('action' in build_system_prompt([]))"` 输出 `True`。  
- **文件/知识点**：`app/agent/prompts.py`｜第四册 第4章  
  
### Q-10 ReAct 执行器  
  
- **做什么**：组织消息、解析决策、调工具、回填观察，最多 5 轮。  
- **为什么**：不设最大步数会无限调用模型；工具异常不兜底会让整轮 500。  
- **输入**：用户问题、客户端、注册表。  
- **输出**：最终答案。  
- **验收**：用假客户端跑一次，`python -c "...print(AgentExecutor(Fake(), ToolRegistry()).run('hi'))"` 输出 `ok`（完整命令见 B 阶段）。  
- **文件/知识点**：`app/agent/executor.py`｜第四册 第4章  
  
### Q-11 命令行问答脚本  
  
- **做什么**：提供 `python -m scripts.ask "<问题>"` 直接调用执行器。  
- **为什么**：没有脚本时只能进交互式 Python 拼三行导入，无法快速验证在线链。  
- **输入**：问题字符串。  
- **输出**：终端打印答案。  
- **验收**：`python -m scripts.ask "这个项目部署时用哪个端口？"` 输出含 `8000`。  
- **文件/知识点**：`scripts/ask.py`｜第四册 第4章、第5章  
  
---  
  
## 5. 交付：接口与测试（D-01 ～ D-12）  
  
### D-01 健康检查路由  
  
- **做什么**：提供 `/api/v1/health`，返回服务名与环境。  
- **为什么**：没有探活接口，容器编排无法判断服务是活着还是卡死。  
- **输入**：无。  
- **输出**：JSON 状态。  
- **验收**：`python -c "from app.api.v1.health import router; print([r.path for r in router.routes])"` 输出 `['/health']`。  
- **文件/知识点**：`app/api/v1/health.py`｜第一册 第4章、第五册 第1章  
  
### D-02 聊天接口模型  
  
- **做什么**：定义 `ChatRequest` 与 `ChatResponse` 并加约束。  
- **为什么**：不限制长度时空字符串也会进模型，浪费一次调用。  
- **输入**：请求 JSON。  
- **输出**：校验后的对象。  
- **验收**：`python -c "from app.api.v1.schemas import ChatRequest; print(ChatRequest(question='x').top_k)"` 输出 `5`。  
- **文件/知识点**：`app/api/v1/schemas.py`｜第五册 第1章  
  
### D-03 聊天路由与依赖注入  
  
- **做什么**：建 `POST /chat`，用 `Depends` 注入会话与执行器。  
- **为什么**：没有路由时只能跑脚本，浏览器访问返回 `404 Not Found`。  
- **输入**：`ChatRequest`。  
- **输出**：`ChatResponse`。  
- **验收**：`python -c "from app.api.v1.chat import router; print([r.path for r in router.routes])"` 输出 `['/chat']`。  
- **文件/知识点**：`app/api/v1/chat.py`｜第五册 第1章  
  
### D-04 执行器流式方法  
  
- **做什么**：给 `AgentExecutor` 增加 `stream()`，按轮次产出状态、工具与最终事件。  
- **为什么**：一次性返回时用户空白等 8.4 秒，会刷新导致重复检索与重复计费。  
- **输入**：问题。  
- **输出**：事件迭代器。  
- **验收**：用假客户端收集事件，`python -c "...print([e['type'] for e in ex.stream('x')])"` 输出含 `status`、`final`（完整命令见 B 阶段）。  
- **文件/知识点**：`app/agent/executor.py`｜第五册 第3章  
  
### D-05 流式路由  
  
- **做什么**：增加 `POST /chat/stream`，把事件编码成 SSE。  
- **为什么**：前端需要边收边显示；没有结束标记时无法判断流是否正常结束。  
- **输入**：`ChatRequest`。  
- **输出**：`text/event-stream` 响应。  
- **验收**：`python -c "from app.api.v1.chat import router; print('/chat/stream' in [r.path for r in router.routes])"` 输出 `True`。  
- **文件/知识点**：`app/api/v1/chat.py`｜第五册 第3章  
  
### D-06 应用入口与路由挂载  
  
- **做什么**：创建 FastAPI 应用，挂 health 与 chat 两个路由。  
- **为什么**：路由写了但没 `include_router`，访问一律 `404 Not Found`。  
- **输入**：两个 router。  
- **输出**：可启动的 `app`。  
- **验收**：`python -c "from fastapi.testclient import TestClient; from app.main import app; print(TestClient(app).get('/api/v1/health').status_code)"` 输出 `200`。  
- **文件/知识点**：`app/main.py`｜第五册 第1章  
  
### D-07 注册全局异常处理器  
  
- **做什么**：把 `AppError`、校验错误、未知错误三类处理器挂到应用。  
- **为什么**：不注册时错误返回框架默认的 `{"detail":"Internal Server Error"}`，前端拿不到 `code`。  
- **输入**：应用实例。  
- **输出**：统一的 `{code, message}` 错误响应。  
- **验收**：`python -c "from fastapi.testclient import TestClient; from app.main import app; r=TestClient(app).post('/api/v1/chat',json={'question':''}); print(r.status_code, r.json()['code'])"` 输出 `422 VALIDATION_ERROR`。  
- **文件/知识点**：`app/main.py`、`app/core/exceptions.py`｜第五册 第2章  
  
### D-08 测试夹具  
  
- **做什么**：提供 `db_session`、`client` 与假执行器。  
- **为什么**：测试直连开发库会清空真实数据；没有替身时 CI 无密钥报 401。  
- **输入**：应用与内存库。  
- **输出**：三个 pytest fixture。  
- **验收**：`pytest --collect-only -q` 不报导入错误。  
- **文件/知识点**：`tests/conftest.py`｜第三册 第5章、第五册 第4章  
  
### D-09 流水线测试  
  
- **做什么**：测「切分→入库→检索」与重复导入幂等。  
- **为什么**：改一次分块参数就可能让检索变空，没有测试无法发现。  
- **输入**：内存库会话。  
- **输出**：通过或失败的断言。  
- **验收**：`pytest tests/test_pipeline.py -q` 输出全部通过。  
- **文件/知识点**：`tests/test_pipeline.py`｜第二册 第8章、第三册 第5章  
  
### D-10 接口测试  
  
- **做什么**：测 200、422、502 三条路径。  
- **为什么**：只测成功路径时，错误映射被改坏也没人发现。  
- **输入**：`TestClient` 与假执行器。  
- **输出**：状态码与 `code` 断言。  
- **验收**：`pytest tests/test_api.py -q` 输出全部通过。  
- **文件/知识点**：`tests/test_api.py`｜第五册 第4章  
  
### D-11 流式测试  
  
- **做什么**：测 SSE 的事件类型与结束标记。  
- **为什么**：流式格式被改坏时，前端只会一直转圈，没有测试就查不出。  
- **输入**：流式客户端。  
- **输出**：事件文本断言。  
- **验收**：`pytest tests/test_api.py -q -k stream` 输出通过。  
- **文件/知识点**：`tests/test_api.py`｜第五册 第4章  
  
### D-12 总验收  
  
- **做什么**：启动服务，用一条 curl 完成端到端验证。  
- **为什么**：所有单元测试都过，也不代表「启动→提问→检索→作答→返回」这条链真的通。  
- **输入**：已导入 `data/sample.txt` 的数据库、有效 `.env`。  
- **输出**：含答案与耗时的 JSON。  
- **验收**：见下方总验收命令。  
- **文件/知识点**：无新增文件｜第五册 第1～4章  
  
---  
  
## 6. 总验收  
  
启动：  
  
```bash  
uvicorn app.main:app --host 0.0.0.0 --port 8000  
```  
  
请求：  
  
```bash  
curl -s -X POST http://127.0.0.1:8000/api/v1/chat \  
  -H "Content-Type: application/json" \  
  -d '{"question":"这个项目部署时用哪个端口？"}'  
```  
  
期望返回（字段顺序可能不同）：  
  
```json  
{"answer":"应用服务监听 8000……依据：切片 #1「服务监听 8000」。","elapsed_ms":1234}  
```  
  
判定标准：HTTP 200；`answer` 中含 `8000`；`elapsed_ms` 为大于 0 的整数。  
  
---  
  
以上为交付物 A。确认后我再按「离线链 → 在线链 → 交付」分批输出交付物 B。  
