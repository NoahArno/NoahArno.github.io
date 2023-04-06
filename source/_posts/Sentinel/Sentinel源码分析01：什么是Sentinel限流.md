---
title: Sentinel源码分析01：什么是Sentinel限流
category_bar: true
date: 2023-04-06 23:38:20
tags: [Sentinel]
categories:
  - Sentinel
---

## 第一章 什么是限流

在实际场景中，用户在一定时间内的请求数量是不断变化的，例如我们常见的秒杀常见，可能会存在瞬间的大流量让整个系统都崩溃掉；或者有时候为了防止大量的恶意请求影响系统的稳定性，限制用户的请求流量很重要。然而，限制流量不可避免的会让用户的请求变慢，甚至有时候会直接拒绝掉用户的请求，从而给用户造成体验上的不友好。因此做好选择合适的限流算法，控制限流的平衡性非常重要。

### 1.1 计数器限流算法

这是最简单的限流算法，当系统并不需要严格的限流的时候可以采用该方法。假设该系统仅仅只能同时处理M个请求，就可以在系统中维护一个 AtomicInteger 类型的计数器，每当一个请求进来的时候，就可以将该计数器的值自增。每当一个请求完成的时候就可以将该计数器的值自减。当计数器的值达到系统同时处理请求数量的阈值的时候，就拒绝接下来的请求。

```java
AtomicInteger counter = new AtomicInteger(); // 计数器
Integer threshold = 100; // 阈值

boolean tryAcquire() {
    if (counter.intValue() <= threshold) {
        counter.incrementAndGet();
        return true;
    }
    return false;
}

boolean tryRelease() {
    if (counter.intValue() > 0) {
        counter.decrementAndGet();
        return true;
    }
    return false;
}
```

该限流算法的优点就是足够简单，在不严格要求限流的系统中便于使用，易于维护。然而该算法并没有办法处理突发的流量，比如如果 counter 为 0，并且 threshold 为一个很大的数，那么当系统一瞬间涌入大量流量的时候，该算法并没有起到任何的限制作用。

### 1.2 滑动窗口限流算法

在说滑动窗口之前，先说说固定窗口限流算法。

所谓固定窗口限流算法，其实就是将时间以某个单位进行划分，比如每一秒钟算作一个窗口，如果在单位窗口内的访问次数超过了限制就拒绝接下来的请求。不过有的人会认为固定窗口限流算法和计数限流算法可以划等号，只不过一个是说的单位时间，一个说的是整体的阈值。

```java
Integer counter = 0;
final Integer PERMITS = 100; // 每秒请求限制数
long timestamp = System.currentTimeMillis(); // 上一个窗口的开始时间。

boolean tryAcquire() {
    long cur = System.currentTimeMillis();
    if (cur - timestamp < 1000) {
        // 说明当前请求在该窗口内
        if (counter < PERMITS) {
            counter++;
            return true;
        }
        return false;
    } // 窗口过期了，进行数据的重置，也就是进入下一个窗口
    counter = 0;
    timestamp = cur;
    return true;
}
```

然而固定窗口限流虽然实现简单，但是它还是存在的问题却显而易见：如果前一秒的请求都集中在后半段，下一秒的请求都集中在前半段，就会造成该时间段的流量过大。因此就需要滑动窗口限流算法。

在滑动窗口限流算法中，如果请求在 m 时间到达，我们就可以统计 m 之前一秒内系统所接受的请求数量，然后判断该数量是否超过阈值，不过如果按照这样，对于每次请求都往前推一段时间，然后进行统计，这就导致我们需要统计的数据量过大，并且很多的数据重复统计，浪费资源。

因此就可以将上述的固定窗口划分成多个小格子，每次都移动一小格，而不是固定窗口的长度。该算法解决了前面两个算法的瞬间流量问题，同时，当窗口的划分的粒度越细的时候，对流量的控制将会更加的精准和严格。

然而，滑动限流算法对于超过阈值的请求的处理方法太过强硬了。在某些应用当中，对于超出限制的请求不采用直接拒绝的方式，而是将其放到队列或者其他地方进行平滑处理。

### 1.3 漏桶算法

漏桶算法，可以联想出一个漏桶，水进入漏桶的速度是不受控制的，而出水的速度可以认为是恒定不变的。一旦漏桶的水满了，接下来的多余的水就只能溢出，这就对应于系统承受不了的请求直接给抛弃掉。

该算法相对于滑动窗口算法来说，最大的变化是拥有了一个用于过渡的水槽，针对于处理不了的请求，可以放在水槽中进行一个缓冲，等到系统能够处理过来的时候再慢慢处理。这样的思想倒是有一点像消息队列。

同时，该算法的一个重点在于漏桶的水的流出速度是恒定的，在实现层面上，可以配合队列实现，也就是一旦判断该请求在漏桶阈值之内，就将其添加到队列中，然后只需要开启一个定时任务，周期性的从队列中拿出请求任务进行消耗。

漏桶算法存在的主要目的是用来平滑突发的流量，提供一种机制来确保网络中的突发流量被整合成平滑稳定的流量。然而对于漏桶的流出速度的控制需要严格一点，如果速度过低就无法充分使用系统资源，如果速度过快就会导致系统的负载过大从而形成恶行循环。

```java
private long rate; // 每秒处理数（出水率）
private long currentWater; // 当前剩余水量
private long refreshTime; // 后刷新时间
private long capacity; // 桶容量

boolean tryAcquire() {
    long currentTime = System.currentTimeMillis();  //获取系统当前时间
    long outWater = (currentTime - refreshTime) / 1000 * rate; //流出的水量 =(当前时间-上次刷新时间)* 出水率
    long currentWater = Math.max(0, currentWater - outWater); // 当前水量 = 之前的桶内水量-流出的水量
    refreshTime = currentTime; // 刷新时间

    // 当前剩余水量还是小于桶的容量，则请求放行
    if (currentWater < capacity) {
        currentWater++;
        return true;
    }
    // 当前剩余水量大于等于桶的容量，限流
    return false;
}
// 该示例代码不完整，还需将请求放入队列中进行一定速率的消费
```

### 1.4 令牌桶算法

如何在限制流量速率的情况下，又能允许突发流量呢？令牌桶算法以恒定的速率向桶中分发令牌，而对于到来的请求，会先去桶中查看能否拿到令牌，如果能拿到就执行，如果拿不到就拒绝。同时，如果桶中的令牌满了，新来的令牌也不再放入桶中。

```java
boolean tryAcquire() {
    long now = System.currentTimeMillis();
    long generatedToken = (now - lastAcquireTime) * rate; // 当前时间减去上次取令牌的时间 * 令牌生成的速率
    long leftToken = min(capacity, leftToken + generatedToken);
    if (leftToken >= 1) {
        lastAcquireTime = now;
        leftToken--;
        return true;
    }
    return false;
}
```

若是存在突发流量，桶中剩余的令牌足以应付。

### 1.5 小结

针对于上述的几种限流算法，其实并不存在哪一种算法是绝对完整的，因此还是得考虑实际的应用场景选择合适的限流算法。例如漏桶算法其实不适用于要求低时延的系统。

同时对于单机的限流和分布式的限流，其实将限流的一些变量存放在不同的位置上。针对于令牌桶算法而言，其实可以每次从 Redis 中拿出多个令牌放到对应的机器内存中，这样可以减少系统和 Redis 之间的交互次数，提升性能。

对于限流算法中的阈值的确定其实也是一个难点，yes 大佬博客中所说的一个办法如下：

> 限流上线之后先预估个大概的阈值，然后不执行真正的限流操作，而是采取日志记录方式，对日志进行分析查看限流的效果，然后调整阈值，推算出集群总的处理能力，和每台机子的处理能力(方便扩缩容)。
>
> 然后将线上的流量进行重放，测试真正的限流效果，最终阈值确定，然后上线。

`Google Guava` 提供的限流工具类 `RateLimiter`，是基于令牌桶实现的，并且扩展了算法，支持预热功能。

阿里开源的限流框架`Sentinel` 中的匀速排队限流策略，就采用了漏桶算法。

## 第二章 初识 Sentinel

### 2.1 基本的认识和使用

Sentinel 是面向分布式、多语言异构化服务架构的流量治理组件，主要以流量为切入点，从流量路由、流量控制、流量整形、熔断降级、系统自适应过载保护、热点流量防护等多个维度来帮助开发者保障微服务的稳定性。

在 Sentinel 中，资源是它的关键概念，只要通过 Sentinel 的 API 定义的代码，都可以被称作资源，能够被 Sentinel 给保护起来。

如果只是想简单使用 Sentinel 的核心功能，而不需要和其他的框架，例如 SpringBoot 进行整合，就可以引入 sentinel-core 依赖，然后使用 Sentinel 提供的 API 对想要进行流控的代码进行保护，使其被定义为资源。

在使用方面，一般主要分为如下三个步骤：

1. 定义资源
2. 定义规则
3. 检验规则是否生效

```java
ContextUtil.enter("contextName", "origin");
Entry entry = null;
try {
    entry = SphU.entry("resourceName");
    // 业务代码
} catch (BlockException e) {
    // 进行限流的相关处理
} catch (Throwable ex) {
    // 进行其他的一些异常处理
} finally {
    if (entry != null) {
        entry.exit(1);
    }
}
```

当然，在使用上述代码之前，可以使用纯 API 代码进行流控规则的设置。

然而在大部分的使用场景中，都会将其与 SpringBoot 进行适配，只使用一个注解就能完成上述的功能，降低侵入性，在 Sentinel 中，使用的就是 @SentinelResource 注解，具体的注解原理在以后的章节进行详细介绍。

同时，可以使用官方给定的 dashboard 控制台进行可视化的资源设定，可以随时更改和实时监控，在使用的复杂度方面也大大降低。然而，官方给定的 dashboard 控制台将所有的配置都存放在内存中，这在实际生产环境中肯定是不科学的，因此还需要规则配置的持久化。在实际中，一般都需要根据 Sentinel 和它提供的控制台进行二次开发。

可以通过实现 `DataSource` 接口的方式，来自定义规则的存储数据源。

1. 下载控制台相关 jar 包。

2. 在客户端引入相应的依赖

   ```xml
   <dependency>
       <groupId>com.alibaba.csp</groupId>
       <artifactId>sentinel-transport-simple-http</artifactId>
       <version>1.8.6</version>
   </dependency>
   ```

3. 启动的时候添加 JVM 参数指定控制台地址和端口。

   ```properties
   -Dcsp.sentinel.dashboard.server=consoleIp:port
   ```

4. 确保客户端有访问量，只有第一次访问的时候两者之间才会建立链接。

### 2.2 Sentinel 的基本流程

在 Sentinel 中，最重要的可能就属 ProcessorSlotChain 了，它是 Sentinel 的核心骨架，使用责任链模式将不同的 Slot 按照指定的顺序串在一起，从而将不同的功能，比如限流、降级、系统保护等等，组合在一起。从总体上来说，slot chain 可以分为两部分：统计数据构建部分，以及规则判断部分，如下图所示：

![sentinel-slot-chain](Sentinel源码分析01：什么是Sentinel限流/sentinel-slot-chain-architecture.png)

当然，上述图中的规则判断部分的顺序可能和代码中的不太一样，具体的顺序需要按照实际源码进行分析和判断。同时，Sentinel 还提供了相应的 SPI 机制保留了对该规则判断部分的自定义扩展。

Sentinel 将 `ProcessorSlot` 作为 SPI 接口进行扩展（1.7.2 版本以前 `SlotChainBuilder` 作为 SPI），使得 Slot Chain 具备了扩展的能力。您可以自行加入自定义的 slot 并编排 slot 间的顺序，从而可以给 Sentinel 添加自定义的功能。

```java
public static final int ORDER_NODE_SELECTOR_SLOT = -10000;
public static final int ORDER_CLUSTER_BUILDER_SLOT = -9000;
public static final int ORDER_LOG_SLOT = -8000;
public static final int ORDER_STATISTIC_SLOT = -7000;
public static final int ORDER_AUTHORITY_SLOT = -6000;
public static final int ORDER_SYSTEM_SLOT = -5000;
public static final int ORDER_FLOW_SLOT = -2000;
public static final int ORDER_DEGRADE_SLOT = -1000;
```

在接下来的章节中，笔者将会分析 Sentinel 客户端如何和 Dashboard 进行交互，并且按照 Sentinel 的基本流程进行逐个分析。

## 第三章 如何在生产中使用Sentinel？

Sentinel 毕竟是开源版本的，它的有些特殊的功能肯定无法给放出来开源，然而，Sentinel 本身是一个非常优秀的组件，而想要使用好 Sentinel，就需要对其进行二次开发和扩展， 使其符合我们的业务需求。

但是，笔者作为一个学生，是断断没有资格和眼界去扩展 Sentinel 的，因为我也不知道在实际的企业中，究竟会遇到什么样的困难，但是，这里有一篇博客说的很不错，对当前 Sentinel 的缺点进行了指出，并结合实际环境提出了一些改进意见。

[阿里巴巴开源限流降级神器Sentinel大规模生产级应用实践 (qq.com)](https://mp.weixin.qq.com/s/AjHCUmygTr78yo9yMxMEyg)

获取以后工作之后，有时间会尝试着二次开发 Sentinel 吧。

## 参考文献

[阿里云二面：你对限流了解多少？](https://mp.weixin.qq.com/s?__biz=MzkxNTE3NjQ3MA==&mid=2247488795&idx=1&sn=7cc3377f2b6a3acf46c097cfb4213f1f&scene=21#wechat_redirect)

[面试必备：4种经典限流算法讲解](https://z.itpub.net/article/detail/B049B6F216829EDD0827E97BC1AA9100)

[introduction | Sentinel (sentinelguard.io)](https://sentinelguard.io/zh-cn/docs/introduction.html)
