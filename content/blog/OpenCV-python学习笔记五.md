---
title: OpenCV-python学习笔记五
date: 2020-02-22 17:51:07
tags: [OpenCV]
math: true
toc: true
---

## 一、图像梯度

在谈及梯度之前需要先找到函数，图片是二维的离散函数，二维意味着需要找到求梯度的方向，离散意味着对于图片的梯度不是导数而是差分。下式就是按照水平从左到右方向每隔一个像素点求差分：

$$ \Delta f\left ( i,j \right )=f\left ( i+1,j \right )-f\left ( i,j \right ) $$

### 1.1 Prewitt operator

将上式用卷积的方式处理是就可以是有下面这个卷积核（Prewitt 边缘检测算子）：

$$ Prewitt = \frac{1}{9}\left[ \begin{matrix} -1 & 0 & 1 \newline -1 & 0 & 1 \newline -1 & 0 & 1 \end{matrix} \right] $$

使用 Prewitt 算子处理后，值比较高的像素点意味着梯度较大，就是更解决边缘；值比较低的像素点意味着梯度较小，就是更解决平滑表面。

### 1.2 Sobel operator

考虑到对正在处理行的数据需要更多的重视，对 Prewitt 边缘检测算子改进就形成了 Sobel 边缘检测算子：

$$ Sobel_{x} = \frac{1}{9}\left[ \begin{matrix} -1 & 0 & 1 \newline -2 & 0 & 2 \newline -1 & 0 & 1 \end{matrix} \right] $$

如果是从上到下求差分那么算子就转换成：

$$ Sobel_{y} = \frac{1}{9}\left[ \begin{matrix} -1 & -2 & -1 \newline 0 & 0 & 0 \newline 1 & 2 & 1 \end{matrix} \right] $$

分别计算偏 x 方向的 $ G_{x} $，偏 y 方向的  $ G_{y} $，求绝对值，压缩到 [0, 255]区间，即 $ G\left ( x,y \right )=G_{x}+G_{y} $ 就是 sobel 边缘检测后的图像。

### 1.3 Scharr operator

Sobel 算子在转动上没有完美的对称。因此 Scharr 想要改进这个特性。它曾提出了更大的 5x5 核心，不过后来最常被使用的是:

$$ Scharr_{x} = \frac{1}{9}\left[ \begin{matrix} 3 & 0 & -3 \newline 10 & 0 & -10 \newline 3 & 0 & -3 \end{matrix} \right] $$

或者：

$$ Scharr_{y} = \frac{1}{9}\left[ \begin{matrix} 3 & 10 & 3 \newline 0 & 0 & 0 \newline -3 & -10 & -3 \end{matrix} \right] $$

### 1.4 Laplacian

上面两种算子都是针对图像这种二维离散函数求一阶差分得到的，而Laplace 算子则是求二阶差分。二阶差分定义如下：

$$ \Delta^{2} \left [f  \right ]\left ( i, j \right )=f\left ( i+1, j \right )-2f\left ( i, j \right )+f\left ( i-1, j \right ) $$

包含 x 轴、y 轴两个方向的卷积核为：

$$ Laplacian_{2} = \frac{1}{9}\left[ \begin{matrix} 0 & 1 & 0 \newline 1 & -4 & 1 \newline 0 & 1 & 0 \end{matrix} \right] $$

包含四个方向的卷积核为：

$$ Laplacian_{4} = \frac{1}{9}\left[ \begin{matrix} 1 & 1 & 1 \newline 1 & -8 & 1 \newline 1 & 1 & 1 \end{matrix} \right] $$

使用 Laplace 算子处理后，由于所求为二阶差分，值接近 0 的像素点更有可能是边缘，但是对于灰度值相近的区域，经过卷积后的值也很接近 0。

## 二、边缘检测

Canny边缘检测方法常被誉为边缘检测的最优方法。

```python
import cv2
import numpy as np

img = cv2.imread('handwriting.jpg', 0)
edges = cv2.Canny(img, 30, 70)  # canny边缘检测

cv2.imshow('canny', np.hstack((img, edges)))
cv2.waitKey(0)
```

`cv2.Canny()`进行边缘检测，参数2、3表示最低、高阈值。

### Canny边缘检测

Canny边缘提取的具体步骤如下：

1，使用5×5高斯滤波消除噪声：

边缘检测本身属于锐化操作，对噪点比较敏感，所以需要进行平滑处理。

$$ K=\frac{1}{256}\left[ \begin{matrix} 1 & 4 & 6 & 4 & 1 \newline 4 & 16 & 24 & 16 & 4 \newline 6 & 24 & 36 & 24 & 6 \newline 4 & 16 & 24 & 16 & 4 \newline 1 & 4 & 6 & 4 & 1 \end{matrix} \right] $$ 

2，计算图像梯度的方向：

首先使用Sobel算子计算两个方向上的梯度$ G_x $和$ G_y $，然后算出梯度的方向： $$ \theta=\arctan(\frac{G_y}{G_x}) $$ 保留这四个方向的梯度：0°/45°/90°/135°。

3，取局部极大值：

梯度其实已经表示了轮廓，但为了进一步筛选，可以在上面的四个角度方向上再取局部极大值：

比如，A点在45°方向上大于B/C点，那就保留它，把B/C设置为0。

4，滞后阈值：

经过前面三步，就只剩下0和可能的边缘梯度值了，为了最终确定下来，需要设定高低阈值：

- 像素点的值大于最高阈值，那肯定是边缘（上图A）
- 同理像素值小于最低阈值，那肯定不是边缘
- 像素值介于两者之间，如果与高于最高阈值的点连接，也算边缘，所以上图中C算，B不算

Canny推荐的高低阈值比在2:1到3:1之间。

具体原理请[参考](http://codec.wang/opencv-python-edge-detection/)。

### 阈值分割

其实很多情况下，阈值分割后再检测边缘，效果会更好：

```python
_, thresh = cv2.threshold(img, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)
edges = cv2.Canny(thresh, 30, 70)

cv2.imshow('canny', np.hstack((img, thresh, edges)))
cv2.waitKey(0)
```


# 参考

1. [wiki 差分](https://zh.wikipedia.org/wiki/差分)
2. [数字图像 - 边缘检测原理 - Sobel, Laplace, Canny算子](https://www.jianshu.com/p/2334bee37de5)
3. [图像分割之（五）边缘检测Laplace详解](https://blog.csdn.net/zhougynui/article/details/78837198)
4. [wiki soble](https://zh.wikipedia.org/wiki/索貝爾算子)
5. [codec.wang](http://codec.wang/opencv-python-edge-detection/)
6. [OpenCV](https://docs.opencv.org/master/d4/d86/group__imgproc__filter.html#gaa13106761eedf14798f37aa2d60404c9)

