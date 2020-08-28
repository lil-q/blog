---
title: java-基础
date: 2020-03-20 19:33:14
tags: [java]
toc: true
description: "java数据类型运算方法"
keywords: [java, stream]
math: true
---

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/june/javase1.png" style="zoom:50%;" />

* **J**ava **V**irtual **M**achine，**JVM**：整个 java 实现跨平台的最核心的部分，所有的 java 程序会首先被编译为 .class 的类文件，这种类文件可以在虚拟机上执行，也就是说 class 并不直接与机器的操作系统相对应，而是经过虚拟机间接与操作系统交互，由虚拟机将程序解释给本地系统执行。
* **J**ava **R**untime **E**nvironment，**JRE**：java 运行环境。在 JDK 的安装目录里你可以找到 jre 目录，里面有两个文件夹 bin 和 lib ，在这里可以认为 bin 里的就是 jvm，lib 中则是 jvm 工作所需要的类库，而 bin 和 lib 和起来就称为 JRE。
* **J**ava **D**evelopment **K**it，**JDK**：java 开发工具包，是支持 java 程序开发的最小环境。JDK 对比 JRE 多了 java language 和工具 API。

## 一、数据类型

### 1.1 基本类型

- byte/8  [-128 ~ 127]
- char/16 
- short/16  [-32768 ~ 32767]
- int/32  [-2147483648 ~ 2147483647]
- float/32
- long/64  [-9223372036854775808 ~ 9223372036854775807]
- double/64
- boolean/~

Java语言对布尔类型的存储并没有做规定，因为理论上存储布尔类型只需要1 bit，但是通常JVM内部会把`boolean`表示为4字节整数，为了节省内存而表示为更小的类型可能会在之后处理时带来不必要的麻烦。

### 1.2 包装类型

基本类型都有对应的包装类型，基本类型与其对应的包装类型之间的赋值使用自动装箱与拆箱完成。

```java
Integer x = 2;     // 装箱 调用了 Integer.valueOf(2)
int y = x;         // 拆箱 调用了 X.intValue()
```

### 1.3 缓存池

new Integer(123) 与 Integer.valueOf(123) 的区别在于：

- new Integer(123) 每次都会新建一个对象；
- Integer.valueOf(123) 会使用缓存池中的对象，多次调用会取得同一个对象的引用。

编译器会在自动装箱过程调用 valueOf() 方法，因此多个值相同且值在缓存池范围内的 Integer 实例使用自动装箱来创建，那么就会引用相同的对象。

```java
Integer m = 123;
Integer n = 123;
System.out.println(m == n); // true
```

基本类型对应的缓冲池如下：

- boolean values true and false
- all byte values
- short values between -128 and 127
- int values between -128 and 127
- char in the range \u0000 to \u007F

在使用这些基本类型对应的包装类型时，如果该数值范围在缓冲池范围内，就可以直接使用缓冲池中的对象。

### 1.4 数组

#### 1. 初始化

Java语言中数组必须先初始化，然后才可以使用。

* **静态初始化**：`arrayName = new type[]{element1,element2,element3...};`

  简化写法：`type[] arrayName = {element1,element2,element3...};`

* **动态初始化**：`arrayName = new type[length];`

数组完成初始化后，内存空间中针对该数组的各个元素就有个一个默认值：

* 基本数据类型的整数类型（`byte`、`short`、`int`、`long`）默认值是`0`；
* 基本数据类型的浮点类型（`float`、`double`）默认值是`0.0`；
* 基本数据类型的字符类型（`char`）默认值是`'\u0000'`；
* 基本数据类型的布尔类型（`boolean`）默认值是`false`；
* 类型的引用类型（类、数组、接口、`String`）默认值是`null`。

**注意**：不要同时使用静态初始化和动态初始化，也就是说，不要在进行数组初始化时，既指定数组的长度，也为每个数组元素分配初始值。一旦数组完成初始化，数组在内存中所占的空间将被固定下来，所以数组的长度将不可改变。

#### 2. 打印数组内容

Java标准库提供了类方法`Arrays.toString()`，可以快速打印数组内容。

#### 3. 排序

调用JDK提供的`Arrays.sort()`可以原地升序排序。如需降序：

```java
Arrays.sort(a, Collections.reverseOrder());
```

## 二、运算

### 2.1 运算优先级

在Java的计算表达式中，运算优先级从高到低依次是：

- `()`
- `!` `~` `++` `--`
- `*` `/` `%`
- `+` `-`
- `<<` `>>` `>>>`
- `<` `<=` `>` `>=` `instanceof`
- `==` `!=`
- `&`
- `^`
- `|`
- `&&`
- `||`
- `? :`
- `+=` `-=` `*=` `/=`

### 2.2 位运算

位操作（Bit Manipulation）是程序设计中对位模式或二进制数的一元和二元操作。在许多古老的微处理器上，位运算比加减运算略快，通常位运算比乘除法运算要快很多。在现代架构中，情况并非如此：位运算的运算速度通常与加法运算相同（仍然快于乘法运算）。

[1. 面试题15. 二进制中1的个数](https://leetcode-cn.com/problems/er-jin-zhi-zhong-1de-ge-shu-lcof/)

```java
// 解一：
public class Solution {
    public int hammingWeight(int n) {
        int res = 0;
        while (n != 0) {
            res += n & 1; // 只计算一位
            n >>>= 1; // >>> 符号位一起移动，>> 符号位不移动
        }
        return res;
    }
}
// 解二：巧用 n&(n−1)
public class Solution {
    public int hammingWeight(int n) {
        int res = 0;
        while (n != 0) {
            res++;
            n &= n - 1;
        }
        return res;
    }
}
```

[2. 快速幂](https://leetcode-cn.com/problems/shu-zhi-de-zheng-shu-ci-fang-lcof/)

```java
class Solution {
    public double myPow(double x, int n) {
        if (n == 0) return 1;
        double p = 1;
        long m = n; // int 最小值转为正数会溢出，需要转为 long
        if (m < 0) {
            x = 1 / x;
            m = -m;
        }
        while (m > 0) {
            if ((m & 1) == 1) p *= x;
            x *= x;
            m >>>= 1;
        }
        return p;
    }
}
```

### 2.3 整数运算

#### 1. 隐式类型转换

`short`和`int`计算，结果总是`int`，原因是`short`首先自动被转型为`int`：

```java
short s = 1234;
int i = 123456;
int x = s + i; // s自动转型为int
```

因为字面量 1 是 int 类型，它比 short 类型精度要高，因此不能隐式地将 int 类型向下转型为 short 类型。

```java
short s1 = 1;
// s1 = s1 + 1;
```

但是使用 += 或者 ++ 运算符会执行隐式类型转换。

```java
s1 += 1;
s1++;
```

上面的语句相当于将 s1 + 1 的计算结果进行了向下转型：

```java
s1 = (short) (s1 + 1);
```

#### 2. 强制类型转换

可以将结果强制转型，即将大范围的整数转型为小范围的整数。强制转型使用`(类型)`，例如，将`int`强制转型为`short`：

```java
int i = 12345;
short s = (short) i; // 12345
```

要注意，超出范围的强制转型会得到错误的结果，原因是转型时，`int`的两个高位字节直接被扔掉，仅保留了低位的两个字节。

#### 3. 溢出

为了防止溢出，有些题目会设置答案求余数的要求，如[面试题10- I. 斐波那契数列](https://leetcode-cn.com/problems/fei-bo-na-qi-shu-lie-lcof/)要求答案需要取模 1e9+7（1000000007），这个数小于`int`最大值的一半，利用循环取余: $$ (x + y) \odot p = (x \odot p + y \odot p) \odot p $$

```java
class Solution {
    public int fib(int n) {
        int a = 0, b = 1, sum;
        for(int i = 0; i < n; i++){
            sum = (a + b) % 1000000007;
            a = b;
            b = sum;
        }
        return a;
    }
}
```

### 2.4 浮点数运算

浮点类型的数就是小数，因为小数用科学计数法表示的时候，小数点是可以“浮动”的，如 $1234.5$ 可以表示成 $12.345\times 10^{2}$，也可以表示成 $1.2345\times 10^{3}$，所以称为浮点数。

下面是定义浮点数的例子：

```java
float f1 = 3.14f;
float f2 = 3.14e38f; // 科学计数法表示的3.14x10^38
double d = 1.79e308;
double d2 = -1.79e308;
double d3 = 4.9e-324; // 科学计数法表示的4.9x10^-324
```

**对于`float`类型，需要加上`f`后缀**。浮点数可表示的范围非常大，`float`类型可最大表示$ 3.4\times10^{38}$，而`double`类型可最大表示 $1.79\times10^{308}$。

Java 不能隐式执行向下转型，因为这会使得精度降低。

```java
// float f = 1.1;
```

1.1 字面量属于 double 类型，不能直接将 1.1 直接赋值给 float 变量，因为这是向下转型。

#### 1. 浮点数的不准确性

浮点数运算和整数运算相比，只能进行加减乘除这些数值计算，不能做位运算和移位运算。

在计算机中，浮点数虽然表示的范围大，但是，浮点数有个非常重要的特点，就是浮点数常常**无法精确表示**。

浮点数 0.1 在计算机中就无法精确表示，因为十进制的 0.1 换算成二进制是一个无限循环小数，很显然，无论使用`float`还是`double`，都只能存储一个 0.1 的近似值。但是， 0.5 这个浮点数又可以精确地表示。（乘以若干个 2 后能成为整数的才能准确表示）

### 2.5 布尔运算

#### 1. 短路运算

布尔运算的一个重要特点是短路运算。如果一个布尔运算的表达式能提前确定结果，则后续的计算不再执行，直接返回结果。

因为`false && x`的结果总是`false`，无论`x`是`true`还是`false`，因此，与运算在确定第一个值为`false`后，不再继续计算，而是直接返回`false`。

#### 2. 三元运算符

Java还提供一个三元运算符`b ? x : y`，它根据第一个布尔表达式的结果，分别返回后续两个表达式之一的计算结果。

## 三、String

String 被声明为 final，因此它不可被继承。(Integer 等包装类也不能被继承）

在 Java 9 之后，String 类的实现改用 byte 数组存储字符串，同时使用 `coder` 来标识使用了哪种编码。

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final byte[] value;

    /** The identifier of the encoding used to encode the bytes in {@code value}. */
    private final byte coder;
}
```

value 数组被声明为 final，这意味着 value 数组初始化之后就不能再引用其它数组。并且 String 内部没有改变 value 数组的方法，因此可以保证 String 不可变。

### 3.1 编码

在Java中，`char`类型实际上就是两个字节的`Unicode`编码。如果我们要手动把字符串转换成其他编码，可以这样做：

```java
byte[] b1 = "Hello".getBytes(); // 按系统默认编码转换，不推荐
byte[] b2 = "Hello".getBytes("UTF-8"); // 按UTF-8编码转换
byte[] b2 = "Hello".getBytes("GBK"); // 按GBK编码转换
byte[] b3 = "Hello".getBytes(StandardCharsets.UTF_8); // 按UTF-8编码转换
```

注意：转换编码后，就不再是`char`类型，而是`byte`类型表示的数组。

如果要把已知编码的`byte[]`转换为`String`，可以这样做：

```java
byte[] b = ...
String s1 = new String(b, "GBK"); // 按GBK转换
String s2 = new String(b, StandardCharsets.UTF_8); // 按UTF-8转换
```

始终牢记：Java的`String`和`char`在内存中总是以 **Unicode** 编码表示。

### 3.2 不可变的好处

**1. 可以缓存 hash 值**

因为 String 的 hash 值经常被使用，例如 String 用做 HashMap 的 key。不可变的特性可以使得 hash 值也不可变，因此只需要进行一次计算。

**2. String Pool 的需要**

如果一个 String 对象已经被创建过了，那么就会从 String Pool 中取得引用。只有 String 是不可变的，才可能使用 String Pool。

**3. 安全性**

String 经常作为参数，String 不可变性可以保证参数不可变。例如在作为网络连接参数的情况下如果 String 是可变的，那么在网络连接过程中，String 被改变，改变 String 的那一方以为现在连接的是其它主机，而实际情况却不一定是。

**4. 线程安全**

String 不可变性天生具备线程安全，可以在多个线程中安全地使用。

### 3.3 StringBuffer & StringBuilder

```java
public class Main {
    public static void main(String[] args) {
        var sb = new StringBuilder(1024);
        sb.append("Mr ")
          .append("Bob")
          .append("!")
          .insert(0, "Hello, ");
        System.out.println(sb.toString());
    }
}
```

**1. 可变性**

- String 不可变
- StringBuffer 和 StringBuilder 可变

**2. 线程安全**

- String 不可变，因此是线程安全的
- StringBuilder 不是线程安全的
- StringBuffer 是线程安全的，内部使用 synchronized 进行同步

`StringBuffer`，这是Java早期的一个`StringBuilder`的线程安全版本，它通过同步来保证多个线程操作`StringBuffer`也是安全的，但是同步会带来执行速度的下降。现在完全没有必要使用`StringBuffer`。

### 3.4 StringJoiner

```java
import java.util.StringJoiner;

public class Main {
    public static void main(String[] args) {
        String[] names = {"Bob", "Alice", "Grace"};
        var sj = new StringJoiner(", ", "Hello ", "!");// 分隔符。头，尾
        for (String name : names) {
            sj.add(name);
        }
        System.out.println(sj.toString());
    }
}
```

### 3.5 String Pool

字符串常量池（String Pool）保存着所有字符串字面量（literal strings），这些字面量在编译时期就确定。不仅如此，还可以使用 String 的 intern() 方法在运行过程将字符串添加到 String Pool 中。

当一个字符串调用 intern() 方法时，如果 String Pool 中已经存在一个字符串和该字符串值相等（使用 equals() 方法进行确定），那么就会返回 String Pool 中字符串的引用；否则，就会在 String Pool 中添加一个新的字符串，并返回这个新字符串的引用。

下面示例中，s1 和 s2 采用 new String() 的方式新建了两个不同字符串，而 s3 和 s4 是通过 s1.intern() 方法取得同一个字符串引用。intern() 首先把 s1 引用的字符串放到 String Pool 中，然后返回这个字符串引用。因此 s3 和 s4 引用的是同一个字符串。

```java
String s1 = new String("aaa");
String s2 = new String("aaa");
System.out.println(s1 == s2);           // false
String s3 = s1.intern();
String s4 = s1.intern();
System.out.println(s3 == s4);           // true
```

如果是采用 "bbb" 这种字面量的形式创建字符串，会自动地将字符串放入 String Pool 中。

```java
String s5 = "bbb";
String s6 = "bbb";
System.out.println(s5 == s6);  // true
```

### 3.6 new String("abc")

使用这种方式一共会创建两个字符串对象（前提是 String Pool 中还没有 "abc" 字符串对象）。

- "abc" 属于字符串字面量，因此编译时期会在 String Pool 中创建一个字符串对象，指向这个 "abc" 字符串字面量；
- 而使用 new 的方式会在堆中创建一个字符串对象。

### 3.7 常用方法

```java
import java.util.Arrays;

public class Main {
    public static void main(String[] args) {
        String s1 = "hello";
        String s2 = "HELLO".toLowerCase();
        // 判断相同
        System.out.println(s1 == s2); // false
        System.out.println(s1.equals(s2)); // true
        // 判断为空
        System.out.println("".isEmpty()); // true
        System.out.println("  ".isEmpty()); // false
        // 判断为空白
        System.out.println("  \n".isBlank()); // true
        System.out.println(" Hello ".isBlank()); // false
        // 移除空白
        //   移除字符串首尾空白字符。空白字符包括空格，\t，\r，\n，
        //   trim()并没有改变字符串的内容，而是返回了一个新字符串。
        System.out.println("  \tHello\r\n ".trim()); // "Hello" 
        //   类似中文的空格字符\u3000也会被移除
        System.out.println("\u3000Hello\u3000".strip()); // "Hello"
        System.out.println("\u3000Hello\u3000".stripLeading()); // "Hello "
        System.out.println("\u3000Hello\u3000".stripTrailing()); // " Hello" 
        // 子串
        //   注意到contains()方法的参数是CharSequence而不是String，因为CharSequence是String的父类。
        System.out.println("Hello".contains("ll")); // true 
        System.out.println("Hello".indexOf("l")); // 2
        System.out.println("Hello".lastIndexOf("l")); // 3
        System.out.println("Hello".startsWith("He")); // true
        System.out.println("Hello".endsWith("lo")); // true
        System.out.println("Hello".substring(2)); // "llo"
        System.out.println("Hello".substring(2, 4)); // "ll"
        // 替换子串
        System.out.println("Hello".replace("el", "a")); // "Halo"
        // 分割字符串
        System.out.println(Arrays.toString("A,B,C,D".split(","))); // [A, B, C, D]
        // 拼接字符串
        String[] arr = {"A", "B", "C"};
        System.out.println(String.join(", ", arr)); // "A, B, C"
        // 类型转换
        System.out.println(String.valueOf(123)); // "123"
        System.out.println(String.valueOf(new Object())); // "java.lang.Object@5caf905d"
        System.out.println(Integer.parseInt("123")); // 123
        System.out.println(Integer.parseInt("ff", 16)); // 按十六进制转换，255
        //   如果修改了char[]数组，String并不会改变
        //   new String(char[])并不会直接引用传入的char[]数组，而是会复制一份
        char[] cs = "Hello".toCharArray(); // String -> char[]
        String s = new String(cs); // char[] -> String
        // 翻转
        System.out.println(new StringBuilder(s).reverse().toString()); // olleH
        // 判断是否为数字
        char ch = s.charAt(1); // e
        System.out.println(Character.isDigit(ch)); // false
    }
}
```

## 四、核心类

### 4.1 包装类型

Java核心库为每种基本类型都提供了对应的包装类型：

| 基本类型 | 对应的引用类型      |
| :------- | :------------------ |
| boolean  | java.lang.Boolean   |
| byte     | java.lang.Byte      |
| short    | java.lang.Short     |
| int      | java.lang.Integer   |
| long     | java.lang.Long      |
| float    | java.lang.Float     |
| double   | java.lang.Double    |
| char     | java.lang.Character |

#### 1. Auto Boxing

因为`int`和`Integer`可以互相转换：

```java
int i = 100;
Integer n = Integer.valueOf(i);
int x = n.intValue();
```

所以，Java编译器可以帮助我们自动在`int`和`Integer`之间转型：

```java
Integer n = 100; // 编译器自动使用Integer.valueOf(int)
int x = n; // 编译器自动使用Integer.intValue()
```

这种直接把`int`变为`Integer`的赋值写法，称为自动装箱（Auto Boxing），反过来，把`Integer`变为`int`的赋值写法，称为自动拆箱（Auto Unboxing）。

自动装箱和自动拆箱只发生在编译阶段，目的是为了少写代码。装箱和拆箱会影响代码的执行效率，因为编译后的`class`代码是严格区分基本类型和引用类型的。并且，自动拆箱执行时可能会报`NullPointerException`（比如对null自动拆箱）。

#### 2. 静态工厂方法

因为`Integer.valueOf()`可能始终返回同一个`Integer`实例（为了节省内存，`Integer.valueOf()`对于较小的数，始终返回相同的实例），因此，在我们自己创建`Integer`的时候，以下两种方法：

- 方法1：`Integer n = new Integer(100);`
- 方法2：`Integer n = Integer.valueOf(100);`

方法2更好，因为方法1总是创建新的`Integer`实例，方法2把内部优化留给`Integer`的实现者去做，即使在当前版本没有优化，也有可能在下一个版本进行优化。

我们把能创建“新”对象的静态方法称为**静态工厂方法**。`Integer.valueOf()`就是静态工厂方法，它尽可能地返回缓存的实例以节省内存。

### 4.2 枚举类型

```java
enum Weekday {
    SUN, MON, TUE, WED, THU, FRI, SAT;
}
```

#### 1. enum定义枚举的好处

* `enum`常量本身带有类型信息，即`Weekday.SUN`类型是`Weekday`，编译器会自动检查出类型错误。
* 不可能引用到非枚举的值，因为无法通过编译。
* 不同类型的枚举不能互相比较或者赋值，因为类型不符。

#### 2. enum和class的区别

- `enum`类型的每个常量在JVM中只有一个唯一实例，所以可以直接用`==`比较；
- 定义的`enum`类型总是继承自`java.lang.Enum`，且无法被继承；
- 只能定义出`enum`的实例，而无法通过`new`操作符创建`enum`的实例；
- 定义的每个实例都是引用类型的唯一实例；
- 可以将`enum`类型用于`switch`语句。

#### 3. name()

返回常量名，例如：

```java
String s = Weekday.SUN.name(); // "SUN"
```

#### 4. ordinal()

返回定义的常量的顺序，从0开始计数，例如：

```java
int n = Weekday.MON.ordinal(); // 1
```

## 五、方法

方法的名字的第一个单词应以小写字母作为开头，后面的单词则用大写字母开头写，不使用连接符。

下划线可能出现在 JUnit 测试方法名称中用以分隔名称的逻辑组件。例如 `testPop_emptyStack`。

### 5.1 参数

#### 1. 参数传递

Java的参数传递完全等同于赋值运算符的操作。[[3]](https://www.zhihu.com/question/31203609)

* 对于基本类型，赋值运算符会直接改变变量的值，原来的值被覆盖掉。
* 对于引用类型，赋值运算符会改变引用中所保存的地址，原来的地址被覆盖掉。**但是原来的对象不会被改变（重要）。**

#### 2. 可变参数

可变参数用`类型...`定义，可变参数相当于数组类型，可以把可变参数改写为`String[]`类型，但是，调用方需要自己先构造`String[]`，比较麻烦。另外可变参数可以保证无法传入`null`，因为传入0个参数时，接收到的实际值是一个空数组而不是`null`

### 5.2 构造方法

一个构造方法可以调用其他构造方法，这样做的目的是便于代码复用。调用其他构造方法的语法是`this(…)`。

### 5.3 初始化顺序

对象在class文件加载完毕，以及为各成员在方法区开辟好内存空间之后，就开始所谓“初始化"的步骤

1. **基类静态代码块，基类静态成员字段**
2. **派生类静态代码块，派生类静态成员字段**
3. **基类普通代码块，基类普通成员字段**
4. **基类构造函数**
5. **派生类普通代码块，派生类普通成员字段**
6. **派生类构造函数**

其中：1，2，3 和 5 并列优先级，按代码中出现先后顺序执行，1 和 2 只有第一次加载类时执行。

**静态代码块：**

1. 它是**随着类的加载而执行，只执行一次，并优先于主函数**。具体说，**静态代码块是由类调用**的。类调用时，先执行静态代码块，然后才执行主函数的。
2. **静态代码块其实就是给类初始化的，而构造代码块是给对象初始化的**。
3. 静态代码块中的变量是局部变量，与普通函数中的局部变量性质没有区别。

**构造代码块：**

1. 构造代码块的作用是给对象进行初始化。
2. **对象一建立就运行构造代码块了，而且优先于构造函数执行**。这里要强调一下，有对象建立，才会运行构造代码块，类不能调用构造代码块的，而且**构造代码块与构造函数的执行顺序是前者先于后者执行**。
3. 构造代码块与构造函数的区别是：**构造代码块是给所有对象进行统一初始化，而构造函数是给对应的对象初始化**，因为构造函数是可以多个的，运行哪个构造函数就会建立什么样的对象，但无论建立哪个对象，都会先执行相同的构造代码块。也就是说，构造代码块中定义的是不同对象共性的初始化内容。

**构造函数：**

1. **对象一建立，就会调用与之相应的构造函数**，也就是说，不建立对象，构造函数时不会运行的。
2. 构造函数的作用是用于给对象进行初始化。
3. 一个对象建立，构造函数只运行一次，而一般方法可以被该对象调用多次。



### 5.4 流程控制

#### 1. 输入

```java
import java.util.Scanner; // 导入类

public class Main {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in); // 创建Scanner对象
        System.out.print("Input your name: "); // 打印提示
        String name = scanner.nextLine(); // 读取一行输入并获取字符串
        System.out.print("Input your age: "); // 打印提示
        int age = scanner.nextInt(); // 读取一行输入并获取整数
        System.out.printf("Hi, %s, you are %d\n", name, age); // 格式化输出
    }
}
```

#### 2. 输出

`println`是print line的缩写，表示输出并换行。因此，如果输出后不想换行，可以用`print()`。格式化输出使用`System.out.printf()`，通过使用占位符`%?`，`printf()`可以把后面的参数格式化成指定格式

Java的格式化功能提供了多种占位符，可以把各种数据类型“格式化”成指定的字符串：

| 占位符 | 说明                             |
| :----- | :------------------------------- |
| %d     | 格式化输出整数                   |
| %x     | 格式化输出十六进制整数           |
| %f     | 格式化输出浮点数                 |
| %e     | 格式化输出科学计数法表示的浮点数 |
| %s     | 格式化字符串                     |

#### 3. if 语句

* 多个`if ... else`串联要特别注意判断顺序；

* 要注意`if`的边界条件；

* 要注意浮点数判断相等不能直接用`==`运算符；

  应该用`if (Math.abs(x - 0.1) < 0.00001) {...}`

* 引用类型判断内容相等要使用`equals()`，注意避免`NullPointerException`。

  应该用`if (s1 != null && s1.equals("hello")) {...}`

#### 4. switch 语句

* `switch`的计算结果必须是**整型、字符串或枚举类型**；
* `case`语句具有**穿透性**，注意千万不要漏写`break`；
* 总是写上`default`，建议打开`missing default`警告；
* 从Java 14开始，`switch`语句正式升级为表达式，不再需要`break`，并且允许使用`yield`返回值。

```java
public class Main {
    public static void main(String[] args) {
        String fruit = "orange";
        int opt = switch (fruit) {
            case "apple" -> 1;
            case "pear", "mango" -> 2;
            default -> {
                int code = fruit.hashCode();
                yield code; // switch语句返回值
            }
        };
        System.out.println("opt = " + opt);
    }
}
```

### 5.5 关键字

#### 1. final

**（1）数据**

声明数据为常量，可以是编译时常量，也可以是在运行时被初始化后不能被改变的常量。

- 对于基本类型，final 使数值不变；
- 对于引用类型，final 使引用不变，也就不能引用其它对象，但是被引用的对象本身是可以修改的。

```java
final int x = 1;
// x = 2;  // cannot assign value to final variable 'x'
final A y = new A();
y.a = 1;
```

**（2）方法**

声明方法不能被子类重写。

private 方法隐式地被指定为 final，如果在子类中定义的方法和基类中的一个 private 方法签名相同，此时子类的方法不是重写基类方法，而是在子类中定义了一个新的方法。

**（3）类**

声明类不允许被继承。

#### 2. static

**（1）静态变量**

- 静态变量：又称为类变量，也就是说这个变量属于类的，类所有的实例都共享静态变量，可以直接通过类名来访问它。静态变量在内存中只存在一份。
- 实例变量：每创建一个实例就会产生一个实例变量，它与该实例同生共死。

```java
public class A {

    private int x;         // 实例变量
    private static int y;  // 静态变量

    public static void main(String[] args) {
        // int x = A.x;  // Non-static field 'x' cannot be referenced from a static context
        A a = new A();
        int x = a.x;
        int y = A.y;
    }
}
```

**（2）静态方法**

静态方法在类加载的时候就存在了，它不依赖于任何实例。所以静态方法必须有实现，也就是说它不能是抽象方法。

```java
public abstract class A {
    public static void func1(){
    }
    // public abstract static void func2();  // Illegal combination of modifiers: 'abstract' and 'static'
}
```

只能访问所属类的静态字段和静态方法，方法中不能有 this 和 super 关键字，因此这两个关键字与具体对象关联。

```java
public class A {

    private static int x;
    private int y;

    public static void func1(){
        int a = x;
        // int b = y;  // Non-static field 'y' cannot be referenced from a static context
        // int b = this.y;     // 'A.this' cannot be referenced from a static context
    }
}
```

**（3）静态语句块**

静态语句块在类初始化时运行一次。

```java
public class A {
    static {
        System.out.println("123");
    }

    public static void main(String[] args) {
        A a1 = new A();
        A a2 = new A();
    }
}
// 123
```

**（4）静态内部类**

非静态内部类依赖于外部类的实例，也就是说需要先创建外部类实例，才能用这个实例去创建非静态内部类。而静态内部类不需要。

```java
public class OuterClass {

    class InnerClass {
    }

    static class StaticInnerClass {
    }

    public static void main(String[] args) {
        // InnerClass innerClass = new InnerClass(); // 'OuterClass.this' cannot be referenced from a static context
        OuterClass outerClass = new OuterClass();
        InnerClass innerClass = outerClass.new InnerClass();
        StaticInnerClass staticInnerClass = new StaticInnerClass();
    }
}
```

静态内部类不能访问外部类的非静态的变量和方法。

**（5）静态导包**

在使用静态变量和方法时不用再指明 ClassName，从而简化代码，但可读性大大降低。

```java
import static com.xxx.ClassName.*
```

### 5.6 stream

Stream API的特点是：

- Stream API提供了一套新的流式处理的抽象序列；
- Stream API支持函数式编程和链式操作；
- Stream可以表示无限序列，并且大多数情况下是惰性求值的。

#### 1. lambda

从Java 8开始，我们可以用Lambda表达式替换单方法接口`FunctionalInterface`，如果只有一行`return xxx`的代码，完全可以用更简单的写法：

```java
Arrays.sort(array, (s1, s2) -> s1.compareTo(s2));
```

返回值的类型也是由编译器自动推断的，这里推断出的返回值是`int`，因此，只要返回`int`，编译器就不会报错。

**[面试题40. 最小的k个数](https://leetcode-cn.com/problems/zui-xiao-de-kge-shu-lcof/)**

```java
public ArrayList<Integer> GetLeastNumbers_Solution(int[] nums, int k) {
    if (k > nums.length || k <= 0)
        return new ArrayList<>();
    PriorityQueue<Integer> maxHeap = new PriorityQueue<>((o1, o2) -> o2 - o1);
    for (int num : nums) {
        maxHeap.add(num);
        if (maxHeap.size() > k)
            maxHeap.poll();
    }
    return new ArrayList<>(maxHeap);
}
```

#### 2. 方法引用

如果某个方法**签名**和**返回类型**恰好一致，就可以直接传入方法引用。对于实例方法有一个隐含的`this`参数，比如`String`类的`compareTo()`方法在实际调用的时候，第一个隐含参数总是传入`this`。

#### 3. stream创建

创建`Stream`的方法有 ：

**（1）通过指定元素、指定数组、指定`Collection`创建`Stream`:**

```java
Stream<String> stream = Stream.of("A", "B", "C", "D");
Stream<String> stream1 = Arrays.stream(new String[] { "A", "B", "C" });
Stream<String> stream2 = List.of("X", "Y", "Z").stream();
```

**（2）通过`Supplier`创建`Stream`，可以是无限序列：**

```java
import java.util.function.*;
import java.util.stream.*;

public class Main {
    public static void main(String[] args) {
        Stream<Integer> natual = Stream.generate(new NatualSupplier());
        // 注意：无限序列必须先变成有限序列再打印:
        natual.limit(20).forEach(System.out::println);
    }
}

class NatualSupplier implements Supplier<Integer> {
    int n = 0;
    public Integer get() {
        n++;
        return n;
    }
}
```

**（3）通过其他类的相关方法创建：**

创建`Stream`的第三种方法是通过一些API提供的接口，直接获得`Stream`。

例如，`Files`类的`lines()`方法可以把一个文件变成一个`Stream`，每个元素代表文件的一行内容：

```java
try (Stream<String> lines = Files.lines(Paths.get("/path/to/file.txt"))) {
    ...
}
```

此方法对于按行遍历文本文件十分有用。

另外，正则表达式的`Pattern`对象有一个`splitAsStream()`方法，可以直接把一个长字符串分割成`Stream`序列而不是数组：

```java
Pattern p = Pattern.compile("\\s+");
Stream<String> s = p.splitAsStream("The quick brown fox jumps over the lazy dog");
s.forEach(System.out::println);
```

基本类型的`Stream`有`IntStream`、`LongStream`和`DoubleStream`。

#### 4. map()

`map()`方法接收的对象是`Function`接口对象，它定义了一个`apply()`方法，负责把一个`T`类型转换成`R`类型：

```java
<R> Stream<R> map(Function<? super T, ? extends R> mapper);
```

#### 5. filter()

使用`filter()`方法可以对一个`Stream`的每个元素进行测试，通过测试的元素被过滤后生成一个新的`Stream`。

```java
.filter(p -> p.score >= 60); // 找出及格的人
```

#### 6. reduce()

`map()`和`filter()`都是`Stream`的转换方法，而`Stream.reduce()`则是`Stream`的一个聚合方法，它可以把一个`Stream`的所有元素按照聚合函数聚合成一个结果：

```java
.reduce(0, (acc, n) -> acc + n); // 累加
.reduce(0, (acc, n) -> acc * n); // 累乘
```

#### 7. collect()

把`Stream`的每个元素收集到`List`的方法是调用`collect()`并传入`Collectors.toList()`对象，它实际上是一个`Collector`实例，通过类似`reduce()`的操作，把每个元素添加到一个收集器中（实际上是`ArrayList`）。类似的，`collect(Collectors.toSet())`可以把`Stream`的每个元素收集到`Set`中。

#### 8. toArray()

把Stream的元素输出为数组和输出为List类似，我们只需要调用`toArray()`方法，并传入数组的“构造方法”：

```java
List<String> list = List.of("Apple", "Banana", "Orange");
String[] array = list.stream().toArray(String[]::new);
```

注意到传入的“构造方法”是`String[]::new`，它的签名实际上是`IntFunction`定义的`String[] apply(int)`，即传入`int`参数，获得`String[]`数组的返回值。

#### 9. toMap()

```java
public class Main {
    public static void main(String[] args) {
        Stream<String> stream = Stream.of("APPL:Apple", "MSFT:Microsoft");
        Map<String, String> map = stream
                .collect(Collectors.toMap(
                        // 把元素s映射为key:
                        s -> s.substring(0, s.indexOf(':')),
                        // 把元素s映射为value:
                        s -> s.substring(s.indexOf(':') + 1)));
        System.out.println(map);
    }
}
```

# 参考

1. [CyC CS-Notes](https://github.com/CyC2018/CS-Notes)
2. [廖雪峰的官方网站](https://www.liaoxuefeng.com/)
3. [lxf-wiki](https://www.liaoxuefeng.com/wiki/1252599548343744)
4. [值传递还是引用传递](https://www.zhihu.com/question/31203609)
5. [初始化顺序](https://www.zhihu.com/question/49196023)
6. [静态代码块、构造代码块、构造函数](https://www.cnblogs.com/Qian123/p/5713440.html)



