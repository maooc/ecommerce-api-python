# 代码审查发现报告

## 审查发现列表（按严重级别排序）

---

### F001

- **编号**: F001
- **标题**: 订单创建无用户/商品校验，存在数据完整性风险
- **级别**: 严重
- **证据**: services/order.py::create_order() 实现为 `order.created_at = datetime.utcnow(); orders.append(order); save_orders(orders)`，无 users.json 或 products.json 访问
- **问题说明**: 创建订单时不校验 user_id 是否存在、不校验订单商品是否有效、不校验 total_amount 是否匹配商品计算。系统可接受任意伪造数据。
- **影响范围**: 订单模块，数据完整性
- **是否属于真实生产风险**: 是

---

### F002

- **编号**: F002
- **标题**: 库存与订单完全脱节，存在超卖风险
- **级别**: 严重
- **证据**: services/order.py 无任何 inventory 相关 import 或调用；services/inventory.py 无任何被 order 模块调用的痕迹
- **问题说明**: 创建订单时不扣减库存，不检查库存余量。inventory 模块与订单系统完全独立，无法防止超卖。
- **影响范围**: 订单、库存、商品模块
- **是否属于真实生产风险**: 是

---

### F003

- **编号**: F003
- **标题**: 密码使用裸 SHA256 哈希，无盐值保护
- **级别**: 严重
- **证据**: services/user.py::hash_password() 实现为 `sha256(password.encode()).hexdigest()`
- **问题说明**: SHA256 是快速哈希算法，无盐值保护，极易被彩虹表攻击。不符合现代密码存储标准。
- **影响范围**: 用户认证模块
- **是否属于真实生产风险**: 是

---

### F004

- **编号**: F004
- **标题**: JSON 文件无并发控制，多进程下数据会损坏
- **级别**: 严重
- **证据**: services/product.py::save_products() 使用 `with open(PRODUCTS_FILE, "w")` 直接覆盖写入，无文件锁（fcntl/portalocker 等）
- **问题说明**: 多个请求同时写入同一文件时，后完成的写入会覆盖先完成的，导致数据丢失。
- **影响范围**: 所有使用 JSON 文件的模块
- **是否属于真实生产风险**: 是

---

### F005

- **编号**: F005
- **标题**: JSON 解码错误时静默清空文件，导致数据丢失
- **级别**: 高
- **证据**: services/inventory.py::load_inventory() 中 `except json.JSONDecodeError: ... with open(INVENTORY_FILE, "w") as file: json.dump([], file)`
- **问题说明**: 当 JSON 文件损坏时，系统不告警、不备份，直接清空文件。生产环境数据可能永久丢失。
- **影响范围**: inventory 模块（services/product.py、services/category.py 等也有类似模式）
- **是否属于真实生产风险**: 是

---

### F006

- **编号**: F006
- **标题**: 删除操作返回 204 但带有响应体，违反 HTTP 语义
- **级别**: 高
- **证据**: routes/product.py::remove_product() 声明 `status_code=204` 但函数内 `return {"detail": f"Producto con ID {product_id} eliminado exitosamente."}`
- **问题说明**: HTTP 204 表示无内容，但接口返回 JSON 响应体。某些严格 HTTP 客户端可能拒绝解析。
- **影响范围**: 所有删除路由（routes/product.py::remove_product、routes/category.py::remove_category、routes/inventory.py::delete_stock、routes/cart.py::remove_item_from_cart）
- **是否属于真实生产风险**: 是

---

### F007

- **编号**: F007
- **标题**: 购物车添加不校验用户和商品存在性
- **级别**: 中
- **证据**: services/cart.py::add_to_cart() 实现为检查 `(cart_item.user_id == item.user_id and cart_item.product_id == item.product_id)`，不访问 users.json 或 products.json
- **问题说明**: 可添加不存在的商品到购物车，或给不存在的用户添加购物车。数据无意义但系统接受。
- **影响范围**: 购物车模块
- **是否属于真实生产风险**: 否

---

### F008

- **编号**: F008
- **标题**: 商品和库存数据冗余且无同步机制
- **级别**: 中
- **证据**: models/product.py 定义 `stock: int`；models/Inventory.py 定义 `quantity_available: int`；两模块无互相调用
- **问题说明**: 同一概念在两个地方维护，且无同步逻辑。修改一处另一处不会更新，必然产生不一致。
- **影响范围**: 商品、库存模块
- **是否属于真实生产风险**: 否（推断：目前 inventory 模块未被实际使用）

---

### F009

- **编号**: F009
- **标题**: 订单 ID 由客户端传入，无自增/唯一性保证
- **级别**: 中
- **证据**: services/order.py::create_order() 实现为 `orders.append(order); save_orders(orders)`，无 `if any(o.id == order.id for o in orders)` 类检查
- **问题说明**: 与 product/category 不同，order 创建时不校验 ID 唯一性。重复 ID 会导致查询返回多个结果或覆盖。
- **影响范围**: 订单模块
- **是否属于真实生产风险**: 是

---

### F010

- **编号**: F010
- **标题**: 文件路径硬编码，无法配置化
- **级别**: 低
- **证据**: services/product.py::PRODUCTS_FILE = "data/products.json"；services/order.py::ORDERS_FILE = "data/orders.json"；services/cart.py::CART_FILE = "data/cart.json" 等
- **问题说明**: 数据目录无法通过环境变量调整，测试环境难以隔离，无法灵活部署到不同目录。
- **影响范围**: 所有 service 模块
- **是否属于真实生产风险**: 否

---

### F011

- **编号**: F011
- **标题**: 异常处理过于宽泛，隐藏真实错误信息
- **级别**: 低
- **证据**: routes/product.py::add_product() 中 `except Exception as e: raise HTTPException(status_code=500, detail=str(e))`；routes/order.py、routes/cart.py 等均有相同模式
- **问题说明**: 捕获过于宽泛的异常，可能将内部错误细节暴露给客户端，或掩盖真正的问题根源。
- **影响范围**: 所有 routes 模块
- **是否属于真实生产风险**: 否

---

## 不是 Bug 但值得警惕的设计选择

### D001: 购物车使用复合主键而非独立 ID

说明：CartItem 模型没有独立 id 字段，使用 (user_id, product_id) 作为逻辑主键。

证据：models/Cart.py 定义 `class CartItem(BaseModel): user_id: int; product_id: int; quantity: int`，无 id 字段；services/cart.py::add_to_cart() 中 `if cart_item.user_id == item.user_id and cart_item.product_id == item.product_id` 实现复合主键查找

潜在问题：
- 同一用户无法对同一商品创建多条购物车记录（如不同规格）
- 如果需要扩展购物车功能（如收藏、稍后购买），当前结构受限
- 删除后重新添加会丢失历史记录

---

### D002: 订单 total_amount 由客户端传入而非服务端计算

说明：Order 模型的 total_amount 是必填字段，由调用方提供。

证据：models/Order.py 定义 `total_amount: float`；services/order.py::create_order() 直接使用传入值，无计算逻辑

潜在问题：
- 客户端可随意篡改订单金额
- 服务端无校验逻辑，无法防止恶意低价下单
- 与购物车数据无关联，订单内容与金额可能不匹配

---

### D003: 模型命名风格不一致

说明：models/ 目录下同时存在小写和首字母大写的模块名。

证据：models/product.py（小写）、models/category.py（小写）vs models/Order.py（首字母大写）、models/Cart.py（首字母大写）、models/Inventory.py（首字母大写）、models/Payment.py（首字母大写）

潜在问题：
- 影响代码可读性和维护性
- 可能导致导入时的困惑
- 不符合 Python PEP8 命名规范（模块名应小写）

---

### D004: 库存更新直接设置值而非增减操作

说明：services/inventory.py::update_inventory() 接收 quantity 参数直接赋值，而非增减。

证据：services/inventory.py::update_inventory() 实现为 `inv.quantity_available = quantity`，非 `inv.quantity_available += quantity` 或 `-=`

潜在问题：
- 调用方需要自行计算新库存值，增加出错概率
- 无并发控制时，两个同时的更新请求会导致最终值不确定
- 无法实现扣减 N 件这样的原子操作语义

---
