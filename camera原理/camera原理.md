### 背景

微信支付业务功能实现依托于Android Camera，为了能快速着手相关的业务开发需要快速掌握Camera原理。

### 前言

抛开Android系统，从用相机拍照的过程说起。首先，打开相机预览窗口会展示出要拍摄的相片效果，然后我们根据自己的需求调整相应的参数（闪光灯，镜头，光圈等），接着按下快门生成照片存入存储器。回到Android，用户使用相机应来拍照，这里的相机应用其实是真实相机硬件的抽象，具体拍照实现是Android系统通过驱动来控制摄像头完成的。如果把相机应用看做App层，摄像头看做硬件层，那么这两层中间是否存在其他分层呢？Android如何实现层级之间通信的，又是用什么数据格式来通信的呢？

### 一、Android Camera整体框架

为了满足更加复杂的相机应用场景，Google推出Camera2框架代替Camera1框架。这里说的Camera框架默认为Camera2。

下面给出一张官方整体框架图：

![camera整体框架](/Users/xkidi/Documents/learn_camera/camera整体框架.png)

从图中可以看出Camera大体分为4层：

1）应用层：应用开发者调用AOSP提供的接口即可,AOSP的接口即Android提供的相机应用的通用接口,这些接口将通过Binder与Framework层的相机服务进行操作与数据传递;

2）Framework层：相机Framework服务是承上启下的作用，上与应用交互下与HAL层交互。

3）HAL层：硬件抽象层，Android定义好了Framework服务与HAL层通信的协议及接口，HAL层如何实现由各个Vendor自己实现，如Qcom的老架构mm-Camera,新架构Camx架构,Mtk的P之后的Hal3架构。

4）驱动层：数据由硬件到驱动层处理,驱动层接收HAL层数据以及传递Sensor数据给到HAL层（Sensor芯片不同驱动也不同）。

###  二、Camera2 API涉及到的类

1. CameraManager ：是一个负责查询和建立相机连接的系统服务，它的功能不多，这里列出几个 CameraManager 的关键功能：
   1. 将相机信息封装到 CameraCharacteristics 中，并提获取 CameraCharacteristics 实例的方式。
   2. 根据指定的相机 ID 连接相机设备。
   3. 提供将闪光灯设置成手电筒模式的快捷方式。

2. CameraCharacteristics 是一个只读的相机信息提供者，其内部携带大量的相机信息，包括代表相机朝向的 `LENS_FACING`；判断闪光灯是否可用的 `FLASH_INFO_AVAILABLE`；获取所有可用 AE 模式的 `CONTROL_AE_AVAILABLE_MODES` 等等。如果你对 Camera1 比较熟悉，那么 CameraCharacteristics 有点像 Camera1 的 `Camera.CameraInfo` 或者 `Camera.Parameters`。

3. CameraDevice 代表当前连接的相机设备，它的职责有以下四个：
   1. 根据指定的参数创建 CameraCaptureSession。
   2. 根据指定的模板创建 CaptureRequest。
   3. 关闭相机设备。
   4. 监听相机设备的状态，例如断开连接、开启成功和开启失败等。

4. Surface 是一块用于填充图像数据的内存空间，例如你可以使用 SurfaceView 的 Surface 接收每一帧预览数据用于显示预览画面，也可以使用 ImageReader 的 Surface 接收 JPEG 或 YUV 数据。每一个 Surface 都可以有自己的尺寸和数据格式，你可以从 CameraCharacteristics 获取某一个数据格式支持的尺寸列表。

5. CameraCaptureSession 实际上就是配置了目标 Surface 的 Pipeline 实例，我们在使用相机功能之前必须先创建 CameraCaptureSession 实例。一个 CameraDevice 一次只能开启一个 CameraCaptureSession，绝大部分的相机操作都是通过向 CameraCaptureSession 提交一个 Capture 请求实现的，例如拍照、连拍、设置闪光灯模式、触摸对焦、显示预览画面等等。

6. CaptureRequest 是向 CameraCaptureSession 提交 Capture 请求时的信息载体，其内部包括了本次 Capture 的参数配置和接收图像数据的 Surface。CaptureRequest 可以配置的信息非常多，包括图像格式、图像分辨率、传感器控制、闪光灯控制、3A 控制等等，可以说绝大部分的相机参数都是通过 CaptureRequest 配置的。值得注意的是每一个 CaptureRequest 表示一帧画面的操作，这意味着你可以精确控制每一帧的 Capture 操作。

7. CaptureResult 是每一次 Capture 操作的结果，里面包括了很多状态信息，包括闪光灯状态、对焦状态、时间戳等等。例如你可以在拍照完成的时候，通过 CaptureResult 获取本次拍照时的对焦状态和时间戳。需要注意的是，CaptureResult 并不包含任何图像数据，前面我们在介绍 Surface 的时候说了，图像数据都是从 Surface 获取的。

###三、Android Camera工作大体流程

![大体流程](/Users/xkidi/Documents/learn_camera/大体流程.png)

绿色框中是应用开发者需要做的操作,蓝色为AOSP提供的API,黄色为Native Framework Service,紫色为HAL层Service.
描述一下步骤:

1. App一般在MainActivity中使用SurfaceView或者SurfaceTexture+TextureView或者GLSurfaceView等控件作为显示预览界面的控件,共同点都是包含了一个单独的Surface作为取相机数据的容器.

2. 在MainActivity onCreate的时候调用API 去通知 CameraServer去connect HAL继而打开Camera硬件sensor。

3. openCamera成功会有回调从CameraServer通知到App,在onOpenedCamera回调中去调用startPreview的操作。此时会创建CameraCaptureSession,创建过程中会向CameraServer调用ConfigureStream的操作,ConfigureStream的参数中包含了第一步中空间中的Surface的引用,相当于App将Surface容器给到了CameraServer,CameraServer包装了下该Surface容器为stream,通过HIDL传递给HAL,继而HAL也做configureStream操作

4. ConfigureStream成功后CameraServer会给App回调通知ConfigStream成功,接下来App便会调用AOSP setRepeatingRequest接口给到CameraServer,CameraServer初始化时便起来了一个死循环线程等待来接收Request.

5. CameraServer将request交到Hal层去处理,得到HAL处理结果后取出该Request的处理Result中的Buffer填到App给到的容器中,SetRepeatingRequest为了预览,则交给Preview的Surface容器,如果是Capture Request则将收到的Buffer交给ImageReader的Surface容器.

6. Surface本质上是BufferQueue的使用者和封装者,当CameraServer中App设置来的Surface容器被填满了BufferQueue机制将会通知到应用,此时App中控件取出各自容器中的内容消费掉,Preview控件中的Surface中的内容将通过View提供到SurfaceFlinger中进行合成最终显示出来,即预览;而ImageReader中的Surface被填了,则App将会取出保存成图片文件消费掉。

### 四、Camera Hal3 子系统

Android 的相机硬件抽象层 (HAL) 可将 android.hardware.camera2 中较高级别的相机框架 API 连接到底层的相机驱动程序和硬件。
Android 8.0 引入了 Treble，用于将 CameraHal API 切换到由 HAL 接口描述语言 (HIDL) 定义的稳定接口。

![request整体流程](/Users/xkidi/Documents/learn_camera/request整体流程.png)

1. 应用向相机子系统发出request，一个request对应一组结果，request中包含所有配置信息。其中包括分辨率和像素格式；手动传感器、镜头和闪光灯控件；3A 操作模式；RAW 到 YUV 处理控件；以及统计信息的生成等。一次可发起多个请求，而且提交请求时不会出现阻塞。请求始终按照接收的顺序进行处理。

2. 图中看到request中携带了数据容器Surface,交到framework cameraserver中，打包成Camera3OutputStream实例，在一次CameraCaptureSession中包装成Hal request交给HAL层处理.。HAL层获取到处理数据后返回给CameraServer，即CaptureResult通知到Framework，Framework Cameraserver则得到HAL层传来的数据给他放进Stream中的容器Surface中，而这些Surface正是来自应用层封装了Surface的控件，这样App就得到了相机子系统传来的数据。

3. HAL3 基于captureRequest和CaptureResult来实现事件和数据的传递，一个Request会对应一个Result。



参考：

https://source.android.com/devices/camera

https://www.jianshu.com/p/4308cba918af

https://www.jianshu.com/p/bac0e72351e4

https://blog.csdn.net/sinat_22657459/article/details/79187086

https://blog.csdn.net/TaylorPotter/article/details/105630341

https://blog.csdn.net/TaylorPotter/article/details/105707181?spm=1001.2014.3001.5501

https://blog.csdn.net/TaylorPotter/article/details/106001598?spm=1001.2014.3001.5501

https://www.jianshu.com/p/f7f548c2c0e7

https://github.com/saki4510t/UVCCamera



### 