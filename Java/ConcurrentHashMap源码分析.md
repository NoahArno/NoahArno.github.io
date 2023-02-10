# ConcurrentHashMap源码解析

## 基本概念

> 什么是ConcurrentHashMap？

ConcurrentHashMap是为了解决HashMap线程不安全而出现的一个类。

**底层结构**：在JDK1.7的时候，底层是分段的数组 + 链表；在JDK1.8的时候，使用了数组+ 链表 + 红黑树的底层结构。

**线程安全**：在JDK1.7的时候，ConcurrentHashMap对整个数组进行了分割，形成了多个Segment，而Segment类继承了ReentrantLock，也就是形成了多把锁，每把锁只锁住容器中的一部分数据。多线程访问不同数据段中的数据，就不会造成并发问题。每个Segment都包含一个HashEntry数组，而每个HashEntry是一个链表结构的元素。而在JDK1.8的时候，进行了大修改。采用synchronized和CAS来操作。同时每次使用synchronized只会锁住当前链表或红黑树的头节点，提升并发量。

## JDK7下的ConcurrentHashMap

优点：如果多个线程访问不同的Segment，实际上不会产生冲突。

缺点：Segment的初始值大小为16，而且这个容量一旦确定之后就不可以被修改。

## JDK8下的ConcurrentHashMap

对于ConcurrentHashMap的无参构造器来说，其实是个空构造器，什么都没做

```java
    /*
     * Encodings for Node hash fields. See above for explanation.
     */
    static final int MOVED     = -1; // hash for forwarding nodes
    static final int TREEBIN   = -2; // hash for roots of trees
    static final int RESERVED  = -3; // hash for transient reservations
```

### get方法

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode()); // 计算hash值
    // 要求table不为null，table的长度大于0，该key所处于的桶不为null
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        // 如果头节点就是要查找的key
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // hash值为负数表示该bin在扩容中或者是treebin，这时候调用find方法来查找
        // TreeBin表示红黑树头节点
        else if (eh < 0)
            // 注意TreeBin重写了Node的find方法
            return (p = e.find(h, key)) != null ? p.val : null;
        // 正常的链表遍历
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

> 为什么get方法不需要加锁？

Node的成员val是用volatile修饰的，因此对val的修改是满足可见性的。

而table数组也使用了volatile修饰，主要是保证数组在扩容时的可见性。

### put方法

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // 不允许key或value为null
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode()); //spread方法：return (h ^ (h >>> 16)) & HASH_BITS;
    int binCount = 0;
    // 注意这里的循环
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable(); // 第一次访问的时候，进行初始化，然后进入下一个循环
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 如果使用CAS添加链表表头成功就直接跳出循环
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED) // MOVED = -1
            // 帮忙扩容，之后进入下一循环
            tab = helpTransfer(tab, f);
        else {
            // 
            V oldVal = null;
            // 锁住链表头节点
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        // 遍历链表
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    } // 红黑树
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        // putTreeval会看key是否已经在树种，如果是则返回对应的TreeNode
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                              value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                // 树化操作
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 增加size计数
    addCount(1L, binCount);
    return null;
}
```

**总结**：

1. 如果table未初始化，就会先初始化
2. 如果table[i] == null，就会使用CAS去创建bin。
3. 如果已经有了，就锁住链表头节点，然后进行后续的put操作（链表还是红黑树），接着判断是否需要树化
4. 最后调用addCount方法，这里面会判断是否需要扩容，即是否超过阈值

### initTable方法

```java
/**
     * Initializes table, using the size recorded in sizeCtl.
     */
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // 如果小于0，表示另外的线程执行CAS成功，就要让出CPU使用权
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        // 设置SIZECTL为-1，
        // -1说明正在初始化
        // -N说明有N - 1个线程正在进行扩容
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            // 说明得到了锁（不是sync），该线程就会进行初始化操作，而其他的线程就会yield直到table创建
            try { // 双端检索
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

### （TODO）扩容

对于ConcurrentHashMap来说，它的扩容时机主要有两个：

- 集合中的元素个数到达阈值的时候触发扩容，主要在addCount方法中判断
- 树化的时候，如果此时数组长度小于64，也会触发扩容

[ConcurrentHashMap原理分析(二)-扩容 - 猿起缘灭 - 博客园 (cnblogs.com)](https://www.cnblogs.com/gunduzi/p/13651664.html)