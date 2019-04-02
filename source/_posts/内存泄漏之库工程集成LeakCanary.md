---

title: 内存泄漏之库工程集成LeakCanary
date: 2019-04-02 21:02:03
tags: 内存泄漏

---


业务项目中采用模块化的架构，需要在每个模块集成LeakCanary，这样可以在模块的开发调试阶段发现内存泄漏问题，比较好的做法是把 
LeakCanary放到基础的库工程中。

## 问题

1. 库工程默认只会发布release包，官网的集成依赖方式是release采用no-op空实现，所以需要对库工程做处理
2. LeakCanary的泄漏分析是在子进程中，app依赖库工程后，需要避免application重复初始化

## 解决方案
#### 问题1：
有两种解决办法
1、第一种可以采用官网的依赖方式。由于releaseImplementation是空实现，而库工程默认只会发布release包，导致leakcanary失效。

```gradle
dependencies {
  debugImplementation 'com.squareup.leakcanary:leakcanary-android:1.6.3'
  releaseImplementation 'com.squareup.leakcanary:leakcanary-android-no-op:1.6.3'
}

```

我们需要对库工程打包多个变种，设置publishNonDefault可以生成所有变种包。这样app就可以区分不同的构建类型进行依赖。

```gradle

apply plugin: 'com.android.library'
android {
    ...
    
    publishNonDefault true
}
```
app依赖方式如下，让app工程在debug版本下依赖库工程debug版本，在release版本下依赖库工程release版本
```gradle
dependencies {
    debugImplementation  project(path: ':myLibrary', configuration: 'debug')
    releaseImplementation project(path: ':myLibrary', configuration: 'release')
}
```

2、第二种方法，使用依赖如下：
```gradle
dependencies {
  implementation 'com.squareup.leakcanary:leakcanary-android:1.6.3'
}

```
基础库工程抽离一个BaseApplication做初始化的公共操作。如果release版本则不初始化LeakCanary，代替了第一种方法中releaseImplementation的空实现方式。需要注意的是，库工程BuildConfig.debug一直为false，需要在app工程中判断是否调试版本。项目工程采用了这种方法。

```kotlin
abstract class BaseApplication : Application() {

    companion object {
        private val TAG = this::class.java.simpleName
    }

    override fun onCreate() {
        super.onCreate()

        if (isDebug()) {
            if (LeakCanary.isInAnalyzerProcess(this)) {
                // This process is dedicated to LeakCanary for heap analysis.
                // You should not init your app in this process.
                logd(TAG, "LeakCanary process is starting for heap analysis")
                return
            }
            LeakCanary.refWatcher(this)
                    .listenerServiceClass(LeakService::class.java)
                    .buildAndInstall()
            logd(TAG, "LeakCanary installed")
        }

        // Normal app init code...     
    }
    
      /**
     * 是否为调试版本，需要在app工程判断
     */
    abstract fun isDebug(): Boolean
}        
```

#### 问题2：
由于LeakCanary堆分析是另起子进程进行的，会重新触发application的onCreate。需要记得在子进程中return掉以避免重复初始化。
```kotlin

 if (LeakCanary.isInAnalyzerProcess(this)) {
      // This process is dedicated to LeakCanary for heap analysis.
      // You should not init your app in this process.
      return;
 }
```
我们想把LeakCanary的逻辑放到基础库工程，app工程不需要关心这些操作。所以设计如下，

```kotlin
abstract class BaseApplication : Application() {

    companion object {
        private val TAG = this::class.java.simpleName
    }

    final override fun onCreate() {
        super.onCreate()

        if (isDebug()) {
            if (LeakCanary.isInAnalyzerProcess(this)) {
                // This process is dedicated to LeakCanary for heap analysis.
                // You should not init your app in this process.
                logd(TAG, "LeakCanary process is starting for heap analysis")
                return
            }
            LeakCanary.refWatcher(this)
                    .listenerServiceClass(LeakService::class.java)
                    .buildAndInstall()
            logd(TAG, "LeakCanary installed")
        }

        //公共初始化...

        //子类application初始化，为了避免在LeakCanary子进程中重复初始化
        onAppCreate()
    }


    /**
     * 子类application初始化
     */
    abstract fun onAppCreate()

    /**
     * 是否为调试版本，需要在app工程判断
     */
    abstract fun isDebug(): Boolean

}

```

在app工程中继承BaseApplication

```kotlin
class Application : BaseApplication {

    override fun onAppCreate() {
        //no-op
    }

    override fun isDebug(): Boolean {
        return BuildConfig.DEBUG;
    }
}

```

以上，是库工程集成LeakCanary的个人实现方案。有更好的设计可以留言交流。