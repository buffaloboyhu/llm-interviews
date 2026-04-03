---
sidebar_position: 3
---

# SQLAchemy

> 作者：资深软件工程师
> 适用人群：Python 开发者、后端工程师、数据工程师、全栈开发者
> 技术栈：Python + SQLAlchemy 2.0+ + SQLite / PostgreSQL / MySQL

---

## 🌟 一、前言：为什么选择 SQLAlchemy？

**SQLAlchemy 的定位**：
- Python 中最强大的 **ORM（对象关系映射）** 框架
- 支持 **SQL 表达式语言（Core）** 和 **ORM（对象映射）** 两层抽象
- 跨数据库兼容：支持 SQLite、PostgreSQL、MySQL、Oracle 等
- 高度灵活，开发效率高，适合中小型项目到大型系统

> ✅ 适合场景：
> - Web 后端（Flask/Django/FastAPI）
> - 数据清洗与 ETL
> - 微服务中的持久层
> - 需要自定义 SQL 但又想使用 ORM 的项目

---

## 📚 二、环境准备与安装

### 1. 安装 SQLAlchemy

```bash
pip install sqlalchemy
```

> 建议使用 **SQLAlchemy 2.0+**（目前主流版本，支持异步、新 API）

### 2. 安装数据库驱动（示例使用 SQLite、PostgreSQL）

```bash
# SQLite（无需额外驱动，内置）
pip install sqlalchemy

# PostgreSQL
pip install psycopg2-binary

# MySQL
pip install PyMySQL

# Oracle（可选）
pip install oracledb
```

---

## 🔧 三、核心概念详解

### 1. SQLAlchemy 的两大核心

| 模块 | 功能 | 适用场景 |
|------|------|----------|
| **SQL Expression Language (Core)** | 原生 SQL 表达式，无需 ORM 类 | 复杂查询、性能敏感、DSL 编写 |
| **ORM (Object-Relational Mapping)** | 数据类 → 表，实例 → 行 | 快速开发、模型驱动项目 |

> ✅ 推荐：通常使用 ORM，但掌握 Core 更灵活。

---

## 📦 四、从零开始：定义模型（ORM）

### 1. 基本模型结构

```python
from sqlalchemy import create_engine, Column, Integer, String, DateTime, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from datetime import datetime

# 1. 创建基类
Base = declarative_base()

# 2. 定义模型类（对应数据库表）
class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    name = Column(String(50), nullable=False)
    email = Column(String(100), unique=True, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)

    def __repr__(self):
        return f"<User(name='{self.name}', email='{self.email}')>"
```

---

### 2. 创建数据库连接与表

```python
# 创建引擎（连接字符串）
engine = create_engine("sqlite:///example.db", echo=True)  # echo=True 打印 SQL

# 创建所有表
Base.metadata.create_all(engine)
```

> ✅ `echo=True` 用于调试，生产环境应关闭。

---

## 🧪 五、CRUD 操作实战（Session & Transaction）

### 1. 创建 Session

```python
# 创建会话
Session = sessionmaker(bind=engine)
session = Session()
```

### 2. 增（Create）

```python
# 新增单条
user = User(name="Alice", email="alice@example.com")
session.add(user)
session.commit()  # 提交事务

# 批量添加
users = [
    User(name="Bob", email="bob@example.com"),
    User(name="Charlie", email="charlie@example.com")
]
session.add_all(users)
session.commit()
```

> ⚠️ **注意**：`commit()` 才真正持久化，`rollback()` 可回滚。

---

### 3. 查（Read）

#### 基本查询

```python
# 查询所有用户
all_users = session.query(User).all()

# 条件查询
alice = session.query(User).filter(User.name == "Alice").first()

# 多条件
users = session.query(User).filter(
    User.name.like("A%"),
    User.email.endswith("@example.com")
).all()

# 按 ID 查询
user = session.query(User).get(1)
```

#### 排序与分页

```python
# 排序
users = session.query(User).order_by(User.created_at.desc()).all()

# 分页
from sqlalchemy import desc
page = 1
limit = 10
users = session.query(User).order_by(desc(User.id)).offset((page - 1) * limit).limit(limit).all()
```

---

### 4. 改（Update）

```python
# 更新一条
user = session.query(User).filter(User.name == "Alice").first()
if user:
    user.email = "alice_new@example.com"
    session.commit()
```

### 5. 删（Delete）

```python
user = session.query(User).filter(User.name == "Charlie").first()
if user:
    session.delete(user)
    session.commit()
```

---

## 🔗 六、关系映射（Relationships）

### 1. 一对一（One-to-One）

```python
class Profile(Base):
    __tablename__ = 'profiles'

    id = Column(Integer, primary_key=True)
    bio = Column(String(200))
    user_id = Column(Integer, ForeignKey('users.id'), unique=True)

    user = relationship("User", back_populates="profile")

# User 类补充
class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    name = Column(String(50), nullable=False)
    email = Column(String(100), unique=True, nullable=False)
    profile = relationship("Profile", back_populates="user")
```

> 外键 `user_id` 必须唯一才能实现 One-to-One。

---

### 2. 一对多（One-to-Many）

```python
class Post(Base):
    __tablename__ = 'posts'

    id = Column(Integer, primary_key=True)
    title = Column(String(100))
    user_id = Column(Integer, ForeignKey('users.id'))

    user = relationship("User", backref="posts")  # backref 自动创建反向关系
```

> 查询用户的所有文章：
```python
user = session.query(User).get(1)
print(user.posts)
```

---

### 3. 多对多（Many-to-Many）

```python
# 中间表（关联表）
association_table = Table(
    'user_roles', Base.metadata,
    Column('user_id', Integer, ForeignKey('users.id')),
    Column('role_id', Integer, ForeignKey('roles.id'))
)

class Role(Base):
    __tablename__ = 'roles'
    id = Column(Integer, primary_key=True)
    name = Column(String(50), unique=True)
    users = relationship("User", secondary=association_table, back_populates="roles")

class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    name = Column(String(50))
    roles = relationship("Role", secondary=association_table, back_populates="users")
```

> 使用：
```python
# 创建角色
admin_role = Role(name="admin")
user = User(name="Alice")
user.roles.append(admin_role)

session.add(user)
session.commit()
```

---

## 📊 七、进阶功能（高级查询与表达式）

### 1. 复合条件（AND/OR/NOT）

```python
from sqlalchemy import and_, or_, not_

users = session.query(User).filter(
    and_(
        User.name.contains("A"),
        or_(User.email.endswith("@gmail.com"), User.email.endswith("@yahoo.com"))
    )
).all()
```

### 2. 使用函数与聚合

```python
from sqlalchemy import func

count = session.query(func.count(User.id)).scalar()  # 返回 COUNT(*)
avg_age = session.query(func.avg(User.id)).scalar()

# 分组统计
result = session.query(
    User.name,
    func.count(User.id).label("post_count")
).join(Post).group_by(User.name).all()
```

### 3. 子查询

```python
subquery = session.query(User).filter(User.email.like("%@example.com")).subquery()

users = session.query(User).filter(User.id.in_(subquery.c.id)).all()
```

> ✅ 子查询可嵌套多层，适合复杂分析。

---

## ⚙️ 八、异步支持（SQLAlchemy 2.0+）

### 1. 异步引擎 + 会话

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

# 异步引擎（支持 PostgreSQL、MySQL）
engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/dbname")

# 会话工厂
AsyncSessionLocal = async_sessionmaker(bind=engine, expire_on_commit=False)

# 在异步函数中使用
async def get_user(user_id: int):
    async with AsyncSessionLocal() as session:
        result = await session.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalars().first()
```

> ✅ 在 FastAPI 中非常推荐使用异步模式。

---

## 🛡️ 九、事务与并发控制

### 1. 事务管理（推荐使用上下文管理器）

```python
try:
    with session.begin():  # 自动 commit 或 rollback
        user = User(name="David", email="david@example.com")
        session.add(user)
except Exception as e:
    print(f"Transaction failed: {e}")
```

> `session.begin()` 是推荐方式，避免手动 `commit/rollback`。

---

## 🚀 十、性能优化技巧

| 优化点 | 建议 |
|-------|------|
| **N+1 查询问题** | 使用 `joinedload` 或 `selectinload` |
| **减少数据库查询** | 使用 `session.query().options(joinedload(...))` |
| **批量操作** | 用 `session.execute()` + `Values` 代替 `add_all` |
| **缓存机制** | 结合 `session.expire_all()` 或 LRU 缓存 |
| **避免重复查询** | 建立唯一索引（`unique=True`） |

### 示例：避免 N+1（高效加载）

```python
# ❌ 低效（N+1）
users = session.query(User).all()
for user in users:
    print(user.profile.bio)  # 每个 user 都查一次 profile

# ✅ 高效（预先加载）
from sqlalchemy.orm import joinedload

users = session.query(User).options(joinedload(User.profile)).all()
for user in users:
    print(user.profile.bio)
```

---

## 🧰 十一、与其他框架集成

### 1. 与 Flask 集成

```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy  # 使用 Flask-SQLAlchemy 封装

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///flask.db'

db = SQLAlchemy(app)

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(50), nullable=False)
```

> ✅ 适用中小型 Flask 项目，若需复杂控制建议直接用 SQLAlchemy。

---

### 2. 与 FastAPI 集成

```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

async def get_db() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        yield session
```

使用依赖注入：

```python
@app.get("/users")
async def read_users(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User))
    return result.scalars().all()
```

---

## 📌 十二、最佳实践总结

| 最佳实践 | 说明 |
|--------|------|
| ✅ 使用 `declarative_base()` 定义模型 | 更清晰、可维护 |
| ✅ 所有模型统一继承 `Base` | 便于管理 |
| ✅ 合理使用关系和 `backref`/`relationship` | 提高可读性 |
| ✅ 使用上下文管理器管理事务 | 防止资源泄露 |
| ✅ 选择合适的加载策略（`joinedload` vs `selectinload`） | 提升性能 |
| ✅ 对高频查询建立索引 | 避免全表扫描 |
| ✅ 生产环境关闭 `echo=True` | 提升性能 |
| ✅ 使用异步驱动（PostgreSQL/MySQL） | 适合高并发 |

---

## 📚 十三、学习资源推荐

| 类型 | 资源 |
|------|------|
| 官方文档 | [https://docs.sqlalchemy.org](https://docs.sqlalchemy.org) |
| 书籍 | *SQLAlchemy 2.0 Cookbook*（2024） |
| 教程 | [SQLAlchemy Tutorials (A Flask ORM Guide)](https://github.com/zzheng0518/flask-sqlalchemy-tutorial) |
| GitHub 项目 | [https://github.com/davidfischer/async-sqlalchemy-example](https://github.com/davidfischer/async-sqlalchemy-example) |
| 视频课程 | YouTube “SQLAlchemy ORM Deep Dive” by Coding in Python |

---

## ✅ 总结

SQLAlchemy 是 Python 中最强大、最灵活的 ORM 框架，掌握它意味着：

- 可以高效开发数据库应用
- 可以编写复杂 SQL 查询而不写原生 SQL
- 可以轻松支持多种数据库
- 可以与 Flask/FastAPI 等主流框架无缝集成
- 能写出高性能、可维护、可扩展的持久层代码

---

> 📌 **学习路径建议**：
> 1. 1 天：基础语法 + 模型定义 + CRUD
> 2. 1 天：关系映射 + 复杂查询
> 3. 1 天：异步 + 性能优化
> 4. 1 天：实战项目 → 用 SQLAlchemy 写一个用户博客系统（含登录、文章、评论）

---

📬 **如果你有具体项目需求、报错问题、性能优化案例，欢迎继续提问！**

我将作为你的技术导师，提供 **真正落地的解决方案**。💪

---
✅ **掌握 SQLAlchemy，就是掌握 Python 后端核心能力！**