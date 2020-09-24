## Redis 学习路径

Redis：Remote dictionary service.

### 两大纬度/三大主线
- 应用纬度： 缓存/锁/队列等
- 系统纬度： CPU/内存/网络/存储，设计原理/规范，epoll网络模型/run-to-complete模型等
- 高性能主线：包括线程模型、数据结构、持久化、网络框架
- 高可靠主线：主从复制、哨兵机制
- 高可扩展主线： 集群、数据分片、负载均衡

![redis01](https://static001.geekbang.org/resource/image/79/e7/79da7093ed998a99d9abe91e610b74e7.jpg)


### 基础知识
- 数据结构及操作
    - string 字符串
        - 底层：string 简单动态字符串
        - 操作：get/get/del/expire / scan
    - list 队列
        - 底层：ziplist 压缩列表 / DoublyLinkedList 双向链表
        - 操作：lpop/rpop/lpush/rpush/llen
    - hash 哈希表
        - 底层：ziplist 压缩列表 / hash 哈希表
        - 操作：hget/hset/hdel / hmget/hmset / hgetall/hscan
    - set 集合
        - 底层：array 整数数组 / hash 哈希表
        - 操作：sadd/srem/srandmember/scard / smembers/sscan
    - sorted set / zset 有序集合
        - 底层：ziplist 压缩列表 / skiptable 跳表
        - 操作：zscan
    - 扩展： hyperlog/bitmap/bloomfilter/geohash/stream
- 使用：
    - connect/pconnect/pipeline
    - 发布订阅/lua脚本/事务/排序/二进制位数组/慢查询/监视器
- 高性能
    - 网络模型：请求响应流程/多路复用
    - 线程模型：文件事件/时间事件
    - 内存模型：内存分配器/索引/ hash冲突/rehash过程 / lru
    - 存储模型：RDB/AOF日志
- 高可靠
    - 主从一致性
    - 哨兵机制
    - CAP / 共识算法
- 高可扩展
    - 集群
    - 数据分片
    - 负载均衡

![底层数据结构](https://static001.geekbang.org/resource/image/82/01/8219f7yy651e566d47cc9f661b399f01.jpg)

### 应用场景/典型案例/项目应用
- 缓存
- 分布式锁
- 消息队列
- 延时队列
- 时间序列数据库

### 常见问题/复现/调试
- 应用
    - 缓存刷新策略
    - 缓存击穿/缓存穿透/缓存雪崩/缓存污染
    - 锁实现
- 存储
    - 主从不一致
    - 数据丢失
    - 性能抖动/堵塞/响应延迟/bigkey
- 网络
    - 多实例异常丢包
- 内存
    - 内存占用高
    - 主从同步和AOF的内存竞争
- CPU
    - 不同操作时间复杂度
    - 跨CPU核心访问
- 面试问题
    - Redis如何做持久化
    - 集群方案怎么做
    - 为何使用 ziplist？ CPU缓存友好/内存紧凑

![Redis问题画像图](https://static001.geekbang.org/resource/image/70/b4/70a5bc1ddc9e3579a2fcb8a5d44118b4.jpeg)

### 相关工具
- telnet/netstat
- tcpflow/tcpdump
- 监控等

### 学习资料
- 源码 https://github.com/antirez/redis 
- 中文注释源码 git@github.com:huangz1990/redis-3.0-annotated.git 
- 书 Redis设计与实现 https://drive.google.com/drive/folders/1AwI658M0ysaUcu7X_Ms_LfkzFG8nJenX?usp=sharing
- 书 Redis开发与运维
- 书 Redis深度历险：核心原理与应用实践
- 书 Redis使用手册 微信读书
- 专栏 Redis核心技术与实战 https://time.geekbang.org/column/intro/329
- 官方文档 https://redis.io/commands/
- 