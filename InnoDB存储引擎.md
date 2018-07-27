# InnoDB存储引擎
----
>- 第一个支持ACID事务的MySQL引擎
- 行级锁设计
- 支持MVCC
- 支持外键
- 提供非一致性非锁定读锁
- 最有效使用和以及内存和CPU

## InnoDB体系结构 
>- 后台线程  
  InnoDB是支持多线程的模型，后台有多个不同的线程。
  * Master Thread
    
  ---
>- 内存  
![img](https://github.com/edbulb/InnoDBStudy/blob/master/img/iothread.png)