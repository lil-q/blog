---
title: "Java 并发：CopyOnWriteArrayList"
date: 2021-06-12T16:00:25+08:00
slug: ""
description: ""
keywords: []
tags: [java, concurrency, source_code]
math: false
toc: true
---

**写时复制**（**C**opy On **W**rite，**COW**）是一种程序设计领域的优化策略。其核心思想是，当多个调用者同时请求**相同资源**（如内存或磁盘上的数据）时，他们会共同获取**相同指针**指向相同的资源，知道某个调用者试图修改该资源，系统才会真正复制一个**专用副本**（private copy）给该调用者。对专用副本的修改是无法作用到原始资源上的，事实上，原始资源根本没有被**发布**。

COW 已有很多应用，比如在虚拟内存管理中，一般把共享访问的页面标记为只读，当一个 task 试图向内存中写入数据时，内存管理单元（MMU）抛出一个异常，内核处理该异常时为该 task 分配一份物理内存并复制数据到此内存，重新向 MMU 发出执行该 task 的写操作。Linux 等的文件管理系统也使用了写时复制策略。

Java 中的 *CopyOnWriteArrayList* 采用 COW 策略实现了线程安全版的 *ArrayList*。

```java
public class CopyOnWriteArrayList<E>
    	implements List<E>, RandomAccess, Cloneable, java.io.Serializable {

    // 同步锁
    final transient ReentrantLock lock = new ReentrantLock();

    // 私有数组，只能通过 getArray/setArray 发布
    // 需要 volatile 修饰满足新数组赋值后的可见性
    private transient volatile Object[] array;

    // final 禁止重写，避免逸出
    final Object[] getArray() {
        return array;
    }

    // 同上
    final void setArray(Object[] a) {
        array = a;
    }

    // 既然写入总是复制，初始化就没有确定数组大小的必要了
    public CopyOnWriteArrayList() {
        setArray(new Object[0]);
    }
    
    ...
}
```

## 一、读取时不加锁

 *CopyOnWriteArrayList* 的读取是不需要加锁的，由于多个线程读取的都是同一个不可变对象，读取是线程安全的。

```java
public E get(int index) {
    return get(getArray(), index);
}

private E get(Object[] a, int index) {
    return (E) a[index];
}
```

注意到 `get(int index)` 并不是直接从 array 中取得数据，而是利用 `getArray()` 来得到它。为什么要多此一举呢？其实在 *CopyOnWriteArraySet* 中，会有一个  *CopyOnWriteArrayList* 类的成员变量 al，而为了获取 al 中 array，就需要一定权限（至少为 package）。而 *CopyOnWriteArrayList* 类中的 array 是 private 权限，所以就用 package 权限的 setter 和 getter 来修改和访问 array。至于为什么不把 array 直接设定为 package 权限，可能是一定程度上破坏了封装性。

```java
public class CopyOnWriteArraySet<E> extends AbstractSet<E>
        implements java.io.Serializable {

    private final CopyOnWriteArrayList<E> al;
    
    public boolean containsAll(Collection<?> c) {
        return (c instanceof Set)
            ? compareSets(al.getArray(), (Set<?>) c) >= 0
            : al.containsAll(c);
    }
}
```

## 二、写时复制

当需要修改特定位置上的值时，首先应 `lock.lock()` 加锁。使用一个 final 修饰的局部变量来指向这个锁，这一操作看似多余，却是作者 Doug Lea 独特的编程风格， JIT 也许会对这一操作做出相应优化。接着，获取 array 相应位置上的元素，进行比较。如果不同，就创建一个相同长度的 newElements，将原先数据拷贝到 newElements 中并求改相应位置上的元素，`setArray(newElements)` 更新 array；如果相同，`setArray(elements)` 简单更新 array。最后 `lock.unlock()` 解锁。

```java
public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        E oldValue = get(elements, index);

        if (oldValue != element) {
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len);
            newElements[index] = element;
            setArray(newElements);
        } else {
            // Not quite a no-op; ensures volatile write semantics
            setArray(elements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

为什么 else 语句中明明没有修改任何东西，还是需要进行 `setArray(elements)` 呢？这是为了让调用方也能享受到 happens-before  原则中的 volatile 原则的好处：volatile 变量写入前的其他变量（不论 volatile 与否）的读写不会重排序到 volatile 变量写入之后。

```java
// initial conditions
int nonVolatileField = 0;
CopyOnWriteArrayList<String> list = /* a single String */

// Thread 1
nonVolatileField = 1;                 // (1)
list.set(0, "x");                     // (2)

// Thread 2
String s = list.get(0);               // (3)
if (s == "x") {
    int localVar = nonVolatileField;  // (4)
}
```

上面这个例子中，由于我们能够保证（2）一定对 volatile 变量进行了写入，所以当（2）先于（3）发生时，（1）一定先于（4）发生。

添加元素与修改类似，不过创建新的 newElements 数组时，长度需要加一。移除元素，则 newElements 数组长度减一。

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

```java
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);
        int numMoved = len - index - 1;
        if (numMoved == 0)
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index,
                             numMoved);
            setArray(newElements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

## 三、遍历

*CopyOnWriteArrayList* 下有一个子类 *COWIterator* 来实现迭代器。*COWIterator* 中持有一个 *CopyOnWriteArrayList* 的快照——即 *COWIterator* 创建时刻 *CopyOnWriteArrayList* 中的 array。

由于快照并不具有实时性，`remove()` \ `set()` \ `add()` 这些涉及到实时操作的都不能得到支持。也正是因为这个原因，即使迭代同时进行修改也不会报出 ` ConcurrentModificationException`。

```java
static final class COWIterator<E> implements ListIterator<E> {
    /** Snapshot of the array */
    private final Object[] snapshot;
    /** Index of element to be returned by subsequent call to next.  */
    private int cursor;

    private COWIterator(Object[] elements, int initialCursor) {
        cursor = initialCursor;
        snapshot = elements;
    }

    public boolean hasNext() {
        return cursor < snapshot.length;
    }

    public boolean hasPrevious() {
        return cursor > 0;
    }

    @SuppressWarnings("unchecked")
    public E next() {
        if (! hasNext())
            throw new NoSuchElementException();
        return (E) snapshot[cursor++];
    }

    @SuppressWarnings("unchecked")
    public E previous() {
        if (! hasPrevious())
            throw new NoSuchElementException();
        return (E) snapshot[--cursor];
    }

    public int nextIndex() {
        return cursor;
    }

    public int previousIndex() {
        return cursor-1;
    }

    /**
         * Not supported. Always throws UnsupportedOperationException.
         * @throws UnsupportedOperationException always; {@code remove}
         *         is not supported by this iterator.
         */
    public void remove() {
        throw new UnsupportedOperationException();
    }

    /**
         * Not supported. Always throws UnsupportedOperationException.
         * @throws UnsupportedOperationException always; {@code set}
         *         is not supported by this iterator.
         */
    public void set(E e) {
        throw new UnsupportedOperationException();
    }

    /**
         * Not supported. Always throws UnsupportedOperationException.
         * @throws UnsupportedOperationException always; {@code add}
         *         is not supported by this iterator.
         */
    public void add(E e) {
        throw new UnsupportedOperationException();
    }

    @Override
    public void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        Object[] elements = snapshot;
        final int size = elements.length;
        for (int i = cursor; i < size; i++) {
            @SuppressWarnings("unchecked") E e = (E) elements[i];
            action.accept(e);
        }
        cursor = size;
    }
}
```

## 参考

1. [Why CopyOnWriteArrayList use getArray() to access an array reference?](https://stackoverflow.com/questions/47030394/why-copyonwritearraylist-use-getarray-to-access-an-array-reference)
2. [The Java volatile Happens-Before Guarantee](http://tutorials.jenkov.com/java-concurrency/volatile.html)
3. [Performance of locally copied members ?](http://mail.openjdk.java.net/pipermail/core-libs-dev/2010-May/004165.html)

