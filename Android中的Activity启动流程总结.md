## Android中的Activity启动流程总结
　<small>Activity的启动包括点击应用图标启动、在App内启动等方式，主要分析的就是列出来的两种方式。
> 　学习的主要目的是弄清的问题：  
> 1. Activity是什么？  
> 2.有什么用？  
> 3.系统是怎么创建的(即创建Activity的流程是怎么样的)？  
> ****
> 后面会在写两篇总结的学习文章，主要是讲述Android是怎么将Activity渲染到屏幕上的和怎么处理事件的，也算是给自己的一个TODOlist吧。

　Activity的生命周期和使用中常见的问题，在这里不讲，后面会开一篇总结。
### 1.Activity是什么
　官方的说法是Activity一个应用程序的组件，它提供一个屏幕来与用户交互，以便做一些诸如打电话、发邮件和看地图之类的事情。  
　网上有一个我认为很形象的比喻，Activity、Window、View三者的差别：  
　Activity像一个工匠（控制单元），Window像窗户（承载模型），View像窗花（显示视图） LayoutInflater像剪刀，Xml配置像窗花图纸。  
　在Windows中，有个窗口类来接受一个窗口的活动，系统就可以通过调用窗口类的方法来传递消息，这个窗口类既接受打开和关闭的活动，又接受用户输入事件。而在Android中而不同，Activity只接受打开和关闭等的活动，而不会接受输入事件，那是由Activity内嵌的Window类来接受的，然后转发给相应View。View的添加也是添加到Window上的，不是添加到Activity上（在Activity中调用setContentView(R.layout.xxx)其中实际上是调用的getWindow().setContentView()）。  
### 2.有什么用
　我的理解是Activity利用其内部的Decorview显示视图，利用其内部的Window承载视图和控制交互（如触摸事件）。
### 3.Activity的启动(点击图标的启动)
　这里主要是分两部分：  
　一部分是Zygote进程的启动，在Android系统中，所有的应用程序进程以及系统服务进程SystemServer都是由Zygote进程孕育（fork）出来的，SystemServer进程里运行了很多binder service，例如ActivityManagerService，PackageManagerService，WindowManagerService，这些binder service分别运行在不同的线程中。  
　另一部分就是从Zygote进程Fork出应用进程然后启动Activity。
#### 3.1 Zygote进程的启动
　android系统是基于Linux内核的，而在Linux系统中，所有的进程都是init进程的子孙进程，也就是说，所有的进程都是直接或者间接地由init进程fork出来的。Zygote进程也不例外，它是在系统启动的过程，由init进程创建的。在系统启动脚本system/core/rootdir/init.rc文件中，在init.rc文件中存有启动Zygote进程的脚本命令。  
　init.rc文件：  
<font color='#008B8B'>
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server  
socket zygote stream 666  
onrestart write /sys/android_power/request_state wake  
onrestart write /sys/power/state on  
onrestart restart media  
onrestart restart netd
</font>  

　前面的关键字service告诉init进程创建一个名为**"zygote"的进程**，这个zygote进程要执行的程序是/system/bin/app_process，后面是要传给app_process的参数。接下来的socket关键字表示这个zygote进程需要一个名称为**"zygote"的socket资源**。　　

　Zygote进程要执行的程序便是system/bin/app_process了，它的源代码位于frameworks/base/cmds/app_process/app_main.cpp文件中，入口函数是main。在继续分析Zygote进程启动的过程之前，我们先来看看它的启动序列图：
![](http://hi.csdn.net/attachment/201109/16/0_1316190384ZuU0.gif)
 下面我们就详细分析每一个步骤。  
 **Step 1. app_process.main**  
 这个函数的主要作用就是创建一个AppRuntime（它约继承于AndroidRuntime类）变量，然后调用它的start成员函数。  
　 由于我们在init.rc文件中，设置了app_process启动参数--zygote和--start-system-server，因此，在main函数里面，最终会执行下面语句。
```
runtime.start("com.android.internal.os.ZygoteInit",startSystemServer);
```
　这里的参数startSystemServer为true，表示要启动SystemServer组件。由于AppRuntime没有实现自己的start函数，它继承了父类AndroidRuntime的start函数，因此，下面会执行AndroidRuntime类的start函数。  
　**Step 2. AndroidRuntime.start**  
　这个函数的作用是启动Android系统运行时库，它主要做了三件事情，一是调用函数startVM启动虚拟机，二是调用函数startReg注册JNI方法，三是调用了com.android.internal.os.ZygoteInit类的main函数。  
　**Step 3. ZygoteInit.main　**　  
　  它主要作了三件事情，一个调用registerZygoteSocket函数创建了一个socket接口，用来和ActivityManagerService通讯，二是调用startSystemServer函数来启动SystemServer组件，三是调用runSelectLoopMode函数进入一个无限循环在前面创建的socket接口上等待ActivityManagerService请求创建新的应用程序进程。  
　   ** Step 4. ZygoteInit.registerZygoteSocket ** 
　   ** Step 5. ZygoteInit.startSystemServer**  

```java
public class ZygoteInit {
	......

	private static boolean startSystemServer()
			throws MethodAndArgsCaller, RuntimeException {
		/* Hardcoded command line to start the system server */
		String args[] = {
			"--setuid=1000",
			"--setgid=1000",
			"--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,3001,3002,3003",
			"--capabilities=130104352,130104352",
			"--runtime-init",
			"--nice-name=system_server",
			"com.android.server.SystemServer",
		};
		ZygoteConnection.Arguments parsedArgs = null;

		int pid;

		try {
			parsedArgs = new ZygoteConnection.Arguments(args);

			......

			/* Request to fork the system server process */
			//可以知道SystemServer进程是通过Zygote进程fork出来的
			pid = Zygote.forkSystemServer(
				parsedArgs.uid, parsedArgs.gid,
				parsedArgs.gids, debugFlags, null,
				parsedArgs.permittedCapabilities,
				parsedArgs.effectiveCapabilities);
		} catch (IllegalArgumentException ex) {
			......
		}

		/* For child process */
		if (pid == 0) {
			handleSystemServerProcess(parsedArgs);
		}

		return true;
	}
	
	......
}
```  
这里我们可以看到，Zygote进程通过Zygote.forkSystemServer函数来创建一个新的进程来启动SystemServer组件，返回值pid等0的地方就是新的进程要执行的路径，即新创建的进程会执行handleSystemServerProcess函数。  
　**Step 6. ZygoteInit.handleSystemServerProcess**   
　        由于由Zygote进程创建的子进程会继承Zygote进程在前面Step 4中创建的Socket文件描述符，而这里的子进程又不会用到它，因此，这里就调用closeServerSocket函数来关闭它。这个函数接着调用RuntimeInit.zygoteInit函数来进一步执行启动SystemServer组件的操作。  
　        **Step 7. RuntimeInit.zygoteInit**  
　        这个函数会执行两个操作，一个是调用zygoteInitNative函数来执行一个Binder进程间通信机制的初始化工作，这个工作完成之后，这个进程中的Binder对象就可以方便地进行进程间通信了（完成这一步后，这个进程的Binder进程间通信机制基础设施就准备好了。），另一个是调用上面Step 5传进来的com.android.server.SystemServer类的main函数。  
　         **Step 8. RuntimeInit.zygoteInitNative**   
　         **Step 9. SystemServer.main**  
　           这里的main函数首先会执行JNI方法init1，然后init1会调用这里的init2函数，在init2函数里面，会创建一个ServerThread线程对象来执行一些系统关键服务（如PackageManagerService和ActivityManagerService。）的启动操作。  
　            **Step 10. ZygoteInit.runSelectLoopMode**  
　            等待ActivityManagerService来连接这个Socket，然后调用ZygoteConnection.runOnce函数来创建新的应用程序。  
　            **总结一下：**  
　                    1. 系统启动时init进程会创建Zygote进程，Zygote进程负责后续Android应用程序框架层的其它进程的创建和启动工作。  
　                    2. Zygote进程会首先创建一个SystemServer进程，SystemServer进程负责启动系统的关键服务，如包管理服务PackageManagerService和应用程序组件管理服务ActivityManagerService。  
　                    3. 当我们需要启动一个Android应用程序时，ActivityManagerService会通过Socket进程间通信机制，通知Zygote进程为这个应用程序创建一个新的进程。
##### Zygote进程的启动总结
　Zygote进程由系统启动的时候，linux的init进程执行init.rc脚本创建，创建好的Zygote进程又会通过执行程序system/bin/appprocess来对Android应用程序框架层的其它进程(主要是SystemServer进程)的创建和启动工作。 
#### 3.2 应用进程的启动  
　当系统决定要在一个新的进程中启动一个Activity(如点击Launcher上的应用图标)或者Service时，ActivityManagerService就会通过Socket连接Zygote进程，通过Zygote进程fork出一个新的进程，然后在这个新的进程中启动这个Activity或者Service。
> 在新的进程中启动这个Activity，就是加载ActivityThread类，然后执行它的main方法。这个详细的过程下一节又分析。  

　ActivityManagerService启动新的进程是从其成员函数startProcessLocked开始的，在深入分析这个过程之前，我们先来看一下进程创建过程的序列图，然后再详细分析每一个步骤。
![](http://img.my.csdn.net/uploads/201109/5/0_1315236533f7n7.gif)
**Step 1. ActivityManagerService.startProcessLocked**  
 
```  
......       
int pid = Process.start("android.app.ActivityThread",  
                mSimpleProcessManagement ?app.processName : null, uid, uid,  
                gids, debugFlags, null);
......  

```
　它调用了Process.start函数开始为应用程序创建新的进程，注意，它传入一个第一个参数为"android.app.ActivityThread"，这就是进程初始化时要加载的Java类了，把这个类加载到进程之后，就会把它里面的静态成员函数main作为进程的入口点，后面我们会看到。  
**Step 2. Process.start**  
　打开设备文件设备文件/dev/binder，回到Process.start函数中，它调用startViaZygote函数进一步操作。  
**Step 3. Process.startViaZygote**  
　这个函数将创建进程的参数放到argsForZygote列表中去，如参数"--runtime-init"表示要为新创建的进程初始化运行时库，然后调用zygoteSendAndGetPid函数进一步操作。  
**Step 4. Process.zygoteSendAndGetPid**　  　
　打开这个Socket（这个Socket由ZygoteInit.java文件中的ZygoteInit类在runSelectLoopMode函数侦听的），并写入。  
**Step 5. ZygoteInit.runSelectLoopMode**  
　在runSelectLoopMode中，找到Socket连接（就是上一步的Socket连接）即调用 **peers.get(index)** 得到ZygoteConnection。  
```
done = peers.get(index).runOnce();
```  
**Step 6. ZygoteConnection.runOnce**  
　从Socket读取相关数据之后创建新的进程  
```
pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid,
	parsedArgs.gids, parsedArgs.debugFlags, rlimits);
```  
**Step 7. ZygoteConnection.handleChildProc**  
　要为新创建的进程初始化运行时库，调用RuntimeInit.zygoteInit进一步处理。
**Step 8. RuntimeInit.zygoteInit**  
　这里有两个关键的函数调用，一个是zygoteInitNative函数调用，一个是invokeStaticMain函数调用，前者就是执行Binder驱动程序初始化的相关工作了，正是由于执行了这个工作，才使得进程中的Binder对象能够顺利地进行Binder进程间通信，而后一个函数调用，就是执行进程的入口函数，这里就是执行startClass类的main函数了，而这个startClass即是我们在Step 1中传进来的"android.app.ActivityThread"值，表示要执行android.app.ActivityThread类的main函数。  
　我们先来看一下zygoteInitNative函数的调用过程，然后再回到RuntimeInit.zygoteInit函数中来，看看它是如何调用android.app.ActivityThread类的main函数的。  
**step 9. RuntimeInit.zygoteInitNative**
　com_android_internal_os_RuntimeInit_zygoteInit函数（zygoteInitNative的本地实现方法），实际是执行了AppRuntime类的onZygoteInit函数。   
**Step 10. AppRuntime.onZygoteInit**  
　这里它就是调用ProcessState::startThreadPool启动线程池了，这个线程池中的线程就是用来和Binder驱动程序进行交互的了。  
**Step 11. ProcessState.startThreadPool**  
**Step 12. ProcessState.spawnPooledThread**  
　 这里它会创建一个PoolThread线程类，然后执行它的run函数，最终就会执行PoolThread类的threadLoop函数了。      　　
  
  **Step 13. PoolThread.threadLoop**  
   　这里它执行了IPCThreadState::joinThreadPool函数进一步处理。IPCThreadState也是Binder进程间通信机制的一个基础组件。    
**Step 14. IPCThreadState.joinThreadPool**  
　这个函数首先告诉Binder驱动程序，这条线程要进入循环了
```
mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);
```  
  然后在中间的while循环中通过talkWithDriver不断与Binder驱动程序进行交互，以便获得Client端的进程间调用：
```
result = talkWithDriver();
```
获得了Client端的进程间调用后，就调用excuteCommand函数来处理这个请求：
```
result = executeCommand(cmd);
```
最后，线程退出时，也会告诉Binder驱动程序，它退出了，这样Binder驱动程序就不会再在Client端的进程间调用分发给它了：
```
mOut.writeInt32(BC_EXIT_LOOPER);
talkWithDriver(false);
```  
**Step 15. talkWithDriver**    
　它只要就是通过ioctl文件操作函数来和Binder驱动程序交互的了：  
```
ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)
```  
>　有了这个线程池之后，我们在开发Android应用程序的时候，当我们要和其它进程中进行通信时，只要定义自己的Binder对象，然后把这个Binder对象的远程接口通过其它途径传给其它进程后，其它进程就可以通过这个Binder对象的远程接口来调用我们的应用程序进程的函数了，它不像我们在C++层实现Binder进程间通信机制的Server时，必须要手动调用IPCThreadState.joinThreadPool函数来进入一个无限循环中与Binder驱动程序交互以便获得Client端的请求，这样就实现了我们在文章开头处说的Android应用程序进程天然地支持Binder进程间通信机制。　

　 回到Step 8中的RuntimeInit.zygoteInit函数中，在初始化完成Binder进程间通信机制的基础设施后，它接着就要进入进程的入口函数了。  

**Step 16. RuntimeInit.invokeStaticMain**    
 前面我们说过，这里传进来的参数className字符串值为"android.app.ActivityThread"，这里就通ClassLoader.loadClass函数将它加载到进程中：  
```
cl = loader.loadClass(className);
```  
 然后获得它的静态成员函数main：  
```
m = cl.getMethod("main", new Class[] { String[].class });
```  
　函数最后并没有直接调用这个静态成员函数main，而是通过抛出一个异常ZygoteInit.MethodAndArgsCaller，然后让ZygoteInit.main函数在捕获这个异常的时候再调用android.app.ActivityThread类的main函数。为什么要这样做呢？注释里面已经讲得很清楚了，它是为了清理堆栈的，这样就会让android.app.ActivityThread类的main函数觉得自己是进程的入口函数，而事实上，在执行android.app.ActivityThread类的main函数之前，已经做了大量的工作了。  
　在捕获到异常之后是怎么执行的，通过如下语句：  
```
mMethod.invoke(null, new Object[] { mArgs });
```  
　这里的成员变量mMethod和mArgs都是在前面构造异常对象时传进来的，这里的mMethod就对应android.app.ActivityThread类的main函数了, 这样，android.app.ActivityThread类的main函数就被执行了。  
**Step 17. ActivityThread.main**  
　从这里我们可以看出，这个函数首先会在进程中创建一个ActivityThread对象：  
```
ActivityThread thread = new ActivityThread();
```  
  　 然后进入消息循环中：  
```
Looper.loop();
```  
　这样，我们以后就可以在这个进程中启动Activity或者Service了  
##### 总结
ActivityMangerService通过Socket连接Zygote进程之后，通过Zygote进程fork出一个新的应用进程。当然在fork出的新进程中会执行一些初始的操作：  
1.为进程内的Binder对象提供了Binder进程间通信机制的基础设施  
2.执行进程的入口(也就看起来像是入口，其实在这之前已经做了好多工作)函数是ActivityThread.main
#### 3.3 Activity的启动过程
　经过上面的步骤，应用的进程已经创建了，这节学习如何在应用进程中创建Activity。  
　在手机屏幕中点击应用程序图标的情景就会引发Android应用程序中的默认Activity的启动，从而把应用程序启动起来。这种启动方式的特点是会启动一个新的进程来加载相应的Activity，以这个例子学习。
![](http://img.my.csdn.net/uploads/201108/18/0_1313675675dBp4.gif)
**Step 1. Launcher.startActivitySafely**  
在Android系统中，应用程序是由Launcher启动起来的，其实，Launcher本身也是一个应用程序，其它的应用程序安装后，就会Launcher的界面上出现一个相应的图标，点击这个图标时，Launcher就会对应的应用程序启动起来。  
**Step 2. Activity.startActivity**  
在Step 1中，我们看到，Launcher继承于Activity类，而Activity类实现了startActivity函数，因此，这里就调用了Activity.startActivity函数  
  这个函数实现很简单，它调用startActivityForResult来进一步处理，第二个参数传入-1表示不需要这个Actvity结束后的返回结果。    
**Step 3. Activity.startActivityForResult**  
```
public void startActivityForResult(Intent intent, int requestCode) {
		if (mParent == null) {
			Instrumentation.ActivityResult ar =
				mInstrumentation.execStartActivity(
				this, mMainThread.getApplicationThread(), mToken, this,
				intent, requestCode);
			......
		} else {
			......
		}
```  
这里的mMainThread也是Activity类的成员变量，它的类型是ActivityThread，它代表的是应用程序的主线程.  
这里通过mMainThread.getApplicationThread获得它里面的ApplicationThread成员变量，它是一个Binder对象，后面我们会看到，ActivityManagerService会使用它来和ActivityThread来进行进程间通信。这里我们需注意的是，这里的mMainThread代表的是Launcher应用程序运行的进程。  
这里的mToken也是Activity类的成员变量，它是一个Binder对象的远程接口。
> Intrumentation介绍：   

**Step 4. Instrumentation.execStartActivity**  

```
int result = ActivityManagerNative.getDefault()
				.startActivity(whoThread, intent,
intent.resolveTypeIfNeeded(who.getContentResolver()),
				null, 0, token, target != null ? target.mEmbeddedID : null,
				requestCode, false, false);
```  
里的ActivityManagerNative.getDefault返回ActivityManagerService的远程接口，即ActivityManagerProxy接口。  
**Step 5. ActivityManagerProxy.startActivity**
> 注意：经过这个函数之后，调用就从客户端进入到服务端
  
**Step 6. ActivityManagerService.startActivity**  
上一步Step 5通过Binder驱动程序就进入到ActivityManagerService的startActivity函数来了。  
这里只是简单地将操作转发给成员变量mMainStack的startActivityMayWait函数，这里的mMainStack的类型为ActivityStack。  
**Step 7. ActivityStack.startActivityMayWait**   
  
```
    try {
	ResolveInfo rInfo =
	AppGlobals.getPackageManager().resolveIntent(
		intent, resolvedType,
		PackageManager.MATCH_DEFAULT_ONLY
		| ActivityManagerService.STOCK_PM_FLAGS);
	aInfo = rInfo != null ? rInfo.activityInfo : null;
    } catch (RemoteException e) {
		......
    } 
```  
上面语句对参数intent的内容进行解析，得到要启动的Activity的相关信息，保存在aInfo变量中。  
  接下去就调用startActivityLocked进一步处理了。  
**Step 8. ActivityStack.startActivityLocked**  

```
ActivityRecord r = new ActivityRecord(mService, this, callerApp, callingUid,
	intent, resolvedType, aInfo, mService.mConfiguration,
	resultRecord, resultWho, requestCode, componentSpecified);
```  
创建即将要启动的Activity的相关信息，并保存在r变量中。  
接着调用startActivityUncheckedLocked函数进行下一步操作。  
**Step 9. ActivityStack.startActivityUncheckedLocked**  
在这个函数中进行一些逻辑检查：  
1.查看当前有没有Task可以用来执行这个Activity。由于r.launchMode的值不为ActivityInfo.LAUNCH_SINGLE_INSTANCE，因此，它通过findTaskLocked函数来查找存不存这样的Task，这里返回的结果是null，即taskTop为null，因此，需要创建一个新的Task来启动这个Activity。  
2.查看当前在堆栈顶端的Activity是否就是即将要启动的Activity，有些情况下，如果即将要启动的Activity就在堆栈的顶端，那么，就不会重新启动这个Activity的别一个实例了。  
执行到这里，我们知道，要在一个新的Task里面来启动这个Activity了，于是新创建一个Task。 
 
```
r.task = new TaskRecord(mService.mCurTask, r.info, intent,
		(r.info.flags&ActivityInfo.FLAG_CLEAR_TASK_ON_LAUNCH) != 0);
```  
**Step 10. Activity.resumeTopActivityLocked**  
把当处于Resumed状态的Activity推入Paused状态，然后才可以启动新的Activity。但是在将当前这个Resumed状态的Activity推入Paused状态之前，首先要看一下当前是否有Activity正在进入Pausing状态，如果有的话，当前这个Resumed状态的Activity就要稍后才能进入Paused状态了，这样就保证了所有需要进入Paused状态的Activity串行处理。(这个例子中需要把Launcher的activity放入pauesd状态)  
接着调用startPausingLocked函数把Launcher推入Paused状态去了。
**Step 11. ActivityStack.startPausingLocked**  
把Launcher进程中的ApplicationThread对象取出来，通过它来通知Launcher这个Activity它要进入Paused状态了。当然，这里的prev.app.thread是一个ApplicationThread对象的远程接口，通过调用这个远程接口的schedulePauseActivity来通知Launcher进入Paused状态。  
**Step 12. ApplicationThreadProxy.schedulePauseActivity**  
这个函数通过Binder进程间通信机制进入到ApplicationThread.schedulePauseActivity函数中。  
**Step 13. ApplicationThread.schedulePauseActivity**  
ApplicationThread是ActivityThread的内部类，在这个方法里面调用的函数queueOrSendMessage是ActivityThread类的成员函数。   
**Step 14. ActivityThread.queueOrSendMessage**  
 这里首先将相关（需要暂停的activity的信息）信息组装成一个msg，然后通过mH成员变量发送出去，mH的类型是H，继承于Handler类，是ActivityThread的内部类，因此，这个消息最后由H.handleMessage来处理。  
**Step 15. H.handleMessage**  
这里调用ActivityThread.handlePauseActivity进一步操作，msg.obj是一个ActivityRecord对象的引用，它代表的是Launcher这个Activity。

```
public void handleMessage(Message msg) {  
            ......  
            switch (msg.what) {  
              
            ......  
              
            case PAUSE_ACTIVITY:  
                handlePauseActivity((IBinder)msg.obj, false, msg.arg1 != 0, msg.arg2);  
                maybeSnapshot();  
                break;  
  
            ......  
  
            }  
```    
**Step 16. ActivityThread.handlePauseActivity**   
在这个方法里面做的事情：   
1.调用performPauseActivity函数来调用Activity.onPause函数，我们知道，在Activity的生命周期中，当它要让位于其它的Activity时，系统就会调用它的onPause函数；  
2.它通知ActivityManagerService，这个Activity已经进入Paused状态了，ActivityManagerService现在可以完成未竟的事情，即启动MainActivity了。

```
public final class ActivityThread {
	
	......

	private final void handlePauseActivity(IBinder token, boolean finished,
			boolean userLeaving, int configChanges) {

		ActivityClientRecord r = mActivities.get(token);
		if (r != null) {
			//Slog.v(TAG, "userLeaving=" + userLeaving + " handling pause of " + r);
			if (userLeaving) {
				performUserLeavingActivity(r);
			}

			r.activity.mConfigChangeFlags |= configChanges;
			//调用Activity的onPause生命周期方法
			Bundle state = performPauseActivity(token, finished, true);

			// Make sure any pending writes are now committed.
			QueuedWork.waitToFinish();
          //通知ActivityManagerService已经暂停了
			// Tell the activity manager we have paused.
			try {
				ActivityManagerNative.getDefault().activityPaused(token, state);
			} catch (RemoteException ex) {
			}
		}
	}

	......

}
```  
**Step 17. ActivityManagerProxy.activityPaused**  
这里通过Binder进程间通信机制就进入到ActivityManagerService.activityPaused函数中去了。  
**Step 18. ActivityManagerService.activityPaused**  
这里，又再次进入到ActivityStack类中，执行activityPaused函数。  
**Step 19. ActivityStack.activityPaused**  
对Launcher执行completePauseLocked操作。  
**Step 20. ActivityStack.completePauseLocked**  
调用resumeTopActivityLokced进一步操作，它传入的参数即为代表Launcher这个Activity的ActivityRecord。  
**Step 21. ActivityStack.resumeTopActivityLokced**  
通过上面的Step 9，我们知道，当前在堆栈顶端的Activity为我们即将要启动的MainActivity，这里通过调用topRunningActivityLocked将它取回来，保存在next变量中。之前最后一个Resumed状态的Activity，即Launcher，到了这里已经处于Paused状态了，因此，mResumedActivity为null。最后一个处于Paused状态的Activity为Launcher，因此，这里的mLastPausedActivity就为Launcher。前面我们为MainActivity创建了ActivityRecord后，它的app域一直保持为null。有了这些信息后，上面这段代码就容易理解了，它最终调用startSpecificActivityLocked进行下一步操作。  
**Step 22. ActivityStack.startSpecificActivityLocked**  
注意，这里由于是第一次启动应用程序的Activity，所以下面语句：
  
```
ProcessRecord app = mService.getProcessRecordLocked(r.processName,
	r.info.applicationInfo.uid);
```
  取回来的app为null。在Activity应用程序中的AndroidManifest.xml配置文件中，我们没有指定Application标签的process属性，系统就会默认使用package的名称，这里就是"shy.luo.activity"了。每一个应用程序都有自己的uid，因此，这里uid + process的组合就可以为每一个应用程序创建一个ProcessRecord。当然，我们可以配置两个应用程序具有相同的uid和package，或者在AndroidManifest.xml配置文件的application标签或者activity标签中显式指定相同的process属性值，这样，不同的应用程序也可以在同一个进程中启动。
       函数最终执行ActivityManagerService.startProcessLocked函数进行下一步操作。  
**Step 23. ActivityManagerService.startProcessLocked**  
 这里再次检查是否已经有以process + uid命名的进程存在，在我们这个情景中，返回值app为null，因此，后面会创建一个ProcessRecord 。这里主要是调用Process.start接口来创建一个新的进程，新的进程会导入android.app.ActivityThread类，并且执行它的main函数，这就是为什么我们前面说每一个应用程序都有一个ActivityThread实例来对应的原因。  
**Step 24. ActivityThread.main**   
 这个函数在进程中创建一个ActivityThread实例，然后调用它的attach函数，接着就进入消息循环了，直到最后进程退出。
       函数attach最终调用了ActivityManagerService的远程接口ActivityManagerProxy的attachApplication函数，传入的参数是mAppThread，这是一个ApplicationThread类型的Binder对象，它的作用是用来进行进程间通信的。  

```
public final class ActivityThread {
	......
	private final void attach(boolean system) {
		......
		mSystemThread = system;
		if (!system) {
			......
			IActivityManager mgr = ActivityManagerNative.getDefault();
			try {
				mgr.attachApplication(mAppThread);
			} catch (RemoteException ex) {
			}
		} else {
			......
		}
	}
	......
	public static final void main(String[] args) {	
		.......
		ActivityThread thread = new ActivityThread();
		thread.attach(false);
		......
		Looper.loop();
		.......
		thread.detach();
		......
	}
}
```  
**Step 25. ActivityManagerProxy.attachApplication**  
 这里通过Binder驱动程序，最后进入ActivityManagerService的attachApplication函数中。  
**Step 26. ActivityManagerService.attachApplication**  
这里将操作转发给attachApplicationLocked函数。  
**Step 27. ActivityManagerService.attachApplicationLocked**  
 在前面的Step 23中，已经创建了一个ProcessRecord，这里首先通过pid将它取回来，放在app变量中，然后对app的其它成员进行初始化，最后调用mMainStack.realStartActivityLocked执行真正的Activity启动操作。这里要启动的Activity通过调用mMainStack.topRunningActivityLocked(null)从堆栈顶端取回来（取回来的是ActivityRecord，记录着要启动的Activity的相关信息），这时候在堆栈顶端的Activity就是MainActivity了。  
**Step 28. ActivityStack.realStartActivityLocked**  
这里最终通过app.thread进入到ApplicationThreadProxy的scheduleLaunchActivity函数中，注意，这里的第二个参数r，是一个ActivityRecord类型的Binder对象，用来作来这个Activity的token值。  
**Step 29. ApplicationThreadProxy.scheduleLaunchActivity**  
 这个函数最终通过Binder驱动程序进入到ApplicationThread的scheduleLaunchActivity函数中。  
 **Step 30. ApplicationThread.scheduleLaunchActivity**  
  函数首先创建一个ActivityClientRecord实例，并且初始化它的成员变量，然后调用ActivityThread类的queueOrSendMessage函数进一步处理。  
**Step 31. ActivityThread.queueOrSendMessage**  
 函数把消息内容放在msg中，然后通过mH把消息分发出去，这里的成员变量mH我们在前面已经见过，消息分发出去后，最后会调用H类的handleMessage函数。  
**Step 32. H.handleMessage**  
这里最后调用ActivityThread类的handleLaunchActivity函数进一步处理  
**Step 33. ActivityThread.handleLaunchActivity**  
这里首先调用performLaunchActivity函数来加载这个Activity类，即shy.luo.activity.MainActivity，然后调用它的onCreate函数，最后回到handleLaunchActivity函数时，再调用handleResumeActivity函数来使这个Activity进入Resumed状态，即会调用这个Activity的onResume函数，这是遵循Activity的生命周期的。  
**Step 34. ActivityThread.performLaunchActivity**  

```
public final class ActivityThread {

	......

	private final Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
		//收集要启动的Activity的相关信息，主要package和component信息：
		ActivityInfo aInfo = r.activityInfo;
		if (r.packageInfo == null) {
			r.packageInfo = getPackageInfo(aInfo.applicationInfo,
				Context.CONTEXT_INCLUDE_CODE);
		}

		ComponentName component = r.intent.getComponent();
		if (component == null) {
			component = r.intent.resolveActivity(
				mInitialApplication.getPackageManager());
			r.intent.setComponent(component);
		}

		if (r.activityInfo.targetActivity != null) {
			component = new ComponentName(r.activityInfo.packageName,
				r.activityInfo.targetActivity);
		}
// 然后通过ClassLoader将要启动的MainActivity类加载进来
		Activity activity = null;
		try {
			java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
			activity = mInstrumentation.newActivity(
				cl, component.getClassName(), r.intent);
			r.intent.setExtrasClassLoader(cl);
			if (r.state != null) {
				r.state.setClassLoader(cl);
			}
		} catch (Exception e) {
			......
		}

		try {
		//接下来是创建Application对象，这是根据AndroidManifest.xml配置文件中的Application标签的信息来创建的
			Application app = r.packageInfo.makeApplication(false, mInstrumentation);

			......

			if (activity != null) {
				ContextImpl appContext = new ContextImpl();
				appContext.init(r.packageInfo, r.token, this);
				appContext.setOuterContext(activity);
				CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
				Configuration config = new Configuration(mConfiguration);
				//创建Activity的上下文信息，并通过attach方法将这些上下文信息设置到MainActivity中去
				......
				activity.attach(appContext, this, getInstrumentation(), r.token,
					r.ident, app, r.intent, r.activityInfo, title, r.parent,
					r.embeddedID, r.lastNonConfigurationInstance,
					r.lastNonConfigurationChildInstances, config);

				if (customIntent != null) {
					activity.mIntent = customIntent;
				}
				r.lastNonConfigurationInstance = null;
				r.lastNonConfigurationChildInstances = null;
				activity.mStartedActivity = false;
				int theme = r.activityInfo.getThemeResource();
				if (theme != 0) {
					activity.setTheme(theme);
				}

				activity.mCalled = false;
				mInstrumentation.callActivityOnCreate(activity, r.state);
				......
				r.activity = activity;
				r.stopped = true;
				if (!r.activity.mFinished) {
					activity.performStart();
					r.stopped = false;
				}
				if (!r.activity.mFinished) {
					if (r.state != null) {
						mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
					}
				}
				if (!r.activity.mFinished) {
					activity.mCalled = false;
					//最后还要调用MainActivity的onCreate函数：
					mInstrumentation.callActivityOnPostCreate(activity, r.state);
					if (!activity.mCalled) {
						throw new SuperNotCalledException(
							"Activity " + r.intent.getComponent().toShortString() +
							" did not call through to super.onPostCreate()");
					}
				}
			}
			r.paused = true;

			mActivities.put(r.token, r);

		} catch (SuperNotCalledException e) {
			......

		} catch (Exception e) {
			......
		}

		return activity;
	}

	......
}
```  
这里不是直接调用MainActivity的onCreate函数，而是通过mInstrumentation的callActivityOnCreate函数来间接调用，前面我们说过，mInstrumentation在这里的作用是监控Activity与系统的交互操作，相当于是系统运行日志。  
**Step 35. MainActivity.onCreate**  
这样，MainActivity就启动起来了，整个应用程序也启动起来了。
##### 总结  
用户点击Launcher（Launcher也是Activity）上的图标，在Activity中通过调用ActivityMangerService的远程接口告诉AMS要启动一个Activity，然后转到服务端，AMS又会委托ActivityStack去做。ActivityStack首先会检查是否要创建新的Activity和Task，如果是要创建，那么就会把Launcher进程中的ApplicationThread取出来，人啊后通过它告诉Launcher这个Activity要暂停。通过Binder类型的ApplicationThread，这时从系统服务端的进程转到了Launcher所在的进程。然后ApplicationThread使用ActivityTread中的H发一个消息，在消息的处理的时候，又由ActivityTread实际去暂停Activity（其实是通过Instrumction去调用Activity的onPause的）。当暂停完成得时候，又会通过AMS的代理向AMS发一个暂停完成得消息（这时又从Launcher进程转到系统的服务进程）。然后AMS又会委托ActivityStack去启动一个Activity，由于是第一次启动，这时会使用Process创建一个进程。新进程创建完成之后会绑定应用到ActivityManagerService，然后ActivityThread的Handler处理启动Activity的消息。

**对比学习**

#### 应用进程绑定到ActivityManagerService  

![](http://www.cloudchou.com/wp-content/uploads/2015/05/application_amservice.png)  
将应用进程的ApplicationThread对象绑定到ActivityManagerService，也就是说获得ApplicationThread对象的代理对象。	  

```
//ActivityThread类
private void handleBindApplication(AppBindData data) {
  //...  
  ApplicationInfo instrApp = new ApplicationInfo();
  instrApp.packageName = ii.packageName;
  instrApp.sourceDir = ii.sourceDir;
  instrApp.publicSourceDir = ii.publicSourceDir;
  instrApp.dataDir = ii.dataDir;
  instrApp.nativeLibraryDir = ii.nativeLibraryDir;
  LoadedApk pi = getPackageInfo(instrApp, data.compatInfo,
        appContext.getClassLoader(), false, true);
  ContextImpl instrContext = new ContextImpl();
  instrContext.init(pi, null, this);
    //... 
  if (data.instrumentationName != null) {
       //...
  } else {
       //注意Activity的所有生命周期方法都会被Instrumentation对象所监控，
       //也就说执行Activity的生命周期方法前后一定会调用Instrumentation对象的相关方法
       //并不是说只有跑单测用例才会建立Instrumentation对象，
       //即使不跑单测也会建立Instrumentation对象
       mInstrumentation = new Instrumentation();
  }
  //... 
  try {
     //...
     Application app = data.info.makeApplication(data.restrictedBackupMode, null);
     mInitialApplication = app;
     //...         
     try {
          mInstrumentation.onCreate(data.instrumentationArgs);
      }catch (Exception e) {
             //...
      }
      try {
           //这里会调用Application的onCreate方法
           //故此Applcation对象的onCreate方法会比ActivityThread的main方法后调用
           //但是会比这个应用的所有activity先调用
            mInstrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
           //...
        }
    } finally {
        StrictMode.setThreadPolicy(savedPolicy);
    }
}
```
#### ActivityThread的Handler处理启动Activity的消息  

![](http://www.cloudchou.com/wp-content/uploads/2015/05/activitythread_activity.png)

参考：<a href=http://ju.outofmemory.cn/entry/169878">http://ju.outofmemory.cn/entry/169878</a>
<a href=http://blog.csdn.net/luoshengyang/article/details/6689748">http://blog.csdn.net/luoshengyang/article/details/6689748</a>
 <a href=http://blog.csdn.net/tenggangren/article/details/50925740">http://blog.csdn.net/tenggangren/article/details/50925740</a>

 




   









　  
　
        　         


