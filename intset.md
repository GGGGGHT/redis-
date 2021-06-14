# intset

`intset`是redis用于保存整数值的集合抽象数据结构，它可以保存类型为int64_t,int32_t,int16_t的整数值，并保证集合中不会出现重复元素

```c
typedef struct intset {
    // 编码方式
    uint32_t encoding;
    // 集合包含的元素数量
    uint32_t length;
    // 保存元素的数组
    int8_t contents[];
} intset;

typedef long long               int64_t;
typedef int                     int32_t;
typedef short                   int16_t;

#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))
```
contents数组是整数集合的底层实现，整数集合的每个元素都是contents数组的一个数组项，各个项在数组中按值的大小从小到大有序地排列，关助教数组中不包含任何重复项
length属性记录了整数集合你敢信的元素数量，也即是contents数组的长度
intest结构将contents属性声明为int8_t类型的数组，但实际上contents数组并不保存任何int8_t类型的值，contents数组的真正类型取决于encoding属性的值


![包含5个int6_t类型的整数值的整数集合](https://res.weread.qq.com/wrepub/epub_622000_55)

# 结构升级

当我们要将一个新元素添加到整数集合里面，并且新元素的类型比整数集合现在元素的类型都要长时，整数集合需要先进行升级，然后才能将新元素添加到整数集合里面
```c
/* Insert an integer in the intset */
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    // 获取要插入值的编码
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 1;

    /* Upgrade encoding if necessary. If we need to upgrade, we know that
     * this value should be either appended (if > 0) or prepended (if < 0),
     * because it lies outside the range of existing values. */
    // 判断是否需要升级 如果需要升级则要插入的值可能在最前面或最后面，因为它的范围超过了当前的所有值
    if (valenc > intrev32ifbe(is->encoding)) {
        /* This always succeeds, so we don't need to curry *success. */
        return intsetUpgradeAndAdd(is,value);
    } else {
        /* Abort if the value is already present in the set.
         * This call will populate "pos" with the right position to insert
         * the value when it cannot be found. */
        // 如果集合中已经出现 则退出 如果没有出现则pos保存的即是新元素存储的位置
        if (intsetSearch(is,value,&pos)) {
            if (success) *success = 0;
            return is;
        }

        // 调整大小
        is = intsetResize(is,intrev32ifbe(is->length)+1);
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }

    // 将value保存到intset中
    _intsetSet(is,pos,value);
    // lenght + 1
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}

// 升级并插入新元素
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    // 获得当前的编码
    uint8_t curenc = intrev32ifbe(is->encoding);
    // 获得新的编码
    uint8_t newenc = _intsetValueEncoding(value);
    // 得到长度
    int length = intrev32ifbe(is->length);
    // 判断是追加还是在最前方插入 为了之后维护使用
    int prepend = value < 0 ? 1 : 0;

    /* First set new encoding and resize */
    // 设置新的编码
    is->encoding = intrev32ifbe(newenc);
    is = intsetResize(is,intrev32ifbe(is->length)+1);

    /* Upgrade back-to-front so we don't overwrite values.
     * Note that the "prepend" variable is used to make sure we have an empty
     * space at either the beginning or the end of the intset. */
     // 从后往前遍历
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    /* Set the value at the beginning or the end. */
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

使用intset的优势：
 - 提升灵活性 可以通过自动升级底层数组来适应新元素
 - 节省内存 当intset中只有int16_t的值时，数组则使用int16_t保存，这样做法既可以让集合能同时保存三种不同类型的值，又可以确保升级操作只会在有需要的时候进行，这可以尽量节省内存。
