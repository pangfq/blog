---
title: Activity绘制系列：6. onLayout()
description: 
slug: android-view-source6
date: 2022-03-28T04:14:54-08:00
categories:
    - Android
tags:
    - View绘制
---

## 概述

layout过程是确定视图的位置，也就是确定4个参数，left、top、right、bottom，这4个参数是相对于所在父视图的位置，所以layout的过程是确定位置，而且是确定当前视图所在父视图的位置。

依然从ViewRootImpl类开始看起，这样可以看全。

## 源码

```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {

          private void performTraversals() {
            	// ...
            	// mWidth、mHeight是通过上面measure后得到的宽和高
    					performLayout(lp, mWidth, mHeight);
            	// ...
			    }
          
          private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
            int desiredWindowHeight) {
            	// ...
            	final View host = mView;
			        if (host == null) {
  	  		        return;
        			}
            	// 可以看到，left和top都是0，也就是说DecorView的左上角与父视图的左上角重合，DecorView的父视图就是屏幕了，剩下right和bottom分别是width和height，因为right-left=width，而left=0，所以right=width，height同理。
            	host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
            	// ...
        	}
}
```

```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {		
		@Override
    public final void layout(int l, int t, int r, int b) {
        if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {
            if (mTransition != null) {
                mTransition.layoutChange(this);
            }
          	// ViewGroup调用的是View的layout()
         
            super.layout(l, t, r, b);
        } else {
            // record the fact that we noop'd it; request layout when transition finishes
            mLayoutCalledWhileSuppressed = true;
        }
    }
}
```

```java
public class View implements Drawable.Callback, KeyEvent.Callback,
        AccessibilityEventSource {
          
		public void layout(int l, int t, int r, int b) {
        // ...
      	// 先通过setFrame(l, t, r, b)确定自身的位置
        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
          	// 再调用onLayout()确定子View的位置
            onLayout(changed, l, t, r, b);
        }
      	// 可以看到layout流程和measure正好相反，measure是确定所有子View的宽高，最后再确定自己的宽高
      	// layout是先确定自己的位置，再确定所有子View的位置
      	// ...
    }
          
    protected boolean setFrame(int left, int top, int right, int bottom) {
      	// ...
        mLeft = left;
        mTop = top;
        mRight = right;
        mBottom = bottom;
      	// ...
    }
          
}
```

因为onLayout()在View和ViewGroup中都是抽象方法，具体实现在子类中，我们还是以LinearLayout为例看看其onLayout()的源码：

```java
public class LinearLayout {
	
	@Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
      	// 根据方向属性分成两个方法
        if (mOrientation == VERTICAL) {
            layoutVertical(l, t, r, b);
        } else {
            layoutHorizontal(l, t, r, b);
        }
    }
  
  void layoutVertical(int left, int top, int right, int bottom) {
        // ...

    		// for循环遍历所有子View
        for (int i = 0; i < count; i++) {
            final View child = getVirtualChildAt(i);
            if (child == null) {
                childTop += measureNullChild(i);
            } else if (child.getVisibility() != GONE) {
                // ...
                setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                        childWidth, childHeight);
                // 可以看到下一个子View的top，就要再加上当前View的height，也就是按照竖直方向排列子View
	              childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);
              	// ...
            }
        }
    }

}
```

## 总结

对于layout过程，也可以分为两部分：

- ViewGroup
- View

同样的ViewGroup也要分为DecorView和非DecorView，区别同样是构造传参的差异；

对于DecorView来讲，父视图就是屏幕了，所以left、top都是0，right和bottom是屏幕的宽和高；

对于非DecorView来讲，就看自身所在的父视图是什么布局形式了，如果是LinearLayout，就按照线性排列，比如竖直方向，那么top就要加上上个子视图的height，然后再传给子视图的layout()方法设置位置；

另外ViewGroup是先确定自身的位置，再确定子视图的位置，这个和measure的过程正好相反；

对于View来讲，位置由父视图计算好并传入，直接通过setFrame()设置位置即可。

## 问题

layout的过程，是先确定ViewGroup的位置呢？还是先确定View的位置呢？

答：先确定ViewGroup的位置，再确定View的位置。

这个看过源码就明白了，具体讲下ViewGroup的layout流程：

1. 调用setFrame()设置自己的位置
2. 遍历所有子View
3. 构造子View的位置参数，并调用子View的onLayout()方法

跟measure的顺序不一样。