---
layout: post
title: 在Unity中实现场景中动态血迹效果
comments: true
date: 2018-08-27 07:50:48.000000000 +09:00
author: William Xie
tags: Unity
---

# 概述
>上一篇博客聊到了关于如何制作溅出的血迹附着在场景上的效果。大概效果参见下图：

![screenshot1](https://static.jam.vg/raw/a06/81/z/16599.png)

>上面这张图上可以很明显看到红色和紫色的血迹在地上遗留，这次主要就是要实现这个效果。

# 问题
那实现这个效果主要有哪些问题需要解决呢？

1. 如何以较低的成本获得不同的血的形状？
2. 如何知道血迹应该放置在哪里？
3. 如何使血迹能贴合地附着在物体的表面
4. 如何有效率地绘制大片大片的血迹

# 实现

## 一个可能的做法（但是很明显不是本篇重点）

如果是以前做过fps游戏的读者可能看到这个问题第一反应是直接把血迹贴图画在物体表面的材质纹理上，然后过一段时间再把原来的纹理画回去。这种做法做起来比较简单直接，但是有很多问题没法解决，下面就来大概列一下：

1. 效率问题。首先每绘制一次血迹就要遍历一边血迹贴图的所有像素点，再绘制到物体表面的贴图上。对于fps游戏里的弹痕来说，因为比较小，分辨率很有可能也就`64x64`，这样遍历一遍需要4096次，不算高效，但至少可以接受。如果换成血迹，一个低清的血迹贴图也要在`256x256`左右，也就意味着遍历一遍需要65536次，同时还有cpu绘制贴图等等耗时操作，如果一个角色身上滋滋冒血，整个游戏就得靠眨眼补帧了。此外，由于贴图被破坏，一些原本可以进行的优化就被打断了。
2. 显示问题。事实上，这种方法也不能保证显示正确。在很多情形下物体的贴图并不是严格uv一对一的，拉伸变形，重复铺开是非常常见的。以**Unity**自带的地形系统为例，材质都是`tiled`类型，如果直接画在贴图上，地面上会很有规律地出现多处血迹。而在一些拉伸物体上很有可能导致血迹突然变大变糊。

## 高效实现

那就开始说说我是怎么实现的吧。首先，针对之前提出的问题一个个写出解决思路：

1. 血溅出来肯定是各种各样的，但是要美工一种种画出来成本太高，所以我们只要美工出一份血迹贴图即可。我们使用一张面片`quad`加上贴图来表示血迹。通过对面片在一定范围内的**放大缩小**和**旋转**，可以近似的创造出不同的形状的血迹，这是一个很有用的视觉欺诈技巧。为了更好的效果，我们用着色器`shader`里的`Tint Color`选项来对血迹的颜色也作出一定的变化，比如从浅红到黑红色，这就意味着我们需要的血迹贴图是一个纯白的，这样才方便覆盖上颜色。
2. 角色受到攻击溅出血一般都是用一个粒子效果做出来的，所以使用粒子系统的物理碰撞检测获得溅出来的血与场景碰撞的世界坐标，就知道血迹该画哪了。
3. 同样，通过粒子系统的物理碰撞检测获得碰撞体表面的法线，将血迹面片的朝向与碰撞表面的朝向弄成一致的就行了，为了避免`z-fighting`导致的交替闪烁，需要将面片适当地延法线方向偏移出去一点。
4. 这个问题很重要啊，由于我们需要不同大小的血迹，所以动态批处理`dynamic batching`是指望不上了。权衡后我决定用`gpu instancing`来进行优化。但**Unity**的`gpu instancing`是一个dt的话题，里面有很多坑，一不小心就触犯了什么禁忌，导致优化失败。所以为了方便优化，我们直接使用一个粒子系统来管理所有的血迹，这样粒子系统会帮我们自动进行`gpu instancing`。

### 步骤

#### 创建血迹管理对象

由于我们使用了粒子系统来管理所有的血迹，所以我们先在场景里创造出一个管理的游戏对象。在场景里新建一个空白的`GameObject`，挂上一个`ParticleSystem`组件，然后进行一些必要的设置：

1. 将`Simulation Space`改成`World`，这样保证后面物理碰撞的相关数据都是基于世界空间的而不是父空间的。
2. 取消`Looping`和`Play On Awake`选项，我们只是借`Particle System`做优化，不需要真的做个粒子上去。
3. 取消`Emission`和`Shape`两个模块`Module`选项，因为我们不需要粒子喷射效果。
4. 展开`Render`模块，将`Render Mode`改成`Mesh`，因为我们需要渲染一个个血迹面片。
5. 在下面的网格`Mesh`选项中使用面皮`Quad`。同时在后面的材质`Material`选项中使用血迹材质。但是我们目前还没有血迹材质，所以我们先来创建一个血迹材质。
6. 创建一个新的材质，选择`Particles/Alpha Blended`着色器`shader`，将血迹贴图拖到粒子材质`Particle Texture`选项上。再将这个创建好的材质拖到上面说的粒子系统的材质选项上。
7. 将`Render`模块中的`Render Alignment`选项设为`World`，同时勾上下面的`Enable Mesh GPU Instancing`选项，其他就不用变了。
8. 最后，创建一个血迹管理脚本挂在上面，脚本内容如下：

```csharp
using System;
using UnityEngine;
using Random = UnityEngine.Random;

public class ParticleDecalManager : MonoBehaviour {

	public const int CAPACITY = 2000;
	
	private static Transform decalRoot;

	private static int index;
	private static ParticleSystem particleSystem;
	private static readonly ParticleDecalData[] data = new ParticleDecalData[CAPACITY];
	private static readonly ParticleSystem.Particle[] particles = new ParticleSystem.Particle[CAPACITY];
	
	private void Awake() {
		decalRoot = transform;
		particleSystem = GetComponent<ParticleSystem>();
		for (int i = 0; i < CAPACITY; i++) data[i] = new ParticleDecalData();
	}

	public static void OnParticleHit(ParticleCollisionEvent @event, float size, Color color) {
		SetParticleData(@event, size, color);
	}

	private static void SetParticleData(ParticleCollisionEvent @event, float size, Color color) {
		if (index >= CAPACITY) index = 0;
		ParticleDecalData data = ParticleDecalManager.data[index];
		Vector3 euler = Quaternion.LookRotation(@event.normal).eulerAngles;
		euler.z = Random.Range(0, 360);
		data.position = @event.intersection;
		data.rotation = euler;
		data.size = size;
		data.color = color;

		index++;                 
	}

	public static void DisplayParticles() {
		for (int i = 0, l = data.Length; i < l; i++) {
			ParticleDecalData data = ParticleDecalManager.data[i];
			particles[i].position = data.position;
			particles[i].rotation3D = data.rotation;
			particles[i].startSize = data.size;
			particles[i].startColor = data.color;
		}
		
		particleSystem.SetParticles(particles, CAPACITY);
	}
}

[Serializable]
public class ParticleDecalData {

	public float size;
	public Vector3 position;
	public Vector3 rotation;
	public Color color;
}
```

目前看这个脚本可能还不太知其所以然。我也就先大概说下流程。`Awake`方法是一些初始化操作就不用细说了。`ParticleDecalData`这个类是保存每个血迹的相关数据的，包括大小，位置，旋转和颜色。我们设置`CAPACITY`这个血迹数最大值，如果出现超过`CAPACITY`的血迹数，就将相对的最早的一个血迹拿过来重新设置数据后使用，这是处于优化的考量。`OnParticleHit`方法是方便后面血迹碰撞检测脚本调用的，可以看到里面就调用了下`SetParticleData`方法。而这个方法可以很明显看出是用来设置血迹的相关数据的 -- 位置，旋转，大小和颜色。最后的`DisplayParticles`方法是刷新粒子系统里的血迹数据用的。将这些方法设置为`static`是为了方便后面的碰撞检测脚本可以不需要获取实例就可以直接调用方法。

#### 创建血液碰撞检测

这个问题我们需要先拿到游戏里做出血液飞溅效果的那个预置体`Prefab`。我们的做法是在这个预置体上挂一个粒子系统的碰撞检测脚本，当然了，也就需要对这个血液飞溅效果的粒子系统做一些设置。

1. 勾选上碰撞`Collision`模块`Module`，展开。将`Type`选项设为`World`，`Mode`选项设为`3D`。至于`Collision Quality`的一系列选项就看你了，我建议是只对静态碰撞进行检测就行了。最后购选上`Send Collision Messages`即可，这样才能保证脚本里的`OnParticleCollision`方法会被正确回调。
2. 创建一个碰撞检测脚本挂到预置体上，脚本内容如下：

```csharp
using System.Collections.Generic;
using UnityEngine;

public class ParticleDecalController : MonoBehaviour {

	[Range(0f, 1f)]
	public float decalRate = 1f;
	public float minSize;
	public float maxSize;
	public Gradient colorGradient;
	
	private ParticleSystem _particleSystem;
	private readonly List<ParticleCollisionEvent> _collisionEvents = new List<ParticleCollisionEvent>(4);

	private void Awake() {
		_particleSystem = GetComponent<ParticleSystem>();
	}
	
	private void OnParticleCollision(GameObject other) {
		int count = _particleSystem.GetCollisionEvents(other, _collisionEvents);

		for (int i = 0; i < count; i++) {
			float r = Random.Range(0f, 1f);
			if (r <= decalRate) ParticleDecalManager.OnParticleHit(_collisionEvents[i], Random.Range(minSize, maxSize), colorGradient.Evaluate(r));
		}
		
		ParticleDecalManager.DisplayParticles();
	}
}
```

一个个都捋一遍。首先`decalRate`这个值代表当溅出的血液碰到物体表面后有多大的几率留下血迹。为什么不每个溅出的血液都画血迹呢？这是考虑到一般血液飞溅的粒子系统一次`emission`往往有30～60个粒子，这就意味着但是一次击打就要画出30～60滩血，太过密集，所以我们之画其中一部分。

下面的两个变量，`minSize`和`maxSize`控制着血迹的大小范围了，到时候在两者之间随机选择一个尺寸。最后一个变量`colorGradient`控制着颜色的变化范围，增加血迹的显示多样性。

最后重点说一下`OnParticleCollision`方法。这是一个内置的`Unity Message`。当这个预置体上的每帧粒子发生了碰撞的时候就会调用这个方法。传递的参数`GameObject other`则是检测到的碰撞体所在的`GameObject`。我们使用`ParticleSystem.GetCollision(GameObject, List<ParticleCollisionEvent>)`获得相关的碰撞事件。这里值得一提的是`List<ParticleCollisionEvent>`参数需要我们传进去一个已经初始化好的粒子碰撞事件列表。`ParticleSystem.GetCollision(GameObject, List<ParticleCollisionEvent>)`这个方法会将碰撞事件装入该列表中，同时会返回一个`int`值表示总共的碰撞事件个数，这里我们用`count`变量把个数缓存下来。

接下来我们遍历每个碰撞事件进行处理。首先我们随机出一个0~1之间的数，这个数我们先用它和`decalRate`比较来决定是否生成血迹，再用这个数确定血迹的颜色，即`colorGradient.Evaluate(r)`这一行。如果`r <= decalRate`就是需要绘制血迹的时候，我们调用`ParticleDecalManager.OnParticleHit()`方法，将相关的值传进去。

最后，当所有的血迹数据都设置后，我们调用`ParticleDecalManager.DisplayParticles()`方法对粒子系统的数据进行刷新。

大功告成！

# 效果展示

你自己试试呗，我就懒得再传一张图片到图床上了。

# 缺点

老规矩，没有方法是完美无缺的，下面说下这个方法的缺点：

1. 只能附着在静态的物体上，不能跟随动态物体。因为我们的血迹面片只在碰撞的时候设置了一下位置等数据，所以这个问题实属正常。一种可能的解决方案就是每帧对每个血迹面片的位置朝向等等进行更新。不过，效率就成问题了。所以我们也可以考虑创建两个血迹管理系统，一个负责静态场景上附着的血迹的管理，`CAPACITY`相对较大。另一个负责动态物体上血迹的管理，`CAPACITY`比较下，这样每帧更新起来快一点。
2. 不一定能在不规则的表面正确显示，或者说可能会有血迹延伸出表面。这是因为我们只是粗略地将血迹面片的发现和碰撞的那个面的法线进行了统一，但是如果表面比较复杂，或者更简单，血迹生成在两个相交的表面之间，面片只能保证一个方向，也就是说不可能折叠起来贴和在两个表面上。