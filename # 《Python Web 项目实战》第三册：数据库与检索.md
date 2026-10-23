# 《Python Web 项目实战》第三册：数据库与检索  
  
| 项目 | 内容 |  
|---|---|  
| 系列 | 《Python Web 项目实战》 |  
| 册数 | 第三册（开发阶段三：Day 5-6） |  
| 主题 | 建表 · 读写 · 批量入库 · 检索 · 测试库 |  
| 前置要求 | 第一册（配置、日志、异常）、第二册（`list[dict]` 分块） |  
| 版本 | v1.7 · 2026-10-23 |  
  
## 本册地图  
  
```  
输入：第二册的 chunks（内存 list[dict]）  
  → 第1章 建表      4 个文件：定义存什么  
  → 第2章 读写      2 个文件：定义怎么取、怎么改  
  → 第3章 批量入库  1 个文件：解决写很多、写重复  
  → 第4章 检索      2 个文件：解决按内容找  
  → 第5章 测试库    2 个文件：固定前四章  
输出：可持久化、可检索、可测试的文档库  
```  
  
每章只做一件事，且只在上章产出上增量，不重写。  
  
术语校准：ORM 是对象与表行的映射层，不是数据库；数据仍存在 SQLite 文件里。  
  
## 目录  
  
1. 第 1 章 建表  
2. 第 2 章 读写  
3. 第 3 章 批量入库  
4. 第 4 章 检索  
5. 第 5 章 测试库  
6. 运行与验收  
7. 知识地图  
8. 自测  
9. 规范自检记录  
  
---  
  
## 第 1 章 建表  
  
### 1.1 场景与动机  
  
本章做：定义文档与分块两个对象，并创建两张表。  
本章不做：增删改查、批量写入、检索、分页、测试库。  
  
① 第二册交出的是内存里的 `chunks`（`list[dict]`），进程一退出就没了。  
② 10 份文档、单份 500 块时，重启一次全部丢失，无法按文档查回。  
③ 本章把这两份数据落到数据库。  
④ 选 SQLite 加 ORM：单机零运维，表结构与关系能复用。  
  
### 1.2 数据建模  
  
① 业务对象清单：  
  
- `Document`：一份用户上传的原始文档。  
- `Chunk`：从某份文档切出的一个文本块。  
  
② 字段表：  
  
| 对象 | 字段 | 类型 | 为什么需要 | 约束 |  
|---|---|---|---|---|  
| Document | id | 整数 | 唯一标识、被分块引用 | 主键 |  
| Document | filename | 字符串(255) | 列表展示、按名查找 | 索引、非空 |  
| Document | source_path | 字符串(512) | 回溯原始文件 | 非空 |  
| Document | created_at | 时间 | 列表按时间排序 | 服务端默认 |  
| Chunk | id | 整数 | 唯一标识 | 主键 |  
| Chunk | document_id | 整数 | 归属哪份文档 | 外键、索引、非空 |  
| Chunk | index | 整数 | 文档内的块序号 | 非空 |  
| Chunk | text | 长文本 | 检索与拼上下文 | 非空 |  
| Chunk | char_count | 整数 | 统计与阈值判断 | 非空 |  
  
③ 关系与基数：`Document 1:N Chunk`。一份文档切出多块，一个块只属于一份文档，所以是一对多，不是多对多。  
  
④ 替代建模与否决：把全部分块塞进 `Document` 的一个 JSON 字段。否决理由：无法按块分页检索，改一块要重写整列，单行会膨胀到几百 KB，索引完全失效。  
  
⑤ 呼应动机：两张表加外键，使"按文档取块"和"删文档带走块"同时成立，解决 ② 的丢失与查不回。  
  
### 1.3 最小可运行版  
  
本段做什么：用两个真实模型建出两张表，不写索引、约束、级联、异常、日志、配置。  
  
```python  
# 本段：最小可运行版，两张表、最简实现  
from datetime import datetime  
from sqlalchemy import DateTime, String, Text, create_engine  
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column  
class Base(DeclarativeBase):  
    """模型基类。"""  
class Document(Base):  
    """一份上传的原始文档。"""  
    __tablename__ = "documents"  
    id: Mapped[int] = mapped_column(primary_key=True)  
    filename: Mapped[str] = mapped_column(String(255))  
    source_path: Mapped[str] = mapped_column(String(512))  
    created_at: Mapped[datetime] = mapped_column(DateTime)  
class Chunk(Base):  
    """一个文本分块。"""  
    __tablename__ = "chunks"  
    id: Mapped[int] = mapped_column(primary_key=True)  
    document_id: Mapped[int] = mapped_column()  
    index: Mapped[int] = mapped_column()  
    text: Mapped[str] = mapped_column(Text)  
    char_count: Mapped[int] = mapped_column()  
engine = create_engine("sqlite:///./demo.db")  
Base.metadata.create_all(engine)  
```  
  
验证：  
  
```bash  
python demo_models.py  
sqlite3 demo.db ".tables"  
```  
  
预期：  
  
```  
chunks  documents  
```  
  
### 1.4 分段讲解  
  
| 代码 | 含义 |  
|---|---|  
| `create_engine("sqlite:///./demo.db")` | 建引擎；`sqlite:///` 后是本地库文件路径 |  
| `class Base(DeclarativeBase)` | 模型基类，收集全部表结构 |  
| `__tablename__` | 指定数据库中的真实表名 |  
| `Mapped[int]` | 类型注解，声明列类型 |  
| `mapped_column(primary_key=True)` | 映射为列；设主键 |  
| `String(255)` | 变长字符串，上限 255 |  
| `mapped_column()` | 无参数，类型由注解推断 |  
| `Base.metadata.create_all(engine)` | 建所有已注册表，已存在则跳过 |  
  
**知识点 1：库、表、行、列、主键**  
  
定义：数据库由表组成，表由列定义结构、由行存数据，主键是唯一标识一行的列。  
  
定式（要背）：  
  
```python  
# 心智模型：库=文件，表=一类对象，列=属性，行=一个对象，主键=唯一编号  
```  
  
适用：任何持久化设计前先列字段。例题：`documents` 存 2 条后表中有 2 行，`id` 为 1、2。  
  
**知识点 2：Engine**  
  
定义：Engine 是数据库入口，内部维护连接池，首次执行语句时才建立连接。  
  
定式（要背）：  
  
```python  
engine = create_engine(settings.DATABASE_URL, echo=False)   # 全项目只建一次  
```  
  
适用：模块级单例。例题：只 `create_engine` 不产生文件，`create_all` 后才出现库文件。  
  
**知识点 3：ORM 映射**  
  
定义：ORM 把类映射为表、属性映射为列、实例映射为行。  
  
定式（要背）：  
  
```python  
class User(Base):  
    __tablename__ = "users"  
    id: Mapped[int] = mapped_column(primary_key=True)  
```  
  
适用：需要类型提示与关系导航的业务表。例题：实例 `commit` 后对应表一行，`id` 自动赋值。  
  
**知识点 4：外键与关系**  
  
定义：外键列指向另一张表的主键；`relationship` 提供 Python 侧对象导航。  
  
定式（要背）：  
  
```python  
document_id: Mapped[int] = mapped_column(ForeignKey("documents.id", ondelete="CASCADE"))  
document: Mapped["Document"] = relationship(back_populates="chunks")  
```  
  
适用：一对多、需要级联删除时。例题：`chunk.document` 直接返回所属文档，不用手写 JOIN。  
  
### 1.5 指出缺陷  
  
以下缺陷全部落在"建表"这一动作内，且都能在 1.3 的代码中指到具体类或列。  
  
- **D1**：`Chunk.document_id` 只是普通整数列，未声明 `ForeignKey` → 删除 1 份文档后其 300 条分块残留，统计总数偏大 300。  
- **D2**：`Chunk` 未声明 `UniqueConstraint(document_id, index)` → 同一文档同一序号可写入两条，检索结果重复。  
- **D3**：`Document.created_at` 未设 `server_default` → 绕过 ORM 直接 SQL 插入时该列为 `NULL`，按时间排序报 `TypeError: '<' not supported between instances of 'NoneType' and 'datetime'`。  
- **D4**：`Document` 与 `Chunk` 同处一个文件 → 加关系注解时需互相 import，形成循环导入；改文档字段要滚过分块定义。  
- **D5**：`Document.filename` 未加 `index=True` → 10 万行按文件名查询走全表扫描，单次约 1.2 秒。  
  
### 1.6 结构设计  
  
① 文件清单：  
  
| 文件路径 | 一句话职责 | 依赖谁 |  
|---|---|---|  
| `app/db/base.py` | 建引擎、定义模型基类 | `sqlalchemy`、`app.config` |  
| `app/models/document.py` | 文档表映射 | `app.db.base` |  
| `app/models/chunk.py` | 分块表映射 | `app.db.base` |  
| `app/models/__init__.py` | 集中注册两张表 | 上面两个模型 |  
  
② 拆分理由：  
  
- `base.py` 单独：解决 D4，引擎与基类从模型文件里抽出，模型之间不再互相 import。  
- `document.py` / `chunk.py` 分文件：解决 D4，改文档字段只动一个文件。  
- `ForeignKey` + `relationship(cascade)`：解决 D1。  
- `UniqueConstraint`：解决 D2。  
- `server_default=func.now()`：解决 D3。  
- `index=True`：解决 D5。  
- `__init__.py`：落实数据建模结论"建表前必须注册两张表"，避免 `create_all` 漏表。  
  
③ 不这么拆会怎样：单文件里 `Chunk` 需要 `Document` 才能写关系注解，`Document` 又需要 `Chunk`，启动报 `ImportError: cannot import name 'Document' from partially initialized module`；改文件名索引要滚过 21 行模型定义。  
  
④ 依赖图：  
  
```  
app.config  
    ↓  
app/db/base.py (engine, Base)  
    ↓  
app/models/document.py  ←→  app/models/chunk.py   (字符串引用，不 import)  
    ↓  
app/models/__init__.py  
```  
  
⑤ 分工：设计决策回答"用内存、JSON 还是 ORM+SQLite"；结构设计回答"这四个文件怎么放、谁依赖谁"。  
  
### 1.7 完整版  
  
**设计决策：存储与访问方式选型**  
  
① 痛点：D1–D5 合计造成孤儿数据 300 条、重复块、时间排序崩溃、改字段动两处、按名查询 1.2 秒；裸 SQL 还会在 12 处重复拼接。  
  
② 取舍：  
  
- 方案 A：进程内 `dict`。适用于单次运行、可重算的数据。否决理由：重启即丢，多进程不共享，无法按条件查询。  
- 方案 B：本地 JSON 文件。适用于几百条以内、只整体读写的配置。否决理由：并发写互相覆盖，无法建索引，改一条要重写整个文件。  
- 方案 C：ORM + SQLite。适用于单机、十万级、需要事务与关系。选它。  
  
③ 触发条件：单机部署、数据十万级以内、需要事务或关系时选 C；只读静态配置选 B；纯缓存选 A。迁到多机或多写入时换 PostgreSQL。  
  
**文件 1：`app/db/base.py`**  
  
本段做什么：建引擎，定义所有模型共用的基类。  
  
```python  
# 本段：引擎与基类  
"""数据库基础：引擎与模型基类。"""  
from pathlib import Path                              # 标准库 · 路径 · 建数据目录  
  
from sqlalchemy import create_engine                  # 第三方库 · SQLAlchemy · 建引擎  
from sqlalchemy.orm import DeclarativeBase            # 第三方库 · ORM · 模型基类  
  
from app.config import settings                       # 本项目 · 配置 · 读数据库地址  
  
Path("data").mkdir(parents=True, exist_ok=True)       # 关键行：SQLite 不自动建目录  
  
engine = create_engine(settings.DATABASE_URL, echo=False)  
  
  
class Base(DeclarativeBase):  
    """所有 ORM 模型的基类，负责收集表结构元数据。"""  
```  
  
| 代码 | 含义 |  
|---|---|  
| `Path(...).mkdir(parents=True, exist_ok=True)` | 建目录；已存在不报错 |  
| `create_engine(url, echo=False)` | 建引擎；`echo=True` 打印 SQL |  
| `class Base(DeclarativeBase)` | 继承官方基类，自动收集子类表结构 |  
  
**文件 2：`app/models/document.py`**  
  
本段做什么：定义文档表，声明与分块的一对多关系。  
  
```python  
# 本段：Document 模型  
"""文档表模型。"""  
from datetime import datetime                          # 标准库 · 时间 · 创建时间类型  
  
from sqlalchemy import DateTime, String, func          # 第三方库 · 列类型与函数  
from sqlalchemy.orm import Mapped, mapped_column, relationship  # 第三方库 · ORM  
  
from app.db.base import Base                           # 本项目 · 基类  
  
  
class Document(Base):  
    """一份上传的原始文档，对应 documents 表。"""  
  
    __tablename__ = "documents"  
  
    id: Mapped[int] = mapped_column(primary_key=True)  
    filename: Mapped[str] = mapped_column(String(255), index=True)  
    source_path: Mapped[str] = mapped_column(String(512))  
    created_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())  
  
    chunks: Mapped[list["Chunk"]] = relationship(      # 关键行：一对多 + 级联删除  
        back_populates="document",  
        cascade="all, delete-orphan",  
    )  
```  
  
| 代码 | 含义 |  
|---|---|  
| `String(255)` | 变长字符串上限 255 |  
| `index=True` | 建索引，按文件名查询不全表扫 |  
| `server_default=func.now()` | 由数据库生成插入时间 |  
| `relationship(back_populates, cascade)` | 建关系；`delete-orphan` 父删子删 |  
| `Mapped[list["Chunk"]]` | 声明多的一侧，字符串避免循环导入 |  
  
**文件 3：`app/models/chunk.py`**  
  
本段做什么：定义分块表，加外键、唯一约束与索引。  
  
```python  
# 本段：Chunk 模型  
"""分块表模型。"""  
from sqlalchemy import ForeignKey, Text, UniqueConstraint  # 第三方库 · 列与约束  
from sqlalchemy.orm import Mapped, mapped_column, relationship  # 第三方库 · ORM  
  
from app.db.base import Base                                # 本项目 · 基类  
  
  
class Chunk(Base):  
    """一个文本分块，对应 chunks 表。"""  
  
    __tablename__ = "chunks"  
    __table_args__ = (UniqueConstraint("document_id", "index", name="uq_chunk_doc_index"),)  
  
    id: Mapped[int] = mapped_column(primary_key=True)  
    document_id: Mapped[int] = mapped_column(  
        ForeignKey("documents.id", ondelete="CASCADE"), index=True  
    )  
    index: Mapped[int] = mapped_column()  
    text: Mapped[str] = mapped_column(Text)  
    char_count: Mapped[int] = mapped_column()  
  
    document: Mapped["Document"] = relationship(back_populates="chunks")  
```  
  
| 代码 | 含义 |  
|---|---|  
| `UniqueConstraint(...)` | 同文档同序号只能一条 |  
| `ForeignKey(..., ondelete="CASCADE")` | 外键，数据库层级联删除 |  
| `Text()` | 不限长度文本列 |  
| `index=True` | 按 `document_id` 查询走索引 |  
| `relationship(back_populates="chunks")` | 反向导航 |  
  
**文件 4：`app/models/__init__.py`**  
  
本段做什么：集中导入两张表，保证建表前全部注册。  
  
```python  
# 本段：模型注册  
"""集中导入所有模型，确保 create_all 能发现它们。"""  
from app.models.chunk import Chunk          # 本项目 · 模型 · 注册 chunks  
from app.models.document import Document    # 本项目 · 模型 · 注册 documents  
  
__all__ = ["Chunk", "Document"]  
```  
  
| 代码 | 含义 |  
|---|---|  
| `__all__` | 声明本包公开导出的名字 |  
  
固定模板：`base.py` 的引擎与基类、两个模型类、`__init__.py` 注册。可组合写法：列长度、索引位置、是否级联、时间默认值来源。  
  
已知边界：SQLite 默认不强制外键，需 `PRAGMA foreign_keys=ON`；本册靠 ORM 级联，不强制也能删干净。  
  
调用链：模型由 `create_all` 与后续仓储使用；它们调用 `Base.metadata` 注册结构；建表失败抛 `SQLAlchemyError`，属启动错误，应让进程退出。  
  
### 1.8 结尾块  
  
**本章判据**：一次 `create_all` 生成 `documents` 与 `chunks` 两张表，字段与 1.2 的字段表一致。  
  
**本章产出**：`app/db/base.py`、`app/models/document.py`、`app/models/chunk.py`、`app/models/__init__.py`。  
  
**下一章入口**：打开 `app/db/session.py`，加会话工厂与 `get_db`。  
  
**常见错法 3 条**：  
  
1. 每个文件各自 `create_engine`。  
2. 只写整数列，不写 `ForeignKey`。  
3. 忘记在 `__init__.py` 导入模型，建表少一张。  
  
**自检 3 问**：  
  
1. 加一个字段要动几个文件？  
2. 删除文档后，分块会留下吗？  
3. 同一文档同一序号能写两条吗？  
  
---  
  
## 第 2 章 读写  
  
### 2.1 场景与动机  
  
本章做：为一对表提供统一的增删查改入口。  
本章不做：批量写入、重复导入、检索、分页、测试库。  
  
① 第 1 章交出的是 `app/models/` 下的 `Document` 与 `Chunk` 两张表的映射，数据在 `data/insight.db`。  
② 3 个接口时手写会话还能忍；8 个接口就是 48 行样板，50 并发后报 `database is locked`。  
③ 本章要给读写一个固定入口与事务边界。  
④ 选请求级 Session 加仓储：会话可注入，事务边界集中。  
  
本章不涉及新数据结构。  
  
### 2.2 最小可运行版  
  
本段做什么：一段脚本完成一次写入与一次读取，覆盖两个对象。  
  
```python  
# 本段：最小可运行版，读写一段完成  
from sqlalchemy.orm import Session  
from app.db.base import engine  
from app.models.document import Document  
from app.models.chunk import Chunk  
  
with Session(engine) as session:  
    doc = Document(filename="a.txt", source_path="data/a.txt")  
    session.add(doc)  
    session.commit()  
    chunk = Chunk(document_id=doc.id, index=0, text="内容", char_count=2)  
    session.add(chunk)  
    session.commit()  
    print(session.get(Document, doc.id).filename)  
```  
  
### 2.3 分段讲解  
  
| 代码 | 含义 |  
|---|---|  
| `Session(engine)` | 创建会话，绑定引擎 |  
| `with ... as session` | 离开块自动关闭 |  
| `Document(...)` | 构造待插入对象，未写库 |  
| `session.add(doc)` | 加入待写入集合 |  
| `session.commit()` | 提交事务，真正写库 |  
| `session.get(Document, id)` | 按主键取，未命中返回 `None` |  
  
**知识点 1：Session**  
  
定义：Session 是一次数据库对话的工作区，跟踪对象变化、组织事务、管理连接。  
  
定式（要背）：  
  
```python  
with Session(engine) as session:   # 每个请求或任务一个，不跨请求共享  
    ...  
```  
  
适用：所有数据库读写。例题：两个 `with` 块互不影响，第一块未提交的改动第二块查不到。  
  
**知识点 2：`commit` / `rollback`**  
  
定义：`commit` 提交并结束事务；`rollback` 撤销当前事务。  
  
定式（要背）：  
  
```python  
try:  
    session.add(obj)  
    session.commit()  
except Exception:  
    session.rollback()  
    raise  
```  
  
适用：所有写操作。例题：`add` 后 `rollback`，再查为空。  
  
**知识点 3：`select` 与 `where`**  
  
定义：`select(模型)` 构造查询，`where(条件)` 过滤，`scalars()` 取模型实例。  
  
定式（要背）：  
  
```python  
rows = session.scalars(select(Document).where(Document.id == 1)).all()  
```  
  
适用：列表与条件查询。例题：3 条记录中 `where(id == 2)` 返回 1 条。  
  
**知识点 4：`get`**  
  
定义：`Session.get(模型, 主键)` 按主键查询，未命中返回 `None`。  
  
定式（要背）：  
  
```python  
doc = session.get(Document, 1)   # 不存在返回 None，不抛异常  
```  
  
适用：详情页、删除前查询。例题：`get(Document, 999)` 返回 `None`。  
  
### 2.4 指出缺陷  
  
以下缺陷只评"读写"这一动作，且都能在 2.2 的代码里指到具体类或调用。  
  
- **D1**：`session.add(doc)` 后 `commit()`，随后再 `add(chunk)` 再 `commit()` → 第二次提交失败时，文档已落库、分块没有，出现"有文档、0 块"的半写数据。  
- **D2**：`Session(engine)` 直接写在业务代码里 → 复制到 8 个接口共 48 行样板；测试无法替换成内存库。  
- **D3**：写入与 `session.get(...)` 查询混在同一段 → 查询逻辑无法复用，加排序或过滤要改业务代码。  
- **D4**：`add` 与 `commit` 分散在业务逻辑中 → 没有统一事务入口，新增调用点必漏。  
  
### 2.5 结构设计  
  
① 文件清单：  
  
| 文件路径 | 一句话职责 | 依赖谁 |  
|---|---|---|  
| `app/db/session.py` | 会话工厂与请求级 `get_db` | `app.db.base.engine` |  
| `app/repositories/document_repo.py` | 文档增删查改 | `Session`、`models.document` |  
  
② 拆分理由：  
  
- `session.py`：解决 D2，会话创建收成一处，异常路径也关闭。  
- `document_repo.py`：解决 D1，每个写操作一个事务边界；解决 D3、D4，读写各自成函数，调用方不再自己 `commit`。  
  
③ 不这么拆会怎样：会话代码留在 8 个路由里，新增第 9 个接口必漏关闭；读写混写导致想给列表加排序必须改业务函数，无法单独测试。  
  
④ 依赖图：  
  
```  
app/db/base.py (engine)  
    ↓  
app/db/session.py (SessionLocal, get_db)  
    ↓  
app/repositories/document_repo.py  
    ↓  
app/api/... (Depends 注入)  
```  
  
⑤ 分工：设计决策回答"全局单例还是请求级注入"；结构设计回答"会话与 CRUD 分放哪两个文件"。  
  
### 2.6 完整版  
  
**设计决策：请求级 Session + 仓储**  
  
① 痛点：D2 造成 48 行样板与无法替换测试；D1 造成半写；D4 造成每个调用点各自提交。  
  
② 取舍：  
  
- 方案 A：全局单例 Session。适用于单线程脚本。否决理由：并发请求共享同一事务，一个 `rollback` 会丢掉另一个请求未提交的写入。  
- 方案 B：每个接口手写 `try/finally: close()`。适用于只有 2～3 个入口。否决理由：8 处复制，新增必漏；测试替换点分散。  
  
③ 触发条件：接口 ≥3 且都访问数据库时抽象；单脚本不抽象。只剩 1 个入口且无测试要求时，退回函数内创建。  
  
**文件 1：`app/db/session.py`**  
  
本段做什么：建会话工厂，提供每请求自动关闭的依赖。  
  
```python  
# 本段：会话工厂与依赖  
"""会话工厂与请求级依赖。"""  
from collections.abc import Generator                  # 标准库 · 类型 · 生成器返回类型  
  
from sqlalchemy.orm import Session, sessionmaker       # 第三方库 · ORM · 会话与工厂  
  
from app.db.base import engine                         # 本项目 · 引擎  
  
SessionLocal = sessionmaker(bind=engine, autoflush=False, expire_on_commit=False)  
  
  
def get_db() -> Generator[Session, None, None]:  
    """FastAPI 依赖：每个请求一个 Session，结束后自动关闭。"""  
    db = SessionLocal()  
    try:  
        yield db                                       # 关键行：交给接口使用  
    finally:  
        db.close()                                     # 关键行：无论成败都关闭  
```  
  
| 代码 | 含义 |  
|---|---|  
| `sessionmaker(bind, autoflush, expire_on_commit)` | 生成会话工厂 |  
| `expire_on_commit=False` | 提交后对象仍可读，序列化不再查库 |  
| `SessionLocal()` | 创建一个会话 |  
| `yield db` | 交给调用方，函数在此暂停 |  
| `finally: db.close()` | 调用方结束后必执行 |  
  
**文件 2：`app/repositories/document_repo.py` 第 1 段**  
  
本段做什么：实现新增与按主键查询。  
  
```python  
# 本段：创建与查询  
"""文档仓储：只做数据库读写，不含 HTTP 逻辑。"""  
from sqlalchemy import select                          # 第三方库 · SQL · 构造查询  
from sqlalchemy.orm import Session                     # 第三方库 · ORM · 会话类型  
  
from app.models.document import Document               # 本项目 · 模型  
  
  
def create_document(db: Session, filename: str, source_path: str) -> Document:  
    """新增文档并返回带主键的实例。"""  
    doc = Document(filename=filename, source_path=source_path)  
    db.add(doc)  
    db.commit()  
    db.refresh(doc)                                    # 关键行：取回 id 与默认时间  
    return doc  
  
  
def get_document(db: Session, document_id: int) -> Document | None:  
    """按主键取文档，找不到返回 None。"""  
    return db.get(Document, document_id)  
```  
  
| 代码 | 含义 |  
|---|---|  
| `db.add(doc)` | 加入待写入集合 |  
| `db.commit()` | 提交事务 |  
| `db.refresh(doc)` | 从库重载字段，补自增 id |  
| `db.get(Document, id)` | 按主键查询 |  
  
**文件 2 第 2 段**  
  
本段做什么：实现列表与删除，删除走级联。  
  
```python  
# 本段：列表与删除  
def list_documents(db: Session) -> list[Document]:  
    """按 id 倒序返回全部文档。"""  
    stmt = select(Document).order_by(Document.id.desc())  
    return list(db.scalars(stmt).all())  
  
  
def delete_document(db: Session, document_id: int) -> bool:  
    """删除文档及其分块，返回是否成功。"""  
    doc = db.get(Document, document_id)  
    if doc is None:  
        return False  
    db.delete(doc)                                     # 关键行：级联删除其 chunks  
    db.commit()  
    return True  
```  
  
| 代码 | 含义 |  
|---|---|  
| `select(...).order_by(col.desc())` | 查询并降序 |  
| `db.scalars(stmt).all()` | 取模型实例列表 |  
| `db.delete(doc)` | 标记删除，提交后生效并触发级联 |  
  
固定模板：`session.py` 的工厂与 `get_db`、四个仓储函数签名。可组合写法：排序字段、分页、是否软删除。  
  
已知边界：`get_db` 不自动 `rollback`；写失败必须在仓储内回滚，否则连接归还时事务仍挂着。  
  
调用链：`get_db` 由路由经 `Depends` 注入；仓储由路由或服务层调用；它们调用 `Session.add/commit/get/scalars/delete`；异常冒泡到全局处理器，`get` 返回 `None` 时接口转 404。  
  
### 2.7 结尾块  
  
**本章判据**：四个读写操作都在仓储层完成，接口层看不到 `commit`。  
  
**本章产出**：`app/db/session.py`、`app/repositories/document_repo.py`。  
  
**下一章入口**：打开 `app/services/ingest.py`，加批量写入。  
  
**常见错法 3 条**：  
  
1. 路由里直接 `SessionLocal()`。  
2. 写操作不包 `try/rollback`。  
3. 仓储里返回 HTTP 状态码。  
  
**自检 3 问**：  
  
1. 请求抛异常时会话会关闭吗？  
2. 两次提交之间失败会留半份数据吗？  
3. 测试能换成内存库吗？  
  
---  
  
## 第 3 章 批量入库  
  
### 3.1 场景与动机  
  
本章做：把一份文档的全部分块一次写入。  
本章不做：建表、单条 CRUD、检索、分页、测试库。  
  
① 第 2 章交出的是 `app/repositories/document_repo.py`，能建、查、删文档，但文档建完没有任何分块。  
② 逐条提交写 500 块约 2.3 秒；重复上传一次，条数从 500 变 1000。  
③ 本章要一次写入全部分块，并让重复导入不翻倍。  
④ 选先删后插加单事务：整份覆盖、失败可整体回滚。  
  
本章不涉及新数据结构。  
  
### 3.2 最小可运行版  
  
本段做什么：用循环逐条写入分块，只保证功能。  
  
```python  
# 本段：最小可运行版，逐条写入  
from sqlalchemy.orm import Session  
from app.db.base import engine  
from app.models.document import Document  
from app.models.chunk import Chunk  
  
chunks = [{"index": i, "text": f"块{i}", "char_count": 2} for i in range(3)]  
  
with Session(engine) as session:  
    doc = Document(filename="a.txt", source_path="data/a.txt")  
    session.add(doc)  
    session.commit()  
    for c in chunks:  
        session.add(Chunk(document_id=doc.id, index=c["index"], text=c["text"], char_count=c["char_count"]))  
        session.commit()  
```  
  
### 3.3 分段讲解  
  
| 代码 | 含义 |  
|---|---|  
| 列表推导 | 造 3 条分块数据，形状与第二册一致 |  
| `session.add(doc)` + `commit` | 先拿到文档 id |  
| `for c in chunks` | 逐条处理 |  
| `session.add(Chunk(...))` | 每次加入一条 |  
| `session.commit()` 在循环内 | 每条一次事务 |  
  
**知识点 1：批量写入 `add_all`**  
  
定义：`add_all` 接收对象列表一次加入会话，提交时统一发往数据库。  
  
定式（要背）：  
  
```python  
session.add_all([Model(**row) for row in rows])   # 一次事务写多行  
session.commit()  
```  
  
适用：一次几十条以上。例题：500 条时事务次数从 500 降到 1。  
  
**知识点 2：事务边界**  
  
定义：一个事务内的写入要么全部成功，要么全部撤销。  
  
定式（要背）：  
  
```python  
try:  
    db.execute(delete(...))  
    db.add_all(...)  
    db.commit()  
except Exception:  
    db.rollback()  
    raise  
```  
  
适用：关联写入、先删后插。例题：插入 2 条后主动抛错，回滚后 0 条。  
  
**知识点 3：幂等写入**  
  
定义：同一请求执行多次，结果与执行一次相同。  
  
定式（要背）：  
  
```python  
db.execute(delete(Chunk).where(Chunk.document_id == doc_id))   # 先清  
db.add_all(new_chunks)                                         # 再写  
db.commit()  
```  
  
适用：文件重传、任务重试。例题：同一份 3 块导入两次，表中仍 3 条。  
  
### 3.4 指出缺陷  
  
以下缺陷只评"批量"这一动作，均可在 3.2 代码中定位。  
  
- **D1**：`commit()` 写在 `for c in chunks` 循环内 → 500 块 500 次事务，约 2.3 秒；一次批量提交约 0.05 秒。  
- **D2**：写入前没有删除该文档旧分块 → 重复上传一次，条数从 500 变 1000，检索重复率 50%。  
- **D3**：循环外没有 `try/except` 与 `rollback` → 写到第 200 条抛错时，前 200 条已提交，留下残缺文档。  
- **D4**：写入逻辑直接写在脚本里 → 上传接口、离线导入任务各复制一份，改分块字段要动两处。  
  
### 3.5 结构设计  
  
① 文件清单：  
  
| 文件路径 | 一句话职责 | 依赖谁 |  
|---|---|---|  
| `app/services/ingest.py` | 批量写入与幂等事务 | `Session`、`models.chunk`、`logging` |  
| `app/services/__init__.py` | 标记服务包 | 无 |  
  
② 拆分理由：  
  
- `ingest.py`：解决 D1，`add_all` 加单次提交；解决 D2，先 `delete` 再写；解决 D3，`try` 内提交、异常 `rollback`；解决 D4，逻辑集中一处，接口与离线任务都调它。  
  
③ 不这么拆会怎样：把批量写塞进 `document_repo.py`，该文件从 40 行涨到 120 行，改文档逻辑要连带回归分块；两个调用点各写一份，幂等行为会漂移。  
  
④ 依赖图：  
  
```  
app/models/chunk.py  
        ↑  
app/services/ingest.py (先删后插 + 单事务)  
        ↑  
app/api/upload.py 与 scripts/import.py  
```  
  
⑤ 分工：设计决策回答"逐条写、先删后插还是 upsert"；结构设计回答"批量逻辑放服务层，不塞仓储"。  
  
### 3.6 完整版  
  
**设计决策：先删后插，而不是 upsert**  
  
① 痛点：D1 的 2.3 秒、D2 的翻倍、D3 的半份数据。  
  
② 取舍：  
  
- 方案 A：逐条 `insert`。适用于少于 50 条。否决理由：500 条 500 次事务，且无法处理重复。  
- 方案 B：`upsert`。适用于保留历史版本或只更新部分块。否决理由：本册是整份重传，逐块比对差异复杂度高于直接覆盖，旧块序号变化还会残留。  
  
③ 触发条件：整份覆盖用先删后插；需要保留版本或增量更新时改 upsert。文档不允许重传时去掉删除步骤。  
  
**文件 1：`app/services/ingest.py` 第 1 段**  
  
本段做什么：定义入库异常类型。  
  
```python  
# 本段：异常与导入  
"""入库服务：批量写入分块，重复导入保持幂等。"""  
from sqlalchemy import delete                          # 第三方库 · SQL · 删除语句  
from sqlalchemy.exc import IntegrityError              # 第三方库 · 异常 · 约束冲突  
from sqlalchemy.orm import Session                     # 第三方库 · ORM · 会话  
  
from app.core.logging import get_logger                # 本项目 · 日志  
from app.models.chunk import Chunk                     # 本项目 · 模型  
  
logger = get_logger(__name__)  
  
  
class IngestError(Exception):  
    """分块入库失败。"""  
  
    def __init__(self, document_id: int, reason: str):  
        self.document_id = document_id  
        self.reason = reason  
        super().__init__(f"文档 {document_id} 入库失败: {reason}")  
```  
  
| 代码 | 含义 |  
|---|---|  
| `delete` | 构造 SQL 删除语句 |  
| `IntegrityError` | 唯一约束等完整性错误 |  
| `class IngestError(Exception)` | 业务异常，带文档 id 与原因 |  
| `super().__init__(...)` | 生成可读消息 |  
  
**文件 1 第 2 段**  
  
本段做什么：先清旧块，再批量插入。  
  
```python  
# 本段：覆盖式写入  
def save_chunks(db: Session, document_id: int, chunks: list[dict]) -> int:  
    """覆盖式写入：先删旧块，再批量插入。返回写入块数。"""  
    if not chunks:  
        return 0  
  
    try:  
        db.execute(delete(Chunk).where(Chunk.document_id == document_id))  # 关键行：先清  
        db.add_all([  
            Chunk(  
                document_id=document_id,  
                index=item["index"],  
                text=item["text"],  
                char_count=item["char_count"],  
            )  
            for item in chunks  
        ])  
```  
  
| 代码 | 含义 |  
|---|---|  
| `if not chunks: return 0` | 空输入不开事务 |  
| `db.execute(delete(...).where(...))` | 清掉该文档旧块 |  
| `db.add_all([...])` | 批量加入新块 |  
| `item["index"]` | 取第二册数据结构的字段 |  
  
**文件 1 第 3 段**  
  
本段做什么：提交，冲突时回滚并抛业务异常。  
  
```python  
# 本段：提交与回滚  
        db.commit()                                    # 关键行：删与插同一事务  
    except IntegrityError as e:  
        db.rollback()                                  # 关键行：冲突必须回滚  
        logger.error("写入冲突 | document_id=%s | %s", document_id, e)  
        raise IngestError(document_id, "分块序号冲突") from e  
  
    logger.info("写入完成 | document_id=%s | chunks=%d", document_id, len(chunks))  
    return len(chunks)  
```  
  
| 代码 | 含义 |  
|---|---|  
| `db.commit()` | 删除与插入一并提交 |  
| `db.rollback()` | 撤销整个事务 |  
| `logger.error(...)` | 错误日志，`%s` 由 logging 填充 |  
| `raise ... from e` | 保留数据库原始错误链 |  
  
固定模板：`delete + add_all + commit` 与 `IntegrityError → rollback → 自定义异常`。可组合写法：是否保留历史、失败是否重试。  
  
已知边界：覆盖写入在同一事务内短暂持写锁；SQLite 单写者下并发导入同一文档会串行等待。  
  
调用链：`save_chunks` 由上传接口调用；它调用 `Session.execute/add_all/commit`；`IntegrityError` 包装为 `IngestError`，最终映射 HTTP 409。  
  
### 3.7 结尾块  
  
**本章判据**：同一份分块导入 N 次，表中始终一份；中途失败不留残片。  
  
**本章产出**：`app/services/ingest.py`、`app/services/__init__.py`。  
  
**下一章入口**：打开 `app/repositories/chunk_repo.py`，加按关键词查询。  
  
**常见错法 3 条**：  
  
1. 循环里逐条 `commit`。  
2. 只 `add` 不先清旧数据。  
3. 捕获 `IntegrityError` 后不回滚。  
  
**自检 3 问**：  
  
1. 重复导入后条数会翻倍吗？  
2. 第 200 条失败时前 199 条会留下吗？  
3. 冲突后这个 Session 还能用吗？  
  
---  
  
## 第 4 章 检索  
  
### 4.1 场景与动机  
  
本章做：按关键词查分块，并支持分页。  
本章不做：建表、写入、幂等、测试库。  
  
① 第 3 章交出的是 `app/services/ingest.py`，分块已入 `chunks` 表，但只能按 `document_id` 取，不能按内容找。  
② 10 万行上 `LIKE '%部署%'` 约 1.2 秒；一次返回 10 万条约 40 MB，前端卡死。  
③ 本章要按内容找到相关块，并分页返回。  
④ 选 `LIKE` 加分页：数据万级，实现最简。  
  
本章不涉及新数据结构。  
  
### 4.2 最小可运行版  
  
本段做什么：按关键词取前 5 条，只保证能查到。  
  
```python  
# 本段：最小可运行版，关键词取前 5 条  
from sqlalchemy import select  
from sqlalchemy.orm import Session  
from app.db.base import engine  
from app.models.chunk import Chunk  
  
with Session(engine) as session:  
    stmt = select(Chunk).where(Chunk.text.contains("部署")).limit(5)  
    rows = session.scalars(stmt).all()  
    print([r.text for r in rows])  
```  
  
反例代码（不属于本章正式代码，仅用于说明拼接风险）：  
  
```python  
kw = "50%"  
sql = f"SELECT * FROM chunks WHERE text LIKE '%{kw}%'"  
```  
  
### 4.3 分段讲解  
  
| 代码 | 含义 |  
|---|---|  
| `select(Chunk)` | 构造查分块表的语句 |  
| `.where(Chunk.text.contains("部署"))` | 条件：文本包含该词 |  
| `.limit(5)` | 最多 5 条 |  
| `session.scalars(stmt).all()` | 执行并取实例列表 |  
| 反例中的 f-string | 把关键词直接拼进 SQL 字符串 |  
  
**知识点 1：`contains` 与 `LIKE`**  
  
定义：`contains` 生成 `LIKE '%关键词%'`；`autoescape=True` 时转义 `%`、`_`。  
  
定式（要背）：  
  
```python  
select(Chunk).where(Chunk.text.contains(keyword, autoescape=True))   # 参数绑定 + 转义  
```  
  
适用：万级数据的包含检索。例题：关键词 `"50%"` 按字面匹配，不当中通配符。  
  
**知识点 2：`limit` 与 `offset`**  
  
定义：`limit` 限制条数，`offset` 跳过前 N 条。  
  
定式（要背）：  
  
```python  
stmt = stmt.offset((page - 1) * page_size).limit(page_size)   # 第 page 页  
```  
  
适用：所有列表页。例题：共 25 条，`limit=10, offset=20` 返回 5 条。  
  
**知识点 3：`count`**  
  
定义：`select(func.count())` 统计满足条件的行数。  
  
定式（要背）：  
  
```python  
total = db.scalar(select(func.count()).select_from(Chunk).where(...)) or 0  
```  
  
适用：需要总页数的分页接口。例题：命中 25 条返回 25，与当前页条数无关。  
  
**知识点 4：索引**  
  
定义：索引是为某列额外维护的查找结构，用空间换查询时间。  
  
定式（要背）：  
  
```python  
filename: Mapped[str] = mapped_column(String(255), index=True)   # 高频过滤列  
```  
  
适用：高频出现在 `where`、`order_by`、`join` 的列。例题：10 万行按 `document_id` 查，有索引约 1 毫秒。  
  
### 4.4 指出缺陷  
  
以下缺陷只评"查询"这一动作，均可定位到 4.2 代码或反例块。  
  
- **D1**：`select(Chunk).where(...).limit(5)` 没有 `offset` → 永远只能看到前 5 条，无法翻页。  
- **D2**：没有 `count` 查询 → 前端拿不到总数，无法显示总页数。  
- **D3**：`select` 没有 `order_by` → 两次查询顺序可能不同，翻页出现重复或漏项。  
- **D4**：反例块用 f-string 拼接关键词 → 关键词含 `'` 时报 `sqlite3.OperationalError: near "'": syntax error`，含 `%` 时匹配范围被放大。  
  
### 4.5 结构设计  
  
① 文件清单：  
  
| 文件路径 | 一句话职责 | 依赖谁 |  
|---|---|---|  
| `app/repositories/chunk_repo.py` | 两条原子查询：检索、计数 | `Session`、`models.chunk` |  
| `app/services/search_service.py` | 页码换算与响应组装 | `chunk_repo` |  
  
② 拆分理由：  
  
- `chunk_repo.py`：解决 D3，查询内固定 `order_by`；解决 D4，用参数绑定替代拼接；提供 `offset/limit` 供上层调用。  
- `search_service.py`：解决 D1，把页码换算成 `offset`；解决 D2，调用计数并组装 `total`。  
  
③ 不这么拆会怎样：分页塞进仓储，每个查询都要重复页码校验；接口直接调仓储，换排序要改接口层，`total` 计算散落多个路由。  
  
④ 依赖图：  
  
```  
app/models/chunk.py  
        ↑  
app/repositories/chunk_repo.py (search / count)  
        ↑  
app/services/search_service.py (page rules)  
        ↑  
app/api/... 与后续 Agent 工具  
```  
  
⑤ 分工：设计决策回答"LIKE、FTS5 还是向量"；结构设计回答"查询放仓储、分页放服务"。  
  
### 4.6 完整版  
  
**设计决策：先用 `LIKE`，不上全文索引**  
  
① 痛点：D1–D4 造成只能看前 5 条、无总数、顺序漂移、特殊字符报错。  
  
② 取舍：  
  
- 方案 A：SQLite FTS5。适用于十万级以上、需要分词与相关度。否决理由：要额外建虚拟表与同步触发器，本册数据量未到，维护成本高于收益。  
- 方案 B：向量检索。适用于语义相似。否决理由：需嵌入模型与向量库，超出本阶段。  
  
③ 触发条件：超过 10 万条或要相关度排序上 FTS5；要"意思相近"上向量。少于 1 万条直接 `LIKE`。  
  
**文件 1：`app/repositories/chunk_repo.py` 第 1 段**  
  
本段做什么：按关键词检索，排序稳定。  
  
```python  
# 本段：检索查询  
"""分块检索仓储：只构造查询，不含分页规则。"""  
from sqlalchemy import func, select                    # 第三方库 · SQL · 查询与聚合  
from sqlalchemy.orm import Session                     # 第三方库 · ORM · 会话  
  
from app.models.chunk import Chunk                     # 本项目 · 模型  
  
  
def search_chunks(  
    db: Session,  
    keyword: str,  
    limit: int = 20,  
    offset: int = 0,  
) -> list[Chunk]:  
    """按关键词检索分块，按文档与块序号排序。"""  
    stmt = (  
        select(Chunk)  
        .where(Chunk.text.contains(keyword, autoescape=True))   # 关键行：绑定 + 转义  
        .order_by(Chunk.document_id, Chunk.index)  
        .offset(offset)  
        .limit(limit)  
    )  
    return list(db.scalars(stmt).all())  
```  
  
| 代码 | 含义 |  
|---|---|  
| `contains(keyword, autoescape=True)` | 生成 `LIKE` 并转义通配符 |  
| `.order_by(col1, col2)` | 多列排序，保证分页稳定 |  
| `.offset(n).limit(n)` | 跳过 n 条、最多取 n 条 |  
| `db.scalars(stmt).all()` | 执行并返回实例列表 |  
  
**文件 1 第 2 段**  
  
本段做什么：统计命中总数。  
  
```python  
# 本段：计数查询  
def count_chunks(db: Session, keyword: str) -> int:  
    """统计命中条数，用于分页总数。"""  
    stmt = (  
        select(func.count())  
        .select_from(Chunk)  
        .where(Chunk.text.contains(keyword, autoescape=True))  
    )  
    return db.scalar(stmt) or 0  
```  
  
| 代码 | 含义 |  
|---|---|  
| `select(func.count())` | 构造 `COUNT(*)` |  
| `.select_from(Chunk)` | 指定统计来源表 |  
| `db.scalar(stmt)` | 返回第一行第一列 |  
| `or 0` | 无结果时归一为 0 |  
  
**文件 2：`app/services/search_service.py`**  
  
本段做什么：页码换 offset，组装当前页与总数。  
  
```python  
# 本段：分页服务  
"""检索服务：组合仓储与分页规则。"""  
from sqlalchemy.orm import Session                     # 第三方库 · ORM · 会话类型  
  
from app.repositories.chunk_repo import count_chunks, search_chunks   # 本项目 · 仓储  
  
  
def search_page(db: Session, keyword: str, page: int = 1, page_size: int = 20) -> dict:  
    """返回当前页数据与总数，页码非法时归正为 1。"""  
    if page < 1:  
        page = 1                                        # 关键行：防负 offset  
    offset = (page - 1) * page_size  
    return {  
        "items": search_chunks(db, keyword, limit=page_size, offset=offset),  
        "total": count_chunks(db, keyword),  
        "page": page,  
        "page_size": page_size,  
    }  
```  
  
| 代码 | 含义 |  
|---|---|  
| `page < 1` 归正 | 避免负 offset |  
| `offset = (page - 1) * page_size` | 页码换算为跳过量 |  
| `search_chunks(...)` | 取当前页 |  
| `count_chunks(...)` | 取总数 |  
  
固定模板：`search_chunks` / `count_chunks` + `search_page`。可组合写法：`page_size` 上限、排序策略、叠加文档过滤。  
  
已知边界：`LIKE '%词%'` 用不到普通索引；百万级后需全文索引或向量检索。  
  
调用链：`search_page` 由检索接口与后续 Agent 工具调用；它调用 `search_chunks`、`count_chunks`；本身不抛业务异常，空关键词由接口层拦截。  
  
### 4.7 结尾块  
  
**本章判据**：给定关键词与页码，返回稳定排序的一页数据加总数，特殊字符不报错。  
  
**本章产出**：`app/repositories/chunk_repo.py`、`app/services/search_service.py`。  
  
**下一章入口**：打开 `tests/conftest.py`，加内存库夹具。  
  
**常见错法 3 条**：  
  
1. 用 f-string 拼关键词。  
2. 只 `limit` 不 `offset`。  
3. 不返回 `total`。  
  
**自检 3 问**：  
  
1. 关键词带 `%` 或单引号会报错吗？  
2. 第 3 页会和第 2 页重叠吗？  
3. 10 万行时这次查询多久？  
  
---  
  
## 第 5 章 测试库  
  
### 5.1 场景与动机  
  
本章做：让测试使用独立数据库，互不干扰。  
本章不做：新增业务功能、改表结构、改检索逻辑。  
  
① 第 4 章交出的是 `app/repositories/chunk_repo.py` 与 `search_service.py`，检索可用，但测试都连 `data/insight.db`。  
② 跑一次测试清空开发数据；CI 两个 job 并发写同一文件报 `database is locked`。  
③ 本章要让测试不碰真实库且能并行。  
④ 选内存库加 `StaticPool`：隔离、快、可并行。  
  
本章不涉及新数据结构。  
  
### 5.2 最小可运行版  
  
本段做什么：直接连真实库做一次断言，只保证测试能跑。  
  
```python  
# 本段：最小可运行版，直连真实库  
from sqlalchemy import create_engine  
from sqlalchemy.orm import Session  
from app.db.base import Base  
from app.models.document import Document  
  
def test_create_document():  
    """写入一条文档后断言表非空。"""  
    engine = create_engine("sqlite:///./data/insight.db")  
    Base.metadata.create_all(engine)  
    with Session(engine) as s:  
        s.add(Document(filename="a.txt", source_path="data/a.txt"))  
        s.commit()  
        assert s.query(Document).count() > 0  
```  
  
### 5.3 分段讲解  
  
| 代码 | 含义 |  
|---|---|  
| `create_engine("sqlite:///./data/insight.db")` | 连开发库文件 |  
| `Base.metadata.create_all(engine)` | 在开发库建表 |  
| `Session(engine)` | 建会话 |  
| `s.query(Document).count()` | 统计现有行数 |  
| `> 0` | 只断言"至少有一条" |  
  
**知识点 1：内存库与 StaticPool**  
  
定义：`sqlite://` 是进程内内存库；默认每连接独立，`StaticPool` 让所有连接共享同一份。  
  
定式（要背）：  
  
```python  
create_engine("sqlite://", connect_args={"check_same_thread": False}, poolclass=StaticPool)  
```  
  
适用：所有需要数据库的单元测试。例题：不加 `StaticPool` 时建表连接与查询连接是两个空库，报 `no such table`。  
  
**知识点 2：fixture**  
  
定义：fixture 是 pytest 注入到测试参数里的准备函数，`yield` 前准备、之后清理。  
  
定式（要背）：  
  
```python  
@pytest.fixture()  
def db_session():  
    ...  
    yield session  
    ...  
```  
  
适用：数据库会话、临时文件。例题：两个测试都声明该参数，各拿独立会话，互不影响。  
  
**知识点 3：测试隔离与依赖覆盖**  
  
定义：每个测试独立库、独立事务；接口测试用 `dependency_overrides` 替换真实依赖。  
  
定式（要背）：  
  
```python  
app.dependency_overrides[get_db] = override_get_db   # 仅测试期生效  
```  
  
适用：接口级测试。例题：覆盖后写接口，数据落内存库，真实库不变。  
  
### 5.4 指出缺陷  
  
以下缺陷只评"测试隔离"这一动作，均可定位到 5.2 代码。  
  
- **D1**：`create_engine("sqlite:///./data/insight.db")` 直接连开发库 → 测试写入清空或混入假数据，开发环境被污染。  
- **D2**：断言 `count() > 0` 只验证"至少一条" → 依赖库里已有数据，单独跑或顺序变化时报 `AssertionError: assert False`。  
- **D3**：使用文件库且无清理 → CI 两个 job 并发写同一文件，报 `sqlite3.OperationalError: database is locked`。  
- **D4**：建表与会话写在测试函数内 → 第二个测试文件要复制一遍，两处建表逻辑逐渐漂移。  
  
### 5.5 结构设计  
  
① 文件清单：  
  
| 文件路径 | 一句话职责 | 依赖谁 |  
|---|---|---|  
| `tests/conftest.py` | 建内存库、提供 `db_session` | `app.db.base`、`app.models` |  
| `tests/test_repository.py` | 仓储契约测试 | `conftest`、repositories、services |  
  
② 拆分理由：  
  
- `conftest.py`：解决 D1，内存库替代开发库；解决 D3，无文件锁；解决 D4，建表逻辑只一处。  
- `test_repository.py`：解决 D2，用显式数据与精确断言替代"库非空"。  
  
③ 不这么拆会怎样：夹具写在测试文件里，第二个文件复制一遍建表逻辑，两边漂移；所有用例一个文件，800 行后无法按模块定位。  
  
④ 依赖图：  
  
```  
app/db/base.py + app/models/  
        ↓  
tests/conftest.py (db_session)  
        ↓  
tests/test_repository.py  
```  
  
⑤ 分工：设计决策回答"真实库、复制库还是内存库"；结构设计回答"夹具与用例放哪两个文件"。  
  
### 5.6 完整版  
  
**设计决策：内存库 + StaticPool**  
  
① 痛点：D1 污染开发数据、D2 断言失效、D3 并发锁、D4 逻辑重复。  
  
② 取舍：  
  
- 方案 A：直连真实库。适用于上线前端到端验收。否决理由：每次运行改动真实数据，无法并行，失败后现场被破坏。  
- 方案 B：每测复制一份文件库。适用于验证文件锁行为。否决理由：复制删除有 IO 开销，200 个用例累计约 1.8 秒，还要清理临时文件。  
  
③ 触发条件：单元与仓储测试用内存库；验证并发锁或迁移时另建文件库慢测试。只有一个测试文件时可内联建库，不必抽夹具。  
  
**文件 1：`tests/conftest.py` 第 1 段**  
  
本段做什么：导入依赖并声明夹具。  
  
```python  
# 本段：夹具声明  
"""测试夹具：每个测试一个独立内存库。"""  
import pytest                                          # 第三方库 · 测试框架 · fixture  
from sqlalchemy import create_engine                   # 第三方库 · 引擎  
from sqlalchemy.orm import sessionmaker                # 第三方库 · 会话工厂  
from sqlalchemy.pool import StaticPool                 # 第三方库 · 内存库共享  
  
from app.db.base import Base                           # 本项目 · 基类  
from app.models import Chunk, Document                 # 本项目 · 模型 · 确保建表完整  
  
  
@pytest.fixture()  
def db_session():  
    """提供一个已建表、结束后自动销毁的内存会话。"""  
```  
  
| 代码 | 含义 |  
|---|---|  
| `@pytest.fixture()` | 声明夹具，默认函数级 |  
| `Chunk, Document` 导入 | 触发模型注册，保证建表完整 |  
  
**文件 1 第 2 段**  
  
本段做什么：建库建表，交出会话，结束后清理。  
  
```python  
# 本段：建库与清理  
    engine = create_engine(  
        "sqlite://",  
        connect_args={"check_same_thread": False},  
        poolclass=StaticPool,  
    )  
    Base.metadata.create_all(bind=engine)  
    TestSession = sessionmaker(bind=engine, expire_on_commit=False)  
    session = TestSession()  
    try:  
        yield session                                  # 关键行：交给测试使用  
    finally:  
        session.close()  
        engine.dispose()                               # 关键行：释放连接池  
```  
  
| 代码 | 含义 |  
|---|---|  
| `create_all(bind=engine)` | 在内存库建表 |  
| `sessionmaker(...)` | 测试专用会话工厂 |  
| `yield session` | 暂停并交权，测试后继续 |  
| `engine.dispose()` | 关闭连接池 |  
  
**文件 2：`tests/test_repository.py` 第 1 段**  
  
本段做什么：验证入库后能检索，断言精确。  
  
```python  
# 本段：正常路径  
"""仓储层契约测试。"""  
from app.repositories.chunk_repo import count_chunks, search_chunks   # 本项目 · 被测  
from app.repositories.document_repo import create_document            # 本项目 · 被测  
from app.services.ingest import save_chunks                           # 本项目 · 被测  
  
  
def test_save_then_search(db_session):  
    """模拟：上传文档后按关键词检索（正常路径）。  
    契约：写入 3 块，命中 2 块，总数为 2。"""  
    doc = create_document(db_session, "a.txt", "data/a.txt")  
    chunks = [  
        {"index": 0, "text": "部署使用 Docker", "char_count": 10},  
        {"index": 1, "text": "测试使用 pytest", "char_count": 10},  
        {"index": 2, "text": "部署脚本在 scripts", "char_count": 9},  
    ]  
    assert save_chunks(db_session, doc.id, chunks) == 3  
    assert len(search_chunks(db_session, "部署")) == 2  
    assert count_chunks(db_session, "部署") == 2  
```  
  
| 代码 | 含义 |  
|---|---|  
| `db_session` 参数 | 注入独立内存会话 |  
| `save_chunks(...) == 3` | 断言写入数量 |  
| `len(search_chunks(...)) == 2` | 断言当前页命中数 |  
| `count_chunks(...) == 2` | 断言总数 |  
  
**文件 2 第 2 段**  
  
本段做什么：验证重复导入与空输入。  
  
```python  
# 本段：重复与边界路径  
def test_reimport_is_idempotent(db_session):  
    """模拟：同一文件重复导入（重复路径）。  
    契约：块数不翻倍，仍为 3。"""  
    doc = create_document(db_session, "a.txt", "data/a.txt")  
    chunks = [{"index": i, "text": f"文本{i}", "char_count": 3} for i in range(3)]  
    save_chunks(db_session, doc.id, chunks)  
    save_chunks(db_session, doc.id, chunks)  
    assert count_chunks(db_session, "文本") == 3  
  
  
def test_empty_chunks_writes_nothing(db_session):  
    """模拟：空文档入库（边界路径）。  
    契约：返回 0，表中无记录。"""  
    doc = create_document(db_session, "empty.txt", "data/empty.txt")  
    assert save_chunks(db_session, doc.id, []) == 0  
    assert count_chunks(db_session, "") == 0  
```  
  
| 代码 | 含义 |  
|---|---|  
| 连续两次 `save_chunks` | 模拟重复上传 |  
| `count_chunks(...) == 3` | 断言未翻倍 |  
| `save_chunks(..., []) == 0` | 断言空输入短路 |  
  
**文件 2 第 3 段**  
  
本段做什么：验证删除文档会级联清块。  
  
```python  
# 本段：关联路径  
def test_delete_cascades_chunks(db_session):  
    """模拟：删除文档（关联路径）。  
    契约：其分块一并删除。"""  
    from app.models.chunk import Chunk  
    from app.repositories.document_repo import delete_document  
  
    doc = create_document(db_session, "a.txt", "data/a.txt")  
    save_chunks(db_session, doc.id, [{"index": 0, "text": "内容", "char_count": 2}])  
    assert delete_document(db_session, doc.id) is True  
    assert db_session.query(Chunk).filter_by(document_id=doc.id).count() == 0  
```  
  
| 代码 | 含义 |  
|---|---|  
| `delete_document(...) is True` | 断言删除成功 |  
| `query(Chunk).filter_by(...).count()` | 统计残留分块，应为 0 |  
  
固定模板：`conftest.py` 的夹具三段式。可组合写法：事务回滚隔离、种子数据、覆盖 `get_db`。  
  
已知边界：内存库与文件库锁行为不同；验证并发锁要另建文件库慢测试。  
  
调用链：`db_session` 由 pytest 按参数名注入；它调用 `create_engine`、`create_all`、`sessionmaker`；夹具内异常直接使该测试失败，不影响其他测试。  
  
### 5.7 结尾块  
  
**本章判据**：删掉 `data/insight.db` 后测试全绿，单测总耗时秒级。  
  
**本章产出**：`tests/conftest.py`、`tests/test_repository.py`。  
  
**下一章入口**：本册封版，下一册接检索结果。  
  
**常见错法 3 条**：  
  
1. 测试直连开发库。  
2. 内存库不加 `StaticPool`。  
3. 用例之间靠顺序传数据。  
  
**自检 3 问**：  
  
1. 单独跑一个用例能通过吗？  
2. 两个用例并行会互相影响吗？  
3. 测试失败后连接会释放吗？  
  
**下一册从哪接**：`app/services/search_service.py` 的 `search_page` 将作为第四册 Agent 的工具入口 `kb_search`。  
  
---  
  
## 运行与验收  
  
```bash  
pip install sqlalchemy pytest  
```  
  
文件清单：  
  
```  
app/db/base.py  
app/db/session.py  
app/models/__init__.py  
app/models/document.py  
app/models/chunk.py  
app/repositories/document_repo.py  
app/repositories/chunk_repo.py  
app/services/__init__.py  
app/services/ingest.py  
app/services/search_service.py  
tests/conftest.py  
tests/test_repository.py  
```  
  
验收：  
  
```bash  
python -c "  
from app.db.base import Base, engine  
import app.models  
Base.metadata.create_all(engine)  
print('tables created')  
"  
sqlite3 data/insight.db ".tables"  
pytest -v  
pytest --cov=app --cov-report=term-missing  
```  
  
预期：输出 `tables created`，`.tables` 显示 `chunks` 与 `documents`，测试全绿。  
  
---  
  
## 知识地图  
  
| 知识点 | 落点 |  
|---|---|  
| 建模与字段 | 1.2 |  
| ORM 映射 | 1.4 知识点 3 |  
| 外键与关系 | 1.4 知识点 4 |  
| 会话与事务 | 2.3 知识点 1、2 |  
| 查询 | 2.3 知识点 3、4；4.3 |  
| 批量与幂等 | 3.3 |  
| 分页与计数 | 4.3 知识点 2、3 |  
| 索引 | 4.3 知识点 4 |  
| 测试隔离 | 5.3 |  
  
## 自测  
  
1. 内存库测试为什么必须加 `StaticPool`？  
2. `commit()` 与 `flush()` 有什么区别？  
3. 批量入库为什么先删后插，而不是直接 `add_all`？  
4. 关键词检索为什么不能用 f-string 拼接？  
5. 删除 `Document` 时 `Chunk` 如何被一起删除？  
  
<details>  
<summary>答案要点</summary>  
  
1. 默认每连接独立内存库；`StaticPool` 让所有连接共享同一份，否则建表与查询看到两个库。  
2. `flush` 发语句不结束事务；`commit` 提交并结束事务，内部先 `flush`。  
3. 直接 `add_all` 会追加导致条数翻倍；先删后插在同一事务内覆盖，保证幂等。  
4. f-string 破坏参数绑定，引号导致语法错，`%` 放大匹配范围，并有注入风险。  
5. 数据库层 `ondelete="CASCADE"` 与 ORM 层 `cascade="all, delete-orphan"` 双保险，本册删除走 ORM 级联。  
  
</details>  
  
---  
  
## 规范自检记录  
  
| 条目 | 证据（小节号） | 结论 |  
|---|---|---|  
| 1 交付 | 仅本册，末行停止 | 满足 |  
| 2 方案四段 | 本册为教材 | 不适用 |  
| 3 八段结构 | 1.1–1.8；2.1–2.7 等（无数据结构章跳过 数据建模） | 满足 |  
| 4 场景与动机 | 各章 N.1，含本章做／不做与四问 | 满足 |  
| 5 数据建模五项 | 1.2 | 满足 |  
| 6 最小可运行版 | 1.3 含 Document 与 Chunk；2.2 起无省略 | 满足 |  
| 7 一章一动作与缺陷定位 | 1.5、2.4、3.4、4.4、5.4 | 满足 |  
| 8 结构设计 | 1.6、2.5、3.5、4.5、5.5，均引用真实 D 编号 | 满足 |  
| 9 知识点三件套 | 1.4、2.3、3.3、4.3、5.3 共 18 个 | 满足 |  
| 10 设计决策三段 | 1.7、2.6、3.6、4.6、5.6 | 满足 |  
| 11 代码与注释 | 各代码块 import 三标、类函数 docstring、分段导读 | 部分满足：知识点卡内 3～6 行定式模板未逐行注释，因模板要可直接抄写；正文代码均满足 |  
| 12 读者画像 | 全文 | 满足 |  
| 13 测试标注 | 5.6 每个测试 docstring | 满足 |  
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
  
- **C1**：完整版出现的对象只有 `Document`、`Chunk`，均在 1.2 出现。通过。  
- **C2**：D1–D5 均可指向 1.3 的类或列；D1–D4 的其余各章缺陷均可指向对应最小版代码或 4.2 反例块。通过。  
- **C3**：1.6 引用 D1–D5；2.5 引用 D1–D4；3.5 引用 D1–D4；4.5 引用 D1–D4；5.5 引用 D1–D4。全部真实存在。通过。  
- **C4**：1.5 全属建表；2.4 全属读写；3.4 全属批量；4.4 全属查询；5.4 全属测试隔离。无越界。通过。  
- **C5**：1.3 最小版仅 `Document`、`Chunk`，与 1.2 的名称、数量一致；其余章无新数据结构，已在 N.1 声明。通过。  
- **C6**：每条缺陷均以类名、表名或反例块定位。通过。  
  
本册结束，未开始第四册。  
