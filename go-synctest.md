# CI 跑测试要 30 秒，我升了个 Go，变成 0.01 秒

昨天发版

CI 跑到测试阶段，又卡住了。

终端上挂着一行熟悉的字：

``````go
=== RUN   TestRunDeleteExpiredSessions
``````

整整 **5 秒**，没报错，也没结束。

我盯着屏幕发呆，旁边同事已经开始刷手机了：

> “这破测试每次都要等。”

我点开测试代码，看到了那一行熟的不能再熟的东西

``````go
time.Sleep(5 * time.Second)
``````

那一刻我决定不忍了

我直接干了一件事： **把 Go 版本升到了 1.25。**

然后改了 3几行测试代码。

再跑 CI：

``````go
--- PASS: TestRunDeleteExpiredSessions (0.01s)
``````

整个测试阶段直接飞过去了。

用的就是 Go 1.25 最容易被人忽视的的： **testing/synctest**

## 以前怎么写时间相关测试

写测试最头疼的，就是依赖时间的逻辑。

比如一个后台任务：**定期清理过期会话**。

真实代码一般长这样：

```go
func RunDeleteExpiredSessions(ctx context.Context, store SessionStore, interval time.Duration) {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()
    for {
        select {
        case <-ticker.C:
            store.DeleteExpired()
        case <-ctx.Done():
            return
        }
    }
}
```

测试只能这么写：

```go
func TestRunDeleteExpiredSessions(t *testing.T) {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    store := NewMockSessionStore()
    store.AddSession("session1", time.Now().Add(1*time.Second))

    go RunDeleteExpiredSessions(ctx, store, 1*time.Second)

    // 硬等 3 秒，希望清理逻辑执行了 3 次
    time.Sleep(3 * time.Second)

    if store.Count() != 0 {
        t.Fatal("Expected 0 sessions, got", store.Count())
    }
}
```

问题很明显：

- 每跑一次测试：**真等 3 秒**
- CI 慢一点：你就改成 5 秒、10 秒
- 测试越写越慢
- 发版越来越卡

你以为慢是业务复杂。

其实只是**在等时间**

## synctest 的魔法：虚拟时间推进

Go 1.25 提供了一个新测试工具：

``````go
synctest.Test(t, func(innerT *testing.T) {
    // 在这个函数里，时间是“假的”
})
``````

示例：

```go
import "testing/synctest"

func TestWithSynctest(t *testing.T) {
	synctest.Test(t, func(innerT *testing.T) {
		time.Sleep(10 * time.Second) // 瞬间完成！
		time.Sleep(100 * time.Hour)  // 也瞬间完成！
	})
}
```

为什么？

原理很简单：**当所有 goroutine 都阻塞时，synctest 会自动推进虚拟时间，直到最近一个时间事件发生。**

所以这些都会被“快进”：

- time.Sleep
- time.Ticker
- time.After
- context.WithTimeout

你写的还是原来的代码，

只是：**不再真的等时间。**

## 实战：把那个慢测试改成 0.01 秒

改写刚才那个测试：

```go
func TestRunDeleteExpiredSessions(t *testing.T) {
    synctest.Test(t, func(st *testing.T) {
        ctx, cancel := context.WithCancel(context.Background())
        defer cancel()

        store := NewMockSessionStore()
        store.AddSession("session1", time.Now().Add(1*time.Second))

        go RunDeleteExpiredSessions(ctx, store, 1*time.Second)

        // 虚拟时间推进，不再真实等待
        time.Sleep(3 * time.Second)

        if store.Count() != 0 {
            st.Fatal("Expected 0 sessions, got", store.Count())
        }
    })
}
```

跑一下看看：

```
--- PASS: RunDeleteExpiredSessions (0.01s)
```

从 3 秒降到了 0.01 秒，快了 300 倍。

## 高级玩法：精确控制并发事件顺序

synctest 不只是快，更重要的是： **顺序确定性**

```go
func TestEventOrder(t *testing.T) {
	synctest.Test(t, func(st *testing.T) {
		ch := make(chan struct{})
		var wg sync.WaitGroup
		wg.Add(1) // 先加计数
		go func() {
			<-ch // 阻塞在通道
		}()
		go func() {
			wg.Wait() // 阻塞在 WaitGroup
		}()
		go func() {
			time.Sleep(4 * time.Second) // 阻塞在时间
			wg.Done()                   // 释放 WaitGroup
			close(ch)                   // 释放 channel
		}()
		// 推进虚拟时间，触发上面的链式唤醒
		time.Sleep(5 * time.Second)
	})
}
```

在 synctest 中：

- 所有阻塞点都可控
- 唤醒顺序可预测
- 不会出现：“本地过了，CI 挂了”

这对测试复杂并发逻辑非常重要，你可以精确控制每个事件的发生时机。

## 三个最适合 synctest 的场景

| 场景 | 传统方式问题 | synctest 方式 |
|:-----|:-------------|:--------------|
| 定时任务测试 | 需要真实等待，测试慢 | 虚拟时间瞬间完成 |
| 超时逻辑测试 | 很难稳定复现 | 精确控制 |
| 并发状态流转 | 顺序随机 | 顺序确定 |

## 避坑指南（非常重要）

### **goroutine 必须能退出**

一定要有：

``````go
case <-ctx.Done():
    return
``````

否则 synctest 可能永远推进不了时间。

### 不能在 synctest 里用

- t.Run
- t.Parallel

官方明确：会 panic。

### **不要混用真实时间源**

synctest 只接管标准库 time。

第三方 sleep、硬件定时器无效。

## **写到最后**

以前我们以为：“测试慢是因为逻辑复杂。”

现在才发现：**很多测试慢，只是因为在等时间。**

Go 1.25 的 synctest，本质上是：**把并发测试，从“等时间”变成“推时间”。**

如果你的CI也因为这个原因慢，这表示到了升级的时候了。

