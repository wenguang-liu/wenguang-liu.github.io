---
layout:     post
title:      (db)数据库事务管理（2）
subtitle:   串行化调度与锁模式
date:       2019-02-17
author:     Wenguangliu
header-img: img/post-bg-unix-linux.jpg
catalog: 	 true
tags:
    - 数据库
    - 事务
    - 串行化
    - 调度
    - 锁模式
    - 两阶段封锁
    - 共享锁
    - 排他锁
---

本文主要介绍数据库事务中的另外一个话题，即事务执行的串行化调度与锁模式。为了确保事务执行的串行化，可以有悲观方法和乐观方法。悲观方法主要是通过封锁实现；乐观方法一般可以通过基于时间戳和有效性确认的并发控制策略。   
本文将首先介绍串行化调度，主要介绍串行化，冲突，优先图等概念与知识；然后介绍基于锁的可串行化；在锁模式中，介绍各种锁的类型，包括：共享锁，排他锁，更新锁，增量锁等内容；

## 1.事务串行化调度
前面，我们提到，事务在执行时，如果能够保证事务之间“不耦合”（串行），那么是能够确保状态的一致性和正确性的。但是这样的性能会很低，在实际中，事务之间也是并发执行的，并发就会导致事务的错乱而产生结果与串行执行得到的结果不一样。为了解决这个问题，这里将介绍“可串行化调度”执行的得到的结果与完全串行调度时一样的动作序列。

### 1.1 串行调度与可串行调度
调度是对一个或多个事务的重要动作的一个序列。串行调度是将所有事务的动作按照事务进行归组，然后按照事务顺序一个个执行的方式，即先执行一个事务的所有动作，然后在执行下一个事务的所有动作。

例如，事务T1和T2如下：   
```
T1: R(A, t); t = 2 * t; W(A, t);   
T2: R(A, s); s = s + 10; W(A, s);   
```
那么如果按（T1，T2）顺序执行，则为 R(A, t); t = 2 * t; W(A, t); R(A, s); s = s + 10; W(A, s);   
如果按（T2, T1）顺序执行 ，则为 R(A, s); s = s + 10; W(A, s); R(A, t); t = 2 * t; W(A, t);   
假如A的初始值为5, 那么两个执行顺序得到最终值分别为： 5 * 2 + 10 = 20 和 (5 + 10) * 2 = 30， 因此我们可以看到对于数据库的最终状态与事务顺序有关；

穿行调度是能够确保数据库的一致性。除了串行调度外，实际上还存在其他一些能够确保数据库一致性状态的调度，这些调度称为可串行化调度。    
例如，对于以下事务T1和T2，   
```
T1: R(A, t); t = 2 * t; W(A, t); R(B, t); t = 2 * t; W(B, t);   
T2: R(A, s); s = s + 10; W(A, s); R(B, s); s = s + 10; W(B, s);  
```
假如初始状态A=B=5， 对于以上两个事务，对A和B执行相同的操作，根据一致性需要，A和B最终也需要相等。    
即对于串行事务（T1，T2），最终A = B = 20；对于串行事务（T2，T1），最终A=B=30；
而也存在非穿行事务，能够确保最终状态是一致的，如调度：    
```
R(A, t); t = 2 * t; W(A, t);    事务T1       
R(A, s); s = s + 10; W(A, s);   事务T2     
R(B, t); t = 2 * t; W(B, t);    事务T1  
R(B, s); s = s + 10; W(B, s);   事务T2    
```
最终的状态与调度（T1，T2）一致，这种调度称为可串行化调度。    
但并不是所有调度都是一致的，例如：    
```
R(A, t); t = 2 * t; W(A, t);    事务T1       
R(A, s); s = s + 10; W(A, s);   事务T2     
R(B, s); s = s + 10; W(B, s);   事务T2    
R(B, t); t = 2 * t; W(B, t);    事务T1  
```
最终得到A = 2 * 5 + 10 = 20，而B =(5 + 10) * 2 = 30。   

另外，事务的具体语义对调度也是有影响的，例如将上例中事务T1中的乘于改成加号，那么所有调度上例中非可串行化调度就会是一个可串行调度，但是在实际数据库中，数据库调度时并不会考虑语句的语义。

### 1.2 冲突可串行化与优先图
在事务的动作中，有一些动作之间时冲突的，而有一些是非冲突的。在这里，冲突是指通过改变动作的执行顺序，导致最终的状态不一致。具体来说，以下动作之间是非冲突的：   
- ri(X); rj(Y); 即事务i读取X的值，事务j读取Y的值；
- ri(X); wj(Y); 即事务i读取X的值，事务j写入Y的值；这是因为在两个事务修改的是不同的数据库元素（X != Y）；
- wi(X); rj(Y); 与上类似；
- wi(X); wj(Y); 与上类似，修改的元素不一样；
除了以上这些动作关系外，以下这些动作是冲突，即会影响最终的结果：
- ri(X); wj(Y); 单个事务的动作是不能够被重新排列的；
- wi(X); wj(X); 不同事务对同一个元素的修改，因为如果改变顺序，则会改变数据库元素的最终状态；
- wi(X); rj(X); 不同事务对同一个元素的修改和读取，因为如果改变顺序，则读取的内容是不一样的；
总结以上，可以得到以下结论： 事务之间涉及到同一个数据库元素；并且至少有一个是写入动作。

例如，对于以下两个事务的调度：   
 r1(A); w1(A); r2(A); r1(B); w2(A); w1(B); r2(B); w2(B);    
 通过以下移动可以得到: 
 r2(A); <-> r1(B); => r1(A); w1(A); r1(B); r2(A); w2(A); w1(B); r2(B); w2(B);    
 w2(A); <-> w1(B); => r1(A); w1(A); r1(B); r2(A); w1(B); w2(A); r2(B); w2(B);   
 r2(A); <-> w1(B); => r1(A); w1(A); r1(B); w1(B); r2(A); w2(A); r2(B); w2(B);    
经过以上移动后，得到串行化的一个调度。 在这里，将能够通过交换相邻动作的非冲突交换能将原先调度转成串行化调度，那么称为原始调度为冲突可串行化调度。 

对于一个调度是否为冲突可串行化的调度，可以采用优先图的方法进行判断。在优先图中，对于事务T1, T2， 我们说事务T1优先于T2（T1 < T2），如果事务T1中存在动作A1，事务T2中存在动作A2，并且满足：
- A1在A2前；
- A1和A2都涉及同一个数据库元素；
- A1和A2至少有一个是写入操作；
以上条件也就是不能够交换A1和A2的条件。

根据以上事务的优先关系，我们可以得到优先图，即图中结点为事务，依赖关系为有向边。

在优先图中，如果存在环，那么调度不是冲突可串行化的，如果无环，那么就是冲突可串行化的，而且优先图中的任一拓扑顺序都是冲突等价的一个串行顺序。

例如，对于事务T1和T2的调度：   
r2(A), r1(B), w2(A), w1(B), r2(B), w2(B);   
可以得到 T1 -> T2, 因为对于元素A只有事务T2在修改，对于元素B，先有事务T1的动作都在T2之前，所以存在从事务T1到事务T2的一条有向边。该调度是冲突可串行化调度。   
但是，如果交换w1(B)和r2(B)的顺序，即r2(A), r1(B), w2(A), r2(B), w1(B), w2(B)，对于元素B，事务的动作顺序为 r1，r2，w1，w2，所以存在从事务T1到事务T2的一条边，也存在从事务T2到事务T1的一条边，所以该调度是非冲突可串行化调度。   

## 2. 基于锁的可串行化
当调度器使用锁时，事务在读写数据库元素时还必须进行封锁和解锁的操作。   
锁的使用应该确保事务的正确性，即：动作和锁操作必须按照期望方式发生：
- 事务只有在数据库元素上获取了锁，且未被释放，才能够读写该数据库元素；
- 事务对封锁的数据库元素进行解锁；
此外，还需要确保调度的合法性，即：任何两个事务不能够同时对一个元素进行封锁。    

在实际应用中，常使用两阶段封锁（Two-phase locking）的方式，该方式可以保证一致事务的合法调度时冲突可串行化的。两阶段封锁方式如下：   
  在每一个事务中，所有的封锁请求（第一阶段）均先于所有的解锁请求（第二阶段）。   
两阶段封锁能够确保事务的一致性，但是该方式却存在“死锁”的风险，即让所多个事务一直处于等待另一事务持有的锁。

## 3. 锁模式
在锁模式中，主要有以下几种模式的锁，共享锁，排他锁，更新锁，这几个锁模式是常用的。而增量锁是可有可无的，因为该模式可以通过其他锁模式实现。

### 3.1 共享锁和排他锁
类似于读写锁，对于任一个元素，其上可以有多个共享锁（Sharing lock）和一个排他锁（Exclusive lock），即在任何时候，允许多个事务同时对一个元素进行读取，或者只允许一个事务对一个元素进行写入。但是不允许这两者同时发生。

对于共享锁和排他锁，采用以下的规则：
- 事务一致性：事务要读取元素X，那么该事务必须拥有X的共享锁或排他锁；如果事务要写入元素X，那么该事务必须拥有元素X的排他锁；
- 事务两阶段封锁：封锁要在解锁之前；
- 调度的合法性：一个元素可以被一个事务占有排他锁，或者被多个事务占有共享锁，但是不能够两者同时发生；

### 3.2 更新锁
通过共享锁和排他锁可以实现事务调度的合法性，但是并不是友好的方式。例如，对于占有X共享锁的但是后面需要排他锁的事务T，即在已有一个共享锁的基础上，再申请一个排他锁。这时采用锁升级方式更为友好，即更新锁。   
更新锁只给予事务T读X而不是写X的权限，且只有更新锁能在以后升级为写锁，读锁不能够直接升级为写锁；一旦元素X有了更新锁，那么我们禁止在X上加任何种类的锁。这样做的原因，是防止X上总有其他锁而永远没有机会升级到排他锁。

综合共享锁，排他锁和更新锁，可以得到以下相容性矩阵。   
    S   X   U   
S   Y   N   Y    
X   N   N   N   
U   N   N   N   
上表中，左边表示占有锁的情况，上面的表示占有左边锁其他事务能否占有上面的锁，例如，在占有共享锁S的下，其他事务能够占有共享锁/更新锁，其他所有的均是不允许的。

### 3.3 增量锁
增量锁是用来支持增量操作的锁，即INC(A,c)，表示对元素A增加常量c，这是一个原子操作，READ(A,t), t = t + c, WRITE(A, t)；对于这个原子操作，需要增量锁进行保护，即以下规则：   
- 一致的事务只有在持有增量锁时才能够对元素执行增量动作。但是增量锁并不支持读或写的动作；
- 在一个合法的调度中，任何时候可以有任意多个事务在X上持有增量锁，但是如果某一个事务持有X的增量锁，那么其他事务不能够拥有X上的共享锁或排他锁；   
- 对于j != i，原子增量动作inci(X)动作与rj(X)和wj(X)均冲突，但与incj(X)不冲突；     
满足以上的规则，即可以确保事务的一致性，合法的调度和冲突可串行化。

## 4.总结
本文主要介绍了数据库中串行化相关的概念，包括串行化调度，可串行化调度，冲突可串行化调度，以及用于识别冲突可串行化调度的优先图方法；然后介绍了基于锁的可串行化调度，最后介绍了用于可串行化调度的锁模式，包括共享锁，排他锁，更新锁，增量锁等。

