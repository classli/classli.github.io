---
title: SurfaceView实现原理浅析
date: 2017-12-31 14:08:02
tags:
---
SurfaceView拥有独立的绘图表面，因此SurfaceView的UI就可以在一个独立的线程中进行绘制，SurfaceView的UI是绘制在SurfaceFlinger服务中的另外一个Layer或者LayerBuffer上的。

### SurfaceView的绘图表面的创建过程

当一个窗口需要刷新UI时，就会调用ViewRoot类的成员函数performTraversals，performTraversals在执行的过程中，如果发现当前窗口的绘图表面还没有创建，或者发现当前窗口的绘图表面已经失效了，那么就会请求WindowManagerService服务创建一个新的绘图表面，同时，它还会通过一系列的回调函数来让嵌入在窗口里面的SurfaceView有机会创建自己的绘图表面。
<!-- more -->
	
	public final class ViewRootImpl implements ViewParent,
	        View.AttachInfo.Callbacks, ThreadedRenderer.HardwareDrawCallbacks {
		private void performTraversals() {
			...
			if (mFirst) {
				...
				host.dispatchAttachedToWindow(mAttachInfo, 0);
				...
			}
			...
			if (viewVisibilityChanged) {
				...
				host.dispatchWindowVisibilityChanged(viewVisibility);
				...
			}		
		}        
	        
	}
	
如上，第一次刷新UI时会通过调用DecorView类的dispatchAttachedToWindow，从顶层视图开始通知子视图添加到宿主窗口上去。如果当前窗口的可见性发生了变化，会通过调用DecorView类的dispatchWindowVisibilityChanged，从顶层视图开始通知子视图宿主窗口的可见发生变化了。

在宿主窗口类中，只是将这些通知信息派发给了子视图，具体处理由子视图实现。SurfaceView类的dispatchAttachedToWindow从父view中继承下来，该方法里面又调用了onAttachedToWindow方法进行具体实现。

	public class SurfaceView extends View {
		 @Override
   		 protected void onAttachedToWindow() {
	        super.onAttachedToWindow();
	        mParent.requestTransparentRegion(this);
	        mSession = getWindowSession();
	        mLayout.token = getWindowToken();
	        mLayout.setTitle("SurfaceView - " + getViewRootImpl().getTitle());
	        mLayout.packageName = mContext.getOpPackageName();
	        mViewVisibility = getVisibility() == VISIBLE;
	
	        if (!mGlobalListenersAdded) {
	            ViewTreeObserver observer = getViewTreeObserver();
	            observer.addOnScrollChangedListener(mScrollChangedListener);
	            observer.addOnPreDrawListener(mDrawListener);
	            mGlobalListenersAdded = true;
	        }
	    }
	}

该方法的第一件事就是通过requestTransparentRegion通知父视图(宿主视图),在父视图设置一个透明窗口，因为SurfaceView其实是在父视图的下面，所以需要在父视图上设置透明部分以显示下面的SurfaceView。

然后调用getWindowSession获得一个Binder对象，该Binder主要是与WindowManagerService通信。

dispatchWindowVisibilityChanged与上面的dispatchAttachedToWindow调用路径相似。

SurfaceView的窗口默认类型为TYPE\_APPLICATION\_MEDIA，如显示视频，另外一个SurfaceView的窗口类型为TYPE\_APPLICATION\_MEDIA_OVERLAY，这个Overlay可以用来显示视字幕等信息。

SurfaceView的绘图表面的类型可能等于SURFACE\_TYPE\_NORMAL, 也可能等于SURFACE\_TYPE\_PUSH\_BUFFERS，SURFACE\_TYPE\_NORMAL时, 表面该SurfaceView绘图面使用的内存是一块普通的内存，由SurfaceFlinger服务分配来，我们可以对该内存进行正常操作。等于SURFACE\_TYPE\_PUSH\_BUFFERS时，绘图面的内存不是由SURFACE\_TYPE\_PUSH\_BUFFERS分配，此时我们不能对该内存进行操作，如SurfaceView显示摄像头预览视频播放时。我们可以调用SurfaceHolder的setType来改变这个绘图面类型。

### SurfaceView的绘制过程

窗口在绘制的过程中，每一个子视图的成员函数draw或者dispatchDraw都会被调用到，以便它们可以绘制自己的UI。

	public class SurfaceView extends View {
		@Override
	    public void draw(Canvas canvas) {
	        if (mWindowType != WindowManager.LayoutParams.TYPE_APPLICATION_PANEL) {
	            // draw() is not called when SKIP_DRAW is set
	            if ((mPrivateFlags & PFLAG_SKIP_DRAW) == 0) {
	                // punch a whole in the view-hierarchy below us
	                canvas.drawColor(0, PorterDuff.Mode.CLEAR);
	            }
	        }
	        super.draw(canvas);
	    }
	
	    @Override
	    protected void dispatchDraw(Canvas canvas) {
	        if (mWindowType != WindowManager.LayoutParams.TYPE_APPLICATION_PANEL) {
	            // if SKIP_DRAW is cleared, draw() has already punched a hole
	            if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
	                // punch a whole in the view-hierarchy below us
	                canvas.drawColor(0, PorterDuff.Mode.CLEAR);
	            }
	        }
	        super.dispatchDraw(canvas);
	    }
	}
	
从SurfaceView中的这两个方法可以看出，当前正在处理的SurfaceView不是用作宿主窗口面板的时，SurfaceView会将其所在的区域绘制成黑色。SurfaceView在其宿主窗口的绘图表面上面所做的操作就是将自己所占据的区域绘为黑色。

在一个绘图表面进行UI绘制，基本顺序如下：

1. 在绘图表面的基础上建立一块画布，即获得一个Canvas对象。
2. 利用Canvas类提供的绘图接口在前面获得的画布上绘制任意的UI。
3. 将已填充好UI数据的画布缓冲区提交给SurfaceFlinger服务，以便SurfaceFlinger服务可以将它合成到屏幕上去。

SurfaceHolder接口可以执行上述的第1和第3操作。

	SurfaceView sv = (SurfaceView )findViewById(R.id.surface_view);  
	  
	SurfaceHolder sh = sv.getHolder();  
	  
	Cavas canvas = sh.lockCanvas()  
	  
	//Draw something on canvas  
	......  
	  
	sh.unlockCanvasAndPost(canvas); 
	
上述代码理论上最好在子线程里执行。其中getHolder只是将SurfaceHolder对象返回给了调用者。lockCanvas会直接调用internalLockCanvas方法。

	public class SurfaceView extends View {
		 mSurfaceLock.lock();
		 ....
		 if (!mDrawingStopped && mWindow != null) {
                try {
                    c = mSurface.lockCanvas(dirty);
                } catch (Exception e) {
                    Log.e(LOG_TAG, "Exception locking surface", e);
                }
            }
        ....
        if (c != null) {
                mLastLockTime = SystemClock.uptimeMillis();
                return c;
            }
		 ....
		 mSurfaceLock.unlock();
	}
	
由于创建的画布不是线程安全的所以对其进行了加锁处理，mWindow表示绘图面，最终是调用Surface对象的成员函数lockCanvas来创建一块画布。

Surface的lockCanvas大致就是通过JNI方法来在当前正在处理的绘图表面上获得一个图形缓冲区，并且将这个图形绘冲区封装在一块类型为Canvas的画布中返回给调用者使用。

绘图者在Canvas画布上绘制后，就调用unlockCanvasAndPost将这块画布的数据交给SurfaceFlinger服务来处理，在该方法中会调用Surface对象的unlockCanvasAndPost提交数据给SurfaceFlinger，并同时unlock画布。