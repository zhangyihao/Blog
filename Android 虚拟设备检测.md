---
title: Android 虚拟设备检测
tags:
  - Android
  - 虚拟设备
categories:
  - Android
date: 2017-10-18 22:00:00
comments: false

---

在做App时遇到这么一个需求：要求App在启动的时候检测当前运行设备是否为虚拟设备，如果为虚拟设备则App停止运行并退出。通过网络上搜集的资料，总结了以下检测方法。

<!--more-->

# 基于固定数据检测

虚拟机设备的ID、IMID一般都为固定值，检测到这些信息为默认值即可判定为虚拟设备。

```java
private static String[] imeis = {"000000000000000"};
private static String[] imsiIds = {"310260000000000"};
private static boolean isEmulatorByImei(Context context){
       TelephonyManager tm = (TelephonyManager) context.getSystemService(Context.TELEPHONY_SERVICE);  
       String imei = tm.getDeviceId();
       String imsiID =tm.getSubscriberId();
       
       for (String knowDeviceid : imeis) {
           if (knowDeviceid.equalsIgnoreCase(imei)) {
               Log.v(TAG, "Find device id: "+knowDeviceid);
               return true;
           }
       }
       
       for (String knowImsiId : imsiIds) {
           if (knowImsiId.equalsIgnoreCase(imsiID)) {
               Log.v(TAG, "Find imis id: "+knowImsiId);
               return true;
           }
       }
       
       return false;  
   }
```
除了以上数据检测外，还可以检测设备的默认手机号码，下面是网上给出的默认手机号码（不推荐这种方法，这些手机号码有可能是真实的）：

```java
private static String[] known_numbers = { "15555215554", "15555215556",
        "15555215558", "15555215560", "15555215562", "15555215564",
        "15555215566", "15555215568", "15555215570", "15555215572",
        "15555215574", "15555215576", "15555215578", "15555215580",
        "15555215582", "15555215584", };
```
以上通过系统API获取并不准确，毕竟可以通过修改源码或者通过攻击修改信息。

# 基于硬件信息检测
通过检测设备的手机型号，可以简单判断设备是否为虚拟设备。如Google官方虚拟机的设备型号为google_sdk或sdk。

```java
private static boolean checkEmulatorBuild() {
       String model = android.os.Build.MODEL;
       if (model.equalsIgnoreCase("google_sdk") || model.equalsIgnoreCase("sdk")) {
           Log.v(TAG, "Find build modle is "+model);
           return true;
       }
       return false;
   }
```
除了检测手机型号外，还可以检测下真机独有的硬件设备，如GPS、蓝牙、温度传感器等。

# 虚拟设备特有文件

```java
private static String[] known_files = {"/system/lib/libc_malloc_debug_qemu.so","/sys/qemu_trace","/system/bin/qemu-props"};
private static boolean checkEmulatorFiles() {
	//检测模拟器上特有的几个文件
       for (int i = 0; i < known_files.length; i++) {
           String fileName = known_files[i];
           File qemuFile = new File(fileName);
           if (qemuFile.exists()) {
               Log.v(TAG, "Find Emulator Files!");
               return true;
           }
       }
       return false;
   }
```
# 虚拟机驱动文件检测

```java
private static String[] known_qemu_drivers = {"goldfish"};
private static boolean checkQEmuDriverFile() {
       File driverFile = new File("/proc/tty/drivers");
       if (driverFile.exists() && driverFile.canRead()) {
           byte[] data = new byte[1024];
           try {
               InputStream inStream = new FileInputStream(driverFile);
               inStream.read(data);
               inStream.close();
           } catch (Exception e) {
           }
           String driverData = new String(data);
           for (String knownQemuDriver : known_qemu_drivers) {
               if (driverData.indexOf(knownQemuDriver) != -1) {
                   Log.i(TAG, "Find know_qemu_drivers!");
                   return true;
               }
           }
       }
       return false;
   }
```
以上方法可以检测出大部分的虚拟设备，但并不是万能的，需要根据不同场景组合不同的方法组合进行判定。

附源码地址：
https://github.com/zhangyihao/AndroidSSL/edit/master/AndroidSSL/src/com/zhangyida/util/VirtualDeviceUtil.java

# 参考文章
1. [Android模拟器检测常用方法](https://www.cnblogs.com/qishuai/p/5756209.html)