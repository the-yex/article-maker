# Go的每一个框架都在用的设计模式---装饰器模式

## 模拟一个场景：你决定自己手搓一个框架

假设有一天，你突然脑子一热：

> **“我不想再用 Gin / Echo 了，我想自己手搓一个 Web 框架。”**

目标很简单：

- 能注册路由
- 能处理请求
- 每个接口都要统一支持：
    - 打日志
    - 校验登录
    - 统计耗时

于是你开始写第一个接口：

``````go
func GetUser(w http.ResponseWriter, r *http.Request) {
    log.Println("请求进来了")
    if !checkLogin(r) {
        w.WriteHeader(401)
        return
    }
    start := time.Now()
    user := queryUser()
    log.Println("耗时：", time.Since(start))
    json.NewEncoder(w).Encode(user)
}
``````
跑得挺好，你很满意。

然后你写第二个接口：
``````go
func CreateUser(w http.ResponseWriter, r *http.Request) {
    log.Println("请求进来了")

    if !checkLogin(r) {
        w.WriteHeader(401)
        return
    }

    start := time.Now()
    createUser()
    log.Println("耗时：", time.Since(start))
}
``````

第三个、第四个、第五个……

你突然发现一个可怕的事实：

> **你写的根本不是业务代码，而是在一遍遍复制：**
> - 打日志
> - 校验登录
> - 统计耗时

你的“框架”长这样：

> 业务逻辑 + 日志 + 鉴权 + 计时  
> 业务逻辑 + 日志 + 鉴权 + 计时  
> 业务逻辑 + 日志 + 鉴权 + 计时……

你以为你在造框架，

实际上你在批量生产屎山。

这时候你会意识到一个核心问题：

> **我能不能不改业务代码，只给它“外挂”功能？**

也就是说：

- 业务函数：只关心“干活”
- 日志、鉴权、计时：统一从外面套上去

如果你能做到这一点：

``````go
业务函数
   ↓
日志
   ↓
鉴权
   ↓
计时
``````

那你的“框架”，才算真的成型了。

而这个思路，在设计模式里有一个专门的名字：

> **装饰器模式（Decorator Pattern）**

接下来我们不用抽象概念，

就用 Go 的真实代码，一步一步把这个“迷你框架”搭出来。

## 什么是装饰器模式？

用一句话说清楚：**在不修改原函数的情况下，给函数"包装"额外的功能。**

就像一杯咖啡 ☕：

```
基础功能：纯咖啡

装饰器 1：加糖 → 甜咖啡  
装饰器 2：加奶 → 拿铁  
装饰器 3：加奶泡 → 玛奇朵  
```

你可以自由组合：
- 纯咖啡
- 糖 + 咖啡
- 奶 + 咖啡
- 糖 + 奶 + 咖啡

咖啡本身不关心：

- 谁给它加了糖
- 谁给它加了奶

**它只负责一件事：自己是咖啡。**

**这就是装饰器模式的核心思想：通过"包装"实现功能叠加，而不是修改原代码。**

## 用 Go 实现一个最小装饰器

### **抽象一个最小 Handler**

```go
// 基础处理函数类型
type Handler func(req string) string
```

### **定义三个装饰器**

```go
func LoggingDecorator(next Handler) Handler {
    return func(req string) string {
        log.Printf("📥 收到请求: %s", req)
        resp := next(req)
        log.Printf("📤 返回响应: %s", resp)
        return resp
    }
}

func AuthDecorator(next Handler) Handler {
    return func(req string) string {
        if !isAuthenticated() {
            log.Printf("🚫 认证失败")
            return "401 Unauthorized"
        }
        return next(req)
    }
}

func TimingDecorator(next Handler) Handler {
    return func(req string) string {
        start := time.Now()
        defer func() {
            log.Printf("⏱️ 耗时: %v", time.Since(start))
        }()
        return next(req)
    }
}
```

### **业务逻辑**

```go
func BusinessHandler(req string) string {
    return fmt.Sprintf("处理结果: %s", req)
}
```

##  **像“搭积木”一样组装功能**

```go
func main() {
    // 组合装饰器：日志 → 认证 → 计时 → 业务逻辑
    decoratedHandler := LoggingDecorator(
        AuthDecorator(
            TimingDecorator(BusinessHandler),
        ),
    )
    result := decoratedHandler("/api/users")
    fmt.Println(result)
}
```

结构实际上是：

``````go
LoggingDecorator(
  AuthDecorator(
    TimingDecorator(
      BusinessHandler
    )
  )
)
``````

就像给函数一层层穿衣服：

- 最里面：业务逻辑
- 外面一层：计时
- 再外一层：鉴权
- 最外层：日志

你现在终于可以做到：

- A 接口：日志 + 计时
- B 接口：日志 + 认证
- C 接口：日志 + 认证 + 计时

而不是：再复制一坨代码。

## 5. 实战场景：HTTP 中间件

你每天写的其实就是这个：

``````go
http.HandleFunc("/users", Chain(
    GetUserHandler,
    LoggingMiddleware,
    AuthMiddleware,
    RateLimitMiddleware,
))
``````

补一个最简 Chain 实现：

``````go
func Chain(h HttpHandler, mws ...func(HttpHandler) HttpHandler) HttpHandler {
    for i := len(mws) - 1; i >= 0; i-- {
        h = mws[i](h)
    }
    return h
}
``````

Gin、Echo、net/http 的中间件，本质上全是：

> **装饰器模式的工程化落地**

## 装饰器模式 vs 继承（为什么 Go 更适合）

很多从 Java 转过来的同学会问："为啥不用继承？"

因为继承会导致：**类爆炸**。

如果你要做“糖 + 奶 + 奶泡”的咖啡：

``````java
// Java 继承：类爆炸！
class Coffee {}
class SugarCoffee extends Coffee {}
class MilkCoffee extends Coffee {}
class SugarMilkCoffee extends Coffee {}
class SugarMilkFoamCoffee extends MilkCoffee {}  // 越来越复杂
``````

而用装饰器：

``````go
coffee := Coffee()
coffee = AddSugar(coffee)
coffee = AddMilk(coffee)
coffee = AddFoam(coffee)
``````

| 对比项 | 装饰器模式 | 继承（Java 风格） |
|--------|-----------|------------------|
| 灵活性 | 运行时动态组合 | 编译时静态绑定 |
| 复用性 | 装饰器可以复用 | 子类无法复用父类功能 |
| 扩展性 | 加个新装饰器就行 | 加个新功能要新建子类 |
| 复杂度 | 线性增长 | 指数级膨胀 |

## 避坑指南

### 坑 1：装饰器顺序搞反了

```go
// 错误：认证在计时外面，认证失败的时间也算进去了
handler := TimingDecorator(
    AuthDecorator(BusinessHandler),
)

// 正确：只计算业务逻辑的耗时
handler := AuthDecorator(
    TimingDecorator(BusinessHandler),
)
```

**记住：外层先执行，内层后执行**

### 坑 2：装饰器修改了请求/响应

```go
func BadMiddleware(next HttpHandler) HttpHandler {
    return func(w http.ResponseWriter, r *http.Request) {
        // ❌ 别乱改请求！
        r.URL.Path = "/api/force-change"

        // ❌ 别乱改响应！
        w.WriteHeader(200)  // 提前写入响应头

        next(w, r)
    }
}
```

**装饰器应该"透明"，不要修改原始请求和响应。**

### 坑 3：装饰器太多，性能下降

```go
// 20 个装饰器嵌套，每一层都有函数调用开销
handler := Decorator20(
    Decorator19(
        Decorator18(
            // ... 一层层嵌套
            BusinessHandler,
        ),
    ),
)
```

**解决：把频繁调用的装饰器内联，或者用代码生成。**

> 高频核心接口可内敛优化或者生成链式代码

## 总结

**装饰器模式的核心价值：**
1. 功能复用：一个装饰器可以在多个地方使用
2. 灵活组合：可以随意组合装饰器，实现不同功能
3. 不改原代码：遵循开闭原则，对扩展开放，对修改关闭
4. 符合单一职责：每个装饰器只做一件事

**适用场景：**

- HTTP 中间件：日志、认证、限流、CORS
- 函数增强：计时、缓存、重试、熔断
- 输入输出处理：验证、格式化、转换
- 权限控制：角色验证、资源鉴权

**不适用场景：**

- 只有一两个地方用：直接写就行，别过度设计
- 装饰器有顺序依赖且容易搞错：用显式的组合方式
- 极致性能场景：装饰器有函数调用开销

> "装饰器模式就像给你的功能穿衣服，冬天穿羽绒服，下雨打雨伞。但别穿太多，会热死的。"

**下期预告：策略模式 —— 别再到处写 if-else，学会这招让你的算法切换直接起飞！**