&emsp;上一篇文章介绍了一下JIN的基础内容，感兴趣的小伙伴可以移步：

	[Android JNI编程—JNI基础](http://www.jianshu.com/p/aba734d5b5cd)

看了上面这篇文章的小伙伴有可能感觉比较的晦涩难懂，希望可以结合这篇文章小伙伴们可以对它有更好的理解。今天在开始主要内容之前我想从个人角度对Java(Android平台)和C/C++、NDK编译和IDE编译做一个对比，希望可以帮助大家克服对C/C++以及NDK编译的畏惧心理。

1.Java与C/C++

&emsp;其实小伙伴们知道Java的mama是谁吗？当然是C/C++啦，那Java的baba是谁啊？当然是创造Java的这个家伙啦？好啦，baba和mama都有了，那么作为孩子的Java肯定和它们有几分相似啦！这里以Eclipse开发Android为例(当然目前我们肯定都在使用AS喽)，当我们需要使用别人的开源库或者第三方的库(例如：腾讯的Bugly)时，一般是把一个jar包copy到本地的lib文件夹下面，编译过后我们就可以直接编写代码调用API了，当然每次在调用API的时候IDE自动的帮我们import了相应的类。那么对于C/C++呢，我们要使用别人的库的时候是怎么做的，一般是给你一个包含了很多.h文件的文件夹，这里的.h文件就是头文件，里面声明了很多函数，但是他们大多数都没有具体实现，具体实现的代码被编译成了一个静态库或者so库文件。当需要调用相应的函数，我们首先把包含这个函数的头文件#include<>(这里有没有像我们的import啊)，之后我们就可以调用了。

2.IDE编译与NDK编译

&emsp;前面我们介绍了Java与C/C++引用第三方库，并怎么调用别人的API，现在我们要做的就是编译了，对于我们使用IDE编译的话，只要点击一下按钮，嗖嗖的就完了，但是这个嗖嗖的过程中我们知道它把.java的文件编译为.class文件，并且帮我们找到了需要链接的jar包，总之帮我们自动的完成了很多工作。但是对于Android的JNI编写的C/C++我们必须自己写编译规则，那么在哪里写呢，当然是在Android.mk文件夹下面啦。这个文件是用MakFile文件语法写的。主要就是告诉ndk-build命令我们需要编译哪些文件，包含哪些头文件，链接哪些第三方的库。总之，NDK编译需要我们自己编写编译的规则。

3.简单的总结

Java:import C/C++:include

Java:jar C/C++:.a或者.so

Java:智能的IDE自动编译 C/C++:需要自己手动编写编译规则

##了解基本的环境

&emsp;最基本的当然是要把开发环境搭建起来啦，第一步还是要下载NDK开发工具包，这里提供一个非官方的[下载地址](https://github.com/inferjay/AndroidDevTools)，这里面大家也可以下载其他的Android相关的开发工具(在此感谢作者的辛勤付出)，这样大家就不要科学上网也可以下载。根据自己的平台下载相应的开发工具包并安装，这里就不详细的介绍了，安装好之后我们来看看工具包的目录吧(这里以Mac环境介绍的，如果是其他的平台，请自己对号入座)，如下图所示：


source目录存放的是Android相关的源码目录，

sample目录存放的是Google官方为JNI编程写的一些Demo程序，大家可以把这里面的Demo好好的学习一下，并且将这些Demo都自己实现一下，我相信会对JNI编程会有一定的掌握。

prebuild目录存放的是针对各个平台的gdb调试使用的gdbserver。

toolchains目录存放的是一些工具链，目前我对这里面的工具链也是不是很了解，我就只用过nm的命令行工具分析so库，其他的我也没有用过。

build目录存放的是跟编译相关编译工具、编译脚本以及编译的配置文件；

platform存放的是编译JNI程序时需要的头文件、so库以及静态库，我们看看它的目录吧，因为在后面编程中我们会使用该文件夹下的文件。

这里存放了的Android不用版本对应的lib和include文件，其中可以看到Android3-Android8文件夹下只存放了针对ARM架构编译的文件，Android9以上就有了针对x86和mips架构进行编译的文件了(其实这里说的不同架构，主要是说CPU使用的指令集不一样，感兴趣的大家可以自行了解)。每个目录最后都是有下面两个文件：include和lib，其中include文件夹下存放一些头文件，lib文件夹下存放了一些so库和静态库，我们在后面编写和编译都会用到这些文件，其中最重要的当然是jni.h头文件。

test目录存放的是NDK测试的相关的代码

doc主要存放了一些关于NDK帮助的.html以及文档


##实现JNI编程

&emsp;下面我还是通过直接创建一个工程，来分析如何实现JNI编程。我在Android Studio上新建一个工程AndroidJNI，并在main目录下新建jni目录，工程目录结构如下图所示：

在com/github/songnick/jni包下创建类AndroidJni，该类的详细代码如下：
	
	public class AndroidJni {

    static{
        System.loadLibrary("main");
    }

    public native void dynamicLog();

    public native void staticLog();

	}

这里我们声明了一个static区域，该区域中调用的System.loadLibrary()方法。这里就不详细的分析代码了，在上一篇文章已经详细的做了分析。

&emsp;那么我们来看看如何生成libmain.so库的。在main/jni目录下创建了以下的文件：

1.main.cpp：主要是告诉Java虚拟机调用java方法调用的C/C++代码。怎么做到的，我们看看main.cpp的源码：
	
	#include <jni.h>

	#define LOG_TAG "main.cpp"

	#include "mylog.h"

	static void nativeDynamicLog(JNIEnv *evn, jobject obj){

		LOGE("hell main");
	}

	JNIEXPORT void JNICALL Java_com_github_songnick_jni_AndroidJni_staticLog (JNIEnv *env, jobject obj)
	{
  		LOGE("static register log ");

	}

	JNINativeMethod nativeMethod[] = {{"dynamicLog", "()V", (void*)nativeDynamicLog},};

	JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *jvm, void *reserved) {

		JNIEnv *env;
		if (jvm->GetEnv((void**) &env, JNI_VERSION_1_4) != JNI_OK) {
			return -1;
		}
		LOGE("JNI_OnLoad comming");
		jclass clz = env->FindClass("com/github/songnick/jni/AndroidJni");

		env->RegisterNatives(clz, nativeMethod, sizeof(nativeMethod)/sizeof(nativeMethod[0]));


		return JNI_VERSION_1_4;
	}
	
(1)首先我们引入了头文件jni.h，该头文件的位置在前面已经介绍了，就是在platform文件夹下面，这里还引入了mylog.h的头文件，这个是我自己写的一个头文件，主要是重新定义了打印log的函数名。这段代码的功能以及意思在这里也不讨论了，感兴趣的可以看看上一篇文章的分析。

2.mylog.h：还是看看代码吧，代码还是比较的简单。


	#include <android/log.h>

	#ifndef LOG_TAG
	#error You should define LOG_TAG
	#endif

	#define LOGD(...)  __android_log_print(ANDROID_LOG_DEBUG,LOG_TAG,__VA_ARGS__)
	#define LOGE(...)  __android_log_print(ANDROID_LOG_ERROR,LOG_TAG,__VA_ARGS__)
	

	其实这里就是宏定义了两个函数，LOGD和LOGE。这里我们看到引用了头文件<android/log.h>，那么这个头文件在哪里呢，前面的#ifndef防止重复引用该头文件。

2.Android.mk：编写一些编译时的规则，告诉编译器怎么去编译和链接程序。当我们在该目录下使用ndk-build进行编译的时候，该命令会找到Android.mk文件，并解析编写的规则，根据这些规则编译相应的文件并生成对应的文件(so库、可执行文件等)，还是看看该文件下的源码：

	LOCAL_PATH:= $(call my-dir)

	include $(CLEAR_VARS)

	LOCAL_SRC_FILES := main.cpp

	LOCAL_LDFLAGS := -L$(SYSROOT)/usr/lib/ -llog

	LOCAL_MODULE = main

	include $(BUILD_SHARED_LIBRARY)


这里面涉及到makefile文件的一些语法，大家可以自己看看这方面的知识，其实也是和gradle类似的语法。多看看别人写的，然后自己再多练习就可以了。这里我还是简单解析这段代码。

这里的符号:=为变量赋值，定义了LOCAL_PATH 调用$(call my-dir)返回当前目录，

include $(CLEAR_VARS)初始化变量。

LOCAL_SRC_FILES为编译时需要编译的源文件， 

LOCAL_LDFLAGS编译时需要连接的so库，这里链接了上面我们分析的platform目录下的liblog.so库。 

LOCAL_MODULE编译后生成的so库的名字，

include $(BUILD_SHARED_LIBRARY)表示编译成共享库，这里我们是可以编译成三种类型：第一种就是so库；第二种是静态库；第三种是可执行文件(就是编译成可以在命令行直接运行的文件，这里大家可以看看网上讨论的关于怎么创建一个杀不死的servic，有的就是编译了一个守护进程，这个守护进程就是编译好的一个可执行文件，然后在命令行运行)。



上面的文件创建好之后，通过命令行进入jni文件目录下，使用ndk－build命令，就会生成libmain.so的库，并且存放在上一个目录的libs目录下。如下图所示：



编译好后，我们要在build.gradle目录下配置so库的引用目录。如下所示：

		sourceSets {
        main {
            	jni.srcDirs = []
            	jniLibs.srcDirs = ['src/main/libs']
        	}
    	}

&emsp;其实JNI编程的最终目的就是Java可以调用C/C++写的代码，那么接下来就主要从以下三点学习JNI编程：

	1.Java调用C/C++的代码——两种方法

	2.C/C++调用Java的代码

	3.编译SO库使用

###实现Java调用C/C++代码

&emsp;我们知道在Java程序都是跑在虚拟机上的，那么在调用某个方法的时候肯定是根据自己的一套机制去寻找调用的方法。我们怎么告诉