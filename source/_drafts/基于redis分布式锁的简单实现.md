---
title: 基于redis分布式锁的简单实现
date: 2024-04-22 21:34:21
tags: 
  - redis
  - 分布式锁
categories:
  - redis
      
---

# 1. 分布式锁的核心

锁主要是用来解决线程安全问题

引发线程安全问题的三个条件：

+ 多线程环境
+ 有共享数据
+ 有多条语句操作共享数据/单条语句本身非原子操作

在单机情况下，使用jvm支持的synchronized或者ReentrantLock就行，但是在多机JVM内存不一致的时候解决线程安全问题就需要分布式锁了（使用一块公共地方来进行数据访问的统一）

# 2. 初步实现



# 3. Redisson的封装
