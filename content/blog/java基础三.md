---
title: Java：反射和泛型
date: 2020-03-27 22:45:02
tags: [java]
toc: true
description: "java反射泛型动态代理"
keywords: [java, 反射, 泛型, 动态代理]
---

## 一、反射

由于 JVM 为每个加载的类创建了对应的 Class 实例，并在实例中保存了该类的所有信息，包括类名、包名、父类、实现的接口、所有方法、字段等，因此，如果获取了某个 Class 实例，我们就可以通过这个 Class 实例获取到该实例对应的类的所有信息。这种通过 Class 实例获取类信息的方法称为**反射（Reflection）**。

Java 反射主要提供以下功能：

- 在运行时判断任意一个对象所属的类；
- 在运行时构造任意一个类的对象；
- 在运行时判断任意一个类所具有的成员变量和方法（甚至可以调用 private 方法）；
- 在运行时调用任意一个对象的方法。

**重点：是运行时而不是编译时。**

### 1.1 Class 类

获取一个类的 Class 实例有三个方法：

（1）直接通过一个类的静态变量 class 获取：

```java
Class<?> cls = String.class;
```

（2）如果我们有一个实例变量，可以通过该实例变量提供的 `getClass()` 方法获取：

```java
String s = "Hello";
Class<?> cls = s.getClass();
```

（3）如果知道一个类的完整类名，可以通过静态方法 `Class.forName()` 获取：

```java
Class<?> cls = Class.forName("java.lang.String");
```

`Class.forName()` 加载类时默认会初始化，而 `ClassLoader.loadClass()` 默认**不会初始化**，只完成了类加载中的第一步加载。

注意 Class 实例 == 比较和 instanceof 的差别：

```java
Integer n = new Integer(123);

boolean b1 = n instanceof Integer; // true，因为n是Integer类型
boolean b2 = n instanceof Number; // true，因为n是Number类型的子类

boolean b3 = n.getClass() == Integer.class; // true，因为n.getClass()返回Integer.class
boolean b4 = n.getClass() == Number.class; // false，因为Integer.class!=Number.class
```

用 instanceof 不但匹配指定类型，还匹配指定类型的子类。而用 == 判断 class 实例可以精确地判断数据类型，但不能作子类型比较。通常情况下，我们应该用 instanceof 判断数据类型，因为面向抽象编程的时候，我们不关心具体的子类型。只有在需要精确判断一个类型是不是某个 class 的时候，我们才使用 == 判断 class 实例。

如果获取到了一个 Class 实例，我们就可以通过该 Class 实例来创建对应类型的实例：

```java
// 获取String的Class实例:
Class<?> cls = String.class;
// 创建一个String实例:
String s = (String) cls.newInstance();
```

上述代码相当于 `new String()`。通过 `Class.newInstance()` 可以创建类实例，它的局限是：只能调用 public 的无参数构造方法。带参数的构造方法，或者非 public 的构造方法都无法通过 `Class.newInstance()` 被调用。

### 1.2 Field

对任意的一个 Object 实例，只要获取了它的 Class 实例，就可以获取它的一切信息。Class 类提供了以下几个方法来获取字段：

- `Field getField(name)`：根据字段名获取某个 public 的 field（包括父类）；
- `Field getDeclaredField(name)`：根据字段名获取当前类的某个 field（不包括父类）；
- `Field[] getFields()`：获取所有 public 的 field（包括父类）；
- `Field[] getDeclaredFields()`：获取当前类的所有 field（不包括父类）。

调用 `Field.setAccessible(true)` 可以开放权限获取 private 修饰的字段，但这似乎会破坏类的封装，反射是一种非常规的用法，使用反射，首先代码非常繁琐，其次，它更多地是给工具或者底层框架来使用，目的是在不知道目标实例任何信息的情况下，获取特定字段的值。

此外，`setAccessible(true)` 可能会失败。如果 JVM 运行期存在 SecurityManager，那么它会根据规则进行检查，有可能阻止 `setAccessible(true)`。例如，某个 SecurityManager 可能不允许对 java 和 javax 开头的 package 的类调用 `setAccessible(true)`，这样可以保证 JVM 核心库的安全。

### 1.3 Method

可以通过 Class 实例获取所有 Method 信息。 Class 类提供了以下几个方法来获取 Method：

- `Method getMethod(name, Class...)`：获取某个 public 的 Method（包括父类）；
- `Method getDeclaredMethod(name, Class...)`：获取当前类的某个 Method（不包括父类）；
- `Method[] getMethods()`：获取所有 public 的 Method（包括父类）；
- `Method[] getDeclaredMethods()`：获取当前类的所有 Method（不包括父类）。

用反射调用方法：

```java
String r = (String) m.invoke(s, 6);
```

对 Method 实例调用 `invoke()` 就相当于调用该方法，`invoke()` 的第一个参数是对象实例，即在哪个实例上调用该方法，后面的可变参数要与方法参数一致，否则将报错。

调用静态方法时，由于无需指定实例对象，所以 `invoke()` 方法传入的第一个参数永远为 null:

```java
String r = (String) m.invoke(null, 6);
```

与访问字段相同，对于非 public 方法通过 `Method.setAccessible(true)` 允许其调用。

使用反射调用方法时，仍然遵循**多态原则**：即总是调用实际类型的覆写方法（如果存在）。

### 1.4 Constructor

通过 Class 实例获取 Constructor 的方法如下：

- `getConstructor(Class...)`：获取某个 public 的 Constructor；
- `getDeclaredConstructor(Class...)`：获取某个 Constructor；
- `getConstructors()`：获取所有 public 的 Constructor；
- `getDeclaredConstructors()`：获取所有 Constructor。

调用非 public 的 Constructor 时，必须首先通过 `setAccessible(true)` 设置允许访问。 `setAccessible(true)` 可能会失败。

## 二、泛型

在集合中存储对象并在使用前进行类型转换非常不方便。泛型防止了那种情况的发生。它提供了编译期的类型安全，确保你只能把正确类型的对象放入集合中，避免了在运行时出现 ClassCastException。

可以把 `ArrayList<Integer>` 向上转型为 `List<Integer>`，但不能把 `ArrayList<Integer>` 向上转型为 `ArrayList<Number>`。

### 2.1 静态泛型方法

对于静态方法，我们可以单独改写为“泛型”方法，只需要使用另一个类型即可：

```java
public class Pair<T> {
    private T first;
    private T last;
    public Pair(T first, T last) {
        this.first = first;
        this.last = last;
    }
    public T getFirst() { ... }
    public T getLast() { ... }

    // 静态泛型方法应该使用其他类型区分:
    public static <K> Pair<K> create(K first, K last) {
        return new Pair<K>(first, last);
    }
}
```

### 2.2 多个泛型类型

泛型还可以定义多种类型:

```java
public class Pair<T, K> {
    private T first;
    private K last;
    public Pair(T first, K last) {
        this.first = first;
        this.last = last;
    }
    public T getFirst() { ... }
    public K getLast() { ... }
}
```

使用的时候，需要指出两种类型：

```java
Pair<String, Integer> p = new Pair<>("test", 123);
```

Java 标准库的 Map 就是使用两种泛型类型的例子。它对 Key 使用一种类型，对 Value 使用另一种类型。

### 2.3 Type Erasure

泛型是通过**类型擦除**来实现的，编译器在编译时擦除了所有类型相关的信息，所以在运行时不存在任何类型相关的信息。例如 `List<String>` 在运行时仅用一个 List 来表示。这样做的目的，是确保能和 Java 5 之前的版本开发二进制类库进行兼容。你无法在运行时访问到类型参数，因为编译器已经把泛型类型转换成了原始类型。

- 编译器把类型 T 视为 Object；
- 编译器根据 T 实现安全的强制转型。

使用泛型的时候，我们编写的代码也是编译器看到的代码：

```java
Pair<String> p = new Pair<>("Hello", "world");
String first = p.getFirst();
String last = p.getLast();
```

而虚拟机执行的代码并没有泛型：

```java
Pair p = new Pair("Hello", "world");
String first = (String) p.getFirst();
String last = (String) p.getLast();
```

Java 泛型的**局限**：

**局限一：T 不能是基本类型**，例如 int，因为实际类型是 Object，Object 类型无法持有基本类型。

```java
Pair<int> p = new Pair<>(1, 2); // compile error!
```

**局限二：无法取得带泛型的 Class**，如下代码最后获取到的是同一个 Class，也就是 Pair 类的 Class。

```java
public class Main {
    public static void main(String[] args) {
        Pair<String> p1 = new Pair<>("Hello", "world");
        Pair<Integer> p2 = new Pair<>(123, 456);
        Class c1 = p1.getClass();
        Class c2 = p2.getClass();
        System.out.println(c1==c2); // true
        System.out.println(c1==Pair.class); // true

    }
}
```

**局限三：无法判断带泛型的 Class：**

```Java
Pair<Integer> p = new Pair<>(123, 456);
// Compile error:
if (p instanceof Pair<String>.class) {
}
```

**局限四：不能实例化 T 类型：**

```java
public class Pair<T> {
    private T first;
    private T last;
    public Pair() {
        // Compile error:
        first = new T();
        last = new T();
    }
}
```

上述代码无法通过编译，因为构造方法的两行语句：

```java
first = new T();
last = new T();
```

擦拭后实际上变成了：

```java
first = new Object();
last = new Object();
```

这样一来，创建`new Pair()`和创建`new Pair()`就全部成了 Object，显然编译器要阻止这种类型不对的代码。

要实例化 T 类型，我们必须借助额外的 Class 参数：

```java
public class Pair<T> {
    private T first;
    private T last;
    public Pair(Class<T> clazz) {
        first = clazz.newInstance();
        last = clazz.newInstance();
    }
}
```

上述代码借助 Class 参数并通过反射来实例化 T 类型，使用的时候，也必须传入 Class。例如：

```java
Pair<String> pair = new Pair<>(String.class);
```

因为传入了 Class 的实例，所以我们借助 String.class 就可以实例化 String 类型。

#### 1. 避免泛型覆写

有些时候，一个看似正确定义的方法会无法通过编译。例如：

```java
public class Pair<T> {
    public boolean equals(T t) {
        return this == t;
    }
}
```

这是因为，定义的 `equals(T t)` 方法实际上会被擦拭成 `equals(Object t)`，而这个方法是继承自 Object 的，编译器会阻止一个实际上会变成覆写的泛型方法定义。

换个方法名，避开与 `Object.equals(Object)` 的冲突就可以成功编译：

```java
public class Pair<T> {
    public boolean same(T t) {
        return this == t;
    }
}
```

#### 2. 泛型继承

在父类是泛型类型的情况下，编译器就必须把类型 T 保存到子类的 Class 文件中。

### 2.4 限定通配符

作为方法参数，`<? extends T>` 类型和 `<? super T>` 类型的区别在于：

- `<? extends T>` 允许调用读方法 `T get()` 获取 T 的引用，但不允许调用写方法 `set(T)` 传入 T 的引用（传入 null 除外）；
- ``<? super T>`` 允许调用写方法 `set(T)` 传入 T 的引用，但不允许调用读方法 `T get()` 获取 T 的引用（获取 Object 除外）。

一个是允许读不允许写，另一个是允许写不允许读。

#### 1. PECS原则

PECS原则（Producer Extends Consumer Super）可以帮助记忆何时使用 extends，何时使用 super。

如果需要返回 T，它是生产者（Producer），要使用 extends 通配符；如果需要写入 T，它是消费者（Consumer），要使用 super 通配符。以 Collections 的`copy()`方法为例：

```java
public class Collections {
    public static <T> void copy(List<? super T> dest, List<? extends T> src) {
        for (int i=0; i<src.size(); i++) {
            T t = src.get(i); // src是producer
            dest.add(t); // dest是consumer
        }
    }
}
```

需要返回 T 的 src 是生产者，因此声明为 `List<? extends T>`，需要写入 T 的 dest 是消费者，因此声明为 `List<? super T>`。

### 2.5 无限定通配符

因为 `<?>` 通配符既没有 extends，也没有 super，因此：

- 不允许调用 `set(T)` 方法并传入引用（null 除外）；
- 不允许调用 `T get()` 方法并获取 T 引用（只能获取 Object 引用）。

换句话说，既不能读，也不能写，那只能做一些 null 判断：

```java
static boolean isNull(Pair<?> p) {
    return p.getFirst() == null || p.getLast() == null;
}
```

大多数情况下，可以引入泛型参数 `<T>` 消除 `<?>` 通配符：

```java
static <T> boolean isNull(Pair<T> p) {
    return p.getFirst() == null || p.getLast() == null;
}
```

`<?>` 通配符有一个独特的特点，就是：`Pair<?>` 是所有 `Pair<T>` 的超类。

## 三、反射和泛型

Class 在实例化的时候，T 要替换成具体类。Class 它是个通配泛型，? 可以代表任何类型，所以主要用于声明时的限制情况。比如，我们可以这样做申明：

```java
// 可以
public Class<?> clazz;
// 不可以，因为 T 需要指定类型
public Class<T> clazzT;
```

调用 Class 的 `getSuperclass()` 方法返回的 Class 类型是 	`Class<? super String>`：

```java
Class<? super String> sup = String.class.getSuperclass();
```

## 四、静态代理和动态代理

- 静态代理在编译时就已经实现，编译完成后代理类是一个实际的 Class 文件
- 动态代理是在运行时动态生成的，即编译完成后没有实际的 Class 文件，而是在运行时动态生成类字节码，并加载到 JVM 中

### 4.1 静态代理

需要代理对象和目标对象实现一样的**接口**。

优点：可以在不修改目标对象的前提下扩展目标对象的功能。

缺点：

* 冗余。由于代理对象要实现与目标对象一致的接口，会产生过多的代理类；
* 不易维护。一旦接口增加方法，目标对象与代理对象都要进行修改。

### 4.2 动态代理

* 通过实现接口的方式 -> JDK 动态代理
* 通过继承类的方式 -> CGLIB 动态代理

#### 1. JDK 动态代理

基于 Java **反射**机制实现，**必须是实现了接口的业务类**才能用这种办法生成代理对象。

- 最小化依赖关系，减少依赖意味着简化开发和维护，JDK 本身的支持，可能比 cglib 更加可靠。
- 平滑进行 JDK 版本升级，而字节码类库通常需要进行更新以保证在新版 Java 上能够使用。
- 代码实现简单。

#### 2. CGLIB 动态代理

基于 [ASM](https://juejin.im/post/5b549bcbe51d45169c1c8b66) 机制实现，通过生成业务类的子类作为代理类。因为是继承，所以该类或方法最好不要声明成 final ，final 可以阻止继承和多态。

- 无需实现接口，达到代理类无侵入
- 只操作我们关心的类，而不必为其他相关类增加工作量。
- 高性能

## 参考

1. [廖雪峰的官方网站](https://www.liaoxuefeng.com/)
2. [深入解析 Java 反射](http://www.sczyh30.com/posts/Java/java-reflection-1/)
3. [Java 动态代理作用](https://www.zhihu.com/question/20794107)
4. [10 道 Java 泛型面试题](https://cloud.tencent.com/developer/article/1033693)
5. [JAVA 泛型中的通配符](https://juejin.im/post/5d5789d26fb9a06ad0056bd9)
6. [数组、泛型中的协变和逆变](https://www.jianshu.com/p/2bf15c5265c5)



