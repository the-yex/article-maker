# 面试官让你“手搓路由”？90% 的 Go 程序员当场翻车

前两天朋友跟我吐槽面试经历，说自己是**被一道题直接送走的**。

面试官说了一句：

> “别用 Gin / Echo，
> 手搓一个最简单的 HTTP 路由给我看看。”

同事当场愣住，沉默了十几秒，憋出一句：

> “平时都是框架一把梭……
> 标准库的路由，真没用过。”

面试官点了点头，说了句“好，那我们先聊到这”。

**十分钟结束面试。**

**HR 再也没联系过。**

很多人看到这里会说：

这题是不是有点刁钻？

说实话，**一点都不刁。**

Go 的 http.NewServeMux 就躺在标准库里，

只是——**很多人从来没点开过它。**

## 为什么面试官爱考这个？

框架用久了，很容易产生一种错觉：

> 只要接口能跑，原理不重要。
> 只要返回 200，底层怎么实现不用管。

这就是所谓的“魔法依赖”。

但面试官真正想看的不是你会不会用 Gin，

而是：

> **没了框架的脚手架，**
> **你还能不能从零把轮子搭起来？**

而 Go 标准库里的 http.NewServeMux，

就是最原始、最干净的练手场地。

它本质只有一件事：**把 URL 路径，映射到处理函数。**

## 3 分钟手搓一个简单路由系统

假设：

你要写一个计算器 API：

支持：加、减、乘、除。

核心代码就这些：

```go
func main() {
    mux := http.NewServeMux()

    mux.HandleFunc("/add", AddHandler)
    mux.HandleFunc("/sub", SubHandler)
    mux.HandleFunc("/mul", MulHandler)
    mux.HandleFunc("/div", DivHandler)

    log.Println("Server running on :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

你在干什么？

本质上只做了三件事：

1. 建路由表：http.NewServeMux()
2. 绑定路径：mux.HandleFunc
3. 启服务：http.ListenAndServe

这就是最原始的 HTTP 路由。

## 真正拉开水平差距的：Handler 写法

很多人会下意识写成这样：

``````go
func AddHandler(w http.ResponseWriter, r *http.Request) {
    a, _ := strconv.ParseFloat(r.URL.Query().Get("a"), 64)
    b, _ := strconv.ParseFloat(r.URL.Query().Get("b"), 64)
    fmt.Fprintf(w, "%f", a+b)
}
``````

这种代码在面试官眼里，只有一个评价：

> **“你只会写 Demo。”**

因为它同时踩了三个雷：

❌ 没校验参数

❌ 错误直接忽略

❌ 逻辑全耦合在 HTTP 里

正确姿势是：**拆层**

```go
func AddHandler(w http.ResponseWriter, r *http.Request) {
    a, b, err := ValidateParam(w, r)
    if err != nil {
        return
    }

    result := add(a, b)
    w.Header().Set("Content-Type", "text/plain")
    fmt.Fprintf(w, "Result : %.2f + %.2f = %.2f", a, b, result)

    logger.Printf("Status: %d, IP:: %s, Path: %s",
        http.StatusOK, r.RemoteAddr, r.URL.Path)
}
```

注意 ValidateParam 的位置——**校验逻辑一定要抽出来。**

否则你会在四个 Handler 里复制四遍，

然后在第五个接口开始骂自己。

## 验证逻辑长这样

```go
func ValidateParam(w http.ResponseWriter, r *http.Request) (float64, float64, error) {
    aStr := r.URL.Query().Get("a")
    bStr := r.URL.Query().Get("b")

    if aStr == "" || bStr == "" {
        http.Error(w, "Both 'a' and 'b' are required", http.StatusBadRequest)
        return 0, 0, fmt.Errorf("missing params")
    }

    a, err := strconv.ParseFloat(aStr, 64)
    if err != nil {
        http.Error(w, "param 'a' must be a valid number", http.StatusBadRequest)
        return 0, 0, err
    }

    b, err := strconv.ParseFloat(bStr, 64)
    if err != nil {
        http.Error(w, "param 'b' must be a valid number", http.StatusBadRequest)
        return 0, 0, err
    }

    return a, b, nil
}
```

这一点非常重要，因为它体现：

- 你有分层意识
- 你在为扩展做准备
- 你不是“脚本型写法”

## 业务逻辑必须和 HTTP 解耦

计算函数不应该知道 HTTP 的存在，它只管算数：

```go
func add(a, b float64) float64 { return a + b }
func sub(a, b float64) float64 { return a - b }
func multi(a, b float64) float64 { return a * b }
func div(a, b float64) float64 { return a / b }
```

为什么要这样？

因为这样你可以：

- 单独测业务逻辑
- 未来换成 gRPC / CLI 直接复用
- Handler 只负责“协议层”

这是很多初级程序员意识不到的点。

## ServeMux 的三个硬伤

如果你能在面试中主动说出下面这三点，

面试官大概率会给你加分：

> **因为这说明你不是只会用框架。**

### 1. 不支持路径参数

不能写：

``````go
/users/:id
``````

只能：

``````go
/users?id=123
``````

要么自己拆路径，要么上第三方路由。

### 2. 不区分请求方法

注册了 /add 后：

- GET 能访问
- POST 也能访问
- PUT 也能访问

想限制？

``````go
if r.Method != http.MethodGet {
    http.Error(w, "method not allowed", 405)
}
``````

### 3. 没有中间件机制

没有：

- 统一日志
- 统一鉴权
- 统一跨域

全靠你自己包一层 Handler。

这也是框架存在的意义。

## 面试官真正想看到的是什么？

不是你 API 写得多炫。

而是这几点：

- 你知道 HTTP 服务最小形态

- 你理解路由只是映射关系

- 你会拆 Handler / 业务 / 校验

- 你知道标准库的边界


一句话总结：

> **框架，是在标准库之上；**
> **不是代替你理解标准库。**

## 写在最后

手搓路由不是为了炫技，而是让你理解：框架不是魔法，它只是一层封装。理解了底层，再用框架才能用得明白。

下次面试官再让你手搓路由，别慌，`ServeMux` 一把梭。

最后一个问题：**你上一次不用框架写 HTTP 服务是什么时候？** 还是说，离开了 Gin/Echo 你连一个 hello world 都写不出来？