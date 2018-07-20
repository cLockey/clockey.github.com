---
layout:     post
title:      "Unity与C++交互入门(1)"
subtitle:   " "
date:       2018-07-19
author:     "QiiJii"
header-img: "img/post-bg-unity-bot.jpg"
tags:
    - Unity
---

# Unity与C++交互入门(1)
### 一、什么情况下需要使用C++

1.大量的复杂运算，C++比C#效率高。

2.大多数语言都有调用C++ DLL的途径，若项目中某个模块客户端和服务器都需要使用，可以考虑用C++实现该模块，这样客户端和服务器就不需要重复编写该模块，只需写一些胶水代码即可。

### 二、基本概念

**1.托管(Managed)和非托管(Unmanaged)：**.Net的运行环境是CLR(Common Language Runtime)，运行在CLR上的代码成为托管代码(Managed Code)，CLR提供了自动的垃圾回收机制(GC)。而C++是编译后直接由操作系统执行的代码，不运行在CLR上，所以C++属于非托管代码(Unmanaged Code)。（注：另有托管C++，本质上也属于.Net范畴，不在讨论范围内）

**2.P/Invoke：**P/Invoke（Platform Invoke，平台调用）使得我们可以在托管代码中调用非托管函数，Unity与C++的交互都是通过P/Invoke实现。

### 三、Demo

**1.创建C++ DLL**

在VS中新建C++项目，这里理应选择DLL，但是建议先选成控制台，等我们编写的函数先在控制台调试没问题后可以在项目属性中改为DLL。

![新建C++项目](/img/in-post/unity-cpp-interop-1/01-cpp-new-project.jpg)

![修改项目属性](/img/in-post/unity-cpp-interop-1/02-change-to-dll.jpg)

**2.新建一个类Bridge.h和Bridge.cpp**

![新建Bridge类](/img/in-post/unity-cpp-interop-1/03-new-class.jpg)

```c++
//Bridge.h

#ifdef WIN32
#ifdef	UNITY_CPP_INTEROP_DLL_BRIDGE
#define UNITY_CPP_INTEROP_DLL_BRIDGE	__declspec(dllexport)
#else
#define UNITY_CPP_INTEROP_DLL_BRIDGE	__declspec(dllimport)
#endif
#else
// Linux
#define UNITY_CPP_INTEROP_DLL_BRIDGE
#endif

extern "C"
{
	UNITY_CPP_INTEROP_DLL_BRIDGE int Internal_Add(int a, int b);
}
```

```c++
//Bridge.cpp

#include "Bridge.h"

extern "C"
{	
	int Internal_Add(int a, int b)
	{
		return a + b;
	}
}

```

**3.拷贝DLL**

右键项目生成DLL后，将DLL拷贝到Unity项目中。

![拷贝DLL](/img/in-post/unity-cpp-interop-1/04-copy-dll-to-unity.jpg)

**4.在C#中调用DLL**

在Unity中新建脚本填入如下内容。

```c#
    // Use this for initialization
    void Start()
    {
        int a = 5, b = 6;
        Debug.LogError(string.Format("Internal_Add(): {0} + {1} = {2}", a, b, Internal_Add(a, b)));
        Debug.LogError(string.Format("Add(): {0} + {1} = {2}", a, b, Add(a, b)));
    }

    [DllImport("UnityCppInterop")]
    private static extern int Internal_Add(int a, int b);


    [DllImport("UnityCppInterop", EntryPoint = "Internal_Add")]
    private static extern int Add(int a, int b);
```

其中由两个**extern**修饰的函数与C++中**Internal_Add()**函数对应，二者的区别在于是否指定了**EntryPoint**（入口），**EntryPoint**参数指明了从**UnityCppInterop.dll**中调用的函数名，如果未指定，则会调用与C#中函数名相同的C++函数。本例中，二者调用的是C++中的同一个函数，输出如下：

![Demo输出](/img/in-post/unity-cpp-interop-1/05-demo-print.jpg)

如果将**Add()**的**EntryPoint**删除，会报**EntryPointNotFoundException**异常：

![去掉EntryPoint的输出](/img/in-post/unity-cpp-interop-1/06-demo-print-exception.jpg)



### 四、Android Studio中编译so文件

上面只是介绍了在Windows平台Unity与C++交互的过程，但发布到Android平台后DLL是无法使用的，我们需要将C++源码编译成Android平台可用的库文件xxx.so。

**1.新建Android工程**

勾选**Include C++ support**，一路下一步即可。

![新建Android工程](/img/in-post/unity-cpp-interop-1/07-android-new-projet.png)

创建好后检查工程属性中是否指定了ndk路径：

![QQ图片20180719172301](/img/in-post/unity-cpp-interop-1/08-check-ndk.png)

修改**app的build.gradle**内容（供参考）如下：

```c++
apply plugin: 'com.android.application'

android {
    compileSdkVersion 26
    buildToolsVersion "26.0.3"
    defaultConfig {
        applicationId "com.zqj.unitycppinterop"
        minSdkVersion 14
        targetSdkVersion 26
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
        externalNativeBuild {
            cmake {
                cppFlags ""
            }
        }

        ndk{
            moduleName "native-lib"
            abiFilters "x86", "x86_64", "armeabi-v7a", "arm64-v8a"
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
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile 'com.android.support:appcompat-v7:26.+'
}

```

编译so可以使用**CMake**或**Android.mk**，本文介绍**CMake**，如果没有安装，可在Android Studio的SDK Tools中下载。将之前编写的C++文件拷贝到cpp目录下，最好将头文件和源文件分类，**native-lib.cpp**是新建工程时自动创建的，可以删掉：

![项目结构](/img/in-post/unity-cpp-interop-1/09-project-structure.png)

**CMakeList.txt**

```c#

cmake_minimum_required(VERSION 3.4.1)

#设置头文件目录
set(INCLUDE_DIR
    "src/main/cpp/include/"
    )
include_directories(${INCLUDE_DIR})

#需要编译的源文件
file(GLOB_RECURSE SRC_FILE
    "src/main/cpp/src/*.cpp"
    )

#################################
add_library( # 最后生成的库名称
             UnityCppInterop

             # Sets the library as a shared library.
             SHARED

             # Provides a relative path to your source file(s).
             # Associated headers in the same location as their source
             # file are automatically included.
             src/main/cpp/native-lib.cpp
             ${SRC_FILE} )

find_library( # Sets the name of the path variable.
              log-lib

              # Specifies the name of the NDK library that
              # you want CMake to locate.
              log )
```

Make Project成功后，生成的**libUnityCppInterop.so**位于**app/build/intermediates/cmake/debug**中，把**armeabi-v7a**下的so文件拷贝到**Unity工程Plugins/Android/**目录下。这里的so文件名比DLL多了lib前缀，不需要修改，Unity会自动识别。

### 总结

本文主要介绍Unity与C++交互的基础知识，Demo只演示了最基本类型int型数据的交互，而实际项目中肯定会涉及到string, struct, class, 数组等复杂类型数据的传递，这其中有很多需要注意的地方，下一篇会针对这些进行介绍。