- [Context 分类](#context-分类)
  - [Context](#context)
  - [emptyCtx](#emptyctx)
  - [cancelCtx](#cancelctx)
  - [timerCtx](#timerctx)
  - [valueCtx](#valuectx)
- [Context 使用场景](#context-使用场景)
  - [http-控制超时取消](#http-控制超时取消)
  - [传递上下文信息](#传递上下文信息)
  - [业务层面-控制协程关闭](#业务层面-控制协程关闭)

# Context 分类

## Context

```go
type Context interface {
	// 返回过期时间
	Deadline() (deadline time.Time, ok bool)
	
	// 返回 Done channel 
	Done() <-chan struct{}

	// 返回 err
	Err() error
	
	// 返回对应 key 的 value
	Value(key any) any
}
```

## emptyCtx

仅作为 background

```go
// An emptyCtx is never canceled, has no values, and has no deadline. It is not
// struct{}, since vars of this type must have distinct addresses.
type emptyCtx int
```

## cancelCtx

```go
// A cancelCtx can be canceled. When canceled, it also cancels any children
// that implement canceler.
type cancelCtx struct {
	// 父 Context
	Context
	
	// 保护共享资源的并发读写
	mu       sync.Mutex            // protects following fields
	done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```

```go
func (c *cancelCtx) Done() <-chan struct{} {
	d := c.done.Load()
	if d != nil {
		return d.(chan struct{})
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	d = c.done.Load()
	if d == nil {
		d = make(chan struct{})
		c.done.Store(d)
	}
	return d.(chan struct{})
}
```

- 基于 atomic 包，读取 cancelCtx 中的 chan；倘若已存在，则直接返回；
- 加锁后，在此检查 chan 是否存在，若存在则返回；（double check）
- 初始化 chan 存储到 aotmic.Value 当中，并返回.（懒加载机制）

```go
func (c *cancelCtx) Value(key any) any {
	if key == &cancelCtxKey {
		return c
	}
	return value(c.Context, key)
}

case *cancelCtx:
    if key == &cancelCtxKey {
        return c
    }
    c = ctx.Context
```

- 如果 key 特定值 &cancelCtxKey，则返回 cancelCtx 自身的指针。
- 否则返回上层 context。

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```

- 构建一个子 cancelCtx。将 parent Context 的 cancel 事件传播给子 context。
- 返回生成的 cancelCtx 和对应的 cancelFunc。

```go
// propagateCancel arranges for child to be canceled when parent is.
func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
	if done == nil {
		return // parent is never canceled
	}

	select {
	case <-done:
		// parent is already canceled
		child.cancel(false, parent.Err())
		return
	default:
	}

	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
			// parent has already been canceled
			child.cancel(false, p.err)
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
		atomic.AddInt32(&goroutines, +1)
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}
```

- 如果 parent 没有 done，直接返回。
- 如果 parent 已经终止，则终止子 context，但无需 removeFromParent。
- 如果 parent 是 cancelCtx，如果 parent 有 err，则说明 parent 已经终止，则终止子 context，但无需 removeFromParent。如果 parent 没 err，则加锁，并将子 context 添加到 parent 的 children map 当中。
- 如果 parent 不是 cancelCtx，则启动一个协程，通过多路复用的方式监控 parent 状态，倘若其终止，则同时终止子 context，并透传 parent 的 err。

```go
// cancel closes c.done, cancels each of c's children, and, if
// removeFromParent is true, removes c from its parent's children.
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	d, _ := c.done.Load().(chan struct{})
	if d == nil {
		c.done.Store(closedchan)
	} else {
		close(d)
	}
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```

- 如果 context 已经被 cancel，则直接返回。
- 如果 context 内没有 done channel，则存入一个 done channel，否则直接 close done channel。
- 对于其所有子 context，依次将其 context 进行 cancel。
- 判断是否需要从其父 context 的 children 中删除自身。

## timerCtx

```go
// A timerCtx carries a timer and a deadline. It embeds a cancelCtx to
// implement Done and Err. It implements cancel by stopping its timer then
// delegating to cancelCtx.cancel.
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}
```

```go
//  WithTimeout 和 WithDeadline 的意义相同
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}

func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(false, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```

- 校验 parent 的过期时间是否早于自己，若是，则构造一个 cancelCtx 返回即可。
- 通过 propagateCancel 方法将 cancel 事件传递给子 context。
- 判断过期时间是否已到，若是，直接 cancel timerCtx，并返回 DeadlineExceeded 的错误。
- 启动 time.Timer，时间到期后 cancel 自己。
- 返回 timerCtx，已经一个封装了 cancel 逻辑的闭包 cancel 函数。

## valueCtx

```go
// A valueCtx carries a key-value pair. It implements Value for that key and
// delegates all other calls to the embedded Context.
type valueCtx struct {
	Context
	key, val any
}
```

```go
func WithValue(parent Context, key, val any) Context {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
```

- 构建含有 kv 的 valueCtx。

```go
func (c *valueCtx) Value(key any) any {
	if c.key == key {
		return c.val
	}
	return value(c.Context, key)
}

func value(c Context, key any) any {
	for {
		switch ctx := c.(type) {
		case *valueCtx:
			if key == ctx.key {
				return ctx.val
			}
			c = ctx.Context
		case *cancelCtx:
			if key == &cancelCtxKey {
				return c
			}
			c = ctx.Context
		case *timerCtx:
			if key == &cancelCtxKey {
				return &ctx.cancelCtx
			}
			c = ctx.Context
		case *emptyCtx:
			return nil
		default:
			return c.Value(key)
		}
	}
}
```

- 根据 cancelCtx、timerCtx、emptyCtx 作出不同的处理。 

# Context 使用场景

## http-控制超时取消

func (c *Client) do(req *Request) (retres *Response, reterr error)

1. 当创建一个带有取消的Context，会把Context的内部类中的err变量赋值为CancelErr；
2. 客户端的调用cancelFunc，会向context的Done所绑定的channel写入值；
3. 当channel写入值后，transport.go中的readLoop方法会监听这个channel的写入，从而把context取消的err传给persistConn，并关闭连接；
4. 关闭连接后，数据读取便会遇到连接关闭的网络错误错误，当遇到这个错误，在bodySignal中进行错误处理，这里并不感知连接的关闭，只利用fn分别错误类型，当错误为io.EOF，直接将这个错误置为nil，若不是，便通过bodySignal获取到连接中的错误，再返回这个错误；
5. 最后通过body.read()方法将错误打印出来。
6. 这里复杂在于，每个角色只做自己的工作，遇到错误不是直接返回，而是等待其他角色来读取错误；具体表现为：context负责生成错误消息、传递取消指令给persistConn；persistConn基于bodySignal建立读取数据和连接的关联，响应Context的取消并关闭连接，拿到context的错误信息；client读取数据和错误；bodySignal：分析错误，并传递数据和persistConn的错误消息给client。

原文链接：https://blog.csdn.net/qq_40859492/article/details/120773564

## 传递上下文信息

ctx的生命周期是 伴随请求开始而诞生、请求结束而终止的。在请求中ctx会跨越多个函数多个协程，在打日志时，第一个参数预留给ctx是因为日志库需要从Context中抽取trace ID等信息，从而记录下完整的日志。获取信息时只需要调用 context 的 Value 方法。

若我们的系统也需要请求第三方服务，同样应把trace ID等信息放入HTTP Header后发送请求，其他服务按照同样的流程接收到trace ID后开始内部逻辑处理。这样一个请求在多个系统中就通过trace ID串联起了整个流程。除trace ID外，Context还可以传递 URL Path、请求时间、Caller等信息。

## 业务层面-控制协程关闭

主要用途是预防系统做不必要的工作。

比如用户请求A接口时，A接口内部需要请求A database、B cache 、C System获取各种数据，把这些数据经过计算后组装到一起返回给调用方。

用户取消了该操作，又或是其中某个环节出现了错误，就可以调用 cancelFunc 来让给其他所有协程停止运行。

```go
func getUserInfoBySystemA(ctx context.Context) error {
  time.Sleep(100 * time.Millisecond)
  // 模拟请求出错的情况
  return errors.New("failed")
}

func getOrderInfoBySystemB(ctx context.Context) {
  select {
  case <-time.After(500 * time.Millisecond):
    fmt.Println("process finished")
  case <-ctx.Done():
    fmt.Println("process cancelled")
  }
}

func main() {
  ctx, cancel := context.WithCancel(context.Background())

  //并发从两个服务中获取相关数据
  go func() {
    err := getUserInfoBySystemA(ctx)
    if err != nil {
      // 发生错误，调用cancelFunc
      cancel()
    }
  }()
  getOrderInfoBySystemB(ctx)
}
```