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
因为选择可以减少关系的大小，因此，只要不影响表达式最终结果的条件下，都尽量地在语法上尽可能地下推。  
在定律上，与选择相关的两条定律是：   
- σC1 AND σC2(R)) = σC1(σC2(R)) = σC2(σC1(R))  
- σC1 OR σC2(R)) = σC1(R) ∪ σC2 (R) [注：只有当R为集合时才实用]   
其中，C1和C2为选择语句。    
例如，σ((a=1 OR a=3) AND b < c)(R)) 可以改写成 σ(a=1 OR a=3)(σ(b < c)(R))，然后将后者的OR又可以改写成 σ(a=1)(σ(b < c)(R)) ∪ σ(a=3)(σ(b < c)(R))，因为一个元组不能够同时满足a=1和a=3，所以不管R是否为集合，该转换总是成立的。另外，也可以寄那个b < c 放在外层。

与选择相关的是对二元运算符进行下推选择：积，并，交，差，连接等。根据下推选择到每个参数是否为必须的，可以有以下定律：   
- 对于并，选择必须下推到两个参数重, σC(R ∪ S) = σC(R) ∪ σC(S);   
- 对于差，选择必须下推到第一个参数，下推到第二参数是可选的；σC(R - S) = σC(R) - S = σC(R) - σC(S)    
- 对于其他运算，只要求下推到其中一个参数;    
    σC(R × S) = σC(R) × S = R × σC(S);    
    σC(R JOIN S) = σC(R) JOIN S = R JOIN σC(S);    
    σC(R JOIN_D S) = σC(R) JOIN_D S = R JOIN_D σC(S);     
    σC(R ∩ S) = σC(R) ∩ S = R ∩ σC(S);    

例如，对于前面的例子： σ((a=1 OR a=3) AND b < c)(R JOIN S)) ，其中R（a，b），S（b，c）；a=1和a=3只能够应用到R上，b < c只能够应用到S上。   
首先，σ(a=1 OR a=3)(σ(b < c)(R JOIN S)) = σ(a=1 OR a=3)((R JOIN σ(b < c)(S)) = σ(a=1 OR a=3)(R) JOIN σ(b < c)(S).

此外，还有一些是特殊情况下的定律：   
- 任何对空关系的选择为空；
- 如果C是总是为真的条件（如：a>100 OR a <= 100 应用到不允许a为NULL的关系上），则σC(S) = S；
- 如果R为空，则 R ∪ S = S

除了以上规则之外，并不是所有表达式都是应该将选择语句尽可能往下推的。当查询包好虚视图时，有时候将选择尽可能上移是有必要的。例如对于以下关系：   
StarIn(title, year, starName)   
Movie(title, year, length, studioName)
并定义View如下：
```
CREATE VIEW MovieOf2018 AS 
	SELECT * FROM Movie
	WHERE year = 2018
```
MovieOf2018的关系式为　σ(year=2018) (Movie)
此时，如果想要查询参与演2018年电影的影星，可以利用View，得到关系式： σ(year=2018) (Movie) JOIN StarIn， 而此时选择语句已经位于Movie的最底层，可以采用上面提到的σC(R × S) = σC(R) × S 的逆运算；得到 σ(year=2018) (Movie) JOIN σ(year=2018)(StarIn)，即相当于将选择语句先往上移，然后在下移。

### 投影的定律
对投影进行下推是有用的，因为可以减少元组的长度，但是并不能像选择下推一下，减少下推元组数量那么有效； 

投影下推真实的原理：   
***可以在表达式树的任何结点引入投影，只要所消除的属性是其上运算符从来不会用到的，并且不在表达式树的结果中。***    
具体说来，可以有下面定律：   
- πL(R JOIN S) = πL( πM(R) JOIN πN(S))， 其中M和N是连接属性和包含L的输入属性；  
- πL(R JOIN_C S) = πL( πM(R) JOIN_C πN(S)) ，与上类似；   
- πL(R × S) = πL( πM(R) × πN(S))；     
例如，对于R(a,b,c)和S(c,d,e)； π a+e->x, b->y(R JOIN S) = π a+e->x, b->y(π a,b,c (R) JOIN π c,e (S)；

此外，也可以在包并之前进行投影，即：  
πL(R ∪ S) = πL(R) ∪ πL(S)

但是投影不能够应用到集合并，集合/包的交或差之下。

### 有关连接与积的定律
前面提过，与连接与积相关的定律可以有：交换律，结合律；除了这两个外，还可以有：   
- R JOIN_C S = σC (R × S)
- R JOIN S = πL(σC(R × S))
其中，σC为R和S具有相同名字的属性进行等值比较，L为包含R和S中等值对中一个属性及其他所有属性的列表。   

### 有关消除重复的定律
运算符δ用于消除重复，具体定律如下：  
- 如果R没有重复，则δ(R) = R;
- δ(R × S) = δ(R) × δ(S);
- δ(R JOIN S) = δ(R) JOIN δ(S);
- δ(R JOIN_C S) = δ(R) JOIN_C δ(S);
- δ( σ(R)) = σ ( δ(R));

### 有关分组与聚集的定律
运算符γ表示分组与聚集，在运算符γ中，通用的规则如下：   
- δ(γL(R)) = γL(R)，即运算符γ可以吞并运算符δ；   
- γL(R) = γL(πM(R)),其中M是至少包含L中所有属性；   
- γL(R) = γL(δM(R)), γL是不受重复影响的【对于MAX和MIN是不受重复影响的，而另一些SUM，COUNT，AVG，如果在计算前去重，一般会得到不同的值]；

### 总结
本文首先介绍了用于查询优化的基本代数定律，然后介绍查询各种操作中的相关定律。  

更多内容请参阅其他文章：   
- [(db)数据库语句执行(1)-物理计划执行](https://wenguang-liu.github.io/2019/01/09/database-query-execution-1/)
- [(db)数据库语句执行(2)-语法分析与预处理](https://wenguang-liu.github.io/2019/01/13/database-query-execution-2/)
