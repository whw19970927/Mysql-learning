# count(×)这么慢我该怎么办
## 为什么慢？
在开发系统的时候，我们经常需要计算一个表的行数，一条 select count(*) from t 语句不就解决了吗？
但是，随着系统中记录数越来越多，这条语句执行得也会越来越慢
Myisam是通过计数的方式获取行数，但innodb是如何工作的？
先看看下表的返回结果

![Image](https://img-blog.csdnimg.cn/20200802120605293.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

回话A的查询结果是10000
会话B的查询结果是10002
会话C的查询结果是10001
InnoDB执行 count(*) 的时候，需要把数据一行一行地从引擎里面读出来，然后累积计数。

![Image](https://img-blog.csdnimg.cn/2020080212443345.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

如果通过`show table status`语句，查询出来的只能是估算的行数，误差可能至多到40%-50%

![Image](https://img-blog.csdnimg.cn/20200802124932241.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

所以只能一行一行读来得到行数，所以行数越大 count(*)时间越长，所以速度越慢。

总结：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200802125112769.png)

## 应用场景一
### 2亿用户，mis后台的用户列表功能，怎么读取总数？**

**方案一**
写死总数，mis后台不需要精确总数。     

**方案二**
redis计数，新注册用户+1，注销用户-1，但存在mysql和redis数据不一致的问题，如果常见应用多一致性要求不高可以考虑。比如用户账单是需要强一致性的，方案二就不可以。

**方案三**
Mysql新加一张计数表， 通过事务保证一致性，也就是插数据的时候，计数表也跟着更改。但写性能会有折扣，因为要改两张表。

**Q：那如何做分页？**
`SELECT * FROM TABLE LIMIT N,M;`是一种方法，表明查询从N开始后M个
例如
`SELECT * FROM table LIMIT 5,10; // 检索记录行 6-15`
但这条语句的速度还是很慢的

![Image](https://img-blog.csdnimg.cn/20200802132406962.png)

上图可以看到偏移量为500000的时候，查询需要400ms
MySQL 处理 limit N,M 的做法就是按顺序一个一个地读出来数据集，limit只做过滤，然后丢掉前 N 个，剩下M个记录作为返回结果，因此这一步需要扫描 N+M 行；N越大扫的行数越多； `SELECT * FROM t  LIMIT N,M `无where条件或where条件就是主键索引时， N值越大越需要遍历大量数据页，耗费的时间也越久
https://www.cnblogs.com/goloving/p/8565017.html
这篇博客说的很详细

### 极端情况带where条件，分页如何做

![Image](https://img-blog.csdnimg.cn/20200802133737603.png)

![Image](https://img-blog.csdnimg.cn/20200802133806164.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

带where条件走普通索引，数据到20w偏移量就要600ms，已经很慢了。
 
因为这里需要回表查询，因为没有走主键索引。所以需要有一个回到主键索引树搜素的过程，称之为回表。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200802135526276.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)
总结：SELECT * FROM t WHERE K=X LIMIT N,M，是根据 k索引存储的主键去查找对应的行。K索引和主键索引数据不在同一个物理块上，N值越大越需要遍历大量索引页和数据页，耗费的时间就越久。

**如何避开回表？**
利用覆盖索引优化，减少无用回表。

![Image](https://img-blog.csdnimg.cn/2020080213583979.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)
 
 这一条在刚才的链接里也有提到
 //这次我们之间查询最后一页的数据（利用覆盖索引，只包含id列），如下：
`select id from product limit 866613, 20 0.2秒`
//相对于查询了所有列的37.44秒，提升了大概100多倍的速度

//那么如果我们也要查询所有列，有两种方法，
//一种是id>=的形式，
//另一种就是利用join，看下实际情况：

`SELECT * FROM product WHERE ID > =(select id from product limit 866613, 1) limit 20`
//查询时间为0.2秒！

//另一种写法
`SELECT * FROM product a JOIN (select id from product limit 866613, 20) b ON a.ID = b.id`
//查询时间也很短！![在这里插入图片描述](https://img-blog.csdnimg.cn/20200802142705549.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

## 不同的 count 用法
 主要是 count(*)、count(主键 id)、count(字段) 和 count(1) 等不同用法的性能，有哪些差别
 count() 是一个聚合函数，对于返回的结果集，一行行地判断，如果 count 函数的参数不是 NULL，累计值就加 1，否则不加。最后返回累计值。
 count(*)，count(主键 id) 和 count(1) 都表示返回满足条件的结果集的总行数；而 count(字段），则表示返回满足条件的数据行里面，参数“字段”不为 NULL 的总个数。
 
**对于 count(主键 id) 来说**

InnoDB 引擎会遍历整张表，把每一行的 id 值都取出来，返回给 server 层。server 层拿到 id 后，判断是不可能为空的，就按行累加。

**对于 count(1) 来说**

InnoDB 引擎遍历整张表，但不取值。server 层对于返回的每一行，放一个数字“1”进去，判断是不可能为空的，按行累加。

单看这两个用法的差别的话，你能对比出来，count(1) 执行得要比 count(主键 id) 快。因为从引擎返回 id 会涉及到解析数据行，以及拷贝字段值的操作。

**对于 count(字段) 来说：**

如果这个“字段”是定义为 not null 的话，一行行地从记录里面读出这个字段，判断不能为 null，按行累加；

如果这个“字段”定义允许为 null，那么执行的时候，判断到有可能是 null，还要把值取出来再判断一下，不是 null 才累加。

也就是前面的第一条原则，server 层要什么字段，InnoDB 就返回什么字段。

但是 **count(*) **是例外，并不会把全部字段取出来，而是专门做了优化，不取值。count(*) 肯定不是 null，按行累加。

看到这里，你一定会说，优化器就不能自己判断一下吗，主键 id 肯定非空啊，为什么不能按照 count(*) 来处理，多么简单的优化啊。

当然，MySQL 专门针对这个语句进行优化，也不是不可以。但是这种需要专门优化的情况太多了，而且 MySQL 已经优化过 count(*) 了，你直接使用这种用法就可以了。

所以结论是：按照效率排序的话，count(字段)<count(主键 id)<count(1)≈count(*)
