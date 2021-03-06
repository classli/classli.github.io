---
title: ContentProvider原理浅析
date: 2018-01-07 17:30:10
tags:
---
在Android世界里夸进程传输主要靠的就是Binder, 但单纯使用Binder是很难一次性传输大量数据，实际只有1M的缓存区。而作为Android系统数据共享的另一种方式ContentProvider，采用了匿名共享内存(Ashmem)机制完成数据共享，它可以方便地完成大量数据的传递。

### ContentResolver的获取

通常我们通过getContentResolver获得ContentResolver已进行后续操作，ContextImpl类里实现了该方法，其直接返回了一个ApplicationContentResolver对象。 ApplicationContentResolver 继承至ContentResolver。
<!-- more -->
通过ContentResolver进行的query等操作其实就是调用的ContentResolver里的具体方法。ContentResolver的query通过获取IContentProvider(明显的binder接口)来调用查询接口，获得IContentProvider的具体逻辑在ApplicationContentResolver类的两个方法中实现，最终都是调用ActivityThread类的acquireProvider获取到IContentProvider。

### 获取本进程的ContentProvider

ActivityThread类中的acquireProvider方法最后一个参数代表ContentProvider所在的进程是否存活，如果进程已死则在必要的时候需唤醒它。acquireProvider里的acquireExistingProvider首先尝试从本进程中获取ContentProvider，如果获取不到再请求AMS获取对应ContentProvider。不论是从哪里获取到的ContentProvider，获取完毕之后会调用installProvider来安装ContentProvider在本地进程（ArrayMap表）。

installProvider安装即通过反射创建出这个ContentProvider对象，并取出这个对象中承载跨进程通信的Binder对象IContentProvider，将这个Binder对象作为参数创建出ProviderClientRecord对象保存在ActivityThread内部。

ContentProvider是一个数据共享组件，为了支持跨进程共享采用了Binder，为了共享大量数据使用了匿名共享内存。ContentProvider主要支持query, insert, update, delete操作，由于这个组件一般工作在别的进程，因此这些调用都是Binder调用，这些调用最终都是委托给一个IContentProvider的Binder对象完成。

### 获取其他进程的ContentProvider

如果本地不存在ContentProvider那么会通过AMS的getContentProvider方法查询整个系统中是否存在这个ContentProvider。getContentProvider直接调用了getContentProviderImpl，该方法干了很多事情，最终目的可以总结为两点：

* 使用PackageManagerService的resolveContentProvider方法根据Uri中的auth信息查询对应ContentProivoder的ProviderInfo信息。
* 根据查询到的ContentProvider信息，尝试将这个ContentProvider组件安装到系统上。

PackageManagerService通过mProvidersByAuthority这个ArrayMap保存着各个应用中ContentProvider的信息。这个map是通过scanPackageDirtyLI方法在Android系统启动的时候收集的，Android启动时会扫描一些目录，如/data/app/*，获取里面各个apk的AndroidManifest.xml文件内容信息，这些信息就会保存在PackageManagerService中。

AMS对于ContentProvider的安装必须委托于目标进程。这里有两种情况：

* 目标进程已经运行，那么AMS直接通知目标进程进行安装（已安装不用）
* 目标进程已经死亡，则唤醒它叫他干活。

如果ContentProvider所在进程处于运行状态，那么AMS通过ApplicationThread这个Binder对象完成scheduleInstallProvider调用，最终会调用到目标进程的installProvider方法。

如果ContentProvider所在进程已经死亡，则会调用startProcessLocked来启动新的进程，最终会调用Process类的start方法，该方法通过socket与Zygote进程进行通信，通知Zygote进程fork出一个子进程，然后通过反射调用之前传递过来的一个入口类的main函数，一般来说这个入口类就是ActivityThread，因此子进程fork出来之后会执行ActivityThread类的main函数。

调用startProcessLocked后AMS会在getContentProviderImpl结尾等待目标进程创建好并完成安装，等到目标集成安装完成后就把对应的ContentProvider返回给调用方。

目标进程将ContentProvider安装到AMS是在app启动过程中的handleBindApplication方法，里面调用了installContentProviders方法完成操作，该方法早于Applition类的onCreate回调。

### unstable与stable

一般先获取unstable的Provider，执行失败后再获取stable的Provider，最终都是调用上面说是的acquireProvider来获取IContentProvider。stable与unstable的区别，采用unstable类型的ContentProvider的app不会因为远程ContentProvider进程的死亡而被杀，stable则恰恰相反。