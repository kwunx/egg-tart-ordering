# 微信小程序点餐系统 - 开发阶段计划

## Phase 0：需求与设计冻结

- [x] 产品定位
- [x] V1.0 范围
- [x] 技术架构
- [x] 数据库初稿
- [x] API 初稿
- [x] Figma 页面规划
- [ ] 最终冻结订单/库存/支付规则
- [ ] 最终冻结状态机

## Phase 1：项目初始化

- 创建项目目录
- `git init`
- 使用 Miniconda
- 创建环境：

```bash
conda create -n egg-tart-ordering python=3.12
conda activate egg-tart-ordering
```

- 创建根目录 `requirements.txt`
- 创建 `.env.example`
- 创建 `.gitignore`
- 创建 FastAPI Hello World
- 启动 Swagger `/docs`
- 配置 VS Code 使用 `egg-tart-ordering`

## Phase 2：PostgreSQL/数据库

- 安装/启动 PostgreSQL
- 配置 SQLAlchemy
- 配置 Alembic
- 创建 8 张核心表
- 创建索引和约束
- 执行迁移
- 编写基础数据

## Phase 3：FastAPI 架构

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

- wx.login
- code 换 openid
- 创建/查询 users
- Token
- Token 校验
- 401 处理

## Phase 5：商品/分类/套餐

- 分类 API
- 商品 API
- 商品状态
- 套餐 API
- 套餐数量校验

## Phase 6：订单/库存

重点阶段。

- 创建订单
- 金额重新计算
- 商品有效性校验
- 套餐选择校验
- 原子库存更新
- 事务
- 订单快照
- 订单状态机
- 取消订单
- 库存释放
- 并发测试

核心测试：

```text
库存=1
A/B同时下单
→ 只能一个成功
```

## Phase 7：Mock 支付

- payment 表
- Mock 支付 API
- 支付状态
- 订单 paid
- 重复支付幂等
- 支付失败

## Phase 8：管理端

- 管理员登录
- Jinja2
- HTMX
- Tailwind CSS
- 商品管理
- 分类管理
- 套餐管理
- 库存管理
- 订单管理
- 状态流转
- Dashboard

## Phase 9：微信小程序

- 创建原生小程序
- 登录
- 商品
- 商品详情
- 套餐
- 购物车
- 确认订单
- Mock 支付
- 订单

## Phase 10：前后端集成

- API 对接
- Token
- 异常提示
- 库存不足
- 状态刷新
- 网络异常

## Phase 11：测试

### 功能测试

正常点餐流程。

### 业务测试

- 价格篡改
- 商品下架
- 库存不足
- 套餐数量错误
- 非法状态跳转
- 取消订单

### 并发测试

最后库存并发购买。

### 安全测试

- Token
- 用户订单隔离
- 管理员接口保护
- 参数校验

### 数据一致性

模拟事务中途失败，确认回滚。

## Phase 12：部署

- Docker
- PostgreSQL
- FastAPI
- Nginx
- HTTPS
- 环境变量
- 日志
- 数据备份

## Phase 13：微信发布

- 小程序 AppID
- 合法域名
- HTTPS
- 上传
- 体验版
- 审核
- 发布

## Phase 14：运行准备

- 商品初始化
- 库存初始化
- 管理员初始化
- 日志检查
- 数据备份
- 异常处理流程
- 基础运营检查
