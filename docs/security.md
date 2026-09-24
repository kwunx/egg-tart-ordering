# 安全设计

## 1. 身份认证

顾客：

```text
wx.login
↓
code
↓
后端换取 openid
↓
应用 Token
```

客户端后续使用：

```http
Authorization: Bearer <token>
```

不允许：

```http
/orders/my?openid=xxx
```

作为身份认证方式。

## 2. Token

Token 应具有有效期。

Token 失效返回：

```http
401 Unauthorized
```

小程序重新登录获取 Token。

Token 不写入代码仓库。

## 3. 用户数据隔离

查询订单时：

```text
当前用户 ID
+
订单 ID
```

必须同时匹配。

## 4. 管理员

管理接口必须验证管理员身份。

普通顾客 Token 不能访问管理 API。

## 5. 价格安全

前端传递的价格不可信。

服务器：

```text
product_id
↓
数据库读取价格
↓
Decimal 计算
```

## 6. 库存安全

库存不接受前端直接设置。

普通下单：

```text
数据库原子 UPDATE
```

管理端修改库存：

```text
管理员权限
+
服务端校验
```

## 7. 支付安全

Mock 支付阶段：

- 支付结果由后端决定。
- 客户端不能直接把订单改成 paid。

未来接真实微信支付：

- 必须验证支付回调。
- 必须考虑回调重复。
- 必须保证支付处理幂等。
- 必须考虑退款。

## 8. 配置安全

`.env`：

```text
不提交 Git
```

`.env.example`：

```text
只保留变量名和示例值
```

敏感信息：

- 数据库密码
- Token 密钥
- 微信 AppSecret
- 真实支付密钥

均不能写入代码仓库。

## 9. 输入校验

所有 API 使用 Pydantic 校验：

- 类型
- 长度
- 数量
- 金额
- ID

数据库层继续使用 CHECK/UNIQUE/FK 等约束。

## 10. 日志

日志不能记录：

- AppSecret
- Token
- 密码
- 支付密钥

关键业务日志使用订单号等非敏感标识进行关联。
