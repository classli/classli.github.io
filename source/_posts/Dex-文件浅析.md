---
title: Dex 文件浅析
date: 2017-12-02 15:48:35
tags:
---
清楚的了解dex文件的结构属性，有助于对Android apk加固、Android hotpatch原理、Android虚拟机、Android逆向工程等的学习掌握。是Android高级开发工程师不可或缺的一个知识点。

### dex文件大致结构
	struct DexFile {
	const DexHeader*    pHeader;
	const DexStringId*  pStringIds;
	const DexTypeId*    pStringIds;
	const DexFieldId*   pFieldIds;
	const DexMethodId*  pMethodIds;
	const DexProtoId*   pProtoIds;
	const DexClassDef*  pClassDefs;
	const DexLink*      pLinkData;
	}
总的来说dex文件可以大致分为三部分：
<!-- more -->
1. Header Section

2. Table Section

3. Data Section

具体Dex文件中各结构可查看/dalvik/libdex/DexFile.h

### Header Section

Header部分具体字段由DexHeader结构体定义，各个字段的含义以及字节大小可参照网上N多文章，这里不再细讲，总的可分4种类型字段：

 1. 文件校验字段 magic，checksum和signature，这部分的数据让解析调用该文件的使用者明确这个文件是否已经篡改或者损坏。
 2. 文件内容属性字段 file_size，header_size和endian_tag，dex文件自身属性相关内容。
 3. mapOff字段，用来表示DexMapList在Data Section中的偏移量，通过这个偏移量我们可以得到DexMapList数据，dalvik虚拟机解析dex文件的内容，最终将其映射成DexMapList数据结构，这个DexMapList中包含了dex文件中可能出现的所有类型，作用是用于校验。
 4. 剩余部分的xxx_size和xxx_offset字段，通过这些部分我们可以拿到Table Section中的各项数据。

### Table Section

这部分的内容和Header Section是紧密相连的。从DexFile结构体中可以看出一共有6个table(String、Type、Field、Method、Proto和Class)，而Header Section中也正好对应了这6个table的size和offset。所以我们可以通过Header Section里的size和offset以及它们各自所占的大小来精确的定位到每一个table的在dex文件中的位置。各个table的结构如下：

**String Table：**  

stringDataOff：表示StringData在Data Section中的偏移量，StringData中包含了一个dex文件中的所有字符串资源，比如类名，方法名等等。这个table也将被后面几个table反复的用到。

**Type Table:** 

descriptorIdx：用来索引String Table，得到一个表示类型的字符串，比如方法的返回类型，类类型等。

**Field Table:** 

classIdx：用来索引Type Table，进而通过Type Item得到一个字符串资源用来表示该field所属的类；

typeIdx：用来索引Type Table，进而得到一个字符串资源用来表示该field的类型；

nameIdx：用来索引String Table，得到一个字符串资源用来表示该field的名称。

**Method Table:**

classIdx：用来索引Type Table，进而得到一个字符串资源用来表示该method所属的类；

protoIdx：用来索引Proto Table，从而得到一个method的原型，返回类型和入参类型；

nameIdx：用来索引String Table，得到一个字符串资源用来表示该method的名称。

**Proto Table:**

shortyIdx：用来索引String Table，得到一个字符串资源表示该method的原型；

returnTypeIdx：用来索引Type Table，进而得到该method的返回类型；

parametersOff：用来表示TypeList在Data Section中的偏移量，TypeList中包含了一个方法的所有入参类型。

**ClassDefs Table:**

classIdx：用来索引Type Table，得到一个字符串表示该类的类型；

accessFlags：用来表示该类的访问权限，如public，private等；

superclassIdx：用来索引Type Table，得到一个字符串资源用来表示该类的父类的类型；

interfacesOff：用来表示TypeList(Type Item的集合)在Data Section中的偏移量，TypeList中包含了一个类所含的声明或实现的接口；

sourceFileIdx：用来索引String Table，得到一个字符串表示该类在文件中的名称；

annotationsOff：用来表示DexAnnotationsDirectoryItem在Data Section中的偏移量，DexAnnotationsDirectoryItem中包含了该类所有的注解；

classDataOff：用来表示DexClassData在Data Section中的偏移量，DexClassData用来表示该类的数据；

staticValuesOff:用来表示DexEncodedArray在Data Section中的偏移量，DexEncodedArray用来表示该类的所有静态数据。


### dex文件的调用

在art对各个apk进行编译的时候就需要对dex进行解析，编译成可执行的ELF文件。 从对dex的发起实际操作在class_linker.cc里。查找具体的类调用了ClassLinker::ResolveType方法，

	mirror::Class* ClassLinker::ResolveType(...) {
	   ...
	   mirror::Class* resolved = dex_cache->GetResolvedType(type_idx);
	   if (resolved == nullptr) {
	    	...
	    	const char* descriptor = dex_file.StringByTypeIdx(type_idx);
	    	resolved = FindClass(self, descriptor, class_loader);
	    	...
	     	dex_cache->SetResolvedType(type_idx, resolved);
	    }	 
	    ...
	}
	
当这个类没被从dex中解析过，就会先通过type\_idx 获得这个类的描述，这个type\_idx实际就是用来索引Type Table从而得到这个类型的类型字符串。

对于方法的查找可以查看ClassLinker::ResolveMethod方法，
	
	ArtMethod* ClassLinker::ResolveMethod(...) {
		...
		ArtMethod* resolved = dex_cache->GetResolvedMethod(method_idx, image_pointer_size\_);
		...
		const DexFile::MethodId& method_id = dex_file.GetMethodId(method_idx);
		...
		mirror::Class* klass = ResolveType(dex_file, method_id.class_idx_, dex_cache, class_loader);
		...
		const char* name = dex_file.StringDataByIdx(method_id.name_idx_);
	    const Signature signature = dex_file.GetMethodSignature(method_id);
	    switch (type) {
	        ...
	        resolved = klass->FindDirectMethod(name, signature, image_pointer_size_);
	        ...
	        resolved = klass->FindInterfaceMethod(name, signature, image_pointer_size_);
	        ...
	        resolved = klass->FindVirtualMethod(name, signature, image_pointer_size_);
	        break;
	    }
	}
        
调用该方法后返回一个ArtMethod对象，该对象里记录了要调用方法信息，其中就包括该方法在dex文件中的信息，里面也包括方法的入口指针，篡改该指针可到达热修复的作用，Andfix就是对该对象进行替换实现。

对于方法的查找，首先会拿到要调用方法的method_id，对应上述的Method Table， 然后根据Method Table中的nameIdx获得方法名，最后通过protoIdx索引到Proto Table拿到方法的输入参数Signature，klass即为方法对应的类。具备了上述参数后就可找到类中对应的方法。

从上面的分析看出，平时我们的类查找、方法调用或字段调用，都是强烈的依赖dex文件本身的结构。对于想直接通过修改dex文件以达到需求目的，必须把每个细节掌握清楚，否则只能呵呵！
	
