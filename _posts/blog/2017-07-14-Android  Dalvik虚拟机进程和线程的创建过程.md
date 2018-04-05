---
layout: post
title: Android  Dalvik虚拟机进程和线程的创建过程
description: Android  Dalvik虚拟机进程和线程的创建过程
category: blog
---


### 概述
Dalvik虚拟机除了可以执行Java代码之外，还可以执行Native代码，也就是C/C++函数。这些C/C++函数在执行的过程中，又可以通过本地操作系统提供的系统调用来创建本地操作系统进程或者线程，也就是Linux进程和线程。如果在Native代码中创建出来的进程又加载有Dalvik虚拟机，那么它实际上又可以看作是一个Dalvik虚拟机进程。另一方面，如果在Native代码中创建出来的线程能够执行Java代码，那么它实际上又可以看作是一个Dalvik虚拟机线程。

1. Dalvik虚拟机进程的创建过程；

	Dalvik虚拟机进程可以通过android.os.Process类的静态成员函数start来创建；

2. Dalvik虚拟机线程的创建过程；

	Dalvik虚拟机线程可以通过java.lang.Thread类的成员函数start来创建；

3. 只执行C/C++代码的Native线程的创建过程；

	只执行C/C++代码的Native线程可以通过C++类Thread的成员函数run来创建；

4. 能同时执行C/C++代码和Java代码的Native线程的创建过程。

	能同时执行C/C++代码和Java代码的Native线程也可以通过C++类Thread的成员函数run来创建；

### Dalvik虚拟机进程的创建过程
Dalvik虚拟机进程实际上就是通常我们所说的Android应用程序进程。Android应用程序进程是由ActivityManagerService服务通过android.os.Process类的静态成员函数start来请求Zygote进程创建的，而Zyogte进程最终又是通过dalvik.system.Zygote类的静态成员函数forkAndSpecialize来创建该Android应用程序进程的。

Zygote类的静态成员函数forkAndSpecialize是一个JNI方法，它是由C++层的函数Dalvik_dalvik_system_Zygote_forkAndSpecialize来实现的,它通过调用另外一个函数forkAndSpecializeCommon来创建一个Dalvik虚拟机进程。

```c++
/*
 * Utility routine to fork zygote and specialize the child process.
 */
static pid_t forkAndSpecializeCommon(const u4* args, bool isSystemServer)
{
    pid_t pid;

    uid_t uid = (uid_t) args[0];
    gid_t gid = (gid_t) args[1];
    ArrayObject* gids = (ArrayObject *)args[2];
    u4 debugFlags = args[3];
    ArrayObject *rlimits = (ArrayObject *)args[4];
    int64_t permittedCapabilities, effectiveCapabilities;

    if (isSystemServer) {
        /*
         * Don't use GET_ARG_LONG here for now.  gcc is generating code
         * that uses register d8 as a temporary, and that's coming out
         * scrambled in the child process.  b/3138621
         */
        //permittedCapabilities = GET_ARG_LONG(args, 5);
        //effectiveCapabilities = GET_ARG_LONG(args, 7);
        permittedCapabilities = args[5] | (int64_t) args[6] << 32;
        effectiveCapabilities = args[7] | (int64_t) args[8] << 32;
    } else {
        permittedCapabilities = effectiveCapabilities = 0;
    }

    if (!gDvm.zygote) {
        ......
        return -1;
    }

    ......

    pid = fork();

    if (pid == 0) {
        int err;
        ......

        err = setgroupsIntarray(gids);
        ......

        err = setrlimitsFromArray(rlimits); 
        ......

        err = setgid(gid);
        ......

        err = setuid(uid);
        ......

        err = setCapabilities(permittedCapabilities, effectiveCapabilities);
        ......

        enableDebugFeatures(debugFlags);
        ......

        gDvm.zygote = false;
        if (!dvmInitAfterZygote()) {
            ......
            dvmAbort();
        }
    } else if (pid > 0) {
        /* the parent process */
    }

    return pid;
}
```

函数forkAndSpecializeCommon除了可以用来创建普通的Android应用程序进程之外，还用来创建System进程。Android系统中的System进程和普通的Android应用程序进程一样，也是由Zygote进程负责创建的。

它主要就是调用系统调用fork来创建一个进程。我们知道系统调用fork执行完成之后，会有两次返回，其中一次是返回当前进程中，另外一次是返回到新创建的进程中。当系统调用fork返回到新创建的进程的时候，它的返回值pid就会等于0，这时候就可以调用相应的函数来设置新创建的进程的uid、gid、gids、debugFlags和rlimits，从而将限制了新创建的进程的权限。

两个地方是需要注意的：

1. 只有Zygote进程才有权限创建System进程和Android应用程序进程。
2. 当一个进程使用系统调用fork来创建一个新进程的时候，前者就称为父进程，后者就称为子进程。这时候父进程和子进程共享的地址空间是一样的，但是只要某个地址被父进程或者子进程进行写入操作的时候，这块被写入的地址空间才会在父进程和子进程之间独立开来，这种机制就称为COW（copy on write）。

因此，由Zygote进程创建出来的System进程和Android应用程序进程实际上是共享了很多东西，而且只要这些东西都是只读的时候，它们就会一直被共享着。

Zygote进程在启动的过程中，加载了很多东西，例如，Java和Android核心类库（dex文件）及其JNI方法（so文件）。这些dex文件和so文件的只读段，例如代码段，都会一直在Zygote进程、System进程和Android应用程序进程中进行共享。这样，我们在Zygote进程中进行的大量预加载行为就获得了价值，一方面是可以加快System进程和Android应用程序进程的启动过程中，另外一方面也使得系统的整体内存消耗减少。

forkAndSpecializeCommon还会调用函数dvmInitAfterZygote来进一步对在新创建的进程中运行的Dalvik虚拟机进行初始化，初始化操作包括：

1. 调用函数dvmGcStartupAfterZygote来进行一次GC。
2. 调用函数dvmSignalCatcherStartup来启动一个Linux信号收集线程，主要是用来捕捉SIGQUIT信号，以便可以在进程退出前将各个线程的堆栈DUMP出来。
3. 调用函数dvmStdioConverterStartup来启动一个标准输出重定向线程，该线程负责将当前进程的标准输出（stdout和stderr）重定向到日志输出系统中去，前提是设置了Dalvik虚拟机的启动选项-Xlog-stdio。
4. 调用函数dvmInitJDWP来启动一个JDWP线程，以便我们可以用DDMS工具来调试进程中的Dalvik虚拟机。
5. 调用函数dvmCompilerStartup来启动JIT，前提是当前使用的Dalvik虚拟机在编译时支持JIT，并且该Dalvik虚拟机在启动时指定了-Xint:jit选项。

**从中我们就可以得出结论：一个Dalvik虚拟机进程实际上就是一个Linux进程。**

### Dalvik虚拟机线程的创建过程
在Java代码中，我们可以通过java.lang.Thread类的成员函数start来创建一个Dalvik虚拟机线程：

```Java
/**
     * Starts the new Thread of execution. The <code>run()</code> method of
     * the receiver will be called by the receiver Thread itself (and not the
     * Thread calling <code>start()</code>).
     *
     * @throws IllegalThreadStateException if the Thread has been started before
     *
     * @see Thread#run
     */
    public synchronized void start() {
        if (hasBeenStarted) {
            throw new IllegalThreadStateException("Thread already started."); // TODO Externalize?
        }

        hasBeenStarted = true;

        VMThread.create(this, stackSize);
    }
```

Thread类的成员函数start接下来就继续调用VMThread类的静态成员函数create来创建一个线程。

VMThread类的静态成员函数create是一个JNI方法，它是C++层的函数Dalvik_java_lang_VMThread_create来实现的.

```c++
bool dvmCreateInterpThread(Object* threadObj, int reqStackSize)
{
    pthread_attr_t threadAttr;
    pthread_t threadHandle;
    ......
    Thread* newThread = NULL;
    ......
    int stackSize;
    ......

    if (reqStackSize == 0)
        stackSize = gDvm.stackSize;
    else if (reqStackSize < kMinStackSize)
        stackSize = kMinStackSize;
    else if (reqStackSize > kMaxStackSize)
        stackSize = kMaxStackSize;
    else
        stackSize = reqStackSize;

    pthread_attr_init(&threadAttr);
    pthread_attr_setdetachstate(&threadAttr, PTHREAD_CREATE_DETACHED);
    ......

    newThread = allocThread(stackSize);
    ......

    newThread->threadObj = threadObj;
    ......

    int cc = pthread_create(&threadHandle, &threadAttr, interpThreadStart,
            newThread);
    ......

    while (newThread->status != THREAD_STARTING)
        pthread_cond_wait(&gDvm.threadStartCond, &gDvm.threadListLock);
    ......

    newThread->next = gDvm.threadList->next;
    if (newThread->next != NULL)
        newThread->next->prev = newThread;
    newThread->prev = gDvm.threadList;
    gDvm.threadList->next = newThread;
    ......

    newThread->status = THREAD_VMWAIT;
    pthread_cond_broadcast(&gDvm.threadStartCond);
    ......

    return true;
}
```

函数dvmCreateInterpThread是通过函数pthread_create来创建一个线程。

函数pthread_create实际上是由pthread库提供的一个函数，它最终是通过系统调用clone来请求内核创建一个线程的。由此就可以看出，Dalvik虚拟机线程实际上就是本地操作系统线程。

接下来：

1. 调用函数prepareThread来初始化新创建的Dalvik虚拟机线程。
2. 调用函数dvmCreateJNIEnv来为新创建的Dalvik虚拟机线程创建一个JNI环境。也就是创建一个JNIEnvExt对象，并且为该JNIEnvExt对象创建一个Java代码访问函数表。有了这个Java代码访问函数表之后，我们才可以在JNI方法中访问Java对象，以及执行Java代码。
3. 调用另外一个函数dvmCallMethod来通知Dalvik虚拟机解释器执行java.lang.Thread类的成员函数run。

### 只执行C/C++代码的Native线程的创建过程
我们这里所说的Native线程就是指本地操作系统线程，它们是Dalvik虚拟机在执行C/C++代码的过程中创建的，很显然它们就是Linux线程。例如，在C/C++代码中，我们可以通过C++类Thread的成员函数run来创建一个Native线程。

**只执行C/C++代码的Native线程的创建和运行过程了，从中我们就可以得出结论：Native线程与Dalvik虚拟机线程一样，也是一个Linux线程。**

### 能同时执行C/C++代码和Java代码的Native线程的创建过程
与只能执行C/C++代码的Native线程一样，能同时执行C/C++代码和Java代码的Native线程也是可以通过C++类Thread的成员函数run来创建的。

### 总结
这样，Dalvik虚拟机进程和线程的创建过程分析就分析完成了，从中我们就可以得到它们与本地操作系统的进程和线程的关系：

1. Dalvik虚拟机进程就是本地操作系统进程，也就是Linux进程，区别在于前者运行有一个Dalvik虚拟机实例。
2. Dalvik虚拟机线程就是本地操作系统进程，也就是Linux线程，区别在于前者在创建的时候会自动附加到Dalvik虚拟机中去，而后者在需要执行Java代码的时候才会附加到Dalvik虚拟机中去。

我们可以思考一下：为什么Dalvik虚拟机要将自己的进程和线程使用本地操作系统的进程和线程来实现呢？我们知道，进程调度是一个很复杂的问题，特别是在多核的情况下，它要求高效地利用CPU资源，并且公平地调试各个进程，以达到系统吞吐量最大的目标。Linux内核本身已经实现了这种高效和公平的进程调度机制，因此，就完全没有必要在Dalvik虚拟机内部再实现一套，这样就要求Dalvik虚拟机使用本地操作系统的进程来作为自己的进程。此外，Linux内核没有线程的概念，不过它可以使用一种称为轻量级进程的概念来实现线程，这样Dalvik虚拟机使用本地操作系统的线程来作为自己的线程，就同样可以利用Linux内核的进程调度机制来调度它的线程，从而也能实现高效和公平的原则。

### ART运行时无缝替换Dalvik虚拟机的过程分析
我们知道，Dalvik虚拟机实则也算是一个Java虚拟机，只不过它执行的不是class文件，而是dex文件。

ART运行时就是真的和Dalvik虚拟机一样，实现了一套完全兼容Java虚拟机的接口。

Dalvik虚拟机和ART虚拟机都实现了三个用来抽象Java虚拟机的接口：

1. JNI_GetDefaultJavaVMInitArgs -- 获取虚拟机的默认初始化参数
2. JNI_CreateJavaVM -- 在进程中创建虚拟机实例
3. JNI_GetCreatedJavaVMs -- 获取进程中创建的虚拟机实例

在Android系统中，Davik虚拟机实现在libdvm.so中，ART虚拟机实现在libart.so中。也就是说，libdvm.so和libart.so导出了JNI_GetDefaultJavaVMInitArgs、JNI_CreateJavaVM和JNI_GetCreatedJavaVMs这三个接口，供外界调用。

Dalvik虚拟机执行的是dex字节码，ART虚拟机执行的是本地机器码。这意味着Dalvik虚拟机包含有一个解释器，用来执行dex字节码。当然，Android从2.2开始，也包含有JIT（Just-In-Time），用来在运行时动态地将执行频率很高的dex字节码翻译成本地机器码，然后再执行。通过JIT，就可以有效地提高Dalvik虚拟机的执行效率。即使用采用了JIT，Dalvik虚拟机的总体性能还是不能与直接执行本地机器码的ART虚拟机相比。

ART虚拟机执行的本地机器码是从哪里来的呢？

开发者开发出的应用程序经过编译和打包之后，仍然是一个包含dex字节码的APK文件。AOT进Ahead-Of-Time的简称，它发生在程序运行之前。我们用静态语言（例如C/C++）来开发应用程序的时候，编译器直接就把它们翻译成目标机器码。ART虚拟机并不要求开发者将自己的应用直接编译成目标机器码。这样，将应用的dex字节码翻译成本地机器码的最恰当AOT时机就发生在应用安装的时候。

我们知道，没有ART虚拟机之前，应用在安装的过程，其实也会执行一次“翻译”的过程。只不过这个“翻译”的过程是将dex字节码进行优化，也就是由dex文件生成odex文件。这个过程由安装服务PackageManagerService请求守护进程installd来执行的。从这个角度来说，在应用安装的过程中将dex字节码翻译成本地机器码对原来的应用安装流程基本上就不会产生什么影响。

我们知道，Android系统在启动的时候，会创建一个Zygote进程，充当应用程序进程孵化器。Zygote进程在启动的过程中，又会创建一个Dalvik虚拟机。Zygote进程是通过复制自己来创建新的应用程序进程的。这意味着Zygote进程会将自己的Dalvik虚拟机复制给应用程序进程。通过这种方式就可以大大地提高应用程序的启动速度，因为这种方式避免了每一个应用程序进程在启动的时候都要去创建一个Dalvik。事实上，Zygote进程通过自我复制的方式来创建应用程序进程，省去的不仅仅是应用程序进程创建Dalvik虚拟机的时间，还能省去应用程序进程加载各种系统库和系统资源的时间，因为它们在Zygote进程中已经加载过了，并且也会连同Dalvik虚拟机一起复制到应用程序进程中去

#### Zygote进程中的Dalvik虚拟机和ART虚拟机都是从AndroidRuntime::start这个函数开始创建的。
#### 应用程序在安装过程中将dex字节码翻译为本地机器码的过程
Android系统通过PackageManagerService来安装APK，在安装的过程，PackageManagerService会通过另外一个类Installer的成员函数dexopt来对APK里面的dex字节码进行优化。

```java
public final class Installer {  
    ......  
  
    public int dexopt(String apkPath, int uid, boolean isPublic) {  
        StringBuilder builder = new StringBuilder("dexopt");  
        builder.append(' ');  
        builder.append(apkPath);  
        builder.append(' ');  
        builder.append(uid);  
        builder.append(isPublic ? " 1" : " 0");  
        return execute(builder.toString());  
    }  
  
    ......  
}  
```

Installer通过socket向守护进程installd发送一个dexopt请求，这个请求是由installd里面的函数dexopt来处理的。

1. 函数dexopt首先是读取系统属性persist.sys.dalvik.vm.lib的值，接着在/data/dalvik-cache目录中创建一个odex文件。这个odex文件就是作为dex文件优化后的输出文件。
2. 函数dexopt通过fork来创建一个子进程。如果系统属性persist.sys.dalvik.vm.lib的值等于libdvm.so，那么该子进程就会调用函数run_dexopt来将dex文件优化成odex文件。另一方面，如果系统属性persist.sys.dalvik.vm.lib的值等于libart.so，那么该子进程就会调用函数run_dex2oat来将dex文件翻译成oat文件，实际上就是将dex字节码翻译成本地机器码，并且保存在一个oat文件中。
3. 无论是对dex字节码进行优化，还是将dex字节码翻译成本地机器码，最终得到的结果都是保存在相同名称的一个odex文件里面的，但是前者对应的是一个dey文件（表示这是一个优化过的dex），后者对应的是一个oat文件（实际上是一个自定义的elf文件，里面包含的都是本地机器指令）。通过这种方式，原来任何通过绝对路径引用了该odex文件的代码就都不需要修改了。















