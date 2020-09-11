---
title: OpenCV-python：图像混合与形态学
date: 2020-02-19 13:00:33
tags: [OpenCV]
math: true
toc: true
description: "OpenCV-python图片混合和形态学操作"
keywords: [OpenCV, python, ori]
---

## 一、图像混合

### 1.1 图像混合

图像混合`cv2.addWeighted()`要求两幅图片的形状（高度/宽度/通道数）必须相同，$  \omega_{1} $ 和 $  \omega_{2} $ 分别为两幅图像的权重，b 则是一个偏置：

$$ dst = \omega_{1}\times img1+\omega_{2}\times img2 + b $$

```python
img1 = cv2.imread('img1_name.jpg')
img2 = cv2.imread('img2_name.png')
res = cv2.addWeighted(img1, 0.6, img2, 0.4, 0)
```

特别的，当 $  \omega_{1} $ 和 $  \omega_{2} $ 都为1，b 为 0 时，就是两张图片相加。可以用`cv2.add()`函数。numpy 中可以直接用`res = img1 + img2`相加，但这两者的结果并不相同：

```python
x = np.uint8([250])
y = np.uint8([10])
print(cv2.add(x, y))  # 250 + 10 = 260 => 255 --255封顶
print(x + y)  # 250 + 10 = 260 % 256 = 4 --会溢出
```



### 1.2 位操作

按位操作包括按位与/或/非/异或操作，如果将两幅图片直接相加会改变图片的颜色，如果用图像混合，则会改变图片的透明度，所以我们需要用按位操作。

* `cv2.bitwise_and()`：按位与操作 
* `cv2.bitwise_not()`：按位非操作
* `cv2.bitwise_or()`：按位或操作
* `cv2.bitwise_xor()`：按位异或操作

通过位操作，我们可以得到[掩膜](https://baike.baidu.com/item/掩膜/8544392?fr=aladdin)（mask）——用来对另外一幅图片进行局部的遮挡的二值化图片，通过掩膜把原图中要放 logo 的区域抠出来，再把 logo 放进去就行了：

```python
import cv2

img1 = cv2.imread('lena.jpg')
img2 = cv2.imread('opencv-logo-white.png')

# 把logo放在左上角，所以我们只关心这一块区域
rows, cols = img2.shape[:2]
roi = img1[:rows, :cols]

# 创建掩膜
img2gray = cv2.cvtColor(img2, cv2.COLOR_BGR2GRAY)
ret, mask = cv2.threshold(img2gray, 10, 255, cv2.THRESH_BINARY)
mask_inv = cv2.bitwise_not(mask)

# 保留除logo外的背景
img1_bg = cv2.bitwise_and(roi, roi, mask=mask_inv)
dst = cv2.add(img1_bg, img2)  # 进行融合
img1[:rows, :cols] = dst  # 融合后放在原图上

cv2.imshow("dst",img1)
cv2.waitKey(10000)
```

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/opencv/2020-02-20%20124157.png" style="zoom:50%;" />

## 二、形态学操作



### 2.1 腐蚀膨胀

腐蚀的效果是把图片”变瘦”，其原理是在原图的小区域内取局部最小值。因为是二值化图，只有 0 和 255 ，所以小区域内有一个是 0 该像素点就为 0 。

![](https://qttblog.oss-cn-hangzhou.aliyuncs.com/opencv/cv2_understand_erosion.jpg)

```python
# 腐蚀
img = cv2.imread('j.bmp', 0)
kernel = np.ones((5, 5), np.uint8)
erosion = cv2.erode(img, kernel)  
```

* `kernel`：结构元素，因为形态学操作其实也是应用卷积来实现的。结构元素可以是矩形/椭圆/十字形，可以用`cv2.getStructuringElement()`来生成不同形状的结构元素：
  * 矩形结构：`cv2.getStructuringElement(cv2.MORPH_RECT, (5, 5))`  
  * 椭圆结构：`cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (5, 5))`  
  * 十字形结构：`cv2.getStructuringElement(cv2.MORPH_CROSS, (5, 5))`

![](https://qttblog.oss-cn-hangzhou.aliyuncs.com/opencv/cv2_morphological_struct_element.jpg)

### 2.2 膨胀

膨胀与腐蚀相反，取的是局部最大值，效果是把图片”变胖”：

```python
# 膨胀
dilation = cv2.dilate(img, kernel)  
```

### 2.3 开运算

先腐蚀后膨胀叫开运算（因为先腐蚀会分开物体，这样容易记住），其作用是：分离物体，消除小区域。这类形态学操作用`cv2.morphologyEx()`函数实现：

```python
kernel = cv2.getStructuringElement(cv2.MORPH_RECT, (5, 5))  # 定义结构元素

img = cv2.imread('j_noise_out.bmp', 0)
opening = cv2.morphologyEx(img, cv2.MORPH_OPEN, kernel)  # 开运算
```

### 2.4 闭运算

闭运算则相反：先膨胀后腐蚀（先膨胀会使白色的部分扩张，以至于消除/“闭合”物体里面的小黑洞，所以叫闭运算）

```python
img = cv2.imread('j_noise_in.bmp', 0)
closing = cv2.morphologyEx(img, cv2.MORPH_CLOSE, kernel)  # 闭运算
```

![](https://qttblog.oss-cn-hangzhou.aliyuncs.com/opencv/cv2_morphological_opening_closing.jpg)

### 2.5 其他形态学操作

- 形态学梯度：膨胀图减去腐蚀图，`dilation - erosion`，这样会得到物体的轮廓：

```python
img = cv2.imread('school.bmp', 0)
gradient = cv2.morphologyEx(img, cv2.MORPH_GRADIENT, kernel)
```

- 顶帽：原图减去开运算后的图：`src - opening`

```python
tophat = cv2.morphologyEx(img, cv2.MORPH_TOPHAT, kernel)
```

- 黑帽：闭运算后的图减去原图：`closing - src`

```python
blackhat = cv2.morphologyEx(img, cv2.MORPH_BLACKHAT, kernel)
```

## 参考

1. [codec.wang](http://codec.wang/opencv-python-erode-and-dilate/)