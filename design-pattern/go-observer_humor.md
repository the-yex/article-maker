# 事件驱动架构的基石！Go 观察者模式让你的系统解耦度直接起飞

上周老板提了个需求："用户下单后，要做这几件事：发邮件、发短信、更新积分、记录日志、发优惠券..."

我一看需求，差点把咖啡喷到屏幕上。

"这也太多了吧？我要是在下单服务里直接写这些逻辑，代码会臭到连我自己都认不出来。"

今天不讲废话，我们直接拆解观察者模式，看看怎么把这种"一对多"的场景处理得优雅又解耦。

---

## 一、先看看不用观察者模式会有多恶心

### ❌ 反面教材：面条式代码

```go
func PlaceOrder(order Order) error {
    // 1. 创建订单
    if err := db.Create(order); err != nil {
        return err
    }

    // 2. 发邮件
    if err := emailService.SendOrderEmail(order.UserEmail); err != nil {
        log.Error("发邮件失败", err)  // 邮件失败要不要回滚订单？
    }

    // 3. 发短信
    if err := smsService.SendOrderSMS(order.UserPhone); err != nil {
        log.Error("发短信失败", err)
    }

    // 4. 更新积分
    if err := pointsService.AddPoints(order.UserID, order.Amount); err != nil {
        log.Error("积分更新失败", err)
    }

    // 5. 记录日志
    if err := auditService.LogOrder(order); err != nil {
        log.Error("日志记录失败", err)
    }

    // 6. 发优惠券
    if err := couponService.SendCoupon(order.UserID); err != nil {
        log.Error("优惠券发放失败", err)
    }

    return nil
}
```

**这代码的毛病简直数不过来：**

1. **违反单一职责**：下单服务变成了"大杂烩"，什么活都干
2. **耦合度爆表**：增加一个新通知方式，得改下单服务代码
3. **性能垃圾**：所有通知都是串行执行，发个邮件要 2 秒，发个短信要 1 秒...
4. **错误处理混乱**：邮件失败了，订单要不要回滚？短信失败了怎么处理？
5. **难以测试**：想测下单逻辑，得把所有依赖的服务都 mock 一遍

**这代码维护起来就像是在泥坑里打滚，改一个功能可能把其他功能都搞炸。**

---

## 二、观察者模式核心思想

用一句话说清楚：**"一"发布事件，"多"个观察者自动响应。**

看个生活例子：

```
你订阅了一个技术公众号（注册观察者）
公众号发布新文章（通知所有观察者）
你在手机上收到推送（观察者响应）

同时：
- 张三也订阅了，他也收到推送
- 李四也订阅了，他也收到推送

公众号不需要知道：
- 有多少人订阅了
- 订阅者是谁
- 订阅者收到消息后做什么

公众号只负责：发消息
```

**这就是观察者模式的核心：发布者和Event订阅者互不依赖，彻底解耦。**

---

## 三、Go 版观察者模式实现

### 基础实现

```go
// ========== 观察者接口 ==========
type Observer interface {
    Update(data interface{})  // 收到通知时调用的方法
}

// ========== 被观察者（主题） ==========
type Subject struct {
    observers []Observer
    mu        sync.RWMutex
}

// 注册观察者
func (s *Subject) Attach(observer Observer) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.observers = append(s.observers, observer)
}

// 取消注册
func (s *Subject) Detach(observer Observer) {
    s.mu.Lock()
    defer s.mu.Unlock()

    for i, obs := range s.observers {
        if obs == observer {
            s.observers = append(s.observers[:i], s.observers[i+1:]...)
            break
        }
    }
}

// 通知所有观察者
func (s *Subject) Notify(data interface{}) {
    s.mu.RLock()
    defer s.mu.RUnlock()

    for _, observer := range s.observers {
        observer.Update(data)
    }
}

// ========== 具体观察者 ==========
type EmailObserver struct{}

func (e *EmailObserver) Update(data interface{}) {
    order := data.(Order)
    fmt.Printf("📧 发送订单邮件给 %s\n", order.UserEmail)
}

type SMSObserver struct{}

func (s *SMSObserver) Update(data interface{}) {
    order := data.(Order)
    fmt.Printf("📱 发送订单短信给 %s\n", order.UserPhone)
}

type PointsObserver struct{}

func (p *PointsObserver) Update(data interface{}) {
    order := data.(Order)
    fmt.Printf("🎁 给用户 %d 增加积分: %.2f\n", order.UserID, order.Amount)
}

// ========== 使用 ==========
func main() {
    // 创建主题
    orderSubject := &Subject{}

    // 注册观察者
    orderSubject.Attach(&EmailObserver{})
    orderSubject.Attach(&SMSObserver{})
    orderSubject.Attach(&PointsObserver{})

    // 下单成功后通知
    order := Order{
        ID:        1001,
        UserID:    12345,
        Amount:    999.99,
        UserEmail: "user@example.com",
        UserPhone: "13800138000",
    }

    orderSubject.Notify(order)
}
```

**输出：**

```
📧 发送订单邮件给 user@example.com
📱 发送订单短信给 13800138000
🎁 给用户 12345 增加积分: 999.99
```

**这下下单服务就清爽多了：**

```go
func PlaceOrder(order Order) error {
    // 1. 创建订单
    if err := db.Create(order); err != nil {
        return err
    }

    // 2. 通知所有观察者
    orderSubject.Notify(order)

    return nil
}
```

---

## 四、进阶版：带错误处理和异步执行

上面那个基础版本有个问题：**如果邮件发失败了怎么办？**

版本 1：继续执行其他观察者，记录错误

```go
// ========== 带错误处理的观察者接口 ==========
type Observer interface {
    Update(data interface{}) error  // 返回错误
}

func (s *Subject) Notify(data interface{}) []error {
    s.mu.RLock()
    defer s.mu.RUnlock()

    var errors []error = make([]error, 0)

    for _, observer := range s.observers {
        if err := observer.Update(data); err != nil {
            errors = append(errors, err)
            log.Error("观察者执行失败", err)
"            // 继续执行下一个观察者，不中断
        }
    }

    return errors
}
```

版本 2：异步执行，提高性能

```go
// ========== 异步通知 ==========
func (s *Subject) NotifyAsync(data interface{}) {
    s.mu.RLock()
    defer s.mu.RUnlock()

    var wg sync.WaitGroup

    for _, observer := range s.observers {
        wg.Add(1)
        go func(obs Observer) {
            defer wg.Done()
            if err := obs.Update(data); err != nil {
                log.Error("观察者执行失败", err)
            }
        }(observer)
    }

    wg.Wait()
}
```

版本 3：带超时控制

```go
func (s *Subject) NotifyWithTimeout(data interface{}, timeout time.Duration) error {
    ctx, cancel := context.WithTimeout(context.Background(), timeout)
    defer cancel()

    s.mu.RLock()
    defer s.mu.RUnlock()

    var wg sync.WaitGroup
    errChan := make(chan error, len(s.observers))

    for _, observer := range s.observers {
        wg.Add(1)
        go func(obs Observer) {
            defer wg.Done()

            done := make(chan struct{})
            var err error

            go func() {
                err = obs.Update(data)
                close(done)
            }()

            select {
            case <-done:
                if err != nil {
                    errChan <- err
                }
            case <-ctx.Done():
                errChan <- fmt.Errorf("观察者执行超时")
            }
        }(observer)
    }

    go func() {
        wg.Wait()
        close(errChan)
    }()

    var errors []error
    for err := range errChan {
        errors = append(errors, err)
    }

    if len(errors) > 0 {
        return fmt.Errorf("部分观察者执行失败: %v", errors)
    }

    return nil
}
```

---

## 五、实战案例：股价监控

假设你要做一个股价监控系统，当股价变化时通知所有用户。

```go
// ========== 股价主题 ==========
type StockTicker struct {
    Subject
    symbol string
    price  float64
}

func NewStockTicker(symbol string) *StockTicker {
    return &StockTicker{
        symbol: symbol,
        price:  0.0,
    }
}

// 更新股价并通知
func (s *StockTicker) SetPrice(price float64) {
    if s.price == price {
        return  // 价格没变就不通知
    }

    s.price = price
    fmt.Printf("\n📈 %s 股价更新: %.2f\n", s.symbol, price)

    s.Notify(StockEvent{
        Symbol: s.symbol,
        Price:  price,
        Time:   time.Now(),
    })
}

// ========== 股票事件 ==========
type StockEvent struct {
    Symbol string
    Price  float64
    Time   time.Time
}

// ========== 具体观察者 ==========
type Investor struct {
    Name      string
    SellPrice float64  // 目标卖出价
    BuyPrice  float64  // 目标买入价
}

func (i *Investor) Update(data interface{}) error {
    event := data.(StockEvent)

    if event.Price >= i.SellPrice {
        fmt.Printf("🔔 %s: 股价 %.2f 达到卖出目标，卖出！\n", i.Name, event.Price)
    } else if event.Price <= i.BuyPrice {
        fmt.Printf("🔔 %s: 股价 %.2f 达到买入目标，买入！\n", i.Name, event.Price)
    }

    return nil
}

// ========== 使用 ==========
func main() {
    ticker := NewStockTicker("AAPL")

    // 投资者 A：100 美元卖出，80 美元买入
    ticker.Attach(&Investor{Name: "张三", SellPrice: 100, BuyPrice: 80})

    // 投资者 B：120 美元卖出，90 美元买入
    ticker.Attach(&Investor{Name: "李四", SellPrice: 120, BuyPrice: 90})

    // 模拟股价变化
    ticker.SetPrice(95)
    ticker.SetPrice(85)
    ticker.SetPrice(105)
    ticker.SetPrice(125)
}
```

**输出：**

```
📈 AAPL 股价更新: 95.00

📈 AAPL 股价更新: 85.00
🔔 张三: 股价 85.00 达到买入目标，买入！

📈 AAPL 股价更新: 105.00
🔔 张三: 股价 105.00 达到卖出目标，卖出！

📈 AAPL 股价更新: 125.00
🔔 李四: 股价 125.00 达到卖出目标，卖出！
```

---

## 六、观察者模式 vs Go channel

很多 Go 开发者会问："为啥不用 channel？"

**channel 确实好用，但观察者模式有它的优势。**

| 对比项 | 观察者模式 | Go Channel |
|--------|-----------|------------|
| 灵活性 | 可以动态注册/注销观察者 | channel 数量固定，需要提前定义 |
| 解耦度 | 观察者之间互不感知 | 接收方需要知道所有 channel |
| 广播能力 | 天然支持一对多 | 需要手动遍历多个 channel |
| 类型安全 | 接口约束，编译时检查 | 接收任意类型，可能类型断言失败 |

**场景选择：**

- **固定的一对一通信** → 用 channel
- **动态的一对多通知** → 用观察者模式
- **事件驱动架构** → 用观察者模式

---

## 七、避坑指南（血泪经验）

### ❌ 坑 1：观察者执行顺序影响结果

```go
func (s *Subject) Notify(data interface{}) {
    for _, observer := range s.observers {
        observer.Update(data)  // 执行顺序重要！
    }
}

// 如果先执行"发邮件"，再执行"更新积分"
// 但你的业务逻辑要求"必须先更新积分，再发邮件"
// 这时候顺序问题就会导致 bug
```

**解决：按优先级排序，或者用 DAG（有向无环图）管理依赖关系。**

```go
type ObserverWithPriority struct {
    observer  Observer
    priority  int
}

func (s *Subject) Notify(data interface{}) {
    // 按优先级排序
    sort.Slice(s.observers, func(i, j int) bool {
        return s.observers[i].priority < s.observers[j].priority
    })

    for _, obs := range s.observers {
        obs.observer.Update(data)
    }
}
```

### ❌ 坑 2：观察者执行阻塞主流程

```go
func (s *Subject) Notify(data interface{}) {
    for _, observer := range s.observers {
        observer.Update(data)  // 如果这个观察者要 5 秒才执行完
    }
    // 整个流程就阻塞 5 秒！
}
```

**解决：异步执行。**

```go
func (s *Subject) Notify(data interface{}) {
    for _, observer := range s.observers {
        go observer.Update(data)  // 异步执行
    }
}
```

### ❌ 坑 3：观察者内存泄漏

```go
func (s *Subject) Attach(observer Observer) {
    s.observers = append(s.observers, observer)
    // 如果忘记 Detach，observer 会一直被持有
    // 如果 observer 是个大的对象，就内存泄漏了
}
```

**解决：用弱引用或者定期清理。**

```go
type Observer struct {
    ID        string
    LastSeen  time.Time
}

// 定期清理长时间不活跃的观察者
func (s *Subject) CleanupInactive(timeout time.Duration) {
    now := time.Now()
    var active []Observer

    for _, obs := range s.observers {
        if now.Sub(obs.LastSeen) < timeout {
            active = append(active, obs)
        }
    }

    s.observers = active
}
```

---

## 八、总结

**观察者模式的核心价值：**

1. **解耦**：发布者和订阅者互不依赖，可以独立演化
2. **扩展性**：新增订阅者不用改发布者代码
3. **动态性**：运行时可以动态注册/注销观察者

**适用场景：**

- ✅ 事件驱动架构：用户操作后触发多个副作用
- ✅ 消息通知系统：一个事件通知多个接收方
- ✅ 监控告警系统：指标变化后触发多个告警
- ✅ GUI 框架：按钮点击后触发多个回调

**不适用场景：**

- ❌ 固定的一对一通信：用 channel 更合适
- ❌ 需要严格顺序：用流水线模式
- ❌ 高性能场景：观察者有开销，直接调函数更快

---

**最后送你一句：**

> "观察者模式就像技术公众号，你关注了就能收到推送。但别什么都订阅，消息太多了你根本看不过来。"

**下期预告：装饰器模式——别再为了加个日志功能就到处写重复代码，学会这招让你的代码复用度直接起飞！**