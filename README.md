# iOS 逆向工程-暗柱趣谈

某些Apple明令禁止的App类型，很多是上不了架，那么想要在iOS设备上安装App还有两种渠道：**开发者模式、企业版发布**。

## 开发者模式

只需要申请一个免费的开发者账户，用Xcode运行App即可，但是这个有一个问题，**你得有源代码**。99%的情况下，我们是没有源码的。

## 企业版发布

某些App是不适合在App Store上发布的，比如企业内部员工作业App。那么Apple也提供了专门的一种企业版发布方式，iOS设备下载指定plist文件后会自动安装。

```
* App Store上下载的App是永远不会过期的，但是企业版的App是有一年的使用期限，期限到得重新打包让用户下载（至少一年要更新一次）。
```

## 闪退、无法验证App

一旦企业版证书到期、被吊销，那么你的企业版App就再也无法打开了。问题并不是代码有问题，而是苹果的信任机制。

我只想在我的iPhone上继续用这个App，如何击破这个限制？

企业版App是可以拿到ipa包的，我们可以对ipa包进行重签名（无论是开发证书还是发布证书），然后再安装到我们iPhone上。

## 重签名

App必须经过信任才能在iOS设备上运行，开发者在真机调试的时候有开发证书签名，App Store上的App是苹果公司的证书签名，企业版App是经过苹果签名的公司证书。

```
所谓签名就是一种数字签名技术

1. 把ipa包里面的资源使用散列值算法计算出一个唯一值，md5
2. 使用被认证的私钥对这个散列值进行RSA加密, rsa(md5)
3. 再把这个加密值和公钥放到ipa包里面
4. App运行时，重新计算ipa包的散列值md5_now，然后用公钥解密出md5
5. 如果 md5_now == md5，那么正常运行App，否则闪退或提示

```

因此我们使用有效的证书和相关文件对ipa包重签名即可得到一个正常安装的ipa包（App Store下载的ipa包是加密，这种情况要在越狱环境下先解密得到解密ipa包才能重签名，否则无法做这个操作）。

- 准备工具
	* 企业版ipa包
	* iResign，重签名工具
	* adhoc或inhourse或开发证书、描述文件mobileprovision

选择ipa包路径、描述文件，填写描述文件对应的App ID，如下图所示：

![重签名](https://raw.githubusercontent.com/0xfeedface1993/iOSReverseEngineering/master/images/resined.png)

### 说好的证书和公钥呢？

描述文件里面就有，通过二级制文件查看器，可以看到它包含了一个xml文件，包含权限信息、证书公钥信息和其他信息。

![mobileprovision](https://raw.githubusercontent.com/0xfeedface1993/iOSReverseEngineering/master/images/mobilefile.png)


点击**重新签名**，然后就在原ipa文件夹下生成一个`-resigned`结尾的ipa包文件，这个就是重签名后的包了。

![resignedfile](https://raw.githubusercontent.com/0xfeedface1993/iOSReverseEngineering/master/images/resignedfile.png)

## 安装IPA包

一般来说，使用iTunes安装即可（使用第三方软件可能会再重签名，或者其他操作，这时候很容易出问题）。

iTunes安装ipa非常简单，打开iTunes，插上手机，等待识别出iPhone，然后拖动ipa包到左侧的设备选项，顶部就会开始有提示。

![itns](https://raw.githubusercontent.com/0xfeedface1993/iOSReverseEngineering/master/images/itns.png)

## Oops! 代码有毒！

好的, 很明显，有人在代码里面下了毒，为了防止在越狱设备上运行它的App，这就是**暗柱**，因为暗的看不见就撞上去了，因为这非常危险！

![ipaerror](https://raw.githubusercontent.com/0xfeedface1993/iOSReverseEngineering/master/images/ipaerror.jpg)

那么我们就看它代码是如何实现这个暗柱的，接下来使用逆向工程分析代码。

## 寻找暗柱

- 准备工具：
	*  Hopper静态反编译工具，免费版即可
	*  010 Editor, 二进制文件修改器

首先找到app包内部的可执行文件`xxx_release`，：

![execfile](https://raw.githubusercontent.com/0xfeedface1993/iOSReverseEngineering/master/images/execfile.png)

### 查找关键代码

等到Hopper解析二进制可执行文件完成后，通过大量交叉引用分析（各种慢慢找），然后我们发现实现重签名的检测方法是：

- App启动
	* 初始化
	* 检测app包内是否有`.mobileprovision`文件，若有则弹出对话框强退app

对应的代码如下：

![asmcode](https://raw.githubusercontent.com/0xfeedface1993/iOSReverseEngineering/master/images/asmcode.png)

代码最后一句是最关键的：

```
 tbz w22, 0x0, loc_10008a314
```

寄存器`W22`存放的是`.mobileprovision`文件的”实例“，如果不存在则为0；`tbz`指令检测寄存器的比特位测试位为0发生跳转，imm指定目的寄存器的某一个位，『b5：b40』组成，0-63或者0-31，有b5决定。哪个目的寄存器由Rt指定，label是偏移地址。

![tba](https://raw.githubusercontent.com/0xfeedface1993/iOSReverseEngineering/master/images/tbz.png)

我们重签名以后这个文件肯定是存在的，因此我们要将此处代码改为 **为1发生跳转**，符合这个条件的是另一个很类似的指令`tbnz`:

![tbnz](https://raw.githubusercontent.com/0xfeedface1993/iOSReverseEngineering/master/images/tbnz.png)

### 机器码转换

`tbz`机器码格式，31位由第5位决定:

![tba](https://raw.githubusercontent.com/0xfeedface1993/iOSReverseEngineering/master/images/tbzm.png)

`tbnz`机器码格式，31位由第5位决定:

![tba](https://raw.githubusercontent.com/0xfeedface1993/iOSReverseEngineering/master/images/tbnzm.png)

可见两条指令不同之处在`24`位，因此只需要将`0`改为`1`即可。

程序中实际的`TBZ W22, #0, #0x6B8`语句机器码是:

`D6 35 00 36`

因为在iPhone上指令是按照大端模式存储的，因此要做一个调换：

`36 00 35 D6`

`0011 0110 0000 0000 0011 0101 1101 0110`

转换成 `TBNZ` 指令, 修改好后同样也要做个调换：

`0011 0111 0000 000 0011 0101 1101 0110`

`37 00 35 D6`

`D6 35 00 37`

`TBNZ W22, #0, #0x6B8`

### 修改机器码

好了，现在可以开始真正修改代码了，用**010 Editor**打开可执行文件`xxx_release`，搜索 `D6 35 00 36` 并将其修改为 `D6 35 00 37`。

![010](https://raw.githubusercontent.com/0xfeedface1993/iOSReverseEngineering/master/images/010.png)

源代码修改部分完成！


### 在来一次

重复重签名的步骤，重新安装ipa，成功进入App了！！！！

![010](https://raw.githubusercontent.com/0xfeedface1993/iOSReverseEngineering/master/images/succc.jpg)

## 总结

在这种企业版的app下，有很多防止反编译和调试的方法，但是都是增加了逆向工程的难度（即使是App Store里面的应用，拿到砸壳后的ipa包也是一样），
大量使用c/c++代码、混淆代码才是折磨人的地方，但是这只是为了劝退大部分人，正所谓科技是一把双刃剑。
