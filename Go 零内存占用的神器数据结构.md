你见过一种 Go 数据结构吗？

- `unsafe.Sizeof` 返回 **0**
- 创建 100 万个，**内存几乎不变**
- map 里用它，比 `bool` 还省
- channel 里用它，是官方推荐写法

它就是：**空结构体 `struct{}`**。

## 什么是空结构体？

空结构体是一种**不包含任何字段**的struct类型。它既可以是命名的，也可以是匿名的：

``````go
// 命名的空结构体
type EmptyStruct struct{}

// 匿名的空结构体
var emptyVar struct{}
``````

你可能会有疑问：一个没有字段、没有数据的结构体，到底有什么用？

在回答这个问题之前，我们需要先理解一个关键概念：**内存宽度（Width）**。

## **理解内存宽度**

在Go中，每个类型都有一个"width"属性，表示该类型实例在内存中占用的字节数。我们可以通过`unsafe.Sizeof()`来获取：

> 你可以把 width 理解为：
> **从某个地址开始，连续占用了多少字节**

``````go
var s string
var c complex128
fmt.Println(unsafe.Sizeof(s)) // 16字节
fmt.Println(unsafe.Sizeof(c)) // 16字节

// 数组的宽度是元素宽度的倍数
var a [3]uint32
fmt.Println(unsafe.Sizeof(a)) // 12字节（3 * 4字节）
``````

## 空结构体的关键特性

### 1. 零内存占用

空结构体最显著的特性就是**不占用任何内存**：

``````go
var s struct{}
fmt.Println(unsafe.Sizeof(s)) // 0
``````

这意味着无论你创建多少个空结构体实例，它们都不会增加内存负担。

### 2. 特殊的地址特性

由于空结构体不占用内存，它们的地址表现也很特殊：

``````go
var a, b struct{}
fmt.Println(&a == &b) // true - 地址相同！

// 在切片中也是如此
arr := make([]struct{}, 10)
fmt.Println(&arr[0] == &arr[5]) // true
``````

> ⚠️ 注意：空结构体地址相同是当前 Go 实现的结果，
语言规范只保证它们不占内存，不应依赖“地址相同”这个特性做逻辑判断。
### 3. 完整的类型系统支持

尽管不占内存，空结构体仍然是完整的Go类型，支持所有类型操作：

`````go
// 可以作为方法接收者
type Counter struct{}

func (c *Counter) Increment() {
    // 方法实现
}

// 可以实现接口
type Writer interface {
    Write([]byte) (int, error)
}

type NullWriter struct{}

func (n *NullWriter) Write(data []byte) (int, error) {
    return len(data), nil 
}
`````

## 空结构体的高效用途

### 1. **实现集合（Set）类型 ⭐⭐⭐⭐⭐（必会）**

这是空结构体最经典的用法。利用map的键唯一性，配合空结构体作为值类型，可以实现高效的内存集合：

``````go
type Set map[string]struct{}

func NewSet() Set {
    return make(Set)
}

func (s Set) Add(key string) {
    s[key] = struct{}{}
}

func (s Set) Remove(key string) {
    delete(s, key)
}

func (s Set) Contains(key string) bool {
    _, exists := s[key]
    return exists
}

// 使用示例
func main() {
    set := NewSet()
    set.Add("apple")
    set.Add("banana")
    set.Add("apple") // 重复添加无效
    
    fmt.Println(set.Contains("apple"))  // true
    fmt.Println(set.Contains("orange")) // false
    
    // 内存占用：只有键占用空间，值完全不占用额外内存
}
``````

**优势**：相比使用`bool`作为值类型（每个值占用1字节），空结构体实现集合更节省内存。

### 2. **通道信号传递**

当只需要传递信号而不关心数据内容时，空结构体通道是最佳选择：

``````go
// 优雅地关闭goroutine
func worker(stopCh <-chan struct{}) {
    for {
        select {
        case <-stopCh:
            fmt.Println("收到停止信号，优雅退出")
            return
        default:
            // 正常工作
            fmt.Println("工作中...")
            time.Sleep(1 * time.Second)
        }
    }
}

func main() {
    stopCh := make(chan struct{})
    go worker(stopCh)
    
    // 运行5秒后停止
    time.Sleep(5 * time.Second)
    close(stopCh) // 发送停止信号
    
    time.Sleep(1 * time.Second)
}
``````

**优势**：零内存开销，语义清晰（明确表示只关心信号，不关心数据）。

### 3. **方法接收者占位符**

当需要实现接口但不需要状态时，空结构体是完美的接收者：

``````go
// 标准库中的典型应用
type nullWriter struct{}

func (nullWriter) Write(p []byte) (int, error) {
    return len(p), nil // 不执行任何操作
}

// 提供默认实现
type DefaultHandler struct{}

func (h DefaultHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    http.Error(w, "Not implemented", http.StatusNotImplemented)
}

type singleton struct{}

var instance *singleton
var once sync.Once

func GetInstance() *singleton {
    once.Do(func() {
        instance = &singleton{}
    })
    return instance
}
``````

### 4. **测试和基准测试中的占位符**

在测试中，空结构体可以作为轻量级的测试数据：

``````go
func BenchmarkMapWithEmptyStruct(b *testing.B) {
    m := make(map[int]struct{})
    for i := 0; i < b.N; i++ {
        m[i] = struct{}{}
    }
}

func BenchmarkMapWithBool(b *testing.B) {
    m := make(map[int]bool)
    for i := 0; i < b.N; i++ {
        m[i] = true
    }
}
// 第一个基准测试通常会有更好的内存表现
``````

### 5. **控制并发访问**

使用空结构体通道实现并发控制原语：

``````go
// 限流器
type RateLimiter struct {
    limit    int
    tokens   chan struct{}
    interval time.Duration
}

func NewRateLimiter(limit int, interval time.Duration) *RateLimiter {
    rl := &RateLimiter{
        limit:    limit,
        tokens:   make(chan struct{}, limit),
        interval: interval,
    }
    
    // 定期补充令牌
    go func() {
        ticker := time.NewTicker(interval)
        defer ticker.Stop()
        for {
            <-ticker.C
            select {
            case rl.tokens <- struct{}{}:
            default:
                // 令牌桶已满
            }
        }
    }()
    
    return rl
}

func (rl *RateLimiter) Allow() bool {
    select {
    case <-rl.tokens:
        return true
    default:
        return false
    }
}
``````

## 性能对比：空结构体 vs bool

让我们通过一个简单的对比来看看空结构体的优势：

``````go
func memoryUsageDemo() {
    const size = 1000000
    
    // 使用bool作为map值
    boolMap := make(map[int]bool, size)
    for i := 0; i < size; i++ {
        boolMap[i] = true
    }
    
    // 使用空结构体作为map值
    emptyMap := make(map[int]struct{}, size)
    for i := 0; i < size; i++ {
        emptyMap[i] = struct{}{}
    }
   
}
``````
> bool 在 map 中仍然会占用存储空间，
而 struct{} 的 value 在 map bucket 中是 真正的零内存占用。
## 总结

**只要你只关心「有没有」，不关心「是什么」， 那你直接无脑优先考虑 `struct{}`。**



面试官彩蛋：

Q：为什么 Go 官方推荐用 `chan struct{}` 而不是 `chan bool`？