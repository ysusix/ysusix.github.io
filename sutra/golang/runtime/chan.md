## channel 介绍及源码解析

channel是Golang提供的goroutine间的通信方式，其为Golang并发模型CSP的关键，Golang鼓励用通讯实现数据共享，如果需要跨进程通信，建议使用分布式方案或者消息队列来解决。该文章主要介绍，以下内容:

- channel介绍及范例
- channel用法
- channel使用场景
- channel原理赏析

下面在进入正题之前，简要介绍一下CSP模型：

> 传统并发模型分为Actor模型与CSP模型，其中CSP全称为Communicating Sequential Processess，CSP模型有并发执行体(进程、线程、协程)，和消息通道组成，执行体之间通过消息通道进行通讯，CSP模型关注消息发送的载体，即消息管道，而Actor关注的是内部的状态，那么Golang中执行体对应的是goroutine，消息通道对应的是channel。

#### 一、channel介绍及范例

> 如上所言，channel 提供了一种通信机制，其为gouroutine之间的通信提供了一种可能，执行体拷贝数据，channel负责传递，有以下应用场景:

- 广播，如消费者/生产者模型
- 交换数据
- 并发控制
- 显示通知等

Golang鼓励使用通讯来实现数据共享，而不是经由内存。

##### 1.1 channel特性

1）线程安全：hchan mutex

2）先进先出：copying into and out of hchan buffer

3）channel的高性能所在：

- 调用runtime scheduler实现，OS thread不需要阻塞；
- 跨goroutine栈可以直接进行读写；

##### 1.2 channel类型

channel分为非缓存channel与缓存channel。

- 无缓存channel

  从无缓存的channel中读取消息会堵塞，直到有goroutine往channel中发送消息；同理，向无缓存的channel中发送消息也会堵塞，直到有goroutine从channel中读取消息。

- 有缓存channel

  有缓存channel的声明方式为指定make函数的第二个参数，该参数为channel缓存的容量。
  通过内置len函数可获取chan元素个数，通过cap函数可获取chan的缓存长度

**单项channel** :单向channel为只读/只写channel，单向channel，在编译时，可进行检测。

```
func testSingal(ch chan<- int) <- chan int {
  // 定义工作逻辑
}
```

其中 chan<- int表示只写channel, <-chan int表示只读channel，此类函数/方法声明可防止channel滥用，在编译时可以检测出。

##### 1.3 channel创建

channel使用内置的make函数创建，如下，声明类型为int的channel：

```
// 非缓存channel
ch := make(chan int)
// 缓存channel
bch := make(chan int, 2)
```

channel和map类似，make创建了底层数据结构的引用，当赋值或参数传递时，只是拷贝了一个channel的引用，其指向同一channel对象，与其引用类型一样，channel的空值也为nil。使用`==`可以对类型相同的channel进行比较，只有指向相同对象或同为nil时，结果为true。

##### 1.4 channel的读写操作

channel在使用前，需要初始化，否则永远阻塞。

```
ch := make(chan int)

// 写入channel
ch <- x

// 从channel中读取
y <- ch

// 从channel中读取
z,ok := <- ch
if ok {

}
```

##### 1.5 channel的关闭

golang提供了内置的close函数，对channel进行关闭操作。

```
// 初始化channel
ch := make(chan int)
// 关闭channel ch
close(ch)
```

关于channel的关闭，需要注意以下事项：

- 关闭未初始化的channel(nil)会panic
- 重复关闭同一channel会panic
- 向已关闭channel发送消息会panic
- 从已关闭channel读取数据，不会panic，若存在数据，则可以读出未被读取的消息，若已被读出，则获取的数据为零值，可以通过ok-idiom的方式，判断channel是否关闭
- channel的关闭操作，会产生广播消息，所有向channel读取消息的goroutine都会接受到消息

```
package main

import fmt

func main() {
    // 初始化channel
    ch := make(chan int, 3)
    // 发送消息
    ch <- 1
    ch <- 2
    # 关闭channel
    close(ch)
    // 循环读取
    for c := range ch {
      fmt.Println(c)
    }
}
```

##### 1.6 两类channel

```
ch := make(chan int, 1)
```

有缓存的channel使用环形数组实现，当缓存未满时，向channel发送消息不会阻塞，当缓存满时，发送操作会阻塞，直到其他goroutine从channel中读取消息；同理，当channel中消息不为空时，读取消息不会阻塞，当channel为空时，读取操作会阻塞，直至其他goroutine向channel发送消息。

```
ch := make(chan int)
// 阻塞，因channel ch为空
<- ch
ch := make(chan int, 3)
ch <- 1
ch <- 2
ch <- 3

# 阻塞，因缓存已满
ch <- 4
<- ch
```

------

#### 二、channel的用法

##### 2.1 goroutine通信

看下面《effctive go》中的例子：
主goroutine会阻塞，直至执行sort的goroutine完成

```
// 初始化chan
c := make(chan int)
# 使用goroutine执行list.Sort(),完毕后，发送信号
go func() {
  list.Sort()
  c <- 1
}()
// 处理其他事务
doSomething()
// 读取chan消息
<-c
```

##### 2.2 range遍历

channel也可以使用range取值，并且会一直从chanel中读取数据，直至goroutine关闭该channel，循环才会结束，如下所示。

```
// 初始化channel
ch := make(chan int, 5)
go func(){
  for i := 0; i < 5; i ++ {
    ch <- i
  }
}()

for i := range ch {
  fmt.Println(i)
}
```

等同于

```
// 初始化channel
ch := make(chan int, 5)
go func(){
  for i := 0; i < 5; i ++ {
    ch <- i
  }
}()
for {
  i, ok := <- ch
  if !ok {
    break
  }
  fmt.Println(i)
}
```

##### 2.3 配合select使用

select用法类似IO多路复用，可同时监听多个channel的消息，如下所示：

```
select {
    case <- a;
      fmt.Println("testa")
    case <- b;
      fmt.Println("testb")
    case c <- 3;
      fmt.Println("testc")
    default:
      fmt.Println("testdefault")
}
```

select有以下特性：

- select可同时监听多个channel的读/写
- 执行select时，若只有一个case通过，则执行该case
- 若有多个，则随机执行一个case
- 若所有都不满足，则执行default，若无default，则等待
- 可使用break跳出select

------

#### 三、channel使用场景

##### 3.1 设置超时时间

```
// 初始化channel，数据类型为struct{}
ch := make(chan struct{})
// 以goroutine方式处理func
go func(){
 // 处理逻辑
 // 传递ch，控制goroutine
}(ch)

timeout := time.After(1 * time.Sencond)
select {
  case <- ch:
      fmt.Printfln("任务完成.")
  case <- timeout:
      fmt.Printfln("时间已到.")
}
```

##### 3.2 控制channel

在某些应用场景，工作goroutine一直处理事务，直到收到退出信号

```
mch := make(chan struct{})
quit := make(chan struct{})
for {
  select {
    case <- mch:
        // 正常工作
        work()
    case <- quit:
        // 退出前，处理收尾工作
        doFinish()
        return
  }
}

```

#### 四、channel源码解析

##### 4.1 channel结构体

```

type hchan struct {
    qcount   uint            // 当前队列中剩余元素个数，即len
    dataqsiz uint            // 环形队列长度，即可以存放的元素个数，cap
    buf      unsafe.Pointer  // 环形队列指针：队列缓存，头指针，环形数组实现
    elemsize uint16          // 每个元素的大小
    closed   uint32          // 关闭标志位
    elemtype *_type          // 元素类型
    sendx    uint            // 队列下标，指示元素写入时存放到队列中的位置
    recvx    uint            // 队列下标，指示元素从队列的该位置读出
    recvq    waitq           // 等待读消息的goroutine队列
    sendq    waitq           // 等待写消息的goroutine队列

   // 持有该锁时不要更改另一个G的状态,特别是不要准备好一个G），因为这会因堆栈缩小而死锁。 
   //why?
   lock mutex // 该锁保护hchan所有字段
}
```

- v,ok:=<-chan，用来表述为能否从channel获取数据，如果能取到，ok为true(此时无法判断channel是否已经关闭)，如果取不到ok为false，同时channel已关闭或不存在
- c.sendq和c.recvq中的至少一个为空，除非使用单条goroutine阻塞无缓冲通道，使用select语句发送和接收， 否则，c.sendq和c.recvq的大小仅受select语句的大小限制。

```
c.qcount> 0 表示c.recvq为空。
qcount<c.dataqsiz 表示c.sendq为空。
```

```
func makechan(t *chantype, size int) *hchan{
	...
	switch {
	case mem == 0:
		// unbuffered
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}
}
```

```
// sending/receiving等待队列的链表实现
type waitq struct {
  first *sudog
  last  *sudog
}
```

**hchan类型**

一个channel只能传递一种类型的值，类型信息存储在`hchan`数据结构体中，`_type`结构体中包含`elemtype`及`elemsize`等。

- elemetype代表类型，用于数据传递过程中的赋值
- elemesize代码类型大小，用于在buf中定位元素位置

**hchan环形队列**
hchan内部实现了一个环形队列作为缓冲区，队列的长度是创建channel时指定的。下图展示了一个可缓存6个元素的channel的示意图：

![buf](https://s2.ax1x.com/2019/10/03/u0QEP1.png)

- dataqsiz指示队列长度为6，即可缓存6个元素
- buf指向队列的内存，队列中还剩余两个元素
- qcount表示队列中还有两个元素
- sendx指示后续写入的数据存储的位置，取值[0,6)
- recvx指示从该位置读取数据，取值[0,6)

**hchan等待队列**
从channel读消息，如果channel缓冲区为空或者没有缓存区，当前goroutine会被阻塞。
向channel读消息，如果channel缓冲区已满或者没有缓冲区，当前goroutine会被阻塞。

被阻塞的goroutine将会封装成sudog，加入到channel的等待队列中：

- 因读消息阻塞的goroutine会被channel向channel写入数据的goroutine唤醒
- 因写消息阻塞的goroutine会从channel读消息的goroutine唤醒

![waitq](https://s2.ax1x.com/2019/10/03/u0lo7Q.png)

一般情况下，recvq和sendq至少一个为空，只有一个例外，即同一个goroutine使用select语句向channel一边写数据，一个读数据。

##### 4.2 channel make实现

创建channel的过程实际上是初始化`hchan`结构，其中类型信息和缓存区长度有`makechan`传入，buf的大小则与元素大小和缓冲区长度共同决定，创建`hchan`的实现如下：

```
type chantype struct {
  typ  _type
  elem *_type
  dir  uintptr
}
func makechan(t *chantype, size int) *hchan {
  // 对于chantype暂时不做深究，只需了解上述结构体即可
  elem := t.elem

  // 数据项的大小是编译时检测，需要小于 64KB
  if elem.size >= 1<<16 {
     throw("makechan: invalid channel element type")
  }
  // maxAlign = 8
  if hchanSize%maxAlign != 0 || elem.align > maxAlign {
     throw("makechan: bad alignment")
  }
  // 缓存大小检测
  // 计算要分配的堆内存的大小，并返回是否溢出。
  mem, overflow := math.MulUintptr(elem.size, uintptr(size))
  // 其中maxAlloc在 runtime/malloc.go 217行
  if overflow || mem > maxAlloc-hchanSize || size < 0 {
    panic(plainError("makechan: size out of range"))
  }

  // Hchan does not contain pointers interesting for GC
  // when elements stored in buf do not contain pointers.
  // buf points into the same allocation, elemtype is persistent.
  // SudoG's are referenced from their owning thread so they can't be collected.
  // TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
  var c *hchan
  switch {
  case mem == 0:
    // Queue or element size is zero.
    // 队列或数据项大小为0
    // 其中mallocgc函数，后续在内存分配进行讲解
    c = (*hchan)(mallocgc(hchanSize, nil, true))
    // Race detector uses this location for synchronization.
    c.buf = c.raceaddr()
  case elem.ptrdata == 0:
    // Elements do not contain pointers.
    // Allocate hchan and buf in one call.
    c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
    c.buf = add(unsafe.Pointer(c), hchanSize)
  default:
    // Elements contain pointers.
    c = new(hchan)
    c.buf = mallocgc(mem, elem, true)
  }

  c.elemsize = uint16(elem.size)
  c.elemtype = elem
  c.dataqsiz = uint(size)

  if debugChan {
    print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
  }
  return c
}
```

hchan初始化过程，主要关注点就是switch的三类情况，若不含指针，那么buf与hchan为连续空间，当使用make区创建channel时，实际返回的时一个指向channel的指针，因此
可以在不同的functio之间直接传递channel对象，而不用通过指向channel的指针，如下图所示：

![make](https://s2.ax1x.com/2019/10/04/uDMTbD.png)

**缓存channel**
buffered channel底层数据模型如下：

![buffered](https://s2.ax1x.com/2019/10/04/uBYbJe.md.png)

当我们向channel里面写入消息时，会直接将消息写入buf，当环形队列buf存满后，会呈现下图状态：

![full](https://s2.ax1x.com/2019/10/04/uBYLzd.md.png)

当执行recvq.dequeue()时，如下图所示：

![recvq.dequeue](https://s2.ax1x.com/2019/10/04/uBYXQA.md.png)

##### 4.3 channel发送

向一个channel中发送数据的过程如下：

流程图如下：

![send](https://upload-images.jianshu.io/upload_images/7791606-25b892fa9e42339a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```

func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // 若hchan未初始化，且block为false，则永久阻塞
    if c == nil {
        if !block {
            return false
        }
        // gopark 与gounpark对应
        gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }

    // 检测通过再请求锁，比较锁很费时
    // 是否阻塞 && 未关闭 && （ch的数据项长度为0且接受队列为空）|| (ch的数据项长度大于0且此时队列已满)
    if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
        (c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
        return false
    }

    var t0 int64
    if blockprofilerate > 0 {
        t0 = cputicks()
    }

    lock(&c.lock)

    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }

    // A: 接受队列不为空，跳过缓存队列，直接send
    if sg := c.recvq.dequeue(); sg != nil {
        send(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true
    }

    // B: 接受队列未空，且channel缓存未满，则复制到缓存
    if c.qcount < c.dataqsiz {
        // Space is available in the channel buffer. Enqueue the element to send.
        qp := chanbuf(c, c.sendx)
        typedmemmove(c.elemtype, qp, ep)
        c.sendx++
        if c.sendx == c.dataqsiz {
            c.sendx = 0
        }
        c.qcount++
        unlock(&c.lock)
        return true
    }
    // 接受队列未空，且channel缓存已满
    if !block {
        unlock(&c.lock)
        return false
    }

    // C: 缓存已满，将goroutine加入到send队列
    // Block on the channel. Some receiver will complete our operation for us.
    // acquireSudog 该函数获取当前g
    gp := getg()
    // acquireSudog 该函数获取当前sudog
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    }
    // sudog相关赋值
    mysg.elem = ep
    mysg.waitlink = nil
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.waiting = mysg
    gp.param = nil
    // 将sudog加入到send队列
    c.sendq.enqueue(mysg)
    // 休眠
    goparkunlock(&c.lock, waitReasonChanSend, traceEvGoBlockSend, 3)
   	。。。
}
```

向channel写入数据主要流程如下：

- CASE1：当channel为空或者未初始化，如果block表示阻塞那么向其中发送数据将会永久阻塞；如果block表示非阻塞就会直接return
- CASE2：前置场景，block为非阻塞，且channel没有关闭(已关闭的channel不能写入数据)且(channel为非缓冲队列且receiver等待队列为空)或则( channel为有缓冲队列但是队列已满)，这个时候直接return
- 调用 lock(&c.lock) 锁住channel的全局锁
- CASE3：不能向已经关闭的channel send数据，会导致panic
- CASE4：如果channel上的recv队列非空，则跳过channel的缓存队列，直接向消息发送给接收的goroutine
  1. 调用sendDirect方法，将待写入的消息发送给接收的goroutine
  2. 释放channel的全局锁
  3. 调用goready函数，将接收消息的goroutine设置成就绪状态，等待调度
- CASE5：缓存队列未满，则将消息复制到缓存队列上，然后释放全局锁
- CASE6：缓存队列已满且接收消息队列recv为空，则将当前的goroutine加入到send队列
  - 获取当前goroutine的sudog，然后入channel的send队列
  - 将当前goroutine休眠



##### 4.4 channel接收

从一个channel读数据简单过程如下：

1. 没有缓冲区时，如果等待发送队列sendq不为空，直接从sendq中读取G，把G中数据读出，最后把G唤醒，结束读取过程
2. 有缓冲区时，如果等待发送队列sendq不为空，说明缓冲区已满，从缓冲区中头部读出消息，把G中数据写入缓冲区尾部，把G唤醒，结束读取过程
3. 如果缓冲区有数据，则从缓冲区读数据，结束读取过程
4. 将当前goroutine加入recvq，进入睡眠，等待背斜goroutine唤醒

简单流程图如下所示：

![recvq](https://s2.ax1x.com/2019/10/03/u0YekF.png)

```
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // nil channel接收消息，永久阻塞
    if c == nil {
        if !block {
            return
        }
        gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }
    if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
       c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
       atomic.Load(&c.closed) == 0 {
          return
    }

  var t0 int64
  if blockprofilerate > 0 {
    t0 = cputicks()
  }

  lock(&c.lock)
  // A: channel已经close且为空，则接收到的消息为空值
  if c.closed != 0 && c.qcount == 0 {
    if raceenabled {
      raceacquire(c.raceaddr())
    }
    unlock(&c.lock)
    if ep != nil {
      typedmemclr(c.elemtype, ep)
    }
    return true, false
  }
  // B: 发送队列不为空
  // 则直接从sender recv消息
  if sg := c.sendq.dequeue(); sg != nil {
    recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
    return true, true
  }
  // C: 缓存队列不为空，直接从队列取消息，移动头索引
  if c.qcount > 0 {
    // Receive directly from queue
    qp := chanbuf(c, c.recvx)
    if ep != nil {
      typedmemmove(c.elemtype, ep, qp)
    }
    typedmemclr(c.elemtype, qp)
    c.recvx++
    if c.recvx == c.dataqsiz {
      c.recvx = 0
    }
    c.qcount--
    unlock(&c.lock)
    return true, true
  }

  if !block {
    unlock(&c.lock)
    return false, false
  }
  // D: 缓存队列为空，将goroutine加入recv队列，并阻塞
  // no sender available: block on this channel.
  gp := getg()
  mysg := acquireSudog()
  mysg.releasetime = 0
  if t0 != 0 {
    mysg.releasetime = -1
  }
 
  mysg.elem = ep
  mysg.waitlink = nil
  gp.waiting = mysg
  mysg.g = gp
  mysg.isSelect = false
  mysg.c = c
  gp.param = nil
  c.recvq.enqueue(mysg)
  goparkunlock(&c.lock, waitReasonChanReceive, traceEvGoBlockRecv, 3)

  // someone woke us up
  if mysg != gp.waiting {
    throw("G waiting list is corrupted")
  }
  gp.waiting = nil
  if mysg.releasetime > 0 {
    blockevent(mysg.releasetime-t0, 2)
  }
  closed := gp.param == nil
  gp.param = nil
  mysg.c = nil
  releaseSudog(mysg)
  return true, !closed
}
```

接收channel的数据的流程如下：

- CASE1：前置channel为nil的场景：
  - 如果block为非阻塞，直接return；
  - 如果block为阻塞，就调用gopark()阻塞当前goroutine，并抛出异常。
- 前置场景，block为非阻塞，且channel为非缓冲队列且sender等待队列为空 或则 channel为有缓冲队列但是队列里面元素数量为0，且channel未关闭，这个时候直接return；
- 调用 lock(&c.lock) 锁住channel的全局锁；
- CASE2：channel已经被关闭且channel缓冲中没有数据了，这时直接返回success和空值；
- CASE3：sender队列非空，调用func recv(c *hchan, sg* sudog, ep unsafe.Pointer, unlockf func(), skip int) 函数处理：
  - channel是非缓冲channel，直接调用recvDirect函数直接从sender recv元素到ep对象，这样就只用复制一次；
  - 对于sender队列非空情况下， 有缓冲的channel的缓冲队列一定是满的：
    1. 先取channel缓冲队列的队头元素复制给receiver(也就是ep)
    2. 将sender队列的d队头元素里面的数据复制到channel缓冲队列刚刚弹出的元素的位置，这样缓冲队列就不用移动数据了
    3. 释放channel的全局锁
    4. 调用goready函数标记当前goroutine处于ready，可以运行的状态
- CASE4：sender队列为空，缓冲队列非空，直接取队列元素，移动头索引
- CASE5：sender队列为空、缓冲队列也没有元素且不阻塞协程，直接return (false,false)
- CASE6：sender队列为空且channel的缓存队列为空，将goroutine加入recv队列，并阻塞

##### 4.5 channel关闭

关闭channel时会把recvq中的G全部唤醒，本该写入G的数据位置为nil。把sendq中的G全部唤醒，但这些G会panic。
除此之外，panic出现的常见场景还有：

1. 关闭值为nil的channel
2. 关闭已经被关闭的channel
3. 向已经关闭的channel写数据

```
func closechan(c *hchan) {
  if c == nil {
    panic(plainError("close of nil channel"))
  }

  lock(&c.lock)
  // 关闭closed channel，panic
  if c.closed != 0 {
    unlock(&c.lock)
    panic(plainError("close of closed channel"))
  }

  if raceenabled {
    callerpc := getcallerpc()
    racewritepc(c.raceaddr(), callerpc, funcPC(closechan))
    racerelease(c.raceaddr())
  }
  // 设置关闭标志位
  c.closed = 1

  var glist gList

  // release all readers
  // 唤醒所有receiver
  for {
    sg := c.recvq.dequeue()
    if sg == nil {
      break
    }
    if sg.elem != nil {
      typedmemclr(c.elemtype, sg.elem)
      sg.elem = nil
    }
    if sg.releasetime != 0 {
      sg.releasetime = cputicks()
    }
    gp := sg.g
    gp.param = nil
    if raceenabled {
      raceacquireg(gp, c.raceaddr())
    }
    glist.push(gp)
  }

  // release all writers (they will panic)
  // 唤醒所有sender
  for {
    sg := c.sendq.dequeue()
    if sg == nil {
      break
    }
    sg.elem = nil
    if sg.releasetime != 0 {
      sg.releasetime = cputicks()
    }
    gp := sg.g
    gp.param = nil
    if raceenabled {
      raceacquireg(gp, c.raceaddr())
    }
    glist.push(gp)
  }
  unlock(&c.lock)

  // Ready all Gs now that we've dropped the channel lock.
  for !glist.empty() {
    gp := glist.pop()
    gp.schedlink = 0
    goready(gp, 3)
  }
}
```

关闭的主要流程如下所示：

- 获取全局锁；
- 设置channel数据结构chan的关闭标志位；
- 获取当前channel上面的读goroutine并链接成链表；
- 获取当前channel上面的写goroutine然后拼接到前面的读链表后面；
- 释放全局锁；
- 唤醒所有的读写goroutine。

##### 小结

下面对channel的逻辑，做一个整体分析：
不同goroutine在channel上面读写时，G1往channel中写入数据，G2从channel中读取数据，如下图所示：
![channel](https://s2.ax1x.com/2019/10/03/u0zM8I.md.png)

G1作用于底层`hchan`的流程如下：

![hchan](https://s2.ax1x.com/2019/10/04/uB5B9g.png)

1. 先获取全局锁
2. 然后enqueue元素(通过移动拷贝的方式)
3. 释放锁

G2读取时作用于底层数据结构流程如下图所示：

![hchan](https://s2.ax1x.com/2019/10/04/uBzUh9.png)

1. 先获取全局锁
2. 然后dequeue元素(通过移动拷贝的方式)
3. 释放锁

当channel写入3个数据之后，队列已满，这时候G1再写入时，G1会暂停等待receiver出现。

![hchan](https://s2.ax1x.com/2019/10/04/uBLZTA.png)

goroutine是Golang实现的用户空间的轻量级的线程，有runtime调度器调度，与操作系统的thread有多对一的关系，相关的数据结构如下图(goroutine原理，请关注后续章节):

![scheduler](https://s2.ax1x.com/2019/10/04/uDSMUe.png)

其中M是操作系统的线程，G是用户启动的goroutine，P是与调度相关的context，每个M都拥有一个P，P维护了一个能够运行的goutine队列，用于该线程执行。

当G1向buf已经满了的ch发送数据的时候，当runtime检测到对应的hchan的buf已经满了，会通知调度器，调度器会将G1的状态设置为waiting, 移除与线程M的联系，
然后从P的runqueue中选择一个goroutine在线程M中执行，此时G1就是阻塞状态，但是不是操作系统的线程阻塞，所以这个时候只用消耗少量的资源。调度器完整的处理逻辑如下所示：

![scheduler](https://s2.ax1x.com/2019/10/04/uDCHrq.png)

上图流程大致如下：

1. 当前goroutine（G1）会调用gopark函数，将当前协程置为waiting状态
2. 将M和G1绑定关系断开
3. heduler会调度另外一个就绪态的goroutine与M建立绑定关系，然后M 会运行另外一个G

所以整个过程中，OS thread会一直处于运行状态，不会因为协程G1的阻塞而阻塞。最后当前的G1的引用会存入channel的sender队列(队列元素是持有G1的sudog)。
那么blocked的G1怎么恢复呢？当有一个receiver接收channel数据的时候，会恢复 G1。

实际上hchan数据结构也存储了channel的sender和receiver的等待队列。数据原型如下：

![hchan](https://s2.ax1x.com/2019/10/04/uDiF6s.png)

等待队列里面是sudog的单链表，sudog持有一个G代表goroutine对象引用，elem代表channel里面保存的元素。当G1执行ch<-task4的时候，
G1会创建一个sudog然后保存进入sendq队列，实际上hchan结构如下图：

![hchan](https://s2.ax1x.com/2019/10/04/uDif3Q.png)

此时，**若G1进行一个读取channel操作**，变化如下图：

![hchan](https://s2.ax1x.com/2019/10/04/uDkuQJ.png)

整个过程如下所述：

1. G2调用 t:=<-ch 获取一个元素；
2. 从channel的buffer里面取出一个元素task1；
3. 从sender等待队列里面pop一个sudog；
4. 将task4复制buffer中task1的位置，然后更新buffer的sendx和recvx索引值；
5. 这时候需要将G1置为Runable状态，表示G1可以恢复运行；

此时将G1恢复到可运行状态需要scheduler的参与。G2会调用goready(G1)来唤醒G1。流程如下图所示：

![hchan](https://s2.ax1x.com/2019/10/04/uDAgN6.png)

1. 首先G2会调用goready(G1)，唤起scheduler的调度；
2. 将G1设置成Runable状态；
3. G1会加入到局部调度器P的local queue队列，等待运行。

###### 读取空channel

当channel的buffer里面为空时，这时候如果G2首先发起了读取操作。如下图：


![hchan](https://s2.ax1x.com/2019/10/04/uDeSDH.png)

会创建一个sudog，将代表G2的sudog存入recvq等待队列。然后G2会调用gopark函数进入等待状态，让出OS thread，然后G2进入阻塞态。
这个时候，如果有一个G1执行读取操作，最直观的流程就是：

1. 将recvq中的task存入buffer
2. goready(G2) 唤醒G2

但是我们有更加智能的方法：direct send; 其实也就是G1直接把数据写入到G2中的elem中，这样就不用走G2中的elem复制到buffer中，再从buffer复制给G1。具体过程就是G1直接把数据写入到G2的栈中。这样 G2 不需要去获取channel的全局锁和操作缓冲,如下图：

![hchan](https://s2.ax1x.com/2019/10/04/uDm9QU.png)



#### 5 发现

1、channel可以用redis替代么？

2、channel 实现有锁，如何保证的高性能

原文链接 [https://ustack.io/2019-10-04-Golang%E6%BC%AB%E8%B0%88%E4%B9%8Bchannel%E5%A6%99%E6%B3%95.html](https://ustack.io/2019-10-04-Golang漫谈之channel妙法.html)