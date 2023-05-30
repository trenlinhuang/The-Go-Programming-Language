# 6 Methods

### 6.1 方法声明

Go 和其他面向对象的语言不同，它可以将方法绑定到任何类型上。可以在当前包为包内任何**除指针和接口**之外的类型声明方法。

💡 指针类型无法声明方法（可能是为了避免和指针接收者混淆）：

```go
type pint *int
func (p pint) aMethod() { // Invalid receiver type 'pint'
    //
}
```

### 6.2 指针接收者的方法

由于函数调用时参数传递方式为值传递，当希望改变 receiver 的值时，method 的 receiver 应该使用对应的指针类型：

```go
func (p *Point) ScaleBy(factor float64) {
    p.X *= factor
    p.Y *= factor
}
```

习惯上遵循如果一个类型的任何一个方法使用指针接收者，那么它所有的方法都应该使用指针接收者。命名类型与指向它们的指针为了防止混淆，不允许本身是指针的类型进行方法声明：

```go
type P *int
func (P) f() { ... } // 编译错误：非法的接收者类型
```

如果 p 是 Point 类型的变量，但方法要求接收者类型为 \*Point，我们仍然可以直接调用：

```go
p := Point{1, 2}
p.ScaleBy(2)
```

**编译器会对变量进行 \&p 的隐式转换**。但不能够对一个不能取地址的 Point 接收者调用 \*Point 方法，因为无法获取变量的地址。

```go
Point{1, 2}.ScaleBy(2) // 编译错误：不能获得 Point 字面量的地址
```

💡 但是能调用接收者类型为 Point 的方法：

```go
func (p Point) Print() {
    fmt.Println(p.X, p.Y)
}

Point{1, 2}.Print() // 1 2
```

反过来，如果变量类型为 \*Point，它是可以合法调用接收者类型为 Point 的方法的。

#### 6.2.1 nil 是合法的接收者

```go
func (list *IntList) Sum() int {
    if list == nil {
        return 0
    }
    return list.Value + list.Tail.Sum()
}
```

### 6.3 通过结构体嵌入构造的类型

```go
type Point struct { X, Y float64 }
type ColoredPoint struct {
    Point
    Color color.RGBA
}
```

我们能够通过类型为 ColorPoint 的接收者调用内嵌类型 Point 声明的方法。

匿名字段类型可以是个指向命名类型的指针，这个时候，字段和方法间接地来自于所指向的对象。

```go
type ColoredPoint struct {
    Point
    color.RGBA
}
```

结构体类型可以拥有多个匿名字段，除了自己声明的方法外，还拥有所有匿名字段所拥有的方法。当编译器在解析 selector（💡 selector 即 "."）如 p.ScaleBy 的时候，它首先查找是否声明了 ScaleBy 方法，再从匿名字段 Point、RGBA 查找是否存在被提升一次的 ScaleBy 方法，然后再从内嵌于 Point、RGBA 的匿名字段中查找是否存在被提升两次的 ScaleBy 方法，以此类推。当同一个查找级别中有同名方法时，编译器会报告 selector 不明确的错误。

内嵌机制使得未命名结构体类型也能拥有方法，有时候这种机制能派上用场：

```go
var cache = struct {
    sync.Mutex
    mapping map[string]string
} {
    mapping: make(map[string]string)
}

func Lookup(key string) string {
    cache.Lock()
    defer cache.Unlock()
    return cache.mapping[key]
}
```

### 6.4 方法变量与表达式

方法变量可以绑定接收者，从而函数只需要提供参数即可调用：

<pre class="language-go"><code class="lang-go">p := Point{1, 2}
q := Point{4, 6}
<strong>distanceFromP := p.Distance // 方法变量
</strong>distanceFromP(q) // 5
</code></pre>

方法表达式写成 T.f 或者 (\*T).f，生成了一个以接收者为第一个参数的函数变量：

```go
p := Point{1, 2}
q := Point{4, 6}

distance := Point.Distance // 方法表达式
fmt.Println(distance(p, q)) // 5
fmt.Printf("%T\n", distance) // func(Point, Point) float64

scale := (*Point).ScaleBy
fmt.Printf("%T\n", scale) // func(*Point, float64)
scale(&p, 2)
```

### 6.5 示例：位向量

...

💡 实现 String 方法的类型在输出时会根据 String 方法进行输出

```go
func (p *Point) String() string {
    return fmt.Sprintf("x=%d, y=%d", p.X, p.Y)
}

p := &Point{1, 2}
fmt.Println(p) // x=1, y=2

p := Point{1, 2}
fmt.Println(p) // {1 2}
```

### 6.6 封装

如果一个方法或变量被封装，那么我们不能在外部通过对象对其进行访问。Go 唯一实现封装的方式是通过标识符首字母的大小写来表示其可见性。要封装一个对象，实现对其访问机制的控制，必须构造一个结构体：

```go
type IntSet struct {
    words []uint64
}
```

在 Go 中，封装的单元是包而不是类型。无论是函数、方法还是结构体中的字段，对于同一个包中的所有代码都是可见的。
