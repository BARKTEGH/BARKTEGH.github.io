---
title: InnoDB存储引擎（一）- InnoDB体系结构
date: 2019-08-02 20:06:20
tags:
- MySQL
categories:
- MySQL
---

### InnoDB体系结构

![](https://i.imgur.com/uMvsQ0Z.png)

### 后台线程

### 内存



## InnoDB关键特性

InnoDB存储引擎关键特性：

* 插入缓冲
* 两次写
* 自适应哈希索引
* 异步IO
* 刷新邻接页

### 插入缓冲


#### Insert Buffer





### 两次写


![](https://i.imgur.com/LpnbgZh.png)

doublewrite由两部分组成，一部分是内存中的doublewrite buffer，大小为2MB，另一部分是物理磁盘上共享表空间中连续128个页，大小同样为2MB。





> Mysql技术内幕（InnoDB存储引擎）


