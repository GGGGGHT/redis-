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
