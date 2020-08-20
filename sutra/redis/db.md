
## 单机数据库设计实现

### 源码
```
// 服务端对象
struct redisServer {
    char *configfile;   // 配置文件的绝对路径
    int hz;             // serverCron() 每秒调用的次数
    redisDb *db;        // 数据库数组
    int dbnum;          // 数据库数量
    ...
    dict *commands; // 命令表
    ...
    long long stat_keyspace_hits; // 成功查找键的次数
    long long stat_keyspace_misses; // 查找键失败的次数
    ...
    long long dirty; // 自从上次 SAVE 执行以来，数据库被修改的次数
    ...
    list *clients; // 客户端链表
    redisClient *lua_client; // Lua 执行客户端
}

// DB对象
typedef struct redisDb {
    int id;                     /* Database ID */
    dict *dict;                 /* The keyspace for this DB */
    dict *expires;              /* Timeout of keys with a timeout set */
    ...
} redisDb;

// 客户端对象
typedef struct redisClient {
    // 描述/状态
    int fd;         // 套接字描述符
    redisDb *db;    // 当前正在使用的数据库
    int dictid;     // 当前正在使用的数据库的 id （号码）
    robj *name;     // 客户端的名字

    time_t ctime;   // 客户端创建时间
    time_t lastinteraction; // 客户端最后一次和服务器互动的时间
    time_t obuf_soft_limit_reached_time; // 客户端的输出缓冲区超过软性限制的时间
    int flags;             // 客户端状态标志 REDIS_SLAVE | REDIS_MONITOR | REDIS_MULTI ...
    int authenticated;     // 0 代表未认证， 1 代表已认证

    // 请求
    sds querybuf;   // 查询缓冲区
    size_t querybuf_peak;   // 查询缓冲区长度峰值

    int argc;       // 参数数量
    robj **argv;    // 参数对象数组
    struct redisCommand *cmd, *lastcmd; // 记录被客户端执行的命令
    int reqtype; // 请求的类型：内联命令还是多条命令
    int multibulklen;// 剩余未读取的命令内容数量 
    long bulklen;   // 命令内容的长度

    // 回复
    list *reply;    // 回复链表
    unsigned long reply_bytes; // 回复链表中对象的总大小
    int sentlen;    // 已发送字节

    int bufpos; // 回复偏移量
    char buf[REDIS_REPLY_CHUNK_BYTES]; // 回复缓冲区

    // 主从/事务/堵塞/监视/消息队列
    ...
} redisClient;


// 命令对象
struct redisCommand {
    char *name;     // 命令名字
    redisCommandProc *proc; // 实现函数
    int arity;      // 参数个数
    char *sflags;   // 字符串表示的 FLAG
    int flags;      // 实际 FLAG
    ...
    long long microseconds, calls; // 执行耗费的总毫微秒数,总次数
};

// 命令标志
#define REDIS_CMD_WRITE 1                   /* "w" flag */
#define REDIS_CMD_READONLY 2                /* "r" flag */
#define REDIS_CMD_DENYOOM 4                 /* "m" flag */
#define REDIS_CMD_NOT_USED_1 8              /* no longer used flag */
#define REDIS_CMD_ADMIN 16                  /* "a" flag */
#define REDIS_CMD_PUBSUB 32                 /* "p" flag */
#define REDIS_CMD_NOSCRIPT  64              /* "s" flag */
#define REDIS_CMD_RANDOM 128                /* "R" flag */
#define REDIS_CMD_SORT_FOR_SCRIPT 256       /* "S" flag */
#define REDIS_CMD_LOADING 512               /* "l" flag */
#define REDIS_CMD_STALE 1024                /* "t" flag */
#define REDIS_CMD_SKIP_MONITOR 2048         /* "M" flag */
#define REDIS_CMD_ASKING 4096               /* "k" flag */

```

### 数据库
- 数据库定义/切换库
- 键空间 / 添加/删除/更新/查询键等
    - 更新 server 对象 hit/miss/dirty 个数
    - 更新 object 对象 lru 时间
    - 删除过期 key
    - watch 期间对键标记 dirty
    - 更新 server 对象 
    - 通知从库
    - ...
- 键过期 / 设置/移除/查询过期时间
    - 删除策略
        - 惰性删除 db.c/expireIfNeeded
        - 定期删除 redis.c/activeExpireCycle 随机检查
    - 持久化策略
        - RDB 不存过期键 / 载入时忽略过期键
        - AOF 追加 DEL 命令 / 重写时忽略过期键
    - 从服务器不处理过期键，等待主库 DEL 命令
- 数据库通知 消息订阅模式

### 客户端
- 套接字 -1 伪客户端，AOF 文件或 Lua 脚本，>-1 正常客户端
- 标志 REDIS_MASTER/REDIS_SLAVE 等
- 输入缓冲区 大于 1GB 会关闭连接
- 命令与命令参数/命令实现函数
- 输出缓冲区 固定大小 16KB 存储长度较小的回复 / 可变大小存储长回复
- 身份验证
- 时间
- 创建 主动连接
- 关闭 
    - 主动关闭
    - 客户端进程退出或被杀死导致网络连接断开
    - 数据不符合协议格式
    - 超时
    - 请求数据超过1GB限制
    - 回复数据超过限制（硬性/软性限制）client-output-buffer-limit normal/slave/pubsub 256mb 64mb 60 

### 服务端
- 启动过程
    - 初始化服务器状态结构 
        - 运行ID/运行频率/配置文件路径/架构/端口号/命令表等
    - 载入配置
    - 初始化服务器数据结构
        - clients/db/订阅/Lua/慢日志等
        - 信号处理器/共享对象/监听端口/AOF/IO模块
        - 为 ServerCron 创建时间事件
    - 还原数据库 AOF/RDB
    - 执行事件循环
- 命令执行过程
    - 查找命令实现
    - 执行预备操作 
        - 检查参数个数/授权状态/剩余内存
        - 检查服务端状态（BGSAVE出错/载入中/Lua脚本）
        - 检查客户端状态（SUB/事务）
        - 发给监视器
    - 调用命令
    - 后续
        - 慢日志记录
        - 更新 redisCommand 结构
        - AOF写入
        - 主动同步
- ServerCron 100ms执行一次
    - 更新时间/LRU时间/每秒执行命令次数（估算）/内存峰值等状态/统计数据
    - 处理 SIGTERM 信号，优雅关闭
    - 管理客户端资源 超时检查/清理缓冲区/关闭缓存超出客户端
    - 管理数据库资源 删除过期键/收缩字典
    - 执行被延迟的 BGREWRITEAOF
    - 检查持久化操作的运行状态 BGSAVE/BGREWRITEAOF
    - AOF 缓存持久化

### key操作
  - 基本操作： del / exists / type
  - 批量操作： keys / scans
  - 扩展操作： randomkey / rename[|nx] / move
  - 排序操作： sort [|by] [|limit] [|get] [|asc|desc] [|alpha] [|store]
  - 过期操作： [|p]expire[|at] / [|p]ttl / persist

### 其他操作
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

