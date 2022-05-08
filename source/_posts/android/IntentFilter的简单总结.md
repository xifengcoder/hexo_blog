---
title: IntentFilter的主要标签：
urlname: IntentFilter的主要标签：
date: 2022-04-14 10:54:54
tags: Android
categories: Android
description: IntentFilter匹配规则的简要总结
---

#### 一、IntentFilter的主要标签：

##### 1. \<category>

1. 主菜单进入必须使用 `<category android:name="android.intent.category.LAUNCHER" />`

2. 接收隐式Intent必须使用`` <category android:name="android.intent.category.DEFAULT" />``

##### 2. \<action>

主菜单进入必须使用

`<actionandroid:name="android.intent.action.MAIN" />`

对于隐式的Intent必须指定action

##### 3. \<data>

URI的内容组成：

 ` scheme://host:port/path`

例如：

```java
content://com.fx.demo:200/system/etc
http://www.google.com/getDetails?id=123
```

主要标签

```xml
<data android:host="*string*"
   android:mimeType="*string*"
   android:path="*string*"
   android:pathPattern="*string*"
   android:pathPrefix="*string*"
   android:port="*string*"
   android:scheme="*string*"/>
```

#### 二、IntentFilter匹配隐式Intent规则

第1步：匹配action

第2步：匹配category

第3步：匹配data

以下几句简单的话帮助记忆：

1、没有data的Intent只能匹配没有data的Filter;

2、只有URI没有数据类型的Intent只能能匹配只有 URI没有数据类型的Filter;

3、没有URI只有数据类型的Intent只能能匹配没有URI只有数据类型的Filter;

4、 URI和数据类型都有的Intent能匹配：

  ① 只有相同数据类型的Filter

  ② 只有相同URI的Filter

  ③ URI和数据类型都相同的Filter

注：如果Filter没有指明scheme，默认支持：`content:`和`file:`。
