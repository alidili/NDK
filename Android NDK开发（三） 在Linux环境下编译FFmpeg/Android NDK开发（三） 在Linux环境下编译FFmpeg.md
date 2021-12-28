> # <center><big>Android NDK开发（三） 在Linux环境下编译FFmpeg</big></center>

> 作者：杨乐
>
> 日期：2021/12/28
>
> 版本：1.0

[TOC]

FFmpeg是一套可以用来记录、转换数字音频、视频，并能将其转化为流的开源计算机程序。它提供了录制、转换以及流化音视频的完整解决方案。

# 0.环境搭建

> 操作系统：Ubuntu 16.04 64bit
>
> NDK版本：android-ndk-r14b-linux-x86_64
>
> FFmpeg版本：3.4.2

NDK 下载地址：https://developer.android.com/ndk/downloads/index.html

FFmpeg 下载地址：http://ffmpeg.org

将NDK和FFmpeg源码解压至指定目录，文中解压在了Home > ffmpeg目录下，注意NDK的版本一定要选择Linux版的，在编译的时候我就犯了这个错误，使用了Windows版的NDK，一直报错也找不到原因，切记切记！

# 1.编译FFmpeg

在ffmpeg-3.4.2目录下创建build_android.sh编译脚本，文件名可以随便取，这个脚本是本文中最关键的内容，一起来看下：

```
#!/bin/bash

# 设置临时文件夹，需要提前手动创建
export TMPDIR="/home/yangle/ffmpeg/ffmpeg-3.4.2/ffmpegtemp"

# 设置NDK路径
NDK=~/ffmpeg/android-ndk-r14b

# 设置编译针对的平台，可以根据实际需求进行设置
# 当前设置为最低支持android-14版本，arm架构
SYSROOT=$NDK/platforms/android-14/arch-arm

# 设置编译工具链，4.9为版本号
TOOLCHAIN=$NDK/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64

function build_one
{
./configure \
    --enable-cross-compile \
    --enable-shared \
    --disable-static \
    --disable-doc \
    --disable-ffmpeg \
    --disable-ffplay \
    --disable-ffprobe \
    --disable-ffserver \
    --disable-avdevice \
    --disable-doc \
    --disable-symver \
    --prefix=$PREFIX \
    --cross-prefix=$TOOLCHAIN/bin/arm-linux-androideabi- \
    --target-os=android \
    --arch=arm \
    --sysroot=$SYSROOT \
    --extra-cflags="-Os -fpic $ADDI_CFLAGS" \

$ADDITIONAL_CONFIGURE_FLAG
make clean
make
make install
}

# 设置编译后文件的输出目录
CPU=arm
PREFIX=$(pwd)/android/$CPU
ADDI_CFLAGS="-marm"
build_one
```

前四项配置按照你本机的实际路径修改就可以了，注意ffmpegtemp这个临时文件夹需要提前手动创建，【~/】 在Linux上代表 【/home/用户名/】，看下build_one这个方法，enable、disable分别代表打开和关闭一些功能，重点看下下面几个配置：

```
--enable-shared
--disable-static
```

编译动态库，shared和static的开关是相对的。

```
--prefix=$PREFIX

CPU=arm
PREFIX=$(pwd)/android/$CPU
```

设置编译后文件的输出目录，$(pwd)代表当前目录，所以文件的输出路径为：ffmpeg-3.4.2/android/arm

```
--cross-prefix=$TOOLCHAIN/bin/arm-linux-androideabi-
```

设置交叉编译器，按照实际路径修改就可以了。

```
--target-os=android
```

设置编译针对的系统，网上查到很多资料说编译前需要修改configure文件，设置这项后就不用修改了，系统会自动为我们修改，效果一样。

```
--arch=arm
```

设置编译so库的架构，当前设置为arm，可以根据实际需求修改。

```
--sysroot=$SYSROOT
```

设置编译针对的平台，可以根据实际需求进行设置，当前设置为最低支持android-14版本，arm架构。

```
--extra-cflags="-Os -fpic $ADDI_CFLAGS" 

ADDI_CFLAGS="-marm"
```

还没有完全理解这几个参数的作用，有知道的同学可以给我留言啊，谢谢！

到这里，编译脚本就写完了，在当前目录下打开终端窗口，执行脚本，加上管理员权限，以防权限不足：

```
$ sudo ./build_android.sh
```

回车提示输入密码，密码不会显示出来或者显示成*号，直接输完然后回车就可以了，看下执行效果：

![开始编译](http://upload-images.jianshu.io/upload_images/3270074-79f4e8aba74868b6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

编译中：

![编译中](http://upload-images.jianshu.io/upload_images/3270074-c8c76626f4f4e603.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果一切顺利的话，大概十多分钟就可以编译完成：

![编译完成](http://upload-images.jianshu.io/upload_images/3270074-1bccd8a0be74c355.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

编译完成后，在当前目录下（ffmpeg-3.4.2）会生成一个android目录，so文件就在android > arm > lib目录下：

![生成so文件](http://upload-images.jianshu.io/upload_images/3270074-34c688f5202f57fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看下生成的so包是做什么用的：

- libavcodec：用于各种类型声音、图像编解码。

- libavformat：用于各种音视频封装格式的生成和解析，读取音视频帧等功能。

- libavutil：包含一些公共的工具函数。

- libavfilter：提供了各种音视频过滤器。

- libswresample：用于音频重采样，采样格式转换和混合。

- libswscale：用于视频场景比例缩放、色彩映射转换。

# 2.将FFmpeg编译成一个so文件

在上文中我们成功的编译了FFmpeg，但是编译出来的so文件一共有6个，如果觉的使用起来比较麻烦，可以编译成一个so文件，下面我们来修改下脚本：

```
#!/bin/bash

# 设置临时文件夹，需要提前手动创建
export TMPDIR="/home/yangle/ffmpeg/ffmpeg-3.4.2/ffmpegtemp"

# 设置NDK路径
NDK=~/ffmpeg/android-ndk-r14b

# 设置编译针对的平台，可以根据自己的需求进行设置
# 当前设置为最低支持android-14版本，arm架构
SYSROOT=$NDK/platforms/android-14/arch-arm

# 设置编译工具链，4.9为版本号
TOOLCHAIN=$NDK/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64

function build_one
{
./configure \
    --enable-cross-compile \
    --enable-static \
    --disable-shared \
    --disable-doc \
    --disable-ffmpeg \
    --disable-ffplay \
    --disable-ffprobe \
    --disable-ffserver \
    --disable-avdevice \
    --disable-doc \
    --disable-symver \
    --prefix=$PREFIX \
    --cross-prefix=$TOOLCHAIN/bin/arm-linux-androideabi- \
    --target-os=android \
    --arch=arm \
    --sysroot=$SYSROOT \
    --extra-cflags="-Os -fpic $ADDI_CFLAGS" \

$ADDITIONAL_CONFIGURE_FLAG
make clean
make
make install

# 合并生成的静态库
$TOOLCHAIN/bin/arm-linux-androideabi-ld \
-rpath-link=$SYSROOT/usr/lib \
-L$SYSROOT/usr/lib \
-L$PREFIX/lib \
-soname libffmpeg.so -shared -nostdlib -Bsymbolic --whole-archive --no-undefined -o \
$PREFIX/libffmpeg.so \
    libavcodec/libavcodec.a \
    libavfilter/libavfilter.a \
    libswresample/libswresample.a \
    libavformat/libavformat.a \
    libavutil/libavutil.a \
    libswscale/libswscale.a \
    -lc -lm -lz -ldl -llog --dynamic-linker=/system/bin/linker \
    $TOOLCHAIN/lib/gcc/arm-linux-androideabi/4.9.x/libgcc.a
}

# 设置编译后的文件输出目录
CPU=arm
PREFIX=$(pwd)/android/$CPU
ADDI_CFLAGS="-marm"
build_one
```

看下和编译单个so文件的脚本有什么区别：

生成静态库：

```
--disable-static
--enable-shared

修改为：

--enable-static
--disable-shared
```

增加下面的指令，将生成的静态库合并成一个动态库：

```
$TOOLCHAIN/bin/arm-linux-androideabi-ld \
-rpath-link=$SYSROOT/usr/lib \
-L$SYSROOT/usr/lib \
-L$PREFIX/lib \
-soname libffmpeg.so -shared -nostdlib -Bsymbolic --whole-archive --no-undefined -o \
$PREFIX/libffmpeg.so \
    libavcodec/libavcodec.a \
    libavfilter/libavfilter.a \
    libswresample/libswresample.a \
    libavformat/libavformat.a \
    libavutil/libavutil.a \
    libswscale/libswscale.a \
    -lc -lm -lz -ldl -llog --dynamic-linker=/system/bin/linker \
    $TOOLCHAIN/lib/gcc/arm-linux-androideabi/4.9.x/libgcc.a
```

重点看下下面两个配置：

```
-soname libffmpeg.so
```

指定动态库，也就是so文件的名称。

```
$TOOLCHAIN/lib/gcc/arm-linux-androideabi/4.9.x/libgcc.a
```

设置编译器的路径，根据实际路径进行修改，编译前一定要检查一遍路径是否正确！

脚本中还涉及到一些编译指令，都可以在网上查到作用，在这里就不再一一说明了，OK，开始执行编译脚本：

```
$ sudo ./build_android_all.sh
```

编译完成后，生成的libffmpeg.so就在android > arm目录下：

![将FFmpeg编译成一个so文件](http://upload-images.jianshu.io/upload_images/3270074-f44803c44786f0a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 3.遇到的问题

**NDK路径错误**

遇到这个错误仔细检查一遍设置的路径有没有问题，我就栽在了这个坑里：

![NDK路径错误](http://upload-images.jianshu.io/upload_images/3270074-8c7da51bb7a3afaf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**android文件夹带锁**

编译成功后，android文件夹是带锁的，执行下面的命令就可以解锁：

```
sudo chown 用户名 android/ -R
```

# 4.写在最后

第一次使用Linux系统和编译FFmpeg，一直在网上查命令和各种报错的解决方案，这篇文章前前后后写了一个多月，如有错误的地方，可以给我留言指正，多谢！

文中用到的FFmpeg源码、编译脚本以及编译生成的so文件已经上传至GitHub，后续还会更新，欢迎Start、Fork！

GitHub传送门：https://github.com/alidili/FFmpeg4Android

本篇文章到这里就结束了，下一篇文章将会继续介绍如何在Android项目中使用FFmepg，敬请期待！