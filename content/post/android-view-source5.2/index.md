---
title: Activity绘制系列：5.2 MeasureSpec
description: 
slug: android-view-source5.2
date: 2022-03-28T04:14:54-08:00
categories:
    - Android
tags:
    - View绘制
---

## 概述

MeasureSpec，测量规格，用在View的measure过程，是一个int类型的值，但表示了两个属性：

- 测量模式（mode）
- 测量大小（size）

下面通过源码来讲解其是如何用一个int值表示两个属性的。

## 源码

先来看下View的onMeasure()：

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
		// ...
}
```

onMeasure()会传入两个int类型的值，分别是宽的测量规格和高的测量规格。

我们可以通过MeasureSpec.getMode(widthMeasureSpec)和MeasureSpec.getSize(widthMeasureSpec)来获取宽的测量模式和测量大小；

还可以通过MeasureSpec.makeMeasureSpec(size, mode)将size和mode合并成测量规格，并返回一个int值；

所以合并和分解的代码都在MeasureSpec这个类中，我们来看下源码：

```java
// MeasureSpec是定义在View类中的静态内部类
public class View implements Drawable.Callback, KeyEvent.Callback,
        AccessibilityEventSource {
		// ...
		public static class MeasureSpec {
      	// 移动30位
        private static final int MODE_SHIFT = 30;
      	// 掩码，高两位都是1，低30位都是0
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

        public @interface MeasureSpecMode {}

        /**
         * Measure specification mode: The parent has not imposed any constraint
         * on the child. It can be whatever size it wants.
         */
        public static final int UNSPECIFIED = 0 << MODE_SHIFT;// 高两位是00，低30位都是0

        /**
         * Measure specification mode: The parent has determined an exact size
         * for the child. The child is going to be given those bounds regardless
         * of how big it wants to be.
         */
        public static final int EXACTLY     = 1 << MODE_SHIFT;// 高两位是01，低30位都是0

        /**
         * Measure specification mode: The child can be as large as it wants up
         * to the specified size.
         */
        public static final int AT_MOST     = 2 << MODE_SHIFT;// 高两位是10，低30位都是0

        /**
         * Creates a measure specification based on the supplied size and mode.
         *
         * The mode must always be one of the following:
         * <ul>
         *  <li>{@link android.view.View.MeasureSpec#UNSPECIFIED}</li>
         *  <li>{@link android.view.View.MeasureSpec#EXACTLY}</li>
         *  <li>{@link android.view.View.MeasureSpec#AT_MOST}</li>
         * </ul>
         *
         * <p><strong>Note:</strong> On API level 17 and lower, makeMeasureSpec's
         * implementation was such that the order of arguments did not matter
         * and overflow in either value could impact the resulting MeasureSpec.
         * {@link android.widget.RelativeLayout} was affected by this bug.
         * Apps targeting API levels greater than 17 will get the fixed, more strict
         * behavior.</p>
         *
         * @param size the size of the measure specification
         * @param mode the mode of the measure specification
         * @return the measure specification based on size and mode
         */
      	// 把size和mode合并成测量规格
        public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size, int mode) {
            if (sUseBrokenMakeMeasureSpec) {
                return size + mode;
            } else {
              	// 1. 先用掩码分别与size、mode进行与运算，清洗数据；
              	// 2. 然后将清洗后的结果进行或运算合并到一起得到一个int值。
                return (size & ~MODE_MASK) | (mode & MODE_MASK);
              	// 问题：size | mode 直接进行或运算就可以合并在一起，那为啥还要各自先和掩码进行与运算呢？
              	// 答：因为size和mode是从外部传进来的，所以无法保证mode只用到高2位，size只用到低30位，所以要进行数据的清洗，方法就是与运算，mode和高2位是1的值进行与运算，size和低30位是1的值进行与运算，最后将计算完的结果进行或运算即可。
            }
        }

        

        /**
         * Extracts the mode from the supplied measure specification.
         *
         * @param measureSpec the measure specification to extract the mode from
         * @return {@link android.view.View.MeasureSpec#UNSPECIFIED},
         *         {@link android.view.View.MeasureSpec#AT_MOST} or
         *         {@link android.view.View.MeasureSpec#EXACTLY}
         */
        @MeasureSpecMode
        public static int getMode(int measureSpec) {
            //noinspection ResourceType
          	// 与高2位是1低30位是0的掩码进行与运算得到mode
            return (measureSpec & MODE_MASK);
        }

        /**
         * Extracts the size from the supplied measure specification.
         *
         * @param measureSpec the measure specification to extract the size from
         * @return the size in pixels defined in the supplied measure specification
         */
        public static int getSize(int measureSpec) {
          	// 与高2位是0低30位是1的掩码进行与运算得到size
            return (measureSpec & ~MODE_MASK);
        }
        // ...
    }
}
```

## 总结

具体分析都加到源码的注释里了，Android系统源码很多都有用到位运算的地方，使用位运算优点是高效，但缺点就是难懂，因为要转成二进制才能理解。

这里的MeasureSpec将size和mode两个属性通过位运算合并到一个int值里，可以减少些内存占用，但造成了理解上的障碍，是非功过，各位自行评说吧。