## BlockCanary 使用介绍

> 本文摘录自[官方文档](https://github.com/markzhai/AndroidPerformanceMonitor/blob/master/README_CN.md#如何分析log)和[作者博客](http://blog.zhaiyifan.cn/2016/01/16/BlockCanaryTransparentPerformanceMonitor/)

BlockCanary 是一个 Android 平台的一个非侵入式的性能监控组件，应用只需要实现一个抽象类，提供一些该组件需要的上下文环境，就可以在平时使用应用的时候检测主线程上的各种卡慢问题，并通过组件提供的各种信息分析出原因并进行修复。

### 使用方法

1. gradle 配置


	dependencies {
	    compile 'com.github.markzhai:blockcanary-android:1.5.0'
	
	    // 仅在debug包启用BlockCanary进行卡顿监控和提示的话，可以这么用
	    debugCompile 'com.github.markzhai:blockcanary-android:1.5.0'
	    releaseCompile 'com.github.markzhai:blockcanary-no-op:1.5.0'
	}

2. 实现自己的监控上下文（强烈建议看清所有配置项，避免使用错误）：


	public class AppBlockCanaryContext extends BlockCanaryContext {
	    // 实现各种上下文，包括应用标示符，用户uid，网络类型，卡慢判断阙值，Log保存位置等
	
	    /**
	     * Implement in your project.
	     *
	     * @return Qualifier which can specify this installation, like version + flavor.
	     */
	    public String provideQualifier() {
	        return "unknown";
	    }
	
	    /**
	     * Implement in your project.
	     *
	     * @return user id
	     */
	    public String provideUid() {
	        return "uid";
	    }
	
	    /**
	     * Network type
	     *
	     * @return {@link String} like 2G, 3G, 4G, wifi, etc.
	     */
	    public String provideNetworkType() {
	        return "unknown";
	    }
	
	    /**
	     * Config monitor duration, after this time BlockCanary will stop, use
	     * with {@code BlockCanary}'s isMonitorDurationEnd
	     *
	     * @return monitor last duration (in hour)
	     */
	    public int provideMonitorDuration() {
	        return -1;
	    }
	
	    /**
	     * Config block threshold (in millis), dispatch over this duration is regarded as a BLOCK. You may set it
	     * from performance of device.
	     *
	     * @return threshold in mills
	     */
	    public int provideBlockThreshold() {
	        return 1000;
	    }
	
	    /**
	     * Thread stack dump interval, use when block happens, BlockCanary will dump on main thread
	     * stack according to current sample cycle.
	     * <p>
	     * Because the implementation mechanism of Looper, real dump interval would be longer than
	     * the period specified here (especially when cpu is busier).
	     * </p>
	     *
	     * @return dump interval (in millis)
	     */
	    public int provideDumpInterval() {
	        return provideBlockThreshold();
	    }
	
	    /**
	     * Path to save log, like "/blockcanary/", will save to sdcard if can.
	     *
	     * @return path of log files
	     */
	    public String providePath() {
	        return "/blockcanary/";
	    }
	
	    /**
	     * If need notification to notice block.
	     *
	     * @return true if need, else if not need.
	     */
	    public boolean displayNotification() {
	        return true;
	    }
	
	    /**
	     * Implement in your project, bundle files into a zip file.
	     *
	     * @param src  files before compress
	     * @param dest files compressed
	     * @return true if compression is successful
	     */
	    public boolean zip(File[] src, File dest) {
	        return false;
	    }
	
	    /**
	     * Implement in your project, bundled log files.
	     *
	     * @param zippedFile zipped file
	     */
	    public void upload(File zippedFile) {
	        throw new UnsupportedOperationException();
	    }
	
	
	    /**
	     * Packages that developer concern, by default it uses process name,
	     * put high priority one in pre-order.
	     *
	     * @return null if simply concern only package with process name.
	     */
	    public List<String> concernPackages() {
	        return null;
	    }
	
	    /**
	     * Filter stack without any in concern package, used with @{code concernPackages}.
	     *
	     * @return true if filter, false it not.
	     */
	    public boolean filterNonConcernStack() {
	        return false;
	    }
	
	    /**
	     * Provide white list, entry in white list will not be shown in ui list.
	     *
	     * @return return null if you don't need white-list filter.
	     */
	    public List<String> provideWhiteList() {
	        LinkedList<String> whiteList = new LinkedList<>();
	        whiteList.add("org.chromium");
	        return whiteList;
	    }
	
	    /**
	     * Whether to delete files whose stack is in white list, used with white-list.
	     *
	     * @return true if delete, false it not.
	     */
	    public boolean deleteFilesInWhiteList() {
	        return true;
	    }
	
	    /**
	     * Block interceptor, developer may provide their own actions.
	     */
	    public void onBlock(Context context, BlockInfo blockInfo) {
	
	    }
	}

3. 在 Application 中使用：


	public class DemoApplication extends Application {
	    @Override
	    public void onCreate() {
	        // 在主进程初始化调用哈
	        BlockCanary.install(this, new AppBlockCanaryContext()).start();
	    }
	}

### 原理

熟悉 Message/Looper/Handler 系列的同学们一定知道 Looper.java 中这么一段：

	private static Looper sMainLooper;  // guarded by Looper.class
	
	...
	
	/**
	 * Initialize the current thread as a looper, marking it as an
	 * application's main looper. The main looper for your application
	 * is created by the Android environment, so you should never need
	 * to call this function yourself.  See also: {@link #prepare()}
	 */
	public static void prepareMainLooper() {
	    prepare(false);
	    synchronized (Looper.class) {
	        if (sMainLooper != null) {
	            throw new IllegalStateException("The main Looper has already been prepared.");
	        }
	        sMainLooper = myLooper();
	    }
	}
	
	/** Returns the application's main looper, which lives in the main thread of the application.
	 */
	public static Looper getMainLooper() {
	    synchronized (Looper.class) {
	        return sMainLooper;
	    }
	}

即整个应用的主线程，只有这一个 looper，不管有多少 handler，最后都会回到这里。

如果再细心一点会发现在 Looper 的 loop 方法中有这么一段

	public static void loop() {
	    ...
	
	    for (;;) {
	        ...
	
	        // This must be in a local variable, in case a UI event sets the logger
	        Printer logging = me.mLogging;
	        if (logging != null) {
	            logging.println(">>>>> Dispatching to " + msg.target + " " +
	                    msg.callback + ": " + msg.what);
	        }
	
	        msg.target.dispatchMessage(msg);
	
	        if (logging != null) {
	            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
	        }
	
	        ...
	    }
	}

是的，就是这个 Printer - mLogging，它在每个 message 处理的前后被调用，而如果主线程卡住了，不就是在 dispatchMessage 里卡住了吗？

核心流程图：

<div align="center" >
   <img src="blockcanary_1.png" width = "900"  align="center" />
</div>

该组件利用了主线程的消息队列处理机制，通过

	Looper.getMainLooper().setMessageLogging(mainLooperPrinter);

并在 **mainLooperPrinter** 中判断 start 和 end，来获取主线程 dispatch 该 message 的开始和结束时间，并判定该时间超过阈值(如2000毫秒)为主线程卡慢发生，并 dump 出各种信息，提供开发者分析性能瓶颈。

	...
	@Override
	public void println(String x) {
	    if (!mStartedPrinting) {
	        mStartTimeMillis = System.currentTimeMillis();
	        mStartThreadTimeMillis = SystemClock.currentThreadTimeMillis();
	        mStartedPrinting = true;
	    } else {
	        final long endTime = System.currentTimeMillis();
	        mStartedPrinting = false;
	        if (isBlock(endTime)) {
	            notifyBlockEvent(endTime);
	        }
	    }
	}
	
	private boolean isBlock(long endTime) {
	    return endTime - mStartTimeMillis > mBlockThresholdMillis;
	}
	...

说到此处，想到是不是可以用mainLooperPrinter来做更多事情呢？既然主线程都在这里，那只要parse出app包名的第一行，每次打印出来，是不是就不需要打点也能记录出用户操作路径？ 再者，比如想做onClick到页面创建后的耗时统计，是不是也能用这个原理呢？ 之后可以试试看这个思路（目前存在问题是获取线程堆栈是定时3秒取一次的，很可能一些比较快的方法操作一下子完成了没法在stacktrace里面反映出来）。

### 如何分析 log

除了图形界面可以供开发、测试阶段直接看卡顿原因外，更多的使用场景其实在于大范围的 log 采集和分析：如线上环境和 monkey，或者测试同学们在整个测试阶段的 log 收集和分析。

对于分析，主要可以从以下维度

 - 卡顿时间
 - 同堆栈的卡顿出现次数 进行排序和归类。

接着说说对各个 log 分析的过程。

 - 首先可以根据手机性能，如核数、机型、内存来判断对应耗时是不是应该判定为卡顿。如一些差的机器，或者内存本身不足的时候。
 - 根据 CPU 情况，是否是 app 拿不到 cpu，被其他应用拿走了。
 - 看 timecost 和 threadtimecost，如果两者差得很多，则是主线程被等待或者资源被抢占。
 - 看卡顿发生前最近的几次堆栈，如果堆栈相同，则可以判定为是该处发生卡顿，否则需要比较分析。

