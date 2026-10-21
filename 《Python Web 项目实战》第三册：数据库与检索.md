# 《Python Web 项目实战》第三册：数据库与检索

| 项目     | 内容                                                         |
| -------- | ------------------------------------------------------------ |
| 所属系列 | 《Python Web 项目实战》                                      |
| 册数     | 第三册（对应阶段三：Day 5-6）                                |
| 主题     | 数据库建模 · SQLAlchemy · 增删改查 · 事务 · 关键词检索 · 检索抽象 |
| 前置要求 | 完成第一、二册                                               |
| 版本     | v1.0（2026-10-22）                                           |

> **本册定位**：第二册把文件切成了 `list[dict]`，但它只活在内存里，程序一关就没了。这一册要把这些块**存进数据库**，并且能**按关键词查出来**。
>
> 读法同前：需求 → A 版能跑 → 逐行 → 发现毛病 → B 版 → 3 句话总结。

---

## 目录

- 第 0 章 本册地图
- 第 1 章 数据库最小常识
- 第 2 章 从裸 SQL 到 ORM
- 第 3 章 定义模型（建表）
- 第 4 章 Engine 与 Session
- 第 5 章 增删改查四件套
- 第 6 章 把 chunk 存进去
- 第 7 章 检索：从 Python 过滤到 SQL 查询
- 第 8 章 检索抽象（为向量检索留口子）
- 第 9 章 测试
- 第 10 章 接入 API
- 第 11 章 自测

---

# 第 0 章 本册地图

```
第二册产出：list[dict]（内存里）
        ↓ 第1~3章：建表、定义模型
        ↓ 第4章：连上数据库
        ↓ 第5~6章：写进去
        ↓ 第7章：查出来（关键词）
        ↓ 第8章：把"查"抽象成接口
        ↓ 第9~10章：测试 + 暴露成 API
```

本册新增文件：

```
app/
├── db/
│   ├── base.py              # Base 基类
│   └── session.py           # engine + SessionLocal + get_db
├── models/
│   └── document.py          # Document / Chunk 两张表
├── repositories/
│   └── document_repo.py     # 所有数据库操作都放这里
├── schemas/
│   └── document.py          # 请求/响应模型
├── services/
│   └── retriever.py         # 检索抽象 + 关键词实现
└── api/v1/
    └── documents.py         # 三个接口
tests/
└── test_repo.py
```

**先记住这条分层规则**（后面每一章都在用它）：

> **API 层不碰数据库，数据库操作只写在 repository 层，模型只描述表结构。**
> 这样以后换数据库、加缓存，只改 repository，不动接口。

---

# 第 1 章 数据库最小常识

## 1.1 需求

我们要存两类东西：

- **文档**：文件名、原文、上传时间
- **文本块**：属于哪个文档、第几块、内容、字数

而且一块必须知道它属于哪个文档。

## 1.2 用 Excel 类比

| 数据库术语           | Excel 类比         | 我们的例子                               |
| -------------------- | ------------------ | ---------------------------------------- |
| 表（table）          | 一个 sheet         | `documents` 表、`chunks` 表              |
| 行（row / record）   | 一行数据           | 一个文档、一个块                         |
| 列（column / field） | 表头               | `id`、`filename`、`content`              |
| 主键（primary key）  | 唯一编号列         | `id`，每行唯一、不重复                   |
| 外键（foreign key）  | 指向另一张表的编号 | `chunks.document_id` 指向 `documents.id` |
| 一对多（1:N）        | 一个订单多个商品   | **一个文档多个块**                       |

**关系图：**

```
documents                        chunks
+----+----------+          +----+-------------+-------+------+
| id | filename |          | id | document_id | index | text |
+----+----------+          +----+-------------+-------+------+
|  1 | a.md     | ←──────→ |  1 |      1      |   0   | ...  |
|  2 | b.txt    |          |  2 |      1      |   1   | ...  |
+----+----------+          |  3 |      2      |   0   | ...  |
                           +----+-------------+-------+------+
        ↑ 一个文档                         ↑ 每个块都记着"我属于谁"
```

## 1.3 只需会 5 条 SQL

SQL 是数据库的"普通话"。你不必精通，但这 5 条要能看懂：

```sql
-- 1. 建表
CREATE TABLE documents (id INTEGER PRIMARY KEY, filename TEXT);

-- 2. 插入
INSERT INTO documents (filename) VALUES ('a.md');

-- 3. 查询（WHERE 是筛选，ORDER BY 排序，LIMIT 限条数）
SELECT * FROM documents WHERE filename LIKE '%a%' ORDER BY id DESC LIMIT 5;

-- 4. 更新
UPDATE documents SET filename = 'b.md' WHERE id = 1;

-- 5. 删除
DELETE FROM documents WHERE id = 1;
```

| 关键字              | 作用            | 记忆         |
| ------------------- | --------------- | ------------ |
| `WHERE`             | 筛选条件        | “只要满足的” |
| `LIKE '%关键词%'`   | 模糊匹配        | “包含关键词” |
| `ORDER BY ... DESC` | 排序，DESC 倒序 | “按什么排”   |
| `LIMIT n`           | 只要前 n 条     | “最多给几条” |

## 1.4 为什么不用 Python 的 list 存

| 问题           | 用 list           | 用数据库       |
| -------------- | ----------------- | -------------- |
| 程序重启       | 数据全丢          | 数据还在       |
| 数据量 10 万条 | 全在内存，慢      | 有索引，毫秒级 |
| 多个请求同时写 | 会互相覆盖        | 有事务，安全   |
| 按条件查       | 要自己写 for 循环 | 一条 SQL       |

### 本章只需记住 3 句话

1. 表 = sheet，行 = 一条记录，主键 = 唯一编号，外键 = 指向别的表。
2. 一对多关系靠**外键**实现（chunk 记着 document_id）。
3. 需要持久化 / 查询 / 并发，就该上数据库，别用 list。

---

# 第 2 章 从裸 SQL 到 ORM

## 2.1 A 版：用 Python 自带的 `sqlite3` 写裸 SQL

```python
import sqlite3

conn = sqlite3.connect("data/insight.db")      # ① 连接数据库文件
cursor = conn.cursor()                          # ② 拿一个"执行器"

cursor.execute("""
    CREATE TABLE IF NOT EXISTS documents (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        filename TEXT NOT NULL
    )
""")

cursor.execute(
    "INSERT INTO documents (filename) VALUES (?)",   # ③ ? 是占位符
    ("a.md",)
)
conn.commit()                                   # ④ 提交，否则不生效

cursor.execute("SELECT id, filename FROM documents")
rows = cursor.fetchall()                        # ⑤ 返回元组列表 [(1, 'a.md')]
print(rows)

conn.close()
```

**逐行讲：**

| 代码                    | 含义                                              |
| ----------------------- | ------------------------------------------------- |
| `sqlite3.connect(路径)` | 打开（或创建）一个数据库文件                      |
| `conn.cursor()`         | 拿到执行器，所有 SQL 都通过它执行                 |
| `execute(sql, 参数)`    | 执行一条 SQL                                      |
| `?` 占位符              | **千万不能用 f-string 拼 SQL**，否则会有 SQL 注入 |
| `conn.commit()`         | 提交。不提交 = 白写                               |
| `fetchall()`            | 取出所有结果，每行是一个元组                      |

## 2.2 A 版的三个毛病

**（1）查询结果是元组，要用下标取**

```python
row = rows[0]
print(row[1])      # ❌ 1 是什么？三个月后你自己都不记得
```

**（2）手写 SQL 字符串，改字段就要改一堆地方**

加个 `created_at` 字段，建表、插入、查询、更新全都要改。

**（3）没有类型提示，字段名拼错只能运行时才发现**

## 2.3 B 版：ORM 是什么

**ORM = Object Relational Mapping，对象关系映射。**

一句话：**让你用"操作 Python 对象"的方式操作数据库表。**

| 你想做的事 | 裸 SQL                                       | ORM                     |
| ---------- | -------------------------------------------- | ----------------------- |
| 取一个文档 | `SELECT * FROM documents WHERE id=1`         | `db.get(Document, 1)`   |
| 改文件名   | `UPDATE documents SET filename=? WHERE id=1` | `doc.filename = "b.md"` |
| 取它的块   | `SELECT * FROM chunks WHERE document_id=1`   | `doc.chunks`            |
| 新增       | `INSERT INTO ...`                            | `db.add(doc)`           |

**核心映射关系：**

```
Python 类      ↔  数据库表
类属性          ↔  表的列
一个实例        ↔  表中的一行
实例.属性       ↔  列的值
```

> **关键认知**：ORM 不是"不用懂数据库"，而是"用对象语法写 SQL"。**下面每条 ORM 语句，你都要能在心里翻译成 SQL**——这样出问题时才查得动。

### 本章只需记住 3 句话

1. 裸 `sqlite3` 要手写 SQL、结果是元组、改字段很痛。
2. ORM 把"表/行/列"映射成"类/实例/属性"。
3. **ORM 语句要能翻译回 SQL**，否则等于不会用。

---

# 第 3 章 定义模型（建表）

## 3.1 本章新增 import 一览

| 导入                                                         | 它是谁            | 干什么用                                 |
| ------------------------------------------------------------ | ----------------- | ---------------------------------------- |
| `from sqlalchemy.orm import DeclarativeBase`                 | 声明式基类        | 所有模型的父类                           |
| `from sqlalchemy.orm import Mapped, mapped_column`           | 类型工具 + 列定义 | 现代写法定义列                           |
| `from sqlalchemy import ForeignKey, String, Text, DateTime, func` | 各种列类型 / 函数 | 外键、定长字符串、长文本、时间、SQL 函数 |
| `from datetime import datetime`                              | Python 时间类型   | 类型注解用                               |

## 3.2 先建 Base

```python
# app/db/base.py
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    """所有模型的公共父类。空着就行，它的作用是收集所有表的信息。"""
    pass
```

> `DeclarativeBase` 是 SQLAlchemy 2.0 的固定写法。**当成模板背下来**：建一个空类继承它，起名 `Base`。

## 3.3 定义两张表

```python
# app/models/document.py
from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, String, Text, func
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.db.base import Base


class Document(Base):
    __tablename__ = "documents"          # ① 表名（必须写）

    id: Mapped[int] = mapped_column(primary_key=True)          # ② 主键，自增
    filename: Mapped[str] = mapped_column(String(255))         # ③ 定长字符串
    content: Mapped[str] = mapped_column(Text)                 # ④ 长文本
    created_at: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now()                    # ⑤ 由数据库自动填时间
    )

    chunks: Mapped[list["Chunk"]] = relationship(              # ⑥ Python 层面的"一对多"
        back_populates="document",
        cascade="all, delete-orphan",                          # ⑦ 删文档时，块一起删
    )


class Chunk(Base):
    __tablename__ = "chunks"

    id: Mapped[int] = mapped_column(primary_key=True)
    document_id: Mapped[int] = mapped_column(
        ForeignKey("documents.id", ondelete="CASCADE")         # ⑧ 外键
    )
    index: Mapped[int]                                          # ⑨ 不写字长，默认类取决于类型注解
    text: Mapped[str] = mapped_column(Text)
    char_count: Mapped[int]

    document: Mapped["Document"] = relationship(back_populates="chunks")
```

**逐行讲（重点看 ②③④⑥⑧）：**

| 编号 | 代码                               | 含义                                                         |
| ---- | ---------------------------------- | ------------------------------------------------------------ |
| ①    | `__tablename__`                    | 告诉 SQLAlchemy 这个类对应哪张表                             |
| ②    | `mapped_column(primary_key=True)`  | 主键，SQLite 下自动自增                                      |
| ③    | `String(255)`                      | 最大 255 字符。超了会报错（这是保护，不是限制）              |
| ④    | `Text`                             | 不限长度的字符串，存正文用                                   |
| ⑤    | `server_default=func.now()`        | 插入时**数据库自己**填当前时间，不用 Python 传               |
| ⑥    | `relationship(...)`                | **不是数据库字段**，是 Python 层面的导航：`doc.chunks` 直接拿到列表 |
| ⑦    | `cascade="all, delete-orphan"`     | 删 Document 时，它的 Chunk 全部自动删除                      |
| ⑧    | `ForeignKey("documents.id")`       | 真·外键，写在数据库里                                        |
| ⑨    | `Mapped[int]` 不写 `mapped_column` | 简单字段可以省略，SQLAlchemy 自动推断为 Integer              |

**`Mapped[list["Chunk"]]` 怎么读？**

`Mapped[list[Chunk]]` = "这个属性是一个 Chunk 的列表"。字符串 `"Chunk"` 是因为 Chunk 还没定义，先写个名字占位（前向引用）。

## 3.4 建表：`create_all`

```python
# app/db/session.py 片段
from app.db.base import Base
from app.models.document import Chunk, Document   # noqa: F401  必须导入，否则 Base 不知道有这两张表

Base.metadata.create_all(bind=engine)
```

> **坑**：`create_all` 只认识"已经被导入过的模型类"。如果忘了 import `Document`，它不会建表，也不会报错——**这是新手最常见的"表不存在"原因**。

> **正式项目用 Alembic 做迁移**（改表结构不改数据）。学习阶段先用 `create_all`，够用。

### 本章只需记住 3 句话

1. 模型 = 继承 `Base` 的类 + `__tablename__` + `Mapped[...] = mapped_column(...)`。
2. 一对多 = 子表写 `ForeignKey`，父表写 `relationship`。
3. `create_all` 前必须 import 所有模型，否则表不会建。

---

# 第 4 章 Engine 与 Session

## 4.1 两个核心对象

| 对象      | 是什么                           | 类比                 |
| --------- | -------------------------------- | -------------------- |
| `Engine`  | 连接数据库的"总工厂"，管理连接池 | 自来水厂             |
| `Session` | 一次数据库对话                   | 你打开水龙头接一桶水 |

**规则：Engine 全项目只建一个；Session 每个请求建一个、用完就关。**

## 4.2 完整代码

```python
# app/db/session.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from app.config import settings
from app.db.base import Base
from app.models.document import Chunk, Document  # noqa: F401

# ① 引擎：连接串 + 是否打印 SQL
engine = create_engine(
    settings.DATABASE_URL,      # "sqlite:///./data/insight.db"
    echo=settings.DEBUG,        # 开发时打印 SQL，方便学习和排查
    future=True,
)

# ② 会话工厂：调用一次 = 造一个 Session
SessionLocal = sessionmaker(
    bind=engine,
    autoflush=False,            # 不自动 flush，手动控制提交时机
    expire_on_commit=False,     # commit 后对象还能继续读（否则属性会失效）
)


# ③ 建表（学习阶段放这里，正式项目用 Alembic）
def init_db() -> None:
    Base.metadata.create_all(bind=engine)


# ④ FastAPI 依赖：每个请求拿一个 Session，用完自动关
def get_db():
    """用法：def api(db: Session = Depends(get_db))"""
    db = SessionLocal()
    try:
        yield db                # yield 给接口用
    finally:
        db.close()              # 不管成功失败都关
```

**逐行讲：**

| 代码                               | 含义                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| `create_engine(url, echo=...)`     | 造引擎。`echo=True` 会把执行的 SQL 打到控制台，**学习期强烈建议开** |
| 连接串格式                         | `sqlite:///./data/insight.db`（三个斜杠 = 相对路径）         |
| `sessionmaker(bind=engine)`        | 造一个"Session 工厂"，不是 Session 本身                      |
| `expire_on_commit=False`           | 默认 commit 后对象会过期，再读属性会重新查库；关掉它更适合 Web |
| `yield db` + `finally: db.close()` | 生成器形式的依赖注入（第二册第 6 章的 yield 又用上了）       |

## 4.3 commit / rollback / refresh

| 方法              | 作用                                  | 什么时候用         |
| ----------------- | ------------------------------------- | ------------------ |
| `db.add(obj)`     | 把对象加入"待写入"队列                | 新增/修改前        |
| `db.commit()`     | **真正写进数据库**                    | 一个完整操作结束时 |
| `db.rollback()`   | 撤销本次会话所有未提交的改动          | 出异常时           |
| `db.refresh(obj)` | 从数据库重新读一遍对象（拿到自增 id） | commit 之后        |
| `db.flush()`      | 把 SQL 发给数据库但**不提交**         | 需要提前拿 id 时   |

**最重要的一句**：`add` 只是"记在账上"，**`commit` 才是真正落库**。

### 本章只需记住 3 句话

1. Engine 全局一个，Session 每请求一个。
2. `add` 只是排队，`commit` 才真正写入。
3. `get_db` 用 `yield` 提供 Session，`finally` 里关闭。

---

# 第 5 章 增删改查四件套

所有数据库操作都写在 **repository 层**，接口层只调用函数。

```python
# app/repositories/document_repo.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from app.models.document import Chunk, Document


# ---------- 增 ----------
def create_document(db: Session, filename: str, content: str) -> Document:
    doc = Document(filename=filename, content=content)   # ① 造对象（还没进库）
    db.add(doc)                                          # ② 加入待写入队列
    db.commit()                                          # ③ 提交
    db.refresh(doc)                                      # ④ 回读，拿到自增 id
    return doc


# ---------- 查（单个）----------
def get_document(db: Session, doc_id: int) -> Document | None:
    return db.get(Document, doc_id)      # 按主键查，最简单


# ---------- 查（列表）----------
def list_documents(db: Session, limit: int = 20, offset: int = 0) -> list[Document]:
    stmt = (
        select(Document)                 # ① 相当于 SELECT * FROM documents
        .order_by(Document.id.desc())    # ② 按 id 倒序
        .limit(limit)                    # ③ 最多几条
        .offset(offset)                  # ④ 跳过几条（分页用）
    )
    return list(db.scalars(stmt))        # ⑤ 取出结果


# ---------- 改 ----------
def rename_document(db: Session, doc_id: int, new_name: str) -> Document | None:
    doc = db.get(Document, doc_id)
    if doc is None:
        return None
    doc.filename = new_name              # ① 直接改属性
    db.commit()                          # ② 提交，SQLAlchemy 自动生成 UPDATE
    return doc


# ---------- 删 ----------
def delete_document(db: Session, doc_id: int) -> bool:
    doc = db.get(Document, doc_id)
    if doc is None:
        return False
    db.delete(doc)                       # ① 标记删除（级联删掉它的 chunks）
    db.commit()
    return True
```

**几个必须理解的语法：**

| 代码                 | 含义                                                         |
| -------------------- | ------------------------------------------------------------ |
| `db.get(模型, 主键)` | 按主键查，等价 `SELECT * WHERE id=?`                         |
| `select(Document)`   | 构造一条 SELECT 语句（SQLAlchemy 2.0 新写法）                |
| `db.scalars(stmt)`   | 执行查询，返回"单个值"的迭代器（这里单个值 = 一个 Document 对象） |
| `list(...)`          | 把结果转成列表                                               |
| `doc.filename = "x"` | **改属性就等于改表**，commit 后生效                          |
| `db.delete(doc)`     | 标记删除，commit 后生效                                      |

## 5.1 A 版 vs B 版：批量插入的事务问题

**A 版：一条一条提交**

```python
for c in chunk_dicts:
    db.add(Chunk(**c))
    db.commit()          # ❌ 1000 条 = 1000 次磁盘写，慢到爆
```

**B 版：一次提交**

```python
db.add_all([Chunk(**c) for c in chunk_dicts])
db.commit()              # ✅ 一次事务，快几十倍
```

**为什么差别这么大？** 每次 `commit` 都是一次磁盘同步，有固定开销。合并成一次事务，开销只付一次。

**事务的意义**：`commit` 之前的所有改动是一个整体——要么全成功，要么全失败（`rollback`）。

```python
try:
    db.add_all(chunks)
    db.commit()
except Exception:
    db.rollback()        # 出错就全部撤销，不留半截数据
    raise
```

> **这一条非常重要**：批量写数据时，"要么全进，要么全不进"，否则会出现"文档存了、块只存了一半"的脏数据。

### 本章只需记住 3 句话

1. 增删改查 = `add` / `get|select` / 改属性 / `delete`，最后都要 `commit`。
2. 批量写入用 `add_all` + **一次 commit**，不要循环 commit。
3. 出错要 `rollback`，保证"要么全成功，要么全失败"。

---

# 第 6 章 把 chunk 存进去

## 6.1 组装逻辑

第二册的 `process()` 返回 `list[dict]`，现在要塞进 `Chunk` 表，并挂到某个 `Document` 下。

```python
# app/repositories/document_repo.py 追加
from app.services.splitter import process


def save_document_with_chunks(
    db: Session,
    filename: str,
    content: str,
    chunk_size: int = 300,
    overlap: int = 50,
) -> Document:
    """一步完成：存文档 + 切块 + 存块。"""
    # ① 幂等：同名文档先删旧的，避免重复上传产生多份
    old = db.scalars(select(Document).where(Document.filename == filename)).first()
    if old is not None:
        db.delete(old)
        db.flush()          # 立刻执行删除，为后面腾出空间

    # ② 建文档对象
    doc = Document(filename=filename, content=content)
    db.add(doc)
    db.flush()              # 提前拿到 doc.id（还没 commit）

    # ③ 切块
    pieces = process(content, size=chunk_size, overlap=overlap)

    # ④ 造 Chunk 对象，挂到 doc 上
    doc.chunks = [
        Chunk(index=p["index"], text=p["text"], char_count=p["char_count"])
        for p in pieces
    ]

    # ⑤ 一次提交
    try:
        db.commit()
    except Exception:
        db.rollback()
        raise

    db.refresh(doc)
    return doc
```

**逐段讲：**

| 代码                             | 含义                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| `select(...).where(...).first()` | 查第一条匹配的，没有返回 `None`                              |
| `db.flush()`                     | **发 SQL 但不提交**，用来提前拿到自增 id                     |
| `doc.chunks = [...]`             | 直接给 relationship 赋列表，SQLAlchemy 会自动填 `document_id` |
| `Chunk(index=..., text=...)`     | 不用手动写 `document_id`，关系会自动补                       |

**为什么 `flush()` 这么关键？**

`doc.id` 是数据库自增的，不执行 INSERT 就拿不到。`flush()` = "先发给数据库但不提交"，这样既能拿到 id，又保留了"可以整体回滚"的能力。

> **关系赋值 vs 手动填外键**：
> `doc.chunks = [chunk1, chunk2]` 是 ORM 的写法，自动补 `document_id`；
> 手动写 `Chunk(document_id=doc.id, ...)` 也行，但更啰嗦、更容易错。

### 本章只需记住 3 句话

1. 存主体用 `flush()` 拿 id，再挂子对象，最后统一 `commit`。
2. `doc.chunks = [...]` 会自动填外键，不用手写 `document_id`。
3. 幂等：同名先删旧，避免重复数据。

---

# 第 7 章 检索：从 Python 过滤到 SQL 查询

## 7.1 A 版：全查出来，在 Python 里筛

```python
def search_bad(db: Session, keyword: str) -> list[Chunk]:
    all_chunks = list(db.scalars(select(Chunk)))     # ① 全表读进内存
    return [c for c in all_chunks if keyword in c.text]   # ② Python 过滤
```

**毛病：**

| 问题                  | 后果                         |
| --------------------- | ---------------------------- |
| 全表读进内存          | 10 万条就卡死                |
| Python 里做字符串匹配 | 无法利用数据库索引           |
| 没有排序、没有截断    | 结果顺序随机，可能返回上万条 |

## 7.2 B 版：让数据库来筛

```python
def search_chunks(db: Session, keyword: str, top_k: int = 5) -> list[Chunk]:
    stmt = (
        select(Chunk)
        .where(Chunk.text.contains(keyword))   # ① 等价 SQL: text LIKE '%关键词%'
        .order_by(Chunk.id)                     # ② 按 id 排序，保证结果稳定
        .limit(top_k)                           # ③ 只要前 top_k 条
    )
    return list(db.scalars(stmt))
```

**翻译成 SQL：**

```sql
SELECT * FROM chunks WHERE text LIKE '%部署%' ORDER BY id LIMIT 5;
```

| ORM                               | SQL                      | 作用                           |
| --------------------------------- | ------------------------ | ------------------------------ |
| `.where(Chunk.text.contains(kw))` | `WHERE text LIKE '%kw%'` | 模糊匹配                       |
| `.order_by(Chunk.id)`             | `ORDER BY id`            | 排序（不加排序，分页结果会乱） |
| `.limit(5)`                       | `LIMIT 5`                | 限制条数                       |

> **`.contains()` 安全吗？** 安全。它不是字符串拼接，而是参数化查询（`?` 占位符），**天然防 SQL 注入**。

## 7.3 返回 richer 的结果

实际业务里，光返回 Chunk 不够，还要知道它来自哪个文档：

```python
def search_with_source(db: Session, keyword: str, top_k: int = 5) -> list[dict]:
    stmt = (
        select(Chunk)
        .where(Chunk.text.contains(keyword))
        .order_by(Chunk.id)
        .limit(top_k)
    )
    chunks = db.scalars(stmt)

    return [
        {
            "chunk_id": c.id,
            "text": c.text,
            "document_id": c.document_id,
            "filename": c.document.filename,   # ① 通过 relationship 直接取
        }
        for c in chunks
    ]
```

`c.document` 是 relationship 提供的**懒加载**：访问时才去查关联的 Document。**注意**：这会在循环里产生 N 次查询（N+1 问题），数据量大时要改成 join 预加载。学习阶段先这样写，知道有这回事即可。

## 7.4 分页

```python
def search_paged(db: Session, keyword: str, page: int = 1, size: int = 10):
    stmt = (
        select(Chunk)
        .where(Chunk.text.contains(keyword))
        .order_by(Chunk.id)
        .limit(size)
        .offset((page - 1) * size)
    )
    return list(db.scalars(stmt))
```

`offset((page-1)*size)`：第 1 页跳过 0 条，第 2 页跳过 10 条……

## 7.5 索引：什么时候该加

```python
document_id: Mapped[int] = mapped_column(ForeignKey("documents.id"), index=True)
```

| 场景                 | 是否加索引                 |
| -------------------- | -------------------------- |
| 经常用某列做 `WHERE` | ✅ 加                       |
| 经常按某列排序       | ✅ 加                       |
| 外键列               | ✅ 建议加                   |
| 很少查的列           | ❌ 别加（占空间、拖慢写入） |

> **`LIKE '%关键词%'` 用不上普通索引**（前面有通配符）。数据量大了要上全文索引或向量检索——这正是第四册要解决的。

### 本章只需记住 3 句话

1. 筛选、排序、限制**交给数据库**，不要全读进 Python。
2. ORM 的 `.where/.order_by/.limit` 就是 SQL 的 `WHERE/ORDER BY/LIMIT`。
3. 分页用 `limit + offset`；排序必须写，否则分页会乱。

---

# 第 8 章 检索抽象（为向量检索留口子）

## 8.1 为什么要抽象

第四册要加"向量检索"，以后可能还要"混合检索"。如果接口层直接写死 `search_chunks(db, ...)`，每加一种方式就要改接口。

**目标：接口层只认一个方法 `search(query, top_k)`，具体用哪种实现，由配置决定。**

## 8.2 抽象基类：`ABC` + `abstractmethod`

```python
# app/services/retriever.py
from abc import ABC, abstractmethod
from dataclasses import dataclass

from sqlalchemy import select
from sqlalchemy.orm import Session

from app.models.document import Chunk


@dataclass
class SearchResult:                 # ① 统一的结果结构
    chunk_id: int
    text: str
    filename: str
    score: float = 1.0              # 相似度，关键词检索暂时固定为 1.0


class BaseRetriever(ABC):           # ② 抽象基类（"接口"）
    @abstractmethod                 # ③ 子类必须实现它
    def search(self, query: str, top_k: int = 5) -> list[SearchResult]:
        """检索接口：输入问题，返回最相关的若干片段。"""
        ...
```

| 代码                                  | 含义                                                 |
| ------------------------------------- | ---------------------------------------------------- |
| `from abc import ABC, abstractmethod` | 标准库里的"抽象基类"工具                             |
| `class BaseRetriever(ABC)`            | 继承 `ABC` 后，这个类**不能直接实例化**              |
| `@abstractmethod`                     | 标记"这个方法子类必须实现"，否则子类也无法实例化     |
| `@dataclass`                          | 自动生成 `__init__` 等样板代码的装饰器（少写十几行） |
| `...`                                 | 函数体占位符，表示"这里先空着"                       |

> **为什么要用 ABC？** 因为它会在**创建对象时**就报错，而不是运行到一半才发现"哦，这个方法没实现"。**错误越早暴露越好。**

## 8.3 关键词检索实现

```python
class KeywordRetriever(BaseRetriever):
    """基于 SQL LIKE 的关键词检索。"""

    def __init__(self, db: Session):
        self.db = db               # 把 Session 存起来，供 search 使用

    def search(self, query: str, top_k: int = 5) -> list[SearchResult]:
        stmt = (
            select(Chunk)
            .where(Chunk.text.contains(query))
            .order_by(Chunk.id)
            .limit(top_k)
        )
        return [
            SearchResult(
                chunk_id=c.id,
                text=c.text,
                filename=c.document.filename,
            )
            for c in self.db.scalars(stmt)
        ]
```

## 8.4 向量检索占位（第四册实现）

```python
class VectorRetriever(BaseRetriever):
    """语义检索：第四册实现。"""

    def search(self, query: str, top_k: int = 5) -> list[SearchResult]:
        raise NotImplementedError("向量检索将在第四册实现")
```

## 8.5 工厂函数：一处决定用哪种

```python
def get_retriever(db: Session, mode: str = "keyword") -> BaseRetriever:
    """根据配置返回对应的检索器。新增方式只需在这里加一行。"""
    if mode == "keyword":
        return KeywordRetriever(db)
    if mode == "vector":
        return VectorRetriever(db)
    raise ValueError(f"未知检索模式: {mode}")
```

**这就是多态的价值**：接口层写 `retriever.search(q)`，完全不知道底下是 SQL 还是向量。**换实现 = 改一行配置，业务代码零改动。**

### 本章只需记住 3 句话

1. 抽象基类 = `ABC` + `@abstractmethod`，强制子类实现方法。
2. 统一返回 `dataclass`，上层只认一种结构。
3. 工厂函数按配置返回实现，**换算法不改业务代码**。

---

# 第 9 章 测试

## 9.1 测试数据库不能用真实文件

三个理由：污染开发数据、测试之间互相干扰、跑得慢。

**用内存数据库：`sqlite://`（注意没有文件路径）。**

```python
# tests/conftest.py
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import StaticPool

from app.db.base import Base
from app.models.document import Chunk, Document  # noqa: F401


@pytest.fixture
def db():
    """每个测试一个全新的内存数据库。"""
    engine = create_engine(
        "sqlite://",                                  # ① 纯内存
        connect_args={"check_same_thread": False},    # ② 允许跨线程
        poolclass=StaticPool,                         # ③ 关键！
    )
    Base.metadata.create_all(bind=engine)

    TestingSession = sessionmaker(bind=engine)
    session = TestingSession()
    try:
        yield session
    finally:
        session.close()
        Base.metadata.drop_all(bind=engine)           # ④ 清干净
```

**为什么必须 `StaticPool`？**

内存数据库的生命周期跟着**连接**走。默认每次新建连接都是**另一个空数据库**，所以 `create_all` 在 A 连接建的表，测试用 B 连接时就不见了。`StaticPool` 让所有会话**共用同一个连接**，表才留得住。

> 这是 SQLite 内存测试的经典坑，记住配方即可。

## 9.2 测试用例

```python
# tests/test_repo.py
from app.repositories.document_repo import (
    delete_document,
    get_document,
    save_document_with_chunks,
    search_chunks,
)


def test_save_and_get(db):
    doc = save_document_with_chunks(db, "a.md", "第一句话。第二句话。", chunk_size=5)

    assert doc.id is not None                 # 自增 id 拿到了
    assert len(doc.chunks) > 0                # 块建好了
    assert get_document(db, doc.id).filename == "a.md"


def test_chunks_have_document_id(db):
    doc = save_document_with_chunks(db, "b.md", "测试内容。" * 20)
    assert all(c.document_id == doc.id for c in doc.chunks)   # 外键自动填了


def test_search_hits(db):
    save_document_with_chunks(db, "c.md", "部署方式使用 Docker。")
    results = search_chunks(db, "Docker")
    assert len(results) == 1
    assert "Docker" in results[0].text


def test_search_miss(db):
    save_document_with_chunks(db, "d.md", "完全无关的内容。")
    assert search_chunks(db, "不存在的词") == []


def test_delete_cascade(db):
    doc = save_document_with_chunks(db, "e.md", "删除测试。" * 10)
    doc_id = doc.id
    assert delete_document(db, doc_id) is True
    assert get_document(db, doc_id) is None


def test_same_filename_is_idempotent(db):
    save_document_with_chunks(db, "f.md", "第一版。")
    save_document_with_chunks(db, "f.md", "第二版。")
    from sqlalchemy import select
    from app.models.document import Document
    count = len(list(db.scalars(select(Document).where(Document.filename == "f.md"))))
    assert count == 1                          # 同名只留一份
```

**运行：**

```bash
pytest -v tests/test_repo.py
```

**测试设计原则回顾**：只测行为（"能存能查""同名只留一份"），不测实现（不关心你用了几个 SQL）。

### 本章只需记住 3 句话

1. 测试用内存库 `sqlite://`，`fixture` 负责建表和清理。
2. 内存库必须配 `StaticPool`，否则每次连接都是新库。
3. 用 `yield` 写 fixture：`yield` 前准备，`yield` 后清理。

---

# 第 10 章 接入 API

## 10.1 先定数据结构（老规矩）

```python
# app/schemas/document.py
from pydantic import BaseModel, Field


class DocumentCreate(BaseModel):
    """上传文档的请求体。"""
    filename: str = Field(min_length=1, max_length=255)
    content: str = Field(min_length=1)


class DocumentOut(BaseModel):
    """文档列表的响应体。"""
    id: int
    filename: str
    chunk_count: int

    model_config = {"from_attributes": True}   # ① 允许从 ORM 对象转过来


class SearchHit(BaseModel):
    chunk_id: int
    text: str
    filename: str
    score: float
```

| 代码                                       | 含义                                                         |
| ------------------------------------------ | ------------------------------------------------------------ |
| `Field(min_length=1)`                      | 字段级校验，不满足自动返回 422                               |
| `model_config = {"from_attributes": True}` | Pydantic v2 开关：允许 `DocumentOut.model_validate(orm_obj)` |

## 10.2 三个接口

```python
# app/api/v1/documents.py
from fastapi import APIRouter, Depends, Query
from sqlalchemy.orm import Session

from app.db.session import get_db
from app.repositories.document_repo import (
    delete_document,
    list_documents,
    save_document_with_chunks,
)
from app.schemas.document import DocumentCreate, DocumentOut, SearchHit
from app.services.retriever import get_retriever

router = APIRouter(prefix="/documents", tags=["文档"])


@router.post("", response_model=DocumentOut, summary="上传文档（纯文本）")
def create_doc(payload: DocumentCreate, db: Session = Depends(get_db)):
    doc = save_document_with_chunks(db, payload.filename, payload.content)
    return DocumentOut(
        id=doc.id, filename=doc.filename, chunk_count=len(doc.chunks)
    )


@router.get("", response_model=list[DocumentOut], summary="文档列表")
def get_docs(limit: int = 20, offset: int = 0, db: Session = Depends(get_db)):
    docs = list_documents(db, limit=limit, offset=offset)
    return [
        DocumentOut(id=d.id, filename=d.filename, chunk_count=len(d.chunks))
        for d in docs
    ]


@router.get("/search", response_model=list[SearchHit], summary="关键词检索")
def search(
    q: str = Query(min_length=1, description="检索关键词"),
    top_k: int = Query(5, ge=1, le=50),
    mode: str = Query("keyword", description="检索模式"),
    db: Session = Depends(get_db),
):
    retriever = get_retriever(db, mode=mode)     # ① 只认接口
    return retriever.search(q, top_k=top_k)      # ② 不关心底层实现


@router.delete("/{doc_id}", summary="删除文档")
def remove_doc(doc_id: int, db: Session = Depends(get_db)):
    ok = delete_document(db, doc_id)
    if not ok:
        raise NotFoundError(f"文档 {doc_id} 不存在")
    return {"deleted": True}
```

**逐条讲：**

| 代码                             | 含义                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| `APIRouter(prefix="/documents")` | 这个文件下所有接口都自动带 `/documents` 前缀                 |
| `payload: DocumentCreate`        | 请求体自动按模型校验                                         |
| `db: Session = Depends(get_db)`  | **依赖注入**：FastAPI 自动调 `get_db()`，把 Session 传进来，请求结束自动关闭 |
| `Query(min_length=1)`            | 查询参数的校验和文档说明                                     |
| `retriever.search(...)`          | 多态调用：不知道是关键词还是向量                             |

## 10.3 挂到主应用

```python
# app/main.py 里加一行
from app.api.v1 import documents, health

app.include_router(health.router, prefix="/api/v1")
app.include_router(documents.router, prefix="/api/v1")
```

## 10.4 动手验证

```bash
# 1) 上传
curl -X POST http://127.0.0.1:8000/api/v1/documents \
  -H "Content-Type: application/json" \
  -d '{"filename":"demo.md","content":"本项目使用 Docker 部署。Docker 很方便。"}'

# 2) 列表
curl http://127.0.0.1:8000/api/v1/documents

# 3) 检索
curl "http://127.0.0.1:8000/api/v1/documents/search?q=Docker&top_k=3"
```

### 本章只需记住 3 句话

1. 出参模型加 `from_attributes=True`，才能从 ORM 对象转。
2. `Depends(get_db)` 自动管理 Session 生命周期。
3. 接口只调 repository 和 retriever，**不写一句 SQL**。

---

# 第 11 章 自测（10 题）

1. 主键和外键分别是什么？为什么 chunks 表需要 `document_id`？
2. ORM 里"类、实例、属性"分别对应数据库的什么？
3. `__tablename__` 不写会怎样？
4. `db.add()` 和 `db.commit()` 的区别？
5. 为什么批量插入不能循环 `commit`？
6. `.where(Chunk.text.contains(q))` 翻译成 SQL 是什么？
7. 分页时为什么必须写 `order_by`？
8. `ABC` + `@abstractmethod` 解决了什么问题？
9. 测试为什么用 `sqlite://` 而不是真实文件？`StaticPool` 又是干什么的？
10. `Depends(get_db)` 在什么时候创建和关闭 Session？

<details>
<summary>答案要点</summary>

1. 主键唯一标识一行；外键指向另一张表的主键。`document_id` 让每个块知道属于哪个文档，实现一对多。
2. 类 ↔ 表，实例 ↔ 一行，属性 ↔ 列。
3. 会报错（必须声明表名），SQLAlchemy 不知道往哪张表存。
4. `add` 只是加入待写入队列（内存中），`commit` 才真正执行 SQL 写入磁盘。
5. 每次 commit 都是一次磁盘同步，1000 条 = 1000 次开销；而且中途失败会留下半截数据。应 `add_all` + 一次 commit。
6. `WHERE text LIKE '%q%'`（参数化，防注入）。
7. 数据库不保证返回顺序，不排序则每页结果可能重复或遗漏。
8. 强制子类实现接口；没实现时创建对象就报错，而不是运行到一半才崩。
9. 内存库更快、不污染真实数据、测试间隔离。`StaticPool` 让所有会话共用一个连接，否则每次连接都是一个新的空内存库，表会"消失"。
10. 每个请求进来时创建 Session，请求结束（`finally`）自动关闭。

</details>

---

# 附录 本册知识地图

| 知识点   | 本册落点                                               |
| -------- | ------------------------------------------------------ |
| 数据库   | 第 1 章 表/主键/外键、第 3 章 建模、第 7 章 查询与索引 |
| OOP      | 第 3 章 模型类、第 8 章 抽象基类与多态                 |
| 异常处理 | 第 5 章 事务回滚、第 10 章 抛业务异常                  |
| 数据结构 | 第 6 章 `list[Chunk]`、第 8 章 `dataclass`             |
| 函数结构 | 第 5 章 repository 函数、第 8 章工厂函数               |
| 高级特性 | 第 4 章 `yield` 依赖注入                               |
| Web 开发 | 第 10 章 路由、依赖注入、Pydantic 模型                 |
| 测试调试 | 第 9 章 内存库 fixture、行为测试                       |
| 项目结构 | 全册贯穿：api / repository / model / service 分层      |

---

# 下一册预告

**第四册：向量化与语义检索**

- 什么是 embedding（把文字变成一串数字）
- `Chroma` 本地向量库的增删查
- 实现第三册留的 `VectorRetriever`
- 混合检索（关键词 + 向量）与结果融合
- 用 mock 测外部 API 调用

> 建议顺序：**先把本册第 5、6 章跑通（能存），再跑第 7、10 章（能查），最后补第 9 章测试。** 卡在哪一步告诉我。