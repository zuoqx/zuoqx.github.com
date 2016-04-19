---
layout: post
title: Android Context是什么
description: 深入理解Context.
category: blog
---


### 1. Context简单认识 
* "上下文" ：贯穿整个应用
* "运行环境"：提供应用运行所需要的信息、资源、系统服务等
* "交互场景"：用户操作和系统交互场景，比如启动Activity、Service、以及系统服务等

### 2. Context的继承关系

![继承图](/images/context.jpg)  
说明：

1. `Context`是纯粹抽象类，它的具体实现类是`ContextImpl`  
2. `ContextWrapper`是`Context`的一个包装类(装饰者模式)，保存`ContextImpl`的一个实例

### 3. 不同`Context`分析
***`ContextImpl`***----真正实现`Context`功能的类

    class ContextImpl extends Context {
            // 各种系统服务Manager
            private static AlarmManager sAlarmManager;
            private static WifiManager sWifiManager;
            // apk相关信息
            LoadedApk mPackageInfo;
            private Resources mResources;
            // 主线程(UI 线程)
            ActivityThread mMainThread;
            private Context mOuterContext;
            private IBinder mActivityToken = null;
            // 主题
            private Resources.Theme mTheme = null;
            private PackageManager mPackageManager;
            private ActivityManager mActivityManager = null;
            private Context mReceiverRestrictedContext = null;
            private LayoutInflater mLayoutInflater = null;
            private static long sInstanceCount = 0;

            public static long getInstanceCount() {
                return sInstanceCount;
            }

            @Override
            public AssetManager getAssets() {
                return mResources.getAssets();
            }

            @Override
            public Resources getResources() {
                return mResources;
            }

            @Override
            public PackageManager getPackageManager() {
                ..............................
            }

            @Override
            public Looper getMainLooper() {
                return mMainThread.getLooper();
            }

            @Override
            public Context getApplicationContext() {
                return (mPackageInfo != null) ?
                        mPackageInfo.getApplication() : mMainThread.getApplication();
            }

            @Override
            public void setTheme(int resid) {
                mThemeResource = resid;
            }

            @Override
            public ClassLoader getClassLoader() {
                return mPackageInfo != null ?
                        mPackageInfo.getClassLoader() : ClassLoader.getSystemClassLoader();
            }

            @Override
            public SharedPreferences getSharedPreferences(String name, int mode) {
                ....................
            }

            @Override
            public File getExternalFilesDir(String type) {
                ....................
            }

            @Override
            public File getCacheDir() {
                .......................
            }

            @Override
            public File getExternalCacheDir() {
                ..................................
            }

            @Override
            public SQLiteDatabase openOrCreateDatabase(String name, int mode, CursorFactory factory) {
                File f = validateFilePath(name, true);
                SQLiteDatabase db = SQLiteDatabase.openOrCreateDatabase(f, factory);
                setFilePermissionsFromMode(f.getPath(), mode, 0);
                return db;
            }

            @Override
            public boolean deleteDatabase(String name) {
                .......................
            }

            @Override
            public void startActivity(Intent intent) {
                ...................................
            }

            @Override
            public void sendBroadcast(Intent intent) {
                ................................
            }

            @Override
            public void sendOrderedBroadcast(Intent intent,
                    String receiverPermission) {
                .................................
            }

            @Override
            public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
                return registerReceiver(receiver, filter, null, null);
            }

            @Override
            public void unregisterReceiver(BroadcastReceiver receiver) {
                ...................................
            }

            @Override
            public ComponentName startService(Intent service) {
                ....................................
            }

            @Override
            public boolean stopService(Intent service) {
                ....................................
            }

            @Override
            public boolean bindService(Intent service, ServiceConnection conn,
                    int flags) {
                ................................
            }

            @Override
            public void unbindService(ServiceConnection conn) {
                ..............................
            }

            // 注册系统的各种服务
            @Override
            public Object getSystemService(String name) {
                if (WINDOW_SERVICE.equals(name)) {
                    return WindowManagerImpl.getDefault();
                } else if (LAYOUT_INFLATER_SERVICE.equals(name)) {
                    synchronized (mSync) {
                        LayoutInflater inflater = mLayoutInflater;
                        if (inflater != null) {
                            return inflater;
                        }
                        mLayoutInflater = inflater =
                            PolicyManager.makeNewLayoutInflater(getOuterContext());
                        return inflater;
                    }
                } else if (ACTIVITY_SERVICE.equals(name)) {
                    return getActivityManager();
                } else if (INPUT_METHOD_SERVICE.equals(name)) {
                    return InputMethodManager.getInstance(this);
                } else if (ALARM_SERVICE.equals(name)) {
                    return getAlarmManager();
                } else if (ACCOUNT_SERVICE.equals(name)) {
                    return getAccountManager();
                } else if (POWER_SERVICE.equals(name)) {
                    return getPowerManager();
                } else if (CONNECTIVITY_SERVICE.equals(name)) {
                    return getConnectivityManager();
                } else if (THROTTLE_SERVICE.equals(name)) {
                    return getThrottleManager();
                } else if (WIFI_SERVICE.equals(name)) {
                    return getWifiManager();
                } else if (NOTIFICATION_SERVICE.equals(name)) {
                    return getNotificationManager();
                } else if (KEYGUARD_SERVICE.equals(name)) {
                    return new KeyguardManager();
                } else if (ACCESSIBILITY_SERVICE.equals(name)) {
                    return AccessibilityManager.getInstance(this);
                } else if (LOCATION_SERVICE.equals(name)) {
                    return getLocationManager();
                } else if (SEARCH_SERVICE.equals(name)) {
                    return getSearchManager();
                } else if (SENSOR_SERVICE.equals(name)) {
                    return getSensorManager();
                } else if (STORAGE_SERVICE.equals(name)) {
                    return getStorageManager();
                } else if (USB_SERVICE.equals(name)) {
                    return getUsbManager();
                } else if (VIBRATOR_SERVICE.equals(name)) {
                    return getVibrator();
                } else if (STATUS_BAR_SERVICE.equals(name)) {
                    synchronized (mSync) {
                        if (mStatusBarManager == null) {
                            mStatusBarManager = new StatusBarManager(getOuterContext());
                        }
                        return mStatusBarManager;
                    }
                } else if (AUDIO_SERVICE.equals(name)) {
                    return getAudioManager();
                } else if (TELEPHONY_SERVICE.equals(name)) {
                    return getTelephonyManager();
                } else if (CLIPBOARD_SERVICE.equals(name)) {
                    return getClipboardManager();
                } else if (WALLPAPER_SERVICE.equals(name)) {
                    return getWallpaperManager();
                } else if (DROPBOX_SERVICE.equals(name)) {
                    return getDropBoxManager();
                } else if (DEVICE_POLICY_SERVICE.equals(name)) {
                    return getDevicePolicyManager();
                } else if (UI_MODE_SERVICE.equals(name)) {
                    return getUiModeManager();
                } else if (DOWNLOAD_SERVICE.equals(name)) {
                    return getDownloadManager();
                } else if (NFC_SERVICE.equals(name)) {
                    return getNfcManager();
                }

                return null;
            }

            private AccountManager getAccountManager() {
                .........................
            }

            private ActivityManager getActivityManager() {
                .......................
            }

            private AlarmManager getAlarmManager() {
                ......................
            }

            private PowerManager getPowerManager() {
                .......................
            }

            @Override
            public Context createPackageContext(String packageName, int flags)
                throws PackageManager.NameNotFoundException {
                .
            }

            static ContextImpl createSystemContext(ActivityThread mainThread) {
                ContextImpl context = new ContextImpl();
                context.init(Resources.getSystem(), mainThread);
                return context;
            }

            ContextImpl() {           
            }

            final void init(LoadedApk packageInfo,
                    IBinder activityToken, ActivityThread mainThread) {
                init(packageInfo, activityToken, mainThread, null);
            }

            final void setOuterContext(Context context) {
                mOuterContext = context;
            }    
    }

***`ContextWrapper`***----`Context`的包装类

该类中通过`attachBaseContext`方法将`ContextImpl`对象赋值给`mBase`成员变量,然后调用`ContextImpl`对象的方法。

***`Application`***中的`Context`

1. `ActivityThread`在创建的时候(具体是在mian方法里)，会创建一个`ContextImpl`  对象，用于创建整个应用的Application对象  

        mInstrumentation = new Instrumentation();

        ContextImpl context = ContextImpl.createAppContext(this, getSystemContext().mPackageInfo);

        //利用ContextImpl创建整个应用的Application对象
        mInitialApplication = context.mPackageInfo.makeApplication(true, null);

        //调用Application对象的onCreate方法
        mInitialApplication.onCreate();

2. mPackageInfo其实是个LoadApk对象，代码如下：  

    **LoadedApk#makeApplication**  

        public Application makeApplication(boolean forceDefaultAppClass,
                Instrumentation instrumentation) {
            //第一次进来mApplication==null条件不满足，之后创建Activity的时候条件满足直接返回当前Application对象
            if (mApplication != null) {
                return mApplication;
            }
            Application app = null;
            try {
                //为Appliaction创建ContextImpl对象
                ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
                //调用Instrumentation类中的newApplication方法创建Application
                app = mActivityThread.mInstrumentation.newApplication(cl, appClass, appContext);
                //给ContextImpl设置外部引用
                appContext.setOuterContext(app);
            } 
          ....................
            return app;
        }

    **Instrumentation#newApplication**

        public Application newApplication(ClassLoader cl, String className, Context context)
                throws InstantiationException, IllegalAccessException, 
                ClassNotFoundException {
            return newApplication(cl.loadClass(className), context);
        }

        static public Application newApplication(Class<?> clazz, Context context)
                throws InstantiationException, IllegalAccessException, 
                ClassNotFoundException {
            Application app = (Application)clazz.newInstance();
            app.attach(context);
            return app;
        }

注：最终是调用`Instrumentation`的newApplication方法创建Application，并且保存当前Context引用到父类的mBase变量里，如下：

**Application#attach**

    public class Application extends ContextWrapper implements ComponentCallbacks2 {
        ..................
        final void attach(Context context) {
            // 调用父类ContextWrapper的方法
            attachBaseContext(context);
            mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
        }
    ...............
    }

**`Activity`中的`Context`**

当Application创建完成后，AMS会通过Binder机制，通知ActivityThread去创建需要的Activity了。最后会在ActivityThread的performLaunchActivity方法里创建：

**ActivityThread#performLaunchActivity**

    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ........................

        Activity activity = null;
        try {
            //通过Instrumentation类的newActivity方法来创建一个Activity对象
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            //跟创建Application类似，还是通过Instrumentation来创建
            activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
           ..........................
        }
        try {
            //获取当前应用的Application对象，该对象的唯一作用就是作为参数传递到Activity里，
            然后在Activity类中可以获得调用getApplication方法来获取Application对象
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

          .............................
            if (activity != null) {
                //为Activity创建ContextImpl对象
                Context appContext = createBaseContextForActivity(r, activity);
                //为Activity赋值初始化
                 activity.attach(appContext, this, getInstrumentation(), r.token,r.ident, app, r.intent, r.activityInfo, title, r.parent, r.embeddedID, r.lastNonConfigurationInstances, config,r.voiceInteractor);
               ...................
                //获取当前应用的主题资源
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                //设置主题
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                if (r.isPersistable()) {
                //辗转到Activity，调用Activity的生命周期onCreate方法
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                .............
                r.activity = activity;
                r.stopped = true;
                if (!r.activity.mFinished) {
                //调用Activity的生命周期onStart方法
                    activity.performStart();
                    r.stopped = false;
                }
                ......................
        return activity;
    }

**ActivityThread#createBaseContextForActivity方法**

    private Context createBaseContextForActivity(ActivityClientRecord r,
            final Activity activity) {
        ContextImpl appContext = ContextImpl.createActivityContext(this, r.packageInfo, r.token);
        //ContextImpl持有外部Activity对象的引用，目的是在ContextImpl类中注册一些服务，设置主题等都需要外部Activity对象的引用
        appContext.setOuterContext(activity);
        Context baseContext = appContext;
       .............
       return baseContext；
    }

**Activity是在attach方法里进行初始化的**

    public class Activity extends ContextThemeWrapper {

    ............................

    final void attach(.....) {

            //调用父类方法对mBase变量赋值
            attachBaseContext(context);

            //创建一个Activity的窗口
            mWindow = PolicyManager.makeNewWindow(this);

            //给Window窗口设置回调事件
            mWindow.setCallback(this);
            mWindow.setOnWindowDismissedCallback(this);
            mWindow.getLayoutInflater().setPrivateFactory(this);

            //设置键盘弹出状态
            if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
                mWindow.setSoftInputMode(info.softInputMode);
            }
            if (info.uiOptions != 0) {
                mWindow.setUiOptions(info.uiOptions);
            }
            mUiThread = Thread.currentThread();

            mMainThread = aThread;
            mInstrumentation = instr;
            mToken = token;
            mIdent = ident;

            //此处注意，将整个应用的Application对象赋值给Activity的mApplication成员变量。
            //目的是为了能在Activity中通过getApplication方法来直接获取Application对象
            mApplication = application;
           ......................
    }

    //在Activity中返回当前应用的Application对象
    /** Return the application that owns this activity. */
    public final Application getApplication() {
        return mApplication;
    }

***`Service`中的`Context`***

同样创建Service是AMS通过Binder机制，通知ActivityThread类去创建的，最后调用ActivityThread的handleCreateService方法来创建一个Service服务：

**ActivityThread#handleCreateService**

    private void handleCreateService(CreateServiceData data) {

        LoadedApk packageInfo = getPackageInfoNoCheck(data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
            //通过类加载器创建Service服务
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = (Service) cl.loadClass(data.info.name).newInstance();
            } 
            .............
        try {
          ............
          //此处为Service创建一个ContextImpl对象
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);

            //同样为ContextImpl类设置外部对象，目的还是让ContextImpl持有外部类的引用
            //在ContextImpl类中的许多方法需要使用到外部Context对象引用
            context.setOuterContext(service);
            ................
            //获得当前应用的Applicaton对象，该对象在整个应用中只有一份，是共享的。
            Application app = packageInfo.makeApplication(false, mInstrumentation);
            //将ContextImpl对象和Application对象作为attach方法参数来初始化Service
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());

            //Service初始化完成之后系统自动调用onCreate生命周期方法
            service.onCreate();
            ................
    }

**Service#attach**

    public abstract class Service extends ContextWrapper implements ComponentCallbacks2 {
        //默认构造方法调用父类构造方法并且参数为null，意味着构造方法里并没有对Service初始化
        public Service() {
            super(null);
        }

        //在Service中返回整个应用的Application对象
        /** Return the application that owns this service. */
        public final Application getApplication() {
            return mApplication;
        }
        ..............
     
        public final void attach(...) {
            //调用父类方法去注册ContextImpl对象
            attachBaseContext(context);
            mThread = thread;           // NOTE:  unused - remove?
            mClassName = className;
            mToken = token;
            //将整个应用的Applicaton对象赋值给Service类的成员变量mApplication
            mApplication = application;
            mActivityManager = (IActivityManager)activityManager;

        }

***总结***

1. ActivityThread是整个应用的主线程，所有关于Context的创建是都在这个类里；
2. 每个应用都会创建一个唯一Application对象，Activity或者Service持有当前的Application对象，通过getApplication方法获得；
3. Application、Activity、Service都会创建一个ContextImpl对象，然后注册到他们中，之后Application、Activity、Service就可以使用Context的所有功能了（真正实现抽象类Context功能的类就是ContextImpl）；
4. 一个App中Context的个数 = 1个Application + Activity的个数 + service的个数。

### 4. Context使用场景

| 功能                   | Application   | Service  |Activity|
| -------------          |:-------------:|:-------:|:-------:|
| Start an Activity      | NO1           | NO1      |YES     |
| Show a Dialog          | NO            | NO       |YES     |
| Layout Inflation       | YES           | YES      |YES     |
| Start a Service        | YES           | YES      |YES     |
| Bind to a Service      | YES           | YES      |YES     |
| Send a Broadcast           | YES       | YES      |YES     |
| Register BroadcastReceiver | YES       | YES      |YES     |
| Load Resource Values       | YES       | YES      |YES     |

**注意：**

1. NO1表示Application和Service可以启动一个Activity，但是需要创建一个新的task。比如你在Application中调用startActivity（intent）时系统会报如下错误：

        java.lang.RuntimeException: Unable to create application 
        com.xjp.toucheventdemo.MyApplication: android.util.AndroidRuntimeException: Calling 
        startActivity() from outside of an Activity  context requires the FLAG_ACTIVITY_NEW_TASK 
        flag. Is this really what you want?

2. 除了Activity可以创建一个Dialog，其他都不可以创建Dialog。比如在Application中创建Dialog会报如下错误：

        Caused by: java.lang.IllegalStateException: You need to use a Theme.AppCompat theme (or descendant) with this activity.

    原因是因为在创建Dialog的时候会使用到Context对象去获取当前主题信息，而Activity是继承自ContextThemeWrapper，该类是实现了关于主题功能的。

### 5. getApplication和getApplicationContext区别

    这两个方法返回值都是指向同一个Application对象，仅仅是返回类型和实现方法的地方不同而已。

1. `getApplicationContext`是ContextWrappe类里实现，返回的是一个Context对象：

        public class ContextWrapper extends Context {
             @Override
                public Context getApplicationContext() {
                    return mBase.getApplicationContext();
                }
        }

    由于mBase变量指向的是ContextImpl对象，因此真正实现的地方是ContextImpl类中

        class ContextImpl extends Context {
             @Override
                public Context getApplicationContext() {
                    return (mPackageInfo != null) ?mPackageInfo.getApplication() : mMainThread.getApplication();
                }
        }

2. `getApplication`返回的是当前应用的Application对象，由于一个应用只有一份LoadedApk对象，此处返回的也是系统的唯一Application对象
        
        public final class LoadedApk {
            ............
            Application getApplication() {
                    return mApplication;
                }
            ..........
        }

3. `getApplication`方法是在Activity，Service类中实现的；而`getApplicationContext`方法是在ContextWrapper类中实现的。因此调用方式不同：

        context.getApplicationContext(); 

        不可以这样：context.getApplication();

        可以这样：((Activity)context).getApplication();

### 6. Context内存泄漏问题

