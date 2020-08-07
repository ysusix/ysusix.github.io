## 了解内部结构及实现意义
- 理解各种操作的时间复杂度
- 估算各类型值的内存占用

## sds 简单动态字符串
- 特点： 预留额外空间，支持动态扩容
- 应用： 字符串类型值
- 源码：
    ```
    // 结构体声明
    struct sdshdr {
        int len; // buf 中已占用空间的长度
        int free; // buf 中剩余可用空间的长度
        char buf[]; // 数据空间
    };
    // 动态扩容规则
    if (newlen < SDS_MAX_PREALLOC)
        // 如果新长度小于 SDS_MAX_PREALLOC 
        // 那么为它分配两倍于所需长度的空间
        newlen *= 2;
    else
        // 否则，分配长度为目前长度加上 SDS_MAX_PREALLOC
        newlen += SDS_MAX_PREALLOC;
    ```
- 操作：
    - 基本操作： set / setnx / setex / psetex / get / getset
    - 批量操作： mset / msetnx / mget
    - 字符串扩展操作： strlen / append / setrange / getrange
    - 数字扩展操作： incr / incrby / incrbyfloat / decr / decrby

## adlist 双端链表
- 特点： 双端，较灵活；链表，随机访问复杂度高
- 应用： 队列类型值/发布订阅/慢查询？/监视器？
- 源码：
    ```
    // 链表
    typedef struct list {
        listNode *head; // 表头节点
        listNode *tail; // 表尾节点 
        unsigned long len; // 链表所包含的节点数量
        ...
    } list;
    // 节点
    typedef struct listNode {
        struct listNode *prev; // 前置节点
        struct listNode *next; // 后置节点
        void *value; // 节点的值
    } listNode;
    ```
- 操作：
    - 基本操作： lpush  / rpush / lpop / rpop / llen / rpoplpush
    - 扩展操作： lpushx / rpushx
    - 随机操作： lrange / lindex / linsert / lset / lrem / ltrim
    - 堵塞操作： blpop / brpop / brpoplpush

## dict 字典
- 特点： 拉链法解决哈希冲突，渐进式扩容
- 应用： 哈希类型值，全局索引
- 源码：
    ```
    // 字典
    typedef struct dict {
        ...
        dictht ht[2]; // 两个哈希表
        int rehashidx; // rehash 索引。当 rehash 不在进行时，值为 -1
        ...
    } dict;
    // 哈希表
    typedef struct dictht {
        dictEntry **table; // 哈希表数组
        unsigned long size; // 哈希表大小
        unsigned long sizemask; // 哈希表大小掩码，用于计算索引值。总是等于 size - 1
        unsigned long used; // 该哈希表已有节点的数量
    } dictht;
    // 哈希表节点
    typedef struct dictEntry {
        void *key; // 键
        union { // 值
            void *val;
            uint64_t u64;
            int64_t s64;
        } v;
        struct dictEntry *next; // 指向下个哈希表节点，形成链表
    } dictEntry;
    // 哈希算法 MurmurHash2

    // 冲突解决 拉链法
    
    // 扩展收缩

    // 渐进式rehash
    ```
- 操作：
    - 基本操作： hset / hsetnx / hget / hexists / hdel / hlen
    - 批量操作： hmset / hmget / hkeys / hvals / hgetall / hscan
    - 扩展操作： hstrlen / hincrby / hincrbyfloat
