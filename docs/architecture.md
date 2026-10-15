# 微信小程序点餐系统 - 系统架构

## 1. 总体架构

V1.0 采用前后端分离：顾客端和管理端分别独立开发、构建，通过 HTTP JSON API 调用后端。后端采用轻量级单体架构。

```text
customer-app：uni-app + Vue 3 + TypeScript
     │ 编译为微信小程序（V1.0 发布目标）
     │ HTTPS
     ↓
   FastAPI JSON API /api/v1/
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

管理端是独立 Web SPA：

```text
浏览器
  ↓
admin-web：Vue 3 + TypeScript + Vite + Element Plus
  │ 页面路由 /admin/，HTTP JSON 请求
  ↓
FastAPI /api/v1/admin/
  ↓
PostgreSQL
```

两套前端与后端使用同一仓库下的独立工程，后端只负责 JSON 接口和业务处理；管理页面由浏览器端 Vue 应用渲染。

uni-app 支持编译到微信、支付宝、抖音等小程序，以及 H5 和 App；本项目 V1.0 仅交付和验收微信小程序受控体验版，不公开营业、不实际收款或交付商品。其他平台接入需另行完成登录、支付、平台配置和真机验证，不能把编译成功当成业务已兼容。框架能力见 [uni-app 官方说明](https://uniapp.dcloud.net.cn/quickstart-cli.html)。

另运行同一代码库中的轻量订单回收 worker，复用订单关闭 service 并访问同一个 PostgreSQL。它只负责启动补扫和定时回收，不是独立业务服务；V1.0 不为此引入消息队列、Redis 或 Celery。

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

- Vue 3
- TypeScript
- Vite
- Vue Router（管理页面路由）
- Pinia（登录态和页面状态）
- Element Plus（Web 管理界面组件）
- 浏览器 fetch 封装的 JSON 请求层

管理端采用浏览器组件体系，独立于顾客端的 uni-app 组件。参见 [Element Plus 官方说明](https://element-plus.org/en-US/)。

### 顾客端

- uni-app 的 Vue 3 + TypeScript CLI 工程，使用官方 `vite-ts` 模板
- Vue 单文件组件、uni-app 内置组件和页面配置
- `uni.request` 封装网络层，`uni.login` 封装微信登录
- uni-app 存储和页面/应用生命周期适配层
- V1.0 构建目标 `mp-weixin`，生成微信小程序产物交给微信开发者工具

WXML/WXSS 属于微信编译产物，业务源码以 Vue 组件和样式维护。顾客端不能依赖浏览器 DOM、Web 管理端组件或直接散落的微信平台调用；必须通过客户端适配层访问平台能力。CLI 模板及构建目标见 [uni-app 命令行指南](https://uniapp.dcloud.net.cn/quickstart-cli.html)。

### 数据库

- PostgreSQL

### 部署

- Docker
- Nginx
- HTTPS
- 前端静态产物与后端分别构建、部署

前端构建使用 Node.js 22.12 或满足所选 Vite/uni-app 模板要求的后续兼容版本；两前端分别保存依赖锁文件，记录可复现的版本，不使用未锁定的 `latest` 作为部署约定。参见 [Vite 环境要求](https://vite.dev/guide/)。

---

## 3. 工程组织与后端分层

以下是待开发的同仓工程结构，不表示当前已经创建这些工程：

```text
egg-tart-ordering/
├── backend/
│   ├── app/
│   │   ├── api/
│   │   ├── core/
│   │   ├── models/
│   │   ├── schemas/
│   │   ├── services/
│   │   ├── repositories/
│   │   ├── workers/
│   │   └── main.py
│   ├── requirements.txt
│   └── .env.example
├── admin-web/          # 独立 Web 前端及 package.json、锁文件
├── customer-app/       # uni-app 前端及 package.json、锁文件
└── docs/
```

后端 Python 依赖、数据库迁移和配置放在 `backend/`；两个前端各自管理源码、依赖、公开 API 地址配置和构建脚本。前端不读取数据库、不携带后端密钥。

职责：

- `api`：HTTP 路由
- `schemas`：请求/响应数据结构
- `services`：业务逻辑
- `repositories`：数据库访问
- `models`：SQLAlchemy 模型
- `core`：配置、认证、数据库等基础能力
- `workers`：待支付订单超时扫描入口，调用与 API 相同的关闭 service
- `main.py`：应用入口

---

## 4. 认证架构

```text
uni.login({ provider: 'weixin' })
↓ 微信端登录适配
微信登录能力
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

微信端通过 uni-app 登录适配层取得 code 后调用现有顾客登录 API；后端仍验证 code 并换取微信 openid，不能直接信任前端身份。网络层通过 `uni.request` 附加 Bearer Token；存储和前后台生命周期封装后继续复用下单请求标识、恢复原订单以及同步倒计时。其他平台不能拿非微信 code 调用此接口。参见 [uni-app 网络请求](https://uniapp.dcloud.net.cn/api/request/request.html)。

V1.0 登录取得可信 openid 后，还必须匹配后端 `MOCK_ALLOWED_OPENIDS` 测试白名单才签发顾客 Token；已有 Token 每次访问受保护顾客接口时同样重新核验。空白名单或合法非白名单身份返回 HTTP 403、业务码 `40301`。公开商品与摊位状态查询仍可浏览，不能据此创建订单或模拟支付。白名单不是客户端提交字段，也不替代 code 校验和订单归属验证。

Token 失效时，小程序重新登录。

V1.0 可使用 JWT 或安全随机 Token 实现，具体方案在 Phase 4 决定。

### 4.1 管理员认证

管理前端调用 `POST /api/v1/admin/auth/login`，以 JSON 提交用户名和密码。后端验证管理员密码哈希和启用状态后签发独立管理员 JWT，返回 `token`、`token_type=Bearer`、`expires_in` 及管理员基本信息。

- 默认 `ADMIN_TOKEN_TTL_SECONDS=28800`（8 小时），JWT 含 `sub`、`actor=admin`、`aud=admin`、`exp`，签名密钥保存在后端。
- 管理接口验证签名、有效期、身份类型、audience 及管理员启用状态；不能仅凭顾客 Token 中相同的数字 ID 推断管理员身份。
- 管理前端将 Token 存于 `sessionStorage`，请求层附加 `Authorization: Bearer <token>`；401 时清除认证状态并跳转 SPA `/admin/login`，403 展示无权限提示。
- 退出管理端清理本地 Token 和登录态，此操作不宣称使已签发 JWT 在服务端失效。未知结果的库存调整请求按管理员隔离保存，重新登录同一管理员后仍可恢复原请求。
- 所有认证错误均返回 JSON；API 不返回登录页 HTML，也不重定向。顾客和管理员认证分别处理，不增加复杂 RBAC 或会话表。

---

## 5. 订单/库存架构

### 5.1 创建与幂等

订单创建必须是事务，PostgreSQL 使用 `READ COMMITTED`。客户端提交 `client_request_id`，服务端规范化商品、数量、套餐配方及备注后计算 `request_hash`。先验证身份和请求结构，再查询 `(user_id, client_request_id)`：命中同内容返回原订单当前状态，命中不同内容返回冲突；命中结果不重新读取当前商品价格或校验库存。

```text
BEGIN
 ↓
插入订单行，占用用户 + client_request_id 唯一键，固化创建及截止时间
 ↓
锁定 store_settings(id=1) FOR SHARE 并校验接单开关
 ↓
校验商品
 ↓
校验套餐
 ↓
读取数据库价格
 ↓
汇总实际普通商品需求，按商品 ID 升序原子扣减库存
 ↓
检查 UPDATE 影响行数
 ↓
填入订单最终金额，保留初始 expires_at
 ↓
创建订单明细
 ↓
创建支付记录
 ↓
COMMIT
```

唯一键必须在库存操作及依赖当前商品状态的业务校验前建立。首次插入时取一次 `t=clock_timestamp()`，写入 `created_at=t`、`expires_at=t+PAYMENT_TIMEOUT_SECONDS`；后续处理或重放不延长该期限。事务内订单行允许暂存金额 0，提交前必须写入最终金额、订单明细及支付记录；业务失败全部回滚，不保留占位订单。并发同键插入冲突时，回滚当前事务后读取已提交的胜者并比较指纹。不同请求 ID 的订单仍正常竞争库存，不能将所有数据库冲突都当成幂等命中。

控制行共享锁在任何当前商品/分类/套餐配置读取之前取得，持续到提交；配置读取规则见第 5.5 节。控制行缺失时拒绝新单；接单已暂停时回滚并返回 `40906`。已成功创建订单的同键重放在进入控制行和当前目录校验前返回原结果，不因当前停售或暂停而另建订单。

规范化、响应码和示例见 [API 设计](api-design.md)，唯一约束和字段见 [数据库设计](database-design.md)。首次成功返回 `201`，原单重放返回 `200`，内容冲突返回 `409`。重放不更新金额或截止时间。

库存原子更新：

```sql
UPDATE products
SET stock = stock - :quantity
WHERE id = :id
  AND product_type = 'single'
  AND is_active = TRUE
  AND stock >= :quantity;
```

`:quantity` 是服务端汇总后的正整数实际需求，影响 0 行表示未完成扣减。需区分商品已停用/无效与有效商品库存不足，只有后者返回 `40901`；不得提交只扣减了一部分商品的订单。

套餐份数乘以每份配方得到实际需求；所有单品和套餐按普通商品 ID 汇总。每个订单明细保存 `stock_deductions`，取消时同样汇总这些快照再恢复，不能读取当前套餐配置重算。

### 5.2 状态事务与锁顺序

支付、手动取消和超时关闭统一采用“订单 → 支付记录 → 所需商品按 ID 升序”的锁顺序。取得订单及支付锁后重查状态，使用 `clock_timestamp()` 取得实际当前时间判断是否到期；不能用固定在事务开始时刻的 `now()` 判断等待锁后的期限。参见 [PostgreSQL 时间函数](https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-CURRENT)。

首次创建订单的唯一键占用发生在库存操作前，支付记录随订单事务创建，提交前其他请求无法操作该未提交订单。已有订单的状态操作统一由 service 层管理，repository 不单独提交事务。

订单关闭必须在同一事务中完成以下变化：

```text
pending_payment + pending/failed
  ↓ 锁后确认未被支付/关闭，读取库存扣减快照
恢复一次库存 + order.cancelled + payment.closed + cancel_reason
  ↓
COMMIT
```

未到期主动取消的原因为 `manual`，到期关闭为 `timeout`，重复取消保留原结果。已支付订单不可取消；已取消订单不可支付。订单与支付组合、重复调用语义以 [订单状态机](order-state-machine.md) 为准。

service 将“需提交的业务结果”和“需要回滚的异常”区分开：到期支付必须先提交关闭及库存释放，再由路由返回 `40904`；Mock 确认失败须先提交 `payment.failed`，再返回 `40905`。不能在事务上下文内直接抛出导致回滚的 HTTP 异常。数据库故障仍回滚并返回服务端错误，不能伪装成确认失败。

### 5.3 超时回收 worker

- `PAYMENT_TIMEOUT_SECONDS=900`：创建订单时固化截止时间；修改配置只影响新订单。
- `ORDER_EXPIRY_SCAN_INTERVAL_SECONDS=30`：扫描间隔，不是支付宽限期；两个配置均校验为正整数。
- worker 启动立即补扫，此后按间隔查询已到期的待支付订单，按截止时间及 ID 顺序分批处理，每单一个事务。
- 每个候选仍需进入统一关闭 service，锁后重新判断状态和时间；扫描结果不能直接作为更新依据。单笔异常记录后继续其他订单，下一轮重试。
- 多实例重复扫描由行锁和状态判断保证安全。由部署进程管理器保持 worker 运行，不在每个 Uvicorn worker 中各自启动定时器。
- worker 停机期间支付入口仍拒绝过期支付并完成回收；其他遗留占用由重启补扫恢复。记录扫描、关闭数量及失败，供排查未回收订单。

### 5.4 管理库存增减

`products.stock` 仍是唯一的可售库存余额；普通商品初始为 0，补货和修正统一走 `delta` 调整。新增 `stock_adjustments` 记录调整事实，不引入独立库存余额表或预占表。

请求包含 `adjustment_request_id`、非零整数 `delta` 和原因，操作者由认证上下文提供。先查询幂等记录；未命中则锁定商品，并在锁后重新查询幂等记录，再校验和执行增减，防止等待锁期间同一调整已完成。调整后不得出现负库存。

调整记录和库存增减同事务提交，`UNIQUE(admin_user_id, adjustment_request_id)` 处理并发重复请求；唯一冲突须回滚本次增减后读取原记录。同键同内容返回原调整结果，异内容返回 `40902`。普通商品编辑接口不能覆盖已有 `stock`，套餐不能单独调库存。

### 5.5 商品目录保存与配置一致性

目录包括分类、商品价格/所属分类/启停状态、套餐每份数量及允许选择的普通商品集合。`READ COMMITTED` 下同一事务多次普通查询可能看到不同已提交版本，不能只依靠“读取和写入都在事务内”保证完整配置。参见 [PostgreSQL 隔离级别说明](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-READ-COMMITTED)。

V1.0 复用 `store_settings(id=1)` 作为营业开关和目录读写协调的控制行，采用以下共同协议，不新增配置版本表：

| 操作 | 控制锁与处理顺序 |
|---|---|
| 首次新下单 | 订单幂等唯一键 → 控制行 `FOR SHARE` → 接单状态检查 → 完整目录读取/合法性校验/计价 → 商品 ID 升序原子扣库存 → 快照、支付记录 → 提交 |
| 目录创建、编辑、上下架及分类启停 | 控制行直接 `FOR UPDATE` → 必要的既有商品按 ID 升序加锁 → 校验并完整保存目录定义与套餐成员关系 → 提交 |
| 暂停/恢复接单 | 更新同一控制行，与新单共享锁互斥，沿用既有接单语义 |
| 库存 delta、旧单支付/取号、取消/超时恢复、制作/取餐 | 不获取目录控制锁，沿用各自的订单/支付/商品锁顺序 |

新单先拿到共享锁，则按完整旧配置提交后才允许目录编辑；编辑方先取得独占锁，则提交后新单读取完整新配置。先点击按钮或事务先开始均不决定版本。共享锁允许订单之间并行，只有目录编辑短暂阻挡新单；原子库存更新仍裁决实际库存竞争。

控制行应由初始化流程建立。新单或目录写事务发现控制行缺失、未能取得该行时，必须整体回滚并返回通用 HTTP `500` 运行配置异常，不能无锁继续。环境/支付模式不允许模拟操作使用的 `50301` 不用于此异常。

该保护来自**所有目录写入口遵守同一协议**，不是控制行自动锁住其他表。禁止绕过 service 修改在线配置、先锁商品再获取控制锁，或先共享锁再升级；目录编辑也不得反向锁已有订单。仅取锁不修改 `ordering_enabled`、notice、营业变更操作者或营业更新时间。锁内只进行短数据库操作，不等待网络、人工确认或前端继续交互。初始化/迁移在无在线订单的维护流程执行。共享锁和独占锁的冲突规则见 [PostgreSQL 行锁说明](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-ROWS)。

管理端分类写接口和商品完整定义接口由 [API 设计](api-design.md) 明确。套餐元数据与 `combo_products` 关联在同一个事务完整保存，任一失败全部回滚；商品创建后类型不可变。普通商品必须提交 `combo_quantity=null`、`allowed_product_ids=[]`，不能配置套餐数量或成员；商品编辑不能覆盖库存。

顾客可售性统一要求商品与所属分类均启用。分类停用不改写子商品自身启用标记，顾客查询过滤、套餐选项及新单校验采用同一规则。套餐配置可保留存在但停用的普通商品关联，停用成员不能被新单选中；已创建订单继续使用原快照支付、制作或恢复，不重新按当前目录检查。

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

仅使用一个订单对应一条支付记录：`pending/failed → success`，失败仍保留原订单库存并允许期限内重试。取消或超时将未成功的记录置为 `closed`。合法组合为待支付与 `pending/failed`、已支付/制作中/待取餐/已完成与 `success`、已取消与 `closed`。`refunded` 仅为未来预留，V1.0 不产生该状态。

Mock 默认返回成功，失败由服务端测试配置控制，不接受客户端指定结果。成功重复调用返回当前成功结果，不改首次支付时间或流水；客户端结果未知时先查询订单。到期判断先于新的 Mock 尝试，已成功的支付重放不因截止时间已过而被取消。

V1.0 所有支付记录均为 `payment_method=mock`；`success` 只表示模拟确认成功，不是收款证据。取餐号、制作、ready 和 completed 仍可在隔离环境完成全链路验收，但均是测试流程，不能进入真实制作与实物交付队列。前端全局及订单、支付、取餐号、管理队列必须显示“模拟测试／未实际收款／不制作交付”，不能只依靠发布说明告知。

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
- 超时回收扫描及失败日志
- 请求幂等重放与内容冲突日志

关键业务应包含 `order_no` 等关联标识，方便排查问题。

库存调整的持久记录以同事务写入的 `stock_adjustments` 为准，应用日志仅用于诊断。日志中关联请求标识、操作者和业务结果，不记录 Token、密码或密钥。错误码映射详见 [API 设计](api-design.md)。

---

## 9. 运行环境与发布边界

开发环境：

- macOS
- Miniconda
- Python 3.12
- 本地 PostgreSQL/Docker

部署基础设施：

- Linux
- Docker
- PostgreSQL
- Nginx
- HTTPS

敏感配置通过后端 `.env` 注入。管理员 Token 有效期为后端配置；前端构建配置只包含 API 地址等公开值，不能包含数据库密码、JWT 密钥、微信 AppSecret 或真实支付密钥。

V1.0 只允许运行于 `development`、`test`、`staging`，使用独立测试数据库、测试库存和测试账号。Linux、Docker、Nginx、HTTPS 可以用于 staging，不能据部署方式认定已获准正式营业。

后端显式配置 `APP_ENV` 与 `PAYMENT_MODE`。缺失/未知值应启动失败；V1.0 仅实现 `PAYMENT_MODE=mock`，`APP_ENV=production` 配合 Mock 必须拒绝启动，改成未实现的 `wechat` 同样不能启动。API 与 worker 使用相同环境、支付模式及数据库，启动校验都不得被跳过。运行中意外检测到不允许模拟操作的配置时，业务入口拒绝执行并返回 HTTP 503、业务码 `50301`，不产生订单或支付副作用。

`MOCK_ALLOWED_OPENIDS` 为仅后端维护的逗号分隔测试名单，读取时去除各项两端空白、去重后逐值精确匹配；空集合不签发顾客 Token，也不允许受保护顾客调用，但有效测试环境中的管理员及 worker 仍可运行。实际微信 code 必须先经服务端换取 openid，不能传一个名单中的 openid 绕过认证。已有 Token 不豁免名单重查，撤出名单后拒绝其受保护调用。

V1.0 发布止于上传和受控体验验收，不包含公众接单或实物营业。Mock 失败注入只决定模拟成功/失败，关闭该配置、恢复接单或增加取餐号均不能开放正式营业。未来营业必须独立评审真实收款确认、支付回调/幂等/退款、测试数据与真实履约隔离以及端到端验收；V1.0 不实现该解锁流程。完整边界见 [V1.0 业务与运行边界规则](v1-operation-rules.md)。

### 9.1 构建与访问路径

- 管理端设置 Vite `base=/admin/`，Vue Router 使用相同基础路径；独立构建产生 `admin-web/dist/` 静态文件。
- 顾客端通过 `npm run build:mp-weixin` 构建，产物为 `customer-app/dist/build/mp-weixin/`；在微信开发者工具预览并上传受控体验版，配置小程序 AppID 与合法 HTTPS 请求域名。
- 后端独立启动 FastAPI 与超时回收 worker，前端构建不依赖服务端模板渲染。
- V1.0 staging 由同域 Nginx 提供 `/admin/` 静态页面，并将 `/api/v1/` 转发后端；SPA 深链接刷新回退到管理端 index，API 未命中必须返回 JSON 404，不能落入 SPA HTML 回退。未来营业部署可复用此路由结构，但须先通过独立营业准入。
- 前后端分离允许同域部署；将静态前端和 API 部署到不同域时，才需要额外设置浏览器跨域规则。

### 9.2 开发代理与跨域

管理端 Vite 开发服务器将 `/api/v1/` 代理至本地 FastAPI（默认 `http://127.0.0.1:8000`），保留路径前缀，前端统一使用请求层配置的 API 基础地址。

确需浏览器直接跨域访问时，由后端配置精确 Origin 白名单，允许实际使用的 GET、POST、PUT、OPTIONS 及 Authorization、Content-Type 请求头；Bearer 方案不依赖 Cookie，不开启携带 Cookie 的跨域凭证，也不使用任意来源通配符替代白名单。开发、测试和生产分别配置。详见 [FastAPI CORS 说明](https://fastapi.tiangolo.com/tutorial/cors/)。

微信小程序使用配置好的绝对 HTTPS API 地址并遵守平台域名规则，不经过管理端 Vite 代理；小程序合法请求域名与浏览器 CORS 是两类配置。未来增加 H5 目标时再核验 H5 Origin、登录方案及各浏览器行为。

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


---

## 11. V1.0 摊位运营扩展

后端单体增加 `pickup_daily_counters`（模拟支付成功的原子取餐编号）和 `store_settings`（接单开关及目录配置读写协调）两张辅助表，无须额外微服务。管理端按 `paid → making → ready → completed` 验证模拟制作与交付状态；新单按幂等键→控制行共享锁→完整配置读取→商品库存执行，目录写操作采用同一控制行独占锁。共 11 张表，详见 [V1.0 业务与运行边界规则](v1-operation-rules.md)。
