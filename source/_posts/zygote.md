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

```cpp
//AndroidRuntime.cpp
```