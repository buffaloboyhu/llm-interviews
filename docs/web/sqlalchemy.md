---
sidebar_position: 3
---

SQLAlchemy 是 Python 中最著名的 ORM（对象关系映射）框架，它允许你使用 Python 类和对象来操作数据库，而无需编写繁琐的 SQL 语句。2.x 版本相比 1.x 版本，在异步支持和类型提示（Type Hinting）上有了巨大的改进。

本教程将带你从环境搭建到核心概念，再到实战代码，循序渐进地掌握它。

---

### 🚀 第一部分：环境准备与核心概念

在开始写代码之前，我们需要理解几个核心概念，这将帮助你更好地构建数据库应用。

#### 1. 安装
使用 pip 安装 SQLAlchemy：
```bash
pip install sqlalchemy
```
*如果你使用 MySQL 或 PostgreSQL，还需要安装对应的驱动（如 `pymysql` 或 `psycopg2`）。*

#### 2. 核心概念速览
| 概念 | 作用 | 形象理解 |
| :--- | :--- | :--- |
| **Engine** | 数据库连接管理器 | 它是通往数据库的“桥梁”或“管道”。 |
| **Base** | 模型基类 | 所有数据库表对应的 Python 类的“祖先”。 |
| **Model** | 数据模型 | 对应数据库中的“表”，类属性对应“列”。 |
| **Session** | 事务会话 | 它是你与数据库交互的“工作台”，负责增删改查。 |

---

### 🛠️ 第二部分：基础搭建 (连接与建模)

我们将以 **SQLite** 为例（无需安装数据库软件），演示如何建立连接并定义一张“用户表”。

#### 1. 建立连接与定义模型
```python
from sqlalchemy import create_engine, Column, Integer, String, ForeignKey
from sqlalchemy.orm import declarative_base, sessionmaker, relationship

# 1. 创建引擎 (Echo=True 会打印生成的 SQL 语句，方便调试)
# SQLite 语法: sqlite:///文件名.db
engine = create_engine("sqlite:///test.db", echo=True)

# 2. 创建基类
Base = declarative_base()

# 3. 定义用户模型 (对应数据库中的 users 表)
class User(Base):
    __tablename__ = "users"  # 表名

    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String(50), nullable=False)
    email = Column(String(100), unique=True)

    # 定义一对多关系：一个用户有多个订单
    # back_populates 用于建立双向关联
    orders = relationship("Order", back_populates="user")

    def __repr__(self):
        return f"<User(id={self.id}, name='{self.name}', email='{self.email}')>"

# 4. 创建会话工厂
SessionLocal = sessionmaker(bind=engine, autoflush=False, autocommit=False)

# 5. 创建所有表 (如果表中不存在)
Base.metadata.create_all(bind=engine)
```

---

### 📝 第三部分：CRUD 核心操作 (增删改查)

这是日常开发中最常用的部分。在 SQLAlchemy 2.x 中，推荐使用 `select()` 语句进行查询。

#### 1. 新增数据 (Create)
支持单条插入和批量插入。

```python
def create_data():
    # 获取会话
    with SessionLocal() as session:
        # --- 单条插入 ---
        user1 = User(name="张三", email="zhang@example.com")
        session.add(user1)
        
        # --- 批量插入 ---
        users = [
            User(name="李四", email="li@example.com"),
            User(name="王五", email="wang@example.com")
        ]
        session.add_all(users)
        
        # 提交事务 (必须执行，否则数据不会保存)
        session.commit()
        print("✅ 数据插入成功")

create_data()
```

#### 2. 查询数据 (Read)
SQLAlchemy 2.x 使用 `select()` 构造查询语句。

```python
from sqlalchemy import select

def read_data():
    with SessionLocal() as session:
        # --- 查询所有 ---
        stmt = select(User)
        results = session.execute(stmt).scalars().all()
        print(f"所有用户: {results}")

        # --- 条件查询 (WHERE) ---
        # 查询名字为 '张三' 的用户
        stmt = select(User).where(User.name == "张三")
        user = session.execute(stmt).scalar_one_or_none() # 获取单个结果
        if user:
            print(f"找到用户: {user}")

        # --- 多条件查询 ---
        # 查询 ID > 1 且 名字包含 '四' 的用户
        stmt = select(User).where(User.id > 1, User.name.like("%四%"))
        users = session.execute(stmt).scalars().all()
        print(f"筛选结果: {users}")

read_data()
```

#### 3. 更新数据 (Update)
通常先查询出对象，修改属性后提交。

```python
def update_data():
    with SessionLocal() as session:
        # 1. 查询对象
        stmt = select(User).where(User.name == "张三")
        user = session.execute(stmt).scalar_one()
        
        # 2. 修改属性
        user.email = "zhang_new@example.com"
        
        # 3. 提交
        session.commit()
        print(f"✅ 更新后邮箱: {user.email}")

update_data()
```

#### 4. 删除数据 (Delete)

```python
def delete_data():
    with SessionLocal() as session:
        # 1. 查询要删除的对象
        stmt = select(User).where(User.name == "王五")
        user = session.execute(stmt).scalar_one()
        
        # 2. 删除对象
        session.delete(user)
        
        # 3. 提交
        session.commit()
        print("✅ 删除成功")

delete_data()
```

---

### 🔗 第四部分：进阶关系与事务管理

#### 1. 一对多关系 (One-to-Many)
假设我们有一个 `Order` (订单) 表，每个订单属于一个用户。

```python
# 定义 Order 模型 (接在 User 模型后面)
class Order(Base):
    __tablename__ = "orders"
    id = Column(Integer, primary_key=True)
    order_no = Column(String(20))
    user_id = Column(Integer, ForeignKey("users.id")) # 外键

    # 关联回 User
    user = relationship("User", back_populates="orders")

# 记得重新创建表以应用新模型 (实际开发中通常使用 Alembic 迁移)
Base.metadata.create_all(engine)

def relation_example():
    with SessionLocal() as session:
        # 1. 获取用户
        user = session.execute(select(User).where(User.name == "张三")).scalar_one()
        
        # 2. 创建订单并关联
        new_order = Order(order_no="ORD_001", user=user)
        session.add(new_order)
        session.commit()
        
        # 3. 查询关联数据 (自动加载)
        # 访问 user.orders 即可获取该用户的所有订单
        print(f"张三的订单: {[o.order_no for o in user.orders]}")

relation_example()
```

#### 2. 事务管理 (Transaction)
在涉及多个步骤的操作时（如转账），必须保证原子性。如果中间出错，需要回滚。

```python
def transaction_example():
    with SessionLocal() as session:
        try:
            # 模拟业务逻辑
            user = session.execute(select(User).where(User.id == 1)).scalar_one()
            user.name = "新名字"
            
            # 模拟一个错误
            # raise Exception("模拟系统故障")
            
            session.commit() # 一切正常则提交
        except Exception as e:
            session.rollback() # 发生错误则回滚，数据保持不变
            print(f"❌ 操作失败，已回滚: {e}")

transaction_example()
```

---

### 💡 第五部分：最佳实践与性能优化

1.  **批量操作**：
    插入大量数据时，使用 `session.add_all()` 比循环 `add` 效率高得多。对于超大数据量，可以每 1000 条提交一次。

2.  **懒加载与急加载**：
    *   **懒加载 (Lazy Loading)**：默认行为。访问关联对象（如 `user.orders`）时才发送 SQL 查询。
    *   **急加载 (Eager Loading)**：使用 `joinedload`。在查询主表时通过 `JOIN` 一次性把关联数据查出来，避免 "N+1 查询问题"。
    ```python
    from sqlalchemy.orm import joinedload
    
    # 一次性查出用户及其所有订单
    stmt = select(User).options(joinedload(User.orders))
    users = session.execute(stmt).scalars().unique().all()
    ```

3.  **原生 SQL 执行**：
    如果 ORM 无法满足复杂的查询需求，可以直接执行原生 SQL。
    ```python
    with SessionLocal() as session:
        result = session.execute("SELECT * FROM users WHERE age > :age", {"age": 25})
        for row in result:
            print(row)
    ```

这份教程涵盖了 SQLAlchemy 2.x 的核心骨架。建议你在本地创建一个 `.py` 文件，把代码复制进去运行，通过修改参数来观察控制台打印的 SQL 语句，这是学习 ORM 最快的方式！