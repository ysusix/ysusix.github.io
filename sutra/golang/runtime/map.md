# map 源码梳理

## 1、Map怎么用

```
func main() {

   mmap := make(map[int]int, 0)

   mmap[1] = 1 // 增

   if mmap[1] == 1 { // 查询1
      fmt.Println("111")
   }

   mmap[1] = 2 // 改

   if v, ok := mmap[1]; ok { // 查询2
      fmt.Println("v is ", v)
   }

	for k,v := range mmap{
		fmt.Println("k,v is ", k,v)
	}
   delete(mmap, 1) //删
}
```

## 2 、Map相关问题

- [哈希表原理](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/)
  - 开放寻址法
  - 拉链法，数组+链表
  - 拉链法的哈希中引入红黑树，以优化性能 [java  hashMap](https://www.cnblogs.com/liwei2222/p/8013367.html)
  - 多个哈希函数

- 2.1 map遍历时是是否有序？
- 2.2 map是线程安全的吗？是否能同时读写？如何解决？
- 2.3 golang 实现的map，如何hash，如何解决碰撞问题？

## 3、源码解析

以下分底层结构，增删改查，两种不同扩容方式（倍增扩容及等量扩容），遍历及随机性（扩容期间如何遍历）几部分解析。

###  3.1 Map底层结构

Go 语言运行时同时使用了多个数据结构组合表示哈希表，其中使用 [`hmap`](https://github.com/golang/go/blob/ed15e82413c7b16e21a493f5a647f68b46e965ee/src/runtime/map.go#L115-L129) 结构体来表示哈希，我们先来看一下这个结构体内部的字段：

```go
type hmap struct {
	count     int    //当前哈希表中的元素数
	flags     uint8  //标记读写状态
	B         uint8  // map中桶的个数的对数，例如桶个数为 8，则B=3
	noverflow uint16 // 当前溢出桶的个数，大概个数，超过一定数量就不准了
	hash0     uint32 // 哈希的种子，为哈希函数的结果引入随机性，这个值在创建哈希表时确定，并在调用哈希函数时      作为参数传入
	buckets    unsafe.Pointer
	oldbuckets unsafe.Pointer // 哈希在扩容时用于保存之前 buckets 的字段，哈希发生扩容时开辟新的内存空间，buckets赋值给 oldbuckets， buckets指向新的内存空间。
	nevacuate  uintptr  //标记已搬迁桶个数

  extra *mapextra     // overflow 溢出桶的内存指针，在桶个数较多时(B>4)，会提前申请对应的溢出桶内存。溢出桶的个数大概是 2**(B-4)个
}
```

flags 有如下几个标记位

```
// flags
iterator     = 1 // 遍历新桶，there may be an iterator using buckets 
oldIterator  = 2 // 遍历旧桶there may be an iterator using oldbuckets
hashWriting  = 4 // 当前有线程写 map,a goroutine is writing to the map
sameSizeGrow = 8 // map正在等量扩容，the current map growth is to a new map of the same size
```

![hmap-and-buckets](/Users/baijichuan/Documents/学习笔记/2019-12-30-15777168478811-hmap-and-buckets.png)



如上图所示哈希表 `hmap` 的桶就是 `bmap`，每一个 `bmap` 都能存储 8 个键值对，当哈希表中存储的数据过多，单个桶无法装满时就会使用 `extra.overflow` 中的溢出桶存储数据。上述两种不同的桶在内存中是连续存储的，我们在这里将它们分别称为正常桶和溢出桶，上图中黄色的 `bmap` 就是正常桶，绿色的 `bmap` 是溢出桶，溢出桶是在 Go 语言还使用 C 语言实现时就使用的设计[3](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/#fn:3)，由于它能够减少扩容的频率所以一直使用至今。

这个桶的结构体 `bmap` 在 Go 语言源代码中的定义只包含一个简单的 `tophash` 字段，`tophash` 存储了键的哈希的高 8 位，通过比较不同键的哈希的高 8 位可以减少访问键值对次数以提高性能：

```go
type bmap struct {
	tophash [bucketCnt]uint8 // key 哈希值的高 8 位，目前哈希值小于5为特殊标记位
}
```

tophash 可能有的值

```
emptyRest      = 0 // 当前cell及该cell之后都为空，可以减少遍历长度。
emptyOne       = 1 //当前cell为空
evacuatedX     = 2 // key/elem is valid.  Entry has been evacuated to first half of larger table.
evacuatedY     = 3 // same as above, but evacuated to second half of larger table.
evacuatedEmpty = 4 // 当前cell为空，数据已经迁移到了新桶
```

![hashtable-overflow-bucket](https://img.draveness.me/2019-12-30-15777168478823-hashtable-overflow-bucket.png)

`bmap` 结构体其实不止包含 `tophash` 字段，由于哈希表中可能存储不同类型的键值对并且 Go 语言也不支持泛型，所以键值对占据的内存空间大小只能在编译时进行推导，这些字段在运行时也都是通过计算内存地址的方式直接访问的，所以它的定义中就没有包含这些字段，但是我们能根据编译期间的 [`cmd/compile/internal/gc.bmap`](https://github.com/golang/go/blob/be64a19d99918c843f8555aad580221207ea35bc/src/cmd/compile/internal/gc/reflect.go#L82-L187) 函数对它的结构重建：

```go
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}
```



如果哈希表存储的数据逐渐增多，我们会对哈希表进行扩容或者使用额外的桶存储溢出的数据，不会让单个桶中的数据超过 8 个，不过溢出桶只是临时的解决方案，创建过多的溢出桶最终也会导致哈希的扩容。

从 Go 语言哈希的定义中就可以发现，它比前面两节提到的数组和切片复杂得多，结构体中不仅包含大量字段，还使用了较多的复杂结构，在后面的小节中我们会详细介绍不同字段的作用。

### 3.2 Map初始化

```go
func makemap(t *maptype, hint int, h *hmap) *hmap {
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}
	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()

	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}
	return h
}
```

这个函数的执行过程会分成以下几个部分：

1. 计算哈希占用的内存是否溢出或者超出能分配的最大值；
2. 调用 `fastrand` 获取一个随机的哈希种子 hash0；
3. 根据传入的 `hint` 计算出需要的最小需要的桶的数量，即B的值；
4. 使用 [`runtime.makeBucketArray`](https://github.com/golang/go/blob/dcd3b2c173b77d93be1c391e3b5f932e0779fb1f/src/runtime/map.go#L344-L387) 创建用于保存桶的数组；

[`runtime.makeBucketArray`](https://github.com/golang/go/blob/dcd3b2c173b77d93be1c391e3b5f932e0779fb1f/src/runtime/map.go#L344-L387) 函数会根据传入的 `B` 计算出的需要创建的桶数量在内存中分配一片连续的空间用于存储数据：

```go
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b)
	nbuckets := base
	if b >= 4 {
		nbuckets += bucketShift(b - 4)
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}

	buckets = newarray(t.bucket, int(nbuckets))
	if base != nbuckets {
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```

当桶的数量对数 `B` 小于4 时（16个桶），由于数据较少、使用溢出桶的可能性较低，这时就会省略创建的过程以减少额外开销；当桶的数量对数 `B` 多于 4 时，就会额外创建 2**(B−4) 个溢出桶，根据上述代码，我们能确定在正常情况下，正常桶和溢出桶在内存中的存储空间是连续的，只是被 `hmap` 中的不同字段引用，当溢出桶数量较多时会通过 [`runtime.newobject`](https://github.com/golang/go/blob/921ceadd2997f2c0267455e13f909df044234805/src/runtime/malloc.go#L1164) 创建新的溢出桶。

### 3.3 查找和写入

查找有以下两种形式

```go
v     := hash[key] // => v     := *mapaccess1(maptype, hash, &key)
v, ok := hash[key] 
```

赋值语句左侧接受参数的个数会决定使用的运行时方法：

1. 当接受参数仅为一个时，会使用 [`runtime.mapaccess1`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L394-L450)，该函数仅会返回一个指向目标值的指针；
2. 当接受两个参数时，会使用 [`runtime.mapaccess2`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L452-L508)，除了返回目标值之外，它还会返回一个用于表示当前键对应的值是否存在的布尔值：

[`runtime.mapaccess1`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L394-L450) 函数会先通过哈希表设置的哈希函数、种子获取当前键对应的哈希，再通过 `bucketMask` 和 `add` 函数拿到该键值对所在的桶序号和哈希最上面的 8 位数字。

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))
	m := bucketMask(h.B)
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
	top := tophash(hash)
bucketloop:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if alg.equal(key, k) {
				v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				return v
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0])
}
```

// 为什么不用数组取址，而要算内存地址

在 `bucketloop` 循环中，哈希会依次遍历正常桶和溢出桶中的数据，它会比较这 8 位数字和桶中存储的 `tophash`，每一个桶都存储键对应的 `tophash`，每一次读写操作都会与桶中所有的 `tophash` 进行比较，用于选择桶序号的是哈希的最低几位，而用于加速访问的是哈希的高 8 位，不相等直接跳过，如果hash值为 `emptyRest`，说明该cell及之后所有高位，包括溢出桶的cell都为空，直接跳出循环，没找到数据。

![hashtable-mapaccess](https://img.draveness.me/2019-12-30-15777168478817-hashtable-mapaccess.png)

**图 3-13 访问哈希表中的数据**

如上图所示，每一个桶都是一整片的内存空间，当发现桶中的 `tophash` 与传入键的 `tophash` 匹配之后，我们会通过指针和偏移量获取哈希中存储的键 `keys[0]` 并与 `key` 比较，如果两者相同就会获取目标值的指针 `values[0]` 并返回。

另一个同样用于访问哈希表中数据的 [`runtime.mapaccess2`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L452-L508) 只是在 [`runtime.mapaccess1`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L394-L450) 的基础上多返回了一个标识键值对是否存在的布尔值：

```go
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool) {
	...
bucketloop:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if alg.equal(key, k) {
				v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				return v, true
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0]), false
}
```

使用 `v, ok := hash[k]` 的形式访问哈希表中元素时，我们能够通过这个布尔值更准确地知道当 `v == nil` 时，`v` 到底是哈希中存储的元素还是表示该键对应的元素不存在，所以在访问哈希时，更推荐使用这一种方式先判断元素是否存在。

#### 写入

当形如 `hash[k]` 的表达式出现在赋值符号左侧时，该表达式也会在编译期间转换成调用 [`runtime.mapassign`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L571-L683) 函数，该函数与 [`runtime.mapaccess1`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L394-L450) 比较相似，我们将该其分成几个部分分析，首先是函数会根据传入的键拿到对应的哈希和桶：

```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))

	h.flags ^= hashWriting

again:
	bucket := hash & bucketMask(h.B)
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
	top := tophash(hash)
```

然后通过遍历比较桶中存储的 `tophash` 和键的哈希，如果找到了相同结果，说明存在相同key，为更新，获取目标位置的地址并返回，其中 `inserti` 表示目标元素的在桶中的索引，`insertk` 和 `val` 分别表示键值对的地址，获得目标地址之后会直接通过算术计算进行寻址获得键值对 `k` 和 `val`：

```go
	var inserti *uint8
	var insertk unsafe.Pointer
	var val unsafe.Pointer
bucketloop:
	for {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if isEmpty(b.tophash[i]) && inserti == nil {
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				}
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if !alg.equal(key, k) {
				continue
			}
			val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
			goto done
		}
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		b = ovf
	}
```

在上述的 for 循环中会依次遍历正常桶和溢出桶中存储的数据，整个过程会依次判断 `tophash` 是否相等、`key` 是否相等，遍历结束后会从循环中跳出。

![hashtable-overflow-bucket](https://img.draveness.me/2019-12-30-15777168478823-hashtable-overflow-bucket.png)

**图 3-15 哈希遍历溢出桶**

如果当前桶已经满了，哈希会调用 `newoverflow` 函数创建新桶或者使用 `hmap` 预先在 `noverflow` 中创建好的桶来保存数据，新创建的桶不仅会被追加到已有桶的末尾，还会增加哈希表的 `noverflow` 计数器（B<15时`newoverflow`为精确值，B>15后为大概值 ）。

```go
	if inserti == nil {
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		val = add(insertk, bucketCnt*uintptr(t.keysize))
	}

	typedmemmove(t.key, insertk, key)
	*inserti = top
	h.count++

done:
	return val
}
```

如果当前键值对在哈希中不存在，哈希为新键值对规划存储的内存地址，通过 `typedmemmove` 将键移动到对应的内存空间中并返回键对应值的地址 `val`，如果当前键值对在哈希中存在，那么就会直接返回目标区域的内存地址。哈希并不会在 `mapassign` 这个运行时函数中将值拷贝到桶中，该函数只会返回内存地址，真正的赋值操作是在编译期间插入的：

```go
00018 (+5) CALL runtime.mapassign_fast64(SB)
00020 (5) MOVQ 24(SP), DI               ;; DI = &value
00026 (5) LEAQ go.string."88"(SB), AX   ;; AX = &"88"
00027 (5) MOVQ AX, (DI)                 ;; *DI = AX
```

[`runtime.mapassign_fast64`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map_fast64.go#L92-L180) 与 [`runtime.mapassign`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L571-L683) 函数的实现差不多，我们需要关注的是后面的三行代码，`24(SP)` 就是该函数返回的值地址，我们通过 `LEAQ` 指令将字符串的地址存储到寄存器 `AX` 中，`MOVQ` 指令将字符串 `"88"` 存储到了目标地址上完成了这次哈希的写入。

### 3.4 扩容

我们在介绍哈希的写入过程时省略了扩容操作，随着哈希表中元素的逐渐增加，哈希的性能会逐渐恶化，所以我们需要更多的桶和更大的内存保证哈希的读写性能：

```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	...
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again
	}
	...
}
```

[`runtime.mapassign`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L571-L683) 函数会在以下两种情况发生时触发哈希的扩容：

1. 装载因子已经超过 6.5；
2. 哈希使用了太多溢出桶；

不过由于 Go 语言哈希的扩容不是一个原子的过程，所以 [`runtime.mapassign`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L571-L683) 函数还需要判断当前哈希是否已经处于扩容状态，避免二次扩容造成混乱。

根据触发的条件不同扩容的方式分成两种，如果这次扩容是溢出的桶太多导致的，那么这次扩容就是等量扩容 `sameSizeGrow`，`sameSizeGrow` 是一种特殊情况下发生的扩容，当我们持续向哈希中插入数据并将它们全部删除时，如果哈希表中的数据量没有超过阈值，就会不断积累溢出桶造成缓慢的内存泄漏。

[runtime: limit the number of map overflow buckets](https://github.com/golang/go/commit/9980b70cb460f27907a003674ab1b9bea24a847c) 引入了 `sameSizeGrow` 通过重用已有的哈希扩容机制，一旦哈希中出现了过多的溢出桶，它就会创建新桶保存数据，垃圾回收会清理老的溢出桶并释放内存[5](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/#fn:5)。

扩容的入口是 [`runtime.hashGrow`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L1017-L1058) 函数：

```go
func hashGrow(t *maptype, h *hmap) {
	bigger := uint8(1)
	if !overLoadFactor(h.count+1, h.B) {
		bigger = 0
		h.flags |= sameSizeGrow
	}
	oldbuckets := h.buckets
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
	h.nevacuate = 0
	h.noverflow = 0

	h.extra.oldoverflow = h.extra.overflow
	h.extra.overflow = nil
	h.extra.nextOverflow = nextOverflow
}
```

哈希在扩容的过程中会通过 [`runtime.makeBucketArray`](https://github.com/golang/go/blob/dcd3b2c173b77d93be1c391e3b5f932e0779fb1f/src/runtime/map.go#L344-L387) 创建一组新桶和预创建的溢出桶，随后将原有的桶数组设置到 `oldbuckets` 上并将新的空桶设置到 `buckets` 上，溢出桶也使用了相同的逻辑进行更新，下图展示了触发扩容后的哈希：

![hashtable-hashgrow](https://img.draveness.me/2019-12-30-15777168478830-hashtable-hashgrow.png)

**图 3-15 哈希表触发扩容**

我们在 [`runtime.hashGrow`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L1017-L1058) 中还看不出来等量扩容和翻倍扩容的太多区别，等量扩容创建的新桶数量只是和旧桶一样，该函数中只是创建了新的桶，并没有对数据进行拷贝和转移，哈希表的数据迁移的过程在是 [`runtime.evacuate`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L1128-L1240) 函数中完成的，它会对传入桶中的元素进行『再分配』。

```go
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
	newbit := h.noldbuckets()
	if !evacuated(b) {
		var xy [2]evacDst
		x := &xy[0]
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
		x.k = add(unsafe.Pointer(x.b), dataOffset)
		x.v = add(x.k, bucketCnt*uintptr(t.keysize))

		y := &xy[1]
		y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
		y.k = add(unsafe.Pointer(y.b), dataOffset)
		y.v = add(y.k, bucketCnt*uintptr(t.keysize))
```

[`runtime.evacuate`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L1128-L1240) 函数会将一个旧桶中的数据分流到两个新桶，所以它会创建两个用于保存分配上下文的 `evacDst` 结构体，这两个结构体分别指向了一个新桶：

![hashtable-evacuate-destination](https://img.draveness.me/2019-12-30-15777168478836-hashtable-evacuate-destination.png)

**图 3-16 哈希表扩容目的**

如果这是一等量扩容，旧桶与新桶之间是一对一的关系，所以两个 `evacDst` 结构体只会初始化一个，当哈希表的容量翻倍时，每个旧桶的元素会都被分流到新创建的两个桶中，我们仔细分析一下分流元素的逻辑：

```go
		for ; b != nil; b = b.overflow(t) {
			k := add(unsafe.Pointer(b), dataOffset)
			v := add(k, bucketCnt*uintptr(t.keysize))
			for i := 0; i < bucketCnt; i, k, v = i+1, add(k, uintptr(t.keysize)), add(v, uintptr(t.valuesize)) {
				top := b.tophash[i]
				k2 := k
				var useY uint8
				hash := t.key.alg.hash(k2, uintptr(h.hash0))
				if hash&newbit != 0 {
					useY = 1
				}
				b.tophash[i] = evacuatedX + useY
				dst := &xy[useY]

				if dst.i == bucketCnt {
					dst.b = h.newoverflow(t, dst.b)
					dst.i = 0
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
					dst.v = add(dst.k, bucketCnt*uintptr(t.keysize))
				}
				dst.b.tophash[dst.i&(bucketCnt-1)] = top
				typedmemmove(t.key, dst.k, k)
				typedmemmove(t.elem, dst.v, v)
				dst.i++
				dst.k = add(dst.k, uintptr(t.keysize))
				dst.v = add(dst.v, uintptr(t.valuesize))
			}
		}
		...
}
```

只使用哈希函数是不能定位到具体某一个桶的，哈希函数只会返回很长的哈希，例如：`b72bfae3f3285244c4732ce457cca823bc189e0b`，我们还需一些方法将哈希映射到具体的桶上，在很多时候我们都会使用取模或者位操作来获取桶的编号，假如当前哈希中包含 4 个桶，那么它的桶掩码就是 `0b11(3)`，使用位操作就会得到 3， 我们就会在 3 号桶中存储该数据：

```ruby
0xb72bfae3f3285244c4732ce457cca823bc189e0b & 0b11 #=> 0
```

如果新的哈希表有 8 个桶，在大多数情况下，原来经过桶掩码 `0b11` 结果为 3 的数据会因为桶掩码增加了一位编程 `0b111` 而分流到新的 3 号和 7 号桶，所有数据也都会被 `typedmemmove` 拷贝到目标桶中：

![hashtable-bucket-evacuate](https://img.draveness.me/2019-12-30-15777168478845-hashtable-bucket-evacuate.png)

**图 3-17 哈希表桶数据的分流**

[`runtime.evacuate`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L1128-L1240) 最后会调用 [`runtime.advanceEvacuationMark`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L1242-L1264) 增加哈希的 `nevacuate` 计数器，在所有的旧桶都被分流后清空哈希的 `oldbuckets` 和 `oldoverflow` 字段：

```go
func advanceEvacuationMark(h *hmap, t *maptype, newbit uintptr) {
	h.nevacuate++
	stop := h.nevacuate + 1024
	if stop > newbit {
		stop = newbit
	}
	for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) {
		h.nevacuate++
	}
	if h.nevacuate == newbit { // newbit == # of oldbuckets
		h.oldbuckets = nil
		if h.extra != nil {
			h.extra.oldoverflow = nil
		}
		h.flags &^= sameSizeGrow
	}
}
```

之前在分析哈希表访问函数 [`runtime.mapaccess1`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L394-L450) 时其实省略了扩容期间获取键值对的逻辑，当哈希表的 `oldbuckets` 存在时，就会先定位到旧桶并在该桶没有被分流时从中获取键值对。

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	...
	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))
	m := bucketMask(h.B)
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			m >>= 1
		}
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
		if !evacuated(oldb) {
			b = oldb
		}
	}
bucketloop:
	...
}
```

因为旧桶中还没有被 [`runtime.evacuate`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L1128-L1240) 函数分流，其中还保存着我们需要使用的数据，会替代新创建的空桶提供数据。

我们在 [`runtime.mapassign`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L571-L683) 函数中也省略了一段逻辑，当哈希表正在处于扩容状态时，每次向哈希表写入值时都会触发 [`runtime.growWork`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L1104-L1113) 对哈希表的内容进行增量拷贝：

```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	...
again:
	bucket := hash & bucketMask(h.B)
	if h.growing() {
		growWork(t, h, bucket)
	}
	...
}
```

当然除了写入操作之外，删除操作也会在哈希表扩容期间触发 [`runtime.growWork`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L1104-L1113)，触发的方式和代码与这里的逻辑几乎完全相同，都是计算当前值所在的桶，然后对该桶中的元素进行拷贝。

我们简单总结一下哈希表的扩容设计和原理，哈希在存储元素过多时会触发扩容操作，每次都会将桶的数量翻倍，整个扩容过程并不是原子的，而是通过 [`runtime.growWork`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L1104-L1113) 增量触发的，在扩容期间访问哈希表时会使用旧桶，向哈希表写入数据时会触发旧桶元素的分流；除了这种正常的扩容之外，为了解决大量写入、删除造成的内存泄漏问题，哈希引入了 `sameSizeGrow` 这一机制，在出现较多溢出桶时会对哈希进行『内存整理』减少对空间的占用。

```
func growWork(t *maptype, h *hmap, bucket uintptr) {
	evacuate(t, h, bucket&h.oldbucketmask())

	// evacuate one more oldbucket to make progress on growing
	if h.growing() {
		 //
		evacuate(t, h, h.nevacuate)
	}
}

func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
	newbit := h.noldbuckets()
	if !evacuated(b){
	
		// 当前 bucket 没有迁移过，进行迁移操作
		....
	}
	// 
	if oldbucket == h.nevacuate {
		advanceEvacuationMark(h, t, newbit)
	}
}

func advanceEvacuationMark(h *hmap, t *maptype, newbit uintptr) {
	h.nevacuate++
	stop := h.nevacuate + 1024
	if stop > newbit {
		stop = newbit
	}
	for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) {
		h.nevacuate++
	}
	if h.nevacuate == newbit { // newbit == # of oldbuckets
		// Growing is all done. Free old main bucket array.
		h.oldbuckets = nil
		// Can discard old overflow buckets as well.
		// If they are still referenced by an iterator,
		// then the iterator holds a pointers to the slice.
		if h.extra != nil {
			h.extra.oldoverflow = nil
		}
		h.flags &^= sameSizeGrow
	}
}

```

1、每次插入或删除数据时，会判断是否需要迁移，进行迁移操作

2、 每次除了迁移当前传入的bucket，还会再进行一次迁移，h.nevacuate 为当前的迁移进度。这里保证了迁移效率，最多产生n次增删行为时, 可以完成迁移。

3、2中的迁移操作会触发调用  advanceEvacuationMark，判断是否完成了迁移

### 3.5 删除

如果想要删除哈希中的元素，就需要使用 Go 语言中的 `delete` 关键字，这个关键的唯一作用就是将某一个键对应的元素从哈希表中删除，无论是该键对应的值是否存在，这个内建的函数都不会返回任何的结果。

![hashtable-delete](https://img.draveness.me/2019-12-30-15777168478853-hashtable-delete.png)

**图 3-18 哈希表删除操作**

在编译期间，`delete` 关键字会被转换成操作为 `ODELETE` 的节点，而 `ODELETE` 会被 [cmd/compile/internal/gc.walkexpr](https://github.com/golang/go/blob/4d5bb9c60905b162da8b767a8a133f6b4edcaa65/src/cmd/compile/internal/gc/walk.go#L439-L1532) 转换成 `mapdelete` 函数簇中的一个，包括 `mapdelete`、`mapdelete_faststr`、`mapdelete_fast32` 和 `mapdelete_fast64`：

```go
func walkexpr(n *Node, init *Nodes) *Node {
	switch n.Op {
	case ODELETE:
		init.AppendNodes(&n.Ninit)
		map_ := n.List.First()
		key := n.List.Second()
		map_ = walkexpr(map_, init)
		key = walkexpr(key, init)

		t := map_.Type
		fast := mapfast(t)
		if fast == mapslow {
			key = nod(OADDR, key, nil)
		}
		n = mkcall1(mapfndel(mapdelete[fast], t), nil, init, typename(t), map_, key)
	}
}
```

这些函数的实现其实差不多，我们来分析其中的 [`runtime.mapdelete`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L685-L791) 函数，哈希表的删除逻辑与写入逻辑非常相似，只是触发哈希的删除需要使用关键字，如果在删除期间遇到了哈希表的扩容，就会对即将操作的桶进行分流，分流结束之后会找到桶中的目标元素完成键值对的删除工作。

```go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
	...
	if h.growing() {
		growWork(t, h, bucket)
	}
	...
search:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break search
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			k2 := k
			if !alg.equal(key, k2) {
				continue
			}
			*(*unsafe.Pointer)(k) = nil
			v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
			*(*unsafe.Pointer)(v) = nil
			b.tophash[i] = emptyOne
			...
		}
	}
}
```

我们其实只需要知道 `delete` 关键字在编译期间经过[类型检查](https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-typecheck/)和[中间代码生成](https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-ir-ssa/)阶段被转换成 [`runtime.mapdelete`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L685-L791) 函数簇中的一员就可以，用于处理删除逻辑的函数与哈希表的 [`runtime.mapassign`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L571-L683) 几乎完全相同，不太需要刻意关注。

### 3.6 遍历

在遍历哈希表时，编译器会使用 [`runtime.mapiterinit`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L797-L844) 和 [`runtime.mapiternext`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L846-L970) 两个运行时函数重写原始的 for/range 循环：

```go
ha := a
hit := hiter(n.Type)
th := hit.Type
mapiterinit(typename(t), ha, &hit)
for ; hit.key != nil; mapiternext(&hit) {
    key := *hit.key
    val := *hit.val
}
```

![map iter loop](https://user-images.githubusercontent.com/7698088/57976471-ad2ebf00-7a13-11e9-8dd8-d7be54f96440.png)

上述代码是 `for key, val := range hash {}` 生成的，在 [`cmd/compile/internal/gc.walkrange`](https://github.com/golang/go/blob/440f7d64048cd94cba669e16fe92137ce6b84073/src/cmd/compile/internal/gc/range.go#L155-L456) 函数处理 `TMAP` 节点时会根据接受 range 返回值的数量在循环体中插入需要的赋值语句：

![golang-range-map](https://img.draveness.me/2020-01-17-15792766877639-golang-range-map.png)

但是，现实并没有这么简单。还记得前面讲过的扩容过程吗？扩容过程不是一个原子的操作，它每次最多只搬运 2 个 bucket，所以如果触发了扩容操作，那么在很长时间里，map 的状态都是处于一个中间态：有些 bucket 已经搬迁到新家，而有些 bucket 还待在老地方。

因此，遍历如果发生在扩容的过程中，就会涉及到遍历新老 bucket 的过程，这是难点所在。

迭代器的结构体定义：

```
type hiter struct {
	// key 指针
	key         unsafe.Pointer
	// value 指针
	value       unsafe.Pointer
	// map 类型，包含如 key size 大小等
	t           *maptype
	// map header
	h           *hmap
	// 初始化时指向的 bucket
	buckets     unsafe.Pointer
	// 当前遍历到的 bmap
	bptr        *bmap
	overflow    [2]*[]*bmap
	// 起始遍历的 bucet 编号
	startBucket uintptr
	// 遍历开始时 cell 的编号（每个 bucket 中有 8 个 cell）
	offset      uint8
	// 是否从头遍历了
	wrapped     bool
	// B 的大小
	B           uint8
	// 指示当前 cell 序号
	i           uint8
	// 指向当前的 bucket
	bucket      uintptr
	// 因为扩容，需要检查的 bucket
	checkBucket uintptr
}
```

`mapiterinit` 就是对 hiter 结构体里的字段进行初始化赋值操作。

前面已经提到过，即使是对一个写死的 map 进行遍历，每次出来的结果也是无序的。下面我们就可以近距离地观察他们的实现了。

```
// 生成随机数 r
r := uintptr(fastrand())
if h.B > 31-bucketCntBits {
	r += uintptr(fastrand()) << 31
}

// 从哪个 bucket 开始遍历
it.startBucket = r & (uintptr(1)<<h.B - 1)
// 从 bucket 的哪个 cell 开始遍历
it.offset = uint8(r >> h.B & (bucketCnt - 1))
```

例如，B = 2，那 `uintptr(1)<<h.B - 1` 结果就是 3，低 8 位为 `0000 0011`，将 r 与之相与，就可以得到一个 `0~3` 的 bucket 序号；bucketCnt - 1 等于 7，低 8 位为 `0000 0111`，将 r 右移 2 位后，与 7 相与，就可以得到一个 `0~7` 号的 cell。

于是，在 `mapiternext` 函数中就会从 it.startBucket 的 it.offset 号的 cell 开始遍历，取出其中的 key 和 value，直到又回到起点 bucket，完成遍历过程。

源码部分比较好看懂，尤其是理解了前面注释的几段代码后，再看这部分代码就没什么压力了。所以，接下来，我将通过图形化的方式讲解整个遍历过程，希望能够清晰易懂。

假设我们有下图所示的一个 map，起始时 B = 1，有两个 bucket，后来触发了扩容（这里不要深究扩容条件，只是一个设定），B 变成 2。并且， 1 号 bucket 中的内容搬迁到了新的 bucket，`1 号`裂变成 `1 号`和 `3 号`；`0 号` bucket 暂未搬迁。老的 bucket 挂在在 `*oldbuckets` 指针上面，新的 bucket 则挂在 `*buckets` 指针上面。

[![map origin](https://user-images.githubusercontent.com/7698088/57978113-f8a79400-7a38-11e9-8e27-3f3ba4fa557f.png)](https://user-images.githubusercontent.com/7698088/57978113-f8a79400-7a38-11e9-8e27-3f3ba4fa557f.png)

这时，我们对此 map 进行遍历。假设经过初始化后，startBucket = 3，offset = 2。于是，遍历的起点将是 3 号 bucket 的 2 号 cell，下面这张图就是开始遍历时的状态：

[![map init](https://user-images.githubusercontent.com/7698088/57980268-a4fa7200-7a5b-11e9-9ad1-fb2b64fe3159.png)](https://user-images.githubusercontent.com/7698088/57980268-a4fa7200-7a5b-11e9-9ad1-fb2b64fe3159.png)

标红的表示起始位置，bucket 遍历顺序为：3 -> 0 -> 1 -> 2。

因为 3 号 bucket 对应老的 1 号 bucket，因此先检查老 1 号 bucket 是否已经被搬迁过。判断方法就是：

```
func evacuated(b *bmap) bool {
	h := b.tophash[0]
	return h > empty && h < minTopHash
}
```

如果 b.tophash[0] 的值在标志值范围内，即在 (0,4) 区间里，说明已经被搬迁过了。

```
empty = 0
evacuatedEmpty = 1
evacuatedX = 2
evacuatedY = 3
minTopHash = 4
```

在本例中，老 1 号 bucket 已经被搬迁过了。所以它的 tophash[0] 值在 (0,4) 范围内，因此只用遍历新的 3 号 bucket。

依次遍历 3 号 bucket 的 cell，这时候会找到第一个非空的 key：元素 e。到这里，mapiternext 函数返回，这时我们的遍历结果仅有一个元素：

[![iter res](https://user-images.githubusercontent.com/7698088/57980302-56010c80-7a5c-11e9-8263-c11ddcec2ecc.png)](https://user-images.githubusercontent.com/7698088/57980302-56010c80-7a5c-11e9-8263-c11ddcec2ecc.png)

由于返回的 key 不为空，所以会继续调用 mapiternext 函数。

继续从上次遍历到的地方往后遍历，从新 3 号 overflow bucket 中找到了元素 f 和 元素 g。

遍历结果集也因此壮大：

[![iter res](https://user-images.githubusercontent.com/7698088/57980349-2d2d4700-7a5d-11e9-819a-a59964f70a7c.png)](https://user-images.githubusercontent.com/7698088/57980349-2d2d4700-7a5d-11e9-819a-a59964f70a7c.png)

新 3 号 bucket 遍历完之后，回到了新 0 号 bucket。0 号 bucket 对应老的 0 号 bucket，经检查，老 0 号 bucket 并未搬迁，因此对新 0 号 bucket 的遍历就改为遍历老 0 号 bucket。那是不是把老 0 号 bucket 中的所有 key 都取出来呢？

并没有这么简单，回忆一下，老 0 号 bucket 在搬迁后将裂变成 2 个 bucket：新 0 号、新 2 号。而我们此时正在遍历的只是新 0 号 bucket（注意，遍历都是遍历的 `*bucket` 指针，也就是所谓的新 buckets）。所以，我们只会取出老 0 号 bucket 中那些在裂变之后，分配到新 0 号 bucket 中的那些 key。

因此，`lowbits == 00` 的将进入遍历结果集：

[![iter res](https://user-images.githubusercontent.com/7698088/57980449-6fa35380-7a5e-11e9-9dbf-86332ea0e215.png)](https://user-images.githubusercontent.com/7698088/57980449-6fa35380-7a5e-11e9-9dbf-86332ea0e215.png)

和之前的流程一样，继续遍历新 1 号 bucket，发现老 1 号 bucket 已经搬迁，只用遍历新 1 号 bucket 中现有的元素就可以了。结果集变成：

[![iter res](https://user-images.githubusercontent.com/7698088/57980487-e8a2ab00-7a5e-11e9-8e47-050437a099fc.png)](https://user-images.githubusercontent.com/7698088/57980487-e8a2ab00-7a5e-11e9-8e47-050437a099fc.png)

继续遍历新 2 号 bucket，它来自老 0 号 bucket，因此需要在老 0 号 bucket 中那些会裂变到新 2 号 bucket 中的 key，也就是 `lowbit == 10` 的那些 key。

这样，遍历结果集变成：

[![iter res](https://user-images.githubusercontent.com/7698088/57980574-ae85d900-7a5f-11e9-8050-ae314a90ee05.png)](https://user-images.githubusercontent.com/7698088/57980574-ae85d900-7a5f-11e9-8050-ae314a90ee05.png)

最后，继续遍历到新 3 号 bucket 时，发现所有的 bucket 都已经遍历完毕，整个迭代过程执行完毕。

顺便说一下，如果碰到 key 是 `math.NaN()` 这种的，处理方式类似。核心还是要看它被分裂后具体落入哪个 bucket。只不过只用看它 top hash 的最低位。如果 top hash 的最低位是 0 ，分配到 X part；如果是 1 ，则分配到 Y part。据此决定是否取出 key，放到遍历结果集里。

map 遍历的核心在于理解 2 倍扩容时，老 bucket 会分裂到 2 个新 bucket 中去。而遍历操作，会按照新 bucket 的序号顺序进行，碰到老 bucket 未搬迁的情况时，要在老 bucket 中找到将来要搬迁到新 bucket 来的 key。



## 4 总结

###  4.1 map 不是线程安全的

在查找、赋值、遍历、删除的过程中都会检测写标志，一旦发现写标志置位（等于1），则直接 panic。赋值和删除函数在检测完写标志是复位之后，先将写标志位置位，才会进行之后的操作。

检测写标志：

```
if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
```

设置写标志：

```
h.flags |= hashWriting
```

可以用 mutex+ map 的方式解决并发问题，或者用官方提供的sync.map

### 4.2 map 遍历结果无序

源码中增加了随机数，会随机到某一个桶，再随机从桶中的一个cell开始遍历

### 4.3 NAN  

nan 可作为key存在map中，并且按key取不到，只能遍历获取

### 4.4  map 缩容

map 没有缩容，仅有等量扩容，期望减少溢出桶。 

### 5 参考文档

[1. go语言设计与实现](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-sync-primitives/#mutex)

[2. go夜读第 44 期 Go 语言 map 源码阅读](https://github.com/qcrao/Go-Questions/tree/master/map)