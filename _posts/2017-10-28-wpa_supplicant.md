---
layout: post
title: wpa_supplicant
---

wpa_supplicant
It implements key negotiation with a WPA Authenticator and it can optionally control roaming and IEEE 802.11 authentication/association of the wlan driver.
使用时可以是用wpa_cli接口指令的形式或者是wpa_ctrl.c　.h那样提供了API,供外部语言如c++调用，需先打开，调用open。。。再发送请求，request，具体见文档，如下链接：


hostapd includes IEEE 802.11 access point management (authentication / association), IEEE 802.1X/WPA/WPA2 Authenticator, EAP server, and RADIUS authentication server functionality. It can be build with various configuration option, e.g., a standalone AP management solution or a RADIUS authentication server with support for number of EAP methods.


some link:http://w1.fi/wpa_supplicant/devel/
http://www.linuxfromscratch.org/blfs/view/svn/basicnet/wpa_supplicant.html
http://w1.fi/wpa_supplicant/devel/ctrl_iface_page.html
http://w1.fi/wpa_supplicant/
http://www.linuxfromscratch.org/blfs/view/svn/basicnet/wpa_supplicant.html
http://blog.csdn.net/haima1998/article/details/43369949