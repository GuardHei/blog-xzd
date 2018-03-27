---
layout: post
title: Unity中实现通知系统
comments: true
date: 2018-02-13 07:50:48.000000000 +09:00
author: William Xie
tags: Unity
---
# 概述

游戏过程中经常会有消息进行通知。比如说解锁了成就，获得了奖励，或者说某一步操作成功。简单说说我是怎么实现的。

## 实现

首先，我们要创建一个`NotificationManager`的脚本，继承自`MonoBehaviour`。由于在游戏中只有一个通知系统，而且存在于多个场景里，所以在`Start()`方法里，我们要加一句`DontDestroyOnLoad(gameObject)`;保证切换场景的时候不被销毁。我们再创建一个类`NotificationModel`作为输入的数据结构，拥有以下属性；

1. Title 标题 string public
2. Type 种类（警告，提示，成就解锁等等）enum public
3. Content 内容 string public
4. Duration 时长 float public

现在，我们在`NotificationManager`里维护一个静态的通知队列来存储通知信息。
```csharp
static Queue<NotificationModel> notifications = new Queue<NotificationModel>();
```

这时我们就可以添加一个AddNotification()添加通知的方法了。
```csharp
public static void AddNotification(NotificationModel model) {
    notifications.Enqeue(model);
    if (notifications.Count == 1) {
        // do display work;
    }
}
```

非常简单的代码，就是把`model`推进队列里，如果队列里的通知数等于1，即只有这条新消息的时候就进行展示的工作。
接下来我们就写一下展示的代码`DisplayNotification()`;
```csharp
static void DisplayNotification() {
    // Load Up Notification Bar
    // 刷新显示通知条ui，具体代码就不写了
    notificationBar.SetActive(true); // 激活通知条ui
    gameObject.GetComponent<NotificationManager>().StartCoroutine(ExeDisplayTask(notifications.Peek()));
}

static IEnumerator ExeDisplayTask(NotificationModel model) {
    yield return new WaitForSeconds(model.duration); // 过一段时间使通知消失
    notifications.Deqeue(); // 让通知从队列里弹出
    notificationBar.SetActive(false); // 休眠通知条ui
    if (notifications.Count > 0) { // 通知数量大于0，说明还有通知可以展示
        DisplayNotification();
    }
}
```

这样基本就搞定了一个通知系统，注意的是最好把通知系统的`Canvas`作为`Notification Manager`搭载`GameObject`的子物体，这样使`Canvas`也不会跟随场景销毁。为了方便以后调用，可以把常用的配置写成常量或者默认值，然后`Overload`一下`AddNotification()`方法，比如说快捷的一个参数的方法。这里面的单例用法有点奇怪，更好用的方法我会在后面的博客里说 -- `MonoBehaviour`的单例。
