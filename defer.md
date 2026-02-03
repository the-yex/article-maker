# Golang Defer:从基础到陷阱

在初学 Go 的过程中，defer 几乎是第一个让人眼前一亮的语言特性。

![defer1](https://www.helloimg.com/i/2026/01/02/6957bbab25b12.png)


但实际上，defer 远不止表面看起来那么简单。
它包含了许多容易让人踩坑的细节，也有不少在日常使用中很少被提及、却相当有意思的特性。

举个例子，defer 在实现层面实际上存在 三种类型：
-	open-coded defer
-	heap-allocated defer
-	stack-allocated defer

这三种 defer 在 性能表现 和 适用场景 上都有明显差异。如果你希望对程序进行性能优化，理解这些差别是非常有价值的。

在本文中，我们将从 defer 的基础用法讲起，逐步过渡到更高级的使用方式，并且会 稍微深入一点点，探究一些其内部实现细节。
## 什么是 defer?

在深入讨论之前，我们先快速回顾一下 defer 的基本概念。

在 Go 语言中，defer 是一个关键字，用于 **延迟执行函数调用**，直到其所在的函数执行结束为止。

```go
func main() {
    defer fmt.Println("hello")
    fmt.Println("world")
}
// 输出:
// world
// hello
```

在这段代码中，defer 语句会将 fmt.Println("hello") 安排在 main 函数即将结束时执行。因此，fmt.Println("world") 会被立即调用，world 也会最先被打印出来。

随后，由于使用了 defer，在 main 函数退出前的最后一步，hello 才会被输出。

这就好比提前安排了一个“收尾任务”，在函数即将返回时再执行。这种机制在清理资源时非常有用，例如：关闭数据库连接、释放互斥锁，或者关闭文件等。

```go
func doSomething() error {
    f, err := os.Open("phuong-secrets.txt")
    if err != nil {
        return err
    }
    defer f.Close()
    // ...
}
```

上面的代码很好地展示了 defer 的工作方式，但从实践角度来说，它其实并不是一个**理想的使用示例**。至于原因，大家先思考以下。

> “那为什么不直接把 f.Close() 放在函数末尾呢？”

这样做其实有几个很合理的理由:

- 我们把关闭操作放在打开操作附近,这样更容易理解逻辑,避免忘记关闭文件。我不想滚动一个函数来检查文件是否被关闭;这会分散我对主要逻辑的注意力。
- **即使函数中发生了 panic**，被 defer 注册的函数依然会被执行。

当 panic 发生时，Go 会对调用栈进行展开（stack unwinding），并按照特定的顺序执行所有已经注册的 defer 函数。

## Defer 是"栈式"执行的

当在同一个函数中使用多个 defer 语句时，它们会以 **栈（stack）** 的方式进行管理和执行，也就是说：**后注册的 defer，会先执行**。

```go
func main() {
    defer fmt.Println(1)
    defer fmt.Println(2)
    defer fmt.Println(3)
}
// 输出:
// 3
// 2
// 1
```

每当你调用一次 defer 语句时，实际上就是把对应的函数 **压入当前 goroutine 的 defer 链表顶部**，大致可以理解成下面这样的结构：

![defer1](https://www.helloimg.com/i/2026/01/02/6957bbab25b12.png)

当函数返回时，Go 会遍历这个链表，并按照上图所示的顺序依次执行每个 defer。

需要注意的是，它 **并不会执行整个 goroutine 中所有的 defer**，而只会执行 **当前返回函数注册的 defer**。这是因为同一个 goroutine 的 defer 链表中可能包含来自多个不同函数的 defer。

```go
func B() {
    defer fmt.Println(1)
    defer fmt.Println(2)
    A()
}

func A() {
    defer fmt.Println(3)
    defer fmt.Println(4)
}
```

也就是说，**只有当前函数（或当前栈帧）中注册的 defer 才会被执行**。

![](https://www.helloimg.com/i/2026/01/02/6957bf1dbfbb5.png)

不过，有一种典型情况会触发 当前 goroutine 中所有 defer 的遍历与执行，那就是 发生 panic 的时候。
## Defer、Panic 与 Recover

除了编译期错误之外，Go 还会出现许多 **运行时错误**，例如：
- 整数除以零
- 数组越界
- 空指针解引用
- 其他类似错误

这些错误会导致程序触发 **panic**。

在 Go 中，**panic** 会停止当前 goroutine 的执行，展开调用栈（stack unwind），并执行该 goroutine 中所有已注册的 defer，从而可能导致应用程序崩溃。

为了处理意外错误并避免程序崩溃，可以在 **defer 中调用 recover()**，从而重新获得对正在 panic 的 goroutine 的控制权。

```go
func main() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered:", r)
        }
    }()
    panic("This is a panic")
}
// 输出:
// Recovered: This is a panic
```

通常，人们会在 panic 中传递一个错误对象，然后通过 recover(...) 捕获它，但实际上，panic 中可以传递 **任何类型的值**，比如字符串、整数等。

在上面的示例中，**只有在 defer 的函数内部**才能调用 recover()。下面我再详细说明一下。

在实际代码中，我们经常看到一些关于 recover 的典型错误写法，我个人至少见过三种常见用法。

**第一种错误用法**是：直接把 recover 当作 defer 的函数来使用：

```go
func main() {
    defer recover()
    panic("This is a panic")
}
```

上面的代码仍然会触发 panic，这是 **Go 运行时的设计使然**。

`recover` 函数的作用是捕获 panic，但它必须在 **defer 的函数内部** 调用才能正常工作。

在底层实现中，我们调用的 recover 实际上对应的是 runtime.gorecover，它会检查 recover 是否在正确的上下文中调用，也就是必须发生在 **panic 发生时对应的 defer 函数内部**。

> “那是不是意味着我们不能在 defer 内部再嵌套的函数里使用 recover，比如这样写？”

```go
func myRecover() {
    if r := recover(); r != nil {
        fmt.Println("Recovered:", r)
    }
}

func main() {
    defer func() {
        myRecover()
        // ...
    }()
    panic("This is a panic")
}
```

没错，上面的代码并不会像你预期的那样工作。这是因为 recover **不是直接在 defer 函数中调用**，而是在其嵌套函数中调用的。

另一种常见错误是，试图捕获 **来自其他 goroutine 的 panic**：

```go
func main() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered:", r)
        }
    }()
    go panic("This is a panic")
    time.Sleep(1 * time.Second) // 等待 goroutine 完成
}
```

这说得通,对吧?我们已经知道 defer 链属于特定的 goroutine。如果一个 goroutine 可以干预另一个 goroutine 来处理 panic,那将会很困难,因为每个 goroutine 都有自己的栈。

不幸的是,在这种情况下,**如果不在该 goroutine 内部处理 panic，唯一的结果就是应用程序崩溃**。



## Defer 类型:**Heap-allocated、Stack-allocated 与 Open-coded defer**

当我们调用 defer 时，Go 编译器会创建一个名为 _defer 的结构体对象（defer object），它包含了 **延迟调用所需的所有信息**。

这个对象会被 **压入当前 goroutine 的 defer 链表**，正如前面所讨论的那样。

每当函数退出时（无论是正常返回还是因为错误退出），编译器都会确保调用 runtime.deferreturn。

这个函数负责：

1. 展开 defer 链表
2. 从 defer 对象中取出存储的信息
3. 按正确顺序执行所有延迟函数

**Heap-allocated defer** 和 **Stack-allocated defer** 的区别在于 defer 对象分配的位置。

在 Go 1.13 之前，所有的 defer 对象都是 **堆分配（heap-allocated）** 的。
![](https://www.helloimg.com/i/2026/01/02/6957c3fee9c9f.png)

而在 Go 1.22 中，如果在 循环中使用 defer，对应的 defer 对象将会 在堆上分配（heap-allocated）。
```go
func main() {
    for i := 0; i < unpredictableNumber; i++ {
        defer fmt.Println(i) // 堆分配 defer
    }
}
```


这里之所以需要 **堆分配**，是因为 defer 对象的数量在运行时可能会变化。堆分配可以确保程序 **无论函数中有多少 defer 或它们出现在什么位置**，都能够安全处理，而不会导致栈空间膨胀。

别紧张，虽然堆分配在性能上通常不理想，但 Go 会尝试通过 **defer 对象池** 来优化这一点。

Go 内部有两个对象池：

1. **逻辑处理器 P 的本地缓存池**：用于避免锁竞争
2. **全局缓存池**：由所有 goroutine 共享，goroutine 会先从全局池取对象，然后再放入对应 P 的本地池

> “那在 Go 1.22 中，如果在 if 语句里使用 defer 呢？这不是也很不确定吗？”

没错，你抓住重点了：在 if 语句中使用 defer 的行为确实可能比较难预测。

自 Go 1.13 起，defer **可以栈分配（stack-allocated）**。这意味着编译器会在栈上创建 _defer 对象，然后将其压入当前 goroutine 的 defer 链表。

如果 if 语句块内的 defer 仅被调用一次，且不在循环或其他动态上下文中，它就能享受 Go 1.13 引入的优化，也就是 **defer 对象会在栈上分配**。

```go
func testDefer(a int) {
    if a == unpredictableNumber {
        defer println("Defer in if") // 栈分配 defer
    }
    if a == unpredictableNumber+1 {
        defer println("Defer in if") // 栈分配 defer
    }
    for range a {
        defer println("Defer in for") // 堆分配 defer
    }
}
```

上面的结论在 **Go 1.23** 中依然成立。

根据 **Open-coded defer 提案** 的优化，在 cmd/go 二进制文件中，这种优化适用于 **370 个静态 defer 站点中的 363 个**。

因此，与之前全部 heap-allocated 的方式相比，这些 defer 站点的性能提升约 **30%**。

> “既然性能这么好，为什么还需要所谓的 ‘open-coded defer’ 呢？”

如果我们直接把 defer 放在函数末尾，性能会比另外两种方式更好。

截至 Go 1.13，大多数 defer 操作耗时约 **35ns**（相比 Go 1.12 的约 50ns 有明显下降）。而直接调用函数仅需 **约 6ns**。

你大概已经猜到了。

Go 会 **将 defer 调用直接内联（inline）到函数末尾**，并且在汇编代码中每个 return 语句前也会插入调用，但这种方式 **有一定限制条件**，只有满足条件的 defer 才会被应用。

回想前面的例子？我在这里再贴一遍，方便后续讨论：

```go
func testDefer(a int) {
    if a == unpredictableNumber {
        defer println("Defer in if") // 栈分配 defer
    }
    if a == unpredictableNumber+1 {
        defer println("Defer in if") // 栈分配 defer
    }
    for range a {
        defer println("Defer in for") // 堆分配 defer
    }
}
```

如果一个函数中至少存在一个 **堆分配（heap-allocated）的 defer**，那么该函数中的所有 defer 都 **不会被内联（inlined）或使用 open-coded defer**。

也就是说，要对上面的函数进行优化，我们应该 **移除或将堆分配的 defer 移到别处**。

```go
func testDefer(a int) {
    if a == unpredictableNumber {
        defer println("Defer in if") // 开放编码 defer
    }
    if a == unpredictableNumber+1 {
        defer println("Defer in if") // 开放编码 defer
    }
}
```

还有一点需要注意：**函数中 defer 的数量与 return 语句数量的乘积，需要 ≤ 15，才能使用 open-coded defer**。

这是因为我们会在 **每个 return 语句前插入 defer 调用**。如果函数有太多退出路径，二进制代码就会变得非常臃肿。
