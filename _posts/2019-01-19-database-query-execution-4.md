---
layout:     post
title:      (db)数据库语句执行（4）
subtitle:   逻辑查询计划
date:       2019-01-19
author:     Wenguangliu
header-img: img/post-bg-unix-linux.jpg
catalog: 	 true
tags:
    - 数据库
    - 语句
    - 关系代数
    - 逻辑查询计划
    - 语法分析树
---

本文主要介绍将语法分析树转成逻辑查询计划的过程，该过程主要有两个步骤。   
- 用关系代数运算符替换语法分析树上的节点和结构；
- 将上一步的运算符转成（优化）另一个最优表达式；
下面将逐一对这两个步骤进行介绍。

### 从语法分析树到关系代数
对于一个包含<Condition>的但不含子查询的<Query>，可以用一个关系代数表达式替换整个成分--选择列表，from列表和条件，表达式从底向上为：
- <FromList>中的关系；
- 选择语句σC, C为<Condition>表达式；
- 投影πL, L为<SelList>的属性列表；

如下图所示：
![avatar](/asserts/execution_plan_sample1.jpeg)

对于<Condition>中包含子查询的语法树，可以采用不带下标σ的***两参数选择***方法，即左子结点为关系R，右结点为关系S每个元组上的条件表达式，两个参数可以是语法树，表达式树或两者的组合。    
对于以上的两参数选择，可以进一步处理，即：  
- 用两阶段选择下的子结点进行替换，如果有重复，则再加上去重运算δ；
- 用一参数选择σC替换两参数选择；
- 给σC一个参数，表示是原关系的积；
对于以上例子，可以得到如下的代数表达式:
![avatar](/asserts/execution_plan_sample2.jpeg)

以上是子查询相关的情况，对于子查询相关时，该过程更为复杂。该过程需要产生一个特定的额外属性的关系，这些关系将与外部定义的属性相比较；然后将子查询到相关属性应用到关系上，并消除不必要的属性和去重。
```
SELECT DISTINCT m1.movieTitle
FROM StarsIn m1
WHERE m1.movieYear - 30 <= (
	SELECT AVG(birthdate)
	FROM StarsIn m2
	WHERE m2.movieTitle = m1.movieTitle AND
		m1.movieTitle = m2.movieYear
);
```
针对以上SQL语句，逻辑运算如下：
![avatar](/asserts/execution_plan_sample3.jpeg);
过程如下：
- 首先是带有两参数的选择；
- 然后是将选择上移，并将两参数选择用子结点替换；
- 最后简化语句，因为选择语句中m1的movieTitle和movieYear与m2的具有相同的值，所以可以进一步简化，即去除m1.

### 改进逻辑查询计划
在转成逻辑查询语句后，从以下几个方面改进逻辑查询计划，进行语句改写：
- 重新分组或排序，例如满足结合律和分配律的运算符，连接顺序；
- 将选择尽可能地下推；
- 尽可能将投影下推；
- 将重复消除移到树中更好的位置；
- 将选择与下面的积进行结合，转成等值连接；
- 对于子树由相同的可结合和可分配的运算符结点所组成的每一个部分，组成单个具有多子女的结点，然后可以用结合律与分配律进行重新分配；


### 总结
以上是将分析树转成逻辑运算的基本过程，后续将介绍与优化相关的一些内容。


更多内容请参阅其他文章：
- [(db)数据库语句执行(1)-物理计划执行](https://wenguang-liu.github.io/2019/01/09/database-query-execution-1/)
- [(db)数据库语句执行(2)-语法分析与预处理](https://wenguang-liu.github.io/2019/01/13/database-query-execution-2/)
- [(db)数据库语句执行(3)-优化查询计划的代数定律](https://wenguang-liu.github.io/2019/01/15/database-query-execution-3/)
- [(db)数据库语句执行(4)-物理查询的代价估计与选择](https://wenguang-liu.github.io/2019/01/21/database-query-execution-5/)
