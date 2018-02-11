
[toc]


## 相关文件
1. Looper.java
2. Handler.java
3. MessageQueue.java
4. android_os_MessageQueue.cpp
5. Looper.cpp


## 流程图

![image](https://github.com/alexhe/droid/raw/master/files/Looper-1.png)

## Demo

### 通过HandlerThread演示Looper用法
```java
HandlerThread th1 =  new HandlerThread("HandlerThreadTest") {
    protected void onLooperPrepared() {
        synchronized (HandlerThreadTest.this) {
            mDidSetup = true;
        }
    }
};

th1.start();

final Handler h1 = new Handler(th1.getLooper()) {
    public void handleMessage(Message msg) {
        assertEquals(TEST_WHAT, msg.what);
    }
};
- 
Message msg = h1.obtainMessage(TEST_WHAT);
h1.sendMessage(msg);

```


### HandlerThread原理
```
public class HandlerThread extends Thread {
    Looper mLooper;
    private @Nullable Handler mHandler;

    protected void onLooperPrepared() {
    }

    public void run() {
        Looper.prepare();// #1
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        onLooperPrepared();
        Looper.loop(); // #2
    }
}
    
```

## 建立消息循环：prepare()和loop()
```java
public final class Looper {
   
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    private static Looper sMainLooper;

    final MessageQueue mQueue;
    final Thread mThread;


    
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));//建立消息队列
    }

    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);//重点
        mThread = Thread.currentThread();
    }
    
    public static void loop() {
        final Looper me = myLooper();
        final MessageQueue queue = me.mQueue;
        for (;;) {
            Message msg = queue.next(); // might block// #1
            if (msg == null) {
                return;
            }
            msg.target.dispatchMessage(msg);//#2      
            msg.recycleUnchecked();
        }
    }
    
    //主线程使用
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }

    public static Looper getMainLooper() {
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }
}

```

### MessageQueue初始化:nativeInit

```java

public final class MessageQueue {
    private final boolean mQuitAllowed;

    private long mPtr; // used by native code
    Message mMessages;

    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();//helin. java2native
    }
}
```

### jni初始化
```c++
android_os_MessageQueue.cpp

static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    //new 一个 NativeMessageQueue 对象
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();

    nativeMessageQueue->incStrong(env);
    return reinterpret_cast<jlong>(nativeMessageQueue);
}
```

### Native MessageQueue初始化
```c++
android_os_MessageQueue.cpp

NativeMessageQueue::NativeMessageQueue() :
        mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        mLooper = new Looper(false);//第一次进入这里
        Looper::setForThread(mLooper);
    }
}

```

### native Looper初始化，建立读机制 mWakeEventFd
```c++
Looper.cpp

Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mPolling(false), mEpollFd(-1), mEpollRebuildRequired(false),
        mNextRequestSeq(0), mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    mWakeEventFd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);//关键 fd

    AutoMutex _l(mLock);
    rebuildEpollLocked();
}
```

### 建立读的的机制：mEpollFd 上绑定 mWakeEventFd 等待读事件

```c++
Looper.cpp

void Looper::rebuildEpollLocked() {
    if (mEpollFd >= 0) {
        close(mEpollFd);
    }
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);//helin

    struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event));
    eventItem.events = EPOLLIN;//helin. wait read
    eventItem.data.fd = mWakeEventFd;
    // helin. mEpollFd 和 mWakeEventFd
    int result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, & eventItem);
}

```

### java层Looper prepare()后，开始进入loop()循环

```java

    public static void loop() {
        final Looper me = myLooper();
        final MessageQueue queue = me.mQueue;
        for (;;) {
            Message msg = queue.next(); // might block// #1
            if (msg == null) {
                return;
            }
            msg.target.dispatchMessage(msg);//#2      
            msg.recycleUnchecked();
        }
    }
```

### java层MessageQueue等待读取msg

```java

   MessageQueue.java

   Message next() {
        final long ptr = mPtr;//helin. mPtr refs for NATIVE
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            //helin. native pollonce, block
            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        mBlocked = false;//helin. 有msg，就不block了
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;//helin. 单向链表取出第一个item
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;//helin. 有消息的时候才返回
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }
            }
        }
    }

```

### MessageQueue的poll等待
```java
MessageQueue.java

Message next() {
    for (;;) {
        //helin. native pollonce, block
        nativePollOnce(ptr, nextPollTimeoutMillis);
    }
}
```
### 进入native的Looper的pollOnce()

```c++
android_os_MessageQueue.cpp

void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    mPollEnv = env;
    mPollObj = pollObj;
    mLooper->pollOnce(timeoutMillis);//helin. looper pollonce
    mPollObj = NULL;
    mPollEnv = NULL;

    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}
```

### native looper 轮询1次

```java
Looper.cpp

int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        result = pollInner(timeoutMillis);
    }
}
```

### native looper poll等到了读消息，唤醒

```c++
Looper.cpp

int Looper::pollInner(int timeoutMillis) {
    // We are about to idle.
    mPolling = true;

    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);//helin.

    // No longer idling.
    mPolling = false;

    // Acquire lock.
    mLock.lock();

    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeEventFd) {
            if (epollEvents & EPOLLIN) {
                awoken();//helin 有读数据了
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake event fd.", epollEvents);
            }
        }
    }
Done: ;
...
}
```

### 唤醒->java层MessageQueue退出阻塞，读取msg返回
```c++
Looper.cpp

void Looper::awoken() {
    uint64_t counter;
    //helin 退出阻塞
    TEMP_FAILURE_RETRY(read(mWakeEventFd, &counter, sizeof(uint64_t)));
}
```
### java looper得到msg后分派消息处理
```java
Looper.java

public static void loop() {
    final Looper me = myLooper();
    final MessageQueue queue = me.mQueue;

    for (;;) {
        Message msg = queue.next(); // might block
        msg.target.dispatchMessage(msg);
    }
}
```

## 处理消息

### 先处理 callback，再处理 handleMessage
```java
public class Handler {
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {// #1
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) { // #2
                    return;
                }
            }
            handleMessage(msg); //#3
        }
    }
}
```


## 发送消息
### Handler发送消息：
```java
Handler.java

public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
```

### msg进入消息队列：

```java
Handler.java

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}

```

### msg加入单向链表

```java
MessageQueue.java

boolean enqueueMessage(Message msg, long when) {
    synchronized (this) {
        if (mQuitting) {
            return false;
        }

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        //helin 消息加入单项链表
        if (p == null || when == 0 || when < p.when) {
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        //唤醒epoll
        if (needWake) {
            nativeWake(mPtr);//helin. wakeup native epoll. mWakeEventFd
        }
    }
    return true;
}
```

### 加入消息，唤醒poll
```java
boolean enqueueMessage(Message msg, long when) {
    //唤醒epoll
    if (needWake) {
        //helin. wakeup native epoll. mWakeEventFd
        nativeWake(mPtr);
    }
}
```

### native 调用looper唤醒
```c++
android_os_MessageQueue.cpp
void NativeMessageQueue::wake() {
    mLooper->wake();//helin. wakeup
}
```

### native looper对fd写入数据，唤醒
```c++
Looper.cpp

void Looper::wake() {
uint64_t inc = 1;
//helin. write any data to mWakeEventFd to wakeup fd.
ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
}
```

### 这样looper等待MessageQueue的阻塞就返回msg了

```java

    public static void loop() {
        final Looper me = myLooper();
        final MessageQueue queue = me.mQueue;
        for (;;) {
            Message msg = queue.next(); // might block// #1
            if (msg == null) {
                return;
            }
            msg.target.dispatchMessage(msg);//#2      
            msg.recycleUnchecked();
        }
    }
```


## Looper技术应用
### Looper卡顿调试
#### setSlowDispatchThresholdMs
```java
    /** {@hide} */
    public void setSlowDispatchThresholdMs(long slowDispatchThresholdMs) {
        mSlowDispatchThresholdMs = slowDispatchThresholdMs;
    }
    
```
#### 设置slowDispatchThresholdMs慢处理门限值

如果消息处理时间超过slowDispatchThresholdMs,就输出log
```java
public static void loop() {
...
    for (;;) {
        Message msg = queue.next(); // might block
        final long start = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
        final long end;
        try {
            msg.target.dispatchMessage(msg);
            end = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }
        if (slowDispatchThresholdMs > 0) {
            final long time = end - start;
            if (time > slowDispatchThresholdMs) {
                Slog.w(TAG, "Dispatch took " + time + "ms on "
                        + Thread.currentThread().getName() + ", h=" +
                        msg.target + " cb=" + msg.callback + " msg=" + msg.what);
            }
        }
    }
}
```

### Looper循环Log
#### setMessageLogging
```java
    public void setMessageLogging(@Nullable Printer printer) {
        mLogging = printer;
    }
```

#### 打开msg循环log

Looper.myLooper().setMessageLogging(new LogPrinter(Log.INFO, "HAHA_LOOPER"));

```java
public static void loop() {
...
    for (;;) {
        final Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }

        try {
            msg.target.dispatchMessage(msg);
            end = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }


        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }
    }
}
```

### looper循环systrace
#### setTraceTag
```java
    /** {@hide} */
    public void setTraceTag(long traceTag) {
        mTraceTag = traceTag;
    }
```

#### systrace 输出
```java
public static void loop() {
...
    for (;;) {
        Message msg = queue.next(); // might block
        final long traceTag = me.mTraceTag;
        if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
            Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
        }
        try {
            msg.target.dispatchMessage(msg);
            end = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }
    }
}
```
