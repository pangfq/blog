---
title: Activity绘制系列：2. findViewById()
description: 
slug: android-view-source2
date: 2022-03-28T04:14:54-08:00
categories:
    - Android
tags:
    - View绘制
---

## 概述

在Activity中，我们知道，是通过findViewById()方法找到对应id的View实例，但内部是如何查找的呢？

## 源码

Activity的内部会调用Window的findViewById()，而Window内部又会调用DecorView的findViewById()，而DecorView是一个FrameLayout，所以最终调用的是ViewGroup的findViewById()；

findViewById()内部又会调用findViewTraversal()，该方法在ViewGroup和View中是两种不同的实现：

- View中是直接判断要查找的id是否是自己的id，如果是则返回自身，如果不是则返回null；

- ViewGroup中是通过一个for循环，遍历自己所有的子View，并调用各自的findViewById()；

综上所述，Activity的findViewById()其实就是**树的深度优先遍历**操作，时间复杂度为O(n)，快慢取决于View的个数和位置，是一个效率比较低的操作。

以下是源码分析：

```java
// 该方法定义在View中
public final <T extends View> T findViewById(@IdRes int id) {
				// 这个NO_ID = -1
        if (id == NO_ID) {
            return null;
        }
        // 另外调用findViewTraversal()方法，看方法名称就知道是通过遍历的方式
        return findViewTraversal(id);
}
// 这是View中的findViewTraversal()方法，因为是View，所以只判断了自己的id
protected <T extends View> T findViewTraversal(@IdRes int id) {
        if (id == mID) {
            return (T) this;
        }
        return null;
}
// 这是ViewGroup中的findViewTraversal()方法
protected <T extends View> T findViewTraversal(@IdRes int id) {
        if (id == mID) {
            return (T) this;
        }

        final View[] where = mChildren;
        final int len = mChildrenCount;
				// 可以看到，是通过一个for循环遍历自己的所有子View实现的
        for (int i = 0; i < len; i++) {
            View v = where[i];

            if ((v.mPrivateFlags & PFLAG_IS_ROOT_NAMESPACE) == 0) {
              	// 继续调用View或者ViewGroup的findViewById()
                v = v.findViewById(id);
								// 如果不为空，则说明找到了
                if (v != null) {
                    return (T) v;
                }
            }
        }
				// 循环正常结束，则说明没找到，则返回null
        return null;
}
```

## 总结

xml布局在经过setContentView()之后，会以View树的形式加载到内存，所以findViewById()查找指定id的View，其实就是在树的的所有结点中查找，也就是树的遍历，并且是深度优先遍历。

## 关键词

- View树
- 树的深度优先遍历









