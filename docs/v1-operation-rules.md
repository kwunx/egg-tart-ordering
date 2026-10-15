# V1.0 业务与运行边界规则

**范围已冻结**：V1.0 面向单摊位自取流程，首发 uni-app 微信小程序、Vue 管理端、FastAPI/PostgreSQL。支付仅为 Mock，不实际扣款；当前发布限受控体验与模拟验收，不作为真实营业系统。本文统一规定取餐流程、接单开关、商品配置一致性及 Mock 运行边界。

## 1. 当日取餐号

- 首次 Mock 成功支付时按数据库 `clock_timestamp()` 的 `Asia/Shanghai` 日期分配号码。每个日期从 1 开始，展示为 `A001`、`A002`，超过三位仍正常显示 `A1000`。
- 未支付、支付失败、取消订单不占号；成功后的重试、详情查询不更换号码。号码用于模拟取餐流程识别，不是查询订单的认证凭据，也不是实收或真实交付凭证。
- orders 增加 `pickup_date DATE NULL`、`pickup_sequence INTEGER NULL`、`UNIQUE(pickup_date,pickup_sequence)`。pending_payment/cancelled 两字段必须为 NULL；paid/making/ready/completed 均非 NULL，序号 >0。展示 `pickup_no` 派生自序号。
- 新表 `pickup_daily_counters(pickup_date DATE PRIMARY KEY, last_sequence INTEGER NOT NULL CHECK(last_sequence>0))`。支付成功事务锁定订单→payment 后，通过 `INSERT ... ON CONFLICT(pickup_date) DO UPDATE SET last_sequence = pickup_daily_counters.last_sequence+1 RETURNING last_sequence` 原子取号；取号、payment.success、order.paid 一起提交，回滚不占号。

## 2. 订单制作与交付

- 状态：`pending_payment → paid → making → ready → completed`，待支付也允许 `→ cancelled`；已支付及后续制作状态对应 payment.success。
- 保留完整流程状态用于验收：paid 是 Mock 确认成功、模拟待制作；making 是模拟制作中；ready 是模拟制作完成、待取餐；completed 是模拟确认已取餐。以上状态均不代表实际收款，不触发真实制作或交付。
- 管理 API：`POST /api/v1/admin/orders/{id}/ready` 仅 making→ready；原 `/complete` 改为仅 ready→completed；禁止 making→completed 直跳。

## 3. 暂停与恢复接单

- 新表 `store_settings(id SMALLINT PRIMARY KEY CHECK(id=1), ordering_enabled BOOLEAN NOT NULL DEFAULT TRUE, notice VARCHAR(200) NOT NULL DEFAULT '', updated_by_admin_id BIGINT NULL REFERENCES admin_users(id), updated_at TIMESTAMPTZ NOT NULL)`。迁移初始化固定 id=1，缺失应拒绝新单，不得默认接单。
- `GET /api/v1/store/status` 公开 `{ordering_enabled,notice}`；`GET /api/v1/admin/store/ordering` 供管理员读取；`PUT /api/v1/admin/store/ordering` 管理员设 `{ "ordering_enabled": false }` 或 true，重复设置相同目标幂等。
- 首次新单在同一事务中先占有 `(user_id,client_request_id)` 唯一键，再 `SELECT ... FROM store_settings WHERE id=1 FOR SHARE`，持有到提交；关闭时整单回滚并返回 HTTP 409、业务码 40906，不占库存。管理员更改状态 UPDATE 同行，与共享锁互斥，先持共享锁的订单可以提交；暂停已提交之后获得该锁的新单必须拒绝。该行同时承担第 5 节的目录配置锁，不增加新表。
- 暂停仅阻止首次新下单；已有订单同 UUID 幂等重放应先返回原单，既有订单支付、取消、超时释放库存、制作和取餐不受影响。顾客仍可看商品和历史订单。

## 4. UI 与验收

- 顾客端首页/结算显示暂停状态，后端拒绝 40906 后购物车内容保留。Mock 成功才显示模拟取餐号；ready 显示模拟待取餐、completed 显示模拟已取餐。顾客端和管理端全程显示“模拟支付，不实际收款”，不得仅在首次进入时提示。
- 管理员提供暂停/恢复开关、取餐号列表、开始制作/制作完成/确认取餐三个逐级动作。
- 使用真实 PostgreSQL 并发验证：同日不同订单取号唯一；跨北京时间午夜重置；成功重放不换号；模拟失败/取消不占号；making→completed 拒绝；暂停竞争与新单锁顺序；旧单支付、取消及超时任务不受暂停影响。

## 5. 商品配置一致性

数据库继续使用 `READ COMMITTED`。同一事务内的多次查询本身不能保证读取同一版本的商品配置；历史 `stock_deductions` 只保证已生成快照的恢复正确，也不能替代下单时的配置一致性保护。

所有目录配置写入统一遵守以下协议，包括分类名称、排序、启停，商品创建、价格、分类、名称、描述、图片、排序、启停，以及套餐数量和允许选择的成员集合：

1. 开启事务后，第一把业务行锁必须是 `store_settings(id=1) FOR UPDATE`，并持有到提交。不得先锁商品，也不得先取 `FOR SHARE` 后升级为独占锁。
2. 在该锁保护下读取并校验配置；如需锁定既有商品行，将涉及的商品 ID 去重并升序取得锁，再执行修改。新商品尚无既有行，按正常插入处理。
3. 商品属性及对应 `combo_products` 成员集合在同一事务内保存，完整提交或全部回滚。父商品行锁不会自动覆盖成员插入、删除、更新，因此所有成员写入也必须先遵循控制行协议。
4. 仅为目录编辑取得控制行锁时，不得修改 `ordering_enabled`、`notice`、`updated_at` 或 `updated_by_admin_id`，也不得顺带修改营业时间等营业配置；只有实际接单设置操作才修改其对应字段。

首次新单的固定顺序为：结构验证与请求规范化 → 查询已有幂等结果 → 事务内插入订单占唯一键 → `store_settings(id=1) FOR SHARE` 并检查接单开关 → 完整读取、校验商品/分类/套餐定义，计算价格和快照 → 按普通商品 ID 汇总、升序原子扣库存 → 保存最终金额、明细、唯一 payment → 提交。共享锁必须覆盖完整配置读取直到提交，不能读完价格即提前释放。已有订单同键重放优先返回原单，不重新取得配置锁、校验现行商品或改价。

若控制行 id=1 缺失或未成功取得所需锁，新单和目录写事务必须整体回滚，按通用 HTTP 500 处理，不能无锁继续或默认接单；`50301` 专用于第 6 节的环境/支付模式保护。

共享锁允许新单之间并行；目录编辑短时阻挡新单。谁先取得控制行锁，决定订单使用完整的旧定义或新定义，禁止出现旧套餐成员校验通过却按新价格成交的混合配置。事务内不执行外部调用，不等待用户操作。

库存 delta、既有订单支付/取号、取消/超时恢复，以及 making/ready/completed 流转不取得此控制行锁，沿用各自订单→payment→商品 ID 升序的必要锁顺序，不能持有商品锁后再反向申请控制行锁。所有目录写入口、后台脚本都必须遵守上述协议；迁移和初始化在维护状态下进行，避免在线绕过保护。V1 不增加配置版本表或改用更高隔离级别。

可售规则统一为：

- 商品自身 `is_active=true` 且所属分类 `is_active=true`，才对顾客有效；套餐本身及所选普通子商品分别满足该条件。
- 套餐配置的允许成员必须引用存在的 `single`，可以保留停用商品或停用分类中的普通商品引用；`combo-options` 只展示当前有效选项，新单最终在共享锁保护下重新校验。
- 分类停用不改写子商品原有 `is_active`。重新启用分类后，各商品仍依自身启用状态决定是否有效。
- 商品 `product_type` 创建后不可变；新普通商品 `stock=0`，首次补货走带审计记录的 delta 调整，通用目录编辑不得覆盖库存。
- 目录停售或配置变化不改历史快照，也不阻止既有订单支付、模拟制作/取餐或按原 `stock_deductions` 恢复库存。

验收必须覆盖套餐改价与成员变更在同一事务提交时的并发下单、分类停用竞争、共享锁允许两个新单并行，以及目录编辑与库存调整/取消并发无反向锁等待。结果只能符合修改前或修改后的完整定义。

## 6. Mock 运行与发布边界

- `APP_ENV` 必须显式设置为 `development`、`test`、`staging` 或 `production`；缺失、未知值（包括缩写 `dev`）均启动拒绝。`PAYMENT_MODE` 也必须显式设置，V1 唯一支持值为 `mock`；缺失或未知模式启动拒绝，`production + mock` 同样启动拒绝。V1 不实现真实微信收款，不通过填入 `wechat` 绕过此限制。
- 若运行期间意外出现不允许的环境/支付模式组合，业务保护检查返回 HTTP 503、业务码 `50301`，在创建订单、扣库存、支付、取号或订单流转等副作用发生之前停止处理。正常启动检查不能替代该运行时保护。
- `MOCK_ALLOWED_OPENIDS` 为后端逗号分隔的精确白名单，逐项去除首尾空白、去重后按完整 openid 比较，不接受前缀、通配符或客户端自报身份。后端用微信 code 换取 openid 后，仅向白名单顾客签发 Token；每次顾客受保护 API 请求还须按服务端用户身份重新检查名单。移出白名单后，既有 Token 也不能继续调用受保护接口。
- 名单为空时禁止所有顾客获取 Token 和调用受保护 API；不因此拒绝后端启动，管理员仍可使用管理功能，有效测试环境中的 worker 仍可回收已有过期订单。非白名单顾客返回 HTTP 403、业务码 `40301`。公开商品浏览及 `GET /api/v1/store/status` 不受顾客白名单限制。
- development/test/staging 使用与真实营业隔离的数据库和库存。体验版仅供授权白名单模拟验收；所有页面、管理操作和取餐号持续标明模拟、不实际收款，Mock 的 paid/making/ready/completed 和取餐号不进入真实制作与交付流程。接单开关只是体验环境内的业务开关，不是生产发布许可。
- V1 全部支付记录均为 `payment_method=mock`，不增加 `is_test` 表字段。真实营业另行设定支付、库存、制作交付和上线准入条件；本轮不要求接入真实支付，也不能把受控体验的完成认定为可正式营业。

验收覆盖缺失/未知配置及 production+mock 启动拒绝、运行保护 50301 且无业务副作用、空名单和非白名单 40301、已有 Token 在移出名单后失效、公开浏览仍可访问，以及有效环境下空名单不阻止 worker 清理。

以上为设计决定，不代表功能已实现或通过测试。
