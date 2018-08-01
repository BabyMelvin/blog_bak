---
title: 'Logger系统 '
date: 2018-07-26 10:24:55
tags: Android调试
categories: Android
---

# 1.Logger日志系统
## 1.1 简介
Android提供的Logger日志系统，基内核中的Logger日志驱动程序实现的，将日志记录保存在内核空间中。有效利用内存空间,Logger日志驱动程序内部使用一个`环形缓冲区` 来保存日志。Logger日志环形缓冲区满了之后，新的日志会覆盖旧的日志。
为了避免重要信息被覆盖，日志类型分为四种：`main`,`system`,`radio`和`events`.对应实现的蠕动为：`/dev/log/main`,`/dev/log/system`,`/dev/log/radio`和`/dev/log/events`.

<!--more-->
## 1.2 日志作用

类型具体作用：

* 类型为`main`的日志是应用程序级别
* 类型为`system`日志为系统界别的
* 类型为`radio`日志是与无线设备相关的，量很大，因此单独记录在一起。
* 类型为`event`日志是专门用来诊断系统问题的，应用程序开发不应该使用这种类型日志。

## 1.3 日志的类

Android系统应用框架层中提供了`android.util.Log`,`android.util.Slog`和`android.util.EventLog`三个java接口来往Logger日志驱动程序写入日志，分别对应的类型为main,system,events。其中特别的是使用`android.util.Log`和`android.util.Slog`接口写入日志标签值以`RIL`开头或者等于`HTL_RIL`，`AT`，`GSM`,`STK`,`CDMA`,`PHONE`和`SMS`时，转换为radio类型吸入到Logger日志中。

在C/C++层，Android系统运行时也提供了三组宏来写Logger日志驱动程序，`SLOGV`,`SLOGD`,`SLOGI`和`SLOGE`用来写入sytem类型日志，宏`LOG_EVENT_INT`,`LOG_EVENT_LONG`和`LOG_EVENT_STRING`用来写入events类型日志。

无论是java日志写入接口还是C/C++日志写入接口，它们重罪通过运行时库层日志的liblog来往Logger日志驱动程序写入到日志中，测外提供一个`Logcat`工具读取和显示Logger日志驱动程序中得日志。

# 2. logger日志格式

其中`main`,`system`和`radio`三种类型的格式相同。

```
priority-tag-msg
```

* priority表示日志优先级，是一个整数.
* tag表示日志标签，一个字符串
* msg表示日志内容，也是也一个字符串

优先级一般划分为六种：`VERBOSE`,`DEBUG`,`INFO`,`WARN`,`ERROR`和`FATAL`

events类型的日志标签是一个整数值，显示时候不具有可读性，因此，Android系统使用设备的日志标签文件`/system/etc/event-log-tags`来描述这些标签值的含义。这样Logcat工具在显示events类型日志时，可以把日志中的标签值转换为字符串。格式：

```
tag number-tag name-format for tag value
```

* `tag number`:表示日志标签值，取值范围0~2147483648
* `tag name`:日志标签值对应的字符串描述.字母数字加`_`
* `format for tag value`:日志内容的值格式.

日志内容值格式：

```
[name|data type|data unit]...
```

* name：日志内容值的名称。
* data type表示内容值数据类型，1~4(分别为：1整数，2长整型，3字符串，4列表)
* data unit:表示日志内容的数据单位，1~6(对象数据numbers of objects,字节数number of bytes,毫秒数number of milliseconds,分配额number of allocation,标志(ID)和百分比Percent)

从`/system/etc/event-log-tags`取出一行内容说明：

```
2722 battery_level (level|1|6),(voltage|1|1),(temperature|1|1))
```

* 2722表示日志标签值，`battery_level`用来描述日志标志值2722含义
* 日志便签值三个值组成：level,voltage,temperature



