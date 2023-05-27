# 8 Groutines and Channels

### 8.1 goroutine

当一个程序启动时，它仅有一个调用 main 函数的 goroutine，我们称之为主 goroutine。新的 goroutine 通过 go 语句进行创建。语法上，一个 go 语句实在普通的函数或方法调用前加上 go 关键字前缀。

### 8.2 示例：并发时钟服务

```go
func main() {
    listener, err := net.Listen("tcp", "localhost:8080")
    if err != nil {
        log.Fatal(err)
    }
    for {
        conn, err := listen.Accept()
        if err != nil {
            log.Print(err)
            continue
        }
        handleConn(conn)
    }
}

func handleConn(c net.Conn) {
    defer c.Close()
    for {
        _, err := io.WriteString(c, time.Now().Format("00:00:00\n"))
        if err != nil {
            return
        }
        time.Sleep(1 * time.Second)
    }
}
```

这种方式一次只能服务一个客户端，因为 handleConn 中的 for 循环并不会主动结束，所以导致 main 函数中的 for 循环始终无法结束。当当前客户端的连接断开后，下一个客户端才能访问服务。

<pre class="language-go"><code class="lang-go">for {
    conn, err := listener.Accept()
    if err != nil {
        log.Print(err)
        continue
    }
<strong>    go handleConn(conn)
</strong>}
</code></pre>

将 handleConn(conn) 放在独立的 goroutine 中运行，则不会阻塞 main 函数处理下一个连接。

### 8.4 通道

```go
ch <- x // 发送
x = <- ch // 将接收值赋值到x
<- ch // 丢弃接收值
```

通道支持的第三个操作是 close，它设置一个标志位来表明不再有数据会被发送。在关闭后的通道上发送数据会导致 panic。可以在已关闭的通道上接收还没有被接收完毕的数据，直到通道为空；此后再进行接收操作会立即完成，并得到对应元素类型的零值。

```go
close(ch)
```

#### 8.4.1 无缓冲通道

直接在无缓冲通道上进行发送或接收会被阻塞，除非已经有其他的 goroutine 在通道上被接收或发送操作阻塞。

#### 8.4.2 管道

通道可以用来连接 goroutine，这样一个 goroutine 的输出是另外一个 goroutine 的输入。被称作管道。

```go
x, ok := <- channel
```

接收方可以通过 ok 来判断此通道是否已被关闭。

结束时，关闭每一个通道不是必须的。只有在想要通知接收方所有的数据发送完毕的时候才需要关闭通道。通道也是可以通过垃圾回收器根据它是否可以访问来决定是否回收它，而不是根据它是否已被关闭。

#### 8.4.3 单向通道类型

```go
func f1(ch chan<- int) // 函数中只能向通道发送
func f2(ch <-chan int) // 函数中只能从通道接收
```

#### 8.4.4 缓冲通道

```go
ch = make(chan int, 3) // 未满时写入或非空时读取不会被阻塞
```

下面展示一个使用缓冲通道的例子。向三个镜像地址发送请求，然后只 return 第一个返回的响应：

```go
func mirroredQuery() string {
    response := make(chan string, 3)
    go func() { response <- request("asia.gopl.io") }()
    go func() { response <- request("europe.gopl.io") }()
    go func() { response <- request("americas.gopl.io") }()
    return <- responses
}
```

如果使用无缓冲通道，那么较慢的两个 goroutine 会被阻塞，这种情况叫做 goroutine 泄露，是一个 bug。泄露的 goroutine 不会被自动回收，所以需要确保 goroutine 是可以运行结束的。

### 8.5 并行循环

```go
func makeThumbnails4(filenames []string) error {
    errors := make(chan error)
    
    for _, f := range filenames {
        go func(f string) {
            _, err := thumbnail.ImageFile(f)
            errors <- err
        }(f)
    }
    for range filenames {
        if err := <- errors; err != nil {
            return err
        }
    }
}
```

当有 error 从通道中读取后，函数将返回，不再有 goroutine 读取这个通道，这导致了后续向通道发送 error 的 goroutine 都会被阻塞，导致 goroutine 泄露。最简单的解决方案是使用缓冲通道。

sync.waitGroup 可以适用于更灵活的场景，比如 filenames 并不是通过能获取大小的切片来传递的：

```go
func makeThumbnails4(filenames <-chan string) error
```

### 8.6 示例：并发的 Web 爬虫

限制并发量可以通过设置固定的通道大小来实现，也可以生成固定数量的 goroutine。

### 8.7 select 多路复用

```go
select {
case <-ch1:
    // ...
case x := <-ch2:
    // ...
case ch3 <- y:
    // ...
default:
    // 所有通道均未准备好的情况下
}
```

有 default 的 select 不会被阻塞。

### 8.9 取消

有时候我们需要让一个 goroutine 停止它当前的任务。

在关闭通道后，所有在通道上的读取操作会立即完成，而不会被阻塞。所以，可以设置一个通道，来实现通知所有相关 goroutine 停止工作的目的。

```go
select {
    case <-done:
        return nil // 从当前 goroutine 返回
    default:
}
```

### 8.10 示例：聊天服务器

...

