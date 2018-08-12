---
title: gradle任务和插件
date: 2017-08-04 09:59:34
tags: Android脚本
categories: Android
---
多种任务创建方式，如何访问任务的方法和属性等信息，对任务进行分组、排序，以及任务一些规则性知识。

<!--more-->
# 1.gradle任务
## 1.1多种创建方式
各种方式都依赖于Project提供快捷方法以及TaskContainer提供相关的create方法。

```
//第一种任务名方式创建任务
def Task createTask1=task(createTask1) //task(一个对象？)

createTask1.doLast{
	println "原型：Task task(String name)"
}
//第二种任务名+配置该任务Map对象创建
def task createTask2=task(createTask,group:BasePlugin.BUILD_GROUP)
createTask2.doLast {
	println "原型：Task task(Map<String,?>args,String name)"
}
//第三种，闭包方式创建
task createTask3{
	description '演示任务创建'
	doLast{
		println "任务描述${description}"
	}
}
//第四种
tasks.create('createTask4'){
	description '演示任务创建'
	doLast{
		println "任务描述${description}"
	}
}
```

第二种方式Map起到配置作用：

Task参数Map的可用配置

|配置项|描述|默认值|
|--|--|--|
|type|基于存在Task创建，类似继承|DefaultTask|
|override|是否替换存在Task，和type结合使用|false|
|dependsOn|用于配置任务的依赖|[]|
|action|添加任务中一个Action或者一个闭包|null|
|description|配置任务的描述|null|
|group|配置任务分组|null|

## 1.2多种访问的方式

1.将任务分配给一个变量，然后用该变量进行操作。任务容器可以这样访问：`tasks['createTask4'].doLast{}`

## 1.3任务分组和描述

任务分组可以更明确分类任务.`myTask.group=BasePlugin.BUILD_GROUP`,这样在查看`gradle tasks`更清晰。

描述，`description`前面已经说明过。

## 1.4 `<<`操作符

`<<`操作符在Gradle的Task是doLast方法短标记形式。

```
task(hello)<<{
	println 'helloTask do last'
}
task(hello).doLast{
	println 'hello doLast'
}
```

## 1.5任务执行

Task执行任务可分为：Task之前执行(doFirst)、Task本身执行(doSelf)以及Task之后执行(doLast)

## 1.6任务排序

通过设置任务的`shouldRunAfter`和`mustRunAfter`两个方法。有用的，比如先执行单元测试，再执行打包，保证APP质量。

* taskB.shouldRunAfter(taskA)应该B是在A之后执行，但不一定。
* taskB.mustRunAfter(taskA)必须这样顺序。

## 1.7任务启动和禁用

Task的enable属性控制，默认为true。

```
task hello2{
}
hello2.enabled=false
```

## 1.8任务onlyIf断言
断言就是一个条件表达式。接受一个闭包作为参数，闭包返回true则该任务执行。

```
final String BUILD_APPS_ALL="all"
final String BUILD_APPS_SHOUFA ="shoufa"
final String BUILD_APPS_EXCLUDE_SHOUFA="exclude_shoufa"
task release1<<{
	println "release1"
}
task release2<<{
	println "release2"
}
task release3<<{
	println "release3"
}
task build{
	group BasePlugin.BUILD_GROUP
	description "打包渠道"
}
build.dependsOn release1 release2 release3 
release1.onlyIf{
	def execute =false
	if(project.hasProperty("build_apps")){
		Object buildApps=project.property("build_apps")
		if(BUILD_APPS_SHOUFA.equals(buildApps)||BUILD_APPS_ALL.equals(buildApps)){
			execute=true
		}else{
			execute=false
		}
		execute
	}
}
//每个release都可以这样判断
```

通过`build_apps`属性来控制打包渠道包：

1.打包所有渠道包

```
./gradlew -Pbuild_apps=all :example48:build
```
命令行中`-P`意思为project指定属性值。

# 2.Gradle插件
Gradle本身提供一个基本概念和整体核心的框架，其他用于描述使用真实场景逻辑的都以插件扩展的方式实现。

## 2.1插件的作用

插件应用到项目中，会扩展很多功能。

* 1.可以添加任务到你项目中完成比如测试、编译、打包
* 2.可以添加依赖配置到项目中，比如第三方库等
* 3.向项目中添加扩展属性、方法等。比如android{}配置块就是AndroidGradle插件一个扩展。
* 4.对项目进行进一步约束。约定`src/main/java`目录下是我们源码存放位置。

## 2.2应用一个插件

插件应用都是通过`Project.apply()`方法完成的，apply方法有几种，插件也分为二进制插件和脚本插件。

### 2.2.1二进制插件
二进制插件实现了`org.gralde.api.Plugin`接口插件，可以有`plugin id`。

```
//应用一个java插件,其中java就是plugin id(唯一的)
apply plugin:'java'

//java是一个短名称，对应类型org.gradle.api.plugins.JavaPlugin
apply plugin:org.gradle.api.Plugin.JavaPlugin

//一般org.gradle.api.Plugins默认的导入
apply plugin:JavaPlugin
````
二进制插件一般打包到一个jar包独立发布的，比如我们自定义插件，plugin id最好用权限定名称，这个plugin id不会重复

### 2.2.2应用脚本插件

**就是把另一个脚本文件加载过来**。from可以是一个脚本文件或者HTTP URL。

```
//build.gradle文件
apply from:'version.gradle'
task helloT<<{
	println "${versionName}"
}

//version.gradle文件
ext {
	versionName ='1.0.0'
}
```

### 2.2.3 apply其他用法

```
//闭包形式
apply {
	plugin 'java'
}

//Action方式
apply(new Action<ObjectConfigurationAction>(){
	@override
	void execute(ObjectConfigurationActio action){
		objectConfigureAction.plugin('java')
	}
})
```

## 2.3使用第三方插件

第三方发布的作为jar的二进制插件，必须在`buildscript{}`里配置其classpath才能使用，这个不像Gradle提供内置插件。

```
//Android Gradle插件
buildscript{
	respositories {
		jcenter()
	}
	dependncies{
		classpath 'com.android.tools.build:gradle:1.5.0'
	}
}
```

`buildscript{}`块是构建项目之前，为项目进行前期准备和初始化相关配置依赖地方，配置所需要依赖，就可以使用应用插件了`apply plugin: 'com.android.application'`

## 2.4自定义插件
自定义插件涉及知识点很多，比如创建任务、创建方法、进行约定等。
下面介绍创建一个任务为例，进行自定义插件介绍。

```
apply plugin:MyPlugin
class MyPlugin implements Plugin<Project>{
	void apply(Project project){
		project.task('myTask')<<{
			println "自定义插件创建的任务"
		}
	}
}
```

现在可以执行`./gralew myTask`来执行任务(这个任务通过自定义插件创建的)

## 2.5发布插件

1.创建一个Groovy工程，然后配置插件开发所需的依赖：

```
apply plugin:'groovy'
dependencies {
	compile gradleApi()
	compile localGroovy()
}
```

2.然后实现插件类

```
package com.githubb.rujews.plugins

import org.gradle.api.Plugin
import org.gradle.api.Project

class MyPlugin implements Plugin<Project>{
	@override
	void apply(Project target){
		target.task('myTask')<<{
			println "这是自定义插件创建的任务"
		}
	}
}
```

Gradle通过META-INF里properties文件来发现对应插件实现类的。首先确定一个在src/main/resource/META-INF/gradle-plugins/目录先建立properties文件`com.github.rujews.plugins.myTask.properties`添加一行内容`implementation-class=com.github.rujews.plugins.MyTask`


配置好久可以给其他人使用了。