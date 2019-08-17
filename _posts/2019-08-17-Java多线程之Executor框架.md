---
layout:     post   				    # 使用的布局（不需要改）
title:      Java多线程之Executor框架				# 标题 
subtitle:   Executor框架是指JDK 1.5中引入的一系列并发库中与Executor相关的功能类，包括Executor、Executors、ExecutorService、Future、Callable等。  #副标题
date:       2019-07-17				# 时间
author:     凌洛 						# 作者
header-img: img/post-bg-desk.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Java
---

# 一、为什么要引入Executor框架？

## 1、如果使用`new Thread(...).start()`的方法处理多线程，有如下缺点：

- 开销大。对于JVM来说，每次新建线程和销毁线程都会有很大的开销。

- 线程缺乏管理。没有一个池来限制线程的数量，如果并发量很高，会创建很多线程，而且线程之间可能会有相互竞争，这将会过多占用系统资源，增加系统资源的消耗量。而且线程数量超过系统负荷，容易导致系统不稳定。

## 2、使用线程池的方法，有如下优点：

- 线程复用。通过复用创建了的线程，减少了线程的创建、消亡的开销。

- 有效控制并发线程数。

- 提供了更简单灵活的线程管理。可以提供定时执行、单线程、可变线程数等多种使用功能。

# 二、Executor框架的UML图
![Executor框架的UML图](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/data/blog_post_bg/f0dc501c74b94d12fc19cca1929d1dad92fc2fe7.png)


# 三、下面开始分析一下Executor框架中几个比较重要的接口和类。

## 1、Callable

Callable位于`java.util.concurrent`包下，它是一个接口，只声明了一个call()方法。

Callable接口类似于Runnable，两者都是为了可能在线程中执行的类而设计的。和Runnable接口中的run()方法类似，Callable 提供的call()方法作为线程的执行体。但是call()比run()方法更强大，体现在

- call方法有返回值。

- call方法可以声明抛出异常。

## 2、Future

Future 接口位于`java.util.concurrent`包下，是Java 1.5中引入的接口。

Future主要用来对具体的Runnable或Callable任务的执行结果进行取消、查询是否完成、获取结果。必要时可以通过get()方法获取执行结果，get()方法会阻塞知道任务返回结果。

当你提交一个Callable对象给线程池时，将得到一个Future对象，并且它和你传入的Callable示例有相同泛型。

Future 接口中的5个方法：
```java
public interface Future<V> {
    //用来取消任务
    //参数mayInterruptIfRunning表示是否允许取消正在执行却没有执行完毕的任务。
    boolean cancel(boolean mayInterruptIfRunning);
    //表示任务是否被取消成功
    boolean isCancelled();
    //表示任务是否已经完成
    boolean isDone();
    //用来获取执行结果，这个方法会产生阻塞，会一直等到任务执行完毕才返回；
    V get()
    //用来获取执行结果，如果在指定时间内，还没获取到结果，会抛出TimeoutException异常。
    V get(long timeout, TimeUnit unit)
}

```
Future提供了三种功能：

- 判断任务是否完成

- 能够中断任务

- 能够获取任务执行结果

## 3、Executor

Executor是一个接口，它将任务的提交与任务的执行分离开来，定义了一个接收Runnable对象的方法executor。Executor是Executor框架中最基础的一个接口，类似于集合中的Collection接口。

## 4、ExecutorService

ExecutorService继承了Executor，是一个比Executor使用更广泛的子类接口。定义了终止任务、提交任务、跟踪任务返回结果等方法。

一个ExecutorService是可以关闭的，关闭之后它将不能再接收任何任务，对于不在使用的ExecutorService，应该将其关闭以释放资源。

ExecutorService方法介绍：
```java
package java.util.concurrent;

import java.util.List;
import java.util.Collection;

public interface ExecutorService extends Executor {

    /**
     * 平滑地关闭线程池，已经提交到线程池中的任务会继续执行完。
     */
    void shutdown();

    /**
     * 立即关闭线程池，返回还没有开始执行的任务列表。
     * 会尝试中断正在执行的任务（每个线程调用 interruput方法），但这个行为不一定会成功。
     */
    List<Runnable> shutdownNow();

    /**
     * 判断线程池是否已经关闭
     */
    boolean isShutdown();

    /**
     * 判断线程池的任务是否已经执行完毕。
     * 注意此方法调用之前需要先调用shutdown()方法或者shutdownNow()方法，否则总是会返回false
     */
    boolean isTerminated();

    /**
     * 判断线程池的任务是否都执行完。
     * 如果没有任务没有执行完毕则阻塞，直至任务完成或者达到了指定的timeout时间就会返回
     */
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    /**
     * 提交带有一个返回值的任务到线程池中去执行（回调），返回的 Future 表示任务的待定结果。
     * 当任务成功完成后，通过 Future 实例的 get() 方法可以获取该任务的结果。
     * Future 的 get() 方法是会阻塞的。
     */
    <T> Future<T> submit(Callable<T> task);

    /**
     *提交一个Runnable的任务，当任务完成后，可以通过Future.get()获取的是提交时传递的参数T result
     * 
     */
    <T> Future<T> submit(Runnable task, T result);

    /**
     * 提交一个Runnable的人无语，它的Future.get()得不到任何内容，它返回值总是Null。
     * 为什么有这个方法？为什么不直接设计成void submit(Runnable task)这种方式？
     * 这是因为Future除了get这种获取任务信息外，还可以控制任务，
     具体体现在 Future的这个方法上：boolean cancel(boolean mayInterruptIfRunning)
     这个方法能够去取消提交的Rannable任务。
     */
    Future<?> submit(Runnable task);

    /**
     * 执行一组给定的Callable任务，返回对应的Future列表。列表中每一个Future都将持有该任务的结果和状态。
     * 当所有任务执行完毕后，方法返回，此时并且每一个Future的isDone()方法都是true。
     * 完成的任务可能是正常结束，也可以是异常结束
     * 如果当任务执行过程中，tasks集合被修改了，那么方法的返回结果将是不确定的，
       即不能确定执行的是修改前的任务，还是修改后的任务
     */
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    /**
     * 执行一组给定的Callable任务，返回对应的Future列表。列表中每一个Future都将持有该任务的结果和状态。
     * 当所有任务执行完毕后或者超时后，方法将返回，此时并且每一个Future的isDone()方法都是true。
     * 一旦方法返回，未执行完成的任务被取消，而完成的任务可能正常结束或者异常结束， 
     * 完成的任务可以是正常结束，也可以是异常结束
     * 如果当任务执行过程中，tasks集合被修改了，那么方法的返回结果将是不确定的
     */
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    /**
     * 执行一组给定的Callable任务，当成功执行完（没抛异常）一个任务后此方法便返回，返回的是该任务的结果
     * 一旦此正常返回或者异常结束，未执行的任务都会被取消。 
     * 如果当任务执行过程中，tasks集合被修改了，那么方法的返回结果将是不确定的
     */
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    /**
     * 执行一组给定的Callable任务，当在timeout（超时）之前成功执行完（没抛异常）一个任务后此方法便返回，返回的是该任务的结果
     * 一旦此正常返回或者异常结束，未执行的任务都会被取消。 
     * 如果当任务执行过程中，tasks集合被修改了，那么方法的返回结果将是不确定的
     */
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

```
`shutdown()` 和 `shutdownNow() `是用来关闭连接池的两个方法，而且这两个方法都是在当前线程立即返回，不会阻塞至线程池中的方法执行结束。调用这两个方法之后，连接池将不能再接受任务。

下面给写几个示例来加深ExecutorService的方法的理解。
先写两个任务类：`ShortTask`和`LongTask`，这两个类都继承了Runnable接口，`ShortTask的run()`方法执行很快，LongTask的run()方法执行时间为10s。
```java
public class LongTask implements Runnable {
    @Override
    public void run() {
        try {
            TimeUnit.SECONDS.sleep(10L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("complete a long task");
    }

}
```

```java
public class ShortTask implements Runnable {
    @Override
    public void run() {
        System.out.println("complete a short task...");
    }
    
}

```
测试shutdown()方法
```java
package OSChina.Client;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.*;

public class Client10 {
    public static void main(String[] args) {
        ExecutorService threadpool = Executors.newFixedThreadPool(4);
        threadpool.submit(new ShortTask());
        threadpool.submit(new ShortTask());
        threadpool.submit(new LongTask());
        threadpool.submit(new ShortTask());
        threadpool.shutdown();
        boolean isShutdown = threadpool.isShutdown();
        System.out.println("线程池是否已经关闭：" + isShutdown);
        final SimpleDateFormat sdf = new SimpleDateFormat("HH:mm:ss");
        try {
            while (!threadpool.awaitTermination(1L, TimeUnit.SECONDS)){
                System.out.println("线程池中还有任务在执行，当前时间：" + sdf.format(new Date()));
            }
            System.out.println("线程池中已经没有在执行的任务，线程池已完全关闭！");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

![image](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/data/blog_post_bg/429d09aab3645b1b57e57d231012f35d90652e05.png)

测试shutdownNow()方法
```java
package OSChina.Client;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class Client11 {
    public static void main(String[] args) {
        ExecutorService threadpool = Executors.newFixedThreadPool(3);
        //将5个任务提交到有3个线程的线程池
        threadpool.submit(new LongTask());
        threadpool.submit(new LongTask());
        threadpool.submit(new LongTask());
        threadpool.submit(new LongTask());
        threadpool.submit(new LongTask());
        try {
            TimeUnit.SECONDS.sleep(1L);
            //关闭线程池
            List<Runnable> waiteRunnables = threadpool.shutdownNow();
            System.out.println("还没有执行的任务数：" + waiteRunnables.size());

            boolean isShutdown = threadpool.isShutdown();
            System.out.println("线程池是否已经关闭：" + isShutdown);

            final SimpleDateFormat sdf = new SimpleDateFormat("HH:mm:ss");
            while (!threadpool.awaitTermination(1L, TimeUnit.SECONDS)) {
                System.out.println("线程池中还有任务在执行，当前时间：" + sdf.format(new Date()));
            }

            System.out.println("线程池中已经没有在执行的任务，线程池已完全关闭！");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
![image](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/data/blog_post_bg/dc654ff2bb2ccfee70d10ee6e4706edca45644d4.png)

当调用`shutdownNow()`后，三个执行的任务都被interrupt了。而且`awaitTermination(1L, TimeUnit.SECONDS)`返回的都是true。

测试`submit(Callable<T> task)`方法
```java
package OSChina.Client;

import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;

public class CallableTask implements Callable{
    @Override
    public Object call() throws Exception {
        TimeUnit.SECONDS.sleep(5L);
        return "success";
    }
}
package OSChina.Client;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class Client12 {
    public static void main(String[] args) {
        ExecutorService threadpool = null;
        threadpool = Executors.newFixedThreadPool(3);
        final SimpleDateFormat sdf = new SimpleDateFormat("HH:mm:ss");
        System.out.println("提交一个callable任务到线程池，现在时间是：" + sdf.format(new Date()));
        Future<String> future = threadpool.submit(new CallableTask());
        try {
            System.out.println("获取callable任务的结果：" + future.get() + "，现在时间是：" + sdf.format(new Date()));
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        } finally {
            if(threadpool!=null){
                threadpool.shutdown();
            }
        }
    }
}
```
![image](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/data/blog_post_bg/ba9777b387f843a7ef256803c2aa4cbf84d9a330.png)