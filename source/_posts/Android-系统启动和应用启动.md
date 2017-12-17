---
title: Android 系统启动和应用启动
date: 2017-12-17 17:47:01
tags:
---
### 系统启动

电源键按下后，引导芯片将开始从预定义的地方开始执行，加载引导程序到 RAM，进而执行引导程序，一般引导程序为针对主板和芯片的预定义程序，引导程序也是OEM厂商或者运营商加锁和限制的地方。此外会初始化内核所需要的比如网络、内存等，此后将会启动内核。在内核启动完成后，便会创建第一个进程，便是 init 进程。 

init 是 Android 系统中用户空间的第一个进程。init 进程负责创建系统中的几个关键进程，例如 log 系统，在此处要注意的是 zygote 进程。
<!-- more -->

zygote 本身是一个 Native 的程序，源码中其入口在 /platform_frameworks_base/cmds/app\_process/
App_main.cpp, zygote 中的 main 函数中主要的功能均由 AppRuntime 完成，此后的活动也均由 AppRuntime 控制，/platform_frameworks_base/core/jni/AndroidRuntime.cpp 

AppRuntime 重载了 onStarted、onZygoteInit 和 onExit 函数，在整个的实现过程中，可以看到通过 startVm 调用 JNI 虚拟机的创建函数，在 startReg 中注册 JNI 函数，要知道 JNI 是连接 Java 和 Native 的桥梁，至此完成了虚拟机的创建。所谓虚拟机在 Linux 下就是一个进程，Android 中每一个程序都有一个单独的进程，此后通过 CallStaticVoidMethod 方法，将进入到 Java 层面。

作为 Java 世界的入口，通过 CallStaticVoidMethod 方法，最终将调用 ZygoteInit.java 的 main 函数。 main 函数里几个主要调用的函数：

* registerZygoteSocket — 建立 IPC 通信服务端，由于 zygote 在与其他进程的通信中没有使用到 Binder 机制，而是采用了基于 AF_UNIX 类型的 Socket。
* preload — 预加载资源，在 preload() 方法中，preloadClasses() 会预加载列表在 /platform_frameworks_base/tools/preload 下的预加载类，preloadResources（）会预加载drawable和color等资源。会进行加载时间的判断，当每个类的加载时间大于 1250 微秒的时候，就会被 zygote 预加载，所以在这里也是一个优化系统启动速度的优化点，但是对技术的要求比较高~
* startSystemServer — 启动 System_server，在这个方法中，通过设置参数和 fork 的方式，zygote 实现了一次分裂，分裂出一个 system_server 进程，即代码中对应 Zygote.forkSystemServer。
* runSelectLoopMode — 开启服务端等待请求
在 zygote 从startSystemServer 返回之后，进入到下一个关键函数：runSelectLoopMode 。在 registerZygoteSocket 中我们注册了一个用户 IPC 通信的 Socket，但是当时并没有启用，而它真正启用的地方便是在 runSelectLoopMode 中，并且在这里 zygote 采用的是高效的 I/O 多路复用机制，保证在没有客户端请求和数据处理的时候休眠，在客户端请求时响应处理。

虽然只是系统启动流程了解，但是到现在结合上面的流程图，总结一波在这里 zygote 究竟干了什么：

1. 创建 AppRuntime 对象并调用它的 start 方法，启动 zygote 进程，此后的操作活动由 AppRuntime 来控制。
2. 调用 startVm 创建 Java 虚拟机，调用 startReg 来注册 JNI 函数。
3. 通过 JNI 调用 com.android.internal.os.ZygoteInit下的 main 函数，初始化整个 Java 层。
4. 通过 registerZygoteSocket 函数创建服务端 Socket，并通过 runSelectLoop 函数等待 ActivityManagerService 的请求来创建新的应用程序进程。
5. 启动 SystemServer 进程。


在 ZygoteInit.java 中，调用了 startSystemServer 进程，SystemServer 进程的进程名实际为：system_server，通过前面内容可知，system_server 是zygote 通过 fork 诞生的，在执行 handleSystemServerProcess 方法的时候，发生了什么呢？通过 nativeZygoteInit 和 application 两个 init 函数来实现 system_server 的初始化。

顺着 nativeZygoteInit() 方法看下去，会发现启动了一个 Binder 的线程池，用于和其他进程进行通信，可以发现 nativeZygoteInit() 主要便是启动 Binder 线程池。

zygote 分裂产生 SystemServer ，其实就是调用 com.android.server.SystemServer 的 main 函数，代码中可以看到把 Service 分为 引导、核心、其他三大类，可以对服务进行管理等操作，通过系统 C/S 架构中的 Binder 进行通信，启动各种系统服务并对其i进行管理。

SystemServer 主要工作为下面三个方面：

1. 启动 Binder 线程池
2. 创建SystemServiceManager用于对系统的服务进行创建、启动和生命周期管理
3. 启动各种系统服务

Android 系统启动的最后一步，是加载 Launcher 应用程序，Launcher 是一个桌面应用，用来展示已经安装的应用程序入口，Launcher 在启动过程中会请 求PackageManagerService 返回系统中已经安装的应用程序的信息，并将这些信息封装成一个快捷图标列表显示在系统屏幕上，这样用户可以通过点击这些快捷图标来启动相应的应用程序。launcher在启动SystemServer中的其他类别中启动，最终会调用 ActivityManagerService 的 startHomeActivityLocked 方法，该方法中会启动Launcher。

系统启动流程总结:

1. 按下电源键 --> 引导程序 Bootloader 加载到 RAM 并开始执行。
2. 内核启动，加载驱动、设置缓存，完成系统设置等 =》启动 init 进程
3. init 进程初始化 --> 启动 Zygote 进程
4. 创建 Java VM 、注册 JNI 、创建服务端 Socket 、启动 SystemServer
5. 启动 Binder 线程池和 SystemServiceManager ，并且启动各种系统服务
6. SystemServer 进程 --> 启动 ActivityManagerService =》启动Launcher，Launcher 将已安装应用的快捷图标显示到界面上。
7. 完成系统启动流程。

### 应用启动

用户点击到某一个应用的时候，点击事件会通过 startActivity(Intent) 方法启动一个 Activity，通过Binder IPC 机制，最终会调用到 ActivityManagerService ，该 Service 会执行以下几步操作：

* 通过 PackageManager 的 resolveIntent() 收集这个 intent 的指向信息。
* 将指向信息存储到一个 intent 中。
* 通过 grantUriPermissionLocked() 方法验证用户是否有足够的权限调用 intent 指向的目标 Activity
* 有权限， AMS 会检查并创建一个新的 task 来启动 Activity
* 检查该进程的 PricessRecord 是否存在，如果为 null AMS 会创建新的进程来实例化 Activity
* 执行 Activity 的生命周期 --> 应用启动完成。

ActivityManagerService 调用 startProcessLocked() 方法来创建新的进程，在该方法中，会通过 socket 将参数传递给 zygote 进程，zygote 分裂，调用 ZygoteInit.man() 方法来实例化 ActivityThread 并最终返回新的进程 pid；随后 ActivityThread 会一次调用 Looper.prepareLoop() 和 Looper.loop() 来开启消息循环。

在进程创建完成后，需要将进程和 Application 进行绑定，通过调用 ActivityThread 对象中的 bindApplication() 方法来完成，发送一个 BIND_APPLICATION 消息到 MessageQueue 中，进而通过 handlerBindApplication() 方法处理该消息，调用 makeApplication() 方法来加载 App 的 classes 到内存中。

经过上述步骤，系统中便存在了该 Application 的进程，后面的调用顺序为从一个已经存在的进程中启动一个新进程的 Activity了。调用方法为 realStartActivity() 它会调用 ApplicationThread 对象中的 scheduleLaunchActivity 发送一个 LAUNCH_ACTIVITY 的消息到消息队列中，通过 handleLaunchActivity() 来处理该消息。

