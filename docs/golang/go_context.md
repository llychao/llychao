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



