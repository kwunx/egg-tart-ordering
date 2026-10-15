# V1.0 自取与暂停接单专项规则

**范围已冻结**：V1.0 单摊位自取，首发 uni-app 微信小程序、Vue 管理端、FastAPI/PostgreSQL，支付仍是 Mock，不实际扣款。

## 1. 当日取餐号

- 首次 Mock 成功支付时按数据库 `clock_timestamp()` 的 `Asia/Shanghai` 日期分配号码。每个日期从 1 开始，展示为 `A001`、`A002`，超过三位仍正常显示 `A1000`。
- 未支付、支付失败、取消订单不占号；成功后的重试、详情查询不更换号码。号码仅用于现场识别，不是查询订单的认证凭据。
- orders 增加 `pickup_date DATE NULL`、`pickup_sequence INTEGER NULL`、`UNIQUE(pickup_date,pickup_sequence)`。pending_payment/cancelled 两字段必须为 NULL；paid/making/ready/completed 均非 NULL，序号 >0。展示 `pickup_no` 派生自序号。
- 新表 `pickup_daily_counters(pickup_date DATE PRIMARY KEY, last_sequence INTEGER NOT NULL CHECK(last_sequence>0))`。支付成功事务锁定订单→payment 后，通过 `INSERT ... ON CONFLICT(pickup_date) DO UPDATE SET last_sequence = pickup_daily_counters.last_sequence+1 RETURNING last_sequence` 原子取号；取号、payment.success、order.paid 一起提交，回滚不占号。

## 2. 订单制作与交付

- 状态：`pending_payment → paid → making → ready → completed`，待支付也允许 `→ cancelled`；已支付及后续制作状态对应 payment.success。
- paid 是已支付待制作；making 是制作中；ready 是制作完成待顾客取餐；completed 是顾客已取餐。
- 管理 API：`POST /api/v1/admin/orders/{id}/ready` 仅 making→ready；原 `/complete` 改为仅 ready→completed；禁止 making→completed 直跳。

## 3. 暂停与恢复接单

- 新表 `store_settings(id SMALLINT PRIMARY KEY CHECK(id=1), ordering_enabled BOOLEAN NOT NULL DEFAULT TRUE, notice VARCHAR(200) NOT NULL DEFAULT '', updated_by_admin_id BIGINT NULL REFERENCES admin_users(id), updated_at TIMESTAMPTZ NOT NULL)`。迁移初始化固定 id=1，缺失应拒绝新单，不得默认接单。
- `GET /api/v1/store/status` 公开 `{ordering_enabled,notice}`；`GET /api/v1/admin/store/ordering` 供管理员读取；`PUT /api/v1/admin/store/ordering` 管理员设 `{ "ordering_enabled": false }` 或 true，重复设置相同目标幂等。
- 首次新单在同一事务中先占有 `(user_id,client_request_id)` 唯一键，再 `SELECT ... FROM store_settings WHERE id=1 FOR SHARE`；关闭时整单回滚并返回 HTTP 409、业务码 40906，不占库存。管理员更改状态 UPDATE 同行，与共享锁互斥，先持共享锁的订单可以提交；暂停已提交之后获得该锁的新单必须拒绝。
- 暂停仅阻止首次新下单；已有订单同 UUID 幂等重放应先返回原单，既有订单支付、取消、超时释放库存、制作和取餐不受影响。顾客仍可看商品和历史订单。

## 4. UI 与验收

- 顾客端首页/结算显示暂停状态，后端拒绝 40906 后购物车内容保留。支付成功才显示取餐号；ready 显示待取餐、completed 显示已取餐。
- 管理员提供暂停/恢复开关、取餐号列表、开始制作/制作完成/确认取餐三个逐级动作。
- 使用真实 PostgreSQL 并发验证：同日不同订单取号唯一；跨北京时间午夜重置；成功重放不换号；模拟失败/取消不占号；making→completed 拒绝；暂停竞争与新单锁顺序；旧单支付、取消及超时任务不受暂停影响。

以上为设计决定，不代表功能已实现或通过测试。