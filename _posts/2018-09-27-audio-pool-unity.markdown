---
layout: post
title: Unity中的音效池
comments: true
date: 2018-09-27 07:50:48.000000000 +09:00
author: William Xie
tags: Unity
---

# 概述
> 音效是独立开发者在开发游戏中比较容易忽视的一个方面。比起直观的画面和玩法，音效不那么容易被注意到。但是，如果缺少了音效，玩家就缺少了一层反馈，游戏的操作手感就会大大改变。Unity引擎已经帮我们封装了相关的API，我们在实际开发的过程中，只需要调用相关的API就可以快速实现一些功能。

# Unity的音频框架
简单的介绍一下Unity的音频框架。在Unity中，所有可以发出声音的物体都需要带有一个`AudioSource`组件。这个组件，即声源，定义了改物体发声的各种参数，包括是否为立体音效，音量衰减规律，多普勒效应等等。当我们想要让这个音源播放声音的时候，只需要把Unity存储音频的对象`AudioClip`的实例给这个声源就行了。

但是很多情况下，我们并不想在每次发出声音的时候都给游戏物体挂上一个`AudioSource`组件，我们只是“一次性”地在某个点发出声音而已，比如UI地操作反馈音效，我们是不需要给每个UI都绑上`AudioSource`组件的。幸运的是，Unity也为这种情况做了准备。Unity提供了一个`AudioSource.PlayClipAtPoint(AudioClip clip, Vector3 position, float volume = 1f)`的**API**。就像这个方法的命名一样，调用这个方法Unity便会在`position`处播放一个音量为`volume`大小的`clip`音频。然而，如果这个方法进行反编译，我们会发现Unity的实现有很大的运行效率问题。Unity的做法是临时创建一个`GameObject`实例，然后再创建一个`AudioSource`组件挂到游戏对象上，最后把游戏对象移动到播放位置，并播放输入的`AudioClip`。播放结束之后，再对该游戏对象进行销毁。

# 问题
我们知道，Unity初始化一个`GameObject`是比较消耗性能的。如果只是做一个小游戏的话，偶尔调用一下这种开销大但使用方便的方法是无所谓的。但是在性能比较吃紧的情况下，我们就需要使用一套兼顾效率与易用性的工具接口。所以，我们考虑自己实现一套音效池，用来进行所有的临时，一次性的音效播放操作。

# 实现
事实上，把这个需求拆来了看，不过是实现一个比较另类的对象池罢了。重点关注的一个是与`MonoBehaviour`生命周期的贴合，另一个就是对于音效开发需求的良好接口拓展。下面是我的实现代码：

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class AudioManager : MonoBehaviour {

	public static int Count => idleAudioSources.Count;

	private static AudioManager instance;
	private static Transform audioRoot;
	private static readonly Queue<AudioSource> idleAudioSources = new Queue<AudioSource>(5);
	
	private void Awake() {
		instance = this;
		GameObject go = new GameObject("Audio Root");
		DontDestroyOnLoad(go);
		audioRoot = go.transform;
	}

	public static void PlayAtPoint(AudioClip clip, Vector3 position, float volume = 1f) {
		AudioSource source = GetAudioSource();
		source.transform.position = position;
		source.clip = clip;
		source.volume = volume;
		source.Play();
		instance.StartCoroutine(ExeRecycleCoroutine(source));
	}

	private static AudioSource GetAudioSource() {
		AudioSource source;
		if (idleAudioSources.Count > 0) {
			source = idleAudioSources.Dequeue();
			source.gameObject.SetActive(true);
		} else {
			GameObject gameObject = new GameObject("Public Audio Source");
			gameObject.transform.parent = audioRoot;
			source = gameObject.AddComponent<AudioSource>();
			source.spatialBlend = 1f;
			source.loop = false;
		}

		return source;
	}

	private static IEnumerator ExeRecycleCoroutine(AudioSource source) {
		float time = source.clip.length;
		yield return new WaitForSeconds(time);
		source.Stop();
		source.gameObject.SetActive(false);
		idleAudioSources.Enqueue(source);
	}
}
```

代码并不难，简单来说，我们维护了一个`Queue<AudioSource>`队列存储音源对象。每次播放一次性音效时，我们先去对象池里找一遍有没有处于闲置状态的音源对象。如果有，就使用这个对象来播放音效，否则就新创建一个音源对象。同时，我们开启了一个协程对播放的片段进行计时回收操作。这样，当该片段播放完毕的时候就会自动进行重置并回到对象池中。