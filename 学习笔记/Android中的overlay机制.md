# Android中的overlay机制

## 什么是Overlay

- Android Overlay是一种资源替换机制，它能在不重新打包apk的情况下，实现资源文件的替换。
- Overlay又分为静态ovrelay(SRO)和运行时overlay(RRO)

## SRO

- Static Resource Overlay 静态资源替换

- 实现方式：

  1. 在device目录下找到对应设备文件夹下的overlay文件夹
  2. 在overlay文件夹下建立与目标资源相同路径的文件夹
  3. 在相同资源的文件夹下放入同名的文件
  4. 在.mk文件中配置 `PRODUCT_PACKAGE_OVERLAYS += device/.../overlay`

- 举例：要替换模拟器aosp_x86_64的快捷设置及导航栏的飞行模式图标

  1. 找到图标对应的资源文件

     `frameworks/base/core/res/res/drawable/ic_qs_airplane.xml`

  2. 找到模拟器对应的设备下的overlay文件夹

     `device/generic/goldfish/overlay`

  3. 在相同路径下放入覆盖后的资源文件

     `device/generic/goldfish/overlay/frameworks/base/core/res/res/drawable/ic_qs_airplane.xml`

- 原理：在通过AAPT打包生成APK时，通过-S命令增加了一个资源目录（overlay），在调用资源时会优先检索overlay下是否存在对应资源，存在则直接使用，不存在继续在其他资源目录下检索



## RRO

- Runtime Resource Overlay运行时资源替换

- 实现方式

  1. 知道要替换的资源的包名和资源名

  2. 建立overlay项目结构：

     ```java
     overlay
         |---res
         |    |---drawable
         |    |      |---ic_qs_airplane.xml
         |    |---values
         |---Android.mk
         |---AndroidManifest.xml
     ```

     整体来说比较简单，在res目录下放入同名（路径不需要相同）的资源文件即可

  3. 编写AndroidManifest.xml文件

     ```java
     <manifest xmlns:android="http://schemas.android.com/apk/res/android"   固定的
         package="com.android.iconairplane"         当前包名，随便写
         android:versionCode="1"
         android:versionName="1.0">
         <overlay android:targetPackage="com.android.theme.systemui"     overlay目标的包名
                 android:priority="1"/>                                  优先级（1-999）
     <application android:hasCode="false"/>         固定的
     </manifest>
     
     ```

  4. 编写Android.mk文件

     ```java
     LOCAL_PATH:= $(call my-dir)							当前路径
     include $(CLEAR_VARS)								清除之前的变量配置
     LOCAL_RRO_THEME := IconAirplane						编译生成文件的文件夹
     LOCAL_PRODUCT_MODULE := true						编译生成文件是否在product目录下
     LOCAL_SRC_FILES := $(call all-subdir-java-files)	编译生成文件是.apk形式
     LOCAL_RESOURCE_DIR := $(LOCAL_PATH)/res				资源文件路径
     LOCAL_PACKAGE_NAME := IconAirplaneOverlay			编译生成文件名称
     LOCAL_SDK_VERSION := current						编译使用的sdk
     include $(BUILD_RRO_PACKAGE)
     ```

  5. 模块编译，把生成的apk用adb install命令放进模拟器里。
