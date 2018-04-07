---
layout: post
title: Unity中实现简单的非连续的随时间变化而改变的光照效果
comments: true
date: 2018-04-07 07:50:48.000000000 +09:00
author: William Xie
tags: Unity
---
# 概述
>最近在开发一个移动端项目，涉及到了一天之内场景的光线随时间变化而改变，比如午后的阳光和黄昏的光线就是不一样的。这可以使场景更动态，但是却对机器的机能有了更高的要求。下面我简单的说说我的实现方法。

# 实现

## 最简单的一种实现
最简单的方法我想大家一定都能想到，用代码动态改变`light`组件的`color`属性和`transform`信息，从而实现连续的，真实的动态光照。这种做法的优势是十分明显的，即实现起来简单，而且是连续的，效果相对真实，但是对设备的机能要求比较苛刻，因为是实时渲染光照。作为一个移动端的项目，如果场景稍微大一点的话，帧率下降比较严重，耗电也更快。同时对于一些风格化的游戏来说，光照的色彩改变可能不够理想，不够强调，当然这一点是和真实性相对的。总而言之，这种方法很少会用在实际当中。

## 网上的一些实现
我之前看到网上大部分提出的都是通过动态加载`lightmap`光照贴图的方式来解决问题的，这样的实现除了不能做到连续的变化和包体积增大（想想每次bake完场景那恐怖的gl cache的体积）以外基本算比较合理的了。但是！`Unity`自己挖了不少坑给开发者，动态加载光照贴图听起来不难，但做起来还是有不少弯弯绕的，也要花不少时间。不过这已经是兼顾多个方面对解决方案了。

## 我的实现
前面扯了这么多，赶紧来说说我是怎么实现的吧。首先我们要来考虑一下随着时间的变化，光线是怎么变的？首先物体的影子会移动，其次就是光线的亮度和色彩。然而在世纪的过程中我们会发现，物体影子的方向的改变是不那么容易被察觉的，因为玩家大部分情况下不会一直盯着影子不看，最能直观的体现出时间变化的其实是光线的亮度和色彩。所以我的方法其实就是改变光线的色泽，忽视掉影子的变化。

那么怎么做呢？改变`light`的`color`属性？那不就变成实时渲染了吗？所以我们不能使用这种方式，而是通过改变`Post Processing Stack`中的`lut`属性来处理。

### 什么是Post Processing Stack？
`Post Processing Stack`是新版`Unity`中进行画面处理的插件，需要在`AssetStore`中下载并导入进来，当然，这个是`Unity`自己的产品，所以是免费的（话说原来的`Image Effect`包还要pro版的才能用）。这个插件顾名思义，是对画面进行后处理。什么叫后处理？简单的说就是在把像素即将画到屏幕上的时候在进行处理，是整个渲染管线的末端（就是处理整个画面啦）。通过后处理，我们可以实现包括噪点，雾气，阴影边缘，色阶等等各种效果，非常方便。

### 什么是Lut？
`Lut`即`Look Up Table`，是显示器根据rgb值输出像素点颜色时的参照表，所以如果我们改变`Lut`参照，同样的rgb输入就会显示出不一样的色彩，从而达到动态改变光照颜色的效果。`Lut`本身是一张长条形的`Texture2D`的贴图，请美工出一下就行了，这里就不具体讲怎么做了。

### 配置Post Processing Stack
好的，下面进入正式的操作部分，首先我们下载并导入`Post Processing Stack`这个插件，然后找到当前场景中的`camera`，点击`Add Component`按钮添加`Post Processing Behaviour`组件。这个组件其实就是一个脚本，在`Inspector`上暴露了一个`Profile`的变量。这个变量便是用于配置修改处理效果的了，所以我们还需要在资源目录下创建一个`Post Processing Profile`的配置资产，并把它拖进组件的`Profile`变量上赋值。

这些完成之后，我们就开始配置`Post Processing Profile`了。在`Inspector`上打开这个`Profile`，你会发现有不少的属性，由于这边博文不是介绍`Post Processing Stack`怎么用的，所以我们只关注我们需要的那一个属性。找到`User Lut`这一栏，先钩上左上的圆圈，表示启用`User Lut`，然后展开这个选项，为`Lut`那个属性选择美术给你的`Lut`贴图即可，注意`Lut`贴图的导入选项是`Default`。下面那个`Contribution`属性是这个`Lut`的强度，即影响力。因为我们需要通过`Lut`来改变色彩，所以这个值是1。

OK，目前为止你应该可以看到场景的色彩突然变化了吧，如果没有的话可能是`Lut`变化不是非常明显，需要和美术好好沟通一下呢。

### 动态改变Lut
刚刚我们对`Lut`有了初步的感知，但是有的人会问，现在`Lut`是定死的，那我们怎么在运行时改变它呢？

首先我们先常见一个`Monobehaviour`脚本挂在`camera`上面，就起名叫`EnvironmentController`吧！ `EnvironmentController`代码如下：

```csharp
using UnityEngine;
using UnityEngine.PostProcessing;
using System;
using System.Collections;

public class EnvironmentController : MonoBehaviour {
    // 不同时间点使用不同的Lut
	public Texture2D dawnLut;
	public Texture2D morningLut;
	public Texture2D noonLut;
	public Texture2D duskLut;
	public Texture2D nightLut;

	PostProcessingBehaviour postProcessingBehaviour;
	PostProcessingProfile profile;

	void Start() {
		postProcessingBehaviour = GetComponent<PostProcessingBehaviour>(); // 获取camera上挂载的Post Processing Behaviour脚本
		profile = postProcessingBehaviour.profile; // 获取脚本上的profile字段
	}

	public void ChangeLut(TimeOfDay timeOfDay) {
		UserLutModel.Settings settings = profile.userLut.settings; // 创建新的profile属性设定
		switch (timeOfDay) { // 给lut赋值，注意是Texture2D类型的
			case TimeOfDay.Dawn: settings.lut = dawnLut; break;
			case TimeOfDay.Morning: settings.lut = morningLut; break;
			case TimeOfDay.Noon: settings.lut = noonLut; break;
			case TimeOfDay.Dusk: settings.lut = duskLut; break;
			case TimeOfDay.Night: settings.lut = nightLut; break;
		}
		profile.userLut.settings = settings; // 赋值给profile的userLut的settings属性
		profile.userLut.OnValidate(); // 刷新效果
		Debug.Log("Change To " + timeOfDay + " Lut !");
	}
}

public enum TimeOfDay {
	Dawn, Morning, Noon, Dusk, Night
}
```

代码其实不难，本质上就是不同的时间点给`Lut`贴上不同的贴图。这里有两个点需要注意：
1. `UserLutModel.Settings`是一个`struct`而不是`class`，所以需要将`profile.userLut.settings`整体替换掉而不是只替换`settings.lut`这个属性。
2. 当更改完成之后需要调用一下`profile.userLut.OnValidate();`这句代码祈祷了刷新的作用，如果没有的话，画面是不会有改变的！一定要加上！

这样，我们就可以随着时间改变而使用不同的`Lut`，达到光线变化的效果。事实上，在一些风格化的游戏中，因为画风的原因，使用`Lut`可以更好的自定义画面，而不是全都用自然光模拟。

## 缺点
老规矩，列一下这个方法的问题：
1. 影子不能动！这是这个方案最大的不足！
2. 因为色彩是整个画面一起变，所以如果涉及到ui的话，需要使用另一个`camera`，专门渲染ui
3. `Lut`生成需要美术的合作，不能使用`Unity`直接生成

## 疑问
有的人可能会好奇，说`Post Processing Stack`这种全屏的像素处理是不是会有比较大的性能消耗。事实上这边问题不大，首先我们只对`Lut`进行处理，并不是一个非常高开销的操作，同时由于移动端屏幕分辨率和屏幕大小的限制，实际的像素点比电脑上要少许多，基本上不用担心效率的问题。

# 总结
这个方法就是这样的，其实比较适合那种卡通风格的游戏啦，不是很写实的，自定义调`Lut`效果更好。如果真的是商业级的大型项目我还是建议各位去把动态加载`lightmap`看看吧，这边仅提供一种解决方案。
