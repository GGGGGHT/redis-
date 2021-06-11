```c
struct sdshdr {
    // 记录 buf 数组中已使用字节的数量
    // 等于 SDS 所保存字符串的长度
    int len;

    // 记录 buf 数组中未使用字节的数量
    int free;

    // 字节数组，用于保存字符串
    char buf[];
};
```

![image](https://user-images.githubusercontent.com/26846402/121488141-60a4ea80-ca05-11eb-968a-1fb653b47481.png)

* free属性值为0，表示这个SDS没有分配任务未使用空间
* len属性值为5，表示这个SDS保存了5个字节长的字符串
* buf属性是个char类型的数组，前5个字节保存了`Redis`这5个字符,最后一个字节保存了空字符,空字符的字节空间不计算在SDS的len中,空字符添加到字符串末尾是由sds函数自动完成的。对使用者无感

buf的总长度为`(len + free + 1)`

> 遵循空字符结尾这一惯例的好处是， SDS 可以直接重用一部分 C 字符串函数库里面的函数。

eg: 
```c
printf("%s", s->buf);
```
打印出: Redis



---
Q:Redis为什么使用sds来代替c的字符串？
A:c语言的字符串不能满足Redis对字符串在安全性，效率以及功能方面的要求。
  - c语言字符串不记录自身的长度信息，为了获取字符串长度，需要遍历整个字符串，时间复杂度为O(n)
  - 容易造成缓冲区溢出
  - 内存重分配次数增多
  - 非二进制安全 只能保存文本数据，不能保存像图片，音频，视频，压缩文件这样的二进制数据 
![image](https://user-images.githubusercontent.com/26846402/121624132-8bde1700-caa3-11eb-93b0-b2b71cde531b.png)

![image](https://user-images.githubusercontent.com/26846402/121537243-55b67e00-ca36-11eb-80ed-c5a1f3181612.png)

Q: SDS的优势
A: * O(1)的复杂度获取字符串长度
   * 杜绝缓冲区溢出
   * 减少修改字符串长度时所需的内存重分配次数
   * 二进制安全
   * 兼容部分C字符串函数


<pre>
|---------------|
|len |free |buf |
|---------------|
^          ^
|          | 
|          sds
sdshdr *sh

sh = (void*) (s-(sizeof(struct sdshdr)));
即sds-8获取到sdshdr的位置
</pre>

## 空间预分配和惰性空间释放
```c
sds sdsMakeRoomFor(sds s, size_t addlen) {
    struct sdshdr *sh, *newsh;
    size_t free = sdsavail(s);
    size_t len, newlen;

    // 如果可用大于需要添加 直接返回
    if (free >= addlen) return s;
    len = sdslen(s);
    sh = (void*) (s-(sizeof(struct sdshdr)));
    newlen = (len+addlen);
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;
    newsh = zrealloc(sh, sizeof(struct sdshdr)+newlen+1);
    if (newsh == NULL) return NULL;

    newsh->free = newlen - len;
    return newsh->buf;
}
```

- 空间预分配
  *  如果sds的长度小于1MB 则设置free的长度与len的长度一样 
  *  如果sds的长度大于1MB 则设置free长度为1MB
- 惰性分配
    惰性空间释放用于优化SDS的字符串缩短操作,当SDS的API需要缩短SDS保存的字符串时，程序并不立即使用内存重分配来回收缩短后多出来的字节，而是使用free属性将这些字节的数量记录起来，留待将来使用
    
<br/>  
在扩展SDS空间之前,SDS会先检查未使用空间是否足够,如果足够的话会直接使用未分配的空间，无须执行再分配，通过预分配策略，可以减少分配次数



```c
sds sdsnewlen(const void *init, size_t initlen) {
    struct sdshdr *sh;

    if (init) {
        // 有值时使用malloc +1是为了保存最后一个'\0'
        sh = zmalloc(sizeof(struct sdshdr)+initlen+1);
    } else {
        // 没值时使用calloc
        sh = zcalloc(sizeof(struct sdshdr)+initlen+1);
    }
    if (sh == NULL) return NULL;
    // 初始化len 与 free 
    sh->len = initlen;
    sh->free = 0;
    if (initlen && init)
        // 将init的内容拷贝到buf中 
        memcpy(sh->buf, init, initlen);
    // 将len+1的位置显示的置为'\0'
    sh->buf[initlen] = '\0';
    return (char*)sh->buf;
}
```

## maloc和calloc的区别
malloc和colloc都是库函数用来在运行时动态分配内存
```c
void* malloc(size_t size);
```
malloc: 用来分配指定大小的内存块并且返回一个指向开头的指针，malloc不初始化内存，如果尝试访问初始化之前的内存，将会报错或返回垃圾值

```c
void* calloc(size_t num, size_t size);
```
calloc: 分配内存并且初始化内存块为0值，如果访问块内容将会得到0值
param: 
    * num 要分配的块数
    * size 每块的大小
return: 在 malloc() 和 calloc() 中成功分配后，返回指向内存块的指针，否则返回 NULL 值，表示分配失败。

可以使用malloc() + memset() 来实现calloc()的功能
eg:
```c
ptr = malloc(size);
// 将分配的所有的空间置0
memset(ptr, 0, size);
```

尽量使用malloc而不是calloc因为malloc的效率比calloc的效率更高，除非想初始化0值
