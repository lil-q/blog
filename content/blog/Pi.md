---
title: "无理数 π"
date: 2021-04-23T16:17:53+08:00
slug: ""
description: "pi的历史，在pi中找你的生日"
keywords: [Pi, java]
tags: [math]
math: true
toc: true
---

每年的 3 月 14 日是 Pi Day[[1]](https://www.piday.org/)，由 π 的常用近似值 3.14 得来，活动当天，数学家们会一边吃 pie 一边讨论有关 π 的问题。

## 一、割圆术

早在公元前三世纪，古希腊数学家阿基米德分别计算了单位圆的外切正 6 边形和内切正 6 边形的周长，以此来确定 π 的上界和下界。对于其中 $\sqrt{3}$ 的处理，阿基米德采用了不等式：$\frac{265}{153}<\sqrt{3}<\frac{1351}{780}$，却未加解释[[2]](https://math.ntnu.edu.tw/~horng/letter/vol4no2f.htm)。为缩小 π 的范围，阿基米德不断倍增正多边形的边数，当逐步迭代至正 96 边形时，得到 π 介于 $\frac{223}{71} $ 和 $\frac{22}{7} $ 之间。

同时期，秦始皇统一中国，制定了度量衡标准。又过了五个世纪，魏晋时期的数学家刘徽首创割圆术，为计算圆周率建立了严密的理论和完善的算法。与阿基米德不同，刘徽只考虑圆的内切正多边形，并从面积的角度确定 π 上界[[3]](https://zh.wikipedia.org/wiki/%E5%89%B2%E5%9C%86%E6%9C%AF_(%E5%88%98%E5%BE%BD)#%E5%9C%86%E5%91%A8%E7%8E%87%E6%8D%B7%E6%B3%95)。当计算至正 3072 边形时可得，π 近似于 3.1416。

五世纪，祖冲之在刘徽割圆术的基础上，计算得到 π 介于 3.1415926 和 3.1415927 之间。在之后的八百年里，这个结果在世界范围内都是最准确的。

在 1630 年，奥地利天文学家克里斯托夫·格林伯格用 $10^{40}$ 边形将 π 计算到第 38 位小数，至今这仍是利用多边形算法可以达到的最准确的结果。

借助计算机实现割圆法（单位圆半径取 1），利用不同正多边形所能计算出的 π 值如下：

```java
public static void piAlgorithm(int n) {
    double a = 1; // 初始边长
    int k = 6; // 初始边数
    for (int i = 0; i < n; i++) {
        double h = Math.sqrt(1 - (a * a / 4));
        a = Math.sqrt((1 - h) * (1 - h) + (a * a / 4));
        k *= 2;
    }
    System.out.printf("pi = %.10f —— 正 %d 边形\n", (k * a / 2), k);
}
```

 ```txt
pi = 3.1058285412 —— 正 12 边形
pi = 3.1326286133 —— 正 24 边形
pi = 3.1393502030 —— 正 48 边形
pi = 3.1410319509 —— 正 96 边形 —— 阿基米德
pi = 3.1414524723 —— 正 192 边形
pi = 3.1415576079 —— 正 384 边形
pi = 3.1415838921 —— 正 768 边形
pi = 3.1415904632 —— 正 1536 边形
pi = 3.1415921060 —— 正 3072 边形  —— 刘徽
pi = 3.1415925167 —— 正 6144 边形
pi = 3.1415926194 —— 正 12288 边形 —— 祖冲之
pi = 3.1415926450 —— 正 24576 边形
 ```

## 二、π 与无穷级数

当割圆术难以取得进展时，与 π 相关的无穷级数被发现了。

1. [A Leibniz formula:](https://en.wikipedia.org/wiki/Leibniz_formula_for_pi)

   $$\pi \simeq 4\sum_{n=0}^{N} \cfrac {(-1)^n}{2n+1}.$$

2. [Bailey-Borwein-Plouffe series:](https://en.wikipedia.org/wiki/Bailey–Borwein–Plouffe_formula)

   $$\pi \simeq \sum_{n = 1}^{N} \left( \frac{1}{16^{n}} \left( \frac{4}{8n+1} - \frac{2}{8n+4} - \frac{1}{8n+5} - \frac{1}{8n+6} \right) \right).$$

3. [Ramanujan's formula:](https://en.wikipedia.org/wiki/Approximations_of_π#Efficient_methods)

   $$\frac{1}{\pi} \simeq \frac{2\sqrt{2}}{9801} \sum_{n=0}^N \frac{(4n)!(1103+26390n)}{(n!)^4 396^{4n}}.$$

4. [Chudnovsky brothers' formula:](https://en.wikipedia.org/wiki/Chudnovsky_algorithm)

   $$\frac{1}{\pi} \simeq 12 \sum_{n=0}^N \frac {(-1)^{n}(6n)!(545140134n+13591409)}{(3n)!(n!)^{3}640320^{{3n+3/2}}}.$$

目前，Chudnovsky brothers' formula 仍然是迭代最快（相对时间来说）的算法。截至 2020 年，计算机使用该算法将 π 计算到了小数点后 $5\times 10^{13}$ 位[[4]](https://www.wikiwand.com/en/Approximations_of_%CF%80#/overview)。

> 实际上，Chudnovsky brothers' formula 是楚德诺夫斯基兄弟对 Ramanujan's formula 的改进，而 Ramanujan's formula 是由印度天才数学家拉马努金凭借 “敏锐的洞察力” 提出，直到 1987 年才被 Borwein 兄弟证明。

那么计算 π 的意义是什么呢？按照约尔格·阿恩特（Jörg Arndt）及克里斯托夫·黑内尔（Christoph Haenel）的计算，39 个数位已足够运算绝大多数的宇宙学的计算需求，因为这个精确度已能够将可观测宇宙圆周的精确度准确至一个原子大小！

除了满足人类对打破纪录的欲望外，高精度 π 可以用来验证计算机的完备性和算法的正确性。在纯粹数学领域，计算的位数则可用来评定 π 的随机性。目前看来，在 π 的 $10^{12}$ 位中，每个数字分别出现了 $10^{11}$ 次，这佐证了 π 的正规性。

数学总是这样，现象如此诡谲，原理却总是质朴。当我们暂时还找不到 π 的原理时，只好继续研究其现象了。

## 三、在 π 中找到你的生日

1879 的 Pi day，爱因斯坦出生了，是否能在 π 的小数部分找到他的生日呢？

如果将生日按年月日排列组成 8 位数字，如 18790314，然后在 π 中寻找，能找到的概率大概是 0.99995，这是在 π 中各数字完全随机出现的前提下得到的[[5]](https://www.angio.net/pi/whynotpi.html)。推广后可以得到在 n 位精度的 π 中找到 d 位数字的概率：

$$P  =  1- \left ( 1-0.1^{d}  \right )^{n}  $$

从 [MIT](https://stuff.mit.edu/afs/sipb/contrib/pi/) 可以下载精确到小数点后十亿位的 π，这是一个九百多兆的 pi-billion.txt 文件，里面记录了 “3.” 以及后面的十亿个数字。为了后续计算的方便，这里删去 “3.” 只留下 π 的小数部分。Java 中，*String* 类用一个字符数组 value 保存数据，所以字符串的最大长度受 value.length 限制。实际测试中，这个值比 Integer.MAX_VALUE 略小一些。十亿这个量级正好小于 Integer.MAX_VALUE，可以用一个字符串来表示 π。写一个简单的双指针就可以完成复杂度 $O(n)$ 的字符串匹配算法了。

```java
String PI = ... // 省略了 io 读取 PI 的过程
    
public static int findBirthday(String str) {
    char[] chs = str.toCharArray();
    int p = 0; // 指向正在匹配的位
    for (int i = 0; i < PI.length(); i++) {
        if (chs[p] == PI.charAt(i)) {
            p++; // 匹配成功就进一位
        } else {
            p = 0; // 匹配失败从零开始
        }
        if (p == chs.length) {
            // 全部匹配则直接返回，注意：从 1 开始计数
            return i - chs.length + 2;
        }
    }
    return -1;
}
```

```txt
18830020
```

最后，在 π 小数点后的第 18830020 位找到了爱因斯坦的生日 18790314。在 [Pi-Search](https://www.angio.net/pi/piquery.html) 上可以验证这个答案，或者查找自己的生日。

Pi-Search 搜索的速度非常快，显然，上述方法还需要优化。

### 3.1 数据压缩

 回头看看那个九百多兆的 pi-billion.txt 文件，它的大小是 976563KB，是否还能压缩呢？通过记事本打开这个文件，可以看到编码格式是 UTF-8，也就是说每一个字符（也就是 π 中的数字）都需要一个字节（8 bit）来记录。 976563KB 就是十亿个字节除以 1024 得到的。

对于一个数字来说，用一个字节来存放其实浪费了空间，因为 4bit 就足够表示 0~9 这十个数字了。也就是说，一个字节可以存放两个数字——高四位和低四位分别存放一个数字。

基于这个理论，对数据压缩，并用一个字节数组来表示 π，具体的做法如下：

```java
public static void main(String[] args) throws IOException {
    // 读取文件
    File file = new File("pi-billion.txt");
    // 文件不存在时报错
    if (!file.exists()) throw new FileNotFoundException();
    // 从文件中读取字节数组
    byte[] pi = Files.readAllBytes(file.toPath());
    // 数据压缩
    byte[] newPi = recode(pi);
    // 写到一个新文件里，之后的匹配直接匹配这个新文件
    Files.write(Path.of("newPi.txt"), newPi);
}

private static byte[] recode(byte[] pi) {
    byte[] newPi = new byte[pi.length / 2];
    for (int i = 0; i < pi.length / 2; i++) {
        byte b1 = pi[2 * i];
        byte b2 = pi[2 * i + 1];
        // 把 b1 置于高四位，b2 置于低四位
        newPi[i] = (byte) (((b1 - 48) << 4) + b2 - 48);
    }
    return newPi;
}
```



## 参考

1. [piday](https://www.piday.org/)
2. [為什麼是阿基米德開平方法？](https://math.ntnu.edu.tw/~horng/letter/vol4no2f.htm)
3. [刘徽-wiki](https://zh.wikipedia.org/wiki/%E5%89%B2%E5%9C%86%E6%9C%AF_(%E5%88%98%E5%BE%BD)#%E5%9C%86%E5%91%A8%E7%8E%87%E6%8D%B7%E6%B3%95)
4. [pi-wiki](https://www.wikiwand.com/en/Approximations_of_%CF%80#/overview)
5. [Probability of Finding Strings In Pi](https://www.angio.net/pi/whynotpi.html)

