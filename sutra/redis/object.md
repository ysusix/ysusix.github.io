## 源码
```
// redis.h / object.c
// t_string.c / t_list.c / t_hash.c / t_set.c / t_zset.c

// 对象
typedef struct redisObject {
    unsigned type:4; // 类型
    unsigned encoding:4; // 编码
    unsigned lru:REDIS_LRU_BITS; // 对象最后一次被访问的时间
    int refcount; // 引用计数
    void *ptr; // 指向实际值的指针
} robj;

// 对象类型
#define REDIS_STRING 0
#define REDIS_LIST 1
#define REDIS_SET 2
#define REDIS_ZSET 3
#define REDIS_HASH 4

// 对象编码
#define REDIS_ENCODING_RAW 0     /* Raw representation */
#define REDIS_ENCODING_INT 1     /* Encoded as integer */
#define REDIS_ENCODING_HT 2      /* Encoded as hash table */
#define REDIS_ENCODING_ZIPMAP 3  /* Encoded as zipmap 弃用*/
#define REDIS_ENCODING_LINKEDLIST 4 /* Encoded as regular linked list */
#define REDIS_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define REDIS_ENCODING_INTSET 6  /* Encoded as intset */
#define REDIS_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define REDIS_ENCODING_EMBSTR 8  /* Embedded sds string encoding */

// 压缩编码默认边界
#define REDIS_HASH_MAX_ZIPLIST_ENTRIES 512
#define REDIS_HASH_MAX_ZIPLIST_VALUE 64
#define REDIS_LIST_MAX_ZIPLIST_ENTRIES 512
#define REDIS_LIST_MAX_ZIPLIST_VALUE 64
#define REDIS_SET_MAX_INTSET_ENTRIES 512
#define REDIS_ZSET_MAX_ZIPLIST_ENTRIES 128
#define REDIS_ZSET_MAX_ZIPLIST_VALUE 64

#define REDIS_ENCODING_EMBSTR_SIZE_LIMIT 39

```

## string 字符串
  - 编码： int / embstr / raw
  - 源码：
  ```
  if (len <= 21 && string2l(s,len,&value)) {
    ...
    o->encoding = REDIS_ENCODING_INT;
    o->ptr = (void*) value;
    ...
  if (len <= REDIS_ENCODING_EMBSTR_SIZE_LIMIT) {
    ...
    emb = createEmbeddedStringObject(s,sdslen(s));
  ...

  ```
  - 操作：
    - 基本操作： set[|nx|xx|ex|px] / psetex / get / getset
    - 批量操作： mset[|nx] / mget
    - 字符串扩展操作： strlen / append / setrange / getrange
    - 数字扩展操作： incr[|by[|float]] / decr[|by]

## list 列表
  - 编码： ziplsit / linkedlist
  - 源码：
  ```
  if (sdsEncodedObject(value) &&
        sdslen(value->ptr) > server.list_max_ziplist_value)
            listTypeConvert(subject,REDIS_ENCODING_LINKEDLIST);
  ...
  if (subject->encoding == REDIS_ENCODING_ZIPLIST &&
    ziplistLen(subject->ptr) >= server.list_max_ziplist_entries)
        listTypeConvert(subject,REDIS_ENCODING_LINKEDLIST);
  ```
  - 操作：
    - 基本操作： [l|r][push[|x]|pop] / llen / rpoplpush
    - 随机操作： lrange / lindex / linsert / lset / lrem / ltrim
    - 堵塞操作： b[l|r]pop / brpoplpush

## hash 哈希表
  - 编码： ziplist / hashtable
  - 操作：
    - 基本操作： hset[|nx] / hget / hexists / hdel / hlen
    - 批量操作： hmset / hmget / hkeys / hvals / hgetall / hscan
    - 扩展操作： hstrlen / hincrby[|float]

## set 集合
  - 编码： setint / hashtable
  - 源码：
  ```
  if (isObjectRepresentableAsLongLong(value,&llval) == REDIS_OK) {
  ...
  } else {
      ...
      setTypeConvert(subject,REDIS_ENCODING_HT);
      ...
  }
  ...
  if (intsetLen(subject->ptr) > server.set_max_intset_entries)
      setTypeConvert(subject,REDIS_ENCODING_HT);
  ```
  - 操作：
    - 基本操作： sadd / srem / sismember / scard / spop
    - 批量操作： smembers / srandmember / sscan
    - 集合操作： smove / s[inter|union|diff][|store]

## zset / sortedset 有序集合
  - 编码： ziplist / skiplist
  - 操作：
    - 基本操作： zadd / zrem / zcard / zscore / z[|rev]rank
    - 批量查询： zrange[|byscore|bylex] / z[rev]range[byscore] / zscan
    - 批量删除： zremrangeby[rank|score|lex]
    - 扩展操作： zincreby / zlexcount / zcount
    - 集合操作： z[union|inter]store

## HyperLogLog
  - 操作： pfadd / pfcount / pfmerge

## geo 地理位置
  - 操作： geoadd / geopos / geodist / georadius[|bymember] / geohash

## bitmap 位图
  - 操作： 
    - setbit / getbit / bitcount / bitpos
    - bitop [and|or|xor|not]
    - bitfield [get|set|increby|overflow [wrap|sat|fail]]


## key
  - 基本操作： del / exists / type
  - 批量操作： keys / scans
  - 扩展操作： randomkey / rename[|nx] / move
  - 排序操作： sort [|by] [|limit] [|get] [|asc|desc] [|alpha] [|store]
  - 过期操作： [|p]expire[|at] / [|p]ttl / persist

## 全局
  - DB： select / dbsize / flushdb / flushall / swapdb
  - 事务： multi / exec / discard / watch / unwatch
  - 脚本： eval / evalsha / script_[load|exists|flush|kill]
  - 持久化： save / bgsave / bgrewriteaof / lastsave
  - 发布订阅： publish / [|p][|un]subscribe / pubsub
  - 复制： slaveof / role
  - 客户端与服务端： auth / quit / info / shutdown / time / client_[getname|kill|list|setname]
  - 配置： config_[set|get|resetstat|rewrite]
  - 调试： ping / echo / object / slowlog / monitor / debug_[object|segfault]
  - 内部命令： migrate / dump / restore / sync / psync
