---
title: Activity绘制系列：7. onDraw()
description: 
slug: android-view-source7
date: 2022-03-28T04:14:54-08:00
categories:
    - Android
tags:
    - View绘制
---

## 概述

draw过程是确定视图外观，会传入canvas参数进行绘制。

我们还是从RootViewImpl开始看起。

## 源码

```java
public class ViewRootImpl {
  	// 直接new出来的Surface
  	public final Surface mSurface = new Surface();
  
		private void performDraw() {
  			// ...
	      boolean canUseAsync = draw(fullRedrawNeeded);
      	// ...
    }

  	private boolean draw(boolean fullRedrawNeeded) {
        Surface surface = mSurface;
      
      	// ...
      	if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                        scalingRequired, dirty, surfaceInsets)) {
                    return false;
        }
      	// ...
    }
  
  	private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
            boolean scalingRequired, Rect dirty, Rect surfaceInsets) {
    		final Canvas canvas;
      	// ...
      	// 通过Surface获取的canvas
	      canvas = mSurface.lockCanvas(dirty);
      	// 对canvas做一些初始化操作
      	// ...
      	// 将canvas传入DecorView的draw进行绘制
      	mView.draw(canvas);
      	// ...
  	}
}
```

因为ViewGroup没有覆写draw()，所以我们看下View的draw()：

```java
public class View {
  
	public void draw(Canvas canvas) {
				// ...
				// 有趣的是，draw的流程在源码中有清晰的注释，我们直接翻译下即可
        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         				// 绘制背景
         *      1. Draw the background
         				// 保存canvas图层为后续淡出做准备（可选）
         *      2. If necessary, save the canvas' layers to prepare for fading
         				// 绘制内容
         *      3. Draw view's content
         				// 绘制子View
         *      4. Draw children
         				// 绘制淡出边缘并恢复canvas（可选）
         *      5. If necessary, draw the fading edges and restore layers
         				// 绘制装饰（比如滚动条）
         *      6. Draw decorations (scrollbars for instance)
         */

        // Step 1, draw the background, if needed
        int saveCount;

        drawBackground(canvas);

        // skip step 2 & 5 if possible (common case)
        final int viewFlags = mViewFlags;
        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
        if (!verticalEdges && !horizontalEdges) {
            // Step 3, draw the content
            onDraw(canvas);

            // Step 4, draw the children
            dispatchDraw(canvas);

            drawAutofilledHighlight(canvas);

            // Overlay is part of the content and draws beneath Foreground
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }

            // Step 6, draw decorations (foreground, scrollbars)
            onDrawForeground(canvas);

            // Step 7, draw the default focus highlight
            drawDefaultFocusHighlight(canvas);

            if (debugDraw()) {
                debugDrawFocus(canvas);
            }

            // we're done...
            return;
        }
    		// ...
     }
}
```

## 总结

draw的流程其实挺简单的，传入的canvas参数，最初是从ViewRootImpl中创建的，一直从DecorView传递到子View，都用同一个Canvas对象；

ViewGroup比View多了个dispatchDraw()方法，就是绘制子View，其余都一样。

1. 绘制背景
2. 绘制内容
3. 绘制子View
4. 绘制装饰

## 问题

先绘制ViewGroup还是先绘制View？

答：这个问题就不能简单的像measure和layou那样回答是ViewGroup或者View了，因为看绘制的流程源码，ViewGroup要先绘制背景，绘制内容，然后绘制子View，最后绘制装饰，所以是按照图层来的，比如ListView，先绘制列表的背景色，再绘制子View，就是一个个item，最后绘制滚动条，而滚动条也属于ViewGroup，所以要看具体的ViewGroup是什么。

如果只是简单的LinearLayout，那确实是先绘制ViewGroup，后绘制View，而ScrollView就不是这样了，因为最后还要绘制LinearLayout。