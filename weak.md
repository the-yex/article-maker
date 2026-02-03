# 90% 的 Go 项目不该用弱指针？用对了就离不开

> 作为一个老gopher，对很多新的语言特性不太能接受，后面我将逐渐和大家一起去学习一些新的特性，以及如何应用到我们项目当中

Go 1.24 引入了弱指针（Weak Pointers），解决了一些内存管理痛点，但很多人第一次看时会被搞懵：什么时候该用？我们来简单探讨一下。

---

## 什么是弱指针

> 这让我第一个想到的就是rust 的所有权管理

弱指针是一种**不会阻止垃圾回收（GC）回收对象的引用**。

- **强引用（Strong Pointer）**：只要存在，GC 永远不会回收对象。
- **弱指针（Weak Pointer）**：引用对象，但不会延长对象生命周期。对象如果没有其他强引用，GC 会直接回收，弱指针返回 `nil`。

简单比喻：弱指针就像办公室的闲人，只站在旁边看热闹，不管对象活多久。

---

## Go 1.24 的 weak 包使用示例

```go
import "weak"

type User struct {
    Name string
}

func main() {
    u := &User{Name: "Alice"}
    wp := weak.Make(u) // 生成弱指针

    u = nil // 取消强引用
    runtime.GC()
    // GC 后 wp.Value() 可能返回 nil
    if obj := wp.Value(); obj != nil {
        fmt.Println(obj.Name)
    } else {
        fmt.Println("对象已被回收") // 很可能输出这一行
    }
}
```

**关键点：**使用弱指针前必须通过 `Value()` 获取强引用副本，否则可能 nil。

---

## **千万别以为弱指针能延长对象生命周期**

很多人会写成：

```go
func foo() weak.Pointer[*Obj] {
    obj := &Obj{}
    return weak.Make(obj)
}
```

结果呢？函数返回后，obj 是局部变量，没有其他强引用，GC 立即收走，弱指针返回必然是 nil。

**弱指针的真正价值**：对象活着时作为旁观者，而不是替代强引用。

---

## 典型使用场景

### 1. 缓存（Cache）

弱指针最经典用途是缓存：

- 对象是否还活着，由业务逻辑的强引用决定
- 缓存只是“影子索引”，不阻止 GC
- 对象无人使用时，GC 自动清理，缓存中的弱指针变 nil

示例：

```go
package main

import (
    "fmt"
    "runtime"
    "sync"
    "weak"
)

type User struct {
    Name string
}

var cache sync.Map // map[int]weak.Pointer[*User]

func GetUser(id int) *User {
    // ① 尝试从缓存拿数据
    if wp, ok := cache.Load(id); ok {
      	// 如果有，在没有被GC之前可以给别人用
        if u := wp.(weak.Pointer[User]).Value(); u != nil {
            fmt.Println("cache hit")
            return u
        }
    }

    // ② 不存在或弱引用已经被GC，直接从数据库拿去
    u := &User{Name: fmt.Sprintf("user-%d", id)}
    cache.Store(id, weak.Make(u))
    fmt.Println("load from DB")
    return u
}

func main() {
    u := GetUser(1)
    fmt.Println(u.Name)

    runtime.GC() 

    u = nil      // 取消强引用
    runtime.GC() // 

    _ = GetUser(1) // 如果已被 GC，输出 "load from DB"
}
```

### 2. 观察者模式 / 订阅发布

监听者通常不由主体控制，用弱指针引用主体可以避免内存泄漏。

### 3. Canonicalization

比如 Go 1.23 的 `strings.Unique`，多个相同的字符串共用一个指针，节约内存。弱指针也可以实现类似效果。

### 4. 图结构 / 循环引用

弱指针可以打破循环强引用，让 GC 正常回收对象。

## 何时不用弱指针

> 凡是你能明确 delete 或释放的对象，就不该用 weak。

弱指针的价值在于：

1. **对象生命周期模糊**：你不知道谁还在用
2. **不想显式管理释放**：避免引用计数或复杂释放逻辑
3. **允许缓存/索引保持非强制性引用**

**一句话总结**：delete 是人为控制，weak 是承认我控制不了。

---

## 总结

1. **弱指针不会延长对象生命周期**，它是旁观者。
2. **弱指针适合**：生命周期不明确、多方共享、缓存、观察者模式、去重表、循环引用。
3. **弱指针不适合**：你能明确对象何时结束的场景。
4. **设计铁律**：凡是能 `delete` 的地方，就不该用 weak；凡是“不知道谁该 delete”的地方，weak 才有意义。
5. **字符串去重 & 固定缓存**：weak 可以结合 list 或 Unique 实现高效内存节约。

> 弱指针不是性能优化，而是管理模糊生命周期的设计工具。用对了，Go 代码更安全、更灵活；用错了，你甚至连对象都拿不到。

