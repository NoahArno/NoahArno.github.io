---
title: Disruptor：高性能无锁队列
category_bar: true
date: 2023-04-07 23:00:14
tags: [Disruptor]
categories:
---

# Disruptor：高性能无锁队列

从数据结构上来看，Disruptor 是一个支持生产者消费者模型的高性能无锁队列，它内部利用 RingBuffer 环形数组存储数据，能够让消费者按照它们之间的依赖关系对 event 进行消费。不过现在 Disruptor 的官方更新已经不那么频繁了。一个是因为它的性能已经足够优秀而导致难以变化；二来是它的作用范围只支持单机，在如今分布式的时代已经逐渐用处不大了。然而，它内部的一些实现还是值得我们去借鉴。

## 1. 基本使用

在依赖上，引入 Disruptor 相关依赖：

```xml
<!-- https://mvnrepository.com/artifact/com.lmax/disruptor -->
<dependency>
    <groupId>com.lmax</groupId>
    <artifactId>disruptor</artifactId>
    <version>3.4.4</version>
</dependency>
```

`Disruptor` 的核心是 RingBuffer，甚至来说，我们完全可以不使用 Disruptor 而直接使用 RingBuffer 就能完成所有的操作。不过官方给 RingBuffer 包装了一层 Disruptor，在使用方面还是较之前简便不少。

一般而言，使用 Disruptor 需要如下步骤：

1. 创建 Disruptor 实例对象：指定 RingBuffer 的大小、事件工厂、线程池、生产者策略和等待策略。
2. 根据消费需求，使用 `handlerEventsWith` 或 `handleEventsWithWorkerPool` 增加消费者。
3. 调用 `Disruptor#start` 方法启动 Disruptor。
4. 在启动之后，就可以往 Disruptor 生产数据了。在生产中，一定得根据创建 Disruptor 时指定的生产者策略进行，不能指定单生产者模式下启动多个生产者并行生产。

示例如下所示：

```java
public static void main(String[] args) throws Exception {  
    Disruptor<Trade> disruptor = new Disruptor<Trade>(
        new EventFactory<Trade>() {
            public Trade newInstance() {
                return new Trade();
            }
        },
        1024 * 1024,
        Executors.defaultThreadFactory(),
        ProducerType.SINGLE,
        new BusySpinWaitStrategy());

    // 并行操作
    disruptor.handleEventsWith(new Handler1(), new Handler2(), new Handler3());

    // 多消费者竞争，对于 eventHandler 而言需要实现 WorkHandler 接口
    disruptor.handleEventsWithWorkerPool(new Handler1(), new Handler2());

    // 启动disruptor
    RingBuffer<Trade> ringBuffer = disruptor.start();
    // 生产 event
    long sequence = ringBuffer.next();	//0	
    try {
        Trade event = ringBuffer.get(sequence);
        //3 进行实际的赋值处理
        event.setValue(data.getLong(0));			
    } finally {
        //4 提交发布操作
        ringBuffer.publish(sequence);			
    }
}
```

对于 Disruptor 而言，使用限制其实比较低，对于生产者和消费者的配合较为灵活，得益于它内部设计的 EventHandlerGroup 和 SequencerBarrier 机制。

## 2. Disruptor的实现优点

Disruptor 的性能优越，它在多个方面进行了细致的优化：

- 不使用锁，而是使用的 CAS，并且对于单生产者而言用 long 来保存序号，而多生产者用的是 AtomicLong，因为单生产者不涉及到竞争。不过这里的 AtomicLong 其实是类 AtomicLong，原因是 Sequence 类其实也是用的 Unsafe 保证原子更新。

  > 由于 Java8 中新增了 LongAdder 类，它在多线程竞争下的性能比 AtomicLong 还要高，我们似乎可以用其对 Sequence 进行优化。

- 使用 RingBuffer 设计。对于传统的队列而言，需要维护 tail、head、size 等变量，入队的时候需要对 tail 进行竞争，出队的时候要对 head 进行竞争，同时还得维护 size 来确认队列的 empty or full。而 RingBuffer 只维护序号来保存下一个可用空间，减小竞争。

- 使用额外的变量，以空间换时间的策略来解决伪共享问题。对于 RingBuffer 存放真正数据的数组也进行了缓存行优化。

- RingBuffer 在初始化的时候就对环中的数据进行了预填充，详情见 RingBuffer#fill，解决频繁创建和回收对象的垃圾回收问题。

  ```java
  // 内存预加载机制的实现，也就是在创建的时候，就将环中的数据填充为空对象
  private void fill(EventFactory<E> eventFactory) {
      for (int i = 0; i < bufferSize; i++) {   // 由于数组左边遭受到填充，因此在设置元素的时候，需要跳过
          entries[BUFFER_PAD + i] = eventFactory.newInstance();
      }
  }
  ```

## 3. 缓存行优化最佳实战

在读取数据的时候，CPU 会将内存中的一块连续的数据读取到 CPU 缓存中，该连续的数据被称为缓存块，也就是 CPU Cache Line，一般是 64 字节大小。不过假如存在两个 CPU 核心，并且存在变量 A 和变量 B 位于同一缓存块中。当 CPU1 想要读取 A 的时候，会将 AB 一起读取到它的缓存当中，同时 CPU2 想要读取 B，也会进行相同的操作。这时候 CPU1 和 CPU2 各自的缓存行处于共享状态。

此时 CPU1 想要更新 A，就必须通过总线将 CPU2 对应的缓存行标记为失效。而 CPU2 想要更新 B，发现缓存行被标记为失效了，就得重新从内存中读取，并且更新之前还得将 CPU1 的缓存行给标记失效。

如果两个 CPU 核心反复执行更新操作，就会造成缓存失效，也就是伪共享问题。

### 3.1 对于普通变量的伪共享解决方案

在 Sequence 类当中，对于变量 value 而言，由于需要频繁更新，消除伪共享就显得额外重要。并且 value 属性是 long 类型，它占 8字节，而一个缓存行一般都是 64 字节大小，因此就需要在 value 的左右两边分别填充 56 字节，也就是分别填充7 个 long 类型变量。

```java
class LhsPadding{
    protected long p1, p2, p3, p4, p5, p6, p7;
}

class Value extends LhsPadding{
    protected volatile long value;
}

class RhsPadding extends Value{
    protected long p9, p10, p11, p12, p13, p14, p15;
}
```

不过在 Disruptor 当中，并没有写一个从来都不会被调用的方法去使用这些额外填充的无用变量。我们不清楚编译器会不会将这些填充变量给优化掉，因此在自己的实现中，还是添加这么一个方法比较保险。

同时 JDK1.8 中提供了 `@sun.misc.Contended` 注解用于将变量自动独占缓存行，不过需要添加 `-XX: -RestrictContended` 参数才能够生效。

### 3.2 针对数组的伪共享解决方案

对于变量而言，它的伪共享解决方案其实较为简单，因为变量的内存占用大小一般都是固定的。不过针对于数组而言，它的内存占用并不确定，因此无法极限利用空间，而是可以选择在数组的两侧额外填充 64 字节或者 128 字节。

在 Disruptor 中，对它的处理倒是显得较为复杂，在 `RingBuffer.class` 中：

```java
abstract class RingBufferFields<E> extends RingBufferPad {
    private static final int BUFFER_PAD;
    private static final long REF_ARRAY_BASE;
    private static final int REF_ELEMENT_SHIFT;
    private static final Unsafe UNSAFE = Util.getUnsafe();

    /**
     * 通过 Unsafe 的 arrayBaseOffset 和 arrayIndexScale
     * 分别获取数组首元素的偏移地址 和 单个元素大小因子
     * 后续的相关操作，就可以用这个来确定
     */
    static {
        // 在 JVM 知识当中，对象是真正被分配到堆中，然后用指针引用，这里使用 Object数组来判断对象的大小，
        // 其实说的是数组中指针的大小，就是 4 字节，所以不必纠结这里用的是 Object 而实际的 entries 可以是 Object 的子类
        final int scale = UNSAFE.arrayIndexScale(Object[].class); // 4
        // 如果开启指针压缩，指针就是 4 字节，没压缩就是 8 字节
        if (4 == scale) {
            REF_ELEMENT_SHIFT = 2;
        } else if (8 == scale) {
            REF_ELEMENT_SHIFT = 3;
        } else {
            throw new IllegalStateException("Unknown pointer size");
        }
        // 将数组的前后都填充 128 字节，不过不知道为啥不填充 64 字节
        BUFFER_PAD = 128 / scale; // 32
        // Including the buffer pad in the array base offset
        // 由于我们在数组的前面填充了 128 字节，因此数组的有效元素偏移量就需要往后推
        REF_ARRAY_BASE = UNSAFE.arrayBaseOffset(Object[].class) + (BUFFER_PAD << REF_ELEMENT_SHIFT);
    }
}
```

在创建和填充 entires 数组的时候，代码如下：

```java
this.entries = new Object[sequencer.getBufferSize() + 2 * BUFFER_PAD];
entries[BUFFER_PAD + i] = eventFactory.newInstance();
```

使用 `elementAt` 方法获取元素：

```java
protected final E elementAt(long sequence) { 
    // 位运算，这就需要 entries 的长度大小为 2 的幂次方，
    // sequence & indexMask：将 sequence 进行截断
    return (E) UNSAFE.getObject(entries, REF_ARRAY_BASE + ((sequence & indexMask) << REF_ELEMENT_SHIFT));
}
```

如上所示，它的使用还是较为复杂的，并且还使用了 UnSafe 类。并且它采用数组真实起始地址和元素偏移量的方式获取元素的设计。如果直接使用数组下标倒是会显得更加麻烦，因为数组是存在两侧的填充的，很难通过下标真实的确定位置。

## 4. 生产者原理

Disruptor 支持单生产者和多生产者模式，并且使用 Sequencer 管理 Sequence。并且它们都使用了二阶段提交机制。

对于单生产者而言，也就是 SingleProducerSequencer，使用 long 类型的 nextValue 表示占位，因为它并不涉及到竞争。同时对于表示真正的生产进度的 Sequencer 类型的 cursor 来说，也就是真实填充了数据的指针，当数据在 nextValue 指定的位置填充好之后，就会更新 cursor 的值。

```
// 1. 一开始的状态
1    2   3   4   5   6
     ^
nextValue、cursor

// 2. next 方法去申请1个位置
1    2          3   4   5   6
     ^          ^
   cursor    nextValue

// 3. 调用 publish 发布之后
1    2   3   4   5   6
         ^
  nextValue、cursor
```

对于多生产者而言，也就是 MultiProducerSequencer，为了防止多个线程重复写同一个元素，Disruptor 为每个线程使用 CAS 获取不同的一段数组空间进行操作，只需要在分配元素的时候通过 CAS 判断一下这段空间是否已经被分配出去即可。然而为了防止读取的时候读取到还未写入的元素，就需要一个简单的二阶段提交：它使用了 availableBuffer 辅助数组，当某个位置写成功的时候就会将 availableBuffer 相应的位置标记为写入成功。消费者想要读取元素的时候就会遍历 availableBuffer 来判断元素是否已经就绪。

availableBuffer 中的值默认为 -1。在设置 availableBuffer 的时候，也就是调用 publish 发布的时候，就需要计算出当前 sequence 是处于哪一轮的，从而设置对应的标志位。这样是因为数组是循环使用的，使用固定特殊值作为标记的方法不可取。

多生产者模式下大致步骤如下：

1. 当前生产者先看 RingBuffer 中空间是否充足，如果空间不够不停循环判断。

   > 这里的判断会查看消费者的进度，获取消费进度最小的那个消费者的序号。看当前生产进度是否超过。

2. 如果空间足够就使用 CAS 操作获取指定的空间。

   > 这里注意的是，多生产者模式下的 cursor 变量也会用作占位：`cursor.compareAndSet(current, next)`

3. 获取成功之后就可以开始往里面写入数据了，同时也会去更新 availableBuffer 中的标志位。

## 5. 消费者原理

Disruptor 支持多个消费者对同一个消息进行并行获取有依赖关系的消费，也支持多个消息去竞争消息消费。

- 消费者实现 EventHandler 接口，并且调用 `disruptor.handleEventsWith` 方法添加消费者，多个 handler 就会对同一个消息进行重复消费，也可以结合 then、after 等方法对数据进行多边形计算。**场景上适用于链路计算，和 CompletableFuture 结合线程池的方法相类似**。
- 消费者实现 WorkerHandler 接口，并且调用 `disruptor.handleEventsWithWorkerPool`，将多个 WorkerHandler 的实现类传入该方法，就可以看作有一个叫做 WorkerPool 的消费者，消息会被它内部的多个 workerHandler 给竞争消费，也就是消息无法被重复消费。**场景上适用于传统的生产者消费者模型**。

前面也说了，消费者支持依赖关系，也就是消费者 A 需要等待消费者 B 消费完某个消息之后才能消费。在实现原理上面，其实是通过 SequenceBarrier 内部的 dependentSequence，比如 A 想要消费序号 N 中的消息，它就会去看 B 的消费进度是否超过了 N，如果没有超过就只能等待。

对于 WorkerHandler 模式下的消费者竞争安全性问题，通过 WorkerPool（WorkerPool 中包含了多个 WorkerHandler）中的 workSequence 属性来进行的，也就是多个 WorkerHandler 去通过 CAS 修改该变量，如果能修改成功表示自己获取到了消费该消息的资格，然后进一步消费，也可以理解为一个二阶段。

## 6. 总结

在看完 Disruptor 的源码之后，对于一些细节上的优化有了进一步的了解。在听到 Disruptor 号称高性能无锁队列之后，一开始我还以为它能够在不使用锁和 CAS 的情况下也能保证线程安全性，不过后来就有点小失望——它内部还是需要 CAS 机制。

它在多生产者模式下的二阶段机制较为新颖，将抢占位置和填充数据给分离开来。不过似乎用处不是很大：按照正常的逻辑也是先 CAS 成功再填充数据，也是个二阶段。反倒是它的代码分离导致使用上还复杂了一点，因为需要调用 next 方法获取到可用序号之后，才从 RingBuffer 中获取到序号指定的元素再进行填充。甚至于 Disruptor 官方也提供了 EventTranslator 接口去帮助我们屏蔽掉这些细节，只关注于消息填充。
