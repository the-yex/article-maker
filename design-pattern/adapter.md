# 多 AI 厂商接入的工程解法：适配器模式

如果你做过 AI 网关或统一代理服务，大概率遇到过这种场面：

​	今天接 OpenAI，明天接 Anthropic，后天老板说再加个通义。

​	每个厂商：参数不一样、返回结构不同、错误处理也不尽相同

一旦设计错了，后面每接一个新厂商，都是一次全项目重构。

这时候，**适配器模式就是救星**。它可以把不同厂商的接口“翻译”成你系统统一的接口，让系统优雅对接任何 AI 服务，而不用改业务逻辑。

## 一、适配器模式到底是干嘛的？

一句话概括：**把一个接口翻译成另一个接口，让不兼容的系统协同工作**。

生活例子更直观：

```
你：用 Type-C 充电线
旧设备：只支持 Micro-USB

解决：买个转接头（适配器）
Type-C 转 Micro-USB，接口不兼容也能充电

特点：
- 你的充电器不用改
- 旧设备不用改
- 只有转接头需要做“翻译”
```

这就是适配器模式的核心：**在两个不兼容的接口之间加一层翻译**。

## 二、为什么不直接改系统？

假设你直接改自己的统一 AI 接口去兼容各厂商：

``````go
type AIProcessor interface {
    Generate(prompt string, model string, temperature float64, vendor string) (string, error)
}
``````

看起来很方便，但问题来了：

- 每增加一个厂商，你都要改接口
- 所有调用方都得改
- 违反开闭原则，对测试和维护都是灾难

最终只会导致：

- 接口参数越来越长
- 调用方越来越痛苦
- 新的厂商一旦接入，全项目都要跟着改

然后系统就变成了“屎山代码”

## 三、Go 版适配器模式实现（AI 服务示例）

### 基础实现：类适配器

```go
type AIProcessor interface {
    Generate(prompt string) (string, error)
}
```

### **厂商接口示例(假的)**

``````go
type OpenAIClient struct {}
func (c *OpenAIClient) CreateCompletion(text string) (string, error) {
    return "OpenAI 回答: " + text, nil
}

type AnthropicClient struct {}
func (c *AnthropicClient) GetResponse(input string) (string, error) {
    return "Anthropic 回答: " + input, nil
}
``````

### **适配器实现**

``````go
type OpenAIAdapter struct {
    client *OpenAIClient
}

func (a *OpenAIAdapter) Generate(prompt string) (string, error) {
    return a.client.CreateCompletion(prompt)
}

type AnthropicAdapter struct {
    client *AnthropicClient
}

func (a *AnthropicAdapter) Generate(prompt string) (string, error) {
    return a.client.GetResponse(prompt)
}
``````

### **统一使用接口**

``````go
func main() {
    openAI := &OpenAIAdapter{client: &OpenAIClient{}}
    anthropic := &AnthropicAdapter{client: &AnthropicClient{}}

    processors := []AIProcessor{openAI, anthropic}

    for _, p := range processors {
        res, _ := p.Generate("写一篇 Go 适配器模式的文章")
        fmt.Println(res)
    }
}
``````

**输出：**

```
OpenAI 回答: 写一篇 Go 适配器模式的文章
Anthropic 回答: 写一篇 Go 适配器模式的文章
```

## 四、适配器模式最佳实践

1. **只做接口翻译**：不要在适配器里写业务逻辑或副作用
2. **保留原始错误信息**：便于排查
3. **避免过度设计**：简单类型转换直接处理即可

**坏例子（适配器里做太多事）**：

``````go
func (a *OpenAIAdapter) Generate(prompt string) (string, error) {
    if len(prompt) == 0 { return "", fmt.Errorf("prompt不能为空") }
    user := db.GetUser("123")        // ❌ 不该在适配器访问 DB
    email.Send(user.Email, "你有新的请求") // ❌ 副作用
    return a.client.CreateCompletion(prompt)
}
``````

**正确做法：只做接口翻译**：

``````go
func (a *OpenAIAdapter) Generate(prompt string) (string, error) {
    return a.client.CreateCompletion(prompt)
}
``````

## 五、避坑指南

### 坑 1：适配器做了太多事

**适配器只负责"接口翻译"，别让它干杂活。**

- 业务逻辑、数据库访问、副作用操作都不应该在适配器里做
- 适配器只负责“接口翻译”

### 坑 2：**返回错误信息丢失**

- 原始错误信息要保留，便于排查问题

``````go
result, err := client.CreateCompletion(prompt)
if err != nil {
    return "", fmt.Errorf("调用 OpenAI 失败: %v", err)
}
``````

### 坑 3：过度适配

- 简单类型转换或格式转换，不需要写适配器，直接处理即可

---

## 六、总结

**适配器模式的核心价值：**

1. **解耦系统**：新老系统，不同厂商互不影响
2. **不修改原接口**：遵循开闭原则，对扩展开放
3. **统一接口**：多个不同系统能用同一套接口调用
4. **渐进迁移**：可以慢慢迁移老系统，不影响新系统

**适用场景：**

- ✅ 集成第三方系统：支付、物流、短信等第三方服务
- ✅ 接口不兼容：两个系统接口不兼容但需要协同工作
- ✅ 遗留系统改造：把老系统接入新架构
- ✅ 重构：重构时用适配器让新旧代码共存

**不适用场景：**

- ❌ 接口本来就很兼容：直接用就行
- ❌ 简单的参数转换：直接转，别过度设计
- ❌ 性能敏感场景：适配器有转换开销

---

**最后送你一句：**

> “适配器模式就像旅行插头，让你的设备在全世界都能充电；接入各种 AI 服务，也能统一接口，优雅无忧。”

**下期预告：代理模式——别让大对象到处跑，学会这招让你的系统性能直接起飞！**