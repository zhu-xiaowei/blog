---
title: Kotlin混合开发上
date: 2019-04-30 15:10:15
tags: 技术
---

![](https://img.carlwe.com/kotlin_history.jpg)

<!--more-->

> 本文会从kotlin语言的发展引出kotlin一些特点和用法，从而对kotlin简单上手。

## 了解Kotlin

### Kotlin的发展

如上图所示，kotlin已经经历了将近10年的发展。"Kotlin"来源于一个岛屿的名字，全称是 Kotlin Island，是英语「科特林岛」之意。这个小岛属于俄罗斯，但是开发Kotlin的公司JetBrains 是来自捷克的公司，公司总部位于捷克首都布拉格，那为什么找一个俄罗斯的小岛呢？那是因为JetBrains在俄罗斯的圣彼得堡设有分公司。Kotlin的主要开发工作正是由俄罗斯的圣彼得堡分公司的程序员团队完成的，他们想：Java语言的名字是来自于一个岛，那个岛就是印度尼西亚的爪哇（Java）岛，因盛产咖啡而闻名。Java如此伟大，所以 Kotlin 也得选一个岛作为名字，由此能看出Kotlin团队的雄心勃勃！哈哈哈...

另外：2018 年10月29日发布了 kotlin 1.3，正式发布了"协程"，我们后面会进行介绍。

### Kotlin的基本语法

Kotlin在基础语法上吸取了现代语言的很多优点，去除了很多不必要的标点符号，使得代码相对java简洁很多。

#### 变量

```kotlin
val a = 1;
var str1:String = "hello"
var str2:String = null
val list = arrayListOf("10", "11", "1001")//kotlin方法自动识别类型
```

#### 条件

```kotlin
when(a){
    1 -> println("等于1")
    2 -> println("等于2")
}
```

#### 循环

```kotlin
for ((index, element) in list.withIndex()) {
    println("$index = $element")
}
```

#### 方法

```kotlin
fun max(a: Int, b: Int): Int{
    return if (a > b) a else b
}
```

#### 类

```kotlin
class Book(val title: String, val age: Int)
```

### Kotlin的特点

![](https://img.carlwe.com/kotlin_advantage.jpg)

#### 实用性

1.单例：

```java
//Java
public class SingletonDemo {
    private static SingletonDemo instance=new SingletonDemo();
    private SingletonDemo(){
    
    }
    public static SingletonDemo getInstance(){
        return instance;
    }
}
```

```kotlin
//Kotlin
object SingletonDemo
```

2.默认参数

```java
//Java
public void showEmpty(){
   ...
}
public void showEmpty(String buttonName){
   ...
}
public void showEmpty(String buttonName,String desc){
   ...
}
```

```kotlin
//Kotlin 直接给出默认值，方法的参数，想传哪个就传哪个
//当参数较多时，kotlin推荐在调用方法时显示声明参数名
fun showEmpty(buttonName: String = "",desc: String = ""){
    ...
}
```

#### 简洁

1.类定义

```java
//Java
class Book {
    private String title;
    private int age;
    public String getTitle() {
        return title;
    }
    public void setTitle(String title) {
        this.title = title;
    }
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }
}
```

```kotlin
//Kotlin
class Book(val title: String, val age: Int)
```

2.集合操作

```java
//Java
for(People person: people){
    if(person.getAge() > 20){
        println(person.getName());
    }
}
```

```kotlin
//Kotlin
val people = listOf(Person("xiaohong", 23),Person("xiaoming", 15))
println(peoples.filter { it.age > 20 }.map { it.name })
```

#### 安全

```kotlin
//先定义三个类 方便下面使用
class Address(val city: String, val country: String)
class Company(val name: String, val address: Address?)
class Person(val name: String, val company: Company?)
```

```kotlin
//Kotlin
fun Person.countryName(): String? {
    return company?.address?.country ?: "Unknow"
}
fun Test(){
    val person = Person("xiaohua", null)
    println(person.countryName())
 	println(person.let{it.company})
}
```

```Java
//Java
public void getCountryName(Person person){
    if(person!=null && person.company!=null 
       && person.company.address!=null 
       && person.company.address.country!=null){
        return person.company.address.country;
    }else{
        return "Unknow";
    }
}
public void Test(){
    Person person = new Person("xiaohua",null)
    System.out.println(getCountryName(person))
}
```

> Kotlin推出了很多关键字来解决空安全，例如：`?.` `?:`  等，同时在编译时会对调用进行空检查，非常好的将空指针异常扼杀在了编译期。

## Kotlin用法分享

### 扩展函数

当我们使用第三方框架时(例如Java），发现其现有的代码方法不能满足我们的需求，这个时候我们要么自己去继承这个类，重写方法，要么直接使用源码在里面去修改，但是这两种方法代价都比较大。那有没有什么好的方法呢，正好Kotlin的出现给我们带来了一个神奇的函数——扩展函数：

Kotlin 可以直接对一个类的属性和方法进行扩展，且不需要继承，扩展是一种静态行为，对被扩展的类代码本身不会造成任何影响，只是定义在类的外面，而且扩展后在任意地方都可以直接调用该方法。

例如给String类添加一个方法：

```kotlin
fun String.lastChar():Char = this.get(this.length - 1)
//使用
val name = "xiaohong"
println(name.lastChar()) //输出 g
```

### 运算符重载

Kotiln中的很多特性的实现都十分相似，很多时候都是通过调用自己代码中的函数来实现特定的功能。例如如果你在类中定义了一个名为plus的特殊方法，那么按照kotlin`约定`你就可以使用+运算符。Kotlin中把这种通过简单的运算符就可以调用特定方法的技术称之为**约定** 。

例如定义两个点的相加操作：

```kotlin
data class Point(val x:Int,val y:Int){
    operator fun plus(other: Point):Point{
        return Point(x + other.x, y + other.y)//具体相加实现可按照实际需求定义
    }
}
//使用
val p1 = Point(2,3)
val p2 = Point(4,1)
println(p1+ p2) //输出 Point(6,4)
```

### Lambada编程

Lambda 表达式，或简称 lambda，本质上就是可以传递给其他函数的一小段代码 。 有了 lambda，可以轻松地把通用的代码结构抽取成库函数， Kotlin标准库就大量地 使用了它们。最常见的一种lambda用途就是和集合一起工作。lambda在kotlin中的常用的两个函数就是 with和apply

**with** ：with 标准库函数允许你调用同 一个对象的多个方法，而不需要反复写出这个对象的引用 。例如：

```kotlin
val leftAmount = with(mBuyInfo){
    println(totalamount)
    leftamount//直接调用mBuyInfo的属性 在最后一行作为返回值
}
```

**apply** : apply 函数让你使用构建者风格的 API 创建和初始化任何对象。 

```kotlin
fun createTextView(context:Context) = TextView(context).apply{
    text = "sample text"
    textSize = 20.0f
}
```

### 高阶函数

Lambda 是一个构建抽象概念的强大工具，它的能力并不局限于标准库中提供的集合和其他类，我们可以创建属于自己的高阶函数。 类似于数学中的高阶函数 f(g(x))，高阶函数的概念是：以函数作为参数或者返回值的函数。

在了解高阶函数之前，我们先来看看**函数类型**：

```kotlin
val sum = { x: Int,y: Int -> x + y }
```

能够清晰的看出，函数类型分为参数定义和具体的逻辑实现。在kotlin中函数也可以当作一个变量存储起来。

下面让我们来看看高阶函数的定义：

```kotlin
fun twoAndThree(operation: (Int, Int)-> Int){
    val result = operation(2, 3)
    println("this result is $result")
}
//调用twoAndThree函数传入上面定义的sum函数
twoAndThree(sum)//this result is 5
```

可以看到我们将一个函数类型当作参数传入到了高阶函数中，operation方法调用的就是sum函数类型定义的操作。

### 协程

上面讲到Kotlin从1.3开始正式发布了协程，主要的目的是让异步执行的代码，转变成看起来是顺序执行的，之前开启一个异步任务，我们肯跟会想到如下几种方式：

> 1. 开启一个线程处理
> 2. 利用AsyncTask
> 3. 利用CallBack
> 4. 利用RxJava

上面的方式都是在一个线程处理比较好耗时的操作，然后还会涉及到线程的切换，只是写法上有的会更先进一些。但是通过kotlin的协程则可以使其变得更人性化，代码开起来更直观。

例之前在安卓上下载文件的例子：

```java
 private class DownloadFilesTask extends AsyncTask<URL, Integer, Long> {
     protected Long doInBackground(URL... urls) {
         int count = urls.length;
         long totalSize = 0;
         for (int i = 0; i < count; i++) {
             totalSize += Downloader.downloadFile(urls[i]);
             publishProgress((int) ((i / (float) count) * 100));
             // Escape early if cancel() is called
             if (isCancelled()) break;
         }
         return totalSize;
     }

     protected void onProgressUpdate(Integer... progress) {
         setProgressPercent(progress[0]);
     }

     protected void onPostExecute(Long result) {
         showDialog("Downloaded " + result + " bytes");
     }
 }
```

当回调过多，代码层级嵌套就会越来愈多，从而形成了地狱回调。但通过使用Kotlin的async/await之后呢：

```kotlin
fun main(args: Array<String>) {
    val future = async<String> {
        (1..5).map {
            await(startLongAsyncOperation(it)) 
        }.joinToString("\n")
    }
    println(future.get())
}
```

代码看起来是不是简洁很多，kotlin通过协程完美的解决了之前在java上会出现的地狱回调的问题。

## 总结

本文简要的介绍的kotlin的一些特性和用法，对于想上手的朋友来说，了解了这些已经足够进行日常的开发工作了。但是想要深入的了解背后的原理，和一些特殊的用法，推荐如下两种方式：

1.可以通过阅读 **《kotlin实战》**这本书系统学习kotlin。
2.通过查阅[官方文档](<https://kotlinlang.org/docs/reference/>)进行学习和了解最新的特性和用法。

如果说这篇文章是kotlin混合开发的开胃菜，那么下篇文章则将会给大家带来kotlin混合开发中最需要注意的问题，都是我在实际开发中的一些总结，值得期待。

