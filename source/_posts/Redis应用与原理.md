---
title: Redis应用与原理
date: 2020-10-10 23:35:14
tags: Redis 数据库
---

# Redis基础

### Redis基本数据类型
redis一共有8种数据类型
String，List，Set，Zset，Hash，HyperLogLog，Geo，Streams

### String类型

setnx：设置值，如果key存在则不成功，基于此可以实现分布式锁
如果释放锁失败，导致其他节点永远获取不到锁，可以设置过期时间，单独使用expire，无法保证原子性，使用多参数命令
`set key value [expiration EX seconds|PX milliseconds][NX|XX]`
`set lock 1 EX 10 NX`

##### 存储（实现）原理
- 字符串类型的内部编码
1. int，存储8个字节的长整型（long，2^63-1）
2. embstr，代表embstr格式的SDS（Simple Dynamic String 简单动态字符串），存储小于44个字节的字符串
3. raw，存储大于44个字节的字符串（最长能存储512M）

- redis为什要用SDS来存储字符串
C语言本身没有字符串类型（只能用字符数组char[]来实现）
1. 使用字符数组必须先给目标变量分配足够的空间，否则可能溢出
2. 如果要获取字符串的长度，需要遍历字符数组，时间复杂度是 O(n)
3. C字符串长度改变，会重新分配字符数组的内存
4. 从字符数组开始，到遇到第一个“\0”来标记一个字符串的结束，因此不能保存图片，音频，视频，压缩文件等二进制保存的内容，二进制不安全。

SDS的特点：
1. 不需要担心内存溢出问题，如果需要会对SDS进行扩容
2. 获取字符串的长度的时间复杂度是 O(1)，因为定义了len属性
3. 通过“空间预分配“（sdsMakeRoomFor）和“惰性空间释放”，防止多次重分配内存
4. 判断是否结束的标志是len属性，可以包含"\0"

- embstr和raw的区别
embstr的使用只分配一次内存空间（因为redisObject和SDS是连续的），而raw需要分配两次内存空间（分别为redisObject和SDS分配空间），因此与raw相比，embstr的好处是创建时少分配一次空间，删除时，少释放一次空间，对象的所有数据存放在一起，查找方便。而embstr的坏处也很明显，如果字符串的长度增加，需要重新分别内存时，整个redisObject和SDS都需要重新分配空间，因此redis里的embstr实现为只读的。

- int和embstr什么时候转化成raw
当int数据不再是整数，或大小超过了long的范围（2^63-1）时，自动转换成raw
对于embstr，由于是只读的，当修改embstr的值，都会转化成raw，再进行修改。因此只要是修改embstr对象，修改后的对象一定是raw的，无论是否达到44个字节。

- 为什么要对底层的数据结构进行一次封装呢
通过封装，可以根据对象的类型动态的选择存储的数据结构和可以使用的命令，时间节约空间和优化查询速度


##### 应用场景
1. 缓存
String类型
例如：热点数据缓存，对象缓存，全页缓存。 可以提高热点数据的访问速度
2. 数据共享分布式
String类型，因为redis是分布式的独立服务，可以在多个应用中共享
例如：分布式session
```xml
<dependency>
    <groupId>org.springframework.session<groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
``` 
3. 分布式锁
String类型的setnx方法，只有不存在时才能添加成功。
4. 全局ID
int类型，incrby方法，利用原子性
5. 计数器
int类型， incrby方法，例如：文章阅读量，点赞数，允许一定的延迟，先写入redis，再定时同步到数据库
6. 限流
int类型，incyby方法，以访问者的IP和其他信息作为key，访问一次，增加一次计数，超过次数返回false。
7. 位统计
String类型的bitcount，因为bit非常节省空间（1M=8388608 bit），可以用来做大数据量统计，例如：在线用户统计，留存用户统计

### Hash哈希
包含键值对的无序列表。value只能存字符串，不能嵌套其他类型
- 同样是存储字符串，Hash和String的区别？
1. 把所有相关的值聚集到一个key中，节省内存空间
2. 只使用一个key，减少key冲突
3. 当需要批量获取值的时候，只需要一个命令，减少内存/IO/CPU的消耗
- Hash不适合的场景
1. field不能单独设置过期时间
2. 没有bit操作
3. 需要考虑数据量分布的问题（value非常大的时候，无法分布到多个节点）

##### 存储（实现）原理
Redis的Hash本身也是一个KV的结构，类似于Java的hashmap。
外层的哈希（Redis KV实现）只用到了hashtable。当存储hash数据时，我们把它叫内层的哈希。内层的哈希可以使用两种数据结构实现：
ziplist：OBJ_ENCODING_ZIPLIST（压缩列表）
hashtable：OBJ_ENCODING_HT（哈希表）

ziplist是一个经过特殊编码的双向链表，它不存储指向上一个链表节点和指向下一个链表节点的指针，而是存储上一个节点长度和当前节点长度，通过牺牲部分读写性能，来换取高效的内存空间利用率，是一种时间换空间的思想。只用在字段个数少，字段值小的场景里面。
`<zlbytes><zltail><zllen><entry><entry>...<entry><zlend>`
```cpp
typedef struct zlentry {
    unsigned int prevrawlensize; /*上一个链表节点占用的长度*/
    unsigned int prevrawlen;     /*存储上一个链表节点的长度数值所需要的字节数*/
    unsigned int lensize;        /*存储当前链表节点长度数值所需要的字节数*/
    unsigned int len;            /*当前链表节点占用的长度*/
    unsigned int headersize;     /*当前链表节点的头部大小（prevrawlensize+lensize），即非数据域的大小*/
    unsigned char encoding;      /*编码方式*/
    unsigned char *p;            /*压缩链表以字符串的形式保存，该指针指向当前节点起始位置*/
}zlentry;
```

- 什么时候用ziplist存储
当hash对象同时满足以下两个条件的时候，使用ziplist编码：
1. 所有的键值对的键和值的字符串长度都小于等于64byte（一个字节）
2. 哈希对象保存的键值对数量小于512个

在Redis中，hashtable被称为字典（dictionary），它是一个数组+链表的结构

```cpp
typedef struct dictEntry {
    void *key;/*key关键字定义*/
    union {
        void*val;uint64_tu64;/*value定义*/i
        nt64_ts64;doubled;
    }v;
    struct dictEntry *next;/*指向下一个键值对节点*/
}dictEntry;
```

```cpp
typedef struct dictht {
    dictEntry **table;/*哈希表数组*/
    unsigned longsize;/*哈希表大小*/
    unsigned longsizemask;/*掩码大小，用于计算索引值。总是等于size-1*/
    unsigned longused;/*已有节点数*/
}dictht;
```

```cpp
typedef struct dict {
    dictType *type;/*字典类型*/
    void *privdata;/*私有数据*/
    dictht ht[2];/*一个字典有两个哈希表*/
    long rehashidx;/*rehash索引*/
    unsigned long iterators;/*当前正在使用的迭代器数量*/
}dict;
```
- 为什么要定义两个哈希表呢？ ht[2]
Redis的hash默认使用的是ht[0]，ht[1]不会初始化和分配空间
哈希表dichht是用链地址法来解决碰撞问题的，在这种情况下，哈希表的性能取决于他的大小（size属性）和它保存的节点数量（used属性）之间的比率
    - 比率在1:1时（一个哈希表ht只存储一个节点entry），哈希表的性能最好
    - 如果节点数量比哈希表的大小要大很多的话（这个比例用ratio表示，5表示平均5一个ht存储5个entry），那么哈希表就会退化成多个链表，哈希表本身的性能优势就不再存在

在这种情况下需要扩容。Redis里面的这种操作叫做rehash
rehash的步骤：
1. 为字符ht[1]哈希表分配空间，这个哈希表的空间大小取决于要执行的操作，以及ht[0]当前包含的键值对的数量。扩展：ht[1]的大小为第一个大于等于ht[0].used*2。
2. 将所有的ht[0]上的节点rehash到ht[1]上，重新计算hash值和索引，然后放入指定的位置。
3. 当ht[0]全部迁移到了ht[1]之后，释放ht[0]的空间，将ht[1]设置为ht[0]表，并创建新的ht[1]，为下次rehash做准备。




# Redis原理

### Rdis高级特性，发布订阅，事务，lua脚本

### Redis底层原理，单线程工作机制，内存回收，持久化


# Redis分布式

### Redis主从配置与原理

### Redis哨兵机制

### Redis分布式各种方案对比



# Redis实战

### Java客户端实现 Pipline 和分布式锁方法和原理

### Redis数据一致性问题和解决方法

### Redis高并发下各种问题解决方案

