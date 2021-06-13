---
title: "Java 并发：ConcurrentHashMap"
date: 2021-06-13T16:30:25+08:00
slug: ""
description: ""
keywords: []
tags: [java, concurrency, source_code]
math: false
toc: true
---

*ConcurrentHashMap* 对读取提供了**完全**的并发支持，对写入提供了**高性能**的并发支持。在读取数据时，*ConcurrentHashMap* 无需上锁；而在写入数据时，也不用对整个 Map 上锁。

由于读取数据不上锁，读取时可能与写入发生重叠，*ConcurrentHashMap* 只能反映最近完成的写入所产生的结果。与 volatile 类似，这也是符合 happens-before 原则的：如果通过 *ConcurrentHashMap* 读取到某次写入后的值，那么写入前的程序的执行一定先于读取后的程序。

与 *CopyOnWriteArrayList* 相同，*ConcurrentHashMap* 的迭代器在进行迭代时，不会因为写入或删除操作而报出 *ConcurrentModificationException*。但是，每次获得的迭代器只能被一个线程所使用，不应当传送给其他线程。一旦一个迭代器被创建，其状态就是确定的（一个快照），类似 `size()` \ `isEmpty()` \ `containsValue()` 等访问实时数据的函数就不能够用来控制程序了。

## 一、初始化

*CopyOnWriteArrayList* 中比较重要的成员变量如下： 

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {

	// 最大容量，hashcode前两位用作控制，所以最大为1<<30
    private static final int MAXIMUM_CAPACITY = 1 << 30;

    // 默认容量
    private static final int DEFAULT_CAPACITY = 16;

    // 负载因子，需要扩容的键值对临界值和容量的比值，默认情况下用 n - (n >>> 2) 实现
    private static final float LOAD_FACTOR = 0.75f;

    // 链表转化为红黑树的临界值
    static final int TREEIFY_THRESHOLD = 8;

    // 红黑树转化为链表的临界值
    static final int UNTREEIFY_THRESHOLD = 6;

    // 启用红黑树转化的最小容量，小于这个值时，优先扩容
    static final int MIN_TREEIFY_CAPACITY = 64;

    // 标记状态
    static final int MOVED     = -1; // hash for forwarding nodes
    static final int TREEBIN   = -2; // hash for roots of trees
    static final int RESERVED  = -3; // hash for transient reservations
    static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
 
    // 现有表
    transient volatile Node<K,V>[] table;

    // 更新后的表
    private transient volatile Node<K,V>[] nextTable;
    
    /* 用来控制表的初始化和扩容,具有以下几种状态
     * -1  : 正在初始化
     * < -1: 正在被-sizeCtl-1个线程扩容
     * >= 0: 未初始化时，初始化的容量；初始化后，下一次扩容的容量。*/
    private transient volatile int sizeCtl;
}
```

初始化时，若未指定初始容量，根据默认值创建；若指定了初始容量，将 sizeCtl 设置为下一次扩容的容量，即乘以 1.5 倍后向上取整（2 的幂）。

```java
    public ConcurrentHashMap() {
    }

	// 指定大小
    public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        // 下一次扩容的容量
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }

	// 向上取整（2的幂）的快速算法
    private static final int tableSizeFor(int c) {
        int n = c - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

不管是哪种初始化方式，都没有创建 table。这是一种懒加载策略，只有在第一次插入数据时，table 才会被创建。table 中存放着 *Node* 类保存的键值对，在设计时应保证其安全的发布。

```java
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;
        volatile Node<K,V> next;

        Node(int hash, K key, V val, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.val = val;
            this.next = next;
        }

        public final K getKey()       { return key; }
        public final V getValue()     { return val; }
        public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }

        // 在其子类中将被重写，转化为红黑树后查找逻辑是不同的
        Node<K,V> find(int h, Object k) {
            Node<K,V> e = this;
            if (k != null) {
                do {
                    K ek;
                    if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                } while ((e = e.next) != null);
            }
            return null;
        }
    }
```

## 二、写入

为了尽可能减小锁的粒度，*CopyOnWriteArrayList* 中引入了 *Unsafe* 提供的硬件支持的原子操作：

```java
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }

    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }

    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
```

`put()` 调用了 `putVal()`，传入 `putVal()` 的 onlyIfAbsent 表示是否只有在不存在相应 key 时才加入。可见 `put()` 始终传入 false，而另一个 public 方法 `putIfAbsent()` 则始终传入 true。`putValue()` 首先通过 `(hashcode ^ (hashcode >>> 16)) & HASH_BITS` 获取 hash 值。然后在一个循环中处理一下四种情况，直到跳出：

1. 如果 table 未初始化，初始化 table；
2. 如果 table 已经初始化，但是相应桶为空，直接创建 *Node* 加入其中，跳出；
3. 如果 table 已经初始化，且相应桶不为空，但该桶正在迁移（扩容中），当前线程将帮助迁移；
4. 如果 table 已经初始化，且相应桶不为空，且该桶未在迁移，将键值对加入到桶中，有两种情况：
   1. 桶中是链表，则遍历这个链表：
      1. 若找到相同 key，直接修改 value，跳出；
      2. 若未找到，创建新的 *Node* 加到尾部，跳出。
   2. 桶中是红黑树，通过 *TreeNode* 中的 `putTreeValue()` 写入，逻辑与上面类似，跳出。

在完成上面步骤之后，还会检查 binCount，如果满足转化红黑树的条件则进行转化。

```java
	public V put(K key, V value) {
        return putVal(key, value, false);
    }

    // onlyIfAbsent为true时，只能添加不能修改
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode()); // 取有效hash
        int binCount = 0; // 计算桶内个数，用以控制扩容或树的转化
        for (Node<K,V>[] tab = table;;) { // 循环，直到写入完成跳出
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable(); // 未初始化则先初始化
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break; // 桶为空，通过原子操作写入，不需要加锁
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f); // 该节点正在迁移，该线程也帮助迁移
            else {
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) { // 上锁和检查头节点是否有变
                        if (fh >= 0) { // 大于等于0，则为链表
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) { // 遍历同时记录binCount
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break; // 找到了，则修改
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break; // 没找到，则添加在尾部
                                }
                            }
                        }
                        else if (f instanceof TreeBin) { // 处理红黑树
                            Node<K,V> p;
                            binCount = 2;
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
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i); // 链表长度超过临界，转化为红黑树
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount); // 计数
        return null;
    }
```

### 2.1 初始化 table

不管是初始化 table 还是对 table 进行扩容，sizeCtl 都是至关重要的一环。在初始化 table 时，并不能保证只有当前线程在初始化，当有多个线程同时初始化 table 时，竞争就出现了。通过 sizeCtl 处理竞争很容易，若当前线程赢得了竞争，开始初始化，则把 sezeCtl 设置为 -1（这个操作必须原子性的）。其他线程在读到 sizeCtl = -1 时就放弃初始化，进而 `yield()` 让出资源等待初始化的完成，这就是为什么需要 while 循环的原因。但是这里还有两个问题：

1. 为什么在检查 table 是否初始化时需要 `(tab = table) == null || tab.length == 0`，似乎 `tab.length == 0` 总是不能单独成立的？
2. 为什么最后的 `sizeCtl = sc` 需要 finally 句式来保证其一定执行。

第一个问题我还没有答案，不过第二问题有以下解释。在创建新的 table 时，可能会因为空间不足而报出 `OutOfMemoryError`，该线程就无法执行下去，如果没有 finally 保证 `sizeCtl = sc` 一定执行，sizeCtl 将永远为 -1，而 table 也将永远无法创建。

```java
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
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

