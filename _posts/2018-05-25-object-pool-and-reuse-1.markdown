---
layout: post
title: Unity游戏开发优化 —— 通用对象池与复用（一）
comments: true
date: 2018-05-25 07:50:48.000000000 +09:00
author: William Xie
tags: Unity
---

# 概述
>游戏开发的一个大坑就是优化。这边博客主要讨论什么是对象池缓存技术以及如何基于Unity设计一个通用的对象池缓存框架。

## 为什么要用对象池？
引入对象池的概念是为了减少内存碎片的产生与降低创建实例时的cpu消耗，一般适用于各种粒子特效，或是射击游戏里的子弹之类的大量重复出现的游戏物体。

### 什么是内存碎片？
当一个实例的所有引用都被赋予`null`时，这个实例就变成了内存垃圾，即将被系统清理。当它被`GC`清理掉后，内存里就会留下一条空的可以使用的内存。然而问题来了，假设这段空余内存有4个字节的大小，整个系统还剩8个字节大小的内存可以使用，但是如果系统试图创建一个8字节大小的实例后会发现出现`OutOfMemory`的异常，这是为什么呢？

其实原因很简单，大多数创建的实例会开辟出一段连续的内存空间存放数据（重点！），但是在这个系统中，虽然有8个字节的内存剩余，但是连续可用的最大内存也只有4个字节，系统当然不能把实例拆成一段段的塞到不同的地方去，因此抛出`OutOfMemory`的异常。

内存碎片导致的异常往往在短时间的运行测试中无法被发现，因为这时候内存还没有完全碎片化，然而我们知道对于一些放置类手游或是后台进程长期不终结的项目，内存的碎片化是完全可能导致问题的，对此，不少游戏开发公司会对即将上线的游戏放置运行24小时，以便确定不存在内存泄漏或是碎片化。

### 什么是创建实例时的cpu消耗？
这个概念就简单多了，当系统想要创建一个新的实例的时候，它需要先委托cpu帮它在内存中开辟出一片连续的空余内存用于存放实例的数据，而这个寻找的过程则是需要花时间的，因为这不是简单的内存寻址，还需要比对内存的连续性。当然，由于绝对值不高，一般情况下很少需要考虑到这一点，然而如果需要在游戏的每一帧中创建数以百计的实例，比如烟花，这个开销就不容忽视了。

## 所以什么是对象池？
对象池，顾名思义，就是弄一个大的池子把一定量对象扔去，如果要用了就从里面捞出来一个，擦擦弄弄使用，用完了洗洗干净，再放回去。这么一个流程，就是对象池的初始化，与对象的获取，回收的过程。由于重新使用一个新对象的时候并没有创建一个新实例，而是继续使用之前的对象，因此实例会被持续引用，不会被`GC`清理成内存碎片。

## 设计对象池
首先，根据上述的流程，我们设计一个最简单的对象池，用一个`Queue<T>`来作为对象的容器，代码如下：

```csharp
public interface IReusable { // 所有需要复用的对象都需要实现的接口
	void Init(); // 用于从对象池里提取后的初始化
	void Recycle(); // 用于使用完将对象回收至对象池
}

public class ObjectPool<T> where T : IReusable, new() { // 使用泛型适配不同种类的对象，保证包含一个无参构造器

	public const int INIT_CAPACITY = 20; // 初始化对象池大小

	private Queue<T> objects;

	public ObjectPool() {
		objects = new Queue<T>(INIT_CAPACITY);
	}

	public ObjectPool(int capacity) {
		objects = new Queue<T>(capacity);
	}

	private void Instantiate() {
		T t = new T();
		objects.Enqueue(t);
	}

	public T Get() {
		if (objects.Count < 0) { // 如果对象池没有剩余可用的对象，就新建一个对象，推进对象队列
			Instantiate();
			return Get();
		} else {
			T t = objects.Dequeue(); // 从对象队列里挤出一个可用对象
			t.Init(); // 初始化对象状态
			return t;
		}
	}

	public void Recycle(T t) {
		t.Recycle(); // 回收对象至闲置状态
		objects.Enqueue(t); // 将回收后的对象推进队列
	}
}
```

上面的代码非常简单，就不详细分析了，下面给出一段使用的实例代码：

```csharp
Object<Bullet> bulletPool = new ObjectPool<Bullet>(); // 创建一个对象池
Bullet bullet = bulletPool.Get(); // 从对象池里获得对象
bullet.Shoot(); // 让获得的对象做一些事
if (bullet.hitted) { 
	bulletPool.Recycle(bullet); // 回收对象
}
```

但是这样的对象池非常简陋，存在不少问题。一个主要问题就是当实例从对象池中被挑出以使用的时候，对象池丧失了对其的控制权。同时，我们还要考虑到当对象池应该有一个限定的大小以便更好的控制内存，避免一个不被频繁使用的对象池占据大量空间。

```csharp
public enum ObjectPoolMaximumOption { // 当对象池数量达到限制时获取新对象的策略
	Ignore, // 忽略此次获取请求
	Rob // 将第一个对象强制回收使用
}

public class ObjectPool<T> where T : IReusable { // 使用泛型适配不同种类的对象

	public const int INIT_CAPACITY = 20; // 初始化对象池大小
	public static int MAX_CAPACITY = 40; // 对象池大小上限

	private Queue<T> idleObjects; // 闲置（可用）对象的队列
	private ArrayList<T> busyObjects; // 忙碌（使用中）对象的队列

	public int Size { // 获得对象池总大小
		get {
			return idleObjects.Count + busyObjects.Count;
		}
	}

	private ObjectPoolLimitOption option;

	public ObjectPool(ObjectPoolLimitOption option = ObjectPoolLimitOption.Rob) { // 默认选用Rob策略
		this.option = option;
		idleObjects = new Queue<T>(INIT_CAPACITY);
		busyObjects = new List<T>(INIT_CAPACITY);
	}

	private void Instantiate() {
		T t = new T();
		idleObjects.Enqueue(t);
	}

	public T Get() {
		if (idleObjects.Count == 0) {
			if (Size < MAX_CAPACITY) {
				Instantiate();
				return Get();
			} else if (option == ObjectPoolLimitOption.Ignore) {
				return null; // 忽略，所以返回null
			} else {
				T t = busyObjects[0]; // 获得第一个使用中对象
				Recycle(t); // 强制回收
				return Get();
			}
		} else {
			T t = idleObjects.Dequeue(); // 挑出对象
			busyObjects.Add(t); // 放入忙碌列表
			t.Init();
			return t;
		}
	}

	public void Recycle(T t) {
		t.Recycle();
		busyObjects.Remove(t); // 将对象从忙碌列表中剔除
		idleObjects.Enqueue(t); // 将回收后的对象推进闲置队列
	}
}
```

上面代码里，我们使用两个容器来存储闲置和忙碌的对象。当对象池大小达到上限的时候就不予创建新实例了，而是根据设置的策略，或是忽略该次请求，返回空对象，或是强制回收一个忙碌对象。

对于`Ignore`策略来说，每次获取对象的时候需要注意空对象异常，增加了啰嗦的代码内容。而对于`Rob`策略来说，虽然强制回收了一个已使用的对象，但是在实际的应用中，比如粒子系统中，玩家是不太可能会注意到之前的一个闪光粒子突然消失了，反之，如果一个对刚刚操作提供反馈的对象没有出现更容易被发现，所以个人推荐使用`Rob`策略。

# 缺点
目前我们已经搭建了一个较为完备的对象池模型了，然而如果将需求细致到Unity开发中，我们不得不考虑到`GameObject`与`MonoBehaviour`对象独特的生命周期，同时也需要考虑使用一个全局的单例类对所有的对象池进行自动的创建，加载与销毁。这些部分就放到下篇来说了。