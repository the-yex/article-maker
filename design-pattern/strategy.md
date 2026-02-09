# Go 写成这样，难怪你天天改需求：if-else 地狱的终极解法

产品经理在群里丢下一句话：“下周加个新促销活动吧。”

你回了句：“行，很快。”

但你心里其实在想：“千万别又动那个函数……”

然后你打开代码，看到了这个像长寿面一样的函数：

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
    }
    ......
}
```

更绝的是：

同样的 if-else，在三个地方各有一份：

```go
// 订单服务
func ValidatePromotion(promotionType string) bool { ... }

// 支付服务
func GetPromotionDescription(promotionType string) string { ... }

// 营销服务
func CalculateTax(price float64, promotionType string) float64 { ... }
```

你突然意识到一件事：



> ❌ 改一次规则，要改 3 个服务
> ❌ 漏改一个，线上就是真金白银
> ❌ 需求越多，这坨代码越肥

今天不讲废话，我们直接拆解策略模式，看看怎么把这种"多算法切换"的场景处理得优雅又扩展。

---

## 问题根本不在 if-else，而在“你把选择和算法写在了一起”

现在这段代码，本质在做两件事：

1. 决定用哪种优惠方式
2. 执行具体优惠算法

但你把它们写成了这样：

```
选策略 + 算价格 = 写在一个函数里
```

结果就是：

- 策略一多，函数爆炸

- 改算法，所有地方跟着改

- 想测试一个策略？先跑完整套 if-else

这时候，你真正需要的不是“少写几个 else”，

而是：**把“策略”本身拆出来。**

这正是：**策略模式（Strategy Pattern）**

---

## 策略模式一句话解释

> **把一堆“不同算法”封装成独立策略，运行时选择其中一个执行。**

就像导航软件：

- 最快路线
- 最短距离
- 避开收费
- 避开拥堵

你只说一句： “给我最快的。”

至于怎么算，是它的事，不是你的事。

**算法怎么实现 ≠ 谁来决定用哪个算法**

## 先看反面教材：if-else 套娃

```go
func ProcessOrder(order Order) float64 {
    if order.PromotionType == "percent_discount" {
        return order.BasePrice * 0.8
    } else if order.PromotionType == "fixed_discount" {
        return order.BasePrice - 50
    } else if order.PromotionType == "buy_2_get_1" {
        return order.BasePrice * 2 / 3
    }
    // ...
}
```

这种代码的最终形态一定是：

1. **if-else 爆炸**：6 种策略 × 3 个服务 = 18 个 if-else
2. **重复代码**：同样的逻辑在多个地方复制
3. **难以扩展**：加一个新策略要改 3 个文件
4. **难以测试**：想测一个规则，得带着所有规则一起测
5. **容易漏改**：改一个地方，另一个地方忘了

**这代码维护起来就像在玩打地鼠，一个刚改完，另一个又冒出来。**

## Go 版策略模式：把“算法”独立成对象

### 先定义统一接口：

```go
type PromotionStrategy interface {
    Calculate(basePrice float64) float64
    Description() string
}
```

然后每种优惠各写一个策略：

``````go
type PercentDiscountStrategy struct {
    rate float64
}

func (p *PercentDiscountStrategy) Calculate(basePrice float64) float64 {
    return basePrice * p.rate
}

func (p *PercentDiscountStrategy) Description() string {
    return "打折优惠"
}
``````

``````go
type FixedDiscountStrategy struct {
    amount float64
}

func (f *FixedDiscountStrategy) Calculate(basePrice float64) float64 {
    price := basePrice - f.amount
    if price < 0 {
        return 0
    }
    return price
}

func (f *FixedDiscountStrategy) Description() string {
    return "满减优惠"
}
``````

上下文只负责“用哪个策略”：

``````go
type PromotionContext struct {
    strategy PromotionStrategy
}

func (c *PromotionContext) SetStrategy(s PromotionStrategy) {
    c.strategy = s
}

func (c *PromotionContext) FinalPrice(price float64) float64 {
    return c.strategy.Calculate(price)
}
``````

使用时：

``````go
ctx := &PromotionContext{}
ctx.SetStrategy(&PercentDiscountStrategy{rate: 0.8})
fmt.Println(ctx.FinalPrice(1000))

ctx.SetStrategy(&FixedDiscountStrategy{amount: 50})
fmt.Println(ctx.FinalPrice(1000))
``````

此刻发生了什么变化？

-  if-else 消失

-  新策略 = 新 struct

- 老代码一行不用改

- 每个策略可独立测试

---

## 再进阶：用 map + 注册表彻底干掉分支

上面的实现还需要手动 `SetStrategy`，我们可以做得更智能。

```go
var registry = map[string]func() PromotionStrategy{}

func Register(name string, f func() PromotionStrategy) {
    registry[name] = f
}

func GetStrategy(name string) PromotionStrategy {
    return registry[name]()
}
```

注册策略：

``````go
func init() {
    Register("percent", func() PromotionStrategy {
        return &PercentDiscountStrategy{rate: 0.8}
    })
    Register("fixed", func() PromotionStrategy {
        return &FixedDiscountStrategy{amount: 50}
    })
}
``````

使用时：

``````go
s := GetStrategy(order.PromotionType)
ctx.SetStrategy(s)
price := ctx.FinalPrice(order.BasePrice)
``````

**新增策略时，只需要：**

1. 新建一个策略结构体
2. 在 `init()` 里注册一次
3. **其他代码一行都不用动！**

> **这才是真正的：对扩展开放，对修改关闭**


## 策略模式 ≠ 工厂模式，别被写法骗了

很多人会说：

> “你这个根据类型返回不同结构体，不就是工厂模式吗？”

表面看像，其实关注点完全不同。

**一句话区分：**

- 工厂模式：解决的是「创建哪个对象」
- 策略模式：解决的是「运行时用哪种算法」

用到优惠场景里：

- 工厂模式关心的是：

  percent 返回 PercentDiscountStrategy

   fixed 返回 FixedDiscountStrategy

- 策略模式关心的是：

  Calculate() 这一步怎么算

  上下文只调用接口，不关心具体实现

换句话说：

> 🏭 工厂：负责“选人”
> 🧠 策略：负责“这个人怎么干活”

在真实项目中，它们**经常一起用**：

``````go
strategy := Factory(order.PromotionType) // 工厂选策略
price := strategy.Calculate(order.Price) // 策略算价格
``````

所以你看到的往往是：

> **工厂模式 + 策略模式的组合写法**

而不是二选一。

**`只负责“创建对象”的，是工厂`**
**`真正负责“切换算法行为”的，才是策略`**

## 你每天都在用策略模式，只是没意识到

标准库里的：

``````go
sort.Slice(products, func(i, j int) bool {
    return products[i].Price < products[j].Price
})
``````

这个 `func(i, j int) bool`本质就是：排序策略

换个函数，就换一种排序方式。

**在 Go 里：函数就是最轻量的策略对象**




## 什么时候该用策略模式？

当你看到代码长这样：

``````go
if type == A { ... }
else if type == B { ... }
else if type == C { ... }
``````

并且这些分支是：

- 同一件事

- 不同做法

- 以后还会加

那基本可以确定：

> **你需要的是策略模式，不是更多的 else。**

## 最后送你一句

> **策略模式不是为了优雅，是为了活命。**
> 当 if-else 开始指数级增长时，它就是你的止血钳。

**下期预告：适配器模式——别为了接个老系统就把代码改得面目全非，学会这招让你的系统集成优雅又省事！**

