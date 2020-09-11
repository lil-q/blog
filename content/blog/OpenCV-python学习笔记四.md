---
title: OpenCV-python：亮度与对比度
date: 2020-02-20 14:14:07
tags: [OpenCV]
math: true
toc: true
description: "OpenCV-python和Numpy分析图片亮度和对比度"
keywords: [OpenCV, python, Numpy]
---

## 一、OpenCV与NumPy

OpenCV 读取一张图片，返回的实际上是一个`numpy.ndarray`类。所以一些对`numpy.ndarray`的操作可以直接对`cv2.imread()`返回的对象使用。

```python
import cv2
import numpy as np

img = cv2.imread('connelly.jpg')
print(type(img)) # <class 'numpy.ndarray'>

rows, cols = img.shape[:2]
print(rows, cols) # 210 270
```

### 1.1 访问和修改像素值

根据像素值的行和列坐标可以访问特定像素点。对于 BGR 图像，它返回一个蓝色、绿色、红色值数组。灰度图像，只返回相应的亮度。

```python
px = img[100, 120]
img_gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
px_gray = img_gray[100, 120]
print(px) # [130 131 181]
print(px_gray) # 146
```

可以用同样的方式修改像素值。

```python
img[100, 120] = [255,255,255]
```

NumPy 是一个用于快速数组计算的优化库。因此，使用原生 python 的数组简单地访问每个像素值，并修改它将非常缓慢，不推荐这种方法。

上述方法通常用于选择数组区域，例如前 5 行和最后 3 列。对于单个像素访问，但用 NumPy 数组的方法、`array.tem()`和`array.itemset()`会更适合。因为它总是返回一个标量。因此，如果想访问所有的 B，G，R 值，你需要为所有通道分别调用`array.tem()`。

```python
# 获取红色通道值
print(img.item(100, 120, 2)) # 181

# 修改红色通道值
img.itemset((100, 120, 2), 255)
```

### 1.2 访问图像属性

图像属性包括行数，列数和通道数，图像数据类型，像素数等。

`img.shape`可以访问图像的形状。 它返回一组行，列和通道的元组（如果图像是彩色的），如果图像是灰度，则返回的元组仅包含行数和列数。因此，检查加载的图像是灰度还是彩色图像是一种很好的方法。：

```python
print(img.shape) # (210, 270, 3)
```

`img.size`访问的像素总数：

```python
print(img.size) # 170100
```

图像数据类型由`img.dtype`获得：

```python
print(img.dtype) # uint8
```

 `img.dtype`在调试时非常重要，因为 OpenCV-Python 代码中的大量错误是由无效的数据类型引起的。

### 1.3 提取ROI

ROI (Region of Interest) 即感兴趣的区域，

```python
roi = img[100:200, 200:300]
```

* 直接指定范围即可，上面表示行数为`100-200`，列数为`200-300`的区域

### 1.4 拆分和合并图像通道

有时需要在 B,G,R 通道图像上单独工作。需要将 BGR 图像分割为单个平面或者需要将这些单独的通道连接到 BGR 图像。可以通过以下方式完成：

```python
b,g,r = cv2.split(img)
img = cv2.merge((b,g,r))
```

`cv2.split()` 是一项代价昂贵的的操作（就运算时间而言）。所以只有在需要时才使用这个方法。否则请使用 Numpy 索引。

```python
b = img[:,:,0]
```

假设，想要将所有红色像素设为零，不需要像这样分割并将其等于零。 可以简单地使用 Numpy 索引，这样更快。

```python
img[:,:,2] = 0
```

## 二、亮度与对比度

简单的改变图片的亮度和对比度可以是一个线性的过程：

$$ g\left (i,j  \right )= \alpha \cdot f\left ( i,j \right )+\beta\tag{1} $$

式子1 中 $ f\left ( i,j \right ) $ 相当于就是原图像每个像素点对应色彩或灰度的映射。直观上看，$ \alpha $ 决定了对比度，$ \alpha $ 越大，各像素之间的差别越大，对比度越大。$ \beta $ 决定了亮度，$ \beta $ 越大，各像素整体都变大，亮度提升了。

但其实这种表述并不准确，因为在对图像进行线性处理时存在溢出的问题，当 $ \beta $ 值太大时，$ g\left (i,j  \right ) $ 由于超过数据类型的上限被 OpenCV 置为特定值（比如 CV_8U 会被置为 255）。对于灰度图，这些值都被置为相同值，相当于这部分像素点对比度调节完全失效了；对于三通道的彩色图，有一部分通道溢出，则会导致对比度比预期小的现象。

另一个层面上，$ \alpha $ 的改变也影响了亮度，因为上式中 $ \alpha $ 的变化会统一的放大或缩小所有像素点的灰度值。当想要提高对比度时，如果不变化 $ \beta $ 的值，视觉上的”亮度“还是会提升。不妨把上式改写成如下式子：

$$ g\left (i,j  \right )= \alpha \cdot \left (f\left ( i,j \right )-T  \right )+\beta\tag{2} $$

当 $ \alpha $ 大于1时，式子2 表示将比 T 小的像素点的灰度值变得更小，把比 T 大的像素点的灰度值变得更大，这样当 T 值等于图像的平均灰度值（当然，是否就是平均灰度值也有待商榷）时就不会带来 $ \alpha $ 的改变影响亮度的问题了。式子1 实际上就是式子2 中 T 等于 0 的特例，所有像素点的灰度值都大于等于 0，所以整幅图像的亮度被 $ \alpha $ 影响了。

完整的调整代码如下：

```python
import cv2
import numpy as np

img = cv2.imread('connelly.jpg')

# 此处需注意，下面有说明
res = np.uint8(np.clip((2 * (np.int16(img) - 60) + 50), 0, 255))
tmp = np.hstack((img, res))  # 两张图片横向合并显示

cv2.imshow('image', tmp)
cv2.waitKey(0)
```

输出效果如下：

![](https://qttblog.oss-cn-hangzhou.aliyuncs.com/opencv/compare.jpg)

在用 python 改变图片的亮度和对比度时，有两个问题需要注意，第一个问题是数据类型，式子中其实有一个乘法，这是极有可能出现溢出的，如果代码是这样`res = np.uint8(np.clip((2 * img - 60) + 50), 0, 255))`，就会出现这样的结果:

![](https://qttblog.oss-cn-hangzhou.aliyuncs.com/opencv/compare_wrong.jpg)

解决的办法有两个，一是改变 img 的类型：使用`np.int16(img)`代替`img`；二是用浮点数`2.0`代替`2`，这样`2.0 * img`的数据类型会自动变成`float64`。

```python
print((2.0 * img).dtype) # float64
```

第二个问题是对于溢出 NumPy 和 OpenCV 的处理方式不同：

```python
x = np.uint8([250])
y = np.uint8([10])
print(cv2.add(x, y))  # 250 + 10 = 260 => 255 --255封顶
print(x + y)  # 250 + 10 = 260 % 256 = 4 --会溢出
```

如果我们使用 NumPy 处理就需要使用[`np.clip()`](https://docs.scipy.org/doc/numpy/reference/generated/numpy.clip.html#numpy.clip)函数将数据限定在 0~255。



# 参考

1. [numpy](https://www.numpy.org.cn/article/other/py_basic_ops.html#目标)
2. [opencv关于对比度和亮度的误解](https://blog.csdn.net/abc20002929/article/details/40474807)
3. [OpenCV tutorial](https://docs.opencv.org/2.4/doc/tutorials/core/basic_linear_transform/basic_linear_transform.html#basic-linear-transform)

