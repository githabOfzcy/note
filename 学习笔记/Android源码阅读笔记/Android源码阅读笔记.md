# 根目录结构

- art：ART运行环境
- bionic：系统C库
- bootable：启动引导相关代码
- build：系统编译规则文件及generic等基础开发包配置
- cts：Android兼容性测试套件标准
- dalvik：dalvik虚拟机（Android平台的Java虚拟机）
- developers：开发者目录（开发者的一些工具，sdk啥的）
- development：开发的一些例程
- devices：设备参数的相关配置
- doce：参考文档
- external：andriod使用的一些开源模组
- frameworks：核心框架
- hardware：硬件抽象层，HAL层代码
- kernel：内核层代码
- out：编译完成后的代码输出
- sdk：sdk和模拟器
- packages：应用程序包
- system：底层文件系统库 应用和组件
- vendor：厂家定制内容



## Frameworks

### base

- api：android应用框架层声明类、属性、资源
- cmds：android系统的一些命令
- core：框架组件
- data：android系统的资源
- docs：android项目说明
- drm：权限管理、数字内容加密等的实现
- graphics：图像渲染模块
- keystore：秘钥库
- libs：库信息
- location：位置信息
- media：手机媒体管理
- native：本地方法
- nfc-extras：近场通信（nfc功能）
- obex：蓝牙
- opengl：2D和3D图形的绘制
- packages：框架层的实现
- proto：
- rs：
- samples：例程
- sax：解析xml文件
- services：基于手机的服务
- telephone：手机操作
- tools：自带工具
- wifi：WiFi模块



# 源码阅读

## ServiceManager和Zytoge

serviceManager的功能是提供binder通讯服务

Zygote的功能是创建java进程

在ServiceManager启动后启动Zytoge，Zytoge会创建Systemserver线程。



## SystemServer -- Java进程启动的源头

- SystemServer是系统服务，它是系统启动时调用的程序，由Native层调用并启动；SystemServer启动后创建并初始化其他manager和server。

##### main()

- SystemServer中的main()方法只有一条：

  ```Java
  	public static void main(String[] args) {
          //表明SystemServer是运行在一个独立的线程中
          new SystemServer().run();
      }
  ```

##### run()方法

1. 设置启动的初始信息：启动时间、关机延迟、系统时间、时区、系统语言、非单项服务、启用安全设置、设置默认值、禁用数据库等等

   ```Java
   if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
       Slog.w(TAG, "System clock is before 1970; setting to 1970.");
       SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
       //设置系统时间最早为1970，因为早于这个时间将有很多API无法使用
   }
   ```

2. 进入Systemserver：

   - 重新设置时间

   - 保持Systemserver始终运行、定义指纹生成、指定用户

     ```Java
      VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);
      Build.ensureFingerprintProperty();
     ```

   - 设置Bundle、堆栈、Binder属性，以保证SystemServer可以正常运行

   - 加载本地库

3. 检测上次shutdown是是否失败，是的话重新启动

   ```java
   performPendingShutdown();
   private void performPendingShutdown() {
       ...;
       if (shutdownAction != null && shutdownAction.length() > 0) {
           ...;
           Runnable runnable = new Runnable() {
               @Override
               public void run() {
                   synchronized (this) {
                       ShutdownThread.rebootOrShutdown(null, reboot, reason);
                   }
               }
           };
           ...;
       }
   }
   ```

4. 创建SystemContext并设置SystemUiContext的主题

   ```Java
   createSystemContext();
   private void createSystemContext() {
       ActivityThread activityThread = ActivityThread.systemMain();
       mSystemContext = activityThread.getSystemContext();
       mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);
   
       final Context systemUiContext = activityThread.getSystemUiContext();
       systemUiContext.setTheme(DEFAULT_SYSTEM_THEME);
   }
   ```

5. 创建并初始化SystemServerManager，把SystemServerManager加入到本地服务

   ```Java
   mSystemServiceManager = new SystemServiceManager(mSystemContext);
   ```

6. start system servicce，这是SystemServer的核心功能

   ```Java
   startBootstrapServices(){ //1、启动引导服务，包括看门狗、各种ManagerService、启动各种初始service等等。
       ...
       ActivityTaskManagerService atm = mSystemServiceManager.startService(
           ActivityTaskManagerService.Lifecycle.class).getService();
       mActivityManagerService = ActivityManagerService.Lifecycle.startService(mSystemServiceManager, atm);
       mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
       mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
           mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
       mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);
       ...
   }
   
   startCoreServices(){ //2、启动各种核心服务：Battery、Web、Binder、RollbackManager、BugreportManager等
   	mWebViewUpdateService = mSystemServiceManager.startService(WebViewUpdateService.class);
   }
   
   startOtherServices(){  //3、启动一些其他必要的功能模块服务：通信、账户、SystemProviders、输入、windows、蓝牙、网络、位置、媒体等等
   	networkManagement = NetworkManagementService.create(context);
       wm = WindowManagerService.main(context, inputManager, !mFirstBoot, mOnlyCore,
                       new PhoneWindowManager(), mActivityManagerService.mActivityTaskManager);
       inputManager = new InputManagerService(context);
   }
   ```

7. 在startOtherServices()方法的最后，会调用mActivityManagerService.systemReady()方法

   ```Java
   mActivityManagerService.systemReady(() -> {...})
   //包含webUI、车载服务、SystemUI、飞行模式、网络、IP、通信、防火墙、地址、输入、媒体等模块的准备和启动
   ```

8. 关闭StrictMode

9. 长时间未启动成功处理



## SystemServiceManager -- 所有系统服务的管理者

- SystemServiceManager（SSM）是用来管理系统运行所要用到的具体Service的（这些Service都继承于SystemService）。
- SSM由SystemServer创建，可以启动系统级服务并控制SystemService的生命周期

1. 三种方式启动系统级服务

   ```Java
   //第一种，通过类名启动
   public SystemService startService(String className) {
       ...;
       serviceClass = (Class<SystemService>)Class.forName(className);
       ...;
       return startService(serviceClass);
   }
   //第二种，通过serviceClass启动
   public <T extends SystemService> T startService(Class<T> serviceClass) {
       ...;
       Constructor<T> constructor = serviceClass.getConstructor(Context.class);
       service = constructor.newInstance(mContext);
       ...;
       startService(service);
   }
   //第三种，通过SystemService启动
   public void startService(@NonNull final SystemService service) {
       mServices.add(service); //所有系统级服务都存储在mServices链表中，SSM通过管理这个链表来实现对各个SystemService的管理
       ...;
       service.onStart();
       ...;
   }
   ```

   <!--不难发现前两种启动方式最终都调用了第三种启动方式，service.onStart()中具体有哪些事情详见  **SystemService -- 所有系统级服务的父类**  章节。-->

2. 启动检测

3. 和用户相关的方法

   ```Java
   public void startUser(final int userHandle) {//启动用户
       ...
       final SystemService service = mServices.get(i);
       service.onStartUser(userHandle);
   }
   public void unlockUser(final int userHandle){...}
   public void switchUser(final int userHandle){...}
   public void stopUser(final int userHandle) {...}
   public void cleanupUser(final int userHandle) {...}
   //这几个方法与startUser类似，都是通过调用SystemService中相应的方法来实现
   ```

   

### SystemService -- 所有系统服务的父类

1. 系统服务的构造方法：

   ```Java
   public SystemService(Context context) {//需要传递一个Context变量来构造，SystemService又为这个context提供了get()方法
       mContext = context;
   }
   ```

2. 提供Context和UiContext：

   ```Java
   public final Context getContext() {
       return mContext;
   }
   public final Context getUiContext() {//UiContext是从当前活动所在线程获取的
       return ActivityThread.currentActivityThread().getSystemUiContext();
   }
   ```

3. 定义了两种服务类型：

   ```Java
   protected final void publishBinderService(...){//第一种：BinderService，这种服务可以被其他服务或者应用访问
       ServiceManager.addService(...);//开启BinderService的方式就是把服务加到ServiceManager中以获取binder通讯的能力
   }
   protected final void publishLocalService(...) {//第二种：LocalService，这种服务不可以被其他服务或者应用访问
       LocalServices.addService(...);//加入到LocalService中
   }
   ```

   LocalService的作用和ServiceManager类似，用来保存那些不需要进行binder通信的服务



#### 常见的SystemService

- 继承自SystemService的类很多很多，使用`grep -r 'extends SystemService'`在framework中一查便知

  <!--这部分内容会随时补充-->

1. UsageStatsService：使用统计
2. SoundTriggerService：声音感应
3. VoiceInteractionManagerService：语音交互管理
4. PrintManagerService：打印管理
5. CompanionDeviceManagerService：配套设备管理
6. RestrictionsManagerService：权限管理



# Activity相关组件

## 整体结构

- 个人理解：可以吧所有和Activity相关的类分成这么几种：

  1. ##### 前端相关的，这部分类位于frameworks/base/core/java/android/app下

     - Activity：Activity生命周期、事件响应等的具体实现

     - ActionBar：负责标题栏的相关实现，比如LOGO、ICON等等的set和get

     - 其他前端相关的，比如FragmentTransition、ActivityView等

  2. ##### 应用程序的后端相关，这部分类位也于frameworks/base/core/java/android/app下

     - Application：应用程序的基类，可以在AndroidManifest.xml中通过android:name属性来完成自己的实现
     - ActivityManager：提供Activity、service的状态信息并与之交互

     - ActivityThread：应用程序的主线程，APP的入口，根据ActivityManager的请求调度并执行Activity、broadcast等
     - ActivityTaskManager：提供活动容器(Task、stack等)的信息并与之交互
     - 其他后端相关的，比如BroadcastOptions、DownloadManager等

  3. ##### 系统相关实现，这部分位于frameworks/base/services/core/java/com/android/server/wm下

     - ActivityTaskManagerService：统筹Activity的启动和调度
  - ActivityStarter：用于启动Activity
     - ActivityRecord：是Activity在system_server进程中的镜像，Activity实例与ActivityRecord实例一一对应，ActivityRecord包含有对应Activity的全部信息
     - TaskRecord：任务栈，里面包含多个ActivityRecord，可以依照Activity的启动方式进行出栈、入栈等操作
     - ActivityStack：是TaskRecord的一个分组，用于管理TaskRecord列表,列表中的TaskRecord可以重排顺序
       - 一般情况下，一个应用程序对应一个ActivityStack，这个ActivityStack中也仅有一个TaskRecord。
     - ActivityStackSupervisor：是ActivityStack的管理者，内部管理了mHomeStack、mFocusedStack和mLastFocusedStack三个ActivityStack。

  4. ##### 其他相关支持，这部分位于frameworks/base/services/core/java/com/android/server/am下

     - ActivityServices：在activity中用到的服务的管理
     - ActivityManagerConstants：设置ActivityManager的常量
     - ActivityManagerService：四大组件的创建和管理，其中和Activity相关的方法都放在了ActivityTaskManagerService中

- 接下来就对这些类进行逐一进行探究：



## Activity

### 整体作用

- 实现了在应用开发时某个Activity中的所有方法以及底层回调：
  1. 各个生命周期
  2. 图片模式
  3. Fragment
  4. 控件
  5. ActionBar
  6. 按键事件处理
  7. Windows相关
  8. Menu
  9. Dialog
  10. 权限
  11. Activity的启动

### 代码分析

#### 初始化阶段

1. 定义Activity创建、关闭和Task关闭情况的常量
2.  相关Manager、Handler等的创建
3. Intent、Application、WindowsManager、Windows等的get()、set()方法

#### 各个生命周期的回调

```Java
private void dispatchActivityPreCreated(@Nullable Bundle savedInstanceState) {
    //在预备创建阶段的状态回调，其他生命周期都是用同样的方法进行回调的
    //Created()、PostCreated()、PreStarted()、Started()、PostStarted()、PreResumed()
    //Resumed()、PostResumed()、PrePaused()、Paused()、PostPaused()、PreStopped()
    //Stopped()、PostStopped()、PreSaveInstanceState()、SaveInstanceState()、PostSaveInstanceState()
    //PreDestroyed()、Destroyed()、PostDestroyed()
    //以上阶段都会进行回调以在Application中调用底层实现
    Object[] callbacks = collectActivityLifecycleCallbacks();
    ((Application.ActivityLifecycleCallbacks) callbacks[i]).onActivityPreCreated(this,savedInstanceState);
}

protected void onCreate(@Nullable Bundle savedInstanceState) {
    ...
    dispatchActivityCreated(savedInstanceState);
    ...
}
```

#### 各个生命周期

##### onCreate

```Java
protected void onCreate(@Nullable Bundle savedInstanceState) {//关键方法罗列如下
    //加载碎片
    mFragments.restoreLoaderNonConfig(mLastNonConfigurationInstances.loaders);
    //设置ActionBar的默认显示
    mActionBar.setDefaultDisplayHomeAsUpEnabled(true);
    //创建自动填充
    getAutofillManager().onCreate(savedInstanceState);
    //回调
    dispatchActivityCreated(savedInstanceState);
    //为Activity设置语音交互
    mVoiceInteractor.attachActivity(this);
}
```

##### onStart

```Java
protected void onStart() {
    //使加载的碎片执行onStart方法
    mFragments.doLoaderStart();
    //回调
    dispatchActivityStarted();
    //需要进行自动填充时进行自动填充
    if (mAutoFillResetNeeded) {
        getAutofillManager().onVisibleForAutofill();
    }
}
```

##### onResume

```Java
protected void onResume() {
    //通过层层调用最终实现了在onResume完成之前就显示Activity的视图
    mActivityTransitionState.onResume(this);
    //通知内容捕获器
    notifyContentCaptureManagerIfNeeded(CONTENT_CAPTURE_RESUME);
}
protected void onPostResume() {
    //调用Window的makeActivity方法，实现事件响应
    final Window win = getWindow();
    win.makeActive();
}
```

##### saveInstanceState、saveManagedDialogs

在Activity被kill之前存储它的状态

##### onPause

```Java
protected void onPause() {
    //回调
    dispatchActivityResumed();
    //在自动填充中取消这个Activity
    getAutofillManager().notifyViewExited(focus);
    //通知内容捕获器
    notifyContentCaptureManagerIfNeeded(CONTENT_CAPTURE_PAUSE);
}
```

##### onStop

```Java
protected void onStop() {
    //隐藏ActionBar
    mActionBar.setShowHideAnimationEnabled(false);
    //停止显示
    mActivityTransitionState.onStop();
    //回调
    dispatchActivityStopped();
    //自动填充不可见
    getAutofillManager().onInvisibleForAutofill();
}
```

##### onDestroy

```java
protected void onDestroy() {
    //关闭对话框
    md.mDialog.dismiss();
    //关闭光标
    c.mCursor.close();
    //关闭搜索框
    mSearchManager.stopSearch();
    //销毁ActionBar
    mActionBar.onDestroy();
    //回调
    dispatchActivityDestroyed();
    //通知内容捕获器
    notifyContentCaptureManagerIfNeeded(CONTENT_CAPTURE_STOP);
}
```

#### 图片模式PictureMode

```Java
public boolean enterPictureInPictureMode(@NonNull PictureInPictureParams params) {
    //输入图片
    return ActivityTaskManager.getService().enterPictureInPictureMode(mToken, params);
}
public void setPictureInPictureParams(@NonNull PictureInPictureParams params) {
    //更新图片
    ActivityTaskManager.getService().setPictureInPictureParams(mToken, params);
}
```

#### Fragment相关

```Java
public FragmentManager getFragmentManager() {
    return mFragments.getFragmentManager();
}
public void onAttachFragment(Fragment fragment) {
}
```

#### 控件相关

```java
public <T extends View> T findViewById(@IdRes int id) {
    //获取静态控件
    return getWindow().findViewById(id);
}
public void setContentView(View view) {
    //设置动态控件
    getWindow().setContentView(layoutResID);
}
public void addContentView(View view, ViewGroup.LayoutParams params) {
    //添加动态控件
    getWindow().addContentView(view, params);
}
```

#### ActionBar相关

```java
public ActionBar getActionBar() {
    initWindowDecorActionBar();
    return mActionBar;
}
public void setActionBar(@Nullable Toolbar toolbar) {
    final ActionBar ab = getActionBar();
    //销毁已存在的ActionBar
    ab.onDestroy();
    //将传入的参数Toolbar转换成ToolbarActionBar
    final ToolbarActionBar tbab = new ToolbarActionBar(toolbar, getTitle(), this);
    mActionBar = tbab;
    //设置ActionBar
    mWindow.setCallback(tbab.getWrappedWindowCallback());
}
```

#### 按键处理

```java
public final void setDefaultKeyMode(@DefaultKeyMode int mode) {
    //设置默认按键处理模式，有五种不同模式，不细说了
}
public boolean onKeyDown(int keyCode, KeyEvent event)  {
    //按下某个键时的处理，可以重写此方法。例如实现点击关闭Activity时询问是否关闭等等
    ...;
}
public boolean onKeyLongPress(int keyCode, KeyEvent event) {
    //长按事件，如果需要定义可以重写此方法
    return false;
}
public void onBackPressed() {
    //按下返回键时的处理事件，默认是弹出当前Activity；可以进行重写
}
public boolean onKeyShortcut(int keyCode, KeyEvent event) {
    //重写设置快捷键
}
public boolean onTouchEvent(MotionEvent event) {
    //重写设置没有被任何控件响应的点击事件
}
...
```

#### 和Windows相关

```java
//这些方法都是需要响应可以进行重写
public void onWindowFocusChanged(boolean hasFocus) {}
public void onAttachedToWindow() {}
public void onDetachedFromWindow() {}
public boolean hasWindowFocus() {
	...;
}
```

#### 各种点击事件的回调

```java
public boolean dispatchKeyShortcutEvent(KeyEvent event) {
    onUserInteraction();
    if (getWindow().superDispatchKeyShortcutEvent(event)) {//在这里回调
        return true;
    }
    return onKeyShortcut(event.getKeyCode(), event);
}
...
```

#### 三种基本类型Menu

```java
OptionsMenu：（选项菜单）
ContextMenu：（上下文菜单）
PopupMenu：（弹出式菜单）
//包含这三种菜单的创建、事件响应、销毁等方法
```

#### Dialog对话框

#### requestPermissions

#### startActivity相关

​	各种启动模式

#### ...



## ActivityManager

### 整体作用

这个类主要用来调试，可以通过其中的各个内部类来获取各种信息。



## ActivityTaskManager

### 整体作用

ActivityTaskManager通过与操作IActivityTaskManager的实例来与底层服务ActivityTaskManagerService实现通信

### 代码分析

#### 常量设定

#### 获取IActivityTaskManager的实现类对像

```java
public static IActivityTaskManager getService() {
    //提供IActivityTaskManager的实现类对象
    return IActivityTaskManagerSingleton.get();
}

private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
        new Singleton<IActivityTaskManager>() {
    		//获取IActivityTaskManager的实现类对象
            @Override
            protected IActivityTaskManager create() {
                //IActivityTaskManager继承与于IInterface，IInterface实现了Binder通信
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                return IActivityTaskManager.Stub.asInterface(b);
            }
		};
```

#### 设置堆栈模式

```java
public void setTaskWindowingMode(int taskId, int windowingMode, boolean toTop){
    //先获取IActivityTaskManager的实现类对象，再调用它的相关方法
    getService().setTaskWindowingMode(taskId, windowingMode, toTop);
}
```

#### 设置主屏幕任务模式

```java
public void setTaskWindowingModeSplitScreenPrimary(...) throws SecurityException {
    getService().setTaskWindowingModeSplitScreenPrimary(...);
}
```

#### 删除任务

```java
public void removeStacksInWindowingModes(int[] windowingModes) throws SecurityException {
    //删除某种窗口模式下的所有任务
    getService().removeStacksInWindowingModes(windowingModes);
}
public void removeStacksWithActivityTypes(int[] activityTypes) throws SecurityException {
    //删除包含某种类型的Activity的任务
    getService().removeStacksWithActivityTypes(activityTypes);
}
public void removeAllVisibleRecentTasks() {
    //删除所有可见的任务
    getService().removeAllVisibleRecentTasks();
}
```

#### 移动Activity

```java
ublic boolean moveTopActivityToPinnedStack(int stackId, Rect bounds) {
    //把某个堆栈顶部的Activity移动到固定堆栈
    return getService().moveTopActivityToPinnedStack(stackId, bounds);
}
```

#### 系统锁定任务模式

```java
 public void startSystemLockTaskMode(int taskId) {
     //开启系统锁定任务模式
     getService().startSystemLockTaskMode(taskId);
 }
public void stopSystemLockTaskMode() {
    //停止系统锁定任务模式
    getService().stopSystemLockTaskMode();
}
```





## ActivityTaskManagerService -- Activity的管理者

### 整体作用

- ATM是用来对Activity进行管理和调度的，通过调用ActivityRecord和ActivityStack中的具体实现来啊完成对Activity的各种设置

### ATM的启动

- 在Systemserver中调用ActivityTaskManagerService.Lifecycle.onStart()来启动的ATM

  ```Java
  public static final class Lifecycle extends SystemService {
      ...;
      mService = new ActivityTaskManagerService(context);
  	...;
      @Override
      public void onStart() {
          //ATM是一项需要进程间通信的服务，把它归属到BinderService类型中
          publishBinderService(Context.ACTIVITY_TASK_SERVICE, mService);
          //Lifecycle.onStart()方法又调用了ATM的start()方法，把ActivityTaskManagerInternal添加到本地服务中
          mService.start();
      }
  }
  ```

### ATM的主要方法

#### 内部类FontScaleSettingObserver：更新字体比例

#### 检查设置

```java
public void retrieveSettings(ContentResolver resolver) {
    //一些对各种格式是否支持的判断标志
    ...;
    //设置布局方向
    configuration.setLayoutDirection(configuration.locale);
    //如果对这些格式支持，那么调用WindowsManagerService来进行相应设置
    ...;
    //加载资源
    final Resources res = mContext.getResources();
    mThumbnailWidth = res.getDimensionPixelSize(...);
    mThumbnailHeight = res.getDimensionPixelSize(...);
}
```

#### 初始化

```java
public void initialize(...) {
    //Looper、Handler等的初始化
    mH = new H(looper);
    mUiHandler = new UiHandler();
    ...;
}
//主要属性的get、set方法
public void setWindowManager(WindowManagerService wm) {
    mWindowManager = wm;
    mLockTaskController.setWindowManager(wm);
    mStackSupervisor.setWindowManager(wm);
    mRootActivityContainer.setWindowManager(wm);
}
UserManagerService getUserManager() {
    IBinder b = ServiceManager.getService(Context.USER_SERVICE);
    mUserManager = (UserManagerService) IUserManager.Stub.asInterface(b);
    return mUserManager;
}
//windows、权限、UserManeger、RecentTask、
```

#### 内部类Lifecycle

```java
public static final class Lifecycle extends SystemService {
    //在Lifecycle的构造器中创建了ActivityTaskManagerService的对象
    public Lifecycle(Context context) {
        super(context);
        mService = new ActivityTaskManagerService(context);
    }
    //在这里绑定Service
    public void onStart() {
        publishBinderService(Context.ACTIVITY_TASK_SERVICE, mService);
        mService.start();
    }
    //解锁用户
    public void onUnlockUser(int userId) {...;}
    //清除用户
    public void onCleanupUser(int userId) {...;}
    //提供刚才创建的ActivityTaskManagerService的对象
    public ActivityTaskManagerService getService() {
        return mService;
    }
}
```

#### startActivity

- startActivity()

  ```Java
  @Override
  public final int startActivity(...) {//用来启动Activity
      return startActivityAsUser(...);
  }
  int startActivityAsUser(...) {
      return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
          ...;//在这里最终调用了ActivityStarter中的execute()方法来启动Activity
  }
  ```

- startActivity()的其他相关方法

  ```Java
  public boolean startNextMatchingActivity(...){//启动下一个匹配的Activity，用于隐式Interent启动
      List<ResolveInfo> resolves = AppGlobals.getPackageManager().queryIntentActivities(...)
          .getList();//将所有匹配的结果封装成一个链表
      final int N = resolves.size();
      for (int i=0; i<N; i++) {//取到下一个匹配Activity的activityInfo
          ResolveInfo rInfo = resolves.get(i);
          aInfo = resolves.get(i).activityInfo;
      }
      //用这个activityInfo设置intent，启动Activity
      intent.setComponent(new ComponentName(aInfo.applicationInfo.packageName, aInfo.name));
      final int res = getActivityStartController().obtainStarter(intent, "startNextMatchingActivity")
  } 
  
  public final WaitResult startActivityAndWait(...){//启动Activity并等待，用于线程间的同步
       getActivityStartController().obtainStarter(intent, "startActivityAndWait")
           .setWaitResult(res);
  }
  
  public final int startActivityWithConfig(...){//使用配置启动活动
      return getActivityStartController().obtainStarter(intent, "startActivityWithConfig")
          .setGlobalConfiguration(config);
  }
  
  public final int startActivityAsCaller(...){}//作为调用方启动Activity；
  //用这种方式启动的Activity将像在自己的应用程序中一样可以进行任何操作，因此只有这两种情况才能使用
  //1、调用方是core framework中的应用且作为系统应用运行时
  //2、调用方获得一次性的permissionToken令牌时
  //获得permissionToken令牌可以通过调用requestStartActivityPermissionToken()申请
  
  public int startVoiceActivity(...){...}//启动语音Activity
  
  public int startAssistantActivity(...){...}//启动辅助Activity
  
  public void startRecentsActivity(...){...}//启动最近关闭的活动
  ```

#### finishActivity

- finishActivity()

  ```java
  public final boolean finishActivity(...) {//在某个Activity中调用finish()时，就会调用这个方法
      if (finishTask == Activity.FINISH_TASK_WITH_ACTIVITY || (finishWithRootActivity && r == rootR)) {
          //当钱Activity是根Activity时，将会删除整个程序栈并关闭应用
          res = mStackSupervisor.removeTaskByIdLocked(tr.taskId, false,finishWithRootActivity, "finish-activity");
      } else {
          //否则关闭当前Activity
          res = tr.getStack().requestFinishActivityLocked(token, resultCode, resultData, "app-request", true);
      }
  }
  
  public boolean finishActivityAffinity(IBinder token) {//关闭关联Activity
      //调用了ActivityStack中的finishActivityLocked方法，
      //最终调用了ActivityRecord.intent.addFlags()方法来关闭Activity。
      return task.getStack().finishActivityAffinityLocked(r);
  }
  ```


#### resumeActivity

- activityResumed()

  ```Java
  public final void activityResumed(IBinder token) {
      ActivityRecord.activityResumedLocked(token);
      //调用WM的notifyAppResumedFinished()方法，在个方法中会对当前Activity进行聚焦
      mWindowManager.notifyAppResumedFinished(token);
  }
  public final void activityTopResumedStateLost() {
      //调用ActivityStackSupervisor中的activityTopResumedStateLost方法
      //最终调用了ActivityRecord.setState方法
      mStackSupervisor.handleTopResumedStateReleased(false);
  }
  ```

#### pauseActivity

- activityPaused()

  ```Java
  public final void activityPaused(IBinder token) {
      ActivityStack stack = ActivityRecord.getStackLocked(token);
      //调用ActivityStack的activityPausedLocked()方法
      //最终调用了ActivityRecord.setState方法
      stack.activityPausedLocked(token, false);
  }
  ```

#### stopActivity

- activityStopped()

  ```Java
  public final void activityStopped(...) {
      //使一个Activity进入onStop状态，如果一个应用程序的所有Activity都成为onStop状态了，那么这个应用程序也将会被标记为onStop状态
      r.activityStoppedLocked(icicle, persistentState, description);
  }
  ```

#### destroyActivity

- activityDestroyed()

  ```Java
  public final void activityDestroyed(IBinder token) {
      ActivityStack stack = ActivityRecord.getStackLocked(token);
      if (stack != null) {
          stack.activityDestroyedLocked(token, "activityDestroyed");
      }
  }
  ```

#### 屏幕方向

```java
public void setRequestedOrientation(IBinder token, int requestedOrientation) {
    //调用了ActivityRecord中的setRequestedOrientation来设置屏幕方向
    ActivityRecord r = ActivityRecord.isInStackLocked(token);
    r.setRequestedOrientation(requestedOrientation);
}
@Override
public int getRequestedOrientation(IBinder token) {
    //获取屏幕方向
    return r.getOrientation();
}
```

#### 全屏显示（沉浸式体验）

```java
public void setImmersive(IBinder token, boolean immersive) {
    final ActivityRecord r = ActivityRecord.isInStackLocked(token);
    r.immersive = immersive;
}
public boolean isImmersive(IBinder token) {...}
public boolean isTopActivityImmersive(){...}
```

#### 跳转动画

```java
public void overridePendingTransition(...) {
    //传入的参数包括指明跳转对象的Binder，弹出动画、弹入动画的时间。
    ActivityRecord self = ActivityRecord.isInStackLocked(token);
    self.getDisplay().mDisplayContent.mAppTransition.overridePendingAppTransition(...);
}
```

#### 最上层Activity兼容模式

```java
public int getFrontActivityScreenCompatMode() {
	//获取最上层Activity
    final ActivityRecord r = getTopDisplayFocusedStack().topRunningActivityLocked();
    //获取兼容模式
    return mCompatModePackages.computeCompatModeLocked(r.info.applicationInfo);
}
public void setFrontActivityScreenCompatMode(int mode) {
    //获取最上层Activity
    final ActivityRecord r = getTopDisplayFocusedStack().topRunningActivityLocked();
    //设置兼容模式
    //兼容模式有四种：COMPAT_MODE_NEVER、COMPAT_MODE_ALWAYS、COMPAT_MODE_ENABLED、COMPAT_MODE_DISABLED
    ai = r.info.applicationInfo;
    mCompatModePackages.setPackageScreenCompatModeLocked(ai, mode);
    }
}
```

#### RecentTasks的相关方法

```java
public void setFocusedStack(int stackId) {...}//设置当前Stack
public void setFocusedTask(int taskId) {...}//设置当前Task
public boolean removeTask(int taskId) {...}//根据堆栈的ID删除堆栈
public void removeAllVisibleRecentTasks() {...}//删除所有可见的堆栈
public boolean shouldUpRecreateTask(IBinder token, String destAffinity) {...}//
public void moveTaskToFront(...) {//将Task移动到前台
    moveTaskToFrontLocked(...);
}
void moveTaskToFrontLocked(...) {
    //需要调用ActivityStackSupervisor中的方法
    mStackSupervisor.findTaskToMoveToFront(...);
    //获取当前最上层的Activity并显示
    final ActivityRecord topActivity = task.getTopActivity();
    topActivity.showStartingWindow(...);
}	
```







## ActivityManagerService -- 其他三大组件的管理者

- 继承自IActivityManager的Stub类，实现了Watchdog.Monitor接口和BatteryStatsImpl.BatteryCallback接口
- AMS组件用来管理Activity、Service、Broadcast、Provider的创建与调度。

1. 启动过程

   - 在SystemServer的startBootstrapServices()阶段，会创建并启动ASM

     ```Java
     startBootstrapServices(){
         ...
         //启动AMS之前，先启动了一个ActivityTaskManagerService atm；atm是用来管理Activiy的启动和调度的。
         //SystemServiceManager.startService()方法调用的其实是这个服务的onStart方法
         ActivityTaskManagerService atm = mSystemServiceManager.startService(
             ActivityTaskManagerService.Lifecycle.class).getService();
         //将atm作为参数，启动AMS
         mActivityManagerService = ActivityManagerService.Lifecycle.startService(
             mSystemServiceManager, atm);
         //设置AMS的SSM和Installer属性
         mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
         mActivityManagerService.setInstaller(installer);
         ...
     }
     ```

     

### Activity组件的启动过程

- 在SystemServer的startOtherServices()中会调用ActivityManagerService.systemReady()方法，systemReady()中会启动Launcher，Launcher是应用程序启动器，可以把它看做一个应用程序，这个应用程序用来显示系统中已经安装的应用程序。

1. 点击应用程序图标时，实际是点击了Launcher界面中的应用程序快捷方式；首先会调用Launcher中的startActivitySafely方法，





## Launcher -- 应用程序启动器

