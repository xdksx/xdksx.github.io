---
title: android_howto_makeapk_cmd
date: 2018-05-20 17:08:46
tags: android_make
categories: android
---
### how to make a android by cmd :
     在解压了一个APK文件之后会发现它包含的内容：签名文件夹（三个文件），res文件夹（放置资源），Dalvik,class文件,AndroidMainifest,resources.arsc,若是通过ndk开发，则还包含jni
<!-- more -->

### way 1:方法１
#### prepare
     android studio 创建了一个工程，然后手动在命令行打包，进入工程里的.  或者direct use andrid create project创建

cd ~/Desktop/FirstTest/app/src/main
mkdir gen
mkdir build
mkdir out

在android工程目录下建立Makefile文件，添加如下代码：
	SDK=~/Android/Sdk　　　　
	BUILD_TOOLS=$(SDK)/build-tools/25.0.3
	PLATFORMS=$(SDK)/platforms/android-25
	aapt=$(BUILD_TOOLS)/aapt 
	dx=$(BUILD_TOOLS)/dx
	aidl=$(BUILD_TOOLS)/aidl
	apkbuilder=$(SDK)/tools/apkbuilder2 //后面不支持apkbuilder所以
	adb=$(SDK)/platform-tools/adb


#### 资源编译，生成 R.java

	aapt_task:
	    $(aapt) package \
	    -f \ #如果编译出来的文件已经存在，强制覆盖
	    -M  AndroidManifest.xml  \ # Mainifest.xml 的路径
	    -I  $(PLATFORMS)/android.jar \ # 某个版本平台的 android.jar 的路径　#依赖多个文件时用:隔开如:-I xx.jar:xxx.jar:...
	    -S  res/ \ # res 文件夹路径
	    -J gen/ \ # 生成 R.java 的输出目录
	    -m  #使得生成的包的目录放在 -J 参数指定的目录

#### 代码编译，生成 .class

	javac_task:
	    javac -source 1.7 -target 1.7 \ # 使用 jdk1.8 编译 1.7 的 .class 文件
	    -encoding UTF-8 \ 
	    -bootclasspath  $(PLATFORMS)/android.jar \ #覆盖引导类文件的位置 依赖多个文件时用:隔开如:-I xx.jar:xxx.jar:...
	    -d build/ \ #指定放置生成的类文件的位置
	    java/thereisnospon/dextest/*.java \
	    gen/thereisnospon/dextest/*.java \
    
    
#### 生成 .dex

	dx_task:
		$(dx) --dex --output=build/classes.dex \
		build  
	
#### 资源文件初始包

	resapk_task:
		$(aapt) package -f \
	    -M  AndroidManifest.xml  \
	    -I  $(PLATFORMS)/android.jar \
	    -S  res/ \
	    -F  out/resources


#### 将.dex 文件加入到资源文件初始包中
　　
注意这一步由于－rf xxx得在,src文件夹的母文件夹下执行

	apk_task:	
		java -cp $(SDK)/tools/lib/sdklib-25.3.2.jar \
		  com.android.sdklib.build.ApkBuilderMain \
		  Demo.apk -v -u -z src/main/out/resources\
		  -f src/main/build/classes.dex -rf src
	
#### 签名，使用debug的签名

	signer:
		jarsigner -verbose \
	    -keystore ~/.android/debug.keystore \
	    -storepass android \
	    -keypass android \
	    Demo.apk  androiddebugkey
	
#### 一次性打包

	pkg: 
		make apk_task
		make signer 
	

#### 卸载apk
	uninstall:
		$(adb) uninstall  thereisnospon.dextest
	

#### 安装apk

	install: 
		$(adb) install out/app.apk

#### 运行

run:
	make pkg 
	make uninstall
	make install
	

		$(adb) shell am start -n thereisnospon.dextest/        thereisnospon.dextest.MainActivity  

### 方法２，用gradle,

首先用AS 建立工程，之后在工程文件下，之星执行，gradle clean
gradle build
即生成apk文件
   

  