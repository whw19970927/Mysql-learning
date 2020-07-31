## 锁的分类

![image](https://github.com/whw19970927/Mysql-learning/blob/master/images/image-20200719142651715.png)



### 全局锁

![image](https://github.com/whw19970927/Mysql-learning/blob/master/images/image-20200719143553570.png)


#### 概念

上面也写到了，主要是做全库逻辑备份**mysqldump**。也就是把整库每个表都select出来成文本。

SQL 提供了2种加全局读锁的方法，
方法一： Flush tables with read lock (FTWRL)。
方法二：set global readonly=true

当你需要让整个库处于只读状态的时候，可以使用这个命令，之后其他线程的以下语句会被阻塞：数据更新语句（数据的增删改）、数据定义语句（包括建表、修改表结构等）和更新类事务的提交语句。

但是这样会导致备份过程中整个库完全处于只读的状态。

数据库只读的危险性：

* 如果在主库上做备份，那么在备份期间都不能执行更新，业务也就停止

* 如果在从库上做备份，那么备份期间从库不能执行主库同步过来的binlog，会导致主从延迟

  注：上面逻辑备份，是不加`--single-transaction`参数

这里就要介绍一下--single-transaction，在mysqldump中指定single-transaction时，会使用可重复读(REPEATABLE READ)事务隔离级别来保证整个dump过程中数据一致性，数据之前就会启动一个事务，来确保拿到一致性快照视图。该选项仅对InnoDB表有用，而myisam引擎无法保证，必须加上`--lock-all-tables`。在这期间，如果其他innodb引擎的线程修改了表的数据并提交，对该dump线程的数据并无影响，在这期间不会锁表。



#### 为什么需要全局读锁？

在支持事务的数据库引擎中，例如innodb，当 mysqldump 使用参数--single-transaction的时候，导数据之前就会启动一个事务，来确保拿到一致性快照视图。而由于 MVCC 的支持，这个过程中数据是可以正常更新的。

但没有事务特性的存储引擎，例如myisam，就只能通过FTWRL方法，**否则的话，备份系统备份的得到的库不是一个逻辑时间点，这个数据是逻辑不一致的。**

这也是innodb取代了myisam的一个原因吧。



#### 为什么不推荐使用set global readonly=true?

上述提到了两种加全局锁的方式，既然要全库只读，**为什么不使用 set global readonly=true 的方式呢**？

* 一是，在有些系统中，readonly 的值会被用来做其他逻辑，比如用来判断一个库是主库还是备库。因此，修改 global 变量的方式影响面更大。

  具体可以参见：https://blog.csdn.net/u013636377/article/details/50905768

* 二是，在异常处理机制上有差异。如果执行`FTWRL 命令之后由于客户端发生异常断开，那么 MySQL 会自动释放这个全局锁`，整个库回到可以正常更新的状态。而将整个库设置为 readonly 之后，如果客户端发生异常，则数据库就会一直保持 readonly 状态，这样会导致整个库长时间处于不可写状态，风险较高

* 三是，readonly 对super用户权限无效

  *注 ：业务的更新不只是增删改数据（DML)，还有可能是加字段等修改表结构的操作（DDL）。不论是哪种方法，一个库被全局锁上以后，你要对里面任何一个表做加字段操作，都是会被锁住的。*

  

### 表锁



![image](https://github.com/whw19970927/Mysql-learning/blob/master/images/image-20200719144035351.png)

其实表锁可以分为：

* 表锁
* 元数据锁（meta data lock，MDL）

表锁的语句是

`lock tables 表名 read`

与 FTWRL 类似，可以用 unlock tables 主动释放锁，也可以在客户端断开的时候自动释放。需要注意，lock tables 语法除了会限制别的线程的读写外，也限定了本线程接下来的操作对象。

举个例子, 如果在某个线程 A 中执行 lock tables t1 read, t2 write; 这个语句，则其他线程写 t1、读写 t2 的语句都会被阻塞。同时，线程 A 在执行 unlock tables 之前，也只能执行读 t1、读写 t2 的操作。连写 t1 都不允许，自然也不能访问其他表。

如图中所说在还没有出现更细粒度的锁的时候，表锁是最常用的处理并发的方式。而对于 InnoDB 这种支持行锁的引擎，一般不使用 lock tables 命令来控制并发，毕竟锁住整个表的影响面还是太大.



#### 元数据锁



**MDL作用是防止DDL和DML并发的冲突** ，**保证读写的正确性**

**MDL**不需要显式使用`，在访问一个表的时候会被自动加上。`MDL 的作用是，保证读写的正确性。你可以想象一下，如果一个查询正在遍历一个表中的数据，而执行期间另一个线程对这个表结构做变更，删了一列，那么查询线程拿到的结果跟表结构对不上，肯定是不行的。

- 读锁之间不互斥，因此你可以有多个线程同时对一张表增删改查。
- 读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。因此，如果有两个线程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行。



![image](https://github.com/whw19970927/Mysql-learning/blob/master/images/image-20200719154521105.png)

举个例子：

事务A开启，对表加了一个读锁，然后事务B需要的也是MDL读锁，读锁之间不互斥，所以可以正常运行。但C会被block，因为事务A的锁还没有被释放，而事务C也是需要锁的，所以会被阻塞住，不仅如此，后续的事务因为C的阻塞想获取MDL读锁都将失败，都会被阻塞住，所以所有对表的增删改查操作都需要先申请MDL 读锁，而这时读锁没有释放，对表alter ，产生了mdl写锁，把表t锁住了，这时候就对表t完全不可读写了。

**处理方法**

想解决MDL锁阻塞的情况，就要提交或者回滚导致阻塞的事务。可以通过information_schema.innodb_trx查询事务的执行时间。

例如:

`mysql> ``select` `* ``from` `information_schema.innodb_trx ``where` `TIME_TO_SEC(timediff(now(),trx_started))>60\G;`

语义是查看时间超过60s的事务

随后调用`show processlist`

![image](https://github.com/whw19970927/Mysql-learning/blob/master/images/image-20200719161144158.png)

查看此时的HOST，如果是localhost，只需要进入commit或者rollback即可，如果不是的话，就要考量是否可以kill掉。



### 共享锁和排它锁

![image](https://github.com/whw19970927/Mysql-learning/blob/master/images/image-20200719161308241.png)

- `LOCK TABLES t READ`：`InnoDB`存储引擎会对表`t`加表级别的`S锁`。

- `LOCK TABLES t WRITE`：`InnoDB`存储引擎会对表`t`加表级别的`X锁`。

注意：select语句默认不加锁，如果需要可以加上for update

-------------------------------------------------------------------------------------------------------------------------------------------------------

求证：

![image](https://github.com/whw19970927/Mysql-learning/blob/master/images/image-20200719172602023.png)



![image](https://github.com/whw19970927/Mysql-learning/blob/master/images/image-20200719172526959.png)


重开一个事务，发现更新成功

![image](https://github.com/whw19970927/Mysql-learning/blob/master/images/image-20200719172221701.png)

但是发现第一个事务中的数据并没有发生更改，仍然是张飞

![image](https://github.com/whw19970927/Mysql-learning/blob/master/images/image-20200719172639904.png)

从而说明第一个窗口的事务里的select并没有锁，但是具有隔离性并没有读取到窗口2里面更新后的数据。

这次我们加上for update（**在这里我踩了个小坑，一定要两个事务都提交后，才能执行update操作，否则会出现加锁超时的情况**）

![image](https://github.com/whw19970927/Mysql-learning/blob/master/images/image-20200719174443879.png)

而此时事务B再更改的时候

![image](https://github.com/whw19970927/Mysql-learning/blob/master/images/image-20200719174821950.png)

会发现被阻塞住了。

-------------------------------------------------------------------------------------------------------------------------------------------------------

在InnoDB中，既支持表锁，也支持行锁。表锁实现简单，占用资源较少，不过粒度很粗，有时候你仅仅需要锁住几条记录，但使用表锁的话相当于为表中的所有记录都加锁，所以性能比较差。行锁粒度更细，可以实现更精准的并发控制。

### 行锁

InnoDB支持，上面也说到了行锁和表锁的区别，由于Myisam不支持行锁，所以innodb在并发这方面的优势非常明显。

行锁的劣势：开销大；加锁慢；会出现死锁
行锁的优势：锁的粒度小，发生锁冲突的概率低；处理并发的能力强

#### record lock

![image](https://github.com/whw19970927/Mysql-learning/blob/master/images/image-20200719171243971.png)

这里我的个人理解是record lock锁住的是索引记录，而不是数据行



#### Gap Lock

![image](https://github.com/whw19970927/Mysql-learning/blob/master/images/image-20200719181217933.png)

之前提到过，mysql可以在rr的隔离级别避免幻读的问题，就是通过gap lock来完成的，他的作用也仅仅是用来避免幻读。

例如图中的例子：

在id6和10之间有了一个gap lock，也就意味着在事务提交之前，6和10之间都是不可以插入数据的。



####  Next-key Lock

Gap Lock + Record Lock = Next-Key Lock

![image](https://github.com/whw19970927/Mysql-learning/blob/master/images/image-20200719181715528.png)

它既能保护该条记录，又能阻止别的事务将新记录插入被保护记录前边的`间隙`。



#### 插入意向锁

我们说一个事务在插入一条记录时需要判断一下插入位置是不是被别的事务加了所谓的`gap锁`（`next-key锁`也包含`gap锁`，后边就不强调了），如果有的话，插入操作需要等待，直到拥有`gap锁`的那个事务提交。但是设计`InnoDB`的大叔规定事务在等待的时候也需要在内存中生成一个`锁结构`，表明有事务想在某个`间隙`中插入新记录，但是现在在等待。设计`InnoDB`的大叔就把这种类型的锁命名为插入意向锁。（**插入意向锁不等于意向锁！**）

引用上面的例子

![image](https://github.com/whw19970927/Mysql-learning/blob/master/images/image-20200719182618364.png)

如果此时有事务A和事务B想在6 - 10之间插入记录，那他们的琐结构会是这样

![image](https://github.com/whw19970927/Mysql-learning/blob/master/images/image-20200719182744810.png)

当当前事务提交之后会释放掉间nextkey lock，这样t2的状态就会由is_waiting的true变成false，也就是可以获得值为10的插入意向锁，从而插入数据。



#### 意向锁

1、表明“某个事务正在某些行持有了锁、或该事务准备去持有锁”

2、意向锁的存在是为了协调行锁和表锁的关系，支持多粒度（表锁与行锁）的锁并存。

3、例子：事务A修改user表的记录r，会给记录r上一把行级的排他锁（X），同时会给user表上一把意向排他锁（IX），这时事务B要给user表上一个表级的排他锁就会被阻塞。意向锁通过这种方式实现了行锁和表锁共存且满足事务隔离性的要求。

![image](https://github.com/whw19970927/Mysql-learning/blob/master/images/image-20200719183026653.png)

也就是，意向锁的存在会使更高粒度的锁阻塞，直到意向锁释放。

