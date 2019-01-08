---
layout:     post
title:      MySQL在线Schema变更(Online DDL)
subtitle:   iOS定时器详解
date:       2019-01-07
author:     Wenguangliu
header-img: img/post-bg-unix-linux.jpg
catalog: 	 true
tags:
    - DDL
    - MySQL
    - Schema
    - 在线
    - Lock
    - 算法
---

本文将简单介绍MySQL InnoDB在5.6后的新特性-Online DDL，在后续文章将会有更详细的介绍。

## 1. 简单介绍
ONLine DDL特性支持了IN-PLACE的表修改和并发DML，有以下好处：
- 提高mysql可用性，在DDL时候，DML没有被阻塞； 
- 提供可选项，通过在DDL上使用LOCK语句允许在性能和并发上取得平衡；
- 相对于表COPY方式，更少的磁盘使用率和IO过载；  

在缺省情况下，InnoDB会优先使用IN-PLACE，尽量少的锁进行DDL部署。但用户仍然可以通过在Alter Table使用ALGORITHM和LOCK语句来控制DDL执行，具体语法如：   
`	ALTER TABLE tbl_name ADD PRIMARY KEY (column), ALGORITHM=INPLACE, LOCK=NONE;   `
ALGORITHM是用表示执行DDL的算法；其中，LOCK语句是用来控制DML/DQL等访问表的并发度，后面会详细介绍。  
## 2. ALGORITHM语句和LOCK语句  

### 2.1 ALGORITHM语句  
对于ALGORITHM语句，InnoDB提供了以下两个方式：  
**1）ALGORITHM=INPLACE**  
  在执行DDL时不进行复制，而是在已有数据进行，可以避免重建表带来的IO和CPU消耗，保证在执行DDL的时候，DB仍有良好的性能和并发。   
**2）ALGORITHM=COPY**  
  复制原始表，在执行DDL的时候，并允许DML的写，但是允许DQL查询。在执行复制时候，需要UNDO/REDO log，使得COPY的性能比INPLACE要低；并且对于COPY操作，需要将数据读入Buffer Pool，这增加了从内存中清除频繁访问的数据。而INPLACE方式相对需要读取较少的数据到Buffer Pool中。  

### 2.2 LOCK语句  
其中，对于LOCK语句，InnoDB提供了以下四种锁力度；   
**1）LOCK=NONE**  
不适用锁，表示在执行DDL时候允许并发的DML；  
**2）LOCK=SHARED**  
使用共享锁，表示在DDL时候允许执行DQL，但是不允许DML；  
**3）LOCK=DEFAULT**  
使用缺省锁，允许尽量多的并发（包括DQL，DML），不写LOCK语句与LOCK=DEFAULT具有相同的效果；  
**4）LOCK=EXCLUSIVE**  
使用排他锁，不允许DQL和DML。一般会在期望该语句能够尽快执行，且DQL和DML不需要的时候；或者为了确保Inno服务是空间的，禁止有DB的访问。  

## 3. 在线DDL执行过程  
在线DDL的执行过程可以分成三个阶段：  
**1）初始化**  
在初始化阶段，InnoDB会根据存储引擎的能力，DDL中操作语句和ALGORITHM/LOCK语句，确定在DDL执行时候允许的并发数。在这个阶段，会占用“共享升级元数据锁”（Shared Upgradeable Metadata Lock），以保护当前表定义不受到其他session的修改。   
**2）执行**   
在该阶段，DDL语句会被预处理和真正执行。这个阶段，根据初始化阶段处理的结果，决定是否“共享升级元数据锁”继续升级到排他锁。如果排他锁需要的，会在预处理阶段占用。  
**3）提交表定义**  
在提交表定义阶段，元数据锁需要升级到排他锁，并将旧的表定义去除，提交新的表定义。一旦完成，将释放排他锁。  
在上述阶段，元数据的锁被其他并发事务占据时，DDL必须要要等待占据元数据锁的全部事务结束。在DDL执行之前或DDL操作过程中，并发事务可能会占据元数据的锁。当有一些长事务或非活跃事务时，DDL操作可能因为等待元数据排他锁超时而导致DDL失败。另外，一个正在等待元数据排他锁的DDL操作也会阻塞后续的事务。   

下面是一个简单例子说明这种情况。  
会话1:  

(```)
	CREATE TABLE t1 (c1 INT) ENGINE=InnoDB;   
	START TRANSACTION;   
	SELECT * FROM t1;   
(```)

创建表t1，并开始一个事务，但不提交。此时，该事务会占据表t1元数据的共享锁；  
会话2:  
`	ALTER TABLE t1 ADD COLUMN x INT, ALGORITHM=INPLACE, LOCK=NONE;`   
执行DDL语句，此语句因为抢占不到表t1的元数据锁而阻塞，一直等待排他锁；  
会话3:   
`   SELECT * FROM t1;  `
执行一个DQL语句，因为会话2在等待表t1的排他锁，所以会话3因为没有办法占据共享锁而阻塞。  




