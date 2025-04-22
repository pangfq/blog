---
title: Activity绘制系列：5. onMeasure()
description: 
slug: android-view-source5
date: 2022-03-28T04:14:54-08:00
categories:
    - Android
tags:
    - View绘制
---

## 概述

Activity绘制到屏幕上要经历3个步骤，measure、layout、draw，分别是测量、布局、绘制，作用分别是确定宽高、确定位置、确定外观。

下面先从measure过程讲起，还是看源码，但是要从哪里看起呢？

Activity的绘制是从DecorView开始的，当然是从调用DecorView的measure()地方开始看起，也就是ViewRootImpl类。

## 源码

```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
          
       	public final WindowManager.LayoutParams mWindowAttributes = new WindowManager.LayoutParams();

        private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        	if (mView == null) {
        	    return;
        	}
        	Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        	try {
            	// 此处开始正式调用DecorView的measure()方法，可以看到需要传入两个参数，分别是宽的测量规格和高的测量规格，我们看下这两个参数是从哪里构造出来的
        	    mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        	} finally {
        	    Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        	}
    		}
        
          
        private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
            final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
        int childWidthMeasureSpec;
        int childHeightMeasureSpec;
        
        if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT) {
            // On large screens, we don't want to allow dialogs to just
            // stretch to fill the entire width of the screen to display
            // one line of text.  First try doing the layout at a smaller
            // size to see if it will fit.
            final DisplayMetrics packageMetrics = res.getDisplayMetrics();
            res.getValue(com.android.internal.R.dimen.config_prefDialogWidth, mTmpValue, true);
            int baseSize = 0;
            if (mTmpValue.type == TypedValue.TYPE_DIMENSION) {
                baseSize = (int)mTmpValue.getDimension(packageMetrics);
            }
            
            if (baseSize != 0 && desiredWindowWidth > baseSize) {
              	// 通过size和lp(LayoutParam)来构造DecorView的MeasureSpec
              	// baseSize就是屏幕的宽
              	// lp来自于成员变量mWindowAttributes，是一个WindowManager.LayoutParams，宽高都是MATCH_PARENT
                childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
                childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);

              	performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            }
        }

      	// ...
        return windowSizeMayChange;
    }
          
    private void performTraversals() {
     		// ...       
        WindowManager.LayoutParams lp = mWindowAttributes;
      	// ...
    }
          
		private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {

        case ViewGroup.LayoutParams.MATCH_PARENT:
            // Window can't resize. Force root view to be windowSize.
            // 通过size和mode来创建MeasureSpec
            // 对于DecorView来讲，size就是屏幕尺寸，lp是MATCH_PARENT，所以mode就是EXACTLY
            // 换句话说，DecorView的MeasureSpec是由屏幕来构造出来的
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            // Window can resize. Set max size for root view.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            // Window wants to be an exact size. Force root view to be that size.
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
    }

}
```

接下来我们看看ViewGroup的measure过程。

ViewGroup是一个抽象类，没有定义测量的具体过程，具体实现由子类完成，下面以LinearLayout为例说明下：

```java
public class LinearLayout extends ViewGroup {

		@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        if (mOrientation == VERTICAL) {
            measureVertical(widthMeasureSpec, heightMeasureSpec);
        } else {
            measureHorizontal(widthMeasureSpec, heightMeasureSpec);
        }
    }
  
  void measureHorizontal(int widthMeasureSpec, int heightMeasureSpec) {
        // ...
				// 获取父视图传来的测量模式
        final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
    		// ...
    		// 遍历所有子视图
		    for (int i = 0; i < count; ++i) {
          	// 测量子视图
    				measureChildBeforeLayout(child, i, widthMeasureSpec, usedWidth,
                        heightMeasureSpec, 0);
						// 获取子视图的宽
		        final int childWidth = child.getMeasuredWidth();   
          	// ...
        }
    
    		// 测量完所有子视图之后，根据子视图的宽高，最后设置自身的宽高
    		setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                heightSizeAndState);

  }
  
  void measureChildBeforeLayout(View child, int childIndex,
            int widthMeasureSpec, int totalWidth, int heightMeasureSpec,
            int totalHeight) {
    		// 调用的是ViewGroup中的方法
        measureChildWithMargins(child, widthMeasureSpec, totalWidth,
                heightMeasureSpec, totalHeight);
  }
}
```

```java
public abstract class ViewGroup {
		
  	protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
      	// 获取子视图的LayoutParams
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

      	// 构造子视图的MeasureSpec
      	// size：要减去父视图的padding和子视图的margin
      	// mode：父视图的MeasureSpec和子视图的LayoutParams
      	// 所以，可以看出传入子视图measure()的测量规格（size和mode），是由父视图和子视图共同决定的
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

      	// 测量子视图
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
		}
}
```

最后看下View的measure过程：

```java
public class View {
  
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    		// 对于View来讲，测量规格已经由父视图根据父视图的MeasureSpec和View自己的LayoutParams计算过了，所以如果没有特殊需求，可以直接设置自身的宽高即可
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
	}
}
```

## 总结

对于measure的过程，可以分为两部分看：

- ViewGroup
- View

ViewGroup又可以分为顶级ViewGroup（DecorView）和非顶级ViewGroup，区别只是在构造MeasureSpec上有差异，虽然两者的MeasureSpec都是来自父视图，但DecorView因为已经是顶级视图，它没有父视图了，它的MeasureSpec是来自于屏幕，或者说Window，size和mode都是固定的，size是屏幕宽高，mode是exactly，其实也可以把DecorView看成是特殊的ViewGroup；

对于非顶级的ViewGroup来讲，MeasureSpec同样来自父视图，size由父视图的size减去padding，再减去自身的margin；mode是由父视图的MeasureSpec和自身的LayoutParams共同决定；

measure流程是先遍历其所有子视图，依次进行measure，并得到measure之后的宽高，最后据此来设置自身的宽高。

对于View来讲，MeasureSpec参数的构造和非ViewGroup一样，不再赘述；

而measure流程是直接设置通过MeasureSpec参数设置宽高，因为MeasureSpec已经由父视图计算好了，如果没有其他特殊需求，直接设置即可。

## 问题

一个Activity中有一个LinearLayout，LinearLayout中又有一个Button，请问measure的过程，LinearLayout和Button哪个先measure完成？

答：Button，说下ViewGroup的measure源码流程即可：

1. 遍历所有子视图
2. 依次measure子视图，并获取其measure后的宽高
3. 根据measure后的宽高来确定自身的宽高

可以看到ViewGroup中调用了子视图的measure()，所以，Button先measure完成。

