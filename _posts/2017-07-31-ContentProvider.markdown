---
layout: post
title:  "[Android] ContentProvider may kill processes"
date:   2017-07-31 23:26:00 +0800
categories: Android
---
ContentProvider is one of the Android 4 types of App Components. We can query or modify the data through ContentProvider between processes. In this scenario, the process that provider the data is called server process, the process that request the data is called client process. However,  unfortunately and unexpectedly, in some cases, the client process may be killed by the ActivityManagerService.

## Log Analysis

```
06-06 21:57:52.892   916  2275 I ActivityManager: Start proc 26931:com.example.music/u0a103 for content provider com.example.music/.sharedfileaccessor.ContentProviderImpl
06-06 21:57:53.393   916   941 I ActivityManager: Process com.example.music (pid 26931) has died
06-06 21:57:53.423   916   941 I ActivityManager: Killing 16141:com.example.music:service/u0a103 (adj 2): depends on provider com.example.music/.sharedfileaccessor.ContentProviderImpl in dying proc com.example.music
```

There are 3 lines of important logs that ActivityManagerService kills the client process:

1. The server process of the ContentProvider has not started, so the ActivityManagerService starts the server process;
2. Server Process is killed in some cases, maybe because of low memory killer;
3. ActivityManagerService kills the innocent client process of the ContentProvider because the server process has died;

Why does the ActivityMangerService kill client process? We should learn from the Android source code.

## Clear the ContentProviders of dying process

Let's see what will the ActivityManagerService do for the dying process base on the clue of "had died". (Only the lines code relative are shown.)
```java
    final void appDiedLocked(ProcessRecord app, int pid, IApplicationThread thread,
            boolean fromBinderDied) {
        // Clean up already done if the process has been re-started.
        if (app.pid == pid && app.thread != null &&
                app.thread.asBinder() == thread.asBinder()) {
            boolean doLowMem = app.instrumentationClass == null;
            boolean doOomAdj = doLowMem;
            if (!app.killedByAm) {
                // The "has died" log is printed here!!!
                Slog.i(TAG, "Process " + app.processName + " (pid " + pid
                        + ") has died");
                mAllowLowerMemLevel = true;
            } else {
                // Note that we always want to do oom adj to update our state with the
                // new number of procs.
                mAllowLowerMemLevel = false;
                doLowMem = false;
            }
            EventLog.writeEvent(EventLogTags.AM_PROC_DIED, app.userId, app.pid, app.processName);
            if (DEBUG_CLEANUP) Slog.v(TAG_CLEANUP,
                "Dying app: " + app + ", pid: " + pid + ", thread: " + thread.asBinder());
            
            handleAppDiedLocked(app, false, true);

            if (doOomAdj) {
                updateOomAdjLocked();
            }
            if (doLowMem) {
                doLowMemReportIfNeededLocked(app);
            }
        } else if (app.pid != pid) {
            // A new process has already been started.
            Slog.i(TAG, "Process " + app.processName + " (pid " + pid
                    + ") has died and restarted (pid " + app.pid + ").");
            EventLog.writeEvent(EventLogTags.AM_PROC_DIED, app.userId, app.pid, app.processName);
        } else if (DEBUG_PROCESSES) {
            Slog.d(TAG_PROCESSES, "Received spurious death notification for thread "
                    + thread.asBinder());
        }
    }
```

The "has died" log is printed in the `appDiedLocked()` of ActivityManagerService. And then let's dig into the `handleAppDiedLocked()` method.

```java
    private final void handleAppDiedLocked(ProcessRecord app,
            boolean restarting, boolean allowRestart) {
        int pid = app.pid;
        boolean kept = cleanUpApplicationRecordLocked(app, restarting, allowRestart, -1);
    }

    private final boolean cleanUpApplicationRecordLocked(ProcessRecord app,
            boolean restarting, boolean allowRestart, int index) {
        // Take care of any launching providers waiting for this process.
        if (cleanupAppInLaunchingProvidersLocked(app, false)) {
            restart = true;
        }
    }

    boolean cleanupAppInLaunchingProvidersLocked(ProcessRecord app, boolean alwaysBad) {
        // Look through the content providers we are waiting to have launched,
        // and if any run in this process then either schedule a restart of
        // the process or kill the client waiting for it if this process has
        // gone bad.
        boolean restart = false;
        for (int i = mLaunchingProviders.size() - 1; i >= 0; i--) {
            ContentProviderRecord cpr = mLaunchingProviders.get(i);
            if (cpr.launchingApp == app) {
                if (!alwaysBad && !app.bad && cpr.hasConnectionOrHandle()) {
                    restart = true;
                } else {
                    removeDyingProviderLocked(app, cpr, true);
                }
            }
        }
        return restart;
    }


    private final boolean removeDyingProviderLocked(ProcessRecord proc,
            ContentProviderRecord cpr, boolean always) {
        for (int i = cpr.connections.size() - 1; i >= 0; i--) {
            ContentProviderConnection conn = cpr.connections.get(i);
            if (conn.waiting) {
                // If this connection is waiting for the provider, then we don't
                // need to mess with its process unless we are always removing
                // or for some reason the provider is not currently launching.
                if (inLaunching && !always) {
                    continue;
                }
            }
            //Got the information of the Client Process of this ContentProvider!!!
            ProcessRecord capp = conn.client;
            conn.dead = true;
            //This is an important checking, stableCount must large than 0.
            if (conn.stableCount > 0) {
                if (!capp.persistent && capp.thread != null
                        && capp.pid != 0
                        && capp.pid != MY_PID) {

                    //This is exactly where the Client Process is killed!!!
                    capp.kill("depends on provider "
                            + cpr.name.flattenToShortString()
                            + " in dying proc " + (proc != null ? proc.processName : "??")
                            + " (adj " + (proc != null ? proc.setAdj : "??") + ")", true);
                }
            } else if (capp.thread != null && conn.provider.provider != null) {
            }
        }
    }
```

By digging into the call stack: handleAppDiedLocked() -> cleanUpApplicationRecordLocked() -> cleanupAppInLaunchingProvidersLocked() -> removeDyingProviderLocked(). Finally, we find where the client process is killed by the ActivityManagerService. The killing reason in the code is the same as the log shown before. However, there are numbers of checking before it's killed. The most important one is `stableCount` must large than 0. What makes the `stableCount` increase or decrease?

## Modification of stableCount

stableCount++ in the `incProviderCountLocked()` method of ActivityManagerService. The call stack is: AMS.getContentProvide() -> AMS.getContentProviderImpl() -> AMS.incProviderCountLocked().

```java
    ContentProviderConnection incProviderCountLocked(ProcessRecord r,
            final ContentProviderRecord cpr, IBinder externalProcessToken, boolean stable) {
        if (r != null) {
            for (int i=0; i<r.conProviders.size(); i++) {
                ContentProviderConnection conn = r.conProviders.get(i);
                if (conn.provider == cpr) {
                    if (stable) {
                        //The stableCount is increased here!!!
                        conn.stableCount++;
                        conn.numStableIncs++;
                    }
                }
            }
            ContentProviderConnection conn = new ContentProviderConnection(cpr, r);
            if (stable) {
                //If there is no target ContentProvider found in conProviders, then create a new instance. And initialize the stableCount to 1.
                conn.stableCount = 1;
                conn.numStableIncs = 1;
            }
            cpr.connections.add(conn);
            r.conProviders.add(conn);
        }
    }

    private ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
            String name, IBinder token, boolean stable, int userId) {
        synchronized(this) {
            boolean providerRunning = cpr != null && cpr.proc != null && !cpr.proc.killed;
            if (providerRunning) {
                conn = incProviderCountLocked(r, cpr, token, stable);
            }

            if (!providerRunning) {
                conn = incProviderCountLocked(r, cpr, token, stable);
            }
            checkTime(startTime, "getContentProviderImpl: done!");
        }
    }

    @Override
    public final ContentProviderHolder getContentProvider(
            IApplicationThread caller, String name, int userId, boolean stable) {
        return getContentProviderImpl(caller, name, null, stable, userId);
    }
```

ActivityManagerService.getContentProvider() method is invoked in the `ActivityThread`.

```java
    public final IContentProvider acquireProvider(
            Context c, String auth, int userId, boolean stable) {
        try {
            holder = ActivityManagerNative.getDefault().getContentProvider(
                    getApplicationThread(), auth, userId, stable);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
        return holder.provider;
    }
```

ActivityThread.acquireProvider() method is invoked in the`ContextImpl`.

```java
    private static final class ApplicationContentResolver extends ContentResolver {
        private final ActivityThread mMainThread;
        private final UserHandle mUser;

        public ApplicationContentResolver(
                Context context, ActivityThread mainThread, UserHandle user) {
            super(context);
            mMainThread = Preconditions.checkNotNull(mainThread);
            mUser = Preconditions.checkNotNull(user);
        }

        @Override
        protected IContentProvider acquireProvider(Context context, String auth) {
            return mMainThread.acquireProvider(context,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), true);
        }
    }

    private ContextImpl(ContextImpl container, ActivityThread mainThread,
            LoadedApk packageInfo, IBinder activityToken, UserHandle user, int flags,
            Display display, Configuration overrideConfiguration, int createDisplayWithId) {
        //Create a new instance in the constructor.
        mContentResolver = new ApplicationContentResolver(this, mainThread, user);
    }

    @Override
    public ContentResolver getContentResolver() {
        return mContentResolver;
    }
```

`ContextImpl` is the implementation class of the `Context`, which is quite familiar to the Android App Developers. `mContentResolver` is initialized in the constructor of `ContextImpl`. Whenever the `getContentResolver()` method is called, it returns the `mContentResolver` to the caller.
`mContentResolver` is a class of `ApplicationContentResolver`, which implements the interface of `ContentResolver`. The implementation of `acquireProvider()` method directly invokes the `ActivityThread.acquireProvider()` methods.

```java
    public final IContentProvider acquireProvider(Uri uri) {
        if (!SCHEME_CONTENT.equals(uri.getScheme())) {
            return null;
        }
        final String auth = uri.getAuthority();
        if (auth != null) {
            // calls the abstract method, which is implemented in the ApplicationContentResolver
            return acquireProvider(mContext, auth);
        }
        return null;
    }
```

`acquireProvider()` and `releaseProvider()` is a pair which are called in the method of `insert()`, `delete()` and `query()` in ContentResolver.

```java
    public final @Nullable Uri insert(@RequiresPermission.Write @NonNull Uri url,
                @Nullable ContentValues values) {
        IContentProvider provider = acquireProvider(url);
        try {
        } catch (RemoteException e) {
        } finally {
            releaseProvider(provider);
        }
    }

    public final int delete(@RequiresPermission.Write @NonNull Uri url, @Nullable String where,
            @Nullable String[] selectionArgs) {
        IContentProvider provider = acquireProvider(url);
        try {
        } catch (RemoteException e) {
        } finally {
            releaseProvider(provider);
        }
    }
    
    public final @Nullable Cursor query(final @RequiresPermission.Read @NonNull Uri uri,
            @Nullable String[] projection, @Nullable String selection,
            @Nullable String[] selectionArgs, @Nullable String sortOrder,
            @Nullable CancellationSignal cancellationSignal) {
        try {
            try {
                qCursor = unstableProvider.query(mPackageName, uri, projection,
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);
            } catch (DeadObjectException e) {
                stableProvider = acquireProvider(uri);
            }
        } catch (RemoteException e) {
            // Arbitrary and not worth documenting, as Activity
            // Manager will kill this process shortly anyway.
            return null;
        } finally {
            if (stableProvider != null) {
                releaseProvider(stableProvider);
            }
        }
    }
```

## Decrease stableCount

As we know, `stableCount` increase  in `acquireProvider()`. Possibly, `stableCount` may decrease in `releaseProvider()`. Talk is cheap, show you the code. ApplicationContentResolver.releaseProvider() directly call ActivityThread.releaseProvider().

```java
    private static final class ApplicationContentResolver extends ContentResolver {
        @Override
        public boolean releaseProvider(IContentProvider provider) {
            return mMainThread.releaseProvider(provider, true);
        }
    }
```

`stableCount` decreases in ActivityThread.releaseProvider() as expected.

```java
    public final boolean releaseProvider(IContentProvider provider, boolean stable) {
        synchronized (mProviderMap) {
            if (stable) {
                prc.stableCount -= 1;
            }
        }
    }
```

Therefore, we figure it out how the ActivityManagerService kill the client process when using ContentProvider:

1. The client process gets the ContentResolver from Context;
2. When using the ContentResolver to query, insert or delete, before finished, the `stableCount` is large than 0;
3. If the server process dies at time, the ActivityManagerService clear the ContentProvider of server process;
4. For those ContentProviders that the `stableCount` is large than 0, then the client processes of those ContentProviders will be killed by the ActivityManagerService;

## Another killing Scenario

In the ActivityManagerService.getContentProviderImpl() method, if the server process of ContentProvider has not started yet, startProcessLocked() will be called to start it. Then incProviderCountLocked() method is called to increase the `stableCount`. Once the server process has started, ActivityManagerService.attachApplicationLocked() method is called. We should pay attention to 2 points here:

* send a 10-seconds delay message `CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG` to the `mHandler`;
* ActivityThread.bindApplication() is called;

```java
    private ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
            String name, IBinder token, boolean stable, int userId) {
        synchronized(this) {
            boolean providerRunning = cpr != null && cpr.proc != null && !cpr.proc.killed;
            if (!providerRunning) {
                // If the provider is not already being launched, then get it
                // started.
                if (i >= N) {
                    try {
                        ProcessRecord proc = getProcessRecordLocked(
                                cpi.processName, cpr.appInfo.uid, false);
                        if (proc != null && proc.thread != null && !proc.killed) {
                        } else {
                            proc = startProcessLocked(cpi.processName,
                                    cpr.appInfo, false, 0, "content provider",
                                    new ComponentName(cpi.applicationInfo.packageName,
                                            cpi.name), false, false, false);
                        }
                    }
                }
                //increase stableCount
                conn = incProviderCountLocked(r, cpr, token, stable);
            }
        }
    }

    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {
        if (providers != null && checkAppInLaunchingProvidersLocked(app)) {
            Message msg = mHandler.obtainMessage(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG);
            msg.obj = app;
            mHandler.sendMessageDelayed(msg, CONTENT_PROVIDER_PUBLISH_TIMEOUT);
        }
        try {
            ProfilerInfo profilerInfo = profileFile == null ? null
                    : new ProfilerInfo(profileFile, profileFd, samplingInterval, profileAutoStop);
            thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                    profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                    app.instrumentationUiAutomationConnection, testMode,
                    mBinderTransactionTrackingEnabled, enableTrackAllocation,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(mConfiguration), app.compat,
                    getCommonServicesLocked(app.isolated),
                    mCoreSettingsObserver.getCoreSettingsLocked());
        }
    }
```

To handle `CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG`, the `mHandler` calls processContentProviderPublishTimedOutLocked() -> cleanupAppInLaunchingProvidersLocked(), which kills the client process if `stableCount` is large than 0. How dangerous this message is!

```java
    final class MainHandler extends Handler {
        public MainHandler(Looper looper) {
            super(looper, null, true);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
            case CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG: {
                ProcessRecord app = (ProcessRecord)msg.obj;
                synchronized (ActivityManagerService.this) {
                    processContentProviderPublishTimedOutLocked(app);
                }
            } break;
            }
        }
    }

    private final void processContentProviderPublishTimedOutLocked(ProcessRecord app) {
        cleanupAppInLaunchingProvidersLocked(app, true);
        removeProcessLocked(app, false, true, "timeout publishing content providers");
    }
```

It only has 10 seconds to defuse the bomb. Let's see how ActivityThread.bindApplication() do the job. It sends a `BIND_APPLICATION` message to the handler.

```java
    private class ApplicationThread extends ApplicationThreadNative {
        public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableBinderTracking, boolean trackAllocation,
                boolean isRestrictedBackupMode, boolean persistent, Configuration config,
                CompatibilityInfo compatInfo, Map<String, IBinder> services, Bundle coreSettings) {
            sendMessage(H.BIND_APPLICATION, data);
        }
    }
```

To handle the `BIND_APPLICATION` message, handleBindApplication() method is called.

```java
                case BIND_APPLICATION:
                    handleBindApplication(data);
                    break;
```

The implementation of handleBindApplication() show us that it calls the installContentProviders(), in which we should focus on 2 points:

* installProvider() calls the ContentProvider.attachInfo();
* installContentProviders() calls ActivityManagerService.publishContentProviders();

```java
    private void handleBindApplication(AppBindData data) {
        try {
            if (!data.restrictedBackupMode) {
                if (!ArrayUtils.isEmpty(data.providers)) {
                    installContentProviders(app, data.providers);
                }
            }
        }
    }

    private void installContentProviders(
            Context context, List<ProviderInfo> providers) {
                final ArrayList<IActivityManager.ContentProviderHolder> results =
            new ArrayList<IActivityManager.ContentProviderHolder>();

        for (ProviderInfo cpi : providers) {
            IActivityManager.ContentProviderHolder cph = installProvider(context, null, cpi,
                    false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
            if (cph != null) {
                cph.noReleaseNeeded = true;
                results.add(cph);
            }
        }
        
        try {
            ActivityManagerNative.getDefault().publishContentProviders(
                getApplicationThread(), results);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }

    private IActivityManager.ContentProviderHolder installProvider(Context context,
            IActivityManager.ContentProviderHolder holder, ProviderInfo info,
            boolean noisy, boolean noReleaseNeeded, boolean stable) {
        ContentProvider localProvider = null;
        IContentProvider provider;
        if (holder == null || holder.provider == null) {
            try {
                //attachInfo calls the ContentProvider.onCreate() method
                localProvider.attachInfo(c, info);
            }
        }
    }
```

ContentProvider.attachInfo() calls the ContentProvider.onCreate(), which is an abstract method and must be implemented by the subclass of ContentProvider.

```java
    public void attachInfo(Context context, ProviderInfo info) {
        attachInfo(context, info, false);
    }

    private void attachInfo(Context context, ProviderInfo info, boolean testing) {
        mNoPerms = testing;

        /*
         * Only allow it to be set once, so after the content service gives
         * this to us clients can't change it.
         */
        if (mContext == null) {
            ContentProvider.this.onCreate();
        }
    }

    public abstract boolean onCreate();
```

In the ActivityManagerService.publishContentProviders() method, the bomb is defused finally. The `CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG` message is removed.

```java
    public final void publishContentProviders(IApplicationThread caller,
            List<ContentProviderHolder> providers) {
        synchronized (this) {
            final int N = providers.size();
            for (int i = 0; i < N; i++) {
                if (dst != null) {
                    if (wasInLaunchingProviders) {
                    mHandler.removeMessages(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG, r);
                    }
                }
            }
        }
    }
```

In conclusion, ActivityManagerService only gives the ContentProvider 10 seconds to start the server process, including the ContentProvider.onCreate(). Otherwise, the client process will be killed. So remember not to do time consuming work in the ContentProvider.onCreate().

#### Think twice before decide to use ContentProvider.

## References

[理解ContentProvider原理 Posted by Gityuan on July 30, 2016](http://gityuan.com/2016/07/30/content-provider/)

[ContentProvider引用计数 Posted by Gityuan on May 3, 2016](http://gityuan.com/2016/05/03/content_provider_release/)