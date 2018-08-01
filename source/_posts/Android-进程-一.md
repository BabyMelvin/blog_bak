---
title: Android 进程(一)
date: 2018-07-29 22:55:00
tags: Android进程
categories: Android
---

# 1.优先级

Android 进程优先级，分为10级。优先级调度方法`setThreadPriority(int tid,int priority)`

<!--more-->
## 1.1 进程优先级

|进程优先级|nice值|解释|
|--|--|--|
|THREAD_PRIORITY_LOWEST|19|最低优先级|
|THREAD_PRIORITY_BACKGROUND|10|后台|
|THREAD_PRIORITY_LESS_FAVOERABLE|1|比默认低|
|THREAD_PRIORITY_DEFAULT|0|默认|
|THREAD_PRIORITY_MORE_FAVORABLE|-1|比默认高|
|THREAD_PRIORITY_FORGROUND|-2|前台|
|THRAED_PRIORITY_DISPLAY|-4|显示相关|
|THREAD_PRIORITY_URGENT_DISPLAY|-8|显示（更为重要），input事件|
|THREAD_PRIORITY_AUDIO|-16|音频相关|
|THREAD_PRIORITY_URGENT_AUDIO|-19|音频（更为重要）|

## 1.2组优先级别

|组优先级|取值|解释|
|--|--|--|
|THREAD_GROUP_DEFAULT|-1|仅用于setProcessGroup,将优先级M=10进程提升到-2|
|THREAD_GROUP_BG_NONINTERACTIVE|0|CPU分时的时长缩短|
|THREAD_GROUP_FROGROUND|1|CPU分时的时长正常|
|THREAD_GROUP_SYSTEM|2|系统线程组|
|THREAD_GROUP_AUDIO_APP|3|应用程序音频|
|THREAD_GROUP_AUDIO_SYS|4|系统程序音频|

# 2. 调度器选择

设置调度器方法：`setThreadSchduler(int tid int policy,int priority)`

|调度器|名称|解释|
|--|--|--|
|SCHED_OTHER|默认|标准round-robin分时共享策略|
|SCHED_BATCH|批处理调度|针对具有batch风格(批处理)进程的调度策略|
|SCHED_IDLE|空闲调度|针对优先级非常低的适合在后台运行的进程|
|SCHED_FIFO|先进先出|实时调度策略，android暂未实现|
|SCHED_RR|循环调度|实时调度策略，Android暂未实现|

# 3.Android进程声明周期

进程的重要性，划分5级：

* 1.前台进程(Foreground process)
	* 用户当前操作所必需的进程。
* 2.可见进程(Visible process)
	* 没有任何前台组件、但仍会影响用户在屏幕上所见内容的进程。
* 3.服务进程(Service process)
	* 尽管服务进程与用户所见内容没有直接关联，但是它们通常在执行一些用户关心的操作（例如，在后台播放音乐或从网络下载数据）。
* 4.后台进程(Background process)
	* 后台进程对用户体验没有直接影响，系统可能随时终止它们，以回收内存供前台进程、可见进程或服务进程使用。
* 5.空进程(Empty process)
	* 保留这种进程的的唯一目的是用作缓存，以缩短下次在其中运行组件所需的启动时间。 为使总体系统资源在进程缓存和底层内核缓存之间保持平衡，系统往往会终止这些进程。

前台进程的重要性最高，依次递减，空进程的重要性最低。

# 4.Lowmemorykiller

Android中对于内存的回收，主要依靠Lowmemorykiller来完成，是一种根据阈值级别触发相应力度的内存回收的机制。

## 4.1  ADJ级别
定义在ProcessList.java文件，oom_adj划分为16级，从`-17`到`16`之间取值。

|ADJ级别|取值|解释|
|--|--|--|
|UNKNOWN_ADJ|16|一般指将要会缓存进程，无法获取确定值|
|CACHED_APP_MAX_ADJ|15|不可见进程的adj最大值 1|
|CACHED_APP_MIN_ADJ|9|不可见进程的adj最小值 2|
|SERVICE_B_AD|8|B List中的Service（较老的、使用可能性更小）|
|PREVIOUS_APP_ADJ|7|上一个App的进程(往往通过按返回键)|
|HOME_APP_ADJ|6|Home进程|
|SERVICE_ADJ|5|服务进程(Service process)|
|HEAVY_WEIGHT_APP_ADJ|4|后台的重量级进程，`system/rootdir/init.rc`文件中设置|
|BACKUP_APP_ADJ|3|备份进程 3|
|PERCEPTIBLE_APP_ADJ|2|可感知进程，比如后台音乐播放 4|
|VISIBLE_APP_ADJ|1|可见进程(Visible process) 5|
|FOREGROUND_APP_ADJ|0|前台进程（Foreground process） 6|
|PERSISTENT_SERVICE_ADJ	|-11|关联着系统或persistent进程|
|PERSISTENT_PROC_ADJ|-12|系统persistent进程，比如telephony|
|SYSTEM_ADJ|-16|系统进程|
|NATIVE_ADJ|-17|native进程（不被系统管理）|

## 4.2 进程state级别
定义在ActivityManager.java文件，process_state划分18类，从`-1`到`16`之间取值。

|state级别|取值|解释|
|--|--|--|
|PROCESS_STATE_CACHED_EMPTY|16|进程处于cached状态，且为空进程|
|PROCESS_STATE_CACHED_ACTIVITY_CLIENT|15|进程处于cached状态，且为另一个cached进程(内含Activity)的client进程|
|PROCESS_STATE_CACHED_ACTIVITY|14|进程处于cached状态，且内含Activity|
|PROCESS_STATE_LAST_ACTIVITY|13|后台进程，且拥有上一次显示的Activity|
|PROCESS_STATE_HOME|12|后台进程，且拥有home Activity|
|PROCESS_STATE_RECEIVER|11|后台进程，且正在运行receiver|
|PROCESS_STATE_SERVICE|10|后台进程，且正在运行service|
|PROCESS_STATE_HEAVY_WEIGHT|9|后台进程，但无法执行restore，因此尽量避免kill该进程|
|PROCESS_STATE_BACKUP|8|后台进程，正在运行backup/restore操作|
|PROCESS_STATE_IMPORTANT_BACKGROUND|7|对用户很重要的进程，用户不可感知其存在|
|PROCESS_STATE_IMPORTANT_FOREGROUND|6|对用户很重要的进程，用户可感知其存在|
|PROCESS_STATE_TOP_SLEEPING|5|与PROCESS_STATE_TOP一样，但此时设备正处于休眠状态|
|PROCESS_STATE_FOREGROUND_SERVICE|4|拥有给一个前台Service|
|PROCESS_STATE_BOUND_FOREGROUND_SERVICE|3|拥有给一个前台Service，且由系统绑定|
|PROCESS_STATE_TOP|2|拥有当前用户可见的top Activity|
|PROCESS_STATE_PERSISTENT_UI|1|persistent系统进程，并正在执行UI操作|
|PROCESS_STATE_PERSISTENT|0|persistent系统进程|
|PROCESS_STATE_NONEXISTENT|-1|不存在的进程|

## 4.3 lmk策略

lowmemorykiller根据当前可用内存情况来进行进程释放，总设计了6个级别：

* 1.CACHED_APP_MAX_ADJ
* 2.CACHED_APP_MIN_ADJ
* 3.BACKUP_APP_ADJ
* 4.PERCEPTIBLE_APP_ADJ
* 5.VISIBLE_APP_ADJ
* 6.FOREGROUND_APP_ADJ

系统内存从很宽裕到不足，`Lowmemorykiller`也会相应地从CACHED_APP_MAX_ADJ(第1档)开始杀进程，如果内存还不足，那么会杀CACHED_APP_MIN_ADJ(第2档)，不断深入，直到满足内存阈值条件。