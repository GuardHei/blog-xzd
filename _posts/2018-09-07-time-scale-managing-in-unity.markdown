---
layout: post
title: Unity中时间尺度管理
comments: true
date: 2018-09-07 07:50:48.000000000 +09:00
author: William Xie
tags: Unity
---

# 概述
> 在**这就是IB**这个项目中，为了提高游戏的打击感，在玩家使用近战武器打进敌人身体里的时候，降低了时间尺度`timeScale`来表现肉体对于武器的阻碍——关于如何做好打击感我可能会在下篇文章说，这篇主要集中在`Unity`的时间上——这就导致了一个问题：如果武器同时打进了多个敌人并且这些肉体的阻碍程度不同时，时间该怎么调整？如果这时我们还有剧情演出需要改变时间尺度，我们又该以什么为准呢？

# 问题
主要问题我们已经提到一些了，归根结底还是处理不同时间尺度改变的请求之间的冲突。其实最好的方法应该是更改策划的需求来消除冲突。或者在这个例子里，改变武器的攻击动画的播放速度来体现滞后感，这样就不会和剧情演出冲突了。不过当没有办法消除这些冲突的时候，我们还是得考虑一个普遍的使用方法。

# 解决方案
既然是解决冲突，那么我们无非有如下三种方案：

1. 以特定规则忽略一些请求
2. 以特定规则选取一个请求
3. 以特定规则叠加一些请求

然而由于我们存在多个需求一剧情演出，战斗需要等等一我们可以考虑把这些请求进行分层处理，每一层负责一类请求，最后把各个层级一一叠加。

那么首先考虑战斗层。在武器进入不同的肉体中，可能第一反应是把这两个时间请求叠加，但其实，我们应该以最小的时间尺度为标准，因为总是最有阻碍力的肉体在工作。所以我们可以在每次处理战斗发出的时间请求时，先遍历所有已存在的战斗时间请求，然后找出最低的请求值——即时间尺度来作为当前时间尺度。

我们再来解决剧情演出的需求。在剧情演出中其实很少会出现多个时间尺度改变的请求。而且因为剧情演出的复杂性，很难说什么时候以最小值为标准，什么时候以最大值为标准，因此，我个人的建议是使用一个`int`类型的变量来作为该请求的优先性标示。那么比如说我们有两个请求，一个请求的优先级是1，另一个优先级是2，那么我们就只选用优先级为2的请求。

最后我们还要考虑其它一些杂七杂八的请求，比如说主角可能有一个子弹时间的能力（没错，就是黑客帝国），或者说需要一个全局的最终调校，所以我们应该在增加一个全局层级来处理这些事情。

那么相关代码如下：

```csharp
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class TimeManager : MonoBehaviour {

	public static float MinimumTimeScale = 0;
	public static float MaximumTimeScale = 100f;
	public static TimeManager Instance {
		get;
		private set;
	}

	public static float TimeScale {
		get { return timeScale; }
		private set {
			timeScale = value;
			Time.timeScale = value;
		}
	}

	private static bool isPaused;
	private static float timeScale;
	
	private static readonly List<TimeEffectRequest> globalRequests = new List<TimeEffectRequest>();
	private static readonly List<TimeEffectRequest> storyRequests = new List<TimeEffectRequest>();
	private static readonly List<TimeEffectRequest> actionRequests = new List<TimeEffectRequest>();

	private void Awake() {
		timeScale = Time.timeScale;
		Instance = this;
	}

	private void Update() {
		if (!isPaused) {
			bool flag = false;
			float timeDiff = Time.unscaledDeltaTime;
			for (int i = 0; i < globalRequests.Count; i++) {
				TimeEffectRequest request = globalRequests[i];
				request.lifeRemained -= timeDiff;
				if (request.lifeRemained <= 0f) {
					globalRequests.RemoveAt(i);
					flag = true;
				}
			}
			
			for (int i = 0; i < storyRequests.Count; i++) {
				TimeEffectRequest request = storyRequests[i];
				request.lifeRemained -= timeDiff;
				if (request.lifeRemained <= 0f) {
					storyRequests.RemoveAt(i);
					flag = true;
				}
			}
			
			for (int i = 0; i < actionRequests.Count; i++) {
				TimeEffectRequest request = actionRequests[i];
				request.lifeRemained -= timeDiff;
				if (request.lifeRemained <= 0f) {
					actionRequests.RemoveAt(i);
					flag = true;
				}
			}
			
			if (flag) CalculateTimeScale();
		}
	}

	public static void Play() {
		isPaused = false;
		CalculateTimeScale();
	}

	public static void Pause() {
		isPaused = true;
		TimeScale = 0;
	}

	public static void HandleRequest(TimeEffectRequest request) {
		request.lifeRemained = request.duration;
		switch (request.layer) {
			case TimeEffectLayer.Globe: globalRequests.Add(request);
				break;
			case TimeEffectLayer.Story: storyRequests.Add(request);
				break;
			case TimeEffectLayer.Action: actionRequests.Add(request);
				break;
		}

		CalculateTimeScale();
	}

	private static void CalculateTimeScale() {
		float scale = CalculateGlobalScale() * CalculateStoryScale() * CalculateActionScale();
		if (scale < MinimumTimeScale) scale = MinimumTimeScale;
		else if (scale > MaximumTimeScale) scale = MaximumTimeScale;
		TimeScale = scale;
	}

	private static float CalculateGlobalScale() {
		float globalScale = 1f;
		int priority = -1;
		foreach (var request in globalRequests)
			if (request.priority >= priority) globalScale = request.value;
		return globalScale;
	}

	private static float CalculateStoryScale() {
		float storyScale = 1f;
		int priority = -1;
		foreach (var request in storyRequests)
			if (request.priority >= priority) storyScale = request.value;
		return storyScale;
	}

	private static float CalculateActionScale() {
		float actionScale = 1f;
		foreach (var request in actionRequests)
			if (request.value < actionScale) actionScale = request.value;
		return actionScale;
	}
}

[Serializable]
public class TimeEffectRequest {

	public TimeEffectLayer layer;
	public int priority;
	public float value;
	public float duration;
	public float lifeRemained;
}

public enum TimeEffectLayer {
	Globe,
	Story,
	Action
}
```

好吧，代码有点多，不过大概缕完思路后也就清晰了。首先我们有两个方法`Play()`和`Pause()`，可能很多人感觉奇怪，我们直接把发送一个把时间尺度设为0的请求不就行了？为什么要单独做呢？这是因为考虑到大多时候暂停游戏都是为了调出游戏菜单界面（当然你要说魂系游戏那就算了），当暂停的时候其他的时间请求应该冻结，所以这个和正常的时间尺度改变的请求是不一样的。

然后是我们的时间请求对象`TimeEffectRequest`。可以看到我们存储了请求所在层级，优先级，值，持续时间和剩余持续时间这些数据。而层级我们使用枚举，简单地划分了三个层次：全局，剧情，战斗。

在`Update()`方法中，如果没有暂停的话，我们遍历所有的请求并相继扣除它们的剩余时间。当剩余时间为0时，我们剔除这个请求，并重新计算新的时间尺度。

方法`HandleRequest(TimeEffectRequest)`是用于接收时间尺度改变请求的。我们把请求放入属于其层级的列表中，然后调用`CalculateTimeScale()`方法刷新时间尺度。由于不同层级的时间尺度叠加规则不同，我们又分别写了三个方法来计算各个层级的时间尺度，最后相乘等到最终尺度。我们还设定了最大和最小尺度进行规范。

最后的最后，我们使用单例模式进行管理，由于牵扯到`MonoBehaviour`，与`Unity`脚本独特的生命周期挂钩，所以在`Awake()`方法里对静态访问对象赋值。

# 缺点
谈到不足，首先一个问题是精度问题。我们对每个请求的生存周期计算是逐帧减去`Time.unscaledDeltaTime`，而这个值并不是精确的，再加上浮点数误差的积累，每个请求的生命周期会不精确。因此，策划在填数值的时候最好不要太小。

另外一个问题是效率。考虑到每一帧都要遍历三个列表，当请求数增多的时候，效率会有较大的影响。但这不是一个特别需要在意的问题，因为总的来说绝对值较小，不应该成为效率的瓶颈。