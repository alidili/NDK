# <center><big>Android NDK开发（五） 开发中遇到的问题汇总</big></center>

> 作者：杨乐
>
> 日期：2021/12/28
>
> 版本：1.0

[TOC]


# 0.abiFilters是做什么用的？

我们在项目的gradle中经常会看到这样的配置：

```
defaultConfig {
	...
	
	ndk {
		abiFilters "armeabi-v7a", "x86"
	}
}
```

那为什么要这样配置呢，一起来看下：

如果我们在项目中引入了某个SDK，这个SDK中支持 **armeabi**、**armeabi-v7a**、**arm64-v8a**、**x86**、**x86_64** 五种ABI，但是我们的项目中只支持 **armeabi-v7a** 和 **x86** 两种ABI，这样打包APK后就会生成五种ABI目录，其中SDK中的so库是正常的，但是我们项目中的so库就会只存在于 **armeabi-v7a** 和 **x86** 两个目录中，应用安装在 **armeabi-v7a** 或 **x86** 架构的设备上是没有问题的，但是安装在其他架构的设备上（比如 **arm64-v8a** ）就会出现下面的异常，找不到对应的so库：

```
java.lang.UnsatisfiedLinkError: Couldn't load native-lib from loader dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/com.yl.ndkdemo-1.apk"]
nativeLibraryDirectories=[/data/app-lib/com.yl.ndkdemo-1, /vendor/lib, /system/lib, /system/lib/arm]]]: findLibrary returned null
```

那该怎么解决呢？有人会说可以把SDK中多出来的三种ABI删掉，嗯，也是个方法，但是如果是远程库引用就没办法了，这时就轮到 **abiFilters** 出马了，限制安装包ABI的架构，不管SDK或者项目中有多少种ABI架构，最终打包进APK的so库都以abiFilters中规定的为准。

**也就是说，如果我们的项目中有五种ABI库，但是abiFilters中值设置了两种ABI支持，那么我们最终打包的APK中只会包含这两种ABI库。**

# 1.ABIs [armeabi] are not supported for platform. 

最新的NDK r17版本，已经去掉了armeabi、mips、mips64的ABI支持。

如果在编译的过程中遇到下面的报错，不要方，去掉gradle中配置的armeabi选项就可以了：

```
ABIs [armeabi] are not supported for platform. Supported ABIs are [armeabi-v7a, arm64-v8a, x86, x86_64].
```

去掉abiFilters中的armeabi选项，armeabi-v7a会兼容armeabi：

```
defaultConfig {
	...
	
	ndk {
		abiFilters "armeabi-v7a", "arm64-v8a", "x86", "x86_64"
	}
}
```

# 2.如何debug C/C++ 源码？

首先安装LLDB调试工具（File > Settings > Appearance & Behavior > System Settings > Android SDK > SDK Tools）：

![LLDB](https://upload-images.jianshu.io/upload_images/3270074-804a3761c439c8a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

LLDB安装完成后就可以debug源码了，和debug java代码的流程基本一致，看下效果：

![debug源码](https://upload-images.jianshu.io/upload_images/3270074-25d02b38f48b01c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在debug的过程中可能会遇到一些诡异的问题，比如下面这行报错：

```
Attention! No symbol directories found - please check your native debug configuration.
```

大概意思是symbol路径找不到，请检查一遍你的debug配置，如果你的项目中JNI代码在Library中，这个错误发生的几率会比较大。

在网上查到这样一种方法，打开Run > Edit Configurations > Debugger > Symbol Directories选项，在里面配置一下Library的路径，如下图所示：

![配置Symbol Directories](https://upload-images.jianshu.io/upload_images/3270074-db8530f547f8fe5d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

debug运行一下，正常了，但是第二次再debug的时候继续报上面的错，继续查啊查，看到这样一条解决方案，把local.properties文件中ndk的路径改成错的，编译一下（肯定报错），再改回原来的就正常了，什么？还有这样的操作，抱着怀疑的态度试了下，居然可以了...可以了...

![local.properties](https://upload-images.jianshu.io/upload_images/3270074-766ff6b8d51e8428.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个问题随机出现，可能是AS抽风了，遇到了可以这样试试。

# 3.如何使用编译好的so库？

有读者在评论中问我这个问题，在这里总结下，写一个简单的Demo，返回一个字符串给Java层，方法名是通过 **Java\_包名\_类名\_方法名** 的方式命名的，其中包名和类名就是Java层调用类的包名和类名，这是一种规定的命名方式，不可以修改，看下代码：

```
#include <jni.h>
#include <string>

extern "C"
JNIEXPORT jstring

JNICALL
Java_com_yl_ndklibrary_NDKLibrary_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}
```

Java层：

```
package com.yl.ndklibrary;

public class NDKLibrary {

    static {
        System.loadLibrary("native-lib");
    }

    public static native String stringFromJNI();
}
```

编译好so库后，如果想要提供给其他项目使用，则项目中调用so库的类必须和so库编译时的类（例如：NDKLibrary）类名和包名完全一致，否则就会报下面的错误：

```
java.lang.UnsatisfiedLinkError: No implementation found for java.lang.String com.yl.ndkdemo.NDKLibrary.stringFromJNI()
```

通常我们会把so库和调用类打包成一个Library提供给第三方调用者，调用者拿到Library后，直接调用其中的方法就可以了，不必关心类名与包名，这也是开源项目中的常见做法。

# 4.写在最后

文章中如有遗漏、错误的地方，可以给我留言指正，多谢！

本文中用到的源码已上传至GitHub，欢迎Fork，觉得还不错就Start一下吧！

GitHub传送门：https://github.com/alidili/Demos/tree/master/NDKLibraryDemo