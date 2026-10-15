# 微信小程序点餐系统 - 开发阶段计划

## Phase 0：需求与设计冻结

- [x] 产品定位
- [x] V1.0 范围
- [x] 技术架构：FastAPI 后端、独立 Vue 3 管理端、uni-app + Vue 3 顾客端
- [x] 测试边界设计：V1.0 仅构建、上传及体验版受控验证微信 `mp-weixin`，不公开营业或交付实物
- [x] 数据库初稿
- [x] API 初稿
- [x] Figma 页面规划
- [x] 下单幂等：持久化 `client_request_id`，同用户同标识同内容恢复原单，内容冲突拒绝
- [x] 套餐数量与快照：每份配方 × 份数，全单汇总普通商品消耗，保存真实扣减快照
- [x] 状态互斥：支付、取消、超时统一锁顺序，关闭与库存恢复同事务
- [x] 支付超时：默认 15 分钟，独立 worker 每 30 秒扫描并启动补扫
- [x] Mock 支付与状态机：失败原单可重试，重复成功幂等，取消后支付关闭
- [x] 库存调整：非零增减量、必填原因、独立幂等标识及调整记录
- [x] 取餐号、ready 待取餐、暂停接单纳入 V1.0 设计
- [x] 目录管理设计：分类完整定义、商品/套餐属性与成员原子保存，分类和商品均启用才可售
- [x] 配置一致性设计：新单共享配置锁、目录编辑排他配置锁，订单使用完整旧/新配置
- [x] Mock 隔离设计：显式环境与模式校验、测试身份白名单、模拟标识及独立测试数据

以上勾选仅表示设计决定已记录，不代表实现或测试通过。本次只更新文档；目录管理、配置锁、Mock 隔离以及原六项业务规则、取餐号/ready/暂停功能均须按后续任务实现并验证。仓库已完成的 Git 初始化不因此变为待办。具体待实施项目见文末清单，完整验收场景见 `testing-plan.md`。

## Phase 1：项目初始化

- 保留已初始化的 Git 仓库，在同仓规划 `backend/`、`admin-web/`、`customer-app/` 三个独立工程
- 后端使用 Python 3.12 + FastAPI；应用位于 `backend/app/`，独立超时 worker 的源码统一放在 `backend/app/workers/`
- 后端使用 Miniconda 创建环境：

```bash
conda create -n egg-tart-ordering python=3.12
conda activate egg-tart-ordering
```

- 创建 `backend/requirements.txt` 和 `backend/.env.example`
- 声明 `PAYMENT_TIMEOUT_SECONDS=900`、`ORDER_EXPIRY_SCAN_INTERVAL_SECONDS=30`、`ADMIN_TOKEN_TTL_SECONDS=28800`；Mock 失败注入仅允许服务端测试配置启用
- 明确 `APP_ENV`、`PAYMENT_MODE` 必填；环境仅 `development/test/staging/production`，V1.0 模式仅 `mock`。缺失、未知、未实现模式或 `production + mock` 拒绝启动，改填 `wechat` 不解除限制
- 示例使用显式 `APP_ENV=development`、`PAYMENT_MODE=mock`；`MOCK_ALLOWED_OPENIDS` 为后端逗号分隔、去首尾空白并去重的精确名单，空名单拒绝顾客测试，不阻止有效环境的管理端和 worker 启动
- 前端使用满足所选 Vite 版本要求的 Node.js 22.12+，分别维护 `package.json` 和锁文件，固定依赖版本，不依赖未固定的 latest
- 初始化 `admin-web/`：Vue 3 + TypeScript + Vite + Vue Router + Pinia + Element Plus
- 初始化 `customer-app/`：uni-app CLI `vite-ts` 模板，Vue 3 + TypeScript；V1.0 构建目标为 `mp-weixin`
- 分别配置管理端开发 API 代理、顾客端 API 绝对地址与微信 AppID；前端环境配置不得包含后端密钥
- 创建 `.gitignore`
- 创建 FastAPI Hello World
- 启动 Swagger `/docs`
- 配置 VS Code 使用 `egg-tart-ordering`

## Phase 2：PostgreSQL/数据库

- 安装/启动 PostgreSQL
- 配置 SQLAlchemy
- 配置 Alembic
- 创建 8 张核心表、stock_adjustments 与 pickup_daily_counters、store_settings 两张运营表（共 11 张）
- 增加下单幂等标识/内容校验数据、到期时间、取消原因和订单明细 `stock_deductions` 真实消耗快照
- 创建索引和约束，含下单用户/请求标识唯一、`(admin_user_id, adjustment_request_id)` 唯一，以及支付与库存状态约束
- 执行迁移
- 编写基础数据；普通商品初始库存为 `0`，后续通过库存调整入口录入补货
- 目录锁复用 `store_settings` 单例行，模拟订单使用 `payment_method=mock` 识别；不增加配置锁表或 `is_test` 列，维持 11 张表

## Phase 3：FastAPI 架构

以下分层位于 `backend/app/`，超时 worker 位于 `backend/app/workers/`。FastAPI 提供 JSON API，两个前端独立构建。

- api
- core
- models
- schemas
- services
- repositories
- 统一响应
- 异常处理
- 日志

## Phase 4：顾客认证

- 微信目标使用 `uni.login({ provider: 'weixin' })` 获取 code
- code 换 openid
- 创建/查询 users
- 决定顾客应用 Token 的具体方案；本次管理员 JWT 选型不替代该决策
- Token 校验
- 401 处理
- code 换得 openid 后按精确测试名单决定是否签发顾客 Token；已有 Token 每次访问受保护接口仍校验名单，非名单 HTTP `403`/`40301`；公开目录及接单状态不受测试名单限制

## Phase 5：商品/分类/套餐

- 分类 POST/PUT 完整定义 `name/sort_order/is_active`；商品与套餐复用商品 POST/PUT，含 `combo_quantity/allowed_product_ids` 的完整定义
- 套餐定义与成员在同一事务替换；single 显式 `combo_quantity=null`、`allowed_product_ids=[]`，禁止通过通用编辑修改类型或库存
- 管理配置允许引用停用但存在的 single；顾客可售规则要求相关分类和商品均启用，分类停用不改写子商品自身标记
- 所有目录创建、编辑、停用先锁 `store_settings FOR UPDATE`，再按商品 ID 升序获取所需商品锁，原子保存完整定义与成员；不得修改营业字段、不得先商品后配置锁或将共享锁升级
- 套餐数量校验：`quantity` 为份数，`selections` 为每份配方，子商品 ID 不得重复
- 同商品同配方合并、不同配方分行的统一规则

## Phase 6：订单/库存

重点阶段。

- 创建订单，前端 UUID + 服务端锁/唯一约束实现一单一扣
- 同键同内容重放返回 HTTP `200` 和原单当前状态，首次创建 HTTP `201`，同键异内容返回 `40902`；重放不重新校验当前商品价格/状态
- 金额重新计算
- 商品有效性校验
- 套餐选择校验
- 首次新单取得唯一请求键后，锁 `store_settings FOR SHARE` 并检查接单状态，再读取完整分类/商品/套餐配置、服务端价格及快照，保持该锁至库存及订单事务提交
- 新单与目录编辑按配置锁先后选择完整旧/新定义；同一配置上的多个新单可持共享锁并行，商品库存仍按 ID 升序更新
- 全单按普通商品汇总消耗，按商品 ID 升序执行原子库存更新
- 事务
- 商品/价格/每份配方快照和每条明细的 `stock_deductions` 快照
- 冻结的订单/支付合法状态组合及 `cancel_reason`
- 取消订单，支付/取消/超时统一锁订单 → 支付 → ID 升序商品；锁后使用 `clock_timestamp()` 判断到期
- 按真实扣减快照恢复库存，与 `cancelled + closed` 在同一事务提交，重复请求不重复释放
- 创建时保存 `expires_at`；独立轻量 worker 逐单事务扫描，启动补扫，支持多实例竞争
- 到期支付先提交超时取消/库存恢复，再返回 `40904`；到期后手动取消原因也为 `timeout`
- 支付取号、取消/超时、制作/取餐和库存 delta 调整不取得目录配置锁，维持既有锁顺序；目录变更不影响原订单支付或按历史快照恢复库存
- 并发测试

核心测试示例（待执行）：

```text
库存=1
A/B同时下单
→ 只能一个成功
```

## Phase 7：Mock 支付

- payment 表
- Mock 支付 API
- 全部 V1.0 订单为 `payment_method=mock`，独立测试数据验证完整模拟流程，不进行实物履约
- 启动校验运行环境与支付模式；运行期发现不允许的配置拒绝受保护操作，返回 HTTP `503`/`50301`
- 默认由后端返回成功；客户端不得指定结果，失败仅由服务端测试配置触发
- `pending_payment + pending/failed` 可在到期前支付，成功原子更新为 `paid + success`
- 失败后复用原订单和 payment 重试，不重复创建记录或扣库存
- 重复成功返回 HTTP `200`，保留首次流水和 `paid_at`
- 支付/取消/超时互斥和结果未知时查询原单的恢复流程
- 支付失败返回 `40905`，状态冲突返回 `40903`，过期返回 `40904`

## Phase 8：独立 Web 管理端

- `admin-web/` 使用 Vue 3 + TypeScript + Vite，Vue Router 基础路径为 `/admin/`，Pinia 管理状态，Element Plus 提供管理页面组件
- 管理员登录页面 `/admin/login`，调用 `POST /api/v1/admin/auth/login`，后端返回 JWT 和 `expires_in`，默认有效期 8 小时
- Token 仅保存在 `sessionStorage`；统一 `fetch` JSON 请求层为 `/api/v1/admin/*` 附带 Bearer Token；401/退出清除本地认证并跳转登录，403展示无权限
- 管理页面导航与 JSON API 路径分离，前端路由守卫与后端管理员鉴权配合；普通顾客 Token 不得访问管理接口
- 商品/套餐完整定义编辑表单、分类表单及启停操作，套餐属性与成员一次保存，失败保留输入并回显错误
- 库存增减表单：仅普通商品、非零 `delta`、必填 `reason`、独立持久化 `adjustment_request_id`
- 库存调整与 `stock_adjustments` 同事务，重试恢复原结果，禁止负库存；商品编辑不能覆盖已有库存
- 订单管理
- 状态流转
- Dashboard
- 管理页面、订单队列及取餐号明确标记“模拟测试、未实际收款、不制作交付”，制作/完成/取餐按钮采用模拟动作文案

## Phase 9：uni-app 顾客端（V1.0 微信小程序）

- 使用 `customer-app/` 的 uni-app CLI `vite-ts` 工程开发 Vue 3 + TypeScript 页面，采用 uni-app 组件/API，由编译器生成微信小程序产物
- 微信登录保留 code → 后端 → 微信 openid → 应用 Token 链路
- 统一 `uni.request` 请求层：API 绝对地址、Bearer Token、HTTP 与业务码处理、业务失败与未知结果区分
- 使用 uni-app 存储和生命周期 API，适配请求标识持久化、返回前台及网络恢复
- 商品
- 商品详情
- 套餐每份选择与购买份数
- 购物车同配方合并、不同配方分行
- 确认订单；发请求前按用户持久化下单 UUID 和完整内容，断网/重启复用恢复
- Mock 支付倒计时、继续支付、取消、失败原单重试及结果未知时查询
- 订单列表/详情展示原快照、到期时间、支付状态及取消原因
- 全局、支付页、订单卡、详情及取餐号显著显示测试标识；处理 `40301` 测试资格拒绝与 `50301` 环境拒绝，不循环登录或绕过守卫
- 不将浏览器 DOM 或 Element Plus 组件用于小程序；H5、App、支付宝、抖音的登录、支付和发布不计入 V1.0

## Phase 10：前后端集成

- API 对接
- Token
- 异常提示
- 库存不足
- 状态刷新
- 网络异常：区分业务失败与结果未知，先恢复原单或原调整结果
- 对齐目录创建 `201`、更新/停用 `200`、结构错误 `422`、完整定义校验 `400/40001`、目标不存在 `404/40401`，以及既有 `40901` 至 `40906`、测试资格 `40301` 和运行配置 `50301`
- 管理端 Vite 开发代理 `/api/v1` → `http://127.0.0.1:8000`，不改写 API 前缀；微信开发工具使用顾客端配置的 API 绝对地址及合法 HTTPS 域名
- 校验 `/admin/` 页面路由与 `/api/v1/admin/*` JSON API 分离，SPA 回退不能吞掉 API 的 401/403/404 响应
- 受控部署默认同域反代；仅实际跨域部署时启用准确 origin 白名单，跨域不是前后端分离的前提

## Phase 11：测试

### 功能测试

正常点餐流程。

执行管理端独立构建、SPA 深链接刷新、登录/401/403、API 代理路径验证，以及微信 `mp-weixin` 编译、真机登录、uni-app 存储和生命周期验证；具体场景见测试计划。其他目标的编译能力不等同于其他平台业务可用，不扩展 V1.0 发布验收。

### 业务测试

- 价格篡改
- 商品下架
- 库存不足
- 套餐数量错误
- 非法状态跳转
- 取消订单

### 并发测试

最后库存并发购买。

依照 `testing-plan.md` 执行原六组业务验收及取餐/ready/暂停验收，并补目录 CRUD、套餐原子保存、配置编辑与新单先后竞争、完整旧/新定义、两个新单共享锁并行和锁顺序检查。使用真实 PostgreSQL 独立事务验证并发与锁，不仅测试单条正常请求。

### 安全测试

- Token
- 用户订单隔离
- 管理员接口保护
- 参数校验
- Mock 环境/模式启动拒绝、运行期守卫、空白名单和既有 Token 被移出名单的拒绝；验证独立测试数据与模拟标识，不进入实物交付

### 数据一致性

模拟事务中途失败，确认回滚。

检查到期支付返回错误前已经提交关闭与库存恢复；检查订单创建后更改套餐配置仍按原扣减快照恢复。记录实际执行结果后才判定测试完成。

## Phase 12：受控测试环境部署

- Docker
- PostgreSQL
- FastAPI
- 从 `backend/` 构建和部署 Python API 与 worker，与两个前端的依赖和产物分离
- 独立构建管理端到 `admin-web/dist`；Vite base 与 Vue Router base 均为 `/admin/`
- 独立订单超时 worker 进程/容器，配置进程守护、启动补扫及失败日志；可多实例运行但逐单锁后重检
- Nginx 同域提供 `/admin/` 静态资源、SPA 深链接回退，并将 `/api/v1/` 反向代理到 FastAPI；API 路径优先匹配，不能回退为管理端 HTML
- HTTPS
- 环境变量
- 核对默认支付时限 `900` 秒、扫描间隔 `30` 秒；变更配置不回写已有订单的 `expires_at`
- 明确仅部署 `development/test/staging + mock`；V1.0 的 `production + mock` 及未实现模式不得启动，不用修改环境名绕过正式营业准入
- 受控测试环境的 Mock 失败模拟默认关闭，仅专项用例临时启用；关闭失败注入不等于真实收款或营业放行
- 隔离测试数据库、库存、管理员队列及接口地址；配置后端测试 openid 精确白名单，空名单仍可运行管理端与独立 worker 做准备及清理
- 核对管理员 JWT 默认有效期 `28800` 秒；密钥和数据库配置仅保留在后端部署环境
- 日志
- 数据备份
- 部署后验证过期订单可自动关闭、worker 重启可补扫、HTTP 服务与 worker 使用同一数据库及业务规则

## Phase 13：微信上传与体验版受控测试

- 将 `customer-app/` 构建为 `customer-app/dist/build/mp-weixin`，导入微信开发者工具；管理端 Web 产物独立部署
- 小程序 AppID
- 合法域名
- HTTPS
- 上传
- 配置授权测试人员和后端测试白名单
- 体验版验证完整模拟下单、制作、待取餐、完成流程，不制作交付实物
- 核对顾客端与管理端模拟标识，以及测试数据库、库存和队列隔离

本阶段止于微信体验版受控验收，不包含正式公开营业、审核发布或其他平台发布。跨平台框架具备其他编译目标也不代表其认证与支付已适配。

## Phase 14：受控测试运行准备

- 商品初始化
- 普通商品库存初始化为 `0`，通过带原因和幂等标识的库存调整操作补货并核对调整记录
- 管理员初始化
- 日志检查
- 数据备份
- 异常处理流程
- 演练下单/支付/库存调整响应丢失后的原请求恢复，以及超时 worker 停机后的补扫
- 检查待支付到期时间、扫描处理日志和异常库存记录；确认未付款遗留订单可回收
- 基础运营检查

## 未来正式营业准入（不属于本轮 V1.0）

- 单独设计、实现并验证可信真实收款及其订单/库存/履约一致性，隔离测试与真实营业数据，再进行完整营业验收。
- 完成上述前置工作后才另行决定公开营业部署与发布；当前实现只支持 Mock，修改 `PAYMENT_MODE=wechat`、恢复 `ordering_enabled` 或关闭失败注入都不能解除限制。
- 此为本项目的范围与准入约定，不是微信平台规则断言；当前可继续开发并通过体验版验证，无需现在提前接入真实支付。


---

## V1.0 新增实施任务（尚未完成）

- [ ] 数据库增加取餐编号字段与唯一约束、每日计数器表、营业配置单例表。
- [ ] 新单暂停准入检查、原订单重放恢复、暂停/恢复管理 API。
- [ ] 首次支付成功同事务生成当日取餐号。
- [ ] 管理订单 making→ready→completed 状态，顾客端取餐号及暂停接单提示。
- [ ] 并发与跨日编号、暂停竞争、旧单继续支付/取消的真实数据库测试。详见 [V1.0 业务与运行边界规则](v1-operation-rules.md)。
- [ ] 实现分类和商品/套餐完整定义 CRUD、原子成员替换及可售规则。
- [ ] 实现目录编辑排他锁与新单共享锁，完成并发与回滚验收。
- [ ] 实现环境/模式启动校验、运行守卫、openid 精确白名单及旧 Token 持续校验。
- [ ] 实现全端模拟标识和隔离测试部署，完成体验版验收，不解锁真实履约。
