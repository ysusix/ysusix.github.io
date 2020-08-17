# sync.map

### 1、sync.Map 与  map区别

map不是线程安全的，并发读写的时候会报错

多线程读写map的时候，为了保证并发安全，只能加锁，有一定性能损耗

sync.Map 是官方提供的并发安全版本的map，在 Go 1.9 引入

sync.Map  的读取，插入，删除也都保持着常数级的时间复杂度。

sync.Map  适合读多写少的场景。对于写多的场景，会导致 read map 缓存失效，需要加锁，导致冲突变多；而且由于未命中 read map 次数过多，导致 dirty map 提升为 read map，这是一个 O(N) 的操作，会进一步降低性能。

| 实现方式    | 原理                                                         | 适用场景                                                     |
| ----------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| map+Mutex   | 通过Mutex互斥锁来实现多个goroutine对map的串行化访问          | 读写都需要通过Mutex加锁和释放锁，适用于读写比接近的场景      |
| map+RWMutex | 通过RWMutex来实现对map的读写进行读写锁分离加锁，从而实现读的并发性能提高 | 同Mutex相比适用于读多写少的场景                              |
| sync.Map    | 底层通分离读写map和原子指令来实现读的近似无锁，并通过延迟更新的方式来保证读的无锁化 | 读多修改少，元素增加删除频率不高的情况，在大多数情况下替代上述两种实现 |

### 2、sync.Map 使用姿势

```golang
package main

import (
	"fmt"
	"sync"
)

func main()  {
	var m sync.Map
	// 1. 写入/更新
	m.Store("qcrao", 18)
	m.Store("qcrao", 19)
	m.Store("stefno", 20)

	// 2. 读取
	age, _ := m.Load("qcrao")
	fmt.Println(age.(int))

	// 3. 遍历
	m.Range(func(key, value interface{}) bool {
		name := key.(string)
		age := value.(int)
		fmt.Println(name, age)
		return true
	})

	// 4. 删除
	m.Delete("qcrao")
	age, ok := m.Load("qcrao")
	fmt.Println(age, ok)

	// 5. 读不到则写入
	m.LoadOrStore("stefno", 100)
	age, _ = m.Load("stefno")
	fmt.Println(age)
}
```

第 1 步，写入两个 k-v 对；

第 2 步，使用 Load 方法读取其中的一个 key；

第 3 步，遍历所有的 k-v 对，并打印出来；

第 4 步，删除其中的一个 key，再读这个 key，得到的就是 nil；

第 5 步，使用 LoadOrStore，尝试读取或写入 "Stefno"，因为这个 key 已经存在，因此写入不成功，并且读出原值。

程序输出：

```shell
18
stefno 20
qcrao 18
<nil> false
20
```

### 3、源码探究

先白话文说下大概逻辑。让下文看的更快。（大概只有是这样流程就好）
**写：直写。读：先读read，没有再读dirty。**

![sync.map 读写](/Users/baijichuan/Desktop/E94163B5-0437-472E-A544-220A52CD7BAC.png)

![image.png](https://img2018.cnblogs.com/blog/1506724/201912/1506724-20191230011542279-1556421363.png)

### 3.1 基础结构

sync.Map的核心数据结构:

```
type Map struct {
	mu Mutex
	read atomic.Value // readOnly
	dirty map[interface{}]*entry
	misses int
}
```

| 说明   | 类型                   | 作用                                                         |
| ------ | ---------------------- | ------------------------------------------------------------ |
| mu     | Mutex                  | 加锁作用。保护后文的dirty字段                                |
| read   | atomic.Value           | 存读的数据。因为是atomic.Value类型，只读，所以并发是安全的。实际存的是readOnly的数据结构。 |
| misses | int                    | 计数器。每次从read中读失败，则计数+1。                       |
| dirty  | map[interface{}]*entry | 包含最新写入的数据。当misses计数达到一定值，将其赋值给read。 |

这里有必要简单描述一下，大概的逻辑，

readOnly的数据结构：

```
type readOnly struct {
    m  map[interface{}]*entry
    amended bool 
}
```

| 说明    | 类型                   | 作用                                                   |
| ------- | ---------------------- | ------------------------------------------------------ |
| m       | map[interface{}]*entry | 单纯的map结构                                          |
| amended | bool                   | Map.dirty的数据和这里的 m 中的数据不一样的时候，为true |

entry的数据结构：

```
type entry struct {
    //可见value是个指针类型，虽然read和dirty存在冗余情况（amended=false），但是由于是指针类型，存储的空间应该不是问题
    p unsafe.Pointer // *interface{}
}
```

这个结构体主要是想说明。虽然前文read和dirty存在冗余的情况，但是由于value都是指针类型，其实存储的空间其实没增加多少。

### 3.2 查询

```
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
    // 因read只读，线程安全，优先读取
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    
    // 如果read没有，并且dirty有新数据，那么去dirty中查找
    if !ok && read.amended {
        m.mu.Lock()
        // 双重检查（原因是前文的if判断和加锁非原子的，害怕这中间发生故事）
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        
        // 如果read中还是不存在，并且dirty中有新数据
        if !ok && read.amended {
            e, ok = m.dirty[key]
            // m计数+1
            m.missLocked()
        }
        
        m.mu.Unlock()
    }
    
    if !ok {
        return nil, false
    }
    return e.load()
}

func (m *Map) missLocked() {
    m.misses++
    if m.misses < len(m.dirty) {
        return
    }
    
    // 将dirty置给read，因为穿透概率太大了(原子操作，耗时很小)
    m.read.Store(readOnly{m: m.dirty})
    m.dirty = nil
    m.misses = 0
}
```

流程图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190720235756624.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE5NTc3NTg=,size_16,color_FFFFFF,t_70)
这边有几个点需要强调一下：

> 如何设置阀值？

这里采用**miss计数和dirty长度**的比较，来进行阀值的设定。

> 为什么dirty可以直接换到read？

因为写操作只会操作dirty，所以保证了dirty是最新的，并且数据集是肯定包含read的。
（可能有同学疑问，dirty不是下一步就置为nil了，为何还包含？后文会有解释。）

> 为什么dirty置为nil？

我不确定这个原因。猜测：一方面是当read完全等于dirty的时候，读的话read没有就是没有了，即使穿透也是一样的结果，所以存的没啥用。另一方是当存的时候，如果元素比较多，影响插入效率。

### 3.3 删

```
func (m *Map) Delete(key interface{}) {
    // 读出read，断言为readOnly类型
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    // 如果read中没有，并且dirty中有新元素，那么就去dirty中去找。这里用到了amended，当read与dirty不同时为true，说明dirty中有read没有的数据。
    
    if !ok && read.amended {
        m.mu.Lock()
        // 再检查一次，因为前文的判断和锁不是原子操作，防止期间发生了变化。
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        
        if !ok && read.amended {
            // 直接删除
            delete(m.dirty, key)
        }
        m.mu.Unlock()
    }
    
    if ok {
    // 如果read中存在该key，则将该value 赋值nil（采用标记的方式删除！）
        e.delete()
    }
}

func (e *entry) delete() (hadValue bool) {
    for {
    	// 再次再一把数据的指针
        p := atomic.LoadPointer(&e.p)
        if p == nil || p == expunged {
            return false
        }
        
        // 原子操作
        if atomic.CompareAndSwapPointer(&e.p, p, nil) {
            return true
        }
    }
}
```

流程图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190720235535762.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE5NTc3NTg=,size_16,color_FFFFFF,t_70)

这边有几个点需要强调一下：

> 1.为什么dirty是直接删除，而read是标记删除？

read的作用是在dirty前头优先度，遇到相同元素的时候为了不穿透到dirty，所以采用标记的方式。
同时正是因为这样的机制+amended的标记，可以保证read找不到&&amended=false的时候，dirty中肯定找不到

> 2.为什么dirty是可以直接删除，而没有先进行读取存在后删除？

删除成本低。读一次需要寻找，删除也需要寻找，无需重复操作。

> 3.如何进行标记的？

将值置为nil。（这个很关键）

重新给 dirty赋值的时候，会将 nil改为 expunged，并且不会写入到dirty中，这样在进行dirty到 read 替换的时候，自然删除

如果在此期间对应的 key被重新赋值，要同步写入到 dirty中，避免丢失数据

### 3.4 增(改)

```
func (m *Map) Store(key, value interface{}) {
    // 如果m.read存在这个key，并且没有被标记删除，则尝试更新。
    read, _ := m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }
    
    // 如果read不存在或者已经被标记删除
    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)
   
    if e, ok := read.m[key]; ok { // read 存在该key
    // 如果entry被标记expunge，则表明dirty没有key，可添加入dirty，并更新entry。
        if e.unexpungeLocked() { 
            // 加入dirty中，这儿是指针
            m.dirty[key] = e
        }
        // 更新value值
        
         e.storeLocked(&value)
    } else if e, ok := m.dirty[key]; ok { // dirty 存在该key，更新
        e.storeLocked(&value)
        
    } else { // read 和 dirty都没有
        // 如果read与dirty相同，则触发一次dirty刷新（因为当read重置的时候，dirty已置为nil了）
        if !read.amended { 
            // 将read中未删除的数据加入到dirty中
            m.dirtyLocked() 
            // amended标记为read与dirty不相同，因为后面即将加入新数据。
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        m.dirty[key] = newEntry(value) 
    }
    m.mu.Unlock()
}

// 将read中未删除的数据加入到dirty中
func (m *Map) dirtyLocked() {
    if m.dirty != nil {
        return
    }
    
    read, _ := m.read.Load().(readOnly)
    m.dirty = make(map[interface{}]*entry, len(read.m))
    
    // 遍历read。
    for k, e := range read.m {
        // 通过此次操作，dirty中的元素都是未被删除的，可见标记为expunged的元素不在dirty中！！！
        if !e.tryExpungeLocked() {
            m.dirty[k] = e
        }
    }
}

// 判断entry是否被标记删除，并且将标记为nil的entry更新标记为expunge
func (e *entry) tryExpungeLocked() (isExpunged bool) {
    p := atomic.LoadPointer(&e.p)
    
    for p == nil {
        // 将已经删除标记为nil的数据标记为expunged
        if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
            return true
        }
        p = atomic.LoadPointer(&e.p)
    }
    return p == expunged
}

// 对entry尝试更新 （原子cas操作）
func (e *entry) tryStore(i *interface{}) bool {
    p := atomic.LoadPointer(&e.p)
    if p == expunged {
        return false
    }
    for {
        if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
            return true
        }
        p = atomic.LoadPointer(&e.p)
        if p == expunged {
            return false
        }
    }
}

// read里 将标记为expunge的更新为nil
func (e *entry) unexpungeLocked() (wasExpunged bool) {
    return atomic.CompareAndSwapPointer(&e.p, expunged, nil)
}

// 更新entry
func (e *entry) storeLocked(i *interface{}) {
    atomic.StorePointer(&e.p, unsafe.Pointer(i))
}
```

流程图：
![增加/更新](https://img2018.cnblogs.com/blog/1506724/201912/1506724-20191230011542808-74052706.png)
这边有几个点需要强调一下：

> 1. read中的标记为已删除的区别？

标记为nil，说明是正常的delete操作，此时dirty中不一定存在
a. dirty赋值给read后，此时dirty不存在
b. dirty初始化后，肯定存在

标记为expunged，说明是在dirty初始化的时候操作的，此时dirty中肯定不存在。

> 1. 可能存在性能问题？

初始化dirty的时候，虽然都是指针赋值，但read如果较大的话，可能会有些影响。

### 3.5 range

因为for ... range map是内建的语言特性，所以没有办法使用for range遍历sync.Map, 但是可以使用它的Range方法，通过回调的方式遍历。

```
func (m *Map) Range(f func(key, value interface{}) bool) {
    read, _ := m.read.Load().(readOnly)
    // 如果m.dirty中有新数据，则提升m.dirty,然后在遍历
    if read.amended {
        //提升m.dirty
        m.mu.Lock()
        read, _ = m.read.Load().(readOnly) //双检查
        if read.amended {
            read = readOnly{m: m.dirty}
            m.read.Store(read)
            m.dirty = nil
            m.misses = 0
        }
        m.mu.Unlock()
    }
    // 遍历, for range是安全的
    for k, e := range read.m {
        v, ok := e.load()
        if !ok {
            continue
        }
        if !f(k, v) {
            break
        }
    }
}
```

Range方法调用前可能会做一个m.dirty的提升，不过提升m.dirty不是一个耗时的操作。

### 4、sync.Map 总结

1、核心思想是用空间换时间，用两个map来存储数据，`read`和`dirty`，`read`支持原子操作，可以看作是`dirty` 的cache，`dirty`是更底层的数据存储层

2、此种map替换逻辑，使 sync.map具有了缩容功能。

3、虽然`read`和`dirty`有冗余，但这些map的value数据是通过指针指向同一个数据，所以尽管实际的value会很大，但是冗余的空间占用还是有限的。

4、如果对map的读操作远远多于写操作（写操作包括新增和删除key），那么sync.Map是很合适，能够大大提升性能

### 5、思维扩散

还有优化空间么？
如何优化？
[concurrent-map](https://github.com/orcaman/concurrent-map)


### 6、参考文档

[1.golang 标准库 sync.Map 中 nil 和 expunge 区别](https://www.cnblogs.com/DillGao/p/11118842.html)

[2.图解Go里面的sync.Map了解编程语言核心实现源码](https://www.cnblogs.com/buyicoding/p/12117370.html)

[3.Go 1.9 sync.Map揭秘](https://segmentfault.com/a/1190000010294041)

[4.由浅入深聊聊Golang的sync.Map](https://blog.csdn.net/u011957758/article/details/96633984)