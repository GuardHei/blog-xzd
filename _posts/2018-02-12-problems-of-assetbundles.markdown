---
layout: post
title: Unity AssetBundle使用的几个坑
comments: true
date: 2018-02-12 07:50:48.000000000 +09:00
author: 谢祖地
tags: Unity
---
## 概述

为了实现热更新，我在最新的项目里使用了AssetBundle， 每次初始化资源的时候从本地AssetBundle文件了动态加载资源并设置默认值，里面有一些坑要注意一下，在此记录，以免后面忘了

### 资源重复加载的问题

当游戏在中途进行了一些重新开始或者退到初始化界面的时候，会重新加载AssetBundles。这个时候会报错 “不能重复加载相同的AssetBundle”。我一开始是在资源加载完之后加一句Unload(); 但是后面发现这样有点问题，Unload后有可能会销毁已经加载的资源，所以要加一个参数Unload(assetbundle, false); 这个false就是指不销毁已加载资源。当然这样会存在内存泄漏，所以在回收资源的时候要记得Destroy(asset); 我现在的项目Assetbundle最大也不会超过10mb，所以还没有担心过这个问题。

### AssetBundle的更新

我不建议用WWW.LoadFromCacheOrDownload()来做，因为这个是异步方法，效率比较低下。我的做法是使用WWW类先将AssetBundle写进内存，然后再写入本地。这会带来一个问题，即如果在写入本地的时候程序被迫中断退出，可能会造成恶劣的影响。我的做法是先写入一个本地的Temper文件。当本地写入完成之后使用File.Move()函数将Temper文件覆盖旧的AssetBundle文件，如果读写过程被打断，旧的文件并没有被错误的覆盖，避免了问题。

当AssetBundle体积变大了之后就要考虑断点续传或者多线程下载的问题。解决方案网上有很多，我就不重复了。这里要注意的就是需要在AssetBundle下完之后提前Load一遍任意一个资源来检测包是否完整，字节流断的是否正确，避免后面加载出问题无处可查。

### AssetBundle的文件夹系统

理论上来说应该是把不同种类的资源分别打成不同的AssetBundle包，但是有的时候项目比较小，或者后端人员开发麻烦，就使用一个AssetBundle来处理所有资源。有时候，需求要根据文件夹内容动态加载菜单或者可选项的时候，由于AssetBundle是并行管理文件的，就不可行了。我的做法是把资源按照对应文件夹写在Resources底下，然后写一个Editor脚本，Pipeline打包的时候，把资源的名称改成文件夹树的形式，用“@”替代“/”作为路径分割符。这样加载资源的时候就可以通过名称来判断归类了。
