Mysql的一条查询语句是经过：

连接器 -> 分析器 ->优化器 -> 执行器

最后到达执行引擎


![image](https://github.com/whw19970927/Mysql-learning/blob/master/Image/image-20200714182138374.png)



连接器也就是要用于连接数据库，比如Java中的JDBC就是这个作用

查询缓存之前说道现在已经废用，因为在高频写的情况下，每次一个表上有更新的时候，查询缓存会失效，会讲整张表的缓存结果清空。

然后分析器会通过语法分析知道这是一条更新语句，优化器会选择一条优化的索引，然后由执行器来执行并更新。

而更新流程和查询语句的区别是，更新流程存在redo log和binlog。

## redo log

假设Mysql每次更新都要刷盘，那么磁盘io的成本会很高，之前包括像buffer pool和change buffer都是为了通过顺序读写的方式尽量减少随机读写磁盘io的消耗

在Mysql中，WAL应用非常广泛，也就是write ahead logging，先写日志再刷盘，可以理解为当一条记录需要更新的时候，InnoDB会先把记录写到redolog中，并更新内存，这个时候更新就完成了，InnoDB会适时的将操作记录更新到磁盘中

这里的“适当”在下图有体现：

![image](https://github.com/whw19970927/Mysql-learning/blob/master/Image/image-20200714184719923.png)



在产生写操作的时候，innodb都会产生redolog，redo log的大小是固定的，是一个跟踪写的操作，一边写一边后移，比如可以配置为一组 4 个文件，每个文件的大小是 1GB，那redo log总共就可以记录 4GB 的操作。写满则会落盘到表空间中。发生write时，会记录每个数据页的修改。

write_pos是当前记录的位置，check_point是擦除的位置，两者都是往后推移并且循环的，擦除前要把记录更新到文件，两者之间空着的部分就是用来记录新的操作的，如果前者追上后者，说明没有足够的空间进行操作了，肉色部分已经全部记录满了，此时需要停下来擦除一些记录，并把check point推进。

因此，InnoDB可以保证即使数据库发生异常重启，即如果Mysql 进程异常重启了，系统会自动去检查redo log，将未写入到Mysql的数据从redo log恢复到Mysql中去，也就是crash-safe，得意于WAL技术和redo log。



## crash safe

crash safe中有一个两阶段提交的概念。

MySQL是多存储引擎的，不管使用那种存储引擎，都会有binlog，而不一定有redo log，简单的说，binlog是MySQL Server层的，redo log是InnoDB层的。下图用于解释两阶段提交：

![image](https://github.com/whw19970927/Mysql-learning/blob/master/Image/image-20200714191058055.png)

1）执行器调用存储引擎接口，存储引擎将修改更新到内存中后，将修改操作写到redo log里面，此时redo log处于prepare状态；
2）存储引擎告知执行器执行完毕，执行器开始将操作写入到bin log中，写完后调用存储引擎的接口提交事务；
3）存储引擎将redo log的状态置为commit。

在开启binlog后，为了保证Binlog和redo日志的一致性，Mysql内部会自动将普通事务当做一个内部分布式事务来处理，由Binlog来通知InnoDB引擎来执行prepare，commit或者rollback步骤，再看一张图

![image](https://github.com/whw19970927/Mysql-learning/blob/master/Image/image-20200714193237556.png)

以上的图片中可以看到，事务的提交主要分为两个主要步骤：

* 准备阶段（Storage Engine（InnoDB） Transaction Prepare Phase）

此时SQL已经成功执行，并生成xid信息及redo和undo的内存日志。然后调用prepare方法完成第一阶段，papare方法实际上什么也没做，将事务状态设为TRX_PREPARED，并将redo log刷磁盘。

* 提交阶段(Storage Engine（InnoDB）Commit Phase)
  * 记录协调者日志，即Binlog日志。

如果事务涉及的所有存储引擎的prepare都执行成功，则调用TC_LOG_BINLOG::log_xid方法将SQL语句写到binlog（write()将binary log内存日志数据写入文件系统缓存，fsync()将binary log文件系统缓存日志数据永久写入磁盘）。此时，事务已经铁定要提交了。否则，调用ha_rollback_trans方法回滚事务，而SQL语句实际上也不会写到binlog。

* 告诉引擎做commit。

最后，调用引擎的commit完成事务的提交。会清除undo信息，刷redo日志，将事务设为TRX_NOT_STARTED状态。



再看一张图加深理解

有如下代码：

![image](https://github.com/whw19970927/Mysql-learning/blob/master/Image/image-20200715003014318.png)

![image](https://github.com/whw19970927/Mysql-learning/blob/master/Image/image-20200715003024422.png)

![image](https://github.com/whw19970927/Mysql-learning/blob/master/Image/image-20200715002125404.png)

这张图的最后三步，就是一个两阶段提交的过程，“写入redolog 处于prepare阶段”就是已经写入了redolog buffer了，但此时状态是prepare，再写binlog，再把redolog状态置为commit状态。可以设想，如果redolog写失败，这时候宕机了，这就是一次不完整的事务，可以把redolog舍弃掉。如果redolog和binlog都完成了发生了宕机，其实是没有影响的，因为redolog已经处于prepare阶段，重启时自动置为commit状态即可，不会有数据丢失。



总的而言，两阶段提交可以保证数据的持久性

## Binlog

Binlog存在于server层，因为一开始mysql是没有Innodb存储引擎的，一开始mysql默认的是myisam，仅仅依靠Binlog(归档日志)是无法完成crash safe的。所有的存储引擎都有Binlog。

Binlog是在事务提交的时候写，属于逻辑日志，redolog是物理日志，也就是redolog按配置和innodb自身的数据页进行记录存储，Binlog记录的事这个语句的原始逻辑，比如上图中的给ID = 2这一行的C字段 + 1，是从逻辑上翻译的一种日志。

