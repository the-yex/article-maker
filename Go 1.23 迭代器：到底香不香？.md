# Go 1.23 迭代器：到底香不香？
Go 1.23 有一个特性被吵翻了。

有人说：“终于统一了，早该这样。”
有人说：“Go 正在变复杂。”

我看了文档，写了 benchmark，但说实话——

到现在 Go 1.26 都出来了，我还没在项目里真正用过它。

## 一、Go 官方为什么一定要搞迭代器？
**一句话：Go 的迭代方式真的太乱了。**
你只要稍微写过点偏底层或标准库相关的代码，一定遇到过这些“各玩各的”迭代 API：

- archive/tar.Reader.Next() → 返回 (T, error)
- bufio.Scanner.Scan() → 返回 bool
- container/ring.Ring.Do() → 传回调
- sync.Map.Range() → 还是回调
- runtime.Frames.Next() → 也是 bool，但语义还不一样

**结果就是：**
> 每用一个新“可迭代对象”，
> 你都得先看一遍文档，搞清楚：
>
> - 什么时候停？
> - 怎么 break？
> - error 是不是结束信号？

这件事，**非常不 Go**。

Go 一直追求的是： *“代码长得都差不多，你一眼就知道怎么玩。”*

于是，Go 官方干了一件很激进、也很 Go 的事：

> **让 for range 支持函数类型。**

## 二、Go 1.23 的核心设计：迭代器
Go 没有引入什么 Iterator 接口，

而是选择了一种**非常函数式，但又很克制的方案**。

```go
func(yield func(V) bool)
```
解释成人话就是：

- 你写一个函数
- Go 在 for range 时，把一个 yield 函数塞给你
- 你往 yield 里“推”值
- yield 返回 true：继续
- 返回 false：立刻停

**控制权在调用方，而不是你。**

### **一个非常经典的例子：遍历二叉树**

```go
func (t *Tree[K, V]) All(yield func(K, V) bool) {
    if t == nil {
        return
    }
    if t.left.All(yield) && yield(t.key, t.value) {
        t.right.All(yield)
    }
}
```
使用方式：

``````go
for k, v := range tree.All {
    fmt.Println(k, v)
}
``````

这一刻你会发现一个很爽的点：

- 递归遍历逻辑 **完全封装在结构体内部**
- 调用方 **只看到一个 for range**
- break / continue / return **全都能正常工作**

**这是以前回调式 API 很难做到的。**

### 除了上面那个方式,还有另一种玩法：Pull 迭代器
```go
func() (V, bool)
```
用法你应该很熟：
```go
next := list.Iter()
for v, ok := next(); ok; v, ok = next() {
    fmt.Println(v)
}
```
甚至还能直接 for range。

但注意一句话：

> **Pull 不是 Go 的原生模型，而是“被适配出来的”。**

这点，在性能测试里会非常明显。

## 三、顺手一个彩蛋：range 整数

这个反而是我最喜欢的“糖”：

```go
for i := range 10 {
    fmt.Println(i) // 0, 1, 2, ..., 9
}
```
再也不用写 `for i := 0; i < n; i++` 了！
## 四、性能测试：迭代器到底慢不慢？
说再多“设计优雅”，

**不跑 benchmark 都是耍流氓。**

我用 Go 1.23 对 **10000 个元素的切片** 做了对比测试。

### 测试结果
| 方式 | 耗时 (ns/op) | 内存分配 |
|------|-------------|----------|
| **传统 for range** | ~3,300 | 0 |
| slices.Values 迭代器 | ~21,300 | 0 |
| slices.All 迭代器 | ~21,000 | 0 |
| 自定义 Push 迭代器 | ~21,000 | 0 |
| **Pull 迭代器** | ~464,000 | 400B/13次 |
### **结论很明显**

1. **Push 迭代器：慢 ≈ 6 倍**
   - 没有额外分配
   - 主要是函数调用成本
2. **Pull 迭代器：慢 ≈ 140 倍**
   - 因为底层要用 goroutine 把 push 转 pull
3. **循环越短，相对开销越夸张**

### 所以到底该不该用？

**什么时候可以用？**

- 业务层代码
- 非热路径
- 自定义数据结构（树、图、游标式遍历）
- 想**统一消费方式、提升可读性**

**什么时候别用？**

- 性能敏感路径
- 高频循环
- 能直接 for range slice/map 的地方
- **Pull 迭代器基本别进核心逻辑**

### 完整测试代码
```go
package main
import (
	"iter"
	"slices"
	"testing"
)
var data = make([]int, 10000)
func init() {
	for i := range data {
		data[i] = i
	}
}
// 传统 for range
func BenchmarkTraditionalForRange(b *testing.B) {
	sum := 0
	for i := 0; i < b.N; i++ {
		for _, v := range data {
			sum += v
		}
	}
}
// slices.Values 迭代器
func BenchmarkIteratorSlicesValues(b *testing.B) {
	sum := 0
	for i := 0; i < b.N; i++ {
		for v := range slices.Values(data) {
			sum += v
		}
	}
}
// Pull 迭代器
func BenchmarkPullIterator(b *testing.B) {
	sum := 0
	for i := 0; i < b.N; i++ {
		next, stop := iter.Pull(slices.Values(data))
		for {
			v, ok := next()
			if !ok {
				break
			}
			sum += v
		}
		stop()
	}
}
```
运行命令：
```bash
go test -bench=. -benchmem iter_benchmark_test.go
```
## 五、元芳我这样看😁
Go 1.23 的迭代器，并不是为了“更快”，

而是为了：

> **让 Go 的代码，在结构上更一致。**

它确实：

- 提高了理解成本
- 引入了函数式味道
- 在性能上付出了代价

但从长期来看，它解决的是一个**十几年的历史包袱**。

**不必强求，也不必抗拒。**
---

Go 还是那个 Go，只是多了一颗——**你得知道什么时候吃的糖。**



最后留个问题：

你会在生产项目里用 Go 1.23 的迭代器吗？

- 已经在用了  
- 想用，但还在观望  
- 除非被逼，否则不会  
- 坚决不用  

我选第四个。你呢？
