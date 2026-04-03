---
sidebar_position: 4
---

# FastAPI

> ✅ 适用人群：Python 开发者、后端工程师、全栈工程师
> 🔧 技术栈：Python 3.9+，FastAPI 0.100+，Pydantic v2，Docker，PostgreSQL，Redis，JWT，Uvicorn
> 📦 学习目标：掌握 FastAPI 全栈开发能力，构建高性能可维护的 Web API 服务

---

## 🧩 课程结构概览

| 模块 | 内容 |
|------|------|
| 1. FastAPI 快速入门 | 项目搭建、路由、请求/响应 |
| 2. Pydantic 数据验证 | 模型定义、嵌套结构、校验规则 |
| 3. HTTP 请求处理 | GET/POST/PUT/DELETE、查询参数、路径参数、请求体、文件上传 |
| 4. 异步编程 | 与 FastAPI 的完美集成（async/await） |
| 5. 数据库操作 | SQLAlchemy 与 ORM（异步支持） |
| 6. 依赖注入系统 | 依赖项复用、权限控制、数据库连接池 |
| 7. 路由分组与中间件 | 模块化设计、中间件过滤、日志记录 |
| 8. API 文档与测试 | Swagger UI / ReDoc、单元测试、集成测试 |
| 9. 权限认证与授权 | JWT + OAuth2、用户登录、身份验证 |
| 10. 缓存与性能优化 | Redis 缓存、响应压缩、连接池调优 |
| 11. 日志与监控 | Structured Logging、Prometheus、Health Checks |
| 12. 部署与 CI/CD | Docker 容器化、Nginx 反向代理、GitHub Actions |
| 13. 项目实战 | 构建一个完整的用户管理系统（带前端示例） |
| 14. 最佳实践 | 代码组织、错误处理、版本控制、API 设计规范 |

---

## ✅ 模块详解

---

### 🔹 模块 1：FastAPI 快速入门

#### 1.1 安装与启动

```bash
pip install fastapi uvicorn[standard]
```

#### 1.2 第一个 API 接口

```python
# main.py
from fastapi import FastAPI

app = FastAPI(title="FastAPI Demo", description="A simple hello world API")

@app.get("/")
def read_root():
    return {"message": "Hello, FastAPI!"}

@app.get("/items/{item_id}")
def read_item(item_id: int, query: str = None):
    return {"item_id": item_id, "query": query}
```

启动服务：

```bash
uvicorn main:app --reload
```

访问：
- `http://localhost:8000` → 返回 JSON
- `http://localhost:8000/docs` → 自动生成 Swagger UI
- `http://localhost:8000/redoc` → ReDoc 文档

---

### 🔹 模块 2：Pydantic 数据验证（核心）

FastAPI 依赖 Pydantic 实现自动类型检查与文档生成。

#### 2.1 定义数据模型

```python
# models.py
from pydantic import BaseModel
from typing import Optional
from datetime import datetime

class UserBase(BaseModel):
    username: str
    email: str

class UserCreate(UserBase):
    password: str  # 这里不会暴露

class UserResponse(UserBase):
    id: int
    created_at: datetime

    class Config:
        from_attributes = True  # Pydantic v2 支持
```

> ✅ **关键点**：
> - `from_attributes = True`：允许从 ORM 对象（如 SQLAlchemy 模型）构建实例。
> - 所有字段自动进行类型校验、默认值、格式检查。

#### 2.2 在路由中使用模型

```python
@app.post("/users/", response_model=UserResponse)
def create_user(user: UserCreate):
    # 自动校验：username 必须是 str，email 必须合法
    return {"id": 1, "username": user.username, "email": user.email, "created_at": datetime.now()}
```

- 请求不合法：FastAPI 返回 422 Unprocessable Entity，附带详细错误信息。
- 通过 Pydantic 自动实现 “类型 + 校验 + 文档” 三位一体。

---

### 🔹 模块 3：HTTP 请求处理

#### 3.1 路径参数 & 查询参数

```python
@app.get("/users/{user_id}")
def get_user(user_id: int, active: bool = True):
    return {"user_id": user_id, "active": active}
```

访问：`/users/5?active=false`

#### 3.2 请求体（JSON）

```python
@app.post("/login")
def login(user: UserCreate):
    return {"token": "fake-jwt-token"}
```

提交 JSON：

```json
{
  "username": "alice",
  "email": "alice@example.com",
  "password": "123456"
}
```

#### 3.3 文件上传

```python
from fastapi import File, UploadFile

@app.post("/upload/")
async def upload_file(file: UploadFile = File(...)):
    content = await file.read()
    return {"filename": file.filename, "size": len(content)}
```

---

### 🔹 模块 4：异步编程（async/await）

FastAPI 基于 Uvicorn + ASGI，天然支持异步。

#### 4.1 异步函数写法

```python
import asyncio

@app.get("/async")
async def async_endpoint():
    await asyncio.sleep(1)
    return {"message": "This is async!"}
```

#### 4.2 异步数据库操作（示例：SQLAlchemy async）

```python
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

@app.get("/db-test")
async def read_db(db: AsyncSession):
    result = await db.execute(select(User))
    return {"count": len(result.all())}
```

> ⚠️ 注意：需要使用 `async_sessionmaker` 的异步会话。

---

### 🔹 模块 5：数据库操作（SQLAlchemy + Async）

#### 5.1 安装依赖

```bash
pip install sqlalchemy asyncpg pydantic
```

#### 5.2 ORM 模型定义

```python
# models.py (SQLAlchemy)
from sqlalchemy import Column, Integer, String
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, index=True)
    email = Column(String, unique=True, index=True)
```

#### 5.3 数据库连接池（推荐用 `async_engine`）

```python
# database.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/myapp"

engine = create_async_engine(DATABASE_URL, echo=True)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_db():
    async with AsyncSessionLocal() as session:
        yield session
```

> ✅ 建议：使用依赖注入（`Depends(get_db)`）获取数据库会话。

---

### 🔹 模块 6：依赖注入系统（DIP）

依赖注入是 FastAPI 的核心设计模式。

#### 6.1 创建依赖项

```python
# dependencies.py
from fastapi import Depends, HTTPException

async def check_user_exists(user_id: int, db: AsyncSession):
    result = await db.execute(select(User).where(User.id == user_id))
    user = result.scalars().first()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

# 使用方式
@app.get("/users/{user_id}")
async def get_user(user: User = Depends(check_user_exists)):
    return {"username": user.username}
```

> ✅ 优点：
> - 代码复用
> - 可测试性高
> - 权限校验、日志、认证等都能封装为依赖

---

### 🔹 模块 7：路由分组与中间件

#### 7.1 路由分组（API 版本化）

```python
# api/v1/users.py
from fastapi import APIRouter

router = APIRouter(prefix="/users", tags=["users"])

@router.get("/")
def get_users():
    return ["user1", "user2"]

@router.post("/")
def create_user():
    return {"msg": "created"}
```

注册到主应用：

```python
# main.py
from fastapi import FastAPI
from api.v1.users import router as users_router

app = FastAPI(title="My API")
app.include_router(users_router)
```

路径：`/users/` → /v1/users/

#### 7.2 自定义中间件（日志拦截）

```python
from fastapi import Request, Response

@app.middleware("http")
async def log_requests(request: Request, call_next):
    print(f"Request: {request.method} {request.url}")
    response = await call_next(request)
    print(f"Response: {response.status_code}")
    return response
```

---

### 🔹 模块 8：API 文档与测试

#### 8.1 自动生成文档

- `/docs`：Swagger UI（推荐）
- `/redoc`：ReDoc（更美观）

支持动态请求示例、参数说明、响应结构。

#### 8.2 单元测试（使用 `pytest`）

```python
# test_main.py
import pytest
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_read_root():
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello, FastAPI!"}
```

运行测试：

```bash
pytest test_main.py -v
```

> ✅ 推荐使用 `pytest` + `TestClient` 模拟完整请求流程。

---

### 🔹 模块 9：认证与授权（JWT）

#### 9.1 使用 OAuth2PasswordBearer

```python
# auth.py
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import jwt, JWTError
import datetime

SECRET_KEY = "secret"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="login")

def create_token(data: dict):
    to_encode = data.copy()
    expire = datetime.datetime.utcnow() + datetime.timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: int = payload.get("sub")
        if user_id is None:
            raise HTTPException(status_code=401, detail="Invalid token")
        return user_id
    except JWTError:
        raise HTTPException(status_code=401, detail="Authentication failed")
```

#### 9.2 使用示例

```python
@app.post("/login")
def login(form_data: OAuth2PasswordRequestForm = Depends()):
    # 验证用户（虚构逻辑）
    user = {"id": 1, "username": "alice"}
    token = create_token({"sub": str(user["id"])})
    return {"access_token": token, "token_type": "bearer"}

@app.get("/me")
def read_users_me(current_user: int = Depends(get_current_user)):
    return {"user_id": current_user, "username": "alice"}
```

---

### 🔹 模块 10：缓存与性能优化

#### 10.1 使用 Redis 作为缓存

```python
import aioredis

redis = aioredis.from_url("redis://localhost", encoding="utf8", decode_responses=True)

@app.get("/cached-data")
async def get_cached_data():
    cached = await redis.get("data_key")
    if cached:
        return {"cached": True, "data": cached}
    # 生成数据（可能耗时）
    data = "very slow computation result"
    await redis.setex("data_key", 300, data)  # 5分钟过期
    return {"cached": False, "data": data}
```

#### 10.2 响应压缩（Gzip）

```python
# 在启动时启用
uvicorn.run(app, compress=True)  # 或使用 Uvicorn 配置
```

---

### 🔹 模块 11：日志与监控

#### 11.1 结构化日志（使用 `structlog`）

```python
import structlog

logger = structlog.get_logger()

@app.get("/slow")
async def slow():
    logger.info("Handling slow request", user_id=123)
    await asyncio.sleep(2)
    return {"msg": "done"}
```

#### 11.2 健康检查接口

```python
@app.get("/health")
def health_check():
    return {"status": "healthy", "timestamp": "now"}
```

#### 11.3 Prometheus 监控（可选）

使用 `fastapi-prometheus`：

```python
from fastapi_prometheus import PrometheusMiddleware, metrics
app.add_middleware(PrometheusMiddleware)
app.add_route("/metrics", metrics)
```

---

### 🔹 模块 12：部署与 CI/CD

#### 12.1 Docker 容器化

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
docker build -t my-fastapi-app .
docker run -p 8000:8000 my-fastapi-app
```

#### 12.2 Nginx 反向代理（生产推荐）

```nginx
# nginx.conf
server {
    listen 80;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

#### 12.3 GitHub Actions 自动部署

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build and push Docker image
        run: |
          docker build -t ghcr.io/yourname/myapp:latest .
          docker push ghcr.io/yourname/myapp:latest
```

---

### 🔹 模块 13：实战项目：用户管理系统

> ✅ 功能列表：
> - 用户注册 / 登录（JWT）
> - 获取用户列表（分页）
> - 创建 / 更新 / 删除用户
> - 文件上传头像（可选）
> - API 文档 + 单元测试 + Docker + Redis 缓存
> - 前端可选（React + Axios）

项目结构建议：

```
user_system/
├── main.py
├── models.py
├── database.py
├── schemas.py
├── dependencies.py
├── api/
│   ├── v1/
│   │   ├── users.py
│   │   └── auth.py
├── tests/
│   ├── test_users.py
├── requirements.txt
├── Dockerfile
├── .env
└── README.md
```

---

### 🔹 模块 14：最佳实践（SRE）

| 实践 | 说明 |
|------|------|
| ✅ 使用 `pydantic v2` | 类型安全、性能好 |
| ✅ 不暴露 `password` 字段 | 用 `model_config = {"exclude": ["password"]}` |
| ✅ 错误统一返回格式 | 定义 `APIResponse` 模型 |
| ✅ 使用环境变量 | `pydantic-settings` |
| ✅ API 版本控制 | `/v1/`, `/v2/` |
| ✅ 日志结构化 | 不用 `print()`，用 `structlog` |
| ✅ 响应码标准化 | 200/201/400/401/403/404/500 |
| ✅ 用 `async/await` 优化 I/O | 避免阻塞 |
| ✅ 不在 FastAPI 上直接写业务逻辑 | 拆分 `services` 层 |
| ✅ 使用 **FastAPI 项目模板** | 推荐：[fastapi-project-template](https://github.com/tiangolo/full-stack-fastapi-postgresql) |

---

## 🎁 附加资源

| 类型 | 推荐链接 |
|------|---------|
| 官方文档 | [https://fastapi.tiangolo.com](https://fastapi.tiangolo.com) |
| Pydantic v2 | [https://docs.pydantic.dev](https://docs.pydantic.dev) |
| GitHub 模板 | [full-stack-fastapi-postgresql](https://github.com/tiangolo/full-stack-fastapi-postgresql) |
| 实战项目 | [FastAPI + React + PostgreSQL 示例](https://github.com/microsoft/azure-devops-demo-apis) |
| 学习社区 | FastAPI Slack / Discord / Reddit r/fastapi |

---

## ✅ 总结：FastAPI 的优势

| 优势 | 说明 |
|------|------|
| 💥 极快性能 | 比 Flask 快 2