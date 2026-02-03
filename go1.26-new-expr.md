# 🚀 Go 1.26 新特性：`new(expr)`  
## 一个被严重低估的小改动，正在悄悄提升你的性能

很多 Go 开发者看到新版本特性时，第一反应往往是：

> “这不就是语法糖吗？”

但 Go 1.26 对 `new` 的这个改动，  
属于那种：

> **改动很小，收益很大，用上就回不去的特性。**

一句话概括它带来的变化：

- ✅ 代码更简洁  
- ✅ 不用再写一堆 `xxxPtr`  
- ✅ 更少堆分配  
- ✅ 性能可提升约 30%  

---

## 一个存在十几年的老痛点

在 Go 中，我们经常会定义这种结构体：

```go
type Config struct {
    Port *int
    Name *string
}
```

初始化时却很难受：

```go
func intPtr(v int) *int {
    return &v
}
func strPtr(v string) *string {
    return &v
}

cfg := Config{
    Port: intPtr(8080),
    Name: strPtr("service-a"),
}
```

根本原因就是**Go 不能对字面量取地址**

```go
&42      // ❌
&"abc"   // ❌
```



## Go 1.26：`new` 可以接受`任意表达式`

```go
p := new(expr)
```

`expr` 可以是：

- 字面量  
- 函数返回值  
- 计算表达式  
- 任意有确定类型的值  

比如：

```go
p1 := new(42)
p2 := new("hello")
p3 := new(time.Now())
p4 := new(a + b)
```

语义等价于：

```go
tmp := expr
p := &tmp
```


## 最直观的感受就是：结构体初始化更干净

旧写法：

```go
cfg := Config{
    Port: intPtr(8080),
    Name: strPtr("service-a"),
}
```

新写法：

```go
cfg := Config{
    Port: new(8080),
    Name: new("service-a"),
}
```

语义非常自然。


## 函数返回值也能直接变成指针

```go
func defaultPort() int {
    return 9000
}

cfg := struct {
    Port *int
}{
    Port: new(defaultPort()),
}
```

不需要中间变量，也不需要辅助函数。

---

## 其实我们之前已经“提前用过”这个特性了

如果你用的是go 1.23+，很可能用过这种写法：

```go
func ptr[T any](data T) *T {
    return &data
}
```

然后在代码里这样用：

```go
Timeout: ptr(3 * time.Second),
Retry:   ptr(5),
Name:    ptr("order-service"),
```

它解决的本质问题和 `new(expr)` 是一样的：

> **“我想要一个字面量的指针”**

从语义上看：

```go
ptr(100)
new(100)
```

几乎没有区别。

可以说：

- `ptr[T]` 是工程师实现的“民间版本”  
- `new(expr)` 是语言层面的“官方版本”

---

## `ptr[T]` 和 `new(expr)`对比性能如何？

你的 `ptr[T]`：

```go

//go:noinline
func ptr[T any](data T) *T {
	return &data
}

func main() {
	a := new(2)
	b := ptr(2)
	println(a)
	println(b)
}

```

是一个普通函数调用，并且返回了参数地址。
编译器往往无法证明这个指针不会被长期持有，于是：

> 👉 **`data` 很容易逃逸到堆上**

你可以用：

```bash
 go build -gcflags="-m"
```

常见输出是：

```
./main.go:11:17: moved to heap: data
```

也就是说：

```go
ptr(2)
```

背后很可能是一次堆分配。

---

而 `new(expr)` 是编译器内建路径：

```go
p := new(2)
```

编译器非常清楚：

- expr 是一个值  
- 生命周期确定  
- 没有函数调用  
- 没有逃逸不确定性  

因此在很多场景下：

> 可以直接分配在栈上  
> 甚至被完全优化掉  
> 不产生堆分配  

所以我们猜一猜他们性能会有差距吗？

---

## 真实对比：`ptr` vs `new(expr)`

### 泛型 `ptr` 版本：

```go
//go:noinline
func ptr[T any](v T) *T {
    return &v
}
func BenchmarkPtr(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = ptr(100)
    }
}
```

### `new(expr)` 版本：

```go
func BenchmarkNew(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = new(100)
    }
}
```

### 基准测试结果
```shell
 zouyuxi@zouyx  ~/workspace/go-behavior/table/1.26: go test -bench=. -benchmem -benchtime=3s                                   
goos: darwin
goarch: arm64
pkg: t6
cpu: Apple M1
BenchmarkPtr-8          306165134               11.77 ns/op            8 B/op          1 allocs/op
BenchmarkNew-8          1000000000               0.5133 ns/op          0 B/op          0 allocs/op
PASS
ok      t6      5.368s

```

结论很明确：

> 🔥 更快  
> 🔥 少一次堆分配  
> 🔥 更低 GC 压力  

但是实际上编译器帮我们优化了这一部分,我们刚刚禁用了内联优化，实际上编译器会帮我们实行内联优化，我们不禁用试试看

``````go
//go:noinline
func ptr[T any](v T) *T {
    return &v
}
func BenchmarkPtr(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = ptr(100)
    }
}
``````

### 基准测试结果

``````
 zouyuxi@zouyx  ~/workspace/go-behavior/table/1.26: go test -bench=. -benchmem -benchtime=3s 
goos: darwin
goarch: arm64
pkg: t6
cpu: Apple M1
BenchmarkPtr-8          1000000000               0.5119 ns/op          0 B/op          0 allocs/op
BenchmarkNew-8          1000000000               0.5117 ns/op          0 B/op          0 allocs/op
PASS
ok      t6      1.137s

``````

是不是发现性能几乎一致了

## 还有一个隐藏收益：避免“大对象逃逸”

```go
type Big struct {
    Data [10240]byte
    Flag int
}

b := Big{Flag: 1}

x := struct {
    Flag *int
}{
    Flag: new(b.Flag),
}
```

如果你写：

```go
Flag: &b.Flag
```

可能导致整个 `b` 逃逸到堆。

而：

```go
new(b.Flag)
```

只复制一个 `int`，  
更容易被编译器优化。


## 迁移建议（非常工程化）

如果你现在项目里大量使用：

```go
ptr(x)
```

在 Go 1.26 稳定后，可以逐步替换为：

```go
new(x)
```

收益：

- ✅ 少维护一个工具函数  
- ✅ 代码更直观  
- ✅ 更少逃逸  
- ✅ 更少堆分配  
- ✅ 更容易被编译器优化  

尤其适合：

- 配置结构体  
- DTO 构造  
- 高频初始化路径  

---

## 十、总结

Go 1.26 的 `new(expr)`：

- 看起来只是一个语法扩展  
- 实际解决了一个十几年的老痛点  
- 顺手带来了性能收益  

它不是那种“炫技型特性”，而是：

> **你每天写业务代码都会用到的特性。**

如果你：

- 经常写 `*int`、`*string`  
- 经常写配置结构  
- 经常用 `ptr[T]`  
- 在意逃逸和 GC  

那么：

> `new(expr)`  
> 很可能成为你用得最多的 Go 新语法之一。
