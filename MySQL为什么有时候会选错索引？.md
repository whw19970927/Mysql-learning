本讲说明了一些明明有索引，但mysql却选错了索引导致了速度很慢的情况。

## 案例1

先建一张表：

**CREATE TABLE `t` (**

   **`id` int(11) auto_increment,** 

   **`a` int(11) DEFAULT NULL,** 

   **`b` int(11) DEFAULT NULL,** 

   **PRIMARY KEY (`id`), KEY `a` (`a`), KEY `b` (`b`)**

**) ENGINE=InnoDB**;



随后插入100000条数据

delimiter ;;

create procedure idata()

begin

 declare i int;

 set i=1;

 while(i<=100000)do

  insert into t (`a`,`b`) values(i, i);

  set i=i+1;

 end while;

end;;

delimiter ;



随后执行如图操作：

![image](https://img-blog.csdnimg.cn/20200729095911377.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

**注：在mysql较高版本已经由tx_isolation改成transaction_ioslation**

执行语句会发现做select操作的时候，并没有走索引，而是全表扫描

![image](https://img-blog.csdnimg.cn/20200729100613808.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

（*这个例子我个人机器走的还是范围查询*

![image](https://img-blog.csdnimg.cn/2020072910002113.png)

于是重新删表做了第二次测试

![image](https://img-blog.csdnimg.cn/20200729100037272.png)

**为什么开启了session A,session B扫描行数变成10W？**

因为通过索引优化的机制，mysql的优化器会根据每个索引的cost进行评估，并选择cost较小的索引。

![image](https://img-blog.csdnimg.cn/2020072910004568.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

![image](https://img-blog.csdnimg.cn/20200729100056764.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

![image](https://img-blog.csdnimg.cn/20200729100113175.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)
cost计算公式如图，是将数据大小除每一页的数据量 + 总行数 * 0.2



## 统计数据

**InnoDB提供了两种存储统计数据的方式：**

* 统计数据存储在磁盘上。

* 统计数据存储在内存中，当服务器关闭时这些这些统计数据就都被清除掉了。

MySQL给我们提供了系统变量innodb_stats_persistent来控制到底采用哪种方式去存储统计数据。在MySQL 5.6.6之前，innodb_stats_persistent的值默认是OFF，也就是说InnoDB的统计数据默认是存储到内存的，之后的版本中innodb_stats_persistent的值默认是ON，也就是统计数据默认被存储到磁盘中。

不过InnoDB默认是以表为单位来收集和存储统计数据的，也就是说我们可以把某些表的统计数据（以及该表的索引统计数据）存储在磁盘上，把另一些表的统计数据存储在内存中。我们可以在创建和修改表的时候通过指定STATS_PERSISTENT属性来指明该表的统计数据存储方式。

![image](https://img-blog.csdnimg.cn/20200729100118731.png)



## 统计数据策略

1、基于磁盘的永久性统计数据

当我们选择把某个表以及该表索引的统计数据存放到磁盘上时，实际上是把这些统计数据存储到了两个表里：

* innodb_table_stats存储了关于表的统计数据，每一条记录对应着一个表的统计数据

* innodb_index_stats存储了关于索引的统计数据，每一条记录对应着一个索引的一个统计项的统计数据



2、定期更新统计数据

* 系统变量innodb_stats_auto_recalc决定着服务器是否自动重新计算统计数据，它的默认值是ON，也就是该功能默认是开启的。每个表都维护了一个变量，该变量记录着对该表进行增删改的记录条数，如果发生变动的记录数量超过了表大小的10%，并且自动重新计算统计数据的功能是打开的，那么服务器会重新进行一次统计数据的计算，并且更新innodb_table_stats和innodb_index_stats表。不过自动重新计算统计数据的过程是异步发生的，也就是即使表中变动的记录数超过了10%，自动重新计算统计数据也不会立即发生，可能会延迟几秒才会进行计算。

* 如果innodb_stats_auto_recalc系统变量的值为OFF的话，我们也可以手动调用ANALYZE TABLE语句来重新计算统计数据。ANALYZE TABLE single_table;



3、控制执行计划

Index Hints

* USE INDEX：限制索引的使用范围，在数据表里建立了很多索引，当MySQL对索引进行选择时，这些索引都在考虑的范围内。但有时我们希望MySQL只考虑几个索引，而不是全部的索引，这就需要用到USE INDEX对查询语句进行设置。

* IGNORE INDEX ：限制不使用索引的范围

* FORCE INDEX：我们希望MySQL必须要使用某一个索引(由于 MySQL在查询时只能使用一个索引，因此只能强迫MySQL使用一个索引)。这就需要使用FORCE INDEX来完成这个功能。



## 案例2

![image](https://img-blog.csdnimg.cn/20200729100143291.png)

使用`force index(a)`强制使用索引，就会走范围查询使用索引，效率也会提升

![image](https://img-blog.csdnimg.cn/2020072910014993.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

可以看到第一个的查询时间是Query_time是0.031，但使用了强制索引之后只用了大概0.016



### 原因对比



![image](https://img-blog.csdnimg.cn/20200729100154564.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

![image](https://img-blog.csdnimg.cn/20200729100201628.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

对上图扫描分析注释：

- index_dives_for_eq_ranges：是否使用了index dive，该值会被参数eq_range_index_dive_limit变量值影响。
- rowid_ordered：该range扫描的结果集是否根据PK值进行排序
- using_mrr：是否使用了mrr
- index_only：表示是否使用了覆盖索引
- rows：扫描的行数
- cost：索引的使用成本
- chosen：表示是否使用了该索引

第一幅图是不使用强制索引的情况下，会对cost进行评估并选择索引。并且chosen是false，表明没有使用索引，并且注明了原因cause，也就是对cost进行了评估

第二张图则chosen是直接true，说明使用了索引，所以速度也要快不少



## 案例3

建如下表：

create table t1(id int auto_increment primary key, a int, b int, c int, v varchar(1000), key iabc(a,b,c), key ic(c)) engine = innodb;



insert into t1 select null,null,null,null,null;

insert into t1 select null,null,null,null,null from t1;

insert into t1 select null,null,null,null,null from t1;

insert into t1 select null,null,null,null,null from t1;

insert into t1 select null,null,null,null,null from t1;

insert into t1 select null,null,null,null,null from t1;



update t1 set a=id/2, b=id/4, c=6-id/8, v=repeat('a',1000);



使用explain分析一下

![image](https://img-blog.csdnimg.cn/20200729100209360.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

MySQL5.6遇到order by limit时，由于只考虑能够优化 order by的索引，而放弃了其他更好的索引，导致的bug，那么MySQL5.7的解决办法是进入low_limit后，不是只考虑优化order by的索引，而是考虑了全部索引。（*现在使用8.0.20版本，不会出现上述问题*）

https://www.jianshu.com/p/7ce0421751be



## optimizer_trace

- **QUERY**

跟踪语句的文本。

![image](https://img-blog.csdnimg.cn/20200729100217232.png)

* **TRACE:**

跟踪，JSON格式。



* **MISSING_BYTES_BEYOND_MAX_MEM_SIZE:**

每个记住的跟踪都是一个字符串，随着优化的进行扩展并将其附加数据。该 optimizer_trace_max_mem_size 变量设置所有当前记忆的跟踪所使用的内存总量的限制。如果达到此限制，则当前跟踪不会扩展（因此是不完整的），并且该MISSING_BYTES_BEYOND_MAX_MEM_SIZE 列显示该跟踪丢失的字节数。



* **INSUFFICIENT_PRIVILEGES:**

如果跟踪的查询使用SQL SECURITY值为的 视图或存储的例程 DEFINER，则可能是拒绝了除定义者之外的其他用户查看查询的跟踪。在这种情况下，跟踪显示为空， INSUFFICIENT_PRIVILEGES值为1。否则，值为0。

![image](https://img-blog.csdnimg.cn/20200729100221963.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)
像之前的cost计算可以根据上图的json字符串中的内容计算



## analyze

在生产环境中，索引的更新操作可能非常频繁。如果再每次索引发生更新操作时就对其进行Cardinality的统计，那么将会对数据库带来很大的负担。另外需要考虑的是，如果一张表的数据量非常大，比如一张表有50GB的数据，那么统计一次Cardinality信息所需要的时间可能非常长。这在生产环境的应用中也是不能接受的。因此，数据库对于Cardinality的统计都是通过采样(sample)的方法来完成的。

  在InnoDB存储引擎中，Cardinality统计信息的更新发生在两个操作中：INSERT和UPDATE。根据前面的叙述，不可能在每次发生INSERT和UPDATE时都去更新Cardinality的信息，

这会增加数据库系统的负荷，同事对大表进行统计时，时间上也不允许。

因此InnoDB存储引擎对于更新Cardinality信息的策略为：

* show index from tablename可查看索引信息，其中Cardinality表示索引中唯一值的估计值（最好能接近1，如果非常小则可以考虑删除该索引），优化器根据该值判断是否选择使用该索引。

* cardinality值是通过简单抽样统计出来的，默认随机取8个叶子节点数据统计其平均值(不足8页则全取)。

* cardinality更新策略：表中1/16的数据已经发生变化或stat_modified_counter>2000000000（每个表维护一个stat_modified_counter，每次DML更新1行就加1，直到满足阈值则自动收集统计信息，并把此值清0；）

![image](https://img-blog.csdnimg.cn/20200729100228347.png)





使用ANALYZE TABLE分析表的过程中，数据库系统会对表加一个只读锁。在分析期间，只能读取表中的记录，不能更新和插入记录。ANALYZE TABLE语句能够分析InnoDB和MyISAM类型的表。

命令： ANALYZE TABLE tablename;

![image](https://img-blog.csdnimg.cn/20200729100234565.png)





## 总结

* **强制使用force index（key）进行语句查询。**



* **使用ANALYZE TABLE tablename; (不建议使用)**

 (会产生锁表的问题)

* **删除不必要的索引**
