---
title: Android-ContentProvider的装载流程
date: 2019-05-29 19:22:00
tags:
	- Android
	- ContentProvider

category: Android
comments: ture
---

前两天项目中使用了 `android jetpak`中的`lifecyle-process`组件，帮助我们管理生命周期，接入后发现不需要任何初始化操作，它的源码也比较简单，就4个文件，发现它使用`ContentProvider`的装载特性来做自动初始化操作。联想到多年前在做插件化时遇到`ContentProvider` 需要提前处理的情况。才发现，`Google`也会`投机取巧`啦。  本文我们就分析一下`ContentProvider`的装载流程。  

### 1.ContentProvider 启动装载的场景  

先把结论抛出来,触发 `ContentProvider`装载的几个场景有:  
 
 * 进程启动，初始化Application时
 * 三方应用通过`ContentResolver`调用`ContentProvider`的相关功能时

 我们先跟踪一下 进程启动时的场景  
 
### 2. ContentProvider 进程启动装载  

我们都知道 `Android`进程启动需要给定一个入口类，而每个程序的进程入口类就是`ActivityThread`。  

_我们基于android-28，也就是Android P_  

`ActivityThread`的入口函数为`main`  

```java
public static void main(String[] args) {
		 ...
        Looper.prepareMainLooper();
        
        ...
        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
		  ...
    }
```

会调用 `ActivityThread `的`attach`方法  

```java
private void attach(boolean system, long startSeq) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {//我们的进程属于 非系统进程，会执行该逻辑
        	  ...
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManager.getService();
            try {
                // 通知 AMS 执行 attachApplication 操作
                mgr.attachApplication(mAppThread, startSeq);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            ...
        } else {//系统进程执行的逻辑，因为允许在系统进程，直接生产 Application 实例，执行生命周期方法即可
            ...
            try {
                mInstrumentation = new Instrumentation();
                mInstrumentation.basicInit(this);
                ContextImpl context = ContextImpl.createAppContext(
                        this, getSystemContext().mPackageInfo);
                mInitialApplication = context.mPackageInfo.makeApplication(true, null);
                mInitialApplication.onCreate();
            } catch (Exception e) {...}
        }
        ...
    }
```

我继续跟入 `ActivityManagerService`的`attachApplication` 方法  

```java
@Override
    public final void attachApplication(IApplicationThread thread, long startSeq) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid, callingUid, startSeq);
            Binder.restoreCallingIdentity(origId);
        }
    }
```

继续`attachApplicationLocked `方法  

```java
private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid, int callingUid, long startSeq) {
            //前面 获取 ProcessRecord
            ...
            
             //因为系统进程已经ready ，所以这里就会向 PackageManagerService 查询 apk的所有 ContentProvider 标签对应的 ProviderInfo
             boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
        	 List<ProviderInfo> providers = normalMode ? generateApplicationProvidersLocked(app) : null;
        	 
        	 ...
        	 
        	//到目前为止 已经有了 ContentProvider的信息了，但是 ConentProvider 还没有装载
        	
        	if (app.isolatedEntryPoint != null) {
                // This is an isolated process which should just call an entry point instead of
                // being bound to an application.
                thread.runIsolatedEntryPoint(app.isolatedEntryPoint, app.isolatedEntryPointArgs);
            } else if (app.instr != null) {
                thread.bindApplication(processName, appInfo, providers,
                        app.instr.mClass,
                        profilerInfo, app.instr.mArguments,
                        app.instr.mWatcher,
                        app.instr.mUiAutomationConnection, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.persistent,
                        new Configuration(getGlobalConfiguration()), app.compat,
                        getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, isAutofillCompatEnabled);
            } else {
                thread.bindApplication(processName, appInfo, providers, null, profilerInfo,
                        null, null, null, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.persistent,
                        new Configuration(getGlobalConfiguration()), app.compat,
                        getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, isAutofillCompatEnabled);
            }
  }
```

正常情况下我们没有使用 `Instrumentation`所以 会执行 最后一个 `else`  

以后执行时序会再次进入到 `ActivityThread`中，也就是从AMS 进程进入到了我们的App进程中 ，同时把前面的  `providers`传入。  

```java
 public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableBinderTracking, boolean trackAllocation,
                boolean isRestrictedBackupMode, boolean persistent, Configuration config,
                CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
                String buildSerial, boolean autofillCompatibilityEnabled) {

            setCoreSettings(coreSettings);

            AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providers;
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.instrumentationUiAutomationConnection = instrumentationUiConnection;
            data.debugMode = debugMode;
            data.enableBinderTracking = enableBinderTracking;
            data.trackAllocation = trackAllocation;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfilerInfo = profilerInfo;
            data.buildSerial = buildSerial;
            data.autofillCompatibilityEnabled = autofillCompatibilityEnabled;
            sendMessage(H.BIND_APPLICATION, data);
        }
``` 
 到目前为止，通过`Handler` 消息 将 `Binder` 进程释放，这样就避免了一个App的启动持续 Block 住系统进程  
 
 ```java
 public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    handleBindApplication(data);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
 ```
 
 之后又进入到了 `handleBindApplication `方法  
 
 ```java
 private void handleBindApplication(AppBindData data) {
 
 	...
 	final InstrumentationInfo ii;
 	...
 	if (ii != null) {
 	 try {
 	 			...
                final ClassLoader cl = instrContext.getClassLoader();
                mInstrumentation = (Instrumentation)
                    cl.loadClass(data.instrumentationName.getClassName()).newInstance();
            } catch (Exception e) {
                throw new RuntimeException(
                    "Unable to instantiate instrumentation "
                    + data.instrumentationName + ": " + e.toString(), e);
            }
         ...
 	} else { //按照前面的分析，如果没有 AMS 调用时没有传入 instrumentationName，就会自动生成一个
 		mInstrumentation = new Instrumentation();
      mInstrumentation.basicInit(this);
 	}
 	
 	...
 	 try {
            
            //生成Application实例，内部还会调用Application 的 attach 方法
            app = data.info.makeApplication(data.restrictedBackupMode, null);

            // Propagate autofill compat state
            app.setAutofillCompatibilityEnabled(data.autofillCompatibilityEnabled);

            mInitialApplication = app;

            // don't bring up providers in restricted mode; they may depend on the
            // app's custom Application class
            if (!data.restrictedBackupMode) {
                if (!ArrayUtils.isEmpty(data.providers)) {
                // ！！！！！重点重点，这里就开启装载ContentProvider的流程
                    installContentProviders(app, data.providers);
                    // For process that contains content providers, we want to
                    // ensure that the JIT is enabled "at some point".
                    mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);
                }
            }

            // Do this after providers, since instrumentation tests generally start their
            // test thread at this point, and we don't want that racing.
            try {
                           mInstrumentation.onCreate(data.instrumentationArgs);
            }
            catch (Exception e) {
                throw new RuntimeException(
                    "Exception thrown in onCreate() of "
                    + data.instrumentationName + ": " + e.toString(), e);
            }
            try {
             //!!!!! 通过 mInstrumentation 调用 Application的 onCreate 方法
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!mInstrumentation.onException(app, e)) {
                    throw new RuntimeException(
                      "Unable to create application " + app.getClass().getName()
                      + ": " + e.toString(), e);
                }
            }
        } finally {
            // If the app targets < O-MR1, or doesn't change the thread policy
            // during startup, clobber the policy to maintain behavior of b/36951662
            if (data.appInfo.targetSdkVersion < Build.VERSION_CODES.O_MR1
                    || StrictMode.getThreadPolicy().equals(writesAllowedPolicy)) {
                StrictMode.setThreadPolicy(savedPolicy);
            }
        }
 }
 ```
 
 通过这段代码分析，我们清楚的知道，`ContentProvider`的装载时机在 `Application`的`attach`之后，在 `onCreate`之前。这就是当初做插件化的时候，必须在`onCreate`之前替换掉 `ProviderInfo`的原因,否则装载 `ContentProvider`必定失败。  
 
 下面我们再分析一下 `installContentProviders`方法  
 
 ```java
 private void installContentProviders(
            Context context, List<ProviderInfo> providers) {
        final ArrayList<ContentProviderHolder> results = new ArrayList<>();

        for (ProviderInfo cpi : providers) {
            //遍历装载每一个 ContentProvider
            ContentProviderHolder cph = installProvider(context, null, cpi,
                    false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
            if (cph != null) {
                cph.noReleaseNeeded = true;
                results.add(cph);
            }
        }

        try {
        		//装载完成后通知 AMS，ContentProvider 装载完毕，后面就可以正常使用了
            ActivityManager.getService().publishContentProviders(
                getApplicationThread(), results);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
 ```
 
 ```java
  private ContentProviderHolder installProvider(Context context,
            ContentProviderHolder holder, ProviderInfo info,
            boolean noisy, boolean noReleaseNeeded, boolean stable) {
     ContentProvider localProvider = null;
        IContentProvider provider;
        if (holder == null || holder.provider == null) {//前面传入的 holder 为null
                        Context c = null;
            ApplicationInfo ai = info.applicationInfo;
            ...
            
            if (info.splitName != null) {
                try {
                    c = c.createContextForSplit(info.splitName);
                } catch (NameNotFoundException e) {
                    throw new RuntimeException(e);
                }
            }

            try {
                final java.lang.ClassLoader cl = c.getClassLoader();
                LoadedApk packageInfo = peekPackageInfo(ai.packageName, true);
                if (packageInfo == null) {
                    // System startup case.
                    packageInfo = getSystemContext().mPackageInfo;
                }
                //通过反射得到生成一个 Provider 的实例
                localProvider = packageInfo.getAppFactory()
                        .instantiateProvider(cl, info.name);
                        //获取Provider的Binder接口
                provider = localProvider.getIContentProvider();
                if (provider == null) {
                    Slog.e(TAG, "Failed to instantiate class " +
                          info.name + " from sourceDir " +
                          info.applicationInfo.sourceDir);
                    return null;
                }
                //该方法最终会调动 Provider的 onCreate 方法
                localProvider.attachInfo(c, info);
            } catch (java.lang.Exception e) {
                              return null;
            }
        } else {
            provider = holder.provider;
        }   
  }
 ```
 
 到目前为止，`ContentProvider` 已经启动完成。 
 后面的操作就是建立 `ContentProviderHolder`,将 `ContentProviderHolder` 存放到`mProviderMap`,`mLocalProviders`,`mLocalProvidersByName`等集合中，方便下次使用和索引。  
 
 再回到调用栈的上一级`installContentProviders`  
 
 ```java
 private void installContentProviders(
            Context context, List<ProviderInfo> providers) {
        final ArrayList<ContentProviderHolder> results = new ArrayList<>();

        for (ProviderInfo cpi : providers) {
            if (DEBUG_PROVIDER) {
                StringBuilder buf = new StringBuilder(128);
                buf.append("Pub ");
                buf.append(cpi.authority);
                buf.append(": ");
                buf.append(cpi.name);
                Log.i(TAG, buf.toString());
            }
            ContentProviderHolder cph = installProvider(context, null, cpi,
                    false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
            if (cph != null) {
                cph.noReleaseNeeded = true;
                results.add(cph);
            }
        }

        try {
            ActivityManager.getService().publishContentProviders(
                getApplicationThread(), results);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
 ```
 
 在该方法的最后执行了  
  
 ```java
 ActivityManager.getService().publishContentProviders(
                getApplicationThread(), results);
 ```
 它会通知`AMS`,`AMS `进程会将 `providers`信息存储在 成员变量中，方便其他进程使用  
 
 
 ```java
 public final void publishContentProviders(IApplicationThread caller,
            List<ContentProviderHolder> providers) {
        if (providers == null) {
            return;
        }

        enforceNotIsolatedCaller("publishContentProviders");
        synchronized (this) {
        		//获取调用者的进程信息，这里就是我们provider程序的进程信息
            final ProcessRecord r = getRecordForAppLocked(caller);
            ...

            final long origId = Binder.clearCallingIdentity();

            final int N = providers.size();
            for (int i = 0; i < N; i++) {//遍历装载好的Provider
                ContentProviderHolder src = providers.get(i);
                if (src == null || src.info == null || src.provider == null) {
                    continue;
                }
                //从之前解析好的ProviderInfo 列表中查找 出对应的 ContentProviderRecord
                ContentProviderRecord dst = r.pubProviders.get(src.info.name);
                if (DEBUG_MU) Slog.v(TAG_MU, "ContentProviderRecord uid = " + dst.uid);
                if (dst != null) {//因为在Provider装载之前就通过 generateApplicationProvidersLocked 方法生成了ContentProviderRecord实例，所以不会为null
                    ComponentName comp = new ComponentName(dst.info.packageName, dst.info.name);
                    //!!!! 重点:将Provider的信息记录到mProviderMap中，方便下次调用
                    mProviderMap.putProviderByClass(comp, dst);
                    String names[] = dst.info.authority.split(";");
                    for (int j = 0; j < names.length; j++) {
                        mProviderMap.putProviderByName(names[j], dst);
                    }
						...//省略部分的操作是将等待Provider装载的状态重置
                    synchronized (dst) {
                    	//将 刚装载成功的Provider的Binder接口记录到ContentProviderRecord中，在此之前，ContentProviderRecord实例中都没有Provider 的Binder引用，都是不可用状态
                        dst.provider = src.provider;
                        dst.proc = r;
                        dst.notifyAll();
                    }
                    updateOomAdjLocked(r, true);
                    maybeUpdateProviderUsageStatsLocked(r, src.info.packageName,
                            src.info.authority);
                }
            }

            Binder.restoreCallingIdentity(origId);
        }
    }
 ```
 
 从此之后，`ContentProvider`已经是Ready状态了，可以直接供使用了。  
 
### 3. Client 调用 ContentProvider  

常规的调用方式是通过`ContentResolver`,获取`ContentResolver ` 都是通过`Context`来实现  

_android.content.Context.java_

```java
public abstract ContentResolver getContentResolver();
```

_android.content.ContextWrapper.java_

```java
@Override
    public ContentResolver getContentResolver() {
        return mBase.getContentResolver();
    }
```

我们知道真正的`Context`实例是 `ContextImpl`类型的，下面分析一下`ContextImpl `的实现方式:  
_android.content. ContextImpl.java_  

```java
@Override
    public ContentResolver getContentResolver() {
        return mContentResolver;
    }
```
`mContentResolver` 对象是 `ApplicationContentResolver`类型的  

```java
private static final class ApplicationContentResolver extends ContentResolver {
        private final ActivityThread mMainThread;

        public ApplicationContentResolver(Context context, ActivityThread mainThread) {
            super(context);
            mMainThread = Preconditions.checkNotNull(mainThread);
        }

        @Override
        protected IContentProvider acquireProvider(Context context, String auth) {
            return mMainThread.acquireProvider(context,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), true);
        }

        @Override
        protected IContentProvider acquireExistingProvider(Context context, String auth) {
            return mMainThread.acquireExistingProvider(context,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), true);
        }

        @Override
        public boolean releaseProvider(IContentProvider provider) {
            return mMainThread.releaseProvider(provider, true);
        }

        @Override
        protected IContentProvider acquireUnstableProvider(Context c, String auth) {
            return mMainThread.acquireProvider(c,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), false);
        }

        @Override
        public boolean releaseUnstableProvider(IContentProvider icp) {
            return mMainThread.releaseProvider(icp, false);
        }

        @Override
        public void unstableProviderDied(IContentProvider icp) {
            mMainThread.handleUnstableProviderDied(icp.asBinder(), true);
        }

        @Override
        public void appNotRespondingViaProvider(IContentProvider icp) {
            mMainThread.appNotRespondingViaProvider(icp.asBinder());
        }

        /** @hide */
        protected int resolveUserIdFromAuthority(String auth) {
            return ContentProvider.getUserIdFromAuthority(auth, getUserId());
        }
    }
```

我们以 `ContentResolver`的`insert`方法作为切入点，进行分析  

```java
public final @Nullable Uri insert(@RequiresPermission.Write @NonNull Uri url,
                @Nullable ContentValues values) {
        Preconditions.checkNotNull(url, "url");
        //1.首先根据URI 找到 ConentProvider的远端Binder引用
        IContentProvider provider = acquireProvider(url);
        if (provider == null) {
            throw new IllegalArgumentException("Unknown URL " + url);
        }
        try {
            long startTime = SystemClock.uptimeMillis();
            //2.调用 ConentProvider
            Uri createdRow = provider.insert(mPackageName, url, values);
            long durationMillis = SystemClock.uptimeMillis() - startTime;
            maybeLogUpdateToEventLog(durationMillis, url, "insert", null /* where */);
            return createdRow;
        } catch (RemoteException e) {
            ...
            return null;
        } finally {
            releaseProvider(provider);
        }
    }
```

由于我们分析的是 `ContentProvider`的装载流程，所以只分析 `acquireProvider `方法  

最终会进入到`ApplicationContentResolver `的`acquireProvider `方法  

```java
@Override
        protected IContentProvider acquireProvider(Context context, String auth) {
            return mMainThread.acquireProvider(context,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), true);
        }
```
mMainThread 就是 ActivityThread 对象  

```java
public final IContentProvider acquireProvider(
            Context c, String auth, int userId, boolean stable) {
            //!!! 首先在本进程空间内查询 已经装载的 ContentProvider
        final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
        if (provider != null) {
            return provider;
        }

        // There is a possible race here.  Another thread may try to acquire
        // the same provider at the same time.  When this happens, we want to ensure
        // that the first one wins.
        // Note that we cannot hold the lock while acquiring and installing the
        // provider since it might take a long time to run and it could also potentially
        // be re-entrant in the case where the provider is in the same process.
        ContentProviderHolder holder = null;
        try {
            synchronized (getGetProviderLock(auth, userId)) {
            //如果本进程空间内不存在ConentProvider，就委托AMS去查询
                holder = ActivityManager.getService().getContentProvider(
                        getApplicationThread(), auth, userId, stable);
            }
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
        if (holder == null) {
            Slog.e(TAG, "Failed to find provider info for " + auth);
            return null;
        }

        // Install provider will increment the reference count for us, and break
        // any ties in the race.
        holder = installProvider(c, holder, holder.info,
                true /*noisy*/, holder.noReleaseNeeded, stable);
        return holder.provider;
    }
```

这个方法 首先支持在本进程空间内查询 `ConentProvider`,其实就是从前面分析中提到的 `mProviderMap`中获取，在 随进程启动时装载的`Provider` 被放在`mProviderMap`中，这时就能直接访问使用了。  但也有可能需要查询不到,出现这种情况的情景如下:  

* 请求的`Provider`需要独立进程允许，但是该进程还未启动，所以 `Provider` 未被装载。
* 请求的`Provider`是第三方程序的，需要通过AMS 获得。
* 请求的`Provider`是系统进程所有的，比如媒体库的`Provider`,这也需要AMS提供

不管是以上三种情况的哪一种，都会进入到AMS， 我们选一种最长路径的情况做分析  
<mark>__假设我们请求的`Provider`是一个第三方APP的，但是这个App 还没有运行。__</mark>  

```java
 public final ContentProviderHolder getContentProvider(
            IApplicationThread caller, String name, int userId, boolean stable) {
        enforceNotIsolatedCaller("getContentProvider");
        if (caller == null) {
            String msg = "null IApplicationThread when getting content provider "
                    + name;
            Slog.w(TAG, msg);
            throw new SecurityException(msg);
        }
        // The incoming user check is now handled in checkContentProviderPermissionLocked() to deal
        // with cross-user grant.
        return getContentProviderImpl(caller, name, null, stable, userId);
    }
```

真正执行的是`getContentProviderImpl `  

```java
private ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
            String name, IBinder token, boolean stable, int userId) {
   ...
     		// First check if this content provider has been published...
            cpr = mProviderMap.getProviderByName(name, userId);
            // If that didn't work, check if it exists for user 0 and then
            // verify that it's a singleton provider before using it.
            if (cpr == null && userId != UserHandle.USER_SYSTEM) {
                cpr = mProviderMap.getProviderByName(name, UserHandle.USER_SYSTEM);
                if (cpr != null) {
                    cpi = cpr.info;
                    if (isSingleton(cpi.processName, cpi.applicationInfo,
                            cpi.name, cpi.flags)
                            && isValidSingletonCall(r.uid, cpi.applicationInfo.uid)) {
                        userId = UserHandle.USER_SYSTEM;
                        checkCrossUser = false;
                    } else {
                        cpr = null;
                        cpi = null;
                    }
                }
            }
             boolean providerRunning = cpr != null && cpr.proc != null && !cpr.proc.killed;
            if (providerRunning) {//根据我们的假设，Provider 是未运行状态
            
            ...         
}
```

首先AMS 从 `mProviderMap`中获取，前面分析随进程启动而装载的Provider 在自己进程装载完成后会通过`publishContentProviders`将 `Provider`列表保持在AMS中，其实就是保存在AMS的`mProviderMap`中。  
根据我们的假设 Provider 所在的进程还未启动，所以 `mProviderMap `中是查询不到的  

```java
if (!providerRunning) {
	
	try {
         checkTime(startTime, "getContentProviderImpl: before resolveContentProvider");
         // ！！！首先通过PMS 查询 Provider的Info信息，为下一步的装载做准备
         cpi = AppGlobals.getPackageManager().
             resolveContentProvider(name,
                            STOCK_PM_FLAGS | PackageManager.GET_URI_PERMISSION_PATTERNS, userId);
                    checkTime(startTime, "getContentProviderImpl: after resolveContentProvider");
       } catch (RemoteException ex) {
       }
    //如果找不大，说明未在 Manifest 文件中注册或者 程序未安装，直接返回NUll
   if (cpi == null) {
       return null;
   }
    
   //是否是singleton？其实就是 Provider在 Manifest 注册的时候是否增加了'singleUser'属性
   //如果 singleUser 属性设置为 true，意味着多用户环境下可以共用Provider对象。
   //其他情况就是系统Provider 是保持单例的
   boolean singleton = isSingleton(cpi.processName, cpi.applicationInfo,
                        cpi.name, cpi.flags)
                        && isValidSingletonCall(r.uid, cpi.applicationInfo.uid);
   if (singleton) {
      userId = UserHandle.USER_SYSTEM;
   } 
   //按照之前的假设，我们要访问的Provider 是不支持多用户共享的，所以需要重新装载
   
   ...
   //后面就是一些检验Provider 的访问权限的校验，这里就不分析了
   
   ...
   //获取目标Provider的进程信息，如果进程信息存在，说明目前进程是启动状态
    ProcessRecord proc = getProcessRecordLocked(
                                cpi.processName, cpr.appInfo.uid, false);
    if (proc != null && proc.thread != null && !proc.killed) {
        if (DEBUG_PROVIDER) Slog.d(TAG_PROVIDER,"Installing in existing process " + proc);
        if (!proc.pubProviders.containsKey(cpi.name)) {
            checkTime(startTime, "getContentProviderImpl: scheduling install");
            proc.pubProviders.put(cpi.name, cpr);
            try {
            //通知目标进程装载指定的Provider
                 proc.thread.scheduleInstallProvider(cpi);
            } catch (RemoteException e) {
              }
            }
        } else {
          checkTime(startTime, "getContentProviderImpl: before start process");
          //如果目标进程未启动，先启动目标进程
          proc = startProcessLocked(cpi.processName,
                                    cpr.appInfo, false, 0, "content provider",
                                    new ComponentName(cpi.applicationInfo.packageName,
                                            cpi.name), false, false, false);
          checkTime(startTime, "getContentProviderImpl: after start process");
          if (proc == null) {
              return null;
          }
     }
     cpr.launchingApp = proc;
     //将ProcessRecord添加到正在等待启动的列表中，如果完成Provider的装载后会从mLaunchingProviders列表中
     mLaunchingProviders.add(cpr);
}
```

从上诉代码中得知:

1. 当目标 Provider 未装载运行时，会通过PMS 获取Provider信息，为装载做准备
2. 检测目标 Provider 的进程是否在运行
3. 如果目标进程已经运行，会自己通知目标进程去装载指定的Provider
4. 如果目标进程未运行，会先启动进程

当是情况`3`时， 代码最终会执行进入 ActivityThread.installContentProviders(Context context, List<ProviderInfo> providers) 中，这个方法在分析 随进程启动时装载Provider 中已经介绍过。  

当是情况`4`时，会执行 AMS 的 `startProcessLocked`方法  

```java
final ProcessRecord startProcessLocked(String processName,
            ApplicationInfo info, boolean knownToBeDead, int intentFlags,
            String hostingType, ComponentName hostingName, boolean allowWhileBooting,
            boolean isolated, boolean keepIfLarge) {
        return startProcessLocked(processName, info, knownToBeDead, intentFlags, hostingType,
                hostingName, allowWhileBooting, isolated, 0 /* isolatedUid */, keepIfLarge,
                null /* ABI override */, null /* entryPoint */, null /* entryPointArgs */,
                null /* crashHandler */);
    }
```

继而又调用 另一个重载方法:  

```java
final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
            boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName,
            boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge,
            String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {
        long startTime = SystemClock.elapsedRealtime();
        ProcessRecord app;
        //正常情况下我们的目标进程不是隔离进程，前一步也的确传入false
        ...

        // We don't have to do anything more if:
        // (1) There is an existing application record; and
        // (2) The caller doesn't think it is dead, OR there is no thread
        //     object attached to it so we know it couldn't have crashed; and
        // (3) There is a pid assigned to it, so it is either starting or
        //     already running.
        //如源码注释所说，启动前判断一下目标进程其实是存活的，什么都不做
        if (app != null && app.pid > 0) {
            if ((!knownToBeDead && !app.killed) || app.thread == null) {
                app.addPackage(info.packageName, info.versionCode, mProcessStats);
                checkTime(startTime, "startProcess: done, added package to proc");
                return app;
            }
            //这种情况下，说明需要认定目标进程已经死了，以下代码处理清除目标进程的操作
            checkTime(startTime, "startProcess: bad proc running, killing");
            killProcessGroup(app.uid, app.pid);
            handleAppDiedLocked(app, true, true);
            checkTime(startTime, "startProcess: done killing old proc");
        }

        String hostingNameStr = hostingName != null
                ? hostingName.flattenToShortString() : null;

			//正常情况下 app 为 null，我们需要启动进程
        if (app == null) {
            checkTime(startTime, "startProcess: creating new process record");
            //创建代表进程状态的 ProcessRecord 对象
            app = newProcessRecordLocked(info, processName, isolated, isolatedUid);
         }
            ...
            //继续启动进程
        final boolean success = startProcessLocked(app, hostingType, hostingNameStr, abiOverride);
        checkTime(startTime, "startProcess: done starting proc!");
        return success ? app : null;
    }
```

继续启动进程  

```java
 private final boolean startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, boolean disableHiddenApiChecks, String abiOverride) {
        //定义进程启动的入口类是 "android.app.ActivityThread"
       final String entryPoint = "android.app.ActivityThread";
       return startProcessLocked(hostingType, hostingNameStr, entryPoint, app, uid, gids,
                    runtimeFlags, mountExternal, seInfo, requiredAbi, instructionSet, invokeWith,
                    startTime);
 }
```
上诉代码规定了，进程的入口是 "android.app.ActivityThread"   

后面会继续进入到 `startProcess`方法  

```java
private ProcessStartResult startProcess(String hostingType, String entryPoint,
            ProcessRecord app, int uid, int[] gids, int runtimeFlags, int mountExternal,
            String seInfo, String requiredAbi, String instructionSet, String invokeWith,
            long startTime) {
        try {
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Start proc: " +
                    app.processName);
            checkTime(startTime, "startProcess: asking zygote to start proc");
            final ProcessStartResult startResult;
            if (hostingType.equals("webview_service")) {
                startResult = startWebView(entryPoint,
                        app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                        app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                        app.info.dataDir, null,
                        new String[] {PROC_START_SEQ_IDENT + app.startSeq});
            } else {
                startResult = Process.start(entryPoint,
                        app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                        app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                        app.info.dataDir, invokeWith,
                        new String[] {PROC_START_SEQ_IDENT + app.startSeq});
            }
            checkTime(startTime, "startProcess: returned from zygote!");
            return startResult;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        }
    }
```
这里发现对 `webview_service` 进程做了单独处理,应该是因为Android接入的是 Chromium 内核,
最后 通过 `Process.start`方法开启真正的进程之旅啊  
下面顺带着简单介绍下进程的启动流程，不过不会深入

```java
public static final ProcessStartResult start(final String processClass,
                                  final String niceName,
                                  int uid, int gid, int[] gids,
                                  int runtimeFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String invokeWith,
                                  String[] zygoteArgs) {
        return zygoteProcess.start(processClass, niceName, uid, gid, gids,
                    runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, invokeWith, zygoteArgs);
    }
```
发现通过 Zygote 进程孵化  

__android.os.ZygoteProcess.java__

```java
public final Process.ProcessStartResult start(final String processClass,
                                                  final String niceName,
                                                  int uid, int gid, int[] gids,
                                                  int runtimeFlags, int mountExternal,
                                                  int targetSdkVersion,
                                                  String seInfo,
                                                  String abi,
                                                  String instructionSet,
                                                  String appDataDir,
                                                  String invokeWith,
                                                  String[] zygoteArgs) {
        try {
            return startViaZygote(processClass, niceName, uid, gid, gids,
                    runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, invokeWith, false /* startChildZygote */,
                    zygoteArgs);
        } catch (ZygoteStartFailedEx ex) {
            ...
        }
    }
    
    
private Process.ProcessStartResult startViaZygote(final String processClass,
                                                      final String niceName,
                                                      final int uid, final int gid,
                                                      final int[] gids,
                                                      int runtimeFlags, int mountExternal,
                                                      int targetSdkVersion,
                                                      String seInfo,
                                                      String abi,
                                                      String instructionSet,
                                                      String appDataDir,
                                                      String invokeWith,
                                                      boolean startChildZygote,
                                                      String[] extraArgs)
                                                      throws ZygoteStartFailedEx {
        ArrayList<String> argsForZygote = new ArrayList<String>();

		// 添加一堆 Zygote 参数
        // --runtime-args, --setuid=, --setgid=,
        // and --setgroups= must go first
        argsForZygote.add("--runtime-args");
        argsForZygote.add("--setuid=" + uid);
        argsForZygote.add("--setgid=" + gid);
        argsForZygote.add("--runtime-flags=" + runtimeFlags);
        if (mountExternal == Zygote.MOUNT_EXTERNAL_DEFAULT) {
            argsForZygote.add("--mount-external-default");
        } else if (mountExternal == Zygote.MOUNT_EXTERNAL_READ) {
            argsForZygote.add("--mount-external-read");
        } else if (mountExternal == Zygote.MOUNT_EXTERNAL_WRITE) {
            argsForZygote.add("--mount-external-write");
        }
        argsForZygote.add("--target-sdk-version=" + targetSdkVersion);

        // --setgroups is a comma-separated list
        if (gids != null && gids.length > 0) {
            StringBuilder sb = new StringBuilder();
            sb.append("--setgroups=");

            int sz = gids.length;
            for (int i = 0; i < sz; i++) {
                if (i != 0) {
                    sb.append(',');
                }
                sb.append(gids[i]);
            }

            argsForZygote.add(sb.toString());
        }

        if (niceName != null) {
            argsForZygote.add("--nice-name=" + niceName);
        }

        if (seInfo != null) {
            argsForZygote.add("--seinfo=" + seInfo);
        }

        if (instructionSet != null) {
            argsForZygote.add("--instruction-set=" + instructionSet);
        }

        if (appDataDir != null) {
            argsForZygote.add("--app-data-dir=" + appDataDir);
        }

        if (invokeWith != null) {
            argsForZygote.add("--invoke-with");
            argsForZygote.add(invokeWith);
        }

        if (startChildZygote) {
            argsForZygote.add("--start-child-zygote");
        }

        argsForZygote.add(processClass);

        if (extraArgs != null) {
            for (String arg : extraArgs) {
                argsForZygote.add(arg);
            }
        }

        synchronized(mLock) {
            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
        }
    }
```
需要注意的是 __`openZygoteSocketIfNeeded(abi)`__ 这里打开了  zygote的socket连接，说明 孵化进程通过经典的`socket `进行通信。有兴趣的同学可以深入了解下  

`zygote socket` 打开之后，我们继续分析`zygoteSendArgsAndGetResult` 方法  

```java
private static Process.ProcessStartResult zygoteSendArgsAndGetResult(
            ZygoteState zygoteState, ArrayList<String> args)
            throws ZygoteStartFailedEx {
        try {
            int sz = args.size();
            for (int i = 0; i < sz; i++) {
                if (args.get(i).indexOf('\n') >= 0) {
                    throw new ZygoteStartFailedEx("embedded newlines not allowed");
                }
            }
            //往 socket 中写入命令数据，包括 进程的入口类
            final BufferedWriter writer = zygoteState.writer;
            final DataInputStream inputStream = zygoteState.inputStream;

            writer.write(Integer.toString(args.size()));
            writer.newLine();

            for (int i = 0; i < sz; i++) {
                String arg = args.get(i);
                writer.write(arg);
                writer.newLine();
            }

            writer.flush();

            // Should there be a timeout on this?
            Process.ProcessStartResult result = new Process.ProcessStartResult();

			//从 socket 中读取启动结果
            result.pid = inputStream.readInt();
            result.usingWrapper = inputStream.readBoolean();

            if (result.pid < 0) {
                throw new ZygoteStartFailedEx("fork() failed");
            }
            return result;
        } catch (IOException ex) {
            zygoteState.close();
            throw new ZygoteStartFailedEx(ex);
        }
    }
```  
再往后 具体如何调用 `ActivityThread`的`main`方法就不做具体深入了。  
最终会执行`ActivityThread`的`main`，这又回到了文章开头介绍的，进程启动时装载`Provider`的场景。  

### 4.总结  

1. `Provider`的装载分两种场景，分别是 _随进程启动而装载 和 被调用时装载_
2. 随进程启动的`Provider `装载时机在`Application`的`onCreate`方法之前，`attach`方法之后
3. 装载后的`Provider`会记录在本进程空间的`mProviderMap`中，同时也会通知 AMS，AMS 也会保持在它自己的 `mProviderMap `中
4. 在 Client 通过`ContentResolver`调用`Provider`时 首先到调用者的进程空间中去查询，如果存在直接返回使用; 如果不存在，通知AMS 去获取。如果AMS中直接有保持的实例，直接返回实例，否则AMS去装载Provider。
5. AMS 去装载时有区分是否目标进程已经存在还是未存在; 如果已经存在，直接调用 目标进程的`scheduleInstallProvider`方法去装载；如果目标进程不存在，同 `zygote` 进程去启动进程,进而 `Provider` 随着进程启动而装载。
6. 还需要注意的是，如果`Provider`是设置了 `singleUser `属性，会被多个用户共享。
7. 如第`4`条。如果通过`AMS` 获取到了别的进程的 `Provider`实例，需要在自己的进程空间保留一份引用，方便下次调用时，直接返回，而不再需要通过AMS获取。












            



