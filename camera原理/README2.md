# 背景

# 一、相机子系统基本分层
相机子系统从上到下可以分为4层：
1. 应用层：以Java的Camera类为主，为相机应用提供操作相机的接口。
2. Framework层：以C++的CameraService类为主，是应用层和HAL层通信的桥梁。
3. HAL层：硬件抽象层，以CameraHardwareInterface.h为主，规定硬件层的虚拟方法，具体实现要看各个系统对抽虚拟方法的实现。
4. 驱动层：以V4L2驱动程序为主，直接与摄像头通信。
 
HAL层出现的意义在于向上提供了统一接口，针对不同硬件平台只需替换HAL的实现，不用更改上层的API的调用逻辑。层级之间是通过什么来通信的呢？
应用层与Framework层之间通过Binder进行通信，系统开机初始化的时候会开启一个CameraService守护进程，为上层应用提供Camera对应的功能接口。Framework与硬件抽象层之间通过回调函数的方式来传递数据。
抽象层位于用户空间，驱动层位于内核空间，他们的通信要通过系统调用如open()、read()、ioctl()等进行数据传递。
# 二、CameraService与HAL
Android Camera采用C/S架构，Client与Server两个独立进程之间使用Binder通信，系统开机的时候会启动CameraService并注册到ServiceManager中。Client通过
ServiceManager获取CameraService代理对象，就能调用
# 三、

# 透过现象看本质
1. 打开相机为什么能看到预览影像？
2. 相机应用调整参数是如何影响摄像头的？
3. 按下拍照键的时候发生了什么？
