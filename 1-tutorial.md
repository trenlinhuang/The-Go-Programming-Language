# 1 Tutorial

### 1.1 hello, world

import声明必须跟在package声明之后。import导入声明后面，是组成程序的函数、变量、常量、类型（func, var, const, type）。Go不需要再语句或声明后面使用分号结尾，除非有多个语句或声明出现在同一行。事实上，跟在特定符号后面的换行符被转换为分号。换行会影响Go代码的解析，如"{"必须和关键字func在同一行，在 x+y 中，换行符可以在 + 后面，但不能在 + 前面。

```go
fmt.Println(x+
y)
```

Go对代码的格式化要求非常严格。gofmt工具将代码以标准格式进行重写，许多IDE可以配置为在每次保存文件时自动运行gofmt。

### 1.2 命令行参数

变量 os.Args 是一个字符串 slice，第一个元素为命令本身，另外的元素为执行时的参数。

for range语句中，每一次迭代range产生一对值：索引和索引元素的值。

```go
for index, element range elements {
    ...
}
```

### 1.3 找出重复行

**map中的迭代顺序是不固定的，每次运行都不一致**

bufio包可以简便高效地处理输入和输出。其中Scanner可以以行或单词为单位读取输入。

```go
input := bufio.NewScanner(os.Stdin)
```

| verb       | description                        |
| ---------- | ---------------------------------- |
| %d         | 十进制数                               |
| %x, %o, %b | 十六进制、八进制、二进制数                      |
| %f, %g, %e | 浮点数，%e科学计数法，%g根据情况选择%e或%f（无末尾0）的输出 |
| %t         | 布尔型                                |
| %c         | rune类型，数值对应的Unicode字符              |
| %s         | 字符串                                |
| %q         | 带引号的字符串                            |
| %v         | 内置格式的任何值（自动推断类型）                   |
| %T         | 任何值的类型                             |
| %%         | 百分号本身                              |

以 ln 结尾的输出如 Println，则以 %v 为格式进行输出，并自带换行。

bufio.Newscanner(\*os.File) 返回 \*Scanner，Scanner可以对文件进行逐行（换行）扫描。值得注意的是，os.Stdin也是文件类型。

```go
func countLines(f *os.File, counts map[string]int) {
    input := bufio.NewScanner(f)
    for input.Scan() {
        counts[input.Text()]++
    }
}
```

map是一个由make创建的数据结构的引用，函数接收到的是引用的副本，而不是数据的副本。countLines是以行为单位对数据进行处理，读取一行处理一行，是流式读取数据，所以原则上可以处理任意次数的输入。💡 os.Open获取文件类型指针后交给bufio.Scanner处理

```go
data, err := ioutil.ReadFile(filename)
...
for _, line := range strings.Split(string(data), "\n") {
    counts[line]++
}
```

ioutil.ReadFile() 返回整个文件内容的字节切片，转为string后再按换行符分割进行处理。

### 1.4 GIF 动画

...

### 1.5 获取一个URL的内容

<pre class="language-go"><code class="lang-go">func Fetch(url string) {
    resp, err := http.Get(url)
    if err != nil {
        fmt.Fprintf(os.Stderr, "fetch: %v\n", err)
        os.Exit(1)
    }
<strong>    b, err := ioutil.ReadAll(resp.Body)
</strong>    resp.Body.Close()
    if err != nil {
        fmt.Fprintf(os.Stderr, "fetch: reading %s: %v\n", url, err)
        os.Exit(1)
    }
<strong>    fmt.Printf("%s\n", b)
</strong>}
</code></pre>

resp.body 实现了 io.Reader，\[]byte可以通过 %s 直接输出。

### 1.6 并发获取多个URL的内容

<pre class="language-go"><code class="lang-go">start := time.Now()
<strong>secs := time.Since(start)
</strong></code></pre>

time.Since可以计算时差。

### 1.7 一个Web 服务器

<pre class="language-go"><code class="lang-go">func handler(w http.ResponseWriter, r *http.Request) {
<strong>    fmt.Fprintf(w, "%s" r.URL.path)
</strong>}
</code></pre>

Fprintf 向指定 io.Writer 写入。

### 1.8 杂项

#### 流程控制

```go
switch coinflip() {
case "head":
    head++
case "tail":
    tail++
default:
    fmt.Println("landed on edge")
}

switch {
case x > 0:
    return +1
default:
    return 0
case x < 0:
    return -1
}
```

后者称为 `tagless switch` ，等价于 switch true。💡 **switch语句的本质即将switch后的表达式和case后的表达式进行匹配，选项值相同的分支。**（当 switch false {...} 时，则会选择case为false的分支）

💡 值得注意的是，只匹配第一个值相同的分支：

```go
num := 2
switch {
case num > 1:
    fmt.Println("greater than 1")
case num > 0:
    fmt.Println(1)
case num < 0:
    fmt.Println(-1)
default:
    fmt.Println(0)
}
// output: greater than 1
```

break可以跳出case块：

```go
num := 2
switch {
case num > 0:
    fmt.Println(1)
    if num >= 3 {
	break
    }
    fmt.Println("smaller than 3")
case num < 0:
    fmt.Println(-1)
default:
    fmt.Println(0)
}
// output: smaller than 3
```

#### 指针

Go 提供了指针，但不支持指针的算术运算

