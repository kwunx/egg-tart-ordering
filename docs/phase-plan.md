# 微信小程序点餐系统 - 开发阶段计划

## Phase 0：需求与设计冻结

- [x] 产品定位
- [x] V1.0 范围
- [x] 技术架构：FastAPI 后端、独立 Vue 3 管理端、uni-app + Vue 3 顾客端
- [x] 发布边界：顾客端 V1.0 仅构建、发布和验收微信 `mp-weixin`；其他平台后续适配
- [x] 数据库初稿
- [x] API 初稿
- [x] Figma 页面规划
- [x] 下单幂等：持久化 `client_request_id`，同用户同标识同内容恢复原单，内容冲突拒绝
- [x] 套餐数量与快照：每份配方 × 份数，全单汇总普通商品消耗，保存真实扣减快照
- [x] 状态互斥：支付、取消、超时统一锁顺序，关闭与库存恢复同事务
- [x] 支付超时：默认 15 分钟，独立 worker 每 30 秒扫描并启动补扫
- [x] Mock 支付与状态机：失败原单可重试，重复成功幂等，取消后支付关闭
- [x] 库存调整：非零增减量、必填原因、独立幂等标识及调整记录

以上勾选仅表示技术选型、发布边界和六项业务设计规则已确定，不代表相关软件已实现或测试通过。本次只更新文档，以下目录和构建任务是实施计划。后续阶段列出实施与验收任务，前后端工程及六项功能尚待实现和验证；仓库已完成的 Git 初始化不因此变为待办。完整验收场景见 `testing-plan.md`。

## Phase 1：项目初始化

- 保留已初始化的 Git 仓库，在同仓规划 `backend/`、`admin-web/`、`customer-app/` 三个独立工程
- 后端使用 Python 3.12 + FastAPI；原 `app/` 和 `workers/` 规划放入 `backend/`
- 后端使用 Miniconda 创建环境：

```bash
conda create -n egg-tart-ordering python=3.12
conda activate egg-tart-ordering
```

- 创建 `backend/requirements.txt` 和 `backend/.env.example`
- 声明 `PAYMENT_TIMEOUT_SECONDS=900`、`ORDER_EXPIRY_SCAN_INTERVAL_SECONDS=30`、`ADMIN_TOKEN_TTL_SECONDS=28800`；Mock 失败注入仅允许服务端测试配置启用
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
- 创建 8 张核心表及 1 张 `stock_adjustments` 库存调整记录表
- 增加下单幂等标识/内容校验数据、到期时间、取消原因和订单明细 `stock_deductions` 真实消耗快照
- 创建索引和约束，含下单用户/请求标识唯一、`(admin_user_id, adjustment_request_id)` 唯一，以及支付与库存状态约束
- 执行迁移
- 编写基础数据；普通商品初始库存为 `0`，后续通过库存调整入口录入补货

## Phase 3：FastAPI 架构

以下分层位于 `backend/app/`，超时 worker 位于 `backend/workers/`。FastAPI 提供 JSON API，两个前端独立构建。

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

## Phase 5：商品/分类/套餐

- 分类 API
- 商品 API
- 商品状态
- 套餐 API
- 套餐数量校验：`quantity` 为份数，`selections` 为每份配方，子商品 ID 不得重复
- 同商品同配方合并、不同配方分行的统一规则

## Phase 6：订单/库存

重点阶段。

- 创建订单，前端 UUID + 服务端锁/唯一约束实现一单一扣
- 同键同内容重放返回 HTTP `200` 和原单当前状态，首次创建 HTTP `201`，同键异内容返回 `40902`；重放不重新校验当前商品价格/状态
- 金额重新计算
- 商品有效性校验
- 套餐选择校验
- 全单按普通商品汇总消耗，按商品 ID 升序执行原子库存更新
- 事务
- 商品/价格/每份配方快照和每条明细的 `stock_deductions` 快照
- 冻结的订单/支付合法状态组合及 `cancel_reason`
- 取消订单，支付/取消/超时统一锁订单 → 支付 → ID 升序商品；锁后使用 `clock_timestamp()` 判断到期
- 按真实扣减快照恢复库存，与 `cancelled + closed` 在同一事务提交，重复请求不重复释放
- 创建时保存 `expires_at`；独立轻量 worker 逐单事务扫描，启动补扫，支持多实例竞争
- 到期支付先提交超时取消/库存恢复，再返回 `40904`；到期后手动取消原因也为 `timeout`
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
- 商品管理
- 分类管理
- 套餐管理
- 库存增减表单：仅普通商品、非零 `delta`、必填 `reason`、独立持久化 `adjustment_request_id`
- 库存调整与 `stock_adjustments` 同事务，重试恢复原结果，禁止负库存；商品编辑不能覆盖已有库存
- 订单管理
- 状态流转
- Dashboard

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
- 不将浏览器 DOM 或 Element Plus 组件用于小程序；H5、App、支付宝、抖音的登录、支付和发布不计入 V1.0

## Phase 10：前后端集成

- API 对接
- Token
- 异常提示
- 库存不足
- 状态刷新
- 网络异常：区分业务失败与结果未知，先恢复原单或原调整结果
- 对齐 `201/200/409` 及整数业务码 `40901` 至 `40905`
- 管理端 Vite 开发代理 `/api/v1` → `http://127.0.0.1:8000`，不改写 API 前缀；微信开发工具使用顾客端配置的 API 绝对地址及合法 HTTPS 域名
- 校验 `/admin/` 页面路由与 `/api/v1/admin/*` JSON API 分离，SPA 回退不能吞掉 API 的 401/403/404 响应
- 默认生产同域反代；仅实际跨域部署时启用准确 origin 白名单，跨域不是前后端分离的前提

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

依照 `testing-plan.md` 执行六组专项验收：下单幂等与断网恢复、套餐份数及历史快照、支付/取消/超时互斥、worker 补扫与跨时段运行、Mock 失败与原单重试、库存调整幂等与审计。使用真实 PostgreSQL 独立事务验证并发与锁，不仅测试单条正常请求。

### 安全测试

- Token
- 用户订单隔离
- 管理员接口保护
- 参数校验

### 数据一致性

模拟事务中途失败，确认回滚。

检查到期支付返回错误前已经提交关闭与库存恢复；检查订单创建后更改套餐配置仍按原扣减快照恢复。记录实际执行结果后才判定测试完成。

## Phase 12：部署

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
- 生产环境关闭 Mock 失败注入配置
- 核对管理员 JWT 默认有效期 `28800` 秒；密钥和数据库配置仅保留在后端部署环境
- 日志
- 数据备份
- 部署后验证过期订单可自动关闭、worker 重启可补扫、HTTP 服务与 worker 使用同一数据库及业务规则

## Phase 13：微信发布

- 将 `customer-app/` 构建为 `customer-app/dist/build/mp-weixin`，导入微信开发者工具；管理端 Web 产物独立部署
- 小程序 AppID
- 合法域名
- HTTPS
- 上传
- 体验版
- 审核
- 发布

本阶段只发布微信小程序。跨平台框架提供的其他构建目标不代表已完成对应平台的认证、支付、域名或发布审核，后续另行设计和验收。

## Phase 14：运行准备

- 商品初始化
- 普通商品库存初始化为 `0`，通过带原因和幂等标识的库存调整操作补货并核对调整记录
- 管理员初始化
- 日志检查
- 数据备份
- 异常处理流程
- 演练下单/支付/库存调整响应丢失后的原请求恢复，以及超时 worker 停机后的补扫
- 检查待支付到期时间、扫描处理日志和异常库存记录；确认未付款遗留订单可回收
- 基础运营检查
