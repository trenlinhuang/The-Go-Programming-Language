# 9 Concurrency with Shared Variables

### 9.1 竞态

如果我们无法自信地说一个事件肯定先于另一个事件，那么这两个事情就是并发的。竞态是指在多个 goroutine 的一些操作交错执行的情形下程序不能给出正确的结果的情况。

数据争用（data race）发生于多个 goroutine 并发读写同一个资源并且至少其中一个是写入时。有三种方法来避免数据争用。

**第一种是不修改变量。**考虑如下 map，如果 Icon 调用是并发的，访问 map 时就存在数据争用：

```go
var icons = make(map[string]image.Image)
func loadIcon(name string) image.Image
func Icon(name string) image.Image {
    icon, ok := icons[name]
    if !ok {
        icon = loadIcon(name)
        icons[name] = icon
    }
    return icon
}
```

如果在访问前初始化好所有的 icon，函数只读取不修改 icons 的状态就可以避免数据争用。

**第二种方法是避免从多个 goroutine 访问同一个变量**。变量的修改操作被限制（confined）在一个 goroutine 中，其他 goroutine 必须使用通道来向限制（confining） goroutine 发送查询请求或者更新变量。这也是 “**不要通过共享内存来通信，而应该通过通信来共享内存**” 的含义。

**第三种方式是限制同一时间只有一个 goroutine 可以访问变量的互斥机制**。

### 9.2 互斥锁：sync.Mutex

```go
mu.Lock()
balance = balance + amount // 临界区
mu.Unlock()
```

这种函数、互斥锁、变量的组合方式称为监控（monitor）模式。

```go
mu.Lock()
defer mu.Unlock()
balance = balance + amount
```

defer 的 Unlock 即使在临界区 panic 的情况下也会正确执行，这种方式在使用了 recover 的程序（💡 使用 recover 也代表着已经预料到会有 panic 的情况）中可能非常重要。当然，defer 的执行成本比显式调用 Unlock 略大一些，但这并不足以称为代码不清晰的理由。✨ **在处理并发程序时，永远应当优先考虑清晰度，并且拒绝过早优化**。

### 9.3 读写互斥锁：sync.RWMutex

允许只读操作可以并发执行，但写操作需要获得完全独享的访问权限。这种锁称为多读单写锁。

```go
var mu sync.RWMutex
mu.RLock()
mu.RUnlock()
mu.Lock()
mu.Unlock()
```

### 9.4 内存同步

现代计算机一般都具有多个处理器，每个处理器都有主存的本地缓存。为了提高效率，对内存的写入是缓存在每个处理器种的，只在必要时才刷回内存。甚至有可能提交到主存的顺序和 goroutine 执行写操作时的顺序不同。

```go
var x, y int
go func() {
    x = 1
    fmt.Print("y:", y, " ")
}()
go func() {
    y = 1
    fmt.Print("x:", x, " ")
}()
```

程序是有可能输出 x:0 y:0 或者 y:0 x:0 的。**尽管很容易把并发简单理解为多个 goroutine 中语句的某种交错执行方式**，但正如上面的例子所显式的，**这并不是一个现代编译器和 CPU 的工作方式**。因为赋值和 Print 对应不同的变量，所以编译器就可能会认为两个语句的执行顺序不会影响结果，然后就交换了这两个语句的执行顺序。如果两个 goroutine 运行在了不同的 CPU 上，每个都有自己独立的 cache。一个 goroutine 的写入操作在同步到内存之前对于另一个 goroutine 来说可能是不可见的。

### 9.5 延迟初始化：sync.Once

sync.Once 是并发安全的只执行一次的解决方案，可以用于延迟初始化。

```go
func loadIcons() {
    //...
}

var loadIconsOnece sync.Once
var icons map[string]image.Image

// 并发安全
func Icon(name string) image.Image {
    loadIconsOnce.Do(loadIcons)
    return icons[name]
}
```

从概念上来说，Once 包含一个布尔变量和一个互斥量，布尔变量记录初始化是否已经完成，互斥量则负责保护这个布尔变量和客户端的数据结构。每次调用 Do 时会先锁定互斥量并检查里面的布尔变量。在第一次调用时，这个布尔变量为假，Do 会调用 loadIcons 然后把变量设置为真。后续的调用相当于空操作，只是通过互斥量的同步来保证 loadIcons 对内存产生的效果对所有的 goroutine 可见。以这种方式来使用 sync.Once，可以避免变量在被正确构造之前被其他 goroutine 共享。

### 9.6 争用检测器（The Race Detector）

想要写出没有并发错误的代码是困难的。幸运的是 Go 提供了争用检测器。

简单地将 -race 标志加到 go build，go run，go test 命令即可。编译器会为你的应用或测试构建一个修改后的版本，有着能够高效地记录程序执行时共享资源的访问信息的设施，信息包括读或写资源的 goroutine 实体。

竞争检测器报告所有已经执行部分的数据争用。它只能检测运行时是否存在竞态；但不能证明不会有竞态出现。

### 9.7 示例：并发非阻塞缓存

并发非阻塞的缓存系统能够解决函数记忆问题，即缓存函数的结果，达到多次调用但只需计算一次的效果。

```go
type Func func(key string) (interface{}, error)
type result struct {
    value  interface{}
    err    error
}
type entry struct {
    res result
    ready chan struct{} // res 准备好后 close 掉
}
```

方案1：利用锁互斥访问

```go
type Memo struct {
    f     Func
    mu    sync.Mutex // 互斥访问 cache
    cache map[string]*entry
}
func New(f Func) *Memo {
    return &Memo{f: f, cache: make(map[string]*entry)}
}
func (memo *Memo) Get(key string) (value interface{}, err error) {
    memo.mu.Lock()
    e := memo.cache[key]
    if e == nil {
        e = &entry{ready: make(chan struct{})}
        memo.cache[key] = e
        memo.mu.Unlock()
        
        e.res.value, e.res.err = memo.f(key)
        close(e.ready)
    } else {
        memo.mu.Unlock()
        <-e.ready // 在被 close 后不会被阻塞
    }
    return e.res.value, e.res.err
}
```

方案2：将临界资源的访问放入一个监控 goroutine 中

```go
type request struct {
    key        string
    response   chan<- result
}
type Memo struct { requests chan request }

func New(f Func) *Memo {
    memo := &Memo{ requests chan request }
    go memo.server(f)
    return memo
}
func (memo *Memo) Get(key string) (interface{}, error) {
    response := make(chan response)
    memo.requests <- request{key, response}
    res := <-response
    return res.value, res.err
}
func (memo *Memo) Close() { close(memo.requests) } // 非关键
func (memo *Memo) server(f Func) {
    cache := make(map[string]*entry)
    for req := range memo.requests {
        e := cache[req.key]
        if e == nil {
            // 对这个key第一次请求
            e = &entry{ready: make(chan struct{})}
            cache[req.key] = e
            go e.call(f, req.key)
        }
        go e.deliver(req.response)
    }
}
func (e *entry) call(f Func, key string) {
    e.res.value, e.res.err = f(key)
    close(e.ready)
}
func (e *entry) deliver(response chan<- result) {
    <-e.ready
    response<- e.res
}
```

### 9.8 goroutine 与线程

#### 9.8.1 可增长的栈

每个 OS 线程都有一个固定大小的栈内存（通常为 2MB）。对于一个小的 goroutine，2MB 的栈是一个巨大的浪费。另外，对于大多数复杂且深度递归的函数，固定大小的栈常常不够大。

相反，一个 goroutine 在声明周期开始时只有一个很小的栈，通常 2KB。与 OS 线程相似，goroutine 的栈也用于存放那些正在执行或临时暂停的函数中的局部变量。但不同的是，**goroutine 栈的大小是按需增大和缩小的**。

#### 9.8.2 goroutine 调度

OS 线程由 OS 内核来调度。每隔几毫秒，一个硬件时钟中断发到CPU，CPU调用一个叫调度器的内核函数，保存运行线程的状态到内存，再恢复另外一个线程的状态，最后更新调度器的数据结构。这个操作是缓慢的。

Go 的运行时包含一个自己的调度器，使用了一个被称为 m:n 调度的技术，因为它在 n 个 OS 线程上复用（或调度）m 个 goroutine。

Go 调度器是由特定的 Go 语言结构来触发的。比如当一个 goroutine 调用 time.Sleep 或被通道阻塞或对互斥量操作时，调度器进行调度。

#### 9.8.3 GOMAXPROCS

Go 调度器使用 GOMAXPROCS 参数来确定可以使用多少个 OS 线程来同时执行代码，默认值是机器上的 CPU 数量。正在休眠或者正被通道通信阻塞的 goroutine 不需要占用线程。阻塞在 I/O 和其他系统调用以及调用其他语言写的函数的 goroutine 确实需要一个 OS 线程来管理，但 GOMAXPROCS 不将这些 goroutine 的数量计入。

#### 9.8.4 goroutine 没有 id

大多数支持多线程的操作系统和编程语言的线程都有一个独立的 id。这让我们可以轻松地构建一个线程的线程本地存储，实质上即一个以线程 id 为 key 的全局 map。

goroutine 没有可供程序员访问的标识。这是刻意设计的，因为线程的本地存储机制有可能被滥用。
