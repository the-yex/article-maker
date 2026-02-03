# 一个让性能暴跌8倍的并发“优化”

最近在重构老项目，其中有一个排序算法，处理大规模数据的时候会产生性能瓶颈，因为用的是归并排序，所以这里我想着并发处理一下会不会更快。

简单的归并如下：  

![](https://www.helloimg.com/i/2026/01/08/695f25c674c2a.png)

它的核心思路是**:**

1. **不断拆分列表**，直到每个子列表只有一个元素；

1. **逐层合并**子列表，最终得到有序列表。



看起来它是可以并行运行的算法,于是我满怀信心地写了一个并发版本，替换了原来的顺序实现。上线前还跟同事说：“这次优化完，排序速度至少提升一倍。”

测试系统监控显示原接口耗时从原来的平均500ms飙升到4000ms以上,整整慢了8倍！我赶紧回滚代码，开始排查这个“反向优化”到底是怎么回事。

---

## **三年老项目的稳定版排序，看似没问题**

```go

func merge(s []int, middle int) {
	left := append([]int(nil), s[:middle]...)  
	right := append([]int(nil), s[middle:]...) 

	...
}

func sequentialMergesort(s []int) {
	if len(s) <= 1 {
		return
	}
	m := len(s) / 2
	sequentialMergesort(s[:m])
	sequentialMergesort(s[m:])
	merge(s, m)
}
```

这是项目里用了三年的排序实现，虽然就是基础的排序,不算优,但一直稳定可靠。

---

## **我以为加了并发就是快，但它慢了8倍**

```go
func parallelMergesort(s []int) {
    if len(s) <= 1 {
        return
    }

    m := len(s) / 2
    var wg sync.WaitGroup
    wg.Add(2)

    go func() {
        defer wg.Done()
        parallelMergesort(s[:m])
    }()

    go func() {
        defer wg.Done()
        parallelMergesort(s[m:])
    }()

    wg.Wait()
    merge(s, m)
}
```

我满怀信心地把左右子数组交给不同的 goroutine 并行处理，心里想着：

**“8核 CPU，上去就快翻倍，没毛病。”**

我跑了基准测试：

**环境：Apple M1 / 8核 / 16GB 内存，10000 个随机整数**

结果一出来，我直接傻眼——耗时不仅没降低，居然比顺序版本还慢了整整 8 倍！

``````text
goos: darwin
goarch: arm64
pkg: go-behavior/compare-concurrency
cpu: Apple M1
Benchmark_normal-8                  2085            602761 ns/op         1110795 B/op      19998 allocs/op
Benchmark_concurrency-8           238           4905086 ns/op         2340052 B/op      51133 allocs/op

``````

**并发版本比顺序版本慢了整整8倍多！**
内存分配多了2.5倍，GC压力巨大。

---

## **为什么并发反而让性能掉进深渊？**

###  **goroutine 数量爆炸**

我的并发实现每一层递归都创建2个goroutine，当数据量为10万时：

- 递归深度约14层
- 理论最大 goroutine 数 ≈ 16,384

实际由于递归终止条件，虽然不会真的创建这么多，但仍然是一个巨大的数字。每个goroutine的创建、调度、销毁都有成本。

### **小任务并发是"亏本买卖"**

当切片被拆得很小时（比如只剩几个元素），排序本身只需要微秒级时间。但启动goroutine、等待同步的成本远高于排序本身。

这就好比：你为了搬一块砖，专门请了一个搬家公司。砖是搬了，但钱花得更多。

### 内存密集型操作遇上并发瓶颈

归并排序的 `merge` 会频繁复制数据：

```go
left := append([]int(nil), s[:middle]...)
```

大量 goroutine 同时分配内存，CPU 缓存命中率下降，内存带宽成为瓶颈，GC频繁触发。多核CPU不是在干活，而是在“排队抢内存”。

## **并发归并排序真的不能并发吗？**

我们知道了goroutine 在合并少量元素时效率不高，那我们可以强行让大slice才进行并发处理试试看。

优化版本如下：

```go
const threshold = 2048

func parallelMergesortV2(s []int) {
	if len(s) <= 1 {
		return
	}
	if len(s) <= threshold {
		sequentialMergesort(s)
	} else {
		middle := len(s) / 2

		var wg sync.WaitGroup
		wg.Add(2)

		go func() { 
			defer wg.Done()
			parallelMergesortV2(s[:middle])
		}()

		go func() {
			defer wg.Done()
			parallelMergesortV2(s[middle:])
		}()

		wg.Wait()
		merge(s, middle) // Merges the halves
	}
}

```

和 V1 唯一的区别：  
**增加了一个阈值，slice 大小达不到阈值，不再并发，直接顺序排序。**

测试结果如下：

``````
goos: darwin
goarch: arm64
pkg: go-behavior/compare-concurrency
cpu: Apple M1
Benchmark_normal-8                  1713            612492 ns/op         1110796 B/op      19998 allocs/op
Benchmark_concurrency-8           283           5247920 ns/op         2335285 B/op      51081 allocs/op
Benchmark_concurrency_V2-8          2949            396718 ns/op         1111591 B/op      20019 allocs/op

``````

比顺序版本快了约30%，内存分配回到合理水平，调整阈值还能进一步优化。

---

## **血泪教训：并发也能坑死你**

- **并发不是银弹**：goroutine 不是免费的，上下文切换和 WaitGroup 都有成本
- **任务够重才值得并发**：微任务用并发只会赔本
- **控制粒度是艺术**：设置阈值，避免小任务抢CPU和内存

> 并发是一把利器，但用不好，它能帮你，也能坑你。

---

## **如果你也想优化项目，这几条经验必须记住**

1. **先基准测试**：优化前必须要有性能基线
2. **渐进式并发**：从最上层开始并发，而不是每一层
3. **设置合理阈值**：根据测试数据动态调整
4. **监控GC压力**：并发可能带来的隐藏成本

**并发不是让代码跑得更快的魔法，而是需要精心调控的武器。在错误的粒度上使用它，它就会反过来攻击你的系统性能。**
