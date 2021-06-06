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
# 二、相机初始化
CameraService在系统启动时new了一个实例，以“media.camera”为名字注册到ServiceManager中。应用层的Cammear.java 调用open方法进入Native的世界，
在ServiceManager中找到CameraService的Binder代理，调用CameraService的connect方法实例化HAL接口hardware，hardware调用initialize进入HAL层打开Camear驱动。CameraService::connet()返回client的时候，就表明客户端和服务端连接建立。相机完成初始化，可以进行拍照和preview等动作。一个看似简单的Camera初始化过程，经历了好几层的调用。java->JNI->Binder IPC->系统调用(open)
# 三、相机Preview
初始化Camera后开始预览取景preview。所有拥有拍照功能的应用，它在预览的时候都要实现SurfaceHolder.Callback接口，并实现其surfaceCreated、surfaceChanged、surfaceDestoryed三个函数。同时声明一个用于预览的窗口SurfaceView，还要设置Camera预览的surface缓冲区，以供底层获取的preview数据可以不断放入到surface缓冲区内。设置好以上参数后，就可以调用startPreview方法进行取景预览。startPreview()也是一层层往下调用，最后到达CameraService。相机应用--->Camera.java(框架)--->android_hardware_camera.cpp(JNI)--->Camera.cpp(客户端)--->CameraService.cpp(服务端)。  
Framework层与HAL层是通过回调进行通信的，

# 透过现象看本质
1. 打开相机为什么能看到预览影像？
2. 相机应用调整参数是如何影响摄像头的？
3. 按下拍照键的时候发生了什么？
