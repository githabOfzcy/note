## Android原生SystemUI篇

### SystemUI简介

SystemUI，其实是一个系统级的APP，主要负责系统界面的运行逻辑和显示。它是Android中内置的，不能被卸载，但可以通过overlay的方式改变它的显示效果。原生的SystemUI代码位于/frameworks/base/packages/SystemUI目录下。



### SystenUI启动流程

时序图：

```sequence
title:SystemUI的启动流程
participant SystemServer as A
participant SystemUIService as B
participant SystemUIApplication as C
participant PhoneWindowManager as D
participant KeyguardServiceDelegate as E
participant KeyguadService as F
participant KeyguardViewMediator as H
participant ActivityManagerService as G

A->A:startOtherService
A->A:startSystemUI
A->B:context.startServiceAsUser
B->B:onCreate
B->C:startServicesIfNeeded
C->C:mService[i].start
C->H:start
A->D:onSystemUIStsrted
D->E:bindService
E->E:new ServiceConnection
A->G:systemReady
E->F:onSystemReady
F->H:onSystemReady
```

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
}
private static void startSystemUi(Context context
                                  ,WindowManagerService windowManager) {
    Intent intent = new Intent();
    intent.setComponent(new ComponentName("com.android.systemui",
                                          "com.android.systemui.SystemUIService"));
    intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
    //启动SystemUIService
    context.startServiceAsUser(intent, UserHandle.SYSTEM);
    //调用PhoneWindowManager的onSystemUiStarted()方法
    windowManager.onSystemUiStarted();
}
```

这里会有两条线：

- **第一个条通过intent启动SystemUIService**
- **第二条是调用WindowManagerService的onSystemUiStarted方法**

下面先分析第一条线（第2、3部分），再分析第二条线（第4部分）。

#### 2.启动SystemUIService

##### 2.1.SystemUIService.java

位于frameworks/base /packages/SystemUI/src/com/android/systemui/目录下

SystemUIService.java主要有两个内容：

- onCreate()方法中调用了SystemUIApplication的startServicesIfNeeded()方法来启动SystemUI的相关服务，SystemUIApplication是SystemUI这个APP的主要类。
- 重写了Service的dumpServices()方法

##### 2.2.SystemUIApplication中的startServicesIfNeeded()方法

- 首先获得所有需要启动的服务名称，并将这些服务的名称保存在数组services中。
- 通过名称获取这些服务对象，并把它们转为SystemUI类型的，保存在数组mServices中。
- 依次调用mServices[i].start();来启动这些服务。

```java
public void startServicesIfNeeded() {
    String[] names = getResources()
        //获取config_systemUIServiceComponents字符串数组的值，是在xml文件中写死的
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
        //通过反射获取服务名称对应的class
        cls = Class.forName(clsName);
        Object o = cls.newInstance();
        if (o instanceof SystemUI.Injector) {
            //再通过class获取相应的服务类的实例对象
            o = ((SystemUI.Injector) o).apply(this);
        }
        mServices[i] = (SystemUI) o;
        //设置该实例对象的属性
        mServices[i].mContext = this;
        mServices[i].mComponents = mComponents;
        ...;
        //调用该对象的.start()方法
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

下面依次介绍这些服务对应的UI界面

#### 3.启动SystemUIService相关服务

##### 3.1.NotificationChannels

- 对应通知，主要用来创建并初始化各类型的通知，以便systemui组件调用。

##### 3.2.KeyguardViewMediator

- 对应锁屏界面，初始化并设置锁屏的各种属性：锁屏广播、锁屏延迟、时间、锁屏声音、解锁声音等等。

##### 3.3.Recents

- 对应近期任务界面

##### 3.4.VolumeUI

- 对应音量调节对话框

<figure class="forth">
    <img src="/home/chaoyangzhang/Desktop/GM工作/GM源码阅读/SystemUI/图片/notification.png" width="140">
    <img src="/home/chaoyangzhang/Desktop/GM工作/GM源码阅读/SystemUI/图片/keyguard.png" width="140">
    <img src="/home/chaoyangzhang/Desktop/GM工作/GM源码阅读/SystemUI/图片/recents.png" width="140">
    <img src="/home/chaoyangzhang/Desktop/GM工作/GM源码阅读/SystemUI/图片/volume.png" width="140">
</figure>


##### 3.5.StatusBar

- 对应屏幕上方的状态栏；在start()方法中初始化了屏幕生命周期的观察者、WindowManager、DreamManager、display、controller等。

##### 3.6.StorageNotification

- 对应存储设备的通知，比如SD卡拔插。

##### 3.7.PowerUI

- 对应电源显示界面，比如充电、低电量等

##### 3.8.RingtonePlayer

- 对应手机铃声，比如闹钟、来电铃声等

##### 3.9.KeyboardUI

- 对应键盘

<figure class="third">
    <img src="/home/chaoyangzhang/Desktop/GM工作/GM源码阅读/SystemUI/图片/statusbar.png" width="140">
    <img src="/home/chaoyangzhang/Desktop/GM工作/GM源码阅读/SystemUI/图片/keyboard.png" width="140">
    <img src="/home/chaoyangzhang/Desktop/GM工作/GM源码阅读/SystemUI/图片/toast.png" width="140">
</figure>


##### 3.10.ShortcutKeyDispatcher

- 对应快捷方式

##### 3.11.@string/config_systemUIVendorServiceComponent

- 自定义systemUI服务，会链接到com.android.systemui.VendorServices，开发者可以在这里定义自己的SystemUI相关服务

##### 3.12.GarbageMonitor$Service

- 负责SystemUI的维护检查和垃圾清理

##### 3.13.LatencyTester

- 测试类

##### 3.14.ScreenDecorations

- 对应屏幕装饰，比如全面屏适配刘海

##### 3.15.InstantAppNotifier

- 对应即时通知的显示

##### 3.16.ThemeOverlayController

- 对应系统主题的替换

##### 3.17.accessibility/

- SystemUI与framework的通信、SystemUI与WindowsManager的通信

##### 3.18.ToastUI

- 对应toast提示

#### 4.PhoneWindowManager的启动

在这条线中，主要内容是将SystenUI下的KeyguardService绑定到PhoneWindowManager的context中。

##### 4.1.在SystemServer中的获取流程

- 在调用startSystemUi()方法时，会传入两个参数，一个是context，另一个是WindowManagerService类型的windowManagerF；windowManagerF被定义在startOtherService()中并被赋值为wm，wm也是定义在startOtherService()中，通过调用WindowManagerService.main()来赋值。

```java
private void startOtherServices(@NonNull TimingsTraceAndSlog t) {
    WindowManagerService wm = null;
    wm = WindowManagerService.main(context, inputManager, !mFirstBoot, mOnlyCore,
                                   new PhoneWindowManager(), mActivityManagerService.mActivityTaskManager);
    ...;
    final WindowManagerService windowManagerF = wm;
    ...;
    startSystemUi(context, windowManagerF);
}
```

- 再来看WindowManagerService的main()方法，在WindowManagerService中，调用main()会返回WindowManagerService的单例对象sInstance，而传入的对象new PhoneWindowManager()则会被添加到LocalServices中作为WindowManager的对象，其他地方通过反射调用WindowManagerPolicy.class对应的对象是就会获得这个对象。

```java
public static WindowManagerService main(...,WindowManagerPolicy policy,...) {
    sInstance = new WindowManagerService(..., policy,...);
    return sInstance;
}
private WindowManagerService(..., WindowManagerPolicy policy,...) {
    ...;
    mPolicy = policy;
    ...;
    LocalServices.addService(WindowManagerPolicy.class, mPolicy);
}
```

##### 4.2.在WindowManagerService中的调用

- 在SystemServre中的startSystemUi()方法中，会调用windowManager的onSystemUiStarted()方法，而在WindowManagerService中，onSystemUiStarted()会调用对应WindowManager的onSystemUiStarted()方法。也就是说会调用PhoneWindowManager的onSystemUiStarted()方法。

```java
//SystemServer.java
private static void startSystemUi(Context context, WindowManagerService windowManager) {
    ...;
    windowManager.onSystemUiStarted();
}
//WindowManagerService.java
public void onSystemUiStarted() {
    mPolicy.onSystemUiStarted();
}
```

##### 4.3.PhoneWindowManager的onSystemUiStarted()方法

- PhoneWindowManager重写了WindowManagerPolicy的onSystemUiStarted()方法，并在其中调用了bindKeyguard()方法，而bindKeyguard()方法调用了KeyguardServiceDelegate中的bindService()方法。

```java
@Override
public void onSystemUiStarted() {
    bindKeyguard();
}
private void bindKeyguard() {
    ...;
    //在调用bindService时会传入mContext作为参数
    mKeyguardDelegate.bindService(mContext);
}
mKeyguardDelegate = new KeyguardServiceDelegate(mContext,new StateCallback() {
    //创建KeyguardServiceDelegate对象，传入的参数包括一个匿名StateCallback对象
    //需要重写它的onTrustedChanged()和onShowingChanged()方法。
    @Override
    public void onTrustedChanged() {
        mWindowManagerFuncs.notifyKeyguardTrustedChanged();
    }
    @Override
    public void onShowingChanged() {
        mWindowManagerFuncs.onKeyguardShowingAndNotOccludedChanged();
    }
});
```

##### 4.4.KeyguardServiceDelegate中的bindService()方法

- KeyguardServiceDelegate是用来缓存Keyguard状态的，其中包含一个内部类KeyguardState，KeyguardState中有记录各种记录keyguard状态的值，并且在每次初始化时会重置。
- KeyguardServiceDelegate的bindService()方法会绑定位于SystemUI下的KeyguardService。

```java
public void bindService(Context context) {
    Intent intent = new Intent();
    //字符config_keyguardComponent的内容是com.android.systemui/com.android.systemui.keyguard.KeyguardService，
    //也就是说这里创建了systemui下的KeyguardService对应的Component，并绑定了这个服务。
    final ComponentName keyguardComponent = ComponentName.unflattenFromString(
        resources.getString(com.android.internal.R.string.config_keyguardComponent));
    intent.setComponent(keyguardComponent);
    ...;
    //绑定服务，这里会初始化mKeyguardConnection这个值，mKeyguardConnection中有对KeyguardServiceWrapper的调用
    if (!context.bindServiceAsUser(intent, ...)) {
        //如果无法绑定，需要设置mKeyguardState中对应的值
        mKeyguardState.showing = false;
        ...;
    }}
```

##### 4.5.绑定KeyguardService的具体实现

- 在KeyguardServiceWrapper的构造方法中，要求传入一个KeyguardService类型的参数，并把它赋值给mService
- 在每个生命周期，都会调用mService的相应生命周期的方法。

```java
//KeyguardServiceDelegate.java
private final ServiceConnection mKeyguardConnection = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
        //获得一个KeyguardServiceWrapper的对象
        mKeyguardService = new KeyguardServiceWrapper(mContext,IKeyguardService.Stub.asInterface(service), mCallback);
        if (mKeyguardState.systemIsReady) {
            //在各个生命状态都会调用这个对象的对应方法，让它们两个的生命周期保持一致
            mKeyguardService.onSystemReady();
            if (mKeyguardState.currentUser != UserHandle.USER_NULL) {
                mKeyguardService.setCurrentUser(mKeyguardState.currentUser);
            }
            ...;}
        ...;}};
//KeyguardServiceWrapper.java
public KeyguardServiceWrapper(..., IKeyguardService service,...) {
    //构造方法中会将传入的KeyguardService的对象
    mService = service;
}
...;
@Override
public void onFinishedWakingUp() {
    //调用KeyguardService对象的对应方法
    mService.onFinishedWakingUp();
}
...;
```

##### 4.6.KeyguardService的探究

- 在KeyguardService的绑定方法中，不同的生命周期都会调用KeyguardViewMediator的相应方法。至此完成对Keyguard的绑定。

```java
@Override
public IBinder onBind(Intent intent) {
    return mBinder;
}
private final IKeyguardService.Stub mBinder = new IKeyguardService.Stub() {
    ...;
    @Override 
    public void verifyUnlock(IKeyguardExitCallback callback) {
        ...;
        mKeyguardViewMediator.verifyUnlock(callback);
    }
}
```





### keyguard流程分析

#### 1.keyguard简介

##### 1.1.作用范围

keyguard主要包括锁屏的界面显示、锁屏状态下的消息处理、解锁等锁屏的相关内容。

##### 1.2.前世今生

*在最初的Android中，Keyguard是作为一个与SystemUI平级的系统级应用存在的，其代码位于frameworks/base/packages/Keyguard目录下。从Android5.0开始，Keyguard逐渐与SystemUI合并，到Android8.0已经完全成为了SystemUI的一部分。现在（Android10.0）打开frameworks/base/packages/Keyguard目录已经空了（遗留的AndroidManifest.xml见证了这段历史）。*

##### 1.3.代码结构

keyguard界面各个View的代码位于frameworks/base/packages/SystemUI/src/com/android/keyguard/目录下，这个目录下也包括状态回调的KeyguardUpdateMonitor.java。

逻辑处理的代码则位于frameworks/base/packages/SystemUI/src/com/android/systemui/keyguard目录下。

锁屏状态下的通知显示、状态栏变更等的代码位于frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone下。

##### 1.4.生命周期

keyguard的生命周期可以从KeyguardService中看到，主要包含以下几个生命周期：

```java
public void onSystemReady() {}//开机时系统准备就绪

public void onDreamingStarted() {}//屏保开始
public void onDreamingStopped() {}//屏保结束

public void onStartedWakingUp() {}//开始唤醒
public void onFinishedWakingUp() {}//完成唤醒
public void onScreenTurningOn() {}//开始亮屏
public void onScreenTurnedOn() {}//完成亮屏

public void onStartedGoingToSleep() {}//开始进入睡眠
public void onFinishedGoingToSleep() {}//完成进入睡眠
public void onScreenTurningOff() {}//开始息屏
public void onScreenTurnedOff() {}//完成息屏
```

#### 2.keyguard的主要界面介绍

##### 2.1.keyguard界面

- 也就是包括时间、消息通知、锁屏壁纸等内容的界面
- 由StatusBar控制，在NotificationPanelViewController中生成view控件。

##### 3.2.bouncer界面

- 也就是输密码的界面，keyguard界面上滑进入bouncer界面
- 由KeyguardBouncer控制，在KeyguardSecurityContainer中生成Bouncer界面

#### 3.keyguard的开机启动流程

```sequence
title:Keyguard的开机启动流程
participant KeyguardViewMediator as A
participant StatusBarKeyguardViewManager as B
participant StatusBar as C
participant KeyguardBouncer as D

A->A:start
A->A:handleSystemReady
A->A:handleShow
A->B:show
B->B:reset
B->C:showKeyguard
C->C:showKeyguardImpl
B->D:prepare
C->D:show
```



- 开机启动时，keyguard主要进行onSystemReady的生命周期

##### 3.1.KeyguardViewMediator.start()

keyguard中最主要的类，负责处理keyguard与其他对象的交互、手机状态变化时对其他keyguard类的回调。比如对keyguard状态的查询、电源状态的影响、锁屏状态下消息的显示、输入限制等。

- SystemUI启动时会最先调用KeyguardViewMediator的start()方法，start()又调用了setupLocked()，在setupLocked()中进行了一系列的初始化。
- 完成初始化之后等待systemReady触发PhoneWindowManager的onSystemReady()方法，最终会调用KeyguardViewMediator的onSystemReady()方法。

```java
@Override
public void start() {setupLocked();}
private void setupLocked() {
    //设置唤醒锁
    mShowKeyguardWakeLock = mPM.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "show keyguard");
    mShowKeyguardWakeLock.setReferenceCounted(false);
    //注册广播
    ...;
    //设置时钟
    mAlarmManager = (AlarmManager) mContext.getSystemService(Context.ALARM_SERVICE);
    //设置当前用户
    KeyguardUpdateMonitor.setCurrentUser(ActivityManager.getCurrentUser());
    //判断keyguard是否可用，并调用setShowingLocked()方法。
    if (isKeyguardServiceEnabled()) {
        setShowingLocked(..., true);
    } else {
        setShowingLocked(false, true);
    }
    //初始化锁屏声音
    mLockSounds = new SoundPool.Builder()
        ... 
        .build();
    ...;
}
```

##### 3.2.KeyguardViewMediator.onSystemReady()

- onSystemReady()会给mHandler发送SYSTEM_READY的信息，收到信息后会调用handleSystemReady()方法。
- handleSystemReady()的主要内容是在同步线程下调用了doKeyguardLocked()并传入null作为参数来进行锁屏操作。
- doKeyguardLocked()在判断了各种情况（比如是否设置了禁用锁屏等）后，把传入的null作为参数调用了showLocked()方法。

```java
public void onSystemReady() {mHandler.obtainMessage(SYSTEM_READY).sendToTarget();}
private Handler mHandler = new Handler(Looper.myLooper(), null, true) {
    //只要涉及到与keyguard的交互，都需要在此Handler中进行线程通信以调用keyguard的相应操作。
    //这样可以通过多线程增加启动速度
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
                ...;
            case SYSTEM_READY:
                handleSystemReady();
                break;}}};
private void handleSystemReady() {
    synchronized (this) {
        doKeyguardLocked(null);
        ...;}
    ...;}
private void doKeyguardLocked(Bundle options) {
    ...;//一系列状态的判断，比如是否有锁屏，是否有密码等。
    showLocked(options);//显示锁屏界面
}
```

##### 3.3.KeyguardViewMediator.showLocked()

- showLocked()会给mHandler发送SHOW的信息，收到信息后会调用handleShow()方法；这期间依然会传递值为null的Bundle对象。
- 在handleShow()方法中，会调用setShowingLocked来进行输入限制的相关调用
- 另外会调用StatusBarKeyguardViewManager的show()方法，进行keyguard的内容绘制
- 最后调用KeyguardDisplayManager控制NavigationBar不显示

```java
private void showLocked(Bundle options) {
    ...;
    Message msg = mHandler.obtainMessage(SHOW, options);
    mHandler.sendMessage(msg);
    ...;
}
public void handleMessage(Message msg) {
    //调用keyguard的锁屏同样要在Handler中进行
    switch (msg.what) {
        case SHOW:
            handleShow((Bundle) msg.obj);
            break;
            ...;}}
private void handleShow(Bundle options) {
    //处理锁屏逻辑的方法
    ...;
    synchronized (KeyguardViewMediator.this) {
        ...;//判断是否已经SystenReady
        ...;//一系列keyguard的状态设置
        setShowingLocked(true);//会调用到updateInputRestrictedLocked()方法，进行输入受限状态的更改
        mKeyguardViewControllerLazy.get().show(options);//调用StatusBarKeyguardViewManager的show()方法
        ..;//StatusBar、PowerManager等的状态更新
        ...;//设置keyguardStateGingAway的值为false
    }
    mKeyguardDisplayManager.show();//设置NavigationBar为不显示
}
```

##### 3.4.StatusBarKeyguardViewManager.show()

- 进行一系列的状态变更通知，并且调用StatuaBar绘制keyguard。

```java
@Override
public void show(Bundle options) {
    mShowing = true;
    //打开notification显示，并打开notification的输入事件响应
    mNotificationShadeWindowController.setKeyguardShowing(true);
    ...;//设置keyguard的相关状态
    reset(true);//重新启动时的相关设置
}
public void reset(boolean hideBouncerWhenShowing) {
    if (mShowing) {
        mNotificationPanelViewController.resetViews(true);//可显示通知
        showBouncerOrKeyguard(hideBouncerWhenShowing);//显示keyguard
        resetAlternateAuth(false);//禁用除密码外的任何身份验证方式
        mKeyguardUpdateManager.sendKeyguardReset();//更新监听器KeyguardUpdateMonitor的监听状态
    }
}
protected void showBouncerOrKeyguard(boolean hideBouncerWhenShowing) {
    mStatusBar.showKeyguard();//调用statusbar.showKeyguard
    hideBouncer(shouldDestroyViewOnReset());//隐藏bouncer
    mBouncer.prepare();//初始化bouncer
    updateStates();//更新navigationBar、statusbar等的显示状态
}
```

##### 3.5.StatusBar.showKeyguard()

- keyguard界面是由StatusBar控制的，在StatusBar的inflateStatusBarWindow()，会生成NotificationPanelViewController的实例对象，NotificationPanelViewController是专门用来绘制StatusBar相关的view的。
- showKeyguard()会调用showKeyguardImpl()来显示Keyguard。

```java
private void inflateStatusBarWindow() {
    ...;
    mNotificationPanelViewController = statusBarComponent.getNotificationPanelViewController();
}
//NotificationPanelViewController.java
private void onFinishInflate() {
    //keyguard的主体view
    mKeyguardStatusBar = mView.findViewById(R.id.keyguard_header);
    ...;
    //keyguard底部的view(包括手电筒、相机等)
    mKeyguardBottomArea = (KeyguardBottomAreaView) mLayoutInflater.inflate(
                R.layout.keyguard_bottom_area, mView, false);
}
```

##### 3.6.KeyguardBouncer.prepare()

- 在这里进行bouncer的初始化

```java
public void prepare() {
    ensureView();
    showPrimarySecurityScreen();
}
protected void ensureView() {
    inflateView();//进行bouncer界面的view绘制
}
private void showPrimarySecurityScreen() {
    //调用KeyguardViewController展示密码解锁界面
    mKeyguardViewController.showPrimarySecurityScreen();
}
//KeyguardHostViewController.java
public void showPrimarySecurityScreen() {
    mKeyguardSecurityContainerController.showPrimarySecurityScreen(false);
}//会调用KeyguardSecurityContainerController中的方法进行密码界面相关的判断，比如密码形式等
```

#### 4.keyguard的亮屏与解锁

- 在keyguard的亮屏和解锁过程中，会经过**onStartedWakingUp() --> onScreenTurningOn() --> onScreenTurnedOn() --> onFinishedWakingUp()**四个生命周期

##### 4.1.PowerManager.weakUp()

- 解锁的第一步要先按电源键。这时就会调用到PowerManager中的weakUp()方法。
- 在经过一系列的调用后，最终会调用到PhoneWindowManager的startedWakingUp()方法，中间的流程我们就不详细探究了。

```java
//PowerManager.java
final IPowerManager mService;
public void wakeUp(long time, @WakeReason int reason, String details) {
    mService.wakeUp(time, reason, details, mContext.getOpPackageName());
}
```

##### 4.2.PhoneWindowManager.startedWakingUp()

- PhoneWindowManager已经和keyguardService绑定了，在这里会先会调用KeyguardServiceDelegate的onStartedWakingUp()。
- KeyguardServiceDelegate再调用keyguardService的onStartedWakingUp()。

```java
@Override
public void startedWakingUp(@PowerManager.WakeReason int pmWakeReason) {
    mKeyguardDelegate.onStartedWakingUp(pmWakeReason, mCameraGestureTriggered);
}
//keyguardService.java
@Override
public void onStartedWakingUp(
    mKeyguardViewMediator.onStartedWakingUp(cameraGestureTriggered);
}
```

##### 4.3.KeyguardViewMediator.onStartedWakingUp()

- keyguardService会再调用KeyguardViewMediator的onStartedWakingUp()，在这里进行唤醒的操作。
- onStartedWakingUp()阶段会先调用doKeyguardLocked，着这里进行一系列是否要展示keyguard的判断后通过Handler调用handleShow。后续内容见3.3

```java
public void onStartedWakingUp(boolean cameraGestureTriggered) {
    doKeyguardLocked(null);//启动keyguard
    notifyStartedWakingUp();
}
private void doKeyguardLocked(Bundle options) {
    ...;//判断是否有开启锁屏、是否有其他APP禁用锁屏、是否是系统用户等
    showLocked(options);
}
private void showLocked(Bundle options) {
    Message msg = mHandler.obtainMessage(SHOW, options);
    mHandler.sendMessage(msg);
}
```

##### 4.4.KeyguardViewMediator.notifyStartedWakingUp()

- 在执行doKeyguardLocked后会接着执行notifyStartedWakingUp()方法，通过Handler调用handleNotifyStartedWakingUp方法
- handleNotifyStartedWakingUp()会调用StatusBarKeyguardViewManager的onStartedWakingUp()方法
- StatusBarKeyguardViewManager会再调用StatusBar进行动画的显示。

```java
private void notifyStartedWakingUp() {
    mHandler.sendEmptyMessage(NOTIFY_STARTED_WAKING_UP);
}
private void handleNotifyStartedWakingUp() {
    mKeyguardViewControllerLazy.get().onStartedWakingUp();
}
//StatusBarKeyguardViewManager.java
@Override
public void onStartedWakingUp() {
    mStatusBar.getNotificationShadeWindowView().getWindowInsetsController()
        .setAnimationsDisabled(false);
}
```

##### 4.5.KeyguardViewMediator.onScreenTurningOn()

- onScreenTurningOn()会通过Handler调用handleNotifyScreenTurningOn()方法。
- 到这里就完成了亮屏前的准备工作，接下来就返回到PowerManagerService的updatePowerStateLocked()方法中进行下一步。

```java
private void handleNotifyScreenTurningOn(IKeyguardDrawnCallback callback) {
    mKeyguardViewControllerLazy.get().onScreenTurningOn();
    //但是由于在息屏或者开机时就已经绘制好了keyguard，因此不需要再绘制一遍。
    //在android 10中StatusBarKeyguardViewManager在onScreenTurningOn()和onScreenTurnedOn()阶段就没有进行任何操作了
}
```

#### 5.keyguard的息屏流程

- 在keyguard的息屏过程中，会经过**onStartedGoingToSleep() --> onScreenTurningOff() --> onScreenTurnedOff() --> onFinishedGoingToSleep()**四个生命周期

##### 5.1.PowerManager.goToSleep()

- 与亮屏时类，也会先按电源键，也就是会调用PowerManager的goToSleep()方法
- 和亮屏时一样，在经过PhoneWindowManager的中转后会调用到KeyguardService中的onStartedGoingToSleep()
- KeyguardService依旧会调用KeyguardViewMediator的onStartedGoingToSleep()方法

```java
public void goToSleep(long time, int reason, int flags) {
    mService.goToSleep(time, reason, flags);
}
//KeyguardService.java
@Override
public void onStartedGoingToSleep(@PowerManager.GoToSleepReason int pmSleepReason) {
    mKeyguardViewMediator.onStartedGoingToSleep(...);
}
```

##### 5.2.KeyguardViewMediator.onStartedGoingToSleep()

- 这里主要会调用notifyStartedGoingToSleep()方法，然后通过Handler发送值为NOTIFY_STARTED_GOING_TO_SLEEP的msg
- Handler会调用handleNotifyStartedGoingToSleep来调用StatusBarKeyguardViewManager的onStartedGoingToSleep()方法。

```java
public void onStartedGoingToSleep(@WindowManagerPolicyConstants.OffReason int offReason) {
    notifyStartedGoingToSleep();
}
private void notifyStartedGoingToSleep() {
    mHandler.sendEmptyMessage(NOTIFY_STARTED_GOING_TO_SLEEP);
}
private Handler mHandler = new Handler(Looper.myLooper(), null, true /*async*/) {
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case NOTIFY_STARTED_GOING_TO_SLEEP:
                handleNotifyStartedGoingToSleep();
                break;
        }
    }
}
private void handleNotifyStartedGoingToSleep() {
    mKeyguardViewControllerLazy.get().onStartedGoingToSleep();
}
```

##### 5.3.StatusBarKeyguardViewManager.onStartedGoingToSleep()

- 这里会禁用动画

```java
@Override
public void onStartedGoingToSleep() {
    mStatusBar.getNotificationShadeWindowView().getWindowInsetsController()
        .setAnimationsDisabled(true);
}
```

##### 5.4.KeyguardViewMediator.onFinishedGoingToSleep()

- 在完成onStartedGoingToSleep后，继续执行onFinishedGoingToSleep生命周期
- 之后还会调用reset()方法绘制Keyguard以及Bouncer的绘制。

```java
public void onFinishedGoingToSleep(...) {
    notifyFinishedGoingToSleep();
    maybeHandlePendingLock();
}
//StatusBarKeyguardViewManager.java
public void onFinishedGoingToSleep() {
    mBouncer.onScreenTurnedOff();
}
public void reset(boolean hideBouncerWhenShowing) {
    showBouncerOrKeyguard(hideBouncerWhenShowing);
}
protected void showBouncerOrKeyguard(boolean hideBouncerWhenShowing) {
    mStatusBar.showKeyguard();
    mBouncer.prepare();
}
```



### StatusBar流程分析

#### 1.statusBar简介

##### 1.1.作用范围

statusBar主要包括

#### 2.开机与初始化

##### 2.1.StatusBar.start()

```java
public void start() {
    ...;
    mWindowManager = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);
    updateDisplaySize();

    mVibrateOnOpening = mContext.getResources().getBoolean(
            R.bool.config_vibrateOnIconAnimation);

    // start old BaseStatusBar.start().
    mWindowManagerService = WindowManagerGlobal.getWindowManagerService();
    mDevicePolicyManager = (DevicePolicyManager) mContext.getSystemService(
            Context.DEVICE_POLICY_SERVICE);

    mAccessibilityManager = (AccessibilityManager)
            mContext.getSystemService(Context.ACCESSIBILITY_SERVICE);

    mKeyguardUpdateMonitor.setKeyguardBypassController(mKeyguardBypassController);
    mBarService = IStatusBarService.Stub.asInterface(
            ServiceManager.getService(Context.STATUS_BAR_SERVICE));

    mKeyguardManager = (KeyguardManager) mContext.getSystemService(Context.KEYGUARD_SERVICE);
    mWallpaperSupported =
            mContext.getSystemService(WallpaperManager.class).isWallpaperSupported();

    // Connect in to the status bar manager service
    mCommandQueue.addCallback(this);

    // Listen for demo mode changes
    mDemoModeController.addCallback(this);

    RegisterStatusBarResult result = null;
    try {
        result = mBarService.registerStatusBar(mCommandQueue);
    } catch (RemoteException ex) {
        ex.rethrowFromSystemServer();
    }

    createAndAddWindows(result);

    if (mWallpaperSupported) {
        // Make sure we always have the most current wallpaper info.
        IntentFilter wallpaperChangedFilter = new IntentFilter(Intent.ACTION_WALLPAPER_CHANGED);
        mBroadcastDispatcher.registerReceiver(mWallpaperChangedReceiver, wallpaperChangedFilter,
                null /* handler */, UserHandle.ALL);
        mWallpaperChangedReceiver.onReceive(mContext, null);
    } else if (DEBUG) {
        Log.v(TAG, "start(): no wallpaper service ");
    }

    // Set up the initial notification state. This needs to happen before CommandQueue.disable()
    setUpPresenter();

    if (containsType(result.mTransientBarTypes, ITYPE_STATUS_BAR)) {
        showTransientUnchecked();
    }
    onSystemBarAttributesChanged(mDisplayId, result.mAppearance, result.mAppearanceRegions,
            result.mNavbarColorManagedByIme, result.mBehavior, result.mAppFullscreen);

    // StatusBarManagerService has a back up of IME token and it's restored here.
    setImeWindowStatus(mDisplayId, result.mImeToken, result.mImeWindowVis,
            result.mImeBackDisposition, result.mShowImeSwitcher);

    // Set up the initial icon state
    int numIcons = result.mIcons.size();
    for (int i = 0; i < numIcons; i++) {
        mCommandQueue.setIcon(result.mIcons.keyAt(i), result.mIcons.valueAt(i));
    }


    if (DEBUG) {
        Log.d(TAG, String.format(
                "init: icons=%d disabled=0x%08x lights=0x%08x imeButton=0x%08x",
                numIcons,
                result.mDisabledFlags1,
                result.mAppearance,
                result.mImeWindowVis));
    }

    IntentFilter internalFilter = new IntentFilter();
    internalFilter.addAction(BANNER_ACTION_CANCEL);
    internalFilter.addAction(BANNER_ACTION_SETUP);
    mContext.registerReceiver(mBannerActionBroadcastReceiver, internalFilter, PERMISSION_SELF,
            null);

    if (mWallpaperSupported) {
        IWallpaperManager wallpaperManager = IWallpaperManager.Stub.asInterface(
                ServiceManager.getService(Context.WALLPAPER_SERVICE));
        try {
            wallpaperManager.setInAmbientMode(false /* ambientMode */, 0L /* duration */);
        } catch (RemoteException e) {
            // Just pass, nothing critical.
        }
    }

    // end old BaseStatusBar.start().

    // Lastly, call to the icon policy to install/update all the icons.
    mIconPolicy.init();

    mKeyguardStateController.addCallback(this);
    startKeyguard();

    mKeyguardUpdateMonitor.registerCallback(mUpdateCallback);
    mDozeServiceHost.initialize(
            this,
            mStatusBarKeyguardViewManager,
            mNotificationShadeWindowViewController,
            mNotificationPanelViewController,
            mAmbientIndicationContainer);
    mDozeParameters.addCallback(this::updateLightRevealScrimVisibility);

    mConfigurationController.addCallback(this);

    mBatteryController.observe(mLifecycle, this);
    mLifecycle.setCurrentState(RESUMED);

    // set the initial view visibility
    int disabledFlags1 = result.mDisabledFlags1;
    int disabledFlags2 = result.mDisabledFlags2;
    mInitController.addPostInitTask(
            () -> setUpDisableFlags(disabledFlags1, disabledFlags2));

    mFalsingManager.addFalsingBeliefListener(mFalsingBeliefListener);

    mPluginManager.addPluginListener(
            new PluginListener<OverlayPlugin>() {
                private ArraySet<OverlayPlugin> mOverlays = new ArraySet<>();

                @Override
                public void onPluginConnected(OverlayPlugin plugin, Context pluginContext) {
                    mMainThreadHandler.post(
                            () -> plugin.setup(getNotificationShadeWindowView(),
                                    getNavigationBarView(),
                                    new Callback(plugin), mDozeParameters));
                }

                @Override
                public void onPluginDisconnected(OverlayPlugin plugin) {
                    mMainThreadHandler.post(() -> {
                        mOverlays.remove(plugin);
                        mNotificationShadeWindowController
                                .setForcePluginOpen(mOverlays.size() != 0, this);
                    });
                }

                class Callback implements OverlayPlugin.Callback {
                    private final OverlayPlugin mPlugin;

                    Callback(OverlayPlugin plugin) {
                        mPlugin = plugin;
                    }

                    @Override
                    public void onHoldStatusBarOpenChange() {
                        if (mPlugin.holdStatusBarOpen()) {
                            mOverlays.add(mPlugin);
                        } else {
                            mOverlays.remove(mPlugin);
                        }
                        mMainThreadHandler.post(() -> {
                            mNotificationShadeWindowController
                                    .setStateListener(b -> mOverlays.forEach(
                                            o -> o.setCollapseDesired(b)));
                            mNotificationShadeWindowController
                                    .setForcePluginOpen(mOverlays.size() != 0, this);
                        });
                    }
                }
            }, OverlayPlugin.class, true /* Allow multiple plugins */);
}
```
