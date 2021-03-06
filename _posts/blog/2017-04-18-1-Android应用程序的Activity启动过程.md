---
layout: post
title: Android应用程序的Activity启动过程
description: Android应用程序的Activity启动过程
category: blog
---


### 概述  
1. Android中，可以通过点击应用程序图标来启动Activity，以及通过Activity内部调用startActivity接口来启动新的Activity，这两种启动Activity的方式，都要借助于应用程序框架层的ActivityManagerService服务进程来完成的。  
2. Service也是由ActivityManagerService进程来启动的。在Android应用程序框架层中，ActivityManagerService是一个非常重要的接口，它不但负责启动Activity和Service，还负责管理Activity和Service。  
  
### 点击应用程序图标启动Activity    
1.  在Android系统中，应用程序是由Launcher启动起来的，Launcher本身也是一个应用程序，其它的应用程序安装后，就会Launcher的界面上出现一个相应的图标，点击这个图标时，Launcher调用 Launcher.startActivitySafely启动应用程序。  
2.  因为Launcher是一个Activity，所以函数调到Activity类的startActivityForResult函数
3.  而在Activity类内是调用Instrumentation.execStartActivity方法，Instrumentation类是用来监控应用程序和系统的交互。
4.  进而，通过Binder调到ActivityManagerService.startActivity的方法  
5.  ActivityManagerService通过Binder进程间通信机制通知Launcher进入Paused状态  
6.  Launcher通过Binder进程间通信机制通知ActivityManagerService，它已经准备就绪进入Paused状态，于是ActivityManagerService就创建一个新的进程，用来启动一个ActivityThread实例，即将要启动的Activity就是在这个ActivityThread实例中运行  
7.  ActivityThread通过Binder进程间通信机制将一个ApplicationThread类型的Binder对象传递给ActivityManagerService，以便以后ActivityManagerService能够通过这个Binder对象和它进行通信  
8.  ActivityManagerService通过Binder进程间通信机制通知ActivityThread，现在一切准备就绪，它可以真正执行Activity的启动操作了  
9.  而在ActivityThread内采用类加载机制，来启动Activity  
10. 启动之后Instrumentation调用Activity的onCreate方法，一个Activity真正启动起来了  
11. 应用程序默认Activity的启动过程，一般来说，这种默认Activity是在新的进程和任务中启动的，但也可以在同一个进程里启动，只要配置两个应用程序具有相同的uid和package，或者在AndroidManifest.xml配置文件的application标签或者activity标签中显式指定相同的process属性值，这样，不同的应用程序也可以在同一个进程中启动。  

#### Android应用程序内部启动Activity过程（startActivity）  
1. 应用程序内部启动非默认Activity的过程的源代码，这种非默认Activity一般是在原来的进程和任务中启动的  
2. 应用程序的MainActivity通过Binder进程间通信机制通知ActivityManagerService，它要启动一个新的Activity  
3. ActivityManagerService通过Binder进程间通信机制通知MainActivity进入Paused状态  
4. MainActivity通过Binder进程间通信机制通知ActivityManagerService，它已经准备就绪进入Paused状态，于是ActivityManagerService就准备要在MainActivity所在的进程和任务中启动新的Activity了  
5. ActivityManagerService通过Binder进程间通信机制通知MainActivity所在的ActivityThread，现在一切准备就绪，它可以真正执行Activity的启动操作了  
6. 点击应用程序图标启动Activity，这里在应用程序内部启动新的Activity的过程少了中间创建新的进程这一步，这是因为新的Activity是在已有的进程和任务中执行的，无须创建新的进程和任务。  
7. 

