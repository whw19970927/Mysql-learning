# orderby是怎么工作的
首先有如下表，

![Image](https://img-blog.csdnimg.cn/2020080423441869.png)

需求：查询城市是“杭州”按照姓名排序的前1000个人的姓名、年龄、城市信息

一般这么写

![Image](https://img-blog.csdnimg.cn/20200804235034421.png)

主要涉嫌到两部分字段，一个是查询的字段，一个排序的字段

**addon字段：需要查询额外存储的字段，语句中的city，name，age**
**sort字段 ： 用于排序的字段，语句中的name**

## sort_buffer

![Image](https://img-blog.csdnimg.cn/20200804235904881.png)

在官方文档中有sort_buffer_size相关设置

![Image](https://img-blog.csdnimg.cn/20200805000503410.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

## 全字段排列过程

全字段排序指，我在取数据的时候就已经将所有字段放在sort_buffer中了
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200805000129469.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)
在经过上述排序过程后，将前1000行返回给客户端

**Q: 排序内存不够了怎么办？**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200805000709787.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)
图中“按 name 排序”这个动作，可能 在内存中完成，也可能需要使用外部排序，这取决于排序所需的内存和参数 sort_buffer_size。
如果要排序的数据量<sort_buffer_size，排序就在内存中完成。但如果排序数据量太大，内存放不下，则不得不利用磁盘临时文件辅助排序。
可以用explain计划看一下语句

![Image](https://img-blog.csdnimg.cn/20200805000956512.png)

可以看到命中city索引并且Using filesort（使用排序）
使用OPTIMIZER_TRACE能查看是否使用了临时文件，排序模式等更多详细信息  

![Image](https://img-blog.csdnimg.cn/20200805001158781.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

第四步打印出来的是JSON格式信息

![Image](https://img-blog.csdnimg.cn/20200805001512261.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)
*  filesort_priority_queue_optimization 使用优先队列进行排序的分析 
*  number_of_tmp_files 表示的是，排序过程中使用的临时文件数。 临时文件数量越大说明sort_buffer越来越不够用，都在借助磁盘信息进行排序
*  sort_mode 采用了哪种排序方式 . 这里的chosen为false表明并没有使用优先队列排序
*  examined_rows 检查了多少行数据记录 
*  sort_buffer_size 占用了多少排序空间

**Q: number_of_tmp_files的值大小和sort_buffer_size有什么关系？**

sort_buffer_size设置越小， number_of_tmp_files可能就越大。

内存放不下时，例如图中number_of_tmp_files 有值存在时， 就需要使用外部排序，外部排序一般使用归并排序算法。可以这么简单理解，MySQL 将需要排序的数据分成 2 份，每一份单独排序后存在这些临时文件中。然后把这  2 个有序文件再合并成一个有序的大文件。

## 全字段排序的问题
如果排序语句查询返回的字段(addon字段)很多，单行记录数据很大怎么办？
单条记录数据很大，加载到sort_buffer中的记录数就会很少，使用到的临时文件很多，排序效果很不理想

例如：
我们的业务中有一张经常使用的数据表，表中含有几个text、varchar(2048)这种大的字段，造成了InnoDB一个数据页存储数据量很少，查询需要加载更多的数据页。这个时候数据表该怎样优化？

将表中大的字段拆分到另外一张表中去，通过主键id进行关联

## rowId排序
如果Mysql认为排序的单行长度太大，会采用另一种算法，也就是rowId

**max_length_for_sort_data**，是 MySQL 中专门控制用于排序的行数据的长度的一个参数。它的意思是，如果单行的长度超过这个值，MySQL 就认为单行太大，要换一个算法。 实现思路:使用rowid 排序时，内存中sort_buffer只将rowid或者主键做为额外的字段，然后进行回表抽取数据。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200805094153520.png)


**Q:排序字段太大对排序有什么影响？**
排序字段过大会导致外排，sort_buffer装不下，导致性能降低。

 **全字段排序中将额外存储字段(addon字段)加载到sort_buffer中时使用的是字段实际占用字节大小还是字段在表中的定义大小？** 
会使用实际占用的字节大小。例如varchar，会使用元素实际大小。

 **Q：你认为mysql是怎样计算排序单行长度值的？以此来决定使用row id排序还是全字段排序。**

但这时，排序的结果就因为少了 city 和 age 字段的值，不能直接返回了，整个执行流程就变成如下所示的样子：

* 初始化 sort_buffer，确定放入两个字段，即 name 和 id；

* 从索引 city 找到第一个满足 city='杭州’条件的主键 id，也就是图中的 ID_X；

* 到主键 id 索引回表取出整行，取 name、id 这两个字段，存入 sort_buffer 中；

* 从索引 city 取下一个记录的主键 id；

* 重复步骤 3、4 直到不满足 city='杭州’条件为止，也就是图中的 ID_Y；

* 对 sort_buffer 中的数据按照字段 name 进行排序；

* 遍历排序结果，取前 1000 行，并按照 id 的值回到原表中取出 city、name 和 age 三个字段返回给客户端。

![Image](https://img-blog.csdnimg.cn/20200805094609499.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

### rowid 排序 & 全字段对比

![Image](https://img-blog.csdnimg.cn/20200805095443478.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

![Image](https://img-blog.csdnimg.cn/20200805095446674.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)
可以看到chosen和sort_mode产生了变化，同时number_of_tmp_files变成0了，并且选择了优先队列排序。说明采用rowid后可以更好利用sort_buffer空间，因为只将name和id放入sort_bufffer中，字段更少。

如果 MySQL 实在是担心排序内存太小，会影响排序效率，才会采用 rowid 排序算法，这样排序过程中一次可以排序更多行，但是需要再回到原表去取数据。

如果 MySQL 认为内存足够大，会优先选择全字段排序，把需要的字段都放到 sort_buffer 中，这样排序后就会直接从内存里面返回查询结果了，不用再回到原表去取数据。

这也就体现了 MySQL 的一个设计思想：如果内存够，就要多利用内存，尽量减少磁盘访问。

### 优化
#### 不排序行不行？

MySQL 做排序是一个成本比较高的操作。那是不是所有的 order by 都需要排序操作呢？如果不排序就能得到正确的结果，那对系统的消耗会小很多，语句的执行时间也会变得更短。MySQL 之所以需要生成临时表，并且在临时表上做排序操作，其原因是原来的数据都是无序的。如果能够保证从 city 这个索引上取出来的行，天然就是按照 name 递增排序的话，就不需要排序了。可以创建联合索引

```sql
alter table t add index city_user(city, name);
```
在这个索引里面，我们依然可以用树搜索的方式定位到第一个满足 city='杭州’的记录，并且额外确保了，接下来按顺序取“下一条记录”的遍历过程中，只要 city 的值是杭州，name 的值就一定是有序的。

这样整个查询过程的流程就变成了：

* 从索引 (city,name) 找到第一个满足 city='杭州’条件的主键 id；

* 到主键 id 索引取出整行，取 name、city、age 三个字段的值，作为结果集的一部分直接返回；

* 从索引 (city,name) 取下一个记录主键 id；

* 重复步骤 2、3，直到查到第 1000 条记录，或者是不满足 city='杭州’条件时循环结束。

![Image](https://img-blog.csdnimg.cn/20200805100450730.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

也就是无需在sort_buffer进行操作了


#### 不回表行不行？
可以采用覆盖索引避免回表查询。

```sql
alter table t add index city_user_age(city, name, age);
```
这时，对于 city 字段的值相同的行来说，还是按照 name 字段的值递增排序的，此时的查询语句也就不再需要排序了。这样整个查询语句的执行流程就变成了：

* 从索引 (city,name,age) 找到第一个满足 city='杭州’条件的记录，取出其中的 city、name 和 age 这三个字段的值，作为结果集的一部分直接返回；

* 从索引 (city,name,age) 取下一个记录，同样取出这三个字段的值，作为结果集的一部分直接返回；

* 重复执行步骤 2，直到查到第 1000 条记录，或者是不满足 city='杭州’条件时循环结束。

## mysql排序算法
内存排序（优先队列 order by limit 返回少量行常用，提高排序效率，但是注意order by limit n,m 如果n过大可能会涉及到排序算法的切换 
内存排序（快速排序）
归并排序 （需要磁盘临时文件辅助）

![Image](https://img-blog.csdnimg.cn/2020080510203336.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

**Q：排序的单行长度怎么计算的？**
选择排序算法的主要标准就是参数max_length_for_sort_data，其默认大小为1024字节，单行长度的计算为（sort字段长度+addon字段的总和）是否超过了max_length_for_sort_data。其次如果使用了“全字段排序”算法，那么将会对addon字段的每个字段做一个pack（打包），主要目的在于压缩那些为空的字节，节省空间。例如变长字段的压缩。
* 循环本表全部字段
* 过滤掉不需要存储的字段

 ![Image](https://img-blog.csdnimg.cn/20200805103449436.png)
 
* 获取字段的长度：这里就是实际的长度了比如我们的a1 varchar(300)，且字符集为UTF8，那么其长度≈ 300*3 （900）
* 获取可以pack(打包)字段的长度，只有可变长度的类型的字段才需要进行打包技术。
* 循环结束，获取addon字段的总长度，获取可以pack（打包）字段的总长度，循环结束后可以获取addon字段的总长度，但是需要注意addon字段和sort字段可能包含重复的字段
* .  选择算法的条件： addon字段的总长度+sort字段的总长度 > max_length_for_sort_data

![Image](https://img-blog.csdnimg.cn/20200805103540473.png)

**Q：每次排序一定会分配sort_buffer_size参数指定的内存大小吗？**

不是这样的，MySQL会做一个初步的计算，通过比较Innodb中聚集索引可能存储的行上限和sort_buffer_size参数指定大小内存可以容纳的行上限，获取它们小值 进行确认最终内存分配的大小，目的在于节省内存空间。

* 大概计算出Innodb层主键叶子结点的行数这一步主要通过（聚集索引叶子结点的空间大小/聚集索引每行大小 * 2）计算出一个行的上限，调入函数ha_innobase::estimate_rows_upper_bound，源码如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200805103743683.png)
* 根据前面计算的每行长度计算出sort buffer可以容下的最大行数，这一步将计算sort buffer可以容纳的最大行数如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200805103808751.png)
* 对比两者的最小值，作为分配内存的标准

![Image](https://img-blog.csdnimg.cn/20200805103826869.png)

* 根据结果分配内存

![Image](https://img-blog.csdnimg.cn/20200805103841107.png)

**Q：排序中的一行记录如何组织？**

一行排序记录，由sort字段+addon字段 组成，其中sort字段为order by 后面的字段，而addon字段为需要访问的字段，比如‘select a1,a2,a3 from test order by a2,a3’，其中sort字段为‘a2,a3’，addon字段为‘a1,a2,a3’。sort字段中的可变长度字段不能打包（pack）压缩，比如varchar，使用的是定义的大小计算空间，注意这是排序使用空间较大的一个重要因素。

 如果在计算sort字段空间的时候，某个字段的空间大小大于了max_sort_length大小则按照max_sort_length指定的大小计算。 

一行排序记录，如果sort字段+addon字段 的长度大于了max_length_for_sort_data的大小，那么addon字段将不会存储，而使用sort字段+ref字段代替，ref字段为主键或者ROWID，这个时候就会使用original filesort algorithm（回表排序）的方式了。 

如果addon字段包含可变字段比如varchar字段，则会使用打包（pack）技术进行压缩，节省空间
