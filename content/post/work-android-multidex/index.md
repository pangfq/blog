---
title: MultiDex
description: 
slug: work-android-multidex
date: 2022-03-28T04:14:54-08:00
categories:
    - Android
tags:
    - multidex
---

## 概述

MultiDex，是Android官方出的，一个用来解决Dalvik无法运行大型项目问题的方案。

先来看下MultiDex产生的历史背景，其实就是Dalvik为什么无法运行大型项目。

## 历史背景

在Dalvik虚拟机时代，由于Google的短视，在设计Dalvik虚拟机的时候，没有想到有朝一日，一个APK中的dex会如此之大，导致Dalvik虚拟机无法加载大型项目，具体表现为以下3点：

1. 只能加载一个dex文件
2. 单个dex文件中的方法数不能超过65536（方法数超限）
3. 单个dex文件中的方法占用内存不能超过指定内存（LinearAlloc，线性内存超限）

### 方法数超限

Dalvik在首次加载apk中的dex的时候，会使用dexopt工具对dex进行优化，优化过程中会使用short类型的字段索引dex中的方法数，short是2个字节，2^16=65536，所以这也是为什么单个dex中的方法数不能超过65536的原因。

### 线性内存超限

和方法数超限的问题一样，都是发生在dexopt过程，会使用LinearAlloc来缓存dex中的方法信息，LinearAlloc是一个固定的缓存区（Android4.0之前是4M，Android4.0之后是8M或16M），所以可能在方法数还未达到65536就已经超过线性内存了。

## 解决方案

Android官方出了一个Multidex的方案，就是多个dex的意思，大概是在打包apk的时候，判断dex中的方法数是否超过65536，如果超了则对dex进行拆分，拆分成一个主dex和多个分dex，会把直接引用放在主dex中，间接引用放在分dex中，并让主dex调用分dex。

你会有疑问，上述解决方案只能解决65536方法数的问题，那LinearAlloc的问题如何解决呢？

LinearAlloc的问题一般出现在低端机型，比如Android4.0之前的机型，其LinearAlloc线性内存缓存大小才4M，所以非常容易出现线性内存的问题，如果想要适配低端机型，可以指定dex方法数来进一步降低方法数，比如你可以设置成40000，表示方法数不能超过40000。

下面是相关代码与Gradle配置：

```groovy
// 1. 在app模块的build.gradle中配置开启multidex
android {
    defaultConfig {
        
        multiDexEnabled true

    }
  
  // 以下可选，用来配置dex方法数，解决低端机型的LinearAlloc问题
  afterEvaluate { 
  	tasks.matching { 
    	it.name.startsWith('dex') 
  	}.each { dx -> 
    	if (dx.additionalParameters == null) { 
      	dx.additionalParameters = []
    	}
    	// 设置每一个dex的方法数不超过48000，占用内存不超过4M，即可保证不在2.3的低端机型上出现线性内存的限制
    	dx.additionalParameters += '--set-max-idx-number=48000'
    
    	// --main-dex-list= 参数是一个类列表的文件，在该文件中的类会被打包在第一个 dex 中。
    	// multidex.keep 里面列上需要打包到第一个 dex 的 class 文件，注意，如果需要混淆的话需要写混淆之后的 class 。
    	dx.additionalParameters += "--main-dex-list=$projectDir/multidex.keep".toString()
  	} 
	}
}
// 2. 添加依赖
dependencies {
    implementation 'androidx.multidex:multidex:2.0.1'
}
```

```java
// 3. 自定义Application继承MultidexApplication
class App : MultiDexApplication() {
	
  @Override
  protected void attachBaseContext(Context base) {
    super.attachBaseContext(base);
    // 除了直接继承MultidexApplication外，还可以自己覆写attachBaseContext()方法，并自行调用MultiDex.install(this);
    // MultiDex.install(this);
  }
  
}
```

## 相关问题

### ART有这个问题吗？

答：没有。

因为ART虚拟机，在安装APK时，不需要用到Dex2Opt工具，所以就不存在上述问题。

ART虚拟机会扫描所有Dex文件，并将它们编译为一个后缀名为.oat的机器码文件。

### Dalvik为什么要进行Dex2Opt？

答：因为Dalvik加载dex时，会用到很多依赖，Dex2Opt的过程，就是将这些依赖提前都准备好，并和dex一同打包成.odex文件，这是一种空间换时间来提高加载效率的方式。

### Dex2Opt的时机是什么？

答：有两个时机：

1. 安装APK的时候，对主dex进行opt
2. 首次运行APP的时候，在Application中执行MultiDex.install(this)，是对其余dex进行opt