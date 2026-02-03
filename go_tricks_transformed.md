# 一个写了 5 年 Go 的人，告诉你哪些代码别再写了

今天分享 12 个我在生产项目中常用的 Go 语言技巧，每个都能让你的代码更简洁、高效，还能避免很多常见坑。

## 1. 用 defer 优雅地跟踪函数执行时间

你是不是经常写这种代码来统计函数执行时间？

```go
func doSomething() {
    start := time.Now()
    // 业务逻辑
    fmt.Printf("耗时: %v\n", time.Since(start))
}
```

看起来没问题，但如果函数有多个返回路径，你就得在每个 return 前都加一遍统计代码，容易漏写。

**技巧：用 defer 实现统一的执行时间跟踪**

```go
func doSomething() {
    defer func(start time.Time) {
        fmt.Printf("耗时: %v\n", time.Since(start))
    }(time.Now())

    // 业务逻辑
    // 即使有多个 return，统计代码也会执行
}
```

这样不管函数怎么返回，defer 都会确保执行时间被正确统计。

## 2. 双阶段 defer：初始化和清理一步搞定

在处理资源时，我们通常需要先初始化，再使用，最后清理。比如数据库连接：

```go
func processData() error {
    // 1. 初始化
    db, err := sql.Open("mysql", "dsn")
    if err != nil {
        return err
    }

    // 2. 使用
    _, err = db.Exec("sql")
    if err != nil {
        db.Close() // 这里需要手动关闭
        return err
    }

    db.Close() // 这里也要手动关闭
    return nil
}
```

这种写法不仅啰嗦，还容易在错误处理时漏写 Close。

**技巧：双阶段 defer 处理资源**

```go
func processData() error {
    db, err := sql.Open("mysql", "dsn")
    if err != nil {
        return err
    }
    closed := false
    defer func() {
        if !closed {
            db.Close()
        }
    }()

    // 使用
    _, err = db.Exec("sql")
    if err != nil {
        return err
    }

    closed = true
    db.Close() // 主动关闭
    return nil
}
```

通过一个标志位，defer 会确保资源被正确清理，不管函数是正常返回还是出错。

## 3. 切片预分配：避免频繁扩容

Go 语言的切片会自动扩容，但每次扩容都会分配新内存并拷贝数据，严重影响性能。

**技巧：明确知道切片大小的话，提前预分配容量**

```go
// 普通写法
func collectData() []int {
    var data []int
    for i := 0; i < 1000; i++ {
        data = append(data, i)
    }
    return data
}

// 优化后
func collectData() []int {
    data := make([]int, 0, 1000) // 预分配 1000 的容量
    for i := 0; i < 1000; i++ {
        data = append(data, i)
    }
    return data
}
```

对于已知大小的切片，预分配容量能避免多次内存分配，性能提升显著。

## 4. 方法链式调用：代码更流畅

Go 语言虽然不支持传统的方法链式调用，但我们可以通过返回 receiver 来实现。

**技巧：让方法返回 receiver，实现链式调用**

```go
type Calculator struct {
    value int
}

func (c *Calculator) Add(n int) *Calculator {
    c.value += n
    return c
}

func (c *Calculator) Multiply(n int) *Calculator {
    c.value *= n
    return c
}

func (c *Calculator) Result() int {
    return c.value
}

// 使用
func main() {
    res := &Calculator{0}.Add(10).Multiply(2).Result() // 20
    fmt.Println(res)
}
```

链式调用让代码更简洁易读，减少了临时变量的声明。

## 5. 切片转数组：从 unsafe 到原生支持

早期 Go 版本中，切片转数组需要用 unsafe 包。Go 1.17 开始支持切片转**数组指针**，Go 1.20 进一步支持切片直接转**数组**。

**技巧：使用原生语法转换切片**

```go
// Go 1.17 之前：需要 unsafe
import "unsafe"

func main() {
    s := []int{1, 2, 3, 4}
    p := (*[4]int)(unsafe.Pointer(&s[0]))
    fmt.Println(p) // &[1 2 3 4]
}

// Go 1.17+：切片转数组指针
func main() {
    s := []int{1, 2, 3, 4}
    p := (*[4]int)(s) // 直接转换为数组指针
    fmt.Println(p)    // &[1 2 3 4]
}

// Go 1.20+：切片直接转数组（会拷贝数据）
func main() {
    s := []int{1, 2, 3, 4}
    a := [4]int(s)   // 直接转换为数组，注意这里会拷贝
    fmt.Println(a)   // [1 2 3 4]
}
```

注意：转数组指针不拷贝数据，转数组会拷贝数据，按需选择。

## 6. _ 导入包：只执行初始化代码

有时候我们只需要执行包的初始化代码，不需要使用包中的任何函数或变量。

**技巧：使用 _ 导入包**

```go
import (
    _ "github.com/go-sql-driver/mysql" // 只执行初始化代码
    "database/sql"
)

func main() {
    db, err := sql.Open("mysql", "dsn") // 可以正常使用 mysql 驱动
    // ...
}
```

这种方式常用于数据库驱动、日志库等需要在程序启动时初始化的包。

## 7. . 导入包：简化代码但需谨慎

有时候我们会频繁使用某个包中的函数，每次都写包名会很啰嗦。

**技巧：使用 . 导入包**

```go
import (
    . "fmt"
)

func main() {
    Println("Hello, World!") // 不需要写 fmt.Println
}
```

但要谨慎使用，过度使用会导致代码可读性下降。

## 8. errors.Join()：合并多个错误

在处理多个操作时，我们通常需要合并所有错误信息。

**技巧：使用 Go 1.20 新增的 errors.Join()**

```go
func processFiles(files []string) error {
    var errs []error

    for _, file := range files {
        if err := processFile(file); err != nil {
            errs = append(errs, err)
        }
    }

    if len(errs) > 0 {
        return errors.Join(errs...)
    }
    return nil
}

// 输出的错误信息会包含所有文件的错误
// "open file1.txt: no such file or directory; open file2.txt: permission denied"
```

errors.Join() 会将多个错误合并成一个，保留所有错误信息。

## 9. 接口实现的编译时检查

Go 语言的接口是隐式实现的，有时候我们会不小心漏掉某个方法。

**技巧：使用编译时检查确保接口实现**

```go
type MyInterface interface {
    Method1()
    Method2()
}

type MyType struct{}

func (t *MyType) Method1() {}
func (t *MyType) Method2() {}

// 编译时检查 MyType 是否实现了 MyInterface
var _ MyInterface = (*MyType)(nil)
```

如果 MyType 缺少任何一个方法，编译时就会报错。

## 10. 泛型实现类似三元运算符的功能

Go 语言一直没有三元运算符，但我们可以用泛型实现类似的功能。

**技巧：使用泛型实现三元运算符**

```go
func If[T any](condition bool, trueVal, falseVal T) T {
    if condition {
        return trueVal
    }
    return falseVal
}

// 使用
func main() {
    x := 10
    y := If(x > 5, "大于 5", "小于等于 5") // "大于 5"
    fmt.Println(y)
}
```

虽然语法不如真正的三元运算符简洁，但比写 if-else 语句要简洁得多。

## 11. 避免裸参数：代码更易读

使用裸参数的函数调用可读性很差，维护者需要去函数定义里看参数的含义。

**技巧：使用结构体或变量提高代码可读性**

```go
// 裸参数调用（不好）
price := calculatePrice(100, 0.1, true) // 这些参数分别代表什么？

// 优化后：使用结构体
type PriceParams struct {
    BasePrice float64
    Discount  float64
    IsMember  bool
}

func calculatePrice(p PriceParams) float64 {
    // ...
}

// 调用时参数含义一目了然
func main() {
    price := calculatePrice(PriceParams{
        BasePrice: 100,
        Discount:  0.1,
        IsMember:  true,
    })
    fmt.Println(price)
}
```

优化后的代码一眼就能看出每个参数的含义。

## 12. 检查接口是否真正为 nil

Go 语言的接口包含两个部分：类型（type）和值（value）。只有当两者都为 nil 时，接口才等于 nil。这是很多人踩过的坑。

**技巧：理解接口的 nil 陷阱**

```go
type MyError struct{}

func (e *MyError) Error() string { return "my error" }

func doSomething(fail bool) error {
    var err *MyError = nil  // 具体类型的 nil 指针
    if fail {
        err = &MyError{}
    }
    return err  // 返回时被包装成接口，类型部分是 *MyError，不是 nil！
}

func main() {
    err := doSomething(false)  // 传入 false，err 值是 nil
    if err != nil {
        fmt.Println("有错误")  // 会执行！因为接口的类型部分不是 nil
    }
}
```

**正确做法：函数返回接口类型时，直接返回 nil，不要返回具体类型的 nil 指针**

```go
func doSomething(fail bool) error {
    if fail {
        return &MyError{}
    }
    return nil  // 直接返回 nil，而不是返回一个 nil 指针
}
```

这个坑在返回 error 接口时特别常见，一定要注意。

## 总结

这 12 个技巧都是我在生产项目中的经验总结，每个都能解决实际开发中的痛点。掌握这些技巧，你的 Go 代码会变得更简洁、高效，还能避免很多常见的坑。

技术学习没有捷径，只有不断踩坑、总结、实践，才能真正提高。希望这些技巧能帮你少走一些弯路。

最后，如果你觉得这些技巧有用，别忘了转发给你的同事，一起提升代码质量。