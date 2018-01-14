---
title: Dalvik虚拟机
date: 2018-01-14 14:15:13
tags:
---
### Davilk与Java虚拟机区别

Dalvik虚拟机与Java虚拟机最大区别是他们分别具有不同类型的文件格式以及指令集，davilK使用dex(Dalvik Executable）格式的类文件，而Java虚拟机使用的是class格式的类文件，一个dex文件可以包含多个类，多个类中重复的字符串和常数只保存一次，所以文件更小，更适合手机。

Davlik基于寄存器、java虚拟机基于堆栈。java虚拟机占用更多cpu时间、Davlik占用更多空间。
<!-- more -->
Davlik适用于AOT,即在解释运行前先编译成本地代码，java虚拟机适合JIT，即运行时编译。

总结Davlik优点

        1. 将多个类文件收集到同一个dex文件中，以便节省空间；

        2. 使用只读的内存映射方式加载dex文件，以便可以多进程共享dex文件，节省程序加载时间；

        3. 提前调整好字节序（byte order）和字对齐（word alignment）方式，使得它们更适合于本地机器，以便提高指令执行速度；

        4. 尽量提前进行字节码验证（bytecode verification），提高程序的加载速度；

        5. 需要重写字节码的优化要提前进行。

### 内存管理

获得应用分配的堆栈大小方式

	public clss ActivityManager{

	  /**
	     * Return the approximate per-application memory class of the current
	     * device.  This gives you an idea of how hard a memory limit you should
	     * impose on your application to let the overall system work best.  The
	     * returned value is in megabytes; the baseline Android memory class is
	     * 16 (which happens to be the Java heap limit of those devices); some
	     * device with more memory may return 24 or even higher numbers.
	     */
	    public int getMemoryClass() {
	        return staticGetMemoryClass();
	    }
	
    /** @hide */
    static public int staticGetMemoryClass() {
        // Really brain dead right now -- just take this from the configured
        // vm heap size, and assume it is in megabytes and thus ends with "m".
        String vmHeapSize = SystemProperties.get("dalvik.vm.heapgrowthlimit", "");
        if (vmHeapSize != null && !"".equals(vmHeapSize)) {
            return Integer.parseInt(vmHeapSize.substring(0, vmHeapSize.length()-1));
        }
        return staticGetLargeMemoryClass();
     }
	}
	
android触发垃圾回收后输出日志为

	D/dalvikvm(9050): GC_CONCURRENT freed 2049K, 65% free 3571K/9991K, external 4703K/5261K, paused 2ms+2ms 


### 进程和线程
一般来说，虚拟机中的进程和线程与目标机器本地操作系统的进程和线程一一对应。Linux上没有纯粹的线程概念，两个进程共享一个地址空间，即可认为他们是同一进程的两个线程。

**android进程特点**

每个android应用都对应一个davilk虚拟机, android启动后init进程会fork一个Zygote进程，Zygote启动时会创建一个虚拟机，该虚拟机会将所有java核心类库加载起来。当zygote创建一个android进程时，会通过复制自身，即通过fork系统调用实现，这样一方面复制了zygote中的虚拟机，另一方面共享了同一套java核心库，这样应用加载快并节省了内存空间。


### Davlik虚拟机的启动过程

zygote在进程启动时会调用 AndroidRuntime::start函数，如下：

	void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
	{
	   static const String8 startSystemServer("start-system-server");
	   ...
	   const char* rootDir = getenv("ANDROID_ROOT");
	   ...
	   setenv("ANDROID_ROOT", rootDir, 1);
		...
		/* start the virtual machine */
		if (startVm(&mJavaVM, &env, zygote) != 0) {
        	return;
    	}
    	...
    	/*
	     * Register android functions.
	     */
	    if (startReg(env) < 0) {
	        ALOGE("Unable to register all android natives\n");
	        return;
	    }
	    ...
	    /*
	     * Start VM.  This thread becomes the main thread of the VM, and will
	     * not return until the VM exits.
	     */
    	char* slashClassName = toSlashClassName(className);
    	....
    	jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
		...
		env->CallStaticVoidMethod(startClass, startMeth, strArray);
	}

start函数主要工作如下：

1. 调用startVm创建虚拟机，并保存到mJavaVM中。
2. 调用startReg来注册一些Android核心类的JNI方法。
3. 调用className所描述的Java类的main方法，来作为Zygote进程的Java层入口。该class即为ZygoteInit，该类的main会加载大量Android核心类和系统资源文件。

Davlik虚拟机创建实例过程：

	int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote)
	{
		...
		property_get("dalvik.vm.checkjni", propBuf, "");
		...
		property_get("dalvik.vm.execution-mode", propBuf, "");
		...
		parseRuntimeOption("dalvik.vm.usejit", usejitOptsBuf, "-Xusejit:");
		...
		/*
		 * Initialize the VM.
		 *
		 * The JavaVM* is essentially per-process, and the JNIEnv* is per-thread.
		 * If this call succeeds, the VM is ready, and we can start issuing
		 * JNI calls.
		 */
		if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
		    ALOGE("JNI_CreateJavaVM failed\n");
		    return -1;
		}
		return 0;
	}
	
该函数大部分代码是对虚拟机进行配置，如：java jni合法性检测、堆栈大小、最后通过JNI_CreateJavaVM实际创建并初始化虚拟机。

	extern "C" jint JNI_CreateJavaVM(JavaVM** p_vm, JNIEnv** p_env, void* vm_args) {
		...
		const JavaVMInitArgs* args = static_cast<JavaVMInitArgs*>(vm_args);
		....
		
		options.push_back(std::make_pair(std::string(option->optionString), option->extraInfo));
		...
		if (!Runtime::Create(options, ignore_unrecognized)) {
		return JNI_ERR;
		}
		...
		Runtime* runtime = Runtime::Current();
		bool started = runtime->Start();
		...
		*p_env = Thread::Current()->GetJniEnv();
		*p_vm = runtime->GetJavaVM();
		return JNI_OK;
	}
	
该函数根据vm\_args参数，创建出java虚拟机和当前线程的JNI环境，然后通过p\_vm和p\_env返回给调用者。

本地接口表由全局变量gNativeInterface来描述, 它是JNINativeInterface类型的结构体，该结构体中定义这native层调用java层的接口函数，如：FindClass、GetMethodID。

在davlik虚拟机创建好后，startReg来注册Android核心类的JNI方法。

	int AndroidRuntime::startReg(JNIEnv* env) {
		...
		/*
	     * This hook causes all future threads created in this process to be
	     * attached to the JavaVM.  (This needs to go away in favor of JNI
	     * Attach calls.)
	     */
	    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);
	    ....
	    register_jni_procs(gRegJNI, NELEM(gRegJNI), env);
	    ....
	}

1. 通过调用androidSetCreateThreadFunc创建了一个钩子函数，当创建一个Native线程时，javaCreateThreadEtc会被用来初始化该线程的JNI环境。
2. 调用register_jni_procs来注册Android核心类的JNI方法。

Davlik启动过程主要是完成了一下四件事情：

1. 创建了一个Dalvik虚拟机实例；

2. 加载了Java核心类及其JNI方法；

3. 为主线程的设置了一个JNI环境；

4. 注册了Android核心类的JNI方法。

### Davlik虚拟机的运行过程

经过JNI_CreateJavaVM调用后，就获得了Zygote的Davlik实例JavaVM，以及Zygote进程的主线程的JNI环境JNIEnv。紧接着Zygote进程就会通过JNIEnv的CallStaticVoidMethod调用ZygoteInit的main函数，依次作为Java代码的入口函数。

jni.h中的CallStaticVoidMethod如下：

	void CallStaticVoidMethod(jclass clazz, jmethodID methodID, ...)
	    {
	        va_list args;
	        va_start(args, methodID);
	        functions->CallStaticVoidMethodV(this, clazz, methodID, args);
	        va_end(args);
	    }
	    
functions 指向JNINativeInterface结构体，调用的是他的成员变量CallStaticVoidMethodV（函数指针），该成员变量指向的同名函数CallStaticVoidMethodV，通过一系列调用，期间会判断要调用的方法是JNI方法还是JAVA方法，是JNI方法就直接执行，是JAVA方法就需要通过解释器来执行，Dalvik针对每个平台都有一个优化后的解释器也有一个通用解释器。解释器支持调试状态。

### Davlik虚拟机JNI方法的注册过程

JNI函数是在操作系统本地执行的，不是由Davlik虚拟机。JNI方法在能够被调用之前，首先需要加载到当前进程的地址空间里来，加载方法为`System.loadLibrary("libxxx"); `

	public final class System {
		public static void loadLibrary(String libname) {
	        Runtime.getRuntime().loadLibrary0(VMStack.getCallingClassLoader(), libname);
	    }
	}

System.loadLibrary通过调用Runtime的loadLibrary0进行实际加载。

	public class Runtime{
		synchronized void loadLibrary0(ClassLoader loader, String libname) {
			if (loader != null) {
	            ....
	            String filename = loader.findLibrary(libraryName);
	            ....
	            String error = doLoad(filename, loader);
	            ....
	            return;
	       }
	       String filename = System.mapLibraryName(libraryName);
	       for (String directory : mLibPaths) { 
	       	String candidate = directory + filename; 
	       	....
	       	String error = doLoad(candidate, loader);
	       	....
	       }
		}
	}
	
每一个类都有对应的类加载器，在Davlik虚拟机中，类加载器不仅知道所加载的类的路径，还知道所属APK存放so文件的路径。loadLibrary0里又通过doLoad进行加载，而doload又调用了nativeLoad方法，该方法为native方法实际调用的是Runtime.c中的Runtime_nativeLoad。

	Runtime_nativeLoad(JNIEnv* env, jclass ignored, jstring javaFilename,
	                   jobject javaLoader, jstring javaLibrarySearchPath)
	{
	    return JVM_NativeLoad(env, javaFilename, javaLoader, javaLibrarySearchPath);
	}
	
JVM\_NativeLoad里实际调用LoadNativeLibrary，该方法在java\_vm_ext.cc文件里。

	bool JavaVMExt::LoadNativeLibrary(JNIEnv* env,
	                                  const std::string& path,
	                                  jobject class_loader,
	                                  jstring library_path,
	                                  std::string* error_msg) {
	                                  
	   // See if we've already loaded this library.  If we have, and the class loader
	  // matches, return successfully without doing anything.
	  // TODO: for better results we should canonicalize the pathname (or even compare
	  // inodes). This implementation is fine if everybody is using System.loadLibrary.
	  ....
	  {
	    // TODO: move the locking (and more of this logic) into Libraries.
	    MutexLock mu(self, *Locks::jni_libraries_lock_);
	    library = libraries_->Get(path);
	  }
	  ....
	  if (library != nullptr) {
	    // Use the allocator pointers for class loader equality to avoid unnecessary weak root decode.
	    if (library->GetClassLoaderAllocator() != class_loader_allocator) {
	      // The library will be associated with class_loader. The JNI
	      // spec says we can't load the same library into more than one
	      // class loader.
	      StringAppendF(error_msg, "Shared library \"%s\" already opened by "
	          "ClassLoader %p; can't open in ClassLoader %p",
	          path.c_str(), library->GetClassLoader(), class_loader);
	      LOG(WARNING) << error_msg;
	      return false;
	    }
	    VLOG(jni) << "[Shared library \"" << path << "\" already loaded in "
	              << " ClassLoader " << class_loader << "]";
	    if (!library->CheckOnLoadResult()) {
	      StringAppendF(error_msg, "JNI_OnLoad failed on a previous attempt "
	          "to load \"%s\"", path.c_str());
	      return false;
	    }
	    return true;
	  }
	  ....
	  void* handle = android::OpenNativeLibrary(env,
	                                            runtime_->GetTargetSdkVersion(),
	                                            path_str,
	                                            class_loader,
	                                            library_path);
	  ....                                          
	  // Create SharedLibrary ahead of taking the libraries lock to maintain lock ordering.
    std::unique_ptr<SharedLibrary> new_library(
        new SharedLibrary(env, self, path, handle, class_loader, class_loader_allocator));
	  ....
	  sym = library->FindSymbol("JNI_OnLoad", nullptr);                                          
	  ....
	}

该函数首先检查这个so文件是否被加载过，如果被加载过且以前的类加载器和本次的不同或者以前加载失败，则返回false表示不能在本进程加载该so文件。

如果该so没有被加载过，则调用OpenNativeLibrary将该文件加载到当前进程，返回对应文案的句柄handle，然后还会调用所加载so文件中的JNI_OnLoad方法注册其中的JNI函数。

JNI_OnLoad方法中通过调用 jniRegisterNativeMethods 进行注册。

	extern "C" int jniRegisterNativeMethods(C_JNIEnv* env, const char* className,
	const JNINativeMethod* gMethods, int numMethods)
	{
		 ....
		 scoped_local_ref<jclass> c(env, findClass(env, className));
		 ....
		 (*env)->RegisterNatives(e, c.get(), gMethods, numMethods)
		 ....
	}
从上面的代码可以看出实际又调用了JNIEnv结构体的成员函数RegisterNatives，而RegisterNatives 又调用了Dalvik虚拟机内部定义的函数RegisterNatives。

JNI注册其实就是JNI函数的函数地址和调用JNI函数的Bridge函数绑定到method方法里的某些字段上，这就让Java方法和JNI函数产生了联系。

### Dalvik虚拟机进程和线程的创建过程

#### Dalvik虚拟机进程的创建过程

Dalvik虚拟机进程的创建即为android应用进程的创建，最终会通过Zygote类的forkAndSpecialize方法进行创建。forkAndSpecialize主要是调用了nativeForkAndSpecialize方法，该方法是一个JNI方法，是由C++层的com\_android\_internal\_os\_Zygote\_nativeForkAndSpecialize进行具体实现的。


	static jint com_android_internal_os_Zygote_nativeForkAndSpecialize(
	        JNIEnv* env, jclass, jint uid, jint gid, jintArray gids,
	        jint debug_flags, jobjectArray rlimits,
	        jint mount_external, jstring se_info, jstring se_name,
	        jintArray fdsToClose, jstring instructionSet, jstring appDataDir) {
	 ....
	 return ForkAndSpecializeCommon(env, uid, gid, gids, debug_flags,
	            rlimits, capabilities, capabilities, mount_external, se_info,
	            se_name, false, fdsToClose, instructionSet, appDataDir);       
	}

最终是调用了ForkAndSpecializeCommon

	static pid_t ForkAndSpecializeCommon(JNIEnv* env, uid_t uid, gid_t gid, jintArray javaGids,
	                                     jint debug_flags, jobjectArray javaRlimits,
	                                     jlong permittedCapabilities, jlong effectiveCapabilities,
	                                     jint mount_external,
	                                     jstring java_se_info, jstring java_se_name,
	                                     bool is_system_server, jintArray fdsToClose,
	                                     jstring instructionSet, jstring dataDir) {
		.....
		pid_t pid = fork();
		if (pid == 0) {
			 ....
			 bool use_native_bridge = !is_system_server && (instructionSet != NULL)
	        && android::NativeBridgeAvailable();
		    if (use_native_bridge) {
		      ScopedUtfChars isa_string(env, instructionSet);
		      use_native_bridge = android::NeedsNativeBridge(isa_string.c_str());
		    }
	    	 .....
			 if (!is_system_server) {
			    int rc = createProcessGroup(uid, getpid());
			    .....
		    }
			SetGids(env, javaGids);
			if (use_native_bridge) {
		      ......
		      android::PreInitializeNativeBridge(data_dir.c_str(), isa_string.c_str());
	    	}
	    	rc = setresuid(uid, uid, uid);
	    	....
	    	rc = selinux_android_setcontext(uid, is_system_server, se_info_c_str, se_name_c_str);
	    	....
	    	SetThreadName(se_name_c_str);
	    	....
	    	env->CallStaticVoidMethod(gZygoteClass, gCallPostForkChildHooks, debug_flags,
                              is_system_server, instructionSet);
	    	....
	    	return pid;
		}                                     
	}
	
	
函数ForkAndSpecializeCommon除了创建应用进程还创建系统进程，当is\_system\_server为true即为系统进程时，会调用createProcessGroup设置该进程的uid和groupId，以获得某些特权。

通过 fork创建出来新的子进程，然后对子进程进行相应的设置。

经过上面处理一个Davlik虚拟机进程就创建出来了，实际上就是一个Linux进程。

#### Dalvik虚拟机线程的创建过程

Dalivk线程是通过Thread类的start方法来创建。

	public class Thread implements Runnable {
		public synchronized void start() {
		 ....
		 nativeCreate(this, stackSize, daemon);
		 ....
		}
	}
	
start方法通过调用nativeCreate这个JNI方法实现，经过中间的一些调用最后运行到了CreateNativeThread函数。

	void Thread::CreateNativeThread(JNIEnv* env, jobject java_peer, size_t stack_size, bool is_daemon) {
		....
		thread_name = java_name->ToModifiedUtf8();
		...
		Runtime* runtime = Runtime::Current();
		....
		// Atomically start the birth of the thread ensuring the runtime isn't shutting down.
	    bool thread_start_during_shutdown = false;
	    ....
	    Thread* child_thread = new Thread(is_daemon);
	    ....
	    stack_size = FixStackSize(stack_size);
	    ....
	  	// Try to allocate a JNIEnvExt for the thread. We do this here as we might be out of memory and
	    // do not have a good way to report this on the child's side.
	    std::unique_ptr<JNIEnvExt> child_jni_env_ext(
	      JNIEnvExt::Create(child_thread, Runtime::Current()->GetJavaVM()));
	    if (child_jni_env_ext.get() != nullptr) {
	    	....
	    	CHECK_PTHREAD_CALL(pthread_attr_setstacksize, (&attr, stack_size), stack_size);
	   		pthread_create_result = pthread_create(&new_pthread,
	                                           &attr,
	                                           Thread::CreateCallback,
	                                           child_thread);
	    }            
	    if (pthread_create_result == 0) {
	      // pthread_create started the new thread. The child is now responsible for managing the
	      // JNIEnvExt we created.
	      // Note: we can't check for tmp_jni_env == nullptr, as that would require synchronization
	      //       between the threads.
	      child_jni_env_ext.release();
	      return;
	    }
	    ....
	}
	
stack\_size表示要创建的线程栈大小，如果没指定则用默认大小。Dalvik虚拟机线程实际上具有两个栈，一个是Java栈，另一个是Native栈。Native栈是在调用Native代码时使用的，它是由操作系统来管理的，而Java栈是由Dalvik虚拟机来管理的。上述的child_thread指针中即保存着native层的Thread对象。

然后调用pthread\_create来创建一个线程，pthread_create实际由pthread库提供的一个函数，他是通过系统调用clone来请求内核创建一个线程的。由此就可以看出，Dalvik虚拟机线程实际上就是本地操作系统线程。

CreateCallback是创建的线程的入口函数，即新创建的Davlik虚拟机线程启动入口。

	void* Thread::CreateCallback(void* arg) {
		Thread* self = reinterpret_cast<Thread*>(arg);
		...
		// Invoke the 'run' method of our java.lang.Thread.
		mirror::Object* receiver = self->tlsPtr_.opeer;
		jmethodID mid = WellKnownClasses::java_lang_Thread_run;
		ScopedLocalRef<jobject> ref(soa.Env(), soa.AddLocalReference<jobject>(receiver));
		InvokeVirtualOrInterfaceWithJValues(soa, ref.get(), mid, nullptr);
		...
	}
	
该函数首先会对新键的线程进行初始化，然后调用 Thread.run()方法，最后run方法返回后会对线程进清理。

#### 只执行C/C++代码的Native线程的创建过程

从C++类Threads.cpp的成员函数run开始分析Dalvik虚拟机中的Native线程的创建过程。

	status_t Thread::run(const char* name, int32_t priority, size_t stack)
	{
		.....
		if (mCanCallJava) {
		    res = createThreadEtc(_threadLoop,
		            this, name, priority, stack, &mThread);
		} else {
		    res = androidCreateRawThreadEtc(_threadLoop,
		            this, name, priority, stack, &mThread);
		}
		.....
	}
	
如果当前线程可以调用java方法则调用createThreadEtc，否则调用androidCreateRawThreadEtc方法。

	int androidCreateRawThreadEtc(android_thread_func_t entryFunction,
	                               void *userData,
	                               const char* threadName __android_unused,
	                               int32_t threadPriority,
	                               size_t threadStackSize,
	                               android_thread_id_t *threadId)
	{
		pthread_attr_t attr;
		pthread_attr_init(&attr);
		pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
		.....
		int result = pthread_create(&thread, &attr,
		                    (android_pthread_entry)entryFunction, userData);
		pthread_attr_destroy(&attr);
		....
	}


androidCreateRawThreadEtc里面是通过pthread库提供的pthread\_create来创建线程的，entryFunction为线程入口函数，entryFunction实际为传入的_threadLoop。

	int Thread::_threadLoop(void* user)
	{
		Thread* const self = static_cast<Thread*>(user);
		
		sp<Thread> strong(self->mHoldSelf);
		wp<Thread> weak(strong);
		self->mHoldSelf.clear();
		....
		do {
			if (first) {
			    first = false;
			    self->mStatus = self->readyToRun();
			    result = (self->mStatus == NO_ERROR);
			
			    if (result && !self->exitPending()) {
			    	result = self->threadLoop();
			    }
			} else {
			    result = self->threadLoop();
			}
			if (result == false || self->mExitPending) {
				....
				break;
			}
			.....
			// Release our strong reference, to let a chance to the thread
			// to die a peaceful death.
			strong.clear();
			// And immediately, re-acquire a strong reference for the next loop
			strong = weak.promote();
		} while(strong != 0);
		....
	}
	
_threadLoop不断地执行Thread对象的成员函数threadLoop，来作为前面所创建的Native线程的执行主体, 直到满足一下三个条件之一。

1. Thread对象self的成员函数threadLoop的返回值等于false。
2. threadLoop在执行过程中，调用了requestExit请求前面的Native退出，此时mExitPending=true。
3. Thread对象被销毁。

#### 能同时执行C/C++代码和Java代码的Native线程的创建过程

如上面所述，此时创建线程调用的是createThreadEtc， 其内部又是调用了androidCreateThreadEtc，androidCreateThreadEtc又调用gCreateThreadFn所指向的函数来创建线程。gCreateThreadFn默认指向androidCreateRawThreadEtc，在Zygote启动过程中会将改指针重新设为javaCreateThreadEtc。所以实际上是调用javaCreateThreadEtc来创建线程。

	int AndroidRuntime::javaCreateThreadEtc(
	                                android_thread_func_t entryFunction,
	                                void* userData,
	                                const char* threadName,
	                                int32_t threadPriority,
	                                size_t threadStackSize,
	                                android_thread_id_t* threadId) {
	                                
	                                
	....
	result = androidCreateRawThreadEtc(AndroidRuntime::javaThreadShell, args,
	        threadName, threadPriority, threadStackSize, threadId);
	....                                
	}
	
该函数调用了上述Threads.cpp中的androidCreateRawThreadEtc，实际调用的也是pthread_create来创建线程。与上面的主要区别是回调函数不同。
	 
	int AndroidRuntime::javaThreadShell(void* args) {
		.....
		/* hook us into the VM */
		if (javaAttachThread(name, &env) != JNI_OK)
		    return -1;
		/* start the thread running */
		result = (*(android_thread_func_t)start)(userData);
		
		/* unhook us */
		javaDetachThread();
		free(name);
		....

	}
	
该函数主要执行以下三个操作：

1. 调用javaAttachThread将当前线程附加到当前进程的虚拟机中去，使得当前线程不仅可以执行Native代码也能执行Java代码。
2. 调用start所指向的函数主体，即_threadLoop。
3. 在_threadLoop返回后，调用javaDetachThread将当前线程从虚拟机中移除。

javaAttachThread实际上就是为当前线程建立以JNI上下文环境，其创建的JNIEnvExt结构体中的funcTable指向了一系列Davlik虚拟机回调函数，正是有了这些回调函数当前线程才能访问Davlik虚拟机中的JAVA代码。
