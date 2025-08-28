---
layout: post
title: "深入理解Java synchronized关键字"
categories: [ Java, Concurrency ]
description: "详细介绍Java中synchronized关键字的使用方法、实现原理以及在多线程环境下的应用"
keywords: [java,synchronized,concurrency,thread,multithreading,线程同步]
---

# 深入理解Java synchronized关键字

> Java中的synchronized关键字是实现线程同步的基础工具，对于编写线程安全的代码至关重要。本文将详细介绍其使用方法和实现原理。

## 简介

在多线程编程中，当多个线程同时访问共享资源时，可能会导致数据不一致的问题。为了解决这个问题，Java提供了synchronized关键字作为最基本的同步机制。它可以确保同一时刻只有一个线程能够执行被synchronized修饰的代码段。

## synchronized的核心特性

使用synchronized可以实现两个重要特性：

1. **互斥性**：同一时刻只有一个线程可以执行被同步的代码
2. **可见性**：线程对共享变量的修改对其他线程立即可见

## synchronized的三种用法

### 1. 同步实例方法

```java
public class Counter {
    private int count = 0;
    
    public synchronized void increment() {
        count++;
    }
    
    public synchronized void decrement() {
        count--;
    }
    
    public synchronized int value() {
        return count;
    }
}
```

当synchronized修饰实例方法时，锁对象是当前实例对象(this)。这意味着同一实例的同步方法在同一时刻只能被一个线程访问。

### 2. 同步静态方法

```java
public class Counter {
    private static int count = 0;
    
    public static synchronized void increment() {
        count++;
    }
    
    public static synchronized int value() {
        return count;
    }
}
```

当synchronized修饰静态方法时，锁对象是当前类的Class对象(如Counter.class)。所有实例的同步静态方法在同一时刻只能被一个线程访问。

### 3. 同步代码块

```java
public class Counter {
    private int count = 0;
    private final Object lock = new Object();
    
    public void increment() {
        synchronized(lock) {
            count++;
        }
    }
    
    public void decrement() {
        synchronized(this) {
            count--;
        }
    }
    
    public int value() {
        synchronized(Counter.class) {
            return count;
        }
    }
}
```

同步代码块提供了更细粒度的控制，可以灵活指定锁对象，缩小同步范围，提高并发性能。

## synchronized的工作原理

### 对象锁与类锁

在Java中，每个对象都有一把锁，称为监视器锁（monitor lock）或内置锁（intrinsic lock）。当一个线程想要执行被synchronized修饰的代码时，它必须先获得相应的锁：

- 对于实例方法，线程需要获取该实例对象的锁
- 对于静态方法，线程需要获取该类的Class对象的锁
- 对于同步代码块，线程需要获取synchronized括号中指定对象的锁

### 锁的获取与释放

当线程进入synchronized代码块时，会自动获取锁；当线程退出synchronized代码块时（无论是正常完成还是抛出异常），会自动释放锁。

### 可重入性

synchronized是可重入的，这意味着一个线程可以多次获取同一个对象的锁。例如，一个同步方法可以调用另一个同步方法，而这两个方法都属于同一个对象。

```java
public class ReentrantExample {
    public synchronized void method1() {
        System.out.println("method1 executed");
        method2(); // 可以直接调用，不会发生死锁
    }
    
    public synchronized void method2() {
        System.out.println("method2 executed");
    }
}
```

## 实际应用场景

### 线程安全的单例模式

```java
public class Singleton {
    private static Singleton instance;
    
    private Singleton() {}
    
    public static synchronized Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

这是线程安全的懒加载单例模式实现，但效率较低，因为每次调用getInstance()方法都需要获取锁。

### 双重检查锁定优化

```java
public class Singleton {
    private static volatile Singleton instance;
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

这种实现方式既保证了线程安全，又提高了性能。

### 生产者-消费者模型

```java
import java.util.LinkedList;
import java.util.Queue;

public class ProducerConsumerExample {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int capacity = 5;
    
    public synchronized void produce(int item) throws InterruptedException {
        while (queue.size() == capacity) {
            // 队列满时等待
            wait();
        }
        queue.add(item);
        System.out.println("Produced: " + item);
        notifyAll(); // 通知消费者
    }
    
    public synchronized int consume() throws InterruptedException {
        while (queue.isEmpty()) {
            // 队列空时等待
            wait();
        }
        int item = queue.poll();
        System.out.println("Consumed: " + item);
        notifyAll(); // 通知生产者
        return item;
    }
}
```

## synchronized的局限性

尽管synchronized关键字使用简单，但它也有一些局限性：

1. **性能问题**：在竞争激烈的情况下，synchronized的性能较差
2. **缺乏灵活性**：无法中断等待获取锁的线程，也无法尝试获取锁
3. **无法设置超时**：线程只能一直等待直到获取锁

## 与ReentrantLock的比较

Java 5引入了java.util.concurrent包，其中的ReentrantLock提供了比synchronized更强大的功能：

| 特性 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 可中断等待 | 否 | 是 |
| 超时获取锁 | 否 | 是 |
| 公平锁 | 否 | 是 |
| 条件变量 | 一个对象只能有一个 | 可以有多个 |

```java
import java.util.concurrent.locks.ReentrantLock;

public class Counter {
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();
    
    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }
}
```

## 使用建议

### 1. 尽量缩小同步范围

```java
// 不推荐：同步范围过大
public synchronized void doSomething() {
    // 一些不需要同步的操作
    prepareData();
    
    // 只有这部分需要同步
    modifySharedData();
    
    // 一些不需要同步的操作
    processData();
}

// 推荐：缩小同步范围
public void doSomething() {
    // 一些不需要同步的操作
    prepareData();
    
    // 只同步必要的部分
    synchronized(this) {
        modifySharedData();
    }
    
    // 一些不需要同步的操作
    processData();
}
```

### 2. 避免在同步代码中执行耗时操作

在同步代码块中应尽量避免执行耗时操作，如文件I/O、网络请求等，这样会降低程序的并发性能。

### 3. 避免死锁

当需要获取多个锁时，应确保所有线程都以相同的顺序获取锁，以避免死锁的发生。

## 结语

synchronized关键字是Java中最基本的线程同步机制，简单易用。通过合理使用synchronized，我们可以有效地解决多线程环境下的数据一致性问题。但在高性能要求的场景下，我们可能需要考虑使用更高级的同步工具，如ReentrantLock等。

掌握synchronized的使用方法和工作原理对于Java开发者来说是基础且重要的，它有助于编写线程安全的代码，也是理解Java并发编程的起点。