---
title: 诡异的BadTokenException

date: 2017-10-12 21:34:33
categories: Android
---
### 写作目的

&emsp;&emsp;最近遇到一个线上问题，Activity在启动的时候按home键之后会报一个BadTokenException的崩溃异常，由于log不全，没有分析出原因。不过还是学到不少东西，准备记录一下，所以看完之后你并不能找到解决方案，但是应该能搞明白以下三个问题：

1. **为什么创建Dialog时的context参数只能是Ativity**
2. **为什么Dialog依赖的Activity finish之后，Dialog不能show**
3. **为什么有些情况下Activity启动时按home键之后应用会崩溃**

&emsp;&emsp;以下是基于7.0源码分析

<!--more-->

### 问题分析

#### BadTokenException异常在哪里抛的

&emsp;&emsp;一般这种异常的堆栈信息如下：

```
Caused by: android.view.WindowManager$BadTokenException: Unable to add window -- token null is not valid; is your activity running?
     at android.view.ViewRootImpl.setView(ViewRootImpl.java:699)
     at android.view.WindowManagerGlobal.addView(WindowManagerGlobal.java:319)
     at android.view.WindowManagerImpl.addView(WindowManagerImpl.java:85)
     at android.app.Dialog.show(Dialog.java:325)
                                                       at me.yokeyword.itemtouchhelperdemo.MainActivity.onCreate(MainActivity.java:30)
     at android.app.Activity.performCreate(Activity.java:6365)
     at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1124)
     at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2655)
     at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2773) 
     at android.app.ActivityThread.-wrap11(ActivityThread.java) 
     at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1587) 
     at android.os.Handler.dispatchMessage(Handler.java:111) 
     at android.os.Looper.loop(Looper.java:207) 
     at android.app.ActivityThread.main(ActivityThread.java:5940) 
     at java.lang.reflect.Method.invoke(Native Method) 
     at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:929) 
     at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:790) 
```

&emsp;&emsp;从崩溃栈中可以看到，异常在ViewRootImpl类的setView方法中，该方法如下（省略部分代码）

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        //省略部分代码
        try {
                    mOrigWindowType = mWindowAttributes.type;
                    mAttachInfo.mRecomputeGlobalAttributes = true;
                    collectViewAttributes();
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
                } catch (RemoteException e) {
                //省略部分代码
                } finally {
                //省略部分代码
                }
                //省略部分代码
                if (res < WindowManagerGlobal.ADD_OKAY) {
                    mAttachInfo.mRootView = null;
                    mAdded = false;
                    mFallbackEventHandler.setView(null);
                    unscheduleTraversals();
                    setAccessibilityFocus(null, null);
                    switch (res) {
                        case WindowManagerGlobal.ADD_BAD_APP_TOKEN:
                        case WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN:
                            throw new WindowManager.BadTokenException(
                                    "Unable to add window -- token " + attrs.token
                                    + " is not valid; is your activity running?");
                        case WindowManagerGlobal.ADD_NOT_APP_TOKEN:
                            throw new WindowManager.BadTokenException(
                                    "Unable to add window -- token " + attrs.token
                                    + " is not for an application");
                       //省略部分代码
                }
```

&emsp;&emsp;当res的值为WindowManagerGlobal.ADD\_BAD\_APP\_TOKEN、WindowManagerGlobal.ADD\_BAD\_SUBWINDOW\_TOKEN和WindowManagerGlobal.ADD\_NOT\_APP\_TOKEN等都会抛出这个异常。

####为什么会抛这个异常

&emsp;&emsp;上面可以看到，当res为为某些值得时候就会抛出异常。那这个res的值是什么呢？上面代码有如下一句：

```java
res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
```
&emsp;&emsp;这个值是mWindowSession的addToDisplay方法的返回值，mWindowSession是IWindowSession类型，它是WindowManagerGlobal类的getWindowSession返回的，该方法通过binder和WindowManagerService通信，得到一个Session类在客户端的binder对象，即mWindowSession，所以调用addToDisplay方法就会调用Session类中对应的方法，Session类的addToDisplay方法直接调用了WindowManagerService的addWidow方法，如下：

```java
public int addWindow(Session session, IWindow client, int seq,
            WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
            Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
            InputChannel outInputChannel) {
            
            //省略部分代码
            final int type = attrs.type;
            
            //省略部分代码
            WindowToken token = mTokenMap.get(attrs.token);
            AppWindowToken atoken = null;
            if (token == null) {
                if (type >= FIRST_APPLICATION_WINDOW && type <= LAST_APPLICATION_WINDOW) {
                    Slog.w(TAG_WM, "Attempted to add application window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                //省略部分代码
            }
            //省略部分代码
}
```
&emsp;&emsp;对于Activity和Dialog来说，Window的type都是TYPE_APPLICATION，因为WindowManager.LayoutParams的默认构造函数中被赋值为该值。此时当token为null时，就会返回WindowManagerGlobal.ADD\_BAD\_APP\_TOKEN，结合【2.1】中所述，此时就会导致抛出BadTokenException异常。

&emsp;&emsp;从这个函数的其他逻辑也可以看出，如果type为某些值得时候，返回值也可以是WindowManagerGlobal.ADD\_OKAY。不过遗憾的是WindowManager.LayoutParams的type属性并不能被随便赋值，如果不是系统所允许的值，AS中就会有红色波浪线提示。这个可能是通过@ViewDebug.ExportedProperty注解来控制的，没细研究，只看到源码中type使用了这个注解。

&emsp;&emsp;所以要解决这个问题，就要保证token不为null。这个值为null无外乎两种情况：1、没有向WMS添加过；2、向WMS添加过，但是之后又删除了。第一个情况就对应了【1】中第一个问题的答案，第二个情况对应了【1】中第二个和第三个问题的答案。

#### token是如何被创建和添加的

&emsp;&emsp;在看Activity、AMS和WMS时经常遇到这个token，对这个token不了解的话有时候就会看着有点晕。接下来就看看Activity相关的token是如何被创建并添加到WMS中的。

##### token是如何被创建的

&emsp;&emsp;Activity类中有个mToken变量，这个变量在attach方法中被赋值：

```java
final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window) {
            //省略部分代码
            mToken = token;
            //省略部分代码
            mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
}
```
&emsp;&emsp;这个方法是在哪里被调用的呢？这个就牵扯到Activity的启动过程了([传送门](http://blog.csdn.net/luoshengyang/article/details/6689748))，这里中简要说一下。Activity的attach方法是在ActivityThread中调用的，调用关系为scheduleLaunchActivity->handleLaunchActivity->performLaunchActivity，其中scheduleLaunchActivity是ActivityThread的内部类ApplicationThread中的方法。

&emsp;&emsp;scheduleLaunchActivity方法如下：

```java
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                int procState, Bundle state, PersistableBundle persistentState,
                List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

            updateProcessState(procState, false);

            ActivityClientRecord r = new ActivityClientRecord();

            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.referrer = referrer;
            r.voiceInteractor = voiceInteractor;
            r.activityInfo = info;
            r.compatInfo = compatInfo;
            r.state = state;
            r.persistentState = persistentState;

            r.pendingResults = pendingResults;
            r.pendingIntents = pendingNewIntents;

            r.startsNotResumed = notResumed;
            r.isForward = isForward;

            r.profilerInfo = profilerInfo;

            r.overrideConfig = overrideConfig;
            updatePendingConfiguration(curConfig);

            sendMessage(H.LAUNCH_ACTIVITY, r);
        }
```

&emsp;&emsp;该方法通过一个发送一个H.LAUNCH\_ACTIVITY消息来调用handleLaunchActivity，源码如下：

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
        //省略部分代码
        Activity a = performLaunchActivity(r, customIntent);
        //省略部分代码
```

&emsp;&emsp;handleLaunchActivity直接调用了performLaunchActivity方法，源码如下：

```java
priivate Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        //省略部分代码
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
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
        //省略部分代码
        if(activity != null) {
        //省略部分代码
            activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window);
        }
        //省略部分代码
```
&emsp;&emsp;可以看到，在performLaunchActivity中创建一个Activity并调用了它的attach方法。attach方法的token参数最初是来自scheduleLaunchActivity方法，那么scheduleLaunchActivity方法中的token参数来自哪里呢？它是ActivityStackSupervisor在realStartActivityLocked方法，代码如下：

```java
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException {
            //省略部分代码
app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    new Configuration(task.mOverrideConfig), r.compat, r.launchedFromPackage,
                    task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                    newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);
                    //省略部分代码

}
```

&emsp;&emsp;其中app.thread是一个binder对象，代表客户端的ApplicationThread对象。可以看到scheduleLaunchActivity的token参数是通过ActivityRecord的对象r获得的。这个ActivityRecord对象时在ActivityStarter的startActivityLocked方法中创建的，源码如下：

```java
final int startActivityLocked(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
            ActivityRecord[] outActivity, ActivityStackSupervisor.ActivityContainer container,
            TaskRecord inTask) {
            //省略部分代码
            ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage,
                intent, resolvedType, aInfo, mService.mConfiguration, resultRecord, resultWho,
                requestCode, componentSpecified, voiceSession != null, mSupervisor, container,
                options, sourceRecord);
                //省略部分代码
}
```
&emsp;&emsp;在ActivityRecord的构造方法中会new一个Token对象赋值给appToken。至此找到了token创建的地方。

##### token如是何被添加的

&emsp;&emsp;上面找到了token被创建的逻辑，它是在Activity启动的时候由server端创建的。其实在Activity启动的时候AMS还会白创建好的token添加到WMS中。源码在ActivityStack中，该类的startActivityLocked方法会调用addConfigOverride方法，在该方法中会调用添加token的方法，代码如下：

```java
void addConfigOverride(ActivityRecord r, TaskRecord task) {
        final Rect bounds = task.updateOverrideConfigurationFromLaunchBounds();
        // TODO: VI deal with activity
        mWindowManager.addAppToken(task.mActivities.indexOf(r), r.appToken,
                r.task.taskId, mStackId, r.info.screenOrientation, r.fullscreen,
                (r.info.flags & FLAG_SHOW_FOR_ALL_USERS) != 0, r.userId, r.info.configChanges,
                task.voiceSession != null, r.mLaunchTaskBehind, bounds, task.mOverrideConfig,
                task.mResizeMode, r.isAlwaysFocusable(), task.isHomeTask(),
                r.appInfo.targetSdkVersion);
        r.taskConfigOverride = task.mOverrideConfig;
    }
```
&emsp;&emsp;其中，mWindowManager是WMS对象，WMS类中的addAppToken方法代码如下：

```java
public void addAppToken(int addPos, IApplicationToken token, int taskId, int stackId,
            int requestedOrientation, boolean fullscreen, boolean showForAllUsers, int userId,
            int configChanges, boolean voiceInteraction, boolean launchTaskBehind,
            Rect taskBounds, Configuration config, int taskResizeMode, boolean alwaysFocusable,
            boolean homeTask, int targetSdkVersion) {
        if (!checkCallingPermission(android.Manifest.permission.MANAGE_APP_TOKENS,
                "addAppToken()")) {
            throw new SecurityException("Requires MANAGE_APP_TOKENS permission");
        }

        // Get the dispatching timeout here while we are not holding any locks so that it
        // can be cached by the AppWindowToken.  The timeout value is used later by the
        // input dispatcher in code that does hold locks.  If we did not cache the value
        // here we would run the chance of introducing a deadlock between the window manager
        // (which holds locks while updating the input dispatcher state) and the activity manager
        // (which holds locks while querying the application token).
        long inputDispatchingTimeoutNanos;
        try {
            inputDispatchingTimeoutNanos = token.getKeyDispatchingTimeout() * 1000000L;
        } catch (RemoteException ex) {
            Slog.w(TAG_WM, "Could not get dispatching timeout.", ex);
            inputDispatchingTimeoutNanos = DEFAULT_INPUT_DISPATCHING_TIMEOUT_NANOS;
        }

        synchronized(mWindowMap) {
            AppWindowToken atoken = findAppWindowToken(token.asBinder());
            if (atoken != null) {
                Slog.w(TAG_WM, "Attempted to add existing app token: " + token);
                return;
            }
            atoken = new AppWindowToken(this, token, voiceInteraction);
            atoken.inputDispatchingTimeoutNanos = inputDispatchingTimeoutNanos;
            atoken.appFullscreen = fullscreen;
            atoken.showForAllUsers = showForAllUsers;
            atoken.targetSdk = targetSdkVersion;
            atoken.requestedOrientation = requestedOrientation;
            atoken.layoutConfigChanges = (configChanges &
                    (ActivityInfo.CONFIG_SCREEN_SIZE | ActivityInfo.CONFIG_ORIENTATION)) != 0;
            atoken.mLaunchTaskBehind = launchTaskBehind;
            atoken.mAlwaysFocusable = alwaysFocusable;
            if (DEBUG_TOKEN_MOVEMENT || DEBUG_ADD_REMOVE) Slog.v(TAG_WM, "addAppToken: " + atoken
                    + " to stack=" + stackId + " task=" + taskId + " at " + addPos);

            Task task = mTaskIdToTask.get(taskId);
            if (task == null) {
                task = createTaskLocked(taskId, stackId, userId, atoken, taskBounds, config);
            }
            task.addAppToken(addPos, atoken, taskResizeMode, homeTask);

            mTokenMap.put(token.asBinder(), atoken);

            // Application tokens start out hidden.
            atoken.hidden = true;
            atoken.hiddenRequested = true;
        }
    }
```
&emsp;&emsp;该方法中会new一个AppWindowToken对象atoken，然后使用token的asBinder的返回值最为key来保存atoken。在【2.2】节中的addWindow方法中会从mTokenMap变量中获取token。

&emsp;&emsp;至此，token的创建、添加以及使用理清楚了。总结一下就是：

1. 启动Activity的时候AMS会创建token
2. 创建token之后AMS会把它添加到WMS中
3. Dialog显示的时候会在ViewRootImpl的setView中调用WMS的addWindow方法，此时会查询token是否存在，如果不存在最后就会抛出BadTokenException。
&emsp;&emsp;到这里问题好像是解释清楚了，可是Dialog中的token来自哪里呢？

#### Dialog的token来自哪里

##### 倒推法跟踪token的来源
&emsp;&emsp;Dialog也包含一个Window和WM对象，当Dialog show的时候最终会调用ViewRootImpl的setView方法（见【1】节崩溃堆栈），进而会调用WMS的addWindow方法，这个方法的WindowManager.LayoutParams类型的参数attrs中包含了一个token，WMS在mTokenMap中查询这个token是否存在，不存在就会抛出异常。这个attrs中的token来自哪里呢？

&emsp;&emsp;从【1】中的崩溃栈可以看出，这个attrs可能来自于Dialog的show方法，改方法源码如下：

```java
public void show() {
        //省略部分代码
        WindowManager.LayoutParams l = mWindow.getAttributes();
        //省略部分代码
        mWindowManager.addView(mDecor, l);
}
```
&emsp;&emsp;可以看到，WMS方法addWindow的attrs参数来自于Dialog的mWindow对象，这是个PhoneWindow对象，该对象在Dialog的构造方法中new出来的，PhoneWindow的getAttributes方法返回的是其父类Window的mWindowAttributes成员，该成员变量在声明的时候直接被赋值为一个new出来WindowManager.LayoutParams对象。前面我们说过Dialog的窗体类型为TYPE\_APPLICATION，原因就在于此。

&emsp;&emsp;不过到目前为止开没有看到给WindowManager.LayoutParams对象的toke属性赋值，它仍然为null。不急，慢慢来，先看一下WindowManagerGlobal对象的addView方法，这是把View添加到WMS的一个中间环节，源码如下：

```java
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        //省略部分代码
        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        if (parentWindow != null) {
           parentWindow.adjustLayoutParamsForSubWindow(wparams);
        }
        //省略部分代码
}
```
&emsp;&emsp;可以看到，当parentWindow参数不为null的时候，会调用Window的adjustLayoutParamsForSubWindow方法。这个参数从WindowManagerImpl的addView方法传进来的，源码如下：

```java
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }
```
&emsp;&emsp;这个mParentWindow不是null，它其实是Activity对应的Window对象（这句很重要，圈起来要考），这个后面会解释。因此adjustLayoutParamsForSubWindow会被调用，该方法的源码如下：

```java
void adjustLayoutParamsForSubWindow(WindowManager.LayoutParams wp) {
//省略部分代码 
           if (wp.token == null) {
                wp.token = mContainer == null ? mAppToken : mContainer.mAppToken;
            }
//省略部分代码
}
```
&emsp;&emsp;该方法根据wp的type不同对应不同的处理，对于Dialog来说，有用的代码就是上面贴出来的。根据前面的分析知道，wp的token就是null，mContainer也是为null，因此wp的token就被赋值为mAppToken。那个这个MAPPToken又是个啥呢？它是在Window类的源码的setWindowManager方法被赋值的。

&emsp;&emsp;在Dialog的源码中可以看到Window对象调用setWindowManager方法时传入的token为null！WTF，感情前面一堆都白分析了？No！这篇文章中最精彩的部分马上就要来了。扶好扶手，马上要飙车了。

##### Activity和Dialog是如何共用token的

&emsp;&emsp;前面遇到了分析到最后token为null的尴尬。接下来就要实现华丽的逆转。

&emsp;&emsp;还记得前面说过一个圈起来要考的重点吗？上面之所以分析到最后token为null，关键在于我们分析错了Window对象，其实我们调用的是parentWindow（即Activity）的adjustLayoutParamsForSubWindow方法，是否有点蒙圈？刚开始我也是的。

&emsp;&emsp;把上面的分析先暂放一下，先看看Dialog构造方法的源码：

```java
public Dialog(@NonNull Context context, @StyleRes int themeResId) {
        this(context, themeResId, true);
    }

    Dialog(@NonNull Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {
        if (createContextThemeWrapper) {
            if (themeResId == 0) {
                final TypedValue outValue = new TypedValue();
                context.getTheme().resolveAttribute(R.attr.dialogTheme, outValue, true);
                themeResId = outValue.resourceId;
            }
            mContext = new ContextThemeWrapper(context, themeResId);
        } else {
            mContext = context;
        }

        //这句就是关键
        mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);

        final Window w = new PhoneWindow(mContext);
        mWindow = w;
        w.setCallback(this);
        w.setOnWindowDismissedCallback(this);
        w.setWindowManager(mWindowManager, null, null);
        w.setGravity(Gravity.CENTER);

        mListenersHandler = new ListenersHandler(this);
    }
```
&emsp;&emsp;从上面的代码中可以看到，mWindowManager是通过context的getSystemService方法获得的，这个context是什么非常关键。一般情况下这个context都是一个Activity，因此可以看看Activity类中的getSystemService方法，源码如下：

```java
@Override
public Object getSystemService(@ServiceName @NonNull String name) {
        if (getBaseContext() == null) {
            throw new IllegalStateException(
                    "System services not available to Activities before onCreate()");
        }

        if (WINDOW_SERVICE.equals(name)) {
            return mWindowManager;
        } else if (SEARCH_SERVICE.equals(name)) {
            ensureSearchManager();
            return mSearchManager;
        }
        return super.getSystemService(name);
    }
```
&emsp;&emsp;Activity重写了getSystemService，对应WMS的获取进行了拦截，直接返回了它自己的mWindowManager变量。它是在Activity的attach方法中被赋值的，见上面的代码。在Activity的attach方法中调用了Activity成员mWindow的setWindowManager方法，源码如下：

```java
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
            boolean hardwareAccelerated) {
        mAppToken = appToken;
        mAppName = appName;
        mHardwareAccelerated = hardwareAccelerated
                || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
        if (wm == null) {
            wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
        }
        mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
    }
```
&emsp;&emsp;token就是Activity的mToken成员，即Activity的成员mWindow对象的mAppToken为Activity的mToken。接下来，调用WindowManagerImpl的createLocalWindowManager（注意参数为this，即Activity的mWindow对象）方法创建WindowManager对象，这个对象在attach方法中通过mWindow.getWindowManager赋值给Activity的mWindowManager成员。createLocalWindowManager方法的源码如下：

```java
private WindowManagerImpl(Context context, Window parentWindow) {
        mContext = context;
        mParentWindow = parentWindow;
    }

    public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
        return new WindowManagerImpl(mContext, parentWindow);
    }
```
&emsp;&emsp;代码很简单，就是new一个WindowManagerImpl对象，注意这个parentWindow就是Activity中的mWindow对象，这就解释了在【2.4.1】中说的圈起来要考的那句话。

&emsp;&emsp;Activity通过重写getSystemService方法实现和Dialog共用mWindowManager，进而共用token的目的。

&emsp;&emsp;总结一下：

1. Activity重写getSystemService，返回自己的mWindowManager。
2. Activity的mWindowManager成员把Activity的mWindow保存在自己的mParentWindow成员中
3. Activity的mWindow成员把Activity的mToken成员保存在自己的mAppToken中
4. Dialog在构造中调用context的getSystemService返回一个WM对象
5. Dialog在show方法调用4中返回的WM对象的addView方法
6. WM对象接着调用WindowManagerGlobal的addView方法，并传进去WM对象的mParentWindow成员
7. 在WindowManagerGlobal 的addView中调用parentWindow的adjustLayoutParamsForSubWindow，把parentWindow的mAppToken赋值给attrs的token属性，注意这个token是Activity的mToken。

#### Q&A

##### 为什么创建Dialog时的context参数只能是Ativity

&emsp;&emsp;如果context不为Activity对象，则通过context的getSystemService方法返回的WM对象的mParentWindow成员即为null，因此调用WM对象的addView方法时最终不会给WindowManager.LayoutParams对象的token赋值，即该值仍未null，当调用到WMS的addWindow方法时就不能根据这个null找到对应的token（APPWindowToken），因此就会返回一个错误的值导致抛出BadTokenException。

##### 为什么Dialog依赖的Activity finish之后，Dialog不能show

&emsp;&emsp;看Activity的启动流程就可以知道，当一个Activity finish的时候就会把WMS中对应的token给删除掉，因此当调用show的时候，虽然最终调用WMS的addWindow方法时attrs的token不为null，但是在WMS中仍不能查找到，因此会返回一个错误值导致抛出BadTokenException。

##### 为什么有些情况下Activity启动时按home键之后应用会崩溃

&emsp;&emsp;按home键之后，系统会回到launcher界面，同时AMS会stop应用的Activity的，如果该Activity在manifest文件中设置了noHistory属性为true时，stop方法就会启动一个定时，超过10秒时Activity仍未执行完stop时AMS就会在server端执行finish操作，从而删除Activity对应的token，接下来就会发生和【2.5.2】相同的情况。之所以会超时，是因为在Application类或Activity的onCreate方法中执行的时间太久，当server端把token删除之后，client端的Activity才执行view的添加（即resume），这样就会发生BadTokenException。注意，虽然client端被卡主，但是server端（AMS）仍然能执行stop和finish操作，因为他们是两个进程。

&emsp;&emsp;不过我们应用的首页Activity并没有设置noHistory属性，但是仍然在启动时跑出了BadTokenException，就显得很诡异，由于无法复现，暂时还未分析出原因。
