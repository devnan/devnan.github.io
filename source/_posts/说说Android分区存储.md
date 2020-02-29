---
title: 说说Android分区存储
date: '2020-02-23 15:32:19'
tags: Android
abbrlink: b1cb81e6
---

![](https://cdn.jsdelivr.net/gh/devnan/pic/blog/20200223155102.jpg)
<!-- more -->

前几天，Android11预览版出来了。和Android10一样，继续加强权限限制和隐私保护，我们也都看到了scoped storage(本文称为分区存储)这块的变化，即Android11将强制执行分区存储。详见[https://developer.android.com/preview/privacy]( https://developer.android.com/preview/privacy)

![](https://cdn.jsdelivr.net/gh/devnan/pic/blog/20200222213902.png)

分区存储是什么？可能有些开发者还没适配Android10，所以这里简单说一下枯燥的概念，先直接上图。

![](https://cdn.jsdelivr.net/gh/devnan/pic/blog/20200223120131.png)

所谓分区，就是每个App都有一个属于自己的目录，我们称为App-specific目录，App在这个目录可以直接访问文件，无需申明权限。App-specific目录其实包括了两个部分，一个是我们以前所说的App内存目录`/data/data/packageName`，另一个就是App外存目录`/sdcard/Android/data/packageName`。当应用卸载时，App-specific目录的内容也自然会被删除。

除了定义了App-specific目录，分区存储的主要目的其实是限制应用对App-specific目录以外的文件操作。想想Android 10之前，应用的写入权限范围真的太大了，可以随便在sdcard的任何一个目录创建文件，导致文件管理混乱，而且就算应用卸载了，很多文件也遗留在sdcard里占用着存储空间，用户也完全不知道，Android设备经常出现存储空间不够。所以，Android10才大搞分区存储来限制权限，不让开发者这么乱来了。那么应用要在App-specific目录以外创建文件该怎么做？

首先，我们可以把App-specific目录以外的区域称为公共目录。公共目录又分为了多媒体目录和下载目录。在多媒体目录下，应用可以通过MediaStore API来创建多媒体文件（图片、音频、视频）；而在下载目录下，也就是对于其他类型的文件（文档等），应用只能通过Android提供的SAF存储访问框架来创建，用户可以用系统的文件选择器明确地选择想要的存储位置。至于公共目录为什么还要细分，可能是因为不同的文件类型使用场景和访问频率不一样，像多媒体文件比较常用，总不能创建一个图片也要让用户选择位置吧。

理解分区存储的概念以后，我们来看看应用适配分区存储要怎么做。首先要知道，对于很多不遵循存储规范的较大体量应用，适配分区存储会比较难受。所以Android10 beta版本刚出来的时候，开发者都在Android社区反馈适配的复杂性。于是，google爸爸在开发者体验和用户体验中间选择了站在开发者这边，Android10 beta3版本在分区存储上选择了妥协，就是应用可以在Manifest设置`requestLegacyExternalStorage`，以继续使用传统的存储方式，这样就给了开发者们更多的适配时间。目前可以确定的是，Android11会强制执行分区存储，如果使用google的亲儿子系列机型，升级至最新的Android11预览版本，一大波没有适配分区存储的应用都会崩溃。不过离Android11稳定版出来还有一段时间，还没适配分区存储的应用可以开始了。

回到应用适配分区存储怎么做的问题上，我们可以直接在项目代码中全局搜索`Environment.getExternalStorageDirectory()`，看看在sdcard根目录而非App-specific目录下，直接通过File API来操作文件的代码有多少处，基本就可以判断适配分区存储的工作量了。具体的适配细节可以阅读官方文档，参考以下链接：[https://developer.android.com/training/data-storage/files/external-scoped?hl=zh-cn](https://developer.android.com/training/data-storage/files/external-scoped?hl=zh-cn)

另外，开发者需要注意的是，应用适配了Android10，也就是target version升级到29之后，用户升级此应用后，应用还是沿用传统的存储方式。也就是说，分区存储特性只针对Android10上新安装的应用生效。所以，开发的时候记得卸载重装，不然发现不了Android10设备上应用存在的存储问题。

总的来看，分区存储的结果对于用户体验来说是相当直观的，最起码Android根目录会变得非常整洁。但是，个人觉得这件事还是太晚做了，看看iOS从一开始就不允许应用在公共存储里随便放东西，Android直到版本10才把这件事情做了一半。
