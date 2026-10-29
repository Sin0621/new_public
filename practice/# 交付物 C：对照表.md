# 交付物 C：对照表  
  
| 文件路径 | 实现步骤号 | 依赖哪些文件 | 被谁调用 |  
|---|---|---|---|  
| `requirements.txt` | O-01 | 无 | `pip install`、CI |  
| `.env.example` | O-02 | 无 | `app/config.py` |  
| `app/__init__.py` | O-01 | 无 | 所有 `from app.xxx` |  
| `app/core/__init__.py` | O-01 | 无 | 包标记 |  
| `app/db/__init__.py` | O-01 | 无 | 包标记 |  
| `app/services/__init__.py` | O-01 | 无 | 包标记 |  
| `app/repositories/__init__.py` | O-01 | 无 | 包标记 |  
| `scripts/__init__.py` | O-01 | 无 | `python -m scripts.*` |  
| `data/.gitkeep` | O-01 | 无 | 目录占位 |  
| `app/config.py` | O-03 | `.env`、`pydantic-settings` | `logging.py`、`db/base.py`、`llm_client.py` |  
| `app/core/logging.py` | O-04 | `config.py` | 各模块、`main.py` |  
| `app/core/exceptions.py` | O-05、D-07 | `fastapi`、`logging.py` | `loader.py`、`ingest.py`、`llm_client.py`、`main.py` |  
| `app/core/retry.py` | Q-02 | `logging.py` | `llm_client.py` |  
| `app/db/base.py` | O-06 | `config.py`、`SQLAlchemy` | `models/*`、`init_db.py`、`session.py` |  
| `app/models/document.py` | O-07 | `db/base.py` | `models/__init__.py`、`document_repo.py` |  
| `app/models/chunk.py` | O-08 | `db/base.py` | `models/__init__.py`、`ingest.py`、`chunk_repo.py` |  
| `app/models/__init__.py` | O-09 | `document.py`、`chunk.py` | `init_db.py` |  
| `app/db/init_db.py` | O-09 | `db/base.py`、`models/__init__.py` | `import_docs.py`、`ask.py`、`conftest.py` |  
| `app/services/loader.py` | O-10 | `exceptions.py`、`logging.py` | `import_docs.py` |  
| `app/services/cleaner.py` | O-11 | 无 | `import_docs.py`、`test_pipeline.py` |  
| `app/services/splitter.py` | O-12 | 无 | `import_docs.py`、`test_pipeline.py` |  
| `app/db/session.py` | O-13 | `db/base.py` | `chat.py`、`import_docs.py`、`ask.py`、`conftest.py` |  
| `app/repositories/document_repo.py` | O-14 | `models/document.py`、`Session` | `import_docs.py`、`chat.py`、`test_pipeline.py` |  
| `app/services/ingest.py` | O-15 | `models/chunk.py`、`exceptions.py`、`logging.py` | `import_docs.py`、`test_pipeline.py` |  
| `scripts/import_docs.py` | O-16 | `loader`、`cleaner`、`splitter`、`ingest`、`document_repo`、`session` | 命令行 |  
| `app/agent/__init__.py` | Q-01 | `agent/schemas.py` | `from app.agent` |  
| `app/agent/schemas.py` | Q-01 | `pydantic` | `llm_client.py`、`base.py`、`registry.py`、`executor.py` |  
| `app/agent/llm_client.py` | Q-03 | `schemas.py`、`config.py`、`exceptions.py`、`retry.py`、`httpx` | `ask.py`、`chat.py`、`conftest.py` |  
| `app/agent/base.py` | Q-04 | `schemas.py` | `registry.py`、`kb_search.py` |  
| `app/agent/registry.py` | Q-05 | `base.py`、`schemas.py` | `executor.py`、`ask.py`、`chat.py` |  
| `app/repositories/chunk_repo.py` | Q-06 | `models/chunk.py` | `search_service.py`、`test_pipeline.py` |  
| `app/services/search_service.py` | Q-07 | `chunk_repo.py` | `kb_search.py`、`test_pipeline.py` |  
| `app/agent/tools/__init__.py` | Q-08 | 无 | 包标记 |  
| `app/agent/tools/kb_search.py` | Q-08 | `base.py`、`search_service.py` | `ask.py`、`chat.py` |  
| `app/agent/prompts.py` | Q-09 | `schemas.py` | `executor.py` |  
| `app/agent/executor.py` | Q-10、D-04 | `llm_client.py`、`registry.py`、`schemas.py`、`prompts.py`、`exceptions.py` | `ask.py`、`chat.py`、`conftest.py` |  
| `scripts/ask.py` | Q-11 | `executor`、`llm_client`、`registry`、`kb_search`、`session`、`init_db` | 命令行 |  
| `app/api/__init__.py` | D-01 | 无 | 包标记 |  
| `app/api/v1/__init__.py` | D-01 | 无 | 包标记 |  
| `app/api/v1/health.py` | D-01 | `config.py` | `main.py` |  
| `app/api/v1/schemas.py` | D-02 | `pydantic` | `chat.py`、`test_api.py` |  
| `app/api/v1/chat.py` | D-03、D-05 | `executor`、`llm_client`、`registry`、`kb_search`、`api/schemas`、`session` | `main.py` |  
| `app/main.py` | D-06、D-07 | `api/v1/chat.py`、`api/v1/health.py`、`exceptions.py`、`logging.py` | `uvicorn`、`conftest.py` |  
| `tests/__init__.py` | D-08 | 无 | pytest |  
| `tests/conftest.py` | D-08 | `main.py`、`chat.get_executor`、`db/base.py` | `test_pipeline.py`、`test_api.py` |  
| `tests/test_pipeline.py` | D-09 | `cleaner`、`splitter`、`ingest`、`search_service`、`document_repo`、`models/chunk` | pytest |  
| `tests/test_api.py` | D-10、D-11 | `main.py`、`conftest.py`、`exceptions.py` | pytest |  
  
---  
  
# 交付物 D：跟敲路线  
  
共 7 天，40 个锚点。顺序按数据流，不按文件树。  
  
## 第 1 天：O-01 ～ O-06（6 个锚点）  
  
| 顺序 | 锚点 | 文件 | 备注 |  
|---|---|---|---|  
| 1 | O-01 | `requirements.txt`、各 `__init__.py`、`data/.gitkeep` | 先建目录再装依赖 |  
| 2 | O-02 | `.env.example` | 复制为 `.env` |  
| 3 | O-03 | `app/config.py` | |  
| 4 | O-04 | `app/core/logging.py` | |  
| 5 | O-05 | `app/core/exceptions.py` | **本文件只敲【O-05】那一段（异常类）；处理器在 Day 7 追加** |  
| 6 | O-06 | `app/db/base.py` | |  
  
**敲完能验证**：配置能读、日志能打、异常类能实例化、引擎指向正确的 SQLite 文件。  
  
```bash  
python -c "from app.db.base import engine; print(engine.url)"  
```  
  
期望：`sqlite:///./data/insight.db`  
  
---  
  
## 第 2 天：O-07 ～ O-12（6 个锚点）  
  
| 顺序 | 锚点 | 文件 |  
|---|---|---|  
| 1 | O-07 | `app/models/document.py` |  
| 2 | O-08 | `app/models/chunk.py` |  
| 3 | O-09 | `app/models/__init__.py`、`app/db/init_db.py` |  
| 4 | O-10 | `app/services/loader.py` |  
| 5 | O-11 | `app/services/cleaner.py` |  
| 6 | O-12 | `app/services/splitter.py` |  
  
**敲完能验证**：两张表建出来；资料文件能读、能洗、能切。  
  
```bash  
python -c "from app.db.init_db import init_db; init_db(); import sqlite3; print(sorted(r[0] for r in sqlite3.connect('data/insight.db').execute(\"select name from sqlite_master where type='table'\")))"  
```  
  
期望：`['chunks', 'documents']`  
  
---  
  
## 第 3 天：O-13 ～ O-17（5 个锚点）  
  
| 顺序 | 锚点 | 文件 |  
|---|---|---|  
| 1 | O-13 | `app/db/session.py` |  
| 2 | O-14 | `app/repositories/document_repo.py` |  
| 3 | O-15 | `app/services/ingest.py` |  
| 4 | O-16 | `scripts/import_docs.py` |  
| 5 | O-17 | 无新增文件，执行验收 |  
  
**敲完能验证**：一条命令把资料文件写入两张表。  
  
```bash  
printf '服务监听 8000，反向代理负责 80/443。\n' > data/sample.txt  
python -m scripts.import_docs data/sample.txt  
sqlite3 data/insight.db "select count(*) from documents; select count(*) from chunks;"  
```  
  
期望：`imported 1 document, 1 chunks`；随后输出 `1` 和 `1`。  
  
---  
  
## 第 4 天：Q-01 ～ Q-06（6 个锚点）  
  
| 顺序 | 锚点 | 文件 |  
|---|---|---|  
| 1 | Q-01 | `app/agent/__init__.py`、`app/agent/schemas.py` |  
| 2 | Q-02 | `app/core/retry.py` |  
| 3 | Q-03 | `app/agent/llm_client.py` |  
| 4 | Q-04 | `app/agent/base.py` |  
| 5 | Q-05 | `app/agent/registry.py` |  
| 6 | Q-06 | `app/repositories/chunk_repo.py` |  
  
**敲完能验证**：协议能解析、客户端能构造、工具能注册、检索查询能执行。  
  
```bash  
python -c "from app.agent.llm_client import LLMClient; print(LLMClient().timeout)"  
```  
  
期望：`30.0`  
  
---  
  
## 第 5 天：Q-07 ～ Q-11（5 个锚点）  
  
| 顺序 | 锚点 | 文件 | 备注 |  
|---|---|---|---|  
| 1 | Q-07 | `app/services/search_service.py` | |  
| 2 | Q-08 | `app/agent/tools/__init__.py`、`app/agent/tools/kb_search.py` | |  
| 3 | Q-09 | `app/agent/prompts.py` | |  
| 4 | Q-10 | `app/agent/executor.py` | **本文件只敲【Q-10】的 `run()` 与 `_initial_messages()`；`stream()` 在 Day 6 追加** |  
| 5 | Q-11 | `scripts/ask.py` | |  
  
**敲完能验证**：执行器有 `run`，在线链可在命令行跑通（需 `.env` 中有效密钥）。  
  
```bash  
python -c "from app.agent.executor import AgentExecutor; print(hasattr(AgentExecutor, 'run'))"  
```  
  
期望：`True`  
  
有密钥时可再跑：  
  
```bash  
python -m scripts.ask "这个项目部署时用哪个端口？"  
```  
  
期望：输出含 `8000` 的答案。  
  
---  
  
## 第 6 天：D-01 ～ D-05（5 个锚点）  
  
| 顺序 | 锚点 | 文件 | 备注 |  
|---|---|---|---|  
| 1 | D-01 | `app/api/__init__.py`、`app/api/v1/__init__.py`、`app/api/v1/health.py` | |  
| 2 | D-02 | `app/api/v1/schemas.py` | |  
| 3 | D-03 | `app/api/v1/chat.py` | **本文件先敲【D-03】的普通问答路由** |  
| 4 | D-04 | `app/agent/executor.py` | **本文件只敲【D-04】的 `stream()` 那一段** |  
| 5 | D-05 | `app/api/v1/chat.py` | **同一天第二段：只敲【D-05】的流式路由** |  
  
**敲完能验证**：两个路由都注册成功，健康检查可用。  
  
```bash  
python -c "from app.api.v1.chat import router; print([r.path for r in router.routes])"  
```  
  
期望：`['/chat', '/chat/stream']`  
  
---  
  
## 第 7 天：D-06 ～ D-12（7 个锚点）  
  
| 顺序 | 锚点 | 文件 | 备注 |  
|---|---|---|---|  
| 1 | D-06 | `app/main.py` | **先敲【D-06】创建应用与挂路由** |  
| 2 | D-07 | `app/core/exceptions.py`、`app/main.py` | **两个文件都只敲【D-07】那一段：处理器与注册** |  
| 3 | D-08 | `tests/__init__.py`、`tests/conftest.py` | |  
| 4 | D-09 | `tests/test_pipeline.py` | |  
| 5 | D-10 | `tests/test_api.py` | **先敲【D-10】的接口用例** |  
| 6 | D-11 | `tests/test_api.py` | **同一天第二段：只敲【D-11】的流式用例** |  
| 7 | D-12 | 无新增文件，执行总验收 | |  
  
**敲完能验证**：全部测试通过。  
  
```bash  
pytest -q  
```  
  
期望：全部通过，无失败。  
  
**最后一天总验收**：  
  
```bash  
uvicorn app.main:app --host 0.0.0.0 --port 8000  
```  
  
另开一个终端：  
  
```bash  
curl -s -X POST http://127.0.0.1:8000/api/v1/chat \  
  -H "Content-Type: application/json" \  
  -d '{"question":"这个项目部署时用哪个端口？"}'  
```  
  
期望：HTTP 200，返回体含 `"answer"`（内容含 `8000`）与 `"elapsed_ms"`（正整数）。  
  
---  
  
## 跟敲约定  
  
1. 每天只按表里的锚点顺序敲，不回头看文件树。  
2. 锚点注释原样保留，它是需求文档、代码、对照表三者唯一的对账依据。  
3. 同一天跨文件的锚点，先敲上游文件，再敲下游文件；表里的顺序就是依赖顺序。  
4. 某天验收不通过，不进入下一天；先修当天的锚点。  
5. 第 7 天 `pytest -q` 不通过，不跑总验收。  
  
---  
  
三件套到此完整交付：A 需求文档、B 全部代码（三批）、C 对照表、D 跟敲路线。  
