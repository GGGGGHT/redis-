```c
typedef struct zskiplistNode {
    // 成员对象
    robj *obj;
    // 分值
    double score;
    // 后退指针
    struct zskiplistNode *backward;
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        unsigned int span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    // 头 尾
    struct zskiplistNode *header, *tail;
    // 长度
    unsigned long length;
    // 层数
    int level;
} zskiplist;

typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;
```

跳表的level数组可以包含多个元素，每个元素都包含一个指向其他节点的指针，程序可以通过这些层来回忆访问其他节点的速度，
level[i].span用于记录两个节点之前的跨度，两个节点之间的跨度越大，它们相距的就越远

![zskiplist](https://res.weread.qq.com/wrepub/epub_622000_45)


# API
```c
创建一个新的跳表    O(1)
zskiplist *zslCreate(void) 
释放给定的跳表以及表中包含的所有节点 O(N) n为跳表的长度
void zslFreeNode(zskiplistNode *node)
将给定的成员插入跳表中 最坏O(N)
zskiplistNode *zslInsert(zskiplist *zsl, double score, robj *obj)

---
zset的编码可以是`OBJ_ENCODING_ZIPLIST`或者`OBJ_ENCODING_SKIPLIST`
