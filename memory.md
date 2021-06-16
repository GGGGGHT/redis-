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
refcount 
- 在创建一个新对象时，引用计数的值会初始化为1
- 当对象被一个新程序使用时，它的引用计数值会被加1
- 当对象不再被一个程序使用时，它的引用计数值会减1

