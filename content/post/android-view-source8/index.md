---
title: Activity绘制系列：8. requestLayout()
description: 
slug: android-view-source8
date: 2022-03-28T04:14:54-08:00
categories:
    - Android
tags:
    - View绘制
---

## 概述

requestLayout()，字面意思是请求布局，那是不是只调用了layout过程呢？具体看下源码便知。

## 源码

```java
public class View implements Drawable.Callback, KeyEvent.Callback,
        AccessibilityEventSource {
          
	public void requestLayout() {
        if (mMeasureCache != null) mMeasureCache.clear();

        if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
            // Only trigger request-during-layout logic if this is the view requesting it,
            // not the views in its parent hierarchy
          	// 这里会获取ViewRootImpl对象
            ViewRootImpl viewRoot = getViewRootImpl();
          	// 判断是否正在执行ViewRootImpl的layout操作，如果正在执行，则将请求requestLayout的View放到一个List中，等View树layout结束之后，才会去调用刚才requestLayout的View
          	// 为什么要有这个判断呢？因为在layout()流程中又调用了requestLayout()，如果不加判断，就又会重新走layout()，导致死循环。
            if (viewRoot != null && viewRoot.isInLayout()) {
                if (!viewRoot.requestLayoutDuringLayout(this)) {
                    return;
                }
            }
            mAttachInfo.mViewRequestingLayout = this;
        }

        mPrivateFlags |= PFLAG_FORCE_LAYOUT;
        mPrivateFlags |= PFLAG_INVALIDATED;

        if (mParent != null && !mParent.isLayoutRequested()) {
          	// mParent是一个ViewParent类型的变量，ViewGroup和ViewRootImpl都实现了该接口，所以View中的mParent就是自己的ViewGroup
          	// 子View会调用父View的requestLayout()，如此这般调用，层层向上传递，最终会到ViewRootImpl的requestLayout()中
            mParent.requestLayout();
        }
        if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
            mAttachInfo.mViewRequestingLayout = null;
        }
    }
}
```

```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
          
        ArrayList<View> mLayoutRequesters = new ArrayList<View>();
          
        // 在layout过程中请求layout
				boolean requestLayoutDuringLayout(final View view) {
        	// ...
  				// 放到一个ArrayList中
        	if (!mLayoutRequesters.contains(view)) {
          	  mLayoutRequesters.add(view);
        	}
        	// ...
    		}
          
        @Override
    		public void requestLayout() {
        	if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            // 这个方法很熟悉吧，内部通过handler，先post一个同步屏障消息，然后再post一个异步消息，触发Activity绘制的三大流程，所以从该方法就可以看出来，调用View的requestLayout()，其实并不是只会触发layout操作，而是触发三大流程
            scheduleTraversals();
        	}	
    		}
          
          private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
            int desiredWindowHeight) {
        		// ...
            // 调用layout()之前，会将标志位置为true
            mInLayout = true;
        try {
          	// 开始调用DecorView的layout()，进行View树的layout过程
            host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

          	// 调用layout()之后，会将标志位置为false
            mInLayout = false;
          
          	// 到这一步，View树的layout阶段已经执行完成
          	// 接下来，判断之前在layout()过程中是否有子View调用了requestLayout()重新请求layout
          	// 如果有的话，再依次取出执行requestLayout()方法
          	// 此处就解释了在View树的layout阶段如果有子View调用了requestLayout()为什么不会出现死循环的原因
            int numViewsRequestingLayout = mLayoutRequesters.size();
            if (numViewsRequestingLayout > 0) {
                // requestLayout() was called during layout.
                // If no layout-request flags are set on the requesting views, there is no problem.
                // If some requests are still pending, then we need to clear those flags and do
                // a full request/measure/layout pass to handle this situation.
                ArrayList<View> validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters,
                        false);
                if (validLayoutRequesters != null) {
                    // Set this flag to indicate that any further requests are happening during
                    // the second pass, which may result in posting those requests to the next
                    // frame instead
                    mHandlingLayoutInLayoutRequest = true;

                    // Process fresh layout requests, then measure and layout
                    int numValidRequests = validLayoutRequesters.size();
                    for (int i = 0; i < numValidRequests; ++i) {
                        final View view = validLayoutRequesters.get(i);
                        Log.w("View", "requestLayout() improperly called by " + view +
                                " during layout: running second layout pass");
                        view.requestLayout();
                    }
                    measureHierarchy(host, lp, mView.getContext().getResources(),
                            desiredWindowWidth, desiredWindowHeight);
                    mInLayout = true;
                    host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

                    mHandlingLayoutInLayoutRequest = false;

                    // Check the valid requests again, this time without checking/clearing the
                    // layout flags, since requests happening during the second pass get noop'd
                    validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters, true);
                    if (validLayoutRequesters != null) {
                        final ArrayList<View> finalRequesters = validLayoutRequesters;
                        // Post second-pass requests to the next frame
                        getRunQueue().post(new Runnable() {
                            @Override
                            public void run() {
                                int numValidRequests = finalRequesters.size();
                                for (int i = 0; i < numValidRequests; ++i) {
                                    final View view = finalRequesters.get(i);
                                    Log.w("View", "requestLayout() improperly called by " + view +
                                            " during second layout pass: posting in next frame");
                                    view.requestLayout();
                                }
                            }
                        });
                    }
                }

            }
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        mInLayout = false;
    }
}
```

## 总结

requestLayout()需要掌握两点：

1. 会触发整个View树的measure、layout、draw过程，而并非只有layout
2. 如果在View树的layout过程中有子View在其layout()中调用了requestLayout()会不会造成死循环？不会，因为ViewRootImpl会等到View树的layout()执行后才会去调用之前requestLayout()的子View