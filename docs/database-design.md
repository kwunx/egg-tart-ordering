# 微信小程序点餐系统 - 数据库设计

## 1. 数据库

PostgreSQL，业务事务使用 READ COMMITTED。V1.0 共 8 张核心业务表、1 张 `stock_adjustments` 调整表及 2 张运营辅助表（`pickup_daily_counters`、`store_settings`），共 11 张；不引入独立余额表、库存预占表、通用幂等表、支付尝试表或配置版本表。目录配置一致性复用 `store_settings(id=1)` 的共享/独占行锁，规则见 [V1.0 业务与运行边界规则](v1-operation-rules.md#5-商品配置一致性)。

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
| name | VARCHAR(50) NOT NULL | 分类名，非空白 |
| sort_order | INTEGER NOT NULL DEFAULT 0 | 排序 |
| is_active | BOOLEAN NOT NULL DEFAULT TRUE | 是否启用 |
| created_at | TIMESTAMPTZ NOT NULL | 创建时间 |
| updated_at | TIMESTAMPTZ NOT NULL | 更新时间 |

分类停用不批量改写子商品 `is_active`。顾客有效性同时检查商品与所属分类的启用状态；分类名称、排序、启停等全部配置写入先取得 `store_settings(id=1) FOR UPDATE`，再读取/修改目录数据，并在同一事务提交。

---

## 5. products

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGSERIAL PK | 主键 |
| category_id | BIGINT NOT NULL FK | 所属分类必须存在 |
| name | VARCHAR(100) NOT NULL | 商品名，非空白 |
| description | TEXT NULL | 描述，可空 |
| image_url | TEXT NULL | 图片，可空 |
| price | NUMERIC(10,2) NOT NULL | 价格，非负 |
| product_type | VARCHAR(20) NOT NULL | single/combo，创建后不可变 |
| stock | INTEGER NOT NULL DEFAULT 0 | 当前可售库存；新普通商品为 0，套餐固定为 0 |
| combo_quantity | INTEGER NULL | 每份套餐所需数量；单品为 NULL，套餐为正整数 |
| sort_order | INTEGER NOT NULL DEFAULT 0 | 排序 |
| is_active | BOOLEAN NOT NULL DEFAULT TRUE | 商品自身是否启用 |
| created_at | TIMESTAMPTZ NOT NULL | 创建时间 |
| updated_at | TIMESTAMPTZ NOT NULL | 更新时间 |

约束：

- `product_type` 只能为 `single` 或 `combo`。创建后类型不可变，由服务层拒绝通用编辑中的类型变更，保持套餐引用与既有扣减快照有效。
- `stock >= 0`，新普通商品库存为 0，首次补货通过 delta 调整写入审计记录。
- 只有普通商品拥有可售库存；套餐 `stock=0`，不独立扣减。
- 单品 `combo_quantity IS NULL`；套餐 `combo_quantity IS NOT NULL AND combo_quantity > 0`，表示每份所需数量，条件 CHECK 不得让 NULL 绕过。
- 管理端通用商品编辑禁止覆盖 stock；库存增减只能经订单事务、关闭事务或有审计记录的 delta 调整事务。
- 顾客可售条件为商品自身及所属分类均启用；套餐本身和所选普通子商品分别满足该条件。后台允许配置停用商品，不能把目录存在性与顾客可售性混为一项。
- 创建、编辑及停用均先取得控制行独占锁；需要锁定的既有商品按 ID 升序取得锁。套餐价格、数量、属性和成员集合原子保存，不能分成多个独立提交。

---

## 6. combo_products

定义套餐允许选择哪些普通商品。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGSERIAL PK | 主键 |
| combo_id | BIGINT NOT NULL FK | 套餐商品 |
| product_id | BIGINT NOT NULL FK | 允许选择的普通商品 |
| created_at | TIMESTAMPTZ NOT NULL | 创建时间 |

唯一约束：

```text
UNIQUE(combo_id, product_id)
```

两端类型由服务层在控制行锁保护下验证为 combo/single。允许成员可引用停用但仍存在的普通商品，顾客 `combo-options` 仅展示商品及其分类均启用的成员；下单时再次检查。完整套餐定义保存时成员集合与 products 同事务替换，不允许绕过控制行锁单独增删关系。单品不拥有套餐成员，套餐必须有至少一个允许成员；成员 ID 不得重复。

---

## 7. orders

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGSERIAL PK | 主键 |
| order_no | VARCHAR(32) UNIQUE | 订单号 |
| user_id | BIGINT NOT NULL FK | 顾客，也是订单创建幂等范围 |
| client_request_id | UUID NOT NULL | 客户端创建请求标识 |
| request_hash | CHAR(64) NOT NULL | 规范化业务内容的 SHA-256 小写十六进制指纹 |
| total_amount | NUMERIC(10,2) NOT NULL | 最终总额；未提交占位行可先暂填 0 |
| status | VARCHAR(30) NOT NULL | 订单状态 |
| remark | VARCHAR(500) NOT NULL DEFAULT '' | 缺省/NULL 标准化为空字符串 |
| expires_at | TIMESTAMPTZ NOT NULL | 固化支付截止时间 |
| cancel_reason | VARCHAR(20) NULL | 仅取消时为 manual 或 timeout |
| pickup_date | DATE NULL | 支付成功对应 Asia/Shanghai 经营日 |
| pickup_sequence | INTEGER NULL | 当日正整数取餐序号 |
| created_at | TIMESTAMPTZ | 创建时间 |
| updated_at | TIMESTAMPTZ | 更新时间 |

状态：

- pending_payment
- paid
- making
- ready
- completed
- cancelled

---

### 7.1 订单幂等及时间约束

- `UNIQUE(user_id, client_request_id)`；同标识不同用户互不冲突。
- `request_hash` 仅包含规范化 items、quantity、selections 和 remark，不包含客户端价格、服务端当前数据或时间。规范化配方、合并等价行及稳定 JSON 算法以 [API 设计第 4.1 节](api-design.md#41-规范化与幂等结果) 为准。
- `expires_at > created_at`。首次插入订单行时从数据库 `clock_timestamp()` 取得同一个 t，写 `created_at=t`、`expires_at=t + PAYMENT_TIMEOUT_SECONDS`；默认 900 秒，提交前后不再重置此期限。
- `cancel_reason` 仅允许 NULL、manual、timeout。`status=cancelled` 必须有取消原因，其他状态必须为 NULL。
- 创建事务先插入 orders 行占有唯一键，再取得 `store_settings(id=1) FOR SHARE` 并检查接单开关；在共享锁持续保护下完整读取当前商品/分类/套餐/价格并校验、生成快照，按普通商品 ID 升序扣库存，最后补齐最终总额、明细及唯一 payment 后提交。任何业务校验或写入失败全部回滚，不能留下占位订单。
- 原单同键重放优先于当前接单开关及商品配置校验，返回原订单，不重新计价、生成快照或扣库存。控制行和库存锁等待都不延长首次插入时固化的 expires_at。
- 同键并发唯一约束争用失败者必须先回滚自身事务，再读取胜者按指纹返回已有订单或冲突；不得在占有唯一键前扣库存，也不得在失败事务内继续查询。

---

## 8. order_items

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGSERIAL PK | 主键 |
| order_id | BIGINT NOT NULL FK | 订单 |
| product_id | BIGINT FK NULL | 商品 |
| product_name | VARCHAR(100) | 名称快照 |
| unit_price | NUMERIC(10,2) NOT NULL | 单价快照 |
| quantity | INTEGER NOT NULL | 单品个数或套餐份数，必须大于 0 |
| subtotal | NUMERIC(10,2) NOT NULL | 小计 |
| item_type | VARCHAR(20) NOT NULL | single/combo |
| selections | JSONB NOT NULL DEFAULT '[]' | 每份套餐选择及子商品名称快照；单品为空数组 |
| stock_deductions | JSONB NOT NULL | 该行实际消耗的普通商品 ID/总数量快照，非空数组 |
| created_at | TIMESTAMPTZ | 创建时间 |

历史订单依赖快照，不依赖当前商品名称、价格或套餐配置。`unit_price` 是单个商品或每份套餐价格，`subtotal = unit_price × quantity`。

同商品同配方可合并行数量，不同配方分行。套餐 `selections` 存每份配方，子商品 ID 不重复并升序；示例为两份“6 个任选”：

```json
{
  "quantity": 2,
  "selections": [
    {"product_id": 1, "product_name": "原味蛋挞", "quantity": 2},
    {"product_id": 2, "product_name": "芝士蛋挞", "quantity": 2},
    {"product_id": 3, "product_name": "芋泥蛋挞", "quantity": 2}
  ],
  "stock_deductions": [
    {"product_id": 1, "quantity": 4},
    {"product_id": 2, "quantity": 4},
    {"product_id": 3, "quantity": 4}
  ]
}
```

单品同样保存扣减快照，例如商品 1 购买 3 个：

```json
{
  "quantity": 3,
  "selections": [],
  "stock_deductions": [{"product_id": 1, "quantity": 3}]
}
```

两类 JSON 均由服务端生成并在创建后保持不变。实际扣减快照按普通商品 ID 聚合、升序保存，所有 quantity 为正整数；套餐快照实际数量等于每份选择数量乘份数。下单按各行快照汇总整单总消耗；取消/超时原样汇总这些快照恢复库存，不能重新读取当前套餐定义推算。

---

## 9. payments

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGSERIAL PK | 主键 |
| order_id | BIGINT NOT NULL UNIQUE FK | 每个订单恰好一条支付记录 |
| payment_method | VARCHAR(30) NOT NULL DEFAULT 'mock' | V1 仅允许 mock，表示模拟确认而非实收 |
| payment_status | VARCHAR(30) NOT NULL DEFAULT 'pending' | pending/success/failed/closed；refunded 仅未来预留 |
| transaction_id | VARCHAR(100) NULL | 成功时生成的支付流水；成功重放不得改变 |
| amount | NUMERIC(10,2) NOT NULL | 支付金额 |
| paid_at | TIMESTAMPTZ NULL | 首次成功支付时间；成功重放不得改变 |
| created_at | TIMESTAMPTZ | 创建时间 |
| updated_at | TIMESTAMPTZ | 更新时间 |

---

### 9.1 支付组合与重试

| 订单状态 | 支付状态 |
|---|---|
| pending_payment | pending、failed |
| paid、making、ready、completed | success |
| cancelled | closed |

未到期的 failed 可在原 payment 行上重试；失败不释放库存、不改变订单期限。取消/超时把 pending 或 failed 置为 closed。success 必须有 transaction_id 和 paid_at，后续重复支付请求只返回原结果；pending、failed、closed 的这两个字段为 NULL。refunded 仅未来预留，V1.0 不生成该状态。

V1 的 `success`、流水和时间仅记录 Mock 确认，paid/making/ready/completed 均用于模拟验收，不证明实际收款或交付。数据库为 `payment_method` 增加 `CHECK(payment_method = 'mock')`；未来接入 wechat 须另行迁移，不能在当前 V1 写入。所有订单都属于隔离的 Mock 环境，不新增 `is_test` 列；运行环境和顾客白名单约束见 [运行与发布边界](v1-operation-rules.md#6-mock-运行与发布边界)。

订单状态、支付状态及库存变化必须同一事务提交，支付金额等于原订单总额。跨表组合及金额一致性由服务层事务保证，不能用普通单表 CHECK 假装覆盖；须有集成测试验证。详细转换以 [订单状态机](order-state-machine.md) 为准。

---

## 10. stock_adjustments

仅记录管理员对可售库存的增减及首次补货。它是审计记录，不是库存余额、库存预占或全量库存流水表；库存唯一余额仍是 `products.stock`。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGSERIAL PK | 调整记录主键 |
| adjustment_request_id | UUID NOT NULL | 管理端本次调整请求标识 |
| request_hash | CHAR(64) NOT NULL | 规范化 product_id/delta/reason 的 SHA-256 指纹 |
| product_id | BIGINT NOT NULL FK | 被调整的普通商品 |
| delta | INTEGER NOT NULL | 非 0 增减量 |
| stock_before | INTEGER NOT NULL | 本次调整前的可售库存 |
| stock_after | INTEGER NOT NULL | 本次调整后的可售库存 |
| reason | VARCHAR(500) NOT NULL | 1～500 字符，不能全空白，原样保存 |
| admin_user_id | BIGINT NOT NULL FK | 从管理员认证上下文获取 |
| created_at | TIMESTAMPTZ NOT NULL | 调整时间 |

约束：

- `UNIQUE(admin_user_id, adjustment_request_id)`；不同管理员的标识互不冲突。
- `delta <> 0`，`stock_before >= 0`，`stock_after >= 0`，`stock_after::BIGINT = stock_before::BIGINT + delta::BIGINT`。
- 指纹使用 [API 设计](api-design.md) 定义的稳定 JSON 算法；商品 ID 纳入指纹，防止同一标识跨商品被重用。请求标识不纳入指纹。
- 库存更新与审计记录同事务，成功记录不得覆写。同键同内容返回此原记录，同键异内容冲突。
- 先查幂等记录；缺失时锁定商品行，再重查记录，之后才校验类型、执行 delta 更新和插入记录。READ COMMITTED 保证锁等待后重查看到已提交结果。
- 并发 UNIQUE 冲突时完整回滚自身库存变化，再读取胜者比较指纹；不得留下“库存已变而记录未写”的状态。

---

## 11. 表关系

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
         ├── combo_products
         └── stock_adjustments
                  │
                  └── admin_users
```

---

## 12. 库存与事务

不增加独立 inventory 余额表，使用 `products.stock`。新增的 stock_adjustments 只记录管理员调整。

创建订单先取得 orders 幂等唯一键，再取 `store_settings(id=1) FOR SHARE` 并检查接单状态，在共享锁保护下读取、校验完整商品/分类/套餐定义并从服务端生成快照，按普通商品 ID 汇总整单消耗并升序扣减；该共享锁持有到提交。数量参数必须是正整数：

```sql
UPDATE products
SET stock = stock - :quantity
WHERE id = :id
  AND product_type = 'single'
  AND is_active = TRUE
  AND stock >= :quantity;
```

商品及其分类均启用的资格先在共享锁保护下校验；目录写操作遵守独占锁协议，因此资格、价格和成员集合在订单提交前不会改变。库存仍可能被其他订单或调整改变，须依靠上述条件更新裁决。检查受影响行数，0 行时须区分商品不存在/停售等不可售状态与数量不足，仅库存不足映射 40901；任何失败均整单回滚。不能把同一商品在不同单品/套餐中的消耗遗漏，也不能仅扣一份套餐。

所有目录配置写事务的第一把业务行锁为 `store_settings(id=1) FOR UPDATE` 并持有到提交，随后所需既有商品按 ID 升序锁定，商品及 combo_products 同事务保存。禁止先锁商品再取控制行，或先共享后升级；只取配置锁时不改写接单开关、公告、时间和操作者。所有在线写入口均遵守协议，迁移/初始化在维护状态执行；事务中不等待外部调用或人工操作。该协议使 READ COMMITTED 下多次配置查询仍使用一致定义；不另设版本表。控制行缺失或未成功取得锁时，新单/目录写整体回滚并返回通用 HTTP 500，不得无锁继续；50301 只用于环境/支付模式保护。

支付、取消、超时关闭统一先锁 orders，再锁 payments，最后按普通商品 ID 升序锁必要商品；既有订单操作不获取 store_settings 控制行锁。取得必要订单/payment 锁后用数据库 `clock_timestamp()` 判断是否达到 expires_at；只有仍为 pending_payment 的订单可关闭。关闭按原 `stock_deductions` 聚合结果恢复：

```sql
UPDATE products
SET stock = stock + :quantity
WHERE id = :id;
```

恢复仅按原实际扣减快照的商品 ID 和数量，不受当前停售状态影响；每次必须影响 1 行，引用缺失等异常须整单回滚。恢复库存、订单 cancelled、cancel_reason 和 payment closed 必须同一事务提交，任一步失败全部回滚。重复关闭只返回原结果。支付服务发现过期后须提交关闭事务，再返回 40904，不能因错误响应回滚库存恢复；取消接口无论此次关闭原因为 manual 还是 timeout，提交后均返回 200。

库存 delta 调整采用目标商品行锁及条件更新，事务内保存前后值和调整记录：

```sql
UPDATE products
SET stock = stock + :delta
WHERE id = :id
  AND product_type = 'single'
  AND stock::BIGINT + :delta BETWEEN 0 AND 2147483647
RETURNING stock;
```

delta 需预先校验为非 0 严格整数且在 INTEGER 范围内；商品锁后先重查幂等记录，再决定是否实际增减。普通商品创建时为 0，首次补货也走此流程，通用商品编辑不得覆盖 stock。库存调整不获取 store_settings、订单或支付锁；支付取号及 making/ready/completed 流转同样不获取控制行锁，避免反向锁序。

超时扫描间隔默认 `ORDER_EXPIRY_SCAN_INTERVAL_SECONDS=30` 秒，查询待支付且 `expires_at <= clock_timestamp()` 的订单，逐单进入统一关闭服务；锁后必须再次检查状态和时间。时限默认 `PAYMENT_TIMEOUT_SECONDS=900` 秒。任务可重复执行，重启后补扫；不需要单独的超时任务表。

---

## 13. 索引

建议：

- `products(category_id)`
- `products(is_active)`
- `orders(user_id)`
- `orders(status)`
- `orders(created_at DESC)`
- `stock_adjustments(product_id, created_at DESC)`
- 到期扫描部分索引：`orders(expires_at, id) WHERE status = 'pending_payment'`

唯一约束自动提供唯一索引：

- users.openid
- admin_users.username
- orders.order_no
- payments.order_id
- orders(user_id, client_request_id)
- stock_adjustments(admin_user_id, adjustment_request_id)

---

## 14. 数据完整性

订单/支付/明细中的身份外键、金额、状态、数量和快照等业务必需字段必须 NOT NULL，以上表显式标明 NULL 的字段为例外（例如 order_items.product_id）；新增字段的可空性以上表为准。created_at/updated_at 使用数据库时间；订单 created_at 与 expires_at 按第 7.1 节一次生成同一基准。API 的 server_time 在每次响应时读取数据库当前时间，不增加持久化字段。

必须增加相应 CHECK，而非只依赖 Pydantic：

```text
stock >= 0
price >= 0
quantity > 0
unit_price >= 0
subtotal >= 0 AND subtotal = unit_price * quantity
total_amount >= 0
amount >= 0
expires_at > created_at
product_type / item_type 只能为 single 或 combo
status / payment_status / cancel_reason 限定本文枚举
payment_method = 'mock'
request_hash 必须是 64 位小写十六进制
jsonb_typeof(selections) = 'array'
jsonb_typeof(stock_deductions) = 'array' AND jsonb_array_length(stock_deductions) > 0
```

另按字段章节落实产品类型/套餐数量条件、取消原因与状态组合、支付时间/流水与状态组合、库存调整前后值等单表约束。NOT NULL 与 CHECK 配合使用，不能依靠 `quantity > 0` 单独排除 NULL。

JSON 子字段必须由服务端验证：子商品 ID 唯一、为正整数，数量是严格正整数，拒绝布尔值和小数；selections 每份数量合计与套餐一致，stock_deductions 与首次实际扣减一致。单品 selections 为 `[]`；套餐 selections 非空；快照创建后不随商品变化修改。

跨表条件由事务服务保证：combo_products 两端分别是套餐/普通商品，套餐允许成员非空、单品无成员，顾客可售商品及所属分类均启用，调整对象是普通商品，订单总额等于明细和、支付金额等于订单额；不得将这些规则误写为无法引用其他表的普通 CHECK。目录非空白名称由 API 校验并以单表 CHECK 兜底。

商品采用停用而非物理删除，保留历史订单、扣减快照和库存调整引用。涉及商品、订单、支付与调整记录的外键不允许级联删除历史业务数据；即使 order_items.product_id 允许 NULL，库存释放也必须使用仍可解析的实际扣减商品快照。V1.0 不提供物理删除这些业务对象的入口。

完整接口及规范化约定见 [API 设计](api-design.md)，并发/超时规则见 [库存设计](inventory-design.md)，验收见 [测试计划](testing-plan.md)。


---

## 附录：V1.0 取餐编号与接单配置

- `orders` 增加 `UNIQUE(pickup_date,pickup_sequence)`；pending_payment/cancelled 两字段同为 NULL，paid/making/ready/completed 都必须非 NULL 且序号 > 0；`pickup_no` 由服务端展示为 A001，不单独保存。
- `pickup_daily_counters(pickup_date DATE PRIMARY KEY, last_sequence INTEGER NOT NULL CHECK(last_sequence>0))`；首次支付成功在订单/payment 事务内用 `INSERT ... ON CONFLICT DO UPDATE ... RETURNING` 原子取号，事务回滚不占号。
- `store_settings(id SMALLINT PRIMARY KEY CHECK(id=1), ordering_enabled BOOLEAN NOT NULL DEFAULT TRUE, notice VARCHAR(200) NOT NULL DEFAULT '', updated_by_admin_id BIGINT NULL REFERENCES admin_users(id), updated_at TIMESTAMPTZ NOT NULL)`；迁移须插入 id=1。
- 新单建立幂等键后 `SELECT ... FROM store_settings WHERE id=1 FOR SHARE`，共享锁覆盖接单检查、完整目录读取、库存扣减直到提交；暂停关闭返回 40906 并回滚。接单状态 UPDATE 及所有目录配置写入的 FOR UPDATE 均与此共享锁互斥。目录编辑只取锁，不改变该行的业务内容或审计字段。详见 [V1.0 业务与运行边界规则](v1-operation-rules.md)。
