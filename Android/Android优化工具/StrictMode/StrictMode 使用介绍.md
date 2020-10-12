## StrictMode 使用介绍

StrictMode 是 Android 2.3 引入的一个工具类，它被称为严苛模式，是 Android 提供的一种**运行时检测机制**，可以用来帮助开发人员用来检测代码中一些不规范的问题。对于我们的项目当中，可能会成千上万行代码，如果我们用肉眼 Review，这样不仅效率非常低效，而且比较容易出问题。使用 StrictMode 之后，**系统会自动检测出来在主线程中的一些异常情况，并按照我们的配置给出相应的反应。**

StrictMode 这个工具是非常强大的，但是我们可能因为对它不熟悉而忽略掉它。StrictMode 主要用来检测两大问题：

**1、线程策略**

线程策略的检测内容，是一些自定义的耗时调用、磁盘读取操作以及网络请求等。

线程策略使用 StrictMode.ThreadPolicy.Builder 构建()，主要方法如下：

	detectNetwork()：用于检查UI线程中是否有网络请求

    detectDiskReads()和detectDiskWrites()：UI线程磁盘读写检查

	detectResourceMismatches(): 最低版本为API23  发现资源不匹配

    detectCustomSlowCalls()：主要用于帮助开发者发现UI线程中调用的哪些方法执行的比较慢，要和StrictMode.noteSlowCall配合使用，只有通过  StrictMode.noteSlowCall标记“可能会”执行比较慢的方法，只有标记过的方法才能被检测到，日志中会记录方法的执行时间（注意：只有在主线程中执行的方法才会显示执行时间，在其他线程中执行的方法，就算是使用StrictMode.noteSlowCall标记，在日志中也不会打印执行时间）


**2、虚拟机策略**

虚拟机策略的检测内容如下：

 - **Activity 泄漏**
 - **Sqlite 对象泄漏**
 - **检测实例数量**

虚拟机策略使用 StrictMode.VmPolicy.Builder() 构建，主要方法如下：

	detectActivityLeaks() 用户检查 Activity 的内存泄露情况
	
	detectLeakedClosableObjects() 和 detectLeakedSqlLiteObjects()，资源没有正确关闭时回触发，detectLeakedSqlLiteObjects() 和 detectLeakedClosableObjects()的用法类似，只不过是用来检查 SQLiteCursor 或者 其他 SQLite 对象是否被正确关闭
	
	detectLeakedRegistrationObjects() 用来检查 BroadcastReceiver 或者 ServiceConnection 注册类对象是否被正确释放
	
	setClassInstanceLimit()，设置某个类的同时处于内存中的实例上限，可以协助检查内存泄露

	detectCleartextNetwork(): 最低版本为API23  检测明文的网络

	detectFileUriExposure()//最低版本为API18   检测file://或者是content://

**3、惩罚：**

	penaltyDeath()：当触发违规条件时，直接Crash掉当前应用程序
	
	penaltyDeathOnNetwork()：当触发网络违规时，Crash掉当前应用程序
	
	penaltyDialog()：触发违规时，显示对违规信息对话框
	
	penaltyFlashScreen()：会造成屏幕闪烁，不过一般的设备可能没有这个功能
	
	penaltyDropBox()：将违规信息记录到 dropbox 系统日志目录中（/data/system/dropbox），你可以通过如下命令进行插件

	penaltyLog()：一旦检测到将信息以LogCat的形式打印出来

**StrictMode 实战**

如果要在应用中使用 StrictMode，只需要在 Applicaitoin 的 onCreate 方法中对 StrictMode 进行统一配置，代码如下所示：

	// 设置线程策略
	StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder()//线程策略（ThreadPolicy）
	                    .detectDiskReads()//检测在UI线程读磁盘操作
	                    .detectDiskWrites()//检测UI线程写磁盘操作
	                    .detectCustomSlowCalls()//发现UI线程调用的哪些方法执行得比较慢
	                    .detectResourceMismatches()//最低版本为API23  发现资源不匹配
	//                    .detectNetwork() //检测在UI线程执行网络操作
	                    .detectAll()
	//                    .penaltyDialog()//一旦检测到弹出Dialog
	//                    .penaltyDeath()//一旦检测到应用就会崩溃
	//                    .penaltyFlashScreen()//一旦检测到应用将闪屏退出 有的设备不支持
	//                    .penaltyDeathOnNetwork()//一旦检测到应用就会崩溃
	                    .penaltyDropBox()//一旦检测到将信息存到DropBox文件夹中 data/system/dropbox
	                    .penaltyLog()//一旦检测到将信息以LogCat的形式打印出来
	//                    .permitDiskReads()//允许UI线程在磁盘上读操作
	                    .build());
	// 设置虚拟机策略
	StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder()//虚拟机策略（VmPolicy）
	                    .detectActivityLeaks()//最低版本API11 用户检查 Activity 的内存泄露情况
	                    .detectCleartextNetwork()//最低版本为API23  检测明文的网络
	                    .detectFileUriExposure()//最低版本为API18   检测file://或者是content://
	                    .detectLeakedClosableObjects()//最低版本API11  资源没有正确关闭时触发
	                    .detectLeakedRegistrationObjects()//最低版本API16  BroadcastReceiver、ServiceConnection是否被释放
	                    .detectLeakedSqlLiteObjects()//最低版本API9   资源没有正确关闭时回触发
	//                    .setClassInstanceLimit(MyClass.class, 2)//设置某个类的同时处于内存中的实例上限，可以协助检查内存泄露
	                    .penaltyLog()//与上面的一致
	//                    .penaltyDeath()
	                    .build());


最后，在日志输出栏中注意使用 “StrictMode” 关键字过滤出对应的 log 即可。