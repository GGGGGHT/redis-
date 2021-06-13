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
