---
layout:     post
title:      (db)数据库语句执行（2）
subtitle:   语法分析与预处理
date:       2019-01-13
author:     Wenguangliu
header-img: img/post-bg-unix-linux.jpg
catalog: 	 true
tags:
    - 数据库
    - 语句
    - 执行
    - 语法分析
    - 预处理
---

本文将介绍数据库查询编译器中的语法分析和预处理，该步骤是查询的第一个步骤，即将语句进行语法分析得到语法分析树，并进行相关的预处理。   

## 语法分析
语法分析的工作就是接收类似SQL的语言语句文本，并将之转换成语法分析树的数据结构。    

### 语法分析树基本概念
语法分析树的节点可以分成两种：   
***1）原子（叶）节点***   
语法成本（如关键字），关系或属性的名字，常数，括号，运算符，以及其他模式成分。   

***2）语法类节点***   
在一个查询中起相似作用的查询子成分所形成的族的名称。例如，\<Query\>表示select from where形式的查询；\<Condition\>是指在where之后的表达式；   
其子节点通常是该语言的语法***规则***之一进行描述。   

### 一个小例子的规则定义
如select查询语句的一个子集规则：  
***1) 查询***     
这条简单的语句只是支持以下的规则，不支持GROUP BY，HAVING等。    
```
<Query> := SELECT <SelList> FROM <FromList> WHERE <Condition>    
```
- \<SelList\>：为选择列表；    
- \<FromList\>：为数据源的列表（From列表）；    
- \<Condition\>：为数据过滤的条件；

***2）选择列表***    
下面是选择列表的规则，表示选择列表可以是一个属性加上逗号和另一个选择列表；或者就是一个属性。       
```
<SelList> := <Attribute>, <SelList>  
<SelList> := <Attribute>
```

***3）From列表***   
From列表与选择列表相似，表示可以有单个关系，或者一个关系，逗号和另一个From列表组成。     
```
<FromList> := <Relation>,  <FromList>    
<FromList> := <Relation>    
```

***4）条件***
条件的推倒语法如下，即可以是一个条件，AND和另一个条件；单个属性，IN和一个Query；属性，=，属性；属性， Like， Pattern四个推倒语法。   
```
<Condition> := <Condition> AND <Condition>  
<Condition> := <Attribute> IN (Query)  
<Condition> := <Attribute> = <Attribute>  
<Condition> := <Attribute> LIKE <Pattern>    
```
***5）基本语法类***   
如上面出现的\<Attribute\>, \<Relation\>, \<Pattern\> 不是通过规则定义的，而是通过代表的原子规则定义的。  

### 一个例子的语法分析树
根据以上定义的规则，定义以下两个关系：    
```
StarsIn(movieTitle, movieYear, startName);
MovieStar(name, address, gender, birthdate);
```
对于以下需求：查找出生于1980年影星拍摄的电影名，可以有以下两条语句：
```
SELECT movieTitle FROM StarsIn WHERE starName IN (SELECT name FROM MovieStar WHERE birthdate LIKE '1960%')
```
和
```
SELECT movieTitle FROM StarsIn, MovieStar WHERE starName = name AND birthdate LIKE '1960%'
```

这条Query对应的语法分析树分别如下（左/右）：    
![avatar](/asserts/query-analysis-tree-sample.jpeg)


## 预处理
预处理是将语法分析树作进一步处理的过程，主要包括以下几点：   
1）如果查询语句中用到的关系实际上是一个视图View，那么用到该关系的地方必须用该视图的语法树来代替；   
2）检查关系的使用，即From出现的关系必须是当前模式中的关系或视图；   
3）检查与解析属性的使用，在SELECT或WHERE子句中提到的每个属性必须是当前范围中某一个关系的属性，如果在语句中没有显式地附加到属性上，典型的查询处理器此时通过给属性加上它所引用关系的信息来解析每一属性；   
4）检查类型，所有属性的类型比较与其使用相适应。   

本文主要介绍了语法分析和语句的预处理，更多内容请参阅其他文章：   
- [(db)数据库语句执行(1)-物理计划执行](https://wenguang-liu.github.io/2019/01/09/database-query-execution-1/)
- [(db)数据库语句执行(3)-优化查询计划的代数定律](https://wenguang-liu.github.io/2019/01/15/database-query-execution-3/)
