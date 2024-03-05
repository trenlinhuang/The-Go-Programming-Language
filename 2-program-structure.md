# 2 Program Structure

### 2.1 名称

Go 中的函数名、变量名、常量名、类型名、语句标签、包名都遵循一个简单的规则：以一个字符或下划线以及可能的任意长度的字符、数字及下划线，并区分大小写。

保留关键字：

* **fallthrough:** switch 的 case 语句中 可以使用 fallthrough 强制执行后面的 case 代码，fallthrough 不会判断下一条 case 的 expr 结果是否为 true。

可以用于变量定义的预定义关键字（💡 谨慎使用，会覆盖原有的用法）：

* 常量
  * **iota**
  * **nil**
* 类型
  * **uintptr:** uintptr is an integer type that is large enough to hold the bit pattern of any pointer.
  *   **complex128, complex64:** 复数，实部虚部均为 float(64,32) 类型

      ```go
      var c complex64 = complex(1, 2) // 1+2i
      real(c),imag(c) // 1,2
      ```
  * **rune:** int32，unicode码点
* 函数
  * **cap:** array, slice, channel 占用的空间
  *   **copy:** 将源切片元素复制到目的切片

      ```go
       s1 := []int{1, 2, 3}
       s2 := []int{4, 5, 6, 7, 8}
       copy(s1, s2)
       s1[0] = 1
       fmt.Println(s1) // [1 5 6]
       fmt.Println(s2) // [4 5 6 7 8]
      ```
  * **recover:** 将 panic 的 goroutine 恢复运行\
    recover 仅在延迟函数 defer 中有效，在正常的执行过程中，调用 recover 会返回 nil 并且没有其他任何效果，如果当前的 goroutine 陷入恐慌，调用 recover 可以捕获到 panic 的输入值，并且恢复正常的执行。\
    通常来说，不应该对进入 panic 宕机的程序做任何处理，但有时，需要我们可以从宕机中恢复，至少我们可以在程序崩溃前，做一些操作，举个例子，当 web 服务器遇到不可预料的严重问题时，在崩溃前应该将所有的连接关闭，如果不做任何处理，会使得客户端一直处于等待状态，如果 web 服务器还在开发阶段，服务器甚至可以将异常信息反馈到客户端，帮助调试。\
    defer func() { err := recover(); ... }() // 获取panic传递的上下文并打印\
    [http://c.biancheng.net/view/64.html](http://c.biancheng.net/view/64.html)

Go没有限制名称的长度，但作用域小的名称建议简短，作用域大的名称建议用长且具有意义的名称。

相较于在单词之间使用下划线，Go更倾向于单词间首字母大写的风格。

### 2.2 声明

...

### 2.3 变量

#### 2.3.2 指针

值不一定都有地址，但变量都有地址。使用指针可以在不知道变量的名称的情况下对变量进行间接的读写。

指针对于 flag 包很关键，标识参数的值需要通过指针来访问。**需要注意标识参数需要写到非标识参数前。**

💡 标志参数即用 flag 显式声明的参数

```go
n := flag.Int("i", 100, "int arg")
flag.Parse()
args := flag.Args()
fmt.Println(args)
fmt.Println(*n)
fmt.Println("---flag.Args()---")
for _, arg := range args {
    fmt.Printf("%T, %v\n", arg, arg)
}
fmt.Println("---os.Args---")
for _, arg := range os.Args {
    fmt.Printf("%T, %v\n", arg, arg)
}
```

命令及输出：

```
>go run . -i 10 "a" "b"
[a b]
10
---flag.Args()---
string, a
string, b
---os.Args---
string, C:\Users\***\AppData\Local\Temp\go-build******\b001\exe\**.exe
string, -i
string, 10
string, a
string, b
```

💡 如果在命令行中把标志参数写到非标识参数后，将无法被正常解析：

```
> go run . "a" "b" -i 10
[a b -i 10]
100
---flag.Args()---
string, a
string, b
string, -i
string, 10
---os.Args---
string, C:\Users\acer\AppData\Local\Temp\go-build1468167120\b001\exe\main.exe
string, a
string, b
string, -i
string, 10

```

#### 2.3.3 new函数

new(T) 创建一个未命名的T类型变量，初始化 T 类型的零值，并返回地址。

new 在创建不包含信息的结构时，可能返回相同的地址

```go
a := new(struct{})
b := new(struct{})
fmt.Printf("%p\n", a) // 0x1194b40
fmt.Printf("%p\n", b) // 0x1194b40
fmt.Println(a == b) // true
fmt.Println("=======")
c := new([0]int)
d := new([0]int)
fmt.Printf("%p\n", c) // 0x1194b40
fmt.Printf("%p\n", d) // 0x1194b40
fmt.Println(c == d) // true
```

#### 2.3.4 变量的生命周期

包级别变量的生命周期是整个程序的执行时间。相反，局部变量有个一动态的生命周期：每次执行声明语句时创建一个新的实体，直到其变得不可访问，不可访问后它的存储空间可能会被回收。

### 2.4 赋值

...

### 2.5 类型声明

```go
type name underlyingType
```

type 声明定义一个新的命名类型，它和某个已有类型使用同样的底层类型。**对同一底层类型，命名类型提供了一种区分不同的以及可能不兼容的使用相同底层类型的手段，这样可以避免无意间的类型混淆。**

对于每个类型 T，都有一个对应的类型转换操作 T(x) 将值转换为类型 T。如果两个类型具有相同的底层类型，或者二者都是指向相同底层类型变量的未命名指针类型，那么两者间可以相互转换。（💡 非指针类型变量本身及其指针）

💡 类型转换和其指针类型转换：

<pre class="language-go"><code class="lang-go">type Balance int
type Amount int
var b Balance = 100
fmt.Println(Amount(b)) // 100
var amount Amount = 100
var addressA = &#x26;amount
<strong>var addressB = (*Balance)(addressA)
</strong>fmt.Println(*addressB) // 100
</code></pre>

命名类型的底层类型决定了它的结构和使用方式，以及它支持的内部操作集合，这些内部操作与直接使用底层类型的情况相同。通过 == 和 < 之类的比较操作符，命名类型可以与其相同类型的值或者底层类型相同的**未命名**的值相比较。但是不同命名类型的值不能直接比较：

```go
var f1 tF1 = 0.0
var f float32 = 0.0
fmt.Println(f1 == 0.0) // true
fmt.Println(f == f1) // mismatched types float32 and tF1
fmt.Println(f == float32(f1)) // true
```

### 2.6 包和文件

#### 2.6.2 包初始化

包初始化从初始化包级别变量开始，初始化按照变量声明的顺序进行，有依赖关系时为例外：

```go
var a = b + c // 最后初始化 a
var b = f() // 其次通过调用 f 初始化 b
var c = 1 // 首先初始化 c
func f() int { return c + 1 }
```

如果包有多个 .go 文件，go 工具在将包中的文件交给编译器前会将文件按名字排序，编译器再将文件按接收到的顺序进行初始化。

任意文件可以包含任意数量的 init 函数，💡 init 函数一般用于复杂数据的初始化。

```go
func init() { ... }
```

init 函数不能被调用与引用，但除此之外和普通函数没有其他不同。在每个文件中，init 函数根据声明的顺序自动被执行。

按照程序中导入的顺序，每次初始化一个包，依赖优先初始化。所以包 p 导入包 q 时，q 被完整初始化后再开始 p 的初始化。因此，在 main 函数执行之前，所有的包都被初始化完毕了。

### 2.7 作用域

声明将名字和程序实体关联起来，如一个函数或一个变量。声明的作用域是指用到声明时声明名字的源代码片段。

作用域和生命周期是不同的概念，声明的作用域是程序文本中出现的区域，是编译时属性。变量的生命周期是变量在程序执行期间能被程序的其他部分所引用的起止时间，是运行时属性。💡 作用域是从代码段的角度，限制访问范围，生命周期是指变量的分配和回收的过程。

语法块（syntactic block）是由大括号围起来的一个语句序列。我们可以将这个概念扩展到没有显式被大括号包围的一组声明里面，称为词法块（lexical block）。包含了全部代码的词法块叫做全局块（universe block）。每一个文件，每一个 for, if 和 switch 语句，以及 switch 和 select 语句中的每一个条件，都是写在一个词法块里的。词法块包含语法块。

一个声明的词法块决定声明的作用域大小。内置的类型、函数或常量在全局块中声明且对于整个程序可见。包级别的声明可以被同一个包里的任何文件引用。导入的包的声明是文件级别的，仅可在当前文件引用。💡 导入一个包可以看成是进行了一次声明：

```go
import "fmt" // fmt 即是一个声明
```

不是所有的词法块都对应于显式大括号包围的语句序列，有一些词法块是隐式的。如 for 循环中的迭代变量 i 所在行即为一个隐式块，i 的作用域包括条件、后置语句（i++），以及 for 语句体本身。

短变量声明需要留意作用域：

```go
var cwd string
func init() {
    cwd, err := os.Getwd() // unused: cwd
    if err != nil {
        log.Fatal(err)
    }
}
```

cwd 在函数中被声明为了一个新的局部变量（💡不在同一作用域的短变量声明会创建新变量）

```go
var cwd string
func init() {
    var err error
    cwd, err = os.Getwd()
    if err != nil {
        log.Fatal(err)
    }
}
```

将 err 提前声明则不会出现问题。
