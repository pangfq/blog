---
title: adb命令
description: adb，是Android Debug Bridge，Android调试桥。
date: 2025-03-29T07:57:48-08:00
categories:
    - Example Category
tags:
    - Android
    - ADB
---

### 概述

adb，是Android Debug Bridge，Android调试桥。

其实就是Android提供的一个命令行工具，可以与设备进行通信，方便调试。

### 工作原理

看着小小的adb工具，其实是一个C/S架构的程序，一共涉及到3个组件：

1. Client：客户端，用于发送命令。就是在电脑上打开的终端窗口，输入adb命令，就是一个adb客户端了，我们可以打开多个窗口。
2. Server：服务端，用于接收命令并转发到设备。服务端在电脑上作为后台进程运行。
3. adbd：守护程序，用于在设备上运行命令。守护程序在每个设备上作为后台进程运行。

### USB连接

USB连接调试，就是通过数据线连接，调试之前要去“开发者模式”中把“USB调试”打开。

### WIFI连接

没想到吧，还可以WIFI连接调试，但这个有点鸡肋，因为首先要用数据线连接，输入adb tcpip port 告诉设备打开一个端口建立一个TCP/IP连接，然后才能实现WIFI连接，问题是我手边有数据线还使用WIFI连接干啥？！（Android11以上的设备不需要先用数据线连接设置，可以直接到开发者模式下打开相关设置即可）

另一个鸡肋的原因是，测试机我们不会专门给它充电，全靠上班连电脑开发时顺便充下电，所以WIFI连接用的很少，但也要知道可以实现WIFI连接调试。

我们看下具体的操作过程（Android10及以下）：

1. 连接数据线
2. 在终端输入adb tcpip 5555，表示通知设备开始监听5555端口的tcp/ip连接
3. 拔掉数据线
4. 找到设备ip地址
5. 在终端输入adb connect 设备ip地址

断开连接：adb disconnect

### 常用命令

#### 查看设备

```shell
adb devices
```



#### 安装应用

```shell
// -r 是replace的首字母，表示覆盖安装
adb install -r apk绝对路径
```



#### 复制文件

```shell
// 表示从手机上把某个文件复制到电脑上
adb pull remote local

// 表示从电脑上把某个文件复制到设备上
adb push local remote
```



#### 停止adb服务器

```shell
// 如果发生一些异常情况，可以尝试停止服务器解决
adb kill-server
// 杀掉adb服务器之后，随便在终端输入任意adb命令即可重新启动adb服务器
```

#### 指定设备发送

```
$ adb devices
List of devices attached
emulator-5554 device
emulator-5555 device

// 通过-s指定设备序列号，即可发送命令到指定设备
$ adb -s emulator-5555 install helloWorld.apk
```



#### 发送shell命令

```shell
// 有两种方式
// 1. 进入shell后直接使用shell命令
$ adb shell
$>ls

// 2. 不进入shell
$ adb shell ls
```

shell命令有很多，比如截屏、录制视频、启动activity、操作sqlite等等

```shell
adb shell screencap /sdcard/screen.png
// ...
```





