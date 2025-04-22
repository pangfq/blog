---
title: Activity绘制系列：1. setContentView()
description: 
slug: android-view-source1
date: 2022-03-28T04:14:54-08:00
categories:
    - Android
tags:
    - View绘制
---

## 概述

在APP的日常开发中，无论Android还是iOS，UI相关的需求都是非常多的，需求有大有小，比如小到改变字体颜色，大到自定义复杂的View，这些都离不开View的绘制，所以掌握绘制原理，是每个移动开发工程师的必备技能。

## 思考

在阅读文章之前，先思考一个问题，带着问题去学习，效果会更好。

我们在开发UI需求时，会在xml中写布局代码，这个xml最终会被Activity解析、加载、绘制出来，但具体是如何解析？如何加载？又是如何绘制的呢？这些过程涉及到了哪些方法，又涉及到了哪些对象呢？

上述问题都将在这几篇系列文章中得到解答。

## 源码

想要回答上面的问题，看源码是最好的方式。

```java
/** Activity.java **/
public void setContentView(@LayoutRes int layoutResID) {
  			// 调用Window的setContentView()，而Window的唯一实现类是PhoneWindow
        getWindow().setContentView(layoutResID);
  			// 初始化ActionBar
        initWindowDecorActionBar();
}
```

```java
/** PhoneWindow.java **/
// This is the view in which the window contents are placed. It is either
// mDecor itself, or a child of mDecor where the contents go.
// 翻译：
// 		这是一个在Window中用来替换成内容的View。可以是Decor自己，也可以是Decor的子View。
// 理解：
// 		1. 我们在xml中写的布局对于整个Activity来讲，叫content（这也是为什么Activity使用setContentView()来加载xml的原因），此处的mContentParent就是我们写的xml布局的直接父视图；
// 		2. 另外，mContentParent可能是DecorView(顶层视图)，也可能是DecorView的子视图，这个取决于给Window设置的feature属性，比如：android:theme="@android:style/Theme.NoTitleBar"，全屏窗口，此时mContentParent就是DecorView；
ViewGroup mContentParent;
public void setContentView(int layoutResID) {
  			// 如果mContentParent为空，则安装Decor。
  			// 1. mContentParent：content视图的直接父视图，可能是DecorView，也可能是DecorView的子视图
  			// 2. Decor：DecorView，顶层视图
        if (mContentParent == null) {
          	// 初始化DecorView和mContentParent
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
          	// 把content视图添加到mContentParent视图中
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
}

private void installDecor() {
        if (mDecor == null) {
          	// 生成DecorView
            mDecor = generateDecor(-1);
            // ...
        } else {
            mDecor.setWindow(this);
        }
        if (mContentParent == null) {
          	// 生成mContentParent，并传入上面刚生成的DecorView实例，可见mContentParent依赖于mDecor
            mContentParent = generateLayout(mDecor);
            // ...
}

protected DecorView generateDecor(int featureId) {
        // ...
  			// 直接new一个DecorView，其继承于FrameLayout，是Activity的顶层视图
        return new DecorView(context, featureId, this, getAttributes());
}

protected ViewGroup generateLayout(DecorView decor) {
        // ...
        int layoutResource;
        int features = getLocalFeatures();
        // ...
			  // 上面省略的代码是：通过判断feature类型来给layoutResource设置不同的系统布局，比如带ActionBar的，带TitleBar的，啥都不带的等
  			// ...
  			// 把layoutResource的布局添加到DecorView中
        mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
				// 调用DecorView的findViewById()，查找指定ID为com.android.internal.R.id.content的视图，这个视图就是mContentParent
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        // ...
        return contentParent;
}

public <T extends View> T findViewById(@IdRes int id) {
  			// 调用的是DecorView的findViewById()
        return getDecorView().findViewById(id);
}  
```

## 流程总结

1. Activity的setContentView()内部调用的是PhoneWindow的setContentView()
2. 先创建一个DecorView，其继承于FrameLayout，这是Activity的顶层视图
3. 再根据Window设置的feature属性给DecorView选择添加对应的子布局，有带ActionBar，有不带ActionBar的等等
4. 接着调用DecorView的findViewById(android.R.id.content)找到加载Content的父视图mContentParent
5. 最后调用LayoutInflate.inflate(content, mContentParent)将content视图添加到mContentParent上

## 布局结构

![](https://upload-images.jianshu.io/upload_images/2004563-9f5d4e9e11bce7c5.png)

上面是DecorView完整的组成，其中注意DecorView的两个子视图中有一个是状态栏，我们在源码追踪中没有说明，具体是在DecorView的updateColorViewInt()方法中创建并添加的。

另外一个子视图就是根据不同feature属性选择inflate的布局，在这些布局中会有一个固定id为android.R.id.content的FrameLayout，也就是mContentParent，用来添加我们自己写的xml布局。

## 总结

所以setContentView()只是解析xml创建View并设置到DecorView中，并没有进行绘制，所以绘制的代码我们还需要再找。

## 关键词

- DecorView
- 根据不同的feature属性inflate不同的布局
- android.R.id.content是一个FrameLayout











