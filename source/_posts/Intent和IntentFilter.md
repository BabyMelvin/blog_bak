---
title: Intent和IntentFilter
date: 2018-08-15 21:24:17
tags: Android组件
categories: Android
---

`Intent` 是一个消息传递对象，您可以使用它从其他应用组件请求操作。尽管 Intent 可以通过多种方式促进组件之间的通信.

<!--more-->

用法主要三个分类：

* **启动Activity**：启动Activity实例,并携带数据。两种启动方式。
	* `startActivity()`
	* `startActivityForResult()`，在主Activity的`onActivityResult()`回调返回结果。

* **启动服务**：Service不是用界面后台执行的组件。启动方式有：
	* `startService`:交给服务自己处理，一次性操作（下载文件，等）
	* `bindService()`绑定服务，交互操作。

* **传递广播**：广播是任何应用均可接受的消息。有几种类型的广播：
	* `sendBroadcast()`无序的
	* `sendOrderedBroadcast()`有序的广播
	* `sendStickyBroadcast()`保持广播（任何人都能够接收的广播）

# 1.Intent类型
Intent分为两种类型：

* **显示Intent**：按名称(完全限定类名)指定要启动的组件。通常自己应用中使用显示Intent来启动组件，应为自己知道类名。
	* 创建显示Intent启动Activity或服务时，系统立即启动指定的组件
* **隐式Intent**:不指定特定的组件，而是声明要执行的操作。如：显示用户位置，可以使用隐式Intent，请求具有此功能地图显示指定的位置。
	* 隐式启动和mainfest中的IntentFilter进行过滤。

# 2.构建Intent
Intent中包含主要信息如下：

* **ComponentName**。有是**显示**，没有则是**隐式**。Intent中字段`ComponentName`可以使用`setComponent()`,`setClass()`和`setClassName()`或Intent构造函数设置组件名称。

* **Action**：通用想要的动作使用`setAction()`或者Intent构造函数，如显示view或选择。这个动作最终取决于data和extras的内容。启动Intent一些常量，可以通用使用。如启动一个Activity:
	* `ACTION_VIEW`，字符串常量内容为`android.intent.action.VIEW`.结合`startActivity`能够启动activity显示，如图片或者地图信息。
	* `ACTION_SEND`,可以在email或者一个社交媒体分享。

* **data**:Uri或者是MIME类型。要什么样data类型由action来决定。一般都要指定，系统知道设么样的类型数据。`setData()`和`setType`，都有就设置`setDataAndType()`
* **category**:指定intent组件的类型。`addCategory()`大部分不需要category.
	* `CATEGORY_BROWSABLE`
	* `CATEGORY_LAUNCHER`：点击图标启动的activity
	* `CATEGORY_HOME`:桌面应用。

Intent上面提到的，决定Intent功能。可以携带一些其它的信息，用**Extras**来实现携带。

* **Extras**:Key-value形式。`putExtra()`
* **Flags**:**怎样启动activity**(activity task)和**启动后怎么处理**(是否属于最近的activity列表)。`setFlags()`

## 2.1 启动例子

**显示启动**

```java
//fileUrl:如：http://www.baidu.com/hello.png
Intent downloadIntent=new Intent(this,DownloadService.class);
downloadIntent.setData(Uri.parse(fileUrl));
startService(downloadIntent);
```

**隐式启动**

```java
//create the text message with a string
Intent sendIntent=new Intent();
sendIntent.setAction(Intent.ACTION_SEND);
sendIntent.putExtra(Intent.EXTRA_TEXT,textMessage);
sendIntent.setType("text/plain");

//verify that the intent will resolver to an activity
if(sendIntent.resolveActivity(getPackageManager()!=null){
	startActivity(sendIntent);
}
```

**强制app选择器**：默认动作是用户可以选择框，可以选择默认的app。但是如果想要每次都可选多个app(不去默认)。应该下面这样使用：

```java
Intent sendIntent=new Intent(Intent.ACTION_SEND);

//Always use string resource for UI text
//this says something like "Share this photo with"
String title=getResources().getString(R.string.chooser_title);
//create intent to show the choose dialog
Intent chooser=Intent.createChooser(sendIntent,title);

//verify the orignal intent will resolve to at least one activity
if(sendIntent.resolveActivity(getPackageManager())!=null){
	startActivity(chooser);
}
```

# 3.IntentFilter
接收隐式Intent。在mainfest中声明`<intent-filter>`，每个组件可以声明**一个或多个**。每个intent-filter对应具体的intent。只有通过intent filter才能使用提供的组件。

**注意**：显示的intent总是能够启动目标，忽略组件中的intent filters。

`<intent-filter>`中使用**一个或多个**下面三个元素：

* `<action>`：action的值。可以声明0个或多个该元素。多个必须匹配其中一个action。
	* 如果filter中没有action，intent没有匹配，所有intent都失败。
	* 但是如果Intent没有指定action，只要filter包含action，就通过测试。
* `<data>`：0个或多个属性。(sheme,host,port,path)和MIME type。
	* URI格式`<scheme>://<host>:<port>/<path>`,如`content://com.example.project:200/folder/subfolder/etc`
		* 如果没有scheme，host会被忽略
		* 如果host没有，port会被忽略
		* 如果scheme和host都没有path会被忽略
	* 对intent的URI和filter中URI比较，对比较filter中提供的URI部分，比如：
		* 如果filter只有scheme，匹配所有该scheme的intent
		* 提供哪里，配置哪里就可以了。
	* URI和MIME的intent，规则：
		* Intent中都两个都没有，值匹配也都没有的filter.
		* intent只有一个(排除显示的，和继承URI)，filter也只有提供的那个。
		* 两个都提供，filter也要都提供。如果过滤器仅列出MIME类型，则假定组件支持`content:`和`file:`数据。组件能够从本地获得文件或者contentprovider。因此，只列出data type不需要明确scheme(content,file).

```html
<!--从content provider中获取图片并显示,不需要sheme-->
<intent-filter>
	<data android:mimeType="image/*"/>
</intent-filter>

<!--一个scheme和一个data type-->
<intent-filter>
	<data android:scheme="http" android:mimeType="video/*"/>
</intent-filter>
```
* `<category>`:category名称。可以包含0个或多个category。
	* intent没有category，总是能够匹配过滤。

**注意**：Android默认对隐式`startActivity()`和`startActivityForResult()`添加了`CATEGORY_DEFAULT`选项，如果你的activity想使用隐式intent，一定要声明`CATEGORY_DEFAULT`。

## 3.1 例子
一个activity声明接收`ACTION_SEND`,data 类型是text.

```html
<!--可以使用exported=false限制访问-->
<activity android:name="ShareActivity">
	<intent-filter>
		<action android:name="android.intent.action.SEND"/>
		<category android:name="android.intent.category.DEFAULT"/>
		<data android:mimeType="text/plain"/>
	</intent-filter>
</activity>
```

**注意**：避免无意运行一个service，应该总是显示声明你自己的service，不用对自己的service添加intent filter。

```
<activity android:name="MainActivity">
	<!-- main entry,appear in app launcher -->
	<intent-filter>
		<action android:name="android.intent.action.MAIN"/>
		<category android:name="android.intent.category.LAUNCHER"/>
	</intent-filter>
</activity>
<activity android:name="ShareActivity">
	<!--handle "SEND" actions with text data -->
	<intent-filter>
		<action android:name="android.intent.action.SEND"/>
		<category android:name="android.intent.category.DEFAULT"/>
		<data android:mimeType="text/plain"/>
	</intent-filter>
	<!-- handle "send" and "send_multiple" with media data-->
	<intent-filter>
		<action android:name="android.intent.action.SEND"/>
		<action android:name="android.intent.action.SEND_MULTIPLE"/>
		<category android:name="android.intent.category.DEFAULT"/>
		<data android:mimeType="application/vnd.google.panoramsa360+jgp"/>
		<data andriod:mimeType="image/*"/>
		<data android:mimeType="video/*"/>
	</intent-filter>
</activity>
```

`MainActivity`app主入口，用launcher图标启动的activity。

* `ACTION_MAIN`表明主入口，不需要data
* `CATEGORY_LAUNHCER`这个activity的图标应该放在系统app的launcher。如果activity没有提供图标，应该从`<application>`中获得。

# 4.pending intent使用

`Pending intent`目的是授予别的app使用intent就好像自己的app使用自己进程一样。