---
title: java：Map
date: 2020-04-02 15:37:30
tags: [java, source_code]
toc: true
description: "java常见Map"
keywords: [java, Map, HashMap, concurrentHashMap]
---

## 一、Map

Map 是一种键-值映射表，当我们调用`put(K key, V value)`方法时，就把 key 和 value 做了映射并放入 Map。当我们调用`V get(K key)`时，就可以通过 key 获取到对应的 value。如果 key 不存在，则返回 null。和 List 类似，Map 也是一个接口，最常用的实现类是 HashMap。

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/june/Map3.png"  />

- **TreeMap**：基于**红黑树**实现。
- **HashMap**：基于**哈希表**实现。
- **HashTable**：和 HashMap 类似，但它是线程安全的，这意味着同一时刻多个线程同时写入 HashTable 不会导致数据不一致。它是遗留类，不应该去使用它，而是使用 ConcurrentHashMap 来支持线程安全，ConcurrentHashMap 的效率会更高，因为 ConcurrentHashMap 引入了分段锁。
- **LinkedHashMap**：使用双向链表来维护元素的顺序，顺序为插入顺序或者最近最少使用（LRU）顺序。

### 1.1 Map 遍历

要遍历 key 可以使用`for each`循环遍历 Map 实例的`keySet()`方法返回的 Set 集合，它包含不重复的 key 的集合：

```java
for (String key : map.keySet()) {...}
```

同时遍历 key 和 value 可以使用`for each`循环遍历 Map 对象的`entrySet()`集合，它包含每一个 key-value 映射：

```java
for (Map.Entry<String, Integer> entry : map.entrySet()) {...}
```

### 1.2 equals() & hashCode()

正确使用 Map 必须保证：

1. 作为 key的对象必须正确覆写`equals()`方法，相等的两个 key 实例调用`equals()`必须返回`true`；
2. 作为 key 的对象还必须正确覆写`hashCode()`方法，且`hashCode()`方法要严格遵循以下规范：
   - 如果两个对象相等，则两个对象的`hashCode()`必须相等；
   - 如果两个对象不相等，则两个对象的`hashCode()`尽量不要相等。

自己写`hashCode()`时R 一般取 31，因为它是一个奇素数，如果是偶数的话，当出现乘法溢出，信息就会丢失，因为与 2 相乘相当于向左移一位，最左边的位丢失。并且一个数与 31 相乘可以转换成移位和减法：`31*x == (x<<5)-x`，编译器会自动进行这个优化。

```java
@Override
public int hashCode() {
    int result = 17;
    result = 31 * result + x;
    result = 31 * result + y;
    result = 31 * result + z;
    return result;
}
```

和实现`equals()`方法遇到的问题类似，如果传入值为 null，就会抛`NullPointerException`。为了解决这个问题，我们在计算`hashCode()`的时候，经常借助`Objects.hash()`来计算：

```java
int hashCode() {
    return Objects.hash(firstName, lastName, age);
}
```

所以，编写`equals()`和`hashCode()`遵循的原则是：`equals()`用到的用于比较的每一个字段，都**必须**在`hashCode()`中用于计算；`equals()`中没有使用到的字段，**绝不可**放在`hashCode()`中计算。

`Objects.hash()`内部实现实则为`Arrays.hashCode()`方法

```java
public static int hash(Object... values) {
    return Arrays.hashCode(values);
}
```

`Arrays.hashCode()`源码：

```java
public static int hashCode(Object a[]) {
    if (a == null)
        return 0;

    int result = 1;

    for (Object element : a)
        result = 31 * result + (element == null ? 0 : element.hashCode());

    return result;
}
```

注意 Objects.hash(Object...)，它的参数为不定参数，需要为 Object 对象。这会有以下一些影响：

1. 对基本类型做 hashCode 需要转换为包装类型，如 long 转换为 Long；
2. 会创建一个 Object 数组。

如果`hashCode()`方法被频繁调用的话，会有一定的性能影响。

### 1.3 hashCode()

`hashCode()`内部使用了数组，在初始化时默认的数组大小只有 16，任何 key 无论`hashCode()`多大都可以用类似方式处理：

```java
int index = key.hashCode() & 0xf;
```

当这个 hashMap 添加超过16个 key-value 时，hashMap 会自动扩容，没扩容一次容量就会翻倍。因为内存每次扩容会导致重新分配已有的 key-value，所以频繁扩容对 HashMap 的性能影响很大。如果确定要使用 n 个 key-value 的 HashMap，最好在创建时就指定容量：

```java
Map<String, Integer> map = new HashMap<>(n);
```

### 1.4 EnumMap

如果作为 key 的对象是 enum 类型，那么，还可以使用 Java 集合库提供的一种 EnumMap，它在内部以一个非常紧凑的数组存储 value，并且根据 enum 类型的 key 直接定位到内部数组的索引，并不需要计算`hashCode()`，不但效率最高，而且没有额外的空间浪费。

### 1.5 TreeMap

SortedMap 在遍历时严格按照 Key 的顺序遍历，最常用的实现类是 TreeMap。作为 SortedMap 的 Key 必须实现 Comparable 接口，或者传入 Comparator：

```java
Map<Person, Integer> map = new TreeMap<>(new Comparator<Person>() {
    public int compare(Person p1, Person p2) {
        return p1.name.compareTo(p2.name);
    }
});
```

注意到 Comparator 接口要求实现一个比较方法，它负责比较传入的两个元素 a 和 b:

* 如果 a < b，则返回负数，通常是 -1；
* 如果 a == b，则返回 0；
* 如果 a > b，则返回正数，通常是 1。

TreeMap 内部根据比较结果对 Key 进行排序。要严格按照`compare()`规范实现比较逻辑，否则，TreeMap 将不能正常工作。

[nowcoder面试题 找工作](https://www.nowcoder.com/practice/46e837a4ea9144f5ad2021658cb54c4d?tpId=98&tqId=32824&tPage=1&rp=1&ru=/ta/2019test&qru=/ta/2019test/question-ranking)

```java
import java.util.*;

public class Main {
    public static void main(String[] args) {
        Scanner jin = new Scanner(System.in);
        int N = jin.nextInt();
        int M = jin.nextInt();
        int[][] arr = new int[N][2];
        // 排序
        for (int i = 0; i < N; i++) {
            arr[i][0] = jin.nextInt();
            arr[i][1] = jin.nextInt();
        }
        Arrays.sort(arr, (e1, e2) -> e1[0] - e2[0]);
        // 找出难度对应的最大工资，并保存到TreeMap中
        TreeMap<Integer, Integer> map = new TreeMap();
        for (int i = 0; i < N; i++) {
            if (i > 0 && arr[i][1] < arr[i - 1][1]) arr[i][1] = arr[i - 1][1];
            map.put(arr[i][0], arr[i][1]);
        }
        // 利用.floorKey()找到最大的小于等于能力值的工作
        for (int i = 0; i < M; i++) {
            Integer index = map.floorKey(jin.nextInt());
            if (index == null) {
                System.out.println(0);
            } else {
                System.out.println(map.get(index));
            }
        }
    }
}
```

## 二、HashMap

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/june/Map2.png" alt="hashmap"  />

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {...}
```

默认的容量是 16：

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
```

### 2.1 储存结构

**（1）table**

HashMap 内部包含了一个 Node 类型的数组 table。

```java
transient Node<K,V>[] table;
```

Node 实现了 Map.Entry 接口，包含了四个字段，存储着键值对 Entry(key, value)，hash 和 next。数组 table 中的每个位置被当成一个桶，一个桶存放一个链表。HashMap 使用**拉链法**来解决冲突，同一个链表中存放**哈希值和散列桶取模运算结果相同**的 Node。

```java
/**N
* Basic hash bin node, used for most entries.  (See below for
* TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
*/
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```

**（2）put()**

public 修饰的`put()`方法调用了`putVal()`方法：

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 当table不存在或长度为0时，新建一个table
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 当桶下标对应的node不存在时，新建该Node加入table
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 当桶下标对应的Node已经存在时
    else {
        Node<K,V> e; K k;
        // key相同时
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // Node为TreeNode时
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // key不相同，采用尾插法插入Node
        else {
            //遍历链表
            for (int binCount = 0; ; ++binCount) {
                // 到链表末尾但没有相同key，则添加新Node
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 链表过长，将链表转为树结构
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 找到相同key
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // onlyIfAbsent 表示是否仅在 oldValue 为 null 的情况下更新键值对的值
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

**（3）get()**

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

**（4）遍历**

遍历所有的键时，首先要获取键集合 KeySet 对象，然后再通过  KeySet 的迭代器 KeyIterator 进行遍历。KeyIterator 类继承自 HashIterator 类，核心逻辑也封装在 HashIterator 类中。HashIterator 的逻辑并不复杂，在初始化时，HashIterator 先从桶数组中找到包含链表节点引用的桶。然后对这个桶指向的链表进行遍历。遍历完成后，再继续寻找下一个包含链表节点引用的桶，找到继续遍历。找不到，则结束遍历。

**（5）桶下标**

首先，传入`putVal()`的`hash(key)`处理如下，对于 key 为 null 的键值对，返回的 hash 值为 0；其余则是将所得 hashCode 右移 16 位后于原 hashCode 作异或处理：

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

在后续的处理中会根据 HashMap 的容量大小取余，得到桶下标：

```java
n = table.length;
i = (n - 1) & hash;
```

从上面的代码可以发现 hashMap 的 **size 取 2 的幂的好处**，再求桶下标时，只需要进行简单快速的位运算即可求余，且余数中的每一位都得到了保留，减少了哈希碰撞的可能。

**（6）拉链法**

JDK1.8 开始，HashMap 由链表的头插法改变成了**尾插法**，因此不再会造成死循环，改成尾插法也是为了能够更好的维护 jdk1.8 中 HashMap 的红黑树结构。

**（7）黑红树**

当链表变长时，查找和添加的速度会变慢，JDK1.8 后加入了链表转换为黑红树的机制，当链表长度超过 TREEIFY_THRESHOLD（默认为8）时，会转化成黑红树。

```java
static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;
```

在键类没有实现 comparable 接口的情况下，HashMap 会做以下三步处理：

1. 比较键与键之间 hash 值的大小，如果 hash 值相同，则
2. 检测键类是否实现了 Comparable 接口，是则调用 `compareTo()` 方法进行比较，否则
3. 如果仍未比较出大小，就需要进行仲裁了，仲裁方法为 tieBreakOrder。

**（8）加载因子**

初始容量 initialCapacity 是哈希表中初始桶的数量，加载因子 loadFactor 是哈希表在其容量自动扩容之前可以达到多满的一种度量。当哈希表中的条目数超出了**加载因子与当前容量的乘积**时，则要对该哈希表进行扩容、rehash 操作（即重建内部数据结构），扩容后的哈希表将具有**两倍**的原容量。

通常，加载因子需要**在时间和空间成本上寻求一种折衷**，默认为 0.75f。加载因子过高，例如为 1，虽然减少了空间开销，提高了空间利用率，但同时也增加了查询时间成本；加载因子过低，例如 0.5，虽然可以减少查询时间成本，但是空间利用率很低，同时提高了 rehash 操作的次数。在设置初始容量时应该考虑到映射中所需的条目数及其加载因子，以便最大限度地减少 rehash 操作次数，所以，一般在使用 HashMap 时建议根据预估值设置初始容量，减少扩容操作。

### 2.2 扩容

设 HashMap 的 table 长度为 M，需要存储的键值对数量为 N，如果哈希函数满足均匀性的要求，那么每条链表的长度大约为 N/M，因此查找的复杂度为 O(N/M)。

为了让查找的成本降低，应该使 N/M 尽可能小，因此需要保证 M 尽可能大，也就是说 table 要尽可能大。HashMap 采用动态扩容来根据当前的 N 值来调整 M 值，使得空间效率和时间效率都能得到保证。

```java
// 键值对的数量
transient int size;
// 装载因子，table 能够使用的比例，threshold = (int)(capacity* loadFactor)。
final float loadFactor;
// size 的临界值，当 size 大于等于 threshold 就必须进行扩容操作。
int threshold;
// 默认的装载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```

扩容使用 `resize()` 实现，需要注意的是，扩容操作同样需要把 oldTable 的所有键值对重新插入 newTable 中，包括链表和黑红树的拆分，因此这一步是很费时的。

在进行扩容时，需要把键值对重新计算桶下标，从而放到对应的桶上。在前面提到，HashMap 使用 hash % capacity 来确定桶下标。capacity 为 2 的 n 次方这一特点能够极大降低重新计算桶下标操作的复杂度。

假设原数组长度 capacity 为 16，扩容之后 new capacity 为 32：

```java
capacity     : 00010000
new capacity : 00100000
```

对于一个 Key，它的哈希值 hash 在第 5 位：

- 为 0，那么 hash%00010000 = hash%00100000，桶位置和原来一致；
- 为 1，hash%00010000 = hash%00100000 + 16，桶位置是原位置 + 16。

### 2.3 与  Hashtable  的比较

- Hashtable 使用 synchronized 来进行同步。
- HashMap 可以插入键为 null 的 Entry。
- HashMap 的迭代器是 fail-fast 迭代器。
- HashMap 不能保证随着时间的推移 Map 中的元素次序是不变的。

### 2.4 简易实现

```java
public class MyHashMap<K,V> {

    // 最大容量，即桶数组 table 的大小
    int capacity;

    // 当前 key-value 条目数
    int size = 0;

    // 负载因子，当 size >= table * loadFactor 时，扩容
    final float loadFactor;

    // 桶数组，存放 Entry 链表
    Entry<K,V>[] table;

    // 键值对 Entry，格外保存 hash 和 next
    static class Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Entry<K, V> next;

        Entry(int hash, K key, V value) {
            this.hash = hash;
            this.key = key;
            this.value = value;
        }
    }

    // 求 hash，对 Object 方法 HashCode() 得到的 hashcode 上下 16 位异或
    // 因为后续求桶下标实际上可能不会用到全部 hashcode，将高位异或到低位可以增加离散度
    static int hash(Object key) {
        int h = key.hashCode();
        return h ^ (h >>> 16);
    }

    public MyHashMap() {
        capacity = 16;
        loadFactor = 0.75f;
        table = new Entry[capacity];
    }

    public V get(Object key) {
        return getEntry(hash(key), key);
    }

    private V getEntry(int hash, Object key) {
        int index = hash & (capacity - 1);
        Entry<K,V> curr = table[index];
        while (curr != null) {
            if (curr.key.equals(key)) {
                return curr.value;
            }
            curr = curr.next;
        }
        return null;
    }

    public void put(K key, V value) {
        if (size >= capacity * loadFactor) {
            resize();
        }
        size++;
        putVal(hash(key), key, value);
    }

    private void putVal(int hash, K key, V value) {
        int index = hash & (capacity - 1);
        if (table[index] == null) {
            table[index] = new Entry<>(hash, key, value);
        } else {
            Entry<K, V> curr = table[index], prev = null;
            while (curr != null && !curr.key.equals(key)) {
                prev = curr;
                curr = curr.next;
            }
            if (curr == null) {
                prev.next = new Entry<>(hash, key, value);
            } else {
                curr.value = value;
            }
        }
    }

    private void resize() {
        capacity <<= 1;
        Entry<K,V>[] oldTable = table;
        table = new Entry[capacity];
        for (Entry entry : oldTable) {
            if (entry == null) continue;
            Entry curr = entry;
            while (curr != null) {
                // 注意到调用的是 putVal，这不会增加 size 的计数
                putVal(entry.hash, (K)entry.key, (V)entry.value);
                curr = curr.next;
            }
        }
    }
}
```



## 三、ConcurrentHashMap

### 3.1 存储结构

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/june/Map1.png" alt="img" style="zoom: 100%;" />

ConcurrentHashMap 和 HashMap 实现上类似，最主要的差别是 ConcurrentHashMap 采用了分段锁（Segment），每个分段锁维护着几个桶（HashEntry），多个线程可以同时访问不同分段锁上的桶，从而使其并发度更高（并发度就是 Segment 的个数）。

Segment 继承自 ReentrantLock。默认的并发级别为 16，也就是说默认创建 16 个 Segment。

```java
static final class HashEntry<K,V> {
    final int hash;
    final K key;
    volatile V value;
    volatile HashEntry<K,V> next;
}

static final int DEFAULT_CONCURRENCY_LEVEL = 16;
```

### 3.2 size 操作

每个 Segment 维护了一个 count 变量来统计该 Segment 中的键值对个数。

在执行 size 操作时，需要遍历所有 Segment 然后把 count 累计起来。

ConcurrentHashMap 在执行 size 操作时先尝试不加锁，如果连续两次不加锁操作得到的结果一致，那么可以认为这个结果是正确的。

尝试次数使用 RETRIES_BEFORE_LOCK 定义，该值为 2，retries 初始值为 -1，因此尝试次数为 3。

如果尝试的次数超过 3 次，就需要对每个 Segment 加锁。

### 3.3 JDK 1.8 的改动

JDK 1.7 使用分段锁机制来实现并发更新操作，核心类为 Segment，它继承自重入锁 ReentrantLock，并发度与 Segment 数量相等。

JDK 1.8 使用了 **CAS** 操作来支持更高的并发度，在 CAS 操作失败时使用内置锁 **synchronized**。

并且 JDK 1.8 的实现也在链表过长时会转换为**红黑树**。

#### 1. 读不加锁

我们通常使用读写锁来保护对一堆数据的读写操作。读时加读锁，写时加写锁。 ConcurrentHashMap 对数据的写操作是不需要分 2 次写的（没有中间状态），读操作也是不需要 2 次读取的。假如一个写操作需要分多次写，必然会有中间状态，如果读不加锁，那么可能就会读到中间状态，那就不对了。

虽然 ConcurrentHashMap 的读不需要锁，但是需要保证能读到最新数据，所以必须加 **volatile**。即数组的引用需要加 volatile，同时一个 Node 节点中的 **val** 和 **next** 属性也必须要加 volatile。

#### 2. 迁移中的并发处理

（1）在某个桶的**迁移过程**中，别的线程要对该桶进行 put 操作

一旦某个桶在迁移过程中了，必然要获取该桶的锁，所以其他线程的put操作要被阻塞，一旦迁移完毕，该桶中第一个元素就会被设置成 ForwardingNode 节点，所以其他线程 put 时需要重新判断下桶中第一个元素是否被更改了，如果被改了重新获取重新执行。

（2）某个桶已经**迁移完成（其他桶还未完成）**，别的线程要对该桶进行 put 操作

该线程会首先检查是否还有未分配的迁移任务，如果有则先去执行迁移任务，如果没有即全部任务已经分发出去了，那么此时该线程可以直接对新的桶进行插入操作（映射到的新桶必然已经完成了迁移，所以可以放心执行操作）。

## 四、LinkedHashMap

```java
public class LinkedHashMap<K,V> extends HashMap<K,V>
    implements Map<K,V> {...}
```

继承自 HashMap，因此具有和 HashMap 一样的快速查找特性。

```java
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

Entry 类继承自 HashMap.Node，可见内部维护了一个双向链表，用来维护插入顺序或者 LRU 顺序。

```java
// The head (eldest) of the doubly linked list.
transient LinkedHashMap.Entry<K,V> head;

// The tail (youngest) of the doubly linked list.
transient LinkedHashMap.Entry<K,V> tail;
```

accessOrder 决定了顺序，默认为 false，此时维护的是插入顺序。显式设置为 true，代表以访问顺序进行迭代。

```java
final boolean accessOrder;
```

LinkedHashMap 最重要的是以下用于维护顺序的函数，它们会在 `put()`、`get()` 等方法中调用。LinkedHashMap 并没有覆写`put()`，而是覆写了`afterNodeAccess(Node<K,V> p)`。

```java
void afterNodeAccess(Node<K,V> p) { }
void afterNodeInsertion(boolean evict) { }
```

### 4.1 afterNodeAccess()

当一个节点被访问时，如果 accessOrder 为 true，则会将该节点移到链表尾部。也就是说指定为 LRU 顺序之后，在每次访问一个节点时，会将这个节点移到链表尾部，保证链表尾部是最近访问的节点，那么链表首部就是最近最久未使用的节点。

```java
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```

### 4.2 afterNodeInsertion()

在 `put()` 等操作之后执行，当 `removeEldestEntry()` 方法返回 true 时会移除最晚的节点，也就是链表首部节点 first。evict 只有在构建 Map 的时候才为 false，在这里为 true。

```java
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}Copy to clipboardErrorCopied
```

`removeEldestEntry()` 默认为 false，如果需要让它为 true，需要继承 LinkedHashMap 并且覆盖这个方法的实现，这在实现 LRU 的缓存中特别有用，通过移除最近最久未使用的节点，从而保证缓存空间足够，并且缓存的数据都是热点数据。

```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```

### 4.3 实现LRU缓存

以下是使用 LinkedHashMap 实现的一个 LRU 缓存：

```java
class LRUCache<K, V> extends LinkedHashMap<K, V> { // 继承自LinkedHashMap
    // 最大缓存空间MAX_ENTRIES，默认值为3
    private static final int MAX_ENTRIES = 3;
    
    // 覆写
    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > MAX_ENTRIES;
    }
    
    // 初始化，将 accessOrder 设置为 true
    LRUCache() {
        super(MAX_ENTRIES, 0.75f, true);
    }
    
    public int get(int key) {
        return super.getOrDefault(key, -1);
    }
    
    public void put(int key, int value) {
        super.put(key, value);
    }
}
```

## 五、WeakHashMap

WeakHashMap 的 Entry 继承自 **WeakReference**，被 WeakReference 关联的对象在下一次垃圾回收时会被回收。

WeakHashMap 主要用来实现缓存，通过使用 WeakHashMap 来引用缓存对象，由 JVM 对这部分缓存进行回收。

```java
public class WeakHashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V> {...}
```

### 5.1 弱引用

当一个对象仅仅被 WeakReference 引用时，在下个垃圾收集周期时候该对象就会被回收，可以通过下面代码创建一个 WeakReference：

```java
Integer referent = 1;  
WeakReference<Integer> wr = new WeakReference<Integer>(referent);
referent = null;
```

当把 referent 赋值为 null 的时候，wr 会在下一个垃圾收集周期中被回收，因为已经没有强引用指向它。

### 5.2 ConcurrentCache

Tomcat 中的 ConcurrentCache 使用了 WeakHashMap 来实现缓存功能。

ConcurrentCache 采取的是分代缓存：

- 经常使用的对象放入 eden 中，eden 使用 ConcurrentHashMap 实现，不用担心会被回收（伊甸园）；
- 不常用的对象放入 longterm，longterm 使用 WeakHashMap 实现，这些老对象会被垃圾收集器回收。
- 当调用 get() 方法时，会先从 eden 区获取，如果没有找到的话再到 longterm 获取，当从 longterm 获取到就把对象放入 eden 中，从而保证经常被访问的节点不容易被回收。
- 当调用 put() 方法时，如果 eden 的大小超过了 size，那么就将 eden 中的所有对象都放入 longterm 中，利用虚拟机回收掉一部分不经常使用的对象。

## 六、补充

由于 Java 的集合设计非常久远，中间经历过大规模改进，我们要注意到有一小部分集合类是遗留类，不应该继续使用：

- Hashtable：一种线程安全的 Map 实现；
- Vector：一种线程安全的 List 实现；
- Stack：基于 Vector 实现的 LIFO 的栈。

还有一小部分接口是遗留接口，也不应该继续使用：

- Enumeration：已被 Iterator 取代。

Java的集合类定义在 java.util 包中，支持泛型。Java集合使用统一的 Iterator 遍历，尽量不要使用遗留接口。



# 参考

1. [CyC CS-Notes](https://cyc2018.github.io/CS-Notes/#/notes/Java 容器)
2. [廖雪峰java教程](https://www.liaoxuefeng.com/wiki/1252599548343744/1265118019954528)
3. [jdk1.8的HashMap和ConcurrentHashMap](https://my.oschina.net/pingpangkuangmo/blog/817973#h2_17)
4. [Java中的WeakHashMap](https://juejin.im/entry/5a085809f265da430c114c8b)
5. [Weak references - how useful are they?](https://stackoverflow.com/questions/7136620/weak-references-how-useful-are-they)
6. [AQS](https://zhuanlan.zhihu.com/p/86072774)