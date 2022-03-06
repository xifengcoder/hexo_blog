---
title: Looper类的设计
urlname: android_looper
date: 2021-05-01 16:16:54
tags: Android
categories: Android
description: Looper的底层是用C++实现的，本文主要分析Looper的C++层的设计与实现。
---

### 1. Looper类的设计

1. 没有事件发生时， 每隔30s pollOnce()会被wake函数唤醒一次，timeoutMillis = -1;
2. Looper主线程负责循环；
3. 添加、删除fd的操作在system_server其他线程中完成；

[/system/core/include/utils/Looper.h]

```c++
/**
 * @param timeoutMillis 超时时间(ms), 在事件驱动循环中这个timoutMillis参数是动态计算的。
 *     timeoutMillis = 0，立即返回；
 *     timeoutMillis < 0，无限等待直到有event到来；
 * @return 返回值有以下几种：
 *     POLL_WAKE(-1): 在timeoutMillis消耗完之前wake()被调用；
 *     POLL_CALLBACK(-2): 由一个或多个callback函数被调用；
 *     POLL_TIMEOUT(-3):在timeoutMillis消耗完之前没有数据到来；
 *     POLL_ERROR(-4): 出现错误。
 *     >= 0: 有事件发生、且该事件的fd没有callback函数时，返回该fd携带的ident，即addFd()函数传入的indent; 
 *           当且仅当此时，outFd、outEvents和outData参数包含与该fd相关的信息； 
 *           除此之外，outFd和outEvents被赋值0, *out被赋值NULL(如果这些实参非NULL的话)。
 */
int pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData);

/**
 * @brief: 异步的唤醒poll;
 *         该函数可以在任一thread中被调用，而且立即返回。
 */
void wake();

/**
 * @brief:
 * @param fd：要添加的fd;
 * @param ident: 该事件的标识(identifier),该值要么是POLL_CALLBACK，要么必须>=0;
 * @param events: 
 * @param callback: 当该fd上有事件发生时，要调用的函数；
 * @data: callback函数的参数。
 */
int addFd(int fd, int ident, int events, const sp<LooperCallback>& callback, void* data);

/**
 * @brief 从Looper中移除以前添加过的fd，该函数可以在任一线程中调用。
 * @return 1: 该fd成功移除；
 *         0: 该fd未曾注册过。
 */
int removeFd(int fd);

/**
 * @brief: 入列一个Message，等待特定的handler来处理, 该函数可以在任一线程中调用。
 * @param handler: handler必须非空。
 * /
void sendMessage(const sp<MessageHandler>& handler, const Message& message);
```

### 2. Looper类的成员变量

```c++
const bool mAllowNonCallbacks;//表示是否允许空的CallBack出现, 不可改变。
int mWakeEventFd; //在构造函数中赋值。

Mutex mLock;
KeyedVector<int, Request> mRequests; //由mLock保护
int mEpollFd; //有mLock保护，但是只在looper线程中被修改.
bool mEpollRebuildRequired; //由mLock保护.

int mNextRequestSeq; //下一个要请求添加的文件描述符序号。
size_t mResponseIndex;

//在pollOnce函数中被修改， 调用epoll_wait()前被置true， epoll_wait()返回后立即被置false，无需加锁。
volatile bool mPolling; 
Vector<Response> mResponses; //只在pollOnce()函数中被修改。
```

### 3. Looper内部数据结构

```c++
enum {
    /**
     * Result from Looper_pollOnce() and Looper_pollAll():
     * The poll was awoken using wake() before the timeout expired
     * and no callbacks were executed and no other file descriptors were ready.
     */
    POLL_WAKE = -1,

    /**
     * Result from Looper_pollOnce() and Looper_pollAll():
     * One or more callbacks were executed.
     */
    POLL_CALLBACK = -2,

    /**
     * Result from Looper_pollOnce() and Looper_pollAll():
     * The timeout expired.
     */
    POLL_TIMEOUT = -3,

    /**
     * Result from Looper_pollOnce() and Looper_pollAll():
     * An error occurred.
     */
    POLL_ERROR = -4,
};
enum {
    EVENT_INPUT = 1 << 0,
    EVENT_OUTPUT = 1 << 1,
    EVENT_ERROR = 1 << 2,
    EVENT_HANGUP = 1 << 3,
    EVENT_INVALID = 1 << 4,
};

class LooperCallback : public virtual RefBase {
protected:
    virtual ~LooperCallback() { }
public:

    /** handleEvent(返回0表示从Looper中解除该fd; 返回1表示继续接受监听。*/
    virtual int handleEvent(int fd, int events, void* data) = 0;
};
struct Request {
    int fd;
    int ident;
    int events; 
    int seq;
    sp<LooperCallback> callback;
    void* data;

    /** 使用Request的成员变量初始化epoll_event实例。*/
    void initEventItem(struct epoll_event* eventItem) const;
};

struct Response {
    int events; //值为EVENT_INPUT、EVENT_OUTPUT、EVENT_ERROR和EVENT_HANGUP中的0、1或多个。
    Request request;
};
```

### 4. Looper的实现

```c++
[/system/core/libutils/Looper.cpp]

/**
 * @brief Looper的构造函数中做了以下事情：
 * 1. 创建事件通知的非阻塞文件描述符mWakeEventFd；
 * 2. 重新创建epoll文件描述符mEpollFd，添加mWakeEventFd、mRequest列表中已有fd集合到epoll监听队列中。
 */
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mPolling(false), mEpollFd(-1), mEpollRebuildRequired(false),
        mNextRequestSeq(0), mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    mWakeEventFd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);

    AutoMutex _l(mLock);
    rebuildEpollLocked();
}

void Looper::rebuildEpollLocked() {
    if (mEpollFd >= 0) {
        close(mEpollFd);
    }

    mEpollFd = epoll_create(EPOLL_SIZE_HINT);

    struct epoll_event eventItem;
    memset(&eventItem, 0, sizeof(epoll_event));
    eventItem.events = EPOLLIN;
    eventItem.data.fd = mWakeEventFd;
    int result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, & eventItem);

    for (size_t i = 0; i < mRequests.size(); i++) {
        const Request& request = mRequests.valueAt(i);
        struct epoll_event eventItem;
        request.initEventItem(&eventItem);

        int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, request.fd, & eventItem);
        if (epollResult < 0) {
            ALOGE("...");
        }
    }//for
}

/**
 * @brief: 先调用epoll_ctrl()对fd作添加或修改操作, 然后构建请求信息(struct Request)，以fd为索引，添加到mRequests中。
 * @param fd：要添加的fd;
 * @param ident: 该事件的标识(identifier), 由pollOnce()函数返回得来；该值要么是POLL_CALLBACK，要么必须>=0;
 * @param events: 
 * @param callback: 
     当该fd上有事件发生时，要调用的函数；
 *   callback为NULL时, mAllowNonCallbacks必须为true且ident必须>0, 否则return -1；
 *   callback不为NULL时，实参ident值可忽略, 会被重新赋值为POLL_CALLACK。 
 * @data: callback函数的参数。
 * @return: 添加成功, 返回1；添加失败, 返回-1。
 */
int Looper::addFd(int fd, int ident, int events, const sp<LooperCallback>& callback, void* data) {
    if (!callback.get()) {
        //mAllowNonCallbacks为false表示不允许空的CallBack出现。
        if (! mAllowNonCallbacks) {
            return -1;
        }

        if (ident < 0) {
            return -1;
        }
    } else { //callback不为NULL.
        ident = POLL_CALLBACK;
    }

    { // acquire lock
        AutoMutex _l(mLock);

        Request request;
        request.fd = fd;
        request.ident = ident;
        request.events = events;
        request.seq = mNextRequestSeq++;
        request.callback = callback;
        request.data = data;
        if (mNextRequestSeq == -1) mNextRequestSeq = 0;

        struct epoll_event eventItem;
        request.initEventItem(&eventItem);

        ssize_t requestIndex = mRequests.indexOfKey(fd);
        if (requestIndex < 0) {
            //mRequests中不存在fd.
            int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, fd, & eventItem);
            if (epollResult < 0) {
                return -1;
            }
            mRequests.add(fd, request);
        } else {
            //mRequests中已经存在fd.
            int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_MOD, fd, & eventItem);
            if (epollResult < 0) {
                //如果是ENOENT错误，则尝试重新添加。
                if (errno == ENOENT) {
                    epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, fd, & eventItem);
                    if (epollResult < 0) {
                        return -1;
                    }
                    scheduleEpollRebuildLocked();
                } else {
                    return -1;
                }
            }
            mRequests.replaceValueAt(requestIndex, request);
        }
    } // release lock

    return 1;
}

/**
 * @brief 从Map集合mRequests中移除文件描述符fd, 如果参数seq不等于-1, 则需要校验Request对象中的seq是否与实参seq一致。
 */
int Looper::removeFd(int fd, int seq) {
    { // acquire lock
        AutoMutex _l(mLock);
        ssize_t requestIndex = mRequests.indexOfKey(fd);
        if (requestIndex < 0) {
            return 0;
        }

        if (seq != -1 && mRequests.valueAt(requestIndex).seq != seq) {
            return 0;
        }

        mRequests.removeItemsAt(requestIndex);

        int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_DEL, fd, NULL);
        if (epollResult < 0) {
            if (seq != -1 && (errno == EBADF || errno == ENOENT)) {
                scheduleEpollRebuildLocked();
            } else {
                scheduleEpollRebuildLocked();
                return -1;
            }
        }
    } // release lock

    return 1;
}


/**
 * @brief 将mEpollRebuildRequired设为true, 并且调用wake()函数，使得rebuildEpollLocked()函数能马上执行。
 */
void Looper::scheduleEpollRebuildLocked() {
    if (!mEpollRebuildRequired) {
        mEpollRebuildRequired = true;
        wake();
    }
}


/**
 * @brief 往mWakeEventFd中写入1.
 */
void Looper::wake() {
    uint64_t inc = 1;
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
    if (nWrite != sizeof(uint64_t)) {
        if (errno != EAGAIN) {
            ALOGW("Could not write wake signal, errno=%d", errno);
        }
    }
}


/**
 * @param timeoutMillis 超时时间(ms)
 *     timeoutMillis = 0，立即返回；
 *     timeoutMillis < 0，无限等待直到有event到来；
 * @return 返回值有以下几种：
 *     POLL_WAKE(-1): 在timeoutMillis消耗完之前wake()被调用；
 *     POLL_CALLBACK(-2): 由一个或多个callback函数被调用；
 *     POLL_TIMEOUT(-3):在timeoutMillis消耗完之前没有数据到来；
 *     POLL_ERROR(-4): 出现错误。
 *     >= 0: 有事件发生、且该事件的fd没有callback函数时，返回该fd携带的ident，即addFd()函数传入的indent; 
 *           当且仅当此时，outFd、outEvents和outData参数包含与该fd相关的信息； 
 *           除此之外，outFd和outEvents被赋值0, *out被赋值NULL(如果这些实参非NULL的话)。
 */
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        //首先遍历[mResponseIndex, mResponse.size()), 找出ident>=0的项，直接返回该indent。 同时，
        //mResponseIndex指向下一个要处理的项。
        while (mResponseIndex < mResponses.size()) {
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            //ident>=0，表示调用者提供的callback函数为NULL， 返回信息给调用者，由其自己处理。
            if (ident >= 0) {
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;

                if (outFd != NULL) *outFd = fd;
                if (outEvents != NULL) *outEvents = events;
                if (outData != NULL) *outData = data;
                return ident;
            }
        }

        if (result != 0) {
            if (outFd != NULL) *outFd = 0;
            if (outEvents != NULL) *outEvents = 0;
            if (outData != NULL) *outData = NULL;
            return result;
        }

        //pollInner()函数会重置并填充mResponses，重置mResponseIndex为0。
        result = pollInner(timeoutMillis);
    } //for(;;)
}

/**
 * @brief: 
 *     1. 重新计算超时时间；
 *     2. 清空mResponse(Vector<Response>类型)，重置mResponseIndex为0，result为POLL_WAKE，mPolling为true；
 *     3. 等待epoll_ctr()函数返回， 返回后置mPolling为false, 然后加锁处理以下内容：
 *         返回值 < 0： 置result为POLL_ERROR；
 *         返回值 = 0： 置result为POLL_TIMEOUT;
 *         返回值 > 0： 表示发生事件的数量, 依次将所有事件fd、及对应类型插入到mResponses(Vector<Response>类型)中。
 *         將锁释放；
 *     4. 依次处理mResponses中的所有事件，如果有POLL_CALLBACK的事件，则置result为POLL_CALLBACK；
 *     5. 返回result。
 *
int Looper::pollInner(int timeoutMillis) {
    /**
     * 重新计算超时时间：
      1. 如果mNextMessageUptime已被赋值, 则计算mNextMessageUptime与当前时间的差值(换算成ms), 记为messageTimeoutMillis。
            若messageTimeoutMillis < 0, 则超时超时时间为timeOutMills；
            若messageTimeoutMillis >= 0, 且messageTimeoutMillis < timeoutMillis, 则超时时间调整为messageTimeoutMillis。
      2. 如果mNextMessageUptime为LLONG_MAX, 则超时时间为timeoutMills。
     */
    if (timeoutMillis != 0 && mNextMessageUptime != LLONG_MAX) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        int messageTimeoutMillis = toMillisecondTimeoutDelay(now, mNextMessageUptime);
        if (messageTimeoutMillis >= 0
                && (timeoutMillis < 0 || messageTimeoutMillis < timeoutMillis)) {
            timeoutMillis = messageTimeoutMillis;
        }
    }

    int result = POLL_WAKE;
    mResponses.clear();
    mResponseIndex = 0;

    mPolling = true;

    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    mPolling = false;

    // Acquire lock.
    mLock.lock();

    if (mEpollRebuildRequired) {
        mEpollRebuildRequired = false;
        rebuildEpollLocked();
        goto Done;
    }

    //epoll出错。
    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        ALOGW("Poll failed with an unexpected error, errno=%d", errno);
        result = POLL_ERROR;
        goto Done;
    }

    //epoll超时
    if (eventCount == 0) {
        result = POLL_TIMEOUT;
        goto Done;
    }

    //将epoll_ctrl()函数返回值中读取发生的事件信息，依次将所有的事件fd、及对应事件类型插入到mResponses向量表中。
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;

        if (fd == mWakeEventFd) {
            if (epollEvents & EPOLLIN) {
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake event fd.", epollEvents);
            }
        } else {
            //如果fd已经从mRequests中移除, 则不再处理该fd的事件。
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                //构建新的Response实例, 添加到mResponses中，Response实例由events和Request两部分组成。
                int events = 0;
                if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
                pushResponse(events, mRequests.valueAt(requestIndex));
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on fd %d that is "
                        "no longer registered.", epollEvents, fd);
            }
        }//if
    }//for

Done: ;       
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) {
            {
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock();
                handler->handleMessage(message);
            } // release handler

            mLock.lock();
            mSendingMessage = false;
            result = POLL_CALLBACK;
        } else {
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }//while

    // Release lock.
    mLock.unlock();

    //处理mResponse中的所有事件。
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        //如果ident不是POLL_BACK时，不做任何处理。
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;

            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            //如果handleEvent()函数返回0, 则从mRequests中移除fd。
            if (callbackResult == 0) {
                removeFd(fd, response.request.seq);
            }

            response.request.callback.clear();
            result = POLL_CALLBACK;
        }
    }

    return result;
}
```

