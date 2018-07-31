---
title: bugreport
date: 2018-07-25 23:43:26
tags: Android调试
categories: Android
---

# 1.概述

bugreport中路径为`framework/native/cmds`,生成可执行文件`/system/bin/bugreport`.bugreport信息量非常大，几乎涵盖系统各个层面内容。

* 通过启动`property_set("ctl.start","dumpstate");`启动dumpstate服务(由init进程fork出来).
* bugreport进程通过socket套接字与dumpstate进行通讯
* bugreport作为客户端读取`length=read(s,buffer,sizeofo(buffer))`,并将读到的内容进行输出。`fwrite(buffer,1,length,stdout)`

# 2.`dumpstate`

## 2.1 处理SIGPIPE信号

dumpstate为socket服务端，如果一个客服端关闭调用两次write，第二次会产生SIGPIPE信号，该信号默认结束进程。

```c
memset(&sigact,0,sizeof(sigact));
//句柄来处理，SIG_IGNORE则丢弃不处理该信号
sigact.sa_hanlder=sigpipe_handler;
sigaction(SIGPIPE,&sigact,NULL);
```

## 2.2 设置进程优先级

将dumpstate设置成高优先级，避免被OOM杀掉。

```c
//1.linux设置优先级
setpriority(PRIO_PROCESS,0,-20);
//2.self表示当前进程(链接),android oom概念
FILE *oom_adj=fopen("/proc/self/oom_adj","w");
if(oom_adj){
    fputs("-17",oom_adj);
    fclose(oom_adj);
}
```

其中`setpriority(int which,int who,int pri)`,表示设置程序调度优先级.可用`getpriority`获取：

* which:取值可能为`PRIO_PROCESS`,`PRIO_PGRP`,`PRIO_USER`。
* who:是和which取值相对应相关。`PRIO_PROCESS`时候表示取进程id,`PRIO_PGRP`时候取进程组id,`PRIO_USER`时候取用户id。取0的时候表示调用的当前进程.
* pri:范围是`-20~19`,默认为0.越小，优先级越高.

## 2.3 收集Dalvik和native进程栈信息

`dump_traces()`函数来实现收集过程，`/data/anr/traces.txt`文件是由属性`dalvik.vm.stack-trace-file`指定。

* 如果存在`/data/anr/traces.txt`文件，通过rename改成`/data/anr/traces.txt.anr`文件
* 新建新的一个空的`traces.txt`文件来收集栈的dump信息。


遍历`/proc`目录，然后`kill -QUIT`(-3,相当于ctrl+d)所有Dalik进程
```c
DIR *proc=opendir("/proc");
//当进程完成dump，发出通知
int ifd=inotify_init();
//指定pathname,mask关心的通知事件，通知写入发出通知
int wfd=inotify_add_watch(ifd,traces_path,IN_CLOSE_WRITE);
struct dirent *d;
int dalvik_found=0;
//读一个目录
while((d=readdir(proc))){
    // proc子目录是进程(号)
    int pid=atoi(d->d_name);
    //proc/%d/exe 执行程序名称
    //1. linux:ps -p 2345 -o comm
    //2. android:busybox ps
    snprint(path,sizeof(path),"/proc/%d/exe",pid);
    ssize_t len=readlink(path,data,sizeof(data)-1);
    if(!strcmp(data,"/system/bin/app_process")){
        //执行的命令行参数
        snprintf(path,sizeof(pat),"proc/%d/cmdline",pid)
        int fd=open(path,O_READONLY);
        len=read(fd,date,sizeoof(data)-1);
        close(fd);
        //跳过zygote进程
        if(!strcmp(data,"zygote")){
            continue;
        }
        //说明dalvik进程来自app_process
        ++dalvik_found;
        if(kill(pid,SIGQUIT)){
            fprintf(stderr,"kill(%d,SIGQUIT):%s\n",pid,strerror(errno));
            continue;
        }
        //等待写完成通知
        struct pollfd pfd={ifd,POLLIN,0};
        //利用inotify来添加200ms超时判断.
        int ret=poll(&pfd,1,200);
        if(ret<0){
        }else if(ret==0){
            fprintf(stderr,"warining:time out dumping pid%d\n",pid);
        }else{
            struct inotify_event id;
            read(ifd,&ie,sizeof(ie));
        }
    }else if(should_dump_native_traces(data)){
        //dump native process if appropriate
        if(lseek(fd,0,SEEK_END)<0){
            fprintf(stderr,"lseek:%s\n",strerror(errno));
        }else{
            dump_backtrace_to_file(pid,fd);
        }
    }
    //完成返回文件名为 traces.txt.bugreport
}
```

## 2.4回到非root用户和组

dumpstate由init启动，具有root权限
```c
gid_t groups[] ={AID_LOG,AID_SDCARD,AID_SDCARD_RW,AID_MOUNT,AID_INET,AID_NET_BW_STATS};
setgroups(sizeof(groups)/sizeof(groups[0],groups)!=0);
setgid(AID_SHELL);
setuid(AID_SHELL)
```

## 2.5 dumpstate()正真执行
下面单独分析
## 2.6 通知bugreport操作

```c
if(do_broadcast && use_outfile && do_fb){
    //run_command(title,timeout,char*command,...);
    run_command(NULL,5,"/system/bin/am","broadcast","--user","0"
    "-a","android.intent.action.BUGREPORT_FINISHED",
    "--es","android.intent.extra.BUGREPORT",path,
    "--es","android.intent.extra.SCREENSHOT",screenshot_path,
    "--receiver-permission","android.permission.DUMP",NULL); 
}
```

utils.c文件中`run_command()`

```c
int run_command(const char*title,int timeout_seconds,const char*command){
    fflush(stdout);
    clock_t start=clock();
    pid_t pid=fork();
    if(pid<0){
        printf("fork:%s\n",strerror(errno));
        return pid;
    }else if(pid==0){//子进程
        const char *args[1024]={command};
        size_t arg;
        //父进程死了，自己成发送SIGKILL
        prctl(PR_SET_PDEATHSIG,SIGKILL);
        va_list ap;
        va_start(ap,command);
        if(title)printf("----- %s (%s",title,command);
        for(arg=1;arg<sizeof(args)/sizeof(args[0]);++args){
            args[arg]=va_arg(ap,const char*);
            if(args[arg]==NULL)break;
            if(title)printf(" %s",args[arg]);
        }
        if(title)printf(") -----\n");
        fflush(stdout);
        execvp(command,(char**)args);
        printf("***exec(%s):%s\n",command,strerror(errno));
        fflush(stdout);
        _exit(-1);
    }
    //主进程处理
    for(;;){
        int status;
        pid_t p=waitpid(pid,&status,WNOHANG);
        float elapsed=(float)(clock()-start)/CLOCKS_PER_SEC;
        //fork()返回子进程pid
        if(p==pid){
        //判断子进程返回的状态
           if(WIFSIGNALED(status)){
                printf("*** %s Killed by signal %d\n",command,WIERMSIG(status));
           }else if(WIFEXITED(status)&& WEXISTATUS(status)>0){
                printf("** %s Exit code %d\n",command,WEXISTATUS(status));
           }
           if(title) printf("%s %.1fs elasped\n",command,elapsed);
            return status;
        }
        if(timeout_seconds && elapsed > timeout_seconds){
            printf("**** %sTimeout after %.1fs (killing pid %d)\n",command,elapsed,pid);
            kill(pid,SIGTERM);
            return -1;
        }
        usleep(100000);//pll every 0.1 sec
    }
}
```

# 3.dumpstate分析
主要通过`run_command`,`dump_file`,`do_dmesg`,`print_properties`,`for_each_pid`，完成各种信息的dump.

## 3.1 `dump_file`

```c
int dump_file(const char* tile,const char *path){
    int fd=open(path,O_RDONLY);
    int newline=0;
    for(;;){
        int ret=read(fd,buffer,sizeof(buffer));
        ret=fwrite(buffer,ret,1,stdout);
    }
    if(reg<=0) break;
}
close(fd);
```

## 3.2 `do_dmesg`
打印KERNEL信息。

```c
void do_dmesg(){
    printf("----------KERNEL LOG(dmesg)------\n");
    //get size of kernel buffer
    int size=klogctl(KLOG_SIZE_BUFFER,NULL,0);
    char * buf=(char*)malloc(size+1);
    int retval=klogctl(KLOG_READ_ALL,buf,size);
    buf[retval]='\0';
    printf("%s\n\n",buf);
    free(buf);
    return;
}
```

## 3.3 `print_properties()`
打印所有的系统属性

```c
size_t num_props=0;
static char*props[2000];
static void print_prop(const char*key,const char*name,void*user){
    //未使用该变量，避免编译器出现 unused but defined warning.
    (void) user;
    //属性数目小于2000
    if(num_props<sizeof(props)/sizeof(props[0])){
        char buf[PROPERTY_KEY_MAX+PROPERTY_VALUE_MAX+10];
        snprintf(buf,sizeof(buf),"[%s] : [%s]\n",key,name);
        props[num_props++]=strdup(buf);
    }
}
static int compare_prop(const void*a,const void*b){
    return strcmp(*(char* const*)a,*(char*const*)b);
}
void print_properties(){
    size_t i;
    num_props=0;
    //遍历所有的属性，并回调回来
    property_list(print_prop,NULL);
    //排序数组
    qsort(&prop,num_props,sizeof(props[0]),compare_prop);
    printf("----------------SYSTEM PROPERTIES ---------\n");
    for(i=0;i<num_props;++i){
        fputs(props[i],stdout);
        free(props[i]);
    }
    printf("\n");
}
```

其中属性通过内存映射将`/dev/__properties__`到一个全局变量`__system_property_area__`.利用二叉树，管理属性。

## 3.4 `for_each_pid` 和 `for_each_tid`

调用`for_each_pid(do_showmap,"SMAPS OF ALL PROCESS")`和`for_each_tid(show_wchan,"BLOCKED PROCESS WAIT-CHANNELS")`

```c
void for_each_pid(for_each_pid_func func,const char*header){
    __for_each_pid(for_each_pid_helper,header,func); 
}
void for_each_pid_helper(int pid,const char*cmline,void*arg){
    for_each_pid_func *func=arg;
    func(pid,cmdline);
}
void __for_each_pid(void(*helper)(int,const char*,void*),const char*header,void *arg){
    DIR*d;
    struct dirent *de;
    if(!(d=opendir("/proc")));
    while((de=readdir(d))){
        int pid,fd;
        char cmdpath[256],cmdline[256];
        if(!(pid=atoi(de->[d_name]))){
            continue;
        }
        sprintf(cmdpath,"/proc/%d/cmdline",pid);
        memset(cmdline,0,sizeof(cmdline));
        if((fd=open(cmdpath,O_RDONLY))<0){
            strcpy(cmdline,"N/A");
        }else{
            read(fd,cmdline,sizeof(cmdline),-1);
            close(fd);
        }
        helper(pid,cmdline,arg);
    }
    closedir(d);
}
```
for_each_tid类似。

# 4.总结
bugreport通过socket与dumpstate服务建立通信。dumpstate()主要5大类:

* current log:`kernel,system,event,radio`
* last log:`kernel ,system,radio`
* vm traces:`just now,last ANR,tombstones`
* dumpsys:`all checkin,app`
* system info:`cpu,memory,io`

从bugreport内容输出顺序角度，详细内容:

* 系统build以及运行时长等信息。`/proc/version`,运行时间`uptime`
* 内存/CPU进程等信息。`/proc/meminfo`,CPU信息：`top -n 1 -d 1 -m 30 -t`,`/proc/vmstat`,进程`ps -P`,线程`ps -t -p -P`
* `kernel log`
* `lsof`,`map`,`wait-channels`,LIST OF OPEN FILE(`/system/xbin/su lsof`)
* `system log`,`logcat -v threadtime -d *:v`
* `event log`,`logcat -b events -v threadtime -d *:v`
* `radio log`:`logcat -b radio -v threadtime -d *:v`
* `vm traces`
    * VM TRACES JUST NOW(`/data/anr/traces.txt.bugreport`)(抓bugreport时出发)
    * VM TRACES AT LAST ANR(`/data/anr/traces.txt`)(存在就输出)
    * TOMBSTONE(/data/tombstones/tombstone_xx)(存在就输出)
* network相关信息:NETWORK DEV INFO`/proc/net/dev`,NETWORK ROUTES`/proc/net/route`
* `last kernel log`,`proc/last_kmsg`
* `last panic console`,`/data/dontpanic/apanic_console`
* `last panic threads`,`/data/notpanic/apanic_threads`
* SYSTEM SETTINGS:`sqlite3 /data/data/com.android.providers.settings/databases/settings.db pragma user_version; select * from system; select * from secure; select * from global;`
* `last system log`
* `ip相关信息`
* `中断向量表`
* `property`以及fs等信息
* `last radio log`
* Binder相关信息
* dumpsys all;
* dumpsys checkin相关：
    * dumpsys batterystats电池统计
    * dumpsys meminfo内存
    * dumpsys netstats 网络统计
    * dumpsys procstats 进程统计
    * dumpsys usagestats使用统计
    * dumpsys package
* dumpsys app相关
    * dumpsys activity
    * dumpsys acitivty service all
    * dumpsys activity provider all

## 4.1 ChkBugReport

1.通过命令生成bugreport文件

```c
bugreport > bugreport.txt

```

2.执行chkbugreport，命令中jar和txt都必须填写相应文件的完全路径

```c
Java -jar chkbugreport.jar bugreport.txt
```
当然可以吧`.jar`添加path，则直接使用`chkbugreport bugreport.txt`

3.通过浏览器打开`/bugreport_out/index.html`，可视化信息出现

**参考**：



[bugreport源码篇](http://gityuan.com/2016/06/10/bugreport/)

