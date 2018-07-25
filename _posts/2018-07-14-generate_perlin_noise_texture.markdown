---
layout: post
title: 代码生成Perlin噪点图
comments: true
date: 2018-07-14 07:50:48.000000000 +09:00
author: William Xie
tags: Unity
---

# 概述
>最近在弄的游戏项目需要一些渲染特效，所以在弄shader。大部分实现复杂特效的shader都需要一个甚至多个噪点图进行随机采样。但网上往往下不到合适的噪点图，而且也不可能一张噪点图从头用到尾，所以我就翻了翻官方文档，弄了一个自动生成Perlin噪点图的编辑器拓展。

PS: 这期内容有点水，因为最近都在忙*this-is-ib*这个项目，好不容易抽点时间写篇博客（一月两更可不能忘🌚）。

# 思路
基本思路是使用`Mathf.PerlinNoise(float x, float y)`方法获得特定uv坐标的噪点值，转换为灰度写进纹理，然后将这个纹理保存在本地。我也懒得说废话了，直接上代码吧。

# 实现

```csharp
public static void GeneratePerlinNoiseMap(float scale) {
		string path = EditorUtility.SaveFilePanelInProject("生成柏林噪点图", "perlin_noise_texture_" + scale + "x", "png", "保存", Application.dataPath + "/Resources/Textures");
		if (!string.IsNullOrEmpty(path)) {
			if (EditorUtility.DisplayCancelableProgressBar("生成柏林噪点图", "初始化", 0f)) {
				ClearProgressBar();
				return;
			}
			int size = 256;
			int sizeSqr = size * size;
            Texture2D texture2D = new Texture2D(size, size);
			float oX = Random.value;
			float oY = Random.value;
			for (int i = 0; i < size; i++) {
				for (int j = 0; j < size; j++) {
					float greyScale = Mathf.PerlinNoise(oX + ((float) i) / ((float) size) * scale, ((float) j) / ((float) size) * scale);
					texture2D.SetPixel(i, j, new Color(greyScale, greyScale, greyScale));
					if (j % 100 == 0) {
						if (EditorUtility.DisplayCancelableProgressBar("生成柏林噪点图", greyScale.ToString(), (float) (size * i + j + 1) / sizeSqr)) {
							ClearProgressBar();
							return;
						}
					}
				}
			}
			texture2D.Apply();
			File.WriteAllBytes(path, texture2D.EncodeToPNG());
			EditorUtility.ClearProgressBar();
			AssetDatabase.ImportAsset(path.Substring(path.IndexOf("Assets")));
		}
	}
```

代码写得稍微有点魔幻，但也不是不能看😂，逐行分析吧。首先方法内第一句`string path = EditorUtility.SaveFilePanelInProject("生成柏林噪点图", "perlin_noise_texture_" + scale + "x", "png", "保存", Application.dataPath + "/Resources/Textures");`是调出保存文件的窗口，选择即将创建的噪点图保存位置。要注意的是最后一个参数是默认路径，得确保是确实存在的（没有就手动创建一个呗）。

然后就进入了一个大的`if`分支。这个是对上一部获取到的保存路径进行有效性检验。如果使用者（比如某策划）后悔进行了这个操作，在保存文件的窗口选择了取消，那返回的路径就是无效的。我们用`string.IsNullOrEmpty(string toTest)`进行有效性判断。

下面一行是调用编辑器的可取消进度条显示当前操作进度，接受一下返回值判断使用者有没有点击取消操作。我之所以用可取消进度条是因为之前作死弄了一个**10000x10000**分辨率的纹理操作，然后编辑器就卡死了😂，不得已只能强退。

好了，终于开始正式的内容了。我们先声明一下噪点纹理的大小，我准备都用**256x256**分辨率。这个分辨率在我目前的项目里精度足够了，而且大小也还好。然后用`sizeSqr`缓存一下总像素个数（用来在后面显示操作进度，不重要）。根据`size`数据创建**2D**纹理对象。

接下来的两行可能有些令人迷惑。简单点来说，`oX`和`oY`是采样的起始点，通过`UnityEngine.Random.value`产生随机的效果。因为一张uv图的坐标范围是在(0, 0)到(1, 1)之间的，所以直接用`UnityEngine.Random.value`返回的就是一个[0, 1]之间的随机浮点数。

继续看下去，进入了一个嵌套的`for`循环。对，你猜得没错，现在开始采样噪点并写入我们的纹理的时候了。由于Perlin噪点图是灰度图，没有彩色数据。而我们知道，对于一个用**rgb**法表示的颜色，如果`[r, g, b]`每个值都相等的话，得到的就是一个白色到黑色间的颜色。由此我们把使用`Mathf.PerlinNoise(float x, float y)`的采样结果，一个`float`对象做为当前颜色所有的部分（rgb）的值大小。在Unity中，`Color`结构体的rgb也是[0, 1]间的浮点数，而不是常见的[0, 256]的整形。值得一提的是，`Mathf.PerlinNoise`返回的`float`在大部分的情况下会在[0, 1]区间，但是官方文档特地指出是有可能大于1的。当然了，我们这里就不用对结果clamp了，因为`Color`的构造函数会自动把大于1的参数强制调到1。

那么使用`Mathf.PerlinNoise(float x, float y)`采样的坐标参数怎么确定呢？对于每一个方向（即xy），我们从起点开始，加上当前遍历到的像素的相对坐标（`当前位置／该方向长度`，即`i / size`，因为要得到一个浮点结果，所以强转一下`((float) i / (float) size)`）。当然了我还乘了一个传进来的参数`scale`。这是啥呢？简单来说，`scale`控制了噪点的密集程度，越大越密集，相似的噪点图案重复的越多，因为采样的偏移更快了。

得到值后，`texture2D.SetPixel(i, j, new Color(greyScale, greyScale, greyScale))`把结果写入我们的纹理中。接下来的代码是显示操作进度的，每绘制100个像素更新一下进度条。之所以不每像素更新是因为效率考量，想想看**256x256**一共有多少个像素，每次更新**GUI**的开销不小。同时记得如果点了操作取消就退出这个方法。

最后最后，采样结束后，`text2D.Apply()`保存写入的数据。然后使用`File.WriteAllBytes(string path, byte[] data)`将纹理写入纹理。我们使用`.png`的编码方式，所以还要在用`texture2D.EncodeToPNG()`来获得对应格式的二进制数据。

这样，操作主体结束了！`EditorUtility.ClearProgressBar()`移除进度条。最后，使用`AssetDatabase.ImportAsset(string path)`将这个纹理导入到项目里，这样就可以在编辑器里看到它了（不然得手动刷新项目才行）。这里得注意一下，传入的路径不是基于操作系统的，而是从项目的资源目录开始的（Assets/...），所以我把之前的保存路径截取一下传进去。

# 封装
为了方便使用，我们需要让这个功能可以在编辑器里面调用。在`Editor`文件下新建一个脚本`MaterialTool.cs`，内容如下：

```csharp
using System.IO;
using UnityEditor;
using UnityEngine;
using Random = UnityEngine.Random;

public static class MaterialTool {

	[MenuItem("材质工具/生成噪点图/Perlin 1X")]
	public static void GeneratePerlinNoiseMap1X() {
		GeneratePerlinNoiseMap(1);
	}
	
	[MenuItem("材质工具/生成噪点图/Perlin 2X")]
	public static void GeneratePerlinNoiseMap2X() {
		GeneratePerlinNoiseMap(2);
	}
	
	[MenuItem("材质工具/生成噪点图/Perlin 3X")]
	public static void GeneratePerlinNoiseMap3X() {
		GeneratePerlinNoiseMap(3);
	}
	
	[MenuItem("材质工具/生成噪点图/Perlin 4X")]
	public static void GeneratePerlinNoiseMap4X() {
		GeneratePerlinNoiseMap(4);
	}
	
	[MenuItem("材质工具/生成噪点图/Perlin 5X")]
	public static void GeneratePerlinNoiseMap5X() {
		GeneratePerlinNoiseMap(5);
	}
	
	[MenuItem("材质工具/生成噪点图/Perlin 6X")]
	public static void GeneratePerlinNoiseMap6X() {
		GeneratePerlinNoiseMap(6);
	}
	
	[MenuItem("材质工具/生成噪点图/Perlin 7X")]
	public static void GeneratePerlinNoiseMap7X() {
		GeneratePerlinNoiseMap(7);
	}
	
	[MenuItem("材质工具/生成噪点图/Perlin 8X")]
	public static void GeneratePerlinNoiseMap8X() {
		GeneratePerlinNoiseMap(8);
	}
	
	[MenuItem("材质工具/生成噪点图/Perlin 9X")]
	public static void GeneratePerlinNoiseMap9X() {
		GeneratePerlinNoiseMap(9);
	}
	
	public static void GeneratePerlinNoiseMap(float scale) {
		string path = EditorUtility.SaveFilePanelInProject("生成柏林噪点图", "perlin_noise_texture_" + scale + "x", "png", "保存", Application.dataPath + "/Resources/Textures");
		if (!string.IsNullOrEmpty(path)) {
			if (EditorUtility.DisplayCancelableProgressBar("生成柏林噪点图", "初始化", 0f)) {
				ClearProgressBar();
				return;
			}
			int size = 256;
			int sizeSqr = size * size;
            Texture2D texture2D = new Texture2D(size, size);
			float oX = Random.value;
			float oY = Random.value;
			for (int i = 0; i < size; i++) {
				for (int j = 0; j < size; j++) {
					float greyScale = Mathf.PerlinNoise(oX + ((float) i) / ((float) size) * scale, ((float) j) / ((float) size) * scale);
					texture2D.SetPixel(i, j, new Color(greyScale, greyScale, greyScale));
					if (j % 100 == 0) {
						if (EditorUtility.DisplayCancelableProgressBar("生成柏林噪点图", greyScale.ToString(), (float) (size * i + j + 1) / sizeSqr)) {
							ClearProgressBar();
							return;
						}
					}
				}
			}
			texture2D.Apply();
			File.WriteAllBytes(path, texture2D.EncodeToPNG());
			EditorUtility.ClearProgressBar();
			AssetDatabase.ImportAsset(path.Substring(path.IndexOf("Assets")));
		}
	}
}
```

OK，大功告成！

# 改进
你难道不觉得给每种密集程度的噪点图写个编辑器菜单拓展很傻吗？可以直接写一个自定义的`EditorWindow`提供更便捷的操作。