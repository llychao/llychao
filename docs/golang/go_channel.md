# 浅析 go channel

channel 是 goroutine 之间通信的一种方式，可以类比成 Unix 中的进程的通信方式管道。

channel 是 go 中最核心的 feature 之一，因此理解 Channel 的原理对于学习和使用 go 非常重要。

### CSP 模型

传统的并发模型主要分为 Actor 模型和 CSP 模型.

CSP 模型由并发执行实体(进程，线程或协程)，和消息通道组成，实体之间通过消息通道发送消息进行通信

Go 语言的并发模型参考了 CSP 理论，其中执行实体对应的是 goroutine， 消息通道对应的就是 channel。

### channel 介绍

#### channel 创建

channel 使用内置的 make 函数创建，下面声明了一个 chan int 类型的 channel:
`ch := make(chan int)`

#### channel 的读写操作
```go
ch := make(chan int)

// write to channel
ch <- x

// read from channel
x <- ch

// another way to read
x = <- ch
```

**注意：** channel 一定要初始化后才能进行读写操作，否则会永久阻塞。

#### channel 关闭
```go
ch := make(chan int)

close(ch)
```

有关 channel 的关闭，你需要注意以下事项:

* 关闭一个未初始化(nil) 的 channel 会产生 panic
* 重复关闭同一个 channel 会产生 panic
* 向一个已关闭的 channel 中发送消息会产生 panic
* 从已关闭的 channel 读取消息不会产生 panic，且能读出 channel 中还未被读取的消息，若消息均已读出，则会读到类型的零值。从一个已关闭的 channel 中读取消息永远不会阻塞，并且会返回一个为 false 的 ok-idiom，可以用它来判断 channel 是否关闭
* 关闭 channel 会产生一个广播机制，所有向 channel 读取消息的 goroutine 都会收到消息

```go
ch := make(chan int, 10)
ch <- 11
ch <- 12

close(ch)

for x := range ch {
    fmt.Println(x)
}

x, ok := <- ch
fmt.Println(x, ok)


-----
output:

11
12
0 false
```

### channel 的类型

channel 分为不带缓存的 channel 和带缓存的 channel。

#### 无缓存的 channel

从无缓存的 channel 中读取消息会阻塞，直到有 goroutine 向该 channel 中发送消息；同理，向无缓存的 channel 中发送消息也会阻塞，直到有 goroutine 从 channel 中读取消息。

#### 有缓存的 channel

有缓存的 channel 的声明方式为指定 make 函数的第二个参数，该参数为 channel 缓存的容量

`ch := make(chan int, 10)`

通过 len 函数可以获得 chan 中的元素个数，通过 cap 函数可以得到 channel 的缓存长度。

有缓存的 channel 类似一个阻塞队列(采用环形数组实现)。当缓存未满时，向 channel 中发送消息时不会阻塞，当缓存满时，发送操作将被阻塞，直到有其他 goroutine 从中读取消息；相应的，当 channel 中消息不为空时，读取消息不会出现阻塞，当 channel 为空时，读取操作会造成阻塞，直到有 goroutine 向 channel 中写入消息。

```go
ch := make(chan int, 3)

// blocked, read from empty buffered channel
<- ch
```
```go
ch := make(chan int, 3)
ch <- 1
ch <- 2
ch <- 3

// blocked, send to full buffered channel
ch <- 4
```

### channel 的用法

#### range 遍历

channel 使用 range 取值，并且会一直从 channel 中读取数据, 直到有 goroutine 对改 channel 执行 close 操作，循环才会结束。

```go
// consumer worker
ch := make(chan int, 10)
for x := range ch{
    fmt.Println(x)
}
```

等价于

```go
for {
    x, ok := <- ch
    if !ok {
        break
    }
    
    fmt.Println(x)
}
```

#### 配合 select 使用

可以同时监听多个 channel 的消息状态
```go
select {
    case <- ch1:
    ...
    case <- ch2:
    ...
    case ch3 <- 10;
    ...
    default:
    ...
}
```

* select 可以同时监听多个 channel 的写入或读取
* 执行 select 时，若只有一个 case 通过(不阻塞)，则执行这个 case 块
* 若有多个 case 通过，则随机挑选一个 case 执行
* 若所有 case 均阻塞，且定义了 default 模块，则执行 default 模块。若未定义 default 模块，则 select 语句阻塞，直到有 case 被唤醒。
* 使用 break 会跳出 select 块

##### 1. 设置超时时间

```go
ch := make(chan struct{})

// finish task while send msg to ch
go doTask(ch)

timeout := time.After(3 * time.Second)	//func After(d Duration) <-chan Time 
select {
    case <- ch:
        fmt.Println("task finished.")
    case <- timeout:
        fmt.Println("task timeout.")
}
```

#####  2. quite channel

有一些场景中，一些 worker goroutine 需要一直循环处理信息，直到收到 quit 信号

```go
msgCh := make(chan struct{})
quitCh := make(chan struct{})
for {
    select {
    case <- msgCh:
        doWork()
    case <- quitCh:
        finish()
        return
}
```

最后写个demo，大家猜猜运行结果：
```go
var ch chan int
ch = make(chan int,3)

ch <- 1
ch <- 2
fmt.Println("chan len",len(ch))

var wg sync.WaitGroup
wg.Add(1)

go func(){
    for {
        var b int
        b, ok := <-ch
        fmt.Println("chan out",b,ok)
        if ok == false {
            fmt.Println("chan is close")
            break
        }
    }
    // 等价于：
    //for b := range ch{
    //  fmt.Println("chan out",b)
    //}

    fmt.Println("chan done")
    wg.Done()
}()

for i := 0; i < 10; i++ {
    ch <- i
    if i==5{
        time.Sleep(2*time.Second)
    }
}
close(ch)
wg.Wait()
  
-----
output:

chan len 2
chan out 1 true
chan out 2 true
chan out 0 true
chan out 1 true
chan out 2 true
chan out 3 true
chan out 4 true
chan out 5 true
chan out 6 true
chan out 7 true
chan out 8 true
chan out 9 true
chan out 0 false
chan is close
chan done

```
