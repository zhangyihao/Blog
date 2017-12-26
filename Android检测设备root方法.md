---
title: Android检测设备root方法
date: 2017-12-14 20:37:27
tags:
	- Android
	- root
categories:
  - Android
comments: false

---

项目上出于安全因素考虑，提出了Android App是不能运行在已经Root过的设备上的需求，下面介绍下常见的Root检测方法（具体安全问题，不在本文讨论范围内）。

# 查看系统是否测试版本 #

我们可以通过查看系统版本方式检测，系统版本分为：test-keys（测试版）、release-keys（发布报）。在代码中可以通过***android.os.Build.TAGS***获取到当前系统的版本，如果不是*release-keys*，则有可能已经root过，存在风险。

<!--more-->

然而在实际情况下，有些手机厂商的系统发布出来系统未root，但系统版本就是test-keys的，所以是否使用这种方式需要斟酌。

# 检测是否存在Superuser.apk #

Superuser.apk是一个使用率较高的root软件，可以通过检测这个app是否存在判断设备是否root。

# 检测Su命令 #

Android是基于Linux开发的，那么可以使用su命令把当前用户切换到超级用户下，而Android系统出于安全等方面因素考虑，去掉了一些Linux下的命令，例如su、find、mount等，因此可以通过检查系统同是否有su命令或者su命令是否可执行来判断设备是否root。

## 检测常用目录下是否存在su ##

检测的常用目录包含：/system/bin/、/system/xbin/, /system/sbin/, /sbin/, /vendor/bin/。
这个方法缺点是有可能漏掉一些不常用的目录。

## 使用which命令检测是否存在su ##

which命令是linux下搜索某个命令的位置的命令。可以通过执行which su命令，查找系统中是否存在su命令。

## 执行su，看能否获取root权限 ##

直接在代码中使用语句*Runtime.getRuntime().exec("su");*执行su命令，如果存在这个命令，那么就会执行这个命令，并向用户提示给app开启root权限。缺点就是用户可以看到请求开启root权限的提示，用户不友好。

# 结论 #

通过以上方法介绍，然后综合各个方式的优缺点，最后采用*检测常用目录下是否存在su命令*及*使用which命令检测是否存在su*的方法检测设备是否root。

具体代码可参考：https://github.com/zhangyihao/AndroidSSL/blob/master/AndroidSSL/src/com/zhangyida/util/RootCheckUtil.java


# 参考文章 #

1. [Android root检测方法小结](http://blog.csdn.net/lintax/article/details/70988565)
2. [【Android】不弹root请求框检测手机是否root](http://www.cnblogs.com/waylife/p/android_root_check.html)