---
layout: post  
title: "手写AQS实现Lock——从线程安全到自定义锁"  
categories: [ java, concurrency, lock, aqs ]  
description: 从线程安全问题出发，逐步实现一个基于 AQS 思想的自定义锁 MyLock，涵盖自旋锁、等待队列、虚假唤醒优化，以及公平/非公平锁的实现。  
keywords: Java, 并发, AQS, Lock, 自旋锁, 队列锁, 线程安全, CAS, park/unpark  
mermaid: false  
sequence: false  
flow: false  
mathjax: false  
mindmap: false  
mindmap2: false  
---



# 手写AQS实现Lock

## 问题引入-不加锁会怎么样

```java
package com.binge.demo;

import java.util.ArrayList;
import java.util.List;

public class TestAQS {
    static int[] data = new int[]{100 * 100};

    public static void main(String[] args) {
        List<Thread> threads = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            Thread thread = new Thread(() -> {
                for (int j = 0; j < 100; j++) {
                    data[0]--;
                }
            });
            threads.add(thread);
        }


        threads.forEach(Thread::start);

        threads.forEach(thread -> {
            try {
                thread.join();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        });
        System.out.println("data: " + data[0]);
        System.out.println("执行完毕，主线程退出");

    }
}
```
打印结果：预期应该是0，但是结果不是，说明没有加锁，存在线程安全问题。
```
data: 55
执行完毕，主线程退出
```

用Java自带的锁，可以解决线程安全问题。
```java
package com.binge.demo;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class TestAQS {
    static int[] data = new int[]{100 * 100};

    static Lock lock = new ReentrantLock();

    public static void main(String[] args) {
        List<Thread> threads = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            Thread thread = new Thread(() -> {
                lock.lock();
                try {
                    for (int j = 0; j < 100; j++) {
                        data[0]--;
                    }
                } catch (Exception e) {
                    throw new RuntimeException(e);
                } finally {
                    lock.unlock();
                }
            });
            threads.add(thread);
        }

        threads.forEach(Thread::start);

        threads.forEach(thread -> {
            try {
                thread.join();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        });
        System.out.println("data: " + data[0]);
        System.out.println("执行完毕，主线程退出");

    }
}

```
预期结果：
```
data: 0
执行完毕，主线程退出
```



接下来，我们使用AQS实现一个简单的锁。用MyLock类来实现。将上文的ReentrantLock改成MyLock进行实现
```java
package com.binge.demo;

public class MyLock {

    void lock() {

    }

    void unlock() {

    }
}

```


## 实现思路
首先，使用CAS操作来实现锁。CAS操作是原子性的，多线程并发调用lock操作，只有一个线程会成功修改flag的值，其他线程会失败。解锁后，其他线程才有可能修改flag的值。
```java
package com.binge.demo;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.atomic.AtomicBoolean;

@Slf4j
public class MyLock {
    AtomicBoolean flag = new AtomicBoolean(false);

    void lock() {
        while (true) {
            boolean result = flag.compareAndSet(false, true);
            if (result) {
                log.info("线程:{} 争抢锁成功", Thread.currentThread().getName());
                return;
            }
            try {
                Thread.sleep(10);
                log.info("线程:{} 争抢锁失败", Thread.currentThread().getName());
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
    }

    void unlock() {
        flag.compareAndSet(true, false);
        log.info("线程:{} 释放锁成功", Thread.currentThread().getName());
    }
}


```

            
存在的问题：一个线程获取锁后，其他线程都在死循环中尝试获取锁，导致CPU资源浪费。
- 使用线程的等待和通知机制来改进:LockSupport.park + LockSupport.unpark。但是我们unpark通知哪个线程呢？可以让等待的线程放到一个队列里面，unpark头节点的线程即可。

我们只能让持有锁的线程释放锁，其他线程无法释放锁：
- 定义一个ownerThread，解锁前进行校验。


```java
package com.binge.demo;

import lombok.Data;
import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicReference;
import java.util.concurrent.locks.LockSupport;

@Slf4j
public class MyLock {
    AtomicBoolean flag = new AtomicBoolean(false);
    Thread ownerThread = null;

    //    头尾节点的指针，使用原子引用类进行包装
    AtomicReference<Node> head = new AtomicReference<>();
    AtomicReference<Node> tail = new AtomicReference<>(head.get());


    void lock() {
        while (true) {
            boolean result = flag.compareAndSet(false, true);
            if (result) {
                log.info("线程:{} 争抢锁成功", Thread.currentThread().getName());
                ownerThread = Thread.currentThread();
                return;
            }
            log.info("线程:{} 争抢锁失败，需要将当前创建的节点，安全的放到队尾", Thread.currentThread().getName());
//            创建当前节点，将其插入队尾，但是需要保证线程安全，所以可以用原子引用类，它可以对引用的对象进行CAS操作
            Node current = new Node();
            current.thread = Thread.currentThread();
//            但是不能确保当前节点是否插入了真正的尾节点，所以需要CAS判断当前节点是否是真正的尾节点，如果是，那就将新创建的节点插入尾节点
            while (true){
//                拿到的尾节点，不一定是此时此刻真正的尾节点，需要循环进行CAS判断
                Node currentTail = tail.get();
//                如果拿到的尾节点是此时此刻的尾节点，那就立刻修改成新创建的尾节点，目的：确保尾节点插入过程的原子性
                if (tail.compareAndSet(currentTail, current)){  //上一步 Node currentTail = tail.get()拿到的currentTail，执行到这一行代码的时候，tail可能变化了（其他线程插入尾节点成功了），需要CAS确保tail没有变化
                    current.pre = currentTail;
                    currentTail.next = current;
                    break;
                }
            }
//            此时已经将当前节点安全的插入了尾节点，需要等待锁拥有者释放锁，才能继续执行
            LockSupport.park();
        }
    }

    void unlock() {
        if (Thread.currentThread() != ownerThread) {
            throw new RuntimeException("当前线程不是锁的拥有者，无法释放锁");
        }
        flag.compareAndSet(true, false);
        log.info("线程:{} 释放锁成功", Thread.currentThread().getName());

    }


    @Data
    class Node {
        Node pre;
        Node next;
        Thread thread;
    }
}

```

虚假唤醒问题：线程在没有显式调用 unpark() 的情况下意外被唤醒。这可能发生在：
- LockSupport.park() 由于系统调度、硬件中断等外部因素，线程被唤醒。
- Condition.await() 也可能发生类似的情况，因此官方推荐用 循环 而不是 if 来检查条件。

考虑一直没有被唤醒的极端情况：
- 加锁方法，真正阻塞自己之前，先走一遍判断并尝试加锁；而不是直接阻塞自己

```java
package com.binge.demo;

import lombok.Data;
import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicReference;
import java.util.concurrent.locks.LockSupport;

@Slf4j
public class MyLock {
    AtomicBoolean flag = new AtomicBoolean(false);
    Thread ownerThread = null;

    // 使用哨兵节点初始化head和tail，避免null问题
    private final AtomicReference<Node> head;
    private final AtomicReference<Node> tail;

    public MyLock() {
        // 创建一个哨兵节点，作为初始头节点
        Node dummy = new Node();
        head = new AtomicReference<>(dummy);
        tail = new AtomicReference<>(dummy);
    }


    void lock() {

        boolean result = flag.compareAndSet(false, true);
        if (result) {
            log.info("线程:{} 争抢锁成功", Thread.currentThread().getName());
            ownerThread = Thread.currentThread();
            return;
        }


//            创建当前节点，将其插入队尾，但是需要保证线程安全，所以可以用原子引用类，它可以对引用的对象进行CAS操作
        Node current = new Node();
        current.thread = Thread.currentThread();
//            但是不能确保当前节点是否插入了真正的尾节点，所以需要CAS判断当前节点是否是真正的尾节点，如果是，那就将新创建的节点插入尾节点
        while (true) {
//                拿到的尾节点，不一定是此时此刻真正的尾节点，需要循环进行CAS判断
            Node currentTail = tail.get();
//                如果拿到的尾节点是此时此刻的尾节点，那就立刻修改成新创建的尾节点，目的：确保尾节点插入过程的原子性
            if (tail.compareAndSet(currentTail, current)) {  //上一步 Node currentTail = tail.get()拿到的currentTail，执行到这一行代码的时候，tail可能变化了（其他线程插入尾节点成功了），需要CAS确保tail没有变化
                current.pre = currentTail;
                currentTail.next = current;
                log.info("当前线程（对应节点）:{} 成功加入链表尾", Thread.currentThread().getName());
                break;
            }
        }
//            此时已经将当前节点安全的插入了尾节点，需要将当前线程阻塞，等待被唤醒
        while (true) {
//            真正阻塞自己之前，可以先走一遍判断，是否能拿到锁，如果能拿到锁，那就不用阻塞了；如果不能拿到，那就阻塞自己；
//            为什么要先判断，因为如果某个持有锁的线程一直没有解锁，那么所有其他竞争的线程都在这里阻塞了，
//            举例：解锁的线程先修改flag，然后unpark，但是unpark前意外崩溃了，导致没有进行唤醒

//                被唤醒后，走下面的判断
//                被唤醒的线程，应该是头节点的下一个节点，因为头节点对应上一个刚刚解锁完的线程
//                而不是被虚假唤醒的其他线程（对应的节点），因为park后可能被意外唤醒，所以需要再次进行检查，看看被唤醒的是不是正确的节点
//                head->A->B-C   被唤醒的应该是A节点，（head节点刚刚解锁完，调用了唤醒方法）但是由于虚假唤醒的可能性，被唤醒的可能是B或者C，所以需要再次检查
//                检查条件：当前节点的前一个节点，等于head节点的下一个节点，那么我就可以CAS进行上锁了
            if (current.pre == head.get() &&
                    flag.compareAndSet(false, true)) { //CAS进行上锁
                log.info("线程:{} 被唤醒后 成功拿到锁", Thread.currentThread().getName());
                ownerThread = Thread.currentThread();
//                    修改head指向最新的头节点 这里不需要CAS操作，因为已经抢锁成功了
                head.set(current);
//                    断掉此前的头节点与当前节点的关联关系，防止内存泄漏
                current.pre.next = null;
                current.pre = null;
                return;
            }
//            先判断一次，后阻塞，被唤醒后，因为循环，所以会再次走判断
            LockSupport.park(); // 阻塞自己，
        }
    }

    void unlock() {
        if (Thread.currentThread() != ownerThread) {
            throw new RuntimeException("当前线程不是锁的拥有者，无法释放锁");
        }
        flag.set(false); // 释放锁，这里不需要CAS，因为只有持有锁的线程才能释放锁
        log.info("线程:{} 释放锁成功", Thread.currentThread().getName());
//        唤醒head的下一个节点, 因为head节点对应当前持有锁的线程，当前线程即将解锁，自然应该唤醒头节点的下一个节点
        Node headNode = head.get();
        Node next = headNode.next;
        if (next != null) {
            log.info("线程:{} 唤醒下一个节点:{}", Thread.currentThread().getName(), next.thread.getName());
            LockSupport.unpark(next.thread);
        }
    }


    @Data
    class Node {
        Node pre;
        Node next;
        Thread thread;
    }
}

```

测试结果：
```
10:57:27.053 [Thread-97] INFO com.binge.demo.MyLock - 当前线程（对应节点）:Thread-97 成功加入链表尾
10:57:27.054 [Thread-83] INFO com.binge.demo.MyLock - 当前线程（对应节点）:Thread-83 成功加入链表尾
10:57:27.058 [Thread-25] INFO com.binge.demo.MyLock - 当前线程（对应节点）:Thread-25 成功加入链表尾
......
......
10:57:27.085 [Thread-86] INFO com.binge.demo.MyLock - 线程:Thread-86 释放锁成功
10:57:27.085 [Thread-86] INFO com.binge.demo.MyLock - 线程:Thread-86 唤醒下一个节点:Thread-43
10:57:27.086 [Thread-43] INFO com.binge.demo.MyLock - 线程:Thread-43 被唤醒后 成功拿到锁
10:57:27.086 [Thread-43] INFO com.binge.demo.MyLock - 线程:Thread-43 释放锁成功
data: 0
执行完毕，主线程退出
```


## 公平与非公平锁
只需要在上锁方法里面，一开始CAS成功修改了Flag，也要加入队列，就变成了公平锁。上面的代码是非公平锁。
公平锁版本：
```java

package com.binge.demo;

import lombok.Data;
import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicReference;
import java.util.concurrent.locks.LockSupport;

@Slf4j
public class MyLock {
    AtomicBoolean flag = new AtomicBoolean(false);
    Thread ownerThread = null;

    // 使用哨兵节点初始化head和tail，避免null问题
    private final AtomicReference<Node> head;
    private final AtomicReference<Node> tail;

    public MyLock() {
        // 创建一个哨兵节点，作为初始头节点
        Node dummy = new Node();
        head = new AtomicReference<>(dummy);
        tail = new AtomicReference<>(dummy);
    }


    void lock() {

//        注释掉这里的CAS操作，就变成公平锁了
//        boolean result = flag.compareAndSet(false, true);
//        if (result) {
//            log.info("线程:{} 争抢锁成功", Thread.currentThread().getName());
//            ownerThread = Thread.currentThread();
//            return;
//        }
        


//            创建当前节点，将其插入队尾，但是需要保证线程安全，所以可以用原子引用类，它可以对引用的对象进行CAS操作
        Node current = new Node();
        current.thread = Thread.currentThread();
//            但是不能确保当前节点是否插入了真正的尾节点，所以需要CAS判断当前节点是否是真正的尾节点，如果是，那就将新创建的节点插入尾节点
        while (true) {
//                拿到的尾节点，不一定是此时此刻真正的尾节点，需要循环进行CAS判断
            Node currentTail = tail.get();
//                如果拿到的尾节点是此时此刻的尾节点，那就立刻修改成新创建的尾节点，目的：确保尾节点插入过程的原子性
            if (tail.compareAndSet(currentTail, current)) {  //上一步 Node currentTail = tail.get()拿到的currentTail，执行到这一行代码的时候，tail可能变化了（其他线程插入尾节点成功了），需要CAS确保tail没有变化
                current.pre = currentTail;
                currentTail.next = current;
                log.info("当前线程（对应节点）:{} 成功加入链表尾", Thread.currentThread().getName());
                break;
            }
        }
//            此时已经将当前节点安全的插入了尾节点，需要将当前线程阻塞，等待被唤醒
        while (true) {
//            真正阻塞自己之前，可以先走一遍判断，是否能拿到锁，如果能拿到锁，那就不用阻塞了；如果不能拿到，那就阻塞自己；
//            为什么要先判断，因为如果某个持有锁的线程一直没有解锁，那么所有其他竞争的线程都在这里阻塞了，
//            举例：解锁的线程先修改flag，然后unpark，但是unpark前意外崩溃了，导致没有进行唤醒

//                被唤醒后，走下面的判断
//                被唤醒的线程，应该是头节点的下一个节点，因为头节点对应上一个刚刚解锁完的线程
//                而不是被虚假唤醒的其他线程（对应的节点），因为park后可能被意外唤醒，所以需要再次进行检查，看看被唤醒的是不是正确的节点
//                head->A->B-C   被唤醒的应该是A节点，（head节点刚刚解锁完，调用了唤醒方法）但是由于虚假唤醒的可能性，被唤醒的可能是B或者C，所以需要再次检查
//                检查条件：当前节点的前一个节点，等于head节点的下一个节点，那么我就可以CAS进行上锁了
            if (current.pre == head.get() &&
                    flag.compareAndSet(false, true)) { //CAS进行上锁
                log.info("线程:{} 被唤醒后 成功拿到锁", Thread.currentThread().getName());
                ownerThread = Thread.currentThread();
//                    修改head指向最新的头节点 这里不需要CAS操作，因为已经抢锁成功了
                head.set(current);
//                    断掉此前的头节点与当前节点的关联关系，防止内存泄漏
                current.pre.next = null;
                current.pre = null;
                return;
            }
//            先判断一次，后阻塞，被唤醒后，因为循环，所以会再次走判断
            LockSupport.park(); // 阻塞自己，
        }
    }

    void unlock() {
        if (Thread.currentThread() != ownerThread) {
            throw new RuntimeException("当前线程不是锁的拥有者，无法释放锁");
        }
        flag.set(false); // 释放锁，这里不需要CAS，因为只有持有锁的线程才能释放锁
        log.info("线程:{} 释放锁成功", Thread.currentThread().getName());
//        唤醒head的下一个节点, 因为head节点对应当前持有锁的线程，当前线程即将解锁，自然应该唤醒头节点的下一个节点
        Node headNode = head.get();
        Node next = headNode.next;
        if (next != null) {
            log.info("线程:{} 唤醒下一个节点:{}", Thread.currentThread().getName(), next.thread.getName());
            LockSupport.unpark(next.thread);
        }
    }


    @Data
    class Node {
        Node pre;
        Node next;
        Thread thread;
    }
}
```