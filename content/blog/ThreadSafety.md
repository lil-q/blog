---
title: "Java 并发：共享对象"
date: 2021-06-07T16:27:42+08:00
slug: ""
description: ""
keywords: [java, concurrency]
tags: []
math: false
toc: true
---

【Java 并发】系列是对《Java Concurrency in Practice》的学习整理。

## 一、发布和逸出

**发布**（publishing）对象就是使一个对象能够被外界代码使用。以下方式可以实现发布：

* 将对象保存到共享区域（*public static*）；
* 使用一个 *public* 方法返回该对象；
* 通过其他类的方法传入该对象。

### 1.1 共享区域

想要发布对象到共享区域，先要初始化一个容器用于存放这些对象：

```java
public static Set<Secret> knownSecrets; // 注意：存在风险

public void initialize() { 
    knownSecrets = new HashSet<Secret>();
}
```

然后把对象加入到这个共享的容器中，但是通过这个容器，别的线程就可以很轻易地获得 *Secret* 并对其进行修改，这是不合理的。如果这些 *Secret* 是不可变对象，就很好的解决了这个问题。

### 1.2  *public* 方法

用 *public* 方法发布一个对象时，同样不应将 *private* 对象直接暴露出来：

```java
class UnsafeStates { 
    
    private String[] states = new String[] { "AK", "AL" ...}; 
    
    public String[] getStates() { 
        return states; // 注意：存在风险
    } 
}
```

更合理的方式应该是进行拷贝：

```java
class UnsafeStates { 
    
    private String[] states = new String[] { "AK", "AL" ...}; 
    
    public String[] getStates() { 
        return Arrays.copyOf(states, states.length); 
    } 
}
```

### 1.3 传入对象

监听器模式就是通过事件源的函数传入 *Listener*，将其发布出去。

```java
public class ThisEscape { 
    
    // 注意：存在风险
    public ThisEscape(EventSource source) { 
        source.registerListener( 
            new EventListener() { // 匿名内部类
            	public void onEvent(Event e) { 
                	doSomething(e);
				} 
        	}); 
    } 
}
```

上述代码存在一个很大问题，即在构造函数中使用了匿名类。当 *ThisEscape* 下的匿名类发布的同时，会隐式发布 thisEscape 对象。由于此时 thisEscape 对象还没有构造完毕，`doSomething(e)` 可能会引出很多奇怪的问题。**不要再构造函数中使用匿名类**。

取而代之的，使用一个私有的构造函数和一个公共的工厂方法，可以避免不正确的创建。

```java
public class SafeListener { 
    
    private final EventListener listener;

    private SafeListener() { 
        listener = new EventListener() { 
            public void onEvent(Event e) { 
                doSomething(e);
            } 
        }; 
    }

    public static SafeListener newInstance(EventSource source) { 
        SafeListener safe = new SafeListener(); // 从构造函数返回，初始化完毕
        source.registerListener(safe.listener); 
        return safe;
    } 
}
```

通常情况下，由于封装性，我们并不希望对象被发布；而另一些情况下，又需要将一些对象发布出去供另一方使用，比如 *Listener*。所以发布也是**理想下的封装**对**实际情况**的一种妥协，这就会造成一些问题。如果一个对象还没有完全构造完就发布出去，这种情况称为**逸出**（escape）。

一个对象只有通过**构造函数返回后才处于可预测、稳定的状态**。所以要避免在构造函数中 this 引用的逸出，即使 this 引用是在构造函数最后一行发布的。

导致 this 引用逸出的一个常见错误就是在构造函数中启动一个线程。当对象在构造函数中创建一个线程，由于 `Thread` 或 `Runnable` 是对象类的内部类，this 引用几乎总是被新线程共享，导致逸出。正确的做法是在构造函数中创建线程，并不要启动它，而是把启动的过程放在类似 `start()` 或 `initialize()` 方法中。

## 二、线程封闭

对一些简单的常见，实际上并不需要共享数据，这时候可以采用**线程封闭**的方式。如果数据只在单线程中被访问，就不需要任何同步了。这在 Swing 和 JDBC 中都有应用。

利用 volatile 可以实现一种 Ad-hoc（不那么严格的）线程封闭，比如下面用于实现顺序打印的程序：

```java
class Foo {
    
    private volatile int state = 1;

    public Foo() {}

    public void first(Runnable printFirst) throws InterruptedException {
        printFirst.run();
        state = 2;
    }

    public void second(Runnable printSecond) throws InterruptedException {
        while (state != 2) { }; // 自旋
        printSecond.run();
        state = 3;
    }

    public void third(Runnable printThird) throws InterruptedException {
        while (state != 3) { }; // 自旋
        printThird.run();
        state = 1;
    }
}
```

虽然有多个线程共享 state，但是可以保证同一时间只有一个线程在写入。当然，state 必须要用 volatile 修饰以保证其可见性。

## 三、安全发布

为了安全地发布对象，对象的引用以及对象的状态必须同时对其他线程可见。一个正确创建的对象可以通过下列条件安全地发布：

* 通过静态初始化器初始化对象的引用；
* 将其引用存储到 volatile 域或 AtomicReference；
* 将其引用存储到正确创建的对象的 final 域中；
* 将其引用存储到由锁正确保护的域（如线程安全的容器）中。

