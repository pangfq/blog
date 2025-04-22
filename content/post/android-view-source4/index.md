---
title: Activity绘制系列：4. startActivity()
description: 
slug: android-view-source4
date: 2022-03-28T04:14:54-08:00
categories:
    - Android
tags:
    - View绘制
---

## 概述

在此之前，我们看了setContentView()的源码，看了LayoutInflater.inflate()的源码，都没发现Activity绘制的触发代码，那触发Activity绘制的代码到底在哪里呢？在Activity里根本没找到，我想静静啊。

我闭上了眼睛，放空大脑，就这样放着放着，突然灵光一现：既然不在Activity里，那是不是在Activity外呢？

因为Activity的生命周期都是由Framework层去调用的，其中生命周期函数onResume()表示Activity的可见性，Activity绘制出来才能可见，所以如果找到了Activity的onResume()函数的调用处，那是不是就找到了触发Activity绘制的代码了呢？

确定了思路，马上开始看源码，那从哪里看起呢？当然是启动Activity的startActivity()方法了。

## 源码

```java
public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
          	// 调用startActivityForResult()
            startActivityForResult(intent, -1, options);
        } else {
            startActivityForResult(intent, -1);
        }
}
```

```java
public void startActivityForResult(Intent intent, int requestCode, Bundle options) {
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
          	// 调用Instrumentation的execStartActivity()
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
          	// ...
        }
}
```

```java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        // ...
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
          	// 调用ActivityTaskManager.getService().startActivity()
            int result = ActivityTaskManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
}
```

```java
// ActivityTaskManager.getService()会得到一个IActivityTaskManager实例
public static IActivityTaskManager getService() {
  			// 这个实例是放在一个单例中的
        return IActivityTaskManagerSingleton.get();
}
// 就是这个Singleton
private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
            new Singleton<IActivityTaskManager>() {
                @Override
                protected IActivityTaskManager create() {
                  	// 此处创建IActivityTaskManager对象，并返回
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                  	// 原来IActivityTaskManager是一个aidl，也就是Binder，所以startActivity()还是一个跨进程通信的过程
                    return IActivityTaskManager.Stub.asInterface(b);
                }
};
```

```java
// ActivityTaskManagerService类实现了IActivityTaskManager.Stub接口，所以跨进程到ATMS中进行处理
// ATMS所在进程叫SystemServer进程，这部分内容后续专门写文章讲，此处不再赘述
public class ActivityTaskManagerService extends IActivityTaskManager.Stub {
	@Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
      	// 调用了startActivityAsUser()
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }
  
  int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
            boolean validateIncomingUser) {
        enforceNotIsolatedCaller("startActivityAsUser");

        userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

				// 又调用了ActivityStarter.execute()
        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setMayWait(userId)
                .execute();

    }
}
```

```java
class ActivityStarter {
	int execute() {
        try {
            if (mRequest.mayWait) {
                return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                        mRequest.callingPackage, mRequest.realCallingPid, mRequest.realCallingUid,
                        mRequest.intent, mRequest.resolvedType,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                        mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                        mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                        mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup,
                        mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
            } else {
              	// 调用startActivity()
                return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                        mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                        mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                        mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                        mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                        mRequest.outActivity, mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup,
                        mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
            }
        } finally {
            onExecutionComplete();
        }
    }
  
  // 参数真的多啊
  private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                ActivityRecord[] outActivity, boolean restrictedBgActivity) {
        int result = START_CANCELED;
        final ActivityStack startedActivityStack;
        try {
            mService.mWindowManager.deferSurfaceLayout();
          	// 调用startActivityUnchecked()
            result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, outActivity, restrictedBgActivity);
        }
    		// ...
  }
  
  private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity, boolean restrictedBgActivity) {
    	// ...
	    if (mDoResume) {
        // 调用RootActivityContainer的resumeFocusedStacksTopActivities()
      	mRootActivityContainer.resumeFocusedStacksTopActivities();
      }
    	// ...
  }
}
```

```java
class RootActivityContainer extends ConfigurationContainer
        implements DisplayManager.DisplayListener {
  
	boolean resumeFocusedStacksTopActivities(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {

        if (!mStackSupervisor.readyToResume()) {
            return false;
        }

        boolean result = false;
        if (targetStack != null && (targetStack.isTopStackOnDisplay()
                || getTopDisplayFocusedStack() == targetStack)) {
          	// 调用ActivityStack的resumeTopActivityUncheckedLocked()
            result = targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }
  }
}
```

```java
class ActivityStack extends ConfigurationContainer {
	private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) 	{
    		// ...
        mStackSupervisor.startSpecificActivityLocked(next, true, false);
    		// ...
	}
}
```

```java
public class ActivityStackSupervisor implements RecentTasks.Callbacks {
	void startSpecificActivityLocked(ActivityRecord r, boolean andResume, boolean checkConfig)	{				
        realStartActivityLocked(r, wpc, andResume, checkConfig);
	}
  
  boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {
    		// 封装成ClientTransaction
	      mService.getLifecycleManager().scheduleTransaction(clientTransaction);
  }

}
```

```java
class ClientLifecycleManager {
		void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
      	// IApplicationThread是客户端的Binder，ATMS就是通过该Binder最终回到APP进程的
        final IApplicationThread client = transaction.getClient();
        transaction.schedule();
        if (!(client instanceof Binder)) {
          	// the transaction is executed on client in ActivityThread.
          	// 这个Transaction是被客户端的ActivityThread执行的
            transaction.recycle();
        }
    }
}
```

```java
public final class ActivityThread extends ClientTransactionHandler {
		
  // ApplicationThread继承了IApplicationThread.Stub，所以是一个主进程的Binder
  // 是ActivityThread的内部类
  private class ApplicationThread extends IApplicationThread.Stub {
        // ...
    		// ATMS会通过binder回调该方法
        public void scheduleApplicationInfoChanged(ApplicationInfo ai) {
            mH.removeMessages(H.APPLICATION_INFO_CHANGED, ai);
          	// 通过主线程的Handler回到主线程
            sendMessage(H.APPLICATION_INFO_CHANGED, ai);
        }
    		// ...
	}
  
  private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        // ...
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
          	// 调用Instrumentation去创建Activity
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }
    
    	// ...
    	if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
        						// 创建完Activity后，还是通过Instrumentation来调用Activity的生命周期
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
  }
  
  @Override
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
     		// ...
        // 调用Activity的onResume()生命周期方法
        final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
        
				// ...
        final Activity a = r.activity;

       	// ...
        if (r.window == null && !a.mFinished && willBeVisible) {
          	// 取出Activity的Window
            r.window = r.activity.getWindow();
          	// 取出Activity的DecorView
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
          	// 取出Activity的WindowManager
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            // ...
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
										
                  	// 调用WindowManager的addView()，将DecorView添加到Window中
                  	// 内部会调用WindowManagerGlobal的addView()
                  	// 再继续调用ViewRootImpl的setView()触发DecorView的绘制
                  	// 所以此处就是就是Activity绘制的开始
                    wm.addView(decor, l);
                } else {
                    a.onWindowAttributesChanged(l);
                }
            }

        } else if (!willBeVisible) {
            if (localLOGV) Slog.v(TAG, "Launch " + r + " mStartedActivity set");
            r.hideForNow = true;
        }

        if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
            // ... 
            WindowManager.LayoutParams l = r.window.getAttributes();
            if ((l.softInputMode
                    & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                    != forwardBit) {
                l.softInputMode = (l.softInputMode
                        & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                        | forwardBit;
                if (r.activity.mVisibleFromClient) {
                    ViewManager wm = a.getWindowManager();
                    View decor = r.window.getDecorView();
                  	// 此处也会触发DecorView的绘制
                  	// WindowManager实现了ViewManager接口，该接口中有三个方法:addView()、updateViewLayout()、removeView()，分别对应View的添加、更新、删除操作，所以添加和更新一定会触发View的绘制
                    wm.updateViewLayout(decor, l);
                }
            }
          	// ...
        }

        Looper.myQueue().addIdleHandler(new Idler());
    }
}
```

```java
// Activity的生命周期都是通过Instrumentation调用的
public class Instrumentation {
  // 创建Activity
	public Activity newActivity(ClassLoader cl, String className,
            Intent intent)
            throws InstantiationException, IllegalAccessException,
            ClassNotFoundException {
        String pkg = intent != null && intent.getComponent() != null
                ? intent.getComponent().getPackageName() : null;
              // 调用了一个工厂去创建Activity
        return getFactory(pkg).instantiateActivity(cl, className, intent);
	}
  
  // 调用Activity的生命周期方法onResume()
  public void callActivityOnResume(Activity activity) {
        activity.mResumed = true;
        activity.onResume();
        
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    am.match(activity, activity, activity.getIntent());
                }
            }
        }
    }
  public void callActivityOnCreate(Activity activity){//...}
  public void callActivityOnDestroy(Activity activity){//...}
  public void callActivityOnStart(Activity activity){//...}
  // ...
}
```

```java
public class AppComponentFactory {
	public @NonNull Activity instantiateActivity(@NonNull ClassLoader cl, @NonNull String 	className,
            @Nullable Intent intent)
            throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    		// 原来最后是通过一个ClassLoader创建出的Activity，这个ClassLoader是从Context获取的
        return (Activity) cl.loadClass(className).newInstance();
	}
}
```

```java
public final class WindowManagerGlobal {
  	// 是一个单例，一个进程就只有一个
		public static WindowManagerGlobal getInstance() {
        synchronized (WindowManagerGlobal.class) {
            if (sDefaultWindowManager == null) {
                sDefaultWindowManager = new WindowManagerGlobal();
            }
            return sDefaultWindowManager;
        }
   	}
  
  	public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        // ...

        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            // ...
          	// 创建ViewRootImpl对象，该对象会持有DecorView，跟它的名字一样，ViewRoot，视图根
          	// WindowManager通过ViewRootImpl来管理View，就跟访问链表要先通过head指针一样
            root = new ViewRootImpl(view.getContext(), display);

            view.setLayoutParams(wparams);

            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);

            try {
              	// 进入到ViewRootImpl中的setView()
                root.setView(view, wparams, panelParentView);
            } catch (RuntimeException e) {
                // BadTokenException or InvalidDisplayException, clean up.
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
                throw e;
            }
        }
    }
}
```

```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
       
          public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
          			// ...
            		requestLayout();
            		// ...
       		}
          
          public void requestLayout() {
            	// ...
	            scheduleTraversals();
            	// ...
        	}
    			
          void scheduleTraversals() {
        			if (!mTraversalScheduled) {
            			mTraversalScheduled = true;
                	// 因为涉及到View，所以要通过Handler提交到主线程处理
                	// 这里就有意思了，发现先提了一个同步屏障消息
                	// 同步屏障消息的作用是屏蔽掉当前MessageQueue中的同步消息，优先让异步消息执行
                	// 而绘制View的操作会放到Runnable中，并将Message设置成异步消息，这样绘制View就会优先得到处理
            			mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            			mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        			}
    			}
          
          final class TraversalRunnable implements Runnable {
     			   @Override
			        public void run() {
      			      doTraversal();
        			}	
    			}
          
          void doTraversal() {
            	if (mTraversalScheduled) {
          	    mTraversalScheduled = false;
                // 开始执行了，就把同步屏障移除吧
          	    mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
          	    // 执行遍历，为啥要叫遍历呢？是因为DecorView本身就是一个View树，所以它的绘制就是一个深度优先遍历的过程，所以就叫Tralversals
                performTraversals();
          	}
    			}
          
          private void performTraversals() {
            	// ...
          		performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);  
            	// ...
            	performLayout(lp, mWidth, mHeight);
            	// ...
            	performDraw();
            	// ...
          }
          
          private void performMeasure(int childWidthMeasureSpec,int childHeightMeasureSpec){
          		if (mView == null) {
              		return;
          		}
          		Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
          		try {
                	// 调用DecorView的measure()开始进入测量环节
              		mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
          		} finally {
              		Trace.traceEnd(Trace.TRACE_TAG_VIEW);
          		}
    			}
          
          // performLayout()和performDraw()就不贴出来了
        
}
```



源码跟踪到此结束。

## 总结

我们最终通过Activity的startActivity()找到了Activity绘制的触发点，还顺便了解了Activity的启动流程，这一趟没白来，下面简要总结下整个流程：

1. Activity的startActivity()开始
2. Instrumentation调用Binder对象IActivityTaskManager跨进程进入到ActivityTaskManagerService(ATMS)中
3. 经过ATMS的一系列处理，包括Activity是否在Manifest.xml中注册过等等
4. ATMS处理完，会调用Binder对象IApplicationThread跨进程回到APP进程ActivityThread中
5. ActivityThread会调用主线程的Handler回调到主线程
6. ActivityThread会调用performCreateActivity()，由Instrumentation通过ClassLoader创建出Activity对象，并调用Activity的onCreate()
7. ActivityThread会调用handleResumeActivity()，同样通过Instrumentation调用Activity的onResume()；并通过Activity的WindowManager调用addView(mDecorView)将Activity的DecorView添加进来
8. WindowManager内部通过桥梁模式继续调用WindowManagerGlobal调用addView()
9. WindowManagerGlobal的addView()内部会创建ViewRootImpl并持有DecorView，然后调用ViewRootImpl的setView(mDecorView)
10. ViewRootImpl的setView()内部会先通过Handler发送一个同步屏障消息，然后将触发DecorView绘制的操作放到一个异步消息中，来保证View绘制优先处理。

## 关键词

- Instrumentation
- IActivityTaskManager
- ActivityTaskManagerService
- IApplicationThread
- ActivityThread
- WindowManager extends ViewManager
- WindowManagerGlobal
- ViewRootImpl