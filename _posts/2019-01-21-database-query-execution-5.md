---
layout:     post
title:      (db)数据库语句执行（5）
subtitle:   物理查询的代价估计与选择
date:       2019-01-21
author:     Wenguangliu
header-img: img/post-bg-unix-linux.jpg
catalog: 	 true
tags:
    - 数据库
    - 物理查询
    - 关系代数
    - 代价估计
    - 逻辑计算
---

在计算出逻辑计划并优化后，我们将派生出多个不同的物理执行计划。这些不同的物理计划会产生不同的执行代价，因此，一般在执行之前会进行代价估计，然后执行引擎将选择代价最小的物理计划。根据代价，将确定从逻辑计划到物理计划的选择，包括：   
- 对于可结合与可分配的运算的次序与分组，如连接，并，交；    
- 逻辑计划中每一个运算符的算法，如，使用嵌套循环链接或散列连接或索引连接；   
- 其他运算符，如扫描，排序等；   
- 参数从一个运算符传送到下一个运算符的方式，如将中间结果保存到硬盘还是在内存缓冲区；  
本文首先将介绍执行计划的估计方法，然后将介绍基于代价的选择。

## 执行计划的估计
本章节将介绍执行计划中各种运算符的估计方法。   

### 投影运算大小的估计
通常，投影时元组会消除成分，是减小关系的；对于投影产生新成分的，可能会增加关系。但是都不增加元组的数量。

### 选择运算大小的估计
对于S = σA=c(R)，估算T(S) = T(R)/V(R,A)，即T的元组数量除于T的A列的不同值的数量；   
对于S = σa< c(R)，估算T(S) = T(R)/3；   
对于S = σ C1 or C2(R)，如果R有n个元组，满足C1和C2的元组分别为m1和m2，则假定S的个数为： n(1-(1-m1/n)(1-m2/n))。  

### 连接运算大小的估计
对于连接 R(X,Y) JOIN S(Y,Z)：   
- 如果两个关系有不相交的Y值，那么连接后元组个数为0；   
- 如果Y是S的键，且为R的外键，那么连接后元组个数与R相同；   
- 如果S和R都具有相同的Y值，那么连接后的元组个数为T(R)T(S)；

在实际中，一般用下面公司计算连接的元组个数：   
T(R JOIN S) = T(R)T(S)/max(V(R,Y),V(S,Y))

#### 多连接属性的自然连接估计
对于多连接属性，与上面类似，采用下面方法进行估算：  
即通过计算T(R)T(S)，然后除了Y中所有公共属性y中，V(R,y)和V(S,y)的较大值；

#### 多个关系的连接
对于多个连接关系的连接，S=R1 JOIN R2 JOIN R3 JOIN R4 ... JOIN RN；   
那么可以通过以下方式进行计算：   
首先计算所有关系元组数量的积，然后对于每个出现过两次的属性A，除于除了V(R,A)最小值意外的其他值；     

### 其他运算大小的估计
对于其他的运算，则可以利用下面方式进行计算：   
1) 并   
采用两数之和的一半；   

2) 交    
较小者的一半；

3) 差   
对于R-S，取T(R)-T(S)/2;

4) 消除重复  
取T(R)/2与所有属性V(R, a)之积较小的一个；

5) 分组与聚集   
取T(R)/2与所有属性V(R, a)之积较小的一个；

## 执行计划的选择
在以上的基础上，本章节将介绍基于以上估计的计划选择方法，首先介绍估计值的一些方法。

### 估计值的获取与记录
对于用于估计的统计量，可以采用定期进行计算。这是因为，这些统计量一半不会突变；且如果进行实时统计，那么会影响性能，浪费计算资源。

常用于记录每个属性统计量可以采用***直方图***进行记录。常见的直方图有：
- 等宽，即选定宽度，高度表示在这个范围内出现的次数；
- 等高，公共的百分点，即比最小值多p，2p...等对应的属性值；
- 最频值，记录最经常出现的值和频数，对于其他不经常出现的，可以用总和进行记录；

根据这些统计量，可以更加精确地计算估计值。

### 减少逻辑查询计划代价
启发式逻辑优化是利用启发式（如下推选择，去重下推）对逻辑算子进行优化，然后对优化后的树进行代价估计，如果代价有优化，那么则进行对应的优化。

### 物理计划估算方法
在将逻辑计划转成物理计划时存在多种组合方法。最简单的方法就是***穷尽法***，即穷尽所有的物理计划组合，找到代价最小的物理计划。   
除此之外，还存在其他求解物理计划的方法，大致上可以分成两类：   
- 自顶向下：从逻辑树的树根开始向下进行，计算参数的所有可能组合的代价，选择最优的一个；   
- 自底向上：对于每一次字表达式，计算子表达式的所有方法的代价，并按所有可能方式与父节点运算符相结合。

### 自底向上的选择方法
这里介绍将逻辑计划转成物理计划的自底向上的算法。  

#### 启发式选择
定义一些启发式的选择规则，例如：   
- 如果逻辑计划中选择A=c属性，且关系在A上有索引，则执行一个索引查找；   
- 如果逻辑计划中有选择A=c与其他条件，可以先执行一个索引查找，然后用filter进行过滤；  
- 如果连接的一个参数在连接属性上有索引，那么采用索引连接；   
- 如果一个参数时排序的，那么采用排序连接比散列连接好；   
- 当计算三个/多个关系的并或交时，先执行最小关系；

#### 分支界定计划枚举
类似于决策树中的剪枝法。令一个物理计划的代价为C，然后考虑这个子查询的的其他计划时，然后去除那些代价大于C的子查询计划。如果构造出代价小于C的完整计划，则用该计划代替原有计划。该方法可以判定何时中断搜索，即已经搜索的到的物理计划已经超过了C，那么就终止搜索。   


#### 爬山法
根据启发式选定的物理计划开始，然后不断搜索。即通过对物理计划的简单修改，如果找到更优的，则替换原有的计划。也可以通过结合律和交换律对连接进行排序，找到相邻的计划。

#### 动态规划
对于每一个子表达式，只保留最小代价的计划，不断向上进行归约。

#### Selinger风格优化
对动态规划进行修改，即每一个子表达式不仅保留最优的计划，还保留较优代价且可能对上层表达式有用的计划。这类物理计划的结果按以下属性进行排序：
- 在上层结点排序运算符中说明的属性；
- 分组运算符的分组属性；
- 连接运算的连接属性；


## 总结
本文主要介绍了不同的运算方式的代价估计，基于这些估计将选择正在的物理执行计划，介绍了计划的选择的方法。


更多内容可以参考