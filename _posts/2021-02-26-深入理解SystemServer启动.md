---
title: 深入理解SystemServer启动
author: DongXiaoFat
date: 2021-02-26 13:45:00 +0800
categories: [Android, Framework]
tags: [Framework]
excerpt: 深入理解SystemServer启动
math: true
mermaid: true
thumbnail: /assets/img/favicons/av.jpg
---
# 深入理解SystemServer启动

> SystemServer 进程是我们Android系统中重中之重的一个进程服务，Android中重要的Service几乎都是有它管理运行，从《深入理解Zygote启动》一文中，我们知道了，SystemServer是由Zygote进程进行fork出来，其Binder线程是由 ZygoteInit 进行创建，第一个函数入口是SystemServer.java 里的main 函数。

## 1. SystemServer 入口——main

### 1.1 SystemServer 对象构造
1. 首先通过查询SYSPROP_START_COUNT当前启动的次数+1.
2. 在systemserver被创建的时候首先会去通过SystemClock.elapsedRealtime()和SystemClock.uptimeMillis()计算系统启动到目前的时间。

3. SystemServer 对象创建的时候会去检查属性“sys.boot_completed”，如果该属性为1，说明系统已经是启动完成过的，这一次只是运行过程的重启，而不是断电重启。

```java
private static final String SYSPROP_START_COUNT = "sys.system_server.start_count";
public SystemServer() {
    // Check for factory test mode.
    mFactoryTestMode = FactoryTest.getMode();
    // Record process start information.
    // Note SYSPROP_START_COUNT will increment by *2* on a FDE device when it fully boots;
    // one for the password screen, second for the actual boot.
    mStartCount = SystemProperties.getInt(SYSPROP_START_COUNT, 0) + 1;
    mRuntimeStartElapsedTime = SystemClock.elapsedRealtime();
    mRuntimeStartUptime = SystemClock.uptimeMillis();
    // Remember if it's runtime restart(when sys.boot_completed is already set) or reboot
    // We don't use "mStartCount > 1" here because it'll be wrong on a FDE device.
    // TODO: mRuntimeRestart will *not* be set to true if the proccess crashes before
    // sys.boot_completed is set. Fix it.
    mRuntimeRestart = "1".equals(SystemProperties.get("sys.boot_completed"));
}
```

### 1.2 main

main函数里就一句代码，调用私有的run方法。
```java
    /**
     * The main entry point from zygote.
     */
    public static void main(String[] args) {
        new SystemServer().run();
    }
```

SystemServer里的run方法就很长了：
1. 首先会去将systemserver 对象初始化时，赋值的启动时间、启动次数给写回property里。

2. 通过Binder的setWarnOnBlocking(true)方法设置Binder的sWarnOnBlocking 标志位为true，意味着Binder调用应都是oneway接口，如果不是就会抛出警告——"Outgoing transactions from this process must be FLAG_ONEWAY"

3. 通过 VMRuntime.getRuntime().clearGrowthLimit()清除内存。

4. 通过Build.ensureFingerprintProperty()检查指纹属性，没有的话会帮其生产唯一指纹。

5. 通过BinderInternal.disableBackgroundScheduling(true)保证Binder call 总是在前台。

6. 通过BinderInternal.setMaxThreads(sMaxBinderThreads)设置binder最大线程数31.

```java
// maximum number of binder threads used for system_server
// will be higher than the system default
private static final int sMaxBinderThreads = 31;
```
7. 设置当前执行这个main方法的主线程为前台线程。

```java 
// Prepare the main looper thread (this thread).
android.os.Process.setThreadPriority(
        android.os.Process.THREAD_PRIORITY_FOREGROUND);
android.os.Process.setCanSelfBackground(false);
```

8. 准备主Loop

```java
Looper.prepareMainLooper();
Looper.getMainLooper().setSlowLogThresholdMs(
        SLOW_DISPATCH_THRESHOLD_MS, SLOW_DELIVERY_THRESHOLD_MS);
```

9. 加载库 System.loadLibrary("android_servers")

(System.load()必须绝对路径，loadLibrary是库文件名不含扩展名——.so啥的)

10. 初始化initZygoteChildHeapProfiling

11. 检查我们上次尝试是否无法关闭 performPendingShutdown

12. 通过ActivityThread创建SystemContext也即ContextImpl createSystemContext()

13. 初始化initializeMainlineModules，包含TelephonyServiceManager和StatsServiceManager。

14. 创建SystemServiceManager对象,并将对象保存至LocalServices中

```java
// Create the system service manager.
mSystemServiceManager = new SystemServiceManager(mSystemContext);
mSystemServiceManager.setStartInfo(mRuntimeRestart,
        mRuntimeStartElapsedTime, mRuntimeStartUptime);
LocalServices.addService(SystemServiceManager.class,mSystemServiceManager);
```

15. 为可以并行化的初始化任务创建线程池 SystemServerInitThreadPool.start()

16. 设置默认的WTF（what the fuck）问题处理者——SystemServer::handleEarlySystemWtf

17. startBootstrapServices.

18. startCoreServices

19. startOtherServices

20. Loop.Loop()

## SystemServer 之 loadLibrary

这里使用 loadLibrary 加载名为“android_servers” 的动态库，在Linux系统库的命名有个规范如下：

![](res/picture/libname)

所以要查找这个库的时候要是搜 "libandroid_servers",这个库里主要是包含了一些native服务如输入、电源、图像、蓝牙等等，

- 库所在路径 frameworks/base/services/Android.bp

```java
cc_library_shared {
name: “libandroid_servers”,
defaults: [“libservices.core-libs”],
whole_static_libs: [“libservices.core”],
}
```

System.load 与 System.loadLibrary 区别 System.load 需要指明绝对路径。

## 三大服务启动

### startBootstrapServices

顾名思义，启动引导服务，启动服务以下：

1. Installer
2. DeviceIdentifiersPolicyService
3. ActivityManagerService.Lifecycle(初始化 AMS)
4. RecoverySystemService
5. LightService
6. DisplayManagerService
7. PackageManagerService
8. UserManagerService
9. OverlayManagerService

通过 AMS 的 setSystemProcess() 方法将 AMS 实例添加到 ServiceManagerNative中。

### startCoreServices

- BatteryService
- UsageStatesService
- WebViewUpdateService
- BinderCallStatesService (Binder调用细节跟踪开关 persist.sys.binder_calls_detailed_tracking)

### startOtherServces

- VibratorService vibrator
- DynamicSystemService dynamicSystem
- IStorageManager storageManager
- NetworkManagementService networkManagement
- IpSecService ipSecService
- NetworkStatsService networkStats
- NetworkPolicyManagerService networkPolicy
- ConnectivityService connectivity
- NsdService serviceDiscovery
- WindowManagerService wm
- SerialService serial
- NetworkTimeUpdateService networkTimeUpdater
- InputManagerService inputManager
- TelephonyRegistry telephonyRegistry
- ConsumerIrService consumerIr
- MmsServiceBroker mmsService
- DropBoxManagerService
- HardwarePropertiesManagerService hardwarePropertiesService

创建的 wms 会传递给 AMS。
```java
 wm = WindowManagerService.main(context, inputManager, !mFirstBoot, mOnlyCore,
        new PhoneWindowManager(), mActivityManagerServimActivityTaskManager);
ServiceManager.addService(Context.WINDOW_SERVICE, wm, allowIsolated= */ false,
        DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PROTO);
ServiceManager.addService(Context.INPUT_SERVICE, inputManager,
        /* allowIsolated= */ falsDUMP_FLAG_PRIORITY_CRITICAL);
traceEnd(
traceBeginAndSlog("SetWindowManagerService");
mActivityManagerService.setWindowManager(wm);
traceEnd();
```

调用 WMS的 onInitReady()方法，
1. initPolicy()该方法会初始化显示策略初始化PhoneWindowManager。
2. openSurfaceTransaction() 初始化surfaceControl 全局的transaction对象。
3. createWatermarkInTransaction() 创建水印

wm.displayReady() 初始化DisplayContent 和 WindowAnimator

通知 WindowManagerService.systemReady()

updateConfiguration

PowerManagerService.systemReady()

PackageManagerService.systemReady()

ActivityManagerService.systemReady()

在 AMS 的 systemReady 中，首先会通过 startSystemUi 启动systemui 应用 com.android.systemui.SystemUIService. AMS 内部的 systemReady()还会去掉起它内部的服务的systemready 方法。它还会去启动persistent的APP，比较重要的是还会去启动所有display的Home应用。



