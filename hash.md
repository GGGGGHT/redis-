# hash

定义
```c
哈希表中每个节点的结构 
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;

typedef struct dictType {
    unsigned int (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType;

/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. */
// 哈希表结构 
typedef struct dictht {
    // 哈希表
    dictEntry **table;
    // 哈希表大小
    unsigned long size;
    // 哈希表大小掩码 sizemask = size - 1
    unsigned long sizemask;
    //已有节点数量
    unsigned long used;
} dictht;

typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int iterators; /* number of iterators currently running */
} dict;

/* If safe is set to 1 this is a safe iterator, that means, you can call
 * dictAdd, dictFind, and other functions against the dictionary even while
 * iterating. Otherwise it is a non safe iterator, and only dictNext()
 * should be called while iterating. */
typedef struct dictIterator {
    dict *d;
    long index;
    int table, safe;
    dictEntry *entry, *nextEntry;
    /* unsafe iterator fingerprint for misuse detection. */
    long long fingerprint;
} dictIterator;

/* Create a new hash table */
static dict *dictCreate(dictType *type, void *privDataPtr) {
    dict *ht = malloc(sizeof(*ht));
    _dictInit(ht,type,privDataPtr);
    return ht;
}

/* Initialize the hash table */
static int _dictInit(dict *ht, dictType *type, void *privDataPtr) {
    _dictReset(ht);
    ht->type = type;
    ht->privdata = privDataPtr;
    return DICT_OK;
}

/* Reset an hashtable already initialized with ht_init().
 * NOTE: This function should only called by ht_destroy(). */
static void _dictReset(dict *ht) {
    ht->table = NULL;
    ht->size = 0;
    ht->sizemask = 0;
    ht->used = 0;
}

```


![](https://res.weread.qq.com/wrepub/epub_622000_28)


#Rehash
```c
int dictRehash(dict *d, int n) {
    // 一次只处理10n
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    if (!dictIsRehashing(d)) return 0;
    // ht[0]非空
    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        // 将所有的key从旧桶中移动到新桶
        while(de) {
            uint64_t h;

            nextde = de->next;
            /* Get the index in the new hash table */
            // 找到在新桶中的位置 
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table... */
    // 如果rehash过了 释放ht[0],并将ht[0]指向ht[1],重新申请一个新的ht[1] 供下次rehash使用
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```

rehash的目的: 为了让哈希表的负载因子维持在一个合理的范围之内
何时触发rehash: redis会触发`后台`操作,如`关闭超时连接`,`key过期`,`重新分配大小`,`rehash`.  跟配置文件中的<b>hz</b>参数配置有关 默认是10 范围在1-500内,并不是上述所有的操作都是同频率执行,redis会检查`hz`的值 当`hz`参数被增大时,会更加消耗CPU资源,当有许多键同时过期时可以提高响应率


# API
```c
// 新建hash表
dict *dictCreate(dictType *type, void *privDataPtr)
// 添加元素到hash表中
int dictAdd(dict *d, void *key, void *val) 

```
