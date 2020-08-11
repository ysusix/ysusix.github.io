## 了解内部结构及实现意义
- 理解各种操作的时间复杂度
- 估算各类型值的内存占用

## sds 简单动态字符串
- 特点： 预留额外空间，支持动态扩容
- 应用： 字符串类型值，各种应用字符串的地方
- 源码：
    ```
    // sds.h / sds.c

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

## adlist 双端链表
- 特点： 双端，较灵活；链表，随机访问复杂度高
- 应用： 队列类型值/发布订阅/慢查询？/监视器？
- 源码：
    ```
    // adlist.h / adlist.c

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

## dict 字典
- 特点： 拉链法解决哈希冲突，渐进式扩容
- 应用： 哈希类型值，全局索引
- 源码：
    ```
    // dict.h / dict.c

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
    h = dictHashKey(d, key);
    idx = h & d->ht[table].sizemask;

    // 冲突解决 拉链法
    entry->next = ht->table[index];
    ht->table[index] = entry;

    // 扩展收缩
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        return dictExpand(d, d->ht[0].used*2);
    }
    ...
    minimal = d->ht[0].used;
    if (minimal < DICT_HT_INITIAL_SIZE)
        minimal = DICT_HT_INITIAL_SIZE;
    return dictExpand(d, minimal);

    // 渐进式rehash
    if (dictIsRehashing(d)) _dictRehashStep(d);

    // rehash 过程中从操作
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0]; // 新增
    for (table = 0; table <= 1; table++) { // 查找(可能重复)/删除
    ```

## skiplist 跳表
- 特点： 查找 O(logn) 复杂度，支持范围查找
- 应用： 有序集合类型值
- 源码：
    ```
    // redis.h / t_zset.c

    /* ZSETs use a specialized version of Skiplists */
    // 跳跃表节点
    typedef struct zskiplistNode {
        robj *obj; // 成员对象
        double score; // 分值
        struct zskiplistNode *backward; // 后退指针        
        struct zskiplistLevel { // 层
            struct zskiplistNode *forward; // 前进指针
            unsigned int span; // 跨度
        } level[];
    } zskiplistNode;

    // 跳跃表
    typedef struct zskiplist {
        struct zskiplistNode *header, *tail;  // 表头节点和表尾节点
        unsigned long length; // 表中节点的数量
        int level; // 表中层数最大的节点的层数
    } zskiplist;

    // 有序集合
    typedef struct zset {
        // 字典，键为成员，值为分值
        // 用于支持 O(1) 复杂度的按成员取分值操作
        dict *dict;
        // 跳跃表，按分值排序成员
        // 用于支持平均复杂度为 O(log N) 的按分值定位成员操作
        // 以及范围操作
        zskiplist *zsl;
    } zset;

    ```

## intset 整数集合
- 特点： 节省内存
- 应用： 集合类型值
- 源码：
    ```
    // intset.h / intset.c

    // intset 的编码方式
    #define INTSET_ENC_INT16 (sizeof(int16_t))
    #define INTSET_ENC_INT32 (sizeof(int32_t))
    #define INTSET_ENC_INT64 (sizeof(int64_t))

    typedef struct intset {
        uint32_t encoding; // 编码方式
        uint32_t length; // 集合包含的元素数量
        int8_t contents[]; // 保存元素的数组
    } intset;
    ```

## ziplist 压缩列表
- 特点： 节省内存
- 应用： 列表/哈希表/有序集合类型值
- 源码：
    ```
    // ziplist.h / ziplist.c

    // ziplist 节点信息
    typedef struct zlentry {

        // prevrawlen ：前置节点的长度
        // prevrawlensize ：编码 prevrawlen 所需的字节大小
        unsigned int prevrawlensize, prevrawlen;

        // len ：当前节点值的长度
        // lensize ：编码 len 所需的字节大小
        unsigned int lensize, len;

        // 当前节点 header 的大小
        // 等于 prevrawlensize + lensize
        unsigned int headersize;

        // 当前节点值所使用的编码类型
        unsigned char encoding;

        // 指向当前节点的指针
        unsigned char *p;

    } zlentry;
    ```