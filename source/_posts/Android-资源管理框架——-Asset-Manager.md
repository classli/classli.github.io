---
title: Android 资源管理框架—— Asset Manager
date: 2018-01-07 15:04:06
tags:
---

### 资源的编译和打包过程分析

我们在编译一个android应用程序时，至少涉及两个包，其中一个是被应用的系统资源包，另一个就是当前的应用资源包。一个包引用其他包的资源通过资源ID方式实习。资源ID是一个4字节的无符号整数，其中，最高字节表示Package ID，次高字节表示Type ID，最低两字节表示Entry ID。
<!-- more -->
Package ID相当于资源的命名空间，系统资源为0x01,应用资源为0x7f.
Type ID指资源类型ID。
Entry ID指该资源在该类型下出现的次序。

在编译应用资源前，aapt工具会创建一个AaptAssets对象，用来收集资源文件。
然后会将该AaptAssets对象中的数据添加到资源表中，后续会根据该表生成资源索引表，即生成resources.arsc文件。

然后再为其他资源分配ID,前面准备的资源ID就是为后续编译xml文件做准备。

解析xml文件是为了可以在内存中用一系列树形结构XMLNode来表示，然后将前面的资源ID赋值给该对象中的资源。

android:id属性值“@+id/button_start_in_process”，其中，“@”表示后面描述的属性是引用类型的，“+”表示如果该引用不存在，那么就新建一个，“id”表示引用的资源类型是id, “button_start_in_process”表示引用的名称。应用属性前面还可以指定包名，例如：“@+[package:]id/button_start_in_process”

### Asset Manager 创建过程分析

Activity通过ContextImpl类中的getResources和getAssets得到Resources对象和AssetManager对象，其中，Resources对象通过资源id访问资源，AssetManager则是通过文件名访问资源，实际上Resources是先通过id找到资源文件名，在通过AssetManager进行资源获取。

ContextImpl中的mResources指向了Resources对象，而Resources里的mAssets则指向了AssetManager。getAssets就是按照这条路径来获取AssetManager对象。系统的资源打包在/system/framework/framework-res.apk文件中，应用程序通过单独的一个Resources和AssetManager来访问，通过getSystem即可访问到这两个单独的对象。

Java层的AssetManager主要是通过C++层的AssetManager来实现。C++层的AssetManager类有三个重要的成员变量mAssetPaths、mResources和mConfig。其中，mAssetPaths保存的是资源存放目录，mResources指向的是一个资源索引表，而mConfig保存的是设备的本地配置信息。

每个Activity加载过程中会创建ContextImpl来初始化上下文，Resources和AssetManager也是此时创想。

	class ContextImpl extends Context {
	
		private ContextImpl(...) {
			 ....
			 mPackageInfo = packageInfo;
	        mResourcesManager = ResourcesManager.getInstance();
	        .....
	        Resources resources = packageInfo.getResources(mainThread);
	        ....
	        if (container != null) {
                    // This is a nested Context, so it can't be a base Activity context.
                    // Just create a regular Resources object associated with the Activity.
                    resources = mResourcesManager.getResources(
                            activityToken,
                            packageInfo.getResDir(),
                            packageInfo.getSplitResDirs(),
                            packageInfo.getOverlayDirs(),
                            packageInfo.getApplicationInfo().sharedLibraryFiles,
                            displayId,
                            overrideConfiguration,
                            compatInfo,
                            packageInfo.getClassLoader());
             } else {
                    // This is not a nested Context, so it must be the root Activity context.
                    // All other nested Contexts will inherit the configuration set here.
                    resources = mResourcesManager.createBaseActivityResources(
                            activityToken,
                            packageInfo.getResDir(),
                            packageInfo.getSplitResDirs(),
                            packageInfo.getOverlayDirs(),
                            packageInfo.getApplicationInfo().sharedLibraryFiles,
                            displayId,
                            overrideConfiguration,
                            compatInfo,
                            packageInfo.getClassLoader());
            } 
           .....
           mResources = resources;   
	   }
	   
	}
	
packageInfo指向LoadedApk对象，这个LoadedApk对象正是当前启动Activity的apk，mResources最后通过LoadedApk对象的成员函数getResources来创建。

	public class LoadedApk {
		public Resources getResources(ActivityThread mainThread) {
		        if (mResources == null) {
		            mResources = mainThread.getTopLevelResources(mResDir, mSplitResDirs, mOverlayDirs,
		                    mApplicationInfo.sharedLibraryFiles, Display.DEFAULT_DISPLAY, this);
		        }
		        return mResources;
		    }
	}

其中，mainThread指向描述当前进程的一个ActivityThread对象。调用mainThread.getTopLevelResources时需要传入apk路径。getTopLevelResources里又调用了ResourcesManager的getResources方法。

ResourcesManager里会在一个Map中查找该apk资源的resource对象是否已存在，存在就返回，否则就new一个resource对象和AssetManager对象。

	public class ResourcesManager {
		AssetManager createAssetManager(@NonNull final ResourcesKey key) {
				AssetManager assets = new AssetManager();
				// resDir can be null if the 'android' package is creating a new Resources object.
		        // This is fine, since each AssetManager automatically loads the 'android' package
		        // already.
		        if (key.mResDir != null) {
		            if (assets.addAssetPath(key.mResDir) == 0) {
		                Log.e(TAG, "failed to add asset path " + key.mResDir);
		                return null;
		            }
		        }
		
		        if (key.mSplitResDirs != null) {
		            for (final String splitResDir : key.mSplitResDirs) {
		                if (assets.addAssetPath(splitResDir) == 0) {
		                    Log.e(TAG, "failed to add split asset path " + splitResDir);
		                    return null;
		                }
		            }
		        }
		        .....
		}
	}
	
	
下面看看AssetManager的addAssetPath方法, 在AssetManager初始化时会调用C++层中的addDefaultAssets函数将系统资源apk路径添加进去。

	bool AssetManager::addDefaultAssets()
	{
	    const char* root = getenv("ANDROID_ROOT");
	    LOG_ALWAYS_FATAL_IF(root == NULL, "ANDROID_ROOT not set");
	
	    String8 path(root);
	    path.appendPath(kSystemAssets);
	
	    return addAssetPath(path, NULL, false /* appAsLib */, true /* isSystemAsset */);
	}
	
这个c++层的AssetManager对象会被保存在Java层AssetManager的mObject中。

接下来就是调用java层中AssetManager对象的addAssetPath来添加应用程序资源文件路径，java层的addAssetPath又通过C++层的android\_content\_AssetManager\_addAssetPath来进行实现。

 > android\_util\_AssetManager.cpp

	static jint android_content_AssetManager_addAssetPath(JNIEnv* env, jobject clazz,
	                                                       jstring path, jboolean appAsLib)
	{
	    ScopedUtfChars path8(env, path);
	    if (path8.c_str() == NULL) {
	        return 0;
	    }
	
	    AssetManager* am = assetManagerForJavaObject(env, clazz);
	    if (am == NULL) {
	        return 0;
	    }
	
	    int32_t cookie;
	    bool res = am->addAssetPath(String8(path8.c_str()), &cookie, appAsLib);
	
	    return (res) ? static_cast<jint>(cookie) : 0;
	}
	
首先建立C++层的AssetManager对象然后在调用addAssetPath将path所描述的apk路径添加进去。其过程如上所述。

然后就会通过所创建的AssetManager对象来创建Resources对象, 创建Resorces期间会根据获得config和metrics数据来更新设备的当前配置信息（屏幕大小、国家语言等）。android资源管理器的创建过程主要就是创建和初始化用来访问应用程序资源的AssetManager对象和Resources对象，其中，初始化操作包括设置AssetManager对象的资源文件路径以及设备配置信息等。

### 应用资源查找过程分析

从Activity中onCreate的setContentView开始分析，Activity中的setContentView最终调用的是PhoneWindow中的setContentView。

	public class PhoneWindow extends Window implements MenuBuilder.Callback {
	
		@Override
	    public void setContentView(int layoutResID) {
	        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
	        // decor, when theme attributes and the like are crystalized. Do not check the feature
	        // before this happens.
	        if (mContentParent == null) {
	            installDecor();
	        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
	            mContentParent.removeAllViews();
	        }
	
	        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
	            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
	                    getContext());
	            transitionTo(newScene);
	        } else {
	            mLayoutInflater.inflate(layoutResID, mContentParent);
	        }
	        mContentParent.requestApplyInsets();
	        final Callback cb = getCallback();
	        if (cb != null && !isDestroyed()) {
	            cb.onContentChanged();
	        }
	        mContentParentExplicitlySet = true;
	    }
	
	}
	
	
mContentParent 指向类型为DecorView的视图对象，或者是DecorView的子视图对象。如果mContentParent为null说明该Activity还没有创建视图，则调用installDecor()进行创建。调用
mLayoutInflater.inflate将layoutResID所描述的布局设置到mContentParent视图容器中去，这样Activity的视觉UI就创建出来。然后调用cb.onContentChanged来通知Activity他的视图发生了变化。

具体的资源加载解析靠的是inflate。

	public abstract class LayoutInflater {
		
		public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
			....
			final XmlResourceParser parser = res.getLayout(resource);
	        try {
	            return inflate(parser, root, attachToRoot);
	        } finally {
	            parser.close();
	        }
		}
	
	}
	
首先获得解析XML的parser对象，然后parser作为参赛调用inflate进行解析。getLayout在Resources类中，getLayout又直接转调了loadXmlResourceParser

	public class Resources {
		XmlResourceParser loadXmlResourceParser(@AnyRes int id, @NonNull String type){
			.....
			final TypedValue value = obtainTempTypedValue();
			.....
			final ResourcesImpl impl = mResourcesImpl;
	            impl.getValue(id, value, true);
	            if (value.type == TypedValue.TYPE_STRING) {
	                return impl.loadXmlResourceParser(value.string.toString(), id,
	                        value.assetCookie, type);
	            }
			.....
		}
	}
	
首先通过getValue获得资源id对应的资源(文件名)，并存储在TypedValue中，最后调用loadXmlResourceParser加载该资源。ResourcesImpl的getValue又直接调用了AssetManager对象的getResourceValue，getResourceValue实际是调用loadResourceValue加载参数并保存到TypedValue中，该方法通过资源id到资源表中查找该资源对应的资源项值及其配置信息。loadResourceValue是一个JNI方法，在资源查找过程中会根据配置查找到最合适的资源返回。


	public class Resources {
		XmlResourceParser loadXmlResourceParser(@NonNull String file, @AnyRes int id, int assetCookie,
	            @NonNull String type)
	            throws NotFoundException {
				...
				synchronized (mCachedXmlBlocks) {
                    final int[] cachedXmlBlockCookies = mCachedXmlBlockCookies;
                    final String[] cachedXmlBlockFiles = mCachedXmlBlockFiles;
                    final XmlBlock[] cachedXmlBlocks = mCachedXmlBlocks;
                    // First see if this block is in our cache.
                    final int num = cachedXmlBlockFiles.length;
                    for (int i = 0; i < num; i++) {
                        if (cachedXmlBlockCookies[i] == assetCookie && cachedXmlBlockFiles[i] != null
                                && cachedXmlBlockFiles[i].equals(file)) {
                            return cachedXmlBlocks[i].newParser();
                        }
                    }

                    // Not in the cache, create a new block and put it at
                    // the next slot in the cache.
                    final XmlBlock block = mAssets.openXmlBlockAsset(assetCookie, file);
                    if (block != null) {
                        final int pos = (mLastCachedXmlBlockIndex + 1) % num;
                        mLastCachedXmlBlockIndex = pos;
                        final XmlBlock oldBlock = cachedXmlBlocks[pos];
                        if (oldBlock != null) {
                            oldBlock.close();
                        }
                        cachedXmlBlockCookies[pos] = assetCookie;
                        cachedXmlBlockFiles[pos] = file;
                        cachedXmlBlocks[pos] = block;
                        return block.newParser();
                    }
                }
				...
		}
	}
	
mCachedXmlBlocks、mCachedXmlBlockIds和mLastCachedXmlBlockIndex是缓存xml资源的变量，有缓存就直接newParser一个XmlResourceParser返回，否则调用mAssets.openXmlBlockAsset来打开file指定的资源文件，并返回XmlBlock对象，最后调用其newParser方法返回XmlResourceParser对象。

有了这个XmlResourceParser对象，就可调用另一个inflate方法进行参数加载。

	public abstract class LayoutInflater {
		public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
			 synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;
			....
            // Look for the root node.
            int type;
            while ((type = parser.next()) != XmlPullParser.START_TAG &&
                    type != XmlPullParser.END_DOCUMENT) {
                // Empty
            }

            if (type != XmlPullParser.START_TAG) {
                throw new InflateException(parser.getPositionDescription()
                        + ": No start tag found!");
            }

            final String name = parser.getName();
            
            ....

            if (TAG_MERGE.equals(name)) {
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }

                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                // Temp is the root view that was found in the xml
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                ViewGroup.LayoutParams params = null;

                if (root != null) {
                    if (DEBUG) {
                        System.out.println("Creating params from root: " +
                                root);
                    }
                    // Create layout params that match root, if supplied
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        // Set the layout params for temp if we are not
                        // attaching. (If we are, we use addView, below)
                        temp.setLayoutParams(params);
                    }
                }
				...

                // Inflate all children under temp against its context.
                rInflateChildren(parser, temp, attrs, true);
				...

                // We are supposed to attach all the views we found (int temp)
                // to root. Do that now.
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }

                // Decide whether to return the root that was passed in or the
                // top view found in xml.
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }
			.....
            return result;
        }
		}
	}
	
	
从上面可以看出，inflate主要是处理xml中的各个节点，然后在对节点的字节点调用rInflateChildren进行循环解析。view节点用过createViewFromTag来创建。

其中如果inflate中的第二个参数root不等于null，那么新建的view就会用root的layoutParams参数作为自身的参数，如果inflate第三个参数attachToRoot等于true，那么创建的view作为子view直接添加到root中。如果xml跟节点为merge那么就直接处理其中的字节点。

	public abstract class LayoutInflater {
		View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
            boolean ignoreThemeAttr) {
			if (name.equals("view")) {
            	name = attrs.getAttributeValue(null, "class");
        	}
        	....
        	View view;
            if (mFactory2 != null) {
                view = mFactory2.onCreateView(parent, name, context, attrs);
            } else if (mFactory != null) {
                view = mFactory.onCreateView(name, context, attrs);
            } else {
                view = null;
            }

            if (view == null && mPrivateFactory != null) {
                view = mPrivateFactory.onCreateView(parent, name, context, attrs);
            }

            if (view == null) {
                final Object lastContext = mConstructorArgs[0];
                mConstructorArgs[0] = context;
                try {
                    if (-1 == name.indexOf('.')) {
                        view = onCreateView(parent, name, attrs);
                    } else {
                        view = createView(name, null, attrs);
                    }
                } finally {
                    mConstructorArgs[0] = lastContext;
                }
            }
            return view;
            ....
		}
	}
	
	
如果xml中节点的名字为"view"那么实际的view名字存在attrs属性集中一个名称为“class”的属性中，getAttributeValue就是获得实际的UI名。UI控件实际通过Factory对象(UI工厂)的onCreateView方法进行创建，如果控件名中包括“.”那么就表明该控件为自定义控件。

当控件为自定义控件时，调用了成员函数onCreateView，该函数在其子类PhoneLayoutInflater进行了实现。PhoneLayoutInflater只会对 "android.widget."、"android.webkit."、"android.app."三种UI进行处理，其他直接交给了他的父类中的onCreateView，否则就调用父类中的createView。

	public abstract class LayoutInflater {
		public final View createView(String name, String prefix, AttributeSet attrs)
            throws ClassNotFoundException, InflateException {
		        Constructor<? extends View> constructor = sConstructorMap.get(name);
		        ....
		        if (constructor == null) {
	                // Class not found in the cache, see if it's real, and try to add it
	                clazz = mContext.getClassLoader().loadClass(
	                        prefix != null ? (prefix + name) : name).asSubclass(View.class);
	                
	                if (mFilter != null && clazz != null) {
	                    boolean allowed = mFilter.onLoadClass(clazz);
	                    if (!allowed) {
	                        failNotAllowed(name, prefix, attrs);
	                    }
	                }
	                constructor = clazz.getConstructor(mConstructorSignature);
	                constructor.setAccessible(true);
	                sConstructorMap.put(name, constructor);
	            } else {
	                // If we have a filter, apply it to cached constructor
	                if (mFilter != null) {
	                    // Have we seen this name before?
	                    Boolean allowedState = mFilterMap.get(name);
	                    if (allowedState == null) {
	                        // New class -- remember whether it is allowed
	                        clazz = mContext.getClassLoader().loadClass(
	                                prefix != null ? (prefix + name) : name).asSubclass(View.class);
	                        
	                        boolean allowed = clazz != null && mFilter.onLoadClass(clazz);
	                        mFilterMap.put(name, allowed);
	                        if (!allowed) {
	                            failNotAllowed(name, prefix, attrs);
	                        }
	                    } else if (allowedState.equals(Boolean.FALSE)) {
	                        failNotAllowed(name, prefix, attrs);
	                    }
	                }
	            }
	
	            Object[] args = mConstructorArgs;
	            args[1] = attrs;
	
	            final View view = constructor.newInstance(args);
	            if (view instanceof ViewStub) {
	                // Use the same context when inflating ViewStub later.
	                final ViewStub viewStub = (ViewStub) view;
	                viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
	            }
	            return view;
	            .....
            }
	}
 
 该函数首先缓存中提取控件的构造函数，这些构造函数必须具有两个参数，一个Context一个是AttributeSet，在创建自定义控件时会将这两个参数缓存在mConstructorArgs中，后面可以直接取出这两个参数初始化对应控件。**如果自定义的控件构造函数没有这两个参数，那么该自定义控件就不可以在UI布局文件中使用。**
 
 其中mFilter指向的是一个Filter对象，用来过滤该控件是否允许被创建，该过滤器只有第一次创建控件时才调用，后续通过缓存的mFilterMap直接确定。
 
 
 
   