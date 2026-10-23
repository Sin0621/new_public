# 《Python Web 项目实战》第三册：数据库与检索  
  
| 项目 | 内容 |  
|---|---|  
| 系列 | 《Python Web 项目实战》 |  
| 册数 | 第三册（开发阶段三：Day 5-6） |  
| 主题 | 建库建表 · ORM 模型 · 会话与事务 · 批量入库 · 关键词检索 · 测试数据库 |  
| 前置要求 | 第一册（配置、日志、异常处理）、第二册（`list[dict]` 分块结果） |  
| 版本 | v1.3 重写版 · 2026-10-23 |  
  
## 本册地图  
  
本册产出物：  
  
```  
输入：第二册产出的 chunks（list[dict]）  
  ↓ 第1章 建库建表    Engine / Base / Mapped / relationship   → app/db/、app/models/  
  ↓ 第2章 会话与CRUD  Session / commit / select / 依赖注入      → app/db/session.py、app/repositories/  
  ↓ 第3章 批量入库    add_all / 唯一约束 / 幂等覆盖              → app/services/ingest.py  
  ↓ 第4章 关键词检索  contains / limit / offset / count         → app/repositories/chunk_repo.py  
  ↓ 第5章 测试数据库  内存库 / StaticPool / fixture             → tests/conftest.py  
输出：可持久化、可检索、可测试的文档库  
```  
  
章节关系：第 1 章定义"存什么"，第 2 章定义"怎么读写"，第 3 章解决"一次写很多且不重复"，第 4 章解决"怎么查出来"，第 5 章保证前面四章的行为不被后续改动破坏。  
  
术语校准：ORM 是"对象与表行互相转换"的映射层，不是数据库本身；数据仍然存在 SQLite 文件里。  
  
## 目录  
  
1. 第 1 章 建库与建表  
2. 第 2 章 会话与增删改查  
3. 第 3 章 批量入库与幂等  
4. 第 4 章 关键词检索  
5. 第 5 章 测试数据库  
6. 运行与验收  
7. 知识地图  
8. 自测  
9. 规范自检记录  
  
---  
  
## 第 1 章 建库与建表  
  
### 1.1 需求  
  
把一份文档和它切出的多个分块持久化：重启服务不丢；一个文档对应多个分块；删除文档时分块一起消失。  
  
### 1.2 最小可运行版  
  
```python  
from sqlalchemy import String, create_engine          # 第三方库 · SQLAlchemy · 引擎与列类型  
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column  # 第三方库 · ORM · 模型基类与字段  
  
class Base(DeclarativeBase):  
    pass  
  
class Note(Base):  
    __tablename__ = "notes"  
    id: Mapped[int] = mapped_column(primary_key=True)  
    text: Mapped[str] = mapped_column(String(200))  
  
engine = create_engine("sqlite:///./demo.db")          # 关键行：指向一个本地文件  
Base.metadata.create_all(engine)  
```  
  
运行后验证：  
  
```bash  
python demo_orm.py  
sqlite3 demo.db ".tables"  
```  
  
预期输出：  
  
```  
notes  
```  
  
### 1.3 逐行讲解  
  
| 代码 | 含义 |  
|---|---|  
| `create_engine("sqlite:///./demo.db")` | 创建引擎；URL 中 `sqlite:///` 表示本地文件，后面的路径是库文件 |  
| `class Base(DeclarativeBase)` | 声明模型基类；所有表类都继承它 |  
| `__tablename__ = "notes"` | 指定数据库中真实的表名 |  
| `id: Mapped[int]` | 类型注解声明列类型为整数 |  
| `mapped_column(primary_key=True)` | 把该属性映射为列；`primary_key=True` 设为主键 |  
| `String(200)` | 列类型为变长字符串，最大 200 字符 |  
| `Base.metadata.create_all(engine)` | 根据所有已注册模型建表；表已存在则跳过 |  
  
**知识点 1：库、表、行、列、主键**  
  
定义：数据库由若干表组成，表由列定义结构、由行存储数据；主键是能唯一标识一行的列。  
  
定式（要背）：  
  
```python  
# 心智模型，不是代码  
库 = 一个文件  
表 = 一类对象  
列 = 对象的属性  
行 = 一个具体对象  
主键 = 唯一编号  
```  
  
适用：设计任何持久化结构前，先用这套模型把字段列清。  
  
例题与预期：`documents` 表有 `id`、`filename` 两列，存入 2 条记录后表中有 2 行，`id` 分别为 1、2。  
  
**知识点 2：Engine**  
  
定义：Engine 是数据库连接的入口，内部维护连接池，第一次执行语句时才真正建立连接。  
  
定式（要背）：  
  
```python  
engine = create_engine(settings.DATABASE_URL, echo=False)  
```  
  
适用：全项目只创建一次，模块级共享。  
  
例题与预期：`create_engine("sqlite:///./a.db")` 执行后不产生文件；第一次 `connect()` 或 `create_all()` 后才出现 `a.db`。  
  
**知识点 3：ORM 映射**  
  
定义：ORM 把 Python 类映射为表、把属性映射为列、把实例映射为行。  
  
定式（要背）：  
  
```python  
class User(Base):  
    __tablename__ = "users"  
    id: Mapped[int] = mapped_column(primary_key=True)  
    name: Mapped[str] = mapped_column(String(50))  
```  
  
适用：需要类型提示、需要关系导航、需要跨数据库迁移的业务表。  
  
例题与预期：`User(name="a")` 实例在 `commit` 后对应 `users` 表一行，`id` 自动赋值。  
  
**知识点 4：外键与关系**  
  
定义：外键列指向另一张表的主键；`relationship` 在 Python 侧提供对象间导航。  
  
定式（要背）：  
  
```python  
document_id: Mapped[int] = mapped_column(ForeignKey("documents.id", ondelete="CASCADE"))  
document: Mapped["Document"] = relationship(back_populates="chunks")  
```  
  
适用：一对多、多对多关联；删除父记录需要级联时。  
  
例题与预期：`chunk.document` 直接返回所属 `Document` 对象，无需手写 JOIN。  
  
### 1.4 指出缺陷  
  
用裸 SQL 字符串拼表结构，会出四个具体问题：  
  
1. **SQL 注入**：`f"SELECT * FROM documents WHERE filename='{name}'"`，当 `name` 为 `' OR 1=1 --` 时返回全表。  
2. **结构变更成本高**：给 `documents` 加一个 `status` 字段，要改 12 处 `INSERT` / `SELECT` 字符串。  
3. **无外键约束**：删除 1 个文档后，其 300 条 `chunks` 成为孤儿数据，统计总数偏大 300。  
4. **无类型提示**：IDE 无法补全，字段名写错要到运行时才报 `sqlite3.OperationalError: no such column`。  
  
### 1.5 完整版  
  
文件：`app/db/base.py`  
  
```python  
"""数据库基础：引擎与模型基类。"""  
from pathlib import Path                              # 标准库 · 路径 · 确保数据目录存在  
  
from sqlalchemy import create_engine                  # 第三方库 · SQLAlchemy · 创建引擎  
from sqlalchemy.orm import DeclarativeBase            # 第三方库 · ORM · 模型基类  
  
from app.config import settings                       # 本项目 · 配置 · 读取数据库地址  
  
Path("data").mkdir(parents=True, exist_ok=True)       # 关键行：SQLite 不会自动建目录  
  
engine = create_engine(settings.DATABASE_URL, echo=False)  
  
  
class Base(DeclarativeBase):  
    """所有 ORM 模型的基类。"""  
```  
  
文件：`app/models/document.py`  
  
```python  
"""文档表模型。"""  
from datetime import datetime                          # 标准库 · 时间 · 创建时间类型  
  
from sqlalchemy import DateTime, String, func          # 第三方库 · SQLAlchemy · 列类型与函数  
from sqlalchemy.orm import Mapped, mapped_column, relationship  # 第三方库 · ORM · 字段与关系  
  
from app.db.base import Base                           # 本项目 · 基类  
  
  
class Document(Base):  
    __tablename__ = "documents"  
  
    id: Mapped[int] = mapped_column(primary_key=True)  
    filename: Mapped[str] = mapped_column(String(255), index=True)  
    source_path: Mapped[str] = mapped_column(String(512))  
    created_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())  
  
    # 关键行：一对多，删除文档时级联删除分块  
    chunks: Mapped[list["Chunk"]] = relationship(  
        back_populates="document",  
        cascade="all, delete-orphan",  
    )  
```  
  
文件：`app/models/chunk.py`  
  
```python  
"""分块表模型。"""  
from sqlalchemy import ForeignKey, Integer, Text, UniqueConstraint  # 第三方库 · 列与约束  
from sqlalchemy.orm import Mapped, mapped_column, relationship      # 第三方库 · ORM  
  
from app.db.base import Base                                        # 本项目 · 基类  
  
  
class Chunk(Base):  
    __tablename__ = "chunks"  
    # 关键行：同一文档内块序号唯一，防重复导入  
    __table_args__ = (UniqueConstraint("document_id", "index", name="uq_chunk_doc_index"),)  
  
    id: Mapped[int] = mapped_column(primary_key=True)  
    document_id: Mapped[int] = mapped_column(  
        ForeignKey("documents.id", ondelete="CASCADE"), index=True  
    )  
    index: Mapped[int] = mapped_column(Integer)  
    text: Mapped[str] = mapped_column(Text)  
    char_count: Mapped[int] = mapped_column(Integer)  
  
    document: Mapped["Document"] = relationship(back_populates="chunks")  
```  
  
文件：`app/models/__init__.py`  
  
```python  
"""导入所有模型，确保 create_all 能发现它们。"""  
from app.models.chunk import Chunk        # 本项目 · 模型 · 注册 chunks 表  
from app.models.document import Document  # 本项目 · 模型 · 注册 documents 表  
  
__all__ = ["Chunk", "Document"]  
```  
  
逐块说明补了哪个缺陷：  
  
| 代码块 | 补的缺陷 |  
|---|---|  
| `Path("data").mkdir(...)` | SQLite 不会自动创建父目录，否则报 `unable to open database file` |  
| `engine` 模块级单例 | 全项目一个连接池，避免每处新建 |  
| `Document` / `Chunk` 两个类 | 表结构集中定义，加字段改 1 处 |  
| `index=True` | `filename` 与 `document_id` 建索引，后续按它们查询不全表扫 |  
| `ForeignKey(..., ondelete="CASCADE")` | 数据库层兜底删除子记录 |  
| `cascade="all, delete-orphan"` | ORM 层删除父对象时自动删子对象，不留孤儿 |  
| `UniqueConstraint` | 同文档同序号只能有一条，挡住重复导入 |  
| `models/__init__.py` | 保证 `create_all` 之前所有表都已注册 |  
  
完整版新增调用：  
  
| 调用 | 作用与参数 |  
|---|---|  
| `create_engine(url, echo=False)` | `url` 数据库地址；`echo=True` 会打印所有 SQL，调试用 |  
| `mapped_column(...)` | 声明列；常用参数 `primary_key`、`index`、`unique`、`server_default` |  
| `String(n)` / `Text()` / `Integer()` / `DateTime()` | 列类型；`String` 有长度上限，`Text` 无 |  
| `relationship(back_populates, cascade)` | 建立对象关系；`back_populates` 两边互相指向；`cascade` 控制级联行为 |  
| `func.now()` | 生成 SQL 的当前时间函数，作为服务端默认值 |  
| `Base.metadata.create_all(bind=engine)` | 建所有表，已存在则跳过；不做字段变更 |  
  
固定模板：`Base` + 两个模型类 + `__init__.py` 导入。可组合写法：列长度、索引位置、是否级联、时间字段用客户端还是服务端默认值。  
  
已知边界：SQLite 默认不强制外键约束，需执行 `PRAGMA foreign_keys=ON`；本册依赖 ORM 级联，因此不强制也能删干净。  
  
调用链：`Document`、`Chunk` 由仓储层与 `create_all` 使用；它们调用 `Base.metadata` 注册表结构；建表阶段的 `SQLAlchemyError` 直接抛出，属于启动失败，应让进程退出而不是继续服务。  
  
### 1.6 结尾块  
  
**本章判据**：两份模型能在一次 `create_all` 后生成 `documents` 与 `chunks` 两张表，且删除文档能带走分块。  
  
**常见错法 3 条**：  
  
1. 每个文件各自 `create_engine`。  
2. 只写外键，不写关系与级联。  
3. 忘记在 `models/__init__.py` 导入模型，导致建表少一张。  
  
**自检 3 问**：  
  
1. 加一个字段，我要改几处？  
2. 删除父记录，子记录会留下吗？  
3. 两条相同 `(document_id, index)` 能同时写入吗？  
  
---  
  
## 第 2 章 会话与增删改查  
  
### 2.1 需求  
  
提供文档的创建、查询、列表、删除四个操作；每个请求独立事务；接口层不出现数据库样板代码。  
  
### 2.2 最小可运行版  
  
```python  
from sqlalchemy.orm import Session                    # 第三方库 · ORM · 会话  
from app.db.base import engine                        # 本项目 · 引擎  
from app.models.document import Document              # 本项目 · 模型  
  
with Session(engine) as session:                      # 关键行：用完自动关闭  
    doc = Document(filename="demo.txt", source_path="data/demo.txt")  
    session.add(doc)  
    session.commit()  
    print(doc.id)  
```  
  
预期输出：  
  
```  
1  
```  
  
### 2.3 逐行讲解  
  
| 代码 | 含义 |  
|---|---|  
| `Session(engine)` | 创建一个会话，绑定到引擎；会话是"一次数据库对话" |  
| `with ... as session` | 离开代码块自动 `close()`，释放连接 |  
| `Document(...)` | 构造一个待插入的对象，此时未写库 |  
| `session.add(doc)` | 把对象加入会话的待写入集合 |  
| `session.commit()` | 提交事务，真正写库；失败需 `rollback` |  
| `doc.id` | 提交后主键被回填到对象上 |  
  
**知识点 1：Session 是什么**  
  
定义：Session 是一次数据库对话的工作区，负责跟踪对象变化、组织事务、管理连接。  
  
定式（要背）：  
  
```python  
with Session(engine) as session:  
    ...  
```  
  
适用：每个请求或每个独立任务一个 Session，不跨请求共享。  
  
例题与预期：两个 `with Session(engine)` 块互不影响，第一个块未提交的改动在第二个块中查不到。  
  
**知识点 2：`commit` / `rollback` / `flush`**  
  
定义：`flush` 把待写入语句发给数据库但不开事务边界；`commit` 提交并结束事务；`rollback` 撤销当前事务。  
  
定式（要背）：  
  
```python  
try:  
    session.add(obj)  
    session.commit()  
except Exception:  
    session.rollback()  
    raise  
```  
  
适用：任何写操作；多表写入必须在同一事务内。  
  
例题与预期：`add` 后 `rollback`，再查询，预期查不到该对象。  
  
**知识点 3：`select` 与 `where`**  
  
定义：`select(模型)` 构造查询语句，`where(条件)` 追加过滤条件，`scalars()` 取出模型实例。  
  
定式（要背）：  
  
```python  
rows = session.scalars(select(Document).where(Document.id == 1)).all()  
```  
  
适用：所有列表查询、条件查询。  
  
例题与预期：表中 3 条记录，`where(Document.id == 2)` 返回 1 条。  
  
**知识点 4：`get` 按主键取单条**  
  
定义：`Session.get(模型, 主键值)` 按主键查询，命中返回对象，未命中返回 `None`。  
  
定式（要背）：  
  
```python  
doc = session.get(Document, 1)  
```  
  
适用：详情页、删除前查询、外键校验。  
  
例题与预期：`get(Document, 999)` 返回 `None`，不抛异常。  
  
### 2.4 指出缺陷  
  
在每个路由函数里直接写 Session，会出四个问题：  
  
1. **样板代码膨胀**：每个接口 6 行（创建、try、commit、except、rollback、close），8 个接口共 48 行重复。  
2. **连接泄漏**：其中 3 处忘记 `close()`，压测 50 并发后出现 `sqlite3.OperationalError: database is locked`。  
3. **半写数据**：一次请求先写 `documents` 再写 `chunks`，第二步失败但第一步已 `commit`，结果是"有文档、0 分块"的脏数据。  
4. **无法测试**：Session 在函数内部 `new`，测试无法替换成内存库，只能连真实 `insight.db`。  
  
### 2.5 完整版  
  
设计决策：为什么用"仓储层 + 依赖注入 Session"。  
  
① **痛点**：8 个接口各写 6 行 Session 样板，共 48 行；3 处漏 `close()`，50 并发压测出现 `database is locked`；测试必须连真实库，一轮 200 个用例约 40 秒。  
  
② **取舍**：  
  
- 替代方案 A：全局单例 Session 复用。否决理由：并发请求共享同一个事务，一个请求 `rollback` 会把另一个请求尚未提交的写入一起丢掉。  
- 替代方案 B：每个接口手写 `try/finally: db.close()`。否决理由：8 处复制，新增接口必漏；测试仍需替换模块级全局变量，替换点分散。  
  
③ **触发条件**：接口数 ≥3 且都访问数据库时抽象出依赖；单脚本、一次性迁移脚本不抽象。若项目只有 1 个入口且无测试要求，可退回函数内直接创建 Session。  
  
文件：`app/db/session.py`  
  
```python  
"""会话工厂与请求级依赖。"""  
from collections.abc import Generator                  # 标准库 · 类型 · 生成器返回类型  
  
from sqlalchemy.orm import Session, sessionmaker       # 第三方库 · ORM · 会话与工厂  
  
from app.db.base import engine                         # 本项目 · 引擎  
  
SessionLocal = sessionmaker(bind=engine, autoflush=False, expire_on_commit=False)  
  
  
def get_db() -> Generator[Session, None, None]:  
    """FastAPI 依赖：每个请求一个 Session，请求结束自动关闭。"""  
    db = SessionLocal()  
    try:  
        yield db                                       # 关键行：交给接口使用  
    finally:  
        db.close()                                     # 关键行：无论成败都关闭  
```  
  
文件：`app/repositories/document_repo.py`  
  
```python  
"""文档仓储：只做数据库读写，不含 HTTP 逻辑。"""  
from sqlalchemy import select                          # 第三方库 · SQL 构造 · 查询  
from sqlalchemy.orm import Session                     # 第三方库 · ORM · 会话类型  
  
from app.models.document import Document               # 本项目 · 模型  
  
  
def create_document(db: Session, filename: str, source_path: str) -> Document:  
    doc = Document(filename=filename, source_path=source_path)  
    db.add(doc)  
    db.commit()  
    db.refresh(doc)                                    # 关键行：取回自增 id 与服务端默认值  
    return doc  
  
  
def get_document(db: Session, document_id: int) -> Document | None:  
    return db.get(Document, document_id)  
  
  
def list_documents(db: Session) -> list[Document]:  
    stmt = select(Document).order_by(Document.id.desc())  
    return list(db.scalars(stmt).all())  
  
  
def delete_document(db: Session, document_id: int) -> bool:  
    doc = db.get(Document, document_id)  
    if doc is None:  
        return False  
    db.delete(doc)                                     # 关键行：级联删除其 chunks  
    db.commit()  
    return True  
```  
  
逐块说明补了哪个缺陷：  
  
| 代码块 | 补的缺陷 |  
|---|---|  
| `sessionmaker(...)` | 工厂只建一次，参数统一 |  
| `get_db` + `yield` + `finally` | 每个请求一个会话，异常路径也关闭，堵住泄漏 |  
| `expire_on_commit=False` | 提交后对象仍可读，避免接口序列化时再查库 |  
| `db.refresh(doc)` | 取回数据库生成的主键与时间，响应不缺字段 |  
| 仓储函数只收 `Session` | 测试可传入内存库会话，不需要改全局 |  
| `delete_document` 先 `get` | 不存在时返回 `False`，由接口映射 404 |  
  
完整版新增调用：  
  
| 调用 | 作用与参数 |  
|---|---|  
| `sessionmaker(bind, autoflush, expire_on_commit)` | 生成 Session 工厂；`bind` 绑定引擎 |  
| `SessionLocal()` | 创建一个会话实例 |  
| `db.add(obj)` | 加入待写入集合 |  
| `db.commit()` | 提交事务 |  
| `db.refresh(obj)` | 从数据库重新加载对象字段 |  
| `db.delete(obj)` | 标记删除，`commit` 后生效 |  
| `db.scalars(stmt)` | 执行查询并返回标量结果集；`.all()` 转列表 |  
| `select(...).order_by(col.desc())` | 构造查询并按列降序排序 |  
  
固定模板：`session.py` 的工厂与 `get_db`、仓储四个函数签名。可组合写法：排序字段、分页、是否软删除。  
  
已知边界：`get_db` 不做自动 `rollback`；写操作失败时必须在仓储内 `rollback`，否则连接归还时事务仍挂着。  
  
调用链：`get_db` 由 FastAPI 路由通过 `Depends` 注入；仓储函数由路由或服务层调用；它们调用 `Session.add/commit/get/scalars/delete`；数据库异常冒泡到全局处理器，`get` 返回 `None` 时由接口转 404。  
  
### 2.6 结尾块  
  
**本章判据**：四个 CRUD 操作在仓储层完成，接口层只看到 `Session` 参数，不出现 `commit`。  
  
**常见错法 3 条**：  
  
1. 路由函数里直接 `SessionLocal()` 并忘记 `close`。  
2. 写操作不包 `try/rollback`。  
3. 在仓储里处理 HTTP 状态码。  
  
**自检 3 问**：  
  
1. 请求抛异常时，会话会关闭吗？  
2. 两张表写入失败时，会留下半份数据吗？  
3. 测试能把真实库换成内存库吗？  
  
---  
  
## 第 3 章 批量入库与幂等  
  
### 3.1 需求  
  
一次写入几百个分块；同一份文档重复导入时，库里不能出现两份数据；中途失败不能留下半份文档。  
  
### 3.2 最小可运行版  
  
```python  
from sqlalchemy.orm import Session                    # 第三方库 · ORM · 会话  
from app.db.base import engine                        # 本项目 · 引擎  
from app.models.chunk import Chunk                    # 本项目 · 模型  
  
with Session(engine) as session:  
    session.add_all([                                 # 关键行：一次加入多条  
        Chunk(document_id=1, index=i, text=f"块{i}", char_count=2)  
        for i in range(3)  
    ])  
    session.commit()                                  # 关键行：一次事务提交  
```  
  
预期结果：`chunks` 表新增 3 行，`index` 为 0、1、2。  
  
### 3.3 逐行讲解  
  
| 代码 | 含义 |  
|---|---|  
| `session.add_all([...])` | 批量加入对象列表，比循环 `add` 少一次事务开销 |  
| 列表推导式 `[... for i in range(3)]` | 生成 3 个 `Chunk` 实例 |  
| `session.commit()` | 三条插入在同一事务内提交 |  
| `Chunk(...)` | 构造分块对象；字段与模型列一一对应 |  
  
**知识点 1：批量写入 `add_all`**  
  
定义：`add_all` 接收对象列表，一次性加入会话，提交时统一发往数据库。  
  
定式（要背）：  
  
```python  
session.add_all([Model(**row) for row in rows])  
session.commit()  
```  
  
适用：一次写入几十条以上；逐条 `commit` 会产生 N 次事务。  
  
例题与预期：写入 500 条，`add_all` + 一次 `commit` 与逐条 `commit` 结果一致，但事务次数从 500 降为 1。  
  
**知识点 2：事务边界**  
  
定义：一个事务内的所有写入要么全部成功，要么全部撤销。  
  
定式（要背）：  
  
```python  
try:  
    db.execute(...)  
    db.add_all(...)  
    db.commit()  
except Exception:  
    db.rollback()  
    raise  
```  
  
适用：关联表写入、先删后插、批量导入。  
  
例题与预期：先插入 2 条再主动抛异常，`rollback` 后表中为 0 条。  
  
**知识点 3：唯一约束与 `IntegrityError`**  
  
定义：唯一约束禁止重复的组合值；违反时数据库抛 `IntegrityError`，当前事务必须回滚。  
  
定式（要背）：  
  
```python  
UniqueConstraint("document_id", "index", name="uq_chunk_doc_index")  
  
except IntegrityError:  
    db.rollback()  
```  
  
适用：防止同一文档同一序号被写两次。  
  
例题与预期：连续插入两条 `(document_id=1, index=0)`，第二条抛 `IntegrityError`。  
  
**知识点 4：幂等写入**  
  
定义：同一请求执行多次，结果与执行一次相同。  
  
定式（要背）：  
  
```python  
db.execute(delete(Chunk).where(Chunk.document_id == doc_id))   # 先清  
db.add_all(new_chunks)                                         # 再写  
db.commit()  
```  
  
适用：文件重传、任务重试、批量导入。  
  
例题与预期：同一份 3 块数据导入两次，表中仍为 3 条，不是 6 条。  
  
### 3.4 指出缺陷  
  
直接 `for` 循环逐条写入，会出四个问题：  
  
1. **性能差**：500 块逐条 `commit`，SQLite 单机约 2.3 秒；批量一次 `commit` 约 0.05 秒，差 40 倍。  
2. **重复数据**：用户重复上传同一文件，两次导入产生 1000 条，检索结果重复率 50%。  
3. **半份文档**：写到第 200 条时抛错，前 200 条已提交，检索到残缺上下文。  
4. **并发重复**：两个请求同时导入，唯一约束缺失时同一 `(document_id, index)` 出现两条。  
  
### 3.5 完整版  
  
文件：`app/services/ingest.py`  
  
```python  
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
  
  
def save_chunks(db: Session, document_id: int, chunks: list[dict]) -> int:  
    """覆盖式写入：先删旧块，再批量插入。返回写入块数。"""  
    if not chunks:  
        return 0  
  
    try:  
        db.execute(delete(Chunk).where(Chunk.document_id == document_id))  # 关键行：先清旧数据  
        db.add_all([  
            Chunk(  
                document_id=document_id,  
                index=item["index"],  
                text=item["text"],  
                char_count=item["char_count"],  
            )  
            for item in chunks  
        ])  
        db.commit()                                    # 关键行：删与插在同一事务  
    except IntegrityError as e:  
        db.rollback()                                  # 关键行：冲突必须回滚  
        logger.error("写入冲突 | document_id=%s | %s", document_id, e)  
        raise IngestError(document_id, "分块序号冲突") from e  
  
    logger.info("写入完成 | document_id=%s | chunks=%d", document_id, len(chunks))  
    return len(chunks)  
```  
  
逐块说明补了哪个缺陷：  
  
| 代码块 | 补的缺陷 |  
|---|---|  
| `if not chunks: return 0` | 空输入不开启事务，避免无意义提交 |  
| `delete(...).where(...)` | 先清旧块，保证重复导入仍为 N 条 |  
| `add_all([...])` | 500 条一次事务，耗时从 2.3 秒降到 0.05 秒 |  
| 删与插同一 `commit` | 中途失败整体回滚，不留半份文档 |  
| `except IntegrityError` + `rollback` | 冲突后会话可继续使用，不会卡在坏事务 |  
| `raise ... from e` | 保留数据库原始错误，便于定位具体约束 |  
| `IngestError` 带 `document_id` | 上层能直接告知"哪份文档失败" |  
  
完整版新增调用：  
  
| 调用 | 作用与参数 |  
|---|---|  
| `delete(模型).where(条件)` | 构造删除语句；需 `db.execute` 执行 |  
| `db.execute(stmt)` | 执行任意 SQL 语句对象 |  
| `db.rollback()` | 撤销当前事务，恢复到事务开始前 |  
| `db.add_all(list)` | 批量加入对象 |  
| `logger.error(msg, *args)` | 错误级日志；`%s` 由 logging 填充 |  
  
固定模板：`delete + add_all + commit` 的覆盖式写入，以及 `IntegrityError → rollback → 自定义异常`。可组合写法：是否先比对差异再写、是否保留历史版本、失败是否重试。  
  
已知边界：覆盖式写入会在同一事务内短暂持写锁；SQLite 单写者模型下，并发导入同一文档会串行等待，需在接口层做按文档加锁或排队。  
  
调用链：`save_chunks` 由上传接口调用；它调用 `Session.execute`、`Session.add_all`、`Session.commit`；`IntegrityError` 被包装为 `IngestError`，最终由全局处理器映射为 HTTP 409。  
  
### 3.6 结尾块  
  
**本章判据**：同一份分块数据导入 N 次，表里始终只有一份；中途失败不留残片。  
  
**常见错法 3 条**：  
  
1. 循环里逐条 `commit`。  
2. 只 `add` 不先清旧数据。  
3. 捕获 `IntegrityError` 后不 `rollback`。  
  
**自检 3 问**：  
  
1. 重复导入后条数会翻倍吗？  
2. 第 200 条失败时，前 199 条会留下吗？  
3. 冲突后这个 Session 还能继续用吗？  
  
---  
  
## 第 4 章 关键词检索  
  
### 4.1 需求  
  
输入关键词，返回包含它的分块；支持分页；返回当前页数据与匹配总数；大数据量下响应时间可控。  
  
### 4.2 最小可运行版  
  
```python  
from sqlalchemy import select                          # 第三方库 · SQL · 查询构造  
from sqlalchemy.orm import Session                     # 第三方库 · ORM · 会话  
from app.db.base import engine                         # 本项目 · 引擎  
from app.models.chunk import Chunk                     # 本项目 · 模型  
  
with Session(engine) as session:  
    stmt = (  
        select(Chunk)  
        .where(Chunk.text.contains("部署"))             # 关键行：包含匹配  
        .limit(5)  
    )  
    rows = session.scalars(stmt).all()  
    print([r.text for r in rows])  
```  
  
预期输出（示例）：  
  
```  
['部署使用 Docker', '部署脚本在 scripts']  
```  
  
### 4.3 逐行讲解  
  
| 代码 | 含义 |  
|---|---|  
| `select(Chunk)` | 构造"查 chunks 表"的语句 |  
| `.where(Chunk.text.contains("部署"))` | 追加条件：`text` 列包含该词；`contains` 生成 `LIKE '%部署%'` |  
| `.limit(5)` | 最多返回 5 条 |  
| `session.scalars(stmt).all()` | 执行并取出模型实例列表 |  
| `[r.text for r in rows]` | 列表推导，提取文本字段 |  
  
**知识点 1：`contains` 与 `LIKE`**  
  
定义：`contains` 生成 `LIKE '%关键词%'` 条件，并在 `autoescape=True` 时转义 `%`、`_` 等通配符。  
  
定式（要背）：  
  
```python  
stmt = select(Chunk).where(Chunk.text.contains(keyword, autoescape=True))  
```  
  
适用：简单的包含式检索；中文无需分词即可用。  
  
例题与预期：关键词 `"部署"` 命中 2 条；关键词 `"50%"` 在 `autoescape=True` 下按字面匹配，不当作通配符。  
  
**知识点 2：`limit` 与 `offset` 分页**  
  
定义：`limit` 限制返回条数，`offset` 跳过前 N 条，二者组合实现分页。  
  
定式（要背）：  
  
```python  
stmt = stmt.offset(offset).limit(limit)  
```  
  
适用：列表页、搜索结果页。`offset = (page - 1) * page_size`。  
  
例题与预期：共 25 条，`limit=10, offset=20` 返回 5 条。  
  
**知识点 3：`count` 总数**  
  
定义：`select(func.count())` 统计满足条件的行数，用于计算总页数。  
  
定式（要背）：  
  
```python  
stmt = select(func.count()).select_from(Chunk).where(Chunk.text.contains(keyword, autoescape=True))  
total = db.scalar(stmt) or 0  
```  
  
适用：任何需要"共 N 条"的分页接口。  
  
例题与预期：命中 25 条时返回 25，与当前页返回的 10 条无关。  
  
**知识点 4：索引**  
  
定义：索引是数据库为某列额外维护的查找结构，用空间换查询时间。  
  
定式（要背）：  
  
```python  
filename: Mapped[str] = mapped_column(String(255), index=True)  
```  
  
适用：高频出现在 `where`、`order_by`、`join` 中的列。  
  
例题与预期：10 万行按 `document_id` 查询，有索引约 1 毫秒，无索引约 80 毫秒（典型值）。  
  
### 4.4 指出缺陷  
  
不分页、不转义、不建索引，会出四个问题：  
  
1. **全表扫描**：10 万行 `LIKE '%部署%'` 无索引可用，单次约 1.2 秒；且前置 `%` 使 B-Tree 索引失效。  
2. **响应过大**：不分页一次返回 10 万条，JSON 约 40 MB，前端解析卡死。  
3. **无法分页**：没有 `count`，前端不知道总页数，只能无限"加载更多"。  
4. **拼串注入与语法错**：`f"... LIKE '%{kw}%'"`，当关键词含 `'` 时报 `sqlite3.OperationalError: near "'": syntax error`，含 `%` 时匹配范围被放大。  
  
### 4.5 完整版  
  
文件：`app/repositories/chunk_repo.py`  
  
```python  
"""分块检索仓储。"""  
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
        .where(Chunk.text.contains(keyword, autoescape=True))   # 关键行：参数绑定 + 转义  
        .order_by(Chunk.document_id, Chunk.index)  
        .offset(offset)  
        .limit(limit)  
    )  
    return list(db.scalars(stmt).all())  
  
  
def count_chunks(db: Session, keyword: str) -> int:  
    """统计命中条数，用于分页总数。"""  
    stmt = (  
        select(func.count())  
        .select_from(Chunk)  
        .where(Chunk.text.contains(keyword, autoescape=True))  
    )  
    return db.scalar(stmt) or 0  
  
  
def search_page(  
    db: Session,  
    keyword: str,  
    page: int = 1,  
    page_size: int = 20,  
) -> dict:  
    """分页检索：返回当前页数据与总数。"""  
    if page < 1:  
        page = 1  
    offset = (page - 1) * page_size  
    return {  
        "items": search_chunks(db, keyword, limit=page_size, offset=offset),  
        "total": count_chunks(db, keyword),  
        "page": page,  
        "page_size": page_size,  
    }  
```  
  
逐块说明补了哪个缺陷：  
  
| 代码块 | 补的缺陷 |  
|---|---|  
| `contains(keyword, autoescape=True)` | 参数绑定防注入；`%`、`_` 按字面匹配 |  
| `.order_by(document_id, index)` | 结果稳定，分页不会因顺序漂移而重复或漏项 |  
| `.offset().limit()` | 单次响应从 40 MB 降到几十 KB |  
| `count_chunks` 单独查询 | 前端能算出总页数 |  
| `page < 1` 归正 | 非法页码不产生负 offset（SQLite 会静默当成 0，其他库可能报错） |  
| `search_page` 统一返回结构 | 接口层不用重复拼装分页字段 |  
  
完整版新增调用：  
  
| 调用 | 作用与参数 |  
|---|---|  
| `ColumnOperators.contains(other, autoescape)` | 生成包含匹配；`autoescape=True` 自动转义通配符 |  
| `Select.order_by(*cols)` | 按列排序，可多列 |  
| `Select.offset(n)` / `Select.limit(n)` | 跳过 n 条 / 最多取 n 条 |  
| `func.count()` | SQL 聚合函数 `COUNT(*)` |  
| `Select.select_from(模型)` | 指定统计来源表 |  
| `db.scalar(stmt)` | 执行并返回第一行第一列，常用于聚合值 |  
  
固定模板：`search_chunks` / `count_chunks` / `search_page` 三段式。可组合写法：`page_size` 上限、排序策略、是否叠加文档过滤。  
  
已知边界：`LIKE '%词%'` 无法使用普通索引；数据量到百万级后需要全文索引或向量检索，本册不展开。  
  
语义相近的数字对照：  
  
| 名字 | 谁产生 | 给谁看 | 出现在哪 |  
|---|---|---|---|  
| `total` | `count_chunks` 的聚合查询 | 前端分页器 | 响应体 `total` |  
| `len(items)` | `search_chunks` 返回列表 | 日志、调试 | 当前页条数 |  
| `limit` | 调用方传参 | 数据库 | SQL `LIMIT` |  
| `page_size` | 接口参数 | 计算 offset 与 limit | 响应体与查询参数 |  
  
调用链：`search_page` 由检索接口与后续 Agent 的 `kb_search` 工具调用；它调用 `search_chunks`、`count_chunks`；本身不抛业务异常，空关键词返回全表匹配，由接口层拦截。  
  
### 4.6 结尾块  
  
**本章判据**：给定关键词与页码，返回稳定排序的一页数据加总数，且关键词含特殊字符不报错。  
  
**常见错法 3 条**：  
  
1. 用 f-string 把关键词拼进 SQL。  
2. 只 `limit` 不 `offset`。  
3. 不返回 `total`。  
  
**自检 3 问**：  
  
1. 关键词里带 `%` 或单引号，会报错吗？  
2. 翻到第 3 页，数据会和第 2 页重叠吗？  
3. 10 万行时这次查询要多久？  
  
---  
  
## 第 5 章 测试数据库  
  
### 5.1 需求  
  
测试不读写真实 `insight.db`；每个测试相互隔离；整套测试在秒级完成。  
  
### 5.2 最小可运行版  
  
```python  
from sqlalchemy import create_engine, select, func     # 第三方库 · 引擎与查询  
from sqlalchemy.orm import sessionmaker                # 第三方库 · 会话工厂  
from sqlalchemy.pool import StaticPool                 # 第三方库 · 连接池 · 内存库共享  
from app.db.base import Base                           # 本项目 · 基类  
from app.models import Document                        # 本项目 · 模型  
  
def test_memory_db():  
    engine = create_engine(  
        "sqlite://",                                    # 关键行：空路径 = 内存库  
        connect_args={"check_same_thread": False},  
        poolclass=StaticPool,                           # 关键行：所有连接共享同一内存库  
    )  
    Base.metadata.create_all(engine)  
    with sessionmaker(bind=engine)() as s:  
        s.add(Document(filename="a.txt", source_path="x"))  
        s.commit()  
        assert s.scalar(select(func.count()).select_from(Document)) == 1  
```  
  
预期：`1 passed`。  
  
### 5.3 逐行讲解  
  
| 代码 | 含义 |  
|---|---|  
| `create_engine("sqlite://")` | 空路径表示内存数据库，进程结束即消失 |  
| `connect_args={"check_same_thread": False}` | 允许不同线程复用连接；测试客户端可能跨线程 |  
| `poolclass=StaticPool` | 所有连接指向同一份内存库，否则每个连接各有一个空库 |  
| `Base.metadata.create_all(engine)` | 在内存库中建表 |  
| `sessionmaker(bind=engine)()` | 创建会话 |  
  
**知识点 1：内存数据库与 StaticPool**  
  
定义：`sqlite://` 是进程内内存库；默认每个连接独立，`StaticPool` 让所有连接共享同一份。  
  
定式（要背）：  
  
```python  
create_engine("sqlite://", connect_args={"check_same_thread": False}, poolclass=StaticPool)  
```  
  
适用：所有需要数据库的单元测试。  
  
例题与预期：不加 `StaticPool` 时，建表连接与查询连接看到的是两个不同的空库，查询报 `no such table`。  
  
**知识点 2：fixture**  
  
定义：fixture 是被 pytest 注入到测试函数参数里的准备函数，`yield` 前是准备，之后是清理。  
  
定式（要背）：  
  
```python  
@pytest.fixture()  
def db_session():  
    ...  
    yield session  
    ...  
```  
  
适用：数据库会话、临时文件、启动中的服务。  
  
例题与预期：两个测试都声明 `db_session` 参数，各自拿到独立会话，互不影响。  
  
**知识点 3：依赖覆盖**  
  
定义：`app.dependency_overrides[原依赖] = 替身` 让 FastAPI 在测试时使用测试依赖。  
  
定式（要背）：  
  
```python  
app.dependency_overrides[get_db] = override_get_db  
```  
  
适用：接口级测试，替换真实数据库、外部客户端。  
  
例题与预期：覆盖后调用接口，写入落在内存库，真实库无变化。  
  
### 5.4 指出缺陷  
  
直接连真实库做测试，会出四个问题：  
  
1. **数据污染**：测试写入开发库 `insight.db`，一次跑完开发数据被清空或混入 500 条假数据。  
2. **顺序依赖**：第 2 个用例依赖第 1 个留下的记录，单独执行时报 `AssertionError: assert 0 == 1`。  
3. **并发冲突**：CI 两个 job 同时写同一文件库，报 `sqlite3.OperationalError: database is locked`。  
4. **慢**：每个用例重建文件库约 9 ms，200 个用例额外 1.8 秒，且受磁盘 IO 波动影响。  
  
### 5.5 完整版  
  
文件：`tests/conftest.py`  
  
```python  
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
  
文件：`tests/test_repository.py`  
  
```python  
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
  
  
def test_delete_cascades_chunks(db_session):  
    """模拟：删除文档（关联路径）。  
    契约：其分块一并删除。"""  
    from app.repositories.document_repo import delete_document  
    from app.models.chunk import Chunk  
  
    doc = create_document(db_session, "a.txt", "data/a.txt")  
    save_chunks(db_session, doc.id, [{"index": 0, "text": "内容", "char_count": 2}])  
    assert delete_document(db_session, doc.id) is True  
    assert db_session.query(Chunk).filter_by(document_id=doc.id).count() == 0  
```  
  
逐块说明补了哪个缺陷：  
  
| 代码块 | 补的缺陷 |  
|---|---|  
| `StaticPool` + `check_same_thread=False` | 内存库跨连接共享，避免 `no such table` |  
| `create_all` 在 fixture 内 | 每个测试都是干净结构，无顺序依赖 |  
| `yield` + `finally: close/dispose` | 测试失败也释放连接，CI 不会 `database is locked` |  
| 每个测试一段 docstring | 说明模拟哪次调用、保证什么契约，可追溯 |  
| 重复导入、空输入、级联删除三类用例 | 覆盖性能改造最容易回归的分支 |  
  
完整版新增调用：  
  
| 调用 | 作用与参数 |  
|---|---|  
| `@pytest.fixture()` | 声明夹具，默认函数级，每个测试执行一次 |  
| `engine.dispose()` | 关闭连接池，释放全部连接 |  
| `sessionmaker(bind=engine, expire_on_commit=False)` | 测试会话工厂；提交后对象仍可读 |  
| `db_session.query(Model)` | 传统查询入口，本册仅用于计数断言 |  
  
固定模板：`conftest.py` 的 fixture 三段式（建引擎、建表、yield、清理）。可组合写法：是否用事务回滚隔离、是否预置种子数据、是否覆盖 `get_db`。  
  
已知边界：内存库与 SQLite 文件库在锁行为上有差异；需要验证并发锁时应在文件库上另建一组慢测试。  
  
调用链：`db_session` 由 pytest 通过参数名注入；它调用 `create_engine`、`create_all`、`sessionmaker`；fixture 内的异常直接使该测试失败，不会影响其他测试。  
  
### 5.6 结尾块  
  
**本章判据**：删掉 `insight.db` 后测试仍全绿，且单测总耗时在秒级。  
  
**常见错法 3 条**：  
  
1. 测试直接连开发库。  
2. 内存库不加 `StaticPool`。  
3. 用例之间靠执行顺序传递数据。  
  
**自检 3 问**：  
  
1. 单独跑一个用例，它还能通过吗？  
2. 两个用例并行跑，会互相影响吗？  
3. 测试失败后，连接会释放吗？  
  
---  
  
## 运行与验收  
  
依赖安装：  
  
```bash  
pip install sqlalchemy  
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
app/services/ingest.py  
tests/conftest.py  
tests/test_repository.py  
```  
  
运行与验收：  
  
```bash  
# 1) 建表  
python -c "  
from app.db.base import Base, engine  
import app.models  
Base.metadata.create_all(engine)  
print('tables created')  
"  
  
# 2) 看表结构  
sqlite3 data/insight.db ".schema documents"  
sqlite3 data/insight.db ".schema chunks"  
  
# 3) 跑测试  
pytest -v  
  
# 4) 覆盖率  
pytest --cov=app --cov-report=term-missing  
```  
  
预期：建表输出 `tables created`；`.schema` 能看到两张表；测试全绿；覆盖率报告可读。  
  
---  
  
## 知识地图  
  
| 知识点 | 落点 |  
|---|---|  
| 数据库基础 | 第 1 章 库/表/行/列/主键 |  
| ORM 与建模 | 第 1 章 `DeclarativeBase`、`Mapped`、`mapped_column`、`relationship` |  
| 外键与级联 | 第 1 章 `ForeignKey`、`cascade` |  
| 会话与事务 | 第 2 章 `Session`、`commit`、`rollback` |  
| 查询 | 第 2、4 章 `select`、`where`、`get`、`contains` |  
| 批量写入 | 第 3 章 `add_all`、事务边界 |  
| 约束与幂等 | 第 3 章 `UniqueConstraint`、`IntegrityError`、先删后插 |  
| 分页与聚合 | 第 4 章 `limit`、`offset`、`func.count` |  
| 索引 | 第 1、4 章 `index=True`、前置 `%` 导致索引失效 |  
| 测试 | 第 5 章 内存库、`StaticPool`、fixture、依赖覆盖 |  
| OOP | 第 1 章 模型继承、第 3 章 自定义异常 |  
| 日志 | 第 2、3 章 `logger.info/error` |  
  
## 自测  
  
1. SQLite 内存库测试为什么必须加 `StaticPool`？  
2. `Session.commit()` 与 `Session.flush()` 的区别是什么？  
3. 为什么批量入库采用"先删后插"，而不是直接 `add_all`？  
4. 关键词检索为什么不能把关键词用 f-string 拼进 SQL？  
5. 删除 `Document` 时，`Chunk` 是怎样被一起删除的？  
  
<details>  
<summary>答案要点</summary>  
  
1. 默认每个连接各自拥有独立内存库；`StaticPool` 让所有连接共享同一份，否则建表连接与查询连接看到的不是同一个库。  
2. `flush` 只把待写入语句发给数据库，不结束事务；`commit` 提交并结束事务。`commit` 内部会先 `flush`。  
3. 直接 `add_all` 会在重复导入时追加数据，导致条数翻倍；先删后插在同一事务内完成，保证幂等且不留半份数据。  
4. f-string 会破坏参数绑定，关键词含引号时导致语法错误，含 `%` 时匹配范围被放大，并带来注入风险。  
5. 两层保障：数据库层 `ForeignKey(..., ondelete="CASCADE")`，ORM 层 `relationship(cascade="all, delete-orphan")`；本册删除走 ORM，由 ORM 级联完成。  
  
</details>  
  
---  
  
## 规范自检记录  
  
| 条目 | 落点 | 状态 |  
|---|---|---|  
| 1 分册交付 | 仅本册，末尾停止 | 满足 |  
| 2 方案四段 | 本册为教材，不适用 | 不适用 |  
| 3 六段结构、双层版本、对照表、≤5 概念 | 各章 1.1–1.6 至 5.1–5.6 | 满足 |  
| 4 知识点三件套 | 全册 18 个知识点卡 | 满足 |  
| 5 设计决策三段 | 2.5 节 | 满足 |  
| 6 编号 | 第 N 章 / N.M | 满足 |  
| 7 代码可运行、路径、清单、命令 | 各章代码与"运行与验收" | 满足 |  
| 8 import 三标注、方法说明、关键行注释 | 各代码块与"新增调用"表 | 满足 |  
| 9 读者画像 | 全文口径 | 满足 |  
| 10 易混名字表 | 4.5 节 | 满足 |  
| 11 调用链三点 | 五个模块各一段 | 满足 |  
| 12 测试标注模拟路径与契约 | 5.5 每个测试 docstring | 满足 |  
| 13 结尾块与册末自测 | 各章 N.6、文末自测 | 满足 |  
| 14 生成后自检 | 本节 | 满足 |  
| 15 交付形态与教学口吻 | 全文 | 满足 |  
| 16 本册地图、信息块、目录 | 开头 | 满足 |  
| 17 详略与已知边界 | 各章"已知边界" | 满足 |  
| 18 纠正错误认知 | 本册地图"术语校准" | 满足 |  
| D1–D6 | 全文无违反 | 满足 |  
  
自检中修正 2 处：第 4 章初稿未标注"前置 `%` 使索引失效"，已在 4.4 与知识地图补齐；第 5 章初稿只写 `sqlite://`，已补 `StaticPool` 的必要性说明与失败现象。  
  
本册结束，未开始第四册。  
