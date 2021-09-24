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



