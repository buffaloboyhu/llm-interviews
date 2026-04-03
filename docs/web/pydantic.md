---
sidebar_position: 1
---


# Pydantic

> ✅ 适用于：Python 3.9+，Pydantic v2（2023 年发布，全面支持类型提示）
>
> 🎯 用途：数据验证、配置管理、API 请求/响应模型、序列化/反序列化、类型安全

---

## 🌟 一、Pydantic 是什么？

Pydantic 是一个基于 Python 类型提示（`typing`）的 **数据验证库**，支持：

- **自动验证数据结构**
- **类型检查（类型安全）**
- **错误清晰提示（带上下文）**
- **支持 JSON/Dict/类对象互转**
- **与 FastAPI 深度集成**（API 开发首选）
- **支持嵌套模型、`Optional`、`List`、`Dict` 等复杂结构**

> 🚀 特性：比 `jsonschema` 更直观，比 `attrs`/`dataclasses` 更强的验证能力。

---

## 🚀 二、安装与入门

```bash
pip install pydantic
```

### 简单示例（Hello World）

```python
from pydantic import BaseModel, ValidationError

class User(BaseModel):
    name: str
    age: int
    email: str

# 正常创建
user = User(name="Alice", age=25, email="alice@example.com")
print(user)
# 输出: name='Alice' age=25 email='alice@example.com'

# 验证失败
try:
    bad_user = User(name="Bob", age="not_an_int", email="invalid-email")
except ValidationError as e:
    print(e)
```

输出错误：
```
1 validation error for User
age
  Input should be a valid integer, unable to parse string as an integer [type=int_parsing, input_value='not_an_int', input_type=str]
```

✅ **自动类型转换失败，返回详细错误信息。**

---

## 🧱 三、核心组件：`BaseModel`

所有模型都继承自 `BaseModel`。

### 3.1 基本字段类型

| Python 类型 | Pydantic 类型 | 说明 |
|------------|--------------|------|
| `str`      | `str`        | 字符串 |
| `int`      | `int`        | 整数 |
| `float`    | `float`      | 浮点数 |
| `bool`     | `bool`       | 布尔值 |
| `list`     | `list[T]`    | 列表（T 为类型） |
| `dict`     | `dict[str, T]` | 字典（键必须为 str） |
| `None`     | `None`       | 显式为 `None` |

```python
from typing import List, Dict, Optional

class Product(BaseModel):
    name: str
    price: float
    tags: List[str]
    metadata: Dict[str, str]  # 字符串键值对
    description: Optional[str] = None  # 可选字段，可设默认值
```

---

## 🔍 四、验证规则与字段配置

### 4.1 使用 `Field()` 配置字段

```python
from pydantic import Field

class User(BaseModel):
    name: str = Field(..., min_length=2, max_length=50, description="用户名")
    age: int = Field(..., ge=0, le=150, description="年龄需在0-150之间")
    email: str = Field(..., pattern=r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$")
```

#### 常用参数：

| 参数 | 说明 |
|------|------|
| `default` | 默认值 |
| `default_factory` | 工厂函数（如 `default_factory=list`） |
| `alias` | 字段别名（用于 JSON key 不匹配类名） |
| `description` | 文档描述 |
| `min_length` / `max_length` | 字符串长度限制 |
| `min_items` / `max_items` | 列表长度 |
| `gt` / `ge` / `lt` / `le` | 数值比较（>、>=、<、<=） |
| `pattern` | 正则表达式匹配 |
| `title`, `example` | 用于 API 文档（如 FastAPI） |

### 示例：别名 + 默认值

```python
class ConfigModel(BaseModel):
    user_name: str = Field(alias="username", default="guest")
    active: bool = Field(default=True)

# JSON -> Model
data = {"username": "john", "active": False}
config = ConfigModel.model_validate(data)
print(config.user_name)  # john
```

---

## 📦 五、嵌套模型（Nested Models）

Pydantic 支持任意嵌套结构。

```python
class Address(BaseModel):
    street: str
    city: str
    zip_code: str

class User(BaseModel):
    name: str
    address: Address  # 嵌套对象
    friends: List[Optional["User"]] = []  # 递归引用（自引用）

# 使用
user = User(
    name="Alice",
    address=Address(street="123 Main St", city="NYC", zip_code="10001"),
    friends=[User(name="Bob", address=Address(street="456 Oak Ave", city="LA", zip_code="90210"))]
)
```

> 🔥 注意：自引用需要使用字符串引号（如 `"User"`）避免循环导入。

---

## 📥 六、从 JSON / Dict 转换模型

### 6.1 `model_validate()`（v2 推荐）

```python
data = {
    "name": "Alice",
    "age": 25,
    "email": "alice@example.com"
}

try:
    user = User.model_validate(data)
    print(user)
except ValidationError as e:
    print(e)
```

> 🆕 Pydantic v2 强制使用 `model_validate()`，不再推荐 `parse_obj`（已弃用）。

### 6.2 `model_validate_json()`（从 JSON 字符串）

```python
json_data = '{"name": "Bob", "age": 30, "email": "bob@example.com"}'
user = User.model_validate_json(json_data)
```

### 6.3 `model_dump()`（转为字典/JSON）

```python
user_dict = user.model_dump()  # 返回 dict
print(user_dict)
# {'name': 'Bob', 'age': 30, 'email': 'bob@example.com'}

user_json = user.model_dump_json(indent=2)  # 返回 JSON 字符串
print(user_json)
```

---

## 🛠 七、高级配置与自定义行为

### 7.1 模型配置（`model_config`）

```python
from pydantic import BaseModel, field_validator

class User(BaseModel):
    name: str
    age: int

    model_config = {
        "title": "User Model",
        "description": "User information with validation",
        "extra": "ignore",  # 忽略额外字段（default: "allow", "forbid"）
        "validate_default": True,  # 在验证默认值时也执行验证
        "frozen": True,  # 使模型不可修改（只读）
        "pop_by_alias": True,  # 反序列化时使用 alias 提取字段
    }
```

| `extra` 可选值 | 含义 |
|---------------|------|
| `allow`       | 允许额外字段 |
| `ignore`      | 忽略额外字段 |
| `forbid`      | 拒绝额外字段（报错） |

---

### 7.2 `field_validator`（自定义验证）

```python
from pydantic import field_validator

class User(BaseModel):
    name: str
    age: int

    @field_validator("name")
    @classmethod
    def validate_name(cls, v: str) -> str:
        if len(v) < 2:
            raise ValueError("Name must be at least 2 characters long")
        return v.upper()

    @field_validator("age")
    @classmethod
    def validate_age(cls, v: int) -> int:
        if v < 0:
            raise ValueError("Age cannot be negative")
        if v > 120:
            raise ValueError("Age seems unrealistic")
        return v
```

### 7.3 `model_validator`（跨字段验证）

```python
from pydantic import model_validator

class User(BaseModel):
    name: str
    age: int
    is_adult: bool = False

    @model_validator(mode='after')
    def check_age_adult(self):
        if self.age >= 18 and not self.is_adult:
            raise ValueError("User must be marked as adult if age >= 18")
        return self
```

> ✅ `mode='after'`：在所有字段验证后执行。

---

## 🔄 八、类型提示与泛型支持（Pydantic v2 特性）

例如：`List[User]`, `Dict[str, float]`

```python
class Order(BaseModel):
    items: List[Product]
    total: float

class Store(BaseModel):
    name: str
    inventory: Dict[str, Product]
```

### 泛型模型（高级）

```python
from typing import Generic, TypeVar

T = TypeVar('T')

class Response(Generic[T]):
    data: T
    success: bool

# 使用
resp = Response[int](data=42, success=True)
```

> ⚠️ 泛型支持需要 `pydantic` + `typing-extensions`

---

## 🔄 九、配置管理：`BaseSettings`（环境变量读取）

用于从 `.env` 文件或环境变量加载配置。

```python
from pydantic import BaseSettings, Field

class Settings(BaseSettings):
    database_url: str = Field(..., env="DB_URL")
    debug: bool = False
    max_connections: int = 100

    class Config:
        env_file = ".env"  # 加载 .env 文件
        extra = "ignore"

# 使用
settings = Settings()
print(settings.database_url)
```

`.env` 文件内容：
```env
DB_URL=postgresql://user:pass@localhost:5432/mydb
DEBUG=true
```

> 🚀 推荐：`pydantic-settings`（单独模块，v2 强烈推荐）

```bash
pip install pydantic-settings
```

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    debug: bool = False

    class Config:
        env_file = ".env"
```

---

## 🧩 十、与 FastAPI 深度集成

```python
from fastapi import FastAPI, Depends
from pydantic import BaseModel

app = FastAPI()

class CreateUser(BaseModel):
    name: str
    email: str

@app.post("/users")
def create_user(user: CreateUser):
    return {"message": f"User {user.name} created"}
```

- FastAPI 自动使用 `Pydantic` 验证请求体。
- 返回值自动序列化为 JSON。
- 自动生成 OpenAPI 文档（Swagger UI）。

---

## 📌 十一、最佳实践（Tips）

| 实践 | 说明 |
|------|------|
| ✅ 永远使用 `model_validate()` 和 `model_dump()` | v2 风格 |
| ✅ 给所有字段加 `Field(...)` 声明 | 提升可读性、文档 |
| ✅ 使用 `model_config` 配置额外行为 | 如 `extra='ignore'` |
| ✅ 使用 `BaseSettings` 管理配置 | 安全、可复用 |
| ✅ 异常统一处理 | 在 FastAPI 中用 `HTTPException` 包装 |
| ✅ 避免在模型中放逻辑 | 只做数据验证与结构定义 |
| ✅ 使用 `strict=True`（生产环境） | 严格类型检查 |
| ✅ 启用 `draft-7` 模式（如需兼容） | v2 默认启用更严格的 JSON Schema |

---

## 📚 十二、参考文档与资源

- 📚 官方文档：[https://docs.pydantic.dev](https://docs.pydantic.dev)
- 📦 GitHub：[https://github.com/pydantic/pydantic](https://github.com/pydantic/pydantic)
- 📖 FastAPI + Pydantic 教程：[https://fastapi.tiangolo.com](https://fastapi.tiangolo.com)
- 🎥 视频教程：B站/YouTube 搜索 “Pydantic v2 教程”

---

## 🎁 附录：常见错误与解决方案

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| `ValidationError: 1 validation error for Model` | 字段类型不匹配 | 检查类型（如 `str` vs `int`） |
| `field required` | 未提供必填字段 | 加 `default` 或 `Field(...)` |
| `extra fields not permitted` | `extra='forbid'` 但传了额外字段 | 改 `extra='ignore'` 或清理输入 |
| `Cannot resolve forward reference` | 自引用模型没加引号 | 写 `List['User']` 而不是 `List[User]` |

---

## ✅ 总结：Pydantic v2 的核心优势

| 特性 | 说明 |
|------|------|
| ✅ **类型安全** | 静态检查 + 运行时验证 |
| ✅ **自动序列化/反序列化** | `model_dump/json` / `model_validate/json` |
| ✅ **清晰错误信息** | 提供字段、路径、类型、期望值 |
| ✅ **与 FastAPI 天然协同** | 推荐配合使用 |
| ✅ **支持复杂嵌套结构** | 如 `List[Dict[str, User]]` |
| ✅ **支持环境变量配置** | 分级管理配置 |
| ✅ **自定义验证强大** | 支持 `field_validator`、`model_validator` |

---

📌 **学习建议：**

1. 从简单模型开始
2. 练习 `model_validate` + `model_dump`
3. 尝试嵌套模型与 `Field`
4. 迁移旧项目使用 `pydantic-settings`
5. 项目中用 FastAPI 测试真实流程

---

如果你想要一个 **可运行的完整示例项目（含 `.env`、FastAPI 路由、Pydantic 模型）**，我可以为你生成一个 GitHub 风格项目结构模板。

是否需要？我可以继续为你打包一份 MVC 架构模板（FastAPI + Pydantic + SQLAlchemy + Docker）。

--- 

✅ **你的 Pydantic 全面掌握之旅，从此刻开始！** 🚀