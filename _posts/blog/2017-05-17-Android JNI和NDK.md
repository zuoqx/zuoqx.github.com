---
layout: post
title: Android JNI和NDK事件传递机制
description: Android事件传递机制
category: blog
---


### JavaVM 和JNIEnv
JNI提供两种主要数据结构，**“JavaVM”**和**“JNIEnv”**。其本质上是**指向函数表指针的指针**。

**JavaVM**提供执行接口函数，使用这些函数可以创建和销毁JavaVM。理论上每个进程可以有多个JavaVM，但Android只运行一个。

**JNIEnv**提供大多数的JNI函数，所以native函数的第一个参数是一个JNIEnv。

JNIEnv用于线程本地存储，因此，不能在线程之间共享JNIEnv变量。如果一个线程无法获取它的JNIEnv参数，那么应该共享JavaVM参数，然后使用函数**GetEnv**获取线程的JNIEnv

### 线程
在Android里，所有的线程都是Linux线程，都由内核调度。通常是从受控代码（managed code）开始，例如**Thread.start**,但是线程也可以在任何地方创建，并且依附（attached）到JavaVM上。比如，由**pthread_create**创建的线程，可以由函数JNI函数**AttachCurrentThread**或者**AttachCurrentThreadAsDaemon**来完成。在这之前，线程是没有JNIEnv的，不能执行JNI函数调用。

attach本地创建的线程，会创建一个**java.lang.Thread**对象，并且添加到主线程组里。已经attach的线程上调用**AttachCurrentThread**是多余的。

Android不会挂起正在执行本地代码的线程，如果当前垃圾回收器正在运行，或者调试器遇到问题需要挂起时，Android在下次JNI调用时才会暂停线程。

通过JNI依附的线程，在退出之前必须调用**DetachCurrentThread**。

### jclass,jmethodID,and jfieldID
通过本地代码访问Java对象的属性，按照下面的步骤：

* 使用**FindClass**获取类，以及类的对象
* 使用**GetFieldID**获取属性、方法的ID
* 使用**GetIntField**等合适的方法，或者对于属性、方法ID的值

类的引用、方法和属性ID这些在类卸载之前都是有效的。而类仅仅当和它关联的类加载器被垃圾回收器回收后，才会被卸载，这通常不太常见，但在Android里是可能的。因此，需注意的是，应该调用**NewGlobalRef**保存jclass这个类的引用。

如果类从来不会被卸载或者重新加载，当类加载的时候，自动缓存这些ID的代码如下：

```java
	 /*
     * We use a class initializer to allow the native code to cache some
     * field offsets. This native function looks up and caches interesting
     * class/field/method IDs. Throws on failure.
     */
    private static native void nativeInit();

    static {
        nativeInit();
    }
```
在C/C++方法里定义这个native方法，用来或者这些ID，当类初始化的时候，这个函数就会执行一次，当类没有被卸载，再次加载时，函数就会再次执行。

### 局部和全局引用
传入本地函数的所有参数，以及大多数JNI函数返回的对象，都是局部引用。也就是说，本地函数在当前线程执行期间是有效的。**即使本地函数返回后，这个对象仍然存活，这个引用也是无效的。**这种情况同样适用于**jobject**的子类，包括jclass、jstring、jarray。

获取全局引用的唯一方法是，通过调用函数**NewGlobalRef**和**NewWeakGlobalRef**。函数NewGlobalRef传入局部引用，返回全局引用。这个全局引用在调用**DeleteGlobalRef**之前，它一直是有效的。

注意，**jfield**和**jmethod**是透明类型，不是对象类型，也就没有必要传入到NewGlobalRef函数里，因为它们本身是不变的。像**GetStringUtfChars**和**GetByteArrayElements**函数返回的原始数据指针，也不是对象，也就可以在线程间传递。

有种情形需要注意的是，如果通过AttachCurrentThread依附的线程，


















































