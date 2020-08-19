# MySQL有哪些“饮鸩止渴”提高性能的方法
 ## 短连接风暴

![Image](https://img-blog.csdnimg.cn/20200814101052772.png#pic_center)

 正常的短连接模式就是连接到数据库后，执行很少的 SQL 语句就断开，下次需要的时候再重连。如果使用的是短连接，在业务高峰期的时候，就可能出现连接数突然暴涨的情况。
 
 在数据库压力比较小的时候，这些额外的成本并不明显。

但是，短连接模型存在一个风险，就是一旦数据库处理得慢一些，连接数就会暴涨。max_connections 参数，用来控制一个 MySQL 实例同时存在的连接数的上限，超过这个值，系统就会拒绝接下来的连接请求，并报错提示“Too many connections”。对于被拒绝连接的请求来说，从业务角度看就是数据库不可用。

在机器负载比较高的时候，处理现有请求的时间变长，每个连接保持的时间也更长。这时，再有新建连接的话，就可能会超过 max_connections 的限制。

![Image](https://img-blog.csdnimg.cn/20200819150634904.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70#pic_center)

如何避免短连接风暴？

![Image](https://img-blog.csdnimg.cn/20200819150722618.png#pic_center)

**慢查询：**

![Image](https://img-blog.csdnimg.cn/20200819180951821.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70#pic_center)

慢查询日志，顾名思义，就是查询慢的日志，是指mysql记录所有执行超过long_query_time参数设定的时间阈值的SQL语句的日志。该日志能为SQL语句的优化带来很好的帮助。默认情况下，慢查询日志是关闭的，要使用慢查询日志功能，首先要开启慢查询日志功能。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200819154117370.png#pic_center)
当Mysql性能下降时，通过开启慢查询来获得哪条SQL语句造成的响应过慢，进行分析处理。当然开启慢查询会带来CPU损耗与日志记录的IO开销，所以我们要间断性的打开慢查询日志来查看Mysql运行状态。
慢查询能记录下所有执行超过long_query_time时间的SQL语句, 用于找到执行慢的SQL, 方便我们对这些SQL进行优化.
所以慢查询是比较常用的sql优化的方式

* 框架级别对mysql操作的慢⽇志及告警；
* 关注程序使⽤的mysql监控
* 对陌⽣的⼤表进⾏查询时，先explain⼀下是否使⽤了合适的索引，适
当运⽤force index 
* 写每条sql时，⽆论当前qps如何，主动关注表上的现有索引
* 关注主/从库
在一些比较大的语句的explain时，需要注意

![Image](https://img-blog.csdnimg.cn/20200819160700958.png#pic_center)

SQL聚合函数：
–COUNT：统计行数量
–SUM：获取单个列的合计值
–AVG：计算某个列的平均值
–MAX：计算列的最大值
–MIN：计算列的最小值

例如：SELECT AVG(student_age)FROM t_student;

**QPS突增问题**
有时候由于业务突然出现高峰，或者应用程序 bug，导致某个语句的 QPS 突然暴涨，也可
能导致 MySQL 压力过大，影响服务。

一般应对不同的场景有如下情况（暂不理解，后续结合业务学习）
* 一种是由全新业务的 bug 导致的。假设你的 DB 运维是比较规范的，也就是说白名单是
一个个加的。这种情况下，如果你能够确定业务方会下掉这个功能，只是时间上没那么
快，那么就可以从数据库端直接把白名单去掉。
* 如果这个新功能使用的是单独的数据库用户，可以用管理员账号把这个用户删掉，然后
断开现有连接。这样，这个新功能的连接不成功，由它引发的 QPS 就会变成 0。
* 如果这个新增的功能跟主体功能是部署在一起的，那么我们只能通过处理语句来限制。
这时，我们可以使用上面提到的查询重写功能，把压力最大的 SQL 语句直接重写
成"select 1"返回
