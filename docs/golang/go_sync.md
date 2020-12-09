# 浅析 go sync包

### 背景介绍
尽管 Golang 推荐通过 channel 进行通信和同步，但在实际开发中 sync 包用得也非常的多

```go
var a = 0
// 启动 100 个协程，需要足够大
// var lock sync.Mutex
for i := 0; i < 100; i++ {
    go func(idx int) {
        // lock.Lock()
        // defer lock.Unlock()
        a += 1
        fmt.Printf("goroutine %d, a=%d\n", idx, a)
    }(i)
}
// 等待 1s 结束主程序
// 确保所有协程执行完
time.Sleep(time.Second)
```

### 互斥锁sync.Mutex，读写锁sync.RWMutex

锁的一些概念及使用方法,

整个包围绕 Locker 进行，这是一个 interface：

```go
type Locker interface {
        Lock()
        Unlock()
}
```

#### 互斥锁 Mutex

```go
func (m *Mutex) Lock()
func (m *Mutex) Unlock()
```

使用须知：
- 一个互斥锁只能同时被一个 goroutine 锁定，其它 goroutine 将阻塞直到互斥锁被解锁（重新争抢对互斥锁的锁定）

- 对一个未锁定的互斥锁解锁将会产生运行时错误。


#### 读写锁 RWMutex

```go
func (rw *RWMutex) Lock()       //写锁定
func (rw *RWMutex) Unlock()     //写解锁

func (rw *RWMutex) RLock()      //读锁定
func (rw *RWMutex) RUnlock()    //读解锁
```

使用须知：
- 当有一个 goroutine 获得写锁定，其它无论是读锁定还是写锁定都将阻塞直到写解锁；
- 当有一个 goroutine 获得读锁定，其它读锁定任然可以继续；
- 当有一个或任意多个读锁定，写锁定将等待所有读锁定解锁之后才能够进行写锁定。所以说这里的读锁定（RLock）目的其实是告诉写锁定：有很多人正在读取数据，你给我站一边去，等它们读（读解锁）完你再来写（写锁定）。


```go
var count int
var rw sync.RWMutex

func main() {
    ch := make(chan struct{}, 10)
    for i := 0; i < 5; i++ {
        go read(i, ch)
    }
    for i := 0; i < 5; i++ {
        go write(i, ch)
    }

    for i := 0; i < 10; i++ {
        <-ch
    }
}

func read(n int, ch chan struct{}) {
    rw.RLock()
    fmt.Printf("goroutine %d 进入读操作...\n", n)
    v := count
    fmt.Printf("goroutine %d 读取结束，值为：%d\n", n, v)
    rw.RUnlock()
    ch <- struct{}{}
}

func write(n int, ch chan struct{}) {
    rw.Lock()
    fmt.Printf("goroutine %d 进入写操作...\n", n)
    v := rand.Intn(1000)
    count = v
    fmt.Printf("goroutine %d 写入结束，新值为：%d\n", n, v)
    rw.Unlock()
    ch <- struct{}{}
}
```

### sync.Waitgroup,sync.Once

#### WaitGroup 

用于等待一组 goroutine 结束，用法很简单。它有三个方法：
```go
func (wg *WaitGroup) Add(delta int)
func (wg *WaitGroup) Done()
func (wg *WaitGroup) Wait()
```

**说明:** Add 用来添加 goroutine 的个数。Done 执行一次数量减 1。Wait 用来等待结束.

**注意:** wg.Add() 方法一定要在 goroutine 开始前执行哦。

```go
var wg sync.WaitGroup

for i, s := range seconds {
    // 计数加 1
    wg.Add(1)
    go func(i, s int) {
        // 计数减 1
        defer wg.Done()
        fmt.Printf("goroutine%d 结束\n", i)
    }(i, s)
}

// 等待执行结束
wg.Wait()
fmt.Println("所有 goroutine 执行结束")
```

#### Once

`func (o *Once) Do(f func())`

使用 sync.Once 对象可以使得函数多次调用只执行一次


```go
var once sync.Once
onceBody := func() {
    fmt.Println("Only once")
}
done := make(chan bool)
for i := 0; i < 10; i++ {
    go func() {
        once.Do(onceBody)
        done <- true
    }()
}
for i := 0; i < 10; i++ {
    <-done
}

----
# 打印结果
Only once
```

### sync.Map

sync.Map是一个并发版本的Go语言的map

- 使用Store(interface {}，interface {})添加元素。
- 使用Load(interface {}) interface {}检索元素。
- 使用Delete(interface {})删除元素。
- 使用LoadOrStore(interface {}，interface {}) (interface {}，bool)检索或添加之前不存在的元素。如果键之前在map中存在，则返回的布尔值为true。
- 使用Range遍历元素。

```go
var m sync.Map
// m:=&sync.Map{}

// 添加元素
m.Store(1, "one")
m.Store(2, "two")

// 迭代所有元素
m.Range(func(key, value interface{}) bool {
    fmt.Printf("%d: %s\n", key.(int), value.(string))
    return true
})

// 获取元素1
value, ok := m.Load(1)
fmt.Println(value,ok)	//one true

// 返回已存value，否则把指定的键值存储到map中
value, loaded := m.LoadOrStore(1, "three")
fmt.Println(value,loaded)	//one true

value1, loaded1 := m.LoadOrStore(3, "three")
fmt.Println(value1,loaded1)	//three false

m.Delete(3)
```

### sync.Pool




### sync.Cond