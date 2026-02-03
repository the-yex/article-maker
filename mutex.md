# Go sync.Mutex：普通模式与饥饿模式

今天我们来聊聊 Go 语言中一个既基础又关键的同步原语——sync.Mutex。你是不是也曾经在并发编程中遇到过数据竞争、结果不对的问题？这篇文章将带你深入理解 Mutex 的两种工作模式，并告诉你如何正确使用它来保护共享资源。
## 为什么需要 sync.Mutex？

在 Go 语言中，sync.Mutex（互斥锁）用于确保同一时间只有一个 goroutine 能访问某个共享资源。无论是一段代码、一个变量、一个 map，还是一个结构体或 channel，只要存在并发访问的可能，就需要考虑同步保护。

如果你在并发环境下直接读写 `map`，很可能会看到这个熟悉的错误：

```
fatal error: concurrent map read and map write
```

这说明你缺少对共享资源的保护。虽然 Go 提供了 `sync.Map`，但今天我们聚焦在更基础、更灵活的 `sync.Mutex` 上。

## 一个经典的竞态示例

先来看一段代码：
```go
var counter = 0
var wg sync.WaitGroup

func incrementCounter() {
	counter++
	wg.Done()
}

func main() {
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go incrementCounter()
	}
	wg.Wait()
	fmt.Println(counter)
}
```

你觉得输出会是多少？理论上应该是 `1000`，但实际运行往往少于这个值。为什么？

因为 `counter++` 不是原子操作，它包含**读取、修改、写回**三个步骤。在多个 goroutine 并发执行时，可能出现"写覆盖"，导致部分自增操作丢失。这就是典型的**竞态条件（race condition）**。

在 ARM64 下，对应的汇编类似如下：

```
MOVD main.counter(SB), R0
ADD  $1, R0, R0
MOVD R0, main.counter(SB)
```

如果两个 goroutine 几乎同时执行到第一行指令，就会发生写覆盖，导致结果丢失，这就是**竞态条件（race condition）**。

![mutex1](https://www.helloimg.com/i/2026/01/02/69572ff20d5e6.png)

如图：goroutine `G1` 先读取了 counter 的值，但在它把更新后的结果写回之前，goroutine G2 也读取了同一个值。随后两者分别将各自计算后的结果写回。由于它们读取的是相同的原始值，最终只有一次自增生效，另一次实际上被覆盖掉了。

---

## 使用 Mutex 解决竞态问题

我们只需要加上一把锁：

```go
var counter = 0
var wg sync.WaitGroup
var mutex sync.Mutex

func incrementCounter() {
	mutex.Lock()
	counter++
	mutex.Unlock()
	wg.Done()
}
```

现在，输出稳定为 `1000`。Mutex 确保了同一时刻只有一个 goroutine 能执行 `counter++`。

> ⚠️ 注意：不要对已解锁的 Mutex 再次调用 `Unlock()`，否则会 panic。建议使用 `defer mutex.Unlock()` 确保解锁。

---

## sync.Mutex 的内部结构

`sync.Mutex` 在标准库中定义非常简洁：

```go
package sync

type Mutex struct {
	state int32
	sema  uint32
}
```

- `state`：32 位整数，包含锁状态、等待者数量、是否饥饿等标志位。
- `sema`：信号量，用于 goroutine 的阻塞与唤醒。

![](https://www.helloimg.com/i/2026/01/02/6957327a2741e.png)
### state 各比特位含义

| 位段 | 含义 |
|:--:|:--:|
| Locked (bit 0) | mutex 是否被锁定 |
| Woken (bit 1) | 是否已有 goroutine 被唤醒 |
| Starving (bit 2) | 是否处于饥饿模式 |
| Waiter (bit 3~31) | 等待锁的 goroutine 数量 |

---

## Mutex 是如何加锁的？

`Lock()` 分为两条路径：

1. **快速路径（fast path）**：如果当前锁空闲，直接通过 CAS 获取锁，效率极高，通常会被内联。
2. **慢速路径（slow path）**：如果锁已被占用，则进入自旋等待或排队。

```go
func (m *Mutex) Lock() {
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
	m.lockSlow()
}
```
快速路径（fast path）被设计得非常高效，主要用于处理大多数情况下 mutex 尚未被占用时的加锁操作。
此路径通常会被 内联（inlined），也就是说，它的代码会直接嵌入到调用该函数的上下文中：

```go
$ go build -gcflags="-m"

./main.go:13:12: inlining call to sync.(*Mutex).Lock
./main.go:15:14: inlining call to sync.(*Mutex).Unlock
./main.go:16:9: inlining call to sync.(*WaitGroup).Done
```

顺便说一下，这个内联的快速路径是一个巧妙的优化技巧，它利用了 Go 的内联优化，并在 Go 源码中被广泛使用。



**什么是自旋（spinning）？**

> 自旋指的是 goroutine 进入一个紧凑循环，不断检查 mutex 的状态，而不主动放弃 CPU。
>
> 注意：单核 CPU 不会启用自旋，因为那样只会浪费时间。

在这里，自旋并不是简单的 for 循环，而是使用低级汇编指令实现的“spin-wait”。下面我们快速看一下 ARM64 架构下的相关代码示例：

``````go
TEXT runtime·procyield(SB),NOSPLIT,$0-0
	MOVWU	cycles+0(FP), R0
again:
	YIELD
	SUBW	$1, R0
	CBNZ	R0, again
	RET
``````

汇编代码会执行一个紧循环，运行约 30 个周期（runtime.procyield(30)），在此过程中不断让出 CPU 并递减自旋计数器。

自旋结束后，它会再次尝试获取锁。如果仍未成功，还有 **三次机会继续自旋**，总共最多尝试 120 个周期。如果依然无法获取锁，goroutine 会 **增加等待者计数**，将自己加入等待队列，然后休眠，等待被唤醒的信号，再继续尝试。

> **为什么需要自旋？**

自旋的目的是**短暂等待**，希望 mutex 很快释放，这样 goroutine 就可以直接获取锁，而无需经历完整的睡眠-唤醒开销。

**`如果计算机只有单核，自旋就不会启用，因为那样只会浪费 CPU 时间。`**

> **“那如果已经有其他 goroutine 在等待这个 mutex 呢？这个 goroutine 先获取锁是不是不公平？”**

这就是为什么 Go 的 mutex 有两种模式：**普通模式（Normal）和饥饿模式（Starvation）**。在饥饿模式下，自旋机制不会起作用。

---

## 普通模式（Normal Mode）

在普通模式下，等待 mutex 的 goroutine 会按照 **先进先出（FIFO）** 排队。当某个 goroutine 被唤醒尝试获取锁时，它并不会立刻获得控制权，而需要与同时到来的新 goroutine 竞争锁。

这种竞争对新 goroutine 更有利，因为它们已经在 CPU 上运行，可以快速尝试获取锁，而排队的 goroutine 则还在唤醒过程中。

![normal](https://www.helloimg.com/i/2026/01/02/695743077b4ec.png)

结果是，刚被唤醒的 goroutine 经常会在与新到 goroutine 的竞争中失败，并被重新放回队列前端。

---

## 饥饿模式（Starvation Mode）

>  **“如果这个 goroutine 特别倒霉，总是在新 goroutine 到来时被唤醒怎么办？”😭**

很好的问题。如果一直发生这种情况，它可能永远无法获取锁。这就是为什么需要将 mutex 切换到 **饥饿模式（Starvation Mode）**。在 GitHub 上有一个 issue (https://github.com/golang/go/issues/13086) 专门讨论了之前设计的不公平性。

当一个 goroutine 连续超过 **1 毫秒** 获取不到锁时，饥饿模式就会启动。这个模式确保等待队列中的 goroutine 最终能公平地获得 mutex。

在饥饿模式下，当一个 goroutine 释放 mutex 时，它会**直接将控制权交给队列前端的 goroutine**。也就是说，不再有竞争，也没有来自新 goroutine 的抢占。新到的 goroutine 不会尝试抢锁，只会排在等待队列的末尾。

![hungry](https://www.helloimg.com/i/2026/01/02/695745435a0fe.png)

如上图所示，在饥饿模式下，mutex 会依次将访问权交给 G1、G2 等等待的 goroutine。
每个被唤醒的 goroutine 都会检查两个条件：
	1.	它是否是等待队列中的最后一个 goroutine；
	2.	它等待的时间是否少于 1 毫秒。

如果满足其中任意一个条件，mutex 就会切换回 普通模式（Normal Mode）。

---

## Mutex.Unlock() 的执行流程

解锁流程比加锁流程要简单一些。依然分为两条路径：`快速路径（内联执行）`和 `慢路径（处理非常规情况）`。

快速路径会清除 mutex 状态中的锁定位（locked bit），也就是开头所说的 mutex.state 的第 0 位。如果清除后 state 变为 0，说明没有其他标志被设置（比如没有等待的 goroutine），此时 mutex 已完全释放。

> **但是，如果 state 仍不为 0 呢？**

这时就会进入慢路径，慢路径需要判断 mutex 当前是处于普通模式还是饥饿模式。下面是慢路径的代码实现：
```go
func (m *Mutex) unlockSlow(new int32) {
	if (new+mutexLocked)&mutexLocked == 0 {
		fatal("sync: unlock of unlocked mutex")
	}
	if new&mutexStarving == 0 {
		old := new
		for {
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else {
		runtime_Semrelease(&m.sema, true, 1)
	}
}
```

在 **普通模式** 下，如果有等待的 goroutine，并且没有其他 goroutine 被唤醒或已经获取锁，mutex 会尝试 **原子性地减少等待者计数** 并 **设置 mutexWoken 标志**。如果成功，它会释放信号量，从而唤醒一个等待的 goroutine 去获取 mutex。

在 **饥饿模式** 下，mutex 会 **原子性地增加信号量（mutex.sema）**，并将 mutex 的所有权 **直接交给队列中最前面的等待 goroutine**,runtime_Semrelease 的第二个参数用于指示是否执行这种直接交付（handoff）。



## 什么时候用 Mutex？

✅ **适用场景：**
- 保护共享的 `map`、`slice`、结构体等数据
- 需要同时更新多个相关字段保持一致性
- 包装非线程安全的第三方库或资源

❌ **不适用场景：**
- **读多写少** → 用 `sync.RWMutex` 提升性能
- **简单计数器** → 用 `sync/atomic` 更高效
- **需要重入锁** → Go 的 Mutex 不支持，会死锁
- **goroutine 间通信** → 用 Channel 更符合 Go 风格
- **并发 Map** → 考虑 `sync.Map`

---

## 总结

`sync.Mutex` 不是简单的"先来后到"，它通过**普通模式**提升并发性能，通过**饥饿模式**保证公平性。理解这两种模式，能帮助你写出更高效、更稳定的并发程序。

下次当你使用 Mutex 时，不妨想一想：你的场景更注重性能，还是更注重公平？你选对同步工具了吗？你锁对了吗？



