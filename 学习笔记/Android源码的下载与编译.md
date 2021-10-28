# 下载repo



# repo init

- 新建目录：`mkdir xxx`
- 建立本地repo仓库：`repo init`



# 下载初始化包

- 从同事电脑copy

  `scp Xxx@192.101.101.101:<path>/aosp-latest.tar .`

- 从清华的源下载

  `wget -c https://mirrors.tuna.tsinghua.edu.cn/aosp-monthly/aosp-latest.tar`



# 解压

- 解压初始化压缩包：`tar xf aosp-latest.tar`



# 同步源码

- 进入解压后的目录：`cd AOSP`
- 选择分支：`repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest -b android-10.0.0_r47`
- 同步：`repo sync --no-tags -cd -j4`



# 全编译

- 第一次编译必须选择全编译
- 初始化配置：`source build/envsetup.sh`
- 选择分支：`lunch`
- 编译：make



# 目录结构简介

1. 应用程序层：packages/apps

2. 应用程序框架层：framework

3. 运行库层：

   运行库：bionic

   运行时环境：libcore、dalvik

4. HAL层：hardware

5. Linux内核层：kernel



# framework目录

|-- base                        （基本内容）
|   |-- api                （都是xml文件，定义了java的api）
|   |-- awt                （AWT库）
|   |-- build                （空的）
|   |-- camera                （摄像头服务程序库）
|   |-- cmds                （重要命令：am、app_proce等）
|   |-- core                （核心库）
|   |-- data                （字体和声音等数据文件）
|   |-- docs                （文档）
|   |-- graphics        （图形相关）
|   |-- include                （头文件）
|   |-- keystore        （和数据签名证书相关）
|   |-- libs                （库）
|   |-- location        （地区库）
|   |-- media                （媒体相关库）
|   |-- obex                （蓝牙传输库）
|   |-- opengl                （2D-3D加速库）
|   |-- packages        （设置、TTS、VPN程序）
|   |-- sax                （XML解析器）
|   |-- services        （各种服务程序）
|   |-- telephony        （电话通讯管理）
|   |-- test-runner        （测试工具相关）
|   |-- tests                （各种测试）
|   |-- tools                （一些叫不上名的工具）
|   |-- vpn                （VPN）
|   |-- wifi                （无线网络）
|-- opt                        （可选部分）
|   |-- com.google.android                                （有个framework.jar）
|   |-- com.google.android.googlelogin                （有个client.jar）
|-- emoji                （standard message elements）
  -- policies                （Product policies are operating system directions aimed at specific uses）-- base                
        |-- mid        （MID设备）
          -- phone        （手机类设备，一般用这个）



# 模块编译

- 进入build目录：`cd bulid`

- 获得envsetup.sh文件执行权限：`chmod +x envsetup.sh`

- 执行envsetup.sh文件：`source ~/Code/aosp/build/envsetup.sh`

  <!--在执行envsetup.sh文件之后可以获得一些额外的命令，包括mmm-->

- 编译指定模块：`mmm <path>`或者`make moudleName`

- 直接覆盖（不需要卸载）：`adb install -r xxx.apk `

- 重新打包系统镜像：`make snod`



# 启动模拟器

```java
export PATH=$PATH:/work/android_src/out/host/linux-x86/bin/
export ANDROID_PRODUCT_OUT=/work/android_src/out/target/product/generic/
lunch
emulator
//启动模拟器时依然要使用sudo chown userName /dev/kvm
```



# 把源码导入AS

- 执行source命令：`. build/envsetup.sh`
- 选择分支：`lunch xx`
- 编译idegn模块：`mmm development/tools/idegen`
- 执行idegen文件：`development/tools/idegen/idegen.sh`
- 会生成android.iml和android.ipr文件，android.iml里面记录了项目包含的moudel、依赖关系等
- 需要用as查看调试哪一部分的源码，就把android.iml和android.ipr文件复制到哪一部分，然后用as打开android.ipr即可查看并调试。



# 查看UI

hierarchyviewer





```
aapt p -f -I [ANDROID_SDK_PATH]/android.jar \
    -S [OVERLAY_PATH]/res \
    -M [OVERLAY_PATH]/AndroidManifest.xml \
    -c [PRODUCT_AAPT_CONFIG] \
    -F [MODULE_NAME]-overlay.apk
```

