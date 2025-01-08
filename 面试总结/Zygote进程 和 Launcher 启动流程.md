>来源：
>
>[Android 中的 Zygote 和 Copy-on-Write 机制详解](https://juejin.cn/post/7437141342075256867)
>
>[Android启动流程](https://www.cnblogs.com/anywherego/p/18221943)
>
>[第3章 Zygote进程相关](https://blog.csdn.net/u010687761/article/details/131404918)
>
>[安卓12homeActivity的启动流程--第一次启动的是fallbackhome](https://blog.csdn.net/m0_46857231/article/details/130636677)
>
>[【Android】系统启动流程分析 —— Zygote 进程启动过程](https://blog.csdn.net/cnwutianhao/article/details/137050676)

[TOC]

### 完整启动流程
![image-20250108141555473](https://raw.githubusercontent.com/mendax92/pic/main/20250108141555473.png)
1. 引导加载程序（Bootloader）启动： 当设备上电或者重启时，首先会由引导加载程序负责启动。引导加载程序通常存储在设备的固件中，它的主要任务是初始化硬件，并加载并启动操作系统内核。引导加载程序会首先运行自身的初始化代码，然后加载操作系统内核到内存中。
2. 内核加载： 引导加载程序会根据预定义的配置从设备存储中加载操作系统内核。在Android设备中，通常使用的是Linux内核。引导加载程序将内核加载到内存中的指定位置。
3. 内核初始化： 一旦内核加载到内存中，引导加载程序会将控制权转交给内核。内核开始执行初始化过程，包括对硬件进行初始化、建立虚拟文件系统、创建进程和线程等。
4. 启动 init 进程： 内核初始化完成后，会启动名为init的用户空间进程。init进程是Android系统的第一个用户空间进程，它负责系统的进一步初始化和启动。init进程会读取系统配置文件（例如 init.rc），并根据其中的指令启动系统服务和应用程序。
5. 启动 Zygote 进程： 在init进程启动后，会启动名为Zygote的进程。Zygote进程是Android应用程序的孵化器，它会预加载常用的Java类和资源，以加速应用程序的启动。
6. 启动系统服务： 在Zygote进程启动后，还会启动一系列系统服务，例如SurfaceFlinger、ActivityManager、PackageManager等。这些系统服务负责管理系统的各个方面，包括显示、应用程序生命周期、包管理等。
7. 启动桌面程序： 一旦系统服务启动完成，Android系统就处于可用状态。就会启动桌面程序，用户可以开始启动应用程序并使用设备进行各种操作了。
### Zygote进程(孵化器进程)
#### Zygote简介
- Zygote进程是一个用户进程，由init进程(1号进程)fork而来。
- Zygote进程的主要任务是加载系统的核心类库（如Java核心库和Android核心库）和安卓系统服务(SystemService)，然后进入一个循环，等待请求来创建新的 Android 应用程序进程。
- Zygote进程通过fork的方式创建新的应用程序进程，新进程会继承 Zygote 的所有资源，fork 会复制父进程的内存空间，但在 COW 机制下，系统不会立即复制整个内存，而是让子进程与父进程共享同一片内存区域。只有当子进程或父进程试图修改这片内存时，系统才会为修改方复制一份新的内存区域。这种方法显著减少了内存使用，并提高了进程启动效率。

#### Zygote启动流程

![image-20250108151639485](https://raw.githubusercontent.com/mendax92/pic/main/20250108151639485.png)

#### init进程解析init.rc脚本
Zygote由init进程解析init.rc脚本启动的。脚本传入app_process的main方法做分割，根据字符串命令做相应逻辑
现在机器分为32位和64位，Zygote的启动脚本init.rc也各有区别:

```
init.zygote32.rc： 
zygote进程对应的执行程序是app_process(纯32bit模式)  
init.zygote64.rc：
zygote进程对应的执行程序是app_process64(纯64bit模式)  
init.zygote32_64.rc ：
启动两个zygote进程，对应的执行程序分别是app_process32(主模式)、app_process64  
init.zygote64_32.rc：
启动两个zygote进程，对应的执行程序分别是app_process64(主模式)、app_process32
```

init.rc文件中包含启动Zygote的指令脚本的主要代码

```
// 创建zygote服务
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
class main
// 创建zygote socket，与系统和应用程序做通信
socket zygote stream 660 root system
// 定义了zygote服务重启时的一些操作
onrestart write /sys/android_power/request_state wake
onrestart write /sys/power/state on
onrestart restart media
onrestart restart netd
```
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server:

```
service：服务标识
zygote：表示要开启的服务名字
/system/bin/app_process64：服务对应的路径
-Xzygote：作为虚拟机启动时所需要的参数,在AndroidRuntime.cpp中的 startVm() 中调用JNI_CreateJavaVM 使用到
/system/bin：代表虚拟机程序所在目录,因为 app_process 可以不和虚拟机在一个目录,所以 app_process 需要知道虚拟机所在的目录
–zygote ：指明以 ZygoteInit 类作为入口,否则需要指定需要执行的类名
–start-system-server：仅在有 --zygote 参数时可用,告知 ZygoteInit 启动完毕后孵化出的第一个进程是 SystemServer
```
- 第一个if中"--zygote"命中，zygote变量置为true表示要启动zygote进程，并将进程名改成了zygote或zygote64
- 第二个if中"--start-system-server"命中，startSystemServer变量置为true表示要启动SystemServer进程

#### `main`方法

`app_process`主入口点是`main`方法，它是整个进程启动流程的起点。以下是其主要代码和解释：

> 源码：[[Android14的app_main.cpp](http://aospxref.com/android-14.0.0_r2/xref/frameworks/base/cmds/app_process/app_main.cpp)

```
int main(int argc, char* const argv[])
{
    // 创建并初始化AppRuntime对象runtime
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv)); 
    // 代码见源码，此处略
    if (zygote) {
        // 调用AppRuntime的start方法，开始加载ZygoteInit类
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (!className.isEmpty()) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}
```

#### `AppRuntime`类(AndroidRuntime)

**AppRuntime**继承自**AndroidRuntime**(ART)，是Android中的一个关键类，负责管理和启动 Android 应用程序或系统服务的 Java 虚拟机 (JVM)。

> [Android14的AndroidRuntime源码地址](http://aospxref.com/android-14.0.0_r2/xref/frameworks/base/core/jni/AndroidRuntime.cpp)

#####  `AndroidRuntime`类的`start`方法

`app_process`的main方法调用了`AppRuntime`的start方法，也就是`AppRuntime`的父类`AndroidRuntime`的start方法

```
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    // 初始化Java Native Interface (JNI)。JNI是Java和C/C++之间的接口，它允许Java代码和C/C++代码相互调用
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;  // JNIEnv环境指针
    // 初始化虚拟机
    if (startVm(&mJavaVM, &env, zygote, primary_zygote) != 0) {
        return;
    }
    // 注册JNI方法
    if (startReg(env) < 0) {
        return;
    }
    /*
     * 以下代码执行后，当前线程（即运行 AndroidRuntime::start 方法的线程）将成为Java虚拟机（JVM）的主线程，并且在调用env->CallStaticVoidMethod启动指定的Java类的 main 方法后，这个方法不会返回，直到 JVM 退出为止。(官方文档说明)
     */
    // 将"com.android.internal.os.ZygoteInit"转换为"com/android/internal/os/ZygoteInit"
    char* slashClassName = toSlashClassName(className != NULL ? className : "");
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        // 没有找到ZygoteInit.main()方法
    } else {
        // 通过JNI调用ZygoteInit.main()方法
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
          if (startMeth == NULL) {
              ALOGE("JavaVM unable to find main() in '%s'\n", className);
              /* keep going */
          } else {
             env->CallStaticVoidMethod(startClass, startMeth, strArray);
        }
    }
}
```

1. startVm() ：创建虚拟机
2. startReg()： 动态注册 java 调用 native 的 jni
3. 反射调用 ZygoteInit 的 main()

#### ZygoteInit

在`AndroidRuntime`的**start**方法，通过JNI调用ZygoteInit.main()，系统第一次进入Java层(ZygoteInit是系统运行的第一个Java类)，当前线程也正式成为Java虚拟机（JVM）的主线程。

> [Android14的ZygoteInit源码地址](http://aospxref.com/android-14.0.0_r2/xref/frameworks/base/core/java/com/android/internal/os/ZygoteInit.java)

通过main方法完成资源预加载、启动系统服务等功能，为Launcher桌面程序做准备。

```
public static void main(String[] argv) {
    // 创建ZygoteServer
    ZygoteServer zygoteServer = null;
    ...
    // 预加载资源
    preload(bootTimingsTraceLog);
    ...
    // 初始化ZygoteServer
    zygoteServer = new ZygoteServer(isPrimaryZygote);
    ...
    // 通过fork的形式初始化SystemServer
    Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);
    if (r != null) {
        r.run();
        return;
    }
    ...
    // 启动Loop，监听消息
    caller = zygoteServer.runSelectLoop(abiList);
    ...
}
```

##### ZygoteInit.preload()预加载

通过preload方法预加载系统常用的类、资源和库，能够显著减少应用启动时的延迟，并通过共享这些预加载的内容来降低内存使用，提高系统性能。

```
static void preload(TimingsTraceLog bootTimingsTraceLog) {
    preloadClasses(); //加载常用的Java类
    preloadResources(); //加载常用的资源（如布局、图片等）
    preloadOpenGL(); //加载OpenGL库
    preloadSharedLibraries(); //加载常用的本地共享库
    preloadTextResources(); //加载常用的文本资源
    ...
}
```

###### 常用类

- Android框架中的基础类，如Activity、Service、BroadcastReceiver等。
- 常用的UI组件类，如TextView、Button、ImageView等。

###### 常用资源

常用布局文件（layout）。
常用图片资源（drawable）。
常用字符串（strings.xml）。

###### 常用库

标准C库（libc.so）。
图形处理库（libskia.so）。
OpenGL ES库（libGLESv2.so）。

#### 启动System Server

System Server是Android系统中的关键进程，负责启动和管理核心系统服务。启动过程的核心代码:

```
public static void main(String argv[]) {
    // 初始化ZygoteServer
    ZygoteServer zygoteServer = new ZygoteServer();
    // 启动System Server
    if (startSystemServer) {
        startSystemServer(abiList, socketName, zygoteServer);
    }
    // 进入Zygote的主循环，等待新进程的启动请求
    zygoteServer.runSelectLoop();
}

private static boolean startSystemServer(String abiList, String socketName, ZygoteServer zygoteServer) {
    /* 调用native方法fork系统服务器 */
    int pid = Zygote.forkSystemServer(...);
    if (pid == 0) {
        // 在子进程中执行System Server的main方法
        handleSystemServerProcess(parsedArgs);
    }
}

private static void handleSystemServerProcess(ZygoteConnection.Arguments parsedArgs) {
    // 通过反射调用SystemServer的main方法
    ClassLoader cl = ClassLoader.getSystemClassLoader();
    Class<?> clz = cl.loadClass("com.android.server.SystemServer");
    Method m = clz.getMethod("main", new Class[] { String[].class });
    m.invoke(null, new Object[] { parsedArgs.remainingArgs });
}
```

- ZygoteServer是一个Socket,Zygote进程通过这个Socket和SystemService及其他应用程序进程做通信
- 通过fork创建的SystemServer进程是一个独立运行的进程，避免SystemServer进程受到其他进程的影响。
- 关于SystemServer，后面还会更详细的介绍

#### 系统服务 System Server

在`Zygote`中通过`Zygote.forkSystemServer`方法创建了System Server进程，然后通过`Java的反射机制`调用`com.android.server.SystemServer`的`main`方法来启动System Server。

> [Android14的System Server源码地址](http://aospxref.com/android-14.0.0_r2/xref/frameworks/base/services/java/com/android/server/SystemServer.java)

##### SystemServer.java

在`SystemServer.java`的`main`方法调用了自身的`run`方法，在run方法中启动了具体的系统服务，代码如下：

```
public static void main(String[] args) {
    new SystemServer().run();
}

private void run() {
    // 初始化系统属性，时区、语言、环境等，代码略
    ...
    // 加载本地服务
    System.loadLibrary("android_servers");
    ...
    // 初始化系统上下文
    createSystemContext();
    // 初始化主线模块
    ActivityThread.initializeMainlineModules();
    ...
    // 创建系统服务管理器
    mSystemServiceManager = new SystemServiceManager(mSystemContext);
    ...

    /* 启动系统服务 */
    // 启动引导服务
    startBootstrapServices(t);
    // 启动核心服务
    startCoreServices(t);
    // 启动其他服务
    startOtherServices(t);
    // 启动 APEX 服务
    startApexServices(t);
    ...

}
```

##### System Server启动的主要服务

以下为System Server启动的主要服务列表，具体实现可在源码中查看。

| 服务名称                            | 功能说明                                                     |
| ----------------------------------- | ------------------------------------------------------------ |
| Activity Manager Service (AMS)      | 管理应用程序的生命周期，包括启动和停止应用、管理任务和活动栈、处理广播等 |
| Package Manager Service (PMS)       | 管理应用包的安装、卸载、更新、权限分配等                     |
| System Config Service               | 管理系统配置和资源                                           |
| Power Manager Service               | 管理设备的电源状态和电源策略，如休眠、唤醒等                 |
| Display Manager Service             | 管理显示设备，如屏幕亮度、显示模式等                         |
| User Manager Service                | 管理用户账户和用户信息                                       |
| Battery Service                     | 监控和管理电池状态和电池使用情况                             |
| Vibrator Service                    | 控制设备的振动功能                                           |
| Sensor Service                      | 管理设备的传感器，如加速度计、陀螺仪等                       |
| Window Manager Service (WMS)        | 管理窗口和显示内容，包括窗口的创建、删除、布局等             |
| Input Manager Service               | 管理输入设备，如触摸屏、键盘等                               |
| Alarm Manager Service               | 提供定时任务调度功能                                         |
| Connectivity Service                | 管理网络连接，如 Wi-Fi、移动数据等                           |
| Network Management Service          | 管理网络接口和网络连接                                       |
| Telephony Registry                  | 管理电话和短信服务                                           |
| Input Method Manager Service (IMMS) | 管理输入法框架                                               |
| Accessibility Manager Service       | 管理无障碍服务，为有特殊需要的用户提供辅助功能               |
| Mount Service                       | 管理存储设备的挂载和卸载                                     |
| Location Manager Service            | 管理位置服务，如 GPS 和网络定位                              |
| Search Manager Service              | 管理系统搜索功能                                             |
| Clipboard Service                   | 管理剪贴板功能                                               |
| DevicePolicy Manager Service        | 管理设备的安全策略和企业管理功能                             |
| Status Bar Service                  | 管理状态栏显示和操作                                         |
| Wallpaper Manager Service           | 管理壁纸设置和操作                                           |
| Media Router Service                | 管理媒体设备路由                                             |

在系统服务全部启动完成后，就开始启动系统桌面程序Launcher了。

#### 桌面程序Launcher(Home)的启动流程

![image-20250108164814113](https://raw.githubusercontent.com/mendax92/pic/main/image-20250108164814113.png)

- **SystemServer 启动所有服务**: SystemServer类在run方法中调用startOtherServices方法，启动其他系统服务，包括ActivityManagerService。
- **ActivityManagerService准备系统**: ActivityManagerService 在systemReady方法中调用mAtmInternal.startHomeOnAllDisplays方法，开始在所有显示器上启动桌面程序。
- **ActivityTaskManagerService启动**: Home Activity:ActivityTaskManagerService 调用RootWindowContainer的startHomeOnAllDisplays方法。
- **RootWindowContainer循环所有显示器**: RootWindowContainer 遍历每个显示器，并调用startHomeOnDisplay方法。
- **启动Home Activity**: 在每个显示器上，通过TaskDisplayArea调用ActivityStartController的startHomeActivity方法，最终调用ActivityManagerService的startActivity方法启动Home Activity。
- **Home应用启动**: ActivityManagerService处理启动请求，启动Home应用的Activity展示桌面界面。

##### SystemServer.java

> [Android14的System Server源码地址](http://aospxref.com/android-14.0.0_r2/xref/frameworks/base/services/java/com/android/server/SystemServer.java)

```
private void run() {
    ...
    // 启动其他服务
    startOtherServices(t);
    ...
}

private void startOtherServices(@NonNull TimingsTraceAndSlog t) {
    ...
    // 启动ActivityManagerService
    mActivityManagerService = mSystemServiceManager.startService(
            ActivityManagerService.Lifecycle.class).getService();
    ...
    // 启动Launcher
    mActivityManagerService.systemReady(...)
    ...
}
```

##### ActivityManagerService.java

> [Android14的ActivityManagerService源码地址](http://aospxref.com/android-14.0.0_r2/xref/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java)

```java
public void systemReady(final Runnable goingCallback, TimingsTraceAndSlog t) {
    ...
    // 在所有显示器上启动Launcher
    mAtmInternal.startHomeOnAllDisplays(currentUserId, "systemReady");
    ...
}
```

- 此行代码最终会调用到`ActivityTaskManagerService.java`的`startHomeOnAllDisplays`方法

##### ActivityTaskManagerService.java

> [Android14的ActivityTaskManagerService源码地址](http://aospxref.com/android-14.0.0_r2/xref/frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService)

```java
void startHomeOnAllDisplays(int userId, String reason) {
    synchronized (mGlobalLock) {
        // 在所有显示器上启动Launcher
        return mRootWindowContainer.startHomeOnAllDisplays(userId, reason);
    }
}
```

- 此行代码最终会调用到`RootWindowContainer.java`的`startHomeOnAllDisplays`方法

##### RootWindowContainer.java

> [Android14的RootWindowContainer源码地址](http://aospxref.com/android-14.0.0_r2/xref/frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java)

```java
 boolean startHomeOnAllDisplays(int userId, String reason) {
    boolean homeStarted = false;
    for (int i = getChildCount() - 1; i >= 0; i--) {
        final int displayId = getChildAt(i).mDisplayId;
        homeStarted |= startHomeOnDisplay(userId, reason, displayId);
    }
    return homeStarted;
}

boolean startHomeOnDisplay(int userId, String reason, int displayId) {
    return startHomeOnDisplay(userId, reason, displayId, false /* allowInstrumenting */,
            false /* fromHomeKey */);
}

boolean startHomeOnDisplay(int userId, String reason, int displayId, boolean allowInstrumenting,
        boolean fromHomeKey) {
    // 如果displayId无效，则回退到默认桌面
    if (displayId == INVALID_DISPLAY) {
        final Task rootTask = getTopDisplayFocusedRootTask();
        displayId = rootTask != null ? rootTask.getDisplayId() : DEFAULT_DISPLAY;
    }
    final DisplayContent display = getDisplayContent(displayId);
    return display.reduceOnAllTaskDisplayAreas((taskDisplayArea, result) ->
                    result | startHomeOnTaskDisplayArea(userId, reason, taskDisplayArea,
                            allowInstrumenting, fromHomeKey),
            false /* initValue */);
}

boolean startHomeOnTaskDisplayArea(int userId, String reason, TaskDisplayArea taskDisplayArea,
        boolean allowInstrumenting, boolean fromHomeKey) {
    ...
    
    Intent homeIntent = null;
    ActivityInfo aInfo = null;
    if (taskDisplayArea == getDefaultTaskDisplayArea()) {
        // 构建一个category为CATEGORY_HOME的Intent，表明是Home Activity
        homeIntent = mService.getHomeIntent();
        // 通过PMS从系统所用已安装的应用中，找到一个符合HomeItent的Activity
        aInfo = resolveHomeActivity(userId, homeIntent);
    } else if (shouldPlaceSecondaryHomeOnDisplayArea(taskDisplayArea)) {
        Pair<ActivityInfo, Intent> info = resolveSecondaryHomeActivity(userId, taskDisplayArea);
        aInfo = info.first;
        homeIntent = info.second;
    }
   ...
    // 启动Home Activity--Luncher
    mService.getActivityStartController().startHomeActivity(homeIntent, aInfo, myReason,
            taskDisplayArea);
    return true;
}
 
```

- startHomeOnTaskDisplayArea()首先通过getHomeIntent()来构建一个category为CATEGORY_HOME的Intent，表明是Home Activity；然后通过resolveHomeActivity()从系统所用已安装的引用中，找到一个符合HomeItent的Activity，最终调用startHomeActivity()来启动Activity。

- getHomeIntent()方法构建一个category为CATEGORY_HOME的Intent，表明是Home Activity。 Intent.CATEGORY_HOME = “android.intent.category.HOME” 这个category会在Launcher3的 AndroidManifest.xml中配置，表明是Home Acivity

```
Intent getHomeIntent() {
    Intent intent = new Intent(mTopAction, mTopData != null ? Uri.parse(mTopData) : null);
    intent.setComponent(mTopComponent);
    intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
    //不是生产模式，add一个CATEGORY_HOME
    if (mFactoryTest != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
        intent.addCategory(Intent.CATEGORY_HOME);
    }
    return intent;
}
```

resolveHomeActivity()方法通过Binder跨进程通知PackageManagerService从系统所用已安装的引用中，找到一个符合HomeItent的Activity

```
ActivityInfo resolveHomeActivity(int userId, Intent homeIntent) {
   final int flags = ActivityManagerService.STOCK_PM_FLAGS;

   //系统正常启动时，component为null
   final ComponentName comp = homeIntent.getComponent(); 
   ActivityInfo aInfo = null;
   ...

       if (comp != null) {
           // Factory test.
           aInfo = AppGlobals.getPackageManager().getActivityInfo(comp, flags, userId);
       } else {
           //系统正常启动时，走该流程
           final String resolvedType =
                   homeIntent.resolveTypeIfNeeded(mService.mContext.getContentResolver());
           
           //resolveIntent做了两件事：
           // 1.通过queryIntentActivities来查找符合HomeIntent需求Activities
           // 2.通过chooseBestActivity找到最符合Intent需求的Activity信息
           final ResolveInfo info = AppGlobals.getPackageManager()
                   .resolveIntent(homeIntent, resolvedType, flags, userId);
           if (info != null) {
               aInfo = info.activityInfo;
           }
       }

   ...
   aInfo = new ActivityInfo(aInfo);
   aInfo.applicationInfo = mService.getAppInfoForUser(aInfo.applicationInfo, userId);
   return aInfo;
}
```

##### ActivityStartController.java

> [Android14的ActivityStartController源码地址](http://aospxref.com/android-14.0.0_r2/xref/frameworks/base/services/core/java/com/android/server/wm/ActivityStartController.java)

      void startHomeActivity(Intent intent, ActivityInfo aInfo, String reason,
              TaskDisplayArea taskDisplayArea) {
          final ActivityOptions options = ActivityOptions.makeBasic();
          options.setLaunchWindowingMode(WINDOWING_MODE_FULLSCREEN);
          if (!ActivityRecord.isResolverActivity(aInfo.name)) {
              // The resolver activity shouldn't be put in root home task because when the
              // foreground is standard type activity, the resolver activity should be put on the
              // top of current foreground instead of bring root home task to front.
              options.setLaunchActivityType(ACTIVITY_TYPE_HOME);
          }
          final int displayId = taskDisplayArea.getDisplayId();
          options.setLaunchDisplayId(displayId);
          options.setLaunchTaskDisplayArea(taskDisplayArea.mRemoteToken
                  .toWindowContainerToken());
      
          // The home activity will be started later, defer resuming to avoid unnecessary operations
          // (e.g. start home recursively) when creating root home task.
          mSupervisor.beginDeferResume();
          final Task rootHomeTask;
          try {
              // Make sure root home task exists on display area.
              rootHomeTask = taskDisplayArea.getOrCreateRootHomeTask(ON_TOP);
          } finally {
              mSupervisor.endDeferResume();
          }
      
          mLastHomeActivityStartResult = obtainStarter(intent, "startHomeActivity: " + reason)
                  .setOutActivity(tmpOutRecord)
                  .setCallingUid(0)
                  .setActivityInfo(aInfo)
                  .setActivityOptions(options.toBundle())
                  .execute();
          mLastHomeActivityStartRecord = tmpOutRecord[0];
          if (rootHomeTask.mInResumeTopActivity) {
              // If we are in resume section already, home activity will be initialized, but not
              // resumed (to avoid recursive resume) and will stay that way until something pokes it
              // again. We need to schedule another resume.
              mSupervisor.scheduleResumeTopActivities();
          }
      }

obtainStarter() 方法返回的是 ActivityStarter 对象，它负责 Activity 的启动，一系列 setXXX() 方法传入启动所需的各种参数，最后的 execute() 是真正的启动逻辑。

###  总结

所有的进程都由Zygote创建，zygote主要用来孵化`system_server进程`和`应用程序进程`。在孵化出第一个进程system_server后通过`runSelectLoop`等待并处理消息，分裂应用程序进程仍由system_server控制，`等待 AMS 给他发消息（告诉 zygote 创建进程）`，如app启动时创建子进程。

从AndroidRuntime到ZygoteInit，主要分为3大过程：

> 1、`创建虚拟机——startVm()`:调用JNI虚拟机创建函数
> 2、`注册JNI函数——startReg()`：前面已经创建虚拟机，这里给这个虚拟机注册一些JNI函数（后续java世界用到的函数是native实现，这里需要提前注册注册这些函数）
> 3、此时就要`执行CallStaticViodMethod`，通过这个函数将进入android精心打造的java世界，这个函数将调用com.android.internal.os.ZygoteInit的main函数

在 ZygoteInit.main函数中进入Java世界，主要有4个关键步骤：

> 1、`预加载类和资源——preload()`
> 主要是preloadClasses和preloadResources，其中preloadClasses一般是加载时间超过1250ms的类，因而需要在zygote预加载
> 2、`建立IPC通信服务——初始化ZygoteServer，内部初始化了ZygoteSocket`
> zygote及系统中其他程序的通信并没有使用Binder，而是采用基于AF_UNIX类型的Socket，作用正是建立这个Socket
> 3、`启动system_server——forkSystemServer()`
> 这个函数会创建Java世界中系统Service所驻留的进程system_server,该进程是framework的核心，也是zygote孵化出的第一个进程。如果它死了，就会导致zygote自杀。
> 4、`等待请求——runSelectLoop()`
> zygote从startSystemServer返回后，将进入第四个关键函数runSelectLoop，在第一个函数ZygoteServer中注册了一个用于IPC的Socket将在这里使用，这里Zygote采用高效的I/O多路复用机制，保证在没有客户端请求时或者数据处理时休眠，否则响应客户端的请求。`等待 AMS 给他发消息（告诉 zygote 创建进程）`。此时zygote完成了java世界的初创工作，调用runSelectLoop便开始休眠了，当收到请求或者数据处理便会随时醒来，继续工作。

#### 面试题

##### 1 init进程作用是什么

init进程起着承上启下的作用，Android本身是基于Linux而来的，init进程是Linux系统中用户空间的第一个进程。init进程属于一个守护进程，准确的说，它是Linux系统中用户控制的第一个进程，它的进程号为1（进程号为0的为内核进程），它的生命周期贯穿整个Linux内核运行的始终。Android中所有其它的进程共同的鼻祖均为init进程。
Android Q(10.0) 的init入口函数由原先的init.cpp 调整到了main.cpp，把各个阶段的操作分离开来，使代码更加简洁命令。
作为天子第1号进程，init被赋予了很多重要的职责，主要分为三个阶段：

> 1. init进程第一阶段做的主要工作是`挂载分区`，创建设备节点和一些关键目录，初始化日志输出系统,启用SELinux安全策略。
> 2. init进程第二阶段主要工作是`初始化属性系统`，解析SELinux的匹配规则，处理子进程终止信号，启动系统属性服务，可以说每一项都很关键，如果说第一阶段是为属性系统，SELinux做准备，那么第二阶段就是真正去把这些功能落实。
> 3. init进行第三阶段主要是`解析init.rc来启动其他进程`，进入无限循环，进行`子进程实时监控(守护)`。

其中第三阶段通过initrc启动其他进程，我们常见的比如`启动Zygote进程`、`启动SeviceManager进程`等。

##### 2 Zygote进程最原始的进程是什么进程(或者Zygote进程由来)

Zygote最开始是`app_process`，它是在 init 进程启动时被启动的，在`app_main.cpp`才被修改为 Zygote。

##### 3 Zygote 是在内核空间还是在用户空间？

因为 init 进程的创建在用户空间，而 Zygote 是由 init 进程创建启动的，所以`Zygote是在用户空间。`

##### 4 Zygote为什么需要用到Socket通信而不是Binder

Zygote是Android中的一个重要进程，它是启动应用程序进程的父进程。Zygote使用Socket来与应用程序进程进行通信，而不是使用Android中的IPC机制Binder，这是因为Socket和Binder有不同的优缺点，而在Zygote进程中使用Socket可以更好地满足Zygote进程的需求。

1. `Zygote 用 binder 通信会导致死锁`
   假设 Zygote 使用 Binder 通信，因为 Binder 是支持多线程的，存在并发问题，而并发问题的解决方案就是加锁，如果进程 fork 是在多线程情况下运行，Binder 等待锁在锁机制下就可能会出现死锁。
2. `Zygote 用 binder 通信会导致读写错误`
   根本原因在于要 new 一个 ProcessState 用于 Binder 通信时，需要 mmap 申请一片内存用以提供给内核进行数据交换使用。而如果直接 fork 了的话，子进程在进行 binder 通信时，内核还是会继续使用父进程申请的地址写数据，而此时会触发子进程 COW（Copy on Write），从而导致地址空间已经重新映射，而子进程还尝试访问之前父进程 mmap 的地址，会导致 SIGSEGV、SEGV_MAPERR段错误。
3. `Zygote初始化时，Binder还没开始初始化。`
4. `Socket具有良好的跨平台性`，能够在不同的操作系统和语言之间进行通信。这对于Zygote进程来说非常重要，因为它需要在不同的设备和架构上运行，并且需要与不同的应用程序进程进行通信。使用Socket可以让Zygote进程更加灵活和可扩展，因为它不需要考虑Binder所带来的特定限制和要求。
5. `Socket具有简单的API和易于使用的特点`。Zygote进程需要快速启动并与应用程序进程建立通信，Socket提供了快速、可靠的通信方式，并且使用Socket API也很容易实现。相比之下，Binder需要更多的配置和维护工作，这对于Zygote进程来说可能会增加不必要的复杂性和开销。
6. `Socket在数据传输时具有更低的延迟和更高的吞吐量`，这对于Zygote进程来说非常重要。Zygote进程需要在较短的时间内启动应用程序进程，并且需要传输大量的数据和代码，Socket的高性能和低延迟使其成为更好的选择。

**总之，Zygote进程使用Socket而不是Binder是基于其优点和需求而做出的选择。虽然Binder在Android中扮演着重要的角色，但在某些情况下，使用Socket可以提供更好的性能和更大的灵活性。再者，Binder当初并不成熟，团队成员对于进程间通讯更倾向于用Socket，后面为了做了很多优化，才使得Binder通讯变得成熟稳定。**

##### 5 每个App都会将系统的资源，系统的类都加载一遍吗

zygote进程的作用：
1.创建一个Service端的Socket，开启一个ServerSocket实现和别的进程通信。
2.加载系统类，系统资源。
3.启动System Server进程

**Zygote进程预加载系统资源后，然后通过它孵化出其他的虚拟机进程，进而共享虚拟机内存和框架层资源`（共享内存）`，这样大幅度提高应用程序的启动和运行速度。**

##### 6 PMS 是干什么的，你是怎么理解PMS

**包管理，包解析，结果缓存，提供查询接口。**

1. 遍历`/data/app`的文件夹
2. 解压`apk`文件
3. dom解析`AndroidManifest.xml`文件。

##### 7 为什么会有AMS AMS的作用

1. 查询PMS
2. 反射生成对象
3. 管理Activity生命周期

AMS缓存中心：`ActivityThread`

##### 8 AMS如何管理Activity，探探AMS的执行原理

Activity在应用端由`ActivityClientRecord`负责描述其生命周期的过程与状态，但最终这些过程与状态是由`ActivityManagerService(以下简称AMS)`来管理和控制的

1. `BroadcastRecord`：描述了应用进程的BroadcastReceiver，由`BroadcastQueue`负责管理。
2. `ServiceRecord`：描述了Service服务组件，由`ActiveServices`负责管理。
3. `ContentProviderRecord`：描述ContentProvider内容提供者，由`ProviderMap`管理。
4. `ActivityRecord`：用于描述Activity，由`ActivityStackSupervisor`进行管理。
