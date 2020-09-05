# 基本gc算法与golang gc算法

### 1 基本术语

 1、垃圾（Garbage）

就是需要回收的对象。

作为编写程序的人，是可以做出“这个对象已经不再需要了”这样的判断，但计算机是做不到的。因此，如果程序（通过某个变量等等）可能会直接或间接地引用一个对象，那么这个对象就被视为“存活”；与之相反，已经引用不到的对象被视为“死亡”。将这些“死亡”对象找出来，然后作为垃圾进行回收，这就是GC（**Garbage Collection**）的本质。

2、根（Root）

就是判断对象是否可被引用的起始点。

至于哪里才是根，不同的语言和编译器都有不同的规定，但基本上是将变量和运行栈空间作为根。好了，用上面这两个术语，我们来讲一讲主要的GC算法。



## 2 三大基础GC算法

###  1、标记清除法/标记压缩法

标记清除（Mark and Sweep）是最早开发出的GC算法（1960年）。它的原理非常简单，首先从根开始将可能被引用的对象用递归的方式进行标记，然后将没有标记到的对象作为垃圾进行回收。

![img](https://segmentfault.com/img/bVtJJC)



1）部分显示了随着程序的运行而分配出一些对象的状态，一个对象可以对其他的对象进行引用。

2）部分中，GC开始执行，从根开始对可能被引用的对象打上“标记”。大多数情况下，这种标记是通过对象内部的标志（Flag）来实现的。于是，被标记的对象我们把它们涂黑。

3）部分中，被标记的对象所能够引用的对象也被打上标记。重复这一步骤的话，就可以将从根开始可能被间接引用到的对象全部打上标记。到此为止的操作，称为标记阶段（Mark phase）。标记阶段完成时，被标记的对象就被视为“存活”对象。

4）部分中，将全部对象按顺序扫描一遍，将没有被标记的对象进行回收。这一操作被称为清除阶段（Sweep phase）。

在扫描的同时，还需要将存活对象的标记清除掉，以便为下一次GC操作做好准备。标记清除算法的处理时间，是和存活对象数与对象总数的总和相关的。

作为标记清除的变形，还有一种叫做标记压缩（Mark and Compact）的算法，它不是将被标记的对象清除，而是将它们不断压缩。

![img](https://img2018.cnblogs.com/blog/1452742/201910/1452742-20191015114610541-1910673071.png)

###  2、复制收集算法

标记清除算法有一个缺点，就是在分配了大量对象，并且其中只有一小部分存活的情况下，所消耗的时间会大大超过必要的值，这是因为在清除阶段还需要对大量死亡对象进行扫描。复制收集（Copy and Collection）则试图克服这一缺点。在这种算法中，会将从根开始被引用的对象复制到另外的空间中，然后，再将复制的对象所能够引用的对象用递归的方式不断复制下去。

![img](https://segmentfault.com/img/bVtJJw)



1）部分是GC开始前的内存状态，这和图1的（1）部分是一样的。

2）部分中，在旧对象所在的“旧空间”以外，再准备出一块“新空间”，并将可能从根被引用的对象复制到新空间中。

3）部分中，从已经复制的对象开始，再将可以被引用的对象像一串糖葫芦一样复制到新空间中。复制完成之后，“死亡”对象就被留在了旧空间中。

4）部分中，将旧空间废弃掉，就可以将死亡对象所占用的空间一口气全部释放出来，而没有必要再次扫描每个对象。下次GC的时候，现在的新空间也就变成了将来的旧空间。

通过图2我们可以发现，复制收集方式中，只存在相当于标记清除方式中的标记阶段。由于清除阶段中需要对现存的所有对象进行扫描，在存在大量对象，且其中大部分都即将死亡的情况下，全部扫描一遍的开销实在是不小。而在复制收集方式中，就不存在这样的开销。

但是，和标记相比，将对象复制一份所需要的开销则比较大，因此在“存活”对象比例较高的情况下，反而会比较不利。这种算法的另一个好处是它具有局部性（Lo-cality）。在复制收集过程中，会按照对象被引用的顺序将对象复制到新空间中。于是，关系较近的对象被放在距离较近的内存空间中的可能性会提高，这被称为局部性。局部性高的情况下，内存缓存会更容易有效运作，程序的运行性能也能够得到提高。

###  3、引用计数法

引用计数（Reference Count）方式是GC算法中最简单也最容易实现的一种，它和标记清除方式差不多是在同一时间发明出来的。它的基本原理是，在每个对象中保存该对象的引用计数，当引用发生增减时对计数进行更新。引用计数的增减，一般发生在变量赋值、对象内容更新、函数结束（局部变量不再被引用）等时间点。当一个对象的引用计数变为0时，则说明它将来不会再被引用，因此可以释放相应的内存空间。

![img](https://segmentfault.com/img/bVtJJJ)



1）部分中，所有对象中都保存着自己被多少个其他对象进行引用的数量（引用计数），图中每个对象右上角的数字就是引用计数。

2）部分中，当对象引用发生变化时，引用计数也跟着变化。在这里，由对象B到对象D的引用失效了，于是对象D的引用计数变为0。由于对象D的引用计数为0，因此由对象D到对象C和E的引用数也分别相应减少。结果，对象E的引用计数也变为0，于是对象E也被释放掉了。

3）部分中，引用计数变为0的对象被释放，“存活”对象则保留了下来。大家应该注意到，在整个GC处理过程中，并不需要对所有对象进行扫描。

实现容易是引用计数算法最大的优点。标记清除和复制收集这些GC机制在实现上都有一定难度；而引用计数方式的话，凡是有些年头的C++程序员（包括我在内），应该都曾经实现过类似的机制，可以说这种算法是相当具有普遍性的。

除此之外，当对象不再被引用的瞬间就会被释放，这也是一个优点。其他的GC机制中，要预测一个对象何时会被释放是很困难的，而在引用计数方式中则是立即被释放的。

由于释放操作是针对每个对象个别执行的，因此和其他算法相比，由GC而产生的中断时间（Pause time）就比较短，这也是一个优点。

引用计数方式的缺点另一方面，这种方式也有缺点。

1、引用计数最大的缺点，就是无法释放循环引用的对象。

![img](https://segmentfault.com/img/bVtJJM)

图4中，A、B、C三个对象没有被其他对象引用，而是互相之间循环引用，因此它们的引用计数永远不会为0，结果这些对象就永远不会被释放。

不过针对这个问题，也出了很多解决方案，比如强引用等。

循环引用有哪些解决方案。

2、 引用计数的第二个缺点，就是必须在引用发生增减时对引用计数做出正确的增减，而如果漏掉了某个增减的话，就会引发很难找到原因的内存错误。

引用数忘了增加的话，会对不恰当的对象进行释放；而引用数忘了减少的话，对象会一直残留在内存中，从而导致内存泄漏。如果语言编译器本身对引用计数进行管理的话还好，否则，如果是手动管理引用计数的话，那将成为孕育bug的温床。

3、 最后一个缺点就是，引用计数管理并不适合并行处理。如果多个线程同时对引用计数进行增减的话，引用计数的值就可能会产生不一致的问题（结果则会导致内存错误）。

为了避免这种情况的发生，对引用计数的操作必须采用独占的方式来进行。如果引用操作频繁发生，每次都要使用加锁等并发控制机制的话，其开销也是不可小觑的。综上所述，引用计数方式的原理和实现虽然简单，但缺点也很多，因此最近基本上不再使用了。现在，依然采用引用计数方式的语言主要有Perl和[Python](http://lib.csdn.net/base/python)，但它们为了避免循环引用的问题，都配合使用了其他的GC机制。这些语言中，GC基本上是通过引用计数方式来进行的，但偶尔也会用其他的算法来执行GC，这样就可以将引用计数方式无法回收的那些对象处理掉。

### 3  golang GC算法

当前Golang使用的垃圾回收机制是**三色标记发**配合**写屏障**和**辅助GC**，三色标记法是**标记-清除法**的一种增强版本。

原始的标记清除法分为两个步骤：

1. 标记。先STP(Stop The World)，暂停整个程序的全部运行线程，将被引用的对象打上标记
2. 清除没有被打标记的对象，即回收内存资源，然后恢复运行线程。

这样做有个很大的问题就是要通过STW保证GC期间标记对象的状态不能变化，整个程序都要暂停掉，在外部看来程序就会卡顿。

## 三色标记法

针对原生标记-清扫算法标记过程会STW的缺点，三色标记法改进了标记方案。三色标记法将所有对象分为三类:

- 白色: GC的候选对象集合(待处理)
- 灰色: 可从根访问，并且还未扫描对白色集合对象的引用(处理中,不会被GC,但引用待确认)
- 黑色: 可从根访问，并且不存在对白色集合的引用(处理完成)

步骤如下:

1. 初始化，所有对象都是白色
2. 从根遍历，所有可达对象标记为灰色
3. 从灰色对象队列中取出对象，将其引用的对象标记为灰色，并将自己标记为黑色
4. 重复第三步，直到灰色队列为空，此时白色对象即为孤儿对象，进行回收

三色标记法有个重要的不变量: **黑色对象不会引用任何白色对象**，因此白色对象可以在灰色对象处理完成之后立即回收。此算法最大的特点在于将标记过程拆分和量化，使得用户程序和标记过程可并行执行(需要其它技术追踪标记过程中的对象引用变更)，不用Stop-the-world，算法可按照各个集合的大小阶段性执行GC，并且不用遍历整个内存空间。

也可以参考下面的动图辅助理解：

![img](https://github.com/guyan0319/golang_development_notes/blob/master/images/9.6.1.gif?raw=true)



## Go GC如何工作

上面介绍的是GO GC采用的三色标记算法，但是好像并没有体现出来怎么减少STW对程序的影响呢？其实是因为**Golang GC的大部分处理是和用户代码并行的**。

GC期间用户代码可能会改变某些对象的状态，如何实现GC和用户代码并行呢？先看下GC工作的完整流程：

1. Mark: 包含两部分:

- Mark Prepare: 初始化GC任务，包括开启写屏障(write barrier)和辅助GC(mutator assist)，统计root对象的任务数量等。**这个过程需要STW**
- GC Drains: 扫描所有root对象，包括全局指针和goroutine(G)栈上的指针（扫描对应G栈时需停止该G)，将其加入标记队列(灰色队列)，并循环处理灰色队列的对象，直到灰色队列为空。**该过程后台并行执行**

1. Mark Termination: 完成标记工作，重新扫描(re-scan)全局指针和栈。因为Mark和用户程序是并行的，所以在Mark过程中可能会有新的对象分配和指针赋值，这个时候就需要通过写屏障（write barrier）记录下来，re-scan 再检查一下。**这个过程也是会STW的。**
2. Sweep: 按照标记结果回收所有的白色对象，**该过程后台并行执行**
3. Sweep Termination: 对未清扫的span进行清扫, 只有上一轮的GC的清扫工作完成才可以开始新一轮的GC。 如果标记期间用户逻辑改变了刚打完标记的对象的引用状态，怎么办呢？

## 写屏障(Write Barrier)

写屏障：该屏障之前的写操作和之后的写操作相比，先被系统其它组件感知。 好难懂哦，结合上面GC工作的完整流程就好理解了，就是在每一轮GC开始时会初始化一个叫做“屏障”的东西，然后由它记录第一次scan时各个对象的状态，以便和第二次re-scan进行比对，引用状态变化的对象被标记为灰色以防止丢失，将屏障前后状态未变化对象继续处理。

## 辅助GC

从上面的GC工作的完整流程可以看出Golang GC实际上把单次暂停时间分散掉了，本来程序执⾏可能是“⽤户代码-->⼤段GC-->⽤户代码”，那么分散以后实际上变成了“⽤户代码-->⼩段 GC-->⽤户代码-->⼩段GC-->⽤户代码”这样。如果GC回收的速度跟不上用户代码分配对象的速度呢？ Go 语⾔如果发现扫描后回收的速度跟不上分配的速度它依然会把⽤户逻辑暂停，⽤户逻辑暂停了以后也就意味着不会有新的对象出现，同时会把⽤户线程抢过来加⼊到垃圾回收⾥⾯加快垃圾回收的速度。这样⼀来原来的并发还是变成了STW，还是得把⽤户线程暂停掉，要不然扫描和回收没完没了了停不下来，因为新分配对象⽐回收快，所以这种东⻄叫做辅助回收。



### 触发

- 阈值：默认内存扩大一倍，启动gc，可配置
- 定期：默认2min触发一次gc，src/runtime/proc.go:forcegcperiod，可配置
- 手动：runtime.gc()



1、操作系统本身内存管理，会区分 栈空间，堆空间么？

2、程序申请内存的时候，会区分栈空间，堆空间么？

3、goroutine(G)栈 对应的是哪个栈,如何维护？



1、根结点是啥

2、如何判断一个节点引用了另一个节点

3、如何判断一个节点不用了



## 4、 go gc 源码

GO的GC是并行GC, 也就是GC的大部分处理和普通的go代码是同时运行的, 这让GO的GC流程比较复杂.
GC有四个阶段:

- Sweep Termination: 对未清扫的span进行清扫, 只有上一轮的GC的清扫工作完成才可以开始新一轮的GC
- Mark: 扫描所有根对象, 和根对象可以到达的所有对象, 标记它们不被回收
- Mark Termination: 完成标记工作, 重新扫描部分根对象(要求STW)
- Sweep: 按标记结果清扫span

在GC过程中会有两种后台任务(G), 一种是标记用的后台任务, 一种是清扫用的后台任务.
标记用的后台任务会在需要时启动, 可以同时工作的后台任务数量大约是P的数量的25%, 也就是go所讲的让25%的cpu用在GC上的根据.
清扫用的后台任务在程序启动时会启动一个, 进入清扫阶段时唤醒.

目前整个GC流程会进行两次STW(Stop The World), 第一次是Mark阶段的开始, 第二次是Mark Termination阶段.
第一次STW会准备根对象的扫描, 启动写屏障(Write Barrier)和辅助GC(mutator assist).
第二次STW会重新扫描部分根对象, 禁用写屏障(Write Barrier)和辅助GC(mutator assist).
需要注意的是, 不是所有根对象的扫描都需要STW, 例如扫描栈上的对象只需要停止拥有该栈的G.
从go 1.9开始, 写屏障的实现使用了Hybrid Write Barrier, 大幅减少了第二次STW的时间.

### GC的触发条件

GC在满足一定条件后会被触发, 触发条件有以下几种:

- gcTriggerAlways: 强制触发GC
- gcTriggerHeap: 当前分配的内存达到一定值就触发GC
- gcTriggerTime: 当一定时间没有执行过GC就触发GC
- gcTriggerCycle: 要求启动新一轮的GC, 已启动则跳过, 手动触发GC的`runtime.GC()`会使用这个条件

### 写屏障(Write Barrier)

因为go支持并行GC, GC的扫描和go代码可以同时运行, 这样带来的问题是GC扫描的过程中go代码有可能改变了对象的依赖树,
例如开始扫描时发现根对象A和B, B拥有C的指针, GC先扫描A, 然后B把C的指针交给A, GC再扫描B, 这时C就不会被扫描到.
为了避免这个问题, go在GC的标记阶段会启用写屏障(Write Barrier).

启用了写屏障(Write Barrier)后, 当B把C的指针交给A时, GC会认为在这一轮的扫描中C的指针是存活的,
即使A可能会在稍后丢掉C, 那么C就在下一轮回收.
写屏障只针对指针启用, 而且只在GC的标记阶段启用, 平时会直接把值写入到目标地址.

go在1.9开始启用了[混合写屏障(Hybrid Write Barrier)](https://github.com/golang/proposal/blob/master/design/17503-eliminate-rescan.md), 伪代码如下:

```
writePointer(slot, ptr):
    shade(*slot)
    if any stack is grey:
        shade(ptr)
    *slot = ptr
```

混合写屏障会同时标记指针写入目标的"原指针"和“新指针".

混合写屏障可以让GC在并行标记结束后不需要重新扫描各个G的堆栈, 可以减少Mark Termination中的STW时间.
除了写屏障外, 在GC的过程中所有新分配的对象都会立刻变为黑色, 在上面的mallocgc函数中可以看到.

### 辅助GC(mutator assist)

为了防止heap增速太快, 在GC执行的过程中如果同时运行的G分配了内存, 那么这个G会被要求辅助GC做一部分的工作.
在GC的过程中同时运行的G称为"mutator", "mutator assist"机制就是G辅助GC做一部分工作的机制.

辅助GC做的工作有两种类型, 一种是标记(Mark), 另一种是清扫(Sweep).
辅助标记的触发可以查看上面的mallocgc函数, 触发时G会帮助扫描"工作量"个对象, 工作量的计算公式是:

```
debtBytes * assistWorkPerByte
```

意思是分配的大小乘以系数assistWorkPerByte, assistWorkPerByte的计算在函数[revise](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgc.go#L497)中

### 根对象

在GC的标记阶段首先需要标记的就是"根对象", 从根对象开始可到达的所有对象都会被认为是存活的.
根对象包含了全局变量 mheap, 各个G的栈上的变量等（[各个G的栈](./g的栈空间)）, GC会先扫描根对象然后再扫描根对象可到达的所有对象.
扫描根对象包含了一系列的工作, 它们定义在[https://github.com/golang/go/blob/go1.9.2/src/runtime/mgcmark.go#L54]函数:

- Fixed Roots: 特殊的扫描工作
  - fixedRootFinalizers: 扫描析构器队列
  - fixedRootFreeGStacks: 释放已中止的G的栈
- Flush Cache Roots: 释放mcache中的所有span, 要求STW
- Data Roots: 扫描可读写的全局变量
- BSS Roots: 扫描只读的全局变量
- Span Roots: 扫描各个span中特殊对象(析构器列表)
- Stack Roots: 扫描各个G的栈

标记阶段(Mark)会做其中的"Fixed Roots", "Data Roots", "BSS Roots", "Span Roots", "Stack Roots".
完成标记阶段(Mark Termination)会做其中的"Fixed Roots", "Flush Cache Roots".



### 标记队列

GC的标记阶段会使用"标记队列"来确定所有可从根对象到达的对象都已标记, 上面提到的"灰色"的对象就是在标记队列中的对象.
举例来说, 如果当前有[A, B, C]这三个根对象, 那么扫描根对象时就会把它们放到标记队列:

```
work queue: [A, B, C]
```

后台标记任务从标记队列中取出A, 如果A引用了D, 则把D放入标记队列:

```
work queue: [B, C, D]
```

后台标记任务从标记队列取出B, 如果B也引用了D, 这时因为D在gcmarkBits中对应的bit已经是1所以会跳过:

```
work queue: [C, D]
```

如果并行运行的go代码分配了一个对象E, 对象E会被立刻标记, 但不会进入标记队列(因为确定E没有引用其他对象).
然后并行运行的go代码把对象F设置给对象E的成员, 写屏障会标记对象F然后把对象F加到运行队列:

```
work queue: [C, D, F]
```

后台标记任务从标记队列取出C, 如果C没有引用其他对象, 则不需要处理:

```
work queue: [D, F]
```

后台标记任务从标记队列取出D, 如果D引用了X, 则把X放入标记队列:

```
work queue: [F, X]
```

后台标记任务从标记队列取出F, 如果F没有引用其他对象, 则不需要处理.
后台标记任务从标记队列取出X, 如果X没有引用其他对象, 则不需要处理.
最后标记队列为空, 标记完成, 存活的对象有[A, B, C, D, E, F, X].

实际的状况会比上面介绍的状况稍微复杂一点.
标记队列会分为全局标记队列和各个P的本地标记队列, 这点和协程中的运行队列相似.
并且标记队列为空以后, 还需要停止整个世界并禁止写屏障, 然后再次检查是否为空.

### 源代码分析

go触发gc会从[gcStart](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgc.go#L1190)函数开始:

接下来一个个分析gcStart调用的函数, 建议配合"回收对象的流程"中的图理解.

下图是比较完整的GC流程, 并按颜色对这四个阶段进行了分类:

![img](https://images2017.cnblogs.com/blog/881857/201711/881857-20171122165749274-1840348396.png)



函数[gcBgMarkStartWorkers](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgc.go#L1650)用于启动后台标记任务, 先分别对每个P启动一个:

[gcResetMarkState](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgc.go#L2040)函数会重置标记相关的状态:

[stopTheWorldWithSema](https://github.com/golang/go/blob/go1.9.2/src/runtime/proc.go#L987)函数会停止整个世界, 这个函数必须在g0中运行:

[finishsweep_m](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgcsweep.go#L33)函数会清扫上一轮GC未清扫的span, 确保上一轮GC已完成:

[clearpools](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgc.go#L2065)函数会清理sched.sudogcache和sched.deferpool, 让它们的内存可以被回收:

[startCycle](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgc.go#L420)标记开始了新一轮的GC:

[setGCPhase](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgc.go#L279)函数会修改表示当前GC阶段的全局变量和**是否开启写屏障**的全局变量:

[gcBgMarkPrepare](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgc.go#L1667)函数会重置后台标记任务的计数:

[gcMarkRootPrepare](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgcmark.go#L54)函数会计算扫描根对象的任务数量:

[gcMarkTinyAllocs](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgcmark.go#L1358)函数会标记所有tiny alloc等待合并的对象:

[startTheWorldWithSema](https://github.com/golang/go/blob/go1.9.2/src/runtime/proc.go#L1070)函数会重新启动世界:

重启世界后各个M会重新开始调度, 调度时会优先使用上面提到的findRunnableGCWorker函数查找任务, 之后就有大约25%的P运行后台标记任务.
后台标记任务的函数是[gcBgMarkWorker](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgc.go#L1681):

[gcDrain](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgcmark.go#L889)函数用于执行标记:

[markroot](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgcmark.go#L178)函数用于执行根对象扫描工作:

[scang](https://github.com/golang/go/blob/go1.9.2/src/runtime/proc.go#L830)函数负责扫描G的栈:

[scanblock](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgcmark.go#L1059)函数是一个通用的扫描函数, 扫描全局变量和栈空间都会用它, 和scanobject不同的是bitmap需要手动传入:

[greyobject](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgcmark.go#L1216)用于标记一个对象存活, 并把它加到标记队列(该对象变为灰色):

gcDrain函数扫描完根对象, 就会开始消费标记队列, 对从标记队列中取出的对象调用[scanobject](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgcmark.go#L1098)函数:

在所有后台标记任务都把标记队列消费完毕时, 会执行[gcMarkDone](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgc.go#L1345)函数准备进入完成标记阶段(mark termination):
在并行GC中gcMarkDone会被执行两次, 第一次会禁止本地标记队列然后重新开始后台标记任务, 第二次会进入完成标记阶段(mark termination)。

[gcMarkTermination](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgc.go#L1449)函数会进入完成标记阶段:

[gcSweep](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgc.go#L1987)函数会唤醒后台清扫任务:
后台清扫任务会在程序启动时调用的[gcenable](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgc.go#L214)函数中启动.

[gosweepone](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgcsweep.go#L134)函数会从sweepSpans中取出单个span清扫:

span的[sweep](https://github.com/golang/go/blob/go1.9.2/src/runtime/mgcsweep.go#L179)函数用于清扫单个span:





