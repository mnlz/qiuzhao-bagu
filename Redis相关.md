# Redis相关

## 1.Redis的数据结构

**首先Redis的常用的几种数据结构：5+3**

String、List、Hash、Set、ZSet、BitMap、GEO、hyperloglog

String 类似 java中的Map，存储的就是k，v 有个加减1的操作

List是双端列表，比如微信的公众号消息或者评论

Hash相当于Java的Map<String,Map<Objeetc,Object>>，value是一个对象，比如如果实现一个类似购物车的功能

Set：不允许重复，可以随机弹出元素

ZSet：每个元素都会有一个分数，可以排名

BitMap：存储的01 二进制数组

hyperloglog：用于一些基数统计（不包含）DAU 日活

GEO地理位置：

- 将三维的地球变为二维的坐标
- 在将二维的坐标转换为一维的点块
- 最后将一维的点块转换为二进制再通过base32编码

**其次，每种数据类型其实在redis的底层是不区分的，有一个共同的RedisObject对象**

<img src=".\Redis相关.assets\image-20240901151334018.png" alt="image-20240901151334018" style="zoom: 50%;" />

redisObject的结构体里面，有一个type字段，用来区分string、list、hash、set、zset

其中还有一个 *ptr 指针，这个指针指向的是真正存储数据的地方

**下面依次说一下几种数据类型的真正数据结构**

String类型，String有三种方式存储

- int 保存在long的64位范围内的整数

- embstr 数据就存储在ptr指针的下面 字符串长度小于44

- SDS（简单动态字符串） ptr指向指向的就是SDS的结构体

  <img src=".\Redis相关.assets\image-20240901152005465.png" alt="image-20240901152005465" style="zoom: 25%;" />

Hash类型，有两种存储方法：压缩列表（ziplist）和哈希表（HashTable）

[Redis：hash类型底层数据结构剖析-阿里云开发者社区 (aliyun.com)](https://developer.aliyun.com/article/1346861)

压缩列表本质上就是字节数组，一堆01，会将存储的内容编码

<img src=".\Redis相关.assets\image-20240901152455513.png" alt="image-20240901152455513" style="zoom:25%;" />

主要是通过记录上一个entry的长度，通过长度的计算获取下一个元素，不是使用链表的pre和next

HashTable的底层数据结构：字典

ptr指向会指向一个字典

**每个哈希表节点保存一个键值对，每个哈希表由多个哈希表节点构成，而字典则是对哈希表的进一步封装**。

<img src="E:\aaa找工作\秋招\Redis相关.assets\image-20240901152956085.png" alt="image-20240901152956085" style="zoom: 50%;" />

其中，dict代表字典，dictht代表哈希表，dictEntry代表哈希表节点。可以看出，dictEntry是一个数组，这很好理解，因为一个哈希表里要包含多个哈希表节点。而dict里包含2个dictht，多出的哈希表用于REHASH。

List的底层数据结构：quckList快速链表 LinkedList + zipList

<img src=".\Redis相关.assets\image-20240901162245016.png" alt="image-20240901162245016" style="zoom:33%;" />

Set的底层数据结构：inset和hashTable

inyset存储整数的结构体

hashtable字典

Zset的数据结构：zipList和skipList

<img src="E:\aaa找工作\秋招\Redis相关.assets\image-20240901162905504.png" alt="image-20240901162905504" style="zoom: 33%;" />

利用类似二分查找，建立冗余索引，时间复杂度为logn

为什么不使用B+树：

- 设计的目的不同：MySQL中，为了减少磁盘的IO次数，需要存储的树尽量的低，三层b+树是可以索引千万级别的数据，b+树的插入和删除都会涉及树的分裂和合并，redis就是在内存中，不在意查询的次数，不需要考虑树的平衡，需要简单的设计即可
- 占用空间：B+树的指针管理和实现都比跳表复杂
- 1，时间复杂度方面：大部分情况下，跳跃表的效率和平衡树媲美；
  2，算法实现方面：跳跃表的实现比平衡树、b+树更为简单；
  <img src="https://i-blog.csdnimg.cn/blog_migrate/30de678fc87cf5c454562518bc8e0d00.png" alt="image-20230219184543704" style="zoom:50%;" />

## 2.