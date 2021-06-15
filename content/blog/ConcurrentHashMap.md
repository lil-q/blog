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

*ConcurrentHashMap* 中比较重要的成员变量如下： 

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

为了尽可能减小锁的粒度，*ConcurrentHashMap* 中引入了 *Unsafe* 提供的硬件支持的原子操作：

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
            tab = initTable(); // 【未初始化则先初始化】
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
                    treeifyBin(tab, i); // 【链表长度超过临界，转化为红黑树或者扩容】
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount); // 【计数，超过sizeCtl会进行扩容】
    return null;
}
```

`putVal()` 中有三个函数是重点：`initTable()` \ `treeifyBin()` \ `addCount`。 `initTable()`必须要保证只有一个线程对 table 进行初始化。`treeifyBin()` 和 `addCount` 中都包含了扩容操作，但是扩容的结果却是不同的。为了便于区分，我们将 `treeifyBin()` 中使用 `tryPresize()` 进行的扩容称为 P 扩容，而将由 `addCount()` 进行的扩容称为 C 扩容。

### 2.1 initTable

不管是初始化 table 还是对 table 进行扩容，sizeCtl 都是至关重要的一环。在初始化 table 时，并不能保证只有当前线程需要初始化，当有多个线程同时初始化 table 时，竞争就出现了。通过 sizeCtl 处理竞争很容易，若当前线程赢得了竞争，开始初始化，则把 sezeCtl 设置为 -1（这个操作必须原子性的）。其他线程在读到 sizeCtl = -1 时就放弃初始化，进而 `yield()` 让出资源等待初始化的完成，这就是为什么需要 while 循环的原因。但是这里还有两个问题：

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

### 2.2 treeifyBin

如果插入一个键值对后，链表长度大于等于 TREEIFY_THRESHOLD，就需要进行扩容或转化红黑树。`treeifyBin()` 是这一操作的入口，首先判断 table 容量是否小于 MIN_TREEIFY_CAPACITY，如果是则进行 P 扩容（由 `tryPresize()` 进行的，有别于下文中由 `addCount()` 进行的），否则进行红黑树转化。

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY) // 扩容
            tryPresize(n << 1); 
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) { // 转化红黑树
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

### 2.3 addCount

`addCount()` 通过一个 *CounterCell* 数组计算键值对个数，虽然只是简单的计数，但为了尽可能减少线程冲突，其中蕴含了非常巧妙的设计。这部分内容将在第四章中介绍。

`addCount()` 中的 check 用来标识是否需要考虑扩容，`putVal()`调用时， check 值实际上就是 `putVal()` 中的 binCount，有以下可能：

* 写入前桶内为空：0；
* 写入前桶内为链表：写入的节点位置；
* 写入前桶内为红黑树：2。

当 check 大于等于 0 且满足以下三个条件时，就需要进行 C 扩容：

1. 键值对数达到sizeCtl；
2. table非空；
3. table长度非最大值。

实际上，C 扩容的步骤和 `tryPresize()` 中的一个分支完全一致，可以理解为 C 扩容是 P 扩容的一个子集，这部分将在下一章中介绍。

```java
    private final void addCount(long x, int check) {
        // 计数部分,暂不考虑
        CounterCell[] as; long b, s;
        ...
        s = sumCount(); // 键值对数
        
        // 判断是否C扩容
        if (check >= 0) { 
            Node<K,V>[] tab, nt; int n, sc;
            // 满足三个条件则C扩容：1.键值对数达到sizeCtl；2.table非空；3.table长度非最大值
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                // 以下为C扩容，和tryPresize()中的一个分支完全一致
                int rs = resizeStamp(n);
                if (sc < 0) {
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```

## 三、扩容

*ConcurrentHashMap* 的扩容非常有趣，正如上文所说，当前线程如果发现自己要处理的桶正在迁移，并不会干等着，而是利用自己的计算资源帮助完成整个 table 的迁移。

另一个有趣的点是，对于同样长度的 table，可能扩容出不同长度的新 table。来看下面一个例子：

```java
// 测试例
public class ConcurrentMapTest {
    public static void main(String[] args)
            throws NoSuchFieldException, IllegalAccessException {
        final int DEFAULT_CAPACITY = 16; // 默认容量
        ConcurrentHashMap<Integer, Integer> map1 = new ConcurrentHashMap<>();
        ConcurrentHashMap<Integer, Integer> map2 = new ConcurrentHashMap<>();
        // c扩容
        for (int i = 0; i < 12; i++) {
            map1.put(i, i); // key不同
            int length;
            if ((length = getLength(map1)) != DEFAULT_CAPACITY) {
                System.out.printf("第%d次插入后map1扩容：%d\n", i + 1, length);
                break;
            }
        }
        // p扩容
        for (int i = 0; i < 12; i++) {
            map2.put(i * 16, i); // key不同
            int length;
            if ((length = getLength(map2)) != DEFAULT_CAPACITY) {
                System.out.printf("第%d次插入后map1扩容：%d\n", i + 1, length);
                break;
            }
        }
    }

    // 通过反射获取table.length
    private static int getLength(ConcurrentHashMap map)
            throws NoSuchFieldException, IllegalAccessException {
        Field tableField = ConcurrentHashMap.class.getDeclaredField("table");
        tableField.setAccessible(true);
        Object[] table = (Object[]) tableField.get(map);
        return (table == null ? 0 : table.length);
    }
}

```

```txt
第12次插入后map1扩容：32
第9次插入后map1扩容：128
```

上面两个 map 扩容后出现了不同的容量，这是因为两者触发了不同的扩容机制。map1 中每次插入的 key 为 1、2、3 等，由于 *Integer* 的 hashcode 等于其自身的开箱值，可以确保 map1 中的键值对是均匀分布的。因此，导致 map1 扩容的原因是键值对的数量达到sizeCtl（12）。map2 中每次插入的是 16 的倍数，所以每次插入的键值对都放进了第一个桶内，这导致第一个桶的链表越来越长，当长度大于 TREEIFY_THRESHOLD（8）时，触发了 `treeifyBin()`，根据 `treeifyBin()` 的逻辑，map2 当前容量小于 MIN_TREEIFY_CAPACITY（64），优先通过 `tryPresize()` 进行 P 扩容。

### 3.1 P 扩容

从上面的测试例可以看出，P 扩容正好是 C 扩容的四倍，正并不是巧合。

```java
private final void tryPresize(int size) {
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
    tableSizeFor(size + (size >>> 1) + 1); // 获得下一个
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        else if (tab == table) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                Node<K,V>[] nt;
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}
```



## 四、计数

