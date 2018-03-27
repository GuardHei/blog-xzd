---
layout: post
title: Unity中实现在Inspector上对Monobehaviour脚本方法的选择（不使用UnityAction）
comments: true
date: 2018-03-26 07:50:48.000000000 +09:00
author: William Xie
tags: Unity
---
# 概述
>游戏开发中时动态调用函数非常常见，前面一篇文章里我们讨论了Delegate, Reflection和UnityAction的性能比对，并推荐大家尽可能使用Delegate与Reflection结合的方案。然而这样有个问题，如果我们想让项目组里的非程序员同志来编辑场景或者人物行为（比如说美工大佬或者策划大佬），我们需要暴露一个能够进行可视化操作的接口放在Unity的Inspector面板上，也就是说，如何在Inspector上面显示并选择MonoBehaviour所包含的方法。当然使用Unity库里的UnityEvent，UnityAction的话，系统是会自动进行序列化的。但是我们前面发现了UnityAction有着不是十分理想的性能，所以我们还是自己写一个InspectorUI的方法，

## 实现
PS: 一下代码有些库需要using但是没写出来。
首先我们使用继承Editor类来重写特定脚本的InspectorUI。为了叙述方便，我们假定拥有一个`MyBehaviour的`类，内容如下：
```csharp
public class MyBehaviour : MonoBehaviour {
    void Start() {}
    void Update() {}
    public void MyInstanceMethod() {
        Debug.Log("Call MyInstanceMethod !");
    }
    public static MyStaticMethod() {
        Debug.Log("Call MyStaticMethod !");
    }
}
```
好的，这样我们再针对这个脚本写一个`Editor`脚本，如下（有关`Unity Inspector`显示的具体教程请自行查找）：
```csharp
[ExecuteInEditMode] // 只能在编辑器下运行
[CanEditMultipleObjects] // 可以多个物体同时编辑
[CustomEditor(typeof(MyBehaviour))] // 声明需要显示的脚本MyBehaviour
public class MyBehaviourEditor : Editor {
    MyBehaviour script;
    Action selectedAction; // 假设我们需要的方法没有返回参数，简化一下流程
    void OnEable() {
        script = target as MyBehaviour; // target是父类中的对象，即需要显示的脚本
    }
    public override void OnInspectorGUI() {
        // base.OnInspectorGUI(); 不使用默认的显示方法
        EditorGUILayout.BeginVertical(); // 使用垂直布局
        // 我们即将处理的区域
        EditorGUILayout.EndVertical();
    }
}
```

想一想我们应该怎样解决这个问题，首先先确定需求，即应该是一个水平的UI容器，左边是标签注释和所属的脚本名，右侧是一个可以进行选择的下拉框，下拉框里是所有符合条件的方法以及一个空的方案，即不选择。

明确了需求，下面开始理一理思路，这个组件可以用一个Popup组件来做。我们首先通过反射获取钙脚本的所有方法信息（`MethodInfo`），然后进行筛选，只取出需要的，用它们的方法名创建一个`string`类型的数组作为`Popup`的显示选项，同时这个数组还要加一个空方法的元素（比如“`null`”）作为不选择方法的选项。

好的，下面开始编写相关代码：
```csharp
EditorGUILayout.BeginHorizontal(); // 使用水平布局
if (script != null) {
	int selectedIndex = 0; // 默认当前选项为第0个方法
	MethodInfo[] methods = script.GetType().GetMethods(BindingFlags.Public | BindingFlags.Instance); // 筛选出所有非public的实例方法，即非静态的protected和private的方法
	List<MethodInfo> methodList = new List<MethodInfo>();
	for (int i = 0; i < methods.Length; i++) {
		MethodInfo method = methods[i];
		if (!method.IsConstructor && method.GetParameters().Length == 0 && method.ReturnType == typeof(void)) { // 选出所有没有参数，没有返回值且不为构造函数的方法
			methodList.Add(method);
			if (selectedAction != null && selectedAction.method.name == method.Name) { // 如果selectedAction不为null，且方法名相同则说明这是当前选中的选项的序列号，这里我们不用考虑重载而导至方法名重复的问题，因为我们只保留没有参数的方法
				selectedIndex = methodList.Count; // 记录当前选中选项的序号，即List的末尾序号 + 1 （methodList.Count - 1 + 1），之所以+1是因为一会我们会在前面插一个不选的选项，所以索引要移动一个单位
			}
		}
	}
	string[] methodNames = new string[methodList.Count + 1]; // 数组大小是方法数量+1，因为有一个不选的选项
    methodNames[0] = "Null"; // 第一个选项为不选方法
	for (int i = 1; i < methodNames.Length; i++) {
		methodNames[i] = methodList[i].Name + " ( )"; // 将每个方法名添加到数组里，尾部加括号显得更美观
	}
    EditorGUILayout.Label("MyAction"); // 显示该选择框注释
	selectedIndex = EditorGUILayout.Popup(script.GetType().ToString(), selectedIndex, methodNames); // 显示Popup，参数分别为注释，当前选中的选项的序号，所有选项的名字
    if (selectedIndex != 0) { // 判断有没有选择方法
        selectedAction = (Action)Delegate.CreateDelegate(typeof(Action), target, methodList[selectedIndex]); // 如果选了方法就通过这种方式创建一个委托
    } else {
        selectedAction = null; // 如果不选就赋值为null
    }
} else {
       EditorGUILayout.HelpBox("Need A Object !", MessageType.Warning); // 如果script值为null就发出警报，当然目前是不用担心的
}
EditorGUILayout.EndHorizontal();
```

代码的逻辑稍微有点复杂，但是看着注释看懂应该不是大问题，下面我来解释一下几个可能让人困惑或者重要的地方。

首先，我们看这一处
```csharp
MethodInfo[] methods = script.GetType().GetMethods(BindingFlags.Public | BindingFlags.Instance);
```
这里是核心步骤之一，通过反射获取相关方法。我们通过`script.GetTpe()`获得该类的元数据，再通过`GetMethods(BindingFlags flag)`获得方法。这里就要说一说`GetMethods(BindingFlags flag)`的参数了，BindingFlags是一个枚举类型，且用一个`int32`来表示，这意味着一个BindingFlags的枚举对象可以是多个枚举选项的叠加，具体的关于位运算的操作请自行搜索，我们这里仅用到`|`运算符进行枚举项的叠加。那么我们假设需要获得所有的非public的实例方法，即非静态的protected和private的方法，那么BindingFlags参数就应该为`BindingFlags.Public | BindingFlags.Instance`。那如果需求有变，比如也许要显示静态方法怎麽办？很简单，再叠加一次就好`BindingFlags.Public | BindingFlags.Instance | BindingFlags.Static`。其他的枚举选项细节请自行查询`API`。

接下来有人可能在这里感觉迷惑，
```csharp
!method.IsConstructor && method.GetParameters().Length == 0 && method.ReturnType == typeof(void)
```
咦，这个`typeof(void)`是什么鬼，void也能作为类型吗？答案是，对的，`void`在这里就是类型。想想看，当你声明一个方法的时候，如果没有返回值是不是就写void？那么void不就应该是一种类型吗？当然，有人也会奇怪，那我为什么在`System`库里找不到这个类型呢？其实，这个类型是包含在`System`库里的，就是`System.Void`，然而由于这个`void`只在方法返回值的时候出现，所以微软的程序员把它在开发者使用`using`关键字导入库的时候隐藏掉了，于是我们就“找不到”它了。

最后，我们来看看创造Action对象的关键一步，
```csharp
selectedAction = (Action)Delegate.CreateDelegate(typeof(Action), script, methodList[selectedIndex]);
```
我们慢慢解析，`Delegate.CreateDelegate(Type type, Object target, MethodInfo methodInfo)`可以将一个给的MethodInfo对象转为一个Delegate对象。参数`Type type`为转换后的`Delegate`对象的类型，这里我们使用`Action`类型所以是`typeof(Action)`。第二个参数`Object target`是一个可选的参数，如果不填的话则说明你输入的是一个静态方法，填的话则需要填一个实例化的对象，即指定的挂载这个方法的对象。最后一个参数`MethodInfo methodInfo`就是反射得到的方法信息了。别忘了在前面加一个强制转换(`cast`)成`Action`类型，不然默认是`Delegate`类型的返回值。

## 封装
参考Unity自己提供的EditorGUILayout库，似乎UI都是通过一个方法来调用的，再通过返回值返回选项。这样的好处是复用性更强，那么我们就把它封装成一个静态的方法吧。
```csharp
public static Action MethodsPopup(string label, Object target, Action selectedAction, BindingFlags bindingFlags) {
    Action action = null;
	Layout.BeginHorizontal();
	if (target != null) {
		int selectedIndex = 0;
		MethodInfo[] methods = target.GetType().GetMethods(bindingFlags);
		List<MethodInfo> methodList = new List<MethodInfo>();
		for (int i = 0; i < methods.Length; i++) {
			MethodInfo method = methods[i];
			if (!method.IsConstructor && method.GetParameters().Length == 0 && method.ReturnType == typeof(void)) {
				methodList.Add(method);
				if (selectedAction.Length != 0 && selectedAction == method.Name) {
					selectedIndex = methodList.Count;
				}
			}
		}
		string[] methodNames = new string[methodList.Count + 1];
        methodNames[0] = "Null";
		for (int i = 0; i < methodNames.Length; i++) {
			methodNames[i] = methodList[i].Name + " ( )";
		}
		selectedIndex = Layout.Popup(label, selectedIndex, methodNames);
        if (selectedIndex != 0) {
            action = (Action)Delegate.CreateDelegate(typeof(Action), target, methodList[selectedIndex]);
        } else {
            action = null;
        }
	} else {
		Layout.HelpBox("Need A Object !", MessageType.Warning);
	}
	Layout.EndHorizontal();
    return action;
}
```

如果你想要更加简便地调用这个方法，那么就请自己设置默认值与方法重载吧！

## 问题
这个方法目前仍有不少问题，如下所示：
1. 无法设置方法参数
2. 无法利用返回值
3. 每帧都使用反射是比较耗性能的，如果机器比较差，该UI方法又被频繁调用，可能会造成卡顿
4. 无法选择方法所属的脚本，缺乏灵活性

## 改进
上述的问题中，有些是可以轻松解决的，有些则比较麻烦，有些我也没有好的方法或者思路，下面大致lie一下解决思路：
1. 使用`Unity`自带的`PropertyField()`对部分参数进行序列化，如果方法拥有不可序列化的参数话，就不予显示。使用`for`循环调用`PropertyField()`来显示所以参数。同时将返回值改为一个`Dictionary`类型对象，第一个对象为委托对象，后面可以为各个参数。然而这样无法解决动态指定委托接受参数的问题，仍需要通过大量重载来解决问题
2. 这我也没办法
3. 如果不封装的话可以在自己编写的继承`Editor`对象里放一个`List<MethodInfo>`对象来缓存方法，每帧比对`target`或者`script`对象有没有变化，如果变的话就进行更新（注意`null`的清形）。然而这种方法对于复用不是很合适，不可能每个`Editor`对象都要重写选择`Method`的代码。这样一来，我们可以考虑把这个UI封装成一个类，这样我们就可存储相关信息了，不管与`Unity`的代码风格就不太一样了
4. 这个的话可以再调用`PropertyField()`方法来选择脚本，如果侦测到变化就更新缓存的方法列表。值得注意的是，如果一行内容过多的话，显示会不完整，建议酌情考虑分行显示

# 总结
这里我的实现过程只是一个非常粗浅的封装，帮助大家启发思路，各位还要按照实际需求自行定制。