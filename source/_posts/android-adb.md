---
title: android_adb
date: 2018-05-20 17:50:14
tags: android_make
categories: android
---
#### android adb command
从android群英传中学习到额外的几个adb指令，是之前没接触到的：
adb list targets
adb install -r　xx.apk -r为覆盖
adb shell df
<!-- more -->
adb shell pm list packages -f
adb shell input keyevent 3　　－－模拟按键输入，这里为点击home建
adb shell touchscreen ..模拟滑动
adb shell dumpsys 监听Activity运行状态
adb shell screenrecord /sdcard/demo.mp4  录制　屏幕
adb shell am start -n 包名/包名＋类名

更多，见google　develop中android studio的部分
另在源码目录中/system/core/toolbox  和/frameworks/base/cmds为所有ADB命令和shell命令来源
