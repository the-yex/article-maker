# 别再为了加个日志功能就到处写重复代码！Go 装饰器模式让你的代码复用度直接起飞

上周我在 code review 的时候，看到了这样一段代码：

```go
func HandleRequest(req Request) Response {
    // start := time.Now()  // 第一处：计算耗时
    // defer log.Printf("耗时: %v", time.Since(start))

    // 验证认证
    if !auth.Check(req.Header["Authorization"]) {
        return Response{Status: 401}
    }

    // log.Printf("请求处理中: %s", req.Path)  // 第二处：日志

    // 验证参数
    if err := validate(req); err != nil {
        return Response{Status: 400, Error: err}
    }

    // 处理业务逻辑
    resp := processBusiness(req)

    // log.Printf("请求完成: %s, 状态: %d", req.Path, resp.Status)  // 第三处：日志

    return resp
}

func HandleAdminRequest(req Request) Response {
    // start := time.Now()  // 又复制了一遍！
    // defer log.Printf("耗时: %v", time.Since(start))

    if !auth.CheckAdmin(req.Header["Authorization"]) {
        return Response{Status: 403}
    }

    // log.Printf("管理员请求: %s", req.Path)  // 又复制了一遍！

    resp := processAdmin(req)

    // log.Printf("管理员请求完成")  // 又复制了一遍！

    return resp
}
```

**这代码写得跟屎山一样，同样的日志逻辑、计时逻辑、验证逻辑在好几个地方复制粘贴。**

今天不讲废话，我们直接拆解装饰器模式，看看怎么把这种"功能叠加"的场景处理得优雅又复用。

---

## 一、装饰器模式到底是干嘛的？

用一句话说清楚：**在不修改原函数的情况下，给函数"包装"额外的功能。**

看个生活例子：

```
基础功能：一杯纯咖啡

装饰器 1：加糖 -> 甜咖啡
装饰器 2：加奶 -> 咖啡拿铁
装饰器 3：加奶泡 -> 拿铁玛奇朵

你可以随意组合：
- 纯咖啡
- 糖 + 咖啡
- 奶 + 咖啡
- 糖 + 奶 + 咖啡

原来的咖啡不需要知道：
- 会被加什么
- 被加了多少层

咖啡只负责：是咖啡
装饰器负责：加什么
```

**这就是装饰器模式的核心：通过"包装"实现功能叠加，而不是修改原代码。**

---

## 二、传统做法的痛点

假设你有 3 个核心功能：
- **A**: 日志记录
- **B**: 认证验证
- **C**: 计算耗时

你有 5 个接口需要不同的组合：
- 接口 1：需要 A + C
- 接口 2：需要 A + B + C
- 接口 3：需要 B + C
- 接口 4：需要 A
- 接口 5：需要 C

### ❌ 传统做法：到处复制粘贴

```go
// 接口 1
func Handler1(req Request) Response {
    start := time.Now()  // C: 计时
    defer log.Printf("耗时: %v", time.Since(start))

    log.Printf("请求: %s", req.Path)  // A: 日志

    resp := process1(req)

    return resp
}

// 接口 2
func Handler2(req Request) Response {
    start := time.Now()  // C: 计时（复制）
    defer log.Printf("耗时: %v", time.Since(start))

    log.Printf("请求: %s", req.Path)  // A: 日志（复制）

    if !auth.Check(req.Header["Authorization"]) {  // B: 认证
        return Response{Status: 401}
    }

    resp := process2(req)

    return resp
}

// ... 接口 3、4、5 同样的重复代码
```

**这代码的毛病：**

1. **重复代码爆炸**：5 个接口 × 3 个功能 = 15 处重复
2. **难以维护**：改一个日志格式，要改 5 个地方
3. **容易漏写**：加一个新接口，很容易忘记加某个装饰器
4. **顺序问题**：日志、认证、计时的顺序容易搞错

**这代码维护起来就像是在玩俄罗斯方块，每加一个功能就要重新拼一遍。**

---

## 三、Go 版装饰器模式实现

### 基础实现：函数包装

```go
// ========== 基础处理函数类型 ==========
type Handler func(req string) string

// ========== 装饰器函数 ==========
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

// ========== 核心处理函数 ==========
func BusinessHandler(req string) string {
    return fmt.Sprintf("处理结果: %s", req)
}

// ========== 使用：自由组合 ==========
func main() {
    // 组合装饰器：日志 -> 认证 -> 计时 -> 业务逻辑
    decoratedHandler := LoggingDecorator(
        AuthDecorator(
            TimingDecorator(BusinessHandler),
        ),
    )

    result := decoratedHandler("/api/users")
    fmt.Println(result)
}
```

**装饰器的执行顺序：**

```
LoggingDecorator (外层)
    ↓
AuthDecorator (中层)
    ↓
TimingDecorator (内层)
    ↓
BusinessHandler (核心)
```

**输出：**

```
📥 收到请求: /api/users
⏱️ 耗时: 123ms
📤 返回响应: 处理结果: /api/users
```

---

## 四、实战案例：HTTP 中间件链

这才是装饰器模式在 Go 里最经典的应用场景。

```go
// ========== HTTP 处理器类型 ==========
type HttpHandler func(w http.ResponseWriter, r *http.Request)

// ========== 中间件 ==========
func LoggingMiddleware(next HttpHandler) HttpHandler {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        log.Printf("📥 %s %s", r.Method, r.URL.Path)

        next(w, r)

        log.Printf("📤 耗时: %v", time.Since(start))
    }
}

func AuthMiddleware(next HttpHandler) HttpHandler {
    return func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")

        if !isValidToken(token) {
            w.WriteHeader(http.StatusUnauthorized)
            w.Write([]byte("401 Unauthorized"))
            return
        }

        next(w, r)
    }
}

func CORSMiddleware(next HttpHandler) HttpHandler {
    return func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE")

        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }

        next(w, r)
    }
}

func RateLimitMiddleware(next HttpHandler) HttpHandler {
    return func(w http.ResponseWriter, r *http.Request) {
        if !checkRateLimit(r.RemoteAddr) {
            w.WriteHeader(http.StatusTooManyRequests)
            w.Write([]byte("429 Too Many Requests"))
            return
        }

        next(w, r)
    }
}

// ========== 中间件链组装 ==========
func Chain(handler HttpHandler, middlewares ...func(HttpHandler) HttpHandler) HttpHandler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        handler = middlewares[i](handler)
    }
    return handler
}

// ========== 核心处理函数 ==========
func GetUserHandler(w http.ResponseWriter, r *http.Request) {
    user := getUserFromDB(r.URL.Query().Get("id"))
    json.NewEncoder(w).Encode(user)
}

func CreateUserHandler(w http.ResponseWriter, r *http.Request) {
    var user User
    json.NewDecoder(r.Body).Decode(&user)

    created := createUser(user)
    json.NewEncoder(w).Encode(created)
}

// ========== 注册路由 ==========
func main() {
    // GET /users: 日志 + CORS + 认证
    http.HandleFunc("/users", Chain(
        GetUserHandler,
        LoggingMiddleware,
        CORSMiddleware,
        AuthMiddleware,
    ))

    // POST /users: 日志 + CORS + 认证 + 限流
    http.HandleFunc("/users/create", Chain(
        CreateUserHandler,
        LoggingMiddleware,
        CORSMiddleware,
        AuthMiddleware,
        RateLimitMiddleware,
    ))

    http.ListenAndServe(":8080", nil)
}
```

**这代码写得简直太优雅了：**

1. **复用性极强**：中间件可以随意组合
2. **扩展性好**：加个新中间件，其他代码不用动
3. **顺序清晰**：组装时一眼就能看出执行顺序
4. **易于测试**：可以单独测试每个中间件

---

## 五、进阶：带参数的装饰器

有时候装饰器需要传参数，比如限流中间件要传速率。

```go
// ========== 带参数的限流中间件 ==========
func RateLimitMiddleware(maxRequests int, duration time.Duration) func(HttpHandler) HttpHandler {
    limiter := rate.NewLimiter(rate.Every(duration/time.Duration(maxRequests)), maxRequests)

    return func(next HttpHandler) HttpHandler {
        return func(w http.ResponseWriter, r *http.Request) {
            if !limiter.Allow() {
                w.WriteHeader(http.StatusTooManyRequests)
                w.Write([]byte("429 Too Many Requests"))
                return
            }

            next(w, r)
        }
    }
}

// ========== 使用 ==========
func main() {
    // 限制：100 请求/秒
    http.HandleFunc("/api", Chain(
        BusinessHandler,
        RateLimitMiddleware(100, time.Second),  // 传参数
    ))

    // 限制：10 请求/秒
    http.HandleFunc("/api/admin", Chain(
        AdminHandler,
        RateLimitMiddleware(10, time.Second),  // 不同参数
    ))
}
```

---

## 六、装饰器模式 vs 继承

很多从 Java 转过来的同学会问："为啥不用继承？"

**Go 没有继承，但这其实是好事！**

| 对比项 | 装饰器模式 | 继承（Java 风格） |
|--------|-----------|------------------|
| 灵活性 | 运行时动态组合 | 编译时静态绑定 |
| 复用性 | 装饰器可以复用 | 子类无法复用父类功能 |
| 扩展性 | 加个新装饰器就行 | 加个新功能要新建子类 |
| 类爆炸 | 只需要几个装饰器 | 各种组合都要新建子类 |

**例子：如果你想实现"糖 + 奶 + 奶泡"的咖啡：**

```java
// Java 继承：类爆炸！
class Coffee {}
class SugarCoffee extends Coffee {}
class MilkCoffee extends Coffee {}
class SugarMilkCoffee extends Coffee {}
class SugarMilkFoamCoffee extends MilkCoffee {}  // 越来越复杂
```

```go
// Go 装饰器：简单！
coffee := Coffee()
coffee = AddSugar(coffee)
coffee = AddMilk(coffee)
coffee = AddFoam(coffee)
```

---

## 七、避坑指南（血泪经验）

### ❌ 坑 1：装饰器顺序搞反了

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

**记住：外层装饰器先执行，内层装饰器后执行。**

### ❌ 坑 2：装饰器修改了请求/响应

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

### ❌ 坯 3：装饰器太多，性能下降

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

### ✅ 最佳实践：定义一个链式组装器

```go
type MiddlewareChain struct {
    middlewares []func(HttpHandler) HttpHandler
}

func NewMiddlewareChain() *MiddlewareChain {
    return &MiddlewareChain{middlewares: make([]func(HttpHandler) HttpHandler, 0)}
}

func (c *MiddlewareChain) Use(middleware func(HttpHandler) HttpHandler) *MiddlewareChain {
    c.middlewares = append(c.middlewares, middleware)
    return c
}

func (c *MiddlewareChain) Build(handler HttpHandler) HttpHandler {
    for i := len(c.middlewares) - 1; i >= 0; i-- {
        handler = c.middlewares[i](handler)
    }
    return handler
}

// ========== 使用：更优雅的写法 ==========
func main() {
    chain := NewMiddlewareChain().
        Use(LoggingMiddleware).
        Use(CORSMiddleware).
        Use(AuthMiddleware).
        Use(RateLimitMiddleware(100, time.Second))

    http.Handle("/api", chain.Build(BusinessHandler))
}
```

---

## 八、总结

**装饰器模式的核心价值：**

1. **功能复用**：一个装饰器可以在多个地方使用
2. **灵活组合**：可以随意组合装饰器，实现不同功能
3. **不改原代码**：遵循开闭原则，对扩展开放，对修改关闭
4. **符合单一职责**：每个装饰器只做一件事

**适用场景：**

- ✅ HTTP 中间件：日志、认证、限流、CORS
- ✅ 函数增强：计时、缓存、重试、熔断
- ✅ 输入输出处理：验证、格式化、转换
- ✅ 权限控制：角色验证、资源鉴权

**不适用场景：**

- ❌ 只有一两个地方用：直接写就行，别过度设计
- ❌ 装饰器有顺序依赖且容易搞错：用显式的组合方式
- ❌ 极致性能场景：装饰器有函数调用开销

---

**最后送你一句：**

> "装饰器模式就像给你的功能穿衣服，冬天穿羽绒服，下雨打雨伞。但别穿太多，会热死的。"

**下期预告：策略模式——别再到处写 if-else，学会这招让你的算法切换直接起飞！**