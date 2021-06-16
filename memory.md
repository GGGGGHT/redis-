# memory

由于c语言并不具备自动内存回收功能，所以Redis在自己的结构体中构建了一个引用计数`refcount` 用以实现内存回收机制，通过这一机制，程序可以通过跟踪对象的引用计数信息，在适当的时候自动释放对象并进行内存回收。
```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */
    int refcount;
    void *ptr;
} robj;


void decrRefCount(robj *o) {
    if (o->refcount <= 0) redisPanic("decrRefCount against refcount <= 0");
    // 初始化时refcount即是1 当refcount再降为1时 需要释放内存 调用相应的free函数
    if (o->refcount == 1) {
        switch(o->type) {
        case REDIS_STRING: freeStringObject(o); break;
        case REDIS_LIST: freeListObject(o); break;
        case REDIS_SET: freeSetObject(o); break;
        case REDIS_ZSET: freeZsetObject(o); break;
        case REDIS_HASH: freeHashObject(o); break;
        default: redisPanic("Unknown object type"); break;
        }
        zfree(o);
    } else {
        o->refcount--;
    }
}
```
### refcount 
- 在创建一个新对象时，引用计数的值会初始化为1
- 当对象被一个新程序使用时，它的引用计数值会被加1
- 当对象不再被一个程序使用时，它的引用计数值会减1


### 对象共享
```c
struct sharedObjectsStruct {
    robj *crlf, *ok, *err, *emptybulk, *czero, *cone, *cnegone, *pong, *space,
    *colon, *nullbulk, *nullmultibulk, *queued,
    *emptymultibulk, *wrongtypeerr, *nokeyerr, *syntaxerr, *sameobjecterr,
    *outofrangeerr, *noscripterr, *loadingerr, *slowscripterr, *bgsaveerr,
    *masterdownerr, *roslaveerr, *execaborterr, *noautherr, *noreplicaserr,
    *busykeyerr, *oomerr, *plus, *messagebulk, *pmessagebulk, *subscribebulk,
    *unsubscribebulk, *psubscribebulk, *punsubscribebulk, *del, *rpop, *lpop,
    *lpush, *emptyscan, *minstring, *maxstring,
    *select[REDIS_SHARED_SELECT_CMDS],
    *integers[REDIS_SHARED_INTEGERS],
    *mbulkhdr[REDIS_SHARED_BULKHDR_LEN], /* "*<value>\r\n" */
    *bulkhdr[REDIS_SHARED_BULKHDR_LEN];  /* "$<value>\r\n" */
};

#define REDIS_SHARED_SELECT_CMDS 10
#define REDIS_SHARED_INTEGERS 10000
#define REDIS_SHARED_BULKHDR_LEN 32

```

`OBJECT IDLETIME`命令可以打印出给定键的空转时长(多长时间没有被读取/写入) 这一空转时间是通过将`当前时间`  - `键的值对象的lru时间计算得出`
键的空转时长还有另外一项作用: 如果服务器打开了maxmemory选项，并且服务器用于回收内存的算法为`volatile-lru`或者`allkeys-lru`,那么当服务器占用的内存超过了`maxmemroy`选项所设置的上限值时，空转时长较高的那部分键会优先被服务器释放，从而回收内存。
