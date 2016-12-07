### Android中的亲和性(Affinity)
> <small>理解知识的步骤：1.是什么？2.有什么用？3.值哪里来？4.注意什么?

#### 1.Affinity是什么？
Affinity是Activity的一个属性
#### 2.Affinity有什么用？
affinity对Activity来说，就像是身份证一样，可以告诉所在的Task，自己属于其中的一员。（即Affinity用来表示Activity应该属于哪个Task）
> 有一个例子可以说明它的作用：当开始一个没有Intent.FLAG_ACTIVITY_NEW_TASK标志的Activity时亲和性用Affinitys不会影响将会运行该新活动的Task:它总是运行在启动它的Task里。但是，如果使用了NEW_TASK标志，那么共用性（affinity）将被用来判断是否已经存在一个有相同共用性（affinity）的Task。如果是这样，这项Task将被切换到前面而新的Activity会启动于这个Task的顶层。

#### 3.值哪里来？
默认情况下一个应用的所有Activity都是具有相同的affinity，都是从application中继承，application的affinity默认就是manifest的包名。
> 注意：   
> 1.任务（Task）不仅可以跨应用（Application），还可以跨进程（Process）。就是说两个不同的进程的应用程序，可以使用同一个Task。   
> 2. Actvity的affinity是由taskAffinity属性定义的。Task的affinity是通过读取根Activity的affinity决定。因此，根Activity总是位于相同affinity的Task里。

#### 4.知识扩展：跟 Task 有关的 manifest文件中Activity的特性值介绍
##### 4.1 android:allowTaskReparenting
用来标记Activity能否从启动的Task移动到有着相同affinity的Task，rue”，表示mo'ren能移动，“false”，表示它必须呆在启动时呆在的那个Task里。设置成true后，需要与taskAffinity属性配合使用，当原来的task转到后台，亲属task转到前台，那么此时activity就会属于当前的task。  
<font color='#008B8B'>
例子：
如果 email中包含一个web页的链接，点击它就会启动一个Activity来显示这个页面。这个Activity是由Browser应用程序定义的，但是，现在它作为email Task的一部分。如果它重新宿主到Browser Task里，当Browser下一次进入到前台时，它就能被看见，并且，当email Task再次进入前台时，就看不到它了。 
</font>
##### 4.2 android:alwaysRetainTaskState 
总是保留task的状态。默认值是false。此时会遵循系统的清理栈的行为。如果设置为true，系统会保留栈中所有activity及其状态，当用户再次回来的时候，发现一切如初。   
<font color='#008B8B'>
例子：
当用户从主画面重新选择这个Task时，系统会对这个Task进行清理（从stack中删除位于根Activity之上的所有Activivity）。典型的情况，当用户有一段时间没有访问这个Task时也会这么做，例如30分钟。当此属性设置为true的时候，就不会清理完整的保存着。 
</font>
##### 4.3 android:clearTaskOnLaunch 
当再次启动task时，系统会清理掉除了根activity以外的所有activity。默认值是false。
##### 4.4 android:finishOnTaskLaunch 
这个属性与上面的clearTaskOnLaunch很像，不过它是指单个activity的，而不是整个栈。当设置为true时，task重启后，这个activity就不显示了（finish结束了）。默认值为false。优先级高于allowTaskReparenting。
##### 4.5 android:noHistory 
如果设置true，当离开activity并不可见时，此activity会从栈中移除并不留下记录
##### 4.6 android:taskAffinity
这个就是上面讲的亲和性