---
title: Android persistent机制
date: '2019/3/6 21:26:11'
abbrlink: 8417e24a
---

![](https://cdn.jsdelivr.net/gh/devnan/pic/blog/20200126213606.jpg)
<!-- more -->

本文简单分析persistent属性的相关源码流程，总结persistent的作用及注意事项。

# 前言
在一次调试系统应用过程中，修改部分代码逻辑后，执行`adb install -r` 并启动，发现应用界面更新了，但是修改到的逻辑并没有变，还是之前的版本逻辑。
分别执行了pm clear和am force-stop再起来应用，发现这两种做法进程id都没有变。
于是直接kill掉对应进程id，发现进程id变了，进程重启了，但是发现修改到的逻辑还是没变...
最后重启了一下机器？应用的界面和逻辑都正常更新了。

为什么呢？可以看一下此系统应用的manifest

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.seewo.trunning"
    android:sharedUserId="android.uid.system">
    
    ...
    
    <application
        android:name=".TRunningApplication"
        android:label="@string/app_name"
        android:persistent="true"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
    ...
    </application>
</manifest>
```
原来是persistent属性在作怪。persistent有什么作用？看看官网解释：
![](https://raw.githubusercontent.com/devnan/pic/master/20190507223758.png)

可以知道，persistent表示此应用是可持久的，也就是常驻应用。官网的解释就这么多，下面只能从源码入手来了解persistent的原理，并解释上面例子的原因。

# 开机启动过程

众所周知，带persistent属性的系统应用开机会自启动。并且persistent应用启动时机很早，早于Launcher启动及开机广播。可以看一下framework层是怎么做的？
android系统开机启动时，AMS会调用到systemReady()，然后扫描所有安装的应用，并启动属性persistent为true的应用。


`/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java`
```java
public void systemReady(final Runnable goingCallback, TimingsTraceLog traceLog) {

    ...
    
    synchronized (this) {
        startPersistentApps(PackageManager.MATCH_DIRECT_BOOT_AWARE);
        
        // Start up initial activity.
        mBooting = true;
        
        ...
        
        startHomeActivityLocked(currentUserId, "systemReady");  //0
          
        ...
    }
}

void startPersistentApps(int matchFlags) {
    if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL) return;

    synchronized (this) {
        try {
            final List<ApplicationInfo> apps = AppGlobals.getPackageManager()
                    .getPersistentApplications(STOCK_PM_FLAGS | matchFlags).getList(); //1
            for (ApplicationInfo app : apps) {
                if (!"android".equals(app.packageName)) {
                    addAppLocked(app, null, false, null /* ABI override */);
                }
            }
        } catch (RemoteException ex) {
        }
    }
}
```
上面注释0处，startHomeActivityLocked()会启动Launcher，晚于persistent应用的启动；
注释1处，通过PMS的getPersistentApplications()方法获取persistent应用, 最后再通过addAppLocked()启动应用进程

`/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java`

```java
final ProcessRecord addAppLocked(ApplicationInfo info, String customProcess, boolean isolated,
            boolean disableHiddenApiChecks, String abiOverride) {
   
    ...

    if ((info.flags & PERSISTENT_MASK) == PERSISTENT_MASK) {
        app.persistent = true;
        app.maxAdj = ProcessList.PERSISTENT_PROC_ADJ;  //2
    }
    if (app.thread == null && mPersistentStartingProcesses.indexOf(app) < 0) {
        mPersistentStartingProcesses.add(app);
        startProcessLocked(app, "added application",  //3
                customProcess != null ? customProcess : app.processName, disableHiddenApiChecks,
                abiOverride);  
    }

    ...
}

public final class ProcessList {
    ...
    
    // This is a system persistent process, such as telephony.  Definitely
    // don't want to kill it, but doing so is not completely fatal.
    static final int PERSISTENT_PROC_ADJ = -800;
    
    ...
}

```
上面主要做了两件事，注释2处设定进程adj为PERSISTENT_PROC_ADJ，值为-800，定义在ProcessList类中，表示进程属于高优先级，当系统资源紧张时，lowmemorykiller机制回收应用时会跳过persistent进程。
注释3处就是启动进程了，startProcessLocked方法最后会发消息到zygote进程，然后fork出应用进程，这里不再深入。
通过上面的步骤，android系统就在启动时预加载了persistent应用，并赋予了应用进程高优先级以保证持久性，但是，如果persistent应用被杀死或运行时异常崩溃，android如何对应用进行拉活？继续往下看。


# 重启机制

android在进程启动后调用到`attachApplicationLocked`时会创建一个进程死亡监听器`AppDeathRecipient`



`/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java`


```java
private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid, int callingUid, long startSeq) {
    ...
    
    try {
        AppDeathRecipient adr = new AppDeathRecipient(
                app, pid, thread);
        thread.asBinder().linkToDeath(adr, 0);
        app.deathRecipient = adr;
    }
    
    ...
}

private final class AppDeathRecipient implements IBinder.DeathRecipient {
    
    ...

    @Override
    public void binderDied() {
        if (DEBUG_ALL) Slog.v(
            TAG, "Death received in " + this
            + " for thread " + mAppThread.asBinder());
        synchronized(ActivityManagerService.this) {
            appDiedLocked(mApp, mPid, mAppThread, true); //4
        }
    }
}

```

当应用进程死亡时，系统会回调到死亡监听器的binderDied()方法，执行注释4处的appDiedLocked()做一些进程死亡后的相关善后工作。
继续走下去，appDiedLocked()最终会执行到cleanUpApplicationRecordLocked()，我们可以看一下这个方法做了什么。

```java
private final boolean cleanUpApplicationRecordLocked(ProcessRecord app,
        boolean restarting, boolean allowRestart, int index, boolean replacingPid) {
        
    ...
    
    if (!app.persistent || app.isolated) {
    
        ...
        
    } else if (!app.removed) {
        // This app is persistent, so we need to keep its record around.
        // If it is not already on the pending app list, add it there
        // and start a new process for it.
        if (mPersistentStartingProcesses.indexOf(app) < 0) {
            mPersistentStartingProcesses.add(app);
            restart = true;
        }
    }
    
    ...
}
```
上面的注释很清楚，如果是persistent应用，会继续保留其进程记录，然后重新启动进程。
当然，如果是persistent应用进程死亡，系统则直接回收所有的进程资源。
到这里，我们可以了解到，当persistent应用被异常杀死时，系统通过进程死亡监听器来重启进程，并且过程中不会清理进程记录ProcessRecord。而ProcessRecord对象其实记录着进程的四大组件和进程状态等信息。

## persistent带来的问题

我们可以看到，系统应用声明persistent，可以开机立即加载进程，更快响应；可以保证一些关键应用常驻在系统中，不会被系统回收；比如长连接应用等。但是persistent的使用同时也会带来一些问题，如长期占用系统资源不释放；如果app出现不可恢复的crash，将陷入一直崩溃启动的死循环；应用自升级不会kill并清理进程，需要特殊处理，并且升级后会失去persistent特性。

#### 系统应用自升级

正常的应用自升级流程，覆盖安装时系统会kill应用进程并清理在AMS中的进程记录，在新版应用重新启动时，系统会新建一个进程并重新加载各种组件运行。
但是，persistent应用自升级时，系统不会kill此应用进程，AMS也不会清理进程记录，系统只会把新版应用中的各种组件信息记录到AMS中，这样可能出现两个版本逻辑融合到一起，导致应用功能出现错乱。
解决方法有，一种是应用自升级后重启系统，这样会重新加载进程，但是这个明显不符合需求，影响用户体验，不可取。
第二种是接收新版应用的安装广播，调用context的startInstrumentation方法，会强制系统kill应用进程，清理AMS对各种组件的状态记录，并重新启动应用。

解决上面这个问题以后，还有一个问题，升级后的persistent系统应用会变成普通应用，导致后续失去persistent特性。

#### 低端设备禁用硬件加速

在低端设备上对于persistent进程会禁用硬件加速，下面代码注释已经说明了。android系统这么做，可能就是因为persistent应用是常驻的，这样可以避免占用太多的资源。

`/frameworks/base/core/java/android/view/ViewRootImpl.java`
```java
private void enableHardwareAcceleration(WindowManager.LayoutParams attrs) {
    if (hardwareAccelerated) {
        if (!ThreadedRenderer.isAvailable()) {
            return;
        }

        // Persistent processes (including the system) should not do
        // accelerated rendering on low-end devices.  In that case,
        // sRendererDisabled will be set.  In addition, the system process
        // itself should never do accelerated rendering.  In that case, both
        // sRendererDisabled and sSystemRendererDisabled are set.  When
        // sSystemRendererDisabled is set, PRIVATE_FLAG_FORCE_HARDWARE_ACCELERATED
        // can be used by code on the system process to escape that and enable
        // HW accelerated drawing.  (This is basically for the lock screen.)

        final boolean fakeHwAccelerated = (attrs.privateFlags &
                WindowManager.LayoutParams.PRIVATE_FLAG_FAKE_HARDWARE_ACCELERATED) != 0;
        final boolean forceHwAccelerated = (attrs.privateFlags &
                WindowManager.LayoutParams.PRIVATE_FLAG_FORCE_HARDWARE_ACCELERATED) != 0;
    }
 }
```

## 其他问题
回头看最开始的调试例子应该清楚了。说到应用界面更新了，但是修改到的逻辑并没有变，其实是因为刚好应用界面跑在子进程里，这也说明了persistent属性只能对应用主进程生效，子进程不生效。
persistent进程的adj值是负数，普通进程一般大于等于0，我们可以直接查看进程adj来验证一下
![](https://raw.githubusercontent.com/devnan/pic/master/20190507224132.png)

![](https://raw.githubusercontent.com/devnan/pic/master/20190507224146.png)

可以发现，其中home进程是子进程，adj为0，是前台进程。应用主进程pid为1091，adj为-11，其优先级很高，是persistent进程。

## 总结

1. 普通应用manifest中声明persistent属性是无效的，只有系统应用才能生效；
2. persistent应用启动早于Launcher启动早于开机广播，开发中需要注意时序问题；
3. persistent应用会一直常驻在系统中，进程资源不会被系统回收，应用异常退出后系统会立即重启应用，并保留进程记录；
4. persistent应用自升级需要做特殊处理。所以，开发中覆盖安装persistent应用后，必须重启android设备才能正常运行最新代码。当然，第一次覆盖安装后，该应用会失去persistent特性。
5. persistent应用的子进程不具备persistent特性；