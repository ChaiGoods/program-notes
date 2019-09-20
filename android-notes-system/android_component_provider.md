# Android Provider 实现分析

## 前言

ContentProvider 是 Android 系统的四大组件之一，它提供了一种共享数据的机制，同时也是一种进程间通信的方式，下面基于 Android 6.0.1 系统源码分析 Provider 组件的工作及实现原理，分为两部分，Provider 的查询和 Provider 的安装。

## 使用方式

一般的开发工作很少用到 ContentProvider，下面说一下它的用法。

### 实现数据提供者

首先需要实现自定义的数据提供者，需要实现增删查改的数据操作方法，还可实现自定义操作的 `call` 方法：

```java
public class ExampleContentProvider extends ContentProvider {

  @Override public boolean onCreate() {
    return false;
  }

  @Override
  public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) { return null; }

  @Override public String getType(Uri uri) { return null; }

  @Override public Uri insert(Uri uri, ContentValues values) { return null; }

  @Override
  public int delete(Uri uri, String selection, String[] selectionArgs) { return 0; }

  @Override
  public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs) { return 0; }
}
```

### 注册数据提供者

实现数据提供者后就可以在清单文件中注册了，注册时需要指定 Uri，当其他进程需要访问时，使用 Uri 即可获得 ContentProvider 的远程代理，然后使用代理即可向 ContentProvider 发送数据操作请求。

```xml
<application>
	<provider
        android:name="ExampleContentProvider"
        android:authorities="l0neman.provider.example" />
</application>
```

### 访问数据提供者

上面的工作完成后就可以向数据提供者发送请求了，使用 `content://` 前缀加 Uri 即可，还可在后面加数据访问路径：

```java
Bundle result = contenxt.getContentResolver().call(
  Uri.parse("content://l0neman.provider.example/path"), "method", null, null);
```

下面从对 ContentProvider 的 call 调用开始分析 Provider 的实现。

## Provider 的查询

首先从获得 ContentProvider 开始分析，即 `content.getCOntentResolver()`。

### Client

#### ContextImpl

```java
private final ApplicationContentResolver mContentResolver;
...

public ContextImpl() {
    ...
    mContentResolver = new ApplicationContentResolver(this, mainThread, user);
}

@Override
public ContentResolver getContentResolver() {
    return mContentResolver;
}
```

它由 `ApplicationContentProvider` 类实现。

```java
private static final class ApplicationContentResolver extends ContentResolver {
  ...
}
```

call 方法的实现在父类 `ContentProvider` 中。

#### ContentProvider

```java
// ContentProvider.java

public final @Nullable Bundle call(@NonNull Uri uri, @NonNull String method,
        @Nullable String arg, @Nullable Bundle extras) {
    Preconditions.checkNotNull(uri, "uri");
    Preconditions.checkNotNull(method, "method");
    // 首先获得 provider 引用。
    IContentProvider provider = acquireProvider(uri);
    if (provider == null) {
        throw new IllegalArgumentException("Unknown URI " + uri);
    }
    try {
        // 发起 call 请求。
        return provider.call(mPackageName, method, arg, extras);
    } catch (RemoteException e) {
        // 随意且不值得记录，因为活动管理器无论如何都会很快终止这个过程。
        return null;
    } finally {
        releaseProvider(provider);
    }
}
```

上面首先使用 `acquireProvider` 方法获取 provider 句柄，然后再调用 call，`acquireProvider` 是一个抽象方法，实现者是 `ApplicationContentResolver`。

#### ApplicationContentResolver

```java
// ApplicationContentResolver.java

@Override
protected IContentProvider acquireProvider(Context context, String auth) {
    return mMainThread.acquireProvider(context,
            ContentProvider.getAuthorityWithoutUserId(auth),
            resolveUserIdFromAuthority(auth), true);
}
```

`mMainThread` 为 `ActivityThread`，它在构造器中被传入。

#### ActivityThread

```java
// ActivityThread.java

public final IContentProvider acquireProvider(
        Context c, String auth, int userId, boolean stable) {
    // 获得已存在的 provider。
    final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
    if (provider != null) {
        return provider;
    }

    // 这里可能发生竞争。另一个线程可能会尝试获取同一个 provider。
    // 当这种情况发生时，我们希望确保第一个获胜。
    // 请注意，我们无法在获取和安装提供程序时保持锁定，因为它可能需要很
    // 长时间才能运行，并且在提供程序处于同一进程的情况下也可能是重入的。
    IActivityManager.ContentProviderHolder holder = null;
    try {
        holder = ActivityManagerNative.getDefault().getContentProvider(
                getApplicationThread(), auth, userId, stable);
    } catch (RemoteException ex) {
    }
    if (holder == null) {
        Slog.e(TAG, "Failed to find provider info for " + auth);
        return null;
    }

    // 安装 provider 会增加我们的引用计数，并打破竞赛中的任何关系。
    holder = installProvider(c, holder, holder.info,
            true /*noisy*/, holder.noReleaseNeeded, stable);
    return holder.provider;
}
```

上面首先获取已存在的 provider，如果有则直接返回，否则向 AMS 请求获取 ContentProvider。

```java
// ActivityThread.java

public final IContentProvider acquireExistingProvider(
        Context c, String auth, int userId, boolean stable) {
    synchronized (mProviderMap) {
        final ProviderKey key = new ProviderKey(auth, userId);
        // 查询 provider 客户端记录。
        final ProviderClientRecord pr = mProviderMap.get(key);
        if (pr == null) {
            return null;
        }

        IContentProvider provider = pr.mProvider;
        IBinder jBinder = provider.asBinder();
        if (!jBinder.isBinderAlive()) {
            // provider 的宿主进程已经死亡；我们不能使用这个。
            Log.i(TAG, "Acquiring provider " + auth + " for user " + userId
                    + ": existing object's process dead");
            handleUnstableProviderDiedLocked(jBinder, true);
            return null;
        }

        // 如果我们只有一个，那么只增加引用计数。如果我们不这样做，
        // 那么 provider 不会被引用计数，也不需要被释放。
        ProviderRefCount prc = mProviderRefCountMap.get(jBinder);
        if (prc != null) {
            // 增加引用计数。
            incProviderRefLocked(prc, stable);
        }
        return provider;
    }
}
```

这里从客户端缓存的 `mProviderMap` 中查询已存在的 provider，回到上面。

下面将通过进程间通信向 AMS 发出 `getContentProvider` 请求。

#### ActivityManagerProxy

```java
// ActivityManagerProxy.java

public ContentProviderHolder getContentProvider(IApplicationThread caller,
        String name, int userId, boolean stable) throws RemoteException {
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(caller != null ? caller.asBinder() : null);
    data.writeString(name);
    data.writeInt(userId);
    data.writeInt(stable ? 1 : 0);
    mRemote.transact(GET_CONTENT_PROVIDER_TRANSACTION, data, reply, 0);
    reply.readException();
    int res = reply.readInt();
    ContentProviderHolder cph = null;
    if (res != 0) {
        cph = ContentProviderHolder.CREATOR.createFromParcel(reply);
    }
    data.recycle();
    reply.recycle();
    return cph;
}
```

####ActivityManagerNative

```java
// ActivityManagerNative.java
@Override
public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
        throws RemoteException {
    switch (code) {
    ...
    case GET_CONTENT_PROVIDER_TRANSACTION: {
        data.enforceInterface(IActivityManager.descriptor);
        IBinder b = data.readStrongBinder();
        IApplicationThread app = ApplicationThreadNative.asInterface(b);
        String name = data.readString();
        int userId = data.readInt();
        boolean stable = data.readInt() != 0;
        ContentProviderHolder cph = getContentProvider(app, name, userId, stable);
        reply.writeNoException();
        if (cph != null) {
            reply.writeInt(1);
            cph.writeToParcel(reply, 0);
        } else {
            reply.writeInt(0);
        }
        return true;
    }
    ...
}
```

#### ActivityManagerService

```java
// ActivityManagerService.java

@Override
public final ContentProviderHolder getContentProvider(
        IApplicationThread caller, String name, int userId, boolean stable) {
    enforceNotIsolatedCaller("getContentProvider");
    if (caller == null) {
        String msg = "null IApplicationThread when getting content provider "
                + name;
        Slog.w(TAG, msg);
        throw new SecurityException(msg);
    }
    // 现在，在 checkContentProviderPermissionLocked（）中检查传入的用户以处理跨用户授权。
    return getContentProviderImpl(caller, name, null, stable, userId);
}
```

```java
// ActivityManagerService.java

private final ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
        String name, IBinder token, boolean stable, int userId) {
    ContentProviderRecord cpr;
    ContentProviderConnection conn = null;
    ProviderInfo cpi = null;

    synchronized(this) {
        long startTime = SystemClock.elapsedRealtime();

        ProcessRecord r = null;
        if (caller != null) {
            r = getRecordForAppLocked(caller);
            if (r == null) {
                throw new SecurityException(
                        "Unable to find app for caller " + caller
                      + " (pid=" + Binder.getCallingPid()
                      + ") when getting content provider " + name);
            }
        }

        boolean checkCrossUser = true;

        checkTime(startTime, "getContentProviderImpl: getProviderByName");

        // 首先检查这个 provider 是否已发布……
        cpr = mProviderMap.getProviderByName(name, userId);
        // 如果不工作，请检查用户 0 是否存在，然会在使用它之前验证
        // 它是否为单例 provider。
        if (cpr == null && userId != UserHandle.USER_OWNER) {
            cpr = mProviderMap.getProviderByName(name, UserHandle.USER_OWNER);
            if (cpr != null) {
                cpi = cpr.info;
                if (isSingleton(cpi.processName, cpi.applicationInfo,
                        cpi.name, cpi.flags)
                        && isValidSingletonCall(r.uid, cpi.applicationInfo.uid)) {
                    userId = UserHandle.USER_OWNER;
                    checkCrossUser = false;
                } else {
                    cpr = null;
                    cpi = null;
                }
            }
        }

        boolean providerRunning = cpr != null;
        if (providerRunning) {
            cpi = cpr.info;
            String msg;
            checkTime(startTime, "getContentProviderImpl: before checkContentProviderPermission");
            if ((msg = checkContentProviderPermissionLocked(cpi, r, userId, checkCrossUser))
                    != null) {
                throw new SecurityException(msg);
            }
            checkTime(startTime, "getContentProviderImpl: after checkContentProviderPermission");

            if (r != null && cpr.canRunHere(r)) {
                // 此 provider 已发布或正在发布……但它也运行在调用者的进程
                // 中运行，因此不要建立连接，只是让调用者实例化自己的实例。
                ContentProviderHolder holder = cpr.newHolder(null);
                // 不给调用者 provider 的对象，它需要自己创建。
                holder.provider = null;
                return holder;
            }

            final long origId = Binder.clearCallingIdentity();

            checkTime(startTime, "getContentProviderImpl: incProviderCountLocked");

            // 在这种情况下， provider 实例已经存在,
            // 所以我们可以立即返回。
            conn = incProviderCountLocked(r, cpr, token, stable);
            if (conn != null && (conn.stableCount+conn.unstableCount) == 1) {
                if (cpr.proc != null && r.setAdj <= ProcessList.PERCEPTIBLE_APP_ADJ) {
                    // 如果这是一个访问 provider 的可以被感知的 app，请确保
                    // 将其视为可被访问的，从而可备份到 LRU 列表。
                    // 这样会很好，因为 provider 通常开销比较大。
                    checkTime(startTime, "getContentProviderImpl: before updateLruProcess");
                    updateLruProcessLocked(cpr.proc, false, null);
                    checkTime(startTime, "getContentProviderImpl: after updateLruProcess");
                }
            }

            if (cpr.proc != null) {
                if (false) {
                    if (cpr.name.flattenToShortString().equals(
                            "com.android.providers.calendar/.CalendarProvider2")) {
                        Slog.v(TAG, "****************** KILLING "
                            + cpr.name.flattenToShortString());
                        Process.killProcess(cpr.proc.pid);
                    }
                }
                checkTime(startTime, "getContentProviderImpl: before updateOomAdj");
                boolean success = updateOomAdjLocked(cpr.proc);
                maybeUpdateProviderUsageStatsLocked(r, cpr.info.packageName, name);
                checkTime(startTime, "getContentProviderImpl: after updateOomAdj");
                if (DEBUG_PROVIDER) Slog.i(TAG_PROVIDER, "Adjust success: " + success);
                // 注意：即使我们设法更新其 adj 等级，仍然存在一个可能
                // 在该过程中等待裁决的竞争。 
                // 不知道该怎么做，但至少现在竞争变小了。
                if (!success) {
                    // 哦……看起来 provider 所在进程已经被我们杀死了。
                    // 我们需要等待一个新进程的启动，并确保它的死亡不会杀死我们的进程。
                    Slog.i(TAG, "Existing provider " + cpr.name.flattenToShortString()
                            + " is crashing; detaching " + r);
                    boolean lastRef = decProviderCountLocked(conn, cpr, token, stable);
                    checkTime(startTime, "getContentProviderImpl: before appDied");
                    appDiedLocked(cpr.proc);
                    checkTime(startTime, "getContentProviderImpl: after appDied");
                    if (!lastRef) {
                        // 这不是我们的进程对提供商的最后一个引用……
                        // 我们现在已被杀死，释放。
                        return null;
                    }
                    providerRunning = false;
                    conn = null;
                }
            }

            Binder.restoreCallingIdentity(origId);
        }

        boolean singleton;
        if (!providerRunning) {
            try {
                checkTime(startTime, "getContentProviderImpl: before resolveContentProvider");
                cpi = AppGlobals.getPackageManager().
                    resolveContentProvider(name,
                        STOCK_PM_FLAGS | PackageManager.GET_URI_PERMISSION_PATTERNS, userId);
                checkTime(startTime, "getContentProviderImpl: after resolveContentProvider");
            } catch (RemoteException ex) {
            }
            if (cpi == null) {
                return null;
            }
            // 如果提供者是单例 AND（它是被同一用户调用，那么 provider 是特权 app）
            // 然后允许连接到单例 provider。
            singleton = isSingleton(cpi.processName, cpi.applicationInfo,
                    cpi.name, cpi.flags)
                    && isValidSingletonCall(r.uid, cpi.applicationInfo.uid);
            if (singleton) {
                userId = UserHandle.USER_OWNER;
            }
            cpi.applicationInfo = getAppInfoForUser(cpi.applicationInfo, userId);
            checkTime(startTime, "getContentProviderImpl: got app info for user");

            String msg;
            checkTime(startTime, "getContentProviderImpl: before checkContentProviderPermission");
            if ((msg = checkContentProviderPermissionLocked(cpi, r, userId, !singleton))
                    != null) {
                throw new SecurityException(msg);
            }
            checkTime(startTime, "getContentProviderImpl: after checkContentProviderPermission");

            if (!mProcessesReady && !mDidUpdate && !mWaitingUpdate
                    && !cpi.processName.equals("system")) {
                // If this content provider does not run in the system
                // process, and the system is not yet ready to run other
                // processes, then fail fast instead of hanging.
                throw new IllegalArgumentException(
                        "Attempt to launch content provider before system ready");
            }

            // Make sure that the user who owns this provider is running.  If not,
            // we don't want to allow it to run.
            if (!isUserRunningLocked(userId, false)) {
                Slog.w(TAG, "Unable to launch app "
                        + cpi.applicationInfo.packageName + "/"
                        + cpi.applicationInfo.uid + " for provider "
                        + name + ": user " + userId + " is stopped");
                return null;
            }

            ComponentName comp = new ComponentName(cpi.packageName, cpi.name);
            checkTime(startTime, "getContentProviderImpl: before getProviderByClass");
            cpr = mProviderMap.getProviderByClass(comp, userId);
            checkTime(startTime, "getContentProviderImpl: after getProviderByClass");
            final boolean firstClass = cpr == null;
            if (firstClass) {
                final long ident = Binder.clearCallingIdentity();
                try {
                    checkTime(startTime, "getContentProviderImpl: before getApplicationInfo");
                    ApplicationInfo ai =
                        AppGlobals.getPackageManager().
                            getApplicationInfo(
                                    cpi.applicationInfo.packageName,
                                    STOCK_PM_FLAGS, userId);
                    checkTime(startTime, "getContentProviderImpl: after getApplicationInfo");
                    if (ai == null) {
                        Slog.w(TAG, "No package info for content provider "
                                + cpi.name);
                        return null;
                    }
                    ai = getAppInfoForUser(ai, userId);
                    cpr = new ContentProviderRecord(this, cpi, ai, comp, singleton);
                } catch (RemoteException ex) {
                    // pm is in same process, this will never happen.
                } finally {
                    Binder.restoreCallingIdentity(ident);
                }
            }

            checkTime(startTime, "getContentProviderImpl: now have ContentProviderRecord");

            if (r != null && cpr.canRunHere(r)) {
                // If this is a multiprocess provider, then just return its
                // info and allow the caller to instantiate it.  Only do
                // this if the provider is the same user as the caller's
                // process, or can run as root (so can be in any process).
                return cpr.newHolder(null);
            }

            if (DEBUG_PROVIDER) Slog.w(TAG_PROVIDER, "LAUNCHING REMOTE PROVIDER (myuid "
                        + (r != null ? r.uid : null) + " pruid " + cpr.appInfo.uid + "): "
                        + cpr.info.name + " callers=" + Debug.getCallers(6));

            // This is single process, and our app is now connecting to it.
            // See if we are already in the process of launching this
            // provider.
            final int N = mLaunchingProviders.size();
            int i;
            for (i = 0; i < N; i++) {
                if (mLaunchingProviders.get(i) == cpr) {
                    break;
                }
            }

            // If the provider is not already being launched, then get it
            // started.
            if (i >= N) {
                final long origId = Binder.clearCallingIdentity();

                try {
                    // Content provider is now in use, its package can't be stopped.
                    try {
                        checkTime(startTime, "getContentProviderImpl: before set stopped state");
                        AppGlobals.getPackageManager().setPackageStoppedState(
                                cpr.appInfo.packageName, false, userId);
                        checkTime(startTime, "getContentProviderImpl: after set stopped state");
                    } catch (RemoteException e) {
                    } catch (IllegalArgumentException e) {
                        Slog.w(TAG, "Failed trying to unstop package "
                                + cpr.appInfo.packageName + ": " + e);
                    }

                    // Use existing process if already started
                    checkTime(startTime, "getContentProviderImpl: looking for process record");
                    ProcessRecord proc = getProcessRecordLocked(
                            cpi.processName, cpr.appInfo.uid, false);
                    if (proc != null && proc.thread != null) {
                        if (DEBUG_PROVIDER) Slog.d(TAG_PROVIDER,
                                "Installing in existing process " + proc);
                        if (!proc.pubProviders.containsKey(cpi.name)) {
                            checkTime(startTime, "getContentProviderImpl: scheduling install");
                            proc.pubProviders.put(cpi.name, cpr);
                            try {
                                proc.thread.scheduleInstallProvider(cpi);
                            } catch (RemoteException e) {
                            }
                        }
                    } else {
                        checkTime(startTime, "getContentProviderImpl: before start process");
                        proc = startProcessLocked(cpi.processName,
                                cpr.appInfo, false, 0, "content provider",
                                new ComponentName(cpi.applicationInfo.packageName,
                                        cpi.name), false, false, false);
                        checkTime(startTime, "getContentProviderImpl: after start process");
                        if (proc == null) {
                            Slog.w(TAG, "Unable to launch app "
                                    + cpi.applicationInfo.packageName + "/"
                                    + cpi.applicationInfo.uid + " for provider "
                                    + name + ": process is bad");
                            return null;
                        }
                    }
                    cpr.launchingApp = proc;
                    mLaunchingProviders.add(cpr);
                } finally {
                    Binder.restoreCallingIdentity(origId);
                }
            }

            checkTime(startTime, "getContentProviderImpl: updating data structures");

            // Make sure the provider is published (the same provider class
            // may be published under multiple names).
            if (firstClass) {
                mProviderMap.putProviderByClass(comp, cpr);
            }

            mProviderMap.putProviderByName(name, cpr);
            conn = incProviderCountLocked(r, cpr, token, stable);
            if (conn != null) {
                conn.waiting = true;
            }
        }
        checkTime(startTime, "getContentProviderImpl: done!");
    }

    // Wait for the provider to be published...
    synchronized (cpr) {
        while (cpr.provider == null) {
            if (cpr.launchingApp == null) {
                Slog.w(TAG, "Unable to launch app "
                        + cpi.applicationInfo.packageName + "/"
                        + cpi.applicationInfo.uid + " for provider "
                        + name + ": launching app became null");
                EventLog.writeEvent(EventLogTags.AM_PROVIDER_LOST_PROCESS,
                        UserHandle.getUserId(cpi.applicationInfo.uid),
                        cpi.applicationInfo.packageName,
                        cpi.applicationInfo.uid, name);
                return null;
            }
            try {
                if (DEBUG_MU) Slog.v(TAG_MU,
                        "Waiting to start provider " + cpr
                        + " launchingApp=" + cpr.launchingApp);
                if (conn != null) {
                    conn.waiting = true;
                }
                cpr.wait();
            } catch (InterruptedException ex) {
            } finally {
                if (conn != null) {
                    conn.waiting = false;
                }
            }
        }
    }
    return cpr != null ? cpr.newHolder(conn) : null;
}
```



## Provider 的安装
