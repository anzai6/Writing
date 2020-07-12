# Android 进阶解密摘要
> 参照《Android 进阶解密》做的摘要。

## Android 系统启动过程

### init 进程启动

Linux 内核加载完成后，它会寻找 init.rc 文件并启动 init 进程，init 进程的入口函数为 init.cpp 中的 main 函数，可以此为代码入口跟踪 init 进程的处理任务。

init 进程启动后主要做了三件事：

1. 创建和挂载启动所需的文件目录；
2. 初始化和启动属性服务，启动 ServiceManger 对各种服务进行管理；
3. 解析 init.rc 配置文件并启动 Zygote 进程

### Zygote 进程启动

Zygote 进程被称为孵化器，SystemServer 进程和应用程序进程都是通过 fork Zygote 进程来创建的。

**Zygote 进程启动流程：**

 - **app_main.cpp：**main()  
 ↓
 - **AndroidRunTime.cpp：**start() → startVm() → startReg()  
 ↓
 - **ZygoteInit.java：**main() → ZygoteServer.registerServerSocket() → preload() → startSystemServer()     
 ↓													
 - **ZygoteServer.java：**runSelectLoop()   


Zygote 进程启动主要做了一下几件事：

1. 创建 APPRuntime 并调用其 strat 方法，启动 Zygote 进程，即进入 Zygote 进程的入口函数 main，详情可看 app_main.cpp；
2. 创建 Java 虚拟机并为 Java 虚拟机注册 JNI 方法，因此后面 fork Zygote 创建的进程都会持有 Java 虚拟机实例以及 JNI 调用能力；
3. 通过 JNI 调用 ZygoteInit 的 main 函数进入 Zygote 的 Java 框架层，注意此前包括 init 进程操作等等都是在 C/C++ 层处理，经过上面步骤 2 后才能进入 Java 框架层；
4. 通过 registerZygoteSocket 方法创建服务端 Socket，并通过 runSelectLoop 方法等待 AMS 的请求来创建新的应用程序进程；
5. 启动 SystemServer 进程。

> 注： 为什么这里需要创建 Socket 进程通讯而不是通过 Binder 进行进程间通讯？因为进程的 fork() 对多线程不友好，仅会将发起调用的线程拷贝到子进程，其他线程都会立即停止，那如果一个线程在 fork() 前占用了某个互斥量，fork() 后被立即停止，这个互斥量就得不到释放，再去请求该互斥量就会发生死锁了。尽管 Zygote 没有采取 Binder 机制，它也不是单线程的，内部还跑了一些虚拟机相关的守护线程，但它在 fork() 前主动停止了其他线程，fork() 后再重新启动。相比来说自己启用的线程会比 Binder 线程池的线程更好控制。我猜还有一个问题是如果使用 Binder ，那 fork() 的时候停止了 Binder 线程池就可能导致错过某个进程的创建请求。

### SystemServer 进程启动

**SystemServer 进程启动流程：**

 - **ZygoteInit.java：**main() → startSystemServer() → Zygote.forkSystemServer() → handleSystemServerProcess() → zygoteInit() → nativeZygoteInit();  
 ↓
 - **RuntimeInit.java：**applicationInit() → invokeStaticMain()  
 ↓
 - **MethodAndArgsCaller.java：**run()  
 ↓											
 - **SystemServer.java：**main() → startBootstrapServices() → startCoreServices() → startOtherServices() → Looper.loop()        


SystemServer 进程主要用于创建系统服务，如 AMS、WMS 和 PMS 等，并维持这些服务与参与应用的管理和通讯。

SystemServer 进程启动主要做了一下几件事：

1. 启动 binder 线程池，这样就可以与其它进程进行通讯；
2. 创建 SystemServiceManager ，其用于对系统服务进行创建、启动和生命周期管理；
3. 启动各种系统服务，官方分为三大类：引导服务、核心服务和其它服务，具体类别可自行查询。

### Launcher 桌面应用启动

SystemServer 进程在启动的过程中会启动 PackageManagerService，PackageManagerService 启动后会将系统中的应用程序安装完成，然后已启动的 AMS 会将 Launcher 启动起来。Launcher 启动后会读取已安装的应用显示到桌面，具体逻辑课查看 Launcher.java 。

### Android 系统启动整体流程

1. **启动电源以及 BootLoader：**当电源按下时引导芯片会从固定位置加载引导程序 BootLoader，然后 BootLoader 会把系统 OS 拉起来并运行；
2.** Linux 内核启动：**当内核启动时，设置缓存、被保护存储器、计划列表、加载驱动。然后会在系统文件中寻找 init.rc 文件启动 init 进程；
3. **进程启动：**启动 init 进程，进而启动 Zygote 进程接着启动 SystemServer 进程，最后启动 Launcher 桌面应用，到此粗略的系统启动流程完成。

## 应用程序进程启动过程

**AMS 发送启动应用程序进程请求流程：**

 - **AMS.java：**startProcessLocked()  
 ↓
 - **Process.java：**start() → startViaZygote() → openZygoteSocketIfNeeded() → zygoteSendArgsAndGetResult()      
 ↓
 - **ZygoteState.java：**connect()   
   
**Zygote 接受请求并创建应用程序进程流程：**

 - **ZygoteInit.java：**main()  
 ↓
 - **SystemServer.java：**runSelectLoop()  
 ↓
 - **ZygoteConnection.java：**runOnce() → Zygote.forkAndSpecialize() → handleChildProc()  
 ↓											
 - **ZygoteInit.java：**zygoteInit() → nativeZygoteInit()   
 ↓	
 - **RuntimeInit.java：**applicationInit() → invokeStaticMain()  
 ↓
 - **MethodAndArgsCaller.java：**run()    
 ↓
 - **ActivityThread.java：**main()    

**解释：**

1. AMS 的 startProcessLocked() 会将 ActivityThread.java 的全类名作为进程入口参数向下传导，所以 Zygote 进程创建子进程后才能定位到子进程的入口并执行 ActivityThread.main() ；

2. 应用程序进程启动过程中会通过 ZygoteInit.nativeZygoteInit() 创建 Binder 线程池。

## 四大组件的工作过程

### 根 Activity 的启动过程

**Launcher 请求 AMS 过程：**

 - **Launcher.java：**startActivitySafely()  
 ↓
 - **Activity.java：**startActivity() → startActivityForResult()  
 ↓
 - **Instrumentation.java：**execStartActivity()  
 ↓											
 - **IActivityManager.java：**startActivity()   
 ↓	
 - **AMS.java：**startActivity()    
  
**AMS 到 ApplicationThread 的调用过程：**

 - **AMS.java：**startActivity() → startActivityAsUser()    
 ↓
 - **ActivityStarter.java：**startActivityMayWay() → startActivityLocked() → startActivity() → startActivity() → startActivityUnchecked()    
 ↓
 - **ActivityStackSupervisor.java：**resumeFocusedStackTopActivityLocked()  
 ↓											
 - **ActivityStack.java：**resumeTopActivityUncheckedLocked() →   resumeTopActivityInnerLocked()   
 ↓	
 - **ActivityStackSupervisor.java：**startSpecificActivityLocked() →  realStartActivityLocked()    
 ↓	
 - **ApplicationThread.java：**scheduleLaunchActivity()  

**ActivityThread 启动 Activity 的过程：**

 - **ApplicationThread.java：**scheduleLaunchActivity()  
 ↓
 - **ActivityThread.java：**sendMessage()   
 ↓
 - **H.java：**handleMessage()  
 ↓											
 - **ActivityThread.java：**handleLaunchActivity() →   performLaunchActivity()       
 ↓	
 - **Instrumentation.java：**callActivityOnCreate()    
 ↓	
 - **Activity.java：**performCreate() → onCreate()     

其中 ActivityThread 调用 performLaunchActivity() 时会通过 createBaseContextForActivity() 创建 ContextImpl 并通过 ClassLoader 创建 Activity 实例，然后通过 activity.attach() 绑定 ContextImpl 并关联 Window，performLaunchActivity() 结束后还会调用 handleResumeActivity() 将 Activity 的状态置为 Resume。

### Service 的启动过程

**ContextImpl 到 AMS 的调用过程：**

 - **ContextWrapper.java：**startService()  
 ↓
 - **ContextImpl.java：**startService() → startServiceCommon()   
 ↓
 - **IActivityManager.java：**startService()  
 ↓											
 - **AMS.java：**startService()       
  

**AMS 到 AcitivtyThread 的调用过程：**

 - **AMS.java：**startService()  
 ↓
 - **ActiveServices.java：**startServiceLocked() → startServiceInnerLocked() → bringUpServiceLocked() → realStartServiceLocked()     
 ↓
 - **ApplicationThread.java：**scheduleCreateService()  
 ↓
 - **ActivityThread.java：**sendMessage()   
 ↓
 - **H.java：**handleMessage()    
 ↓											
 - **ActivityThread.java：**handleCreateService()       
 ↓	
 - **Service.java：**onCreate()    
 
 其中 ActivityThread 调用 handleCreateService() 时会通过 ContextImpl.createAppContext 创建 ContextImpl 并通过 ClassLoader 创建 Service 实例，然后通过 service.attach() 绑定 ContextImpl 并初始化 service  , 然后调用 service.onCreate() 走 service 的生命周期函数。 

### Service 的绑定过程

**ContextImpl 到 AMS 的调用过程：**

 - **ContextWrapper.java：**bindService()  
 ↓
 - **ContextImpl.java：**bindService() → bindServiceCommon()   
 ↓
 - **IActivityManager.java：**bindService()  
 ↓											
 - **AMS.java：**bindService()       

**AMS 到 AcitivtyThread 的调用过程：**

 - **AMS.java：**bindService()  
 ↓
 - **ActiveServices.java：**bindServiceLocked() → requestServiceBindingLocked()     
 ↓
 - **ApplicationThread.java：**scheduleBindService()  
 ↓
 - **ActivityThread.java：**sendMessage()   
 ↓
 - **H.java：**handleMessage()    
 ↓											
 - **ActivityThread.java：**handleBindService()       
 ↓	
 - **AMS.java：**publishService()    
 
**Service 的绑定过程剩余流程：**

 - **AMS.java：**publishService()  
 ↓
 - **ActiveServices.java：**publishServiceLocked()     
 ↓
 - **InnerConnection.java：**connected()  
 ↓
 - **ServiceDiapatcher.java：**connected()   
 ↓
 - **RunConnection.java：**run()    
 ↓											
 - **ServiceDiapatcher.java：**doConnected()       
 ↓	
 - **ServiceConnection.java：**onServiceConnected()  
       
### 广播的注册、发送和接收过程

#### 广播注册

**ContextImpl 到 AMS 的调用过程：**

 - **ContextWrapper.java：**registerReceiver()  
 ↓
 - **ContextImpl.java：**registerReceiver() → registerReceiverInternal()   
 ↓
 - **IActivityManager.java：**registerReceiver()  
 ↓											
 - **AMS.java：**registerReceiver()    

其中会将 BroadcastReceiver 封装为 Binder 类型的 IIntentReceiver 传递到 AMS ，AMS 会将 IntentFilter 和 IIntentReceiver 封装为 BroadcastFilter 保存，当收到广播后会匹配对应的 IntentFilter 然后通过 ApplicationThread 回传 IIntentReceiver 进行广播接收者调用。

#### 广播发送

**ContextImpl 到 AMS 的调用过程：**

 - **ContextWrapper.java：**sendBroadcast()  
 ↓
 - **ContextImpl.java：**sendBroadcast() → sendBroadcast()   
 ↓
 - **IActivityManager.java：**broadcastIntent()  
 ↓											
 - **AMS.java：**broadcastIntent() → broadcastIntentLocked()   

**AMS 到 BroadcastReceiver 的调用过程：**

 - **BroadcastQueue.java：**scheduleBroadcastsLocked() → BroadcastHandler.handlerMessage() → deliverToRegisteredReceiverLocked() → performReceiverLocked()  
 ↓
 - **ApplicationThread.java：**scheduleRegisteredReceiver()  
 ↓
 - **InnerReceiver.java：**performReceiver()  
 ↓											
 - **ReceiverDispatcher.java：**performReceiver()   
 ↓											
 - **Args.java：**run()    
 ↓											
 - **BroadcastReceiver.java：**onReceiver()    
 
### ContentProvider 的启动过程 

**query 方法到 AMS 的调用过程：**

 - **ContentResolver.java：**query() → acquireUnstableProvider  
 ↓
 - **ApplicationContentResolver.java：**acquireUnstableProvider()  
 ↓
 - **ActivityThread.java：**acquireProvider()  
 ↓											
 - **IActivityManager.java：**getContentProvider()  
 ↓											
 - **AMS.java：**getContentProvider() → getContentProviderImpl() 
 ↓											
 - **ActivityThread.java：**main() → attach() 
 ↓											
 - **IActivityManager.java：**attachApplication() 
 ↓											
 - **AMS.java：**attachApplication() 

**AMS 启动 ContentProvider 的过程：**

 - **AMS.java：**attachApplication() → attachApplicationLocked()  
 ↓
 - **ApplicationThread.java：**bindApplication()  
 ↓
 - **ActivityThread.java：**sendMessage() → H.handleMessage() → handleBindApplication() → installContentProviders() → installProvider()  
 ↓											
 - **ContentProvider.java：**attachInfo() → onCreate()   
 
## 理解上下文 Context

Context 意为上下文，经常使用它来启动四大组件、访问资源、调用系统服务或者弹出 Toast 、创建 Dialog 之类的。它有两个子类 ContextWrapper 和 ContextImpl，ContextImpl 负责具体逻辑细节实现，ContextWrapper 则只是对 ContextImpl 进行基本的装饰，后面的类则继承自 ContextWrapper 进行进一步功能增强，比如针对主题强化的 ContextThemeWrapper 。Activity、Service、Application 都继承自 ContextWrapper，所以应用进程的 Context 数量就等于它们三者相加之和。

**Application Context 的创建过程：**

 - **ActivityThread.java：**main()   
 ↓
 - **LoadedApk.java：**makeApplication() → ContextImpl.createAppContext()  
 ↓
 - **Instrumentation.java：**newApplication()  
 ↓
 - **Application.java:**attach()  
 ↓
 - **ContextWrapper.java:**attachBaseContext()

Activity 和 Service 的 Context 创建在上面四大组件启动中可以看到，最终都是 ContextImpl 创建通过 attachBaseContext 设置的。

## 理解 ActivityManagerService 

ActivityManagerService 简称 AMS ，创建于 SystemServer 进程，主要负责四大组件的管理工作。Android 8.0 之前。应用程序与 AMS 通讯使用的是原始的 Binder 通讯，相对麻烦，8.0 开始则改为了使用 AIDL 接口，简化了代码交互逻辑。

### AMS 重要的数据结构

#### ActivityRecord

内部记录了 Activity 的所有信息，包括 AMS 的引用、AndroidManifest 节点信息、Activity 状态、Activity 资源信息和 Activity 进程相关信息以及 TaskRecord。

#### TaskRecord

就是我们通常所说的 Activity 任务栈，里面保存了栈中的各个 Activity 的 ActivityRecord 信息。

#### ActivityStack

ActivityStack 是一个管理类，用来管理系统所有 Activity，并按不同的状态进行分类。它内部存储各种不同状态的列表，列表的元素主要有 ActivityRecord 和 TaskRecord，比如 mTaskHistory 列表存储的是所有没有被销毁的 Activity 任务栈 TaskRecord。

## 理解 WindowManager 

Window 是一个每一个界面的载体，具体的实现为 PhoneWindow，它对 View 进行管理；WindowManager 继承接口 ViewManager ，实现类为 WindowManagerImpl，主要负责与 WindowManagerService 通信，WindowManagerService 则负责 Window 的具体管理工作。如果要对 Window 进行添加、更新和删除操作，需要借助 WindowManager 将具体的工作交由 WMS 来处理，WindowManager 和 WMS 通过 Binder 进行跨进程通信。

Activity 的 PhoneWindow 创建是在调用 Activity.attach() 方法内部创建的，接着PhoneWindow 又通过 setWindowManager 创建一个 WindowManagerImpl 实例，而 WindowManagerImpl 的内部又会将具体实现委托给 WindowManagerGlobal, WindowManagerGlobal 是一个单例，一个进程中只有一个 WindowManagerGlobal 实例。

### Window 的属性

#### Window 的类型

Window 的主要分为三大类型，分别是 Application Window（应用程序窗口），Sub Window（子窗口）、System Window（系统窗口）

1. **应用程序窗口：**Activity 就是一个典型的应用程序窗口，应用程序窗口的 Type 范围是 1-99 ，根据 Type 值的不同又会分为很多种应用程序窗口，具体可查阅资料或源码；
2. **子窗口：**子窗口无法独立存在，需要依附在其它窗口才可以，PopupWindow 就属于子窗口，子窗口的 Type 范围是 1000-1999；
3. **系统窗口：** Toast、出入法窗口、系统音量条窗口、系统错误窗口都属于系统窗口，系统窗口的 Type 范围是 2000-2999 ；

#### Window 的显示次序

如果把手机屏幕虚拟的用 X、Y、Z 轴表示，其中 Z 轴垂直于屏幕，则 Window 的显示次序可以看作是在 Z 轴上的次序，通常 Type 值越大，则 Window 的显示就越靠近用户，所以应用程序窗口 < 子窗口 < 系统窗口,< 代表显示层级。

#### Window 的操作

**Activity 的 Window 添加过程：**

 - **ActivityThread.java：**handleResumeActivity()   
 ↓
 - **WindowManagerImpl.java：**addView()  
 ↓
 - **WindowManagerGlobal.java：**addView()  
 ↓
 - **ViewRootImpl.java:**setView()  
 ↓
 - **Session.java:**addToDisPlay()  
 ↓
 - **WMS.java:**addWindow()

其中 WindowManagerGlobal 保存了每个窗口对应的 View 和 ViewRootImpl，具体的添加过程则是通过 ViewRootImpl 来进行的。ViewRootImpl 身负了很多职责，主要有一下几点：

 - View 树的根并管理 View 树；
 - 触发 View 的测量、布局和绘制；
 - 输入事件的中转站；
 - 管理 Surface；
 - 负责与 WMS 进行进程间通信。

**Window 的更新过程：**

 - **WindowManagerImpl.java：**updateViewLayout()  
 ↓
 - **WindowManagerGlobal.java：**updateViewLayout()  
 ↓
 - **ViewRootImpl.java:**setLayoutParams() -> scheduleTraversals() -> doTraversal() -> performTraversals() -> relaoutWindow()  
 ↓
 - **Session.java:**relayout()    
 ↓
 - **WMS.java:**relayoutWindow()

其中 ViewRootImpl 在 performTraversals() 调用 relaoutWindow() 后就会继续往下执行 View 的绘制三部曲： performMeasure() -> performLayout() -> performDraw() ;

## 理解 WindowManagerService

### WMS 的职责

WMS 是 Android 中重要的服务，它有很多职责，每个职责都会涉及重要且复杂的系统：

1. **窗口管理：**它是窗口的管理者，负责窗口的启动、添加和删除，还有对窗口的大小和显示层级进行管理；
2. **窗口动画：**窗口动画有 WMS 的动画子系统来负责，动画子系统的管理者为 WindowAnimator ；
3. **输入系统的中转站:**通过对窗口的触摸从而产生触摸事件，InputManagerService（IMS）会对触摸事件进行处理，它会寻找一个最合适的窗口来处理触摸反馈信息，WMS 是窗口的管理者，它作为输入系统的中转站再合适不过了；
4. **Surface 管理：**窗口并不具备绘制的功能，因此每个窗口都需要有一块 Surface 来供自己绘制，为每个窗口分配 Surface 是由 WMS 来完成的。

### WMS 的创建过程

WMS 的创建过程**依次**分为三个线程进行处理：

1. **system_server：**startCoreService() -> WMS.main() -> WMS.displayReady()
2. **android.display：**new WMS() -> WMS.initPolicy()
3. **android.ui：**PWM.init() 

android.display 的优先级比 system_server 高，所以 system_server 会等待 android.display 处理完毕，同理 android.display 也会等待 android.ui 线程处理完毕，具体实现可查看源码。

### WMS 的重要成员

#### mSessions: ArraySet

mSessions 的元素是 Session，它主要用于进程间通信，每个应用程序进程想和 WMS 进行通信都需要经过 Session ，并且每个应用程序进程都会对应一个  Session ，WMS 保存这些 Session 用来记录所有向 WMS 提出窗口管理服务的客户端。

#### mFinishedStarting: ArrayList

mFinishedStarting 的元素是 AppWindowToken，它是 WindowToken 的子类，WindowToken 主要有两个作用：

1. 窗口令牌，当应用程序想要向 WMS 申请新创建一个窗口，则需要向 WMS 出示有效的 WindowToken。所以每个 Activity 都对应一个 APPWindowToken；
2. WindowToken 会将同一个组件（比如同一个 Activity ）的窗口（WindowState）集合在一起，方便管理，即 Dialog 的窗口和 Activity 的窗口是放在一起管理的。

mFinishedStarting 存储已经完成启动的应用程序窗口的 AppWindowToken 列表。

### Window 的 WMS 添加过程

Window 的 WMS 添加过程主要集中在 addWindow() 方法，该方法主要做了四件事：

 - 对所要添加的窗口进行检查，如果窗口不满足一些条件，就不会再执行下面的代码逻辑；
 - WindowToken 相关的处理，比如有的窗口类型需要提供 WindowToken，没有提供的话就不会执行下面的代码逻辑，有的窗口类型则需要由 WMS 隐式创建 WindowToken；
 - WindowState 的创建和相关处理，将 WindowToken 和 WindowState 相关联；
 - 创建和配置 DisplayContent，完成窗口添加到系统前的准备工作。

### Window 的 删除过程

**Window 的删除过程：**

 - **WindowManagerImpl.java：**removeView()  
 ↓
 - **WindowManagerGlobal.java：**removeView() -> removeViewLocked()  
 ↓
 - **ViewRootImpl.java:**die() -> doDie() (doDie 内部会遍历子 View 调用 dispatchDetachedFromWindow())  
 ↓
 - **WindowManagerGlobal.java:**doRemoveView()    
 ↓
 - **WMS.java:**relayoutWindow()

**dispatchDetachedFromWindow 流程：**

 - **ViewRootImpl.java：**dispatchDetachedFromWindow()  
 ↓
 - **Session.java：**remove()  
 ↓
 - **WMS.java:**removeWindow() -> removeIfPossible() -> removeImmediately()  

Window 的删除逻辑比较复杂，以下作简单总结：

 - 检查删除线程的正确性，即只能是创建 View 的线程（通常称为 UI 线程，不一定是主线程）才能操作 View；
 - 从 ViewRootImpl 列表、布局参数列表和 View 列表中删除与 View 对应的元素；
 - 判断是否可以直接执行删除操作，如果不能就推迟删除操作；
 - 执行删除操作，清理和释放与 View 相关的一切资源。

## JNI 原理

JNI 是 Java 世界和 Native 世界的桥梁。

### Native 方法注册

Native 方法注册分为静态注册和动态注册，其中静态注册多用于 NDK 开发，而动态注册多用于 Framework 开发。

**静态注册**使用的是约定大于配置的思想，通过包名+类名+方法名的格式将 Java 方法和 JNI 函数建立关联

**动态注册**则需要手动将 Java 方法和 JNI 函数进行绑定注册，通常会在 JNI_OnLoad 函数中进行。

### 方法签名

JNI 的方法签名的格式为： (参数签名格式...)返回值签名格式

### 解析 JNIEnv

JNIEnv 是 Native 世界中 Java 环境的代表，通过 JNIEnv *指针可以在 Native 世界中访问 Java 世界的代码进行操作，它只在创建它的线程中有效，不能跨线程传递，因此不同线程的 JNIEnv 是彼此独立的，JNIEnv 的主要作用有以下两点：

 - 调用 Java 的方法。
 - 操作 Java (操作 Java 中的变量和对象等)。

JNIEnv 是一个结构体，其内部又包含了 JNINativeInterface。JNIEnv 中常用函数很多，这里列举三个，FindClass 用来找到 Java 中指定名称的类，GetMethodID 用来得到 Java 中的方法，GetFieldID 用来得到 Java 中的成员变量。

### 引用类型

JNI 的引用类型分别是本地引用（Local References）、全局引用（Global References）和弱全局引用（Weak Global References）。

本地引用：

 - 当 Native 函数返回时，这个本地引用就会被自动释放。
 - 只在创建它的线程中有效，不能够跨线程使用。
 - 局部引用是 JVM 负责的引用类型，受 JVM 管理。

全局引用：

 - 在 native 函数返回时不会被自动释放，因此全局引用需要手动来进行释放，并且不会被 GC 回收。
 - 全局引用是可以跨线程使用的。
 - 全局引用不受到 JVM 管理。

弱全局引用：

 - 弱全局引用是一种特殊的全局引用，它和全局引用的特点相似，不同的是弱全局引用是可以被 GC 回收的，弱全局引用被 GC 回收之后会只想 NULL。

## ClassLoader

Android 中的 ClassLoader 类型和 Java 中的 ClassLoader 类型类似，也分为两种类型，分别是系统类加载器和自定义类加载。其中系统类加载器主要包括 3 种，分别是 BootClassLoader 、 PathClassLoader 和 DexClassLoader。

**1.BootClassLoader**

Android 系统启动时会使用 BootClassLoader 来预加载常用类，与 JDK 中的 BootClassLoader 不同，它不是由 C/C++ 代码实现的，而是由 Java 实现的，BootClassLoader 是 ClassLoader 的内部类，进继承自 ClassLoader。

**2.DexClassLoader**

DexClassLoader 可以加载 dex 文件以及包含 dex 的压缩文件（apk 和 jar 文件），不管加载哪种文件，最终都要加载 dex 文件。

**3.DexClassLoader**

Android 系统使用 PathClassLoader 来加载系统类和应用程序的类。

PathClassLoader 和 DexClassLoader 都是继承自 BaseDexClassLoader，所以它们的构造方法都有如下 4 个参数：

 - dexPath：dex 相关文件路径集合，多个路径用文件分隔符分割，默认文件分隔符为":"。
 - optimizedDirectory：解压的 dex 文件存储路径，这个路径必须是一个内部存储路径，即应用程序的内部存储空间。
 - librarySearchPath：包含 C/C++ 库的路径集合，多个路径用文件分隔符分割，可以为 null。
 - parent：父加载器。

PathClassLoader 中的 optimizedDirectory 固定为 /data/dalvik-cache，所以 PathClassLoader 无法定义解压的 dex 文件存储路径，因此 PathClassLoader 通常用来加载已经安装的 apk 的 dex 文件（安装的 apk 的 dex 文件会存储在 /data/dalvik-cache 中）。DexClassLoader 则通常用于插件化中加载新的 dex 文件。

**ClassLoader 查找流程：**

 - **ClassLoader：**loadClass()  
 ↓
 - **BaseDexClassLoader.java：**findClass()  
 ↓
 - **DexPathList.java：**findClass()     
 ↓													
 - **Element.java：**findClass()   
 ↓													
 - **DexFile.java：**loadClassBinaryName()   

BootClassLoader 和 PathClassLoader 都是在 Zygote 进程的 ZygoteInit 中创建的。