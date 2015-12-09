&emsp;最近看到了很多关于热补的开源项目——Depoxed(阿里)、AnFix（阿里）、DynamicAPK(携程)等，它们都用到了JNI编程，并且JNI编程也贯穿了Android系统，学会JNI编程对于我们学习研究Android源码、Android安全以及Android安全加固等都是有所帮助的。但是对于我们这些写Android应用的，大部分时间都是在使用Java编程，很少使用C/C++编程，对于JNI编程也了解的比较少，那么我们就来简单的了解一下JNI编程的基础吧。

##什么是JNI，怎么使用
&emsp;JNI——Java Native Interface，它是Java平台的一个特性(并不是Android系统特有的)。其实主要是定义了一些JNI函数，让开发者可以通过调用这些函数实现Java代码调用C/C++的代码，C/C++的代码也可以调用Java的代码，这样就可以发挥各个语言的特点了。那么怎么使用JNI呢，一般情况下我们首先是将写好的C/C++代码编译成对应平台的动态库(windows一般是dll文件，linux一般是so文件等)，这里我们是针对Android平台，所以只讨论so库。由于JNI编程支持C和C++编程，这里我们的栗子都是使用C++，对于C的版本可能会有些差异，但是主要的内容还是一致的，大家可以触类旁通。

##从一个栗子说起

&emsp;这里还是直接从代码说起，这样更加形象和直观，也便于理解。今天使用的Java代码如下：

	public class AndroidJni {

    	static{
        	System.loadLibrary("main");
    	}

    	public native void dynamicLog();

    	public native void staticLog();

	}

这里我们定义了两个声明为native的方法，并声明了一块静态区域，在该静态区域类加载名为libmain.so的库，这里我们说是libmain.so库，但是加载的时候却只写了“main”，其实大家只要知道这是约定的就可以了。	

C++代码如下：

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

这里引用了两个头文件，jni.h和mylog.h，其中jni.h是定义了很多我们使用的JNI函数和结构体，mylog.h是我自己定义的打印Android log的函数(功能和Java的Log类相同)。
这里暂时不讨论怎么编译成so库以及so库的一些规范，会在下一篇文章中介绍。这里假定将上面的C++程序编译成了一个叫libmain.so的文件。在Java层使用System.loadLibarary("main")方法将该so库加载起来，使得dynamicLog()、staticLog()和对应的Java_com_github_songnick_jni_AndroidJni_staticLog()、nativeDynamicLog()两个native方法链接起来，当然这部分工作都是由Java虚拟机完成的，那么具体是怎么完成的，接下来将根据上面的代码进行分析。

###静态注册native方法

&emsp;在上面的代码中看到了JNIEXPORT和JNICALL关键字，这两个关键字是两个宏定义，他主要的作用就是说明该函数为JNI函数，在Java虚拟机加载的时候会链接对应的native方法，在AndroidJni.java的类中声明了staticLog()为native方法，他对应的JNI函数就是Java_com_github_songnick_jni_AndroidJni_staticLog(),那么是怎么链接的呢，在Java虚拟机加载so库时，如果发现含有上面两个宏定义的函数时就会链接到对应Java层的native方法，那么怎么知道对应Java中的哪个类的哪个native方法呢，我们仔细观察JNI函数名的构成其实是：Java_PkgName_ClassName_NativeMethodName，以Java为前缀，并且用“_”下划线将包名、类名以及native方法名连接起来就是对应的JNI函数了。一般情况下我们可以自己手动的去按照这个规则写，但是如果native方法特别多，那么还是有一定的工作量，并且在写的过程中不小心就有可能写错，其实Java给我们提供了javah的工具帮助生成相应的头文件。在生成的头文件中就是按照上面说的规则生成了对应的JNI函数，我们在开发的时候直接copy过去就可以了。这里上面的代码为例，在AndroidStudio中编译后，进入项目的目录app/build/intermediates/classes/debug下，运行如下命令：

	javah -d jni com.github.songnick.jni.AndroidJni

这里-d指定生成.h文件存放的目录(如果没有就会自动创建)，com.github.songnick.jni.AndroidJni表示指定目录下的class文件。这里简单介绍一下生成的JNI函数包含两个固定的参数变量，分别是JNIEnv和jobject，其中JNIEnv后面会介绍，jobject就是当前与之链接的native方法隶属的类对象(类似于Java中的this)。这两个变量都是Java虚拟机生成并在调用时传递进来的。

###动态注册

&emsp;上面我们介绍了静态注册native方法的过程，就是Java层声明的native方法和JNI函数是一一对应的，那么有没有方法让Java层的native方法和任意的JNI函数链接起来，当然是可以的，这就得使用动态注册的方法。接下来就看看如何实现动态注册的。

####JNI_OnLoad函数

&emsp;当我们使用System.loadLibarary()方法加载so库的时候，Java虚拟机就会找到这个函数并调用该函数，因此可以在该函数中做一些初始化的动作，其实这个函数就是相当于Activity中的onCreate()方法。该函数前面有三个关键字，分别是JNIEXPORT、JNICALL和jint，其中JNIEXPORT和JNICALL是两个宏定义，用于指定该函数是JNI函数。jint是JNI定义的数据类型，因为Java层和C/C++的数据类型或者对象不能直接相互的引用或者使用，JNI层定义了自己的数据类型，用于衔接Java层和JNI层，至于这些数据类型我们在后面介绍。这里的jint对应Java的int数据类型，该函数返回的int表示当前使用的JNI的版本，其实类似于Android系统的API版本一样，不同的JNI版本中定义的一些不同的JNI函数。该函数会有两个参数，其中*jvm为Java虚拟机实例，JavaVM结构体定义了以下函数：

	DestroyJavaVM
   	AttachCurrentThread
   	DetachCurrentThread
   	GetEnv

这里我们使用了GetEnv函数获取JNIEnv变量，上面的JNI_OnLoad函数中有如下代码：

	JNIEnv *env;
	if (jvm->GetEnv((void**) &env, JNI_VERSION_1_4) != JNI_OK) {

		return -1;
	}

这里调用了GetEnv函数获取JNIEnv结构体指针，其实JNIEnv结构体是指向一个函数表的，该函数表指向了对应的JNI函数，我们通过调用这些JNI函数实现JNI编程，在后面我们还会对其进行介绍。

####获取Java对象，完成动态注册

上面介绍了如何获取JNIEnv结构体指针，得到这个结构体指针后我们就可以调用JNIEnv中的RegisterNatives函数完成动态注册native方法了。该方法如下：

	jint RegisterNatives(jclass clazz, const JNINativeMethod* methods, jint nMethods)

第一个参数是Java层对应包含native方法的对象(这里就是AndroidJni对象)，通过调用JNIEnv对应的函数获取class对象(FindClass函数的参数为需要获取class对象的类描述符)：

	jclass clz = env->FindClass("com/github/songnick/jni/AndroidJni");

第二个参数是JNINativeMethod结构体指针，这里的JNINativeMethod结构体是描述Java层native方法的，它的定义如下：

	typedef struct {
    	const char* name;//Java层native方法的名字
    	const char* signature;//Java层native方法的描述符
    	void*       fnPtr;//对应JNI函数的指针
	} JNINativeMethod;

第三个参数为注册native方法的数量。一般会动态注册多个native方法，首先会定义一个JNINativeMethod数组，然后将该数组指针作为RegisterNative函数的参数传入，所以这里定义了如下的JNINativeMethod数组：

	JNINativeMethod nativeMethod[] = {{"dynamicLog", "()V", (void*)nativeDynamicLog}};


最后调用RegisterNative函数完成动态注册：

	env->RegisterNatives(clz, nativeMethod, sizeof(nativeMethod)/sizeof(nativeMethod[0]));


##JNIEnv结构体

&emsp;上面提到JNIEnv这个结构体，它就老厉害了，指向一个函数表，该函数表指向一系列的JNI函数，我们通过调用这些JNI函数可以实现与Java层的交互，这里简单的看看几个定义的函数：

	..........
	jfieldID GetFieldID(jclass clazz, const char* name, const char* sig)
	jboolean GetBooleanField(jobject obj, jfieldID fieldID)
	jmethodID GetMethodID(jclass clazz, const char* name, const char* sig)
	CallVoidMethod(jobject obj, jmethodID methodID, ...)
	CallBooleanMethod(jobject obj, jmethodID methodID, ...)
	..........

这里简单的看看上面的四个函数，GetFieldID()函数是获取Java对象中某个变量的ID，GetBooleanField()函数是根据变量的ID获取数据类型为Boolean的变量。GetMethodID()函数是获取Java对象中对应方法的ID，CallVoidMethod()根据methodID调用对应对象中的方法，并且该方法的返回值为Void类型。通过这些函数我们可以实现调用Java层的代码。更多的函数大家还是看看API文档吧！

##JNI数据类型
上面我们提到JNI定义了一些自己的数据类型。这些数据类型是衔接Java层和C/C++层的，如果有一个对象传递下来，那么对于C/C++来说是没办法识别这个对象的，同样的如果C/C++的指针对于Java层来说它也是没办法识别的，那么就需要JNI进行匹配，所以需要定义一些自己的数据类型。

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

&emsp;前面我们在获取AndroidJni对象的使用通过定义jclass引用，然后调用FindClass函数获取了该对象，所以JNI也定义了一些引用类型以便JNI层调用，具体的引用类型如下：
	
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
&emsp;当需要调用Java中的某个方法的时候我们首先要获取它的ID，根据ID调用JNI函数获取该方法，变量的获取过程也是同样的过程，这些ID的结构体定义如下：

	struct _jfieldID;              /* opaque structure */ 
	typedef struct _jfieldID *jfieldID;   /* field IDs */ 
 
	struct _jmethodID;              /* opaque structure */ 
	typedef struct _jmethodID *jmethodID; /* method IDs */ 

##描述符

1.类描述符
&emsp;前面为了获取Java的AndroidJni对象，是通过调用FindClass()函数获取的，该函数参数只有一个字符串参数，我们发现该字符串如下所示：

	com/github/songnick/jni/AndroidJni

其实这个就是JNI定义了对类的描述符，它的规则就是将"com.github.songnick.jni.AndroidJni"中的“.”用“/”代替。
	

2.方法描述符
&emsp;前面我们动态注册native方法的时候结构体JNINativeMethod中含有方法描述符，就是确定native方法的参数和返回值，我们这里定义的dynamicLog()方法没有参数，返回值为空所以对应的描述符为："()V"，括号类为参数，V表示返回值为空。下面还是看看几个栗子吧：

Method Descriptor           |   Java Language Type   
----------------------------|-------------------------  
	"()Ljava/lang/String;"  |    String f();         
	"(ILjava/lang/Class;)J" | long f(int i, Class c);
	  "([B)V"               |  String(byte[] bytes); 

上面的栗子我们看到方法的返回类型和方法参数有引用类型以及boolean、int等基本数据类型，对于这些类型的描述符在下个部分介绍。这里数组的描述符以"["和对应的类型描述符来表述。对于二维数组以及三维数组则以"[["和"[[["表示：
	
 Descriptor  | Java Langauage Type 
-------------|--------------------
  "[[I"   	 |  		int[][]	   
 "[[[D"      |  double[][][]	   


3.数据类型描述符
&emsp;前面我们说了方法的描述符，那么针对boolean、int等数据类型描述符是怎样的呢，JNI对基本数据类型的描述符定义如下：

Field Desciptor  | Java Language Type 
-----------------|-------------------
     Z           |     boolean        
     B           |     byte           
     C           |     char           
     S           |     short          
     I           |     int            
     J           |     long           
     F           |     floa           
     D           |     double         

对于引用类型描述符是以"L"开头";"结尾，示例如下所示：

  Field Desciptor     | Java Language Type 
----------------------|--------------------
"Ljava/lang/String;"  |     String         
"[Ljava/lang/Object;" |     Object[]     





##总结

&emsp;上面的部分我们通过一个栗子简单的对JNI编程进行了分析，这里只是对简单的进行了介绍，只是JNI编程的一部分，我相信任何一门技术或者技术点都不能通过一篇文章达到精通，更多的还是靠实践，只有在实践的过程中发现问题－解决问题，才能对知识更好的理解和认识，从而达到精通。所以希望通过这边文章你可以对JNI编程有一个初步的认识，不会感觉JNI编程很难。大家可以多看看JNI的[API文档](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/)，这样会对JNI有更多的了解。这里还要说一下对于Android的JNI编程还是有点区别的，大家可以多看看Google官方文档对于JNI编程的一些指导和Demo程序。下一篇文章将介绍Android NDK相关的内容，将JNI编程运用到Android开发中。

希望在Android学习的路上，大家共同成长！


