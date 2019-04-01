---

title: Android分屏多窗口实现
date: 2016-07-29 09:56:32    
tags: android 

---

# 前言
基于公司产品的要求，需要研究app如何在Android N中支持分屏模式。以下是自己思路的总结，也顺便帮助到各位。
# 概述
**Android N** 的新特性就是对应用分屏的支持。在分屏模式中，系统以左右并排或上下并排的方式分屏显示两个应用。用户可以通过拖动两个应用之间的分界线来改变两个应用的尺寸。

用户可以通过以下方式切换到分屏模式：

 - 打开 Overview 屏幕并长按 Activity 标题，则可以拖动该 Activity 至屏幕突出显示的区域，使Activity 进入多窗口模式。
 - 长按 Overview 按钮，设备上的当前 Activity 将进入分屏模式，同时将打开 Overview 屏幕，可在该屏幕中选择要共享屏幕的另一个 Activity。

另外，用户可以在两个 Activity 共享屏幕的同时在这两个 Activity 之间拖放数据 

# 分屏模式下的生命周期 
首先，分屏模式并没有改变Activity的生命周期。
在分屏模式中，与用户交互的 Activity 为活动状态。另一个 Activity 虽然可见，但处于暂停状态。 如果用户与此暂停的 Activity 交互，该 Activity 将恢复，而之前的Activity 将暂停。所以，在分屏模式下，应用在对用户可见的状态下进入paused状态，可能仍需要继续其操作。就如一个视频播放器，如果进入了分屏模式，不应该在onPaused()中暂停视频播放，而应该在onStop()中才暂停视频，然后对应的在onStart中恢复视频播放。

另外，在Android N的Activity类中，增加了一个void onMultiWindowChanged(boolean
> inMultiWindow)回调，Activity 进入或退出分屏模式时系统将调用此方法。 在 Activity
> 进入分屏模式时，系统向该方法传递 true 值，在退出分屏模式时，则传递 false 值。

#处理运行时变更
在分屏模式显示应用，调整应用大小，或将应用恢复到全屏模式时，系统将通知 Activity 发生配置变更。这与**横竖屏切换**时的 Activity 生命周期影响相同，即Activity会重启(调用onPause–>onStop–>onDestory–>onCreate–>onStart–>onResume),
为了分屏后使Activity恢复原来的状态，一般有两种方法。一是在Activity重启前保存数据；二是阻止Activity的重启。

 - 在Activity重启前保存数据 

1、通过onSaveInstanceState()保存bundle数据,以保证该状态可以在onCreate(Bundle)或者onRestoreInstanceState(Bundle)中恢复
2、重启 Activity需要恢复大量数据、重新建立网络连接等，那么因配置变更而引起的Activity重启会很缓慢，这时可以通过保存 Fragment来减轻重新初始化 Activity的负担，具体做法是，在Activity中包含此Fragment的引用，而Fragment包含要保存的对象的引用；接着，通过在Fragment的onCreate方法中设置setRetainInstance(true)对Fragment进行非中断式的保存；这样，当Activity被销毁重启时，fragment实例还在，可以Activity中恢复了对fragment的引用从而恢复数据。具体可参考[处理运行时变更](https://developer.android.com/guide/topics/resources/runtime-changes.html?hl=zh-cn#RetainingAnObject)。

 - 阻止Activity的重启

在清单文件中声明：

```
    <activity 
       android:name=".MyActivity"
       android:configChanges="orientation|keyboardHidden|screenSize"
       android:label="@string/app_name">
```

android:configChanges属性最常用的值包括 "orientation" ， "keyboardHidden"和"screenSize"，分别用于避免因屏幕方向，可用键盘改变和屏幕尺寸而导致重启。
这种方法不同于第一种，系统不会自动根据当前的配置去应用资源。所以如果需要的话，要在onConfigurationChanged()中自行设置备用资源，即通过setContentView()去调用相应的layout文件。不需要基于这些配置变更去更新应用，则可不用实现onConfigurationChanged()。

注意，第二种方法官方不推荐使用，主要是因为Activity销毁重启不仅仅在分屏模式下会出现，比如手机内存不够时Activity同样可能被销毁，而这时候我们还是需要在Activity销毁前做好数据状态的保存。当然，如果没有备用资源的话，也就是Activity横向和纵向都共用同一个layout文件，则第二种方法无疑是更好的选择。


# 配置分屏模式

在清单的activity或application节点中设置该属性，启用或禁用分屏显示：

    android:resizeableActivity=["true" | "false"]

如果该属性设置为 true，Activity 将能以分屏和自由形状模式启动。 如果此属性设置为 false，Activity 将不支持多窗口模式。如果该值为 false，且用户尝试在分屏模式下启动 Activity，该 Activity 将全屏显示。 

> 如果应用没有适配到Android N（targetSDKVersion < Android
> N，compileSDKVersion < Android N)，也是可以支持分屏的。系统将强制调整应用大小。不过会显示对话框提醒用户应用可能会发生异常。 

另外，在Android N中，可以在清单文件中添加layout节点，并设置一些新增加的属性，通过这些属性来设置分屏模式的行为，如最小尺寸等
例如，以下节点显示了如何指定 Activity 在自由形状模式中显示时 Activity 的默认大小、位置和最小尺寸：

```
    <activity android:name=".MyActivity">
        <layout android:defaultHeight="500dp"
              android:defaultWidth="600dp"
              android:gravity="top|end"
              android:minimalHeight="450dp"
              android:minimalWidth="300dp" />
    </activity>
```

# 参考

https://developer.android.com/preview/features/multi-window.html#configuring
http://blog.csdn.net/airk000/article/details/38557605












