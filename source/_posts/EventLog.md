---
title: EventLog
date: 2018-07-22 06:18:35
tags: Android调试
categories: Android 
---

# 概述
Eventlog可以展示当前Activity各种状态，还可以显示window信息。
tags格式定义位于文件`/system/etc/event-log-tags`.
终端输入：`logcat -b events`

那么会输出大量类似这样的信息：

```
06-01 13:44:55.518  7361  8289 I am_create_service: [0,111484394,.StatService,10094,7769]
06-01 13:44:55.540  7361  8343 I am_proc_bound: [0,3976,com.android.providers.calendar]
06-01 13:44:55.599  7361  8033 I am_create_service: [0,61349752,.UpdateService,10034,1351]
06-01 13:44:55.625  7361  7774 I am_destroy_service: [0,61349752,1351]
...
```
通过字面意思，就能得到不少信息量，比如`am_create_service`，创建service，但是后面括号中内容的具体含义，其实有很高的价值。 接下来通过一张表格来展示含义

# EventLog
## ActivityManager

|num|TagName|格式|功能|
|--|--|--|--|
|30001|am_finish_activity|User,Token,TaskID,ComponentName,Reason||

下面列举tag可能使用的部分场景：

* `am_low_memory`：位于AMS.killAllBackgroundProcesses或者AMS.appDiedLocked，记录当前Lru进程队列长度。

Activity生命周期相关的方法:

* `am_on_resume_called`:位于AT.performResumeActivity

Window相关


[参考][1]
[1]: http://gityuan.com/2016/05/15/event-log/ "Android EventLog含义"
