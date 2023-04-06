---
title: Sentinel源码分析05：滑动窗口算法在Sentinel中的应用
category_bar: true
date: 2023-04-06 23:41:18
tags: [Sentinel]
categories:
  - Sentinel
---

# 滑动窗口算法在Sentinel中的应用

在 Sentinel 中，采用高性能的滑动窗口数据结构 `LeapArray` 来统计实时的秒级指标数据，可以很好的支撑写多于读的高并发场景。

在介绍滑动窗口之前，先来看看 `StatisticSlot` 究竟做了什么事情。

## 第一章 StatisticSlot

在官方给的解释中，`StatisticSlot` 是 Sentinel 中最为重要的类之一，用于根据规则判断结果进行相应的统计操作。

entry 的时候：依次执行后面的判断 slot。每个 slot 触发流控的话就会抛出异常。如果有 BlockException 抛出，则记录 block 数据；若无异常抛出则算作可通过，记录 pass 数据。

exit 的时候：若无 error，无论是业务异常还是流控异常，记录 complete（success）以及 RT，线程数 -1.

记录数据的维度：线程数 + 1、记录当前 DefaultNode 数据、记录对应的 originNode 数据（若存在 origin）、累计 IN 统计数据（若流量类型为 IN）。

```java
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                  boolean prioritized, Object... args) throws Throwable {
    try {
        // Do some checking.
        // 向后传递：调用 SlotChain 中后续的所有 Slot，完成所有规则检测（执行过程中可能会抛出异常）
        fireEntry(context, resourceWrapper, node, count, prioritized, args);

        // Request passed, add thread count and pass count.
        // 如果检测通过，就对 node 增加线程数和 通过的请求
        node.increaseThreadNum();
        node.addPassRequest(count); // 涉及到滑动窗口，计算qps

        // 对于整个应用来说，去增加相应的数据
        if (context.getCurEntry().getOriginNode() != null) {
            // Add count for origin node.
            context.getCurEntry().getOriginNode().increaseThreadNum();
            context.getCurEntry().getOriginNode().addPassRequest(count);
        }

        // 什么是入口流量：一般就代表我们的接口对外提供服务，那么通常就是控制入口流量
        // 如果我们在用户服务中使用 getOrderInfo 方法，而 getOrderInfo 会去调用订单服务，
        // 那么对于 getOrderInfo 来说，压力都在订单服务当中，我们就可以指定它为出口流量。
        if (resourceWrapper.getEntryType() == EntryType.IN) {
            // 全局的，用于 SystemRule 检查
            // Sentinel 针对所有的入口流量，使用了一个全局的 ENTRY_NODE，进行统计
            // Add count for global inbound entry node for global statistics.
            Constants.ENTRY_NODE.increaseThreadNum();
            Constants.ENTRY_NODE.addPassRequest(count);
        }

        // Handle pass event with registered entry callback handlers.
        // MetricEntryCallback，也就是在 MetricCallbackInit 中引入的
        for (ProcessorSlotEntryCallback<DefaultNode> handler : StatisticSlotCallbackRegistry.getEntryCallbacks()) {
            handler.onPass(context, resourceWrapper, node, count, args);
        }
    } catch (PriorityWaitException ex) {
        // 只有在 DefaultController 中才会有机会抛出该异常
        node.increaseThreadNum();
        if (context.getCurEntry().getOriginNode() != null) {
            // Add count for origin node.
            context.getCurEntry().getOriginNode().increaseThreadNum();
        }

        if (resourceWrapper.getEntryType() == EntryType.IN) {
            // Add count for global inbound entry node for global statistics.
            Constants.ENTRY_NODE.increaseThreadNum();
        }
        // Handle pass event with registered entry callback handlers.
        for (ProcessorSlotEntryCallback<DefaultNode> handler : StatisticSlotCallbackRegistry.getEntryCallbacks()) {
            handler.onPass(context, resourceWrapper, node, count, args);
        }
    } catch (BlockException e) {
        // Blocked, set block exception to current entry.
        context.getCurEntry().setBlockError(e);

        // Add block count.
        node.increaseBlockQps(count);
        if (context.getCurEntry().getOriginNode() != null) {
            context.getCurEntry().getOriginNode().increaseBlockQps(count);
        }

        if (resourceWrapper.getEntryType() == EntryType.IN) {
            // Add count for global inbound entry node for global statistics.
            Constants.ENTRY_NODE.increaseBlockQps(count);
        }

        // Handle block event with registered entry callback handlers.
        for (ProcessorSlotEntryCallback<DefaultNode> handler : StatisticSlotCallbackRegistry.getEntryCallbacks()) {
            handler.onBlocked(e, context, resourceWrapper, node, count, args);
        }

        throw e;
    } catch (Throwable e) {
        // Unexpected internal error, set error to current entry.
        context.getCurEntry().setError(e);

        throw e;
    }
}
```

我们先来看 entry 方法：

1. 先直接向后传递，先看看规则判断部分的 slot 中，经过检查之后，是否会抛出相应的异常。
2. 如果检测通过
   1. 增加对应 DefaultNode 的线程数和请求通过数。不过同时也会对资源相对应的 ClusterNode 进行相同的统计数据增加。
   2. 对于整个应用来说，也就是我们之前通过 ContextUtil 配置的 origin，它也会存在一个 originNode，也对它进行相应的统计。不过如果是默认的，也就是 “”，将不会存在 originNode。
   3. 同时还会判断当前资源是否是入口流量，如果是，就使用一个关于系统全局的 `ENTRY_NODE`，也就是一个 ClusterNode，去统计相应的数据，该 node 的主要作用是留着给 `SystemRule` 进行检查。
3. 如果抛出了 `PriorityWaitException` 异常，我们也算该请求是通过的，同样进行相应的数据统计，但是不对上面第二点所操作的节点进行 `addPassRequest` 操作。
4. 如果发现是 `BlockException` 异常，就添加 blockQps、setBlockError等等。

> 什么是入口流量：一般而言就代表我们的接口对外提供服务，那么通常就是控制入口流量。如果我们在用户服务中使用 getOrderInfo 方法，而该方法会去调用订单服务，那么对于该方法来说，压力都在订单服务当中，我们就可以指定它为出口流量。

对于整个 `StatisticSlot` 的基本流程而言，就如上面分析的那样，不过还是有几个细节的地方需要进一步分析：

1. 可以看到，当规则校验通过的时候，会有 `ProcessorSlotEntryCallback` 的实现类去执行相应的 `onPass` 方法，而如果失败了也会去执行 `onBlocked` 方法，那么这些操作的作用究竟是什么呢？
2. 我们在进行数据统计的时候，它的详细过程究竟是怎么样的呢？滑动窗口算法的具体实现是什么呢？

## 第二章 关于规则检验之后通过或失败的回调函数

我们将目光放在下列代码上：

```java
for (ProcessorSlotEntryCallback<DefaultNode> handler : StatisticSlotCallbackRegistry.getEntryCallbacks()) {
    handler.onPass(context, resourceWrapper, node, count, args);
}
```

可以看到我们是使用 `StatisticSlotCallbackRegistry` 去获取我们的回调，而它的内部又是使用 `Map<String, ProcessorSlotEntryCallback<DefaultNode>> entryCallbackMap` 去存放我们的回调，于是我们通过查看它的 add 方法，可以发现，我们可以使用两个类去向该注册中心添加相应的回调：`MetricCallbackInit` 和 `ParamFlowStatisticSlotCallbackInit`，其中后者是需要额外引入一个依赖，也就是热点规则判断，我们不进行分析；而前者就是我们所熟悉的了，在分析 Sentinel 和控制台进行交互的时候，除了 CommandCenterInitFunc 和 心跳检测之外，还引入了一个 InitFunc，就是我们的 `MetricCallbackInit`。

```java
@Override
public void init() throws Exception {
    StatisticSlotCallbackRegistry.addEntryCallback(MetricEntryCallback.class.getCanonicalName(),
                                                   new MetricEntryCallback());
    StatisticSlotCallbackRegistry.addExitCallback(MetricExitCallback.class.getCanonicalName(),
                                                  new MetricExitCallback());
}
```

可以看到，就是该 InitFunc 引入了 `MetricEntryCallback`。

然而，从实际调试结果来看，该 MetricEntryCallback 似乎只是提供给我们的一个扩展点，并没有什么实质的作用，当然，可能 Sentinel 在和其他的组件在适配过程中添加了新的作用也不一定。不过我们也暂且知道了 `MetricCallbackInit` 的作用是什么样的了。

## 第三章 滑动窗口算法的体现

我们就以 `DefaultNode` 为例，在它新增通过的请求数量的时候，调用的其实是父类的方法，也就是 `StatisticNode` ，如下所示：

```java
@Override
public void addPassRequest(int count) {
    rollingCounterInSecond.addPass(count);
    rollingCounterInMinute.addPass(count);
}
```

至于这两个类究竟是什么呢？

```java
// 存储最近 1000 ms 的统计数据，并将 1000 ms 划分成 2 部分
private transient volatile Metric rollingCounterInSecond = new ArrayMetric(SampleCountProperty.SAMPLE_COUNT, IntervalProperty.INTERVAL);

// 存储最近一分钟的统计数据，然后该 window 被划分成 60 个 bucket，
// 这也就意味着我们可以准确的获取每秒的统计数据
private transient Metric rollingCounterInMinute = new ArrayMetric(60, 60 * 1000, false);
```

可以看到，它们其实是 `ArrayMetric` 实例对象，我们查看它的 addPass 方法：

```java
@Override
public void addPass(int count) {
    // 获取当前属于的 bucket，并将其 value 增加 count 值
    // WindowWrap：包装类，包装了 MetricBucket，也就是时间窗口的内容，以及 window 的时间长度，还有该 window 的起始时间
    WindowWrap<MetricBucket> wrap = data.currentWindow();
    wrap.value().addPass(count);
}
```

其实 ArrayMetric 是没有相关滑动窗口的能力的，它的核心能力来源与 data 实例变量，也就是 `LeapArray<MetricBucket>`。现在我们知道了，所谓的滑动时间窗口算法，其实就是一个 LeapArray，你可以将其想象成一个环形数组。举个例子，当我们只需要对最近一分钟的数据进行限流，于是我们就需要保留该请求一分钟之前的统计信息，因此该 LeapArray 的总长度应该是一分钟。但是为了方便统计（具体的原因在第一篇博客介绍传统的限流算法的时候有介绍到），就需要将一分钟给划分成多个小格，我们这里是 60 个 bucket，也就是说，LeapArray 数组的每一个空位的长度应该是一秒钟，这其实就是我们的 `MetricBucket`。

**在使用过程中，就可以根据请求的当前时间戳，计算出它在 LeapArray 中的具体的位置，然后取出对应的 MetricBucket，对其进行数据的统计。**

然而，我们上述分析的都是基于分钟的，也就是我们有六十个小格，所造成的误差其实是非常小的。但是，**对于 `rollingCounterInSecond` 来说，我们只会存储最近的 1000 ms 的数据，并且只会将其划分成两部分，这样的误差其实就非常大**。 

基于这样的疑问，我们来看 `ArrayMetric` 的构造函数，也就是初始化 LeapArray 的地方，可以发现，它提供了下列这两个构造函数：

```java
public ArrayMetric(int sampleCount, int intervalInMs) {
    // rollingCounterInSecond 使用的
    // 如果对于秒级别的维度统计数据采用 BucketLeapArray 的话，会有较大的误差
    // 因为对于秒级别的来说，array 中只有两个 bucket，精度太低
    // 因而需要使用 OccupiableBucketLeapArray
    this.data = new OccupiableBucketLeapArray(sampleCount, intervalInMs);
}

public ArrayMetric(int sampleCount, int intervalInMs, boolean enableOccupy) {
    if (enableOccupy) {
        this.data = new OccupiableBucketLeapArray(sampleCount, intervalInMs);
    } else {
        // rollingCounterInMinute 使用的，enableOccupy 为 false
        this.data = new BucketLeapArray(sampleCount, intervalInMs);
    }
}
```

现在我们知道了，对于秒级别的维度来说，使用的是 `OccupiableBucketLeapArray`，而对于分钟级别的来说，使用的就是 `BucketLeapArray`。正是 LeapArray 的实现类不同，正是由于对于秒级维度的统计采用特殊的处理，才能够减少它的误差。不过我们这里只是去分析分钟维度的滑动窗口的做法，也就是分析 `BucketLeapArray`。

于是我们来看 LeapArray 的 `currentWindow` 方法：

```java
public WindowWrap<T> currentWindow(long timeMillis) {
    if (timeMillis < 0) {
        return null;
    }
    // 获得当前请求的时间 在 array 中的索引位置
    // long timeId = timeMillis / windowLengthInMs; 
    // return (int)(timeId % array.length());
    int idx = calculateTimeIdx(timeMillis);
    
    // Calculate current bucket start time.
    // 计算当前的 bucket 的开始的时间
    // return timeMillis - timeMillis % windowLengthInMs;
    long windowStart = calculateWindowStart(timeMillis); 

    /*
         * Get bucket item at given time from the array.
         *
         * (1) Bucket is absent, then just create a new bucket and CAS update to circular array.
         * (2) Bucket is up-to-date, then just return the bucket.
         * (3) Bucket is deprecated, then reset current bucket and clean all deprecated buckets.
         */
    while (true) {
        // 根据索引 idx，在采用窗口数组中取得一个时间窗口 old
        WindowWrap<T> old = array.get(idx);
        if (old == null) {
            /*
                 *     B0       B1      B2    NULL      B4
                 * ||_______|_______|_______|_______|_______||___
                 * 200     400     600     800     1000    1200  timestamp
                 *                             ^
                 *                          time=888
                 *            bucket is empty, so create new and update
                 *
                 * If the old bucket is absent, then we create a new bucket at {@code windowStart},
                 * then try to update circular array via a CAS operation. Only one thread can
                 * succeed to update, while other threads yield its time slice.
                 */
            WindowWrap<T> window = new WindowWrap<T>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
            if (array.compareAndSet(idx, null, window)) {
                // Successfully updated, return the created bucket.
                return window;
            } else {
                // Contention failed, the thread will yield its time slice to wait for bucket available.
                Thread.yield();
            }
        } else if (windowStart == old.windowStart()) {
            /*
                 *     B0       B1      B2     B3      B4
                 * ||_______|_______|_______|_______|_______||___
                 * 200     400     600     800     1000    1200  timestamp
                 *                             ^
                 *                          time=888
                 *            startTime of Bucket 3: 800, so it's up-to-date
                 *
                 * If current {@code windowStart} is equal to the start timestamp of old bucket,
                 * that means the time is within the bucket, so directly return the bucket.
                 */
            return old;
        } else if (windowStart > old.windowStart()) {
            /*
                 *   (old)
                 *             B0       B1      B2    NULL      B4
                 * |_______||_______|_______|_______|_______|_______||___
                 * ...    1200     1400    1600    1800    2000    2200  timestamp
                 *                              ^
                 *                           time=1676
                 *          startTime of Bucket 2: 400, deprecated, should be reset
                 *
                 * If the start timestamp of old bucket is behind provided time, that means
                 * the bucket is deprecated. We have to reset the bucket to current {@code windowStart}.
                 * Note that the reset and clean-up operations are hard to be atomic,
                 * so we need a update lock to guarantee the correctness of bucket update.
                 *
                 * The update lock is conditional (tiny scope) and will take effect only when
                 * bucket is deprecated, so in most cases it won't lead to performance loss.
                 */
            if (updateLock.tryLock()) {
                try {
                    // Successfully get the update lock, now we reset the bucket.
                    return resetWindowTo(old, windowStart);
                } finally {
                    updateLock.unlock();
                }
            } else {
                // Contention failed, the thread will yield its time slice to wait for bucket available.
                Thread.yield();
            }
        } else if (windowStart < old.windowStart()) {
            // Should not go through here, as the provided time is already behind.
            return new WindowWrap<T>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
        }
    }
}
```

1. 先根据当前请求的时间计算出它在 array 中的索引位置，并计算出当前的 bucket 的开始时间，然后取得该位置的之前的窗口 old。
2. 如果该 old 窗口根本就不存在，就只需要创建一个新的窗口给放到该位置上。
3. 如果该 old 窗口的开始时间和我们的请求计算出来的开始时间相同，就直接返回，因为该 old 就是我们需要找的窗口。
4. 如果发现 old 过期了，就将其替换成一个新的就行。

最后在说一下 `MetricBucket` 是如何存储数据的：

```java
public MetricBucket() {
    // 将所有的 Metric 采用 LongAdder 进行计数
    MetricEvent[] events = MetricEvent.values();
    this.counters = new LongAdder[events.length];
    for (MetricEvent event : events) {
        counters[event.ordinal()] = new LongAdder();
    }
    // Get the max RT value that Sentinel could accept for system BBR strategy.
    // 默认是 5000
    initMinRt();
}
```

可以看到，其实它的内部就是通过 LongAdder 去进行计数

到这里就把如何使用滑动窗口统计数据就讲明白了，之后的一些流控规则的操作就是根据这里面的数据进行统计分析了。
