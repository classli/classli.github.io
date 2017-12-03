---
title: dex2oat浅析
date: 2017-12-03 22:02:37
tags:
---

dex2oat是一个能将dex文件转化成oat文件的工具，它是一个可执行ELF文件，adb连接手机后可直接调用。其源文件路径/art/dex2oat/dex2oat.cc，那么我们apk安装时它时如何被调用的呢？

当我们安装一个app的时候，实际会跳转到了一个安装界面，即PackageInstaller中的一个Activity界面 InstallAppProgress，点击确定按钮后开始安装，后续实际调用的是ApplicationPackageManager中的installPackage方法。

installCommon中又是调用了PMS的installPackageAsUser进行安装.
<!-- more -->

	public void installPackageAsUser(...) {
		enforceCrossUserPermission(callingUid, userId,
	                true /* requireFullPermission \*/, true /* checkShell */, "installPackageAsUser");
	    ...
	    final Message msg = mHandler.obtainMessage(INIT_COPY);
	    ...
	    mHandler.sendMessage(msg);
	}
	
installPackageAsUser中当安装进程是shell或者root时，会打上INSTALL\_FROM\_ADB标，当具有INSTALL\_FROM\_ADB和INSTALL\_ALL\_USERS标时才是安装给所有用户。

Android6.0后当程序需要运行时权限时，会弹窗提醒用户授权，但系统app却不需要，这是因为系统app中有INSTALL\_GRANT\_RUNTIME\_PERMISSIONS权限，installPackageAsUser中就会检查该权限。

从installPackageAsUser开始安装大致分三个部分。

* 权限检查
* apk复制
* 装载应用

权限检查主要是检查进程是否有权限安装。

### apk复制

权限检查完后会发起一个INIT\_COPY消息，PackageHandler接收该信息并开始处理。在INIT\_COPY消息处理的case中，主要是将handler绑定到一个异步线程中，并对安装请求进行排队处理，对应每个安装请求又发送一个MCS_BOUND消息。

在MCS_BOUND消息的处理case中调用了InstallParams类的startCopy()方法来执行实际的拷贝操作。startCopy()方法通过调用其子类InstallParams的handleStartCopy()来完成拷贝操作。apk被拷贝到临时阶段性文件夹/data/app/vmdl<安装回话id>.tmp/这个目录下。

### 装载应用

当apk拷贝完后startCopy会调用handleReturnCode方法，handleReturnCode 又直接调用了processPendingInstall方法。processPendingInstall中直接post了一个Runnable对象，后续可以进行异步处理。

在run方法中调用了installPackageTracedLI方法，最终是调用了installPackageLI方法。

	private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {
		...
		pkg = pp.parsePackage(tmpPackageFile, parseFlags);
		...
		mPackageDexOptimizer.performDexOpt(pkg, pkg.usesLibraryFiles,
	                    null /* instructionSets \*/, false /* checkProfiles */,
	                    getCompilerFilterForReason(REASON_INSTALL));
	   ...
	   if (replace) {
	      replacePackageLIF(pkg, parseFlags, scanFlags | SCAN_REPLACING, args.user,
	                        installerPackageName, res);
	   } else {
	      installNewPackageLIF(pkg, parseFlags, scanFlags | SCAN_DELETE_DATA_ON_FAILURES,
	                        args.user, installerPackageName, volumeUuid, res);
	   }
	   ...
	}
	
在installPackageLI里其实做了很多工作，如pp.parsePackage对apk的manifest进行解析，收集apk签名，覆盖安装的处理，apk中定义权限处理等，最后会调用installNewPackageLI或replacePackageLIF进行安装。

### dex2oat 调用

我们这里关注的是mPackageDexOptimizer.performDexOpt，这里才开始了正在的dex2oat操作，将apk中的dex文件转换为oat文件，oat文件一般都放在/data/dalvik-cache/目录下。

PackageDexOptimizer类中的performDexOpt又调用了performDexOptLI方法。performDexOptLI里直接调用了mInstaller.dexopt方法进行转换。Installer的dexopt直接调用了InstallerConnection的dexopt。

	public void dexopt(String apkPath, int uid, String pkgName, String instructionSet,
	            int dexoptNeeded, String outputPath, int dexFlags, String compilerFilter,
	            String volumeUuid, String sharedLibraries) throws InstallerException {
	        execute("dexopt",
	                apkPath,
	                uid,
	                pkgName,
	                instructionSet,
	                dexoptNeeded,
	                outputPath,
	                dexFlags,
	                compilerFilter,
	                volumeUuid,
	                sharedLibraries);
	    }
里面将"dexopt"作为参数调用了execute，execute会将传入的参数组装成一个命令字符串cmd，最终通过socekt的方式传给了Socekt服务端installd.cpp。installd.cpp里会死循环等待这个链接，一旦有链接接入，就会执行传过来的命令字符串。

在installd.cpp的execute函数中会通过一个映射方式将传入的命令字符串与对应的函数对应起来，前面出入的dexopt就对应着do_dexopt函数。

	struct cmdinfo cmds[] = {
	    { "ping",                 0, do_ping },
	    ...
	    { "dexopt",              10, do_dexopt },
	   ...
	    { "dump_profiles",        3, do_dump_profiles },
	};

do_dexopt函数直接调用了commands.cpp的dexopt函数，该dexopt函数会fork一个子线程，在子线程里又调用了run\_dex2oat函数进行oat编译。
	
	static void run_dex2oat(...) {
		...
		static const char* DEX2OAT_BIN = "/system/bin/dex2oat";
		...
		int i = 0;
		argv[i++] = DEX2OAT_BIN;
		...
		execv(DEX2OAT_BIN, (char * const *)argv);
		...
	}

在run\_dex2oat里最终通过execv函数执行命令调用了dex2oat工具进行oat转换。

整个dex2oat的调用过程就是上面这样了~~