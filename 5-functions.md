# 5 Functions

### 5.1 函数声明

函数的类型称作函数签名。

```go
func add(x, y int) int { return x + y }
fmt.Printf("%T\n", add) // func(int, int) int
```

Go 语言函数中没有默认参数值的概念也不能通过名字指定参数。参数传递方式为值传递，所以函数接收到的是参数的副本。

### 5.2 递归

...

### 5.3 多返回值

...

### 5.4 错误

一个错误可能是空值（nil）或非空值，空值意味着成功而非空值意味着失败，且非空的错误类型有一个错误消息字符串，可以通过调用它的 Error 方法或者调用输出函数直接输出错误消息。

与许多其他语言不同，Go 语言通过使用普通的值而非异常来报告错误。尽管 Go 语言有异常机制，这将在 5.9 节进行介绍，但是 Go 语言的异常只是针对程序 bug 导致的预料外的错误。而不能作为常规的错误处理方法出现在程序中。这样做的原因是异常会陷入带有错误消息的控制流去处理它。

### 5.5 函数的值

函数在 Go 语言中是头等重要的值：就像其他值，函数变量也有类型，而且它们可以赋给变量或者传递或者从其他函数中返回。

函数值让我们不仅能在数据上对函数进行参数化，还能在行为上对函数进行参数化。

```go
func add1(r rune) rune { return r+1 }
fmt.Println(strings.Map(add1, "Admix")) // Benjy
```

### 5.6 匿名函数

命名函数只能在包级别的作用域进行声明，但我们能够使用函数字面量在任何表达式内指定函数值。匿名函数可以获取到整个词法环境，因此里层的函数可以使用外层函数中的变量：

```go
func squares() func() int {
    var x int
    return func() int {
        x++
        return x * x
    }
}

func main() {
    f := squares()
    fmt.Println(f()) // 1
    fmt.Println(f()) // 4
    fmt.Println(f()) // 9
    fmt.Println(f()) // 16
}
```

（💡 squares() 返回了内层函数，内层函数的词法环境有一个变量 x，x 每次访问都会增加 1）示例函数 squares 说明了函数不仅仅是代码，还能拥有状态。内层匿名函数能够访问并更新外围函数（enclosing function）squares 的局部变量。被隐藏的变量引用正是我们把函数归为引用类型的原因，也正因为此函数值之间不可比较。这样的函数值使用了一个被叫做闭包（closure）的技巧。（💡 闭包函数的 “闭包” 是在描述这个函数处于一个特定词法环境的状态，**闭包函数使得其外围函数中被其引用的变量的生命周期被延长**）这里我们再一次看到这个例子里面变量的生命周期不是由其作用域所决定的。

当一个匿名函数需要进行递归，需要先声明一个变量然后再将匿名函数赋值给这个变量：

```go
func topoSort(...) ... {
    var visitAll func(items []string)
    visitAll = func(items []string) {
        ...
        if ... {
            visitAll(...)
        }
    }
}
```

如果将两个步骤合并成一个声明，函数字面量将不能存在于 visitAll 变量的作用域中，这样也不能递归地调用自己了：

```go
func topoSort(...) ... {
    visitAll := func(items []string) {
        ...
        if ... {
            visitAll(...) // compile error: undefined: visitAll
        }
    }
}
```

#### 5.6.1 警告：捕获迭代变量

在闭包函数引用了外围局部变量时需要留意：

```go
var prints []func()

for i := 0; i < 5; i++ {
    prints = append(prints, func() {
        fmt.Print(i, " ")
    })
}

for _, f := range prints {
    f()
}
// 5 5 5 5 5
```

✨ 都输出 5 是因为 5 个闭包函数引用的是同一个外围局部变量 i 。

```go
var prints []func()

for i := 0; i < 5; i++ {
    num := i
    prints = append(prints, func() {
        fmt.Print(num, " ")
    })
}

for _, f := range prints {
    f()
}
// 0 1 2 3 4
```

for 循环为每个 闭包函数都独立声明了一个新的变量 num，存储引用的值。

### 5.7 变长函数

```go
func printVar(format string, v ...interface{}) {
    fmt.Printf(format, v...)
}

printVar("%s: %d\n", "total", 100)
```

### 5.8 延迟函数调用

延迟执行的函数在返回表达式更新函数返回值之后执行。

```go
func double(x int) (result int) {
    defer func() { fmt.Printf("double(%d) = %d"), x, result }
    return x + x
}
_ = double(4) // double(4) = 8
```

```go
func triple(x int) (result int) {
    defer func() { result += x}()
    return double(x)
}
```

💡 可以理解为：

```go
result = double(x)
result += x
return result
```

defer 语句不到函数最后一刻是不会执行的，所以在循环访问、释放资源时要特别小心。（💡可以理解为，defer 在返回值计算完毕后，函数返回的前一步执行）

```go
for _, filename := range filenames {
    f, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer f.Close() // 文件描述符在所在函数返回前并不会释放，所以可能用尽文件描述符
}
```

延迟函数倒序执行：

```go
defer fmt.Println(1)
defer fmt.Println(2)
defer fmt.Println(3)
fmt.Printf("%s: %d\n", "total", 100)
// total: 100
// 3
// 2
// 1
```

### 5.9 panic

在运行时发生的错误，会触发panic。当 panic 发生时，Go routine 正常的程序执行会中止，所有已调用的 defer 函数会倒序执行。

对于一个不应该发生的情况，panic 时最好的处理方式：

```go
switch ... {
case ...:
case ...:
case ...:
default:
    panic("invalid ...")
}
```

由于 panic 会引起程序异常退出，因此只有在发生严重错误时才会使用 panic。

### 5.10 recover

```go
func Parse(input string) (s *Syntax, err error) {
    defer func() {
        if p := recover(); p!= nil {
            err = fmt.Errorf("internal error: %v", p)
        }
    }
}
```

在 defer 调用的函数中加入 recover 使程序能够从 panic 状态恢复。

```go
func panicRecover() {
    defer func() {
        if p := recover(); p != nil {
            fmt.Fprintln(os.Stderr, p)
        }
    }()
    for i := 3; i >= 0; i-- {
        fmt.Println(6 / i)
    }
    fmt.Println("panicRecover end")
}

func TestPanicRecover(t *testing.T) {
    panicRecover()
    fmt.Println("end")
}
// 2
// 3
// 6
// runtime error: integer divide by zero
// end
```

对同一个包内发生的 panic 进行恢复有助于简化处理复杂和未知的错误，但不应恢复另外一个包内发生的 panic。
