---
title: APK安装流程
description: 
slug: work-android-apk_install
date: 2022-03-29T04:14:54-08:00
categories:
    - Android
tags:
    - apk
---
## 概述

1. 解压：复制并解压到data/app目录下；
2. 校验签名；
3. 解析AndroidManifest.xml：在data/data/目录下创建以应用包名命名的目录；
4. 优化dex：Dalvik虚拟机会使用Dex2Opt工具对主dex优化成odex文件，其实就是将涉及到的依赖和dex打包到一起；如果是ART虚拟机将会扫描所有dex文件并转成oat机器码文件；
5. 注册四大组件：将AndroidManifest.xml解析出的四大组件注册到PackageManagerService中，这样APP在启动某个四大组件时会通过跨进程通信到SystemServer进程中进行校验（PMS、AMS等各种MS都在SystemServer进程运行）；
6. 最后发送安装完成的广播。