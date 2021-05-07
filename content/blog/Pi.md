---
title: "圆周率 π"
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

早在公元前三世纪，古希腊数学家阿基米德分别计算了单位圆的外切正 6 边形和内切正 6 边形的周长，以此来确定 π 的上界和下界。对于其中 $\sqrt{3}$ 的处理，阿基米德采用了如下不等式，却未加解释[[2]](https://math.ntnu.edu.tw/~horng/letter/vol4no2f.htm)。

$$\frac{265}{153}<\sqrt{3}<\frac{1351}{780}$$

通过这个不等式，避免了开根号的问题。为缩小 π 的范围，阿基米德不断倍增正多边形的边数，当逐步迭代至正 96 边形时，得到 π 介于 $\frac{223}{71} $ 和 $\frac{22}{7} $ 之间。

同时期，秦始皇统一中国，制定了度量衡标准。又过了五个世纪，魏晋时期的数学家刘徽首创割圆术，为计算圆周率建立了严密的理论和完善的算法。与阿基米德不同，刘徽只考虑圆的内切正多边形，并从面积的角度确定 π 上界[[3]](https://zh.wikipedia.org/wiki/%E5%89%B2%E5%9C%86%E6%9C%AF_(%E5%88%98%E5%BE%BD)#%E5%9C%86%E5%91%A8%E7%8E%87%E6%8D%B7%E6%B3%95)。当时采用筹算进行开平方，过程非常繁难，刘徽发明了一种快速算法，该算法对多边形的面积差做了等比近似，从而避免了最后四次最为复杂的开方。当计算至正 3072 边形时可得，π 近似于 3.1416。

五世纪，祖冲之在刘徽割圆术的基础上，计算得到 π 介于 3.1415926 和 3.1415927 之间。在之后的八百年里，这个结果在世界范围内都是最准确的。此外，祖冲之还通过调日法得出 π 的约率 $\frac{22}{7} $ 和密率 $\frac{355}{113} $，后者在五位数表示的分数中是最接近 π 的。

在 1630 年，奥地利天文学家克里斯托夫·格林伯格用 $10^{40}$ 边形将 π 计算到第 38 位小数，至今这仍是利用多边形算法可以达到的最准确的结果。

### 1.1 割圆术的计算机实现

借助计算机实现割圆法，由于开根号的精度有一定保障，可以计算出较为精准的 π 。为了简化计算，使用单位圆，半径取 1。

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

不同正多边形所能计算出的 π 值如下：

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

当割圆术难以取得进展时，与 π 相关的无穷级数被发现了。用无穷级数计算 π 有一个很明显的好处，不用再频繁开根号了。以下列举了一些可用于计算 π 无穷级数，他们的迭代速度也不尽相同。

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

1879 年的 Pi day，爱因斯坦出生了，是否能在 π 的小数部分找到他的生日呢？

如果将生日按年月日排列组成 8 位数字，如 18790314，然后在 π 中寻找，能找到的概率大概是 0.99995，这是在 π 中各数字完全随机出现的前提下得到的[[5]](https://www.angio.net/pi/whynotpi.html)。推广后得到在 n 位精度的 π 中找到 d 位数字的概率：

$$P  =  1- \left ( 1-0.1^{d}  \right )^{n}  $$

从 [MIT](https://stuff.mit.edu/afs/sipb/contrib/pi/) 可以下载精确到小数点后十亿位的 π，这是一个九百多兆的 pi-billion.txt 文件，里面记录了 “3.” 以及后面的十亿个数字。为了后续计算的方便，这里删去 “3.” 只留下 π 的小数部分。Java 中，*String* 类用一个字符数组 value 保存数据，所以字符串的最大长度受 value.length 限制。实际测试中，这个值比 Integer.MAX_VALUE 略小一些，十亿这个量级正好小于 Integer.MAX_VALUE。

用朴素的匹配方法可以实现 $O(nm)$ 的复杂度：

```java
static String PI = ... // 省略了 io 读取 PI 的过程，这是一个全局变量，后面还会用到
    
public static int findBirthday(String str) {
    for (int i = 0; i <= PI.length() - str.length(); i++) {
        int j = 0;
        while (str.charAt(j) == PI.charAt(i + j)) {
            if (++j == str.length()) return i + 1; // 从 1 开始计数
        }
    }
    return -1;
}
```

```txt
18830020
```

最后，匹配用时 34ms，在 π 小数点后的第 18830020 位找到了爱因斯坦的生日 18790314。在 [Pi-Search](https://www.angio.net/pi/piquery.html) 上可以验证这个答案，或者查找自己的生日。此外，Pi-Search 还提出了一些优化。

### 3.1 数据压缩

回头再看那个九百多兆的 pi-billion.txt 文件，它的具体大小是 976563KB，是否还能压缩呢？通过记事本打开这个文件，可以看到编码格式是 UTF-8，也就是说每一个字符（即 π 中的数字）都需要一个字节（8 bit）来记录。 976563KB 就是十亿个字节除以 1024 得到的。

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

### 3.2 减少比较次数

如果只是压缩数据，仍然采用一位一位地匹配，比较次数并没有减少，相应的运算量和运行时间也都相差无几。Pi-Search 给出了这样的优化：当 π 和匹配串字节对齐时，完全可以一次匹配两个数字，即一个字节；字节不对齐时，则提出单个的数字匹配，后续对齐部分同样按字节匹配。

```txt
Pi:      14 15 92 65 35 89 79 32 38 46 26 43...
Query:   65 35
         ^^ both checked in one step, because they're just one byte

Pi:      14 15 92 65 35 89 79 32 38 46 26 43...
Query:    6 53 5
          ^^^ still has to do multiple comparisons because they bytes don't line up
```

如果不限制输入字符串的位数，实现起来就比较复杂了，需要充分考虑奇偶情况。

```java
public static int findYourBirthday(byte[] pi, String str) {
    // 记录匹配串的开头和结尾
    int start = str.charAt(0) - 48;
    int end = (str.charAt(str.length() - 1) - 48) << 4;

    // 分别用于对齐匹配和不对齐匹配的字节数组
    byte[] birthday1 = new byte[str.length() / 2];
    byte[] birthday2 = new byte[(str.length() + 1) / 2 - 1];

    // 初始化匹配字节数组
    for (int i = 0; i < birthday1.length; i++) {
        birthday1[i] = (byte) (((str.charAt(2 * i) - 48) << 4)
                               + str.charAt(2 * i + 1) - 48);
    }
    for (int i = 0; i < birthday2.length; i++) {
        birthday2[i] = (byte) (((str.charAt(2 * i + 1) - 48) << 4)
                               + str.charAt(2 * i + 2) - 48);
    }

    // 奇偶需要分开讨论
    if ((str.length() & 1) == 0) {
        for (int i = 0; i < pi.length - str.length() / 2; i++) {
            if (match(pi, birthday1, i))
                return 2 * i + 1;
            if (start == (pi[i] & 15)
                && end == (pi[i + birthday1.length] & 240)
                && match(pi, birthday2, i + 1))
                return 2 * i + 2;
        }
        // 处理特例
        if (match(pi, birthday1, pi.length - birthday1.length))
            return 2 * (pi.length - birthday1.length) + 1;
    } else {
        for (int i = 0; i < pi.length - str.length() / 2; i++) {
            if (end == (pi[i + birthday1.length] & 240)
                && match(pi, birthday1, i)) 
                return 2 * i + 1;
            if (start == (pi[i] & 15)
                && match(pi, birthday2, i + 1)) 
                return 2 * i + 2;
        }
    }
    return -1;
}

private static boolean match(byte[] pi, byte[] birthday, int i) {
    for (int j = 0; j < birthday.length; j++) {
        if (pi[i + j] != birthday[j]) return false;
    }
    return true;
}
```

通过对 π 随机位上的测试，优化后的算法大概能提升 10% 左右的性能，并不是很理想。Pi-Searcher 还提到，他们针对现代 CPU 的流水线作业方式对算法进行了改进。比如，连续对四个数进行比较，而不是等待前一个比较出结果后才比较下一个。目前我还无法确定 Java 程序是否会自动优化这些步骤。

### 3.3 降低算法复杂度

虽然以上改进提升了一些速度，但是匹配模式始终是最朴素的——对 π 上的每一位进行匹配，每次匹配从左到右核对字符串。字符串匹配是一个很常见的问题，目前已经有了很多改进算法，比如 [Boyer-Moore](https://en.wikipedia.org/wiki/Boyer–Moore_string_search_algorithm) or [Knuth-Morris-Pratt](https://en.wikipedia.org/wiki/Knuth–Morris–Pratt_algorithm)，后者就是 KMP 算法。

KMP 算法首先要对匹配串进行预处理，找到每个位置上与自己的最长公共前缀。在匹配时，借助公共前缀可以省略一些重复的匹配（比较）。

```java
public static int kmpSearch(String str) {
    // 预处理
    int[] prefix = kmp(str);
    // 匹配
    int state = 0;
    for (int i = 0; i < PI.length(); i++) {
        while (state != 0 && PI.charAt(i) != str.charAt(state)) {
            state = prefix[state - 1];
        }
        if (PI.charAt(i) == str.charAt(state)) {
            if (++state == str.length())
                return i - str.length() + 2;
        }
    }
    return -1;
}

private static int[] kmp(String str) {
    int[] prefix = new int[str.length()];
    int state = 0;
    for (int i = 1; i < str.length(); i++) {
        while (state != 0 && str.charAt(i) != str.charAt(state)) {
            state = prefix[state - 1];
        }
        if (str.charAt(i) == str.charAt(state)) {
            prefix[i] = ++state;
        }
    }
    return prefix;
}
```

测试发现，KMP 算法比基于字符串的朴素算法还要慢 20%。其实这个结果也不意外，这类算法本质上是通过字符串上的一些规律来减少比较次数，而 π 本身没什么规律，算法本身的复杂度反而使其更慢了。

但是，如果把线性复杂度直接降到对数级，那结果就完全不一样了！这部分内容 Pi-Search 已经完成，借助[后缀数组](https://en.wikipedia.org/wiki/Suffix_array)和二分法能把复杂度降低到 $O(log_2n)$，当然也需要付出一些空间代价（保存后缀数组）。具体的做法如下：

* 当匹配串长度小于等于 5 时，用改进后的朴素算法。对于五位数，平均匹配到第 100,000 位就能匹配到。
* 当匹配串长度大于等于 6 时，采用后缀数组和二分法。匹配串可能会重复出现，要找到最先出现的仍然需要小范围的线性搜索。

## 参考

1. [piday](https://www.piday.org/)
2. [為什麼是阿基米德開平方法？](https://math.ntnu.edu.tw/~horng/letter/vol4no2f.htm)
3. [刘徽-wiki](https://zh.wikipedia.org/wiki/%E5%89%B2%E5%9C%86%E6%9C%AF_(%E5%88%98%E5%BE%BD)#%E5%9C%86%E5%91%A8%E7%8E%87%E6%8D%B7%E6%B3%95)
4. [pi-wiki](https://www.wikiwand.com/en/Approximations_of_%CF%80#/overview)
5. [Probability of Finding Strings In Pi](https://www.angio.net/pi/whynotpi.html)
6. [leetcode 上的 KMP 算法](https://leetcode-cn.com/problems/implement-strstr/solution/shi-xian-strstr-by-leetcode-solution-ds6y/)

