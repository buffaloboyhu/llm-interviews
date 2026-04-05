---
sidebar_position: 3
---

### 📚 第一阶段：核心概念与架构

在写代码之前，必须理解 SQLAlchemy 的“四层架构”，这能帮你解决 90% 的疑惑。

| 概念 | 对应类/对象 | 作用 | 比喻 |
| :--- | :--- | :--- | :--- |
| **引擎 (Engine)** | `create_engine` | 数据库连接工厂，管理连接池 | **自来水厂**（负责供水，不直接喝水） |
| **会话 (Session)** | `sessionmaker` | 与数据库交互的“工作台”，暂存对象 | **水桶**（从水厂接水，在这里洗洗涮涮，最后倒进下水道或存起来） |
| **基类 (Base)** | `DeclarativeBase` | 所有模型类的父类，持有元数据 | **建筑图纸**（定义了房子长什么样） |
| **模型 (Model)** | `class User(Base)` | 映射数据库表的 Python 类 | **具体的房子**（根据图纸盖出来的实体） |

---

### 🛠️ 第二阶段：环境搭建与模型定义 (ORM)

这是你之前报错的根源所在。我们需要定义模型，并正确创建表。

#### 1. 基础配置 (`database.py`)
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, DeclarativeBase

# 1. 创建引擎 (方言+驱动://用户:密码@主机:端口/库名)
# echo=True 会在控制台打印生成的 SQL，调试神器
engine = create_engine(
    "postgresql+psycopg2://user:password@localhost/mydb", 
    echo=True 
)

# 2. 创建会话工厂
SessionLocal = sessionmaker(bind=engine, autoflush=False, autocommit=False)

# 3. 定义基类
class Base(DeclarativeBase):
    pass
```

#### 2. 定义模型 (`models.py`)
注意：定义类不会自动建表，必须配合 `Base.metadata.create_all` 或 Alembic。

```python
from sqlalchemy import Column, Integer, String, DateTime, ForeignKey, func
from sqlalchemy.orm import relationship
from database import Base

class User(Base):
    __tablename__ = "users"  # 数据库中的表名

    id = Column(Integer, primary_key=True, index=True, autoincrement=True)
    name = Column(String(50), unique=True, nullable=False, index=True) # 索引加速查询
    email = Column(String(100), unique=True, nullable=False)
    age = Column(Integer, nullable=True)
    created_at = Column(DateTime, server_default=func.now()) # 数据库自动生成时间

    # 一对多关系：一个用户有多个订单
    # back_populates 实现双向绑定
    orders = relationship("Order", back_populates="owner", cascade="all, delete-orphan")

class Order(Base):
    __tablename__ = "orders"
    
    id = Column(Integer, primary_key=True, index=True)
    order_no = Column(String(20), nullable=False)
    user_id = Column(Integer, ForeignKey("users.id")) # 外键

    # 多对一关系
    owner = relationship("User", back_populates="orders")
```

#### 3. 解决“表不存在”
在第一次运行前，执行一次建表操作（开发环境）：
```python
# 在 main.py 或初始化脚本中运行一次
from database import engine, Base
from models import User, Order # 必须导入模型，否则 Base 不知道有哪些表

# 这行代码会检查数据库，如果表不存在则创建
Base.metadata.create_all(bind=engine)
```

---

### 💻 第三阶段：CRUD 核心操作 (增删改查)

SQLAlchemy 2.0 强制推荐使用 `select()` 语句，而不是旧的 `.query()`。

#### 1. 封装数据库会话
为了避免连接泄露，建议使用上下文管理器或依赖注入。

```python
from sqlalchemy.orm import Session

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

#### 2. 增 (Create)
```python
def create_user(db: Session, name: str, email: str):
    # 实例化对象
    db_user = User(name=name, email=email)
    db.add(db_user)       # 加入会话（暂存）
    db.commit()           # 提交事务（真正写入数据库）
    db.refresh(db_user)   # 刷新对象（获取数据库生成的 id 等字段）
    return db_user
```

#### 3. 查 (Read) - 2.0 新风格
```python
from sqlalchemy import select

def get_user(db: Session, user_id: int):
    # 方式一：主键查询（最快）
    return db.get(User, user_id)

def search_users(db: Session, name: str = None, min_age: int = None):
    # 方式二：构建查询语句
    stmt = select(User)
    
    # 动态添加条件
    if name:
        stmt = stmt.where(User.name.contains(name)) # 模糊查询
    if min_age:
        stmt = stmt.where(User.age >= min_age)
        
    # 执行查询
    result = db.execute(stmt).scalars().all() # scalars() 提取模型对象
    return result
```

#### 4. 改 (Update)
```python
def update_user(db: Session, user_id: int, email: str):
    # 先查后改
    user = db.get(User, user_id)
    if user:
        user.email = email
        db.commit()
        db.refresh(user)
    return user
```

#### 5. 删 (Delete)
```python
def delete_user(db: Session, user_id: int):
    user = db.get(User, user_id)
    if user:
        db.delete(user)
        db.commit()
        return True
    return False
```

---

### 🔗 第四阶段：关联查询与性能优化

#### 1. 关联查询 (Join & Relationship)
假设你要查询“张三的所有订单”。

*   **懒加载 (Lazy Loading)**：默认行为。访问 `user.orders` 时才去查数据库（会产生 N+1 问题）。
*   **急加载 (Eager Loading)**：一次性把用户和订单都查出来。

```python
from sqlalchemy.orm import joinedload

def get_user_with_orders(db: Session, user_id: int):
    # 使用 joinedload 预加载 orders
    stmt = select(User).options(joinedload(User.orders)).where(User.id == user_id)
    result = db.execute(stmt).scalars().first()
    return result
```

#### 2. 事务控制 (Transaction)
数据库操作必须保证原子性。

```python
def transfer_money(db: Session, from_id, to_id, amount):
    try:
        user_a = db.get(User, from_id)
        user_b = db.get(User, to_id)
        
        user_a.age -= amount # 假设用 age 模拟余额
        user_b.age += amount
        
        db.commit() # 成功则提交
    except Exception as e:
        db.rollback() # 失败则回滚，保证数据一致性
        print(f"转账失败: {e}")
        raise
```

---

### 🚀 第五阶段：进阶与最佳实践

#### 1. 批量操作
一次性插入或更新大量数据时，不要循环 `commit`。

```python
users = [User(name=f"User{i}", email=f"user{i}@test.com") for i in range(1000)]
db.add_all(users)
db.commit() # 一次性提交
```

#### 2. 执行原生 SQL
当 ORM 无法满足复杂查询时，可以直接执行 SQL。

```python
result = db.execute("SELECT * FROM users WHERE age > :age", {"age": 25})
rows = result.fetchall()
```

#### 3. 生产环境建议
1.  **连接池**：生产环境务必配置 `pool_size` 和 `max_overflow`。
2.  **迁移工具**：绝对不要在生产环境用 `create_all`，请使用 **Alembic** 管理表结构变更。
3.  **类型提示**：SQLAlchemy 2.0 对类型提示支持很好，建议配合 Mypy 使用。

### 📌 学习路线总结
1.  **Day 1**: 理解 Engine/Session/Base，跑通 `create_all`，定义 User 模型。
2.  **Day 2**: 练习 CRUD，重点掌握 `select().where()` 和 `session.commit()`。
3.  **Day 3**: 学习一对多关系（User-Order），理解 `ForeignKey` 和 `relationship`。
4.  **Day 4**: 引入 **Alembic**，彻底抛弃手动建表，学习版本控制。

按照这个结构学习，你就能从“写报错代码”进阶到“构建稳健的数据库应用”了。