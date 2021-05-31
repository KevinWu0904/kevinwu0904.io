---
title: Golang Context源码剖析
toc: true
authors:
  - kevinwu
tags:
  - golang
date: '2021-04-20'
lastmod: '2021-04-20'
draft: false
---

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-common/golang-logo.png)

Context是Go语言标准库的组成之一，在Goroutine之间传播，能够提供Cancel和KV功能。

## Why Context
### 问题一：Goroutine Cancellation
Goroutine基于[CSP](http://www.usingcsp.com/cspbook.pdf)理论模型，是Go语言的最小并发单元和任务执行的载体。由于Goroutine的生命周期是互相独立的，即使是由父Goroutine创建出子Goroutine的场景也不例外，因此我们**不能**在一个Goroutine运行过程中去杀死另一个Goroutine。

但某些情况下，我们需要停止一个Goroutine的运行。例如：调用一个可能耗时较长的API，一定时间内如果不能返回结果就返回Timeout错误。这种时候需要依赖channel来完成这样的通信：
![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-context/goroutine-channel.png)

完整的例子：（通过timeout channel来通知退出信号，同时在另一个Goroutine中监听这个退出信号）
```go
func main() {
	timeout := make(chan struct{})
	time.AfterFunc(time.Second, func() {
		timeout <- struct{}{}
	})
	if err := call(timeout); err != nil {
		fmt.Println(err)
	}
}

func call(timeout <-chan struct{}) error {
	finish := make(chan struct{})

	go func() {
		// TODO
		finish <- struct{}{}
	}()

	select {
	case <-timeout:
		return errors.New("timeout")
	case <-finish:
		return nil
	}
}
```

因此，**Context的第一大功能：封装Cancellation，让Goroutine退出更加优雅**。
```go
func main() {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	if err := call(ctx); err != nil {
		fmt.Println(err)
	}
}

func call(ctx context.Context) error {
	finish := make(chan struct{})

	go func() {
		time.Sleep(time.Second * 10)
		finish <- struct{}{}
	}()

	select {
	case <-ctx.Done():
		return ctx.Err()
	case <-finish:
		return nil
	}
}
```

进一步的，由于Context本身会串联起一棵Context树，因此对于某个ctx子节点的cancel操作能够cancel它的整棵子树：
![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-context/context-cancellation.png)

### 问题二：Goroutine Local Storage
所谓的**Goroutine Local Storage**是指Goroutine级别的本地存储，不同Goroutine之间互不可见。通俗点说，GLS可以想象成是一个`map[string]interface{}`，其中map的key是Goroutine ID。

官方十分不推荐开发者实现GLS，引用官方的说法：
> Please don't use goroutine local storage. It's highly discouraged. In fact, IIRC, we used to expose Goid, but it is hidden since we don't want people to do this.
> 
> Potential problems include:
> 
> 1. when goroutine goes away, its goroutine local storage won't be GCed. (you can get goid for the current goroutine, but you can't get a list of all running goroutines)
> 2. what if handler spawns goroutine itself? the new goroutine suddenly loses access to your goroutine local storage. You can guarantee that your own code won't spawn other goroutines, but in general you can't make sure the standard library or any 3rd party code won't do that.
> 
> thread local storage is invented to help reuse bad/legacy code that assumes global state, Go doesn't have legacy code like that, and you really should design your code so that state is passed explicitly and not as global (e.g. resort to goroutine local storage)

翻译过来大意如下：

1. 当Goroutine结束后，它对应的GLS无法被GC。因为我们虽然能获取当前运行时的GoID，但是却无法获取到所有运行中的GoID，所以无法确定需要GC的是哪些GoID。
2. 当前Goroutine若生成新Goroutine，则新Goroutine会失去当前Goroutine的GLS控制权。而且即便开发者能确保自身代码不生成新Goroutine，却无法保证标准库和第三方库中不这么做。（这一条本质上是指Goroutine的GLS在父子Goroutine之间无法共享）

不仅如此，Go官方还刻意不提供获取GoID标准库方法，就是为了防止开发者自己实现GLS。（注：不过由于在runtime.Stack中能够截取出GoID，因此如果非要的话，GLS仍然是可以实现的）

虽然Go不建议我们使用全局的GLS，但在某些情况下，我们仍需要类似的功能。例如：HTTP Server模型中，每一个Request Goroutine都需要保存一定的上下文信息，并且该信息需要保证并发安全且只在当前Goroutine及其子Goroutine之间传播。

因此，**Context的第二大功能：在单个Goroutine或父子Goroutine中传递KV**。

完整例子：
```go
func main() {
	ctx := context.WithValue(context.Background(), "key", "value")

	go call1(ctx)
	go call2(ctx)

	time.Sleep(time.Second)
}

func call1(ctx context.Context) {
	fmt.Println(ctx.Value("key"))
}

func call2(ctx context.Context) {
	fmt.Println(ctx.Value("key"))
}
```

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-context/context-kv.png)

## Context源码剖析
context包的源码加注释一共不超过600行（Go1.16版本），因此阅读起来并没有太多心智负担。context包的主要实现包括：1个接口和4个实现。

### 1个接口
```go
type Context interface {
  Deadline() (deadline time.Time, ok bool)
  Done() <-chan struct{}
  Err() error
  Value(key interface{}) interface{}
}
```

1. Deadline返回Context的死亡时间戳，如果无限长则ok返回false
2. Done返回一个channel，常配合select关键字，用于控制超时退出
3. Err返回一个error，当Context Done之后，返回非空error解释原因
4. Value可用于查询指定key的值

### 4个实现
context包中一共有4种实现，它们分别是：

1. emptyCtx
2. valueCtx
3. cancelCtx
4. timerCtx

这4个实现包含继承关系：
![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-context/context-inheritance.png)

#### emptyCtx
emptyCtx通常作为root context，它不包含任何KV，且永不过期，emptyCtx的实现非常简单：
```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}
```

context标准库对外提供了2个常用的方法：
```go
var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

// Background returns a non-nil, empty Context. It is never canceled, has no
// values, and has no deadline. It is typically used by the main function,
// initialization, and tests, and as the top-level Context for incoming
// requests.
func Background() Context {
	return background
}

// TODO returns a non-nil, empty Context. Code should use context.TODO when
// it's unclear which Context to use or it is not yet available (because the
// surrounding function has not yet been extended to accept a Context
// parameter).
func TODO() Context {
	return todo
}
```

其中`context.Background()`通常用于最顶层Context产生的位置，`context.TODO()`则用于暂时还不确定应该传递何种Context类型的位置。

#### valueCtx
valueCtx能够为当前ctx附加一个KV信息。这里非常容易产生误解的点是：valueCtx仅仅能够附加一个KV，而不是一个map键值对。

因此valueCtx在检索某个key信息时，本质上是对当前节点及其父节点进行向上遍历，最终取出整条路径上面包含有该key节点的value。
```go
// WithValue returns a copy of parent in which the value associated with key is
// val.
//
// Use context Values only for request-scoped data that transits processes and
// APIs, not for passing optional parameters to functions.
//
// The provided key must be comparable and should not be of type
// string or any other built-in type to avoid collisions between
// packages using context. Users of WithValue should define their own
// types for keys. To avoid allocating when assigning to an
// interface{}, context keys often have concrete type
// struct{}. Alternatively, exported context key variables' static
// type should be a pointer or interface.
func WithValue(parent Context, key, val interface{}) Context {
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

// A valueCtx carries a key-value pair. It implements Value for that key and
// delegates all other calls to the embedded Context.
type valueCtx struct {
	Context              // 保存父Context
	key, val interface{} // 仅附加一个KV信息
}

func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key { // 找到这个key则返回对应val
		return c.val
	}
	return c.Context.Value(key) // 递归寻找父Context是否包含这个key
}
```
![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-context/context-find-kv.png)

#### cancelCtx
cancelCtx是context标准库中的核心功能，它被广泛用于Goroutine的退出。context包中提供了常用方法`context.WithCancel()`：
```go
// WithCancel returns a copy of parent with a new Done channel. The returned
// context's Done channel is closed when the returned cancel function is called
// or when the parent context's Done channel is closed, whichever happens first.
//
// Canceling this context releases resources associated with it, so code should
// call cancel as soon as the operations running in this Context complete.
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	c := newCancelCtx(parent) // 新建cancelCtx
	propagateCancel(parent, &c) // 父子ctx建立关联关系
	return &c, func() { c.cancel(true, Canceled) }
}
```
该方法返回子ctx和一个名为cancel的func，cancel不需要任何参数就能执行，它的作用在于控制该ctx进入Done状态。**任何进入Done状态的ctx，ctx.Done()方法将会得到channel通道的消息**。

cancelCtx定义如下：
```go
// A cancelCtx can be canceled. When canceled, it also cancels any children
// that implement canceler.
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```
可以看到：在cancelCtx中定义了children map，存放了所有子cancelCtx的信息。**这是cancel之所以能够实现父子ctx同步cancel的核心点**！另外一点值得注意的是，为了防止并发问题，cancelCtx定义了一个sync.Mutex。

propagateCancel方法是第一个需要重点关注的实现：
```go
// propagateCancel arranges for child to be canceled when parent is.
func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
	if done == nil {
		return // parent is never canceled // 父ctx不具备cancel能力，则无需建立父子关联
	}

  // 判断父ctx是否已经进入Done状态
	select {
	case <-done:
		// parent is already canceled
		child.cancel(false, parent.Err())
		return
	default:
	}

	if p, ok := parentCancelCtx(parent); ok { // parentCancelCtx方法需要重点关注
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
    
    // 当父ctx非cancelCtx标准实现的情况下，需要同时监听父子ctx的Done()信号
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

propagateCancel将会调用parentCancelCtx，这是第二个需要重点关注的实现：
```go
// parentCancelCtx returns the underlying *cancelCtx for parent.
// It does this by looking up parent.Value(&cancelCtxKey) to find
// the innermost enclosing *cancelCtx and then checking whether
// parent.Done() matches that *cancelCtx. (If not, the *cancelCtx
// has been wrapped in a custom implementation providing a
// different done channel, in which case we should not bypass it.)
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
	done := parent.Done()
	if done == closedchan || done == nil {
		return nil, false
	}
	p, ok := parent.Value(&cancelCtxKey).(*cancelCtx) // 通过一个特殊的cancelCtxKey向上递归寻找父路径中最近的cancelCtx
	if !ok {
		return nil, false
	}
	p.mu.Lock()
	ok = p.done == done // 如果父ctx.Done和父路径中最近的cancelCtx的Done不是同一个值，则表示父ctx非cancelCtx标准实现
	p.mu.Unlock()
	if !ok {
		return nil, false
	}
	return p, true
}
```

由此可见，cancelCtx在创建出来之后，会分为两种情况：

1. 父ctx是cancelCtx，通过children map完成父子关联。
2. 父ctx不是cancelCtx但实现了Done()方法，通过新goroutine select完成父子关联。

无论哪种方式，当完成了父子关联之后，使用方只需要通过cancel func就能够完成退出功能：

1. 父ctx是cancelCtx，依次循环children，逐个child cancel即可。
2. 父ctx不是cancelCtx但实现了Done()方法，只需要close(c.done)即可。

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
 
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done) // 非标准cancelCtx，可以简单close当前的done channcel
	}
  
  // 标准cancelCtx，可以循环每个child，依次cancel
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

总的来说，cancelCtx的实现极为精巧，同时又逻辑清晰，建议读者对源码细细阅读，学习其设计思想。

#### timerCtx
timerCtx继承自cancelCtx，它的主干实现都是基于cancelCtx来完成的。timerCtx在cancelCtx的基础上面，附加了timer和deadline，用于在截止时间调用cancel func：
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

context包提供了2个常用方法：

1. context.WithDeadline
2. context.WithTimeout

其中context.WithTimeout是直接复用了context.WithDeadline实现，源码如下：
```go
// WithTimeout returns WithDeadline(parent, time.Now().Add(timeout)).
//
// Canceling this context releases resources associated with it, so code should
// call cancel as soon as the operations running in this Context complete:
//
// 	func slowOperationWithTimeout(ctx context.Context) (Result, error) {
// 		ctx, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
// 		defer cancel()  // releases resources if slowOperation completes before timeout elapses
// 		return slowOperation(ctx)
// 	}
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
  return WithDeadline(parent, time.Now().Add(timeout)) // 直接调用WithDeadline，以time.Now().Add(timeout)计算出Deadline
}
```

而context.WithDeadline则基本复用了cancelCtx的逻辑，只是在此基础上新增了time.AfterFunc逻辑，源码如下：
```go
// WithDeadline returns a copy of the parent context with the deadline adjusted
// to be no later than d. If the parent's deadline is already earlier than d,
// WithDeadline(parent, d) is semantically equivalent to parent. The returned
// context's Done channel is closed when the deadline expires, when the returned
// cancel function is called, or when the parent context's Done channel is
// closed, whichever happens first.
//
// Canceling this context releases resources associated with it, so code should
// call cancel as soon as the operations running in this Context complete.
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
		c.timer = time.AfterFunc(dur, func() { // 主要多出time.AfterFunc相关逻辑
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```

总的来说，timerCtx是cancelCtx的衍生，在cancelCtx的基础上增加了一个时间任务，并在到期时候自动调用cancel。

## Context最佳实践

* 最佳实践原则一：不传递nil，不确定使用什么类型的情况下，传递context.TODO()。
* 最佳实践原则二：使用函数参数传递而不是结构体，占函数第一个参数位置，并命名为ctx。
* 最佳实现原则三：只在Context中附加与请求有关的元数据，不依赖于Context来传递可选参数。
* 最佳实现原则四：Context可以传递到不同的Goroutine，并发安全。

## 参考
1. [https://blog.golang.org/context](https://blog.golang.org/context)
2. [https://faiface.github.io/post/context-should-go-away-go2/](https://faiface.github.io/post/context-should-go-away-go2/)

## 总结
本文介绍了Go语言Context的功能，并通过源码剖析详细介绍context包中的1个接口和4个实现。Context整体设计颇为精巧，但也极具侵入性，往往整个工程最终都需要ctx参数。但目前为止，Context仍然作为Go语言官方推荐的最佳实践之一，作为开发者的我们也需要明确这个事实。