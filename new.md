# **这些 Go 1.26 新特性，专治你写过的那些丑代码**

前两天我们已经聊过 Go 1.26 最出圈的两件事：

- 新默认 GC：**Green Tea GC**
- 消灭辅助函数的：**new(expr)**

其实，如果你只盯着这两点，你就低估了这个版本。

Go 1.26 真正厉害的地方在于：**开始在标准库层面，认真帮你少写 Bug。**

今天就给大家整理一份「Go 1.26 生产力全家桶」，看看有哪些改动，让你写代码更安全、更优雅、更高效。

##  `errors.AsType`：错误处理终于进化了

说实话，这样的代码我写了无数次：

```go
var target *AppError
if errors.As(err, &target) {
    fmt.Println(target.Message)
}
```

这写法有两个不爽的点：
1. **先声明后使用**：得先声明个变量，再传指针地址。
2. **非类型安全**：`errors.As` 的第二个参数是 `any`，万一你传错了（比如没传指针），运行时直接 **Panic**。

### **Go 1.26 新写法：**

```go
if target, ok := errors.AsType[*AppError](err); ok {
    fmt.Println(target.Message)
}
```

好处：

- **类型安全**：编译期检查，避免传错类型。
- **作用域干净**：变量只在 if 内有效。
- **性能更稳**：官方 benchmark 显示，在高频错误判断场景下，性能约为 errors.As 的 **3 倍**。

## `netip.Prefix.Compare`：子网排序不用再手写了

如果你处理过 CIDR 列表（比如路由表、ACL、白名单），

一定写过这种排序代码：

比 IP → 比 mask → 再比长度。

Go 1.26 为 netip.Prefix 增加了 Compare 方法，现在可以直接：

```go
prefixes := []netip.Prefix{
    netip.MustParsePrefix("10.1.0.0/16"),
    netip.MustParsePrefix("10.0.0.0/16"),
}
// 一行代码搞定排序
slices.SortFunc(prefixes, netip.Prefix.Compare)
```

排序规则遵循 IANA 约定：

1. 先比IP家族(IPv4 < IPv6)
2. 再比网络地址
3. 再比前缀长度

这种 API 的价值在于：

> **把“大家都会写错一次”的逻辑，变成标准库能力。**

##  `slog.MultiHandler`：日志多路输出“官方方案”

以前用 slog 的人，十个里九个自己封装 MultiHandler。

现在官方直接提供：

```go
multi := slog.NewMultiHandler(
    slog.NewJSONHandler(file, nil),
    slog.NewTextHandler(os.Stdout, nil),
)
logger := slog.New(multi)
```

更细节的是：

如果某个 handler 出错，它会：

- 继续执行其他 handler
- 把所有错误聚合返回

## `bytes.Buffer.Peek`：写解析器更优雅了

以前想“偷看”缓冲区前几个字节，但不消费：

``````go
buf := bytes.NewBufferString("I love Go")
next := buf.Bytes()[0]  // 或手动切片、管理 offset
``````

Go 1.26 新方法：

```go
buf := bytes.NewBufferString("I love Go")
next, _ := buf.Peek(1) // next == "I"
```

- **缓冲区指针不动**
- 适合协议解析、编译器前端、状态机、流式处理

💡 **升级建议**：

写业务逻辑用得少，但写底层库/协议解析/流处理必备。

## 泛型终于能“约束自己”了：递归类型约束

我们先回顾一下，泛型常见写法：

``````go
type List[T any] struct{}
func Reverse[T any](s []T)
``````

可以加类型约束：

``````go
type Map[K comparable, V any] struct{}

func Compact[S ~[]E, E comparable](s S) S
``````

**但以前有一个能力是缺失的：**❌ 类型约束不能引用“自己”

比如这种写法在 Go 1.25 之前是非法的：

``````go
type T[P T[P]] struct{}
``````

编译器会直接报错：

``````shell
invalid recursive type: T refers to itself
``````

### **Go 1.26：允许递归类型约束**

现在可以写成这样：

``````go
type Ordered[T Ordered[T]] interface {
    Less(T) bool
}
``````

这行代码表达了一件以前做不到的事：

> **T 必须能和“同类型的 T”比较大小**

也就是说：

- 类型参数和自身建立了约束关系

- 泛型不再只是“外部限定”，而是**自我限定**

### **这有什么用？**

想象你要写一个**通用容器**，比如 Tree、Heap、SkipList。

容器里存的元素必须能**互相比较**，才能排序或组织。

以前，你只能这样写：

``````go
type Ordered interface {
    ~int | ~float64 | ~string
}
``````

问题：

- **枚举型**：只能列出内建类型
- **封闭**：用户自定义类型无法直接用
- **不灵活**：只能做有限的泛型容器

Go 1.26 递归类型约束允许你写：

``````go
type Ordered[T Ordered[T]] interface {
    Less(T) bool
}
``````

意思是：

- **T 可以是任意类型**，只要实现了 Less(T) 方法
- **开放的**：自定义类型也可以用
- **基于行为**：只看它能不能比较，而不是具体是什么类型

举个例子：

``````go
type Tree[T Ordered[T]] struct {
    nodes []T
}

type MyInt int

func (a MyInt) Less(b MyInt) bool { return a < b }

t := Tree[MyInt]{}
``````

- Tree 可以存 MyInt、netip.Addr 或任何实现了 Less 的类型
- 不再局限 int / string / float64

💡 **直观理解**：

以前泛型是**模板工具** → 只套固定类型

现在泛型是**抽象结构** → 只要你能比较，就能用

> 等1.26 出来我一定第一时间升级把数据库封装的重写一下

##  一些不显眼，但很值钱的优化

- **CGO 与 Syscall 加速**：得益于内部寄存器的优化，跨语言调用和系统调用变得更轻量了。
- **内存分配器升级**：在多核争用场景下，内存分配的吞吐量进一步提升。
- **编译器改进**：更多逃逸场景被消除，减少 GC 压力。

这些不会写在你代码里，但会体现在**延迟、吞吐、抖动**上。

## 总结：值不值得升？

Go 1.26 是一次**全方位的“开发者体验”升级**。
- `new(expr)` 让你代码变短。
- `errors.AsType` 让你更安全、更高效。
- `Green Tea GC` 让你的性能更稳定。
- 递归类型约束：泛型终于能表达“同类关系”

**我的建议是：正式版发布后，闭眼冲！**

**你觉得 Go 1.26 最实用的改动是哪个？欢迎留言区讨论！**
