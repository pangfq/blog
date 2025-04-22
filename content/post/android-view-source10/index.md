---
title: Activity绘制系列：10. postInvalidate()
description: 
slug: android-view-source10
date: 2022-03-28T04:14:54-08:00
categories:
    - Android
tags:
    - View绘制
---

## 概述

postInvalidate()和invalidate()的区别只有一个，就是postInvalidate()可以在子线程中进行调用刷新视图，因为其内部会通过主线程的Handler去处理调用，源码就不看了，比较简单，看名字也能大概猜到。

## 总结

这里把requestLayout()、invalidate()、postInvalidate()三个方法放在一块儿总结下：

- 如果想要重新测量、布局、绘制，则调用requestLayout()，但不要频繁调用；

- 如果只想重新绘制，并且在主线程，则调用invalidate()，如果在子线程，则调用postInvalidate()，要比调用requestLayout()更高效。