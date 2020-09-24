
## 概览
- 计算机关键部件：CPU / 内存 / IO 控制芯片
- CPU
    - 由 单一总线 发展为 北桥（高速 PCI Bridge） + 南桥（低俗 ISA bridge）
    - CPU 频率达到 4GHz 后，摩尔定律 失效，CPU 向多核心发展
- 分层
    - 计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决，即各种 Interface
    - 应用程序编程接口（Application Programming Interface），如 POSIX API / Win32
    - 系统调用接口（System call Interface），软件中断（Software Interrupt）方式提供
    - 硬件规格（Hardware Specification），生产商提供
- 操作系统：向上提供抽象接口，向下管理硬件资源，挖掘硬件潜力
    - CPU管理：多道程序（粒度大） / 分时系统（主动让出） / 多任务系统（进程/独立地址空间/抢占式）
    - 硬件驱动
- 内存
    - 如何将有限内存分配给多个程序使用？
        - 简单内存分配：地址空间不隔离 / 置换所有数据，内存使用率低 / 程序运行地址不确定
        - 分段：通过引入虚拟地址空间，使地址确定且相互隔离，但粒度仍为进程粒度，内存使用率低
        - 分页：分页 4KB，粒度更小，提高了内存使用率
    - 虚拟页 VP Virtual Page / 物理页 PP Pysical Page / 磁盘页 DP Disk Page
    - CPU - Virtual Address -> MMU - Physical Address -> Pysical Memory
    - MMU Memory Management Unit, 在 CPU 中
- 线程
    - 线程 Thread，即 轻量级进程 LWP Lightweight Process
    - 多线程场景
        - 线程会陷入等待，如等待网络响应
        - 计算时间长，影响用户交互
        - 逻辑需要并发，如多端下载
        - 多核心 CPU
        - 高效率共享数据
    - 线程共享数据： 全局变量 / 堆上数据 / 函数里的静态变量 / 程序代码 / 打开的文件
    - 线程私有数据： 局部变量 / 函数的参数 / TLS数据（Thread Local Storage）
    - 线程调度 Thread Schedule： IO密集型比CPU密集型优先级高，即将饿死优先级高
    - 线程状态： Running / Ready / Waiting
    - Linux 下创建线程： frok / exec / clone
    - 锁： 信号量 / 二元信号量 Binary Semaphore / 互斥量 Mutex / 临界区 Critical Section / 读写锁 Read-Write Lock / 条件变量 Conditoin Variable
    - 可重入： 不依赖单资源锁 / 仅依赖调用方提供参数 / 不使用非const全局变量
    - 过度优化： volatile / barrier
    - 线程模型： 一（Kernel Thread）对一（User Tread） / 一对多 / 多对多


## CPU

- 平均负载
- 使用率
- 中断

## 内存
- 基础
    - 虚拟地址空间： 内核空间（高位 1G/128T） / 用户空间（地位 3G/128T）
    - 内核空间关联相同的物理空间
    - TLB Translation Lookaside Buffer 转译后备缓冲区，MMU 缓存
    - 解决页表项过多： 多级页表 / 大页
- 用户空间（高位到低位）
    - 栈（局部变量和函数调用的上下文等，一般固定 8MB）
    - 文件映射（动态库/共享内存等）
    - 堆（动态分配的内存，从低位向高位增长）
    - 数据段（全局变量等）
    - 只读段（代码/常亮）
- 内存分配
    - C语言使用 malloc 通过 buddy 算法管理内存，对应系统调用 brk / mmap
    - brk 适用小于 128K 的小块内存，分配在堆中，释放后不会立刻归还系统
    - mmap 适用大于 128K 的大块内存，分配在文件映射段
    - 内核空间使用 kmalloc 通过 salb 算法管理小内存
    - 内存只有在首次访问时才会通过缺页异常，由内核来分配，并记录在页表中
    - Buffer 是对磁盘数据的缓存 / Cache 是对文件数据的缓存
- 内存回收
    - 回收缓存，如使用 LRU 算法，回收不常使用的内存页面
    - 把不常使用的内存，交换到磁盘 Swap 分区
    - OOM 杀死进程（消耗内存越大，占用CPU越少，oom_score 越大，/proc/{pid}/oom_adj[-17, 15] 手动调整）
- 工具
    - 查看总体内存 /proc/meminfo / free/top/ps/vmstat（sysstat包）/sar
    - 查看进程内存 /proc/{pid}/smaps / pmap / pidstat
    - 查看缓存命中率 cachestat / cachetop (bcc包)
    - 查看指定文件缓存 pcstat (Go)
    - 查看内存泄漏 memleak （bcc包）/ valgrind
    - 查看slab缓存使用 slabtop
    - 查看进程swap使用 /proc/{pid}/status
    - 缓存清理 echo 3 > /proc/sys/vm/drop_caches 清理文件页/目录页/Inodes等

## 网络

## 存储