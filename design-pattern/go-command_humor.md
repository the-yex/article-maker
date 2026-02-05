# 别让业务逻辑散落一地！Go 命令模式让你的操作可撤销、又可记录

上周出了个线上事故：一个运营人员不小心点击了"批量删除用户"按钮，结果把 1000 个正常用户都删了。

我查了一下代码，发现了一个严重问题：

```go
func DeleteUsers(userIDs []string) error {
    for _, userID := range userIDs {
        // ❌ 直接删，无法撤销！
        err := db.Delete("users", userID)
        if err != nil {
            return err
        }
    }
    return nil
}

// 前端调用
func HandleDeleteRequest(req DeleteRequest) error {
    return DeleteUsers(req.UserIDs)
}
```

"这也太危险了吧？删了就删了，根本没有撤销机制？运营人员手滑一下，就可能导致严重后果！"

**更坑的是，这种操作连日志都没有记录，根本不知道是谁在什么时候删了哪些用户。**

今天不讲废话，我们直接拆解命令模式，看看怎么把这种"操作"封装成对象，让它们可撤销、又可记录。

---

## 一、命令模式到底是干嘛的？

用一句话说清楚：**把"操作"封装成对象，让操作可以被存储、传递、撤销、重做。**

看个生活例子：

```
场景：你在用 Word 写文档

操作：
- 输入文字
- 删除文字
- 撤销
- 重做

问题：你怎么实现"撤销"功能？

不用命令模式：
- 操作直接执行
- 无法记录历史
- 无法撤销

用命令模式：
- 每个操作都是一个"命令对象"
- 执行命令前，记录到历史栈
- 撤销时，从栈里弹出命令，执行 undo()

命令对象包含：
- execute(): 执行操作
- undo(): 撤销操作
- 可以存储、传递、序列化
```

**这就是命令模式的核心：操作 = 对象，可以被管理。**

---

## 二、不用命令模式会有多恶心？

假设你要实现一个文本编辑器，支持撤销和重做。

### ❌ 反面教材：操作逻辑散落一地

```go
type TextEditor struct {
    content string
}

func (e *TextEditor) Insert(text string) {
    // ❌ 操作直接执行，无法记录
    e.content += text
}

func (e *TextEditor) Delete(length int) {
    // ❌ 操作直接执行，无法记录
    e.content = e.content[:len(e.content)-length]
}

func (e *TextEditor) Undo() {
    // ❌ 没法撤销！历史没记录
    log.Println("❌ 无法撤销，没有实现！")
}

// ========== 使用 ==========
func main() {
    editor := &TextEditor{content: "Hello"}

    editor.Insert(" World")  // content: "Hello World"
    editor.Delete(5)          // content: "Hello "

    // ❌ 想撤销回"Hello World"？做不到！
    editor.Undo()  // ❌ 没有任何效果
}
```

**这代码的毛病：**

1. **无法撤销**：操作直接执行，历史没记录
2. **无法重做**：撤销后无法重做
3. **无法记录**：不知道做了哪些操作
4. **无法序列化**：无法保存操作到文件或数据库
5. **无法批量执行**：无法批量执行一组操作

**这代码就像是你写作业没有草稿纸，写错了就没法改。**

---

## 三、Go 版命令模式实现

### 基础实现：可撤销的文本编辑器

```go
// ========== 命令接口 ==========
type Command interface {
    Execute() error
    Undo() error
}

// ========== 文本编辑器（接收者） ==========
type TextEditor struct {
    content string
}

func (e *TextEditor) GetContent() string {
    return e.content
}

func (e *TextEditor) SetContent(content string) {
    e.content = content
}

// ========== 插入命令 ==========
type InsertCommand struct {
    editor *TextEditor
    text   string
}

func NewInsertCommand(editor *TextEditor, text string) *InsertCommand {
    return &InsertCommand{
        editor: editor,
        text:   text,
    }
}

func (i *InsertCommand) Execute() error {
    i.editor.SetContent(i.editor.GetContent() + i.text)
    fmt.Printf("✏️ 插入文字: '%s'\n", i.text)
    return nil
}

func (i *InsertCommand) Undo() error {
    newContent := i.editor.GetContent()[:len(i.editor.GetContent())-len(i.text)]
    i.editor.SetContent(newContent)
    fmt.Printf("↩️ 撤销插入: '%s'\n", i.text)
    return nil
}

// ========== 删除命令 ==========
type DeleteCommand struct {
    editor    *TextEditor
    length    int
    deletedText string  // 保存删除的内容，用于撤销
}

func NewDeleteCommand(editor *TextEditor, length int) *DeleteCommand {
    return &DeleteCommand{
        editor: editor,
        length: length,
    }
}

func (d *DeleteCommand) Execute() error {
    content := d.editor.GetContent()
    if len(content) < d.length {
        return fmt.Errorf("删除长度超过内容长度")
    }

    // 保存删除的内容
    d.deletedText = content[len(content)-d.length:]

    // 执行删除
    newContent := content[:len(content)-d.length]
    d.editor.SetContent(newContent)
    fmt.Printf("🗑️ 删除文字: '%s' (长度: %d)\n", d.deletedText, d.length)
    return nil
}

func (d *DeleteCommand) Undo() error {
    d.editor.SetContent(d.editor.GetContent() + d.deletedText)
    fmt.Printf("↩️ 撤销删除: '%s'\n", d.deletedText)
    return nil
}

// ========== 命令历史（撤销栈） ==========
type CommandHistory struct {
    history []Command
    current int  // 当前指针
}

func NewCommandHistory() *CommandHistory {
    return &CommandHistory{
        history: make([]Command, 0),
        current: -1,
    }
}

func (h *CommandHistory) ExecuteCommand(cmd Command) error {
    err := cmd.Execute()
    if err != nil {
        return err
    }

    // 如果撤销过，删除当前位置之后的历史
    if h.current < len(h.history)-1 {
        h.history = h.history[:h.current+1]
    }

    // 添加到历史
    h.history = append(h.history, cmd)
    h.current++

    return nil
}

func (h *CommandHistory) Undo() error {
    if h.current < 0 {
        return fmt.Errorf("没有可撤销的操作")
    }

    cmd := h.history[h.current]
    err := cmd.Undo()
    if err != nil {
        return err
    }

    h.current--
    return nil
}

func (h *CommandHistory) Redo() error {
    if h.current >= len(h.history)-1 {
        return fmt.Errorf("没有可重做的操作")
    }

    h.current++
    cmd := h.history[h.current]
    return cmd.Execute()
}

// ========== 使用 ==========
func main() {
    editor := &TextEditor{content: "Hello"}
    history := NewCommandHistory()

    fmt.Printf("初始内容: '%s'\n\n", editor.GetContent())

    // 执行操作
    history.ExecuteCommand(NewInsertCommand(editor, " World"))
    history.ExecuteCommand(NewInsertCommand(editor, "!"))
    history.ExecuteCommand(NewDeleteCommand(editor, 6))

    fmt.Printf("\n当前内容: '%s'\n", editor.GetContent())

    // 撤销
    fmt.Println("\n--- 撤销操作 ---")
    history.Undo()  // 撤销删除
    fmt.Printf("撤销后内容: '%s'\n", editor.GetContent())

    history.Undo()  // 撤销插入 "!"
    fmt.Printf("撤销后内容: '%s'\n", editor.GetContent())

    // 重做
    fmt.Println("\n--- 重做操作 ---")
    history.Redo()  // 重做插入 "!"
    fmt.Printf("重做后内容: '%s'\n", editor.GetContent())

    history.Redo()  // 重做删除
    fmt.Printf("重做后内容: '%s'\n", editor.GetContent())
}
```

**输出：**

```
初始内容: 'Hello'

✏️ 插入文字: ' World'
✏️ 插入文字: '!'
🗑️ 删除文字: 'World!' (长度: 6)

当前内容: 'Hello'

--- 撤销操作 ---
↩️ 撤销删除: 'World!'
撤销后内容: 'Hello World!'
↩️ 撤销插入: '!'
撤销后内容: 'Hello World'

--- 重做操作 ---
✏️ 插入文字: '!'
重做后内容: 'Hello World!'
🗑️ 删除文字: 'World!' (长度: 6)
重做后内容: 'Hello'
```

---

## 四、实战案例：数据库事务模拟

命令模式非常适合模拟数据库事务的 commit 和 rollback。

```go
// ========== 数据库接收者 ==========
type Database struct {
    users map[string]User
    mutex sync.RWMutex
}

func NewDatabase() *Database {
    return &Database{
        users: make(map[string]User),
    }
}

func (d *Database) InsertUser(user User) error {
    d.mutex.Lock()
    defer d.mutex.Unlock()

    if _, exists := d.users[user.ID]; exists {
        return fmt.Errorf("用户已存在: %s", user.ID)
    }

    d.users[user.ID] = user
    fmt.Printf("✅ 插入用户: %s (%s)\n", user.Name, user.ID)
    return nil
}

func (d *Database) DeleteUser(userID string) error {
    d.mutex.Lock()
    defer d.mutex.Unlock()

    if _, exists := d.users[userID]; !exists {
        return fmt.Errorf("用户不存在: %s", userID)
    }

    delete(d.users, userID)
    fmt.Printf("🗑️ 删除用户: %s\n", userID)
    return nil
}

func (d *Database) GetUser(userID string) (User, error) {
    d.mutex.RLock()
    defer d.mutex.RUnlock()

    user, exists := d.users[userID]
    if !exists {
        return User{}, fmt.Errorf("用户不存在: %s", userID)
    }

    return user, nil
}

// ========== 数据库命令 ==========
type InsertUserCommand struct {
    db   *Database
    user User
}

func (c *InsertUserCommand) Execute() error {
    return c.db.InsertUser(c.user)
}

func (c *InsertUserCommand) Undo() error {
    return c.db.DeleteUser(c.user.ID)
}

type DeleteUserCommand struct {
    db     *Database
    userID string
    user   User  // 保存被删除的用户，用于撤销
}

func (c *DeleteUserCommand) Execute() error {
    // 先获取用户信息（用于撤销）
    user, err := c.db.GetUser(c.userID)
    if err != nil {
        return err
    }
    c.user = user

    // 删除用户
    return c.db.DeleteUser(c.userID)
}

func (c *DeleteUserCommand) Undo() error {
    return c.db.InsertUser(c.user)
}

// ========== 事务 ==========
type Transaction struct {
    commands []Command
    executed bool
}

func NewTransaction() *Transaction {
    return &Transaction{
        commands: make([]Command, 0),
    }
}

func (t *Transaction) AddCommand(cmd Command) {
    t.commands = append(t.commands, cmd)
}

func (t *Transaction) Commit() error {
    fmt.Println("\n📝 开始提交事务...")

    for _, cmd := range t.commands {
        if err := cmd.Execute(); err != nil {
            fmt.Printf("❌ 命令执行失败: %v，开始回滚...\n", err)
            t.Rollback()
            return err
        }
    }

    t.executed = true
    fmt.Println("✅ 事务提交成功！")
    return nil
}

func (t *Transaction) Rollback() error {
    fmt.Println("\n🔄 开始回滚事务...")

    // 反向撤销
    for i := len(t.commands) - 1; i >= 0; i-- {
        if err := t.commands[i].Undo(); err != nil {
            fmt.Printf("❌ 回滚失败: %v\n", err)
            return err
        }
    }

    t.executed = false
    fmt.Println("✅ 事务回滚成功！")
    return nil
}

// ========== 使用 ==========
func main() {
    db := NewDatabase()

    // 先插入一些初始数据
    db.InsertUser(User{ID: "1", Name: "张三", Email: "zhangsan@example.com"})
    db.InsertUser(User{ID: "2", Name: "李四", Email: "lisi@example.com"})

    // 创建事务
    tx := NewTransaction()

    // 添加操作
    tx.AddCommand(&InsertUserCommand{
        db: db,
        user: User{ID: "3", Name: "王五", Email: "wangwu@example.com"},
    })
    tx.AddCommand(&DeleteUserCommand{db: db, userID: "2"})
    tx.AddCommand(&InsertUserCommand{
        db: db,
        user: User{ID: "4", Name: "赵六", Email: "zhaoliu@example.com"},
    })

    // 提交事务
    if err := tx.Commit(); err != nil {
        log.Fatal("事务失败:", err)
    }

    fmt.Println("\n当前数据库用户:")
    users := db.users
    for id, user := range users {
        fmt.Printf("  - %s: %s\n", id, user.Name)
    }

    // 模拟另一个会失败的事务
    fmt.Println("\n--- 模拟失败的事务 ---")
    tx2 := NewTransaction()
    tx2.AddCommand(&DeleteUserCommand{db: db, userID: "1"})
    tx2.AddCommand(&DeleteUserCommand{db: db, userID: "999"})  // 这个会失败

    if err := tx2.Commit(); err != nil {
        fmt.Println("预期失败:", err)
    }

    fmt.Println("\n回滚后的数据库用户:")
    for id, user := range db.users {
        fmt.Printf("  - %s: %s\n", id, user.Name)
    }
}
```

**输出：**

```
✅ 插入用户: 张三 (1)
✅ 插入用户: 李四 (2)

📝 开始提交事务...
✅ 插入用户: 王五 (3)
🗑️ 删除用户: 2
✅ 插入用户: 赵六 (4)
✅ 事务提交成功！

当前数据库用户:
  - 1: 张三
  - 3: 王五
  - 4: 赵六

--- 模拟失败的事务 ---
📝 开始提交事务...
🗑️ 删除用户: 1
❌ 命令执行失败: 用户不存在: 999，开始回滚...
🔄 开始回滚事务...
✅ 插入用户: 张三 (1)
✅ 事务回滚成功！

回滚后的数据库用户:
  - 1: 张三
  - 3: 王五
  - 4: 赵六
```

---

## 五、实战案例：宏命令（批量操作）

把多个命令组合成一个"宏命令"，可以一键执行、撤销。

```go
// ========== 宏命令 ==========
type MacroCommand struct {
    commands []Command
}

func NewMacroCommand() *MacroCommand {
    return &MacroCommand{
        commands: make([]Command, 0),
    }
}

func (m *MacroCommand) AddCommand(cmd Command) {
    m.commands = append(m.commands, cmd)
}

func (m *MacroCommand) Execute() error {
    fmt.Println("🎯 执行宏命令...")
    for _, cmd := range m.commands {
        if err := cmd.Execute(); err != nil {
            return err
        }
    }
    fmt.Println("✅ 宏命令执行完成")
    return nil
}

func (m *MacroCommand) Undo() error {
    fmt.Println("🎯 撤销宏命令...")
    for i := len(m.commands) - 1; i >= 0; i-- {
        if err := m.commands[i].Undo(); err != nil {
            return err
        }
    }
    fmt.Println("✅ 宏命令撤销完成")
    return nil
}

// ========== 使用 ==========
func main() {
    editor := &TextEditor{content: ""}
    history := NewCommandHistory()

    // 创建宏命令
    macro := NewMacroCommand()
    macro.AddCommand(NewInsertCommand(editor, "Hello"))
    macro.AddCommand(NewInsertCommand(editor, " "))
    macro.AddCommand(NewInsertCommand(editor, "World"))
    macro.AddCommand(NewInsertCommand(editor, "!"))

    // 执行宏命令
    history.ExecuteCommand(macro)

    fmt.Printf("\n执行宏命令后: '%s'\n", editor.GetContent())

    // 撤销宏命令
    history.Undo()
    fmt.Printf("撤销宏命令后: '%s'\n", editor.GetContent())

    // 重做宏命令
    history.Redo()
    fmt.Printf("重做宏命令后: '%s'\n", editor.GetContent())
}
```

---

## 六、命令模式的额外好处

### 1. 操作日志

```go
type CommandLogger struct {
    db *sql.DB
}

func (l *CommandLogger) LogCommand(cmd Command, userID string) error {
    // 把命令序列化后存到数据库
    cmdData, err := json.Marshal(cmd)
    if err != nil {
        return err
    }

    _, err = l.db.Exec(
        "INSERT INTO command_logs (user_id, command_data, created_at) VALUES (?, ?, ?)",
        userID, cmdData, time.Now(),
    )
    return err
}
```

### 2. 远程命令

```go
type RemoteCommand struct {
    cmd     Command
    service string
}

func (r *RemoteCommand) Execute() error {
    // 把命令发送到远程服务执行
    payload, _ := json.Marshal(r.cmd)

    resp, err := http.Post(r.service+"/execute", "application/json", bytes.NewBuffer(payload))
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    return nil
}
```

### 3. 延迟执行

```go
type DelayedCommand struct {
    cmd      Command
    executeAt time.Time
}

func (d *DelayedCommand) Execute() error {
    if time.Now().Before(d.executeAt) {
        fmt.Printf("⏳ 命令将在 %v 执行\n", d.executeAt.Sub(time.Now()))
        return nil
    }
    return d.cmd.Execute()
}
```

---

## 七、避坑指南（血泪经验）

### ❌ 坑 1：命令无法序列化

```go
// ❌ 包含未导出字段，无法序列化
type BadCommand struct {
    cmd *http.Client  // ❌ 无法序列化
}

// ✅ 只包含可序列化的数据
type GoodCommand struct {
    apiURL string  // ✅ 可以序列化
    apiKey string  // ✅ 可以序列化
}
```

### ❌ 坏 2：Undo 实现不完整

```go
// ❌ Undo 实现有问题，可能导致数据不一致
func (c *BadCommand) Undo() error {
    // 只部分撤销
    return nil
}

// ✅ Undo 要完整、可逆
func (c *GoodCommand) Undo() error {
    // 完整撤销之前的所有操作
    return c.db.Rollback()
}
```

### ❌ 坏 3：命令内存泄漏

```go
// ❌ 命令持有大对象，可能导致内存泄漏
type HugeCommand struct {
    data []byte  // 100MB 数据
}

// ✅ 只存储必要信息
type EfficientCommand struct {
    dataID string  // 只存储 ID
    // 用时再从数据库加载
}
```

---

## 八、总结

**命令模式的核心价值：**

1. **可撤销**：操作可以撤销、重做
2. **可记录**：操作可以被记录、审计
3. **可延迟**：操作可以延迟执行
4. **可组合**：多个操作可以组合成宏命令
5. **可远程**：命令可以发送到远程执行

**适用场景：**

- ✅ GUI 操作：文本编辑器、画图工具的撤销/重做
- ✅ 数据库事务：commit/rollback
- ✅ 任务队列：延迟执行、远程执行
- ✅ 操作日志：审计、追踪
- ✅ 批量操作：宏命令、脚本执行

**不适用场景：**

- ❌ 简单操作：直接执行就行，别封装
- ❌ 无撤销需求：没有 undo 需求，用命令是浪费
- ❌ 性能敏感：封装对象有开销

---

**最后送你一句：**

> "命令模式就像游戏的存档功能，玩错了可以读档重来。但别存太多存档，硬盘会爆的。"

**这是 Golang 设计模式系列的最后一篇了！希望这 8 篇文章能帮你写出更优雅、更可维护的代码。记住：设计模式不是目的，写出好代码才是目的。**