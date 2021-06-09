---
title: "Java并发：组合对象"
date: 2021-06-08T17:48:11+08:00
slug: ""
description: ""
keywords: []
tags: []
math: false
toc: true
---

【Java 并发】系列是对《Java Concurrency in Practice》的学习整理。

## 一、状态空间

首先要明确的是处理并发问题时，我们到底在保护什么？当然是一个对象中的 **field**。如果这些 field 都是基本类型（primitive）的，那么这些 field 就构成了这个对象的完整**状态**（state）；如果其中一个 field 指向另一个对象，那么这个状态应当还包括这个对象中的 field，以此类推。

所谓的保护就是要使每一个对象的状态保持在合理、安全的范围内，这个范围就是安全的**状态空间**（state space）。所以对于不会改动的 field，尽量使用 final 修饰使其不可变，这样可以减小状态空间，简化对对象可能状态的分析。分析包含两个部分：

* 始终通过**不变约束**（invariant）来判断某种**状态**是否合法；
* 通过**后验条件**（postcondition）来判断某种**状态转移**（state transition）是否合法。

不变约束确定了一个 field 的范围，或者说状态空间。比如一个 int 类型的 field 就需要始终保持在 [Integer.MIN_VALUE, Integer.MAX_VALUE] 之间；而一些 field 又需要满足不能是负值。

所谓后验条件，就是在**执行一段代码后**必须成立的条件，这个条件只能在运行期确定。比如对一个计数量自增：`count++`，就需要一些**同步策略**（synchronization policy）保证这个操作的原子性，否则这个状态转移就是不安全的。

同步策略定义了对象如何协调对其状态的访问，并且不会违背他的不变约束和后验条件。它将不可变性、线程限制和锁结合起来，从而维护线程的安全性。为了不违背不变约束，通常用一个synchronazed 函数对 field 进行封装，保证再写入时不会变成一个非法的状态；为了不违背后验条件，涉及状态转移的操作必须满足原子性。

```java
// 伪代码
public final class Counter { 
    
    public AtomicLong count = new AtomicLong(0); // 保证其原子性以满足后验条件
    
    private long value = 0;
    
    public synchronized long getValue() {
        return value;
    }

    public synchronized long increment() { // 用函数封装以满足不变约束
        if (value == Long.MAX_VALUE) 
            throw new IllegalStateException("counter overflow");
        return ++value; 
    } 
}
```

### 二、组合对象的线程安全

对于一个组合对象，状态空间更加复杂，维持其线程安全就会变得困难。下面是采用**监视器模式**（monitor pattern）来实现线程安全的例子——车辆跟踪器。采用这种方式，线程安全由 *MonitorVehicleTracker* 本身来保证。所以，涉及到私有对象 locations 发布的方法都必须用 synchronized 修饰。

为了不破坏封装性，`getLocations()` 返回的是 locations 对象的一个深拷贝。虽然 locations 本身是 final 修饰，不可变的，但是其内部的 MutablePoint 却是可变的。所以，在深拷贝时，总是创建一个新的 MutablePoint 加入到 result 中，最后返回一个不可修改的 result 封装对象。之所以要完成这些看似额外的操作，正是 MutablePoint 的可变性（线程不完全）导致的。

```java
@ThreadSafe
public class MonitorVehicleTracker { 
    
    private final Map<String, MutablePoint> locations;

    public MonitorVehicleTracker( 
        	Map<String, MutablePoint> locations) { 
        this.locations = deepCopy(locations);
	}

    // 需要 synchronized 同步
    public synchronized Map<String, MutablePoint> getLocations() { 
        return deepCopy(locations); // 深拷贝
    }

    // 需要 synchronized 同步
    public synchronized MutablePoint getLocation(String id) { 
        MutablePoint loc = locations.get(id); 
        return loc == null ? null : new MutablePoint(loc);
    }

    // 需要 synchronized 同步
    public synchronized void setLocation(String id, int x, int y) { 
        MutablePoint loc = locations.get(id); 
        if (loc == null) 
            throw new IllegalArgumentException("No such ID: " + id);
        loc.x = x; 
        loc.y = y;
    }

    private static Map<String, MutablePoint> 
        	deepCopy( Map<String, MutablePoint> m) {
        Map<String, MutablePoint> result = 
            	new HashMap<String, MutablePoint>();
        for (String id : m.keySet()) 
            result.put(id, new MutablePoint(m.get(id))); 
        return Collections.unmodifiableMap(result);
} }
```

```java
@NotThreadSafe 
public class MutablePoint { 
    
    public int x, y;

    public MutablePoint() { 
        x = 0; 
        y = 0; 
    } 
    
    public MutablePoint(MutablePoint p) { 
        this.x = p.x; 
        this.y = p.y;
    } 
}
```

如果将可变（线程不安全）的 MutablePoint 改为不可变（线程安全）的 Point，我们在获取 point 时，就不需要对其进行深拷贝，因为其他线程根本无法修改它。

```java
@Immutable 
public class Point { 
    
    public final int x, y;

    public Point(int x, int y) { 
        this.x = x; 
        this.y = y;
    } 
}
```

同样的思路，不妨把 locations 也设计成线程安全的，这样的话，对其发布时就不需要额外同步了。JDK 提供了 *ConcurrentMap* 作为线程安全的 *Map*。

```java
@ThreadSafe 
public class DelegatingVehicleTracker { 
    
    private final ConcurrentMap<String, Point> locations; 
    
    // 可以获取实时的数据
    private final Map<String, Point> unmodifiableMap;    

    public DelegatingVehicleTracker(Map<String, Point> points) { 
        locations = new ConcurrentHashMap<String, Point>(points); 
        unmodifiableMap = Collections.unmodifiableMap(locations);
    }

    public Map<String, Point> getLocations() { 
        return unmodifiableMap;
    }

    public Point getLocation(String id) { 
        return locations.get(id);
    }

    public void setLocation(String id, int x, int y) { 
        // 由于不可变，需要 new 一个新的 point
        if (locations.replace(id, new Point(x, y)) == null) 
            throw new IllegalArgumentException( 
            	"invalid vehicle name: " + id);
    } 
}
```

由于采用 *ConcurrentMap* 的 locations 本身是线程安全的，*DelegatingVehicleTracker* 就不需要额外的处理了，这种方式叫做**委托**（delegate）。同时需要注意到由于 *Point* 是不可变的，在 `setLocation()` 时，就需要用一个新的 point 进行替换了。

将 *Point* 设计成不可变的一大好处是 `getLocations()` 时，只需要返回对象构造时就创建好的 unmodifiableMap 即可。由于 unmodifiableMap 封装了 locations，所以通过 unmodifiableMap 就可以获得实时数据。当然，这种实时性也就意味着无法获得同一时间所有车辆的位置信息了（通过深拷贝则可以做到），实际使用中，应当权衡利弊选择模型。

### 2.1 委托的局限

