# 别再到处写 if-else 了！Go 策略模式让你的算法切换直接起飞

上周 code review 的时候，我看到了一段让我差点吐血的代码：

```go
func CalculatePrice(basePrice float64, promotionType string) float64 {
    if promotionType == "percent_discount" {
        return basePrice * 0.8  // 8 折
    } else if promotionType == "fixed_discount" {
        return basePrice - 50  // 减 50
    } else if promotionType == "buy_2_get_1" {
        return basePrice * 2 / 3  // 买二送一
    } else if promotionType == "vip_discount" {
        return basePrice * 0.7  // VIP 7 折
    } else if promotionType == "flash_sale" {
        return basePrice * 0.5  // 闪购 5 折
    } else if promotionType == "black_friday" {
        return basePrice * 0.6  // 黑五 6 折
    } else {
        return basePrice
    }
}
```

**这代码写得跟意大利面一样，if-else 套了一堆，每加一个促销活动就要加一个分支。**

更坑的是，同样的 if-else 逻辑在其他地方又出现了一遍：

```go
// 订单服务里也有一遍
func ValidatePromotion(promotionType string) bool {
    if promotionType == "percent_discount" {
        return true
    } else if promotionType == "fixed_discount" {
        return true
    } // ... 又复制了一堆
}

// 支付服务里也有一遍
func GetPromotionDescription(promotionType string) string {
    if promotionType == "percent_discount" {
        return "8折优惠"
    } else if promotionType == "fixed_discount" {
        return "立减50元"
    } // ... 又复制了一堆
}
```

今天不讲废话，我们直接拆解策略模式，看看怎么把这种"多算法切换"的场景处理得优雅又扩展。

---

## 一、策略模式到底是干嘛的？

用一句话说清楚：**把不同的算法封装成独立的策略，运行时可以动态切换。**

看个生活例子：

```
导航软件：选择路线

策略 1：最短时间 -> 走高速，快但收费
策略 2：最短距离 -> 走小路，近但慢
策略 3：避开收费 -> 全走国道，免费但远
策略 4：避免拥堵 -> 实时避堵，路线动态变化

你不需要知道：
- 每个策略内部怎么计算
- 地图数据怎么查询
- 路线怎么规划

你只需要说："我要最快的路线"
策略模式自动给你最好的方案
```

**这就是策略模式的核心：算法与调用方解耦，可以随时切换。**

---

## 二、不用策略模式会有多恶心？

假设你有 6 种促销策略：
- 百分比折扣
- 固定金额减免
- 买二送一
- VIP 专享
- 闪购
- 黑五特惠

### ❌ 反面教材：if-else 套娃

```go
// 电商服务
func ProcessOrder(order Order) float64 {
    finalPrice := order.BasePrice

    if order.PromotionType == "percent_discount" {
        finalPrice = order.BasePrice * 0.8
    } else if order.PromotionType == "fixed_discount" {
        finalPrice = order.BasePrice - 50
    } else if order.PromotionType == "buy_2_get_1" {
        finalPrice = order.BasePrice * 2 / 3
    } else if order.PromotionType == "vip_discount" {
        finalPrice = order.BasePrice * 0.7
    } else if order.PromotionType == "flash_sale" {
        finalPrice = order.BasePrice * 0.5
    } else if order.PromotionType == "black_friday" {
        finalPrice = order.BasePrice * 0.6
    }

    return finalPrice
}

// 营销服务（又复制了一遍！）
func GetPromotionInfo(promotionType string) PromotionInfo {
    if promotionType == "percent_discount" {
        return PromotionInfo{Name: "8折优惠", Type: "discount"}
    } else if promotionType == "fixed_discount" {
        return PromotionInfo{Name: "立减50元", Type: "fixed"}
    } // ... 又复制了一遍
}

// 支付服务（又复制了一遍！）
func CalculateTax(price float64, promotionType string) float64 {
    if promotionType == "percent_discount" {
        return price * 0.08  // 折扣后的税
    } else if promotionType == "fixed_discount" {
        return (price - 50) * 0.08
    } // ... 又复制了一遍
}
```

**这代码的毛病：**

1. **if-else 爆炸**：6 种策略 × 3 个服务 = 18 个 if-else
2. **重复代码**：同样的逻辑在多个地方复制
3. **难以扩展**：加一个新策略要改 3 个文件
4. **难以测试**：想测一个策略，得把其他策略都测一遍
5. **容易漏改**：改一个策略逻辑，可能漏改某个服务

**这代码维护起来就像就像在玩打地鼠，打了一个又冒出来一个。**

---

## 三、Go 版策略模式实现

### 基础实现

```go
// ========== 策略接口 ==========
type PromotionStrategy interface {
    Calculate(basePrice float64) float64
    GetName() string
    GetDescription() string
}

// ========== 具体策略 ==========
type PercentDiscountStrategy struct {
    rate float64  // 折扣率，0.8 表示 8 折
}

func (p *PercentDiscountStrategy) Calculate(basePrice float64) float64 {
    return basePrice * p.rate
}

func (p *PercentDiscountStrategy) GetName() string {
    return "percent_discount"
}

func (p *PercentDiscountStrategy) GetDescription() string {
    return fmt.Sprintf("%.0f折优惠", p.rate*10)
}

type FixedDiscountStrategy struct {
    amount float64  // 减免金额
}

func (f *FixedDiscountStrategy) Calculate(basePrice float64) float64 {
    price := basePrice - f.amount
    if price < 0 {
        price = 0
    }
    return price
}

func (f *FixedDiscountStrategy) GetName() string {
    return "fixed_discount"
}

func (f *FixedDiscountStrategy) GetDescription() string {
    return fmt.Sprintf("立减%.0f元", f.amount)
}

type Buy2Get1Strategy struct{}

func (b *Buy2Get1Strategy) Calculate(basePrice float64) float64 {
    return basePrice * 2 / 3  // 买二送一，相当于 3 件钱买 4 件
}

func (b *Buy2Get1Strategy) GetName() string {
    return "buy_2_get_1"
}

func (b *Buy2Get1Strategy) GetDescription() string {
    return "买二送一"
}

type VIPDiscountStrategy struct {
    rate float64
}

func (v *VIPDiscountStrategy) Calculate(basePrice float64) float64 {
    return basePrice * v.rate
}

func (v *VIPDiscountStrategy) GetName() string {
    return "vip_discount"
}

func (v *VIPDiscountStrategy) GetDescription() string {
    return "VIP专享优惠"
}

type FlashSaleStrategy struct {
    rate float64
}

func (f *FlashSaleStrategy) Calculate(basePrice float64) float64 {
    return basePrice * f.rate
}

func (f *FlashSaleStrategy) GetName() string {
    return "flash_sale"
}

func (f *FlashSaleStrategy) GetDescription() string {
    return "限时闪购"
}

// ========== 上下文（使用策略） ==========
type PromotionContext struct {
    strategy PromotionStrategy
}

func NewPromotionContext(strategy PromotionStrategy) *PromotionContext {
    return &PromotionContext{strategy: strategy}
}

func (c *PromotionContext) SetStrategy(strategy PromotionStrategy) {
    c.strategy = strategy
}

func (c *PromotionContext) CalculatePrice(basePrice float64) float64 {
    if c.strategy == nil {
        return basePrice
    }
    return c.strategy.Calculate(basePrice)
}

func (c *PromotionContext) GetPromotionName() string {
    if c.strategy == nil {
        return "无优惠"
    }
    return c.strategy.GetName()
}

func (c *PromotionContext) GetPromotionDescription() string {
    if c.strategy == nil {
        return "无优惠"
    }
    return c.strategy.GetDescription()
}

// ========== 使用 ==========
func main() {
    // 创建上下文
    ctx := NewPromotionContext(&PercentDiscountStrategy{rate: 0.8})

    // 计算价格
    price := ctx.CalculatePrice(1000)
    fmt.Printf("原价: 1000, %s, 实付: %.2f\n",
        ctx.GetPromotionDescription(), price)

    // 切换策略
    ctx.SetStrategy(&FixedDiscountStrategy{amount: 50})
    price = ctx.CalculatePrice(1000)
    fmt.Printf("原价: 1000, %s, 实付: %.2f\n",
        ctx.GetPromotionDescription(), price)

    // 切换到 VIP 策略
    ctx.SetStrategy(&VIPDiscountStrategy{rate: 0.7})
    price = ctx.CalculatePrice(1000)
    fmt.Printf("原价: 1000, %s, 实付: %.2f\n",
        ctx.GetPromotionDescription(), price)
}
```

**输出：**

```
原价: 1000, 8折优惠, 实付: 800.00
原价: 1000, 立¥50元, 实付: 950.00
原价: 1000, VIP专享优惠, 实付: 700.00
```

---

## 四、进阶版：策略工厂 + 注册器

上面的实现还需要手动 `SetStrategy`，我们可以做得更智能。

```go
// ========== 策略注册器 ==========
type StrategyFactory func() PromotionStrategy

var strategyRegistry = make(map[string]StrategyFactory)

func RegisterStrategy(name string, factory StrategyFactory) {
    strategyRegistry[name] = factory
}

func GetStrategy(name string) (PromotionStrategy, error) {
    factory, exists := strategyRegistry[name]
    if !exists {
        return nil, fmt.Errorf("不支持的促销策略: %s", name)
    }
    return factory(), nil
}

// ========== 在每个策略包的 init() 里注册 ==========

func init() {
    RegisterStrategy("percent_discount", func() PromotionStrategy {
        return &PercentDiscountStrategy{rate: 0.8}
    })

    RegisterStrategy("fixed_discount", func() PromotionStrategy {
        return &FixedDiscountStrategy{amount: 50}
    })

    RegisterStrategy("buy_2_get_1", func() PromotionStrategy {
        return &Buy2Get1Strategy{}
    })

    RegisterStrategy("vip_discount", func() PromotionStrategy {
        return &VIPDiscountStrategy{rate: 0.7}
    })

    RegisterStrategy("flash_sale", func() PromotionStrategy {
        return &FlashSaleStrategy{rate: 0.5}
    })
}

// ========== 使用：更简单 ==========
func ProcessOrder(order Order) (float64, error) {
    strategy, err := GetStrategy(order.PromotionType)
    if err != nil {
        return order.BasePrice, err
    }

    ctx := NewPromotionContext(strategy)
    finalPrice := ctx.CalculatePrice(order.BasePrice)

    return finalPrice, nil
}
```

**新增策略时，只需要：**

1. 新建一个策略结构体
2. 在 `init()` 里注册一次
3. **其他代码一行都不用动！**

---

## 五、实战案例：排序策略

标准库的 `sort.Slice` 其实就是策略模式的一个应用。

```go
type Product struct {
    ID    int
    Name  string
    Price float64
}

func main() {
    products := []Product{
        {ID: 1, Name: "iPhone", Price: 999},
        {ID: 2, Name: "MacBook", Price: 1999},
        {ID: 3, Name: "AirPods", Price: 199},
    }

    // 策略 1：按价格排序
    sort.Slice(products, func(i, j int) bool {
        return products[i].Price < products[j].Price
    })
    fmt.Println("按价格排序:", products)

    // 策略 2：按 ID 排序
    sort.Slice(products, func(i, j int) bool {
        return products[i].ID < products[j].ID
    })
    fmt.Println("按 ID 排序:", products)

    // 策略 3：按名称排序
    sort.Slice(products, func(i, j int) bool {
        return products[i].Name < products[j].Name
    })
    fmt.Println("按名称排序: ", products)
}
```

**闭包函数就是策略，传不同的函数就执行不同的排序逻辑。**

---

## 六、实战案例：压缩策略

假设你要支持多种压缩算法：Gzip、Snappy、LZ4。

```go
// ========== 压缩策略接口 ==========
type CompressionStrategy interface {
    Compress(data []byte) ([]byte, error)
    Decompress(data []byte) ([]byte, error)
    Extension() string
}

// ========== Gzip 压缩 ==========
type GzipCompression struct{}

func (g *GzipCompression) Compress(data []byte) ([]byte, error) {
    var buf bytes.Buffer
    writer := gzip.NewWriter(&buf)
    if _, err := writer.Write(data); err != nil {
        return nil, err
    }
    if err := writer.Close(); err != nil {
        return nil, err
    }
    return buf.Bytes(), nil
}

func (g *GzipCompression) Decompress(data []byte) ([]byte, error) {
    reader, err := gzip.NewReader(bytes.NewReader(data))
    if err != nil {
        return nil, err
    }
    defer reader.Close()
    return io.ReadAll(reader)
}

func (g *GzipCompression) Extension() string {
    return ".gz"
}

// ========== Snappy 压缩 ==========
type SnappyCompression struct{}

func (s *SnappyCompression) Compress(data []byte) ([]byte, error) {
    return snappy.Encode(nil, data), nil
}

func (s *SnappyCompression) Decompress(data []byte) ([]byte, error) {
    decoded, err := snappy.Decode(nil, data)
    if err != nil {
        return nil, err
    }
    return decoded, nil
}

func (s *SnappyCompression) Extension() string {
    return ".snappy"
}

// ========== 使用 ==========
type CompressionContext struct {
    strategy CompressionStrategy
}

func NewCompressionContext(strategy CompressionStrategy) *CompressionContext {
    return &CompressionContext{strategy: strategy}
}

func (c *CompressionContext) CompressFile(filename string) error {
    data, err := os.ReadFile(filename)
    if err != nil {
        return err
    }

    compressed, err := c.strategy.Compress(data)
    if err != nil {
        return err
    }

    outputFilename := filename + c.strategy.Extension()
    return os.WriteFile(outputFilename, compressed, 0644)
}

// 根据场景选择策略
func GetCompressionStrategy(compressionType string) CompressionStrategy {
    switch compressionType {
    case "gzip":
        return &GzipCompression{}
    case "snappy":
        return &SnappyCompression{}
    default:
        return &GzipCompression{}  // 默认
    }
}

func main() {
    // 快速压缩用 Snappy
    ctx := NewCompressionContext(GetCompressionStrategy("snappy"))
    ctx.CompressFile("data.txt")

    // 高压缩率用 Gzip
    ctx = NewCompressionContext(GetCompressionStrategy("gzip"))
    ctx.CompressFile("log.txt")
}
```

---

## 七、策略模式 vs 状态模式

很多人会混淆这两个模式。

| 对比项 | 策略模式 | 状态模式 |
|--------|---------|---------|
| 用途 | 算法切换 | 状态切换 |
| 切换时机 | 客户端决定 | 对象内部决定 |
| 关注点 | 算法逻辑 | 状态逻辑 |
| 策略/状态之间关系 | 互相独立 | 有转移关系 |

**策略模式：** 不同的促销策略之间没有关系，可以随时切换。

**状态模式：** 订单状态之间有转移关系，已支付才能发货，不能跳跃。

---

## 八、避坑指南（血泪经验）

### ❌ 坑 1：策略之间有依赖关系

```go
// ❌ 策略之间不应该互相依赖
type StrategyA struct {
    nextStrategy *StrategyB  // ❌ 别这样！
}

func (s *StrategyA) Execute() {
    // ...
    s.nextStrategy.Execute()  // ❌ 策略不应该调用其他策略
}
```

**策略之间应该是独立的，由 Context 决定怎么用。**

### ❌ 坏 2：策略切换太频繁

```go
func ProcessOrders(orders []Order) {
    for _, order := range orders {
        // ❌ 每次都切换策略，开销太大
        ctx.SetStrategy(GetStrategy(order.PromotionType))
        ctx.Process(order)
    }
}
```

**解决：批量处理相同策略的订单。**

```go
func ProcessOrders(orders []Order) {
    // 按策略分组
    groups := make(map[string][]Order)
    for _, order := range orders {
        groups[order.PromotionType] = append(groups[order.PromotionType], order)
    }

    // 每组用一个策略
    for promotionType, groupOrders := range groups {
        ctx.SetStrategy(GetStrategy(promotionType))
        for _, order := range groupOrders {
            ctx.Process(order)
        }
    }
}
```

### ❌ 坏 3：策略和 Context 耦合太紧

```go
// ❌ 策略不应该依赖 Context
type BadStrategy struct {
    ctx *PromotionContext  // ❌ 别这样！
}

func (s *BadStrategy) Execute() {
    // s.ctx.DoSomething()  // ❌ 策略不应该调用 Context
}
```

**策略应该只接收需要的参数，不依赖 Context。**

---

## 九、总结

**策略模式的核心价值：**

1. **算法解耦**：算法与调用方解耦，可以独立演化
2. **易于扩展**：新增策略不影响现有代码
3. **运行时切换**：可以动态切换策略
4. **避免 if-else**：大量 if-else 可以用策略替代

**适用场景：**

- ✅ 多种算法实现：排序、压缩、加密
- ✅ 业务规则变化：促销策略、定价策略
- ✅ 数据验证：不同场景的不同验证规则
- ✅ 算法性能对比：切换不同算法测试性能

**不适用场景：**

- ❌ 只有一个算法：直接写就行，别过度设计
- ❌ 策略之间有依赖关系：用状态模式
- ❌ 策略切换代价太大：用其他方式

---

**最后送你一句：**

> "策略模式就像瑞士军刀，不同的刀对应不同的场景。但别一次性把所有刀都打开，手会被划伤的。"

**下期预告：适配器模式——别为了接个老系统就把代码改得面目全非，学会这招让你的系统集成优雅又省事！**