# 7 Interfaces

Go 语言的接口的特别之处在于它是隐式适配，对于一个具体的类型，无需声明它声明的所有接口，拥有这些接口所需要的方法即可。这种设计使得无需改变已有类型的实现，就能为这些类型创建新的接口。

### 7.1 接口即合约

参数列表中的接口定义了函数和调用者的合约，一方面，这个合约要求调用者提供的类型实现了这个接口，另一方面，这个合约保证了函数能使用任何满足这个接口的参数。

```go
func Fprintf(w io.Writer, format string, args ...interface{}) (int, error)
```

fmt.Fprintf 仅依赖于 io.Writer 接口所约定的方法，对参数的具体类型没有要求，所以我们可以用任何满足 io.Writer 接口的具体类型作为 fmt.Fprintf 的第一个参数。

### 7.2 接口类型

和嵌入结构提类似，接口也可以嵌入

```go
type ReadWriter interface {
    Reader
    Writer
}
```

### 7.3 接口适配（Interface Satisfaction）

如果一个类型拥有一个接口要求的所有方法，那么这个类型就适配了这个接口。

接口的赋值规则很简单：仅当类型适配这个接口时，一个表达式才能够对这个接口进行赋值。所以：

```go
var w io.Writer
w = os.Stdout
w = new(bytes.Buffer)
w = time.Second // compile error
```

```go
var rwc io.ReadWriteCloser
rwc = os.Stdout
w = rwc // ok
rwc = w // compile error
```

运用接口可以限制对变量的操作：

```go
var w io.Writer
w = os.Stdout
w.Write([]byte("hello"))
w.Close() // compile error: io.Writer lacks Close method
```

✨ **一个拥有更多方法的接口，比如 io.ReadWriter，与 io.Reader 相比，给了我们它所指向数据的更多信息。**空接口类型 interface{} 完全不包含任何信息与方法，对其实现类型没有任何要求，所以我们可以把任何值赋给空接口类型。

### 7.4 使用 flag.Value 来解析参数

支持自定义类型只需定义一个满足 flag.Value 接口的类型，其定义如下：

```go
package flag
type Value interface {
    String() string
    Set(string) error
}
```

💡 一个简单实现：

```go
package chapter7

type FlagInt int

func (t *FlagInt) String() string {
    return fmt.Sprint(*t)
}
func (t *FlagInt) Set(s string) error {
    num, err := strconv.Atoi(s)
    if err != nil {
        return err
    }
    *t = FlagInt(num)
    return nil
}
func IntFlag(name string, value FlagInt, usage string) *FlagInt {
    f := value
    flag.CommandLine.Var(&f, name, usage)
    return &f
}
```

使用：

```go
fi := chapter7.IntFlag("flagInt", 1, "...")
flag.Parse()
fmt.Println(fi) // 10
```

### 7.5 接口值

从概念上讲，一个接口类型的值，或者说接口值，由两个部分组成：一个具体的**类型**和该类型的一个**值**。分别称之为**动态类型**和**动态值**。

在我们的概念模型中，一组被称为**类型描述符**的值提供了每个类型的信息，如名称信息以及方法信息。在一个接口值中，类型部分就是用对应的类型描述符来表示。如下四个语句中，变量 w 有三个不同的值：

```go
var w io.Writer
w = os.Stdout
w = new(bytes.Buffer)
w = nil
```

* 第一个语句声明了 w，接口的零值就是把它的动态类型和值都设置为 nil（初始值取决于类型）。
* 第二个语句把一个 \*os.File 类型的值赋给了 w，这次赋值把一个具体类型隐式转换为一个接口类型。接口值的动态类型会被设置为指针类型 \*os.File 的类型描述符，它的动态值会设置为 os.Stdout 的副本，一个代表进程标准输出的 \*os.File 指针。
* 第三个语句把一个 \*bytes.Buffer 类型的值赋给了接口值。这时动态类型是 \*bytes.Buffer，动态值为一个指向新分配缓冲区的指针。
* 最后把 nil 赋给了接口值。

接口值可以用 == 和 != 操作符来比较。如果两个接口值都是 nil 或者二者的动态类型完全一致且二者动态值相等，那么两个接口值也相等。

#### 7.5.1 注意：动态值为 nil 的接口值并不是 nil

```go
func main() {
    var buf *bytes.Buffer
    f(buf)
}

func f(out io.Writer) {
    if out != nil {
        out.Write([]byte("done!\n")) // panic: nil pointer derefernce.
    }
}
```

buf 为 nil 时，赋值给 out 之后 out 并不是 nil，out 的动态值为 nil，但是动态类型为 \*bytes.Buffer。

### 7.6 使用 sort.Interface 来排序

```go
package sort
type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}
```

实现接口即可调用 sort.Sort(instace) 来对具体的实例进行排序。

sort.Reverse 值得留意。sort 包定义了一个未导出的类型 reverse，这个类型是一个嵌入了 sort.Interface 的结构，具体如下：

```go
package sort
type reverse struct{ Interface } // sort.Interface
func (r reverse) Less(i, j int) bool { return r.Interface.Less(j, i) }

func Reverse(data Interface) Interface { return reverse{data} }
```

reverse 类型即覆盖了一下 Interface 类型的 Less 函数，将参数调换顺序来改变了一下大小关系。使用方式如下：

```go
a := []int{1, 6, 3, 9, 0, 3, 4, 2, 8, 7}
sort.Sort(sort.Reverse(sort.IntSlice(a)))
fmt.Println(a) // [9 8 7 6 4 3 3 2 1 0]
```

### 7.7 http.Handler 接口

skipped

### 7.8 error 接口

error 类型实际上只是包含一个返回错误信息方法的接口：

```go
type error interface {
    Error() string
}
```

构造 error 最简单的方法是调用 errors.New，它会返回一个包含指定的错误消息的 error 实例。

```go
func New(text string) error {
    return &errorString{text}
}

// errorString is a trivial implementation of error.
type errorString struct {
    s string
}

func (e *errorString) Error() string {
    return e.s
}
```

比直接调用 errors.New 更常见的方式是 fmt.Errorf：

```go
func Errorf(format string, args ...interface{}) error {
    return errors.New(Sprintf(format, args...))
}
```

### 7.10 类型断言

类型断言是一个作用在接口值上的操作，x.(T)，其中 x 是接口类型的表达式，T 是断言类型。类型断言会检查作为操作数的动态类型是否满足指定的断言类型。

```go
var w io.Writer = os.Stdout
f, ok := w.(*os.File) // ok: true
b, ok := w.(*bytes.Buffer) // ok: false
```

💡 示例：

```go
type Talkable interface {
    Talk()
}
type Movable interface {
    Move()
}
```

Bee 仅适配 Movable 接口

```go
type Bee struct {
    name string
}
func (b *Bee) Move() {
    fmt.Println(b.name, "fly")
}

var move Movable = &Bee{"james"}
talk, ok := move.(Tableable) // <nil>, false
```

Cat 适配 Talkable 以及 Movable 接口

```go
type Cat struct {
    name string
}
func (c *Cat) Talk() {
    fmt.Println(c.name, "meow")
}
func (c *Cat) Move() {
    fmt.Println(c.name, "walks silently")
}

var move Mover = &Cat{"james"}
cat, ok := move.(*Cat) // &{james}, true
talk, ok := move.(Talkable) // &{james}, true
bee, ok := move.(*Bee) // <nil>, false
```

### 7.11 使用类型断言来识别错误

判断错误类型也许可以通过检查是否包含特定字符串来判断：

```go
func IsNotExist(err error) bool {
    return strings.Contains(err.Error(), "file does not exist")
}
```

但这种方式并不健壮，更可靠的方法是用专门的类型来表示结构化的错误值。

```go
package os
type PathError struct {
    Op string
    Path string
    Err error
}

func (e *PathError) Error() string {
    return e.Op + " " + e.Path + ": " + e.Err.Error()
}
```

```go
func IsNotExist(err error) bool {
    if pe, ok := err.(*PathError); ok {
        err = pe.Err
    }
    return err == syscall.ENOENT || err == ErrNotExist
}
```

### 7.12 通过接口类型断言来查询特性

定义一个局部的接口，通过是否实现接口来判断是否具有某些方法

```go
func writeString(w io.Writer, s string) (n int, err error) {
    type stringWriter interface {
        WriteString(string) (n int, err error)
    }
    if sw, ok := w.(stringWriter); ok {
        return sw.WriteString(s) // 高性能
    }
    return w.Write([]byte(s)) // 重新分配内存
}
```

标准库提供了 io.WriteString，这也是向 io.Writer 写入字符串的推荐方法。

### 7.13 类型分支

```go
switch x.(type) {
case nil: //...
case int, uint: //...
case bool: //...
case string: //...
default: //...
}
```

### 7.15 一些建议

当设计一个新 package 时，一个新手 Go 程序员常常先创建一系列接口然后再定义具体的适配这些接口的类型。这样会产生很多只有一个实现的接口，不要这样做。这种接口是不必要的抽象，还有运行时的成本。**仅在有两个或者多个具体类型需要按统一的方式处理时才需要接口**。

Go 语言对面向对象编程风格有很好的支持，但这不代表你只能采用这种方式。并不是所有的东西都应该是一个对象；应该允许函数单独存在，也应该允许未封装数据的存在。
