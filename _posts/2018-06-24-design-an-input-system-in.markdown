---
layout: post
title: Unity中输入系统的设计
comments: true
date: 2018-06-24 07:50:48.000000000 +09:00
author: William Xie
tags: Unity
---

# 概述
>游戏，作为一种交互式艺术，玩家的操作是必不可少的。事实上，对于游戏中操作系统的设计也是非常耗费精力的。本篇博文不是从游戏设计角度上来叙述怎样设计游戏的操作，而是从程序设计的角度提出一种更通用和抽象的输入系统。

PS：本篇内容特别多，本来是想拆成两篇的，但考虑到内容的连贯，还是放到一篇里解决。

# Unity的输入系统
作为一个非常强大的游戏引擎，Unity已经为开发者提供了一层高级的输入抽象。大部分的开发我们都使用`UnityEngine.Input`类来解决问题。`Input`类提供了诸如按键检测以及不同操作平台的适配，甚至还有陀螺仪，定位等输入，可以说对于简单的需求来说，基本都可以满足。为了更好的对输入进行统一和抽象，Unity引进了`InputManager`系统，提出虚拟按键与虚拟轴的概念，对给类型的输入进行统一管理，提高开发效率。当然，Unity也提供了`UnityEngine.ClusterInput`类，作为一个相对底层一些的输入封装，不过在一般的开发过程中不会使用到，所以不与讨论。

## 问题
既然Unity已经提供了如此完备和方便的输入系统，为什么我们还要自己再进行改进设计呢？这主要还是得考虑到这套输入系统设计的初衷。准确的说，这套系统仅仅提供了一个完备的输入检测模块，然而对于怎样将输入操作模块结合到不同的项目中，它是没有考虑的。这是什么意思呢？想想看，如果开发者被要求开发一个抛小球的游戏，玩家可以控制游戏里的人物抛出小球，那么，简单来说，代码大概会长成这个样子吧：

```csharp
\\ 这边以及后面的部分代码片段仅做实例用途，没有真实含义
\\ 省略using

public class PlayerController() {

    public GameObject ballPrototype;

    private void Update() {
        if (Input.GetKeyDown("Return")) {
            GameObject ball = Instantiate(ballPrototype, transform);
            ball.GetComponent<Rigidbody>().AddForce(new Vector3(1, 0, 0));
        }
    }
}
```

这段简单的代码实现了当玩家按下回车键的时候，游戏人物会向x轴正方向扔出一个小球。当然，如果使用Unity的`InputManager`系统的话，我们可以现在`InputManager`界面定义一个`Throw`的虚拟按键，再选定其正向输入(Positive Input)接受为`Return`。最后，判断代码就会变成这样：`Input.GetButtonDown("Throw")`。这其实已经提高了不少开发效率。想想看，往往操作系统的设计和实现是由策划和程序分别制作的，当策划想要修改操作按键的时候，不可能每次都向程序员要求修改源码。并且，每次修改源码后的编译时间也降低了开发效率，故而，这种方式是有效的。

但还有一个问题没有解决。如果现在一个新的需求被提出，当玩家按不同的键的时候，小球可以被礽向不同方向，于是辛勤的开发者把代码调整如下：

```csharp
\\ 省略其他无关重复内容，只有Update方法

private void Update() {
    if (Input.GetButtonDown("Throw Up")) {
        GameObject ball = Instantiate(ballPrototype, transform);
        ball.GetComponent<Rigidbody>().AddForce(new Vector3(0, 1, 0));
    } else if (Input.GetButtonDown("Throw Down")) {
        GameObject ball = Instantiate(ballPrototype, transform);
        ball.GetComponent<Rigidbody>().AddForce(new Vector3(0, -1, 0));
    } else if (Input.GetButtonDown("Throw Right")) {
        GameObject ball = Instantiate(ballPrototype, transform);
        ball.GetComponent<Rigidbody>().AddForce(new Vector3(1, 0, 0));
    } else if (Input.GetButtonDown("Throw Left")) {
        GameObject ball = Instantiate(ballPrototype, transform);
        ball.GetComponent<Rigidbody>().AddForce(new Vector3(-1, 0, 0));
    }
}
```

嗯，看到这么多`if else`结构是不是感觉代码瞬间复杂了许多？这时候如果策划在想开发人员提出使用游戏手柄的遥杆控制呢？

```csharp
\\ 省略其他无关重复内容，只有Update方法

private void Update() {
    if (Input.GetButtonDown("Throw Up")) {
        GameObject ball = Instantiate(ballPrototype, transform);
        ball.GetComponent<Rigidbody>().AddForce(new Vector3(0, 1, 0));
    } else if (Input.GetButtonDown("Throw Down")) {
        GameObject ball = Instantiate(ballPrototype, transform);
        ball.GetComponent<Rigidbody>().AddForce(new Vector3(0, -1, 0));
    } else if (Input.GetButtonDown("Throw Right")) {
        GameObject ball = Instantiate(ballPrototype, transform);
        ball.GetComponent<Rigidbody>().AddForce(new Vector3(1, 0, 0));
    } else if (Input.GetButtonDown("Throw Left")) {
        GameObject ball = Instantiate(ballPrototype, transform);
        ball.GetComponent<Rigidbody>().AddForce(new Vector3(-1, 0, 0));
    } else if (Input.GetAxis("Vertical Axis") > 0) {
        GameObject ball = Instantiate(ballPrototype, transform);
        ball.GetComponent<Rigidbody>().AddForce(new Vector3(0, 1, 0));
    } else if (Input.GetAxis("Vertical Axis") < 0) {
        GameObject ball = Instantiate(ballPrototype, transform);
        ball.GetComponent<Rigidbody>().AddForce(new Vector3(0, -1, 0));
    } else if (Input.GetAxis("Horizontal Axis") > 0) {
        GameObject ball = Instantiate(ballPrototype, transform);
        ball.GetComponent<Rigidbody>().AddForce(new Vector3(1, 0, 0));
    } else if (Input.GetAxis("Horizontal Axis") < 0) {
        GameObject ball = Instantiate(ballPrototype, transform);
        ball.GetComponent<Rigidbody>().AddForce(new Vector3(-1, 0, 0));
    }
}
```

哇，现在`Update`方法变得又长又复杂，当另一个开发人员想要修改的时候，不得不先逐一理清每个逻辑分支的关系，否则修改很有可能导致意想不到的bug。同时，策划又想到了一个主意：根据按键被按下的时间来决定抛出球的力度。我们可怜的程序员只好继续干活：

```csharp
\\ 由于真的这么写实在是太繁复，所以作者也只能选一个操作示意一下

float pressTime;

private void Update() {
    if (Input.GetButtonDown("Throw Up")) {
        pressTime = Time.time;
    } else if (Input.GetButtonUp("Throw Up")) {
        float pressDuration = Time.time - pressTime;
        GameObject ball = Instantiate(ballPrototype, transform);
        ball.GetComponent<Rigidbody>().AddForce(new Vector3(0, 1 * pressDuration, 0));
    }
    // 省略
}
```

看看，就连本文作者也被逼得无可奈何了，不过我们万恶的策划还没有满足，他又提出了一个要求：让玩家可以自定义操作。蛤？这该怎么办？当开发人员花了半天时间翻遍了Unity的开发文档后，他惊异地发现，这么大的一个`InputManager`系统，竟然没有一个暴露出来的代码接口，仅仅是一个`.asset`文件（我绝对不是在以亲身经历吐槽）。那么，我们该如何自定义按键呢？对了，对于键盘的输入，我们可以使用Unity提供的`KeyCode`来自定义映射，但手柄摇杆呢？这可完全没办法呀！

综上所述，我们急需一种可以和游戏其他系统更好互动的输入系统。

## 解决方法

### Input System
事实上Unity的开发团队也意识到了这个问题，所以自从Unity 5开始，他们就推出了一款实验性质的输入系统`Input System`。该系统使用一个`string`对不同输入源进行映射，提供了代码的接口，因此对于策划和程序都是十分友好的（需要自己设计一套本地存储的输入配置解析系统，不过这东西，一个json文件就搞定了，不属于大问题）。但是！（忍不住又要吐槽）虽然现在都有了Unity 2018版本，这个系统仍处于Preview状态，想要使用的话，还需要去GitHub上下载导入项目。最最最重要的是：首先，这套系统更新的比较快，之前的教程基本已经废弃，许多坑找不到解决方案；其次，作为一个Preview项目，Unity不负众望的给它添加了非常非常非常非常多的bug（比如Mac平台上根本无法读取蓝牙手柄），虽然已经在GitHub上开了不少issue，不管显然，没什么人在解决。所以我们还是自己弄一套吧！

### 自己设计

#### 参考借鉴
在设计前，我们不妨参考参考其他游戏引擎是怎样实现输入模块的。给我印象最深的是虚幻4的输入设计，虚幻4使用了函数指针的方式对输入接受和输入处理进行了绑定，这样一来，我们就可以摆脱无尽的`if else`噩梦了。`Input System`也使用了类似的手法，不过由于C#并没有指针的概念，开发人员使用了委托（delegate）和事件（event）进行操作绑定（难得符合ms规范一次）。事实上，对事件的支持其实比单纯的函数指针更方便，因为有时候同一个按键在不同的情形下往往有不同的输入处理，不过这对程序员提出了更高的要求，毕竟event产生的内存泄漏是非常常见的。

同样，为了方便策划的快速改动，我们也需要提供一个字符串的映射，或者说，输入源的序列化。不过这一点由于Unity的封装，搞起来蛮dt的，但在商业项目里又是不得不使用的。

#### 代码框架

这样一来我们的最后设计好的代码框架如下：

|名称|类型|用途|
|:-:|:-:|:-:|
|IInputDetectable|interface|输入源监听的接口|
|InputDetector|abstract class|所有输入源监听的抽象父类，实现了IInputDetectable，添加了一些字段属性|
|KeyDetector|class|继承自InputDetector，实现对按键的监听（包括键盘和手柄按键）|
|AxisDetector|class|继承自InputDetector，实现对手柄摇杆的监听|
|InputMapper|class|继承自MonoBehaviour，提供虚拟按键与输入源，输入处理与虚拟按键的绑定|

#### 代码实现

现在我们来想想一个输入源需要被监听哪些内容？为了能让这篇博文能够在一定篇幅内结束，我们稍稍简化一些需求，只需要实现对所有输入源进行如下的监听即可：

1. 是否被按下
2. 是否保持按下
3. 释放被释放
4. 被按下多久

对于轴类的输入源，如摇杆，我们对其取一个方向，如果在改方向的偏移大于设定的敏感值，则算被按下。举个例子，如果我们定义输入源：`horizontal axis - positive`，设定敏感值（即死区Deadzone）为0.05，如果检测到该轴的偏移是正向的，且距离大于0.05，才算该轴进入被按下的状态。

因此，`IInputDetectable`接口的定义如下：

```csharp
public interface IInputDetectable {

	string Name {
		get;
	}

	bool IsPressed {
		get;
	}

	bool IsHeld {
		get;
	}

	bool IsReleased {
		get;
	}
	
	float ChargeTime {
		get;
	}

	void Refresh();
}
```

我们用属性访问器作为接口的暴露。同时，为了提供序列化的功能，方便策划快速配置，我们再添加一个`Name`属性。而最后的`Refresh()`方法则顾名思义，负责对被监听的输入源的状态进行刷新。

由于接口的限制，我们不能对一些必要的字段进行定义，因此，我们还需要再添加一个抽象类`InputDetector`（事实上可以直接跳过接口，将所有方法置于抽象类中，这里仅仅是为了更好的展示逻辑关系而特意添加了一个接口）。`InputDetector`的定义如下：

```csharp
using UnityEngine;

public abstract class InputDetector : IInputDetectable {
	
	protected string _name;
	protected bool _isPressed;
	protected bool _isHeld;
	protected bool _isReleased;
	protected float _lastPressTime = -1f;

	public string Name {
		get {
			return _name;
		}
	}

	public bool IsPressed {
		get {
			return _isPressed;
		}
	}

	public bool IsHeld {
		get {
			return _isHeld;
		}
	}

	public bool IsReleased {
		get {
			return _isReleased;
		}
	}

	public float ChargeTime {
		get {
			return Time.time - _lastPressTime;
		}
	}
	
	public abstract void Refresh();
}
```

我们对每个属性访问器都添加了私有成员变量。并且，我们通过添加`_lastPressTime`字段，实现了输入源被按下的时间长短的监听。由于对于不同的输入源的监听有不一样的规则，`Refresh()`方法仍然保持抽象。

下面，我们先对`KeyDetector`进行实现。因为可以直接使用Unity自带的`Input`类进行判断，所以代码相对简单：

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;

public sealed class KeyDetector : InputDetector {

	private static readonly Dictionary<string, KeyDetector> keyDetectors;

    private readonly KeyCode _keyCode;

	private KeyDetector(string name) {
		_name = name;
		keyCode = name.ToEnum<KeyCode>();
	}

	private KeyDetector(KeyCode keyCode) {
		_name = Enum.GetName(typeof(KeyCode), keyCode);
		this.keyCode = keyCode;
	}

	public override void Refresh() {
		_isPressed = false;
		_isReleased = false;
		if (Input.GetKeyDown(keyCode)) {
			_isPressed = true;
			_isHeld = true;
			_lastPressTime = Time.time;
		} else if (Input.GetKeyUp(keyCode)) {
			_isReleased = true;
			_isHeld = false;
		}
	}

	public static KeyDetector ToKeyDetector(string name) {
		return keyDetectors.ContainsKey(name) ? keyDetectors[name] : null;
	}

	public static bool operator ==(KeyDetector lhs, KeyDetector rhs) {
		return lhs.keyCode == rhs.keyCode;
	}

	public static bool operator !=(KeyDetector lhs, KeyDetector rhs) {
		return lhs.keyCode != rhs.keyCode;
	}
}
```

我们将该类声明为`sealed`的原因仅仅是因为我们知道不会有类再继承它，因此将其闭合，对于使用`il2cpp`的开发者来说，可以将该类中的虚函数转为普通函数进行调用，降低开销（但其实不会有太多的差异）。

先忽略掉第一个字典对象`keyDetectors`，看到第一个成员字段`_keyCode`，这个指明了被监听的输入源。在第一个构造器中，我们获取到输入源的序列化字符串，再将其与`KeyCode`这个枚举类对应，获得输入源。对于第二个构造器，我们只是把颠倒了下顺序而已。对于`Refresh()`方法的实现，我们直接使用了`Input`类提供的方法，不做详解。接下来，我们忽略掉第一个静态方法，看到两个操作符重载，我们将原来`Object`的hashcode比较改为枚举值`_keyCode`的比较。由于枚举天然的`not nullable`特性，我们不需要进行额外的空指针判断。

好了，现在来说说刚刚被忽略掉的两个东西。由于每个输入源的实体都是唯一的，如果对每个输入源生成了多个监听对象就是一种浪费，也可能造成内存泄漏，因此，我们将所有可用的输入源监听器都先一步静态化存储起来，做出类似枚举的效果。最后，我们再将每个监听器与起输入源的序列化字符串通过字典关联起来。当需要获取输入源监听器时，通过`ToKeyDetector(string name)`方法获得唯一的监听器。最后，再将构造器声明为`private`，不被外部类创建实例。这样，我们还要加上如下一段的代码在该类里作为静态枚举：

```csharp
    #region
	public static readonly KeyDetector A = new KeyDetector(KeyCode.A);
	public static readonly KeyDetector B = new KeyDetector(KeyCode.B);
	public static readonly KeyDetector C = new KeyDetector(KeyCode.C);
	public static readonly KeyDetector D = new KeyDetector(KeyCode.D);
	public static readonly KeyDetector E = new KeyDetector(KeyCode.E);
	public static readonly KeyDetector F = new KeyDetector(KeyCode.F);
	public static readonly KeyDetector G = new KeyDetector(KeyCode.G);
	public static readonly KeyDetector H = new KeyDetector(KeyCode.H);
	public static readonly KeyDetector I = new KeyDetector(KeyCode.I);
	public static readonly KeyDetector J = new KeyDetector(KeyCode.J);
	public static readonly KeyDetector K = new KeyDetector(KeyCode.K);
	public static readonly KeyDetector L = new KeyDetector(KeyCode.L);
	public static readonly KeyDetector M = new KeyDetector(KeyCode.M);
	public static readonly KeyDetector N = new KeyDetector(KeyCode.N);
	public static readonly KeyDetector O = new KeyDetector(KeyCode.O);
	public static readonly KeyDetector P = new KeyDetector(KeyCode.P);
	public static readonly KeyDetector Q = new KeyDetector(KeyCode.Q);
	public static readonly KeyDetector R = new KeyDetector(KeyCode.R);
	public static readonly KeyDetector S = new KeyDetector(KeyCode.S);
	public static readonly KeyDetector T = new KeyDetector(KeyCode.T);
	public static readonly KeyDetector U = new KeyDetector(KeyCode.U);
	public static readonly KeyDetector V = new KeyDetector(KeyCode.V);
	public static readonly KeyDetector W = new KeyDetector(KeyCode.W);
	public static readonly KeyDetector X = new KeyDetector(KeyCode.X);
	public static readonly KeyDetector Y = new KeyDetector(KeyCode.Y);
	public static readonly KeyDetector Z = new KeyDetector(KeyCode.Z);
	public static readonly KeyDetector F1 = new KeyDetector(KeyCode.F1);
	public static readonly KeyDetector F2 = new KeyDetector(KeyCode.F2);
	public static readonly KeyDetector F3 = new KeyDetector(KeyCode.F3);
	public static readonly KeyDetector F4 = new KeyDetector(KeyCode.F4);
	public static readonly KeyDetector F5 = new KeyDetector(KeyCode.F5);
	public static readonly KeyDetector F6 = new KeyDetector(KeyCode.F6);
	public static readonly KeyDetector F7 = new KeyDetector(KeyCode.F7);
	public static readonly KeyDetector F8 = new KeyDetector(KeyCode.F8);
	public static readonly KeyDetector F9 = new KeyDetector(KeyCode.F9);
	public static readonly KeyDetector F10 = new KeyDetector(KeyCode.F10);
	public static readonly KeyDetector F11 = new KeyDetector(KeyCode.F11);
	public static readonly KeyDetector F12 = new KeyDetector(KeyCode.F12);
	public static readonly KeyDetector F13 = new KeyDetector(KeyCode.F13);
	public static readonly KeyDetector F14 = new KeyDetector(KeyCode.F14);
	public static readonly KeyDetector F15 = new KeyDetector(KeyCode.F15);
	public static readonly KeyDetector Alpha0 = new KeyDetector(KeyCode.Alpha0);
	public static readonly KeyDetector Alpha1 = new KeyDetector(KeyCode.Alpha1);
	public static readonly KeyDetector Alpha2 = new KeyDetector(KeyCode.Alpha2);
	public static readonly KeyDetector Alpha3 = new KeyDetector(KeyCode.Alpha3);
	public static readonly KeyDetector Alpha4 = new KeyDetector(KeyCode.Alpha4);
	public static readonly KeyDetector Alpha5 = new KeyDetector(KeyCode.Alpha5);
	public static readonly KeyDetector Alpha6 = new KeyDetector(KeyCode.Alpha6);
	public static readonly KeyDetector Alpha7 = new KeyDetector(KeyCode.Alpha7);
	public static readonly KeyDetector Alpha8 = new KeyDetector(KeyCode.Alpha8);
	public static readonly KeyDetector Alpha9 = new KeyDetector(KeyCode.Alpha9);
	public static readonly KeyDetector UpArrow = new KeyDetector(KeyCode.UpArrow);
	public static readonly KeyDetector DownArrow = new KeyDetector(KeyCode.DownArrow);
	public static readonly KeyDetector LeftArrow = new KeyDetector(KeyCode.LeftArrow);
	public static readonly KeyDetector RightArrow = new KeyDetector(KeyCode.RightArrow);
	public static readonly KeyDetector LeftControl = new KeyDetector(KeyCode.LeftControl);
	public static readonly KeyDetector RightControl = new KeyDetector(KeyCode.RightControl);
	public static readonly KeyDetector LeftShift = new KeyDetector(KeyCode.LeftShift);
	public static readonly KeyDetector RightShift = new KeyDetector(KeyCode.RightShift);
	public static readonly KeyDetector LeftCommand = new KeyDetector(KeyCode.LeftCommand);
	public static readonly KeyDetector RightCommand = new KeyDetector(KeyCode.RightCommand);
	public static readonly KeyDetector LeftApple = new KeyDetector(KeyCode.LeftApple);
	public static readonly KeyDetector RightApple = new KeyDetector(KeyCode.RightApple);
	public static readonly KeyDetector LeftWindows = new KeyDetector(KeyCode.LeftWindows);
	public static readonly KeyDetector RightWindows = new KeyDetector(KeyCode.RightWindows);
	public static readonly KeyDetector LeftAlt = new KeyDetector(KeyCode.LeftAlt);
	public static readonly KeyDetector RightAlt = new KeyDetector(KeyCode.RightAlt);
	public static readonly KeyDetector LeftBracket = new KeyDetector(KeyCode.LeftBracket);
	public static readonly KeyDetector RightBracket = new KeyDetector(KeyCode.RightBracket);
	public static readonly KeyDetector Escape = new KeyDetector(KeyCode.Escape);
	public static readonly KeyDetector BackQuote = new KeyDetector(KeyCode.BackQuote);
	public static readonly KeyDetector Backslash = new KeyDetector(KeyCode.Backslash);
	public static readonly KeyDetector Minus = new KeyDetector(KeyCode.Minus);
	public static readonly KeyDetector Equals = new KeyDetector(KeyCode.Equals);
	public static readonly KeyDetector Comma = new KeyDetector(KeyCode.Comma);
	public static readonly KeyDetector Period = new KeyDetector(KeyCode.Period);
	public static readonly KeyDetector Slash = new KeyDetector(KeyCode.Slash);
	public static readonly KeyDetector Backspace = new KeyDetector(KeyCode.Backspace);
	public static readonly KeyDetector Tab = new KeyDetector(KeyCode.Tab);
	public static readonly KeyDetector Space = new KeyDetector(KeyCode.Space);
	public static readonly KeyDetector Semicolon = new KeyDetector(KeyCode.Semicolon);
	public static readonly KeyDetector Quote = new KeyDetector(KeyCode.Quote);
	public static readonly KeyDetector Return = new KeyDetector(KeyCode.Return);
	public static readonly KeyDetector CapsLock = new KeyDetector(KeyCode.CapsLock);
	public static readonly KeyDetector JoystickButton0 = new KeyDetector(KeyCode.JoystickButton0);
	public static readonly KeyDetector JoystickButton1 = new KeyDetector(KeyCode.JoystickButton1);
	public static readonly KeyDetector JoystickButton2 = new KeyDetector(KeyCode.JoystickButton2);
	public static readonly KeyDetector JoystickButton3 = new KeyDetector(KeyCode.JoystickButton3);
	public static readonly KeyDetector JoystickButton4 = new KeyDetector(KeyCode.JoystickButton4);
	public static readonly KeyDetector JoystickButton5 = new KeyDetector(KeyCode.JoystickButton5);
	public static readonly KeyDetector JoystickButton6 = new KeyDetector(KeyCode.JoystickButton6);
	public static readonly KeyDetector JoystickButton7 = new KeyDetector(KeyCode.JoystickButton7);
	public static readonly KeyDetector JoystickButton8 = new KeyDetector(KeyCode.JoystickButton8);
	public static readonly KeyDetector JoystickButton9 = new KeyDetector(KeyCode.JoystickButton9);
	public static readonly KeyDetector JoystickButton10 = new KeyDetector(KeyCode.JoystickButton10);
	public static readonly KeyDetector JoystickButton11 = new KeyDetector(KeyCode.JoystickButton11);
	public static readonly KeyDetector JoystickButton12 = new KeyDetector(KeyCode.JoystickButton12);
	public static readonly KeyDetector JoystickButton13 = new KeyDetector(KeyCode.JoystickButton13);
	public static readonly KeyDetector JoystickButton14 = new KeyDetector(KeyCode.JoystickButton14);
	public static readonly KeyDetector JoystickButton15 = new KeyDetector(KeyCode.JoystickButton15);
	public static readonly KeyDetector JoystickButton16 = new KeyDetector(KeyCode.JoystickButton16);
	public static readonly KeyDetector JoystickButton17 = new KeyDetector(KeyCode.JoystickButton17);
	public static readonly KeyDetector JoystickButton18 = new KeyDetector(KeyCode.JoystickButton18);
	public static readonly KeyDetector JoystickButton19 = new KeyDetector(KeyCode.JoystickButton19);
	public static readonly KeyDetector Joystick1Button0 = new KeyDetector(KeyCode.Joystick1Button0);
	public static readonly KeyDetector Joystick1Button1 = new KeyDetector(KeyCode.Joystick1Button1);
	public static readonly KeyDetector Joystick1Button2 = new KeyDetector(KeyCode.Joystick1Button2);
	public static readonly KeyDetector Joystick1Button3 = new KeyDetector(KeyCode.Joystick1Button3);
	public static readonly KeyDetector Joystick1Button4 = new KeyDetector(KeyCode.Joystick1Button4);
	public static readonly KeyDetector Joystick1Button5 = new KeyDetector(KeyCode.Joystick1Button5);
	public static readonly KeyDetector Joystick1Button6 = new KeyDetector(KeyCode.Joystick1Button6);
	public static readonly KeyDetector Joystick1Button7 = new KeyDetector(KeyCode.Joystick1Button7);
	public static readonly KeyDetector Joystick1Button8 = new KeyDetector(KeyCode.Joystick1Button8);
	public static readonly KeyDetector Joystick1Button9 = new KeyDetector(KeyCode.Joystick1Button9);
	public static readonly KeyDetector Joystick1Button10 = new KeyDetector(KeyCode.Joystick1Button10);
	public static readonly KeyDetector Joystick1Button11 = new KeyDetector(KeyCode.Joystick1Button11);
	public static readonly KeyDetector Joystick1Button12 = new KeyDetector(KeyCode.Joystick1Button12);
	public static readonly KeyDetector Joystick1Button13 = new KeyDetector(KeyCode.Joystick1Button13);
	public static readonly KeyDetector Joystick1Button14 = new KeyDetector(KeyCode.Joystick1Button14);
	public static readonly KeyDetector Joystick1Button15 = new KeyDetector(KeyCode.Joystick1Button15);
	public static readonly KeyDetector Joystick1Button16 = new KeyDetector(KeyCode.Joystick1Button16);
	public static readonly KeyDetector Joystick1Button17 = new KeyDetector(KeyCode.Joystick1Button17);
	public static readonly KeyDetector Joystick1Button18 = new KeyDetector(KeyCode.Joystick1Button18);
	public static readonly KeyDetector Joystick1Button19 = new KeyDetector(KeyCode.Joystick1Button19);
	public static readonly KeyDetector Joystick2Button0 = new KeyDetector(KeyCode.Joystick2Button0);
	public static readonly KeyDetector Joystick2Button1 = new KeyDetector(KeyCode.Joystick2Button1);
	public static readonly KeyDetector Joystick2Button2 = new KeyDetector(KeyCode.Joystick2Button2);
	public static readonly KeyDetector Joystick2Button3 = new KeyDetector(KeyCode.Joystick2Button3);
	public static readonly KeyDetector Joystick2Button4 = new KeyDetector(KeyCode.Joystick2Button4);
	public static readonly KeyDetector Joystick2Button5 = new KeyDetector(KeyCode.Joystick2Button5);
	public static readonly KeyDetector Joystick2Button6 = new KeyDetector(KeyCode.Joystick2Button6);
	public static readonly KeyDetector Joystick2Button7 = new KeyDetector(KeyCode.Joystick2Button7);
	public static readonly KeyDetector Joystick2Button8 = new KeyDetector(KeyCode.Joystick2Button8);
	public static readonly KeyDetector Joystick2Button9 = new KeyDetector(KeyCode.Joystick2Button9);
	public static readonly KeyDetector Joystick2Button10 = new KeyDetector(KeyCode.Joystick2Button10);
	public static readonly KeyDetector Joystick2Button11 = new KeyDetector(KeyCode.Joystick2Button11);
	public static readonly KeyDetector Joystick2Button12 = new KeyDetector(KeyCode.Joystick2Button12);
	public static readonly KeyDetector Joystick2Button13 = new KeyDetector(KeyCode.Joystick2Button13);
	public static readonly KeyDetector Joystick2Button14 = new KeyDetector(KeyCode.Joystick2Button14);
	public static readonly KeyDetector Joystick2Button15 = new KeyDetector(KeyCode.Joystick2Button15);
	public static readonly KeyDetector Joystick2Button16 = new KeyDetector(KeyCode.Joystick2Button16);
	public static readonly KeyDetector Joystick2Button17 = new KeyDetector(KeyCode.Joystick2Button17);
	public static readonly KeyDetector Joystick2Button18 = new KeyDetector(KeyCode.Joystick2Button18);
	public static readonly KeyDetector Joystick2Button19 = new KeyDetector(KeyCode.Joystick2Button19);
	public static readonly KeyDetector Joystick3Button0 = new KeyDetector(KeyCode.Joystick3Button0);
	public static readonly KeyDetector Joystick3Button1 = new KeyDetector(KeyCode.Joystick3Button1);
	public static readonly KeyDetector Joystick3Button2 = new KeyDetector(KeyCode.Joystick3Button2);
	public static readonly KeyDetector Joystick3Button3 = new KeyDetector(KeyCode.Joystick3Button3);
	public static readonly KeyDetector Joystick3Button4 = new KeyDetector(KeyCode.Joystick3Button4);
	public static readonly KeyDetector Joystick3Button5 = new KeyDetector(KeyCode.Joystick3Button5);
	public static readonly KeyDetector Joystick3Button6 = new KeyDetector(KeyCode.Joystick3Button6);
	public static readonly KeyDetector Joystick3Button7 = new KeyDetector(KeyCode.Joystick3Button7);
	public static readonly KeyDetector Joystick3Button8 = new KeyDetector(KeyCode.Joystick3Button8);
	public static readonly KeyDetector Joystick3Button9 = new KeyDetector(KeyCode.Joystick3Button9);
	public static readonly KeyDetector Joystick3Button10 = new KeyDetector(KeyCode.Joystick3Button10);
	public static readonly KeyDetector Joystick3Button11 = new KeyDetector(KeyCode.Joystick3Button11);
	public static readonly KeyDetector Joystick3Button12 = new KeyDetector(KeyCode.Joystick3Button12);
	public static readonly KeyDetector Joystick3Button13 = new KeyDetector(KeyCode.Joystick3Button13);
	public static readonly KeyDetector Joystick3Button14 = new KeyDetector(KeyCode.Joystick3Button14);
	public static readonly KeyDetector Joystick3Button15 = new KeyDetector(KeyCode.Joystick3Button15);
	public static readonly KeyDetector Joystick3Button16 = new KeyDetector(KeyCode.Joystick3Button16);
	public static readonly KeyDetector Joystick3Button17 = new KeyDetector(KeyCode.Joystick3Button17);
	public static readonly KeyDetector Joystick3Button18 = new KeyDetector(KeyCode.Joystick3Button18);
	public static readonly KeyDetector Joystick3Button19 = new KeyDetector(KeyCode.Joystick3Button19);
	public static readonly KeyDetector Joystick4Button0 = new KeyDetector(KeyCode.Joystick4Button0);
	public static readonly KeyDetector Joystick4Button1 = new KeyDetector(KeyCode.Joystick4Button1);
	public static readonly KeyDetector Joystick4Button2 = new KeyDetector(KeyCode.Joystick4Button2);
	public static readonly KeyDetector Joystick4Button3 = new KeyDetector(KeyCode.Joystick4Button3);
	public static readonly KeyDetector Joystick4Button4 = new KeyDetector(KeyCode.Joystick4Button4);
	public static readonly KeyDetector Joystick4Button5 = new KeyDetector(KeyCode.Joystick4Button5);
	public static readonly KeyDetector Joystick4Button6 = new KeyDetector(KeyCode.Joystick4Button6);
	public static readonly KeyDetector Joystick4Button7 = new KeyDetector(KeyCode.Joystick4Button7);
	public static readonly KeyDetector Joystick4Button8 = new KeyDetector(KeyCode.Joystick4Button8);
	public static readonly KeyDetector Joystick4Button9 = new KeyDetector(KeyCode.Joystick4Button9);
	public static readonly KeyDetector Joystick4Button10 = new KeyDetector(KeyCode.Joystick4Button10);
	public static readonly KeyDetector Joystick4Button11 = new KeyDetector(KeyCode.Joystick4Button11);
	public static readonly KeyDetector Joystick4Button12 = new KeyDetector(KeyCode.Joystick4Button12);
	public static readonly KeyDetector Joystick4Button13 = new KeyDetector(KeyCode.Joystick4Button13);
	public static readonly KeyDetector Joystick4Button14 = new KeyDetector(KeyCode.Joystick4Button14);
	public static readonly KeyDetector Joystick4Button15 = new KeyDetector(KeyCode.Joystick4Button15);
	public static readonly KeyDetector Joystick4Button16 = new KeyDetector(KeyCode.Joystick4Button16);
	public static readonly KeyDetector Joystick4Button17 = new KeyDetector(KeyCode.Joystick4Button17);
	public static readonly KeyDetector Joystick4Button18 = new KeyDetector(KeyCode.Joystick4Button18);
	public static readonly KeyDetector Joystick4Button19 = new KeyDetector(KeyCode.Joystick4Button19);
	#endregion 

	#region
	static KeyDetector() {
		keyDetectors = new Dictionary<string, KeyDetector>(184);
		keyDetectors["A"] = A;
		keyDetectors["B"] = B;
		keyDetectors["C"] = C;
		keyDetectors["D"] = D;
		keyDetectors["E"] = E;
		keyDetectors["F"] = F;
		keyDetectors["G"] = G;
		keyDetectors["H"] = H;
		keyDetectors["I"] = I;
		keyDetectors["J"] = J;
		keyDetectors["K"] = K;
		keyDetectors["L"] = L;
		keyDetectors["M"] = M;
		keyDetectors["N"] = N;
		keyDetectors["O"] = O;
		keyDetectors["P"] = P;
		keyDetectors["Q"] = Q;
		keyDetectors["R"] = R;
		keyDetectors["S"] = S;
		keyDetectors["T"] = T;
		keyDetectors["U"] = U;
		keyDetectors["V"] = V;
		keyDetectors["W"] = W;
		keyDetectors["X"] = X;
		keyDetectors["Y"] = Y;
		keyDetectors["Z"] = Z;
		keyDetectors["F1"] = F1;
		keyDetectors["F2"] = F2;
		keyDetectors["F3"] = F3;
		keyDetectors["F4"] = F4;
		keyDetectors["F5"] = F5;
		keyDetectors["F6"] = F6;
		keyDetectors["F7"] = F7;
		keyDetectors["F8"] = F8;
		keyDetectors["F9"] = F9;
		keyDetectors["F10"] = F10;
		keyDetectors["F11"] = F11;
		keyDetectors["F12"] = F12;
		keyDetectors["F13"] = F13;
		keyDetectors["F14"] = F14;
		keyDetectors["F15"] = F15;
		keyDetectors["Alpha0"] = Alpha0;
		keyDetectors["Alpha1"] = Alpha1;
		keyDetectors["Alpha2"] = Alpha2;
		keyDetectors["Alpha3"] = Alpha3;
		keyDetectors["Alpha4"] = Alpha4;
		keyDetectors["Alpha5"] = Alpha5;
		keyDetectors["Alpha6"] = Alpha6;
		keyDetectors["Alpha7"] = Alpha7;
		keyDetectors["Alpha8"] = Alpha8;
		keyDetectors["Alpha9"] = Alpha9;
		keyDetectors["UpArrow"] = UpArrow;
		keyDetectors["DownArrow"] = DownArrow;
		keyDetectors["LeftArrow"] = LeftArrow;
		keyDetectors["RightArrow"] = RightArrow;
		keyDetectors["LeftControl"] = LeftControl;
		keyDetectors["RightControl"] = RightControl;
		keyDetectors["LeftShift"] = LeftShift;
		keyDetectors["RightShift"] = RightShift;
		keyDetectors["LeftCommand"] = LeftCommand;
		keyDetectors["RightCommand"] = RightCommand;
		keyDetectors["LeftApple"] = LeftApple;
		keyDetectors["RightApple"] = RightApple;
		keyDetectors["LeftWindows"] = LeftWindows;
		keyDetectors["RightWindows"] = RightWindows;
		keyDetectors["LeftAlt"] = LeftAlt;
		keyDetectors["RightAlt"] = RightAlt;
		keyDetectors["LeftBracket"] = LeftBracket;
		keyDetectors["RightBracket"] = RightBracket;
		keyDetectors["Escape"] = Escape;
		keyDetectors["BackQuote"] = BackQuote;
		keyDetectors["Backslash"] = Backslash;
		keyDetectors["Minus"] = Minus;
		keyDetectors["Equals"] = Equals;
		keyDetectors["Comma"] = Comma;
		keyDetectors["Period"] = Period;
		keyDetectors["Slash"] = Slash;
		keyDetectors["Backspace"] = Backspace;
		keyDetectors["Tab"] = Tab;
		keyDetectors["Space"] = Space;
		keyDetectors["Semicolon"] = Semicolon;
		keyDetectors["Quote"] = Quote;
		keyDetectors["Return"] = Return;
		keyDetectors["CapsLock"] = CapsLock;
		keyDetectors["JoystickButton0"] = JoystickButton0;
		keyDetectors["JoystickButton1"] = JoystickButton1;
		keyDetectors["JoystickButton2"] = JoystickButton2;
		keyDetectors["JoystickButton3"] = JoystickButton3;
		keyDetectors["JoystickButton4"] = JoystickButton4;
		keyDetectors["JoystickButton5"] = JoystickButton5;
		keyDetectors["JoystickButton6"] = JoystickButton6;
		keyDetectors["JoystickButton7"] = JoystickButton7;
		keyDetectors["JoystickButton8"] = JoystickButton8;
		keyDetectors["JoystickButton9"] = JoystickButton9;
		keyDetectors["JoystickButton10"] = JoystickButton10;
		keyDetectors["JoystickButton11"] = JoystickButton11;
		keyDetectors["JoystickButton12"] = JoystickButton12;
		keyDetectors["JoystickButton13"] = JoystickButton13;
		keyDetectors["JoystickButton14"] = JoystickButton14;
		keyDetectors["JoystickButton15"] = JoystickButton15;
		keyDetectors["JoystickButton16"] = JoystickButton16;
		keyDetectors["JoystickButton17"] = JoystickButton17;
		keyDetectors["JoystickButton18"] = JoystickButton18;
		keyDetectors["JoystickButton19"] = JoystickButton19;
		keyDetectors["Joystick1Button0"] = Joystick1Button0;
		keyDetectors["Joystick1Button1"] = Joystick1Button1;
		keyDetectors["Joystick1Button2"] = Joystick1Button2;
		keyDetectors["Joystick1Button3"] = Joystick1Button3;
		keyDetectors["Joystick1Button4"] = Joystick1Button4;
		keyDetectors["Joystick1Button5"] = Joystick1Button5;
		keyDetectors["Joystick1Button6"] = Joystick1Button6;
		keyDetectors["Joystick1Button7"] = Joystick1Button7;
		keyDetectors["Joystick1Button8"] = Joystick1Button8;
		keyDetectors["Joystick1Button9"] = Joystick1Button9;
		keyDetectors["Joystick1Button10"] = Joystick1Button10;
		keyDetectors["Joystick1Button11"] = Joystick1Button11;
		keyDetectors["Joystick1Button12"] = Joystick1Button12;
		keyDetectors["Joystick1Button13"] = Joystick1Button13;
		keyDetectors["Joystick1Button14"] = Joystick1Button14;
		keyDetectors["Joystick1Button15"] = Joystick1Button15;
		keyDetectors["Joystick1Button16"] = Joystick1Button16;
		keyDetectors["Joystick1Button17"] = Joystick1Button17;
		keyDetectors["Joystick1Button18"] = Joystick1Button18;
		keyDetectors["Joystick1Button19"] = Joystick1Button19;
		keyDetectors["Joystick2Button0"] = Joystick2Button0;
		keyDetectors["Joystick2Button1"] = Joystick2Button1;
		keyDetectors["Joystick2Button2"] = Joystick2Button2;
		keyDetectors["Joystick2Button3"] = Joystick2Button3;
		keyDetectors["Joystick2Button4"] = Joystick2Button4;
		keyDetectors["Joystick2Button5"] = Joystick2Button5;
		keyDetectors["Joystick2Button6"] = Joystick2Button6;
		keyDetectors["Joystick2Button7"] = Joystick2Button7;
		keyDetectors["Joystick2Button8"] = Joystick2Button8;
		keyDetectors["Joystick2Button9"] = Joystick2Button9;
		keyDetectors["Joystick2Button10"] = Joystick2Button10;
		keyDetectors["Joystick2Button11"] = Joystick2Button11;
		keyDetectors["Joystick2Button12"] = Joystick2Button12;
		keyDetectors["Joystick2Button13"] = Joystick2Button13;
		keyDetectors["Joystick2Button14"] = Joystick2Button14;
		keyDetectors["Joystick2Button15"] = Joystick2Button15;
		keyDetectors["Joystick2Button16"] = Joystick2Button16;
		keyDetectors["Joystick2Button17"] = Joystick2Button17;
		keyDetectors["Joystick2Button18"] = Joystick2Button18;
		keyDetectors["Joystick2Button19"] = Joystick2Button19;
		keyDetectors["Joystick3Button0"] = Joystick3Button0;
		keyDetectors["Joystick3Button1"] = Joystick3Button1;
		keyDetectors["Joystick3Button2"] = Joystick3Button2;
		keyDetectors["Joystick3Button3"] = Joystick3Button3;
		keyDetectors["Joystick3Button4"] = Joystick3Button4;
		keyDetectors["Joystick3Button5"] = Joystick3Button5;
		keyDetectors["Joystick3Button6"] = Joystick3Button6;
		keyDetectors["Joystick3Button7"] = Joystick3Button7;
		keyDetectors["Joystick3Button8"] = Joystick3Button8;
		keyDetectors["Joystick3Button9"] = Joystick3Button9;
		keyDetectors["Joystick3Button10"] = Joystick3Button10;
		keyDetectors["Joystick3Button11"] = Joystick3Button11;
		keyDetectors["Joystick3Button12"] = Joystick3Button12;
		keyDetectors["Joystick3Button13"] = Joystick3Button13;
		keyDetectors["Joystick3Button14"] = Joystick3Button14;
		keyDetectors["Joystick3Button15"] = Joystick3Button15;
		keyDetectors["Joystick3Button16"] = Joystick3Button16;
		keyDetectors["Joystick3Button17"] = Joystick3Button17;
		keyDetectors["Joystick3Button18"] = Joystick3Button18;
		keyDetectors["Joystick3Button19"] = Joystick3Button19;
		keyDetectors["Joystick4Button0"] = Joystick4Button0;
		keyDetectors["Joystick4Button1"] = Joystick4Button1;
		keyDetectors["Joystick4Button2"] = Joystick4Button2;
		keyDetectors["Joystick4Button3"] = Joystick4Button3;
		keyDetectors["Joystick4Button4"] = Joystick4Button4;
		keyDetectors["Joystick4Button5"] = Joystick4Button5;
		keyDetectors["Joystick4Button6"] = Joystick4Button6;
		keyDetectors["Joystick4Button7"] = Joystick4Button7;
		keyDetectors["Joystick4Button8"] = Joystick4Button8;
		keyDetectors["Joystick4Button9"] = Joystick4Button9;
		keyDetectors["Joystick4Button10"] = Joystick4Button10;
		keyDetectors["Joystick4Button11"] = Joystick4Button11;
		keyDetectors["Joystick4Button12"] = Joystick4Button12;
		keyDetectors["Joystick4Button13"] = Joystick4Button13;
		keyDetectors["Joystick4Button14"] = Joystick4Button14;
		keyDetectors["Joystick4Button15"] = Joystick4Button15;
		keyDetectors["Joystick4Button16"] = Joystick4Button16;
		keyDetectors["Joystick4Button17"] = Joystick4Button17;
		keyDetectors["Joystick4Button18"] = Joystick4Button18;
		keyDetectors["Joystick4Button19"] = Joystick4Button19;
    }
    #endregion
```

把与字典的关联放入静态构造器里实现。确实，这样看上去非常费力的做法，但是好处很明显，直接就可以根据字符串获得对应的输入源监听器。不过还能怎么做呢？其实也很简单，对于`_keyCode`这个枚举值来说，我们可以获取起所有的`name`，然后通过一个循环写入字典内。不过这里之所以一一列举，还有个原因就是硬编码的时候方便编辑器检查拼写错误。实际的项目开发中大概是没有这种硬编码做法的，所以可以无视。

然后，我们实现对于轴类输入源的监听：

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;

public sealed class AxisDetector : InputDetector {
	
	private static readonly Dictionary<string, AxisDetector> axisDetectors;

	#region
	public static readonly AxisDetector Axis1stPositive = new AxisDetector("1st axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis2ndPositive = new AxisDetector("2nd axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis3rdPositive = new AxisDetector("3rd axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis4thPositive = new AxisDetector("4th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis5thPositive = new AxisDetector("5th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis6thPositive = new AxisDetector("6th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis7thPositive = new AxisDetector("7th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis8thPositive = new AxisDetector("8th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis9thPositive = new AxisDetector("9th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis10thPositive = new AxisDetector("10th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis11thPositive = new AxisDetector("11th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis12thPositive = new AxisDetector("12th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis13thPositive = new AxisDetector("13th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis14thPositive = new AxisDetector("14th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis15thPositive = new AxisDetector("15th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis16thPositive = new AxisDetector("16th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis17thPositive = new AxisDetector("17th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis18thPositive = new AxisDetector("18th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis19thPositive = new AxisDetector("19th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis20thPositive = new AxisDetector("20th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis21stPositive = new AxisDetector("21st axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis22ndPositive = new AxisDetector("22nd axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis23rdPositive = new AxisDetector("23rd axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis24thPositive = new AxisDetector("24th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis25thPositive = new AxisDetector("25th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis26thPositive = new AxisDetector("26th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis27thPositive = new AxisDetector("27th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis28thPositive = new AxisDetector("28th axis", AxisDirection.Positive);
	public static readonly AxisDetector Axis1stNegative = new AxisDetector("1st axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis2ndNegative = new AxisDetector("2nd axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis3rdNegative = new AxisDetector("3rd axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis4thNegative = new AxisDetector("4th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis5thNegative = new AxisDetector("5th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis6thNegative = new AxisDetector("6th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis7thNegative = new AxisDetector("7th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis8thNegative = new AxisDetector("8th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis9thNegative = new AxisDetector("9th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis10thNegative = new AxisDetector("10th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis11thNegative = new AxisDetector("11th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis12thNegative = new AxisDetector("12th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis13thNegative = new AxisDetector("13th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis14thNegative = new AxisDetector("14th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis15thNegative = new AxisDetector("15th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis16thNegative = new AxisDetector("16th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis17thNegative = new AxisDetector("17th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis18thNegative = new AxisDetector("18th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis19thNegative = new AxisDetector("19th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis20thNegative = new AxisDetector("20th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis21stNegative = new AxisDetector("21st axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis22ndNegative = new AxisDetector("22nd axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis23rdNegative = new AxisDetector("23rd axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis24thNegative = new AxisDetector("24th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis25thNegative = new AxisDetector("25th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis26thNegative = new AxisDetector("26th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis27thNegative = new AxisDetector("27th axis", AxisDirection.Negative);
	public static readonly AxisDetector Axis28thNegative = new AxisDetector("28th axis", AxisDirection.Negative);
	#endregion

	#region
	static AxisDetector() {
		axisDetectors = new Dictionary<string, AxisDetector>(57);
		axisDetectors["Axis1st Positive"] = Axis1stPositive;
		axisDetectors["Axis2nd Positive"] = Axis2ndPositive;
		axisDetectors["Axis3rd Positive"] = Axis3rdPositive;
		axisDetectors["Axis4th Positive"] = Axis4thPositive;
		axisDetectors["Axis5th Positive"] = Axis5thPositive;
		axisDetectors["Axis6th Positive"] = Axis6thPositive;
		axisDetectors["Axis7th Positive"] = Axis7thPositive;
		axisDetectors["Axis8th Positive"] = Axis8thPositive;
		axisDetectors["Axis9th Positive"] = Axis9thPositive;
		axisDetectors["Axis10th Positive"] = Axis10thPositive;
		axisDetectors["Axis11th Positive"] = Axis11thPositive;
		axisDetectors["Axis12th Positive"] = Axis12thPositive;
		axisDetectors["Axis13th Positive"] = Axis13thPositive;
		axisDetectors["Axis14th Positive"] = Axis14thPositive;
		axisDetectors["Axis15th Positive"] = Axis15thPositive;
		axisDetectors["Axis16th Positive"] = Axis16thPositive;
		axisDetectors["Axis17th Positive"] = Axis17thPositive;
		axisDetectors["Axis18th Positive"] = Axis18thPositive;
		axisDetectors["Axis19th Positive"] = Axis19thPositive;
		axisDetectors["Axis20th Positive"] = Axis20thPositive;
		axisDetectors["Axis21st Positive"] = Axis21stPositive;
		axisDetectors["Axis22nd Positive"] = Axis22ndPositive;
		axisDetectors["Axis23rd Positive"] = Axis23rdPositive;
		axisDetectors["Axis24th Positive"] = Axis24thPositive;
		axisDetectors["Axis25th Positive"] = Axis25thPositive;
		axisDetectors["Axis26th Positive"] = Axis26thPositive;
		axisDetectors["Axis27th Positive"] = Axis27thPositive;
		axisDetectors["Axis28th Positive"] = Axis28thPositive;
		axisDetectors["Axis1st Negative"] = Axis1stNegative;
		axisDetectors["Axis2nd Negative"] = Axis2ndNegative;
		axisDetectors["Axis3rd Negative"] = Axis3rdNegative;
		axisDetectors["Axis4th Negative"] = Axis4thNegative;
		axisDetectors["Axis5th Negative"] = Axis5thNegative;
		axisDetectors["Axis6th Negative"] = Axis6thNegative;
		axisDetectors["Axis7th Negative"] = Axis7thNegative;
		axisDetectors["Axis8th Negative"] = Axis8thNegative;
		axisDetectors["Axis9th Negative"] = Axis9thNegative;
		axisDetectors["Axis10th Negative"] = Axis10thNegative;
		axisDetectors["Axis11th Negative"] = Axis11thNegative;
		axisDetectors["Axis12th Negative"] = Axis12thNegative;
		axisDetectors["Axis13th Negative"] = Axis13thNegative;
		axisDetectors["Axis14th Negative"] = Axis14thNegative;
		axisDetectors["Axis15th Negative"] = Axis15thNegative;
		axisDetectors["Axis16th Negative"] = Axis16thNegative;
		axisDetectors["Axis17th Negative"] = Axis17thNegative;
		axisDetectors["Axis18th Negative"] = Axis18thNegative;
		axisDetectors["Axis19th Negative"] = Axis19thNegative;
		axisDetectors["Axis20th Negative"] = Axis20thNegative;
		axisDetectors["Axis21st Negative"] = Axis21stNegative;
		axisDetectors["Axis22nd Negative"] = Axis22ndNegative;
		axisDetectors["Axis23rd Negative"] = Axis23rdNegative;         
		axisDetectors["Axis24th Negative"] = Axis24thNegative;
		axisDetectors["Axis25th Negative"] = Axis25thNegative;
		axisDetectors["Axis26th Negative"] = Axis26thNegative;
		axisDetectors["Axis27th Negative"] = Axis27thNegative;
		axisDetectors["Axis28th Negative"] = Axis28thNegative;
	}
	#endregion

	private float _deadZone;
	private AxisDirection _direction;
	private float _offset;

	public AxisDirection Direction {
		get { return _direction; }
	}

	public float Offset {
		get { return _offset; }
	}

	public AxisDetector(string name, AxisDirection direction, float deadZone = .15f) {
		_name = name;
		_direction = direction;
		_deadZone = deadZone;
	}

	public override void Refresh() {
		_offset = Input.GetAxis(_name);
		_isPressed = false;
		_isReleased = false;
		bool flag = _offset * (int) _direction - _deadZone > 0;
		if (_isHeld) {
			if (!flag) {
				_isReleased = true;
				_isHeld = false;
			}
		} else {
			if (flag) {
				_isPressed = true;
				_isHeld = true;
				_lastPressTime = Time.time;
			}
		}
	}

	public static AxisDetector ToAxisDetector(string name) {
		return axisDetectors.ContainsKey(name) ? axisDetectors[name] : null;
	}

	public static bool operator ==(AxisDetector lhs, AxisDetector rhs) {
		return lhs._name.Equals(rhs._name, StringComparison.CurrentCultureIgnoreCase);
	}

	public static bool operator !=(AxisDetector lhs, AxisDetector rhs) {
		return !lhs._name.Equals(rhs._name, StringComparison.CurrentCultureIgnoreCase);
	}
}

public enum AxisDirection {
	Negative = -1,
	Positive = 1
}
```

总的来说基本思想差不多，但是有不少区别。首先，就是我之前提到的Unity的不便之处，所有的轴类输入源只有先在`Input Manager`里注册过才能监听。因此，我们得先在`Input Manager`里加入所有的轴，并根据一定的规则命名（方便序列化）。同时，由于轴有方向性，我们还需要在初始化的时候传递一个方向参数，这里枚举之所以使用-1和1的值是为了提高运行效率。看到刷新方法内，我们需要知道该轴有没有在给定方向上偏移超过敏感值的距离。我们当然可以用`if else`比较两种方向的情况，但是我们知道`if else`是缓存不友好的，容易造成cpu误判。因此，我们直接将偏移值乘上距离的整数值，就可以全都转为绝对值进行比较，可以提高300倍左右的速度（这个测的误差比较大，而且其实`if else`也蛮快的，两校的情况下没什么差别）。当然，我们也提供了轴类输入源独特的熟悉，偏移值，的暴露，方便蓄力之类的操作。

最后我们再来看看`InputMapper`的实现。这里，我们也使用虚拟按键的想法，方便输入系统和其他系统的解耦。因此，我们还需要提供两套绑定，一套为虚拟按键与输入源监听器的绑定，一套为虚拟按键与输入处理的绑定。这样`InputMapper`的定义如下：

```csharp
using System.Collections.Generic;
using UnityEngine;

public class InputMapper {

	public const string THROW_UP = "Throw Up";
    public const string THROW_UP = "Throw Down";
    public const string THROW_UP = "Throw Left";
    public const string THROW_UP = "Throw Right";
	
	private static readonly Dictionary<string, InputDetector> defaultKeyboardMap;
    private static readonly Dictionary<string, InputDetector> defaultJoystickMap;
	
	public delegate void OnPressed();
	public delegate void OnHeld();
	public delegate void OnReleased();

	public int Count {
		get { return _inputMap.Count; }
	}
	
	private bool _isInControl;
	private readonly Dictionary<string, OnPressed> _onPressedBindings;
	private readonly Dictionary<string, OnHeld> _onHeldBindings;
	private readonly Dictionary<string, OnReleased> _onReleasedBindings;
	private readonly Dictionary<string, InputDetector> _inputMap;

	private void Start() {
        defaultKeyboardMap = new Dictionary<string, InputDetector>(4);
		defaultKeyboardMap[THROW_UP] = KeyDetector.W;
        defaultKeyboardMap[THROW_DOWN] = KeyDetector.S;
        defaultKeyboardMap[THROW_LEFT] = KeyDetector.A;
        defaultKeyboardMap[THROW_RIGHT] = KeyDetector.D;
        
        defaultJoystickMap = new Dictionary<string, InputDetector>(4);
        defaultJoystickMap[THROW_UP] = AxisDetector.Axis2ndPositive;
        defaultJoystickMap[THROW_DOWN] = AxisDetector.Axis2ndNegative;
        defaultJoystickMap[THROW_LEFT] = AxisDetector.Axis1stPositive;
        defaultJoystickMap[THROW_RIGHT] = AxisDetector.Axis1stNegative;

        _inputMap = new Dictionary<string, InputDetector>(4);
		_onPressedBindings = new Dictionary<string, OnPressed>(4);
		_onHeldBindings = new Dictionary<string, OnHeld>(4);
		_onReleasedBindings = new Dictionary<string, OnReleased>(4);
		Reset();
	}

	public void Update() {
		Refresh();
		if (_isInControl) {
			foreach (string name in _onPressedBindings.Keys) {
				if (_inputMap[name].IsPressed) {
					_onPressedBindings[name]();
				}
			}
			foreach (string name in _onHeldBindings.Keys) {
				if (_inputMap[name].IsHeld) {
					_onHeldBindings[name]();
				}
			}
			foreach (string name in _onReleasedBindings.Keys) {
				if (_inputMap[name].IsReleased) _onReleasedBindings[name]();
			}
		}
	}
	
	public void Refresh() {
		foreach (InputDetector detector in _inputMap.Values) {
			if (detector == null) {
				break;
			}
			detector.Refresh();
		}
	}

	public void Reset() {
		_inputMap.Clear();
		foreach (var pair in KeyboardMap) {
			Remap(pair.Key, pair.Value);
		}
	}

	public void Remap(string name, InputDetector detector) {
		if (_inputMap.ContainsKey(name)) {
			new UnityException("Already Contains Input Named [" + name + "] !");
			return;
		}

		_inputMap[name] = detector;
	}
	
	public bool BindPressEvent(string name, OnPressed e) {
		bool binded = _onPressedBindings.ContainsKey(name);
		_onPressedBindings[name] = e;
		return binded;
	}

	public bool BindHoldEvent(string name, OnHeld e) {
		bool binded = _onHeldBindings.ContainsKey(name);
		_onHeldBindings[name] = e;
		return binded;
	}

	public bool BindReleaseEvent(string name, OnReleased e) {
		bool binded = _onReleasedBindings.ContainsKey(name);
		_onReleasedBindings[name] = e;
		return binded;
	}

	public bool UnbindPressEvent(string name) {
		if (!_onPressedBindings.ContainsKey(name)) {
			return false;
		}
		_onPressedBindings.Remove(name);
		return true;
	}

	public bool UnbindHoldEvent(string name) {
		if (!_onHeldBindings.ContainsKey(name)) {
			return false;
		}
		_onHeldBindings.Remove(name);
		return true;
	}

	public bool UnbindReleaseEvent(string name) {
		if (!_onReleasedBindings.ContainsKey(name)) {
			return false;
		}
		_onReleasedBindings.Remove(name);
		return true;
	}

	public OnPressed GetPressEvent(string name) {
		if (_onPressedBindings.ContainsKey(name)) {
			return _onPressedBindings[name];
		}
		return null;
	}

	public OnHeld GetHoldEvent(string name) {
		if (_onHeldBindings.ContainsKey(name)) {
			return _onHeldBindings[name];
		}
		return null;
	}

	public OnReleased GetReleaseEvent(string name) {
		if (_onReleasedBindings.ContainsKey(name)) {
			return _onReleasedBindings[name];
		}
		return null;
	}

	public InputDetector this[string name] {
		get {
			if (!_inputMap.ContainsKey(name)) {
				new UnityException("Cannot Find Input Named [" + name + "] !");
				return null;
			}
			return _inputMap[name];
		}
		set { Remap(name, value); }
	}
}
```

我们和`Input Manager`一样，使用字符串作为虚拟按键。为了方便编辑器检查拼写错误，我们使用常量将这些字符串定义下来。对于每个输入源来说，我们对其被按下，被按住以及被释放的事件进行处理，因此声明三种委托对象`void OnPressed()`，`void OnHeld()`，`void OnReleased()`（偷个懒，释放事件就不加被按下时间长度的参数了，如果要加类似这样`void OnReleased(float duration)`）。对于虚拟按键，我们提供`Remap(string name, InputDetector detector)`方法将虚拟按键与监听器绑定至一个字典内。再提供一个`Reset()`方法进行默认绑定的加载。对于输入事件，我们也是通过字典将虚拟按键与委托进行绑定。最后提供`Refresh()`方法对输入源状态进行更新。我们提供一个`_isInControl`变量表示该脚本是否被起用。无论是否改脚本被启用，我们都需要每帧刷新输入源的监听状态。这是为了防止在脚本被禁止的时候，输入源产生了状态改变，但是由于没有及时更新，导致后面再次启用的时候引起一系列异常。然而只有在脚本启用时，我们才对绑定的事件进行激活。

#### 使用例子

下面我们以上面抛小球为例子，重新实现策划的需求：

```csharp
using UnityEngine;

[RequireComponent(typeof(InputMapper))]
public class PlayerController {

    public GameObject ballPrototype;

    private InputMapper _inputMapper;

    private void Start() {
        _inputMapper = GetComponent<InputMapper>();

        _inputMapper.BindReleaseEvent(InputMapper.THROW_UP, delegate {
            ThrowBall(Vector3.up * _inputMapper[InputMapper.THROW_UP].ChargeTime);
        });
        _inputMapper.BindReleaseEvent(InputMapper.THROW_DOWN, delegate {
            ThrowBall(Vector3.up * _inputMapper[InputMapper.THROW_DOWN].ChargeTime);
        });
        _inputMapper.BindReleaseEvent(InputMapper.THROW_LEFT, delegate {
            ThrowBall(Vector3.left * _inputMapper[InputMapper.THROW_LEFT].ChargeTime);
        });
        _inputMapper.BindReleaseEvent(InputMapper.THROW_RIGHT, delegate {
            ThrowBall(Vector3.right * _inputMapper[InputMapper.THROW_RIGHT].ChargeTime);
        });
    }

    private void ThrowBall(Vector3 force) {
        GameObject ball = Instantiate(ballPrototype, transform);
        ball.GetComponent<Rigidbody>.AddForce(force);
    }
}
```

看上去代码是不是简单清晰了许多？当玩家想要启用游戏手柄的时候，只要简单加一行代码将`defaultJoystickMap`映射到绑定集里即可。

## 缺点

目前这个框架仍然比较简陋。首先，我们没有对绑定的方法进行空指针处理，也就是说我们信任程序员不会传一个`null`的委托对象作为事件处理绑定进来。同样的，由于使用了字符串作为虚拟按键，如果策划输入了错误的内容，有可能导致不正确的绑定。在例子中，我们简单的用`const`常量解决这个问题。但是，最重要的问题依旧是序列化的问题，即如何将字符串转为输入源监听器。由于Unity的`Input Manager`的设计，我们不得不先手动囊括所有的`axis`，再进行检测。这不仅增加了工作量，也降低了运行效率，毕竟对于引擎来说，每一帧都要对每个虚拟按键进行状态更新。