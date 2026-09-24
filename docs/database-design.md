# 微信小程序点餐系统 - 数据库设计

## 1. 数据库

PostgreSQL。

金额字段统一：

```sql
NUMERIC(10,2)
```

时间字段统一：

```sql
TIMESTAMPTZ
```

---

## 2. admin_users

管理员基础信息。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGSERIAL PK | 主键 |
| username | VARCHAR(50) UNIQUE | 用户名 |
| password_hash | VARCHAR(255) | 密码哈希 |
| is_active | BOOLEAN | 是否启用 |
| created_at | TIMESTAMPTZ | 创建时间 |
| updated_at | TIMESTAMPTZ | 更新时间 |

---

## 3. users

顾客用户基础表。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGSERIAL PK | 主键 |
| openid | VARCHAR(64) UNIQUE | 微信身份标识 |
| created_at | TIMESTAMPTZ | 创建时间 |
| updated_at | TIMESTAMPTZ | 更新时间 |

V1.0 只实现 `openid`。未来可增加 `unionid`、昵称、头像、手机号、状态、最后登录时间等。

---

## 4. categories

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGSERIAL PK | 主键 |
| name | VARCHAR(50) | 分类名 |
| sort_order | INTEGER | 排序 |
| is_active | BOOLEAN | 是否启用 |
| created_at | TIMESTAMPTZ | 创建时间 |
| updated_at | TIMESTAMPTZ | 更新时间 |

---

## 5. products

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGSERIAL PK | 主键 |
| category_id | BIGINT FK | 分类 |
| name | VARCHAR(100) | 商品名 |
| description | TEXT | 描述 |
| image_url | TEXT | 图片 |
| price | NUMERIC(10,2) | 价格 |
| product_type | VARCHAR(20) | single/combo |
| stock | INTEGER | 当前可售库存 |
| combo_quantity | INTEGER | 套餐总数量 |
| sort_order | INTEGER | 排序 |
| is_active | BOOLEAN | 是否启用 |
| created_at | TIMESTAMPTZ | 创建时间 |
| updated_at | TIMESTAMPTZ | 更新时间 |

约束：

- `product_type` 只能为 `single` 或 `combo`。
- `stock >= 0`。
- 普通商品使用库存。
- 套餐不拥有独立库存。
- `combo_quantity` 仅套餐使用。

---

## 6. combo_products

定义套餐允许选择哪些普通商品。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGSERIAL PK | 主键 |
| combo_id | BIGINT FK | 套餐商品 |
| product_id | BIGINT FK | 可选普通商品 |
| created_at | TIMESTAMPTZ | 创建时间 |

唯一约束：

```text
UNIQUE(combo_id, product_id)
```

---

## 7. orders

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGSERIAL PK | 主键 |
| order_no | VARCHAR(32) UNIQUE | 订单号 |
| user_id | BIGINT FK | 顾客 |
| total_amount | NUMERIC(10,2) | 总金额 |
| status | VARCHAR(30) | 订单状态 |
| remark | VARCHAR(500) | 备注 |
| created_at | TIMESTAMPTZ | 创建时间 |
| updated_at | TIMESTAMPTZ | 更新时间 |

状态：

- pending_payment
- paid
- making
- completed
- cancelled

---

## 8. order_items

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGSERIAL PK | 主键 |
| order_id | BIGINT FK | 订单 |
| product_id | BIGINT FK NULL | 商品 |
| product_name | VARCHAR(100) | 名称快照 |
| unit_price | NUMERIC(10,2) | 单价快照 |
| quantity | INTEGER | 数量 |
| subtotal | NUMERIC(10,2) | 小计 |
| item_type | VARCHAR(20) | single/combo |
| selections | JSONB | 套餐选择快照 |
| created_at | TIMESTAMPTZ | 创建时间 |

历史订单依赖快照，不依赖当前商品价格。

---

## 9. payments

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGSERIAL PK | 主键 |
| order_id | BIGINT UNIQUE FK | 订单 |
| payment_method | VARCHAR(30) | mock/wechat |
| payment_status | VARCHAR(30) | pending/success/failed/refunded |
| transaction_id | VARCHAR(100) | 支付流水 |
| amount | NUMERIC(10,2) | 支付金额 |
| paid_at | TIMESTAMPTZ | 支付时间 |
| created_at | TIMESTAMPTZ | 创建时间 |
| updated_at | TIMESTAMPTZ | 更新时间 |

---

## 10. 表关系

```text
users
  │
  └── orders
        │
        ├── order_items
        └── payments

categories
  │
  └── products
         │
         └── combo_products
```

---

## 11. 库存设计

V1.0 不增加独立 inventory 表，使用 `products.stock`。

下单时：

```sql
UPDATE products
SET stock = stock - 1
WHERE id = :id
AND stock >= 1;
```

必须检查受影响行数。

套餐下单时，对所选普通商品分别执行原子库存扣减。

如果任何一个商品库存不足，整个订单事务回滚。

取消/超时释放库存时：

```sql
UPDATE products
SET stock = stock + :quantity
WHERE id = :id;
```

释放操作必须与订单取消状态变化处于同一个事务。

---

## 12. 索引

建议：

- `products(category_id)`
- `products(is_active)`
- `orders(user_id)`
- `orders(status)`
- `orders(created_at DESC)`

唯一约束自动提供唯一索引：

- users.openid
- admin_users.username
- orders.order_no
- payments.order_id

---

## 13. 数据完整性

建议增加 CHECK：

```text
stock >= 0
price >= 0
quantity > 0
subtotal >= 0
total_amount >= 0
amount >= 0
```

外键根据历史数据保留需求决定删除策略。

商品采用停用而非物理删除，避免历史订单关联失效。
