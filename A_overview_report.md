# 电商后端API项目概览报告

## 1. 项目目标和边界

### 负责什么
- 基于FastAPI的电商后端API服务，提供基础的电商数据管理功能
- 商品（products）、分类（categories）、购物车（cart）、订单（orders）、支付（payments）、库存（inventory）、用户（users）七个资源的CRUD操作
- 商品Excel导出功能，支持批量数据导出
- 用户认证功能，基于用户名密码的简单验证
- 订单状态管理，支持状态字段的更新和查询

**证据**：`main.py:25-32` 路由注册、`routes/product.py:15-68` 资源端点定义

### 不负责什么
- **不负责**真实支付网关集成，仅记录支付信息无实际资金流转
- **不负责**库存扣减与订单创建的原子性，两个操作完全独立
- **不负责**用户权限管理，仅有认证无角色或权限控制
- **不负责**高并发场景，JSON文件存储无锁机制
- **不负责**数据备份与恢复，无自动备份策略
- **不负责**搜索、分页、过滤等高级查询功能

---

## 2. 分层方式

| 层级 | 职责 | 实现后果 | 证据 |
|------|------|----------|------|
| **main** | FastAPI实例配置、路由注册、元信息设置 | 无中间件、无全局异常处理、无CORS配置，是最简化的FastAPI启动方式 | 证据：`main.py:10-32` FastAPI初始化、`app.include_router`路由注册 |
| **routes** | HTTP接口定义、请求参数绑定、异常状态码转换 | 每个路由函数独立处理异常，无统一异常处理；HTTP状态码硬编码在各路由中 | 证据：`routes/product.py:15-23` `read_products`路由、`routes/product.py:26-33` `read_product`路由 |
| **services** | 业务逻辑、JSON文件读写、数据操作 | 每个service独立维护自己的数据文件，模块间无横向依赖；数据一致性完全依赖业务逻辑实现 | 证据：`services/product.py:13-26` `load_products`函数、`services/product.py:56-63` `create_product`函数 |
| **models** | Pydantic数据模型定义、类型约束 | 仅做字段类型和存在性校验，无业务规则校验（如价格>0、库存>=0等） | 证据：`models/product.py:1-9` Product类定义、`models/Order.py:1-9` Order类定义 |
| **data** | JSON文件持久化存储 | 每个资源一个独立的JSON文件，无关联约束、无外键、无事务支持 | 证据：`data/products.json`商品数据文件、`data/orders.json`订单数据文件 |

**关键设计后果**：
- 模块高度解耦：每个资源有独立的routes/services/models/data文件，修改一个不影响其他
- 无横向依赖：商品模块不知道订单模块存在（推断：基于代码结构，各service独立无相互导入）
- 无事务保障：跨模块操作（如下单→扣库存）无法保证原子性（推断：基于无数据库特性）
- 业务逻辑分散：相同类型的校验逻辑可能分散在不同路由中（推断：基于现有代码中未发现统一校验机制）

---

## 3. 关键执行路径

### 商品链路
```
HTTP请求 → routes/product.py → services/product.py → data/products.json
```
- **创建商品**：`POST /api/products/` → `add_product()` → `create_product()` → 写入JSON文件
  - 行为后果：客户端必须提供完整的商品ID和所有字段信息，服务端仅检查ID是否重复
  - 系统边界：不检查category_id是否真的存在于categories.json中
  
- **查询商品**：`GET /api/products/{product_id}` → `read_product()` → `get_product_by_id()` → 读取JSON文件
  - 行为后果：每次查询都要加载整个products.json到内存，商品数量越多响应越慢
  
- **导出商品**：`GET /api/products/export` → `export_products()` → `export_products_to_excel()` → 生成Excel文件
  - 行为后果：导出文件保存在服务端当前工作目录，客户端无法直接下载

**证据**：`routes/product.py:36-43` `add_product`路由、`services/product.py:56-63` `create_product`函数

---

### 订单链路
```
HTTP请求 → routes/order.py → services/order.py → data/orders.json
```
- **创建订单**：`POST /api/orders/` → `add_order()` → `create_order()` → 写入JSON文件
  - 行为后果1：服务端自动覆盖`created_at`字段，客户端传入的值无效
  - 行为后果2：不检查user_id是否存在，不检查订单商品是否有效
  - 行为后果3：完全不涉及库存模块，库存数量无变化
  
- **查询用户订单**：`GET /api/orders/user/{user_id}` → `read_user_orders()` → `get_user_orders()` → 内存过滤
  - 行为后果：加载所有订单到内存后按user_id过滤，订单数越多效率越低
  
- **更新订单状态**：`PUT /api/orders/{order_id}/status` → `modify_order_status()` → `update_order_status()` → 覆盖写入
  - 行为后果：status参数是query参数不是body参数；可任意设置状态值

**证据**：`routes/order.py:12-17` `add_order`路由、`services/order.py:38-44` `create_order`函数、`routes/order.py:31-38` `modify_order_status`路由

---

### 购物车链路
```
HTTP请求 → routes/cart.py → services/cart.py → data/cart.json
```
- **添加商品到购物车**：`POST /api/cart/` → `add_item_to_cart()` → `add_to_cart()` → 写入JSON文件
  - 行为后果1：若商品已存在则累加数量，否则新增记录
  - 行为后果2：不检查product_id是否在products.json中存在
  - 行为后果3：不检查库存数量是否足够
  
- **查询购物车**：`GET /api/cart/{user_id}` → `read_user_cart()` → `get_user_cart()` → 内存过滤
  - 行为后果：返回的是购物车条目列表，不含商品详情（名称、价格等）
  
- **删除购物车商品**：`DELETE /api/cart/{user_id}/{product_id}` → `remove_item_from_cart()` → `remove_from_cart()` → 覆盖写入
  - 行为后果：必须同时提供user_id和product_id才能删除

**证据**：`routes/cart.py:12-17` `add_item_to_cart`路由、`services/cart.py:37-50` `add_to_cart`函数

---

## 4. 数据存储与状态管理

### JSON文件方案的收益

1. **实现简单**：无数据库依赖，开发速度快
   - 证据：`services/product.py` `load_products`函数
   - 意味着：项目可快速启动，无需DBA参与，部署成本极低

2. **数据直观**：JSON格式人类可读
   - 证据：`data/products.json` 商品数据结构
   - 意味着：开发阶段可直接用文本编辑器修改数据，便于调试和演示

3. **无额外依赖**：Python标准库json模块支持
   - 证据：`requirements.txt` 依赖列表
   - 意味着：部署时只需安装FastAPI相关依赖，无数据库驱动、ORM等额外依赖

4. **易于备份**：单文件存储
   - 证据：`data/` 目录（推断：包含多个独立文件）
   - 意味着：备份只需复制整个data目录，迁移简单

### JSON文件方案的限制

1. **并发安全问题**：读写无锁，并发写入数据丢失
   - 证据：`services/product.py` `save_products`函数直接覆盖文件
   - 意味着：两个请求同时修改时，后写入的会完全覆盖先写入的内容，数据永久丢失

2. **查询性能低下**：O(n)全表扫描
   - 证据：`services/product.py` `get_product_by_id`函数用next()遍历
   - 意味着：1000个商品需要遍历1000次，10000个商品遍历10000次，响应时间线性增长

3. **内存占用随数据量线性增长**：每次加载全量数据
   - 证据：所有`load_*()`函数都返回完整列表（推断：基于代码审查）
   - 意味着：1万条商品数据可能占用几十MB内存，10万条可能占用几百MB

4. **无事务支持**：多步操作无法回滚
   - 证据：`services/order.py` `create_order`只操作orders.json
   - 意味着：创建订单成功但扣库存失败时（推断：假设存在扣库存是独立操作），订单已存在但库存未扣，数据不一致

5. **无关联约束**：引用完整性无法保证
   - 证据：`services/product.py` `delete_product`不检查外键引用
   - 意味着：删除商品后，购物车和订单中仍有该商品ID的引用，查询时可能显示不存在的商品

---

## 5. 新同事接手时最容易误判的地方

### 误判点1：库存与商品的关系
**容易误判**：认为商品的`stock`字段和`inventory`模块是关联同步的  
**实际情况**：两者完全独立，无任何同步机制
- 证据：`models/product.py` Product类有`stock`字段定义
- 证据：`services/product.py` 的CRUD操作完全不涉及`inventory.json`
- 证据：`services/inventory.py` 操作独立的`inventory.json`文件
- **行为后果**：修改商品stock字段，inventory表无变化；修改inventory表，商品stock也无变化

### 误判点2：创建订单时的库存扣减
**容易误判**：认为创建订单会自动检查并扣减库存  
**实际情况**：创建订单接口完全不涉及库存操作
- 证据：`services/order.py` `create_order`函数仅保存订单信息
- 证据：`routes/order.py` `add_order`路由无任何库存相关参数
- **行为后果**：库存为0或负数时也能成功下单，超卖问题严重

### 误判点3：删除操作的级联影响
**容易误判**：认为删除商品/用户时会级联检查关联数据  
**实际情况**：删除操作不检查任何关联记录
- 证据：`services/product.py` `delete_product`函数仅过滤ID
- 证据：`services/cart.py` `remove_from_cart`函数也不检查商品是否存在
- **行为后果**：删除商品后，购物车中该商品条目仍存在，查询时可能导致数据不一致

### 误判点4：订单状态的业务约束
**容易误判**：认为订单状态有状态机约束（如Pending→Paid→Shipped）
**实际情况**：状态是自由字符串，可任意修改
- 证据：`services/order.py` `update_order_status`直接赋值覆盖
- 证据：`models/Order.py` status字段是普通str类型，无枚举约束
- **行为后果**：可以直接将"Cancelled"状态改为"Delivered"，业务规则完全不设防

---

**总结**：这是一个面向学习和演示的电商API原型，采用极简架构实现了电商核心资源的CRUD操作。但在并发安全、数据一致性、业务完整性方面做了大量妥协，适合作为FastAPI学习项目或小型内部工具使用。如果用于生产环境需要解决大量架构和业务逻辑问题。
