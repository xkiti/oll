### 目录
* [背景](https://github.com/xkiti/oll/tree/main/Camera%E5%8E%9F%E7%90%86#背景)
* [安卓相机架构概览](https://github.com/xkiti/oll/tree/main/Camera%E5%8E%9F%E7%90%86#安卓相机架构概览)
* [应用层](https://github.com/xkiti/oll/tree/main/Camera%E5%8E%9F%E7%90%86#一应用层)
* [服务层](https://github.com/xkiti/oll/tree/main/Camera%E5%8E%9F%E7%90%86#二服务层)
* [硬件抽象层](https://github.com/xkiti/oll/tree/main/Camera%E5%8E%9F%E7%90%86#三硬件抽象层)
* [驱动层](https://github.com/xkiti/oll/tree/main/Camera%E5%8E%9F%E7%90%86#五相机驱动层V4L2框架)
# 安卓相机架构概览
Android Camera框架将整个相机行为抽象为两个类型的对象一种静态的Device对象，另一种是动态的Session对象。openCamera方法调用后，经过层层调用，最终App通过回调拿到Device对象。有了Device，首先应该建立相机会话，在建立会话过程中需要先告诉相机App想要的数据类型（宽高、数据格式、旋转角度等）这个步骤叫做StreamConfigure，相机会话建立成功后App通过回调拿到Session对象，有了Session对象App就能使用相机功能了，主要的功能有预览、拍照、录像。

Android Camera框架运用分层思想，将整个框架分为App、Camera Service、Camera Provider、HAL、Driver。如下图：  

<img src="imag/整体架构.png" alt="整体架构" width="50%" height="50%" />  

下面我将由上到下，通过介绍每一层完整的相机行为流程，加深对Camera框架的理解。  

# 一、应用层
App层直接面对的是相机应用，应用通过调用Camera Api 2接口来实现相机功能。Camera Api 2的具体实现是在Camera Framework框架中。整体的过程如下图：  
   
<img src="imag/app_layer_gc.png" alt="app_layer_gc" width="70%" height="70%" />

### 相关接口和类

Camera Api 2与Camera Framework之间通过java接口的方式来进行通信。需要强调的是，App 与Camera Framework之间的通信是双向的，App 在使用Camera Api的过程中也会向Camera Framework注册回调接口。相关的接口和接口实现情况如下图：  

<img src="imag/app_api_framework.png" alt="app_api_framework" /> 

* CameraManager:被定义为一个系统服务，通过Context.getSystemService来获取，主要用于检测、打开相机以及相机属性的获取。
* CameraDevice：代表了一个被打开的系统相机，用于创建CameraCaptureSession以及对于最后相机资源的释放。
* CameraDevice.StateCallback:在App中实现。用于返回创建Camera设备的结果，onOpened表示成功并返回CameraDevice实例，onError返回错误信息。
* CameraCaptureSession:代表了相机会话，建立了与Camera设备的通道，而之后对于Camera 设备的控制都是通过该通道来完成的
* CameraCaptureSessioon.StateCallback:在App中实现。用于返回创建CameraCaptureSession的结果，onConfigured表示成功并返回CameraCaptureSession实例，onConfigureFailed返回错误信息。

* CameraCaptureSession.CaptureCallback：由App实现。用于返回来自Camera Framework的数据和事件，onCaptureStarted方法在下发图像需求之后立即被调用，告知App此次图像需求已经收到，onCaptureProgressed返回partial meta data，onCaptureCompleted返回meta data数据。

* CameraDeviceImpl：在Camera Framework中实现。代表了一个相机设备，其内部持有Camera Service代理对象，会话的创建以及会话请求等工作，都是通过请求Camera Service来实现的。
* CameraCaptureSessionImpl：在Camera Framework中实现。每一个相机设备在一个时间段中，只能创建并存在一个CameraCaptureSession，其中该类包含了两种Session，一种是普通的，适用于一般情况下的会话操作，另一种是用于Reprocess流程的会话操作，该流程主要用于对于现有的图像数据进行再处理的操作。该类维护着来自实例化时传入的Surface列表，这些Surface正是包含了每一个图像请求的数据缓冲区。

Camera Framework为App提供的大部分功能实现，实际上是通过请求底层的Camera Service来完成的。Camera Service作为一个独立的服务进程运行在系统中，等待着其他进程的请求调用。

# 二、服务层

服务层位于Camera Framework层与Camera Provider层之间。Camera Framework通过AIDL向Camera Service发起请求，Camera Service收到请求后经过内部操作后，通过回调将结果返回返回给Framework。所以此处我们重点关注AIDL以及Camera Service 主程序的实现。

### 2.1.Camera AIDL 接口

AIDL是谷歌设计的一种自定义语言，用来方便、快速地实现Binder IPC通信。这边不展开！服务层AIDL接口和接口实现情况如下图：

![aidl](imag/aidl.png?lastModify=1623414222)

* ICameraService.aidl定义了ICameraService 接口。在CameraService类来实现，主要接口如下：

  getNumberOfCameras： 获取系统中支持的Camera 个数

  connectDevice()：打开一个Camera 设备

  addListener(): 添加针对Camera 设备以及闪光灯的监听对象 

* ICameraDeviceCallbacks.aidl文件中定义了ICameraDeviceCallbacks接口。在Framework中的CameraDeviceCallbacks类实现，主要接口如下：

  onResultReceived： 一旦Service收到结果数据，便会调用该接口发送至Framework。

  onCaptureStarted()： 一旦开始进行图像的采集，便调用该接口将部分信息以及时间戳上传至Framework。

  onDeviceError(): 一旦发生了错误，通过调用该接口通知Framework

* ICameraDeviceUser.aidl定义了ICameraDeviceUser接口。由CameraDeviceClient类实现，主要接口如下：

  disconnect： 关闭Camera 设备。

  submitRequestList：发送request。

  beginConfigure： 开始配置Camera 设备，需要在所有关于数据流的操作之前。

  endConfigure： 结束关于Camera 设备的配置，该接口需要在所有Request下发之前被调用。

  createDefaultRequest： 创建一个具有默认配置的Request。

* ICameraServiceListener.aidl定义了ICameraServiceListener接口。由Framework中的CameraManagerGlobal类实现，主要接口如下：

  onStatusChanged： 用于告知当前Camera 设备的状态的变更。

### 2.2 Camera Service 主程序

介绍Camera Service之前，先简单介绍下Camera Provider。它和Camera Service一样也是独立服务进程，是硬件抽象层的提供者，通过HIDL接口向其他进程提供硬件抽象层功能。Camera Service主程序随着系统启动而运行，启动阶段通过HIDL接口建立与Provider之间的通信。在服务阶段主要是提供一个桥梁，将来自Framework的请求经过转化处理交由Provider实现，同时在内部维护了从Framework以及Provider获取到的资源，按照一定的框架结构保持整个Service在稳定高效的状态下运行。所以接下来我们主要通过几个关键类、初始化过程以及处理来自App的请求三个部分来详细介绍下。

#### 关键类解析

![CameraService关键类](imag/CameraService%E5%85%B3%E9%94%AE%E7%B1%BB.png?lastModify=1623414222)

CameraService、DeviceInfo、ProviderInfo、CameraProviderManager及其内部类属于“静态”的，在Camera Service成功启动后就初始化完成，相当于资源，在服务内部使用。详细介绍如下：

* CameraService：实现了AIDL中ICameraService 接口，并且暴露给Camera Framework进行调用。持有CameraProviderManager实例。
* CameraProviderManager：Camera Provider管理者。内部的ProviderInfo列表代表着系统中的所有Camera Provider。

* ProviderInfo：内部维护一个Camera Provider代理，同时实现了Camera Provider端的ICameraProviderCallback接口，能接收来自Provider的事件回调。
* DeviceInfo3：内部维护着Camera Provider中的设备代理(ICameraDevice)，保持了对Camera Provider的控制。

CameraDeviceClient及其内部类是“动态”的，一次打开设备的操作就会实例化一次，关闭设备就会销毁回收。详细介绍如下：

* CameraDeviceClient：实现了ICameraDeviceUser接口，在相机打开过程中与Camera Framework形成了双向道路。其中内部包含了一个Camera3Device对象和FrameProcessorBase对象。
* Camera3Device：实现了Camera Provider端的ICameraDeviceCallbacks接口，通过该接口接收来自Provider的结果上传，进而传给CameraDeviceClient以及FrameProcessBase。
  * 将事件通过notify方法给到CameraDeviceClient。
  * meta data以及image data 会先到FrameProcessBase，然后给到CameraDeviceClient。
* FrameProcessorBase：主要用于metadata以及image data的中转处理。
* RequestThread：Camera3Device的内部线程对象，主要用于处理Request的接收与下发工作。

#### 启动初始化

首先我们看看CameraService是怎么运行起来的：

<img src="imag/camera_start_.png" alt="camera_start_" width="70%" height="70%" align="center"/>

当系统启动的时候会运行cameraserver程序， 在main方法中先初始化CameraService，最后将CameraService加入到service_manager中。在服务初始化的时候主要是实创建并初始化CameraProviderManager对象。而在CameraProviderManager初始化的过程中，主要做了三件事：

* 首先通过service_manager获取CameraProvider服务代理。
*  随后实例化了一个ProviderInfo对象，并对其进行初始化。
*  最后将ProviderInfo加入到一个内部容器中进行管理。

而在ProviderInfo初始化过程中做了四件事：

*  首先将自身注册到Camera Provider中，接收来自Provider的事件回调。
* 其次通过CameraProvider服务代理获取Provider端的设备代理对象。
* 再然后通过设备代理对象获取该设备对应的属性配置，创建DeviceInfo3对象，并且属性配置保存在DeviceInfo3对象中。
*  最后ProviderInfo会将每一个DeviceInfo3存入内部的一个容器中进行统一管理。

通过以上的系列动作，Camera Service进程便运行起来了，获取了Camera Provider的代理，同时也将自身关于Camera Provider的回调注册到了Provider中，这就建立了与Provider的通讯，另一边，通过服务的形式将AIDL接口也暴露给了Framework，静静等待来自  Framework的请求。

### 2.3 处理App请求

相机应用被打开后，首先要调用Camera Api的openCamera方法，经过Framework的内部处理，最终进入到CameraService中。CameraService内部主要做了获取相机设备属性，打开相机设备，之后会回调到App中去，App拿到返回的相机设备，再次下发创建Session以及下发Request请求。

#### 获取属性

由于在Camera Service启动初始化的时候已经获取了相应相机设备属性配置，并存储在DeviceInfo3中，所有直接返回DeviceInfo3中的属性即可。

#### 打开相机设备 

对于打开相机设备动作，内部实现比较复杂，接下来我们简单梳理下。

<img src="imag/opencamera.png" alt="opencamera" />

当Camera Framework调用ICameraService的connectDevice方法的时候就开始了打开相机动作。connectDevice方法主要做了两件事情：

* 实例CameraDeviceClient对象，并将connectDevice方法传入的回调接口保存在其属性变量中，同时创建了Camera3Device对象。
*  对CameraDeviceClient进行初始化，初始化完成后，通过保存的回调接口将自己传给Camera Framework，与之形成双向通信。

在对CameraDeviceClient初始化的过程中主要做了两件事：

* 对Camera3Device进行初始化：通过HIDL接口请求Camera Provider打开相机，并将返回的设备会话代理(ICameraDeviceSession)放入到Camera3Device对象中，最后启动Camera3Device对象中的RequestThread线程等待Request的下发
* 实例化FrameProcessorBase对象并且Camera3Device对象传入其中，这样就建立了FrameProcessorBase和Camera3Device的联系，之后将FrameProcessorBase内部线程运行起来，等待来自Camera3Device的结果。最后将CameraDeviceClient注册到FrameProcessorBase内部，建立FrameProcessorBase与CameraDeviceClient的联系。

connectDevice方法运行完毕，此时App已经获取了一个Camera设备(CameraDevice)，可以进行下一步的操作了。

#### 配置数据流 

配置数据流过程可以看做是一个事务，从startConfigure开始中间经历过deleteStream、createStream、cancelRequest 最后由endConfigure触发配置。

* startConfigure：空实现。
* cancelRequest：在该方法中会去通知Camera3Device将RequestThread中的Request队列清空，停止Request的继续下发。
* deleteStream：删除之前的数据流。
* createStream：为新的操作创建数据流。
* endConfigure：在该方法中会调用Camera3Device的configureStreams的方法，而该方法又会去通过ICameraDeviceSession的configureStreams_3_4的方法最终将需求传递给Provide；

#### 创建Request

App通过调用CameraDeviceImpl中的createCaptureRequest来实现，该方法在Framework中实现，内部会再去调用Camera Service中的AIDL接口createDefaultRequest，该接口的实现是CameraDeviceClient，在其内部又会去调用Camera3Device的createDefaultRequest方法，最后将需求下发到Provider端去创建一个默认的Request配置，一旦操作完成，Provider会将配置上传至Service，进而给到App中。

#### 处理图像需求 

下发图像采集需求大致分为两个流程，一个是预览，一个拍照，两者差异主要体现在拍照的Request优先级高于预览，具体表现是当预览Request在不断下发的时候，来了一次拍照需求，在Camera3Device 的RequestThread线程中，会优先下发此次拍照的Request。这里我们主要梳理下下发拍照request的大体流程：

* 下发拍照Request到Camera Service，其操作主要是由CameraDevcieClient来实现，submitRequestList方法来实现，在该方法中，会调用Camera3Device的setStreamingRequestList方法，将需求发送到Camera3Device中，而Camera3Device将需求又加入到RequestThread中的RequestQueue中，并唤醒RequestThread线程，在该线程被唤醒后，会从RequestQueue中取出Request发送至Provider中。

#### 接收图像结果 

针对结果的获取是通过异步实现，主要分为两个部分，一个是事件的回传，一个是数据的回传，而数据中又根据流程的差异主要分为Meta Data和Image Data两个部分，接下来我们详细介绍下：

* 事件的回传：在下发Request之后，首先从Provider端传来的是Shutter Notify，因为之前已经将Camera3Device作为ICameraDeviceCallback的实现传入Provider中，所以此时会调用Camera3Device的notify方法将事件传入Camera Service中，紧接着通过层层调用，将事件通过CameraDeviceClient的notifyShutter方法发送到CameraDeviceClient中，之后又通过打开相机设备时传入的Framework的CameraDeviceCallbacks接口的onCaptureStarted方法将事件最终传入Framework，进而给到App端。

* Meta Data生成：Camera Provider会通过ICameraDeviceCallback的processCaptureResult_3_4方法将数据给到Camera Service，而该接口的实现对应的是Camera3Device的processCaptureResult_3_4方法，在该方法会通过层层调用，调用sendCaptureResult方法将Result放入一个mResultQueue中，并且通知FrameProcessorBase的线程去取出Result，并且将其发送至CameraDeviceClient中，之后通过内部的CameraDeviceCallbacks远程代理的onResultReceived方法将结果上传至Framework层，进而给到App中进行处理。

* Image Data生成：前期也会按照类似的流程走到Camera3Device中，但是会通过调用returnOutputBuffers方法将数据给到Camera3OutputStream中，而该Stream中会通过BufferQueue这一生产者消费者模式中的生产者的queue方法通知消费者对该buffer进行消费，而消费者正是App端的诸如ImageReader等拥有Surface的类，最后App便可以将图像数据取出进行后期处理了。



# 三、硬件抽象层

硬件抽象层就是对Linux内核驱动程序的封装，向上提供接口，屏蔽低层的实现细节。也就是说，把对硬件的支持分成了两层，一层放在用户空间，一层放在内核空间，其中，硬件抽象层运行在用户空间，而Linux内核驱动程序运行在内核空间。Android分出硬件抽象层的原因是为了隐藏具体的硬件逻辑，从商业的角度看保护了相关厂家的利益。

### 1 Camera HAL3 接口

HAL基本结构hw_module_t，hw_device_t只有open和close方法很难满足Camera这样复杂的设备。因此谷歌通过将这两个基本结构嵌入到更大的结构体内部，同时在更大的结构内部定义了各自模块特有的方法，扩展了HAL接口。运用该方法针对Camera提出了HAL3接口。HAL3接口包括了用于代表一系列操作主体的结构体以及具体操作函数，接下来我们分别进行详细介绍：

### 1.1 核心结构体解析

#### camera_module_t以及camera3_device_t

```c++
typedef struct camera_module {
    hw_module_t common;
    int (*get_number_of_cameras)(void);
    int (*get_camera_info)(int camera_id, struct camera_info *info);
    int (*set_callbacks)(const camera_module_callbacks_t *callbacks);
    void (*get_vendor_tag_ops)(vendor_tag_ops_t* ops);
    int (*open_legacy)(const struct hw_module_t* module, const char* id, uint32_t halVersion, struct hw_device_t** device);
    int (*set_torch_mode)(const char* camera_id, bool enabled);
    int (*init)();
    int (*get_physical_camera_info)(int physical_camera_id, int (*is_stream_combination_supported)(int camera_id,const camera_stream_combination_t *streams);
    int (*is_stream_combination_supported)(int camera_id, const camera_stream_combination_t *streams);
    void (*notify_device_state_change)(uint64_t deviceState);
} camera_module_t;
  
 
typedef struct camera3_device {
    hw_device_t common;
    camera3_device_ops_t *ops; //拓展接口，Camera HAL3定义的标准接口
    void *priv;
} camera3_device_t;
```

#### camera3_stream_configuration

```c++
//该结构体主要用来代表配置的数据流列表，内部装有上层需要进行配置的数据流的指针
typedef struct camera3_stream_configuration {
    uint32_t num_streams;//代表了来自上层的数据流的数量，其中包括了output以及input stream。
    camera3_stream_t **streams;//是streams的指针数组，包括了至少一条output stream以及至多一条input stream。
    uint32_t operation_mode;//当前数据流的操作模式，该模式在camera3_stream_configuration_mode_t中被定义，HAL通过这个参数														//可以针对streams做不同的设置。
    const camera_metadata_t *session_parameters;//该参数可以作为缺省参数，直接设置为NULL即可，       CAMERA_DEVICE_API_VERSION_3_5以上的版本才支持
} camera3_stream_configuration_t;
```


#### camera3_stream_t

```c++
//该结构体主要用来代表具体的数据流实体，在整个的配置过程中，需要在上层进行填充，当下发到HAL中后，HAL会针对其中的各项属性进行配置
typedef struct camera3_stream {
    int stream_type;//表示数据流的类型，类型在camera3_stream_type_t中被定义。
    uint32_t width;//表示当前数据流中的buffer的宽度。
    uint32_t height;//表示当前数据流中buffer的高度。
    int format;//表示当前数据流中buffer的格式，该格式是在system/core/include/system/graphics.h中被定义。
    uint32_t usage;//表示当前数据流的gralloc用法，其用法定义在gralloc.h中。
    uint32_t max_buffers;//指定了当前数据流中可能支持的最大数据buffer数量。
    void *priv;
    android_dataspace_t data_space;//指定了当前数据流buffer中存储的图像数据的颜色空间。
    int rotation;//指定了当前数据流的输出buffer的旋转角度，其角度的定义在camera3_stream_rotation_t中，该参数由Camera 											//Service进行设置，必须在HAL中进行设置，该参数对于input stream并没有效果。
    const char* physical_camera_id;//指定了当前数据流从属的物理camera Id。
}camera3_stream_t;
```


#### camera3_stream_buffer_t

```c++
//该结构体主要用来代表具体的buffer对象
typedef struct camera3_stream_buffer {
    camera3_stream_t *stream;//代表了从属的数据流
    buffer_handle_t *buffer//buffer句柄
    int status;
    int acquire_fence;
    int release_fence;
} camera3_stream_buffer_t;
```

### 1.2 核心接口函数解析

HAL3的核心接口都是在camera3_device_ops中被定义，该结构体定义了一系列的函数指针，用来指向平台厂商实际的实现方法，接下来就其中几个方法简单介绍下：

```c++
typedef struct camera3_device_ops {
    int (*initialize)(const struct camera3_device *, const camera3_callback_ops_t *callback_ops);
    int (*configure_streams)(const struct camera3_device *, camera3_stream_configuration_t *stream_list);
    int (*register_stream_buffers)(const struct camera3_device *,const camera3_stream_buffer_set_t *buffer_set);
    const camera_metadata_t* (*construct_default_request_settings)(const struct camera3_device *, int type);
    int (*process_capture_request)(const struct camera3_device *, camera3_capture_request_t *request);
    void (*get_metadata_vendor_tag_ops)(const struct camera3_device*, vendor_tag_query_ops_t* ops);
    void (*dump)(const struct camera3_device *, int fd);
    int (*flush)(const struct camera3_device *);
    void (*signal_stream_flush)(const struct camera3_device*, uint32_t num_streams, const camera3_stream_t* const* streams);
    int (*is_reconfiguration_required)(const struct camera3_device*, const camera_metadata_t* 				    old_session_params, const camera_metadata_t* new_session_params);
} camera3_device_ops_t;
```

#### initialize：

该方法必须在camera_module_t中的open方法之后，其它camera3_device_ops中方法之前被调用，主要用来将上层实现的回调方法注册到HAL中，并且根据需要在该方法中加入自定义的一些初始化操作。

#### configure_streams：

该方法在完成initialize方法之后，在调用process_capture_request方法之前被调用，主要用于重设当前正在运行的Pipeline以及设置新的输入输出流。

#### construct_default_request_settings：

该方法主要用于构建一系列默认的Camera Usecase的capture 设置项，通过camera_metadata_t来进行描述，其中返回值是一个camera_metadata_t指针，其指向的内存地址是由HAL来进行维护的，同样地，该方法需要在1ms内返回，最长不能超过5ms。

#### process_capture_request：

该方法用于下发单次新的capture request到HAL中， 上层必须保证该方法的调用都是在一个线程中完成，而且该方法是异步的，同时其结果并不是通过返回值给到上层，而是通过HAL调用另一个接口process_capture_result()来将结果返回给上层的，在使用的过程中，通过in-flight机制，保证短时间内下发足够多的request，从而满足帧率要求。

#### dump：

该方法用于打印当前Camera设备的状态，一般是由上层通过dumpsys工具输出debug dump信息或者主动抓取bugreport的时候被调用，该方法必须是非阻塞实现。

#### flush：

当上层需要执行新的configure_streams的时候，需要调用该方法去尽可能快地清除掉当前已经在处理中的或者即将处理的任务，为配置数据流提供一个相对稳定的环境，flush会在所有的buffer都得以释放，所有request都成功返回后才真正返回。

### 1.3 回调函数

上面的一系列方法是上层直接对下控制Camera Hal，而一旦Camera Hal产生了数据或者事件的时候，可以通过camera3_callback_ops中定义的回调方法将数据或者事件返回至上层，该结构体定义如下：

```c++
typedef struct camera3_callback_ops {
void (*process_capture_result)(const struct camera3_callback_ops *, const camera3_capture_result_t *result);
void (*notify)(const struct camera3_callback_ops *, const camera3_notify_msg_t *msg);
  
camera3_buffer_request_status_t (*request_stream_buffers)(
    const struct camera3_callback_ops *,
    uint32_t num_buffer_reqs,
    const camera3_buffer_request_t *buffer_reqs,
    /*out*/uint32_t *num_returned_buf_reqs,
    /*out*/camera3_stream_buffer_ret_t *returned_buf_reqs);
  
void (*return_stream_buffers)( const struct camera3_callback_ops *, uint32_t num_buffers, const camera3_stream_buffer_t* const* buffers);
} camera3_callback_ops_t;
```

其中常用的回调方法主要有两个：

#### process_capture_result

该方法用于返回HAL部分产生的metadata和image buffers，它与request是多对一的关系，同一个request，可能会对应到多个result，比如可以通过调用一次该方法用于返回metadata以及低分辨率的图像数据，再调用一次该方法用于返回jpeg格式的拍照数据，而这两次调用时对应于同一个process_capture_request动作。同一个Request的Metadata以及Image Buffers的先后顺序无关紧要，但是同一个数据流的不同Request之间的Result必须严格按照Request的下发先后顺序进行依次返回的，如若不然，会导致图像数据显示出现顺序错乱的情况。

#### notify

该方法用于异步返回HAL事件到上层，必须非阻塞实现。

### 2 Camera Provider

Android8.0Treble项目中，加入了Camera Provider这一抽象层，该层作为一个独立进程存在于整个系统中，并且通过HIDL成功地将Camera Hal Module从Camera Service中解耦出来，承担起了对Camera HAL的封装工作。在相机架构中，Camera Provider处于Camera Service和硬件抽象层之间。Camera Service通过HIDL请求Camera Provider，Camera Provider调用HAL3接口去控制相机。

<img src="imag/Camera_Provider.png" alt="Camera_Provider" width="50%" height="50%" align="center"/>

#### 2.1 Camera HIDL 接口

HIDL(接口定义语言)，其核心是接口的定义，而谷歌为了使开发者将注意力落在接口的定义上而不是机制的实现上，主动封装了HIDL机制的实现细节，开发者只需要通过*.hal文件定义接口，填充接口内部实际的实现即可，接下来来看下具体定义的几个主要接口：

<img src="imag/hal_define.png" alt="hal_define" style="zoom:50%;" />

#### ICameraProvider.hal

在Camera Provider启动的时候被实例化。主要接口如下：

> getCameraDeviceInterface_V3_x: 该方法主要用于Camera Service获取ICameraDevice，通过该对象可以控制Camera 设备的诸如配置数据流、下发request等具体行为。
> setCallback： 将Camera Service 实现的ICameraProviderCallback传入Camera Provider，一旦Provider有事件产生时便可以通过该对象通知Camera Service。

#### ICameraProviderCallback.hal：

在Camera Service 启动的时候被实例化，通过调用ICameraProvider::setCallback接口注册到Camera Provider中。其主要接口如下：

> cameraDeviceStatusChange： 将Camera 设备状态上传至Camera Service，状态由CameraDeviceStatus定义

#### ICameraDevice.hal源码如下：

其主要接口如下:

> open： 用于创建一个Camera设备，并且将Camera Service中继承ICameraDeviceCallback并实现了相应接口的Camera3Device作为参数传入Provider中，供Provider上传事件或者图像数据。
> getCameraCharacteristics：用于获取Camera设备的属性。

#### ICameraDeviceCallback.hal


通过调用ICameraDevice::open方法注册到Provider中，其主要接口如下：

> processCaptureResult_3_4: 一旦有图像数据产生会通过调用该方法将数据以及meta data上传至Camera Service。
> notify: 通过该方法上传事件至Camera Service中，比如shutter事件等。

#### ICameraDeviceSession.hal

其主要接口如下：

> constructDefaultRequestSettings：用于创建默认的Request配置项。
> configureStreams_3_5：用于配置数据流，其中包括了output buffer/Surface/图像格式大小等属性。
> processCaptureRequest_3_4：下发request到Provider中，一个request对应着一次图像需求。
> close: 关闭当前会话。

### 2.2 Camera Provider 主程序

接下来进入到Provider内部去看看，整个进程是如何运转的，以下图为例进行分析:

<img src="imag/camera_provider_init.png" alt="camera_provider_init" style="zoom:50%;" />

*  在系统初始化的时候，启动Provider进程并加入HWServiceManager中。

* 在启动Provider进程过程中，获取并保存camera_module_t结构体、当前支持的相机数量。同时往硬件抽象实现层传入回调。
* 监听来自CameraService的调用。

接下来以上图为例简单介绍下Provider中几个重要流程：

<img src="imag/camera_provider_简单流程.png" alt="camera_provider_简单流程"  />

* Camera Service请求相机信息：通过调用ICameraProvider接口获取ICameraDevice。在此过程中，Provider会去实例化一个CameraDevice对象，并且将之前存有camera_modult_t结构体的CameraModule对象传入CameraDevice中，这样就可以在CameraDevice内部通过CameraModule访问到camera_module_t的相关资源，然后将CameraDevice内部类TrampolineDeviceInterface_3_2（该类继承并实现了ICameraDevice接口）返回给Camera Service。

* Camera Service请求打开相机：通过之前获取的ICameraDevice，调用其open方法来打开Camera设备。在Provider中会去调用CameraDevice对象的open方法，在该方法内部会去调用camera_module_t结构体的open方法，从而获取到HAL部分的camera3_device_t结构体，紧接着Provider会实例化一个CameraDeviceSession对象，并且将刚才获取到的camera3_device_t结构体以参数的方式传入CameraDeviceSession中，在CameraDeviceSession的构造方法中又会调用CameraDeviceSession的initialize方法，在该方法内部又会去调用camera3_device_t结构体的ops内的initialize方法开始HAL部分的初始化工作，最后CameraDeviceSession对象被作为camera3_callback_ops的实现传入HAL，接收来自HAL的数据或者具体事件，当一切动作都完成后，Provider会将CameraDeviceSession::TrampolineSessionInterface_3_2（该类继承并实现了ICameraDeviceSession接口）对象通过HIDL回调的方法返回给Camera Service中。

* Camera Service请求预览、拍照、录像：通过调用ICameraDevcieSession的configureStreams_3_5接口进行数据流的配置。在Provider中，最终会通过调用之前获取的camera3_device_t结构体内ops的configure_streams方法下发到HAL中进行处理。
  Camera Service通过调用ICameraDevcieSession的processCaptureRequest_3_4接口下发request请求到Provider中，在Provider中，最终依然会通过调用获取的camera3_device_t结构体内ops中的process_capture_request方法将此次请求下发到HAL中进行处理。


# **四、相机驱动层–V4L2框架**

驱动程序在系统内核空间中按照特定协议与相机硬件进行通信，从而达到控制各个硬件设备，获取图像数据的目的。V4L2英文是Video for Linux 2，该框架是诞生于Linux系统，用于提供一个标准的视频控制框架，其中一般默认会嵌入media controller框架中进行统一管理。v4l2提供给用户空间操作节点，media controller控制对于每一个设备的枚举控制能力，于此同时，由于v4l2包含了一定数量的子设备，而这一系列的子设备都是处于平级关系，但是在实际的图像采集过程中，子设备之间往往还存在着包含于被包含的关系，所以为了维护并管理这种关系，media controller针对多个子设备建立了的一个拓扑图，数据流也就按照这个拓扑图进行流转。

### 1 V4L2框架关键结构体

[详细结构体](imag/v4l2-device.h)

以下是设备间的拓扑图：

<img src="imag/v4l2拓扑图.png" alt="v4l2拓扑图" />

从上图不难看出，v4l2_device作为顶层管理者，一方面通过嵌入到一个video_device中，暴露video设备节点给用户空间进行控制，另一方面，video_device内部会创建一个media_entity作为在media controller中的抽象体，被加入到media_device中的entitie链表中，此外，为了保持对所从属子设备的控制，内部还维护了一个挂载了所有子设备的subdevs链表。而对于其中每一个子设备而言，统一采用了v4l2_subdev结构体来进行描述，一方面通过嵌入到video_device，暴露v4l2_subdev子设备节点给用户空间进行控制，另一方面其内部也维护着在media controller中的对应的一个media_entity抽象体，而该抽象体也会链入到media_device中的entities链表中。通过加入entities链表的方式，media_device保持了对所有的设备信息的查询和控制的能力，而该能力会通过media controller框架在用户空间创建meida设备节点，将这种能力暴露给用户进行控制。



### 2 v4l2操作流程简介

整个对于v4l2的操作主要包含了如下几个主要流程：

<img src="imag/v4l2的操作流程.png" alt="v4l2的操作流程" width="50%" height="50%" />

在操作之前，还有一个准备工作需要做，那就是需要找到哪些是我们所需要的设备，而它的设备节点是什么，此时便可以通过打开media设备节点，并且通过ioctl注入MEDIA_IOC_ENUM_ENTITIES参数来获取v4l2_device下的video设备节点，该操作会调用到内核中的media_device_ioctl方法，而之后根据传入的命令，进而调用到media_device_enum_entities方法来枚举所有的设备。

1. 打开video设备：在需要进行视频数据流操作之前，调用open方法打开video设备将返回的字符句柄存在本地，之后的一系列操作都是基于该句柄。

2. 查看并设置设备：通过调用ioctl下发VIDIOC_G_PARM/VIDIOC_S_PARM命令来分别获取和设置参数。

3. 申请帧缓冲区：通过调用ioctl下发VIDIOC_REQBUFS命令，在内核开辟帧缓冲区，之后将缓冲区通过mmap方式映射到用户空间。

4. 将帧缓冲区入队：通过调用ioctl下发VIDIOC_QBUF命令将帧缓冲区加入到v4l2框架中的缓冲区队列中，静等硬件模块将图像数据填充到缓冲区中。

5. 开启数据流：调用ioctl并且下发VIDIOC_STREAMON命令，通知整个框架开始进行数据传输，其中还通知各个子设备开始进行工作，最终将数据填充到V4L2框架中的缓冲区队列中。

6. 将帧缓冲区出队：一旦数据流开始进行流转了，通过调用ioctl下发VIDIOC_DQBUF命令来获取帧缓冲区，并且将缓冲区的图像数据取出，进行预览、拍照或者录像的处理。处理完成之后，需要将此次缓冲区再次放入V4L2框架中的队列中等待下次的图像数据的填充。

### 3 v4l2模块初始化

v4l2框架是在linux内核中实现的，会在系统启动的过程中通过标准的module_init方式进行初始化。而其初始化主要包含了v4l2_device的初始化和v4l2_subde子设备的初始化。由于驱动的具体实现都交由各个平台厂商进行实现，内部逻辑都各不相同，所以这边抽离出主要方法来进行梳理：

* v4l2_device设备初始：

1. 创建v4l2_device结构体，填充信息，通过v4l2_device_register方法向系统注册并且创建video设备节点。
2. 创建media_device结构体，填充信息，通过media_device_register向系统注册，并创建media设备节点，并将其赋值给v4l2_device中的mdev。
3. 创建v4l2_device的media_entity,并将其添加到media controller进行管理。

* v4l2_device设备初始：

1. 创建v4l2_subdev结构体，填充信息，通过v4l2_device_register_subdev向系统注册，并将其挂载到v4l2_device设备中
2. 创建对应的media_entity，并通过media_device_register_entity方法其添加到media controller中进行统一管理。
3. 最后调用v4l2_device_register_subdev_nodes方法，为所有的设置了V4L2_SUBDEV_FL_HAS_DEVNODE属性的子设备创建设备节点。

### 4 高通KMD框架

利用了V4L2可扩展这一特性，高通在相机驱动部分实现了自有的一套KMD框架。创建了一个整体相机控制者的CRM，它以节点video0暴露给用户空间，主要用于管理内核中的Session、Request以及与子设备，同时各个子模块都实现了各自的v4l2_subdev设备，并且以v4l2_subdev节点暴露给用户空间，与此同时，高通还创建了另一个video1设备Camera SYNC，该设备主要用于同步数据流，保证用户空间和内核空间的buffer能够高效得进行传递。


