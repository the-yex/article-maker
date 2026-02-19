# 你写的 Go 代码，正在慢慢变成 Java

有多少次，你写过这样的代码？

``````go
type Option interface {
    Apply(*Config)
}
``````

然后为了实现它，又写了：

``````go
type cities struct {
    n int
}
func (c *cities) Apply(cfg *Config) { … }
``````

只为了干一件事：

 **往 Config 里塞一个值。**

当时你是不是还挺满意：

> “嗯，我这写法很 Go，很面向接口。”

其实，说实话，这完全就是`一大坨屎`

---

## 误区一：接口才是 Go 的“正统”？

很多人学 Go，是从 Java / C++ 转过来的。

潜意识里会觉得：

> 抽象 = 接口
> 设计 = interface
> 优雅 = 多态

但 Go 真正的杀器不是接口。

而是：**一等公民函数。**

先看一个最常见的场景：配置初始化。

### **用函数式选项模式：**

```go
type Config struct{ /* ... */ }

func WithReticulatedSplines(c *Config) { /* ... */ }

type Terrain struct{ config Config }

func NewTerrain(options ...func(*Config)) *Terrain {
    var t Terrain
    for _, option := range options {
        option(&t.config)
    }
    return &t
}

func main() {
    t := NewTerrain(WithReticulatedSplines)
}
```

如果要传参数：

```go
func WithCities(n int) func(*Config) { /* ... */ }

func main() {
    t := NewTerrain(WithCities(9))
}
```

这里的关键点是：

 WithCities 返回的不是值，而是**一个行为**。

现在，换成接口版看看：

```go
type Option interface { Apply(*Config) }

func NewTerrain(options ...Option) *Terrain {
    var config Config
    for _, option := range options {
        option.Apply(&config)
    }
}

    // 实现接口需要定义结构体和方法
type cities struct { cities int }

func (c *cities) Apply(cfg *Config) { /* ... */ }

func WithCities(n int) Option { return &cities{n} }
```

对比一下：
- 函数版本：一个函数搞定
- 接口版本：接口 + struct + 方法

哪个更简单，一目了然。函数本身就是一种类型，为什么要给它包一层壳？

---

## 误区二：只能传数据，不能传行为？

这是从面向对象语言过来的同学最容易踩的坑。

习惯写成：我传一个值,你来决定怎么处理

**但一等公民函数的核心就是：把行为本身当作数据传递。**

来看个计算器例子：

```go
type Calculator struct { acc float64 }

// Add 返回一个"加 n"的行为
func Add(n float64) func(float64) float64 {
    return func(acc float64) float64 { return acc + n }
}

// Do 接收行为并执行
func (c *Calculator) Do(op func(float64) float64) float64 {
    c.acc = op(c.acc)
    return c.acc
}

func main() {
    var c Calculator
    c.Do(Add(10))  // 执行"加 10"的行为
    c.Do(Add(20))  // 执行"加 20"的行为
}
```

Add(10) 返回的不是数字，

而是一个：**“加 10 的动作”**。

你传给 Do 的不是参数，

而是：**规则本身。**

甚至标准库函数也能直接用：

```go
func main() {
    var c Calculator
    c.Do(Add(16))   // acc = 16
    c.Do(math.Sqrt) // acc = 4，直接传 math.Sqrt
}
```

为什么 math.Sqrt 可以直接丢进去？

因为它的签名是：`func(float64) float64`

**用函数还是用接口？问自己一个问题：** 你是需要多种实现，还是只需要一次性的行为？如果是后者，函数就够了。

---

## 误区三：并发一定要靠 Mutex？

这是最容易写成“祖传写法”的地方。

```go
type Mux struct {
    mu    sync.Mutex
    conns map[net.Addr]net.Conn
}

func (m *Mux) Add(conn net.Conn) {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.conns[conn.RemoteAddr()] = conn
}

func (m *Mux) SendMsg(msg string) error {
    m.mu.Lock()
    defer m.mu.Unlock()
    for _, conn := range m.conns {
        err := io.WriteString(conn, msg)
        if err != nil { return err }
    }
    return nil
}
```

问题是：

- 每个方法都要锁
- 忘 unlock 就是事故
- 高并发下锁是瓶颈
- 可维护性极差

**Go 的哲学是：不要通过共享内存来通信，要通过通信来共享内存。**

换一种写法：

```go
type Mux struct { ops chan func(map[net.Addr]net.Conn) }

func (m *Mux) Add(conn net.Conn) {
    m.ops <- func(m map[net.Addr]net.Conn) {
        m[conn.RemoteAddr()] = conn
    }
}

func (m *Mux) SendMsg(msg string) error {
    result := make(chan error, 1)
    m.ops <- func(m map[net.Addr]net.Conn) {
        for _, conn := range m {
            err := io.WriteString(conn, msg)
            if err != nil {
                result <- err
                return
            }
        }
        result <- nil
    }
    return <-result
}

// 后台 goroutine 串行执行所有操作
func (m *Mux) loop() {
    conns := make(map[net.Addr]net.Conn)
    for op := range m.ops { op(conns) }
}
```

注意这里发生了什么变化：

- 所有状态只存在于 loop 这个 goroutine
- 外部只能通过：**发送函数**
- 每个函数都是一次“操作请求”

你不再操作 map，

你是在给 worker 发“指令”。

新增一个私聊功能：

```go
func (m *Mux) PrivateMsg(addr net.Addr, msg string) error {
    result := make(chan net.Conn, 1)
    m.ops <- func(m map[net.Addr]net.Conn) { result <- m[addr] }
    conn := <-result
    if conn == nil { return errors.Errorf("client %v not registered", addr) }
    return io.WriteString(conn, msg)
}
```

没有锁，不阻塞，逻辑清晰。这就是 Actor 模型的 Go 版本——每个操作都是一个函数，通过 channel 传递给串行执行的 worker。

---

## 避坑指南

在项目里用函数当一等公民时，记住这几条：

1. **别为了函数式而函数式**
2. **签名要清晰**：func(*Config) > func(interface{})
3. **闭包注意循环变量**（Go 1.22+ 已修复老坑）
4. **错误别吞掉**，尤其在 channel 里的函数

---

## 最后说一句

一等公民函数不是什么新概念，

但它确实是 Go 最容易被忽视的能力之一。

当你下次设计 API 时，问自己一个问题：这个接口只有一个方法吗？如果是，那它应该就是一个函数。

函数比接口更简单，简单就是好代码。

**你是否在项目中经常第一想到的是接口不是函数？评论区聊聊。**