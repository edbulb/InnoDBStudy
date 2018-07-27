# InnoDB存储引擎
***
- 第一个支持ACID事务的MySQL引擎
- 行级锁设计
- 支持MVCC
- 支持外键
- 提供非一致性非锁定读锁
- 最有效使用和以及内存和CPU

## InnoDB体系结构 
- 后台线程  
  InnoDB是支持多线程的模型，后台有多个不同的线程。
  * Master Thread  
   负责将缓冲池中的数据刷新到磁盘，保证数据的一致性，包括脏页的刷新，合并插入缓冲，UNDO页的回收。
  * IO Thread  
   InnoDB中大量采用AIO方式来处理IO请求，这样可以大大提高数据库的性能。在IO Thread中有四种线程其中包括read、write、log、insert buffer 四个IO Thread，其中read和write有四个，log和insert各只有一个。在命令: *show engine innodb status*可以显示： ![img](https://github.com/edbulb/InnoDBStudy/tree/master/img/iothread.png)
  * Purge Thread  
   事务在被提交后其undolog已经不在需要，因此由purge thread来回收undo页，在InnoDB1.1前是有master thread来回收的，在INnoDb1.1后可以通过 *innodb_purge_thread = 1 * 来实现在线程回收的方式减轻主线程的的工作。
   在InnoDB1.2后用户可以使用4个purge thread来实现undo的回收。
  * page cleaner thread  
   是InnoDB1.2之后将之前在master thread中做的脏页刷新放在线程中做，减轻元master thread的负担。

- 内存  