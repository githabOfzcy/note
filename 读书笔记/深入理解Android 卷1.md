# JNI

- ### 什么是JNI

  - Java Native Interface的缩写，翻译为“Java本地调用”

  - 可以做到以下两点

    1. Java程序中的函数可以调用Native语言写的函数

    2. Native程序中的函数可以调用Java层中的函数

       <!--Native一般指用c/c++编写的函数-->

    ```mermaid
    graph TD
    Java世界--->JNI层
    JNI层--->Java世界
    JNI层--->Native世界
    Native世界--->JNI层
    ```

- ### 实例：MediaScanner

  #### 1.加载JNI库

  - 在调用native函数前需要加载JNI库

  - 一般在类的static块中加载

  #### 2.native函数

  - 使用关键字native定义

  - ##### 注册JNI函数

    1. ###### 静态注册

       - 编写Java代码，编译生成.class文件

       - 使用工具程序Java.h，如：`javah -o output packagename.classname`，生成output.h的JNI层头文件

       - Java层的native_init函数和JNI库中的Java_android_media_MediaScanner_native_linit函数相对应。

         <!--JNI层中的函数为`包名_类名_方法名`，将名称中所有的"."换成"_"，如果Java层中的函数自带"_"，则JNI层转化为"_l"-->

    2. ###### 动态注册

       - 创建一个JNINativeMethod数组，成员是所有Native函数的一一对应关系

         <!--JNINativeMethod是用来记录Java native函数与JNI函数一一对应关系的结构，定义为：-->

         ```java
         typedef struct{
         "native_init",			//Java中的native函数的名字，不用携带包的路径
         "()V",							//签名信息(void *)
         Java_android_media_MediaScanner_native_init
          //JNI层对应函数的函数指针
         }
         ```

       - 注册JNINativeMethod数组，调用AndroidRuntime的registerNativeMethods函数

         ```java
         int register_android_media_MediaScanner(JNIEnv *env){
             //调用AndroidRuntime中的registerNativeMethods函数，第二个参数为Java中的类
             return AndroidRuntime::registerNativeMethods(env,"android/media/MediaScanner",
                                                          gMethods,NELEM(gMethods));
         }
         ```

       - 实现JNI_OnLoad方法，使这些注册被动态调用