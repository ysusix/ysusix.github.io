## context 源码学习

####1、基础 context

    	type Context interface {
        	Deadline() (deadline time.Time, ok bool)
          
          Done() <-chan struct{}
    
          Err() error
    
          Value(key interface{}) interface{}
    }
    
Deadline方法是获取设置的截止时间的意思，第一个返回式是截止时间，到了这个时间点，Context会自动发起取消请求；第二个返回值ok==false时表示没有设置截止时间，如果需要取消的话，需要调用取消函数进行取消。

Done方法返回一个只读的chan，类型为struct{}，我们在goroutine中，如果该方法返回的chan可以读取，则意味着parent context已经发起了取消请求，我们通过Done方法收到这个信号后，就应该做清理操作，然后退出goroutine，释放资源。

Err方法返回取消的错误原因，因为什么Context被取消。

Value方法获取该Context上绑定的值，是一个键值对，所以要通过一个Key才可以获取对应的值，这个值一般是线程安全的。

### 2内置 实现的context

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

func (e *emptyCtx) String() string {
	switch e {
	case background:
		return "context.Background"
	case todo:
		return "context.TODO"
	}
	return "unknown empty Context"
}

var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

func Background() Context {
	return background
}

func TODO() Context {
	return todo
}
```

`background` 和`todo` 本质上都是`emptyCtx`结构体类型，是一个不可取消，没有设置截止时间，没有携带任何值的Context。

### 3、衍生context

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithValue(parent Context, key, val interface{}) Context
```

`WithCancel`函数，传递一个父Context作为参数，返回子Context，以及一个取消函数用来取消Context。 `WithDeadline`函数，和`WithCancel`差不多，它会多传递一个截止时间参数，意味着到了这个时间点，会自动取消Context，当然我们也可以不等到这个时候，可以提前通过取消函数进行取消。

`WithTimeout`实际上是对`WithDeadline`的封装,见代码块

```
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
   return WithDeadline(parent, time.Now().Add(timeout))
}
```

可以使用`context.WithValue`方法附加一对K-V的键值对，这里Key必须是等价性的，也就是具有可比性；Value值要是线程安全的。

### 4、源码学习

#### 4.1 cancelCtx

```
type cancelCtx struct {
    Context
    mu    sync.Mutex      // protects following fields
    done   chan struct{}     // created lazily, closed by first cancel call
    children map[canceler]struct{} // set to nil by the first cancel call
    err   error         // set to non-nil by the first cancel call
}
```

`cancel()`接口实现

`cancel()`内部方法是理解`cancelCtx`的最关键的方法，其作用是关闭自己和其后代，其后代存储在`cancelCtx.children`的map中，其中key值即后代对象，value值并没有意义，这里使用map只是为了方便查询而已。

```
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
  c.mu.Lock()
    
  c.err = err              //设置一个error，说明关闭原因
  close(c.done)           //将channel关闭，以此通知派生的context
    
  for child := range c.children {  //遍历所有children，逐个调用cancel方法
   	child.cancel(false, err)
  }
  c.children = nil
  c.mu.Unlock()
  if removeFromParent {      //正常情况下，需要将自己从parent删除
    removeChild(c.Context, c)
  }
}
```

实际上，WithCancel()返回的第二个用于cancel context的方法正是此cancel()。

WithCancel()方法作了三件事：

- 初始化一个cancelCtx实例
- 将cancelCtx实例添加到其父节点的children中(如果父节点也可以被cancel的话)
- 返回cancelCtx实例和cancel()方法

其实现源码如下所示：

```
`func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {``    ``c := newCancelCtx(parent)``    ``propagateCancel(parent, &c)  //将自身添加到父节点``    ``return &c, func() { c.cancel(true, Canceled) }``}`
```

`propagateCancel`实现源码

```
func propagateCancel(parent Context, child canceler) {
   if parent.Done() == nil {
      return // parent is never canceled
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

这里将自身添加到父节点的过程有必要简单说明一下：

- 如果父节点也支持cancel，也就是说其父节点肯定有children成员，那么把新context添加到children里即可；
- 如果父节点不支持cancel，就继续向上查询，直到找到一个支持cancel的节点，把新context添加到children里；
- 如果所有的父节点均不支持cancel，则启动一个协程等待父节点结束，然后再把当前context结束。



#### 4.2  timerCtx

```
type timerCtx struct {
    cancelCtx
    timer *time.Timer // Under cancelCtx.mu.
    deadline time.Time
}
```

 timerCtx在cancelCtx基础上增加了deadline用于标示自动cancel的最终时间，而timer就是一个触发自动cancel的定时器。

由此，衍生出WithDeadline()和WithTimeout()。实现上这两种类型实现原理一样，只不过使用语境不一样：

- deadline: 指定最后期限，比如context将2018.10.20 00:00:00之时自动结束
- timeout: 指定最长存活时间，比如context将在30s后结束。

对于接口来说，timerCtx在cancelCtx基础上还需要实现Deadline()和cancel()方法，其中cancel()方法是重写的。

Deadline()方法仅是返回timerCtx.deadline。而timerCtx.deadline是WithDeadline()或WithTimeout()方法设置的。

#### 4.3 **valueCtx**

```
type valueCtx struct {
    Context
    key, val interface{}
}
```

valueCtx只是在Context基础上增加了一个key-value对，用于在各级协程间传递一些数据。

由于valueCtx既不需要cancel，也不需要deadline，那么只需要实现Value()接口即可。

Value（）接口实现

由valueCtx数据结构定义可见，valueCtx.key和valueCtx.val分别代表其key和value值。 实现也很简单：

```
`func (c *valueCtx) Value(key interface{}) interface{} {``    ``if c.key == key {``        ``return c.val``    ``}``    ``return c.Context.Value(key)``}`
```

这里有个细节需要关注一下，即当前context查找不到key时，会向父节点查找，如果查询不到则最终返回interface{}。也就是说，可以通过子context查询到父的value值。

WithValue（）方法实现

WithValue()实现也是非常的简单,：

```
`func WithValue(parent Context, key, val interface{}) Context {``    ``if key == nil {``        ``panic("nil key")``    ``}``    ``return &valueCtx{parent, key, val}``}`
```



Context仅仅是一个接口定义，跟据实现的不同，可以衍生出不同的context类型；

cancelCtx实现了Context接口，通过WithCancel()创建cancelCtx实例；

timerCtx实现了Context接口，通过WithDeadline()和WithTimeout()创建timerCtx实例；

valueCtx实现了Context接口，通过WithValue()创建valueCtx实例；

三种context实例可互为父节点，从而可以组合成不同的应用形式；

### 5、源码修改

路径 `/usr/local/Cellar/go/1.13.1/libexec/src/context/context_test.go`

func `XTestParentFinishesChild`

源代码

```
check(parent, "parent")
check(cancelChild, "cancelChild")
check(valueChild, "valueChild")
check(timerChild, "timerChild")

// WithCancel should return a canceled context on a canceled parent.
precanceledChild := WithValue(parent, "key", "value")

select {
case <-precanceledChild.Done():
default:
   t.Errorf("<-precanceledChild.Done() blocked, but shouldn't have")
}
if e := precanceledChild.Err(); e != Canceled {
   t.Errorf("precanceledChild.Err() == %v want %v", e, Canceled)
}
```

改进

```
// WithCancel should return a canceled context on a canceled parent.

precanceledChild := WithValue(parent, "key", "value")
check(parent, "parent")
check(cancelChild, "cancelChild")
check(valueChild, "valueChild")
check(timerChild, "timerChild")
check(precanceledChild, "precanceledChild")
```

#### 6 tantan context

问题 1 为什么 异步调用 grpc ，要衍生一份 context？

grpc 会检查 context.cancel ，如果取消就会终止。



#### 7 总结

context主要用于父子任务之间的同步取消信号，本质上是一种协程调度的方式。另外在使用context时有两点值得注意：

1、上游任务仅仅使用context通知下游任务不再需要，但不会直接干涉和中断下游任务的执行，由下游任务自行决定后续的处理操作，也就是说context的取消操作是无侵入的

2、context 是线程安全的，因为context本身是不可变的（immutable），因此可以放心地在多个协程中传递使用。