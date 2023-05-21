# 4 Composite Types

### 4.1 数组

数组的长度是数组类型的一部分，所以 \[3]int 和 \[4]int 是两种不同的数组类型。数组的长度必须是常量表达式，即这个表达式的值能够在编译时就能计算得出。

```go
q := [3]int{1, 2, 3}
```

数组可以按顺序初始化各个索引对应的值，也可以通过给出键值对的方式来进行初始化：

```go
r := [...]int{99: -1}
```

声明了一个长度为100的 int 数组，最后一个索引为-1，其他均为默认值0。这种情况下，索引可以按照任意顺序出现，并且可省略。

如果元素类型可比较，那么数组也是可比较的，可以直接用 == 和 != 来比较两个数组。

在进行函数传参时，Go把数组看为值传递，在函数内部对数组的影响都仅修改副本而不是原始数组。

### 4.2 slice

**slice 是一种轻量级的数据结构**，可以用来访问数组的部分或者全部的元素，而这个数组称为 slice 的底层数组。**slice 有三个属性：指针、长度和容量**。长度指 slice 中元素的个数，**容量指 slice 的第一个元素到底层数组最后一个元素间的元素个数**。

一个底层数组可以对应多个 slice，这些 slice 可以引用数组的任何位置。

<figure><img src=".gitbook/assets/image (4).png" alt=""><figcaption><p>4.1 月份数组和两个以其为底层数组的切片</p></figcaption></figure>

切片操作超过容量 caps(s) 触发 panic，但超过 len(s) 会扩展数组：

```go
fmt.Println(summer[:20]) // panic: out of range
endlessSummer := summer[:5] // extend a slice
fmt.Println(endlessSummer) // [June July August September October]
```



💡 当有重叠部分时，对一个切片的修改会导致重叠部分的其他切片数据也被修改。当增加切片成员时，若底层数组无法容纳，会将原底层数组复制到一个更大的空间。

<pre class="language-go"><code class="lang-go">a := [...]int{1, 2, 3, 4, 5, 6}
s1 := a[1:]
s2 := a[:]
fmt.Println(cap(s1), cap(s2)) // 5 6
s1[0] = -2
fmt.Println(s1, s2) // [-2 3 4 5 6] [1 -2 3 4 5 6]
s1 = append(s1, 7)
<strong>fmt.Println(cap(s1), cap(s2)) // 10 6
</strong>s1[0] = 0
fmt.Println(s1, s2) // [0 3 4 5 6 7] [1 -2 3 4 5 6]
</code></pre>

当一个函数的参数为切片类型，但数据为数组时，可以为这个数组创造一个切片，对切片的修改即对原数组的修改：

```go
a := [...]int{0, 1, 2, 3, 4, 5}
f(a[:])
```

#### 4.2.1 append 函数

append 对理解 slice 的工作原理很重要，appendInt 函数演示了如何为 \[]int 追加元素：

```go
func appendInt(x []int,y int) []int {
    var z[]int
    zlen := len(x)+1
    if zlen <= cap(x){
        // slice 仍有增长空间，扩展 slice内容
        z = x[:zlen]
    } else {
        // slice已无空间，为它分配一个新的底层数组
        // 为了达到分摊线性复杂性，容量扩展一倍
        zcap := zlen
        if zcap < 2*len(x){
            zcap = 2*len(x)
        }
        z = make([]int，zlen,zcap)
        copy(z，x) // 内置 copy 函数
    }
    z[len(x)]=y
    return z
}
```

💡 所以 append 函数的实际上创建了另外一个切片，而不是修改原切片：

```go
a := make([]int, 3, 10)
b := append(a, 5)
fmt.Println(a, b) // [0 0 0] [0 0 0 5]
```

不仅仅是在调用 append 函数的情况下需要更新 slice 变量。对于任何函数，只要有可能改变 slice 的长度或者容量，或是使得 slice 指向不同的底层数组，都需要更新 slice 变量。虽然底层数组的元素是间接引用的，但是 slice 的指针、长度和容量不是。要更新一个 slice 的指针，长度或者容量必须显式赋值：

```go
runes = append(runes, r)
```

从这个角度看，slice并不是纯引用类型，而是像下面这种聚合类型：

```go
type IntSlice struct {
    ptr              *int
    length, capacity int
}
```

### 4.3 map

map 类型写作 map\[k]v。k必须是可以通过操作符 == 来进行比较的数据类型。

可以通过内置函数 make 来创建一个 map：

```go
ages := make(map[string]int)
ages["alice"] = 31
ages["charlie"] = 34
```

也可以使用 map 字面量来创建：

```go
ages := map[string]int{
    "alice":   31,
    "charlie": 34,
}
```

map 中元素的迭代顺序是不固定的，这样迫使程序在不同的散列算法下变得健壮。

### 4.4 结构体

```go
func NewEmployee() *Employee {
	return &Employee{}
}

m := make(map[string]Employee)
m["a"] = *NewEmployee()
m["a"].Name = "alice" // Cannot assign to m["a"].Name
```

💡 m\["a"] 可以看成返回 value 的一个函数，结构体为值传递，所以并不能改变到原本的结构体成员变量。想要用 map 映射结构体时，值类型应该为结构体指针：

```go
m := make(map[string]*Employee)
```

💡 结构体并非都需要具有名字：

```go
type Employee struct {
	Name string
}

a := struct{ Name string }{"Alice"}
b := struct{ Name string }{"Alice"}
c := Employee{"Alice"}

fmt.Println(a == b) // true
fmt.Println(a == c) // true
```

💡 但是两个具有名字的结构体之间无法直接进行比较：

```go
type People struct {
    Name string
}

c := Employee{"Alice"}
d := People{"Alice"}
fmt.Println(c == d) // Invalid operation: c == d (mismatched types Employee and People)
fmt.Println(c == Employee(d)) // true
```

匿名结构体的成员字段和顺序决定了结构体的类型是否相同：

```go
a := struct {
 Name string
 Age  int
}{"Alice", 1}
b := struct {
 Age  int
 Name string
}{1, "Alice"}

fmt.Println(a == b) // Invalid operation: a == b (mismatched types struct {...} and struct {...})
```

没有任何成员变量的结构体称为空结构体，写作 struct{}。它的大小为0，不带有任何信息但有时候可能有用。一些 Go 程序员用它替代被当作集合使用的 map 中的布尔值，来强调只有键是有用的，但这种方式节约的内存很少且语法复杂，所以一般尽量避免这样用。

#### 4.4.1 结构体字面量

结构体字面量有两种格式，第一种为按字段顺序为每个成员变量指定一个值。这种方式在未来扩充结构体成员变量时可能带来麻烦。第二种为用键值的方式指定变量值，未指定的为零值：

```go
anim := gif.GIF{LoopCount: nframes}
```

#### 4.4.2 结构体比较

如果结构体的所有成员都可以比较，那么这个结构体就是可比较的。

#### 4.4.3 结构体嵌套和匿名成员

Go 允许定义不带名称的结构体成员，只需要指定类型即可；这种结构体成员称作匿名成员。这个结构体成员的类型必须是一个命名类型或者指向命名类型的指针。

```go
type Point struct {
    X, Y int
}
type Circle struct {
    Point
    Radius int
}
type Wheel struct {
    Circle
    Spokes int
}
```

匿名成员的成员变量可以直接访问：

```go
var w Wheel
w.X = 8
w.Y = 8
w.Radius = 5
w.Sopkes = 20
```

结构体字面量必须遵循类型定义时的结构：

```go
w = Wheel{Circle{Point{8, 8}, 5}, 20}
```

### 4.5 JSON

JavaScript Object Notation (JSON) 是一种发送和接收格式化信息的标准。JSON 是 JavaScript 值的 Unicode 编码。

把 Go 的数据结构转换为 JSON 称为 marshal。

```go
data, err := json.Marshal(data)
```

Marshal 生成了一个字节 slice，是不带有任何多余空白字符的字符串。

```go
a := struct {
    Name string
    Age  int
}{"Alice", 1}
data, err := json.Marshal(a)
if err != nil {
    log.Fatal(err)
}
fmt.Printf("%T %[1]s", data) // []uint8 {"Name":"Alice","Age":1}
```

字段标签可以指明转换后字段的名称，omitempty 表示如果字段为空忽略此字段：

```go
a := struct {
    Name string `json:"name"`
    Age  int    `json:"age,omitempty"`
}{Name: "Alice"}
...
fmt.Printf("%T %[1]s", data) // []uint8 {"name":"Alice"}
```

💡 需要特别注意的是，当字段值为默认值时也会被忽略。

marshal 的逆操作将 JSON 字符串解码为 Go 数据结构，这个过程叫做 unmarshal。unmarshal 将 JSON 数据按字段名填充到目标结构中，其余的数据则被忽略。unmashal 阶段字段的大小写不敏感。

```go
data := []byte(`{"Name":"Alice","Age":1}`)
a := Employee{}
if err := json.Unmarshal(data, &a); err != nil {
    log.Fatal(err)
}
fmt.Println(a) // {Alice}
```

### 4.6 文本和 HTML 模板

字符串模板：

```go
const templ = `{{.Name}}:
{{.Age}} years old
`
p, err := template.New("people").Parse(templ)
if err != nil {
    log.Fatal(err)
}
e := struct {
    Name string
    Age  int
}{"Alice", 10}
if err := p.Execute(os.Stdout, e); err != nil {
    log.Fatal(err)
}
// Alice:
// 10 years old
```

...
