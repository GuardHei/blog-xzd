---
layout: post
title: Unity游戏开发优化 —— 通用对象池与复用（二）
comments: true
date: 2018-06-11 07:50:48.000000000 +09:00
author: William Xie
tags: Unity
---

# 概述
>上一篇博文实现了一个简单的对象池基类，对所有继承自`IReusable`的对象实例进行回收利用。然而在实际的开发过程中，不少Unity独有的`GameObject`对象也需要进行对象池优化，比如说FPS游戏里的枪口火花以及各种反复出现的粒子特效，甚至是批量出现的NPC敌人。由于Unity机制的特殊性，`GameObject`对象的生命周期被Unity托管，无法通过构造器初始化等等。这样一来我们就需要对之前的对象池进行改进。

##
注意！
本篇博文的代码风格骤变，主要是最近看了一些`C#`规范，自己也就稍微注意了点。

## 如何解决？
想一想，当我们创建一个`GameObject`的时候，我们并没有使用`new`关键字，而是使用`MonoBehaviour`的静态方法`Instantiate()`。这样来我们就不得不把新的对象池组件挂到场景里去，这就产生关于`MonoBehaviour`单例实现的问题，不过这并不是本文的关键，所以不做详细解释。第二个问题在于`GameObject`对象的状态回收与初始化。`GameObject`并没有提供任何可供初始化的接口，事实上我们对他们的初始化往往也是来自外部，这样一来我们可以考虑使用委托的方式对`GameObject`实例进行初始化和回收的操作。

## 改进对象池
根据上述提出的一些问题和解决思路，适用于`GameObject`的对象池代码如下：

```csharp
public class GameObjectPool : MonoBehaviour {

    public int Size {
		get {
			return idleObjects.Count + busyObjects.Count;
		}
	}

    private bool _ready;
    private int _maxCapacity;
    private string _loadPath;
    private Action<GameObject> _initAction;
    private Action<GameObject> _recycleAction;
    private ObjectPoolLimitOption _limitOption;

    private GameObject _prototype;
    private Queue<GameObject> _idleObjects;
    private List<GameObject> _busyObjects;

    public void Init(int maxCapacity, string loadPath, Action<GameObject> initAction, Action<GameObject> recycleAction, ObjectPoolLimitOption limitOption = ObjectPoolLimitOption.Rob) {
        if (_ready) return;

        _maxCapacity = maxCapacity;
        _loadPath = loadPath;
        _prototype = Resource.Load<GameObject>(_loadPath);
        _initAction = initAction;
        _recycleAction = recycleAction;
        _limitOption = limitOption;
        _idleObjects = new Queue<GameObject>(_maxCapacity);
        _busyObjects = new List<GameObject>(_maxCapacity);

        _ready = true;
    }

    public GameObject Get() {
        if (_ready) return null;

        if (idleObjects.Count == 0) {
			if (Size < _maxCapacity) {
				_idleObjects.Enqueue(Instantiate(_prototype));
				return Get();
			} else if (_limitOption == ObjectPoolLimitOption.Ignore) {
				return null; // 忽略，所以返回null
			} else {
				GameObject obj = _busyObjects[0]; // 获得第一个使用中对象
				Recycle(obj); // 强制回收
				return Get();
			}
		} else {
			GameObject obj = _idleObjects.Dequeue(); // 挑出对象
			_busyObjects.Add(obj); // 放入忙碌列表
			_initAction(obj);
			return obj;
		}
	}

    public void Recycle(GameObject obj) {
        if (_ready) {
            _recycleAction(obj);
            _busyObjects.Remove(obj);
            _idleObjects.Enqueue(obj);
        }
    }
}

public enum ObjectPoolMaximumOption {
	Ignore,
	Rob
}
```

上面代码里，由于我们有特定的`GameObject`对象需求，我们并不需要使用泛型。同时，我们移除了对象池的`Instantiate()`方法，而是用`MonoBehaviour`的静态方法完成。对于初始化与回收，我们通过两个带参无返回值的委托`Action<GameObject>`解决。由于整个脚本是继承自`MonoBehaviour`的，我们不得不加一个`Init(int, string, Action<GameObject>, Action<GameObject>, ObjectPoolLimitOption)`的方法进行初始化。这样一来，我们需要加一个`_ready`变量鉴定改对象池是否初始化过。如果没有初始化过，`Get()`就返回`null`，而`Recycle(GameObject)`就不进行操作。为了降低每次从硬盘读取数据的消耗，初始化对象池时，提前加载一个原型`Prefab`进入内存，方便后面生成对象实例。

# 缺点
首先是这个对象池的生命周期管理。由于初始化`GameObject`的需要，我们不得不继承自`MonoBehaviour`，并且放置于场景之中。这就导致我们需要手动对对象池初始化。对于一些全局的对象池，有可能虽然对象池是`DontDestroyOnLoad(GameObject)`，但是`GameObject`对象的`parent`父对象是场景关联的，因此导致切换场景后发生错误。因此，我们需要对加载与卸载场景的事件进行监听，当检测到场景卸载时可以考虑清空对象队列或是将所有全局对象池生成的对象也归到`DontDestroyOnLoad`的场景下面。