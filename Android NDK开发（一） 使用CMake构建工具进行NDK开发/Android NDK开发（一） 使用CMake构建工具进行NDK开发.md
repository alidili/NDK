# <center><big>Android NDK开发（一） 使用CMake构建工具进行NDK开发</big></center>

> 作者：杨乐
>
> 日期：2021/12/27
>
> 版本：1.0

[TOC]


# 0.了解一些概念

- JNI（Java Native Interface）：

  Java原生接口，是Java和其他原生代码语言（例如 C 和 C++）通信的桥梁。

- NDK（Native Development Kit）：

  原生开发工具集，是一套允许您使用原生代码语言（例如 C 和 C++）实现程序功能的工具集。

- ABI（Application Binary Interface）：

  应用程序二进制接口，不同的CPU支持不同的指令集，CPU与指令集的每种组合都有其自己的应用二进制接口（或ABI），ABI可以非常精确地定义应用的机器代码在运行时如何与系统交互。
  
  [ABI官方文档](https://developer.android.com/ndk/guides/abis.html)
  
  **支持的ABI：armeabi、armeabi-v7a、arm64-v8a、x86、x86_64、mips、mips64**

- CMake：

  Android推荐使用的NDK构建工具，从AS 2.2版本之后开始支持（包含2.2版本）。
  
# 1.环境搭建

**安装NDK开发所需的工具**

![安装NDK开发所需的工具](http://upload-images.jianshu.io/upload_images/3270074-e971cc0e2df7c868.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在SDK Tools中安装以下组件：

- Cmake：NDK构建工具

- LLDB：NDK调试工具

- NDK：NDK开发工具集

**创建NDK项目**

![创建NDK项目](http://upload-images.jianshu.io/upload_images/3270074-cab9fcc82fc0f816.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在创建项目时，勾选【Include C++ support】选项，然后一路下一步，到达【Customize C++ Support】设置页：

![Customize C++ Support](http://upload-images.jianshu.io/upload_images/3270074-ca7631b097798256.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到三个选项：

- C++ Standard：C++标准，选择【Toolchain Default】会使用默认的CMake配置。

- Exceptions Support：支持C++异常处理，标志为 **-fexceptions**。

- Runtime Type Information Support：支持运行时类型识别，标志为
 **-frtti**，程序能够使用基类的指针或引用来检查这些指针或引用所指的对象的实际派生类型。

在这里我们使用默认C++标准，不勾选下面的两个选项，点击【Finish】按钮进入下一个环节。

# 2.NDK项目

看下项目目录：

![项目目录](http://upload-images.jianshu.io/upload_images/3270074-f016faf807930464.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图中用红框标识了NDK项目与普通项目的不同之处，下面分别来看看：

**首先来看下build.gradle配置：**

```
apply plugin: 'com.android.application'

android {
    compileSdkVersion 26
    defaultConfig {
        applicationId "com.yl.ndkdemo"
        minSdkVersion 15
        targetSdkVersion 26
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
        externalNativeBuild {
            cmake {
                cppFlags ""
            }
        }
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
    externalNativeBuild {
        cmake {
            path "CMakeLists.txt"
        }
    }
}

dependencies {
	// 引用一些库
}
```

可以看到build.gradle配置中多了两个externalNativeBuild配置项：

- defaultConfig里面的：

  主要配置了Cmake的命令参数，例如在创建项目时，如果勾选了【Exceptions Support】和【Runtime Type Information Support】选项，是这样配置的：
  
```
externalNativeBuild {
	cmake {
		cppFlags "-fexceptions -frtti"
	}
}
```

更多命令参数可以查看[Android NDK CMake文档](https://developer.android.com/ndk/guides/cmake.html)

- defaultConfig外面的：

  主要定义了CMake的构建脚本CMakeLists.txt的路径。
  
**CMake的构建脚本CMakeLists.txt**

CMakeLists.txt是CMake的构建脚本，作用相当于ndk-build中的Android.mk，看下CMakeLists.txt：

```
# 设置Cmake最小版本
cmake_minimum_required(VERSION 3.4.1)

# 编译library
add_library( # 设置library名称
             native-lib

             # 设置library模式
             # SHARED模式会编译so文件，STATIC模式不会编译
             SHARED

             # 设置原生代码路径
             src/main/cpp/native-lib.cpp )

# 定位library
find_library( # library名称
              log-lib

              # 将library路径存储为一个变量，可以在其他地方用这个变量引用NDK库
              # 在这里设置变量名称
              log )

# 关联library
target_link_libraries( # 关联的library
                       native-lib

                       # 关联native-lib和log-lib
                       ${log-lib} )
```

这是一个基本的CMake构建脚本，更多脚本配置请参考[CMAKE手册](https://cmake.org/cmake-tutorial/)，看不懂！没关系，这里有中文版的[CMAKE手册-中文版](https://www.zybuluo.com/khan-lau/note/254724)。

**原生代码native-lib.cpp**

Android提供了一个简单的JNI交互Demo，返回一个字符串给Java层，方法名是通过 **Java\_包名\_类名\_方法名** 的方式命名的：

```
#include <jni.h>
#include <string>

extern "C"
JNIEXPORT jstring

JNICALL
Java_com_yl_ndkdemo_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}
```

看下如何调用：

```
public class MainActivity extends AppCompatActivity {

    // 加载native-lib，不加lib前缀
    static {
        System.loadLibrary("native-lib");
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // 将获取的字符串显示在TextView上
        TextView tv = (TextView) findViewById(R.id.sample_text);
        tv.setText(stringFromJNI());
    }

    /**
     * native-lib中的原生方法
     */
    public native String stringFromJNI();
}
```

调用方式很简单，代码中已经写了注释，看下效果：

![运行效果](http://upload-images.jianshu.io/upload_images/3270074-0a2ae248f24c36d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**生成so文件**

在CMakeLists.txt中将library的编译模式设置为SHARED模式，点击AS的编译按钮，在app > build > intermediates > cmake > debug > obj目录下会生成不同CPU架构对应的so文件：

![so文件目录](http://upload-images.jianshu.io/upload_images/3270074-676c5d7d1529b5c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

生成的so文件也可以在其他项目中使用，在项目的app > src > main目录下创建jniLibs文件夹，将生成的so文件（带着CPU架构目录）拷贝到jniLibs文件夹中，按照上文中的调用方式即可正常使用。

在app的build.gradle文中配置abiFilters，可以输出指定ABI的so文件：

```
defaultConfig {
	...
	
	ndk {
		abiFilters "armeabi", "armeabi-v7a", "arm64-v8a", "x86", "x86_64", "mips", "mips64"
	}
}
```

# 3.写在最后

[NDK官方使用文档](https://developer.android.com/studio/projects/add-native-code.html)

后续还会更新更多NDK开发系列文章，敬请期待！