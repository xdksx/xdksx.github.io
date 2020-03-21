---
title: android_AndroidManifest
date: 2018-05-20 17:46:34
tags: android_coding
categories: android
---
#### 一个典型的androidmanifest文件：

	<?xml version="1.0" encoding="utf-8"?>
	<manifest xmlns:android="http://schemas.android.com/apk/res/android"
	    package="com.example.ksx.helloworld">
	<!-- more -->
	    <application
	        android:allowBackup="true"
	        android:icon="@mipmap/ic_launcher"
			android:label="hallo"//app的名字和显示在bar上的文字
	        android:roundIcon="@mipmap/ic_launcher_round"
	        android:supportsRtl="true"
	        android:theme="@style/AppTheme">
	        <activity android:name=".MainActivity">//活动的注册
			<android:label="this is hallo">//会覆盖上面的
	            <intent-filter>
	                <action android:name="android.intent.action.MAIN" />//声明为主活动
	
	                <category android:name="android.intent.category.LAUNCHER" />
	            </intent-filter>
	        </activity>
	    </application>
	
	</manifest>