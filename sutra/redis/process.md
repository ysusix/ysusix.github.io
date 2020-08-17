
todo aeApiPoll / 合并db
## 文件事件 file event

- Redis 通过套接字与客户端连接，文件事件是服务器对套接字操作的抽象
- Redis 的文件事件处理器使用I/O多路复用程序来同时监听多个套接字
- 套接字执行 accept/read/write/close 等操作时，便会产生文件事件
- 不同类型的文件事件由不同的文件事件处理器处理

### I/O多路复用程序
- I/O多路复用程序将并发的文件事件放到一个有序/同步的队列中
- I/O多路复用程序是通过包装 select/epool/evport/kqueue 等函数库实现

### 文件事件
- 客户端对套接字执行 connect/write/close 操作时，产生 AE_READABLE 事件
- 客户端对套接字执行 read 操作时，产生 AE_WRITABLE 事件
- 对于同一个套接字 AE_READABLE 事件优先于 AE_WRITABLE 事件

### 文件事件处理器
- 连接应答处理器 networking.c/acceptTcpHandle
- 命令请求处理器 networking.c/readQueryFromClient
- 命令回复处理器 networking.c/sendReplyToCient

## 时间事件 time event
- 时间时间是对 serverCron 等需要定时执行操作的抽象

- 时间事件分为 定时事件 和 周期事件，目前只有周期事件
- 时间事件包含 id 事件ID / when 事件到达时间 / timeProc 处理函数 三个属性
- 处理函数可返回 ae.c/AE_NOMORE 或 下次到达间隔毫秒数
- 时间事件通过 time_events 无序链表记录，目前只有 serverCron 一个周期性时间事件

## 事件调度
- 计算下一个时间事件到达的间隔t -> 堵塞t来等待文件事件 -> 处理文件事件/时间事件  
- 时间事件的处理一般会比设定的晚些

## 源码
```
// redis.c

int main(int argc, char **argv) {
    ...
    initServerConfig(); // 初始化配置
    ...
    initServer(); // 初始化 RedisServer 数据结构
    ...
    redisAsciiArt(); // 打印 ASCII LOGO
    ...
    loadDataFromDisk(); // 从 AOF 文件或者 RDB 文件中载入数据
    ...
    // 运行事件处理器，一直到服务器关闭为止
    aeSetBeforeSleepProc(server.el,beforeSleep);
    aeMain(server.el);
    // 服务器关闭，停止事件循环
    aeDeleteEventLoop(server.el);
    return 0;
}

void beforeSleep(struct aeEventLoop *eventLoop) {
    ...
    // 执行一次快速的主动过期检查
    if (server.active_expire_enabled && server.masterhost == NULL)
        activeExpireCycle(ACTIVE_EXPIRE_CYCLE_FAST);
    ...
    // 将 AOF 缓冲区的内容写入到 AOF 文件
    flushAppendOnlyFile(0);
    ...
    // 在进入下个事件循环前，执行一些集群收尾工作
    if (server.cluster_enabled) clusterBeforeSleep();
}

void initServer(void) {
    signal(SIGHUP, SIG_IGN);
    signal(SIGPIPE, SIG_IGN);
    setupSignalHandlers();
    ...
    createSharedObjects();
    adjustOpenFilesLimit();
    // 事件循环数据结构
    server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR);
    ...
    server.db = zmalloc(sizeof(redisDb)*server.dbnum);
    if (server.port != 0 &&
        listenToPort(server.port,server.ipfd,&server.ipfd_count) == REDIS_ERR)
        exit(1);
    ...
    // 创建时间事件
    if(aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
        redisPanic("Can't create the serverCron time event.");
        exit(1);
    }
    // 创建文件事件
    for (j = 0; j < server.ipfd_count; j++) {
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                redisPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
    }
    ...
    if (server.cluster_enabled) clusterInit();
    replicationScriptCacheInit();
    scriptingInit(1);
    slowlogInit();
    bioInit();
}

int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    updateCachedTime(); /* Update the time cache. */
    run_with_period(100) trackOperationsPerSecond(); // 记录执行命令的次数
    server.lruclock = getLRUClock();
    if (zmalloc_used_memory() > server.stat_peak_memory)
        server.stat_peak_memory = zmalloc_used_memory(); // 内存峰值
    server.resident_set_size = zmalloc_get_rss();
    if (server.shutdown_asap) { // 尝试关闭服务器
        if (prepareForShutdown(0) == REDIS_OK) exit(0);
        ...
    }
    clientsCron(); // 检查客户端，关闭超时客户端，并释放客户端多余的缓冲区
    databasesCron(); // 对数据库执行各种操作
    ...
                // 执行 BGSAVE
                rdbSaveBackground(server.rdb_filename);
    ...
                // 执行 BGREWRITEAOF
                rewriteAppendOnlyFileBackground();
    ...
    // 考虑是否需要将 AOF 缓冲区中的内容写入到 AOF 文件中
    if (server.aof_flush_postponed_start) flushAppendOnlyFile(0);
    // 关闭那些需要异步关闭的客户端
    freeClientsInAsyncFreeQueue();
    ...
    server.cronloops++;
    return 1000/server.hz;
}
```

```
// ae.h
// 文件事件结构
typedef struct aeFileEvent {
    int mask; /* one of AE_(READABLE|WRITABLE) */
    aeFileProc *rfileProc;  // 读事件处理器
    aeFileProc *wfileProc; // 写事件处理器
    void *clientData; // 多路复用库的私有数据
} aeFileEvent;

// 时间事件结构
typedef struct aeTimeEvent {
    long long id; /* time event identifier. */
    long when_sec; /* seconds */
    long when_ms; /* milliseconds */
    aeTimeProc *timeProc; // 事件处理函数
    aeEventFinalizerProc *finalizerProc; // 事件释放函数
    void *clientData; // 多路复用库的私有数据
    struct aeTimeEvent *next; // 指向下个时间事件结构，形成链表
} aeTimeEvent;

// 事件处理器的状态
typedef struct aeEventLoop {
    int maxfd;   /* highest file descriptor currently registered */
    int setsize; /* max number of file descriptors tracked */
    long long timeEventNextId; // 用于生成时间事件 id
    time_t lastTime;     // 最后一次执行时间事件的时间
    aeFileEvent *events; // 已注册的文件事件
    aeFiredEvent *fired; // 已就绪的文件事件
    aeTimeEvent *timeEventHead; // 时间事件
    int stop; // 事件处理器的开关
    void *apidata;  // 多路复用库的私有数据
    aeBeforeSleepProc *beforesleep; // 在处理事件前要执行的函数
} aeEventLoop;

```

```
// ae.c

void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {

        // 如果有需要在事件处理前执行的函数，那么运行它
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);

        // 开始处理事件
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}

int aeProcessEvents(aeEventLoop *eventLoop, int flags){
    ...
            tvp->tv_sec = shortest->when_sec - now_sec;
            ...
            if (tvp->tv_sec < 0) tvp->tv_sec = 0;
    ...
        // 处理文件事件，阻塞时间由 tvp 决定
        numevents = aeApiPoll(eventLoop, tvp);
        for (j = 0; j < numevents; j++) {
            // 从已就绪数组中获取事件
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];

            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int rfired = 0;

            // 读事件
            if (fe->mask & mask & AE_READABLE) {
                // rfired 确保读/写事件只能执行其中一个
                rfired = 1;
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            }
            // 写事件
            if (fe->mask & mask & AE_WRITABLE) {
                if (!rfired || fe->wfileProc != fe->rfileProc)
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
            }

            processed++;
        }
    ...
    // 执行时间事件
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);
    
    return processed
}

static int processTimeEvents(aeEventLoop *eventLoop) {
    ...
    te = eventLoop->timeEventHead;
    maxId = eventLoop->timeEventNextId-1;
    while(te) {
        ...
        if (now_sec > te->when_sec ||
            (now_sec == te->when_sec && now_ms >= te->when_ms))
        {
            int retval;

            id = te->id;
            // 执行事件处理器，并获取返回值
            retval = te->timeProc(eventLoop, id, te->clientData);
            processed++;

            // 记录是否有需要循环执行这个事件时间
            if (retval != AE_NOMORE) {
                // 是的， retval 毫秒之后继续执行这个时间事件
                aeAddMillisecondsToNow(retval,&te->when_sec,&te->when_ms);
            } else {
                // 不，将这个事件删除
                aeDeleteTimeEvent(eventLoop, id);
            }

            // 因为执行事件之后，事件列表可能已经被改变了
            // 因此需要将 te 放回表头，继续开始执行事件
            te = eventLoop->timeEventHead;
        } else {
            te = te->next;
        }
    }
    return processed;
}

```