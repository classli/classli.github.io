---
title: Android运行时ART分析
date: 2017-12-31 15:06:30
tags:
---
ART是一个执行本地机器指令的虚拟机：ART除了实现Java虚拟机接口之外，其内部还有垃圾收集机制，同时还有Java核心类库调用。

ART模式下APK在安装时就执行了AOT(即在安装时对dex文件进行了编译)，ART通过LLVM将Dex字节码编译成机器码，LLVM是一个用来快速开发自己编译器的框架结构。

在APK安装时，PackageManagerService服务通过守护进程installd调用工具dex2oat对Dex进行翻译，该工具就是基于LLVM，翻译后得到ELF格式的oat文件，该文件依然以.odex结尾，并也保存在/data/dalvik-cache目录中。
<!-- more -->

编译出来的ELF文件有两个特殊字段，oatdata和oatexec，分别用来储存原来打包在APK里面的dex文件和翻译这个dex文件得到本地机器指令。

在oat文件的动态段（dymanic section）中，还导出了三个符号oatdata、oatexec和oatlastword，分别用来描述oatdata和oatexec段加段到内存后的起止地址。在oatdata字段中有个OatClass结构体，它指向了相关类对应的机器码在oatexec字段中的位置。

ART的运行原理总结如下：

1. 在Android系统启动时，Zygote进程利用ART运行时导出的Java虚拟机接口创建ART虚拟机。
2. APK在安装的时候，classes.dex文件会被工具dex2oat翻译成本地机器指令，最终得到一个ELF格式的oat文件。
3. APK运行时，上述生成的oat文件会被加载到内存中，并且ART虚拟机可以通过里面的oatdata和oatexec字段找到任意一个类的方法对应的本地机器指令来执行。

在AndroidRuntime类的成员函数start中，ART虚拟机创建和初始化完成后，Zygote进程就会通过它导出的JNI接口CallStaticVoidMetho，使得它以指定的类方法为入口正式进入运行状态，即开始运行ZygoteInit的main函数。

### ART加载OAT文件过程分析

OAT文件本质上是ELF文件，具有ELF文件的一般格式。OAT的生成是通过dex2oat工具。在commands.cpp中通过run_dex2oat调用起该工具。

	static void run_dex2oat(int zip_fd, int oat_fd, int image_fd, const char* input_file_name,
	        const char* output_file_name, int swap_fd, const char *instruction_set,
	        const char* compiler_filter, bool vm_safe_mode, bool debuggable, bool post_bootcomplete,
	        int profile_fd, const char* shared_libraries) {
	        .....
	        static const char* DEX2OAT_BIN = "/system/bin/dex2oat";
	        .....
	        char zip_fd_arg[strlen("--zip-fd=") + MAX_INT_LEN];
		    char zip_location_arg[strlen("--zip-location=") + PKG_PATH_MAX];
		    char oat_fd_arg[strlen("--oat-fd=") + MAX_INT_LEN];
		    .....
		    int i = 0;
		    argv[i++] = DEX2OAT_BIN;
		    argv[i++] = zip_fd_arg;
		    argv[i++] = zip_location_arg;
		    ....
		    execv(DEX2OAT_BIN, (char * const *)argv);
	}
	
其中，参数zip\_fd和oat\_fd都是打开文件描述符，指向的分别是正在安装的APK文件和要生成的OAT文件。当我们选择了ART运行时时，Zygote进程在启动的过程中，会调用libart.so里面java\_vm\_ext.cpp里面的函数JNI_CreateJavaVM来创建一个ART虚拟机。

	extern "C" jint JNI_CreateJavaVM(JavaVM** p_vm, JNIEnv** p_env, void* vm_args) {
		  ....
		  if (!Runtime::Create(options, ignore_unrecognized)) {
		    return JNI_ERR;
		  }
		  // Initialize native loader. This step makes sure we have
		  // everything set up before we start using JNI.
		  android::InitializeNativeLoader();
		
		  Runtime* runtime = Runtime::Current();
		  bool started = runtime->Start();
		  if (!started) {
		    delete Thread::Current()->GetJniEnv();
		    delete runtime->GetJavaVM();
		    LOG(WARNING) << "CreateJavaVM failed";
		    return JNI_ERR;
		  }
		
		  *p_env = Thread::Current()->GetJniEnv();
		  *p_vm = runtime->GetJavaVM();
		  return JNI_OK;
	}
	

Runtime的Create负责创建ART虚拟机，然后通过start启动虚拟机，创建ART虚拟的动作只会在Zygote进程中执行。ART创建过程如下：

	bool Runtime::Create(RuntimeArgumentMap&& runtime_options) {
	  // TODO: acquire a static mutex on Runtime to avoid racing.
	  if (Runtime::instance_ != nullptr) {
	    return false;
	  }
	  instance_ = new Runtime;
	  if (!instance_->Init(std::move(runtime_options))) {
	    // TODO: Currently deleting the instance will abort the runtime on destruction. Now This will
	    // leak memory, instead. Fix the destructor. b/19100793.
	    // delete instance_;
	    instance_ = nullptr;
	    return false;
	  }
	  return true;
	}

可见ART虚拟机就是当前Runtime的一个实例。在Init中会初始化ART虚拟机堆，并将当前线程设置为ART虚拟机的主线程，同时也创建了JavaVMExt实例，调用者可以通过该实例与ART虚拟机进行交互，此外，Init还会调用dex2oat生成/data/dalvik-cache/system@framework@boot.art@classes.oat文件。

ART运行时通过OatFile的Open函数加载oat文件。

	OatFile* OatFile::Open(const std::string& filename,
	                       const std::string& location,
	                       uint8_t* requested_base,
	                       uint8_t* oat_file_begin,
	                       bool executable,
	                       bool low_4gb,
	                       const char* abs_dex_location,
	                       std::string* error_msg) {
	                       
	  ....
	  // Try dlopen first, as it is required for native debuggability. This will fail fast if dlopen is
	  // disabled.
	  OatFile* with_dlopen = OatFileBase::OpenOatFile<DlOpenOatFile>(filename,
	                                                                 location,
	                                                                 requested_base,
	                                                                 oat_file_begin,
	                                                                 false,
	                                                                 executable,
	                                                                 low_4gb,
	                                                                 abs_dex_location,
	                                                                 error_msg);
	    ....
	    // If we aren't trying to execute, we just use our own ElfFile loader for a couple reasons:
		  //
		  // On target, dlopen may fail when compiling due to selinux restrictions on installd.
		  //
		  // We use our own ELF loader for Quick to deal with legacy apps that
		  // open a generated dex file by name, remove the file, then open
		  // another generated dex file with the same name. http://b/10614658
		  //
		  // On host, dlopen is expected to fail when cross compiling, so fall back to OpenElfFile.
		  //
		  //
		  // Another independent reason is the absolute placement of boot.oat. dlopen on the host usually
		  // does honor the virtual address encoded in the ELF file only for ET_EXEC files, not ET_DYN.
		  OatFile* with_internal = OatFileBase::OpenOatFile<ElfOatFile>(filename,
		                                                                location,
		                                                                requested_base,
		                                                                oat_file_begin,
		                                                                false,
		                                                                executable,
		                                                                low_4gb,
		                                                                abs_dex_location,
		                                                                error_msg);
		  return with_internal;
                                                                                       
	}
	
参数requested_base是一个可选参数，用来描述要加载的OAT文件里面的oatdata段要加载在的位置。参数executable表示要加载的OAT是不是应用程序的主执行文件。应用程序也可以在运行时动态加载DEX文件。这些动态加载的DEX文件在加载的时候同样会被翻译成OAT再运行，它们相对于打包在应用程序的classes.dex文件来说，就不属于主执行文件了。第一个OpenOatFile是在编译器为Portable模式下，该模式下需要依靠动态链接器来加载oat文件，实现函数间的依赖；第二个OpenOatFile是在编译器为Quick模式下，直接加载oat文件，后续通过线程的TLS（线程本地区域）提供的函数表实现函数间的依赖。

	template <typename kOatFileBaseSubType>
	OatFileBase* OatFileBase::OpenOatFile(const std::string& elf_filename,
	                                      const std::string& location,
	                                      uint8_t* requested_base,
	                                      uint8_t* oat_file_begin,
	                                      bool writable,
	                                      bool executable,
	                                      bool low_4gb,
	                                      const char* abs_dex_location,
	                                      std::string* error_msg) {
	  std::unique_ptr<OatFileBase> ret(new kOatFileBaseSubType(location, executable));
	
	  ret->PreLoad();
	
	  if (!ret->Load(elf_filename,
	                 oat_file_begin,
	                 writable,
	                 executable,
	                 low_4gb,
	                 error_msg)) {
	    return nullptr;
	  }
	
	  if (!ret->ComputeFields(requested_base, elf_filename, error_msg)) {
	    return nullptr;
	  }
	
	  ret->PreSetup(elf_filename);
	
	  if (!ret->Setup(abs_dex_location, error_msg)) {
	    return nullptr;
	  }
	
	  return ret.release();
	}
	
通过调用Load进行oat文件的实际加载，如果是Portable模式, 会调用DlOpenOatFile里的Load，最终会在Dlopen中调用dlopen通过动态链接器加载文件到内存，如果是Quick模式，则调用ElfOatFile下的Load，最终调用ElfFileOpen函数通过Load将文件加入内存。文件载入后通过ComputeFields确定文件中的某些字段。
	
	bool OatFileBase::ComputeFields(uint8_t* requested_base,
	                                const std::string& file_path,
	                                std::string* error_msg) {
	                                
		....
		begin_ = FindDynamicSymbolAddress("oatdata", &symbol_error_msg);
		....
		end_ = FindDynamicSymbolAddress("oatlastword", &symbol_error_msg);
		...
		// Readjust to be non-inclusive upper bound.
		end_ += sizeof(uint32_t);
		....	
		return true;	                                
	}
	
ComputeField主要确定的是oatdata字段的开始位置和oatexec的结束位置。

Setup函数用来解析oatdata中dex文件相关信息，如DEX文件内容在oatdata段的偏移或者类方法的机器指令的偏移量，最后每一个DEX文件的信息都被封装在一个OatDexFile对象中，以便后续访问。对于每一个类的描述信息都通过一个OatClass对象来描述。

调用OatDexFile类的成员函数GetOatClass可以获得指定类的所有方法的本地机器指令信息：

	OatFile::OatClass OatFile::OatDexFile::GetOatClass(uint16_t class_def_index) const {
		uint32_t oat_class_offset = GetOatClassOffset(class_def_index);
		.....
		return OatFile::OatClass(oat_file_,
		                           status,
		                           type,
		                           bitmap_size,
		                           reinterpret_cast<const uint32_t*>(bitmap_pointer),
		                           reinterpret_cast<const OatMethodOffsets*>(methods_pointer));
	}
	
通过GetOatMethod即可获得指定类方法的本地机器指令信息，该方法返回OatMethod对象，OatMethod对象里包含了两个参数begin_和code_offset_， 参数begin_描述的是OAT文件的OAT头在内存的位置，而参数code\_offset_描述的是类方法的本地机器指令相对OAT头的偏移位置。将这两者相加，就可以得到一个类方法的本地机器指令在内存的位置。

### ART加载类和方法过程分析

ART运行时的入口是com.android.internal.os.ZygoteInit类的静态成员函数main，在AndroidRuntime::start函数启动了此方法。	

	void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
	{
		JNIEnv* env;
		if (startVm(&mJavaVM, &env, zygote) != 0) {
		    return;
		}
		onVmCreated(env);
		/*
		 * Start VM.  This thread becomes the main thread of the VM, and will
		 * not return until the VM exits.
		 */
		char* slashClassName = toSlashClassName(className);
		jclass startClass = env->FindClass(slashClassName);
		if (startClass == NULL) {
		    ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
		    /* keep going */
		} else {
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
	
首先通过调用startVm来创建一个java虚拟机mJavaVM，及其JNI接口env，这个java虚拟机就是ART运行时。接下来，就通过分析JNI接口FindClass和GetStaticMethodID的实现，以便理解ART运行时是如何查找到指定的类和方法的。

JNI函数表实际上是由JNI类的静态成员函数组成的，JNI接口FindClass和GetStaticMethodID分别是由JNI类的静态成员函数FindClass和GetStaticMethodID实现, 该方法在文件jni_internal.cc中。

	static jclass FindClass(JNIEnv* env, const char* name) {
	    CHECK_NON_NULL_ARGUMENT(name);
	    Runtime* runtime = Runtime::Current();
	    ClassLinker* class_linker = runtime->GetClassLinker();
	    std::string descriptor(NormalizeJniClassDescriptor(name));
	    ScopedObjectAccess soa(env);
	    mirror::Class* c = nullptr;
	    if (runtime->IsStarted()) {
	      StackHandleScope<1> hs(soa.Self());
	      Handle<mirror::ClassLoader> class_loader(hs.NewHandle(GetClassLoader(soa)));
	      c = class_linker->FindClass(soa.Self(), descriptor.c_str(), class_loader);
	    } else {
	      c = class_linker->FindSystemClass(soa.Self(), descriptor.c_str());
	    }
	    return soa.AddLocalReference<jclass>(c);
	  }
	  
ART虚拟机通过Current获得一个Runtime单列，ClassLinker对象用来加载和链接类方法。如果IsStarted已经启动了，就通过GetClassLoader来获得类加载器，并通过ClassLinker的FindClass来加载类，否则调用FindSystemClass加载系统类。

在ClassLinker的FindClass里,如果传入的class_loader为空，则通过查找该类所在的Dex文件，并通过DefineClass从Dex文件中加载该类；如果不为空，那么就通过指定的类加载器加载相关类，通过调用这个java.lang.ClassLoader类的成员函数loadClass即可加载指定的类。

系统类加载器在加载系统类实际上也是通过JNI方法调用ClassLinker类的成员函数FindClass来实现的，只不过传入的class_loader为null，要加载的com.android.internal.os.ZygoteInit就属于系统类，ClassLinker 类成员变量boot\_class\_path_里存储着一系列系统Dex文件对象DexFile，通过变量这个变量，即可找到类对应的Dex文件。找到Dex文件之后通过DefineClass加载相关类。

DefineClass里通过调用SetupClass 和 LoadClass来加载具体的类。
	
	mirror::Class* ClassLinker::DefineClass(Thread* self,
	                                        const char* descriptor,
	                                        size_t hash,
	                                        Handle<mirror::ClassLoader> class_loader,
	                                        const DexFile& dex_file,
	                                        const DexFile::ClassDef& dex_class_def) {
	                                        ....
	                                        SetupClass(dex_file, dex_class_def, klass, class_loader.Get());
	                                        ....
	                                        // Add the newly loaded class to the loaded classes table.
	                                        mirror::Class* existing = InsertClass(descriptor, klass.Get(), hash);
	                                        if (existing != nullptr) {
		                                        // We failed to insert because we raced with another thread. Calling EnsureResolved may cause
		                                        // this thread to block.
		                                        return EnsureResolved(self, descriptor, existing);
										    }
										    // Load the fields and other things after we are inserted in the table. This is so that we don't
										    // end up allocating unfree-able linear alloc resources and then lose the race condition. The
										    // other reason is that the field roots are only visited from the class table. So we need to be
										    // inserted before we allocate / fill in these fields.
										    LoadClass(self, dex_file, dex_class_def, klass);  
										    ....
										    if (!LinkClass(self, descriptor, klass, interfaces, &h_new_class)) {
											    // Linking failed.
											    if (!klass->IsErroneous()) {   
											   		mirror::Class::SetStatus(klass, mirror::Class::kStatusError, self);
											    }
											    return nullptr;    
										    }
										    ....                        
	                                        }
	                                        
对类进行加载和解析的过程中，会对类状态SetStatus进行实时更新。SetupClass 和 LoadClass主要完成以下工作：

1. 将参数class_loader描述的ClassLoader设置到klass描述的Class对象中去，即给每一个已加载类关联一个类加载器。

2. 通过DexFile类的成员函数GetIndexForClassDef获得正在加载的类在DEX文件中的类索引号，并且设置到klass描述的Class对象中去。这个类索引号是一个很重要的信息，因为我们需要通过类索引号在相应的OAT文件找到一个OatClass结构体。有了这个OatClass结构体之后，我们才可以找到类方法对应的本地机器指令。

3. 从参数dex_file描述的DEX文件中获得正在加载的类的静态成员变量和实例成员变量个数，并且为每一个静态成员变量和实例成员变量都分配一个ArtField对象，接着通过ClassLinker类的成员函数LoadField对这些ArtField对象进行初始化。初始好得到的ArtField对象全部保存在klass描述的Class对象中。

4. 调用ClassLinker类的成员函数GetOatClass，从相应的OAT文件中找到与正在加载的类对应的一个OatClass结构体oat_class。这需要利用到上面提到的DEX类索引号，这是因为DEX类和OAT类根据索引号存在一一对应关系。

5.  从参数dex_file描述的DEX文件中获得正在加载的类的直接成员函数和虚拟成员函数个数，并且为每一个直接成员函数和虚拟成员函数都分配一个ArtMethod对象，接着通过ClassLinker类的成员函数LoadMethod对这些ArtMethod对象进行初始化。初始好得到的ArtMethod对象全部保存在klass描述的Class对象中。

6.  每一个直接成员函数和虚拟成员函数都对应有一个函数索引号。根据这个函数索引号可以在第4步得到的OatClass结构体中找到对应的本地机器指令，所有与这些成员函数关联的本地机器指令信息通过全局函数LinkCode设置到klass描述的Class对象中。

总结来说，参数klass描述的Class对象包含了一系列的ArtField对象和ArtMethod对象，其中，ArtField对象用来描述成员变量信息，而ArtMethod用来描述成员函数信息。
	
	void ClassLinker::LinkCode(ArtMethod* method, const OatFile::OatClass* oat_class,
	                           uint32_t class_def_method_index) {
									....
									if (oat_class != nullptr) {
									// Every kind of method should at least get an invoke stub from the oat_method.
									// non-abstract methods also get their code pointers.
									const OatFile::OatMethod oat_method = oat_class->GetOatMethod(class_def_method_index);
									oat_method.LinkMethod(method);
									}
									....
	                           
	                           }
	                           
参数method表示要设置本地机器指令的类方法，参数oat\_class表示类方法method在OAT文件中对应的OatClass结构体，参数method\_index表示类方法method的索引号。通过method\_index描述的索引号可以在oat\_class表示的OatClass结构体中找到一个OatMethod结构体oat_method，这个OatMethod结构描述了类方法method的本地机器指令相关信息。然后通过LinkMethod将这些信息设置到参数method描述的ArtMethod对象中去。

LinkMethod 通过调用SetEntryPointFromQuickCompiledCode将OatMethod结构体中的code\_offset_字段设置到ArtMethod中（对应entry\_point\_from\_quick\_compiled\_code_字段)，该字段指向了一个本地机器指令函数。

>调用Runtime类的静态成员函数Current获得的是描述ART运行时的一个Runtime对象。调用这个Runtime对象的成员函数GetInstrumentation获得的是一个Instrumentation对象。这个Instrumentation对象是用来调试ART运行时的，通过调用它的成员函数InterpretOnly可以知道ART虚拟机是否运行在解释模式中.

加载完成后，得到的是一个Class对象，这个Class对象关联有一系列的ArtField对象和ArtMethod对象。其中，ArtField对象描述的是成员变量，而ArtMethod对象描述的是成员函数。对于每一个ArtMethod对象，它都有一个解释器入口点和一个本地机器指令入口点。这样，无论一个类方法是通过解释器执行，还是直接以本地机器指令执行，我们都可以以统一的方式来进行调用。同时，理解了上述的类加载过程后，我们就可以知道，我们在Native层通过JNI接口FindClass查找或者加载类时，得到的一个不透明的jclass值，实际上指向的是一个Class对象。

下面在看看方法的查找。对于JNI类的静态成员函数GetStaticMethodID通过调用一个全局函数FindMethodID来查找指定的类：

	static jmethodID FindMethodID(ScopedObjectAccess& soa, jclass jni_class,
	                              const char* name, const char* sig, bool is_static)
	    SHARED_REQUIRES(Locks::mutator_lock_) {
	  mirror::Class* c = EnsureInitialized(soa.Self(), soa.Decode<mirror::Class*>(jni_class));
	  if (c == nullptr) {
	    return nullptr;
	  }
	  ArtMethod* method = nullptr;
	  auto pointer_size = Runtime::Current()->GetClassLinker()->GetImagePointerSize();
	  if (is_static) {
	    method = c->FindDirectMethod(name, sig, pointer_size);
	  } else if (c->IsInterface()) {
	    method = c->FindInterfaceMethod(name, sig, pointer_size);
	  } else {
	    method = c->FindVirtualMethod(name, sig, pointer_size);
	    if (method == nullptr) {
	      // No virtual method matching the signature.  Search declared
	      // private methods and constructors.
	      method = c->FindDeclaredDirectMethod(name, sig, pointer_size);
	    }
	  }
	  if (method == nullptr || method->IsStatic() != is_static) {
	    ThrowNoSuchMethodError(soa, c, name, sig, is_static ? "static" : "non-static");
	    return nullptr;
	  }
	  return soa.EncodeMethod(method);
	}
	
Class对象c描述的类在加载的过程中，经过解析已经关联上一系列的成员函数。这些成员函数可以分为两类：Direct和Virtual。Direct类的成员函数包括所有的静态成员函数、私有成员函数和构造函数，而Virtual则包括所有的虚成员函数。因此，该函数主要的是通过调用Class对象的一系列函数从Direct或Virtual列表中找到对应的方法信心对象ArtMethod，并返回该对象。

所以GetStaticMethodID返回的id其实是ArtMethod对象，有了这个对象后就可以轻松地得到它的本地机器指令入口，进而对它进行执行。

### ART执行类方法过程分析

从上文可以知道，通过Portable后端和Quick后端生成的OAT文件的本质区别在于，前者使用标准的动态链接器加载，而后者使用自定义的加载器加载。

标准动态链接器在加载SO文件（这里是OAT文件）的时候，会自动处理重定位问题。也就是说，在生成的本地机器指令中，如果有依赖其它的SO导出的函数，那么标准动态链接器就会将被依赖的SO也加载进来，并且从里面找到被引用的函数的地址，用来重定位引用了该函数的符号。生成的本地机器指令引用的一般都是些什么SO呢？其实就是ART运行时库（libart.so）。例如，如果在生成的本地机器指令需要分配一个对象，那么就需要调用ART运行时的堆管理器提供的AllocObject接口来分配。

   自定义加载器的做法就不一样了。它在加载OAT文件时，并不需要做上述的重定位操作。因为Quick后端生成的本地机器指令需要调用一些外部库提供的函数时，是通过一个函数跳转表来实现的。由于在加载过程中不需要执行重定位，因此加载过程就会更快，Quick的名字就是这样得来的。Portable后端生成的本地机器指令在调用外部库提供的函数时，使用了标准的方法，因此它不但可以在ART运行时加载，也可以在其它运行时加载，因此就得名于Portable。
   
   在ART运行时中创建的线程，和Davik虚拟机线程一样，在内部也会通过一个Thread对象来描述。这个新的Thread对象内部除了定义JNI调用函数表之外，还定义了我们在上面提到的外部函数调用跳转表。在ART运行时的启动和初始化过程，其中一个初始化过程便是将主线程关联到ART中去。
   
	bool Runtime::Init(RuntimeArgumentMap&& runtime_options_in) {
		....
		java_vm_ = new JavaVMExt(this, runtime_options);
		Thread::Startup();   
		Thread* self = Thread::Attach("main", false, nullptr, false);
		....
	}
	
通过调用Thread类的静态成员函数Attach将当前线程，也就是主线程，关联到ART运行时去。在关联的过程中，就会初始化一个外部库函数调用跳转表。实际跳转表的初始化是在InitEntryPoints函数中，该函数实现与CPU结构有关。

无论它是解释执行，还是本地机器指令执行，相应的调用入口都是已经通过ArtMehtod类的成员函数SetEntryPointFromCompiledCode和SetEntryPointFromInterpreter设置好了的。

当我们通过调用DexFile类的成员函数loadClass来加载一个类的时候，最终会调用到C++层的JNI函数DexFile_defineClassNative来执行真正的加载操作。在ART运行时中，每当一个类被加载时，ART运行时都会检查该类所属的DEX文件是否已经关联有一个Dex Cache。如果还没有关联，那么就会创建一个Dex Cache，并且建立好关联关系。得到了之前打开的DEX文件之后，接下来就调用ClassLinker类的成员函数RegisterDexFile将它注册到ART运行时中去，以便以后可以查询使用。最后再通过ClassLinker类的成员函数DefineClass来加载参数javaName指定的类。

 Dex Cache的作用是用来缓存包含在一个DEX文件里面的类型（Type）、方法（Method）、域（Field）、字符串（String）和静态储存区（Static Storage）等信息。
 
 有如下结论：
 
1. 在生成的本地机器指令中，一个类方法调用另外一个类方法并不是直接进行的，而是通过Dex Cache来间接进行的。

2. 通过Dex Cache间接调用类方法，可以做到延时解析类方法，也就是等到类方法第一次被调用时才解析，这样可以避免解析那些永远不会被调用的类方法。

3. 一个类方法只会被解析一次，解析的结果保存在Dex Cache中，因此当该类方法再次被调用时，就可以直接从Dex Cache中获得所需要的信息。


方法调用最终通过ArtMethod类的成员函数Invoke的实现：

	void ArtMethod::Invoke(Thread* self, uint32_t* args, uint32_t args_size, JValue* result,
	                       const char* shorty) {
	   ....
		bool have_quick_code = GetEntryPointFromQuickCompiledCode() != nullptr;
		if (LIKELY(have_quick_code)) {
		....
		if (!IsStatic()) {
			(*art_quick_invoke_stub)(this, args, args_size, self, result, shorty);
			} else {
			(*art_quick_invoke_static_stub)(this, args, args_size, self, result, shorty);
			}
		}
		....                                     
	}
	
	
从前面的函数LinkCode可以知道，无论一个类方法是通过解释器执行，还是直接以本地机器指令执行，均可以通过ArtMethod类的成员函数GetEntryPointFromCompiledCode获得其入口点，并且该入口不为NULL。不过，这里并没有直接调用该入口点，而是通过Stub来间接调用。这是因为我们需要设置一些特殊的寄存器。