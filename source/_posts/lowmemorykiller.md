---
title: lowmemorykiller
date: 2018-07-27 04:27:54
tags: Android 内存管理
categories: Android
---

# 1.概述
Android为了快速响应，应用程序退出，还需继续存在系统以再次启动时提高响应时间。太多后台进程，需要进行管理，根据一定策略进行释放进程，有了`lmk`,`lmkd`来决定什么时间杀掉什么进程.来决定什么时间杀掉什么进程.

Linux有类似内存管理策略--OOM killer(Out of Memory Killer).OOM策略更多的是用于分配内存不足时触发，`将得分最高的进程杀掉`。而`lmk`则会每隔一段时间检查一次，当系统剩余可用内存较低时，便会触发进程的策略，根据不同的剩余内存档位来选择杀不同的优先级的进程，而不是等到OOM时再来杀进程，真正OOM时系统可能已经处于异常状态，系统更希望的是未雨绸缪，在内存很低的时候杀掉一些优先级较低的进程来保障后续操作的顺利进行。

涉及到java层：`ProcessList.java`,native层：`system/core/lmkd`,driver：`staging/lowmemroykiller.c`.主要通过三个命令来完成调动操作：

|功能|命令|对应方法|
|--|--|--|
|LMK_PROCPRIO|设置进程adj|`PL.setOomAdj()`|
|LMK_TARGET|更新oom_adj|`PL.updateOomLevels()`|
|LMK_PROCREMOVE|移除进程`PL.remove()`|

# 2.源码分析
`lmkd`通过socket将framework层与native层联系起来，native层通过init解析服务信息与driver内核进程通信。

## 2.1 framwork层

```c
//ActivityManagerService.java
private final boolean aplyOomAdjLocked(Process add,boolean doingAll,
long now,long nowEllaped){
    if(app.curAdj!=app.setAdj){
        ProcessList.setOomAdj(app.id,app.info.uid,app.curAdj);
        app.setAdj=app.curAdj;
    }
}

///ProcessList.java
public static final void setoomAdj(int pid,int uid,int amt){
    //当adj=16,则直接返回
    if(amt==UNKOWN_ADJ) 
        return ;
    long start=SysteClock.elapsedRealTime();
    ByteBuffer buf=ByteBuffer.allocate(4*4);
    buf.putInt(LMK_PROCPRIO);
    buf.putInt(pid);
    buf.putInt(uid);
    buf.putInt(amt);//目标adj
    //将16字节写入socket,
    writeLmkd(buf);
    long now =SystemClock.elapedRealtime();
    if((now-start)>250){
        Slog.w("ActivityManage");
    }
}

private static void writeLmkd(ByteBuffer buf){
    //当socket打开失败会尝试3次
    for(int i=0;i<3;i++){
        if(sLmdSocket ==null){
            //打开socket
            if(openLmkdSocket() == false){ 
                try{
                    Thread.sleep(1000);
                }catch(){
                }
                continue;
            }
        }
        try{
            //将buf信息写入lmkd socket
            sLmkdOututStream.write(buf.arry(),0,buf.position());
            return;
        }catch(IOException ex){
            try{
                sLmkdSocket.close();
            }catch(IOException ex2){
            }
            sLmkdSocket=null;
        }
    }
}
private static boolean openLmkdSocket(){
    try{
        sLmkdSocket=new LocalSocket(LocalSocket.SOCKET_SEQPACKET);
        //远程lmkd守护进程建立socket连接
        sLmkdSocket.connect(new LocalSocketAddress("lmkd",LocalSocketAddress.Namespace.RESERVED));
        sLmkdOutputStream=sLmkdSocket.getOutputStream();
    }catch(){
    }
    return true;
}
```

`sLmkdSocket`采用的是`SOCK_SEQPACKET`，这是类型socket提供顺序确定的可靠的，双向基于连接的socket endpoint。与`SOCK_STREAM`很相似，唯一不同的是SEQPACEKET保留消息边界，而SOCK_STREAM是基于字节流，并不会记录边界。 read端能够知道要读取数据大小。

## 2.2 native层
lmkd是由init进程，通过解析init.rc文件启动的Lmkd守护进程，lmkd会创建名为`lmkd`的socket，节点位于`/dev/socket/lmkd`,该socket用于跟上层framework交互。

```
service lmkd /system/bin/lmkd
    class core
    critical
    socket lmkd seqpacket 0660 system system
    writepid /dev/cpuset/system-backgroud/tasks
```

lmkd启动，执行可制成程序`lmkd`

```c
int main(int argc __unused,char **argv __unused){
    struct shed_param param={
        .sched_priority=1,
    };
    mLockall(MCL_FUTURE);
    shed_setscheduler(0,SCHED_FIFO,&param);
    //初始化
    if(!init()){
        mainloop();
    }
    return 0;
}

static int init(void){
    struct epoll_event epev;
    int i,ret;
    page_k =sysconf(_SC_PAGESIZE);
    if(page_k==-1)
        page_k=PAGE_SIZE;
    page_k/=1024;
    //创建epoll监听文件句柄
    epollfd=epoll_create(MAX_EPOLL_EVENTS);

    //获得lmkd控制描述符
    ctrl_lfd=android_get_control_socket("lmkd");
    //监听lmkd socket
    ret=listen(ctrl_fd,1);

    epev.events=EPOLLIN;
    epev.data.ptr=(void*)ctrl_connect_handler;
    //将文件句柄ctrl_lfd,加入epoll句柄
    epoll_ctl(epollfd,EPOLL_CTL_ADD,ctrl_lfd,&epev);
    maxevents++;
    //该路径是否具有可写的权限
    use_inkernel_interface=!access(INKERNEL_MINFREE_PATH,W_OK);
    if(use_inkernel_interface){
        ALOGI("Using inkernel low memory killer interface"):
    }else{
        ret=init_mp(MEMPERSSURE_WATH_LEVEL,(void*)&mp_event);
        if(ret)
            ALOGE("kernel does not support memeory pressure events or in-kernlr low memory");
    }
    for(i=0;i<=ADJTOSLOT(OOM_SCORE_ADJ_MAX);i++){
        procadjslot_list[i].next=&procadjslot_list[i];
        procadjslog_list[i].prev=&procadjslog_list[i];
    }
    return 0;
}
```

通过检查`/sys/module/lowmemorykiller/parameters/minfree`节点是否具有可写权限来判断是否使用kernel来管理Lmk事件。默认该节点具有系统可写权限。

```c
static void mainloop(void){
    while(1){
        struct epoll_event events[maxevents];
        int events,i;
        ctrl_dfd_reopened=0;
        //等待epollfd上的事件
        nevents=epoll_wait(epollfd,events,maxevents,-1);
        if(events==-1){
            if(errono==EINTR){
                continue;
            }
            continue;
        }
        for(i=0;i<nevents;++i){
            if(events[i].events & EPOLLERR){
                ALOGD("EPOLLERR on events #%d",i);
            }
            //当事件到来，则盗用ctrl_connects_handler方法
            if(events[i].data.ptr){
                (*(*void(*))events[i].data.ptr)(events[i].events);
            }
        }
    }
}
```

主循环调动`epoll_wait()`，等待epollfd的事件，当接收到中断或者不存在事件，则执行continue操作。当事件到来，则调用的ctr_connect_handler方法，该方法是由`init()`过程中设定的方法.

```c
static void ctrl_connet_handler(uint32_t events __unused){
    struct epoll_event epev;
    if(ctrl_dfd >=0){
        ctrl_data_close();
        ctrl_dfd_reopened=1;
    }
    ctrl_dfd=accept(ctrl_lfd,NULL,NULL);
    ALOGI("ActivityManager connected");
    maxevents ++;
    epev.events=EPOLLIN;
    epev.data.ptr=(void*)ctrl_data_handler;
    //将ctrl_lfd添加到epollfd
    if(epoll_ctrl(epollfd,EPOLL_CTL_ADD,ctrl_dfd,&epev)==-1){
        ctrl_data_close();
        return;
    }
}
```

## 2.2.1 处理事件

```c
static void ctrl_data_handler(uint32_t events){
    if(events & EPOLLHUP){
        if(!ctrl_dfd_reopned)
            ctrl_data_close();
    }else if(events & EPOLLIN){
        ctrl_command_handler();
    }
}
static  void ctrl_comand_handler(void){
    int ibuf[CTRL_PACKET_NAX /sizeof(int)];
    int len,cmd=-1,nargs,targets;
    le=ctr_data_read((char*)ibuf,CTRL_PACKET_MAX);
    if(len<=0)
        return;
    nargs=len/sizeof(int)-1;
    if(nargs<0)
        goto wronglen;
    //将网络字节顺序转化为主机字节顺序
    cmd=ntohl(ibuf[0]);
    switch(cmd){
        case LMK_TARGET:
            targets=nargs/2;
            if(nargs & 0x01 || targets >(int)ARRAY_SIZE(lowmem_adj)){
                goto wronglen;
            }
            cmd_target(targets,&ibuf[1]);
            break;
        case LMK_PROCPRIO:
            if(args !=3)
                goto wronglen;
            //设置进程adj
            cmd_proprio(ntohl(ibuf[i]),ntohl(ibuf[2]),ntohl(ibuf[3]));
            break;
        case LMK_PROCREMOVE:
            if(nargs !=1)
                goto wronglen;
            cmd_procremove(ntohl(ibuf[1]));
            break;
        default:
            ALOGE("Received unkown command code %d",cmd);
            return ;
    }
wronglen:
    ALOGE("Wrong control socket read length cmd=%d, len=%d",cmd,len);
}
```

不同的分支,进入相应的分支.

```c
static void cmd_procprio(int pid,int uid,int oomadj){
    struct proc* procp;
    char path[800],val[20];
    snprintf(path,sizeof(path),"/proc/%d/oom_scorre_adj",pid);
    snprintf(val,sizeof(val),"%d",oomadj);
    //向节点/proc/<pid>/oom_score_adj写入oomadj
    writefilestring(path,val);
    //当使用kernel方式则直接返回
    if(use_ikernel_interface)
        return;
    procp=pid_lookup(pid);
    if(!procp){
        procp=malloc(sizeof(struct proc));
        if(!procp)
            return;
        procp->pid=pid;
        procp->uid=uid;
        procp->oomadj=oomadj;
        proc_insert(procp);
    }else{
        procp_unslot(procp);
        procp->oomadj=oomadj;
        procp->slot(procp);
    }
}
```

## 2.3 小结

use_kernel_interface该值后续应该会逐渐采用用户空间策略。不过目前仍然use_inkernel_interface=1:

* LMK_TARGET:`AMS.updateConfiguation()`的过程中调用`updateOomLevels()`方法，分别向`/sys/module/lowmemorykiller/parameters`的`minfree` 和 `adj`节点写入相应信息.
* LMK_PROCPRIO:`AMS.applyOomAdjLocked()`的过程中调动的`setOoAdj()`，向`/proc/<pid>/oom_score_adj`写入omadj,则直接返回。
* LMK_PROCREMOTE:`AMS.handleAppDieLocked`或者`AMS.cleanUpApplicationRecordLocked()`的过程,调用`remove()`,目前不做任何事，直接返回.

# 3.Kernel层
位于`/drivers/staging/Android/lowmemorykiller.c`

```c
static struct shrinker lowmem_shrinker ={
    .scan_objects =lowmem_scan;
	.count_objects=lowmeme_count;
	.seeks=DEFAULT_SEEKS * 16;
};
 static int __init lowmem_init(void){
	register_shrinker(&lowmem_shrinker);
	return 0;
}
static void __exit lowmem_exit(void){
	unregister_shrinker(&lowmem_shrinker);
}
module_init(lowmem_init);
module_exit(lowmem_exit);
```

LMK驱动通过注册shrinker实现的，shrinker是linux kernel标准的回收内存page的机制，由内核线程kswapd负责控制。

当内存不足时kswapd线程会边里一张shrinker链表，并回调已注册的shrinker函数来回收内存page,kswpad还会周期性唤醒来执行内存操作。每个zone维护`active_list`和`inactive_list`，内核根据页面活动状态将page在这两个链表之间移动，最后通过shrink_slab和shrink_zone来回收内存页。

## 3.1 Lowmem_count

```c
static unsigned long lowmem_count(struct shrinker* s,struct shrink_control *sc){
	return global_page_state(NR_ACTIVE_ANON)+
		global_page_state(BR_ACTIVE_FILE)+
		global_page_state(NR+INACTIVE+ANON)+
		global_page_state(NR_INACTIVE_FILE);
}
```

`ANON`代表匿名映射，没有后备存储器；`FILE`代表文件映射；`内存映射`计算公式=`活动匿名内存`+`活动文件内存`+`不活动匿名内存`+`不活动文件内存`.

## 3.2 lowmem_scan

当触发lmkd，则先杀`oom_score_adj`最大的进程，当oom_adj相等时，则选择rss最大的进程。

```c
static unsigned long lowmem_scan(
struct shrinker*s,struck shrinker_control *sc
){
	struct task_struct *tsk,*selected=NULL;
	unsigne long rem=0;
	int tasksize,i,minfree=0,selected_tasksize=0;
	short min_score_ajd=OOM_SCORE_ADJ_MAX+1;
	int array_size=ARRAT_SIZE(lowmem_adj);
	//获取当前剩余内存大小
	int other_free=global_page_state(NR_FREE_PAGES)-totalreserve_pages;
	int other_file=global_page_state(NR_FILE_PAGES)-global_page_state(NR_SHMEM)-
		total_swapcache_pages();
	//获取数组大小
	if(lowmem_adj_size<array_size){
		array_size=lowmem_adj_size;
	}
	if(lowmem_minfree_size<array_size)
		array_size=lowmem_minfree_size;

	//遍历lowmem_minfree数组找出
	for(i =0;i<array_size;i++){
		minfree=lowmem_minfree[i];
		if(other_free<minfree && other_file<minfree){
			min_score_adj=lowmem_adj[i];
			break;
		}
	}
	if(min_score_adj ==OOM_SCORE_ADJ_MAX+1){
		return 0;
	}
	selected_oom_score_adj =min_score_adj;
	rcu_read_lock();
	for_each_process(tsk){
		struct task_struct*p;
		short oom_score_adj;
		if(tsk->flags& PF_KTHREAD)
			continue;
		p=find_lock_task_mm(tsk);
		if(!p)
			continue;
		if(task_tsk_thread_flag(p,TIF_MEMDIE)&&
			time_before_eq(jiffies,lowmem_deathpending_timeout)){
			task_unlock(p);
			rcu_read_unlock();
			return 0;
		}
	}
	oom_score_adj=p->signal->oom_score_adj;
	//小于目标adj进程，则忽略
	if(oom_score_adj<min_score_adj){
		task_unlock(p);
		continue;
	}
	//算法关键，选择oom_score_adj最大的进程中，并且rss内存最大的进程
	if(selected){
		if(oom_score_adj<selected_oom_score_adj)
			continue;
		if(oom_score_adj==selected_oom_score_ajd&& tasksize<=selected_tasksize)
			continue;
	}
	selected=p;
	secleted_tasksize=tasksize;
	selected_oom_score_adj=oom_score_adj;
	lowmem_print(2,"selected ‘%s’ adj %hd ,size %d ,to kill\n",,
	p->comm,p->pid,oom_score_adj,tasksize);
	if(selected){
		long cache_size=other_file*(long)(PAGE_SIZE/1024);
		long cache_limit=minfree*(long)(PAGE_SIZE/10324);
		long free=other_free*(long)(PAGE_SIZE/1024);
		//输出kill 的log
		lowmem_print(1,"killing %s %d ,adj %hd,\n");
		lowmem_deathpending_timeout=jiffies+HZ;
		set_tsk_thread_flag(secleted,TIG_MEMDIE);
		//向选中的目标进程发送signal 9 来杀掉目标进程
		send_sig(SIGKILL,selceted,0);
		rem+=selceted_tasksize;
	}
	rcu_read_unlock();
	return rem;
}
```

`lowmem_minfree`和`lowmem_adj[]`数组大小个数为6,通过如下两条命令:

```
module_param_named(debug_level,lowmem_debug_level,uint,S_IRUGO|S_IWUSE)
module_param_array_named(adj,lowmem_adj,short,&lowmem_adj_size,S_IRUGO|S_IWUSE)
```

当如下节点变化，通过修改`lowmem_minfree[]`和`lowmem_adj[]`数组.

```c
/sys/module/lowmemorykiller/parameters/minfree
/sys/module/lowmemorykiller/parameters/adj
```

# 4.总结

从framework的ProcessList调整adj,通过socket通信将时间发送给native的守护进程lmkd；lmkd再根据具体的命令来执行相应的操作

* 更新进程`oom_score_adj`的值和lowmemorykiller驱动参数`minfree`和`adj`

最后讲到了lowmemorykiller驱动，通过注册shrinker，借助linux标准的内存回收机制，根据当前系统可用内存以及参数配置(adj,minfree)来选取合适的selected_oom_score_adj，再从所有进程中选择adj大于该目标取值并占用rss内存最大的进程，将其杀掉，从而释放出内存。

## 4.1 lmkd参数

* oom_adj:代表进程的优先级，数值越大，优先级越低，越容易被杀，取值范围`[-16,15]`
	* `/proc/<pid>/oom_adj`
* oom_score_adj:取值范围`[-1000,1000]`
	* `/proc/<pid>om_score_adj`
* oom_score:lmkd策略中貌似并没有看下用的地方，应该是oom才会调用
	* `/proc/<pid>/oom_score`

对于`oom_adj`与`oom_score_adj`通过方法`lowmem_oom_adj_to_oom_score_adj()`建立一定映射关系:

* 当`oom_adj=15`,则`oom_score_adj=1000；`
* 当`oom_adj<15`,则`oom_score_adj=oo_adj*1000/17`

## 4.2 driver参数

```
/sys/module/lowmemorykiller/parameters/minfree(代表page个数)
/sys/module/lowmemorykiller/parameters/adj(代表oom_score_adj)
```

举例说明：

* 参数设置：
	* `1,6`写入节点`/sys/module/lowmemorykiller/parameters/adj`
	* `1024,8192`写入节点`/sys/module/lowmemorykiller/parameters/minfree`
* 策略解读:
	* 当系统可用内存低语8192个pages时，则会杀掉`oom_score_adj>=6`的进程
	* 当系统可用内存低于1024个pages时，会杀掉`oom_score_adj>=1`的进程。


