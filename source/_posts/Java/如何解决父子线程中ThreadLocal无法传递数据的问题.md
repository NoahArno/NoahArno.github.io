---
title: 如何解决父子线程中ThreadLocal无法传递数据的问题
category_bar: true
date: 2023-04-06 22:56:57
tags: [Java]
categories:
  - [Java, ThreadLocal]
---

# 如何解决父子线程中ThreadLocal无法传递数据的问题

对于之前分析的 `ThreadLocal`，它虽然能做到数据的线程隔离，也就是不同的线程中存在不同的数据副本，然而，它却存在一个致命问题：无法在父子线程之间传递数据。因为子线程和父线程不是同一个线程，因此它们的数据副本也不会一样。

在 `Netty` 中的 `FastThreadLocal` 虽然以空间换时间的形式实现了自己的 `ThreadLocal`，能在自己的业务需求上提升一定的性能，但是它并没有解决上述问题。

## 1. 官方提供的 InheritableThreadLocal

`InheritableThreadLocal` 继承自 `ThreadLocal`，它的核心部分和 `ThreadLocal` 大差不差，主要是重写了三个方法：

```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {

    // 传递什么参数就返回什么，如果想要这个方法有不同的行为，可以重写该方法
    // 也就是我们可以利用这个在收到父线程传递过来的数据的时候，去做一些特殊的操作，
    // 比如将数据进行修改，或者打印日志
    // This method merely returns its input argument, 
    // and should be overridden if a different behavior is desired.
    protected T childValue(T parentValue) {
        return parentValue;
    }

    ThreadLocalMap getMap(Thread t) {
        return t.inheritableThreadLocals;
    }

    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

也就是通过这三个方法，让我们操控的时候，使用的是 `Thread` 中的另一个 `threadLocalMap` 属性，也就是 `inheritableThreadLocals`，而不是我们之前操控的 `threadLocals`。

同时，在 `new Thread` 的过程中，内部其实是通过复制父线程的 `inheritableThreadLocals` 来进行父子线程之间的数据传递功能，但是这样也造成了一个问题，那就是只有当父线程在创建子线程的过程中，才能数据传递。如果使用场景是线程池，对线程是复用的，并没有新建的操作，这时父子线程关系的`ThreadLocal`值传递已经没有意义，应用需要的实际上是把 **任务提交给线程池时**的`ThreadLocal`值传递到 **任务执行时**。

```java
if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
```

## 2. Alibaba 开源的 ttl

### 2.1 基本使用

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>transmittable-thread-local</artifactId>
    <version>2.14.2</version>
</dependency>
```

`TransmittableThreadLocal` 对于在线程池中的父子线程信息的传递而言，应用需要的实际上是把**任务提交给线程池**时的 `ThreadLocal` 值传递到**任务执行时**。

`TTL` 的使用需要配合 `TtlRunnable` 和 `TtlCallable` 来修饰传入线程池的 `Runnable` 和 `Callable`。然而，**即使时同一个 `Runnable`  任务多次提交到线程池的时候，每次提交的时候都需要通过修饰操作以抓取这次提交时的 `TTL` 上下文的值**；即如果同一个任务下一次提交的时候不执行修饰而仍然使用上一次的 `TtlRunnable`，则提交的任务运行时会是之前修饰操作所抓取的上下文：

```java
TransmittableThreadLocal<String> context = new TransmittableThreadLocal<>();

// 第一次提交
Runnable task = new RunnableTask();
executorService.submit(TtlRunnable.get(task));

// ...业务逻辑代码，
// 并且修改了 TransmittableThreadLocal上下文 ...
context.set("value-modified-in-parent");

// 再次提交
// 重新执行修饰，以传递修改了的 TransmittableThreadLocal上下文
executorService.submit(TtlRunnable.get(task));
```

当然，也可以通过工具类 `TtlExecutors` 来对 `Executor`、`ExecutorService` 和 `ScheduledExecutorService` 进行修饰，省去每次的 `Runnable` 和 `Callable` 传入线程池时的修饰。具体的操作方法可以查看官网。

### 2.2 源码分析

从大体上来说，`TTL` 主要是实现了自己的 `Runnable` ，在任务创建的时候将数据进行传递。

具体的源码分析可以在参考文献中进行查看。

## 3. 参考文献

[alibaba/transmittable-thread-local](https://github.com/alibaba/transmittable-thread-local)

[InheritableThreadLocal与阿里的TransmittableThreadLocal设计思路解析 by liangdu_Zuker](https://blog.csdn.net/u010833547/article/details/99647118)
