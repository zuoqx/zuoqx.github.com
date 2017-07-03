---
layout: post
title: Android插件化开发之Service插件化
description: Android插件化开发之Service插件化
category: blog
---


### 概述
从源码分析来看，Service组件与Activity有着非常多的相似之处：它们都是通过Context类完成启动，接着通过ActivityMnagaerNative进入AMS，最后又通过IApplicationThread这个Binder IPC到App进程的Binder线程池，然后通过H转发消息到App进程的主线程，最终完成组件生命周期的回调；对于Service组件，看起来好像可以沿用Activity组件的插件化方式：Hook掉ActivityManagerNative以及H类，但事实真的如此吗？

### Service与Activity的异同
1. 用户交互对于生命周期的影响

	首先，Activity与Service组件最大的不同点在于，Activity组件可以与用户进行交互；这一点意味着用户的行为会对Activity组件产生影响，对我们来说最重要的影响就是Activity组件的生命周期；用户点击按钮从界面A跳转到界面B，会引起A和B这两个Activity一系列生命周期的变化。而Service组件则代表后台任务，除了内存不足系统回收之外，它的生命周期完全由我们的代码控制，与用户的交互无关。
	
	Activity组件的生命周期受用户交互影响，而这种变化只有Android系统才能感知，因此我们必须把插件的Activity交给系统管理，才能拥有完整的生命周期；但Service组件的生命周期不受外界因素影响，那么自然而然，我们可以手动控制它的生命周期，就像我们对于BroadcastReceiver的插件化方式一样！

2. Activity的任务栈

	任务栈的概念使得Activtiy的创建就代表着入栈，销毁则代表出栈；又由于Activity代表着与用户交互的界面，所以这个栈的深度不可能太深——Activity栈太深意味着用户需要狂点back键才能回到初始界面，这种体验显然有问题；因此，插件框架要处理的Activity数量其实是有限的，所以我们在AndroidManifest.xml中声明有限个StubActivity就能满足插件启动近乎无限个插件Activity的需求。

	但是Service组件不一样，理论情况下，可以启动的Service组件是无限的——除了硬件以及内存资源，没有什么限制它的数目；如果采用Activity的插件化方式，就算我们在AndroidMafenist.xml中声明再多的StubService，总有不能满足插件中要启动的Service数目的情况出现。也许有童鞋会说，可以用一个StubService对应多个插件Service，这确实能解决部分问题；但是，下面的这个区别让这种设想彻底泡汤。
	
3. Service无法拥有多实例

	Service组件与Activity组件另外一个不同点在于，对同一个Service调用多次startService并不会启动多个Service实例，而非特定Flag的Activity是可以允许这种情况存在的，因此如果用StubService的方式，为了实现Service的这种特性，必须建立一个StubService到插件Service的一个Map，Map的这种一一对应关系使得我们使用一个StubService对应多个插件Service的计划成为天方夜谭。

**我们Hook掉ActivityManagerNative之后，像处理Activity一样，可以拦截对于startService以及stopService等方法的调用，拦截之后，我们可以直接对插件Service进行操作,可以吗？**

不行！！

1. 首先，Service存在的意义在于它作为一个后台任务，拥有相对较高运行时优先级；除非在内存及其不足威胁到前台Activity的时候，这个组件才会被系统杀死。上述这种实现完全把Service当作一个普通的Java对象使用了，因此并没有完全实现Service所具备的能力。

2. 其次，Activity以及Service等组件是可以指定进程的，而让Service运行在某个特定进程的情况非常常见——所谓的远程Service；用上述这种办法压根儿没有办法让某个Service对象运行在一个别的进程。Android系统给开发者控制进程的机会太少了，要么在AndroidManifest.xml中通过process属性指定，要么借助Java的Runtime类或者native的fork；这几种方式都无法让我们以一种简单的方式配合上述方案达到目的。

### 代理分发技术

注册一个真正的Service组件ProxyService，让这个Service承载一个真正的Service组件所具备的能力（进程优先级等）；当启动插件的服务比如PluginService的时候，我们统一启动这个ProxyService，当这个ProxyService运行起来之后，再在它的onStartCommand等方法里面进行分发，执行PluginService的onStartCommond等对应的方法；我们把这种方案形象地称为「代理分发技术」

代理分发技术也可以完美解决插件Service可以运行在不同的进程的问题——我们可以在AndroidManifest.xml中注册多个ProxyService，指定它们的process属性，让它们运行在不同的进程；当启动的插件Service希望运行在一个新的进程时，我们可以选择某一个合适的ProxyService进行分发。

### Service插件化的实现
#### Service插件化的实现
我们需要一个货真价实的Service组件来承载进程优先级等功能，因此需要在AndroidManifest.xml中声明一个或者多个（用以支持多进程）这样的Sevice：

```java
<service
        android:name="com.weishu.upf.service_management.app.ProxyService"
        android:process="plugin01"/>
```

#### 拦截startService等调用过程

要手动控制Service组件的生命周期，需要拦截startService,stopService等调用，并且把启动插件Service全部重定向为启动ProxyService（保留原始插件Service信息）；这个拦截过程需要Hook ActvityManagerNative:

#### 分发Service
Hook ActivityManagerNative之后，所有的插件Service的启动都被重定向了到了我们注册的ProxyService，这样可以保证我们的插件Service有一个真正的Service组件作为宿主；但是要执行特定插件Service的任务，我们必须把这个任务分发到真正要启动的Service上去；

#### 加载Service
我们可以在ProxyService里面把任务转发给真正要启动的插件Service组件，要完成这个过程肯定需要创建一个对应的插件Service对象，比如PluginService；但是通常情况下插件存在与单独的文件之中，正常的方式是无法创建这个PluginService对象的，宿主程序默认的ClassLoader无法加载插件中对应的这个类；所以，要创建这个对应的PluginService对象，必须先完成插件的加载过程，让这个插件中的所有类都可以被正常访问；这种技术我们在之前专门讨论过，并给出了「激进方案」和「保守方案」，不了解的童鞋可以参考文章 插件加载机制；这里我选择代码较少的「保守方案」为例（Droid Plugin中采用的激进方案）：

#### 匹配过程

上文中我们把启动插件Service重定向为启动ProxyService，现在ProxyService已经启动，因此必须把控制权交回原始的PluginService；在加载插件的时候，我们存储了插件中所有的Service组件的信息，因此，只需要根据Intent里面的Component信息就可以取出对应的PluginService。

#### 创建以及分发

插件被加载之后，我们就需要创建插件Service对应的Java对象了；由于这些类是在运行时动态加载进来的，肯定不能直接使用new关键字——我们需要使用反射机制。但是下面的代码创建出插件Service对象能满足要求吗？

Service作为Android系统的组件，最重要的特点是它具有Context；所以，直接通过反射创建出来的这个PluginService就是一个壳子——没有Context的Service能干什么？因此我们需要给将要创建的Service类创建出Conetxt；但是Context应该如何创建呢？我们平时压根儿没有这么干过，Context都是系统给我们创建好的。既然这样，我们可以参照一下系统是如何创建Service对象的；在上文的Service源码分析中，在ActivityThread类的handleCreateService完成了这个步骤，摘要如下：

现在我们已经创建出了对应的PluginService，并且拥有至关重要的Context对象；接下来就可以把消息分发给原始的PluginService组件了，这个分发的过程很简单，直接执行消息对应的回调(onStart, onDestroy等）即可；因此，完整的startService分发过程如下：

```java
public void onStart(Intent proxyIntent, int startId) {

    Intent targetIntent = proxyIntent.getParcelableExtra(AMSHookHelper.EXTRA_TARGET_INTENT);
    ServiceInfo serviceInfo = selectPluginService(targetIntent);

    if (serviceInfo == null) {
        Log.w(TAG, "can not found service : " + targetIntent.getComponent());
        return;
    }
    try {

        if (!mServiceMap.containsKey(serviceInfo.name)) {
            // service还不存在, 先创建
            proxyCreateService(serviceInfo);
        }

        Service service = mServiceMap.get(serviceInfo.name);
        service.onStart(targetIntent, startId);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

至此，我们已经实现了Service组件的插件化.

### 总结

不足：由于同一个进程的所有Service都挂载在同一个ProxyService上面，如果系统可用内存不够必须回收Service，杀死一个ProxyService会导致一大票的插件Service歇菜。

实际使用过程中，Service组件的更新频度并不高，因此直接把插件Service注册到主程序也是可以接受的；而且如果需要绑定远程Service，完全可以使用一个Service组件根据不同的Intent返回不同的IBinder，所以不实现Service组件的插件化也能满足工程需要。值得一提的是，我们对于Service组件的插件化方案实际上是一种「代理」的方式，用这种方式也能实现Activity组件的插件化，有一些开源的插件方案比如 DL 就是这么做的。

















