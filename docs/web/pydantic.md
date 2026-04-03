---
sidebar_position: 1
---

# Pydantic

1. [Pydantic 简介](#1-pydantic-简介)
2. [基础：定义模型与数据验证](#2-基础定义模型与数据验证)
3. [字段详解与高级类型](#3-字段详解与高级类型)
4. [模型序列化与导出](#4-模型序列化与导出)
5. [模型配置](#5-模型配置)
6. [自定义验证器 (核心)](#6-自定义验证器-核心)
7. [嵌套模型与复杂结构](#7-嵌套模型与复杂结构)
8. [数据类 与 TypedDict](#8-数据类-dataclass-与-typeddict)
9. [动态模型创建](#9-动态模型创建)
10. [高级工具：TypeAdapter](#10-高级工具typeadapter)
11. [Pydantic V1 与 V2 核心区别速查](#11-pydantic-v1-与-v2-核心区别速查)
12. [最佳实践与性能建议](#12-最佳实践与性能建议)
---
## 1. Pydantic 简介
**Pydantic** 是 Python 中最流行的数据验证和设置管理库。它的核心思想是：**基于 Python 原生的类型注解 来进行数据验证和解析**。
### 为什么使用 Pydantic？
* **强大的类型强制转换**：输入 `"123"`，如果字段类型是 `int`，Pydantic 会自动转为 `123`。
* **严格的数据校验**：类型不符或缺少必填项时，抛出清晰的 `ValidationError`。
* **与 IDE 完美契合**：因为使用的是标准类型注解，自动补全和类型检查极其友好。
* **性能极高**：V2 版本的核心由 Rust 编写，性能比 V1 提升了 5-50 倍。
* **生态基石**：它是 FastAPI、Django Ninja 等现代 Web 框架的底层依赖。
安装：
```bash
pip install pydantic
# 如果需要使用 email 等特殊类型验证
pip install pydantic[email]
```
---
## 2. 基础：定义模型与数据验证
所有的核心都从 `BaseModel` 开始。
```python
from pydantic import BaseModel
class User(BaseModel):
    id: int
    name: str
    is_active: bool = True  # 设置默认值，该字段变为非必填
# 1. 正常传入
user = User(id=1, name="Alice")
print(user.name)       # 输出: Alice
print(user.is_active)  # 输出: True (使用了默认值)
# 2. 类型转换 (核心特性)
user2 = User(id="2", name="Bob", is_active="yes") 
print(type(user2.id))      # <class 'int'> (字符串被转成了整数)
print(user2.is_active)     # True (字符串 "yes" 被转成了布尔值)
# 3. 数据校验失败
from pydantic import ValidationError
try:
    User(id="not_a_number", name="Charlie")
except ValidationError as e:
    print(e)
    """
    输入无效，会清晰指出:
    id: Input should be a valid integer [type=int_type, ...]
    """
```
---
## 3. 字段详解与高级类型
### 3.1 使用 `Field` 增强字段
`Field` 允许你为字段添加额外的验证规则、描述和默认值。
```python
from pydantic import BaseModel, Field
class Product(BaseModel):
    # 必填，且添加描述信息
    name: str = Field(..., description="产品名称") 
    
    # 可选，带默认值，且限制范围 0 <= price <= 10000
    price: float = Field(default=0.0, ge=0, le=10000, description="产品价格")
    
    # 字符串长度限制
    description: str = Field(default=None, min_length=10, max_length=100)
    # 正则表达式验证
    sku: str = Field(..., pattern=r'^[A-Z]{3}-\d{4}$') # 例如: ABC-1234
```
### 3.2 常用高级类型
```python
from typing import Optional, List, Dict, Literal
from pydantic import BaseModel, HttpUrl, EmailStr, ValidationError
class AdvancedTypes(BaseModel):
    # Optional: 可以是 str 或者 None
    nickname: Optional[str] = None 
    
    # 列表、字典
    tags: List[str] = []
    metadata: Dict[str, int] = {}
    
    # Literal: 限制只能为这几个值之一 (类似枚举)
    status: Literal["pending", "active", "closed"] = "pending"
    
    # 特殊类型 (需安装 email 扩展)
    email: EmailStr
    website: HttpUrl
```
### 3.3 延迟注解
在 Python 3.10 之前，如果你在模型内部引用了自己（递归模型）或者引用了尚未定义的类型，需要使用 `Future` 注解或 `model_rebuild()`。Python 3.10+ 推荐使用 `from __future__ import annotations`。

---

## 4. 模型序列化与导出
Pydantic 模型可以轻松转换为字典或 JSON 字符串。**注意：V2 中废弃了 `.dict()` 和 `.json()`，改用 `.model_dump()` 和 `.model_dump_json()`。**
```python
from pydantic import BaseModel, Field
class User(BaseModel):
    id: int
    password: str = Field(..., exclude=True) # 序列化时排除密码
    name: str
user = User(id=1, password="secret", name="Alice")
# 转为字典
print(user.model_dump()) 
# 输出: {'id': 1, 'name': 'Alice'} (password 被排除了)
# 转为 JSON 字符串
json_str = user.model_dump_json()
print(json_str) # 输出: {"id":1,"name":"Alice"}
# 进阶参数控制导出
print(user.model_dump(exclude_unset=True)) # 只输出显式设置过的字段（忽略默认值）
print(user.model_dump(mode="json"))        # 将所有值转为 JSON 兼容类型（如 datetime 变成字符串）
```
---
## 5. 模型配置
在 V2 中，配置通过 `model_config` 属性和 `ConfigDict` 来设置，取代了 V1 的内部类 `Config`。
```python
from pydantic import BaseModel, ConfigDict, Field
class StrictModel(BaseModel):
    model_config = ConfigDict(
        str_strip_whitespace=True,  # 自动去除字符串两端的空格
        str_to_lower=True,          # 字符串自动转小写
        extra='forbid',             # 禁止传入未定义的字段 (默认是 ignore)
        strict=True,                # 开启严格模式：禁止类型转换 (输入 "1" 给 int 会报错)
        populate_by_name=True,      # 允许通过字段名或别名赋值
    )
    
    username: str
# 测试 str_strip_whitespace 和 str_to_lower
m = StrictModel(username="  ALICE  ")
print(m.username) # 输出: "alice"
# 测试 extra='forbid'
try:
    StrictModel(username="bob", age=20) # age 未定义
except Exception as e:
    print(e) # 报错: Extra inputs are not permitted
```
---
## 6. 自定义验证器 (核心)
当内置的类型和 `Field` 无法满足复杂的业务逻辑时（如：密码必须包含大小写和数字、结束时间必须大于开始时间），需要使用验证器。
### 6.1 字段验证器 `field_validator`
替代了 V1 的 `@validator`。
```python
from pydantic import BaseModel, field_validator, ValidationError
class User(BaseModel):
    password: str
    @field_validator('password')
    @classmethod
    def validate_password(cls, v: str) -> str:
        if len(v) < 8:
            raise ValueError('密码长度不能少于 8 位')
        if not any(char.isupper() for char in v):
            raise ValueError('密码必须包含大写字母')
        return v  # 必须返回处理后的值
# User(password="weak") # 会报错
```
### 6.2 模型验证器 `model_validator`
用于需要**跨字段比较**的场景。替代了 V1 的 `@root_validator`。
```python
from pydantic import BaseModel, model_validator
class Event(BaseModel):
    start_time: int
    end_time: int
    @model_validator(mode='after') # mode='after' 表示所有字段验证完后再执行
    def check_time_range(self) -> 'Event':
        if self.start_time >= self.end_time:
            raise ValueError('结束时间必须大于开始时间')
        return self
    # mode='before' 则是接收原始字典，在类型转换前执行
```

---

## 7. 嵌套模型与复杂结构
Pydantic 可以轻松处理深层嵌套的数据。
```python
from pydantic import BaseModel
from typing import List, Optional
class Address(BaseModel):
    city: str
    street: str
class Company(BaseModel):
    name: str
    address: Address
class Employee(BaseModel):
    name: str
    company: Optional[Company] = None  # 嵌套模型可选
    skills: List[str] = []
# 复杂数据解析
data = {
    "name": "Tom",
    "company": {
        "name": "Tech Corp",
        "address": {
            "city": "Beijing",
            "street": "Chaoyang Road"
        }
    },
    "skills": ["Python", "FastAPI"]
}
emp = Employee.model_validate(data)
print(emp.company.address.city) # 输出: Beijing
```

---

## 8. 数据类 (`dataclass`) 与 `TypedDict`
如果你不想继承 `BaseModel`，Pydantic 也支持标准库的 `dataclass`。
```python
from pydantic.dataclasses import dataclass
@dataclass
class Person:
    name: str
    age: int
p = Person(name="Alice", age="25") # 自动转换
print(type(p.age)) # <class 'int'>
```
*注：Pydantic 的 `dataclass` 与标准库 `dataclasses.dataclass` 的区别在于它增加了数据验证功能。*

---

## 9. 动态模型创建
有时你无法在写代码时确定模型的结构（比如从数据库读取表结构动态生成），可以使用 `create_model`。
```python
from pydantic import BaseModel, Field, create_model
# 动态创建一个包含 username 和 age 的模型
DynamicUser = create_model(
    'DynamicUser',
    username=(str, ...), # (类型, 默认值)，... 表示必填
    age=(int, Field(default=18, ge=0))
)
user = DynamicUser(username="dynamic", age="20")
print(user) # username='dynamic' age=20
```

---

## 10. 高级工具：`TypeAdapter`
在 V2 中，如果你**只想验证一个基础类型、列表或字典，而不想定义一个完整的 `BaseModel`**，`TypeAdapter` 是最佳选择。
```python
from pydantic import TypeAdapter, ValidationError
# 验证一个复杂的列表，要求列表内都是正整数
adapter = TypeAdapter(list[int])
# 验证字典
dict_adapter = TypeAdapter(dict[str, float])
print(adapter.validate_python(["1", "2", 3])) # 输出: [1, 2, 3]
try:
    adapter.validate_json('["a", "b"]')
except ValidationError as e:
    print(e)
```

---

## 11. Pydantic V1 与 V2 核心区别速查
如果你在维护老项目，或者在网上看到旧教程，请参考下表：
| 功能场景 | Pydantic V1 语法 | Pydantic V2 语法 |
| :--- | :--- | :--- |
| **导出为字典** | `.dict()` | `.model_dump()` |
| **导出为 JSON** | `.json()` | `.model_dump_json()` |
| **解析 JSON** | `.parse_raw()` | `.model_validate_json()` |
| **解析字典** | `.parse_obj()` | `.model_validate()` |
| **复制模型** | `.copy()` | `.model_copy()` |
| **更新字段** | `.copy(update={...})` | `.model_copy(update={...})` |
| **配置类** | `class Config: ...` | `model_config = ConfigDict(...)` |
| **字段验证器** | `@validator('field')` | `@field_validator('field')` |
| **根验证器** | `@root_validator()` | `@model_validator()` |
| **私有属性** | `_private = PrivateAttr()` | `_private = PrivateAttr()` (没变) |

---

## 12. 最佳实践与性能建议
1. **优先使用内置约束**：能用 `Field(ge=0, le=100)` 解决的，就不要写 `@field_validator`。Rust 层面的内置校验速度远超 Python 层的自定义校验。
2. **尽量减少 Python 级别的验证器**：每个 `@field_validator` 都会带来微小的性能损耗，复杂的模型尽量将逻辑后移或前置，保持 Pydantic 只做纯粹的“数据边界验证”。
3. **序列化时使用 `mode='json'`**：如果你的最终目的是返回 JSON (如在 FastAPI 中)，在 `model_dump` 时加上 `mode='json'` 可以避免将 `datetime` 等非标准 JSON 类型暴露给后续处理。
4. **合理使用 `extra='forbid'`**：在对外暴露的 API 接口层，强烈建议开启 `extra='forbid'`，防止客户端误传无用字段导致潜在逻辑漏洞；在内部解析大量冗余外部数据时，使用默认的 `extra='ignore'` 性能更好。
5. **使用 `TypeAdapter` 替代一次性模型**：如果只是校验一个列表或简单结构，不要为了校验而去定义一个 `class XXX(BaseModel)`，直接用 `TypeAdapter` 更轻量。

---

**总结**：Pydantic 是现代 Python 开发中处理“不可信数据”的利器。掌握了 `BaseModel`、`Field`、`model_dump` 以及两大验证器 (`field_validator` / `model_validator`)，你就已经能够应对 95% 以上的数据校验场景。祝编码愉快！
