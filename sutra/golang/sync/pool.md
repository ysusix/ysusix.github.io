## sync.pool

sync.Pool 是 Go 1.3加入的新特性

这个类设计的目的是用来保存和复用临时对象，以减少内存分配，降低CG压力。

### 1、pool 使用姿势

例子来自官网，[pool](https://golang.org/pkg/sync/#Pool)

```
package main

import (
	"bytes"
	"io"
	"os"
	"sync"
	"time"
)

var bufPool = sync.Pool{
	New: func() interface{} {
		// The Pool's New function should generally only return pointer
		// types, since a pointer can be put into the return interface
		// value without an allocation:
		return new(bytes.Buffer)
	},
}

// timeNow is a fake version of time.Now for tests.
func timeNow() time.Time {
	return time.Unix(1136214245, 0)
}

func Log(w io.Writer, key, val string) {
	b := bufPool.Get().(*bytes.Buffer)
	b.Reset()
	// Replace this with time.Now() in a real logger.
	b.WriteString(timeNow().UTC().Format(time.RFC3339))
	b.WriteByte(' ')
	b.WriteString(key)
	b.WriteByte('=')
	b.WriteString(val)
	w.Write(b.Bytes())
	bufPool.Put(b)
}

func main() {
	Log(os.Stdout, "path", "/search?q=flowers")
}
```



```
type Pool  
    func (p *Pool) Get() interface{}  
    func (p *Pool) Put(x interface{})  
    New  func() interface{}
```

1、Get 返回 Pool 中的任意一个对象。

如果 Pool 为空，则调用 New 返回一个新创建的对象。

如果 new  func不存在，返回nil

2、Put 将对象存入 Pool

Pool 池中的对象，生命周期介于两次gc之间。

放进 Pool 中的对象，会在说不准什么时候被回收掉。

### 2、pool 源码

### 2.1 pool 结构

```
type Pool struct {
   noCopy noCopy

   local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
   localSize uintptr        // size of the local array

   victim     unsafe.Pointer // local from previous cycle
   victimSize uintptr        // size of victims array

   // New optionally specifies a function to generate
   // a value when Get would otherwise return nil.
   // It may not be changed concurrently with calls to Get.
   New func() interface{}
}
// Local per-P Pool appendix.
type poolLocalInternal struct {
    private interface{} // Can be used only by the respective P.
    shared  poolChain   // Local P can pushHead/popHead; any P can popTail.
}

type poolLocal struct {
    poolLocalInternal

    // Prevents false sharing on widespread platforms with
    // 128 mod (cache line size) = 0 .
    pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}
```

可以看到其实结构并不复杂，注意几个细节。

- local这里面真正的是 [P]poolLocal 其中P就是GPM模型中的P，有多少个P数组就有多大，也就是每个P维护了一个本地的poolLocal。
- poolLocal里面维护了一个private一个shared，看名字其实就很明显了，private是给自己用的，而shared的是一个队列，可以给别人用的。注释写的也很清楚，自己可以从队列的头部存然后从头部取，而别的P可以从尾部取。
- victim这个从字面上面也可以知道，幸存者嘛，当进行gc的stw时候，会将local中的对象移到victim中去，也就是说幸存了一次gc

### 2.2 Get

```go
func (p *Pool) Get() interface{} {
    ......
    l, pid := p.pin()
    x := l.private
    l.private = nil
    if x == nil {
        x, _ = l.shared.popHead()
        if x == nil {
            x = p.getSlow(pid)
        }
    }
    runtime_procUnpin()
    ......
    if x == nil && p.New != nil {
        x = p.New()
    }
    return x
}

func (p *Pool) getSlow(pid int) interface{} {
    // See the comment in pin regarding ordering of the loads.
    size := atomic.LoadUintptr(&p.localSize) // load-acquire
    locals := p.local                        // load-consume
    // Try to steal one element from other procs.
    for i := 0; i < int(size); i++ {
        l := indexLocal(locals, (pid+i+1)%int(size))
        if x, _ := l.shared.popTail(); x != nil {
            return x
        }
    }

    // Try the victim cache. We do this after attempting to steal
    // from all primary caches because we want objects in the
    // victim cache to age out if at all possible.
    size = atomic.LoadUintptr(&p.victimSize)
    if uintptr(pid) >= size {
        return nil
    }
    locals = p.victim
    l := indexLocal(locals, pid)
    if x := l.private; x != nil {
        l.private = nil
        return x
    }
    for i := 0; i < int(size); i++ {
        l := indexLocal(locals, (pid+i)%int(size))
        if x, _ := l.shared.popTail(); x != nil {
            return x
        }
    }

    // Mark the victim cache as empty for future gets don't bother
    // with it.
    atomic.StoreUintptr(&p.victimSize, 0)

    return nil
}
```

我去掉了其中一些竞态分析的代码，Get的逻辑其实非常清晰。

- 如果 private 不是空的，那就直接拿来用
- 如果 private 是空的，那就先去本地的shared队列里面从头 pop 一个
- 如果本地的 shared 也没有了，那 getSlow 去拿，其实就是去别的P的 shared 里面偷，偷不到回去 victim 幸存者里面找
- 如果最后都没有，那就只能调用 New 方法创建一个了

我随手画了一下，可能不是特别准确，意思到位了



![img](https:////upload-images.jianshu.io/upload_images/10213518-c5f7694be2ded30a.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1132)

### 2.3 Put

```go
// Put adds x to the pool.
func (p *Pool) Put(x interface{}) {
    if x == nil {
        return
    }
    ......
    l, _ := p.pin()
    if l.private == nil {
        l.private = x
        x = nil
    }
    if x != nil {
        l.shared.pushHead(x)
    }
    runtime_procUnpin()
    ......
}
```

看完Get其实Put就很简单了

- 如果 private 没有，就放在 private
- 如果 private 有了，那么就放到 shared 队列的头部

### 2.4  一些细节

```
func init() {
	runtime_registerPoolCleanup(poolCleanup)
}

func poolCleanup() {
   for _, p := range oldPools {
      p.victim = nil
      p.victimSize = 0
   }
   for _, p := range allPools {
      p.victim = p.local
      p.victimSize = p.localSize
      p.local = nil
      p.localSize = 0
   }
   oldPools, allPools = allPools, nil
}

// src/runtime/mgc.go
//func sync_runtime_registerPoolCleanup(f func()) {
//	poolcleanup = f
//}
//func clearpools() {
//    if poolcleanup != nil {
//      poolcleanup()
//    }
//}
```

```
var (                                                                             
	allPoolsMu Mutex                                                              
                                                                                  
	// allPools is the set of pools that have non-empty primary                   
	// caches. Protected by either 1) allPoolsMu and pinning or 2)                
	// STW.                                                                       
	allPools []*Pool                                                              
                                                                                  
	// oldPools is the set of pools that may have non-empty victim                
	// caches. Protected by STW.                                                  
	oldPools []*Pool                                                              
)                                                                                 
```

可以看到 pool 包在 init 的时候注册了一个 poolCleanup 函数，它会清除所有的 pool 里面的所有缓存的对象，该函数注册进去之后会在每次 gc 之前都会调用，因此 sync.Pool 缓存的期限只是两次 gc 之间这段时间。

 gc 的时候会清掉缓存对象，因此不用担心 pool 会无限增大，pool是一个自动扩容缩容的资源池，缓存对象数量是没有限制的（只受限于内存）

gc 的时候会将victim 清空，将 local 池中的数据转移到 victim 池中，get的时候也会有一步尝试从 victim中获取。

这个优化可以保证不因为gc导致重复对象池过期后大量申请内存空间。而当确实不需要这么多空间时，接下来的gc还是可以清理掉 victim 池。

victim 是go  1.13增加的特性。

可以看到，Pool里面有全局变量持有了所有的Pool, 然后也有一个全局锁来保护数据域的可靠性。

###3、 原文链接

[golang中神奇的sync.Pool](https://www.jianshu.com/p/8fbbf6c012b2)

