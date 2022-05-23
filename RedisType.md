# SDS

Simple Dynamic String

## 结构

```c
//有5种,sdshdr5(不使用)、sdshdr8、sdshdr16、sdshdr32、sdshdr64
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* buf已保存的字符串字节数，不包含结束标示 */
    uint8_t alloc; /* buf申请的总的字节数，不包含结束标示 */
    unsigned char flags; /* 不同SDS的头类型，用来控制SDS的头大小 */
    char buf[];
};
```

例如，一个包含字符串“name”的sds结构如下：

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_18-21-29.png)

## 内存预分配

update命令才会预分配(如:append),set命令只是覆盖

目前有字符串

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_18-25-30.png)

假如要给SDS**追加**一段字符串“,Amy”，这里首先会申请新内存空间：

1. 如果新字符串小于1M，则新空间为扩展后字符串长度的两倍+1；
2. 如果新字符串大于1M，则新空间为扩展后字符串长度+1M+1。称为**内存预分配**

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_18-25-40.png)

## 优点

1. 获取字符串长度的时间复杂度为O(1)
2. 支持动态扩容
3. 减少内存分配次数
4. 二进制安全

# IntSet

IntSet是Redis中set集合的一种实现方式，基于**整数数组**来实现，并且具备长度可变、**有序**等特征。

## 结构

```c
typedef struct intset {
    uint32_t encoding; /* 编码方式，支持存放16位、32位、64位整数*/
    uint32_t length; /* 元素个数 */
    int8_t contents[]; /* 整数数组，保存集合数据，保存数组的指针*/
} intset;
```

其中的encoding包含三种模式，表示存储的整数大小不同：

```c
/* Note that these encodings are ordered, so:
 * INTSET_ENC_INT16 < INTSET_ENC_INT32 < INTSET_ENC_INT64. */
#define INTSET_ENC_INT16 (sizeof(int16_t)) /* 2字节整数，范围类似java的short*/
#define INTSET_ENC_INT32 (sizeof(int32_t)) /* 4字节整数，范围类似java的int */
#define INTSET_ENC_INT64 (sizeof(int64_t)) /* 8字节整数，范围类似java的long */
```

数组大小可以=元素个数*编码方式

数组内每个元素编码方式都是相同的

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_18-41-08.png)

寻址公式：startPtr + (sizeof(int16) * index)

contents的地址(第一个元素的地址)+元素编码方式所占用的字节*元素的角标

## 升级

假设有一个intset，元素为{5,10，20}，采用的编码是``INTSET_ENC_INT16``，则每个整数占2字节：

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_18-45-15.png)

向该其中添加一个数字：50000，这个数字超出了int16_t的范围，intset会自动**升级**编码方式到合适的大小

1. 升级编码为``INTSET_ENC_INT32``, 每个整数占4字节，并按照新的编码方式及元素个数扩容数组
2. **倒序**依次将数组中的元素拷贝到扩容后的正确位置
3. 将待添加的元素放入数组末尾
4. 最后，将inset的encoding属性改为``INTSET_ENC_INT32``，将length属性改为4

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_18-47-28.png)

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_18-48-59.png)

```c
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    uint8_t valenc = _intsetValueEncoding(value);// 获取当前值编码
    uint32_t pos; // 要插入的位置
    if (success) *success = 1;
    // 判断编码是不是超过了当前intset的编码
    if (valenc > intrev32ifbe(is->encoding)) {
        // 超出编码，需要升级
        return intsetUpgradeAndAdd(is,value);
    } else {
        // 在当前intset中查找值与value一样的元素的角标pos
        if (intsetSearch(is,value,&pos)) {//serrch时采用二分查找
            if (success) *success = 0; //如果找到了，则无需插入，直接结束并返回失败
            return is;
        }
        // 数组扩容
        is = intsetResize(is,intrev32ifbe(is->length)+1);
        // 移动数组中pos之后的元素到pos+1，给新元素腾出空间
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }
    // 插入新元素
    _intsetSet(is,pos,value);
    // 重置元素长度
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

```c
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    // 获取当前intset编码
    uint8_t curenc = intrev32ifbe(is->encoding);
    // 获取新编码
    uint8_t newenc = _intsetValueEncoding(value);
    // 获取元素个数
    int length = intrev32ifbe(is->length); 
    // 判断新元素是大于0还是小于0 ，小于0插入队首、大于0插入队尾
    int prepend = value < 0 ? 1 : 0;
    // 重置编码为新编码
    is->encoding = intrev32ifbe(newenc);
    // 重置数组大小
    is = intsetResize(is,intrev32ifbe(is->length)+1);
    // 倒序遍历，逐个搬运元素到新的位置，_intsetGetEncoded按照旧编码方式查找旧元素
    while(length--) // _intsetSet按照新编码方式插入新元素
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));
    /* 插入新元素，prepend决定是队首还是队尾*/
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);
    // 修改数组长度
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

## 特点

1. Intset中的元素唯一、有序
2. 具备类型升级机制，可以节省内存空间
3. 底层采用二分查找方式来查询(**数据量不宜过大**)

# Dict

## 结构

Dict由三部分组成，分别是：字典（Dict）、哈希表（DictHashTable）、哈希节点（DictEntry）

```c
typedef struct dict {
    dictType *type; /* dict类型，内置不同的hash函数*/
    void *privdata; /* 私有数据，在做特殊hash运算时用*/
    dictht ht[2]; /* 一个Dict包含两个哈希表，其中一个是当前数据，另一个一般是空，rehash时使用*/
    long rehashidx; /* rehash的进度，-1表示未进行*/
    int16_t pauserehash; /* rehash是否暂停，1则暂停，0则继续*/
} dict;
```

```c
typedef struct dictht {
    dictEntry **table; /* entry数组,数组中保存的是指向entry的指针*/
    unsigned long size; /* 哈希表大小*/
    unsigned long sizemask; /* 哈希表大小的掩码，总等于size - 1*/
    unsigned long used; /* entry个数*/
} dictht;
```

```c
typedef struct dictEntry {
    void *key; /* 键*/
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v; /* 值*/
    struct dictEntry *next; /* 下一个Entry的指针*/
} dictEntry;
```

dictEntry存储在dictht的哪一个桶中算法与java中hashTable类似

> 对dictEntry的key进行CRC16计算
>
> 计算结果再&16383

相同的桶采用头插法

> 不存在并发问题，redis单线程，
>
> 更快速，不用遍历链表再插入队尾

## 扩容

Dict中的HashTable就是**数组结合单向链表**的实现，当集合中元素较多时，必然导致哈希冲突增多，链表过长，则查询效率会大大降低。

注：dict中的dictht ht[2];为2个数组，第二个rehash时使用

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_19-14-09.png)

每次新增键值对时都会检查**负载因子**（LoadFactor = used/size） ，满足以下两种情况时会触发**哈希表扩容**：

1. 哈希表的 **LoadFactor >= 1**，并且服务器没有执行**BGSAVE 或者 BGREWRITEAOF** 等后台进程；
2. 哈希表的 **LoadFactor > 5** ；

```c
static int _dictExpandIfNeeded(dict *d){
    // 如果正在rehash，则返回ok
    if (dictIsRehashing(d)) return DICT_OK;
    // 如果哈希表为空，则初始化哈希表为默认大小：4
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);
    // 当负载因子（used/size）达到1以上，并且当前没有进行bgrewrite等子进程操作
    // 或者负载因子超过5，则进行 dictExpand ，也就是扩容
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize || d->ht[0].used/d->ht[0].size > dict_force_resize_ratio){
        // 扩容大小为used + 1，底层会对扩容大小做判断，实际上找的是第一个大于等于 used+1 的 2^n
        return dictExpand(d, d->ht[0].used + 1);
    }
    return DICT_OK;
}
```

## 收缩

每次删除元素时，会对负载因子做检查，当**LoadFactor < 0.1** 时，会做哈希表收缩

```c
if (dictDelete((dict*)o->ptr, field) == C_OK) {
    deleted = 1;
    // 删除成功后，检查是否需要重置Dict大小，如果需要则调用dictResize重置
    if (htNeedsResize(o->ptr)) dictResize(o->ptr);
}
```

```c
// server.c 文件
int htNeedsResize(dict *dict) {
    long long size, used;
    // 哈希表大小
    size = dictSlots(dict);
    // entry数量
    used = dictSize(dict);
    // size > 4（哈希表初识大小）并且 负载因子低于0.1
    return (size > DICT_HT_INITIAL_SIZE && (used*100/size < HASHTABLE_MIN_FILL));
}
```

```c
int dictResize(dict *d){
    unsigned long minimal;
    // 如果正在做bgsave或bgrewriteof或rehash，则返回错误
    if (!dict_can_resize || dictIsRehashing(d)) 
        return DICT_ERR;
    // 获取used，也就是entry个数
    minimal = d->ht[0].used;
    // 如果used小于4，则重置为4
    if (minimal < DICT_HT_INITIAL_SIZE)
        minimal = DICT_HT_INITIAL_SIZE;
    // 重置大小为minimal，其实是第一个大于等于minimal的2^n
    return dictExpand(d, minimal);
}
```

## 渐进式rehash

不管是扩容还是收缩，必定会创建新的哈希表，导致哈希表的size和sizemask变化，而key的查询与sizemask有关。因此必须对哈希表中的每一个key重新计算索引，插入新的哈希表，这个过程称为**rehash**

Dict的rehash是分多次、渐进式的完成，因此称为**渐进式rehash**

1. 计算新hash表的size
2. 按照新的size申请内存空间，创建dictht，并赋值给dict.ht[1]
3. 设置dict.rehashidx = 0，标示开始rehash
4. **每次执行新增、查询、修改、删除操作时**，都检查一下dict.rehashidx是否大于-1，如果是则将dict.ht[0].table[rehashidx]的entry链表rehash到dict.ht[1]，并且将rehashidx++。直至dict.ht[0]的所有数据都rehash到dict.ht[1]
5. 将dict.ht[1]赋值给dict.ht[0]，给dict.ht[1]初始化为空哈希表，释放原来的dict.ht[0]的内存
6. 将rehashidx赋值为-1，代表rehash结束
7. 在rehash过程中，新增操作，则直接写入ht[1]，查询、修改和删除则会在dict.ht[0]和dict.ht[1]依次查找并执行。这样可以确保ht[0]的数据只减不增，随着rehash最终为空

# ZipList

**ZipList** 是一种特殊的“双端链表”(**数组组成的链表**) ，由一系列特殊编码的连续内存块组成。可以在任意一端进行压入/弹出操作, 并且该操作的时间复杂度为 O(1)

## 结构

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_19-44-15.png)

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_19-45-45.png)

ZipList 中的**Entry**并不像普通链表那样记录前后节点的指针，因为记录两个指针要占用16个字节，浪费内存。而是采用了下面的结构：

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_20-13-51.png)

1. previous_entry_length：前一节点的长度，占1个或5个字节。
   1. 如果前一节点的长度小于254字节，则采用1个字节来保存这个长度值
   2. 如果前一节点的长度大于254字节，则采用5个字节来保存这个长度值，第一个字节为0xfe，后四个字节才是真实长度数据
2. encoding：编码属性，记录content的数据类型（字符串还是整数）以及长度，占用1个、2个或5个字节
3. contents：负责保存节点的数据，可以是字符串或整数

> 注：ZipList中所有存储长度的数值均采用小端字节序，即低位字节在前，高位字节在后。例如：数值0x1234，采用小端字节序后实际存储值为：0x3412

ZipListEntry中的encoding编码分为字符串和整数两种：

1. 字符串：如果encoding是以“00”、“01”或者“10”开头，则证明content是字符串

   > ![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_20-23-30.png)

2. 整数：如果encoding是以“11”开始，则证明content是整数，且encoding固定只占用1个字节

   > ![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_20-23-35.png)

例如，我们要保存字符串：“ab”和 “bc”

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_20-23-40.png)

例如，一个ZipList中包含两个整数值：“2”和“5”

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_20-30-26.png)

## 连锁更新问题

ZipList的每个Entry都包含previous_entry_length来记录上一个节点的大小，长度是1个或5个字节

1. 如果前一节点的长度小于254字节，则采用1个字节来保存这个长度值
2. 如果前一节点的长度大于等于254字节，则采用5个字节来保存这个长度值，第一个字节为0xfe，后四个字节才是真实长度数据

假设有**N个连续的、长度为250~253字节之间的entry**，因此entry的**previous_entry_length**属性用**1个字节**即可表示，如图所示

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_20-33-41.png)

接着要在头插入一个254字节大小的Entry

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_20-34-42.png)

可以观察到，由于有**n个连续的长度为250的entry**

**在头插入一个大于等于254字节的entry**后

后面一个记录前面entry所占用字节大小的**previous_entry_length**要从1个字节升到5个字节

自己本身占用250个字节，升级后，自己占用254个字节

后面的又需要改变...

插入一个而改变n个的变化，称为连锁更新(条件苛刻，出现概率较低，**面试专用**)

## 特点

1. 压缩列表的可以看做一种连续内存空间的"双向链表"
2. 列表的节点之间不是通过指针连接，而是记录上一节点和本节点长度来寻址，内存占用较低
3. 如果**列表数据过多，导致链表过长，可能影响查询性能**
4. 增或删较大数据时有可能发生连续更新问题

# QuickList

ZipList虽然节省内存，但申请内存必须是连续空间，如果内存占用较多，申请内存效率很低。怎么办？

> 为了缓解这个问题，必须限制ZipList的长度和entry大小

存储大量数据，超出了ZipList最佳的上限该怎么办？

> 可以创建多个ZipList来分片存储数据

数据拆分后比较分散，不方便管理和查找，多个ZipList如何建立联系？

> Redis在3.2版本引入了新的数据结构**QuickList**，它是一个双端链表，只不过链表中的每个节点都是一个ZipList

## 结构

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_20-47-50.png)

```c
typedef struct quicklist {
    quicklistNode *head; /* 头节点指针*/
    quicklistNode *tail; /* 尾节点指针*/
    unsigned long count; /* 所有ziplist的entry的数量*/
    unsigned long len; /* ziplists总数量*/
    int fill : QL_FILL_BITS; /* ziplist的entry上限，默认值 -2 */
    unsigned int compress : QL_COMP_BITS; /* 首尾不压缩的节点数量，默认值0*/
    unsigned int bookmark_count: QL_BM_BITS; /* 内存重分配时的书签数量及数组，一般用不到*/ 
    quicklistBookmark bookmarks[];
} quicklist;
```

```c
typedef struct quicklistNode {
    struct quicklistNode *prev; /* 前一个节点指针*/
    struct quicklistNode *next; /* 下一个节点指针*/
    unsigned char *zl; /* 当前节点的ZipList指针*/
    unsigned int sz; /* 当前节点的ZipList的字节大小*/
    unsigned int count : 16; /* 当前节点的ZipList的entry个数*/
    unsigned int encoding : 2; /* 编码方式：1，ZipList; 2，lzf压缩模式*/
    unsigned int container : 2; /* 数据容器类型（预留）：1，其它；2，ZipList*/
    unsigned int recompress : 1; /* 是否被解压缩。1：则说明被解压了，将来要重新压缩*/
    unsigned int attempted_compress : 1; /* 测试用*/
    unsigned int extra : 10; /*预留字段*/
} quicklistNode;
```

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_20-57-35.png)

## 配置项

``list-max-ziplist-size``，为了避免QuickList中的**每个ZipList中entry过多**

1. 如果值为正，则代表ZipList的允许的entry个数的最大值
2. 如果值为负，则代表ZipList的最大内存大小
   1. -1：每个ZipList的内存占用不能超过4kb
   2. -2：每个ZipList的内存占用不能超过8kb **默认**
   3. -3：每个ZipList的内存占用不能超过16kb
   4. -4：每个ZipList的内存占用不能超过32kb
   5. -5：每个ZipList的内存占用不能超过64kb

``list-compress-depth``可**对节点的ZipList做压缩**，因为链表一般都是从首尾访问较多，所以首尾是不压缩的。这个参数是**控制首尾不压缩的节点个数**

1. 0：特殊值，代表不压缩 **默认**
2. 1：标示QuickList的首尾各有1个节点不压缩，中间节点压缩
3. 2：标示QuickList的首尾各有2个节点不压缩，中间节点压缩
4. 以此类推

## 特点

1. 是一个节点为ZipList的双端链表
2. 节点采用ZipList，解决了传统链表的内存占用问题
3. 控制了ZipList大小，解决连续内存空间申请效率问题
4. 中间节点可以压缩，进一步节省了内存

# SkipList

## 结构

**SkipList（跳表）**首先是链表，但与传统链表相比有几点差异

1. 元素按照升序排列存储
2. 节点可能包含多个指针，指针跨度不同

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_22-23-59.png)

```c
// t_zset.c
typedef struct zskiplist {
    struct zskiplistNode *header, *tail; /* 头尾节点指针*/
    unsigned long length; /* 节点数量*/
    int level; /* 最大的索引层级，默认是1*/
} zskiplist;
```

```c
// t_zset.c
typedef struct zskiplistNode {
    sds ele; /* 节点存储的值*/
    double score; /* 节点分数，排序、查找用*/
    struct zskiplistNode *backward; /* 前一个节点指针*/
    struct zskiplistLevel {
        struct zskiplistNode *forward; /* 下一个节点指针*/
        unsigned long span; /* 索引跨度*/
    } level[]; /* 多级索引数组*/
} zskiplistNode;
```

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_22-32-17.png)

## 特点

1. 跳跃表是一个双向链表，每个节点都包含score和ele值
2. 节点按照score值排序，score值一样则按照ele字典排序
3. 每个节点都可以包含多层指针，层数是1到32之间的随机数
4. 不同层指针到下一个节点的跨度不同，层级越高，跨度越大
5. 增删改查效率与红黑树基本一致，实现却更简单

# RedisObject

Redis中的任意数据类型的键和值都会被封装为一个RedisObject，也叫做Redis对象

## 结构

```c
typedef struct redisObject {
    unsigned type:4; /* 对象类型，分别是string、hash、list、set和zset，占4个bit位*/
    unsigned encoding:4; /* 底层编码方式，共有11种，占4个bit位*/
    unsigned lru:LRU_BITS; /* LRU_BITS为24,lru表示该对象最后一次被访问的时间，其占用24个bit位。便于判断空闲时间太久的key*/
    int refcount; /* 对象引用计数器，计数器为0则说明对象无人引用，可以被回收*/
    void *ptr; /* 指针，指向存放实际数据的空间*/
} robj;
```

0.5+0.5+3+4+8=16字节

## 编码方式

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_22-54-17.png)

## 数据结构

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_22-56-18.png)

#  String

可能存在raw、embstr、int三种编码

## RAW

其基本编码方式是**RAW**，基于简单动态字符串（SDS）实现，存储上限为512mb

> ![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_23-02-14.png)

## EMBSTR

如果存储的SDS长度小于44字节，则会采用**EMBSTR**编码，此时object head与SDS是一段连续空间。申请内存时只需要调用一次内存分配函数，效率更高

> ![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_23-04-22.png)
>
> 为什么是44字节？
>
> redisObject占16字节，sds头占3字节，字符串占44字节，尾占1字节，加起来64字节
>
> redis分配内存以2的n次方分配

## INT

如果存储的字符串是整数值，并且大小在LONG_MAX范围内，则会采用**INT**编码：直接将数据保存在RedisObject的ptr指针位置（刚好8字节），不再需要SDS了

> ![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-21_23-11-20.png)

# List

## QuickList

quickList：LinkedList + ZipList，可以从双端访问，内存占用较低，包含多个ZipList，存储上限高

1. 在3.2版本之前，Redis采用ZipList和LinkedList来实现List，当元素数量小于512并且元素大小小于64字节时采用ZipList编码，超过则采用LinkedList编码。
2. 在3.2版本之后，Redis统一采用QuickList来实现List

```c
/*
 * @param client 保存客户端的信息，包括客户端发送的命令
 * @param where  表示从队首插还是队尾插
 * @param xx     表示是否存在才插入(通常都是1,不存在会自动创建list)
*/
void pushGenericCommand(client *c, int where, int xx) {
    int j;
    // 尝试找到KEY对应的list
    robj *lobj = lookupKeyWrite(c->db, c->argv[1]);
    // 检查类型是否正确
    if (checkType(c,lobj,OBJ_LIST)) return;
    // 检查是否为空
    if (!lobj) {
        if (xx) {
            addReply(c, shared.czero);
            return;
        }
        // 为空，则创建新的QuickList
        lobj = createQuicklistObject();
        //限制zipList的大小,默认:每个zipList不能超过8kb,每个zipList都不压缩
        quicklistSetOptions(lobj->ptr, server.list_max_ziplist_size,
                            server.list_compress_depth);
        dbAdd(c->db,c->argv[1],lobj);
    }
    // 略 ...
}
```

```c
robj *createQuicklistObject(void) {
    // 申请内存并初始化QuickList
    quicklist *l = quicklistCreate();
    // 创建RedisObject，type为OBJ_ENCODING_RAW
    // ptr指向 QuickList
    robj *o = createObject(OBJ_LIST,l);
    // 设置编码为 QuickList
    o->encoding = OBJ_ENCODING_QUICKLIST;
    return o;
}
```

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-22_21-50-27.png)

# Set

可能存在IntSet、Dict

1. 为了查询效率和唯一性，set采用HT编码（Dict）。Dict中的key用来存储元素，value统一为null。
2. 当存储的所有数据**都是整数**，并且元素数量不超过**set-max-intset-entries(默认512)**时，Set会采用IntSet编码，以节省内存

```c
robj *setTypeCreate(sds value) {
    // 判断value是否是数值类型 long long 
    if (isSdsRepresentableAsLongLong(value,NULL) == C_OK)
        // 如果是数值类型，则采用IntSet编码
        return createIntsetObject();
    // 否则采用默认编码，也就是HT
    return createSetObject();
}
```

```c
robj *createIntsetObject(void) {
    // 初始化INTSET并申请内存空间
    intset *is = intsetNew();
    // 创建RedisObject
    robj *o = createObject(OBJ_SET,is);
    // 指定编码为INTSET
    o->encoding = OBJ_ENCODING_INTSET;
    return o;
}
```

```c
robj *createSetObject(void) {
    // 初始化Dict类型，并申请内存
    dict *d = dictCreate(&setDictType,NULL);
    // 创建RedisObject
    robj *o = createObject(OBJ_SET,d);
    // 设置encoding为HT
    o->encoding = OBJ_ENCODING_HT;
    return o;
}
```

## Inset

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-22_22-01-41.png)

## Inset-->Dict

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-22_22-01-42.png)

## Dict

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-22_22-03-12.png)

# Zset

1. 可以根据score值排序后
2. member必须唯一
3. 可以根据member查询分数

zset底层数据结构必须满足**键值存储、键必须唯一、可排序**这几个需求

1. **ZipList**，占用内存少，使用代码维护

2. **Dict**：可以键值存储，并且可以根据key找value

   **SkipList**：可以排序，并且可以同时存储score和ele值（member）

   > 1. 元素数量大于zset_max_ziplist_entries，默认值128
   > 2. 任意元素大于zset_max_ziplist_value字节，默认值64

## ZipList

```c
// zadd添加元素时，先根据key找到zset，不存在则创建新的zset
zobj = lookupKeyWrite(c->db,key);
if (checkType(c,zobj,OBJ_ZSET)) goto cleanup;
// 判断是否存在
if (zobj == NULL) { // zset不存在
    if (server.zset_max_ziplist_entries == 0 ||
        server.zset_max_ziplist_value < sdslen(c->argv[scoreidx+1]->ptr))
    { // zset_max_ziplist_entries设置为0就是禁用了ZipList，
        // 或者value大小超过了zset_max_ziplist_value，采用HT + SkipList
        zobj = createZsetObject();
    } else { // 否则，采用 ZipList
        zobj = createZsetZiplistObject();
    }
    dbAdd(c->db,key,zobj); 
}
// ....
zsetAdd(zobj, score, ele, flags, &retflags, &newscore);
```

```c
robj *createZsetObject(void) {
    // 申请内存
    zset *zs = zmalloc(sizeof(*zs));
    robj *o;
    // 创建Dict
    zs->dict = dictCreate(&zsetDictType,NULL);
    // 创建SkipList
    zs->zsl = zslCreate();
    o = createObject(OBJ_ZSET,zs);
    o->encoding = OBJ_ENCODING_SKIPLIST;
    return o;
}
```

```c
robj *createZsetZiplistObject(void) {
    // 创建ZipList
    unsigned char *zl = ziplistNew();
    robj *o = createObject(OBJ_ZSET,zl);
    o->encoding = OBJ_ENCODING_ZIPLIST;
    return o;
}
```

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-22_22-29-22.png)

## ZipList--->Dict&&SkipList

```c
int zsetAdd(robj *zobj, double score, sds ele, int in_flags, int *out_flags, double *newscore) {
    /* 判断编码方式*/
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {// 是ZipList编码
        unsigned char *eptr;
        // 判断当前元素是否已经存在，已经存在则更新score即可
        if ((eptr = zzlFind(zobj->ptr,ele,&curscore)) != NULL) {
            //...略
            return 1;
        } else if (!xx) {
            // 元素不存在，需要新增，则判断ziplist长度有没有超、元素的大小有没有超
            if (zzlLength(zobj->ptr)+1 > server.zset_max_ziplist_entries
 		|| sdslen(ele) > server.zset_max_ziplist_value 
 		|| !ziplistSafeToAdd(zobj->ptr, sdslen(ele)))
            { // 如果超出，则需要转为SkipList编码
                zsetConvert(zobj,OBJ_ENCODING_SKIPLIST);
            } else {
                zobj->ptr = zzlInsert(zobj->ptr,ele,score);
                if (newscore) *newscore = score;
                *out_flags |= ZADD_OUT_ADDED;
                return 1;
            }
        } else {
            *out_flags |= ZADD_OUT_NOP;
            return 1;
        }
    }
	// 本身就是SKIPLIST编码，无需转换
    if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
       // ...略
    } else {
        serverPanic("Unknown sorted set encoding");
    }
    return 0; /* Never reached. */
}
```

## Dict&&SkipList

```c
// zset结构
typedef struct zset {
    // Dict指针
    dict *dict;
    // SkipList指针
    zskiplist *zsl;
} zset;
```

```c
robj *createZsetObject(void) {
    zset *zs = zmalloc(sizeof(*zs));
    robj *o;
    // 创建Dict
    zs->dict = dictCreate(&zsetDictType,NULL);
    // 创建SkipList
    zs->zsl = zslCreate(); 
    o = createObject(OBJ_ZSET,zs);
    o->encoding = OBJ_ENCODING_SKIPLIST;
    return o;
}
```

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-22_22-13-10.png)

# Hash

Hash结构**默认采用ZipList**编码，用以节省内存。 ZipList中相邻的两个entry 分别保存field和value

当数据量较大时，Hash结构会转为**Dict**编码，触发条件有两个：

1. ZipList中的元素数量超过了hash-max-ziplist-entries（默认512）
2. ZipList中的任意entry大小超过了hash-max-ziplist-value（默认64字节）

## ZipList

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-22_22-33-16.png)

```c
void hsetCommand(client *c) {// hset user1 name Jack age 21
    int i, created = 0;
    robj *o; // 略 ...
    // 判断hash的key是否存在，不存在则创建一个新的，默认采用ZipList编码
    if ((o = hashTypeLookupWriteOrCreate(c,c->argv[1])) == NULL) return;
    // 判断是否需要把ZipList转为Dict
    hashTypeTryConversion(o,c->argv,2,c->argc-1);
    // 循环遍历每一对field和value，并执行hset命令
    for (i = 2; i < c->argc; i += 2)
        created += !hashTypeSet(o,c->argv[i]->ptr,c->argv[i+1]->ptr,HASH_SET_COPY);
    // 略 ...
}
```

```c
robj *hashTypeLookupWriteOrCreate(client *c, robj *key) {
    // 查找key
    robj *o = lookupKeyWrite(c->db,key);
    if (checkType(c,o,OBJ_HASH)) return NULL;
    // 不存在，则创建新的
    if (o == NULL) {
        o = createHashObject();
        dbAdd(c->db,key,o);
    }
    return o;
}
```

## ZipList-->Dict

```c
robj *createHashObject(void) {
    // 默认采用ZipList编码，申请ZipList内存空间
    unsigned char *zl = ziplistNew();
    robj *o = createObject(OBJ_HASH, zl);
    // 设置编码
    o->encoding = OBJ_ENCODING_ZIPLIST;
    return o;
}
```

```c
void hashTypeTryConversion(robj *o, robj **argv, int start, int end) {
    int i;
    size_t sum = 0;
    // 本来就不是ZipList编码，什么都不用做了
    if (o->encoding != OBJ_ENCODING_ZIPLIST) return;
    // 依次遍历命令中的field、value参数
    for (i = start; i <= end; i++) {
        if (!sdsEncodedObject(argv[i]))
            continue;
        size_t len = sdslen(argv[i]->ptr);
        // 如果field或value超过hash_max_ziplist_value，则转为HT
        if (len > server.hash_max_ziplist_value) {
            hashTypeConvert(o, OBJ_ENCODING_HT);
            return;
        }
        sum += len;
    }// ziplist大小超过1G，也转为HT
    if (!ziplistSafeToAdd(o->ptr, sum))
        hashTypeConvert(o, OBJ_ENCODING_HT);
}
```

```c
int hashTypeSet(robj *o, sds field, sds value, int flags) {
    int update = 0;
    // 判断是否为ZipList编码
    if (o->encoding == OBJ_ENCODING_ZIPLIST) {
        unsigned char *zl, *fptr, *vptr;
        zl = o->ptr;
        // 查询head指针
        fptr = ziplistIndex(zl, ZIPLIST_HEAD);
        if (fptr != NULL) { // head不为空，说明ZipList不为空，开始查找key
            fptr = ziplistFind(zl, fptr, (unsigned char*)field, sdslen(field), 1);
            if (fptr != NULL) {// 判断是否存在，如果已经存在则更新
                update = 1;
                zl = ziplistReplace(zl, vptr, (unsigned char*)value,
                        sdslen(value));
            }
        }
        // 不存在，则直接push
        if (!update) { // 依次push新的field和value到ZipList的尾部
            zl = ziplistPush(zl, (unsigned char*)field, sdslen(field),
                    ZIPLIST_TAIL);
            zl = ziplistPush(zl, (unsigned char*)value, sdslen(value),
                    ZIPLIST_TAIL);
        }
        o->ptr = zl;
        /* 插入了新元素，检查list长度是否超出，超出则转为HT */
        if (hashTypeLength(o) > server.hash_max_ziplist_entries)
            hashTypeConvert(o, OBJ_ENCODING_HT);
    } else if (o->encoding == OBJ_ENCODING_HT) {
        // HT编码，直接插入或覆盖
    } else {
        serverPanic("Unknown hash encoding");
    }
    return update;
}
```

## Dict

![img](https://github.com/cbxbj/redis/blob/master/img/Snipaste_2022-05-22_22-34-15.png)
