---
layout:     post
title:      数据库查询查询
subtitle:   物理查询计划
date:       2019-01-09
author:     Wenguangliu
header-img: img/post-bg-unix-linux.jpg
catalog: 	 true
tags:
    - 数据库
    - 查询
    - 执行
    - 执行优化
    - 物理计划
---

数据库的语句执行主要步骤：   
- 词法分析/语法分析，建立查询的分析树；   
- 查询重写，分析树被转化为初始查询计划（查询的代数表达式），然后将查询计划转化为一个预期执行时间较小的等价计划；   
- 物理计划生成，将逻辑查询计划的每一个操作符选择实现算法并选择这些操作符的执行顺序，逻辑计划被转化为物理查询计划。

本文将简单介绍 物理查询计划的执行。