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

