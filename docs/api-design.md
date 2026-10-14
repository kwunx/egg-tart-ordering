# 微信小程序点餐系统 - API 设计

## 1. API 前缀与响应

`backend/` 的 FastAPI 只提供业务 JSON API，所有业务接口使用 `/api/v1/` 前缀：

- 顾客接口保持 `/api/v1/auth/*`、`/api/v1/products/*`、`/api/v1/orders/*` 等现有路径。
- 管理接口统一为 `/api/v1/admin/*`；管理员登录为 `POST /api/v1/admin/auth/login`。
- `/admin/` 属于 `admin-web/` 的独立 SPA 页面，例如 `/admin/login`，不是后端 API 路径。

管理端采用 Vue 3、TypeScript、Vite、Vue Router、Pinia 和 Element Plus，通过 JSON API 获取数据与提交操作。后端不渲染管理页面、不返回 HTML 片段。

顾客和管理员接口统一 JSON：

- 查询接口返回数据及下列 `code/message/data` 包装，不返回页面 HTML。
- 写接口有请求体时使用 `Content-Type: application/json`，不提交 HTML 表单或 HTMX 片段请求；原有无业务请求体的支付、取消等操作保持无业务请求体。
- 认证失败和权限不足分别使用 HTTP 401/403，响应仍为 JSON；API 不返回登录页，也不发送跳往登录页的 302。

成功和失败格式如下。

成功：

```json
{
  "code": 0,
  "message": "success",
  "data": {}
}
```

失败：

```json
{
  "code": 40901,
  "message": "库存不足",
  "data": null
}
```

HTTP 状态：

- 200 成功
- 201 首次创建订单成功；同内容幂等重放返回 200
- 400 参数/业务错误
- 401 未认证
- 403 无权限
- 404 不存在
- 409 资源冲突
- 422 参数校验失败
- 500 服务端异常

金额使用两位小数字符串，例如 `"54.00"`；后端使用 Decimal 计算。响应时间为带时区的 ISO 8601 字符串。

HTTP 409 的业务 `code` 固定为以下整数：

| code | 含义 |
|---|---|
| 40901 | 库存不足，含管理减量后库存将小于 0 |
| 40902 | 同一幂等请求标识对应不同业务内容 |
| 40903 | 当前订单/支付状态不允许操作 |
| 40904 | 支付请求对应的订单已过期并关闭 |
| 40905 | 服务端 Mock 确认失败，本次失败状态已提交 |

错误响应沿用 `code/message/data`，`data=null`。需要保存的超时关闭或 Mock 失败状态必须先提交，再返回 HTTP 409；不能通过抛出异常把这些变化回滚。

### 1.1 路径迁移与前后端对接

| 原设计中的后端接口 | 分离后的 JSON API |
|---|---|
| `POST /admin/login` | `POST /api/v1/admin/auth/login` |
| `/admin/products` 及商品操作子路径 | `/api/v1/admin/products` 及对应子路径 |
| `/admin/categories`、`/admin/combos` | `/api/v1/admin/categories`、`/api/v1/admin/combos` |
| `/admin/orders` 及订单操作子路径 | `/api/v1/admin/orders` 及对应子路径 |
| `/admin/dashboard` | `/api/v1/admin/dashboard` |

旧 `/admin/*` API 地址不继续作为业务 API 别名；`/admin/*` 用于前端页面路由。此次只迁移已有管理接口，不增加分类或套餐写操作等新接口范围。原有库存调整 UUID、JSON 字段、金额快照、业务错误码、超时和事务规则保持不变。

生产建议由同一个 HTTPS 域名的 Nginx 提供 `/admin/` SPA 静态文件，并将 `/api/v1/` 反向代理到 FastAPI。SPA 深链接刷新回退仅作用于管理页面，不能把不存在或失败的 API 请求改写为 `index.html`。前后端分别构建和部署，不要求使用不同域名。

开发时 Vite 将 `/api/v1` 代理到 `http://127.0.0.1:8000`，保留原请求路径；管理端代码使用 `/api/v1` 相对地址。确需浏览器跨域直连时，后端 CORS 仅允许配置中准确列出的 Origin，允许 `GET/POST/PUT/OPTIONS` 及 `Authorization/Content-Type` 请求头；本方案不使用 Cookie 认证，不开启 credentials，也不使用通配 Origin 替代配置。

微信小程序通过配置的 HTTPS API 地址调用服务，其合法请求域名配置与浏览器 CORS 是两件事；开放 CORS 不能代替小程序合法域名配置。前端公开配置只包含 API 地址等非敏感值，Token 签名密钥及微信 AppSecret 保留在 backend。

---

## 2. 顾客认证

### POST `/api/v1/auth/login`

顾客端为 `customer-app/`：uni-app + Vue 3 + TypeScript，使用 CLI `vite-ts` 模板，V1 仅发布 `mp-weixin`。微信端调用 `uni.login({ provider: "weixin" })` 取得 code，再按以下 JSON 发送到既有登录接口；后端继续向微信换取 openid。

跨端框架不改变本版本身份体系。本次不增加支付宝、抖音或 H5 登录接口；顾客应用 Token 的 JWT/安全随机 Token 选择仍按 Phase 4 决定，不因管理员已选 JWT 而改写顾客登录响应。

请求：

```json
{
  "code": "wx-login-code"
}
```

后端：

```text
code
↓
微信服务
↓
openid
↓
users 查找/创建
↓
签发 Token
```

响应：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "token": "...",
    "user_id": 1
  }
}
```

---

## 3. 商品

### GET `/api/v1/categories`

获取启用分类。

### GET `/api/v1/products`

获取商品。

支持按分类、启用状态等筛选。

### GET `/api/v1/products/{id}`

商品详情。

### GET `/api/v1/products/{id}/combo-options`

获取套餐允许选择的普通商品。

---

## 4. 创建与查询订单

### POST `/api/v1/orders`

请求示例：两份同配方“6 个任选”加一个原味单品。

```json
{
  "client_request_id": "550e8400-e29b-41d4-a716-446655440000",
  "items": [
    {
      "product_id": 100,
      "quantity": 2,
      "selections": [
        {"product_id": 1, "quantity": 2},
        {"product_id": 2, "quantity": 2},
        {"product_id": 3, "quantity": 2}
      ]
    },
    {"product_id": 1, "quantity": 1, "selections": []}
  ],
  "remark": "请分开装"
}
```

- `client_request_id` 为客户端生成的 UUID；同一次提交遇到超时、断网或未知结果时复用原值。新的下单意图使用新值。
- `items` 非空；商品 ID、行数量和选择数量均为正整数；数量严格拒绝布尔值及小数。
- `quantity` 对单品表示个数，对套餐表示份数。`selections` 必须提供，单品为 `[]`，套餐元素仅包含 `product_id/quantity`，表示每份配方。
- 单个套餐配方中子商品 ID 不得重复，所选商品必须属于允许的普通商品，合计数量等于套餐规定数量。配方相同可合并份数，不同配方分行。
- `remark` 最长 500 字符，缺省或 NULL 标准化为空字符串。
- **禁止信任前端 price、subtotal、total_amount。** 首次创建读取服务端价格重新计价；客户端不能决定快照、状态或支付结果。

### 4.1 规范化与幂等结果

幂等范围是当前用户，数据库约束为 `UNIQUE(user_id, client_request_id)`。

请求规范化仅使用业务意图：

1. 校验请求结构，拒绝同一配方内重复子商品 ID，缺省/NULL 备注转成 `""`。
2. 每份 `selections` 按子商品 ID 升序排序。
3. 按 `(product_id, 规范化配方)` 合并等价订单行、累加行数量，再按商品 ID 和配方稳定排序。
4. 对仅含 `items/remark` 的对象稳定序列化：JSON 对象键排序、无多余空白、UTF-8、中文不转义。计算 SHA-256 小写十六进制 `request_hash`；不纳入请求标识、服务端价格、名称、库存或时间。

必须先处理已有幂等结果，再校验商品当前可售性、价格、套餐配置和库存：

| 情况 | 响应 |
|---|---|
| 首次创建完整提交 | HTTP 201，返回新订单 |
| 同用户、同标识、同指纹已有订单 | HTTP 200，返回原订单最新状态和原快照，不重新计价或扣库存 |
| 同用户、同标识、不同指纹 | HTTP 409 / 40902，不改动原订单 |
| 首次业务校验或事务失败 | 全部回滚，不留下占位订单；原标识可用于相同意图再次尝试 |

原订单已经支付、取消或到期时，同标识仍只能返回该原订单，不能新建。客户端修改业务内容后应生成新标识。

### 4.2 首次创建事务

采用 PostgreSQL READ COMMITTED。先插入订单行取得幂等唯一键，再校验当前业务数据及扣库存，避免同键并发请求先争库存而误报库存不足。

```text
认证、校验请求结构、规范化、生成指纹
↓
查询既有幂等结果：相同返回原单，不同返回 40902
↓
BEGIN
↓
插入 orders 行，占用 (user_id, client_request_id) 唯一键
↓
校验当前商品/套餐，读取服务端价格
↓
生成订单项及实际扣减快照，汇总每个普通商品的总消耗
↓
按普通商品 ID 升序条件扣减库存并检查受影响行数
↓
写入 order_items、订单最终总额和一条 pending payment
↓
COMMIT → HTTP 201
```

订单行在事务内可暂以 `total_amount=0` 建立唯一键占位。提交前必须补齐最终金额、明细及支付记录；校验失败或任一步失败全部回滚，不能提交不完整订单。

并发唯一键冲突时，争用失败请求先回滚自身事务，再读取已提交胜者并比较指纹，返回原单 200 或 40902。若竞争事务回滚且无已提交订单，则重新进入创建流程。不得在失败事务内继续查询；任何请求在占有订单唯一键前都不得扣库存。

首次插入订单占位行时，以数据库 `clock_timestamp()` 取得同一个时刻 t，设置 `created_at=t`、`expires_at=t + PAYMENT_TIMEOUT_SECONDS`，默认 900 秒。两者此后不变；每单保存自己的到期时间，后续配置变更不改变旧订单期限。

上例假设套餐单价 `"25.00"`、原味单价 `"4.00"`，总额 `"54.00"`。实际扣商品 1 共 5 个，商品 2、3 各 4 个。创建响应（首次 HTTP 201，重放 HTTP 200）：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "order_id": 101,
    "order_no": "ET202610100001",
    "client_request_id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "pending_payment",
    "total_amount": "54.00",
    "remark": "请分开装",
    "created_at": "2026-10-10T10:00:00+08:00",
    "expires_at": "2026-10-10T10:15:00+08:00",
    "server_time": "2026-10-10T10:00:00+08:00",
    "cancel_reason": null,
    "items": [
      {
        "product_id": 1,
        "product_name": "原味蛋挞",
        "item_type": "single",
        "unit_price": "4.00",
        "quantity": 1,
        "subtotal": "4.00",
        "selections": []
      },
      {
        "product_id": 100,
        "product_name": "6 个任选",
        "item_type": "combo",
        "unit_price": "25.00",
        "quantity": 2,
        "subtotal": "50.00",
        "selections": [
          {"product_id": 1, "product_name": "原味蛋挞", "quantity": 2},
          {"product_id": 2, "product_name": "芝士蛋挞", "quantity": 2},
          {"product_id": 3, "product_name": "芋泥蛋挞", "quantity": 2}
        ]
      }
    ],
    "payment": {
      "payment_method": "mock",
      "payment_status": "pending",
      "amount": "54.00",
      "transaction_id": null,
      "paid_at": null
    }
  }
}
```

返回的名称、价格、配方及小计来自订单快照。`server_time` 在创建、创建幂等重放和订单详情每次响应时重新取数据库当前时间，用于校准倒计时；它不是存储字段，也不进入指纹，重放时更新 server_time 但保持原 expires_at。`request_hash`、`stock_deductions` 为内部服务端数据，不由客户端提交。

### GET `/api/v1/orders/my`

只返回当前 Token 对应用户的订单。

### GET `/api/v1/orders/{order_id}`

只能访问当前用户自己的订单。详情 `data` 使用上述订单结构，包含当前订单/支付状态、到期时间、本次 server_time、取消原因和原始快照。

---

## 5. Mock 支付

### POST `/api/v1/orders/{order_id}/pay`

无业务请求体。Mock 默认成功，失败仅由服务端测试配置触发；客户端不能提交成功或失败结果。

支付、手动取消和超时关闭共用锁顺序：**订单行 → payment 行 → 涉及的普通商品行按 ID 升序**。支付成功不再次扣库存。验证订单归属并锁定后，使用数据库 `clock_timestamp()` 判断到期，不能使用事务开始时间或客户端倒计时。

| 锁定后状态 | 行为 |
|---|---|
| `paid/making/completed + success` | HTTP 200，返回原支付结果，不改 `paid_at/transaction_id`；不因当前已超过原期限而关闭已支付订单 |
| `cancelled + closed` | `cancel_reason=timeout` 返回 HTTP 409 / 40904，`manual` 返回 HTTP 409 / 40903；不支付、不再恢复库存 |
| `pending_payment + pending/failed` 且当前时间小于 `expires_at` | 允许服务端进行 Mock 确认 |
| `pending_payment + pending/failed` 且当前时间大于等于 `expires_at` | 同一事务恢复库存，写 `cancelled + closed`、`cancel_reason=timeout`；提交后 HTTP 409 / 40904 |
| 其他组合 | HTTP 409 / 40903，并记录状态异常 |

确认成功：同一事务写 payment `success`、生成 Mock `transaction_id` 和 `paid_at`，订单置为 `paid`。确认失败：仅将 payment 置为 `failed`，订单仍 `pending_payment`，保留库存且不延长到期时间；先提交，再返回 HTTP 409 / 40905。未到期时可在原订单、原 payment 行上重试，不创建第二条支付记录。

成功及成功重放示例（HTTP 200）：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "order_id": 101,
    "status": "paid",
    "payment": {
      "payment_method": "mock",
      "payment_status": "success",
      "amount": "54.00",
      "transaction_id": "mock-101-1",
      "paid_at": "2026-10-10T10:02:00+08:00"
    }
  }
}
```

重放时订单可以已为 `making/completed`，支付流水和首次付款时间保持不变。失败使用第 1 节错误结构，客户端可读取订单详情确认已提交状态。

---

## 6. 手动取消与超时关闭

### POST `/api/v1/orders/{order_id}/cancel`

无业务请求体。验证用户归属后，按统一锁顺序进入事务：

- 已为 `cancelled + closed`：HTTP 200 返回原结果，保留原取消原因，不再恢复库存。
- 已支付、制作中或完成：HTTP 409 / 40903。
- `pending_payment + pending/failed`：锁后使用 `clock_timestamp()` 判断，未到期按 `manual` 取消，到期按 `timeout` 关闭。
- 把所有订单项 `stock_deductions` 按商品 ID 聚合、升序恢复库存，同时写订单 `cancelled`、取消原因及支付 `closed`。
- 此次按 manual 或 timeout 关闭提交后均 HTTP 200，返回实际取消原因；再次取消已关闭订单仍为 HTTP 200。只有支付接口因到期关闭或查询到此前超时关闭时返回 40904。

取消成功或重复取消示例：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "order_id": 101,
    "status": "cancelled",
    "cancel_reason": "manual",
    "payment": {
      "payment_method": "mock",
      "payment_status": "closed",
      "amount": "54.00",
      "transaction_id": null,
      "paid_at": null
    }
  }
}
```

超时任务默认每 `ORDER_EXPIRY_SCAN_INTERVAL_SECONDS=30` 秒扫描到期的待支付订单，逐单调用同一关闭服务。锁后再次核实当前状态和时间，支付、取消或重复扫描只能产生一次最终变化和一次库存恢复。不提供允许顾客任意指定订单的超时管理入口。

任务延迟或重启不延长付款资格：支付入口主动检查到期，任务恢复后补扫所有已过期且仍待支付的订单。详见 [库存设计](inventory-design.md) 和 [订单状态机](order-state-machine.md)。

---

## 7. 管理端

### POST `/api/v1/admin/auth/login`

JSON 请求：

```json
{
  "username": "admin",
  "password": "your-password"
}
```

校验用户名、密码哈希及管理员启用状态，成功后返回 HTTP 200：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "token": "...",
    "token_type": "Bearer",
    "expires_in": 28800,
    "admin": {
      "id": 1,
      "username": "admin"
    }
  }
}
```

V1 管理员使用 JWT，至少包含 `sub`（管理员 ID 字符串）、`actor="admin"`、`aud="admin"` 和 `exp`。`ADMIN_TOKEN_TTL_SECONDS=28800` 为默认有效期，响应 `expires_in` 使用实际配置的秒数。签名由服务端完成，签名密钥不进入前端构建产物。

除管理员登录外，所有 `/api/v1/admin/*` 业务请求携带：

```http
Authorization: Bearer <admin-token>
```

后端验证签名、有效期、actor、aud，再依据 sub 查询管理员存在且 `is_active=true`。顾客 Token 即使对应同一个数字 ID 也不能访问管理接口；前端显示的身份信息不能替代后端校验。失效或缺失凭据返回 401，无管理权限返回 403，均使用第 1 节 JSON 错误包装。

管理 SPA 将 Token 存在当前浏览器会话的 `sessionStorage`，请求层添加 Authorization。收到 401 或管理员主动退出时清理本地 Token 及当前身份状态，再由 Vue Router 前往 `/admin/login`；403 显示无权限反馈，不把 API 响应当作页面。退出是客户端清理，不宣称服务端吊销已签发 JWT，不新增会话表或退出 API。JWT 在失效前仍须通过每次请求的管理员启用状态校验。

登录失效、页面刷新或重新登录不生成新的库存调整请求标识。结果未知的原调整 UUID/内容继续按管理员隔离保存，登录后只能在相同管理员身份下恢复，不得把另一管理员的请求自动重发。

### GET `/api/v1/admin/products`

商品列表。

### POST `/api/v1/admin/products`

创建商品。新普通商品库存为 0，首次补货同样通过 delta 调整接口记录。

### PUT `/api/v1/admin/products/{id}`

编辑商品信息。商品创建后 `product_type` 不可变，防止已有套餐引用和扣减快照的商品类型失效。不得覆盖已有商品 `stock`，库存变化必须通过 delta 调整接口。

### POST `/api/v1/admin/products/{id}/disable`

停用商品。

### POST `/api/v1/admin/products/{id}/stock`

只调整普通商品的当前可售库存，套餐不能设置独立库存。

```json
{
  "adjustment_request_id": "a97637c4-6854-4aa6-85b7-503e6a2eb62c",
  "delta": 12,
  "reason": "补充本批次烤制库存"
}
```

- `adjustment_request_id` 为 UUID，同一次调整重试复用原值。
- `delta` 是非 0 严格整数，拒绝布尔值和小数，正数补货、负数减量；结果不得小于 0 或溢出数据库 INTEGER 范围。
- `reason` 必填，1～500 字符且不能全为空白，原样保存；操作者从管理员身份取得，禁止客户端指定 `admin_user_id`。
- 指纹为 `{product_id, delta, reason}` 按第 4.1 节相同稳定 JSON 规则计算 SHA-256，`product_id` 取路径参数；标识不纳入指纹。
- 唯一约束为 `(admin_user_id, adjustment_request_id)`；同键同内容返回原调整结果，同键异内容返回 HTTP 409 / 40902。

先查询已有调整记录；缺失时开启事务并锁定目标商品，获得锁后再次查幂等记录，再检查类型和库存、执行 delta 条件更新并写 `stock_adjustments`。调整和记录同一事务提交。使用 READ COMMITTED 保证等待后重查看到已提交结果；并发唯一键冲突须回滚自身全部变更，再读胜者比较指纹。不得仅依赖更新后的非事务日志。

首次调整及同内容重放均 HTTP 200：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "adjustment_id": 31,
    "adjustment_request_id": "a97637c4-6854-4aa6-85b7-503e6a2eb62c",
    "product_id": 1,
    "delta": 12,
    "stock_before": 8,
    "stock_after": 20,
    "reason": "补充本批次烤制库存",
    "admin_user_id": 1,
    "created_at": "2026-10-10T09:30:00+08:00"
  }
}
```

重放返回原记录的前后值，不再次增减；这两个值不是重放时的实时库存。减量不足返回 HTTP 409 / 40901，库存与调整记录均不写入。

### GET `/api/v1/admin/categories`

分类管理。

### GET `/api/v1/admin/combos`

套餐管理。

### GET `/api/v1/admin/orders`

订单列表。

### GET `/api/v1/admin/orders/{id}`

订单详情。

### POST `/api/v1/admin/orders/{id}/making`

`paid → making`，支付状态须为 `success`。

### POST `/api/v1/admin/orders/{id}/complete`

`making → completed`，支付状态须为 `success`。

### GET `/api/v1/admin/dashboard`

基础经营概览。

---

## 8. 订单与支付状态组合

| 订单状态 | V1.0 允许的支付状态 |
|---|---|
| pending_payment | pending、failed |
| paid、making、completed | success |
| cancelled | closed |

`refunded` 仅为未来预留，本版本不产生。合法订单转换仍为：

```text
pending_payment → paid
pending_payment → cancelled
paid → making
making → completed
```

支付失败不增加订单状态，取消或超时统一关闭支付。其他组合或转换默认拒绝，客户端不能直接提交任意 status。详见 [订单状态机](order-state-machine.md)。

---

## 9. API 安全原则

1. 顾客 API 使用既有应用 Token，管理 API 使用独立管理员 JWT Bearer 认证；身份与 audience 必须区分。
2. 当前用户及操作者从认证上下文取得，订单操作必须检查归属。
3. 价格、商品状态、库存、支付确认和状态转换以后端规则为准。
4. 幂等标识不能替代认证，查重必须限定当前用户或管理员。
5. 指纹及快照由服务端生成，事务与数据库约束保证一致性，不能仅依靠前端禁止重复点击。

---

## 10. 幂等与并发验收

- 创建订单：同用户、同 UUID 并发请求最多提交一张订单、一次库存扣减和一条支付记录。
- 支付重放：读取原结果，不重复生成支付流水，不改变首次付款时间。
- 手动取消、超时扫描、到期支付请求共用订单锁和关闭服务，只恢复一次库存。
- 单品和多个套餐共用同一商品时，按普通商品 ID 汇总总消耗、升序扣减；恢复只用实际扣减快照。
- 管理调整与审计记录同事务，同标识重放不再增减，与下单并发不会覆盖对方修改。

字段及约束见 [数据库设计](database-design.md)，回归场景见 [测试计划](testing-plan.md)。
