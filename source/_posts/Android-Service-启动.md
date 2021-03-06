---
title: Android Service 启动
date: 2017-12-24 16:55:25
tags:
---
### startService

startService真正实现者是ContextImpl，ContextImpl类的startService开始，然后进入本类的startServiceCommon方法，最终调用ActivityManagerNative.getDefault()对象的startService方法。ActivityManagerNative.getDefault()就是ActivityManagerProxy对象。

ActivityManagerProxy的startService方法通过binder 调用了ActivityManagerService服务的startService方法，里面又调用了 ActiveServices的startServiceLocked方法。startServiceLocked对传入的intent进行了解析 检测权限等操作。
<!-- more -->
后续会调用到bringUpServiceLocked方法，如果Service的进程已经存在 则直接调用ActivityThread 的 scheduleCreateService进行启动，如果不存在则调用ActivityManagerService.startProcessLocked方法来启动新进程，startProcessLocked会调用ActivityThread.main()方法，经过一些列的binder调用（从ActivityThread到ActivityThread，后又回到ApplicationThread）最后到scheduleCreateService。

scheduleServiceArgs会发handler通知，handleCreateService会对这个通知进行处理进行实际的Service创建。

### bindService

ContextImpl中的bindService方法直接调用了bindServiceCommon。方法最终通过ActivityManagerNative借助AMS进而完成Service的绑定过程。该方法里面有个IServiceConnection类型变量，这个IServiceConnection与IApplicationThread以及IIntentReceiver相同，都是ActivityThread给AMS提供的用来与之进行通信的Binder对象。

这个方法最终调用了ActivityManagerNative的bindService, bindService这个方法相当简单，只是做了一些参数校检之后直接调用了ActivityServices类的bindServiceLocked方法。

该方法首先它通过retrieveServiceLocked方法获取到了intent匹配到的需要bind到的Service组件res；然后把ActivityThread传递过来的IServiceConnection使用ConnectionRecord进行了包装，方便接下来使用；最后如果启动的FLAG为BIND\_AUTO_CREATE，那么调用bringUpServiceLocked开始创建Service。

如果Service所在的进程已经启动，那么直接调用realStartServiceLocked方法来真正启动Service组件；如果Service所在的进程还没有启动，那么先在AMS中记下这个要启动的Service组件，然后通过startProcessLocked启动新的进程。

这个方法首先调用了app.thread的scheduleCreateService方法，我们知道，这是一个IApplicationThread对象，它是App所在进程提供给AMS的用来与App进程进行通信的Binder对象，这个Binder的Server端在ActivityThread的ApplicationThread类，因此，我们跟踪ActivityThread类.

ApplicationThread的scheduleCreateService 方法发了一个消息给ActivityThread的H这个Handler，H类收到这个消息之后，直接调用了ActivityThread类的handleCreateService方法。

这里Service类的创建过程与Activity是略微有点不同的，虽然都是通过ClassLoader通过反射创建，但是Activity却把创建过程委托给了Instrumentation类，而Service则是直接进行。

现在ActivityThread里面的handleCreateService方法成功创建出了Service对象，并且调用了它的onCreate方法。回到了AMS进程的realStartServiceLocked方法，在scheduleCreateService调用结束后有个requestServiceBindingsLocked，这里面调用了ApplicationThread的scheduleBindService方法,后面套路一样，handleBindService对service的onBind处理完后调用了AMS的publishService，这个方法简单地调用了publishServiceLocked方法。

在bindServiceLocked方法里面，我们把这个IServiceConnection放到了一个ConnectionRecord的List中存放在ServiceRecord里面，这里所做的就是取出已经被Bind的这个Service对应的IServiceConnection对象，然后调用它的connected方法；我们说过，这个IServiceConnection也是一个Binder对象，它的Server端在LoadedApk.ServiceDispatcher里面。代码到这里已经很明确了LoadedApk.ServiceDispatcher的connected方法完成ServiceConnection回调过程。


