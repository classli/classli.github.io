---
title: OAT文件解析
date: 2017-12-17 16:01:28
tags:
---

在ART虚拟机上，当应用首次安装或者动态地通过ClassLoader加载一个dex的时候，都会去做一个AOT(Ahead Of Time)操作，在这个操作中Android系统会使用dex2oat去生成一个oat文件，如果没有指定路径的话(基本都是没有指定的，除非手动加载dex的情况)，那么这个oat文件存放的路径是/data/dalvik-cache 目录下，如果你的手机是ARM的话，就应该在/data/dalvik-cache/arm或者arm64下，会有一个data@app@[package name]-1@base.apk@classes.dex的文件，虽然这个文件的后缀是.dex，这个它确实就是我们要找的oat文件，我们可以使用dump命令去把它dump出来，具体的命令如下：

oatdump --oat-file=data@app@[package name]-1@base.apk@classes.dex －－output=dump.txt

然后就可以使用adb pull把对应的dump.txt拉取到你想到存放的路径下了。
<!-- more -->
oat文件的头部包括：

	MAGIC:
	oat
	045
	
	CHECKSUM:
	0xc5344e7d
	
	INSTRUCTION SET:
	Arm64
	
	INSTRUCTION SET FEATURES:
	none
	
	DEX FILE COUNT:
	2
	
	EXECUTABLE OFFSET:
	0x00017000
	
	INTERPRETER TO INTERPRETER BRIDGE OFFSET:
	0x00000000
	
	INTERPRETER TO COMPILED CODE BRIDGE OFFSET:
	0x00000000
	
	JNI DLSYM LOOKUP OFFSET:
	0x00000000
	
	PORTABLE IMT CONFLICT TRAMPOLINE OFFSET:
	0x00000000
	
	PORTABLE RESOLUTION TRAMPOLINE OFFSET:
	0x00000000
	
	PORTABLE TO INTERPRETER BRIDGE OFFSET:
	0x00000000
	
	QUICK GENERIC JNI TRAMPOLINE OFFSET:
	0x00000000
	
	QUICK IMT CONFLICT TRAMPOLINE OFFSET:
	0x00000000
	
	QUICK RESOLUTION TRAMPOLINE OFFSET:
	0x00000000
	
	QUICK TO INTERPRETER BRIDGE OFFSET:
	0x00000000
	
	IMAGE PATCH DELTA:
	0 (0x00000000)
	
	IMAGE FILE LOCATION OAT CHECKSUM:
	0x4ebc29b0
	
	IMAGE FILE LOCATION OAT BEGIN:
	0x70db1000
	
	KEY VALUE STORE:
	dex2oat-cmdline = --zip-fd=11 --zip-location=/data/app/com.sven.myapplication-1/base.apk --oat-fd=12 --oat-location=/data/dalvik-cache/arm64/data@app@com.sven.myapplication-1@base.apk@classes.dex --instruction-set=arm64 --instruction-set-features=default --runtime-arg -Xms64m --runtime-arg -Xmx512m --swap-fd=13
	dex2oat-host = Arm
	image-location = /data/dalvik-cache/arm64/system@framework@boot.art
	pic = false
	
	SIZE:
	198956
	
	OatDexFile:
	.....
	
MAGIC表示魔数，和java文件的魔术是一个道理。

CHECKSUM表示校验和，这个也很好理解。

INSTRUCTION SET表示指令集。

INSTRUCTION SET FEATURES表示指令集功能。

DEX FILE COUNT表示oat文件中的dex文件个数。我们知道oat文件包含着两部分——oat data和oat exec。其中oat data里就有被转换的dex文件，而oat exec中有所转换好的native code。

EXECUTABLE OFFSET表示oat exec段的偏移量，也就是oat data和oat exec分隔的地方。

后面的几个字段都表示ARM虚拟机下的Trampoline机制，可以通过Trampoline从native code跳回到ART虚拟机内。

最后看一下KEY VALUE STORE的值。其中–zip-location表示对应apk文件的路径。–oat-location表示生成的oat文件的路径。image-location表示image文件的路径，具体路径是/data/dalvik-cache/arm/system@framework@boot.art，可以把它看成一个特殊的oat文件，里面包含着android系统的dex和对应的native code，所以取名boot，它的作用基本就是用来缓存，预先编译好framework的代码。在这之后，就是dex文件的内容和对应的native code了。

在ART虚拟机中，一个方法会有两个入口，分别是EntryPointFromInterpreter和EntryPointFromCompiledCode，通过对应的setter去设置。虽然ART虚拟机会有AOT操作，但是我们还是可以手动开启解释器模式，另外一些代码可能没有native code，在这样的情况下，ART就需要使用原来dalvik的那部分逻辑，使用解释模式运行，所以现在就存在四种情况：

解释器->native code，解释器->解释器，native code->解释器，native code->native code。

所以每个方法就需要两个入口，保证解释器和native code都可以调用成功。
