---
title: ReentrantLock解析
date: 2024-10-27 19:57:10
tags:
   - java锁
   - juc
categories:
   - 后端开发
---

多线程可以提升任务的执行效率，但是多线程由于隔离程度不够，会出现多个线程同时操作同一变量使得变量值不可控的情况，也就是线程安全问题。

**线程安全问题出现的场景**：

+ 多线程环境
+ 有共享数据
+ 有多条语句操作共享数据/单条语句本身非原子操作（比如i++虽然是单条语句，但并非原子操作）

一般来说解决问题就是需要破坏三个条件中的一个，锁就是将多线程访问变量的过程串行化，破坏多线程环境。另外还可以通过创建副本的方法来破坏第二个条件，lua脚本将多个redis操作合并成原子操作破坏第三个条件。

java中锁的实现大体分两种，分别是基于`Monitor`的`synchronized`和基于`AQS`的`ReentrantLock`，这篇文章来总结一下ReentrantLock的使用与实现。

<!--more-->

# 1. ReentrantLock的使用

```java
public static void main(String[] args) {
        ReentrantLock lock = new ReentrantLock();
        Thread thread1 = new Thread(()->{
            //尝试获取锁
            if (lock.tryLock()) {
                //获取锁成功则执行
                lock.unlock();
            } else {
                // 无法获取锁时的处理逻辑
            }
        });
        Thread thread2 = new Thread(()->{
            //阻塞获取锁
            lock.lock();
        });
    }
```

# 2. ReentrantLock实现原理

```java
/*
	实现lock接口
*/
public class ReentrantLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = 7373984872572414699L;
    /** Synchronizer providing all implementation mechanics */
    private final Sync sync;
    
}
/**
*	lock接口相当于是锁的实现规范，Redisson分布式锁也实现该接口
**/
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
/*
*	Condition是实现类似与Sychronized中的wait和notify机制的规范，实现依赖于LockSupport
*/
public interface Condition {
    void await() throws InterruptedException;
    void awaitUninterruptibly();
    long awaitNanos(long nanosTimeout) throws InterruptedException;
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    boolean awaitUntil(Date deadline) throws InterruptedException;
    void signal();
    void signalAll();
}
```

##  2.1 基础知识

```java
//加锁通过调用sync对象的lock()方法实现 
public void lock() {
        sync.lock();
    }
//Sync内部对象是AbstractQueuedSynchronizer的实现
abstract static class Sync extends AbstractQueuedSynchronizer {
    ...
}
```

先来看看**AbstractQueuedSynchronizer**

### 2.1.1 AQS

```java
//抽象队列同步器，是ReentrantLock实现的核心
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
  
    
    static final int WAITING   = 1;          
    static final int CANCELLED = 0x80000000; 
    static final int COND      = 2;          

    /** CLH 公平队列，用来封装任务 */
    abstract static class Node {
        volatile Node prev;       
        volatile Node next;       
        Thread waiter;            
        volatile int status;      

        
        final boolean casPrev(Node c, Node v) {  
            return U.weakCompareAndSetReference(this, PREV, c, v);
        }
        final boolean casNext(Node c, Node v) { 
            return U.weakCompareAndSetReference(this, NEXT, c, v);
        }
        final int getAndUnsetStatus(int v) {     
            return U.getAndBitwiseAndInt(this, STATUS, ~v);
        }
        final void setPrevRelaxed(Node p) {      
            U.putReference(this, PREV, p);
        }
        final void setStatusRelaxed(int s) {     
            U.putInt(this, STATUS, s);
        }
        final void clearStatus() {              
            U.putIntOpaque(this, STATUS, 0);
        }

        //获得Node对应变量的偏移量
        private static final long STATUS
            = U.objectFieldOffset(Node.class, "status");
        private static final long NEXT
            = U.objectFieldOffset(Node.class, "next");
        private static final long PREV
            = U.objectFieldOffset(Node.class, "prev");
    }

   
    static final class ExclusiveNode extends Node { }
    static final class SharedNode extends Node { }

    static final class ConditionNode extends Node
        implements ForkJoinPool.ManagedBlocker {
        ConditionNode nextWaiter;            
        
        public final boolean isReleasable() {
            return status <= 1 || Thread.currentThread().isInterrupted();
        }

        public final boolean block() {
            while (!isReleasable()) LockSupport.park();
            return true;
        }
    }

    private transient volatile Node head;

    private transient volatile Node tail;

    //状态
    private volatile int state;
    
     private static final Unsafe U = Unsafe.getUnsafe();
    //获得AQS对应变量的内存偏移
    private static final long STATE
        = U.objectFieldOffset(AbstractQueuedSynchronizer.class, "state");
    private static final long HEAD
        = U.objectFieldOffset(AbstractQueuedSynchronizer.class, "head");
    private static final long TAIL
        = U.objectFieldOffset(AbstractQueuedSynchronizer.class, "tail");

    //jvm静态代码块或变量在<clinit>方法中初始化，该代码主要是为了显示加载LockSupport的class文件，防止后续调用LockSupport中静态方法时加载影响效率。
    static {
        Class<?> ensureLoaded = LockSupport.class;
    }
}
```

AQS实现基于LockSupport与Unsafe类

### 2.1.2 UnSafe

UnSafe是java提供的用来直接操作内存的对象。操作内存显然是不安全的，所以叫UnSafe，名字起得真有艺术感。

```java
public final class Unsafe {
    private Unsafe() {}
    private static final Unsafe theUnsafe = new Unsafe();
    public static Unsafe getUnsafe() {
        return theUnsafe;
    }
      @IntrinsicCandidate
    public final native boolean compareAndSetReference(Object o, long offset,
                                                       Object expected,
                                                       Object x);
    @IntrinsicCandidate
    public final native int compareAndExchangeInt(Object o, long offset,
                                                  int expected,
                                                  int x);
    
                                                   @IntrinsicCandidate
    public final boolean compareAndSetByte(Object o, long offset,
                                           byte expected,
                                           byte x) {
        return compareAndExchangeByte(o, offset, expected, x) == expected;
    }int x);
    ...
}
```

如上UnSafe类提供了一系列的CAS方法，简单看下参数。

- `Object`表示属性存在的对象
- `offset`表示属性在内存中的偏移量（可以通过api获取）
- `expected`表示修改后的期望值
- `x`表示修改前的值。
- 返回值为boolean类型表示CAS修改是否成功。

### 2.1.3 LockSupport

```java
public class LockSupport {
    //私有构造函数，对外都是静态方法，和上面提前加载class文件相对应
    private LockSupport() {} 
    private static void setBlocker(Thread t, Object arg) {
        U.putReferenceOpaque(t, PARKBLOCKER, arg);
    }
    //作用和方法名一样: 阻塞线程获取许可证，有许可证则直接通行
    public static void park() {
        U.park(false, 0L);
    }
   //重载方法, blocker表示阻塞该线程的对象或者原因，用于调试不影响主体功能。
    public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        U.park(false, 0L);
        setBlocker(t, null);
    }
    //给某个线程发许可证
     public static void unpark(Thread thread) {
        if (thread != null)
            U.unpark(thread);
    }
}
```

LockSupport主要是通过`park()`和`unpark()`实现阻塞和唤醒的过程，`park()`方法提供了许多重载，包括设置阻塞器、阻塞时间等，底层调用的UnSafe类的native()方法。

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        MyThread myThread = new MyThread(Thread.currentThread());
        myThread.start();
        try {
            // 主线程睡眠3s
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("before park");
        // 获取许可
        LockSupport.park("ParkAndUnparkDemo");
        System.out.println("after park");
    }
}
class MyThread extends Thread {
    private Object object;

    public MyThread(Object object) {
        this.object = object;
    }

    public void run() {
        System.out.println("before unpark");
        // 释放许可
        LockSupport.unpark((Thread) object);
        System.out.println("after unpark");
    }
}
```

### 2.2.4 LockSupport与wait/notify对比

LockSupport与Object中wait()和notify()区别

1. **实现依赖不同**：Object中wait()和notify()基于JVM Monitor机制，LockSupport基于UnSafe类的CAS

2. **功能不同**

   - LockSupport更加灵活，可以先unpark()再park()，不过许可最多只有一个。Object是等待唤醒机制，只能先wait再notify()。

   - Object中wait()会释放资源并加入队列，LockSupport中park()只会一直阻塞（释放锁资源交给Condition await实现）

### 2.2.5 CLH公平队列







## 2.2 加锁、释放锁流程



