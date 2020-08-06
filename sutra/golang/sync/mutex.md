#mutex 

### Mutex 演进史

1. 最朴素的实现互斥锁，拿到锁返回，拿不到就将当前 goroutine 休眠
2. 自旋锁，也就是说大部份 Mutex 锁住时间如果很短，那么自旋可以减小无谓的 runtime 调度。推荐看官方 spin [commit](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fgolang%2Fgo%2Fcommit%2Fedcad8639a902741dc49f77d000ed62b0cc6956f)
3. 公平锁，表现为普通模式/饥饿模式，普通模式当前抢锁中的 goroutine 大概率比休眠的goroutine优先拿到锁，会产生 latency 长尾。新版本中超过一定时间没拿到锁，会转变成饥饿模式，按照FIFO的顺序获取锁，在某些情况下会再转变为普通模式，以便减小长尾的高延迟。推荐大家看 [#issue 13086](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fgolang%2Fgo%2Fissues%2F13086)，这里面反映了问题，另外看 [commit](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fgolang%2Fgo%2Fcommit%2F0556e26273f704db73df9e7c4c3d2e8434dec7be), 里面有很详细的测试数据，值得学习

### 结构

```go
type Mutex struct {
	state int32
	sema  uint32
}
```

`state `的低三位分别表示 `mutexLocked`、`mutexWoken` 和 `mutexStarving`，高29位表示当前等待锁的goroutine的个数

`sema` 是一个互斥的信号量，初始默认值是 0，用于将` goroutine park` 休眠或是唤醒。`sema acquire` 时如果 `sema` 大于 0，那么减一返回，否则休眠等待。`sema release` 将 `sema` 加一，然后唤醒等待队列的第一个 `goroutine`

### 公平锁的实现逻辑



```cpp
    // Mutex fairness.
    //
    // Mutex can be in 2 modes of operations: normal and starvation.
    // In normal mode waiters are queued in FIFO order, but a woken up waiter
    // does not own the mutex and competes with new arriving goroutines over
    // the ownership. New arriving goroutines have an advantage -- they are
    // already running on CPU and there can be lots of them, so a woken up
    // waiter has good chances of losing. In such case it is queued at front
    // of the wait queue. If a waiter fails to acquire the mutex for more than 1ms,
    // it switches mutex to the starvation mode.
    //
    // In starvation mode ownership of the mutex is directly handed off from
    // the unlocking goroutine to the waiter at the front of the queue.
    // New arriving goroutines don't try to acquire the mutex even if it appears
    // to be unlocked, and don't try to spin. Instead they queue themselves at
    // the tail of the wait queue.
    //
    // If a waiter receives ownership of the mutex and sees that either
    // (1) it is the last waiter in the queue, or (2) it waited for less than 1 ms,
    // it switches mutex back to normal operation mode.
    //
    // Normal mode has considerably better performance as a goroutine can acquire
    // a mutex several times in a row even if there are blocked waiters.
    // Starvation mode is important to prevent pathological cases of tail latency.
```

代码以 go1.12 为例，可以看到注释关于公平锁的实现初衷和逻辑。越是基础组件更新越严格，背后肯定有相关测试数据。

1. Mutex 两种工作模式，normal 正常模式，starvation 饥饿模式。normal 情况下锁的逻辑与老版相似，休眠的 goroutine 以 FIFO 链表形式保存在 sudog 中，被唤醒的 goroutine 与新到来活跃的 goroutine 竞解，但是很可能会失败。如果一个 goroutine 等待超过 1ms，那么 Mutex 进入饥饿模式
2. 饥饿模式下，解锁后，锁直接交给 waiter FIFO 链表的第一个，新来的活跃 goroutine 不参与竞争，并放到 FIFO 队尾
3. 如果当前获得锁的 goroutine 是 FIFO 队尾，或是等待时长小于 1ms，那么退出饥饿模式
4. normal 模式下性能是比较好的，但是 starvation 模式能减小长尾 latency

#### 上锁

```go
// Lock locks m.
// If the lock is already in use, the calling goroutine
// blocks until the mutex is available.
func (m *Mutex) Lock() {
    // Fast path: grab unlocked mutex. 快速上锁逻辑
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        if race.Enabled {
            race.Acquire(unsafe.Pointer(m))
        }
        return
    }

    var waitStartTime int64 // waitStartTime 用于判断是否需要进入饥饿模式
    starving := false // 饥饿标记
    awoke := false // 是否被唤醒
    iter := 0 // spin 循环次数
    old := m.state
    for {
        // Don't spin in starvation mode, ownership is handed off to waiters
        // so we won't be able to acquire the mutex anyway. 饥饿模式下不进行自旋，直接进入阻塞队列
        if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // Active spinning makes sense.
            // Try to set mutexWoken flag to inform Unlock
            // to not wake other blocked goroutines.
            if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                awoke = true
            }
            runtime_doSpin()
            iter++
            old = m.state
            continue
        }
        new := old
        // Don't try to acquire starving mutex, new arriving goroutines must queue.
        if old&mutexStarving == 0 { // 只有此时不是饥饿模式时，才设置 mutexLocked，也就是说饥饿模式下的活跃 goroutine 直接排队去
            new |= mutexLocked
        }
        if old&(mutexLocked|mutexStarving) != 0 { // 处于己经上锁或是饥饿时，waiter 计数 + 1
            new += 1 << mutexWaiterShift
        }
        // The current goroutine switches mutex to starvation mode.
        // But if the mutex is currently unlocked, don't do the switch.
        // Unlock expects that starving mutex has waiters, which will not
        // be true in this case. 如果当前处于饥饿模式下，并且己经上锁了，mutexStarving 置 1，接下来 CAS 会用到
        if starving && old&mutexLocked != 0 {
            new |= mutexStarving
        }
        if awoke { // 如果当前 goroutine 是被唤醒的，然后清 mutexWoken 位
            // The goroutine has been woken from sleep,
            // so we need to reset the flag in either case.
            if new&mutexWoken == 0 {
                throw("sync: inconsistent mutex state")
            }
            new &^= mutexWoken
        }
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            if old&(mutexLocked|mutexStarving) == 0 { // 如果 old 没有上锁并且也不是饥饿模式，上锁成功直接退出
                break // locked the mutex with CAS
            }
            // If we were already waiting before, queue at the front of the queue.
            queueLifo := waitStartTime != 0 // 第一次 queueLifo 肯定是 false
            if waitStartTime == 0 {
                waitStartTime = runtime_nanotime() 
            }
            runtime_SemacquireMutex(&m.sema, queueLifo) // park 在这里，如果 queueLifo 为真，那么扔到队头，也就是 LIFO
      // 走到这里，说明被其它 goroutine 唤醒了，继续抢锁时先判断是否需要进入 starving
            starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs // 超过 1ms 就进入饥饿模式
            old = m.state
            if old&mutexStarving != 0 { // 如果原来就是饥饿模式的话，走 if 逻辑
                // If this goroutine was woken and mutex is in starvation mode,
                // ownership was handed off to us but mutex is in somewhat
                // inconsistent state: mutexLocked is not set and we are still
                // accounted as waiter. Fix that.
                if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
                    throw("sync: inconsistent mutex state")
                }
        // 此时饥饿模式下被唤醒，那么一定能上锁成功。因为 Unlock 保证饥饿模式下只唤醒 park 状态的 goroutine
                delta := int32(mutexLocked - 1<<mutexWaiterShift) // waiter 计数 -1
                if !starving || old>>mutexWaiterShift == 1 { // 如果是饥饿模式下并且自己是最后一个 waiter ，那么清除 mutexStarving 标记
                    // Exit starvation mode.
                    // Critical to do it here and consider wait time.
                    // Starvation mode is so inefficient, that two goroutines
                    // can go lock-step infinitely once they switch mutex
                    // to starvation mode.
                    delta -= mutexStarving
                }
                atomic.AddInt32(&m.state, delta) // 更新，抢锁成功后退出
                break
            }
            awoke = true // 走到这里，不是饥饿模式，重新发起抢锁竞争
            iter = 0
        } else {
            old = m.state // CAS 失败，重新发起竞争
        }
    }

    if race.Enabled {
        race.Acquire(unsafe.Pointer(m))
    }
}
```

整体来讲，公平锁上锁逻辑复杂了不少，边界点要考滤的比较多

1. 同样的 fast path 快速上锁逻辑，原来 m.state 为 0，锁就完事了
2. 进入 for 循环，也要走自旋逻辑，但是多了一个判断，如果当前处于饥饿模式禁止自旋，根据实现原理，此时活跃的 goroutine 要直接进入 park 的队列
3. 自旋后面的代码有四种情况：饥饿抢锁成功，饥饿抢锁失败，正常抢锁成历，正常抢锁失败。上锁失败的最后都要 waiter 计数加一后，更新 CAS
4. 如果 CAS 失败，那么重新发起竞争就好
5. 如果 CAS 成功，此时要判断处于何种情况，如果 old 没上锁也处于 normal 模式，抢锁成历退出
6. 如果 CAS 成功，但是己经有人上锁了，那么要根据 queueLifo 来判断是扔到 park 队首还是队尾，此时当前 goroutine park 在这里，等待被唤醒
7.  `runtime_SemacquireMutex` 被唤醒了有两种情况，判断是否要进入饥饿模式，如果老的 old 就是饥饿的，那么自己一定是唯一被唤醒，一定能抢到锁的，waiter 减一，如果自己是最后一个 waiter 或是饥饿时间小于 starvationThresholdNs 那么清除 mutexStarving 标记位后退出
8. 如果老的不是饥饿模式，那么 awoke 置 true，重新竞争

#### 解锁

```go
// Unlock unlocks m.
// It is a run-time error if m is not locked on entry to Unlock.
//
// A locked Mutex is not associated with a particular goroutine.
// It is allowed for one goroutine to lock a Mutex and then
// arrange for another goroutine to unlock it.
func (m *Mutex) Unlock() {
    if race.Enabled {
        _ = m.state
        race.Release(unsafe.Pointer(m))
    }

    // Fast path: drop lock bit. 和原有逻辑一样，先减去 mutexLocked，并判断是否解锁了未上锁的 Mutex, 直接 panic
    new := atomic.AddInt32(&m.state, -mutexLocked)
    if (new+mutexLocked)&mutexLocked == 0 {
        throw("sync: unlock of unlocked mutex")
    }
    if new&mutexStarving == 0 { // 查看 mutexStarving 标记位，如果 0 走老逻辑，否则走 starvation 分支
        old := new
        for {
            // If there are no waiters or a goroutine has already
            // been woken or grabbed the lock, no need to wake anyone.
            // In starvation mode ownership is directly handed off from unlocking
            // goroutine to the next waiter. We are not part of this chain,
            // since we did not observe mutexStarving when we unlocked the mutex above.
            // So get off the way.
            if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
                return
            }
            // Grab the right to wake someone.
            new = (old - 1<<mutexWaiterShift) | mutexWoken
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                runtime_Semrelease(&m.sema, false)
                return
            }
            old = m.state
        }
    } else {
        // Starving mode: handoff mutex ownership to the next waiter.
        // Note: mutexLocked is not set, the waiter will set it after wakeup.
        // But mutex is still considered locked if mutexStarving is set,
        // so new coming goroutines won't acquire it.
        runtime_Semrelease(&m.sema, true) // 直接 runtime_Semrelease 唤醒等待的 goroutine
    }
}
```

1. 原子操作，将 m.state 减去 mutexLocked，然后判断是否释放了未上锁的 Mutex，直接 panic
2. 根据 m.state 的 mutexStarving 判断当前处于何种模式，0 走 normal 分支，1 走 starvation 分支
3. starvation 模式下，直接 `runtime_Semrelease` 做信号量 UP 操作，唤醒 FIFO 队列中的第一个 goroutine
4. noarmal 模式类似原有逻辑，唯一不同的是多了一个  mutexStarving 位判断逻辑

### 参考资料

[图解 mutex](https://www.jianshu.com/p/617b77ff7fa6)
[go语言设计与实现](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-sync-primitives/#mutex)

