---
title: Kotlin中使用data class的代价
date: 2020-01-24 11:21:36
tags: 技术
---

![](https://img.carlwe.com/data_class_user.png)

<!--more-->

## 偶然的发现

最近在做业务删减及apk瘦身，当然得分析安装包的大小，首先就是资源文件以及其他文件的瘦身了，完成之后再继续看了下dex文件，然后按照体积从大到小依次往下找，找到了体积最大的一个类文件。让我惊奇的是，单个体积最大的类文件竟然是一个实体类，而查看代码后发现该类为Kotlin中以简洁著称的 **data class**，一个类文件就有20K😲，比项目中大部分的图片都大了，如下图：

![](https://img.carlwe.com/bigest_data_class_path.png)

下面就来具体讲讲 data class。

## data class优点

正如封面图所示，定义一个类及其属性，只需一行代码，相比java中的class，编译器会自动帮助我们实现构造方法、@Metadata信息及如下方法：

```kotlin
equals()
hashCode()
toString()
componentN()
copy()
```

使用起来可以说是非常方便了。

## data class带来的问题

当data class中的变量较少时，使用起来确实方便，就拿上述User类来说，编译之后会生成的代码如下：

![](https://img.carlwe.com/user_data_class_decompiled.png)

总共89行，看起来还ok，让我们回到上面那个Good类，属性比较多，总共加起来80个属性，通过Android Studio中Tools -> Kotlin -> show kotlin buytcode -> Decompile. 查看编译之后的java代码如下：

![](https://img.carlwe.com/data_class_good_java.png)

一共2320行！确实是比较大的一个文件。因为这个类其实没有用到编译器自动生成的几个方法，我尝试仅仅把data去掉，改成class. 同样查看编译之后的java文件如下：

![](https://img.carlwe.com/class_with_good_java.png)

1044行，少了近一半了，然后再打包之后看看最终的文件大小如下：

![](https://img.carlwe.com/class_with_good_size.png)

效果很明显，Good类大小从20.4K直接降到了6.3K，不到之前三分之一的大小。于是将所有的data class更换成了class，整个apk体积也小了不少。

### 总结

总的来说data class用起来确实很方便，但是这些方便却是通过牺牲最终的文件大小换来的。所以大家以后使用data class 可综合考虑使用场景。