## ThreadLocal总览

对于每个Thread实例对象，它内部都会有一个threadlocals字段，而该属性字段的类型其实就是ThreadLocal中的内部类ThreadLocalMap。它以ThreadLocal为key，同时使用开放定址法来解决哈希冲突。

```java
static class ThreadLocalMap {

    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;
        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    private Entry[] table;
}
```

可以看到，ThreadLocalMap对ThreadLocal进行了虚引用，并且将其进行扩展，组成k-v键值对。

> 虚引用的作用

一般而言，一个ThreadLocal对象被创建出来之后，会被放到ThreadLocalMap中，如果使用的是线程池，那么Thread的生命周期很长，在ThreadLocal失去了它原本的强引用之后，还存在着Map对它的强引用，这可能就会造成内存泄漏

在我们实际开发中，都会初始化一个全局的static的ThreadLocal对象，因此一般而言该TL对象都会有一个外部的强引用指向它，因此此时在get或者set的时候的各种检查过期机制都不会生效，因为我的TL对象是存在强引用的，因为不会被GC掉，从而导致Entry中的key不会为null。

因此我们就必须去调用remove方法，它不仅会显示的将虚引用给断开，同时还会执行一次探测性清除，在这里面会将对应槽位的Entry设置为null，将它的value也设置为null。

如果使用的是全局的static的TL + 线程池的方式，那么调用remove更多的是防止之前的结果对后续的结果造成影响。

### get()

```java
public T get() {
    Thread t = Thread.currentThread();
    // 对于threadlocal的实现来说，其实就是调用t.threadlocals;
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this); 
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

在getEntry方法里面：

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i]; // 如果直接找到了，就返回
    if (e != null && e.get() == key)
        return e;
    else // 没找到
        return getEntryAfterMiss(key, i, e);
}
```

### getEntryAfterMiss

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null) // 执行探测性清除
            expungeStaleEntry(i);
        else 
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

### 小总结

- get的逻辑相对于简单，直接根据key计算出对应的槽位，如果是该key就直接返回；
- 如果不是，就往后遍历，如果发现了过期的key就执行探测性清除
- 如果找不到，就返回null

### set()

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
}
```

上述流程其实非常基本，我们继续往下看看set方法的详情：

```java
private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    // 该hash的计算，其实是通过一个static 的 AtomicInteger nextHashCode 来计算的，
    // 它每次调用的时候都会增加一个黄金分割数，也叫斐波拉契数，
    // 这样子会让 key 的分布更加的均衡
    int i = key.threadLocalHashCode & (len-1); // 计算该key在map中的哈希值，并且计算它的位置

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        if (k == key) {
            e.value = value;
            return;
        }
        // 如果k为null了，就得执行清理工作，下面的就是核心方法
        if (k == null) { 
            replaceStaleEntry(key, value, i);
            return;
        }
    }
	// 上面的没有return掉，说明map中找不到该key，就直接插入在i位置吧
    // 注意这个 i 位置会变化的，可能不是之前通过哈希值计算得到的位置
    tab[i] = new Entry(key, value);
    int sz = ++size;
    // threshold一般为数组长度的三分之二
    // 如果cleanSomeSlots没有清理一个
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

replaceStaleEntry

这是当set的时候发现了key为null的时候才会执行的方法，它会清理掉一些过期的entry

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;
	// slotToExpunge表示它之前的槽位没有过期的entry，
    // 当然这是建立在一段连续的槽位之中
    int slotToExpunge = staleSlot; 
    // 向前搜索
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null) // 如果发现有entry的key为null，就记录好它的位置
            slotToExpunge = i;
	// 开始向后搜索
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        // 如果在向后搜索的过程中，发现了存在set进来的key
        if (k == key) {
            e.value = value;
			// 注意staleSlot对应的槽位其实是过期entry
            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // 如果在向前搜索或者向后搜索的时候都没有发现过期的entry
            if (slotToExpunge == staleSlot)
                slotToExpunge = i; // 说明i位置之前的连续槽位都没有过期entry
            // 于是就得从它的后面开始查看
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // 如果我们都没有找到对应的key，就直接放在一开始的过期entry的位置上吧
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);
	// 如果上述流程期间发现了过期entry，就进行清除
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

### 探测性清除expungeStaleEntry

探测式清理，是以当前遇到的 GC 元素开始，向后不断的清理。直到遇到 null 为止，才停止 rehash 计算

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
	
    // expunge entry at staleSlot 进行当前位置的清理
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i; // 向后进行清除，直到遇见entry为null
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        // 如果该位置过期了，就直接进行清除
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else { // 如果没有过期，就将其移动到最靠近它的hash值所对应的槽位上
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

### 启发式清除cleanSomeSlots

试探的扫描一些单元格，寻找过期元素，也就是被垃圾回收的元素。当添加新元素或删除另一个过时元素时，将调用此函数。它执行对数扫描次数，作为不扫描（快速但保留垃圾）和与元素数量成比例的扫描次数之间的平衡，这将找到所有垃圾，但会导致一些插入花费O（n）时间。

```java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        // 如果发现了过期的entry，就执行一次探测性清除
        if (e != null && e.get() == null) {
            n = len; // 将n给恢复成len
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);  // 如果没有发现过期entry的话，log(n)就结束了
    return removed;
}
```

### rehash

```java
if (!cleanSomeSlots(i, sz) && sz >= threshold)
    rehash();
```

可以看到，当我们执行启发式清除的时候，如果没有发现可清除entry，并且此时的entry数量大于或等于数组长度的三分之二，就会进入rehash逻辑

```java
private void rehash() {
    expungeStaleEntries(); // 做一次全量清理

    // Use lower threshold for doubling to avoid hysteresis
    if (size >= threshold - threshold / 4)
        resize();
}

private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            expungeStaleEntry(j);
    }
}
```

可以看到，rehash的时候一上来就做一次全量清理，也就是从数组的开始到结束，查询每个entry，如果它是过期的就进行一次探测性清除。

如果做完全量清理了发现数组的使用率在百分之五十以上，就会进入resize流程，进行扩容

```java
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; // Help the GC
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }

    setThreshold(newLen);
    size = count;
    table = newTab;
}
```

### 小总结

- 先根据当前的线程，取到它内部的ThreadLocalMap属性，然后根据key计算它的初始位置
- 从当前位置一直往后遍历，直到访问到的槽位为空
  - 如果找到了相同的key就直接进行值的更新
  - 如果发现了过期的key，就执行replaceStaleEntry方法
    - 先从当前位置staleSlot往前走，如果发现了过期的entry，就更新slotToExpunge的值，直到遇到了空的entry就停止
    - 接着从staleSlot往后走，如果遇到了相同的key，就将key-value放入到staleSlot对应的槽上，然后将发现key对应的位置上的槽位值为过期槽位，也就是进行了一个交换，然后在return之前进行一次启发式清理
    - 如果整个往后遍历都结束了，也就是遇到为空的entry，还没return，就直接在staleSlot处插入我们的k-v，然后再进行启发式清理
- 当上述流程遍历完之后都没有return，就将该key放置到经过开放定址法后的槽位当中
- 执行一遍启发式清理，如果没有可清理对象并且size大于len的三分之二，进行rehash
- 在rehash的时候，进行一次全量清理，清理之后如果size还是大于一半就进行扩容。

### remove()

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null) {
        m.remove(this);
    }
}

private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear(); // 显式断开弱引用
            expungeStaleEntry(i); // 探测性清除
            return;
        }
    }
}
```

## FastThreadLocal总览

FTL是Netty写的一个轮子，它内部维护了一个索引常量index，每次创建FTL的时候都会自动加1，保证下标的不重复性，虽然会产生大量的index，但是避免了计算索引下标位置以及处理hash冲突带来的损耗。相当于以空间换取时间。

同时，如果想要使用FTL，就得结合FastThreadLocalThread线程类以及它的子类，Netty也提供了DefaultThreadFactory工厂类：

```java
DefaultThreadFactory defaultThreadFactory = new DefaultThreadFactory(Test.class);
FastThreadLocal<Integer> fastThreadLocal = new FastThreadLocal<>();

Thread thread = defaultThreadFactory.newThread(() -> {
    fastThreadLocal.set(1);
    fastThreadLocal.get();
    fastThreadLocal.remove();
});
thread.start();
```

- **以空间换时间，不需要通过TL的哈希值计算当前的下标，而是直接使用一个自增index**
- **和TL用开放定址法解决哈希冲突不一样，FTL无需解决哈希冲突**
- **TL扩容的时候需要rehash，而FTL并不需要**
- **FTL就算结束不使用remove也不用担心内存泄漏，因为它自己在任务结束后调用removeAll方法**

### FastThreadLocalRunnable

我们上面说到，想要正确使用FTL，就必须结合FTLT线程类以及它的子类，同时构造FTLT的时候，也会通过FastThreadLocalRunnable对Runnable对象进行了包装。

```java
// 使用DefaultThreadFactory构建FastThreadLocalThread的时候，
Thread t = newThread(FastThreadLocalRunnable.wrap(r), prefix + nextId.incrementAndGet());
```

而我们看看FastThreadLocalRunnable的源码：

```java
final class FastThreadLocalRunnable implements Runnable {
    private final Runnable runnable;

    private FastThreadLocalRunnable(Runnable runnable) {
        this.runnable = ObjectUtil.checkNotNull(runnable, "runnable");
    }

    @Override
    public void run() {
        try {
            runnable.run();
        } finally {
            FastThreadLocal.removeAll();
        }
    }

    static Runnable wrap(Runnable runnable) {
        return runnable instanceof FastThreadLocalRunnable ? runnable : new FastThreadLocalRunnable(runnable);
    }
}
```

**其实可以看到，它的run方法执行的时候，会自动的执行removeAll方法，这样其实就不需要我们显示的在方法执行完毕之后去调用FTL的remove方法，其实它的内部会自动的进行处理。**

### InternalThreadLocalMap

和普通的Thread对象一样，在FastThreadLocalThread中，也有一个Map用来存储FTL，而这个Map其实就是InternalThreadLocalMap。

InternalThreadLocalMap继承了UnpaddedInternalThreadLocalMap：

```java
class UnpaddedInternalThreadLocalMap {

    static final ThreadLocal<InternalThreadLocalMap> slowThreadLocalMap = new ThreadLocal<InternalThreadLocalMap>();
    static final AtomicInteger nextIndex = new AtomicInteger();

    /** Used by {@link FastThreadLocal} */
    Object[] indexedVariables;

    // Core thread-locals
    int futureListenerStackDepth;
    int localChannelReaderStackDepth;
    Map<Class<?>, Boolean> handlerSharableCache;
    IntegerHolder counterHashCode;
    ThreadLocalRandom random;
    Map<Class<?>, TypeParameterMatcher> typeParameterMatcherGetCache;
    Map<Class<?>, Map<String, TypeParameterMatcher>> typeParameterMatcherFindCache;

    // String-related thread-locals
    StringBuilder stringBuilder;
    Map<Charset, CharsetEncoder> charsetEncoderCache;
    Map<Charset, CharsetDecoder> charsetDecoderCache;

    // ArrayList-related thread-locals
    ArrayList<Object> arrayList;

    UnpaddedInternalThreadLocalMap(Object[] indexedVariables) {
        this.indexedVariables = indexedVariables;
    }
}
```

### FastThreadLocal初始化

在FTL中，有如下代码：

```java
private static final int variablesToRemoveIndex = InternalThreadLocalMap.nextVariableIndex();
static final AtomicInteger nextIndex = new AtomicInteger(); // 位于FTL的父类中
private final int index;

public FastThreadLocal() {
    index = InternalThreadLocalMap.nextVariableIndex();
}
// 该方法位于InternalThreadLocalMap中
public static int nextVariableIndex() {
    int index = nextIndex.getAndIncrement();
    if (index < 0) {
        nextIndex.decrementAndGet();
        throw new IllegalStateException("too many thread-local indexed variables");
    }
    return index;
}
```

注意看，我们的FTL在类加载阶段，就会将variablesToRemoveIndex变量设置为0，然后将nextIndex变为1。同时，我们每次创建一个FastThreadLocal，都会给它一个索引index，初始值为1，每次加1。

而我们的数据其实是使用一个Object数组进行存储，因此，对于FastThreadLocal来说，**数组的0号位置其实放的是一个存放了所有FastThreadLocal对象的Set，而从1到数组的结尾都是放的value值**

**而对于Object数组的初始化，其实就是给它们填充UNSET，也就是static的new Object**

### get方法

在get方法中，我们会拿到该线程对应的Map，**关于FTL兼容TL的逻辑就在这里体现，后续我们再进行分析**

```java
public final V get() {
    InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
    /*    
    public Object indexedVariable(int index) {
        Object[] lookup = indexedVariables;
        return index < lookup.length? lookup[index] : UNSET;
    }
    */
    Object v = threadLocalMap.indexedVariable(index); // 拿到该FTL对应的value
    if (v != InternalThreadLocalMap.UNSET) { // 说明有值
        return (V) v;
    }
	// 这一步会给当前FTL对应的数组中的位置设置为null
    return initialize(threadLocalMap);
}
```

### set方法

```java
public final void set(V value) {
    // set进去的值不为默认值
    if (value != InternalThreadLocalMap.UNSET) {
        // 拿到当前线程的Map
        InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
        // 核心逻辑
        setKnownNotUnset(threadLocalMap, value);
    } else { // 如果是默认值，就清理，可以将当前的FTL从Set中移除
        remove();
    }
}

private void setKnownNotUnset(InternalThreadLocalMap threadLocalMap, V value) {
    // 如果设置成功了并且是第一次设置成功，也就是返回的值是true：oldValue == UNSET
    if (threadLocalMap.setIndexedVariable(index, value)) {
        // 将FTL添加到Set中
        addToVariablesToRemove(threadLocalMap, this);
    }
}

private static void addToVariablesToRemove(InternalThreadLocalMap threadLocalMap, FastThreadLocal<?> variable) {
    Object v = threadLocalMap.indexedVariable(variablesToRemoveIndex);
    Set<FastThreadLocal<?>> variablesToRemove;
    if (v == InternalThreadLocalMap.UNSET || v == null) {
        // 创建Set
        variablesToRemove = Collections.newSetFromMap(new IdentityHashMap<FastThreadLocal<?>, Boolean>());
        threadLocalMap.setIndexedVariable(variablesToRemoveIndex, variablesToRemove);
    } else {
        variablesToRemove = (Set<FastThreadLocal<?>>) v;
    }
	// 将当前FTL给添加到Set中
    variablesToRemove.add(variable);
}
```

### remove

```java
public final void remove() {
    remove(InternalThreadLocalMap.getIfSet());
}

public final void remove(InternalThreadLocalMap threadLocalMap) {
    if (threadLocalMap == null) {
        return;
    }
    // 将当前FTL在Object数组中对应的index位置上的value设置为UNSET，并返回value
    Object v = threadLocalMap.removeIndexedVariable(index);
    // 从Object数组0号位置上的set中移除掉当前FTL
    removeFromVariablesToRemove(threadLocalMap, this);

    if (v != InternalThreadLocalMap.UNSET) {
        try {
            onRemoval((V) v); // 可以自定义处理逻辑
        } catch (Exception e) {
            PlatformDependent.throwException(e);
        }
    }
}

private static void removeFromVariablesToRemove(
    InternalThreadLocalMap threadLocalMap, FastThreadLocal<?> variable) {
	// 拿到数组0号位置上的Object
    Object v = threadLocalMap.indexedVariable(variablesToRemoveIndex);

    if (v == InternalThreadLocalMap.UNSET || v == null) {
        return;
    }
	// 将其转换为Set
    @SuppressWarnings("unchecked")
    Set<FastThreadLocal<?>> variablesToRemove = (Set<FastThreadLocal<?>>) v;
    variablesToRemove.remove(variable); // 从Set中移除掉当前FTL
}
```

### removeAll

```java
public static void removeAll() {
    InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.getIfSet();
    if (threadLocalMap == null) {
        return;
    }

    try {
        Object v = threadLocalMap.indexedVariable(variablesToRemoveIndex);
        if (v != null && v != InternalThreadLocalMap.UNSET) {
            // 同样是拿到Set
            @SuppressWarnings("unchecked")
            Set<FastThreadLocal<?>> variablesToRemove = (Set<FastThreadLocal<?>>) v;
            // 转换为数组
            FastThreadLocal<?>[] variablesToRemoveArray =
                variablesToRemove.toArray(new FastThreadLocal[0]);
            for (FastThreadLocal<?> tlv: variablesToRemoveArray) {
                // 挨个遍历去remove掉
                tlv.remove(threadLocalMap);
            }
        }
    } finally {
        // 移除掉当前线程所拥有的Map
        /*
        public static void remove() {
        	Thread thread = Thread.currentThread();
        	if (thread instanceof FastThreadLocalThread) {
            	((FastThreadLocalThread) thread).setThreadLocalMap(null);
        	} else {
            	slowThreadLocalMap.remove();
        	}
    	}   
        */
        InternalThreadLocalMap.remove();
    }
}
```

### 如何兼容TL

我们在获取当前线程的FTL的时候，需要执行以下语句：

```java
InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();

public static InternalThreadLocalMap get() {
    Thread thread = Thread.currentThread();
    if (thread instanceof FastThreadLocalThread) {
        return fastGet((FastThreadLocalThread) thread);
    } else {
        return slowGet();
    }
}
```

可以看到，如果我们当前线程是FTLT，就执行fastGet，如果是普通的，就执行slowGet

```java
// 直接从FTLT中获取就行，如果没有就创建
private static InternalThreadLocalMap fastGet(FastThreadLocalThread thread) {
    InternalThreadLocalMap threadLocalMap = thread.threadLocalMap();
    if (threadLocalMap == null) {
        thread.setThreadLocalMap(threadLocalMap = new InternalThreadLocalMap());
    }
    return threadLocalMap;
}

private static InternalThreadLocalMap slowGet() {
    // 其实我们在InternalThreadLocalMap的父类中，有一个static final修饰的常量
    // static final ThreadLocal<InternalThreadLocalMap> slowThreadLocalMap = new ThreadLocal<InternalThreadLocalMap>();
    // 也就是一个ThreadLocal，它会保证每个线程都会有一份InternalThreadLocalMap
    // 我们直接拿就行
    ThreadLocal<InternalThreadLocalMap> slowThreadLocalMap = UnpaddedInternalThreadLocalMap.slowThreadLocalMap;
    InternalThreadLocalMap ret = slowThreadLocalMap.get();
    if (ret == null) {
        ret = new InternalThreadLocalMap();
        slowThreadLocalMap.set(ret);
    }
    return ret;
}
```

