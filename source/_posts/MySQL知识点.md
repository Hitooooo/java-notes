---
serializabletitle: MySQL知识点
date: 2020-04-09 00:22:23
categories: 
- database
tags: 
- sql
- 事务
---

# MySql

## 数据库设计范式

*  **第一范式（1NF）：要求数据库表的每一列都是不可分割的原子数据项。** 某列是地址，其实为了满足第一范式的话还是可以继续分割成省市区列。

*  **第二范式（2NF** ）： **在1NF的基础上，非码属性必须完全依赖于候选码（在1NF基础上消除非主属性对主码的部分函数依赖）** 

  如关系模型（职工号，姓名，职称，项目号，项目名称）中，职工号->姓名，职工号->职称，而项目号->项目名称。显然依赖关系不满足第二范式，常用的解决办法是差分表格，比如拆分为职工信息表和项目信息表。 

*  **第三范式（3NF）：在2NF基础上，任何非主属性不依赖于其它非主属性（在2NF基础上消除传递依赖）** 

  [知乎解释](https://www.zhihu.com/question/24696366)

<!--more-->

## 存储引擎

**常见的存储引擎及其区别**

* InnoDB：MySql默认的存储引擎，设计目标是面向在线事务处理的应用。

  * 特点是行锁设计、支持外键；
  * B+索引
  * 通过多版本并发控制来获得高并发性，提供插入缓存、二次写、自适用哈希索引和预读等高可用功能。
  * 并且实现了SQL的四种隔离级别，默认为**REPEATABLE**级别。
  * 对于表中的数据存储使用了聚集(clustered)方式，因此每张表都是按主键的顺序存放，如果没有设置主键，InnoDB会为每一行生成ROWID，并以此作为主键。

  {% asset_img innodb.jpg innodb %} 

* MyISAM：支持全文索引，主要面向查询服务的应用。***MySQL5.7开始InnoDB已经支持全文索引，而且效果更好***

  * 不支持事务。不是所有应用都需要事务支持，只是查询操作。
  * B+索引
  * 表锁设计、**不支持外键**
  * 只缓存索引文件，而不缓存数据文件，这点和大多数数据库都不同

  {% asset_img myisam.jpg innodb %}  

* Memory：表中数据存储在内存中，重启数据全部消失，适合存储临时表，使用Hash索引而不是B+索引。那为什么不使用这个代替Redis？Redis是为了MySQL减负的，而且数据易丢失不保证可用，需要手动持久化。

**InnoDB是如何实现RR（可重复读）的？**



**InnoDB为什么推荐使用自增ID作为主键**

自增ID可以保证每次插入时B+索引是从右边扩展的，可以避免B+树和频繁合并和分裂（对比使用UUID）。如果使用字符串主键和随机主键，会使得数据随机插入，效率比较差。

## 索引

### 数据结构

**二叉查找树**

一棵二叉树中，左子树的键值总是小于根节点，右子树的键值总是大于根节点，因此可以通过中序遍历获取键值的排序输出。但是如果源数列已经是有序的，构建的二叉查找树就会变成高度与数列长度相同的树，查找效率降为o(n)，为此引入平衡二叉树。

**平衡二叉树**

在二叉排序树的基础上，限制树的左右两个子树的**高度差不大于1.**平衡二叉树的查找性能是比较高的，但是最优二叉树的建立和维护需要大量的操作。

**B树**

1.  平衡二叉树节点最多有两个子树，而 B 树每个节点可以有多个子树，M 阶 B 树表示该树每个节点最多有 M 个子树 
2.  平衡二叉树每个节点只有一个数据和两个指向孩子的指针，而 B 树每个中间节点有 k-1 个关键字（可以理解为数据）和 k 个子树 （ k 介于阶数 M 和 M/2 之间，M/2 向上取整 ）
3.  B 树的所有叶子节点都在同一层，并且叶子节点只有关键字，指向孩子的指针为 null 
4.  和平衡二叉树相同的点在于：B 树的节点数据大小也是按照左小右大，子树与节点的大小比较决定了子树指针所处位置 

{% asset_img b-tree.png b-tree %}

   **B 树的每个节点可以表示的信息更多，因此整个树更加“矮胖”，这在从磁盘中查找数据（先读取到内存、后查找）的过程中，可以减少磁盘 IO 的次数，从而提升查找速度** 。

**B+树**

1.  节点的子树数和关键字数相同，（B 树是关键字数比子树数少一）  
2.  节点的关键字表示的是子树中的最大数，在子树中同样含有这个数据。根节点键值就是最大值。B树不允许键重复。
3.  叶子节点包含了全部数据，同时符合左小右大的顺序 

{% asset_img b+tree.png b-tree %}

相较于B树有什么优点？

1. B+中间节点只保存指针，相较于B树中间节点保存指针和记录， 可以大大增加每个节点存储的 key 值的数量，降低 B+ 树的高度。
2. 所有记录都保存在叶子节点，查找效率稳定
3. 叶子节点之间有指针相连，数据遍历时只需要对叶子节点遍历即可，B+非常适合范围查找

**那为什么MongoDB还选择用B树作为索引结构呢？**

综上可以发现B+最大的好处就是范围查找和遍历操作，MongoDB作为非关系型数据库期望单个Collection的设计和查询就能满足需求，不涉及多表操作，如果你的MongoDB频繁使用lookup，那就需要重写审视Collection的设计。MySQL是关系型数据库，很多情况下表与表之间存在很强的关系，**join操作十分常见。**而join操作就不可避免的需要遍历所有记录，而B+树能很好的支持遍历操作。

**[InnoDB索引最左匹配原则]( https://blog.csdn.net/sinat_41917109/article/details/88944290 )**

 索引的底层是一颗B+树，那么联合索引当然还是一颗B+树，只不过联合索引的健值数量不是一个，而是多个。构建一颗B+树只能根据一个值来构建，因此数据库依据联合索引最左的字段来构建B+树。
例子：假如创建一个（a,b)的联合索引，那么它的索引树是这样的 

{% asset_img b+tree_keys.png b+tree_keys %}


可以看到a的值是有顺序的，1，1，2，2，3，3，而b的值是没有顺序的1，2，1，4，1，2。所以b = 2这种查询条件没有办法利用索引，因为联合索引首先是按a排序的，b是无序的。

同时我们还可以发现在a值相等的情况下，b值又是按顺序排列的，但是这种顺序是相对的。所以最左匹配原则遇上范围查询就会停止，剩下的字段都无法使用索引。例如a = 1 and b = 2 a,b字段都可以使用索引，因为在a值确定的情况下b是相对有序的，而a>1and b=2，a字段可以匹配上索引，但b值不可以，因为a的值是一个范围，在这个范围中b是无序的。

最左匹配原则：最左优先，以最左边的为起点任何连续的索引都能匹配上。同时遇到范围查询(>、<、between、like)就会停止匹配。

 假如建立联合索引（a,b,c）

```mysql
# 用到了索引 where子句几个搜索条件顺序调换不影响查询结果，因为Mysql中有查询优化器，会自动优化查询顺序
select * from table_name where a = '1' and b = '2' and c = '3' 
select * from table_name where b = '2' and a = '1' and c = '3'
```

```mysql
# 都从最左边开始连续匹配，用到了索引
select * from table_name where a = '1' 
select * from table_name where a = '1' and b = '2'  
select * from table_name where a = '1' and b = '2' and c = '3'
```

```mysql
# 没有从最左边开始，最后查询没有用到索引，用的是全表扫描 
select * from table_name where  b = '2' 
select * from table_name where  c = '3'
select * from table_name where  b = '1' and c = '3' 
```

```mysql
# 不连续时，只用到了a列的索引，b列和c列都没有用到 
select * from table_name where a = '1' and c = '3' 
```

**匹配列前缀**

如果列是字符型的话它的比较规则是先比较字符串的第一个字符，第一个字符小的哪个字符串就比较小，如果两个字符串第一个字符相通，那就再比较第二个字符，第二个字符比较小的那个字符串就比较小，依次类推，比较字符串。

如果a是字符类型，那么前缀匹配用的是索引，后缀和中缀只能全表扫描了

```mysql
select * from table_name where a like 'As%'; //前缀都是排好序的，走索引查询
select * from table_name where  a like '%As'//全表查询
select * from table_name where  a like '%As%'//全表查询
```

 **匹配范围值** 

```mysql
# 可以对最左边的列进行范围查询
select * from table_name where  a > 1 and a < 3
```

```mysql
# 个列同时进行范围查找时，只有对索引最左边的那个列进行范围查找才用到B+树索引，也就是只有a用到索引，在1<a<3的范围内b是无序的，不能用索引，找到1<a<3的记录后，只能根据条件 b > 1继续逐条过滤
select * from table_name where  a > 1 and a < 3 and b > 1;
```

```mysql
# 左边的列是精确查找的，右边的列可以进行范围查找,a=1的情况下b是有序的，进行范围查找走的是联合索引
select * from table_name where  a = 1 and b > 3;
```

 **排序** 

一般情况下，我们只能把记录加载到内存中，再用一些排序算法，比如快速排序，归并排序等在内存中对这些记录进行排序，有时候查询的结果集太大不能在内存中进行排序的话，还可能暂时借助磁盘空间存放中间结果，排序操作完成后再把排好序的结果返回客户端。Mysql中把这种再内存中或磁盘上进行排序的方式统称为文件排序。文件排序非常慢，但如果order子句用到了索引列，就有可能省去文件排序的步骤

```mysql
# b+树索引本身就是按照上述规则排序的，所以可以直接从索引中提取数据，然后进行回表操作取出该索引中不包含的列就好了order by的子句后面的顺序也必须按照索引列的顺序给出，比如
select * from table_name order by a,b,c limit 10;
```

```mysql
# 这种颠倒顺序的没有用到索引
select * from table_name order by b,c,a limit 10;
```

```mysql
# 联合索引左边列为常量，后边的列排序可以用到索引
select * from table_name where a =1 order by b,c limit 10;
```

### 索引类型

索引类型针对的是字段

* FULLTEXT 全文索引，类似倒排索引
* NORMAL  仅加速查询 
* UNIQUE   索引列的值必须唯一，但允许有空值。如果是组合索引，则列值的组合必须唯一。 
* SPATIAL   对 空间数据类型的字段建立的索引，MYSQL中的空间数据类型有4种，分别是GEOMETRY、POINT、LINESTRING、POLYGON 

### 索引方法

索引方法即实现索引的方法

* B+树：
* Hash索引，MySQL采用链表法处理Hash冲突。
  *  Hash索引仅仅能满足"=","IN"和"<=>(安全等于)"查询，不能使用范围查询 
  *  Hash索引无法被用来避免数据的排序操作。因为Hash值的大小关系并不一定和Hash运算前的键值完全一样 
  *  Hash索引不能利用部分索引键查询。对于组合索引，Hash索引在计算Hash值的时候是组合索引键合并后再一起计算Hash值，而不是单独计算Hash值，所以通过组合索引的前面一个或几个索引键进行查询的时候，Hash索引也无法被利用 
  *  Hash索引遇到大量Hash值相等的情况后性能并不一定就会比B+树索引高 

## 锁

锁机制用于管理对共享资源的并发访问

### InnoDB中的锁

#### 锁的类型

**按粒度分为：**

1. 表锁： mysql锁中粒度最大的一种锁，表示当前的操作对整张表加锁，资源开销比行锁少（实现逻辑简单），不会出现死锁的情况，但是发生锁冲突的概率很大。 

2. 行锁： 行锁的是mysql锁中粒度最小的一种锁，因为锁的粒度很小，所以发生资源争抢的概率也最小，并发性能最大，但是也会造成死锁，每次加锁和释放锁的开销也会变大。 

   InnoDB实现了两种标准的行级锁：

   * 共享锁（S锁）： 若事务A对数据对象1加上S锁，则事务A可以读数据对象1但不能修改，其他事务只能再对数据对象1加S锁，而不能加X锁，直到事务A释放数据对象1上的S锁。这保证了其他事务可以读数据对象1，但在事务A释放数据对象1上的S锁之前不能对数据对象1做任何修改。 
   * 排他锁（X锁）： 若事务A对数据对象1加上X锁，事务A可以读数据对象1也可以修改数据对象1，其他事务不能再对数据对象1加任何锁，直到事务A释放数据对象1上的锁。这保证了其他事务在事务A释放数据对象1上的锁之前不能再读取和修改数据对象1。 

   |      |   X    |   S    |
   | :--: | :----: | :----: |
   |  X   | 不兼容 | 不兼容 |
   |  S   | 不兼容 |  兼容  |

**为解决不同粒度决锁共存问题，引入意向锁**

意向共享锁（IS）：事务想要在获得表中某些记录的共享锁，需要在表上先加意向共享锁。

意向互斥锁（IX）：事务想要在获得表中某些记录的互斥锁，需要在表上先加意向互斥锁。

意向锁是**表级别锁**。 当我们需要给一个加表锁的时候，我们需要根据意向锁去判断表中有没有数据行被锁定，以确定是否能加成功。如果意向锁是行锁，那么我们就得遍历表中所有数据行来判断。如果意向锁是表锁，则我们直接判断一次就知道表中是否有数据行被锁定了。所以说将意向锁设置成表级别的锁的性能比行锁高的多。 

作用就是：  当一个事务在需要获取资源的锁定时，如果该资源已经被排他锁占用，则数据库会自动给该事务申请一个该表的意向锁。如果自己需要一个共享锁定，就申请一个意向共享锁。如果需要的是某行（或者某些行）的排他锁定，则申请一个意向排他锁。 

|      | IS     | IX     | S      | X      |
| ---- | ------ | ------ | ------ | ------ |
| IS   | 兼容   | 兼容   | 兼容   | 不兼容 |
| IX   | 兼容   | 兼容   | 不兼容 | 不兼容 |
| S    | 兼容   | 不兼容 | 兼容   | 不兼容 |
| X    | 不兼容 | 不兼容 | 不兼容 | 不兼容 |

从表中可以发现：

* 意向锁之间是兼容的
* 意向锁和非意向锁之间的兼容性与行锁之间一致。IS-X=S-X
* 意向锁不会阻塞全表扫以外的任何请求

#### 一致性非锁定读

> MySQL如何做到读写并行？多版本并发控制(Multi Version Concurrency Control, MVVC)

一致性的非锁定读是指**InnoDB存储引擎通过多版本控制方法来读取当前执行时间数据库中行的数据。**之所以叫做非锁定读，因为不需要等待行锁X的释放。

{% asset_img mvvc.png mvvc %}

快照数据是该行的之前版本数据，实现是通过undo段完成。undo用来回滚事务，因此快照数据本身是没有额外开销的。既然是历史数据，到底读取哪个时间点的数据呢？

1. RC模式下，读取最新的快照。因为可以读其他事务的提交，期望尽可能新。
2. RR模式下，读取**当前事务**开始时的快照。可重复读，认为读取开始时的快照能满足要求。

#### 一致性锁定读

显示地对读操作加锁

```mysql
SELECT ... FOR UPDATE
```

```mysql
SELECT ... LOCK IN SHARE MODE
```

### 锁的算法

InnoDB有三种行锁的算法：

* Record Lock：单个行记录上的锁。
* Gap Lock：间隙锁，锁定一个范围，但不包含记录本身
* Next-Key Lock：Gap Lock + Record Lock，锁定一个范围并且锁定记录本身

```mysql
CREATE TABLE z( a INT, b INT, PRIMARY KEY(a), KEY(B));
INSERT INTO z SELECT 1,1;
INSERT INTO z SELECT 3,1;
INSERT INTO z SELECT 5,3;
INSERT INTO z SELECT 7,6;
INSERT INTO z SELECT 10,8;
```

```MYSQL
# 事务A执行以下语句
SELECT * FROM z WHERE b=3 FOR UPDATE;# b对应的a列加RecordLock，b因为是辅助索引加Next-key lock

# 事务B运行下面的语句
SELECT * FROM z WHERE a = 5 LOCK IN SHARE MODE; # 显然阻塞，因为A中b=3对应的a=5行已经被锁定
INSERT INTO z SELECT 4,2; # 还是阻塞。主键不阻塞，但是辅助索引因为next-key lock (1,3)(3,6)两个范围都被锁定了
INSERT INTO z SELECT 6,5;# 阻塞。理由同上

INSERT INTO z SELECT 2,0;# 立即执行。主键和辅助索引都没有被锁
```

通过以上语句可以发现，Gap Lock的作用就是阻止多个事务将记录插入到同一范围。

**那next-key lock什么时候使用Record Lock？**

对于唯一键值的锁定，Next-Key Lock降级为Record Lock仅存在于**查询所有的唯一索引列。**若唯一索引由多个列组成，而查询仅是查询多列中的其中一个，那么查询是Range查询，故仍使用Next-Key Lock进行锁定。

**MySQL在RR隔离级别就能解决幻读，如何实现的？**

```mysql
# 表t主键为a列
# 事务A
BEGIN
SELECT * FROM t WHERE a > 2 FOR UPDATE
COMMIT
# 事务B
BEGIN
INSERT INTO t SELECT 4;
COMMIT
```

在RR级别下，A->B->A 发生幻读。但是InnoDB会对(2,+∞)范围加X锁，事务B是无法插入的，解决了幻读（**同一事务下，连续执行两次相同的SQL语句可能导致不同的结果，第二次SQL可能返回之前不存在的行**）问题。

### 死锁

死锁是指两个或两个以上的事务在执行过程中，因争夺锁资源而造成的一种相互等待的现象。如无外力作用，事务都将无法进行下去。

#### 案例

> id为主键 token辅助索引

1. 不同表相同记录行锁冲突

   | 事务A                                              | 事务B                                              |
   | -------------------------------------------------- | -------------------------------------------------- |
   | delete from table_1 where id = 1;                  |                                                    |
   |                                                    | udpate msg set message='订单' where token = 'asd'; |
   | update msg set message='订单' where token = 'asd'; |                                                    |
   |                                                    | delete from table_1 where id = 1;                  |

2. 相同表记录行锁冲突

   | 事务A                             | 事务B                             |
   | --------------------------------- | --------------------------------- |
   | update from table_1 where id = 1; |                                   |
   |                                   | update form table_1 where id = 2; |
   | update from table_1 where id = 2; |                                   |
   |                                   | update from table_1 where id = 1; |

#### MySQL如何解决死锁问题

1. 最简单的办法，超时回滚。当两个事务相互等待时，当一个等待时间超过设定阈值，其中一个事务回滚，另一个等待事务就能继续运行。仅仅超时处理，那么回滚的总是先开始事务的那个，如果该事务操作复杂，回滚占用很多undo log，显然是不合理的。
2. wait-for graph（等待图）事务之间的资源请求关系用一张有向图来表示，如果图中出现回路就存在死锁。InnoDB会选择回滚undo量最小的事务。

#### Developer如何避免死锁

1. 以固定的顺序访问表和行。案例2中两个事务都按照{1,2}顺序就不会发生死锁
2. 大事务拆成小事务。大事务更倾向于死锁，如果业务允许，将大事务拆小
3. 同一个事务中，尽可能做到一次锁定所需的所有资源
4. 为表添加合理的索引。可以看到如果不走索引将会为表的每一行记录添加上锁，死锁的概率大大增大

## MySQL事务

### 事务特性

事务是访问数据库的一个操作序列，保证了用户的每一次操作都是可靠的，即使出现了异常情况，也不会破坏后台数据的完整性。这就要求事务满足以下四个特性：ACID

* Automic：原子性。事务是不可分割的工作单位，要么都做要么都不做。
* Consistency：一致性。事务将数据库从一种状态转换到另一种一致状态。事务开始之前和之后，数据库的完整性约束没有被打破。例如事务在失败回滚时，不能在唯一列插入重复值
* Isolation：隔离性。事务提交前，对其他事务不可见
* durability：持久性。事务一旦提交成功，其结果是永久性的。即使宕机或其他故障也是能恢复的。

### 控制语句

- **BEGIN** 开始一个事务
- **ROLLBACK** 事务回滚
- **COMMIT** 事务确认

### 并发一致性问题

1. **更新丢失**：后一个事务覆盖前一个事务的结果。
   1. 事务1将行r更新为v1，但未提交
   2. 事务2将行r更新为v2，但未提交
   3. 事务1提交
   4. 事务2提交
2. **脏读**： 事务B读取到了事务A已修改但尚未提交的的数据。脏读隔离看似毫无用处，但是在不需要精确性的查询时，可以大大提高性能
3. **不可重复读**：在一个事务内，多次读同一个数据。在这个事务还没有结束时，另一个事务也访问该同一数据。那么，在第一个事务的两次读数据之间。由于第二个事务的修改，那么第一个事务读到的数据可能不一样，这样就发生了在一个事务内两次读到的数据是不一样的，因此称为不可重复读，即原始读取不可重复。 **一个事务范围内两个相同的查询却返回了不同数据** 
4. **幻读：** 一个事务T1按相同的查询条件重新读取以前检索过的数据，却发现其他事务T2插入了满足其查询条件的新数据，这种现象就称为“幻读” 

#### 数据库提供一定的隔离级别解决上述问题

|     隔离级别     |   读数据一致性   | 脏读 | 不可重复读 | 幻读 |
| :--------------: | :--------------: | :--: | :--------: | :--: |
| Read Uncommitted |     最低级别     |  Y   |     Y      |  Y   |
|  Read Committed  |      语句级      |  X   |     Y      |  Y   |
| Reapeatable Read |      事务级      |  X   |     X      |  Y   |
|   Serializable   | 最好级别，事务级 |  X   |     X      |  X   |

## 常见面试题

