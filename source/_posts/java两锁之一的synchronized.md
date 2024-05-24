---
title: java两锁之一的synchronized
date: 2024-05-08 09:02:00
tags:
   - java锁
categories:
   - 后端开发
---

多线程可以提升任务的执行效率，但是多线程由于隔离程度不够，会出现多个线程同时操作同一变量使得变量值不可控的情况，也就是线程安全问题。

**线程安全问题出现的场景**：

+ 多线程环境
+ 有共享数据
+ 有多条语句操作共享数据/单条语句本身非原子操作（比如i++虽然是单条语句，但并非原子操作）

一般来说解决问题就是需要破坏三个条件中的一个，锁就是将多线程访问变量的过程串行化，破坏多线程环境。另外还可以通过创建副本的方法来破坏第二个条件，lua脚本将多个redis操作合并成原子操作破坏第三个条件。

java中锁的实现有两种，分别是基于`Monitor`的`synchronized`和基于`AQS`的`ReentrantLock`，这篇文章先来总结一下synchronized的使用与实现

<!--more-->

# 1. synchronized的使用

## 1.1 经典的单例模式

为了保证类只有一个实例，需要保证只有一个线程能使用Class文件创建类对象，这样就可以对`SingletonPattern.class`加锁，保证资源独享。

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
   public static void main(String[] args) throws InterruptedException {
           Object o = new Object();
           Thread t1 = new Thread(() -> {
               //对对象进行加锁
               synchronized (o) {
                   System.out.println("test1");
               }
           });
           Thread t2 = new Thread(() -> {
               synchronized (o) {
                   System.out.println("test2");
               }
           });
           t1.start();
           t2.start();
           t1.join();
           t2.join();
       }
   ```

   ```class
    0 aload_0
    1 dup
    2 astore_1
    3 monitorenter
    4 getstatic #25 <java/lang/System.out : Ljava/io/PrintStream;>
    7 ldc #39 <test2>
    9 invokevirtual #33 <java/io/PrintStream.println : (Ljava/lang/String;)V>
   12 aload_1
   13 monitorexit
   14 goto 22 (+8)
   17 astore_2
   18 aload_1
   19 monitorexit
   20 aload_2
   21 athrow
   22 return
   ```

   字节码可以看到代码块通过**monitorenter**和**monitorexit**来实现加锁和释放锁

2. 方法

   ```java
   public static void main(String[] args) throws InterruptedException {
           test();
   }
   //对方法进行加锁
   public static synchronized void test() {
           System.out.println("test1");
   }
   ```
   
   ```java
   //观察字节码方法同步是通过设置标志ACC_SYNCHRONIZED来实现的,(方法中没有monitorenter等同步字节码)
   public static synchronized void test();
       descriptor: ()V
       flags: (0x0029) ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED
       Code:
         stack=2, locals=0, args_size=0
            0: getstatic     #12                 // Field java/lang/System.out:Ljava/io/PrintStream;
            3: ldc           #18                 // String test1
            5: invokevirtual #20                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
            8: return
         LineNumberTable:
           line 12: 0
           line 13: 8
   }
   ```

不论是代码块还是方法，synchronized同步都是针对对象进行的。代码块直接是操作对象，方法会根据是否是static方法来判断是对class对象还是实例对象进行加锁（static方法属于类而不是实例对象）。

# 2. synchronized的实现

## 2.1 概述

**特性**：原子性、可见性、可重入性、有序性

synchronized的实现依赖于JVM，java对象在JVM中会按**对象头、实例数据和对齐填充**的形式存储，了解synchronized我们只需要关注对象头就行了。

**对象头结构**：

- **mark-word**：对象标记字段占 4 个字节，用于存储一些列的标记位，比如：哈希值、轻量级锁的标记位，偏向锁标记位、分代年龄等。

- **Klass Pointer**：Class 对象的类型指针，Jdk1.8 默认开启指针压缩后为 4 字节，关闭指针压缩（-XX:-UseCompressedOops）后，长度为 8 字节。其指向的位置是对象对应的 Class 对象（其对应的元数据对象）的内存地址。

mark-word在各锁状态的总览：

<img src="java两锁之一的synchronized\jvm_markWord.png" alt="jvm_markWord" />

markWord使用3位来表示对象的五种状态（无锁，偏向锁，轻量锁，重量锁，以及 GC 标记），线程在竞争锁时会判断对象加锁情况进行竞争。

## 2.2 无锁->偏向锁

无锁状态时MarkWord中会存放**HashCode**、**分代年龄**等信息。当一个线程来请求获取锁时,此时MarkWord前54bit会用来表示持有该锁的线程，Epoch表示该锁的版本戳，当该线程再次来访问时可以直接访问，不需要同步。

JDK15之后偏向锁就被弃用了，因为使用锁的场景大多是多个线程竞争的情况，而偏向锁的优势是**可以节省同一个线程多次请求同一锁时同步消耗的资源**，这样一来多个线程竞争时反而会因为偏向锁向轻量级锁转变造成资源的浪费。

JDK8之前可以设置`-XX:-UseBiasedLocking`来禁用偏向锁，另外JVM延迟设置偏向锁，所以下面测试偏向锁需要sleep()或者`-XX:BiasedLockingStartupDelay=0`关闭延迟偏向锁

```java
public static void main(String[] args)  throws InterruptedException{
    TimeUnit.SECONDS.sleep(5);
        Object o = new Object();
        synchronized (o){
            //打印，观察对象加锁情况
            System.out.println(ClassLayout.parseInstance(o).toPrintable()); 
        }
    }
/*
<!--查看对象头工具-->
      <dependency>
          <groupId>org.openjdk.jol</groupId>
          <artifactId>jol-core</artifactId>
          <version>0.9</version>
          <scope>test</scope>
      </dependency>
*/
```

//可以看到加上参数后，对象加的是轻量级锁（观察开头4个字节最后三位，顺序是倒过来的，所以最后8位是`c8`，最后两位是00）

<img src="java两锁之一的synchronized\image-20240513145730228.png" alt="image-20240513145730228" />

//不加参数的情况，最后三位是101，加的是偏向锁

<img src="java两锁之一的synchronized\image-20240513150908609.png" alt="image-20240513150908609" />

## 2.3 偏向锁->轻量级锁

多个线程竞争同一把锁且竞争不太激烈时，偏向锁会升级为轻量级锁（CAS自旋来获取），虚拟机会在当前线程的栈帧中创建一个LockRecord空间，储存当前锁对象的MarkWord拷贝（主要是为记录了HashCode，分代年龄等内容）。

线程竞争锁时会先将MarkWord复制到栈帧中，之后**通过CAS尝试将锁对象的LockRecord指针指向栈帧中的LockRecord**，并将owner指针指向锁对象的MarkWord，CAS操作成功后将锁对象MarkWord锁字段更新为00，表示轻量级锁。CAS失败后会检查MarkWord中LockRecord指针是否指向当前线程的栈帧，如果是则表明已抢到锁，否则自旋等待。

**概述**：轻量级锁抢占需要维持对象和线程的双向联系，**锁对象的LockRecord需要指向栈帧中的LockRecord**，**栈帧中的owner指针需要指向锁对象的MarkWord**。只有同时满足这两个联系，才算成功加锁。

### LockRecord和owner分别是什么？

LockRecord在openjdk中通过以下两个类来实现

```c++
// A BasicObjectLock associates a specific Java object with a BasicLock.
// It is currently embedded in an interpreter frame(在栈的解释帧上分配).
class BasicObjectLock {
  friend class VMStructs;
 private:
  BasicLock _lock; // 锁
  oop       _obj; // 持有锁的对象，owner的实现
};


class BasicLock {
 private:
  volatile markOop _displaced_header;
};
```

当字节码解释器执行monitorenter字节码轻量地锁住一个对象时，就会在获取锁的线程的栈上显式或隐式分配一个lock record。

栈帧中解释帧包含一个区域，该区域保存激活拥有的所有监视器的锁记录。在解释的方法执行期间，该区域根据持有的锁数量增长或缩小。lock record在线程的Interpretered Frame（解释帧）上分配。

其实关于LockRecord只需要知道，**在轻量级锁时JVM会在栈帧中创建一个对象（对象中有着owner指针）来进行线程与加锁对象的双向关联**

## 2.4 轻量级锁->重量级锁

当CAS自旋达到一定次数会变成重量级锁，这时线程会进入ObjectMonitor的阻塞队列中，当锁被释放时会随机从队列中唤醒一个进程持有锁。持有锁的线程执行Object.wait()方法阻塞会转移到**WaitSet**队列，等待被notify()或notifyAll()唤醒后会进入**EntryList**中。

**ObjectMonitor结构**：

```java
ObjectMonitor() {
	_header = NULL;
	_count = 0; // 记录个数
	_waiters = 0,
	_recursions = 0; // 线程重入次数
	_object = NULL; // 存储 Monitor 对象
	_owner = NULL; // 持有当前线程的 owner
	_WaitSet = NULL; // 处于 wait 状态的线程，会被加入到 _WaitSet
	_WaitSetLock = 0 ;
	_Responsible = NULL ;
	_succ = NULL ;
	_cxq = NULL ; // 单向列表
	FreeNext = NULL ;
	_EntryList = NULL ; // 处于等待锁 block 状态的线程，会被加入到该列表
	_SpinFreq = 0 ;
	_SpinClock = 0 ;
	OwnerIsThread = 0 ;
	_previous_owner_tid = 0;
}
```

每个对象都会关联一个ObjectMonitor，java通过ObjectMonitor来管理锁（只要有synchronized就离不开ObjectMonitor）。

<img src="java两锁之一的synchronized\java重量级锁.webp" alt="java重量级锁" style="zoom:80%;" />

**重量级锁的升级条件**：

1. 从轻量级锁升级为重量级锁的条件： 自旋超过一定次数（默认10次），可以通过`-XX:PreBlockSpin`设置次数
2. 从无锁/偏向锁直接升级为重量级锁的条件：**调用了object.wait()方法，则会直接升级为重量级锁！**





