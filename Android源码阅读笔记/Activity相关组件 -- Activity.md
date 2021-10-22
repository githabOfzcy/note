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

​	各种启动方式

#### ...