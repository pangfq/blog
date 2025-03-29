---
title: APK打包流程
description: 
slug: work-android-apk_pkg
date: 2022-03-28T04:14:54-08:00
categories:
    - work
tags:
    - android
    - apk
---

### 概述

1. 资源：使用aapt工具生成R文件，就是资源名与资源id的映射表；
2. 代码：使用javac编译成class文件，再转成dex文件；
3. 未签名的apk：使用ApkBuilder生成未签名的APK文件；
4. 签名：使用JarSigner签名，生成META-INF目录以及里面3个文件，号称签名三兄弟；
5. 打包：使用ZipAlign工具压缩打包。

上述是v1签名打包的流程，如果是v2、v3签名打包，则4和5互换顺序，并且签名使用ApkSigner工具，因为v2、v3是对ZIP进行签名的，所以要先打成ZIP包后进行签名，并将签名信息写入到ZIP的某个块中。

