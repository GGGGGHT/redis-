# object

Redis使用对象来表示数据库中的键和值，当我们在Redis的数据识图中新创建一个键值对时，我们至少会创建两个对象，一个对象用作键值对的键，另一个对象用作键值对的值
Redis中每个对象都由一个redisObject结构表示
```c
typedef struct redisObject {
    // 数据的类型
    unsigned type:4;
    // 编码
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    // 引用计数
    int refcount;
    // 指向底层实现数据结构的指针
    void *ptr;
} robj;


Redis的五种类型
/* The actual Redis Object */
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */

Redis类型的编码
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
```

对于Redis的数据库保存的键值对来说，键问题一个字符串对象，而值则可以是五种类型的其中一种
`*ptr`指针指向对象的底层实现数据结构，而这些数据结构由对象的`encoding`属性决定 `encoding`属性记录了对象所使用的编码


---
如果一个字符串对象保存的是整数值，并且这个整数值可以用long类型来表示，那么字符串对象会将整数值保存在字符串对象结构的`ptr`属性里面，将void*转换成long，并将字符串的编码设置为int
```
127.0.0.1:6379> set number 65535
OK
127.0.0.1:6379> debug object number
Value at:0x7f8cf7e9ee20 refcount:1 encoding:int serializedlength:5 lru:13147296 lru_seconds_idle:5
127.0.0.1:6379> object encoding number
"int"
```

---
如果一个字符串对象保存的是一个字符串值，并且这个字符串长度的值小于等于32那么字符串对象的编码是`embstr` 如果长度大于32，则编码是`raw`
```c
/* Create a string object with encoding OBJ_ENCODING_EMBSTR, that is
 * an object where the sds string is actually an unmodifiable string
 * allocated in the same chunk as the object itself. */
robj *createEmbeddedStringObject(const char *ptr, size_t len) {
    robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);
    struct sdshdr8 *sh = (void*)(o+1);

    o->type = OBJ_STRING;
    o->encoding = OBJ_ENCODING_EMBSTR;
    o->ptr = sh+1;
    o->refcount = 1;
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }

    sh->len = len;
    sh->alloc = len;
    sh->flags = SDS_TYPE_8;
    if (ptr == SDS_NOINIT)
        sh->buf[len] = '\0';
    else if (ptr) {
        memcpy(sh->buf,ptr,len);
        sh->buf[len] = '\0';
    } else {
        memset(sh->buf,0,len+1);
    }
    return o;
}
```
embstr编码是专门用于保存短字符串的一种优化编码方式，embstr调用一次内存分配函数来分配一块连接的空间，空间中包含`robj`和`sdshdr`
