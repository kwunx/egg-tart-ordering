# 订单状态机

## 1. 状态

```text
pending_payment
paid
making
completed
cancelled
```

## 2. 合法流转

```text
pending_payment
      ├──→ paid
      └──→ cancelled

paid
  ↓
making
  ↓
completed
```

## 3. 状态职责

### pending_payment

订单已创建，等待支付。

库存已经按照 V1.0 规则处理。

### paid

支付成功，等待制作。

### making

摊主已经开始制作。

### completed

订单完成。

### cancelled

订单取消。

待支付取消时需要释放库存。

## 4. 状态转换原则

状态转换必须由后端完成。

不能允许客户端直接：

```json
{
  "status": "completed"
}
```

## 5. 并发

状态更新前应检查当前状态，必要时锁定订单行。

例如：

```sql
UPDATE orders
SET status = 'making'
WHERE id = :id
AND status = 'paid';
```

根据影响行数判断转换是否成功。

## 6. 日志

每次状态变化应记录：

- order_id
- order_no
- old_status
- new_status
- operator
- created_at
