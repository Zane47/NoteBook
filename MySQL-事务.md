## 锁与事务思维导图

![](https://img-blog.csdnimg.cn/20210526175537918.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NqdzAwMDE=,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/b92fdf49d8784863b8de7c3d2ba6c017.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NyYXp5amFjazE=,size_16,color_FFFFFF,t_70#pic_center)

![](http://raw.githubusercontent.com/IMWYY/AboutMyself/master/picBed/Screenshot1520500121.png)



## 锁和事务的关系
1. 事务具有ACID（原子性、一致性、隔离性和持久性），锁是用于解决隔离性的一种机制。
2. 事务的隔离级别通过锁的机制来实现。另外锁有不同的粒度，同时事务也是有不同的隔离级别的(一般四种：读未提交Read uncommitted， 读已提交Read committed， 可重复读Repeatable read， 可串行化Serializable)。
3. 开启事务就自动加锁。

## 事务的基本知识

### 概念

事务是逻辑上的一组操作，要么都执行，要么都不执行。它满足 ACID 特性的一组操作，可以通过 Commit 提交一个事务，也可以使用 Rollback 进行回滚。

经典例子: 转账。A要给B转账1000元，这个转账会涉及到两个关键操作就是：将A的余额减少1000元，将B的余额增加1000元。万一在这两个操作之间突然出现错误, 比如银行系统崩溃，导致A余额减少而B的余额没有增加，这样就不对了。事务就是保证这两个关键操作要么都成功，要么都要失败。

Auto Commit：MySQL 默认采用自动提交模式。也就是说，如果不显式使用`START TRANSACTION`语句来开始一个事务，那么每个查询操作都会被当做一个事务并自动提交。

### 事务的特性ACID

#### 特性

1. 原子性（Atomicity), 事务被视为不可分割的最小单元，事务的所有操作要么全部提交成功，要么全部失败回滚. 回滚可以用回滚日志（Undo Log）来实现，回滚日志记录着事务所执行的修改操作，在回滚时反向执行这些修改操作即可。
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













## ref

----- 著作权归Guide哥所有。 链接: https://javaguide.cn/database/mysql/transaction-isolation-level/#







