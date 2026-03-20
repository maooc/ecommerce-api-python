# 项目概览报告

## 1. 项目目标和边界

### 这个 API 服务负责什么

这是一个基于 FastAPI 的电商后端 API，提供以下核心功能：

- **商品管理**：商品的 CRUD 操作，支持 Excel 导出
- **分类管理**：商品分类的 CRUD 操作
- **购物车**：用户购物车的增删查
- **订单管理**：订单创建、查询、状态更新
- **支付记录**：支付信息的创建和查询
- **库存管理**：库存记录的 CRUD
- **用户管理**：用户创建、查询、简单认证

证据：`main.py:11-17` 注册了 7 个路由模块，覆盖上述所有功能域。

### 这个 API 服务不负责什么

- **完整的认证授权系统**：仅有 `POST /api/users/authenticate` 做密码校验，无 JWT/Session 机制，无权限控制
- **库存业务逻辑**：库存模块仅做数据存储，不与订单、购物车联动
- **价格计算**：订单的 `total_amount` 由调用方传入，服务端不计算
- **数据一致性保障**：无事务机制，无并发控制
- **前端渲染**：纯 REST API，返回 JSON

证据：`services/order.py:39-45` 创建订单时直接使用传入的 `total_amount`，未做任何计算或校验。

---

## 2. 分层方式

项目采用经典的三层架构，外加数据文件层：

| 层级 | 目录 | 职责 |
|------|------|------|
| 入口层 | `main.py` | FastAPI 应用实例创建、路由注册、全局配置 |
| 路由层 | `routes/` | HTTP 请求处理、参数校验（依赖 Pydantic）、异常转换、响应封装 |
| 服务层 | `services/` | 业务逻辑、数据加载/保存、跨模块协调（实际几乎没有） |
| 模型层 | `models/` | Pydantic 数据模型定义，负责输入输出结构约束 |
| 数据层 | `data/` | JSON 文件存储，无抽象层 |

**分层特点**：

- 路由层极薄，仅做异常类型到 HTTP 状态码的映射
- 服务层直接操作文件，无 Repository 抽象
- 模型层无业务方法，纯数据结构定义

证据：`routes/product.py:25-32` 路由函数仅调用 service 并捕获异常；`services/product.py:18-28` 服务函数直接读写文件。

---

## 3. 关键执行路径

### 3.1 商品查询链路

```
GET /api/products
    ↓
routes/product.py:read_products()
    ↓
services/product.py:get_all_products()
    ↓
services/product.py:load_products()  [读取文件]
    ↓
data/products.json
```

**执行特点**：每次请求都完整读取 JSON 文件，无缓存机制。

证据：`services/product.py:18-28` 的 `load_products()` 函数。

### 3.2 购物车添加链路

```
POST /api/cart
    ↓
routes/cart.py:add_item_to_cart()
    ↓
services/cart.py:add_to_cart()
    ↓
services/cart.py:load_cart()  [读取文件]
    ↓
[检查是否已存在同类项，存在则累加数量]
    ↓
services/cart.py:save_cart()  [全量写入文件]
    ↓
data/cart.json
```

**执行特点**：采用"读取-修改-全量写入"模式，无锁机制。

证据：`services/cart.py:38-55` 的 `add_to_cart()` 函数。

### 3.3 订单创建链路

```
POST /api/orders
    ↓
routes/order.py:add_order()
    ↓
services/order.py:create_order()
    ↓
services/order.py:load_orders()  [读取文件]
    ↓
[设置 created_at 时间戳]
    ↓
services/order.py:save_orders()  [全量写入文件]
    ↓
data/orders.json
```

**执行特点**：订单创建时不校验用户存在性、不检查库存、不扣减库存、不清理购物车。

证据：`services/order.py:39-45` 的 `create_order()` 函数。

---

## 4. 数据存储与状态管理

### JSON 文件方案带来的收益

1. **零依赖部署**：无需安装数据库，仅需 Python 环境
2. **调试友好**：可直接查看/编辑 JSON 文件
3. **版本控制友好**：JSON 文件可纳入 Git 管理
4. **学习成本低**：代码逻辑直观，无 ORM 学习曲线

证据：`requirements.txt` 中无数据库驱动依赖。

### JSON 文件方案带来的限制

1. **无事务支持**：跨文件操作无法保证原子性（如创建订单同时扣减库存）
2. **无并发控制**：多进程/多线程同时写入会导致数据丢失
3. **全量读写**：每次操作都读写整个文件，数据量大时性能急剧下降
4. **无索引机制**：查询依赖内存遍历，O(n) 复杂度
5. **无数据完整性约束**：外键关系靠约定，无强制校验

证据：`services/product.py:31-36` 的 `save_products()` 函数采用全量覆盖写入；`services/cart.py:44-48` 通过列表遍历查找目标项。

### 数据冗余问题

`products.json` 中的 `stock` 字段与 `inventory.json` 中的 `quantity_available` 字段语义重叠，但无同步机制：

- 证据：`data/products.json:L3` 的 `"stock": 50` 与 `data/inventory.json:L3` 的 `"quantity_available": 50` 数值相同，但代码中无关联逻辑

---

## 5. 新同事接手时最容易误判的地方

### 5.1 "Product.stock 和 Inventory 是一回事"

**误判**：认为修改 `Product.stock` 会同步到 `Inventory`，或反之。

**实际**：两者完全独立，`Product.stock` 在商品模块维护，`Inventory.quantity_available` 在库存模块维护，互不影响。

证据：`services/product.py` 和 `services/inventory.py` 无互相调用。

### 5.2 "Order.created_at 需要客户端传入"

**误判**：看到 `models/Order.py:L8` 定义了 `created_at: datetime`，认为创建订单时必须传入该字段。

**实际**：服务层会自动设置该字段，客户端传入的值会被覆盖。

证据：`services/order.py:42` 强制设置 `order.created_at = datetime.utcnow()`。

### 5.3 "用户密码是加密存储的"

**误判**：看到 `services/user.py:19-21` 有 `hash_password()` 函数，认为所有密码都是加密存储。

**实际**：`data/users.json` 中的初始数据是明文密码，只有通过 API 新创建的用户才会被 hash。

证据：`data/users.json:L5` 的 `"password": "contraseña123"` 是明文；`services/user.py:51` 仅在 `create_user()` 时调用 hash。

### 5.4 "DELETE 接口返回 204 就没有响应体"

**误判**：看到路由装饰器 `status_code=204`，认为接口不会返回任何内容。

**实际**：代码中返回了 `{"detail": "..."}` 字典，虽然 HTTP 204 规范要求无响应体，但 FastAPI 仍会序列化该字典。

证据：`routes/product.py:52-55` 设置 `status_code=204` 但返回了字典。

### 5.5 "认证接口是 POST body 传参"

**误判**：认为 `POST /api/users/authenticate` 会从请求体获取用户名密码。

**实际**：使用查询参数 `?username=xxx&password=xxx` 传递，密码会出现在 URL 和服务器日志中。

证据：`routes/user.py:35` 的函数签名 `def login(username: str, password: str)` 未指定来源，FastAPI 默认作为查询参数处理。

---

## 6. 架构图示

```
┌─────────────────────────────────────────────────────────────┐
│                         main.py                              │
│                    (FastAPI App Entry)                       │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ routes/product│    │  routes/cart  │    │ routes/order  │
│ routes/...    │    │  routes/...   │    │ routes/...    │
└───────────────┘    └───────────────┘    └───────────────┘
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│services/product│   │ services/cart │    │services/order │
│ services/...   │   │ services/...  │    │ services/...  │
└───────────────┘    └───────────────┘    └───────────────┘
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              ▼
                    ┌───────────────────┐
                    │   data/*.json     │
                    │  (File Storage)   │
                    └───────────────────┘
```

**关键特征**：各模块完全独立，无横向调用，数据层无抽象。
