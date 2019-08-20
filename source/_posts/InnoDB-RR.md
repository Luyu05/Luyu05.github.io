---
title: MySQL核心-事务、锁以及日志
date: 2019-02-14 13:48:16
tags: 数据库
categories: 数据库
---
MySQL核心-事务、锁以及日志
<!-- more -->
# 锁分类
粒度划分：表锁、行锁、范围锁（GAP）
锁的模式：共享、排他、意向

### 1.表/行（S锁、X锁）
S:读锁
X:写锁

### 2.意向锁（IS锁、IX锁）
意向锁是表（共享锁、排它锁）和行（共享锁、排它锁）之间的桥梁
表级别的锁
一个事务A给某张表加了一个意向锁IX，暗示接下来要对该表的一行数据加行X锁，这样事务B如果要给整张表加X锁就不用遍历每一行查看是否能加X锁了。反过来，一个事务如果要给某张表的某一行加S锁，那么需要先获得整张表的IS（X同理）
IS和表级别的S不互斥，IX和表级别的X互斥。

<img src='003.jpg'>

### 3.AI
自增锁->表级别
保证同一个事务连续的insert操作的自增列（一般id）连续

### 4.Next-Key锁(临键锁/下文有详细介绍)

---
# 事务分类

<img src='002.jpg'>
这里重点强调下更新丢失
### 第一类更新丢失
执行时间重合的两个事务，均更新了某条记录的field1，第一个事务更新成功并commit，第二个事务却更新失败并执行回滚这时候如果没有任何处理措施，会将第一个事务的更新覆盖。这种情况称为第一类更新丢失，目前这四个隔离级别都没有这个问题了，具体解决方式不清楚，后续查下资料。

### 第二类更新丢失
就是业务上常见的场景，同样执行时间重合的两个事务，均更新了某条记录的fiedl1（均是先读取再更新），事务1和事务2读取到了当前db中记录的值，接着事务1对它进行了+1并commit，而事务2进行了-1并commit，这时候因为事务2读取的是旧值而将事务1的更新覆盖了。

针对第二类更新丢失常见的解决方法如下

#### 方法1：单条语句的原子性
update T set account = account + 1 where userId = 1

Ps：update Ticket set count = count -  1 where ticketId = 1 and count > 0

#### 方法2：悲观锁
```sql
start transaction
int b = select account from T where userId = 1 for update
b = b + 1
update T set account = b where userId = 1
```

```sql
start transaction
int b = select account from T where userId = 1 for update
b = b - 1
update T set account = b where userId = 1
```

#### 方法3：乐观锁
一般和while循环一起使用
```sql
start transaction
int b,v1 = select account, version from T where userId = 1 
b = b + 1
update T set account = b, version = v1 + 1 where userId = 1 and version = v1;
```

```sql
start transaction
int b,v1 = select account, version from T where userId = 1 
b = b - 1
update T set account = b, version = v1 + 1 where userId = 1 and version = v1;
```
---

# 快照读
RR隔离级别，事务启动时会创建一个整个库的视图
Ps RC隔离级别，每条sql单独创建一个视图
- 辅助知识点
   db中每行数据都有多个版本，每个版本有自己的row trx_id（对其修改的事务id）

- 具体的实现方式
   1.事务启动时，会维持一个数组，用来保存启动瞬间，当前『活跃』的事务ID
   2.小于数组中最低ID的数据都是可见的；大于数组中最大ID的数据都是不可见的
   3.大于最低ID&小于最高ID的数据，如果位于数组中那么是不可见的；反之可见

MVCC基于快照读，保证读读不加锁，读写不加锁，极大提升了性能 类似JUC里的CopyOnWriteList

# 当前读
update/delete/select for update/select lock in share mode都属于当前读
前3个语句会对行加X锁，最后一个加S锁
Ps:如果存在unique index那么insert时候也会加锁

---
# 幻读
## 条件
- 插入
- 当前读

本质：无法对未来插入的符合加锁条件的行加锁，从而产生一些『不可控的状况』

## 举例
Ps：假设此时并未引入Next-Key Lock(其实就是RC隔离级别的做法)
table(id key, name) 

id|name
-|-
1|mike
5|luyu
10|jack

|session1|session2
-|-|-
T1|begin;<br>select * from table where id=5 for update;<br>update table set id=2 where id=5;|  
T2||insert into table values(5, john);<br>update table set name='mary' where name = 'john';
T3|commit;|

## 问题
- 语义上：session1的select语句表示要将id=5的记录都上锁，不准其他事务进行读写；但是session2中对id=5的记录进行了update操作
- 一致性：上述情况执行完毕binlog会如下记载

```sql
insert into table values(5, john);
update table set name='jack' where name = 'john';
update table set id=2 where id=5;
```
如果这个binlog拿去从库执行，就会出现主从数据的不一致

---
# 如何解决幻读
## Next-Key锁
归根结底，问题出现的原因就是，session2中insert语句的执行
此时需要引入next-key lock(RR默认)，对可能执行id=5插入的区间加锁，在上面例子中RR级别下id位于[1,10)的记录的insert操作都将会block

## 加锁举例
RR隔离级别
table(id key, name) 

id|name
-|-
1|mike
5|luyu
10|jack

select * from table where id = 5 for update;

表中横轴代表id是哪种类型的索引，纵轴代表sql语句执行的结果（范围锁一般只针对非唯一索引）
<img src='001.jpg'>

---
# 索引
## 数据结构的演进
### 从二叉树到B树
B树其实就是多叉树，当数据量较大时二叉树的深度过大，而索引可能是存储在磁盘上的，这可能会导致某个具体数据的定位过程耗时较长，为此引入了B树（多叉树）
### 从B树到B+树
相比B树，B+树只有叶子节点才存储具体的数据，这意味着非叶子节点可以存储更多的索引意味着树的高度会相应降低；而且因为所有的data都存储在叶子节点，range查询变得更为方便
## InnoDB VS MyISAM
Innodb与MyISAM的索引都采用B+树，二者的区别主要如下
- InnoDB的数据文件本身就是索引文件;MyISAM索引文件和数据文件是分离的，索引文件仅保存数据记录的地址
- InnoDB的辅助索引data域存储相应记录主键的值而不是地址

---
# MySQL log

## redo log
前滚日志、物理日志记录数据页的修改
事务1，执行了多个update操作，需要修改不同的磁盘Page，如果直接去修改磁盘那么性能简直爆炸。为此不直接写磁盘，而是在内存中进行事务提交，然后再通过后台线程异步地把内存中的数据写到磁盘中。
but，如果宕机，会出现数据丢失问题
为此，我们在提交到内存后-》会去写日志（Write-Ahead log）-》后台线程异步地把内存中的数据写到磁盘中
如果宕机的话可以通过日志降低数据丢失量

InnoDB引擎的Write-Ahead log就是Redo Log。而Redo log的刷盘也是异步的（Redo buffer 到 Redo log），可以配置三种刷盘策略
1）秒级别    /s
2）事务级别  /transaction 一般使用该策略 几乎不会丢失数据 但是性能较差
3）不刷盘（指定时间）

log刷盘一般是刷到同一块盘，性能比刷内存提交要好，这就是redo log存在的意义

## undo log
回滚日志、逻辑日志、记录行记录的变更
本质：『备份数据』，事务未提交之前的时间里的备份数据，提交事务后，如果没有其他事务引用历史版本，就可以删除了
如下图，当T100（事务100）完成时候就可以删除V95了。UNDO LOG是MVCC的基础，和JUC中的CopuOnWrite思想很像，实现了读读不加锁，读写不冲突。
RR隔离级别下，事务启动瞬时会去寻找小于当前事务id的最新的数据视图，当事务修改数据后会将新的数据放到链表尾部
<img src='004.jpg'>
显然，当比较耗时的事务执行过程中会使得很多数据的老版本无法删除（这条链可能会很长）

## bin  log
位于Server层，所有引擎通用
逻辑日志，常用于归档
bin log和redo log是二阶段提交

为啥需要二阶段提交？
1）假如（执行了一条update后）先写redo再写binlog，写binlog时MySQL crash了，进程恢复后可以根据redo log恢复上次的update操作

但是binlog中没有记录这次变更，如果拿这个binlog去备库执行，那么会丢失数据

2）先写bin log再写redo log，同样crash崩溃恢复后会丢失本次更新，但是bin log已经记录，再拿去备库执行，备库会比主库多一次数据变更

和redo log的异同？
1）redo时InnoDB引擎特有；binlog时MySQL的Server层实现的，所有引擎都可以适用
2）redo是物理日志，记录数据页变更；binlog是逻辑日志，记录的是语句的逻辑，比如更新id=2的记录

---
参考文章：
http://blog.jobbole.com/24006/
http://hedengcheng.com/?p=771#_%E7%BB%84%E5%90%88%E4%B8%80%EF%BC%9Aid%E4%B8%BB%E9%94%AE+RC