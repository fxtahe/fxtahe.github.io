---
title: CompletableFuture 异步编排
date: 2022-09-02 22:57:08
tags:
	- CompletableFuture 
categories:
	- 并发编程
---


## 从Future聊起
`Future`是java 1.5引入的异步编程api,它表示一个异步计算结果，提供了获取异步结果的能力，解决了多线程场景下`Runnable`线程任务无法获取结果的问题。  

但是其获取异步结果的方式并不够优雅，我们必须使用Future.get的方式阻塞调用线程，或者使用轮询方式判断 Future.isDone 任务是否结束，再获取结果。
```java
public interface Future<V> {

    //任务是否完成
    boolean isDone();
    
    //阻塞调用线程获取异步结果
    V get() throws InterruptedException, ExecutionException;

   //在指定时间内阻塞线程获取异步结果
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
假如存在多个异步任务相互依赖，一个或多个异步线程任务需要依赖上一个异步线程任务结果，并且多个异步任务能够组合结果，显然这种阻塞线程的方式并不能优雅解决。我们更希望能够提供一种异步回调的方式，组合各种异步任务，而无需开发者对多个异步任务结果的监听编排。
为了解决优化上述问题，java8 新增了`CompletableFuture`API ,其大大扩展了`Future`能力，并提供了异步任务编排能力。  
<!-- more -->
## CompletableFuture
`CompletableFuture`实现了新的接口`CompletionStage`，并扩展了`Future`接口。查看类图
[![CompletableFuture.png](https://s1.ax1x.com/2022/09/02/vIYOiV.png)](https://imgse.com/i/vIYOiV)


### 创建异步任务

`CompletableFuture` 提供了四种方法去创建一个异步任务。

-  `static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier)`:创建一个有返回值的异步任务实例
-  `static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,Executor executor)`:创建一个有返回值的异步任务实例，可以指定线程池
-  `static CompletableFuture<Void> runAsync(Runnable runnable)`:创建一个无返回值的任务实例
-  `static CompletableFuture<Void> runAsync(Runnable runnable,Executor executor)`:创建一个无返回值的任务实例，允许指定线程池

着几个方法本质上是有返回值和无返回值两种类型方法，supply方法可以获取异步结果，而run方法则无返回值，根据需要使用。  
同时两种类型的方法均提供了指定线程池的重载，如果不指定线程池会默认使用`ForkJoinPool.commonPool()`,默认线程数为cpu核心数，**建议使用自定义线程池的方式，避免线程资源竞争**

一个简单样例
```java
        CompletableFuture<Void> runAsync = CompletableFuture.runAsync(() -> { System.out.println("无返回值任务"); });
        runAsync.get();
        CompletableFuture<String> supplyAsync = CompletableFuture.supplyAsync(() -> "hello completableFuture");
        String result = supplyAsync.get();
        System.out.println(result);
```
我们依然可以通过`get()`方法阻塞获取异步结果任务，但是`CompletableFuture`主要还是用于异步回调及异步任务编排使用。

### 异步回调
在任务执行结束后我们希望能够自动触发回调方法，`CompletableFuture`提供了两种方法实现。

- `CompletableFuture<T> whenComplete(
        BiConsumer<? super T, ? super Throwable> action)`:当上一阶段任务执行结束后，回调方法接受上一阶段结果或者异常，返回上一阶段任务结果
- `<U> CompletableFuture<U> handle(
        BiFunction<? super T, Throwable, ? extends U> fn)`:当上一阶段任务执行结束后，回调方法接受上一阶段结果或者异常,并最终返回回调方法处理结果
- `CompletableFuture<T> exceptionally(
        Function<Throwable, ? extends T> fn)`:上一阶段任务出现异常后的回调，返回结果是回调函数的返回结果。

**whenComplete 与 handle 区别**：两者均接受上一阶段任务结果或异常，但是whenComplete 回调中没有返回值，所以其结果是上一阶段任务，而handle 最终返回的是其回调方法方法，其主要是`BiConsumer`与`BiFunction`的区别。



### 异步编排

`CompletionStage`表示异步计算的一个阶段，当一个计算处理完成后会触发其他依赖的阶段。当然一个阶段的触发也可以是由多个阶段的完成触发或者多个中的任意一个完成触发。该接口定义了异步任务编排的各种场景，`CompletableFuture`则实现了这些场景。

可以把这些场景大致分为三类：串行、AND和OR。下面会逐个分析各个场景，**接口中定义的以`Async`结尾的方法，指下一阶段任务会被单独提交到线程池中执行，后面不在赘述。**

#### 串行
当上一阶段任务执行完毕后，继续提交执行其他任务

- `<U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)`:接收上一阶段任务结果，并可获取返回值。
- `CompletableFuture<Void> thenAccept(Consumer<? super T> action)`:接收上一阶段任务结果，无返回值。
- `CompletableFuture<Void> thenRun(Runnable action)`:不接收上一阶段任务结果，并且无返回值。

*T：上一个任务返回结果的类型 U：当前任务的返回值类型*

#### AND
组合多个异步任务，当多个任务执行完毕继续执行其他任务

- `<U,V> CompletableFuture<V> thenCombine(
        CompletionStage<? extends U> other,
        BiFunction<? super T,? super U,? extends V> fn)`:上一阶段任务与other任务均执行结束，接收两个任务的结果，并可获取返回值
- `<U> CompletableFuture<U> thenCompose(
        Function<? super T, ? extends CompletionStage<U>> fn)`: 使用上一阶段任务的结果，返回一个新的`CompletableFuture`实例
- `<U> CompletableFuture<Void> thenAcceptBoth(
        CompletionStage<? extends U> other,
        BiConsumer<? super T, ? super U> action)`:上一阶段任务与other任务均执行结束，接收两个任务的结果,无返回值
- `CompletableFuture<Void> runAfterBoth(CompletionStage<?> other,
                                                Runnable action)`:上一阶段任务与other任务均执行结束，不接收两个任务的结果,无返回值
- `static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs)`:等待所有异步任务执行结束

*T：上一个任务返回结果的类型 U：上一个other任务的返回值类型 V:当前任务返回值*

#### OR
当多个任务中任意任务执行完成则继续执行其他任务。

- `<U> CompletableFuture<U> applyToEither(
        CompletionStage<? extends T> other, Function<? super T, U> fn)`: 接收上一阶段任务与other任务最快执行完成的结果，并可获取返回值
- `CompletableFuture<Void> acceptEither(
        CompletionStage<? extends T> other, Consumer<? super T> action)`:接收上一阶段任务与other任务最快执行完成的结果，无返回值
- `CompletableFuture<Void> runAfterEither(CompletionStage<?> other,         Runnable action)`:上一阶段任务与other任务任意任务完成执行，不接收结果，无返回值

- `static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs)`:组合多个任务，返回最快执行结束的任务结果




### Future 机制扩展
`CompletableFuture`不仅实现了`Future`接口，同时对其进行了扩展，提供了更加优雅的实现。

- `T join() `:与`get()`方法用法一致，阻塞调用线程获取结果，但是不会抛出具体异常，简化了使用上下文
- `T getNow(T valueIfAbsent)`:当任务结束返回任务结果，否则返回给定的结果valueIfAbsent。
- `boolean complete(T value)`:当任务未结束时设置给定的结果value并结束任务,已结束的任务不会生效。
- `boolean completeExceptionally(Throwable ex)`:当任务未结束时设置异常结果并结束任务，已结束的任务不会生效


## CompletableFuture 实践
我们通过`CompletableFuture`实现一个经典的烧水程序。

[![烧水.png](https://s1.ax1x.com/2022/09/02/vITI41.png)](https://imgse.com/i/vITI41)

我们可以把这个流程分为三个异步任务。任务1：洗水壶->烧水，任务2：洗水壶->洗茶杯->拿茶叶，任务3：泡茶，需要等待任务1与任务2结束。



通过代码模拟实现
```java
        CompletableFuture<String> task1 = CompletableFuture.supplyAsync(() -> {
            System.out.println("洗水壶");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException ex) {
                ex.printStackTrace();
            }
            return "水壶";
        }).thenApply(e->{
            System.out.println("烧水");
            try {
                Thread.sleep(5000);
            } catch (InterruptedException ex) {
                ex.printStackTrace();
            }
            return "热水";
        });
        //洗水壶->洗水杯->拿茶叶
        CompletableFuture<String> task2 = CompletableFuture.supplyAsync(() -> {
            System.out.println("洗茶壶");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException ex) {
                ex.printStackTrace();
            }
            return "茶壶";
        }).thenApply(e->{
            try {
                Thread.sleep(2000);
            } catch (InterruptedException ex) {
                ex.printStackTrace();
            }
            System.out.println("洗水杯");
            return "水杯";
        }).thenApply(e->{
            System.out.println("拿茶叶");
            return "茶叶";
        });
        //泡茶
        CompletableFuture<String> task3 = task1.thenCombine(task2, (a, b) -> {
            System.out.println("泡茶");
            return "茶";
        });
        String tea = task3.join();
        System.out.println(tea);
```



