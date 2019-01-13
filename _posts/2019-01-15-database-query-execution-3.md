---
layout:     post
title:      (db)数据库语句执行（3）
subtitle:   优化查询计划的代数定律
date:       2019-01-15
author:     Wenguangliu
header-img: img/post-bg-unix-linux.jpg
catalog: 	 true
tags:
    - 数据库
    - 语句
    - 执行
    - 查询计划
    - 代数定律
---

本文主要介绍在查询中能够用于查询计划优化的代数定律，主要包括将介绍基本定律；选择条件的定律；投影的定律；有关连接与积的定律；有关消除重复的定律；有关分组与聚集的定律。    

### 交换律与结合律
关系代数的多个运算同时满足交换律与结合律。   
- S×R = R×S；（S×R）×T = S×（R×T）；
- S JOIN R = R JOIN S; (S JOIN R) JOIN T = R JOIN (S JOIN T);
- S UNION R = R UNION S; (S UNION R) UNION T = R UNION (S UNION T); 
- S INTERSECT R = R INTERSECT S; (S INTERSECT R) INTERSECT T = R INTERSECT (S INTERSECT T); 

另外，S JOIN_c R = R JOIN_c S;   
其中，JOIN_c表示带条件的JOIN；  

需要注意的是，有一些定律对于包与集合的效果是不一样的。例如：A INTERSECT (B UNION C) = (A INTERSECT B) UNION (A INTERSECT C); 该定律对于集合是成立的，而对于包却是不成立的。

### 选择条件的定律


### 投影的定律


### 有关连接与积的定律


### 有关消除重复的定律


### 有关分组与聚集的定律


### 总结
本文首先介绍了用于查询优化的基本代数定律，然后介绍查询各种操作中的相关定律。  

更多内容请参阅其他文章：   
- [(db)数据库语句执行(1)-物理计划执行](https://wenguang-liu.github.io/2019/01/09/database-query-execution-1/)
- [(db)数据库语句执行(2)-语法分析与预处理](https://wenguang-liu.github.io/2019/01/13/database-query-execution-2/)
