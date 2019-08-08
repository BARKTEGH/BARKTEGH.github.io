---
title: Redis源码分析
date: 2019-05-28 20:16:20
tags:
- Redis
- 分布式
categories:
- Redis
---


## Redis 数据结构



### 简单动态字符串SDS

----------


Redis没有直接使用传统字符串表示，而是构建一种名为简单动态字符串的抽象类型，并将SDS用作Redis默认字符串表示。

SDS定义如下：

    struct sdshdr{

		//记录buf数组中已使用字节的数量
		//	等于SDS所保存字符串的长度
		int len；
		//记录buf数组中未使用字节数量；
		int free；
		//字节数组，用于保存字符串；
		char buf[]；

	}

SDS与C字符串区别：

1. 常数复杂度获取字符串长度
2. 杜绝缓冲区溢出
3. 减少修改字符串时带来的内存重分配次数
4. 二进制安全
5. 兼容部分C字符串函数


### 链表
----
链表节点定义：
	
    typedef struct listNode {

	    // 前置节点
	    struct listNode *prev;
	
	    // 后置节点
	    struct listNode *next;
	
	    // 节点的值
	    void *value;
	
	} listNode;

链表节点定义：

	typedef struct list {

	    // 表头节点
	    listNode *head;
	
	    // 表尾节点
	    listNode *tail;
	
	    // 链表所包含的节点数量
	    unsigned long len;
	
	    // 节点值复制函数
	    void *(*dup)(void *ptr);
	
	    // 节点值释放函数
	    void (*free)(void *ptr);
	
	    // 节点值对比函数
	    int (*match)(void *ptr, void *key);
	
	} list;

Redis 的链表实现的特性可以总结如下：

- 双端： 链表节点带有 prev 和 next 指针， 获取某个节点的前置节点和后置节点的复杂度都是 O(1) 。
- 无环： 表头节点的 prev 指针和表尾节点的 next 指针都指向 NULL ， 对链表的访问以 NULL 为终点。
- 带表头指针和表尾指针： 通过 list 结构的 head 指针和 tail 指针， 程序获取链表的表头节点和表尾节点的复杂度为 O(1) 。
- 带链表长度计数器： 程序使用 list 结构的 len 属性来对 list 持有的链表节点进行计数， 程序获取链表中节点数量的复杂度为 O(1)。
- 多态： 链表节点使用 void* 指针来保存节点值， 并且可以通过 list 结构的 dup 、 free 、 match 三个属性为节点值设置类型特定函数， 所以链表可以用于保存各种不同类型的值。

### 字典
-------

Redis 的字典使用哈希表作为底层实现， 一个哈希表里面可以有多个哈希表节点， 而每个哈希表节点就保存了字典中的一个键值对。


**哈希表节点**

哈希表节点使用 dictEntry 结构表示， 每个 dictEntry 结构都保存着一个键值对：
	typedef struct dictEntry {
	
	    // 键
	    void *key;
	
	    // 值
	    union {
	        void *val;
	        uint64_t u64;
	        int64_t s64;
	    } v;
	
	    // 指向下个哈希表节点，形成链表
	    struct dictEntry *next;
	
	} dictEntry;

key 属性保存着键值对中的键， 而 v 属性则保存着键值对中的值， 其中键值对的值可以是一个指针， 或者是一个 uint64_t 整数， 又或者是一个 int64_t 整数。

next 属性是指向另一个哈希表节点的指针， 这个指针可以将多个哈希值相同的键值对连接在一次， 以此来解决键冲突（collision）的问题。

**哈希表**

	typedef struct dictht {

	    // 哈希表数组
	    dictEntry **table;
	
	    // 哈希表大小
	    unsigned long size;
	
	    // 哈希表大小掩码，用于计算索引值
	    // 总是等于 size - 1
	    unsigned long sizemask;
	
	    // 该哈希表已有节点的数量
	    unsigned long used;
	
	} dictht;

table 属性是一个数组， 数组中的每个元素都是一个指向 dict.h/dictEntry 结构的指针， 每个 dictEntry 结构保存着一个键值对。

size 属性记录了哈希表的大小， 也即是 table 数组的大小， 而 used 属性则记录了哈希表目前已有节点（键值对）的数量。

sizemask 属性的值总是等于 size - 1 ， 这个属性和哈希值一起决定一个键应该被放到 table 数组的哪个索引上面。


一个大小为 4 的空哈希表 （没有包含任何键值对）
![](https://i.imgur.com/YzynOZ6.png)


**字典**


	typedef struct dict {
	
	    // 类型特定函数
	    dictType *type;
	
	    // 私有数据
	    void *privdata;
	
	    // 哈希表
	    dictht ht[2];
	
	    // rehash 索引
	    // 当 rehash 不在进行时，值为 -1
	    int rehashidx; /* rehashing not in progress if rehashidx == -1 */
	
	} dict;

![](https://i.imgur.com/e4ZYw1P.png)

*使用链地址头插法发来解决键冲突*

**Redis rehash**

随着操作的不断执行， 哈希表保存的键值对会逐渐地增多或者减少， 为了让哈希表的负载因子（load factor）维持在一个合理的范围之内， 当哈希表保存的键值对数量太多或者太少时， 程序需要对哈希表的大小进行相应的扩展或者收缩。

扩展和收缩哈希表的工作可以通过执行 rehash （重新散列）操作来完成， Redis 对字典的哈希表执行 rehash 的步骤如下：



1. 为字典的 ht[1] 哈希表分配空间， 这个哈希表的空间大小取决于要执行的操作， 以及 ht[0] 当前包含的键值对数量 （也即是ht[0].used 属性的值）：
	- 如果执行的是扩展操作， 那么 ht[1] 的大小为第一个大于等于 ht[0].used * 2 的 2^n （2 的 n 次方幂）；
	- 如果执行的是收缩操作， 那么 ht[1] 的大小为第一个大于等于 ht[0].used 的 2^n 。
2. 将保存在 ht[0] 中的所有键值对 rehash 到 ht[1] 上面： rehash 指的是重新计算键的哈希值和索引值， 然后将键值对放置到 ht[1] 哈希表的指定位置上。
3. 当 ht[0] 包含的所有键值对都迁移到了 ht[1] 之后 （ht[0] 变为空表）， 释放 ht[0] ， 将 ht[1] 设置为 ht[0] ， 并在 ht[1] 新创建一个空白哈希表， 为下一次 rehash 做准备。


**Redis 渐进式reshash**

因此， 为了避免 rehash 对服务器性能造成影响， 服务器不是一次性将 ht[0] 里面的所有键值对全部 rehash 到 ht[1] ， 而是分多次、渐进式地将 ht[0] 里面的键值对慢慢地 rehash 到 ht[1] 。

以下是哈希表渐进式 rehash 的详细步骤：

1. 为 ht[1] 分配空间， 让字典同时持有 ht[0] 和 ht[1] 两个哈希表。
2. 在字典中维持一个索引计数器变量 rehashidx ， 并将它的值设置为 0 ， 表示 rehash 工作正式开始。
3. 在 rehash 进行期间， 每次对字典执行添加、删除、查找或者更新操作时， 程序除了执行指定的操作以外， 还会顺带将 ht[0] 哈希表在 rehashidx 索引上的所有键值对 rehash 到 ht[1] ， 当 rehash 工作完成之后， 程序将 rehashidx 属性的值增一。
4. 随着字典操作的不断执行， 最终在某个时间点上， ht[0] 的所有键值对都会被 rehash 至 ht[1] ， 这时程序将 rehashidx 属性的值设为 -1 ， 表示 rehash 操作已完成。

渐进式 rehash 的好处在于它采取分而治之的方式， 将 rehash 键值对所需的计算工作均滩到对字典的每个添加、删除、查找和更新操作上， 从而避免了集中式 rehash 而带来的庞大计算量。

**哈希表的扩展与收缩**

当以下条件中的任意一个被满足时， 程序会自动开始对哈希表执行扩展操作：

1. 服务器目前没有在执行 BGSAVE 命令或者 BGREWRITEAOF 命令， 并且哈希表的负载因子大于等于 1 ；
2. 服务器目前正在执行 BGSAVE 命令或者 BGREWRITEAOF 命令， 并且哈希表的负载因子大于等于 5 ；

		# 负载因子 = 哈希表已保存节点数量 / 哈希表大小
		load_factor = ht[0].used / ht[0].size


### 跳跃表
-------

![](https://i.imgur.com/1Z2bIox.png)


Redis 的跳跃表由 redis.h/zskiplistNode 和 redis.h/zskiplist 两个结构定义， 其中 zskiplistNode 结构用于表示跳跃表节点， 而 zskiplist结构则用于保存跳跃表节点的相关信息， 比如节点的数量， 以及指向表头节点和表尾节点的指针， 等等。

![](https://i.imgur.com/HlBGVDS.png)

展示了一个跳跃表示例， 位于图片最左边的是 zskiplist 结构， 该结构包含以下属性：

- header ：指向跳跃表的表头节点。
- tail ：指向跳跃表的表尾节点。
- level ：记录目前跳跃表内，层数最大的那个节点的层数（表头节点的层数不计算在内）。
- length ：记录跳跃表的长度，也即是，跳跃表目前包含节点的数量（表头节点不计算在内）。

位于 zskiplist 结构右方的是四个 zskiplistNode 结构， 该结构包含以下属性：

- 层（level）：节点中用 L1 、 L2 、 L3 等字样标记节点的各个层， L1 代表第一层， L2 代表第二层，以此类推。每个层都带有两个属性：前进指针和跨度。前进指针用于访问位于表尾方向的其他节点，而跨度则记录了前进指针所指向节点和当前节点的距离。在上面的图片中，连线上带有数字的箭头就代表前进指针，而那个数字就是跨度。当程序从表头向表尾进行遍历时，访问会沿着层的前进指针进行。
- 后退（backward）指针：节点中用 BW 字样标记节点的后退指针，它指向位于当前节点的前一个节点。后退指针在程序从表尾向表头遍历时使用。
- 分值（score）：各个节点中的 1.0 、 2.0 和 3.0 是节点所保存的分值。在跳跃表中，节点按各自所保存的分值从小到大排列。
- 成员对象（obj）：各个节点中的 o1 、 o2 和 o3 是节点所保存的成员对象。

**跳跃表节点**

跳跃表节点的实现由 redis.h/zskiplistNode 结构定义：

	typedef struct zskiplistNode {
	
	    // 后退指针
	    struct zskiplistNode *backward;
	
	    // 分值
	    double score;
	
	    // 成员对象
	    robj *obj;
	
	    // 层
	    struct zskiplistLevel {
	
	        // 前进指针
	        struct zskiplistNode *forward;
	
	        // 跨度
	        unsigned int span;
	
	    } level[];
	
	} zskiplistNode;



与红黑树等平衡树相比，跳跃表具有以下优点：

- 插入速度非常快速，因为不需要进行旋转等操作来维护平衡性；
- 更容易实现；
- 支持无锁操作。

### 整数集合

--------

整数集合（intset）是 Redis 用于保存整数值的集合抽象数据结构， 它可以保存类型为 int16_t 、 int32_t 或者 int64_t 的整数值， 并且保证集合中不会出现重复元素。

每个 intset.h/intset 结构表示一个整数集合：

    typedef struct intset {

	    // 编码方式
	    uint32_t encoding;
	
	    // 集合包含的元素数量
	    uint32_t length;
	
	    // 保存元素的数组
	    int8_t contents[];
	
	} intset;

contents 数组是整数集合的底层实现： 整数集合的每个元素都是 contents 数组的一个数组项（item）， 各个项在数组中按值的大小从小到大有序地排列， 并且数组中不包含任何重复项。

length 属性记录了整数集合包含的元素数量， 也即是 contents 数组的长度。

虽然 intset 结构将 contents 属性声明为 int8_t 类型的数组， 但实际上 contents 数组并不保存任何 int8_t 类型的值 —— contents 数组的真正类型取决于 encoding 属性的值。

**升级**

每当我们要将一个新元素添加到整数集合里面， 并且新元素的类型比整数集合现有所有元素的类型都要长时， 整数集合需要先进行升级（upgrade）， 然后才能将新元素添加到整数集合里面。

升级整数集合并添加新元素共分为三步进行：

- 根据新元素的类型， 扩展整数集合底层数组的空间大小， 并为新元素分配空间。
- 将底层数组现有的所有元素都转换成与新元素相同的类型， 并将类型转换后的元素放置到正确的位上， 而且在放置元素的过程中， 需要继续维持底层数组的有序性质不变。
- 将新元素添加到底层数组里面。

升级的好处：

- 提升灵活性：
	
	因为 C 语言是静态类型语言， 为了避免类型错误， 我们通常不会将两种不同类型的值放在同一个数据结构里面。 因为整数集合可以通过自动升级底层数组来适应新元素， 所以我们可以随意地将 int16_t 、 int32_t 或者 int64_t 类型的整数添加到集合中， 而不必担心出现类型错误， 这种做法非常灵活。

- 节约内存

	整数集合现在的做法既可以让集合能同时保存三种不同类型的值， 又可以确保升级操作只会在有需要的时候进行， 这可以尽量节省内存。

**降级**

整数集合不支持降级操作， 一旦对数组进行了升级， 编码就会一直保持升级后的状态。

### 压缩列表

压缩列表是 Redis 为了节约内存而开发的， 由一系列特殊编码的连续内存块组成的顺序型（sequential）数据结构。

一个压缩列表可以包含任意多个节点（entry）， 每个节点可以保存一个字节数组或者一个整数值。

图 7-1 展示了压缩列表的各个组成部分， 表 7-1 则记录了各个组成部分的类型、长度、以及用途。

![](https://i.imgur.com/vRS8HAs.png)

![](https://i.imgur.com/do4tdiY.png)

**压缩列表节点的构成**

每个压缩列表节点都由 previous_entry_length 、 encoding 、 content 三个部分组成。

![](https://i.imgur.com/wck9ARV.png)

***previous_entry_length***

节点的 previous_entry_length 属性以字节为单位， 记录了压缩列表中前一个节点的长度。
previous_entry_length 属性的长度可以是 1 字节或者 5 字节：

- 如果前一节点的长度小于 254 字节， 那么 previous_entry_length 属性的长度为 1 字节： 前一节点的长度就保存在这一个字节里面。
- 如果前一节点的长度大于等于 254 字节， 那么 previous_entry_length 属性的长度为 5 字节： 其中属性的第一字节会被设置为 0xFE（十进制值 254）， 而之后的四个字节则用于保存前一节点的长度。

**encoding**

节点的 encoding 属性记录了节点的 content 属性所保存数据的类型以及长度：

- 一字节、两字节或者五字节长， 值的最高位为 00 、 01 或者 10 的是字节数组编码： 这种编码表示节点的 content 属性保存着字节数组， 数组的长度由编码除去最高两位之后的其他位记录；
- 一字节长， 值的最高位以 11 开头的是整数编码： 这种编码表示节点的 content 属性保存着整数值， 整数值的类型和长度由编码除去最高两位之后的其他位记录；

**content**

节点的 content 属性负责保存节点的值， 节点值可以是一个字节数组或者整数， 值的类型和长度由节点的 encoding 属性决定。

 其中，字节数组可以是以下三种长度的其中一种：

- 长度小于等于 63 （2^{6}-1）字节的字节数组；
- 长度小于等于 16383 （2^{14}-1） 字节的字节数组；
- 长度小于等于 4294967295 （2^{32}-1）字节的字节数组；

而整数值则可以是以下六种长度的其中一种：

- 4 位长，介于 0 至 12 之间的无符号整数；
- 1 字节长的有符号整数；
- 3 字节长的有符号整数；
- int16_t 类型整数；
- int32_t 类型整数；
- int64_t 类型整数。

**连锁更新**

 在一个压缩列表中， 有多个连续的、长度介于 250 字节到 253 字节之间的节点 e1 至 eN。

因为 e1 至 eN 的所有节点的长度都小于 254 字节， 所以记录这些节点的长度只需要 1 字节长的 previous_entry_length 属性， 换句话说，e1 至 eN 的所有节点的 previous_entry_length 属性都是 1 字节长的。

如果我们将一个长度大于等于 254 字节的新节点 new 设置为压缩列表的表头节点， 那么 new 将成为 e1 的前置节点。


因为 e1 的 previous_entry_length 属性仅长 1 字节， 它没办法保存新节点 new 的长度， 所以程序将对压缩列表执行空间重分配操作， 并将e1 节点的 previous_entry_length 属性从原来的 1 字节长扩展为 5 字节长。

现在， 麻烦的事情来了 —— e1 原本的长度介于 250 字节至 253 字节之间， 在为 previous_entry_length 属性新增四个字节的空间之后， e1的长度就变成了介于 254 字节至 257 字节之间， 而这种长度使用 1 字节长的 previous_entry_length 属性是没办法保存的。

因此， 为了让 e2 的 previous_entry_length 属性可以记录下 e1 的长度， 程序需要再次对压缩列表执行空间重分配操作， 并将 e2 节点的previous_entry_length 属性从原来的 1 字节长扩展为 5 字节长。

正如扩展 e1 引发了对 e2 的扩展一样， 扩展 e2 也会引发对 e3 的扩展， 而扩展 e3 又会引发对 e4 的扩展……为了让每个节点的previous_entry_length 属性都符合压缩列表对节点的要求， 程序需要不断地对压缩列表执行空间重分配操作， 直到 eN 为止。

Redis 将这种在特殊情况下产生的连续多次空间扩展操作称之为“连锁更新”（cascade update）。

### 对象
-----------
Redis 使用对象来表示数据库中的键和值， 每次当我们在 Redis 的数据库中新创建一个键值对时， 我们至少会创建两个对象， 一个对象用作键值对的键（键对象）， 另一个对象用作键值对的值（值对象）。


	typedef struct redisObject {

	    // 类型
	    unsigned type:4;
	
	    // 编码
	    unsigned encoding:4;
	
	    // 指向底层实现数据结构的指针
	    void *ptr;
	
	    // ...
	
	} robj;


**类型**

对象的 type 属性记录了对象的类型

| 类型常量 | 对象的名称 |
| ------------- | ------------- |
| REDIS_STRING	| 字符串对象
| REDIS_LIST	| 列表对象
| REDIS_HASH	| 哈希对象
| REDIS_SET	| 集合对象
| REDIS_ZSET	| 有序集合对象

**编码**

对象的 ptr 指针指向对象的底层实现数据结构， 而这些数据结构由对象的 encoding 属性决定。通过 encoding 属性来设定对象所使用的编码， 而不是为特定类型的对象关联一种固定的编码， 极大地提升了 Redis 的灵活性和效率， 因为 Redis 可以根据不同的使用场景来为一个对象设置不同的编码， 从而优化对象在某一场景下的效率。

| 编码常量 |	编码所对应的底层数据结构 |
| ------------- | ------------- |
| REDIS_ENCODING_INT	| long 类型的整数
| REDIS_ENCODING_EMBSTR	| embstr 编码的简单动态字符串
| REDIS_ENCODING_RAW	| 简单动态字符串
| REDIS_ENCODING_HT	| 字典
| REDIS_ENCODING_LINKEDLIST |	双端链表
| REDIS_ENCODING_ZIPLIST	| 压缩列表
| REDIS_ENCODING_INTSET	| 整数集合
| REDIS_ENCODING_SKIPLIST	| 跳跃表和字典

| 对象所使用的底层数据结构	| 编码常量	| OBJECT ENCODING 命令输出
| ------------- | ------------- | ------------------ |
| 整数	| REDIS_ENCODING_INT	| "int"
| embstr 编码的简单动态字符串（SDS）	| REDIS_ENCODING_EMBSTR	| "embstr"
| 简单动态字符串	| REDIS_ENCODING_RAW	|"raw"
| 字典	| REDIS_ENCODING_HT	| "hashtable"
| 双端链表	| REDIS_ENCODING_LINKEDLIST	 | "linkedlist"
| 压缩列表	| REDIS_ENCODING_ZIPLIST	| "ziplist"
| 整数集合	| REDIS_ENCODING_INTSET 	| "intset"
| 跳跃表和字典	| REDIS_ENCODING_SKIPLIST	| "skiplist"

**字符串对象**

字符串对象的编码可以是 int 、 raw 或者 embstr 。

**列表对象**

列表对象的编码可以是 ziplist 或者 linkedlist 。

当列表对象可以同时满足以下两个条件时， 列表对象使用 ziplist 编码：

- 列表对象保存的所有字符串元素的长度都小于 64 字节；
- 列表对象保存的元素数量小于 512 个；

不能满足这两个条件的列表对象需要使用 linkedlist 编码。

注意以上两个条件的上限值是可以修改的， 具体请看配置文件中关于 list-max-ziplist-value 选项和 list-max-ziplist-entries 选项的说明。

**哈希对象**

哈希对象的编码可以是 ziplist 或者 hashtable 。

ziplist 编码的哈希对象使用压缩列表作为底层实现， 每当有新的键值对要加入到哈希对象时， 程序会先将保存了键的压缩列表节点推入到压缩列表表尾， 然后再将保存了值的压缩列表节点推入到压缩列表表尾， 因此：

- 保存了同一键值对的两个节点总是紧挨在一起， 保存键的节点在前， 保存值的节点在后；
- 先添加到哈希对象中的键值对会被放在压缩列表的表头方向， 而后来添加到哈希对象中的键值对会被放在压缩列表的表尾方向。

当哈希对象可以同时满足以下两个条件时， 哈希对象使用 ziplist 编码：

- 哈希对象保存的所有键值对的键和值的字符串长度都小于 64 字节；
- 哈希对象保存的键值对数量小于 512 个；

不能满足这两个条件的哈希对象需要使用 hashtable 编码。

注意这两个条件的上限值是可以修改的， 具体请看配置文件中关于 hash-max-ziplist-value 选项和 hash-max-ziplist-entries 选项的说明。


**集合对象**

集合对象的编码可以是 intset 或者 hashtable 。

hashtable 编码的集合对象使用字典作为底层实现， 字典的每个键都是一个字符串对象， 每个字符串对象包含了一个集合元素， 而字典的值则全部被设置为 NULL 。

当集合对象可以同时满足以下两个条件时， 对象使用 intset 编码：

- 集合对象保存的所有元素都是整数值；
- 集合对象保存的元素数量不超过 512 个；

不能满足这两个条件的集合对象需要使用 hashtable 编码。

**有序集合对象**

有序集合的编码可以是 ziplist 或者 skiplist 。


ziplist 编码的有序集合对象使用压缩列表作为底层实现， 每个集合元素使用两个紧挨在一起的压缩列表节点来保存， 第一个节点保存元素的成员（member）， 而第二个元素则保存元素的分值（score）。压缩列表内的集合元素按分值从小到大进行排序， 分值较小的元素被放置在靠近表头的方向， 而分值较大的元素则被放置在靠近表尾的方向。

zset 结构中的 zsl 跳跃表按分值从小到大保存了所有集合元素， 每个跳跃表节点都保存了一个集合元素： 跳跃表节点的 object 属性保存了元素的成员， 而跳跃表节点的 score 属性则保存了元素的分值。


 Redis 选择了**同时使用字典和跳跃表**两种数据结构来实现有序集合。这两种数据结构都会通过指针来共享相同元素的成员和分值， 所以同时使用跳跃表和字典来保存集合元素不会产生任何重复成员或者分值， 也不会因此而浪费额外的内存。

![](https://i.imgur.com/EXxjhor.png)


当有序集合对象可以同时满足以下两个条件时， 对象使用 ziplist 编码：

- 有序集合保存的元素数量小于 128 个；
- 有序集合保存的所有元素成员的长度都小于 64 字节；

不能满足以上两个条件的有序集合对象将使用 skiplist 编码。

## Redis数据库实现

### 数据库
----

Redis数据库服务器将所有数据库保存在服务器状态redis.h/redisServer结构的db数据汇总

	struct redisServer{
	//...
	// 保存服务器中所有数据库 数组
	redisDb *db;
	//	服务器数据库数量，默认为16
	int dbnum;
	
	};


Redis 是一个键值对（key-value pair）数据库服务器， 服务器中的每个数据库都由一个 redis.h/redisDb 结构表示， 其中， redisDb 结构的dict 字典保存了数据库中的所有键值对， 我们将这个字典称为键空间（key space）：

	typedef struct redisDb {

	    // ...
	
	    // 数据库键空间，保存着数据库中的所有键值对
	    dict *dict;
	
	    // ...
	
	} redisDb;

**键的生存时间**

- EXPIRE key ttl
- PEXPIRE key ttl
- EXPIREAT key timestamp
- PEXPIREAT key timestamp



**过期键删除策略**

- 定时删除
- 惰性删除
- 定期删除

redis服务器实际使用惰性删除和定期删除相结合来删除过期键。

惰性删除：对输入键进行检查，如果过期就删除键。

定期删除：redis服务器周期性执行activeExpireCycle函数，随机检查数据库中的键过期时间，并删除过期键。








### RDB持久化
----

### AOF持久化
---

　　AOF 则以协议文本的方式，将所有对数据库进行过写入的命令（及其参数）记录到 AOF
文件，以此达到记录数据库状态的目的。

#### AOF持久化实现

##### 命令追加
 　　在AOF持久化功能处于打开状态时，服务器在执行完一个写命令后，或以协议格式将被执行的写命令追加到服务器的aof——buf缓冲区末尾；

		struct redisServer{
			//....
			//aof缓冲区
			sds aod_buf;
		}

##### AOF文件的写入与同步

　　因为服务器在处理文件事件时可能会执行写命令， 使得一些内容被追加到 aof_buf 缓冲区里面， 所以在服务器每次结束一个事件循环之前， 它都会调用 flushAppendOnlyFile 函数， 考虑是否需要将 aof_buf 缓冲区中的内容写入和保存到 AOF 文件里面， 这个过程可以用以下伪代码表示：

	
	def eventLoop():

	    while True:
	
	        # 处理文件事件，接收命令请求以及发送命令回复
	        # 处理命令请求时可能会有新内容被追加到 aof_buf 缓冲区中
	        processFileEvents()
	
	        # 处理时间事件
	        processTimeEvents()
	
	        # 考虑是否要将 aof_buf 中的内容写入和保存到 AOF 文件里面
	        flushAppendOnlyFile()


　　flushAppendOnlyFile 函数的行为由服务器配置的 appendfsync 选项的值来决定， 各个不同值产生的行为如表 TABLE_APPENDFSYNC 所示。


![](https://i.imgur.com/3fjFb3a.png)

 　　如果用户没有主动为 appendfsync 选项设置值， 那么 appendfsync 选项的默认值为 everysec ， 关于 appendfsync 选项的更多信息， 请参考 Redis 项目附带的示例配置文件 redis.conf 。


#### AOF文件载入与数据还原

因为AOF文件包含重建数据库状态的所有写命令，所以服务器秩序重新执行AOF文件里保存的写命令。

Redis读取AOF文件并还原数据库状态如下：

1. 创建一个不带网络连接的为客户端
2. 从AOF文件分析并读取一条写命令
3. 使用客户端执行这条写命令
4. 重复2,3直到AOF文件所有写命令被处理

#### AOF重写

　　因为AOF持久化是通过保存被执行的写命令来记录数据库状态，所以随着服务器运行时间流逝，AOF文件中内容会越来越多，使用AOF文件来进行数据还原所需的时间越多。实际上ＡＯＦ文件重写是通过读取服务器当前数据库状态来实现的，而不是对现有的ＡＯＦ文件进行读取分析写入。这样能将对一个键的多个写命令替换为一个写命令。

　　因为AOF重写函数会进行大量的写入操作，如果调用这个函数会长时间阻塞，所以redis将aof重写程序放入到子进程执行。使用子进程同时会带来一个问题：在子进程进行AOF重写期间服务器继续处理了写请求，会导致服务器当前状态与重写后的AOF文件保存的数据库状态不一致。

　　为了解决这个问题，redis服务器设置了一个AOF重写缓冲区，当redis服务器在重写aof阶段，执行完一个写命令会同时将这个写命令发送给AOF缓冲区和AOF重写缓冲区。在子进程完成AOF重写工作后，父进程会调用程序将AOF重写缓冲区中内容写入到新AOF文件中。 


	

>  Redis设计与实现