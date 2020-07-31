## 回顾wal机制

概念：全程是Write-Ahead Logging， 直译就是更新数据之前先写日志。每一次更新操作会写到redo log中，实际更新由后台操作。

**Q：为什么要使用WAL？**
A：磁盘的写操作是随机IO，比较消耗性能，如果每次都更新到磁盘，IO成本、查找成本都很高。 如果每次更新都先写入 redo log 中，那么就成了顺序写操作。 而且对于client端，延迟就降低了。

回顾一下update的过程:
**update T set c=c+1 where ID=2;**

![image](https://img-blog.csdnimg.cn/20200728180612819.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

浅色部分是innodb执行引擎进行操作，深色部分是server层操作
写redo log之前会先从磁盘拿出数据放到内存中，并更新内存
redolog记录了修改的动作，binlog可以被关闭。
这里使用了之前提到的两阶段提交，为了让redolog和binlog保持一致。

当内存页与磁盘数据页不一致的时候，我们称这个内存页为“脏页”。内存数据写到磁盘后，内存和磁盘的数据页内容就一致了，称为“干净页”。
其实之前的第二讲就可以解释为什么会发生mysql抖动的问题，就是抖动的时候很可能redo log刷盘了。

## flush

flush也就是刷盘，在第二章的时候说了，如下图

![image](https://img-blog.csdnimg.cn/20200728233159190.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

在产生写操作的时候，innodb都会产生redolog，redo log的大小是固定的，是一个跟踪写的操作，一边写一边后移，比如可以配置为一组 4 个文件，每个文件的大小是 1GB，那redo log总共就可以记录 4GB 的操作。写满则会落盘到表空间中。发生write时，会记录每个数据页的修改。

write_pos是当前记录的位置，check_point是擦除的位置，两者都是往后推移并且循环的，擦除前要把记录更新到文件，两者之间空着的部分就是用来记录新的操作的，如果前者追上后者，说明没有足够的空间进行操作了，肉色部分已经全部记录满了，此时需要停下来擦除一些记录，并把check point推进。**也就是在write_pos追上check point的时候会发生刷盘的情况，也就是抖一下**

以上是flush的第一种情况，还有别的几种：

* 系统内存不足，需要淘汰一部分数据页，空出内存页给其他数据使用。如果淘汰的正好是“脏页”，那么就需要将脏页写到磁盘。
* 系统空闲的时候，进行刷脏页。
* MySQL 正常关闭的时候，会把内存的脏页都刷到磁盘上。

**Q:上述四种情况对性能的影响？**
A:
第一种情况 redo log 写满了，要刷脏页，出现这个情况，整个系统都不接受更新了，所以这个情况是要尽量避免的。

第二种是内存不够了，需要将脏页写到磁盘中，InnoDB 策略是尽量使用内存，所以这个情况是常态。 InnoDB 用 buffer pool （缓存池）管理内存。

下述情况会明显影响性能：
1.查询导致刷脏页的个数太多，导致查询响应的时间变长。
2.日志写满，写性能跌为0，对敏感业务，是不能接受的。

应对策略：
设置正确的磁盘参数并告知InnoDB
`innodb_io_capacity`
如果刷太慢的话会导致内存脏页过多，redolog写满，所以需要考虑脏页的比例和redolog的flush速度

**脏页比例计算**

![image](https://img-blog.csdnimg.cn/20200728235857464.png)

![image](https://img-blog.csdnimg.cn/20200729000107103.png?x-oss-
process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ZpbmNlX1dhbmcx,size_16,color_FFFFFF,t_70)

也就是如果当前的脏页比例大于脏页比例上限，返回100（全力刷盘），否则返回对应的百分比
当前脏页比例是通过** Innodb_buffer_pool_pages_dirty/Innodb_buffer_pool_pages_total**

在Mysql8.0后可以通过
`SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_pages_%'`
查看脏页比

InnoDB每次写入的日志都有一个序号，当前写入的序号跟checkpoint对应的序号之间的差值，我们假设为N。InnoDB会根据这个N算出一个范围在0到100之间的数字，这个计算公式可以记为F2(N)。F2(N)算法比较复杂，你只要知道N越大，算出来的值越大就好了。

然后，根据上述算得的F1(M)和F2(N)两个值，取其中较大的值记为R，之后引擎就可以按照innodb_io_capacity定义的能力乘以R%来控制刷脏页的速度。

上述的计算流程比较抽象，不容易理解，所以我画了一个简单的流程图。图中的F1、F2就是上面我们通过脏页比例和redo log写入速度算出来的两个值![在这里插入图片描述](https://img-blog.csdnimg.cn/20200729001532634.png)
。所以，无论是你的查询语句在需要内存的时候可能要求淘汰一个脏页，还是由于刷脏页的逻辑会占用IO资源并可能影响到了你的更新语句，都可能是业务端感知到MySQL“抖”了一下的原因。

要尽量避免这种情况，你就要合理地设置innodb_io_capacity的值，并且平时要多关注脏页比例，不要让它经常接近75%。

一旦一个查询请求需要在执行过程中先flush掉一个脏页时，这个查询就可能要比平时慢了。而MySQL中的一个机制，可能让你的查询会更慢：在准备刷一个脏页的时候，如果这个数据页旁边的数据页刚好是脏页，就会把这个“邻居”也带着一起刷掉；而且这个把“邻居”拖下水的逻辑还可以继续蔓延，也就是对于每个邻居数据页，如果跟它相邻的数据页也还是脏页的话，也会被放到一起刷。

在InnoDB中，innodb_flush_neighbors 参数就是用来控制这个行为的，值为1的时候会有上述的“连坐”机制，值为0时表示不找邻居，自己刷自己的。

找“邻居”这个优化在机械硬盘时代是很有意义的，可以减少很多随机IO。机械硬盘的随机IOPS一般只有几百，相同的逻辑操作减少随机IO就意味着系统性能的大幅度提升。

而如果使用的是SSD这类IOPS比较高的设备的话，建议把innodb_flush_neighbors的值设置成0。因为这时候IOPS往往不是瓶颈，而“只刷自己”，就能更快地执行完必要的刷脏页操作，减少SQL语句响应时间。

在MySQL 8.0中，innodb_flush_neighbors参数的默认值已经是0了
