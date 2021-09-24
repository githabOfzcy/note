# 线程池

##### 线程池的四种创建方法：

1. Executors.newCachedThreadPool() 创建一个可缓存的线程池；如果线程池长度超过需要则回收空闲线程，如果线程池长度不满足需要则创建新的线程
2. Executors.newFixedThreadPool() 创建一个定长的线程池
3. Executors.newSingleThreadExecutor() 创建一个单线程的线程池，用来保证所有任务按照指定顺序执行
4. Executors.newScheduledThreadPool() 创建一个定长的线程池，支持定时得以周期性的任务执行

##### 使用方法：

- 新建线程类并获取线程对象（显式或者隐式获取都可以）
- 新建线程池
- 向线程池中提交线程对象：`submit()`
- 线程池使用完毕后调用shutdown()方法结束线程池

##### 继承关系

1. Executor是一个顶层接口，在它里面只声明了一个方法execute(Runnable)，返回值为void，参数为Runnable类型

2. ExecutorService接口继承了Executor接口，并声明了一些方法：submit、invokeAll、invokeAny以及shutDown等；

3. 抽象类AbstractExecutorService实现了ExecutorService接口，基本实现了ExecutorService中声明的所有方法；

4. 然后ThreadPoolExecutor继承了类AbstractExecutorService，有几个非常重要的方法：

   `execute()、submit()、shutdown()、shutdownNow()`



# NIO

##### NIO的构成：

- Channel（通道）：用来建立程序与要读写的对象之间的联系
- Buffer（缓冲区）：用来临时存储读写的内容，通过Channel穿梭于程序和目标地址之间
- Selector（选择器）：单个线程使用Selector实现对多个Channel的管理

##### Channel：

1. FileChannel：连接文件
2. DatagrameChannle：通过UDP连接网络
3. SocketChannel：通过TCP连接网络
4. ServerSocketChannel：监听新的TCP连接，对每一个新建的连接都创建一个对应的SocketChannel

##### Buffer：

- 类型

  ByteBuffer、CharBuffer、DoubleBuffer、FloatBuffer、……

- 使用步骤

  1. 新建Buffer
  2. Buffer读数据：`mChannel.read(mByteBuffer)`
  3. 切换为写数据模式：`mByteBuffer.flip()`
  4. Buffer写数据：`mByteBuffer.get()`
  5. 切换为读数据模式：`mByteBuffer.clear()`或`mByteBuffer.compact()`

- 读写过程

  - capacity,position和limit

  - capacity是Buffer能储存数据的最大量，是不变的

    在Buffer读数据时：

    ​	position从0逐渐变大，position所在位置之前的内存空间存储着读取的数据

    ​	limit保持与capacity相等

    在Buffer写数据时：

    ​	首先将position的值赋给limit，position的值变为0

    ​	接下来position从0逐渐增大到limit，position所在位置之前的内存空间存储的数据被写出

- 其他方法

  rewimd()：重读

  mark()：标记position

  reset()：把position恢复到mark()的位置

  equals()：两个Buffer类型相同、Buffer中剩余的字符类型和个数相同