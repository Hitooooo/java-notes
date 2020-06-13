---
title: Redis
date: 2020-06-13 21:17:20
categories:
- database
tags:
- Interview
---

>  Redis is an open source (BSD licensed), in-memory data structure store, used as a database, cache and message broker.  Redis has built-in replication, Lua scripting, LRU eviction, transactions and different levels of on-disk persistence, and provides high availability.

<!--more-->

## 一、 基础

####  为什么用**Redis**，而不是Map或Guava的Cache

* Redis可以做分布式缓存和远程缓存，满足高可用的需求，而Map只能缓存在本地，重启数据消失，Cache不能做到分布式
* Redis有缓存过期机制，Map没有
* Redis可多个服务器共享，但是Map不行

####  Redis原理

1. **Redis为什么是单线程**  

   因为CPU不是Redis的瓶颈。Redis的瓶颈最有可能是机器内存或者网络带宽

2. **如果万一CPU成为你的Redis瓶颈了，或者，你就是不想让服务器其他核闲置，那怎么办**

   那也很简单，你多起几个Redis进程就好了。Redis是keyvalue数据库，又不是关系数据库，数据之间没有约束。只要客户端分清哪些key放在哪个Redis进程上就可以了。redis-cluster可以帮你做的更好。 

3. **单线程模型** 

   Redis客户端对服务端的每次调用都经历了发送命令，执行命令，返回结果三个过程。其中执行命令阶段，由于Redis是单线程来处理命令的，所有每一条到达服务端的命令不会立刻执行，所有的命令都会进入一个队列中，然后逐个被执行。并且多个客户端发送的命令的执行顺序是不确定的。但是可以确定的是不会有两条命令被同时执行，不会产生并发问题，这就是Redis的单线程基本模型。

4. **单线程模型每秒万级别处理能力的原因** 

   * 纯内存访问。 数据存放在内存中，内存的响应时间大约是 100纳秒 ，这是Redis每秒万亿级别访问的重要基础。
   * 非阻塞I/O ，Redis采用**epoll**做为I/O多路复用技术的实现 ，再加上Redis自身的事件处理模型将epoll中的连接，读写，关闭都转换为了时间，不在I/O上浪费过多的时间。
   * 单线程 避免了线程切换和竞态产生的消耗 。
   * Redis采用单线程模型，**每条命令执行如果占用大量时间**， 会造成其他线程阻塞，对于Redis这种高性能服务是致命的，所以Redis是面向高速执行的数据库。

### 1.1 Redis的数据结构与应用场景

#### String

 最基本的数据类型，一个key对应一个value . String类型是二进制安全的，意思是 redis 的 string 可以包含任何字符，不会像C语言中将\0作为字符的结束判断。

1.缓存： 经典使用场景，把常用信息，字符串，图片或者视频等信息放到redis中，redis作为缓存层，mysql做持久化层，降低MySQL的读写压力。

2.计数器：redis是单线程模型，一个命令执行完才会执行下一个，同时数据可以一步落地到其他的数据源。

3.session：常见方案spring session + redis实现session共享 

#### Hash

 一个Mapmap，指值本身又是一种键值对结构

{% asset_img redis-map-map.png b-tree %}

![Hash结构](.\Redis\redis-map-map.png)

场景： 缓存，能直观，相比string更节省空间的维护缓存信息，如用户信息，视频信息等 

#### List 链表

 redis 使用双端链表实现List， 左右两边都能进行插入和删除数据 ，配合不同的操作可是实现不同的数据结构。

- lpush+lpop=Stack(栈)

- lpush+rpop=Queue（队列）

- lpush+ltrim=Capped Collection（有限集合）

- lpush+brpop（阻塞操作）=Message Queue（消息队列）

  ![img](.\Redis\redis-list.png)
  
  {% asset_img redis-list.png redis-list %}  

#### Set 集合

无序不重复的字符串列表。

1. 不允许有重复的元素，
2. 集合中的元素是无序的，不能通过索引下标获取元素，
3. 支持集合间的操作，可以取多个集合取交集、并集、差集。 

 ![img](E:\belogs\JavaNote\source\_posts\Redis\redis-set.png)

 {% asset_img redis-set.png redis-set%}  

应用：标签集合。一个用户的多种标签，协同过滤

#### ZSet 有序集合

在Set的基础之上，加入了**有序**功能，排序根据得分（手动设置）

 {% asset_img redis-zset.png redis-zset %} 

 ![img](E:\belogs\JavaNote\source\_posts\Redis\redis-zset.png) 

应用：排行榜。

 [避免缓存击穿的利器之BloomFilter](https://juejin.im/post/5db69365518825645656c0de) 

## 二、进阶

#### 如果有大量的key需要设置同一时间过期（缓存雪崩），一般需要注意什么？

 如果大量的key过期时间设置的过于集中，到过期的那个时间点，**Redis**可能会出现短暂的卡顿现象。严重的话会
出现缓存雪崩，我们一般需要在时间上加一个随机值，使得过期时间分散一些。 

``` shell
setRedis（Key，value，time + Math.random() * 10000）
```



#### Redis分布式锁

 拿**setnx**来争抢锁，抢到之后，再用**expire**给锁加一个过期时间防止锁忘记了释放。 

如果在setnx之后执行expire之前进程意外crash或者要重启维护了，那会怎么样? 同时把**setnx**和**expire**合成一条指令 。其实自己实现分布式锁很容易出现问题的，市场上有很多开源实现，如Redisson.可以保证获取锁的客户端保证完成业务逻辑后再释放锁，通过WathcDog。而且支持分布式Redis集群模式，但是由于主从同步问题可能导致锁数据同步失败，可以考虑用Redis自带的Redlock算法提高可用率。

#### Redis里面有1亿个key，其中有10w个key是以某个固定的已知的前缀开头的，如何将它们全部找出来

 使用**keys**指令可以扫出指定模式的key列表 。这个redis正在给线上的业务提供服务，那使用keys指令会有什么问题？Redis的单线程的。keys指令会导致线程阻塞一段时间，线上服务会停顿，直到指令执行完毕，服务才能恢复。这个时候可以使用**scan**指令，**scan**指令可以无阻塞的提取出指定模式的key列表，但是会有一定的重复概率，在客户端做一次去重就可以了，但是整体所花费的时间会比直接用keys指令长.

#### 使用过Redis做异步队列么，你是怎么用的

 一般使用list结构作为队列，**rpush**生产消息，**lpop**消费消息。当lpop没有消息的时候，要适当sleep一会再重试。 

#### Redis是怎么持久化的？

RDB做镜像全量持久化，AOF做增量持久化。因为RDB会耗费较长时间，不够实时，在停机的时候会导致大量丢失数据，所以需要AOF来配合使用。在redis实例重启时，会使用RDB持久化文件重新构建内存，再使用AOF重放近期的操作指令来实现完整恢复重启之前的状态。

RDB: 生成多个数据文件，每个数据文件分别都代表了某一时刻**Redis**里面的数据M适合做**冷备**，完整的数据运维设置定时任务，定时同步到远端的服务器. **RDB**对**Redis**的**性能**影响非常小，是因为在同步数据的时候他只是**fork**了一个子进程去做持久化的，而且他在数据恢复的时候速度比**AOF**来的快。RDB都是快照文件，都是默认五分钟甚至更久的时间才会生成一次，AOF是秒级别的，数据完整性差。

AOF: **AOF**是一秒一次去通过一个后台的线程`fsync`**记录操作**，那最多丢这一秒的数据。  **AOF**在对日志文件进行操作的时候是以`append-only`的方式去写的，日志文件写入快速。 一样的数据，**AOF**文件比**RDB**还要大。 

#### Redis的同步机制，服务主从数据怎么交互的？

Redis可以使用主从同步，从从同步。第一次同步时，主节点做一次**bgsave**，并同时将后续修改操作记录到内存buffer，待完成后将RDB文件全量同步到复制节点，复制节点接受完成后将RDB镜像加载到内存。加载完成后，再通知主节点将期间修改的操作记录同步到复制节点进行重放就完成了同步过程。后续的增量数据通过AOF日志同步即可。

#### 是否使用过Redis集群，集群的高可用怎么保证，集群的原理是什么

**Redis Cluster** 着眼于扩展性，在单个redis内存不足时，使用Cluster进行分片存储。 Master是主，Slave是从，Master具有读写权限，Slave只有读权限 ，提高访问性能。

**Redis Sentinal** 哨兵模式。在集群的基础上多引入若干个哨兵节点，哨兵监控Master可用性，Master失效则从Slave中重新选举出新的Master。着眼于高可用，在master宕机时会自动将slave提升为master，继续提供服务。

集群监控：负责监控 Redis master 和 slave 进程是否正常工作。

消息通知：如果某个 **Redis** 实例有故障，那么哨兵负责发送消息作为报警通知给管理员。

故障转移：如果 master node 挂掉了，会自动转移到 slave node 上。

配置中心：如果故障转移发生了，通知 client 客户端新的 master 地址。

#### 那你了解缓存穿透和击穿么，可以说说他们跟雪崩的区别么

1. 缓存击穿：强调的是一点点某个点的击穿这一动作。某个Key对于的热点数据，频繁被访问，然后这个Key突然过期，然后导致请求全部落到数据库。雪崩指的是某一时刻大量的热点数据同时失效。 设置热点数据永远不过期，或者牺牲性能加上互斥锁。
2. 缓存穿透：每次请求的数据都不在Redis缓存中，需要直接访问MySQL。如使用恶意不存的用户ID访问某接口。处理：接口校验。[避免缓存击穿的利器之BloomFilter](https://juejin.im/post/5db69365518825645656c0de) 。

#### Redis的过期策略和内存淘汰机制

**过期策略**: 定期删除+惰性删除。

定期删除，指的是redis默认是每隔100ms就随机抽取一些设置了过期时间的key，检查其是否过期，如果过期就删除。注意，这里可不是每隔100ms就遍历所有的设置过期时间的key，那样就是一场性能上的灾难。实际上redis是每隔100ms随机抽取一些key来检查和删除的。

 **惰性删除：** 在你获取某个key的时候，redis会检查一下 ，这个key如果设置了过期时间那么是否过期了？如果过期了此时就会删除，不会给你返回任何东西。 

 redis的内存占用过多的时候，此时会进行内存淘汰 ：

noeviction：当内存不足以容纳新写入数据时，新写入操作会报错，这个一般没人用吧

**allkeys-lru：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的key（这个是最常用的）**

allkeys-random：当内存不足以容纳新写入数据时，在键空间中，随机移除某个key，这个一般没人用吧

volatile-lru：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的key（这个一般不太合适）

volatile-random：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个key

volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的key优先移除

#### 手写LRU算法

1. 基于LinkedHashMap实现.构造函数的第三个参数设为True，重写removeEldestEntry方法，满了返回True。

```java
final int cacheSize = 100;
Map<String, String> map = new LinkedHashMap<String, String>((int) Math.ceil(cacheSize / 0.75f) + 1, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, String> eldest) {
    	return size() > cacheSize;
    }
};
```

2. 手动用过 双向链表加HashMap实现 [146. LRU缓存机制](https://leetcode-cn.com/problems/lru-cache/)

   ```java
   class LRUCache {
     private int capacity;
       private HashMap<Integer, Node> cache;
       private DoubleList doubleList;
   
       public LRUCache(int capacity) {
           // 1. 初始化双向链表和HashMap
           cache = new HashMap<>(capacity);
           doubleList = new DoubleList();
           this.capacity = capacity;
       }
   
       public int get(int key) {
           // 存在，返回并将Node移动到链表头
           // 不存在，返回-1
           int value = -1;
           if (cache.keySet().contains(key)) {
               Node node = cache.get(key);
               doubleList.moveToHead(node);
               value = node.val;
           }
           return value;
       }
   
       public void put(int key, int value) {
           // 存在更新值，不存在判断是否能添加
           Node old = cache.get(key);
           if (old == null) {
               // 满了，删除最后一个，没满，添加到头
               Node newNode = new Node(key, value);
               if (doubleList.size() == capacity) {
                   Node last = doubleList.removeLast();
                   cache.remove(last.key);
               }
               cache.put(key, newNode);
               doubleList.add(newNode);
           } else {
               old.val = value;
               doubleList.moveToHead(old);
           }
       }
   
       /**
        * 双向链表，remove addFirst removeLast
        */
       public class DoubleList {
   
           // 引入头尾结点同意操作
           private Node head;
   
           private Node tail;
   
           private int size = 0;
   
           public DoubleList() {
               head = new Node(0, 0);
               tail = new Node(0, 0);
               head.next = tail;
               head.pre = tail;
               tail.next = head;
               tail.pre = head;
           }
   
           // head tail 是手动加的，默认不会删除
           public void remove(Node node) {
               if (isEmpty()) {
                   return;
               }
               node.pre.next = node.next;
               node.next.pre = node.pre;
               size -= 1;
           }
   
           // 新增结点放在头部
           public void add(Node node) {
               head.next.pre = node;
               node.next = head.next;
               head.next = node;
               node.pre = head;
               size += 1;
           }
   
           public void moveToHead(Node node) {
               remove(node);
               add(node);
           }
   
           public Node removeLast() {
               if (isEmpty()) {
                   return null;
               }
               Node temp = tail.pre;
               remove(temp);
               return temp;
           }
   
           public int size() {
               return size;
           }
   
           public boolean isEmpty() {
               return size == 0;
           }
       }
   
       /**
        * 双向链表中的结点定义
        */
       public class Node {
           public int key;
           public int val;
           public Node next;
           public Node pre;
   
           public Node(int key, int val) {
               this.key = key;
               this.val = val;
           }
       }
   }
   ```
