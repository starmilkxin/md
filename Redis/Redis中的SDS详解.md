[原文地址：面试杀手锏：Redis源码之SDS](https://mp.weixin.qq.com/s/uYUQ1P8Dq1Cdknxif7lF-g)

# 数据结构
首先是SDS（Simple Dynamic String）的数据结构。Redis6.x与Redis3.x的数据结构有着一些不同。
+ Redis3.x

```C
struct sdshdr {
    //记录buf数组中已使用字节的数量
    //等于SDS所保存字符串的长度
    unsigned int len;

    //记录buf数组中未使用字节的数量
    unsigned int free;

    //char数组，用于保存字符串
    char buf[];
};
```

+ Redis6.x
```C
// 注意：sdshdr5从未被使用，Redis中只是访问flags。
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 低3位存储类型, 高5位存储长度 */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* 已使用 */
    uint8_t alloc; /* 总长度，用1字节存储 */
    unsigned char flags用来存储header的类型; /* 低3位存储类型, 高5位预留 */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* 已使用 */
    uint16_t alloc; /* 总长度，用2字节存储 */
    unsigned char flags; /* 低3位存储类型, 高5位预留 */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* 已使用 */
    uint32_t alloc; /* 总长度，用4字节存储 */
    unsigned char flags; /* 低3位存储类型, 高5位预留 */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* 已使用 */
    uint64_t alloc; /* 总长度，用8字节存储 */
    unsigned char flags; /* 低3位存储类型, 高5位预留 */
    char buf[];
};
```

![SDS数据结构](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/无标题.png)
+ len表示已使用的长度。注意，SDS中的buf是char*类型的（为了能够使用C的api），所以其会最后默认加上'\0'。len并不包括最后的'\0'。
+ free表示剩余的空间大小。
+ buf为char类型的数组。

Redis6.x对于SDS的改进，主要在于有五种类型的header
+ len表示已使用的长度
+ alloc表示总长度，分别用1，2，4，8字节来存储，sdshdr5因为表示字符串小于一个字节，所以压根不存储
+ flags用来标识类型，用一个char来存储，因为一共五种类型，所以三位便可以标识，剩下的五位待用。

关于uintxx_t的解释

```C
typedef unsigned char uint8_t;
typedef unsigned short uint16_t;
typedef unsigned int uint32_t;
typedef unsigned long long uint64_t;
```

# 非对齐填充
在 Redis6.x 的源码中 SDS 的结构体为struct \__attribute\__ ((\__packed\__))与struct有较大的差别，这其实和我们熟知的对齐填充有关。<br/>
<br/>
我们知道现代计算机中，内存空间按照字节划分，理论上可以从任何起始地址访问任意类型的变量。<br/>
但实际中在访问特定类型变量时经常在特定的内存地址访问，这就需要各种类型数据按照一定的规则在空间上排列，而不是顺序一个接一个地存放，这就是对齐。<br/>
<br/>
为什么需要对齐填充是由于各个硬件平台对存储空间的处理上有很大的不同。<br/>
<br/>
一些平台对某些特定类型的数据只能从某些特定地址开始存取。最常见的是如果不按照适合其平台的要求对数据存放进行对齐，会在存取效率上带来损失。<br/>
<br/>
比如有些平台每次读都是从偶地址开始，如果一个int型（假设为 32位）存放在偶地址开始的地方，那么一个读周期就可以读出，而如果存放在奇地址开始的地方，就可能会需要2个读周期，并对两次读出的结果的高低字节进行拼凑才能得到该int数据，导致在读取效率上下降很多。<br/>
<br/>
更改对齐方式
+ 使用伪指令#pragma pack(n)：C编译器将按照n个字节对齐；
+ 使用伪指令#pragma pack()：取消自定义字节对齐方式。

另外，还有如下的一种方式(GCC特有语法)：
+ \__attribute((aligned (n)))：让所作用的结构成员对齐在n字节自然边界上。如果结构体中有成员的长度大于n，则按照最大成员的长度来对齐。
+ \__attribute__ ((packed))：取消结构在编译过程中的优化对齐，按照实际占用字节数进行对齐。

![字节对齐](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220220212111.png)
所以Redis就是取消了字节对齐，而原因呢就是：
1. SDS 的指针并不是指向 SDS 的起始位置（len位置），而是直接指向buf[]，使得 SDS 可以直接使用 C 语言string.h库中的某些函数，做到了兼容。
2. 如果不进行对齐填充，那么在获取当前 SDS 的类型时则只需要后退一步即可flagsPointer = ((unsigned char\*)s)-1；相反，若进行对齐填充，由于 Padding 的存在，我们在不同的系统中不知道退多少才能获得flags，并且我们也不能将 sds 的指针指向flags，这样就无法兼容 C 语言的函数了，也不知道前进多少才能得到 buf[]。

# SDS优势
## O(1)时间复杂度获取字符串长度
由于C字符串不记录自身的长度，所以为了获取一个字符串的长度程序必须遍历这个字符串，直至遇到'0'为止，整个操作的时间复杂度为O(N)。而我们使用SDS封装字符串则直接获取len属性值即可，时间复杂度为O(1)。

## 二进制安全
C字符串中的字符除了末尾字符为'\0'外其他字符不能为空字符，否则会被认为是字符串结尾(即使实际上不是)。<br/>
这限制了C字符串只能保存文本数据，而不能保存二进制数据。而SDS使用len属性的值判断字符串是否结束，所以不会受'\0'的影响。

## 杜绝缓冲区溢出
字符串的拼接操作是使用十分频繁的，在C语言开发中使用char \*strcat(char \*dest,const char \*src)方法将src字符串中的内容拼接到dest字符串的末尾。由于C字符串不记录自身的长度，所有strcat方法已经认为用户在执行此函数时已经为dest分配了足够多的内存，足以容纳src字符串中的所有内容，而一旦这个条件不成立就会产生缓冲区溢出，会把其他数据覆盖掉。<br/>
<br/>
与C字符串不同，SDS 的自动扩容机制完全杜绝了发生缓冲区溢出的可能性：<br/>
当SDS API需要对SDS进行修改时，API会先检查 SDS 的空间是否满足修改所需的要求，如果不满足，API会自动将SDS的空间扩展至执行修改所需的大小，然后才执行实际的修改操作，所以使用 SDS 既不需要手动修改SDS的空间大小，也不会出现缓冲区溢出问题。<br/>
<br/>
SDS 的sds sdscat(sds s, const char \*t)方法在字符串拼接时会进行扩容相关操作。<br/>
<br/>
自动扩容机制——sdsMakeRoomFor方法<br/>
strcatlen中调用sdsMakeRoomFor完成字符串的容量检查及扩容操作，重点分析此方法：

```C
/* s: 源字符串
 * addlen: 新增长度
 */
sds sdsMakeRoomFor(sds s, size_t addlen) {
    void *sh, *newsh;
    // sdsavail: s->alloc - s->len, 获取 SDS 的剩余长度
    size_t avail = sdsavail(s);
    size_t len, newlen, reqlen;
    // 根据 flags 获取 SDS 的类型 oldtype
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;
    size_t usable;

    /* Return ASAP if there is enough space left. */
    // 剩余空间大于等于新增空间，无需扩容，直接返回源字符串
    if (avail >= addlen) return s;
    // 获取当前长度
    len = sdslen(s);
    // 
    sh = (char*)s-sdsHdrSize(oldtype);
    // 新长度
    reqlen = newlen = (len+addlen);
    // 断言新长度比原长度长，否则终止执行
    assert(newlen > len);   /* 防止数据溢出 */
    // SDS_MAX_PREALLOC = 1024*1024, 即1MB
    if (newlen < SDS_MAX_PREALLOC)
        // 新增后长度小于 1MB ，则按新长度的两倍扩容
        newlen *= 2;
    else
        // 新增后长度大于 1MB ，则按新长度加上 1MB 扩容
        newlen += SDS_MAX_PREALLOC;
    // 重新计算 SDS 的类型
    type = sdsReqType(newlen);

    /* Don't use type 5: the user is appending to the string and type 5 is
     * not able to remember empty space, so sdsMakeRoomFor() must be called
     * at every appending operation. */
    // 不使用 sdshdr5 
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;
    // 获取新的 header 大小
    hdrlen = sdsHdrSize(type);
    assert(hdrlen + newlen + 1 > reqlen);  /* Catch size_t overflow */
    if (oldtype==type) {
        // 类型没变
        // 调用 s_realloc_usable 重新分配可用内存，返回新 SDS 的头部指针
        // usable 会被设置为当前分配的大小
        newsh = s_realloc_usable(sh, hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL; // 分配失败直接返回NULL
        // 获取指向 buf 的指针
        s = (char*)newsh+hdrlen;
    } else {
        // 类型变化导致 header 的大小也变化，需要向前移动字符串，不能使用 realloc
        newsh = s_malloc_usable(hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL;
        // 将原字符串copy至新空间中
        memcpy((char*)newsh+hdrlen, s, len+1);
        // 释放原字符串内存
        s_free(sh);
        s = (char*)newsh+hdrlen;
        // 更新 SDS 类型
        s[-1] = type;
        // 设置长度
        sdssetlen(s, len);
    }
    // 获取 buf 总长度(待定)
    usable = usable-hdrlen-1;
    if (usable > sdsTypeMaxSize(type))
        // 若可用空间大于当前类型支持的最大长度则截断
        usable = sdsTypeMaxSize(type);
    // 设置 buf 总长度
    sdssetalloc(s, usable);
    return s;
}
```

自动扩容机制总结：
+ 扩容阶段：
+ + 若 SDS 中剩余空闲空间 avail 大于新增内容的长度 addlen，则无需扩容；
+ + 若 SDS 中剩余空闲空间 avail 小于或等于新增内容的长度 addlen：
+ + + 若新增后总长度 len+addlen < 1MB，则按新长度的两倍扩容；
+ + + 若新增后总长度 len+addlen > 1MB，则按新长度加上 1MB 扩容。
+ 内存分配阶段：
+ + 根据扩容后的长度选择对应的 SDS 类型：
+ + 若类型不变，则只需通过 s_realloc_usable扩大 buf 数组即可；
+ + 若类型变化，则需要为整个 SDS 重新分配内存，并将原来的 SDS 内容拷贝至新位置。
+ + 扩容后的 SDS 不会恰好容纳下新增的字符，而是多分配了一些空间(预分配策略)，这减少了修改字符串时带来的内存重分配次数。

## 内存重分配次数优化
(1) 空间预分配策略<br/>
因为 SDS 的空间预分配策略， SDS 字符串在增长过程中不会频繁的进行空间分配。<br/>
通过这种分配策略，SDS 将连续增长N次字符串所需的内存重分配次数从必定N次降低为最多N次。<br/>
<br/>
(2) 惰性空间释放机制<br/>
空间预分配策略用于优化 SDS 增长时频繁进行空间分配，而惰性空间释放机制则用于优化 SDS 字符串缩短时并不立即使用内存重分配来回收缩短后多出来的空间，而仅仅更新 SDS 的len属性，多出来的空间供将来使用。<br/>
<br/>
SDS 中调用sdstrim方法来缩短字符串：

```C
* sdstrim 方法删除字符串首尾中在 cset 中出现过的字符
 * 比如:
 * s = sdsnew("AA...AA.a.aa.aHelloWorld     :::");
 * s = sdstrim(s,"Aa. :");
 * printf("%s\n", s);
 *
 * SDS 变成了 "HelloWorld"
 */
sds sdstrim(sds s, const char *cset) {
    char *start, *end, *sp, *ep;
    size_t len;

    sp = start = s;
    ep = end = s+sdslen(s)-1;
    // strchr()函数用于查找给定字符串中某一个特定字符
    while(sp <= end && strchr(cset, *sp)) sp++;
    while(ep > sp && strchr(cset, *ep)) ep--;
    len = (sp > ep) ? 0 : ((ep-sp)+1);
    if (s != sp) memmove(s, sp, len);
    s[len] = '\0';
    // 仅仅更新了len
    sdssetlen(s,len);
    return s;
}
```

举例：<br/>
字符串："XXXXYYXBXXXBXXXYYY" ,修剪集合 "XY"<br/>
修剪结果 "为BXXXB"， 而不是"BB"<br/>
<br/>
SDS 并没有释放多出来的5字节空间，仅仅将 len 设置成了7，剩余空间为5。如果后续字符串增长时则可以派上用场（可能不需要再分配内存）。<br/>
<br/>
也许各位又会有疑问了，这没真正释放空间，是否会导致内存泄漏呢？<br/>
<br/>
放心，SDS为我们提供了真正释放SDS未使用空间的方法sdsRemoveFreeSpace。