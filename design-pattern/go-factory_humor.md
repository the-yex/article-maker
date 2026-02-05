# 别再到处 new 对象了！Go 工厂模式让你的代码优雅度直接起飞

上周 code review 的时候，我看到了一段让我头皮发麻的代码：

```go
func CreatePaymentProcessor(provider string) PaymentProcessor {
    if provider == "stripe" {
        return &StripeProcessor{
            apiKey: "sk_xxx",
            client: &http.Client{},
        }
    } else if provider == "paypal" {
        return &PayPalProcessor{
            clientId: "client_xxx",
            secret:   "secret_xxx",
            timeout:  30 * time.Second,
        }
    } else if provider == "alipay" {
        return &AlipayProcessor{
            appId:     "app_xxx",
            privateKey: loadPrivateKey(),
        }
    }
    return nil
}
```

**这代码写得跟屎山一样，到处都是重复的创建逻辑，改一个支付方式要改三四个地方。**

今天不讲废话，我们直接拆解工厂模式，看看怎么把这段代码改得优雅又好维护。

---

## 一、工厂模式到底是干嘛的？

用一句话说清楚：**把"创建对象"这件事封装起来，让调用方不用关心对象是怎么 new 出来的。**

看个生活例子：

```
你："给我来辆车"
4S店：(后台操作) -> 新车开过来了

你不需要关心：
- 车是怎么装配的
- 发动机型号是什么
- 用了哪些零件
你只关心：拿到能开的车
```

**这就是工厂模式的思想：调用方说"我要啥"，工厂负责"怎么造"。**

---

## 二、不用工厂模式会有多恶心？

假设你要接入三个支付渠道：Stripe、PayPal、支付宝。

### ❌ 反面教材：到处都是创建逻辑

```go
// 支付服务 A 处
func ProcessOrderA(order Order) {
    var processor PaymentProcessor
    if order.PaymentMethod == "stripe" {
        processor = &StripeProcessor{
            apiKey:   "sk_xxx",
            endpoint: "https://api.stripe.com",
            client:   &http.Client{Timeout: 30 * time.Second},
        }
    } else if order.PaymentMethod == "paypal" {
        processor = &PayPalProcessor{
            clientId: "client_xxx",
            secret:   "secret_xxx",
            timeout:  30 * time.Second,
        }
    }
    // ... 处理支付
}

// 支付服务 B 处（重复的创建！）
func ProcessOrderB(order Order) {
    var processor PaymentProcessor
    if order.PaymentMethod == "stripe" {
        processor = &StripeProcessor{ /* 重复的配置 */ }
    } else if order.PaymentMethod == "paypal" {
        processor = &PayPalProcessor{ /* 重复的配置 */ }
    }
    // ... 处理支付
}
```

**这代码的毛病：**

1. **重复代码满天飞**：同样的创建逻辑在好几个地方复制粘贴
2. **改一个配置要改 N 个地方**：Stripe 换了个 API Key，你得改三四个文件
3. **难以测试**：想 mock 某个支付方式，得把创建逻辑也一起改
4. **违反开闭原则**：新增一个支付方式，得改所有调用处

**这代码维护起来就像是在拆炸弹，改一处就要担心别的地方会不会炸。**

---

## 三、简单工厂模式：先把屎山扫了

最简单的改进：把创建逻辑抽到一个函数里。

```go
type PaymentProcessor interface {
    Process(amount float64) string
}

type StripeProcessor struct {
    apiKey   string
    endpoint string
    client   *http.Client
}

func (s *StripeProcessor) Process(amount float64) string {
    return fmt.Sprintf("Stripe 支付: %.2f", amount)
}

type PayPalProcessor struct {
    clientId string
    secret   string
    timeout  time.Duration
}

func (p *PayPalProcessor) Process(amount float64) string {
    return fmt.Sprintf("PayPal 支付: %.2f", amount)
}

// ========== 简单工厂 ==========
func NewPaymentProcessor(provider string) (PaymentProcessor, error) {
    switch provider {
    case "stripe":
        return &StripeProcessor{
            apiKey:   os.Getenv("STRIPE_API_KEY"),
            endpoint: "https://api.stripe.com/v1",
            client:   &http.Client{Timeout: 30 * time.Second},
        }, nil
    case "paypal":
        return &PayPalProcessor{
            clientId: os.Getenv("PAYPAL_CLIENT_ID"),
            secret:   os.Getenv("PAYPAL_SECRET"),
            timeout:  30 * time.Second,
        }, nil
    default:
        return nil, fmt.Errorf("不支持的支付方式: %s", provider)
    }
}

// ========== 调用方清爽多了 ==========
func ProcessOrder(order Order) error {
    processor, err := NewPaymentProcessor(order.PaymentMethod)
    if err != nil {
        return err
    }

    result := processor.Process(order.Amount)
    fmt.Println(result)
    return nil
}
```

**改进点：**

1. ✅ 创建逻辑集中管理，改配置只改一个地方
2. ✅ 调用方代码简洁，不用关心对象怎么创建
3. ✅ 易于测试，可以 mock 工厂函数

**但是还有一个问题：**

```go
// 新增支付方式还是得改这个 switch
func NewPaymentProcessor(provider string) (PaymentProcessor, error) {
    switch provider {
    case "stripe":
        // ...
    case "paypal":
        // ...
    case "alipay":  // 新增！要改这里
        return &AlipayProcessor{ /* ... */ }, nil
    default:
        return nil, fmt.Errorf("不支持的支付方式: %s", provider)
    }
}
```

**每新增一个支付方式，都要改这个函数，违反开闭原则。**

---

## 四、工厂方法模式：让扩展不用改代码

**核心思想：让每个支付方式自己管自己的创建，工厂只负责"调用"。**

### 方案 1：注册器模式（推荐）

```go
// ========== 全局注册器 ==========
type ProcessorFactory func() PaymentProcessor

var processorFactories = make(map[string]ProcessorFactory)

// 注册支付方式
func RegisterPaymentProcessor(name string, factory ProcessorFactory) {
    processorFactories[name] = factory
}

// 创建支付处理器
func NewPaymentProcessor(provider string) (PaymentProcessor, error) {
    factory, exists := processorFactories[provider]
    if !exists {
        return nil, fmt.Errorf("不支持的支付方式: %s", provider)
    }
    return factory(), nil
}

// ========== 各个支付方式的包自行注册 ==========

// stripe.go
func init() {
    RegisterPaymentProcessor("stripe", func() PaymentProcessor {
        return &StripeProcessor{
            apiKey:   os.Getenv("STRIPE_API_KEY"),
            endpoint: "https://api.stripe.com/v1",
            client:   &http.Client{Timeout: 30 * time.Second},
        }
    })
}

// paypal.go
func init() {
    RegisterPaymentProcessor("paypal", func() PaymentProcessor {
        return &PayPalProcessor{
            clientId: os.Getenv("PAYPAL_CLIENT_ID"),
            secret:   os.Getenv("PAYPAL_SECRET"),
            timeout:  30 * time.Second,
        }
    })
}

// alipay.go
func init() {
    RegisterPaymentProcessor("alipay", func() PaymentProcessor {
        return &AlipayProcessor{
            appId:     os.Getenv("ALIPAY_APP_ID"),
            privateKey: loadPrivateKey(),
        }
    })
}
```

**这方案绝了！**

新增支付方式时：

1. 新建一个 `wechatpay.go` 文件
2. 在 `init()` 里注册一次
3. **主工厂代码一行都不用动！**

```go
// wechatpay.go - 新增支付方式，其他代码不用改！
func init() {
    RegisterPaymentProcessor("wechatpay", func() PaymentProcessor {
        return &WeChatPayProcessor{
            appId:  os.Getenv("WECHAT_APP_ID"),
            mchId:  os.Getenv("WECHAT_MCH_ID"),
            apiKey: os.Getenv("WECHAT_API_KEY"),
        }
    })
}
```

**这才叫符合开闭原则！**

---

## 五、抽象工厂模式：产品族批量创建

有时候你不止要创建一个对象，要创建一整套"产品族"。

比如：你要创建一套 UI 组件，有按钮、输入框、下拉框... 但要支持多套主题（暗色、亮色、复古）。

```go
// ========== 抽象产品接口 ==========
type Button interface {
    Render() string
}

type InputBox interface {
    Render() string
}

// ========== 抽象工厂 ==========
type UIFactory interface {
    CreateButton() Button
    CreateInputBox() InputBox
}

// ========== 暗色主题工厂 ==========
type DarkThemeFactory struct{}

func (f *DarkThemeFactory) CreateButton() Button {
    return &DarkButton{color: "#333"}
}

func (f *DarkThemeFactory) CreateInputBox() InputBox {
    return &DarkInputBox{bgColor: "#222", textColor: "#fff"}
}

// ========== 亮色主题工厂 ==========
type LightThemeFactory struct{}

func (f *LightThemeFactory) CreateButton() Button {
    return &LightButton{color: "#fff"}
}

func (f *LightThemeFactory) CreateInputBox() InputBox {
    return &LightInputBox{bgColor: "#f5f5f5", textColor: "#333"}
}

// ========== 使用 ==========
func CreateUI(theme string) (Button, InputBox) {
    var factory UIFactory

    switch theme {
    case "dark":
        factory = &DarkThemeFactory{}
    case "light":
        factory = &LightThemeFactory{}
    default:
        factory = &LightThemeFactory{}
    }

    button := factory.CreateButton()
    input := factory.CreateInputBox()

    return button, input
}

// 调用
button, input := CreateUI("dark")
fmt.Println(button.Render())   // DarkButton 渲染
fmt.Println(input.Render())    // DarkInputBox 渲染
```

**抽象工厂的价值：保证创建的整套组件风格一致。**

---

## 六、工厂模式在微服务架构中的应用

### 场景：多租户 SaaS 系统

不同客户可能用不同的数据库、不同的缓存、不同的消息队列。

```go
// ========== 基础设施工厂 ==========
type InfraFactory interface {
    CreateDatabase() Database
    CreateCache() Cache
    CreateMessageQueue() MessageQueue
}

// ========== AWS 方案 ==========
type AWSInfraFactory struct{}

func (f *AWSInfraFactory) CreateDatabase() Database {
    return &RDS{endpoint: "rds.amazonaws.com"}
}

func (f *AWSInfraFactory) CreateCache() Cache {
    return &ElastiCache{endpoint: "elasticache.amazonaws.com"}
}

func (f *AWSInfraFactory) CreateMessageQueue() MessageQueue {
    return &SQS{queueURL: "sqs.amazonaws.com"}
}

// ========== Azure 方案 ==========
type AzureInfraFactory struct{}

func (f *AzureInfraFactory) CreateDatabase() Database {
    return &AzureSQL{endpoint: "azure.microsoft.com"}
}

func (f *AzureInfraFactory) CreateCache() Cache {
    return &AzureRedis{endpoint: "redis.azure.com"}
}

func (f *AzureInfraFactory) CreateMessageQueue() MessageQueue {
    return &AzureServiceBus{endpoint: "servicebus.azure.com"}
}

// ========== 根据客户配置创建 ==========
func CreateInfrastructure(tenant Tenant) (*Infrastructure, error) {
    var factory InfraFactory

    switch tenant.CloudProvider {
    case "aws":
        factory = &AWSInfraFactory{}
    case "azure":
        factory = &AzureInfraFactory{}
    default:
        return nil, fmt.Errorf("不支持的云服务商")
    }

    return &Infrastructure{
        DB:      factory.CreateDatabase(),
        Cache:   factory.CreateCache(),
        Queue:   factory.CreateMessageQueue(),
    }, nil
}
```

**扩展新的云服务商时，只加一个新工厂类，主逻辑不用动。**

---

## 七、避坑指南（血泪经验）

### ❌ 坑 1：工厂做了太多事

```go
func NewPaymentProcessor(provider string) (PaymentProcessor, error) {
    // ❌ 不要在工厂里做这种事！
    _, err := http.Get("https://api.example.com/validate")
    if err != nil {
        return nil, err
    }

    switch provider {
    case "stripe":
        // ❌ 不要读取外部配置
        config, _ := loadConfigFromDB()

        // ❌ 不要建立长连接
        conn, _ := connectToRedis()

        return &StripeProcessor{ /* ... */ }, nil
    }
}
```

**工厂只负责"创建对象"，别让它干杂活。**

- 配置读取：依赖注入，别在工厂里读取
- 外部连接：延迟初始化，对象自己处理
- 业务逻辑：对象内部处理，别塞给工厂

### ✅ 正确姿势：

```go
func NewStripeProcessor(config StripeConfig) *StripeProcessor {
    return &StripeProcessor{
        apiKey: config.APIKey,
        // 只传配置，不初始化连接
    }
}

// 连接延迟初始化
func (s *StripeProcessor) ensureConnected() {
    if s.client == nil {
        s.client = &http.Client{Timeout: 30 * time.Second}
    }
}
```

### ❌ 坑 2：工厂返回 nil 而不是 error

```go
func NewPaymentProcessor(provider string) PaymentProcessor {
    if provider == "stripe" {
        return &StripeProcessor{ /* ... */ }
    }
    return nil  // ❌ 别这样！
}

// 调用方要到处检查 nil
processor := NewPaymentProcessor("unknown")
if processor == nil {  // ❌ 很容易忘记检查
    processor.Process(100)  // 💥 panic
}
```

**工厂创建失败应该返回 error，让调用方必须处理。**

```go
func NewPaymentProcessor(provider string) (PaymentProcessor, error) {
    switch provider {
    case "stripe":
        return &StripeProcessor{ /* ... */ }, nil
    default:
        return nil, fmt.Errorf("不支持的支付方式: %s", provider)
    }
}

// 调用方必须处理 error
processor, err := NewPaymentProcessor("unknown")
if err != nil {
    return err  // ✅ 必须处理
}
```

### ❌ 坑 3：过度设计

**简单场景用简单工厂，别瞎折腾。**

```go
// ❌ 别为了一两个对象就搞抽象工厂
type AnimalFactory interface {
    CreateDog() Dog
    CreateCat() Cat
}

// ✅ 直接用简单工厂
func NewAnimal(type string) Animal {
    switch type {
    case "dog": return &Dog{}
    case "cat": return &Cat{}
    }
}
```

---

## 八、总结

| 模式 | 复杂度 | 扩展性 | 适用场景
|------|--------|--------|----------
| 不用工厂 | 🟢 | 🔴 | 极少创建逻辑的简单对象
| 简单工厂 | 🟡 | 🟡 | 单一产品的创建，品种固定
| 工厂方法（注册器） | 🟡 | 🟢 | 多种产品，频繁扩展新品种
| 抽象工厂 | 🔴 | 🟢 | 需要创建整套产品族 |

**选择口诀：**

1. **简单直接**：一两种创建方式 → 简单工厂
2. **频繁扩展**：经常要加新品种 → 工厂方法+注册器
3. **成套产品**：创建一整套相关对象 → 抽象工厂

---

**最后送你一句：**

> "写代码就像做饭，简单工厂是预制菜，工厂方法是厨师现场做，抽象工厂是订制宴席。别用订制宴席的方式煮泡面。"

**下期预告：观察者模式——事件驱动架构的基石，学会这招让你的系统解耦度直接起飞！**