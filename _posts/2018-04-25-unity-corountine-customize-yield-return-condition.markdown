---
layout: post
title: Unity中自定义Coroutine的yield return返回条件
comments: true
date: 2018-04-25 07:50:48.000000000 +09:00
author: William Xie
tags: Unity
---
# 概述
>当游戏开发涉及到人工智能设计的部分时，判断人物动画／寻路等这类持续一段时间而不是单帧的操作何时结束是一个能扰乱代码质量的地方。一种最简单的方法是在`update()`中每帧进行检验，但这种方式会使用大量的`if else`结构，使得代码混论难懂，耦合高，同时即使该帧没有操作正在进行也需要一个`if else`判断是否需要检测操作进度，消耗cpu性能。如果我们能使用`coroutine`协程来解决这个问题，进度检测代码就可以和主更新方法分离开了来了。\
>然而如果仍然使用`if else`判断动作是否结束，再`yield return`，固然可以实现功能，但是每一处都要写一遍，不利于代码的重用。因此，如果可以仿照`Unity`自己的诸如`WaitForSeconds`，`WaitUntil`自定义出形如`WaitForAnimationFinished`，`WaitForNavigationFinished`的`yield return` 返回条件，一来代码看得更加直观，二来代码风格统一，三来高度解耦合，方便代码重构。

# 实现
那么如何实现自定义`yield return`返回条件呢？其实`Unity`已经为我们考虑到了这一点，特地提供了一个父类`CustomYieldInstruction`，方便我们继承它来自定条件。下面我们新建一个新的`WaitForCustomCondition`脚本继承自`CustomYieldInstruction`来演示下用法：

```csharp
using System.Collections;
using UnityEngine;

public class WaitForCustomCondition : CustomYieldInstruction {

	public override bool keepWaiting {
		get {
            // 这里进行是否返回的判断，注意true为继续等待，即不返回，false才是返回
			return true;
		}
	}
}
```

可以看到代码其实非常的简单，只需要重写`keepWaiting`这个属性访问器即可，非常方便。如果返回**true**，则继续等待，如果返回**false**，则执行`yield return`后面的代码。

现在让我们回到之前提到的问题，怎么样使用`CustomYieldInstruction`来自定义检测动画片段播放完毕和寻路结束事件呢？

大体思路就是，新建`WaitForAnimationFinished`和`WaitForNavgationFinished`类，继承自`CustomYieldInstruction`类，然后在构造方法中传入相关组件进行初始化（如`Animator`，`NavMeshAgent`这一类组件），最后重写`keepWaiting`进行判断，相关代码如下：

```csharp
public class WaitForAnimationFinished : CustomYieldInstruction {

	AnimatorStateInfo animatorStateInfo;

	public override bool keepWaiting {
		get {
			// 动画控制器的标准化时间小于1，说明没有播放完，不返回
			return animatorStateInfo.normalizedTime < 1f;
		}
	}

	// 传入Animator组件初始化
	public WaitForAnimationFinished(AnimatorStateInfo animatorStateInfo) {
		this.animatorStateInfo = animatorStateInfo;
	}
}
```

```csharp
public class WaitForNavigationFinished : CustomYieldInstruction {

	public static readonly float FLOAT_PRECISION = 0.00001f;

	NavMeshAgent agent;

	public override bool keepWaiting {
		get {
			// 寻路代理器的剩余路程大于制动距离，说明没有寻完路，不返回，注意浮点数误差
			return agent.remainingDistance > agent.stoppingDistance + FLOAT_PRECISION;
		}
	}

	// 传入NavMeshAgent组件初始化
	public WaitForAnimationFinished(NavMeshAgent agent) {
		this.agent = agent;
	}
}
```

这里的代码都十分的直观，我就不多解释了。下面来举个实际使用场景的例子：

```csharp
// 下面代码只是演示，没有通用性
[RequireComponent(typeof(Animator))] // 要求附着物体有Animator组件
[RequireComponent(typeof(NavMeshAgent))] // 要求附着物体有NavMeshAgent组件
public class AICharater : MonoBehaviour {

	Animator animator;
	NavMeshAgent agent;

	bool alive;

	void Awake() {
		// 获取相关组件
		animator = GetComponent<Animator>();
		agent = GetComponent<NavMeshAgent>();
	}

	void Start() {
		alive = true;
		// 开始动画与寻路的协程
		StartCoroutine(ExeAnimationTask());
		StartCoroutine(ExeNavigationTask());
	}
	
	IEnumerator ExeAnimationTask() {
		while (alive) {
			// 这里的_Animate参数是随便填的，不要在意
			animator.SetBool("_Animate", true);
			// 像new WaitForSeconds(float time)一样使用
			yield return new WaitForAnimationFinished(animator.GetCurrentAnimatorStateInfo(0));
			animator.SetBool("_Animate", false);
		}
	}

	IEnumerator ExeNavigationTask() {
		while (alive) {
			// 这里的destination的值是随便填的，不要在意
			agent.destination = new Vector(1f, 1f, 1f);
			// 像new WaitForSeconds(float time)一样使用
			yield return new WaitForNavigationFinished(agent);
		}
	}
}
```

# 缺点
这里的缺点到不仅是针对自定义协程返回条件的，也有判断动画播放结束和寻路结束的一些潜在问题：

1. 如果一个动作包含多个连续的动画片段，仅使用`normalizedTime`判断是否播放结束是不可行的，需要同时检测当前播放动画的名称。然而这样增加了代码量，动画名称可能是硬编码写入程序，不方便调试和策划或美工人员的编辑
2. 可能会创建大量条件检测的实例，消耗cpu时间，可以考虑使用对象池模式对实例进行复用。
3. `NavMeshAgent`在设置完目的地后到计算出路径之前，`remainingDistance`都为0，可能会造成判断的bug。

# 总结
主要也就是展示一下`CustomYieldInstruction`的用法和使用场景，如果各位有更好的检测动画播放结束或者寻路结束的方法，也可以拿出来讨论讨论。💻☕️