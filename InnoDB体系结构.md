落盘机制：

* 定时master thread 1s刷新 or 10s刷新
* 业务使用（提交或查询）
* 缓冲区达到阈值

**Q:为什么change buffer不适用于唯一索引？**

A:对于唯一索引来说，所有的更新操作都要先判断这个操作是否违反唯一性约束。而这必须要将数据页读入内存才能判断。如果都已经读入到内存了，那直接更新内存会更快，就没必要使用 change buffer 了。

因此，唯一索引的更新就不能使用 change buffer，实际上也只有普通索引可以使用。

## 日志缓冲区

![image](https://github.com/whw19970927/Mysql-learning/blob/master/Image/image-20200716102302781.png)

**Q：为什么需要redo log？**
假设仅仅修改了一个字段，或者是一个字节，把值从1改成2，如果仅仅是这个操作就需要往磁盘上写一次数据的话代价是非常大的，因为内存和磁盘的交互是通过页的形势的，
一个页的大小是16kb，为了这一个字节要做16kb的io是不是代价太大了。redo log就仅仅记录了在哪个表空间的哪个页对什么数据进行了修改，仅仅记录了你的操作，然后把你的操作的数据刷到磁盘
，这是一个顺序写的过程。如果是修改例如，将所有一年级的年龄改为一个值，这种在磁盘上更改数据的行为是随机写的，之前老师提到过随机写的效率是比顺序写低很多的。我们记录变化往redolog里写是一个顺序写
所以redo log的好处在于redolog虽然也是按页单位交互的，但只记录了发生了什么变化，同时是顺序写的操作。

## InnoDB磁盘架构

InnoDB的主要的磁盘文件主要分为三大块：一是系统表空间，二是用户表空间，三是redo日志文件和归档文件。二进制文件(binlog)等文件是MySQL Server层维护的文件，所以未列入InnoDB的磁盘文件中

### InnoDB表空间

![image](https://github.com/whw19970927/Mysql-learning/blob/master/Image/image-20200716102319395.png)

表空间由段、区、页组成

每张表的表空间内存放的只是数据、索引和插入缓冲，其他类的数据，如撤销（Undo）信息、系统事务信息、二次写缓冲（double write buffer）等还是存放在原来的共享表空间内。即使在启用了参数innodb_file_per_table之后，共享表空间还是会不断地增加其大小

常见的段有数据段、索引段、回滚段等，表空间是由分散的页和段组成

**区**是由64个连续的页组成的，每个页大小为16KB，即每个区的大小为1MB

而页也就是InnoDB磁盘管理的最小单位，默认大小是16kb

## Mysql优化决策

Mysql优化是基于成本的，成本的计算公式是 IO成本 + CPU成本

* I/O成本：从磁盘到内存这个加载的过程损耗的时间称之为I/O成本

* CPU成本：读取以及检测记录是否满足对应的搜索条件、对结果集进行排序等这些操作损耗的时间称之为CPU成本

基于成本优化的步骤

* 根据搜索条件找出所有可能的索引
* 计算全表扫描的代价
* 计算使用不同索引执行查询的代价
* 对比各种执行方案，找出成本最低的一个

例如：
这张表

CREATE TABLE single_table (
    id INT NOT NULL AUTO_INCREMENT,
    key1 VARCHAR(100),
    key2 INT,
    key3 VARCHAR(100),
    key_part1 VARCHAR(100),
    key_part2 VARCHAR(100),
    key_part3 VARCHAR(100),
    common_field VARCHAR(100),
    PRIMARY KEY (id),
    KEY idx_key1 (key1),
    UNIQUE KEY idx_key2 (key2),
    KEY idx_key3 (key3),
    KEY idx_key_part(key_part1, key_part2, key_part3)
) Engine=InnoDB CHARSET=utf8;

查询：

SELECT * FROM single_table WHERE  key1 IN ('a', 'b', 'c') AND  key2 > 10 AND key2 < 1000 AND  key3 > key2 AND  key_part1 LIKE '%hello%' AND common_field = '123';

### 全表扫描

全表扫描就是把聚簇索引中的记录都依次和给定的搜索条件做一下比较，把符合条件的记录加入到结果集里，所以需要聚簇索引对应的页面加载到内存中，然后检测记录是否满足条件

**全表扫描代价如何计算？**

首先需要知道两条数据

* 聚簇索引的页面数（通过页面数可以计算io成本）

* 该表中的记录数（聚簇索引的记录都需要和搜索条件作比较）

可以用show table status like 'tableName' \G

![image](https://github.com/whw19970927/Mysql-learning/blob/master/Image/image-20200716103431334.png)

在这里可以计算出

聚簇索引的页面数量 = 1589248 / 16（每页字节数） / 1024（字节数）= 97

IO代价 + CPU成本 = [ 97 x 1.0 + 1.1] + [ 9693 x 0.2 + 1.0]

### 计算不同索引执行查询的代价

Mysql计算索引代价时，会分别分析单独使用这些索引的查询成本，最后还要分析是否会使用到索引合并，同时，**Mysql查询优化器会先分析使用唯一二级索引的成本，再分析使用普通索引的成本。**

这里只有idx_key1和idx_key2会用到索引，后者满足有限计算唯一二级索引的条件，所以先计算。一般会对于回表方式的查询。

#### idx_key2

这里需要的参数有：

**范围区间数量**
粗暴的认为读取索引的一个范围区间的I/O成本和读取一个页面是相同的。1 x 1.0 = 1.0
      
**需要回表的记录数**
区间最左记录 与 区间最右记录
父节点中对应的目录项记录之间隔着几条记录

计算方式将二者相乘 + 1

I/O成本 = 1.0（二级索引的页面） + 1.0（读取索引的IO成本，上面有提到） * 95（回表的记录数量）

CPU成本 = 95 x 0.2 + 0.01 + 95 x 0.2 = 38.01 （读取二级索引记录的成本 + 读取并检测回表后聚簇索引记录的成本）

IO成本 + CPU成本 = 最后计算的成本


#### idx_key1


* 范围区间数量 3 * 1.0
* 需要回表的记录数  a + b + c区间的和

![image](https://github.com/whw19970927/Mysql-learning/blob/master/Image/image-20200716102059031.png)



IO成本：3.0 + 118 x 1.0 = 121.0 (范围区间的数量 + 预估的二级索引记录条数)

CPU成本：118 x 0.2 + 0.01 + 118 x 0.2 = 47.21 （读取二级索引记录的成本 + 读取并检测回表后聚簇索引记录的成本）

终上所述，很明显idx_key2的成本计算成本是最低的，所以选择idx_key2来执行查询。

承接上文，基本就是通过这种计算方式来找出成本最低的一个进行优化。



### 基于索引统计数据的成本计算

有时候使用索引执行查询会出现许多单点区间，比如刚刚的where k1 IN ('a', 'b', 'c')，计算对应的二级索引记录条数需要获取索引对应的B+树的最左区间和最右区间，然后再计算这两条记录之前有多少记录。

*以上的记录条数的方式称为index dive*

http://tech.it168.com/a2014/0805/1653/000001653165.shtml

(这个链接是对index dive的解释)

但是如果IN的内容很多，如果有两万次，那难道要做两万次的区间查询么

这种行为无疑非常耗费性能，所以Mysql提供系统变量eq_range_index_dive_limit，

![image](https://github.com/whw19970927/Mysql-learning/blob/master/Image/image-20200716113913569.png)

也就是说如果我们的IN语句中的参数个数小于200个的话，将使用index dive的方式计算各个单点区间对应的记录
条数，如果大于或等于200个的话，可就不能使用index dive了，要使用所谓的索引统计数据来进行估算。像会为
每个表维护一份统计数据一样，MySQL也会为表中的每一个索引维护一份统计数据，查看某个表中索引的统计数
据可以使用 SHOW INDEX FROM 表名的语法。


Cardinality属性表示索引列中不重复值的个数。比如对于一个一万行记录的表来说，某个索引列的Cardinality属性是10000，那意味着该列中没有重复的值，如果Cardinality属性是1的话，就意味着该列的值全部是重复的。不过需要注意的是，对于InnoDB存储引擎来说，使用SHOW INDEX语句展示出来的某个索引列的Cardinality属性是一个估计值，并不是精确的。可以根据这个属性来估算IN语句中的参数所对应的记录数：

- 使用SHOW TABLE STATUS展示出的Rows值，也就是一个表中有多少条记录。
- 使用SHOW INDEX语句展示出的Cardinality属性。
- 根据上面两个值可以算出idx_key1索引对于的key1列平均单个值的重复次数：Rows/Cardinality
- 所以总共需要回表的记录数就是：IN语句中的参数个数*Rows/Cardinality
