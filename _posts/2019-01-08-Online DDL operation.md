---
layout:     post
title:      在线DDL操作(Online DDL)
subtitle:   InnoDB的在线操作类型
date:       2019-01-08
author:     Wenguangliu
header-img: img/post-bg-unix-linux.jpg
catalog: 	 true
tags:
    - DDL
    - MySQL
    - Online
    - 操作
    - InnoDB
---

本文将介绍Inno存储引擎在线DDL的相关操作在执行时的相关行为和状态，主要包括索引（Index）操作，主键（Primary Key）操作，列（Column）操作，外键（Foreign Key）操作，表（Table）操作。   
在介绍这些操作之前，先介绍在执行DDL时相关概念。   

## 1. 基本概念  

**1）INPLACE/COPY算法**   
在执行DDL时复制表数据，INPLACE表示不会复制表数据，COPY表示会复制表数据，更多相关内容可以参阅前面内容。    
**2）重建表（Rebuild Table）**   
在执行DDL的时是否需要重建表，例如对于InnoDB，因为其是基于主键（聚集索引）组织数据的，所以如果修改主键（如增加一列），则会需要重建表。   
**3）允许并发DML**  
在执行DDL时是否允许并发的DML。例如，在增加一个二级索引时，允许对数据库执行DML等操作。   
**4）只修改元数据**  
在执行DDL时是否只修改表的元数据，而不需要修改数据。例如，对于修改表名，只需要修改该表的元数据，而不需要触及该表的数据。   

## 2. 索引操作

本小节将介绍5.6开始支持的与索引相关Online DDL操作在执行时的行为和状态。  

操作|INPLACE|重建表|允许并发DML|只改元数据|说明
---|:--:|---:|---:|---:|---:
Creating or adding a secondary index|Yes|No|Yes|No|创建的二级索引包含在创建index结束之前提交的数据，不包含未提交的数据，旧版本的数据，被标记为删除但未从旧索引删除的数据
Dropping an index | Yes | No | Yes | Yes | 只有在所有访问该表的事务都结束后才会结束删除索引，即索引的初始状态是表的最近状态。
Renaming an index| Yes | No | Yes | Yes | 只修改元数据而不触及数据
Adding a FULLTEXT index | Yes* | No* | No | No | 如果没有用户定义的FTS_DOC_ID列，则需要重建表；其他的不需要重建表
Adding a SPATIAL index | Yes | No | No | No | 与FULLTEXT相同。 
Changing the index type | Yes | No | Yes | Yes | 修改索引类型，USING BTREE | HASH，与重建索引类似。 

## 3. 主键操作  
本小节介绍与主键相关操作在在线DDL时的行为和状态。   

操作|INPLACE|重建表|允许并发DML|只改元数据|说明
---|:--:|---:|---:|---:|---:
Adding a primary key | Yes* | Yes* | Yes | No | 新增主键，需要重建数据表（根据聚集索引重排），如果有列必须要转成NOT NULL，那么不允许INPLACE
Dropping a primary key | No | Yes | No | No | 在删除主键而不增加新主键时，只有COPY方式才能被使用
Dropping a primary key and adding another | Yes | Yes | Yes | No | 删除主键和增加新的主键

注：   
1）新增主键： 当新建UNIQUE和PK时，MySQL需要做重复值检查，对于PK，还需要检查不包含NULL值。  
使用COPY方式时，MySQL会将NULL转为对应的默认值。为了支持INPLACE，需要设置SQL_MODE（strict_trans_tables或strict_all_tables）。   
2）新建主键的过程如下：从数据从原表t中复制到新的临时表t1（新的主键/聚集索引）中，然后将原表重命名到一个临时表的名字t2，然后将新索引的临时表t1重命名为原表名t，最后将旧表t2删除。   
3）当采用INPLACE时，即使数据仍然需要复制，但性能仍然会比COPY方式要好：   
- INPLACE不需要UNDO/REDO log；  
- 二级索引是预排序的，能够有序地读取；  
- 不需要使用Change Buffer，因为没有二级索引的随机插入；  

## 4. 列操作. 
本小节主要介绍列操作相关的在线DDL行为。    

操作|INPLACE|重建表|允许并发DML|只改元数据|说明
---|:--:|---:|---:|---:|---:
Adding a column | Yes | Yes | Yes* | No | 当增加的是auto-increment的列时，并发DML不允许。因为是行存储，需要重建数据表。最低要求为ALGORITHM=INPLACE和LOCK=SHARED
Dropping a column | Yes | Yes | Yes | No | 需要重组数据
Renaming a column | Yes | No | Yes* | Yes | 允许并发DML，使用数据类型且只修改列名。当重命名被作为外键的列时候，只能够使用INPLACE
Reordering columns | Yes | Yes | Yes | No | 需要重组数据
Setting a column default value | Yes | No | Yes | Yes | 只修改元数据，元数据存在.frm文件中
Changing the column data type | No | Yes | No | No | 只能够使用ALGORITHM=COPY，为了进行数据转换
Dropping the column default value | Yes | No | Yes | Yes | 修改元数据
Changing the auto-increment value | Yes | No | Yes | No* | 修改一个存在内存中的值
Making a column NULL | Yes | Yes* | Yes | No | rebuild数据表
Making a column NOT NULL | Yes* | Yes* | Yes | No | 重建数据表INPLACE，需要设置：STRICT_ALL_TABLES or STRICT_TRANS_TABLES SQL_MODE，如果包含NULL值，将会失败
Modifying the definition of an ENUM or SET column | Yes | No | Yes | Yes | 在增加枚举值时，应该将新值加在最后，这是可以是INPLACE。但是当存储大小变化时，例如在有8个枚举值的新增一个，需要COPY。在枚举中间增加一个值也需要COPY

## 5. 外键操作
本小节简单介绍与外键操作相关的在线DDL执行的行为和状态。   

 操作|INPLACE|重建表|允许并发DML|只改元数据|说明
---|:--:|---:|---:|---:|---:
Adding a foreign key constraint | Yes* |  | Yes | Yes | 当foreign_key_checks被禁用时，可以用INPLACE；否则只能COPY
Dropping a foreign key constraint | Yes |  | Yes | Yes | 同上

## 6. 表操作
本小节简单介绍与表操作相关的在线DDL执行的行为和状态。   

 操作|INPLACE|重建表|允许并发DML|只改元数据|说明
---|:--:|---:|---:|---:|---:
Changing the ROW_FORMAT | Yes | Yes | Yes | No | 修改表的行格式，将会重建数据表
Changing the KEY_BLOCK_SIZE | Yes | Yes | Yes | No | 修改表的键块的大小，将会重建数据表
Setting persistent table statistics | Yes | No | Yes | Yes | 只修改表的元数据
Specifying a character set | Yes | Yes* | No | No | 将会重建表格
Converting a character set | No | Yes | No | No | 将会重建表格
Optimizing a table | Yes* | Yes | Yes | No | 优化表格，使用INPLACE（5.6.17后），但不支持ALGORITHM和LOCK的语法。但是有FULLTEXT索引的表不能够使用INPLACE
Rebuilding with the FORCE option | Yes* | Yes | Yes | No | 使用INPLACE（5.6.17后）。但是有FULLTEXT索引的表不能够使用INPLACE
Performing a null rebuild | Yes* | Yes | Yes | No | 同上
Renaming a table | Yes | No | Yes | Yes | 将会重命名表名，但不COPY

## 7.总结   
本文主要介绍与MySQL（5.6后）中InnoDB中支持Online DDL的操作相关内容。其他更多信息可以参阅：   
[InnoDB引擎的Online DDL](https://wenguang-liu.github.io/2019/01/07/MySQL-Online-DDL/)   
[Online DDL Operations](https://dev.mysql.com/doc/refman/5.6/en/innodb-online-ddl-operations.html)   

