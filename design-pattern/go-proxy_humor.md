# 别让大对象到处跑！Go 代理模式让你的系统性能直接起飞

上周线上出了个问题：某个报表接口突然变得超慢，本来 500ms 能返回，现在要 5 秒。

我查了一下代码，发现了一个致命问题：

```go
type ReportService struct {
    dataSource *DataSource  // 每次查询都要从数据库拉大量数据
}

func (r *ReportService) GenerateReport(reportID string) Report {
    // ❌ 每次都从数据库查！
    data := r.dataSource.LoadAllData()  // 这个查询要 3 秒！

    // 处理数据
    report := r.processData(data)

    return report
}
```

"这也太蠢了吧！同一个报表，可能一小时内被调用 100 次，每次都查一遍数据库？"

**这个接口被 100 次调用，就意味着数据库要被查 100 次，每次 3 秒，总共 300 秒！**

今天不讲废话，我们直接拆解代理模式，看看怎么把这种"大对象访问"的场景优化得优雅又高效。

---

## 一、代理模式到底是干嘛的？

用一句话说清楚：**给对象加一层"代理"，控制对对象的访问。**

看个生活例子：

```
场景：你要买房，但不想亲自找房子

你：想买房
房东：有 100 套房子，分布在全城

问题：你要自己跑遍全城看房子，累死！

解决：找个房产中介（代理）
你 -> 中介 -> 房东

中介做的事：
- 帮你筛选房子（过滤）
- 帮你砍价（增强功能）
- 帮你看房（延迟加载）
- 帮你记住看过的房子（缓存）

你不用管：
- 房子在哪
- 房东是谁
- 怎么联系房东

你只管：告诉中介你要啥
```

**这就是代理模式的核心：通过"代理"控制对真实对象的访问。**

---

## 二、不用代理模式会有多恶心？

假设你有一个需要从远程 API 获取的大对象。

### ❌ 反面教材：直接访问，每次都查远程

```go
// ========== 数据源（真实对象）==========
type RemoteDataSource struct {
    apiURL string
}

func NewRemoteDataSource(apiURL string) *RemoteDataSource {
    return &RemoteDataSource{apiURL: apiURL}
}

// 这个查询很慢！要 3 秒
func (r *RemoteDataSource) FetchData() Data {
    fmt.Println("📡 正在从远程 API 获取数据...")

    // 模拟远程调用
    time.Sleep(3 * time.Second)

    return Data{
        ID:      1,
        Content: "这是从远程获取的大量数据...",
        Size:    1000000,  // 1MB 数据
    }
}

// ========== 业务服务 ==========
type ReportService struct {
    dataSource *RemoteDataSource  // 直接持有数据源
}

func (r *ReportService) GenerateReport(reportID string) Report {
    // ❌ 每次都从远程获取，太慢了！
    data := r.dataSource.FetchData()  // 3 秒！

    report := r.processData(data)
    return report
}

// ========== 使用 ==========
func main() {
    dataSource := NewRemoteDataSource("https://api.example.com/data")
    service := &ReportService{dataSource: dataSource}

    // 第一次调用：3 秒
    report1 := service.GenerateReport("REPORT_001")
    fmt.Println("报告 1 生成:", report1)

    // 第二次调用：又 3 秒！
    report2 := service.GenerateReport("REPORT_002")
    fmt.Println("报告 2 生成:", report2)

    // 第三次调用：又 3 秒！
    report3 := service.GenerateReport("REPORT_003")
    fmt.Println("报告 3 生成:", report3)
}
```

**输出：**

```
📡 正在从远程 API 获取数据...
报告 1 生成: ...
📡 正在从远程 API 获取数据...  // 又调了一次！
报告 2 生成: ...
📡 正在从远程 API 获取数据...  // 又调了一次！
报告 3 生成: ...
```

**这代码的毛病：**

1. **性能垃圾**：同一个数据被重复获取 3 次，每次 3 秒
2. **浪费资源**：远程 API 被频繁调用，可能被限流
3. **延迟高**：用户每次都要等 3 秒，体验极差
4. **成本高**：远程 API 调用可能按次计费，浪费钱

**这代码就像是你每次要喝水都去打井，而不是用杯子装起来。**

---

## 三、Go 版代理模式实现

### 基础实现：缓存代理

```go
// ========== 数据源接口 ==========
type DataSource interface {
    FetchData() Data
}

// ========== 真实数据源 ==========
type RealDataSource struct {
    apiURL string
}

func NewRealDataSource(apiURL string) *RealDataSource {
    return &RealDataSource{apiURL: apiURL}
}

func (r *RealDataSource) FetchData() Data {
    fmt.Println("📡 正在从远程 API 获取数据...")
    time.Sleep(3 * time.Second)  // 模拟慢查询

    return Data{
        ID:      1,
        Content: "这是从远程获取的大量数据...",
        Size:    1000000,
    }
}

// ========== 代理（带缓存） ==========
type CacheProxy struct {
    realDataSource *RealDataSource
    cachedData     *Data
    cacheMutex     sync.RWMutex
    lastCacheTime  time.Time
    cacheDuration  time.Duration  // 缓存有效期
}

func NewCacheProxy(realDS *RealDataSource) *CacheProxy {
    return &CacheProxy{
        realDataSource: realDS,
        cacheDuration:  10 * time.Minute,  // 缓存 10 分钟
    }
}

func (c *CacheProxy) FetchData() Data {
    c.cacheMutex.RLock()

    // 检查缓存是否有效
    if c.cachedData != nil && time.Since(c.lastCacheTime) < c.cacheDuration {
        fmt.Println("✅ 从缓存获取数据")
        cached := *c.cachedData
        c.cacheMutex.RUnlock()
        return cached
    }
    c.cacheMutex.RUnlock()

    // 缓存无效，获取新数据
    c.cacheMutex.Lock()
    defer c.cacheMutex.Unlock()

    // 双重检查（防止并发时多次获取）
    if c.cachedData != nil && time.Since(c.lastCacheTime) < c.cacheDuration {
        fmt.Println("✅ 从缓存获取数据（双重检查）")
        return *c.cachedData
    }

    // 从真实数据源获取
    data := c.realDataSource.FetchData()

    // 更新缓存
    c.cachedData = &data
    c.lastCacheTime = time.Now()

    return data
}

// ========== 使用 ==========
func main() {
    realDS := NewRealDataSource("https://api.example.com/data")
    proxy := NewCacheProxy(realDS)  // 用代理包装真实数据源

    // 第一次调用：3 秒（从远程获取）
    data1 := proxy.FetchData()
    fmt.Printf("数据 1: %s\n", data1.Content)

    // 第二次调用：0 秒（从缓存获取）
    data2 := proxy.FetchData()
    fmt.Printf("数据 2: %s\n", data2.Content)

    // 第三次调用：0 秒（从缓存获取）
    data3 := proxy.FetchData()
    fmt.Printf("数据 3: %s\n", data3.Content)
}
```

**输出：**

```
📡 正在从远程 API 获取数据...
数据 1: 这是从远程获取的大量数据...
✅ 从缓存获取数据
数据 2: 这是从远程获取的大量数据...
✅ 从缓存获取数据
数据 3: 这是从远程获取的大量数据...
```

**性能对比：**

- **不用代理**：3 次调用 × 3 秒 = 9 秒
- **用代理**：1 次远程调用（3 秒）+ 2 次缓存（0 秒）= 3 秒

**节省了 6 秒，性能提升 3 倍！**

---

## 四、实战案例：懒加载代理

有时候大对象创建很慢，但可能根本用不到，这时候可以用懒加载。

```go
// ========== 大对象接口 ==========
type BigObject interface {
    DoSomething() string
}

// ========== 真实大对象（创建很慢）==========
type RealBigObject struct {
    data []byte  // 假设这是一个很大的对象，加载要 2 秒
}

func NewRealBigObject() *RealBigObject {
    fmt.Println("⏳ 正在创建大对象...")
    time.Sleep(2 * time.Second)  // 模拟慢加载

    return &RealBigObject{
        data: make([]byte, 100*1024*1024),  // 100MB
    }
}

func (r *RealBigObject) DoSomething() string {
    return fmt.Sprintf("大对象工作，数据大小: %d bytes", len(r.data))
}

// ========== 懒加载代理 ==========
type LazyProxy struct {
    realObject BigObject
    created    bool
    mutex      sync.Mutex
}

func NewLazyProxy() *LazyProxy {
    return &LazyProxy{}
}

func (l *LazyProxy) DoSomething() string {
    l.mutex.Lock()
    defer l.mutex.Unlock()

    // 只在第一次使用时才创建真实对象
    if !l.created {
        l.realObject = NewRealBigObject()  // 懒加载
        l.created = true
    }

    return l.realObject.DoSomething()
}

// ========== 使用 ==========
func main() {
    proxy := NewLazyProxy()

    fmt.Println("🚀 代理已创建（但大对象还没加载）")

    // 做一些其他事情...
    fmt.Println("做其他事情...")

    // 只有真正需要时才加载
    result := proxy.DoSomething()
    fmt.Println(result)
}
```

**输出：**

```
🚀 代理已创建（但大对象还没加载）
做其他事情...
⏳ 正在创建大对象...
大对象工作，数据大小: 104857600 bytes
```

**优势：**

- 启动快：创建代理几乎不花时间
- 按需加载：只有用到时才创建大对象
- 可能根本不创建：如果某些场景用不到，大对象根本不会创建

---

## 五、实战案例：权限代理

有些资源只有特定用户才能访问，可以用代理做权限控制。

```go
// ========== 文档接口 ==========
type Document interface {
    Read() string
    Write(content string)
}

// ========== 真实文档 ==========
type RealDocument struct {
    content string
}

func NewRealDocument(content string) *RealDocument {
    return &RealDocument{content: content}
}

func (r *RealDocument) Read() string {
    return r.content
}

func (r *RealDocument) Write(content string) {
    r.content = content
}

// ========== 权限代理 ==========
type PermissionProxy struct {
    realDocument *RealDocument
    user         User
}

type User struct {
    ID       string
    Username string
    Role     string  // "admin", "user", "guest"
}

func NewPermissionProxy(realDoc *RealDocument, user User) *PermissionProxy {
    return &PermissionProxy{
        realDocument: realDoc,
        user:         user,
    }
}

func (p *PermissionProxy) Read() string {
    // 所有用户都可以读
    fmt.Printf("👤 用户 %s 正在读取文档...\n", p.user.Username)
    return p.realDocument.Read()
}

func (p *PermissionProxy) Write(content string) {
    // 只有 admin 才能写
    if p.user.Role != "admin" {
        fmt.Printf("🚫 用户 %s 没有写权限！\n", p.user.Username)
        return
    }

    fmt.Printf("✏️ 管理员 %s 正在写入文档...\n", p.user.Username)
    p.realDocument.Write(content)
}

// ========== 使用 ==========
func main() {
    doc := NewRealDocument("这是机密文档内容")

    // 普通用户
    user1 := User{ID: "1", Username: "zhangsan", Role: "user"}
    proxy1 := NewPermissionProxy(doc, user1)

    // 可以读
    fmt.Println(proxy1.Read())

    // 不能写
    proxy1.Write("黑客内容")

    // 管理员
    user2 := User{ID: "2", Username: "admin", Role: "admin"}
    proxy2 := NewPermissionProxy(doc, user2)

    // 可以写
    proxy2.Write("管理员更新了文档")
}
```

**输出：**

```
👤 用户 zhangsan 正在读取文档...
这是机密文档内容
🚫 用户 zhangsan 没有写权限！
✏️ 管理员 admin 正在写入文档...
```

---

## 六、实战案例：日志代理

记录所有方法调用，方便调试和监控。

```go
// ========== 日志代理 ==========
type LoggingProxy struct {
    realDataSource *RealDataSource
}

func NewLoggingProxy(realDS *RealDataSource) *LoggingProxy {
    return &LoggingProxy{realDataSource: realDS}
}

func (l *LoggingProxy) FetchData() Data {
    start := time.Now()

    log.Printf("📝 调用 FetchData()")

    data := l.realDataSource.FetchData()

    elapsed := time.Since(start)
    log.Printf("📝 FetchData() 完成，耗时: %v，数据大小: %d bytes",
        elapsed, data.Size)

    return data
}
```

---

## 七、代理模式 vs 装饰器模式

很多人会混淆这两个模式。

| 对比项 | 代理模式 | 装饰器模式 |
|--------|---------|-----------|
| 目的 | 控制访问 | 增强功能 |
| 关注点 | 隐藏真实对象 | 增加新功能 |
| 用途 | 缓存、懒加载、权限控制 | 日志、计时、验证 |
| 客户端感知 | 不知道有代理 | 知道有装饰器 |

**简单记忆：**

- **代理**：控制"能不能"访问，怎么访问
- **装饰器**：给功能"加"点东西

---

## 八、避坑指南（血泪经验）

### ❌ 坑 1：缓存失效问题

```go
// ❌ 缓存永不过期
type BadProxy struct {
    cache *Data
}

// 如果真实数据更新了，缓存里的还是旧数据！
```

**解决：设置缓存过期时间，或者主动刷新缓存。**

```go
// ✅ 设置过期时间
func (p *Proxy) FetchData() Data {
    if p.cache != nil && time.Since(p.lastCacheTime) < p.cacheTTL {
        return *p.cache
    }
    // 刷新缓存
}
```

### ❌ 坏 2：缓存并发问题

```go
// ❌ 没有并发控制
func (p *Proxy) FetchData() Data {
    if p.cache == nil {
        p.cache = p.realDS.FetchData()  // 多个 goroutine 会并发创建！
    }
    return *p.cache
}
```

**解决：使用 sync.Once 或双重检查锁。**

```go
// ✅ 使用 sync.Once
type SafeProxyOnce struct {
    cache  *Data
    once   sync.Once
}

func (p *SafeProxyOnce) FetchData() Data {
    p.once.Do(func() {
        p.cache = &p.realDS.FetchData()
    })
    return *p.cache
}
```

### ❌ 坏 3：代理链太长

```go
// ❌ 代理链太长，性能下降
data := LoggingProxy(
    PermissionProxy(
        CacheProxy(
            RealDataSource(),
        ),
    ),
).FetchData()
```

**解决：合并功能，或者用更高效的数据结构。**

---

## 九、总结

**代理模式的核心价值：**

1. **性能优化**：缓存、懒加载，提升性能
2. **权限控制**：控制访问权限，保证安全
3. **资源管理**：管理大对象的创建和销毁
4. **日志监控**：记录方法调用，方便调试

**适用场景：**

- ✅ 远程代理：访问远程对象，控制网络开销
- ✅ 虚拟代理：延迟加载大对象
- ✅ 缓存代理：缓存结果，减少重复计算
- ✅ 保护代理：控制访问权限
- ✅ 智能引用：记录访问次数、引用计数

**不适用场景：**

- ❌ 小对象：用代理反而增加复杂度
- ❌ 简单场景：直接访问就行
- ❌ 性能极度敏感：代理有开销

---

**最后送你一句：**

> "代理模式就像你的私人助理，帮你搞定脏活累活。但别找个助理倒杯水都慢半拍，那就适得其反了。"

**下期预告：命令模式——别让业务逻辑散落一地，学会这招让你的操作变得可撤销、又可记录！**