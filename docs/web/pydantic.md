---
sidebar_position: 1
---

# Pydantic 教程

Pydantic 是 Python 生态中最主流的数据验证与序列化库。它利用 Python 的类型注解（Type Hints）来实现“声明式”的数据校验，广泛应用于 FastAPI、Django 等现代 Web 框架中，也是管理应用配置的神器。

本教程基于目前主流的 **Pydantic V2** 版本编写，带你从入门到实战全面掌握。

---

### 🚀 第一部分：快速开始

#### 1. 安装
推荐使用 pip 进行安装。如果你需要邮箱验证等扩展功能，可以安装完整版。

```bash
# 基础安装
pip install pydantic

# 完整安装（包含 email 验证等）
pip install pydantic[email]
```

#### 2. 核心概念：BaseModel
Pydantic 的核心是 `BaseModel` 类。你只需要定义一个继承自 `BaseModel` 的类，并使用 Python 的类型注解来声明字段，Pydantic 就会自动为你处理数据验证、序列化和反序列化。

**它的核心能力包括：**
*   **类型检查**：确保数据符合定义的类型。
*   **数据转换**：尝试将输入数据转换为指定类型（例如将字符串 `"123"` 转为整数 `123`）。
*   **错误处理**：数据不合法时抛出清晰的 `ValidationError`。

---

### 🛠️ 第二部分：基础用法详解

#### 1. 定义模型与字段
我们可以定义必填字段、选填字段（带默认值）以及复杂类型。

```python
from datetime import datetime
from typing import Optional, List
from pydantic import BaseModel, Field

class User(BaseModel):
    # 必填字段
    id: int
    username: str
    
    # 选填字段（默认为 None）
    email: Optional[str] = None
    
    # 带默认值的字段
    age: int = 18
    
    # 复杂类型：列表
    hobbies: List[str] = []
    
    # 使用 Field 进行更详细的约束
    # min_length: 最小长度, pattern: 正则表达式
    password: str = Field(..., min_length=6, description="用户密码")
```

#### 2. 数据验证与“宽进严出”
Pydantic V2 的核心理念是**解析（Parsing）**。它不仅校验类型，还会尝试进行智能转换。

```python
# 模拟从 API 接收到的数据（注意 id 是字符串，age 是字符串）
raw_data = {
    "id": "1001",          # 字符串，但会被自动转为 int
    "username": "zhangsan",
    "age": "25",           # 字符串，会被转为 int
    "hobbies": ["coding", "reading"],
    "password": "123456"
}

try:
    user = User(**raw_data)
    print("验证成功！")
    print(f"ID 类型: {type(user.id)}")  # <class 'int'>
    print(f"用户信息: {user}")
except Exception as e:
    print("验证失败:", e)
```

**输出结果：**
```text
验证成功！
ID 类型: <class 'int'>
用户信息: id=1001 username='zhangsan' email=None age=25 hobbies=['coding', 'reading'] password='123456'
```

#### 3. 处理验证错误
当数据不符合规则时，Pydantic 会抛出详细的错误信息，帮助你快速定位问题。

```python
from pydantic import ValidationError

bad_data = {"id": "abc", "username": "li", "password": "123"} # id不是数字，密码太短

try:
    User(**bad_data)
except ValidationError as e:
    # e.json() 可以输出格式化的错误详情
    print(e.json(indent=2))
```

---

### ⚙️ 第三部分：进阶功能

#### 1. 常用配置 (model_config)
在 Pydantic V2 中，配置通过 `model_config` 类属性进行统一管理，取代了 V1 的 `class Config`。

```python
from pydantic import ConfigDict

class Product(BaseModel):
    model_config = ConfigDict(
        str_strip_whitespace=True,  # 自动去除字符串首尾空格
        extra="forbid",             # 禁止传入未定义的字段（默认是 ignore）
        frozen=False                # 是否冻结模型（不可变）
    )
    
    name: str
    price: float
```

#### 2. 自定义验证器
除了类型检查，你还可以编写自定义逻辑来验证数据（例如检查密码强度或跨字段验证）。V2 推荐使用 `@field_validator`。

```python
from pydantic import field_validator

class UserRegister(BaseModel):
    username: str
    password: str
    confirm_password: str

    # 单字段验证
    @field_validator('username')
    @classmethod
    def username_must_not_start_with_number(cls, v):
        if v[0].isdigit():
            raise ValueError('用户名不能以数字开头')
        return v

    # 多字段/跨字段验证 (model_validator)
    # 注意：这里仅作演示，实际逻辑需结合 model_validator 处理
```

#### 3. 序列化与反序列化
Pydantic 提供了强大的方法将模型转换为字典或 JSON。

| 方法 | 作用 | 示例 |
| :--- | :--- | :--- |
| **model_dump()** | 转为 Python 字典 | `user.model_dump()` |
| **model_dump_json()** | 转为 JSON 字符串 | `user.model_dump_json(indent=2)` |
| **model_validate()** | 从字典实例化模型 | `User.model_validate(data_dict)` |
| **model_validate_json()** | 从 JSON 字符串实例化 | `User.model_validate_json(json_str)` |

---

### 📊 第四部分：Pydantic V1 vs V2 关键差异

如果你是从旧版本迁移过来，或者阅读旧教程，请注意以下关键区别：

| 特性 | Pydantic V1 | Pydantic V2 (当前标准) |
| :--- | :--- | :--- |
| **配置方式** | `class Config:` 内部类 | `model_config = ConfigDict(...)` |
| **核心方法** | `.dict()`, `.json()` | `.model_dump()`, `.model_dump_json()` |
| **验证器** | `@validator` | `@field_validator` (推荐) |
| **性能** | 纯 Python 实现 | 核心使用 **Rust** 重写，速度提升 5-50 倍 |

---

### 💡 总结与建议

1.  **拥抱 V2**：新项目请直接使用 Pydantic V2，性能更强且 API 更规范。
2.  **类型提示是关键**：写好 Python 类型注解（如 `List[str]`, `Optional[int]`）是发挥 Pydantic 威力的前提。
3.  **与 FastAPI 结合**：如果你做 Web 开发，Pydantic 是 FastAPI 的基石，定义好模型后，接口文档（Swagger UI）会自动生成。

希望这份教程能帮你快速上手 Pydantic！如果有具体的代码场景需要调试，随时发给我。
