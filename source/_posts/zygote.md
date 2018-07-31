---
title: zygote
date: 2018-07-30 09:58:33
tags: Android进程
categories: Android
---

```
/frameworks/base/cmds/app_process/App_main.cpp
/frameworks/base/core/jni/AndroidRuntime.cpp
/frameworks/base/core/java/com/android/internal/os/
  - ZygoteInit.java
  - Zygote.java
  - ZygoteConnection.java
/frameworks/base/core/java/android/net/LocalServerSocket.java
/system/core/libutils/Threads.cpp
```
# 1,概述

Zygote是由init.rc解析创建的，zygote所对应可执行程序`app_process`，所对应的资源文件是`App_main.cpp`,进程名为zygote。

```
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
	class main
	socket zygote stream 660 root system
	onrestart write /sys/android_power/request_state wake
	onrestart write /sys/poweer/state on
	onrestart restart media
	onrestart restart netd
```

* `app_process`使用:`Usage: app_process [java-options] cmd-dir start-class-name [options]\n`.有两种`class name`或者`--zygote`.
	* 第一种，本文具体分析
	* 第二种，如`am`命令使用。

```
#!/system/bin/sh
base=/system
export CLASSPATH=$base/framework/am.jar
exec app_process $base/bin com.android.command.am.Am "$@"
```
Zygote能够重启的地方：

* servicemanager进程被杀(onrestart)
* surfaceflinger进程被杀(onrestart)
* zygote进程自己被杀(oneshot=fasle)
* system_server进程被杀(waitpid)

从App_main()开始,zygote启动过程函数调用大致：

```
App_main.main()-->AndroidRuntime.start()--->startVm()---->startReg()--->ZygoteInit.main()

--->registerZygoteSocket()--->preload()--->startSystemServer()--->runSelectLoop()
```

# 2.Zygote启动过程

## 2.1 App_main.main

```cpp
//App_main.cpp
int main(int argc,char*const argv[]){
	//参数为：--Xzygote /system/bin --zygote --start-system-server
	 AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    argc--; argv++; //忽略第一个参数
	int i;
	for(i=0;i<argc;i++){
		if(argv[i][0] !='-'){
			break;
		}
		if(argv[i][1]=='-' &&n argv[i][2] ==0){
			++i;
			break;
		}
		runtime.addOption(strdup(argv[i]));
	}
	//参数解析
	boolan zygote=false;
	bool startSystemServer=false;
	bool application=false;
	String8 niceName;
	String8 className;
	++i;
	while(i<argc){
		const char* arg=argv[i++];
		if(strcmp(arg,"--zygote")==0){
			zygote=true;
			//对于64系统nice_name为zygote64,32位为zygote
			niceName=ZYGOTE_NICE_NAME;
		}else if(strcmp(arg,"--start-system-server")==0){
			startSystemServer=true;
		}else if(strncmp(arg,"--nice-name","12")==0){
			niceName.setTo(arg+12);
		}else if(strncmp(arg,"--",2)!=0){
			className.setTo(arg);
			break;
		}else{
			--i;
			break;
		}
	}
	Vector<String8> args;
	if(!className.isEmpty(){
		//运行application或tool程序
		args.add(application?String8("application"):String8("tool"));
		runtime.setClassNameAndArgs(className,argc-i,argv+i);
	}else{
		//进入zygote模式，创建/data/dalvik-cache路径
		maybeCreateDalvikCache();
		if(startSystemServer){
			args.add(String8("start-system-server"));
		}
		char prop[PROP_VALUE_MAX];
        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
            return 11;
        }
        String8 abiFlag("--abi-list=");
        abiFlag.append(prop);
        args.add(abiFlag);

        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
	}
	//设置进程名
	if(!niceName.isEmpty()){
		runtime.setArgv0(niceName.string());]
		set_process_name(niceName.string());
	}
	if(zygote){
		runtime.start("com.android.internal.as.ZygoteInit",args,zygote);
	}else if(className){
		runtime.start("com.android.internal.os.RuntimeInit",args,zygote);
	}else{
		//没有定义类名或zygote
		retrun 10;
	}
}
```

## 2.2 start

启动Android runtime(JNI层，本质是cpp，调用java需要JNI接口实现)。包含启动一个虚拟机，并调用类名中的`main()`方法.进入java世界**启动虚拟机**
```cpp
//AndroidRuntime.cpp
void AndroidRuntime::start(const char*className,const Vector<String8>& options,bool zygote)
{
	static const String8 startSystemServer("start-system-server");
	for(size_t i=0;i<options.size();++i){	
		if(options[i]==startSystemServer){
			const int LOG_BOOT_PROGRESS_START=3000；
		}
	}
	//ANDROID_ROOT=/system
	const char* rootDir=getenv("ANDROID_ROOT");
	if(rootDir==NULL){
		rootDir="/system";
		if(!hasDir("/system")){
			return;
		}
		setenv("ANDROID_ROOT",rootDir,1);
	}
	JniInvocation jni_invocation;
	Jni_invocation.Init(NULL);
	JNIEnv *env;
	//虚拟机创建
	if(startVm(&mJavaVM,&env,zygote)!=0){
		return;
	}
	onVmCreated(env);
	//JNI方法注册
	if(startReg(env)<0){
		return;
	}
	jclass stringClass;
	jobjectArray strArray;
	jstring classNameStr;
	//等价strArray=new String[options.size()+1];
	stringClass=env->FindClass("java/lang/String");
	strArray=env->NewObjectArray(options.size()+1,stringClass,NULL);
	//等价strArray[0]="com.android.internal.os.ZygoteInit";
	classNameStr=env->NewStringUTF(className);
	env->SetObjectArrayElement(strArray,0,classNameStr);
	//等价strArray[1]="start-system-server"
	//strArray[2]="--abi-list=xxx"
	//其中xxx是体系架构
	for(size_t i=0;i<options.size();++i){
		jstring optionsStr=env->NewStringUTF(options.itemAt(i).string());
		env->SetObjectArrayElement(strArray,i+1,optionStr);
	}
	//将"com.android.internal.os.ZygoteInit"转换为"com/android/internal/os/ZygoteInit"
	//启动VM，该线程变为VM的主线程，直到VM退出返回
	char* slashClassName=toSlashClassName(className);
	jclass startClass=env->FindClass(slashClassName);
	if(startClass==NULL){
	}else{
		jmethodID startMeth=env->GetStaticMethodID(startClass,"main","([Ljava/lang/String;)V");
		//调用ZygoteInit.main()方法
		env->CallStaticVoidMethod(startClass,startMeth,strArray);
	}
	//释放相应对象空间，意味main不返回，直到主动退出？
	ALOGD("Shutting down VM\n");
	free(slashClassName);
	mJavaVM->deleteCurrentThread();
	mJavaVM->DestoryJavaVM();
}
```

## 2.3 startVM

创建Java虚拟机方法的主要关于虚拟机参数设置，下面调优过程中常用参数

```cpp
//AndroidRuntime.cpp
int AndroidRuntime::startVM(JavaVM** pJavaVM,JNIEnv **pEnv,bool zygote)
{
	//JNI检测功能，用于native层调用jni函数进程常规检测，比较如字符串格式是否符合要求
	bool checkJni=false;
	property_get("dalvik.vm.checkjni",propBuf,"");
	if(strcmp(propBuf,"true")==0){
		checkJni=true;
	}else if(strcmp(propBuf,"false")dalvik.vm.checkjni",propBuf)!=0){
		property_get("ro.kernel.android.checkjni",propBuf,"");
		if(propBuf[0]=="1"){
			checkJni=true;
		}
	}
	if(checkJni){
		addOption("-Xcheck:jni");
	}
	//虚拟机产生的trace文件，主要分析系统问题
	parseRuntimeOption("dalvik.vm.stack-trace-file",stackTraceFileBuf,"-Xstacktracefile:");
	
	//对于不同的软硬件环境，这些参数需要调整，优化，从而使系统达到最佳性能
	parseRuntimeOption("dalvik.vm.heapstartsize", heapstartsizeOptsBuf, "-Xms", "4m");
    parseRuntimeOption("dalvik.vm.heapsize", heapsizeOptsBuf, "-Xmx", "16m");
    parseRuntimeOption("dalvik.vm.heapgrowthlimit", heapgrowthlimitOptsBuf, "-XX:HeapGrowthLimit=");
    parseRuntimeOption("dalvik.vm.heapminfree", heapminfreeOptsBuf, "-XX:HeapMinFree=");
    parseRuntimeOption("dalvik.vm.heapmaxfree", heapmaxfreeOptsBuf, "-XX:HeapMaxFree=");
    parseRuntimeOption("dalvik.vm.heaptargetutilization",
                       heaptargetutilizationOptsBuf, "-XX:HeapTargetUtilization=");
    ...
	//preload-classes文件内容手WritePreloadedClassFile.java生成的
	//ZygoteInit类中会预加载工作将其classes提前加载到内存，提高性能
	if(!hasFile("/system/etc/preload-classes")){
		return -1;
	}
	//初始化虚拟机
	//JavaVM对每个进程，JNIEnv是对每个线程。启动完成，可以调用JNI。
	if(JNI_CreateJavaVM(pJavaVM,pEnv,&initArgs)<0){
		ALOGE("JNI_CreateJavaVM failed\n");
		return -1;
	}
}
```

* `dalvik.vm.checkjni`和`ro.kernel.android.checkjni`检查属性设置

## 2.4 startReg

增加自己的JNI接口。
```cpp
int AndroidRuntime::startReg(JNIEnv* env){
	//设置线程创建方法为javaCreateThreadEtc
	androidSetCreateThreadFunc((android_create_thread_fn)javaCreateThreadEtc);
	env->PushLocalFrame(200);
	//进程JNI方法的注册
	if(register_jni_procs(gRegJNI,NELEM(gRegJNI),env)<0){
		env->PopLocalFrame(NULL);
		return -1;
	}
	env->PopLocalFrame(NULL);
	return 0;
}

//Threads.cpp
//gCreateThreadFn-->javaCreateThreadEtc
void androidSetCreateThreadFunc(android_create_thread_fn func){
	gCreateThreadFn=func;
}

static int register_jni_procs(const RegJNIRec array[],size_t count,JNIEnv*env){
	for(size_t i=0;i<count;i++){
		if(array[i].mProc(env)<0){
			return -1;
		}
	}
	return 0;
}

static const RegJNIRec gRegJNI[]={
	REG_JNI(register_com_android_internal_os_RuntimeInit),
    REG_JNI(register_android_os_Binder)，
	//相当于,结构体数组
	//{register_com_android_internal_os_RuntimeInit},
	//{register_android_os_Binder}，
    ...
};

#define REG_JNI(name) {name}
struct RegJNIRec{
	int (*mProc)(JNIEnv*);
}
```

# 3.进入Java层

```
//ZygoteInit.java
public static void main(String argv[]){
	try{
		RuntimeInit.enableDdms();//开启DDMS功能
		SamplingProfilerIntegeration.start();
		boolean startSystemServer=false;
		String socketName="zygote";
		String abiList=null;
		for(int i=1;i<argv.lenght;i++){
			if("start-system-server".equals(argv[i])){
				startSystemServer=true;
			}else if(argv[i].startsWith(ABI_LIST_ARGS)){
				abiList=argv[i].substring(ABI_LIST_ARG.length());
			}else if(argv[i].startsWith(SOCKET_NAME_ARG)){
				socketName=argv[i].substring(SOCKET_NAME_ARG.length());
			}else{
				throw new RuntimeException("unknown command line argument:"+argv[i]);
			}
		}
		...
		registerZygoteSocket(socketName);
		//这里可考虑进行快速启动
		preload();//预加载类和资源
		SamplingProfilerIntergration.writeZygoteSnapshot();
		gcAndFinalize();//GC操作
		if(startSystemServer){
			startSystemServer(abiList,socketName);//启动system_server
		}
		runSelectLoop(abiList);
		closeServerSocket();
	} catch (MethodAndArgsCaller caller) {
        caller.run(); //启动system_server中会讲到。
    } catch (RuntimeException ex) {
        closeServerSocket();
        throw ex;
    }
}
```

## 3.1 registerZygoteSocket

```java
//ZygoteInit.java
private static void registerZygoteSocket(String socketName){
	if(sServerSocket == null){
		int fileDesc;
		final String fullSocketName=ANDROID_SOCKET_PREFIX+socketName;
		try{
			String env=System.getenv(fullSocketName);
			fileDesc=Integer.parseInt(env);
		}catch(){
		}
		try{
			FileDescriptor fd=new FileDescriptor();
			fd.setInt$(fileDesc);
			sServerSocket=new LocalServerSocket(fd);//创建Socket的本地服务端
		}catch(IOException ex){
		}
	}
}
```

## 3.2 preload

```java
//ZygoteInit.java
static void preload(){
	//预加载位于preloaded-classes文件中的类（framework/base）在framework.jar打包
	preloadClasses();
	//预加载资源，包含drawable和color资源
	preloadResource();
	//预加载OpenGL
	preloadOpenGL();
	//通过System.loadLibrary()方法,预加载android compile_rt,jnigraphics
	preloadSharedLibraries();
	//预加载 文本连接符资源
	prelaodTextResources();
	//仅用于zygote进程，用于内存共享进程
	WebViewFactory.prepareWebViewInZygote();
}
```

### 3.2.1 加载类

zygote进程初始化，加载和初始化通用classes，大部分类分配几百字节，有的类几百k
```java
private static void preloadClasses(){
	//单例模式
	final VMRuntime runtime=VMRuntime.getRuntime();
	InputStream is=ClassLoader.getSystemClassLoader().getResourceAsStream(PRELOADED_CLASSES);
	Log.i("Preloading classes...");
	long startTime=SystemClock.uptimeMillis();
	//运行静态初始化，放弃root权限
	 setEffectiveGroup(UNPRIVILEGED_GID);
     setEffectiveUser(UNPRIVILEGED_UID);
	//通过目标heap利用率(utilization)，明确GC，不会有其他影响
	float defaultUtiliziton=runtime.getTargetHeapUtilization();
	runtime.setTargetHeapUtilization(0.8f);
	//开始一个干净的面板（slate)
	System.gc();
	runtime.runFinalizationSync();
	Debug.startAllocCounting();
	try{
		BufferReader br=new BufferReader(new InputStreamReader(is),256);
		int count=0;
		String line;
		while((line=br.readLine())!=null){
			//跳过注释和空行
			line =line.trim();
			if(line.startsWith("#") || line.equal("")){
				continue;
			}
			try{
				if(false){
					Log.v(TAG,"preloading"+line+"...");
				}
				//这个加载类
				Class.forName(line);
				//PRELOAD_GC_THRESHOLD=50000；超过这个GC
				if(Debug.getGlobalAllocSize()>PRELOAD_GC_THRESHOLD){
					if(false){
						Log.v(TAG,"GC at"+Debug.getGlobalAllocSize());
					}
					System.gc();
					runtime.runFinalizationSync();
					Debug.resetGlobalAllocSize();
				}
				count++;
			}catch(){
			}
			Log.i(TAG, "...preloaded " + count + " classes in "
				+ (SystemClock.uptimeMillis()-startTime) + "ms.");
		}
	}catch{
	}finally{
		IoUtils.closeQuitely(is);
		// Restore default.
        runtime.setTargetHeapUtilization(defaultUtilization);

        // Fill in dex caches with classes, fields, and methods brought in by preloading.
        runtime.preloadDexCaches();

        Debug.stopAllocCounting();

        // Bring back root. We'll need it later.
        setEffectiveUser(ROOT_UID);
        setEffectiveGroup(ROOT_GID);
	}
}
```
### 3.2.2 加载资源

加载常用资源，能够进行多进程共享。通常是几k，大多数在20k-40k，有时候更大。

```java
private static void preloadResoures(){
	final VMRuntime runtime=VMRuntime.getRuntime();
	Debug.startAllocCounting();
	try{
		System.gc();
		runtime.runFinalizationSync();
		mResources=Resouces.getSystem();
	}catch(){
	}finally{
	}
}
```

执行Zygote进程的初始化,对于类加载，采用反射机制`Class.forName()`方法来加载。对于资源加载，主要是 `com.android.internal.R.array.preloaded_drawables`和`com.android.internal.R.array.preloaded_color_state_lists`，在应用程序中以`com.android.internal.R.xxx`开头的资源，便是此时由`Zygote`加载到内存的。

zygote进程内加载了`preload()`方法中的所有资源，当需要fork新进程时，采用`copy on write`技术，如下：

![zygote](zygote/zygote_fork.jpg)

## 3.3 startSystemServer
```java
//ZygoteInit.java
private static void startSystemServer(String abiList, String socketName)throw MethodAndArgsCaller,RuntimeException{
	long capabilities=posixCapabilitiesAsBits(
		OsConstants.CAP_BLOCK_SUSPEND,
		OsConstants.CAP_KILL,
		OsConstants.CAP_NET_ADMIN,
		OsConstants.CAP_NET_BIND_SERVICE,
		OsConstants.CAP_NET_BROADCAST,
		OsConstants.CAP_NET_RAW,
		OsConstants.CAP_SYS_MODULE,
		OsConstants.CAP_SYS_NICE,
		OsConstants.CAP_SYS_RESOURCE,
		OsConstants.CAP_SYS_TIME,
		OsConstants.CAP_SYS_TTY_CONFIG
	);
	//参数准备
	String args[]={
		"--setuid=1000",
		"--setgid=1000",
		"--setgroups=1001,1002,1003,1004,1005,1006,10071008,1009,1010,1018,1021,1032,3001,3002,3003,3006,3007",
		 "--capabilities=" + capabilities + "," + capabilities,
        "--nice-name=system_server",
        "--runtime-args",
        "com.android.server.SystemServer",
	};
	ZygoteConnection.Arguments parsedArgs=null;
	int pid;
	try{
		//用于解析参数，生成目标格式
		parseArgs=new ZygoteConnection.Arguments(args);
		ZygoteConnection.applyDebuggerSystemProperty(parseArgs);
		ZygoteConneciton.applyInvokeWithSystemProperty(parseArgs);
		//fork 子进程,用于运行system_server
		pid=Zygote.forkSystemServer(
			parseArgs.uid,parsedArgs.gid,
			parseArgs.gids,
			parseArgs.debugFlags,
			null,
			parseArgs.permittedCapabilities,
			parseArgs.effectiveCapabilities
		);
	}catch(IllegalArgumentException ex) {
        throw new RuntimeException(ex);
	}
	//进入子进程system_server
	if(pid==0){
		if(hasSecondZygote(abiList)){
			waitForSecondaryZygote(socketName);
		}
		//完成system_server进程剩余的工作
		handleSystemServerProcess(parsedArgs);
	}
	return true;
}
```

## 3.4 runSelectLoop

```java
ZygoteInit.java

```

在app进程中分析。

# 4.小结

* 1.解析`init.zygote.rc`中的参数，创建`AppRuntime`并调用`AppRuntime.start()`方法；
* 2.调用`AndroidRuntime`的`startVM()`方法创建虚拟机，再调用`startReg()`注册JNI函数；
* 3.通过JNI方式调用`ZygoteInit.main()`，第一次进入`Java世界`；
* 4.`registerZygoteSocket()`建立socket通道，zygote作为通信的服务端，用于响应客户端请求；
* 5.`preload()`预加载通用类、`drawable`和`color`资源、`openGL`以及共享库以及WebView，用于提高app启动效率；
* 6.`zygote`完毕大部分工作，接下来再通过`startSystemServer()`，fork得力帮手`system_server`进程，也是上层framework的运行载体。
* 7.`zygote`功成身退，调用`runSelectLoop()`，随时待命，当接收到请求创建新进程请求时立即唤醒并执行相应工作。