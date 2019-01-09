---
layout:     post
title:      (mysql)InnoDBL在线DDL(3)
subtitle:   在线DDL讨论
date:       2019-01-09
author:     Wenguangliu
header-img: img/post-bg-unix-linux.jpg
catalog: 	 true
tags:
    - DDL
    - MySQL
    - Online
    - InnoDB
---


## 1. 在线DDL对磁盘的要求
**1）临时日志文件**   
在执行DDL时候，并发的DML会被阻塞，因此需要日志文件保存这些请求。可调整参数：innodb_sort_buffer_size，innodb_online_alter_log_max_size。如果临时文件超过上限，那么DDL执行将会失败，没有提交的DML也会回滚。     
**2）临时排序文件**   
DDL执行如果需要重建表，那么会将临时的排序文件写到MySQL的临时目录中。
**3）中间表格文件**   
部分DDL在重建表格时候，会创建一个临时表文件（创建索引），这个文件一般会跟表的原文件大小相似。

## 2.在线DDL失败原因   
在线DDL执行发生失败时，一般的原因可以归结于一下几点：     
**1）ALGORITHM语句指定的雨DDL操作的不符合**       
例如，修改字段类型时候，指定ALGORITHM=INPLACE。而修改字段类型是必须要进行复制的，所以会导致失败。      
**2）LOCK语句指定的锁粒度 (SHARED or NONE)比需要的要低**      
**3）在请求表元数据的排他锁时超时**      
因为在DDL时，如果允许并发DML或DQL语句，这些语句需要表元数据的共享锁，如果存在非活跃的事务或者是长事务，那么DDL将可能会没办法占据排他锁而超时，导致DDL执行失败。      
**4）磁盘空间不足**      
在执行DDL创建索引时，会在tmpdir或innodb_tmpdir临时表，如果磁盘不足，那么将会导致DDL执行失败。    
**5）DML日志超过配置项**    
在执行DDL时候，如果允许DML，那么MySQL会将DML写入到临时日志文件中。如果日志大小超过innodb_online_alter_log_max_size配置值，那么将会导致DDL执行失败，错误代码：DB_ONLINE_LOG_TOO_BIG。     
**6）在执行DDL时，DML发生与新DDL数据不相容的修改**     
与执行DDL并发的DML会将修改旧的表格定义，而不是新的。因此，当MySQL尝试将并发DML修改应用到新的schema时，DDL可能会失败。    
例如，并发DML在某一列插入了重复数据，而同时DDL是将该列改为UNIQUE_INDEX；或者在将该字段上创建主键时，插入NULL值。如果这些情况下，DML先提交，那么DDL就必须回滚。    

## 3. 在线DDL的不足   
虽然在线DDL具有明显的优势，但是也存在以下不足：    
1）如果在临时表上创建索引，那么表会COPY    
2）当表上有Constraint（ON...CASCADE或ON...SET NULL）时，DDL不允许LOCK=NONE    
3）在线DDL必须要等所有占有表元数据锁的事务结束，这会让DDL阻塞，且可能因为超时而失败   
4）在线DDL如果修改外键，那么DDL不会等待外键对应表上的事务，即DDL的事务在更新时，占据自身表元数据的排他锁和外键表元数据的共享锁，在执行DDL的最后一个阶段，DDL会请求外键表元数据的排他锁；而如果这时外键表也执行类似的DDL，那么就可能发生死锁    
5）在DDL执行时，DDL的线程会执行online 日志中的并发DML操作，当DML操作执行时，可能会引发duplicate key的错误(ERROR 1062 (23000): Duplicate entry)，该错误是临时的，且会被后面的请求恢复【没看懂...】。   
6）OPTIMIZE TABLE是重建表，更新所有统计信息，并释放聚集索引中不使用的空间，在MySQL5.6.17之前是不支持该操作的。因为数据是按照主键排序的，二级索引的创建也会耗资源。   
7）MySQL5.6之前创建的Table，如果表中包含DATE, DATETIME或TIMESTAMP字段，那么在该表不支持ALGORITHM=INPLACE的重建方式，除非在这之前用过ALGORITHM=COPY进行过重建。    
8）对于大表而言：
 a）没有方式中断在线DDL，且没办法监控在线DDL操作占用的I/O CPU使用情况；
 b）在MySQL 5.7.6之后才可以通过 Performance Schema stage events监控进度；
 c）在线DDL的回滚是昂贵的；
 d）耗时长的DDL可能会导致replication的延迟，即online DDL操作只有在master上执行完，才会apply到slave上。同样地，并发的DML也是只有在master上先执行。


其他请参阅：   
- [InnoDBL在线DDL(1)-基本介绍](https://wenguang-liu.github.io/2019/01/07/InnoDBL在线DDL-1)    
- [InnoDBL在线DDL(2)-在线操作Online DDL Operation](https://wenguang-liu.github.io/2019/01/08/InnoDBL在线DDL-2)    
- [Online DDL operation（MySQL文档）](https://dev.mysql.com/doc/refman/5.6/en/innodb-online-ddl.html)  


