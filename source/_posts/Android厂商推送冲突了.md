---
title: Android厂商推送冲突了
date: 2020-03-21 14:55:26
tags: 技术
---

![](http://img.carlwe.com/push_confilt_xiaomi.png)

<!--more-->

* **关于厂商推送**

推送已经成为当下大部分App的必备功能了，相信大家每天都会收到新闻、聊天消息、普通App的活动等消息推送，而为了提升推送的到达率，大家也做了各种优化，最初应用进程被杀后，就收不到推送了，所以前几年就出了各种应用保活的方法，而Android 8.0以后应用保活的“妙招”就很难生效了。为了提升推送的到达率，当应用被杀后大家都会选择走厂商的推送通道，各大厂商在系统级别会有一个长链接来统一处理推送消息，从而确保当应用被杀后你也能顺利收到推送，下图描述了小米的厂商推送流程。

![](https://img.carlwe.com/MiPush逻辑图.png)

* **通知栏消息和透传消息**

通知栏消息：很好理解，就是收到推送自动显示到通知栏的消息。

透传消息：顾名思义就是直接把消息内容传到客户端，对用户来说是透明的，收到消息后是否显示及显示形式由客户端控制，使用起来更加灵活，很多第三方SDK中称之为自定义消息。一般App中的自定义消息也都是用的透传消息，App收到通知后可通过拿到的消息通知的类型，以及需要前往的页面进行相应处理。 

## 问题由来

最开始我们App中使用网易云信来支持聊天业务；使用极光推送来处理业务的推送消息，由于上述Android版本的限制，进程保活变得困难，也是不推荐的方式。这也导致了应用被杀后消息收不到，所以极光和云信都陆续提供了自家关于厂商推送的接入方案。

由于业务需要我们先接入了极光的厂商推送，极光针对不同厂商提供的**厂商推送SDK**进行了封装，你只需要引入对应封装好的包就行，如下：

``` groovy
dependencies {
    ...
    implementation 'cn.jiguang.sdk:jpush:3.3.9'
    implementation 'cn.jiguang.sdk:jcore:2.1.6'
    implementation 'cn.jiguang.sdk.plugin:xiaomi:3.3.9'
    implementation 'cn.jiguang.sdk.plugin:huawei:3.3.9'
    implementation 'cn.jiguang.sdk.plugin:oppo:3.3.9'
    ...
}
```

以小米为例，我们看看极光引入的`cn.jiguang.sdk.plugin:xiaomi:3.3.9` arr包是如何工作的：

![](http://img.carlwe.com/plugin_xiaomi_push_remark.png)

可以看到极光提供的这个arr包中直接把小米官方提供的`MiPush_SDK_Client_3_6_18.jar`(处理小米厂商推送的SDK) 包了进来，同时提供了一个`PluginXiaomiPlatformsReceiver`类，让其继承自上述小米jar包中的`PushMessageReceiver`，打包后`PluginXiaomiPlatformsReceiver`会被添加到Manifest文件，当系统收到推送后，会将消息转发到继承了`PushMessageReceiver`的类，所以`PluginXiaomiPlatformsReceiver`就会收到消息，并将消息传递给极光自己的SDK进行处理，后面的流程就和App在前台的推送流程一样了。简单总结下这个Plugin类：

>  Plugin类会被注册到Manifest从而接收系统消息，并在对应的回调方法中将消息转发给极光SDK处理。

在网易云信兼容厂商推送之前这一切工作的都很好，应用进程被杀后，push推送可正常收到，问题从云信消息推送兼容厂商推送开始：

### 问题一

按照网易云信提供的 **[接入方法](https://dev.yunxin.163.com/docs/product/IM即时通讯/SDK开发集成/Android开发集成/推送?#小米推送)** 需要接入小米的推送SDK，因为极光的已经引入，所以再次引入会冲突，这里就直接不引入，使用极光的就行，然后按照接入流程接入即可，在接入流程后面我们注意到AndroidManifest.xml文件中会插入如下内容：

```xml
<receiver
    android:name="com.netease.nimlib.mixpush.mi.MiPushReceiver"
    android:exported="true">
    <intent-filter android:priority="0x7fffffff"> //这里设置了优先级
        <action android:name="com.xiaomi.mipush.RECEIVE_MESSAGE" />
        <action android:name="com.xiaomi.mipush.MESSAGE_ARRIVED" />
        <action android:name="com.xiaomi.mipush.ERROR" />
    </intent-filter>
</receiver>
```

这个`MiPushReceiver`我们查看源码会发现它主要是处理并转发小米厂商推送的各种事件，`MiPushReceiver`同样是继承自小米push sdk中的`PushMessageReceiver`，`MiPushReceiver`代码如下：

![](http://img.carlwe.com/xiaomi_push_receiver.png)

到这里官方文档说已经可以开始测试推送消息，于是把手机进程杀掉，给手机发送一条消息，确实能够收到。但进程杀掉后原本接收正常的极光推送，现在却收不到了🤪，其他厂商机型有的能收到，但点击推送消息不能打开App，我们看下图来分析原因：

![](http://img.carlwe.com/push_confilt_xiaomi.png)

>不管是极光的消息还是云信的消息，首先都会把消息推给小米的推送云服务，然后小米手机系统会和小米的推送云服务保持一个长链接，MiPush SDK收到后，首先会找到继承了`PushMessageReceiver` 并且注册到Manifest的Receiver，并把消息传给这个Receiver，因为极光和云信在Manifest中都注册了`PushMessageReceiver`，所以这个时候谁能收到就存在不确定性了。如果配置了`priority` 优先级，则优先级高的会收到。

回到上面我们注意到网易云信的 `MiPushReceiver `设置了优先级，所以要解释为什么极光的消息就收不到呢，我赶紧查看了下打包后Manifest中极光的`PluginXiaomiPlatformsReceiver` 如下：

```xml
 <receiver
     android:name="cn.jpush.android.service.PluginXiaomiPlatformsReceiver"
     android:exported="true">
     <intent-filter>
         <action android:name="com.xiaomi.mipush.RECEIVE_MESSAGE" />
     </intent-filter>
     <intent-filter>
         <action android:name="com.xiaomi.mipush.MESSAGE_ARRIVED" />
     </intent-filter>
     <intent-filter>
         <action android:name="com.xiaomi.mipush.ERROR" />
     </intent-filter>
 </receiver>
```

果真，极光并没有设置优先级，这就能解释为什么极光的推送在网易云信接入厂商推送后收不到了。

由于不同的厂商接入厂商推送的方式不同，对于上述这种冲突的表现也不太一样，像小米手机云信的消息总是优先于极光的推送，oppo、vivo都会显示消息，但点击通知栏消息无反应(消息没有传到对应的Receiver)，而华为的部分手机则能正常区分。**总之两个Receiver同时去接收厂商的推送，会出现冲突的情况。**

然后我们继续在网易云信和极光的集成文档中寻找解决这种冲突的方案，终于我们在网易云信的文档后面找到了，紧接着我们遇到了第二个问题。

### 问题二

网易云信的推送文档中提供了**[小米推送兼容性](https://dev.yunxin.163.com/docs/product/IM即时通讯/SDK开发集成/Android开发集成/推送?pos=toc-0-0-2)**的处理方案，云信提供了一个`MiPushMessageReceiver` ,让其他接入了厂商推送并处理推送转发逻辑的Receiver继承这个`MiPushMessageReceiver`，然后在对应的回调方法中处理处理相应的逻辑，`MiPushMessageReceiver`如下：

```java
public class MiPushMessageReceiver extends BroadcastReceiver{
    @Override
    public final void onReceive(Context context, Intent intent) {}
    public void onReceivePassThroughMessage(Context context, MiPushMessage message) {}
    public void onNotificationMessageClicked(Context context, MiPushMessage message) {}
    public void onNotificationMessageArrived(Context context, MiPushMessage message) {}
    public void onReceiveRegisterResult(Context context, MiPushCommandMessage message) {}
    public void onCommandResult(Context context, MiPushCommandMessage message) {}
}
```

然后将自己的Receiver添加到Manifest中，不去设置priority优先级：

```xml
<receiver android:name="xxx.YourSelfReceiver">
    <intent-filter>
        <action android:name="com.xiaomi.mipush.RECEIVE_MESSAGE"/>
        <action android:name="com.xiaomi.mipush.MESSAGE_ARRIVED"/>
        <action android:name="com.xiaomi.mipush.ERROR"/>
    </intent-filter>
</receiver>
```

这样就能保证推送都由网易云信的`MiPushReceiver`先接收到，然后通过判断是否是自己的推送消息，是自己的就直接处理，不是自己的就交给继承自`MiPushMessageReceiver`的Receiver处理，查看网易云信的源码发现确实是这样：

```java
public final class MiPushReceiver extends PushMessageReceiver {
    public MiPushReceiver() {}

    public final void onNotificationMessageClicked(Context var1, MiPushMessage var2) {
        if (g.a(var2.getExtra())) {
            c.a(5).onNotificationClick(var1, var2); //自己处理
        } else {
            MiPushMessageReceiver var3;
            if ((var3 = a.a(var1)) != null) {
                var3.onNotificationMessageClicked(var1, var2);//交给MiPushMessageReceiver处理
            }
        }
    }
    ...
}
```

如果按照云信推荐的方法，处理之后就是这样的流程：

![](http://img.carlwe.com/push_confilct_right.png)

好了到这里处理方式和原理都弄清楚了，我们现在也就只需要将极光处理推送的`PluginXiaomiPlatformsReceiver`改为继承`MiPushMessageReceiver`，然后按照上面的方法将其添加到Manifest中即可，看起来很简单，然后我们再来看看极光的`PluginXiaomiPlatformsReceiver`：

![](http://img.carlwe.com/plugin_xiaomi_push.png)

呃... 那么问题来了，这个类是包在极光推送的arr中的，**怎么去修改打好的arr包中类的继承呢？**这个问题似乎不太好解决啊～

## 解决问题 

### 寻求云信和极光的帮助

首先想到的是这种处理同时监听厂商推送冲突的方案是云信提供的，那就先问问云信的技术有没有解决方案，云信给出的答复如下：

![](http://img.carlwe.com/yunxin_wechat_remark.png)

云信的意思是，他们只提供这种继承的兼容方案，如果是第三方封装了，他们也没太好的办法，然后推荐我们去找极光技术人员，商量把对应的类拆出来，首先想到的是如果极光能提供源码，我们直接修改下继承关系就好了，于是就赶紧找了极光的技术进行了沟通：

![](http://img.carlwe.com/jiguang_wechat_remark.png)

极光的技术表示他们只提供统一封装的版本，同时也没有考虑和其他第三方同时接入SDK导致的冲突问题，并且建议我们只集成一家的厂商通道... 

好吧！云信的人让我们找极光商量处理，极光的不但没有提供方案，还让我们别集成多家的厂商通道。不集成肯定满足不了业务需要。不过同时也能理解，不同的第三方在考虑接入厂商通道的时候应该也都是以自身能实现厂商通道来优先考虑，是否会影响其他的第三方，其他第三方是如何实现的，怎么去兼容，他们也管不了那么多，不过像云信还提供了兼容方案的，确实算不错了！后面发现极光的SDK混淆过，所以不提供源码也挺正常。看来拿不到极光`PluginXiaomiPlatformsReceiver` 的源码，云信和极光两方都提供不了有力帮助，问题只能我们自己想办法解决了。

### 分析问题原理，找解决方案

* 分析作用

回过头来再思考下`PluginXiaomiPlatformsReceiver` 类的作用，在极光的SDK中，这个类继承了小米官方的`PushMessageReceiver` ，然后在打包后被添加到了Manifest文件中，从而有了监听小米系统推送、并转发消息给极光的SDK进行处理的能力，同时`PluginXiaomiPlatformsReceiver`类在其他地方并没有被调用。

* 使用继承呢？

既然我们修改不了源码，第一个想到的是能否通过继承该类来实现呢？不过java是单继承，继承了极光的，就没办法再去继承云信的兼容类了，看来继承行不通。

* 从需求出发

其实我们现在只需要有一个类，内部实现逻辑和云信的 `PluginXiaomiPlatformsReceiver` 一样，能将收到的消息转发给云信SDK，并且该类能任意修改继承关系。好了不知道你想到没有，我们可以在自己的代码里写一个一模一样的类，内部的代码直接把`PluginXiaomiPlatformsReceiver`的拷贝过来，然后修改继承关系不就可以了！是的，我们还是来看下云信的`PluginXiaomiPlatformsReceiver`：

![](http://img.carlwe.com/plugin_xiaomi_push.png)

看到虽然这个类混淆了，不过没关系，源码都在sdk中，在外部也可以调用，我们可以直接把代码拷贝到自己新建的类`PluginXiaomiPlatformsReceiverYx`中:

```java
import cn.jpush.android.thirdpush.xiaomi.a;//引入极光被混淆的包
...

public class PluginXiaomiPlatformsReceiverYx extends MiPushMessageReceiver {

    private static final String TAG = "XMPlatformsReceiver";
    public PluginXiaomiPlatformsReceiverYx() {}

    public void onReceivePassThroughMessage(Context var1, MiPushMessage var2) {
        Logger.dd("XMPlatformsReceiver", "onReceivePassThroughMessage is called. " + var2);
    }

    public void onNotificationMessageClicked(Context var1, MiPushMessage var2) {
        Logger.dd("XMPlatformsReceiver", "onNotificationMessageClicked is called. " + var2);
        if (var2 == null) {
            Logger.v("XMPlatformsReceiver", "message was null");
        } else {
            //虽然混淆了，但是用的都是极光sdk中的方法一样可以正常工作。
            a.a(var1, var2, "action_notification_clicked");
        }
    }
    ...
}
```

可以看到这个类继承了云信提供的`MiPushMessageReceiver`，其每个回调实现和极光之前的一模一样，这样能把收到的消息传给极光处理，然后按照云信的文档将该类添加到Manifest中：

![](http://img.carlwe.com/push_manifest.png)

这里需要注意，为了只让云信去监听厂商的推送，还需要将极光SDK在编译时自动添加到Manifest中的`PluginXiaomiPlatformsReceiver` 手动remove掉，关于Manifest的merge规则我们可以查看Android文档[合并多个清单文件](https://developer.android.google.cn/studio/build/manifest-merge.html)。

这样修改之后，相当于我们就把极光的`PluginXiaomiPlatformsReceiver` **“架空”**了，云信和极光的消息推送就统一由云信来接收，不是云信的消息会交给`PluginXiaomiPlatformsReceiverYx`转发到极光再去处理，流程和上面提到的就一样了：

![](http://img.carlwe.com/push_confilct_right.png)

这里只是以小米厂商推送的冲突为例，其他像华为、魅族、OPPO、VIVO等都可以以同样的方式处理。通过上述方案，可以顺利的完成云信和极光的厂商推送兼容。当升级极光SDK版本时，如果极光各厂商以"Plugin"开头的Receiver内部实现有变化，则直接拷贝对应的内容到自定义的Receiver中，这点需要注意。

## 总结 

虽然按照上面的方式可以解决当前的冲突问题，但这里面有一点就是厂商推送的SDK都是包在极光的SDK中，云信自己并没有单独的集成（如果单独集成会冲突）。这就导致后面云信和极光sdk有升级时，可能两家兼容厂商推送sdk的版本不同。比如华为推送sdk有更新，极光兼容了，但网易云信没有兼容，这个时候还是会出一些问题。所以在升级的时候还需要查看下各个厂商对应的兼容情况再升级，针对上述厂商通道推送冲突如果你有更好的解决方案，欢迎在留言区提出！peace ✌️