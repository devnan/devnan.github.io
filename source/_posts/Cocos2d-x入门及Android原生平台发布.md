---
title: Cocos2d-x入门及Android原生平台发布
date: '2018/1/26 20:46:25'
abbrlink: '9e519237'
---

![](https://cdn.jsdelivr.net/gh/devnan/pic/blog/20200126225352.jpg)
<!-- more -->

# 0、前言
本文简单介绍了Cocos2d-x的基本概念以及Cocos Creator开发环境搭建和入门，最后是Android平台的发布和打包。

# 1、Cocos2d-x 是什么
Cocos2d-x 是一款开源的跨平台游戏引擎，支持C++，Lua和JavaScript语言；Cocos2d 有很多个分支，蓝色区域是目前的主流解决方案。
![](![](https://raw.githubusercontent.com/devnan/pic/master/CocosFamily-20190331225612.png)

---

## 1-1、Cocos2d与Unity3d

Unity3d：

- 适合3d游戏，大型2d游戏
- Unity3d引擎更强大，可以做更多事件，比如VR，AR
- 要收费

Cocos2d：

- 适合小型2d游戏
- 相比Unity3d速度快，安装包小
- 免费开源
- 可能会踩坑较多

---

## 1-2、其他游戏引擎

- Libgdx
- AndEngine

- Unreal
- Corona
......
---


## 1-3、Cocos2d-x与OpenGL ES、DirectX
1）Cocos2d-x 的图形渲染基于OpenGL ES   
2）Cocos2d-x 还包含其他如资源管理，音频播放、物理引擎等模块，比直接使用 OpenGL ES 开发游戏，更加迅速简单。让开发者将精力集中在游戏本身，而不是底层的图像绘制上。   
3）DirectX是Windows平台图形编程接口，Cocos2d-x通过AngleProject间接支持windows平台，AngleProject的作用是将OpenGL ES转化为DirectX 

所以，Cocos2d-x的跨平台特性其实也就是借助了OpenGL ES的跨平台性，从而横跨iOS, Android, WP8三个主要平台。

---

# 2、开发环境
#### Cocos Creator：
类似于 Unity3D 的游戏编辑器，并且内部已经包含完整的 JavaScript 引擎和 cocos2d-x 原生引擎
#### VS Code (WebStorm、SublimeText):
JavaScript脚本代码编辑器

___


下图是mac平台显示Cocos Creator包内容的目录：

![](https://raw.githubusercontent.com/devnan/pic/master/20190331225933.png)
---

#### Android原生平台配置：

![](https://raw.githubusercontent.com/devnan/pic/master/20190331230052.png)

注：js引擎和cocos2d-x引擎在Cocos Creator已经自带了，不用我们配置。另外，使用android studio也要配置ANT路径。

---

# 3、关于Cocos Creator

- 组件化
- 脚本化
- 数据驱动
- 跨平台发布

---

Cocos Creator 的技术架构图

![](https://raw.githubusercontent.com/devnan/pic/master/20190331232557.png)

---

## 3-1、组件化概念
Cocos Creator 工作流以组件化开发为核心，也就是基于组件的实体系统开发模式 (Entity-Component System)。简单的说，就是以组合而非继承的方式进行实体的构建。

![](https://raw.githubusercontent.com/devnan/pic/master/20190401000534.jpeg)

一个实体指的是存在于游戏世界中的物体，多个组件可以挂载到实体上，比如动画组件，碰撞组件，渲染组件等。

---

## 3-2、数据驱动代替代码驱动

- 所有场景都会序列化为数据，在运行时使用这些数据来重新构建场景，界面，动画等元素。
- 数据驱动实现了场景的可视化，使得场景可以被自由的编辑。从而使入口点变成了编辑器，而不是代码。

---

# 4、脚本组件开发

## 4-1、基本概念

- 导演和场景
场景相当于一部电影的某个场景画面，导演就负责画面的切换。

- 节点和组件
把功能点设计封装成组件(Component)的形式，然后将这些组件，按需挂载到一个个类似于容器的节点上，从而形成一个个功能各异的实体(Entity)

---

## 4-2、简单的脚本示例


```
cc.Class({
    extends: cc.Component,
    
    //属性声明，会展示在属性检查器中
    properties: {
        username: "Devnan", //基本JavaScript类型, string
        age: 18, //基本JavaScript类型, number
        girlFriend: cc.Node //cc类型，cc.Node用于获取其他节点(实体)
    },
    
    //构造函数
    ctor: function () {
    }

    // 首次加载，做初始化操作
    onLoad: function () {
    },

    // 每一帧渲染都会调用此函数进行更新
    update: function (dt) {
    },
});
```

---

## 4-3、生命周期

- onLoad：组件首次激活时触发，在 start 调用前执行，一般做初始化操作
- start：第一次执行 update 前触发，初始化一些可能在update时发生改变的中间状态
- update：每一帧渲染前触发，做更新操作
- lateUpdate：update 执行完之后触发，做动画更新后的一些额外操作
- onDestroy：当组件或者所在节点调用了destroy()，则会调用此回调，并做回收操作
......

---

## 4-4、事件分发机制

- node.on(type, callback, target)：持续监听 node 的 type 事件。
- node.once(type, callback, target)：监听一次 node 的 type 事件。
- node.off(type, callback, target)：取消监听所有 type 事件或取消 type 的某个监听器（用 callback 和 target 指定）。
- node.emit(type, detail)：通知所有监听 type 事件的监听器，可以发送一个附加参数。
- node.dispatchEvent(event)：发送一个事件给它的监听器，支持冒泡。


---

**node.dispatchEvent** 采用冒泡派送的方式分发事件。冒泡派送会将事件从事件发起节点，不断地向上传递给他的父级节点，直到到达根节点或者在某个节点通过event.stopPropagation() 拦截了此事件。

**触摸事件**也支持节点树的事件冒泡。

注：和android的事件分发机制有差别，cocos2d没有事件捕获的过程，而且消费事件后需要主动去拦截事件传递

---

# 5、其他组件

- 物理组件
- 动画组件
......

---

# 6、构建发布流程

- link、default：使用源码引擎构建，速度慢

- binary：使用预编译库构建，无需编译C++，速度快，但是无法在原生工程里调试引擎源码

---
![](https://raw.githubusercontent.com/devnan/pic/master/20190401000622.png)

---

下面目录可以拿到构建编译后android平台下的apk

![](https://raw.githubusercontent.com/devnan/pic/master/20190401000659.png)

---

这样是不是都搞定了？可以开心地玩游戏了

![](https://raw.githubusercontent.com/devnan/pic/master/20190401000725.jpg)

---

最后还要导入到原生平台Android Studio进行开发，以修改一些默认配置，比如应用图标，混淆开关等
![](https://raw.githubusercontent.com/devnan/pic/master/20190401000758.png)


这里可以用脚本处理整个流程，从构建编译，导出原生平台，到部分配置项的修改，以加快打包速度。



可以在工程主目录新建android-build文件夹，builder.json是构建编译的配置文件，build.sh是整个流程的脚本，如下，

```
#!/usr/bin/env bash
PROJECT_PATH=$(dirname `pwd`)

# 构建编译
buildCompile(){
    echo "PROJECT_PATH:" $PROJECT_PATH
    /Applications/CocosCreator.app/Contents/MacOS/CocosCreator --path $PROJECT_PATH --build "configPath=$PROJECT_PATH/android-build/builder.json;autoCompile=true"
}
 
handleAndroidProject() {
    # 适配环境
    echo "android.enableAapt2=false" >> $ANDROID_PATH"/gradle.properties"
    
    # 替换资源
    ANDROID_PATH=$PROJECT_PATH"/build/jsb-binary/frameworks/runtime-src/proj.android-studio"
    RES_PATH=${ANDROID_PATH}"/app/res"
    echo "ANDROID_PATH:" $ANDROID_PATH

    echo "replace $RES_PATH/values/strings.xml"
    rm $RES_PATH"/values/strings.xml"
    cp ./res/strings.xml $RES_PATH"/values/"

    echo "replace $RES_PATH/mipmap-xhdpi/ic_launcher.png"
    rm $RES_PATH"/mipmap-xhdpi/ic_launcher.png"
    cp ./res/ic_launcher.png $RES_PATH"/mipmap-xhdpi/"
    rm $RES_PATH"/mipmap-xxhdpi/ic_launcher.png"
    cp ./res/ic_launcher.png $RES_PATH"/mipmap-xxhdpi/"
}


# 重新打包
generateAPK(){
    cd $ANDROID_PATH/app/
    gradle clean assembleRelease

    mkdir $PROJECT_PATH/android-build/apk
    cp $ANDROID_PATH/app/build/outputs/apk/release/*.apk $PROJECT_PATH/android-build/apk
    echo "regenerate apk in $PROJECT_PATH/android-build/apk"
}

# 执行流程
buildCompile
handleAndroidProject
generateAPK

```

---


# 7、推荐资料

- Cocos Creator v1.8.x 用户手册
http://docs.cocos.com/creator/manual/zh/
- Cocos Creator 之旅：
http://www.cocos.com/607




