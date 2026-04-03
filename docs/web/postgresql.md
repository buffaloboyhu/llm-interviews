---
sidebar_position: 2
---

# PostgreSQL

> ✅ 适合人群：数据库初学者、后端工程师、DevOps、数据分析师、DBA
> 📅 学习周期：4–6 周（每日 1–2 小时）
> 💡 语言：中文 + 英文术语对照，附官方文档引用

---

## 🌟 第零章：绪论 —— 为什么选择 PostgreSQL？

### 1.1 PostgreSQL 优势概览
| 特性 | 说明 |
|------|------|
| **开源免费** | MIT 协议，无隐性成本 |
| **功能强大** | 支持 JSON、全文搜索、GIS、时序数据、存储过程等 |
| **标准兼容** | SQL 标准支持度极高（ANSI/ISO 标准） |
| **可扩展性** | 自定义数据类型、函数、操作符、索引方法等 |
| **高可用与复制** | 支持流复制、逻辑复制、主从/主主架构 |
| **事务安全** | ACID 全支持，MVCC 架构 |
| **活跃社区** | 官方团队 + 企业支持（如 2ndQuadrant、Elsayed, Amazon RDS 等） |

> 🔗 **官方主页**：[https://www.postgresql.org](https://www.postgresql.org)
> 📚 **文档**：[https://www.postgresql.org/docs](https://www.postgresql.org/docs)

---

## 📚 第一章：环境搭建与基础操作

### 1.1 安装 PostgreSQL

#### 【推荐平台】
- **macOS**：`brew install postgresql`
- **Ubuntu/Debian**：
  ```bash
  sudo apt update
  sudo apt install postgresql postgresql-contrib
  ```
- **Windows**：[https://www.enterprisedb.com/download-postgresql](https://www.enterprisedb.com/download-postgresql)
- **Docker（快速测试）**：
  ```bash
  docker run -d --name postgres \
    -e POSTGRES_PASSWORD=yourpassword \
    -p 5432:5432 \
    postgres:16
  ```

#### 验证安装
```bash
psql --version
# 或登录
psql -U postgres -d postgres -h localhost
```

### 1.2 初始配置与用户管理

```sql
-- 创建新用户（角色）
CREATE USER myuser WITH PASSWORD 'mypass';

-- 创建数据库（自动授权给用户）
CREATE DATABASE myapp OWNER myuser;

-- 切换用户与数据库
\c myapp myuser
```

> 💡 `psql` 命令行交互模式技巧：
- `\l`：列出所有数据库
- `\du`：列出所有角色
- `\dt`：列出当前数据库的所有表
- `\dv`：列出视图
- `\q`：退出

---

## 📚 第二章：核心 SQL 语法

### 2.1 数据定义语言（DDL）

```sql
-- 创建表
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE,
    age INT CHECK (age >= 18),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    status TEXT DEFAULT 'active',
    metadata JSONB
);

-- 添加列
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- 修改列类型
ALTER TABLE users ALTER COLUMN phone TYPE TEXT;

-- 删除列
ALTER TABLE users DROP COLUMN phone;

-- 重命名表/列
ALTER TABLE users RENAME TO clients;
ALTER TABLE clients RENAME COLUMN name TO full_name;

-- 添加约束
ALTER TABLE users ADD CONSTRAINT chk_age CHECK (age BETWEEN 18 AND 120);
```

### 2.2 数据操作语言（DML）

```sql
-- 插入数据
INSERT INTO users (name, email, age) VALUES
('Alice', 'alice@example.com', 25),
('Bob', 'bob@example.com', 30);

-- 批量插入
INSERT INTO users (name, email, age) 
VALUES ('Charlie', 'charlie@example.com', 35)
ON CONFLICT (email) DO NOTHING; -- 避免重复冲突

-- 查询数据
SELECT * FROM users WHERE age > 21;

-- 更新数据
UPDATE users SET status = 'inactive' WHERE id = 1;

-- 删除数据
DELETE FROM users WHERE id = 1;
```

> ✅ **ON CONFLICT** 是 PostgreSQL 独特且强大的特性，用于处理唯一键冲突。

### 2.3 数据查询语言（DQL）

#### 基本查询
```sql
SELECT name, email FROM users
WHERE created_at > '2025-01-01'
ORDER BY age DESC
LIMIT 10 OFFSET 0;
```

#### 聚合函数 & 分组
```sql
SELECT 
    status,
    COUNT(*) AS total,
    AVG(age) AS avg_age
FROM users
GROUP BY status
HAVING COUNT(*) > 1;
```

#### JOIN 操作
```sql
-- 内连接
SELECT u.name, o.amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- 左连接（保留左表所有数据）
SELECT u.name, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- 交叉连接
SELECT u.name, p.product_name
FROM users u
CROSS JOIN products p;
```

#### 子查询
```sql
SELECT name FROM users
WHERE age > (SELECT AVG(age) FROM users);
```

---

## 📚 第三章：高级数据类型与扩展功能

### 3.1 特殊数据类型

| 类型 | 用途 | 示例 |
|------|------|------|
| `JSONB` | 二进制 JSON，支持索引 | `{"city": "Beijing", "score": 90}` |
| `ARRAY` | 数组类型 | `ARRAY[1,2,3]` |
| `HSTORE` | 键值对存储（已逐渐被 JSONB 替代） | `'key => value'` |
| `UUID` | 全球唯一标识符 | `uuid_generate_v4()` |
| `BOX`, `PATH`, `CIRCLE` | 几何图形类型 | GIS 基础 |
| `INET`, `MACADDR` | IP 地址 & MAC 地址 | 显式类型校验 |

🛠️ 示例：使用 JSONB
```sql
-- 插入 JSON 数据
INSERT INTO users (name, metadata) VALUES
('Alice', '{"preferences": {"theme": "dark", "lang": "zh"}}');

-- 查询嵌套字段
SELECT metadata->'preferences'->>'lang' AS language FROM users;

-- 支持索引（GIN 索引加速查找）
CREATE INDEX idx_users_metadata ON users USING GIN (metadata);
```

### 3.2 索引优化

| 索引类型 | 适用场景 |
|---------|--------|
| B-tree（默认） | 一般等值/范围查询 |
| Hash | 精确匹配（=） |
| GIN | JSONB、数组、全文搜索 |
| GiST | 空间数据、复杂类型 |
| BRIN | 大表按物理顺序排列（时间序列、日志） |

```sql
-- 创建 B-tree 索引
CREATE INDEX idx_users_email ON users(email);

-- 创建 GIN 索引（JSONB 查询加速）
CREATE INDEX idx_users_jsonb ON users USING GIN (metadata);

-- 复合索引
CREATE INDEX idx_users_age_status ON users(age, status);
```

> 🔍 **索引关键原则**：
> - 避免过度索引（影响写入性能）
> - 索引应优先匹配高频查询字段
> - 考虑使用 `INCLUDE`（覆盖索引，减少回表）

### 3.3 视图（Views）

```sql
CREATE VIEW active_users AS
SELECT id, name, email, created_at
FROM users
WHERE status = 'active';

-- 查询视图（效果如普通表）
SELECT * FROM active_users;
```

> ✅ 优势：封装复杂逻辑、简化权限控制

---

## 📚 第四章：事务与并发控制

### 4.1 ACID 特性与 MVCC

PostgreSQL 使用 **多版本并发控制（MVCC）** 实现高并发读写，**不会阻塞读操作**。

### 4.2 事务管理

```sql
BEGIN;

INSERT INTO users (name, email, age) VALUES ('Charlie', 'charlie@example.com', 30);

-- 检查是否成功
-- 如果需要回滚
ROLLBACK;

-- 如果确认无误
COMMIT;
```

### 4.3 隔离级别（默认为 `read committed`）

| 隔离级别 | 特点 |
|--------|------|
| `read uncommitted` | 不推荐，可能脏读 |
| `read committed` | 默认，每次查询读取最新提交 |
| `repeatable read` | 同事务内读取一致性，避免不可重复读 |
| `serializable` | 最高隔离度，保证无幻读，但性能代价高 |

> ⚠️ PostgreSQL 中 `serializable` 内部通过 `serialization failure` 自动重试。

---

## 📚 第五章：存储过程与函数（PL/pgSQL）

```sql
-- 创建函数：计算用户年龄等级
CREATE OR REPLACE FUNCTION get_age_category(age INT)
RETURNS TEXT AS $$
BEGIN
    IF age < 18 THEN
        RETURN 'minor';
    ELSIF age < 60 THEN
        RETURN 'adult';
    ELSE
        RETURN 'senior';
    END IF;
END;
$$ LANGUAGE plpgsql;

-- 调用函数
SELECT name, get_age_category(age) AS category FROM users;
```

### 5.1 控制结构
- `IF-THEN-ELSE`
- `LOOP`, `WHILE`, `FOR` 循环
- `EXCEPTION` 异常处理

### 5.2 事件驱动：触发器（Triggers）

```sql
-- 创建函数：每日记录登录次数
CREATE OR REPLACE FUNCTION log_login()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO login_logs (user_id, login_time)
    VALUES (NEW.id, NOW());
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 创建触发器
CREATE TRIGGER trigger_log_login
AFTER INSERT ON users
FOR EACH ROW EXECUTE FUNCTION log_login();
```

---

## 📚 第六章：性能优化与调优

### 6.1 执行计划分析（EXPLAIN）

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'alice@example.com';
```

> 输出关键信息：
- `Seq Scan`：全表扫描（慢，应避免）
- `Index Scan`：使用索引（好）
- `Bitmap Index Scan`：复合索引优化
- `Join Type`：决定 JOIN 效率（Nested Loop / Hash Join / Merge Join）

### 6.2 参数调优（`postgresql.conf`）

常见调优项（建议按硬件调整）：

```conf
# 内存相关
shared_buffers = 2GB                  # 建议占系统内存 25%
effective_cache_size = 8GB            # OS 缓存 + shared_buffers
work_mem = 8MB                        # 排序/哈希内存
maintenance_work_mem = 1GB

# 并发连接
max_connections = 100

# WAL 日志（写入日志）
synchronous_commit = on
wal_buffers = 16MB

# 自动清理
autovacuum = on
autovacuum_vacuum_scale_factor = 0.1
autovacuum_analyze_scale_factor = 0.05
```

### 6.3 分区表（Partitioning） —— 处理超大数据

```sql
-- 按年份分区
CREATE TABLE sales (
    id SERIAL,
    amount DECIMAL(10,2),
    sale_date DATE NOT NULL
) PARTITION BY RANGE (sale_date);

-- 创建分区
CREATE TABLE sales_2023 PARTITION OF sales
FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE sales_2024 PARTITION OF sales
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

> ✅ 优势：查询性能提升，自动清理老数据

---

## 📚 第七章：高可用与灾备

### 7.1 主从流复制（Streaming Replication）

```bash
# 1. 配置主库（master）
# postgresql.conf
wal_level = replica
max_wal_senders = 10
synchronous_commit = on
archive_mode = on
archive_command = 'cp %p /path/to/archive/%f'

# pg_hba.conf
host replication replicator 192.168.1.100/32 md5

# 2. 初始化从库（standby）
pg_basebackup -h 192.168.1.100 -D /var/lib/postgresql/data \
  -U replicator -P -v --xlog-method=stream

# 3. 创建恢复文件（recovery.conf）
echo "standby_mode = 'on'
primary_conninfo = 'host=192.168.1.100 port=5432 user=replicator password=...'"
> /var/lib/postgresql/data/recovery.conf
```

> 🔁 从库自动同步日志，支持自动故障转移（配合 Patroni/Consul）

### 7.2 逻辑复制（Logical Replication）

- 适合跨数据库分片、迁移数据
- 基于 WAL 行级逻辑解码

```sql
-- 创建发布端
CREATE PUBLICATION mypub FOR TABLE users;

-- 订阅端创建订阅
CREATE SUBSCRIPTION mysub
CONNECTION 'host=master hostaddr=192.168.1.100 port=5432 dbname=app user=replicator'
PUBLICATION mypub;
```

---

## 📚 第八章：安全与权限管理

### 8.1 角色与权限模型

- **角色（Role）** 是用户/组的抽象
- 权限可授予角色，作用于表/函数/数据库

```sql
-- 创建角色
CREATE ROLE analyst;
CREATE ROLE developer LOGIN PASSWORD 'devpass';

-- 授予权限
GRANT SELECT, INSERT ON TABLE users TO analyst;
GRANT ALL PRIVILEGES ON DATABASE myapp TO developer;

-- 设置继承
ALTER ROLE analyst INHERIT;
```

> ✅ 推荐使用角色分层：`app_user`, `dba`, `reporter` 等

### 8.2 行级安全（RLS）

```sql
-- 启用行级安全
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- 创建策略
CREATE POLICY user_policy ON users
FOR SELECT USING (current_user = 'manager' OR user_id = CURRENT_USER::regrole);

-- 仅管理员可查看所有，普通用户只能看自己
```

---

## 📚 第九章：集成与工具链

### 9.1 ORM 集成
| 框架 | 支持程度 |
|------|--------|
| Django (Python) | 原生支持，全功能 |
| SQLAlchemy (Python) | 完美支持 |
| Hibernate (Java) | 通过 JDBC 支持 |
| Prisma (TypeScript) | 支持 PostgreSQL，推荐使用 `pg` 驱动 |
| GORM (Go) | 优秀支持 `.Where("age > ?", 18)` |

### 9.2 开发/运维工具
| 工具 | 用途 |
|------|------|
| **pgAdmin** | Web 界面管理工具（推荐） |
| **DBeaver** | 开源通用数据库客户端 |
| **Adminer** | 轻量级 Web 管理 |
| **pgBadger** | 日志分析（慢查询、负载） |
| **Prometheus + Exporter** | 监控（CPU, IOPS, WAL, Connections） |
| **pgBackRest** | 备份恢复工具（企业级） |

---

## 📚 第十章：实战项目示例（推荐）

### 项目：电商订单系统

#### 数据结构设计
- `users`, `products`, `orders`, `order_items`, `inventory`
- 使用 JSONB 存储订单元数据
- 用分区表按月分拆订单数据
- 启用 RLS，限制用户只能访问自己的订单
- 使用触发器维护库存

#### 性能挑战
- 亿级订单数据查询 → 分区 + 索引
- 实时报表 → 物化视图（MATERIALIZED VIEW）
- 使用 `EXPLAIN ANALYZE` 定位慢查询

---

## 🏁 结语：PostgreSQL 进阶路径

| 阶段 | 学习目标 |
|------|--------|
| 初学者 | 熟练 SQL、基本建表、增删改查 |
| 中级 | 索引优化、函数、视图、触发器 |
| 高级 | 分区、复制、逻辑复制、性能调优 |
| 专家 | 自定义类型/函数、调优内核参数、故障排查、自动化部署 |

---

## 📎 附录资源推荐

- 📘 官方文档：[https://www.postgresql.org/docs](https://www.postgresql.org/docs)
- 📘 《PostgreSQL 16 从入门到精通》（中文书籍）
- 📘 《Designing Data-Intensive Applications》（理解数据库原理）
- 📊 SQL 优化工具：[https://pganalyze.com](https://pganalyze.com)（付费，但强大）
- 🐳 Docker