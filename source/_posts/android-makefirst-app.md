---
title: android_makefirst_app
date: 2018-05-20 17:38:24
tags: android_make 
categories: android
---
###  install and make first app
20170608
今天主要是安装了android-studio环境并成功开发第一个helloworld   app 在模拟器和手机上运行，下面是整个教程：在ubuntu下
#### 1 安装java-jdk:<!-- more -->
a  先下载java-jdk:Java SE Development Kit 8 Downloads
http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html
下载对应系统的版本

b  下载到某个目录，解压到要安装的目录：如：/usr/jdk-8/

c   设置全局环境变量：如上述的安装目录，则将
export JAVA_HOME=/usr/jdk-8
export JRE_HOME=$JAVA_HOME/jre
export CLASSPATH=.:$CLASSPATH:$JAVA_HOME/lib:$JRE_HOME/lib
export PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
添加到/etc/profile文件中，在末尾另起一行添加

d 使用source /etc/profile命令使刚才配置的信息生效

e 测试是否成功：java -version
测试,编写java文件：
public class test{
	public static void main(String[] args){
		System.out.println("hello world");
	}
}
保存为test.java，
生成字节码：javac test.java
运行:java test


### 2 安装android-studio:
a 下载：在官网或者其他网站下载后
b 按照installion说明进行：一般是先选择要安装的目录：如/usr/android-studio
c　直接在终端,进入bin目录后:./studio.sh
d 还有其他配置．比如在任意目录都能打开软件．
e 其他见说明文件等


3 第一个app helloworld
在模拟器上运行
打开软件－－>选择新建一个工程，适当选择之后->选择build->make project->点击run--
-->会提示选择安装下载模拟器-->选择合适的，最好不要是带锁的
－－－>之后会提示成功．再选择建立一个模拟器，之后就可以了

注意有两种架构，x86的比arm的快10倍,但是，linux下需要加速kvm
如果不能运行x86的，可能需要安装kvm:
也可以使用genymotion模拟器
#### 4 安装kvm可选
要知道Android的编译环境Google首推Linux平台（64位的Ubuntu）而Mac系统排到第二位。那么在Linux平台下是如何硬件加速的呢？
那就是传说中的kvm（Kernel-based Virtual Machine），同样的，它需要硬件的支持，比如intel的VT和AMD的V，它是基于硬件的完全虚拟化。
首先要确定你的cpu满足要求，下面有几个命令可以参考：

$ egrep -c '(vmx|svm)' /proc/cpuinfo
4

打印的值不为0即可。

下面安装kvm：

$ sudo apt-get install qemu-kvm
$ sudo adduser linc kvm
$ sudo apt-get install libvirt-bin ubuntu-vm-builder bridge-utils
$ sudo adduser linc libvirtd

(linc为用户名，适当改）
检验安装是否成功：

$ sudo virsh -c qemu:///system list
 Id    Name                           State
----------------------------------------------------

运行，在有模拟器的目录中：
如/home/ksx/Android/Sdk/emulator/emulator -avd Nexus_4_API_22 -qemu -m 2047 -enable-kvm

使用起来果然飞快，连打开网页的速度都令人惊奇。当然了，如果不用命令行启动，直接在Android Studio中启动x86_64架构的模拟器，速度也是很快，唯独arm架构的模拟器启动速度奇慢无比。话又说回来，既然有了比较不错的cpu，那么机器的其他配置一定差不了，这样的配置跑起模拟器来肯定要比原来强。


如果觉得用自带的模拟器不能够满足你的要求，那么可以使用第三方的模拟器Genymotion,网传开发者反应良好。



#### 5在手机上运行app
首先连接手机，打开usb调试
还是一样，但是选择app那里不是app,而是选择Edit Configurations
之后选择usb　device，ok，就可以了，接着运行


####  关于项目结构模式：
默认为android
设置为project可以看到整个完整的目录结构：
　　　　　.gradle和.idea 为自动生成
　　　　　app
　　　　　build   编译时自动生成的文件，不用太关心
　　　　　gradle   包含gradle wrapper的配置文件，使用gradle wrapper的方式不需要提前将gradle下载好，会自动根据本地缓存情况来决定是否联网下载gradle
     .gitignore
     build.gradle
     gradle.properties  全局的gradle配置文件，这里配置的属性会影响项目中所有的的gradle编译脚本
     gradlew　和gradlew.bat在命令行中执行gradle命令
     local.properties　　指定本机android sdk路径
     setting.gradle指定项目中引入的模块
     
     
     app目录下
     build　　为自动生成，同上
     lib   项目使用的第三方库
     test 测试用例
     proguard-rules.pro  代码混淆规则

#### build gradle intruduce


gradle  Groovy 领域语言　　DSL    摒弃了Ant 和Maven 

在app 外有一个 build.gradle。在app中有一个build.gradle
  
在外面的　gradle 也可以构建c++等项目，
buildscript {
    repositories {
        jcenter()　　　//代码托管仓库　利用它可以轻松引用jcenter开源项目
        
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.3.2'　　//声明构建的是android

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

allprojects {
    repositories {
        jcenter()
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}

app中
apply plugin: 'com.android.application'　//表明为android 应用程序模块，为com.android.library表示库模块

android {　　　//安卓闭包
    compileSdkVersion 25　　　　//项目的编译版本，25为API 25,对应android 7.1
    buildToolsVersion "25.0.3"   //项目构建工具版本
    defaultConfig {
        applicationId "org.example.myactivity1"　　　项目包名
        minSdkVersion 25　　　　　//项目最低兼容的android系统版本
        targetSdkVersion 25　　//表明如22表示只在22测试充分，如不启动运行时权限，android６　的运行时权限就不会加，表明只在5上做充分测试
        versionCode 1　　　项目版本编号
        versionName "1.0"　　　项目版本名
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
    }
    buildTypes {//分debug和release版本
        release {
            minifyEnabled false　//是否混淆代码
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'//混淆时使用的规则文件，.txt为sdk文件。pro为根目录文件
        }
    }
}

dependencies {//分本地依赖，库依赖和远程依赖
    compile fileTree(dir: 'libs', include: ['*.jar'])　　//本地依赖声明
    androidTestCompile('com.android.support.test.espresso:espresso-core:2.2.2', {
        exclude group: 'com.android.support', module: 'support-annotations'
    })
    compile 'com.android.support:appcompat-v7:25.3.1'　　//　依赖库
    compile 'com.android.support.constraint:constraint-layout:1.0.2'
    testCompile 'junit:junit:4.12'　　//测试用例库
}

其他：分区工具：gpart,在官网上找到live usb之后下载并做成启动盘，就可以对任意的格式文件系统进行分区，在ubuntu 本地是无法对根目录扩容的，所以可以这样做来扩容

