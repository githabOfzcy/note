自定义SystemService可以分为以下几步

### 1.自定义AIDL文件

- 在frameworks/base/core/java/android/service下新建文件夹myservice

- 在myservice下新建IMyService.aidl

  ```java
  package android.service.myservice;
  
  import android.os.Bundle;
  
  interface IMyService {
  //这个服务提供四则运算
      double add(double num1,double num2);
      double sub(double num1,double num2);
      double mul(double num1,double num2);
      double dev(double num1,double num2);
  
  }
  ```

- 在frameworks/base/Android.bp中添加编译新建的AIDL

  ```java
  java_defaults {
      name: "framework-defaults",
      installable: true,
      srcs: [
      	...
          "core/java/android/service/myservice/IMyService.aidl",
          ...
      ],
      ...
  }
  ```

- 在终端里编译一次，生成AIDL接口

### 2.编写MyService.java文件

- Myservice继承IMyService.Stub，并重写IMyService的抽象方法

  ```java
  private class MyService extends IMyService.Stub {
      @Override
      public double add(double num1,double num2)throws RemoteException {
          return num1 + num2;
      }
      @Override
      public double sub(double num1,double num2)throws RemoteException {
          return num1 - num2;
      };
      @Override
      public double mul(double num1,double num2)throws RemoteException {
          return num1 * num2;
      };
      @Override
      public double dev(double num1,double num2)throws RemoteException {
          if (num2 == 0){
              return 0;
          }
          return num1/num2;
      };
  }
  ```

### 3.添加服务

- 定义服务名称，在frameworks/base/core/java/android/content/Context.java中添加服务名称

  ```java
  @StringDef(suffix = { "_SERVICE" }, value = {
      ...;
      //myService
     	MY_ARITHMETIC_SERVICE,
      ...;
  }
  ...;
  /**
  * This is MY_ARITHMETIC_SERVICE
  *
  */
  public static final String MY_ARITHMETIC_SERVICE = "my_arithmetic_service";
  ...;
  ```

- 在frameworks/base/service/java/com/android/server/SystemServer.java中启动服务

  ```java
  private void startOtherServices() {
      ...;
      try{
          /**
      	* 这里添加MyArithmeticService
      	*/
          ...;
          traceBeginAndSlog("StartMyArithmeticService");
          ServiceManager.addService(Context.MY_ARITHMETIC_SERVICE, new MyService());
          traceEnd();
          ...;
      }
  }
  ```

### 4.创建MyService的管理类

- 该管理类将service 封装起来，方便调用者操作；添加在frameworks/base/core/java/android/app/MyServiceManager.java

- ```java
  package android.app;
  import android.content.Context;
  import android.service.myservice.IMyService;
  public class MyServiceManager {
      private IMyService mIMyService;
      public MyServiceManager(Context ctx, IMyService service){
          mIMyService = service;
      }
      public double add(double num1,double num2){
          try {
              return mIMyService.add(num1,num2);
          }catch (Exception e){
              e.printStackTrace();
          }
          return 0;
      }
      public double sub(double num1,double num2){
          try {
              return mIMyService.sub(num1,num2);
          }catch (Exception e){
              e.printStackTrace();
          }
          return 0;
      }
      public
      double mul(double num1,double num2){
          try {
              return mIMyService.mul(num1,num2);
          }catch (Exception e){
              e.printStackTrace();
          }
          return 0;
      }
      public
      double dev(double num1,double num2){
          try {
              return mIMyService.dev(num1,num2);
          }catch (Exception e){
              e.printStackTrace();
          }
          return 0;
      }
  }
  ```


### 5.注册服务

- 将服务注册在SystemServiceRegistry中，这样我们就可以通过getSystemService来获取

- frameworks/base/core/java/android/app/SystemServiceRegistry.java

- ```java
  final class SystemServiceRegistry {
      ...;
      static {
          ...;
          //myService
          registerService(Context.MY_SERVICE, MyServiceManager.class,
                  new CachedServiceFetcher<MyServiceManager>() {
                      @Override
                      public MyServiceManager createService(ContextImpl ctx) {
                          //通过服务名称获取其binder
                          IBinder b = ServiceManager.getService(Context.MY_SERVICE);
                          //将binder转为接口对象，传递给manager管理
                          IMyService service = IMyService.Stub.asInterface(b);
                          return new MyServiceManager(ctx,service);
                      }});
          ...;
      }
      ...;
  }
  ```

### 6.编译

- 先使用 `make update-api`命令更新API
- 再使用 `make`命令编译

