---
title: Android编译
date: 2018-07-31 04:48:46
tags: Android脚本
categories: Android
---

# 0.概述

`Android Build系统`用来编译`andorid系统`，`Android SDK`以及`相关文档`。
Android Build系统主要由`Make文件`(主要)，`shell脚本`，以及`Python脚本`组成。

主要介绍两个：

* 1.make系统介绍
* 2.添加新产品和新模块.

整个Build系统的Make分为三类：

* 1.整个build系统的框架:`/build/core`
* 2.公司名和产品名两级目录：`/device/amlogic/xxx`
* 3.针对某个模块:`Android.mk`内容执行模块的编译

# 1.编译整个系统

```
source build/envsetup.sh
lunch full-eng
make -j32
```

## 1.1 `/build/envsetup.sh`

定义常用函数：

|序号|函数|说明|
|--|--|--|
|1|croot|切换到源码目录|
|2|m|切换到根目录并执行make|
|3|mm|Build当前目录下的模块|
|4|mmm|Build指定目录下的模块|
|5|cgrep|在所有`c/c++`文件执行grep|
|6|jgrep|在所有Java文件中执行grep|
|7|resgrep|在所有res/*.xml文件上执行grep|
|8|godir|转到包含某个文件的目录路径|
|9|printconfig|显示当前Build配置信息|
|10|add_lunch_combo|在lunch函数菜单中添加一条目录|

## 1.2 build目录结构

* `/out/host`:该目录包含针对主机ANDROID开发工具产物，emulator,adb,aapt等
* `out/target/common`:针对设备共同编译产物，主要是java应用代码和java库
* `out/target/product`:包含特定编译结果以及平台相关`C/C++库`和二进制文件
* `/out/dist`:多种分发而准备的包


## 1.3 Make文件说明


|序号|文件名|说明|
|--|--|---|
|1|`/base/Makefile`|指向`/build/core/main.mk`|
|2|