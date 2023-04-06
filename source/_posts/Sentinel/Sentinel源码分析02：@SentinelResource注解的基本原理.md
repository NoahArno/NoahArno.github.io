---
title: Sentinel源码分析02：@SentinelResource注解的基本原理
category_bar: true
date: 2023-04-06 23:38:55
tags: [Sentinel]
categories:
  - Sentinel
---

# @SentinelResource的原理分析

在 Sentinel 中，除了在代码中直接使用它提供的 API 进行资源的控制之外，官方为了减少开发的复杂度，对大部分的主流框架进行了适配，包括 Web Servlet、Dubbo、Spring Cloud、gRPC 等等。

为了简化分析，只保留核心流程，笔者只分析如下依赖对应的源代码：

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-annotation-aspectj</artifactId>
</dependency>
```

而有关 @SentinelResource 的源代码如下所示：

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
public @interface SentinelResource {

    String value() default "";

    /**
     * @return the entry type (inbound or outbound), outbound by default
     * // 判断规则是入口流量还是出口流量
     */
    EntryType entryType() default EntryType.OUT;

    /**
     * @return the classification (type) of the resource
     * @since 1.7.0
     */
    int resourceType() default 0;

    String blockHandler() default "";

    Class<?>[] blockHandlerClass() default {};

    String fallback() default "";

    String defaultFallback() default "";

    Class<?>[] fallbackClass() default {};

    Class<? extends Throwable>[] exceptionsToTrace() default {Throwable.class};

    Class<? extends Throwable>[] exceptionsToIgnore() default {};
}

```



## 第一章 核心流程总览

上述依赖包是将 Sentinel 和 Spring 进行整合，于是，对于注解的解析来说，万变不离其宗，都离不开 Spring 的 AOP，因此就如 `SentinelResourceAspect` 类所示：

```java
@Aspect
public class SentinelResourceAspect extends AbstractSentinelAspectSupport {

    @Pointcut("@annotation(com.alibaba.csp.sentinel.annotation.SentinelResource)")
    public void sentinelResourceAnnotationPointcut() {
    }

    @Around("sentinelResourceAnnotationPointcut()")
    public Object invokeResourceWithSentinel(ProceedingJoinPoint pjp) throws Throwable {
        Method originMethod = resolveMethod(pjp);

        SentinelResource annotation = originMethod.getAnnotation(SentinelResource.class);
        if (annotation == null) {
            // Should not go through here.
            throw new IllegalStateException("Wrong state for SentinelResource annotation");
        }
        // 获取资源的名称，如果注解没有指定value，就使用方法所在的类名 + ：+ 方法名 + 参数列表
        // 例如：com.alibaba.csp.sentinel.demo.annotation.aop.service.TestServiceImpl:test()
        String resourceName = getResourceName(annotation.value(), originMethod);
        EntryType entryType = annotation.entryType(); // 获取流量类型 IN/OUT
        int resourceType = annotation.resourceType();
        Entry entry = null;
        try {
            // 定义资源，这里就是走 slotChain 的逻辑了，责任链模式
            entry = SphU.entry(resourceName, resourceType, entryType, pjp.getArgs());
            return pjp.proceed();
        } catch (BlockException ex) {
            // 出现这个异常说明本次请求被 sentinel 给限流了，就需要走限流逻辑
            return handleBlockException(pjp, annotation, ex);
        } catch (Throwable ex) {
            // 如果配置了 exceptionsToIgnore，就进行排查
            Class<? extends Throwable>[] exceptionsToIgnore = annotation.exceptionsToIgnore();
            // The ignore list will be checked first.
            if (exceptionsToIgnore.length > 0 && exceptionBelongsTo(ex, exceptionsToIgnore)) {
                throw ex;
            }
            // 如果没有配置 exceptionsToTrace，默认就是 Throwable
            if (exceptionBelongsTo(ex, annotation.exceptionsToTrace())) {
                // Trace：用来记录除了 BlockException 之外的其他的异常
                traceException(ex); // TODO 暂时不知道 Trace 的作用
                return handleFallback(pjp, annotation, ex); // 进行 fallback 的处理逻辑
            }

            // No fallback function can handle the exception, so throw it out.
            throw ex;
        } finally {
            if (entry != null) {
                entry.exit(1, pjp.getArgs());
            }
        }
    }
}
```

如上述源码所示，Sentinel 定义了关于 @SentinelResource 注解的切面， 并对它进行环绕通知增强。

1. 先解析注解中的相关信息。
2. 使用核心 API `SphU.entry` 定义相关 resourceName 的资源，即根据 slotChain 进行相应的统计信息和资源控制。
3. 如果出现了 BlockException，即该资源被限制，就进行 handleBlockException 相应逻辑。
4. 如果是除了 BlockException 之外的其他的异常，就先看该异常是否可以被忽略，如果不能，先记录该异常（暂时不知道为啥需要使用 Trace 来记录异常），然后进行 fallback 的处理逻辑。
5. 最后，在最后，需要对创建好的 entry 进行 exit 操作，还原一些系统变量资源。

> 更新：这里的 Trace 作用是用来记录异常，在内部的实现中，其实大体就是 `Context.getContext().getCurEntry().setError(e)`，追踪 Entry 的 getError 方法，可以看到它在 StatisticSlot 中进行了相关的处理，具体流程将在后续分析 exit 源码的时候进行详细解析。

## 第二章 #handleBlockException的分析

当我们定义的资源在经过规则检查之后，如果被限流了，就会走到该逻辑：

```java
protected Object handleBlockException(ProceedingJoinPoint pjp, SentinelResource annotation, BlockException ex)
    throws Throwable {
    // 这一步就是根据注解上配置好的相关信息寻找处理 BlockException 的发方法
    // Execute block handler if configured.
    Method blockHandlerMethod = extractBlockHandlerMethod(pjp, annotation.blockHandler(),
                                                          annotation.blockHandlerClass());
    // 去执行对应的 block handler
    if (blockHandlerMethod != null) {
        Object[] originArgs = pjp.getArgs();
        // Construct args.
        Object[] args = Arrays.copyOf(originArgs, originArgs.length + 1);
        args[args.length - 1] = ex;
        return invoke(pjp, blockHandlerMethod, args);
    }
    // 如果找不到这样的 blockHandler 方法，就走 fallback 逻辑
    // 也就是如果只配置了 fallback 而没有配置 blockHandler，如果出现了 BlockException，就会执行 fallback 逻辑
    // If no block handler is present, then go to fallback.
    return handleFallback(pjp, annotation, ex);
}
```

从整体流程来看，该异常处理机制大概就是看我们是否在注解上配置了相应的 BlockExceptionHandler，如下列代码所示：

```java
@Override
@SentinelResource(value = "test", blockHandler = "handleException", blockHandlerClass = {ExceptionUtil.class})
public void test() {
    System.out.println("Test");
}

public final class ExceptionUtil {

    // 这里的方法返回值需要和 使用该 Exception Handler 的方法的返回值一致
    // TestServiceImpl#test()
    public static void handleException(BlockException ex) {
        // Handler method that handles BlockException when blocked.
        // The method parameter list should match original method, with the last additional
        // parameter with type BlockException. The return type should be same as the original method.
        // The block handler method should be located in the same class with original method by default.
        // If you want to use method in other classes, you can set the blockHandlerClass
        // with corresponding Class (Note the method in other classes must be static).
        System.out.println("Oops: " + ex.getClass().getCanonicalName());
    }
}
```

1. 首先检查是否在注解上配置相应的异常处理器。
2. 如果我们配置了 BlockExceptionHandler，就执行 Handler 中定义好的异常处理器。
3. 如果没有配置，就使用 FallBack 相应的逻辑。

**也就是，如果我们只配置了 fallback，而没有配置 blockHandler 的话，如果出现了 BlockException，就会执行 fallback 逻辑**。当然，关于 fallback 的逻辑还得后续进行详细分析，现在让我们将目光聚集在如果解析注解上关于异常处理器的相关信息，以及如何找到相应的类中的相应的方法：

```java
// in AbstractSentinelAspectSupport.class
private Method extractBlockHandlerMethod(ProceedingJoinPoint pjp, String name, Class<?>[] locationClass) {
    // name：annotation.blockHandler
 	// locationClass：blockHandlerClass
    if (StringUtil.isBlank(name)) {
        return null;
    }
    // 如果 @SentinelResource 注解中配置了 blockHandlerClass，就从配置好的数组中的第一个所代表的类中找对应的 blockHandler 方法
    boolean mustStatic = locationClass != null && locationClass.length >= 1;
    Class<?> clazz;
    if (mustStatic) {
        clazz = locationClass[0];
    } else {
        // By default current class. 如果没有 blockHandlerClass，就直接当前方法中寻找
        clazz = pjp.getTarget().getClass();
    } // 从缓存 map 中找是否存在以 clazz:name 为 key 的 MethodWrapper
    MethodWrapper m = ResourceMetadataRegistry.lookupBlockHandler(clazz, name);
    if (m == null) {
        // First time, resolve the block handler. 总体来说，解析并找到对应的 blockHandler
        Method method = resolveBlockHandlerInternal(pjp, name, clazz, mustStatic);
        // Cache the method instance. 将解析到的结果保存在 map 中
        ResourceMetadataRegistry.updateBlockHandlerFor(clazz, name, method);
        return method;
    }
    if (!m.isPresent()) {
        return null;
    }
    return m.getMethod();
}
```

1. 可以看到，首先我们去查看我们是否配置了 blockHandlerClass，如果配置了就从该类中找，如果没有配置，就从当前方法所在的类中寻找。
2. 然后从缓存中查找是否存在以 clazz:name 为 key 的 MethodWrapper，如果存在，直接返回对应的 Method 对象。
3. 如果不存在，就使用反射进行解析，找到对应类中的对应配置好的 blockHandler 对应的 Method。当然，有关于如何使用反射来进行解析的具体流程笔者就不进行分析了，如果对于反射机制感兴趣的可以自行查看源码。
4. 找到解析结果之后，就将结果存放在 map 进行缓存，因为反射其实是比较消耗性能的，适当的缓存有利于提高性能。

在说完如何处理 BlockException 之后，接下来我们去看看 Sentinel 如何处理其他的异常，也就是 fallback 逻辑。

## 第三章 #handleFallback的分析

如第一章所述，在总体流畅上，如果出现了除了 BlockException 之外的其他异常，就需要结合 exceptionsToTrace 和 exceptionsToIgnore 进行异常的过滤，然后使用 Trace 记录了异常之后，就进入到 fallback 的处理逻辑。

```java
protected Object handleFallback(ProceedingJoinPoint pjp, SentinelResource annotation, Throwable ex)
    throws Throwable {
    return handleFallback(pjp, annotation.fallback(), annotation.defaultFallback(), annotation.fallbackClass(), ex);
}

protected Object handleFallback(ProceedingJoinPoint pjp, String fallback, String defaultFallback,
                                Class<?>[] fallbackClass, Throwable ex) throws Throwable {
    Object[] originArgs = pjp.getArgs();

    // Execute fallback function if configured.
    Method fallbackMethod = extractFallbackMethod(pjp, fallback, fallbackClass);
    if (fallbackMethod != null) {
        // Construct args.
        int paramCount = fallbackMethod.getParameterTypes().length;
        Object[] args;
        if (paramCount == originArgs.length) {
            args = originArgs;
        } else {
            args = Arrays.copyOf(originArgs, originArgs.length + 1);
            args[args.length - 1] = ex;
        }

        return invoke(pjp, fallbackMethod, args);
    } // 如果没有配置 fallback，就使用默认的 fallback
    // If fallback is absent, we'll try the defaultFallback if provided.
    return handleDefaultFallback(pjp, defaultFallback, fallbackClass, ex);
}
```

总体来说，fallback 的处理逻辑和 blockException 的处理逻辑大体相同，包括如何使用反射进行方法的解析等等。然而，**如果没有配置 blockHandler 还可以使用 fallback 进行兜底，而如果连 fallback 都没有配置，那么就只能使用 Sentinel 提供的默认的处理逻辑了**。

```java
protected Object handleDefaultFallback(ProceedingJoinPoint pjp, String defaultFallback,
                                       Class<?>[] fallbackClass, Throwable ex) throws Throwable {
    // Execute the default fallback function if configured.
    Method fallbackMethod = extractDefaultFallbackMethod(pjp, defaultFallback, fallbackClass);
    if (fallbackMethod != null) {
        // Construct args.
        Object[] args = fallbackMethod.getParameterTypes().length == 0 ? new Object[0] : new Object[] {ex};
        return invoke(pjp, fallbackMethod, args);
    }
    // 由于 @SentinelResource 中的 defaultFallback 不存在，就直接会抛出异常
    // 因此反应给用户的页面就是 Whitelabel Error Page
    // There was an unexpected error (type=Internal Server Error, status=500).
    throw ex;
}
```

可以看到，如果我们配置了 defaultFallback，就可以自定义相关逻辑，然后给用户一个良好的体验。

然而，如果我们并没有配置，那么只能将异常给抛出去，呈现给用户的页面就是我们常见的 Whitelabel Error Page 了。

## 参考文献

[open-source-framework-integrations | Sentinel (sentinelguard.io)](https://sentinelguard.io/zh-cn/docs/open-source-framework-integrations.html)
