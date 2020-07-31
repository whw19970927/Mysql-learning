## mysql体系结构

* Mysql采用单进程多线程架构

![image](https://github.com/whw19970927/Mysql-learning/blob/master/Image/image-20200713095023365.png)

 

### 物理结构

1.查看位置命令

![image](https://github.com/whw19970927/Mysql-learning/blob/master/image/image-20200713101057143.png)



2.数据目录

![image](https://github.com/whw19970927/Mysql-learning/blob/master/image/image-20200713101139909.png)



* Innodb如何存储数据的？

  1. Innodb使用页作为基本单位来管理存储空间，页大小为16KB
  2. 采用B+树数据结构，每一个节点都是一个数据页，在叶子节点通过双向链表进行维护。
  3. Innodb采用的是聚簇索引

* 系统表空间

  1.系统表空间可以对应文件系统上一个或多个实际的文件，默认，InnoDB会在数据目录下创建一个名为ibdata1

* 客户端服务端交互流程

![image](https://github.com/whw19970927/Mysql-learning/blob/master/image/image-20200713102108994.png)

* Q：Mysql为什么不支持全文索引？

  A:	Mysql插件式表存储引擎，如果一个库里有多张表，可以给每张表设立不同的存储引擎。

  Q：MySQL速度快是因为它不支持事务吗？

  Q：数据量大于1000万时MySQL的性能会急剧下降吗？

## Mysql逻辑架构

## Server层

包括连接器、查询缓存、分析器、优化器、执行器等，涵盖 MySQL 的大多数核心服务功能，以及所有的内置函数（如日期、时间、数学和加密函数等），所有跨存储引擎的功能都在这一层实现



### 向mysql发送一个请求

#### 1.**连接**器

（1）如果用户名和密码不对，我们就会收到一个“Access denied for user”的错误，然后客户端程序结束执行

（2）如果用户名跟密码认证通过，连接器会在权限表里查出我所拥有的权限，之后这个连接里面的权限判断逻辑，都将依赖于此时读到的权限

相关命令：show processlist

*尽量使用长链接，因为发起链接的代价很昂贵*

#### 2.查询缓存

MySQL将缓存存放在一个引用表中，通过一个哈希值引用，这个哈希值包括了以下因素，即查询本身、当前要查询的数据库、客户端协议的版本等一些其他可能影响返回结果的信息。

*在mysql8.0中已删除，因为当插入或者分析的操作很频繁时，缓存会经常失效很难命中*

#### 3.分析器

![image](https://github.com/whw19970927/Mysql-learning/blob/master/image/image-20200713105038314.png)

#### 4.优化器

![image](https://github.com/whw19970927/Mysql-learning/blob/master/image/image-20200713105343611.png)

同一句sql可能存在不用的执行计划

例如

select * from T1 t1 join T2 t2 using(ID) where t1.c = 10 and t2.d = 20;

可能存在如下这两种情况，这两种执行方法结果一样，但执行效率可能不同

![image](https://github.com/whw19970927/Mysql-learning/blob/master/image/image-20200713105636290.png)

优化器也就是会选择一个成本最少的执行计划最终执行，通过CBO进行优化



#### 5.执行引擎

各种不同的存储引擎向上层MySQL server层提供统一的调用接口（存储引擎API），会根据相对于的执行计划调用相对的api

**查询权限判断**

* 在执行器阶段仍然在判断权限，因为存在之前的步骤无法判断的内容，例如调用了存储过程，只有执行阶段才能进行判断，在分析器上无法判断

  https://segmentfault.com/q/1010000020193349

  （对于权限检查时机的讨论）

  1.如果命中查询缓存，会在查询缓存返回结果的时候，做权限验证

  2.查询也会在优化器之前调用precheck验证权限

  3.执行器判断一下我们对这个表T有没有执行查询的权限

**API获取数据过程**

     		1. 首先调用InnoDB的引擎接口获取表的第一行，判断该ID是否满足条件，如果满足条件则将结果缓存到结果集中，否则跳过
                  		2. 调用引擎接口读取下一行，重复第一步的判断直到读完整张表
               		3. 执行引擎将筛选出来的满足条件的记录行的记录作为结果集返回给客户端

Q：返回的数据保存在哪？如果大量查询结果？

A：Mysql是一个增量逐步的返回过程，每查到一个结果，mysql就可以开始hi向客户端逐步返回结果集了

## 引擎层

负责数据的存储和提取，其架构模式是插件式的，支持 InnoDB、MyISAM、Memory 等多个存储引擎

![image](https://github.com/whw19970927/Mysql-learning/blob/master/image/image-20200713111903707.png)

在mysql 5.5.5 以后默认存储引擎是innodb



### InnoDB体系结构

![image](https://github.com/whw19970927/Mysql-learning/blob/master/image/image-20200713112607087.png)

* 整体可分为三层：

1. 内存结构，这一层在Mysql服务进程内
2. OS cache， 这一部分属于内核态内存
3. 磁盘结构，这一层在文件系统上

* InnoDB存储引擎由内存池，后台线程和磁盘文件三大部分组成

  

![image](https://github.com/whw19970927/Mysql-learning/blob/master/image/image-20200713113317326.png)

同时之前有提到Mysql采用单进程多线程模型

例如：

**Master Thread：**刷新脏页到磁盘，保证数据一致性（10秒操作与1秒操作）

**IO Thread：**大量使用异步处理写IO请求，包括4类IO Thread

**Purge Thread：**回收已经使用并分配的undo页

**Page Cleaner Thread：**1.2.X版本以上引入，脏页刷新，减轻master的工作，提高性能



### InnoDB管理单位

1. InnoDB从磁盘上读取记录的方式是:

   * 将数据分成果然的页，以页作为磁盘和内存之间交互的基本单位
   * InnoDB的页大小一般为16kb，也就是一次性读取16kb的数据，一次最少从磁盘读16kb的数据到内存，或者一次最少把16kb的数据刷到磁盘中

   

![image](https://github.com/whw19970927/Mysql-learning/blob/master/image/image-20200713114058829.png)

Q：Innodb写16kb的数据时，操作系统只能4kb的写，但突然数据库崩了，只写了4k就出问题了，别的没写成功，应该怎么办？



### InnoDB内存架构

#### 缓冲池(读请求，减少磁盘IO)

![image](https://github.com/whw19970927/Mysql-learning/blob/master/image/image-20200713114300111.png)

*上图的New Sublist和Old sublist，分别保存了最近被放访问过的年轻页，尾部则保存了最近被访问的old page,  这样划分也就使新的子列表中保存更重要的page(热点数据)，旧的列表则保存较少使用的page，也就是被清除的候选page*

InnoDB在内存中主要包含：

* 缓冲池
* Change缓冲区
* 自适应哈希索引
* Log缓冲区

技术要点：

* Page：为了高效读取，缓冲池划分为页结构组织
* LRU：最近最少使用算法，当需求添加新的page的时候，最近最少使用的Page被清楚，同时新的页面被添加到页表的中间部分，但InnoDB里的LRU还进行了优化

为什么要这样设计LRU？

* 在执行像全表扫描这样的操作时，有些新查到的大数据，如果存在了前5/8，会把一些热点数据淘汰，所以会先存放在后3/8的区域，避免淘汰热点数据

* 预读策略：InnoDB认为当前的请求可能之后会读取某些页面，就预先把它们加载到BufferPool中。

  也就是预读策略也会导致预读的数据存入前5/8的缓冲池中，将热点数据淘汰
  

![image](https://github.com/whw19970927/Mysql-learning/blob/master/image/image-20200713120717422.png)

#### Change Buffer（写请求）

对二级索引的数据操作缓存下来，以减少二级索引的随机IO，并达到操作合并的效果.

https://www.jianshu.com/p/09da43bda609
