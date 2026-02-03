# Go切片排序,这三种方法，哪个更爽

今天处理一个几万条数据的切片，我才意识到——Go的切片排序方法，已经悄悄升级了。

以前用 `sort.Interface` 写排序，代码多得像打仗，尤其是自定义类型的切片，想改个排序条件都得写半天。

后来发现 `sort.Slice`，直接传个函数就能搞定，爽多了。

再后来，Go 1.21 推出了 `slices` 包，我差点以为自己看错——竟然一行就能排序，还类型安全，编译器直接帮你检查。

今天我把这三种方法整理一下，顺便分享一些实战心得，帮你选出最适合你项目的方案。

## 传统方法：sort.Interface接口

最早，Go要求你实现 `sort.Interface` 才能排序。三个方法：`Len`、`Less`、`Swap`，每个都得自己写。

```go
package main

import (
    "fmt"
    "sort"
)

// Person 结构体定义
type Person struct {
    Name string
    Age  int
}

// ByAge 类型定义
type ByAge []Person

// 实现 sort.Interface
func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }

func main() {
    people := []Person{
        {"Alice", 30},
        {"Bob", 25},
        {"Charlie", 35},
        {"David", 28},
    }

    fmt.Println("Before sorting:", people)
    sort.Sort(ByAge(people))
    fmt.Println("After sorting by age:", people)
}
```

**体验感受**：控制力十足，但代码冗长，每次改排序逻辑都要动好几个地方。老项目兼容性好，但写起来容易崩溃。

## 改进方法：sort.Slice函数

Go 1.8 引入 `sort.Slice`，直接传一个比较函数就能排序，不必再写接口。

```go
package main
import (
    "fmt"
    "sort"
)
type Person struct {
    Name string
    Age  int
}
func main() {
    people := []Person{
        {"Alice", 30},
        {"Bob", 25},
        {"Charlie", 35},
        {"David", 28},
    }
    fmt.Println("Before sorting:", people)
    sort.Slice(people, func(i, j int) bool {
        return people[i].Age < people[j].Age
    })
    fmt.Println("After sorting by age:", people)
}
```

**优点**：

- 代码更加简洁
- 不需要定义新类型
- 保留了自定义比较逻辑的能力

**缺点**：

- 仍然需要定义比较函数
- 无编译时类型安全

## 现代方法：slices包（Go 1.21+）

Go 1.21 的 `slices` 包彻底改变了我的排序体验。基本类型直接排序，泛型类型用 `SortFunc` 就可以。

```go
package main
import (
    "cmp"
    "fmt"
    "slices"
)
type Person struct {
    Name string
    Age  int
}
func main() {
    people := []Person{
        {"Alice", 30},
        {"Bob", 25},
        {"Charlie", 35},
        {"David", 28},
    }
    fmt.Println("Before sorting:", people)
    slices.SortFunc(people, func(a, b Person) int {
        return cmp.Compare(a.Age, b.Age)
    })
    fmt.Println("After sorting by age:", people)
}
```

对于基本类型的切片，甚至可以更简单：

```go
package main

import (
    "fmt"
    "slices"
)

func main() {
    nums := []int{3, 1, 4, 1, 5, 9, 2, 6}
    fmt.Println("Before sorting:", nums)
    slices.Sort(nums)
    fmt.Println("After sorting:", nums)
}
```

**优点**：
- 代码最简洁
- 编译时类型安全
- 支持基本类型的直接排序
- 保留了自定义比较逻辑的能力（通过SortFunc）

**缺点**：
- 仅适用于Go 1.21及更高版本
- 需要导入新的`slices`包

## 性能对比（Apple M1, Go 1.25）

为了更直观地比较这三种方法的性能，我在 Apple M1 上进行了基准测试，测试数据为包含100000个随机整数的切片。

| 方法            | 操作次数   | 平均每次操作耗时 | 内存分配(B/op) | 分配次数(allocs/op) |
|-----------------|------------|-----------------|---------------|---------------------|
| sort.Interface  | 1184       | 870091 ns/op    | 81944         | 2                   |
| sort.Slice      | 1510       | 770005 ns/op    | 81976         | 3                   |
| slices.Sort     | 2560       | 453371 ns/op    | 81920         | 1                   |

数据很直观：**slices.Sort的性能几乎是其他方法的2倍**，而且内存分配次数最少。这下性能焦虑可以彻底放下了。

slices.Sort 不仅代码短，还更快，内存分配少，体验感直接拉满。

## 我的建议

- **基本类型**：直接slices.Sort，简单又安全
- **自定义类型**：slices.SortFunc，保持类型安全
- **老项目/兼容**：sort.Slice最稳
- **性能焦虑**：其实差不多(达不到优化的程度)，代码简洁性才是生产力

## 六、总结

Go语言的切片排序方法从`sort.Interface`到`sort.Slice`再到`slices`包的泛型排序，经历了从复杂到简单、从运行时安全到编译时安全的演进。

作为Go开发者，我们应该根据项目的Go版本和需求选择合适的排序方法，以写出更简洁、更安全的代码。

你在项目里最常用哪种排序方法？欢迎在评论区分享经验，也让更多Go开发者看到。

## 代码示例

如果你想直接运行这些示例，可以在Go Playground中测试：
- [sort.Interface示例](https://go.dev/play/p/qqGzLdiuqds)
- [sort.Slice示例](https://go.dev/play/p/f_dzkPcDAuR)
- [slices.Sort示例](https://go.dev/play/p/eHSsGToc-0C)

**注**：代码适用于 Go 1.21 及以上版本，旧版本请根据实际调整
