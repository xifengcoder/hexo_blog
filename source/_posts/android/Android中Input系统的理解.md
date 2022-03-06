---
title: Input子系统的理解
urlname: android_input
date: 2021-05-01 16:16:54
tags: Android
categories: Android
description: 自底向上来分析Android Input子系统的框架及事件分发细节。
---

​    大概在2015至2016年那会，我主要负责云真机项目的开发，可以在Web端实时显示手机画面、并可以通过鼠标实现手机Touch事件的精准输入。当时曾对Android手机的Input子系统做过深入的研究，但是一直没有公开发表出来，这篇文章的大部分内容就是当时所写的。

### 一、Input系统介绍

1. framework层的InputManagerService服务实现了WatchDog.Monitor接口，每隔30s会wake一下native层的InputDispatcher的Looper线程；
2. notifyMotion(const NotifyMotionArgs* args)利用传入的参数args构建一个MotionEntry实例，插入到mInboundQueue(Queue类型)队列中，如果插入前该队列为空, 则需要wake一下Looper线程。 注意：日志显示，正常的手指滑动时，该队列一直为空，因此一直需要执行wake操作。

### 二、InputManage的启动
#### 1. SystemServer
```java
[/frameworks/base/services/java/com/android/server/SystemServer.java]

Override
public void run() {
    ...
    WindowManagerService wm = null;
    DisplayManagerService display = null;
    InputManagerService inputManager = null
    ...
    try {
        ...

        //创建wmHandler
        HandlerThread wmHandlerThread = new HandlerThread("WindowManager");
        wmHandlerThread.start();
        Handler wmHandler = new Handler(wmHandlerThread.getLooper());
        wmHandler.post(new Runnable() {
            android.os.Process.setThreadPriority(
                    android.os.Process.THREAD_PRIORITY_DISPLAY);
            android.os.Process.setCanSelfBackground(false);
        }

        ...
        /* InputManager的初始化 */
        inputManager = new InputManagerService(context, wmHandler);
        ...

        /** 添加InputManagerService到Servicemanager中 */
        ServiceManager.addService(Context.INPUT_SERVICE, inputManager);

        ActivityManagerService.self().setWindowManager(wm);
        inputManager.setWindowManagerCallbacks(wm.getInputMonitor());

        /** InputManager的启动 */
        inputManager.start();

        display.setWindowManager(wm);
        display.setInputManager(inputManager);
        ...
    } catch (RuntimeException e) {
        ...
    }
}
```
#### 2.  Java层的InputManagerService
```java
[frameworks/base/services/core/java/com/android/server/input/InputManagerService.java]

public class InputManagerService extends IInputManager.Stub
        implements Watchdog.Monitor, DisplayManagerService.InputManagerFuncse {
    private final Context mContext;
    private final InputManagerHandler mHandler;

    //保存Native层的InputManager的指针
    private final long mPtr;
    ...

    //native函数
    private static native int nativeInit(InputManagerService service,
        Context context, MessageQueue messageQueue);
    private static native void nativeStart(int ptr);

    //构造函数
    public InputManagerService(Context context, Handler handler) {
        this.mContext = context;
        this.mHandler = new InputManagerHandler(handler.getLooper());
        ...
        mPtr = nativeInit(this, mContext, mHandler.getLooper().getQueue());
    }
    ...

    public void start() {
        //使用Native的InputManager指针做为参数，调用nativeStart()函数
        nativeStart(mPtr);

        // Add ourself to the Watchdog monitors.
        Watchdog.getInstance().addMonitor(this);

        registerPointerSpeedSettingObserver();
        registerShowTouchesSettingObserver();

        mContext.registerReceiver(new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                updatePointerSpeedFromSettings();
                updateShowTouchesFromSettings();
            }
        }, new IntentFilter(Intent.ACTION_USER_SWITCHED), null, mHandler);

        updatePointerSpeedFromSettings();
        updateShowTouchesFromSettings();
    }
}
```
#### 3. JNI接口层
```c++
[frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp]

static jint nativeInit(JNIEnv* env, jclass clazz,
        jobject serviceObj, jobject contextObj, jobject messageQueueObj) {

    //通过传入的Java层MessageQueue参数获取Native的MessageQueue实例
    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(
        env, messageQueueObj);

    //使用Java层的InputManagerService实例参数、Context参数和Looper参数构造NativeInputManager实例；
    NativeInputManager* im = new NativeInputManager(contextObj, serviceObj,
         messageQueue->getLooper());

    im->incStrong(serviceObj);
    return reinterpret_cast<jint>(im);
}

static void nativeStart(JNIEnv* env, jclass clazz, jint ptr) {
    NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);

    status_t result = im->getInputManager()->start();
    if (result) {
        jniThrowRuntimeException(env, "Input manager could not be started.");
    }
}
```
#### Native层的InputManager（重点）
1. InputManage构造函数的参数是：EventHub、ReaderPolicy和DispatcherPolicy。
2. InputManager启动了两个线程：InputReaderThread和InputDispatcherThread。
    InputReaderThread封装了InputReader类实例，启动线程时会循环调用InputReader的loopOnce()函数；
    InputDispatcherThread封装了InputDispatcher实例，启动线程时会循环调用InputDispatcher的threadLoop()函数；
    Inputdispatcher构造函数的参数是DispatcherPolicy；InputReader构造函数的参数是EventHub、ReaderPolicy和InputDispatcher。
```c++
[frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp]

class NativeInputManager : public virtual RefBase,
    public virtual InputReaderPolicyInterface,
    public virtual InputDispatcherPolicyInterface,
    public virtual PointerControllerPolicyInterface {
public:
    NativeInputManager(jobject contextObj, jobject serviceObj,
            const sp<Looper>& looper);
    inline sp<InputManager> getInputManager() const { return mInputManager; }
    ...
private:
    sp<InputManager> mInputManager;

}

NativeInputManager::NativeInputManager(jobject contextObj,
        jobject serviceObj, const sp<Looper>& looper) :
        mLooper(looper) {
    JNIEnv* env = jniEnv();

    mContextObj = env->NewGlobalRef(contextObj);
    mServiceObj = env->NewGlobalRef(serviceObj);

    {
        AutoMutex _l(mLock);
        mLocked.systemUiVisibility =
            ASYSTEM_UI_VISIBILITY_STATUS_BAR_VISIBLE;
        mLocked.pointerSpeed = 0;
        mLocked.pointerGesturesEnabled = true;
        mLocked.showTouches = false;
    }

    sp<EventHub> eventHub = new EventHub();
    mInputManager = new InputManager(eventHub, this, this);
}
```

###  三、应用程序请求注册监听事件  

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view;
            ...
            requestLayout();
            //创建InputChannel实例，创建后的InputChannel是未初始化的，可以通过从Parcel对象中读取信息初始化, 或者调用transferTo(InputChannel outParameter)从另一个InputChannel实例中获取初始化信息。
            if ((mWindowAttributes.inputFeatures
                    & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                mInputChannel = new InputChannel();
            }

            try {
                ...
                //addToDisplay()方法会调用WindowManagerService的addWindow()方法。
                res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(),
                        mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                        mAttachInfo.mOutsets, mInputChannel);
            } catch (RemoteException e) {
                ...
                mView = null;
                mInputChannel = null;
                ...
                throw new RuntimeException("Adding window failed", e);
            } finally {
                ...
            }

            if (mInputChannel != null) {
                if (mInputQueueCallback != null) {
                    mInputQueue = new InputQueue();
                    mInputQueueCallback.onInputQueueCreated(mInputQueue);
                }

                //创建WindowInputEventReceiver实例，使用前面创建好的InputChannel实例作参数。
                /**
             * 创建InputEventReceiver实例，参数有两个：
             * （1）InputChannel实例：前面创建好的inputChannel[1];
             * （2）Looper实例：与当前线程关联的Looper实例，它包含了一个MessageQueue实例、一个Thread实例(指向当前线程)；
             1. InputEventReceiver的通过jni完成；
             2. 在jni中构建一个NativeInputEventReceiver实例，参数有：
                 receiverWeak——new WeakReference<InputEventReceiver>(this)；//从Java层的InputEventReceiver封装而来；
                 InputChannel——传入的mInputChannel实参；
                 MessageQueue——从looper的getQueue获取；
                 初始化初始化的成员变量有：
                    mReceiverWeakGlobal赋值为receiverWeak；
                    使用inputChannel作参数构建mInputConsumer实例；
                    mMessageQueue赋值为messageQueue；
                    mBatchedInputEventPending为false;
                    mFdEvents初始化为0。
             3. 将inputChannel[1]的socketpair的fd添加到与当前线程关联的Looper实例的监听队列中。如果fd上有事件发生，则会回调
                NativeInputEventReceiver实例的handleEvent(int receiveFd，int events, void* data)函数。
                在handleEvents()函数中会调用consumeEvents(env, false, -1, NULL)。


                 执行NativeInputEventReceiver的initialize()函数, 即调用setFdEvents(ALOOPER_EVENT_NPUT)。所做的工作是：
                 设置mFdEvents为ALOOPER_EVENT_NPUT；
                 从mInputConsumer中getChannel()->getFd()获取fd，然后调用：
                 mMessageQueue->getLooper()->addFd(fd, 0, events, this, NULL);
            */
                mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                        Looper.myLooper());
            }
        }
    }
}

public int addWindow(Session session, IWindow client, int seq,
        WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
        Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
        InputChannel outInputChannel) {
    ...
    synchronized(mWindowMap) {
        ...
        if (outInputChannel != null && (attrs.inputFeatures
                & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
            String name = win.makeInputChannelName();
            //创建名为name的socketpair实例，将socket[0](服务端)注册到InputManagerService中，
            //将socket[1](客户端)传递给应用进程。
            InputChannel[] inputChannels = InputChannel.openInputChannelPair(name);
            win.setInputChannel(inputChannels[0]);
            inputChannels[1].transferTo(outInputChannel);

            mInputManager.registerInputChannel(win.mInputChannel, win.mInputWindowHandle);
        }
    }

    return res;
}
```



在ViewRootImpl类的setView()方法中，如果还没有创建过InputChannel，即WindowManager.LayoutParams属性不包含INPUT_FEATURE_NO_INPUT_CHANNEL，则会创建一个InputChannel类的实例。

对应的C++层代码：
```c++
status_t NativeInputEventReceiver::consumeEvents(JNIEnv* env,
        bool consumeBatches/*false*/, nsecs_t frameTime/*-1*/, bool* outConsumedBatch/*NULL*/) {
    bool skipCallbacks = false;
    for (;;) {
        uint32_t seq;
        InputEvent* inputEvent;
        status_t status = mInputConsumer.consume(&mInputEventFactory,
                consumeBatches/*false*/, frameTime/*-1*/, &seq, &inputEvent);
    }
}

[/frameworks/native/libs/input/InputTransport.cpp]
status_t InputConsumer::consume(InputEventFactoryInterface* factory,
        bool consumeBatches/*false*/, nsecs_t frameTime/*-1*/, uint32_t* outSeq, InputEvent** outEvent) {
    *outSeq = 0;
    *outEvent = NULL;
    while (!*outEvent) {
        status_t result = mChannel->receiveMessage(&mMsg);
        ...
        switch (mMsg.header.type) {
        case AINPUT_EVENT_TYPE_MOTION: {
            MotionEvent* motionEvent = factory->createMotionEvent();
            if (! motionEvent) return NO_MEMORY;

            *outSeq = mMsg.body.motion.seq;
            *outEvent = motionEvent;
            break;
        default:
            return UNKNOWN_ERROR;
    }
    return OK;

}
```

#### Session

```java
[frameworks/base/services/core/java/com/android/server/wm/Session.java]  

@Override
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,InputChannel outInputChannel) {
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId, outContentInsets, outStableInsets, outInputChannel);
}

@Override
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,InputChannel outInputChannel) {
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId, outContentInsets, outStableInsets, outInputChannel);
}
```

​    其实调用的是WindowsManagerService的addWindow()方法。在addView()方法中会创建一个Input Channel Pair，其中Server Channel 为WindowManagerService, Client Channel为ViewRootImpl。  

​      InputChannel类提供了openInputChannelPair()方法，用于创建一个input channel pair实例，第1个input channel供Input Dispatcher使用，用于pulish输入事件；第2个input channel供Application的输入队列使用，用于consume输入事件。  

  [frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java]  

```java
public class WindowManagerService extends IWindowManager.Stub
    implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {
    ...
    final InputManagerService mInputManager;
    ...
    //Constructor
    private WindowManagerService(Context context, InputManagerService inputManager,
            boolean haveInputMethods, boolean showBootMsgs, boolean onlyCore) {
        ...
        mInputManager = inputManager; // Must be before createDisplayContentLocked.
    }

    public int addWindow(Session session, IWindow client, int seq,
            WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
            Rect outContentInsets, Rect outStableInsets, InputChannel outInputChannel) {
        ...
        WindowState win = null;
        ...

        synchronized(mWindowMap) {
            ...
            win = new WindowState(this, session, client, token,
                attachedWindow, appOp[0], seq, attrs, viewVisibility, displayContent);

            if (outInputChannel != null && (attrs.inputFeatures
                    & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                String name = win.makeInputChannelName();
                InputChannel[] inputChannels = InputChannel.openInputChannelPair(name);
                win.setInputChannel(inputChannels[0]);
                inputChannels[1].transferTo(outInputChannel);
                mInputManager.registerInputChannel(win.mInputChannel, win.mInputWindowHandle);
            }
        }
    }//addWindow

}// class WindowManagerSerivice
```


### InputDispatcher

```c++
EventEntry* mPendingEvent; //初始值为NULL, 在每次loopOnce()循环中，如果mPendingEvent为NULL, 则当InboundQueue队列不为NULL时，从队列头取出一个元素赋值给mPendingEvent; 然后，在mPendingEvent处理完成后，重置mPendingEvent为NULL。 下一次loopOnce()循环重复这一过程。

 Vector<sp<InputWindowHandle> > mWindowHandles; //当前界面所有InputWindowHandle的列表，用于在分发事件时从中查找事件落在了哪个InputWindowHandle范围内(见findTouchedWindowTargetsLocked()函数)。

 Queue mInboundQueue; //事件通知者(InputReader)发过来的事件存放在该队列中。每处理一个事件就会从队列头取出一个元素。 
Queue mCommandQueue; //在postCommandLocked(Command command)函数中会构建一个CommandEntry实例，插入mCommandQueue队列尾部。 在runCommandsLockedInterruptible()函数中会逐一取出队列中的CommandEntry实例，执行相应的命令，直至mCommandQueue为空。 

KeyedVector<int, sp<Connection> > mConnectionsByFd;  //Key为InputChannel持有的文件fd， Value为Connection实例的指针。
```

键值对的插入是在： 

```c++
status_t InputDispatcher::registerInputChannel(const sp& inputChannel, const sp& inputWindowHandle, bool monitor) {
    ...    {
        // acquire lock
        AutoMutex _l(mLock);     
        if (getConnectionIndexLocked(inputChannel) >= 0) {
            //该inputChannel已经注册过。
            return BAD_VALUE;   
        }     
        sp connection = new Connection(inputChannel, inputWindowHandle, monitor); 
        int fd = inputChannel->getFd();    
        mConnectionsByFd.add(fd, connection);     
        if (monitor) {
            mMonitoringChannels.push(inputChannel);    
        }     
        mLooper->addFd(fd, 0, ALOOPER_EVENT_INPUT, handleReceiveCallback, this);    
    } // release lock     
    // Wake the looper because some connections have changed.    
    mLooper->wake();    
    return OK; 
}
```

键值对的删除操作：
```c++
status_t InputDispatcher::unregisterInputChannelLocked(const sp& inputChannel,    bool notify) {
    ssize_t connectionIndex = getConnectionIndexLocked(inputChannel);
    if (connectionIndex < 0) {        
        //该InputChannel已经解注册过。        
    return BAD_VALUE;    
    }     
    sp connection = mConnectionsByFd.valueAt(connectionIndex);
    mConnectionsByFd.removeItemsAt(connectionIndex);
    if (connection->monitor) {
        removeMonitorChannelLocked(inputChannel);
    }
    mLooper->removeFd(inputChannel->getFd());
    nsecs_t currentTime = now();
    abortBrokenDispatchCycleLocked(currentTime, connection, notify);
    connection->status = Connection::STATUS_ZOMBIE;
    //注意connection指针的释放，由sp完成。
    return OK;
}
```

### InputChannel的创建 	

#### 1.  InputPublisher
```c++
class InputPublisher {
public:
    //创建一个与指定InputChannel关联的InputPublisher实例。
    explicit InputPublisher(const sp<InputChannel>& channel);

    ~InputPublisher();

    inline sp<InputChannel> getChannel() { return mChannel; }

    /** @brief 发布一个key event事件到input channel中，返回值有：
     *  OK(0):  发布成功；
     *  WOULD_BLOCK: 见InputChannel的sendMessage()函数；
     *  DEAD_OBJECT: 见InputChannel的sendMessage()函数； 
     *  BAD_VALUE: seq为0；
     *  其它错误。 */
    status_t publishKeyEvent(uint32_t seq, int32_t deviceId, int32_t source, int32_t action, int32_t flags, 
            int32_t keyCode, int32_t scanCode, int32_t metaState, int32_t repeatCount,
            nsecs_t downTime, nsecs_t eventTime);

    /** @brief 发布一个motion event事件到input channel中， 返回值以下：
     *  OK(0): 发布成功；
     *  WOULD_BLOCK: 见InputChannel的sendMessage()函数；
     *  DEAD_OBJECT: 见InputChannel的sendMessage()函数；
     *  BAD_VALUE: seq为0，或者pointerCound<1，或者pointerCound>MAX_POINTERS;
     *  其它错误。*/
    status_t publishMotionEvent(
            uint32_t seq, int32_t deviceId, int32_t source, int32_t action, int32_t actionButton,
            int32_t flags, int32_t edgeFlags, int32_t metaState, int32_t buttonState,
            float xOffset, float yOffset, float xPrecision, float yPrecision,
            nsecs_t downTime, nsecs_t eventTime, uint32_t pointerCount,
            const PointerProperties* pointerProperties, const PointerCoords* pointerCoords);

    /** @brief 接收Consumer发挥的finished信号, 如果成功接收，则outSeq保存消息序列号，
     *  outHandled保存Consumer是否已处理该消息。 
     **/
    status_t receiveFinishedSignal(uint32_t* outSeq, bool* outHandled);

private:
    sp<InputChannel> mChannel;
};
```
#### 2. InputChannel
```c++
class InputChannel : public RefBase {
protected:
    virtual ~InputChannel();

public:
    InputChannel(const String8& name, int fd);

    //创建一对InputChannel，创建成功时返回OK(0)。
    static status_t openInputChannelPair(const String8& name,
            sp<InputChannel>& outServerChannel, sp<InputChannel>& outClientChannel);

    inline String8 getName() const { return mName; }
    inline int getFd() const { return mFd; }

    /** 
     * 发送数据，返回值有以下几种：
     * OK(0): 发送成功；
     * WOULD_BLOCK(-EWOULDBLOCK)：如果errno为EAGAIN或EWOULDBLOCK，则返回WOULD_BLOCK，表示发送缓冲区已满；     
     * DEAD_OBJECT(-EPIPE): 如果errno为EPIPE或ENOTCONN或ECONNREFUSED或ECONNRESET，则返回DEAD_OBJECT, 表示对方socket已关闭；
     * 其它错误(-errno)。      
     */
    status_t sendMessage(const InputMessage* msg);

    /** 
     * 接收数据，返回值有以下几种：
     * OK(0): 接收成功；
     * WOULD_BLOCK(-EWOULDBLOCK)：如果errno为EAGAIN或EWOULDBLOCK，则返回WOULD_BLOCK，表示没有收到任何数据；     
     * DEAD_OBJECT(-EPIPE): 如果errno为EPIPE或ENOTCONN或ECONNREFUSED，或读取数据长度为0，则返回DEAD_OBJECT；
     * 其它错误(-errno)。       
     */
    status_t receiveMessage(InputMessage* msg);

    //返回有相同fd的一个新的InputChannel实例。
    sp<InputChannel> dup() const;

private:
    String8 mName;
    int mFd;
};
```



#### 3. Connection

Connection与唯一的一个InputChannel对应，用来管理事件派发的状态。
```c++
class Connection : public RefBase { 
protected: 
    virtual ~Connection();
    
public:
    enum Status {
        STATUS_NORMAL, 
        STATUS_BROKEN, //发生不可恢复的通信错误；
        STATUS_ZOMBIE //InputChannel已经解注册；
    };

    Status status;
    sp<InputChannel> inputChannel;
    sp<InputWindowHandle> inputWindowHandle;
    bool monitor;
    InputPublisher inputPublisher;
    InputState inputState;

    bool inputPublisherBlocked;

    //即将要发送给Consumer的事件，存放在该队列中。
    Queue<DispatchEntry> outboundQueue;

    // Queue of events that have been published to the connection but that have not
    // yet received a "finished" response from the application.
    //已经发送给Consumer的、但还没有从Consumer接收到finished信号的事件，存放在在该队列中。
    Queue<DispatchEntry> waitQueue;

    explicit Connection(const sp<InputChannel>& inputChannel,
            const sp<InputWindowHandle>& inputWindowHandle, bool monitor);

    inline const char* getInputChannelName() const { return inputChannel->getName().string(); }

    const char* getWindowName() const;
    const char* getStatusLabel() const;

    DispatchEntry* findWaitQueueEntry(uint32_t seq);
};
```
#### 4. InputTarget
InputTarget用来描述接收input event事件的特定窗口(Window)。
```c++
struct InputTarget {
    enum {
        FLAG_FOREGROUND = 1 << 0,
        FLAG_WINDOW_IS_OBSCURED = 1 << 1,
        FLAG_SPLIT = 1 << 2,
        FLAG_ZERO_COORDS = 1 << 3,
        FLAG_DISPATCH_AS_IS = 1 << 8,
        FLAG_DISPATCH_AS_OUTSIDE = 1 << 9, 
        FLAG_DISPATCH_AS_HOVER_ENTER = 1 << 10,
        FLAG_DISPATCH_AS_HOVER_EXIT = 1 << 11,
        FLAG_DISPATCH_AS_SLIPPERY_EXIT = 1 << 12,
        FLAG_DISPATCH_AS_SLIPPERY_ENTER = 1 << 13,
        FLAG_DISPATCH_MASK = FLAG_DISPATCH_AS_IS
                | FLAG_DISPATCH_AS_OUTSIDE
                | FLAG_DISPATCH_AS_HOVER_ENTER
                | FLAG_DISPATCH_AS_HOVER_EXIT
                | FLAG_DISPATCH_AS_SLIPPERY_EXIT
                | FLAG_DISPATCH_AS_SLIPPERY_ENTER,
    };

    // The input channel to be targeted.
    sp<InputChannel> inputChannel;

    // Flags for the input target.
    int32_t flags;

    // The x and y offset to add to a MotionEvent as it is delivered.(ignored for KeyEvents)
    float xOffset, yOffset;

    // Scaling factor to apply to MotionEvent as it is delivered.(ignored for KeyEvents)
    float scaleFactor;

    // The subset of pointer ids to include in motion events dispatched to this input target if FLAG_SPLIT is set.
    BitSet32 pointerIds;
};
```
#### 5. EventHub

  EventHub的重点是getEvents()函数。  

```c++
class EventHub {    
    size_t EventHub::getEvents(int timeoutMillis, RawEvent* buffer, 
			size_t bufferSize) {
		...
		AutoMutex _l(mLock);
		struct input_event readBuffer[bufferSize];
		RawEvent* event = buffer;
		size_t capacity = bufferSize;
		bool awoken = false;
		for (;;) {
	    	nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
		    //默认为false，在配置发生变化时重新打开设备;
	    	if (mNeedToReopenDevices) {
				mNeedToReopenDevices = false;
	        	closeAllDevicesLocked();
	        	mNeedToScanDevices = true;
	        	break;
			}
	
			//默认为false, 如果有设备正在关闭时为true;
	    	while (mClosingDevices) {
				Device* device = mClosingDevices;
				mClosingDevices = device->next;
	        	event->when = now;
	        	event->deviceId = device->id == mBuiltInKeyboardId ?BUILT_IN_KEYBOARD_ID : device->id;
	        	event->type = DEVICE_REMOVED;
	        	event += 1;
	        	delete device;
	        	mNeedToSendFinishedDeviceScan = true;
	        	if (--capacity == 0) {
	            	break;
	        	}
			}
	
			//默认为true,表示需要扫描设备；在开始扫描前立即将mNeedToScanDevices设为false;
			//扫描完成后将mNeedToSendFinishedDeviceScan设为true
			if (mNeedToScanDevices) {
	        	mNeedToScanDevices = false;
	        	scanDevicesLocked();
	        	mNeedToSendFinishedDeviceScan = true;
	    	}
	
			while (mOpeningDevices != NULL) {
	        	Device* device = mOpeningDevices;
	        	mOpeningDevices = device->next;
	        	event->when = now;
	        	event->deviceId = device->id == mBuiltInKeyboardId ? 
					0 : device->id;
	        	event->type = DEVICE_ADDED;
	        	event += 1;
	
	        	mNeedToSendFinishedDeviceScan = true;
	        	if (--capacity == 0) {
	            	break;
	        	}
	    	}//while
            
			//所有的设备都添加完后，最后添加一个类型为FINISHED_DEVICE_SCAN的event;
			if (mNeedToSendFinishedDeviceScan) {
	        	mNeedToSendFinishedDeviceScan = false;
	        	event->when = now;
	        	event->type = FINISHED_DEVICE_SCAN;
	        	event += 1;
	
	        	if (--capacity == 0) {
	            	break;
	        	}
	    	}
	
			bool deviceChanged = false;
	    	while (mPendingEventIndex < mPendingEventCount) {
				const struct epoll_event& eventItem = 
					mPendingEventItems[mPendingEventIndex++];
				
				//如果eventItem.data.u32等于EPOLL_ID_INOTIFY；
				if (eventItem.data.u32 == EPOLL_ID_INOTIFY) {
	            	if (eventItem.events & EPOLLIN) {
	                	mPendingINotify = true;
	            	}
	            	continue;
	        	}
				
				//eventItem.data.u32等于EPOLL_ID_WAKE，直接丢掉读取的数据；
				if (eventItem.data.u32 == EPOLL_ID_WAKE) {
	            	if (eventItem.events & EPOLLIN) {
	                	awoken = true;
	                	char buffer[16];
	                	ssize_t nRead;
	                	do {
	                    	nRead = read(mWakeReadPipeFd, buffer, 
								sizeof(buffer));
	                	} while ((nRead == -1 && errno == EINTR) ||
							 nRead == sizeof(buffer));
	            	}
	            	continue;
	        	}
	
				ssize_t deviceIndex = mDevices.indexOfKey(eventItem.data.u32);
				Device* device = mDevices.valueAt(deviceIndex);
	
				if (eventItem.events & EPOLLIN) {
					int32_t readSize = read(device->fd, readBuffer,
	                    sizeof(struct input_event) * capacity);
					if (readSize == 0 || (readSize < 0 && errno == ENODEV)) {
						deviceChanged = true;
	                	closeDeviceLocked(device);
					} else if (readSize < 0) {
						...
					} else if ((readSize % sizeof(struct input_event)) != 0) {
						int32_t deviceId = device->id == mBuiltInKeyboardId ? 
								0 : device->id;
						
	                	size_t count = size_t(readSize) / sizeof(struct input_event);
	                	for (size_t i = 0; i < count; i++) {
							const struct input_event& iev = readBuffer[i];
							event->when = now;
							event->deviceId = deviceId;
	                    	event->type = iev.type;
	                   	 	event->code = iev.code;
	                    	event->value = iev.value;
	                    	event += 1;
						}
						
						capacity -= count;
	                	if (capacity == 0) {
	                    	mPendingEventIndex -= 1;
	                    	break;
	                	}
					}
				} else if (eventItem.events & EPOLLHUP) {
					deviceChanged = true;
	            	closeDeviceLocked(device);
				} else {
					...
				}
			}//while
			
			if (mPendingINotify && mPendingEventIndex >= mPendingEventCount) {
	        	mPendingINotify = false;
	        	readNotifyLocked();
	        	deviceChanged = true;
	    	}
	
			if (deviceChanged) {
	        	continue;
	    	}
	
			//如果已经读到数据或者被显式唤醒；
			if (event != buffer || awoken) {
	        	break;
	    	}
	
			mPendingEventIndex = 0;
	
	    	mLock.unlock(); 
	    	release_wake_lock(WAKE_LOCK_ID);
	
			int pollResult = epoll_wait(mEpollFd, mPendingEventItems, 
					EPOLL_MAX_EVENTS, timeoutMillis);
			
			acquire_wake_lock(PARTIAL_WAKE_LOCK, WAKE_LOCK_ID);
	    	mLock.lock();
	
			if (pollResult == 0) {
	        	// Timed out.
	        	mPendingEventCount = 0;
	        	break;
	    	} 
	
			if (pollResult < 0) {
	        	mPendingEventCount = 0;
	        	if (errno != EINTR) {
	            	usleep(100000);
	        	}
	    	} else {
	        	mPendingEventCount = size_t(pollResult);
	    	}
		}//for(;;)
	
		return event - buffer;
	}//getEvents
	
	/**
	 * 扫描/dev/input目录，读取设备列表，并添加到mDevices中。
	 * 如果没有添加过Virtual KeyBoard, 则额外添加。
	 */
	void EventHub::scanDevicesLocked() {
		status_t res = scanDirLocked(DEVICE_PATH);
		if(res < 0) {
		    ALOGE("scan dir failed for %s\n", DEVICE_PATH);
		}
		if (mDevices.indexOfKey(VIRTUAL_KEYBOARD_ID) < 0) {
		    createVirtualKeyboardLocked();
		}
	}
	
	/**
	 * 添加路径为devicePath的设备到mDevices中，并为devicePath文件注册epoll监听函数。
	 */
	status_t EventHub::openDeviceLocked(const char *devicePath) {
		...
		int fd = open(devicePath, O_RDWR | O_CLOEXEC);
		...
	
		int32_t deviceId = mNextDeviceId++;
		Device* device = new Device(fd, deviceId, String8(devicePath),
	 			identifier);
		
		//注册epoll;
		struct epoll_event eventItem;
		memset(&eventItem, 0, sizeof(eventItem));
		eventItem.events = EPOLLIN;
		eventItem.data.u32 = deviceId;
		if (epoll_ctl(mEpollFd, EPOLL_CTL_ADD, fd, &eventItem)) {
			delete device;
	    	return -1;
		}
		
		...
		addDeviceLocked(device);
		return 0;
	}//openDeviceLocked
	
	/**
	 * 将device添加到mDevices向量中；
	 */
	void EventHub::addDeviceLocked(Device* device) {
		mDevices.add(device->id, device);
		device->next = mOpeningDevices;
		mOpeningDevices = device;
	}
}
```

### SocketPair的创建：
```c++
[frameworks/native/libs/input/InputTransport.cpp]  

/*
 * @brief 创建socketpair, 将socket[0]赋予outServerChannel，socket[1]赋予outClientChannel。
 */
status_t InputChannel::openInputChannelPair(const String8& name,
        sp<InputChannel>& outServerChannel, sp<InputChannel>& outClientChannel) {
    int sockets[2];
    if (socketpair(AF_UNIX, SOCK_SEQPACKET, 0, sockets)) {
        status_t result = -errno;
        outServerChannel.clear();
        outClientChannel.clear();
        return result;
    }

    int bufferSize = SOCKET_BUFFER_SIZE;
    setsockopt(sockets[0], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[0], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[1], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[1], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));

    String8 serverChannelName = name;
    serverChannelName.append(" (server)");
    outServerChannel = new InputChannel(serverChannelName, sockets[0]);

    String8 clientChannelName = name;
    clientChannelName.append(" (client)");
    outClientChannel = new InputChannel(clientChannelName, sockets[1]);
    return OK;
}
```

### Looper线程处理

```c++
void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {

    if (! mPendingEvent) {

        if (mInboundQueue.isEmpty()) {

            if (mKeyRepeatState.lastKeyEntry) {
                if (currentTime >= mKeyRepeatState.nextRepeatTime) {
                    //为mPendingEvent赋值。
                    mPendingEvent = synthesizeKeyRepeatLocked(currentTime);
                } else {
                    if (mKeyRepeatState.nextRepeatTime < *nextWakeupTime) {
                        //更新nextWakeupTime。
                        *nextWakeupTime = mKeyRepeatState.nextRepeatTime;
                    }
                }
            }

            //如果mPendingEvent为空，直接返回。
            if (!mPendingEvent) {
                return;
            }
        } else {
            //如果mInboundQueue不为空, 则从mInboundQueue队列头部取出一个元素赋给mPendingEvent。
            mPendingEvent = mInboundQueue.dequeueAtHead();
        }
    }

    ALOG_ASSERT(mPendingEvent != NULL);
    bool done = false;

    switch (mPendingEvent->type) {
        ...
    case EventEntry::TYPE_MOTION:
        MotionEntry* typedEntry = static_cast<MotionEntry*>(mPendingEvent);
        done = dispatchMotionLocked(currentTime, typedEntry,
                &dropReason, nextWakeupTime);
        break;
    default:
        ALOG_ASSERT(false);
        break;
    }//switch

    if (done) {
        if (dropReason != DROP_REASON_NOT_DROPPED) {
            dropInboundEventLocked(mPendingEvent, dropReason);
        }
        mLastDropReason = dropReason;

        releasePendingEventLocked();
        *nextWakeupTime = LONG_LONG_MIN; 
    }//if
}

bool InputDispatcher::dispatchMotionLocked(
        nsecs_t currentTime, MotionEntry* entry, DropReason* dropReason, nsecs_t* nextWakeupTime) {

    if (! entry->dispatchInProgress) {
        entry->dispatchInProgress = true;
    }

    //是否需要drop该event。
    if (*dropReason != DROP_REASON_NOT_DROPPED) {
        setInjectionResultLocked(entry, *dropReason == DROP_REASON_POLICY
                ? INPUT_EVENT_INJECTION_SUCCEEDED : INPUT_EVENT_INJECTION_FAILED);
        return true;
    }

    bool isPointerEvent = entry->source & AINPUT_SOURCE_CLASS_POINTER;

    Vector<InputTarget> inputTargets;

    bool conflictingPointerActions = false;
    int32_t injectionResult;
    if (isPointerEvent) {
        //找到当前界面的的所有InputTargets，添加到向量表inputTargets中。经过调试，发现该InputTargets列表
        包含:NavigationBar、StatusBar、当前Activity(如com.android.browser/com.android.browser.BrowserActivity)和WindowManager四部分。
        // Pointer event.  (eg. touchscreen)
        injectionResult = findTouchedWindowTargetsLocked(currentTime,
                entry, inputTargets, nextWakeupTime, &conflictingPointerActions);
    } else {
        ...
    }

    if (injectionResult == INPUT_EVENT_INJECTION_PENDING) {
        return false;
    }

    setInjectionResultLocked(entry, injectionResult);
    if (injectionResult != INPUT_EVENT_INJECTION_SUCCEEDED) {
        ...
        return true;
    }

    dispatchEventLocked(currentTime, entry, inputTargets);
    return true;
}

void InputDispatcher::dispatchEventLocked(nsecs_t currentTime,
        EventEntry* eventEntry, const Vector<InputTarget>& inputTargets) {

    ALOG_ASSERT(eventEntry->dispatchInProgress); // should already have been set to true

    pokeUserActivityLocked(eventEntry);

    for (size_t i = 0; i < inputTargets.size(); i++) {
        const InputTarget& inputTarget = inputTargets.itemAt(i);
        ssize_t connectionIndex = getConnectionIndexLocked(inputTarget.inputChannel);
        if (connectionIndex >= 0) {
            sp<Connection> connection = mConnectionsByFd.valueAt(connectionIndex);
            prepareDispatchCycleLocked(currentTime, connection, eventEntry, &inputTarget);
        }
    }
}

void InputDispatcher::prepareDispatchCycleLocked(nsecs_t currentTime,
        const sp<Connection>& connection, EventEntry* eventEntry, const InputTarget* inputTarget) {
    if (connection->status != Connection::STATUS_NORMAL) {
        return;
    }

    // Split a motion event if needed.
    if (inputTarget->flags & InputTarget::FLAG_SPLIT) {
        //需要拆分该事件.
        ...
        return;
    }

    //不需求拆分事件。
    enqueueDispatchEntriesLocked(currentTime, connection, eventEntry, inputTarget);
}

void InputDispatcher::enqueueDispatchEntriesLocked(nsecs_t currentTime,
        const sp<Connection>& connection, EventEntry* eventEntry, const InputTarget* inputTarget) {
    bool wasEmpty = connection->outboundQueue.isEmpty();

    //以 FLAG_DISPATCH_AS_OUTSIDE 模式将eventEntry实例插入队列；FLAG_DISPATCH_AS_OUTSIDE表示
    //有个AMOTION_EVENT_ACTION_DOWN的MotionEvent落在了该Target的外面，因此会被以
    //AMOTION_EVENT_ACTION_OUTSIDE的形式传递给该Target。
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
            InputTarget::FLAG_DISPATCH_AS_OUTSIDE);

    //以FLAG_DISPATCH_AS_IS模式将eventEntry实例插入队列；FLAG_DISPATCH_AS_IS表示该MotionEvent落在该Target
    //的范围内。
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
            InputTarget::FLAG_DISPATCH_AS_IS);
    ...

    // If the outbound queue was previously empty, start the dispatch cycle going.
    if (wasEmpty && !connection->outboundQueue.isEmpty()) {
        startDispatchCycleLocked(currentTime, connection);
    }
}

void InputDispatcher::enqueueDispatchEntryLocked(
        const sp<Connection>& connection, EventEntry* eventEntry, const InputTarget* inputTarget,
        int32_t dispatchMode) {
    int32_t inputTargetFlags = inputTarget->flags;
    if (!(inputTargetFlags & dispatchMode)) {
        return;
    }

    DispatchEntry* dispatchEntry = new DispatchEntry(eventEntry, // increments ref
            inputTargetFlags, inputTarget->xOffset, inputTarget->yOffset,
            inputTarget->scaleFactor);

    switch (eventEntry->type) {
        MotionEntry* motionEntry = static_cast<MotionEntry*>(eventEntry);
        ...
        if (!connection->inputState.trackMotion(motionEntry,
                dispatchEntry->resolvedAction, dispatchEntry->resolvedFlags)) {
            delete dispatchEntry;
            return; // skip the inconsistent event
        }
        break;
    }//switch

    //将该DispatchEntry实例放入connection的outboundQueue队列中。
    connection->outboundQueue.enqueueAtTail(dispatchEntry);
}

void InputDispatcher::startDispatchCycleLocked(nsecs_t currentTime,
        const sp<Connection>& connection) {
    while (connection->status == Connection::STATUS_NORMAL
            && !connection->outboundQueue.isEmpty()) {
        DispatchEntry* dispatchEntry = connection->outboundQueue.head;
        dispatchEntry->deliveryTime = currentTime;

        // Publish the event.
        status_t status;
        EventEntry* eventEntry = dispatchEntry->eventEntry;

        switch (eventEntry->type) {
        case EventEntry::TYPE_KEY: {
            ...
            break;
        }
        case EventEntry::TYPE_MOTION: {
            MotionEntry* motionEntry = static_cast<MotionEntry*>(eventEntry);
            //发布消息，如果成功，则返回OK(0).
            // Publish the motion event.
            status = connection->inputPublisher.publishMotionEvent(dispatchEntry->seq,
                    motionEntry->deviceId, motionEntry->source,
                    dispatchEntry->resolvedAction, motionEntry->actionButton,
                    dispatchEntry->resolvedFlags, motionEntry->edgeFlags,
                    motionEntry->metaState, motionEntry->buttonState,
                    xOffset, yOffset, motionEntry->xPrecision, motionEntry->yPrecision,
                    motionEntry->downTime, motionEntry->eventTime,
                    motionEntry->pointerCount, motionEntry->pointerProperties,
                    usingCoords);
            break;
        }
        default:
            ALOG_ASSERT(false);
            return;
        }

        //错误处理。
        if (status) {
            if (status == WOULD_BLOCK) {
                if (connection->waitQueue.isEmpty()) {
                    abortBrokenDispatchCycleLocked(currentTime, connection, true /*notify*/);
                } else {
                    connection->inputPublisherBlocked = true;
                }
            } else {
                abortBrokenDispatchCycleLocked(currentTime, connection, true /*notify*/);
            }

            return;
        }

        //将该DispatchEntry实例从outboundQueue队列中，取出来放入waitQueue中。
        connection->outboundQueue.dequeue(dispatchEntry);
        connection->waitQueue.enqueueAtTail(dispatchEntry);
    }//while
}
```

