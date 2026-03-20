# 电商后端API项目审查发现报告

## 审查发现按严重级别排序

---

### 编号：F001
**标题**：JSON文件写操作无锁机制，并发写入导致数据丢失
**级别**：Critical
**证据**：`services/product.py` `save_products`函数、`services/cart.py` `save_cart`函数
**问题说明**：数据写操作都采用"读取整个文件到内存→修改内存数据→覆盖写入文件"的模式。整个过程无文件锁、无乐观锁、无任何并发控制机制。当两个请求同时执行写操作时，后写入的请求会完全覆盖先写入的内容，造成数据永久丢失。
**影响范围**：已检查模块的写操作（创建、更新、删除）
**是否属于真实生产风险**：是

---

### 编号：F002
**标题**：创建订单时完全不检查和扣减库存，超卖风险严重
**级别**：Critical
**证据**：`services/order.py` `create_order`函数
**问题说明**：创建订单接口的业务逻辑仅包含：读取现有订单列表→设置created_at时间戳→添加新订单→保存到文件。完全不涉及库存的检查或扣减逻辑，即使库存为0或负数也能成功下单。
**影响范围**：订单模块
**是否属于真实生产风险**：是

---

### 编号：F003
**标题**：删除商品或分类时不检查本模块外的引用，可能产生关联数据引用不存在ID
**级别**：Critical
**证据**：`services/product.py` `delete_product`函数、`services/category.py` `delete_category`函数
**问题说明**：删除商品或分类时，仅简单地从列表中过滤掉对应ID的记录，不检查（也无法检查，因无跨模块调用）其他JSON文件中是否存在关联记录。删除后，其他文件中的关联记录会引用不存在的ID。
**影响范围**：商品删除、分类删除
**是否属于真实生产风险**：是

---

### 编号：F004
**标题**：订单状态无约束，可任意修改无状态机校验
**级别**：Critical
**证据**：`services/order.py` `update_order_status`函数、`models/Order.py` status字段定义
**问题说明**：订单状态是自由字符串类型，无枚举约束；更新状态时直接赋值覆盖，无任何状态转移规则校验（如Pending→Paid→Shipped）。可以将"Cancelled"直接改为"Delivered"，业务规则完全不设防。
**影响范围**：订单状态管理
**是否属于真实生产风险**：是

---

### 编号：F005
**标题**：商品stock字段与inventory模块完全独立，双库存数据不一致
**级别**：High
**证据**：`models/product.py` Product类stock字段、`services/inventory.py` 库存操作函数
**问题说明**：商品模型中有stock字段，但inventory模块维护独立的库存记录。两者之间无任何同步机制：修改商品stock时，inventory表无变化；修改inventory表时，商品stock也无变化。
**影响范围**：商品模块、库存模块
**是否属于真实生产风险**：是

---

### 编号：F006
**标题**：用户密码仅用SHA-256哈希，无加盐处理易受彩虹表攻击
**级别**：High
**证据**：`services/user.py` `hash_password`函数
**问题说明**：密码哈希使用简单的`sha256(password.encode()).hexdigest()`，无随机盐。相同密码会产生相同哈希值，易受彩虹表暴力破解攻击。
**影响范围**：用户认证模块
**是否属于真实生产风险**：是

---

### 编号：F007
**标题**：购物车添加商品不验证商品存在性和库存
**级别**：High
**证据**：`services/cart.py` `add_to_cart`函数
**问题说明**：添加商品到购物车时，仅检查该用户购物车中是否已有该商品，不验证product_id是否真的存在于products.json中，也不检查库存数量是否足够。可以成功添加不存在的商品。
**影响范围**：购物车模块
**是否属于真实生产风险**：是

---

### 编号：F008
**标题**：商品更新接口允许URL参数与body数据中的ID不匹配，更新后商品ID被覆盖为body中的ID
**级别**：High
**证据**：`services/product.py` `update_product`函数、`routes/product.py` `update_existing_product`路由
**问题说明**：更新商品时，URL路径中的`product_id`参数用于查找要更新的商品，但找到匹配的商品后，直接用请求body中的`product`对象完全覆盖。如果URL中的`product_id`与body中的`product.id`不一致，会导致：1) 实际更新的是URL指定ID的商品，但2) 保存后该商品的ID变为body中指定的ID。极端情况下，如果body中的id与其他商品冲突时会覆盖其他商品的数据。
**影响范围**：商品更新操作
**是否属于真实生产风险**：是

---

### 编号：F009
**标题**：订单状态更新接口使用query参数传递状态，不符合RESTful设计规范
**级别**：Medium
**证据**：`routes/order.py` `modify_order_status`路由
**问题说明**：PUT /api/orders/{order_id}/status接口的status参数是通过query参数传递的，不是通过request body。这种设计不符合RESTful API的最佳实践。
**影响范围**：订单状态更新接口
**是否属于真实生产风险**：否（语义设计问题）

---

### 编号：F010
**标题**：异常处理不统一，RuntimeError与Exception捕获混用
**级别**：Medium
**证据**：`routes/order.py` 捕获RuntimeError、`routes/cart.py` 捕获RuntimeError、`routes/product.py` 捕获Exception
**问题说明**：有些路由捕获`RuntimeError`返回500，有些直接捕获`Exception`。服务层抛出的`RuntimeError`与真正的运行时异常（如IO错误）无法区分，错误处理逻辑混乱。
**影响范围**：所有路由的异常处理
**是否属于真实生产风险**：否（影响调试和错误定位）

---

### 编号：F011
**标题**：创建订单时覆盖客户端传入的created_at字段
**级别**：Medium
**证据**：`services/order.py` `order.created_at = datetime.utcnow()`
**问题说明**：创建订单时，服务端无条件地用当前时间覆盖`created_at`字段。即使客户端传入了有效的created_at值，也会被忽略。
**影响范围**：订单创建功能
**是否属于真实生产风险**：否（行为一致性问题）

---

### 编号：F012
**标题**：用户认证成功后无token或session返回
**级别**：Medium
**证据**：`routes/user.py` `login`路由
**问题说明**：用户认证接口仅返回"Autenticación exitosa."成功消息，无任何token或session返回。后续请求无法识别用户身份，认证接口形同虚设。
**影响范围**：用户认证模块
**是否属于真实生产风险**：是（认证机制不完整）

---

## 不是bug但值得警惕的设计选择

### 1. JSON文件存储的可扩展性瓶颈
**设计说明**：所有数据都存储在JSON文件中，每次操作都要完整读写。
- 潜在问题：当数据量增长时，内存占用和响应延迟会显著增加
- 证据：所有`load_*()`函数都将整个文件加载到内存（推断：基于代码审查）
- 警惕点：这种简单方案在小规模数据下运行良好，但数据量增长后性能可能急剧下降

### 2. 模块间无依赖验证，数据完整性完全依赖调用者
**设计说明**：各模块完全独立，创建订单时不检查关联ID是否存在
- 潜在问题：API使用者容易传入无效ID，系统中积累大量无效引用
- 证据：`services/order.py` `create_order`函数不验证外键
- 警惕点：这种"信任调用者"的设计在内部控制的系统中可能可行，但对外暴露API时会产生大量无效数据

### 3. 全表扫描的查询方式
**设计说明**：所有按ID查询都用遍历列表方式实现（next()或列表推导式）
- 潜在问题：O(n)时间复杂度，数据量大时查询性能急剧下降
- 证据：`services/product.py` `get_product_by_id`函数用next()遍历
- 警惕点：简单实现掩盖了算法复杂度问题，在压力测试前难以发现

### 4. 硬编码的文件路径
**设计说明**：数据文件路径在每个service文件中硬编码
- 潜在问题：部署结构变更时需要修改多处源码，不同环境切换配置困难
- 证据：`services/product.py` `PRODUCTS_FILE`常量定义、`services/order.py` `ORDERS_FILE`常量定义
- 警惕点：缺乏配置中心或环境变量支持，运维灵活性差
