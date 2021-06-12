# 背景：
智能时代下如何搭建安全稳定的Android客户端框架

# 一、现有架构
### 1、MVP
先来看一下MVC、MVP、MVVM大概的优缺点：
|  | MVC |MVP|MVVM|
| ----- | ----- |-----|-----|
| 优点 | 开发速度快|方便测试、View和Model之间解耦|数据驱动view的更新，低耦合|
| 缺点 | 业务复杂的时候严重影响迭代速度 |View和Present相互持有，存在生命周期一致性问题、业务复杂会造成Present臃肿|数据绑定导致出现Bug不好定位 |
 
由图表可以看出在Android框架设计中，MVC更适合页面逻辑简单，业务相对稳定的应用。MVVM最大的特点是数据驱动，数据的变化直接体现在View上减少了被引用的情况，碍于当时Android的开发环境对数据绑定的支持有限，而且对于数据绑定的bug没有很好的处理方案。从项目现在和未来的规划中，我们否认了MVC的架构方式，由于MVVM当时的不成熟放弃了MVVM。项目的业务会越来越复杂，但是只要保证业务逻辑和页面逻辑相互隔离不混乱，带来的代码臃肿还是能接受的，针对存在的生命周期一致性问题，解决方案很成熟而且实现简单。从这两点来看MVP符合当下以及将来的项目情况。为此我们设计实现了如下图的MVP架构：
<img src="imag/mvp_c.png" width="70%" height="50%"  /><br/>
这套MVP架构还为我们带来了一个额外的好处：我们有了足够明确的开发规范和标准。细致到了每一个类应该放到哪个包下，哪个类具体应该负责什么职责等等。这对于我们的Code Review、接手他人的功能模块等都提供了极大的便利
### MVP问题的解决方案
MVP存在的生命周期一致性问题指的是：Presenter的生命周期比View生命周期长，View的提前结束会导致内存泄漏。我们用可以在View生命周期结束后解绑Presenter，但这种情况下的粗暴解绑很容易造成空指针异常。为此我们让View实现了ViewInterface，Presenter不再持有View的实例而是持有ViewInterface的动态代理对象，这样Presenter在更新View之前就拥有了拦截的机会，也就解决了问题。
### 2、多进程
### 3、插件化


# 二、别的优秀app架构设计思路与优缺点
在想不到设计方案，或者拿捏不了架构好坏的时候，我们应该跳出“环境”，看看其他优秀架构的设计思路。分析他们的优缺点，避免自己陷入闭门造车。以下两个优秀app方案带给我新的思路。

### 1.微信支付UseCase

微信支付流程多，界面跳转复杂，传统的MV(X)模式忽略了业务流程（ViewController和ViewContrller之间该怎么跳转）。

### 架构设计思路：业务流程抽象

![usecase_solution](/Users/xkidi/Documents/app架构/usecase_solution.png)

将业务流程抽象为一个独立的角色 `UseCase`。同时, 把界面抽象为 `UIPage`。 一个大的业务流程可以分解为一个个小的业务流程。

####  优点

* 业务流程的代码能够聚合到 UseCase 中，而不是分散各个 ViewController，Activity 中。

* 业务流程和界面得到了复用。

* 契合微信支付多流程，界面跳转复杂的业务特点

### 2.饿了么EMC架构

饿了么移动APP从单一APP发展为多APP齐头并进的格局。需要快速复用业务组件。

###架构设计思路：业务模块注册机制

E(Excalibur)M(Modules)C(Common)。

<img src="/Users/xkidi/Desktop/EMC.png" alt="EMC" style="zoom:50%;" />

每个业务模块对外提供相应的业务接口，同时在系统启动的时候向Excalibur系统注册自己模块的Scheme（Excalibur是饿了么移动用来保存Scheme与模块之间映射的系统，同时能根据Scheme进行Class反射返回）。 当其他业务模块对该业务模块有依赖时，从Excalibur系统中获取相关实例，并调用相应接口来实现调用，从而实现了业务模块之间的解耦目的。

### 优点

* 高可用：保证了组件复用的同时，保留了原本组件的架构方案。
* 低耦合，避免了业务组件之间的直接依赖。

#### 缺点

* 需要手动注册模块Scheme
* class会带来性能损耗

# 三、架构优化点



# 四、整体架构
先看看看看模块化的整体设计图  
<img src="imag/整体架构.png" width="80%" height="50%"  /><br/>
## 3.1组件化与模块化
基础组件与业务无关且相对独立，可以复用避免重复造轮子。模块化更多的是考虑到并行开发和测试效率。智家App的模块化设计图如下：
  
<img src="imag/all_arc.png" width="70%" height="50%"  /><br/>
整个项目分为三层，从下往上分别是：
* Basic Component Layer: 基础组件层，顾名思义就是一些基础组件，包含了各种开源库以及和业务无关的各种自研工具库；
* Business Component Layer: 业务组件层，这一层的所有组件都是业务相关的；
* Business Module Layer: 业务module层，在Android Studio中每块业务对应一个单独的module。Module都必须准遵守前面提到的MVP架构。

同时针对模块化也需定义了一些自己的游戏规则:
* 对于Business Module Layer，各业务模块之间的通讯跳转采用路由框架ARouter来实现;
* 对于Business Component Layer，单一业务组件只能对应某一项具体的业务，对于有个性化需求的对外部提供接口让调用方定制;
* 各Layer间严禁反向依赖，横向依赖关系由各业务Leader和技术小组商讨决定。  

对于模块化项目，每个单独的business module都可以单独编译成APK。在开发阶段需要单独打包编译，项目发布的时候又需要它作为项目的一个module来整体编译打包。简单的说就是开发时是application，发布时是library。因此需要你在business module的gradle配置文件中加入如下代码：
```
if(isBuildModule.toBoolean()){
    apply plugin: 'com.android.application'
}else{
    apply plugin: 'com.android.library'
}
```
### 3.2路由架构

# 结尾
以上就是我对于智家App架构的整理，以及对App架构的一些粗浅看法，有很多不足，但也让我得到了成长。感恩！
 

