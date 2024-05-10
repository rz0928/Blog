---
title: java两大锁之一的synchronized
date: 2024-05-08 09:02:00
tags:
   - java锁
categories:
   - 后端开发
---

多线程可以提升任务的执行效率，但是多线程由于隔离程度不够，会出现多个线程同时操作同一变量的问题，也就是线程安全问题。

**线程安全问题出现的场景**：

+ 多线程环境
+ 有共享数据
+ 有多条语句操作共享数据/单条语句本身非原子操作（比如i++虽然是单条语句，但并非原子操作）

这时就需要锁来保证同一个变量在某一刻只能有一个线程来使用，java中锁的实现有两种，分别是基于`Monitor`的`synchronized`和基于`AQS`的`ReentrantLock`

这篇文章先来总结一下synchronized的使用与实现

<!--more-->

# 1. synchronized的使用

## 1.1 经典的单例模式

为了保证类只有一个实例，需要保证只有一个线程能使用Class文件创建类，这样就可以对`SingletonPattern.class`加锁，保证资源独享。

```java
public class SingletonPattern {
    //volatile 用于禁止JVM指令重排
    public static volatile SingletonPattern INSTANCE;
    public SingletonPattern getINSTANCE() {
        if(INSTANCE == null){
            synchronized (SingletonPattern.class){
                if(INSTANCE == null){
                    //具体的初始化逻辑
                    INSTANCE = new SingletonPattern();
                    try{
                        Thread.sleep(350L);
                    }catch (InterruptedException e){
                        e.printStackTrace();
                    }
                }
            }
        }
        return INSTANCE;
    }
}
```

## 1.2 用法

synchronized用法一般可以分为两大类，对方法进行同步和对代码块进行同步。

1. 代码块

   ```java
   ```

   

2. 方法

   ```java
   ```

   

不论是代码块还是方法，synchronized同步都是针对对象进行的（class也可看作对象）。

# 2. synchronized的实现

