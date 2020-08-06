## waitgroup

#### 结构体

[`sync.WaitGroup`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L20-L29) 结构体中的成员变量非常简单，其中只包含两个成员变量：

```go
type WaitGroup struct {
	noCopy noCopy
	state1 [3]uint32
}
```

- `noCopy` — 保证 [`sync.WaitGroup`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L20-L29) 不会被开发者通过再赋值的方式拷贝；
- `state1` — 存储着状态和信号量；

[`sync.WaitGroup`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L20-L29) 对外暴露了三个方法 

`sync.WaitGroup.Add` 

```
func (wg *WaitGroup) Add(delta int) {
   statep, semap := wg.state()
  
   state := atomic.AddUint64(statep, uint64(delta)<<32)
   v := int32(state >> 32)
   w := uint32(state)
   if v < 0 {
      panic("sync: negative WaitGroup counter")
   }
   // 在有协程等待的时候 delta >0 并且 add结果等于 delta ，panic
   if w != 0 && delta > 0 && v == int32(delta) {
      panic("sync: WaitGroup misuse: Add called concurrently with Wait")
   }
   //正常退出
   if v > 0 || w == 0 {
      return
   }
   if *statep != state {
      panic("sync: WaitGroup misuse: Add called concurrently with Wait")
   }
   // Reset waiters count to 0.

   //v =0，唤醒所有等待的协程
   *statep = 0
   for ; w != 0; w-- {
      runtime_Semrelease(semap, false, 0)
   }
}
```

1、state 高32 位为运行中的 goroutine 个数，计为counter， 低32位为 等待counter等于0 的 goroutine的个数，计为waiter
2、如果v 小于0 会panic

3、有协程wait的时候，add会导致panic

4、当v ==0  的时候，依次唤醒 所有等待的协程

`sync.WaitGroup.Wait`

```
func (wg *WaitGroup) Wait() {
   statep, semap := wg.state()
   for {
      state := atomic.LoadUint64(statep)
      v := int32(state >> 32)
      w := uint32(state)
      if v == 0 {
         // Counter is 0, no need to wait.
         return
      }
      // Increment waiters count.
      if atomic.CompareAndSwapUint64(statep, state, state+1) {
         runtime_Semacquire(semap)
         if *statep != 0 {
            panic("sync: WaitGroup is reused before previous Wait has returned")
         }
         return
      }
   }
}
```

1、如果当前 counter数为0 ，无需做任何事，直接返回

2、waiter数字+1，阻塞当前协程，等待 counter为0时被唤醒

3、唤醒后会检查state的值，如果不为0会 panic

`sync.WaitGroup.Done` done 是对add 方法对简单封装，等同于 add(-1)

### 小结

1、可以有多个 waiter等待 counter 归零，具体使用场景？

2、wait返回前，waitgroup 不能复用 

###参考资料

[Go 语言设计与实现](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-sync-primitives/#waitgroup)

