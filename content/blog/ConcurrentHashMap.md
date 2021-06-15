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

从上面的测试例可以看出，P 扩容正好是 C 扩容的四倍，这是巧合吗？上一章节中，我们已经对 `addCount()` 有了初步的认识，它会先计数，然后根据计数判断是否进行 C 扩容。在具体分析 C 扩容前，先来看看 `tryPresize()` 是如何实现 P 扩容的，也许能找到他们之间的一些联系。

首先  `tryPresize()` 会获得一个扩容要求 c，他是由传入 size 乘以 1.5 倍后向上舍入得到的。例如，初始 table.length=16，传入 size=32，可以得到 c=64。得到 c 后，通过 while 循环不断检验 sizeCtl 是否大于 c，有以下分支：

1. 如果 table 未初始化：初始化 table；
2. 如果 sizeCtl >= c：扩容完毕，跳出；
3. 进行 C 扩容。

现在我们知道，P 扩容实际上提供了 table 的初始化（在对空 map 进行 `putAll()` 时需要）和扩容的终止条件，其余时间都是在循环 C 扩容。上面的例子中，P 扩容后容量是 C 扩容的四倍，说明 P 扩容进行了三次 C 扩容。

1. 第一次扩容：table.length=32，c=64 > sizeCtl=24；
2. 第二次扩容：table.length=64，c=64 > sizeCtl=48；
3. 第三次扩容：table.length=128，c=64 < sizeCtl=96，跳出。

总结，C 扩容将 table.length 翻倍，而 P 扩容进行若干次 C 扩容，直到满足条件。之所以 P 扩容比 C 扩容大得多，是因为触发 P 扩容时，map 已经察觉到明显的哈希冲突了。

```java
// 注意: 如果是由treeifyBin()调用，传入的size已经是两倍的原table长度
private final void tryPresize(int size) {
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
    tableSizeFor(size + (size >>> 1) + 1); // 获得扩容要求c
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
        // 如果table没初始化，先初始化
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
        // 满足扩容要求，扩容完毕，跳出
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        // 与C扩容一致
        else if (tab == table) {
            int rs = resizeStamp(n); // 获得扩容标志rs
            // 有其他线程正在扩容
            if (sc < 0) {
                Node<K,V>[] nt;
                // 出现以下情况不应该帮助迁移，见下文
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                // 原子操作将sizeCtl中的线程数加一
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    // 把table迁移到nextTable
                    transfer(tab, nt);
            }
            // 没有其他线程在扩容，通过原子操作尝试扩容，把sizeCtl设置为相应负数
            // 原子操作成功，将(rs << RESIZE_STAMP_SHIFT) + 2)赋值给sizeCtl
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                // 创建一个两倍长度的新table，并赋值到nextTable上
                // 开始迁移
                transfer(tab, null);
        }
    }
}
```

## 3.2 C 扩容

C 扩容的逻辑相对简单，但要理解它却比较困难，罪魁祸首就是我们老朋友 sizeCtl！

源码中的注解表示，当多个线程同时进行扩容时，sizeCtl=-(1+N)，其中 N 表示正在扩容的线程数。如果按照这个逻辑，在扩容时，sizeClt 应当是一个比较接近 0 的负数， 但事实上 sizeCtl 通常都是一个非常小的负数。出现这种情况的原因是 sizeCtl 的高 16 位被用来标识 table 了，其低 16 位才标识 1+N。先来看扩容标识 rs 是怎么获得的。

```java
static final int resizeStamp(int n) {
    // 取前导0的个数，并把低16位的第一位置1              RESIZE_STAMP_BITS默认16
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```

为了方便理解，还是以初始长度为 16 的 table 为例，前导零一共 27 个，可得：

```txt
rs = 0000 0000 0000 0000 1000 0000 0010 0111
                        (置1)           (27)
```

有了扩容标识 rs 后，判断 sizeCtl，如果 sizeCtl 为负，说明有其他线程正在扩容；如果 sizeCtl 非负，则通过原子操作将 sizeCtl 设置为相应负数，原子操作成功就开始扩容。

```java
U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2);
// 原子操作成功，则siztCtl = 1000 0000 0010 0111 0000 0000 0000 0010   
```

原子操作成功后，sizeCtl 就变成了一个很小的负数，事实上，我们不应该关注这个值本身的大小，只需要关心其高 16 位和低 16 位的真正含义就可以了。sizeCtl 修改完毕后，`transfer(tab, null)` 将先创建一个两倍长度的 nextTable，然后开始将 table 上的数据迁移到 nextTable 上。

这时候其他线程就会觉察到正在迁移（sizeCtl<0）并试图帮助迁移。首先判断是否有资格帮助迁移，以下情况将不予参与扩容并直接跳出：

* (sc >>> RESIZE_STAMP_SHIFT) != rs：扩容标识对不上，说明不是同一个 table，帮不上忙；
* sc == rs + 1：扩容已经结束，收工了；
*  sc == rs + MAX_RESIZERS：已经达到最大扩容线程数，人手够了；
* (nt = nextTable) == null：nextTab还没创建，工厂还没建完；
* transferIndex <= 0：所有桶都已经分配出去，来晚了一步。

如果有资格参与扩容，就进行一次原子操作对 sizeCtl 加一，成功就可以 `transfer(tab, nt)`，将 table 中的数据迁移到 nextTable。

### 3.3 transfer

终于，关于扩容只剩下最后一个问题需要搞清楚——table 和 nextTable 之间的数据是如何迁移的。

`transfer()` 非常长，包含了很多逻辑，仔细拆分后能更好理解。我们首先列出总体框架，然后对细节进行推敲。

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 根据CPU数确定划分table的步长
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    // 若传入的nextTab为null，则创建nextTable
    if (nextTab == null) { // initiating
        try {
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) { // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n; // 用于控制扩容区间的分配
    }
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab); // 用于标记已处理的空桶
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    // 循环处理，i表示当前处理的桶下标（原table），bound表示分得区间得下界
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        // 领取区间 / 控制桶的序号 i--
        while (advance) {
            ...
        }
        // 判断是否结束
        if (i < 0 || i >= n || i + n >= nextn) {
            ...
        }
        // 当前桶为空，插入一个 fwd
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // 当前桶已经迁移
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        // 迁移桶内数据
        else {
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    // 处理节点是链表的情况
                    if (fh >= 0) {
                        ...
                    }
                    // 处理节点是红黑树的情况
                    else if (f instanceof TreeBin) {
                        ...
                    }
                }
            }
        }
    }
}
```

我们先将 `transfer()` 拆分为以下几个部分：

1. 根据 CPU 数确定划分 table 的步长：
   1. 如果是单核 CPU，步长就是 table.length；
   2. 如果是多核 CPU，步长为 `max(n >>> 3) / NCPU, MIN_TRANSFER_STRIDE)`，MIN_TRANSFER_STRIDE 默认 16；
2. 若传入的 nextTab 为null，则创建 nextTable：
   1. 将创建的 nextTab 赋给全局变量 nextTable；
   2. 将用于控制扩容区间的分配 transferIndex 初始化为 table.length；
3. 开始循环处理区间内各个桶得迁移，由 advance 控制程序：
   1. while 循环对区间 [bound, i] 进行分配或调整，`advance = false`；
   2. 判断当前区间的迁移是否结束，结束则 return，未结束 `i = n` `advance = true`，下一个循环检查整张表；
   3. 当前桶为空，则插入一个 fwd，`advance = casTabAt(tab, i, null, fwd)`；
   4. 当前桶已经迁移，`advance = true`；
   5. 迁移桶内数据：

```java
/**
 * Moves and/or copies the nodes in each bin to new table. See
 * above for explanation.
 * 
 * transferIndex 表示转移时的下标，初始为扩容前的 length。
 * 
 * 我们假设长度是 32
 */
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 将 length / 8 然后除以 CPU核心数。如果得到的结果小于 16，那么就使用 16。
    // 这里的目的是让每个 CPU 处理的桶一样多，避免出现转移任务不均匀的现象，如果桶较少的话，默认一个 CPU（一个线程）处理 16 个桶
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range 细分范围 stridea：TODO
    // 新的 table 尚未初始化
    if (nextTab == null) {            // initiating
        try {
            // 扩容  2 倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            // 更新
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            // 扩容失败， sizeCtl 使用 int 最大值。
            sizeCtl = Integer.MAX_VALUE;
            return;// 结束
        }
        // 更新成员变量
        nextTable = nextTab;
        // 更新转移下标，就是 老的 tab 的 length
        transferIndex = n;
    }
    // 新 tab 的 length
    int nextn = nextTab.length;
    // 创建一个 fwd 节点，用于占位。当别的线程发现这个槽位中是 fwd 类型的节点，则跳过这个节点。
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    // 首次推进为 true，如果等于 true，说明需要再次推进一个下标（i--），反之，如果是 false，那么就不能推进下标，需要将当前的下标处理完毕才能继续推进
    boolean advance = true;
    // 完成状态，如果是 true，就结束此方法。
    boolean finishing = false; // to ensure sweep before committing nextTab
    // 死循环,i 表示下标，bound 表示当前线程可以处理的当前桶区间最小下标
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        // 如果当前线程可以向后推进；这个循环就是控制 i 递减。同时，每个线程都会进入这里取得自己需要转移的桶的区间
        while (advance) {
            int nextIndex, nextBound;
            // 对 i 减一，判断是否大于等于 bound （正常情况下，如果大于 bound 不成立，
            // 说明该线程上次领取的任务已经完成了。那么，需要在下面继续领取任务）
            // 如果对 i 减一大于等于 bound（还需要继续做任务），或者完成了，
            // 修改推进状态为 false，不能推进了。任务成功后修改推进状态为 true。
            // 通常，第一次进入循环，i-- 这个判断会无法通过，从而走下面的 nextIndex 
            // 赋值操作（获取最新的转移下标）。其余情况都是：如果可以推进，将 i 减一，
            // 然后修改成不可推进。如果 i 对应的桶处理成功了，改成可以推进。
            if (--i >= bound || finishing)
                advance = false;// 这里设置 false，是为了防止在没有成功处理一个桶的情况下却进行了推进
            // 这里的目的是：1. 当一个线程进入时，会选取最新的转移下标。2. 当一个线程处理完自己的区间时，如果还有剩余区间的没有别的线程处理。再次获取区间。
            else if ((nextIndex = transferIndex) <= 0) {
                // 如果小于等于0，说明没有区间了 ，i 改成 -1，推进状态变成 false，
                // 不再推进，表示，扩容结束了，当前线程可以退出了
                // 这个 -1 会在下面的 if 块里判断，从而进入完成状态判断
                i = -1;
                advance = false;// 这里设置 false，是为了防止在没有成功处理一个桶的情况下却进行了推进
            }// CAS 修改 transferIndex，即 length - 区间值，留下剩余的区间值供后面的线程使用
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;// 这个值就是当前线程可以处理的最小当前区间最小下标
                i = nextIndex - 1; // 初次对i 赋值，这个就是当前线程可以处理的当前区间的最大下标
                advance = false; // 这里设置 false，是为了防止在没有成功处理一个桶的情况下却进行了推进，这样对导致漏掉某个桶。下面的 if (tabAt(tab, i) == f) 判断会出现这样的情况。
            }
        }
        // 如果 i 小于 0 （不在 tab 下标内，按照上面的判断，领取最后一段区间的线程扩容结束）
        //  如果 i >= tab.length(不知道为什么这么判断)
        //  如果 i + tab.length >= nextTable.length  （不知道为什么这么判断）
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) { // 如果完成了扩容
                nextTable = null;// 删除成员变量
                table = nextTab;// 更新 table
                sizeCtl = (n << 1) - (n >>> 1); // 更新阈值
                return;// 结束方法。
            }
            // 如果没完成
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {// 尝试将 sc -1. 表示这个线程结束帮助扩容了，将 sc 的低 16 位减一。
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)// 如果 sc - 2 不等于标识符左移 16 位。如果他们相等了，说明没有线程在帮助他们扩容了。也就是说，扩容结束了。
                    return;// 不相等，说明没结束，当前线程结束方法。
                finishing = advance = true;// 如果相等，扩容结束了，更新 finising 变量
                i = n; // 再次循环检查一下整张表
            }
        }
        else if ((f = tabAt(tab, i)) == null) // 获取老 tab i 下标位置的变量，如果是 null，就使用 fwd 占位。
            advance = casTabAt(tab, i, null, fwd);// 如果成功写入 fwd 占位，再次推进一个下标
        else if ((fh = f.hash) == MOVED)// 如果不是 null 且 hash 值是 MOVED。
            advance = true; // already processed // 说明别的线程已经处理过了，再次推进一个下标
        else {// 到这里，说明这个位置有实际值了，且不是占位符。对这个节点上锁。为什么上锁，防止 putVal 的时候向链表插入数据
            synchronized (f) {
                // 判断 i 下标处的桶节点是否和 f 相同
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;// low, height 高位桶，低位桶
                    // 如果 f 的 hash 值大于 0 。TreeBin 的 hash 是 -2
                    if (fh >= 0) {
                        // 对老长度进行与运算（第一个操作数的的第n位于第二个操作数的第n位如果都是1，那么结果的第n为也为1，否则为0）
                        // 由于 Map 的长度都是 2 的次方（000001000 这类的数字），那么取于 length 只有 2 种结果，一种是 0，一种是1
                        //  如果是结果是0 ，Doug Lea 将其放在低位，反之放在高位，目的是将链表重新 hash，放到对应的位置上，让新的取于算法能够击中他。
                        int runBit = fh & n;
                        Node<K,V> lastRun = f; // 尾节点，且和头节点的 hash 值取于不相等
                        // 遍历这个桶
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            // 取于桶中每个节点的 hash 值
                            int b = p.hash & n;
                            // 如果节点的 hash 值和首节点的 hash 值取于结果不同
                            if (b != runBit) {
                                runBit = b; // 更新 runBit，用于下面判断 lastRun 该赋值给 ln 还是 hn。
                                lastRun = p; // 这个 lastRun 保证后面的节点与自己的取于值相同，避免后面没有必要的循环
                            }
                        }
                        if (runBit == 0) {// 如果最后更新的 runBit 是 0 ，设置低位节点
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun; // 如果最后更新的 runBit 是 1， 设置高位节点
                            ln = null;
                        }// 再次循环，生成两个链表，lastRun 作为停止条件，这样就是避免无谓的循环（lastRun 后面都是相同的取于结果）
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            // 如果与运算结果是 0，那么就还在低位
                            if ((ph & n) == 0) // 如果是0 ，那么创建低位节点
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else // 1 则创建高位
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        // 其实这里类似 hashMap 
                        // 设置低位链表放在新链表的 i
                        setTabAt(nextTab, i, ln);
                        // 设置高位链表，在原有长度上加 n
                        setTabAt(nextTab, i + n, hn);
                        // 将旧的链表设置成占位符
                        setTabAt(tab, i, fwd);
                        // 继续向后推进
                        advance = true;
                    }// 如果是红黑树
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        // 遍历
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            // 和链表相同的判断，与运算 == 0 的放在低位
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            } // 不是 0 的放在高位
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        // 如果树的节点数小于等于 6，那么转成链表，反之，创建一个新的树
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        // 低位树
                        setTabAt(nextTab, i, ln);
                        // 高位数
                        setTabAt(nextTab, i + n, hn);
                        // 旧的设置成占位符
                        setTabAt(tab, i, fwd);
                        // 继续向后推进
                        advance = true;
                    }
                }
            }
        }
    }
}

```



## 四、计数



## 参考

1. [ConcurrentHashMap 1.8 源码分析](https://www.cnblogs.com/zerotomax/p/8687425.html#go7)
2. [sizeCtl 含义的辨析](https://blog.csdn.net/Unknownfuture/article/details/105350537)
3. [initTable 的问题](https://stackoverflow.com/questions/61722136/java-concurrenthashmap-initialization)
4. [Java Concurrent Hashmap initTable() Why the try/finally block?](https://stackoverflow.com/questions/63651473/java-concurrent-hashmap-inittable-why-the-try-finally-block)
5. [How does ConcurrentHashMap resizeStamp method work?](https://stackoverflow.com/questions/47175835/how-does-concurrenthashmap-resizestamp-method-work)
6. [关于 sc == rs + 1 || sc == rs + MAX_RESIZERS](https://stackoverflow.com/questions/53493706/how-the-conditions-sc-rs-1-sc-rs-max-resizers-can-be-achieved-in)

