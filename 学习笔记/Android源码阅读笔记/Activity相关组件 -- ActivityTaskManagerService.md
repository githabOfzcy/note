## ActivityTaskManagerService -- Activity的管理者

- ATM是用来对Activity进行管理和调度的

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

- 内部类FontScaleSettingObserver：更新字体比例

- 初始化：initialize()、onInitPowerManagement()等

#### Activity的基本生命周期

##### startActivity

- startActivity()

  ```Java
  @Override
  public final int startActivity(...) {//用来启动Activity
      return startActivityAsUser(...);
  }
  int startActivityAsUser(...) {
      return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
          ...;//在这里会用setXXX()来对Activity进行初始化设置
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

##### resumeActivity

- activityResumed()

  ```Java
  public final void activityResumed(IBinder token) {
      ActivityRecord.activityResumedLocked(token);
      mWindowManager.notifyAppResumedFinished(token);//调用WM的notifyAppResumedFinished()方法，在个方法中会对当前Activity进行聚焦
  }
  ```

##### pauseActivity

- activityPaused()

  ```Java
  public final void activityPaused(IBinder token) {
      ActivityStack stack = ActivityRecord.getStackLocked(token);
      stack.activityPausedLocked(token, false);//调用ActivityStack的activityPausedLocked()方法，会给Activity栈+1
  }
  ```

##### stopActivity

- activityStopped()

  ```Java
  public final void activityStopped(...) {
      //使一个Activity进入onStop状态，如果一个应用程序的所有Activity都成为onStop状态了，那么这个应用程序也将会被标记为onStop状态
      r.activityStoppedLocked(icicle, persistentState, description);
  }
  ```

##### destroyActivity

- activityDestroyed()

  ```Java
  public final void activityDestroyed(IBinder token) {
      ActivityStack stack = ActivityRecord.getStackLocked(token);
      if (stack != null) {
          stack.activityDestroyedLocked(token, "activityDestroyed");
      }
  }
  ```

##### finishActivity

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
  ```

- finishActivity()的其他相关方法

  ```Java
  public boolean finishActivityAffinity(IBinder token) {//关闭关联Activity
      return task.getStack().finishActivityAffinityLocked(r);
  }
  ```

