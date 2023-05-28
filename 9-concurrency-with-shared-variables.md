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
