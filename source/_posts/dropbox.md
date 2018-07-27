---
title: dropbox
date: 2018-07-27 02:34:49
tags: Android 调试
categories: Android
---
# 1.DropBoxManager
 
dropboxmanagerSerivce(DBMS)，记录系统关键log信息，主要用于Debug调试。收集的信息都存放在`/data/system/dropbox`目录下。

当出现crash,anr,wtf,lowmem，以及开机完成时会通过DropBoxManager，收集系统重要的信息。根据不同场景输出相应的信息：

* CRASH：输出发生crash时的当前线程的调用栈信息
* ANR：输出Cpuinfo,以及重要进程的各个线程的traces文件(kill -3);
* watchdog:也输出重要进程的各个线程traces文件(kill -3)
