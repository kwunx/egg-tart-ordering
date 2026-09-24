# 微信小程序点餐系统 - 系统架构

## 1. 总体架构

V1.0 采用轻量级单体架构：

```text
微信小程序
     │ HTTPS
     ↓
   FastAPI
     │
     ├── Auth
     ├── Product
     ├── Cart/Order
     ├── Inventory
     ├── Payment
     └── Admin
     │
     ↓
 PostgreSQL
```

管理端：

```text
浏览器
  ↓
Jinja2 + HTMX + Tailwind CSS
  ↓
FastAPI
  ↓
PostgreSQL
```

不采用前后端分离管理端，V1.0 以降低复杂度、突出工程实践为目标。

---

## 2. 技术栈

### 后端

- Python 3.12
- FastAPI
- Uvicorn
- SQLAlchemy
- Alembic
- Pydantic
- psycopg
- python-dotenv

### 管理端

- Jinja2
- HTMX
- Tailwind CSS

### 顾客端

- 原生微信小程序
- JavaScript
- WXML
- WXSS

### 数据库

- PostgreSQL

### 部署

- Docker
- Nginx
- HTTPS

---

## 3. 后端分层

```text
app/
├── api/
├── core/
├── models/
├── schemas/
├── services/
├── repositories/
└── main.py
```

职责：

- `api`：HTTP 路由
- `schemas`：请求/响应数据结构
- `services`：业务逻辑
- `repositories`：数据库访问
- `models`：SQLAlchemy 模型
- `core`：配置、认证、数据库等基础能力
- `main.py`：应用入口

---

## 4. 认证架构

```text
wx.login()
↓
code
↓
FastAPI
↓
微信服务
↓
openid
↓
users
↓
签发应用 Token
```

后续 API：

```http
Authorization: Bearer <token>
```

后端从 Token 获得当前用户 ID。

Token 失效时，小程序重新登录。

V1.0 可使用 JWT 或安全随机 Token 实现，具体方案在 Phase 4 决定。

---

## 5. 订单/库存架构

订单创建必须是事务。

```text
BEGIN
 ↓
校验商品
 ↓
校验套餐
 ↓
读取数据库价格
 ↓
原子扣减库存
 ↓
检查 UPDATE 影响行数
 ↓
创建订单
 ↓
创建订单明细
 ↓
创建支付记录
 ↓
COMMIT
```

库存原子更新：

```sql
UPDATE products
SET stock = stock - 1
WHERE id = :id
AND stock >= 1;
```

影响 0 行表示库存不足。

待支付订单取消/超时时，在事务中恢复库存。

---

## 6. 钱与金额

数据库使用：

```sql
NUMERIC(10,2)
```

Python 使用：

```python
Decimal
```

禁止使用 Python `float` 作为核心金额计算类型。

---

## 7. 支付架构

V1.0：

```text
订单 → Mock Payment → 支付成功/失败
```

未来：

```text
订单
 ↓
微信支付
 ↓
支付回调
 ↓
幂等处理
 ↓
更新 payment
 ↓
更新 order
```

真实支付不得依赖客户端“支付成功”参数，必须以可信的支付结果为准。

---

## 8. 日志与错误处理

至少记录：

- 请求日志
- 异常日志
- 订单创建日志
- 订单状态变化日志
- 支付日志
- 库存异常日志

关键业务应包含 `order_no` 等关联标识，方便排查问题。

---

## 9. 环境

开发环境：

- macOS
- Miniconda
- Python 3.12
- 本地 PostgreSQL/Docker

生产环境：

- Linux
- Docker
- PostgreSQL
- Nginx
- HTTPS

敏感配置通过 `.env` 注入。

---

## 10. 工程原则

1. 后端是业务事实来源。
2. 前端只负责展示和提交意图。
3. 数据库负责数据一致性。
4. 关键库存操作必须原子化。
5. 多步业务必须使用事务。
6. 历史订单必须保存快照。
7. 状态变化必须有明确状态机。
8. 不为了“以后可能用到”过度设计。
