---
title: gradle-入门
date: 2018-08-03 09:06:31
tags: Android脚本
categories: Android
---

# 0 介绍

Gradle是一个构建系统工具，DSL基于Groovy实现。Gradle构建大部分通过插件方式来实现，内置插件不能满足，可以自己自定义插件。DSL（Domain Specific Language）领域特定语言。java是一种通用语言。软件开发大师Martin Flower<<领域特定语言>>介绍。
<!--more-->

## 1.Gradle简介
## 1.1 Hello world
建立文件build.gradle，内容为：

```
//build.gradle
task hello{
	doLast{
		println 'hello world'
	}
}
```

终端在该文件目录下执行,gradle未指定文件`-d`默认执行文件是build.gradle：

```
$gradle -q hello
输出：hello world
```

* 其中hello是一个task
* doLast是一个Action(task会执行会完毕，回调doLast)
* `-q`控制gradle输出日志级别。

## 1.2Graddle Wrapper
`Wrapper`就是对Graddle进行一层包装，便于统一Gradle构建的版本，避免版本不同冲突。项目开发都是以wrapper方式开发的(windows 下面批处理脚本，linux就是一个shell)。

当Wapper启动Graddle时候，Wapper会检查Gradle有么有被下载关联，没有下载，则从配置地址从官方库下载(一般是Gradle官方库)，这样不需要手动配置，很方便。wapper需要添加到git中的，这样统一开发环境配置。

### 1.2.1生成Wapper
Gradle提供了一个`Wapper task`帮助生成Wapper所需要目录文件。(利用task来建立，这个task叫wapper).`$graddle wrapper`

生成目录如下：

![wapper目录](gradle1/01.png)

* gradlew和gradlew.bat分别是Linux和Window脚本，和gradle一样。
* gradle-wrapper.jar具体业务逻辑实现的jar包，gradlew最终使用java执行这个jar包执行相关操作。
* graddle-wrapper.properties是配置文件，用于配置哪个版本的Gradle等。
	* 其中distributionUrl是Gradle发行版压缩包的下载地址(看能否下载，避免卡主)

### 1.2.2自定义Wapper

指定自己需要的一些信息：

```
//继承Wrapper
task wrapper(type:Wrapper){
	gradleVersion='2.4'
	distributionUrl=""
}
```

## 1.3日志

日志也分几个级别：ERROR,QUIET,WARNING,LIFECYCLE,INFO,DEBUG.可以通过命令行来控制输出级别：

```
//输出QUITE级别及之上日志信息-i -d
$gradle -q tasks
```

打开堆栈信息可通过：`-s`输出关键字堆栈信息。`-S`输出所有的堆栈信息。

使用日志调试信息。Project的getLogger()方法获得的Logger对象实例logger来输出：

```
logger.quiet();
logger.error();
logger.warn();
logger.lifecycle();
logger.info();
logger.debug();
```

## 1.4命令行
AndrodStudio中双击Task就可以执行了。

```
//帮助
./gradlew ? 
./gradlew -h
./gradlew -help
//查看所有可执行的task
./gradlew tasks

//help task
./gradlew help --task

//强制刷新依赖
./gradlew --refresh-dependencies assemable

//执行多任务.任务之间空格隔开即可
./gradlew clean jar
```

**通过任务名字缩写执行**任务用驼峰书写：比如connectCheck，执行时使用`./gradlew connectCheck`,可简单写成`./graddle cc`

# 2.Groovy基础

Groovy基于JVM虚拟机的一种动态语言，完全兼容java，基础增加很多动态类型和灵活特性，`支持闭包`，支持DSL，**是一个灵活动态脚本语言**.

## 2.1字符串

分号不是必须的，单引号和双引号都可以表示字符串。类型为`java.lang.String`.单引号没有运算能力，双引号有运算能力。

## 2.2集合
常见集合List，Set，Map和Queue。

### 2.2.1List

```
task printList<<{
	def numList=[1,2,3,4,5];
	println numList.getClass().name
	//访问最后一个元素
	println numList[-1]
	//访问2到4
	println numList[1..3]
}
```

List方便的迭代操作，each方法。接受一个闭包作为参数，访问List每个元素：

```
task printList<<{
	numList.each{
		println it;
	}
}
```

it变量是正在迭代的元素。

### 2.2.2Map

```
task printlnMap<<{
	def map1={'width':1024,'height':768}
	println map1.height
	println map1['width']
	map1.each{
		println"Key:${it.key},Value:${it.value}"
	}
}
```

## 2.3.方法(闭包演变)

* 括号可以省略。`invokeMethod(parm1,parm2)`变成`invokeMethod parm2,parm2`
* return可以不写。当没有return，最后一句作为返回值。
* 代码块作为参数传递。

代码块(**一段被花括号包围的代码**)，其实就是`闭包`。

```
//呆板写法其实就是
numList.each({println it})

//格式化一下代码
numList.each({
	println it
})

//Groovy规定，如果方法最后一个参数是闭包，可以将放到方法外面
numList.each(){
	println
}
//然后方法可以省略变成这个样子
numList.each{
	println it
}
```

## 2.4 JavaBean
JavaBean不得不使用setter/getter方法，使用繁琐。Groovy很大改善。

```
task helloJavaBean{
	Person p=new Persion();
	println "名字是 ${p.name}"
	p.name="张三"
	println "名字是 ${p.name}"
}
class Person{
	private String name
	public int getAge(){
		12
	}
}
```
没赋值输出是null，赋值后输出为张三。

## 2.5闭包
### 2.5.1初始闭包

`it`变量的由来.

```
task helloClosure<<{
	//使用我们自定义的闭包
	customEach{
		println it
	}
}
def customEach(closure){
	//模拟一个有10个元素集合
	for(int i in 1..10){
		closure(i)
	}
}
```
如果close只有一个参数就是我们的`it`变量。

### 2.5.2向闭包传递参数

当一个参数默认参数就是it，当多个参数时，it不能表示了，我们需要一一列举出来

```
task helloClosure<<{
	//多个参数
	eachMap{k,v->
		println "${k} is ${v}"
	}
}
def eachMap(closure){
	def map1={'name':"张三",'age':"18"}
	map1.each{
		//两个参数，为一个map。则相当于一个参数
		closure{it.key,it.value}
	}
}
```

### 2.5.3闭包委托

Groovy闭包强大在于支持闭包方法的委托。Groovy闭包有`thisObject`，`owner`，`delegate`三个属性。当在闭包调用的方法时候，用来决定哪个对象来处理，默认情况下delegate和owner是相等的，但是delage是可以被修改的。闭包很多功能通过修改delegate实现的

```
task helloDelegate<<{
	new Delegate().test{
		println "${thisObeject.getClass()}"
		println "${owner.getClass()}"
		println "${delegate.getClass()}"
		method1()
		it.methods1()
	}
}
def method1(){
	println "${this.getClass()} in root"
	println "method1 in root"
}
class Delegate{
	def method1(){
		println "delegate this ${this.getClass()}"
	}
	def test(Closure<Delegate> closure){
		closure(this)
	}
}
```
闭包内方法处理顺序是:`thisObject`>`owner`>`delegate`.

在DSL，中一般指定delegate为当前的it，闭包内就可以对该it进行配置：

```
task configClosure<<{
	person{
		personName="张三"
		personAge=20
		dumpPerson()
	}
}
class person{
	String personName
	int personAge
	def dumpPerson(){
		println " name:${personName}"
	}
	def person(Closure<Person> closure){
		Person p=new Person()
		closure.delegate=p
		//委托模式优先
		closure.setResolveStragegy(Closure.DELEGATE_FIRST)
		closure(p)
	}
}
```
委托对象为当前创建的Person实例，设置委托模式优先。在使用person方法创建一个Person实例时，闭包里可以直接对Person实例配置。我们Gralde中task创建一个Task用法很像。

Gradle也基本使用delegate方式使用闭包进行配置和操作。

# 3.Gradle构建脚本
## 3.1Setting文件
默认名字`settings.gradle`是一个设置文件，初始化以及`工程树的配置`，在根目录中。project和module的概念。一个module只有在settings文件中配置才被gradle构建识别

```
rootProject.name="book"
include ':example02'
project().projectDir=new File(rootDir,'chapter01/example02')
```
* 配置子项目`include ':example02'`
* 指定子项目路径`project().projectDir`，不执行默认是当前目录。

## 3.2Build文件
每个project都有一个build文件(子项目也是)，该文件为`project入口`,可以针对Project进行配置，比如，`配置版本`，`需要哪些插件`，`依赖哪些库`等.

在root Project可以对child Project进行统一配置。比如应用插件，依赖Maven中心库等。可进行下面配置。

```
//配置所有child project仓库为jcenter
subprojects{
	repositories{
		jcenter()
	}
}
```