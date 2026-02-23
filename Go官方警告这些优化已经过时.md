# 别再迷信这些 Go 性能优化了：官方自己都在打脸

Go 圈有一套流传多年的“性能圣经”：

> 少用堆分配
> 多用预分配
> 别用接口
> 能 sync.Pool 就 sync.Pool

这些话你一定听过。

甚至已经刻进肌肉记忆了。

问题是：**这些经验，是给现在的 Go 用的吗？**

当年它们是对的。

那时候：

- 编译器不够聪明
- GC 对分配极度敏感
- 稍微多分配一点就会抖成筛子

但现在是 **Go 1.25+**。

我用 benchmark 跑了一组实验，结果非常反直觉：

 很多“金科玉律”，已经开始过期了。

---

## 分配 ≠ 成本，生命周期才是成本

很多人对 Go 性能的第一反应是：

- 栈 = 快
- 堆 = 慢
- GC = 贵

所以只要看到 new，条件反射就是：

**“完了，这里要慢了。”**

但在现代 Go 里，这个逻辑已经不成立了。

### **实验：短生命周期对象**

```go
package main

import "testing"

var sink *int

func allocNoEscape(n int) {
	sum := 0
	for i := 0; i < n; i++ {
		s := new(int)
		*s = i
		sum += *s
	}
	sink = &sum
}

func noAlloc(n int) {
	s := 0
	for i := 0; i < n; i++ {
		s = i
		sink = &s // 同样逃逸
	}
}

func BenchmarkShortLivedAlloc(b *testing.B) {
	for b.Loop() {
		allocNoEscape(100)
	}
}

func BenchmarkShortLived_NoAlloc(b *testing.B) {
	for b.Loop() {
		noAlloc(100)
	}
}

```

**测试结果：**

| Benchmark | ns/op | B/op | allocs/op |
|:----------|:-------------|:-----|:----------|
| ShortLivedAlloc (with new) | 96 | 8 | 1 |
| ShortLived_NoAlloc | 129 | 8 | 1 |

你会发现：

- 用了 new(int)
- **分配次数没有增加**
- 两种写法的 allocs/op 都是 1

说明什么？

> 在对象不逃逸出函数作用域的前提下，
> **new 并不必然意味着频繁堆分配。**

### **第一个真相**

**用不用指针，不决定在不在堆上。**

**真正决定分配位置的，是逃逸分析。**

也就是说：

> ❌ new → 堆 → 慢
> ✅ 是否逃逸 → 生命周期多长 → 才决定成本

**在现代 Go 中：**

> new 只是语法
> 逃逸才是命运

---

## 预分配：猜对是优化，猜错是负优化

“切片一定要预分配”

几乎是每个 Go 程序员的信仰。

问题是：

你真的知道要多大吗？还是在瞎估算？

### **实验：切片增长策略**

```go
package main

import "testing"

const sliceN = 256

// sinks 防止编译器优化掉工作
var sinkSlice []int
var sinkInt int

func buildNoPrealloc(n int) []int {
    out := []int{}
    for i := 0; i < n; i++ {
        out = append(out, i)
    }
    return out
}

func buildExactPrealloc(n int) []int {
    out := make([]int, 0, n)
    for i := 0; i < n; i++ {
        out = append(out, i)
    }
    return out
}

func buildOverPrealloc(n int) []int {
    out := make([]int, 0, n*16) // 过度预alloc
    for i := 0; i < n; i++ {
        out = append(out, i)
    }
    return out
}

func BenchmarkSlices_NoPrealloc(b *testing.B) {
    for b.Loop() {
        sinkSlice = buildNoPrealloc(sliceN)
    }
}

func BenchmarkSlices_ExactPrealloc(b *testing.B) {
    for b.Loop() {
        sinkSlice = buildExactPrealloc(sliceN)
    }
}

func BenchmarkSlices_OverPrealloc(b *testing.B) {
    for b.Loop() {
        sinkSlice = buildOverPrealloc(sliceN)
    }
}
```

**测试结果：**

| Benchmark | ns/op | B/op | allocs/op |
|:----------|:----------------|:-----|:----------|
| Slices_NoPrealloc (n=256) | 864.3 | 4088 | 9 |
| Slices_ExactPrealloc | 454.9 | 2048 | 1 |
| Slices_OverPrealloc (x16) | 3475 | 32768 | 1 |

结论非常刺眼：

- ✅ 精确预分配：

  - 分配次数从 9 次 → 1 次
  - 内存减半
  - 时间减半

  

- ❌ 过度预分配：

  - 分配次数很好看（1 次）
  - 但直接分配了 **32KB 用不上的内存**
  - 性能比不预分配还慢 **4 倍**

### **第二个真相**

> allocs/op 不是性能指标
> **分配了多少字节 + 活多久，才是关键**

---

## 接口确实慢，但通常不是你系统的瓶颈

“接口慢”这件事是事实。

### **实验：具体类型 vs 接口 vs 泛型**

```go
package main

import "testing"

type Adder interface {
	Add(x int) int
}

// sinks 防止编译器优化掉工作
var sinkSlice []int
var sinkInt int

type impl struct{}

func (i impl) Add(x int) int {
	return x + 1
}

func callConcrete(v impl, n int) int {
	sum := 0
	for i := 0; i < n; i++ {
		sum += v.Add(i)
	}
	return sum
}

func callInterface(v Adder, n int) int {
	sum := 0
	for i := 0; i < n; i++ {
		sum += v.Add(i)
	}
	return sum
}

func callGeneric[T Adder](v T, n int) int {
	// 实际实现需要 method constraint
	sum := 0
	for i := 0; i < n; i++ {
		//sum += v.Add(i)
	}
	return sum
}

func BenchmarkConcrete(b *testing.B) {
	v := impl{}
	for b.Loop() {
		sinkInt = callConcrete(v, 1024)
	}
}

func BenchmarkInterface(b *testing.B) {
	v := impl{}
	for b.Loop() {
		sinkInt = callInterface(v, 1024)
	}
}

func BenchmarkGeneric(b *testing.B) {
	v := impl{}
	for b.Loop() {
		sinkInt = callGeneric(v, 1024)
	}
}

```

**测试结果：**

| Benchmark | ns/op | B/op | allocs/op |
|:----------|:----------------|:-----|:----------|
| Concrete call | 1064 | 0 | 0 |
| Interface call | 3746 | 0 | 0 |
| Generic call | 576.9 | 0 | 0 |

接口调用慢了大约 3 倍。

但注意：

这是一个**纯 CPU 循环测试**：

- 没有 IO
- 没有 map
- 没有锁
- 没有内存访问

而真实系统里是什么？

- 网络 IO
- JSON 编解码
- 锁竞争
- cache miss

### **第三个真相**

> 接口有成本
> 但几乎很少是第一瓶颈
> **优化前先 profile**

---

## sync.Pool 很容易变成负优化

很多人把 sync.Pool 当成 GC 的解药。

但在某些场景里，它比直接 make 还慢。

### **实验：分配 vs 池**

```go
package main

import (
    "sync"
    "testing"
)

var bufPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 1024)
    },
}

func allocBuffers(n int) {
    for i := 0; i < n; i++ {
        _ = make([]byte, 1024)
    }
}

func poolBuffers(n int) {
    for i := 0; i < n; i++ {
        buf := bufPool.Get().([]byte)
        // 使用 buf...
        bufPool.Put(buf)
    }
}

func BenchmarkAlloc(b *testing.B) {
    for b.Loop() {
        allocBuffers(10)
    }
}

func BenchmarkPool(b *testing.B) {
    for b.Loop() {
        poolBuffers(10)
    }
}
```

**测试结果：**

| Benchmark | ns/op | B/op | allocs/op |
|:----------|:----------------|:-----|:----------|
| Alloc (make) | 9.68 | 0 | 0 |
| Pool (Get/Put) | 397.6 | 240 | 10 |

sync.Pool 慢了 **40 倍**。

原因很简单：

- 编译器把 make 优化掉了
- pool 路径反而多了同步和 bookkeeping

### **第四个真相**

> 如果对象本来就不会进堆
> **pool = 纯额外成本**

---

## 真正杀性能的不是分配，而是“保留”

这是最容易被忽略、

也是最致命的一类问题。

```go
package main

import "testing"

var sink2 [][]byte

func badRetention(n int) [][]byte {
    out := make([][]byte, 0, n)
    for i := 0; i < n; i++ {
        payload := make([]byte, 64*1024) // 64 KB
        // 只用一小部分，但保留整个 payload
        out = append(out, payload[:8])
    }
    return out
}

func goodRetention(n int) [][]byte {
    out := make([][]byte, 0, n)
    for i := 0; i < n; i++ {
        payload := make([]byte, 64*1024) // 64 KB
        // 提取小切片，让大 payload 被回收
        small := make([]byte, 8)
        copy(small, payload[:8])
        out = append(out, small)
    }
    return out
}

func BenchmarkBadRetention(b *testing.B) {
    for b.Loop() {
        sink2 = badRetention(128)
    }
}

func BenchmarkGoodRetention(b *testing.B) {
    for b.Loop() {
        sink2 = goodRetention(128)
    }
}
```

**测试结果：**

| Benchmark | ns/op | B/op | allocs/op |
|:----------|:----------------|:-----|:----------|
| BadRetention | 431175 | 8391827 | 129 |
| GoodRetention | 290683 | 4224 | 129 |

分配次数一样。

性能却差了三个数量级。

### **最终真相**

现代 Go 的性能问题

越来越多是：**生命周期设计问题**

而不是：分配技巧问题

不是：

- new 还是 make
- 指针还是值
- 要不要 pool

而是：

- 数据什么时候该死
- 谁拥有它
- 会不会被意外引用住

---

## 写在最后

Go 变了。

很多早年为了绕过运行时限制的“技巧”，

现在可能已经变成：

- 无效优化
- 负优化
- 增加复杂度

现在更重要的是：

> 设计好生命周期
> 比减少一次分配重要得多

---

你项目里有没有那种

**“凭感觉加的优化”**？

比如：

- 到处都是 sync.Pool
- 所有切片都提前预分配
- 接口全部改成具体类型

跑一次 benchmark 和 pprof 看看，

你可能会发现：

自己在和编译器对着干。