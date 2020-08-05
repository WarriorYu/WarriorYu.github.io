---
title: ButtterKnife和AOP
categories:
  - Android源码分析
tags:
  - Android第三方库源码分析
date: 2020-08-05 09:55:50
---

1. ButtterKnife 的工作原理

   - 编译的时候扫描注解，并做相应处理，生成java代码，生成 java 代码是调用 javapoet 库生成的。
   - 调用 ButterKnife.bing(this)；方法的时候，将 ID 与对应的上下文绑定在一起。

2. AOP 插桩工具 AspectJ、ASM、ReDex

   练习了AspectJ的使用

   参考：

   [Android开发高手课：27 | 编译插桩的三种方法：AspectJ、ASM、ReDex](https://time.geekbang.org/column/article/82761?utm_source=pinpaizhuanqu&utm_medium=geektime&utm_campaign=guanwang&utm_term=guanwang&utm_content=0511)

   [深入理解Android之AOP](https://blog.csdn.net/innost/article/details/49387395)

   [AspectJ 在 Android 中的使用](https://blog.csdn.net/yxhuang2008/article/details/94193201)

   https://github.com/raphw/byte-buddy

   [HujiangTechnology](https://github.com/HujiangTechnology)/[gradle_plugin_android_aspectjx](https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx)