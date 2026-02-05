# 别为了接个老系统就把代码改得面目全非！Go 适配器模式让你的系统集成优雅又省事

上周有个紧急需求："公司接了一个大客户，他们用的是 10 年前的老支付系统，接口跟我们的完全不兼容，必须对接。"

我一看那个老系统的 API 文档，差点当场辞职：

```go
// 我们的支付系统（标准接口）
type PaymentProcessor interface {
    Pay(orderID string, amount float64) error
    Refund(orderID string, amount float64) error
}

// 客户的老支付系统（完全不兼容！）
type LegacyPayPal struct{}

func (l *LegacyPayPal) ProcessPayment(orderID string, amount float64) (bool, error) {
    // 返回 bool 表示是否成功，跟我们用 error 完全不一样
}

func (l *LegacyPayPal) CancelTransaction(transactionID string) (string, error) {
    // 返回的是取消确认码，不是我们需要的 orderID
}

func (l *LegacyPayPal) GetTransactionStatus(transactionID string) int {
    // 返回的是状态码：0=成功, 1=处理中, 2=失败
    // 我们用的是枚举类型！
}
```

"这也太坑了吧！难道我要为了接这个老系统，把我们整个支付模块都改了？"

今天不讲废话，我们直接拆解适配器模式，看看怎么把这种"接口不兼容"的场景处理得优雅又解耦。

---

## 一、适配器模式到底是干嘛的？

用一句话说清楚：**把一个接口"翻译"成另一个接口，让不兼容的系统能协同工作。**

看个生活例子：

```
你：用 Type-C 充电线
旧手机：用 Micro-USB 充电口

问题：你的线插不进去！

解决：买个转接头（适配器）
Type-C 转接头 -> 插进旧手机的 Micro-USB

转接头做的事：
- 一端接你的 Type-C 线
- 另一端转成 Micro-USB
- 让你和旧手机都能充电

关键点：
- 你的充电器不用改
- 旧手机不用改
- 只有转接头需要做转换
```

**这就是适配器模式的核心：在两个不兼容的接口之间加一层"翻译"。**

---

## 二、不用适配器模式会有多恶心？

假设你要对接一个老系统，但接口完全不兼容。

### ❌ 反面教材：直接改现有代码

```go
// 我们的订单服务（标准接口）
type OrderService struct {
    payment PaymentProcessor  // 标准接口
}

func (o *OrderService) CreateOrder(order Order) error {
    // 处理订单逻辑...

    // 支付
    err := o.payment.Pay(order.ID, order.Amount)
    if err != nil {
        return err
    }

    return nil
}

// ❌ 为了接老系统，要把 PaymentProcessor 改掉！
type NewPaymentProcessor interface {
    Pay(orderID string, amount float64, customerID string) error  // 加了 customerID
    Refund(transactionID string, amount float64, reason string) error  // 参数都改了
    GetStatus(transactionID string) string  // 返回类型也改了
}

// ❌ 所有调用方都要改！
func (o *OrderService) CreateOrder(order Order) error {
    // 现在要传 customerID 了
    err := o.payment.Pay(order.ID, order.Amount, order.CustomerID)

    // 现在要传 reason 了
    // o.payment.Refund(order.ID, order.Amount, "用户取消")

    return nil
}
```

**这代码的毛病：**

1. **破坏现有接口**：为了接老系统，改了自己的标准接口
2. **影响所有调用方**：改一个接口，所有调用方都要改
3. **违反开闭原则**：对扩展不友好，对修改却大开方便之门
4. **难以测试**：想测标准接口，现在还得测老系统的接口
5. **可能引入 Bug**：改接口改错了，整个系统都可能出问题

**这代码维护起来就像是在给旧系统擦屁股，越擦越脏。**

---

## 三、Go 版适配器模式实现

### 基础实现：类适配器

```go
// ========== 我们的支付系统（标准接口） ==========
type PaymentProcessor interface {
    Pay(orderID string, amount float64) error
    Refund(orderID string, amount float64) error
}

// ========== 客户的老支付系统（被适配者） ==========
type LegacyPayPal struct{}

func (l *LegacyPayPal) ProcessPayment(orderID string, amount float64) (bool, error) {
    fmt.Printf("老系统处理支付: orderID=%s, amount=%.2f\n", orderID, amount)
    return true, nil
}

func (l *LegacyPayPal) CancelTransaction(transactionID string) (string, error) {
    fmt.Printf("老系统取消交易: transactionID=%s\n", transactionID)
    return "CANCELLED_" + transactionID, nil
}

func (l *LegacyPayPal) GetTransactionStatus(transactionID string) int {
    fmt.Printf("老系统查询状态: transactionID=%s\n", transactionID)
    return 0  // 0=成功
}

// ========== 适配器 ==========
type PayPalAdapter struct {
    legacy *LegacyPayPal  // 持有老系统的引用
}

func NewPayPalAdapter(legacy *LegacyPayPal) *PayPalAdapter {
    return &PayPalAdapter{legacy: legacy}
}

// 把我们的标准接口翻译成老系统的接口
func (p *PayPalAdapter) Pay(orderID string, amount float64) error {
    success, err := p.legacy.ProcessPayment(orderID, amount)

    // 老系统返回 bool，我们转成 error
    if !success {
        return fmt.Errorf("支付失败")
    }

    return err
}

func (p *PayPalAdapter) Refund(orderID string, amount float64) error {
    // 老系统用 CancelTransaction，我们用 Refund
    _, err := p.legacy.CancelTransaction(orderID)
    return err
}

// ========== 使用 ==========
func main() {
    // 创建老系统
    legacyPayPal := &LegacyPayPal{}

    // 用适配器包装老系统
    paymentProcessor := NewPayPalAdapter(legacyPayPal)

    // 现在老系统可以像标准接口一样使用了！
    err := paymentProcessor.Pay("ORDER_123", 99.99)
    if err != nil {
        log.Fatal("支付失败:", err)
    }

    err = paymentProcessor.Refund("ORDER_123", 99.99)
    if err != nil {
        log.Fatal("退款失败:", err)
    }
}
```

**输出：**

```
老系统处理支付: orderID=ORDER_123, amount=99.99
老系统取消交易: transactionID=ORDER_123
```

---

## 四、实战案例：多支付渠道适配

假设你要对接多个支付渠道，每个渠道的接口都不一样。

```go
// ========== 标准接口 ==========
type PaymentProcessor interface {
    Pay(order Order) PaymentResult
    Refund(orderID string, amount float64) RefundResult
}

type Order struct {
    ID         string
    Amount     float64
    Currency   string
    CustomerID string
    Product    string
}

type PaymentResult struct {
    Success     bool
    TransactionID string
    Message     string
}

type RefundResult struct {
    Success   bool
    RefundID  string
    Message   string
}

// ========== Stripe 适配器 ==========
type StripeAdapter struct {
    apiKey string
    client *http.Client
}

func NewStripeAdapter(apiKey string) *StripeAdapter {
    return &StripeAdapter{
        apiKey: apiKey,
        client: &http.Client{Timeout: 30 * time.Second},
    }
}

func (s *StripeAdapter) Pay(order Order) PaymentResult {
    // Stripe 的支付请求格式
    stripeReq := map[string]interface{}{
        "amount":      int(order.Amount * 100),  // Stripe 用分，我们用元
        "currency":    order.Currency,
        "description": order.Product,
        "metadata": map[string]string{
            "order_id":     order.ID,
            "customer_id":  order.CustomerID,
        },
    }

    // 调用 Stripe API
    // response := s.callStripeAPI("/v1/charges", stripeReq)

    // 转换成我们的标准返回
    return PaymentResult{
        Success:       true,
        TransactionID: "ch_" + order.ID,
        Message:       "支付成功",
    }
}

func (s *StripeAdapter) Refund(orderID string, amount float64) RefundResult {
    // 调用 Stripe 退款 API
    // ...

    return RefundResult{
        Success:  true,
        RefundID: "re_" + orderID,
        Message:  "退款成功",
    }
}

// ========== PayPal 适配器 ==========
type PayPalAdapter struct {
    clientID string
    secret   string
}

func NewPayPalAdapter(clientID, secret string) *PayPalAdapter {
    return &PayPalAdapter{
        clientID: clientID,
        secret:   secret,
    }
}

func (p *PayPalAdapter) Pay(order) PaymentResult {
    // PayPal 的支付请求格式完全不一样
    paypalReq := map[string]interface{}{
        "intent": "sale",
        "payer": map[string]interface{}{
            "payment_method": "credit_card",
        },
        "transactions": []map[string]interface{}{
            {
                "amount": map[string]interface{}{
                    "total":    fmt.Sprintf("%.2f", order.Amount),
                    "currency": order.Currency,
                },
                "description": order.Product,
            },
        },
    }

    // 调用 PayPal API
    // ...

    return PaymentResult{
        Success:       true,
        TransactionID: "PAY-" + order.ID,
        Message:       "支付成功",
    }
}

func (p *PayPalAdapter) Refund(orderID string, amount float64) RefundResult {
    // 调用 PayPal 退款 API
    // ...

    return RefundResult{
        Success:  true,
        RefundID: "REF-" + orderID,
        Message:  "退款成功",
    }
}

// ========== 支付工厂 ==========
type PaymentFactory struct {
    processors map[string]PaymentProcessor
}

func NewPaymentFactory() *PaymentFactory {
    factory := &PaymentFactory{
        processors: make(map[string]PaymentProcessor),
    }

    // 注册支付处理器
    factory.Register("stripe", NewStripeAdapter(os.Getenv("STRIPE_API_KEY")))
    factory.Register("paypal", NewPayPalAdapter(
        os.Getenv("PAYPAL_CLIENT_ID"),
        os.Getenv("PAYPAL_SECRET"),
    ))

    return factory
}

func (f *PaymentFactory) Register(name string, processor PaymentProcessor) {
    f.processors[name] = processor
}

func (f *PaymentFactory) GetProcessor(name string) (PaymentProcessor, error) {
    processor, exists := f.processors[name]
    if !exists {
        return nil, fmt.Errorf("不支持的支付方式: %s", name)
    }
    return processor, nil
}

// ========== 订单服务 ==========
type OrderService struct {
    paymentFactory *PaymentFactory
}

func (o *OrderService) CreateOrder(order Order, paymentMethod string) error {
    // 获取支付处理器
    processor, err := o.paymentFactory.GetProcessor(paymentMethod)
    if err != nil {
        return err
    }

    // 支付（不管底层是什么系统，接口都一样！）
    result := processor.Pay(order)

    if !result.Success {
        return fmt.Errorf("支付失败: %s", result.Message)
    }

    fmt.Printf("订单 %s 支付成功，交易 ID: %s\n", order.ID, result.TransactionID)
    return nil
}

// ========== 使用 ==========
func main() {
    factory := NewPaymentFactory()
    orderService := &OrderService{paymentFactory: factory}

    order := Order{
        ID:         "ORDER_123",
        Amount:     99.99,
        Currency:   "USD",
        CustomerID: "CUST_456",
        Product:    "iPhone 15",
    }

    // 用 Stripe 支付
    err := orderService.CreateOrder(order, "stripe")
    if err != nil {
        log.Fatal(err)
    }

    // 用 PayPal 支付（代码不用改，换个名字就行）
    err = orderService.CreateOrder(order, "paypal")
    if err != nil {
        log.Fatal(err)
    }
}
```

**这代码写得简直太优雅了：**

1. **标准接口统一**：不管底层是什么系统，接口都一样
2. **新系统不影响旧代码**：加一个新支付方式，其他代码不用动
3. **易于扩展**：对接新系统，写个适配器就行
4. **易于测试**：可以 mock 标准接口，不用管底层实现

---

## 五、进阶：双向适配器

有时候你需要让两个不同的系统互相调用。

```go
// ========== 系统 A 的接口 ==========
type SystemA interface {
    MethodA(data string) string
}

// ========== 系统 B 的接口 ==========
type SystemB interface {
    MethodB(data int) int
}

// ========== 系统 A 的实现 ==========
type ConcreteSystemA struct{}

func (c *ConcreteSystemA) MethodA(data string) string {
    fmt.Println("System A 处理:", data)
    return "A_RESULT_" + data
}

// ========== 系统 B 的实现 ==========
type ConcreteSystemB struct{}

func (c *ConcreteSystemB) MethodB(data int) int {
    fmt.Println("System B 处理:", data)
    return data * 2
}

// ========== 双向适配器 ==========
type BidirectionalAdapter struct {
    systemA *ConcreteSystemA
    systemB *ConcreteSystemB
}

func NewBidirectionalAdapter(systemA *ConcreteSystemA, systemB *ConcreteSystemB) *BidirectionalAdapter {
    return &BidirectionalAdapter{
        systemA: systemA,
        systemB: systemB,
    }
}

// 实现 SystemA 接口（内部调用 SystemB）
func (b *BidirectionalAdapter) MethodA(data string) string {
    // 把 string 转成 int
    num, _ := strconv.Atoi(data)

    // 调用 SystemB
    result := b.systemB.MethodB(num)

    // 把 int 转成 string
    return fmt.Sprintf("%d", result)
}

// 实现 SystemB 接口（内部调用 SystemA）
func (b *BidirectionalAdapter) MethodB(data int) int {
    // 把 int 转成 string
    str := fmt.Sprintf("%d", data)

    // 调用 SystemA
    result := b.systemA.MethodA(str)

    // 把 string 转成 int
    num, _ := strconv.Atoi(result)
    return num
}

// ========== 使用 ==========
func main() {
    systemA := &ConcreteSystemA{}
    systemB := &ConcreteSystemB{}

    // 创建双向适配器
    adapter := NewBidirectionalAdapter(systemA, systemB)

    // 当成 SystemA 用（内部调用 SystemB）
    var a SystemA = adapter
    resultA := a.MethodA("123")
    fmt.Println("Result A:", resultA)  // 输出 246（123 * 2）

    // 当成 SystemB 用（内部调用 SystemA）
    var b SystemB = adapter
    resultB := b.MethodB(456)
    fmt.Println("Result B:", resultB)  // 输出转换后的结果
}
```

---

## 六、避坑指南（血泪经验）

### ❌ 坑 1：适配器做了太多事

```go
type BadAdapter struct{}

func (b *BadAdapter) Pay(order Order) PaymentResult {
    // ❌ 别在适配器里做业务逻辑！
    if order.Amount <= 0 {
        return PaymentResult{Success: false, Message: "金额不能为 0"}
    }

    if order.Amount > 10000 {
        return PaymentResult{Success: false, Message: "金额超限"}
    }

    // ❌ 别在适配器里访问数据库！
    customer := db.GetCustomer(order.CustomerID)

    // ❌ 别在适配器里发邮件！
    email.Send(order.CustomerID, "支付成功")

    // 适配器只应该做接口转换，不该做其他事！
}
```

**适配器只负责"接口翻译"，别让它干杂活。**

- 业务逻辑：在适配器之前或之后处理
- 数据访问：在业务层处理
- 副作用（发邮件、发通知）：在业务层处理

### ❌ 坏 2：适配器返回错误时信息丢失

```go
func (p *PayPalAdapter) Pay(order Order) PaymentResult {
    result, err := p.legacy.ProcessPayment(order.ID, order.Amount)
    if err != nil {
        // ❌ 别丢掉原始错误！
        return PaymentResult{Success: false, Message: "支付失败"}
    }
}

// ✅ 正确：保留原始错误
func (p *PayPalAdapter) Pay(order Order) PaymentResult {
    result, err := p.legacy.ProcessPayment(order.ID, order.Amount)
    if err != nil {
        return PaymentResult{
            Success: false,
            Message: fmt.Sprintf("支付失败: %v", err),
        }
    }
}
```

### ❌ 坑 3：过度适配

**简单的场景直接用就行，别搞适配器。**

```go
// ❌ 别为了一两个不同的参数就搞适配器
type StringToIntAdapter struct{}

func (s *StringToIntAdapter) Convert(str string) int {
    num, _ := strconv.Atoi(str)
    return num
}

// ✅ 直接转换
num, err := strconv.Atoi(str)
```

---

## 七、总结

**适配器模式的核心价值：**

1. **解耦系统**：新老系统解耦，互不影响
2. **不修改原接口**：遵循开闭原则，对扩展开放
3. **统一接口**：多个不同系统能用同一套接口调用
4. **渐进迁移**：可以慢慢迁移老系统，不影响新系统

**适用场景：**

- ✅ 集成第三方系统：支付、物流、短信等第三方服务
- ✅ 接口不兼容：两个系统接口不兼容但需要协同工作
- ✅ 遗留系统改造：把老系统接入新架构
- ✅ 重构：重构时用适配器让新旧代码共存

**不适用场景：**

- ❌ 接口本来就很兼容：直接用就行
- ❌ 简单的参数转换：直接转，别过度设计
- ❌ 性能敏感场景：适配器有转换开销

---

**最后送你一句：**

> "适配器模式就像旅行插头转换器，让你的设备在全世界都能充电。但别为了个 USB 接口就买个转换器，直接插进去就行。"

**下期预告：代理模式——别让大对象到处跑，学会这招让你的系统性能直接起飞！**