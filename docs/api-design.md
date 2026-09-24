# 微信小程序点餐系统 - API 设计

## 1. API 前缀

```text
/api/v1/
```

统一 JSON：

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
  "code": 40001,
  "message": "库存不足",
  "data": null
}
```

HTTP 状态：

- 200 成功
- 201 创建成功
- 400 参数/业务错误
- 401 未认证
- 403 无权限
- 404 不存在
- 409 资源冲突
- 422 参数校验失败
- 500 服务端异常

---

## 2. 顾客认证

### POST `/api/v1/auth/login`

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

## 4. 订单

### POST `/api/v1/orders`

创建订单。

请求只提交：

- product_id
- quantity
- combo selections
- remark

**禁止信任前端 price、subtotal、total_amount。**

后端重新读取商品价格并计算金额。

处理：

```text
BEGIN
↓
校验商品
↓
校验套餐选择
↓
读取价格
↓
原子扣库存
↓
创建 order
↓
创建 order_items
↓
创建 payment
↓
COMMIT
```

库存不足：

```http
409 Conflict
```

### GET `/api/v1/orders/my`

只返回当前 Token 对应用户的订单。

### GET `/api/v1/orders/{order_id}`

只能访问当前用户自己的订单。

---

## 5. Mock 支付

### POST `/api/v1/orders/{order_id}/pay`

V1.0 模拟支付。

后端：

```text
验证订单属于当前用户
↓
验证订单状态 pending_payment
↓
验证 payment pending
↓
更新 payment = success
↓
更新 order = paid
```

支付成功与订单状态更新必须在事务中。

---

## 6. 取消订单

### POST `/api/v1/orders/{order_id}/cancel`

只允许待支付订单取消。

事务：

```text
BEGIN
↓
锁定订单
↓
检查状态
↓
恢复商品库存
↓
order → cancelled
↓
payment → failed/保持 pending，按最终规则处理
↓
COMMIT
```

必须防止重复取消导致库存被重复释放。

---

## 7. 管理端

### POST `/admin/login`

管理员登录。

### GET `/admin/products`

商品列表。

### POST `/admin/products`

创建商品。

### PUT `/admin/products/{id}`

编辑商品。

### POST `/admin/products/{id}/disable`

停用商品。

### POST `/admin/products/{id}/stock`

修改库存。

### GET `/admin/categories`

分类管理。

### GET `/admin/combos`

套餐管理。

### GET `/admin/orders`

订单列表。

### GET `/admin/orders/{id}`

订单详情。

### POST `/admin/orders/{id}/making`

`paid → making`

### POST `/admin/orders/{id}/complete`

`making → completed`

### GET `/admin/dashboard`

基础经营概览。

---

## 8. 状态转换

允许：

```text
pending_payment → paid
pending_payment → cancelled
paid → making
making → completed
```

其他转换默认拒绝。

所有状态转换由后端判断，前端不能直接提交任意 status。

---

## 9. API 安全原则

1. 顾客 API 使用 Token。
2. 管理 API 使用管理员认证。
3. 当前用户从 Token 得到，不能从请求参数决定。
4. 价格以后端数据库为准。
5. 库存以后端数据库为准。
6. 订单归属以后端 Token 为准。
7. 支付结果以后端逻辑为准。
8. 状态转换以后端规则为准。

---

## 10. 幂等与并发

支付接口必须考虑重复点击。

例如同一个订单连续调用两次：

```text
第一次 → success
第二次 → 不重复产生支付记录
```

订单取消同样必须幂等。

库存更新必须使用数据库原子条件更新。
