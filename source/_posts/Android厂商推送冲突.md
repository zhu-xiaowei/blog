---
title: Android厂商推送冲突
date: 2020-03-21 14:55:26
tags: 技术
---

![](https://img.carlwe.com/MiPush逻辑图.png)

* **关于厂商推送**

推送已经成为当下大部分App的必备功能了，相信大家每天都会收到新闻、聊天消息、普通App的活动等消息推送，而为了提升推送的到达率，大家也做了各种优化，最初应用进程被杀后，就收不到推送了，所以前几年就出了各种应用保活的方法，而Android 8.0以后应用保活的“妙招”就很难生效了。为了提升推送的到达率，当应用被杀后大家都会选择走厂商的推送通道，各大厂商在系统级别会有一个长链接来统一处理推送消息，从而确保当应用被杀后你也能顺利收到推送。

* **通知栏消息和透传消息**

通知消息：很好理解，就是收到推送自动显示到通知栏的消息。

透传消息：顾名思义就是直接把消息内容传到客户端，对用户来说是透明的，收到消息后是否显示及显示形式由客户端控制，使用起来更加灵活，很多第三方SDK中称之为自定义消息。一般App中的自定义消息也都是用的透传消息，App收到通知后可通过拿到的消息通知的类型，以及需要前往的页面进行相应处理。 

## 问题由来

最开始我们App中使用网易云信的SDK来支持聊天业务；使用极光推送来处理业务的推送消息，由于上述Android版本的限制，极光和云信都陆续提供了自家关于厂商推送的接入方案，先来说极光的厂商推送，极光针对不同厂商提供的推送SDK进行的封装，你只需要引入对应封装好的包就行，如下：

``` groovy
dependencies {
    ...
    implementation 'cn.jiguang.sdk:jpush:3.3.9'
    implementation 'cn.jiguang.sdk:jcore:2.1.6'
    implementation 'cn.jiguang.sdk.plugin:xiaomi:3.3.9'
    implementation 'cn.jiguang.sdk.plugin:huawei:3.3.9'
    implementation 'cn.jiguang.sdk.plugin:meizu:3.3.9'
    implementation 'cn.jiguang.sdk.plugin:oppo:3.3.9'
    implementation 'cn.jiguang.sdk.plugin:vivo:3.3.9'
}
```

以小米为例，

##解决问题 