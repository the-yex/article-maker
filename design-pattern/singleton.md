# Go 设计模式最佳实践·单例模式

关于单例模式，网上 90% 的文章都只停留在"怎么用"的层面。

"就是搞个全局变量呗，简单！"

然后线上就炸了：并发场景下竟然初始化了两次，配置中心拉取了两份配置，把数据库写坏了。

今天不讲废话，我们直接来看看 Go 里单例模式到底该怎么写。

---

## 一、为什么你要用单例？

先搞清楚：**单例不是什么银弹，它只是管理“全局唯一状态”的工具。**

适合用单例的场景只有这几类：

1. **配置管理器**：全局只需要一份配置
2. **数据库连接池**：一个进程一个池就够了

[//]: # (3. **日志器 / 监控器**：统一出口)

不适用的场景更多：

- 为了省那点内存硬用单例（这点内存算个屁）
- 为了代码写得爽到处用全局变量（这叫懒惰，不叫设计）
- 把请求级数据塞进单例（并发直接乱套）

反正你记住： **单例解决的是“唯一性”，不是“方便性”。**

---

## 二、90% 的人都会犯的错

最常见的“朴素写法”：

```go
var instance *ConfigManager
func GetInstance() *ConfigManager {
    if instance == nil {
        instance = &ConfigManager{
            AppConfig: map[string]string{"env": "prod"},
        }
    }
    return instance
}
```

单线程下没问题。

**并发一来，直接变灾难。**

假设 3 个 goroutine 同时进来：

```
Goroutine 1: instance == nil? Yes -> 开始初始化...
Goroutine 2: instance == nil? Yes -> 开始初始化... (还没完！)
Goroutine 3: instance == nil? Yes -> 开始初始化... (也没完！)
```

结果就是：**初始化了三次，返回了三个不同的实例！**

这在配置管理场景下就意味着：
- 配置拉了三遍，浪费资源
- 三份配置可能不一致，导致数据错乱
- 最坑的是：你可能根本不知道这个 bug 的存在

---

## 三、第一次尝试：加锁解决并发问题

很多人第一反应就是："加个锁不就完事了？"

```go
var (
    instance *ConfigManager
    mu       sync.Mutex
)
func GetInstance() *ConfigManager {
    mu.Lock()
    defer mu.Unlock()
    if instance == nil {
        instance = &ConfigManager{
            AppConfig: map[string]string{"env": "prod"},
        }
    }
    return instance
}
```

**这个方案能用，但是不完美：**

- **性能垃圾**：每次调用都要拿锁，哪怕单例早就初始化好了
- **锁竞争**：100 万次调用就是 100 万次锁竞争，延迟直接起飞


**为了一个已经初始化好的单例，每次调用都得排队拿锁？这设计简直是反人类。**

---

## 四、终极方案：sync.Once 才是正解

Go 标准库早就给你准备了现成工具：

```go
var (
    once     sync.Once
    instance *ConfigManager
)

func GetInstance() *ConfigManager {
    once.Do(func() {
        instance = &ConfigManager{AppConfig: map[string]string{"env": "prod"}}
    })
    return instance
}
```

**就这几行代码，稳得一批。**

为什么 `sync.Once` 这么牛？

底层实现用的是 `atomic` 原子操作 + 双重检查：

1. **第一次检查**：用原子操作读取状态，无锁，极快
2. **抢锁初始化**：只有第一次调用才会真正拿锁
3. **第二次检查**：防止并发竞态
4. **后续调用**：直接返回，连原子操作都快

具体我之前写过一篇文章专门介绍过`Once`它的用法及各种坑，可以看看【】



实测数据（MacBook Pro M1，单次调用 GetInstance）：

```
sync.Mutex 版本：1000 万次调用，~200ms
sync.Once 版本：1000 万次调用，~60ms
```

---

## 五、**生产环境常见用法**

###  1：配置管理器

```go
type ConfigManager struct {
    AppConfig map[string]string
}

var (
    once     sync.Once
    instance *ConfigManager
)

func GetConfigManager() *ConfigManager {
    once.Do(func() {
        // 从环境变量/配置中心加载配置
        instance = &ConfigManager{
            AppConfig: map[string]string{
                "env":      os.Getenv("ENV"),
                "db_host":  os.Getenv("DB_HOST"),
                "api_key":  os.Getenv("API_KEY"),
            },
        }
    })
    return instance
}
```

### ：数据库连接池

```go
type DBPool struct {
    *sql.DB
}
var (
    dbOnce  sync.Once
    dbPool  *DBPool
)
func GetDBPool() *DBPool {
    dbOnce.Do(func() {
        // 假设 DSN 从环境变量获取
        dsn := os.Getenv("DB_DSN")
        db, err := sql.Open("mysql", dsn)
        if err != nil {
            log.Fatal("数据库连接失败：", err)
        }

        // 设置连接池参数
        db.SetMaxOpenConns(100)
        db.SetMaxIdleConns(10)
        db.SetConnMaxLifetime(time.Hour)

        dbPool = &DBPool{DB: db}
    })
    return dbPool
}
```

---

## 六、避坑指南

### ❌ 坑 1：在单例初始化里做耗时操作

```go
once.Do(func() {
    // 不要这样做！初始化太慢会阻塞所有首次调用
    instance = &ConfigManager{Data: fetchFromRemoteAPI()}  // 可能要 3 秒
})
```

**解决：使用懒加载或预加载**

```go
// 方案 1：预加载（应用启动时）
func init() {
    GetInstance()
}

// 方案 2：带超时的懒加载
func GetInstance() *ConfigManager {
    once.Do(func() {
        ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
        defer cancel()
        data, err := fetchFromRemoteAPIWithContext(ctx)
        if err != nil {
            // 降级处理：使用默认配置
            data = getDefaultConfig()
        }
        instance = &ConfigManager{Data: data}
    })
    return instance
}
```

### ❌ 坑 2：**测试被单例污染**

```go
// 测试 A
func TestConfig(t *testing.T) {
    config := GetConfigManager()
    config.AppConfig["test"] = "value_a"
}

// 测试 B 会受测试 A 的影响！
func TestConfig2(t *testing.T) {
    config := GetConfigManager()
    fmt.Println(config.AppConfig["test"])  // 输出 "value_a"，但预期是空的
}
```

**解决：测试时重置单例**

```go
// 暴露一个重置方法（仅用于测试）
func ResetInstanceForTest() {
    once = sync.Once{}  // 重置 sync.Once
    instance = nil
}

func TestConfig(t *testing.T) {
    defer ResetInstanceForTest()  // 测试结束后重置

    config := GetConfigManager()
    // ... 测试逻辑
}
```

### ❌ 坑 3：**用单例存请求数据**

```go
func HandleRequest(req *http.Request) {
    config := GetConfigManager()
    config.CurrentRequest = req  // ❌ 别这么干！
}
```

**问题：并发请求会互相覆盖，数据直接乱套。**

**解决：请求作用域的数据传参数，别搞全局。**

## **七、重要边界：不是所有初始化都适合用单例**

这里必须强调一个常被忽略的点：

> sync.Once 的语义是：
> **只执行一次，不管成功还是失败。**

也就是说：

- 如果初始化逻辑可能失败
- 并且你希望：失败后还能重试

那么：

> ❌ 这个初始化逻辑不适合放进单例懒加载里

例如：

- 数据库连接（可能暂时不可用）
- 远程配置中心
- RPC 客户端初始化

这类更适合：

- 启动时显式初始化
- 或使用可重试逻辑
- 而不是“懒汉式单例”

> 👉 sync.Once 的原理与错误语义，我在之前的文章已经单独拆过，这里不再展开。

---

## 八、总结

| 写法 | 并发安全 | 性能 | 适用场景 |
|------|---------|------|----------|
| 直接赋值 | ❌ | ⚡⚡⚡ | 单线程/测试代码 |
| 全局加锁 | ✅ | 🐢 | 简单场景，低频调用 |
| **sync.Once** | ✅ | ⚡⚡⚡ | **生产环境，高频调用** |

**记住三句话：**

1. **单例解决的是“唯一性”，不是“偷懒”**
2. **Go 里实现单例，标准答案就是** **sync.Once**
3. **可能失败的初始化，不适合做懒加载单例**

---

**最后送你一句保命口诀：**

> 全局变量是野兽，
> 单例模式是笼子，
> sync.Once 是门锁。

**下期预告：工厂模式——别再到处 new 对象了，学会这招让你的代码优雅度起飞！**