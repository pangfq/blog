---
title: Activity绘制系列：3. LayoutInflater.inflate()
description: 
slug: android-view-source3
date: 2022-03-28T04:14:54-08:00
categories:
    - Android
tags:
    - View绘制
---

## 概述

我们知道，在Activity的setContentView()中，还有ListView的Adapter中，都是调用LayoutInflater.inflate()方法将xml布局渲染成View视图的，但具体是如何渲染的呢？我们来看下它的源码。

## 源码

```java
/** LayoutInflater.java **/
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
        return inflate(resource, root, root != null);
}

public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                  + Integer.toHexString(resource) + ")");
        }

        View view = tryInflatePrecompiled(resource, res, root, attachToRoot);
        if (view != null) {
            return view;
        }
  			// 得到一个xml资源解析器
        XmlResourceParser parser = res.getLayout(resource);
        try {
          	// 将解析器传入inflate()方法中
            return inflate(parser, root, attachToRoot);
        } finally {
          	// 这个解析器看来比较占资源
            parser.close();
        }
}
```



```java
/**XmlResourceParser.java**/
// xml资源解析器是一个接口，并且继承于XmlPullParser接口，也就是Android的Pull解析，跟SAX类似
public interface XmlResourceParser extends XmlPullParser, AttributeSet, AutoCloseable {
    String getAttributeNamespace (int index);

    /**
     * Close this parser. Calls on the interface are no longer valid after this call.
     */
    public void close();
}
```

```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            // ...

            try {
                // ...

              	// <merge/>标签的解析代码
                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }

                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml
                  	// 根据标签名称创建View，并赋值给命名为temp的临时变量
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                    // ...

                    // Inflate all children under temp against its context.
                  	// 开始解析子View
                    rInflateChildren(parser, temp, attrs, true);

                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }

                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                  	// 如果root参数不为空，并且attachToRoot为真，则将上面生成的View temp添加到root视图中
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                  	// 如果root为空，或attachToRoot为假，则直接将View temp当成结果返回
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }

            } catch (XmlPullParserException e) {
                // ...
            } catch (Exception e) {
                // ...
            } finally {
                // ...
            }
            return result;
        }
}
```

```java
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
            boolean ignoreThemeAttr) {
  			// ...
        try {
          	// 自定义流程
          	// 尝试创建View，这个方法内部会调用Factory、Factory2来尝试创建，如果用户实现了的话
          	// Factory是LayoutInflater对外提供的自定义解析xml的接口，比如想自定义一个标签名进行解析就可以实现Factory接口自己解析即可
            View view = tryCreateView(parent, name, context, attrs);

            if (view == null) {
                final Object lastContext = mConstructorArgs[0];
                mConstructorArgs[0] = context;
                try {
                    if (-1 == name.indexOf('.')) {
                      	// 正常流程
                      	// 走默认的xml解析来创建View
                        view = onCreateView(context, parent, name, attrs);
                    } else {
                        view = createView(context, name, null, attrs);
                    }
                } finally {
                    mConstructorArgs[0] = lastContext;
                }
            }

            return view;
        } catch (InflateException e) {
						// ...
        } catch (ClassNotFoundException e) {
            // ...
        } catch (Exception e) {
	          // ...
        }
}

public interface Factory {
        View onCreateView(String name, Context context, AttributeSet attrs);
}
```

```java
// 自定义解析流程
public final View tryCreateView(@Nullable View parent, @NonNull String name,
        @NonNull Context context,
        @NonNull AttributeSet attrs) {
        if (name.equals(TAG_1995)) {
            // Let's party like it's 1995!
            return new BlinkLayout(context, attrs);
        }

        View view;
        if (mFactory2 != null) {
          	// 优先使用Factory2创建
            view = mFactory2.onCreateView(parent, name, context, attrs);
        } else if (mFactory != null) {
          	// Factory2为空，则使用Factory创建
            view = mFactory.onCreateView(name, context, attrs);
        } else {
            view = null;
        }
  			// 问题1：为什么要对外提供Factory接口自定义解析xml呢？
  			// 答：在xml中除了系统默认支持的控件标签外，我们只能定义自定义的控件的标签，并且标签名必须是自定义控件的全限定类名，就是因为LayoutInflater自带的xml解析器是通过反射的方式直接创建控件的；但如果我们不想在xml中通过全限定类名来定义控件标签，就可以实现Factory接口自己解析创建控件
				
  			// Factory和Factory2都是用户自己创建的，如果都为空，则使用PrivateFactory，这是Framework创建的
        if (view == null && mPrivateFactory != null) {
            view = mPrivateFactory.onCreateView(parent, name, context, attrs);
        }
  
  			// 问题2：为什么系统Framework还要实现Factory接口呢?
  			// 答：因为Android系统后面新增了<fragment/>标签，这个没法通过反射类名创建对象，所以也得实现Factory接口
  			// 这个有点意思，感觉发现了新大陆
        return view;
}
```

```java
// 正常解析流程
private static final HashMap<String, Constructor<? extends View>> sConstructorMap =
            new HashMap<String, Constructor<? extends View>>();
public final View createView(@NonNull Context viewContext, @NonNull String name,
            @Nullable String prefix, @Nullable AttributeSet attrs)
            throws ClassNotFoundException, InflateException {
        Objects.requireNonNull(viewContext);
        Objects.requireNonNull(name);
  			// 通过标签名从一个静态的HashMap中取出Constructor对象
  			// Constructor对象为反射后得到的构造器对象
        Constructor<? extends View> constructor = sConstructorMap.get(name);
        if (constructor != null && !verifyClassLoader(constructor)) {
            constructor = null;
            sConstructorMap.remove(name);
        }
        Class<? extends View> clazz = null;

        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);

          	// 如果之前没创建过该标签的Constructor
            if (constructor == null) {
                // Class not found in the cache, see if it's real, and try to add it
              	// 通过标签名直接反射得到Class对象
                clazz = Class.forName(prefix != null ? (prefix + name) : name, false,
                        mContext.getClassLoader()).asSubclass(View.class);

                if (mFilter != null && clazz != null) {
                    boolean allowed = mFilter.onLoadClass(clazz);
                    if (!allowed) {
                        failNotAllowed(name, prefix, viewContext, attrs);
                    }
                }
              	// 通过Class对象得到其Constructor对象
                constructor = clazz.getConstructor(mConstructorSignature);
                constructor.setAccessible(true);
              	// 并存储到静态HashMap中缓存起来
                sConstructorMap.put(name, constructor);
            } else {
                // If we have a filter, apply it to cached constructor
                if (mFilter != null) {
                    // Have we seen this name before?
                    Boolean allowedState = mFilterMap.get(name);
                    if (allowedState == null) {
                        // New class -- remember whether it is allowed
                        clazz = Class.forName(prefix != null ? (prefix + name) : name, false,
                                mContext.getClassLoader()).asSubclass(View.class);

                        boolean allowed = clazz != null && mFilter.onLoadClass(clazz);
                        mFilterMap.put(name, allowed);
                        if (!allowed) {
                            failNotAllowed(name, prefix, viewContext, attrs);
                        }
                    } else if (allowedState.equals(Boolean.FALSE)) {
                        failNotAllowed(name, prefix, viewContext, attrs);
                    }
                }
            }

            Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = viewContext;
            Object[] args = mConstructorArgs;
            args[1] = attrs;

            try {
              	// 通过Constructor的newInstance()方法，也就是反射的方式就行创建View对象，因为任何控件都继承于View
                final View view = constructor.newInstance(args);
                if (view instanceof ViewStub) {
                    // Use the same context when inflating ViewStub later.
                    final ViewStub viewStub = (ViewStub) view;
                    viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
                }
                return view;
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        } catch (NoSuchMethodException e) {
            final InflateException ie = new InflateException(
                    getParserStateDescription(viewContext, attrs)
                    + ": Error inflating class " + (prefix != null ? (prefix + name) : name), e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;

        } catch (ClassCastException e) {
            // If loaded class is not a View subclass
            final InflateException ie = new InflateException(
                    getParserStateDescription(viewContext, attrs)
                    + ": Class is not a View " + (prefix != null ? (prefix + name) : name), e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;
        } catch (ClassNotFoundException e) {
            // If loadClass fails, we should propagate the exception.
            throw e;
        } catch (Exception e) {
            final InflateException ie = new InflateException(
                    getParserStateDescription(viewContext, attrs) + ": Error inflating class "
                            + (clazz == null ? "<unknown>" : clazz.getName()), e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
}
```

## 总结

综上所述，LayoutInflater.inflate()内部是通过Pull的方式解析xml，然后通过标签名进行反射创建出View对象，并将其的Class对象的Constructor对象保存到静态HashMap中缓存起来，这样下次解析同名的控件时就可以直接取出其的Constructor对象创建View了。另外，如果不想通过标签名反射创建View的，还可以实现LayoutInflater.Factory接口，并通过setFactory()传入实现类对象，从而实现自定义解析。

## 关键词

- Pull解析xml
- 反射标签名创建View
- 静态HashMap缓存Constructor
- LayoutInflater.Factory接口实现自定义解析