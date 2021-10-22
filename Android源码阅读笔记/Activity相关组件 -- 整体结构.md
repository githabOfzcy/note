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