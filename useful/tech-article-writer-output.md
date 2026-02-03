# 我用这12个Go技巧，解决了90%的日常开发问题

作为一名写了3年Go的后端程序员，我最近整理了自己日常开发中最常用的12个小技巧。这些技巧不是什么高大上的架构设计，而是能直接帮你节省时间、减少bug的实用方法。

话不多说，直接上干货！

## 1. 一行代码统计函数执行时间

你是不是经常需要统计某个函数的执行时间？以前我会写一堆重复的代码：

```go
func foo() {
    start := time.Now()
    defer func() {
        fmt.Printf("foo took %v\n", time.Since(start))
    }()
    // 函数逻辑
}
```

后来我发现可以用 `defer` 结合自定义函数来简化：

```go
func TrackTime(start time.Time, name string) {
    elapsed := time.Since(start)
    log.Printf("%s took %v\n", name, elapsed)
}

// 使用
func foo() {
    defer TrackTime(time.Now(), "foo")
    // 函数逻辑
}
```

这行代码虽然简单，但能帮你在调试和优化性能时节省大量时间。

## 2. 两阶段defer：初始化与清理的完美结合

有时候我们需要在函数开始时做一些初始化工作，比如打开文件、建立数据库连接，然后在函数结束时确保这些资源被正确释放。这时候可以用两阶段 `defer`：

```go
func processFile(path string) error {
    // 初始化
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    // 第一阶段defer：确保文件关闭
    defer func() {
        if err := f.Close(); err != nil {
            log.Printf("Error closing file: %v", err)
        }
    }()

    // 第二阶段defer：可以根据函数执行结果调整清理逻辑
    var cleanup func() error = nil
    defer func() {
        if cleanup != nil {
            if err := cleanup(); err != nil {
                log.Printf("Error cleaning up: %v", err)
            }
        }
    }()

    // 函数逻辑...

    return nil
}
```

这种模式能让你的代码更简洁、更安全，避免遗漏资源释放。

## 3. 预分配切片空间提升性能

Go中的切片底层是数组，当切片容量不足时会自动扩容，每次扩容都会分配新的内存并复制数据。如果能提前知道切片的预期大小，预分配空间能显著提升性能：

```go
// 不好的写法
func processData(data []int) []int {
    var result []int
    for _, v := range data {
        if v > 0 {
            result = append(result, v)
        }
    }
    return result
}

// 优化后的写法
func processData(data []int) []int {
    result := make([]int, 0, len(data)) // 预分配与原切片相同的容量
    for _, v := range data {
        if v > 0 {
            result = append(result, v)
        }
    }
    return result
}
```

我曾在处理大量数据时用这个技巧把处理时间从10秒降到了3秒！

## 4. 函数指针接收器返回自身实现链式调用

如果你写过一些工具类或建造者模式的代码，应该会喜欢这个技巧。通过让方法返回自身的指针，你可以实现链式调用：

```go
type Config struct {
    Host string
    Port int
    Timeout time.Duration
}

func (c *Config) SetHost(host string) *Config {
    c.Host = host
    return c
}

func (c *Config) SetPort(port int) *Config {
    c.Port = port
    return c
}

func (c *Config) SetTimeout(timeout time.Duration) *Config {
    c.Timeout = timeout
    return c
}

// 使用
func main() {
    cfg := &Config{}
    cfg.SetHost("localhost").SetPort(8080).SetTimeout(30 * time.Second)
    fmt.Printf("Config: %+v\n", cfg)
}
```

这样的代码更易读，也更符合现代人的编程习惯。

## 5. Go 1.20新特性：切片转数组/数组指针

Go 1.20 允许我们将切片直接转换为数组或数组指针，这在处理固定大小的数据时非常方便：

```go
func main() {
    // 切片转数组
    s := []byte("Hello, World!")
    var arr [5]byte = [5]byte(s[:5])

    // 切片转数组指针（更常用）
    ptr := (*[5]byte)(s[:5])
    fmt.Printf("First 5 bytes: %v\n", ptr)
}
```

需要注意的是，这种转换要求切片的长度必须大于等于目标数组的长度，否则会导致运行时panic。

## 6. import下划线：只执行包初始化，不使用包内容

有时候我们只需要导入一个包来执行它的初始化函数，而不需要使用包中的任何内容。这时候可以在import语句前加下划线：

```go
import (
    _ "github.com/mattn/go-sqlite3" // 只执行sqlite3驱动的初始化
    "database/sql"
)

func main() {
    db, err := sql.Open("sqlite3", "test.db")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()
}
```

这种技巧常用于数据库驱动、网络协议等需要在程序启动时注册自身的包。

## 7. import点号：省略包名访问导出标识符

如果你经常使用某个包中的函数，可以用import点号来省略包名：

```go
import . "fmt"

func main() {
    Println("Hello, World!") // 不需要写fmt.Println
}
```

但这个技巧要谨慎使用，因为它会让代码的可读性下降，尤其是在多个包有同名函数时。

## 8. Go 1.20新特性：errors.Join合并多个错误

Go 1.20新增了`errors.Join`函数，可以将多个错误合并成一个：

```go
func processFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }

    content, err := io.ReadAll(f)
    if err != nil {
        return errors.Join(err, f.Close())
    }

    if err := f.Close(); err != nil {
        return err
    }

    // 处理文件内容...

    return nil
}
```

这个技巧能让你更清晰地了解错误链，有助于快速定位问题。

## 9. 编译时检查接口实现

在Go中，接口是隐式实现的，这意味着我们不需要像Java那样显式地声明类实现了哪个接口。但这也带来了一个问题：如果我们修改了结构体的方法签名，可能会导致它不再满足接口要求，但编译器不会报错。

解决方法是在代码中添加一个编译时检查：

```go
type MyInterface interface {
    DoSomething() error
    DoAnotherThing() string
}

type MyStruct struct {
    // 字段
}

// 编译时检查MyStruct是否实现了MyInterface
var _ MyInterface = (*MyStruct)(nil)
```

如果`MyStruct`没有完全实现`MyInterface`中的所有方法，编译器会报错。

## 10. 泛型实现三元操作

Go语言没有内置三元运算符，但我们可以用泛型实现一个通用的三元操作函数：

```go
func If[T any](condition bool, trueVal, falseVal T) T {
    if condition {
        return trueVal
    }
    return falseVal
}

// 使用
func main() {
    a := 10
    b := 20
    max := If(a > b, a, b)
    fmt.Printf("Max: %d\n", max)
}
```

虽然这不如内置三元运算符简洁，但在Go中已经是最接近的解决方案了。

## 11. 判断接口是否真nil

在Go中，判断接口是否为nil有时会让人困惑，因为接口包含两个部分：类型和值。只有当类型和值都是nil时，接口才是真nil：

```go
func IsInterfaceNil(v interface{}) bool {
    return v == nil || (reflect.ValueOf(v).Kind() == reflect.Ptr && reflect.ValueOf(v).IsNil())
}

func main() {
    var err error = nil
    fmt.Println(err == nil) // 输出：true

    var p *int = nil
    err = p
    fmt.Println(err == nil) // 输出：false
    fmt.Println(IsInterfaceNil(err)) // 输出：true
}
```

这个技巧在处理一些底层代码时非常重要，能避免一些隐蔽的bug。

## 12. 自定义Duration类型解析JSON

当你需要解析JSON中的时间字符串（如"30s"、"1h30m"）时，可以用自定义Duration类型：

```go
type Duration time.Duration

func (d *Duration) UnmarshalJSON(data []byte) error {
    str := strings.Trim(string(data), `"`)
    duration, err := time.ParseDuration(str)
    if err != nil {
        return err
    }
    *d = Duration(duration)
    return nil
}

func (d Duration) MarshalJSON() ([]byte, error) {
    return json.Marshal(time.Duration(d).String())
}

// 使用
type Config struct {
    Timeout Duration `json:"timeout"`
}

func main() {
    jsonStr := `{"timeout": "30s"}`
    var cfg Config
    if err := json.Unmarshal([]byte(jsonStr), &cfg); err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Timeout: %v\n", time.Duration(cfg.Timeout))
}
```

这个技巧能让你的代码更符合Go的编码风格，也更易读。

## 总结

以上12个Go语言技巧是我在日常开发中总结出来的，它们涵盖了性能优化、代码简洁性、错误处理、类型安全等方面。我希望这些技巧能帮助你提高开发效率，写出更优雅、更稳定的代码。

如果你有其他常用的Go技巧，欢迎在评论区分享！