---
title: ThreadLocal小结
date: 2024-04-30 14:26:21
tags: 
  - juc
categories:	
  - 后端开发
---



# ThreadLocal

ThreadLocal主要是为了解决多线程间信息隔离的问题（创建副本，用空间换时间）。

```java
//泛型为需要共享的数据的类型
public class ThreadLocal<T> {
    ...
}
```

<!-- more -->

## 简单示例

经典的数据源连接

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class DataSourceManager {
    private static ThreadLocal<Connection> connectionHolder = ThreadLocal.withInitial(() -> {
        try {
            // 创建数据库连接
            return DriverManager.getConnection("jdbc:mysql://localhost:3306/mydatabase", "username", "password");
        } catch (SQLException e) {
            e.printStackTrace();
            return null;
        }
    });

    // 获取当前线程的数据库连接
    public static Connection getConnection() {
        return connectionHolder.get();
    }

    // 关闭当前线程的数据库连接
    public static void closeConnection() {
        Connection connection = connectionHolder.get();
        if (connection != null) {
            try {
                connection.close();
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
        connectionHolder.remove();
    }

    public static void main(String[] args) {
        // 在多个线程中使用数据源连接服务
        Thread thread1 = new Thread(() -> {
            Connection connection = DataSourceManager.getConnection();
            // 执行数据库操作
            DataSourceManager.closeConnection();
        });

        Thread thread2 = new Thread(() -> {
            Connection connection = DataSourceManager.getConnection();
            // 执行数据库操作
            DataSourceManager.closeConnection();
        });

        thread1.start();
        thread2.start();
    }
}
```

**综述**：

- 不进行信息隔离？那一个线程在进行操作时，另一个线程关闭连接怎么办？
- 不创建副本？每次操作数据源都加锁？
- 手动创建副本？有现成的ThreadLocal不用，ThreadLocal还使用弱引用在线程结束时自动释放副本。

## ThreadLocal初始化

```java
//使用构造函数（对你没有看错，我也没有省略，构造函数是空的）
public ThreadLocal() {
}
//使用withInitial()，new有初值的ThreadLocal对象(Supplier返回值作为初值)
public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
        return new SuppliedThreadLocal<>(supplier);
}
```

## ThreadLocal方法

```java
//常用的set()设置值和get()获取值,以及使用结束后remove()清除副本
//new 创建ThreadLocal get之前需要先set，不然会抛空指针异常
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
	public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            map.set(this, value);
        } else {
            createMap(t, value);
        }
	}
 	public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null) {
             m.remove(this);
         }
     }

```

## ThreadLocal原理初探

每个Thread对象里面都有ThreadLocalMap对象（有两个，感兴趣自行查阅），Thread通过ThreadLocalMap来存储每个线程的ThreadLocal对象副本

```java
public class Thread implements Runnable {
    ...
	/* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

    /*
     * InheritableThreadLocal values pertaining to this thread. This map is
     * maintained by the InheritableThreadLocal class.
     */
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
 	...   
}
```

### ThreadLocalMap对象

```java
//ThreadLocalMap里面的对象及属性
static class ThreadLocalMap {

        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

        /**
         * The initial capacity -- MUST be a power of two.
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;

        /**
         * The number of entries in the table.
         */
        private int size = 0;

        /**
         * The next size value at which to resize.
         */
        private int threshold; // Default to 0
     ...
 }
```

可以看到ThreadLocalMap里面有一个Entry类型的数组，Entry则是内部一个静态的键值对的类。（Map的常见实现方式，数组存储键值对）

### Entry类

键值对的**key**是**ThreadLocal对象实例**，**value**是**ThreadLocal泛型数据实例**。所以ThreadLocal要隔离的数据其实是在使用时存放在线程中的，ThreadLocal主要是充当模版的作用。（有点类和对象的感觉）

可能有人注意到Entry类继承了`WeakReference<ThreadLocal<?>>`，这是什么东西？

（具体省略，感兴趣自行查阅）

四种引用中的弱引用。

```java
//强引用和弱引用的对比
static class Entry {
    //作为属性来引用ThreadLocal对象，强引用
    ThreadLocal<?> k;
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        value = v;
    }
}
//弱引用
 static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

#### ThreadLocal引用关系

使用ThreadLocal时，Thread会持有ThreadLocalMap强引用，ThreadLocalMap会持有Entry强引用，Entry会持有ThreadLocal弱引用。

在ThreadLocal不使用时（线程依旧持有ThreadLocalMap），Entry中ThreadLocal对象的**key**会被回收掉，不过由于**value**是强引用，所以会出现**key为null，value有值的无效Entry**。虽然ThreadLocal在`get()`时会检查null值的key然后删除（具体查阅`getEntry()`中`getAfterMisss()`方法），不过如果不显示的调用remove()清除ThreadLocalMap，value的生命会与Thread线程实例绑定。

#### ThreadLocal内存泄漏

原本就算value与Thread生命绑定，在Thread示例销毁时value也会销毁。

但是线程池为了资源的复用，里面的核心线程会一直跑着循环来判断是否有新任务，这就会造成value永远无法回收（也就是内存泄漏的问题），泄漏内存为**线程池核心线程数 × value对象大小**

###  ThreaLocalMap的散列方式

ThreadLocal中map使用的是斐波那契散列法，详细见//todo









