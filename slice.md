> 说在前面: 本文适合对 Go 内存模型有一定了解的开发者

你以为切片就是动态数组？小心！一个简单的 append() 可能让你共享数据的切片指针‘自行消失’。

在本文中，我将从切片基础出发，深入探讨它的内存结构、容量增长机制以及栈与堆的分配规则，帮你真正搞懂切片的底层秘密。

> 在进入细节之前，我建议先查看一下之前关于 **数组** 的文章。

---

## 切片背后的秘密

一旦你声明了一个具有特定长度的数组，该长度就作为其类型的一部分被“锁定”。例如，`[1024]byte` 和 `[512]byte` 是完全不同的类型。

切片比数组灵活得多，因为它们是建立在数组之上的一层抽象。切片可以动态调整大小，并且你可以使用 `append()` 向其追加素。

创建切片的方式有多种：

```go
// a is a nil slice
var a []byte

// slice literal
b := []byte{1, 2, 3}

// slice from an array
c := b[1:3]

// slice with make
d := make([]byte, 1, 3)

// slice with new  不常见，但它是合法的。
e := *new([]byte)
```

在数组中 `len` 和 `cap` 是常量并且总是相等，Go 编译器预先知道数组的长度与容量，并且将它们硬编码到  汇编代码中。![arr](https://www.helloimg.com/i/2025/12/30/69536f89adc17.png)

但对于切片来说，`len` 和 `cap` 是动态的，这意味着它们可以在运行时发生变化。

切片实际上是对底层数组的一个‘片段’的描述。。

例如，你有一个像 `[1:3]` 这样的切片，它从索引 1 开始，到索引 3 之前结束，因此长度是 `3 - 1 = 2`：

```go
func main() {
    array := [6]int{0, 1, 2, 3, 4, 5}

    slice := array[1:3]
    fmt.Println(slice, len(slice), cap(slice))
}

// 输出：
// [1 2] 2 5
```

可以简单理解成以下示意图：
![slice1](https://www.helloimg.com/i/2025/12/30/6953715fc569b.png)

图中，切片的 `len` 表示它包含多少个元素。在这里我们有两个元素 `[1, 2]`；`cap` 则表示从切片起点到原始数组末尾的元素数量。

由于切片指向底层数组，因此对切片所做的任何更改都会更改底层数组。

> “通过 len 和 cap 函数知道切片的长度和容量，但我怎么确定切片其实从哪里开始？”
> 为了解答这个问题，下面展示了三种方法来查看内部表示形式。 

第一种是用 println 直接打印：

```go
func main() {
    array := [6]byte{0, 1, 2, 3, 4, 5}
    slice := array[1:3]
    println("array:", &array)
    println("slice:", slice, len(slice), cap(slice))
}

// Output:
// array: 0x1400004e6f2
// slice: [2/5]0x1400004e6f3 2 5
```

从输出结果来看，切片底层数组的地址和原始数组地址不同，这是为啥哪？我们通过简单的图示来说明这一点：

![](https://www.helloimg.com/i/2025/12/30/6953747547e0c.png)

如果你看过上一篇关于`数组` 的文章，你就会明白数组中元素的存储方式。实际上，切片是指向**`在数组中起始位置的地址`**。

第二种是使用 unsafe.SliceData 获取底层数组指针：

```go
func main() {
    array := [6]byte{0, 1, 2, 3, 4, 5}
    slice := array[1:3]
    arrPtr := unsafe.SliceData(slice)
    println("array[1]:", &array[1])
    println("slice.array:", arrPtr)
}

// Output:
// array[1]: 0x1400004e6f3
// slice.array: 0x1400004e6f3
```
当你将切片传给`unsafe.SliceData`时,它会做一些检查来决定返回什么:

1. 如果 `cap>0` ,返回指向切片第一个元素的指针(array[1])
2. 如果`slice == nil` ,返回`nil`
3. 如果`slice!=nil && cap==0` ,会返回一个指向`zerobase` 的内存地址

> `zerobase`：在Go中，对 `zero-size` 类型，例如`空struct{}`、`[0]int`这样的, Go运行时不会为它们每个实例都分配唯一地址，而是直接返回一个特殊变量`zerobase` 的地址
>
> 源码如下：
>
> ``````go
> 
> // base address for all 0-byte allocations
> var zerobase uintptr
> 
> //go:linkname mallocgc
> func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
> 	if gcphase == _GCmarktermination {
> 		throw("mallocgc called with gcphase == _GCmarktermination")
> 	}
> 
> 	if size == 0 {
> 		return unsafe.Pointer(&zerobase)
> 	}
>   ...
> }
> ``````
>
> 我们简单测试打印一下：
>
> ``````go
> func main() {
> 	var a struct{}
> 	fmt.Printf("struct{}: %p\n", &a)
> 
> 	var b [0]int
> 	fmt.Printf("[0]int: %p\n", &b)
> 
> 	fmt.Println("unsafe.SliceData([]int{}):", unsafe.SliceData([]int{}))
> }
> 
> // Output:
> // struct{}: 0x104f24900
> // [0]int: 0x104f24900
> // unsafe.SliceData([]int{}): 0x104f24900
> ``````
>
> 是不是都指向同一个地址，这就是我们常说的`空 struct`不占内存的原因。

第三种则是直接查看切片的内部结构：

```go
type sliceHeader struct {
    array unsafe.Pointer
    len   int
    cap   int
}

func main() {
    array := [6]byte{0, 1, 2, 3, 4, 5}
    slice := array[1:3]
    println("slice", slice)

    header := (*sliceHeader)(unsafe.Pointer(&slice))
    println("sliceHeader:", header.array, header.len, header.cap)
}

// Output:
// slice [2/5]0x1400004e6f3
// sliceHeader: 0x1400004e6f3 2 5
```

上面的输出正是我们预期的，我们通常将这种内部结构称为 sliceHeader。

---

## 切片扩容内幕：看cap如何变化

前面我说:"cap是从切片第一个元素到底层数组末尾的元素数量"，其实这并不完全正确。

例如，当你通过切片操作创建新切片时，可以选择指定切片的cap:

``````go
func main() {
	array := [6]int{0, 1, 2, 3, 4, 5}
	slice := array[1:3:4]

	println(slice)
}

// Output:
// [2/3]0x1400004e718
``````

默认情况下，如果切片操作没有指定第三个参数，切片的容量就是原数组剩余部分的长度。

在这个例子中，切片的容量被设置为4(不包含4)。

![](https://www.helloimg.com/i/2025/12/30/69537a3f197d9.png)

所以**`切片的容量是它在需要增长之前能容纳的最大元素数量`**。当你不断向切片追加元素并超过当前容量时，Go 会自动创建一个更大的数组，并将元素复制过去，然后让切片指向这个新数组。

我们创建一个实际案例来演示一下：

``````go
func main() {
	array := [6]int{0, 1, 2, 3, 4, 5}

	slice := array[1:3:4]
	fmt.Println("slice:", slice)

	slice = append(slice, 6)
	fmt.Println("slice after appending 6:", slice, unsafe.SliceData(slice))
	fmt.Println("array now:", array)

	slice = append(slice, 7)
	fmt.Println("slice after appending 7:", slice, unsafe.SliceData(slice))
	fmt.Println("array now:", array)
}

// Output:
// slice: [1 2]
// slice after appending 6: [1 2 6] 0x14000128038
// array now: [0 1 2 6 4 5]
// slice after append 7: [1 2 6 7] 0x14000128090
// array now: [0 1 2 6 4 5]
``````
当我们向切片[1 2] 追加元素6时，切片任然指向原来的底层数据，这时，会直接修改原来数组array 的值，array[3]=6，于是数组变成了`[0 1 2 6 4 5]`。
![](https://www.helloimg.com/i/2025/12/30/69537b9badc38.png)
但是当我们向切片追加7时，切片自身的cap已经不足，因此Go会创建一个新的底层数组，把原有的元素赋值过去，然后在这个新的数组上追加7。

![](https://www.helloimg.com/i/2025/12/30/69537e11bfdec.png)

数组的地址已经发生了改变。

这也是新手开发者常犯的一个错误：原本共享同一底层数组的两个切片，在执行 append() 操作后可能不再共享数据。最典型的场景是当切片作为函数参数传递时：

``````go
func changeSlice(slice []int) {
	slice[0] = 100
	slice = append(slice, 400, 500)
}

func main() {
	slice := []int{1, 2, 3}
	changeSlice(slice)

	fmt.Println(slice)
}

// Output: [100 2 3]
``````

> **那新切片的容量是如何设置的？**

当 Go 为了扩容而创建新数组时，它通常会将容量翻倍；但是当切片达到一定大小时，这种增长策略会有所调整。

如果无限制地翻倍，随着切片变大，会导致大量内存分配。为了避免这种情况，当切片达到一定大小(通常为256)时，Go会调整增长速度，满足公式如下(go >= 1.8)：

``````go
newCap =oldCap + (oldCap + 3*256)/4
``````

> 当oldCap 超级大的时候，无线趋近于1.25倍扩容	(go<1.8 就是 ==1.25, go1.8加入了平滑因子，扩容更丝滑了)。
> 这只是一个近似值，在实际的内存分配中还涉及到 `字节对齐`等



---

## 切片真的真是分配在堆上吗？

通常来说，在函数内部，任何动态的或大小未知的东西最终都会被分配到堆上，这可能会让大家觉得切片也总是分配在堆上。

事实上这是一个常见的误解，我们需要把两件事情分开来看：**切片本身（也就是切片头）**，以及 **底层数组**

### **第一种情况：切片和底层数组都分配在栈上**

```go
func main() {
	a := byte(1)
	println("a's address:", &a)

	s := make([]byte, 1)
	println("slice's address:", &s)
	println("underlying array's address:", s)
}

// Output:
// a's address: 0x1400004e71e
// slice's address: 0x1400004e720
// underlying array's address: [1/1]0x1400004e71f
```
输出结果所示，局部变量 a、切片 s，以及底层数组 s.array 都分配在栈上，它们的地址彼此非常接近。

### **第二种情况：底层数组最初分配在栈上，随后增长并迁移到堆上**

```go
func main() {
	slice := make([]int, 0, 3)
	println("slice:", slice, "- slice addr:", &slice)

	slice = append(slice, 1, 2, 3)
	println("slice full cap:", slice)

	slice = append(slice, 4)
	println("slice after exceed cap:", slice)
}

// Output:
// slice: [0/3]0x1400004e720 - slice addr: 0x1400004e738
// slice full cap: [3/3]0x1400004e720
// slice after exceed cap: [4/6]0x14000016210
```
可以注意到，当切片超过自身容量时，底层数组的地址发生了明显变化，从 0x1400004e720 变成了 0x14000016210。

此时，它已经不再位于当前 goroutine 的栈上了。

这也是为什么**提前设置合适的容量**是一个好习惯，可以避免不必要的堆分配。即使你在编译期并不知道确切的大小，给切片一个大致的容量估计，也比把容量留为 0 要好。

### **第三种情况：底层数组直接分配在堆上**

即使你提前设置了一个在编译期已知的容量，在某些情况下，底层数组仍然会被分配到堆上。

其中一种情况就是使用 make() 创建切片时，如果容量超过了 64 KB：

```go
func main() {
	sliceA := make([]byte, 64 * 1024)
	println("sliceA address:", &sliceA)
	println("sliceA:", sliceA)

	sliceB := make([]byte, 64 * 1024 + 1)
	println("sliceB address:", &sliceB)
	println("sliceB:", sliceB)
}

// Output:
// sliceA address: 0x1400019ff20
// sliceA: [65536/65536]0x1400018ff08
// sliceB address: 0x1400019ff08
// sliceB: [65537/65537]0x14000102000

```
在这里，sliceA 的底层数组被分配在栈上，并且与 sliceA 本身的地址正好相差 64 KB。而对于 sliceB，它的底层数组则被分配到了堆上。

我们再来看一下在这种情况下，逃逸分析是怎么说的：

``````go
# command-line-arguments
./main.go:3:6: can inline main
./main.go:4:16: make([]byte, 65536) does not escape
./main.go:8:16: make([]byte, 65537) escapes to heap
``````

> 这说明 **sliceB 的底层数组被分配到了堆上，但 slice 的头部本身并没有**。
> 我们是怎么知道的呢？如果 slice 头部变量 sliceB 本身也发生了逃逸，被移动到了堆上，那么逃逸分析的输出里就会明确出现类似 **“moved to heap: sliceB”** 这样的提示。
> 实际上，要让 slice 的底层数组被分配到堆上是非常容易的：**只要在编译期无法确定大小或行为的内容（也就是动态的情况），最终都会被分配到堆上**。

---

## 总结

由于栈大小在编译期就确定，而 slice 的底层数组大小要到运行时才能确定，所以大多数情况下，slice 的底层数组会被分配到 **堆上**。

那怎么尽量避免堆分配呢？

1. **提前预估容量**：使用 make() 指定一个合理的 capacity，可以减少后续扩容导致的额外堆分配。
2. **复用切片**：sync.Pool 是个好东西，可以复用相同用途的切片，避免频繁分配。
3. **重置长度**：把切片放回池前记得 slice = slice[:0]，下次取出来就能继续 append() 使用。

这样做，不仅减少堆分配，还能显著提高性能。如果想更深入了解背后的原理，可以看看《Go sync.Pool and the Mechanics Behind It》。

