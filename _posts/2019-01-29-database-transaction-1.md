---
layout:     post
title:      (db)数据库事务管理（1）
subtitle:   故障，一致性与日志策略
date:       2019-01-29
author:     Wenguangliu
header-img: img/post-bg-unix-linux.jpg
catalog: 	 true
tags:
    - 数据库
    - 故障
    - 日志
    - 事务
    - Redo日志
    - Undo日志
    - 一致性
---

本文将开始介绍数据库管理系统中的故障以及应对策略。在数据库管理中，最常见的应对策略就是日志，通过Redo，Undo日志来维护数据库事务中出现的错误。   
本文首先将介绍数据库中常见的故障模式，然后介绍Undo日志，Redo日志，Redo/Undo日志，最后介绍应对故障的其他一些策略。

## 故障模式与事务
数据库中出现故障是常有的事情，这些问题大体可以分为：
- 错误数据输入： 用户将错误数据输入到数据库，违背了数据库的一致性。例如电话号码输错一位。一般采用约束和促发器识别这些错误。
- 介质故障： 硬盘的局部故障，例如改变了一位或几位，这种可以通过奇偶校验码进行检测和纠正；如果整个磁盘都损坏了，那么需要RAID，备份等途径进行恢复。
- 灾难性故障：是指外部如火灾，地震等造成的故障。面对这种故障一般采用分布备份方法进行预防和恢复。
- 系统故障：数据库中的事物是一步一步进行的，当进行到一半发生故障（如断电），会造成事物状态的丢失，造成数据的不一致。针对这种错误，则需要日志方法进行修复，也就是本文的重点。

数据库事务是数据库操作执行的单元，为了确保数据库数据的一致性，必须保证数据库事务具有：
- 原子性（Atomic）：即事务要么全部完成，要么像没有执行过一样，最终状态不能够时中间的某一个不合法的状态；
- 一致性（Consistency）：即事务执行前和执行后，均要满足数据库的数据一致性。例如，A向B转账100元，那么A的账户首先至少有100元的余额；在完成转账过程中，A的账户减少了100元，B的账户多了100元，满足全量不变性；
- 隔离性（Isolation）： 事务之间不能够影响，即事务之间的状态是不可见的。隔离性按照隔离粒度不同，可以分为：未提交读（READ UNCOMMITED），即事务能够读到其他事务中未提交的状态；提交读（READ COMMITTED）：基于锁机制的并发控制需要对选定对象的写锁一直保持到事物结束，但是读锁是操作后马上释放，因此可能出现“不可重复读”现象；可重复读（REPEATABLE READS）：即基于锁并发控制需要对选定对象的读锁和写锁均保持到事务结束，但不要求“范围锁”，所以可能出现“幻读”；可串行化（SERIABLE）：最高的隔离级别，在基于锁的并发控制中，除了要求选定对象上的读锁和写锁要保持到事务结束后才释放，并且在SELECT语句中WHERE子句描述一个范围时，应该获得一个“范围锁”，避免幻读。
- 持久性（Durability）：事务提交后，系统的状态时永久的。

因为在执行事务的过程中，可能因为操作自身原因或是外部原因，导致事务中止，从而使数据库处于不一致的状态。为了解决这个不一致，数据库系统需要Undo日志，Redo日志或两者来使数据恢复到一致状态。

首先看一下日志形式：
- <START T>: 事务T开始
- <COMMIT T>: 提交T
- <ABORT T>:事务T执行失败


## Undo日志
Undo日志就是用来撤销还没提交的执行语句产生的效果，以恢复数据库的状态。   
在Undo日志中，还会记录更新记录，即三元组：<T, X, v>，表示事务T对X修改之前的值为v；

### Undo日志规则
为了事务和缓冲区管理器能够根据Undo日志从故障中恢复，那么需要遵循一下两条规则：   
- 事务T修改了数据库元素X的值，那么日志三元组<T，X，v>的日志必须要在X的新值写入磁盘之前写到磁盘；
- 对于事务提交COMMIT日志，它必须要在事务改变的所有数据库元素写入到磁盘之后才写到磁盘中【尽快】；

以上也就是说，对于一个事务，需要按如下顺序写入磁盘：   
1. 指明改变数据库元素的日志记录；
2. 改变的数据库元素自身数据；
3. COMMIT日志记录；

### 使用Undo日志恢复
对于Undo日志而言，保留了完整的记录信息。当故障发生时，可以将Undo日志从头开始扫描，并将事务分成已完成的事务和未完成的事务。   
- 对于已完成的事务【有<START T>和<COMMIT T>的事务】，我们不需要做任何事情，因为根据以上的日志规则，能够保证数据已经落盘；   
- 对于未完成的事务【有 <START T> 但没有 <COMMIT T> 的事务】，我们需要对Undo日志进行相关处理，以恢复数据的一致性；    

对于未完成的事务，采用一下过程进行恢复：将日志从尾部开始扫描日志，在扫描过程中，记录所有有<COMMIT T>或<ABORT T>记录的事务；在扫描过程中，如果发现修改日志记录<T, X, v>，则：
- 如果事务T已有COMMIT记录，则不需要额外的操作，因为事务T已被提交；
- 否则，事务T未完成或中止，则需要将数据库中X的值改为v，以确保数据库的一致性；

### 检查点
对于直接使用Undo日志进行恢复，需要扫描整个完整的Undo日志，为了减少这种扫描的日志长度，可以对日志中加入检查点(CheckPoint)。

在一个检查点中，我们可以：
1. 系统暂停接受新的事务；
2. 等所有活跃的事务COMMIT或者ABORT，并将COMMIT/ABORT日志落盘；
3. 写入日志记录<CKPT>，并刷新日志；
4. 开始接受新的事务请求。

在恢复中，依然是从尾部开始扫描，处理方式同上，直到遇到一个<CKPT>日志，停止扫描。因为<CKPT>之前的所有事务均已经完成，不需要进行恢复。

这种方法的效率很低，如果在加入检查点时，正好有一个长事务，那么就会阻塞其他所有的事务。

为了避免这种情况，我们可以采用 非静态检查点的技术。非晶态检查点的过程包括：
1. 写入<START CKPT(T1,..., Tk)>并刷日志到硬盘，其中T1,...,Tk是当前所有活跃事务的标识符；
2. 等待T1,...，Tk事务完成，待完成后，写入日志记录<END CKPT>并刷日志；

在进行日志恢复的时候，可以分为以下两种情况：
- 先遇到<END CKPT>：那么所有未完成的事务均是在上一个<START CKPT>开始的，那么只需要从后扫描到上一个<START CKPT>便可停止；
- 先遇到<START CKPT(T1,...,Tk)>：那么没有办法确定T1,...,Tk的事务已经完成，而且上一个<START CKPT>开始的事务均可能未完成，因此扫描时需要扫描到上一个<START CKPT>。此外，为了降低扫描成本，我们还可以将同一事务的日志链到一起，方便查找日志。


## Redo日志
相对于Undo日志，Redo日志可以避免将立即将数据元素备份到磁盘中：
- Undo日志是在恢复时消除未完成事务的影响，而Redo日志则是忽略未完成的事务，并重复执行事务所做的改变；
- Undo日志要求在COMMIT日志记录到达磁盘前将修改数据后的元素写到磁盘；而Redo日志要求COMMIT记录在任何修改后的值到达磁盘前出现在磁盘上；
- Undo日志中需要记录修改前的旧指，而Redo日志需要记录修改后的新值；

在Redo日志中，日志记录<T, X, v> 的含义是“事务T为数据库元素X写入新值v”，每当一个事务T修改一个数据库元素X时，必须往日志中写入一条Redo日志。

### Redo日志规则
对于Redo的日志规则，也称为先写日志规则（Write-Ahead-Log）：
R1: 在修改磁盘上的任何数据库元素X以前，要保证与X的这一修改相关的所有日志记录，包括<T, X, v> Redo日志和COMMIT日志，都必须写入到磁盘中。

即对于Redo而言，与一个事务相关的材料写入到磁盘的顺序为：
1. 修改元素的记录Redo日志；
2. COMMIT日志
3. 改变的数据库元素

### 使用Redo日志恢复
对于Redo日志而言，只要日志中没有出现<COMMIT T>的日志，那么事务T的任何修改没有写入到磁盘中；而对于有<COMMIT T>日志的事务，那么我们不能知道数据修改有没有写入到磁盘中，那么这时候需要根据Redo日志进行恢复。在系统发生崩溃时，我们需要做以下步骤：
1. 确定已提交的事务；
2. 从日志首部开始扫描，对于遇到每一条<T,X,v>记录：
   1. 如果T是未提交的事务，则什么也不做；
   2. 如果T是已经提交的事务，那么将v写入到X中，并落盘；
3. 对于未完成的事务T，在日志中写入<ABORT T>的记录；

### 检查点
对于Redo日志的检查点，需要以下过程：
1. 写日志<START CKPT(T1,..., Tk)> ,其中，T1,..., Tk是活跃的事务，即未完成的事务；
2. 将START CKPT记录写入日志时，所有已经提交事务已经写到缓冲区的但未写到磁盘的数据元素写到磁盘；
3. 写入<END CKPT>日志；

对于拥有Checkpoint的日志，在日志恢复时采用以下方案：
- 如果崩溃前日志最后一条是<END CKPT>，对于在对应的<START CKPT(T1,..., Tk)>前提交的事务均已经写入磁盘，但是对于T1,..., Tk，和开始的事务，即使是已经提交<COMMIT>，那么也不能够确定该事务的数据以ing被写入磁盘，因此需要重做这些事务；
- 假如最后一条日志是<START CKPT(T1,..., Tk，那么不能确定在该Checkpoint开始前提交的事务是否已经写入磁盘，因此需要找到上一个<END CKPT>和对应的<START CKPT>，并重做上一个<START CKPT>开始/进行中并提交的事务。

## Undo/Redo日志
对于Undo和Redo日志都有对应的不足，为了弥补这些不足，我们可以使用Undo/Redo日志确保数据一致性。

### Undo/Redo规则
Undo/Redo日志是一个四元组<T, X, v, w>表示对于事务T修改数据元素X的值，修改前值为v，修改后值为w；

对于Undo/Redo日志系统必须遵循以下规则：
对于事务T，所产生的数据元素的改变在写入磁盘之前，必须将日志<T, X, v, w>写入到磁盘中；不对<COMMIT T>日志记录落盘有依赖；

### 使用Undo/Redo日志恢复
当系统发生崩溃时，根据日志信息进行以下恢复：
- 将日志从前往后扫描，重做所有提交的事务；
- 将日志从后扫描，撤销所有未提交的事务；

### 检查点
对于Undo/Redo日志，定期写入检查点的策略如下：
1. 写日志<START CKPT(T1,..., Tk)> ,其中，T1,..., Tk是活跃的事务；
2. 将所有“脏缓冲（有被修改过的）”写入到磁盘；
3. 写入日志记录<END CKPT>

基于Undo/Redo日志的事务在恢复中存在奇怪的行为，即U改之前X的值为v，然后U修改X未v1，然后T修改X为v2，然后T提交，系统崩溃。：
- 如果先重做，事务T提交了并被重做，然而，T读取了一个值X，该值是另一个未提交的事务并被撤销的事务U写入的。
- 如果先撤销，使X具有由T写入的值v2；

这是由于事务之间的隔离机制决定的，将在后续中进行分析与解决。

对于具有检查点的恢复，采用以下策略：
- 如果崩溃前最后一条是<END CKPT>，那么在对应<START CKPT(T1,..., Tk)>前提交的事务均写入磁盘中，对于(T1,..., Tk)和START CKPT开始的事务，则需要重做（COMMIT）或撤销（为提交）。
- 如果崩溃前最后一条是<START CKPT>，那么与Redo日志相似的，需要找到上一个START CKPT的日志，并重做活撤销该日志之后的事务

## 介质故障策略
介质故障是磁盘发生物理故障导致数据丢失，为了避免这个问题，一般采用“备份”的策略。对于备份，我们具有两种不同的备份策略：
- 完全转储，即拷贝整个完整的数据库；
- 增量转储，只拷贝上一次完全转储或增量转储后改变的数据库元素。

### 非静止转储
在实际线上环境中，大多数数据库为了做备份拷贝而停服，因此，我们需要与非静止检查点相似的非静止转储策略。   
在非静止转储过程中，需要建立一个转储开始时数据库的一个拷贝，而在转储过程中的几分钟/几小时中，数据活动可能改变了磁盘上的数据。   
如果需要在备份中恢复数据库时，在转储过程中的日志可以用来弥补数据，使数据库达到一致的状态。   
另外，如果在转储过程中数据发生改变，即事务进行未提交，而之后才提交，那么拷贝到备份数据库的元素可能是也可能不是转储开始的值，但是只要保留这些事务的日志，就可以将备份恢复到一致。

具体说来，建立备份的过程如下：
1. 写入日志<START DUMP>；
2. 根据日志策略（Undo，Redo，Undo/Redo）执行一个合适检查点；
3. 进行完全转储或增量转储，确保数据拷贝已经到达安全的远程结点；
4. 确保足够的日志被拷贝到远程结点（即至少2中检查点之前且包括该检查点的日志在数据库介质故障后仍存在）；
5. 写入<END DUMP>日志；

当介质发生故障时，可以根据以下步骤利用转储数据和日志进行恢复：
1. 根据备份恢复数据库。
   1. 找到最近的完全转储（备份），并将备份拷到数据库；
   2. 如果有增量转储，则从前往后利用增量转储修改数据库；
2. 用保留的日志进行修改数据库，使对应日志策略进行恢复。

## 总结
本文主要介绍了数据事务中出现的不一致，故障类型，然后介绍了用于恢复数据一致性的几种策略，即：Undo日志，Redo日志，Undo/Redo日志。最后介绍了针对介质故障的保护策略--备份。
