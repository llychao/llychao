# 浅析 go context

```
1. Context的用法、有无父子关系？怎么去做并发控制？底层实现！

2. 为什么需要 Context?
```

### Context类型：

1. emptyCtx(用于默认Context):   
    目前有两个实例化的ctx: background和TODO，background作为整个运行时的默认ctx，而TODO主要用来临时填充未确定具体Context类型的ctx参数
2. cancelCtx(带cancel的Context):    
    最核心的，是WithCancel的底层实现，且可包含多个cancelCtx子节点，从而构成一棵树。
3. timerCtx(计时并带cancel的Context)、
4. valueCtx(携带kv键值对)，

多种类型可以以父子节点形式相互组合其功能形成新的Context。

### cancelCtx的cancel有几种方式

1. 主动调用cancel
2. 其父ctx被cancel，触发子ctx的cancel
3. time.Timer事件触发timerCtx的cancel回调

当一个ctx被cancel后，ctx内部的负责通知的channel被关闭，从而触发select此channel的goroutine获得通知，完成相应逻辑的处理


### 几种主要Context的实现
```go

type Context interface {
  // 只用于timerCtx，即WithDeadline和WithTimeout
  Deadline() (deadline time.Time, ok bool)
  // 需要获取通知的goroutine可以select此chan，当此ctx被cancel时，会close此chan
  Done() <-chan struct{}
  // 错误信息
  Err() error
  // 只用于valueCtx
  Value(key interface{}) interface{}
}


// cancelCtx
type cancelCtx struct {
  Context
  mu       sync.Mutex            
  done     chan struct{}         
  // 主要用于存储子cancelCtx和timerCtx
  // 当此ctx被cancel时，会自动cancel其所有children中的ctx
  children map[canceler]struct{} 
  err      error                 
}
// timeCtx
type timerCtx struct {
  cancelCtx
  // 借助计时器触发timeout事件
  timer *time.Timer
  deadline time.Time
}
// valueCtx 
type valueCtx struct {
  Context
  key, val interface{}
}

// cancel逻辑
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
  /* ... */
  c.err = err
  // 如果在第一次调用Done之前就调用cancel，则done为nil
  if c.done == nil {
    c.done = closedchan
  } else {
    close(c.done)
  }
  for child := range c.children {
    // NOTE: acquiring the child's lock while holding parent's lock.
    // 不能将子ctx从当前移除，由于移除需要拿当前ctx的锁
    child.cancel(false, err)
  }
  // 直接置为nil让gc处理子ctx的回收?
  c.children = nil
  c.mu.Unlock()

  // 把自己从parent里移除，注意这里需要拿parent的锁
  if removeFromParent {
    removeChild(c.Context, c)
  }
}

```


### 外部接口
```go
// Background
func Background() Context {
  // 直接返回默认的顶层ctx
  return background
}

// WithCancel
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
  // 实例化cancelCtx
  c := newCancelCtx(parent)
  // 如果parent是cancelCtx类型，则注册到parent.children，否则启用
  // 新的goroutine专门负责此ctx的cancel，当parent被cancel后，自动
  // 回调child的cancel
  propagateCancel(parent, &c)
  return &c, func() { c.cancel(true, Canceled) }
}

// WithDeadline
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc) {
  // 如果parent是deadline，且比当前早，则直接返回cancelCtx
  if cur, ok := parent.Deadline(); ok && cur.Before(deadline) {
    return WithCancel(parent)
  }
  c := &timerCtx{
    cancelCtx: newCancelCtx(parent),
    deadline:  deadline,
  }
  propagateCancel(parent, c)
  d := time.Until(deadline)
  // 已经过了
  if d <= 0 {
    c.cancel(true, DeadlineExceeded) // deadline has already passed
    return c, func() { c.cancel(true, Canceled) }
  }
  c.mu.Lock()
  defer c.mu.Unlock()
  if c.err == nil {
    // time.Timer到时则自动回调cancel
    c.timer = time.AfterFunc(d, func() {
      c.cancel(true, DeadlineExceeded)
    })
  }
  return c, func() { c.cancel(true, Canceled) }
}

// WithTimeout
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
  // 直接使用WithDeadline的实现即可
  return WithDeadline(parent, time.Now().Add(timeout))
}
```


### Go标准库Context使用demo: 
    https://www.liwenzhou.com/posts/Go/go_context/

### 怎么并发控制

控制并发主要包括两种方式：一种是WaitGroup，另外一种是Context。

WaitGroup是一种控制多个goroutine并发执行的方式
```go
var wg sync.WaitGroup

func service1()  {
    time.Sleep(2*time.Second)
    fmt.Println("service1 done")
    wg.Done()
}

func service2()  {
    time.Sleep(2*time.Second)
    fmt.Println("service2 done")
    wg.Done()
}

func main() {
    wg.Add(2)
    go service1()
    go service2()
    wg.Wait()
    fmt.Println("all done")
}
```

上面的例子是协程内自己处理结束后调用wg.Done退出，实际使用中我们可能需要从外部去结束一个协程。不然它会一直跑，就泄漏了。

如何从外部去结束一个goroutine，很容易想到的一个方法就是定义一个全局变量，然后再外部修改这个变量的值，goroutine不断的轮训这个变量是否改变。

这种方式也可以，但是首先我们要保证这个变量在多线程下的安全，基于此，有一种更好的方式：channel + select

```go
func testChannel() {
    stop := make(chan bool)

    go func() {
        for {
            select {
            case <-stop:
                fmt.Println("goroutine done")
                return
            default:
                fmt.Println("goroutine is running")
                time.Sleep(2 * time.Second)
            }
        }
    }()

    time.Sleep(10 * time.Second)
    fmt.Println("cancel goroutine")
    stop<- true
    //为了检测监控过是否停止，如果没有监控输出，就表示停止了
    time.Sleep(5 * time.Second)
}
```

这种方式也有局限性，如果有很多goroutine都需要控制结束怎么办呢？如果这些goroutine又衍生了其他更多的goroutine怎么办呢？如果一层层的无穷尽的goroutine呢？这就非常复杂了，即使我们定义很多chan也很难解决这个问题，因为goroutine的关系链就导致了这种场景非常复杂。

context可以很好的解决上面的问题。下面用context的方式改写上面的例子。

```go
func testContext() {
    ctx, cancel := context.WithCancel(context.Background())
    go func(ctx context.Context) {
        for {
            select {
            case <-ctx.Done():
                fmt.Println("goroutine done")
                return
            default:
                fmt.Println("goroutine is running")
                time.Sleep(2 * time.Second)
            }
        }
    }(ctx)

    time.Sleep(10 * time.Second)
    fmt.Println("cancel goroutine")
    cancel()
    //为了检测监控过是否停止，如果没有监控输出，就表示停止了
    time.Sleep(5 * time.Second)
}
```

context.Background() 返回一个空的Context，这个空的Context一般用于整个Context树的根节点。然后我们使用context.WithCancel(parent)函数，创建一个可取消的子Context，然后当作参数传给goroutine使用，这样就可以使用这个子Context跟踪这个goroutine。

在goroutine中，使用select调用<-ctx.Done()判断是否要结束，如果接受到值的话，就可以返回结束goroutine了；如果接收不到，就会继续进行监控。

那么是如何发送结束指令的呢？这就是示例中的cancel函数啦，它是我们调用context.WithCancel(parent)函数生成子Context的时候返回的，第二个返回值就是这个取消函数，它是CancelFunc类型的。我们调用它就可以发出取消指令，然后我们的监控goroutine就会收到信号，就会返回结束。

Context控制多个goroutine

```go
func testMultiContext() {
    ctx, cancel := context.WithCancel(context.Background())
    go watch(ctx, "watch1")
    go watch(ctx, "watch2")

    time.Sleep(10 * time.Second)
    fmt.Println("cancel all goroutine")
    cancel()
    //为了检测监控过是否停止，如果没有监控输出，就表示停止了
    time.Sleep(5 * time.Second)
}

func watch(ctx context.Context, name string)  {
    for {
        select {
        case <-ctx.Done():
            fmt.Println(name," is done")
            return
        default:
            fmt.Println(name," is running")
            time.Sleep(2 * time.Second)
        }
    }
}
```

### Context 使用原则
- 不要把Context放在结构体中，要以参数的方式传递
- 以Context作为参数的函数方法，应该把Context作为第一个参数，放在第一位。
- 给一个函数方法传递Context的时候，不要传递nil，如果不知道传递什么，就使用context.TODO
- Context的Value相关方法应该传递必须的数据，不要什么数据都使用这个传递
- Context是线程安全的，可以放心的在多个goroutine中传递








