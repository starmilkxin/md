[https://mp.weixin.qq.com/s/LavkCpqMTled_1m9CpJQ6w](https://mp.weixin.qq.com/s/LavkCpqMTled_1m9CpJQ6w)

# 位图简介
如果我们需要记录某一用户在一年中每天是否有登录我们的系统这一需求该如何完成呢？如果使用KV存储，每个用户需要记录365个，当用户量上亿时，这所需要的存储空间是惊人的。<br/>
<br/>
Redis 为我们提供了位图这一数据结构，每个用户每天的登录记录只占据一位，365天就是365位，仅仅需要46字节就可存储，极大地节约了存储空间。<br/>
<br/>
位图数据结构其实并不是一个全新的玩意，我们可以简单的认为就是个数组，只是里面的内容只能为0或1而已(二进制位数组)。

# BitMap源码分析
## 数据结构
![BitMap](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220306211756.png)

Redis 中的每个对象都是有一个 redisObject 结构表示的。

```Java
typedef struct redisObject {
 // 类型
 unsigned type:4;
 // 编码
 unsigned encoding:4;
 unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */
 // 引用计数
 int refcount;
 // 执行底层实现的数据结构的指针
 void *ptr;
} robj;
```

+ type 的值为 REDIS_STRING表示这是一个字符串对象
+ sdshdr.len 的值为1表示这个SDS保存了一个1字节大小的位数组
+ buf数组中的buf[0]实际保存了位数组
+ buf数组中的buf[1]为自动追加的\0字符

<br/>

## GETBIT
GETBIT命令源码如下所示：

```C
void getbitCommand(client *c) {
    robj *o;
    char llbuf[32];
    uint64_t bitoffset;
    size_t byte, bit;
    size_t bitval = 0;
    // 获取offset
    if (getBitOffsetFromArgument(c,c->argv[2],&bitoffset,0,0) != C_OK)
        return;
    // 查找对应的位图对象
    if ((o = lookupKeyReadOrReply(c,c->argv[1],shared.czero)) == NULL ||
        checkType(c,o,OBJ_STRING)) return;
  // 计算offset位于位数组的哪一行
    byte = bitoffset >> 3;
    // 计算offset在一行中的第几位，等同于取模
    bit = 7 - (bitoffset & 0x7);
    // #define sdsEncodedObject(objptr) (objptr->encoding == OBJ_ENCODING_RAW || objptr->encoding == OBJ_ENCODING_EMBSTR)
    if (sdsEncodedObject(o)) {
        // SDS 是RAW 或者 EMBSTR类型
        if (byte < sdslen(o->ptr))
            // 获取指定位置的值
            // 注意它不是真正的一个二维数组不能用((uint8_t*)o->ptr)[byte][bit]去获取呀~
            bitval = ((uint8_t*)o->ptr)[byte] & (1 << bit);
    } else {
        //  SDS 是 REDIS_ENCODING_INT 类型的整数，先转为String
        if (byte < (size_t)ll2string(llbuf,sizeof(llbuf),(long)o->ptr))
            bitval = llbuf[byte] & (1 << bit);
    }

    addReply(c, bitval ? shared.cone : shared.czero);
}
```

![GITBIT顺序图](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220306212115.png)

因为 GETBIT 命令执行的所有操作都可以在常数时间内完成，所以该命令的算法复杂度为O(1)。

## SETBIT
SETBIT用于将位数组在偏移量的二进制位的值设为value，并向客户端返回旧值。<br/>
<br/>
SITBIT命令的执行过程如下：
1. 计算len = offset / 8 + 1，len值记录了保存offset偏移量指定的二进制位至少需要多少字节
2. 检查位数组的长度是否小于len，如果是的话，将SDS的长度扩展为len字节,并将所有新扩展空间的二进制位设置为0
3. 计算byte = offset / 8，byte值表示指定的offset位于位数组的那个字节(就是计算在那个buf[i]中的i)
4. 使用bit = (offset mod 8) + 1计算可得目标的具体第几位
5. 根据byte和bit的值，首先保存oldValue，然后将新值value设置到目标位上
6. 返回旧值

因为SETBIT命令执行的所有操作都可以在常数时间内完成，所以该命令的算法复杂度为O(1)。<br/>
<br/>

SETBIT命令源码如下所示：

```C
void setbitCommand(client *c) {
    robj *o;
    char *err = "bit is not an integer or out of range";
    uint64_t bitoffset;
    ssize_t byte, bit;
    int byteval, bitval;
    long on;
    // 获取offset
    if (getBitOffsetFromArgument(c,c->argv[2],&bitoffset,0,0) != C_OK)
        return;
    // 获取我们需要设置的值
    if (getLongFromObjectOrReply(c,c->argv[3],&on,err) != C_OK)
        return;

    /* 判断指定值是否为0或1 */
    if (on & ~1) {
        // 设置了0和1之外的值，直接报错
        addReplyError(c,err);
        return;
    }
    // 根据key查询SDS对象（会自动扩容）
    if ((o = lookupStringForBitCommand(c,bitoffset)) == NULL) return;

    /* 获得当前值 */
    byte = bitoffset >> 3;
    byteval = ((uint8_t*)o->ptr)[byte];
    bit = 7 - (bitoffset & 0x7);
    bitval = byteval & (1 << bit);

    /* 更新值并返回旧值 */
    byteval &= ~(1 << bit);
    byteval |= ((on & 0x1) << bit);
    ((uint8_t*)o->ptr)[byte] = byteval;
    // 发送数据修改通知
    signalModifiedKey(c,c->db,c->argv[1]);
    notifyKeyspaceEvent(NOTIFY_STRING,"setbit",c->argv[1],c->db->id);
    server.dirty++;
    addReply(c, bitval ? shared.cone : shared.czero);
}
```

![SETBIT流程图](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/2e142b25d66e0df695922c823493e330.png)

## BITCOUNT
+ 统计一个位数组中非0二进制位的数量在数学上被称为"计算汉明重量"。
+ BITCOUNT命令用于统计给定位数组中值为1的二进制位的数量。功能似乎不复杂，但实际上要高效地实现这个命令并不容易，需要用到一些精巧的算法。

### 暴力遍历
小数据量还好，大数据量直接PASS!

### 查表法
对于一个有限集合来说，集合元素的排列方式是有限的，并且对于一个有限长度的位数组来说，它能表示的二进制位排列也是有限的。根据这个原理，我们可以创建一个表，表的键为某种排列的位数组，而表的值则是相应位数组中值为1的二进制位的数量。<br/>
查表法耗内存。

### 二进制位统计算法:variable-precision SWAR
目前已知效率最好的通用算法为variable-precision SWAR算法，该算法通过一系列位移和位运算操作，可以在常数时间内计算多个字节的汉明重量，并且不需要使用任何额外的内存。<br/>
<br/>
SWAR算法代码如下所示：

```C
uint32_t swar(uint32_t i) {
    // 5的二进制：0101
    i = (i & 0x55555555) + ((i >> 1) & 0x55555555);
    // 3的二进制：0011
    i = (i & 0x33333333) + ((i >> 2) & 0x33333333);
    i = (i & 0x0F0F0F0F) + ((i >> 4) & 0x0F0F0F0F);
    i = (i*(0x01010101) >> 24);
    return i;
  
    i = i - ((i >> 1) & 0x55555555);
    i = (i & 0x33333333) + ((i >> 2) & 0x33333333);
    return (((i + (i >> 4)) & 0x0F0F0F0F) * 0x01010101) >> 24;
}
```

1. 步骤一计算出的值i的二进制表示可以按每两个二进制位为一组进行分组，各组的十进制表示就是该组的1的数量；
2. 步骤二计算出的值i的二进制表示可以按每四个二进制位为一组进行分组，各组的十进制表示就是该组的1的数量；
3. 步骤三计算出的值i的二进制表示可以按每八个二进制位为一组进行分组，各组的十进制表示就是该组的1的数量；
4. 步骤四的i*0x01010101语句计算出bitarray中1的数量并记录在二进制位的最高八位，而>>24语句则通过右移运算，将bitarray的汉明重量移动到最低八位，得出的结果就是bitarray的汉明重量。

![步骤1](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/c0bc55c4c403804abeb92c6dbb4ac1ed.png)

![步骤3](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/78b12077879c3bd87c814047fe66c1e7.png)

![步骤3](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/78b12077879c3bd87c814047fe66c1e7.png)

### 源码分析
Redis 中通过调用redisPopcount方法统计汉明重量，源码如下所示：

```C
long long redisPopcount(void *s, long count) {
    long long bits = 0;
    unsigned char *p = s;
    uint32_t *p4;
    // 为查表法准备的表
    static const unsigned char bitsinbyte[256] = {0,1,1,2,1,2,2,3,1,2,2,3,2,3,3,4,1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,4,5,5,6,5,6,6,7,5,6,6,7,6,7,7,8};
    // CPU一次性读取8个字节，如果4字节跨了两个8字节，需要读取两次才行
    // 所以考虑4字节对齐，只需读取一次就可以读取完毕
    while((unsigned long)p & 3 && count) {
        bits += bitsinbyte[*p++];
        count--;
    }

    // 一次性处理28字节，单独看一个aux就容易理解了，其实就是SWAR算法
    // uint32_t：4字节
    p4 = (uint32_t*)p;
    while(count>=28) {
        uint32_t aux1, aux2, aux3, aux4, aux5, aux6, aux7;

        aux1 = *p4++;// 一次性读取4字节
        aux2 = *p4++;
        aux3 = *p4++;
        aux4 = *p4++;
        aux5 = *p4++;
        aux6 = *p4++;
        aux7 = *p4++;
        count -= 28;// 共处理了4*7=28个字节，所以count需要减去28

        aux1 = aux1 - ((aux1 >> 1) & 0x55555555);
        aux1 = (aux1 & 0x33333333) + ((aux1 >> 2) & 0x33333333);
        aux2 = aux2 - ((aux2 >> 1) & 0x55555555);
        aux2 = (aux2 & 0x33333333) + ((aux2 >> 2) & 0x33333333);
        aux3 = aux3 - ((aux3 >> 1) & 0x55555555);
        aux3 = (aux3 & 0x33333333) + ((aux3 >> 2) & 0x33333333);
        aux4 = aux4 - ((aux4 >> 1) & 0x55555555);
        aux4 = (aux4 & 0x33333333) + ((aux4 >> 2) & 0x33333333);
        aux5 = aux5 - ((aux5 >> 1) & 0x55555555);
        aux5 = (aux5 & 0x33333333) + ((aux5 >> 2) & 0x33333333);
        aux6 = aux6 - ((aux6 >> 1) & 0x55555555);
        aux6 = (aux6 & 0x33333333) + ((aux6 >> 2) & 0x33333333);
        aux7 = aux7 - ((aux7 >> 1) & 0x55555555);
        aux7 = (aux7 & 0x33333333) + ((aux7 >> 2) & 0x33333333);
        bits += ((((aux1 + (aux1 >> 4)) & 0x0F0F0F0F) +
                    ((aux2 + (aux2 >> 4)) & 0x0F0F0F0F) +
                    ((aux3 + (aux3 >> 4)) & 0x0F0F0F0F) +
                    ((aux4 + (aux4 >> 4)) & 0x0F0F0F0F) +
                    ((aux5 + (aux5 >> 4)) & 0x0F0F0F0F) +
                    ((aux6 + (aux6 >> 4)) & 0x0F0F0F0F) +
                    ((aux7 + (aux7 >> 4)) & 0x0F0F0F0F))* 0x01010101) >> 24;
    }
    /* 剩余的不足28字节，使用查表法统计 */
    p = (unsigned char*)p4;
    while(count--) bits += bitsinbyte[*p++];
    return bits;
}
```

不难发现 Redis 中同时运用了查表法和SWAR算法完成BITCOUNT功能。