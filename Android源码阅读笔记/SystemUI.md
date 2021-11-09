# Android原生SystemUI篇

### SystemUI简介

SystemUI，顾名思义就是Android的系统界面，主要包括界面上方的StatusBar，

![image-20211105174518857](/home/chaoyangzhang/snap/typora/42/.config/Typora/typora-user-images/image-20211105174518857.png)

### SystenUI启动流程

#### 1.SystemSever中的相关启动

开机时Java层会最先执行SystenSever.java中的程序，SystenSever中有startOtherServices()方法。startOtherServices()中会另起线程来执行mActivityManagerService.systemReady()方法，systemReady()中调用了startSystemUi()。

```java
public final class SystemServer {
    ...;
    private void startOtherServices() {
        ...;
        mActivityManagerService.systemReady(() -> {
            ...;
            try {
                startSystemUi(context, windowManagerF);
            }
        },BOOT_TIMINGS_TRACE_LOG);
    }
    private static void startSystemUi(Context context, WindowManagerService windowManager) {
        Intent intent = new Intent();
        intent.setComponent(new ComponentName("com.android.systemui",
                               "com.android.systemui.SystemUIService"));
        intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
        context.startServiceAsUser(intent, UserHandle.SYSTEM);
        windowManager.onSystemUiStarted();
    }
}
```

这里会引出有两条线：

- **会通过intent启动SystemUIService**
- **会调用WM的onSystemUiStarted()**

#### 2.SystemUIService这条线

##### 2.1.SystemUIService.java

SystemUIService.java主要有这两个内容：

- onCreate()方法中调用了SystemUIApplication的startServicesIfNeeded()方法来启动SystemUI的相关服务，SystemUIApplication是SystemUI这个APP的主要类。
- 重写了Service的dumpServices()方法

##### 2.2.SystemUIApplication中的startServicesIfNeeded()方法

- 首先获得所有需要启动的服务名称，并将这些服务的名称保存在数组services中。
- 通过名称获取这些服务对象，并把它们转为SystemUI类型的，保存在数组mServices中。
- 依次调用mServices[i].start();来启动这些服务。

```java
public void startServicesIfNeeded() {
    String[] names = getResources()
        .getStringArray(R.array.config_systemUIServiceComponents);
    startServicesIfNeeded("StartServices", names);
}
private void startServicesIfNeeded(String metricsPrefix, String[] services) {
    mServices = new SystemUI[services.length];
    final int N = services.length;
    for (int i = 0; i < N; i++) {
        ...;
        String clsName = services[i];
        Class cls;
        cls = Class.forName(clsName);
        Object o = cls.newInstance();
        if (o instanceof SystemUI.Injector) {
            o = ((SystemUI.Injector) o).apply(this);
        }
        mServices[i] = (SystemUI) o;
        
        mServices[i].mContext = this;
        mServices[i].mComponents = mComponents;
        ...;
        mServices[i].start();
    }
    mServicesStarted = true;
}
```

这里可以看出：

- **需要启动的服务都继承自SystemUI**
- **这些服务的名称保存在数组config_systemUIServiceComponents中**

config_systemUIServiceComponents包含的内容：

```xml
frameworks/base/packages/SystemUI/res/values/config.xml
    <string-array name="config_systemUIServiceComponents" translatable="false">
        <item>com.android.systemui.util.NotificationChannels</item>
        <item>com.android.systemui.keyguard.KeyguardViewMediator</item>
        <item>com.android.systemui.recents.Recents</item>
        <item>com.android.systemui.volume.VolumeUI</item>
        <item>com.android.systemui.statusbar.phone.StatusBar</item>
        <item>com.android.systemui.usb.StorageNotification</item>
        <item>com.android.systemui.power.PowerUI</item>
        <item>com.android.systemui.media.RingtonePlayer</item>
        <item>com.android.systemui.keyboard.KeyboardUI</item>
        <item>com.android.systemui.shortcut.ShortcutKeyDispatcher</item>
        <item>@string/config_systemUIVendorServiceComponent</item>
        <item>com.android.systemui.util.leak.GarbageMonitor$Service</item>
        <item>com.android.systemui.LatencyTester</item>
        <item>com.android.systemui.globalactions.GlobalActionsComponent</item>
        <item>com.android.systemui.ScreenDecorations</item>
        <item>com.android.systemui.biometrics.AuthController</item>
        <item>com.android.systemui.SliceBroadcastRelayHandler</item>
        <item>com.android.systemui.statusbar.notification.InstantAppNotifier</item>
        <item>com.android.systemui.theme.ThemeOverlayController</item>
        <item>com.android.systemui.accessibility.WindowMagnification</item>
        <item>com.android.systemui.accessibility.SystemActions</item>
        <item>com.android.systemui.toast.ToastUI</item>
        <item>com.android.systemui.wmshell.WMShell</item>
    </string-array>
```

##### 2.3.相关服务的start()方法

2.3.1.NotificationChannels

###### 2.3.1.NotificationChannels

- 对应通知，主要用来创建并初始化各类型的通知，以便systemui组件调用。

```java
public void start() {
    createAll(mContext);
}
public static void createAll(Context context) {
    //创建NotificationManager的对象，NotificationManager定义在frameworks/case/core/java/android/app/下
    final NotificationManager nm = context.getSystemService(NotificationManager.class);
    //创建电池通知，NotificationChannel也定义在frameworks/case/core/java/android/app/下
    final NotificationChannel batteryChannel = new NotificationChannel(...
                             NotificationManager.IMPORTANCE_MAX);//电池通知的级别为5，最高级别。
    final String soundPath = Settings.Global.getString(...);//电池通知的声音存储路径
    batteryChannel.setSound(...);//设置声音
    batteryChannel.setBlockable(true);//设置锁屏时可进行通知
    //创建警报通知，级别为4；一般用于确认操作，比如删除APP、下载APP等
    final NotificationChannel alerts = new NotificationChannel(...);
    //创建普通通知，级别为1；锁屏界面的通知。
    final NotificationChannel general = new NotificationChannel(...);
    //创建storage通知，当应用于电视时级别为3，否则为2
    final NotificationChannel storage = new NotificationChannel(...);
    //创建hint通知，级别为3
    final NotificationChannel hint = new NotificationChannel(...);
    //将这些通知纳入NotificationManager的管理，后续可以通过NotificationManager来获取并使用这些通知。
    nm.createNotificationChannels(Arrays.asList(
        alerts,general,storage,
        createScreenshotChannel(//额外增加屏幕截图的通知，但在Android P 中已经移除了。
            context.getString(R.string.notification_channel_screenshot),
            nm.getNotificationChannel(SCREENSHOTS_LEGACY)),
        batteryChannel,hint
    ));}
```

###### 2.3.2.KeyguardViewMediator

- 对应锁屏界面，初始化并设置锁屏的各种属性：锁屏广播、锁屏延迟、时间、锁屏声音、解锁声音等等。

###### 2.3.3.Recents

- 对应近期任务界面

###### 2.3.4.VolumeUI

- 对应音量调节对话框

###### 2.3.5.StatusBar

- 对应屏幕上方的状态栏，在start()方法中初始化了屏幕生命周期的观察者、WindowManager、DreamManager、display、controller等。
