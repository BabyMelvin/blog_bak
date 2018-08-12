---
title: Android编译
date: 2017-06-31 04:48:46
tags: Android脚本
categories: Android
---

# 0.概述

`Android Build系统`用来编译`andorid系统`，`Android SDK`以及`相关文档`。
Android Build系统主要由`Make文件`(主要)，`shell脚本`，以及`Python脚本`组成。

<!--more-->
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
命令说明：

* 第1行：映入辅助Shell函数，包含lunch函数。
* 第2行：列表的内容会根据当前 Build 系统中包含的产品配置而不同
* 第3行：真正进行编译。j(Job)，是个整数。该值通常是编译主机 CPU 支持的并发线程总数的 1 倍或 2 倍（例如：在一个 4 核，每个核支持两个线程的 CPU 上，可以使用 make -j8 或 make -j16）。调用 make 命令时，如果**没有指定任何目标**，则将使用默认的名称为`droid`目标，该目标会**编译出完整的 Android系统镜像**。
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


## 1.3 编译生成镜像
Build 的产物中最重要的是三个镜像文件，它们都位于 `/out/target/product/<product_name>/ `目录下。这三个文件是：

* `system.img`：包含了 Android OS 的系统文件，库，可执行文件以及预置的应用程序，将被挂载为`根分区`。
* `ramdisk.img`：在启动时将被 Linux 内核挂载为只读分区，它包含了 `/init 文件`和`一些配置文件`。它用来挂载其他系统镜像并启动 `init 进程`。
* `userdata.img`：将被挂载为 `/data`，包含了应用程序相关的数据以及和用户相关的数据。

## 1.3 Make文件说明
整个 Build 系统的入口文件是源码树根目录下名称为**Makefile**的文件，当在源代码根目录上调用 make 命令时，make 命令首先将读取该文件。Makefile 文件的内容只有一行：`include build/core/main.mk`。在 main.mk 文件中又会包含其他的文件，其他文件中又会包含更多的文件，这样就引入了整个 Build 系统。

这些 Make 文件间的包含关系是相当复杂的，图 3 描述了这种关系，该图中黄色标记的文件（且除了 $开头的文件）都位于 build/core/ 目录下。

![Make文件关系](Android编译/image004.png)

|序号|文件名|说明|
|--|--|---|
|1|`/base/Makefile`|指向`/build/core/main.mk`|
|2|main.mk|最主要的 Make 文件，该文件中首先将对`编译环境进行检查`，同时引入其他的 Make 文件。另外，该文件中还定义了几个最主要的 Make 目标，例如 `droid`，`sdk`，等（参见后文“Make 目标说明”）。|
|3|help.mk|说明  包含了名称为`help`的 Make目标的定义，该目标将列出主要的 Make 目标及其说明。|
|4|pathmap.mk|将许`多头文件的路径`通过`名值对的方式定义为映射表`，并提供 `include-path-for`函数来获取。例如，通过 `$(call include-path-for, frameworks-native)`便可以获取到framework本地代码需要的头文件路径。|
|5|envsetup.mk|配置Build系统需要的环境变量，例如：`TARGET_PRODUCT`，`TARGET_BUILD_VARIANT`，`HOST_OS`，`HOST_ARCH` 等。当前`编译的主机平台信息`（例如操作系统，CPU 类型等信息）就是在这个文件中确定的。另外，该文件中还指定了`各种编译结果的输出路径`。|
|6|`combo/select.mk`|根据当前编译器的平台选择平台相关的 Make 文件。|
|7|dumpvar.mk|在 Build 开始之前，显示此次 Build 的配置信息。|
|8|**config.mk**|整个Build系统的`配置文件`。该文件中主要包含以下内容：1.定义了许多的常量来负责不同类型模块的编译。2.定义编译器参数以及常见文件后缀，例如`.zip`,`.jar`,`.apk`。3.根据 BoardConfig.mk 文件,配置产品相关的参数。4.设置一些常用工具的路径，例如`flex`，`e2fsck`,`dx`。|
|9|**definitions.mk**|在其中定义了大量的函数。这些函数都是Build系统的其他文件将用到的。例如：`my-dir`，`all-subdir-makefiles`，`find-subdir-files`，`sign-package`等，关于这些函数的说明请参见每个函数的代码注释。|
|10|distdir.mk|针对dist目标的定义。dist 目标用来拷贝文件到指定路径。|
|11|dex_preopt.mk|针对启动 jar 包的预先优化|
|12|pdk_config.mk|顾名思义，针对 pdk（Platform Developement Kit）的配置文件。|
|13|`${ONE_SHOT_MAKEFILE}`| ONE_SHOT_MAKEFILE 是一个变量，当使用`mm`编译某个目录下的模块时，此变量的值即为当前指定路径下的Make文件的路径。|
|14|`${subdir_makefiles}`|各个模块的 `Android.mk`文件的集合，这个集合是通过 Python脚本扫描得到的。|
|15|post_clean.mk| 在前一次 Build 的基础上检查当前 Build 的配置，并执行必要清理工作。|
|16|legacy_prebuilts.mk|该文件中只定义了 GRANDFATHERED_ALL_PREBUILT 变量。|
|17|Makefile|被main.mk包含,该文件中的内容是辅助main.mk的一些额外内容。|

## 1.4 多模块，类型

Android 源码中包含了许多的模块，模块的类型有很多种。
* Java 库，C/C++ 库，APK 应用，以及可执行文件等 。并且，Java 或者 C/C++ 库还可以分为静态的或者动态的。
* 库或可执行文件既可能是针对设备（本文的“设备”指的是 Android 系统将被安装的设备，例如某个型号的手机或平板）的也可能是针对主机（本文的“主机”指的是开发 Android 系统的机器，例如装有 Ubuntu 操作系统的 PC 机或装有 MacOS 的 iMac 或 Macbook）的。

不同类型的模块的编译步骤和方法是不一样，为了能够一致且方便的执行各种类型模块的编译，在**config.mk**中定义了许多的常量，这其中的每个常量描述了一种类型模块的编译方式，这些常量有：

* BUILD_HOST_STATIC_LIBRARY
* BUILD_HOST_SHARED_LIBRARY
* BUILD_STATIC_LIBRARY
* BUILD_SHARED_LIBRARY
* BUILD_EXECUTABLE
* BUILD_HOST_EXECUTABLE
* BUILD_PACKAGE
* BUILD_PREBUILT
* BUILD_MULTI_PREBUILT
* BUILD_HOST_PREBUILT
* BUILD_JAVA_LIBRARY
* BUILD_STATIC_JAVA_LIBRARY
* BUILD_HOST_JAVA_LIBRARY

在模块的Android.mk文件中，只要包含进这里对应的常量便可以**执行相应类型模块的编译**。这些常量的值都是另外一个 Make 文件的路径，详细的编译方式都是在对应的 Make 文件中定义的。这些常量和 Make 文件的是一一对应的，对应规则也很简单：常量的名称是 Make 文件的文件名除去后缀全部改为大写然后加上“BUILD_”作为前缀。例如常量BUILD_HOST_PREBUILT的值对应的文件就是 host_prebuilt.mk。

|序号|文件名|说明|
|--|--|--|
|1|host_static_library.mk|定义了如何编译主机上的静态库。|
|2|host_shared_library.mk|定义了如何编译主机上的共享库。|
|3|static_library.mk|定义了如何编译设备上的静态库。|
|4|shared_library.mk|定义了如何编译设备上的共享库。|
|5|executable.mk|定义了如何编译设备上的可执行文件。|
|6|host_executable.mk|定义了如何编译主机上的可执行文件。|
|7|package.mk|定义了**如何编译 APK 文件**。|
|8|prebuilt.mk|定义了**如何处理一个已经编译好的文件** ( 例如 Jar 包 )。|
|9|multi_prebuilt.mk|定义了如何处理一个或多个已编译文件，该文件的实现依赖 prebuilt.mk。|
|10|java_library.mk|定义了如何编译设备上的共享 Java 库。|
|11|static_java_library.mk|定义了如何编译设备上的静态 Java 库。|
|12|host_java_library.mk|定义了如何编译主机上的共享 Java 库。|

不同类型的模块的编译过程会有一些相同的步骤，例如：编译一个 Java 库和编译一个 APK 文件都需要定义如何编译 Java 文件。因此，表 3 中的这些 Make 文件的定义中会包含一些共同的代码逻辑。为了减少代码冗余，需要将共同的代码复用起来，复用的方式是将共同代码放到专门的文件中，然后在其他文件中包含这些文件的方式来实现的。

![模块编译](Android编译/image005.png)

# 2 Make 目标说明

## 2.1 make /make droid

如果在源码树的根目录直接调用“make”命令而不指定任何目标，则会选择默认目标：“droid”（在 main.mk 中定义）。因此，这和执行“make droid”效果是一样的。

`droid 目标`将编译出整个系统的镜像。从源代码到编译出系统镜像，整个编译过程非常复杂。这个过程并不是在droid一个目标中定义的，而是**droid目标会依赖许多其他的目标**，这些目标的互相配合导致了整个系统的编译。

![droid 目标所依赖](Android编译/image006.png)

|序号|名称|说明|
|--|--|--|
|1|app_only|该目标将编译出当前配置下不包含`user`,`userdebug`,`eng`标签（关于标签，请参见后文**添加新的模块**）的应用程序。|
|2|droidcore|该目标仅仅是所依赖的几个目标的组合，其本身不做更多的处理。|
|3|dist_files|该目标用来拷贝文件到 /out/dist 目录。|
|4|files| 该目标仅仅是所依赖的几个目标的组合，其本身不做更多的处理。|
|5|prebuilt|该目标依赖于 `$(ALL_PREBUILT)`，`$(ALL_PREBUILT)`的作用就是处理所有已编译好的文件。|
|6|$(modules_to_install)|modules_to_install变量包含了当前配置下所有会被安装的模块（一个模块是否会被安装依赖于该产品的配置文件，模块的标签等信息),因此**该目标将导致所有会被安装的模块的编译**。|
|7|$(modules_to_check)|该目标用来确保我们定义的构建模块是没有冗余的。|
|8|$(INSTALLED_ANDROID_INFO_TXT_TARGET)|该目标会生成一个关于当前 Build 配置的设备信息的文件，该文件的生成路径是：`out/target/product/<product_name>/android-info.txt`|
|9|systemimage|生成 system.img。|
|10|$(INSTALLED_BOOTIMAGE_TARGET)|生成 boot.img。|
|10|$(INSTALLED_RECOVERYIMAGE_TARGET)| 生成 recovery.img。|
|11|$(INSTALLED_USERDATAIMAGE_TARGET)|生成 userdata.img。|
|12|$(INSTALLED_CACHEIMAGE_TARGET)|生成 cache.img。|
|13|$(INSTALLED_FILES_FILE)|该目标会生成`out/target/product/<product_name>/ installed-files.txt`文件，该文件中内容是当前系统镜像中已经安装的文件列表。|

## 2.2其他目标

Build 系统中包含的其他一些 Make 目标说明如表 5 所示：

|序号|目标|说明|
|--|--|--|
|1|make clean|执行清理，等同于：`rm -rf out/`|
|2|make sdk|编译出 Android 的 SDK。|
|3|make clean-sdk|清理 SDK 的编译产物。|
|4|make update-api|更新 API。在 framework API 改动之后，需要首先执行该命令来更新 API，公开的API记录在 `frameworks/base/api`目录下|
|5|make dist|执行 Build，并将 `MAKECMDGOALS`变量定义的输出文件拷贝到`/out/dist`目录|
|6|make all|编译所有内容，不管当前产品的定义中是否会包含|
|7|make help|帮助信息，显示主要的 make 目标|
|8|make snod|从已经编译出的包快速重建系统镜像|
|9|make libandroid_runtime|编译所有 JNI framework 内容|
|10|**make framework**|编译所有 Java framework 内容|
|11|**make services**|编译系统服务和相关内容|
|12|make `<local_target>`|编译一个指定的模块，local_target 为模块的名称|
|13|make `clean-<local_target>`|清理一个指定模块的编译结果|
|14|make dump-products|显示所有产品的编译配置信息，例如：`产品名`，`产品支持的地区语言`，`产品中会包含的模块`等信息|
|15|make PRODUCT-xxx-yyy|编译某个指定的产品。|
|16|make bootimage|生成 boot.img|
|17|make recoveryimage|生成 recovery.img|
|18|make userdataimage|生成 userdata.img|
|19|make cacheimage|生成 cache.img|

# 3.在 Build 系统中添加新的产品
当我们要开发一款新的 Android 产品的时候，我们首先就需要在 Build 系统中添加对于该产品的定义。

在 Android Build 系统中对产品定义的文件通常位于 device 目录下（另外还有一个可以定义产品的目录是 vender 目录，这是个历史遗留目录，Google 已经建议不要在该目录中进行定义，而应当选择 device 目录）。device 目录下根据公司名以及产品名分为二级目录.

通常，对于一个产品的定义通常至少会包括四个文件：
* 1.AndroidProducts.mk:该文文件中的内容很简单，其中只需要定义一个变量，名称为“PRODUCT_MAKEFILES”，该变量的值为产品版本定义文件名的列表，例如：

```makefile
PRODUCT_MAKEFILES := \ 
$(LOCAL_DIR)/full_stingray.mk \ 
$(LOCAL_DIR)/stingray_emu.mk \ 
$(LOCAL_DIR)/generic_stingray.mk
```
* 2.产品版本定义文件:顾名思义，该文件中包含了对于特定产品版本的定义。该文件可能不只一个，因为同一个产品可能会有多种版本（例如，面向中国地区一个版本，面向美国地区一个版本）。该文件中可以定义的变量以及含义说明如表 6 所示：

|序号|常量|说明|
|--|--|--|
|1|PRODUCT_NAME|最终用户将看到的完整产品名，会出现在**关于手机**信息中|
|2|PRODUCT_MODEL|产品的型号，这也是最终用户将看到的|
|3|PRODUCT_LOCALES|该产品支持的地区，以空格分格，例如：en_GB de_DE es_ES fr_CA|
|4|PRODUCT_PACKAGES|该产品版本中包含的 APK 应用程序，以空格分格，例如：Calendar Contacts|
|5|PRODUCT_DEVICE|该产品的工业设计的名称|
|6|PRODUCT_MANUFACTURER|制造商的名称|
|7|PRODUCT_BRAND|该产品专门定义的商标（如果有的)|
|8|PRODUCT_PROPERTY_OVERRIDES|对于商品属性的定义|
|9|PRODUCT_COPY_FILES|编译该产品时需要拷贝的文件，以**源路径:目标路径**的形式|
|10|PRODUCT_OTA_PUBLIC_KEYS|对于该产品的 OTA 公开 key 的列表|
|11|PRODUCT_POLICY|产品使用的策略|
|12|PRODUCT_PACKAGE_OVERLAYS|指出是否要使用默认的资源或添加产品特定定义来覆盖|
|13|PRODUCT_CONTRIBUTORS_FILE|HTML 文件，其中包含项目的贡献者|
|14|PRODUCT_TAGS|该产品的标签，以空格分格|

通常情况下，我们并不需要定义所有这些变量。Build 系统的已经预先定义好了一些组合，它们都位于`/build/target/product`下，每个文件定义了一个组合，我们只要继承这些预置的定义，然后再覆盖自己想要的变量定义即可。例如：

```makefile
# 继承 full_base.mk 文件中的定义
$(call inherit-product, $(SRC_TARGET_DIR)/product/full_base.mk) 
# 覆盖其中已经定义的一些变量
PRODUCT_NAME := full_lt26 
PRODUCT_DEVICE := lt26 
PRODUCT_BRAND := Android 
PRODUCT_MODEL := Full Android on LT26
```

* 3.BoardConfig.mk:该文件用来配置硬件主板，它其中定义的都是设备底层的硬件特性。例如：该设备的主板相关信息，Wifi 相关信息，还有 bootloader，内核，radioimage 等信息。对于该文件的示例，请参看 Android 源码树已经有的文件。
* 4.verndorsetup.sh。该文件中作用是通过 add_lunch_combo 函数在 lunch 函数中添加一个菜单选项。该函数的参数是产品名称加上编译类型，中间以“-”连接，例如：`add_lunch_combo full_lt26-userdebug`。`/build/envsetup.sh`会扫描所有`device`和`vender`二 级目 录下的名称 为`vendorsetup.sh`文件，并根据其中的内容来确定`lunch`函数的 菜单选项。

在配置了以上的文件之后，便可以编译出我们新添加的设备的系统镜像了。

首先，调用“source build/envsetup.sh”该命令的输出中会看到 Build 系统已经引入了刚刚添加的 vendorsetup.sh 文件。

然后再调用“lunch”函数，该函数输出的列表中将包含新添加的 vendorsetup.sh 中添加的条目。然后通过编号或名称选择即可。

最后，调用“make -j8”来执行编译即可。

# 4.添加新的模块
在源码树中，一个模块的所有文件通常都位于同一个文件夹中。为了将当前模块添加到整个 Build 系统中，每个模块都需要一个专门的 Make 文件，该文件的名称为“Android.mk”。Build 系统会扫描名称为“Android.mk”的文件，并根据该文件中内容编译出相应的产物。

**需要注意的是**：在 Android Build 系统中，编译是以`模块`（而不是文件）作为单位的，**每个模块都有一个唯一的名称**，**一个模块的依赖对象只能是另外一个模块**，而不能是其他类型的对象。对于已经编译好的二进制库，`如果要用来被当作是依赖对象，那么应当将这些已经编译好的库作为单独的模块`。对于这些已经编译好的库使用`BUILD_PREBUILT`或 `BUILD_MULTI_PREBUILT`。例如：当编译某个 Java 库需要依赖一些 Jar 包时，`并不能直接指定 Jar 包的路径作为依赖`，`而必须首先将这些 Jar 包定义为一个模块`，然后在编译 Java 库的时候通过模块的名称来依赖这些 Jar 包。

下面，我们就来讲解 Android.mk 文件的编写：

```makefile
#设置当前模块的编译路径为当前文件夹路径
LOCAL_PATH := $(call my-dir) 
#清理（可能由其他模块设置过的）编译环境中用到的变量
include $(CLEAR_VARS)
```

为了方便模块的编译，Build 系统设置了很多的编译环境变量。要编译一个模块，只要在编译之前根据需要设置这些变量然后执行编译即可。它们包括：

* `LOCAL_SRC_FILES`：当前模块包含的所有源代码文件
* `LOCAL_MODULE`：当前模块的名称，这个名称应当是唯一的，模块间的依赖关系就是通过这个名称来引用的。
* `LOCAL_C_INCLUDES`：C 或 C++ 语言需要的头文件的路径。
* `LOCAL_STATIC_LIBRARIES`：当前模块在`静态链接`时需要的库的名称
* `LOCAL_SHARED_LIBRARIES`：当前模块在`运行时`依赖的动态库的名称
* `LOCAL_CFLAGS`：提供给 C/C++ 编译器的额外编译参数
* `LOCAL_JAVA_LIBRARIES`：当前模块依赖的 Java 共享库
* `LOCAL_STATIC_JAVA_LIBRARIES`：当前模块依赖的 Java 静态库
* `LOCAL_PACKAGE_NAME`：当前 APK 应用的名称
* `LOCAL_CERTIFICATE`：签署当前应用的证书名称
* `LOCAL_MODULE_TAGS`：当前模块所包含的标签，一个模块可以包含多个标签。标签的值可能是 `debug`, `eng`, `user`，`development` 或者 `optional`。其中，optional 是默认标签。标签是提供给编译类型使用的。不同的编译类型会安装包含不同标签的模块.

|序号|名称|说明|
|--|--|--|
|1|eng|默认类型，该编译类型适用于开发阶段。当选择这种类型时，编译结果将：1.安装包含 eng, debug, user，development 标签的模块，2.安装所有没有标签的非 APK 模块，3.安装所有产品定义文件中指定的 APK 模块|
|2|user|该编译类型适合用于最终发布阶段。当选择这种类型时，编译结果将：1.安装所有带有user标签的模块2.安装所有没有标签的非 APK 模块3.安装所有产品定义文件中指定的 APK 模块，APK 模块的标签将被忽略|
|3|userdebug|该编译类型适合用于debug 阶段。该类型和 user 一样，除了：会安装包含 debug 标签的模块编译出的系统具有 root 访问权限|

Build 系统中还定义了一些便捷的函数以便在 Android.mk 中使用，包括：

* `$(call my-dir)`：获取当前文件夹路径。
* `$(call all-java-files-under, <src>)`：获取指定目录下的所有 Java 文件。
* `$(call all-c-files-under, <src>)`：获取指定目录下的所有 C 语言文件。
* `$(call all-Iaidl-files-under, <src>)`：获取指定目录下的所有 AIDL 文件
* `$(call all-makefiles-under, <folder>)`：获取指定目录下的所有 Make 文件
* `$(call intermediates-dir-for, <class>, <app_name>, <host or target>, <common?> )`：获取 Build 输出的目标文件夹路径。

## 4.1 编译一个 APK 文件

```makefile
LOCAL_PATH := $(call my-dir) 
include $(CLEAR_VARS) 
# 获取所有子目录中的 Java 文件
LOCAL_SRC_FILES := $(call all-subdir-java-files)          
# 当前模块依赖的静态 Java 库，如果有多个以空格分隔
LOCAL_STATIC_JAVA_LIBRARIES := static-library 
# 当前模块的名称
LOCAL_PACKAGE_NAME := LocalPackage 
# 编译 APK 文件
include $(BUILD_PACKAGE)
```

## 4.2 编译一个 Java 的静态库

```makefile
LOCAL_PATH := $(call my-dir) 
include $(CLEAR_VARS) 
  
# 获取所有子目录中的 Java 文件
LOCAL_SRC_FILES := $(call all-subdir-java-files) 
  
# 当前模块依赖的动态 Java 库名称
LOCAL_JAVA_LIBRARIES := android.test.runner 
  
# 当前模块的名称
LOCAL_MODULE := sample 
  
# 将当前模块编译成一个静态的 Java 库
include $(BUILD_STATIC_JAVA_LIBRARY)
```