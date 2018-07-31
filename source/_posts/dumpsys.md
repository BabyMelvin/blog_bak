---
title: dumpsys
date: 2018-07-22 09:53:57
tags: Android调试
categories: Android
---

# 1.Dumpsys 源码

```cpp
//framework/native/cmds/dumpsys/dumpsys.cpp
int main(int argc,char* const argv[]){
    signal(SIGPIPE,SIG_IGN);
    //获取ServiceManager
    sp<IServiceManager> sm=defaultSerivceManager();
    fflush(stdout);
    if(sm==NULL){
        return 20;
    }
    Vector<String16> services;
    Vector<String16> args;
    bool showListOnly=true;
    //命令 dumpsys -l 
    if((argc==2)&&(strcmp(argv[1],"-l")==0)){
        showListOnly=true;
    }
    if((argc==1)||showListOnly){
        //不带参数命令 dumpsys
        services=sm->ListServices();
        services=sort(sort_func);
        args.add(String16("-a"))
    }else{
        //带参数则只能指定服务器的信息
        services.add(String16(argv[1]));
        for(int i=2;i<argc;i++){
            args.add(String16(argv[1]);
        }
    }
    const size_t N =service.size();
    if(N>1){
        //打印出第一行信息
        aout<<"Currently running services:"<<endl;
        for(size_t i =0;i<N;i++){
            //获取相应服务
            sp<IBinder> service=sm->checkSerivce(services[i]);
            if(service !=NULL){
              aout<<""<<service[i]<<endl;  
            }
        }
    }
    if(showListOnly){
        return 0;
    }
    for(size_t i =0;i<N;i++){
        sp<IBinder> service=sm->checkService(service[i]);
        if(service !=NULL){
            if(N>1){
                aout<<"------------------------------------------"<<endl;
                aout<<"Dump OF SERVICE "<<services[i]<<" : "<<endl;
            }
            //调用service相应的dump方法，这个是整个dumpsys命令精华
            int err=service->dump(STDOUT_FIFLENO,args);
            if(err!=0){
                aerr<<"Error dumping service info:"<<strerror(erro)<<" )"<<endl;
            }
        }else{
            aerr<<"Can't find service: "<<services[i]<<endl;
        }
    }
    return 0;
}
```

dumpsys主要工作分为4个步骤：

* `defaultSericeManager()`获取ServiceManager对象
* `sm->listServices()`获取系统所有向ServiceManager注册过的服务
* `sm->checkSerivce()`获取系统中年指定的Service
* `service->dump()`调用远程服务中`dump()`方法来获取相应的dump信息


# 2.使用
## 2.1 dumpsys命令用法
通过dumpsys命令查询系统服务运行状态：`dumpsys 服务名`

```
dumpsys activity //查询AMS服务相关信息
dumpsys window  //查询WMS服务相关信息
dumpsys cpuinfo //查询CPU情况
dumpsys meminfo //查询内存信息
```
可查询服务很多，查看当前支持的dump服务：

```
adb shell dumpsys -l
adb shell service list
```

## 2.2 系统服务

重要服务

|服务名|类名|功能|
|--|--|--|
|activity|ActivityManagerService|AMS相关信息|
|package|PackageManagerService|PMS相关信息|
|window|WindowManagerService|WMS相关信息|
|input|InputManagerService|IMS相关信息|
|power|PowerManagerService|PMS相关信息|
|batterstats|BatterystatsService|电池统计消息|
|battery|BatteryService|电池信息|
|alarm|AlarmManagerService|闹钟信息|
|dropbox|DropboxManagerService|调试相关|
|prostats|ProcessStatsService|进程统计|
|cpuinfo|CpuBinder|CPU|
|meminfo|MemBinder|内存|
|gxinfo|GraphicsBinder|图像|
|dbinfo|DbBinder|数据库|


其他服务

|服务名|功能|
|--|--|
|SurfaceFlinger|图像相关|
|appops|app使用情况|
|permission|权限|
|processinfo|进程服务|
|batteryproperties|电池相关|
|audio|查看声音相关|
|netstats|查看网络统计信息|
|diskstats|查看空间free状态|
|jobscheduler|查看任务计划|


## 2.3 Actitivty场景

**场景1**： 查看某个APP所有Service状态

```
dumpsys acitivty s com.sina.weibo
```

* Service类型为`com.morgoo.droidplugin.PluginManagerService`
* 运行在进程pid=7720，进程名为`com.sina.weibo`,uid=10094
* 通过bindeService连接该服务的进程Pid=7306，进程名为`com.sina.weibo:PluginP03`

当然还有`packageName`，`baseDir(apk路径)`，`dataDir(apk数据路径)`，`createTime`等各种信息。另外，新浪微博采用的是360开源的Android插件机制(`com.morgoo.droidplugin`)，主要用于hotfix等功能。

**场景2**：查询某个APP所有的广播状态

```
dumpsys activity b com.sina.weibo
```

* ` android.intent.action.SCREEN_ON`代表手机亮屏广播；
* 接收该广播的receiver有很多个，其中一个所在进程为pid=7220，进程名为`com.sina.weibo`

**场景3**:查询某个App所有的Activity状态

```
dumpsys acitivty a com.sina.weibo
```

* 格式：`TaskRecord{Hashcode #TaskId Affinity UserId=0 Activity个数=1}`；所以上图信息解析后就是TaskId=1802，Affinity=com.sina.weibo，当前Task中Activity个数为1。
* `effectiveUid`为当前task所属Uid，mCallingUid为调用者Uid=u0a94，mCallingPackage为调用者包名，这里是com.sina.weibo；
* `realActivity`:task中的已启动的Activity组件名com.sina.weibo/.SplashActivity

**场景4**：查询某个App的进程状态

```
dumpsys activity p com.sina.weibo
```
* 格式：`ProcessRecord{Hashcode pid:进程名/uid}`，进程pid=7306，进程名为com.sina.weibo:PluginP03，uid=10094.
* 该进程中还有Services，Connections, Providers, Receivers，可以看出该进程是没有Activity的进程。

```
dumpsys activity top
dumpsys acitivity oom //查看进程状态
```
