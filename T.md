# 从 [int] 到“彻底隐形”的五年进化史

还记得 Go 1.18 泛型刚发布时的场景吗？

那时候不仅很多库不支持，连我们自己写代码时，为了让编译器“闭嘴”，都不得不写上一堆繁琐的类型参数：

```go
Min[int](a, b)
```

现在的 Go 泛型（Go 1.23+），就像一位整容成功的“隐形人”。

它依然在那里，但你几乎感觉不到它的存在。

今天我们就来聊聊 Go 泛型的这段“进化论”：

从早期的“显式调用”到如今的“智能推断”，看它是如何一步步“消失”的。

## 史前时代：`interface{}` 的噩梦

在泛型出现之前，如果想写一个通用的 `Min` 函数，我们要么写

-  `MinInt`
- `MinFloat`，
- `MinInt64`
- ......

要么,就只能祭出 `interface{}` 大法，然后疯狂断言：

```go
func Min(a, b interface{}) interface{} {
    switch a.(type) {
    case int:
        // ...大量的重复逻辑
    }
    return nil
}
```

这种代码不仅丑，而且慢，还没类型安全，工程师每次看都想哭。

## 青铜时代（Go 1.18）：有泛型，但你得显式告诉我

Go 1.18 终于带来了泛型。

我们欢呼雀跃，然后写出了这样的代码：

```go
// Go 1.18，需要手动枚举类型
func Min[T int | int32 | int64](a, b T) T {
    if a < b {
        return a
    }
    return b
}

func main() {
    x := Min[int](10, 20) // 必须显式指定类型
    fmt.Println(x)         // 10
}
```

通用性问题解决了，但新的问题也来了。

那个 [int]，就像黏在鞋底的口香糖：

**你明知道它是多余的，但就是甩不掉。**

每次调用都得显式告诉编译器：

> “听好了，这是俩 int，别自己瞎猜。”

说白了，这个阶段的泛型更像是：

**你在教编译器写代码，而不是写给人看的代码。**

## 白银时代（Go 1.20+）：constraints.Ordered 的初现

到了 `Go 1.20`,constraints.Ordered 出现在标准库里，泛型写法可以更简洁：

``````go
import "golang.org/x/exp/constraints"

// Go 1.20+
func Min[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}
func main() {
    x := Min(10, 20) // 编译器自动推断类型 T=int
    fmt.Println(x)    // 10
}
``````

你会发现：

- [int] 不见了
- 调用终于“像正常代码”了

但别高兴太早

这时候的类型推断还很“保守”

一旦遇到复杂结构体、接口约束、或者自定义类型——**你还是得乖乖把类型写出来。**

## 黄金时代（Go 1.21+）: 约束推断的魔法

Go 1.21 引入了**约束类型推断（Constraint Type Inference）**，

泛型第一次开始“自己动脑子”。

```go
func Scale[S ~[]E, E constraints.Integer](s S, c E) S {
	result := make(S, len(s))
	for i, v := range s {
		result[i] = v * c
	}
	return result
}
```

这里有两个泛型参数：

- S：切片类型
- E：元素类型

如果你定义了一个自定义类型：

```go
type Point []int32
var p Point = []int32{1, 2}
```

**在早期 Go 版本中，你大概率要这么写：**

``````go
Scale[Point, int32](p, 2)
``````

而现在，你只需要：

``````go
Scale(p, 2)
``````

编译器会自动推断出：

- S = Point
- 再根据 S ~[]E
- 顺藤摸瓜推导出 E = int32

这是第一次，Go 泛型真正做到：**“由整体推局部”。**

## 现代战争（Go 1.23+）: 彻底隐形

到了 Go 1.23，泛型已经不满足于“少写一点”，

而是开始 **主动从代码里消失**。

### 1. 匿名函数的类型推断

以前如果我们把一个匿名函数传给泛型函数，往往需要显式写出参数类型。现在，上下文推断更加强大。

```go
func Map[T, U any](s []T, f func(T) U) []U { ... }

nums := []int{1, 2, 3}

// 以前需要在 func(i int) 里写 int
// 现在很多场景下，编译器能根据 nums 的元素类型 T=int，
// 自动推断出 f 的入参必须是 int
Map(nums, func(i int) string { return fmt.Sprint(i*2) })
```

你几乎不再关心 T 是什么。

**上下文已经帮你把一切都定好了。**

### 2. **迭代器与** **range over func**

Go 1.23 引入了 iter 包和 range over func，让泛型与迭代器无缝融合：

```go
import "slices"

func main() {
    // slices.Values 返回一个迭代器 iter.Seq[int]
    seq := slices.Values([]int{100, 200, 300})
    
    // slices.Collect[E any](seq iter.Seq[E]) []E
    // 这里完全不需要指定类型，编译器自动推断出 E 是 int，返回值是 []int
    collected := slices.Collect(seq)
    
    fmt.Println(collected) // [100 200 300]
}
```

这里没有 [T]，

没有类型声明，

甚至没有“我在用泛型”的感觉。

但它依然是 **100% 静态类型安全**。

> 泛型已经退居幕后，
> 编译器站到了舞台中央。

## 总结：**进化的终点，是“你感觉不到它”**

从 Go 1.18 到 Go 1.23+，

Go 泛型的进化路线非常清晰：

> **把复杂留给编译器，把简洁留给开发者。**

现在的 Go 代码，看起来越来越像 Python、TypeScript，

但它依然保持着 Go 引以为傲的性能和安全性。

当你写下：

``````go
slices.Sort(list)
``````

却完全不去想 T 是什么的时候，

并不是你变懒了——

是编译器在背后，把复杂替你扛走了。

也许这就是 Go 泛型最理想的状态：

它还在那里，

但你已经不需要再注意它了。

最后我想说：**Go 泛型，你变了，变得让我快认不出来了——但我很喜欢。**
你更喜欢显式的 Go 泛型，还是智能推断的隐形风格？
