## 1.基础架构：一条SQL查询语句是如何执行的？

```SQL
mysql> select * from T where ID=10;
```

#### MySQL的基本架构示意图

1. Server层

   - 连接器：管理连接，权限认证

     - 负责跟客户端建立连接，获取权限、维持和管理连接。一般的连接命令：

       ```sql
       mysql -h$ip -P$port -u$user -p
       ```

       使用show processlist;查看连接命令：

       ![20200620225713](F:\FileRecv\myDocs\1219784057\Image\SharePic\20200620225713.png)

       

   - 查询缓存：命中则直接返回结果（8版本已经删除）

     - 大多数情况下不建议使用查询缓存

       mysql拿到一个查询请求后，会先到查询缓存里面看看，是不是存在缓存否则执行查询语句。之前查询到的语句机器结果可能会以<K,V>对的方式被直接缓存在内存中。K是查询语句，V是查询结果。因为查询缓存的失效非常频繁，只要有对一个表的更新操作，这个表的所有查询缓存都会被清空。除非你使用一张静态表，且很长时间跟新一次。

       可以将参数query_cache_type设置为DEMAND，这样对于默认的SQL语句都不适用查询缓存。或则使用一下的查询命令禁用查询缓存：

       ```sql
       mysql> select SQL_CACHE * from T where ID=10;
       ```

   - 分析器：词法分析，语法分析

     - 分析器会做词法分析。mysql需要识别你的SQL语句里面的字符串分别代表什么。

       如果你的语句不对，就会收到"You have an error in your SQL syntax"的错误提醒。比如：

       ```sql
       mysql> elect * from user where id=1;
       ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'elect * from user where id=1' at line 1
       ```

   - 优化器：执行计划生成，索引选择

     - 优化器是在表里面由多个索引的时候，决定使用那个索引；或者在一个语句有关联查询的时候，决定各个表的连接顺序。

   - 执行器：操作引擎，返回结果

     - 先会判断用户是否有权限。

2. 存储引擎层

   负责数据的存储和提取。其架构模式是插件式的，支持InnoDB、MyISAM、Memory等多个存储引

Q1：

> 如果表T中没有字段k，而你执行以下这个语句时，会在那个阶段报错。
>
> ```sql
> select * from T where k=1;
> ```

A1：

> 分析层。

## 2.日志系统：一条SQL语句更新语句是如何执行的？

MySQL重要的两个日志模块：**redo log**（重做日志），**binlog**（归档日志）

#### redo log

如果每一次更新操作都要写进磁盘，然后磁盘也要找到对应的那条记录，然后再更新，整个过程的IO成本、查找成本都很高。所以mysql的设计者就用了类似酒店掌柜用粉板记账，之后再写入账本的思路来提升跟新效率。粉板和账本配合的整个过程，就是mysql里面经常说到的WAL（Write-Ahead Logging）技术，关键点就是先写日志，再写磁盘。

具体来说，当有一条记录需要更新的时候，InnoDB引擎就会先把记录写到redo log里面，并跟新内训，这个时候跟新就算完成了。同时InnoDB会在适当的时候，将这个操作记录更新到磁盘里面（往往是在空闲的时候）。

有了redo log，InnoDB就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这称为cache-safe。

#### binlog

Server层也有自己的日志，binlog。

这两种日志的不同：

	1. redo log属于InnoDB，binlog属于Server层实现的。
 	2. redo log是物理日志，记录的是“在某个数据也上做了什么修改”；binlog是逻辑日志，记录的是这个语句的原始逻辑。
 	3. redo log是循环写的，空间固定会用完；binlog是追加写的，不会覆盖以前的日志。

接下来，看一看下面这条更新语句是什么执行的内部流程：

```sql
mysql> update T set c=c+1 where ID=2;
```

	1. 执行器先找引擎ID=2这一行。ID是主键，引擎直接用树搜索找到这一行。如果这一行存在在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后再返回。
 	2. 执行器拿到行数据后，把这个值加1，得到新的行数据，再调用引擎接口写入这行新数据。
 	3. 引擎将这行新数据更新到内存中，同时将这个更新记录到redo log里面，此时redo log处于prepare状态。然后告知执行器执行完成了，随时可以提交事务。
 	4. 执行器生成这个操作的binlog，并把binlog写入磁盘。
 	5. 执行器调用引擎的提交事务接口，引擎把刚刚写入的redo log改成commit状态，更新完成。

这里使用了两阶段提交！可以是redo log和binlog这两个表示事务的提交状态，保持逻辑上的一致。

#### 小结

- redo log用于保证crash-safe的能力。**innodb_flush_log_at_trx_commit**这个参数设置为1的时候，表示每次事务的redo log都直接持久化到磁盘。这个参数建议设置为1，可以保证mysql异常重启后数据不丢失。
- **sync_binlog**参数建议设置成1，表示每次事务的binlog都持久化到磁盘。

Q2：

> 在什么场景下，一天一备会比一周一备更有优势？或者说，他影响了这个数据库系统的哪个指标？

A2：

> 最长恢复时间更短。RTO（恢复目标时间）

## 3.事务隔离：为什么你改了我还看不见？

#### 隔离性与隔离级别

事务的四大特性ACID：

- Atomicity：原子性
- Consistency：一致性
- Isolation：隔离性
- Durability：持久性

SQL标准的事务隔离级别：

- read uncommitted（读未提交）:一个事务还未提交，它做的变更就能被别的事务看到。
- read committed（读提交）:一个事务提交后，它做的变更才会被别的事务看到。
- repeatable read（可重复读）:一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的。同时未提交的变更对其它事务也是不可见的。
- serializable（串行化）:写会加写锁，读会加读锁。当出现锁冲突的时候，后访问到的事务必须等前一个事务执行完成，才能继续执行。

在实现上，数据库里面会创建一个视图，访问的时候以视图的逻辑结果为准。在“Repeatable Read”隔离级别下，这个视图是在事务启动的时候创建的，整个事务存在期间都用这个视图。

在“read committed”隔离级别下，这个视图是在每个SQL语句开始执行的时候创建的。

使用

```sql
mysql> show variables like 'transaciton_isolation'; #8#
或者
mysql> show variables like 'tx_isolation';

#结果
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| tx_isolation  | REPEATABLE-READ |
+---------------+-----------------+
1 row in set (0.00 sec)
```

查看隔离级别。

#### 事务隔离的实现

以“repeatable read”为例。在mysql中实际上每条记录在更新的时候都会同时记录一条回滚操作。记录上的最新值，通过回滚操作，都可以得到前一个状态的值。

假设一个值从1按顺序改成了2，3，4，在**undo log**（回滚日志）里面就会有类似下面的记录：

![20200621015057](F:\FileRecv\myDocs\1219784057\Image\SharePic\20200621015057.png)

当前值是4，但是查询这条记录的时候，不同时刻启动的事务会有不同的read-view。同一条记录在系统中可以存在不同的版本，就是数据库的MVCC（多版本并发控制）。对于read-view A，要想得到1，就必须将当前值依次执行图中所有的回滚操作得到。

**undo log**什么时候删除？系统会判断，当没有事务再需要用到这些回滚日志的时候，回滚日志会删除。就是当系统中没有比回滚日志更早的read-view的时候。

基于上面的说明，应该避免使用长事务。长事务意味着会存在很老的事务视图。由于这些事务随时可能访问数据库里面的任何数据，所以这个事务提交之前，数据库里面它可能会用到的回滚记录都必须保留，这回导致大量占用存储空间。并且长事务还会占用锁资源。

#### 事务的启动方式

MySQL的事务启动方式有以下几种：

1. 显示启动事务语句，begin和start transaction。提交语句commit，回滚语句rollback；
2. set autocommit=0这个命令会将这个线程的自动提交关掉。需要主动commit或rollback。

建议使用set autocommit=1，通过显示的语句来启动事务。

Q3：

> 如何避免长事务。

A3：

> 

## 4.深入浅出索引

索引的出现其实是为了提高数据的查询效率，就像书的目录一样。

索引常见的模型：

- 哈希表：是一种以key-value存储数据的结构。将值放入数组中，用一个hash函数吧key换算成一个确定的位置，然后把value放入数组的这个位置。如果多个key出现在同一个位置，则会拉出一个链表。哈希表适用于只有等值查询的场景。
- 有序数组：有序数组在等值查询和范围查询中的性能都非常优秀。但是在更新数据的时候，成本会很高，在插入数据的时候需要挪到后面所有的记录。**所以只适用于静态存储引擎。**
- 搜索树：用于N叉树在读写上的性能优点，以及适配磁盘的访问模式，被广泛的运用在数据库引擎中。

数据库底层的核心就是基于这些数据模型。

#### InnoDB的索引模型

在InnoDB中，表是根据主键顺序以索引的形式存放的，这种存储方式的表称为索引组织表。InnoDB使用了B+树模型。

每一个索引都是在InnoDB里面一颗B+树。

假设，创建一个主键为ID的表，表中有字段k，并且在k上有索引：

```sql
mysql> create table T(
    -> id int primary key,
    -> k int not null,
    -> name varchar(16),
    -> index (k))engine=InnoDB;
```

插入（ID，k）值为（100，1），（200，2），（300，3），（500，5），（600，6），两棵树的实例图如下：

![20200621024339](F:\FileRecv\myDocs\1219784057\Image\SharePic\20200621024339.png)

根据叶子节点的内容，索引类型分为主键索引和非主键索引。

- 主键索引的叶子节点存的是整行数据，在InnoDB里，主键索引也被称为聚簇索引（clustered index）；
- 非主键索引的叶子节点内容是主键的值，在InnoDB里，非主键索引又被称为二级索引（secondary index）；

基于主键索引和普通索引的查询有什么区别呢？

- 如果语句是**select * from T where ID=500**，即主键查询方式，则只需要搜索ID这棵B+树
- 如果语句是**select * from T where k=5**，即普通索引查询方式，则需要先搜索k索引树，得到ID的值为500，再到ID索引树搜索一次。这个过程称为回表。

所以应尽量使用主键查询。

#### 索引维护

B+树为了维护索引的有序性，在插入新值的时候会做必要的维护。如页分裂，页合并。

案例：哪些场景适合使用自增主键。

自增主键的定义：**NOT NULL PRIMARY KEY AUTO_INCREMENT**

自增主键的插入数据模式，正符合了我们前面提到的递增插入的场景。每次插入一条新记录，都是追加操作，都不会挪动其他记录，也不会触发叶子节点的页分裂。

同时用整型作为自增主键，只要4个字节，用长整型（bigint）则是8个字节。主键长度越小，普通索引的叶子节点就越小，普通索引占用的空间也就越小。

所以自增主键从性能和存储空间方面🤔，都是一个合理的选择。

Q4：

> 重建索引k：
>
> ```sql
> alter table T drop index k;
> alter table T add index(k);
> ```
>
> 重建主键索引：
>
> ```sql
> alter table T drop primary key;
> alter table T add primary key(id);
> ```
>
> 哪些更合适，为什么？

A4：

> 重建索引k是合理的。但是重建主键索引过程不合理。不论是删除主键还是创建主键，都会将整个表重建。可以使用alter table T engine=InnoDB。

---

在T表中，如果执行

```sql
select * from T where k between 3 and 5;
```

共查询k索引树3条记录，回表了两次。

### **那么，如何通过索引优化，避免回表呢？**

#### 覆盖索引

如果执行的语句是：

```sql
select ID from T where k between 3 and 5;
```

这时只需要查ID的值，而ID的值已经在k索引树上了，索引可以直接提供查询结果，不需要回表。在这个查询中，索引k已经覆盖了我们的查询请求，我们称为覆盖索引。

**由于索引覆盖可以减少树的搜索次数，显著提升查询性能，所以使用覆盖索引是一个常用的性能优化手段。**

Example：

> 在一个市民信息表上，是否有必要将身份证号和名字建立联合索引？
>
> 如果现在有一个高频请求，要根据身份证号查询姓名，这个联合索引就变得非常有必要。它可以在这个高频请求上用到覆盖索引，不在需要回表查整行记录，减少语句的执行时间。当然，索引字段的维护也是有代价的。

#### 最左前缀原则

在建立联合索引的时候，如何安排索引内的字段顺序？

第一原则是，如果通过调整顺序，可以少维护一个索引，那么这个顺序往往就是需要优先考虑采用的。接下来才考虑空间。

#### 索引下推

在MySQL5.6之后引入了索引下推优化（index condition pushdown），可以减少索引遍历过程中，对索引中包括的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数。

Example：

> 以市民表的联合索引（name，age）为例。执行以下语句：
>
> ```sql
> mysql> select * from tuser where name like '张%' and age=10 and ismale=1;
> ```
>
> 在MySQL5.6之前，查到姓名前缀为'张'的记录后，只能一个一个的回表，在对比字段值。
>
> 而存在索引下推优化后，InnoDB在（name，age）索引内部就判断了age是否等于10，对于不等于10的记录直接跳过，这样明显可以减少回表。

Q5:

>现在有一个表，表结构定义如下：
>
>```sql
>create table `geek`(
>	`a` int (11) not null,
>    `b` int (11) not null,
>    `c` int (11) not null,
>    `d` int (11) not null,
>    primary key(`a`,`b`),
>    key `c` (`c`) ,
>    key `ca` (`c`,`a`),
>    key `cb` (`c`,`b`)
>)engine=InnoDB;
>```
>
>由于历史原因，这个表需要a,b做联合主键。
>
>且业务中有以下查询语句：
>
>```sql
>select * from geek where c=N order by a limit 1;
>select * from geek where c=N order by b limit 1;
>```
>
>请问为了这两个查询语句，“ca”，“cb”这两个索引是否都必须存在？

A5:

>主键a,b的聚簇索引组织顺序相当于order by a,b，也就是先按a排序，再按b排序，c无序。
>
>```sql
>(a,b)
>-a--|-b--|-c--|-d--
>```
>
>索引c,a的组织是先按c排序，再按a排序，同时记录主键；这个跟索引c的数据是一模一样的。
>
>```sql
>(c,a)
>-c--|-a--|-b--
>
>(c)
>-c--|-a--|-b--
>```
>
>索引c,b的组织是先按c排序，再按b排序，同时记录主键
>
>```sql
>(c,b)
>-c--|-b--|-a--
>```
>
>结论是ca可以去掉，cb需要保留。

## 5.全局锁、表锁、行锁

### 全局锁

全局锁就是对整个数据库实例加锁。MySQL提供了一个加全局锁的方法，命令是**Flush tables with read lock**(FTWRL)。当你需要整个数据库处于**只读**状态的时候，可以使用这个命令，之后其他线程的以下命令都会被阻塞：数据库更新命令、数据库定义语句和更新事务的提交语句。

**全局锁的典型应用场景：做全库逻辑备份。**

同时官方也自带的逻辑备份工具是：**mysqldump**。当mysqldump使用参数single-transaction的时候，导数据之前就会启动一个事务，来确保拿到一致性视图。而由于MVCC的支持，这个过程是可以正常更新的。前提是引擎要支持这个隔离级别。

### 表级锁

MySQL里面的表级锁分为两种：一种是表锁，一种是元数据锁。

表锁的语法是：lock tables ... read/write.对于InnoDB这种支持行锁的引擎，不建议使用lock tables命令来控制并发。

另一种就是**MDL**（metadata lock）。MDL不需要显示使用，在访问一个表时会被自动加上。MDL的作用是，保证读写的正确性。

当对一个表做增删改查操作的时候，加MDL读锁；当要对表结构变更操作的时候，加MDL写锁。

读锁之间不互斥，读写锁、写锁之间互斥，用来保证变更表结构操作的安全性。

Example：

>假设现在给一个小表T加个字段（实验环境5.6）
>
>![20200621161659](F:\FileRecv\myDocs\1219784057\Image\SharePic\20200621161659.png)
>
>可以发现session A先启动，这时候会对表t加一个MDL读锁。由于session B需要的也是MDL读锁，因此可以正常执行。之后session C会被blocked，是因为session A的读锁还没有被释放，而session C需要写锁，因此只能被阻塞。之后所有在表t上新申请MDL读锁的请求也会被阻塞。等于这个表完全不可读了。
>
>如果某个表上的查询语句频繁，而且客户端有重试机制，这个库的线程很快就会爆满。
>
>那么如何安全的给小表加字段呢？
>
>首先可以先解决长事务，事务不提交，就会一直占着MDL锁。可以暂停长事务或者kill掉。
>
>如果是一个热点表，kill掉后，立马会有新请求来。比较理想的机制是，在alter table语句里面设定等待时间。
>
>```sql
>alter table tbl_name nowait add column ...
>alter table tbl_name wait N add column ...
>```

Q6:

> 备份一般会在备库上执行，你在用-single-transaction方法做逻辑备份的过程中，如果主库上的小表做了一个DDL，比如给一个表上加了一列。这时候，从备库上会看到什么现象？

A6：

> 当备库做逻辑备份时，从主库的binlog传来一个DDL语句。
>
> 假设这个DDL是针对表t1的，这里把备份记录过程中的几个关键语句列出：
>
> ```sql
> Q1：set session transaction isolation level repeatable read;
> Q2：start transaction with consistent snapshot;
> /*other tables*/
> Q3：savepoint sp;
> /*时刻1*/
> Q4：show create table `t1`;
> /*时刻2*/
> Q5：select * from `t1`;
> /*时刻3*/
> Q6：rollback to savepoint sp;
> /*时刻4*/
> /*other tables*/
> ```
>
> 在备份开始的时候，确保是RR的隔离级别。
>
> 启动事务，这里用**with consistent snapshot**确保这个语句执行完后就可以得到一个一致性视图（Q2）；
>
> 设置一个保存点，这个很重要（Q3）；
>
> show create 是为了拿到表结构（Q4），然后正式导数据（Q5），回滚到SAVEPOINT sp，在这里的作用是释放t1的MDL锁（Q6）。
>
> 情况如下：
>
> 1. 如果在Q4之前到达，备份拿到的是DDL后的表结构。
> 2. 如果在时刻2到达，则表结构被改过，Q5执行的时候，报Table definition has changed,please retry transaction,现象：mysqldump终止。
> 3. 如果在时刻2和时刻3之间到达，mysqldump占着t1的MDL读锁，binlog被阻塞，现象：主从延迟，直到Q6执行完成。
> 4. 从时刻4开始，mysqldump释放了MDL的读锁，现象：没有影响，备份拿到的是DDL前的表结构。

### 行锁

行锁就是针对数据表中行记录的锁

#### 两阶段锁

Example：

> 在下面的操作序列中，事务B的update语句执行时回事什么现象呢？（字段id是表t的主键）
>
> ![20200621181520](F:\FileRecv\myDocs\1219784057\Image\SharePic\20200621181520.png)
>
> 实际上事务B的update会被阻塞，直到事务A执行commit之后，事务B才能继续执行。

在InnoDB事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是等到事务结束才释放。这个就是两阶段协议。

**如果你的事务中需要锁多个行，要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放。**

#### 死锁和死锁检测

当并发系统中出现不同线程循环资源依赖，涉及的线程都在等待别的线程释放资源时，就会导致这几个线程都进入无限等待的状态，称为死锁。

当出现死锁后，有两种策略：

- 直接进入等待，直到超时。这个超时时间可以通过参数**innodb_lock_wait_timeout**来设置。
- 发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行。将参数**innodb_deadlock_detect**设置为**on**，表示开启这个逻辑。

正常情况下，使用第二种策略，即：主动检测死锁，而且**innodb_deadlock_detect**的默认值本身就是**on**。

Example：

> 如果所有的事务都要更新同一行的场景呢？
>
> 每个新来的被堵住的线程，都要判断会不会由于自己的加入导致了死锁，这是一个时间复杂度为O(n)的操作。假设有1000个并发线程要同时更新同一行，那么死锁的检测操作就是100万量级的。虽然最终检测的结果是没有死锁，但这期间要消耗大量的CPU资源。因此，你可能会看到CPU资源利用率很高，但是每秒却执行不了几个事务。
>
> 那么怎么解决这种**热点行更新**导致的性能问题呢？
>
> - 临时关闭死锁检测
>
> - 控制并发度
>
>   底层源码修改：对于相同行的更新，在进入引擎前排队。
>
>   或者从业务角度减少每一行的并发度。

Q7：

> 如果你要删除一个表中的前10000行数据，有以下三张方法可以做到：
>
> 第一种：直接执行 delete from T limit 10000;
>
> 第二种：在一个连接中循环执行20次delete from T limit 500;
>
> 第三种：在20个连接中同时执行 delete from T limit 500.
>
> 你会选择哪一种？

A7：

> 第二种。
>
> 第一种单句语句占用时间长，锁的时间也比较长；而且大事务还会导致主从延迟。
>
> 第三种会人为造成锁冲突。

## 6.事务到底是隔离还是不隔离的？

Example：

> 下面是一个只有两行的表的初始化语句：
>
> ```sql
> mysql> create table `t`(
> 	`id` int(11) not null,
>     `k` int(11) default null,
>     primary key (`id`)
> )engine=InnoDB;
> insert into t(id,k) values(1,1),(2,2);
> ```
>
> ![图1](F:\FileRecv\myDocs\1219784057\Image\SharePic\20200621203655.png)
>
> 注意事务的启动时机。begin/start transaction 命令并不是一个事务的起点，在执行到它们之后的第一个操作InnoDB表的语句，事务才真正启动。如果你想要马上启动一个事务，可以使用start transaction with consistent snapshot这个命令。
>
> > 第一种启动方式，一致性视图是在第一个快照度语句时创建的。
> >
> > 第二种启动方式一致性视图是在执行start transaction with consistent snapshot时创建的。
>
> 默认使用autocommit=1
>
> 在MySQL中，有两个视图的概念：
>
> - 一个是view。它是一个用查询语句定义的虚拟表，在调用的时候执行查询语句并生成结果。创建视图的语法是create view ...，而它的查询方法与表一样。
>
> - 另一个是InnoDB在实现MVCC时用到的一致性视图，即consistent read view，用于支持RC（Read Committed）和RR（Repeatable Read）隔离级别的实现。

####  "快照"在MVCC里是怎么工作的？

在RR级别下，事务在启动的时候就“拍了个快照”。这个快照是基于整库的❓

实际上InnoDB里面每个事务有一个唯一的事务ID，叫作**transaction id**。它是在事务开始的时候向InnoDB的事务系统申请的，是按照顺序严格递增的。

而每行数据也都是有多个版本的。每次事务更新的时候，都会生成一个新的数据版本，并且把transaction id赋值给这个数据版本的事务ID，记为**row trx_id**。同时，旧的数据，版本要保留，并且在新的数据版本中，能够有信息可以直接拿到它。

也就是说，数据表的一行记录，其实可能有多个版本（row），每个版本有自己的row trx_id。

如图所示，就是一个记录被多个事务连续更新后的状态。

![20200621215902](F:\FileRecv\myDocs\1219784057\Image\SharePic\20200621215902.png)

图中框内是同一行数的四个版本，当前最新版本是V4，k的值是22，它是被transaction id为25的事务更新的，因此它的row trx_id也是25。

图中的U1，U2，U3就是**undo log**（回滚日志）；而V1，V2，V3并不是物理上真实存在的，而是每次通过当前版本的值和**undo log**计算出来的。比如V2的值就是通过V4依次执行U3，U2算出来的。

根据RR的定义，一个事务启动的时候，能够看到所有提交的事务结果。但是之后，这个事务的执行期间，其它事务的更新对它不可见。

在实现上，InnoDB为每个事务构造了一个数组，用来保存这个事务启动瞬间，当前正在活跃的所有的事务ID。“活跃”指的是启动了还未提交。

数组里面事务ID的最小值即为低水位，当前系统里面创建过的事务ID的最大值加1即为高水位。

这个数组和高水位，就组成了当前事务的一致性视图（**read-view**）。

而数据版本的可见性规则，就是基于数据的row trx_id和这个一致性视图的对比结果得到的：

![20200621222416](F:\FileRecv\myDocs\1219784057\Image\SharePic\20200621222416.png)

这样，对于当前事务的启动瞬间来说，一个数据版本的row trx_id,有以下几种可能：

1. 如果落在绿色部分，表示这个版本是已提交的事务或者当前事务自己生成的，这个数据是可见的。
2. 如果落在红色部分，表示这个版本是由将来启动的事务提交的，是肯定不可见的。
3. 如果落在黄色部分，有以下两种情况：
   - 若row trx_ix在数组中，表示这个版本是由还没提交的事务生成的，是不可见的。
   - 若row trx_ix不在数组中，表示这个版本是已经提交的事务生成的，是可见的。

NOW！通过以上分析，可以得出，**InnoDB利用了“所有数据都有多个版本”的这个特性，实现了“秒级创建快照”的能力。**

Example：

> 分析之前的三个事务：
>
> 假设：
>
> 1. 事务A开始前，系统里面只有一个活跃ID是99；
> 2. 事务A、B、C的版本号分别是100、101、102，且当前系统里只有四个事务；
> 3. 三个事务开始前，（1，1）这一行数据的row trx_id是90。
>
> 这样，事务A的视图数组为【99，100】，事务B的视图数组为【99，100，101】，事务C的视图数组为【99，100，101，102】。
>
> 事务A查询数据逻辑图：
>
> ![20200621232726](F:\FileRecv\myDocs\1219784057\Image\SharePic\20200621232726.png)
>
> 对于事务A来说，只有历史版本2的row trx_id=90，比低水位小，处于绿色区域，可见。
>
> 这样执行下来，虽然期间这一行数据被修改过，但是事务A不论在什么时候查询，看到的这行数据结果都是一致的，所以称之为一致性读。
>
> 一个数据版本，对于一个事务视图来说，除了自己的更新总是可见以外，有三种情况：
>
> 1. 版本未提交，不可见；
> 2. 版本已提交，但是在视图创建后提交，不可见；
> 3. 版本已提交，而且是在视图创建前提交，可见。

#### 更新逻辑

那么事务B的update语句，如果按照一致性读，结果会是如何呢？

![20200621235512](F:\FileRecv\myDocs\1219784057\Image\SharePic\20200621235512.png)

当事务B要去更新数据的时候，就不能再在历史版本上更新了，否则事务C的更新就丢失了。因此事务B此时的set k=k+1是在（1，2）的基础上进行操作的。

**更新数据都是先读后写，而这个读，只能读当前的值，称为当前读（current read）**

除了update语句，select语句如果加锁，也可以成为当前读：

```sql
mysql> select k from t where id=1 lock in share mode; /*读锁，S锁，共享锁*/
mysql> select k from t where id=1 for update;/*写锁，X锁，排他锁*/
```

如果事务C更新完后没有马上提交，而是变成了下面的事务C'：

![20200622000134](F:\FileRecv\myDocs\1219784057\Image\SharePic\20200622000134.png)

这时候就要用到**两阶段锁协议**，事务C'没提交，也就是说（1，2）这个版本上的写锁还没有释放。而B是当前读，必须要读到最新版本，而且必须加锁，因此就被锁住了，必须等到事务C'释放这个锁，才能继续它的当前读。

所以事务的可重复读的能力是怎么实现的？

- RR的核心就是一致性读（consistent read）；而事务更新数据的时候，只能用当前读。如果当前纪录的行锁被其他事务占用的话，就需要进入锁等待。
- 而读提交的逻辑和RR的逻辑类似，主要区别为：
  - 在RR隔离级别下，只需要在事务开始的时候创建一张视图，之后事务里的其他查询都共用这个一致性视图；
  - 在读提交隔离级别下，每一句语句执行前都会重新算出一个新的视图。

在读隔离级别下，事务A、B查询到的k应该是多少？

“start transaction with consistent snapshot”的意思是从这个语句开始，创建一个持续整个事务的一致性快照。所以在读提交隔离级别下，这个用法就没有意义了，等效于普通的start transaction。

事务A的查询语句，可以获取的时序是（1，2），（1，3）；（1，3）还未提交，不可见，（1，2）提交，可见。查询得到事务A返回结果是k=2。事务B的结果是k=3。

Q8：

>事务隔离级别为RR，现在要把“所有字段c和id值相等的行”的c值清零，但是发现一个诡异的、改不掉的情况。
>
>```sql
>mysql> create table `t`(
>	`id` int(11) not null,
>	`c` int(11) default null,
>	primary key (`id`)
>)ENGINE=InnoDB;
>insert into t(id,c) values(1,1),(2,2),(3,3),(4,4)
>```
>
>执行以下语句：将“字段c和id值相等的行”的c值清零。
>
>```sql
>1. begin;
>2. select * from t;
>3. update t set c=0 where id =c;
>4. select * from t;
>```
>
>请构造这种数据无法修改的情况，并说明原理。

A8：

>可以在语句3之前，先对所有c值修改。比如在语句3之前，语句2之后执行：
>
>```sql
>update t set c=c+1;
>```
>
>这种场景就是所谓的“乐观锁”。我们平时会基于version字段对row进行cas式的更新，类似：
>
>```sql
>update ... set where id = xxx and version =xxx;
>```
>
>如果version被其他事务抢先更新，即自己的事务更新失败，trx_id没有变成自身事务的id，任然是事务之前的trx_id,这时候再去select，就会出现数据无法修改的情况。解决方法就是每次CAS更新不管成功还是失败，结束当前事务。如果失败则重新起一个事务进行查询更新。

## 7.普通索引和唯一索引，应该如何选择

