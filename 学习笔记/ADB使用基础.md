# 什么是ADB

- Android Debug Bridge



# 常见命令

### 基础使用

- 查看当前连接设备：`adb devices`
- 启动： `adb root`
- 重启： `adb reboot`
- 当前存在多个设备时：`adb -s 设备号 其他指令`
- 查看日志：`adb logcat`
- 查看手机分辨率： `adb shell wm size`

### 安装软件(非系统软件)

- 安装apk文件： `adb install xxx.apk`
- 覆盖安装： `adb install -r xxx.apk`
- 卸载APP： `adb uninstall packageName`
- 保留数据卸载： `adb uninstall -k packageName`

### 传递数据

- 向手机传递文件： `adb push fileName <path>`

  例：向手机SD卡中传递文件： `adb push xx.mp3 /sdcard/`

- 从手机下载文件： `adb pull /sdcard/xx.mp3`

  <!--下载的地址是当前路径-->

### 运行

- 查看手机端安装的所有包名： `adb shell pm list packages`

##### adb shell am

<!--am的意思是Activity Manager-->

- 启动Activity: `adb shell am start packageName/absoultActivityPath`

  例如：`adb shell am start com.zhy.aaa/com.zhy.aaa.MainActivity`

  <!--如果需要传参，使用 -e key Value-->

  `除了默认启动的Activity，想要启动其他Activity需要在AndroidManifest中添加android:exported="true"属性`

- 启动隐式intent：`adb shell am start -a "action" -c "category" -d "data_uri"`

- 发送广播： `adb shell am broadcast -a "broadcastactionfilter"`

- 启动服务： `adb shell am startservice "packageName/absoultServicePath"`

### 模拟操作

- 屏幕截图： `adb shell screencap /sdcard/screen.png`
- 录制视频： `adb shell screenrecord /sdcard/screen.mp4`

##### adb shell input

- 输入文字： `adb shell input text "hi"`

- 点击事件：`adb shell input tap 500 1450`

- 滑动时间： `adb shell input swipe 240 700 240 200 200` 

- 长按：`adb shell input swipe 240 700 240 700 1000`

- 实体按钮：`adb shell keyevent 25` 

  <!--25是keyevent中定义的一个事件常量，代表调低音量-->

### dumpsys输出

dumpsys用来查看某种类型的相关信息，例如adb shell dumpsys battery用来查询电池相关信息

- activity：AMS（ActivityManagerService）
- package：PMS（PackageManagerService）
- window：WMS（WindowManagerService）
- input：IMS（InputManagerServices）
- power：PMS（PowerManagerServices）
- procstats：（ProcessStatsServices）
- battery：（BatteryServices）
- alarm：（AlarmManagerServices）
- meminfo：（MenBinder）内存

例如：`adb shell dumpsys activity activities`

