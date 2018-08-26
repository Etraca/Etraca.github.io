---
layout:     post
title:      JVM的运行时数据分布
subtitle:   JVM的运行时数据分布
date:       2018-08-26
author:     WPF
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - jvm
    - java
---

## 概述
jvm在执行Java程序的过程中会把它管理的内存划分成几个不同的数据区域，这些数据区域有着各自的用途和生命周期。如图：

![](https://images0.cnblogs.com/i/288799/201405/281726404166686.jpg)



