![思维导图](https://pic4.zhimg.com/v2-b6b408dc996bfa47a89dd421ff27171b_r.jpg)


![](https://mmbiz.qpic.cn/mmbiz_png/Z0fxkgAKKLNc1rZcnHlq151m3KPG6SrM1dLia006cQRb0ngOjmpV4kglmGpQTJ2wBMJTQMR8xVMnzGCD8fjYtIg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



# 基础知识

## 数据库范式





# 存储引擎



# 索引



## 为什么用B+树, 不用B树

因为B+比B树更适合实际应用中操作系统的文件索引和数据库索引. 

1. B+树的磁盘读写代价更低
   B+的内部结点并**没有指向关键字具体信息的指针** 。因此其**内部结点相对B树更小**。如果把所有同一内部结点的关键字存放在同一盘块中，那么盘块所能容纳的关键字数量也越多。一次性读入内存中的需要查找的关键字也就越多。相对来说IO读写次数也就降低了。

2. B+tree的查询效率更加稳定
   由于非终结点并不是最终指向文件内容的结点，而只是叶子结点中关键字的索引。所以任何关键字的查找必须走一条从根结点到叶子结点的路。所有关键字查询的路径长度相同，导致每一个数据的查询效率相当。

3. B树只适合**随机检索**，而B+树同时支持**随机检索和顺序检索**；

4. 增删文件时，效率更高
因为B+树的叶子节点包含所有关键字，并以有序的链表结构存储，这样可很好提高增删效率。
B+树的叶子节点有一条链相连，而B树的叶子节点各自独立。
由于B+树的内部节点只存放键，不存放值，因此，一次读取，可以在内存页中获取更多的键，有利于更快地缩小查找范围。
B+树的叶节点由一条链相连，因此，当需要进行一次全数据遍历的时候，B+树只需要使用O(logN)时间找到最小的一个节点，然后通过链进行O(N)的顺序遍历即可。
而B树则需要对树的每一层进行遍历，这会需要更多的内存置换次数，因此也就需要花费更多的时间


##  回表查询

使用explain查看sql语句的执行计划.

在说到什么是回表查询的时候，有两个概念需要先解释清楚：分别是聚集索引(聚簇索引)和非聚集索引(非聚簇索引)

### 聚集索引(cluster)和非聚集索引(uncluster)

MySQL规定，在使用InnoDB存储引擎的时候，必须**有且仅有一个**聚集索引(cluster)，非聚集索引(uncluster)也就是普通索引就看自己设置的有多少个了

* 聚类索引: 索引结构和数据一起存放的索引。**主键索引属于聚集索引**

    * 优点：聚集索引的查询速度非常的快，因为整个B+树本身就是一颗多叉平衡树，叶子节点也都是有序的，定位到索引的节点，就相当于定位到了数据。
    * 缺点：1. 依赖于有序的数据，不是有序的数据的话，插入或查找的速度肯定比较慢。2. 更新代价大。

* 非聚类索引: 索引结构和数据分开存放的索引。叶子节点存的是键值和数据所在物理地址

    * 优点：更新代价比聚集索引要小 。
    * 缺点：1. 依赖于有序的数据，不是有序的数据的话，插入或查找的速度肯定比较慢。2. 可能会二次查询(回表)，当查到索引对应的指针或主键后，可能还需要根据指针或主键再到数据文件或表中查询。

* 聚集索引和非聚集索引的区别：
    1. 聚集索引(cluster)中的非叶子节点存储的是表的主键, 非聚集索引(uncluster)的非叶子节点存储的是自己设置的索引字段对应的值(如果是联合索引，那就是联合索引的几个字段对应的值)
    2. 聚集索引(cluster)的叶子节点，存储着当前表中每条记录的所有信息；非聚集索引(uncluster)的叶子节点，只存储当前记录对应的主键ID(也就是聚集索引的非叶子节点存储的值)

左边的是主键索引(聚集索引)，右边的是普通索引(非聚集索引)

![img](https://img-blog.csdnimg.cn/20200926083503245.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NQTEFTRl8=,size_16,color_FFFFFF,t_70#pic_center)

### 回表

如果是通过**非主键索引**进行查询，select所要获取的字段**不能**通过非主键索引获取到，需要通过非主键索引获取到的主键，**从聚集索引再次查询一遍**，获取到所要查询的记录，这个查询的过程就是回表

通过执行计划的Extra字段的值，可以看到当前sql是否进行了回表
如果Extra字段值为null，我不太确定是否是一定走回表的(待确认)
但是如果Extra为Using index,则表示当前查询，通过索引覆盖就可以获取到当前select要的值，无需通过回表查询记录

### 验证

```sql
create TABLE hui_biao_test (id int not null auto_increment,name varchar(20),age int(11),first_name varchar(20),CONSTRAINT pk_id PRIMARY KEY(id));

create index index_name on `hui_biao_test`(name);
```

```sql
explain select id,name from `hui_biao_test` where name = '张三';

explain select name from `hui_biao_test` where name = '张三';

explain select name,age from `hui_biao_test` where name = '张三';
```

下面截图分别是这三个sql的执行计划，可以发现：

第一个sql和第二个sql的Extra是Using index，说明前两个sql是索引覆盖，无需回表即可查询到当前要用的数据；
但是第三个sql就不可以，因为有一个age字段，在name这个索引中，没有age字段的信息，只有id和name（name索引对应的B+树中的非叶子节点就是name字段的值，id就是叶子节点存储的元素），所以需要通过回表，去主键索引查询到对应的记录，然后获取到对应的age属性
![img1](https://img-blog.csdnimg.cn/20200925163154788.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NQTEFTRl8=,size_16,color_FFFFFF,t_70#pic_center)

![img22](https://img-blog.csdnimg.cn/20200925163348338.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NQTEFTRl8=,size_16,color_FFFFFF,t_70#pic_center)

![img3](https://img-blog.csdnimg.cn/20200925163408874.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NQTEFTRl8=,size_16,color_FFFFFF,t_70#pic_center)

假如说我把name和age设置为联合索引，那最后一个sql的执行计划就会有变化

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200925163953427.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NQTEFTRl8=,size_16,color_FFFFFF,t_70#pic_center)

可以看到，这个sql的执行计划中Extra就是Using index

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200925164004890.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NQTEFTRl8=,size_16,color_FFFFFF,t_70#pic_center)

说明当前联合索引的非叶子节点中，存储的是联合索引对应的信息；仅通过联合索引就可以获取到select所需要的信息，这样，就无须再经过主键索引查询了

---

## 索引失效

https://segmentfault.com/a/1190000021464570













# MySQL事务
>如何解决幻读?
>
>隔离级别有哪些, 每个级别有什么问题, 怎么解决. MySQL默认隔离级别
>
>如何通过枷锁的方式解决幻读

## 锁和事务的关系

1. 事务具有ACID（原子性、一致性、隔离性和持久性），锁是用于解决隔离性的一种机制。
2. 事务的隔离级别通过锁的机制来实现。另外锁有不同的粒度，同时事务也是有不同的隔离级别的(一般四种：读未提交Read uncommitted， 读已提交Read committed， 可重复读Repeatable read， 可串行化Serializable)。
3. 开启事务就自动加锁。

## 事务的基本知识

### 概念

事务是逻辑上的一组操作，要么都执行，要么都不执行。它满足 ACID 特性的一组操作，可以通过 Commit 提交一个事务，也可以使用 Rollback 进行回滚。主要用于处理操作量大，复杂度高的数据。比如说，在人员管理系统中，你删除一个人员，你即需要删除人员的基本资料，也要删除和该人员相关的信息，如信箱，文章等等，这样，这些数据库操作语句就构成一个事务！

经典例子: 转账。A要给B转账1000元，这个转账会涉及到两个关键操作就是：将A的余额减少1000元，将B的余额增加1000元。万一在这两个操作之间突然出现错误, 比如银行系统崩溃，导致A余额减少而B的余额没有增加，这样就不对了。事务就是保证这两个关键操作要么都成功，要么都要失败。

Auto Commit：MySQL 默认采用自动提交模式。也就是说，如果不显式使用`START TRANSACTION`语句来开始一个事务，那么每个查询操作都会被当做一个事务并自动提交。

### 事务的特性ACID

#### 特性

1. 原子性（Atomicity), 事务被视为不可分割的最小单元，事务的所有操作要么全部提交成功，要么全部失败回滚(RollBack). 回滚可以用回滚日志（Undo Log）来实现，回滚日志记录着事务所执行的修改操作，在回滚时反向执行这些修改操作即可。
2. 一致性（Consistency） 数据库在执行事务前后，数据都保持一致, 在一致性状态下，所有事务对同一个数据的读取结果都是相同的。例如转账业务中，无论事务是否成功，转账者和收款人的总额应该是不变的；
3. 隔离性（Isolation）并发访问数据库时，一个用户的事务不被其他事务所干扰，各并发事务之间数据库是独立的；一个事务所做的修改在最终提交以前，对其它事务是不可见的。
4. 持久性（Durability）一个事务被提交之后。它对数据库中数据的改变是持久的，即使数据库发生故障也不应该对其有任何影响。系统发生崩溃可以用重做日志（Redo Log）进行恢复，从而实现持久性。与回滚日志记录数据的逻辑修改不同，重做日志记录的是数据页的物理修改。

#### ACID之间的关系

![](https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/image-20191207210437023.png)

这几个特性不是一种平级关系

- 只有满足一致性，事务的执行结果才是正确的。

- 在无并发的情况下，事务串行执行，隔离性一定能够满足。此时只要能满足原子性，就一定能满足一致性。

- 在并发的情况下，多个事务并行执行，事务不仅要满足原子性，还需要满足隔离性，才能满足一致性。

- 事务满足持久化是为了能应对系统崩溃的情况。

### AutoCommit

MySQL 默认采用自动提交模式。也就是说，如果不显式使用`START TRANSACTION`语句来开始一个事务，那么每个查询操作都会被当做一个事务并自动提交

## 并发事务的并发一致性问题 -> 隔离级别

### 问题分类

- 更新丢失（Lost Update)：T1 和 T2 两个事务都对一个数据进行修改，T1 先修改并提交生效，T2 随后修改，T2 的修改覆盖了 T1 的修改。指在一个事务读取一个数据时，另外一个事务也访问了该数据，那么在第一个事务中修改了这个数据后，第二个事务也修改了这个数据。这样第一个事务内的修改结果就被丢失，因此称为丢失修改。例如：事务1读取某表中的数据A=20，事务2也读取A=20，事务1修改A=A-1，事务2也修改A=A-1，最终结果A=19，事务1的修改被丢失.

  <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/image-20191207221744244.png" style="zoom:33%;" />

- 脏读(Dirty Read)：读脏数据指在不同的事务下，当前事务可以读到另外事务未提交的数据。例如：T1 修改一个数据但未提交，T2 随后读取这个数据。如果 T1 撤销了这次修改(rollback)，那么 T2 读取的数据是脏数据。

  <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/image-20191207221920368.png" style="zoom:33%;" />

- 不可重复读（NonRepeatable Reads)：在一个事务内多次读取同一数据集合。在这一事务还未结束前，另一事务也访问了该同一数据集合并做了修改，由于第二个事务的修改，第一次事务的两次读取的数据可能不一致。例如：T2 读取一个数据，T1 对该数据做了修改。如果 T2 再次读取这个数据，此时读取的结果和第一次读取的结果不同。

  <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/image-20191207222102010.png" style="zoom:33%;" />

- 幻读（Phantom Reads)：幻读与不可重复读类似。它发生在一个事务A读取了几行数据，接着另一个并发事务B插入了一些数据时。在随后的查询中，事务A就会发现多了一些原本不存在的记录，就好像发生了幻觉一样，所以称为幻读。

### 幻读和不可重复读的区别

- **不可重复读的重点是修改**：在同一事务中，同样的条件，第一次读的数据和第二次读的数据不一样。（因为中间有其他事务提交了修改）
- **幻读的重点在于新增或者删除**：在同一事务中，同样的条件,，第一次和第二次读出来的记录数不一样。（因为中间有其他事务提交了插入/删除）
- 不可重复读重点在于update和delete，而幻读的重点在于insert

例1（同样的条件, 你读取过的数据, 再次读取出来发现值不一样了 ）：事务1中的A先生读取自己的工资为     1000的操作还没完成，事务2中的B先生就修改了A的工资为2000，导        致A再读自己的工资时工资变为  2000；这就是不可重复读。

例2（同样的条件, 第1次和第2次读出来的记录数不一样 ）：假某工资单表中工资大于3000的有4人，事务1读取了所有工资大于3000的人，共查到4条记录，这时事务2 又插入了一条工资大于3000的记录，事务1再次读取时查到的记录就变为了5条，这样就导致了幻读。

---

如果使用锁机制来实现这两种隔离级别，在可重复读中，该sql第一次读取到数据后，就将这些数据加锁，其它事务无法修改这些数据，就可以实现可重复读了。但这种方法却无法锁住insert的数据，所以当事务A先前读取了数据，或者修改了全部数据，事务B还是可以insert数据提交，这时事务A就会发现莫名其妙多了一条之前没有的数据，这就是幻读，不能通过行锁来避免。需要**Serializable**隔离级别 ，**读用读锁，写用写锁，读锁和写锁互斥**，这么做可以有效的**避免幻读**、不可重复读、脏读等问题，但会**极大的降低数据库的并发能力**。

所以说**不可重复读和幻读最大的区别，就在于如何通过锁机制来解决他们产生的问题**。

上文说的，是使用**悲观锁机制**来处理这两种问题，但是MySQL、ORACLE、PostgreSQL等成熟的数据库，出于性能考虑，都是使用了以**乐观锁为理论基础的MVCC（多版本并发控制）**来避免这两种问题。


### 并发事务处理带来的问题的解决办法：

> 产生并发不一致性问题的主要原因是破坏了事务的隔离性，解决方法是通过**并发控制来保证隔离性**。并发控制可以通过封锁来实现，但是封锁操作需要用户自己控制，相当复杂。数据库管理系统提供了事务的隔离级别，让用户以一种更轻松的方式处理并发一致性问题。-> 隔离级别也是通过锁来实现的

- “更新丢失”通常是应该完全避免的。但防止更新丢失，并不能单靠数据库事务控制器来解决，需要应用程序对要更新的数据加必要的锁来解决，因此，防止更新丢失应该是应用的责任。

- “脏读” 、 “不可重复读”和“幻读” ，其实都是数据库读一致性问题，必须由数据库提供一定的事务隔离机制来解决：

- - 一种是加锁：在读取数据前，对其加锁，阻止其他事务对数据进行修改。
  - 另一种是数据多版本并发控制（MultiVersion Concurrency Control，简称 **MVCC** 或 MCC），也称为多版本数据库：不用加任何锁， 通过一定机制生成一个数据请求时间点的一致性数据快照 （Snapshot)， 并用这个快照来提供一定级别 （语句级或事务级） 的一致性读取。从用户的角度来看，好象是数据库可以提供同一数据的多个版本。

### MySQL的隔离级别

在数据库操作中，为了有效保证并发读取数据的正确性，提出的事务隔离级别。我们的数据库锁，也是为了构建这些隔离级别存在的。

**SQL 标准定义了四个隔离级别：**

- **READ-UNCOMMITTED(读取未提交)：** 最低的隔离级别，允许读取尚未提交的数据变更，**可能会导致脏读、幻读或不可重复读**。
- **READ-COMMITTED(读取已提交)：** 允许读取并发事务已经提交的数据，**可以阻止脏读，但是幻读或不可重复读仍有可能发生**。
- **REPEATABLE-READ(可重复读)：**  对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，**可以阻止脏读和不可重复读，但幻读仍有可能发生**。
- **SERIALIZABLE(可串行化)：** 最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，**该级别可以防止脏读、不可重复读以及幻读**。

| 隔离级别                     | 脏读（Dirty Read） | 不可重复读（NonRepeatable Read） | 幻读（Phantom Read） |
| :--------------------------- | :----------------- | :------------------------------- | :------------------- |
| 未提交读（Read uncommitted） | 可能               | 可能                             | 可能                 |
| 已提交读（Read committed）   | 不可能             | 可能                             | 可能                 |
| 可重复读（Repeatable read）  | 不可能             | 不可能                           | 可能                 |
| 可串行化（Serializable ）    | 不可能             | 不可能                           | 不可能               |

- 未提交读(Read Uncommitted)：允许脏读，也就是可能读取到其他会话中未提交事务修改的数据
- 提交读(Read Committed)：只能读取到已经提交的数据。Oracle等多数数据库默认都是该级别 (不重复读)
- 可重复读(Repeated Read)：可重复读。在同一个事务内的查询都是事务开始时刻一致的，MySQL的InnoDB默认级别。在SQL标准中，该隔离级别消除了不可重复读，但是还存在幻象读
- 串行读(Serializable)：完全串行化的读，每次读都需要获得表级共享锁，读写相互都会阻塞, 读用读锁, 写用写锁

查看当前数据库的事务隔离级别: `show variables like 'tx_isolation'`

```mysql
# 查看事务隔离级别 5.7.20 之后
show variables like 'transaction_isolation';
SELECT @@transaction_isolation

# 5.7.20 之后
SELECT @@tx_isolation
show variables like 'tx_isolation'

+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| tx_isolation  | REPEATABLE-READ |
+---------------+-----------------+
```

## MySQL锁机制
>MySQL 间隙锁有没有了解，死锁有没有了解，写一段会造成死锁的 sql 语句，死锁发生了如何解决，MySQL 有没有提供什么机制去解决死锁
> 数据库的乐观锁和悲观锁？
>MySQL 中有哪几种锁，列举一下？
>MySQL中InnoDB引擎的行锁是怎么实现的？

<img src="https://mmbiz.qpic.cn/mmbiz_png/J0g14CUwaZdBuf5m7spQh3GeDNHF7jQDwKsLJfxCEapwyBScZ0IquHg9UEDnhS65bQ0nnS5fDS9SNfAyQSjwuw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" style="zoom: 43%;" />

在并发环境下，事务的隔离性很难保证，因此会出现很多并发一致性问题。
当数据库有并发事务的时候，可能会产生数据的不一致，这时候需要一些机制来保证访问的次序，锁机制就是这样的一个机制。
锁的作用：用于管理对共享资源的并发访问，保证数据库的完整性和一致性











### Next-Key Locks

MVCC 不能解决幻读问题，Next-Key Locks 就是为了解决这个问题而存在的。在可重复读（REPEATABLE READ-RR）隔离级别下，使用 MVCC + Next-Key Locks 可以解决幻读问题。

> Next-Key锁 = record lock行锁 + gap lock（间隙锁）。
>
> 行锁防止别的事务修改或删除，GAP锁防止别的事务新增，行锁和GAP锁结合形成的的Next-Key锁共同解决了RR级别在写数据时的幻读问题。













## MVCC(多版本并发控制)

### 基本思想

多版本并发控制（Multi-Version Concurrency Control, MVCC）是 MySQL 的 InnoDB 存储引擎实现隔离级别的一种具体方式，用于实现**提交读(RC)**和**可重复读(RR)**这两种隔离级别。***解决脏读和不可重复读, 但是不能解决幻读***

> 幻读解决: 在可重复读（REPEATABLE READ）隔离级别下，使用 MVCC + Next-Key Locks 可以解决幻读问题

而**未提交读隔离级别(RU)**总是读取最新的数据行，要求很低，无需使用 MVCC。

可**串行化隔离级别(Serializable)**需要对所有读取的行都加锁，单纯使用 MVCC 无法实现

在锁一节中可以知道, 加锁能解决多个事务同时执行时出现的并发一致性问题。在实际场景中读操作往往多于写操作，因此又引入了读写锁来避免不必要的加锁操作，例如读和读没有互斥关系。读写锁中读和写操作仍然是互斥的，

> 而 MVCC 利用了**多版本**的思想，**写操作更新最新的版本快照**，而**读操作去读旧版本快照**，没有互斥关系，这一点和 *CopyOnWrite* 类似。

在 MVCC 中事务的**修改**操作（DELETE、INSERT、UPDATE）会为数据行新增一个版本快照。

脏读和不可重复读最根本的原因是事务读取到其它事务未提交的修改。在事务进行读取操作时，为了解决脏读和不可重复读问题，MVCC 规定**只能读取已经提交的快照**。当然一个事务可以读取自身未提交的快照，这不算是脏读

### 版本号

- 系统版本号 SYS_ID：是一个递增的数字，每开始一个新的事务，系统版本号就会自动递增。
- 事务版本号 TRX_ID ：事务开始时的系统版本号。

### Undo 日志

MVCC 的多版本指的是多个版本的快照，**快照存储在 Undo 日志中**，该日志通过`回滚指针 ROLL_PTR` 把一个数据行的所有快照连接起来。

例如在 MySQL 创建一个表 t，包含主键 id 和一个字段 x。我们先插入一个数据行，然后对该数据行执行两次更新操作。

```Mysql
INSERT INTO t(id, x) VALUES(1, "a");
UPDATE t SET x="b" WHERE id=1;
UPDATE t SET x="c" WHERE id=1;
```

因为没有使用 `START TRANSACTION` 将上面的操作当成一个事务来执行，根据 MySQL 的 AUTOCOMMIT 机制，每个操作都会被当成一个事务来执行，所以上面的操作总共涉及到三个事务。快照中除了记录事务版本号 TRX_ID 和操作之外，还记录了一个 bit 的 DEL 字段，用于标记是否被删除。

![](https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/image-20191208164808217.png)

INSERT、UPDATE、DELETE 操作会创建一个日志，并将事务版本号 TRX_ID 写入。DELETE 可以看成是一个特殊的 UPDATE，还会额外将 DEL 字段设置为 1。

### ReadView

MVCC 维护了一个 ReadView 结构，主要包含了当前系统未提交的事务列表 TRX_IDs {TRX_ID_1, TRX_ID_2, ...}，还有该列表的最小值 TRX_ID_MIN 和 TRX_ID_MAX。

<img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/image-20191208171445674.png" style="zoom: 50%;" />

在进行 SELECT 操作时，根据数据行快照的 TRX_ID 与 TRX_ID_MIN 和 TRX_ID_MAX 之间的关系，从而判断数据行快照是否可以使用：

- TRX_ID < TRX_ID_MIN，表示该数据行快照时在当前所有未提交事务之前进行更改的，因此可以使用。
- TRX_ID > TRX_ID_MAX，表示该数据行快照是在事务启动之后被更改的，因此不可使用。
- TRX_ID_MIN <= TRX_ID <= TRX_ID_MAX，需要根据隔离级别再进行判断：
  - 提交读：如果 TRX_ID 在 TRX_IDs 列表中，表示该数据行快照对应的事务还未提交，则该快照不可使用。否则表示已经提交，可以使用。
  - 可重复读：都不可以使用。因为如果可以使用的话，那么其它事务也可以读到这个数据行快照并进行修改，那么当前事务再去读这个数据行得到的值就会发生改变，也就是出现了不可重复读问题。

在数据行快照不可使用的情况下，需要沿着 Undo Log 的回滚指针 ROLL_PTR 找到下一个快照，再进行上面的判断

### 快照读与当前读

#### 快照读

MVCC 的 SELECT 操作是快照中的数据，不需要进行加锁操作。

```sql
SELECT * FROM table ...;
```

#### 当前读

MVCC 其它会对数据库进行修改的操作（INSERT、UPDATE、DELETE）需要进行加锁操作，从而读取最新的数据。可以看到 MVCC 并不是完全不用加锁，而只是避免了 SELECT 的加锁操作。

```sql
INSERT;
UPDATE;
DELETE;
```

在进行 SELECT 操作时，可以强制指定进行加锁操作。以下第一个语句需要加 S 锁，第二个需要加 X 锁。

```sql
SELECT * FROM table WHERE ? lock in share mode;
SELECT * FROM table WHERE ? for update;
```





## 如何解决幻读

## 幻读产生的原因

幻读: 幻读与不可重复读类似。它发生在一个事务A读取了几行数据，接着另一个并发事务B插入了一些数据时。在随后的查询中，事务A就会发现多了一些原本不存在的记录，就好像发生了幻觉一样，所以称为幻读

**行锁只能锁住行，但是新插入记录这个动作，要更新的是记录之间的“间隙”。因此，为了解决幻读问题，InnoDB只好引入新的锁，也就是间隙锁(Gap Lock)**



### MVCC + Next-key-Lock 防止幻读

`InnoDB`存储引擎在 RR (repeatable read)级别下通过 `MVCC`和 `Next-key Lock` 来解决幻读问题：

**1、执行普通 `select`，此时会以 `MVCC` 快照读的方式读取数据**

在快照读的情况下，RR 隔离级别只会在事务开启后的第一次查询生成 `Read View` ，并使用至事务提交。所以在生成 `Read View` 之后其它事务所做的更新、插入记录版本对当前事务并不可见，实现了可重复读和防止快照读下的 “幻读”

**2、执行 select...for update/lock in share mode、insert、update、delete 等当前读**

在当前读下，读取的都是最新的数据，如果其它事务有插入新的记录，并且刚好在当前事务查询范围内，就会产生幻读！`InnoDB` 使用 [Next-key Lock  (opens new window)](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-next-key-locks) 来防止这种情况。当执行当前读时，会锁定读取到的记录的同时，锁定它们的间隙，防止其它事务在查询范围内插入数据。只要我不让你插入，就不会发生幻读














---

# 读写分离









# 分库分表





---

# 一条SQL语句在MySQL执行过程
分析一个 sql 语句在 MySQL 中的执行流程，包括 sql 的查询在 MySQL 内部会怎么流转，sql 语句的更新是怎么完成的。

在分析之前我会先带着你看看 MySQL 的基础架构，知道了 MySQL 由那些组件组成以及这些组件的作用是什么，可以帮助我们理解和解决这些问题。

## 一 MySQL 基础架构分析

### 1.1 MySQL 基本架构概览

下图是 MySQL  的一个简要架构图，从下图你可以很清晰的看到用户的 SQL 语句在 MySQL 内部是如何执行的。

先简单介绍一下下图涉及的一些组件的基本作用帮助大家理解这幅图，在 1.2 节中会详细介绍到这些组件的作用。

- **连接器：** 身份认证和权限相关(登录 MySQL 的时候)。
- **查询缓存：** 执行查询语句的时候，会先查询缓存（MySQL 8.0 版本后移除，因为这个功能不太实用）。
- **分析器：** 没有命中缓存的话，SQL 语句就会经过分析器，分析器说白了就是要先看你的 SQL 语句要干嘛，再检查你的 SQL 语句语法是否正确。
- **优化器：** 按照 MySQL 认为最优的方案去执行。
<!-- - **执行器：** 执行语句，然后从存储引擎返回数据。 -->

![](https://guide-blog-images.oss-cn-shenzhen.aliyuncs.com/javaguide/13526879-3037b144ed09eb88.png)

简单来说 MySQL  主要分为 Server 层和存储引擎层：

- **Server 层**：主要包括连接器、查询缓存、分析器、优化器、执行器等，所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图，函数等，还有一个通用的日志模块 binglog 日志模块。
- **存储引擎**： 主要负责数据的存储和读取，采用可以替换的插件式架构，支持 InnoDB、MyISAM、Memory 等多个存储引擎，其中 InnoDB 引擎有自有的日志模块 redolog 模块。**现在最常用的存储引擎是 InnoDB，它从 MySQL 5.5.5 版本开始就被当做默认存储引擎了。**

### 1.2 Server 层基本组件介绍

#### 1) 连接器

连接器主要和身份认证和权限相关的功能相关，就好比一个级别很高的门卫一样。

主要负责用户登录数据库，进行用户的身份认证，包括校验账户密码，权限等操作，如果用户账户密码已通过，连接器会到权限表中查询该用户的所有权限，之后在这个连接里的权限逻辑判断都是会依赖此时读取到的权限数据，也就是说，后续只要这个连接不断开，即时管理员修改了该用户的权限，该用户也是不受影响的。

#### 2) 查询缓存(MySQL 8.0 版本后移除)

查询缓存主要用来缓存我们所执行的 SELECT 语句以及该语句的结果集。

连接建立后，执行查询语句的时候，会先查询缓存，MySQL 会先校验这个 sql 是否执行过，以 Key-Value 的形式缓存在内存中，Key 是查询预计，Value 是结果集。如果缓存 key 被命中，就会直接返回给客户端，如果没有命中，就会执行后续的操作，完成后也会把结果缓存起来，方便下一次调用。当然在真正执行缓存查询的时候还是会校验用户的权限，是否有该表的查询条件。

MySQL 查询不建议使用缓存，因为查询缓存失效在实际业务场景中可能会非常频繁，假如你对一个表更新的话，这个表上的所有的查询缓存都会被清空。对于不经常更新的数据来说，使用缓存还是可以的。

所以，一般在大多数情况下我们都是不推荐去使用查询缓存的。

MySQL 8.0 版本后删除了缓存的功能，官方也是认为该功能在实际的应用场景比较少，所以干脆直接删掉了。

#### 3) 分析器

MySQL 没有命中缓存，那么就会进入分析器，分析器主要是用来分析 SQL 语句是来干嘛的，分析器也会分为几步：

**第一步，词法分析**，一条 SQL 语句有多个字符串组成，首先要提取关键字，比如 select，提出查询的表，提出字段名，提出查询条件等等。做完这些操作后，就会进入第二步。

**第二步，语法分析**，主要就是判断你输入的 sql 是否正确，是否符合 MySQL 的语法。

完成这 2 步之后，MySQL 就准备开始执行了，但是如何执行，怎么执行是最好的结果呢？这个时候就需要优化器上场了。

#### 4) 优化器 

优化器的作用就是它认为的最优的执行方案去执行（有时候可能也不是最优，这篇文章涉及对这部分知识的深入讲解），比如多个索引的时候该如何选择索引，多表查询的时候如何选择关联顺序等。

可以说，经过了优化器之后可以说这个语句具体该如何执行就已经定下来。

#### 5) 执行器

当选择了执行方案后，MySQL 就准备开始执行了，首先执行前会校验该用户有没有权限，如果没有权限，就会返回错误信息，如果有权限，就会去调用引擎的接口，返回接口执行的结果。

## 二 语句分析 

### 2.1 查询语句

说了以上这么多，那么究竟一条 sql 语句是如何执行的呢？其实我们的 sql 可以分为两种，一种是查询，一种是更新（增加，更新，删除）。我们先分析下查询语句，语句如下：

```sql
select * from tb_student  A where A.age='18' and A.name=' 张三 ';
```

结合上面的说明，我们分析下这个语句的执行流程：

* 先检查该语句是否有权限，如果没有权限，直接返回错误信息，如果有权限，在 MySQL8.0 版本以前，会先查询缓存，以这条 sql 语句为 key 在内存中查询是否有结果，如果有直接缓存，如果没有，执行下一步。
* 通过分析器进行词法分析，提取 sql 语句的关键元素，比如提取上面这个语句是查询 select，提取需要查询的表名为 tb_student，需要查询所有的列，查询条件是这个表的 id='1'。然后判断这个 sql 语句是否有语法错误，比如关键词是否正确等等，如果检查没问题就执行下一步。
* 接下来就是优化器进行确定执行方案，上面的 sql 语句，可以有两种执行方案：
  
        a.先查询学生表中姓名为“张三”的学生，然后判断是否年龄是 18。
        b.先找出学生中年龄 18 岁的学生，然后再查询姓名为“张三”的学生。
    那么优化器根据自己的优化算法进行选择执行效率最好的一个方案（优化器认为，有时候不一定最好）。那么确认了执行计划后就准备开始执行了。

* 进行权限校验，如果没有权限就会返回错误信息，如果有权限就会调用数据库引擎接口，返回引擎的执行结果。

### 2.2 更新语句

以上就是一条查询 sql 的执行流程，那么接下来我们看看一条更新语句如何执行的呢？sql 语句如下：

```
update tb_student A set A.age='19' where A.name=' 张三 ';
```
我们来给张三修改下年龄，在实际数据库肯定不会设置年龄这个字段的，不然要被技术负责人打的。其实这条语句也基本上会沿着上一个查询的流程走，只不过执行更新的时候肯定要记录日志啦，这就会引入日志模块了，MySQL 自带的日志模块是 **binlog（归档日志）** ，所有的存储引擎都可以使用，我们常用的 InnoDB 引擎还自带了一个日志模块 **redo log（重做日志）**，我们就以 InnoDB 模式下来探讨这个语句的执行流程。流程如下：

* 先查询到张三这一条数据，如果有缓存，也是会用到缓存。
* 然后拿到查询的语句，把 age 改为 19，然后调用引擎 API 接口，写入这一行数据，InnoDB 引擎把数据保存在内存中，同时记录 redo log，此时 redo log 进入 prepare 状态，然后告诉执行器，执行完成了，随时可以提交。
* 执行器收到通知后记录 binlog，然后调用引擎接口，提交 redo log 为提交状态。
* 更新完成。

**这里肯定有同学会问，为什么要用两个日志模块，用一个日志模块不行吗?**

这是因为最开始 MySQL 并没有 InnoDB 引擎（InnoDB 引擎是其他公司以插件形式插入 MySQL 的），MySQL 自带的引擎是 MyISAM，但是我们知道 redo log 是 InnoDB 引擎特有的，其他存储引擎都没有，这就导致会没有 crash-safe 的能力(crash-safe 的能力即使数据库发生异常重启，之前提交的记录都不会丢失)，binlog 日志只能用来归档。

并不是说只用一个日志模块不可以，只是 InnoDB 引擎就是通过 redo log 来支持事务的。那么，又会有同学问，我用两个日志模块，但是不要这么复杂行不行，为什么 redo log 要引入 prepare 预提交状态？这里我们用反证法来说明下为什么要这么做？

* **先写 redo log 直接提交，然后写 binlog**，假设写完 redo log 后，机器挂了，binlog 日志没有被写入，那么机器重启后，这台机器会通过 redo log 恢复数据，但是这个时候 bingog 并没有记录该数据，后续进行机器备份的时候，就会丢失这一条数据，同时主从同步也会丢失这一条数据。
* **先写 binlog，然后写 redo log**，假设写完了 binlog，机器异常重启了，由于没有 redo log，本机是无法恢复这一条记录的，但是 binlog 又有记录，那么和上面同样的道理，就会产生数据不一致的情况。

如果采用 redo log 两阶段提交的方式就不一样了，写完 binglog 后，然后再提交 redo log 就会防止出现上述的问题，从而保证了数据的一致性。那么问题来了，有没有一个极端的情况呢？假设 redo log 处于预提交状态，binglog 也已经写完了，这个时候发生了异常重启会怎么样呢？
这个就要依赖于 MySQL 的处理机制了，MySQL 的处理过程如下：

* 判断 redo log 是否完整，如果判断是完整的，就立即提交。
* 如果 redo log 只是预提交但不是 commit 状态，这个时候就会去判断 binlog 是否完整，如果完整就提交 redo log, 不完整就回滚事务。

这样就解决了数据一致性的问题。

## 三 总结

* MySQL 主要分为 Server 层和引擎层，Server 层主要包括连接器、查询缓存、分析器、优化器、执行器，同时还有一个日志模块（binlog），这个日志模块所有执行引擎都可以共用，redolog 只有 InnoDB 有。
* 引擎层是插件式的，目前主要包括，MyISAM,InnoDB,Memory 等。
* 查询语句的执行流程如下：权限校验（如果命中缓存）--->查询缓存--->分析器--->优化器--->权限校验--->执行器--->引擎
* 更新语句执行流程如下：分析器---->权限校验---->执行器--->引擎---redo log(prepare 状态)--->binlog--->redo log(commit状态)

---

# 断电恢复

https://zhuanlan.zhihu.com/p/100009790



---

# 三大日志(redo log, undo log, binlog)

redo log, undo log, binlog





# 性能优化







---

# 一条SQL语句执行得很慢的原因有哪些？

https://mp.weixin.qq.com/s/YKmFEtHcZPBn1S9so0kxYw







---

# Ref

* 执行SQL语句: https://snailclimb.gitee.io/javaguide/#/docs/database/mysql/how-sql-executed-in-mysql
* 回表查询:
  版权声明：本文为CSDN博主「小小少年_」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
  原文链接：https://blog.csdn.net/CPLASF_/article/details/108799381
* [MySQL 三万字精华总结 + 面试100 问](https://mp.weixin.qq.com/s/MCFHNOQnTtJ6MGVjM3DP4A)
* [javaguide](https://javaguide.cn/database/mysql/transaction-isolation-level/#)
* [cyc2018](http://www.cyc2018.xyz/%E6%95%B0%E6%8D%AE%E5%BA%93/%E6%95%B0%E6%8D%AE%E5%BA%93%E7%B3%BB%E7%BB%9F%E5%8E%9F%E7%90%86.html)
* [[Innodb中的事务隔离级别和锁的关系](https://tech.meituan.com/2014/08/20/innodb-lock.html)





https://mp.weixin.qq.com/s/MCFHNOQnTtJ6MGVjM3DP4A

http://www.cyc2018.xyz/%E6%95%B0%E6%8D%AE%E5%BA%93/%E6%95%B0%E6%8D%AE%E5%BA%93%E7%B3%BB%E7%BB%9F%E5%8E%9F%E7%90%86.html#%E5%85%AD%E3%80%81next-key-locks

https://javaguide.cn/database/mysql/transaction-isolation-level/#%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB%E7%BA%A7%E5%88%AB

https://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247497197&idx=1&sn=9f82f73d876636944fb75348ef568c01&scene=21#wechat_redirect

https://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247496769&idx=1&sn=30990d141185303fd0c7ecf63c125b30&scene=21#wechat_redirect



https://mp.weixin.qq.com/s/4HWMXZAj1aZDHr5A301d8w

https://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247498122&idx=2&sn=f2062523af7e977a6deebe4d8be4501a&scene=21#wechat_redirect

https://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247496932&idx=1&sn=5bd840a32040998aa60c6317ccad71ac&scene=21#wechat_redirect

https://javaguide.cn/database/mysql/innodb-implementation-of-mvcc/#





https://www.cnblogs.com/zhoujinyi/p/3435982.html

https://segmentfault.com/a/1190000016566788

https://www.cnblogs.com/jian0110/p/15080603.html



https://blog.csdn.net/C_J33/article/details/79487941

https://www.jianshu.com/p/4f2311f38040

https://cloud.tencent.com/developer/article/1546932

https://tech.meituan.com/2014/08/20/innodb-lock.html

https://zhuanlan.zhihu.com/p/52678870