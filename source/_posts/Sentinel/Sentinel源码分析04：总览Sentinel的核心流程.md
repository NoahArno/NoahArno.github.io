---
title: Sentinel源码分析04：总览Sentinel的核心流程
category_bar: true
date: 2023-04-06 23:40:22
tags: [Sentinel]
categories:
  - Sentinel
---

# 总览 Sentinel 的核心流程

在分析完 `@SentinelResource` 和 Sentinel 如何和控制台进行交互之后，本篇文章开始分析 Sentinel 的核心流程。

Sentinel 里面的各种种类的统计节点：

- `StatisticNode`：最为基础的统计节点，包含秒级和分钟级两个滑动窗口结构。
- `DefaultNode`：链路节点，用于统计调用链路上某个资源的数据，维持树状结构。
- `ClusterNode`：簇点，用于统计每个资源全局的数据（不区分调用链路），以及存放该资源的按来源区分的调用数据（类型为 `StatisticNode`）。特别地，`Constants.ENTRY_NODE` 节点用于统计全局的入口资源数据。
- `EntranceNode`：入口节点，特殊的链路节点，对应某个 Context 入口的所有调用数据。`Constants.ROOT` 节点也是入口节点。

构建的时机：

- `EntranceNode` 在 `ContextUtil.enter(xxx)` 的时候就创建了，然后塞到 Context 里面。
- `NodeSelectorSlot`：根据 context 创建 `DefaultNode`，然后 set curNode to context。
- `ClusterBuilderSlot`：首先根据 resourceName 创建 `ClusterNode`，并且 set clusterNode to defaultNode；然后再根据 origin 创建来源节点（类型为 `StatisticNode`），并且 set originNode to curEntry。

对于整体流程来说，可以大致概括为如下四个步骤：

1. 构建 Context
2. 为每个资源构建自己的 slotChain
3. 针对该 slotChain 进行信息统计和规则检验
4. 资源调用结束时候的 exit 操作详情

## 第一章 什么是 Context？

在官网中，是这么解释 Context 的：

> Context 代表调用链路上下文，贯穿一次调用链路中的所有 Entry。Context 维持着入口节点（entranceNode）、本次调用链路的 curNode、调用来源（origin）等信息。Context 名称即为调用链路入口名称。
>
> Context 维持的方式：通过 ThreadLocal 传递，只有在入口 enter 的时候生效。由于 Context 是通过 ThreadLocal 传递的，因此对于异步调用链路，线程切换的时候会丢掉 Context，因此需要手动通过 `ContextUtil.runOnContext(context, f)` 来变换 Context。

正如官网所说，Context 只会在该线程第一次调用的时候才会初始化，就算线程后续想换个 Context 也无法成功。

如前面博客分析的那样，Env 会做一些初始化操作，比如 `CommandHandler` 和 心跳检测，而我们会使用 `SphU` 对资源进行创建，然而，真正的对资源进行操作的类是 `CtSph`。

```java
private Entry entryWithPriority(ResourceWrapper resourceWrapper, int count, boolean prioritized, Object... args)
    throws BlockException {
    /*
      Context 代表调用链路上下文，贯穿一次调用链路中的所有 Entry。
      Context 维持着入口节点也就是 entranceNode、本次调用链路正在处理的 curNode、调用来源 origin 等信息。
      Context 名称即称为调用链路入口名称

      Context 维持的方式：通过 ThreadLocal 进行传递，只有在入口 enter 的时候才生效 。
      当使用异步调用链路，线程切换之间会丢掉 Context，因此需要手动使用 ContextUtil.runOnContext(context, f) 来交换 context

      Context 只会在该线程第一次调用的时候才会初始化，就算该线程后续想换个 Context，也是不成功的。
    * */
    Context context = ContextUtil.getContext();
    if (context instanceof NullContext) {
        // The {@link NullContext} indicates that the amount of context has exceeded the threshold,
        // so here init the entry only. No rule checking will be done.
        // 表示 Context 的数量超过了阈值，所以仅仅只是初始化该 entry，而不做任何规则的检查
        // 然而我们在使用中默认都只会创建同一个 context，也就是 sentinel_default_context
        // 但是不清楚实际企业环境中，会不会有变化。
        return new CtEntry(resourceWrapper, null, context);
    }

    if (context == null) {
        // Using default context.
        // 使用默认的，即 sentinel_default_context，origin 为 “”
        // 笔者个人认为如果手动使用 ContextUtil.enter(name, origin) 进行设置的作用不大
        // 向 sentinel 官方的 annotation 中，其实也使用的是默认的
        context = InternalContextUtil.internalEnter(Constants.CONTEXT_DEFAULT_NAME);
    }

    // Global switch is close, no rule checking will do.
    if (!Constants.ON) {
        return new CtEntry(resourceWrapper, null, context);
    }

    // 构造 slot chain，每个资源都有自己的一个 slot chain，无论该资源在哪个 Context 中
    ProcessorSlot<Object> chain = lookProcessChain(resourceWrapper);

    /*
     * Means amount of resources (slot chain) exceeds {@link Constants.MAX_SLOT_CHAIN_SIZE},
     * so no rule checking will be done.
     */
    if (chain == null) {
        return new CtEntry(resourceWrapper, null, context);
    }

    Entry e = new CtEntry(resourceWrapper, chain, context);
    try {
        // 开始按照构造好的 chain 进行规则检查
        chain.entry(context, resourceWrapper, null, count, prioritized, args);
    } catch (BlockException e1) {
        e.exit(count, args);
        throw e1;
    } catch (Throwable e1) {
        // This should not happen, unless there are errors existing in Sentinel internal.
        RecordLog.info("Sentinel unexpected exception", e1);
    }
    return e;
}
```

当线程第一次调用的时候，Context 肯定是 null 的，因此就需要去创建 Context，然而，**每个 contextName 其实都关联着同一个 EntranceNode**，因此就需要根据一个 `contextNameNodeMap` 去判断以当前 contextName 是否被创建过，如果没有创建过，就以 contextName 为 key，以 `new EntranceNode(new StringResourceWrapper(name, EntryType.IN), null)` 为 value 存放到 `contextNameNodeMap` 中。具体的代码在 `InternalContextUtil.internalEnter(Constants.CONTEXT_DEFAULT_NAME)` 中。

> EntranceNode：入口节点，特殊的链路节点，对应某个 Context 入口的所有调用数据。`Constants.ROOT` 节点也是入口节点。

```java
protected static Context trueEnter(String name, String origin) {
    Context context = contextHolder.get();
    if (context == null) {
        Map<String, DefaultNode> localCacheNameMap = contextNameNodeMap;
        DefaultNode node = localCacheNameMap.get(name);
        if (node == null) {
            if (localCacheNameMap.size() > Constants.MAX_CONTEXT_NAME_SIZE) {
                setNullContext();
                return NULL_CONTEXT;
            } else {
                LOCK.lock();
                try {
                    node = contextNameNodeMap.get(name);
                    if (node == null) {
                        if (contextNameNodeMap.size() > Constants.MAX_CONTEXT_NAME_SIZE) {
                            setNullContext();
                            return NULL_CONTEXT;
                        } else {
                            node = new EntranceNode(new StringResourceWrapper(name, EntryType.IN), null);
                            // Add entrance node.
                            Constants.ROOT.addChild(node);

                            Map<String, DefaultNode> newMap = new HashMap<>(contextNameNodeMap.size() + 1);
                            newMap.putAll(contextNameNodeMap);
                            newMap.put(name, node);
                            contextNameNodeMap = newMap;
                        }
                    }
                } finally {
                    LOCK.unlock();
                }
            }
        }
        context = new Context(node, name);
        context.setOrigin(origin);
        contextHolder.set(context);
    }

    return context;
}
```

## 第二章 如何构建 SlotChain？

对于 SlotChain 来说，每个资源都有自己的 Slot Chain，无论该资源在哪个 Context 中。

```java
ProcessorSlot<Object> lookProcessChain(ResourceWrapper resourceWrapper) {
    ProcessorSlotChain chain = chainMap.get(resourceWrapper);
    if (chain == null) {
        synchronized (LOCK) {
            chain = chainMap.get(resourceWrapper);
            if (chain == null) {
                // Entry size limit. 超过了限制
                if (chainMap.size() >= Constants.MAX_SLOT_CHAIN_SIZE) {
                    return null;
                }
                // 进行链的构造
                chain = SlotChainProvider.newSlotChain();
                Map<ResourceWrapper, ProcessorSlotChain> newMap = new HashMap<ResourceWrapper, ProcessorSlotChain>(
                    chainMap.size() + 1);
                newMap.putAll(chainMap);
                newMap.put(resourceWrapper, chain);
                chainMap = newMap;
            }
        }
    }
    return chain;
}
```

上述代码还是比较传统和简单的：先从缓存中拿，如果拿不到，使用 DLC，然后进行一些判断，接着就执行核心逻辑，然后将结果添加到缓存中。

在 `SlotChainProvider.newSlotChain()` 中，会使用 SPI 机制获取 `SlotChainBuilder` 的实现类，然后只选择加载出来的第一个，如果没有就直接使用默认的，也就是 `DefaultSlotChainBuilder`，然后调用它的 `build` 方法。

```java
@Spi(isDefault = true)
public class DefaultSlotChainBuilder implements SlotChainBuilder {

    @Override
    public ProcessorSlotChain build() {
        ProcessorSlotChain chain = new DefaultProcessorSlotChain();

        List<ProcessorSlot> sortedSlotList = SpiLoader.of(ProcessorSlot.class).loadInstanceListSorted();
        for (ProcessorSlot slot : sortedSlotList) {
            if (!(slot instanceof AbstractLinkedProcessorSlot)) {
                RecordLog.warn("The ProcessorSlot(" + slot.getClass().getCanonicalName() + ") is not an instance of AbstractLinkedProcessorSlot, can't be added into ProcessorSlotChain");
                continue;
            }

            chain.addLast((AbstractLinkedProcessorSlot<?>) slot);
        }

        return chain;
    }
}
```

同样的，还是 SPI 机制加上排序，不过加入的插槽必须是 `AbstractLinkedProcessorSlot`  的子类。

在 `Constants` 中，就声明了 slotChain 中的各个 slot 的顺序：

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

当然，还有个关于热点规则的 `ParamFlowSlot`，不过需要引入依赖 `sentinel-parameter-flow-control`。

## 第三章 NodeSelectorSlot

```java
private volatile Map<String, DefaultNode> map = new HashMap<String, DefaultNode>(10);

@Override
public void entry(Context context, ResourceWrapper resourceWrapper, Object obj, int count, boolean prioritized, Object... args)
    throws Throwable {
    /*
     * It's interesting that we use context name rather resource name as the map key.
     *
     * Remember that same resource({@link ResourceWrapper#equals(Object)}) will share
     * the same {@link ProcessorSlotChain} globally, no matter in which context. So if
     * code goes into {@link #entry(Context, ResourceWrapper, DefaultNode, int, Object...)},
     * the resource name must be same but context name may not.
     *
     * If we use {@link com.alibaba.csp.sentinel.SphU#entry(String resource)} to
     * enter same resource in different context, using context name as map key can
     * distinguish the same resource. In this case, multiple {@link DefaultNode}s will be created
     * of the same resource name, for every distinct context (different context name) each.
     *
     * Consider another question. One resource may have multiple {@link DefaultNode},
     * so what is the fastest way to get total statistics of the same resource?
     * The answer is all {@link DefaultNode}s with same resource name share one
     * {@link ClusterNode}. See {@link ClusterBuilderSlot} for detail.
     */
    // 注意这里是使用 context‘s name 去获取 DefaultNode，
    // 是因为对于一个资源来说，它在不同的 context 中会创建不同的 DefaultNode，
    // 然而所有的由同一个资源id所代表的 DefaultNode，都共享着同一个 ClusterNode
    // 如果我们这里使用 resource name 去作为 map 的 key 的话，就会让位于不同 context 中的 资源拥有相同的 DefaultNode
    // 如果 entry 嵌套，比如资源 a 的代码中嵌套了 资源 b，那么 a 和 b 岂不是会使用同一个 DefaultNode？然而经过测试发现，它们之间的 DefaultNode 不同，暂未发现相关代码实现
    // 原来这个 map 不是 static 的，在创建 slotChain 的时候，每个资源都有自己的 chain，也就是每个资源都有自己的 NodeSelectorSLot
    DefaultNode node = map.get(context.getName());
    if (node == null) {
        synchronized (this) {
            node = map.get(context.getName());
            if (node == null) {
                node = new DefaultNode(resourceWrapper, null);
                HashMap<String, DefaultNode> cacheMap = new HashMap<String, DefaultNode>(map.size());
                cacheMap.putAll(map);
                cacheMap.put(context.getName(), node);
                map = cacheMap;
                // Build invocation tree 这里是构建链路数的核心代码
                ((DefaultNode) context.getLastNode()).addChild(node);
            }

        }
    }

    context.setCurNode(node);
    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```

NodeSelectorSlot 作用：负责收集资源的路径，并将这些资源的调用路径，以树状结构存储起来，用于根据调用路径来限流降级；

在代码中，可以看到我们是使用 contextName 为 key 从 map 中获取 DefaultNode，而 DefaultNode 是链路节点，用于统计调用链路上某个资源的数据，维持树状结构。**每一个资源在每一个 Context 中都拥有一个 DefaultNode；而每一个资源都只有一个 ClusterNode，不论它在哪个 Context 中**，但是为什么它这里要使用 contextName 作为 key？

本来我一直对这里很迷糊，甚至一度怀疑是否这里是 bug，然而在调试之后发现，同一个资源在两个不同的 Context 中，的确是两个不同的 DefaultNode，经过再三确定之后，原来这个 map 是实例变量，而不是类变量。又知道每个资源都有自己的 slotChain，因此就意味着都有自己的 NodeSelectorSlot 实例对象，因而，使用 contextName 就能做到同一个资源在不同的链路中拥有不同的 DefaultNode。

之后通过 `fireEntry` 进入下一个 slot，也就是 `ClusterBuilderSlot`。

## 第四章 ClusterBuilderSlot

前面说到，资源是和 `ClusterNode` 一一对应的，它们通过资源的名字建立连接。

此插槽用于构建资源的 `ClusterNode` 以及调用来源节点。`ClusterNode` 保持资源运行统计信息（响应时间、QPS、block 数目、线程数、异常数等）以及原始调用者统计信息列表。

```java
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                  boolean prioritized, Object... args)
    throws Throwable {
    if (clusterNode == null) {
        synchronized (lock) {
            if (clusterNode == null) {
                // Create the cluster node.
                clusterNode = new ClusterNode(resourceWrapper.getName(), resourceWrapper.getResourceType());
                HashMap<ResourceWrapper, ClusterNode> newMap = new HashMap<>(Math.max(clusterNodeMap.size(), 16));
                newMap.putAll(clusterNodeMap);
                newMap.put(node.getId(), clusterNode);

                clusterNodeMap = newMap;
            }
        }
    }
    node.setClusterNode(clusterNode);

    /*
     * if context origin is set, we should get or create a new {@link Node} of
     * the specific origin.
     * 如果是我们自己设置了 origin，也就是 ContextUtil.enter(name, origin)，就也要给该 origin 创建一个 StatisticNode
     * 如果是使用的默认的，就不用管
     */
    if (!"".equals(context.getOrigin())) {
        Node originNode = node.getClusterNode().getOrCreateOriginNode(context.getOrigin());
        context.getCurEntry().setOriginNode(originNode);
    }

    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```

这里只需要注意，如果我们自己设置了 origin，也就是在 `ContextUtil.enter(name, origin)`中进行设置，就需要给该 origin 创建一个 `StatisticNode`，当然，我们这里使用的是默认的 ""，就不用管了。

## 参考文献

[Sentinel 核心类解析 · alibaba/Sentinel Wiki (github.com)](https://github.com/alibaba/Sentinel/wiki/Sentinel-核心类解析)
