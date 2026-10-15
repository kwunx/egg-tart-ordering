# 订单状态机

## 1. 状态与允许组合

订单状态和支付状态分别保存，但必须符合以下组合：

| 订单状态 | 支付状态 | 含义及库存处理 |
|---|---|---|
| `pending_payment` | `pending` | 已创建订单并扣减库存，尚未确认支付结果 |
| `pending_payment` | `failed` | Mock 确认失败，保留库存，可在截止前重试 |
| `paid` | `success` | 已支付，等待制作，不再次扣库存 |
| `making` | `success` | 制作中 |
| `ready` | `success` | 制作完成，待取餐 |
| `completed` | `success` | 顾客已取餐 |
| `cancelled` | `closed` | 手动或超时关闭，原扣减库存已经恢复 |

V1.0 不允许其他组合。`refunded` 仅作为未来真实支付扩展的预留值，V1.0 不产生退款状态或退款流程。

订单初次创建时保存 `expires_at`；取消时保存 `cancel_reason=manual/timeout`，未取消时原因为空。已取消订单不可重新打开。

## 2. 合法流转

```text
pending_payment + pending/failed
      ├── Mock 成功 ──→ paid + success
      ├── Mock 失败 ──→ pending_payment + failed
      └── 手动/超时关闭 ──→ cancelled + closed

paid + success
      ↓ 管理员开始制作
making + success
      ↓ 管理员制作完成
ready + success
      ↓ 管理员确认取餐
completed + success
```

同单支付失败可复用唯一的 payment 记录再次尝试，不创建第二条支付主记录。状态转换和支付结果必须由后端决定，客户端不能提交任意 `status` 或“支付成功”参数。

## 3. 操作结果与重放

| 操作与当前状态 | 处理 | HTTP / 业务码 |
|---|---|---|
| 首次创建订单成功 | 同事务保存订单、支付、明细并扣库存 | 201 / 0 |
| 创建订单同用户同请求键、同内容 | 返回原订单当前状态，不重新计价或扣库存 | 200 / 0 |
| 创建订单同用户同请求键、不同内容 | 幂等内容冲突 | 409 / `40902` |
| 待支付且未到期，Mock 成功 | payment→success，order→paid，不更新库存 | 200 / 0 |
| 待支付且未到期，Mock 确认失败 | payment→failed，保留待支付订单和库存；提交后返回失败 | 409 / `40905` |
| paid/making/ready/completed，再次支付 | 返回当前订单和首次支付结果，不修改支付流水及时间 | 200 / 0 |
| 待支付但已到期，尝试支付 | 先关闭并恢复库存，提交后返回过期 | 409 / `40904` |
| 已因 timeout 关闭，再次支付 | 保留关闭状态，不恢复第二次库存 | 409 / `40904` |
| 已因 manual 关闭，再次支付 | 不允许支付 | 409 / `40903` |
| 待支付，手动取消 | 关闭并恢复库存；已到期优先认定 timeout | 200 / 0 |
| 已取消，再次取消 | 返回原关闭结果，保留首次原因，不更新库存 | 200 / 0 |
| paid/making/ready/completed，尝试取消 | 拒绝取消 | 409 / `40903` |
| paid，管理员开始制作 | order→making，payment 保持 success | 200 / 0 |
| making，管理员制作完成 | order→ready，payment 保持 success | 200 / 0 |
| ready，管理员确认取餐 | order→completed，payment 保持 success | 200 / 0 |

未列出的状态转换默认拒绝，返回 HTTP 409、业务码 `40903`；认证、归属和参数错误按对应 API 规则处理。管理员制作/完成不提供任意目标状态写入接口。

创建订单使用 UUID `client_request_id`，数据库约束 `UNIQUE(user_id, client_request_id)`；结构验证及幂等查询后，先在同一事务插入订单行占唯一键（金额 0 仅为事务内占位），再校验当前商品/套餐、扣库存并填入最终金额、明细快照和 payment，完整提交。整单失败全部回滚，不留下占位订单。详情见 [库存设计](inventory-design.md)。

## 4. Mock 支付失败与重试

Mock 默认成功；失败仅由服务端测试配置控制，不接受客户端指定支付结果。V1.0 不对接外部支付服务。

- 确认的 Mock 失败是业务结果：将 payment 更新为 `failed` 并提交，订单仍为 `pending_payment`，库存仍被预占。
- 截止前再次显式调用支付接口可复用该 payment 发起新一次 Mock 尝试；未到期的 `pending` 和 `failed` 均可尝试。
- 数据库或程序异常不等于 Mock 确认失败，事务应回滚，不留下订单与支付不匹配的中间状态。
- `transaction_id` 和 `paid_at` 仅在首次成功时写入；成功后的重复请求不重建流水、不刷新时间。
- 网络结果未知时，客户端先查询订单，再决定是否继续付款；创建订单的请求 UUID 不充当支付尝试标识。

V1.0 支付幂等保证首次成功后不重复产生业务效果；不要求保留并重放每次已失败尝试的响应，不新增支付尝试表。所有尝试均通过订单锁串行裁决。

## 5. 事务与统一锁顺序

数据库事务隔离级别统一使用 `READ COMMITTED`。支付、手动取消和超时回收必须使用共同的订单业务服务，按以下顺序取得所需行锁：

```text
BEGIN
↓
锁定订单，验证归属/调用权限
↓
锁定该订单唯一的 payment
↓
重查订单与支付状态，取得数据库当前时间
↓
裁决重放、支付、失败或关闭
↓
如果关闭：按商品 ID 升序恢复 stock_deductions 汇总库存
↓
同步写入订单及支付状态、原因/首次成功信息
↓
COMMIT
↓
返回成功或已提交的业务错误
```

锁后必须重查当前状态，不能根据锁前读取的 `pending_payment` 执行更新。支付、取消、超时回收都先锁订单再锁支付，不能使用相反次序。库存操作对整单普通商品汇总后按 ID 升序进行。

库存恢复、`cancelled`、`closed` 和取消原因必须同事务提交。库存只在首次有效关闭时恢复，后续重放或竞争失败者不得再次恢复。历史商品停用不阻止恢复；恢复使用订单明细实际扣减快照，不读取现行套餐配置。

管理员 `paid→making`、`making→ready`、`ready→completed` 可用条件更新并检查影响行数：

```sql
UPDATE orders
SET status = 'making'
WHERE id = :id
  AND status = 'paid';
```

管理员制作流程不修改支付状态或库存。影响 0 行时重新读取结果并按非法状态处理，不能报告转换成功。

## 6. 到期边界与关闭提交

默认支付时限 `PAYMENT_TIMEOUT_SECONDS=900`。首次插入订单行占唯一键时取一次 `t=clock_timestamp()`，保存 `created_at=t`、`expires_at=t+PAYMENT_TIMEOUT_SECONDS`；后续等待锁、填入最终金额、重放及配置变化均不延长已有订单期限。

取得订单及 payment 行锁之后，用 PostgreSQL `clock_timestamp()` 取得当前时间，`当前时间 >= expires_at` 即到期。`now()` / `CURRENT_TIMESTAMP` 固定于事务开始，不能用于锁等待后的截止判断。

先处理已支付或已关闭终态的重放；仅对待支付订单判断到期。支付和手动取消都会在锁后检查截止时间，到期时统一关闭并写入 `cancel_reason=timeout`；未到期的手动取消写入 `manual`。已关闭订单保留首次原因。

到期支付请求的处理顺序必须为：

```text
事务内：恢复库存 + order.cancelled + payment.closed + cancel_reason.timeout
↓
提交成功
↓
事务外：HTTP 409 / 40904 ORDER_EXPIRED
```

业务服务应返回需要提交的处理结果，待事务提交后再由接口层映射为错误响应。不得在事务内部抛出导致回滚的 HTTP/业务异常。Mock 确认失败也先提交 `payment.failed`，再返回 HTTP 409 / `40905`；真正的数据库或程序异常仍全部回滚。

独立轻量 worker 每 `ORDER_EXPIRY_SCAN_INTERVAL_SECONDS=30` 秒扫描，并在 worker 启动时补扫。每单独立事务、锁后重查，复用同一关闭逻辑，支持多实例、重启和重复扫描；不在每个 Web worker 中创建定时器，不引入 Redis/Celery。扫描延迟不提供额外付款宽限期。

## 7. 日志与验收不变量

每次真实订单状态变化记录 `order_id`、`order_no`、旧/新状态、operator、时间；关闭时记录原因，支付尝试记录成功或确认失败结果。无状态变化的重放可记录请求日志，不伪造第二次业务状态变更。

必须验证：

- 任意已提交记录都符合第 1 节的订单/支付状态组合。
- 支付与取消/超时并发，只能得到 `paid + success` 或 `cancelled + closed`，不能出现成功支付但库存已释放。
- 双支付不重复写首次支付信息；双取消、手动取消与自动回收竞争只恢复一次。
- Mock 失败可以在截止前重试成功，也可以手动或超时关闭。
- 等待行锁期间跨越到期时刻，必须按锁后当前时间关闭；返回过期错误后数据库中的关闭和库存恢复已提交。
- worker 重复扫描及中断重启不重复释放；事务异常不产生部分支付、部分关闭或部分库存恢复。


---

## 8. 取餐号与暂停接单

- 首次成功支付分配取餐编号和付款成功同事务，失败/取消不分配；paid/making/ready/completed 支付状态保持 success。
- 制作完成只能 making→ready，确认顾客取餐才能 ready→completed。
- 首次下单在幂等占位后锁定接单单例配置，暂停时回滚并返回 40906；旧单流转不受暂停影响，详见 [V1.0 自取与暂停接单规则](v1-operation-rules.md)。
