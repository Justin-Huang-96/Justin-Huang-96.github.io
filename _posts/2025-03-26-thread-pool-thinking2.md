---
layout: post
title: "简单手写一个线程池"
categories: [ java, 线程池 ]
description: "一步步手写线程池"
keywords: [线程池]
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

# 手写线程池

## 单线程线程池版本
```java

package com.example;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class MyThreadPool {

    BlockingQueue<Runnable> blockingQueue = new ArrayBlockingQueue<>(1024);

    /**
     * 1. 线程何时创建
     * 2. 线程的runnable是什么？
     *
     * @param command
     */
    public void execute(Runnable command) {
//        new Thread(command).start();   //得管理
//        add和offer区别 ：阻塞队列满了的时候，add会抛出异常，offer不会
//        offer会返回是否成功添加了元素
//        blockingQueue.add(command);
        boolean offer = blockingQueue.offer(command);
    }


    Thread thread = new Thread(() -> {

        while (true) {
            //凭空消耗CPU资源，需要采用阻塞队列，当阻塞队列为空的情况下，阻塞线程
//            if (commandList.size() > 0) {
//                Runnable command = commandList.remove(0);
//                command.run();
//            }
            try {
                //当阻塞队列为空的情况下，阻塞线程 不会CPU空转
                Runnable take = blockingQueue.take();
                take.run();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }

    }, "MyThreadPool-唯一Thread");

    {
        thread.start();
    }
}

```

测试demo

```java

//测试demo
package com.example;

import lombok.extern.slf4j.Slf4j;

@Slf4j
public class TestPool {
    public static void main(String[] args) {
        MyThreadPool myThreadPool = new MyThreadPool();
        for (int i = 0; i < 5; i++) {

            myThreadPool.execute(() -> {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.info("threadName:{}", Thread.currentThread().getName());
            });
        }

        log.info("主线程没有被阻塞");
    }
}

```


## 仅考虑核心线程和额外线程的线程池

```java
package com.example;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class MyThreadPool {

    public static final int corePoolSize = 5;
    public static final int maxPoolSize = 10;
    BlockingQueue<Runnable> blockingQueue = new ArrayBlockingQueue<>(1024);
    //线程池应该有多少个线程合适？用corePoolSize变量表示
    List<Thread> coreThreadList = new ArrayList<>();
    //    阻塞队列满了，需要启动额外线程来处理任务
    List<Thread> extraThreadList = new ArrayList<>();


    /**
     * 1. 线程何时创建
     * 2. 线程的runnable是什么？
     * 3. 为什么要加锁，因为判断大小和添加到线程List，两个操作不是原子的
     *
     * @param command
     */
    public synchronized void execute(Runnable command) {
        //线程池中线程数量小于corePoolSize，创建线程
        if (coreThreadList.size() < corePoolSize) {
            //每个线程绑定一个task，task就是循环从阻塞队列获取任务，执行任务的这么一个逻辑
            Thread thread = new Thread(task);
            coreThreadList.add(thread);
            thread.start();//启动这个逻辑
        }
        //任务添加到阻塞队列
        boolean offerResult = blockingQueue.offer(command);
        if (offerResult) {
            return;
        }

        //阻塞队列满了，怎么处理？使用额外线程
        if (extraThreadList.size() + coreThreadList.size() < maxPoolSize) {
            Thread thread = new Thread(task);
            extraThreadList.add(thread);
            thread.start();
            return;
        }
        //如果额外线程也满了？先抛异常
        throw new RuntimeException("阻塞队列和额外线程也满了");

    }

    private final Runnable task = () -> {
        while (true) {
            try {
                //当阻塞队列为空的情况下，阻塞线程 不会CPU空转
                Runnable take = blockingQueue.take();
                take.run();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }

    };

}


```

代码优化版本：

```java
package com.example;

import lombok.extern.slf4j.Slf4j;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;

@Slf4j
public class MyThreadPool {

    public int corePoolSize = 5;
    public int maxPoolSize = 10;
    public int TIMEOUT = 1;
    public TimeUnit TIME_UNIT = TimeUnit.SECONDS;
    BlockingQueue<Runnable> blockingQueue = new ArrayBlockingQueue<>(1024);

    public MyThreadPool(int corePoolSize, int maxPoolSize, int timeout, TimeUnit timeUnit, BlockingQueue<Runnable> blockingQueue) {
        this.corePoolSize = corePoolSize;
        this.maxPoolSize = maxPoolSize;
        this.TIMEOUT = timeout;
        this.TIME_UNIT = timeUnit;
        this.blockingQueue = blockingQueue;
    }

    public MyThreadPool() {
    }

    //线程池应该有多少个线程合适？用corePoolSize变量表示
    List<Thread> coreThreadList = new ArrayList<>();
    //    阻塞队列满了，需要启动额外线程来处理任务
    List<Thread> extraThreadList = new ArrayList<>();


    /**
     * 1. 线程何时创建
     * 2. 线程的runnable是什么？
     * 3. 为什么要加锁，因为判断大小和添加到线程List，两个操作不是原子的
     *
     * @param command
     */
    public synchronized void execute(Runnable command) {
        //线程池中线程数量小于corePoolSize，创建线程
        if (coreThreadList.size() < corePoolSize) {
            //每个线程绑定一个task，task就是循环从阻塞队列获取任务，执行任务的这么一个逻辑
            Thread thread = new CoreThread();
            coreThreadList.add(thread);
            thread.start();//启动这个逻辑
        }
        //任务添加到阻塞队列
        boolean offerResult = blockingQueue.offer(command);
        if (offerResult) {
            //添加任务成功
            return;
        }
        //此时添加任务失败
        //阻塞队列满了，怎么处理？使用额外线程
        if (extraThreadList.size() + coreThreadList.size() < maxPoolSize) {
            Thread thread = new ExtraThread();
            extraThreadList.add(thread);
            thread.start();
            log.info("启动额外线程:{}", thread.getName());
        }
        //因为启动了额外的线程，因此能够继续添加任务
        if (!blockingQueue.offer(command)) {
            //如果此时还添加任务失败，先抛异常
            throw new RuntimeException("阻塞队列在启动额外线程后，还是满了！");
        }

    }


    class CoreThread extends Thread {
        @Override
        public void run() {
            while (true) {
                try {
                    //当阻塞队列为空的情况下，阻塞线程 不会CPU空转
                    Runnable take = blockingQueue.take();
                    take.run();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }


    class ExtraThread extends Thread {
        @Override
        public void run() {
            while (true) {
                try {
                    //阻塞队列，带超时时间的获取任务
                    Runnable poll = blockingQueue.poll(TIMEOUT, TIME_UNIT);
                    if (poll == null) {
                        //额外线程，如果指定时间内，没有拿到任务，那么应该结束掉当前额外线程，这里也就是线程池的缩容
                        //跳出循环，那么extraTask就结束了，对应的Thread因为Runnable结束了，本身也会结束了
                        break;
                    }
                    poll.run();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
            log.info("额外线程：{} 在指定时间内，从阻塞队列里面没有获取到任务，结束了", Thread.currentThread().getName());

        }
    }


}



```

极端测试用例： 阻塞队列长度特别短
```java

package com.example;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.TimeUnit;

@Slf4j
public class TestPool {
    public static void main(String[] args) {
        MyThreadPool myThreadPool = new MyThreadPool(
                2,4,1, TimeUnit.SECONDS,new ArrayBlockingQueue<>(1)
        );
        for (int i = 0; i < 5; i++) {

            myThreadPool.execute(() -> {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.info("任务执行完成 线程名:{}", Thread.currentThread().getName());
            });
        }

        log.info("主线程没有被阻塞");
    }
}

```
结果：
```
15:18:20.652 [main] INFO com.example.MyThreadPool -- 启动额外线程:Thread-2
15:18:20.658 [main] INFO com.example.MyThreadPool -- 启动额外线程:Thread-3
Exception in thread "main" java.lang.RuntimeException: 阻塞队列在启动额外线程后，还是满了！
	at com.example.MyThreadPool.execute(MyThreadPool.java:68)
	at com.example.TestPool.main(TestPool.java:16)
15:18:21.664 [Thread-0] INFO com.example.TestPool -- 任务执行完成 线程名:Thread-0
15:18:21.664 [Thread-2] INFO com.example.TestPool -- 任务执行完成 线程名:Thread-2
15:18:21.664 [Thread-3] INFO com.example.MyThreadPool -- 额外线程：Thread-3 在指定时间内，从阻塞队列里面没有获取到任务，结束了
15:18:21.664 [Thread-1] INFO com.example.TestPool -- 任务执行完成 线程名:Thread-1
15:18:22.665 [Thread-2] INFO com.example.MyThreadPool -- 额外线程：Thread-2 在指定时间内，从阻塞队列里面没有获取到任务，结束了

```
可以看到，实际上五个任务只完成了3个，另外两个任务因为抛异常了，实际上没有启动


## 加上拒绝策略的线程池

定义拒绝策略接口
```java
package com.example;

public interface RejectHandler {

    void reject(Runnable rejectCommand, MyThreadPool pool);
}

```
常见的拒绝策略：
```java
public class ThrowExceptionRejectHandler implements RejectHandler {
    @Override
    public void reject(Runnable rejectCommand, MyThreadPool pool) {
        throw new RuntimeException("阻塞队列满了-执行抛异常的拒绝策略");
    }
}

```
```java
package com.example;

import lombok.extern.slf4j.Slf4j;

@Slf4j
public class DiscardRejectHandler implements RejectHandler {
    @Override
    public void reject(Runnable rejectCommand, MyThreadPool pool) {
        //抛弃掉一个任务，重新加入当前的任务
        pool.blockingQueue.poll();
        pool.execute(rejectCommand);
        log.info("抛弃掉一个任务，重新加入当前的任务");
    }
}

```


带有拒绝策略的自定义线程池：

```java

package com.example;

import lombok.extern.slf4j.Slf4j;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;

@Slf4j
public class MyThreadPool {

    public int corePoolSize = 5;
    public int maxPoolSize = 10;
    public int TIMEOUT = 1;
    public TimeUnit TIME_UNIT = TimeUnit.SECONDS;
    BlockingQueue<Runnable> blockingQueue = new ArrayBlockingQueue<>(1024);
    public RejectHandler rejectHandler = new RejectHandler() {
        @Override
        public void reject(Runnable rejectCommand, MyThreadPool pool) {
            log.error("触发默认的拒绝策略");
        }
    };

    public MyThreadPool(int corePoolSize, int maxPoolSize, int timeout, TimeUnit timeUnit, BlockingQueue<Runnable> blockingQueue, RejectHandler rejectHandler) {
        this.corePoolSize = corePoolSize;
        this.maxPoolSize = maxPoolSize;
        this.TIMEOUT = timeout;
        this.TIME_UNIT = timeUnit;
        this.blockingQueue = blockingQueue;
        this.rejectHandler = rejectHandler;
    }

    public MyThreadPool() {
    }

    //线程池应该有多少个线程合适？用corePoolSize变量表示
    List<Thread> coreThreadList = new ArrayList<>();
    //    阻塞队列满了，需要启动额外线程来处理任务
    List<Thread> extraThreadList = new ArrayList<>();


    /**
     * 1. 线程何时创建
     * 2. 线程的runnable是什么？
     * 3. 为什么要加锁，因为判断大小和添加到线程List，两个操作不是原子的
     *
     * @param command
     */
    public synchronized void execute(Runnable command) {
        //线程池中线程数量小于corePoolSize，创建线程
        if (coreThreadList.size() < corePoolSize) {
            //每个线程绑定一个task，task就是循环从阻塞队列获取任务，执行任务的这么一个逻辑
            Thread thread = new CoreThread();
            coreThreadList.add(thread);
            thread.start();//启动这个逻辑
        }
        //任务添加到阻塞队列
        boolean offerResult = blockingQueue.offer(command);
        if (offerResult) {
            //添加任务成功
            return;
        }
        //此时添加任务失败
        //阻塞队列满了，怎么处理？使用额外线程
        if (extraThreadList.size() + coreThreadList.size() < maxPoolSize) {
            Thread thread = new ExtraThread();
            extraThreadList.add(thread);
            thread.start();
            log.info("启动额外线程:{}", thread.getName());
        }
        //因为启动了额外的线程，因此能够继续添加任务
        if (!blockingQueue.offer(command)) {
            //如果此时还添加任务失败，使用拒绝策略处理
            rejectHandler.reject(command, this);
        }

    }


    class CoreThread extends Thread {
        @Override
        public void run() {
            while (true) {
                try {
                    //当阻塞队列为空的情况下，阻塞线程 不会CPU空转
                    Runnable take = blockingQueue.take();
                    take.run();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }


    class ExtraThread extends Thread {
        @Override
        public void run() {
            while (true) {
                try {
                    //阻塞队列，带超时时间的获取任务
                    Runnable poll = blockingQueue.poll(TIMEOUT, TIME_UNIT);
                    if (poll == null) {
                        //额外线程，如果指定时间内，没有拿到任务，那么应该结束掉当前额外线程，这里也就是线程池的缩容
                        //跳出循环，那么extraTask就结束了，对应的Thread因为Runnable结束了，本身也会结束了
                        break;
                    }
                    poll.run();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
            log.info("额外线程：{} 在指定时间内，从阻塞队列里面没有获取到任务，结束了", Thread.currentThread().getName());

        }
    }


}

```
测试代码：
```java
package com.example;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.TimeUnit;

@Slf4j
public class TestPool {
    public static void main(String[] args) {
        MyThreadPool myThreadPool = new MyThreadPool(
                2, 4, 1, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(1),
                new ThrowExceptionRejectHandler()
        );
        for (int i = 0; i < 5; i++) {
            int finalI = i;
            log.info("即将启动任务：{}",finalI);
            myThreadPool.execute(() -> {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.info("任务:{} 执行完成 线程名:{}", finalI, Thread.currentThread().getName());
            });
        }

        log.info("主线程没有被阻塞");
    }
}

```
测试结果：
```
15:33:38.311 [main] INFO com.example.TestPool -- 即将启动任务：0
15:33:38.324 [main] INFO com.example.TestPool -- 即将启动任务：1
15:33:38.325 [main] INFO com.example.MyThreadPool -- 启动额外线程:Thread-2
Exception in thread "main" java.lang.RuntimeException: 阻塞队列满了-执行抛异常的拒绝策略
	at com.example.ThrowExceptionRejectHandler.reject(ThrowExceptionRejectHandler.java:6)
	at com.example.MyThreadPool.execute(MyThreadPool.java:77)
	at com.example.TestPool.main(TestPool.java:19)
15:33:39.334 [Thread-0] INFO com.example.TestPool -- 任务:0 执行完成 线程名:Thread-0
15:33:39.334 [Thread-2] INFO com.example.MyThreadPool -- 额外线程：Thread-2 在指定时间内，从阻塞队列里面没有获取到任务，结束了


使用丢弃的拒绝策略：
15:40:11.072 [main] INFO com.example.TestPool -- 即将启动任务：0
15:40:11.080 [main] INFO com.example.TestPool -- 即将启动任务：1
15:40:11.080 [main] INFO com.example.MyThreadPool -- 启动额外线程:Thread-2
15:40:11.081 [main] INFO com.example.TestPool -- 即将启动任务：2
15:40:11.081 [main] INFO com.example.MyThreadPool -- 启动额外线程:Thread-3
15:40:11.081 [main] INFO com.example.DiscardRejectHandler -- 抛弃掉一个任务，重新加入当前的任务
15:40:11.081 [main] INFO com.example.TestPool -- 即将启动任务：3
15:40:11.081 [main] INFO com.example.TestPool -- 即将启动任务：4
15:40:11.081 [main] INFO com.example.TestPool -- 主线程没有被阻塞
15:40:12.095 [Thread-0] INFO com.example.TestPool -- 任务:0 执行完成 线程名:Thread-0
15:40:12.095 [Thread-2] INFO com.example.TestPool -- 任务:3 执行完成 线程名:Thread-2
15:40:12.095 [Thread-1] INFO com.example.TestPool -- 任务:2 执行完成 线程名:Thread-1
15:40:12.095 [Thread-3] INFO com.example.TestPool -- 任务:4 执行完成 线程名:Thread-3
15:40:13.110 [Thread-2] INFO com.example.MyThreadPool -- 额外线程：Thread-2 在指定时间内，从阻塞队列里面没有获取到任务，结束了
15:40:13.110 [Thread-3] INFO com.example.MyThreadPool -- 额外线程：Thread-3 在指定时间内，从阻塞队列里面没有获取到任务，结束了

```



## 加上shutdown功能

1. 添加状态标志：引入一个状态标志来表示线程池是否正在关闭。
2. 修改 execute 方法：在 execute 方法中检查线程池是否正在关闭，如果是，则拒绝新任务。
3. 实现 shutdown 方法：设置状态标志，不再接受新任务，但允许现有任务完成。
4. 处理线程终止：确保 CoreThread 和 ExtraThread 在完成当前任务后能够正常退出。

```java
package com.example;

import lombok.extern.slf4j.Slf4j;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;

@Slf4j
public class MyThreadPool {

    private volatile boolean isShutdown = false;


    public int corePoolSize = 5;
    public int maxPoolSize = 10;
    public int TIMEOUT = 1;
    public TimeUnit TIME_UNIT = TimeUnit.SECONDS;
    BlockingQueue<Runnable> blockingQueue = new ArrayBlockingQueue<>(1024);
    public RejectHandler rejectHandler = new RejectHandler() {
        @Override
        public void reject(Runnable rejectCommand, MyThreadPool pool) {
            log.error("触发默认的拒绝策略");
        }
    };

    public MyThreadPool(int corePoolSize, int maxPoolSize, int timeout, TimeUnit timeUnit, BlockingQueue<Runnable> blockingQueue, RejectHandler rejectHandler) {
        this.corePoolSize = corePoolSize;
        this.maxPoolSize = maxPoolSize;
        this.TIMEOUT = timeout;
        this.TIME_UNIT = timeUnit;
        this.blockingQueue = blockingQueue;
        this.rejectHandler = rejectHandler;
    }

    public MyThreadPool() {
    }

    //线程池应该有多少个线程合适？用corePoolSize变量表示
    List<Thread> coreThreadList = new ArrayList<>();
    //    阻塞队列满了，需要启动额外线程来处理任务
    List<Thread> extraThreadList = new ArrayList<>();


    /**
     * 1. 线程何时创建
     * 2. 线程的runnable是什么？
     * 3. 为什么要加锁，因为判断大小和添加到线程List，两个操作不是原子的
     *
     * @param command
     */
    public synchronized void execute(Runnable command) {

        if (isShutdown){
            rejectHandler.reject(command,this);
            return;
        }

        //线程池中线程数量小于corePoolSize，创建线程
        if (coreThreadList.size() < corePoolSize) {
            //每个线程绑定一个task，task就是循环从阻塞队列获取任务，执行任务的这么一个逻辑
            Thread thread = new CoreThread();
            coreThreadList.add(thread);
            thread.start();//启动这个逻辑
        }
        //任务添加到阻塞队列
        boolean offerResult = blockingQueue.offer(command);
        if (offerResult) {
            //添加任务成功
            return;
        }
        //此时添加任务失败
        //阻塞队列满了，怎么处理？使用额外线程
        if (extraThreadList.size() + coreThreadList.size() < maxPoolSize) {
            Thread thread = new ExtraThread();
            extraThreadList.add(thread);
            thread.start();
            log.info("启动额外线程:{}", thread.getName());
        }
        //因为启动了额外的线程，因此能够继续添加任务
        if (!blockingQueue.offer(command)) {
            //如果此时还添加任务失败，使用拒绝策略处理
            rejectHandler.reject(command, this);
        }

    }

    public void shutdown() {
        isShutdown = true;
        // 中断所有线程，以便它们能够检查isShutdown标志并退出
        for (Thread thread : coreThreadList) {
            thread.interrupt();
        }
        for (Thread thread : extraThreadList) {
            thread.interrupt();
        }
    }




    class CoreThread extends Thread {
        @Override
        public void run() {
            while (true) {
                try {
                    //当阻塞队列为空的情况下，阻塞线程 不会CPU空转
                    Runnable take = blockingQueue.take();
                    take.run();
                } catch (InterruptedException e) {
                    // 如果线程被中断，检查是否需要退出
                    if (isShutdown){
                        break;
                    }
                    throw new RuntimeException(e);
                }
            }
        }
    }


    class ExtraThread extends Thread {
        @Override
        public void run() {
            while (true) {
                try {
                    //阻塞队列，带超时时间的获取任务
                    Runnable poll = blockingQueue.poll(TIMEOUT, TIME_UNIT);
                    if (poll == null) {
                        //额外线程，如果指定时间内，没有拿到任务，那么应该结束掉当前额外线程，这里也就是线程池的缩容
                        //跳出循环，那么extraTask就结束了，对应的Thread因为Runnable结束了，本身也会结束了
                        break;
                    }
                    poll.run();
                } catch (InterruptedException e) {
                    // 如果线程被中断，检查是否需要退出
                    if (isShutdown){
                        break;
                    }
                    throw new RuntimeException(e);
                }
            }
            log.info("额外线程：{} 在指定时间内，从阻塞队列里面没有获取到任务，结束了", Thread.currentThread().getName());

        }
    }


}

```