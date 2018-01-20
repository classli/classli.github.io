---
title: TextView的实现原理
date: 2018-01-20 16:26:17
tags:
---
TextView是一个比较有代表性的控件，许多控件都是继承自该控件。

应用程序窗口，即Activity窗口，是由一个PhoneWindow对象，一个DecorView对象，以及一个ViewRoot对象来描述的。其中，PhoneWindow对象用来描述窗口对象，DecorView对象用来描述窗口的顶层视图，ViewRoot对象除了用来与WindowManagerService服务通信之外，还用来接收用户输入。

各个View间是以树形的结构组织在一起，这里加上TextView直接以窗口的顶层视图为父视图，即以DecorView为父视图
<!-- more -->
### TextView控件的UI绘制框架

为了告诉父视图自己所占据空间大小，所有控件必须要重写父类View中的onMeasure。

	public class TextView extends View implements ViewTreeObserver.OnPreDrawListener {  
	    @Override  
	    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {  
	        int widthMode = MeasureSpec.getMode(widthMeasureSpec);  
	        int heightMode = MeasureSpec.getMode(heightMeasureSpec);  
	        int widthSize = MeasureSpec.getSize(widthMeasureSpec);  
	        int heightSize = MeasureSpec.getSize(heightMeasureSpec);  
	  
	        int width;  
	        int height;  
	  		...
	  		
	        setMeasuredDimension(width, height);  
	    }  
	}  
	
widthMeasureSpec和heightMeasureSpec分别用来高宽的测量范围，该范围包含两个分量。

第一个是mode分量，高两位表示。有三种类型分别是MeasureSpec.UNSPECIFIED（0）、MeasureSpec.EXACTLY（1）、和MeasureSpec.AT_MOST（2）

第二个是size分量，使用低30位表示。

* 当mode=MeasureSpec.EXACTLY, size等于父视图要求当前控件的高宽。
* 当mode=MeasureSpec.AT_MOST, size等于父视图限定当前控件的最大高宽。
* 当mode=MeasureSpec.UNSPECIFIED, 父视图不限定当前控件的高宽。

根据上述规则控件测量好自己的高宽后，需调用setMeasuredDimension来通知父视图自己的高宽。

对于onLayout只有容器类控件才需要布局操作，因为子控件的位置由父控件来设置，而TextView控件不是容器类控件，所以它可以不重写父类View的onLayout。

经过测量和布局后，控件的位置和大小就确定下来了，为了能绘制控件的UI, 需要重写父类view中的onDraw。

为什么要规定所有与UI相关的操作都必须在主线程中执行呢？因为UI相关的操作需要大量地访问控件的内部成员以及窗口绘图面的图形缓冲区，因此如果不将所有的UI操作都限定在同一线程里，那么就会涉及多线程同步问题，关键是线程同步开销将很大。

### TextView控件获取键盘输入的过程

每一个窗口创建的时候，都会与系统输入管理器建立一个输入接收通道。输入管理器中有两个线程，一个线程是用来监听用户的输入行为，另一个线程用来将该输入事件分发给当前激活的窗口，这个分发的线程就是通过前面建立的通道进行的。

当前窗口接收到输入事件后，就将该事件封装成消息发送到主线程的消息队列中去，当处理该消息时就会调用当前窗口所关联的ViewRoot对象的deliverKeyEvent或者deliverPointerEvent来将前面接收到的用户输入分发给合适的控件。deliverKeyEvent负责分发键盘输入事件，deliverPointerEvent负责分发触摸屏输入事件。

在最新的android代码中ViewRoot类变成了ViewRootImpl类，上述两个方法也不存在了。ViewRootImpl通过Handler来处理各种消息。

	public final class ViewRootImpl implements ViewParent,
	        View.AttachInfo.Callbacks, ThreadedRenderer.HardwareDrawCallbacks {
		final class ViewRootHandler extends Handler {
			public void handleMessage(Message msg) {
				...
				case MSG_PROCESS_INPUT_EVENTS:
                mProcessInputEventsScheduled = false;
                doProcessInputEvents();
                break;
				...
			}
		}	             
	}
	
其中 doProcessInputEvents会调用deliverInputEvent传递输入事件，deliverInputEvent会调用InputStage内部类的deliver方法进行处理,最后会调用到ViewPreImeInputStage内部类的 processKeyEvent方法。

	public final class ViewRootImpl implements ViewParent,
		        View.AttachInfo.Callbacks, ThreadedRenderer.HardwareDrawCallbacks 	{
		final class ViewPreImeInputStage extends InputStage {
			 private int processKeyEvent(QueuedInputEvent q) {
	            final KeyEvent event = (KeyEvent)q.mEvent;
	            if (mView.dispatchKeyEventPreIme(event)) {
	                return FINISH_HANDLED;
	            }
	            return FORWARD;
	        }
		}	             
	}
	

mView指向窗口的顶层视图，即一个DecorView对象，系统将先调用dispatchKeyEventPreIme方法让它优先于输入法处理键盘事件，如果dispatchKeyEventPreIme返回为true，那么说明键盘处理已完成，后续的输入法处理流程不需要进行了。

需要注意的是，只有当前窗口在显示输入法的情况下，ViewRootImpl才将键盘事件分发给输入法处理，这是通过检查mLastWasImTarget变量是否等于true来实现。

当通过dispatchInputEvent将键盘事件交给输入法处理时，同时还会传递一个回调函数给输入法，以遍当输入结束后，调用该函数来通知ViewRootImpl，然后将该键盘事件分发给窗口处理。


下面先分析下输入法处理前的键盘处理过程。

DecorView类的dispatchKeyEventPreIme从ViewGroup继承下来。

	public abstract class ViewGroup extends View implements ViewParent, ViewManager {
	    @Override
	    public boolean dispatchKeyEventPreIme(KeyEvent event) {
	        if ((mPrivateFlags & (PFLAG_FOCUSED | PFLAG_HAS_BOUNDS))
	                == (PFLAG_FOCUSED | PFLAG_HAS_BOUNDS)) {
	            return super.dispatchKeyEventPreIme(event);
	        } else if (mFocused != null && (mFocused.mPrivateFlags & PFLAG_HAS_BOUNDS)
	                == PFLAG_HAS_BOUNDS) {
	            return mFocused.dispatchKeyEventPreIme(event);
	        }
	        return false;
	    }
	}

在当前视图容器能够获得焦点并且容器已经计算过大小的情况下，就调用视图容器父类中的dispatchKeyEventPreIme。如果上述条件不满足，但视图容器中有一个子视图可以获得焦点，并且子视图大小也计算过了，那么将调用子视图的dispatchKeyEventPreIme。

最终都是通过调用View类的dispatchKeyEventPreIme来在输入法之前处理event事件, dispatchKeyEventPreIme只是简单地调用了onKeyPreIme, 只要我们重写onKeyPreIme方法即可在输入法之前处理键盘事件，该方法返回true表示键盘事件处理完成不在传递下去，否则返回false。

当上面执行完之后，如果event事件继续传递，最终又会通过上述的ViewRootImpl的消息处理机制调用到ViewPostImeInputStage内部onProcess方法。

	public final class ViewRootImpl implements ViewParent,
				        View.AttachInfo.Callbacks, ThreadedRenderer.HardwareDrawCallbacks 	{
		final class ViewPostImeInputStage extends InputStage {
			 protected int onProcess(QueuedInputEvent q) {
	            if (q.mEvent instanceof KeyEvent) {
	                return processKeyEvent(q);
	            }
	            ....
       	   	 }
	       	 private int processKeyEvent(QueuedInputEvent q) {
	            final KeyEvent event = (KeyEvent)q.mEvent;
	
	            // Deliver the key to the view hierarchy.
	            if (mView.dispatchKeyEvent(event)) {
	                return FINISH_HANDLED;
	            }
	            ....
	            // Handle automatic focus changes.
	            if (event.getAction() == KeyEvent.ACTION_DOWN) {
	                int direction = 0;
	                switch (event.getKeyCode()) {
	                    case KeyEvent.KEYCODE_DPAD_LEFT:
	                        if (event.hasNoModifiers()) {
	                            direction = View.FOCUS_LEFT;
	                        }
	                        break;
	                    ...
	                }
	                if (direction != 0) {
	                    View focused = mView.findFocus();
	                    if (focused != null) {
	                        View v = focused.focusSearch(direction);
	                        if (v != null && v != focused) {
	                            // do the math the get the interesting rect
	                            // of previous focused into the coord system of
	                            // newly focused view
	                            focused.getFocusedRect(mTempRect);
	                            if (mView instanceof ViewGroup) {
	                                ((ViewGroup) mView).offsetDescendantRectToMyCoords(
	                                        focused, mTempRect);
	                                ((ViewGroup) mView).offsetRectIntoDescendantCoords(
	                                        v, mTempRect);
	                            }
	                            if (v.requestFocus(direction, mTempRect)) {
	                                playSoundEffect(SoundEffectConstants
	                                        .getContantForFocusDirection(direction));
	                                return FINISH_HANDLED;
	                            }
	                        }
	                        ...
	                    }
                    }
	         }
		}	             
	}

上面的mView.dispatchKeyEvent就是将键盘事件交给当前激活窗口的顶层视图来处理，mView即指向DecorView。如果顶层视图还希望该键盘事件可以被子视图处理，dispatchKeyEvent就会返回false，然后就会根据屏幕触点方向调整子视图的焦点，子视图通过requestFocus获得焦点。

接下来分享下DecorView的dispatchKeyEvent方法。

	public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {
		@Override
	    public boolean dispatchKeyEvent(KeyEvent event) {
	        final int keyCode = event.getKeyCode();
	        final int action = event.getAction();
	        final boolean isDown = action == KeyEvent.ACTION_DOWN;
	        ....	
	        if (!mWindow.isDestroyed()) {
	            final Window.Callback cb = mWindow.getCallback();
	            final boolean handled = cb != null && mFeatureId < 0 ? cb.dispatchKeyEvent(event)
	                    : super.dispatchKeyEvent(event);
	            if (handled) {
	                return true;
	            }
	        }
	
	        return isDown ? mWindow.onKeyDown(mFeatureId, event.getKeyCode(), event)
	                : mWindow.onKeyUp(mFeatureId, event.getKeyCode(), event);
	    }
	}

假设当前处理DecorView对象的顶层视图是一个Activity，并且该Activity实现了个一Window.Callback接口，那么将调用该Activity的Window.Callback接口。如果Window.Callback的dispatchKeyEvent返回false，那该键盘事件将继续传递给PhoneWindow对像的onKeyDown或onKeyUp处理，这两个方法主要用来处理一些特殊按键电话键或音量键。

Activity类的dispatchKeyEvent方法如下。

	 public boolean dispatchKeyEvent(KeyEvent event) {
	        onUserInteraction();
	        ...
	        Window win = getWindow();
	        if (win.superDispatchKeyEvent(event)) {
	            return true;
	        }
	        View decor = mDecor;
	        if (decor == null) decor = win.getDecorView();
	        return event.dispatch(this, decor != null
	                ? decor.getKeyDispatcherState() : null, this);
	 }
	 
该方法首先将event事件交给了当前Activity的PhoneWindow对象处理，然后再通过dispatch将该event事件交给当前的Activity组件处理，dispatch执行过程就是调用Activity类的成员函数onKeyDown、onKeyUp或者onKeyMultiple来处理参数event所描述的键盘事件。

PhoneWindow的superDispatchKeyEvent最终还是调用了ViewGroup的dispatchKeyEvent处理，处理方式和ViewGroup中的dispatchKeyEventPreIme类似。dispatchKeyEvent会调用到View类中的dispatchKeyEvent，然后又调用了KeyEvent类的dispatch，该dispatch用于分发event事件，最后被点击的TextView就可以获得这个键盘事件，TextView重写了父类的onKeyDown方法，里面会调用doKeyDown进行焦点获取等处理。
