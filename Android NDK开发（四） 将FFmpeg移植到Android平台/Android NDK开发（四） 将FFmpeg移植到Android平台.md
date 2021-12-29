# <center><big>Android NDK开发（四） 将FFmpeg移植到Android平台</big></center>

> 作者：杨乐
>
> 日期：2021/12/28
>
> 版本：1.0

[TOC]

> FFmpeg是一套可以用来记录、转换数字音频、视频，并能将其转化为流的开源计算机程序。它提供了录制、转换以及流化音视频的完整解决方案。

# 0.写在前面

在上一篇文章 《Android NDK开发（三） 在Linux环境下编译FFmpeg》 中，我们学习了如何将FFmpeg源码编译成so文件，但是这些so文件还不能直接引用到Android工程中，还需要再次编译加工才能使用，今天就让我们来学习下如何将FFmpeg移植到Android平台，以及在Android项目中如何通过命令行的方式使用FFmpeg。

# 1.环境搭建

> 操作系统：Windows 10 64bit
>
> NDK版本：android-ndk-r14b-windows-x86_64
>
> FFmpeg版本：3.4.2
>
> 编译器：Android Studio 3.0
>
> NDK构建工具：CMake

**创建FFmpeg项目**

![创建FFmpeg项目](https://github.com/alidili/NDK/raw/main/Android%20NDK%E5%BC%80%E5%8F%91%EF%BC%88%E5%9B%9B%EF%BC%89%20%E5%B0%86FFmpeg%E7%A7%BB%E6%A4%8D%E5%88%B0Android%E5%B9%B3%E5%8F%B0/resources/%E5%88%9B%E5%BB%BAFFmpeg%E9%A1%B9%E7%9B%AE.png)

在创建项目时，勾选【Include C++ support】选项，然后一路下一步，到达【Customize C++ Support】设置页：

![Customize C++ Support](https://github.com/alidili/NDK/raw/main/Android%20NDK%E5%BC%80%E5%8F%91%EF%BC%88%E5%9B%9B%EF%BC%89%20%E5%B0%86FFmpeg%E7%A7%BB%E6%A4%8D%E5%88%B0Android%E5%B9%B3%E5%8F%B0/resources/Customize%20C%2B%2B%20Support.png)

可以看到三个选项：

- C++ Standard：C++标准，选择【Toolchain Default】会使用默认的CMake配置。

- Exceptions Support：支持C++异常处理，标志为 -fexceptions。

- Runtime Type Information Support：支持运行时类型识别，标志为 -frtti，程序能够使用基类的指针或引用来检查这些指针或引用所指的对象的实际派生类型。

在这里我们使用默认C++标准，不勾选下面的两个选项，点击【Finish】按钮进入下一个环节，看下项目结构：

![项目结构](https://github.com/alidili/NDK/raw/main/Android%20NDK%E5%BC%80%E5%8F%91%EF%BC%88%E5%9B%9B%EF%BC%89%20%E5%B0%86FFmpeg%E7%A7%BB%E6%A4%8D%E5%88%B0Android%E5%B9%B3%E5%8F%B0/resources/%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.png)

# 2.FFmpeg移植

**准备工作**

- 将编译生成的include文件夹拷贝至 **src\main\cpp** 目录下。

- 将FFmpeg源码中fftools目录下的 **cmdutils.c、cmdutils.h、ffmpeg.c、ffmpeg.h、ffmpeg_filter.c、ffmpeg_opt.c** 类拷贝至 **src\main\cpp** 目录下。

- 将编译生成的so文件拷贝至 **src\main\jniLibs\armeabi** 目录下，如果没有此目录，新建就可以了。

- 将 **src\main\jniLibs\armeabi** 目录下的 **native-lib.cpp** 重命名为 **ffmpeg_cmd.c**。 

- 设置ABI，本篇文章只编译armeabi架构的so文件，所以需要在build.gradle中设置一下：

```
android {
    ...
	
    defaultConfig {
        ...
		
        ndk {
            abiFilters "armeabi"
        }
    }
}
```

看下此时的项目结构：

![项目结构](https://github.com/alidili/NDK/raw/main/Android%20NDK%E5%BC%80%E5%8F%91%EF%BC%88%E5%9B%9B%EF%BC%89%20%E5%B0%86FFmpeg%E7%A7%BB%E6%A4%8D%E5%88%B0Android%E5%B9%B3%E5%8F%B0/resources/%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%842.png)

**编译脚本**

准备工作到这里就基本完成了，下面来写一下CMake的构建脚本CMakeLists.txt：

```
# 设置Cmake版本
cmake_minimum_required(VERSION 3.4.1)

# 设置cpp目录路径
set(CPP_DIR ${CMAKE_SOURCE_DIR}/src/main/cpp)

# 设置jniLibs目录路径
set(LIBS_DIR ${CMAKE_SOURCE_DIR}/src/main/jniLibs)

# 添加库
add_library( # 库名称
             ffmpeg

             # 动态库，生成so文件
		     SHARED

		     # 源码
		     ${CPP_DIR}/cmdutils.c
		     ${CPP_DIR}/ffmpeg.c
		     ${CPP_DIR}/ffmpeg_filter.c
		     ${CPP_DIR}/ffmpeg_opt.c
		     ${CPP_DIR}/ffmpeg_cmd.c )

# 用于各种类型声音、图像编解码
add_library( # 库名称
             avcodec

             # 动态库，生成so文件
             SHARED

             # 表示该库是引用的不是生成的
             IMPORTED )

# 引用库文件
set_target_properties( # 库名称
                       avcodec

                       # 库的路径
                       PROPERTIES IMPORTED_LOCATION
                       ${LIBS_DIR}/armeabi/libavcodec.so )

# 用于各种音视频封装格式的生成和解析，读取音视频帧等功能
add_library( avformat
             SHARED
             IMPORTED )

set_target_properties( avformat
                       PROPERTIES IMPORTED_LOCATION
                       ${LIBS_DIR}/armeabi/libavformat.so )

# 包含一些公共的工具函数
add_library( avutil
             SHARED
             IMPORTED )

set_target_properties( avutil
                       PROPERTIES IMPORTED_LOCATION
                       ${LIBS_DIR}/armeabi/libavutil.so )

# 提供了各种音视频过滤器
add_library( avfilter
             SHARED
             IMPORTED )

set_target_properties( avfilter
                       PROPERTIES IMPORTED_LOCATION
                       ${LIBS_DIR}/armeabi/libavfilter.so )

# 用于音频重采样，采样格式转换和混合
add_library( swresample
             SHARED
             IMPORTED )

set_target_properties( swresample
                       PROPERTIES IMPORTED_LOCATION
                       ${LIBS_DIR}/armeabi/libswresample.so )

# 用于视频场景比例缩放、色彩映射转换
add_library( swscale
             SHARED
             IMPORTED )

set_target_properties( swscale
                       PROPERTIES IMPORTED_LOCATION
                       ${LIBS_DIR}/armeabi/libswscale.so )

# 引用源码 ../代表上级目录
include_directories( ../../ffmpeg-3.4.2/
                     ${CPP_DIR}/include/ )

# 关联库
target_link_libraries( ffmpeg
                       avcodec
                       avformat
                       avutil
                       avfilter
                       swresample
                       swscale )
```

脚本中已经写了很全的注释，不再多说，重点看下include_directories这个方法，可以看到里面引用了FFmpeg和编译生成的include目录源码，其中有很多是重复的，看到网上很多文章都是只引用了FFmpeg源码就可以了，但是我试了下会有几个文件找不到，所以就两个都引用了。

**修改FFmpeg源码**

- src\main\cpp\ffmpeg.c

修改main方法：

```
// 修改前：
int main(int argc, char **argv)

// 修改后：
int run(int argc, char **argv)
```

执行完指令之后重新初始化：

```
// return之前增加：

nb_filtergraphs = 0;
progress_avio = NULL;
input_streams = NULL;
nb_input_streams = 0;
input_files = NULL;
nb_input_files = 0;
output_streams = NULL;
nb_output_streams = 0;
output_files = NULL;
nb_output_files = 0;
```

注释掉下面的代码，否则执行完指令后会crash：

```
exit_program(received_nb_signals ? 255 : main_return_code);
```

- src\main\cpp\ffmpeg.h

```
// 增加：
int run(int argc, char **argv);
```

- src\main\cpp\cmdutils.c

修改退出方法，原代码中退出后，会直接把APP也退出，所以需要修改下：

```
// 修改前：
void exit_program(int ret)
{
    if (program_exit)
        program_exit(ret);

    exit(ret);
}

// 修改后：
int exit_program(int ret)
{
    return ret;
}
```

- src\main\cpp\cmdutils.h

```
// 修改前：
void exit_program(int ret) av_noreturn;

// 修改后：
int exit_program(int ret);
```

**修改ffmpeg_cmd.c**

```
#include <jni.h>
#include "ffmpeg.h"

JNIEXPORT jint

JNICALL
Java_com_yl_ffmpeg4android_MainActivity_run(
        JNIEnv *env, jclass obj, jobjectArray commands) {
    int argc = (*env)->GetArrayLength(env, commands);
    char *argv[argc];

    int i;
    for (i = 0; i < argc; i++) {
        jstring js = (jstring) (*env)->GetObjectArrayElement(env, commands, i);
        argv[i] = (char *) (*env)->GetStringUTFChars(env, js, 0);
    }
    return run(argc, argv);
}
```

方法很简单，传入指令（String[] 类型），然后调用ffmpeg.h中的run方法执行这些指令，点击编译按钮，不出意外的话，肯定会报错的，不要方，继续往下看。

**设置编译模式**

先看下报错：

![编译报错](https://github.com/alidili/NDK/raw/main/Android%20NDK%E5%BC%80%E5%8F%91%EF%BC%88%E5%9B%9B%EF%BC%89%20%E5%B0%86FFmpeg%E7%A7%BB%E6%A4%8D%E5%88%B0Android%E5%B9%B3%E5%8F%B0/resources/%E7%BC%96%E8%AF%91%E6%8A%A5%E9%94%99.png)

大概意思是编译模式不对，话说这个报错真的是让人头疼，查完百度查谷歌，也没有查到解决方法，后来一遍又一遍的看CMake文档，一个参数一个参数的试，终于解决了，心情瞬间舒畅了，来看下如何解决：

```
android {
    ...
	
    defaultConfig {
        ...
		
        externalNativeBuild {
            cmake {
                arguments '-DANDROID_ARM_MODE=arm'
            }
        }
}
```

把ANDROID_ARM_MOD设置为arm就可以了，默认是thumb，OK，继续编译，什么！又报错：

![编译报错](https://github.com/alidili/NDK/raw/main/Android%20NDK%E5%BC%80%E5%8F%91%EF%BC%88%E5%9B%9B%EF%BC%89%20%E5%B0%86FFmpeg%E7%A7%BB%E6%A4%8D%E5%88%B0Android%E5%B9%B3%E5%8F%B0/resources/%E7%BC%96%E8%AF%91%E6%8A%A5%E9%94%992.png)

有一些方法没有定义，还好，在ffmpeg.c中加几个空方法：

```
HWDevice *hw_device_get_by_name(const char *name) {
}

int hw_device_init_from_string(const char *arg, HWDevice **dev) {
}

void hw_device_free_all(void) {
}

int hw_device_setup_for_decode(InputStream *ist) {
}

int hw_device_setup_for_encode(OutputStream *ost) {
}
```

再次编译：

![编译完成](https://github.com/alidili/NDK/raw/main/Android%20NDK%E5%BC%80%E5%8F%91%EF%BC%88%E5%9B%9B%EF%BC%89%20%E5%B0%86FFmpeg%E7%A7%BB%E6%A4%8D%E5%88%B0Android%E5%B9%B3%E5%8F%B0/resources/%E7%BC%96%E8%AF%91%E5%AE%8C%E6%88%90.png)

久违的信息出现了，看下编译出的so文件在哪里：

![生成libffmpeg.so](https://github.com/alidili/NDK/raw/main/Android%20NDK%E5%BC%80%E5%8F%91%EF%BC%88%E5%9B%9B%EF%BC%89%20%E5%B0%86FFmpeg%E7%A7%BB%E6%A4%8D%E5%88%B0Android%E5%B9%B3%E5%8F%B0/resources/%E7%94%9F%E6%88%90libffmpeg.so.png)

# 3.测试

写一个简单的Demo来测试下编译成功的FFmpeg，将一个视频的前100帧截取成一个gif动图，上代码：

```
public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    static {
        System.loadLibrary("ffmpeg");
    }

    private ImageView ivGif;
    private Button btnConvert;
    private ProgressDialog progressDialog;
    // 设备根目录路径
    private String path = Environment.getExternalStorageDirectory().getAbsolutePath();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        ivGif = findViewById(R.id.iv_gif);
        btnConvert = findViewById(R.id.btn_convert);
        btnConvert.setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
        // 截取视频的前100帧
        final String cmd = "ffmpeg -i " + path + "/video.mp4 -vframes 100 -y -f gif -s 480×320 " + path + "/video_100.gif";
        // 显示loading
        progressDialog = new ProgressDialog(this);
        progressDialog.setTitle("截取中...");
        progressDialog.show();

        new Thread() {
            @Override
            public void run() {
                super.run();
                // 执行指令
                cmdRun(cmd);

                // 隐藏loading
                runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        progressDialog.dismiss();
                        progressDialog = null;

                        // 显示gif
                        Glide.with(MainActivity.this)
                                .load(new File(path + "/video_500.gif"))
                                .into(ivGif);
                    }
                });
            }
        }.start();
    }

    /**
     * 以空格分割指令，生成String类型的数组
     *
     * @param cmd 指令
     * @return 执行code
     */
    private int cmdRun(String cmd) {
        String regulation = "[ \\t]+";
        final String[] split = cmd.split(regulation);
        return run(split);
    }

    /**
     * ffmpeg_cmd中定义的run方法
     *
     * @param cmd 指令
     * @return 执行code
     */
    public native int run(String[] cmd);
}
```

看下截取的gif：

![截取的gif](https://github.com/alidili/NDK/raw/main/Android%20NDK%E5%BC%80%E5%8F%91%EF%BC%88%E5%9B%9B%EF%BC%89%20%E5%B0%86FFmpeg%E7%A7%BB%E6%A4%8D%E5%88%B0Android%E5%B9%B3%E5%8F%B0/resources/%E6%88%AA%E5%8F%96%E7%9A%84gif.gif)

# 4.写在最后

文中如有错误的地方，可以给我留言指正，多谢！

文中用到的FFmpeg源码、编译脚本以及编译生成的so文件已经上传至GitHub，后续还会更新，欢迎Start、Fork！

GitHub传送门：https://github.com/alidili/FFmpeg4Android