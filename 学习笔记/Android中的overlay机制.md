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

