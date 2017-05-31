&emsp;我们在排查内存的时候，往往会通过以下命令：

		dumpsys meminfo package_name

获取对应应用的详细内存信息，一般我们会获取到如下信息(对于不同的Android版本以及不同厂商的ROM，以下信息会有一些差异)：

![android_mem](http://upload-images.jianshu.io/upload_images/1098335-95a37c2c15b02a81.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
对于这些参数刚开始看到的时候比较晕，其中很多的内容不是非常的了解，也不了解这些数据来自于哪里，现在我希望通过源码分析的方式，找到这些数据来自于哪里，然后再分析它们的具体含义。

&emsp;对于 [dumpsys](http://androidxref.com/4.2.2_r1/xref/frameworks/native/cmds/dumpsys/dumpsys.cpp) 这个命令我们可以在[androidxref.com](androidxref.com)中很容易的搜到它的源码，部分代码如下所示：

		Vector<String16> services;
	    Vector<String16> args;
	    //默认argc是等于1
	    if (argc == 1) {
	        services = sm->listServices();
	        services.sort(sort_func);
	        args.add(String16("-a"));
	    } else {
	    	//如果我们输入dumpsys meminfo 就会走到这里；
	        services.add(String16(argv[1]));
	        for (int i=2; i<argc; i++) {
	            args.add(String16(argv[i]));
	        }
	    }
	    ......
	    sp<IBinder> service = sm->checkService(services[i]);

首先根据main函数中输入的参数个数以及参数字符串数组，获取到我们需要调用的service的名字，并添加到services中，最后通过调用sm(ServiceManager)获取需要调用的service，最后调用：

		int err = service->dump(STDOUT_FILENO, args);

这里就调用到了service的dump方法了，其中args存放的是dumpsys后面的参数；对于这个sm(ServiceManager)对象是Android系统中管理Service的类(其实它自己本身也是一个Service)，对于一般的Service都是通过调用Java层的ServiceManager类操作，我们通过搜索调用

	ServiceManager.addService(String name, IBinder service);

方法的位置，可以发现在ActivityManagerService中的[setSystemProcess()](http://androidxref.com/4.2.2_r1/xref/frameworks/base/services/java/com/android/server/am/ActivityManagerService.java#1363)存在如下代码：

		ActivityManagerService m = mSelf;

		ServiceManager.addService("activity", m, true);
		ServiceManager.addService("meminfo", new MemBinder(m));
		ServiceManager.addService("gfxinfo", new GraphicsBinder(m));
		ServiceManager.addService("dbinfo", new DbBinder(m));

这里可以很清晰的看到调用ServiceManager的addServcie()方法，将ActivityManagerService、MemBinder等添加到了ServiceManager中，找到这里我们终于发现了我们的meminfo对于的Service——MemBinder。这里的MemBinder其实是继承Binder，并重写了dump()方法：

          
	protected void dump(FileDescriptor fd, PrintWriter pw, String[] args) {
        if (mActivityManagerService.checkCallingPermission(android.Manifest.permission.DUMP)
                != PackageManager.PERMISSION_GRANTED) {
            pw.println("Permission Denial: can't dump meminfo from from pid="
                    + Binder.getCallingPid() + ", uid=" + Binder.getCallingUid()
                    + " without permission " + android.Manifest.permission.DUMP);
            return;
        }

        mActivityManagerService.dumpApplicationMemoryUsage(fd, pw, "  ", args,
                false, null, null, null);
    }

通过上面的代码发现，最终调用的方法是mActivityManagerService中的dumpApplicationMemoryUsage()方法，该方法在最开始的时候对输入的参数进行了解析：

		int opti = 0;
        while (opti < args.length) {
            String opt = args[opti];
            if (opt == null || opt.length() <= 0 || opt.charAt(0) != '-') {
                break;
            }
            opti++;
            if ("-a".equals(opt)) {
                dumpAll = true;
            } else if ("--oom".equals(opt)) {
                oomOnly = true;
            } else if ("-h".equals(opt)) {
                pw.println("meminfo dump options: [-a] [--oom] [process]");
                pw.println("  -a: include all available information for each process.");
                pw.println("  --oom: only show processes organized by oom adj.");
                pw.println("If [process] is specified it can be the name or ");
                pw.println("pid of a specific process to dump.");
                return;
            } else {
                pw.println("Unknown argument: " + opt + "; use -h for help");
            }
        }

这里主要是解析命令行中带有的参数，根据参数信息确定要现实的数据内容以及帮助。当解析完参数的信息的后，就会根据解析的结果显示对应的信息

		//获取所有的进程信息，这里主要是通过ProcessRecord信息获取
		ArrayList<ProcessRecord> procs = collectProcesses(pw, opti, args);
        if (procs == null) {
            return;
        }
        //检查参数中是否存在“--checkin”参数，一般情况下并不加入，可以加入这个参数看看结果；
        //对于这个参数在ActivityThread里面的解释：For checkin, we print one long comma-separated list of values
        //就是已逗号的分隔将所有的数据打印出来，这里的数据就是内存
        final boolean isCheckinRequest = scanArgs(args, "--checkin");
        long uptime = SystemClock.uptimeMillis();
        long realtime = SystemClock.elapsedRealtime();
        //如果进程数只有一个或者包含'--checkin'
        if (procs.size() == 1 || isCheckinRequest) {
            dumpAll = true;
        }
        if (isCheckinRequest) {
            // short checkin version
            pw.println(uptime + "," + realtime);
            pw.flush();
        } else {
        	//这里就是我们通过dumpsys meminfo package_name 命令显示的第一行文字
            pw.println("Applications Memory Usage (kB):");
            pw.println("Uptime: " + uptime + " Realtime: " + realtime);
        }

当获取到所有的进程信息后，就是轮询的检查需要显示的内存信息，最终调用如下的代码：

		if (!isCheckinRequest && dumpAll) {
			//内存显示详细信息的第二行
	        pw.println("\n** MEMINFO in pid " + r.pid + " [" + r.processName + "] **");
	        pw.flush();
	    }
	    Debug.MemoryInfo mi = null;
	    if (dumpAll) {
	        try {
	            mi = r.thread.dumpMemInfo(fd, isCheckinRequest, dumpAll, innerArgs);
	        } catch (RemoteException e) {
	            if (!isCheckinRequest) {
	                pw.println("Got RemoteException!");
	                pw.flush();
	            }
	        }
	    } else {
	        mi = new Debug.MemoryInfo();
	        Debug.getMemoryInfo(r.pid, mi);
	    }

在上面的内存获取的过程中，一般dumpAll为true，所以会调用Process.thread参数，这里的thread为IApplcationThread对象，对于熟悉Activity的过程对于这个接口并不陌生(如果不是很清楚可以自行补补哈)，这个最终存储的是ActivityThread中的ApplicationThread对象(这里如果不是很理解可以参考一下Android中[App启动过程的分析](http://blog.csdn.net/luoshengyang/article/details/6689748))，这里直接调用了该对象的dumpMeminfo()方法(ActivityThread中的ApplicationThread)：

		long nativeMax = Debug.getNativeHeapSize() / 1024;
	    long nativeAllocated = Debug.getNativeHeapAllocatedSize() / 1024;
	    long nativeFree = Debug.getNativeHeapFreeSize() / 1024;
	    Debug.MemoryInfo memInfo = new Debug.MemoryInfo();
	    Debug.getMemoryInfo(memInfo);
	    if (!all) {
	        return memInfo;
	    }
	    Runtime runtime = Runtime.getRuntime();
	    long dalvikMax = runtime.totalMemory() / 1024;
	    long dalvikFree = runtime.freeMemory() / 1024;
	    long dalvikAllocated = dalvikMax - dalvikFree;
	    ...........

这部分的内容主要是获取java heap和native heap相关的内存信息，通过这部分内容，我们可以看到native heap信息的获取是通过Debug类中的静态方法获取的，对于java heap的内存信息是通过Runtime类中的静态方法获取到的，对于java heap的内存信息的获取还是比较简单的。对于native heap可能就比较的复杂，Debug中的调用的静态方法为native方法，这部分的JNI实现在[android_os_Debug.cpp](http://androidxref.com/4.2.2_r1/xref/frameworks/base/core/jni/android_os_Debug.cpp)中，这里需要看看getMemoryInfo()方法的实现：

	//JNI方法注册过程中java层方法与native方法的对应关系，关于这部分不是很懂的，可以参考
	//获取native heap相关的信息
	{ "getNativeHeapSize","()J",(void*) android_os_Debug_getNativeHeapSize },
    { "getNativeHeapAllocatedSize", "()J",(void*) android_os_Debug_getNativeHeapAllocatedSize },
    { "getNativeHeapFreeSize",  "()J",(void*) android_os_Debug_getNativeHeapFreeSize }

	//getMemoryInfo对应android_os_Debug_getDirtyPages()方法
	{ "getMemoryInfo", "(Landroid/os/Debug$MemoryInfo;)V",(void*) android_os_Debug_getDirtyPages }

上面的jni注册方法数组可以获得，getNativeHeapSize()方法，最终调用的是android_os_Debug_getNativeHeapSize()方法，该方法实现如下：

		static jlong android_os_Debug_getNativeHeapSize(JNIEnv *env, jobject clazz)
		{
		#ifdef HAVE_MALLOC_H
			//该函数是获取内存分配的信息，其中usmblks表示总的分配内存大小
		    struct mallinfo info = mallinfo();
		    return (jlong) info.usmblks;
		#else
		    return -1;
		#endif
		}

其他的获取native的内存类似，对于getMemoryInfo()的方法对应的，实现方法如下：

		static void android_os_Debug_getDirtyPages(JNIEnv *env, jobject clazz, jobject object)
		{
		    android_os_Debug_getDirtyPagesPid(env, clazz, getpid(), object);
		}
		static void android_os_Debug_getDirtyPagesPid(JNIEnv *env, jobject clazz,
		        jint pid, jobject object)
		{
		    stats_t stats[_NUM_HEAP];
		    memset(&stats, 0, sizeof(stats));
		    load_maps(pid, stats);
		    .....
		}
通过上面的调用关系可以看到最终调用的是load_maps()方法获取内存相关的信息，并存储在stats数组中：

		static void load_maps(int pid, stats_t* stats)
		{
		    char tmp[128];
		    FILE *fp;
		    sprintf(tmp, "/proc/%d/smaps", pid);
		    fp = fopen(tmp, "r");
		    if (fp == 0) return;
		    read_mapinfo(fp, stats);
		    fclose(fp);
		}

这里从调用的方法可以直观的看到，获取这部分的内存信息是读取系统目录下proc文件夹下面对应进程中的smaps文件，然后将这个文件的信息读取到stats数组中，read_mapinfo()方法的主要功能如下：
		
		.....
		while (!done) {
		    prevHeap = whichHeap;
		    prevEnd = end;
		    whichHeap = HEAP_UNKNOWN;
		    skip = false;
		    len = strlen(line);
		    if (len < 1) return;
		    line[--len] = 0;
		    //这段代码的功能主要是确定读取内存信息的类型：native、dalvik等
		    if (sscanf(line, "%lx-%lx %*s %*x %*x:%*x %*d%n", &start, &end, &name_pos) != 2) {
		        skip = true;
		    } else {
		        while (isspace(line[name_pos])) {
		            name_pos += 1;
		        }
		        name = line + name_pos;
		        nameLen = strlen(name);

		        if (strstr(name, "[heap]") == name) {
		            whichHeap = HEAP_NATIVE;
		        } else if (strstr(name, "/dev/ashmem/dalvik-") == name) {
		            whichHeap = HEAP_DALVIK;
		        } else if (strstr(name, "/dev/ashmem/CursorWindow") == name) {
		            whichHeap = HEAP_CURSOR;
		        }.....
		    }
		    //这段代码就是确定读取内存类型的大小：size、Rss、Pss等
		    while (true) {
		        if (fgets(line, 1024, fp) == 0) {
		            done = true;
		            break;
		        }
		        if (sscanf(line, "Size: %d kB", &temp) == 1) {
		            size = temp;
		        } else if (sscanf(line, "Rss: %d kB", &temp) == 1) {
		            resident = temp;
		        } .......
		    }
		    //将获取的内存类型存储到status数组中
		    if (!skip) {
		        stats[whichHeap].pss += pss;
		        stats[whichHeap].privateDirty += private_dirty;
		        stats[whichHeap].sharedDirty += shared_dirty;
		    }
		}

上面的主要功能就是读取native、dalvik以及so等类型关于pss、private_dirty、shared_dirty的内存信息；最终存储在stats数组中。

&esp;通过上面的分析，获取到了native heap 以及 native、dalvik、so等的pss、priveDirtyPss、sharedDirtyPss等相关信息，我们回到ActivityThread中的ApplicationThread的dumpMemInfo()方法，在获得这些信息以后就会打印这些信息：

		// otherwise, show human-readable format，这里是打印内存信息的头部信息
		printRow(pw, HEAP_COLUMN, "", "", "Shared", "Private", "Heap", "Heap", "Heap");
		printRow(pw, HEAP_COLUMN, "", "Pss", "Dirty", "Dirty", "Size", "Alloc", "Free");
		printRow(pw, HEAP_COLUMN, "", "------", "------", "------", "------", "------",
		        "------");
		 //打印nNative 和 Dalvik相关的数据
		printRow(pw, HEAP_COLUMN, "Native", memInfo.nativePss, memInfo.nativeSharedDirty,
		        memInfo.nativePrivateDirty, nativeMax, nativeAllocated, nativeFree);
		printRow(pw, HEAP_COLUMN, "Dalvik", memInfo.dalvikPss, memInfo.dalvikSharedDirty,
		        memInfo.dalvikPrivateDirty, dalvikMax, dalvikAllocated, dalvikFree);

		int otherPss = memInfo.otherPss;
		int otherSharedDirty = memInfo.otherSharedDirty;
		int otherPrivateDirty = memInfo.otherPrivateDirty;
		//这里主要打印:
		//.so mmap
		//.apk mmap等相关数据，可以参考Debug.MemoryInfo.getOtherLabel(i)方法，主要是获取这些字段
		for (int i=0; i<Debug.MemoryInfo.NUM_OTHER_STATS; i++) {
		    printRow(pw, HEAP_COLUMN, Debug.MemoryInfo.getOtherLabel(i),
		            memInfo.getOtherPss(i), memInfo.getOtherSharedDirty(i),
		            memInfo.getOtherPrivateDirty(i), "", "", "");
		    otherPss -= memInfo.getOtherPss(i);
		    otherSharedDirty -= memInfo.getOtherSharedDirty(i);
		    otherPrivateDirty -= memInfo.getOtherPrivateDirty(i);
		}.....
		//后面这里主要是打印activity、view等相关信息，大家可以自行查看


&emsp;通过上面源码的分析大概了解了获取内存信息的流程，这里整体总结一下：

1.方法的调用：

			dumpsys.cpp:service.dump()

			ActivityManagerService:	MeminfoBinder.dump()

			ActivityManagerService: dumpApplicationMemoryUsage()

			ActivityThread:	ApplicationThread->dumpMemInfo() 


2.内存的获取：

		java heap：
				Dalvik Pss:	Debug.MemoryInfo.dalvikPss
				Dalvik SharedPrivateDirty:	Debug.MemoryInfo.dalvikSharedDirty
				......
				Dalvik Size: Runtime.totalMemory()
				Dalvik Free: Runtime.freeMemory()
				Dalvik Alloc: Runtime.totalMemory() - Runtime.freeMemory()

		native heap:
				//读取MemoryInfo的值
				Native Pss: Debug.MemoryInfo.nativePss
				Native SharedPrivateDirty:MemoryInfo.nativeSharedDirty
				.....
				//调用native方法
				Native Size：Debug.getNativeHeapSize()
				Native Alloc: Debug.getNativeHeapAllocatedSize()
				Native Free: Debug.getNativeHeapFreeSize()

这里Debug.MemoryInfo相关的信息主要通过读取/proc/PID/smaps文件中的信息，对于native size、alloc以及free主要是通过调用[android_os_Debug.cpp](http://androidxref.com/4.2.2_r1/xref/frameworks/base/core/jni/android_os_Debug.cpp)中的方法，最后调用
[mallinfo()](http://man7.org/linux/man-pages/man3/mallinfo.3.html)方法获取，java层的size、alloc、以及free主要是根据Runtime来获取；

&emsp;以上所有的分析都是基于4.4.2的源码进行分析的，这里我只是简单的分析了一下命令行是怎么获取到这些信息的，但是并没有详细的分析内存的来源以及原理，这部分是可以继续深究的！

