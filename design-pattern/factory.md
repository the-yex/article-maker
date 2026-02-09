# **Go 工厂模式实战：从屎山到优雅**
> 从屎山代码到优雅扩展，只差一个设计模式

上周修复老项目Bug的时候，我看到一段代码，**直接让我头皮发麻**

支付模块里是这样的(简化样例)：

```go
func CreatePaymentProcessor(provider string) PaymentProcessor {
    if provider == "stripe" {
        return &StripeProcessor{ /* 一堆配置 */ }
    } else if provider == "paypal" {
        return &PayPalProcessor{ /* 一堆配置 */ }
    } else if provider == "alipay" {
        return &AlipayProcessor{ /* 一堆配置 */ }
    }
    return nil
}
```

**这代码写得跟屎山一样，到处都是重复的创建逻辑，改一个支付方式要改三四个地方。**

今天不讲废话，我们直接拆解工厂模式，看看怎么把这段代码改得优雅又好维护。

---

## 一、工厂模式到底在解决什么？

用一句话说就是：**把“怎么创建对象”这件事，从业务逻辑里剥离出去。**

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

**这就是工厂模式的思想：你只管说你要什么，工厂负责怎么造"。**

---

## 二、不用工厂模式会有多恶心？

假设你要接入三个支付渠道：Stripe、PayPal、支付宝。

### 反面教材：**创建逻辑到处飞**

```go
func ProcessOrderA(order Order) {
    var processor PaymentProcessor
    if order.PaymentMethod == "stripe" {
        processor = &StripeProcessor{ /* 配置 */ }
    } else if order.PaymentMethod == "paypal" {
        processor = &PayPalProcessor{ /* 配置 */ }
    }
}
```

另一个文件：

``````go
func ProcessOrderB(order Order) {
    var processor PaymentProcessor
    if order.PaymentMethod == "stripe" {
        processor = &StripeProcessor{ /* 重复 */ }
    } else if order.PaymentMethod == "paypal" {
        processor = &PayPalProcessor{ /* 重复 */ }
    }
}
``````

**这代码的毛病：**

- Stripe 换 API Key → 改 N 个地方
- 新增支付方式 → 改所有 if-else
- 想 mock Stripe → 改业务代码
- 可维护性指数：💀💀💀

典型的 **创建逻辑污染了业务逻辑。**

---

## 三、简单工厂模式：先把屎山扫了

先把创建逻辑集中起来。

```go
func NewPaymentProcessor(provider string) (PaymentProcessor, error) {
    switch provider {
    case "stripe":
        return &StripeProcessor{ /* 配置 */ }, nil
    case "paypal":
        return &PayPalProcessor{ /* 配置 */ }, nil
    default:
        return nil, fmt.Errorf("不支持的支付方式")
    }
}
```

调用方瞬间干净：

``````go
func ProcessOrder(order Order) error {
    processor, err := NewPaymentProcessor(order.PaymentMethod)
    if err != nil {
        return err
    }
    return processor.Process(order.Amount)
}
``````

**优点：**

- 创建逻辑集中管理
- 调用方不关心 new 细节
- 改配置只改一处

但是还有一个问题：**每新增一个支付方式，都要改switch，违反开闭原则。**

---

## 四、**进阶：工厂方法 + 注册器（工程级写法）**

**核心思想：让每个产品自己注册到工厂，而不是工厂认识所有产品。**

```go
type ProcessorFactory func() PaymentProcessor

var factories = make(map[string]ProcessorFactory)

func Register(name string, factory ProcessorFactory) {
    factories[name] = factory
}

func NewPaymentProcessor(provider string) (PaymentProcessor, error) {
    factory, ok := factories[provider]
    if !ok {
        return nil, fmt.Errorf("不支持的支付方式")
    }
    return factory(), nil
}
```

### **每个支付方式自己注册：**

``````go
func init() {
    Register("stripe", func() PaymentProcessor {
        return &StripeProcessor{ /* 配置 */ }
    })
}
``````

``````go
func init() {
    Register("paypal", func() PaymentProcessor {
        return &PayPalProcessor{ /* 配置 */ }
    })
}
``````

### **新增微信支付？**

``````go
func init() {
    Register("wechat", func() PaymentProcessor {
        return &WeChatPayProcessor{ /* 配置 */ }
    })
}
``````

**主工厂代码：0 修改。**

**这才叫符合开闭原则！**

> 这也是很多 Go 框架（如 database/sql、grpc、image 包）在用的扩展方式。

---

## 五、什么时候该用抽象工厂？

当你要创建的是“一整套对象”：

- 数据库
- 缓存
- 消息队列

```go
type InfraFactory interface {
    CreateDB() DB
    CreateCache() Cache
    CreateMQ() MQ
}
```

AWS / Azure 各自实现：

``````go
type AWSFactory struct{}
func (f *AWSFactory) CreateDB() DB { return &RDS{} }
func (f *AWSFactory) CreateCache() Cache { return &Redis{} }
func (f *AWSFactory) CreateMQ() MQ { return &SQS{} }
``````

**抽象工厂的价值：保证创建的整套组件风格一致,并且同一套基础设施来自同一厂商**

---

## 六、工厂模式三大坑

### 坑 1：工厂里干脏活

```go
func NewX() X {
    connectDB()
    callHTTP()
    readFile()
}
```

这样你会得到一个：

- 创建慢

- 难测试

- 难复用

的“巨型工厂函数”。

###  坑 2：工厂返回 nil 而不是 error

```go
func NewX() X { return nil }
```

**工厂创建失败应该返回 error，让调用方必须处理。**

```go
func NewX() (X, error)
```

###  坑 3：过度设计

只有 2 种对象？

别上抽象工厂

```go
// 别为了一两个对象就搞抽象工厂
type AnimalFactory interface {
    CreateDog() Dog
    CreateCat() Cat
}

//  直接用简单工厂
func NewAnimal(type string) Animal {
    switch type {
    case "dog": return &Dog{}
    case "cat": return &Cat{}
    }
}
```

---

## 七、选型口诀

| **场景** | **模式** |
| -------- | -------- |
| 少量对象 | 简单工厂 |
| 经常扩展 | 工厂方法 |
| 成套创建 | 抽象工厂 |

口诀：

> **少 → 简单工厂**
> **多 → 工厂方法**
> **成套 → 抽象工厂**

## 八、总结

你写 if-else 创建对象的时候：

- 业务逻辑被污染

- 可扩展性下降

- 单测困难

用工厂模式后：

- 创建逻辑集中

- 扩展不改旧代码

- 架构更清晰


---

**最后送你一句：**

> "写代码就像做饭，简单工厂是预制菜，工厂方法是现炒，抽象工厂是订制宴席。别用订制宴席的方式煮泡面。"

**下期预告：观察者模式——事件驱动架构的基石，学会这招让你的系统解耦度直接起飞！**