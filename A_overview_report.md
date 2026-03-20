# 项目概览报告

## 1. 项目目标和边界

### 负责范围

这是一个基于 FastAPI 的电商后端 API。

证据：main.py 中 FastAPI 实例声明 title="E-commerce Backend API"，并挂载了 product、category、cart、order、payment、inventory、user 七个路由模块。

核心能力包括：
- 商品管理：增删改查、Excel 导出
- 分类管理：商品分类维护
- 购物车：添加、查询、删除
- 订单管理：创建、查询、状态更新
- 支付记录：记录和查询
- 库存管理：独立维护的库存记录
- 用户管理：注册、查询、简单认证

### 不负责范围

- 无真实支付集成
  证据：services/payment.py::create_payment() 仅将 payment 对象追加到列表并保存，无外部支付接口调用。

- 无库存扣减联动
  证据：services/order.py::create_order() 实现中无任何 inventory 或 product 相关调用，仅操作 orders.json。

- 无事务保证
  证据：所有 service 的写操作均为单文件全量覆盖（如 services/product.py::save_products()），无多文件事务协调机制。

- 无并发控制
  证据：services/product.py::save_products() 使用 `with open(PRODUCTS_FILE, "w")` 直接覆盖写入，无文件锁（fcntl/portalocker 等）。

- 无权限细控
  证据：routes/ 所有路由均未使用 FastAPI 的 Depends 进行权限校验，仅 user 模块有简单 authenticate_user 函数。

- 无数据关联校验
  证据：services/order.py::create_order() 不检查 user_id 是否存在于 users.json，也不检查订单涉及的商品是否有效。

---

## 2. 分层方式与职责

### main.py（应用入口层）

职责：FastAPI 实例创建、路由注册、全局元数据配置。

实现分析：
- 使用 `app.include_router()` 将七个路由模块统一挂载到 `/api` 前缀
- 无中间件配置（如 CORS、认证中间件、日志中间件）
- 元数据包含联系信息和许可证，但无实际安全或监控功能

这意味着：所有跨切面关注点（认证、日志、异常统一处理）均未在入口层实现，各路由模块自行处理。

### routes/（路由/控制器层）

职责：HTTP 请求处理、参数解析、状态码映射、异常转换。

实现分析：
- 统一使用 APIRouter 组织路由
- 异常处理模式固定：ValueError → 404/400，RuntimeError → 500
  证据：routes/product.py::add_product() 中 `except ValueError as e: raise HTTPException(status_code=400, detail=str(e))`
- 所有路由函数均包裹 try-except，捕获 Exception 并转为 HTTP 500

这意味着：异常处理过于宽泛，可能掩盖真实错误根源；同时所有错误信息直接透传给客户端，存在信息泄露风险。

### services/（业务逻辑层）

职责：核心业务逻辑、数据持久化、文件 I/O 操作。

实现分析：
- 每个 service 直接操作对应的 JSON 文件
  证据：services/product.py::PRODUCTS_FILE = "data/products.json"
- 文件不存在时自动创建空数组
  证据：services/product.py::load_products() 中 `except FileNotFoundError: ... json.dump([], file)`
- JSON 解码错误时静默返回空数组或打印错误
  证据：services/inventory.py::load_inventory() 中 `except json.JSONDecodeError: ... json.dump([], file)`
- 内存操作后全量写回文件
  证据：所有 save_X() 函数均使用 `json.dump([x.dict() for x in items], file, indent=4)`

这意味着：
- 数据完整性依赖文件系统操作的成功，无回滚机制
- JSON 损坏时数据被静默清空，无告警机制
- 全量读写导致性能随数据量线性下降
- 无并发控制，多进程环境下数据可能损坏

### models/（数据模型层）

职责：Pydantic 模型定义，用于请求/响应校验和序列化。

实现分析：
- 纯数据类，无业务方法
  证据：models/product.py 仅定义字段，无方法
- 部分模型使用 datetime 字段
  证据：models/Order.py 有 created_at: datetime；models/Inventory.py 有 last_updated: datetime
- 命名风格不一致
  证据：models/product.py（小写）vs models/Order.py（首字母大写）

这意味着：模型层仅提供数据校验和序列化功能，无领域逻辑；命名不一致反映缺乏代码规范约束。

### data/（数据存储层）

职责：JSON 文件持久化存储。

文件清单：
- products.json、categories.json、cart.json、orders.json、payments.json、inventory.json、users.json

实现分析：
- 每个实体独立文件，无关联约束
- 数据以明文 JSON 存储，无加密

这意味着：数据关系通过业务逻辑隐式维护，数据库层面的外键约束、级联删除等均不存在。

---

## 3. 关键执行路径分析

### 路径一：商品创建流程

```
POST /api/products/ → routes/product.py::add_product() 
  → services/product.py::create_product()
    → load_products() 读取 products.json
    → 检查 id 是否已存在
    → 追加到列表
    → save_products() 全量写回文件
```

关键观察：

1. 使用客户端传入的 id 作为主键
   证据：services/product.py::create_product() 中 `if any(p.id == product.id for p in products)`

2. 重复 id 会抛出 ValueError
   证据：services/product.py::create_product() 中 `raise ValueError(f"El producto con ID {product.id} ya existe.")`

3. 无分类存在性校验
   证据：create_product() 不检查 product.category_id 是否存在于 categories.json

这意味着：可创建指向不存在分类的商品；客户端需自行保证 id 唯一性。

### 路径二：订单创建流程

```
POST /api/orders/ → routes/order.py::add_order()
  → services/order.py::create_order()
    → load_orders() 读取 orders.json
    → 自动设置 created_at = datetime.utcnow()
    → 追加到列表
    → save_orders() 全量写回文件
```

关键观察：

1. 订单 id 由客户端传入，无自增逻辑
   证据：services/order.py::create_order() 直接 `orders.append(order)`，无 id 生成或校验

2. 不校验 user_id 是否存在
   证据：create_order() 不访问 users.json

3. 不校验订单内商品
   证据：create_order() 不访问 products.json；Order 模型无商品明细字段，仅 total_amount

4. 不扣减库存
   证据：create_order() 无任何 inventory 相关调用

5. 订单创建后购物车不会自动清空
   证据：create_order() 不访问 cart.json

这意味着：
- 可创建归属不存在用户的订单
- total_amount 由客户端传入，无计算校验，可被恶意篡改
- inventory 模块与订单完全脱节，超卖风险极高
- 订单与购物车无关联，用户需手动清理购物车

### 路径三：购物车添加流程

```
POST /api/cart/ → routes/cart.py::add_item_to_cart()
  → services/cart.py::add_to_cart()
    → load_cart() 读取 cart.json
    → 检查 (user_id, product_id) 组合是否已存在
      存在 → 累加 quantity
      不存在 → 追加新记录
    → save_cart() 全量写回文件
```

关键观察：

1. 复合主键逻辑：(user_id, product_id) 唯一
   证据：services/cart.py::add_to_cart() 中 `if cart_item.user_id == item.user_id and cart_item.product_id == item.product_id`

2. 不校验 user_id 是否存在
   证据：add_to_cart() 不访问 users.json

3. 不校验 product_id 是否存在
   证据：add_to_cart() 不访问 products.json

4. 不校验库存是否充足
   证据：add_to_cart() 不访问 inventory.json

这意味着：
- 可添加不存在的商品到购物车
- 可给不存在的用户添加购物车
- 可超量添加（超过库存）商品到购物车
- 同一用户同一商品多次添加表现为 quantity 累加

---

## 4. 数据存储与状态管理

### JSON 文件方案的收益

1. 零依赖部署
   证据：requirements.txt 仅依赖 fastapi、uvicorn、pydantic、openpyxl，无数据库驱动
   这意味着：无需安装和配置数据库服务，适合快速原型和演示环境。

2. 可读性强
   证据：data/products.json 以明文 JSON 存储，字段清晰可辨
   这意味着：便于调试和手动修改，但这也意味着数据无加密，敏感信息（如用户密码哈希）明文存储。

3. 结构简单
   证据：每个实体独立文件（products.json、orders.json 等）
   这意味着：职责清晰，但也导致跨实体查询需要多次文件读取。

4. 序列化便利
   证据：services/product.py::save_products() 使用 `[product.dict() for product in products]`
   这意味着：Pydantic 模型与 JSON 天然映射，代码简洁，但全量序列化性能随数据量下降。

### JSON 文件方案的限制

1. 无并发安全
   证据：services/product.py::save_products() 直接 `open(PRODUCTS_FILE, "w")` 覆盖写入
   这意味着：多进程/线程同时写入同一文件时，后完成的写入会覆盖先完成的，导致数据丢失。

2. 无事务支持
   证据：创建订单和扣减库存是两个独立文件操作，无协调机制
   这意味着：中间失败会导致数据不一致（如订单创建成功但库存未扣减）。

3. 全量读写性能瓶颈
   证据：所有 service 的 load_X() 均读取整个文件，save_X() 均写回整个文件
   这意味着：数据量增大时性能线性下降，内存占用随数据量增长。

4. 无查询能力
   证据：services/product.py::get_product_by_id() 使用 `next((p for p in products if p.id == product_id), None)`
   这意味着：所有查询均为线性扫描，无法高效过滤、排序、分页。

5. 数据冗余与一致性问题
   证据：models/product.py 有 stock 字段；models/Inventory.py 独立维护 quantity_available
   这意味着：同一概念在两个地方维护，且无同步逻辑，必然产生不一致。
   推断：inventory 模块可能是后期添加，与 product.stock 存在职责重叠。

---

## 5. 新同事接手时最容易误判的地方

### 5.1 "订单创建会校验用户和商品" → 实际不会

误判原因：常规电商系统创建订单时都会校验用户和商品。

实际情况：services/order.py::create_order() 仅做简单追加，无任何外键校验。

证据：services/order.py::create_order() 实现为 `order.created_at = datetime.utcnow(); orders.append(order); save_orders(orders)`，不访问 users.json 或 products.json。

后果：可能创建大量无效订单，或订单金额被恶意篡改。

### 5.2 "库存会自动扣减" → 实际不会

误判原因：存在 inventory 模块，容易误以为与订单有关联。

实际情况：orders 和 inventory 完全独立，无任何联动。

证据：services/order.py 无任何 inventory 相关 import 或调用。

后果：超卖风险，inventory 数据沦为摆设。

### 5.3 "删除返回 204 就真的没有响应体" → 实际有

误判原因：FastAPI 中 status_code=204 通常表示无响应体。

实际情况：routes/product.py::remove_product() 返回 {"detail": "..."}。

证据：routes/product.py::remove_product() 声明 `status_code=204` 但函数内 `return {"detail": f"Producto con ID {product_id} eliminado exitosamente."}`。

后果：HTTP 语义冲突，某些严格 HTTP 客户端可能解析失败。

### 5.4 "密码是安全存储的" → 实际不够安全

误判原因：看到使用了 SHA256 哈希。

实际情况：services/user.py::hash_password() 使用裸 SHA256，无盐值、无迭代。

证据：services/user.py::hash_password() 实现为 `sha256(password.encode()).hexdigest()`。

后果：易被彩虹表攻击，不符合现代密码存储标准。

### 5.5 "JSON 文件路径是配置化的" → 实际硬编码

误判原因：通常文件路径会提取到配置。

实际情况：所有 service 中 X_FILE = "data/xxx.json" 硬编码。

证据：services/product.py::PRODUCTS_FILE = "data/products.json"；services/order.py::ORDERS_FILE = "data/orders.json" 等。

后果：无法通过环境变量调整数据目录，测试环境难以隔离。

### 5.6 "异常处理很完善" → 实际有隐患

误判原因：看到大量 try-except 块。

实际情况：services/inventory.py::load_inventory() 在 JSONDecodeError 时会静默清空文件。

证据：services/inventory.py::load_inventory() 中 `except json.JSONDecodeError as e: ... with open(INVENTORY_FILE, "w") as file: json.dump([], file)`。

后果：数据损坏时无告警，原始数据丢失。
