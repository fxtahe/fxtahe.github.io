---
title: 并发编程基础
date: 2022-09-01 22:39:04
categories: 
- 并发编程
tags:
    - 多线程
    - 线程池
    - Future
---

## 线程
>线程是程序中一个单一的顺序控制流程.在单个程序中同时运行多个线程完成不同的工作,称为多线程.

线程是cpu执行的最小基本单位，如果一个cpu只能运行一个线程，那你的电脑就只能运行一个程序，这种情况是难以接受的，所以多线程就是为了解决并行执行任务的能力。cpu通过时间片切换的方式去处理多个线程，视觉上就出现了并行执行的效果。
<!-- more -->
### 线程状态
线程主要包括以下几种状态：1.新建 2.就绪 3.运行 4.阻塞 5.死亡  
阻塞又分为等待阻塞（调用wait方法），同步阻塞（同步未获得所情况下）及其他类型阻塞（线程调用sleep或join方法）
下图展示了线程的状态流转，展示了线程的整个生命周期。

[![QDXJts.md.jpg](https://s2.ax1x.com/2019/12/10/QDXJts.md.jpg)](https://imgse.com/i/QDXJts)

## java线程
Java线程主要通过线程类Thread实现。
先介绍下Thread类的方法  
- start():创建线程后，用于启动线程。
- run():线程普通方法，直接调用依然只会有主线程。
- sleep(long millis):线程睡眠方法，线程再调用该方法后不会释放锁。
- join():A线程调用B线程的join()方法，将会使A等待B执行，直到B线程终止。如果传入time参数，将会使A等待B执行time的时间，如果time时间到达，将会切换进A线程，继续执行A线程。
- yield():为相同或高于优先级线程让出线程执行时间，等待下次执行，将当前线程置为就绪态
- interrupt():中断处于阻塞状态的线程  

++**start与run的区别：** start方法将创建的线程置为就绪状态，等待cpu执行该线程。run方法则是普通方法，直接调用程序依然顺序执行++

### 多线程实现
JAVA实现多线程实现有两种方式：**继承Thread与实现Runnable接口**。

- 继承Thread

通过继承Thread类，重写run方法
```java
public class Mythread extends Thread{
	public Mythread(String name){
		super(name);
	}
	@Override
	public void run(){
		for (int i = 0; i <10; i++) {
		    //TODO
			System.out.println(Thread.currentThread().getName()+"--"+i);
		}
	}
	public static void main(String[] args) {
		Mythread m1 = new Mythread("线程1");
		Mythread m2 = new Mythread("线程2");
		m1.start();
		m2.start();
	}
}


```
- 实现Runnable

通过实现Runnable接口，并重写run方法，然后作为Thread构造器参数传入。
```java
public class MyRunnable implements Runnable{
	@Override
	public void run() {
		for (int i = 0; i < 10; i++) {
			System.out.println(Thread.currentThread().getName()+"--"+i);
		}
	}
	public static void main(String[] args) {
		MyRunnable m1 = new MyRunnable();
		MyRunnable m2 = new MyRunnable();
		Thread t1 = new Thread(m1,"线程1");
		Thread t2 = new Thread(m2,"线程2");
		t1.start();
		t2.start();
	}
}
```

**关于两种实现方式：**
这里所说的多线程实现其实是将线程的线程工作单元与任务执行单元糅合在了一起，本质上说**Java的线程模型抽象只有`Thread`，而`Runnable`只是任务执行单元**,从上面的demo便可以看出，`Runnable`实例依赖于`Thread`线程模型的执行，然后通过调用`start()`方法告知jvm 调度线程工作，调用`run()`完成任务执行。

`Thread`类其实是Runnable接口的实现
```java
public
class Thread implements Runnable
```
所以`Thread`既可以作为一个线程，也可以作为一个线程任务执行 。
java还提供了另外一个任务执行单元接口`Callable`,下面会介绍


### 线程池 ThreadPoolExecutor
线程池就是将线程池化，为了更好管理线程资源，减少创建线程和销毁线程的资源损耗，提高系统性能。

`ThreadPoolExecutor` 提供了多个构造重载方法，其核心构造如下
```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
其中就包含了线程池的核心属性
- corePoolSize:核心线程数
- maximumPoolSize：最大线程数
- keepAliveTime：当线程数大于核心线程数时，多余空闲线程在终止前等待新任务的最大时间
- TimeUnit：时间单位
- BlockingQueue：在任务执行之前用于保存任务的队列。这个队列只保存execute方法提交的Runnable任务
- ThreadFactory：线程创建工厂类，可以自定义线程属性。
- RejectedExecutionHandler：无可用线程资源时的任务拒绝策略，默认实现了四种策略
    - AbortPolicy：当任务添加到线程池中被拒绝时，它将抛出RejectExecutionException异常,默认配置。
    - CallerRunsPolicy：当任务添加到线程池中被拒绝时，会使用调用线程池的Thread线程对象处理被拒绝的任务。
    - DiscardPolicy:当任务添加到线程池中被拒绝时，线程池将丢弃被拒绝的任务。
    - DiscardOldestPolicy:当任务添加到线程池中被拒绝时，线程池会放弃等待队列中最旧的未处理任务，然后将任务添加到等待队列中。
    
#### 核心原理

线程池的核心工作机制如下  
1.线程池提交任务会创建核心线程  
2.当核心线程数等于corePoolSize，会把任务放到阻塞队列BlockingQueue中  
3.当任务队列满了，会继续创建线程  
4.当最大线程数等于maximumPoolSize，则会执行拒绝策略  
5.当在keepAliveTime时间内 没有新任务提交则会销毁空闲线程  

[![核心原理.png](https://s1.ax1x.com/2022/09/01/v5RFsI.png)](https://imgse.com/i/v5RFsI)


#### 常用线程池
java 线程池工具类`Executors` 提供了几种常用线程池
- FixThreadPool:固定线程数量线程池
```java
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```
- SingleThreadExecutor：单线程的线程池
```java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```
- CachedThreadPool：线程数量最大为Integer.MAX.VALUE 的无界线程池
```java
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```
**注意**：因为这些线程池队列没有指定大小，可能会导致OOM,所以建议手动创建线程池方式去使用线程池

#### 线程池任务提交

线程池提供了多个任务提交方法，可根据不同场景使用
- `void execute(Runnable command)`:提交一个`Runable`线程任务到线程池，无返回值
- `Future<?> submit(Runnable task)`: 提交一个`Runable`线程任务到线程池，返回一个执行结果为`null`的`Future`。
- `<T> Future<T> submit(Runnable task, T result)`:提交一个`Runable`线程任务到线程池，返回一个执行结果为`result`的`Future`。
- `<T> Future<T> submit(Callable<T> task)`:提交一个`Callable`线程任务到线程池，返回可以获取`Callable`处理结果的`Future`。

通过上述方法名可以区分两种任务的提交方式。`execute`提交无返回值任务而`submit`方法则提交有返回值的任务。

**`Future`:表示异步执行的结果，通过阻塞调用`get`方法线程获取结果**
**`Callable`:可以返回结果的线程任务，补充Runable无法返回结果的场景**
#### 优雅关闭线程池
线程池创建使用完成后需要手动关闭，线程池提供了两种方法去关闭线程池

- `void shutdown()`:等待已提交的任务执行完成，根据拒绝策略直接拒绝后续新提交的任务。
- `List<Runnable> shutdownNow()`:终结正在运行的任务，停止等待任务的处理，并返回等待任务的列表。

通常情况下我们不希望正在执行的任务被打断，并且在主线程关闭线程池时可以阻塞主线程等待任务执行完毕，线程池提供了相应的方法

- ` boolean awaitTermination(long timeout, TimeUnit unit)`:关闭线程操作之后，可以在指定时间内阻塞调用线程直到所有线程池任务执行完毕。

结合上述的方法我们可以实现更加优雅的关闭线程池
```java
        //关闭线程池
        threadPool.shutdown();

        try {
            //阻塞当前线程等待任务完成
            if(!threadPool.awaitTermination(5,TimeUnit.SECONDS)){
                //等待时间结束后后立即关闭线程池
                threadPool.shutdownNow();
            }
        } catch (InterruptedException exception) {
            //异常情况立即关闭
            threadPool.shutdownNow();
        }
```


### 多线程的结果返回：Future与Callable
在使用`Thread`与`Runnable`创建线程任务时无法获取返回值，为了对这种使用场景进行完善，java 1.5 提供了一套新的API,即`Future`与`Callable`。  

#### Future
在类注释中
> A Future represents the result of an asynchronous computation. Methods are provided to check if the computation is complete, to wait for its completion, and to retrieve the result of the computation. The result can only be retrieved using method get when the computation has completed, blocking if necessary until it is ready. Cancellation is performed by the cancel method. Additional methods are provided to determine if the task completed normally or was cancelled. Once a computation has completed, the computation cannot be cancelled. If you would like to use a Future for the sake of cancellability but not provide a usable result, you can declare types of the form Future<?> and return null as a result of the underlying task.    

> Future表示异步计算的结果。方法用于检查计算是否完成、等待计算完成以及检索计算结果。只有在计算完成后，才能使用get方法检索结果，在必要时阻塞直到它准备好。取消是由cancel方法执行的。还提供了其他方法来确定任务是正常完成还是被取消。一旦计算完成，就不能取消计算。如果为了可取消性而使用Future，但又不想提供一个可用的结果，你可以声明形式Future<?>并返回null作为底层任务的结果。

简单来说，`Future`提供了对异步操作结果的处理api,逐个看下

- `boolean cancel(boolean mayInterruptIfRunning)`:取消一个正在执行的任务。
    - 对已结束、已取消或者无法取消任务执行该操作会失败,返回false。
    - 如果调用成功，而此任务尚未启动，则此任务将永不运行
    - mayInterruptIfRunning参数决定了是否向执行任务的线程发出interrupt操作
- `boolean isCancelled()`:是否在任务执行结束前被取消
- `boolean isDone()`:任务否结束，包括正常结束、被终结、异常或者被取消
- `V get()`:阻塞等待任务完成，并返回结果
- `V get(long timeout, TimeUnit unit)`:在指定时间内阻塞等待任务完成，并返回结果

#### 带返回值的任务Callable
`Callable`与`Runnable`均被设计用于线程任务，但是`Callable`任务带返回值，并且可以抛出异常，在需要获取返回值的场景中使用

```java
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

#### 线程的结果返回

在线程池中提供了带返回值的任务提交方法，查看源码
```java
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
    
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }
```
该方法提交任务会创建一个接受`Callable`的`FutureTask`实例,并调用无返回值方法`execute`执行。查看`FutureTask`类关系图

[![FutureTask.png](https://s1.ax1x.com/2022/09/01/v5fkUf.png)](https://imgse.com/i/v5fkUf)

本质上 `FutureTask`也是一个`Runnable`实例，所以可以在线程池中执行，并且其实现了`Future`,则可以通过`get()`方法获取最终的结果。
所有我们也可以通过创建普通线程的方式实现

```java
public class MyCallable implements Callable<String> {
    @Override
    public String call() throws Exception {
        TimeUnit.SECONDS.sleep(2);
        return "hello world";
    }
}
```
```java
    public static void main(String[] args) throws ExecutionException, InterruptedException {

        FutureTask<String> future = new FutureTask<>(new MyCallable());
        new Thread(future).start();
        String result = future.get();
        System.out.println(result);

    }
```
`Future`会阻塞主线程等待任务处理结束，当然我们不推荐手动创建线程的方式去使用，看下线程池的使用例子
通过一个实例演示下
```java
        List<Future<String>> list = new ArrayList<>();
        Future<String> future1 = threadPoolExecutor.submit(() -> {
            Thread.sleep(2000);
            return "hello callable1";
        });
        list.add(future1);
        Future<String> future2 = threadPoolExecutor.submit(() -> {
            Thread.sleep(3000);
            return "hello callable2";
        });
        list.add(future2);
        for(Future<String> future:list){
            String result = future.get();
            System.out.println("result: "+result);
        }
        System.out.println("end");
```
```
result: hello callable1
result: hello callable2
end
```
我们提交了多个任务但是会等待所有任务执行完毕，本质上是`Future`阻塞了主线程等待任务执行结束。
