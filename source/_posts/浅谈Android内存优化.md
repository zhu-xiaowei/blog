---
title: 浅谈Android内存优化
date: 2019-01-11 16:39:04
tags: 技术
---

![](https://img.carlwe.com/android_memory_logo.png-h)

今天我们来聊一聊Android 内存优化，这篇文章本来很早就应该写了，但因为小游戏开发太吸引人了，所以这个就拖到了现在才开始，不过我觉得也不晚😁
<!--more-->

>这篇文章主要通过如下三个方面对Android内存优化进行介绍：
>
>1. Android内存分配与回收机制
>2. Android常用的内存优化方法
>3. Android内存分析与监控

文章不会涉及到native内存的优化，因为普通App开发中涉及的较少，如果想了解可以参考[极客时间](https://time.geekbang.org/column/article/71277)张绍文老师的Android开发高手课。

## 一、Android内存分配与回收机制

想要优化Android内存，一些必备的基础知识是不能少的。所以在第一部分，我们先从Application Framework、Dalvik/Art、Linux内核三个部分由浅入深来讲解关于Androd内存相关的知识。

### Application Framework

首先来看下进程的优先级：

![](https://img.carlwe.com/process_priority.jpg)

`前台进程`：用户当前操作所必需的进程。

`可见进程`：没有任何前台组件、但仍会影响用户在屏幕上所见内容的进程。

`服务进程`：正在运行已使用 startService() 方法启动的服务。（后台播放音乐，网络下载数据）

`后台进程`：对用户不可见的 Activity 的进程（已调用 Activity 的 onStop() 方法）

`空进程`：不含任何活动应用组件的进程。保留这种进程的的唯一目的是用作缓存，以缩短下次在其中运行组件所需的启动时间

`进程生命周期`：Android 系统将尽量长时间地保持应用进程，但为了新建进程或运行更重要的进程，最终需要移除旧进程来回收内存。 为了确定保留或终止哪些进程，系统会根据进程中正在运行的组件以及这些组件的状态，将每个进程放入“重要性层次结构”中。 必要时，系统会首先消除重要性最低的进程，然后是重要性略高的进程，来回收系统资源。（一般情况下前台进程就是与用户交互的进程了,如果连前台进程都需要回收那么此时系统几乎不可用了）。由此也衍生了很多进程保活的方法（提高优先级，互相唤醒，native保活等等），出现各种杀不死的进程的APP。

`最后我们需要知道`：Android中由ActivityManagerService 类集中管理所有进程的内存资源分配，我们可以查看其源码来具体分析实现过程。

### Dalvik/Art 虚拟机

#### Android Dalvik Heap

![](https://img.carlwe.com/dalvik_art_gc.jpeg)

`简介`：Android Dalvik Heap与原生Java一样，将堆的内存空间分为三个区域，Young Generation新生代，Old Generation年老代， Permanent Generation持久代。

`对象分配过程`：最近分配的对象会存放在新生代区域，新生代区域分为eden区（伊甸园，圣经中指上帝为亚当夏娃创造的生活乐园）、so区和s1区，s1和s0区也被称为from区和to区（合称Survivor区），他们是两块大小相等并且可以互换角色的空间，绝大多数情况下,对象首先分配在eden区，在一次新生代回收后，如果对象还存活会进入s0或者s1区，之后每一次gc，存活的对象年龄都会相应增加，当达到一定年龄则会进入老年代，最后累积一定时间再移动到持久代区域。系统会根据内存中不同的内存数据类型分别执行不同的gc操作。

`问题`：GC发生的时候，所有的线程都是会被暂停的。执行GC所占用的时间和它发生在哪一个Generation也有关系，新生代中的每次GC操作时间是最短的，年老代其次，持久代最长。GC时会导致线程暂停、界面卡顿的问题在Android Art中得到了优化。

#### Dalvik虚拟机执行模式

![](https://img.carlwe.com/dalvik_gc.jpg)

`Dalvik垃圾回收过程`：GC会去标记和查找所有可访问到的活动对象，这个时候整个程序的线程就会挂起，并且虚拟机内部的所有线程也会同时挂起(左下图) 。之所以要挂起所有线程是确保：所有程序没有进行任何变更，与此同时GC会隐藏所有处理过的对象，最终确保标记了所有需要回收的对象后，GC才会恢复所有线程，并释放空间。

`大内存对象分配`：当发现需要给一个较大的对象(蓝色方块)分配空间时，发现可用空间还是够的，但没有这么大的连续空间供新对象使用，这个时候就不得不进行一次GC回收（红色方块，右下图），为大对象腾出较大并且连续的空间。这就是我们在分配一个较大对象的时候非常容易引起丢帧和卡顿的原因之一，所以Android5.0以前大家都认为Android卡顿是因为Darvik虚拟机的效率低下导致的。

`总结`：Dalvik虚拟机的三个问题

1. GC时挂起所有线程 
2. 大而连续的空间紧张 
3. 内存碎片化严重

#### ART虚拟机的优化

![](https://img.carlwe.com/art_gc.jpg)

`GC过程`：在ART中GC会要求程序在分配空间的时候标记自身的堆栈，这个过程非常短，不需要挂起所有程序的线程.这样就节约了很大一部分时间去查找活动对象。

`大内存对象分配`：ART里会有一个独立的LOS供Bitmap使用，从而提高了GC的管理效率和整体性能.

`内存碎片化`在ART里还会有一个moving collector来压缩活动对象(绿色方块)，使得内存空间更加紧凑。

`总结` ：Google在ART里对GC做了非常大的优化(更高效的回收算法),使ART内存分配的效率提高了10倍，GC的效率提高了2-3倍（可见原来效率有多低），不过主要还是优化中断和阻塞的时间，频繁的GC还是会导致卡顿。

### Linux内核

![](https://img.carlwe.com/linux_kernel.jpg)

`Lowmemorykiller`：ActivityManagerService中trimApplications() 函数中会执行一个叫做 updateOomAdjLocked() 的函数，updateOomAdjLocked 将针对每一个进程更新一个名为 adj 的变量，（用来表示发生内存不足时杀死进程的优先级顺序）并将其告知 Linux 内核，内核同样维护一个包含 adj 的数据结构（即进程表），并通过 lowmemorykiller 检查系统内存的使用情况，在内存不足时，遍历所有进程，选出低优先级的进程杀死，最终由内核去完成真正的内存回收。

`Oom_killer` ：如果上述各种方法都无法释放出足够的内存空间，那么当为新的进程分配内存时将发生 Out of Memory 异常，OOM_killer 将尽最后的努力杀掉一些进程来释放空间。Android 中的oom_killer同样会遍历进程，并计算所有进程的 badness 值，选择 badness 最大的那个进程将其杀掉。

`Oom的条件`：只要allocated + 新分配的内存 >= dalvik heap(堆内存) 最大值的时候就会发生OOM（Art运行环境的统计规则还是和dalvik保持一致）

### 内存不优化会导致哪些问题？

![](https://img.carlwe.com/memory_problem.jpg)

上面介绍了Android内存分配从应用层到Linux层的一些知识，所以我总结出上图内存会导致的一些问题，但是上图只是列出了一些常见情况，前后并没有绝对的因果关系，最后来说下内存抖动。

`内存抖动`：Memory Churn，内存抖动是因为在短时间内大量的对象被创建又马上被释放。瞬间产生大量的对象会严重占用内存区域，当达到阀值，剩余空间不够的时候，会触发GC从而导致刚产生的对象又很快被回收。即使每次分配的对象占用了很少的内存，但是他们叠加在一起会增加Heap的压力，从而触发更多其他类型的GC。这个操作有可能会影响到帧率，并使得用户感知到性能问题。

## 二、Android常用的内存优化方法

在Android中内存优化的方式实在是太多了，往细了说，到你写的每一行代码其实都和内存优化相关。在这里我从三个方面来说下Android内存优化的方法：

>1. 降低运行时内存
>2. 代码优化
>3. 内存泄漏优化

在实际开发中我们可以先考虑降低应用的运行时内存，然后针对代码写的不好的地方着重优化，最后通过规避一些可能导致内存泄漏的编码方式，去提前避免内存泄漏的问题。

### 降低运行时内存

![](https://img.carlwe.com/reduce_running_memory.jpg)

降低运行时内存可以分为减小APK的体积和Bitmap优化两部分：

* 减小APK体积

>1. 去除无用的资源和代码，通过合理使用git，一些由于业务变更而基本不会用到的代码，该删除的绝不能手软。即使以后要用到，通过git也能找回。同时一些图片资源未用到的也应该删除，因为即使gradle配了sharkresource选项，发布的时候这些没有用到的图片依然会被打包到你的apk。
>2. 尽量复用资源，其实这是一种比较好的编码习惯。
>3. 对应用的启动图引导页图片进行压缩，往往这些图片占据了大部分空间，压缩后可以起到很好的效果。平时开发中对于分辨率大雨100*100的图片基本上都会进行压缩，很多好的压缩算法经常可以减少一半的大小，而感官上基本看不出有任何改变。

* Bitmap优化

>1. 统一的bitmap加载器，选择Glide、Fresco、Picasso中的一个作为图片加载框架。实际开发中加载到view的图片的大小不应该超过view的大小，图片加载框架默认会对图片进行缓存，按view实际大小加载。在开发中为了减少apk的大小，一般只放一套3X图片，但是这些图片在小分辨率的手机上直接加载就会出现内存浪费。统一的bitmap加载器就可以很好的解决该问题。
>2. 图片存在像素浪费，对于.9图，美工可能在出图时在拉伸与非拉伸区域都有大量的像素重复。而这些图片是可以缩小，但并不影响显示效果。
>3. inSampleSize:缩放比例，在把图片载入内存之前，我们需要计算一个合适的缩放比例，避免不必要的大图载入。
>4. 选择ARGB_8888/RBG_565/ARGB_4444/ALPHA_8，存在很大差异。
>5. inBitmap：这个参数用来实现Bitmap内存的复用，但复用存在一些限制，具体体现在：在Android 4.4之前只能重用相同大小的Bitmap的内存，而Android 4.4及以后版本则只要后来的Bitmap比之前的小即可。使用inBitmap参数前，每创建一个Bitmap对象都会分配一块内存供其使用，而使用了inBitmap参数后，多个Bitmap可以复用一块内存，这样可以提高性能。

参考：

Android 官网文档[Managing Bitmap Memory](https://developer.android.com/topic/performance/graphics/manage-memory?hl=zh-cn)、[Handling bitmaps](https://developer.android.com/topic/performance/graphics/?hl=zh-cn)

### 代码优化

这里介绍一些好的编码习惯：

![](https://img.carlwe.com/code_optimize.jpg)

1. 考虑使用ArrayMap/SpareseArray而不是传统的HashMap等数据结构，Android系统为移动系统设计的容器ArrayMap更加高效，占用内存更少，因为HashMap需要一个额外的实例对象来记录Mapping的操作。而SparesArray高效的避免了key和value的自动装箱，而且避免了装箱后的解箱。详细参考[Android性能优化典范](http://hukai.me/android-performance-patterns-season-3/)

2. 在onDraw这种频繁调用的方法要避免对象的创建操作，因为他会迅速增加内存的使用，引起频繁的gc，甚至内存抖动。

3. SoftReference(软引用)、WeakReference(弱引用)、PhantomReference(虚引用)

   > `SoftReference`：如果一个对象只具有软引用，则内存空间足够，垃圾回收器就不会回收它；如果内存空间不足了，就会回收这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。软引用可用来实现内存敏感的高速缓存。
   >
   > `WeakReference`：与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程，因此不一定会很快发现那些只具有弱引用的对象。 
   >
   > `PhantomReference`：虚引用”顾名思义，就是形同虚设，与其他几种引用都不同，虚引用并不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收器回收。虚引用主要用来跟踪对象被垃圾回收器回收的活动。虚引用与软引用和弱引用的一个区别在于：虚引用必须和引用队列 （ReferenceQueue）联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之 关联的引用队列中。    

4. 谨慎使用large heap，android设备由于软硬件的差异，heap阀值不同，特殊情况下可以在manifest中使用`largeheap=true`声明一个更大的heap空间，使用getLargeMemoryClass()来获取到这个更大的空间。但是要谨慎使用，因为额外的空间会影响到系统整体的用户体验，切换任务时性能大打折扣，对于oom异常是治标不治本的一种做法。

5. 谨慎使用多进程，使用多进程可以把应用中的部分组件运行在单独的进程当中，这样可以扩大应用的内存占用范围，但是这个技术必须谨慎使用，绝大多数应用都不应该贸然使用多进程，一方面是因为使用多进程会使得代码逻辑更加复杂，另外如果使用不当，它可能反而会导致显著增加内存。当你的应用需要运行一个常驻后台的任务，而且这个任务并不轻量，可以考虑使用这个技术，一个典型的例子是创建一个可以长时间后台播放的Music Player。如果整个应用都运行在一个进程中，当后台播放的时候，前台的那些UI资源也没有办法得到释放。类似这样的应用可以切分成2个进程：一个用来操作UI，另外一个给后台的Service。

6. 考虑第三方库的大小，如果会和现有的代码或其他库的代码重复，考虑不要真个引入而是把库的代码精简之后再引入。

### 内存泄漏优化

内存泄漏的原因有很多，下面介绍一些常见的，我们需要在开发中多注意：

![](https://img.carlwe.com/memory_leak_optimize.jpg)

1. Activity调用了finish，但是引用Activity的对象未被释放(生命周期没有结束)，Activity Context被传递到其他实例中，可能导致自身被引用而发生泄露，建议使用weakReferce。

2. 除必须使用Activity Context的情况(Dialog的context必须是Activity),我们可以使用Application Context来避免Activity泄露。

3. 大多数情况下，我们对Bitmap对象增加缓存机制，但是有时候部分bitmap需要及时回收。比如我们临时创建的摸个相对大的bitmap对象，变换得到新的bitmap对象后，尽快回收原始的bitmap，及时释放原来的空间。

4. webview引起的内存泄漏主要是因为org.chromium.android_webview.AwContents 类中注册了component callbacks，但是未正常反注册而导致的。让onDetachedFromWindow先走，在主动调用destroy()之前，把webview从它的parent上面移除掉(Basewebfragment onDestroy())

5. 虽然单例模式简单实用，提供了很多便利性，但是因为单例的生命周期和应用保持一致，使用不合理很容易出现持有对象的泄漏。

6. 我们在对数据库进行操作时，使用完cursor没有及时关闭，cursor的泄露，会对内存管理带来负面影响。

7. 谨慎使用static对象，因为static的生命周期过长，和应用的进程保持一致，使用不当很可能导致对象泄漏。

`总结`：在实际的线上环境中发现，大部分内存泄漏是因为被调用的对象生命周期不同步导致，生命周期不同步不仅仅会导致内存泄漏，更会出现异常，崩溃等更严重的问题。

### 做好上面说的1、2、3就够了吗？

![](https://img.carlwe.com/memory_is_enough.jpg)

前面我们已经从系统级别了解了Android Framework、Darlvik/Art虚拟机、Linux在内存分配上的原理，接着又在代码级别分别从减少内存占用、避免内存泄漏和代码优化三个方面介绍了如何避免内存问题，再加上当前科技发展是如此迅速，4GB内存已经是很常见的手机配置。LPDDR4X的高速闪存也越来越被广泛的使用。对于内存优化我们是不是就已经可以高枕无忧了，有上面这些就够了吗？

我想即使我们再了解内存，写的代码再好，用户的手机再先进，总还是有出错的时候，那么事后的内存分析和监控是必不可少的了！

## 三、Android内存分析与监控

Android内存分析和监控主要介绍如下四种方式：

>1. 查看GC日志
>2. 查看内存使用情况
>3. 通过LeakCanary监控内存 泄漏
>4. 线上监控

### 查看GC日志

#### GC的类型：

![](https://img.carlwe.com/gc_type.jpg)

`Concurrent`： 不会暂停应用线程的并发垃圾回收。此垃圾回收在后台线程中运行，而且不会阻止分配。

`Alloc`： 您的应用在堆已满时尝试分配内存引起的垃圾回收。在这种情况下分配线程中发生了垃圾回收。

`Explicit`：由应用明确请求的垃圾回收，例如，通过调用system.gc()。与 Dalvik 相同，在 ART 中，最佳做法是您应信任垃圾回收并避免请求显式垃圾回收（如果可能）。不建议使用显式垃圾回收，因为它们会阻止分配线程并不必要地浪费 CPU 周期。如果显式垃圾回收导致其他线程被抢占，那么它们也可能会导致卡顿（应用中出现间断、抖动或暂停）

`NativeAlloc`：原生分配（如位图或 RenderScript 分配对象）导致出现原生内存压力，进而引起的回收。

#### 查看垃圾回收日志

![](https://img.carlwe.com/gc_log.jpg)

在AndroidStudio Logcat过滤GC，然后操作App一段时间后会出现上图的GC内容：

> `垃圾回收原因+垃圾回收的名称+释放对象+释放对象大小+释放大型对象的大小+堆统计数据+暂停时间`
>
> `LOS objects`是前面所说到的Art虚拟机新增的
>
> 着重关注最后面的暂停时间，超过16ms会影响界面，一般大于700ms会影响体验，Android Vitals 将连续丢帧超过 700 毫秒定义为冻帧，也就是42帧

### 查看内存使用情况

通过查看内存使用情况来分析App的内存占用是非常必要的，下面分别介绍如下两种方式：

>1. adb shell
>2. Profiler

#### 查看内存使用情况

![](https://img.carlwe.com/adb_dumpsys.jpg)

详细的使用请参考AndroidDeveloper[调查RAM使用情况]( https://developer.android.com/studio/profile/investigate-ram?hl=zh-cn)

#### 使用Profiler分析内存

AndroidStudio的Profiler功能越来越强大，不仅集成了内存分析，还有电量、CPU、网络等数据的分析。

![](https://img.carlwe.com/use_profiler.jpg)

如何通过Profiler进行内存的分析，如何找到内存泄漏请查看

[使用 Memory Profiler 查看 Java 堆和内存分配](https://developer.android.com/studio/profile/memory-profiler)

这里要说下，Android官网的很多文章都被翻译成了中文，这对国内的开发者来说越来越有好了，但要注意中文翻译的文章会比较滞后，最新版一般都是英文。

### 使用LeakCanary监控内存泄漏

![](https://img.carlwe.com/leakcanary_logo.png)

`LeakCanary名字的由来`：Canary是煤矿中金丝雀表达的参考，暗示了矿工将随身携带进入矿井隧道的笼养金丝雀（鸟类）。如果在矿井中收集到一氧化碳等危险气体，这些气体会在杀死矿工之前杀死金丝雀，从而提供警告立即离开隧道。

`原理`：LeakCanary通过ApplicationContext统一注册监听的方式，通过application.registerActivityLifecycleCallbacks来绑定Activity生命周期的监听，从而监控所有Activity; 在Activity执行onDestroy时，开始检测当前页面是否存在内存泄漏，并分析结果。KeyedWeakReference与ReferenceQueue联合使用，在弱引用关联的对象被回收后，会将引用添加到ReferenceQueue；清空后，可以根据是否继续含有该引用来判定是否被回收；判定回收， 手动GC, 再次判定回收，采用双重判定来确保当前引用是否被回收的状态正确性；如果两次都未回收，则确定为泄漏对象。

`LeakCanary的问题 `：LeakCanary也有一定的不确定性，一般同一个地方反复泄漏5次，算是一个泄漏，同时不建议用在线上环境。

详细查看 [Github](https://github.com/square/leakcanary)

### 线上监控

线上的内存监控一般都是一些大公司在做，例如美团的[Probe](https://static001.geekbang.org/con/19/pdf/593bc30c21689.pdf)还有微信最近开源的[Matrix](https://mp.weixin.qq.com/s/muX_RgK3cXiMd4j2B0L_lA)，个人觉得这个可以去了解下，大公司用户数多时会用到，小公司App接入必要性不是很大，一般来说把上面的介绍的部分做好了就足够了。