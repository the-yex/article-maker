上周产品提了个需求：

> “用户下单后，要：发邮件、发短信、更新积分、记录日志、发优惠券……”

我当场脑补了一下代码结构，差点把咖啡喷到屏幕上。

因为如果直接写，很可能会变成这样：

``````go
func PlaceOrder(order Order) error {
    db.Create(order)

    emailService.Send(order.UserEmail)
    smsService.Send(order.UserPhone)
    pointsService.Add(order.UserID)
    logService.Log(order)
    couponService.Send(order.UserID)

    return nil
}
``````

问题很明显：

- 下单服务成了“大杂烩”
- 新增一个动作就要改这里（耦合爆表）
- 所有逻辑串行执行（性能灾难）
- 想测试下单逻辑，要 mock 一堆服务（噩梦）

这就是典型的：**“一件事，引发一堆副作用”的场景。**

这正是  **观察者模式的用武之地。**

---

## 一、观察者模式在干嘛？

用一句话说清楚： **"一个地方发生变化，多个地方自动收到通知"。**

就像公众号：

- 你订阅了
- 它发文章
- 所有人收到推送

公众号不关心你是谁，

只负责：**发消息。**

**这就是观察者模式的核心：发布者和订阅者互不依赖，彻底解耦。**

---

## 二、Go 版最小实现

### **定义观察者**

```go
type Observer interface {
    Update(data interface{}) error
}
```

### **定义主题（被观察者）**

``````go
type Subject struct {
    observers []Observer
    mu sync.RWMutex
}

func (s *Subject) Attach(o Observer) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.observers = append(s.observers, o)
}

func (s *Subject) Notify(data interface{}) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    for _, o := range s.observers {
        o.Update(data)
    }
}
``````

### **具体观察者**

``````go
type EmailObserver struct{}
func (e *EmailObserver) Update(data interface{}) error {
    order := data.(Order)
    fmt.Println("发邮件给", order.UserEmail)
    return nil
}

type PointsObserver struct{}
func (p *PointsObserver) Update(data interface{}) error {
    order := data.(Order)
    fmt.Println("给用户加积分", order.UserID)
    return nil
}
``````

### **使用方式**

``````go
orderSubject := &Subject{}
orderSubject.Attach(&EmailObserver{})
orderSubject.Attach(&PointsObserver{})

orderSubject.Notify(order)
``````

现在下单逻辑只剩：

``````go
func PlaceOrder(order Order) error {
    db.Create(order)
    orderSubject.Notify(order)
    return nil
}
``````

下单服务只负责一件事：

**下单成功 → 发事件**

---

## 三、进阶版：不影响主流程

如果发邮件要 2 秒，更新积分要 1 秒，你肯定不希望用户一直等。

```go
func (s *Subject) NotifyAsync(data interface{}) {
    s.mu.RLock()
    defer s.mu.RUnlock()

    var wg sync.WaitGroup
    for _, o := range s.observers {
        wg.Add(1)
        go func(obs Observer) {
            defer wg.Done()
            if err := obs.Update(data); err != nil {
                log.Println("observer error:", err)
            }
        }(o)
    }
    wg.Wait()
}
```

现在效果是：

- 下单逻辑不被阻塞
- 某个通知失败不影响其他
- 错误集中记录

---

## **四、真实业务场景**

### **用户注册后：**

- 发欢迎邮件
- 送积分
- 初始化推荐数据

```go
func (s *UserService) Register(user User) error {
    s.repo.Save(user)

    s.subject.Notify(UserRegisteredEvent{
        UserID: user.ID,
        Email: user.Email,
    })
    return nil
}
```

订阅者自己处理：

- 邮件服务
- 积分服务

以后还可以持续迭代追加：

- 推荐系统初始化
- 日志记录
- 拉新任务

**注册逻辑一行不改。**

---

## 五、观察者vs  channel

很多人会说：“这不就是 channel 吗？”

区别在这：

| **场景**     | **用什么** |
| ------------ | ---------- |
| 协程通信     | channel    |
| 业务事件广播 | 观察者模式 |

channel 是通信工具，

观察者是**架构模式**。

---

## 六、避坑指南

###  坑 1：**观察者阻塞主流程**

```go
observer.Update() // 慢操作
```

✅ 解决：异步

### 坑 2：顺序依赖

比如：必须先更新积分，再发邮件

但你却：

```go
for _, o := range observers {
    o.Update()
}
```

✅ 解决：

- 按优先级
- 或拆事件

### 坑 3：观察者内存泄漏

```go
func (s *Subject) Attach(observer Observer) {
    s.observers = append(s.observers, observer)
    // 如果忘记 Detach，observer 会一直被持有
    // 如果 observer 是个大的对象，就内存泄漏了
}
```

**✅ 解决：：用弱引用或者定期清理。**

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

## 七、总结

观察者模式帮你解决三件事：

- ✅ 解耦：下单不关心谁来处理后续
- ✅ 扩展：新增功能不用改主流程
- ✅ 清晰：主业务只剩“关键路径”

它特别适合：

- 用户注册后的一堆操作
- 订单状态变化
- 消息通知系统
- 监控告警系统

> **观察者模式就像公众号：**
> **你发文章，不关心谁看；**
> **谁想看，就自己订阅。**

**下期预告：装饰器模式——别再为了加日志，到处 copy 代码了。**