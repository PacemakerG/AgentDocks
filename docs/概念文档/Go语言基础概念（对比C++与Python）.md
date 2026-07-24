# Go 语言基础概念（对比 C++ 与 Python）

## 1. `:=` 与 `=`

- `:=`：声明并赋值，只能在函数内部用，类型自动推导。
- `=`：给已声明的变量赋值，变量必须提前存在。

```go
x := 10       // 声明新变量
x = 20        // 给已存在的 x 赋值
y = 30        // 错误：y 从未声明过
```

多重赋值时，只要左边有一个新变量，`:=` 就合法，已存在的变量会被复用（赋值），不会重新声明：

```go
a, err := f1()
b, err := f2()   // err 被复用，b 是新变量
```

块级作用域（`if`/`for`/`{}`）中用 `:=` 会产生新的局部变量，遮蔽外层同名变量，这是常见的 bug 来源。

---

## 2. 关键字 与 预声明标识符

Go 只有 25 个真正的关键字，语法层面固定死，不能挪作他用：

```text
break    default   func    interface  select
case     defer     go      map        struct
chan     else      goto    package    switch
const    fallthrough  if   range      type
continue for       import  return     var
```

`int`、`string`、`bool`、`error`、`true`、`false`、`nil`、`len`、`make` 等**不是关键字**，而是"预声明标识符"——只是在最外层作用域预先声明好的普通标识符，理论上可以被局部变量遮蔽（不推荐）：

```go
int := 100        // 合法：遮蔽了内置类型 int
var x int = 10     // 若在 int := 100 之后同一作用域，会报错 "int is not a type"

var := 10          // 编译错误：var 是关键字，语法解析阶段就不允许这样用
```

区分标准：关键字过不了语法解析；预声明标识符能过语法解析，只是语义上通常不该覆盖。

---

## 3. 变量声明语法

`var` 声明中，类型写在变量名**后面**，与 C++/Python 顺序相反：

```go
var fs flag.FlagSet   // 变量名 fs，类型 flag.FlagSet
var err error          // 变量名 err，类型 error（内置接口类型）
```

类比：

```cpp
FlagSet fs;   // C++：类型在前
```

```python
fs = FlagSet()   # Python：不写类型
```

`var` 和 `:=` 不能在同一条语句混用：

```go
var x int := 10   // 编译错误：语法互斥，只能选一种声明方式
```

---

## 4. 错误处理：没有异常机制

Go 没有 `try/catch`/`throw`，错误通过返回值传递：

```go
result, err := someFunc()
if err != nil {
    // 处理错误
}
```

对比：

```cpp
try { result = someFunc(); } catch (...) { ... }
```

```python
try:
    result = some_func()
except Exception as e:
    ...
```

`log.Fatalf` 打印日志后直接 `os.Exit(1)`，不会执行 `defer`。

---

## 5. `defer`：延迟执行

`defer` 后的语句在函数返回前执行，无论正常返回还是 panic，按后进先出顺序执行：

```go
defer file.Close()
```

类比 C++ 的 RAII（析构函数自动清理），或 Python 的 `try...finally` / `with`。

---

## 6. 多返回值

Go 原生支持函数返回多个值，无需用 tuple/struct 包装：

```go
func divide(a, b int) (int, error) {
    if b == 0 {
        return 0, errors.New("除数不能为0")
    }
    return a / b, nil
}
```

C++ 需要 `std::tuple` 或输出参数模拟，Python 用 tuple 解包模拟，但都不是语言原生的"多返回值"机制。

---

## 7. `for`：唯一的循环关键字

Go 没有 `while`/`do-while`，只有 `for`：

```go
for i := 0; i < 10; i++ { }   // 标准三段式，等价 C++ for

for x < 100 { }                // 当 while 用

for { }                        // 死循环，等价 while(true)

for i, v := range mySlice { }  // 遍历，等价 Python 的 enumerate
```

---

## 8. `switch`：默认不贯穿

```go
switch day {
case 1:
    fmt.Println("周一")
case 2, 3:
    fmt.Println("周二或周三")
default:
    fmt.Println("其他")
}
```

每个 `case` 执行完自动 `break`，与 C++ 相反；想贯穿到下一个 `case` 需显式写 `fallthrough`。

---

## 9. `struct` 与方法：没有 `class`

Go 没有 `class` 关键字，用 `struct` 定义数据，方法在外部通过"接收者"绑定：

```go
type Person struct {
    Name string   // 首字母大写：包外可访问（类似 public）
    age  int      // 首字母小写：仅包内可访问（类似 private）
}

func (p Person) Greet() string {
    return "Hello, " + p.Name
}
```

`(p Person)` 是方法接收者，相当于 C++ 隐式的 `this` 或 Python 显式的 `self`，但写在函数名前面而不是类体内部。

---

## 10. `interface`：隐式实现，没有继承

Go 不支持类继承，用组合（结构体嵌入）代替：

```go
type Animal struct{}
func (a Animal) Eat() { fmt.Println("吃东西") }

type Dog struct {
    Animal   // 嵌入，Dog 拥有 Animal 的方法，但不是 is-a 关系
}
```

接口的实现是隐式的，不需要声明"我实现了某接口"，只要方法签名匹配即可：

```go
type Animal interface {
    Speak() string
}

type Dog struct{}
func (d Dog) Speak() string { return "汪汪" }   // 自动满足 Animal 接口
```

类似 Python 的鸭子类型，区别是 Go 在**编译期**检查，Python 在**运行时**检查。

结论：Go 是面向对象语言，支持封装和多态，不支持类继承，用"组合优于继承"的设计哲学替代。

---

## 11. `go`：启动协程

```go
go doSomething()   // 异步执行，不阻塞当前流程
```

类比 C++ 的 `std::thread`（但更轻量，类似绿色线程）或 Python 的 `asyncio.create_task`/`threading.Thread`，但语法上是普通函数调用加一个关键字，不需要额外的异步语法。

---

## 12. `chan` 与 `select`：协程间通信

```go
ch := make(chan int)       // 创建 channel，类似线程安全的队列
go func() { ch <- 42 }()   // 发送
val := <-ch                 // 接收

select {
case v := <-ch1:
    fmt.Println(v)
case <-time.After(5 * time.Second):
    fmt.Println("超时")
}
```

`chan` 类比 Python 的 `queue.Queue`；`select` 类比同时监听多个队列，谁先到就处理谁。

---

## 13. `context.Context`：取消信号传递

`Context` 在函数间传递取消信号和超时控制：

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
```

类比 C++20 的 `std::stop_token`，或手动传递一个共享的取消标志位；类比 Python 中 `asyncio` 的任务取消机制。

---

## 14. 语言定位

Go 是静态类型、编译型的多范式语言：

```text
面向过程   完全支持
面向对象   支持封装与多态，不支持继承，用组合 + 接口实现
函数式     函数是一等公民，可作参数/返回值
并发编程   语言级原生支持（goroutine + channel）
```

---

## 15. 核心概念对照表

| Go | C++ | Python |
|---|---|---|
| `error` 返回值 | 错误码 / `std::expected` | 一般用异常，非返回值检查 |
| `defer` | RAII | `try/finally` / `with` |
| `go func(){}()` | `std::thread`（更重） | `asyncio.create_task` |
| `context.Context` | 自定义 stop_token | `asyncio` 取消信号 |
| `chan` | 队列 + 条件变量 | `queue.Queue` |
| `:=` | `auto` | 普通赋值 |
| `struct{Field: val}` | 聚合初始化 `{.field=val}` | 关键字参数构造 |
| `NewXxx()` 函数 | 构造函数 | `__init__` |
| `interface`（隐式实现） | 抽象基类（需显式继承） | 鸭子类型（运行时） |
| 结构体嵌入 | 无直接对应 | 无直接对应（组合模式） |
