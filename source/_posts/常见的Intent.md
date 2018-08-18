---
title: 常见的Intent
date: 2018-08-17 21:10:17
tags: Android组件
categories: Android
---
Intent启动另外一个应用的activity，如Intent和Intent filter使用说明中。许多系统应用已经设置了内部的隐式Intent。

<!--more-->

# 1.闹钟(alarm clock)

* action：`ACTION_SET_ALARM`
* extras:
	* `EXTRA_HOUR`:闹钟小时数
	* `EXTRA_MINUTES`:闹钟分钟数
	* `EXTRA_MESSAGE`:闹钟的说明信息
	* `EXTRA_DAYS`：每周哪几天重复Calender中的数据，组成一个ArrayList.
	* `EXTRA_RINGTONE`:提供铃声的URI，或者没有铃声`VALUE_RINGTONE_SILENT`
	* `EXTRA_VIBRATE`:是否振动，布尔值。
	* `EXTRA_SKIP_UI`:是否跳过响应app的UI。

**使用说明**
```java
public void createAlarm(String message,int hour,int minutes){
	Intent intent=new Intent(AlarmClock.ACTION_SET_ALARM)
		.putExtra(AlarmClock.EXTRA_MESSAGE,message)
		.putExtra(AlarmClock.EXTRA_HOUR,hour)
		.putExtra(AlarmClock.EXTRA_MINUTES,minutes);
	if(intent.resolveActivity(getPackageManager())!=null){
		startActivity(intent);
	}
}
//需要声明权限
<uses-permission android:name="com.android.alarm.permission.SET_ALARM" />
```

响应app的xml过滤：

```
<activity ...>
    <intent-filter>
        <action android:name="android.intent.action.SET_ALARM" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

# 2.创建计时器(timer)

* action:`ACTION_SET_TIMER`
* extras:
	* `EXTRA_LENGTH`:计时器秒数
	* `EXTRA_MESSAGE`:计时器定制消息
	* `EXTRA_SKIP_UI`:是否显示目标UI

**使用说明**
```
public void startTimer(String message, int seconds) {
    Intent intent = new Intent(AlarmClock.ACTION_SET_TIMER)
            .putExtra(AlarmClock.EXTRA_MESSAGE, message)
            .putExtra(AlarmClock.EXTRA_LENGTH, seconds)
            .putExtra(AlarmClock.EXTRA_SKIP_UI, true);
    if (intent.resolveActivity(getPackageManager()) != null) {
        startActivity(intent);
    }
}
//权限
<uses-permission android:name="com.android.alarm.permission.SET_ALARM" />
```

**响应app**

```xml
<activity ...>
    <intent-filter>
        <action android:name="android.intent.action.SET_TIMER" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

# 3.显示所有闹钟

4.4之后添加。 这个需要闹钟功能的app同时提供完成。

* action:`ACTION_SHOW_ALARMS`

**响应app**

```
<activity ...>
    <intent-filter>
        <action android:name="android.intent.action.SHOW_ALARMS" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

# 4.日历（calender）

添加日历时间既要action，也要提供URI。
* action：`ACTION_INSERT`
* data uri:`Events.CONTENT_URI`
* mimetype:`vnd.android.cursor.dir/event`
* extras:
	* `EXTRA_EVENT_ALL_DAY`:是否是全天事件
	* `EXTRA_EVENT_BEGIN_TIME`：事件开始的时间(1970以后毫秒数)
	* `EXTRA_EVENT_END_TIME`:事件结束的时间(1970以后毫秒数)
	* `TITLE`:事件标题
	* `DESCRIPTION`:事件描述
	* `EVENT_LOCATION`:事件地点
	* `EVENT_EMAIL`:逗号分开邮箱地址

**使用说明**

```
public void addEvent(String title, String location, long begin, long end) {
    Intent intent = new Intent(Intent.ACTION_INSERT)
            .setData(Events.CONTENT_URI)
            .putExtra(Events.TITLE, title)
            .putExtra(Events.EVENT_LOCATION, location)
            .putExtra(CalendarContract.EXTRA_EVENT_BEGIN_TIME, begin)
            .putExtra(CalendarContract.EXTRA_EVENT_END_TIME, end);
    if (intent.resolveActivity(getPackageManager()) != null) {
        startActivity(intent);
    }
}
```

**响应app**

```
<activity ...>
    <intent-filter>
        <action android:name="android.intent.action.INSERT" />
        <data android:mimeType="vnd.android.cursor.dir/event" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

# 5.相机
## 5.1 捕获图片或录像并返回

* action:
	* `ACTION_IMAGE_CAPTURE`:捕获图片
	* `ACTION_VIDEO_CAPTURE`:捕获视频

* extras：
	* `EXTRA_OUTPUT`:返回内容的保存URI。
	
当相机应用成功返回到你自己的Activity(`onActivityResult()`回调)，就可以在`EXTRA_OUPUT`路径中找到返回内容。

**使用**

```
static final int REQUEST_IMAGE_CAPTURE=1;
static final Uri mLocationForPhotos;
public void capturePhoto(String targetFilename){
	Intent intent=new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
	intent.putExtra(MediaStore.EXTRA_OUTPUT,Uri.withAppendedPath(mLocaitonForPhotos,targetFilename));
	if(intent.resolveActivity(getPackageManager()) != null){
		startActivityForResult(intent, REQUEST_IMAGE_CAPTURE);
	}
}
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == REQUEST_IMAGE_CAPTURE && resultCode == RESULT_OK) {
        Bitmap thumbnail = data.getParcelable("data");
        // Do other work with full size photo saved in mLocationForPhotos
        ...
    }
}
```

**响应的app**

```
<activity ...>
    <intent-filter>
        <action android:name="android.media.action.IMAGE_CAPTURE" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

响应的app处理intent，首先检查`EXTRA_OUTPUT`,并且调用`setResult()`包含缩略图extra的key是**data**
样例：

* [take photos](https://developer.android.com/training/camera/photobasics)
* [record videos](https://developer.android.com/training/camera/videobasics)

## 5.2 静态图片启动相机

* aciton:`INTENT_ACTION_IMAGE_CAMERA`

**使用**

```
public void capturePhoto() {
    Intent intent = new Intent(MediaStore.INTENT_ACTION_STILL_IMAGE_CAMERA);
    if (intent.resolveActivity(getPackageManager()) != null) {
        startActivityForResult(intent);
    }
}
```

**响应app**


```
<activity ...>
    <intent-filter>
        <action android:name="android.media.action.STILL_IMAGE_CAMERA" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

## 5.3 用视频模式启动相机

* action:`INTENT_ACTION_VIDEO_CAMERA`

**使用**

```
public void capturePhoto() {
    Intent intent = new Intent(MediaStore.INTENT_ACTION_VIDEO_CAMERA);
    if (intent.resolveActivity(getPackageManager()) != null) {
        startActivityForResult(intent);
    }
}
```

**响应**

```
<activity ...>
    <intent-filter>
        <action android:name="android.media.action.VIDEO_CAMERA" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

# 6 联系人
## 6.1 选择一个联系人
通过`onActivityResult()`返回内容(URI).如果没有提供`READ_CONTENT`权限，响应app提供临时权限。
* action：`ACTION_PICK`
* mime type:`Contacts.CONTENT_TYPE`

**使用**
```
static final int REQUEST_SELECT_CONTACT = 1;

public void selectContact() {
    Intent intent = new Intent(Intent.ACTION_PICK);
    intent.setType(ContactsContract.Contacts.CONTENT_TYPE);
    if (intent.resolveActivity(getPackageManager()) != null) {
        startActivityForResult(intent, REQUEST_SELECT_CONTACT);
    }
}

@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == REQUEST_SELECT_CONTACT && resultCode == RESULT_OK) {
        Uri contactUri = data.getData();
        // Do something with the selected contact at contactUri
        ...
    }
}
```

见 [Retrieve details for a contact](https://developer.android.com/training/contacts-provider/retrieve-details)

## 6.2 选择特定联系人
特定联系人如号码，姓名等。

* action:`ACTION_PICK`
* mime type:
	* `CommonDataKinds.Phone.CONTENT_TYPE`:号码选择联系人
	* `CommonDataKinds.Email.CONTENT_TYPE`:邮箱地址选择联系人
	* `CommonDataKinds.StructuredPostal.CONTENT_TYPE`:邮寄地址选择联系人


**使用**

```
static final int REQUEST_SELECT_PHONE_NUMBER=1;
public void selectContact(){
	//start an activity for the user to pick a phone number from contacts
	Intent intent=new Intent(Intent.ACTION_PICK);
	intent.setType(CommonDataKinds.Phone.CONTENT_TYPE);
	if(intent.resolveActivity(getPackageManager())!=null){
		startActivityForResult(intent,REQUEST_SELECT_PHONE_NUMBER);
	}
}

@Override
protected void onActivityResult(int requestCode,int resultCode,Intent data){
	//get the URI and query the content provider for the phone number
	Uri contactUri=data.getData();
	String[] projection=new String[]{CommonDataKinds.Phone.NUMBER};
	Cursor cursor=getContentResolver().query(contactUri,projection,null,null,null);
	//if the cursor returned is valid,get the phone number
	if(cursor!=null && cursor.moveToFirst()){
		int numberIndex=cursor.getColumnIndex(CommonDataKinds.Phone.NUMBER);
		String number=cursor.getString(numberIndex);
		//do something with the phone number
	}
}
```

## 6.3显示contact
两种方式获得contact的URI

* 使用`ACTION_PICK`,前面说的那样(不需要app权限)
* 直接获得联系人所有列表（需要app`READ_CONTACTS`权限）。[Retrieve a list of contacts](https://developer.android.com/training/contacts-provider/retrieve-names)

* action:`ACTION_VIEW`
* uri scheme:`content:<URI>`

**使用**

```
public void viewContact(Uri contactUri) {
    Intent intent = new Intent(Intent.ACTION_VIEW, contactUri);
    if (intent.resolveActivity(getPackageManager()) != null) {
        startActivity(intent);
    }
}
```

## 6.4编辑已有联系人

* action:`ACTION_EDIT`
* URI scheme:`content:<URI>`

```
public void editContact(Uri contactUri, String email) {
    Intent intent = new Intent(Intent.ACTION_EDIT);
    intent.setData(contactUri);
    intent.putExtra(Intents.Insert.EMAIL, email);
    if (intent.resolveActivity(getPackageManager()) != null) {
        startActivity(intent);
    }
}
```

样例：[Modify contacts using intents](https://developer.android.com/training/contacts-provider/modify-data)

## 6.5插入联系人

* action:`ACTION_INSERT`
* mime type:`Contacts.CONTENT_TYPE`

**使用**

```
public void insertContact(String name, String email) {
    Intent intent = new Intent(Intent.ACTION_INSERT);
    intent.setType(Contacts.CONTENT_TYPE);
    intent.putExtra(Intents.Insert.NAME, name);
    intent.putExtra(Intents.Insert.EMAIL, email);
    if (intent.resolveActivity(getPackageManager()) != null) {
        startActivity(intent);
    }
}
```

# 7.邮件

编写邮件很多action，由是都需要附件和一些详细信息决定，如接受者，主题等。

* action:
	* `ACTION_SENDTO`:没有附件
	* `ACTION_SEND`:一个附件
	* `ACTION_SEND_MULTIPLE`:多个附件

* mime type:
	* `text/plain`
	* `*/*`

* extras:
	* `Intent.EXTRA_EMAIL`:所有接受人的地址
	* `Intent.EXTRA_CC`:所有CC接受者
	* `Intent.EXTRA_BCC`:所有BCC接受者
	* `Intent.EXTRA_SUBJECT`:邮件主题
	* `Intent.EXTRA_TEXT`:邮件内容
	* `Intent.EXTRA_STREAM`:附件Uri，如果是多个附件，要提供邮件数组。

**使用**

```
public void composeEmail(String[] addresses, String subject, Uri attachment) {
    Intent intent = new Intent(Intent.ACTION_SEND);
    intent.setType("*/*");
    intent.putExtra(Intent.EXTRA_EMAIL, addresses);
    intent.putExtra(Intent.EXTRA_SUBJECT, subject);
    intent.putExtra(Intent.EXTRA_STREAM, attachment);
    if (intent.resolveActivity(getPackageManager()) != null) {
        startActivity(intent);
    }
}
```

如果保证邮件是被email应用，而不是社交应用使用。使用`ACTION_SENDTO`,包含数组格式(scheme)`mailto:`

```
public void composeEmail(String[] addresses, String subject) {
    Intent intent = new Intent(Intent.ACTION_SENDTO);
    intent.setData(Uri.parse("mailto:")); // only email apps should handle this
    intent.putExtra(Intent.EXTRA_EMAIL, addresses);
    intent.putExtra(Intent.EXTRA_SUBJECT, subject);
    if (intent.resolveActivity(getPackageManager()) != null) {
        startActivity(intent);
    }
}
```

**响应过滤**

```xml
<activity ...>
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <data android:type="*/*" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.SENDTO" />
        <data android:scheme="mailto" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

# 8.文件存储
## 8.1 获得指定类型的文件

使用`ACTION_GET_CONTENT`和特定MIME类型来获得用户选择的文件，并给app返回一个引用。**返回的应用是和当前activity的声明周期同步的**。要之后使用，自己要进行备份。

在`onActivityResult()`返回指向文件的返回的Uri可以是任意类型.Uri，如`http:`,`file:`,`content:`。如果限制只是来至content provider(`content:`)通过`openFileDescriptor()`获得，应该在intent中添加`CATEGORY_OPENABLE`.

4.3以上允许选择多个文件`EXTRA_ALLOW_MULTIPLE`.可以通过`getClipData()`来获得多个文件内容。

* action:`ACTION_GET_CONTENT`
* mime type:和用户选择文件类型相对应
* extras:
	* `EXTRA_ALLOW_MULTIPLE`:是否能够同时选择多个文件。
	* `EXTRA_LOCAL_ONLY`:只是本地文件

* category(可选的)
	* `CATEGORY_OPENABLE`:返回openable文件通过`openFileDescriptor()`来进行表示。

**获得一个图片**

```
static final int REQUEST_IMAGE_GET=1;
public void selectImage(){
	Intent intent=new Intent(Intent.ACTION_GET_CONTENT);
	intent.setType("image/*");
	if(intent.resolveActivity(getPackageManager())!=null){
		startActivityForResult(intent,REQUEST_IMAGE_GET);
	}

}

@Override
protected void onActivityResult(int requestCode,int resultCode,Intent data){
	if(requestCode == REQUEST_IMAGE_GET && resultCode== RESULT_OK){
		Bitmap thumbnail =data.getParcelable("data");
		Uri fullPotoUri=data.getData();

		//do work with photo saved at fullPhotoUri
		...
	}
}
```

**响应返回一个图片的过滤**

```
<activity>
	<intent-filter>
		<action android:name="android.intent.action.GET_CONTENT"/>
		<data android:type="image/*"/>
		<!--The OPENABLE category declares that the returned file is accessible
             from a content provider that supports OpenableColumns
             and ContentResolver.openFileDescriptor()-->
		<category android:name="android.intent.category.OPENABLE"/>
	</intent-filter>
</activity>
```

## 8.2 打开特定类型的文件 

`ACTION_GET_CONTENT`是使用文件内容的副本导入到自己的app。4.4以后可以使用别的app来打开和创建文件，并且由其他应用来管理。使用`ACTION_OPEN_DOCUMENT`和`ACTION_CREATE_DOCUMENT`.

* action:`ACTION_OPEN_DOCUMENT`和`ACTION_CREATE_DOCUMENT`
* extras:
	* `EXTRA_MIME_TYPES`:和文件类型对应的MIME 类型。但是用这个必须初始MIME类型`setType()`值为`*/*`.
	* `EXTRA_ALLOW_MUTIPLE`:是否允许同时选择多个文件
	* `EXTRA_TITLE`:针对`ACTION_CREATE_DOCUMENT`具体化新建文件的名称。
	* `EXTRA_LOCAL_ONLY`:是否选择本地文件。
* category:
	* `CATEGORY_OPENABLE`:返回的"openable"文件，能够使用`openFileDescriptor()`文件打开。

**获得一个图片**

```
static final int REQUEST_IMAGE_OPEN=1;
public void selectImage(){
	Intent intent=new Intent(Intent.ACTION_OPEN_DOCUMENT);
	intent.setType("image/*");
	intent.addCategory(Intent.CATEGORY_OPENABLE);
	// Only the system receives the ACTION_OPEN_DOCUMENT, so no need to test.
    startActivityForResult(intent, REQUEST_IMAGE_OPEN);
}

@Override
protected void onActivityResult(int requestCode,int resultCode,Intent data){
	if(requestCode==REQUEST_IMAGE_OPEN && resultCode == RESULT_OK){
		Uri fullPhotoUri=data.getData();
		//do work with full size phote saved at fullPhotoUri
	}
}
```
第三方app不能对`ACTION_OPEN_DOCUMENT`做出反应。而是系统接收这个显示所有的文件。


为了允许你的app文件，允许其他应用打开他们，不需要实现`DocumentProvider`并且包含一个intent-filter`PROVIDER_INTERFACE`

```
<provider ...
    android:grantUriPermissions="true"
    android:exported="true"
    android:permission="android.permission.MANAGE_DOCUMENTS">
    <intent-filter>
        <action android:name="android.content.action.DOCUMENTS_PROVIDER" />
    </intent-filter>
</provider>
```

样例：[Open files using storage access framework](https://developer.android.com/guide/topics/providers/document-provider)

# 9.音乐或视频

## 9.1播放一个音乐文件

* action:`ACTION_VIEW`
* uri schemem:
	* `file:<URI>`
	* `content:<URI>`
	* `http:<URL>`

* MIME type:
	* `audio/*`
	* `application/ogg`
	* `application/x-ogg`
	* `application/itunes`
	* 或其他类型

**使用**

```
public void playMedia(Uri file){
	Intent intent=new Intent(Intent.ACTION_VIEW);
	intent.setData(file);
	if(intent.resolveActiivty(getPackageManager()!=null){
		startActivity(intent);
	}
}
```

**响应app 过滤**

```
<activity>
	<intent-filter>
		<action android:name="android.intent.action.VIEW"/>
		<data android:type="audio/*"/>
		<data android:type="application/ogg"/>
		<category android:name="android.intent.category.DEFAULT"/>
	</intent-filter>
</activity>
```

## 9.2播放基于搜索的音乐

可以用来响应语音的intent。搜索匹配的列表，播放其中的内容。提供`EXTRA_MEDIA_FOCUS`提供模式。

* action:`INTENT_ACTION_MEDIA_PLAY_FROM_SEACRCH`
* extras:
	* `MediaStore.EXTRA_MEDIA_FOCUS`(必须的)显示搜索模式（艺术家，专辑，歌曲或者播放列表）。如果想听一个歌曲，应该有另外三个额外的内容。歌曲标题，艺术家和专辑。有下面搜索模式：
		* **任意**：`vnd.android.cursor.item/*`播放任意音乐。接收app应该自己选择比如播放者的播放列表。额外参数:
			* `QUERY`(必须的)。空字符串。为了向后兼容，比如当不知道搜索模式可以用这个作为未结构化搜索。