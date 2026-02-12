# Go语言Option模式：让你的构造函数瞬间变优雅 

你是否遇到过这样的场景：写一个客户端、数据库连接或者任务调度器，结构体字段一长，构造函数也跟着长成“巨无霸”？

默认值、可选值、布尔开关……越改越麻烦，越看越头大。

今天，我们就来聊聊 Go 开发中几乎无处不在的 **Option 模式**，它让对象初始化 **灵活、可读、可扩展**，彻底告别丑陋构造函数。

## Go 的构造函数为什么老是让人头痛

先对比一下其他语言：

- **Java**：有 Builder 链式调用，但类和方法一大堆，写起来累，稍不注意就容易忘记某个字段。
- **Python / JS**：可以靠关键字参数或字典传参，虽然灵活，但默认值逻辑分散，调用时可读性一般。
- **Go**：没有函数重载，也没有默认参数——可选参数要么写一堆构造函数，要么用配置结构体，麻烦到哭。

比如一个客户端配置结构体：

```go
type Client struct {
    Addr    string
    Port    int
    Timeout time.Duration
    MaxConn int
    TLS     bool
}
```

**传统做法痛点**：

```go
func NewClient(addr string, port int) *Client { ... }
func NewClientWithTimeout(addr string, port int, timeout time.Duration) *Client { ... }
func NewClientWithAllOptions(addr string, port int, timeout time.Duration, maxConn int, tls bool) *Client { ... }
```

- 参数顺序容易搞混
- 代码冗余，新增字段要改构造函数
- 默认值分散，逻辑不清

即便用配置结构体，也会发现：

```go
config := ClientConfig{Timeout: 60 * time.Second}
client := NewClient("127.0.0.1", 8080, config)
```

- 还得额外创建配置对象
- 默认值管理依旧分散
- 可读性一般

这就是 Go 开发中很多新手碰到的痛点。

## Option模式：优雅的解决方案

### 核心思想

**把可选参数封装成函数，通过可变参数传入构造函数**，在初始化对象时应用。

### 优雅实现方式

```go
type Client struct {
    Addr    string
    Port    int
    Timeout time.Duration
    MaxConn int
    TLS     bool
}

// Option 定义配置选项函数类型
type Option func(*Client)

// NewClient 创建Client实例，接受可变参数的配置选项
func NewClient(addr string, port int, opts ...Option) *Client {
    c := &Client{
        Addr:    addr,
        Port:    port,
        Timeout: 30 * time.Second,
        MaxConn: 100,
        TLS:     false,
    }
    for _, opt := range opts {
        opt(c)
    }
    return c
}

// 配置选项函数
func WithTimeout(timeout time.Duration) Option { return func(c *Client) { c.Timeout = timeout } }
func WithMaxConn(maxConn int) Option { return func(c *Client) { c.MaxConn = maxConn } }
func WithTLS(tls bool) Option { return func(c *Client) { c.TLS = tls } }
```

### **调用方式美观又清晰**

```go
// 只使用必填参数
client1 := NewClient("127.0.0.1", 8080)

// 一个可选参数
client2 := NewClient("127.0.0.1", 8080, WithTimeout(60*time.Second))

// 多个可选参数
client3 := NewClient("127.0.0.1", 8080,
    WithTimeout(60*time.Second),
    WithMaxConn(200),
    WithTLS(true),
)
```

优势一览无余：

- **代码简洁**：每个选项独立函数
- **可读性高**：调用时选项名称清楚
- **扩展性强**：新增参数只需加一个函数
- **默认值集中管理**：所有默认值在 NewClient 中统一定义

## Option模式的高级玩法

###  **参数验证**

```go
func WithMaxConn(maxConn int) Option {
    if maxConn <= 0 {
        panic("maxConn must be positive")
    }
    return func(c *Client) { c.MaxConn = maxConn }
}
```

### 配置选项组合

```go
func WithProductionConfig() Option {
    return func(c *Client) {
        WithTimeout(60*time.Second)(c)
        WithMaxConn(1000)(c)
        WithTLS(true)(c)
    }
}

// 调用
client := NewClient("api.example.com", 443, WithProductionConfig())
```

### **隐藏内部字段，提供接口**

```go
type client struct {
    addr    string
    port    int
    timeout time.Duration
    maxConn int
    tls     bool
}

type Client interface { DoRequest() error }

func newClient(addr string, port int, opts ...Option) Client {
    c := &client{addr: addr, port: port, timeout: 30*time.Second, maxConn: 100}
    for _, opt := range opts { opt(c) }
    return c
}
```

⚠️ **Tips**：

- Option 模式不仅仅是默认值，它还能灵活组合可选参数、做校验、隐藏复杂逻辑。
- 新增字段时，无需修改原有构造函数，符合开闭原则。

## 为什么 Go 开发者钟爱 Option 模式

Option模式解决了 Go 对象初始化的核心痛点：

1. **去除了构造函数的冗余代码**
2. **提高了代码的可读性和可维护性**
3. **符合开闭原则**：扩展配置项时不修改已有的代码
4. **默认值集中管理**

在 SDK、客户端、任务调度器、ORM 等各种场景里，它几乎成为标准做法。
