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
