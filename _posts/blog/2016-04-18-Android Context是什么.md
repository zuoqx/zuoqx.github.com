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



