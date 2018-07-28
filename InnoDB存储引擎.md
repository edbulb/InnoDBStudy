# InnoDB存储引擎
- 第一个支持ACID事务的MySQL引擎
- 行级锁设计
- 支持MVCC
- 支持外键
- 提供非一致性非锁定读锁
- 最有效使用和以及内存和CPU

## InnoDB体系结构 
- **后台线程**  
  InnoDB是支持多线程的模型，后台有多个不同的线程。
  * Master Thread  
   负责将缓冲池中的数据刷新到磁盘，保证数据的一致性，包括脏页的刷新，合并插入缓冲，UNDO页的回收。
  * IO Thread  
   InnoDB中大量采用AIO方式来处理IO请求，这样可以大大提高数据库的性能。在IO Thread中有四种线程其中包括read、write、log、insert buffer 四个IO Thread，其中read和write有四个，log和insert各只有一个。在命令: *show engine innodb status*可以显示：  ![img](https://raw.githubusercontent.com/edbulb/InnoDBStudy/master/img/iothread.png)
  * Purge Thread  
   事务在被提交后其undolog已经不在需要，因此由purge thread来回收undo页，在InnoDB1.1前是有master thread来回收的，在INnoDb1.1后可以通过 *innodb_purge_thread = 1 * 来实现在线程回收的方式减轻主线程的的工作。
   在InnoDB1.2后用户可以使用4个purge thread来实现undo的回收。
  * page cleaner thread  
   是InnoDB1.2之后将之前在master thread中做的脏页刷新放在线程中做，减轻元master thread的负担。
- **内存**  
    * 缓冲池  
   InnoDB是基于磁盘存储的，但是磁盘和CPU的速度有很大的差距，所以在实际工作中基于磁盘的数据库系统一般使用缓冲池的技术来实现提高数据库整体性能。  
   缓冲池其实来说就是一块内存区域，通过内存的速度来弥补磁盘速度。在读取页中首先把目标放入缓冲池中，，下一次读的时候直接先在缓冲池中寻找。对于数据的修改也是先在缓冲池中修改，然后再于一定的频率刷新到磁盘中。页发生修改的时候后并不是每一次修改后都发生，而是通过Checkpoint的机制刷新会磁盘。CheckPoint也是为提高数据库性能。
   由上可知，缓冲区域越大对于数据库的性能提高越有帮助，在32位系统中系统最多将缓冲池的值设置为3G，我们可以在系统的PEA选项将值获取32位下最大的63G支持。在InnoDB核心中建议数据库采用64位。
   获取缓冲尺的容量大小： * show vaiables like \' innodb_buffer_pool_size \' *:   ![img](https://raw.githubusercontent.com/edbulb/InnoDBStudy/master/img/innodn_buffer_pool_size.png)
   具体在缓冲池中有：索引页、数据页，[undo页](https://blog.csdn.net/alexdamiao/article/details/51872477)、插入缓冲、自适应哈希索引、InnoDB存储所信息、数据字典信息等。在其中索引页和数据页占了很大一部分。如图：  ![img](https://raw.githubusercontent.com/edbulb/InnoDBStudy/master/img/huanchongchi.png)   
   InnoDB允许多个缓冲池的存在，每个页根据不同的哈希值分配到不同的缓冲池中，这样可以减少数据库的内部支援竞争。  
   使用 *show variables like innodb_buffer_pool_instancs*可以查看当前缓冲池的个数。（当前使用的是mysql5.6）如图：   
   ![img](https://raw.githubusercontent.com/edbulb/InnoDBStudy/master/img/innodb_buffer_pool_instances.png)
    * LRU List、Free List和Flush List  
   InnoDB存储引擎对内存的管理，通常innodb的缓冲池页为16k。
   缓冲池使用（last recent used）最近最少使用，方法来管理数据。最多使用的页放在LRU前列，最不频繁使用的放在LRU后端，当缓冲池不能存放新读取的页的时候把最靠后的页释放。
   在InnoDB中最新访问的页不会一开始就放在LRU的最前列，而是放在midpoint的位置上使用 *show variables like 'innodb_old_bloacks_pct' *可以查询出 midpoint的位置：    ![img](https://raw.githubusercontent.com/edbulb/InnoDBStudy/master/img/midpoint.png)
   其中37为活跃热点
   为什么不把新读取的页放入到RLU的最前端：因为在其它的一些SQL操作中的页会被刷新，而这些页通常只会使用一次，这样会大大降低缓冲池的工作效率。（这些SQl操作通常是：索引或者数据的扫描）。
   （midpoint之后的列表成为old列表，之前的列表成为new列表：热端）
   innodb_old_blocks_time这个值就是用于表存在多久会被放入RLU的热端（前端）innodb_old_blocks_time是可以设置的：（SQL） * set global innodb_old_blocks_time = 1000 *   
   当页从old转变成new的时候成为 page make young 当old页因为innodb_old_blocks_time的设置没有从old转到new的时候成为page not make young，使用 show engine innodb status;时可以看到上面的数据。Buffer pool hit rate表示缓冲池命中的概率，一般是95%以上，如果没有的话应该检查是否由于全表扫描造成。在innodb1.2后可以通过查看innodb_buffer_pool_status来查看缓冲池的状态。

   
