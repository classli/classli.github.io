---
title: app编写注意事项
date: 2017-12-30 14:29:36
tags:
---

1. 对磁盘的随机读写的I/O性能较低，里面存在“写入放大”效应，即将一页数据写入某一数块时，当该块区已存在其他有效数据时，需要先对整个区块进行复制，然后擦除该块内容，最后在写入旧数据和新数据。
	* 当手机长期使用、磁盘空间不足、app触发大量数据读写时，该效应最易出现。
2. 在UNIX中，LD_PRELOAD环境变量可以定义在程序运行前优先加载动态链接库。
3. Native Hook
	* 修改环境变量 LD_PRELOAD
		* 重写系统原有函数，将so库放进环境变量LD_PRELOAD中，在程序调用系统函数时，会先去环境变量里找，这样就会调用重写的系统函数。这种hook会对整个系统都生效，hook数据巨大，容易卡死。
<!-- more -->
	* 修改GOT表
		* 引用外部函数时，在编译时将外部函数的地址以Stub的形式存放在.GOT表中，加载时linker再进行重定位。hook思路就是替换.GOT表中的外部函数地址。
	* Inline Hook.
4. 对于需要多次访问的数据，在第一次取出数据时，将数据放到缓存中，下次直接读取缓存，这样减少对文件的读写操作。（静态变量）
5. 要避免在主线程进行I/O操作。
6. 使用ObjectOutputStream进行序列化时，在其上面封装一个缓冲buffer(如ByteArrayOutputStream)，先将序列化后的信息写到缓存区，后一次性写到磁盘上。原因是因为ObjectOutputStream在保存对象时，每个数据成员会带来一次I/O操作。
7. 读写文件的buffer不宜过小，容易造成读写次数上升，推荐8k，不小于4k。
8. 解压文件，ZipFile适合解压磁盘中的文件或者解压ZIP包中的中间部分文件。解压网络接收的数据或者顺序解压适合ZipInputStream。
9. 使用数据库的主键AUTOINCREMENT会额外增加系统负担，因为此时会自动创建sqlite_sequence表来维护主键。
10. Bitmap解码尽量使用decodeStream，其性能高于decodeFile，decodeResourceStream性能也高于decodeResource.
11. Sharepreference提交尽量使用apply(异步)，避免commit(同步).

12. NDK应该仅在以下两种情况需被使用：1.你有大量C++库被复用；2.所编写的APP是高CPU负责，如游戏引擎、物理模仿等。
13. Activity结束时解除内部类和外部类的引用关系。
14. 不在出现的Activity要最好显示调用finish方法，完成生命周期，确保释放。
15. 功能退出前，结束timer定时器。