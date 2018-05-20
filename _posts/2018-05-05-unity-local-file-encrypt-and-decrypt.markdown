---
layout: post
title: Unity游戏开发中对本地文件的加密读写
comments: true
date: 2018-05-05 07:50:48.000000000 +09:00
author: William Xie
tags: Unity
---

# 概述
在实际项目中，总是需要将一些数据持久化在本地，比如游戏存档，偏好设置或是热更新的一些资源。对于其中一些需求如偏好设置来说，直接以明文的方式写入`PlayerPrefs`就可以了，但是对于存档这类的文件一般都是在写入在项目的`Application.persistentDataPath`路径下。然而直接明文读写的话可能会导致存档被玩家篡改，或者重要的信息被第三方获取到（比如与服务器通讯的秘钥），因此，对于敏感的文件，我们需要对它进行加密操作。

# 实现

## 二进制存储
第一种实现的方法非常简单，就是存储文件的时候不要以文本的形式写入本地，而是写一个二进制文件，并且更改文件后缀为一个自定义的后缀，如`.interesting`之类的。对于大部分玩家来说，一个不常见的二进制文件往往不能直接使用记事本读写软件打开，也就放弃了对存档之类的数据篡改。关键代码如下：

```csharp
byte[] binaryContents = Encoding.UTF8.GetBytes("some contents");
string contents = Encoding.UTF8.GetString(binaryContents);
```

这段代码非常简单，`Encoding`里面提供不同的编码方式的工具类，这里可以根据需求选择字符串编码方式，一般使用**UTF-8**的编码，`GetBytes(string str)`返回一个`byte[]`，即该字符串的二进制编码，而`GetString(byte[] bytes)`返回一个`string`，即将二进制数据按照指定编码转为字符串。

当然，项目中肯定会有很多处需要进行本地读写的地方，我们不可能每一处都添加这么几行代码，因此，我们需要设计一个`CipherManager`类来对文档加密读写操作进行统一封装与管理。

```csharp
public static class CipherManager {

    public static byte[] ToBinary(string toBinary) {
        return Encoding.UTF8.GetBytes(toBinary);
    }

    public static string FromBinary(byte[] fromBinary) {
        return Encoding.UTF8.GetString(fromBinary);
    }

    public static void WriteToFile(string path, string toWrite) {
        File.WriteAllBytes(path, ToBinary(toWrite));
    }

    public static string ReadFromFile(string path) {
        return FromBinary(File.ReadAllBytes(path));
    }
}
```

这里我们简单使用一个`static class`创建一个单例管理类，当需要读写加密数据的时候直接调用`WriteToFile(string path, string toWrite)`与`ReadFromFile(string path)`方法即可，将加密过程封装起来，以便复用。

事实上，不少游戏像侠盗猎车手3（Grand Theft Auto III）就使用了这一方法对部分数据进行了加密。GTA3游戏目录下的`.dat`文件，直接使用记事本是打不开的，当转换后缀名为`.txt`，并指定解码方式为**UTF8**就可以成功看到里面的文本内容。游戏里的各种道具的数值都是用这种方式存储的。

### 缺点
这种存储方法非常简单，但也有不小的问题。首先数据本身其实并没有进行加密，我们只是利用了操作系统默认通过后缀名判断文件格式的漏洞。事实上，对于目前的各种文本编辑器，它们对文件格式的判断是会读取二进制数据以判断是不是文本编码，因此如果转换为**UTF-8**，**Unicode**之类的二进制格式，很有可能一下就被读出文本内容。如果有人对游戏的本地数据图谋不轨的话这种尝试是最基本的，因此，我们需要加入真正的加密。

## AES（Rijndael）加密
不要被突然高大上起来的标题吓到了，这边博客只会简单谈谈什么是**AES**加密技术，只有实现嘛。。。只要知道`C#`提供了相关**api**就行了。

### 什么是AES？
**AES**，即**Advanced Encrytion Standard**，又称**Rijndael**加密法，是美国联邦政府选用的一种区块加密标准，由比利时密码学家**Joan Daemen**和**Vincent Rijmen**所设计（看名字应该能看出来😂）。总而言之，这是一种比较新的，比较安全的加密方式，虽然可以被破解，但是大部分破解都不是针对密码本身的，而是基于不安全的系统进行攻击，因此不足为虑。

**AES**算法需要我们提供一个`byte[]`格式的秘钥，字节长度必须为32的整数倍，且以128位为下限，265位为上限，但事实上我们只需要提供一个32位的秘钥就行了，因为`C#`自带的加密**api**会将不够位数的秘钥进行重复叠加来填充位数。

重新设计过的`CipherManager`类实现如下：

```csharp
public static class CipherManager {

	readonly static byte[] KEY = Encoding.UTF8.GetBytes("guardheiguardheiguardheiguardhei");

	static RijndaelManaged rij;
	static ICryptoTransform encryptor;
	static ICryptoTransform decryptor;

	static CipherManager() {
		rij = new RijndaelManaged();
		rij.Key = KEY;
		rij.Mode = CipherMode.ECB;
		rij.Padding = PaddingMode.PKCS7;
		encryptor = rij.CreateEncryptor();
		decryptor = rij.CreateDecryptor();
	}

	public static string Encrypt(string toEncrypt) {
		byte[] toEncryptArray = Encoding.UTF8.GetBytes(toEncrypt);
		byte[] encryptedArray = encryptor.TransformFinalBlock(toEncryptArray, 0, toEncryptArray.Length);
		return Convert.ToBase64String(encryptedArray);
	}

	public static string Decrypt(string toDecrypt) {
		byte[] toDecryptArray = Convert.FromBase64String(toDecrypt);
		byte[] decryptedArray = decryptor.TransformFinalBlock(toDecryptArray, 0, toDecryptArray.Length);
		return Encoding.UTF8.GetString(decryptedArray);
	}

	public static void EncryptFile(string path) {
		string text = File.ReadAllText(path);
		File.WriteAllText(path, Encrypt(text));
	}

	public static string DecryptFile(string path) {
		return Decrypt(File.ReadAllText(path));
	}
}
```

是不是看上去复杂了很多？不要急，慢慢来看。

一开始我们首先需要创建一个秘钥`KEY`，这里我们通过获取传入的字符串的二进制数组实现的。我们需要保证字符串的字节长度是32的位数（注意不是字符串的长度，而时候字节长度！），对于**UTF-8**编码来说，一个英文字符占一个字节，一个中文字节占两个字节，对于**Unicode**编码来说，任意字符都占两个字节。

创建好秘钥后，我们声明一个静态的加密管理对象`rij`，`RijndaelManaged`类的实例，以及两个密码变换实例`encryptor`和`decryptor`。加密管理对象比较好理解，它管理着加密的各种参数，其中`Key`就是一个`byte[]`型秘钥，`mode`是指加密模式，`padding`是补位方式。

秘钥我们已经在前面解释过了，那么加密模式和补位方式又是啥？

加密模式可以理解为统一加密标准的不同实现。这是因为现实应用中，不同的项目需求的加密数据量是不同，数据格式也是不定的，加密硬件也会有差异，为了更好的兼顾安全性与性能，针对不同的加密需求，选择不同的加密模式。常见的加密模式如下：

1. **ECB**（**Electronic Code Book**）电子密码本模式
比较简单的模式，可以进行并行运算且不会传递误差，但更加容易被主动攻击，所以适用于比较简短的消息
2. **CBC**（**Cipher Block Chaining**）加密快链模式
不要被名字吓到，这玩意儿和区块链（**Block Chaining**）没有什么关系。这种模式比**ECB**更安全，适合加密长度较长的消息，但是会传递误差
3. **CFB**（**Cipher Feedback Mode**）加密反馈模式
比**ECB**安全，更适合加密流数据，虽然加密不能并行运算但解密可以，同时会传递误差
4. **OFB**（**Output Feedback Mode**）输出反馈模式
比**ECB**要安全但没有**CBC**或**CFB**那么安全，也适合加密流数据，但是不能进行并行运算，且传递误差
5. **CTR**（**Counter**）计数模式
这种模式不需要实现解密算法，只要实现加密算法即可，因此最为简单。同时因为允许并行计算与比重足够大的预处理，可以以“空间换时间”提高加密效率，还不会传递误差，有着不低的安全性。但是如果密文传输过程中丢失字节会导致后续字节无法正确解密（这主要发生在网络通讯中）

补位方式说起来就有点复杂，具体说来加密模式分为块加密（**ECB**和**CBC**）和流加密（**CFB**和**OFB**）。块加密，又称分组加密，就是将待加密的数据分组，即分割成一个个固定大小的数据块，对每一个块进行加密。然而不可能待加密的数据大小正好可以被分割到每一个块里且充分的填充每一个块，因此我们需要对最后一个块里没有填充完的部分填入“补位数据”，而填充方式就是“补位方式”。流加密是不需要这一点的。

一般来说**ECB**的加密模式就可以满足我们的需求了，补位方式选择**PKCS7**即可（不要在意细节啦），如果有特殊需求的情自行尝试。

接下来说说`ICryptoTransform`的两个实例：`encryptor`和`decryptor`。在加密与解密的过程中，**AES**算法对传入的数据进行了种种变换而进行加密／解密。也就是说算法的具体实现是由它们决定的，抓重点就是这个实例可以给我们提供如何变换数据的方法，因此它们被当作加密器和解密器。可以看到我们并不是直接`new`了两个对象，而是初始化完`rij`对象的相关属性后使用`CreateEncrytor()`和`CreateDecrytor()`方法获取的。

到了这一步，我们可以谈一谈加密的步骤了，可以看到在`Encrypt(string toEncrypt)`方法中，我们先获取待加密文本的**UTF-8**格式的字节数据，然后通过`encryptor.TransformFinalBlock(byte[] buffer, int offset, int length)`加密得到加密后的字节数据。`TransformFinalBlock`这个名字听起来怪怪的，其实就是是获得变换到最后的块数据，讲白了就是加密完成的数据。三个参数分别是待加密的字节数据，偏移（从第几位开始后加密，从头开始当然是0啦）和需要加密的字节的长度（加密全部的所以直接`buffer.length`就可以啦）。有些人可能会奇怪最后一行代码为什么会使用一个`Convert.ToBase64String(byte[] inArray)`返回一个`string`。这里我们来稍微说下**Base64**这个东西。

**Base64**是一种常见的用于传递8位字节码的编码标准，是一种用64个可打印的字符来表示二进制数据的方法，可以在**HTTP**下传递较长的标识信息，其他的应用如迅雷的专用下载链接其实就是在正常下载地址的前后分别添加字符串**AA**与**ZZ**，再对新字符串进行**Base64**编码。直白点说，我们可以用**Base64**对字符串和二进制数据再进行一层简单的加密。想想看如果直接把存档存成二进制文件，破解者可能直接会考虑从此入手寻找**AES**的破解方法，而如果先用文本包一层奇怪的外表，破解者如果不清楚我们使用了**Base64**的话可能都无从下手。

解密的步骤基本就是加密的逆向，最后返回一个能够被我们的游戏解析的数据信息，没有必要多说。

最后再封装两个对文件进行加密解密的方法，这样，一个简单的加密管理类就完成了（所以说确实不用把**AES**加密方式理解的十分透彻，会用就行啦！）。

### 缺点
**AES**是比较主流的商业加密方法，像之前说的，本身并没有特别大的缺陷，但是有一个问题值得开发者思考，那就是：

**Unity**打包的项目不加密脚本代码啊！！！

所以说如果我们把加密的秘钥用字符串直接硬编码在代码里的话，有概率被用**Unity**解包器给揪出来，不过如果我们选择**il2cpp**这种发布方式的话，理论上会好一下，不容易被解。实在不行就撺掇公司去买个商业级的代码加密方案吧😂。