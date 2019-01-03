---
layout:     post
title:      分布式哈希表（Distributed Hash Table，DHT）
subtitle:   算法视角(In Algorithm Perspective)
date:       2018-12-29
author:     Wenguangliu
header-img: img/post-bg-ios9-web.jpg
catalog: 	 true
tags:
    - 分布式
    - 哈希表(Hashtable)
    - 算法
    - 路由
    - 组通信
    - DHT 
---

目录
1. 基础知识 3
2. 环的原子管理算法 3
2.1 环管理中存在的问题 3
2.2 环管理并发控制算法 5
2.2.1 环管理锁算法 5
2.2.2 环原子管理的算法 7
3. 路由算法 10
3.1 深度递归算法 10
3.2 广度迭代算法 11
3.3 传递算法 12
3.4 贪心算法 13
4. 组通信算法 14
4.1 广播算法 14
4.2 批量操作算法 16
4.3 其他 17
5. 副本管理算法 17
5.1 对称备份策略 17
5.1.1 节点管理算法 18
5.1.2 数据管理算法 19
5.2 多哈希备份策略 20
5.3 后续列表和叶集合 20
6. 分布式哈希表的应用 21
6.1 底层存储 21
6.2 主机发现和移动 22
6.3 网页缓存 22
6.4 其他应用场景 23
7 总结 23


分布式哈希表，即提供Key/Value的存储服务。最基础的两个接口就是PUT(Key,Value)和GET(Key)，即将数据保存到DHT和从DHT中读取某个特定的Key对应的Value值。
分布式哈希表必须具备以下特性：动态可拓展，数据具被分散性，系统自管理。简单地说就是能够动态扩展节点，每个节点上的数据尽量分散，系统能够自动容错等。

更多请参考
1. 分布式哈希表-算法角度描述 (分布式哈希表-DistributedHashTable.pdf)
2. 分布式哈希表-PPT (DistributedHashTable-PPT.pdf)


