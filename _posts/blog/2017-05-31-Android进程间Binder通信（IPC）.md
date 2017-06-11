---
layout: post
title: Android进程间通信(Binder)
description: Android进程间通信(Binder)
category: blog
---


### 概述
Android系统是基于Linux内核的，而Linux内核继承和兼容了丰富的Unix系统进程间通信（IPC）机制。有传统的管道（Pipe）、信号（Signal）和跟踪（Trace），这三项通信手段只能用于父进程与子进程之间，或者兄弟进程之间；后来又增加了命令管道（Named Pipe），使得进程间通信不再局限于父子进程或者兄弟进程之间；为了更好地支持商业应用中的事务处理，在AT&T的Unix系统V中，又增加了三种称为“System V IPC”的进程间通信机制，分别是报文队列（Message）、共享内存（Share Memory）和信号量（Semaphore）；后来BSD Unix对“System V IPC”机制进行了重要的扩充，提供了一种称为插口（Socket）的进程间通信机制。

Android系统没有采用上述提到的各种进程间通信机制，而是采用Binder机制.

在Android系统的Binder机制中，由一系统组件组成，分别是**Client**、**Server**、**Service Manager**和**Binder驱动程序**，其中Client、Server和Service Manager运行在用户空间，Binder驱动程序运行内核空间。Binder就是一种把这四个组件粘合在一起的粘结剂了，其中，核心组件便是Binder驱动程序了，Service Manager提供了辅助管理的功能，Client和Server正是在Binder驱动和Service Manager提供的基础设施上，进行Client-Server之间的通信。Service Manager和Binder驱动已经在Android平台中实现好，开发者只要按照规范实现自己的Client和Server组件就可以了。

![关系图](/images/client_sever_manager.gif)

1. Client、Server和Service Manager实现在用户空间中，Binder驱动程序实现在内核空间中
2. Binder驱动程序和Service Manager在Android平台中已经实现，开发者只需要在用户空间实现自己的Client和Server
3. Binder驱动程序提供设备文件/dev/binder与用户空间交互，Client、Server和Service Manager通过open和ioctl文件操作函数与Binder驱动程序进行通信
4. Client和Server之间的进程间通信通过Binder驱动程序间接实现
5. Service Manager是一个守护进程，用来管理Server，并向Client提供添加和查询Server接口的能力

#### Service Manager介绍
既然Service Manager组件是用来管理Server并且向Client提供查询Server远程接口的功能，那么，Service Manager就必然要和Server以及Client进行通信了。我们知道，Service Manger、Client和Server三者分别是运行在独立的进程当中，这样它们之间的通信也属于进程间通信了，而且也是采用Binder机制进行进程间通信，因此，Service Manager在充当Binder机制的守护进程的角色的同时，也在充当Server的角色，然而，它是一种特殊的Server，下面我们将会看到它的特殊之处。

Service Manager的入口位于service_manager.c文件中的main函数：

```c
int main(int argc, char **argv)
{
    struct binder_state *bs;
    
    //它表示Service Manager的句柄为0。
    //Binder通信机制使用句柄来代表远程接口，这个句柄的意义和Windows编程中用到的句柄是差不多的概念。
    //前面说到，Service Manager在充当守护进程的同时，它充当Server的角色，当它作为远程接口使用时，
    //它的句柄值便为0，这就是它的特殊之处，其余的Server的远程接口句柄值都是一个大于0 而且由Binder驱动程序自动进行分配的。
    void *svcmgr = BINDER_SERVICE_MANAGER;

    bs = binder_open(128*1024);

    if (binder_become_context_manager(bs)) {
        LOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }

    svcmgr_handle = svcmgr;
    binder_loop(bs, svcmgr_handler);
    return 0;
}
```

main函数主要有三个功能：一是打开Binder设备文件；二是告诉Binder驱动程序自己是Binder上下文管理者，即我们前面所说的守护进程；三是进入一个无穷循环，充当Server的角色，等待Client的请求.

函数首先是执行打开Binder设备文件的操作binder_open，这个函数位于frameworks/base/cmds/servicemanager/binder.c文件中：

```c
struct binder_state *binder_open(unsigned mapsize)
{
    struct binder_state *bs;

    bs = malloc(sizeof(*bs));
    if (!bs) {
        errno = ENOMEM;
        return 0;
    }

	 // 打开/dev/binder设备文件
    bs->fd = open("/dev/binder", O_RDWR);
    if (bs->fd < 0) {
        fprintf(stderr,"binder: cannot open device (%s)\n",
                strerror(errno));
        goto fail_open;
    }

    bs->mapsize = mapsize;
    
    // 内存映射
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    if (bs->mapped == MAP_FAILED) {
        fprintf(stderr,"binder: cannot map device (%s)\n",
                strerror(errno));
        goto fail_map;
    }

        /* TODO: check version */

    return bs;

fail_map:
    close(bs->fd);
fail_open:
    free(bs);
    return 0;
}
```

这里为什么会同时使用进程虚拟地址空间和内核虚拟地址空间来映射同一个物理页面呢？

    这就是Binder进程间通信机制的精髓所在了，同一个物理页面，一方映射到进程虚拟地址空间，一方面映射到内核虚拟地址空间，这样，进程和内核之间就可以减少一次内存拷贝了，提到了进程间通信效率。
    举个例子如，Client要将一块内存数据传递给Server，一般的做法是，Client将这块数据从它的进程空间拷贝到内核空间中，然后内核再将这个数据从内核空间拷贝到Server的进程空间，这样，Server就可以访问这个数据了。
    但是在这种方法中，执行了两次内存拷贝操作，而采用我们上面提到的方法，只需要把Client进程空间的数据拷贝一次到内核空间，然后Server与内核共享这个数据就可以了，整个过程只需要执行一次内存拷贝，提高了效率。

#### Service Manager是成为Android进程间通信（IPC）机制Binder守护进程的过程：     
1. 打开/dev/binder文件：open("/dev/binder", O_RDWR);
2. 建立128K内存映射：mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
3. 通知Binder驱动程序它是守护进程：binder_become_context_manager(bs);
4. 进入循环等待请求的到来：binder_loop(bs, svcmgr_handler);

### Binder中Server和Client获得Service Manager接口过程
Service Manager在Binder机制中既充当守护进程的角色，同时它也充当着Server角色，然而它又与一般的Server不一样。对于普通的Server来说，Client如果想要获得Server的远程接口，那么必须通过Service Manager远程接口提供的getService接口来获得，这本身就是一个使用Binder机制来进行进程间通信的过程。而对于Service Manager这个Server来说，Client如果想要获得Service Manager远程接口，却不必通过进程间通信机制来获得，因为Service Manager远程接口是一个特殊的Binder引用，它的引用句柄一定是0。

#### 获取Service Manager远程接口的过程
获取Service Manager远程接口的函数是defaultServiceManager：

```c
sp<IServiceManager> defaultServiceManager()
{

    if (gDefaultServiceManager != NULL) return gDefaultServiceManager;

    {
        AutoMutex _l(gDefaultServiceManagerLock);
        if (gDefaultServiceManager == NULL) {
            gDefaultServiceManager = interface_cast<IServiceManager>(
                ProcessState::self()->getContextObject(NULL));
        }
    }

    return gDefaultServiceManager;
}
```

![ServiceManager继承类图](/images/service_manager.gif)

BpInterface是一个模板类：

```c
template<typename INTERFACE>
class BpInterface : public INTERFACE, public BpRefBase
{
public:
	BpInterface(const sp<IBinder>& remote);

protected:
	virtual IBinder* onAsBinder();
};
```

IServiceManager类继承了IInterface类，而IInterface类和BpRefBase类又分别继承了RefBase类。

在BpRefBase类中，有一个成员变量mRemote，它的类型是IBinder*，实现类为BpBinder，它表示一个Binder引用，引用句柄值保存在BpBinder类的mHandle成员变量中。

BpBinder类通过IPCThreadState类来和Binder驱动程序交互，而IPCThreadState又通过它的成员变量mProcess来打开/dev/binder设备文件，mProcess成员变量的类型为ProcessState。ProcessState类打开设备/dev/binder之后，将打开文件描述符保存在mDriverFD成员变量中，以供后续使用。

最终目的是要创建一个BpServiceManager实例，并且返回它的IServiceManager接口。创建Service Manager远程接口主要是下面语句：

```c++

gDefaultServiceManager = interface_cast<IServiceManager>(
           ProcessState::self()->getContextObject(NULL));
```

返回一个全局唯一的ProcessState实例变量，就是单例模式了，这个变量名为gProcess。如果gProcess尚未创建，就会执行创建操作，在ProcessState的构造函数中，会通过open文件操作函数打开设备文件/dev/binder，并且返回来的设备文件描述符保存在成员变量mDriverFD中。

```c++
gDefaultServiceManager = new BpServiceManager(new BpBinder(0));
```

这样，Service Manager远程接口就创建完成了，它本质上是一个BpServiceManager，包含了一个句柄值为0的Binder引用。

#### 在Android系统的Binder机制中，Server和Client拿到这个Service Manager远程接口之后怎么用？
对Server来说，就是调用IServiceManager::addService这个接口来和Binder驱动程序交互了，即调用BpServiceManager::addService 。而BpServiceManager::addService又会调用通过其基类BpRefBase的成员函数remote获得原先创建的BpBinder实例，接着调用BpBinder::transact成员函数。在BpBinder::transact函数中，又会调用IPCThreadState::transact成员函数，这里就是最终与Binder驱动程序交互的地方了。回忆一下前面的类图，IPCThreadState有一个PorcessState类型的成中变量mProcess，而mProcess有一个成员变量mDriverFD，它是设备文件/dev/binder的打开文件描述符，因此，IPCThreadState就相当于间接在拥有了设备文件/dev/binder的打开文件描述符，于是，便可以与Binder驱动程序交互了。

对Client来说，就是调用IServiceManager::getService这个接口来和Binder驱动程序交互了。具体过程上述Server使用Service Manager的方法是一样的，这里就不再累述了。

###  Android系统进程间通信（IPC）机制Binder中的Server启动过程分析
在Android系统中Binder进程间通信机制中的Server角色获得Service Manager远程接口，即defaultServiceManager函数的实现，Server获得了Service Manager远程接口之后，就要把自己的Service添加到Service Manager中去，然后把自己启动起来，等待Client的请求。

![MediaPlayerServices](/images/MediaPlayerService.gif)

BnMediaPlayerService是一个Binder Native类，**其实就是Binder的Stub实现类**，用来处理Client请求的。BnMediaPlayerService继承于BnInterface<IMediaPlayerService>类，BnInterface是一个模板类，它定义在frameworks/base/include/binder/IInterface.h文件中：

```c
template<typename INTERFACE>  
class BnInterface : public INTERFACE, public BBinder  
{  
	public:  
	    virtual sp<IInterface>      queryLocalInterface(const String16& _descriptor);  
	    virtual const String16&     getInterfaceDescriptor() const;  
	  
	protected:  
	    virtual IBinder*            onAsBinder();  
};
```

BnMediaPlayerService实际是继承了IMediaPlayerService和BBinder类。IMediaPlayerService和BBinder类又分别继承了IInterface和IBinder类，IInterface和IBinder类又同时继承了RefBase类。

实际上，BnMediaPlayerService并不是直接接收到Client处发送过来的请求，而是使用了IPCThreadState接收Client处发送过来的请求，而IPCThreadState又借助了ProcessState类来与Binder驱动程序交互。

IPCThreadState接收到了Client处的请求后，就会调用BBinder类的transact函数，并传入相关参数，BBinder类的transact函数最终调用BnMediaPlayerService类的onTransact函数，于是，就开始真正地处理Client的请求了。

#### MediaPlayerService是如何启动的
启动MediaPlayerService的代码位于frameworks/base/media/mediaserver/main_mediaserver.cpp文件中：

```c++
int main(int argc, char** argv)  
{  
	 // 创建 ProcessState
    sp<ProcessState> proc(ProcessState::self()); 
    
    // 创建BpServiceManager 
    sp<IServiceManager> sm = defaultServiceManager(); 
     
    LOGI("ServiceManager: %p", sm.get());  
    AudioFlinger::instantiate(); 
     
    // 初始化MediaPlayerService服务，把MediaPlayerService添加到Service Manger中去
    MediaPlayerService::instantiate();  
    
    CameraService::instantiate();  
    AudioPolicyService::instantiate();  
    
    // 
    ProcessState::self()->startThreadPool();  
    IPCThreadState::self()->joinThreadPool();  
}  
```

ProcessState::self()这个函数的作用：

1. 通过open_driver函数打开Binder设备文件/dev/binder，并将打开设备文件描述符保存在成员变量mDriverFD中；
2. 通过mmap来把设备文件/dev/binder映射到内存中

打开/dev/binder设备文件后，Binder驱动程序就为MediaPlayerService进程创建了一个struct binder_proc结构体实例来维护MediaPlayerService进程上下文相关信息。

```c++
void MediaPlayerService::instantiate() {
    defaultServiceManager()->addService(
            String16("media.player"), new MediaPlayerService());
}
```

defaultServiceManager返回的实际是一个BpServiceManger类实例，因此，我们看一下BpServiceManger::addService的实现

```c++
virtual status_t addService(const String16& name, const sp<IBinder>& service)
    {
        Parcel data, reply;
        data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
        data.writeString16(name);
        data.writeStrongBinder(service);
        status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
        return err == NO_ERROR ? reply.readExceptionCode() : err;
    }
```

这里的remote成员函数来自于BpRefBase类，它返回一个BpBinder指针

```c++
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}
```

主要调用了talkWithDriver函数来与Binder驱动程序进行交互.

###  Android系统进程间通信（IPC）机制Binder中的Client获得Server远程接口过程源代码分析
获取MediaPlayerService这个远程接口的本质问题就变为从Service Manager中获得MediaPlayerService的一个句柄

```c++
// establish binder interface to MediaPlayerService
/*static*/const sp<IMediaPlayerService>&
IMediaDeathNotifier::getMediaPlayerService()
{
    LOGV("getMediaPlayerService");
    Mutex::Autolock _l(sServiceLock);
    if (sMediaPlayerService.get() == 0) {
        sp<IServiceManager> sm = defaultServiceManager();
        sp<IBinder> binder;
        do {
            binder = sm->getService(String16("media.player"));
            if (binder != 0) {
                break;
             }
             LOGW("Media player service not published, waiting...");
             usleep(500000); // 0.5 s
        } while(true);

        if (sDeathNotifier == NULL) {
        sDeathNotifier = new DeathNotifier();
    }
    binder->linkToDeath(sDeathNotifier);
    sMediaPlayerService = interface_cast<IMediaPlayerService>(binder);
    }
    LOGE_IF(sMediaPlayerService == 0, "no media player service!?");
    return sMediaPlayerService;
}
```

接下去的while循环是通过sm->getService接口来不断尝试获得名称为“media.player”的Service，即MediaPlayerService。为什么要通过这无穷循环来得MediaPlayerService呢？因为这时候MediaPlayerService可能还没有启动起来，所以这里如果发现取回来的binder接口为NULL，就睡眠0.5秒，然后再尝试获取，这是获取Service接口的标准做法。

最终过程是，通过IPCThreadState::talkWithDriver与驱动程序进行交互，最终会返回一个BpBinder对象，最后，函数调用：

```c++
sMediaPlayerService = interface_cast<IMediaPlayerService>(binder);  
```

从而创建一个BpMediaPlayerService对象：

```c++
intr = new BpMediaPlayerService(new BpBinder(handle)); 
```

有了这个BpMediaPlayerService这个远程接口之后，MediaPlayer就可以调用MediaPlayerService的服务了。

### Android系统进程间通信Binder机制在应用程序框架层的Java接口源代码分析
应用程序框架中的基于Java语言的Binder接口是通过JNI来调用基于C/C++语言的Binder运行库来为Java应用程序提供进程间通信服务的。

主要就是Service Manager、Server和Client这三个角色的实现了。通常，在应用程序中，我们都是把Server实现为Service的形式，并且通过IServiceManager.addService接口来把这个Service添加到Service Manager，Client也是通过IServiceManager.getService接口来获得Service接口，接着就可以使用这个Service提供的功能了，这个与运行时库的Binder接口是一致的。

#### 一.  获取Service Manager的Java远程接口

![ServiceManagerProxy](/images/binder_service.gif)

ServiceManagerProxy类实现了IServiceManager接口，IServiceManager提供了getService和addService两个成员函数来管理系统中的Service。从ServiceManagerProxy类的构造函数可以看出，它需要一个BinderProxy对象的IBinder接口来作为参数。因此，要获取Service Manager的Java远程接口ServiceManagerProxy，首先要有一个BinderProxy对象。

ServiceManager类有一个静态成员函数getIServiceManager，它的作用就是用来获取Service Manager的Java远程接口了，而这个函数又是通过ServiceManagerNative来获取Service Manager的Java远程接口的。

```c++
public final class ServiceManager {
	......
	private static IServiceManager sServiceManager;
	......
	private static IServiceManager getIServiceManager() {
		if (sServiceManager != null) {
			return sServiceManager;
		}

		// Find the service manager
		sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
		return sServiceManager;
	}
	......
}
```

BinderInternal.getContextObject是一个JNI方法

```c++

public class BinderInternal {
	......
	/**
	* Return the global "context object" of the system.  This is usually
	* an implementation of IServiceManager, which you can use to find
	* other services.
	*/
	public static final native IBinder getContextObject();
	
	......
}
```

它实现在frameworks/base/core/jni/android_util_Binder.cpp文件中：

```c
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    return javaObjectForIBinder(env, b);
}
```

ProcessState::self()->getContextObject函数返回一个BpBinder对象, 接着调用javaObjectForIBinder把这个BpBinder对象转换成一个BinderProxy对象。

回到ServiceManager.getIServiceManager中，从下面语句返回：

```c++
sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());  
```

相当于是：

```c++
sServiceManager = ServiceManagerNative.asInterface(new BinderProxy());  
```

接下去就是调用ServiceManagerNative.asInterface函数了，

```c++
public abstract class ServiceManagerNative ......
{
	......
	static public IServiceManager asInterface(IBinder obj)
	{
		if (obj == null) {
			return null;
		}
		IServiceManager in =
			(IServiceManager)obj.queryLocalInterface(descriptor);
		if (in != null) {
			return in;
		}

		return new ServiceManagerProxy(obj);
	}
	......
}
```

返回到ServiceManager.getIServiceManager中，从下面语句返回：

```c++
sServiceManager = ServiceManagerNative.asInterface(new BinderProxy());  
```

就相当于是：

```c++
sServiceManager = new ServiceManagerProxy(new BinderProxy());  
```

于是，我们的目标终于完成了。

总结一下，就是在Java层，我们拥有了一个Service Manager远程接口ServiceManagerProxy，而这个ServiceManagerProxy对象在JNI层有一个句柄值为0的BpBinder对象与之通过gBinderProxyOffsets关联起来。

这样获取Service Manager的Java远程接口的过程就完成了。

#### 二. MyService接口的定义
在Java层，接口是定义在AIDL（Android Interface Define Language）里定义的。

```java
/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: /Users/zerro/AndroidStudioProjects/JNIProject/app/src/main/aidl/com/example/zerro/jniproject/IPlayerInterface.aidl
 */
package com.example.zerro.jniproject;
// Declare any non-default types here with import statements

public interface IPlayerInterface extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.example.zerro.jniproject.IPlayerInterface {
        private static final java.lang.String DESCRIPTOR = "com.example.zerro.jniproject.IPlayerInterface";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.example.zerro.jniproject.IPlayerInterface interface,
         * generating a proxy if needed.
         */
        public static com.example.zerro.jniproject.IPlayerInterface asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.example.zerro.jniproject.IPlayerInterface))) {
                return ((com.example.zerro.jniproject.IPlayerInterface) iin);
            }
            return new com.example.zerro.jniproject.IPlayerInterface.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_play: {
                    data.enforceInterface(DESCRIPTOR);
                    this.play();
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_stop: {
                    data.enforceInterface(DESCRIPTOR);
                    this.stop();
                    reply.writeNoException();
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.example.zerro.jniproject.IPlayerInterface {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            /**
             * Demonstrates some basic types that you can use as parameters
             * and return values in AIDL.
             */
            @Override
            public void play() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_play, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }

            @Override
            public void stop() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_stop, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }

        static final int TRANSACTION_play = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_stop = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    }

    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    public void play() throws android.os.RemoteException;

    public void stop() throws android.os.RemoteException;
}
```

其实就是Proxy-Stub的设计模式






















