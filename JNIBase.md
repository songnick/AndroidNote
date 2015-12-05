&emsp;最近看到了很多关于热补的开源项目——Depoxed(阿里)、AnFix（阿里）、DynamicAPK(携程)等，它们都用到了JNI编程，并且JNI编程也贯穿了Android系统，学会JNI编程对于我们学习研究Android源码、Android安全以及Android安全加固等都是有所帮助的。但是对于我们这些写Android应用的，大部分时间都是在使用Java编程，很少使用C/C++编程，对于JNI编程也了解的比较少，那么我们就来简单的了解一下JNI编程的基础吧。

##什么是JNI，怎么使用
&emsp;JNI——Java Native Interface，它是Java平台的一个特性(并不是Android系统所拥有的)。其实主要是定义了一些native的接口，让开发者可以通过调用这些接口实现Java方法调用C/C++的native方法，C/C++的native方法也可以调用Java的代码。这样就可以发挥各个语言的特点了。那么怎么使用JNI呢，一般情况下我们首先是将写好的C/C++代码编译成对应平台的动态库(windows一般是dll文件，linux一般是so文件等)，这里我们是针对Android平台，所以只讨论so库。

##从一个栗子说起

&emsp;这里还是直接从代码说起，这样更加形象和直观，也便于理解。今天使用的Java代码如下：

	public class AndroidJni {

    	static{
        	System.loadLibrary("main");
    	}

    	public native void dynamicLog();

    	public native void staticLog();

	}

这里我们定义了两个声明为native的方法，并声明了一块静态区域，并在该静态区域类加载名为libmain.so的库，这里我们说是libmain.so库，但是加载的时候却只写了“main”，其实大家只要知道这是约定的就可以了。	

相应的C/C++代码如下：

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

这里引用了两个头文件，jni.h和mylog.h，其中jni.h是定义了很多我们使用的JNI接口，mylog.h是我自己定义的打印Android log的函数，在JNI编程实践中我将详细介绍，这里只要知道它的功能就可以了。
这里暂时不讨论怎么编译成so库以及so库的一些规范，会在下一个JNI实践编程中介绍。这里假定将上面的C/C++程序编译成了一个叫libmain.so的文件。在Java层使用System.loadLibarary("main")方法将该so库加载起来，使得dynamicLog()、staticLog()和对应的Java_com_github_songnick_jni_AndroidJni_staticLog()、nativeDynamicLog()两个native方法链接起来，当然这部分工作都是由Java虚拟机完成的，那么具体是怎么完成的，接下来将根据上面的代码进行分析。

###静态注册native方法

&emsp;在C/C++代码中看到了JNIEXPORT和JNICALL是两个宏定义，他主要的作用就是确定JNI函数和对应的native方法，在AndroidJni.java的类中声明了staticLog()的native方法，他对应的JNI函数就是Java_com_github_songnick_jni_AndroidJni_staticLog(),那么怎知道链接的呢，在Java虚拟机加载so库是，如果发现上面两个含有上面两个宏定义的方法时就会链接到对应Java层的native方法，那么怎么知道呢，其实自己观察JNI函数名它的构成其实是：Java_PkgName_ClassName_NativeMethodName，以Java为前缀，并且用“_”下划线将包名、类名以及native方法名连接起来就是对应的JNI函数了。一般情况下我们可以自己手动的去按照这个规则写，但是如果native方法特别多，那么还是有一定的工作量，并且在写的过程中不小心就有可能写错，其实Java给我们提供了javah的工具帮助生成相应的头文件。这个过程中我们并没有指定Java层的native方法对应的JNI函数，只是按照JNI编程的规则编写了对应的JNI函数，所以该函数名是唯一的

###动态注册

&emsp;上面我们介绍了静态注册native方法的过程，就是Java层声明的native方法和JNI函数是一一对应的，那么有没有方法让Java层的native方法和任意的JNI函数链接起来，当然是可以的，那么接下来就看看如何实现动态注册的。

####JNI_OnLoad函数

&emsp;当我们使用System.loadLibarary()方法加载so库的时候，Java虚拟机就会找到这个函数并调用该函数，因此可以在该函数中做一些初始化的动作，其实这个函数就是相当于Activity中的onCreate()方法。该函数前面有三个关键字，分别是JNIEXPORT、JNICALL和jint，其中JNIEXPORT和JNICALL是两个宏定义，用于指定该函数是JNI函数。jint是JNI定义的数据类型，因为Java层和C/C++的数据类型或者对象不能直接相互的引用或者使用，JNI层定义了自己的数据类型，用于衔接Java层和JNI层，至于这些数据类型我们在后面介绍。这里的jint对应Java的int数据类型，该函数返回的int表示当前使用的JNI的版本，其实类似于Android系统的API版本一样，不同的JNI版本中定义的一些不同的JNI函数。该函数会有两个参数，其中*jvm为Java虚拟机实例JavaVM定义了以下函数：

	DestroyJavaVM
   	AttachCurrentThread
   	DetachCurrentThread
   	GetEnv

其中我们可以看到上面的JNI_OnLoad函数中有如下代码：

	JNIEnv *env;
	if (jvm->GetEnv((void**) &env, JNI_VERSION_1_4) != JNI_OK) {

		return -1;
	}

这里调用了GetEnv函数获取JNIEnv借口指针，其实JNIEnv借口是指向一个函数表的，该函数表指向了对应的JNI函数，我们通过调用这些JNI函数完成完成JNI编程，

####获取Java对象，完成动态注册

上面介绍了如何获取JNIEnv结构体指针，得到这个结构体指针后我们就可以调用JNIEnv中的RegisterNatives函数完成动态注册native方法了。该方法如下：

	jint RegisterNatives(jclass clazz, const JNINativeMethod* methods, jint nMethods)

第一个参数是Java层对应包含native方法的对象(这里就是AndroidJni对象)，通过调用JNIEnv对应的函数获取class对象(FindClass函数的参数为需要获取class对象的类描述符)：

	jclass clz = env->FindClass("com/github/songnick/jni/AndroidJni");

第二个参数是JNINativeMethod结构体指针，这里的JNINativeMethod结构体是描述Java层native方法的，它的定义如下：

	typedef struct {
    	const char* name;//Java层native方法的名字
    	const char* signature;//Java层native方法的描述符，因为我们知道在java中存在重载的问题。
    	void*       fnPtr;//对应JNI函数的指针
	} JNINativeMethod;

第三个参数为注册native方法的数量。一般会动态注册多个native方法，首先会定义一个JNINativeMethod数组，然后将该数组指针作为RegisterNative函数的参数传入，所以这里定义了如下的JNINativeMethod数组：

	JNINativeMethod nativeMethod[] = {{"dynamicLog", "()V", (void*)nativeDynamicLog},};


最后调用RegisterNative函数完成动态注册：

	env->RegisterNatives(clz, nativeMethod, sizeof(nativeMethod)/sizeof(nativeMethod[0]));

&emsp;，

然后调用了RegisterNatives通过动态注册的方法将AndroidJni中的


&emsp;JNIEXPORT和JNICALL是两个宏定义，在jni.h中他们定义如下：

	#define JNIIMPORT
	#define JNIEXPORT  __attribute__ ((visibility ("default")))
	#define JNICALL __NDK_FPABI__

这两个宏定义的作用是指定JNI方法实现和对用的Java方法。当使用
这里暂时不讨论怎么编译成so库的过程，会在下一个JNI实践编程中介绍。这里假定将上面的C/C++程序编译成了一个叫libmain.so的文件。在Java层使用System.loadLibarary()加载so库时就会讲该方法与对应的Java方法链接其他，那么它怎么知道是那么个方法呢，我们看看方法的名字就知道啦，上面的代码可以看到Java_com_github_songnick_jni_AndroidJni_staticLog的方法名，在Java后面将相应的"_"换成"."就是对应Java层相应包名下某个类的方法啦。它的返回值为jint类型，该返回值位于JNIEXPORT和JNICALL之间。

##数据类型
上面我们提到JNI定义了一些自己的数据类型和数据结构。因为它是衔接Java层和C/C++层的，如果有一个对象传递下来，那么对于C/C++来说是没办法识别这个对象的，同样的如果C/C++的指针对于Java层来说它也是没办法识别的，那么就需要JNI进行匹配。所以需要定义一些自己的数据类型。

1.原始数据类型

| Java Type  |  Native Typ   |  Description     |
| ---------- |:-------------:| ----------------:|
| boolean    | jboolean      | unsigned 8 bits  |
| byte       | jbyte         |   signed 8 bits  |
| char       | jchar         | unsigned 16 bits |
| short      | jshort        | signed 16 bits   |
| int        | jint          | signed 32 bits   |
| long       | jlong         | signed 64 bits   |
| float      | jfloat        | 32 bits			|
| double	 | jdouble		 | 64 bits			|
| void 		 | void 		 | N/A     			|

2.引用类型
	
	jobject 					(all Java objects)
	|
	|-- jclass					(java.lang.Class objects)
	|-- jstring					(java.lang.String objects)
	|-- jarray					(array)
	|	  |--jobjectArray       (object arrays)
	|	  |--jbooleanArray		(boolean arrays)
	|	  |--jbyteArray			(byte arrays)
	|	  |--jcharArray			(char arrays)
	|	  |--jshortArray		(short arrays)
	|	  |--jintArray			(int arrays)
	|     |--jlongArray			(long arrays)
	|	  |--jfloatArray		(float arrays)
	|	  |--jdoubleArray 		(double arrays)
	|
	|--jthrowable

3.方法和变量的ID
&emsp;当需要调用Java中的某个方法的时候我们首先要获取它的ID，根据ID调用该方法，变量也是这样，他们定义的如下：

	struct _jfieldID;              /* opaque structure */ 
	typedef struct _jfieldID *jfieldID;   /* field IDs */ 
 
	struct _jmethodID;              /* opaque structure */ 
	typedef struct _jmethodID *jmethodID; /* method IDs */ 

4.

##Java代码和Native代码的链接

&emsp;使用JNI最终的目的是Java层调用C/C++的代码，那么怎么才能使他们链接好呢。首先看看下面两个内容：

1.JNI_OnLoad

